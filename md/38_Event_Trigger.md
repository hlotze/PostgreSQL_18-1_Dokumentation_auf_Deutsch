# 38. Event-Trigger

Als Ergänzung zu dem in [Kapitel 37](37_Trigger.md) besprochenen Triggermechanismus stellt PostgreSQL auch Event-Trigger bereit. Anders als reguläre Trigger, die an eine einzelne Tabelle gebunden sind und nur DML-Ereignisse erfassen, gelten Event-Trigger global für eine bestimmte Datenbank und können DDL-Ereignisse erfassen.

Wie reguläre Trigger können Event-Trigger in jeder prozeduralen Sprache geschrieben werden, die Event-Trigger unterstützt, oder in C, aber nicht in einfachem SQL.

## 38.1. Überblick über das Verhalten von Event-Triggern

Ein Event-Trigger feuert immer dann, wenn das Ereignis, dem er zugeordnet ist, in der Datenbank auftritt, in der er definiert ist. Derzeit werden die Ereignisse `login`, `ddl_command_start`, `ddl_command_end`, `table_rewrite` und `sql_drop` unterstützt. Unterstützung für weitere Ereignisse kann in zukünftigen Versionen hinzugefügt werden.

### 38.1.1. login

Das Ereignis `login` tritt auf, wenn sich ein authentifizierter Benutzer am System anmeldet. Jeder Fehler in einer Triggerprozedur für dieses Ereignis kann eine erfolgreiche Anmeldung am System verhindern. Solche Fehler lassen sich umgehen, indem `event_triggers` entweder in einer Verbindungszeichenkette oder in einer Konfigurationsdatei auf `false` gesetzt wird. Alternativ können Sie das System im Einzelbenutzermodus neu starten, da Event-Trigger in diesem Modus deaktiviert sind. Details zur Verwendung des Einzelbenutzermodus finden Sie auf der Referenzseite zu `postgres`. Das Ereignis `login` feuert auch auf Standby-Servern. Um zu verhindern, dass Server unzugänglich werden, dürfen solche Trigger nichts in die Datenbank schreiben, wenn sie auf einem Standby laufen. Außerdem wird empfohlen, lang laufende Abfragen in `login`-Event-Triggern zu vermeiden. Beachten Sie zum Beispiel, dass das Abbrechen einer Verbindung in `psql` den gerade laufenden Login-Trigger nicht abbricht.

Ein Beispiel zur Verwendung des `login`-Event-Triggers finden Sie in [Abschnitt 38.5](#385-ein-datenbanklogineventtriggerbeispiel).

### 38.1.2. ddl_command_start

Das Ereignis `ddl_command_start` tritt unmittelbar vor der Ausführung eines DDL-Befehls auf. DDL-Befehle in diesem Kontext sind:

- `CREATE`
- `ALTER`
- `DROP`
- `COMMENT`
- `GRANT`
- `IMPORT FOREIGN SCHEMA`
- `REINDEX`
- `REFRESH MATERIALIZED VIEW`
- `REVOKE`
- `SECURITY LABEL`

`ddl_command_start` tritt auch unmittelbar vor der Ausführung eines Befehls `SELECT INTO` auf, da dieser äquivalent zu `CREATE TABLE AS` ist.

Als Ausnahme tritt dieses Ereignis nicht bei DDL-Befehlen auf, die auf gemeinsam genutzte Objekte zielen:

- Datenbanken
- Rollen, also Rollendefinitionen und Rollenmitgliedschaften
- Tablespaces
- Parameterprivilegien
- `ALTER SYSTEM`

Dieses Ereignis tritt außerdem nicht bei Befehlen auf, die auf Event-Trigger selbst zielen.

Bevor der Event-Trigger feuert, wird nicht geprüft, ob das betroffene Objekt existiert oder nicht existiert.

### 38.1.3. ddl_command_end

Das Ereignis `ddl_command_end` tritt unmittelbar nach der Ausführung derselben Menge von Befehlen wie `ddl_command_start` auf. Um weitere Details zu den ausgeführten DDL-Operationen zu erhalten, verwenden Sie die mengenrückgebende Funktion `pg_event_trigger_ddl_commands()` aus dem Event-Trigger-Code für `ddl_command_end` (siehe [Abschnitt 9.30](09_Funktionen_und_Operatoren.md#930-eventtriggerfunktionen)). Beachten Sie, dass der Trigger feuert, nachdem die Aktionen stattgefunden haben, aber bevor die Transaktion committet wird; die Systemkataloge können daher bereits im geänderten Zustand gelesen werden.

### 38.1.4. sql_drop

Das Ereignis `sql_drop` tritt unmittelbar vor dem Event-Trigger `ddl_command_end` für jede Operation auf, die Datenbankobjekte löscht. Beachten Sie, dass neben den offensichtlichen `DROP`-Befehlen auch einige `ALTER`-Befehle ein Ereignis `sql_drop` auslösen können.

Um die gelöschten Objekte aufzulisten, verwenden Sie die mengenrückgebende Funktion `pg_event_trigger_dropped_objects()` aus dem Event-Trigger-Code für `sql_drop` (siehe [Abschnitt 9.30](09_Funktionen_und_Operatoren.md#930-eventtriggerfunktionen)). Beachten Sie, dass der Trigger ausgeführt wird, nachdem die Objekte aus den Systemkatalogen gelöscht wurden; es ist daher nicht mehr möglich, sie dort nachzuschlagen.

### 38.1.5. table_rewrite

Das Ereignis `table_rewrite` tritt unmittelbar auf, bevor eine Tabelle durch bestimmte Aktionen der Befehle `ALTER TABLE` und `ALTER TYPE` umgeschrieben wird. Zwar stehen andere Steuerbefehle zur Verfügung, um eine Tabelle umzuschreiben, etwa `CLUSTER` und `VACUUM`, aber das Ereignis `table_rewrite` wird durch diese nicht ausgelöst. Um die OID der Tabelle zu finden, die umgeschrieben wurde, verwenden Sie die Funktion `pg_event_trigger_table_rewrite_oid()`. Um den oder die Gründe für das Umschreiben zu ermitteln, verwenden Sie die Funktion `pg_event_trigger_table_rewrite_reason()` (siehe [Abschnitt 9.30](09_Funktionen_und_Operatoren.md#930-eventtriggerfunktionen)).

### 38.1.6. Event-Trigger in abgebrochenen Transaktionen

Event-Trigger können wie andere Funktionen nicht in einer abgebrochenen Transaktion ausgeführt werden. Wenn ein DDL-Befehl also mit einem Fehler fehlschlägt, werden alle zugehörigen Trigger `ddl_command_end` nicht ausgeführt. Umgekehrt gilt: Wenn ein Trigger `ddl_command_start` mit einem Fehler fehlschlägt, feuern keine weiteren Event-Trigger, und es wird kein Versuch unternommen, den Befehl selbst auszuführen. Ebenso werden die Wirkungen der DDL-Anweisung zurückgerollt, wenn ein Trigger `ddl_command_end` mit einem Fehler fehlschlägt, genau wie in jedem anderen Fall, in dem die enthaltende Transaktion abbricht.

### 38.1.7. Event-Trigger erstellen

Event-Trigger werden mit dem Befehl `CREATE EVENT TRIGGER` erstellt. Um einen Event-Trigger zu erstellen, müssen Sie zuerst eine Funktion mit dem speziellen Rückgabetyp `event_trigger` erstellen. Diese Funktion muss und darf keinen Wert zurückgeben; der Rückgabetyp dient lediglich als Signal, dass die Funktion als Event-Trigger aufgerufen werden soll.

Wenn für ein bestimmtes Ereignis mehr als ein Event-Trigger definiert ist, feuern sie in alphabetischer Reihenfolge nach Triggername.

Eine Triggerdefinition kann außerdem eine `WHEN`-Bedingung angeben, sodass zum Beispiel ein Trigger `ddl_command_start` nur für bestimmte Befehle feuert, die der Benutzer abfangen möchte. Eine häufige Verwendung solcher Trigger besteht darin, den Bereich der DDL-Operationen einzuschränken, die Benutzer ausführen dürfen.

## 38.2. Event-Triggerfunktionen in C schreiben

Dieser Abschnitt beschreibt die Low-Level-Details der Schnittstelle zu einer Event-Triggerfunktion. Diese Informationen werden nur benötigt, wenn Event-Triggerfunktionen in C geschrieben werden. Wenn Sie eine höhere Sprache verwenden, werden diese Details für Sie behandelt. In den meisten Fällen sollten Sie eine prozedurale Sprache in Betracht ziehen, bevor Sie Ihre Event-Trigger in C schreiben. Die Dokumentation jeder prozeduralen Sprache erklärt, wie ein Event-Trigger in dieser Sprache geschrieben wird.

Event-Triggerfunktionen müssen die Funktionsmanager-Schnittstelle „Version 1“ verwenden.

Wenn eine Funktion vom Event-Trigger-Manager aufgerufen wird, werden ihr keine normalen Argumente übergeben, sondern ein „context“-Zeiger, der auf eine Struktur `EventTriggerData` zeigt. C-Funktionen können prüfen, ob sie vom Event-Trigger-Manager aufgerufen wurden, indem sie das folgende Makro ausführen:

```c
CALLED_AS_EVENT_TRIGGER(fcinfo)
```

Dies expandiert zu:

```c
((fcinfo)->context != NULL && IsA((fcinfo)->context, EventTriggerData))
```

Wenn dies wahr zurückgibt, ist es sicher, `fcinfo->context` in den Typ `EventTriggerData *` umzuwandeln und die Struktur `EventTriggerData`, auf die gezeigt wird, zu verwenden. Die Funktion darf weder die Struktur `EventTriggerData` noch irgendeine der Daten verändern, auf die sie zeigt.

`struct EventTriggerData` ist in `commands/event_trigger.h` definiert:

```c
typedef struct EventTriggerData
{
    NodeTag     type;
    const char *event;     /* event name */
    Node       *parsetree; /* parse tree */
    CommandTag  tag;       /* command tag */
} EventTriggerData;
```

Die Mitglieder sind wie folgt definiert:

- `type`
  - Immer `T_EventTriggerData`.

- `event`
  - Beschreibt das Ereignis, für das die Funktion aufgerufen wird, eines von `"login"`, `"ddl_command_start"`, `"ddl_command_end"`, `"sql_drop"` oder `"table_rewrite"`. Die Bedeutung dieser Ereignisse wird in [Abschnitt 38.1](#381-überblick-über-das-verhalten-von-eventtriggern) beschrieben.

- `parsetree`
  - Ein Zeiger auf den Parse-Baum des Befehls. Details finden Sie im PostgreSQL-Quellcode. Die Struktur des Parse-Baums kann sich ohne Ankündigung ändern.

- `tag`
  - Das Befehlstag, das mit dem Ereignis verbunden ist, für das der Event-Trigger ausgeführt wird, zum Beispiel `"CREATE FUNCTION"`.

Eine Event-Triggerfunktion muss einen `NULL`-Zeiger zurückgeben, nicht einen SQL-Nullwert; setzen Sie also nicht `isNull` auf wahr.

## 38.3. Ein vollständiges Event-Triggerbeispiel

Hier ist ein sehr einfaches Beispiel für eine in C geschriebene Event-Triggerfunktion. Beispiele für Trigger, die in prozeduralen Sprachen geschrieben sind, finden Sie in der Dokumentation der jeweiligen prozeduralen Sprachen.

Die Funktion `noddl` löst bei jedem Aufruf eine Ausnahme aus. Die Event-Triggerdefinition verknüpft die Funktion mit dem Ereignis `ddl_command_start`. Die Wirkung ist, dass alle DDL-Befehle, mit den in [Abschnitt 38.1](#381-überblick-über-das-verhalten-von-eventtriggern) genannten Ausnahmen, an der Ausführung gehindert werden.

Dies ist der Quellcode der Triggerfunktion:

```c
#include "postgres.h"

#include "commands/event_trigger.h"
#include "fmgr.h"

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(noddl);

Datum
noddl(PG_FUNCTION_ARGS)
{
    EventTriggerData *trigdata;

    if (!CALLED_AS_EVENT_TRIGGER(fcinfo)) /* internal error */
        elog(ERROR, "not fired by event trigger manager");

    trigdata = (EventTriggerData *) fcinfo->context;

    ereport(ERROR,
            (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
             errmsg("command \"%s\" denied",
                    GetCommandTagName(trigdata->tag))));

    PG_RETURN_NULL();
}
```

Nachdem Sie den Quellcode kompiliert haben (siehe [Abschnitt 36.10.5](36_SQL_erweitern.md#36105-dynamisch-geladene-funktionen-kompilieren-und-linken)), deklarieren Sie die Funktion und den Trigger:

```sql
CREATE FUNCTION noddl() RETURNS event_trigger
    AS 'noddl' LANGUAGE C;

CREATE EVENT TRIGGER noddl ON ddl_command_start
    EXECUTE FUNCTION noddl();
```

Nun können Sie die Arbeitsweise des Triggers testen:

```text
=# \dy
                     List of event triggers
 Name |        Event       | Owner | Enabled | Function | Tags
------+--------------------+-------+---------+----------+------
 noddl | ddl_command_start | dim   | enabled | noddl    |
(1 row)

=# CREATE TABLE foo(id serial);
ERROR: command "CREATE TABLE" denied
```

Um in dieser Situation bei Bedarf trotzdem einige DDL-Befehle ausführen zu können, müssen Sie den Event-Trigger entweder löschen oder deaktivieren. Es kann praktisch sein, den Trigger nur für die Dauer einer Transaktion zu deaktivieren:

```sql
BEGIN;
ALTER EVENT TRIGGER noddl DISABLE;
CREATE TABLE foo (id serial);
ALTER EVENT TRIGGER noddl ENABLE;
COMMIT;
```

Denken Sie daran, dass DDL-Befehle auf Event-Trigger selbst nicht von Event-Triggern betroffen sind.

## 38.4. Ein Event-Triggerbeispiel für Tabellenumschreibung

Dank des Ereignisses `table_rewrite` ist es möglich, eine Richtlinie für das Umschreiben von Tabellen zu implementieren, die das Umschreiben nur in Wartungsfenstern erlaubt.

Hier ist ein Beispiel, das eine solche Richtlinie implementiert:

```sql
CREATE OR REPLACE FUNCTION no_rewrite()
 RETURNS event_trigger
 LANGUAGE plpgsql AS
$$
---
--- Implement local Table Rewriting policy:
---     public.foo is not allowed rewriting, ever
---     other tables are only allowed rewriting between 1am and 6am
---     unless they have more than 100 blocks
---
DECLARE
   table_oid oid := pg_event_trigger_table_rewrite_oid();
   current_hour integer := extract('hour' from current_time);
   pages integer;
   max_pages integer := 100;
BEGIN
   IF pg_event_trigger_table_rewrite_oid() = 'public.foo'::regclass
   THEN
      RAISE EXCEPTION 'you''re not allowed to rewrite the table %',
                      table_oid::regclass;
   END IF;

   SELECT INTO pages relpages FROM pg_class WHERE oid = table_oid;
   IF pages > max_pages
   THEN
      RAISE EXCEPTION 'rewrites only allowed for table with less than % pages',
                      max_pages;
   END IF;

   IF current_hour NOT BETWEEN 1 AND 6
   THEN
      RAISE EXCEPTION 'rewrites only allowed between 1am and 6am';
   END IF;
END;
$$;

CREATE EVENT TRIGGER no_rewrite_allowed
                  ON table_rewrite
   EXECUTE FUNCTION no_rewrite();
```

## 38.5. Ein Datenbank-Login-Event-Triggerbeispiel

Der Event-Trigger für das Ereignis `login` kann nützlich sein, um Benutzeranmeldungen zu protokollieren, die Verbindung zu prüfen und Rollen entsprechend den aktuellen Umständen zuzuweisen oder Sitzungsdaten zu initialisieren. Es ist sehr wichtig, dass jeder Event-Trigger, der das Ereignis `login` verwendet, prüft, ob sich die Datenbank in Recovery befindet, bevor er Schreibvorgänge ausführt. Das Schreiben auf einen Standby-Server macht ihn unzugänglich.

Das folgende Beispiel demonstriert diese Möglichkeiten:

```sql
-- create test tables and roles
CREATE TABLE user_login_log (
   "user" text,
   "session_start" timestamp with time zone
);
CREATE ROLE day_worker;
CREATE ROLE night_worker;

-- the example trigger function
CREATE OR REPLACE FUNCTION init_session()
   RETURNS event_trigger SECURITY DEFINER
   LANGUAGE plpgsql AS
$$
DECLARE
   hour integer = EXTRACT('hour' FROM current_time at time zone 'utc');
   rec boolean;
BEGIN
   -- 1. Forbid logging in between 2AM and 4AM.
   IF hour BETWEEN 2 AND 4 THEN
      RAISE EXCEPTION 'Login forbidden';
   END IF;

   -- The checks below cannot be performed on standby servers so
   -- ensure the database is not in recovery before we perform any
   -- operations.
   SELECT pg_is_in_recovery() INTO rec;
   IF rec THEN
      RETURN;
   END IF;

   -- 2. Assign some roles. At daytime, grant the day_worker role,
   -- else the night_worker role.
   IF hour BETWEEN 8 AND 20 THEN
      EXECUTE 'REVOKE night_worker FROM ' || quote_ident(session_user);
      EXECUTE 'GRANT day_worker TO ' || quote_ident(session_user);
   ELSE
      EXECUTE 'REVOKE day_worker FROM ' || quote_ident(session_user);
      EXECUTE 'GRANT night_worker TO ' || quote_ident(session_user);
   END IF;

   -- 3. Initialize user session data
   CREATE TEMP TABLE session_storage (x float, y integer);
   ALTER TABLE session_storage OWNER TO session_user;

   -- 4. Log the connection time
   INSERT INTO public.user_login_log VALUES (session_user, current_timestamp);
END;
$$;

-- trigger definition
CREATE EVENT TRIGGER init_session
  ON login
  EXECUTE FUNCTION init_session();
ALTER EVENT TRIGGER init_session ENABLE ALWAYS;
```
