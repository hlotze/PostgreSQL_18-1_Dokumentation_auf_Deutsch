# 39. Das Regelsystem

Dieses Kapitel behandelt das Regelsystem in PostgreSQL. Produktionsregelsysteme sind konzeptionell einfach, aber ihre tatsächliche Verwendung enthält viele feine Details.

Einige andere Datenbanksysteme definieren aktive Datenbankregeln, die üblicherweise als Stored Procedures und Trigger umgesetzt sind. In PostgreSQL lassen sich solche Mechanismen ebenfalls mit Funktionen und Triggern implementieren.

Das Regelsystem, genauer gesagt das Query-Rewrite-Regelsystem, unterscheidet sich jedoch völlig von Stored Procedures und Triggern. Es verändert Abfragen so, dass Regeln berücksichtigt werden, und übergibt die veränderte Abfrage danach an den Query Planner zur Planung und Ausführung. Es ist sehr leistungsfähig und kann für viele Zwecke verwendet werden, etwa für Query-Language-Prozeduren, Views und Versionierung. Die theoretischen Grundlagen und die Leistungsfähigkeit dieses Regelsystems werden auch in [ston90b] und [ong90] diskutiert.

## 39.1. Der Abfragebaum

Um zu verstehen, wie das Regelsystem arbeitet, muss man wissen, wann es aufgerufen wird und was seine Eingaben und Ergebnisse sind.

Das Regelsystem liegt zwischen Parser und Planner. Es nimmt die Ausgabe des Parsers, also einen Abfragebaum, sowie die benutzerdefinierten Rewrite-Regeln, die ebenfalls Abfragebäume mit einigen Zusatzinformationen sind, und erzeugt daraus null oder mehr Abfragebäume als Ergebnis. Eingabe und Ausgabe sind also immer Dinge, die der Parser selbst hätte erzeugen können. Alles, was das Regelsystem sieht, lässt sich im Grundsatz als SQL-Anweisung darstellen.

Was ist nun ein Abfragebaum? Er ist eine interne Repräsentation einer SQL-Anweisung, in der die einzelnen Bestandteile der Anweisung getrennt gespeichert werden. Diese Abfragebäume können im Server-Log angezeigt werden, wenn die Konfigurationsparameter `debug_print_parse`, `debug_print_rewritten` oder `debug_print_plan` gesetzt sind. Auch die Regelaktionen werden als Abfragebäume im Systemkatalog `pg_rewrite` gespeichert. Sie sind dort nicht wie die Log-Ausgabe formatiert, enthalten aber exakt dieselben Informationen.

Das Lesen eines rohen Abfragebaums erfordert etwas Erfahrung. Da SQL-Darstellungen von Abfragebäumen aber ausreichen, um das Regelsystem zu verstehen, erklärt dieses Kapitel nicht, wie man rohe Abfragebäume liest.

Beim Lesen der SQL-Darstellungen der Abfragebäume in diesem Kapitel muss man erkennen können, in welche Bestandteile eine Anweisung in der Abfragebaumstruktur zerlegt wird. Die Bestandteile eines Abfragebaums sind:

- Der Befehlstyp
  - Dies ist ein einfacher Wert, der angibt, welcher Befehl (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) den Abfragebaum erzeugt hat.

- Die Range Table
  - Die Range Table ist eine Liste der Relationen, die in der Abfrage verwendet werden. In einer `SELECT`-Anweisung sind das die Relationen, die nach dem Schlüsselwort `FROM` angegeben sind.

  - Jeder Range-Table-Eintrag identifiziert eine Tabelle oder View und gibt an, unter welchem Namen sie in den anderen Teilen der Abfrage angesprochen wird. Im Abfragebaum werden Range-Table-Einträge über Nummern statt über Namen referenziert. Deshalb spielt es hier keine Rolle, ob doppelte Namen vorkommen, anders als in einer SQL-Anweisung. Das kann passieren, nachdem die Range Tables von Regeln zusammengeführt wurden. Die Beispiele in diesem Kapitel enthalten diese Situation nicht.

- Die Ergebnisrelation
  - Dies ist ein Index in die Range Table, der die Relation identifiziert, in die die Ergebnisse der Abfrage gehen.

  - `SELECT`-Abfragen haben keine Ergebnisrelation. Der Sonderfall `SELECT INTO` entspricht im Wesentlichen `CREATE TABLE` gefolgt von `INSERT ... SELECT` und wird hier nicht gesondert behandelt.

  - Bei `INSERT`-, `UPDATE`- und `DELETE`-Befehlen ist die Ergebnisrelation die Tabelle oder View, auf die sich die Änderungen auswirken sollen.

- Die Zielliste
  - Die Zielliste ist eine Liste von Ausdrücken, die das Ergebnis der Abfrage definieren. Bei einem `SELECT` sind dies die Ausdrücke, die die endgültige Ausgabe der Abfrage bilden. Sie entsprechen den Ausdrücken zwischen den Schlüsselwörtern `SELECT` und `FROM`. `*` ist nur eine Abkürzung für alle Spaltennamen einer Relation. Der Parser erweitert sie zu den einzelnen Spalten, sodass das Regelsystem `*` nie sieht.

  - `DELETE`-Befehle benötigen keine normale Zielliste, weil sie kein Ergebnis erzeugen. Stattdessen fügt der Planner der leeren Zielliste einen speziellen `CTID`-Eintrag hinzu, damit der Executor die zu löschende Zeile finden kann. `CTID` wird hinzugefügt, wenn die Ergebnisrelation eine gewöhnliche Tabelle ist. Wenn sie eine View ist, fügt das Regelsystem stattdessen eine Whole-Row-Variable hinzu, wie in [Abschnitt 39.2.4](#3924-eine-view-aktualisieren) beschrieben.

  - Bei `INSERT`-Befehlen beschreibt die Zielliste die neuen Zeilen, die in die Ergebnisrelation eingefügt werden sollen. Sie besteht aus den Ausdrücken in der `VALUES`-Klausel oder den Ausdrücken aus der `SELECT`-Klausel von `INSERT ... SELECT`. Der erste Schritt des Rewrite-Prozesses fügt Ziellisteneinträge für alle Spalten hinzu, denen der ursprüngliche Befehl keinen Wert zugewiesen hat, die aber Standardwerte besitzen. Alle verbleibenden Spalten, also solche ohne angegebenen Wert und ohne Standardwert, werden vom Planner mit einem konstanten Null-Ausdruck gefüllt.

  - Bei `UPDATE`-Befehlen beschreibt die Zielliste die neuen Zeilen, die die alten ersetzen sollen. Im Regelsystem enthält sie nur die Ausdrücke aus dem Teil `SET column = expression` des Befehls. Fehlende Spalten behandelt der Planner, indem er Ausdrücke einfügt, die die Werte aus der alten Zeile in die neue kopieren. Wie bei `DELETE` wird eine `CTID`- oder Whole-Row-Variable hinzugefügt, damit der Executor die alte, zu aktualisierende Zeile identifizieren kann.

  - Jeder Eintrag in der Zielliste enthält einen Ausdruck. Dieser kann ein konstanter Wert sein, eine Variable, die auf eine Spalte einer der Relationen in der Range Table zeigt, ein Parameter oder ein Ausdrucksbaum aus Funktionsaufrufen, Konstanten, Variablen, Operatoren usw.

- Die Qualifikation
  - Die Qualifikation der Abfrage ist ein Ausdruck, ähnlich den Ausdrücken in den Einträgen der Zielliste. Das Ergebnis dieses Ausdrucks ist ein boolescher Wert, der angibt, ob die Operation (`INSERT`, `UPDATE`, `DELETE` oder `SELECT`) für die endgültige Ergebniszeile ausgeführt werden soll oder nicht. Sie entspricht der `WHERE`-Klausel einer SQL-Anweisung.

- Der Join-Baum
  - Der Join-Baum der Abfrage zeigt die Struktur der `FROM`-Klausel. Bei einer einfachen Abfrage wie `SELECT ... FROM a, b, c` ist der Join-Baum nur eine Liste der `FROM`-Elemente, weil diese in beliebiger Reihenfolge gejoint werden dürfen. Werden jedoch `JOIN`-Ausdrücke verwendet, insbesondere Outer Joins, muss in der von den Joins vorgegebenen Reihenfolge gejoint werden. In diesem Fall zeigt der Join-Baum die Struktur der `JOIN`-Ausdrücke. Die Einschränkungen bestimmter `JOIN`-Klauseln aus `ON`- oder `USING`-Ausdrücken werden als Qualifikationsausdrücke an die jeweiligen Join-Baumknoten gehängt. Es erweist sich außerdem als praktisch, den obersten `WHERE`-Ausdruck ebenfalls als Qualifikation am obersten Join-Baumelement zu speichern. Tatsächlich repräsentiert der Join-Baum also sowohl die `FROM`- als auch die `WHERE`-Klausel eines `SELECT`.

- Die übrigen Teile
  - Andere Teile des Abfragebaums, etwa die `ORDER BY`-Klausel, sind hier nicht von Interesse. Das Regelsystem ersetzt dort beim Anwenden von Regeln zwar einige Einträge, aber das hat wenig mit den Grundlagen des Regelsystems zu tun.

## 39.2. Views und das Regelsystem

Views werden in PostgreSQL mithilfe des Regelsystems implementiert. Eine View ist im Grunde eine leere Tabelle ohne tatsächlichen Speicher, die eine `ON SELECT DO INSTEAD`-Regel besitzt. Üblicherweise heißt diese Regel `_RETURN`. Eine View wie:

```sql
CREATE VIEW myview AS SELECT * FROM mytab;
```

ist daher fast dasselbe wie:

```sql
CREATE TABLE myview (same column list as mytab);
CREATE RULE "_RETURN" AS ON SELECT TO myview DO INSTEAD
    SELECT * FROM mytab;
```

Tatsächlich kann man dies aber nicht genau so schreiben, weil Tabellen keine `ON SELECT`-Regeln haben dürfen.

Eine View kann auch andere Arten von `DO INSTEAD`-Regeln besitzen. Dadurch können `INSERT`-, `UPDATE`- oder `DELETE`-Befehle auf der View ausgeführt werden, obwohl sie keinen zugrunde liegenden Speicher hat. Dies wird weiter unten in [Abschnitt 39.2.4](#3924-eine-view-aktualisieren) behandelt.

### 39.2.1. Wie SELECT-Regeln funktionieren

`ON SELECT`-Regeln werden als letzter Schritt auf alle Abfragen angewendet, auch wenn der angegebene Befehl ein `INSERT`, `UPDATE` oder `DELETE` ist. Sie haben eine andere Semantik als Regeln auf anderen Befehlstypen, weil sie den Abfragebaum an Ort und Stelle verändern, statt einen neuen zu erzeugen. Deshalb werden `SELECT`-Regeln zuerst beschrieben.

Derzeit darf es in einer `ON SELECT`-Regel nur eine Aktion geben, und diese muss eine unbedingte `SELECT`-Aktion mit `INSTEAD` sein. Diese Einschränkung war erforderlich, um Regeln sicher genug zu machen, damit normale Benutzer sie verwenden können; sie beschränkt `ON SELECT`-Regeln darauf, sich wie Views zu verhalten.

Die Beispiele dieses Kapitels verwenden zwei Join-Views, die Berechnungen durchführen, sowie weitere Views, die wiederum diese Views verwenden. Eine der beiden ersten Views wird später durch Regeln für `INSERT`-, `UPDATE`- und `DELETE`-Operationen angepasst, sodass das Endergebnis eine View ist, die sich wie eine echte Tabelle mit etwas zusätzlicher Funktionalität verhält. Das ist kein ganz einfaches Einstiegsbeispiel und macht den Anfang etwas schwieriger. Es ist aber besser, ein einziges Beispiel zu haben, das alle Punkte Schritt für Schritt abdeckt, als viele verschiedene Beispiele, die man leicht durcheinanderbringen kann.

Die echten Tabellen, die wir für die ersten beiden Beschreibungen des Regelsystems benötigen, sind:

```sql
CREATE TABLE shoe_data (
    shoename   text,                           -- primary key
    sh_avail   integer,                        -- available number of pairs
    slcolor    text,                           -- preferred shoelace color
    slminlen   real,                           -- minimum shoelace length
    slmaxlen   real,                           -- maximum shoelace length
    slunit     text                            -- length unit
);

CREATE TABLE shoelace_data (
    sl_name    text,                           -- primary key
    sl_avail   integer,                        -- available number of pairs
    sl_color   text,                           -- shoelace color
    sl_len     real,                           -- shoelace length
    sl_unit    text                            -- length unit
);

CREATE TABLE unit (
    un_name    text,                          -- primary key
    un_fact    real                           -- factor to transform to cm
);
```

Wie man sieht, stellen sie Daten eines Schuhgeschäfts dar.

Die Views werden so erstellt:

```sql
CREATE VIEW shoe AS
    SELECT sh.shoename,
           sh.sh_avail,
           sh.slcolor,
           sh.slminlen,
           sh.slminlen * un.un_fact AS slminlen_cm,
           sh.slmaxlen,
           sh.slmaxlen * un.un_fact AS slmaxlen_cm,
           sh.slunit
      FROM shoe_data sh, unit un
     WHERE sh.slunit = un.un_name;

CREATE VIEW shoelace AS
    SELECT s.sl_name,
           s.sl_avail,
           s.sl_color,
           s.sl_len,
           s.sl_unit,
           s.sl_len * u.un_fact AS sl_len_cm
      FROM shoelace_data s, unit u
     WHERE s.sl_unit = u.un_name;

CREATE VIEW shoe_ready AS
    SELECT rsh.shoename,
           rsh.sh_avail,
           rsl.sl_name,
           rsl.sl_avail,
           least(rsh.sh_avail, rsl.sl_avail) AS total_avail
      FROM shoe rsh, shoelace rsl
     WHERE rsl.sl_color = rsh.slcolor
       AND rsl.sl_len_cm >= rsh.slminlen_cm
       AND rsl.sl_len_cm <= rsh.slmaxlen_cm;
```

Der Befehl `CREATE VIEW` für die View `shoelace`, die die einfachste dieser Views ist, erzeugt eine Relation `shoelace` und einen Eintrag in `pg_rewrite`. Dieser Eintrag besagt, dass eine Rewrite-Regel angewendet werden muss, wann immer die Relation `shoelace` in der Range Table einer Abfrage referenziert wird. Die Regel besitzt keine Regelqualifikation; diese wird später bei den Nicht-`SELECT`-Regeln behandelt, da `SELECT`-Regeln derzeit keine haben können. Sie ist außerdem `INSTEAD`. Beachten Sie, dass Regelqualifikationen nicht dasselbe sind wie Abfragequalifikationen. Die Aktion unserer Regel hat eine Abfragequalifikation. Die Aktion der Regel ist ein Abfragebaum, der eine Kopie der `SELECT`-Anweisung aus dem View-Erstellungsbefehl ist.

> Hinweis: Die beiden zusätzlichen Range-Table-Einträge für `NEW` und `OLD`, die man im `pg_rewrite`-Eintrag sehen kann, sind für `SELECT`-Regeln nicht von Interesse.

Nun füllen wir `unit`, `shoe_data` und `shoelace_data` und führen eine einfache Abfrage auf einer View aus:

```sql
INSERT INTO unit VALUES ('cm', 1.0);
INSERT INTO unit VALUES ('m', 100.0);
INSERT INTO unit VALUES ('inch', 2.54);

INSERT INTO shoe_data VALUES ('sh1', 2, 'black', 70.0, 90.0, 'cm');
INSERT INTO shoe_data VALUES ('sh2', 0, 'black', 30.0, 40.0,
 'inch');
INSERT INTO shoe_data VALUES ('sh3', 4, 'brown', 50.0, 65.0, 'cm');
INSERT INTO shoe_data VALUES ('sh4', 3, 'brown', 40.0, 50.0,
 'inch');

INSERT INTO shoelace_data VALUES ('sl1', 5, 'black', 80.0, 'cm');
INSERT INTO shoelace_data VALUES ('sl2', 6, 'black', 100.0, 'cm');
INSERT INTO shoelace_data VALUES ('sl3', 0, 'black', 35.0 ,
 'inch');
INSERT INTO shoelace_data VALUES ('sl4', 8, 'black', 40.0 ,
 'inch');
INSERT INTO shoelace_data VALUES ('sl5', 4, 'brown', 1.0 , 'm');
INSERT INTO shoelace_data VALUES ('sl6', 0, 'brown', 0.9 , 'm');
INSERT INTO shoelace_data VALUES ('sl7', 7, 'brown', 60 , 'cm');
INSERT INTO shoelace_data VALUES ('sl8', 1, 'brown', 40 , 'inch');

SELECT * FROM shoelace;
```

```text
 sl_name   | sl_avail | sl_color | sl_len | sl_unit | sl_len_cm
-----------+----------+----------+--------+---------+-----------
 sl1       |        5 | black    |     80 | cm      |        80
 sl2       |        6 | black    |    100 | cm      |       100
 sl7       |        7 | brown    |     60 | cm      |        60
 sl3       |        0 | black    |     35 | inch    |      88.9
 sl4       |        8 | black    |     40 | inch    |     101.6
 sl8       |        1 | brown    |     40 | inch    |     101.6
 sl5       |        4 | brown    |      1 | m       |       100
 sl6       |        0 | brown    |    0.9 | m       |        90
(8 rows)
```

Dies ist das einfachste `SELECT`, das auf unseren Views möglich ist. Deshalb nutzen wir es, um die Grundlagen von View-Regeln zu erklären. `SELECT * FROM shoelace` wurde vom Parser interpretiert und erzeugte diesen Abfragebaum:

```sql
SELECT shoelace.sl_name, shoelace.sl_avail,
       shoelace.sl_color, shoelace.sl_len,
       shoelace.sl_unit, shoelace.sl_len_cm
  FROM shoelace shoelace;
```

Dieser wird an das Regelsystem übergeben. Das Regelsystem durchläuft die Range Table und prüft, ob es für irgendeine Relation Regeln gibt. Beim Verarbeiten des Range-Table-Eintrags für `shoelace`, bisher des einzigen Eintrags, findet es die Regel `_RETURN` mit diesem Abfragebaum:

```sql
SELECT s.sl_name, s.sl_avail,
       s.sl_color, s.sl_len, s.sl_unit,
       s.sl_len * u.un_fact AS sl_len_cm
  FROM shoelace old, shoelace new,
       shoelace_data s, unit u
 WHERE s.sl_unit = u.un_name;
```

Um die View zu expandieren, erzeugt der Rewriter einfach einen Subquery-Range-Table-Eintrag, der den Aktionsabfragebaum der Regel enthält, und ersetzt den ursprünglichen Range-Table-Eintrag, der die View referenzierte, durch diesen Eintrag. Der resultierende umgeschriebene Abfragebaum ist fast derselbe, als hätte man Folgendes geschrieben:

```sql
SELECT shoelace.sl_name, shoelace.sl_avail,
       shoelace.sl_color, shoelace.sl_len,
       shoelace.sl_unit, shoelace.sl_len_cm
  FROM (SELECT s.sl_name,
               s.sl_avail,
               s.sl_color,
               s.sl_len,
               s.sl_unit,
               s.sl_len * u.un_fact AS sl_len_cm
          FROM shoelace_data s, unit u
         WHERE s.sl_unit = u.un_name) shoelace;
```

Es gibt jedoch einen Unterschied: Die Range Table der Unterabfrage enthält zwei zusätzliche Einträge, `shoelace old` und `shoelace new`. Diese Einträge nehmen nicht direkt an der Abfrage teil, weil weder der Join-Baum noch die Zielliste der Unterabfrage auf sie verweisen. Der Rewriter nutzt sie, um die Informationen für die Zugriffsrechteprüfung zu speichern, die ursprünglich im Range-Table-Eintrag vorhanden waren, der auf die View verwies. Auf diese Weise prüft der Executor weiterhin, ob der Benutzer die erforderlichen Rechte für den Zugriff auf die View hat, obwohl die View in der umgeschriebenen Abfrage nicht mehr direkt verwendet wird.

Das war die erste angewendete Regel. Das Regelsystem prüft anschließend die übrigen Range-Table-Einträge der obersten Abfrage, in diesem Beispiel gibt es keine weiteren, und prüft rekursiv die Range-Table-Einträge der hinzugefügten Unterabfrage, um zu sehen, ob sie Views referenzieren. `old` und `new` werden dabei aber nicht expandiert, sonst entstünde eine unendliche Rekursion. In diesem Beispiel gibt es keine Rewrite-Regeln für `shoelace_data` oder `unit`, sodass das Rewrite abgeschlossen ist und das oben gezeigte Ergebnis an den Planner übergeben wird.

Nun möchten wir eine Abfrage schreiben, die herausfindet, für welche Schuhe im Geschäft passende Schnürsenkel vorhanden sind, also passende Farbe und Länge, und bei denen die Gesamtzahl exakt passender Paare größer oder gleich zwei ist.

```sql
SELECT * FROM shoe_ready WHERE total_avail >= 2;
```

```text
 shoename | sh_avail | sl_name | sl_avail | total_avail
----------+----------+---------+----------+-------------
 sh1      |        2 | sl1     |        5 |           2
 sh3      |        4 | sl7     |        7 |           4
(2 rows)
```

Die Ausgabe des Parsers ist diesmal dieser Abfragebaum:

```sql
SELECT shoe_ready.shoename, shoe_ready.sh_avail,
       shoe_ready.sl_name, shoe_ready.sl_avail,
       shoe_ready.total_avail
  FROM shoe_ready shoe_ready
 WHERE shoe_ready.total_avail >= 2;
```

Die erste angewendete Regel ist die für die View `shoe_ready`. Sie führt zu diesem Abfragebaum:

```sql
SELECT shoe_ready.shoename, shoe_ready.sh_avail,
       shoe_ready.sl_name, shoe_ready.sl_avail,
       shoe_ready.total_avail
  FROM (SELECT rsh.shoename,
               rsh.sh_avail,
               rsl.sl_name,
               rsl.sl_avail,
               least(rsh.sh_avail, rsl.sl_avail) AS total_avail
          FROM shoe rsh, shoelace rsl
         WHERE rsl.sl_color = rsh.slcolor
           AND rsl.sl_len_cm >= rsh.slminlen_cm
           AND rsl.sl_len_cm <= rsh.slmaxlen_cm) shoe_ready
 WHERE shoe_ready.total_avail >= 2;
```

Ebenso werden die Regeln für `shoe` und `shoelace` in die Range Table der Unterabfrage eingesetzt. Dadurch entsteht ein endgültiger, dreistufiger Abfragebaum:

```sql
SELECT shoe_ready.shoename, shoe_ready.sh_avail,
       shoe_ready.sl_name, shoe_ready.sl_avail,
       shoe_ready.total_avail
  FROM (SELECT rsh.shoename,
               rsh.sh_avail,
               rsl.sl_name,
               rsl.sl_avail,
               least(rsh.sh_avail, rsl.sl_avail) AS total_avail
          FROM (SELECT sh.shoename,
                       sh.sh_avail,
                       sh.slcolor,
                       sh.slminlen,
                       sh.slminlen * un.un_fact AS slminlen_cm,
                       sh.slmaxlen,
                       sh.slmaxlen * un.un_fact AS slmaxlen_cm,
                       sh.slunit
                  FROM shoe_data sh, unit un
                 WHERE sh.slunit = un.un_name) rsh,
               (SELECT s.sl_name,
                       s.sl_avail,
                       s.sl_color,
                       s.sl_len,
                       s.sl_unit,
                       s.sl_len * u.un_fact AS sl_len_cm
                  FROM shoelace_data s, unit u
                 WHERE s.sl_unit = u.un_name) rsl
         WHERE rsl.sl_color = rsh.slcolor
           AND rsl.sl_len_cm >= rsh.slminlen_cm
           AND rsl.sl_len_cm <= rsh.slmaxlen_cm) shoe_ready
 WHERE shoe_ready.total_avail > 2;
```

Das mag ineffizient aussehen, aber der Planner reduziert dies wieder zu einem einstufigen Abfragebaum, indem er die Unterabfragen „hochzieht“, und plant die Joins dann genauso, als hätten wir sie von Hand ausgeschrieben. Das Zusammenfassen des Abfragebaums ist also eine Optimierung, um die sich das Rewrite-System nicht kümmern muss.

### 39.2.2. View-Regeln in Nicht-SELECT-Anweisungen

Zwei Details des Abfragebaums wurden in der obigen Beschreibung von View-Regeln noch nicht behandelt: der Befehlstyp und die Ergebnisrelation. Tatsächlich wird der Befehlstyp für View-Regeln nicht benötigt, aber die Ergebnisrelation kann beeinflussen, wie der Query Rewriter arbeitet, weil besondere Vorsicht nötig ist, wenn die Ergebnisrelation eine View ist.

Zwischen einem Abfragebaum für ein `SELECT` und einem für irgendeinen anderen Befehl gibt es nur wenige Unterschiede. Offensichtlich haben sie einen anderen Befehlstyp, und bei einem Befehl, der kein `SELECT` ist, zeigt die Ergebnisrelation auf den Range-Table-Eintrag, in den das Ergebnis gehen soll. Alles andere ist völlig gleich. Haben wir also zwei Tabellen `t1` und `t2` mit den Spalten `a` und `b`, dann sind die Abfragebäume für diese beiden Anweisungen fast identisch:

```sql
SELECT t2.b FROM t1, t2 WHERE t1.a = t2.a;

UPDATE t1 SET b = t2.b FROM t2 WHERE t1.a = t2.a;
```

Insbesondere gilt:

- Die Range Tables enthalten Einträge für die Tabellen `t1` und `t2`.
- Die Ziellisten enthalten eine Variable, die auf Spalte `b` des Range-Table-Eintrags für Tabelle `t2` zeigt.
- Die Qualifikationsausdrücke vergleichen die Spalten `a` beider Range-Table-Einträge auf Gleichheit.
- Die Join-Bäume zeigen einen einfachen Join zwischen `t1` und `t2`.

Die Folge ist, dass beide Abfragebäume zu ähnlichen Ausführungsplänen führen: Beide sind Joins über die zwei Tabellen. Beim `UPDATE` werden die fehlenden Spalten aus `t1` vom Planner zur Zielliste hinzugefügt, und der endgültige Abfragebaum liest sich dann als:

```sql
UPDATE t1 SET a = t1.a, b = t2.b FROM t2 WHERE t1.a = t2.a;
```

Der Executor-Lauf über den Join erzeugt damit genau dieselbe Ergebnismenge wie:

```sql
SELECT t1.a, t2.b FROM t1, t2 WHERE t1.a = t2.a;
```

Beim `UPDATE` gibt es jedoch ein kleines Problem: Der Teil des Executor-Plans, der den Join ausführt, kümmert sich nicht darum, wofür die Ergebnisse des Joins gedacht sind. Er erzeugt einfach eine Ergebnismenge von Zeilen. Dass der eine Befehl ein `SELECT` und der andere ein `UPDATE` ist, wird weiter oben im Executor behandelt. Dort ist bekannt, dass es sich um ein `UPDATE` handelt und dass das Ergebnis in Tabelle `t1` gehen soll. Aber welche der vorhandenen Zeilen muss durch die neue Zeile ersetzt werden?

Zur Lösung dieses Problems wird bei `UPDATE`- und auch bei `DELETE`-Anweisungen ein weiterer Eintrag zur Zielliste hinzugefügt: die aktuelle Tupel-ID (`CTID`). Dies ist eine Systemspalte, die die Dateiblocknummer und die Position im Block für die Zeile enthält. Kennt man die Tabelle, kann die `CTID` verwendet werden, um die ursprüngliche, zu aktualisierende Zeile aus `t1` wiederzufinden. Nach dem Hinzufügen der `CTID` zur Zielliste sieht die Abfrage im Grunde so aus:

```sql
SELECT t1.a, t2.b, t1.ctid FROM t1, t2 WHERE t1.a = t2.a;
```

Nun kommt ein weiteres Detail von PostgreSQL ins Spiel. Alte Tabellenzeilen werden nicht überschrieben, und deshalb ist `ROLLBACK` schnell. Bei einem `UPDATE` wird die neue Ergebniszeile in die Tabelle eingefügt, nachdem die `CTID` entfernt wurde. Im Zeilenkopf der alten Zeile, auf die die `CTID` zeigte, werden die Einträge `cmax` und `xmax` auf den aktuellen Befehlszähler und die aktuelle Transaktions-ID gesetzt. Dadurch wird die alte Zeile verborgen, und nachdem die Transaktion committet wurde, kann der Vacuum Cleaner die tote Zeile irgendwann entfernen.

Mit diesem Wissen können wir View-Regeln auf jeden Befehl in genau derselben Weise anwenden. Es gibt keinen Unterschied.

### 39.2.3. Die Leistungsfähigkeit von Views in PostgreSQL

Das Obige zeigt, wie das Regelsystem View-Definitionen in den ursprünglichen Abfragebaum einarbeitet. Im zweiten Beispiel erzeugte ein einfaches `SELECT` aus einer View einen endgültigen Abfragebaum, der ein Join aus vier Tabellen ist; `unit` wurde dabei zweimal unter verschiedenen Namen verwendet.

Der Vorteil der Implementierung von Views über das Regelsystem besteht darin, dass der Planner alle Informationen in einem einzigen Abfragebaum vorliegen hat: welche Tabellen gescannt werden müssen, welche Beziehungen zwischen diesen Tabellen bestehen, welche einschränkenden Qualifikationen aus den Views stammen und welche Qualifikationen aus der ursprünglichen Abfrage kommen. Das bleibt auch dann so, wenn die ursprüngliche Abfrage bereits ein Join über Views ist. Der Planner muss entscheiden, welcher Pfad für die Ausführung der Abfrage am besten ist, und je mehr Informationen er hat, desto besser kann diese Entscheidung ausfallen. Das in PostgreSQL implementierte Regelsystem stellt sicher, dass dem Planner bis zu diesem Punkt alle verfügbaren Informationen über die Abfrage vorliegen.

### 39.2.4. Eine View aktualisieren

Was passiert, wenn eine View als Zielrelation für ein `INSERT`, `UPDATE`, `DELETE` oder `MERGE` genannt wird? Würde man die oben beschriebenen Ersetzungen durchführen, entstünde ein Abfragebaum, in dem die Ergebnisrelation auf einen Subquery-Range-Table-Eintrag zeigt. Das funktioniert nicht. PostgreSQL kann den Eindruck einer aktualisierbaren View jedoch auf mehrere Arten unterstützen. Nach der vom Benutzer wahrgenommenen Komplexität geordnet sind das: die zugrunde liegende Tabelle automatisch anstelle der View einsetzen, einen benutzerdefinierten Trigger ausführen oder die Abfrage gemäß einer benutzerdefinierten Regel umschreiben. Diese Optionen werden im Folgenden behandelt.

Wenn die Unterabfrage aus einer einzelnen Basisrelation auswählt und einfach genug ist, kann der Rewriter die Unterabfrage automatisch durch die zugrunde liegende Basisrelation ersetzen, sodass `INSERT`, `UPDATE`, `DELETE` oder `MERGE` in geeigneter Weise auf die Basisrelation angewendet wird. Views, die dafür „einfach genug“ sind, heißen automatisch aktualisierbar. Ausführliche Informationen dazu, welche Arten von Views automatisch aktualisiert werden können, finden sich bei `CREATE VIEW`.

Alternativ kann die Operation durch einen vom Benutzer bereitgestellten `INSTEAD OF`-Trigger auf der View behandelt werden; siehe `CREATE TRIGGER`. Das Rewrite funktioniert in diesem Fall etwas anders. Bei `INSERT` unternimmt der Rewriter mit der View gar nichts und lässt sie als Ergebnisrelation der Abfrage stehen. Bei `UPDATE`, `DELETE` und `MERGE` muss die View-Abfrage weiterhin expandiert werden, um die „alten“ Zeilen zu erzeugen, die der Befehl zu aktualisieren, zu löschen oder zusammenzuführen versucht. Die View wird also normal expandiert, aber der Abfrage wird ein weiterer, nicht expandierter Range-Table-Eintrag hinzugefügt, der die View in ihrer Rolle als Ergebnisrelation repräsentiert.

Das Problem ist nun, wie die zu aktualisierenden Zeilen in der View identifiziert werden. Wenn die Ergebnisrelation eine Tabelle ist, wird, wie oben beschrieben, ein spezieller `CTID`-Eintrag zur Zielliste hinzugefügt, um die physischen Speicherorte der zu aktualisierenden Zeilen zu identifizieren. Bei einer View funktioniert das nicht, weil eine View keine `CTID` besitzt; ihre Zeilen haben keine tatsächlichen physischen Speicherorte. Stattdessen wird bei einer `UPDATE`-, `DELETE`- oder `MERGE`-Operation ein spezieller Whole-Row-Eintrag zur Zielliste hinzugefügt, der alle Spalten der View umfasst. Der Executor verwendet diesen Wert, um dem `INSTEAD OF`-Trigger die „alte“ Zeile bereitzustellen. Der Trigger muss dann anhand der alten und neuen Zeilenwerte entscheiden, was aktualisiert werden soll.

Eine weitere Möglichkeit besteht darin, dass der Benutzer `INSTEAD`-Regeln definiert, die Ersatzaktionen für `INSERT`-, `UPDATE`- und `DELETE`-Befehle auf einer View angeben. Diese Regeln schreiben den Befehl um, typischerweise in einen Befehl, der eine oder mehrere Tabellen statt Views aktualisiert. Das ist Thema von [Abschnitt 39.4](#394-regeln-für-insert-update-und-delete). Beachten Sie, dass dies mit `MERGE` nicht funktioniert, da `MERGE` derzeit keine Regeln auf der Zielrelation unterstützt, außer `SELECT`-Regeln.

Regeln werden zuerst ausgewertet, indem sie die ursprüngliche Abfrage umschreiben, bevor diese geplant und ausgeführt wird. Wenn eine View also sowohl `INSTEAD OF`-Trigger als auch Regeln für `INSERT`, `UPDATE` oder `DELETE` besitzt, werden zuerst die Regeln ausgewertet. Je nach Ergebnis werden die Trigger dann möglicherweise gar nicht verwendet.

Das automatische Umschreiben einer `INSERT`-, `UPDATE`-, `DELETE`- oder `MERGE`-Abfrage auf einer einfachen View wird immer zuletzt versucht. Wenn eine View also Regeln oder Trigger besitzt, überschreiben diese das Standardverhalten automatisch aktualisierbarer Views.

Gibt es keine `INSTEAD`-Regeln oder `INSTEAD OF`-Trigger für die View und kann der Rewriter die Abfrage nicht automatisch als Aktualisierung der zugrunde liegenden Basisrelation umschreiben, wird ein Fehler ausgelöst, weil der Executor eine View als solche nicht aktualisieren kann.

## 39.3. Materialisierte Views

Materialisierte Views in PostgreSQL verwenden das Regelsystem ähnlich wie Views, speichern die Ergebnisse aber in einer tabellenähnlichen Form dauerhaft. Die wichtigsten Unterschiede zwischen:

```sql
CREATE MATERIALIZED VIEW mymatview AS SELECT * FROM mytab;
```

und:

```sql
CREATE TABLE mymatview AS SELECT * FROM mytab;
```

bestehen darin, dass die materialisierte View später nicht direkt aktualisiert werden kann und dass die Abfrage, mit der die materialisierte View erzeugt wurde, genau so gespeichert wird wie die Abfrage einer View. Dadurch können mit folgendem Befehl frische Daten für die materialisierte View erzeugt werden:

```sql
REFRESH MATERIALIZED VIEW mymatview;
```

Die Informationen über eine materialisierte View in den PostgreSQL-Systemkatalogen entsprechen genau denen einer Tabelle oder View. Für den Parser ist eine materialisierte View also eine Relation, genau wie eine Tabelle oder View. Wenn eine materialisierte View in einer Abfrage referenziert wird, werden die Daten direkt aus der materialisierten View zurückgegeben, wie aus einer Tabelle. Die Regel wird nur zum Befüllen der materialisierten View verwendet.

Der Zugriff auf die in einer materialisierten View gespeicherten Daten ist oft deutlich schneller als der direkte Zugriff auf die zugrunde liegenden Tabellen oder der Zugriff über eine View. Die Daten sind allerdings nicht immer aktuell; manchmal werden aktuelle Daten aber auch gar nicht benötigt. Betrachten Sie eine Tabelle, die Verkäufe aufzeichnet:

```sql
CREATE TABLE invoice (
    invoice_no    integer                            PRIMARY KEY,
    seller_no     integer,                           -- ID of salesperson
    invoice_date date,                               -- date of sale
    invoice_amt   numeric(13,2)                      -- amount of sale
);
```

Wenn historische Verkaufsdaten schnell als Diagramm dargestellt werden sollen, kann eine Zusammenfassung sinnvoll sein; unvollständige Daten für das aktuelle Datum sind dabei möglicherweise unerheblich:

```sql
CREATE MATERIALIZED VIEW sales_summary AS
  SELECT
      seller_no,
      invoice_date,
      sum(invoice_amt)::numeric(13,2) as sales_amt
    FROM invoice
    WHERE invoice_date < CURRENT_DATE
    GROUP BY
      seller_no,
      invoice_date;

CREATE UNIQUE INDEX sales_summary_seller
  ON sales_summary (seller_no, invoice_date);
```

Diese materialisierte View könnte nützlich sein, um ein Diagramm im Dashboard für Vertriebsmitarbeiter anzuzeigen. Ein Job könnte so geplant werden, dass er die Statistik jede Nacht mit dieser SQL-Anweisung aktualisiert:

```sql
REFRESH MATERIALIZED VIEW sales_summary;
```

Eine weitere Verwendung einer materialisierten View besteht darin, schnelleren Zugriff auf Daten zu ermöglichen, die über einen Foreign Data Wrapper aus einem entfernten System hereingeholt werden. Das folgende einfache Beispiel verwendet `file_fdw` und zeigt Laufzeiten. Da hierbei allerdings der Cache auf dem lokalen System verwendet wird, wäre der Performance-Unterschied gegenüber dem Zugriff auf ein entferntes System normalerweise größer als hier gezeigt. Beachten Sie außerdem, dass wir die Möglichkeit nutzen, einen Index auf der materialisierten View anzulegen, während `file_fdw` keine Indizes unterstützt. Dieser Vorteil gilt möglicherweise nicht für andere Arten des Zugriffs auf Fremddaten.

Einrichtung:

```sql
CREATE EXTENSION file_fdw;
CREATE SERVER local_file FOREIGN DATA WRAPPER file_fdw;
CREATE FOREIGN TABLE words (word text NOT NULL)
  SERVER local_file
  OPTIONS (filename '/usr/share/dict/words');
CREATE MATERIALIZED VIEW wrd AS SELECT * FROM words;
CREATE UNIQUE INDEX wrd_word ON wrd (word);
CREATE EXTENSION pg_trgm;
CREATE INDEX wrd_trgm ON wrd USING gist (word gist_trgm_ops);
VACUUM ANALYZE wrd;
```

Nun prüfen wir ein Wort auf korrekte Schreibweise. Direkt mit `file_fdw`:

```sql
SELECT count(*) FROM words WHERE word = 'caterpiler';
```

```text
 count
-------
(1 row)
```

Mit `EXPLAIN ANALYZE` sehen wir:

```text
 Aggregate (cost=21763.99..21764.00 rows=1 width=0) (actual
 time=188.180..188.181 rows=1.00 loops=1)
   -> Foreign Scan on words (cost=0.00..21761.41 rows=1032
 width=0) (actual time=188.177..188.177 rows=0.00 loops=1)
         Filter: (word = 'caterpiler'::text)
         Rows Removed by Filter: 479829
         Foreign File: /usr/share/dict/words
         Foreign File Size: 4953699
 Planning time: 0.118 ms
 Execution time: 188.273 ms
```

Wenn stattdessen die materialisierte View verwendet wird, ist die Abfrage deutlich schneller:

```text
 Aggregate (cost=4.44..4.45 rows=1 width=0) (actual
 time=0.042..0.042 rows=1.00 loops=1)
   -> Index Only Scan using wrd_word on wrd (cost=0.42..4.44
 rows=1 width=0) (actual time=0.039..0.039 rows=0.00 loops=1)
         Index Cond: (word = 'caterpiler'::text)
         Heap Fetches: 0
         Index Searches: 1
 Planning time: 0.164 ms
 Execution time: 0.117 ms
```

In beiden Fällen ist das Wort falsch geschrieben. Suchen wir also danach, was wahrscheinlich gemeint war, wieder mit `file_fdw` und `pg_trgm`:

```sql
SELECT word FROM words ORDER BY word <-> 'caterpiler' LIMIT 10;
```

```text
     word
---------------
 cater
 caterpillar
 Caterpillar
 caterpillars
 caterpillar's
 Caterpillar's
 caterer
 caterer's
 caters
 catered
(10 rows)

 Limit (cost=11583.61..11583.64 rows=10 width=32) (actual
 time=1431.591..1431.594 rows=10.00 loops=1)
   -> Sort (cost=11583.61..11804.76 rows=88459 width=32) (actual
 time=1431.589..1431.591 rows=10.00 loops=1)
         Sort Key: ((word <-> 'caterpiler'::text))
         Sort Method: top-N heapsort Memory: 25kB
         -> Foreign Scan on words (cost=0.00..9672.05 rows=88459
 width=32) (actual time=0.057..1286.455 rows=479829.00 loops=1)
               Foreign File: /usr/share/dict/words
               Foreign File Size: 4953699
 Planning time: 0.128 ms
 Execution time: 1431.679 ms
```

Mit der materialisierten View:

```text
 Limit (cost=0.29..1.06 rows=10 width=10) (actual
 time=187.222..188.257 rows=10.00 loops=1)
   -> Index Scan using wrd_trgm on wrd (cost=0.29..37020.87
 rows=479829 width=10) (actual time=187.219..188.252 rows=10.00
 loops=1)
          Order By: (word <-> 'caterpiler'::text)
          Index Searches: 1
 Planning time: 0.196 ms
 Execution time: 198.640 ms
```

Wenn man eine periodische Aktualisierung der entfernten Daten in die lokale Datenbank tolerieren kann, kann der Performance-Gewinn erheblich sein.

## 39.4. Regeln für INSERT, UPDATE und DELETE

Regeln, die für `INSERT`, `UPDATE` und `DELETE` definiert werden, unterscheiden sich deutlich von den in den vorherigen Abschnitten beschriebenen View-Regeln. Erstens erlaubt ihr `CREATE RULE`-Befehl mehr:

- Sie dürfen keine Aktion haben.
- Sie können mehrere Aktionen haben.
- Sie können `INSTEAD` oder `ALSO`, den Standard, sein.
- Die Pseudorelationen `NEW` und `OLD` werden nützlich.
- Sie können Regelqualifikationen haben.

Zweitens verändern sie den Abfragebaum nicht an Ort und Stelle. Stattdessen erzeugen sie null oder mehr neue Abfragebäume und können den ursprünglichen verwerfen.

> Vorsicht: In vielen Fällen lassen sich Aufgaben, die mit Regeln auf `INSERT`/`UPDATE`/`DELETE` erledigt werden könnten, besser mit Triggern lösen. Trigger sind in der Notation etwas komplizierter, ihre Semantik ist aber viel einfacher zu verstehen. Regeln führen häufig zu überraschenden Ergebnissen, wenn die ursprüngliche Abfrage volatile Funktionen enthält: Volatile Funktionen können beim Ausführen der Regeln öfter ausgeführt werden als erwartet.
>
> Außerdem gibt es einige Fälle, die von diesen Regeltypen überhaupt nicht unterstützt werden, insbesondere `WITH`-Klauseln in der ursprünglichen Abfrage und Mehrfachzuweisungs-Sub-`SELECT`s in der `SET`-Liste von `UPDATE`-Abfragen. Der Grund ist, dass das Kopieren solcher Konstrukte in eine Regelabfrage zu mehrfacher Auswertung der Unterabfrage führen würde, entgegen der ausdrücklichen Absicht des Autors der Abfrage.

### 39.4.1. Wie Update-Regeln funktionieren

Behalten Sie die Syntax im Blick:

```sql
CREATE [ OR REPLACE ] RULE name AS ON event
    TO table [ WHERE condition ]
    DO [ ALSO | INSTEAD ] { NOTHING | command | ( command ; command
 ... ) }
```

Im Folgenden bedeutet Update-Regeln: Regeln, die auf `INSERT`, `UPDATE` oder `DELETE` definiert sind.

Update-Regeln werden vom Regelsystem angewendet, wenn Ergebnisrelation und Befehlstyp eines Abfragebaums dem Objekt und Ereignis entsprechen, die im Befehl `CREATE RULE` angegeben wurden. Für Update-Regeln erzeugt das Regelsystem eine Liste von Abfragebäumen. Anfangs ist diese Liste leer. Es kann null Aktionen geben, also das Schlüsselwort `NOTHING`, eine Aktion oder mehrere Aktionen. Zur Vereinfachung betrachten wir eine Regel mit einer Aktion. Diese Regel kann eine Qualifikation besitzen oder nicht, und sie kann `INSTEAD` oder `ALSO`, den Standard, sein.

Was ist eine Regelqualifikation? Sie ist eine Einschränkung, die festlegt, wann die Aktionen der Regel ausgeführt werden sollen und wann nicht. Diese Qualifikation darf nur auf die Pseudorelationen `NEW` und/oder `OLD` verweisen, die im Wesentlichen die als Objekt angegebene Relation darstellen, allerdings mit besonderer Bedeutung.

Es gibt also drei Fälle, die für eine Regel mit einer Aktion die folgenden Abfragebäume erzeugen:

- Keine Qualifikation, mit entweder `ALSO` oder `INSTEAD`
  - Der Abfragebaum aus der Regelaktion, ergänzt um die Qualifikation des ursprünglichen Abfragebaums.

- Qualifikation angegeben und `ALSO`
  - Der Abfragebaum aus der Regelaktion, ergänzt um die Regelqualifikation und die Qualifikation des ursprünglichen Abfragebaums.

- Qualifikation angegeben und `INSTEAD`
  - Der Abfragebaum aus der Regelaktion mit der Regelqualifikation und der Qualifikation des ursprünglichen Abfragebaums; außerdem der ursprüngliche Abfragebaum mit der negierten Regelqualifikation.

Wenn die Regel schließlich `ALSO` ist, wird der unveränderte ursprüngliche Abfragebaum zur Liste hinzugefügt. Da nur qualifizierte `INSTEAD`-Regeln den ursprünglichen Abfragebaum bereits hinzufügen, erhalten wir bei einer Regel mit einer Aktion am Ende entweder einen oder zwei Ausgabe-Abfragebäume.

Bei `ON INSERT`-Regeln wird die ursprüngliche Abfrage, sofern sie nicht durch `INSTEAD` unterdrückt wird, vor allen durch Regeln hinzugefügten Aktionen ausgeführt. Dadurch können die Aktionen die eingefügte(n) Zeile(n) sehen. Bei `ON UPDATE`- und `ON DELETE`-Regeln wird die ursprüngliche Abfrage dagegen nach den durch Regeln hinzugefügten Aktionen ausgeführt. Dadurch können die Aktionen die zu aktualisierenden oder zu löschenden Zeilen sehen; andernfalls würden die Aktionen möglicherweise nichts tun, weil sie keine Zeilen finden, die ihren Qualifikationen entsprechen.

Die aus Regelaktionen erzeugten Abfragebäume werden erneut in das Rewrite-System gegeben, und möglicherweise werden weitere Regeln angewendet, sodass zusätzliche oder weniger Abfragebäume entstehen. Daher müssen die Aktionen einer Regel entweder einen anderen Befehlstyp oder eine andere Ergebnisrelation haben als die Regel selbst. Andernfalls würde dieser rekursive Prozess in einer Endlosschleife enden. Eine rekursive Expansion einer Regel wird erkannt und als Fehler gemeldet.

Die in den Aktionen des Systemkatalogs `pg_rewrite` gespeicherten Abfragebäume sind nur Vorlagen. Da sie auf die Range-Table-Einträge für `NEW` und `OLD` verweisen können, müssen einige Ersetzungen vorgenommen werden, bevor sie verwendet werden können. Für jeden Verweis auf `NEW` wird die Zielliste der ursprünglichen Abfrage nach einem entsprechenden Eintrag durchsucht. Wird einer gefunden, ersetzt dessen Ausdruck den Verweis. Andernfalls bedeutet `NEW` dasselbe wie `OLD`, bei einem `UPDATE`, oder wird bei einem `INSERT` durch einen Nullwert ersetzt. Jeder Verweis auf `OLD` wird durch einen Verweis auf den Range-Table-Eintrag ersetzt, der die Ergebnisrelation ist.

Nachdem das System die Update-Regeln angewendet hat, wendet es View-Regeln auf die erzeugten Abfragebäume an. Views können keine neuen Update-Aktionen einfügen, daher müssen auf die Ausgabe des View-Rewrites keine Update-Regeln angewendet werden.

#### 39.4.1.1. Eine erste Regel Schritt für Schritt

Angenommen, wir möchten Änderungen an der Spalte `sl_avail` in der Relation `shoelace_data` verfolgen. Dazu richten wir eine Log-Tabelle und eine Regel ein, die bedingt einen Log-Eintrag schreibt, wenn ein `UPDATE` auf `shoelace_data` ausgeführt wird.

```sql
CREATE TABLE shoelace_log (
    sl_name    text,                           -- shoelace changed
    sl_avail   integer,                        -- new available value
    log_who    text,                           -- who did it
    log_when   timestamp                       -- when
);

CREATE RULE log_shoelace AS ON UPDATE TO shoelace_data
    WHERE NEW.sl_avail <> OLD.sl_avail
    DO INSERT INTO shoelace_log VALUES (
                                    NEW.sl_name,
                                    NEW.sl_avail,
                                    current_user,
                                    current_timestamp
                                );
```

Nun führt jemand Folgendes aus:

```sql
UPDATE shoelace_data SET sl_avail = 6 WHERE sl_name = 'sl7';
```

und wir sehen uns die Log-Tabelle an:

```sql
SELECT * FROM shoelace_log;
```

```text
 sl_name | sl_avail | log_who | log_when
---------+----------+---------+----------------------------------
 sl7     |        6 | Al      | Tue Oct 20 16:14:45 1998 MET DST
(1 row)
```

Das entspricht der Erwartung. Im Hintergrund ist Folgendes geschehen. Der Parser erzeugte diesen Abfragebaum:

```sql
UPDATE shoelace_data SET sl_avail = 6
  FROM shoelace_data shoelace_data
 WHERE shoelace_data.sl_name = 'sl7';
```

Es gibt eine Regel `log_shoelace`, die `ON UPDATE` ist und diesen Regelqualifikationsausdruck besitzt:

```sql
NEW.sl_avail <> OLD.sl_avail
```

und diese Aktion:

```sql
INSERT INTO shoelace_log VALUES (
       new.sl_name, new.sl_avail,
       current_user, current_timestamp )
  FROM shoelace_data new, shoelace_data old;
```

Dies sieht etwas seltsam aus, da man normalerweise kein `INSERT ... VALUES ... FROM` schreiben kann. Die `FROM`-Klausel zeigt hier nur an, dass es im Abfragebaum Range-Table-Einträge für `new` und `old` gibt. Diese werden benötigt, damit Variablen im Abfragebaum des `INSERT`-Befehls darauf verweisen können.

Die Regel ist eine qualifizierte `ALSO`-Regel, also muss das Regelsystem zwei Abfragebäume zurückgeben: die modifizierte Regelaktion und den ursprünglichen Abfragebaum. In Schritt 1 wird die Range Table der ursprünglichen Abfrage in den Aktionsabfragebaum der Regel eingearbeitet. Daraus ergibt sich:

```sql
INSERT INTO shoelace_log VALUES (
       new.sl_name, new.sl_avail,
       current_user, current_timestamp )
  FROM shoelace_data new, shoelace_data old,
       shoelace_data shoelace_data;
```

In Schritt 2 wird die Regelqualifikation hinzugefügt, sodass die Ergebnismenge auf Zeilen beschränkt wird, bei denen sich `sl_avail` ändert:

```sql
INSERT INTO shoelace_log VALUES (
       new.sl_name, new.sl_avail,
       current_user, current_timestamp )
  FROM shoelace_data new, shoelace_data old,
       shoelace_data shoelace_data
 WHERE new.sl_avail <> old.sl_avail;
```

Dies sieht noch seltsamer aus, weil `INSERT ... VALUES` auch keine `WHERE`-Klausel hat. Planner und Executor haben damit aber keine Schwierigkeiten. Dieselbe Funktionalität müssen sie ohnehin für `INSERT ... SELECT` unterstützen.

In Schritt 3 wird die Qualifikation des ursprünglichen Abfragebaums hinzugefügt. Dadurch wird die Ergebnismenge weiter auf diejenigen Zeilen beschränkt, die von der ursprünglichen Abfrage betroffen gewesen wären:

```sql
INSERT INTO shoelace_log VALUES (
       new.sl_name, new.sl_avail,
       current_user, current_timestamp )
  FROM shoelace_data new, shoelace_data old,
       shoelace_data shoelace_data
 WHERE new.sl_avail <> old.sl_avail
   AND shoelace_data.sl_name = 'sl7';
```

Schritt 4 ersetzt Verweise auf `NEW` durch die Ziellisteneinträge aus dem ursprünglichen Abfragebaum oder durch die passenden Variablenverweise aus der Ergebnisrelation:

```sql
INSERT INTO shoelace_log VALUES (
       shoelace_data.sl_name, 6,
       current_user, current_timestamp )
  FROM shoelace_data new, shoelace_data old,
       shoelace_data shoelace_data
 WHERE 6 <> old.sl_avail
   AND shoelace_data.sl_name = 'sl7';
```

Schritt 5 wandelt `OLD`-Verweise in Verweise auf die Ergebnisrelation um:

```sql
INSERT INTO shoelace_log VALUES (
       shoelace_data.sl_name, 6,
       current_user, current_timestamp )
  FROM shoelace_data new, shoelace_data old,
       shoelace_data shoelace_data
 WHERE 6 <> shoelace_data.sl_avail
   AND shoelace_data.sl_name = 'sl7';
```

Das ist alles. Da die Regel `ALSO` ist, geben wir zusätzlich den ursprünglichen Abfragebaum aus. Kurz gesagt ist die Ausgabe des Regelsystems eine Liste aus zwei Abfragebäumen, die diesen Anweisungen entsprechen:

```sql
INSERT INTO shoelace_log VALUES (
       shoelace_data.sl_name, 6,
       current_user, current_timestamp )
  FROM shoelace_data
 WHERE 6 <> shoelace_data.sl_avail
   AND shoelace_data.sl_name = 'sl7';

UPDATE shoelace_data SET sl_avail = 6
 WHERE sl_name = 'sl7';
```

Sie werden in dieser Reihenfolge ausgeführt, und genau das sollte die Regel bewirken.

Die Ersetzungen und die hinzugefügten Qualifikationen stellen sicher, dass kein Log-Eintrag geschrieben würde, wenn die ursprüngliche Abfrage beispielsweise so lautete:

```sql
UPDATE shoelace_data SET sl_color = 'green'
 WHERE sl_name = 'sl7';
```

In diesem Fall enthält der ursprüngliche Abfragebaum keinen Ziellisteneintrag für `sl_avail`, sodass `NEW.sl_avail` durch `shoelace_data.sl_avail` ersetzt wird. Der zusätzliche Befehl, den die Regel erzeugt, lautet dann:

```sql
INSERT INTO shoelace_log VALUES (
       shoelace_data.sl_name, shoelace_data.sl_avail,
       current_user, current_timestamp )
  FROM shoelace_data
 WHERE shoelace_data.sl_avail <> shoelace_data.sl_avail
   AND shoelace_data.sl_name = 'sl7';
```

Diese Qualifikation kann nie wahr sein.

Das funktioniert auch dann, wenn die ursprüngliche Abfrage mehrere Zeilen verändert. Wenn also jemand diesen Befehl ausführt:

```sql
UPDATE shoelace_data SET sl_avail = 0
 WHERE sl_color = 'black';
```

werden tatsächlich vier Zeilen aktualisiert, nämlich `sl1`, `sl2`, `sl3` und `sl4`. `sl3` hat jedoch bereits `sl_avail = 0`. In diesem Fall ist die Qualifikation des ursprünglichen Abfragebaums anders, und daraus entsteht dieser zusätzliche Abfragebaum:

```sql
INSERT INTO shoelace_log
SELECT shoelace_data.sl_name, 0,
       current_user, current_timestamp
  FROM shoelace_data
 WHERE 0 <> shoelace_data.sl_avail
   AND shoelace_data.sl_color = 'black';
```

Dieser Abfragebaum wird von der Regel erzeugt und fügt sicher drei neue Log-Einträge ein. Und genau das ist korrekt.

Hier sieht man, warum es wichtig ist, dass der ursprüngliche Abfragebaum zuletzt ausgeführt wird. Wäre das `UPDATE` zuerst ausgeführt worden, wären alle Zeilen bereits auf null gesetzt, sodass das protokollierende `INSERT` keine Zeile mehr finden würde, für die `0 <> shoelace_data.sl_avail` gilt.

### 39.4.2. Zusammenarbeit mit Views

Eine einfache Möglichkeit, View-Relationen vor der erwähnten Möglichkeit zu schützen, dass jemand versucht, `INSERT`, `UPDATE` oder `DELETE` auf ihnen auszuführen, besteht darin, diese Abfragebäume verwerfen zu lassen. Wir könnten also folgende Regeln erstellen:

```sql
CREATE RULE shoe_ins_protect AS ON INSERT TO shoe
    DO INSTEAD NOTHING;
CREATE RULE shoe_upd_protect AS ON UPDATE TO shoe
    DO INSTEAD NOTHING;
CREATE RULE shoe_del_protect AS ON DELETE TO shoe
    DO INSTEAD NOTHING;
```

Wenn nun jemand versucht, eine dieser Operationen auf der View-Relation `shoe` auszuführen, wendet das Regelsystem diese Regeln an. Da die Regeln keine Aktionen haben und `INSTEAD` sind, ist die resultierende Liste von Abfragebäumen leer. Die gesamte Abfrage wird damit zu nichts, weil nach Abschluss des Regelsystems nichts mehr übrig ist, was optimiert oder ausgeführt werden könnte.

Eine ausgefeiltere Verwendung des Regelsystems besteht darin, Regeln zu erstellen, die den Abfragebaum in einen Abfragebaum umschreiben, der die richtige Operation auf den echten Tabellen ausführt. Für die View `shoelace` erstellen wir dazu folgende Regeln:

```sql
CREATE RULE shoelace_ins AS ON INSERT TO shoelace
    DO INSTEAD
    INSERT INTO shoelace_data VALUES (
           NEW.sl_name,
           NEW.sl_avail,
           NEW.sl_color,
           NEW.sl_len,
           NEW.sl_unit
    );

CREATE RULE shoelace_upd AS ON UPDATE TO shoelace
    DO INSTEAD
    UPDATE shoelace_data
       SET sl_name = NEW.sl_name,
           sl_avail = NEW.sl_avail,
           sl_color = NEW.sl_color,
           sl_len = NEW.sl_len,
           sl_unit = NEW.sl_unit
     WHERE sl_name = OLD.sl_name;

CREATE RULE shoelace_del AS ON DELETE TO shoelace
    DO INSTEAD
    DELETE FROM shoelace_data
     WHERE sl_name = OLD.sl_name;
```

Wenn `RETURNING`-Abfragen auf der View unterstützt werden sollen, müssen die Regeln `RETURNING`-Klauseln enthalten, die die View-Zeilen berechnen. Bei Views auf einer einzelnen Tabelle ist das normalerweise ziemlich trivial, bei Join-Views wie `shoelace` aber etwas mühsam. Ein Beispiel für den Einfügefall ist:

```sql
CREATE RULE shoelace_ins AS ON INSERT TO shoelace
    DO INSTEAD
    INSERT INTO shoelace_data VALUES (
           NEW.sl_name,
           NEW.sl_avail,
           NEW.sl_color,
           NEW.sl_len,
           NEW.sl_unit
    )
    RETURNING
           shoelace_data.*,
           (SELECT shoelace_data.sl_len * u.un_fact
            FROM unit u WHERE shoelace_data.sl_unit = u.un_name);
```

Beachten Sie, dass diese eine Regel sowohl `INSERT`- als auch `INSERT RETURNING`-Abfragen auf der View unterstützt. Die `RETURNING`-Klausel wird für `INSERT` einfach ignoriert.

Beachten Sie außerdem, dass sich `OLD` und `NEW` in der `RETURNING`-Klausel einer Regel auf die Pseudorelationen beziehen, die der umgeschriebenen Abfrage als zusätzliche Range-Table-Einträge hinzugefügt wurden, nicht auf alte/neue Zeilen in der Ergebnisrelation. Wenn beispielsweise in einer Regel, die `UPDATE`-Abfragen auf dieser View unterstützt, die `RETURNING`-Klausel `old.sl_name` enthielte, würde immer der alte Name zurückgegeben, unabhängig davon, ob die `RETURNING`-Klausel in der Abfrage auf der View `OLD` oder `NEW` angegeben hat. Das kann verwirrend sein. Um diese Verwirrung zu vermeiden und die Rückgabe alter und neuer Werte in Abfragen auf der View zu unterstützen, sollte die `RETURNING`-Klausel in der Regeldefinition auf Einträge der Ergebnisrelation verweisen, etwa `shoelace_data.sl_name`, ohne `OLD` oder `NEW` anzugeben.

Nehmen wir nun an, dass ab und zu ein Paket Schnürsenkel im Geschäft eintrifft, zusammen mit einer großen Stückliste. Sie möchten die View `shoelace` aber nicht jedes Mal manuell aktualisieren. Stattdessen richten wir zwei kleine Tabellen ein: eine, in die die Positionen aus der Stückliste eingetragen werden können, und eine mit einem besonderen Trick. Die Erstellungsbefehle dafür lauten:

```sql
CREATE TABLE shoelace_arrive (
    arr_name    text,
    arr_quant   integer
);

CREATE TABLE shoelace_ok (
    ok_name     text,
    ok_quant    integer
);

CREATE RULE shoelace_ok_ins AS ON INSERT TO shoelace_ok
    DO INSTEAD
    UPDATE shoelace
       SET sl_avail = sl_avail + NEW.ok_quant
     WHERE sl_name = NEW.ok_name;
```

Nun können Sie die Tabelle `shoelace_arrive` mit den Daten aus der Stückliste füllen:

```sql
SELECT * FROM shoelace_arrive;
```

```text
 arr_name | arr_quant
----------+-----------
 sl3      |        10
 sl6      |        20
 sl8      |        20
(3 rows)
```

Ein kurzer Blick auf die aktuellen Daten:

```sql
SELECT * FROM shoelace;
```

```text
 sl_name | sl_avail | sl_color | sl_len | sl_unit | sl_len_cm
----------+----------+----------+--------+---------+-----------
 sl1      |        5 | black    |     80 | cm      |        80
 sl2      |        6 | black    |    100 | cm      |       100
 sl7      |        6 | brown    |     60 | cm      |        60
 sl3      |        0 | black    |     35 | inch    |      88.9
 sl4      |        8 | black    |     40 | inch    |     101.6
 sl8      |        1 | brown    |     40 | inch    |     101.6
 sl5      |        4 | brown    |      1 | m       |       100
 sl6      |        0 | brown    |    0.9 | m       |        90
(8 rows)
```

Nun werden die eingetroffenen Schnürsenkel eingebucht:

```sql
INSERT INTO shoelace_ok SELECT * FROM shoelace_arrive;
```

und die Ergebnisse geprüft:

```sql
SELECT * FROM shoelace ORDER BY sl_name;
```

```text
 sl_name | sl_avail | sl_color | sl_len | sl_unit | sl_len_cm
----------+----------+----------+--------+---------+-----------
 sl1      |        5 | black    |     80 | cm      |        80
 sl2      |        6 | black    |    100 | cm      |       100
 sl7      |        6 | brown    |     60 | cm      |        60
 sl4      |        8 | black    |     40 | inch    |     101.6
 sl3      |       10 | black    |     35 | inch    |      88.9
 sl8      |       21 | brown    |     40 | inch    |     101.6
 sl5      |        4 | brown    |      1 | m       |       100
 sl6      |       20 | brown    |    0.9 | m       |        90
(8 rows)
```

```sql
SELECT * FROM shoelace_log;
```

```text
 sl_name | sl_avail | log_who| log_when
---------+----------+--------+----------------------------------
 sl7     |        6 | Al     | Tue Oct 20 19:14:45 1998 MET DST
 sl3     |       10 | Al     | Tue Oct 20 19:25:16 1998 MET DST
 sl6     |       20 | Al     | Tue Oct 20 19:25:16 1998 MET DST
 sl8     |       21 | Al     | Tue Oct 20 19:25:16 1998 MET DST
(4 rows)
```

Von dem einen `INSERT ... SELECT` bis zu diesen Ergebnissen ist es ein weiter Weg. Die Beschreibung der Abfragebaumtransformation ist die letzte in diesem Kapitel. Zuerst gibt es die Ausgabe des Parsers:

```sql
INSERT INTO shoelace_ok
SELECT shoelace_arrive.arr_name, shoelace_arrive.arr_quant
  FROM shoelace_arrive shoelace_arrive, shoelace_ok shoelace_ok;
```

Nun wird die erste Regel `shoelace_ok_ins` angewendet und wandelt dies um in:

```sql
UPDATE shoelace
   SET sl_avail = shoelace.sl_avail + shoelace_arrive.arr_quant
  FROM shoelace_arrive shoelace_arrive, shoelace_ok shoelace_ok,
       shoelace_ok old, shoelace_ok new,
       shoelace shoelace
 WHERE shoelace.sl_name = shoelace_arrive.arr_name;
```

Sie verwirft außerdem das ursprüngliche `INSERT` auf `shoelace_ok`. Diese umgeschriebene Abfrage wird erneut an das Regelsystem übergeben, und die zweite angewendete Regel `shoelace_upd` erzeugt:

```sql
UPDATE shoelace_data
   SET sl_name = shoelace.sl_name,
       sl_avail = shoelace.sl_avail + shoelace_arrive.arr_quant,
       sl_color = shoelace.sl_color,
       sl_len = shoelace.sl_len,
       sl_unit = shoelace.sl_unit
  FROM shoelace_arrive shoelace_arrive, shoelace_ok shoelace_ok,
       shoelace_ok old, shoelace_ok new,
       shoelace shoelace, shoelace old,
       shoelace new, shoelace_data shoelace_data
 WHERE shoelace.sl_name = shoelace_arrive.arr_name
   AND shoelace_data.sl_name = shoelace.sl_name;
```

Auch dies ist wieder eine `INSTEAD`-Regel, und der vorherige Abfragebaum wird verworfen. Beachten Sie, dass diese Abfrage noch immer die View `shoelace` verwendet. Das Regelsystem ist mit diesem Schritt aber noch nicht fertig; es fährt fort, wendet die Regel `_RETURN` darauf an, und wir erhalten:

```sql
UPDATE shoelace_data
   SET sl_name = s.sl_name,
       sl_avail = s.sl_avail + shoelace_arrive.arr_quant,
       sl_color = s.sl_color,
       sl_len = s.sl_len,
       sl_unit = s.sl_unit
  FROM shoelace_arrive shoelace_arrive, shoelace_ok shoelace_ok,
       shoelace_ok old, shoelace_ok new,
       shoelace shoelace, shoelace old,
       shoelace new, shoelace_data shoelace_data,
       shoelace old, shoelace new,
       shoelace_data s, unit u
 WHERE s.sl_name = shoelace_arrive.arr_name
   AND shoelace_data.sl_name = s.sl_name;
```

Schließlich wird die Regel `log_shoelace` angewendet und erzeugt den zusätzlichen Abfragebaum:

```sql
INSERT INTO shoelace_log
SELECT s.sl_name,
       s.sl_avail + shoelace_arrive.arr_quant,
       current_user,
       current_timestamp
  FROM shoelace_arrive shoelace_arrive, shoelace_ok shoelace_ok,
       shoelace_ok old, shoelace_ok new,
       shoelace shoelace, shoelace old,
       shoelace new, shoelace_data shoelace_data,
       shoelace old, shoelace new,
       shoelace_data s, unit u,
       shoelace_data old, shoelace_data new
       shoelace_log shoelace_log
 WHERE s.sl_name = shoelace_arrive.arr_name
   AND shoelace_data.sl_name = s.sl_name
   AND (s.sl_avail + shoelace_arrive.arr_quant) <> s.sl_avail;
```

Danach gehen dem Regelsystem die Regeln aus, und es gibt die erzeugten Abfragebäume zurück.

Am Ende erhalten wir also zwei endgültige Abfragebäume, die diesen SQL-Anweisungen entsprechen:

```sql
INSERT INTO shoelace_log
SELECT s.sl_name,
       s.sl_avail + shoelace_arrive.arr_quant,
       current_user,
       current_timestamp
  FROM shoelace_arrive shoelace_arrive, shoelace_data
 shoelace_data,
       shoelace_data s
 WHERE s.sl_name = shoelace_arrive.arr_name
   AND shoelace_data.sl_name = s.sl_name
   AND s.sl_avail + shoelace_arrive.arr_quant <> s.sl_avail;

UPDATE shoelace_data
   SET sl_avail = shoelace_data.sl_avail +
 shoelace_arrive.arr_quant
  FROM shoelace_arrive shoelace_arrive,
       shoelace_data shoelace_data,
       shoelace_data s
 WHERE s.sl_name = shoelace_arrive.sl_name
   AND shoelace_data.sl_name = s.sl_name;
```

Das Ergebnis ist, dass Daten, die aus einer Relation kommen und in eine andere eingefügt werden, in Updates auf eine dritte Relation verwandelt werden, dann in eine Aktualisierung einer vierten Relation plus Protokollierung dieser endgültigen Aktualisierung in einer fünften Relation. All das wird auf zwei Abfragen reduziert.

Ein kleines Detail ist etwas unschön. Betrachtet man die beiden Abfragen, zeigt sich, dass die Relation `shoelace_data` in der Range Table zweimal vorkommt, obwohl sie hier eindeutig auf einen Eintrag reduziert werden könnte. Der Planner behandelt das nicht, und daher sieht der Ausführungsplan für die Ausgabe des Regelsystems zum `INSERT` so aus:

```text
Nested Loop
  -> Merge Join
        -> Seq Scan
             -> Sort
                    -> Seq Scan on s
        -> Seq Scan
             -> Sort
                    -> Seq Scan on shoelace_arrive
  -> Seq Scan on shoelace_data
```

Wenn der zusätzliche Range-Table-Eintrag weggelassen würde, ergäbe sich:

```text
Merge Join
  -> Seq Scan
        -> Sort
              ->           Seq Scan on s
  -> Seq Scan
        -> Sort
              ->           Seq Scan on shoelace_arrive
```

Dies erzeugt exakt dieselben Einträge in der Log-Tabelle. Das Regelsystem hat also einen zusätzlichen Scan auf der Tabelle `shoelace_data` verursacht, der absolut nicht nötig ist. Derselbe redundante Scan wird im `UPDATE` noch einmal durchgeführt. Aber es war schon eine wirklich harte Aufgabe, all dies überhaupt möglich zu machen.

Nun folgt eine letzte Demonstration des PostgreSQL-Regelsystems und seiner Leistungsfähigkeit. Angenommen, Sie fügen Ihrer Datenbank einige Schnürsenkel mit außergewöhnlichen Farben hinzu:

```sql
INSERT INTO shoelace VALUES ('sl9', 0, 'pink', 35.0, 'inch', 0.0);
INSERT INTO shoelace VALUES ('sl10', 1000, 'magenta', 40.0, 'inch',
 0.0);
```

Wir möchten eine View erstellen, mit der geprüft werden kann, welche Schnürsenkeleinträge farblich zu keinem Schuh passen. Die View dafür ist:

```sql
CREATE VIEW shoelace_mismatch AS
    SELECT * FROM shoelace WHERE NOT EXISTS
        (SELECT shoename FROM shoe WHERE slcolor = sl_color);
```

Ihre Ausgabe ist:

```sql
SELECT * FROM shoelace_mismatch;
```

```text
 sl_name | sl_avail | sl_color | sl_len | sl_unit | sl_len_cm
---------+----------+----------+--------+---------+-----------
 sl9     |        0 | pink     |     35 | inch    |      88.9
 sl10    |     1000 | magenta |      40 | inch    |     101.6
```

Nun möchten wir erreichen, dass nicht vorrätige, nicht passende Schnürsenkel aus der Datenbank gelöscht werden. Um es für PostgreSQL etwas schwieriger zu machen, löschen wir sie nicht direkt. Stattdessen erstellen wir eine weitere View:

```sql
CREATE VIEW shoelace_can_delete AS
    SELECT * FROM shoelace_mismatch WHERE sl_avail = 0;
```

und führen es so aus:

```sql
DELETE FROM shoelace WHERE EXISTS
    (SELECT * FROM shoelace_can_delete
             WHERE sl_name = shoelace.sl_name);
```

Das Ergebnis ist:

```sql
SELECT * FROM shoelace;
```

```text
 sl_name | sl_avail | sl_color | sl_len | sl_unit | sl_len_cm
---------+----------+----------+--------+---------+-----------
 sl1     |        5 | black    |     80 | cm      |        80
 sl2     |        6 | black    |    100 | cm      |       100
 sl7     |        6 | brown    |     60 | cm      |        60
 sl4     |        8 | black    |     40 | inch    |     101.6
 sl3     |       10 | black    |     35 | inch    |      88.9
 sl8     |       21 | brown    |     40 | inch    |     101.6
 sl10    |     1000 | magenta |      40 | inch    |     101.6
 sl5     |        4 | brown    |      1 | m       |       100
 sl6     |       20 | brown    |    0.9 | m       |        90
(9 rows)
```

Ein `DELETE` auf einer View, mit einer Unterabfragequalifikation, die insgesamt vier geschachtelte/gejointe Views verwendet, wobei eine davon selbst eine Unterabfragequalifikation enthält, die eine View enthält, und bei der berechnete View-Spalten verwendet werden, wird in einen einzigen Abfragebaum umgeschrieben, der die gewünschten Daten aus einer echten Tabelle löscht.

In der Praxis gibt es wahrscheinlich nur wenige Situationen, in denen ein solches Konstrukt nötig ist. Es ist aber beruhigend, dass es funktioniert.

## 39.5. Regeln und Privilegien

Durch das Umschreiben von Abfragen durch das PostgreSQL-Regelsystem werden andere Tabellen oder Views angesprochen als diejenigen, die in der ursprünglichen Abfrage verwendet wurden. Wenn Update-Regeln verwendet werden, kann dies auch Schreibzugriff auf Tabellen einschließen.

Rewrite-Regeln haben keinen eigenen Besitzer. Der Besitzer einer Relation, also einer Tabelle oder View, ist automatisch der Besitzer der für sie definierten Rewrite-Regeln. Das PostgreSQL-Regelsystem verändert das Verhalten des normalen Zugriffskontrollsystems. Mit Ausnahme von `SELECT`-Regeln, die zu Security-Invoker-Views gehören, siehe `CREATE VIEW`, werden alle Relationen, die aufgrund von Regeln verwendet werden, gegen die Privilegien des Regelbesitzers geprüft, nicht gegen die des Benutzers, der die Regel auslöst. Das bedeutet: Außer bei Security-Invoker-Views benötigen Benutzer nur die erforderlichen Privilegien für die Tabellen oder Views, die in ihren Abfragen ausdrücklich genannt sind.

Beispiel: Ein Benutzer besitzt eine Liste von Telefonnummern, von denen einige privat sind, während andere für die Büroassistenz von Interesse sind. Der Benutzer kann Folgendes anlegen:

```sql
CREATE TABLE phone_data (person text, phone text, private boolean);
CREATE VIEW phone_number AS
    SELECT person, CASE WHEN NOT private THEN phone END AS phone
    FROM phone_data;
GRANT SELECT ON phone_number TO assistant;
```

Niemand außer diesem Benutzer und den Datenbank-Superusern kann auf die Tabelle `phone_data` zugreifen. Durch das `GRANT` kann die Assistenz aber ein `SELECT` auf der View `phone_number` ausführen. Das Regelsystem schreibt das `SELECT` von `phone_number` in ein `SELECT` von `phone_data` um. Da der Benutzer der Besitzer von `phone_number` und damit der Besitzer der Regel ist, wird der Lesezugriff auf `phone_data` nun gegen die Privilegien des Benutzers geprüft, und die Abfrage wird erlaubt. Die Prüfung für den Zugriff auf `phone_number` wird ebenfalls durchgeführt, aber gegen den aufrufenden Benutzer. Daher können nur der Benutzer und die Assistenz diese View verwenden.

Die Privilegien werden Regel für Regel geprüft. Die Assistenz ist also zunächst die einzige Person, die die öffentlichen Telefonnummern sehen kann. Sie kann aber eine weitere View einrichten und der Öffentlichkeit Zugriff darauf gewähren. Dann kann jeder die Daten von `phone_number` über die View der Assistenz sehen. Was die Assistenz nicht tun kann, ist eine View zu erstellen, die direkt auf `phone_data` zugreift. Genau genommen kann sie die View erstellen, aber sie funktioniert nicht, weil jeder Zugriff bei den Rechteprüfungen abgelehnt wird. Sobald der Benutzer bemerkt, dass die Assistenz ihre `phone_number`-View geöffnet hat, kann er der Assistenz den Zugriff wieder entziehen. Jeder Zugriff auf die View der Assistenz würde dann sofort fehlschlagen.

Man könnte denken, dass diese Regel-für-Regel-Prüfung ein Sicherheitsloch ist, aber das ist sie tatsächlich nicht. Würde es nicht so funktionieren, könnte die Assistenz eine Tabelle mit denselben Spalten wie `phone_number` anlegen und die Daten einmal täglich dorthin kopieren. Dann wären es ihre eigenen Daten, und sie könnte den Zugriff jedem gewähren, den sie möchte. Ein `GRANT`-Befehl bedeutet: „Ich vertraue dir.“ Wenn jemand, dem man vertraut, das oben Beschriebene tut, ist es Zeit, das zu überdenken und dann `REVOKE` zu verwenden.

Beachten Sie, dass Views zwar mit der oben gezeigten Technik verwendet werden können, um Inhalte bestimmter Spalten zu verbergen, dass sie aber Daten in nicht sichtbaren Zeilen nicht zuverlässig verbergen können, sofern nicht das Flag `security_barrier` gesetzt wurde. Die folgende View ist beispielsweise unsicher:

```sql
CREATE VIEW phone_number AS
    SELECT person, phone FROM phone_data WHERE phone NOT LIKE
 '412%';
```

Diese View mag sicher erscheinen, weil das Regelsystem jedes `SELECT` von `phone_number` in ein `SELECT` von `phone_data` umschreibt und die Qualifikation hinzufügt, dass nur Einträge gewünscht sind, bei denen `phone` nicht mit `412` beginnt. Wenn der Benutzer jedoch eigene Funktionen erstellen kann, ist es nicht schwierig, den Planner dazu zu bringen, die benutzerdefinierte Funktion vor dem Ausdruck `NOT LIKE` auszuführen. Zum Beispiel:

```sql
CREATE FUNCTION tricky(text, text) RETURNS bool AS $$
BEGIN
     RAISE NOTICE '% => %', $1, $2;
     RETURN true;
END;
$$ LANGUAGE plpgsql COST 0.0000000000000000000001;

SELECT * FROM phone_number WHERE tricky(person, phone);
```

Jede Person und jede Telefonnummer in der Tabelle `phone_data` wird als `NOTICE` ausgegeben, weil der Planner sich dafür entscheidet, die billige Funktion `tricky` vor dem teureren `NOT LIKE` auszuführen. Selbst wenn der Benutzer daran gehindert wird, neue Funktionen zu definieren, können eingebaute Funktionen für ähnliche Angriffe verwendet werden. Beispielsweise enthalten die meisten Cast-Funktionen ihre Eingabewerte in den Fehlermeldungen, die sie erzeugen.

Ähnliche Überlegungen gelten für Update-Regeln. In den Beispielen des vorherigen Abschnitts könnte der Besitzer der Tabellen in der Beispieldatenbank jemand anderem die Privilegien `SELECT`, `INSERT`, `UPDATE` und `DELETE` auf der View `shoelace` gewähren, aber nur `SELECT` auf `shoelace_log`. Die Regelaktion zum Schreiben von Log-Einträgen würde trotzdem erfolgreich ausgeführt, und der andere Benutzer könnte die Log-Einträge sehen. Er könnte aber keine gefälschten Einträge erzeugen und vorhandene Einträge weder manipulieren noch entfernen. In diesem Fall gibt es keine Möglichkeit, die Regeln zu unterlaufen, indem man den Planner dazu bringt, die Reihenfolge der Operationen zu verändern, weil die einzige Regel, die `shoelace_log` referenziert, ein unqualifiziertes `INSERT` ist. In komplexeren Szenarien muss das nicht zutreffen.

Wenn eine View Row-Level Security bereitstellen muss, sollte das Attribut `security_barrier` auf die View angewendet werden. Dadurch wird verhindert, dass böswillig gewählte Funktionen und Operatoren Werte aus Zeilen erhalten, bevor die View ihre Arbeit erledigt hat. Wenn die oben gezeigte View beispielsweise so erstellt worden wäre, wäre sie sicher:

```sql
CREATE VIEW phone_number WITH (security_barrier) AS
    SELECT person, phone FROM phone_data WHERE phone NOT LIKE
 '412%';
```

Views, die mit `security_barrier` erstellt wurden, können erheblich schlechter performen als Views ohne diese Option. Im Allgemeinen lässt sich das nicht vermeiden: Der schnellstmögliche Plan muss verworfen werden, wenn er die Sicherheit gefährden könnte. Aus diesem Grund ist diese Option standardmäßig nicht aktiviert.

Der Query Planner hat mehr Flexibilität im Umgang mit Funktionen, die keine Nebeneffekte haben. Solche Funktionen werden als `LEAKPROOF` bezeichnet; dazu gehören viele einfache, häufig verwendete Operatoren, etwa viele Gleichheitsoperatoren. Der Query Planner kann solche Funktionen sicher an jedem Punkt des Abfrageausführungsprozesses auswerten lassen, da ihr Aufruf auf für den Benutzer unsichtbaren Zeilen keine Informationen über diese unsichtbaren Zeilen preisgibt. Außerdem müssen Funktionen, die keine Argumente annehmen oder denen keine Argumente aus der Security-Barrier-View übergeben werden, nicht als `LEAKPROOF` markiert sein, um nach unten geschoben zu werden, da sie nie Daten aus der View erhalten. Im Gegensatz dazu ist eine Funktion, die abhängig von den als Argumenten empfangenen Werten einen Fehler auslösen könnte, etwa bei Überlauf oder Division durch null, nicht leakproof und könnte erhebliche Informationen über die unsichtbaren Zeilen liefern, wenn sie vor den Zeilenfiltern der Security-View angewendet wird.

Beispielsweise kann für Abfragen auf Security-Barrier-Views oder Tabellen mit Row-Level-Security-Policies kein Index-Scan gewählt werden, wenn ein in der `WHERE`-Klausel verwendeter Operator zur Operatorfamilie des Index gehört, seine zugrunde liegende Funktion aber nicht als `LEAKPROOF` markiert ist. Der Meta-Befehl `\dAo+` des Programms `psql` ist nützlich, um Operatorfamilien aufzulisten und festzustellen, welche ihrer Operatoren als leakproof markiert sind.

Es ist wichtig zu verstehen, dass selbst eine mit der Option `security_barrier` erstellte View nur in dem begrenzten Sinne sicher sein soll, dass die Inhalte unsichtbarer Tupel nicht an möglicherweise unsichere Funktionen übergeben werden. Der Benutzer kann durchaus andere Möglichkeiten haben, Rückschlüsse auf die unsichtbaren Daten zu ziehen. Zum Beispiel kann er den Abfrageplan mit `EXPLAIN` sehen oder die Laufzeit von Abfragen gegen die View messen. Ein böswilliger Angreifer könnte möglicherweise etwas über die Menge der unsichtbaren Daten ableiten oder sogar Informationen über die Datenverteilung oder häufigste Werte gewinnen, da solche Dinge die Laufzeit des Plans beeinflussen können oder sich in den Optimizer-Statistiken und damit in der Wahl des Plans widerspiegeln. Wenn solche „Covert Channel“-Angriffe ein Problem darstellen, ist es wahrscheinlich unklug, überhaupt Zugriff auf die Daten zu gewähren.

## 39.6. Regeln und Befehlsstatus

Der PostgreSQL-Server gibt für jeden empfangenen Befehl eine Befehlsstatus-Zeichenkette zurück, zum Beispiel `INSERT 149592 1`. Das ist einfach genug, solange keine Regeln beteiligt sind. Was passiert aber, wenn die Abfrage durch Regeln umgeschrieben wird?

Regeln beeinflussen den Befehlsstatus wie folgt:

- Wenn es für die Abfrage keine unbedingte `INSTEAD`-Regel gibt, wird die ursprünglich angegebene Abfrage ausgeführt, und ihr Befehlsstatus wird wie üblich zurückgegeben. Beachten Sie jedoch: Wenn es bedingte `INSTEAD`-Regeln gab, wurde die Negation ihrer Qualifikationen zur ursprünglichen Abfrage hinzugefügt. Das kann die Anzahl der verarbeiteten Zeilen reduzieren und damit auch den gemeldeten Status beeinflussen.

- Wenn es für die Abfrage irgendeine unbedingte `INSTEAD`-Regel gibt, wird die ursprüngliche Abfrage überhaupt nicht ausgeführt. In diesem Fall gibt der Server den Befehlsstatus für die letzte Abfrage zurück, die durch eine `INSTEAD`-Regel, bedingt oder unbedingt, eingefügt wurde und denselben Befehlstyp (`INSERT`, `UPDATE` oder `DELETE`) wie die ursprüngliche Abfrage hat. Wenn keine Abfrage, die diese Anforderungen erfüllt, durch irgendeine Regel hinzugefügt wird, zeigt der zurückgegebene Befehlsstatus den ursprünglichen Abfragetyp sowie Nullen für Zeilenzahl- und OID-Felder.

Der Programmierer kann im zweiten Fall sicherstellen, dass eine gewünschte `INSTEAD`-Regel den Befehlsstatus setzt, indem er ihr unter den aktiven Regeln den alphabetisch letzten Regelnamen gibt, sodass sie zuletzt angewendet wird.

## 39.7. Regeln versus Trigger

Viele Dinge, die mit Triggern erledigt werden können, lassen sich auch mit dem PostgreSQL-Regelsystem implementieren. Einige Arten von Constraints können jedoch nicht durch Regeln umgesetzt werden, insbesondere Fremdschlüssel. Es ist möglich, eine qualifizierte Regel zu definieren, die einen Befehl in `NOTHING` umschreibt, wenn der Wert einer Spalte in einer anderen Tabelle nicht vorkommt. Dann werden die Daten aber stillschweigend verworfen, und das ist keine gute Idee. Wenn gültige Werte geprüft werden müssen und bei einem ungültigen Wert eine Fehlermeldung erzeugt werden soll, muss dies mit einem Trigger geschehen.

In diesem Kapitel haben wir uns darauf konzentriert, Regeln zum Aktualisieren von Views zu verwenden. Alle Beispiele für Update-Regeln in diesem Kapitel können auch mit `INSTEAD OF`-Triggern auf den Views implementiert werden. Solche Trigger zu schreiben ist oft einfacher als Regeln zu schreiben, besonders wenn komplexe Logik für die Aktualisierung erforderlich ist.

Bei Dingen, die auf beide Arten implementiert werden können, hängt die bessere Wahl von der Nutzung der Datenbank ab. Ein Trigger wird einmal für jede betroffene Zeile ausgelöst. Eine Regel verändert die Abfrage oder erzeugt eine zusätzliche Abfrage. Wenn also viele Zeilen von einer Anweisung betroffen sind, ist eine Regel, die einen zusätzlichen Befehl ausgibt, wahrscheinlich schneller als ein Trigger, der für jede einzelne Zeile aufgerufen wird und immer wieder neu bestimmen muss, was zu tun ist. Der Trigger-Ansatz ist konzeptionell jedoch deutlich einfacher als der Regelansatz und für Einsteiger leichter korrekt umzusetzen.

Hier zeigen wir an einem Beispiel, wie sich die Wahl zwischen Regeln und Triggern in einer Situation auswirkt. Es gibt zwei Tabellen:

```sql
CREATE TABLE computer (
    hostname        text,                   -- indexed
    manufacturer    text                    -- indexed
);

CREATE TABLE software (
    software        text,                   -- indexed
    hostname        text                    -- indexed
);
```

Beide Tabellen haben viele tausend Zeilen, und die Indizes auf `hostname` sind eindeutig. Die Regel oder der Trigger soll einen Constraint implementieren, der Zeilen aus `software` löscht, die auf einen gelöschten `computer` verweisen. Der Trigger würde diesen Befehl verwenden:

```sql
DELETE FROM software WHERE hostname = $1;
```

Da der Trigger für jede einzelne aus `computer` gelöschte Zeile aufgerufen wird, kann er den Plan für diesen Befehl vorbereiten und speichern und den Wert von `hostname` als Parameter übergeben. Die Regel würde so geschrieben:

```sql
CREATE RULE computer_del AS ON DELETE TO computer
    DO DELETE FROM software WHERE hostname = OLD.hostname;
```

Nun betrachten wir verschiedene Arten von Löschungen. Im Fall von:

```sql
DELETE FROM computer WHERE hostname = 'mypc.local.net';
```

wird die Tabelle `computer` per Index gescannt, was schnell ist, und der vom Trigger ausgegebene Befehl würde ebenfalls einen Index-Scan verwenden, ebenfalls schnell. Der zusätzliche Befehl aus der Regel wäre:

```sql
DELETE FROM software WHERE computer.hostname = 'mypc.local.net'
                       AND software.hostname = computer.hostname;
```

Da geeignete Indizes eingerichtet sind, erzeugt der Planner einen Plan wie:

```text
Nestloop
  -> Index Scan using comp_hostidx on computer
  -> Index Scan using soft_hostidx on software
```

Zwischen Trigger- und Regelimplementierung gäbe es also keinen großen Geschwindigkeitsunterschied.

Mit der nächsten Löschung möchten wir alle 2000 Computer entfernen, deren Hostname mit `old` beginnt. Dafür gibt es zwei mögliche Befehle. Einer ist:

```sql
DELETE FROM computer WHERE hostname >= 'old'
                       AND hostname < 'ole'
```

Der durch die Regel hinzugefügte Befehl lautet:

```sql
DELETE FROM software WHERE computer.hostname >= 'old' AND
 computer.hostname < 'ole'
                       AND software.hostname = computer.hostname;
```

mit folgendem Plan:

```text
Hash Join
  -> Seq Scan on software
  -> Hash
    -> Index Scan using comp_hostidx on computer
```

Der andere mögliche Befehl ist:

```sql
DELETE FROM computer WHERE hostname ~ '^old';
```

Dies führt für den durch die Regel hinzugefügten Befehl zu folgendem Ausführungsplan:

```text
Nestloop
  -> Index Scan using comp_hostidx on computer
  -> Index Scan using soft_hostidx on software
```

Das zeigt, dass der Planner nicht erkennt, dass die Qualifikation für `hostname` in `computer` auch für einen Index-Scan auf `software` verwendet werden könnte, wenn mehrere Qualifikationsausdrücke mit `AND` kombiniert sind. Genau das tut er aber bei der Variante mit regulärem Ausdruck. Der Trigger wird einmal für jeden der 2000 alten Computer aufgerufen, die gelöscht werden müssen, was zu einem Index-Scan über `computer` und 2000 Index-Scans über `software` führt. Die Regelimplementierung erledigt dies mit zwei Befehlen, die Indizes verwenden. Ob die Regel in der Situation mit sequentiellem Scan noch schneller ist, hängt von der Gesamtgröße der Tabelle `software` ab. 2000 Befehlsausführungen durch den Trigger über den SPI-Manager kosten Zeit, selbst wenn bald alle Indexblöcke im Cache liegen.

Der letzte Befehl, den wir betrachten, ist:

```sql
DELETE FROM computer WHERE manufacturer = 'bim';
```

Auch dies könnte dazu führen, dass viele Zeilen aus `computer` gelöscht werden. Der Trigger würde also erneut viele Befehle durch den Executor ausführen. Der von der Regel erzeugte Befehl wäre:

```sql
DELETE FROM software WHERE computer.manufacturer = 'bim'
                       AND software.hostname = computer.hostname;
```

Der Plan für diesen Befehl ist wieder die Nested Loop über zwei Index-Scans, nur mit einem anderen Index auf `computer`:

```text
Nestloop
  -> Index Scan using comp_manufidx on computer
  -> Index Scan using soft_hostidx on software
```

In all diesen Fällen sind die zusätzlichen Befehle des Regelsystems mehr oder weniger unabhängig von der Anzahl der betroffenen Zeilen eines Befehls.

Zusammengefasst sind Regeln nur dann deutlich langsamer als Trigger, wenn ihre Aktionen zu großen, schlecht qualifizierten Joins führen, also in einer Situation, in der der Planner versagt.
