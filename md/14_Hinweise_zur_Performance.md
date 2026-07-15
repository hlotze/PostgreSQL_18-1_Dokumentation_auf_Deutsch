# 14. Hinweise zur Performance

Die Performance von Abfragen kann von vielen Dingen beeinflusst werden. Einige davon können Benutzer steuern, andere ergeben sich aus dem grundlegenden Design des Systems. Dieses Kapitel gibt Hinweise dazu, wie sich die Performance von PostgreSQL verstehen und verbessern lässt.

## 14.1. EXPLAIN verwenden

PostgreSQL erstellt für jede empfangene Abfrage einen Abfrageplan. Für gute Performance ist es entscheidend, einen Plan zu wählen, der zur Struktur der Abfrage und zu den Eigenschaften der Daten passt. Deshalb enthält das System einen komplexen Planner, der versucht, gute Pläne auszuwählen. Mit dem Befehl `EXPLAIN` können Sie sehen, welchen Abfrageplan der Planner für eine Abfrage erzeugt. Das Lesen von Plänen ist eine Kunst, für die man etwas Erfahrung braucht; dieser Abschnitt behandelt die Grundlagen.

Die Beispiele in diesem Abschnitt stammen aus der Regressionstest-Datenbank nach einem `VACUUM ANALYZE`, unter Verwendung der Entwicklungsquellen von Version 18. Wenn Sie die Beispiele selbst ausprobieren, sollten Sie ähnliche Ergebnisse erhalten. Geschätzte Kosten und Zeilenzahlen können jedoch leicht abweichen, weil die Statistiken von `ANALYZE` zufällige Stichproben und keine exakten Werte sind und weil Kosten grundsätzlich etwas plattformabhängig sind.

Die Beispiele verwenden das standardmäßige Textausgabeformat von `EXPLAIN`, das kompakt und für Menschen gut lesbar ist. Wenn Sie die Ausgabe von `EXPLAIN` zur weiteren Analyse an ein Programm übergeben wollen, sollten Sie stattdessen eines der maschinenlesbaren Ausgabeformate verwenden: XML, JSON oder YAML.

### 14.1.1. EXPLAIN-Grundlagen

Die Struktur eines Abfrageplans ist ein Baum aus Planknoten. Die Knoten auf der untersten Ebene sind Scan-Knoten: Sie liefern rohe Zeilen aus einer Tabelle zurück. Für verschiedene Tabellenzugriffsmethoden gibt es verschiedene Arten von Scan-Knoten, etwa sequenzielle Scans, Index-Scans und Bitmap-Index-Scans. Es gibt auch Zeilenquellen, die keine Tabellen sind, zum Beispiel `VALUES`-Klauseln und mengenliefernde Funktionen in `FROM`; sie haben eigene Scan-Knotentypen. Wenn die Abfrage Joins, Aggregation, Sortierung oder andere Operationen auf den rohen Zeilen benötigt, gibt es oberhalb der Scan-Knoten zusätzliche Knoten, die diese Operationen ausführen. Auch dafür gibt es meistens mehr als eine mögliche Vorgehensweise, sodass auch hier unterschiedliche Knotentypen auftreten können.

Die Ausgabe von `EXPLAIN` enthält für jeden Knoten im Planbaum eine Zeile. Sie zeigt den grundlegenden Knotentyp und die Kostenschätzungen, die der Planner für die Ausführung dieses Planknotens vorgenommen hat. Weitere eingerückte Zeilen können zusätzliche Eigenschaften des Knotens anzeigen. Die erste Zeile, also die Zusammenfassungszeile des obersten Knotens, enthält die geschätzten Gesamtausführungskosten des Plans; diese Zahl versucht der Planner zu minimieren.

Ein triviales Beispiel zeigt das Ausgabeformat:

```sql
EXPLAIN SELECT * FROM tenk1;
```

```text
                         QUERY PLAN
-------------------------------------------------------------
 Seq Scan on tenk1 (cost=0.00..445.00 rows=10000 width=244)
```

Da diese Abfrage keine `WHERE`-Klausel hat, muss sie alle Zeilen der Tabelle scannen. Der Planner hat deshalb einen einfachen sequenziellen Scan gewählt. Die in Klammern angegebenen Zahlen bedeuten von links nach rechts:

- Geschätzte Startkosten. Das ist die Zeit, die anfällt, bevor die Ausgabephase beginnen kann, zum Beispiel die Zeit für eine Sortierung in einem Sort-Knoten.
- Geschätzte Gesamtkosten. Diese Angabe geht davon aus, dass der Planknoten vollständig ausgeführt wird, also alle verfügbaren Zeilen abgerufen werden. In der Praxis kann der übergeordnete Knoten eines Knotens aufhören, bevor alle verfügbaren Zeilen gelesen wurden; siehe das `LIMIT`-Beispiel weiter unten.
- Geschätzte Anzahl der von diesem Planknoten ausgegebenen Zeilen. Auch hier wird angenommen, dass der Knoten vollständig ausgeführt wird.
- Geschätzte durchschnittliche Breite der von diesem Planknoten ausgegebenen Zeilen in Bytes.

Die Kosten werden in willkürlichen Einheiten gemessen, die durch die Kostenparameter des Planners bestimmt werden; siehe [Abschnitt 19.7.2](19_Serverkonfiguration.md#1972-plannerkostenkonstanten). Traditionell werden Kosten in Einheiten von Disk-Page-Zugriffen gemessen; `seq_page_cost` wird also üblicherweise auf `1.0` gesetzt, und die anderen Kostenparameter werden relativ dazu angegeben. Die Beispiele in diesem Abschnitt laufen mit den Standardkostenparametern.

Wichtig ist, dass die Kosten eines übergeordneten Knotens die Kosten aller seiner Kindknoten enthalten. Ebenso wichtig ist, dass die Kosten nur Dinge widerspiegeln, die der Planner berücksichtigen kann. Insbesondere berücksichtigen sie nicht die Zeit, die dafür benötigt wird, Ausgabewerte in Textform umzuwandeln oder sie an den Client zu übertragen. Diese Faktoren können für die tatsächlich verstrichene Zeit wichtig sein, aber der Planner ignoriert sie, weil er sie durch eine Änderung des Plans nicht beeinflussen kann. Jeder korrekte Plan liefert schließlich dieselbe Ergebnismenge.

Der Wert `rows` ist etwas subtil: Er bezeichnet nicht die Zahl der vom Planknoten verarbeiteten oder gescannten Zeilen, sondern die Zahl der vom Knoten ausgegebenen Zeilen. Das ist oft weniger als die gescannte Zahl, weil am Knoten `WHERE`-Bedingungen als Filter angewendet werden. Idealerweise nähert die Schätzung auf oberster Ebene die Anzahl der Zeilen an, die von der Abfrage tatsächlich zurückgegeben, aktualisiert oder gelöscht werden.

Zurück zum Beispiel:

```sql
EXPLAIN SELECT * FROM tenk1;
```

```text
                         QUERY PLAN
-------------------------------------------------------------
 Seq Scan on tenk1 (cost=0.00..445.00 rows=10000 width=244)
```

Diese Zahlen ergeben sich sehr direkt. Wenn Sie Folgendes ausführen:

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'tenk1';
```

dann sehen Sie, dass `tenk1` 345 Plattenseiten und 10000 Zeilen hat. Die geschätzten Kosten werden berechnet als `(gelesene Plattenseiten * seq_page_cost) + (gescannte Zeilen * cpu_tuple_cost)`. Standardmäßig ist `seq_page_cost` gleich `1.0` und `cpu_tuple_cost` gleich `0.01`, also ergibt sich `(345 * 1.0) + (10000 * 0.01) = 445`.

Nun fügen wir der Abfrage eine `WHERE`-Bedingung hinzu:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 7000;
```

```text
                         QUERY PLAN
------------------------------------------------------------
 Seq Scan on tenk1 (cost=0.00..470.00 rows=7000 width=244)
   Filter: (unique1 < 7000)
```

Die Ausgabe von `EXPLAIN` zeigt, dass die `WHERE`-Klausel als Filterbedingung am Plan-Knoten `Seq Scan` angewendet wird. Das bedeutet, dass der Planknoten die Bedingung für jede gescannte Zeile prüft und nur die Zeilen ausgibt, die sie erfüllen. Die Schätzung der Ausgabezeilen wurde wegen der `WHERE`-Klausel reduziert. Der Scan muss jedoch weiterhin alle 10000 Zeilen besuchen, daher sind die Kosten nicht gesunken. Sie sind sogar etwas gestiegen, genau um `10000 * cpu_operator_cost`, um die zusätzliche CPU-Zeit für die Prüfung der `WHERE`-Bedingung abzubilden.

Die tatsächliche Anzahl der von dieser Abfrage ausgewählten Zeilen beträgt 7000, aber die `rows`-Schätzung ist nur näherungsweise. Wenn Sie dieses Experiment wiederholen, erhalten Sie möglicherweise eine leicht andere Schätzung. Außerdem kann sie sich nach jedem `ANALYZE` ändern, weil die von `ANALYZE` erzeugten Statistiken aus einer zufälligen Stichprobe der Tabelle stammen.

Nun machen wir die Bedingung restriktiver:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100;
```

```text
                                  QUERY PLAN
-------------------------------------------------------------------
 Bitmap Heap Scan on tenk1 (cost=5.06..224.98 rows=100 width=244)
   Recheck Cond: (unique1 < 100)
   -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0)
         Index Cond: (unique1 < 100)
```

Hier hat der Planner einen zweistufigen Plan gewählt: Der untergeordnete Planknoten besucht einen Index, um die Positionen der Zeilen zu finden, die zur Indexbedingung passen; der darüberliegende Planknoten holt diese Zeilen anschließend aus der Tabelle selbst. Einzelne Zeilen abzurufen ist viel teurer, als sie sequenziell zu lesen. Weil aber nicht alle Seiten der Tabelle besucht werden müssen, ist dies immer noch billiger als ein sequenzieller Scan. Zwei Planebenen werden verwendet, weil der obere Planknoten die vom Index identifizierten Zeilenpositionen vor dem Lesen in physische Reihenfolge sortiert, um die Kosten der einzelnen Abrufe zu minimieren. Das im Knotennamen erwähnte „Bitmap“ ist der Mechanismus, der diese Sortierung ermöglicht.

Nun fügen wir der `WHERE`-Klausel eine weitere Bedingung hinzu:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND stringu1 = 'xxx';
```

```text
                                  QUERY PLAN
-------------------------------------------------------------------
 Bitmap Heap Scan on tenk1 (cost=5.04..225.20 rows=1 width=244)
   Recheck Cond: (unique1 < 100)
   Filter: (stringu1 = 'xxx'::name)
   -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0)
         Index Cond: (unique1 < 100)
```

Die zusätzliche Bedingung `stringu1 = 'xxx'` reduziert die geschätzte Anzahl ausgegebener Zeilen, aber nicht die Kosten, denn es muss immer noch dieselbe Menge von Zeilen besucht werden. Die `stringu1`-Klausel kann nicht als Indexbedingung angewendet werden, weil dieser Index nur auf der Spalte `unique1` liegt. Stattdessen wird sie als Filter auf die über den Index abgerufenen Zeilen angewendet. Die Kosten sind daher sogar leicht gestiegen, um diese zusätzliche Prüfung abzubilden.

In manchen Fällen bevorzugt der Planner einen einfachen Index-Scan:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 = 42;
```

```text
                                 QUERY PLAN
-------------------------------------------------------------------
 Index Scan using tenk1_unique1 on tenk1 (cost=0.29..8.30 rows=1 width=244)
   Index Cond: (unique1 = 42)
```

Bei diesem Plantyp werden Tabellenzeilen in Indexreihenfolge geholt. Das macht das Lesen noch teurer, aber hier sind es so wenige Zeilen, dass sich die zusätzlichen Kosten für das Sortieren der Zeilenpositionen nicht lohnen. Diesen Plantyp sieht man am häufigsten bei Abfragen, die nur eine einzelne Zeile abrufen. Er wird auch oft für Abfragen verwendet, deren `ORDER BY`-Bedingung zur Indexreihenfolge passt, weil dann kein zusätzlicher Sortierschritt nötig ist. In diesem Beispiel würde `ORDER BY unique1` denselben Plan verwenden, weil der Index die gewünschte Ordnung bereits implizit bereitstellt.

Der Planner kann eine `ORDER BY`-Klausel auf mehrere Arten umsetzen. Das vorherige Beispiel zeigt, dass eine solche Ordnungsklausel implizit erfüllt werden kann. Der Planner kann auch einen expliziten `Sort`-Schritt hinzufügen:

```sql
EXPLAIN SELECT * FROM tenk1 ORDER BY unique1;
```

```text
                            QUERY PLAN
-------------------------------------------------------------------
 Sort (cost=1109.39..1134.39 rows=10000 width=244)
   Sort Key: unique1
   -> Seq Scan on tenk1 (cost=0.00..445.00 rows=10000 width=244)
```

Wenn ein Teil des Plans eine Ordnung auf einem Präfix der benötigten Sortierschlüssel garantiert, kann der Planner stattdessen einen `Incremental Sort` verwenden:

```sql
EXPLAIN SELECT * FROM tenk1 ORDER BY hundred, ten LIMIT 100;
```

```text
                                              QUERY PLAN
--------------------------------------------------------------------------------
 Limit (cost=19.35..39.49 rows=100 width=244)
   -> Incremental Sort (cost=19.35..2033.39 rows=10000 width=244)
         Sort Key: hundred, ten
         Presorted Key: hundred
         -> Index Scan using tenk1_hundred on tenk1 (cost=0.29..1574.20 rows=10000 width=244)
```

Im Vergleich zu normalen Sortierungen erlaubt inkrementelles Sortieren, Tupel zurückzugeben, bevor die gesamte Ergebnismenge sortiert wurde. Das ermöglicht insbesondere Optimierungen für `LIMIT`-Abfragen. Es kann außerdem den Speicherverbrauch und die Wahrscheinlichkeit verringern, dass Sortierungen auf die Platte ausgelagert werden müssen. Dafür entsteht zusätzlicher Aufwand, weil die Ergebnismenge in mehrere Sortierchargen aufgeteilt wird.

Wenn es für mehrere in `WHERE` referenzierte Spalten separate Indizes gibt, kann der Planner eine `AND`- oder `OR`-Kombination dieser Indizes wählen:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;
```

```text
                                     QUERY PLAN
--------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1 (cost=25.07..60.11 rows=10 width=244)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   -> BitmapAnd (cost=25.07..25.07 rows=10 width=0)
         -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0)
               Index Cond: (unique1 < 100)
         -> Bitmap Index Scan on tenk1_unique2 (cost=0.00..19.78 rows=999 width=0)
               Index Cond: (unique2 > 9000)
```

Das erfordert jedoch den Besuch beider Indizes und ist daher nicht zwangsläufig besser, als nur einen Index zu verwenden und die andere Bedingung als Filter zu behandeln. Wenn Sie die beteiligten Wertebereiche verändern, sehen Sie, wie sich der Plan entsprechend ändert.

Dieses Beispiel zeigt die Wirkung von `LIMIT`:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000
 LIMIT 2;
```

```text
                                      QUERY PLAN
--------------------------------------------------------------------------------
 Limit (cost=0.29..14.28 rows=2 width=244)
   -> Index Scan using tenk1_unique2 on tenk1 (cost=0.29..70.27 rows=10 width=244)
         Index Cond: (unique2 > 9000)
         Filter: (unique1 < 100)
```

Dies ist dieselbe Abfrage wie zuvor, aber mit einem `LIMIT`, sodass nicht alle Zeilen abgerufen werden müssen. Der Planner hat deshalb eine andere Entscheidung getroffen. Die Gesamtkosten und die Zeilenzahl des `Index Scan`-Knotens werden so angezeigt, als würde er vollständig ausgeführt. Der `Limit`-Knoten wird aber erwartungsgemäß nach einem Fünftel dieser Zeilen stoppen, daher betragen seine Gesamtkosten nur ein Fünftel, und das ist die geschätzte tatsächliche Abfragekostenzahl. Dieser Plan wird gegenüber dem vorherigen Plan plus `Limit` bevorzugt, weil `Limit` die Startkosten des Bitmap-Scans nicht vermeiden könnte; die Gesamtkosten lägen dann über 25 Einheiten.

Nun verbinden wir zwei Tabellen über die bisher verwendeten Spalten:

```sql
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;
```

```text
                                      QUERY PLAN
--------------------------------------------------------------------------------
 Nested Loop (cost=4.65..118.50 rows=10 width=488)
   -> Bitmap Heap Scan on tenk1 t1 (cost=4.36..39.38 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   -> Index Scan using tenk2_unique2 on tenk2 t2 (cost=0.29..7.90 rows=1 width=244)
         Index Cond: (unique2 = t1.unique2)
```

In diesem Plan gibt es einen Nested-Loop-Join-Knoten mit zwei Tabellenscans als Eingaben, also als Kindknoten. Die Einrückung der Zusammenfassungszeilen spiegelt die Struktur des Planbaums wider. Das erste, äußere Kind des Joins ist ein Bitmap-Scan ähnlich den vorherigen Beispielen. Seine Kosten und Zeilenzahlen entsprechen denen von `SELECT ... WHERE unique1 < 10`, weil die `WHERE`-Klausel `unique1 < 10` an diesem Knoten angewendet wird. Die Klausel `t1.unique2 = t2.unique2` ist dort noch nicht relevant und beeinflusst daher die Zeilenzahl des äußeren Scans nicht. Der Nested-Loop-Join-Knoten führt sein zweites, inneres Kind einmal für jede Zeile aus, die er vom äußeren Kind erhält. Spaltenwerte aus der aktuellen äußeren Zeile können in den inneren Scan eingesetzt werden; hier steht der Wert `t1.unique2` aus der äußeren Zeile zur Verfügung. Dadurch ergibt sich ein Plan mit Kosten ähnlich einem einfachen `SELECT ... WHERE t2.unique2 = constant`. Die Kosten des Loop-Knotens basieren dann auf den Kosten des äußeren Scans, plus einer Wiederholung des inneren Scans für jede äußere Zeile, hier `10 * 7.90`, plus etwas CPU-Zeit für die Join-Verarbeitung.

In diesem Beispiel entspricht die Ausgabezeilenzahl des Joins dem Produkt der Zeilenzahlen der beiden Scans. Das gilt aber nicht immer, weil zusätzliche `WHERE`-Klauseln existieren können, die beide Tabellen erwähnen und daher erst am Join-Punkt angewendet werden können, nicht an einem der Eingabescans. Beispiel:

```sql
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t2.unique2 < 10 AND t1.hundred < t2.hundred;
```

```text
                                          QUERY PLAN
--------------------------------------------------------------------------------
 Nested Loop (cost=4.65..49.36 rows=33 width=488)
   Join Filter: (t1.hundred < t2.hundred)
   -> Bitmap Heap Scan on tenk1 t1 (cost=4.36..39.38 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   -> Materialize (cost=0.29..8.51 rows=10 width=244)
         -> Index Scan using tenk2_unique2 on tenk2 t2 (cost=0.29..8.46 rows=10 width=244)
               Index Cond: (unique2 < 10)
```

Die Bedingung `t1.hundred < t2.hundred` kann im Index `tenk2_unique2` nicht geprüft werden und wird daher am Join-Knoten angewendet. Das reduziert die geschätzte Ausgabezeilenzahl des Join-Knotens, ändert aber keinen der beiden Eingabescans.

Beachten Sie, dass der Planner hier die innere Relation des Joins materialisiert, indem er einen `Materialize`-Planknoten darüber legt. Das bedeutet, dass der `t2`-Indexscan nur einmal ausgeführt wird, obwohl der Nested-Loop-Join-Knoten diese Daten zehnmal lesen muss, einmal für jede Zeile der äußeren Relation. Der `Materialize`-Knoten speichert die Daten beim Lesen im Speicher und liefert sie bei jedem weiteren Durchlauf aus dem Speicher zurück.

Bei Outer Joins können Join-Planknoten sowohl Bedingungen vom Typ `Join Filter` als auch einfache `Filter`-Bedingungen besitzen. `Join Filter`-Bedingungen stammen aus der `ON`-Klausel des Outer Joins; eine Zeile, die diese Bedingung nicht erfüllt, kann deshalb trotzdem als mit Nullwerten aufgefüllte Zeile ausgegeben werden. Ein einfacher `Filter` wird nach den Outer-Join-Regeln angewendet und entfernt Zeilen daher unbedingt. Bei Inner Joins gibt es zwischen diesen Filterarten keinen semantischen Unterschied.

Wenn wir die Selektivität der Abfrage etwas ändern, kann ein sehr anderer Join-Plan entstehen:

```sql
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;
```

```text
                                        QUERY PLAN
--------------------------------------------------------------------------------
 Hash Join (cost=226.23..709.73 rows=100 width=488)
   Hash Cond: (t2.unique2 = t1.unique2)
   -> Seq Scan on tenk2 t2 (cost=0.00..445.00 rows=10000 width=244)
   -> Hash (cost=224.98..224.98 rows=100 width=244)
         -> Bitmap Heap Scan on tenk1 t1 (cost=5.06..224.98 rows=100 width=244)
               Recheck Cond: (unique1 < 100)
               -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0)
                     Index Cond: (unique1 < 100)
```

Hier hat der Planner einen Hash Join gewählt. Dabei werden die Zeilen einer Tabelle in eine Hash-Tabelle im Speicher eingetragen; anschließend wird die andere Tabelle gescannt, und die Hash-Tabelle wird für jede Zeile nach passenden Einträgen durchsucht. Auch hier zeigt die Einrückung die Planstruktur: Der Bitmap-Scan auf `tenk1` ist die Eingabe für den `Hash`-Knoten, der die Hash-Tabelle aufbaut. Diese wird anschließend an den `Hash Join`-Knoten zurückgegeben, der Zeilen aus seinem äußeren Kindplan liest und für jede davon die Hash-Tabelle durchsucht.

Ein weiterer möglicher Join-Typ ist der Merge Join:

```sql
EXPLAIN SELECT *
FROM tenk1 t1, onek t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;
```

```text
                                         QUERY PLAN
--------------------------------------------------------------------------------
 Merge Join (cost=0.56..233.49 rows=10 width=488)
   Merge Cond: (t1.unique2 = t2.unique2)
   -> Index Scan using tenk1_unique2 on tenk1 t1 (cost=0.29..643.28 rows=100 width=244)
         Filter: (unique1 < 100)
   -> Index Scan using onek_unique2 on onek t2 (cost=0.28..166.28 rows=1000 width=244)
```

Ein Merge Join setzt voraus, dass seine Eingabedaten nach den Join-Schlüsseln sortiert sind. In diesem Beispiel wird jede Eingabe durch einen Index-Scan in der richtigen Reihenfolge gelesen. Es könnten aber auch ein sequenzieller Scan und eine Sortierung verwendet werden. Ein sequenzieller Scan mit anschließender Sortierung ist beim Sortieren vieler Zeilen häufig schneller als ein Index-Scan, weil der Index-Scan nichtsequenziellen Plattenzugriff benötigt.

Eine Möglichkeit, alternative Pläne zu betrachten, besteht darin, den Planner mit den in [Abschnitt 19.7.1](19_Serverkonfiguration.md#1971-plannermethodenkonfiguration) beschriebenen Ein-/Aus-Schaltern dazu zu bringen, eine Strategie zu meiden, die er für am billigsten hält. Das ist ein grobes, aber nützliches Werkzeug; siehe auch [Abschnitt 14.3](#143-den-planner-mit-expliziten-joinklauseln-steuern). Wenn wir zum Beispiel nicht überzeugt sind, dass Merge Join im vorherigen Beispiel der beste Join-Typ ist, können wir Folgendes versuchen:

```sql
SET enable_mergejoin = off;

EXPLAIN SELECT *
FROM tenk1 t1, onek t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;
```

```text
                                        QUERY PLAN
--------------------------------------------------------------------------------
 Hash Join (cost=226.23..344.08 rows=10 width=488)
   Hash Cond: (t2.unique2 = t1.unique2)
   -> Seq Scan on onek t2 (cost=0.00..114.00 rows=1000 width=244)
   -> Hash (cost=224.98..224.98 rows=100 width=244)
         -> Bitmap Heap Scan on tenk1 t1 (cost=5.06..224.98 rows=100 width=244)
               Recheck Cond: (unique1 < 100)
               -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0)
                     Index Cond: (unique1 < 100)
```

Das zeigt, dass der Planner einen Hash Join in diesem Fall für fast 50 Prozent teurer hält als den Merge Join. Die nächste Frage ist natürlich, ob er damit richtig liegt. Das lässt sich mit `EXPLAIN ANALYZE` untersuchen, wie weiter unten beschrieben.

Wenn Ein-/Aus-Schalter verwendet werden, um Planknotentypen zu deaktivieren, halten viele dieser Schalter den Planner nur von der Verwendung des entsprechenden Planknotens ab; sie verbieten ihn nicht absolut. Das ist beabsichtigt, damit der Planner weiterhin einen Plan für eine gegebene Abfrage bilden kann. Wenn der resultierende Plan einen deaktivierten Knoten enthält, zeigt die `EXPLAIN`-Ausgabe dies an:

```sql
SET enable_seqscan = off;
EXPLAIN SELECT * FROM unit;
```

```text
                       QUERY PLAN
---------------------------------------------------------
 Seq Scan on unit (cost=0.00..21.30 rows=1130 width=44)
   Disabled: true
```

Da die Tabelle `unit` keine Indizes hat, gibt es keine andere Möglichkeit, die Tabellendaten zu lesen. Der sequenzielle Scan ist daher die einzige verfügbare Option für den Query Planner.

Manche Abfragepläne enthalten Subpläne, die aus Sub-`SELECT`s der ursprünglichen Abfrage entstehen. Solche Abfragen können manchmal in normale Join-Pläne umgewandelt werden; wenn das nicht möglich ist, entstehen Pläne wie dieser:

```sql
EXPLAIN VERBOSE SELECT unique1
FROM tenk1 t
WHERE t.ten < ALL (SELECT o.ten FROM onek o WHERE o.four = t.four);
```

```text
                               QUERY PLAN
-------------------------------------------------------------------
 Seq Scan on public.tenk1 t (cost=0.00..586095.00 rows=5000 width=4)
   Output: t.unique1
   Filter: (ALL (t.ten < (SubPlan 1).col1))
   SubPlan 1
     -> Seq Scan on public.onek o (cost=0.00..116.50 rows=250 width=4)
           Output: o.ten
           Filter: (o.four = t.four)
```

Dieses künstliche Beispiel illustriert zwei Punkte: Werte aus der äußeren Planebene können in einen Subplan hinuntergereicht werden, hier `t.four`, und die Ergebnisse des Sub-SELECTs stehen dem äußeren Plan zur Verfügung. Diese Ergebniswerte zeigt `EXPLAIN` mit Notationen wie `(SubPlan 1).col1`, die sich auf die erste Ausgabespalte des Sub-`SELECT`s beziehen.

Im obigen Beispiel führt der Operator `ALL` den Subplan für jede Zeile der äußeren Abfrage erneut aus, was die hohen geschätzten Kosten erklärt. Manche Abfragen können einen gehashten Subplan verwenden, um das zu vermeiden:

```sql
EXPLAIN SELECT *
FROM tenk1 t
WHERE t.unique1 NOT IN (SELECT o.unique1 FROM onek o);
```

```text
                                         QUERY PLAN
--------------------------------------------------------------------------------
 Seq Scan on tenk1 t (cost=61.77..531.77 rows=5000 width=244)
   Filter: (NOT (ANY (unique1 = (hashed SubPlan 1).col1)))
   SubPlan 1
     -> Index Only Scan using onek_unique1 on onek o (cost=0.28..59.27 rows=1000 width=4)
(4 rows)
```

Hier wird der Subplan nur einmal ausgeführt und seine Ausgabe in eine Hash-Tabelle im Speicher geladen. Der äußere `ANY`-Operator prüft anschließend diese Hash-Tabelle. Das setzt voraus, dass das Sub-`SELECT` keine Variablen der äußeren Abfrage referenziert und dass der Vergleichsoperator von `ANY` für Hashing geeignet ist.

Wenn das Sub-`SELECT` außerdem höchstens eine Zeile zurückgeben kann, kann es stattdessen als Initplan umgesetzt werden:

```sql
EXPLAIN VERBOSE SELECT unique1
FROM tenk1 t1 WHERE t1.ten = (SELECT (random() * 10)::integer);
```

```text
                                  QUERY PLAN
--------------------------------------------------------------------
 Seq Scan on public.tenk1 t1 (cost=0.02..470.02 rows=1000 width=4)
   Output: t1.unique1
   Filter: (t1.ten = (InitPlan 1).col1)
   InitPlan 1
     -> Result (cost=0.00..0.02 rows=1 width=4)
           Output: ((random() * '10'::double precision))::integer
```

Ein Initplan wird pro Ausführung des äußeren Plans nur einmal ausgeführt, und seine Ergebnisse werden für spätere Zeilen des äußeren Plans gespeichert. In diesem Beispiel wird `random()` also nur einmal ausgewertet, und alle Werte von `t1.ten` werden mit derselben zufällig gewählten Ganzzahl verglichen. Das unterscheidet sich deutlich von dem, was ohne die Sub-`SELECT`-Konstruktion passieren würde.

### 14.1.2. EXPLAIN ANALYZE

Mit der Option `ANALYZE` von `EXPLAIN` lässt sich prüfen, wie genau die Schätzungen des Planners sind. Mit dieser Option führt `EXPLAIN` die Abfrage tatsächlich aus und zeigt anschließend die wirklichen Zeilenzahlen und die in jedem Planknoten akkumulierte tatsächliche Laufzeit an, zusammen mit denselben Schätzungen, die ein normales `EXPLAIN` zeigt. Zum Beispiel:

```sql
EXPLAIN ANALYZE SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;
```

```text
Nested Loop (cost=4.65..118.50 rows=10 width=488) (actual time=0.017..0.051 rows=10.00 loops=1)
  Buffers: shared hit=36 read=6
  -> Bitmap Heap Scan on tenk1 t1 (cost=4.36..39.38 rows=10 width=244) (actual time=0.009..0.017 rows=10.00 loops=1)
        Recheck Cond: (unique1 < 10)
        Heap Blocks: exact=10
        Buffers: shared hit=3 read=5 written=4
        -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..4.36 rows=10 width=0) (actual time=0.004..0.004 rows=10.00 loops=1)
              Index Cond: (unique1 < 10)
              Index Searches: 1
              Buffers: shared hit=2
  -> Index Scan using tenk2_unique2 on tenk2 t2 (cost=0.29..7.90 rows=1 width=244) (actual time=0.003..0.003 rows=1.00 loops=10)
        Index Cond: (unique2 = t1.unique2)
        Index Searches: 10
        Buffers: shared hit=24 read=6
Planning:
  Buffers: shared hit=15 dirtied=9
Planning Time: 0.485 ms
Execution Time: 0.073 ms
```

Die Werte für `actual time` sind reale Millisekunden, während die Kostenschätzungen in willkürlichen Einheiten ausgedrückt werden; sie werden daher kaum übereinstimmen. Meist ist am wichtigsten, ob die geschätzten Zeilenzahlen einigermaßen nah an der Realität liegen. In diesem Beispiel sind die Schätzungen exakt, was in der Praxis eher ungewöhnlich ist.

In manchen Abfrageplänen kann ein Subplanknoten mehr als einmal ausgeführt werden. Im obigen Nested-Loop-Plan wird etwa der innere Indexscan einmal pro äußerer Zeile ausgeführt. In solchen Fällen gibt der Wert `loops` die Gesamtzahl der Knotenausführungen an; die angezeigten Werte für `actual time` und `rows` sind Durchschnittswerte pro Ausführung. Dadurch bleiben sie mit der Darstellung der Kostenschätzungen vergleichbar. Multiplizieren Sie mit `loops`, um die tatsächlich im Knoten verbrachte Gesamtzeit zu erhalten. Im obigen Beispiel wurden insgesamt 0,030 Millisekunden mit Indexscans auf `tenk2` verbracht.

In manchen Fällen zeigt `EXPLAIN ANALYZE` neben Ausführungszeiten und Zeilenzahlen weitere Ausführungsstatistiken. `Sort`- und `Hash`-Knoten liefern zum Beispiel zusätzliche Informationen:

```sql
EXPLAIN ANALYZE SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2
ORDER BY t1.fivethous;
```

```text
 Sort (cost=713.05..713.30 rows=100 width=488) (actual time=2.995..3.002 rows=100.00 loops=1)
   Sort Key: t1.fivethous
   Sort Method: quicksort Memory: 74kB
   Buffers: shared hit=440
   -> Hash Join (cost=226.23..709.73 rows=100 width=488) (actual time=0.515..2.920 rows=100.00 loops=1)
         Hash Cond: (t2.unique2 = t1.unique2)
         Buffers: shared hit=437
         -> Seq Scan on tenk2 t2 (cost=0.00..445.00 rows=10000 width=244) (actual time=0.026..1.790 rows=10000.00 loops=1)
               Buffers: shared hit=345
         -> Hash (cost=224.98..224.98 rows=100 width=244) (actual time=0.476..0.477 rows=100.00 loops=1)
               Buckets: 1024 Batches: 1 Memory Usage: 35kB
               Buffers: shared hit=92
               -> Bitmap Heap Scan on tenk1 t1 (cost=5.06..224.98 rows=100 width=244) (actual time=0.030..0.450 rows=100.00 loops=1)
                      Recheck Cond: (unique1 < 100)
                      Heap Blocks: exact=90
                      Buffers: shared hit=92
                      -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0) (actual time=0.013..0.013 rows=100.00 loops=1)
                            Index Cond: (unique1 < 100)
                            Index Searches: 1
                            Buffers: shared hit=2
 Planning:
   Buffers: shared hit=12
 Planning Time: 0.187 ms
 Execution Time: 3.036 ms
```

Der `Sort`-Knoten zeigt die verwendete Sortiermethode, insbesondere ob im Speicher oder auf der Platte sortiert wurde, und den benötigten Speicher- oder Plattenplatz. Der `Hash`-Knoten zeigt die Anzahl der Hash-Buckets und Batches sowie die maximale Speichernutzung für die Hash-Tabelle. Wenn die Anzahl der Batches größer als eins ist, ist auch Plattenspeicher beteiligt; dieser wird dort jedoch nicht angezeigt.

`Index Scan`-Knoten sowie `Bitmap Index Scan`- und `Index Only Scan`-Knoten zeigen eine Zeile `Index Searches`, die die Gesamtzahl der Suchen über alle Knotenausführungen und Schleifen hinweg angibt:

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE thousand IN (1, 500, 700, 999);
```

```text
 Bitmap Heap Scan on tenk1 (cost=9.45..73.44 rows=40 width=244) (actual time=0.012..0.028 rows=40.00 loops=1)
   Recheck Cond: (thousand = ANY ('{1,500,700,999}'::integer[]))
   Heap Blocks: exact=39
   Buffers: shared hit=47
   -> Bitmap Index Scan on tenk1_thous_tenthous (cost=0.00..9.44 rows=40 width=0) (actual time=0.009..0.009 rows=40.00 loops=1)
         Index Cond: (thousand = ANY ('{1,500,700,999}'::integer[]))
         Index Searches: 4
         Buffers: shared hit=8
 Planning Time: 0.029 ms
 Execution Time: 0.034 ms
```

Hier sehen wir einen `Bitmap Index Scan`, der vier separate Indexsuchen benötigte. Der Scan musste den Index von der Root Page von `tenk1_thous_tenthous` aus einmal pro Ganzzahlwert aus dem `IN`-Prädikat durchsuchen. Die Zahl der Indexsuchen entspricht jedoch oft nicht so einfach dem Abfrageprädikat:

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE thousand IN (1, 2, 3, 4);
```

```text
 Bitmap Heap Scan on tenk1 (cost=9.45..73.44 rows=40 width=244) (actual time=0.009..0.019 rows=40.00 loops=1)
   Recheck Cond: (thousand = ANY ('{1,2,3,4}'::integer[]))
   Heap Blocks: exact=38
   Buffers: shared hit=40
   -> Bitmap Index Scan on tenk1_thous_tenthous (cost=0.00..9.44 rows=40 width=0) (actual time=0.005..0.005 rows=40.00 loops=1)
         Index Cond: (thousand = ANY ('{1,2,3,4}'::integer[]))
         Index Searches: 1
         Buffers: shared hit=2
 Planning Time: 0.029 ms
 Execution Time: 0.026 ms
```

Diese Variante der `IN`-Abfrage führte nur eine Indexsuche aus. Sie verbrachte weniger Zeit mit dem Durchlaufen des Index, weil ihr `IN`-Konstrukt Werte verwendet, die zu Index-Tupeln passen, die nebeneinander auf derselben Blattseite des Index `tenk1_thous_tenthous` gespeichert sind.

Die Zeile `Index Searches` ist auch bei B-Tree-Indexscans nützlich, die die Skip-Scan-Optimierung verwenden, um einen Index effizienter zu durchlaufen:

```sql
EXPLAIN ANALYZE SELECT four, unique1 FROM tenk1
WHERE four BETWEEN 1 AND 3 AND unique1 = 42;
```

```text
 Index Only Scan using tenk1_four_unique1_idx on tenk1 (cost=0.29..6.90 rows=1 width=8) (actual time=0.006..0.007 rows=1.00 loops=1)
   Index Cond: ((four >= 1) AND (four <= 3) AND (unique1 = 42))
   Heap Fetches: 0
   Index Searches: 3
   Buffers: shared hit=7
 Planning Time: 0.029 ms
 Execution Time: 0.012 ms
```

Hier verwendet ein `Index Only Scan` den Index `tenk1_four_unique1_idx`, einen Mehrspaltenindex auf den Spalten `four` und `unique1` der Tabelle `tenk1`. Der Scan führt drei Suchen aus, die jeweils eine einzelne Index-Leaf-Page lesen: `four = 1 AND unique1 = 42`, `four = 2 AND unique1 = 42` und `four = 3 AND unique1 = 42`. Dieser Index ist im Allgemeinen ein guter Kandidat für Skip Scan, weil seine führende Spalte `four` nur vier unterschiedliche Werte enthält, während die zweite und letzte Spalte `unique1` viele unterschiedliche Werte enthält; siehe [Abschnitt 11.3](11_Indizes.md#113-mehrspaltige-indizes).

Eine weitere Art zusätzlicher Information ist die Zahl der durch eine Filterbedingung entfernten Zeilen:

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE ten < 7;
```

```text
 Seq Scan on tenk1 (cost=0.00..470.00 rows=7000 width=244) (actual time=0.030..1.995 rows=7000.00 loops=1)
   Filter: (ten < 7)
   Rows Removed by Filter: 3000
   Buffers: shared hit=345
 Planning Time: 0.102 ms
 Execution Time: 2.145 ms
```

Diese Zählungen können besonders wertvoll sein, wenn Filterbedingungen an Join-Knoten angewendet werden. Die Zeile `Rows Removed` erscheint nur, wenn mindestens eine gescannte Zeile oder, bei einem Join-Knoten, ein mögliches Join-Paar durch die Filterbedingung verworfen wurde.

Ein ähnlicher Fall tritt bei verlustbehafteten Indexscans auf. Betrachten Sie diese Suche nach Polygonen, die einen bestimmten Punkt enthalten:

```sql
EXPLAIN ANALYZE SELECT * FROM polygon_tbl WHERE f1 @> polygon '(0.5,2.0)';
```

```text
 Seq Scan on polygon_tbl (cost=0.00..1.09 rows=1 width=85) (actual time=0.023..0.023 rows=0.00 loops=1)
   Filter: (f1 @> '((0.5,2))'::polygon)
   Rows Removed by Filter: 7
   Buffers: shared hit=1
 Planning Time: 0.039 ms
 Execution Time: 0.033 ms
```

Der Planner nimmt zutreffend an, dass diese Beispieltabelle zu klein ist, um einen Index-Scan zu rechtfertigen. Daher sehen wir einen einfachen sequenziellen Scan, bei dem alle Zeilen durch die Filterbedingung verworfen wurden. Wenn wir einen Index-Scan erzwingen, sehen wir:

```sql
SET enable_seqscan TO off;

EXPLAIN ANALYZE SELECT * FROM polygon_tbl WHERE f1 @> polygon '(0.5,2.0)';
```

```text
 Index Scan using gpolygonind on polygon_tbl (cost=0.13..8.15 rows=1 width=85) (actual time=0.074..0.074 rows=0.00 loops=1)
   Index Cond: (f1 @> '((0.5,2))'::polygon)
   Rows Removed by Index Recheck: 1
   Index Searches: 1
   Buffers: shared hit=1
 Planning Time: 0.039 ms
 Execution Time: 0.098 ms
```

Hier sehen wir, dass der Index eine Kandidatenzeile zurückgegeben hat, die anschließend durch eine erneute Prüfung der Indexbedingung verworfen wurde. Das geschieht, weil ein GiST-Index für Polygon-Enthaltensein-Tests verlustbehaftet ist: Er gibt tatsächlich Zeilen zurück, deren Polygone sich mit dem Ziel überschneiden; anschließend muss der genaue Enthaltensein-Test auf diesen Zeilen ausgeführt werden.

`EXPLAIN` besitzt eine Option `BUFFERS`, die zusätzliche Details zu I/O-Operationen während Planung und Ausführung einer Abfrage liefert. Die angezeigten Buffer-Zahlen zählen die nicht eindeutigen Buffer, die für den jeweiligen Knoten und alle seine Kindknoten getroffen, gelesen, verschmutzt oder geschrieben wurden. Die Option `ANALYZE` aktiviert `BUFFERS` implizit. Wenn das nicht gewünscht ist, kann `BUFFERS` ausdrücklich deaktiviert werden:

```sql
EXPLAIN (ANALYZE, BUFFERS OFF)
SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;
```

```text
 Bitmap Heap Scan on tenk1 (cost=25.07..60.11 rows=10 width=244) (actual time=0.105..0.114 rows=10.00 loops=1)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   Heap Blocks: exact=10
   -> BitmapAnd (cost=25.07..25.07 rows=10 width=0) (actual time=0.100..0.101 rows=0.00 loops=1)
         -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0) (actual time=0.027..0.027 rows=100.00 loops=1)
               Index Cond: (unique1 < 100)
               Index Searches: 1
         -> Bitmap Index Scan on tenk1_unique2 (cost=0.00..19.78 rows=999 width=0) (actual time=0.070..0.070 rows=999.00 loops=1)
               Index Cond: (unique2 > 9000)
               Index Searches: 1
 Planning Time: 0.162 ms
 Execution Time: 0.143 ms
```

Da `EXPLAIN ANALYZE` die Abfrage tatsächlich ausführt, treten alle Seiteneffekte wie üblich auf, auch wenn mögliche Ergebnismengen verworfen werden und stattdessen die `EXPLAIN`-Daten ausgegeben werden. Wenn Sie eine datenverändernde Abfrage analysieren möchten, ohne Tabellen dauerhaft zu verändern, können Sie den Befehl anschließend zurückrollen:

```sql
BEGIN;

EXPLAIN ANALYZE UPDATE tenk1 SET hundred = hundred + 1 WHERE unique1 < 100;

ROLLBACK;
```

```text
 Update on tenk1 (cost=5.06..225.23 rows=0 width=0) (actual time=1.634..1.635 rows=0.00 loops=1)
   -> Bitmap Heap Scan on tenk1 (cost=5.06..225.23 rows=100 width=10) (actual time=0.065..0.141 rows=100.00 loops=1)
         Recheck Cond: (unique1 < 100)
         Heap Blocks: exact=90
         Buffers: shared hit=4 read=2
         -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0) (actual time=0.031..0.031 rows=100.00 loops=1)
               Index Cond: (unique1 < 100)
               Index Searches: 1
               Buffers: shared read=2
 Planning Time: 0.151 ms
 Execution Time: 1.856 ms
```

Wenn die Abfrage ein `INSERT`-, `UPDATE`-, `DELETE`- oder `MERGE`-Befehl ist, wird die eigentliche Arbeit des Anwendens der Tabellenänderungen durch einen obersten `Insert`-, `Update`-, `Delete`- oder `Merge`-Planknoten erledigt. Die darunterliegenden Planknoten suchen die alten Zeilen oder berechnen die neuen Daten. Oben sehen wir also dieselbe Art Bitmap-Tabellenscan wie zuvor; seine Ausgabe wird einem `Update`-Knoten zugeführt, der die aktualisierten Zeilen speichert. Obwohl der datenverändernde Knoten beträchtliche Laufzeit verbrauchen kann, fügt der Planner den Kostenschätzungen derzeit nichts hinzu, um diese Arbeit abzubilden. Der Grund ist, dass diese Arbeit für jeden korrekten Abfrageplan gleich ist und die Planungsentscheidung daher nicht beeinflusst.

Wenn ein `UPDATE`-, `DELETE`- oder `MERGE`-Befehl eine partitionierte Tabelle oder eine Vererbungshierarchie betrifft, kann die Ausgabe so aussehen:

```sql
EXPLAIN UPDATE gtest_parent SET f1 = CURRENT_DATE WHERE f2 = 101;
```

```text
                                        QUERY PLAN
--------------------------------------------------------------------------------
 Update on gtest_parent (cost=0.00..3.06 rows=0 width=0)
   Update on gtest_child gtest_parent_1
   Update on gtest_child2 gtest_parent_2
   Update on gtest_child3 gtest_parent_3
   -> Append (cost=0.00..3.06 rows=3 width=14)
         -> Seq Scan on gtest_child gtest_parent_1 (cost=0.00..1.01 rows=1 width=14)
               Filter: (f2 = 101)
         -> Seq Scan on gtest_child2 gtest_parent_2 (cost=0.00..1.01 rows=1 width=14)
               Filter: (f2 = 101)
         -> Seq Scan on gtest_child3 gtest_parent_3 (cost=0.00..1.01 rows=1 width=14)
               Filter: (f2 = 101)
```

In diesem Beispiel muss der `Update`-Knoten drei Kindtabellen berücksichtigen, aber nicht die ursprünglich genannte partitionierte Tabelle, weil diese selbst keine Daten speichert. Es gibt also drei eingehende Scan-Subpläne, einen pro Tabelle. Zur Klarheit ist der `Update`-Knoten so annotiert, dass die konkreten Zieltabellen angezeigt werden, die aktualisiert werden, und zwar in derselben Reihenfolge wie die entsprechenden Subpläne.

Die von `EXPLAIN ANALYZE` angezeigte Planungszeit ist die Zeit, die benötigt wurde, um aus der geparsten Abfrage den Abfrageplan zu erzeugen und ihn zu optimieren. Sie enthält nicht das Parsen oder Umschreiben.

Die von `EXPLAIN ANALYZE` angezeigte Ausführungszeit enthält Start- und Stoppzeit des Executors sowie die Zeit zum Ausführen ausgelöster Trigger. Sie enthält aber nicht Parsing, Rewriting oder Planning. Zeit für `BEFORE`-Trigger wird, falls vorhanden, in der Zeit des zugehörigen `Insert`-, `Update`- oder `Delete`-Knotens erfasst. Zeit für `AFTER`-Trigger wird dort nicht gezählt, weil `AFTER`-Trigger erst nach Abschluss des gesamten Plans ausgelöst werden. Die Gesamtzeit jedes Triggers, ob `BEFORE` oder `AFTER`, wird zusätzlich separat angezeigt. Deferred Constraint Trigger werden erst am Ende der Transaktion ausgeführt und daher von `EXPLAIN ANALYZE` gar nicht berücksichtigt.

Die Zeit des obersten Knotens enthält keine Zeit, die benötigt wird, um die Ausgabedaten der Abfrage in eine anzeigbare Form umzuwandeln oder sie an den Client zu senden. `EXPLAIN ANALYZE` sendet die Daten nie an den Client, kann aber mit der Option `SERIALIZE` angewiesen werden, die Ausgabedaten in eine anzeigbare Form umzuwandeln und die dafür benötigte Zeit zu messen. Diese Zeit wird separat angezeigt und ist außerdem in der gesamten `Execution Time` enthalten.

### 14.1.3. Einschränkungen

Es gibt zwei wichtige Arten, wie mit `EXPLAIN ANALYZE` gemessene Laufzeiten von der normalen Ausführung derselben Abfrage abweichen können. Erstens werden keine Ausgabezeilen an den Client geliefert, daher sind Netzwerkübertragungskosten nicht enthalten. I/O-Konvertierungskosten sind ebenfalls nicht enthalten, außer wenn `SERIALIZE` angegeben wird. Zweitens kann der Messaufwand von `EXPLAIN ANALYZE` erheblich sein, besonders auf Maschinen mit langsamen Betriebssystemaufrufen von `gettimeofday()`. Mit dem Werkzeug `pg_test_timing` können Sie den Timing-Overhead auf Ihrem System messen.

`EXPLAIN`-Ergebnisse sollten nicht auf Situationen übertragen werden, die sich stark von der tatsächlich getesteten unterscheiden. Ergebnisse auf einer winzigen Testtabelle gelten zum Beispiel nicht zwingend für große Tabellen. Die Kostenschätzungen des Planners sind nicht linear; er kann für eine größere oder kleinere Tabelle einen anderen Plan wählen. Ein extremes Beispiel ist eine Tabelle, die nur eine Plattenseite belegt: Dort erhalten Sie fast immer einen sequenziellen Scan, unabhängig davon, ob Indizes vorhanden sind. Der Planner erkennt, dass in jedem Fall eine Plattenseite gelesen werden muss und dass zusätzliche Seitenzugriffe für einen Index keinen Nutzen bringen. Das war im Beispiel mit `polygon_tbl` zu sehen.

Es gibt Fälle, in denen tatsächliche und geschätzte Werte nicht gut übereinstimmen, ohne dass wirklich etwas falsch ist. Ein solcher Fall tritt ein, wenn die Ausführung eines Planknotens durch `LIMIT` oder einen ähnlichen Effekt vorzeitig gestoppt wird:

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1
WHERE unique1 < 100 AND unique2 > 9000 LIMIT 2;
```

```text
 Limit (cost=0.29..14.33 rows=2 width=244) (actual time=0.051..0.071 rows=2.00 loops=1)
   Buffers: shared hit=16
   -> Index Scan using tenk1_unique2 on tenk1 (cost=0.29..70.50 rows=10 width=244) (actual time=0.051..0.070 rows=2.00 loops=1)
         Index Cond: (unique2 > 9000)
         Filter: (unique1 < 100)
         Rows Removed by Filter: 287
         Index Searches: 1
         Buffers: shared hit=16
 Planning Time: 0.077 ms
 Execution Time: 0.086 ms
```

Die geschätzten Kosten und die geschätzte Zeilenzahl des `Index Scan`-Knotens werden so angezeigt, als würde der Knoten vollständig ausgeführt. Tatsächlich hört der `Limit`-Knoten nach zwei Zeilen auf, weitere Zeilen anzufordern. Die tatsächliche Zeilenzahl beträgt daher nur 2, und die Laufzeit ist kleiner, als die Kostenschätzung nahelegen würde. Das ist kein Schätzfehler, sondern eine Diskrepanz in der Darstellung von Schätzungen und tatsächlichen Werten.

Auch Merge Joins haben Messartefakte, die leicht verwirren. Ein Merge Join beendet das Lesen einer Eingabe, wenn die andere Eingabe erschöpft ist und der nächste Schlüsselwert in der ersten Eingabe größer ist als der letzte Schlüsselwert der anderen Eingabe; dann kann es keine weiteren Treffer geben, und der Rest der ersten Eingabe muss nicht gescannt werden. Dadurch wird ein Kindknoten nicht vollständig gelesen, ähnlich wie bei `LIMIT`. Außerdem wird bei doppelten Schlüsselwerten im äußeren, ersten Kind das innere, zweite Kind zurückgesetzt und für den passenden Bereich erneut gescannt. `EXPLAIN ANALYZE` zählt diese wiederholten Ausgaben derselben inneren Zeilen so, als wären es echte zusätzliche Zeilen. Bei vielen äußeren Duplikaten kann die gemeldete tatsächliche Zeilenzahl des inneren Kindplanknotens deutlich größer sein als die Zahl der Zeilen in der inneren Relation.

`BitmapAnd`- und `BitmapOr`-Knoten melden ihre tatsächlichen Zeilenzahlen wegen Implementierungsbeschränkungen immer als null.

Normalerweise zeigt `EXPLAIN` jeden vom Planner erzeugten Planknoten an. Es gibt jedoch Fälle, in denen der Executor anhand von Parameterwerten, die zur Planungszeit noch nicht verfügbar waren, feststellen kann, dass bestimmte Knoten nicht ausgeführt werden müssen, weil sie keine Zeilen erzeugen können. Derzeit kann das nur bei Kindknoten eines `Append`- oder `MergeAppend`-Knotens passieren, der eine partitionierte Tabelle scannt. In diesem Fall werden diese Planknoten aus der `EXPLAIN`-Ausgabe weggelassen, und stattdessen erscheint eine Annotation `Subplans Removed: N`.

## 14.2. Vom Planner verwendete Statistiken

### 14.2.1. Einspaltige Statistiken

Wie im vorherigen Abschnitt zu sehen war, muss der Query Planner schätzen, wie viele Zeilen eine Abfrage abruft, um gute Abfragepläne auswählen zu können. Dieser Abschnitt gibt einen kurzen Überblick über die Statistiken, die das System für diese Schätzungen verwendet.

Ein Bestandteil der Statistiken ist die Gesamtzahl der Einträge in jeder Tabelle und jedem Index sowie die Anzahl der Disk-Blöcke, die jede Tabelle und jeder Index belegt. Diese Informationen werden in der Tabelle `pg_class` in den Spalten `reltuples` und `relpages` gespeichert. Eine Abfrage kann so aussehen:

```sql
SELECT relname, relkind, reltuples, relpages
FROM pg_class
WHERE relname LIKE 'tenk1%';
```

```text
       relname        | relkind | reltuples | relpages
----------------------+---------+-----------+----------
 tenk1                | r       |     10000 |      345
 tenk1_hundred        | i       |     10000 |       11
 tenk1_thous_tenthous | i       |     10000 |       30
 tenk1_unique1        | i       |     10000 |       30
 tenk1_unique2        | i       |     10000 |       30
(5 rows)
```

Hier sehen wir, dass `tenk1` 10000 Zeilen enthält, ebenso wie seine Indizes. Die Indizes sind jedoch, wenig überraschend, deutlich kleiner als die Tabelle.

Aus Effizienzgründen werden `reltuples` und `relpages` nicht fortlaufend aktualisiert und enthalten daher meist etwas veraltete Werte. Sie werden durch `VACUUM`, `ANALYZE` und einige DDL-Befehle wie `CREATE INDEX` aktualisiert. Eine `VACUUM`- oder `ANALYZE`-Operation, die nicht die gesamte Tabelle scannt, was häufig der Fall ist, aktualisiert die `reltuples`-Zählung inkrementell anhand des gescannten Tabellenteils; das Ergebnis ist ein Näherungswert. In jedem Fall skaliert der Planner die in `pg_class` gefundenen Werte passend zur aktuellen physischen Tabellengröße und erhält so eine bessere Näherung.

Die meisten Abfragen rufen wegen einschränkender `WHERE`-Klauseln nur einen Teil der Zeilen einer Tabelle ab. Der Planner muss daher die Selektivität von `WHERE`-Klauseln schätzen, also den Anteil der Zeilen, die jede Bedingung in der `WHERE`-Klausel erfüllen. Die dafür verwendeten Informationen werden im Systemkatalog `pg_statistic` gespeichert. Einträge in `pg_statistic` werden durch `ANALYZE` und `VACUUM ANALYZE` aktualisiert und sind selbst direkt nach der Aktualisierung immer Näherungswerte.

Wenn Sie Statistiken manuell untersuchen, sollten Sie nicht direkt in `pg_statistic`, sondern in die Sicht `pg_stats` schauen. `pg_stats` ist leichter lesbar. Außerdem ist `pg_stats` für alle lesbar, während `pg_statistic` nur von Superusern gelesen werden kann. Das verhindert, dass unprivilegierte Benutzer aus Statistiken etwas über Inhalte fremder Tabellen lernen. Die Sicht `pg_stats` zeigt nur Zeilen zu Tabellen, die der aktuelle Benutzer lesen darf. Beispiel:

```sql
SELECT attname, inherited, n_distinct,
       array_to_string(most_common_vals, E'\n') as most_common_vals
FROM pg_stats
WHERE tablename = 'road';
```

```text
 attname | inherited | n_distinct | most_common_vals
---------+-----------+------------+------------------
 name    | f         | -0.5681108 | I- 580 Ramp
 ...
 name    | t         | -0.5125    | I- 580 Ramp
 ...
 thepath | f         | 0          |
 thepath | t         | 0          |
(4 rows)
```

Beachten Sie, dass für dieselbe Spalte zwei Zeilen angezeigt werden: eine für die vollständige Vererbungshierarchie ab der Tabelle `road` (`inherited = t`) und eine, die nur die Tabelle `road` selbst umfasst (`inherited = f`). Der Kürze halber sind im Beispiel nur die ersten zehn häufigsten Werte der Spalte `name` gezeigt.

Die Menge der von `ANALYZE` in `pg_statistic` gespeicherten Informationen, insbesondere die maximale Anzahl von Einträgen in den Arrays `most_common_vals` und `histogram_bounds` für jede Spalte, kann spaltenweise mit `ALTER TABLE SET STATISTICS` oder global über die Konfigurationsvariable `default_statistics_target` festgelegt werden. Die Standardgrenze liegt derzeit bei 100 Einträgen. Eine höhere Grenze kann genauere Planner-Schätzungen ermöglichen, besonders für Spalten mit unregelmäßigen Datenverteilungen. Dafür verbraucht `pg_statistic` mehr Platz, und die Berechnung der Schätzungen dauert etwas länger. Umgekehrt kann für Spalten mit einfachen Datenverteilungen eine niedrigere Grenze genügen.

Weitere Details dazu, wie der Planner Statistiken verwendet, finden Sie in [Kapitel 69](69_Wie_der_Planner_Statistiken_verwendet.md).

### 14.2.2. Erweiterte Statistiken

Langsame Abfragen mit schlechten Ausführungsplänen treten häufig auf, wenn mehrere in den Abfrageklauseln verwendete Spalten korreliert sind. Der Planner nimmt normalerweise an, dass mehrere Bedingungen unabhängig voneinander sind; diese Annahme gilt nicht, wenn Spaltenwerte korreliert sind. Normale Statistiken können wegen ihrer spaltenweisen Natur keine Informationen über Korrelationen zwischen Spalten erfassen. PostgreSQL kann jedoch multivariate Statistiken berechnen, die solche Informationen erfassen.

Da die Zahl möglicher Spaltenkombinationen sehr groß ist, wäre es unpraktisch, multivariate Statistiken automatisch für alle Kombinationen zu berechnen. Stattdessen können erweiterte Statistikobjekte, oft einfach Statistikobjekte genannt, erstellt werden, um den Server anzuweisen, Statistiken über interessante Spaltenmengen zu sammeln.

Statistikobjekte werden mit dem Befehl `CREATE STATISTICS` erzeugt. Das Erstellen eines solchen Objekts legt zunächst nur einen Katalogeintrag an, der Interesse an diesen Statistiken ausdrückt. Die eigentliche Datensammlung erfolgt durch `ANALYZE`, entweder manuell oder durch Auto-Analyze im Hintergrund. Die gesammelten Werte können im Katalog `pg_statistic_ext_data` untersucht werden.

`ANALYZE` berechnet erweiterte Statistiken anhand derselben Stichprobe von Tabellenzeilen, die es für normale einspaltige Statistiken verwendet. Da die Stichprobengröße durch Erhöhung des Statistikziels für die Tabelle oder eine ihrer Spalten steigt, führt ein größeres Statistikziel normalerweise zu genaueren erweiterten Statistiken, aber auch zu mehr Berechnungszeit.

Die folgenden Unterabschnitte beschreiben die derzeit unterstützten Arten erweiterter Statistiken.

#### 14.2.2.1. Funktionale Abhängigkeiten

Die einfachste Art erweiterter Statistiken verfolgt funktionale Abhängigkeiten, ein Konzept aus Definitionen von Datenbanknormalformen. Eine Spalte `b` ist funktional von einer Spalte `a` abhängig, wenn die Kenntnis des Werts von `a` ausreicht, um den Wert von `b` zu bestimmen. Es gibt dann also keine zwei Zeilen mit demselben Wert von `a`, aber unterschiedlichen Werten von `b`. In einer vollständig normalisierten Datenbank sollten funktionale Abhängigkeiten nur bei Primärschlüsseln und Superschlüsseln vorkommen. In der Praxis sind viele Datenbestände aus unterschiedlichen Gründen nicht vollständig normalisiert; absichtliche Denormalisierung aus Performancegründen ist ein häufiges Beispiel. Selbst in einer vollständig normalisierten Datenbank kann es partielle Korrelation zwischen Spalten geben, die als partielle funktionale Abhängigkeit ausgedrückt werden kann.

Funktionale Abhängigkeiten beeinflussen direkt die Genauigkeit von Schätzungen in bestimmten Abfragen. Wenn eine Abfrage Bedingungen sowohl auf der unabhängigen als auch auf der abhängigen Spalte enthält, reduzieren die Bedingungen auf den abhängigen Spalten die Ergebnisgröße nicht weiter. Ohne Kenntnis der funktionalen Abhängigkeit nimmt der Query Planner jedoch an, dass die Bedingungen unabhängig sind, und unterschätzt dadurch die Ergebnisgröße.

Um den Planner über funktionale Abhängigkeiten zu informieren, kann `ANALYZE` Messwerte spaltenübergreifender Abhängigkeiten sammeln. Den Abhängigkeitsgrad zwischen allen Spaltenmengen zu bewerten wäre unvertretbar teuer; daher beschränkt sich die Datensammlung auf Spaltengruppen, die gemeinsam in einem mit der Option `dependencies` definierten Statistikobjekt vorkommen. Es ist ratsam, `dependencies`-Statistiken nur für stark korrelierte Spaltengruppen zu erstellen, um unnötigen Aufwand in `ANALYZE` und späterer Abfrageplanung zu vermeiden.

Beispiel:

```sql
CREATE STATISTICS stts (dependencies) ON city, zip FROM zipcodes;

ANALYZE zipcodes;

SELECT stxname, stxkeys, stxddependencies
  FROM pg_statistic_ext join pg_statistic_ext_data on (oid = stxoid)
  WHERE stxname = 'stts';
```

```text
 stxname | stxkeys |             stxddependencies
---------+---------+------------------------------------------
 stts    | 1 5     | {"1 => 5": 1.000000, "5 => 1": 0.423130}
(1 row)
```

Hier ist zu sehen, dass Spalte 1, die Postleitzahl, Spalte 5, die Stadt, vollständig bestimmt; der Koeffizient beträgt also 1,0. Die Stadt bestimmt die Postleitzahl dagegen nur in etwa 42 Prozent der Fälle. Das bedeutet, dass es viele Städte gibt, hier 58 Prozent, die durch mehr als eine Postleitzahl repräsentiert werden.

Wenn der Planner die Selektivität für eine Abfrage mit funktional abhängigen Spalten berechnet, passt er die Selektivitätsschätzungen der einzelnen Bedingungen anhand der Abhängigkeitskoeffizienten an, um keine Unterschätzung zu erzeugen.

##### 14.2.2.1.1. Einschränkungen funktionaler Abhängigkeiten

Funktionale Abhängigkeiten werden derzeit nur bei einfachen Gleichheitsbedingungen angewendet, die Spalten mit konstanten Werten vergleichen, sowie bei `IN`-Klauseln mit konstanten Werten. Sie werden nicht verwendet, um Schätzungen für Gleichheitsbedingungen zu verbessern, die zwei Spalten oder eine Spalte mit einem Ausdruck vergleichen. Auch bei Bereichsklauseln, `LIKE` oder anderen Bedingungstypen werden sie nicht verwendet.

Beim Schätzen mit funktionalen Abhängigkeiten nimmt der Planner an, dass Bedingungen auf den beteiligten Spalten kompatibel und daher redundant sind. Wenn sie inkompatibel sind, wäre die korrekte Schätzung null Zeilen, aber diese Möglichkeit wird nicht berücksichtigt. Beispiel:

```sql
SELECT * FROM zipcodes WHERE city = 'San Francisco' AND zip = '94105';
```

Hier ignoriert der Planner die `city`-Klausel als nicht selektivitätsändernd, was korrekt ist. Er trifft dieselbe Annahme aber auch bei:

```sql
SELECT * FROM zipcodes WHERE city = 'San Francisco' AND zip = '90210';
```

Tatsächlich wird diese Abfrage keine Zeilen erfüllen. Funktionale Abhängigkeitsstatistiken liefern jedoch nicht genug Information, um das zu erkennen.

In vielen praktischen Situationen ist diese Annahme erfüllt; zum Beispiel kann eine grafische Oberfläche in der Anwendung nur kompatible Stadt- und Postleitzahlwerte zur Auswahl anbieten. Wenn das nicht der Fall ist, sind funktionale Abhängigkeiten möglicherweise keine geeignete Option.

#### 14.2.2.2. Multivariate N-Distinct-Zählungen

Einspaltige Statistiken speichern die Zahl der unterschiedlichen Werte in jeder Spalte. Schätzungen der Zahl unterschiedlicher Werte bei Kombination mehrerer Spalten, etwa für `GROUP BY a, b`, sind häufig falsch, wenn der Planner nur einspaltige Statistikdaten besitzt. Das kann zu schlechten Plänen führen.

Um solche Schätzungen zu verbessern, kann `ANALYZE` N-Distinct-Statistiken für Spaltengruppen sammeln. Wie zuvor ist es unpraktisch, dies für jede mögliche Spaltengruppe zu tun. Daher werden Daten nur für Spaltengruppen gesammelt, die gemeinsam in einem mit der Option `ndistinct` definierten Statistikobjekt vorkommen. Für jede mögliche Kombination von zwei oder mehr Spalten aus der angegebenen Spaltenmenge werden Daten gesammelt.

Im Beispiel der Postleitzahlentabelle könnten die N-Distinct-Zählungen so aussehen:

```sql
CREATE STATISTICS stts2 (ndistinct) ON city, state, zip FROM zipcodes;

ANALYZE zipcodes;

SELECT stxkeys AS k, stxdndistinct AS nd
   FROM pg_statistic_ext join pg_statistic_ext_data on (oid = stxoid)
   WHERE stxname = 'stts2';
```

```text
-[ RECORD 1 ]--------------------------------------------------------
k  | 1 2 5
nd | {"1, 2": 33178, "1, 5": 33178, "2, 5": 27435, "1, 2, 5": 33178}
(1 row)
```

Das zeigt, dass drei Spaltenkombinationen jeweils 33178 unterschiedliche Werte haben: Postleitzahl und Bundesstaat, Postleitzahl und Stadt sowie Postleitzahl, Stadt und Bundesstaat. Dass diese Werte gleich sind, ist zu erwarten, da die Postleitzahl in dieser Tabelle allein eindeutig ist. Die Kombination aus Stadt und Bundesstaat hat dagegen nur 27435 unterschiedliche Werte.

Es ist ratsam, `ndistinct`-Statistikobjekte nur für Spaltenkombinationen zu erstellen, die tatsächlich für Gruppierungen verwendet werden und bei denen eine Fehlschätzung der Gruppenzahl schlechte Pläne verursacht. Andernfalls werden nur `ANALYZE`-Zyklen verschwendet.

#### 14.2.2.3. Multivariate MCV-Listen

Eine weitere Statistikart, die für jede Spalte gespeichert wird, sind Listen der häufigsten Werte, sogenannte MCV-Listen. MCV steht für Most Common Values. Solche Listen erlauben sehr genaue Schätzungen für einzelne Spalten, können aber bei Abfragen mit Bedingungen auf mehreren Spalten zu erheblichen Fehlschätzungen führen.

Zur Verbesserung solcher Schätzungen kann `ANALYZE` MCV-Listen für Spaltenkombinationen sammeln. Ähnlich wie bei funktionalen Abhängigkeiten und N-Distinct-Koeffizienten ist es unpraktisch, dies für jede mögliche Spaltengruppe zu tun. In diesem Fall gilt das besonders, weil eine MCV-Liste im Gegensatz zu funktionalen Abhängigkeiten und N-Distinct-Koeffizienten die häufigen Spaltenwerte selbst speichert. Daten werden daher nur für Spaltengruppen gesammelt, die gemeinsam in einem mit der Option `mcv` definierten Statistikobjekt vorkommen.

Im Beispiel der Postleitzahlentabelle könnte die MCV-Liste so untersucht werden; anders als bei einfacheren Statistikarten ist dafür eine Funktion erforderlich:

```sql
CREATE STATISTICS stts3 (mcv) ON city, state FROM zipcodes;

ANALYZE zipcodes;

SELECT m.* FROM pg_statistic_ext join pg_statistic_ext_data on (oid = stxoid),
                pg_mcv_list_items(stxdmcv) m
WHERE stxname = 'stts3';
```

```text
 index |          values        | nulls | frequency | base_frequency
-------+------------------------+-------+-----------+----------------
     0 | {Washington, DC}       | {f,f} | 0.003467 | 2.7e-05
     1 | {Apo, AE}              | {f,f} | 0.003067 | 1.9e-05
     2 | {Houston, TX}          | {f,f} | 0.002167 | 0.000133
     3 | {El Paso, TX}          | {f,f} |     0.002 | 0.000113
     4 | {New York, NY}         | {f,f} | 0.001967 | 0.000114
     ...
(99 rows)
```

Das zeigt, dass die häufigste Kombination aus Stadt und Bundesstaat `Washington, DC` ist, mit einer tatsächlichen Häufigkeit in der Stichprobe von etwa 0,35 Prozent. Die Basishäufigkeit der Kombination, berechnet aus den einfachen Einzelspaltenhäufigkeiten, beträgt nur 0,0027 Prozent, was zu einer Unterschätzung um zwei Größenordnungen führt.

Es ist ratsam, MCV-Statistikobjekte nur für Spaltenkombinationen zu erstellen, die tatsächlich gemeinsam in Bedingungen verwendet werden und bei denen eine Fehlschätzung der Gruppenzahl schlechte Pläne verursacht. Andernfalls werden nur `ANALYZE`- und Planungszyklen verschwendet.

## 14.3. Den Planner mit expliziten JOIN-Klauseln steuern

Mit expliziter `JOIN`-Syntax kann der Query Planner in gewissem Maß gesteuert werden. Um zu verstehen, warum das wichtig sein kann, brauchen wir etwas Hintergrund.

Bei einer einfachen Join-Abfrage wie:

```sql
SELECT * FROM a, b, c WHERE a.id = b.id AND b.ref = c.id;
```

kann der Planner die angegebenen Tabellen in beliebiger Reihenfolge verbinden. Er könnte zum Beispiel einen Plan erzeugen, der zunächst A mit B über die `WHERE`-Bedingung `a.id = b.id` verbindet und anschließend C mit dieser verbundenen Tabelle über die andere `WHERE`-Bedingung. Oder er könnte zuerst B mit C verbinden und danach A mit dem Ergebnis. Oder er könnte A mit C verbinden und anschließend mit B; das wäre allerdings ineffizient, weil das vollständige kartesische Produkt von A und C gebildet werden müsste, da es in der `WHERE`-Klausel keine anwendbare Bedingung zur Optimierung dieses Joins gibt. Alle Joins im PostgreSQL-Executor erfolgen zwischen zwei Eingabetabellen, daher muss das Ergebnis auf die eine oder andere Weise aufgebaut werden. Wichtig ist: Diese verschiedenen Join-Möglichkeiten liefern semantisch gleichwertige Ergebnisse, können aber stark unterschiedliche Ausführungskosten haben. Deshalb untersucht der Planner sie, um den effizientesten Abfrageplan zu finden.

Wenn eine Abfrage nur zwei oder drei Tabellen umfasst, gibt es nicht viele Join-Reihenfolgen. Die Zahl möglicher Join-Reihenfolgen wächst jedoch exponentiell mit der Zahl der Tabellen. Bei mehr als etwa zehn Eingabetabellen ist eine vollständige Suche aller Möglichkeiten nicht mehr praktikabel, und schon bei sechs oder sieben Tabellen kann die Planung störend lange dauern. Wenn es zu viele Eingabetabellen gibt, wechselt der PostgreSQL-Planner von vollständiger Suche zu einer genetischen probabilistischen Suche über eine begrenzte Zahl von Möglichkeiten. Der Umschaltpunkt wird durch den Laufzeitparameter `geqo_threshold` bestimmt. Die genetische Suche benötigt weniger Zeit, findet aber nicht zwingend den bestmöglichen Plan.

Bei Outer Joins hat der Planner weniger Freiheit als bei normalen Inner Joins. Betrachten Sie dieses Beispiel:

```sql
SELECT * FROM a LEFT JOIN (b JOIN c ON (b.ref = c.id)) ON (a.id = b.id);
```

Obwohl die Einschränkungen dieser Abfrage oberflächlich dem vorherigen Beispiel ähneln, ist die Semantik anders: Für jede Zeile von A, die keine passende Zeile im Join von B und C hat, muss eine Zeile ausgegeben werden. Der Planner hat hier daher keine Wahl bei der Join-Reihenfolge: Er muss B mit C verbinden und anschließend A mit diesem Ergebnis. Entsprechend benötigt diese Abfrage weniger Planungszeit als die vorherige. In anderen Fällen kann der Planner feststellen, dass mehr als eine Join-Reihenfolge sicher ist. Beispiel:

```sql
SELECT * FROM a LEFT JOIN b ON (a.bid = b.id) LEFT JOIN c ON (a.cid = c.id);
```

Hier ist es zulässig, A zuerst entweder mit B oder mit C zu verbinden. Derzeit beschränkt nur `FULL JOIN` die Join-Reihenfolge vollständig. Die meisten praktischen Fälle mit `LEFT JOIN` oder `RIGHT JOIN` können in gewissem Umfang umgeordnet werden.

Explizite Inner-Join-Syntax, also `INNER JOIN`, `CROSS JOIN` oder schlicht `JOIN`, ist semantisch dasselbe wie das Auflisten der Eingaberelationen in `FROM`; sie beschränkt die Join-Reihenfolge daher nicht.

Auch wenn die meisten Arten von `JOIN` die Join-Reihenfolge nicht vollständig festlegen, kann man den PostgreSQL Query Planner anweisen, alle `JOIN`-Klauseln trotzdem als Reihenfolgevorgabe zu behandeln. Diese drei Abfragen sind logisch äquivalent:

```sql
SELECT * FROM a, b, c WHERE a.id = b.id AND b.ref = c.id;
SELECT * FROM a CROSS JOIN b CROSS JOIN c WHERE a.id = b.id AND b.ref = c.id;
SELECT * FROM a JOIN (b JOIN c ON (b.ref = c.id)) ON (a.id = b.id);
```

Wenn wir dem Planner aber sagen, dass er die `JOIN`-Reihenfolge respektieren soll, benötigen die zweite und dritte Abfrage weniger Planungszeit als die erste. Bei nur drei Tabellen lohnt es sich nicht, darüber nachzudenken; bei vielen Tabellen kann dieser Effekt jedoch entscheidend sein.

Um den Planner zu zwingen, der durch explizite `JOIN`s angegebenen Join-Reihenfolge zu folgen, setzen Sie den Laufzeitparameter `join_collapse_limit` auf `1`. Andere mögliche Werte werden unten erläutert.

Sie müssen die Join-Reihenfolge nicht vollständig beschränken, um die Suchzeit zu verringern, denn `JOIN`-Operatoren können auch innerhalb von Elementen einer normalen `FROM`-Liste verwendet werden. Beispiel:

```sql
SELECT * FROM a CROSS JOIN b, c, d, e WHERE ...;
```

Mit `join_collapse_limit = 1` erzwingt dies, dass der Planner A mit B verbindet, bevor sie mit anderen Tabellen verbunden werden. Ansonsten bleiben seine Wahlmöglichkeiten frei. In diesem Beispiel wird die Zahl möglicher Join-Reihenfolgen um den Faktor 5 reduziert.

Die Suche des Planners auf diese Weise einzuschränken, ist eine nützliche Technik, sowohl um Planungszeit zu reduzieren als auch um den Planner zu einem guten Abfrageplan zu lenken. Wenn der Planner standardmäßig eine schlechte Join-Reihenfolge wählt, können Sie ihn über `JOIN`-Syntax zu einer besseren Reihenfolge zwingen, vorausgesetzt natürlich, Sie kennen eine bessere Reihenfolge. Experimentieren ist empfehlenswert.

Ein eng verwandtes Thema, das die Planungszeit beeinflusst, ist das Zusammenfalten von Unterabfragen in ihre übergeordnete Abfrage. Beispiel:

```sql
SELECT *
FROM x, y,
    (SELECT * FROM a, b, c WHERE something) AS ss
WHERE somethingelse;
```

Diese Situation kann durch eine Sicht entstehen, die einen Join enthält. Die `SELECT`-Regel der Sicht wird anstelle der Sichtreferenz eingefügt und ergibt eine Abfrage ähnlich der obigen. Normalerweise versucht der Planner, die Unterabfrage in die übergeordnete Abfrage zu integrieren:

```sql
SELECT * FROM x, y, a, b, c WHERE something AND somethingelse;
```

Das führt meist zu einem besseren Plan, als die Unterabfrage separat zu planen. Zum Beispiel können die äußeren `WHERE`-Bedingungen dazu führen, dass ein Join von X mit A früh viele Zeilen aus A entfernt und dadurch die vollständige logische Ausgabe der Unterabfrage gar nicht gebildet werden muss. Gleichzeitig steigt aber die Planungszeit: Aus zwei getrennten Drei-Wege-Join-Problemen wird hier ein Fünf-Wege-Join-Problem. Wegen des exponentiellen Wachstums der Möglichkeiten macht das einen großen Unterschied. Der Planner versucht, riesige Join-Suchprobleme zu vermeiden, indem er eine Unterabfrage nicht zusammenfaltet, wenn dadurch mehr als `from_collapse_limit` `FROM`-Elemente in der übergeordneten Abfrage entstehen würden. Durch Anpassen dieses Laufzeitparameters können Sie Planungszeit gegen Planqualität abwägen.

`from_collapse_limit` und `join_collapse_limit` heißen ähnlich, weil sie fast dasselbe steuern: Der eine Parameter steuert, wann der Planner Unterabfragen glättet, der andere, wann er explizite Joins glättet. Typischerweise setzen Sie `join_collapse_limit` entweder gleich `from_collapse_limit`, sodass explizite Joins und Unterabfragen ähnlich behandelt werden, oder auf `1`, wenn Sie die Join-Reihenfolge mit expliziten Joins kontrollieren wollen. Sie können sie jedoch auch unterschiedlich setzen, wenn Sie die Abwägung zwischen Planungszeit und Laufzeit feinjustieren möchten.

## 14.4. Eine Datenbank befüllen

Beim ersten Befüllen einer Datenbank müssen unter Umständen große Datenmengen eingefügt werden. Dieser Abschnitt enthält Hinweise, wie dieser Prozess möglichst effizient gestaltet werden kann.

### 14.4.1. Autocommit deaktivieren

Wenn Sie mehrere `INSERT`s verwenden, schalten Sie Autocommit aus und führen Sie am Ende nur ein Commit aus. In reinem SQL bedeutet das, am Anfang `BEGIN` und am Ende `COMMIT` auszuführen. Einige Client-Bibliotheken erledigen das im Hintergrund; dann müssen Sie sicherstellen, dass die Bibliothek es genau dann tut, wenn Sie es möchten. Wenn jede Einfügung separat committed wird, muss PostgreSQL für jede hinzugefügte Zeile viel Arbeit leisten. Ein zusätzlicher Vorteil, alle Einfügungen in einer Transaktion auszuführen, besteht darin, dass beim Fehlschlagen einer Zeileneinfügung alle bis dahin eingefügten Zeilen zurückgerollt werden. Sie bleiben also nicht mit teilweise geladenen Daten zurück.

### 14.4.2. COPY verwenden

Verwenden Sie `COPY`, um alle Zeilen in einem Befehl zu laden, statt eine Reihe von `INSERT`-Befehlen auszuführen. `COPY` ist für das Laden großer Zeilenzahlen optimiert. Es ist weniger flexibel als `INSERT`, verursacht bei großen Ladevorgängen aber deutlich weniger Overhead. Da `COPY` ein einzelner Befehl ist, müssen Sie Autocommit nicht deaktivieren, wenn Sie eine Tabelle auf diese Weise befüllen.

Wenn Sie `COPY` nicht verwenden können, kann es helfen, mit `PREPARE` eine vorbereitete `INSERT`-Anweisung zu erzeugen und anschließend `EXECUTE` so oft wie nötig auszuführen. Dadurch wird ein Teil des Overheads vermieden, der durch wiederholtes Parsen und Planen von `INSERT` entsteht. Verschiedene Schnittstellen stellen diese Möglichkeit unterschiedlich bereit; suchen Sie in der Dokumentation der Schnittstelle nach vorbereiteten Anweisungen.

Das Laden vieler Zeilen mit `COPY` ist fast immer schneller als mit `INSERT`, selbst wenn `PREPARE` verwendet wird und mehrere Einfügungen in einer einzelnen Transaktion gebündelt werden.

`COPY` ist am schnellsten, wenn es in derselben Transaktion wie ein vorheriges `CREATE TABLE` oder `TRUNCATE` verwendet wird. In solchen Fällen muss kein WAL geschrieben werden, weil die Dateien mit den neu geladenen Daten im Fehlerfall ohnehin entfernt werden. Diese Überlegung gilt jedoch nur, wenn `wal_level` auf `minimal` steht; andernfalls müssen alle Befehle WAL schreiben.

### 14.4.3. Indizes entfernen

Wenn Sie eine neu erstellte Tabelle laden, ist die schnellste Methode, zuerst die Tabelle zu erstellen, dann die Tabellendaten mit `COPY` massenhaft zu laden und anschließend alle benötigten Indizes zu erstellen. Einen Index auf bereits vorhandenen Daten zu erstellen ist schneller, als ihn inkrementell bei jeder geladenen Zeile zu aktualisieren.

Wenn Sie große Datenmengen zu einer bestehenden Tabelle hinzufügen, kann es sich lohnen, die Indizes zu löschen, die Tabelle zu laden und die Indizes anschließend neu zu erstellen. Natürlich kann während der Zeit ohne Indizes die Datenbankperformance für andere Benutzer leiden. Bei Unique-Indizes sollten Sie außerdem besonders vorsichtig sein, weil die durch die Unique-Constraint bereitgestellte Fehlerprüfung verloren geht, solange der Index fehlt.

### 14.4.4. Fremdschlüssel-Constraints entfernen

Wie bei Indizes kann auch ein Fremdschlüssel-Constraint im Block effizienter geprüft werden als zeilenweise. Daher kann es nützlich sein, Fremdschlüssel-Constraints zu löschen, Daten zu laden und die Constraints anschließend neu zu erstellen. Auch hier gibt es einen Kompromiss zwischen Ladegeschwindigkeit und dem Verlust der Fehlerprüfung, solange das Constraint fehlt.

Außerdem benötigt jede neue Zeile beim Laden von Daten in eine Tabelle mit bestehenden Fremdschlüssel-Constraints einen Eintrag in der Liste ausstehender Trigger-Ereignisse des Servers, weil ein Trigger die Fremdschlüsselbedingung prüft. Das Laden vieler Millionen Zeilen kann dazu führen, dass die Trigger-Ereigniswarteschlange den verfügbaren Speicher übersteigt, was zu unerträglichem Swapping oder sogar zum vollständigen Fehlschlagen des Befehls führen kann. Daher kann es beim Laden großer Datenmengen notwendig und nicht nur wünschenswert sein, Fremdschlüssel zu entfernen und danach wieder anzuwenden. Wenn ein temporäres Entfernen des Constraints nicht akzeptabel ist, bleibt möglicherweise nur, den Ladevorgang in kleinere Transaktionen aufzuteilen.

### 14.4.5. maintenance_work_mem erhöhen

Das temporäre Erhöhen der Konfigurationsvariable `maintenance_work_mem` beim Laden großer Datenmengen kann die Performance verbessern. Es beschleunigt `CREATE INDEX`-Befehle und `ALTER TABLE ADD FOREIGN KEY`-Befehle. Für `COPY` selbst bewirkt es wenig; dieser Hinweis ist also nur nützlich, wenn Sie eine oder beide der oben genannten Techniken verwenden.

### 14.4.6. max_wal_size erhöhen

Auch das temporäre Erhöhen der Konfigurationsvariable `max_wal_size` kann große Ladevorgänge beschleunigen. Der Grund ist, dass das Laden großer Datenmengen in PostgreSQL Checkpoints häufiger auslöst als die normale Checkpoint-Frequenz, die durch die Konfigurationsvariable `checkpoint_timeout` angegeben wird. Bei jedem Checkpoint müssen alle geänderten Seiten auf die Platte geschrieben werden. Wenn `max_wal_size` während Massenladevorgängen temporär erhöht wird, kann die Zahl der erforderlichen Checkpoints reduziert werden.

### 14.4.7. WAL-Archivierung und Streaming-Replikation deaktivieren

Wenn große Datenmengen in eine Installation geladen werden, die WAL-Archivierung oder Streaming-Replikation verwendet, kann es schneller sein, nach Abschluss des Ladevorgangs eine neue Basissicherung zu erstellen, als eine große Menge inkrementeller WAL-Daten zu verarbeiten. Um inkrementelles WAL-Logging während des Ladens zu verhindern, deaktivieren Sie Archivierung und Streaming-Replikation, indem Sie `wal_level` auf `minimal`, `archive_mode` auf `off` und `max_wal_senders` auf null setzen. Beachten Sie aber, dass das Ändern dieser Einstellungen einen Serverneustart erfordert und alle zuvor erstellten Basissicherungen für Archivwiederherstellung und Standby-Server unbrauchbar macht, was zu Datenverlust führen kann.

Neben der eingesparten Zeit für Archiver oder WAL Sender macht dies bestimmte Befehle tatsächlich schneller, weil sie überhaupt kein WAL schreiben müssen, wenn `wal_level` auf `minimal` steht und die aktuelle Subtransaktion oder Top-Level-Transaktion die Tabelle oder den Index erstellt oder geleert hat, den sie verändert. Sie können Crash-Sicherheit dann günstiger durch ein `fsync` am Ende garantieren als durch WAL-Schreiben.

### 14.4.8. Anschließend ANALYZE ausführen

Immer wenn Sie die Datenverteilung innerhalb einer Tabelle erheblich verändert haben, wird dringend empfohlen, `ANALYZE` auszuführen. Dazu gehört auch das massenhafte Laden großer Datenmengen in eine Tabelle. `ANALYZE` oder `VACUUM ANALYZE` stellt sicher, dass der Planner aktuelle Statistiken über die Tabelle besitzt. Ohne Statistiken oder mit veralteten Statistiken kann der Planner schlechte Entscheidungen während der Abfrageplanung treffen, was zu schlechter Performance bei Tabellen mit ungenauen oder fehlenden Statistiken führt. Wenn der Autovacuum-Daemon aktiviert ist, kann er `ANALYZE` automatisch ausführen; weitere Informationen finden Sie in [Abschnitt 24.1.3](24_Regelmäßige_Datenbankwartung.md#2413-plannerstatistiken-aktualisieren) und [Abschnitt 24.1.6](24_Regelmäßige_Datenbankwartung.md#2416-der-autovacuumdaemon).

### 14.4.9. Einige Hinweise zu pg_dump

Von `pg_dump` erzeugte Dump-Skripte wenden mehrere der obigen Richtlinien automatisch an, aber nicht alle. Um einen `pg_dump`-Dump möglichst schnell wiederherzustellen, müssen Sie einige zusätzliche Dinge manuell tun. Diese Punkte gelten beim Wiederherstellen eines Dumps, nicht beim Erstellen. Sie gelten gleichermaßen beim Laden eines Text-Dumps mit `psql` und beim Laden aus einer `pg_dump`-Archivdatei mit `pg_restore`.

Standardmäßig verwendet `pg_dump` `COPY`, und wenn es einen vollständigen Schema-und-Daten-Dump erzeugt, lädt es Daten vor dem Erstellen von Indizes und Fremdschlüsseln. In diesem Fall werden also mehrere Richtlinien automatisch beachtet. Übrig bleibt:

- Setzen Sie passende, also größere als normale, Werte für `maintenance_work_mem` und `max_wal_size`.
- Wenn Sie WAL-Archivierung oder Streaming-Replikation verwenden, erwägen Sie, diese während der Wiederherstellung zu deaktivieren. Setzen Sie dazu vor dem Laden des Dumps `archive_mode` auf `off`, `wal_level` auf `minimal` und `max_wal_senders` auf null. Stellen Sie die Werte danach wieder korrekt ein und erstellen Sie eine frische Basissicherung.
- Experimentieren Sie mit den parallelen Dump- und Restore-Modi von `pg_dump` und `pg_restore`, und ermitteln Sie die optimale Zahl gleichzeitig laufender Jobs. Paralleles Dumpen und Wiederherstellen mit der Option `-j` sollte gegenüber dem seriellen Modus deutlich höhere Performance liefern.
- Überlegen Sie, ob der gesamte Dump als einzelne Transaktion wiederhergestellt werden soll. Übergeben Sie dazu die Kommandozeilenoption `-1` oder `--single-transaction` an `psql` oder `pg_restore`. In diesem Modus führt selbst der kleinste Fehler zum Rollback der gesamten Wiederherstellung und verwirft möglicherweise viele Stunden Verarbeitung. Je nach Verflechtung der Daten kann das gegenüber manueller Bereinigung vorzuziehen sein oder auch nicht. `COPY`-Befehle laufen am schnellsten, wenn Sie eine einzelne Transaktion verwenden und WAL-Archivierung deaktiviert ist.
- Wenn im Datenbankserver mehrere CPUs verfügbar sind, erwägen Sie die Option `--jobs` von `pg_restore`. Sie erlaubt paralleles Laden von Daten und parallele Indexerstellung.
- Führen Sie anschließend `ANALYZE` aus.

Ein reiner Datendump verwendet weiterhin `COPY`, löscht oder erstellt aber keine Indizes neu und berührt normalerweise keine Fremdschlüssel. Sie können die Wirkung deaktivierter Fremdschlüssel mit der Option `--disable-triggers` erzielen, müssen sich aber bewusst sein, dass dies die Fremdschlüsselvalidierung nicht nur aufschiebt, sondern beseitigt; es ist also möglich, fehlerhafte Daten einzufügen. Beim Laden eines reinen Datendumps liegt es daher bei Ihnen, Indizes und Fremdschlüssel zu löschen und neu zu erstellen, wenn Sie diese Techniken nutzen möchten. Es ist weiterhin nützlich, `max_wal_size` während des Ladens zu erhöhen, aber `maintenance_work_mem` brauchen Sie dafür nicht zu erhöhen; das wäre erst beim manuellen Neuerstellen von Indizes und Fremdschlüsseln danach sinnvoll. Vergessen Sie nicht, abschließend `ANALYZE` auszuführen; weitere Informationen finden Sie in [Abschnitt 24.1.3](24_Regelmäßige_Datenbankwartung.md#2413-plannerstatistiken-aktualisieren) und [Abschnitt 24.1.6](24_Regelmäßige_Datenbankwartung.md#2416-der-autovacuumdaemon).

## 14.5. Nicht dauerhafte Einstellungen

Dauerhaftigkeit ist eine Datenbankeigenschaft, die garantiert, dass bestätigte Transaktionen auch dann aufgezeichnet bleiben, wenn der Server abstürzt oder die Stromversorgung ausfällt. Dauerhaftigkeit verursacht jedoch erheblichen Datenbank-Overhead. Wenn Ihre Installation eine solche Garantie nicht benötigt, kann PostgreSQL für deutlich höhere Geschwindigkeit konfiguriert werden. Die folgenden Konfigurationsänderungen können in solchen Fällen die Performance verbessern. Soweit unten nicht anders angegeben, bleibt Dauerhaftigkeit bei einem Absturz der Datenbanksoftware erhalten; nur ein abrupter Absturz des Betriebssystems erzeugt bei Verwendung dieser Einstellungen das Risiko von Datenverlust oder Datenkorruption.

- Legen Sie das Datenverzeichnis des Datenbankclusters in ein speicherbasiertes Dateisystem, also eine RAM-Disk. Dadurch entfällt sämtliches Datenbank-I/O auf die Platte, aber der Datenspeicher ist auf den verfügbaren Arbeitsspeicher und gegebenenfalls Swap begrenzt.
- Schalten Sie `fsync` aus; dann müssen Daten nicht auf die Platte geschrieben werden.
- Schalten Sie `synchronous_commit` aus; dann ist es möglicherweise nicht nötig, WAL-Schreibvorgänge bei jedem Commit auf die Platte zu erzwingen. Diese Einstellung riskiert bei einem Datenbankabsturz den Verlust von Transaktionen, aber keine Datenkorruption.
- Schalten Sie `full_page_writes` aus; dann muss nicht gegen teilweise geschriebene Seiten abgesichert werden.
- Erhöhen Sie `max_wal_size` und `checkpoint_timeout`; das reduziert die Häufigkeit von Checkpoints, erhöht aber den Speicherbedarf von `pg_wal`.
- Erstellen Sie nicht protokollierte Tabellen, um WAL-Schreibvorgänge zu vermeiden. Dadurch sind diese Tabellen jedoch nicht crash-sicher.
