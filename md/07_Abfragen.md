# 7. Abfragen

Die vorigen Kapitel haben erklärt, wie Tabellen erstellt, mit Daten gefüllt und geändert werden. Nun geht es endlich darum, Daten aus der Datenbank abzurufen.

## 7.1. Überblick

Der Vorgang des Abrufens von Daten aus einer Datenbank oder der Befehl dafür heißt Abfrage. In SQL wird der Befehl `SELECT` verwendet, um Abfragen zu formulieren. Die allgemeine Syntax eines `SELECT`-Befehls lautet:

```sql
[WITH with_queries] SELECT select_list FROM table_expression
 [sort_specification]
```

Die folgenden Abschnitte beschreiben die Details der Auswahlliste, des Tabellenausdrucks und der Sortierspezifikation. `WITH`-Abfragen werden zuletzt behandelt, da sie ein fortgeschrittenes Feature sind.

Eine einfache Abfrage hat diese Form:

```sql
SELECT * FROM table1;
```

Angenommen, es gibt eine Tabelle namens `table1`, dann ruft dieser Befehl alle Zeilen und alle benutzerdefinierten Spalten aus `table1` ab. Wie die Daten angezeigt werden, hängt von der Client-Anwendung ab. Das Programm `psql` zeigt beispielsweise eine ASCII-Tabelle auf dem Bildschirm an, während Client-Bibliotheken Funktionen zum Extrahieren einzelner Werte aus dem Abfrageergebnis anbieten. Die Angabe `*` in der Auswahlliste bedeutet alle Spalten, die der Tabellenausdruck bereitstellt. Eine Auswahlliste kann auch nur eine Teilmenge der verfügbaren Spalten auswählen oder Berechnungen mit Spalten durchführen. Wenn `table1` beispielsweise die Spalten `a`, `b` und `c` hat, können Sie schreiben:

```sql
SELECT a, b + c FROM table1;
```

Dabei wird vorausgesetzt, dass `b` und `c` numerische Datentypen haben. Weitere Details finden Sie in [Abschnitt 7.3](07_Abfragen.md#73-auswahllisten).

`FROM table1` ist eine einfache Form eines Tabellenausdrucks: Es wird nur eine Tabelle gelesen. Allgemein können Tabellenausdrücke komplexe Konstrukte aus Basistabellen, Joins und Unterabfragen sein. Sie können den Tabellenausdruck aber auch ganz weglassen und `SELECT` als Taschenrechner verwenden:

```sql
SELECT 3 * 4;
```

Das ist nützlicher, wenn die Ausdrücke in der Auswahlliste veränderliche Ergebnisse liefern. Zum Beispiel können Sie auf diese Weise eine Funktion aufrufen:

```sql
SELECT random();
```

## 7.2. Tabellenausdrücke

Ein Tabellenausdruck berechnet eine Tabelle. Er enthält eine `FROM`-Klausel, der optional `WHERE`-, `GROUP BY`- und `HAVING`-Klauseln folgen. Einfache Tabellenausdrücke verweisen nur auf eine Tabelle auf dem Datenträger, eine sogenannte Basistabelle. Komplexere Ausdrücke können Basistabellen aber auf verschiedene Weise verändern oder kombinieren.

Die optionalen Klauseln `WHERE`, `GROUP BY` und `HAVING` beschreiben eine Pipeline aufeinanderfolgender Transformationen der Tabelle, die in der `FROM`-Klausel gebildet wurde. Alle diese Transformationen erzeugen eine virtuelle Tabelle, deren Zeilen anschließend an die Auswahlliste weitergegeben werden, um die Ausgabezeilen der Abfrage zu berechnen.

### 7.2.1. Die `FROM`-Klausel

Die `FROM`-Klausel leitet eine Tabelle aus einer oder mehreren anderen Tabellen ab, die in einer kommaseparierten Liste von Tabellenreferenzen angegeben werden:

```sql
FROM table_reference [, table_reference [, ...]]
```

Eine Tabellenreferenz kann ein Tabellenname sein, gegebenenfalls schemaqualifiziert, oder eine abgeleitete Tabelle wie eine Unterabfrage, ein `JOIN`-Konstrukt oder eine komplexe Kombination daraus. Wenn in der `FROM`-Klausel mehr als eine Tabellenreferenz angegeben ist, werden die Tabellen per Cross Join verbunden, das heißt, es wird das kartesische Produkt ihrer Zeilen gebildet. Das Ergebnis der `FROM`-Liste ist eine virtuelle Zwischentabelle, auf die anschließend `WHERE`, `GROUP BY` und `HAVING` angewendet werden können; sie ist schließlich das Ergebnis des gesamten Tabellenausdrucks.

Wenn eine Tabellenreferenz eine Tabelle benennt, die Elternteil einer Tabellenvererbungshierarchie ist, erzeugt die Referenz Zeilen nicht nur aus dieser Tabelle, sondern auch aus allen abgeleiteten Tabellen, sofern dem Tabellennamen nicht das Schlüsselwort `ONLY` vorangestellt wird. Es werden jedoch nur die Spalten ausgegeben, die in der benannten Tabelle vorkommen; Spalten, die in Untertabellen hinzugefügt wurden, werden ignoriert.

Statt `ONLY` vor dem Tabellennamen zu schreiben, können Sie ein `*` nach dem Tabellennamen schreiben, um ausdrücklich anzugeben, dass abgeleitete Tabellen eingeschlossen werden. Dafür gibt es heute keinen echten Grund mehr, weil das Durchsuchen abgeleiteter Tabellen inzwischen Standard ist. Die Syntax wird aber aus Kompatibilitätsgründen mit älteren Versionen unterstützt.

#### 7.2.1.1. Verbundene Tabellen

Eine verbundene Tabelle ist eine Tabelle, die nach den Regeln eines bestimmten Join-Typs aus zwei anderen realen oder abgeleiteten Tabellen entsteht. Es gibt Inner Joins, Outer Joins und Cross Joins. Die allgemeine Syntax lautet:

```sql
T1 join_type T2 [ join_condition ]
```

Joins aller Typen können verkettet oder verschachtelt werden: `T1` oder `T2` oder beide können selbst verbundene Tabellen sein. Klammern können verwendet werden, um die Join-Reihenfolge zu steuern. Ohne Klammern werden `JOIN`-Klauseln von links nach rechts verschachtelt.

**Cross Join**

```sql
T1 CROSS JOIN T2
```

Für jede mögliche Kombination von Zeilen aus `T1` und `T2`, also für das kartesische Produkt, enthält die verbundene Tabelle eine Zeile, die aus allen Spalten von `T1` gefolgt von allen Spalten von `T2` besteht. Wenn die Tabellen `N` beziehungsweise `M` Zeilen haben, enthält die verbundene Tabelle `N * M` Zeilen.

`FROM T1 CROSS JOIN T2` ist äquivalent zu `FROM T1 INNER JOIN T2 ON TRUE`. Es ist außerdem äquivalent zu `FROM T1, T2`.

> **Hinweis:** Die letzte Äquivalenz gilt nicht exakt, wenn mehr als zwei Tabellen vorkommen, weil `JOIN` stärker bindet als das Komma. Zum Beispiel ist `FROM T1 CROSS JOIN T2 INNER JOIN T3 ON condition` nicht dasselbe wie `FROM T1, T2 INNER JOIN T3 ON condition`, weil die Bedingung im ersten Fall auf `T1` verweisen kann, im zweiten aber nicht.

**Qualifizierte Joins**

```sql
T1 { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN T2
 ON boolean_expression
T1 { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN T2 USING (join_column_list)
T1 NATURAL { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN T2
```

Die Wörter `INNER` und `OUTER` sind in allen Formen optional. `INNER` ist die Voreinstellung; `LEFT`, `RIGHT` und `FULL` implizieren einen Outer Join.

Die Join-Bedingung wird in der `ON`- oder `USING`-Klausel angegeben oder implizit durch das Wort `NATURAL` gebildet. Sie bestimmt, welche Zeilen der beiden Quelltabellen als passend gelten.

Mögliche qualifizierte Join-Typen sind:

- `INNER JOIN`: Für jede Zeile `R1` von `T1` enthält die verbundene Tabelle eine Zeile für jede Zeile in `T2`, die mit `R1` die Join-Bedingung erfüllt.
- `LEFT OUTER JOIN`: Zuerst wird ein Inner Join durchgeführt. Dann wird für jede Zeile in `T1`, die mit keiner Zeile in `T2` die Join-Bedingung erfüllt, eine verbundene Zeile mit Nullwerten in den Spalten von `T2` hinzugefügt.
- `RIGHT OUTER JOIN`: Entspricht dem umgekehrten Left Join; die Ergebnistabelle enthält immer mindestens eine Zeile für jede Zeile in `T2`.
- `FULL OUTER JOIN`: Zuerst wird ein Inner Join durchgeführt. Zusätzlich werden nicht passende Zeilen beider Seiten mit Nullwerten auf der jeweils anderen Seite ergänzt.

Die `ON`-Klausel ist die allgemeinste Form einer Join-Bedingung. Sie nimmt einen booleschen Wertausdruck derselben Art wie eine `WHERE`-Klausel. Ein Zeilenpaar aus `T1` und `T2` passt, wenn der `ON`-Ausdruck `true` ergibt.

Die `USING`-Klausel ist eine Kurzform für den Fall, dass beide Seiten des Joins dieselben Namen für die Join-Spalten verwenden. `USING (a, b)` erzeugt zum Beispiel die Bedingung `ON T1.a = T2.a AND T1.b = T2.b`. Außerdem unterdrückt `JOIN USING` redundante Ausgabespalten: Für jedes angegebene Spaltenpaar erscheint nur eine Ausgabespalte.

`NATURAL` ist eine Kurzform von `USING`: Es bildet eine `USING`-Liste aus allen Spaltennamen, die in beiden Eingabetabellen vorkommen. Wenn es keine gemeinsamen Spaltennamen gibt, verhält sich `NATURAL JOIN` wie `CROSS JOIN`.

> **Hinweis:** `USING` ist gegenüber Spaltenänderungen relativ sicher, weil nur die ausdrücklich genannten Spalten kombiniert werden. `NATURAL` ist deutlich riskanter, weil jede Schemaänderung, die auf beiden Seiten einen neuen passenden Spaltennamen erzeugt, dazu führt, dass auch diese neue Spalte kombiniert wird.

Angenommen, wir haben die Tabellen `t1` und `t2`:

```text
 num | name
-----+------
   1 | a
   2 | b
   3 | c

 num | value
-----+-------
   1 | xxx
   3 | yyy
   5 | zzz
```

Dann liefern verschiedene Joins unterschiedliche Ergebnisse:

```text
=> SELECT * FROM t1 CROSS JOIN t2;
 num | name | num | value
-----+------+-----+-------
   1 | a    |   1 | xxx
   1 | a    |   3 | yyy
   1 | a    |   5 | zzz
   2 | b    |   1 | xxx
   2 | b    |   3 | yyy
   2 | b    |   5 | zzz
   3 | c    |   1 | xxx
   3 | c    |   3 | yyy
   3 | c    |   5 | zzz
(9 rows)

=> SELECT * FROM t1 INNER JOIN t2 ON t1.num = t2.num;
 num | name | num | value
-----+------+-----+-------
   1 | a    |   1 | xxx
   3 | c    |   3 | yyy
(2 rows)

=> SELECT * FROM t1 LEFT JOIN t2 USING (num);
 num | name | value
-----+------+-------
   1 | a    | xxx
   2 | b    |
   3 | c    | yyy
(3 rows)

=> SELECT * FROM t1 FULL JOIN t2 ON t1.num = t2.num;
 num | name | num | value
-----+------+-----+-------
   1 | a    |   1 | xxx
   2 | b    |     |
   3 | c    |   3 | yyy
     |      |   5 | zzz
(4 rows)
```

Eine mit `ON` angegebene Join-Bedingung kann auch Bedingungen enthalten, die nicht direkt zum Join gehören. Das kann nützlich sein, muss aber sorgfältig bedacht werden:

```text
=> SELECT * FROM t1 LEFT JOIN t2 ON t1.num = t2.num AND t2.value = 'xxx';
 num | name | num | value
-----+------+-----+-------
   1 | a    |   1 | xxx
   2 | b    |     |
   3 | c    |     |
(3 rows)

=> SELECT * FROM t1 LEFT JOIN t2 ON t1.num = t2.num WHERE t2.value = 'xxx';
 num | name | num | value
-----+------+-----+-------
   1 | a    |   1 | xxx
(1 row)
```

Der Unterschied entsteht, weil eine Einschränkung in der `ON`-Klausel vor dem Join verarbeitet wird, eine Einschränkung in der `WHERE`-Klausel dagegen nach dem Join. Bei Inner Joins spielt das keine Rolle, bei Outer Joins sehr wohl.

#### 7.2.1.2. Tabellen- und Spaltenaliase

Tabellen und komplexen Tabellenreferenzen kann ein temporärer Name gegeben werden, der im Rest der Abfrage für Verweise auf die abgeleitete Tabelle verwendet wird. Das heißt Tabellenalias.

```sql
FROM table_reference AS alias
FROM table_reference alias
```

Das Schlüsselwort `AS` ist optional. Ein typischer Einsatz von Tabellenaliasen besteht darin, langen Tabellennamen kurze Bezeichner zu geben:

```sql
SELECT * FROM some_very_long_table_name s
JOIN another_fairly_long_name a ON s.id = a.num;
```

Der Alias wird für die aktuelle Abfrage zum neuen Namen der Tabellenreferenz. Der ursprüngliche Tabellenname darf an anderer Stelle der Abfrage nicht mehr verwendet werden:

```sql
SELECT * FROM my_table AS m WHERE my_table.a > 5; -- falsch
```

Tabellenaliase sind vor allem eine Schreibvereinfachung, sind aber nötig, wenn eine Tabelle mit sich selbst verbunden wird:

```sql
SELECT * FROM people AS mother
JOIN people AS child ON mother.id = child.mother_id;
```

Eine andere Form der Aliasvergabe gibt auch den Spalten der Tabelle temporäre Namen:

```sql
FROM table_reference [AS] alias (column1 [, column2 [, ...]])
```

Wenn weniger Spaltenaliase angegeben werden, als die Tabelle tatsächlich Spalten besitzt, bleiben die übrigen Spalten unverändert. Diese Syntax ist besonders bei Self Joins und Unterabfragen nützlich. Wenn ein Alias auf die Ausgabe einer `JOIN`-Klausel angewendet wird, verbirgt dieser Alias die ursprünglichen Namen innerhalb des Joins.

#### 7.2.1.3. Unterabfragen

Unterabfragen, die eine abgeleitete Tabelle angeben, müssen in Klammern stehen. Sie können einen Tabellenalias und optional Spaltenaliase erhalten:

```sql
FROM (SELECT * FROM table1) AS alias_name
```

Dieses Beispiel entspricht `FROM table1 AS alias_name`. Interessantere Fälle, die sich nicht auf einen einfachen Join reduzieren lassen, entstehen, wenn die Unterabfrage Gruppierung oder Aggregation enthält.

Eine Unterabfrage kann auch eine `VALUES`-Liste sein:

```sql
FROM (VALUES ('anne', 'smith'), ('bob', 'jones'), ('joe', 'blow'))
     AS names(first, last)
```

Ein Tabellenalias ist wiederum optional. Spaltenaliase für die `VALUES`-Liste sind ebenfalls optional, aber gute Praxis. Weitere Informationen finden Sie in [Abschnitt 7.7](07_Abfragen.md#77-valueslisten).

Nach dem SQL-Standard muss für eine Unterabfrage ein Tabellenalias angegeben werden. PostgreSQL erlaubt, `AS` und den Alias wegzulassen; für portierbaren SQL-Code ist ein Alias aber gute Praxis.

#### 7.2.1.4. Tabellenfunktionen

Tabellenfunktionen sind Funktionen, die eine Menge von Zeilen erzeugen, entweder aus Basistypen oder aus zusammengesetzten Datentypen. Sie werden in der `FROM`-Klausel wie Tabellen, Views oder Unterabfragen verwendet. Von Tabellenfunktionen zurückgegebene Spalten können in `SELECT`-, `JOIN`- oder `WHERE`-Klauseln genauso verwendet werden wie Spalten einer Tabelle, View oder Unterabfrage.

Tabellenfunktionen können auch mit der Syntax `ROWS FROM` kombiniert werden. Die Ergebnisse werden dabei in parallelen Spalten zurückgegeben; die Zahl der Ergebniszeilen entspricht dem größten Funktionsergebnis, kleinere Ergebnisse werden mit Nullwerten aufgefüllt.

```sql
function_call [WITH ORDINALITY] [[AS] table_alias [(column_alias [, ...])]]
ROWS FROM(function_call [, ...]) [WITH ORDINALITY]
 [[AS] table_alias [(column_alias [, ...])]]
```

Wenn `WITH ORDINALITY` angegeben ist, wird den Ergebnisspalten der Funktion eine zusätzliche Spalte vom Typ `bigint` hinzugefügt. Diese Spalte nummeriert die Zeilen des Funktionsergebnisses ab 1. Standardmäßig heißt sie `ordinality`; über eine `AS`-Klausel kann ein anderer Name vergeben werden.

Die spezielle Tabellenfunktion `UNNEST` kann mit beliebig vielen Array-Parametern aufgerufen werden. Sie gibt eine entsprechende Anzahl von Spalten zurück, als wäre `UNNEST` ([Abschnitt 9.19](09_Funktionen_und_Operatoren.md#919-arrayfunktionen-und-operatoren)) für jeden Parameter einzeln aufgerufen und mit `ROWS FROM` kombiniert worden.

```sql
UNNEST(array_expression [, ...]) [WITH ORDINALITY]
 [[AS] table_alias [(column_alias [, ...])]]
```

Wenn kein Tabellenalias angegeben ist, wird der Funktionsname als Tabellenname verwendet; bei einem `ROWS FROM()`-Konstrukt wird der Name der ersten Funktion verwendet.

Einige Beispiele:

```sql
CREATE TABLE foo (fooid int, foosubid int, fooname text);

CREATE FUNCTION getfoo(int) RETURNS SETOF foo AS $$
    SELECT * FROM foo WHERE fooid = $1;
$$ LANGUAGE SQL;

SELECT * FROM getfoo(1) AS t1;

SELECT * FROM foo
    WHERE foosubid IN (
        SELECT foosubid
        FROM getfoo(foo.fooid) z
        WHERE z.fooid = foo.fooid
    );

CREATE VIEW vw_getfoo AS SELECT * FROM getfoo(1);

SELECT * FROM vw_getfoo;
```

In manchen Fällen ist es nützlich, Tabellenfunktionen zu definieren, die je nach Aufruf unterschiedliche Spaltenmengen zurückgeben können. Dazu kann die Tabellenfunktion als Rückgabe des Pseudotyps `record` ohne `OUT`-Parameter deklariert werden. Wird eine solche Funktion in einer Abfrage verwendet, muss die erwartete Zeilenstruktur in der Abfrage selbst angegeben werden:

```sql
function_call [AS] alias (column_definition [, ...])
function_call AS [alias] (column_definition [, ...])
ROWS FROM(... function_call AS (column_definition [, ...]) [, ...])
```

Beispiel mit `dblink`:

```sql
SELECT *
    FROM dblink('dbname=mydb', 'SELECT proname, prosrc FROM pg_proc')
      AS t1(proname name, prosrc text)
    WHERE proname LIKE 'bytea%';
```

Die Funktion `dblink` führt eine entfernte Abfrage aus. Da sie für beliebige Abfragen verwendet werden kann, ist sie als Rückgabe von `record` deklariert. Die tatsächliche Spaltenmenge muss in der aufrufenden Abfrage angegeben werden.

Beispiel mit `ROWS FROM`:

```sql
SELECT *
FROM ROWS FROM
    (
        json_to_recordset('[{"a":40,"b":"foo"}, {"a":"100","b":"bar"}]')
            AS (a INTEGER, b TEXT),
        generate_series(1, 3)
    ) AS x (p, q, s)
ORDER BY p;
```

```text
  p | q   | s
----+-----+---
 40 | foo | 1
100 | bar | 2
    |     | 3
```

#### 7.2.1.5. `LATERAL`-Unterabfragen

Unterabfragen in `FROM` können mit dem Schlüsselwort `LATERAL` versehen werden. Dadurch dürfen sie auf Spalten verweisen, die von vorhergehenden `FROM`-Elementen bereitgestellt werden. Ohne `LATERAL` wird jede Unterabfrage unabhängig ausgewertet und kann keine anderen `FROM`-Elemente querreferenzieren.

Tabellenfunktionen in `FROM` können ebenfalls mit `LATERAL` versehen werden; bei Funktionen ist das Schlüsselwort jedoch optional, da Funktionsargumente ohnehin auf Spalten vorhergehender `FROM`-Elemente verweisen dürfen.

Ein einfaches Beispiel:

```sql
SELECT * FROM foo, LATERAL
    (SELECT * FROM bar WHERE bar.id = foo.bar_id) ss;
```

Das ist hier nicht besonders nützlich, weil es dasselbe Ergebnis wie die konventionelle Schreibweise liefert:

```sql
SELECT * FROM foo, bar WHERE bar.id = foo.bar_id;
```

`LATERAL` ist vor allem nützlich, wenn eine querreferenzierte Spalte zur Berechnung der zu verbindenden Zeilen nötig ist. Ein häufiger Anwendungsfall ist die Übergabe eines Arguments an eine mengenliefernde Funktion:

```sql
SELECT p1.id, p2.id, v1, v2
FROM polygons p1, polygons p2,
     LATERAL vertices(p1.poly) v1,
     LATERAL vertices(p2.poly) v2
WHERE (v1 <-> v2) < 10 AND p1.id != p2.id;
```

Es ist oft besonders praktisch, eine `LATERAL`-Unterabfrage per `LEFT JOIN` zu verbinden, damit Quellzeilen auch dann im Ergebnis erscheinen, wenn die `LATERAL`-Unterabfrage für sie keine Zeilen liefert:

```sql
SELECT m.name
FROM manufacturers m LEFT JOIN LATERAL get_product_names(m.id) pname ON true
WHERE pname IS NULL;
```

### 7.2.2. Die `WHERE`-Klausel

Die Syntax der `WHERE`-Klausel lautet:

```sql
WHERE search_condition
```

`search_condition` ist ein Wertausdruck (siehe [Abschnitt 4.2](04_SQL_Syntax.md#42-wertausdrücke)), der einen Wert vom Typ `boolean` zurückgibt.

Nachdem die `FROM`-Klausel verarbeitet wurde, wird jede Zeile der abgeleiteten virtuellen Tabelle gegen die Suchbedingung geprüft. Wenn das Ergebnis `true` ist, bleibt die Zeile in der Ausgabetabelle; wenn es `false` oder `null` ist, wird sie verworfen.

> **Hinweis:** Die Join-Bedingung eines Inner Joins kann entweder in der `WHERE`-Klausel oder in der `JOIN`-Klausel geschrieben werden. Bei Outer Joins gibt es diese Wahl nicht: Sie müssen in der `FROM`-Klausel formuliert werden, weil `ON` oder `USING` dort nicht dasselbe ist wie eine nachträgliche `WHERE`-Bedingung.

Beispiele für `WHERE`-Klauseln:

```sql
SELECT ... FROM fdt WHERE c1 > 5;
SELECT ... FROM fdt WHERE c1 IN (1, 2, 3);
SELECT ... FROM fdt WHERE c1 IN (SELECT c1 FROM t2);
SELECT ... FROM fdt WHERE c1 IN (SELECT c3 FROM t2 WHERE c2 = fdt.c1 + 10);
SELECT ... FROM fdt WHERE c1 BETWEEN (SELECT c3 FROM t2 WHERE c2 = fdt.c1 + 10) AND 100;
SELECT ... FROM fdt WHERE EXISTS (SELECT c1 FROM t2 WHERE c2 > fdt.c1);
```

`fdt` ist die in der `FROM`-Klausel abgeleitete Tabelle. Zeilen, die die Suchbedingung der `WHERE`-Klausel nicht erfüllen, werden daraus entfernt. Die Beispiele zeigen auch skalare Unterabfragen als Wertausdrücke und wie der Namensraum einer äußeren Abfrage in innere Abfragen hineinreicht.

### 7.2.3. Die Klauseln `GROUP BY` und `HAVING`

Nach dem `WHERE`-Filter kann die abgeleitete Eingabetabelle mit `GROUP BY` gruppiert und mit `HAVING` nach Gruppen gefiltert werden.

```sql
SELECT select_list
    FROM ...
    [WHERE ...]
    GROUP BY grouping_column_reference [, grouping_column_reference]...
```

Die `GROUP BY`-Klausel fasst diejenigen Zeilen zusammen, die in allen aufgelisteten Spalten dieselben Werte haben. Die Reihenfolge der Spalten spielt keine Rolle. Jede Menge von Zeilen mit gemeinsamen Werten wird zu einer Gruppenzeile zusammengefasst. Das dient dazu, Redundanz in der Ausgabe zu beseitigen und/oder Aggregate für diese Gruppen zu berechnen.

```text
=> SELECT * FROM test1;
 x | y
---+---
 a | 3
 c | 2
 b | 5
 a | 1
(4 rows)

=> SELECT x FROM test1 GROUP BY x;
 x
---
 a
 b
 c
(3 rows)

=> SELECT x, sum(y) FROM test1 GROUP BY x;
 x | sum
---+-----
 a |   4
 b |   5
 c |   2
(3 rows)
```

`sum` ist hier eine Aggregatfunktion, die einen einzelnen Wert über die gesamte Gruppe berechnet. Weitere Informationen über verfügbare Aggregatfunktionen finden Sie in [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen).

> **Tipp:** Gruppieren ohne Aggregatausdrücke berechnet effektiv die Menge unterschiedlicher Werte in einer Spalte. Das kann auch mit `DISTINCT` erreicht werden (siehe [Abschnitt 7.3.3](#733-distinct)).

Ein realistischeres Beispiel berechnet den Gesamtumsatz je Produkt:

```sql
SELECT product_id, p.name, (sum(s.units) * p.price) AS sales
    FROM products p LEFT JOIN sales s USING (product_id)
    GROUP BY product_id, p.name, p.price;
```

Wenn die Tabelle `products` so definiert ist, dass `product_id` der Primärschlüssel ist, genügt es, im obigen Beispiel nach `product_id` zu gruppieren, da `name` und `price` funktional von der Produkt-ID abhängen.

Wenn nur bestimmte Gruppen interessant sind, kann die `HAVING`-Klausel ähnlich wie `WHERE` verwendet werden, um Gruppen aus dem Ergebnis zu entfernen:

```sql
SELECT select_list FROM ... [WHERE ...] GROUP BY ... HAVING boolean_expression;
```

Beispiele:

```text
=> SELECT x, sum(y) FROM test1 GROUP BY x HAVING sum(y) > 3;
 x | sum
---+-----
 a |   4
 b |   5
(2 rows)

=> SELECT x, sum(y) FROM test1 GROUP BY x HAVING x < 'c';
 x | sum
---+-----
 a |   4
 b |   5
(2 rows)
```

Ein weiteres Beispiel:

```sql
SELECT product_id, p.name, (sum(s.units) * (p.price - p.cost)) AS profit
    FROM products p LEFT JOIN sales s USING (product_id)
    WHERE s.date > CURRENT_DATE - INTERVAL '4 weeks'
    GROUP BY product_id, p.name, p.price, p.cost
    HAVING sum(p.price * s.units) > 5000;
```

Wenn eine Abfrage Aggregatfunktionsaufrufe enthält, aber keine `GROUP BY`-Klausel, findet trotzdem Gruppierung statt: Das Ergebnis ist eine einzelne Gruppenzeile oder keine Zeile, falls diese durch `HAVING` entfernt wird.

### 7.2.4. `GROUPING SETS`, `CUBE` und `ROLLUP`

Komplexere Gruppierungsoperationen sind mit Gruppierungsmengen möglich. Die von `FROM` und `WHERE` ausgewählten Daten werden separat nach jeder angegebenen Gruppierungsmenge gruppiert, Aggregate werden wie bei einfachen `GROUP BY`-Klauseln berechnet, und die Ergebnisse werden zurückgegeben.

```text
=> SELECT * FROM items_sold;
 brand | size | sales
-------+------+-------
 Foo   | L    | 10
 Foo   | M    | 20
 Bar   | M    | 15
 Bar   | L    | 5
(4 rows)

=> SELECT brand, size, sum(sales)
   FROM items_sold
   GROUP BY GROUPING SETS ((brand), (size), ());
 brand | size | sum
-------+------+-----
 Foo   |      | 30
 Bar   |      | 20
       | L    | 15
       | M    | 35
       |      | 50
(5 rows)
```

Jede Teilliste von `GROUPING SETS` kann null oder mehr Spalten oder Ausdrücke angeben und wird so interpretiert, als stünde sie direkt in der `GROUP BY`-Klausel. Eine leere Gruppierungsmenge bedeutet, dass alle Zeilen zu einer einzigen Gruppe aggregiert werden.

Für häufige Gruppierungsmengen gibt es Kurzformen:

```sql
ROLLUP (e1, e2, e3, ...)
```

entspricht den angegebenen Ausdrücken und allen Präfixen der Liste, einschließlich der leeren Liste. Das wird oft für hierarchische Analysen verwendet, etwa Gesamtgehälter nach Abteilung, Bereich und Unternehmen.

```sql
CUBE (e1, e2, ...)
```

steht für die angegebene Liste und alle möglichen Teilmengen. Zum Beispiel ist

```sql
CUBE (a, b, c)
```

äquivalent zu:

```sql
GROUPING SETS (
    (a, b, c),
    (a, b),
    (a, c),
    (a),
    (b, c),
    (b),
    (c),
    ()
)
```

Die Elemente von `CUBE` oder `ROLLUP` können einzelne Ausdrücke oder geklammerte Teillisten sein. Teillisten werden beim Erzeugen der Gruppierungsmengen als Einheiten behandelt.

Wenn in einer einzelnen `GROUP BY`-Klausel mehrere Gruppierungselemente angegeben sind, ist die endgültige Liste der Gruppierungsmengen das kartesische Produkt dieser Elemente. Dabei können Duplikate entstehen. Falls diese unerwünscht sind, können sie mit `GROUP BY DISTINCT` entfernt werden. Das ist nicht dasselbe wie `SELECT DISTINCT`, weil die Ausgabezeilen weiterhin Duplikate enthalten können, wenn nicht gruppierte Spalten `NULL` enthalten.

> **Hinweis:** Das Konstrukt `(a, b)` wird in Ausdrücken normalerweise als Row-Konstruktor erkannt. Innerhalb der `GROUP BY`-Klausel gilt das auf den obersten Ebenen nicht; dort wird `(a, b)` als Liste von Ausdrücken interpretiert. Wenn ein Row-Konstruktor benötigt wird, verwenden Sie `ROW(a, b)`.

### 7.2.5. Verarbeitung von Window-Funktionen

Wenn die Abfrage Window-Funktionen enthält (siehe [Abschnitt 3.5](03_Fortgeschrittene_Funktionen.md#35-windowfunktionen), [Abschnitt 9.22](09_Funktionen_und_Operatoren.md#922-fensterfunktionen) und [Abschnitt 4.2.8](04_SQL_Syntax.md#428-aufrufe-von-windowfunktionen)), werden diese Funktionen nach Gruppierung, Aggregation und `HAVING`-Filterung ausgewertet. Wenn die Abfrage also Aggregate, `GROUP BY` oder `HAVING` verwendet, sehen Window-Funktionen die Gruppenzeilen statt der ursprünglichen Tabellenzeilen aus `FROM` und `WHERE`.

Werden mehrere Window-Funktionen verwendet, ist garantiert, dass alle Window-Funktionen mit äquivalenten `PARTITION BY`- und `ORDER BY`-Klauseln dieselbe Reihenfolge der Eingabezeilen sehen. Für Funktionen mit unterschiedlichen `PARTITION BY`- oder `ORDER BY`-Spezifikationen gibt es keine solche Garantie.

Derzeit benötigen Window-Funktionen immer vorsortierte Daten, sodass die Abfrageausgabe gemäß der `PARTITION BY`-/`ORDER BY`-Klausel der einen oder anderen Window-Funktion sortiert sein kann. Darauf sollte man sich aber nicht verlassen. Verwenden Sie eine ausdrückliche `ORDER BY`-Klausel auf oberster Ebene, wenn die Ergebnisse sicher auf bestimmte Weise sortiert sein sollen.

## 7.3. Auswahllisten

Wie im vorigen Abschnitt gezeigt, konstruiert der Tabellenausdruck im `SELECT`-Befehl eine virtuelle Zwischentabelle, indem er Tabellen und Views kombiniert, Zeilen entfernt, gruppiert usw. Diese Tabelle wird schließlich an die Auswahlliste weitergereicht. Die Auswahlliste bestimmt, welche Spalten der Zwischentabelle tatsächlich ausgegeben werden.

### 7.3.1. Elemente der Auswahlliste

Die einfachste Auswahlliste ist `*`; sie gibt alle Spalten aus, die der Tabellenausdruck erzeugt. Andernfalls ist eine Auswahlliste eine kommaseparierte Liste von Wertausdrücken (siehe [Abschnitt 4.2](04_SQL_Syntax.md#42-wertausdrücke)). Zum Beispiel:

```sql
SELECT a, b, c FROM ...
SELECT tbl1.a, tbl2.a, tbl1.b FROM ...
SELECT tbl1.*, tbl2.a FROM ...
```

Die Spaltennamen sind entweder die tatsächlichen Spaltennamen von Tabellen in der `FROM`-Klausel oder die ihnen zugewiesenen Aliase. Wenn mehr als eine Tabelle eine Spalte mit demselben Namen besitzt, muss auch der Tabellenname angegeben werden.

Wird ein beliebiger Wertausdruck in der Auswahlliste verwendet, fügt er konzeptionell eine neue virtuelle Spalte zur zurückgegebenen Tabelle hinzu. Der Ausdruck wird für jede Ergebniszeile einmal ausgewertet.

### 7.3.2. Spaltenbezeichnungen

Einträge in der Auswahlliste können Namen für spätere Verarbeitung erhalten, etwa für `ORDER BY` oder für die Anzeige durch die Client-Anwendung:

```sql
SELECT a AS value, b + c AS sum FROM ...
```

Wenn kein Ausgabespaltenname mit `AS` angegeben wird, vergibt das System einen Standardnamen. Bei einfachen Spaltenreferenzen ist das der Name der referenzierten Spalte, bei Funktionsaufrufen der Funktionsname und bei komplexen Ausdrücken ein generischer Name.

Das Schlüsselwort `AS` ist normalerweise optional. Wenn der gewünschte Spaltenname jedoch einem PostgreSQL-Schlüsselwort entspricht, muss `AS` geschrieben oder der Name doppelt gequotet werden:

```sql
SELECT a AS from, b + c AS sum FROM ...
SELECT a "from", b + c AS sum FROM ...
```

Zur größtmöglichen Sicherheit gegenüber zukünftigen Schlüsselwörtern empfiehlt es sich, immer entweder `AS` zu schreiben oder den Ausgabespaltennamen doppelt zu quoten.

> **Hinweis:** Die Benennung von Ausgabespalten unterscheidet sich von der Benennung in der `FROM`-Klausel. Eine Spalte kann zweimal umbenannt werden; der in der Auswahlliste vergebene Name ist der weitergereichte.

### 7.3.3. `DISTINCT`

Nachdem die Auswahlliste verarbeitet wurde, kann die Ergebnistabelle optional von doppelten Zeilen bereinigt werden. Das Schlüsselwort `DISTINCT` steht direkt nach `SELECT`:

```sql
SELECT DISTINCT select_list ...
```

Statt `DISTINCT` kann `ALL` verwendet werden, um das Standardverhalten beizubehalten, also alle Zeilen zu behalten. Zwei Zeilen gelten als verschieden, wenn sie sich in mindestens einem Spaltenwert unterscheiden; Nullwerte gelten in diesem Vergleich als gleich.

Alternativ kann ein Ausdruck bestimmen, welche Zeilen als gleich gelten:

```sql
SELECT DISTINCT ON (expression [, expression ...]) select_list ...
```

Für jede Menge von Zeilen, für die alle Ausdrücke gleich sind, wird nur die erste Zeile behalten. Diese „erste Zeile“ ist unvorhersehbar, sofern die Abfrage nicht nach genügend Spalten sortiert ist, um eine eindeutige Reihenfolge zu garantieren.

`DISTINCT ON` gehört nicht zum SQL-Standard und gilt wegen der möglichen Unbestimmtheit manchmal als schlechter Stil. Mit sorgfältiger Verwendung von `GROUP BY` und Unterabfragen lässt es sich vermeiden, ist aber oft die bequemste Alternative.

## 7.4. Abfragen kombinieren (`UNION`, `INTERSECT`, `EXCEPT`)

Die Ergebnisse zweier Abfragen können mit Mengenoperationen kombiniert werden: Vereinigung, Schnittmenge und Differenz.

```sql
query1 UNION [ALL] query2
query1 INTERSECT [ALL] query2
query1 EXCEPT [ALL] query2
```

`UNION` hängt das Ergebnis von `query2` effektiv an das Ergebnis von `query1` an und entfernt doppelte Zeilen wie `DISTINCT`, sofern nicht `UNION ALL` verwendet wird.

`INTERSECT` gibt alle Zeilen zurück, die sowohl im Ergebnis von `query1` als auch im Ergebnis von `query2` vorkommen. Doppelte Zeilen werden entfernt, sofern nicht `INTERSECT ALL` verwendet wird.

`EXCEPT` gibt alle Zeilen zurück, die im Ergebnis von `query1`, aber nicht im Ergebnis von `query2` vorkommen. Auch hier werden Duplikate entfernt, sofern nicht `EXCEPT ALL` verwendet wird.

Für diese Operationen müssen die beiden Abfragen „union-kompatibel“ sein: Sie müssen dieselbe Anzahl von Spalten zurückgeben, und die entsprechenden Spalten müssen kompatible Datentypen haben (siehe [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte)).

Mengenoperationen können kombiniert werden:

```sql
query1 UNION query2 EXCEPT query3
```

Das entspricht:

```sql
(query1 UNION query2) EXCEPT query3
```

Klammern können die Auswertungsreihenfolge steuern. Ohne Klammern binden `UNION` und `EXCEPT` von links nach rechts, während `INTERSECT` stärker bindet.

## 7.5. Zeilen sortieren (`ORDER BY`)

Nachdem eine Abfrage eine Ausgabetabelle erzeugt hat, kann sie optional sortiert werden. Ohne ausdrückliche Sortierung werden die Zeilen in einer nicht spezifizierten Reihenfolge zurückgegeben. Eine bestimmte Reihenfolge ist nur garantiert, wenn der Sortierschritt ausdrücklich gewählt wird.

```sql
SELECT select_list
    FROM table_expression
    ORDER BY sort_expression1 [ASC | DESC] [NULLS { FIRST | LAST }]
          [, sort_expression2 [ASC | DESC] [NULLS { FIRST | LAST }] ...]
```

Sortierausdrücke können beliebige Ausdrücke sein, die in der Auswahlliste gültig wären:

```sql
SELECT a, b FROM table1 ORDER BY a + b, c;
```

Wenn mehrere Ausdrücke angegeben sind, werden spätere Werte nur verwendet, um Zeilen zu sortieren, die nach früheren Werten gleich sind. `ASC` sortiert aufsteigend und ist die Voreinstellung; `DESC` sortiert absteigend. `NULLS FIRST` und `NULLS LAST` bestimmen, ob Nullwerte vor oder nach Nicht-Nullwerten erscheinen. Standardmäßig werden Nullwerte so sortiert, als wären sie größer als jeder Nicht-Nullwert; daher ist `NULLS FIRST` die Voreinstellung bei `DESC`, sonst `NULLS LAST`.

Ein Sortierausdruck kann auch die Spaltenbezeichnung oder Nummer einer Ausgabespalte sein:

```sql
SELECT a + b AS sum, c FROM table1 ORDER BY sum;
SELECT a, max(b) FROM table1 GROUP BY a ORDER BY 1;
```

Ein Ausgabespaltenname muss dabei allein stehen; er kann nicht in einem Ausdruck verwendet werden:

```sql
SELECT a + b AS sum, c FROM table1 ORDER BY sum + c; -- falsch
```

`ORDER BY` kann auch auf das Ergebnis von `UNION`, `INTERSECT` oder `EXCEPT` angewendet werden. Dann darf jedoch nur nach Ausgabespaltennamen oder -nummern sortiert werden, nicht nach Ausdrücken.

## 7.6. `LIMIT` und `OFFSET`

`LIMIT` und `OFFSET` erlauben, nur einen Teil der vom Rest der Abfrage erzeugten Zeilen abzurufen:

```sql
SELECT select_list
    FROM table_expression
    [ORDER BY ...]
    [LIMIT { count | ALL }]
    [OFFSET start]
```

Wenn ein `LIMIT` angegeben ist, werden höchstens so viele Zeilen zurückgegeben. `LIMIT ALL` entspricht dem Weglassen der `LIMIT`-Klausel; dasselbe gilt für `LIMIT` mit einem `NULL`-Argument.

`OFFSET` überspringt die angegebene Anzahl Zeilen, bevor Zeilen zurückgegeben werden. `OFFSET 0` entspricht dem Weglassen der Klausel; dasselbe gilt für `OFFSET` mit einem `NULL`-Argument.

Bei Verwendung von `LIMIT` ist es wichtig, eine `ORDER BY`-Klausel zu verwenden, die die Ergebniszeilen in eine eindeutige Reihenfolge bringt. Andernfalls erhalten Sie eine unvorhersehbare Teilmenge der Abfragezeilen. Der Optimierer berücksichtigt `LIMIT` bei der Planerzeugung; unterschiedliche `LIMIT`-/`OFFSET`-Werte können daher zu unterschiedlichen Plänen und Zeilenreihenfolgen führen.

Die von `OFFSET` übersprungenen Zeilen müssen im Server trotzdem berechnet werden; ein großes `OFFSET` kann daher ineffizient sein.

## 7.7. `VALUES`-Listen

`VALUES` bietet eine Möglichkeit, eine „konstante Tabelle“ zu erzeugen, die in einer Abfrage verwendet werden kann, ohne tatsächlich eine Tabelle auf dem Datenträger zu erstellen und zu befüllen.

```sql
VALUES (expression [, ...]) [, ...]
```

Jede geklammerte Ausdrucksliste erzeugt eine Zeile. Alle Listen müssen dieselbe Anzahl von Elementen haben, und entsprechende Einträge müssen kompatible Datentypen besitzen. Der tatsächliche Datentyp jeder Ergebnisspalte wird nach denselben Regeln wie bei `UNION` bestimmt (siehe [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte)).

```sql
VALUES (1, 'one'), (2, 'two'), (3, 'three');
```

Das liefert eine Tabelle mit zwei Spalten und drei Zeilen und ist effektiv äquivalent zu:

```sql
SELECT 1 AS column1, 'one' AS column2
UNION ALL
SELECT 2, 'two'
UNION ALL
SELECT 3, 'three';
```

PostgreSQL nennt die Spalten einer `VALUES`-Tabelle standardmäßig `column1`, `column2` usw. Da diese Namen nicht vom SQL-Standard festgelegt sind, ist es normalerweise besser, sie mit einer Aliasliste zu überschreiben:

```text
=> SELECT * FROM (VALUES (1, 'one'), (2, 'two'), (3, 'three')) AS t (num, letter);
 num | letter
-----+--------
   1 | one
   2 | two
   3 | three
(3 rows)
```

Syntaktisch wird `VALUES` mit Ausdruckslisten ähnlich wie `SELECT select_list FROM table_expression` behandelt und kann überall erscheinen, wo `SELECT` erscheinen kann. Am häufigsten wird es als Datenquelle in einem `INSERT`-Befehl verwendet, danach als Unterabfrage.

## 7.8. `WITH`-Abfragen (Common Table Expressions)

`WITH` bietet eine Möglichkeit, Hilfsanweisungen für eine größere Abfrage zu schreiben. Diese Anweisungen, oft Common Table Expressions oder CTEs genannt, können als temporäre Tabellen verstanden werden, die nur für eine Abfrage existieren. Jede Hilfsanweisung in einer `WITH`-Klausel kann `SELECT`, `INSERT`, `UPDATE`, `DELETE` oder `MERGE` sein. Die `WITH`-Klausel selbst wird an eine Hauptanweisung angehängt, die ebenfalls `SELECT`, `INSERT`, `UPDATE`, `DELETE` oder `MERGE` sein kann.

### 7.8.1. `SELECT` in `WITH`

Der grundlegende Nutzen von `SELECT` in `WITH` besteht darin, komplizierte Abfragen in einfachere Teile zu zerlegen:

```sql
WITH regional_sales AS (
     SELECT region, SUM(amount) AS total_sales
     FROM orders
     GROUP BY region
), top_regions AS (
     SELECT region
     FROM regional_sales
     WHERE total_sales > (SELECT SUM(total_sales)/10 FROM regional_sales)
)
SELECT region,
        product,
        SUM(quantity) AS product_units,
        SUM(amount) AS product_sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;
```

Die `WITH`-Klausel definiert zwei Hilfsanweisungen, `regional_sales` und `top_regions`. Die Ausgabe von `regional_sales` wird in `top_regions` verwendet, und die Ausgabe von `top_regions` in der primären `SELECT`-Abfrage.

### 7.8.2. Rekursive Abfragen

Der optionale Modifikator `RECURSIVE` macht `WITH` aus einer reinen Schreibvereinfachung zu einem Feature, das Dinge ermöglicht, die sonst in Standard-SQL nicht möglich wären. Mit `RECURSIVE` kann eine `WITH`-Abfrage auf ihre eigene Ausgabe verweisen. Ein einfaches Beispiel summiert die Ganzzahlen von 1 bis 100:

```sql
WITH RECURSIVE t(n) AS (
    VALUES (1)
  UNION ALL
    SELECT n+1 FROM t WHERE n < 100
)
SELECT sum(n) FROM t;
```

Die allgemeine Form einer rekursiven `WITH`-Abfrage besteht aus einem nichtrekursiven Term, dann `UNION` oder `UNION ALL`, dann einem rekursiven Term. Nur der rekursive Term darf auf die eigene Ausgabe der Abfrage verweisen.

Rekursive Abfragen werden typischerweise für hierarchische oder baumartige Daten verwendet. Ein Beispiel findet alle direkten und indirekten Unterteile eines Produkts:

```sql
WITH RECURSIVE included_parts(sub_part, part, quantity) AS (
     SELECT sub_part, part, quantity FROM parts WHERE part = 'our_product'
   UNION ALL
     SELECT p.sub_part, p.part, p.quantity * pr.quantity
     FROM included_parts pr, parts p
     WHERE p.part = pr.sub_part
)
SELECT sub_part, SUM(quantity) AS total_quantity
FROM included_parts
GROUP BY sub_part;
```

#### 7.8.2.1. Suchreihenfolge

Bei einer Baumsuche mit einer rekursiven Abfrage möchte man die Ergebnisse oft in Tiefensuche- oder Breitensuche-Reihenfolge ausgeben. Dazu kann neben den eigentlichen Daten eine Sortierspalte berechnet und am Ende nach ihr sortiert werden. Das steuert nicht die tatsächliche Besuchsreihenfolge während der Auswertung; diese ist implementierungsabhängig. Es sortiert nur das Ergebnis.

Für Tiefensuche kann der bisher besuchte Pfad als Array gespeichert werden:

```sql
WITH RECURSIVE search_tree(id, link, data, path) AS (
    SELECT t.id, t.link, t.data, ARRAY[t.id]
    FROM tree t
  UNION ALL
    SELECT t.id, t.link, t.data, path || t.id
    FROM tree t, search_tree st
    WHERE t.id = st.link
)
SELECT * FROM search_tree ORDER BY path;
```

Wenn mehrere Felder zur Identifikation einer Zeile nötig sind, kann ein Array von Zeilen verwendet werden:

```sql
WITH RECURSIVE search_tree(id, link, data, path) AS (
    SELECT t.id, t.link, t.data, ARRAY[ROW(t.f1, t.f2)]
    FROM tree t
  UNION ALL
    SELECT t.id, t.link, t.data, path || ROW(t.f1, t.f2)
    FROM tree t, search_tree st
    WHERE t.id = st.link
)
SELECT * FROM search_tree ORDER BY path;
```

> **Tipp:** Lassen Sie die `ROW()`-Syntax weg, wenn nur ein Feld verfolgt werden muss. Dann kann ein einfaches Array statt eines Arrays zusammengesetzter Typen verwendet werden, was effizienter ist.

Für Breitensuche kann eine Spalte verwendet werden, die die Tiefe verfolgt:

```sql
WITH RECURSIVE search_tree(id, link, data, depth) AS (
    SELECT t.id, t.link, t.data, 0
    FROM tree t
  UNION ALL
    SELECT t.id, t.link, t.data, depth + 1
    FROM tree t, search_tree st
    WHERE t.id = st.link
)
SELECT * FROM search_tree ORDER BY depth;
```

PostgreSQL bietet außerdem eingebaute Syntax zum Berechnen einer Tiefen- oder Breitensuchsortierspalte:

```sql
WITH RECURSIVE search_tree(id, link, data) AS (
    SELECT t.id, t.link, t.data
    FROM tree t
  UNION ALL
    SELECT t.id, t.link, t.data
    FROM tree t, search_tree st
    WHERE t.id = st.link
) SEARCH DEPTH FIRST BY id SET ordercol
SELECT * FROM search_tree ORDER BY ordercol;

WITH RECURSIVE search_tree(id, link, data) AS (
    SELECT t.id, t.link, t.data
    FROM tree t
  UNION ALL
    SELECT t.id, t.link, t.data
    FROM tree t, search_tree st
    WHERE t.id = st.link
) SEARCH BREADTH FIRST BY id SET ordercol
SELECT * FROM search_tree ORDER BY ordercol;
```

#### 7.8.2.2. Zyklenerkennung

Bei rekursiven Abfragen muss sichergestellt sein, dass der rekursive Teil irgendwann keine Tupel mehr zurückgibt; andernfalls läuft die Abfrage endlos. Manchmal reicht `UNION` statt `UNION ALL`, weil doppelte Ergebniszeilen verworfen werden. Oft besteht ein Zyklus aber nicht aus vollständig doppelten Ausgabezeilen. Dann muss anhand eines oder mehrerer Felder geprüft werden, ob derselbe Punkt bereits erreicht wurde.

Eine übliche Methode ist, ein Array der bereits besuchten Werte zu berechnen:

```sql
WITH RECURSIVE search_graph(id, link, data, depth, is_cycle, path) AS (
    SELECT g.id, g.link, g.data, 0,
      false,
      ARRAY[g.id]
    FROM graph g
  UNION ALL
    SELECT g.id, g.link, g.data, sg.depth + 1,
      g.id = ANY(path),
      path || g.id
    FROM graph g, search_graph sg
    WHERE g.id = sg.link AND NOT is_cycle
)
SELECT * FROM search_graph;
```

Neben dem Verhindern von Zyklen ist der Arraywert oft selbst nützlich, weil er den Pfad darstellt, auf dem eine Zeile erreicht wurde. Wenn mehrere Felder verglichen werden müssen, verwenden Sie ein Array von Zeilen.

PostgreSQL bietet eingebaute Syntax zur Vereinfachung der Zyklenerkennung:

```sql
WITH RECURSIVE search_graph(id, link, data, depth) AS (
    SELECT g.id, g.link, g.data, 1
    FROM graph g
  UNION ALL
    SELECT g.id, g.link, g.data, sg.depth + 1
    FROM graph g, search_graph sg
    WHERE g.id = sg.link
) CYCLE id SET is_cycle USING path
SELECT * FROM search_graph;
```

Die `CYCLE`-Klausel gibt zuerst die Spalten an, die zur Zyklenerkennung verfolgt werden, dann den Namen einer Spalte, die anzeigt, ob ein Zyklus erkannt wurde, und schließlich den Namen einer weiteren Spalte, die den Pfad verfolgt. Zyklus- und Pfadspalte werden implizit zur Ausgabe der CTE hinzugefügt.

> **Tipp:** Die Zykluspfadspalte wird auf dieselbe Weise berechnet wie die Tiefensuche-Sortierspalte aus dem vorigen Abschnitt. Eine Abfrage kann sowohl `SEARCH` als auch `CYCLE` enthalten; bei Tiefensuche wäre das aber oft doppelte Arbeit, sodass es effizienter ist, nur `CYCLE` zu verwenden und nach der Pfadspalte zu sortieren.

Ein hilfreicher Trick beim Testen potenziell endloser Abfragen ist ein `LIMIT` in der äußeren Abfrage:

```sql
WITH RECURSIVE t(n) AS (
    SELECT 1
  UNION ALL
    SELECT n+1 FROM t
)
SELECT n FROM t LIMIT 100;
```

Das funktioniert in PostgreSQL, weil nur so viele Zeilen einer `WITH`-Abfrage ausgewertet werden, wie die Elternabfrage tatsächlich abruft. Für Produktivcode sollte man sich darauf nicht verlassen, und es funktioniert meist nicht, wenn die äußere Abfrage die rekursiven Ergebnisse sortiert oder mit einer anderen Tabelle verbindet.

### 7.8.3. Materialisierung von Common Table Expressions

Eine nützliche Eigenschaft von `WITH`-Abfragen ist, dass sie normalerweise nur einmal pro Ausführung der Elternabfrage ausgewertet werden, auch wenn die Elternabfrage oder andere `WITH`-Abfragen sie mehrfach referenzieren. Teure Berechnungen, die an mehreren Stellen benötigt werden, können so in eine `WITH`-Abfrage verschoben werden. Ebenso kann verhindert werden, dass Funktionen mit Seiteneffekten mehrfach ausgeführt werden.

Die Kehrseite ist, dass der Optimierer Einschränkungen aus der Elternabfrage nicht in eine mehrfach referenzierte `WITH`-Abfrage hinunterschieben kann, weil das alle Verwendungen der Ausgabe beeinflussen könnte. Die mehrfach referenzierte `WITH`-Abfrage wird wie geschrieben ausgewertet, ohne Zeilen zu unterdrücken, die die Elternabfrage später verwerfen würde.

Wenn eine `WITH`-Abfrage jedoch nicht rekursiv und frei von Seiteneffekten ist, also ein `SELECT` ohne volatile Funktionen, kann sie in die Elternabfrage eingefaltet werden. Standardmäßig geschieht das, wenn die Elternabfrage die `WITH`-Abfrage nur einmal referenziert. Sie können diese Entscheidung mit `MATERIALIZED` oder `NOT MATERIALIZED` übersteuern.

```sql
WITH w AS (
    SELECT * FROM big_table
)
SELECT * FROM w WHERE key = 123;
```

Diese `WITH`-Abfrage wird eingefaltet und erzeugt denselben Ausführungsplan wie:

```sql
SELECT * FROM big_table WHERE key = 123;
```

Anders ist der Fall bei mehrfacher Referenz:

```sql
WITH w AS (
    SELECT * FROM big_table
)
SELECT * FROM w AS w1 JOIN w AS w2 ON w1.key = w2.ref
WHERE w2.key = 123;
```

Hier wird `w` materialisiert. Effizienter kann sein:

```sql
WITH w AS NOT MATERIALIZED (
    SELECT * FROM big_table
)
SELECT * FROM w AS w1 JOIN w AS w2 ON w1.key = w2.ref
WHERE w2.key = 123;
```

Ein Fall, in dem `NOT MATERIALIZED` unerwünscht sein kann:

```sql
WITH w AS (
    SELECT key, very_expensive_function(val) AS f FROM some_table
)
SELECT * FROM w AS w1 JOIN w AS w2 ON w1.f = w2.f;
```

Hier stellt Materialisierung sicher, dass `very_expensive_function` nur einmal pro Tabellenzeile ausgewertet wird.

### 7.8.4. Datenändernde Anweisungen in `WITH`

Sie können datenändernde Anweisungen (`INSERT`, `UPDATE`, `DELETE` oder `MERGE`) in `WITH` verwenden. Dadurch lassen sich mehrere Operationen in einer einzigen Abfrage ausführen:

```sql
WITH moved_rows AS (
    DELETE FROM products
    WHERE
        "date" >= '2010-10-01' AND
        "date" < '2010-11-01'
    RETURNING *
)
INSERT INTO products_log
SELECT * FROM moved_rows;
```

Diese Abfrage verschiebt effektiv Zeilen von `products` nach `products_log`. Das `DELETE` in `WITH` löscht die angegebenen Zeilen aus `products` und gibt ihren Inhalt über `RETURNING` zurück; die Hauptabfrage liest diese Ausgabe und fügt sie in `products_log` ein.

Datenändernde Anweisungen in `WITH` haben normalerweise `RETURNING`-Klauseln (siehe [Abschnitt 6.4](06_Datenmanipulation.md#64-daten-aus-geänderten-zeilen-zurückgeben)). Die Ausgabe der `RETURNING`-Klausel, nicht die Zieltabelle der datenändernden Anweisung, bildet die temporäre Tabelle, auf die der Rest der Abfrage verweisen kann. Fehlt `RETURNING`, wird die Anweisung trotzdem ausgeführt, bildet aber keine referenzierbare temporäre Tabelle.

```sql
WITH t AS (
    DELETE FROM foo
)
DELETE FROM bar;
```

Dieses Beispiel entfernt alle Zeilen aus `foo` und `bar`; die dem Client gemeldete Zahl betroffener Zeilen umfasst nur die aus `bar` entfernten Zeilen.

Rekursive Selbstverweise in datenändernden Anweisungen sind nicht erlaubt. Manchmal lässt sich das umgehen, indem auf die Ausgabe eines rekursiven `WITH` verwiesen wird:

```sql
WITH RECURSIVE included_parts(sub_part, part) AS (
    SELECT sub_part, part FROM parts WHERE part = 'our_product'
  UNION ALL
    SELECT p.sub_part, p.part
    FROM included_parts pr, parts p
    WHERE p.part = pr.sub_part
)
DELETE FROM parts
  WHERE part IN (SELECT part FROM included_parts);
```

Datenändernde Anweisungen in `WITH` werden genau einmal und immer bis zum Ende ausgeführt, unabhängig davon, ob die Hauptabfrage ihre Ausgabe ganz, teilweise oder gar nicht liest. Das unterscheidet sich von `SELECT` in `WITH`, dessen Ausführung nur so weit vorangetrieben wird, wie die Hauptabfrage seine Ausgabe benötigt.

Die Unteranweisungen in `WITH` werden gleichzeitig miteinander und mit der Hauptabfrage ausgeführt. Die tatsächliche Reihenfolge der Änderungen ist daher unvorhersehbar. Alle Anweisungen verwenden denselben Snapshot (siehe [Kapitel 13](13_Nebenläufigkeitskontrolle.md)), sodass sie die Auswirkungen der jeweils anderen auf die Zieltabellen nicht sehen können. `RETURNING` ist deshalb der einzige Weg, Änderungen zwischen verschiedenen `WITH`-Unteranweisungen und der Hauptabfrage zu kommunizieren.

```sql
WITH t AS (
    UPDATE products SET price = price * 1.05
    RETURNING *
)
SELECT * FROM products;
```

Die äußere Abfrage gibt hier die ursprünglichen Preise vor dem `UPDATE` zurück. Dagegen gibt diese Abfrage die aktualisierten Daten zurück:

```sql
WITH t AS (
    UPDATE products SET price = price * 1.05
    RETURNING *
)
SELECT * FROM t;
```

Dieselbe Zeile zweimal in einer einzigen Anweisung zu aktualisieren, wird nicht unterstützt. Nur eine der Änderungen findet statt, aber es ist nicht zuverlässig vorhersagbar, welche. Dasselbe gilt für das Löschen einer Zeile, die in derselben Anweisung bereits aktualisiert wurde: Nur die Aktualisierung wird ausgeführt. Vermeiden Sie daher `WITH`-Unteranweisungen, die dieselben Zeilen beeinflussen könnten wie die Hauptanweisung oder eine gleichrangige Unteranweisung.

Derzeit darf eine Tabelle, die Ziel einer datenändernden Anweisung in `WITH` ist, keine bedingte Regel, keine `ALSO`-Regel und keine `INSTEAD`-Regel haben, die zu mehreren Anweisungen expandiert.
