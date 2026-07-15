# 69. Wie der Planner Statistiken verwendet

Dieses Kapitel baut auf dem Material aus [Abschnitt 14.1](14_Hinweise_zur_Performance.md#141-explain-verwenden) und [Abschnitt 14.2](14_Hinweise_zur_Performance.md#142-vom-planner-verwendete-statistiken) auf und zeigt einige zusätzliche Details dazu, wie der Planner Systemstatistiken verwendet, um die Anzahl der Zeilen zu schätzen, die jeder Teil einer Abfrage voraussichtlich zurückgibt. Das ist ein wichtiger Teil des Planungsprozesses, weil es viel Rohmaterial für die Kostenberechnung liefert.

Die Absicht dieses Kapitels ist nicht, den Code im Detail zu dokumentieren, sondern einen Überblick darüber zu geben, wie er funktioniert. Das kann die Lernkurve erleichtern, wenn jemand anschließend den Code lesen möchte.

## 69.1. Beispiele zur Zeilenschätzung

Die unten gezeigten Beispiele verwenden Tabellen aus der PostgreSQL-Regressions-Testdatenbank. Beachten Sie außerdem, dass `ANALYZE` beim Erzeugen von Statistiken Stichproben verwendet; die Ergebnisse ändern sich daher nach jedem neuen `ANALYZE` leicht.

Beginnen wir mit einer sehr einfachen Abfrage:

```sql
EXPLAIN SELECT * FROM tenk1;
```

```text
                         QUERY PLAN
-------------------------------------------------------------
 Seq Scan on tenk1 (cost=0.00..458.00 rows=10000 width=244)
```

Wie der Planner die Kardinalität von `tenk1` bestimmt, wurde in [Abschnitt 14.2](14_Hinweise_zur_Performance.md#142-vom-planner-verwendete-statistiken) behandelt, wird hier der Vollständigkeit halber aber wiederholt. Die Anzahl der Seiten und Zeilen wird in `pg_class` nachgeschlagen:

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'tenk1';
```

```text
 relpages | reltuples
----------+-----------
      358 |     10000
```

Diese Zahlen sind auf dem Stand des letzten `VACUUM` oder `ANALYZE` auf der Tabelle. Danach holt der Planner die tatsächliche aktuelle Anzahl von Seiten in der Tabelle. Das ist eine billige Operation und erfordert keinen Tabellenscan. Wenn sie von `relpages` abweicht, wird `reltuples` entsprechend skaliert, um eine aktuelle Schätzung der Zeilenzahl zu erhalten. Im obigen Beispiel ist der Wert von `relpages` aktuell, sodass die Zeilenschätzung identisch mit `reltuples` ist.

Betrachten wir nun ein Beispiel mit einer Bereichsbedingung in der `WHERE`-Klausel:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 1000;
```

```text
                                    QUERY PLAN
--------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1 (cost=24.06..394.64 rows=1007 width=244)
   Recheck Cond: (unique1 < 1000)
   -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..23.80 rows=1007 width=0)
         Index Cond: (unique1 < 1000)
```

Der Planner untersucht die Bedingung in der `WHERE`-Klausel und schlägt die Selektivitätsfunktion für den Operator `<` in `pg_operator` nach. Sie steht in der Spalte `oprrest`; der Eintrag ist in diesem Fall `scalarltsel`. Die Funktion `scalarltsel` holt das Histogramm für `unique1` aus `pg_statistic`. Für manuelle Abfragen ist es bequemer, in die einfachere View `pg_stats` zu schauen:

```sql
SELECT histogram_bounds FROM pg_stats
WHERE tablename='tenk1' AND attname='unique1';
```

```text
                   histogram_bounds
------------------------------------------------------
 {0,993,1997,3050,4040,5036,5957,7057,8029,9016,9995}
```

Als Nächstes wird bestimmt, welcher Anteil des Histogramms von „`< 1000`“ belegt wird. Das ist die Selektivität. Das Histogramm teilt den Bereich in Buckets gleicher Häufigkeit auf, daher muss nur der Bucket gefunden werden, in dem der Wert liegt, und dann werden dieser Bucket anteilig sowie alle vorhergehenden Buckets gezählt. Der Wert 1000 liegt eindeutig im zweiten Bucket (993 bis 1997). Bei angenommener linearer Verteilung der Werte innerhalb jedes Buckets kann die Selektivität so berechnet werden:

```text
selectivity = (1 + (1000 - bucket[2].min)/(bucket[2].max - bucket[2].min))/num_buckets
            = (1 + (1000 - 993)/(1997 - 993))/10
            = 0.100697
```

Das heißt: ein ganzer Bucket plus ein linearer Anteil des zweiten Buckets, geteilt durch die Anzahl der Buckets. Die geschätzte Anzahl von Zeilen kann nun als Produkt aus Selektivität und Kardinalität von `tenk1` berechnet werden:

```text
rows = rel_cardinality * selectivity
     = 10000 * 0.100697
     = 1007 (gerundet)
```

Nun betrachten wir ein Beispiel mit einer Gleichheitsbedingung in der `WHERE`-Klausel:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE stringu1 = 'CRAAAA';
```

```text
                        QUERY PLAN
----------------------------------------------------------
 Seq Scan on tenk1 (cost=0.00..483.00 rows=30 width=244)
   Filter: (stringu1 = 'CRAAAA'::name)
```

Wieder untersucht der Planner die Bedingung in der `WHERE`-Klausel und schlägt die Selektivitätsfunktion für `=` nach, hier `eqsel`. Für Gleichheitsschätzungen ist das Histogramm nicht nützlich; stattdessen wird die Liste der häufigsten Werte (Most Common Values, MCVs) verwendet. Sehen wir uns die MCVs mit einigen zusätzlichen Spalten an, die später nützlich sind:

```sql
SELECT null_frac, n_distinct, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename='tenk1' AND attname='stringu1';
```

```text
null_frac         | 0
n_distinct        | 676
most_common_vals  | {EJAAAA,BBAAAA,CRAAAA,FCAAAA,FEAAAA,GSAAAA,JOAAAA,MCAAAA,NAAAAA,WGAAAA}
most_common_freqs | {0.00333333,0.003,0.003,0.003,0.003,0.003,0.003,0.003,0.003,0.003}
```

Da `CRAAAA` in der MCV-Liste vorkommt, ist die Selektivität einfach der entsprechende Eintrag in der Liste der Häufigkeiten der häufigsten Werte:

```text
selectivity = mcf[3]
            = 0.003
```

Wie zuvor ist die geschätzte Anzahl von Zeilen nur das Produkt daraus und aus der Kardinalität von `tenk1`:

```text
rows = 10000 * 0.003
     = 30
```

Betrachten wir nun dieselbe Abfrage, aber mit einer Konstanten, die nicht in der MCV-Liste steht:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE stringu1 = 'xxx';
```

```text
                        QUERY PLAN
----------------------------------------------------------
 Seq Scan on tenk1 (cost=0.00..483.00 rows=15 width=244)
   Filter: (stringu1 = 'xxx'::name)
```

Das ist ein anderes Problem: Wie wird die Selektivität geschätzt, wenn der Wert nicht in der MCV-Liste steht? Der Ansatz nutzt die Tatsache, dass der Wert nicht in der Liste steht, zusammen mit den bekannten Häufigkeiten aller MCVs:

```text
selectivity = (1 - sum(mcv_freqs))/(num_distinct - num_mcv)
            = (1 - (0.00333333 + 0.003 + 0.003 + 0.003 + 0.003 + 0.003 +
                    0.003 + 0.003 + 0.003 + 0.003))/(676 - 10)
            = 0.0014559
```

Es werden also alle Häufigkeiten der MCVs addiert und von eins abgezogen; anschließend wird durch die Anzahl der übrigen unterschiedlichen Werte geteilt. Das entspricht der Annahme, dass der Anteil der Spalte, der keiner der MCVs ist, gleichmäßig über alle anderen unterschiedlichen Werte verteilt ist. Da es hier keine Nullwerte gibt, müssen sie nicht berücksichtigt werden; andernfalls würde die Null-Fraktion ebenfalls vom Zähler abgezogen. Die geschätzte Anzahl von Zeilen wird wie üblich berechnet:

```text
rows = 10000 * 0.0014559
     = 15 (gerundet)
```

Das vorherige Beispiel mit `unique1 < 1000` war eine Vereinfachung dessen, was `scalarltsel` wirklich tut. Nachdem wir ein Beispiel für die Verwendung von MCVs gesehen haben, können wir weitere Details ergänzen. Das Beispiel war so weit korrekt, weil `unique1` als eindeutige Spalte keine MCVs hat; offensichtlich ist kein Wert häufiger als ein anderer. Für eine nicht eindeutige Spalte gibt es normalerweise sowohl ein Histogramm als auch eine MCV-Liste, und das Histogramm enthält nicht den Teil der Spaltenpopulation, der durch die MCVs repräsentiert wird. Das wird so gemacht, weil es genauere Schätzungen erlaubt. In dieser Situation wendet `scalarltsel` die Bedingung, etwa „`< 1000`“, direkt auf jeden Wert der MCV-Liste an und addiert die Häufigkeiten der MCVs, für die die Bedingung wahr ist. Das ergibt eine exakte Schätzung der Selektivität innerhalb des Teils der Tabelle, der aus MCVs besteht. Das Histogramm wird danach wie oben verwendet, um die Selektivität in dem Teil der Tabelle zu schätzen, der nicht aus MCVs besteht, und die beiden Zahlen werden kombiniert, um die Gesamtselektivität zu schätzen. Betrachten wir zum Beispiel:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE stringu1 < 'IAAAAA';
```

```text
                         QUERY PLAN
------------------------------------------------------------
 Seq Scan on tenk1 (cost=0.00..483.00 rows=3077 width=244)
   Filter: (stringu1 < 'IAAAAA'::name)
```

Die MCV-Informationen für `stringu1` kennen wir bereits; hier ist sein Histogramm:

```sql
SELECT histogram_bounds FROM pg_stats
WHERE tablename='tenk1' AND attname='stringu1';
```

```text
                                histogram_bounds
--------------------------------------------------------------------------------
 {AAAAAA,CQAAAA,FRAAAA,IBAAAA,KRAAAA,NFAAAA,PSAAAA,SGAAAA,VAAAAA,XLAAAA,ZZAAAA}
```

Beim Prüfen der MCV-Liste ergibt sich, dass die Bedingung `stringu1 < 'IAAAAA'` von den ersten sechs Einträgen erfüllt wird, nicht aber von den letzten vier. Die Selektivität innerhalb des MCV-Teils der Population ist daher:

```text
selectivity = sum(relevant mcfs)
            = 0.00333333 + 0.003 + 0.003 + 0.003 + 0.003 + 0.003
            = 0.01833333
```

Die Summe aller MCFs zeigt außerdem, dass der gesamte durch MCVs repräsentierte Anteil der Population `0.03033333` ist. Der durch das Histogramm repräsentierte Anteil beträgt daher `0.96966667`; wieder gibt es keine Nullwerte, die sonst hier ausgeschlossen werden müssten. Der Wert `IAAAAA` liegt fast am Ende des dritten Histogramm-Buckets. Mit einigen eher groben Annahmen über die Häufigkeit verschiedener Zeichen gelangt der Planner zur Schätzung `0.298387` für den Anteil der Histogrammpopulation, der kleiner als `IAAAAA` ist. Danach werden die Schätzungen für MCV- und Nicht-MCV-Population kombiniert:

```text
selectivity = mcv_selectivity + histogram_selectivity * histogram_fraction
            = 0.01833333 + 0.298387 * 0.96966667
            = 0.307669

rows        = 10000 * 0.307669
            = 3077 (gerundet)
```

In diesem konkreten Beispiel ist die Korrektur durch die MCV-Liste recht klein, weil die Spaltenverteilung tatsächlich ziemlich flach ist; dass die Statistiken diese bestimmten Werte als häufiger zeigen, liegt überwiegend an Stichprobenfehlern. In einem typischeren Fall, in dem einige Werte deutlich häufiger sind als andere, liefert dieses aufwendigere Verfahren eine nützliche Genauigkeitsverbesserung, weil die Selektivität der häufigsten Werte exakt bestimmt wird.

Nun betrachten wir einen Fall mit mehr als einer Bedingung in der `WHERE`-Klausel:

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 1000 AND stringu1 = 'xxx';
```

```text
                                    QUERY PLAN
--------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1 (cost=23.80..396.91 rows=1 width=244)
   Recheck Cond: (unique1 < 1000)
   Filter: (stringu1 = 'xxx'::name)
   -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..23.80 rows=1007 width=0)
         Index Cond: (unique1 < 1000)
```

Der Planner nimmt an, dass die beiden Bedingungen unabhängig sind, sodass die einzelnen Selektivitäten der Klauseln miteinander multipliziert werden können:

```text
selectivity = selectivity(unique1 < 1000) * selectivity(stringu1 = 'xxx')
            = 0.100697 * 0.0014559
            = 0.0001466

rows        = 10000 * 0.0001466
            = 1 (gerundet)
```

Beachten Sie, dass die geschätzte Zeilenzahl des Bitmap Index Scan nur die Bedingung widerspiegelt, die mit dem Index verwendet wird. Das ist wichtig, weil es die Kostenschätzung für die anschließenden Heap-Zugriffe beeinflusst.

Schließlich untersuchen wir eine Abfrage mit einem Join:

```sql
EXPLAIN SELECT * FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 50 AND t1.unique2 = t2.unique2;
```

```text
                                      QUERY PLAN
-----------------------------------------------------------------------------------
 Nested Loop (cost=4.64..456.23 rows=50 width=488)
   -> Bitmap Heap Scan on tenk1 t1 (cost=4.64..142.17 rows=50 width=244)
         Recheck Cond: (unique1 < 50)
         -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..4.63 rows=50 width=0)
               Index Cond: (unique1 < 50)
   -> Index Scan using tenk2_unique2 on tenk2 t2 (cost=0.00..6.27 rows=1 width=244)
         Index Cond: (unique2 = t1.unique2)
```

Die Restriktion auf `tenk1`, `unique1 < 50`, wird vor dem Nested-Loop-Join ausgewertet. Das wird analog zum vorherigen Bereichsbeispiel behandelt. Diesmal fällt der Wert 50 in den ersten Bucket des Histogramms von `unique1`:

```text
selectivity = (0 + (50 - bucket[1].min)/(bucket[1].max - bucket[1].min))/num_buckets
            = (0 + (50 - 0)/(993 - 0))/10
            = 0.005035

rows        = 10000 * 0.005035
            = 50 (gerundet)
```

Die Restriktion für den Join lautet `t2.unique2 = t1.unique2`. Der Operator ist das vertraute `=`, aber die Selektivitätsfunktion wird aus der Spalte `oprjoin` von `pg_operator` geholt und heißt `eqjoinsel`. `eqjoinsel` schlägt die statistischen Informationen für `tenk2` und `tenk1` nach:

```sql
SELECT tablename, null_frac, n_distinct, most_common_vals
FROM pg_stats
WHERE tablename IN ('tenk1', 'tenk2') AND attname='unique2';
```

```text
 tablename | null_frac | n_distinct | most_common_vals
-----------+-----------+------------+------------------
 tenk1     |         0 |         -1 |
 tenk2     |         0 |         -1 |
```

In diesem Fall gibt es keine MCV-Informationen für `unique2`, und alle Werte scheinen eindeutig zu sein (`n_distinct = -1`). Daher wird ein Algorithmus verwendet, der sich auf die Zeilenzahlschätzungen für beide Relationen (`num_rows`, nicht gezeigt, hier jeweils 10000) und die Null-Fraktionen der Spalten stützt, die beide null sind:

```text
selectivity = (1 - null_frac1) * (1 - null_frac2) / max(num_rows1, num_rows2)
            = (1 - 0) * (1 - 0) / max(10000, 10000)
            = 0.0001
```

Die Null-Fraktion wird also für jede der Relationen von eins abgezogen, und das Ergebnis wird durch die Zeilenzahl der größeren Relation geteilt. Im nicht eindeutigen Fall wird dieser Wert skaliert. Die Anzahl von Zeilen, die der Join voraussichtlich ausgibt, wird als Kardinalität des kartesischen Produkts der beiden Eingaben multipliziert mit der Selektivität berechnet:

```text
rows = (outer_cardinality * inner_cardinality) * selectivity
     = (50 * 10000) * 0.0001
     = 50
```

Wenn es für die beiden Spalten MCV-Listen gegeben hätte, hätte `eqjoinsel` die MCV-Listen direkt verglichen, um die Join-Selektivität innerhalb des durch MCVs repräsentierten Teils der Spaltenpopulationen zu bestimmen. Die Schätzung für den Rest der Populationen folgt demselben Ansatz wie hier gezeigt.

Beachten Sie, dass `inner_cardinality` als 10000 gezeigt wurde, also als unveränderte Größe von `tenk2`. Aus der Betrachtung der `EXPLAIN`-Ausgabe könnte es scheinen, als käme die Schätzung der Join-Zeilen aus `50 * 1`, also aus der Zahl äußerer Zeilen mal der geschätzten Zahl von Zeilen, die durch jeden inneren Index Scan auf `tenk2` erhalten werden. Das ist jedoch nicht der Fall: Die Größe der Join-Relation wird geschätzt, bevor irgendein bestimmter Join-Plan betrachtet wurde. Wenn alles gut funktioniert, liefern beide Arten, die Join-Größe zu schätzen, etwa dieselbe Antwort; wegen Rundungsfehlern und anderer Faktoren können sie jedoch manchmal deutlich auseinanderlaufen.

Für weitere Details: Die Schätzung der Größe einer Tabelle vor `WHERE`-Klauseln geschieht in `src/backend/optimizer/util/plancat.c`. Die generische Logik für Klauselselektivitäten steht in `src/backend/optimizer/path/clausesel.c`. Operatorspezifische Selektivitätsfunktionen finden sich größtenteils in `src/backend/utils/adt/selfuncs.c`.

## 69.2. Beispiele zu multivariaten Statistiken

### 69.2.1. Funktionale Abhängigkeiten

Multivariate Korrelation lässt sich mit einem sehr einfachen Datensatz zeigen: einer Tabelle mit zwei Spalten, die beide dieselben Werte enthalten.

```sql
CREATE TABLE t (a INT, b INT);
INSERT INTO t SELECT i % 100, i % 100 FROM generate_series(1, 10000) s(i);
ANALYZE t;
```

Wie in [Abschnitt 14.2](14_Hinweise_zur_Performance.md#142-vom-planner-verwendete-statistiken) erklärt, kann der Planner die Kardinalität von `t` anhand der aus `pg_class` erhaltenen Anzahl von Seiten und Zeilen bestimmen:

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 't';
```

```text
 relpages | reltuples
----------+-----------
       45 |     10000
```

Die Datenverteilung ist sehr einfach: In jeder Spalte gibt es nur 100 unterschiedliche Werte, gleichmäßig verteilt.

Das folgende Beispiel zeigt das Ergebnis der Schätzung einer `WHERE`-Bedingung auf der Spalte `a`:

```sql
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT * FROM t WHERE a = 1;
```

```text
                                 QUERY PLAN
----------------------------------------------------------------------------
 Seq Scan on t (cost=0.00..170.00 rows=100 width=8) (actual rows=100.00 loops=1)
   Filter: (a = 1)
   Rows Removed by Filter: 9900
```

Der Planner untersucht die Bedingung und bestimmt die Selektivität dieser Klausel zu 1 %. Der Vergleich dieser Schätzung mit der tatsächlichen Zeilenzahl zeigt, dass die Schätzung sehr genau ist, tatsächlich exakt, weil die Tabelle sehr klein ist. Wenn die `WHERE`-Bedingung stattdessen die Spalte `b` verwendet, wird ein identischer Plan erzeugt. Beobachten Sie jedoch, was passiert, wenn dieselbe Bedingung auf beide Spalten angewendet und mit `AND` kombiniert wird:

```sql
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT * FROM t WHERE a = 1 AND b = 1;
```

```text
                                QUERY PLAN
--------------------------------------------------------------------------
 Seq Scan on t (cost=0.00..195.00 rows=1 width=8) (actual rows=100.00 loops=1)
   Filter: ((a = 1) AND (b = 1))
   Rows Removed by Filter: 9900
```

Der Planner schätzt die Selektivität für jede Bedingung einzeln und kommt jeweils wieder auf 1 %. Danach nimmt er an, dass die Bedingungen unabhängig sind, und multipliziert ihre Selektivitäten. Daraus entsteht eine endgültige Selektivitätsschätzung von nur 0,01 %. Das ist eine deutliche Unterschätzung, denn die tatsächliche Anzahl passender Zeilen, 100, ist zwei Größenordnungen höher.

Dieses Problem lässt sich beheben, indem ein Statistikobjekt erzeugt wird, das `ANALYZE` anweist, multivariate Statistiken zu funktionalen Abhängigkeiten für die beiden Spalten zu berechnen:

```sql
CREATE STATISTICS stts (dependencies) ON a, b FROM t;
ANALYZE t;
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT * FROM t WHERE a = 1 AND b = 1;
```

```text
                                 QUERY PLAN
----------------------------------------------------------------------------
 Seq Scan on t (cost=0.00..195.00 rows=100 width=8) (actual rows=100.00 loops=1)
   Filter: ((a = 1) AND (b = 1))
   Rows Removed by Filter: 9900
```

### 69.2.2. Multivariate N-Distinct-Zählungen

Ein ähnliches Problem tritt bei der Schätzung der Kardinalität von Mengen mehrerer Spalten auf, zum Beispiel der Anzahl von Gruppen, die durch eine `GROUP BY`-Klausel erzeugt würden. Wenn `GROUP BY` eine einzelne Spalte auflistet, ist die N-Distinct-Schätzung, sichtbar als geschätzte Anzahl von Zeilen, die vom Knoten `HashAggregate` zurückgegeben werden, sehr genau:

```sql
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT COUNT(*) FROM t GROUP BY a;
```

```text
                                       QUERY PLAN
----------------------------------------------------------------------------------------
 HashAggregate (cost=195.00..196.00 rows=100 width=12) (actual rows=100.00 loops=1)
   Group Key: a
   -> Seq Scan on t (cost=0.00..145.00 rows=10000 width=4) (actual rows=10000.00 loops=1)
```

Ohne multivariate Statistiken liegt die Schätzung der Anzahl von Gruppen in einer Abfrage mit zwei Spalten in `GROUP BY`, wie im folgenden Beispiel, jedoch um eine Größenordnung daneben:

```sql
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT COUNT(*) FROM t GROUP BY a, b;
```

```text
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 HashAggregate (cost=220.00..230.00 rows=1000 width=16) (actual rows=100.00 loops=1)
   Group Key: a, b
   -> Seq Scan on t (cost=0.00..145.00 rows=10000 width=8) (actual rows=10000.00 loops=1)
```

Wenn das Statistikobjekt neu definiert wird, sodass es N-Distinct-Zählungen für die beiden Spalten enthält, verbessert sich die Schätzung deutlich:

```sql
DROP STATISTICS stts;
CREATE STATISTICS stts (dependencies, ndistinct) ON a, b FROM t;
ANALYZE t;
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT COUNT(*) FROM t GROUP BY a, b;
```

```text
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 HashAggregate (cost=220.00..221.00 rows=100 width=16) (actual rows=100.00 loops=1)
   Group Key: a, b
   -> Seq Scan on t (cost=0.00..145.00 rows=10000 width=8) (actual rows=10000.00 loops=1)
```

### 69.2.3. MCV-Listen

Wie in [Abschnitt 69.2.1](69_Wie_der_Planner_Statistiken_verwendet.md#6921-funktionale-abhängigkeiten) erklärt, sind funktionale Abhängigkeiten eine sehr billige und effiziente Art von Statistik. Ihre wichtigste Einschränkung ist jedoch ihre globale Natur: Sie verfolgen nur Abhängigkeiten auf Spaltenebene, nicht zwischen einzelnen Spaltenwerten.

Dieser Abschnitt stellt die multivariate Variante von MCV-Listen (Most Common Values) vor, eine einfache Erweiterung der in [Abschnitt 69.1](69_Wie_der_Planner_Statistiken_verwendet.md#691-beispiele-zur-zeilenschätzung) beschriebenen spaltenweisen Statistiken. Diese Statistiken beheben die Einschränkung, indem sie einzelne Werte speichern, sind aber naturgemäß teurer, sowohl beim Aufbau der Statistiken in `ANALYZE` als auch hinsichtlich Speicherung und Planungszeit.

Sehen wir uns die Abfrage aus [Abschnitt 69.2.1](69_Wie_der_Planner_Statistiken_verwendet.md#6921-funktionale-abhängigkeiten) noch einmal an, diesmal aber mit einer MCV-Liste auf derselben Spaltenmenge. Löschen Sie dabei die funktionalen Abhängigkeiten, damit der Planner die neu erzeugten Statistiken verwendet.

```sql
DROP STATISTICS stts;
CREATE STATISTICS stts2 (mcv) ON a, b FROM t;
ANALYZE t;
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT * FROM t WHERE a = 1 AND b = 1;
```

```text
                                 QUERY PLAN
----------------------------------------------------------------------------
 Seq Scan on t (cost=0.00..195.00 rows=100 width=8) (actual rows=100.00 loops=1)
   Filter: ((a = 1) AND (b = 1))
   Rows Removed by Filter: 9900
```

Die Schätzung ist so genau wie mit funktionalen Abhängigkeiten, vor allem weil die Tabelle ziemlich klein ist und eine einfache Verteilung mit einer geringen Anzahl unterschiedlicher Werte hat. Bevor wir die zweite Abfrage betrachten, mit der funktionale Abhängigkeiten nicht besonders gut umgehen konnten, sehen wir uns die MCV-Liste genauer an.

Die MCV-Liste kann mit der set-returning Funktion `pg_mcv_list_items` untersucht werden.

```sql
SELECT m.*
FROM pg_statistic_ext
JOIN pg_statistic_ext_data ON (oid = stxoid),
     pg_mcv_list_items(stxdmcv) m
WHERE stxname = 'stts2';
```

```text
 index | values   | nulls | frequency | base_frequency
-------+----------+-------+-----------+----------------
     0 | {0, 0}   | {f,f} |      0.01 |         0.0001
     1 | {1, 1}   | {f,f} |      0.01 |         0.0001
   ...
    49 | {49, 49} | {f,f} |      0.01 |         0.0001
    50 | {50, 50} | {f,f} |      0.01 |         0.0001
   ...
    97 | {97, 97} | {f,f} |      0.01 |         0.0001
    98 | {98, 98} | {f,f} |      0.01 |         0.0001
    99 | {99, 99} | {f,f} |      0.01 |         0.0001
(100 rows)
```

Das bestätigt, dass es 100 unterschiedliche Kombinationen in den beiden Spalten gibt und dass alle ungefähr gleich wahrscheinlich sind, mit 1 % Häufigkeit für jede. Die Basisfrequenz ist die Häufigkeit, die aus spaltenweisen Statistiken berechnet würde, als gäbe es keine Mehrspaltenstatistiken. Wenn es Nullwerte in einer der Spalten gegeben hätte, wäre das in der Spalte `nulls` erkennbar.

Beim Schätzen der Selektivität wendet der Planner alle Bedingungen auf die Items in der MCV-Liste an und summiert danach die Häufigkeiten der passenden Items. Details stehen in `mcv_clauselist_selectivity` in `src/backend/statistics/mcv.c`.

Im Vergleich zu funktionalen Abhängigkeiten haben MCV-Listen zwei große Vorteile. Erstens speichert die Liste tatsächliche Werte, sodass entschieden werden kann, welche Kombinationen kompatibel sind.

```sql
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT * FROM t WHERE a = 1 AND b = 10;
```

```text
                                QUERY PLAN
------------------------------------------------------------------------
 Seq Scan on t (cost=0.00..195.00 rows=1 width=8) (actual rows=0.00 loops=1)
   Filter: ((a = 1) AND (b = 10))
   Rows Removed by Filter: 10000
```

Zweitens behandeln MCV-Listen eine größere Bandbreite von Klauseltypen, nicht nur Gleichheitsklauseln wie funktionale Abhängigkeiten. Betrachten Sie zum Beispiel die folgende Bereichsabfrage für dieselbe Tabelle:

```sql
EXPLAIN (ANALYZE, TIMING OFF, BUFFERS OFF)
SELECT * FROM t WHERE a <= 49 AND b > 49;
```

```text
                                QUERY PLAN
------------------------------------------------------------------------
 Seq Scan on t (cost=0.00..195.00 rows=1 width=8) (actual rows=0.00 loops=1)
   Filter: ((a <= 49) AND (b > 49))
   Rows Removed by Filter: 10000
```

## 69.3. Planner-Statistiken und Sicherheit

Der Zugriff auf die Tabelle `pg_statistic` ist auf Superuser beschränkt, damit gewöhnliche Benutzer daraus nichts über den Inhalt der Tabellen anderer Benutzer erfahren können. Einige Selektivitätsschätzfunktionen verwenden einen vom Benutzer bereitgestellten Operator, entweder den in der Abfrage vorkommenden Operator oder einen verwandten Operator, um die gespeicherten Statistiken zu analysieren. Um zum Beispiel zu bestimmen, ob ein gespeicherter häufigster Wert anwendbar ist, muss der Selektivitätsschätzer den passenden Operator `=` ausführen, um die Konstante in der Abfrage mit dem gespeicherten Wert zu vergleichen. Dadurch werden die Daten in `pg_statistic` potenziell an benutzerdefinierte Operatoren weitergegeben. Ein entsprechend konstruierter Operator kann die übergebenen Operanden absichtlich preisgeben, etwa indem er sie loggt oder in eine andere Tabelle schreibt, oder sie versehentlich preisgeben, indem er ihre Werte in Fehlermeldungen zeigt. In beiden Fällen könnten Daten aus `pg_statistic` einem Benutzer offengelegt werden, der sie nicht sehen dürfen sollte.

Um das zu verhindern, gilt für alle eingebauten Selektivitätsschätzfunktionen Folgendes: Beim Planen einer Abfrage muss der aktuelle Benutzer, damit gespeicherte Statistiken verwendet werden können, entweder `SELECT`-Rechte auf der Tabelle oder den beteiligten Spalten haben, oder der verwendete Operator muss `LEAKPROOF` sein, genauer gesagt die Funktion, auf der der Operator basiert. Andernfalls verhält sich der Selektivitätsschätzer so, als wären keine Statistiken verfügbar, und der Planner fährt mit Standard- oder Fallback-Annahmen fort. Der Metabefehl `\do+` des Programms `psql` ist nützlich, um zu bestimmen, welche Operatoren als leakproof markiert sind.

Wenn ein Benutzer nicht die erforderlichen Rechte auf der Tabelle oder den Spalten hat, erhält die Abfrage in vielen Fällen letztlich einen Permission-denied-Fehler; dann ist dieser Mechanismus in der Praxis unsichtbar. Wenn der Benutzer jedoch aus einer Security-Barrier-View liest, möchte der Planner möglicherweise Statistiken einer zugrunde liegenden Tabelle prüfen, die dem Benutzer sonst nicht zugänglich ist. In diesem Fall sollte der Operator leakproof sein, sonst werden die Statistiken nicht verwendet. Es gibt dazu keine direkte Rückmeldung, außer dass der Plan möglicherweise suboptimal ist. Wenn man vermutet, dass dies der Fall ist, kann man versuchen, die Abfrage als privilegierterer Benutzer auszuführen, um zu sehen, ob ein anderer Plan entsteht.

Diese Einschränkung gilt nur für Fälle, in denen der Planner einen benutzerdefinierten Operator auf einem oder mehreren Werten aus `pg_statistic` ausführen müsste. Der Planner darf daher generische statistische Informationen, etwa den Anteil von Nullwerten oder die Anzahl unterschiedlicher Werte in einer Spalte, unabhängig von Zugriffsrechten verwenden.

Selektivitätsschätzfunktionen in Erweiterungen von Drittanbietern, die potenziell mit benutzerdefinierten Operatoren auf Statistiken arbeiten, sollten denselben Sicherheitsregeln folgen. Ziehen Sie den PostgreSQL-Quellcode als Orientierung heran.
