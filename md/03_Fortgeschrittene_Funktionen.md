# 3. Fortgeschrittene Funktionen

## 3.1. Einführung

Im vorigen Kapitel haben wir die Grundlagen behandelt, wie Sie SQL verwenden, um Ihre Daten in PostgreSQL zu speichern und darauf zuzugreifen. Nun besprechen wir einige fortgeschrittenere SQL-Funktionen, die die Verwaltung vereinfachen und den Verlust oder die Beschädigung Ihrer Daten verhindern. Abschließend betrachten wir einige PostgreSQL-Erweiterungen.

Dieses Kapitel verweist gelegentlich auf Beispiele aus [Kapitel 2](02_Die_SQL_Sprache.md), um sie zu ändern oder zu verbessern; es ist daher hilfreich, dieses Kapitel gelesen zu haben. Einige Beispiele aus diesem Kapitel befinden sich auch in `advanced.sql` im Tutorial-Verzeichnis. Diese Datei enthält außerdem einige Beispieldaten zum Laden, die hier nicht wiederholt werden. Wie Sie die Datei verwenden, steht in [Abschnitt 2.1](02_Die_SQL_Sprache.md#21-einführung).

## 3.2. Sichten

Kehren Sie zu den Abfragen in [Abschnitt 2.6](02_Die_SQL_Sprache.md#26-joins-zwischen-tabellen) zurück. Angenommen, die kombinierte Auflistung von Wetterdatensätzen und Stadtpositionen ist für Ihre Anwendung besonders interessant, Sie möchten die Abfrage aber nicht jedes Mal eintippen, wenn Sie sie benötigen. Sie können über der Abfrage eine View erstellen; damit geben Sie der Abfrage einen Namen, auf den Sie wie auf eine gewöhnliche Tabelle verweisen können:

```sql
CREATE VIEW myview AS
    SELECT name, temp_lo, temp_hi, prcp, date, location
        FROM weather, cities
        WHERE city = name;

SELECT * FROM myview;
```

Der großzügige Einsatz von Views ist ein zentraler Aspekt guten SQL-Datenbankdesigns. Views erlauben es, die Details der Struktur Ihrer Tabellen, die sich mit der Weiterentwicklung Ihrer Anwendung ändern können, hinter konsistenten Schnittstellen zu kapseln.

Views können fast überall dort verwendet werden, wo eine echte Tabelle verwendet werden kann. Es ist nicht ungewöhnlich, Views auf anderen Views aufzubauen.

## 3.3. Fremdschlüssel

Erinnern Sie sich an die Tabellen `weather` und `cities` aus [Kapitel 2](02_Die_SQL_Sprache.md). Betrachten Sie folgendes Problem: Sie möchten sicherstellen, dass niemand Zeilen in die Tabelle `weather` einfügen kann, zu denen es keinen passenden Eintrag in der Tabelle `cities` gibt. Das nennt man die referenzielle Integrität Ihrer Daten aufrechterhalten. In einfachen Datenbanksystemen würde dies, falls überhaupt, dadurch umgesetzt, dass man zunächst in der Tabelle `cities` nachsieht, ob ein passender Datensatz existiert, und die neuen Wetterdatensätze dann einfügt oder zurückweist. Dieser Ansatz hat mehrere Probleme und ist sehr unpraktisch; PostgreSQL kann das daher für Sie erledigen.

Die neue Deklaration der Tabellen sähe so aus:

```sql
CREATE TABLE cities (
        name     varchar(80) primary key,
        location point
);

CREATE TABLE weather (
        city      varchar(80) references cities(name),
        temp_lo   int,
        temp_hi   int,
        prcp      real,
        date      date
);
```

Versuchen Sie nun, einen ungültigen Datensatz einzufügen:

```sql
INSERT INTO weather VALUES ('Berkeley', 45, 53, 0.0, '1994-11-28');
```

```text
ERROR: insert or update on table "weather" violates foreign key
 constraint "weather_city_fkey"
DETAIL: Key (city)=(Berkeley) is not present in table "cities".
```

Das Verhalten von Fremdschlüsseln kann fein auf Ihre Anwendung abgestimmt werden. In diesem Tutorial gehen wir nicht über dieses einfache Beispiel hinaus, sondern verweisen für weitere Informationen auf [Kapitel 5](05_Datendefinition.md). Die korrekte Verwendung von Fremdschlüsseln verbessert die Qualität Ihrer Datenbankanwendungen deutlich; es wird daher dringend empfohlen, sich mit ihnen zu beschäftigen.

## 3.4. Transaktionen

Transaktionen sind ein grundlegendes Konzept aller Datenbanksysteme. Der wesentliche Punkt einer Transaktion ist, dass sie mehrere Schritte zu einer einzigen Alles-oder-nichts-Operation bündelt. Die Zwischenzustände zwischen den Schritten sind für andere gleichzeitig laufende Transaktionen nicht sichtbar, und wenn ein Fehler auftritt, der den Abschluss der Transaktion verhindert, wirkt sich keiner der Schritte überhaupt auf die Datenbank aus.

Betrachten Sie zum Beispiel eine Bankdatenbank, die Kontostände verschiedener Kundenkonten sowie die Gesamteinlagen von Filialen enthält. Angenommen, wir möchten eine Zahlung von 100,00 Dollar von Alices Konto auf Bobs Konto erfassen. Stark vereinfacht könnten die SQL-Befehle dafür so aussehen:

```sql
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
UPDATE branches SET balance = balance - 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name =
 'Alice');
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
UPDATE branches SET balance = balance + 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name =
 'Bob');
```

Die Details dieser Befehle sind hier nicht wichtig; entscheidend ist, dass an dieser recht einfachen Operation mehrere getrennte Aktualisierungen beteiligt sind. Die Verantwortlichen unserer Bank möchten sicher sein, dass entweder alle diese Aktualisierungen stattfinden oder keine von ihnen. Es wäre sicherlich untragbar, wenn ein Systemfehler dazu führte, dass Bob 100,00 Dollar erhält, ohne dass sie Alice belastet wurden. Ebenso bliebe Alice kaum lange eine zufriedene Kundin, wenn ihr Konto belastet würde, ohne dass Bob eine Gutschrift erhält. Wir brauchen die Garantie, dass, falls mitten in der Operation etwas schiefgeht, keiner der bisher ausgeführten Schritte wirksam wird. Die Zusammenfassung der Aktualisierungen in einer Transaktion gibt uns diese Garantie. Eine Transaktion gilt als atomar: Aus Sicht anderer Transaktionen geschieht sie entweder vollständig oder überhaupt nicht.

Wir möchten außerdem die Garantie, dass eine Transaktion, sobald sie abgeschlossen und vom Datenbanksystem bestätigt wurde, tatsächlich dauerhaft aufgezeichnet wurde und auch dann nicht verloren geht, wenn kurz darauf ein Absturz eintritt. Wenn wir beispielsweise eine Bargeldabhebung von Bob erfassen, darf keine Möglichkeit bestehen, dass die Belastung seines Kontos durch einen Absturz verschwindet, kurz nachdem er die Bank verlassen hat. Eine transaktionale Datenbank garantiert, dass alle von einer Transaktion vorgenommenen Aktualisierungen in dauerhaftem Speicher, also auf Platte, protokolliert werden, bevor die Transaktion als abgeschlossen gemeldet wird.

Eine weitere wichtige Eigenschaft transaktionaler Datenbanken hängt eng mit dem Begriff atomarer Aktualisierungen zusammen: Wenn mehrere Transaktionen gleichzeitig laufen, sollte keine von ihnen die unvollständigen Änderungen der anderen sehen können. Wenn beispielsweise eine Transaktion gerade die Salden aller Filialen summiert, wäre es falsch, die Belastung der Filiale von Alice einzubeziehen, aber nicht die Gutschrift der Filiale von Bob, oder umgekehrt. Transaktionen müssen also nicht nur hinsichtlich ihrer dauerhaften Wirkung auf die Datenbank Alles-oder-nichts sein, sondern auch hinsichtlich ihrer Sichtbarkeit während der Ausführung. Die bisher vorgenommenen Aktualisierungen einer offenen Transaktion sind für andere Transaktionen unsichtbar, bis die Transaktion abgeschlossen ist; dann werden alle Aktualisierungen gleichzeitig sichtbar.

In PostgreSQL wird eine Transaktion eingerichtet, indem die SQL-Befehle der Transaktion mit den Befehlen `BEGIN` und `COMMIT` umschlossen werden. Unsere Banktransaktion sähe also tatsächlich so aus:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
-- etc etc
COMMIT;
```

Wenn wir uns mitten in der Transaktion entscheiden, sie nicht festzuschreiben, vielleicht weil wir gerade bemerkt haben, dass Alices Kontostand negativ wurde, können wir statt `COMMIT` den Befehl `ROLLBACK` ausgeben; alle bisherigen Aktualisierungen werden dann verworfen.

PostgreSQL behandelt tatsächlich jede SQL-Anweisung so, als würde sie innerhalb einer Transaktion ausgeführt. Wenn Sie keinen `BEGIN`-Befehl ausgeben, wird jede einzelne Anweisung implizit von `BEGIN` und, bei Erfolg, `COMMIT` umschlossen. Eine Gruppe von Anweisungen, die von `BEGIN` und `COMMIT` umgeben ist, wird manchmal Transaktionsblock genannt.

> Hinweis: Einige Client-Bibliotheken geben `BEGIN`- und `COMMIT`-Befehle automatisch aus, sodass Sie die Wirkung von Transaktionsblöcken erhalten, ohne dies ausdrücklich anzufordern. Prüfen Sie die Dokumentation der Schnittstelle, die Sie verwenden.

Mit Savepoints lässt sich genauer steuern, welche Anweisungen in einer Transaktion erhalten bleiben. Savepoints erlauben es, Teile der Transaktion gezielt zu verwerfen, während der Rest festgeschrieben wird. Nachdem Sie mit `SAVEPOINT` einen Savepoint definiert haben, können Sie bei Bedarf mit `ROLLBACK TO` zu diesem Savepoint zurückkehren. Alle Datenbankänderungen der Transaktion zwischen der Definition des Savepoints und dem Zurückrollen zu ihm werden verworfen, Änderungen vor dem Savepoint bleiben erhalten.

Nach dem Zurückrollen zu einem Savepoint bleibt dieser weiterhin definiert, sodass Sie mehrmals zu ihm zurückrollen können. Umgekehrt können Sie einen Savepoint freigeben, wenn Sie sicher sind, ihn nicht erneut zu benötigen; das System kann dann einige Ressourcen freigeben. Beachten Sie, dass sowohl das Freigeben als auch das Zurückrollen zu einem Savepoint automatisch alle Savepoints freigibt, die danach definiert wurden.

All dies geschieht innerhalb des Transaktionsblocks, daher ist nichts davon für andere Datenbanksitzungen sichtbar. Wenn und sobald Sie den Transaktionsblock festschreiben, werden die festgeschriebenen Aktionen für andere Sitzungen als Einheit sichtbar, während die zurückgerollten Aktionen überhaupt nie sichtbar werden.

Denken wir wieder an die Bankdatenbank: Angenommen, wir belasten Alices Konto mit 100,00 Dollar und schreiben Bobs Konto den Betrag gut, stellen aber später fest, dass wir Wallys Konto hätten gutschreiben sollen. Mit Savepoints könnten wir das so tun:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
-- oops ... forget that and use Wally's account
ROLLBACK TO my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Wally';
COMMIT;
```

Dieses Beispiel ist natürlich stark vereinfacht, aber mit Savepoints ist innerhalb eines Transaktionsblocks sehr viel Kontrolle möglich. Außerdem ist `ROLLBACK TO` die einzige Möglichkeit, die Kontrolle über einen Transaktionsblock zurückzuerlangen, den das System aufgrund eines Fehlers in den abgebrochenen Zustand versetzt hat, abgesehen davon, ihn vollständig zurückzurollen und neu zu beginnen.

## 3.5. Window-Funktionen

Eine Window-Funktion führt eine Berechnung über eine Menge von Tabellenzeilen aus, die in irgendeiner Weise mit der aktuellen Zeile zusammenhängen. Das ist vergleichbar mit der Art von Berechnung, die mit einer Aggregatfunktion ausgeführt werden kann. Window-Funktionen bewirken jedoch nicht, dass Zeilen wie bei gewöhnlichen Aggregataufrufen ohne Window zu einer einzelnen Ausgabezeile gruppiert werden. Stattdessen behalten die Zeilen ihre eigene Identität. Im Hintergrund kann die Window-Funktion auf mehr als nur die aktuelle Zeile des Abfrageergebnisses zugreifen.

Das folgende Beispiel zeigt, wie das Gehalt jedes Mitarbeiters mit dem Durchschnittsgehalt seiner Abteilung verglichen werden kann:

```sql
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY
 depname) FROM empsalary;
```

```text
  depname | empno | salary |           avg
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
(10 rows)
```

Die ersten drei Ausgabespalten stammen direkt aus der Tabelle `empsalary`, und es gibt eine Ausgabezeile für jede Zeile der Tabelle. Die vierte Spalte stellt den Durchschnitt über alle Tabellenzeilen dar, die denselben `depname`-Wert wie die aktuelle Zeile haben. Tatsächlich ist dies dieselbe Funktion wie das gewöhnliche Aggregat `avg`, aber die `OVER`-Klausel bewirkt, dass sie als Window-Funktion behandelt und über den Window Frame berechnet wird.

Ein Aufruf einer Window-Funktion enthält immer eine `OVER`-Klausel direkt nach dem Namen und den Argumenten der Window-Funktion. Dadurch unterscheidet er sich syntaktisch von einer normalen Funktion oder einem gewöhnlichen Aggregat. Die `OVER`-Klausel bestimmt genau, wie die Zeilen der Abfrage für die Verarbeitung durch die Window-Funktion aufgeteilt werden. Die Klausel `PARTITION BY` innerhalb von `OVER` teilt die Zeilen in Gruppen oder Partitionen, die dieselben Werte der `PARTITION BY`-Ausdrücke besitzen. Für jede Zeile wird die Window-Funktion über die Zeilen berechnet, die in dieselbe Partition wie die aktuelle Zeile fallen.

Sie können außerdem mit `ORDER BY` innerhalb von `OVER` steuern, in welcher Reihenfolge Zeilen von Window-Funktionen verarbeitet werden. Das `ORDER BY` des Windows muss nicht einmal der Reihenfolge entsprechen, in der die Zeilen ausgegeben werden. Hier ist ein Beispiel:

```sql
SELECT depname, empno, salary,
       row_number() OVER (PARTITION BY depname ORDER BY salary
 DESC)
FROM empsalary;
```

```text
  depname | empno | salary | row_number
-----------+-------+--------+------------
 develop   |     8 |   6000 |          1
 develop   |    10 |   5200 |          2
 develop   |    11 |   5200 |          3
 develop   |     9 |   4500 |          4
 develop   |     7 |   4200 |          5
 personnel |     2 |   3900 |          1
 personnel |     5 |   3500 |          2
 sales     |     1 |   5000 |          1
 sales     |     4 |   4800 |          2
 sales     |     3 |   4800 |          3
(10 rows)
```

Wie hier gezeigt, weist die Window-Funktion `row_number` den Zeilen innerhalb jeder Partition fortlaufende Nummern zu, in der von der `ORDER BY`-Klausel festgelegten Reihenfolge; Zeilen mit gleichem Sortierwert werden in nicht näher bestimmter Reihenfolge nummeriert. `row_number` benötigt keinen ausdrücklichen Parameter, weil ihr Verhalten vollständig durch die `OVER`-Klausel bestimmt wird.

Die von einer Window-Funktion betrachteten Zeilen sind die der "virtuellen Tabelle", die von der `FROM`-Klausel der Abfrage erzeugt und gegebenenfalls durch ihre `WHERE`-, `GROUP BY`- und `HAVING`-Klauseln gefiltert wird. Eine Zeile, die entfernt wird, weil sie die `WHERE`-Bedingung nicht erfüllt, wird beispielsweise von keiner Window-Funktion gesehen. Eine Abfrage kann mehrere Window-Funktionen enthalten, die die Daten mit verschiedenen `OVER`-Klauseln unterschiedlich aufteilen; sie arbeiten aber alle auf derselben Zeilensammlung, die durch diese virtuelle Tabelle definiert wird.

Wir haben bereits gesehen, dass `ORDER BY` weggelassen werden kann, wenn die Reihenfolge der Zeilen nicht wichtig ist. Auch `PARTITION BY` kann weggelassen werden; in diesem Fall gibt es eine einzige Partition, die alle Zeilen enthält.

Mit Window-Funktionen ist ein weiteres wichtiges Konzept verbunden: Für jede Zeile gibt es innerhalb ihrer Partition eine Menge von Zeilen, den sogenannten Window Frame. Manche Window-Funktionen wirken nur auf die Zeilen des Window Frames und nicht auf die ganze Partition. Standardmäßig besteht der Frame, wenn `ORDER BY` angegeben ist, aus allen Zeilen vom Beginn der Partition bis einschließlich der aktuellen Zeile sowie allen folgenden Zeilen, die gemäß der `ORDER BY`-Klausel gleich der aktuellen Zeile sind. Wenn `ORDER BY` weggelassen wird, besteht der Standard-Frame aus allen Zeilen der Partition. Es gibt Optionen, den Window Frame auf andere Weise zu definieren; dieses Tutorial behandelt sie jedoch nicht. Details finden Sie in [Abschnitt 4.2.8](04_SQL_Syntax.md#428-aufrufe-von-windowfunktionen). Hier ist ein Beispiel mit `sum`:

```sql
SELECT salary, sum(salary) OVER () FROM empsalary;
```

```text
 salary | sum
--------+-------
   5200 | 47100
   5000 | 47100
   3500 | 47100
   4800 | 47100
   3900 | 47100
   4200 | 47100
   4500 | 47100
   4800 | 47100
   6000 | 47100
   5200 | 47100
(10 rows)
```

Da es oben keine `ORDER BY`-Klausel in der `OVER`-Klausel gibt, entspricht der Window Frame der Partition, und mangels `PARTITION BY` ist diese Partition die ganze Tabelle. Mit anderen Worten: Jede Summe wird über die ganze Tabelle gebildet, sodass wir für jede Ausgabezeile dasselbe Ergebnis erhalten. Wenn wir jedoch eine `ORDER BY`-Klausel hinzufügen, erhalten wir sehr andere Ergebnisse:

```sql
SELECT salary, sum(salary) OVER (ORDER BY salary) FROM empsalary;
```

```text
 salary | sum
--------+-------
   3500 | 3500
   3900 | 7400
   4200 | 11600
   4500 | 16100
   4800 | 25700
   4800 | 25700
   5000 | 30700
   5200 | 41100
   5200 | 41100
   6000 | 47100
(10 rows)
```

Hier wird die Summe vom ersten, also niedrigsten, Gehalt bis zum aktuellen Gehalt gebildet, einschließlich aller Duplikate des aktuellen Gehalts; beachten Sie die Ergebnisse bei den doppelten Gehältern.

Window-Funktionen sind nur in der `SELECT`-Liste und in der `ORDER BY`-Klausel der Abfrage erlaubt. An anderen Stellen, etwa in `GROUP BY`-, `HAVING`- und `WHERE`-Klauseln, sind sie verboten. Das liegt daran, dass sie logisch nach der Verarbeitung dieser Klauseln ausgeführt werden. Außerdem werden Window-Funktionen nach gewöhnlichen Aggregatfunktionen ausgeführt. Das bedeutet: Ein Aggregatfunktionsaufruf darf in den Argumenten einer Window-Funktion stehen, umgekehrt jedoch nicht.

Wenn Zeilen nach der Durchführung der Window-Berechnungen gefiltert oder gruppiert werden müssen, können Sie eine Unterauswahl verwenden. Zum Beispiel:

```sql
SELECT depname, empno, salary, enroll_date
FROM
  (SELECT depname, empno, salary, enroll_date,
     row_number() OVER (PARTITION BY depname ORDER BY salary DESC,
 empno) AS pos
     FROM empsalary
  ) AS ss
WHERE pos < 3;
```

Die obige Abfrage zeigt nur die Zeilen der inneren Abfrage, deren `row_number` kleiner als 3 ist, also die ersten beiden Zeilen jeder Abteilung.

Wenn eine Abfrage mehrere Window-Funktionen enthält, ist es möglich, jede mit einer eigenen `OVER`-Klausel auszuschreiben. Das ist jedoch redundant und fehleranfällig, wenn für mehrere Funktionen dasselbe Window-Verhalten gewünscht ist. Stattdessen kann jedes Window-Verhalten in einer `WINDOW`-Klausel benannt und dann in `OVER` referenziert werden. Zum Beispiel:

```sql
SELECT sum(salary) OVER w, avg(salary) OVER w
  FROM empsalary
  WINDOW w AS (PARTITION BY depname ORDER BY salary DESC);
```

Weitere Details zu Window-Funktionen finden Sie in [Abschnitt 4.2.8](04_SQL_Syntax.md#428-aufrufe-von-windowfunktionen), [Abschnitt 9.22](09_Funktionen_und_Operatoren.md#922-fensterfunktionen), [Abschnitt 7.2.5](07_Abfragen.md#725-verarbeitung-von-windowfunktionen) und auf der Referenzseite zu `SELECT`.

## 3.6. Vererbung

Vererbung ist ein Konzept aus objektorientierten Datenbanken. Sie eröffnet interessante neue Möglichkeiten für das Datenbankdesign.

Erstellen wir zwei Tabellen: eine Tabelle `cities` und eine Tabelle `capitals`. Hauptstädte sind natürlich ebenfalls Städte, daher möchten Sie eine Möglichkeit haben, Hauptstädte implizit mit anzuzeigen, wenn Sie alle Städte auflisten. Wenn Sie besonders erfinderisch sind, könnten Sie sich etwa folgendes Schema ausdenken:

```sql
CREATE TABLE capitals (
   name       text,
   population real,
   elevation int,     -- (in ft)
   state      char(2)
);

CREATE TABLE non_capitals (
   name       text,
   population real,
   elevation int     -- (in ft)
);

CREATE VIEW cities AS
  SELECT name, population, elevation FROM capitals
    UNION
  SELECT name, population, elevation FROM non_capitals;
```

Für Abfragen funktioniert das in Ordnung, wird aber unter anderem unschön, sobald Sie mehrere Zeilen aktualisieren müssen.

Eine bessere Lösung ist diese:

```sql
CREATE TABLE cities (
   name       text,
   population real,
   elevation int      -- (in ft)
);

CREATE TABLE capitals (
  state      char(2) UNIQUE NOT NULL
) INHERITS (cities);
```

In diesem Fall erbt eine Zeile von `capitals` alle Spalten (`name`, `population` und `elevation`) von ihrer Eltern-Tabelle `cities`. Der Typ der Spalte `name` ist `text`, ein nativer PostgreSQL-Typ für Zeichenketten variabler Länge. Die Tabelle `capitals` besitzt eine zusätzliche Spalte `state`, die das Kürzel des Bundesstaats zeigt. In PostgreSQL kann eine Tabelle von null oder mehr anderen Tabellen erben.

Die folgende Abfrage findet beispielsweise die Namen aller Städte, einschließlich Hauptstädten von Bundesstaaten, die auf einer Höhe über 500 Fuß liegen:

```sql
SELECT name, elevation
  FROM cities
  WHERE elevation > 500;
```

Das liefert:

```text
   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
 Madison   |       845
(3 rows)
```

Die folgende Abfrage findet dagegen alle Städte, die keine Hauptstädte von Bundesstaaten sind und auf einer Höhe über 500 Fuß liegen:

```sql
SELECT name, elevation
    FROM ONLY cities
    WHERE elevation > 500;
```

```text
   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
(2 rows)
```

Hier zeigt `ONLY` vor `cities` an, dass die Abfrage nur über die Tabelle `cities` ausgeführt werden soll und nicht über Tabellen unterhalb von `cities` in der Vererbungshierarchie. Viele der bereits besprochenen Befehle, darunter `SELECT`, `UPDATE` und `DELETE`, unterstützen diese `ONLY`-Notation.

> Hinweis: Obwohl Vererbung häufig nützlich ist, wurde sie nicht mit Unique Constraints oder Fremdschlüsseln integriert, was ihre Nützlichkeit einschränkt. Siehe [Abschnitt 5.11](05_Datendefinition.md#511-vererbung) für weitere Details.

## 3.7. Schluss

PostgreSQL besitzt viele Funktionen, die in dieser einführenden Tutorial-Darstellung, die sich an neuere SQL-Benutzer richtet, nicht berührt wurden. Diese Funktionen werden im weiteren Verlauf dieses Buchs ausführlicher behandelt.

Wenn Sie das Gefühl haben, mehr einführendes Material zu benötigen, besuchen Sie bitte die PostgreSQL-Webseite unter <https://www.postgresql.org>. Dort finden Sie Links zu weiteren Ressourcen.
