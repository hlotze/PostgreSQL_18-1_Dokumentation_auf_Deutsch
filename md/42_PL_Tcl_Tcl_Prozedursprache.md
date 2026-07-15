# 42. PL/Tcl - Tcl-Prozedursprache

PL/Tcl ist eine ladbare prozedurale Sprache für das PostgreSQL-Datenbanksystem, mit der die Sprache [Tcl](https://www.tcl.tk/) zum Schreiben von PostgreSQL-Funktionen und -Prozeduren verwendet werden kann.

## 42.1. Überblick

PL/Tcl bietet mit einigen Einschränkungen die meisten Möglichkeiten, die ein Funktionsautor in der Sprache C hat, ergänzt um die leistungsfähigen Bibliotheken zur Zeichenkettenverarbeitung, die für Tcl verfügbar sind.

Eine gute und überzeugende Einschränkung ist, dass alles innerhalb des sicheren Kontexts eines Tcl-Interpreters ausgeführt wird. Zusätzlich zum eingeschränkten Befehlssatz von Safe Tcl stehen nur wenige Befehle zur Verfügung, um über SPI auf die Datenbank zuzugreifen und über `elog()` Meldungen auszugeben. PL/Tcl bietet keine Möglichkeit, auf Interna des Datenbankservers zuzugreifen oder mit den Berechtigungen des PostgreSQL-Serverprozesses Zugriff auf Betriebssystemebene zu erlangen, wie es eine C-Funktion tun kann. Daher kann man nichtprivilegierten Datenbankbenutzern die Verwendung dieser Sprache anvertrauen; sie verleiht ihnen keine unbegrenzten Rechte.

Die andere wichtige Implementierungseinschränkung ist, dass Tcl-Funktionen nicht verwendet werden können, um Ein-/Ausgabefunktionen für neue Datentypen zu erstellen.

Manchmal ist es wünschenswert, Tcl-Funktionen zu schreiben, die nicht auf Safe Tcl beschränkt sind. Zum Beispiel könnte man eine Tcl-Funktion wünschen, die E-Mail sendet. Für solche Fälle gibt es eine Variante von PL/Tcl namens PL/TclU (für untrusted Tcl). Dies ist exakt dieselbe Sprache, außer dass ein vollständiger Tcl-Interpreter verwendet wird. Wenn PL/TclU verwendet wird, muss es als nicht vertrauenswürdige prozedurale Sprache installiert werden, sodass nur Datenbank-Superuser Funktionen darin erstellen können. Der Autor einer PL/TclU-Funktion muss darauf achten, dass die Funktion nicht für unerwünschte Aktionen verwendet werden kann, da sie alles tun kann, was ein als Datenbankadministrator angemeldeter Benutzer tun könnte.

Der Shared-Object-Code für die Call-Handler von PL/Tcl und PL/TclU wird automatisch gebaut und im PostgreSQL-Bibliotheksverzeichnis installiert, wenn Tcl-Unterstützung im Konfigurationsschritt des Installationsverfahrens angegeben wurde. Um PL/Tcl und/oder PL/TclU in einer bestimmten Datenbank zu installieren, verwenden Sie den Befehl `CREATE EXTENSION`, zum Beispiel `CREATE EXTENSION pltcl` oder `CREATE EXTENSION pltclu`.

## 42.2. PL/Tcl-Funktionen und Argumente

Um eine Funktion in der Sprache PL/Tcl zu erstellen, verwenden Sie die Standardsyntax von `CREATE FUNCTION`:

```sql
CREATE FUNCTION funcname (argument-types) RETURNS return-type AS $$
    # PL/Tcl function body
$$ LANGUAGE pltcl;
```

PL/TclU ist gleich, außer dass die Sprache als `pltclu` angegeben werden muss.

Der Funktionsrumpf ist einfach ein Stück Tcl-Skript. Wenn die Funktion aufgerufen wird, werden die Argumentwerte an das Tcl-Skript als Variablen mit den Namen `1` bis `n` übergeben. Das Ergebnis wird aus dem Tcl-Code auf die übliche Weise mit einer `return`-Anweisung zurückgegeben. In einer Prozedur wird der Rückgabewert des Tcl-Codes ignoriert.

Zum Beispiel könnte eine Funktion, die den größeren von zwei ganzzahligen Werten zurückgibt, so definiert werden:

```sql
CREATE FUNCTION tcl_max(integer, integer) RETURNS integer AS $$
    if {$1 > $2} {return $1}
    return $2
$$ LANGUAGE pltcl STRICT;
```

Beachten Sie die Klausel `STRICT`, die uns erspart, über Null-Eingabewerte nachzudenken: Wenn ein Nullwert übergeben wird, wird die Funktion überhaupt nicht aufgerufen, sondern gibt automatisch ein Nullergebnis zurück.

In einer nicht-strikten Funktion wird die entsprechende Variable `$n` auf eine leere Zeichenkette gesetzt, wenn der tatsächliche Wert eines Arguments null ist. Um zu erkennen, ob ein bestimmtes Argument null ist, verwenden Sie die Funktion `argisnull`. Nehmen wir zum Beispiel an, wir möchten, dass `tcl_max` bei einem Nullargument und einem Nicht-null-Argument das Nicht-null-Argument zurückgibt, statt null:

```sql
CREATE FUNCTION tcl_max(integer, integer) RETURNS integer AS $$
    if {[argisnull 1]} {
        if {[argisnull 2]} { return_null }
        return $2
    }
    if {[argisnull 2]} { return $1 }
    if {$1 > $2} {return $1}
    return $2
$$ LANGUAGE pltcl;
```

Wie oben gezeigt, führen Sie `return_null` aus, um aus einer PL/Tcl-Funktion einen Nullwert zurückzugeben. Das kann unabhängig davon geschehen, ob die Funktion strikt ist oder nicht.

Argumente zusammengesetzter Typen werden der Funktion als Tcl-Arrays übergeben. Die Elementnamen des Arrays sind die Attributnamen des zusammengesetzten Typs. Wenn ein Attribut in der übergebenen Zeile den Nullwert hat, erscheint es nicht im Array. Hier ist ein Beispiel:

```sql
CREATE TABLE employee (
    name text,
    salary integer,
    age integer
);

CREATE FUNCTION overpaid(employee) RETURNS boolean AS $$
    if {200000.0 < $1(salary)} {
        return "t"
    }
    if {$1(age) < 30 && 100000.0 < $1(salary)} {
        return "t"
    }
    return "f"
$$ LANGUAGE pltcl;
```

PL/Tcl-Funktionen können auch Ergebnisse zusammengesetzter Typen zurückgeben. Dazu muss der Tcl-Code eine Liste von Paaren aus Spaltenname und Wert zurückgeben, die zum erwarteten Ergebnistyp passen. Spaltennamen, die in der Liste fehlen, werden als null zurückgegeben, und ein Fehler wird ausgelöst, wenn unerwartete Spaltennamen vorkommen. Hier ist ein Beispiel:

```sql
CREATE FUNCTION square_cube(in int, out squared int, out cubed int)
 AS $$
    return [list squared [expr {$1 * $1}] cubed [expr {$1 * $1 *
 $1}]]
$$ LANGUAGE pltcl;
```

Ausgabeargumente von Prozeduren werden auf dieselbe Weise zurückgegeben, zum Beispiel:

```sql
CREATE PROCEDURE tcl_triple(INOUT a integer, INOUT b integer) AS $$
    return [list a [expr {$1 * 3}] b [expr {$2 * 3}]]
$$ LANGUAGE pltcl;

CALL tcl_triple(5, 10);
```

> Die Ergebnisliste kann mit dem Tcl-Befehl `array get` aus einer Array-Darstellung des gewünschten Tupels erstellt werden. Zum Beispiel:
>
> ```sql
> CREATE FUNCTION raise_pay(employee, delta int) RETURNS
>  employee AS $$
>     set 1(salary) [expr {$1(salary) + $2}]
>     return [array get 1]
> $$ LANGUAGE pltcl;
> ```

PL/Tcl-Funktionen können Sets zurückgeben. Dazu sollte der Tcl-Code `return_next` einmal pro zurückzugebender Zeile aufrufen und dabei entweder den passenden Wert übergeben, wenn ein skalarer Typ zurückgegeben wird, oder eine Liste von Paaren aus Spaltenname und Wert, wenn ein zusammengesetzter Typ zurückgegeben wird. Hier ist ein Beispiel, das einen skalaren Typ zurückgibt:

```sql
CREATE FUNCTION sequence(int, int) RETURNS SETOF int AS $$
    for {set i $1} {$i < $2} {incr i} {
        return_next $i
    }
$$ LANGUAGE pltcl;
```

Und hier ist eines, das einen zusammengesetzten Typ zurückgibt:

```sql
CREATE FUNCTION table_of_squares(int, int) RETURNS TABLE (x int, x2
 int) AS $$
    for {set i $1} {$i < $2} {incr i} {
        return_next [list x $i x2 [expr {$i * $i}]]
    }
$$ LANGUAGE pltcl;
```

## 42.3. Datenwerte in PL/Tcl

Die Argumentwerte, die dem Code einer PL/Tcl-Funktion übergeben werden, sind einfach die Eingabeargumente, die in Textform umgewandelt wurden (genau so, als wären sie von einer `SELECT`-Anweisung angezeigt worden). Umgekehrt akzeptieren die Befehle `return` und `return_next` jede Zeichenkette, die ein zulässiges Eingabeformat für den deklarierten Ergebnistyp der Funktion oder für die angegebene Spalte eines zusammengesetzten Ergebnistyps ist.

## 42.4. Globale Daten in PL/Tcl

Manchmal ist es nützlich, globale Daten zu haben, die zwischen zwei Aufrufen einer Funktion erhalten bleiben oder zwischen verschiedenen Funktionen gemeinsam genutzt werden. Das ist in PL/Tcl leicht möglich, aber es gibt einige Einschränkungen, die verstanden werden müssen.

Aus Sicherheitsgründen führt PL/Tcl Funktionen, die von einer bestimmten SQL-Rolle aufgerufen werden, in einem separaten Tcl-Interpreter für diese Rolle aus. Dadurch wird verhindert, dass ein Benutzer versehentlich oder böswillig das Verhalten der PL/Tcl-Funktionen eines anderen Benutzers beeinflusst. Jeder dieser Interpreter hat eigene Werte für alle „globalen“ Tcl-Variablen. Daher teilen sich zwei PL/Tcl-Funktionen genau dann dieselben globalen Variablen, wenn sie von derselben SQL-Rolle ausgeführt werden. In einer Anwendung, in der eine einzelne Sitzung Code unter mehreren SQL-Rollen ausführt (über `SECURITY DEFINER`-Funktionen, die Verwendung von `SET ROLE` usw.), müssen Sie möglicherweise ausdrücklich dafür sorgen, dass PL/Tcl-Funktionen Daten gemeinsam nutzen können. Stellen Sie dazu sicher, dass Funktionen, die miteinander kommunizieren sollen, demselben Benutzer gehören, und markieren Sie sie als `SECURITY DEFINER`. Natürlich müssen Sie darauf achten, dass solche Funktionen nicht für unbeabsichtigte Aktionen verwendet werden können.

Alle PL/TclU-Funktionen, die in einer Sitzung verwendet werden, laufen im selben Tcl-Interpreter, der natürlich von den für PL/Tcl-Funktionen verwendeten Interpretern getrennt ist. Globale Daten werden daher automatisch zwischen PL/TclU-Funktionen geteilt. Dies gilt nicht als Sicherheitsrisiko, weil alle PL/TclU-Funktionen auf derselben Vertrauensebene ausgeführt werden, nämlich auf der eines Datenbank-Superusers.

Um PL/Tcl-Funktionen davor zu schützen, einander unbeabsichtigt zu beeinflussen, wird jeder Funktion über den Befehl `upvar` ein globales Array zur Verfügung gestellt. Der globale Name dieser Variablen ist der interne Name der Funktion, und der lokale Name ist `GD`. Es wird empfohlen, `GD` für persistente private Daten einer Funktion zu verwenden. Verwenden Sie normale globale Tcl-Variablen nur für Werte, die Sie ausdrücklich zwischen mehreren Funktionen teilen möchten. (Beachten Sie, dass die `GD`-Arrays nur innerhalb eines bestimmten Interpreters global sind und daher die oben erwähnten Sicherheitseinschränkungen nicht umgehen.)

Ein Beispiel für die Verwendung von `GD` erscheint unten im Beispiel zu `spi_execp`.

## 42.5. Datenbankzugriff aus PL/Tcl

In diesem Abschnitt folgen wir der üblichen Tcl-Konvention, in einer Syntaxübersicht Fragezeichen statt eckiger Klammern zu verwenden, um ein optionales Element zu kennzeichnen. Die folgenden Befehle stehen zur Verfügung, um aus dem Rumpf einer PL/Tcl-Funktion auf die Datenbank zuzugreifen:

- `spi_exec ?-count n? ?-array name? command ?loop-body?`
  - Führt einen als Zeichenkette angegebenen SQL-Befehl aus. Ein Fehler im Befehl löst einen Fehler aus. Andernfalls ist der Rückgabewert von `spi_exec` die Anzahl der vom Befehl verarbeiteten Zeilen (ausgewählt, eingefügt, aktualisiert oder gelöscht) oder null, wenn der Befehl eine Utility-Anweisung ist. Wenn der Befehl außerdem eine `SELECT`-Anweisung ist, werden die Werte der ausgewählten Spalten wie unten beschrieben in Tcl-Variablen abgelegt.

    Der optionale Wert `-count` weist `spi_exec` an, anzuhalten, sobald `n` Zeilen abgerufen wurden, ähnlich wie wenn die Abfrage eine `LIMIT`-Klausel enthielte. Wenn `n` null ist, wird die Abfrage vollständig ausgeführt, genauso wie wenn `-count` weggelassen wird.

    Wenn der Befehl eine `SELECT`-Anweisung ist, werden die Werte der Ergebnisspalten in Tcl-Variablen abgelegt, die nach den Spalten benannt sind. Wenn die Option `-array` angegeben ist, werden die Spaltenwerte stattdessen in Elemente des benannten assoziativen Arrays gespeichert, wobei die Spaltennamen als Array-Indizes verwendet werden. Zusätzlich wird die aktuelle Zeilennummer innerhalb des Ergebnisses (ab null gezählt) im Array-Element namens `.tupno` gespeichert, sofern dieser Name nicht als Spaltenname im Ergebnis verwendet wird.

    Wenn der Befehl eine `SELECT`-Anweisung ist und kein `loop-body`-Skript angegeben wird, wird nur die erste Ergebniszeile in Tcl-Variablen oder Array-Elementen gespeichert; verbleibende Zeilen, falls vorhanden, werden ignoriert. Es wird nichts gespeichert, wenn die Abfrage keine Zeilen zurückgibt. (Dieser Fall kann durch Prüfen des Ergebnisses von `spi_exec` erkannt werden.) Zum Beispiel:

    ```sql
    spi_exec "SELECT count(*) AS cnt FROM pg_proc"
    ```

    setzt die Tcl-Variable `$cnt` auf die Anzahl der Zeilen im Systemkatalog `pg_proc`.

    Wenn das optionale Argument `loop-body` angegeben ist, ist es ein Stück Tcl-Skript, das einmal für jede Zeile im Abfrageergebnis ausgeführt wird. (`loop-body` wird ignoriert, wenn der angegebene Befehl kein `SELECT` ist.) Die Werte der Spalten der aktuellen Zeile werden vor jeder Iteration in Tcl-Variablen oder Array-Elemente gespeichert. Zum Beispiel:

    ```sql
    spi_exec -array C "SELECT * FROM pg_class" {
        elog DEBUG "have table $C(relname)"
    }
    ```

    gibt für jede Zeile von `pg_class` eine Logmeldung aus. Diese Funktionalität arbeitet ähnlich wie andere Tcl-Schleifenkonstrukte; insbesondere funktionieren `continue` und `break` innerhalb des Schleifenrumpfs auf die übliche Weise.

    Wenn eine Spalte eines Abfrageergebnisses null ist, wird die Zielvariable dafür „unset“ statt gesetzt.

- `spi_prepare query typelist`
  - Bereitet einen Abfrageplan für spätere Ausführung vor und speichert ihn. Der gespeicherte Plan bleibt für die Lebensdauer der aktuellen Sitzung erhalten.

    Die Abfrage kann Parameter verwenden, also Platzhalter für Werte, die immer dann geliefert werden, wenn der Plan tatsächlich ausgeführt wird. In der Abfragezeichenkette verweisen Sie mit den Symbolen `$1` bis `$n` auf Parameter. Wenn die Abfrage Parameter verwendet, müssen die Namen der Parametertypen als Tcl-Liste angegeben werden. (Schreiben Sie eine leere Liste für `typelist`, wenn keine Parameter verwendet werden.)

    Der Rückgabewert von `spi_prepare` ist eine Abfrage-ID, die in späteren Aufrufen von `spi_execp` verwendet wird. Ein Beispiel finden Sie bei `spi_execp`.

- `spi_execp ?-count n? ?-array name? ?-nulls string? queryid ?value-list? ?loop-body?`
  - Führt eine zuvor mit `spi_prepare` vorbereitete Abfrage aus. `queryid` ist die von `spi_prepare` zurückgegebene ID. Wenn die Abfrage Parameter referenziert, muss eine `value-list` angegeben werden. Dies ist eine Tcl-Liste der tatsächlichen Werte für die Parameter. Die Liste muss dieselbe Länge haben wie die zuvor an `spi_prepare` übergebene Parametertyp-Liste. Lassen Sie `value-list` weg, wenn die Abfrage keine Parameter hat.

    Der optionale Wert für `-nulls` ist eine Zeichenkette aus Leerzeichen und `n`-Zeichen, die `spi_execp` mitteilt, welche der Parameter Nullwerte sind. Falls angegeben, muss sie exakt dieselbe Länge wie die `value-list` haben. Wenn sie nicht angegeben wird, sind alle Parameterwerte nicht-null.

    Abgesehen von der Art, wie die Abfrage und ihre Parameter angegeben werden, funktioniert `spi_execp` genau wie `spi_exec`. Die Optionen `-count`, `-array` und `loop-body` sind dieselben, ebenso der Ergebniswert.

    Hier ist ein Beispiel für eine PL/Tcl-Funktion, die einen vorbereiteten Plan verwendet:

    ```sql
    CREATE FUNCTION t1_count(integer, integer) RETURNS integer AS $$
        if {![ info exists GD(plan) ]} {
            # prepare the saved plan on the first call
            set GD(plan) [ spi_prepare \
                    "SELECT count(*) AS cnt FROM t1 WHERE num >= \$1
     AND num <= \$2" \
                    [ list int4 int4 ] ]
        }
        spi_execp -count 1 $GD(plan) [ list $1 $2 ]
        return $cnt
    $$ LANGUAGE pltcl;
    ```

    Wir benötigen Backslashes innerhalb der an `spi_prepare` übergebenen Abfragezeichenkette, um sicherzustellen, dass die Marker `$n` unverändert an `spi_prepare` weitergereicht und nicht durch Tcl-Variablenersetzung ersetzt werden.

- `subtransaction command`
  - Das in `command` enthaltene Tcl-Skript wird innerhalb einer SQL-Subtransaktion ausgeführt. Wenn das Skript einen Fehler zurückgibt, wird diese gesamte Subtransaktion zurückgerollt, bevor der Fehler an den umgebenden Tcl-Code zurückgegeben wird. Weitere Details und ein Beispiel finden Sie in [Abschnitt 42.9](#429-explizite-subtransaktionen-in-pltcl).

- `quote string`
  - Verdoppelt alle Vorkommen von einfachen Anführungszeichen und Backslash-Zeichen in der angegebenen Zeichenkette. Dies kann verwendet werden, um Zeichenketten sicher zu quoten, die in SQL-Befehle eingefügt werden sollen, die an `spi_exec` oder `spi_prepare` übergeben werden. Denken Sie zum Beispiel an eine SQL-Befehlszeichenkette wie:

    ```text
    "SELECT '$val' AS ret"
    ```

    wobei die Tcl-Variable `val` tatsächlich `doesn't` enthält. Dies würde zur endgültigen Befehlszeichenkette führen:

    ```sql
    SELECT 'doesn't' AS ret
    ```

    was während `spi_exec` oder `spi_prepare` einen Parse-Fehler verursachen würde. Damit es korrekt funktioniert, sollte der übermittelte Befehl Folgendes enthalten:

    ```sql
    SELECT 'doesn''t' AS ret
    ```

    Das kann in PL/Tcl so gebildet werden:

    ```text
    "SELECT '[ quote $val ]' AS ret"
    ```

    Ein Vorteil von `spi_execp` ist, dass Sie Parameterwerte nicht auf diese Weise quoten müssen, da die Parameter nie als Teil einer SQL-Befehlszeichenkette geparst werden.

- `elog level msg`
  - Gibt eine Log- oder Fehlermeldung aus. Mögliche Stufen sind `DEBUG`, `LOG`, `INFO`, `NOTICE`, `WARNING`, `ERROR` und `FATAL`. `ERROR` löst eine Fehlerbedingung aus; wenn diese nicht vom umgebenden Tcl-Code abgefangen wird, breitet sich der Fehler bis zur aufrufenden Abfrage aus, wodurch die aktuelle Transaktion oder Subtransaktion abgebrochen wird. Dies ist effektiv dasselbe wie der Tcl-Befehl `error`. `FATAL` bricht die Transaktion ab und beendet die aktuelle Sitzung. (Es gibt vermutlich keinen guten Grund, diese Fehlerstufe in PL/Tcl-Funktionen zu verwenden, sie wird aber der Vollständigkeit halber bereitgestellt.) Die anderen Stufen erzeugen nur Meldungen unterschiedlicher Priorität. Ob Meldungen einer bestimmten Priorität an den Client gemeldet, in das Serverlog geschrieben oder beides werden, wird durch die Konfigurationsvariablen `log_min_messages` und `client_min_messages` gesteuert. Weitere Informationen finden Sie in [Kapitel 19](19_Serverkonfiguration.md) und [Abschnitt 42.8](#428-fehlerbehandlung-in-pltcl).

## 42.6. Trigger-Funktionen in PL/Tcl

Trigger-Funktionen können in PL/Tcl geschrieben werden. PostgreSQL verlangt, dass eine Funktion, die als Trigger aufgerufen werden soll, als Funktion ohne Argumente und mit dem Rückgabetyp `trigger` deklariert wird.

Die Informationen vom Trigger-Manager werden dem Funktionsrumpf in den folgenden Variablen übergeben:

- `$TG_name`
  - Der Name des Triggers aus der Anweisung `CREATE TRIGGER`.

- `$TG_relid`
  - Die Objekt-ID der Tabelle, die den Aufruf der Trigger-Funktion verursacht hat.

- `$TG_table_name`
  - Der Name der Tabelle, die den Aufruf der Trigger-Funktion verursacht hat.

- `$TG_table_schema`
  - Das Schema der Tabelle, die den Aufruf der Trigger-Funktion verursacht hat.

- `$TG_relatts`
  - Eine Tcl-Liste der Tabellenspaltennamen, der ein leeres Listenelement vorangestellt ist. Wenn man also einen Spaltennamen in der Liste mit Tcls Befehl `lsearch` nachschlägt, erhält man die Elementnummer beginnend mit 1 für die erste Spalte, genauso wie Spalten in PostgreSQL üblicherweise nummeriert werden. (Leere Listenelemente erscheinen auch an den Positionen von Spalten, die gelöscht wurden, sodass die Attributnummerierung für Spalten rechts davon korrekt ist.)

- `$TG_when`
  - Die Zeichenkette `BEFORE`, `AFTER` oder `INSTEAD OF`, je nach Art des Trigger-Ereignisses.

- `$TG_level`
  - Die Zeichenkette `ROW` oder `STATEMENT`, je nach Art des Trigger-Ereignisses.

- `$TG_op`
  - Die Zeichenkette `INSERT`, `UPDATE`, `DELETE` oder `TRUNCATE`, je nach Art des Trigger-Ereignisses.

- `$NEW`
  - Ein assoziatives Array mit den Werten der neuen Tabellenzeile für `INSERT`- oder `UPDATE`-Aktionen oder leer für `DELETE`. Das Array wird nach Spaltennamen indiziert. Spalten, die null sind, erscheinen nicht im Array. Für anweisungsbezogene Trigger ist dies nicht gesetzt.

- `$OLD`
  - Ein assoziatives Array mit den Werten der alten Tabellenzeile für `UPDATE`- oder `DELETE`-Aktionen oder leer für `INSERT`. Das Array wird nach Spaltennamen indiziert. Spalten, die null sind, erscheinen nicht im Array. Für anweisungsbezogene Trigger ist dies nicht gesetzt.

- `$args`
  - Eine Tcl-Liste der Argumente an die Funktion, wie sie in der Anweisung `CREATE TRIGGER` angegeben wurden. Diese Argumente sind im Funktionsrumpf auch als `$1` bis `$n` zugänglich.

Der Rückgabewert einer Trigger-Funktion kann eine der Zeichenketten `OK` oder `SKIP` sein oder eine Liste von Paaren aus Spaltenname und Wert. Wenn der Rückgabewert `OK` ist, läuft die Operation (`INSERT`/`UPDATE`/`DELETE`), die den Trigger ausgelöst hat, normal weiter. `SKIP` weist den Trigger-Manager an, die Operation für diese Zeile stillschweigend zu unterdrücken. Wenn eine Liste zurückgegeben wird, weist sie PL/Tcl an, eine geänderte Zeile an den Trigger-Manager zurückzugeben; der Inhalt der geänderten Zeile wird durch die Spaltennamen und Werte in der Liste angegeben. Alle Spalten, die in der Liste nicht erwähnt werden, werden auf null gesetzt. Die Rückgabe einer geänderten Zeile ist nur für zeilenbezogene `BEFORE INSERT`- oder `UPDATE`-Trigger sinnvoll, bei denen die geänderte Zeile statt der in `$NEW` angegebenen eingefügt wird; oder für zeilenbezogene `INSTEAD OF INSERT`- oder `UPDATE`-Trigger, bei denen die zurückgegebene Zeile als Quelldaten für `INSERT RETURNING`- oder `UPDATE RETURNING`-Klauseln verwendet wird. In zeilenbezogenen `BEFORE DELETE`- oder `INSTEAD OF DELETE`-Triggern hat die Rückgabe einer geänderten Zeile dieselbe Wirkung wie die Rückgabe von `OK`, das heißt, die Operation wird fortgesetzt. Für alle anderen Triggertypen wird der Trigger-Rückgabewert ignoriert.

> Die Ergebnisliste kann mit dem Tcl-Befehl `array get` aus einer Array-Darstellung des geänderten Tupels erzeugt werden.

Hier ist eine kleine Beispiel-Trigger-Funktion, die erzwingt, dass ein ganzzahliger Wert in einer Tabelle die Anzahl der Aktualisierungen der Zeile mitverfolgt. Für neu eingefügte Zeilen wird der Wert auf 0 initialisiert und dann bei jeder Aktualisierung inkrementiert.

```sql
CREATE FUNCTION trigfunc_modcount() RETURNS trigger AS $$
    switch $TG_op {
        INSERT {
            set NEW($1) 0
        }
        UPDATE {
            set NEW($1) $OLD($1)
            incr NEW($1)
        }
        default {
            return OK
        }
    }
    return [array get NEW]
$$ LANGUAGE pltcl;

CREATE TABLE mytab (num integer, description text, modcnt integer);

CREATE TRIGGER trig_mytab_modcount BEFORE INSERT OR UPDATE ON mytab
    FOR EACH ROW EXECUTE FUNCTION trigfunc_modcount('modcnt');
```

Beachten Sie, dass die Trigger-Funktion selbst den Spaltennamen nicht kennt; dieser wird über die Trigger-Argumente geliefert. Dadurch kann die Trigger-Funktion mit verschiedenen Tabellen wiederverwendet werden.

## 42.7. Event-Trigger-Funktionen in PL/Tcl

Event-Trigger-Funktionen können in PL/Tcl geschrieben werden. PostgreSQL verlangt, dass eine Funktion, die als Event-Trigger aufgerufen werden soll, als Funktion ohne Argumente und mit dem Rückgabetyp `event_trigger` deklariert wird.

Die Informationen vom Trigger-Manager werden dem Funktionsrumpf in den folgenden Variablen übergeben:

- `$TG_event`
  - Der Name des Ereignisses, für das der Trigger ausgelöst wird.

- `$TG_tag`
  - Der Command Tag, für den der Trigger ausgelöst wird.

Der Rückgabewert der Trigger-Funktion wird ignoriert.

Hier ist eine kleine Beispiel-Event-Trigger-Funktion, die einfach jedes Mal eine `NOTICE`-Meldung ausgibt, wenn ein unterstützter Befehl ausgeführt wird:

```sql
CREATE OR REPLACE FUNCTION tclsnitch() RETURNS event_trigger AS $$
  elog NOTICE "tclsnitch: $TG_event $TG_tag"
$$ LANGUAGE pltcl;

CREATE EVENT TRIGGER tcl_a_snitch ON ddl_command_start EXECUTE
 FUNCTION tclsnitch();
```

## 42.8. Fehlerbehandlung in PL/Tcl

Tcl-Code innerhalb einer PL/Tcl-Funktion oder Code, der aus einer PL/Tcl-Funktion heraus aufgerufen wird, kann einen Fehler auslösen, entweder indem er eine ungültige Operation ausführt oder indem er mit dem Tcl-Befehl `error` oder mit PL/Tcls Befehl `elog` einen Fehler erzeugt. Solche Fehler können innerhalb von Tcl mit dem Tcl-Befehl `catch` abgefangen werden. Wenn ein Fehler nicht abgefangen wird, sondern bis zur obersten Ausführungsebene der PL/Tcl-Funktion weitergegeben wird, wird er in der aufrufenden Abfrage der Funktion als SQL-Fehler gemeldet.

Umgekehrt werden SQL-Fehler, die innerhalb der PL/Tcl-Befehle `spi_exec`, `spi_prepare` und `spi_execp` auftreten, als Tcl-Fehler gemeldet und können daher mit Tcls Befehl `catch` abgefangen werden. (Jeder dieser PL/Tcl-Befehle führt seine SQL-Operation in einer Subtransaktion aus, die bei einem Fehler zurückgerollt wird, sodass jede teilweise abgeschlossene Operation automatisch bereinigt wird.) Auch hier gilt: Wenn sich ein Fehler ohne Abfangen bis zur obersten Ebene ausbreitet, wird er wieder zu einem SQL-Fehler.

Tcl stellt eine Variable `errorCode` bereit, die zusätzliche Informationen über einen Fehler in einer Form darstellen kann, die für Tcl-Programme leicht zu interpretieren ist. Der Inhalt liegt im Tcl-Listenformat vor, und das erste Wort identifiziert das Subsystem oder die Bibliothek, die den Fehler meldet; darüber hinaus bleibt der Inhalt dem jeweiligen Subsystem oder der jeweiligen Bibliothek überlassen. Bei Datenbankfehlern, die von PL/Tcl-Befehlen gemeldet werden, ist das erste Wort `POSTGRES`, das zweite Wort ist die PostgreSQL-Versionsnummer, und zusätzliche Wörter sind Feldname/Wert-Paare, die detaillierte Informationen über den Fehler liefern. Die Felder `SQLSTATE`, `condition` und `message` werden immer geliefert (die ersten beiden stellen den Fehlercode und den Bedingungsnamen dar, wie in [Anhang A](A_PostgreSQL_Fehlercodes.md) gezeigt). Felder, die vorhanden sein können, sind `detail`, `hint`, `context`, `schema`, `table`, `column`, `datatype`, `constraint`, `statement`, `cursor_position`, `filename`, `lineno` und `funcname`.

Eine bequeme Möglichkeit, mit PL/Tcls `errorCode`-Informationen zu arbeiten, besteht darin, sie in ein Array zu laden, sodass die Feldnamen zu Array-Subskripten werden. Code dafür könnte so aussehen:

```sql
if {[catch { spi_exec $sql_command }]} {
    if {[lindex $::errorCode 0] == "POSTGRES"} {
        array set errorArray $::errorCode
        if {$errorArray(condition) == "undefined_table"} {
            # deal with missing table
        } else {
            # deal with some other type of SQL error
        }
    }
}
```

(Die doppelten Doppelpunkte geben ausdrücklich an, dass `errorCode` eine globale Variable ist.)

## 42.9. Explizite Subtransaktionen in PL/Tcl

Das Wiederherstellen nach Fehlern, die durch Datenbankzugriff verursacht wurden, wie in [Abschnitt 42.8](#428-fehlerbehandlung-in-pltcl) beschrieben, kann zu einer unerwünschten Situation führen, in der einige Operationen erfolgreich sind, bevor eine von ihnen fehlschlägt, und die Daten nach der Wiederherstellung von diesem Fehler in einem inkonsistenten Zustand zurückbleiben. PL/Tcl bietet für dieses Problem eine Lösung in Form expliziter Subtransaktionen.

Betrachten Sie eine Funktion, die eine Überweisung zwischen zwei Konten implementiert:

```sql
CREATE FUNCTION transfer_funds() RETURNS void AS $$
    if [catch {
        spi_exec "UPDATE accounts SET balance = balance - 100 WHERE
 account_name = 'joe'"
        spi_exec "UPDATE accounts SET balance = balance + 100 WHERE
 account_name = 'mary'"
    } errormsg] {
        set result [format "error transferring funds: %s"
 $errormsg]
    } else {
        set result "funds transferred successfully"
    }
    spi_exec "INSERT INTO operations (result) VALUES ('[quote
 $result]')"
$$ LANGUAGE pltcl;
```

Wenn die zweite `UPDATE`-Anweisung dazu führt, dass eine Exception ausgelöst wird, protokolliert diese Funktion den Fehlschlag, aber das Ergebnis der ersten `UPDATE`-Anweisung wird dennoch committet. Mit anderen Worten: Das Geld wird von Joes Konto abgebucht, aber nicht auf Marys Konto übertragen. Das geschieht, weil jedes `spi_exec` eine separate Subtransaktion ist und nur eine dieser Subtransaktionen zurückgerollt wurde.

Um solche Fälle zu behandeln, können Sie mehrere Datenbankoperationen in eine explizite Subtransaktion einschließen, die als Ganzes erfolgreich ist oder zurückgerollt wird. PL/Tcl stellt dafür den Befehl `subtransaction` bereit. Wir können unsere Funktion so umschreiben:

```sql
CREATE FUNCTION transfer_funds2() RETURNS void AS $$
    if [catch {
        subtransaction {
             spi_exec "UPDATE accounts SET balance = balance - 100
 WHERE account_name = 'joe'"
             spi_exec "UPDATE accounts SET balance = balance + 100
 WHERE account_name = 'mary'"
        }
    } errormsg] {
        set result [format "error transferring funds: %s"
 $errormsg]
    } else {
        set result "funds transferred successfully"
    }
    spi_exec "INSERT INTO operations (result) VALUES ('[quote
 $result]')"
$$ LANGUAGE pltcl;
```

Beachten Sie, dass die Verwendung von `catch` für diesen Zweck weiterhin erforderlich ist. Andernfalls würde sich der Fehler bis zur obersten Ebene der Funktion ausbreiten und das gewünschte Einfügen in die Tabelle `operations` verhindern. Der Befehl `subtransaction` fängt keine Fehler ab; er stellt nur sicher, dass alle Datenbankoperationen, die in seinem Geltungsbereich ausgeführt werden, gemeinsam zurückgerollt werden, wenn ein Fehler gemeldet wird.

Ein Rollback einer expliziten Subtransaktion erfolgt bei jedem Fehler, der vom enthaltenen Tcl-Code gemeldet wird, nicht nur bei Fehlern, die aus dem Datenbankzugriff stammen. Daher führt auch eine reguläre Tcl-Exception, die innerhalb eines `subtransaction`-Befehls ausgelöst wird, zum Rollback der Subtransaktion. Nichtfehlerhafte Ausstiege aus dem enthaltenen Tcl-Code (zum Beispiel durch `return`) verursachen jedoch keinen Rollback.

## 42.10. Transaktionsverwaltung

In einer Prozedur, die von der obersten Ebene aufgerufen wird, oder in einem anonymen Codeblock (Befehl `DO`), der von der obersten Ebene aufgerufen wird, ist es möglich, Transaktionen zu steuern. Um die aktuelle Transaktion zu committen, rufen Sie den Befehl `commit` auf. Um die aktuelle Transaktion zurückzurollen, rufen Sie den Befehl `rollback` auf. (Beachten Sie, dass es nicht möglich ist, die SQL-Befehle `COMMIT` oder `ROLLBACK` über `spi_exec` oder Ähnliches auszuführen. Dies muss mit diesen Funktionen geschehen.) Nachdem eine Transaktion beendet wurde, wird automatisch eine neue Transaktion gestartet, sodass es dafür keinen separaten Befehl gibt.

Hier ist ein Beispiel:

```sql
CREATE PROCEDURE transaction_test1()
LANGUAGE pltcl
AS $$
for {set i 0} {$i < 10} {incr i} {
    spi_exec "INSERT INTO test1 (a) VALUES ($i)"
    if {$i % 2 == 0} {
        commit
    } else {
        rollback
    }
}
$$;

CALL transaction_test1();
```

Transaktionen können nicht beendet werden, solange eine explizite Subtransaktion aktiv ist.

## 42.11. PL/Tcl-Konfiguration

Dieser Abschnitt listet Konfigurationsparameter auf, die PL/Tcl betreffen.

- `pltcl.start_proc` (`string`)
  - Wenn dieser Parameter auf eine nicht leere Zeichenkette gesetzt ist, gibt er den Namen (möglicherweise schemaqualifiziert) einer parameterlosen PL/Tcl-Funktion an, die immer dann ausgeführt wird, wenn ein neuer Tcl-Interpreter für PL/Tcl erzeugt wird. Eine solche Funktion kann eine sitzungsbezogene Initialisierung durchführen, zum Beispiel zusätzlichen Tcl-Code laden. Ein neuer Tcl-Interpreter wird erzeugt, wenn eine PL/Tcl-Funktion erstmals in einer Datenbanksitzung ausgeführt wird oder wenn ein zusätzlicher Interpreter erzeugt werden muss, weil eine PL/Tcl-Funktion von einer neuen SQL-Rolle aufgerufen wird.

    Die referenzierte Funktion muss in der Sprache `pltcl` geschrieben sein und darf nicht als `SECURITY DEFINER` markiert sein. (Diese Einschränkungen stellen sicher, dass sie in dem Interpreter läuft, den sie initialisieren soll.) Der aktuelle Benutzer muss außerdem die Berechtigung haben, sie aufzurufen.

    Wenn die Funktion mit einem Fehler fehlschlägt, bricht sie den Funktionsaufruf ab, der die Erzeugung des neuen Interpreters verursacht hat, und breitet sich bis zur aufrufenden Abfrage aus, wodurch die aktuelle Transaktion oder Subtransaktion abgebrochen wird. Aktionen, die innerhalb von Tcl bereits ausgeführt wurden, werden nicht rückgängig gemacht; dieser Interpreter wird jedoch nicht erneut verwendet. Wenn die Sprache wieder verwendet wird, wird die Initialisierung in einem frischen Tcl-Interpreter erneut versucht.

    Nur Superuser können diese Einstellung ändern. Obwohl diese Einstellung innerhalb einer Sitzung geändert werden kann, wirken sich solche Änderungen nicht auf Tcl-Interpreter aus, die bereits erzeugt wurden.

- `pltclu.start_proc` (`string`)
  - Dieser Parameter ist genau wie `pltcl.start_proc`, außer dass er für PL/TclU gilt. Die referenzierte Funktion muss in der Sprache `pltclu` geschrieben sein.

## 42.12. Namen von Tcl-Prozeduren

In PostgreSQL kann derselbe Funktionsname für verschiedene Funktionsdefinitionen verwendet werden, wenn die Funktionen in verschiedenen Schemas liegen oder wenn sich die Anzahl der Argumente oder ihre Typen unterscheiden. Tcl verlangt jedoch, dass alle Prozedurnamen eindeutig sind. PL/Tcl behandelt dies, indem es die Argumenttypnamen in den internen Tcl-Prozedurnamen aufnimmt und dann bei Bedarf die Objekt-ID (OID) der Funktion an den internen Tcl-Prozedurnamen anhängt, um ihn von den Namen aller zuvor geladenen Funktionen im selben Tcl-Interpreter zu unterscheiden. PostgreSQL-Funktionen mit demselben Namen und unterschiedlichen Argumenttypen werden daher auch unterschiedliche Tcl-Prozeduren sein. Für einen PL/Tcl-Programmierer ist das normalerweise kein Problem, beim Debugging kann es aber sichtbar werden.

Aus diesem und anderen Gründen kann eine PL/Tcl-Funktion eine andere nicht direkt aufrufen (das heißt innerhalb von Tcl). Wenn Sie das tun müssen, müssen Sie über SQL gehen und `spi_exec` oder einen verwandten Befehl verwenden.
