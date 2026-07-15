# 34. ECPG - Embedded SQL in C

Dieses Kapitel beschreibt das Embedded-SQL-Paket für PostgreSQL. Es wurde von Linus Tolke (<linus@epact.se>) und Michael Meskes (<meskes@postgresql.org>) geschrieben. Ursprünglich war es für C gedacht. Es funktioniert auch mit C++, erkennt aber noch nicht alle C++-Konstrukte.

Diese Dokumentation ist nicht vollständig. Da diese Schnittstelle jedoch standardisiert ist, finden sich zusätzliche Informationen in vielen Ressourcen zu SQL.

## 34.1. Das Konzept

Ein Embedded-SQL-Programm besteht aus Code in einer normalen Programmiersprache, hier C, der mit SQL-Befehlen in besonders markierten Abschnitten gemischt wird. Zum Erstellen des Programms wird der Quelltext (`*.pgc`) zuerst durch den Embedded-SQL-Präprozessor geleitet. Dieser wandelt ihn in ein gewöhnliches C-Programm (`*.c`) um, das anschließend von einem C-Compiler verarbeitet werden kann. Details zum Kompilieren und Linken stehen in [Abschnitt 34.10](34_ECPG_Embedded_SQL_in_C.md#3410-embeddedsqlprogramme-verarbeiten). Umgewandelte ECPG-Anwendungen rufen über die Embedded-SQL-Bibliothek (`ecpglib`) Funktionen der Bibliothek `libpq` auf und kommunizieren mit dem PostgreSQL-Server über das normale Frontend/Backend-Protokoll.

Embedded SQL hat gegenüber anderen Methoden zum Ausführen von SQL-Befehlen aus C-Code mehrere Vorteile. Erstens übernimmt es die mühsame Übergabe von Informationen an Variablen im C-Programm und zurück. Zweitens wird der SQL-Code im Programm bereits beim Build auf syntaktische Korrektheit geprüft. Drittens ist Embedded SQL in C im SQL-Standard spezifiziert und wird von vielen anderen SQL-Datenbanksystemen unterstützt. Die PostgreSQL-Implementierung ist so weit wie möglich an diesen Standard angelehnt, sodass Embedded-SQL-Programme anderer SQL-Datenbanken meist vergleichsweise leicht nach PostgreSQL portiert werden können.

Programme für die Embedded-SQL-Schnittstelle sind normale C-Programme, in die spezieller Code für datenbankbezogene Aktionen eingefügt wird. Dieser spezielle Code hat immer die Form:

```sql
EXEC SQL ...;
```

Solche Anweisungen nehmen syntaktisch die Stelle einer C-Anweisung ein. Je nach konkreter Anweisung können sie auf globaler Ebene oder innerhalb einer Funktion stehen.

Embedded-SQL-Anweisungen folgen den Groß-/Kleinschreibungsregeln von normalem SQL-Code, nicht denen von C. Außerdem erlauben sie gemäß SQL-Standard verschachtelte Kommentare im C-Stil. Der C-Teil des Programms folgt dagegen dem C-Standard und akzeptiert keine verschachtelten Kommentare. Embedded-SQL-Anweisungen verwenden auch beim Parsen von Zeichenketten und Bezeichnern SQL-Regeln, nicht C-Regeln; siehe [Abschnitt 4.1.2.1](04_SQL_Syntax.md#4121-zeichenkettenkonstanten) und [Abschnitt 4.1.1](04_SQL_Syntax.md#411-bezeichner-und-schlüsselwörter). ECPG nimmt an, dass `standard_conforming_strings` eingeschaltet ist. Der C-Teil des Programms verwendet natürlich weiterhin die C-Regeln für Quoting.

Die folgenden Abschnitte erklären die Embedded-SQL-Anweisungen.

## 34.2. Datenbankverbindungen verwalten

Dieser Abschnitt beschreibt, wie Datenbankverbindungen geöffnet, geschlossen und gewechselt werden.

### 34.2.1. Verbindung zum Datenbankserver herstellen

Eine Verbindung zu einer Datenbank wird mit folgender Anweisung aufgebaut:

```sql
EXEC SQL CONNECT TO target [AS connection-name] [USER user-name];
```

Das Ziel `target` kann auf folgende Arten angegeben werden:

- `dbname[@hostname][:port]`
- `tcp:postgresql://hostname[:port][/dbname][?options]`
- `unix:postgresql://localhost[:port][/dbname][?options]`
- ein SQL-Stringliteral, das eine der obigen Formen enthält
- ein Verweis auf eine Zeichenvariable, die eine der obigen Formen enthält
- `DEFAULT`

Das Verbindungsziel `DEFAULT` baut eine Verbindung zur Standarddatenbank unter dem Standardbenutzernamen auf. In diesem Fall kann kein separater Benutzername und kein Verbindungsname angegeben werden.

Wenn das Verbindungsziel direkt angegeben wird, also nicht als Stringliteral oder Variablenreferenz, werden seine Bestandteile durch den normalen SQL-Parser verarbeitet. Das bedeutet zum Beispiel, dass der Hostname wie ein oder mehrere durch Punkte getrennte SQL-Bezeichner aussehen muss und diese Bezeichner ohne doppelte Anführungszeichen in Kleinschreibung gefaltet werden. Optionswerte müssen SQL-Bezeichner, Ganzzahlen oder Variablenreferenzen sein. In der Praxis ist es meist weniger fehleranfällig, ein einfach gequotetes Stringliteral oder eine Variablenreferenz zu verwenden, statt das Verbindungsziel direkt zu schreiben.

Auch der Benutzername kann auf verschiedene Arten angegeben werden:

- `username`
- `username/password`
- `username IDENTIFIED BY password`
- `username USING password`

Wie oben können `username` und `password` SQL-Bezeichner, SQL-Stringliterale oder Verweise auf Zeichenvariablen sein.

Enthält das Verbindungsziel Optionen, bestehen diese aus `keyword=value`-Angaben, die durch kaufmännische Und-Zeichen (`&`) getrennt sind. Die zulässigen Schlüsselwörter sind dieselben, die auch `libpq` erkennt; siehe [Abschnitt 32.1.2](32_libpq_C_Bibliothek.md#3212-parameterschlüsselwörter). Leerzeichen vor Schlüsselwort oder Wert werden ignoriert, nicht jedoch innerhalb oder nach ihnen. Innerhalb eines Wertes kann kein `&` geschrieben werden.

Bei Socket-Verbindungen mit dem Präfix `unix:` muss der Hostname genau `localhost` sein. Um ein anderes Socket-Verzeichnis auszuwählen, schreiben Sie dessen Pfadnamen als Wert einer `host`-Option im Optionsteil des Ziels.

Der `connection-name` dient dazu, mehrere Verbindungen in einem Programm zu verwalten. Er kann ausgelassen werden, wenn ein Programm nur eine Verbindung verwendet. Die zuletzt geöffnete Verbindung wird zur aktuellen Verbindung; sie wird standardmäßig verwendet, wenn eine SQL-Anweisung ausgeführt werden soll.

Beispiele für `CONNECT`-Anweisungen:

```sql
EXEC SQL CONNECT TO mydb@sql.mydomain.com;

EXEC SQL CONNECT TO tcp:postgresql://sql.mydomain.com/mydb AS myconnection USER john;
```

```c
EXEC SQL BEGIN DECLARE SECTION;
const char *target = "mydb@sql.mydomain.com";
const char *user = "john";
const char *passwd = "secret";
EXEC SQL END DECLARE SECTION;

...

EXEC SQL CONNECT TO :target USER :user USING :passwd;
/* oder EXEC SQL CONNECT TO :target USER :user/:passwd; */
```

Das letzte Beispiel verwendet die oben erwähnten Verweise auf Zeichenvariablen. In späteren Abschnitten wird gezeigt, wie C-Variablen in SQL-Anweisungen verwendet werden, indem ihnen ein Doppelpunkt vorangestellt wird.

Das Format des Verbindungsziels ist nicht im SQL-Standard festgelegt. Wer portable Anwendungen entwickeln möchte, sollte daher eine Form wie im letzten Beispiel verwenden und den Verbindungsziel-String an einer geeigneten Stelle kapseln.

Wenn nicht vertrauenswürdige Benutzer Zugriff auf eine Datenbank haben, die kein sicheres Schema-Nutzungsschema verwendet, sollte jede Sitzung damit beginnen, öffentlich beschreibbare Schemas aus `search_path` zu entfernen. Fügen Sie zum Beispiel `options=-c search_path=` zu den Optionen hinzu oder führen Sie nach dem Verbinden `EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);` aus. Diese Überlegung ist nicht ECPG-spezifisch, sondern gilt für jede Schnittstelle, die beliebige SQL-Befehle ausführt.

### 34.2.2. Eine Verbindung auswählen

SQL-Anweisungen in Embedded-SQL-Programmen werden standardmäßig auf der aktuellen Verbindung ausgeführt, also auf der zuletzt geöffneten Verbindung. Wenn eine Anwendung mehrere Verbindungen verwalten muss, gibt es drei Möglichkeiten.

Die erste Möglichkeit ist, für jede SQL-Anweisung ausdrücklich eine Verbindung auszuwählen:

```sql
EXEC SQL AT connection-name SELECT ...;
```

Diese Variante eignet sich besonders, wenn eine Anwendung mehrere Verbindungen in gemischter Reihenfolge verwenden muss.

Wenn eine Anwendung mehrere Ausführungs-Threads verwendet, dürfen diese eine Verbindung nicht gleichzeitig gemeinsam nutzen. Der Zugriff auf die Verbindung muss entweder ausdrücklich kontrolliert werden, etwa mit Mutexen, oder jeder Thread verwendet eine eigene Verbindung.

Die zweite Möglichkeit ist, die aktuelle Verbindung mit einer Anweisung zu wechseln:

```sql
EXEC SQL SET CONNECTION connection-name;
```

Diese Variante ist besonders praktisch, wenn viele Anweisungen auf derselben Verbindung ausgeführt werden sollen.

Beispielprogramm für mehrere Datenbankverbindungen:

```c
#include <stdio.h>

EXEC SQL BEGIN DECLARE SECTION;
char dbname[1024];
EXEC SQL END DECLARE SECTION;

int
main()
{
    EXEC SQL CONNECT TO testdb1 AS con1 USER testuser;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    EXEC SQL CONNECT TO testdb2 AS con2 USER testuser;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    EXEC SQL CONNECT TO testdb3 AS con3 USER testuser;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    /* Diese Abfrage würde in der zuletzt geöffneten Datenbank "testdb3" laufen. */
    EXEC SQL SELECT current_database() INTO :dbname;
    printf("current=%s (should be testdb3)\n", dbname);

    /* Mit "AT" eine Abfrage in "testdb2" ausführen. */
    EXEC SQL AT con2 SELECT current_database() INTO :dbname;
    printf("current=%s (should be testdb2)\n", dbname);

    /* Die aktuelle Verbindung auf "testdb1" wechseln. */
    EXEC SQL SET CONNECTION con1;

    EXEC SQL SELECT current_database() INTO :dbname;
    printf("current=%s (should be testdb1)\n", dbname);

    EXEC SQL DISCONNECT ALL;
    return 0;
}
```

Dieses Beispiel erzeugt folgende Ausgabe:

```text
current=testdb3 (should be testdb3)
current=testdb2 (should be testdb2)
current=testdb1 (should be testdb1)
```

Die dritte Möglichkeit ist, einen SQL-Bezeichner zu deklarieren, der mit der Verbindung verknüpft ist:

```sql
EXEC SQL AT connection-name DECLARE statement-name STATEMENT;
EXEC SQL PREPARE statement-name FROM :dyn-string;
```

Sobald ein SQL-Bezeichner mit einer Verbindung verknüpft ist, kann dynamisches SQL ohne `AT`-Klausel ausgeführt werden. Diese Option verhält sich wie eine Präprozessordirektive; die Verknüpfung gilt daher nur in der Datei.

Beispiel:

```c
#include <stdio.h>

EXEC SQL BEGIN DECLARE SECTION;
char dbname[128];
char *dyn_sql = "SELECT current_database()";
EXEC SQL END DECLARE SECTION;

int
main()
{
    EXEC SQL CONNECT TO postgres AS con1;
    EXEC SQL CONNECT TO testdb AS con2;
    EXEC SQL AT con1 DECLARE stmt STATEMENT;
    EXEC SQL PREPARE stmt FROM :dyn_sql;
    EXEC SQL EXECUTE stmt INTO :dbname;
    printf("%s\n", dbname);

    EXEC SQL DISCONNECT ALL;
    return 0;
}
```

Dieses Beispiel erzeugt auch dann folgende Ausgabe, wenn die Standardverbindung `testdb` ist:

```text
postgres
```

### 34.2.3. Eine Verbindung schließen

Zum Schließen einer Verbindung verwenden Sie:

```sql
EXEC SQL DISCONNECT [connection];
```

Die Verbindung kann auf folgende Arten angegeben werden:

- `connection-name`
- `CURRENT`
- `ALL`

Wenn kein Verbindungsname angegeben wird, wird die aktuelle Verbindung geschlossen. Es ist guter Stil, dass eine Anwendung jede von ihr geöffnete Verbindung ausdrücklich wieder trennt.

## 34.3. SQL-Befehle ausführen

Jeder SQL-Befehl kann aus einer Embedded-SQL-Anwendung heraus ausgeführt werden. Die folgenden Beispiele zeigen die grundlegende Verwendung.

### 34.3.1. SQL-Anweisungen ausführen

Eine Tabelle erzeugen:

```sql
EXEC SQL CREATE TABLE foo (number integer, ascii char(16));
EXEC SQL CREATE UNIQUE INDEX num1 ON foo(number);
EXEC SQL COMMIT;
```

Zeilen einfügen:

```sql
EXEC SQL INSERT INTO foo (number, ascii) VALUES (9999, 'doodad');
EXEC SQL COMMIT;
```

Zeilen löschen:

```sql
EXEC SQL DELETE FROM foo WHERE number = 9999;
EXEC SQL COMMIT;
```

Aktualisierungen:

```sql
EXEC SQL UPDATE foo
    SET ascii = 'foobar'
    WHERE number = 9999;
EXEC SQL COMMIT;
```

`SELECT`-Anweisungen, die genau eine Ergebniszeile zurückgeben, können ebenfalls direkt mit `EXEC SQL` ausgeführt werden. Für Ergebnismengen mit mehreren Zeilen muss eine Anwendung einen Cursor verwenden; siehe [Abschnitt 34.3.2](34_ECPG_Embedded_SQL_in_C.md#3432-cursor-verwenden). Als Spezialfall kann eine Anwendung mehrere Zeilen auf einmal in eine Array-Hostvariable lesen; siehe [Abschnitt 34.4.4.3.1](34_ECPG_Embedded_SQL_in_C.md#344431-arrays).

Einzeiliges `SELECT`:

```sql
EXEC SQL SELECT foo INTO :FooBar FROM table1 WHERE ascii = 'doodad';
```

Auch ein Konfigurationsparameter kann mit `SHOW` abgerufen werden:

```sql
EXEC SQL SHOW search_path INTO :var;
```

Token der Form `:something` sind Hostvariablen, das heißt, sie verweisen auf Variablen im C-Programm. Sie werden in [Abschnitt 34.4](34_ECPG_Embedded_SQL_in_C.md#344-hostvariablen-verwenden) erklärt.

### 34.3.2. Cursor verwenden

Um eine Ergebnismenge mit mehreren Zeilen abzurufen, muss eine Anwendung einen Cursor deklarieren und die Zeilen aus dem Cursor holen. Die Schritte sind: Cursor deklarieren, öffnen, eine Zeile abrufen, wiederholen und schließlich schließen.

`SELECT` mit Cursor:

```sql
EXEC SQL DECLARE foo_bar CURSOR FOR
    SELECT number, ascii FROM foo
    ORDER BY ascii;
EXEC SQL OPEN foo_bar;
EXEC SQL FETCH foo_bar INTO :FooBar, DooDad;
...
EXEC SQL CLOSE foo_bar;
EXEC SQL COMMIT;
```

Weitere Details zum Deklarieren eines Cursors stehen bei `DECLARE`, weitere Details zum Abrufen von Zeilen aus einem Cursor bei `FETCH`.

> **Hinweis**
>
> Der ECPG-Befehl `DECLARE` sendet nicht tatsächlich eine Anweisung an das PostgreSQL-Backend. Der Cursor wird im Backend erst an der Stelle geöffnet, an der der Befehl `OPEN` ausgeführt wird; dabei wird der backendseitige Befehl `DECLARE` verwendet.

### 34.3.3. Transaktionen verwalten

Im Standardmodus werden Anweisungen erst festgeschrieben, wenn `EXEC SQL COMMIT` ausgeführt wird. Die Embedded-SQL-Schnittstelle unterstützt außerdem Autocommit für Transaktionen, ähnlich dem Standardverhalten von `psql`. Aktiviert wird es über die Befehlszeilenoption `-t` von `ecpg` oder über die Anweisung `EXEC SQL SET AUTOCOMMIT TO ON`. Im Autocommit-Modus wird jeder Befehl automatisch festgeschrieben, sofern er sich nicht in einem expliziten Transaktionsblock befindet. Der Modus kann mit `EXEC SQL SET AUTOCOMMIT TO OFF` wieder ausgeschaltet werden.

Folgende Befehle zur Transaktionsverwaltung sind verfügbar:

| Befehl | Bedeutung |
| --- | --- |
| `EXEC SQL COMMIT` | Schreibt eine laufende Transaktion fest. |
| `EXEC SQL ROLLBACK` | Macht eine laufende Transaktion rückgängig. |
| `EXEC SQL PREPARE TRANSACTION transaction_id` | Bereitet die aktuelle Transaktion für Two-Phase Commit vor. |
| `EXEC SQL COMMIT PREPARED transaction_id` | Schreibt eine vorbereitete Transaktion fest. |
| `EXEC SQL ROLLBACK PREPARED transaction_id` | Macht eine vorbereitete Transaktion rückgängig. |
| `EXEC SQL SET AUTOCOMMIT TO ON` | Aktiviert den Autocommit-Modus. |
| `EXEC SQL SET AUTOCOMMIT TO OFF` | Deaktiviert den Autocommit-Modus. Dies ist der Standard. |

### 34.3.4. Vorbereitete Anweisungen

Wenn die Werte, die an eine SQL-Anweisung übergeben werden sollen, zur Compilezeit noch nicht bekannt sind oder dieselbe Anweisung häufig verwendet werden soll, können vorbereitete Anweisungen nützlich sein.

Die Anweisung wird mit dem Befehl `PREPARE` vorbereitet. Für noch unbekannte Werte wird der Platzhalter `?` verwendet:

```sql
EXEC SQL PREPARE stmt1 FROM "SELECT oid, datname FROM pg_database WHERE oid = ?";
```

Wenn eine Anweisung eine einzelne Zeile zurückgibt, kann die Anwendung nach `PREPARE` `EXECUTE` aufrufen und die tatsächlichen Werte für die Platzhalter mit einer `USING`-Klausel übergeben:

```sql
EXEC SQL EXECUTE stmt1 INTO :dboid, :dbname USING 1;
```

Wenn eine Anweisung mehrere Zeilen zurückgibt, kann die Anwendung einen Cursor verwenden, der auf der vorbereiteten Anweisung basiert. Um Eingabeparameter zu binden, muss der Cursor mit einer `USING`-Klausel geöffnet werden:

```c
EXEC SQL PREPARE stmt1 FROM "SELECT oid, datname FROM pg_database WHERE oid > ?";
EXEC SQL DECLARE foo_bar CURSOR FOR stmt1;

/* Wenn das Ende der Ergebnismenge erreicht ist, die while-Schleife verlassen. */
EXEC SQL WHENEVER NOT FOUND DO BREAK;

EXEC SQL OPEN foo_bar USING 100;
...
while (1)
{
    EXEC SQL FETCH NEXT FROM foo_bar INTO :dboid, :dbname;
    ...
}
EXEC SQL CLOSE foo_bar;
```

Wenn die vorbereitete Anweisung nicht mehr benötigt wird, sollte sie freigegeben werden:

```sql
EXEC SQL DEALLOCATE PREPARE name;
```

Weitere Details zu `PREPARE` stehen bei `PREPARE`. Siehe auch [Abschnitt 34.5](34_ECPG_Embedded_SQL_in_C.md#345-dynamisches-sql) für weitere Informationen zur Verwendung von Platzhaltern und Eingabeparametern.

## 34.4. Hostvariablen verwenden

In [Abschnitt 34.3](34_ECPG_Embedded_SQL_in_C.md#343-sqlbefehle-ausführen) wurde gezeigt, wie SQL-Anweisungen aus einem Embedded-SQL-Programm heraus ausgeführt werden können. Einige dieser Anweisungen verwendeten nur feste Werte und boten keine Möglichkeit, vom Benutzer gelieferte Werte in Anweisungen einzusetzen oder vom Programm die von der Abfrage zurückgegebenen Werte zu verarbeiten. Solche Anweisungen sind in echten Anwendungen nur begrenzt nützlich. Dieser Abschnitt erklärt, wie Daten über einen einfachen Mechanismus namens Hostvariablen zwischen dem C-Programm und den Embedded-SQL-Anweisungen ausgetauscht werden.

In einem Embedded-SQL-Programm gelten die SQL-Anweisungen als Gäste im C-Programmcode, der Hostsprache. Deshalb werden die Variablen des C-Programms Hostvariablen genannt.

Eine weitere Möglichkeit, Werte zwischen PostgreSQL-Backends und ECPG-Anwendungen auszutauschen, sind SQL-Deskriptoren; siehe [Abschnitt 34.7](34_ECPG_Embedded_SQL_in_C.md#347-descriptorbereiche-verwenden).

### 34.4.1. Überblick

Die Übergabe von Daten zwischen C-Programm und SQL-Anweisungen ist in Embedded SQL besonders einfach. Statt Daten in eine Anweisung einzubauen, was verschiedene Probleme wie korrektes Quoting mit sich bringt, kann der Name einer C-Variablen direkt in die SQL-Anweisung geschrieben werden, mit einem vorangestellten Doppelpunkt. Beispiel:

```sql
EXEC SQL INSERT INTO sometable VALUES (:v1, 'foo', :v2);
```

Diese Anweisung verweist auf zwei C-Variablen namens `v1` und `v2` und verwendet zusätzlich ein normales SQL-Stringliteral. Das zeigt, dass Sie nicht auf eine der beiden Formen beschränkt sind.

Diese Art, C-Variablen in SQL-Anweisungen einzufügen, funktioniert überall dort, wo in einer SQL-Anweisung ein Wertausdruck erwartet wird.

### 34.4.2. Deklarationsabschnitte

Um Daten aus dem Programm an die Datenbank zu übergeben, etwa als Parameter einer Abfrage, oder um Daten aus der Datenbank zurück ins Programm zu übernehmen, müssen die C-Variablen für diese Daten in besonders markierten Abschnitten deklariert werden. Dadurch kennt der Embedded-SQL-Präprozessor diese Variablen.

Ein solcher Abschnitt beginnt mit:

```sql
EXEC SQL BEGIN DECLARE SECTION;
```

und endet mit:

```sql
EXEC SQL END DECLARE SECTION;
```

Dazwischen stehen normale C-Variablendeklarationen, zum Beispiel:

```c
int      x = 4;
char     foo[16], bar[16];
```

Wie zu sehen ist, kann einer Variablen optional ein Anfangswert zugewiesen werden. Der Gültigkeitsbereich der Variablen wird durch die Position ihres Deklarationsabschnitts im Programm bestimmt. Variablen können auch mit folgender Syntax deklariert werden, die implizit einen Deklarationsabschnitt erzeugt:

```sql
EXEC SQL int i = 4;
```

Ein Programm kann beliebig viele Deklarationsabschnitte enthalten. Die Deklarationen werden außerdem als normale C-Variablen in die Ausgabedatei übernommen, sodass sie nicht noch einmal deklariert werden müssen. Variablen, die nicht in SQL-Befehlen verwendet werden sollen, können wie gewohnt außerhalb dieser speziellen Abschnitte deklariert werden.

Auch die Definition einer Struktur oder Union muss innerhalb eines `DECLARE`-Abschnitts stehen. Andernfalls kann der Präprozessor diese Typen nicht verarbeiten, weil er ihre Definition nicht kennt.

### 34.4.3. Abfrageergebnisse abrufen

Damit können Daten, die das Programm erzeugt, an SQL-Befehle übergeben werden. Umgekehrt stellt sich die Frage, wie Ergebnisse einer Abfrage abgerufen werden. Dafür bietet Embedded SQL spezielle Varianten der üblichen Befehle `SELECT` und `FETCH`. Diese Befehle haben eine besondere `INTO`-Klausel, die angibt, in welchen Hostvariablen die abgerufenen Werte gespeichert werden sollen. `SELECT` wird für Abfragen verwendet, die nur eine einzelne Zeile zurückgeben, `FETCH` für Abfragen mit mehreren Zeilen über einen Cursor.

Beispiel:

```c
/*
 * Angenommen, es gibt diese Tabelle:
 * CREATE TABLE test1 (a int, b varchar(50));
 */

EXEC SQL BEGIN DECLARE SECTION;
int v1;
VARCHAR v2;
EXEC SQL END DECLARE SECTION;

...

EXEC SQL SELECT a, b INTO :v1, :v2 FROM test;
```

Die `INTO`-Klausel steht also zwischen Auswahlliste und `FROM`-Klausel. Die Anzahl der Elemente in der Auswahlliste und in der Liste nach `INTO`, auch Zielliste genannt, muss übereinstimmen.

Beispiel mit `FETCH`:

```c
EXEC SQL BEGIN DECLARE SECTION;
int v1;
VARCHAR v2;
EXEC SQL END DECLARE SECTION;

...

EXEC SQL DECLARE foo CURSOR FOR SELECT a, b FROM test;

...

do
{
    ...
    EXEC SQL FETCH NEXT FROM foo INTO :v1, :v2;
    ...
} while (...);
```

Hier steht die `INTO`-Klausel nach allen normalen Klauseln.

### 34.4.4. Typzuordnung

Wenn ECPG-Anwendungen Werte zwischen dem PostgreSQL-Server und der C-Anwendung austauschen, etwa beim Abrufen von Abfrageergebnissen oder beim Ausführen von SQL-Anweisungen mit Eingabeparametern, müssen diese Werte zwischen PostgreSQL-Datentypen und Variablentypen der Hostsprache, konkret C-Datentypen, konvertiert werden. Ein wichtiger Zweck von ECPG ist, dies in den meisten Fällen automatisch zu erledigen.

Dabei gibt es zwei Arten von Datentypen. Einige einfache PostgreSQL-Datentypen wie `integer` und `text` können direkt von der Anwendung gelesen und geschrieben werden. Andere PostgreSQL-Datentypen wie `timestamp` und `numeric` können nur über spezielle Bibliotheksfunktionen sinnvoll verarbeitet werden; siehe [Abschnitt 34.4.4.2](#34442-auf-spezielle-datentypen-zugreifen).

Tabelle 34.1 zeigt, welche PostgreSQL-Datentypen welchen C-Datentypen entsprechen. Wenn ein Wert eines bestimmten PostgreSQL-Datentyps gesendet oder empfangen werden soll, deklarieren Sie im Deklarationsabschnitt eine C-Variable des entsprechenden C-Datentyps.

**Tabelle 34.1. Zuordnung zwischen PostgreSQL-Datentypen und C-Variablentypen**

| PostgreSQL-Datentyp | Hostvariablentyp |
| --- | --- |
| `smallint` | `short` |
| `integer` | `int` |
| `bigint` | `long long int` |
| `decimal` | `decimal` [a] |
| `numeric` | `numeric` [a] |
| `real` | `float` |
| `double precision` | `double` |
| `smallserial` | `short` |
| `serial` | `int` |
| `bigserial` | `long long int` |
| `oid` | `unsigned int` |
| `character(n)`, `varchar(n)`, `text` | `char[n+1]`, `VARCHAR[n+1]` |
| `name` | `char[NAMEDATALEN]` |
| `timestamp` | `timestamp` [a] |
| `interval` | `interval` [a] |
| `date` | `date` [a] |
| `boolean` | `bool` [b] |
| `bytea` | `char *`, `bytea[n]` |

[a] Dieser Typ kann nur über spezielle Bibliotheksfunktionen verarbeitet werden; siehe [Abschnitt 34.4.4.2](#34442-auf-spezielle-datentypen-zugreifen).

[b] In `ecpglib.h` deklariert, falls nicht nativ vorhanden.

#### 34.4.4.1. Zeichenketten behandeln

Für SQL-Zeichenkettentypen wie `varchar` und `text` gibt es zwei Möglichkeiten, Hostvariablen zu deklarieren.

Eine Möglichkeit ist `char[]`, also ein Array von `char`, die in C üblichste Form für Zeichendaten.

```c
EXEC SQL BEGIN DECLARE SECTION;
char str[50];
EXEC SQL END DECLARE SECTION;
```

Dabei müssen Sie selbst auf die Länge achten. Wird diese Hostvariable als Zielvariable einer Abfrage verwendet, die eine Zeichenkette mit mehr als 49 Zeichen zurückgibt, entsteht ein Pufferüberlauf.

Die andere Möglichkeit ist der Typ `VARCHAR`, ein spezieller von ECPG bereitgestellter Typ. Die Definition eines Arrays vom Typ `VARCHAR` wird für jede Variable in eine benannte Struktur umgewandelt. Eine Deklaration wie

```c
VARCHAR var[180];
```

wird umgewandelt in:

```c
struct varchar_var { int len; char arr[180]; } var;
```

Das Element `arr` enthält die Zeichenkette einschließlich abschließendem Nullbyte. Um eine Zeichenkette in einer `VARCHAR`-Hostvariablen zu speichern, muss die Hostvariable daher mit einer Länge deklariert werden, die das Nullbyte einschließt. Das Element `len` enthält die Länge der in `arr` gespeicherten Zeichenkette ohne abschließendes Nullbyte. Wenn eine Hostvariable als Eingabe für eine Abfrage verwendet wird und `strlen(arr)` und `len` voneinander abweichen, wird der kürzere Wert verwendet.

`VARCHAR` kann groß oder klein geschrieben werden, aber nicht in gemischter Schreibweise. Hostvariablen vom Typ `char` und `VARCHAR` können auch Werte anderer SQL-Typen aufnehmen; diese werden dann in ihrer Zeichenkettenform gespeichert.

#### 34.4.4.2. Auf spezielle Datentypen zugreifen

ECPG enthält einige spezielle Typen, die den Umgang mit bestimmten PostgreSQL-Serverdatentypen erleichtern. Insbesondere gibt es Unterstützung für `numeric`, `decimal`, `date`, `timestamp` und `interval`. Diese Datentypen lassen sich wegen ihrer komplexen internen Struktur nicht sinnvoll auf primitive Hostvariablentypen wie `int`, `long long int` oder `char[]` abbilden. Anwendungen deklarieren dafür Hostvariablen in speziellen Typen und greifen über Funktionen der `pgtypes`-Bibliothek darauf zu. Diese Bibliothek wird in [Abschnitt 34.6](34_ECPG_Embedded_SQL_in_C.md#346-pgtypesbibliothek) genauer beschrieben.

##### 34.4.4.2.1. `timestamp`, `date`

Das folgende Muster zeigt die Behandlung von `timestamp`-Variablen in einer ECPG-Hostanwendung.

Zuerst muss die Headerdatei für den Timestamp-Typ eingebunden werden:

```c
#include <pgtypes_timestamp.h>
```

Dann wird im Deklarationsabschnitt eine Hostvariable vom Typ `timestamp` deklariert:

```c
EXEC SQL BEGIN DECLARE SECTION;
timestamp ts;
EXEC SQL END DECLARE SECTION;
```

Nachdem ein Wert in die Hostvariable gelesen wurde, wird er mit Funktionen der `pgtypes`-Bibliothek verarbeitet. Im folgenden Beispiel wird der Timestamp-Wert mit `PGTYPEStimestamp_to_asc()` in Textform umgewandelt:

```c
EXEC SQL SELECT now()::timestamp INTO :ts;

printf("ts = %s\n", PGTYPEStimestamp_to_asc(ts));
```

Das Beispiel zeigt etwa folgendes Ergebnis:

```text
ts = 2010-06-27 18:03:56.949343
```

Der Typ `date` kann auf dieselbe Weise behandelt werden. Das Programm bindet `pgtypes_date.h` ein, deklariert eine Hostvariable vom Typ `date` und wandelt einen `DATE`-Wert mit `PGTYPESdate_to_asc()` in Textform um.

##### 34.4.4.2.2. `interval`

Die Behandlung des Typs `interval` ähnelt der von `timestamp` und `date`. Allerdings muss Speicher für einen `interval`-Wert ausdrücklich allokiert werden. Anders gesagt: Der Speicher für die Variable muss auf dem Heap liegen, nicht auf dem Stack.

Beispielprogramm:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pgtypes_interval.h>

int
main(void)
{
EXEC SQL BEGIN DECLARE SECTION;
    interval *in;
EXEC SQL END DECLARE SECTION;

    EXEC SQL CONNECT TO testdb;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    in = PGTYPESinterval_new();
    EXEC SQL SELECT '1 min'::interval INTO :in;
    printf("interval = %s\n", PGTYPESinterval_to_asc(in));
    PGTYPESinterval_free(in);

    EXEC SQL COMMIT;
    EXEC SQL DISCONNECT ALL;
    return 0;
}
```

##### 34.4.4.2.3. `numeric`, `decimal`

Die Behandlung der Typen `numeric` und `decimal` ähnelt der des Typs `interval`: Es wird ein Zeiger definiert, Speicher auf dem Heap allokiert und über Funktionen der `pgtypes`-Bibliothek auf die Variable zugegriffen.

Für den Typ `decimal` gibt es keine eigenen Funktionen. Eine Anwendung muss den Wert mit einer `pgtypes`-Bibliotheksfunktion in eine `numeric`-Variable umwandeln, um ihn weiterzuverarbeiten.

Beispielprogramm für `numeric`- und `decimal`-Variablen:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pgtypes_numeric.h>

EXEC SQL WHENEVER SQLERROR STOP;

int
main(void)
{
EXEC SQL BEGIN DECLARE SECTION;
    numeric *num;
    numeric *num2;
    decimal *dec;
EXEC SQL END DECLARE SECTION;

    EXEC SQL CONNECT TO testdb;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    num = PGTYPESnumeric_new();
    dec = PGTYPESdecimal_new();

    EXEC SQL SELECT 12.345::numeric(4,2), 23.456::decimal(4,2)
        INTO :num, :dec;

    printf("numeric = %s\n", PGTYPESnumeric_to_asc(num, 0));
    printf("numeric = %s\n", PGTYPESnumeric_to_asc(num, 1));
    printf("numeric = %s\n", PGTYPESnumeric_to_asc(num, 2));

    /* Decimal in numeric umwandeln, um den Wert anzuzeigen. */
    num2 = PGTYPESnumeric_new();
    PGTYPESnumeric_from_decimal(dec, num2);

    printf("decimal = %s\n", PGTYPESnumeric_to_asc(num2, 0));
    printf("decimal = %s\n", PGTYPESnumeric_to_asc(num2, 1));
    printf("decimal = %s\n", PGTYPESnumeric_to_asc(num2, 2));

    PGTYPESnumeric_free(num2);
    PGTYPESdecimal_free(dec);
    PGTYPESnumeric_free(num);

    EXEC SQL COMMIT;
    EXEC SQL DISCONNECT ALL;
    return 0;
}
```

##### 34.4.4.2.4. `bytea`

Die Behandlung des Typs `bytea` ähnelt der von `VARCHAR`. Die Definition eines Arrays vom Typ `bytea` wird für jede Variable in eine benannte Struktur umgewandelt. Eine Deklaration wie

```c
bytea var[180];
```

wird umgewandelt in:

```c
struct bytea_var { int len; char arr[180]; } var;
```

Das Element `arr` enthält Daten im Binärformat. Anders als `VARCHAR` kann es auch `'\0'` als Teil der Daten enthalten. Die Daten werden von `ecpglib` aus dem Hexformat bzw. in das Hexformat konvertiert und gesendet oder empfangen.

> **Hinweis**
>
> Eine `bytea`-Variable kann nur verwendet werden, wenn `bytea_output` auf `hex` gesetzt ist.

#### 34.4.4.3. Hostvariablen mit nichtprimitiven Typen

Als Hostvariablen können auch Arrays, `typedef`s, Strukturen und Zeiger verwendet werden.

##### 34.4.4.3.1. Arrays

Für Arrays als Hostvariablen gibt es zwei Anwendungsfälle. Der erste ist das Speichern von Zeichenketten in `char[]` oder `VARCHAR[]`, wie in [Abschnitt 34.4.4.1](#34441-zeichenketten-behandeln) beschrieben. Der zweite ist das Abrufen mehrerer Zeilen aus einem Abfrageergebnis ohne Cursor. Ohne Array muss für Ergebnismengen mit mehreren Zeilen ein Cursor mit `FETCH` verwendet werden. Mit Array-Hostvariablen können mehrere Zeilen auf einmal empfangen werden. Das Array muss groß genug für alle Zeilen sein, sonst ist ein Pufferüberlauf wahrscheinlich.

Das folgende Beispiel durchsucht die Systemtabelle `pg_database` und zeigt OIDs und Namen der verfügbaren Datenbanken:

```c
int
main(void)
{
EXEC SQL BEGIN DECLARE SECTION;
    int dbid[8];
    char dbname[8][16];
    int i;
EXEC SQL END DECLARE SECTION;

    memset(dbname, 0, sizeof(char) * 16 * 8);
    memset(dbid, 0, sizeof(int) * 8);

    EXEC SQL CONNECT TO testdb;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    /* Mehrere Zeilen auf einmal in Arrays abrufen. */
    EXEC SQL SELECT oid, datname INTO :dbid, :dbname FROM pg_database;

    for (i = 0; i < 8; i++)
        printf("oid=%d, dbname=%s\n", dbid[i], dbname[i]);

    EXEC SQL COMMIT;
    EXEC SQL DISCONNECT ALL;
    return 0;
}
```

Das Beispiel zeigt etwa folgendes Ergebnis; die genauen Werte hängen von der lokalen Umgebung ab:

```text
oid=1, dbname=template1
oid=11510, dbname=template0
oid=11511, dbname=postgres
oid=313780, dbname=testdb
oid=0, dbname=
oid=0, dbname=
oid=0, dbname=
```

##### 34.4.4.3.2. Strukturen

Eine Struktur, deren Elementnamen den Spaltennamen eines Abfrageergebnisses entsprechen, kann verwendet werden, um mehrere Spalten auf einmal abzurufen. Dadurch lassen sich mehrere Spaltenwerte in einer einzigen Hostvariablen behandeln.

Das folgende Beispiel ruft OIDs, Namen und Größen der verfügbaren Datenbanken aus `pg_database` ab und verwendet dazu `pg_database_size()`. Eine Strukturvariable `dbinfo_t`, deren Elemente den Spalten des `SELECT`-Ergebnisses entsprechen, nimmt eine Ergebniszeile auf, ohne dass mehrere Hostvariablen in der `FETCH`-Anweisung stehen müssen.

```c
EXEC SQL BEGIN DECLARE SECTION;
typedef struct
{
    int oid;
    char datname[65];
    long long int size;
} dbinfo_t;

dbinfo_t dbval;
EXEC SQL END DECLARE SECTION;

memset(&dbval, 0, sizeof(dbinfo_t));

EXEC SQL DECLARE cur1 CURSOR FOR
    SELECT oid, datname, pg_database_size(oid) AS size FROM pg_database;
EXEC SQL OPEN cur1;

/* Wenn das Ende der Ergebnismenge erreicht ist, die while-Schleife verlassen. */
EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    /* Mehrere Spalten in eine Struktur abrufen. */
    EXEC SQL FETCH FROM cur1 INTO :dbval;

    /* Elemente der Struktur ausgeben. */
    printf("oid=%d, datname=%s, size=%lld\n",
           dbval.oid, dbval.datname, dbval.size);
}

EXEC SQL CLOSE cur1;
```

Das Beispiel zeigt etwa folgendes Ergebnis:

```text
oid=1, datname=template1, size=4324580
oid=11510, datname=template0, size=4243460
oid=11511, datname=postgres, size=4324580
oid=313780, datname=testdb, size=8183012
```

Struktur-Hostvariablen „absorbieren“ so viele Spalten, wie die Struktur Felder hat. Zusätzliche Spalten können anderen Hostvariablen zugewiesen werden. Das obige Programm könnte zum Beispiel so umstrukturiert werden, dass `size` außerhalb der Struktur liegt:

```c
EXEC SQL BEGIN DECLARE SECTION;
typedef struct
{
    int oid;
    char datname[65];
} dbinfo_t;

dbinfo_t dbval;
long long int size;
EXEC SQL END DECLARE SECTION;

memset(&dbval, 0, sizeof(dbinfo_t));

EXEC SQL DECLARE cur1 CURSOR FOR
    SELECT oid, datname, pg_database_size(oid) AS size FROM pg_database;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    EXEC SQL FETCH FROM cur1 INTO :dbval, :size;

    printf("oid=%d, datname=%s, size=%lld\n",
           dbval.oid, dbval.datname, size);
}

EXEC SQL CLOSE cur1;
```

##### 34.4.4.3.3. `typedef`s

Mit dem Schlüsselwort `typedef` können neue Typnamen auf vorhandene Typen abgebildet werden.

```c
EXEC SQL BEGIN DECLARE SECTION;
typedef char mychartype[40];
typedef long serial_t;
EXEC SQL END DECLARE SECTION;
```

Alternativ kann auch geschrieben werden:

```sql
EXEC SQL TYPE serial_t IS long;
```

Diese Deklaration muss nicht Teil eines Deklarationsabschnitts sein. `typedef`s können also auch als normale C-Anweisungen geschrieben werden.

Jedes Wort, das als `typedef` deklariert wird, kann später im selben Programm nicht mehr als SQL-Schlüsselwort in `EXEC SQL`-Befehlen verwendet werden. Das folgende Beispiel funktioniert daher nicht:

```c
EXEC SQL BEGIN DECLARE SECTION;
typedef int start;
EXEC SQL END DECLARE SECTION;

...

EXEC SQL START TRANSACTION;
```

ECPG meldet für `START TRANSACTION` einen Syntaxfehler, weil es `START` nicht mehr als SQL-Schlüsselwort erkennt, sondern nur noch als `typedef`. Wenn ein solcher Konflikt auftritt und das Umbenennen des `typedef` unpraktisch ist, kann der SQL-Befehl über dynamisches SQL geschrieben werden.

> **Hinweis**
>
> In PostgreSQL-Versionen vor v16 führte die Verwendung von SQL-Schlüsselwörtern als `typedef`-Namen eher zu Syntaxfehlern beim Gebrauch des `typedef` selbst als beim Gebrauch des Namens als SQL-Schlüsselwort. Das neue Verhalten verursacht weniger wahrscheinlich Probleme, wenn eine bestehende ECPG-Anwendung mit einer neuen PostgreSQL-Version und neuen Schlüsselwörtern erneut kompiliert wird.

##### 34.4.4.3.4. Zeiger

Zeiger auf die gebräuchlichsten Typen können deklariert werden. Beachten Sie jedoch, dass Zeiger ohne Auto-Allokation nicht als Zielvariablen von Abfragen verwendet werden können. Weitere Informationen zur Auto-Allokation stehen in [Abschnitt 34.7](34_ECPG_Embedded_SQL_in_C.md#347-descriptorbereiche-verwenden).

```c
EXEC SQL BEGIN DECLARE SECTION;
int   *intp;
char **charp;
EXEC SQL END DECLARE SECTION;
```

### 34.4.5. Nichtprimitive SQL-Datentypen behandeln

Dieser Abschnitt beschreibt, wie nichtskalare und benutzerdefinierte SQL-Datentypen in ECPG-Anwendungen behandelt werden. Das unterscheidet sich von der im vorherigen Abschnitt beschriebenen Behandlung von Hostvariablen mit nichtprimitiven Typen.

#### 34.4.5.1. Arrays

Mehrdimensionale SQL-Arrays werden in ECPG nicht direkt unterstützt. Eindimensionale SQL-Arrays können auf C-Array-Hostvariablen abgebildet werden und umgekehrt. Beim Erzeugen einer Anweisung kennt `ecpg` jedoch die Spaltentypen nicht und kann daher nicht prüfen, ob ein C-Array in ein entsprechendes SQL-Array eingegeben wird. Beim Verarbeiten der Ausgabe einer SQL-Anweisung besitzt `ecpg` die nötigen Informationen und prüft dann, ob beide Seiten Arrays sind.

Wenn eine Abfrage einzelne Elemente eines Arrays getrennt anspricht, wird die direkte Verwendung von Arrays in ECPG vermieden. Dann sollte eine Hostvariable verwendet werden, deren Typ auf den Elementtyp abgebildet werden kann. Ist ein Spaltentyp beispielsweise ein Array von `integer`, kann eine Hostvariable vom Typ `int` verwendet werden. Bei Elementtypen `varchar` oder `text` können `char[]` oder `VARCHAR[]` verwendet werden.

Beispiel mit folgender Tabelle:

```sql
CREATE TABLE t3 (
    ii integer[]
);
```

```text
testdb=> SELECT * FROM t3;
-------------
 {1,2,3,4,5}
(1 row)
```

Das folgende Beispielprogramm ruft das vierte Element des Arrays ab und speichert es in einer Hostvariablen vom Typ `int`:

```c
EXEC SQL BEGIN DECLARE SECTION;
int ii;
EXEC SQL END DECLARE SECTION;

EXEC SQL DECLARE cur1 CURSOR FOR SELECT ii[4] FROM t3;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    EXEC SQL FETCH FROM cur1 INTO :ii;
    printf("ii=%d\n", ii);
}

EXEC SQL CLOSE cur1;
```

Ausgabe:

```text
ii=4
```

Um mehrere Array-Elemente auf mehrere Elemente einer Array-Hostvariablen abzubilden, müssen jedes Element der Array-Spalte und jedes Element des Hostvariablen-Arrays getrennt behandelt werden:

```c
EXEC SQL BEGIN DECLARE SECTION;
int ii_a[8];
EXEC SQL END DECLARE SECTION;

EXEC SQL DECLARE cur1 CURSOR FOR
    SELECT ii[1], ii[2], ii[3], ii[4] FROM t3;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    EXEC SQL FETCH FROM cur1
        INTO :ii_a[0], :ii_a[1], :ii_a[2], :ii_a[3];
    ...
}
```

Das folgende Beispiel würde dagegen in diesem Fall nicht korrekt funktionieren, weil eine Array-Spalte nicht direkt auf eine Array-Hostvariable abgebildet werden kann:

```c
EXEC SQL BEGIN DECLARE SECTION;
int ii_a[8];
EXEC SQL END DECLARE SECTION;

EXEC SQL DECLARE cur1 CURSOR FOR SELECT ii FROM t3;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    /* FALSCH */
    EXEC SQL FETCH FROM cur1 INTO :ii_a;
    ...
}
```

Eine weitere Umgehung besteht darin, Arrays in ihrer externen Zeichenkettendarstellung in Hostvariablen vom Typ `char[]` oder `VARCHAR[]` zu speichern. Weitere Details zu dieser Darstellung stehen in [Abschnitt 8.15.2](08_Datentypen.md#8152-eingabe-von-arraywerten). Das bedeutet allerdings, dass das Array im Hostprogramm nicht ohne weiteres als Array genutzt werden kann, ohne die Textdarstellung zusätzlich zu parsen.

#### 34.4.5.2. Zusammengesetzte Typen

Zusammengesetzte Typen werden in ECPG nicht direkt unterstützt, aber es gibt einfache Umgehungen. Sie ähneln den oben beschriebenen Array-Umgehungen: entweder jedes Attribut getrennt ansprechen oder die externe Zeichenkettendarstellung verwenden.

Für die folgenden Beispiele gilt dieser Typ und diese Tabelle:

```sql
CREATE TYPE comp_t AS (intval integer, textval varchar(32));
CREATE TABLE t4 (compval comp_t);
INSERT INTO t4 VALUES ((256, 'PostgreSQL'));
```

Die naheliegendste Lösung ist, jedes Attribut getrennt anzusprechen:

```c
EXEC SQL BEGIN DECLARE SECTION;
int intval;
varchar textval[33];
EXEC SQL END DECLARE SECTION;

/* Jedes Element der zusammengesetzten Spalte in die SELECT-Liste setzen. */
EXEC SQL DECLARE cur1 CURSOR FOR
    SELECT (compval).intval, (compval).textval FROM t4;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    /* Jedes Element der zusammengesetzten Spalte in Hostvariablen abrufen. */
    EXEC SQL FETCH FROM cur1 INTO :intval, :textval;

    printf("intval=%d, textval=%s\n", intval, textval.arr);
}

EXEC SQL CLOSE cur1;
```

Die Hostvariablen für `FETCH` können auch in einer Struktur zusammengefasst werden. Die beiden Hostvariablen `intval` und `textval` werden dabei Elemente der Struktur `comp_t`, und die Struktur wird in der `FETCH`-Anweisung angegeben:

```c
EXEC SQL BEGIN DECLARE SECTION;
typedef struct
{
    int intval;
    varchar textval[33];
} comp_t;

comp_t compval;
EXEC SQL END DECLARE SECTION;

EXEC SQL DECLARE cur1 CURSOR FOR
    SELECT (compval).intval, (compval).textval FROM t4;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    /* Alle Werte der SELECT-Liste in eine Struktur legen. */
    EXEC SQL FETCH FROM cur1 INTO :compval;

    printf("intval=%d, textval=%s\n", compval.intval, compval.textval.arr);
}

EXEC SQL CLOSE cur1;
```

Obwohl in `FETCH` eine Struktur verwendet wird, werden die Attributnamen in der `SELECT`-Klausel einzeln angegeben. Das kann durch `*` verbessert werden, um alle Attribute des zusammengesetzten Wertes anzufordern:

```c
EXEC SQL DECLARE cur1 CURSOR FOR SELECT (compval).* FROM t4;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    EXEC SQL FETCH FROM cur1 INTO :compval;

    printf("intval=%d, textval=%s\n", compval.intval, compval.textval.arr);
}
```

Auf diese Weise können zusammengesetzte Typen fast nahtlos auf Strukturen abgebildet werden, obwohl ECPG den zusammengesetzten Typ selbst nicht versteht. Schließlich ist es auch möglich, Werte zusammengesetzter Typen in ihrer externen Zeichenkettendarstellung in Hostvariablen vom Typ `char[]` oder `VARCHAR[]` zu speichern. Dann ist der Zugriff auf die Felder aus dem Hostprogramm heraus allerdings nicht einfach.

#### 34.4.5.3. Benutzerdefinierte Basistypen

Neue benutzerdefinierte Basistypen werden von ECPG nicht direkt unterstützt. Sie können die externe Zeichenkettendarstellung und Hostvariablen vom Typ `char[]` oder `VARCHAR[]` verwenden; diese Lösung ist für viele Typen angemessen und ausreichend.

Das folgende Beispiel verwendet den Datentyp `complex` aus [Abschnitt 36.13](36_SQL_erweitern.md#3613-benutzerdefinierte-typen). Die externe Zeichenkettendarstellung dieses Typs ist `(%f,%f)` und wird durch die Funktionen `complex_in()` und `complex_out()` definiert. Das Beispiel fügt Werte des Typs `complex`, nämlich `(1,1)` und `(3,3)`, in die Spalten `a` und `b` ein und liest sie anschließend aus der Tabelle.

```c
EXEC SQL BEGIN DECLARE SECTION;
varchar a[64];
varchar b[64];
EXEC SQL END DECLARE SECTION;

EXEC SQL INSERT INTO test_complex VALUES ('(1,1)', '(3,3)');

EXEC SQL DECLARE cur1 CURSOR FOR SELECT a, b FROM test_complex;
EXEC SQL OPEN cur1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    EXEC SQL FETCH FROM cur1 INTO :a, :b;
    printf("a=%s, b=%s\n", a.arr, b.arr);
}

EXEC SQL CLOSE cur1;
```

Ausgabe:

```text
a=(1,1), b=(3,3)
```

Eine weitere Umgehung besteht darin, benutzerdefinierte Typen in ECPG nicht direkt zu verwenden, sondern eine Funktion oder einen Cast zu erstellen, der zwischen dem benutzerdefinierten Typ und einem primitiven Typ konvertiert, den ECPG verarbeiten kann. Typecasts, insbesondere implizite, sollten allerdings sehr vorsichtig in ein Typsystem eingeführt werden.

Beispiel:

```sql
CREATE FUNCTION create_complex(r double, i double) RETURNS complex
LANGUAGE SQL
IMMUTABLE
AS $$ SELECT $1 * complex '(1,0')' + $2 * complex '(0,1)' $$;
```

Nach dieser Definition hat Folgendes

```c
EXEC SQL BEGIN DECLARE SECTION;
double a, b, c, d;
EXEC SQL END DECLARE SECTION;

a = 1;
b = 2;
c = 3;
d = 4;

EXEC SQL INSERT INTO test_complex VALUES (create_complex(:a, :b),
                                          create_complex(:c, :d));
```

dieselbe Wirkung wie:

```sql
EXEC SQL INSERT INTO test_complex VALUES ('(1,2)', '(3,4)');
```

### 34.4.6. Indikatoren

Die obigen Beispiele behandeln keine Nullwerte. Tatsächlich lösen die Abrufbeispiele einen Fehler aus, wenn sie einen Nullwert aus der Datenbank lesen. Um Nullwerte an die Datenbank zu übergeben oder aus der Datenbank abzurufen, muss an jede Hostvariable, die Daten enthält, eine zweite Hostvariablenspezifikation angehängt werden. Diese zweite Hostvariable heißt Indikator und enthält ein Flag, das angibt, ob der Wert null ist. In diesem Fall wird der Wert der eigentlichen Hostvariablen ignoriert.

Beispiel für korrektes Abrufen von Nullwerten:

```c
EXEC SQL BEGIN DECLARE SECTION;
VARCHAR val;
int val_ind;
EXEC SQL END DECLARE SECTION;

...

EXEC SQL SELECT b INTO :val :val_ind FROM test1;
```

Die Indikatorvariable `val_ind` ist null, wenn der Wert nicht null war, und negativ, wenn der Wert null war. Oracle-spezifisches Verhalten kann über [Abschnitt 34.16](34_ECPG_Embedded_SQL_in_C.md#3416-oraclekompatibilitätsmodus) aktiviert werden.

Der Indikator hat noch eine weitere Funktion: Ist der Indikatorwert positiv, bedeutet das, dass der Wert nicht null war, aber beim Speichern in der Hostvariablen abgeschnitten wurde.

Wenn dem Präprozessor `ecpg` das Argument `-r no_indicator` übergeben wird, arbeitet er im „No-Indicator“-Modus. In diesem Modus werden Nullwerte ohne angegebene Indikatorvariable bei Zeichenkettentypen als leere Zeichenkette und bei Ganzzahltypen als der niedrigste mögliche Wert des jeweiligen Typs signalisiert, zum Beispiel `INT_MIN` für `int`.

## 34.5. Dynamisches SQL

In vielen Fällen sind die konkreten SQL-Anweisungen, die eine Anwendung ausführen muss, schon bekannt, wenn die Anwendung geschrieben wird. In manchen Fällen werden SQL-Anweisungen jedoch erst zur Laufzeit zusammengesetzt oder von einer externen Quelle bereitgestellt. Dann können Sie die SQL-Anweisungen nicht direkt in den C-Quellcode einbetten; es gibt aber eine Möglichkeit, beliebige SQL-Anweisungen aufzurufen, die Sie in einer Zeichenkettenvariablen bereitstellen.

### 34.5.1. Anweisungen ohne Ergebnismenge ausführen

Die einfachste Möglichkeit, eine beliebige SQL-Anweisung auszuführen, ist der Befehl `EXECUTE IMMEDIATE`. Zum Beispiel:

```c
EXEC SQL BEGIN DECLARE SECTION;
const char *stmt = "CREATE TABLE test1 (...);";
EXEC SQL END DECLARE SECTION;

EXEC SQL EXECUTE IMMEDIATE :stmt;
```

`EXECUTE IMMEDIATE` kann für SQL-Anweisungen verwendet werden, die keine Ergebnismenge zurückgeben, zum Beispiel DDL, `INSERT`, `UPDATE` oder `DELETE`. Anweisungen, die Daten abrufen, etwa `SELECT`, können auf diese Weise nicht ausgeführt werden. Der nächste Abschnitt beschreibt, wie das geht.

### 34.5.2. Eine Anweisung mit Eingabeparametern ausführen

Eine leistungsfähigere Möglichkeit, beliebige SQL-Anweisungen auszuführen, besteht darin, sie einmal vorzubereiten und die vorbereitete Anweisung anschließend beliebig oft auszuführen. Außerdem ist es möglich, eine verallgemeinerte Fassung einer Anweisung vorzubereiten und daraus durch Ersetzen von Parametern konkrete Fassungen auszuführen. Schreiben Sie beim Vorbereiten der Anweisung Fragezeichen an die Stellen, an denen später Parameter eingesetzt werden sollen. Zum Beispiel:

```c
EXEC SQL BEGIN DECLARE SECTION;
const char *stmt = "INSERT INTO test1 VALUES(?, ?);";
EXEC SQL END DECLARE SECTION;

EXEC SQL PREPARE mystmt FROM :stmt;
...
EXEC SQL EXECUTE mystmt USING 42, 'foobar';
```

Wenn Sie die vorbereitete Anweisung nicht mehr benötigen, sollten Sie sie freigeben:

```c
EXEC SQL DEALLOCATE PREPARE name;
```

### 34.5.3. Eine Anweisung mit Ergebnismenge ausführen

Um eine SQL-Anweisung mit genau einer Ergebniszeile auszuführen, kann `EXECUTE` verwendet werden. Um das Ergebnis zu speichern, fügen Sie eine `INTO`-Klausel hinzu.

```c
EXEC SQL BEGIN DECLARE SECTION;
const char *stmt = "SELECT a, b, c FROM test1 WHERE a > ?";
int v1, v2;
VARCHAR v3[50];
EXEC SQL END DECLARE SECTION;

EXEC SQL PREPARE mystmt FROM :stmt;
...
EXEC SQL EXECUTE mystmt INTO :v1, :v2, :v3 USING 37;
```

Ein `EXECUTE`-Befehl kann eine `INTO`-Klausel, eine `USING`-Klausel, beide oder keine von beiden haben.

Wenn eine Abfrage voraussichtlich mehr als eine Ergebniszeile zurückgibt, sollte ein Cursor verwendet werden, wie im folgenden Beispiel. Weitere Details zum Cursor stehen in [Abschnitt 34.3.2](#3432-cursor-verwenden).

```c
EXEC SQL BEGIN DECLARE SECTION;
char dbaname[128];
char datname[128];
char *stmt = "SELECT u.usename as dbaname, d.datname "
             " FROM pg_database d, pg_user u "
             " WHERE d.datdba = u.usesysid";
EXEC SQL END DECLARE SECTION;

EXEC SQL CONNECT TO testdb AS con1 USER testuser;
EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
EXEC SQL COMMIT;

EXEC SQL PREPARE stmt1 FROM :stmt;

EXEC SQL DECLARE cursor1 CURSOR FOR stmt1;
EXEC SQL OPEN cursor1;

EXEC SQL WHENEVER NOT FOUND DO BREAK;

while (1)
{
    EXEC SQL FETCH cursor1 INTO :dbaname,:datname;
    printf("dbaname=%s, datname=%s\n", dbaname, datname);
}

EXEC SQL CLOSE cursor1;

EXEC SQL COMMIT;
EXEC SQL DISCONNECT ALL;
```

## 34.6. pgtypes-Bibliothek

Die `pgtypes`-Bibliothek bildet PostgreSQL-Datenbanktypen auf C-Entsprechungen ab, die in C-Programmen verwendet werden können. Außerdem stellt sie Funktionen bereit, um einfache Berechnungen mit diesen Typen innerhalb von C auszuführen, also ohne Hilfe des PostgreSQL-Servers. Siehe folgendes Beispiel:

```c
EXEC SQL BEGIN DECLARE SECTION;
   date date1;
   timestamp ts1, tsout;
   interval iv1;
   char *out;
EXEC SQL END DECLARE SECTION;

PGTYPESdate_today(&date1);
EXEC SQL SELECT started, duration INTO :ts1, :iv1 FROM datetbl
 WHERE d=:date1;
PGTYPEStimestamp_add_interval(&ts1, &iv1, &tsout);
out = PGTYPEStimestamp_to_asc(&tsout);
printf("Started + duration: %s\n", out);
PGTYPESchar_free(out);
```

### 34.6.1. Zeichenketten

Einige Funktionen, etwa `PGTYPESnumeric_to_asc`, geben einen Zeiger auf eine frisch allozierte Zeichenkette zurück. Diese Ergebnisse sollten mit `PGTYPESchar_free` statt mit `free` freigegeben werden. Das ist nur unter Windows wichtig, wo Speicherallokation und Speicherfreigabe manchmal durch dieselbe Bibliothek erfolgen müssen.

### 34.6.2. Der Typ `numeric`

Der Typ `numeric` erlaubt Berechnungen mit beliebiger Genauigkeit. Den entsprechenden Typ im PostgreSQL-Server beschreibt [Abschnitt 8.1](08_Datentypen.md#81-numerische-typen). Wegen der beliebigen Genauigkeit muss diese Variable dynamisch wachsen und schrumpfen können. Deshalb können `numeric`-Variablen nur auf dem Heap erzeugt werden, und zwar mit den Funktionen `PGTYPESnumeric_new` und `PGTYPESnumeric_free`. Der ähnliche, aber in der Genauigkeit beschränkte Typ `decimal` kann sowohl auf dem Stack als auch auf dem Heap erzeugt werden.

Die folgenden Funktionen können verwendet werden, um mit dem Typ `numeric` zu arbeiten:

`PGTYPESnumeric_new`

Fordert einen Zeiger auf eine neu allozierte `numeric`-Variable an.

```c
numeric *PGTYPESnumeric_new(void);
```

`PGTYPESnumeric_free`

Gibt einen `numeric`-Typ frei und gibt den gesamten zugehörigen Speicher zurück.

```c
void PGTYPESnumeric_free(numeric *var);
```

`PGTYPESnumeric_from_asc`

Parst einen `numeric`-Typ aus seiner Zeichenkettendarstellung.

```c
numeric *PGTYPESnumeric_from_asc(char *str, char **endptr);
```

Gültige Formate sind zum Beispiel `-2`, `.794`, `+3.44`, `592.49E07` oder `-32.84e-4`. Wenn der Wert erfolgreich geparst werden konnte, wird ein gültiger Zeiger zurückgegeben, andernfalls der Nullzeiger. ECPG parst derzeit immer die vollständige Zeichenkette und unterstützt daher aktuell nicht, die Adresse des ersten ungültigen Zeichens in `*endptr` zu speichern. Sie können `endptr` gefahrlos auf `NULL` setzen.

`PGTYPESnumeric_to_asc`

Gibt einen Zeiger auf eine mit `malloc` allozierte Zeichenkette zurück, die die Zeichenkettendarstellung des `numeric`-Typs `num` enthält.

```c
char *PGTYPESnumeric_to_asc(numeric *num, int dscale);
```

Der numerische Wert wird mit `dscale` Nachkommastellen ausgegeben; falls nötig, wird gerundet. Das Ergebnis muss mit `PGTYPESchar_free()` freigegeben werden.

`PGTYPESnumeric_add`

Addiert zwei `numeric`-Variablen in eine dritte.

```c
int PGTYPESnumeric_add(numeric *var1, numeric *var2, numeric
 *result);
```

Die Funktion addiert die Variablen `var1` und `var2` in die Ergebnisvariable `result`. Bei Erfolg gibt die Funktion 0 zurück, im Fehlerfall -1.

`PGTYPESnumeric_sub`

Subtrahiert zwei `numeric`-Variablen und gibt das Ergebnis in einer dritten zurück.

```c
int PGTYPESnumeric_sub(numeric *var1, numeric *var2, numeric
 *result);
```

Die Funktion subtrahiert die Variable `var2` von der Variable `var1`. Das Ergebnis der Operation wird in der Variable `result` gespeichert. Bei Erfolg gibt die Funktion 0 zurück, im Fehlerfall -1.

`PGTYPESnumeric_mul`

Multipliziert zwei `numeric`-Variablen und gibt das Ergebnis in einer dritten zurück.

```c
int PGTYPESnumeric_mul(numeric *var1, numeric *var2, numeric
 *result);
```

Die Funktion multipliziert die Variablen `var1` und `var2`. Das Ergebnis der Operation wird in der Variable `result` gespeichert. Bei Erfolg gibt die Funktion 0 zurück, im Fehlerfall -1.

`PGTYPESnumeric_div`

Dividiert zwei `numeric`-Variablen und gibt das Ergebnis in einer dritten zurück.

```c
int PGTYPESnumeric_div(numeric *var1, numeric *var2, numeric
 *result);
```

Die Funktion dividiert die Variable `var1` durch `var2`. Das Ergebnis der Operation wird in der Variable `result` gespeichert. Bei Erfolg gibt die Funktion 0 zurück, im Fehlerfall -1.

`PGTYPESnumeric_cmp`

Vergleicht zwei `numeric`-Variablen.

```c
int PGTYPESnumeric_cmp(numeric *var1, numeric *var2)
```

Diese Funktion vergleicht zwei `numeric`-Variablen. Im Fehlerfall wird `INT_MAX` zurückgegeben. Bei Erfolg gibt die Funktion eines von drei möglichen Ergebnissen zurück:

- `1`, wenn `var1` größer als `var2` ist
- `-1`, wenn `var1` kleiner als `var2` ist
- `0`, wenn `var1` und `var2` gleich sind

`PGTYPESnumeric_from_int`

Konvertiert eine Variable vom Typ `int` in eine `numeric`-Variable.

```c
int PGTYPESnumeric_from_int(signed int int_val, numeric *var);
```

Diese Funktion nimmt eine Variable vom Typ `signed int` entgegen und speichert sie in der `numeric`-Variable `var`. Bei Erfolg wird 0 zurückgegeben, bei einem Fehler -1.

`PGTYPESnumeric_from_long`

Konvertiert eine Variable vom Typ `long int` in eine `numeric`-Variable.

```c
int PGTYPESnumeric_from_long(signed long int long_val, numeric
 *var);
```

Diese Funktion nimmt eine Variable vom Typ `signed long int` entgegen und speichert sie in der `numeric`-Variable `var`. Bei Erfolg wird 0 zurückgegeben, bei einem Fehler -1.

`PGTYPESnumeric_copy`

Kopiert eine `numeric`-Variable in eine andere.

```c
int PGTYPESnumeric_copy(numeric *src, numeric *dst);
```

Diese Funktion kopiert den Wert der Variable, auf die `src` zeigt, in die Variable, auf die `dst` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler -1.

`PGTYPESnumeric_from_double`

Konvertiert eine Variable vom Typ `double` in `numeric`.

```c
int    PGTYPESnumeric_from_double(double d, numeric *dst);
```

Diese Funktion nimmt eine Variable vom Typ `double` entgegen und speichert das Ergebnis in der Variable, auf die `dst` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler -1.

`PGTYPESnumeric_to_double`

Konvertiert eine Variable vom Typ `numeric` in `double`.

```c
int PGTYPESnumeric_to_double(numeric *nv, double *dp)
```

Die Funktion konvertiert den `numeric`-Wert aus der Variable, auf die `nv` zeigt, in die `double`-Variable, auf die `dp` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einschließlich Überlauf -1. Bei einem Überlauf wird außerdem die globale Variable `errno` auf `PGTYPES_NUM_OVERFLOW` gesetzt.

`PGTYPESnumeric_to_int`

Konvertiert eine Variable vom Typ `numeric` in `int`.

```c
int PGTYPESnumeric_to_int(numeric *nv, int *ip);
```

Die Funktion konvertiert den `numeric`-Wert aus der Variable, auf die `nv` zeigt, in die Ganzzahlvariable, auf die `ip` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einschließlich Überlauf -1. Bei einem Überlauf wird außerdem die globale Variable `errno` auf `PGTYPES_NUM_OVERFLOW` gesetzt.

`PGTYPESnumeric_to_long`

Konvertiert eine Variable vom Typ `numeric` in `long`.

```c
int PGTYPESnumeric_to_long(numeric *nv, long *lp);
```

Die Funktion konvertiert den `numeric`-Wert aus der Variable, auf die `nv` zeigt, in die `long`-Ganzzahlvariable, auf die `lp` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einschließlich Überlauf und Unterlauf -1. Bei einem Überlauf wird die globale Variable `errno` auf `PGTYPES_NUM_OVERFLOW` gesetzt, bei einem Unterlauf auf `PGTYPES_NUM_UNDERFLOW`.

`PGTYPESnumeric_to_decimal`

Konvertiert eine Variable vom Typ `numeric` in `decimal`.

```c
int PGTYPESnumeric_to_decimal(numeric *src, decimal *dst);
```

Die Funktion konvertiert den `numeric`-Wert aus der Variable, auf die `src` zeigt, in die `decimal`-Variable, auf die `dst` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einschließlich Überlauf -1. Bei einem Überlauf wird außerdem die globale Variable `errno` auf `PGTYPES_NUM_OVERFLOW` gesetzt.

`PGTYPESnumeric_from_decimal`

Konvertiert eine Variable vom Typ `decimal` in `numeric`.

```c
int PGTYPESnumeric_from_decimal(decimal *src, numeric *dst);
```

Die Funktion konvertiert den `decimal`-Wert aus der Variable, auf die `src` zeigt, in die `numeric`-Variable, auf die `dst` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler -1. Da der Typ `decimal` als beschränkte Version des Typs `numeric` implementiert ist, kann bei dieser Konvertierung kein Überlauf auftreten.

### 34.6.3. Der Typ `date`

Der Typ `date` in C ermöglicht Programmen den Umgang mit Daten des SQL-Typs `date`. Den entsprechenden Typ im PostgreSQL-Server beschreibt [Abschnitt 8.5](08_Datentypen.md#85-datumszeittypen).

Die folgenden Funktionen können verwendet werden, um mit dem Typ `date` zu arbeiten:

`PGTYPESdate_from_timestamp`

Extrahiert den Datumsteil aus einem Zeitstempel.

```c
date PGTYPESdate_from_timestamp(timestamp dt);
```

Die Funktion erhält einen Zeitstempel als einziges Argument und gibt den daraus extrahierten Datumsteil zurück.

`PGTYPESdate_from_asc`

Parst ein Datum aus seiner Textdarstellung.

```c
date PGTYPESdate_from_asc(char *str, char **endptr);
```

Die Funktion erhält die C-Zeichenkette `str` vom Typ `char *` und einen Zeiger `endptr` auf eine C-Zeichenkette vom Typ `char *`. ECPG parst derzeit immer die vollständige Zeichenkette und unterstützt daher aktuell nicht, die Adresse des ersten ungültigen Zeichens in `*endptr` zu speichern. Sie können `endptr` gefahrlos auf `NULL` setzen.

Beachten Sie, dass die Funktion immer Datumsangaben im MDY-Format annimmt; derzeit gibt es in ECPG keine Variable, um das zu ändern.

Tabelle 34.2 zeigt die zulässigen Eingabeformate.

Tabelle 34.2. Gültige Eingabeformate für `PGTYPESdate_from_asc`

| Eingabe | Ergebnis |
| --- | --- |
| `January 8, 1999` | January 8, 1999 |
| `1999-01-08` | January 8, 1999 |
| `1/8/1999` | January 8, 1999 |
| `1/18/1999` | January 18, 1999 |
| `01/02/03` | February 1, 2003 |
| `1999-Jan-08` | January 8, 1999 |
| `Jan-08-1999` | January 8, 1999 |
| `08-Jan-1999` | January 8, 1999 |
| `99-Jan-08` | January 8, 1999 |
| `08-Jan-99` | January 8, 1999 |
| `08-Jan-06` | January 8, 2006 |
| `Jan-08-99` | January 8, 1999 |
| `19990108` | ISO 8601; January 8, 1999 |
| `990108` | ISO 8601; January 8, 1999 |
| `1999.008` | Jahr und Tag des Jahres |
| `J2451187` | Julianischer Tag |
| `January 8, 99 BC` | Jahr 99 vor unserer Zeitrechnung |

`PGTYPESdate_to_asc`

Gibt die Textdarstellung einer `date`-Variable zurück.

```c
char *PGTYPESdate_to_asc(date dDate);
```

Die Funktion erhält das Datum `dDate` als einzigen Parameter. Sie gibt das Datum in der Form `1999-01-18` aus, also im Format `YYYY-MM-DD`. Das Ergebnis muss mit `PGTYPESchar_free()` freigegeben werden.

`PGTYPESdate_julmdy`

Extrahiert die Werte für Tag, Monat und Jahr aus einer Variable vom Typ `date`.

```c
void PGTYPESdate_julmdy(date d, int *mdy);
```

Die Funktion erhält das Datum `d` und einen Zeiger auf ein Array aus drei Ganzzahlwerten `mdy`. Der Variablenname gibt die Reihenfolge an: `mdy[0]` wird auf die Monatsnummer gesetzt, `mdy[1]` auf den Tag und `mdy[2]` enthält das Jahr.

`PGTYPESdate_mdyjul`

Erzeugt einen Datumswert aus einem Array von drei Ganzzahlen, die Tag, Monat und Jahr des Datums angeben.

```c
void PGTYPESdate_mdyjul(int *mdy, date *jdate);
```

Die Funktion erhält als erstes Argument das Array aus drei Ganzzahlen (`mdy`) und als zweites Argument einen Zeiger auf eine Variable vom Typ `date`, die das Ergebnis der Operation aufnehmen soll.

`PGTYPESdate_dayofweek`

Gibt eine Zahl zurück, die den Wochentag eines Datumswerts darstellt.

```c
int PGTYPESdate_dayofweek(date d);
```

Die Funktion erhält die Datumsvariable `d` als einziges Argument und gibt eine Ganzzahl zurück, die den Wochentag dieses Datums angibt.

- `0` - Sonntag
- `1` - Montag
- `2` - Dienstag
- `3` - Mittwoch
- `4` - Donnerstag
- `5` - Freitag
- `6` - Samstag

`PGTYPESdate_today`

Ermittelt das aktuelle Datum.

```c
void PGTYPESdate_today(date *d);
```

Die Funktion erhält einen Zeiger auf eine `date`-Variable (`d`), die sie auf das aktuelle Datum setzt.

`PGTYPESdate_fmt_asc`

Konvertiert eine Variable vom Typ `date` mithilfe einer Formatmaske in ihre Textdarstellung.

```c
int PGTYPESdate_fmt_asc(date dDate, char *fmtstring, char
 *outbuf);
```

Die Funktion erhält das zu konvertierende Datum (`dDate`), die Formatmaske (`fmtstring`) und die Zeichenkette, die die Textdarstellung des Datums aufnehmen soll (`outbuf`).

Bei Erfolg wird 0 zurückgegeben, bei einem Fehler ein negativer Wert.

Die folgenden Literale sind die Feldbezeichner, die Sie verwenden können:

- `dd` - die Nummer des Tags im Monat
- `mm` - die Nummer des Monats im Jahr
- `yy` - die Jahreszahl als zweistellige Zahl
- `yyyy` - die Jahreszahl als vierstellige Zahl
- `ddd` - der Name des Tages, abgekürzt
- `mmm` - der Name des Monats, abgekürzt

Alle anderen Zeichen werden 1:1 in die Ausgabezeichenkette kopiert.

Tabelle 34.3 zeigt einige mögliche Formate. Sie vermittelt einen Eindruck davon, wie diese Funktion verwendet wird. Alle Ausgabezeilen beruhen auf demselben Datum: dem 23. November 1959.

Tabelle 34.3. Gültige Eingabeformate für `PGTYPESdate_fmt_asc`

| Format | Ergebnis |
| --- | --- |
| `mmddyy` | `112359` |
| `ddmmyy` | `231159` |
| `yymmdd` | `591123` |
| `yy/mm/dd` | `59/11/23` |
| `yy mm dd` | `59 11 23` |
| `yy.mm.dd` | `59.11.23` |
| `.mm.yyyy.dd.` | `.11.1959.23.` |
| `mmm. dd, yyyy` | `Nov. 23, 1959` |
| `mmm dd yyyy` | `Nov 23 1959` |
| `yyyy dd mm` | `1959 23 11` |
| `ddd, mmm. dd, yyyy` | `Mon, Nov. 23, 1959` |
| `(ddd) mmm. dd, yyyy` | `(Mon) Nov. 23, 1959` |

`PGTYPESdate_defmt_asc`

Verwendet eine Formatmaske, um eine C-Zeichenkette vom Typ `char *` in einen Wert vom Typ `date` zu konvertieren.

```c
int PGTYPESdate_defmt_asc(date *d, char *fmt, char *str);
```

Die Funktion erhält einen Zeiger auf den Datumswert, der das Ergebnis der Operation aufnehmen soll (`d`), die Formatmaske zum Parsen des Datums (`fmt`) und die C-Zeichenkette vom Typ `char *`, die die Textdarstellung des Datums enthält (`str`). Es wird erwartet, dass die Textdarstellung zur Formatmaske passt. Sie benötigen jedoch keine 1:1-Abbildung der Zeichenkette auf die Formatmaske. Die Funktion analysiert nur die Reihenfolge und sucht nach den Literalen `yy` oder `yyyy`, die die Position des Jahres angeben, nach `mm` für die Position des Monats und nach `dd` für die Position des Tages.

Tabelle 34.4 zeigt einige mögliche Formate. Sie vermittelt einen Eindruck davon, wie diese Funktion verwendet wird.

Tabelle 34.4. Gültige Eingabeformate für `rdefmtdate`

| Format | Zeichenkette | Ergebnis |
| --- | --- | --- |
| `ddmmyy` | `21-2-54` | `1954-02-21` |
| `ddmmyy` | `2-12-54` | `1954-12-02` |
| `ddmmyy` | `20111954` | `1954-11-20` |
| `ddmmyy` | `130464` | `1964-04-13` |
| `mmm.dd.yyyy` | `MAR-12-1967` | `1967-03-12` |
| `yy/mm/dd` | `1954, February 3rd` | `1954-02-03` |
| `mmm.dd.yyyy` | `041269` | `1969-04-12` |
| `yy/mm/dd` | `In the year 2525, in the month of July, mankind will be alive on the 28th day` | `2525-07-28` |
| `dd-mm-yy` | `I said on the 28th of July in the year` | `2525-07-28` |
| `mmm.dd.yyyy` | `9/14/58` | `1958-09-14` |
| `yy/mm/dd` | `47/03/29` | `1947-03-29` |
| `mmm.dd.yyyy` | `oct 28 1975` | `1975-10-28` |
| `mmddyy` | `Nov 14th, 1985` | `1985-11-14` |

### 34.6.4. Der Typ `timestamp`

Der Typ `timestamp` in C ermöglicht Programmen den Umgang mit Daten des SQL-Typs `timestamp`. Den entsprechenden Typ im PostgreSQL-Server beschreibt [Abschnitt 8.5](08_Datentypen.md#85-datumszeittypen).

Die folgenden Funktionen können verwendet werden, um mit dem Typ `timestamp` zu arbeiten:

`PGTYPEStimestamp_from_asc`

Parst einen Zeitstempel aus seiner Textdarstellung in eine `timestamp`-Variable.

```c
timestamp PGTYPEStimestamp_from_asc(char *str, char **endptr);
```

Die Funktion erhält die zu parsende Zeichenkette (`str`) und einen Zeiger auf ein C-`char *` (`endptr`). ECPG parst derzeit immer die vollständige Zeichenkette und unterstützt daher aktuell nicht, die Adresse des ersten ungültigen Zeichens in `*endptr` zu speichern. Sie können `endptr` gefahrlos auf `NULL` setzen.

Bei Erfolg gibt die Funktion den geparsten Zeitstempel zurück. Bei einem Fehler wird `PGTYPESInvalidTimestamp` zurückgegeben und `errno` auf `PGTYPES_TS_BAD_TIMESTAMP` gesetzt. Beachten Sie die wichtigen Hinweise zu diesem Wert bei `PGTYPESInvalidTimestamp`.

Im Allgemeinen kann die Eingabezeichenkette jede Kombination aus einer zulässigen Datumsangabe, einem Leerraumzeichen und einer zulässigen Zeitangabe enthalten. Beachten Sie, dass Zeitzonen von ECPG nicht unterstützt werden. ECPG kann sie parsen, wendet aber keine Berechnungen an, wie es beispielsweise der PostgreSQL-Server tut. Zeitzonenangaben werden stillschweigend verworfen.

Tabelle 34.5 enthält einige Beispiele für Eingabezeichenketten.

Tabelle 34.5. Gültige Eingabeformate für `PGTYPEStimestamp_from_asc`

| Eingabe | Ergebnis |
| --- | --- |
| `1999-01-08 04:05:06` | `1999-01-08 04:05:06` |
| `January 8 04:05:06 1999 PST` | `1999-01-08 04:05:06` |
| `1999-Jan-08 04:05:06.789-8` | `1999-01-08 04:05:06.789` (Zeitzonenangabe ignoriert) |
| `J2451187 04:05-08:00` | `1999-01-08 04:05:00` (Zeitzonenangabe ignoriert) |

`PGTYPEStimestamp_to_asc`

Konvertiert ein Datum in eine C-Zeichenkette vom Typ `char *`.

```c
char *PGTYPEStimestamp_to_asc(timestamp tstamp);
```

Die Funktion erhält den Zeitstempel `tstamp` als einziges Argument und gibt eine allozierte Zeichenkette zurück, die die Textdarstellung des Zeitstempels enthält. Das Ergebnis muss mit `PGTYPESchar_free()` freigegeben werden.

`PGTYPEStimestamp_current`

Ruft den aktuellen Zeitstempel ab.

```c
void PGTYPEStimestamp_current(timestamp *ts);
```

Die Funktion ruft den aktuellen Zeitstempel ab und speichert ihn in der `timestamp`-Variable, auf die `ts` zeigt.

`PGTYPEStimestamp_fmt_asc`

Konvertiert eine `timestamp`-Variable mithilfe einer Formatmaske in ein C-`char *`.

```c
int PGTYPEStimestamp_fmt_asc(timestamp *ts, char *output, int
 str_len, char *fmtstr);
```

Die Funktion erhält als erstes Argument einen Zeiger auf den zu konvertierenden Zeitstempel (`ts`), einen Zeiger auf den Ausgabepuffer (`output`), die maximale Länge, die für den Ausgabepuffer alloziert wurde (`str_len`), und die Formatmaske für die Konvertierung (`fmtstr`).

Bei Erfolg gibt die Funktion 0 zurück, bei einem Fehler einen negativen Wert.

Für die Formatmaske können Sie die folgenden Formatbezeichner verwenden. Es sind dieselben Formatbezeichner, die in der Funktion `strftime` aus `libc` verwendet werden. Jeder Nicht-Formatbezeichner wird in den Ausgabepuffer kopiert.

- `%A` - wird durch die landesspezifische Darstellung des vollständigen Wochentagsnamens ersetzt
- `%a` - wird durch die landesspezifische Darstellung des abgekürzten Wochentagsnamens ersetzt
- `%B` - wird durch die landesspezifische Darstellung des vollständigen Monatsnamens ersetzt
- `%b` - wird durch die landesspezifische Darstellung des abgekürzten Monatsnamens ersetzt
- `%C` - wird durch `(year / 100)` als Dezimalzahl ersetzt; einstellige Werte werden mit einer Null aufgefüllt
- `%c` - wird durch die landesspezifische Darstellung von Uhrzeit und Datum ersetzt
- `%D` - entspricht `%m/%d/%y`
- `%d` - wird durch den Tag des Monats als Dezimalzahl ersetzt (`01` bis `31`)
- `%E*` `%O*` - POSIX-Locale-Erweiterungen. Die Sequenzen `%Ec`, `%EC`, `%Ex`, `%EX`, `%Ey`, `%EY`, `%Od`, `%Oe`, `%OH`, `%OI`, `%Om`, `%OM`, `%OS`, `%Ou`, `%OU`, `%OV`, `%Ow`, `%OW` und `%Oy` sollen alternative Darstellungen bereitstellen. Zusätzlich ist `%OB` implementiert, um alternative Monatsnamen darzustellen, wenn sie alleinstehend ohne genannten Tag verwendet werden.
- `%e` - wird durch den Tag des Monats als Dezimalzahl ersetzt (`1` bis `31`); einstellige Werte werden mit einem Leerzeichen aufgefüllt
- `%F` - entspricht `%Y-%m-%d`
- `%G` - wird durch ein Jahr mit Jahrhundert als Dezimalzahl ersetzt. Dieses Jahr ist dasjenige, das den größeren Teil der Woche enthält, wobei Montag als erster Tag der Woche gilt.
- `%g` - wird durch dasselbe Jahr wie `%G` ersetzt, jedoch als Dezimalzahl ohne Jahrhundert (`00` bis `99`)
- `%H` - wird durch die Stunde im 24-Stunden-Format als Dezimalzahl ersetzt (`00` bis `23`)
- `%h` - dasselbe wie `%b`
- `%I` - wird durch die Stunde im 12-Stunden-Format als Dezimalzahl ersetzt (`01` bis `12`)
- `%j` - wird durch den Tag des Jahres als Dezimalzahl ersetzt (`001` bis `366`)
- `%k` - wird durch die Stunde im 24-Stunden-Format als Dezimalzahl ersetzt (`0` bis `23`); einstellige Werte werden mit einem Leerzeichen aufgefüllt
- `%l` - wird durch die Stunde im 12-Stunden-Format als Dezimalzahl ersetzt (`1` bis `12`); einstellige Werte werden mit einem Leerzeichen aufgefüllt
- `%M` - wird durch die Minute als Dezimalzahl ersetzt (`00` bis `59`)
- `%m` - wird durch den Monat als Dezimalzahl ersetzt (`01` bis `12`)
- `%n` - wird durch einen Zeilenumbruch ersetzt
- `%O*` - dasselbe wie `%E*`
- `%p` - wird je nach Bedarf durch die landesspezifische Darstellung von „ante meridiem“ oder „post meridiem“ ersetzt
- `%R` - entspricht `%H:%M`
- `%r` - entspricht `%I:%M:%S %p`
- `%S` - wird durch die Sekunde als Dezimalzahl ersetzt (`00` bis `60`)
- `%s` - wird durch die Anzahl der Sekunden seit der Epoche in UTC ersetzt
- `%T` - entspricht `%H:%M:%S`
- `%t` - wird durch einen Tabulator ersetzt
- `%U` - wird durch die Wochennummer des Jahres als Dezimalzahl ersetzt (`00` bis `53`), wobei Sonntag als erster Tag der Woche gilt
- `%u` - wird durch den Wochentag als Dezimalzahl ersetzt (`1` bis `7`), wobei Montag als erster Tag der Woche gilt
- `%V` - wird durch die Wochennummer des Jahres als Dezimalzahl ersetzt (`01` bis `53`), wobei Montag als erster Tag der Woche gilt. Wenn die Woche, die den 1. Januar enthält, vier oder mehr Tage im neuen Jahr hat, ist sie Woche 1; andernfalls ist sie die letzte Woche des vorherigen Jahres, und die nächste Woche ist Woche 1.
- `%v` - entspricht `%e-%b-%Y`
- `%W` - wird durch die Wochennummer des Jahres als Dezimalzahl ersetzt (`00` bis `53`), wobei Montag als erster Tag der Woche gilt
- `%w` - wird durch den Wochentag als Dezimalzahl ersetzt (`0` bis `6`), wobei Sonntag als erster Tag der Woche gilt
- `%X` - wird durch die landesspezifische Darstellung der Uhrzeit ersetzt
- `%x` - wird durch die landesspezifische Darstellung des Datums ersetzt
- `%Y` - wird durch das Jahr mit Jahrhundert als Dezimalzahl ersetzt
- `%y` - wird durch das Jahr ohne Jahrhundert als Dezimalzahl ersetzt (`00` bis `99`)
- `%Z` - wird durch den Zeitzonennamen ersetzt
- `%z` - wird durch den Zeitzonenversatz gegenüber UTC ersetzt; ein führendes Pluszeichen steht für östlich von UTC, ein Minuszeichen für westlich von UTC, Stunden und Minuten folgen jeweils zweistellig und ohne Trennzeichen dazwischen, wie in der üblichen Form für Datums-Header nach RFC 822
- `%+` - wird durch die landesspezifische Darstellung von Datum und Uhrzeit ersetzt
- `%-*` - GNU-libc-Erweiterung; bei numerischen Ausgaben wird keine Auffüllung vorgenommen
- `$_*` - GNU-libc-Erweiterung; gibt explizit Leerzeichen zum Auffüllen an
- `%0*` - GNU-libc-Erweiterung; gibt explizit Nullen zum Auffüllen an
- `%%` - wird durch `%` ersetzt

Siehe auch <https://datatracker.ietf.org/doc/html/rfc822>.

`PGTYPEStimestamp_sub`

Subtrahiert einen Zeitstempel von einem anderen und speichert das Ergebnis in einer Variable vom Typ `interval`.

```c
int PGTYPEStimestamp_sub(timestamp *ts1, timestamp *ts2,
 interval *iv);
```

Die Funktion subtrahiert die `timestamp`-Variable, auf die `ts2` zeigt, von der `timestamp`-Variable, auf die `ts1` zeigt, und speichert das Ergebnis in der `interval`-Variable, auf die `iv` zeigt.

Bei Erfolg gibt die Funktion 0 zurück, bei einem Fehler einen negativen Wert.

`PGTYPEStimestamp_defmt_asc`

Parst einen Zeitstempelwert aus seiner Textdarstellung unter Verwendung einer Formatmaske.

```c
int PGTYPEStimestamp_defmt_asc(char *str, char *fmt, timestamp
 *d);
```

Die Funktion erhält die Textdarstellung eines Zeitstempels in der Variable `str` sowie die zu verwendende Formatmaske in der Variable `fmt`. Das Ergebnis wird in der Variable gespeichert, auf die `d` zeigt.

Wenn die Formatmaske `fmt` `NULL` ist, verwendet die Funktion die Standardformatmaske `%Y-%m-%d %H:%M:%S`.

Dies ist die Umkehrfunktion zu `PGTYPEStimestamp_fmt_asc`. Die möglichen Einträge der Formatmaske finden Sie in der dortigen Dokumentation.

`PGTYPEStimestamp_add_interval`

Addiert eine `interval`-Variable zu einer `timestamp`-Variable.

```c
int PGTYPEStimestamp_add_interval(timestamp *tin, interval
 *span, timestamp *tout);
```

Die Funktion erhält einen Zeiger auf eine `timestamp`-Variable `tin` und einen Zeiger auf eine `interval`-Variable `span`. Sie addiert das Intervall zum Zeitstempel und speichert den resultierenden Zeitstempel in der Variable, auf die `tout` zeigt.

Bei Erfolg gibt die Funktion 0 zurück, bei einem Fehler einen negativen Wert.

`PGTYPEStimestamp_sub_interval`

Subtrahiert eine `interval`-Variable von einer `timestamp`-Variable.

```c
int PGTYPEStimestamp_sub_interval(timestamp *tin, interval
 *span, timestamp *tout);
```

Die Funktion subtrahiert die `interval`-Variable, auf die `span` zeigt, von der `timestamp`-Variable, auf die `tin` zeigt, und speichert das Ergebnis in der Variable, auf die `tout` zeigt.

Bei Erfolg gibt die Funktion 0 zurück, bei einem Fehler einen negativen Wert.

### 34.6.5. Der Typ `interval`

Der Typ `interval` in C ermöglicht Programmen den Umgang mit Daten des SQL-Typs `interval`. Den entsprechenden Typ im PostgreSQL-Server beschreibt [Abschnitt 8.5](08_Datentypen.md#85-datumszeittypen).

Die folgenden Funktionen können verwendet werden, um mit dem Typ `interval` zu arbeiten:

`PGTYPESinterval_new`

Gibt einen Zeiger auf eine neu allozierte `interval`-Variable zurück.

```c
interval *PGTYPESinterval_new(void);
```

`PGTYPESinterval_free`

Gibt den Speicher einer zuvor allozierten `interval`-Variable frei.

```c
void PGTYPESinterval_free(interval *intvl);
```

`PGTYPESinterval_from_asc`

Parst ein Intervall aus seiner Textdarstellung.

```c
interval *PGTYPESinterval_from_asc(char *str, char **endptr);
```

Die Funktion parst die Eingabezeichenkette `str` und gibt einen Zeiger auf eine allozierte `interval`-Variable zurück. ECPG parst derzeit immer die vollständige Zeichenkette und unterstützt daher aktuell nicht, die Adresse des ersten ungültigen Zeichens in `*endptr` zu speichern. Sie können `endptr` gefahrlos auf `NULL` setzen.

`PGTYPESinterval_to_asc`

Konvertiert eine Variable vom Typ `interval` in ihre Textdarstellung.

```c
char *PGTYPESinterval_to_asc(interval *span);
```

Die Funktion konvertiert die `interval`-Variable, auf die `span` zeigt, in ein C-`char *`. Die Ausgabe sieht zum Beispiel so aus: `@ 1 day 12 hours 59 mins 10 secs`. Das Ergebnis muss mit `PGTYPESchar_free()` freigegeben werden.

`PGTYPESinterval_copy`

Kopiert eine Variable vom Typ `interval`.

```c
int PGTYPESinterval_copy(interval *intvlsrc, interval
 *intvldest);
```

Die Funktion kopiert die `interval`-Variable, auf die `intvlsrc` zeigt, in die Variable, auf die `intvldest` zeigt. Beachten Sie, dass Sie den Speicher für die Zielvariable vorher allozieren müssen.

### 34.6.6. Der Typ `decimal`

Der Typ `decimal` ähnelt dem Typ `numeric`. Er ist jedoch auf eine maximale Genauigkeit von 30 signifikanten Ziffern beschränkt. Im Gegensatz zum Typ `numeric`, der nur auf dem Heap erzeugt werden kann, kann der Typ `decimal` entweder auf dem Stack oder auf dem Heap erzeugt werden, letzteres mit den Funktionen `PGTYPESdecimal_new` und `PGTYPESdecimal_free`. Viele weitere Funktionen, die mit dem Typ `decimal` arbeiten, befinden sich im Informix-Kompatibilitätsmodus, der in [Abschnitt 34.15](34_ECPG_Embedded_SQL_in_C.md#3415-informixkompatibilitätsmodus) beschrieben wird.

Die folgenden Funktionen können verwendet werden, um mit dem Typ `decimal` zu arbeiten, und sind nicht nur in der Bibliothek `libcompat` enthalten.

`PGTYPESdecimal_new`

Fordert einen Zeiger auf eine neu allozierte `decimal`-Variable an.

```c
decimal *PGTYPESdecimal_new(void);
```

`PGTYPESdecimal_free`

Gibt einen `decimal`-Typ frei und gibt seinen gesamten Speicher zurück.

```c
void PGTYPESdecimal_free(decimal *var);
```

### 34.6.7. `errno`-Werte von `pgtypeslib`

`PGTYPES_NUM_BAD_NUMERIC`

Ein Argument sollte eine `numeric`-Variable enthalten oder auf eine `numeric`-Variable zeigen, aber seine speicherinterne Darstellung war ungültig.

`PGTYPES_NUM_OVERFLOW`

Es ist ein Überlauf aufgetreten. Da der Typ `numeric` mit nahezu beliebiger Genauigkeit umgehen kann, kann das Konvertieren einer `numeric`-Variable in andere Typen zu einem Überlauf führen.

`PGTYPES_NUM_UNDERFLOW`

Es ist ein Unterlauf aufgetreten. Da der Typ `numeric` mit nahezu beliebiger Genauigkeit umgehen kann, kann das Konvertieren einer `numeric`-Variable in andere Typen zu einem Unterlauf führen.

`PGTYPES_NUM_DIVIDE_ZERO`

Es wurde versucht, durch null zu dividieren.

`PGTYPES_DATE_BAD_DATE`

Der Funktion `PGTYPESdate_from_asc` wurde eine ungültige Datumszeichenkette übergeben.

`PGTYPES_DATE_ERR_EARGS`

Der Funktion `PGTYPESdate_defmt_asc` wurden ungültige Argumente übergeben.

`PGTYPES_DATE_ERR_ENOSHORTDATE`

Die Funktion `PGTYPESdate_defmt_asc` hat in der Eingabezeichenkette ein ungültiges Token gefunden.

`PGTYPES_INTVL_BAD_INTERVAL`

Der Funktion `PGTYPESinterval_from_asc` wurde eine ungültige Intervallzeichenkette übergeben, oder der Funktion `PGTYPESinterval_to_asc` wurde ein ungültiger Intervallwert übergeben.

`PGTYPES_DATE_ERR_ENOTDMY`

In der Funktion `PGTYPESdate_defmt_asc` gab es eine Nichtübereinstimmung in der Zuordnung von Tag, Monat und Jahr.

`PGTYPES_DATE_BAD_DAY`

Die Funktion `PGTYPESdate_defmt_asc` hat einen ungültigen Wert für den Tag des Monats gefunden.

`PGTYPES_DATE_BAD_MONTH`

Die Funktion `PGTYPESdate_defmt_asc` hat einen ungültigen Monatswert gefunden.

`PGTYPES_TS_BAD_TIMESTAMP`

Der Funktion `PGTYPEStimestamp_from_asc` wurde eine ungültige Zeitstempelzeichenkette übergeben, oder der Funktion `PGTYPEStimestamp_to_asc` wurde ein ungültiger Zeitstempelwert übergeben.

`PGTYPES_TS_ERR_EINFTIME`

In einem Kontext, der ihn nicht verarbeiten kann, wurde ein unendlicher Zeitstempelwert gefunden.

### 34.6.8. Besondere Konstanten von `pgtypeslib`

`PGTYPESInvalidTimestamp`

Ein Wert vom Typ `timestamp`, der einen ungültigen Zeitstempel darstellt. Er wird von der Funktion `PGTYPEStimestamp_from_asc` bei einem Parse-Fehler zurückgegeben. Beachten Sie, dass `PGTYPESInvalidTimestamp` wegen der internen Darstellung des Datentyps `timestamp` zugleich auch ein gültiger Zeitstempel ist. Der Wert ist auf `1899-12-31 23:59:59` gesetzt. Um Fehler zu erkennen, stellen Sie sicher, dass Ihre Anwendung nach jedem Aufruf von `PGTYPEStimestamp_from_asc` nicht nur auf `PGTYPESInvalidTimestamp` prüft, sondern auch auf `errno != 0`.

## 34.7. Descriptor-Bereiche verwenden

Ein SQL-Descriptor-Bereich ist eine anspruchsvollere Methode, um das Ergebnis einer `SELECT`-, `FETCH`- oder `DESCRIBE`-Anweisung zu verarbeiten. Ein SQL-Descriptor-Bereich fasst die Daten einer Ergebniszeile zusammen mit Metadaten in einer Datenstruktur zusammen. Die Metadaten sind besonders nützlich, wenn dynamische SQL-Anweisungen ausgeführt werden, bei denen die Art der Ergebnisspalten möglicherweise nicht im Voraus bekannt ist. PostgreSQL stellt zwei Möglichkeiten bereit, Descriptor-Bereiche zu verwenden: benannte SQL-Descriptor-Bereiche und SQLDAs als C-Strukturen.

### 34.7.1. Benannte SQL-Descriptor-Bereiche

Ein benannter SQL-Descriptor-Bereich besteht aus einem Header, der Informationen zum gesamten Descriptor enthält, und einem oder mehreren Item-Descriptor-Bereichen, die im Wesentlichen jeweils eine Spalte in der Ergebniszeile beschreiben.

Bevor Sie einen SQL-Descriptor-Bereich verwenden können, müssen Sie ihn allozieren:

```sql
EXEC SQL ALLOCATE DESCRIPTOR identifier;
```

Der Bezeichner dient als „Variablenname“ des Descriptor-Bereichs. Wenn Sie den Descriptor nicht mehr benötigen, sollten Sie ihn freigeben:

```sql
EXEC SQL DEALLOCATE DESCRIPTOR identifier;
```

Um einen Descriptor-Bereich zu verwenden, geben Sie ihn in einer `INTO`-Klausel als Speicherziel an, statt Hostvariablen aufzulisten:

```sql
EXEC SQL FETCH NEXT FROM mycursor INTO SQL DESCRIPTOR mydesc;
```

Wenn die Ergebnismenge leer ist, enthält der Descriptor-Bereich trotzdem noch die Metadaten aus der Abfrage, also die Feldnamen.

Für vorbereitete, aber noch nicht ausgeführte Abfragen kann die Anweisung `DESCRIBE` verwendet werden, um die Metadaten der Ergebnismenge zu erhalten:

```c
EXEC SQL BEGIN DECLARE SECTION;
char *sql_stmt = "SELECT * FROM table1";
EXEC SQL END DECLARE SECTION;

EXEC SQL PREPARE stmt1 FROM :sql_stmt;
EXEC SQL DESCRIBE stmt1 INTO SQL DESCRIPTOR mydesc;
```

Vor PostgreSQL 9.0 war das Schlüsselwort `SQL` optional, sodass sowohl `DESCRIPTOR` als auch `SQL DESCRIPTOR` benannte SQL-Descriptor-Bereiche erzeugten. Jetzt ist es obligatorisch; wenn das Schlüsselwort `SQL` weggelassen wird, werden SQLDA-Descriptor-Bereiche erzeugt, siehe [Abschnitt 34.7.2](#3472-sqldadescriptorbereiche).

In `DESCRIBE`- und `FETCH`-Anweisungen können die Schlüsselwörter `INTO` und `USING` ähnlich verwendet werden: Sie erzeugen die Ergebnismenge und die Metadaten in einem Descriptor-Bereich.

Wie kommen die Daten nun aus dem Descriptor-Bereich heraus? Man kann sich den Descriptor-Bereich als Struktur mit benannten Feldern vorstellen. Um den Wert eines Felds aus dem Header abzurufen und in einer Hostvariable zu speichern, verwenden Sie folgenden Befehl:

```sql
EXEC SQL GET DESCRIPTOR name :hostvar = field;
```

Derzeit ist nur ein Header-Feld definiert: `COUNT`. Es gibt an, wie viele Item-Descriptor-Bereiche existieren, also wie viele Spalten im Ergebnis enthalten sind. Die Hostvariable muss einen Ganzzahltyp haben. Um ein Feld aus dem Item-Descriptor-Bereich abzurufen, verwenden Sie folgenden Befehl:

```sql
EXEC SQL GET DESCRIPTOR name VALUE num :hostvar = field;
```

`num` kann ein Ganzzahlliteral oder eine Hostvariable sein, die eine Ganzzahl enthält. Mögliche Felder sind:

`CARDINALITY` (`integer`)

Anzahl der Zeilen in der Ergebnismenge.

`DATA`

Das eigentliche Datenelement; der Datentyp dieses Felds hängt daher von der Abfrage ab.

`DATETIME_INTERVAL_CODE` (`integer`)

Wenn `TYPE` den Wert 9 hat, hat `DATETIME_INTERVAL_CODE` den Wert 1 für `DATE`, 2 für `TIME`, 3 für `TIMESTAMP`, 4 für `TIME WITH TIME ZONE` oder 5 für `TIMESTAMP WITH TIME ZONE`.

`DATETIME_INTERVAL_PRECISION` (`integer`)

Nicht implementiert.

`INDICATOR` (`integer`)

Der Indikator, der einen Nullwert oder eine Wertabschneidung anzeigt.

`KEY_MEMBER` (`integer`)

Nicht implementiert.

`LENGTH` (`integer`)

Länge des Datums in Zeichen.

`NAME` (`string`)

Name der Spalte.

`NULLABLE` (`integer`)

Nicht implementiert.

`OCTET_LENGTH` (`integer`)

Länge der Zeichendarstellung des Datums in Byte.

`PRECISION` (`integer`)

Genauigkeit, für den Typ `numeric`.

`RETURNED_LENGTH` (`integer`)

Länge des Datums in Zeichen.

`RETURNED_OCTET_LENGTH` (`integer`)

Länge der Zeichendarstellung des Datums in Byte.

`SCALE` (`integer`)

Skalierung, für den Typ `numeric`.

`TYPE` (`integer`)

Numerischer Code des Datentyps der Spalte.

In `EXECUTE`-, `DECLARE`- und `OPEN`-Anweisungen ist die Wirkung der Schlüsselwörter `INTO` und `USING` unterschiedlich. Ein Descriptor-Bereich kann auch manuell aufgebaut werden, um Eingabeparameter für eine Abfrage oder einen Cursor bereitzustellen. `USING SQL DESCRIPTOR name` ist der Weg, um die Eingabeparameter an eine parametrisierte Abfrage zu übergeben. Die Anweisung zum Aufbau eines benannten SQL-Descriptor-Bereichs lautet:

```sql
EXEC SQL SET DESCRIPTOR name VALUE num field = :hostvar;
```

PostgreSQL unterstützt es, mit einer `FETCH`-Anweisung mehr als einen Datensatz abzurufen. Werden die Daten in Hostvariablen gespeichert, wird in diesem Fall angenommen, dass die Variable ein Array ist. Zum Beispiel:

```c
EXEC SQL BEGIN DECLARE SECTION;
int id[5];
EXEC SQL END DECLARE SECTION;

EXEC SQL FETCH 5 FROM mycursor INTO SQL DESCRIPTOR mydesc;

EXEC SQL GET DESCRIPTOR mydesc VALUE 1 :id = DATA;
```

### 34.7.2. SQLDA-Descriptor-Bereiche

Ein SQLDA-Descriptor-Bereich ist eine C-Sprachstruktur, die ebenfalls verwendet werden kann, um die Ergebnismenge und die Metadaten einer Abfrage zu erhalten. Eine Struktur speichert jeweils einen Datensatz aus der Ergebnismenge.

```c
EXEC SQL include sqlda.h;
sqlda_t         *mysqlda;

EXEC SQL FETCH 3 FROM mycursor INTO DESCRIPTOR mysqlda;
```

Beachten Sie, dass das Schlüsselwort `SQL` weggelassen wird. Die Absätze zu den Anwendungsfällen der Schlüsselwörter `INTO` und `USING` in [Abschnitt 34.7.1](#3471-benannte-sqldescriptorbereiche) gelten auch hier, mit einer Ergänzung. In einer `DESCRIBE`-Anweisung kann das Schlüsselwort `DESCRIPTOR` vollständig weggelassen werden, wenn das Schlüsselwort `INTO` verwendet wird:

```sql
EXEC SQL DESCRIBE prepared_statement INTO mysqlda;
```

Der allgemeine Ablauf eines Programms, das SQLDA verwendet, ist:

1. Eine Abfrage vorbereiten und einen Cursor dafür deklarieren.
2. Eine SQLDA für die Ergebniszeilen deklarieren.
3. Eine SQLDA für die Eingabeparameter deklarieren und initialisieren, einschließlich Speicherallokation und Parametereinstellungen.
4. Einen Cursor mit der Eingabe-SQLDA öffnen.
5. Zeilen aus dem Cursor abrufen und in einer Ausgabe-SQLDA speichern.
6. Werte aus der Ausgabe-SQLDA in Hostvariablen lesen, falls nötig mit Konvertierung.
7. Den Cursor schließen.
8. Den für die Eingabe-SQLDA allozierten Speicherbereich freigeben.

#### 34.7.2.1. SQLDA-Datenstruktur

SQLDA verwendet drei Datenstrukturtypen: `sqlda_t`, `sqlvar_t` und `struct sqlname`.

> Tipp: PostgreSQLs SQLDA hat eine ähnliche Datenstruktur wie die in IBM DB2 Universal Database. Daher können technische Informationen zu DB2s SQLDA helfen, die PostgreSQL-Variante besser zu verstehen.

##### 34.7.2.1.1. Struktur `sqlda_t`

Der Strukturtyp `sqlda_t` ist der Typ der eigentlichen SQLDA. Er enthält einen Datensatz. Zwei oder mehr `sqlda_t`-Strukturen können über den Zeiger im Feld `desc_next` zu einer verketteten Liste verbunden werden und stellen dadurch eine geordnete Sammlung von Zeilen dar. Wenn also zwei oder mehr Zeilen abgerufen werden, kann die Anwendung sie lesen, indem sie dem Zeiger `desc_next` in jedem `sqlda_t`-Knoten folgt.

Die Definition von `sqlda_t` lautet:

```c
struct sqlda_struct
{
    char            sqldaid[8];
    long            sqldabc;
    short           sqln;
    short           sqld;
    struct sqlda_struct *desc_next;
    struct sqlvar_struct sqlvar[1];
};

typedef struct sqlda_struct sqlda_t;
```

Die Felder haben folgende Bedeutung:

`sqldaid`

Enthält die Literalzeichenkette `"SQLDA "`.

`sqldabc`

Enthält die Größe des allozierten Bereichs in Byte.

`sqln`

Enthält die Anzahl der Eingabeparameter für eine parametrisierte Abfrage, wenn sie mithilfe des Schlüsselworts `USING` an `OPEN`-, `DECLARE`- oder `EXECUTE`-Anweisungen übergeben wird. Wenn sie als Ausgabe von `SELECT`-, `EXECUTE`- oder `FETCH`-Anweisungen verwendet wird, entspricht ihr Wert dem Feld `sqld`.

`sqld`

Enthält die Anzahl der Felder in einer Ergebnismenge.

`desc_next`

Wenn die Abfrage mehr als einen Datensatz zurückgibt, werden mehrere verkettete SQLDA-Strukturen zurückgegeben, und `desc_next` enthält einen Zeiger auf den nächsten Eintrag in der Liste.

`sqlvar`

Dies ist das Array der Spalten in der Ergebnismenge.

##### 34.7.2.1.2. Struktur `sqlvar_t`

Der Strukturtyp `sqlvar_t` enthält einen Spaltenwert und Metadaten wie Typ und Länge. Die Definition des Typs lautet:

```c
struct sqlvar_struct
{
    short          sqltype;
    short          sqllen;
    char          *sqldata;
    short         *sqlind;
    struct sqlname sqlname;
};

typedef struct sqlvar_struct sqlvar_t;
```

Die Felder haben folgende Bedeutung:

`sqltype`

Enthält die Typkennung des Felds. Werte finden Sie in `enum ECPGttype` in `ecpgtype.h`.

`sqllen`

Enthält die binäre Länge des Felds, zum Beispiel 4 Byte für `ECPGt_int`.

`sqldata`

Zeigt auf die Daten. Das Format der Daten ist in [Abschnitt 34.4.4](#3444-typzuordnung) beschrieben.

`sqlind`

Zeigt auf den Nullindikator. 0 bedeutet nicht null, -1 bedeutet null.

`sqlname`

Der Name des Felds.

##### 34.7.2.1.3. Struktur `struct sqlname`

Eine Struktur `struct sqlname` enthält einen Spaltennamen. Sie wird als Member der Struktur `sqlvar_t` verwendet. Die Definition der Struktur lautet:

```c
#define NAMEDATALEN 64

struct sqlname
{
        short                          length;
        char                           data[NAMEDATALEN];
};
```

Die Felder haben folgende Bedeutung:

`length`

Enthält die Länge des Feldnamens.

`data`

Enthält den eigentlichen Feldnamen.

#### 34.7.2.2. Eine Ergebnismenge mit einer SQLDA abrufen

Die allgemeinen Schritte, um eine Abfrageergebnismenge über eine SQLDA abzurufen, sind:

1. Eine `sqlda_t`-Struktur deklarieren, um die Ergebnismenge aufzunehmen.
2. `FETCH`-, `EXECUTE`- oder `DESCRIBE`-Befehle ausführen, um eine Abfrage unter Angabe der deklarierten SQLDA zu verarbeiten.
3. Die Anzahl der Datensätze in der Ergebnismenge prüfen, indem `sqln`, ein Member der Struktur `sqlda_t`, betrachtet wird.
4. Die Werte jeder Spalte aus `sqlvar[0]`, `sqlvar[1]` usw., also Membern der Struktur `sqlda_t`, abrufen.
5. Zur nächsten Zeile gehen, also zur nächsten `sqlda_t`-Struktur, indem dem Zeiger `desc_next` gefolgt wird.
6. Dies nach Bedarf wiederholen.

Hier ist ein Beispiel, das eine Ergebnismenge über eine SQLDA abruft.

Zuerst wird eine `sqlda_t`-Struktur deklariert, um die Ergebnismenge aufzunehmen.

```c
sqlda_t *sqlda1;
```

Danach wird die SQLDA in einem Befehl angegeben. Dies ist ein Beispiel für einen `FETCH`-Befehl.

```sql
EXEC SQL FETCH NEXT FROM cur1 INTO DESCRIPTOR sqlda1;
```

Um die Zeilen abzurufen, wird eine Schleife über die verkettete Liste ausgeführt.

```c
sqlda_t *cur_sqlda;

for (cur_sqlda = sqlda1;
     cur_sqlda != NULL;
     cur_sqlda = cur_sqlda->desc_next)
{
    ...
}
```

Innerhalb der Schleife wird eine weitere Schleife ausgeführt, um die Daten jeder Spalte, also die Struktur `sqlvar_t`, der Zeile abzurufen.

```c
for (i = 0; i < cur_sqlda->sqld; i++)
{
    sqlvar_t v = cur_sqlda->sqlvar[i];
    char *sqldata = v.sqldata;
    short sqllen = v.sqllen;
    ...
}
```

Um einen Spaltenwert zu erhalten, prüfen Sie den Wert `sqltype`, einen Member der Struktur `sqlvar_t`. Danach wechseln Sie abhängig vom Spaltentyp zu einer geeigneten Methode, um Daten aus dem Feld `sqlvar` in eine Hostvariable zu kopieren.

```c
char var_buf[1024];

switch (v.sqltype)
{
    case ECPGt_char:
        memset(&var_buf, 0, sizeof(var_buf));
        memcpy(&var_buf, sqldata, (sizeof(var_buf) <= sqllen ?
            sizeof(var_buf) - 1 : sqllen));
        break;

    case ECPGt_int: /* integer */
        memcpy(&intval, sqldata, sqllen);
        snprintf(var_buf, sizeof(var_buf), "%d", intval);
        break;

    ...
}
```

#### 34.7.2.3. Abfrageparameter mit einer SQLDA übergeben

Die allgemeinen Schritte, um Eingabeparameter mit einer SQLDA an eine vorbereitete Abfrage zu übergeben, sind:

1. Eine vorbereitete Abfrage, also eine vorbereitete Anweisung, erstellen.
2. Eine `sqlda_t`-Struktur als Eingabe-SQLDA deklarieren.
3. Speicherbereich für die Eingabe-SQLDA als `sqlda_t`-Struktur allozieren.
4. Eingabewerte in den allozierten Speicher setzen beziehungsweise kopieren.
5. Einen Cursor unter Angabe der Eingabe-SQLDA öffnen.

Hier ist ein Beispiel.

Zuerst wird eine vorbereitete Anweisung erstellt.

```c
EXEC SQL BEGIN DECLARE SECTION;
char query[1024] = "SELECT d.oid, * FROM pg_database d,
 pg_stat_database s WHERE d.oid = s.datid AND (d.datname = ? OR
 d.oid = ?)";
EXEC SQL END DECLARE SECTION;

EXEC SQL PREPARE stmt1 FROM :query;
```

Als Nächstes wird Speicher für eine SQLDA alloziert und die Anzahl der Eingabeparameter in `sqln` gesetzt, einer Membervariable der Struktur `sqlda_t`. Wenn für die vorbereitete Abfrage zwei oder mehr Eingabeparameter erforderlich sind, muss die Anwendung zusätzlichen Speicherplatz allozieren, der als `(Anzahl der Parameter - 1) * sizeof(sqlvar_t)` berechnet wird. Das hier gezeigte Beispiel alloziert Speicherplatz für zwei Eingabeparameter.

```c
sqlda_t *sqlda2;

sqlda2 = (sqlda_t *) malloc(sizeof(sqlda_t) + sizeof(sqlvar_t));
memset(sqlda2, 0, sizeof(sqlda_t) + sizeof(sqlvar_t));

sqlda2->sqln = 2; /* number of input variables */
```

Nach der Speicherallokation werden die Parameterwerte im Array `sqlvar[]` gespeichert. Dies ist dasselbe Array, das beim Abrufen von Spaltenwerten verwendet wird, wenn die SQLDA eine Ergebnismenge aufnimmt. In diesem Beispiel sind die Eingabeparameter `"postgres"` vom Zeichenkettentyp und `1` vom Ganzzahltyp.

```c
sqlda2->sqlvar[0].sqltype = ECPGt_char;
sqlda2->sqlvar[0].sqldata = "postgres";
sqlda2->sqlvar[0].sqllen = 8;

int intval = 1;
sqlda2->sqlvar[1].sqltype = ECPGt_int;
sqlda2->sqlvar[1].sqldata = (char *) &intval;
sqlda2->sqlvar[1].sqllen = sizeof(intval);
```

Durch Öffnen eines Cursors und Angabe der zuvor eingerichteten SQLDA werden die Eingabeparameter an die vorbereitete Anweisung übergeben.

```sql
EXEC SQL OPEN cur1 USING DESCRIPTOR sqlda2;
```

Nachdem Eingabe-SQLDAs verwendet wurden, muss der allozierte Speicherbereich explizit freigegeben werden, anders als bei SQLDAs, die Abfrageergebnisse aufnehmen.

```c
free(sqlda2);
```

#### 34.7.2.4. Beispielanwendung mit SQLDA

Hier ist ein Beispielprogramm, das zeigt, wie Zugriffstatistiken der durch Eingabeparameter angegebenen Datenbanken aus den Systemkatalogen abgerufen werden.

Diese Anwendung verbindet die beiden Systemtabellen `pg_database` und `pg_stat_database` über die Datenbank-OID und ruft außerdem Datenbankstatistiken ab und zeigt sie an, die durch zwei Eingabeparameter ermittelt werden: die Datenbank `postgres` und OID `1`.

Zuerst werden eine SQLDA für die Eingabe und eine SQLDA für die Ausgabe deklariert.

```c
EXEC SQL include sqlda.h;

sqlda_t *sqlda1; /* an output descriptor */
sqlda_t *sqlda2; /* an input descriptor */
```

Als Nächstes wird eine Verbindung zur Datenbank hergestellt, eine Anweisung vorbereitet und ein Cursor für die vorbereitete Anweisung deklariert.

```c
int
main(void)
{
    EXEC SQL BEGIN DECLARE SECTION;
    char query[1024] = "SELECT d.oid,* FROM pg_database d,
 pg_stat_database s WHERE d.oid=s.datid AND ( d.datname=? OR
 d.oid=? )";
    EXEC SQL END DECLARE SECTION;

    EXEC SQL CONNECT TO testdb AS con1 USER testuser;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    EXEC SQL PREPARE stmt1 FROM :query;
    EXEC SQL DECLARE cur1 CURSOR FOR stmt1;
```

Danach werden einige Werte für die Eingabeparameter in die Eingabe-SQLDA eingetragen. Speicher für die Eingabe-SQLDA wird alloziert, und die Anzahl der Eingabeparameter wird auf `sqln` gesetzt. Typ, Wert und Wertlänge werden in der Struktur `sqlvar` in `sqltype`, `sqldata` und `sqllen` gespeichert.

```c
    /* Create SQLDA structure for input parameters. */
    sqlda2 = (sqlda_t *) malloc(sizeof(sqlda_t) + sizeof(sqlvar_t));
    memset(sqlda2, 0, sizeof(sqlda_t) + sizeof(sqlvar_t));
    sqlda2->sqln = 2; /* number of input variables */

    sqlda2->sqlvar[0].sqltype = ECPGt_char;
    sqlda2->sqlvar[0].sqldata = "postgres";
    sqlda2->sqlvar[0].sqllen = 8;

    intval = 1;
    sqlda2->sqlvar[1].sqltype = ECPGt_int;
    sqlda2->sqlvar[1].sqldata = (char *)&intval;
    sqlda2->sqlvar[1].sqllen = sizeof(intval);
```

Nachdem die Eingabe-SQLDA eingerichtet wurde, wird ein Cursor mit der Eingabe-SQLDA geöffnet.

```c
    /* Open a cursor with input parameters. */
    EXEC SQL OPEN cur1 USING DESCRIPTOR sqlda2;
```

Zeilen werden aus dem geöffneten Cursor in die Ausgabe-SQLDA abgerufen. Im Allgemeinen müssen Sie `FETCH` wiederholt in einer Schleife aufrufen, um alle Zeilen der Ergebnismenge abzurufen.

```c
    while (1)
    {
        sqlda_t *cur_sqlda;

        /* Assign descriptor to the cursor */
        EXEC SQL FETCH NEXT FROM cur1 INTO DESCRIPTOR sqlda1;
```

Als Nächstes werden die abgerufenen Datensätze aus der SQLDA gelesen, indem der verketteten Liste der `sqlda_t`-Strukturen gefolgt wird.

```c
        for (cur_sqlda = sqlda1 ;
             cur_sqlda != NULL ;
             cur_sqlda = cur_sqlda->desc_next)
        {
            ...
```

Jede Spalte im ersten Datensatz wird gelesen. Die Anzahl der Spalten ist in `sqld` gespeichert, die eigentlichen Daten der ersten Spalte in `sqlvar[0]`; beide sind Member der Struktur `sqlda_t`.

```c
            /* Print every column in a row. */
            for (i = 0; i < sqlda1->sqld; i++)
            {
                sqlvar_t v = sqlda1->sqlvar[i];
                char *sqldata = v.sqldata;
                short sqllen = v.sqllen;

                strncpy(name_buf, v.sqlname.data, v.sqlname.length);
                name_buf[v.sqlname.length] = '\0';
```

Jetzt sind die Spaltendaten in der Variable `v` gespeichert. Kopieren Sie jedes Datum in Hostvariablen und betrachten Sie dabei `v.sqltype`, um den Typ der Spalte zu bestimmen.

```c
                switch (v.sqltype) {
                    int intval;
                    double doubleval;
                    unsigned long long int longlongval;

                    case ECPGt_char:
                        memset(&var_buf, 0, sizeof(var_buf));
                        memcpy(&var_buf, sqldata, (sizeof(var_buf) <=
                            sqllen ? sizeof(var_buf)-1 : sqllen));
                        break;

                    case ECPGt_int: /* integer */
                        memcpy(&intval, sqldata, sqllen);
                        snprintf(var_buf, sizeof(var_buf), "%d", intval);
                        break;

                    ...

                    default:
                        ...
                }

                printf("%s = %s (type: %d)\n", name_buf, var_buf, v.sqltype);
            }
```

Nach der Verarbeitung aller Datensätze wird der Cursor geschlossen und die Verbindung zur Datenbank getrennt.

```c
    EXEC SQL CLOSE cur1;
    EXEC SQL COMMIT;

    EXEC SQL DISCONNECT ALL;
```

Das vollständige Programm ist in Beispiel 34.1 gezeigt.

Beispiel 34.1. Beispielprogramm für SQLDA

```c
#include <stdlib.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>

EXEC SQL include sqlda.h;

sqlda_t *sqlda1; /* descriptor for output */
sqlda_t *sqlda2; /* descriptor for input */

EXEC SQL WHENEVER NOT FOUND DO BREAK;
EXEC SQL WHENEVER SQLERROR STOP;

int
main(void)
{
    EXEC SQL BEGIN DECLARE SECTION;
    char query[1024] = "SELECT d.oid,* FROM pg_database d,
 pg_stat_database s WHERE d.oid=s.datid AND ( d.datname=? OR
 d.oid=? )";

    int intval;
    unsigned long long int longlongval;
    EXEC SQL END DECLARE SECTION;

    EXEC SQL CONNECT TO uptimedb AS con1 USER uptime;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    EXEC SQL PREPARE stmt1 FROM :query;
    EXEC SQL DECLARE cur1 CURSOR FOR stmt1;

    /* Create an SQLDA structure for an input parameter */
    sqlda2 = (sqlda_t *)malloc(sizeof(sqlda_t) + sizeof(sqlvar_t));
    memset(sqlda2, 0, sizeof(sqlda_t) + sizeof(sqlvar_t));
    sqlda2->sqln = 2; /* a number of input variables */

    sqlda2->sqlvar[0].sqltype = ECPGt_char;
    sqlda2->sqlvar[0].sqldata = "postgres";
    sqlda2->sqlvar[0].sqllen = 8;

    intval = 1;
    sqlda2->sqlvar[1].sqltype = ECPGt_int;
    sqlda2->sqlvar[1].sqldata = (char *) &intval;
    sqlda2->sqlvar[1].sqllen = sizeof(intval);

    /* Open a cursor with input parameters. */
    EXEC SQL OPEN cur1 USING DESCRIPTOR sqlda2;

    while (1)
    {
        sqlda_t *cur_sqlda;

        /* Assign descriptor to the cursor */
        EXEC SQL FETCH NEXT FROM cur1 INTO DESCRIPTOR sqlda1;

        for (cur_sqlda = sqlda1 ;
             cur_sqlda != NULL ;
             cur_sqlda = cur_sqlda->desc_next)
        {
            int i;
            char name_buf[1024];
            char var_buf[1024];

            /* Print every column in a row. */
            for (i=0 ; i<cur_sqlda->sqld ; i++)
            {
                sqlvar_t v = cur_sqlda->sqlvar[i];
                char *sqldata = v.sqldata;
                short sqllen = v.sqllen;

                strncpy(name_buf, v.sqlname.data, v.sqlname.length);
                name_buf[v.sqlname.length] = '\0';

                switch (v.sqltype)
                {
                    case ECPGt_char:
                        memset(&var_buf, 0, sizeof(var_buf));
                        memcpy(&var_buf, sqldata,
                            (sizeof(var_buf)<=sqllen ? sizeof(var_buf)-1 : sqllen) );
                        break;

                    case ECPGt_int: /* integer */
                        memcpy(&intval, sqldata, sqllen);
                        snprintf(var_buf, sizeof(var_buf), "%d", intval);
                        break;

                    case ECPGt_long_long: /* bigint */
                        memcpy(&longlongval, sqldata, sqllen);
                        snprintf(var_buf, sizeof(var_buf), "%lld", longlongval);
                        break;

                    default:
                    {
                        int i;
                        memset(var_buf, 0, sizeof(var_buf));
                        for (i = 0; i < sqllen; i++)
                        {
                            char tmpbuf[16];
                            snprintf(tmpbuf, sizeof(tmpbuf), "%02x",
                                (unsigned char) sqldata[i]);
                            strncat(var_buf, tmpbuf, sizeof(var_buf));
                        }
                    }
                        break;
                }

                printf("%s = %s (type: %d)\n", name_buf, var_buf, v.sqltype);
            }

            printf("\n");
        }
    }

    EXEC SQL CLOSE cur1;
    EXEC SQL COMMIT;

    EXEC SQL DISCONNECT ALL;

    return 0;
}
```

Die Ausgabe dieses Beispiels sollte ungefähr wie folgt aussehen; einige Zahlen werden abweichen.

```text
oid = 1 (type: 1)
datname = template1 (type: 1)
datdba = 10 (type: 1)
encoding = 0 (type: 5)
datistemplate = t (type: 1)
datallowconn = t (type: 1)
dathasloginevt = f (type: 1)
datconnlimit = -1 (type: 5)
datfrozenxid = 379 (type: 1)
dattablespace = 1663 (type: 1)
datconfig = (type: 1)
datacl = {=c/uptime,uptime=CTc/uptime} (type: 1)
datid = 1 (type: 1)
datname = template1 (type: 1)
numbackends = 0 (type: 5)
xact_commit = 113606 (type: 9)
xact_rollback = 0 (type: 9)
blks_read = 130 (type: 9)
blks_hit = 7341714 (type: 9)
tup_returned = 38262679 (type: 9)
tup_fetched = 1836281 (type: 9)
tup_inserted = 0 (type: 9)
tup_updated = 0 (type: 9)
tup_deleted = 0 (type: 9)

oid = 11511 (type: 1)
datname = postgres (type: 1)
datdba = 10 (type: 1)
encoding = 0 (type: 5)
datistemplate = f (type: 1)
datallowconn = t (type: 1)
dathasloginevt = f (type: 1)
datconnlimit = -1 (type: 5)
datfrozenxid = 379 (type: 1)
dattablespace = 1663 (type: 1)
datconfig = (type: 1)
datacl = (type: 1)
datid = 11511 (type: 1)
datname = postgres (type: 1)
numbackends = 0 (type: 5)
xact_commit = 221069 (type: 9)
xact_rollback = 18 (type: 9)
blks_read = 1176 (type: 9)
blks_hit = 13943750 (type: 9)
tup_returned = 77410091 (type: 9)
tup_fetched = 3253694 (type: 9)
tup_inserted = 0 (type: 9)
tup_updated = 0 (type: 9)
tup_deleted = 0 (type: 9)
```

## 34.8. Fehlerbehandlung

Dieser Abschnitt beschreibt, wie Sie Ausnahmesituationen und Warnungen in einem Embedded-SQL-Programm behandeln können. Dafür gibt es zwei unabhängige Möglichkeiten.

- Callbacks können mit dem Befehl `WHENEVER` konfiguriert werden, um Warn- und Fehlerbedingungen zu behandeln.
- Detaillierte Informationen über den Fehler oder die Warnung können aus der Variable `sqlca` abgerufen werden.

### 34.8.1. Callbacks setzen

Eine einfache Methode, Fehler und Warnungen abzufangen, besteht darin, eine bestimmte Aktion festzulegen, die immer dann ausgeführt wird, wenn eine bestimmte Bedingung eintritt. Allgemein:

```sql
EXEC SQL WHENEVER condition action;
```

`condition` kann einer der folgenden Werte sein:

`SQLERROR`

Die angegebene Aktion wird immer dann aufgerufen, wenn bei der Ausführung einer SQL-Anweisung ein Fehler auftritt.

`SQLWARNING`

Die angegebene Aktion wird immer dann aufgerufen, wenn bei der Ausführung einer SQL-Anweisung eine Warnung auftritt.

`NOT FOUND`

Die angegebene Aktion wird immer dann aufgerufen, wenn eine SQL-Anweisung null Zeilen abruft oder betrifft. Diese Bedingung ist kein Fehler, kann aber trotzdem eine besondere Behandlung erfordern.

`action` kann einer der folgenden Werte sein:

`CONTINUE`

Dies bedeutet effektiv, dass die Bedingung ignoriert wird. Das ist die Voreinstellung.

`GOTO label`

`GO TO label`

Springt zum angegebenen Label, unter Verwendung einer C-`goto`-Anweisung.

`SQLPRINT`

Gibt eine Meldung auf die Standardfehlerausgabe aus. Das ist für einfache Programme oder während des Prototypings nützlich. Die Details der Meldung können nicht konfiguriert werden.

`STOP`

Ruft `exit(1)` auf, wodurch das Programm beendet wird.

`DO BREAK`

Führt die C-Anweisung `break` aus. Dies sollte nur in Schleifen oder `switch`-Anweisungen verwendet werden.

`DO CONTINUE`

Führt die C-Anweisung `continue` aus. Dies sollte nur in Schleifenanweisungen verwendet werden. Wenn es ausgeführt wird, kehrt der Kontrollfluss an den Anfang der Schleife zurück.

`CALL name (args)`

`DO name (args)`

Ruft die angegebenen C-Funktionen mit den angegebenen Argumenten auf. Diese Verwendung unterscheidet sich von der Bedeutung von `CALL` und `DO` in der normalen PostgreSQL-Grammatik.

Der SQL-Standard sieht nur die Aktionen `CONTINUE` und `GOTO` beziehungsweise `GO TO` vor.

Hier ist ein Beispiel, das Sie in einem einfachen Programm verwenden könnten. Es gibt bei einer Warnung eine einfache Meldung aus und bricht das Programm ab, wenn ein Fehler auftritt:

```sql
EXEC SQL WHENEVER SQLWARNING SQLPRINT;
EXEC SQL WHENEVER SQLERROR STOP;
```

Die Anweisung `EXEC SQL WHENEVER` ist eine Direktive des SQL-Präprozessors, keine C-Anweisung. Die dadurch gesetzten Fehler- oder Warnaktionen gelten für alle Embedded-SQL-Anweisungen, die unterhalb der Stelle erscheinen, an der der Handler gesetzt wurde, sofern zwischen dem ersten `EXEC SQL WHENEVER` und der SQL-Anweisung, die die Bedingung auslöst, keine andere Aktion für dieselbe Bedingung gesetzt wurde. Der Kontrollfluss im C-Programm spielt dabei keine Rolle. Daher haben die beiden folgenden C-Programmausschnitte nicht den gewünschten Effekt:

```c
/*
 * WRONG
 */
int main(int argc, char *argv[])
{
    ...
    if (verbose) {
        EXEC SQL WHENEVER SQLWARNING SQLPRINT;
    }
    ...
    EXEC SQL SELECT ...;
    ...
}

/*
 * WRONG
 */
int main(int argc, char *argv[])
{
    ...
    set_error_handler();
    ...
    EXEC SQL SELECT ...;
    ...
}

static void set_error_handler(void)
{
    EXEC SQL WHENEVER SQLERROR STOP;
}
```

### 34.8.2. `sqlca`

Für leistungsfähigere Fehlerbehandlung stellt die Embedded-SQL-Schnittstelle eine globale Variable namens `sqlca` bereit, die SQL Communication Area. Sie hat folgende Struktur:

```c
struct
{
    char sqlcaid[8];
    long sqlabc;
    long sqlcode;
    struct
    {
        int sqlerrml;
        char sqlerrmc[SQLERRMC_LEN];
    } sqlerrm;
    char sqlerrp[8];
    long sqlerrd[6];
    char sqlwarn[8];
    char sqlstate[5];
} sqlca;
```

In einem Multithread-Programm erhält jeder Thread automatisch seine eigene Kopie von `sqlca`. Das funktioniert ähnlich wie die Behandlung der globalen Standard-C-Variable `errno`.

`sqlca` deckt sowohl Warnungen als auch Fehler ab. Wenn während der Ausführung einer Anweisung mehrere Warnungen oder Fehler auftreten, enthält `sqlca` nur Informationen über den letzten.

Wenn in der letzten SQL-Anweisung kein Fehler aufgetreten ist, ist `sqlca.sqlcode` 0 und `sqlca.sqlstate` `"00000"`. Wenn eine Warnung oder ein Fehler aufgetreten ist, ist `sqlca.sqlcode` negativ und `sqlca.sqlstate` unterscheidet sich von `"00000"`. Ein positiver Wert von `sqlca.sqlcode` zeigt eine harmlose Bedingung an, zum Beispiel dass die letzte Abfrage null Zeilen zurückgegeben hat. `sqlcode` und `sqlstate` sind zwei verschiedene Fehlercode-Schemata; Details folgen unten.

Wenn die letzte SQL-Anweisung erfolgreich war, enthält `sqlca.sqlerrd[1]` die OID der verarbeiteten Zeile, falls zutreffend, und `sqlca.sqlerrd[2]` enthält die Anzahl der verarbeiteten oder zurückgegebenen Zeilen, falls dies für den Befehl zutrifft.

Im Fall eines Fehlers oder einer Warnung enthält `sqlca.sqlerrm.sqlerrmc` eine Zeichenkette, die den Fehler beschreibt. Das Feld `sqlca.sqlerrm.sqlerrml` enthält die Länge der Fehlermeldung, die in `sqlca.sqlerrm.sqlerrmc` gespeichert ist, also das Ergebnis von `strlen()`, was für C-Programmierer normalerweise wenig interessant ist. Beachten Sie, dass manche Meldungen zu lang sind, um in das Array `sqlerrmc` fester Größe zu passen; sie werden abgeschnitten.

Im Fall einer Warnung wird `sqlca.sqlwarn[2]` auf `W` gesetzt. In allen anderen Fällen wird es auf etwas anderes als `W` gesetzt. Wenn `sqlca.sqlwarn[1]` auf `W` gesetzt ist, wurde ein Wert beim Speichern in einer Hostvariable abgeschnitten. `sqlca.sqlwarn[0]` wird auf `W` gesetzt, wenn eines der anderen Elemente gesetzt ist, um eine Warnung anzuzeigen.

Die Felder `sqlcaid`, `sqlabc`, `sqlerrp` und die übrigen Elemente von `sqlerrd` und `sqlwarn` enthalten derzeit keine nützlichen Informationen.

Die Struktur `sqlca` ist nicht im SQL-Standard definiert, wird aber in mehreren anderen SQL-Datenbanksystemen implementiert. Die Definitionen sind im Kern ähnlich; wenn Sie portable Anwendungen schreiben wollen, sollten Sie die verschiedenen Implementierungen jedoch sorgfältig untersuchen.

Hier ist ein Beispiel, das die Verwendung von `WHENEVER` und `sqlca` kombiniert und den Inhalt von `sqlca` ausgibt, wenn ein Fehler auftritt. Das ist möglicherweise für Debugging oder Prototyping nützlich, bevor ein benutzerfreundlicherer Fehlerhandler eingerichtet wird.

```c
EXEC SQL WHENEVER SQLERROR CALL print_sqlca();

void
print_sqlca()
{
    fprintf(stderr, "==== sqlca ====\n");
    fprintf(stderr, "sqlcode: %ld\n", sqlca.sqlcode);
    fprintf(stderr, "sqlerrm.sqlerrml: %d\n",
        sqlca.sqlerrm.sqlerrml);
    fprintf(stderr, "sqlerrm.sqlerrmc: %s\n",
        sqlca.sqlerrm.sqlerrmc);
    fprintf(stderr, "sqlerrd: %ld %ld %ld %ld %ld %ld\n",
        sqlca.sqlerrd[0], sqlca.sqlerrd[1], sqlca.sqlerrd[2],
        sqlca.sqlerrd[3], sqlca.sqlerrd[4], sqlca.sqlerrd[5]);
    fprintf(stderr, "sqlwarn: %d %d %d %d %d %d %d %d\n",
        sqlca.sqlwarn[0], sqlca.sqlwarn[1], sqlca.sqlwarn[2],
        sqlca.sqlwarn[3], sqlca.sqlwarn[4], sqlca.sqlwarn[5],
        sqlca.sqlwarn[6], sqlca.sqlwarn[7]);
    fprintf(stderr, "sqlstate: %5s\n", sqlca.sqlstate);
    fprintf(stderr, "===============\n");
}
```

Das Ergebnis könnte wie folgt aussehen; hier handelt es sich um einen Fehler wegen eines falsch geschriebenen Tabellennamens:

```text
==== sqlca ====
sqlcode: -400
sqlerrm.sqlerrml: 49
sqlerrm.sqlerrmc: relation "pg_databasep" does not exist on line 38
sqlerrd: 0 0 0 0 0 0
sqlwarn: 0 0 0 0 0 0 0 0
sqlstate: 42P01
===============
```

### 34.8.3. SQLSTATE gegenüber SQLCODE

Die Felder `sqlca.sqlstate` und `sqlca.sqlcode` sind zwei verschiedene Schemata, die Fehlercodes bereitstellen. Beide stammen aus dem SQL-Standard, aber `SQLCODE` wurde in der SQL-92-Ausgabe des Standards als veraltet markiert und in späteren Ausgaben entfernt. Daher wird für neue Anwendungen dringend empfohlen, `SQLSTATE` zu verwenden.

`SQLSTATE` ist ein Array aus fünf Zeichen. Die fünf Zeichen enthalten Ziffern oder Großbuchstaben, die Codes für verschiedene Fehler- und Warnbedingungen darstellen. `SQLSTATE` hat ein hierarchisches Schema: Die ersten beiden Zeichen geben die allgemeine Klasse der Bedingung an, die letzten drei Zeichen eine Unterklasse der allgemeinen Bedingung. Ein erfolgreicher Zustand wird durch den Code `00000` angezeigt. Die `SQLSTATE`-Codes sind größtenteils im SQL-Standard definiert. Der PostgreSQL-Server unterstützt `SQLSTATE`-Fehlercodes nativ; daher lässt sich ein hohes Maß an Konsistenz erreichen, wenn dieses Fehlercode-Schema in allen Anwendungen verwendet wird. Weitere Informationen finden Sie in [Anhang A](A_PostgreSQL_Fehlercodes.md).

`SQLCODE`, das veraltete Fehlercode-Schema, ist eine einfache Ganzzahl. Ein Wert von 0 zeigt Erfolg an, ein positiver Wert Erfolg mit zusätzlichen Informationen, ein negativer Wert einen Fehler. Der SQL-Standard definiert nur den positiven Wert `+100`, der angibt, dass der letzte Befehl null Zeilen zurückgegeben oder betroffen hat, und keine bestimmten negativen Werte. Daher erreicht dieses Schema nur geringe Portabilität und besitzt keine hierarchische Codezuordnung. Historisch hat der Embedded-SQL-Prozessor für PostgreSQL einige spezifische `SQLCODE`-Werte für seine Verwendung zugewiesen. Sie sind unten mit ihrem numerischen Wert und ihrem symbolischen Namen aufgeführt. Beachten Sie, dass diese Werte nicht auf andere SQL-Implementierungen portierbar sind. Um die Portierung von Anwendungen auf das `SQLSTATE`-Schema zu erleichtern, ist auch der entsprechende `SQLSTATE` aufgeführt. Es gibt jedoch keine Eins-zu-eins- oder Eins-zu-viele-Abbildung zwischen den beiden Schemata, sondern tatsächlich eine Viele-zu-viele-Beziehung. Ziehen Sie daher in jedem Fall die globale `SQLSTATE`-Liste in [Anhang A](A_PostgreSQL_Fehlercodes.md) heran.

Dies sind die zugewiesenen `SQLCODE`-Werte:

`0` (`ECPG_NO_ERROR`)

Zeigt an, dass kein Fehler vorliegt. `SQLSTATE 00000`

`100` (`ECPG_NOT_FOUND`)

Dies ist eine harmlose Bedingung, die angibt, dass der letzte Befehl null Zeilen abgerufen oder verarbeitet hat oder dass Sie am Ende des Cursors angekommen sind. `SQLSTATE 02000`

Wenn Sie einen Cursor in einer Schleife verarbeiten, könnten Sie diesen Code verwenden, um zu erkennen, wann die Schleife abgebrochen werden soll:

```c
while (1)
{
    EXEC SQL FETCH ... ;
    if (sqlca.sqlcode == ECPG_NOT_FOUND)
        break;
}
```

`WHENEVER NOT FOUND DO BREAK` erledigt dies jedoch intern, sodass es normalerweise keinen Vorteil hat, dies explizit auszuschreiben.

`-12` (`ECPG_OUT_OF_MEMORY`)

Zeigt an, dass der virtuelle Speicher erschöpft ist. Der numerische Wert ist als `-ENOMEM` definiert. `SQLSTATE YE001`

`-200` (`ECPG_UNSUPPORTED`)

Zeigt an, dass der Präprozessor etwas erzeugt hat, das die Bibliothek nicht kennt. Vielleicht verwenden Sie inkompatible Versionen von Präprozessor und Bibliothek. `SQLSTATE YE002`

`-201` (`ECPG_TOO_MANY_ARGUMENTS`)

Dies bedeutet, dass der Befehl mehr Hostvariablen angegeben hat, als der Befehl erwartet hat. `SQLSTATE 07001` oder `07002`

`-202` (`ECPG_TOO_FEW_ARGUMENTS`)

Dies bedeutet, dass der Befehl weniger Hostvariablen angegeben hat, als der Befehl erwartet hat. `SQLSTATE 07001` oder `07002`

`-203` (`ECPG_TOO_MANY_MATCHES`)

Dies bedeutet, dass eine Abfrage mehrere Zeilen zurückgegeben hat, die Anweisung aber nur darauf vorbereitet war, eine Ergebniszeile zu speichern, zum Beispiel weil die angegebenen Variablen keine Arrays sind. `SQLSTATE 21000`

`-204` (`ECPG_INT_FORMAT`)

Die Hostvariable ist vom Typ `int`, und das Datum in der Datenbank hat einen anderen Typ und enthält einen Wert, der nicht als `int` interpretiert werden kann. Die Bibliothek verwendet für diese Konvertierung `strtol()`. `SQLSTATE 42804`

`-205` (`ECPG_UINT_FORMAT`)

Die Hostvariable ist vom Typ `unsigned int`, und das Datum in der Datenbank hat einen anderen Typ und enthält einen Wert, der nicht als `unsigned int` interpretiert werden kann. Die Bibliothek verwendet für diese Konvertierung `strtoul()`. `SQLSTATE 42804`

`-206` (`ECPG_FLOAT_FORMAT`)

Die Hostvariable ist vom Typ `float`, und das Datum in der Datenbank hat einen anderen Typ und enthält einen Wert, der nicht als `float` interpretiert werden kann. Die Bibliothek verwendet für diese Konvertierung `strtod()`. `SQLSTATE 42804`

`-207` (`ECPG_NUMERIC_FORMAT`)

Die Hostvariable ist vom Typ `numeric`, und das Datum in der Datenbank hat einen anderen Typ und enthält einen Wert, der nicht als numerischer Wert interpretiert werden kann. `SQLSTATE 42804`

`-208` (`ECPG_INTERVAL_FORMAT`)

Die Hostvariable ist vom Typ `interval`, und das Datum in der Datenbank hat einen anderen Typ und enthält einen Wert, der nicht als Intervallwert interpretiert werden kann. `SQLSTATE 42804`

`-209` (`ECPG_DATE_FORMAT`)

Die Hostvariable ist vom Typ `date`, und das Datum in der Datenbank hat einen anderen Typ und enthält einen Wert, der nicht als Datumswert interpretiert werden kann. `SQLSTATE 42804`

`-210` (`ECPG_TIMESTAMP_FORMAT`)

Die Hostvariable ist vom Typ `timestamp`, und das Datum in der Datenbank hat einen anderen Typ und enthält einen Wert, der nicht als Zeitstempelwert interpretiert werden kann. `SQLSTATE 42804`

`-211` (`ECPG_CONVERT_BOOL`)

Dies bedeutet, dass die Hostvariable vom Typ `bool` ist und das Datum in der Datenbank weder `'t'` noch `'f'` ist. `SQLSTATE 42804`

`-212` (`ECPG_EMPTY`)

Die an den PostgreSQL-Server gesendete Anweisung war leer. Das kann in einem Embedded-SQL-Programm normalerweise nicht vorkommen und könnte daher auf einen internen Fehler hinweisen. `SQLSTATE YE002`

`-213` (`ECPG_MISSING_INDICATOR`)

Es wurde ein Nullwert zurückgegeben, und es wurde keine Nullindikatorvariable bereitgestellt. `SQLSTATE 22002`

`-214` (`ECPG_NO_ARRAY`)

Eine gewöhnliche Variable wurde an einer Stelle verwendet, an der ein Array erforderlich ist. `SQLSTATE 42804`

`-215` (`ECPG_DATA_NOT_ARRAY`)

Die Datenbank hat an einer Stelle, an der ein Arraywert erforderlich ist, eine gewöhnliche Variable zurückgegeben. `SQLSTATE 42804`

`-216` (`ECPG_ARRAY_INSERT`)

Der Wert konnte nicht in das Array eingefügt werden. `SQLSTATE 42804`

`-220` (`ECPG_NO_CONN`)

Das Programm hat versucht, auf eine Verbindung zuzugreifen, die nicht existiert. `SQLSTATE 08003`

`-221` (`ECPG_NOT_CONN`)

Das Programm hat versucht, auf eine Verbindung zuzugreifen, die existiert, aber nicht geöffnet ist. Dies ist ein interner Fehler. `SQLSTATE YE002`

`-230` (`ECPG_INVALID_STMT`)

Die Anweisung, die Sie zu verwenden versuchen, wurde nicht vorbereitet. `SQLSTATE 26000`

`-239` (`ECPG_INFORMIX_DUPLICATE_KEY`)

Fehler wegen doppeltem Schlüssel, Verletzung einer Unique-Constraint, im Informix-Kompatibilitätsmodus. `SQLSTATE 23505`

`-240` (`ECPG_UNKNOWN_DESCRIPTOR`)

Der angegebene Descriptor wurde nicht gefunden. Die Anweisung, die Sie zu verwenden versuchen, wurde nicht vorbereitet. `SQLSTATE 33000`

`-241` (`ECPG_INVALID_DESCRIPTOR_INDEX`)

Der angegebene Descriptor-Index lag außerhalb des zulässigen Bereichs. `SQLSTATE 07009`

`-242` (`ECPG_UNKNOWN_DESCRIPTOR_ITEM`)

Es wurde ein ungültiges Descriptor-Element angefordert. Dies ist ein interner Fehler. `SQLSTATE YE002`

`-243` (`ECPG_VAR_NOT_NUMERIC`)

Während der Ausführung einer dynamischen Anweisung hat die Datenbank einen numerischen Wert zurückgegeben, und die Hostvariable war nicht numerisch. `SQLSTATE 07006`

`-244` (`ECPG_VAR_NOT_CHAR`)

Während der Ausführung einer dynamischen Anweisung hat die Datenbank einen nichtnumerischen Wert zurückgegeben, und die Hostvariable war numerisch. `SQLSTATE 07006`

`-284` (`ECPG_INFORMIX_SUBSELECT_NOT_ONE`)

Ein Ergebnis der Unterabfrage ist keine einzelne Zeile, im Informix-Kompatibilitätsmodus. `SQLSTATE 21000`

`-400` (`ECPG_PGSQL`)

Ein vom PostgreSQL-Server verursachter Fehler. Die Meldung enthält die Fehlermeldung des PostgreSQL-Servers.

`-401` (`ECPG_TRANS`)

Der PostgreSQL-Server hat signalisiert, dass die Transaktion nicht gestartet, committet oder zurückgerollt werden kann. `SQLSTATE 08007`

`-402` (`ECPG_CONNECT`)

Der Verbindungsversuch zur Datenbank war nicht erfolgreich. `SQLSTATE 08001`

`-403` (`ECPG_DUPLICATE_KEY`)

Fehler wegen doppeltem Schlüssel, Verletzung einer Unique-Constraint. `SQLSTATE 23505`

`-404` (`ECPG_SUBSELECT_NOT_ONE`)

Ein Ergebnis der Unterabfrage ist keine einzelne Zeile. `SQLSTATE 21000`

`-602` (`ECPG_WARNING_UNKNOWN_PORTAL`)

Es wurde ein ungültiger Cursorname angegeben. `SQLSTATE 34000`

`-603` (`ECPG_WARNING_IN_TRANSACTION`)

Eine Transaktion läuft. `SQLSTATE 25001`

`-604` (`ECPG_WARNING_NO_TRANSACTION`)

Es gibt keine aktive, laufende Transaktion. `SQLSTATE 25P01`

`-605` (`ECPG_WARNING_PORTAL_EXISTS`)

Es wurde ein bereits existierender Cursorname angegeben. `SQLSTATE 42P03`

## 34.9. Präprozessor-Direktiven

Es stehen mehrere Präprozessor-Direktiven zur Verfügung, die verändern, wie der Präprozessor `ecpg` eine Datei parst und verarbeitet.

### 34.9.1. Dateien einbinden

Um eine externe Datei in Ihr Embedded-SQL-Programm einzubinden, verwenden Sie:

```sql
EXEC SQL INCLUDE filename;
EXEC SQL INCLUDE <filename>;
EXEC SQL INCLUDE "filename";
```

Der Embedded-SQL-Präprozessor sucht nach einer Datei namens `filename.h`, verarbeitet sie vor und bindet sie in die resultierende C-Ausgabe ein. Dadurch werden Embedded-SQL-Anweisungen in der eingebundenen Datei korrekt behandelt.

Der Präprozessor `ecpg` sucht eine Datei in mehreren Verzeichnissen in folgender Reihenfolge:

- aktuelles Verzeichnis
- `/usr/local/include`
- PostgreSQL-Include-Verzeichnis, das zur Build-Zeit festgelegt wurde, zum Beispiel `/usr/local/pgsql/include`
- `/usr/include`

Wenn jedoch `EXEC SQL INCLUDE "filename"` verwendet wird, wird nur das aktuelle Verzeichnis durchsucht.

In jedem Verzeichnis sucht der Präprozessor zuerst nach dem Dateinamen wie angegeben. Wird er nicht gefunden, hängt er `.h` an den Dateinamen an und versucht es erneut, sofern der angegebene Dateiname diese Endung nicht bereits besitzt.

Beachten Sie, dass `EXEC SQL INCLUDE` nicht dasselbe ist wie:

```c
#include <filename.h>
```

Diese Datei würde nämlich nicht der SQL-Befehlsvorverarbeitung unterliegen. Natürlich können Sie die C-Direktive `#include` weiterhin verwenden, um andere Header-Dateien einzubinden.

> Hinweis: Beim Namen der Include-Datei wird zwischen Groß- und Kleinschreibung unterschieden, obwohl der Rest des Befehls `EXEC SQL INCLUDE` den normalen SQL-Regeln zur Groß-/Kleinschreibung folgt.

### 34.9.2. Die Direktiven `define` und `undef`

Ähnlich wie die aus C bekannte Direktive `#define` hat Embedded SQL ein vergleichbares Konzept:

```sql
EXEC SQL DEFINE name;
EXEC SQL DEFINE name value;
```

Sie können also einen Namen definieren:

```sql
EXEC SQL DEFINE HAVE_FEATURE;
```

Und Sie können auch Konstanten definieren:

```sql
EXEC SQL DEFINE MYNUMBER 12;
EXEC SQL DEFINE MYSTRING 'abc';
```

Verwenden Sie `undef`, um eine frühere Definition zu entfernen:

```sql
EXEC SQL UNDEF MYNUMBER;
```

Natürlich können Sie in Ihrem Embedded-SQL-Programm weiterhin die C-Versionen `#define` und `#undef` verwenden. Der Unterschied besteht darin, wo Ihre definierten Werte ausgewertet werden. Wenn Sie `EXEC SQL DEFINE` verwenden, wertet der Präprozessor `ecpg` die Definitionen aus und ersetzt die Werte. Wenn Sie zum Beispiel schreiben:

```sql
EXEC SQL DEFINE MYNUMBER 12;
...
EXEC SQL UPDATE Tbl SET col = MYNUMBER;
```

dann führt `ecpg` die Ersetzung bereits durch, und Ihr C-Compiler sieht niemals einen Namen oder Bezeichner `MYNUMBER`. Beachten Sie, dass Sie `#define` nicht für eine Konstante verwenden können, die Sie in einer Embedded-SQL-Abfrage verwenden wollen, weil der Embedded-SQL-Präcompiler diese Deklaration in diesem Fall nicht sehen kann.

Wenn auf der Befehlszeile des Präprozessors `ecpg` mehrere Eingabedateien angegeben werden, wirken `EXEC SQL DEFINE` und `EXEC SQL UNDEF` nicht dateiübergreifend: Jede Datei startet nur mit den Symbolen, die durch `-D`-Schalter auf der Befehlszeile definiert wurden.

### 34.9.3. Die Direktiven `ifdef`, `ifndef`, `elif`, `else` und `endif`

Sie können die folgenden Direktiven verwenden, um Codeabschnitte bedingt zu kompilieren:

`EXEC SQL ifdef name;`

Prüft einen Namen und verarbeitet die folgenden Zeilen, wenn `name` über `EXEC SQL define name` definiert wurde.

`EXEC SQL ifndef name;`

Prüft einen Namen und verarbeitet die folgenden Zeilen, wenn `name` nicht über `EXEC SQL define name` definiert wurde.

`EXEC SQL elif name;`

Beginnt einen optionalen Alternativabschnitt nach einer Direktive `EXEC SQL ifdef name` oder `EXEC SQL ifndef name`. Es können beliebig viele `elif`-Abschnitte erscheinen. Zeilen nach einem `elif` werden verarbeitet, wenn `name` definiert wurde und kein vorheriger Abschnitt desselben `ifdef`-/`ifndef`-...-`endif`-Konstrukts verarbeitet wurde.

`EXEC SQL else;`

Beginnt einen optionalen letzten Alternativabschnitt nach einer Direktive `EXEC SQL ifdef name` oder `EXEC SQL ifndef name`. Die folgenden Zeilen werden verarbeitet, wenn kein vorheriger Abschnitt desselben `ifdef`-/`ifndef`-...-`endif`-Konstrukts verarbeitet wurde.

`EXEC SQL endif;`

Beendet ein `ifdef`-/`ifndef`-...-`endif`-Konstrukt. Die folgenden Zeilen werden normal verarbeitet.

`ifdef`-/`ifndef`-...-`endif`-Konstrukte können bis zu 127 Ebenen tief verschachtelt werden.

Dieses Beispiel kompiliert genau einen der drei `SET TIMEZONE`-Befehle:

```sql
EXEC SQL ifdef TZVAR;
EXEC SQL SET TIMEZONE TO TZVAR;
EXEC SQL elif TZNAME;
EXEC SQL SET TIMEZONE TO TZNAME;
EXEC SQL else;
EXEC SQL SET TIMEZONE TO 'GMT';
EXEC SQL endif;
```

## 34.10. Embedded-SQL-Programme verarbeiten

Nachdem Sie nun eine Vorstellung davon haben, wie Embedded-SQL-C-Programme aufgebaut werden, möchten Sie vermutlich wissen, wie sie kompiliert werden. Vor dem Kompilieren lassen Sie die Datei durch den Embedded-SQL-C-Präprozessor laufen. Er wandelt die verwendeten SQL-Anweisungen in spezielle Funktionsaufrufe um. Nach dem Kompilieren müssen Sie mit einer speziellen Bibliothek linken, die die benötigten Funktionen enthält. Diese Funktionen holen Informationen aus den Argumenten, führen den SQL-Befehl über die `libpq`-Schnittstelle aus und legen das Ergebnis in den für die Ausgabe angegebenen Argumenten ab.

Das Präprozessorprogramm heißt `ecpg` und ist in einer normalen PostgreSQL-Installation enthalten. Embedded-SQL-Programme tragen typischerweise die Endung `.pgc`. Wenn Sie eine Programmdatei namens `prog1.pgc` haben, können Sie sie einfach so vorverarbeiten:

```text
ecpg prog1.pgc
```

Dadurch wird eine Datei namens `prog1.c` erzeugt. Wenn Ihre Eingabedateien nicht dem vorgeschlagenen Namensmuster folgen, können Sie die Ausgabedatei mit der Option `-o` ausdrücklich angeben.

Die vorverarbeitete Datei kann normal kompiliert werden, zum Beispiel:

```text
cc -c prog1.c
```

Die erzeugten C-Quelldateien binden Header-Dateien aus der PostgreSQL-Installation ein. Wenn Sie PostgreSQL an einem Ort installiert haben, der standardmäßig nicht durchsucht wird, müssen Sie der Kompilierbefehlszeile daher eine Option wie `-I/usr/local/pgsql/include` hinzufügen.

Um ein Embedded-SQL-Programm zu linken, müssen Sie die Bibliothek `libecpg` einbinden, etwa so:

```text
cc -o myprog prog1.o prog2.o ... -lecpg
```

Auch hier müssen Sie der Befehlszeile gegebenenfalls eine Option wie `-L/usr/local/pgsql/lib` hinzufügen.

Sie können `pg_config` oder `pkg-config` mit dem Paketnamen `libecpg` verwenden, um die Pfade Ihrer Installation zu ermitteln.

Wenn Sie den Build-Prozess eines größeren Projekts mit `make` verwalten, kann es praktisch sein, die folgende implizite Regel in Ihre Makefiles aufzunehmen:

```makefile
ECPG = ecpg

%.c: %.pgc
        $(ECPG) $<
```

Die vollständige Syntax des Befehls `ecpg` ist bei `ecpg` beschrieben.

Die Bibliothek `ecpg` ist standardmäßig thread-sicher. Es kann jedoch nötig sein, beim Kompilieren Ihres Client-Codes bestimmte Threading-Befehlszeilenoptionen zu verwenden.

## 34.11. Bibliotheksfunktionen

Die Bibliothek `libecpg` enthält vor allem „versteckte“ Funktionen, mit denen die durch Embedded-SQL-Befehle ausgedrückte Funktionalität implementiert wird. Es gibt aber einige Funktionen, die sinnvoll direkt aufgerufen werden können. Beachten Sie, dass Ihr Code dadurch unportabel wird.

- `ECPGdebug(int on, FILE *stream)` schaltet Debug-Logging ein, wenn das erste Argument ungleich null ist. Das Debug-Logging erfolgt auf `stream`. Das Log enthält alle SQL-Anweisungen mit allen eingesetzten Eingabevariablen sowie die Ergebnisse vom PostgreSQL-Server. Das kann bei der Fehlersuche in SQL-Anweisungen sehr nützlich sein.

> Hinweis: Unter Windows stürzt der Funktionsaufruf ab, wenn die `ecpg`-Bibliotheken und eine Anwendung mit unterschiedlichen Flags kompiliert wurden, weil sich die interne Darstellung der `FILE`-Zeiger unterscheidet. Insbesondere sollten Multithread-/Singlethread-, Release-/Debug- und statisch/dynamisch-Flags für die Bibliothek und alle Anwendungen, die diese Bibliothek verwenden, gleich sein.

- `ECPGget_PGconn(const char *connection_name)` gibt das Datenbankverbindungs-Handle der Bibliothek zurück, das durch den angegebenen Namen identifiziert wird. Wenn `connection_name` auf `NULL` gesetzt ist, wird das aktuelle Verbindungs-Handle zurückgegeben. Wenn kein Verbindungs-Handle identifiziert werden kann, gibt die Funktion `NULL` zurück. Das zurückgegebene Verbindungs-Handle kann bei Bedarf verwendet werden, um andere Funktionen aus `libpq` aufzurufen.

> Hinweis: Es ist keine gute Idee, von `ecpg` erzeugte Datenbankverbindungs-Handles direkt mit `libpq`-Routinen zu manipulieren.

- `ECPGtransactionStatus(const char *connection_name)` gibt den aktuellen Transaktionsstatus der angegebenen Verbindung zurück, die durch `connection_name` identifiziert wird. Details zu den zurückgegebenen Statuscodes finden Sie in [Abschnitt 32.2](32_libpq_C_Bibliothek.md#322-funktionen-zum-verbindungsstatus) und bei `PQtransactionStatus` von `libpq`.

- `ECPGstatus(int lineno, const char* connection_name)` gibt wahr zurück, wenn Sie mit einer Datenbank verbunden sind, andernfalls falsch. `connection_name` kann `NULL` sein, wenn eine einzelne Verbindung verwendet wird.

## 34.12. Große Objekte

Große Objekte werden von ECPG nicht direkt unterstützt, aber eine ECPG-Anwendung kann große Objekte über die Large-Object-Funktionen von `libpq` bearbeiten, indem sie das nötige `PGconn`-Objekt durch Aufruf der Funktion `ECPGget_PGconn()` erhält. Die Funktion `ECPGget_PGconn()` und der direkte Zugriff auf `PGconn`-Objekte sollten allerdings sehr vorsichtig verwendet und idealerweise nicht mit anderen ECPG-Datenbankzugriffen gemischt werden.

Weitere Details zu `ECPGget_PGconn()` finden Sie in [Abschnitt 34.11](34_ECPG_Embedded_SQL_in_C.md#3411-bibliotheksfunktionen). Informationen zur Funktionsschnittstelle für große Objekte finden Sie in [Kapitel 33](33_Große_Objekte.md).

Large-Object-Funktionen müssen in einem Transaktionsblock aufgerufen werden. Wenn Autocommit ausgeschaltet ist, müssen daher `BEGIN`-Befehle ausdrücklich ausgegeben werden.

Beispiel 34.2 zeigt ein Beispielprogramm, das demonstriert, wie ein großes Objekt in einer ECPG-Anwendung erzeugt, geschrieben und gelesen wird.

Beispiel 34.2. ECPG-Programm, das auf große Objekte zugreift

```c
#include <stdio.h>
#include <stdlib.h>
#include <libpq-fe.h>
#include <libpq/libpq-fs.h>

EXEC SQL WHENEVER SQLERROR STOP;

int
main(void)
{
    PGconn            *conn;
    Oid                loid;
    int                fd;
    char               buf[256];
    int                buflen = 256;
    char               buf2[256];
    int                rc;

    memset(buf, 1, buflen);

    EXEC SQL CONNECT TO testdb AS con1;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    conn = ECPGget_PGconn("con1");
    printf("conn = %p\n", conn);

    /* create */
    loid = lo_create(conn, 0);
    if (loid < 0)
        printf("lo_create() failed: %s", PQerrorMessage(conn));

    printf("loid = %d\n", loid);

    /* write test */
    fd = lo_open(conn, loid, INV_READ|INV_WRITE);
    if (fd < 0)
        printf("lo_open() failed: %s", PQerrorMessage(conn));

    printf("fd = %d\n", fd);

    rc = lo_write(conn, fd, buf, buflen);
    if (rc < 0)
        printf("lo_write() failed\n");

    rc = lo_close(conn, fd);
    if (rc < 0)
        printf("lo_close() failed: %s", PQerrorMessage(conn));

    /* read test */
    fd = lo_open(conn, loid, INV_READ);
    if (fd < 0)
        printf("lo_open() failed: %s", PQerrorMessage(conn));

    printf("fd = %d\n", fd);

    rc = lo_read(conn, fd, buf2, buflen);
    if (rc < 0)
        printf("lo_read() failed\n");

    rc = lo_close(conn, fd);
    if (rc < 0)
        printf("lo_close() failed: %s", PQerrorMessage(conn));

    /* check */
    rc = memcmp(buf, buf2, buflen);
    printf("memcmp() = %d\n", rc);

    /* cleanup */
    rc = lo_unlink(conn, loid);
    if (rc < 0)
        printf("lo_unlink() failed: %s", PQerrorMessage(conn));

    EXEC SQL COMMIT;
    EXEC SQL DISCONNECT ALL;
    return 0;
}
```

## 34.13. C++-Anwendungen

ECPG bietet begrenzte Unterstützung für C++-Anwendungen. Dieser Abschnitt beschreibt einige Fallstricke.

Der Präprozessor `ecpg` nimmt eine Eingabedatei, die in C oder etwas C-Ähnlichem geschrieben ist, sowie Embedded-SQL-Befehle entgegen, wandelt die Embedded-SQL-Befehle in C-Sprachblöcke um und erzeugt schließlich eine `.c`-Datei. Die Header-Datei-Deklarationen der Bibliotheksfunktionen, die von den durch `ecpg` erzeugten C-Sprachblöcken verwendet werden, werden bei der Verwendung unter C++ in Blöcke `extern "C" { ... }` eingeschlossen. Daher sollten sie in C++ nahtlos funktionieren.

Im Allgemeinen versteht der Präprozessor `ecpg` jedoch nur C. Er behandelt nicht die besondere Syntax und die reservierten Wörter der C++-Sprache. Daher kann Embedded-SQL-Code, der in C++-Anwendungscode geschrieben ist und komplizierte C++-spezifische Funktionen verwendet, möglicherweise nicht korrekt vorverarbeitet werden oder nicht wie erwartet funktionieren.

Eine sichere Möglichkeit, Embedded-SQL-Code in einer C++-Anwendung zu verwenden, besteht darin, die ECPG-Aufrufe in einem C-Modul zu verbergen, das vom C++-Anwendungscode für den Datenbankzugriff aufgerufen wird, und dieses Modul mit dem übrigen C++-Code zusammenzulinken. Siehe dazu [Abschnitt 34.13.2](#34132-canwendungsentwicklung-mit-externem-cmodul).

### 34.13.1. Geltungsbereich für Hostvariablen

Der Präprozessor `ecpg` versteht den Geltungsbereich von Variablen in C. In der Sprache C ist das recht einfach, weil die Geltungsbereiche von Variablen auf ihren Codeblöcken beruhen. In C++ werden Klassen-Membervariablen jedoch in einem anderen Codeblock referenziert als an der Deklarationsstelle; daher versteht der Präprozessor `ecpg` den Geltungsbereich von Klassen-Membervariablen nicht.

Im folgenden Fall kann der Präprozessor `ecpg` zum Beispiel keine Deklaration für die Variable `dbname` in der Methode `test` finden, sodass ein Fehler auftritt.

```cpp
class TestCpp
{
    EXEC SQL BEGIN DECLARE SECTION;
    char dbname[1024];
    EXEC SQL END DECLARE SECTION;

   public:
     TestCpp();
     void test();
     ~TestCpp();
};

TestCpp::TestCpp()
{
    EXEC SQL CONNECT TO testdb1;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;
}

void Test::test()
{
    EXEC SQL SELECT current_database() INTO :dbname;
    printf("current_database = %s\n", dbname);
}

TestCpp::~TestCpp()
{
    EXEC SQL DISCONNECT ALL;
}
```

Dieser Code führt zu einem Fehler wie diesem:

```text
ecpg test_cpp.pgc
test_cpp.pgc:28: ERROR: variable "dbname" is not declared
```

Um dieses Geltungsbereichsproblem zu vermeiden, könnte die Methode `test` so geändert werden, dass eine lokale Variable als Zwischenspeicher verwendet wird. Dieser Ansatz ist jedoch nur ein schlechter Workaround, weil er den Code unansehnlicher macht und die Leistung reduziert.

```cpp
void TestCpp::test()
{
    EXEC SQL BEGIN DECLARE SECTION;
    char tmp[1024];
    EXEC SQL END DECLARE SECTION;

    EXEC SQL SELECT current_database() INTO :tmp;

    strlcpy(dbname, tmp, sizeof(tmp));

    printf("current_database = %s\n", dbname);
}
```

### 34.13.2. C++-Anwendungsentwicklung mit externem C-Modul

Wenn Sie diese technischen Einschränkungen des Präprozessors `ecpg` in C++ verstehen, kommen Sie möglicherweise zu dem Schluss, dass es besser ist, C-Objekte und C++-Objekte beim Linken zusammenzuführen, damit C++-Anwendungen ECPG-Funktionen nutzen können, statt Embedded-SQL-Befehle direkt in C++-Code zu schreiben. Dieser Abschnitt beschreibt anhand eines einfachen Beispiels eine Möglichkeit, einige Embedded-SQL-Befehle vom C++-Anwendungscode zu trennen. In diesem Beispiel ist die Anwendung in C++ implementiert, während C und ECPG für die Verbindung zum PostgreSQL-Server verwendet werden.

Drei Arten von Dateien müssen erstellt werden: eine C-Datei (`*.pgc`), eine Header-Datei und eine C++-Datei.

`test_mod.pgc`

Ein Unterprogrammmodul, um in C eingebettete SQL-Befehle auszuführen. Es wird vom Präprozessor in `test_mod.c` umgewandelt.

```c
#include "test_mod.h"
#include <stdio.h>

void
db_connect()
{
    EXEC SQL CONNECT TO testdb1;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;
}

void
db_test()
{
    EXEC SQL BEGIN DECLARE SECTION;
    char dbname[1024];
    EXEC SQL END DECLARE SECTION;

    EXEC SQL SELECT current_database() INTO :dbname;
    printf("current_database = %s\n", dbname);
}

void
db_disconnect()
{
    EXEC SQL DISCONNECT ALL;
}
```

`test_mod.h`

Eine Header-Datei mit Deklarationen der Funktionen im C-Modul `test_mod.pgc`. Sie wird von `test_cpp.cpp` eingebunden. Diese Datei muss um die Deklarationen einen Block `extern "C"` haben, weil sie aus dem C++-Modul gelinkt wird.

```c
#ifdef __cplusplus
extern "C" {
#endif

void db_connect();
void db_test();
void db_disconnect();

#ifdef __cplusplus
}
#endif
```

`test_cpp.cpp`

Der Hauptcode der Anwendung, einschließlich der Hauptroutine und in diesem Beispiel einer C++-Klasse.

```cpp
#include "test_mod.h"

class TestCpp
{
   public:
     TestCpp();
     void test();
     ~TestCpp();
};

TestCpp::TestCpp()
{
    db_connect();
}

void
TestCpp::test()
{
    db_test();
}

TestCpp::~TestCpp()
{
    db_disconnect();
}

int
main(void)
{
    TestCpp *t = new TestCpp();

    t->test();
    return 0;
}
```

Um die Anwendung zu bauen, gehen Sie wie folgt vor. Wandeln Sie `test_mod.pgc` durch Ausführen von `ecpg` in `test_mod.c` um und erzeugen Sie `test_mod.o`, indem Sie `test_mod.c` mit dem C-Compiler kompilieren:

```text
ecpg -o test_mod.c test_mod.pgc
cc -c test_mod.c -o test_mod.o
```

Als Nächstes erzeugen Sie `test_cpp.o`, indem Sie `test_cpp.cpp` mit dem C++-Compiler kompilieren:

```text
c++ -c test_cpp.cpp -o test_cpp.o
```

Zum Schluss linken Sie diese Objektdateien, `test_cpp.o` und `test_mod.o`, mit dem C++-Compiler-Treiber zu einer ausführbaren Datei:

```text
c++ test_cpp.o test_mod.o -lecpg -o test_cpp
```

## 34.14. Embedded-SQL-Befehle

Dieser Abschnitt beschreibt alle SQL-Befehle, die spezifisch für Embedded SQL sind. Beachten Sie auch die unter SQL-Befehle aufgeführten SQL-Befehle, die ebenfalls in Embedded SQL verwendet werden können, sofern nichts anderes angegeben ist.

`ALLOCATE DESCRIPTOR`

`ALLOCATE DESCRIPTOR` - einen SQL-Descriptor-Bereich allozieren.

**Syntax**

```sql
ALLOCATE DESCRIPTOR name
```

**Beschreibung**

`ALLOCATE DESCRIPTOR` alloziert einen neuen benannten SQL-Descriptor-Bereich, der zum Datenaustausch zwischen dem PostgreSQL-Server und dem Hostprogramm verwendet werden kann.

Descriptor-Bereiche sollten nach der Verwendung mit dem Befehl `DEALLOCATE DESCRIPTOR` freigegeben werden.

**Parameter**

`name`

Ein Name eines SQL-Descriptors; Groß-/Kleinschreibung ist relevant. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

**Beispiele**

```sql
EXEC SQL ALLOCATE DESCRIPTOR mydesc;
```

**Kompatibilität**

`ALLOCATE DESCRIPTOR` ist im SQL-Standard spezifiziert.

**Siehe auch**

`DEALLOCATE DESCRIPTOR`, `GET DESCRIPTOR`, `SET DESCRIPTOR`

`CONNECT`

`CONNECT` - eine Datenbankverbindung herstellen.

**Syntax**

```sql
CONNECT TO connection_target [ AS connection_name ]
 [ USER connection_user ]
CONNECT TO DEFAULT
CONNECT connection_user
DATABASE connection_target
```

**Beschreibung**

Der Befehl `CONNECT` stellt eine Verbindung zwischen dem Client und dem PostgreSQL-Server her.

**Parameter**

`connection_target`

`connection_target` gibt den Zielserver der Verbindung in einer von mehreren Formen an.

```text
[ database_name ] [ @host ] [ :port ]
```

Verbindung über TCP/IP.

```text
unix:postgresql://host [ :port ] / [ database_name ] [ ?connection_option ]
```

Verbindung über Unix-Domain-Sockets.

```text
tcp:postgresql://host [ :port ] / [ database_name ] [ ?connection_option ]
```

Verbindung über TCP/IP.

`SQL string constant`

Enthält einen Wert in einer der obigen Formen.

`host variable`

Hostvariable vom Typ `char[]` oder `VARCHAR[]`, die einen Wert in einer der obigen Formen enthält.

`connection_name`

Ein optionaler Bezeichner für die Verbindung, damit in anderen Befehlen auf sie Bezug genommen werden kann. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

`connection_user`

Der Benutzername für die Datenbankverbindung.

Dieser Parameter kann auch Benutzername und Passwort in einer der Formen `user_name/password`, `user_name IDENTIFIED BY password` oder `user_name USING password` angeben.

Benutzername und Passwort können SQL-Bezeichner, Zeichenkettenkonstanten oder Hostvariablen sein.

`DEFAULT`

Verwendet alle Standard-Verbindungsparameter, wie sie durch `libpq` definiert sind.

**Beispiele**

Hier sind mehrere Varianten zur Angabe von Verbindungsparametern:

```sql
EXEC SQL CONNECT TO "connectdb" AS main;
EXEC SQL CONNECT TO "connectdb" AS second;
EXEC SQL CONNECT TO "unix:postgresql://200.46.204.71/connectdb" AS
 main USER connectuser;
EXEC SQL CONNECT TO "unix:postgresql://localhost/connectdb" AS main
 USER connectuser;
EXEC SQL CONNECT TO 'connectdb' AS main;
EXEC SQL CONNECT TO 'unix:postgresql://localhost/connectdb' AS main
 USER :user;
EXEC SQL CONNECT TO :db AS :id;
EXEC SQL CONNECT TO :db USER connectuser USING :pw;
EXEC SQL CONNECT TO @localhost AS main USER connectdb;
EXEC SQL CONNECT TO REGRESSDB1 as main;
EXEC SQL CONNECT TO AS main USER connectdb;
EXEC SQL CONNECT TO connectdb AS :id;
EXEC SQL CONNECT TO connectdb AS main USER connectuser/connectdb;
EXEC SQL CONNECT TO connectdb AS main;
EXEC SQL CONNECT TO connectdb@localhost AS main;
EXEC SQL CONNECT TO tcp:postgresql://localhost/ USER connectdb;
EXEC SQL CONNECT TO tcp:postgresql://localhost/connectdb USER
 connectuser IDENTIFIED BY connectpw;
EXEC SQL CONNECT TO tcp:postgresql://localhost:20/connectdb USER
 connectuser IDENTIFIED BY connectpw;
EXEC SQL CONNECT TO unix:postgresql://localhost/ AS main USER
 connectdb;
EXEC SQL CONNECT TO unix:postgresql://localhost/connectdb AS main
 USER connectuser;
EXEC SQL CONNECT TO unix:postgresql://localhost/connectdb USER
 connectuser IDENTIFIED BY "connectpw";
EXEC SQL CONNECT TO unix:postgresql://localhost/connectdb USER
 connectuser USING "connectpw";
EXEC SQL CONNECT TO unix:postgresql://localhost/connectdb?
connect_timeout=14 USER connectuser;
```

Hier ist ein Beispielprogramm, das die Verwendung von Hostvariablen zur Angabe von Verbindungsparametern zeigt:

```c
int
main(void)
{
EXEC SQL BEGIN DECLARE SECTION;
    char *dbname     = "testdb";    /* database name */
    char *user       = "testuser";  /* connection user name */
    char *connection = "tcp:postgresql://localhost:5432/testdb";
                                    /* connection string */
    char ver[256];                  /* buffer to store the version
 string */
EXEC SQL END DECLARE SECTION;

    ECPGdebug(1, stderr);

    EXEC SQL CONNECT TO :dbname USER :user;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;
    EXEC SQL SELECT version() INTO :ver;
    EXEC SQL DISCONNECT;

    printf("version: %s\n", ver);

    EXEC SQL CONNECT TO :connection USER :user;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;
    EXEC SQL SELECT version() INTO :ver;
    EXEC SQL DISCONNECT;

    printf("version: %s\n", ver);

    return 0;
}
```

**Kompatibilität**

`CONNECT` ist im SQL-Standard spezifiziert, aber das Format der Verbindungsparameter ist implementierungsspezifisch.

**Siehe auch**

`DISCONNECT`, `SET CONNECTION`

`DEALLOCATE DESCRIPTOR`

`DEALLOCATE DESCRIPTOR` - einen SQL-Descriptor-Bereich freigeben.

**Syntax**

```sql
DEALLOCATE DESCRIPTOR name
```

**Beschreibung**

`DEALLOCATE DESCRIPTOR` gibt einen benannten SQL-Descriptor-Bereich frei.

**Parameter**

`name`

Der Name des Descriptors, der freigegeben werden soll. Groß-/Kleinschreibung ist relevant. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

**Beispiele**

```sql
EXEC SQL DEALLOCATE DESCRIPTOR mydesc;
```

**Kompatibilität**

`DEALLOCATE DESCRIPTOR` ist im SQL-Standard spezifiziert.

**Siehe auch**

`ALLOCATE DESCRIPTOR`, `GET DESCRIPTOR`, `SET DESCRIPTOR`

`DECLARE`

`DECLARE` - einen Cursor definieren.

**Syntax**

```sql
DECLARE cursor_name [ BINARY ] [ ASENSITIVE | INSENSITIVE ]
 [ [ NO ] SCROLL ] CURSOR [ { WITH | WITHOUT } HOLD ]
 FOR prepared_name
DECLARE cursor_name [ BINARY ] [ ASENSITIVE | INSENSITIVE ]
 [ [ NO ] SCROLL ] CURSOR [ { WITH | WITHOUT } HOLD ] FOR query
```

**Beschreibung**

`DECLARE` deklariert einen Cursor zum Durchlaufen der Ergebnismenge einer vorbereiteten Anweisung. Dieser Befehl hat eine etwas andere Semantik als der direkte SQL-Befehl `DECLARE`: Während letzterer eine Abfrage ausführt und die Ergebnismenge für den Abruf vorbereitet, deklariert dieser Embedded-SQL-Befehl lediglich einen Namen als „Schleifenvariable“ zum Durchlaufen der Ergebnismenge einer Abfrage. Die eigentliche Ausführung erfolgt, wenn der Cursor mit dem Befehl `OPEN` geöffnet wird.

**Parameter**

`cursor_name`

Ein Cursorname; Groß-/Kleinschreibung ist relevant. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

`prepared_name`

Der Name einer vorbereiteten Abfrage, entweder als SQL-Bezeichner oder als Hostvariable.

`query`

Ein `SELECT`- oder `VALUES`-Befehl, der die vom Cursor zurückzugebenden Zeilen liefert.

Die Bedeutung der Cursoroptionen ist bei `DECLARE` beschrieben.

**Beispiele**

Beispiele für das Deklarieren eines Cursors für eine Abfrage:

```sql
EXEC SQL DECLARE C CURSOR FOR SELECT * FROM My_Table;
EXEC SQL DECLARE C CURSOR FOR SELECT Item1 FROM T;
EXEC SQL DECLARE cur1 CURSOR FOR SELECT version();
```

Ein Beispiel für das Deklarieren eines Cursors für eine vorbereitete Anweisung:

```sql
EXEC SQL PREPARE stmt1 AS SELECT version();
EXEC SQL DECLARE cur1 CURSOR FOR stmt1;
```

**Kompatibilität**

`DECLARE` ist im SQL-Standard spezifiziert.

**Siehe auch**

`OPEN`, `CLOSE`, `DECLARE`

`DECLARE STATEMENT`

`DECLARE STATEMENT` - einen SQL-Anweisungsbezeichner deklarieren.

**Syntax**

```sql
EXEC SQL [ AT connection_name ] DECLARE statement_name STATEMENT
```

**Beschreibung**

`DECLARE STATEMENT` deklariert einen SQL-Anweisungsbezeichner. Ein SQL-Anweisungsbezeichner kann mit der Verbindung verknüpft werden. Wenn der Bezeichner von dynamischen SQL-Anweisungen verwendet wird, werden die Anweisungen über die zugeordnete Verbindung ausgeführt. Der Namensraum der Deklaration ist die Präkompilierungseinheit, und mehrere Deklarationen für denselben SQL-Anweisungsbezeichner sind nicht erlaubt. Beachten Sie, dass im Informix-Kompatibilitätsmodus des Präcompilers, wenn eine SQL-Anweisung deklariert ist, `"database"` nicht als Cursorname verwendet werden kann.

**Parameter**

`connection_name`

Ein Datenbankverbindungsname, der durch den Befehl `CONNECT` hergestellt wurde.

Die `AT`-Klausel kann weggelassen werden, eine solche Anweisung hat dann aber keine Bedeutung.

`statement_name`

Der Name eines SQL-Anweisungsbezeichners, entweder als SQL-Bezeichner oder als Hostvariable.

**Hinweise**

Diese Zuordnung ist nur gültig, wenn die Deklaration physisch oberhalb einer dynamischen Anweisung steht.

**Beispiele**

```sql
EXEC SQL CONNECT TO postgres AS con1;
EXEC SQL AT con1 DECLARE sql_stmt STATEMENT;
EXEC SQL DECLARE cursor_name CURSOR FOR sql_stmt;
EXEC SQL PREPARE sql_stmt FROM :dyn_string;
EXEC SQL OPEN cursor_name;
EXEC SQL FETCH cursor_name INTO :column1;
EXEC SQL CLOSE cursor_name;
```

**Kompatibilität**

`DECLARE STATEMENT` ist eine Erweiterung des SQL-Standards, kann aber in bekannten DBMSs verwendet werden.

**Siehe auch**

`CONNECT`, `DECLARE`, `OPEN`

`DESCRIBE`

`DESCRIBE` - Informationen über eine vorbereitete Anweisung oder Ergebnismenge abrufen.

**Syntax**

```sql
DESCRIBE [ OUTPUT ] prepared_name USING [ SQL ]
 DESCRIPTOR descriptor_name
DESCRIBE [ OUTPUT ] prepared_name INTO [ SQL ]
 DESCRIPTOR descriptor_name
DESCRIBE [ OUTPUT ] prepared_name INTO sqlda_name
```

**Beschreibung**

`DESCRIBE` ruft Metadateninformationen über die Ergebnisspalten ab, die in einer vorbereiteten Anweisung enthalten sind, ohne tatsächlich eine Zeile abzurufen.

**Parameter**

`prepared_name`

Der Name einer vorbereiteten Anweisung. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

`descriptor_name`

Ein Descriptorname. Groß-/Kleinschreibung ist relevant. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

`sqlda_name`

Der Name einer SQLDA-Variable.

**Beispiele**

```sql
EXEC SQL ALLOCATE DESCRIPTOR mydesc;
EXEC SQL PREPARE stmt1 FROM :sql_stmt;
EXEC SQL DESCRIBE stmt1 INTO SQL DESCRIPTOR mydesc;
EXEC SQL GET DESCRIPTOR mydesc VALUE 1 :charvar = NAME;
EXEC SQL DEALLOCATE DESCRIPTOR mydesc;
```

**Kompatibilität**

`DESCRIBE` ist im SQL-Standard spezifiziert.

**Siehe auch**

`ALLOCATE DESCRIPTOR`, `GET DESCRIPTOR`

`DISCONNECT`

`DISCONNECT` - eine Datenbankverbindung beenden.

**Syntax**

```sql
DISCONNECT connection_name
DISCONNECT [ CURRENT ]
DISCONNECT ALL
```

**Beschreibung**

`DISCONNECT` schließt eine Verbindung oder alle Verbindungen zur Datenbank.

**Parameter**

`connection_name`

Ein Datenbankverbindungsname, der durch den Befehl `CONNECT` hergestellt wurde.

`CURRENT`

Schließt die „aktuelle“ Verbindung. Das ist entweder die zuletzt geöffnete Verbindung oder die Verbindung, die mit dem Befehl `SET CONNECTION` gesetzt wurde. Dies ist auch die Voreinstellung, wenn dem Befehl `DISCONNECT` kein Argument übergeben wird.

`ALL`

Schließt alle offenen Verbindungen.

**Beispiele**

```c
int
main(void)
{
    EXEC SQL CONNECT TO testdb AS con1 USER testuser;
    EXEC SQL CONNECT TO testdb AS con2 USER testuser;
    EXEC SQL CONNECT TO testdb AS con3 USER testuser;

    EXEC SQL DISCONNECT CURRENT;                 /* close con3          */
    EXEC SQL DISCONNECT ALL;                     /* close con2 and con1 */

    return 0;
}
```

**Kompatibilität**

`DISCONNECT` ist im SQL-Standard spezifiziert.

**Siehe auch**

`CONNECT`, `SET CONNECTION`

`EXECUTE IMMEDIATE`

`EXECUTE IMMEDIATE` - eine Anweisung dynamisch vorbereiten und ausführen.

**Syntax**

```sql
EXECUTE IMMEDIATE string
```

**Beschreibung**

`EXECUTE IMMEDIATE` bereitet eine dynamisch angegebene SQL-Anweisung sofort vor und führt sie aus, ohne Ergebniszeilen abzurufen.

**Parameter**

`string`

Eine Literalzeichenkette oder eine Hostvariable, die die auszuführende SQL-Anweisung enthält.

**Hinweise**

Typischerweise ist `string` eine Hostvariablenreferenz auf eine Zeichenkette, die eine dynamisch konstruierte SQL-Anweisung enthält. Der Fall einer Literalzeichenkette ist nicht sehr nützlich; Sie könnten die SQL-Anweisung genauso gut direkt schreiben, ohne die zusätzliche Schreibarbeit von `EXECUTE IMMEDIATE`.

Wenn Sie eine Literalzeichenkette verwenden, beachten Sie, dass alle doppelten Anführungszeichen, die Sie in die SQL-Anweisung aufnehmen möchten, als oktale Escapes (`\042`) geschrieben werden müssen, nicht im üblichen C-Stil `\"`. Das liegt daran, dass sich die Zeichenkette innerhalb eines Abschnitts `EXEC SQL` befindet, sodass der ECPG-Lexer sie nach SQL-Regeln und nicht nach C-Regeln parst. Eingebettete Backslashes werden später nach C-Regeln behandelt; `\"` verursacht jedoch sofort einen Syntaxfehler, weil es als Ende des Literals verstanden wird.

**Beispiele**

Hier ist ein Beispiel, das mit `EXECUTE IMMEDIATE` und einer Hostvariable namens `command` eine `INSERT`-Anweisung ausführt:

```c
sprintf(command, "INSERT INTO test (name, amount, letter) VALUES
 ('db: ''r1''', 1, 'f')");
EXEC SQL EXECUTE IMMEDIATE :command;
```

**Kompatibilität**

`EXECUTE IMMEDIATE` ist im SQL-Standard spezifiziert.

`GET DESCRIPTOR`

`GET DESCRIPTOR` - Informationen aus einem SQL-Descriptor-Bereich abrufen.

**Syntax**

```sql
GET DESCRIPTOR descriptor_name :cvariable = descriptor_header_item
 [, ... ]
GET DESCRIPTOR descriptor_name VALUE column_number :cvariable
 = descriptor_item [, ... ]
```

**Beschreibung**

`GET DESCRIPTOR` ruft Informationen über eine Abfrageergebnismenge aus einem SQL-Descriptor-Bereich ab und speichert sie in Hostvariablen. Ein Descriptor-Bereich wird typischerweise mit `FETCH` oder `SELECT` gefüllt, bevor dieser Befehl verwendet wird, um die Informationen in Variablen der Hostsprache zu übertragen.

Dieser Befehl hat zwei Formen: Die erste Form ruft „Header“-Elemente des Descriptors ab, die für die Ergebnismenge als Ganzes gelten. Ein Beispiel ist die Zeilenanzahl. Die zweite Form, die die Spaltennummer als zusätzlichen Parameter erfordert, ruft Informationen über eine bestimmte Spalte ab. Beispiele sind der Spaltenname und der eigentliche Spaltenwert.

**Parameter**

`descriptor_name`

Ein Descriptorname.

`descriptor_header_item`

Ein Token, das angibt, welches Header-Informationselement abgerufen werden soll. Derzeit wird nur `COUNT` unterstützt, um die Anzahl der Spalten in der Ergebnismenge zu erhalten.

`column_number`

Die Nummer der Spalte, zu der Informationen abgerufen werden sollen. Die Zählung beginnt bei 1.

`descriptor_item`

Ein Token, das angibt, welches Informationselement zu einer Spalte abgerufen werden soll. Eine Liste der unterstützten Elemente finden Sie in [Abschnitt 34.7.1](#3471-benannte-sqldescriptorbereiche).

`cvariable`

Eine Hostvariable, die die aus dem Descriptor-Bereich abgerufenen Daten aufnimmt.

**Beispiele**

Ein Beispiel zum Abrufen der Anzahl der Spalten in einer Ergebnismenge:

```sql
EXEC SQL GET DESCRIPTOR d :d_count = COUNT;
```

Ein Beispiel zum Abrufen einer Datenlänge in der ersten Spalte:

```sql
EXEC SQL GET DESCRIPTOR d VALUE 1 :d_returned_octet_length =
 RETURNED_OCTET_LENGTH;
```

Ein Beispiel zum Abrufen des Datenkörpers der zweiten Spalte als Zeichenkette:

```sql
EXEC SQL GET DESCRIPTOR d VALUE 2 :d_data = DATA;
```

Hier ist ein Beispiel für den vollständigen Ablauf, `SELECT current_database();` auszuführen und die Anzahl der Spalten, die Spaltendatenlänge und die Spaltendaten anzuzeigen:

```c
int
main(void)
{
EXEC SQL BEGIN DECLARE SECTION;
    int d_count;
    char d_data[1024];
    int d_returned_octet_length;
EXEC SQL END DECLARE SECTION;

    EXEC SQL CONNECT TO testdb AS con1 USER testuser;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;
    EXEC SQL ALLOCATE DESCRIPTOR d;

    /* Declare, open a cursor, and assign a descriptor to the cursor */
    EXEC SQL DECLARE cur CURSOR FOR SELECT current_database();
    EXEC SQL OPEN cur;
    EXEC SQL FETCH NEXT FROM cur INTO SQL DESCRIPTOR d;

    /* Get a number of total columns */
    EXEC SQL GET DESCRIPTOR d :d_count = COUNT;
    printf("d_count                 = %d\n", d_count);

    /* Get length of a returned column */
    EXEC SQL GET DESCRIPTOR d VALUE 1 :d_returned_octet_length =
 RETURNED_OCTET_LENGTH;
    printf("d_returned_octet_length = %d\n",
 d_returned_octet_length);

    /* Fetch the returned column as a string */
    EXEC SQL GET DESCRIPTOR d VALUE 1 :d_data = DATA;
    printf("d_data                  = %s\n", d_data);

    /* Closing */
    EXEC SQL CLOSE cur;
    EXEC SQL COMMIT;

    EXEC SQL DEALLOCATE DESCRIPTOR d;
    EXEC SQL DISCONNECT ALL;

    return 0;
}
```

Wenn das Beispiel ausgeführt wird, sieht das Ergebnis so aus:

```text
d_count                 = 1
d_returned_octet_length = 6
d_data                  = testdb
```

**Kompatibilität**

`GET DESCRIPTOR` ist im SQL-Standard spezifiziert.

**Siehe auch**

`ALLOCATE DESCRIPTOR`, `SET DESCRIPTOR`

`OPEN`

`OPEN` - einen dynamischen Cursor öffnen.

**Syntax**

```sql
OPEN cursor_name
OPEN cursor_name USING value [, ... ]
OPEN cursor_name USING SQL DESCRIPTOR descriptor_name
```

**Beschreibung**

`OPEN` öffnet einen Cursor und bindet optional tatsächliche Werte an die Platzhalter in der Cursor-Deklaration. Der Cursor muss zuvor mit dem Befehl `DECLARE` deklariert worden sein. Die Ausführung von `OPEN` bewirkt, dass die Abfrage auf dem Server ausgeführt wird.

**Parameter**

`cursor_name`

Der Name des zu öffnenden Cursors. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

`value`

Ein Wert, der an einen Platzhalter im Cursor gebunden werden soll. Dies kann eine SQL-Konstante, eine Hostvariable oder eine Hostvariable mit Indikator sein.

`descriptor_name`

Der Name eines Descriptors, der Werte enthält, die an die Platzhalter im Cursor gebunden werden sollen. Dies kann ein SQL-Bezeichner oder eine Hostvariable sein.

**Beispiele**

```sql
EXEC SQL OPEN a;
EXEC SQL OPEN d USING 1, 'test';
EXEC SQL OPEN c1 USING SQL DESCRIPTOR mydesc;
EXEC SQL OPEN :curname1;
```

**Kompatibilität**

`OPEN` ist im SQL-Standard spezifiziert.

**Siehe auch**

`DECLARE`, `CLOSE`

`PREPARE`

`PREPARE` - eine Anweisung für die Ausführung vorbereiten.

**Syntax**

```sql
PREPARE prepared_name FROM string
```

**Beschreibung**

`PREPARE` bereitet eine als Zeichenkette dynamisch angegebene Anweisung für die Ausführung vor. Dies unterscheidet sich von der direkten SQL-Anweisung `PREPARE`, die ebenfalls in Embedded-Programmen verwendet werden kann. Der Befehl `EXECUTE` wird verwendet, um beide Arten vorbereiteter Anweisungen auszuführen.

**Parameter**

`prepared_name`

Ein Bezeichner für die vorbereitete Abfrage.

`string`

Eine Literalzeichenkette oder eine Hostvariable, die eine vorbereitbare SQL-Anweisung enthält, also `SELECT`, `INSERT`, `UPDATE` oder `DELETE`. Verwenden Sie Fragezeichen (`?`) für Parameterwerte, die bei der Ausführung geliefert werden.

**Hinweise**

Typischerweise ist `string` eine Hostvariablenreferenz auf eine Zeichenkette, die eine dynamisch konstruierte SQL-Anweisung enthält. Der Fall einer Literalzeichenkette ist nicht sehr nützlich; Sie könnten genauso gut direkt eine SQL-Anweisung `PREPARE` schreiben.

Wenn Sie eine Literalzeichenkette verwenden, beachten Sie, dass alle doppelten Anführungszeichen, die Sie in die SQL-Anweisung aufnehmen möchten, als oktale Escapes (`\042`) geschrieben werden müssen, nicht im üblichen C-Stil `\"`. Das liegt daran, dass sich die Zeichenkette innerhalb eines Abschnitts `EXEC SQL` befindet, sodass der ECPG-Lexer sie nach SQL-Regeln und nicht nach C-Regeln parst. Eingebettete Backslashes werden später nach C-Regeln behandelt; `\"` verursacht jedoch sofort einen Syntaxfehler, weil es als Ende des Literals verstanden wird.

**Beispiele**

```c
char *stmt = "SELECT * FROM test1 WHERE a = ? AND b = ?";

EXEC SQL ALLOCATE DESCRIPTOR outdesc;
EXEC SQL PREPARE foo FROM :stmt;

EXEC SQL EXECUTE foo USING SQL DESCRIPTOR indesc INTO SQL
 DESCRIPTOR outdesc;
```

**Kompatibilität**

`PREPARE` ist im SQL-Standard spezifiziert.

**Siehe auch**

`EXECUTE`

`SET AUTOCOMMIT`

`SET AUTOCOMMIT` - das Autocommit-Verhalten der aktuellen Sitzung setzen.

**Syntax**

```sql
SET AUTOCOMMIT { = | TO } { ON | OFF }
```

**Beschreibung**

`SET AUTOCOMMIT` setzt das Autocommit-Verhalten der aktuellen Datenbanksitzung. Standardmäßig befinden sich Embedded-SQL-Programme nicht im Autocommit-Modus, sodass `COMMIT` bei Bedarf ausdrücklich ausgegeben werden muss. Dieser Befehl kann die Sitzung in den Autocommit-Modus versetzen, in dem jede einzelne Anweisung implizit committet wird.

**Kompatibilität**

`SET AUTOCOMMIT` ist eine Erweiterung von PostgreSQL ECPG.

`SET CONNECTION`

`SET CONNECTION` - eine Datenbankverbindung auswählen.

**Syntax**

```sql
SET CONNECTION [ TO | = ] connection_name
```

**Beschreibung**

`SET CONNECTION` setzt die „aktuelle“ Datenbankverbindung. Das ist die Verbindung, die alle Befehle verwenden, sofern sie nicht überschrieben wird.

**Parameter**

`connection_name`

Ein Datenbankverbindungsname, der durch den Befehl `CONNECT` hergestellt wurde.

`CURRENT`

Setzt die Verbindung auf die aktuelle Verbindung; dadurch geschieht also nichts.

**Beispiele**

```sql
EXEC SQL SET CONNECTION TO con2;
EXEC SQL SET CONNECTION = con1;
```

**Kompatibilität**

`SET CONNECTION` ist im SQL-Standard spezifiziert.

**Siehe auch**

`CONNECT`, `DISCONNECT`

`SET DESCRIPTOR`

`SET DESCRIPTOR` - Informationen in einem SQL-Descriptor-Bereich setzen.

**Syntax**

```sql
SET DESCRIPTOR descriptor_name descriptor_header_item = value
 [, ... ]
SET DESCRIPTOR descriptor_name VALUE number descriptor_item = value
 [, ...]
```

**Beschreibung**

`SET DESCRIPTOR` füllt einen SQL-Descriptor-Bereich mit Werten. Der Descriptor-Bereich wird danach typischerweise verwendet, um Parameter in einer vorbereiteten Abfrageausführung zu binden.

Dieser Befehl hat zwei Formen: Die erste Form gilt für den „Header“ des Descriptors, der unabhängig von einem bestimmten Datum ist. Die zweite Form weist bestimmten, durch Nummern identifizierten Daten Werte zu.

**Parameter**

`descriptor_name`

Ein Descriptorname.

`descriptor_header_item`

Ein Token, das angibt, welches Header-Informationselement gesetzt werden soll. Derzeit wird nur `COUNT` unterstützt, um die Anzahl der Descriptor-Elemente zu setzen.

`number`

Die Nummer des zu setzenden Descriptor-Elements. Die Zählung beginnt bei 1.

`descriptor_item`

Ein Token, das angibt, welches Informationselement im Descriptor gesetzt werden soll. Eine Liste der unterstützten Elemente finden Sie in [Abschnitt 34.7.1](#3471-benannte-sqldescriptorbereiche).

`value`

Ein Wert, der im Descriptor-Element gespeichert werden soll. Dies kann eine SQL-Konstante oder eine Hostvariable sein.

**Beispiele**

```sql
EXEC SQL SET DESCRIPTOR indesc COUNT = 1;
EXEC SQL SET DESCRIPTOR indesc VALUE 1 DATA = 2;
EXEC SQL SET DESCRIPTOR indesc VALUE 1 DATA = :val1;
EXEC SQL SET DESCRIPTOR indesc VALUE 2 INDICATOR = :val1, DATA =
 'some string';
EXEC SQL SET DESCRIPTOR indesc VALUE 2 INDICATOR = :val2null, DATA
 = :val2;
```

**Kompatibilität**

`SET DESCRIPTOR` ist im SQL-Standard spezifiziert.

**Siehe auch**

`ALLOCATE DESCRIPTOR`, `GET DESCRIPTOR`

`TYPE`

`TYPE` - einen neuen Datentyp definieren.

**Syntax**

```sql
TYPE type_name IS ctype
```

**Beschreibung**

Der Befehl `TYPE` definiert einen neuen C-Typ. Er entspricht einem `typedef` in einem Deklarationsabschnitt.

Dieser Befehl wird nur erkannt, wenn `ecpg` mit der Option `-c` ausgeführt wird.

**Parameter**

`type_name`

Der Name für den neuen Typ. Er muss ein gültiger C-Typname sein.

`ctype`

Eine C-Typspezifikation.

**Beispiele**

```c
EXEC SQL TYPE customer IS
    struct
    {
        varchar name[50];
        int     phone;
    };

EXEC SQL TYPE cust_ind IS
    struct ind
    {
        short   name_ind;
        short   phone_ind;
    };

EXEC SQL TYPE c IS char reference;
EXEC SQL TYPE ind IS union { int integer; short smallint; };
EXEC SQL TYPE intarray IS int[AMOUNT];
EXEC SQL TYPE str IS varchar[BUFFERSIZ];
EXEC SQL TYPE string IS char[11];
```

Hier ist ein Beispielprogramm, das `EXEC SQL TYPE` verwendet:

```c
EXEC SQL WHENEVER SQLERROR SQLPRINT;

EXEC SQL TYPE tt IS
    struct
    {
        varchar v[256];
        int     i;
    };

EXEC SQL TYPE tt_ind IS
    struct ind {
        short    v_ind;
        short    i_ind;
    };

int
main(void)
{
EXEC SQL BEGIN DECLARE SECTION;
    tt t;
    tt_ind t_ind;
EXEC SQL END DECLARE SECTION;

    EXEC SQL CONNECT TO testdb AS con1;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;

    EXEC SQL SELECT current_database(), 256 INTO :t:t_ind LIMIT 1;

    printf("t.v = %s\n", t.v.arr);
    printf("t.i = %d\n", t.i);

    printf("t_ind.v_ind = %d\n", t_ind.v_ind);
    printf("t_ind.i_ind = %d\n", t_ind.i_ind);

    EXEC SQL DISCONNECT con1;

    return 0;
}
```

Die Ausgabe dieses Programms sieht so aus:

```text
t.v = testdb
t.i = 256
t_ind.v_ind = 0
t_ind.i_ind = 0
```

**Kompatibilität**

Der Befehl `TYPE` ist eine PostgreSQL-Erweiterung.

`VAR`

`VAR` - eine Variable definieren.

**Syntax**

```sql
VAR varname IS ctype
```

**Beschreibung**

Der Befehl `VAR` weist einer Hostvariable einen neuen C-Datentyp zu. Die Hostvariable muss zuvor in einem Deklarationsabschnitt deklariert worden sein.

**Parameter**

`varname`

Ein C-Variablenname.

`ctype`

Eine C-Typspezifikation.

**Beispiele**

```c
Exec sql begin declare section;
short a;
exec sql end declare section;
EXEC SQL VAR a IS int;
```

**Kompatibilität**

Der Befehl `VAR` ist eine PostgreSQL-Erweiterung.

`WHENEVER`

`WHENEVER` - die Aktion angeben, die ausgeführt werden soll, wenn eine SQL-Anweisung eine bestimmte Bedingungsklasse auslöst.

**Syntax**

```sql
WHENEVER { NOT FOUND | SQLERROR | SQLWARNING } action
```

**Beschreibung**

Definiert ein Verhalten, das in besonderen Fällen beim Ergebnis einer SQL-Ausführung aufgerufen wird, also wenn Zeilen nicht gefunden wurden, SQL-Warnungen auftreten oder Fehler entstehen.

**Parameter**

Eine Beschreibung der Parameter finden Sie in [Abschnitt 34.8.1](#3481-callbacks-setzen).

**Beispiele**

```sql
EXEC SQL WHENEVER NOT FOUND CONTINUE;
EXEC SQL WHENEVER NOT FOUND DO BREAK;
EXEC SQL WHENEVER NOT FOUND DO CONTINUE;
EXEC SQL WHENEVER SQLWARNING SQLPRINT;
EXEC SQL WHENEVER SQLWARNING DO warn();
EXEC SQL WHENEVER SQLERROR sqlprint;
EXEC SQL WHENEVER SQLERROR CALL print2();
EXEC SQL WHENEVER SQLERROR DO handle_error("select");
EXEC SQL WHENEVER SQLERROR DO sqlnotice(NULL, NONO);
EXEC SQL WHENEVER SQLERROR DO sqlprint();
EXEC SQL WHENEVER SQLERROR GOTO error_label;
EXEC SQL WHENEVER SQLERROR STOP;
```

Eine typische Anwendung ist die Verwendung von `WHENEVER NOT FOUND BREAK`, um eine Schleife über Ergebnismengen zu behandeln:

```c
int
main(void)
{
    EXEC SQL CONNECT TO testdb AS con1;
    EXEC SQL SELECT pg_catalog.set_config('search_path', '', false);
    EXEC SQL COMMIT;
    EXEC SQL ALLOCATE DESCRIPTOR d;
    EXEC SQL DECLARE cur CURSOR FOR SELECT current_database(),
 'hoge', 256;
    EXEC SQL OPEN cur;

    /* when end of result set reached, break out of while loop */
    EXEC SQL WHENEVER NOT FOUND DO BREAK;

    while (1)
    {
        EXEC SQL FETCH NEXT FROM cur INTO SQL DESCRIPTOR d;
        ...
    }

    EXEC SQL CLOSE cur;
    EXEC SQL COMMIT;

    EXEC SQL DEALLOCATE DESCRIPTOR d;
    EXEC SQL DISCONNECT ALL;

    return 0;
}
```

**Kompatibilität**

`WHENEVER` ist im SQL-Standard spezifiziert, aber die meisten Aktionen sind PostgreSQL-Erweiterungen.

## 34.15. Informix-Kompatibilitätsmodus

`ecpg` kann in einem sogenannten Informix-Kompatibilitätsmodus ausgeführt werden. Wenn dieser Modus aktiv ist, versucht `ecpg`, sich wie der Informix-Präcompiler für Informix E/SQL zu verhalten. Allgemein erlaubt Ihnen das, das Dollarzeichen statt des Primitivs `EXEC SQL` zu verwenden, um Embedded-SQL-Befehle einzuleiten:

```c
$int j = 3;
$CONNECT TO :dbname;
$CREATE TABLE test(i INT PRIMARY KEY, j INT);
$INSERT INTO test(i, j) VALUES (7, :j);
$COMMIT;
```

> Hinweis: Zwischen dem `$` und einer folgenden Präprozessor-Direktive, also `include`, `define`, `ifdef` usw., darf kein Leerraum stehen. Andernfalls parst der Präprozessor das Token als Hostvariable.

Es gibt zwei Kompatibilitätsmodi: `INFORMIX` und `INFORMIX_SE`.

Wenn Sie Programme linken, die diesen Kompatibilitätsmodus verwenden, denken Sie daran, gegen `libcompat` zu linken, die mit ECPG ausgeliefert wird.

Neben der zuvor erklärten syntaktischen Erleichterung portiert der Informix-Kompatibilitätsmodus einige aus E/SQL bekannte Funktionen für Eingabe, Ausgabe und Transformation von Daten sowie Embedded-SQL-Anweisungen nach ECPG.

Der Informix-Kompatibilitätsmodus ist eng mit der Bibliothek `pgtypeslib` von ECPG verbunden. `pgtypeslib` bildet SQL-Datentypen auf Datentypen im C-Hostprogramm ab, und die meisten zusätzlichen Funktionen des Informix-Kompatibilitätsmodus erlauben Operationen auf diesen C-Hostprogrammtypen. Beachten Sie jedoch, dass der Umfang der Kompatibilität begrenzt ist. Sie versucht nicht, Informix-Verhalten exakt zu kopieren; sie erlaubt mehr oder weniger dieselben Operationen und stellt Funktionen bereit, die denselben Namen und dasselbe grundlegende Verhalten haben. Sie ist aber kein Drop-in-Ersatz, wenn Sie derzeit Informix verwenden. Außerdem unterscheiden sich einige Datentypen. PostgreSQLs `datetime`- und `interval`-Typen kennen zum Beispiel keine Bereiche wie `YEAR TO MINUTE`, daher finden Sie dafür auch in ECPG keine Unterstützung.

### 34.15.1. Zusätzliche Typen

Der Informix-spezielle Pseudotyp `string` zum Speichern rechts getrimmter Zeichenkettendaten wird im Informix-Modus nun ohne `typedef` unterstützt. Tatsächlich lehnt ECPG im Informix-Modus Quelldateien ab, die `typedef sometype string;` enthalten.

```c
EXEC SQL BEGIN DECLARE SECTION;
string userid; /* this variable will contain trimmed data */
EXEC SQL END DECLARE SECTION;

EXEC SQL FETCH MYCUR INTO :userid;
```

### 34.15.2. Zusätzliche oder fehlende Embedded-SQL-Anweisungen

`CLOSE DATABASE`

Diese Anweisung schließt die aktuelle Verbindung. Tatsächlich ist sie ein Synonym für ECPGs `DISCONNECT CURRENT`:

```c
$CLOSE DATABASE;                                /* close the current connection */
EXEC SQL CLOSE DATABASE;
```

`FREE cursor_name`

Aufgrund von Unterschieden zwischen der Arbeitsweise von ECPG und Informix ESQL/C, insbesondere welche Schritte reine Grammatiktransformationen sind und welche Schritte auf die zugrunde liegende Laufzeitbibliothek angewiesen sind, gibt es in ECPG keine Anweisung `FREE cursor_name`. Der Grund ist, dass `DECLARE CURSOR` in ECPG nicht in einen Funktionsaufruf an die Laufzeitbibliothek übersetzt wird, der den Cursornamen verwendet. Das bedeutet, dass es in der ECPG-Laufzeitbibliothek keine Laufzeitbuchführung über SQL-Cursor gibt, sondern nur im PostgreSQL-Server.

`FREE statement_name`

`FREE statement_name` ist ein Synonym für `DEALLOCATE PREPARE statement_name`.

### 34.15.3. Informix-kompatible SQLDA-Descriptor-Bereiche

Der Informix-kompatible Modus unterstützt eine andere Struktur als die in [Abschnitt 34.7.2](#3472-sqldadescriptorbereiche) beschriebene. Sie sieht wie folgt aus:

```c
struct sqlvar_compat
{
    short   sqltype;
    int     sqllen;
    char   *sqldata;
    short *sqlind;
    char   *sqlname;
    char   *sqlformat;
    short   sqlitype;
    short   sqlilen;
    char   *sqlidata;
    int     sqlxid;
    char   *sqltypename;
    short   sqltypelen;
    short   sqlownerlen;
    short   sqlsourcetype;
    char   *sqlownername;
    int     sqlsourceid;
    char   *sqlilongdata;
    int         sqlflags;
    void       *sqlreserved;
};

struct sqlda_compat
{
    short sqld;
    struct sqlvar_compat *sqlvar;
    char   desc_name[19];
    short desc_occ;
    struct sqlda_compat *desc_next;
    void *reserved;
};

typedef struct sqlvar_compat                       sqlvar_t;
typedef struct sqlda_compat                        sqlda_t;
```

Die globalen Eigenschaften sind:

`sqld`

Die Anzahl der Felder im SQLDA-Descriptor.

`sqlvar`

Zeiger auf die feldspezifischen Eigenschaften.

`desc_name`

Nicht verwendet, mit Nullbytes gefüllt.

`desc_occ`

Größe der allozierten Struktur.

`desc_next`

Zeiger auf die nächste SQLDA-Struktur, wenn die Ergebnismenge mehr als einen Datensatz enthält.

`reserved`

Nicht verwendeter Zeiger, enthält `NULL`. Für Informix-Kompatibilität beibehalten.

Die feldspezifischen Eigenschaften stehen unten; sie werden im Array `sqlvar` gespeichert:

`sqltype`

Typ des Felds. Konstanten befinden sich in `sqltypes.h`.

`sqllen`

Länge der Felddaten.

`sqldata`

Zeiger auf die Felddaten. Der Zeiger hat den Typ `char *`; die Daten, auf die er zeigt, liegen im Binärformat vor. Beispiel:

```c
int intval;

switch (sqldata->sqlvar[i].sqltype)
{
    case SQLINTEGER:
        intval = *(int *)sqldata->sqlvar[i].sqldata;
        break;
    ...
}
```

`sqlind`

Zeiger auf den `NULL`-Indikator. Wenn er von `DESCRIBE` oder `FETCH` zurückgegeben wird, ist er immer ein gültiger Zeiger. Wenn er als Eingabe für `EXECUTE ... USING sqlda;` verwendet wird, bedeutet ein `NULL`-Zeigerwert, dass der Wert für dieses Feld nicht `NULL` ist. Andernfalls muss ein gültiger Zeiger vorhanden und `sqlitype` korrekt gesetzt sein. Beispiel:

```c
if (*(int2 *)sqldata->sqlvar[i].sqlind != 0)
    printf("value is NULL\n");
```

`sqlname`

Name des Felds als nullterminierte Zeichenkette.

`sqlformat`

In Informix reserviert; Wert von `PQfformat` für das Feld.

`sqlitype`

Typ der Daten des `NULL`-Indikators. Bei Daten, die vom Server zurückgegeben werden, ist dies immer `SQLSMINT`. Wenn die SQLDA für eine parametrisierte Abfrage verwendet wird, werden die Daten gemäß dem gesetzten Typ behandelt.

`sqlilen`

Länge der Daten des `NULL`-Indikators.

`sqlxid`

Erweiterter Typ des Felds, Ergebnis von `PQftype`.

`sqltypename`, `sqltypelen`, `sqlownerlen`, `sqlsourcetype`, `sqlownername`, `sqlsourceid`, `sqlflags`, `sqlreserved`

Nicht verwendet.

`sqlilongdata`

Entspricht `sqldata`, wenn `sqllen` größer als 32 kB ist.

Beispiel:

```c
EXEC SQL INCLUDE sqlda.h;

sqlda_t        *sqlda; /* This doesn't need to be under
 embedded DECLARE SECTION */

EXEC SQL BEGIN DECLARE SECTION;
char *prep_stmt = "select * from table1";
int i;
EXEC SQL END DECLARE SECTION;

...

EXEC SQL PREPARE mystmt FROM :prep_stmt;

EXEC SQL DESCRIBE mystmt INTO sqlda;

printf("# of fields: %d\n", sqlda->sqld);
for (i = 0; i < sqlda->sqld; i++)
    printf("field %d: \"%s\"\n", sqlda->sqlvar[i]->sqlname);

EXEC SQL DECLARE mycursor CURSOR FOR mystmt;
EXEC SQL OPEN mycursor;
EXEC SQL WHENEVER NOT FOUND GOTO out;

while (1)
{
    EXEC SQL FETCH mycursor USING sqlda;
}

EXEC SQL CLOSE mycursor;

free(sqlda); /* The main structure is all to be free(),
              * sqlda and sqlda->sqlvar is in one allocated
              * area */
```

Weitere Informationen finden Sie im Header `sqlda.h` und im Regressionstest `src/interfaces/ecpg/test/compat_informix/sqlda.pgc`.

### 34.15.4. Zusätzliche Funktionen

`decadd`

Addiert zwei Werte vom Typ `decimal`.

```c
int decadd(decimal *arg1, decimal *arg2, decimal *sum);
```

Die Funktion erhält Zeiger auf den ersten und zweiten Operanden vom Typ `decimal` (`arg1`, `arg2`) sowie einen Zeiger auf einen `decimal`-Wert, der die Summe aufnehmen soll (`sum`). Bei Erfolg gibt sie 0 zurück. Bei Überlauf wird `ECPG_INFORMIX_NUM_OVERFLOW`, bei Unterlauf `ECPG_INFORMIX_NUM_UNDERFLOW` zurückgegeben. Bei anderen Fehlern wird -1 zurückgegeben und `errno` auf die jeweilige `errno`-Nummer von `pgtypeslib` gesetzt.

`deccmp`

Vergleicht zwei Variablen vom Typ `decimal`.

```c
int deccmp(decimal *arg1, decimal *arg2);
```

Die Funktion erhält Zeiger auf zwei `decimal`-Werte und gibt eine Ganzzahl zurück, die angibt, welcher Wert größer ist:

- `1`, wenn der Wert, auf den `arg1` zeigt, größer ist als der Wert, auf den `arg2` zeigt
- `-1`, wenn der Wert, auf den `arg1` zeigt, kleiner ist als der Wert, auf den `arg2` zeigt
- `0`, wenn beide Werte gleich sind

`deccopy`

Kopiert einen `decimal`-Wert.

```c
void deccopy(decimal *src, decimal *target);
```

Die Funktion erhält als erstes Argument einen Zeiger auf den zu kopierenden `decimal`-Wert (`src`) und als zweites Argument einen Zeiger auf die Zielstruktur vom Typ `decimal` (`target`).

`deccvasc`

Konvertiert einen Wert aus seiner ASCII-Darstellung in den Typ `decimal`.

```c
int deccvasc(char *cp, int len, decimal *np);
```

Die Funktion erhält einen Zeiger auf eine Zeichenkette mit der Textdarstellung der zu konvertierenden Zahl (`cp`) sowie deren Länge `len`. `np` ist ein Zeiger auf den `decimal`-Wert, der das Ergebnis speichert. Gültige Formate sind zum Beispiel `-2`, `.794`, `+3.44`, `592.49E07` oder `-32.84e-4`.

Bei Erfolg gibt die Funktion 0 zurück. Bei Überlauf oder Unterlauf wird `ECPG_INFORMIX_NUM_OVERFLOW` beziehungsweise `ECPG_INFORMIX_NUM_UNDERFLOW` zurückgegeben. Wenn die ASCII-Darstellung nicht geparst werden konnte, wird `ECPG_INFORMIX_BAD_NUMERIC` zurückgegeben, oder `ECPG_INFORMIX_BAD_EXPONENT`, wenn das Problem beim Parsen des Exponenten auftrat.

`deccvdbl`, `deccvint`, `deccvlong`

Konvertieren Werte vom Typ `double`, `int` beziehungsweise `long` in Werte vom Typ `decimal`.

```c
int deccvdbl(double dbl, decimal *np);
int deccvint(int in, decimal *np);
int deccvlong(long lng, decimal *np);
```

Die Funktionen erhalten den zu konvertierenden Wert als erstes Argument und als zweites Argument einen Zeiger auf die `decimal`-Variable, die das Ergebnis aufnehmen soll. Bei Erfolg geben sie 0 zurück, bei fehlgeschlagener Konvertierung einen negativen Wert.

`decdiv`, `decmul`, `decsub`

Dividieren, multiplizieren beziehungsweise subtrahieren zwei Variablen vom Typ `decimal`.

```c
int decdiv(decimal *n1, decimal *n2, decimal *result);
int decmul(decimal *n1, decimal *n2, decimal *result);
int decsub(decimal *n1, decimal *n2, decimal *result);
```

Die Funktionen erhalten Zeiger auf die beiden Operanden (`n1`, `n2`) und berechnen `n1/n2`, `n1*n2` beziehungsweise `n1-n2`. `result` ist ein Zeiger auf die Variable, die das Ergebnis aufnehmen soll. Bei Erfolg wird 0 zurückgegeben, bei Fehlern ein negativer Wert. Bei Überlauf oder Unterlauf geben die Funktionen `ECPG_INFORMIX_NUM_OVERFLOW` beziehungsweise `ECPG_INFORMIX_NUM_UNDERFLOW` zurück; `decdiv` gibt bei Division durch null `ECPG_INFORMIX_DIVIDE_ZERO` zurück.

`dectoasc`

Konvertiert eine Variable vom Typ `decimal` in ihre ASCII-Darstellung in einer C-Zeichenkette vom Typ `char *`.

```c
int dectoasc(decimal *np, char *cp, int len, int right)
```

Die Funktion erhält einen Zeiger auf eine `decimal`-Variable (`np`), die in ihre Textdarstellung konvertiert wird. `cp` ist der Ergebnispuffer. Der Parameter `right` legt fest, wie viele Ziffern rechts vom Dezimalpunkt in die Ausgabe aufgenommen werden. Das Ergebnis wird auf diese Anzahl von Nachkommastellen gerundet. `right = -1` bedeutet, dass alle verfügbaren Dezimalziffern ausgegeben werden sollen. Wenn der durch `len` angegebene Ausgabepuffer nicht ausreicht, wird nur ein einzelnes Zeichen `*` gespeichert und -1 zurückgegeben.

Die Funktion gibt entweder -1 zurück, wenn der Puffer `cp` zu klein war, oder `ECPG_INFORMIX_OUT_OF_MEMORY`, wenn der Speicher erschöpft war.

`dectodbl`, `dectoint`, `dectolong`

Konvertieren eine Variable vom Typ `decimal` in `double`, `int` beziehungsweise `long`.

```c
int dectodbl(decimal *np, double *dblp);
int dectoint(decimal *np, int *ip);
int dectolong(decimal *np, long *lngp);
```

Die Funktionen erhalten einen Zeiger auf den zu konvertierenden `decimal`-Wert und einen Zeiger auf die Zielvariable. Bei Erfolg geben sie 0 zurück, bei fehlgeschlagener Konvertierung einen negativen Wert. Bei `dectoint` und `dectolong` wird bei Überlauf `ECPG_INFORMIX_NUM_OVERFLOW` zurückgegeben. Die ECPG-Implementierung unterscheidet sich von Informix: Informix beschränkt `int` auf `-32767` bis `32767` und `long` auf `-2,147,483,647` bis `2,147,483,647`, während die Grenzen in ECPG von der Architektur abhängen.

`rdatestr`

Konvertiert ein Datum in eine C-Zeichenkette vom Typ `char *`.

```c
int rdatestr(date d, char *str);
```

Die Funktion erhält das zu konvertierende Datum (`d`) und einen Zeiger auf die Zielzeichenkette. Das Ausgabeformat ist immer `yyyy-mm-dd`, daher müssen mindestens 11 Byte einschließlich Nullterminator reserviert werden. Bei Erfolg gibt die Funktion 0 zurück, bei einem Fehler einen negativen Wert. Anders als in Informix kann das Ausgabeformat in ECPG nicht über Umgebungsvariablen geändert werden.

`rstrdate`

Parst die Textdarstellung eines Datums.

```c
int rstrdate(char *str, date *d);
```

Die Funktion erhält die Textdarstellung des zu konvertierenden Datums (`str`) und einen Zeiger auf eine Variable vom Typ `date` (`d`). Eine Formatmaske kann nicht angegeben werden; die Funktion verwendet die Informix-Standardmaske `mm/dd/yyyy`. Intern ist sie über `rdefmtdate` implementiert. Wenn möglich, sollten Sie daher `rdefmtdate` verwenden, weil dort die Formatmaske ausdrücklich angegeben werden kann. Die Rückgabewerte entsprechen denen von `rdefmtdate`.

`rtoday`

Ermittelt das aktuelle Datum.

```c
void rtoday(date *d);
```

Die Funktion erhält einen Zeiger auf eine `date`-Variable (`d`) und setzt sie auf das aktuelle Datum. Intern verwendet sie `PGTYPESdate_today`.

`rjulmdy`

Extrahiert Tag, Monat und Jahr aus einer Variable vom Typ `date`.

```c
int rjulmdy(date d, short mdy[3]);
```

Die Funktion erhält das Datum `d` und einen Zeiger auf ein Array aus drei `short`-Ganzzahlwerten `mdy`. `mdy[0]` enthält die Monatsnummer, `mdy[1]` den Tag und `mdy[2]` das Jahr. Derzeit gibt die Funktion immer 0 zurück. Intern verwendet sie `PGTYPESdate_julmdy`.

`rdefmtdate`

Verwendet eine Formatmaske, um eine Zeichenkette in einen Wert vom Typ `date` zu konvertieren.

```c
int rdefmtdate(date *d, char *fmt, char *str);
```

Die Funktion erhält einen Zeiger auf den Ergebniswert (`d`), die Formatmaske (`fmt`) und die C-Zeichenkette mit der Textdarstellung des Datums (`str`). Die Darstellung sollte zur Formatmaske passen; eine 1:1-Abbildung ist jedoch nicht erforderlich. Die Funktion analysiert nur die Reihenfolge und sucht nach `yy` oder `yyyy` für die Position des Jahres, `mm` für den Monat und `dd` für den Tag.

Rückgabewerte:

- `0` - die Funktion wurde erfolgreich beendet
- `ECPG_INFORMIX_ENOSHORTDATE` - das Datum enthält keine Trennzeichen zwischen Tag, Monat und Jahr; die Eingabe muss dann genau 6 oder 8 Byte lang sein, ist es aber nicht
- `ECPG_INFORMIX_ENOTDMY` - die Formatzeichenkette gab die Reihenfolge von Jahr, Monat und Tag nicht korrekt an
- `ECPG_INFORMIX_BAD_DAY` - die Eingabezeichenkette enthält keinen gültigen Tag
- `ECPG_INFORMIX_BAD_MONTH` - die Eingabezeichenkette enthält keinen gültigen Monat
- `ECPG_INFORMIX_BAD_YEAR` - die Eingabezeichenkette enthält kein gültiges Jahr

Intern verwendet diese Funktion `PGTYPESdate_defmt_asc`; dort finden Sie eine Tabelle mit Beispieleingaben.

`rfmtdate`

Konvertiert eine Variable vom Typ `date` mithilfe einer Formatmaske in ihre Textdarstellung.

```c
int rfmtdate(date d, char *fmt, char *str);
```

Die Funktion erhält das zu konvertierende Datum (`d`), die Formatmaske (`fmt`) und die Zeichenkette für die Textdarstellung (`str`). Bei Erfolg wird 0 zurückgegeben, bei einem Fehler ein negativer Wert. Intern verwendet sie `PGTYPESdate_fmt_asc`; dort finden Sie Beispiele.

`rmdyjul`

Erzeugt einen Datumswert aus einem Array von drei `short`-Ganzzahlen, die Tag, Monat und Jahr angeben.

```c
int rmdyjul(short mdy[3], date *d);
```

Die Funktion erhält das Array `mdy` und einen Zeiger auf eine `date`-Variable, die das Ergebnis aufnehmen soll. Derzeit gibt die Funktion immer 0 zurück. Intern verwendet sie `PGTYPESdate_mdyjul`.

`rdayofweek`

Gibt eine Zahl zurück, die den Wochentag eines Datumswerts darstellt.

```c
int rdayofweek(date d);
```

Die Funktion erhält das Datum `d` und gibt eine Ganzzahl für den Wochentag zurück:

- `0` - Sonntag
- `1` - Montag
- `2` - Dienstag
- `3` - Mittwoch
- `4` - Donnerstag
- `5` - Freitag
- `6` - Samstag

Intern verwendet sie `PGTYPESdate_dayofweek`.

`dtcurrent`

Ruft den aktuellen Zeitstempel ab.

```c
void dtcurrent(timestamp *ts);
```

Die Funktion ruft den aktuellen Zeitstempel ab und speichert ihn in der `timestamp`-Variable, auf die `ts` zeigt.

`dtcvasc`

Parst einen Zeitstempel aus seiner Textdarstellung in eine `timestamp`-Variable.

```c
int dtcvasc(char *str, timestamp *ts);
```

Die Funktion erhält die zu parsende Zeichenkette (`str`) und einen Zeiger auf die Zielvariable (`ts`). Bei Erfolg gibt sie 0 zurück, bei einem Fehler einen negativen Wert. Intern verwendet sie `PGTYPEStimestamp_from_asc`; dort finden Sie Beispieleingaben.

`dtcvfmtasc`

Parst einen Zeitstempel aus seiner Textdarstellung mithilfe einer Formatmaske in eine `timestamp`-Variable.

```c
dtcvfmtasc(char *inbuf, char *fmtstr, timestamp *dtvalue)
```

Die Funktion erhält die zu parsende Zeichenkette (`inbuf`), die Formatmaske (`fmtstr`) und einen Zeiger auf die Zielvariable (`dtvalue`). Sie ist über `PGTYPEStimestamp_defmt_asc` implementiert; dort finden Sie die verwendbaren Formatbezeichner. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einen negativen Wert.

`dtsub`

Subtrahiert einen Zeitstempel von einem anderen und gibt eine Variable vom Typ `interval` zurück.

```c
int dtsub(timestamp *ts1, timestamp *ts2, interval *iv);
```

Die Funktion subtrahiert die `timestamp`-Variable, auf die `ts2` zeigt, von der `timestamp`-Variable, auf die `ts1` zeigt, und speichert das Ergebnis in der `interval`-Variable, auf die `iv` zeigt. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einen negativen Wert.

`dttoasc`

Konvertiert eine `timestamp`-Variable in eine C-Zeichenkette vom Typ `char *`.

```c
int dttoasc(timestamp *ts, char *output);
```

Die Funktion erhält einen Zeiger auf die zu konvertierende `timestamp`-Variable (`ts`) und die Zielzeichenkette (`output`). Sie konvertiert `ts` gemäß SQL-Standard in die Textdarstellung `YYYY-MM-DD HH:MM:SS`. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einen negativen Wert.

`dttofmtasc`

Konvertiert eine `timestamp`-Variable mithilfe einer Formatmaske in ein C-`char *`.

```c
int dttofmtasc(timestamp *ts, char *output, int str_len, char
 *fmtstr);
```

Die Funktion erhält den Zeitstempel (`ts`), den Ausgabepuffer (`output`), die maximale Länge des Ausgabepuffers (`str_len`) und die Formatmaske (`fmtstr`). Bei Erfolg gibt sie 0 zurück, bei einem Fehler einen negativen Wert. Intern verwendet sie `PGTYPEStimestamp_fmt_asc`.

`intoasc`

Konvertiert eine `interval`-Variable in eine C-Zeichenkette vom Typ `char *`.

```c
int intoasc(interval *i, char *str);
```

Die Funktion erhält einen Zeiger auf die zu konvertierende Intervallvariable (`i`) und die Zielzeichenkette (`str`). Sie konvertiert `i` gemäß SQL-Standard in die Textdarstellung. Bei Erfolg gibt sie 0 zurück, bei einem Fehler einen negativen Wert.

`rfmtlong`

Konvertiert einen `long`-Ganzzahlwert mithilfe einer Formatmaske in seine Textdarstellung.

```c
int rfmtlong(long lng_val, char *fmt, char *outbuf);
```

Die Funktion erhält den `long`-Wert `lng_val`, die Formatmaske `fmt` und den Ausgabepuffer `outbuf`. Die Formatmaske kann folgende Zeichen enthalten:

- `*` - füllt eine sonst leere Position mit einem Stern
- `&` - füllt eine sonst leere Position mit einer Null
- `#` - wandelt führende Nullen in Leerzeichen um
- `<` - richtet die Zahl in der Zeichenkette linksbündig aus
- `,` - gruppiert Zahlen mit vier oder mehr Ziffern in Dreiergruppen, getrennt durch Kommas
- `.` - trennt den ganzzahligen Teil vom Nachkommateil
- `-` - zeigt ein Minuszeichen bei negativen Werten
- `+` - zeigt ein Pluszeichen bei positiven Werten
- `(` - ersetzt das Minuszeichen vor einer negativen Zahl
- `)` - ersetzt das Minuszeichen und wird hinter dem negativen Wert ausgegeben
- `$` - Währungssymbol

`rupshift`

Konvertiert eine Zeichenkette in Großbuchstaben.

```c
void rupshift(char *str);
```

Die Funktion erhält einen Zeiger auf die Zeichenkette und wandelt jeden Kleinbuchstaben in einen Großbuchstaben um.

`byleng`

Gibt die Anzahl der Zeichen in einer Zeichenkette zurück, ohne abschließende Leerzeichen mitzuzählen.

```c
int byleng(char *str, int len);
```

Die Funktion erwartet eine Zeichenkette fester Länge (`str`) und deren Länge (`len`). Sie gibt die Anzahl der signifikanten Zeichen zurück, also die Länge ohne abschließende Leerzeichen.

`ldchar`

Kopiert eine Zeichenkette fester Länge in eine nullterminierte Zeichenkette.

```c
void ldchar(char *src, int len, char *dest);
```

Die Funktion erhält die zu kopierende Zeichenkette fester Länge (`src`), ihre Länge (`len`) und einen Zeiger auf den Zielspeicher (`dest`). Für `dest` müssen mindestens `len+1` Byte reserviert werden. Die Funktion kopiert höchstens `len` Byte, weniger bei abschließenden Leerzeichen, und fügt den Nullterminator hinzu.

`rgetmsg`, `rtypalign`, `rtypmsize`, `rtypwidth`

Diese Funktionen existieren, sind derzeit aber nicht implementiert.

```c
int rgetmsg(int msgnum, char *s, int maxsize);
int rtypalign(int offset, int type);
int rtypmsize(int type, int len);
int rtypwidth(int sqltype, int sqllen);
```

`rsetnull`

Setzt eine Variable auf `NULL`.

```c
int rsetnull(int t, char *ptr);
```

Die Funktion erhält eine Ganzzahl, die den Typ der Variable angibt, und einen Zeiger auf die Variable selbst, der in einen C-Zeiger vom Typ `char *` gecastet ist.

Die folgenden Typen existieren:

- `CCHARTYPE` - für eine Variable vom Typ `char` oder `char *`
- `CSHORTTYPE` - für eine Variable vom Typ `short int`
- `CINTTYPE` - für eine Variable vom Typ `int`
- `CBOOLTYPE` - für eine Variable vom Typ `boolean`
- `CFLOATTYPE` - für eine Variable vom Typ `float`
- `CLONGTYPE` - für eine Variable vom Typ `long`
- `CDOUBLETYPE` - für eine Variable vom Typ `double`
- `CDECIMALTYPE` - für eine Variable vom Typ `decimal`
- `CDATETYPE` - für eine Variable vom Typ `date`
- `CDTIMETYPE` - für eine Variable vom Typ `timestamp`

Beispiel:

```c
$char c[] = "abc                    ";
$short s = 17;
$int i = -74874;

rsetnull(CCHARTYPE, (char *) c);
rsetnull(CSHORTTYPE, (char *) &s);
rsetnull(CINTTYPE, (char *) &i);
```

`risnull`

Prüft, ob eine Variable `NULL` ist.

```c
int risnull(int t, char *ptr);
```

Die Funktion erhält den Typ der zu prüfenden Variable (`t`) sowie einen Zeiger auf diese Variable (`ptr`). Letzterer muss in `char *` gecastet werden. Eine Liste möglicher Variablentypen finden Sie bei `rsetnull`.

Beispiel:

```c
$char c[] = "abc                     ";
$short s = 17;
$int i = -74874;

risnull(CCHARTYPE, (char *) c);
risnull(CSHORTTYPE, (char *) &s);
risnull(CINTTYPE, (char *) &i);
```

### 34.15.5. Zusätzliche Konstanten

Beachten Sie, dass alle hier aufgeführten Konstanten Fehler beschreiben und alle als negative Werte definiert sind. In den Beschreibungen der einzelnen Konstanten finden Sie auch den Wert, den die Konstante in der aktuellen Implementierung darstellt. Auf diese Zahl sollten Sie sich jedoch nicht verlassen. Sie können sich aber darauf verlassen, dass alle Konstanten negative Werte darstellen.

`ECPG_INFORMIX_NUM_OVERFLOW`

Funktionen geben diesen Wert zurück, wenn bei einer Berechnung ein Überlauf aufgetreten ist. Intern ist er als `-1200` definiert, entsprechend der Informix-Definition.

`ECPG_INFORMIX_NUM_UNDERFLOW`

Funktionen geben diesen Wert zurück, wenn bei einer Berechnung ein Unterlauf aufgetreten ist. Intern ist er als `-1201` definiert.

`ECPG_INFORMIX_DIVIDE_ZERO`

Funktionen geben diesen Wert zurück, wenn versucht wurde, durch null zu dividieren. Intern ist er als `-1202` definiert.

`ECPG_INFORMIX_BAD_YEAR`

Funktionen geben diesen Wert zurück, wenn beim Parsen eines Datums ein ungültiger Jahreswert gefunden wurde. Intern ist er als `-1204` definiert.

`ECPG_INFORMIX_BAD_MONTH`

Funktionen geben diesen Wert zurück, wenn beim Parsen eines Datums ein ungültiger Monatswert gefunden wurde. Intern ist er als `-1205` definiert.

`ECPG_INFORMIX_BAD_DAY`

Funktionen geben diesen Wert zurück, wenn beim Parsen eines Datums ein ungültiger Tageswert gefunden wurde. Intern ist er als `-1206` definiert.

`ECPG_INFORMIX_ENOSHORTDATE`

Funktionen geben diesen Wert zurück, wenn eine Parser-Routine eine kurze Datumsdarstellung benötigt, aber die Datumszeichenkette nicht in der richtigen Länge erhalten hat. Intern ist er als `-1209` definiert.

`ECPG_INFORMIX_DATE_CONVERT`

Funktionen geben diesen Wert zurück, wenn bei der Datumsformatierung ein Fehler aufgetreten ist. Intern ist er als `-1210` definiert.

`ECPG_INFORMIX_OUT_OF_MEMORY`

Funktionen geben diesen Wert zurück, wenn während ihrer Operation der Speicher erschöpft war. Intern ist er als `-1211` definiert.

`ECPG_INFORMIX_ENOTDMY`

Funktionen geben diesen Wert zurück, wenn eine Parser-Routine eine Formatmaske wie `mmddyy` erhalten sollte, aber nicht alle Felder korrekt aufgeführt waren. Intern ist er als `-1212` definiert.

`ECPG_INFORMIX_BAD_NUMERIC`

Funktionen geben diesen Wert zurück, wenn eine Parser-Routine die Textdarstellung eines numerischen Werts wegen Fehlern nicht parsen kann oder wenn eine Routine eine Berechnung mit numerischen Variablen nicht abschließen kann, weil mindestens eine der numerischen Variablen ungültig ist. Intern ist er als `-1213` definiert.

`ECPG_INFORMIX_BAD_EXPONENT`

Funktionen geben diesen Wert zurück, wenn eine Parser-Routine einen Exponenten nicht parsen kann. Intern ist er als `-1216` definiert.

`ECPG_INFORMIX_BAD_DATE`

Funktionen geben diesen Wert zurück, wenn eine Parser-Routine ein Datum nicht parsen kann. Intern ist er als `-1218` definiert.

`ECPG_INFORMIX_EXTRA_CHARS`

Funktionen geben diesen Wert zurück, wenn einer Parser-Routine zusätzliche Zeichen übergeben werden, die sie nicht parsen kann. Intern ist er als `-1264` definiert.

## 34.16. Oracle-Kompatibilitätsmodus

`ecpg` kann in einem sogenannten Oracle-Kompatibilitätsmodus ausgeführt werden. Wenn dieser Modus aktiv ist, versucht es, sich wie Oracle Pro*C zu verhalten.

Konkret verändert dieser Modus `ecpg` auf drei Arten:

- Zeichenarrays, die Zeichenkettentypen aufnehmen, werden bis zur angegebenen Länge mit abschließenden Leerzeichen aufgefüllt.
- Diese Zeichenarrays werden mit einem Nullbyte abgeschlossen, und die Indikatorvariable wird gesetzt, wenn eine Abschneidung auftritt.
- Der Nullindikator wird auf -1 gesetzt, wenn Zeichenarrays leere Zeichenkettentypen aufnehmen.

## 34.17. Interna

Dieser Abschnitt erklärt, wie ECPG intern arbeitet. Diese Informationen können gelegentlich nützlich sein, um Benutzern zu helfen, die Verwendung von ECPG zu verstehen.

Die ersten vier Zeilen, die `ecpg` in die Ausgabe schreibt, sind feste Zeilen. Zwei davon sind Kommentare, zwei sind Include-Zeilen, die für die Schnittstelle zur Bibliothek nötig sind. Danach liest der Präprozessor die Datei und schreibt Ausgabe. Normalerweise gibt er einfach alles unverändert in die Ausgabe weiter.

Wenn er eine `EXEC SQL`-Anweisung sieht, greift er ein und ändert sie. Der Befehl beginnt mit `EXEC SQL` und endet mit `;`. Alles dazwischen wird als SQL-Anweisung behandelt und für Variablensubstitution geparst.

Variablensubstitution tritt auf, wenn ein Symbol mit einem Doppelpunkt (`:`) beginnt. Die Variable mit diesem Namen wird unter den Variablen gesucht, die zuvor in einem Abschnitt `EXEC SQL DECLARE` deklariert wurden.

Die wichtigste Funktion in der Bibliothek ist `ECPGdo`; sie ist für die Ausführung der meisten Befehle zuständig. Sie nimmt eine variable Anzahl von Argumenten entgegen. Das kann leicht auf 50 oder mehr Argumente anwachsen, und es wird angenommen, dass dies auf keiner Plattform ein Problem darstellt.

Die Argumente sind:

`Eine Zeilennummer`

Dies ist die Zeilennummer der ursprünglichen Zeile; sie wird nur in Fehlermeldungen verwendet.

`Eine Zeichenkette`

Dies ist der auszugebende SQL-Befehl. Er wird durch die Eingabevariablen verändert, also durch die Variablen, die zur Kompilierzeit nicht bekannt waren, aber in den Befehl eingesetzt werden sollen. An den Stellen, an denen die Variablen stehen sollen, enthält die Zeichenkette `?`.

`Eingabevariablen`

Jede Eingabevariable erzeugt zehn Argumente, siehe unten.

`ECPGt_EOIT`

Ein Enum-Wert, der angibt, dass keine weiteren Eingabevariablen vorhanden sind.

`Ausgabevariablen`

Jede Ausgabevariable erzeugt zehn Argumente, siehe unten. Diese Variablen werden von der Funktion gefüllt.

`ECPGt_EORT`

Ein Enum-Wert, der angibt, dass keine weiteren Variablen vorhanden sind.

Für jede Variable, die Teil des SQL-Befehls ist, erhält die Funktion zehn Argumente:

1. Den Typ als besonderes Symbol.
2. Einen Zeiger auf den Wert oder einen Zeiger auf den Zeiger.
3. Die Größe der Variable, falls sie ein `char` oder `varchar` ist.
4. Die Anzahl der Elemente im Array, bei Array-Fetches.
5. Den Offset zum nächsten Element im Array, bei Array-Fetches.
6. Den Typ der Indikatorvariable als besonderes Symbol.
7. Einen Zeiger auf die Indikatorvariable.
8. `0`.
9. Die Anzahl der Elemente im Indikatorarray, bei Array-Fetches.
10. Den Offset zum nächsten Element im Indikatorarray, bei Array-Fetches.

Beachten Sie, dass nicht alle SQL-Befehle auf diese Weise behandelt werden. Eine Open-Cursor-Anweisung wie:

```sql
EXEC SQL OPEN cursor;
```

wird zum Beispiel nicht in die Ausgabe kopiert. Stattdessen wird der `DECLARE`-Befehl des Cursors an der Stelle des `OPEN`-Befehls verwendet, weil er den Cursor tatsächlich öffnet.

Hier ist ein vollständiges Beispiel, das die Ausgabe des Präprozessors für eine Datei `foo.pgc` beschreibt. Details können sich je nach konkreter Version des Präprozessors ändern:

```c
EXEC SQL BEGIN DECLARE SECTION;
int index;
int result;
EXEC SQL END DECLARE SECTION;
...
EXEC SQL SELECT res INTO :result FROM mytable WHERE index = :index;
```

wird übersetzt in:

```c
/* Processed by ecpg (2.6.0) */
/* These two include files are added by the preprocessor */
#include <ecpgtype.h>;
#include <ecpglib.h>;

/* exec sql begin declare section */

#line 1 "foo.pgc"

 int index;
 int result;
/* exec sql end declare section */
...
ECPGdo(__LINE__, NULL, "SELECT res FROM mytable WHERE index = ?
 ",
        ECPGt_int,&(index),1L,1L,sizeof(int),
        ECPGt_NO_INDICATOR, NULL , 0L, 0L, 0L, ECPGt_EOIT,
        ECPGt_int,&(result),1L,1L,sizeof(int),
        ECPGt_NO_INDICATOR, NULL , 0L, 0L, 0L, ECPGt_EORT);
#line 147 "foo.pgc"
```

Die Einrückung wurde hier zur besseren Lesbarkeit hinzugefügt und ist nicht etwas, das der Präprozessor selbst erzeugt.
