# 2. Die SQL-Sprache

## 2.1. Einführung

Dieses Kapitel gibt einen Überblick darüber, wie Sie SQL für einfache Operationen verwenden. Dieses Tutorial soll nur eine Einführung bieten und ist keinesfalls ein vollständiges SQL-Tutorial. Über SQL wurden zahlreiche Bücher geschrieben, darunter [melt93] und [date97]. Beachten Sie, dass einige Sprachfunktionen von PostgreSQL Erweiterungen des Standards sind.

In den folgenden Beispielen gehen wir davon aus, dass Sie, wie im vorigen Kapitel beschrieben, eine Datenbank namens `mydb` erstellt haben und `psql` starten konnten.

Die Beispiele in diesem Handbuch finden Sie auch in der PostgreSQL-Quelldistribution im Verzeichnis `src/tutorial/`. Binärdistributionen von PostgreSQL enthalten diese Dateien möglicherweise nicht. Um diese Dateien zu verwenden, wechseln Sie zuerst in dieses Verzeichnis und führen Sie `make` aus:

```console
$ cd .../src/tutorial
$ make
```

Dadurch werden die Skripte erzeugt und die C-Dateien kompiliert, die benutzerdefinierte Funktionen und Typen enthalten. Um das Tutorial zu starten, gehen Sie anschließend wie folgt vor:

```console
$ psql -s mydb

...

mydb=> \i basics.sql
```

Der Befehl `\i` liest Befehle aus der angegebenen Datei ein. Die Option `-s` von `psql` versetzt Sie in den Einzelschrittmodus, der vor dem Senden jeder Anweisung an den Server anhält. Die in diesem Abschnitt verwendeten Befehle stehen in der Datei `basics.sql`.

## 2.2. Konzepte

PostgreSQL ist ein relationales Datenbankmanagementsystem (RDBMS). Das bedeutet, es ist ein System zur Verwaltung von Daten, die in Relationen gespeichert sind. Relation ist im Wesentlichen ein mathematischer Begriff für Tabelle. Die Vorstellung, Daten in Tabellen zu speichern, ist heute so alltäglich, dass sie selbstverständlich erscheinen mag; es gibt jedoch eine Reihe anderer Arten, Datenbanken zu organisieren. Dateien und Verzeichnisse auf Unix-ähnlichen Betriebssystemen bilden ein Beispiel für eine hierarchische Datenbank. Eine modernere Entwicklung ist die objektorientierte Datenbank.

Jede Tabelle ist eine benannte Sammlung von Zeilen. Jede Zeile einer bestimmten Tabelle besitzt dieselbe Menge benannter Spalten, und jede Spalte hat einen bestimmten Datentyp. Während Spalten in jeder Zeile eine feste Reihenfolge haben, ist wichtig zu behalten, dass SQL die Reihenfolge der Zeilen innerhalb der Tabelle in keiner Weise garantiert, auch wenn sie für die Anzeige ausdrücklich sortiert werden können.

Tabellen werden zu Datenbanken zusammengefasst, und eine Sammlung von Datenbanken, die von einer einzelnen PostgreSQL-Serverinstanz verwaltet wird, bildet einen Datenbank-Cluster.

## 2.3. Eine neue Tabelle erstellen

Sie können eine neue Tabelle erstellen, indem Sie den Tabellennamen sowie alle Spaltennamen und ihre Typen angeben:

```sql
CREATE TABLE weather (
    city            varchar(80),
    temp_lo         int,                                -- low temperature
    temp_hi         int,                                -- high temperature
    prcp            real,                               -- precipitation
    date            date
);
```

Sie können dies mit den Zeilenumbrüchen in `psql` eingeben. `psql` erkennt, dass der Befehl erst mit dem Semikolon abgeschlossen ist.

Leerraum, also Leerzeichen, Tabulatoren und Zeilenumbrüche, kann in SQL-Befehlen frei verwendet werden. Das bedeutet, Sie können den Befehl anders ausrichten als oben oder sogar in eine einzige Zeile schreiben. Zwei Bindestriche (`--`) leiten Kommentare ein. Alles, was darauf folgt, wird bis zum Zeilenende ignoriert. SQL unterscheidet bei Schlüsselwörtern und Bezeichnern nicht zwischen Groß- und Kleinschreibung, außer wenn Bezeichner in doppelte Anführungszeichen gesetzt werden, um die Schreibweise zu erhalten; das wurde oben nicht getan.

`varchar(80)` gibt einen Datentyp an, der beliebige Zeichenketten mit bis zu 80 Zeichen speichern kann. `int` ist der normale Ganzzahltyp. `real` ist ein Typ zum Speichern von Gleitkommazahlen einfacher Genauigkeit. `date` sollte selbsterklärend sein. Ja, die Spalte vom Typ `date` heißt ebenfalls `date`. Das kann praktisch oder verwirrend sein; Sie entscheiden.

PostgreSQL unterstützt die Standard-SQL-Typen `int`, `smallint`, `real`, `double precision`, `char(N)`, `varchar(N)`, `date`, `time`, `timestamp` und `interval` sowie weitere allgemein nützliche Typen und einen umfangreichen Satz geometrischer Typen. PostgreSQL kann um beliebig viele benutzerdefinierte Datentypen erweitert werden. Folglich sind Typnamen in der Syntax keine Schlüsselwörter, außer dort, wo dies zur Unterstützung besonderer Fälle im SQL-Standard erforderlich ist.

Das zweite Beispiel speichert Städte und ihre zugehörige geografische Position:

```sql
CREATE TABLE cities (
    name            varchar(80),
    location        point
);
```

Der Typ `point` ist ein Beispiel für einen PostgreSQL-spezifischen Datentyp.

Abschließend sei erwähnt: Wenn Sie eine Tabelle nicht mehr benötigen oder sie anders neu erstellen möchten, können Sie sie mit folgendem Befehl entfernen:

```sql
DROP TABLE tablename;
```

## 2.4. Eine Tabelle mit Zeilen füllen

Die Anweisung `INSERT` wird verwendet, um eine Tabelle mit Zeilen zu füllen:

```sql
INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25,
 '1994-11-27');
```

Beachten Sie, dass alle Datentypen recht naheliegende Eingabeformate verwenden. Konstanten, die keine einfachen numerischen Werte sind, müssen gewöhnlich in einfache Anführungszeichen (`'`) eingeschlossen werden, wie im Beispiel. Der Typ `date` ist tatsächlich recht flexibel darin, was er akzeptiert; für dieses Tutorial bleiben wir jedoch bei dem hier gezeigten eindeutigen Format.

Der Typ `point` erwartet als Eingabe ein Koordinatenpaar, wie hier gezeigt:

```sql
INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');
```

Die bisher verwendete Syntax verlangt, dass Sie sich die Reihenfolge der Spalten merken. Eine alternative Syntax erlaubt es, die Spalten ausdrücklich aufzulisten:

```sql
INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
    VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');
```

Sie können die Spalten in einer anderen Reihenfolge aufführen oder sogar einige Spalten weglassen, zum Beispiel wenn der Niederschlag unbekannt ist:

```sql
INSERT INTO weather (date, city, temp_hi, temp_lo)
    VALUES ('1994-11-29', 'Hayward', 54, 37);
```

Viele Entwickler halten es für besseren Stil, die Spalten ausdrücklich aufzulisten, statt sich implizit auf ihre Reihenfolge zu verlassen.

Geben Sie bitte alle oben gezeigten Befehle ein, damit Sie für die folgenden Abschnitte einige Daten zum Arbeiten haben.

Sie hätten auch `COPY` verwenden können, um große Datenmengen aus einfachen Textdateien zu laden. Das ist normalerweise schneller, weil der Befehl `COPY` für diese Anwendung optimiert ist, bietet aber weniger Flexibilität als `INSERT`. Ein Beispiel wäre:

```sql
COPY weather FROM '/home/user/weather.txt';
```

Der Dateiname der Quelldatei muss auf der Maschine verfügbar sein, auf der der Backend-Prozess läuft, nicht auf dem Client, da der Backend-Prozess die Datei direkt liest. Die oben in die Tabelle `weather` eingefügten Daten könnten auch aus einer Datei eingefügt werden, die Folgendes enthält; die Werte sind durch Tabulatorzeichen getrennt:

```text
San Francisco                   46         50         0.25    1994-11-27
San Francisco                   43         57         0.0    1994-11-29
Hayward    37                   54         \N         1994-11-29
```

Mehr über den Befehl `COPY` erfahren Sie in der Dokumentation zu `COPY`.

## 2.5. Eine Tabelle abfragen

Um Daten aus einer Tabelle abzurufen, wird die Tabelle abgefragt. Dafür wird eine SQL-Anweisung `SELECT` verwendet. Die Anweisung ist in eine Auswahlliste, also den Teil mit den zurückzugebenden Spalten, eine Tabellenliste, also den Teil mit den Tabellen, aus denen die Daten abgerufen werden, und eine optionale Einschränkung gegliedert, die Bedingungen angibt. Um beispielsweise alle Zeilen der Tabelle `weather` abzurufen, geben Sie ein:

```sql
SELECT * FROM weather;
```

Hier ist `*` eine Kurzschreibweise für "alle Spalten". Dasselbe Ergebnis erhalten Sie also mit:

```sql
SELECT city, temp_lo, temp_hi, prcp, date FROM weather;
```

Die Ausgabe sollte so aussehen:

```text
     city      | temp_lo | temp_hi | prcp |    date
---------------+---------+---------+------+------------
 San Francisco |      46 |      50 | 0.25 | 1994-11-27
 San Francisco |      43 |      57 |    0 | 1994-11-29
 Hayward       |      37 |      54 |      | 1994-11-29
(3 rows)
```

Sie können in der Auswahlliste Ausdrücke schreiben, nicht nur einfache Spaltenverweise. Zum Beispiel:

```sql
SELECT city, (temp_hi+temp_lo)/2 AS temp_avg, date FROM weather;
```

Das sollte ergeben:

```text
     city      | temp_avg |    date
---------------+----------+------------
 San Francisco |       48 | 1994-11-27
 San Francisco |       50 | 1994-11-29
 Hayward       |       45 | 1994-11-29
(3 rows)
```

Beachten Sie, wie die `AS`-Klausel verwendet wird, um die Ausgabespalte umzubenennen. Die `AS`-Klausel ist optional.

Eine Abfrage kann durch Hinzufügen einer `WHERE`-Klausel eingeschränkt werden, die festlegt, welche Zeilen gewünscht sind. Die `WHERE`-Klausel enthält einen booleschen Ausdruck, also einen Wahrheitswertausdruck, und nur Zeilen, für die dieser Ausdruck wahr ist, werden zurückgegeben. Die üblichen booleschen Operatoren `AND`, `OR` und `NOT` sind in der Einschränkung erlaubt. Das folgende Beispiel ruft das Wetter von San Francisco an Regentagen ab:

```sql
SELECT * FROM weather
    WHERE city = 'San Francisco' AND prcp > 0.0;
```

Ergebnis:

```text
     city      | temp_lo | temp_hi | prcp |    date
---------------+---------+---------+------+------------
 San Francisco |      46 |      50 | 0.25 | 1994-11-27
(1 row)
```

Sie können verlangen, dass die Ergebnisse einer Abfrage sortiert zurückgegeben werden:

```sql
SELECT * FROM weather
    ORDER BY city;
```

```text
     city      | temp_lo | temp_hi | prcp |    date
---------------+---------+---------+------+------------
 Hayward       |      37 |      54 |      | 1994-11-29
 San Francisco |      43 |      57 |    0 | 1994-11-29
 San Francisco |      46 |      50 | 0.25 | 1994-11-27
```

In diesem Beispiel ist die Sortierreihenfolge nicht vollständig festgelegt, sodass die Zeilen für San Francisco in beliebiger Reihenfolge erscheinen könnten. Das oben gezeigte Ergebnis erhalten Sie jedoch immer, wenn Sie Folgendes tun:

```sql
SELECT * FROM weather
    ORDER BY city, temp_lo;
```

Sie können verlangen, dass doppelte Zeilen aus dem Ergebnis einer Abfrage entfernt werden:

```sql
SELECT DISTINCT city
    FROM weather;
```

```text
     city
---------------
 Hayward
 San Francisco
(2 rows)
```

Auch hier kann die Reihenfolge der Ergebniszeilen variieren. Konsistente Ergebnisse erreichen Sie, indem Sie `DISTINCT` und `ORDER BY` gemeinsam verwenden:

```sql
SELECT DISTINCT city
    FROM weather
    ORDER BY city;
```

## 2.6. Joins zwischen Tabellen

Bisher haben unsere Abfragen jeweils nur auf eine Tabelle zugegriffen. Abfragen können aber auch auf mehrere Tabellen gleichzeitig zugreifen oder dieselbe Tabelle so verwenden, dass mehrere Zeilen der Tabelle gleichzeitig verarbeitet werden. Abfragen, die gleichzeitig auf mehrere Tabellen oder mehrere Instanzen derselben Tabelle zugreifen, heißen Join-Abfragen. Sie kombinieren Zeilen aus einer Tabelle mit Zeilen aus einer zweiten Tabelle, wobei ein Ausdruck angibt, welche Zeilen gepaart werden sollen. Um beispielsweise alle Wetterdatensätze zusammen mit der Position der zugehörigen Stadt zurückzugeben, muss die Datenbank die Spalte `city` jeder Zeile der Tabelle `weather` mit der Spalte `name` aller Zeilen in der Tabelle `cities` vergleichen und diejenigen Zeilenpaare auswählen, bei denen diese Werte übereinstimmen. Dies wird mit folgender Abfrage erreicht:

```sql
SELECT * FROM weather JOIN cities ON city = name;
```

```text
     city      | temp_lo | temp_hi | prcp |    date     |    name      | location
---------------+---------+---------+------+-------------+--------------+-----------
 San Francisco |      46 |      50 | 0.25 | 1994-11-27  | San Francisco | (-194,53)
 San Francisco |      43 |      57 |    0 | 1994-11-29  | San Francisco | (-194,53)
(2 rows)
```

Beachten Sie zwei Dinge an der Ergebnismenge:

- Für die Stadt Hayward gibt es keine Ergebniszeile. Das liegt daran, dass es in der Tabelle `cities` keinen passenden Eintrag für Hayward gibt; der Join ignoriert daher die nicht passenden Zeilen in der Tabelle `weather`. Wir werden gleich sehen, wie sich das beheben lässt.
- Es gibt zwei Spalten, die den Stadtnamen enthalten. Das ist korrekt, weil die Spaltenlisten der Tabellen `weather` und `cities` aneinandergehängt werden. In der Praxis ist das jedoch unerwünscht; Sie werden daher vermutlich die Ausgabespalten ausdrücklich auflisten wollen, statt `*` zu verwenden:

```sql
SELECT city, temp_lo, temp_hi, prcp, date, location
    FROM weather JOIN cities ON city = name;
```

Da die Spalten alle unterschiedliche Namen hatten, konnte der Parser automatisch feststellen, zu welcher Tabelle sie gehören. Wenn es in beiden Tabellen doppelte Spaltennamen gäbe, müssten Sie die Spaltennamen qualifizieren, um zu zeigen, welche Spalte Sie meinen, etwa so:

```sql
SELECT weather.city, weather.temp_lo, weather.temp_hi,
       weather.prcp, weather.date, cities.location
    FROM weather JOIN cities ON weather.city = cities.name;
```

Es gilt weithin als guter Stil, in einer Join-Abfrage alle Spaltennamen zu qualifizieren, damit die Abfrage nicht fehlschlägt, wenn später einer der Tabellen ein gleichnamiger Spaltenname hinzugefügt wird.

Join-Abfragen der bisher gesehenen Art können auch in dieser Form geschrieben werden:

```sql
SELECT *
    FROM weather, cities
    WHERE city = name;
```

Diese Syntax ist älter als die in SQL-92 eingeführte `JOIN`/`ON`-Syntax. Die Tabellen werden einfach in der `FROM`-Klausel aufgeführt, und der Vergleichsausdruck wird der `WHERE`-Klausel hinzugefügt. Die Ergebnisse der älteren impliziten Syntax und der neueren expliziten `JOIN`/`ON`-Syntax sind identisch. Für Leser der Abfrage macht die explizite Syntax die Bedeutung jedoch leichter verständlich: Die Join-Bedingung wird durch ein eigenes Schlüsselwort eingeleitet, während sie früher zusammen mit anderen Bedingungen in die `WHERE`-Klausel gemischt war.

Nun wollen wir herausfinden, wie wir die Hayward-Datensätze zurückbekommen. Die Abfrage soll die Tabelle `weather` durchlaufen und für jede Zeile die passenden Zeilen aus `cities` finden. Wenn keine passende Zeile gefunden wird, sollen für die Spalten der Tabelle `cities` einige "leere Werte" eingesetzt werden. Diese Art von Abfrage heißt Outer Join. Die bisher gesehenen Joins waren Inner Joins. Der Befehl sieht so aus:

```sql
SELECT *
    FROM weather LEFT OUTER JOIN cities ON weather.city =
 cities.name;
```

```text
     city      | temp_lo | temp_hi | prcp |    date     |    name      | location
---------------+---------+---------+------+-------------+--------------+-----------
 Hayward       |      37 |      54 |      | 1994-11-29  |              |
 San Francisco |      46 |      50 | 0.25 | 1994-11-27  | San Francisco | (-194,53)
 San Francisco |      43 |      57 |    0 | 1994-11-29  | San Francisco | (-194,53)
(3 rows)
```

Diese Abfrage heißt Left Outer Join, weil die links vom Join-Operator genannte Tabelle jede ihrer Zeilen mindestens einmal in der Ausgabe hat, während von der rechts stehenden Tabelle nur diejenigen Zeilen ausgegeben werden, die zu irgendeiner Zeile der linken Tabelle passen. Wenn eine Zeile der linken Tabelle ausgegeben wird, zu der es keine passende Zeile der rechten Tabelle gibt, werden für die Spalten der rechten Tabelle leere Werte (`NULL`) eingesetzt.

Übung: Es gibt auch Right Outer Joins und Full Outer Joins. Versuchen Sie herauszufinden, was diese tun.

Wir können eine Tabelle auch mit sich selbst verbinden. Das nennt man Self Join. Angenommen, wir möchten alle Wetterdatensätze finden, die im Temperaturbereich anderer Wetterdatensätze liegen. Dazu müssen wir die Spalten `temp_lo` und `temp_hi` jeder Wetterzeile mit den Spalten `temp_lo` und `temp_hi` aller anderen Wetterzeilen vergleichen. Das geht mit folgender Abfrage:

```sql
SELECT w1.city, w1.temp_lo AS low, w1.temp_hi AS high,
       w2.city, w2.temp_lo AS low, w2.temp_hi AS high
    FROM weather w1 JOIN weather w2
        ON w1.temp_lo < w2.temp_lo AND w1.temp_hi > w2.temp_hi;
```

```text
     city      | low | high |     city      | low | high
---------------+-----+------+---------------+-----+------
 San Francisco |  43 |   57 | San Francisco |  46 |   50
 Hayward       |  37 |   54 | San Francisco |  46 |   50
(2 rows)
```

Hier haben wir die Tabelle `weather` in `w1` und `w2` umbenannt, um die linke und rechte Seite des Joins unterscheiden zu können. Solche Aliasse können Sie auch in anderen Abfragen verwenden, um Tipparbeit zu sparen, zum Beispiel:

```sql
SELECT *
    FROM weather w JOIN cities c ON w.city = c.name;
```

Dieser Abkürzungsstil wird Ihnen häufig begegnen.

## 2.7. Aggregatfunktionen

Wie die meisten anderen relationalen Datenbankprodukte unterstützt PostgreSQL Aggregatfunktionen. Eine Aggregatfunktion berechnet ein einzelnes Ergebnis aus mehreren Eingabezeilen. Es gibt zum Beispiel Aggregate zur Berechnung von `count`, `sum`, `avg` (Durchschnitt), `max` (Maximum) und `min` (Minimum) über eine Menge von Zeilen.

Als Beispiel können wir den höchsten gemessenen Tiefsttemperaturwert überhaupt finden:

```sql
SELECT max(temp_lo) FROM weather;
```

```text
 max
-----
(1 row)
```

Wenn wir wissen wollten, in welcher Stadt oder welchen Städten dieser Wert auftrat, könnten wir Folgendes versuchen:

```sql
SELECT city FROM weather WHERE temp_lo = max(temp_lo); -- WRONG
```

Das funktioniert jedoch nicht, weil das Aggregat `max` nicht in der `WHERE`-Klausel verwendet werden kann. Diese Einschränkung besteht, weil die `WHERE`-Klausel bestimmt, welche Zeilen in die Aggregatberechnung eingehen; sie muss also offensichtlich ausgewertet werden, bevor Aggregatfunktionen berechnet werden. Wie so oft kann die Abfrage jedoch umformuliert werden, um das gewünschte Ergebnis zu erreichen, hier mit einer Unterabfrage:

```sql
SELECT city FROM weather
    WHERE temp_lo = (SELECT max(temp_lo) FROM weather);
```

```text
     city
---------------
 San Francisco
(1 row)
```

Das ist zulässig, weil die Unterabfrage eine unabhängige Berechnung ist, die ihr eigenes Aggregat getrennt von der äußeren Abfrage berechnet.

Aggregate sind auch sehr nützlich in Kombination mit `GROUP BY`-Klauseln. Zum Beispiel können wir die Anzahl der Messungen und die höchste beobachtete Tiefsttemperatur je Stadt abrufen:

```sql
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city;
```

```text
     city      | count | max
---------------+-------+-----
 Hayward       |     1 | 37
 San Francisco |     2 | 46
(2 rows)
```

Das ergibt eine Ausgabezeile pro Stadt. Jedes Aggregatergebnis wird über die Tabellenzeilen berechnet, die zu dieser Stadt gehören. Diese gruppierten Zeilen können wir mit `HAVING` filtern:

```sql
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city
    HAVING max(temp_lo) < 40;
```

```text
  city   | count | max
---------+-------+-----
 Hayward |     1 | 37
(1 row)
```

Das liefert dasselbe Ergebnis nur für die Städte, bei denen alle `temp_lo`-Werte unter 40 liegen. Wenn uns schließlich nur Städte interessieren, deren Namen mit "S" beginnen, könnten wir Folgendes tun:

```sql
SELECT city, count(*), max(temp_lo)
    FROM weather
    WHERE city LIKE 'S%'            -- 1
    GROUP BY city;
```

```text
     city      | count | max
---------------+-------+-----
 San Francisco |     2 | 46
(1 row)
```

1. Der Operator `LIKE` führt Mustervergleiche durch und wird in [Abschnitt 9.7](09_Funktionen_und_Operatoren.md#97-mustervergleich) erklärt.

Es ist wichtig, das Zusammenspiel zwischen Aggregaten und den SQL-Klauseln `WHERE` und `HAVING` zu verstehen. Der grundlegende Unterschied zwischen `WHERE` und `HAVING` ist folgender: `WHERE` wählt Eingabezeilen aus, bevor Gruppen und Aggregate berechnet werden; es steuert also, welche Zeilen in die Aggregatberechnung eingehen. `HAVING` wählt dagegen Gruppenzeilen aus, nachdem Gruppen und Aggregate berechnet wurden. Daher darf die `WHERE`-Klausel keine Aggregatfunktionen enthalten; es ergibt keinen Sinn, mit einem Aggregat zu bestimmen, welche Zeilen Eingaben für die Aggregate sein sollen. Die `HAVING`-Klausel enthält dagegen immer Aggregatfunktionen. Streng genommen dürfen Sie eine `HAVING`-Klausel schreiben, die keine Aggregate verwendet, aber das ist selten nützlich. Dieselbe Bedingung könnte effizienter in der `WHERE`-Phase angewendet werden.

Im vorigen Beispiel können wir die Einschränkung des Stadtnamens in `WHERE` anwenden, da sie kein Aggregat benötigt. Das ist effizienter, als die Einschränkung in `HAVING` aufzunehmen, weil wir Gruppierung und Aggregatberechnungen für alle Zeilen vermeiden, die die `WHERE`-Prüfung nicht bestehen.

Eine weitere Möglichkeit, die Zeilen auszuwählen, die in eine Aggregatberechnung eingehen, ist `FILTER`, eine Option pro Aggregat:

```sql
SELECT city, count(*) FILTER (WHERE temp_lo < 45), max(temp_lo)
    FROM weather
    GROUP BY city;
```

```text
     city      | count | max
---------------+-------+-----
 Hayward       |     1 | 37
 San Francisco |     1 | 46
(2 rows)
```

`FILTER` ähnelt `WHERE`, entfernt Zeilen aber nur aus der Eingabe derjenigen Aggregatfunktion, an die es angehängt ist. Hier zählt das Aggregat `count` nur Zeilen mit `temp_lo` unter 45; das Aggregat `max` wird aber weiterhin auf alle Zeilen angewendet und findet daher weiterhin den Messwert 46.

## 2.8. Aktualisierungen

Vorhandene Zeilen können Sie mit dem Befehl `UPDATE` aktualisieren. Angenommen, Sie stellen fest, dass alle Temperaturmesswerte nach dem 28. November um 2 Grad abweichen. Sie können die Daten wie folgt korrigieren:

```sql
UPDATE weather
    SET temp_hi = temp_hi - 2, temp_lo = temp_lo - 2
    WHERE date > '1994-11-28';
```

Sehen Sie sich den neuen Zustand der Daten an:

```sql
SELECT * FROM weather;
```

```text
     city      | temp_lo | temp_hi | prcp |    date
---------------+---------+---------+------+------------
 San Francisco |      46 |      50 | 0.25 | 1994-11-27
 San Francisco |      41 |      55 |    0 | 1994-11-29
 Hayward       |      35 |      52 |      | 1994-11-29
(3 rows)
```

## 2.9. Löschungen

Zeilen können mit dem Befehl `DELETE` aus einer Tabelle entfernt werden. Angenommen, Sie interessieren sich nicht mehr für das Wetter in Hayward. Dann können Sie die entsprechenden Zeilen so aus der Tabelle löschen:

```sql
DELETE FROM weather WHERE city = 'Hayward';
```

Alle zu Hayward gehörenden Wetterdatensätze werden entfernt.

```sql
SELECT * FROM weather;
```

```text
     city      | temp_lo | temp_hi | prcp |    date
---------------+---------+---------+------+------------
 San Francisco |      46 |      50 | 0.25 | 1994-11-27
 San Francisco |      41 |      55 |    0 | 1994-11-29
(2 rows)
```

Vor Anweisungen der folgenden Form sollte man sich hüten:

```sql
DELETE FROM tablename;
```

Ohne Einschränkung entfernt `DELETE` alle Zeilen aus der angegebenen Tabelle und lässt sie leer zurück. Das System fragt vor dieser Aktion nicht nach einer Bestätigung!
