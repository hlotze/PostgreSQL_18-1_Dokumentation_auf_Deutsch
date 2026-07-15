# 9. Funktionen und Operatoren

PostgreSQL stellt eine große Zahl von Funktionen und Operatoren für die eingebauten Datentypen bereit. Dieses Kapitel beschreibt die meisten davon, auch wenn zusätzliche Spezialfunktionen in den jeweils passenden Abschnitten des Handbuchs erscheinen. Benutzer können außerdem eigene Funktionen und Operatoren definieren, wie in Teil V beschrieben. Die `psql`-Befehle `\df` und `\do` können verwendet werden, um alle verfügbaren Funktionen beziehungsweise Operatoren aufzulisten.

Die in diesem Kapitel verwendete Notation zur Beschreibung der Argument- und Ergebnistypen einer Funktion oder eines Operators sieht so aus:

```text
repeat ( text, integer ) -> text
```

Das bedeutet, dass die Funktion `repeat` ein Argument vom Typ `text` und ein Argument vom Typ `integer` entgegennimmt und ein Ergebnis vom Typ `text` zurückgibt. Der rechte Pfeil wird außerdem verwendet, um das Ergebnis eines Beispiels anzuzeigen:

```text
repeat('Pg', 4) -> PgPgPgPg
```

Wenn Ihnen Portabilität wichtig ist, beachten Sie, dass die meisten in diesem Kapitel beschriebenen Funktionen und Operatoren, mit Ausnahme der einfachsten arithmetischen und Vergleichsoperatoren sowie einiger ausdrücklich gekennzeichneter Funktionen, nicht durch den SQL-Standard spezifiziert sind. Ein Teil dieser erweiterten Funktionalität ist auch in anderen SQL-Datenbankmanagementsystemen vorhanden, und in vielen Fällen ist sie zwischen den verschiedenen Implementierungen kompatibel und konsistent.

## 9.1. Logische Operatoren

Die üblichen logischen Operatoren stehen zur Verfügung:

```text
boolean AND boolean -> boolean
boolean OR boolean -> boolean
NOT boolean -> boolean
```

SQL verwendet ein dreiwertiges Logiksystem mit `true`, `false` und `null`, wobei `null` für "unbekannt" steht. Beachten Sie die folgenden Wahrheitstabellen:

| a | b | a AND b | a OR b |
|---|---|---|---|
| TRUE | TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE | TRUE |
| TRUE | NULL | NULL | TRUE |
| FALSE | FALSE | FALSE | FALSE |
| FALSE | NULL | FALSE | NULL |
| NULL | NULL | NULL | NULL |

| a | NOT a |
|---|---|
| TRUE | FALSE |
| FALSE | TRUE |
| NULL | NULL |

Die Operatoren `AND` und `OR` sind kommutativ, das heißt, Sie können den linken und den rechten Operanden vertauschen, ohne das Ergebnis zu verändern. Es ist jedoch nicht garantiert, dass der linke Operand vor dem rechten ausgewertet wird. Weitere Informationen zur Auswertungsreihenfolge von Teilausdrücken finden Sie in [Abschnitt 4.2.14](04_SQL_Syntax.md#4214-regeln-für-die-ausdrucksauswertung).

## 9.2. Vergleichsfunktionen und -operatoren

Die üblichen Vergleichsoperatoren stehen zur Verfügung, wie Tabelle 9.1 zeigt.

**Tabelle 9.1. Vergleichsoperatoren**

| Operator | Beschreibung |
| --- | --- |
| `datatype < datatype` -> `boolean` | Kleiner als |
| `datatype > datatype` -> `boolean` | Größer als |
| `datatype <= datatype` -> `boolean` | Kleiner oder gleich |
| `datatype >= datatype` -> `boolean` | Größer oder gleich |
| `datatype = datatype` -> `boolean` | Gleich |
| `datatype <> datatype` -> `boolean` | Ungleich |
| `datatype != datatype` -> `boolean` | Ungleich |

> **Hinweis**
> `<>` ist die SQL-Standardschreibweise für „ungleich“. `!=` ist ein Alias und wird sehr früh beim Parsen in `<>` umgewandelt. Deshalb ist es nicht möglich, Operatoren `!=` und `<>` mit unterschiedlichem Verhalten zu implementieren.

Diese Vergleichsoperatoren sind für alle eingebauten Datentypen verfügbar, die eine natürliche Ordnung besitzen, unter anderem numerische Typen, Zeichenkettentypen und Datums-/Zeittypen. Außerdem können Arrays, zusammengesetzte Typen und Bereiche verglichen werden, sofern ihre Komponenten vergleichbar sind.

Auch Werte verwandter Datentypen lassen sich normalerweise vergleichen; zum Beispiel funktioniert `integer > bigint`. Manche solcher Fälle werden direkt durch typübergreifende Vergleichsoperatoren implementiert. Wenn kein solcher Operator verfügbar ist, wandelt der Parser den weniger allgemeinen Typ in den allgemeineren Typ um und wendet dessen Vergleichsoperator an.

Alle Vergleichsoperatoren sind binäre Operatoren und liefern Werte des Typs `boolean`. Ausdrücke wie `1 < 2 < 3` sind daher nicht gültig, weil es keinen Operator `<` gibt, der einen booleschen Wert mit `3` vergleicht. Für Bereichstests verwenden Sie die unten gezeigten `BETWEEN`-Prädikate.

Es gibt außerdem Vergleichsprädikate, wie Tabelle 9.2 zeigt. Sie verhalten sich ähnlich wie Operatoren, haben aber die vom SQL-Standard vorgeschriebene besondere Syntax.

**Tabelle 9.2. Vergleichsprädikate**

| Prädikat | Beschreibung | Beispiele |
| --- | --- | --- |
| `datatype BETWEEN datatype AND datatype` -> `boolean` | Zwischen zwei Werten, einschließlich der Bereichsgrenzen. | `2 BETWEEN 1 AND 3` -> `t`<br>`2 BETWEEN 3 AND 1` -> `f` |
| `datatype NOT BETWEEN datatype AND datatype` -> `boolean` | Nicht zwischen zwei Werten, die Negation von `BETWEEN`. | `2 NOT BETWEEN 1 AND 3` -> `f` |
| `datatype BETWEEN SYMMETRIC datatype AND datatype` -> `boolean` | Zwischen zwei Werten, nachdem die beiden Grenzwerte sortiert wurden. | `2 BETWEEN SYMMETRIC 3 AND 1` -> `t` |
| `datatype NOT BETWEEN SYMMETRIC datatype AND datatype` -> `boolean` | Nicht zwischen zwei Werten, nachdem die beiden Grenzwerte sortiert wurden. | `2 NOT BETWEEN SYMMETRIC 3 AND 1` -> `f` |
| `datatype IS DISTINCT FROM datatype` -> `boolean` | Ungleich, wobei `NULL` als vergleichbarer Wert behandelt wird. | `1 IS DISTINCT FROM NULL` -> `t` statt `NULL`<br>`NULL IS DISTINCT FROM NULL` -> `f` statt `NULL` |
| `datatype IS NOT DISTINCT FROM datatype` -> `boolean` | Gleich, wobei `NULL` als vergleichbarer Wert behandelt wird. | `1 IS NOT DISTINCT FROM NULL` -> `f` statt `NULL`<br>`NULL IS NOT DISTINCT FROM NULL` -> `t` statt `NULL` |
| `datatype IS NULL` -> `boolean` | Prüft, ob der Wert `NULL` ist. | `1.5 IS NULL` -> `f` |
| `datatype IS NOT NULL` -> `boolean` | Prüft, ob der Wert nicht `NULL` ist. | `'null' IS NOT NULL` -> `t` |
| `datatype ISNULL` -> `boolean` | Prüft, ob der Wert `NULL` ist; nicht standardisierte Syntax. |  |
| `datatype NOTNULL` -> `boolean` | Prüft, ob der Wert nicht `NULL` ist; nicht standardisierte Syntax. |  |
| `boolean IS TRUE` -> `boolean` | Prüft, ob der boolesche Ausdruck `true` ergibt. | `true IS TRUE` -> `t`<br>`NULL::boolean IS TRUE` -> `f` statt `NULL` |
| `boolean IS NOT TRUE` -> `boolean` | Prüft, ob der boolesche Ausdruck `false` oder unbekannt ergibt. | `true IS NOT TRUE` -> `f`<br>`NULL::boolean IS NOT TRUE` -> `t` statt `NULL` |
| `boolean IS FALSE` -> `boolean` | Prüft, ob der boolesche Ausdruck `false` ergibt. | `true IS FALSE` -> `f`<br>`NULL::boolean IS FALSE` -> `f` statt `NULL` |
| `boolean IS NOT FALSE` -> `boolean` | Prüft, ob der boolesche Ausdruck `true` oder unbekannt ergibt. | `true IS NOT FALSE` -> `t`<br>`NULL::boolean IS NOT FALSE` -> `t` statt `NULL` |
| `boolean IS UNKNOWN` -> `boolean` | Prüft, ob der boolesche Ausdruck unbekannt ergibt. | `true IS UNKNOWN` -> `f`<br>`NULL::boolean IS UNKNOWN` -> `t` statt `NULL` |
| `boolean IS NOT UNKNOWN` -> `boolean` | Prüft, ob der boolesche Ausdruck `true` oder `false` ergibt. | `true IS NOT UNKNOWN` -> `t`<br>`NULL::boolean IS NOT UNKNOWN` -> `f` statt `NULL` |

Das Prädikat `BETWEEN` vereinfacht Bereichstests:

```sql
a BETWEEN x AND y
```

ist gleichbedeutend mit:

```sql
a >= x AND a <= y
```

`BETWEEN` behandelt die Grenzwerte als eingeschlossen. `BETWEEN SYMMETRIC` funktioniert wie `BETWEEN`, verlangt aber nicht, dass das Argument links von `AND` kleiner oder gleich dem Argument rechts davon ist. Falls nötig, werden die beiden Grenzwerte automatisch vertauscht, sodass immer ein nicht leerer Bereich gemeint ist.

Die verschiedenen Varianten von `BETWEEN` werden mithilfe der gewöhnlichen Vergleichsoperatoren implementiert und funktionieren daher für alle vergleichbaren Datentypen.

> **Hinweis**
> Die Verwendung von `AND` in der `BETWEEN`-Syntax erzeugt eine Mehrdeutigkeit mit `AND` als logischem Operator. Deshalb sind als zweites Argument einer `BETWEEN`-Klausel nur bestimmte Ausdrucksarten erlaubt. Wenn ein komplexerer Teilausdruck nötig ist, setzen Sie ihn in Klammern.

Gewöhnliche Vergleichsoperatoren liefern `NULL`, also „unbekannt“, nicht `true` oder `false`, wenn eine Eingabe `NULL` ist. Zum Beispiel ergibt `7 = NULL` ebenso `NULL` wie `7 <> NULL`. Wenn dieses Verhalten nicht geeignet ist, verwenden Sie die Prädikate `IS [ NOT ] DISTINCT FROM`:

```sql
a IS DISTINCT FROM b
a IS NOT DISTINCT FROM b
```

Bei nicht nullwertigen Eingaben ist `IS DISTINCT FROM` dasselbe wie der Operator `<>`. Wenn jedoch beide Eingaben `NULL` sind, liefert es `false`; wenn nur eine Eingabe `NULL` ist, liefert es `true`. Entsprechend ist `IS NOT DISTINCT FROM` bei nicht nullwertigen Eingaben identisch mit `=`, liefert aber `true`, wenn beide Eingaben `NULL` sind, und `false`, wenn nur eine Eingabe `NULL` ist. Diese Prädikate behandeln `NULL` also effektiv wie einen normalen Datenwert statt wie „unbekannt“.

Um zu prüfen, ob ein Wert `NULL` ist oder nicht, verwenden Sie:

```sql
expression IS NULL
expression IS NOT NULL
```

oder die gleichwertigen, aber nicht standardisierten Prädikate:

```sql
expression ISNULL
expression NOTNULL
```

Schreiben Sie nicht `expression = NULL`, weil `NULL` nicht „gleich“ `NULL` ist. Der Nullwert steht für einen unbekannten Wert, und es ist nicht bekannt, ob zwei unbekannte Werte gleich sind.

> **Tipp**
> Manche Anwendungen erwarten möglicherweise, dass `expression = NULL` `true` ergibt, wenn `expression` den Nullwert hat. Es wird dringend empfohlen, solche Anwendungen an den SQL-Standard anzupassen. Wenn das nicht möglich ist, steht die Konfigurationsvariable `transform_null_equals` zur Verfügung. Ist sie aktiviert, wandelt PostgreSQL Klauseln der Form `x = NULL` in `x IS NULL` um.

Wenn der Ausdruck zeilenwertig ist, ist `IS NULL` wahr, wenn der Zeilenausdruck selbst `NULL` ist oder alle Felder der Zeile `NULL` sind. `IS NOT NULL` ist wahr, wenn der Zeilenausdruck selbst nicht `NULL` ist und alle Felder der Zeile nicht `NULL` sind. Wegen dieses Verhaltens liefern `IS NULL` und `IS NOT NULL` bei zeilenwertigen Ausdrücken nicht immer inverse Ergebnisse; insbesondere liefert ein Zeilenausdruck mit sowohl nullwertigen als auch nicht nullwertigen Feldern für beide Tests `false`.

Beispiele:

```sql
SELECT ROW(1,2.5,'this is a test') = ROW(1, 3, 'not the same');
SELECT ROW(table.*) IS NULL FROM table;     -- Zeilen erkennen, deren Felder alle NULL sind
SELECT ROW(table.*) IS NOT NULL FROM table; -- Zeilen erkennen, deren Felder alle nicht NULL sind
SELECT NOT(ROW(table.*) IS NOT NULL) FROM table; -- Zeilen mit mindestens einem NULL-Feld erkennen
```

In manchen Fällen ist es vorzuziehen, `row IS DISTINCT FROM NULL` oder `row IS NOT DISTINCT FROM NULL` zu schreiben; das prüft einfach, ob der Gesamtwert der Zeile `NULL` ist, ohne zusätzliche Tests auf die einzelnen Zeilenfelder.

Boolesche Werte können außerdem mit diesen Prädikaten geprüft werden:

```sql
boolean_expression IS TRUE
boolean_expression IS NOT TRUE
boolean_expression IS FALSE
boolean_expression IS NOT FALSE
boolean_expression IS UNKNOWN
boolean_expression IS NOT UNKNOWN
```

Diese Prädikate liefern immer `true` oder `false`, niemals `NULL`, auch wenn der Operand `NULL` ist. Eine nullwertige Eingabe wird als logischer Wert „unbekannt“ behandelt. `IS UNKNOWN` und `IS NOT UNKNOWN` sind effektiv dasselbe wie `IS NULL` und `IS NOT NULL`, nur dass der Eingabeausdruck vom booleschen Typ sein muss.

Einige Vergleichsfunktionen stehen ebenfalls zur Verfügung, wie Tabelle 9.3 zeigt.

**Tabelle 9.3. Vergleichsfunktionen**

| Funktion | Beschreibung | Beispiel |
| --- | --- | --- |
| `num_nonnulls ( VARIADIC "any" )` -> `integer` | Gibt die Anzahl der Nicht-`NULL`-Argumente zurück. | `num_nonnulls(1, NULL, 2)` -> `2` |
| `num_nulls ( VARIADIC "any" )` -> `integer` | Gibt die Anzahl der `NULL`-Argumente zurück. | `num_nulls(1, NULL, 2)` -> `1` |

## 9.3. Mathematische Funktionen und Operatoren

Mathematische Operatoren stehen für viele PostgreSQL-Typen zur Verfügung. Für Typen ohne übliche mathematische Konventionen, zum Beispiel Datums-/Zeittypen, wird das tatsächliche Verhalten in späteren Abschnitten beschrieben.

Tabelle 9.4 zeigt die mathematischen Operatoren für die standardmäßigen numerischen Typen. Sofern nichts anderes angegeben ist, sind Operatoren mit `numeric_type` für `smallint`, `integer`, `bigint`, `numeric`, `real` und `double precision` verfügbar. Operatoren mit `integral_type` sind für `smallint`, `integer` und `bigint` verfügbar. Wenn nichts anderes angegeben ist, gibt jede Operatorform denselben Datentyp zurück wie ihr Argument bzw. ihre Argumente. Aufrufe mit mehreren Argumentdatentypen, etwa `integer + numeric`, werden aufgelöst, indem der Typ verwendet wird, der in diesen Listen später erscheint.

**Tabelle 9.4. Mathematische Operatoren**

| Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `numeric_type + numeric_type` -> `numeric_type` | Addition | `2 + 3` -> `5` |
| `+ numeric_type` -> `numeric_type` | Unäres Plus, keine Operation | `+ 3.5` -> `3.5` |
| `numeric_type - numeric_type` -> `numeric_type` | Subtraktion | `2 - 3` -> `-1` |
| `- numeric_type` -> `numeric_type` | Negation | `- (-4)` -> `4` |
| `numeric_type * numeric_type` -> `numeric_type` | Multiplikation | `2 * 3` -> `6` |
| `numeric_type / numeric_type` -> `numeric_type` | Division; bei ganzzahligen Typen wird das Ergebnis gegen null abgeschnitten. | `5.0 / 2` -> `2.5000000000000000`<br>`5 / 2` -> `2`<br>`(-5) / 2` -> `-2` |
| `numeric_type % numeric_type` -> `numeric_type` | Modulo, also Rest; verfügbar für `smallint`, `integer`, `bigint` und `numeric`. | `5 % 4` -> `1` |
| `numeric ^ numeric` -> `numeric`<br>`double precision ^ double precision` -> `double precision` | Potenzierung. Anders als in der üblichen mathematischen Praxis gruppiert `^` standardmäßig von links nach rechts. | `2 ^ 3` -> `8`<br>`2 ^ 3 ^ 3` -> `512`<br>`2 ^ (3 ^ 3)` -> `134217728` |
| `\|/ double precision` -> `double precision` | Quadratwurzel | `\|/ 25.0` -> `5` |
| `\|\|/ double precision` -> `double precision` | Kubikwurzel | `\|\|/ 64.0` -> `4` |
| `@ numeric_type` -> `numeric_type` | Absolutwert | `@ -5.0` -> `5.0` |
| `integral_type & integral_type` -> `integral_type` | Bitweises AND | `91 & 15` -> `11` |
| `integral_type \| integral_type` -> `integral_type` | Bitweises OR | `32 \| 3` -> `35` |
| `integral_type # integral_type` -> `integral_type` | Bitweises exklusives OR | `17 # 5` -> `20` |
| `~ integral_type` -> `integral_type` | Bitweises NOT | `~1` -> `-2` |
| `integral_type << integer` -> `integral_type` | Bitverschiebung nach links | `1 << 4` -> `16` |
| `integral_type >> integer` -> `integral_type` | Bitverschiebung nach rechts | `8 >> 2` -> `2` |

Tabelle 9.5 zeigt die verfügbaren mathematischen Funktionen. Viele dieser Funktionen gibt es in mehreren Formen mit unterschiedlichen Argumenttypen. Wenn nichts anderes angegeben ist, gibt jede Form einer Funktion denselben Datentyp zurück wie ihre Argumente; typübergreifende Fälle werden wie oben für Operatoren beschrieben aufgelöst. Funktionen für `double precision` beruhen meist auf der C-Bibliothek des Hostsystems; Genauigkeit und Verhalten in Grenzfällen können daher je nach Hostsystem variieren.

**Tabelle 9.5. Mathematische Funktionen**

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `abs ( numeric_type )` -> `numeric_type` | Absolutwert | `abs(-17.4)` -> `17.4` |
| `cbrt ( double precision )` -> `double precision` | Kubikwurzel | `cbrt(64.0)` -> `4` |
| `ceil ( numeric )` -> `numeric`<br>`ceil ( double precision )` -> `double precision` | Nächste ganze Zahl größer oder gleich dem Argument | `ceil(42.2)` -> `43`<br>`ceil(-42.8)` -> `-42` |
| `ceiling ( numeric )` -> `numeric`<br>`ceiling ( double precision )` -> `double precision` | Nächste ganze Zahl größer oder gleich dem Argument; dasselbe wie `ceil` | `ceiling(95.3)` -> `96` |
| `degrees ( double precision )` -> `double precision` | Wandelt Radiant in Grad um | `degrees(0.5)` -> `28.64788975654116` |
| `div ( y numeric, x numeric )` -> `numeric` | Ganzzahliger Quotient von `y`/`x`, gegen null abgeschnitten | `div(9, 4)` -> `2` |
| `erf ( double precision )` -> `double precision` | Fehlerfunktion | `erf(1.0)` -> `0.8427007929497149` |
| `erfc ( double precision )` -> `double precision` | Komplementäre Fehlerfunktion (`1 - erf(x)`) ohne Genauigkeitsverlust bei großen Eingaben | `erfc(1.0)` -> `0.15729920705028513` |
| `exp ( numeric )` -> `numeric`<br>`exp ( double precision )` -> `double precision` | Exponentialfunktion, `e` hoch Argument | `exp(1.0)` -> `2.7182818284590452` |
| `factorial ( bigint )` -> `numeric` | Fakultät | `factorial(5)` -> `120` |
| `floor ( numeric )` -> `numeric`<br>`floor ( double precision )` -> `double precision` | Nächste ganze Zahl kleiner oder gleich dem Argument | `floor(42.8)` -> `42`<br>`floor(-42.8)` -> `-43` |
| `gamma ( double precision )` -> `double precision` | Gammafunktion | `gamma(0.5)` -> `1.772453850905516`<br>`gamma(6)` -> `120` |
| `gcd ( numeric_type, numeric_type )` -> `numeric_type` | Größter gemeinsamer Teiler; gibt `0` zurück, wenn beide Eingaben null sind. Verfügbar für `integer`, `bigint` und `numeric`. | `gcd(1071, 462)` -> `21` |
| `lcm ( numeric_type, numeric_type )` -> `numeric_type` | Kleinstes gemeinsames Vielfaches; gibt `0` zurück, wenn eine Eingabe null ist. Verfügbar für `integer`, `bigint` und `numeric`. | `lcm(1071, 462)` -> `23562` |
| `lgamma ( double precision )` -> `double precision` | Natürlicher Logarithmus des Absolutwerts der Gammafunktion | `lgamma(1000)` -> `5905.220423209181` |
| `ln ( numeric )` -> `numeric`<br>`ln ( double precision )` -> `double precision` | Natürlicher Logarithmus | `ln(2.0)` -> `0.6931471805599453` |
| `log ( numeric )` -> `numeric`<br>`log ( double precision )` -> `double precision` | Logarithmus zur Basis 10 | `log(100)` -> `2` |
| `log10 ( numeric )` -> `numeric`<br>`log10 ( double precision )` -> `double precision` | Logarithmus zur Basis 10, dasselbe wie `log` | `log10(1000)` -> `3` |
| `log ( b numeric, x numeric )` -> `numeric` | Logarithmus von `x` zur Basis `b` | `log(2.0, 64.0)` -> `6.0000000000000000` |
| `min_scale ( numeric )` -> `integer` | Minimale Skalierung, also Anzahl der Dezimalstellen, die nötig ist, um den Wert exakt darzustellen | `min_scale(8.4100)` -> `2` |
| `mod ( y numeric_type, x numeric_type )` -> `numeric_type` | Rest von `y`/`x`; verfügbar für `smallint`, `integer`, `bigint` und `numeric` | `mod(9, 4)` -> `1` |
| `pi ( )` -> `double precision` | Näherungswert von π | `pi()` -> `3.141592653589793` |
| `power ( a numeric, b numeric )` -> `numeric`<br>`power ( a double precision, b double precision )` -> `double precision` | `a` hoch `b` | `power(9, 3)` -> `729` |
| `radians ( double precision )` -> `double precision` | Wandelt Grad in Radiant um | `radians(45.0)` -> `0.7853981633974483` |
| `round ( numeric )` -> `numeric`<br>`round ( double precision )` -> `double precision` | Rundet auf die nächste ganze Zahl. Bei `numeric` werden Gleichstände von null weg gerundet; bei `double precision` ist das Verhalten plattformabhängig, meist „round to nearest even“. | `round(42.4)` -> `42` |
| `round ( v numeric, s integer )` -> `numeric` | Rundet `v` auf `s` Dezimalstellen. Gleichstände werden von null weg gerundet. | `round(42.4382, 2)` -> `42.44`<br>`round(1234.56, -1)` -> `1230` |
| `scale ( numeric )` -> `integer` | Skalierung des Arguments, also Anzahl der Dezimalstellen im Bruchteil | `scale(8.4100)` -> `4` |
| `sign ( numeric )` -> `numeric`<br>`sign ( double precision )` -> `double precision` | Vorzeichen des Arguments: `-1`, `0` oder `+1` | `sign(-8.4)` -> `-1` |
| `sqrt ( numeric )` -> `numeric`<br>`sqrt ( double precision )` -> `double precision` | Quadratwurzel | `sqrt(2)` -> `1.4142135623730951` |
| `trim_scale ( numeric )` -> `numeric` | Verringert die Skalierung des Werts, indem nachgestellte Nullen entfernt werden | `trim_scale(8.4100)` -> `8.41` |
| `trunc ( numeric )` -> `numeric`<br>`trunc ( double precision )` -> `double precision` | Schneidet auf eine ganze Zahl ab, gegen null | `trunc(42.8)` -> `42`<br>`trunc(-42.8)` -> `-42` |
| `trunc ( v numeric, s integer )` -> `numeric` | Schneidet `v` auf `s` Dezimalstellen ab | `trunc(42.4382, 2)` -> `42.43` |
| `width_bucket ( operand numeric, low numeric, high numeric, count integer )` -> `integer`<br>`width_bucket ( operand double precision, low double precision, high double precision, count integer )` -> `integer` | Gibt die Nummer des Buckets zurück, in den `operand` in einem Histogramm mit `count` gleich breiten Buckets zwischen `low` und `high` fällt. Die unteren Grenzen sind inklusiv, die oberen exklusiv. Werte unter `low` ergeben `0`, Werte größer oder gleich `high` ergeben `count+1`. Wenn `low > high`, wird das Verhalten gespiegelt. | `width_bucket(5.35, 0.024, 10.06, 5)` -> `3`<br>`width_bucket(9, 10, 0, 10)` -> `2` |
| `width_bucket ( operand anycompatible, thresholds anycompatiblearray )` -> `integer` | Gibt die Nummer des Buckets zurück, in den `operand` fällt, wenn ein Array die inklusiven unteren Bucket-Grenzen enthält. `operand` und Array-Elemente können jeden Typ mit Standardvergleichsoperatoren haben. Das Array `thresholds` muss aufsteigend sortiert sein. | `width_bucket(now(), array['yesterday', 'today', 'tomorrow']::timestamptz[])` -> `2` |

Tabelle 9.6 zeigt Funktionen zur Erzeugung von Zufallszahlen.

**Tabelle 9.6. Zufallsfunktionen**

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `random ( )` -> `double precision` | Gibt einen Zufallswert im Bereich `0.0 <= x < 1.0` zurück. | `random()` -> `0.897124072839091` |
| `random ( min integer, max integer )` -> `integer`<br>`random ( min bigint, max bigint )` -> `bigint`<br>`random ( min numeric, max numeric )` -> `numeric` | Gibt einen Zufallswert im Bereich `min <= x <= max` zurück. Für `numeric` hat das Ergebnis so viele Nachkommastellen wie `min` oder `max`, je nachdem, welcher Wert mehr hat. | `random(1, 10)` -> `7`<br>`random(-0.499, 0.499)` -> `0.347` |
| `random_normal ( [ mean double precision [, stddev double precision ]] )` -> `double precision` | Gibt einen Zufallswert aus der Normalverteilung mit den angegebenen Parametern zurück; `mean` ist standardmäßig `0.0`, `stddev` standardmäßig `1.0`. | `random_normal(0.0, 1.0)` -> `0.051285419` |
| `setseed ( double precision )` -> `void` | Setzt den Startwert für nachfolgende Aufrufe von `random()` und `random_normal()`; das Argument muss zwischen `-1.0` und `1.0` einschließlich liegen. | `setseed(0.12345)` |

Die in Tabelle 9.6 aufgeführten Funktionen `random()` und `random_normal()` verwenden einen deterministischen Pseudozufallszahlengenerator. Er ist schnell, aber nicht für kryptographische Anwendungen geeignet; für eine sicherere Alternative siehe das Modul `pgcrypto`. Wenn `setseed()` aufgerufen wird, kann die Ergebnisfolge nachfolgender Aufrufe in der aktuellen Sitzung wiederholt werden, indem `setseed()` erneut mit demselben Argument aufgerufen wird. Ohne vorherigen `setseed()`-Aufruf in derselben Sitzung bezieht der erste Aufruf einer dieser Funktionen einen Startwert aus einer plattformabhängigen Quelle zufälliger Bits.

Tabelle 9.7 zeigt die verfügbaren trigonometrischen Funktionen. Jede dieser Funktionen gibt es in zwei Varianten: eine misst Winkel im Bogenmaß, die andere in Grad.

**Tabelle 9.7. Trigonometrische Funktionen**

| Funktion | Beschreibung | Beispiel |
| --- | --- | --- |
| `acos ( double precision )` -> `double precision` | Arkuskosinus, Ergebnis im Bogenmaß | `acos(1)` -> `0` |
| `acosd ( double precision )` -> `double precision` | Arkuskosinus, Ergebnis in Grad | `acosd(0.5)` -> `60` |
| `asin ( double precision )` -> `double precision` | Arkussinus, Ergebnis im Bogenmaß | `asin(1)` -> `1.5707963267948966` |
| `asind ( double precision )` -> `double precision` | Arkussinus, Ergebnis in Grad | `asind(0.5)` -> `30` |
| `atan ( double precision )` -> `double precision` | Arkustangens, Ergebnis im Bogenmaß | `atan(1)` -> `0.7853981633974483` |
| `atand ( double precision )` -> `double precision` | Arkustangens, Ergebnis in Grad | `atand(1)` -> `45` |
| `atan2 ( y double precision, x double precision )` -> `double precision` | Arkustangens von `y`/`x`, Ergebnis im Bogenmaß | `atan2(1, 0)` -> `1.5707963267948966` |
| `atan2d ( y double precision, x double precision )` -> `double precision` | Arkustangens von `y`/`x`, Ergebnis in Grad | `atan2d(1, 0)` -> `90` |
| `cos ( double precision )` -> `double precision` | Kosinus, Argument im Bogenmaß | `cos(0)` -> `1` |
| `cosd ( double precision )` -> `double precision` | Kosinus, Argument in Grad | `cosd(60)` -> `0.5` |
| `cot ( double precision )` -> `double precision` | Kotangens, Argument im Bogenmaß | `cot(0.5)` -> `1.830487721712452` |
| `cotd ( double precision )` -> `double precision` | Kotangens, Argument in Grad | `cotd(45)` -> `1` |
| `sin ( double precision )` -> `double precision` | Sinus, Argument im Bogenmaß | `sin(1)` -> `0.8414709848078965` |
| `sind ( double precision )` -> `double precision` | Sinus, Argument in Grad | `sind(30)` -> `0.5` |
| `tan ( double precision )` -> `double precision` | Tangens, Argument im Bogenmaß | `tan(1)` -> `1.5574077246549023` |
| `tand ( double precision )` -> `double precision` | Tangens, Argument in Grad | `tand(45)` -> `1` |

> **Hinweis**
> Eine andere Möglichkeit, mit Winkeln in Grad zu arbeiten, ist die Verwendung der oben gezeigten Umrechnungsfunktionen `radians()` und `degrees()`. Die gradbasierten trigonometrischen Funktionen sind jedoch vorzuziehen, weil sie Rundungsfehler in Sonderfällen wie `sind(30)` vermeiden.

Tabelle 9.8 zeigt die verfügbaren hyperbolischen Funktionen.

**Tabelle 9.8. Hyperbolische Funktionen**

| Funktion | Beschreibung | Beispiel |
| --- | --- | --- |
| `sinh ( double precision )` -> `double precision` | Hyperbolischer Sinus | `sinh(1)` -> `1.1752011936438014` |
| `cosh ( double precision )` -> `double precision` | Hyperbolischer Kosinus | `cosh(0)` -> `1` |
| `tanh ( double precision )` -> `double precision` | Hyperbolischer Tangens | `tanh(1)` -> `0.7615941559557649` |
| `asinh ( double precision )` -> `double precision` | Inverser hyperbolischer Sinus | `asinh(1)` -> `0.881373587019543` |
| `acosh ( double precision )` -> `double precision` | Inverser hyperbolischer Kosinus | `acosh(1)` -> `0` |
| `atanh ( double precision )` -> `double precision` | Inverser hyperbolischer Tangens | `atanh(0.5)` -> `0.5493061443340548` |

## 9.4. Zeichenkettenfunktionen und -operatoren

Dieser Abschnitt beschreibt Funktionen und Operatoren zum Untersuchen und Bearbeiten von Zeichenkettenwerten. Zeichenketten umfassen hier Werte der Typen `character`, `character varying` und `text`. Sofern nichts anderes angegeben ist, akzeptieren und liefern diese Funktionen und Operatoren den Typ `text`. Argumente vom Typ `character varying` werden austauschbar akzeptiert. Werte vom Typ `character` werden vor der Anwendung der Funktion oder des Operators in `text` umgewandelt; dabei werden nachgestellte Leerzeichen entfernt.

SQL definiert einige Zeichenkettenfunktionen, die Schlüsselwörter statt Kommas zur Trennung von Argumenten verwenden. Details stehen in Tabelle 9.9. PostgreSQL stellt außerdem Varianten mit normaler Funktionsaufrufsyntax bereit (siehe Tabelle 9.10).

> **Hinweis**
> Der Zeichenkettenverkettungsoperator `||` akzeptiert auch Nicht-Zeichenketteneingaben, sofern mindestens eine Eingabe ein Zeichenkettentyp ist, wie Tabelle 9.9 zeigt. In anderen Fällen kann eine explizite Umwandlung nach `text` verwendet werden.

**Tabelle 9.9. SQL-Zeichenkettenfunktionen und -operatoren**

| Funktion/Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `text || text` -> `text` | Verkettet zwei Zeichenketten. | `'Post' || 'greSQL'` -> `PostgreSQL` |
| `text || anynonarray` -> `text`<br>`anynonarray || text` -> `text` | Wandelt die Nicht-Zeichenketteneingabe in `text` um und verkettet dann beide Zeichenketten. Die Nicht-Zeichenketteneingabe darf kein Array sein, weil das mit dem Array-Operator `||` mehrdeutig wäre. | `'Value: ' || 42` -> `Value: 42` |
| `btrim ( string text [, characters text ] )` -> `text` | Entfernt am Anfang und Ende von `string` die längste Zeichenfolge, die nur Zeichen aus `characters` enthält; voreingestellt ist ein Leerzeichen. | `btrim('xyxtrimyyx', 'xyz')` -> `trim` |
| `text IS [NOT] [form] NORMALIZED` -> `boolean` | Prüft, ob die Zeichenkette in der angegebenen Unicode-Normalisierungsform vorliegt. `form` kann `NFC` (Voreinstellung), `NFD`, `NFKC` oder `NFKD` sein. Nur bei Serverkodierung `UTF8` verwendbar. | `U&'\0061\0308bc' IS NFD NORMALIZED` -> `t` |
| `bit_length ( text )` -> `integer` | Gibt die Anzahl der Bits in der Zeichenkette zurück, also das Achtfache von `octet_length`. | `bit_length('jose')` -> `32` |
| `char_length ( text )` -> `integer`<br>`character_length ( text )` -> `integer` | Gibt die Anzahl der Zeichen in der Zeichenkette zurück. | `char_length('josé')` -> `4` |
| `lower ( text )` -> `text` | Wandelt die Zeichenkette nach den Regeln der Datenbank-Locale in Kleinbuchstaben um. | `lower('TOM')` -> `tom` |
| `lpad ( string text, length integer [, fill text ] )` -> `text` | Verlängert `string` auf `length`, indem links Zeichen aus `fill` vorangestellt werden; voreingestellt ist ein Leerzeichen. Ist `string` bereits länger, wird rechts abgeschnitten. | `lpad('hi', 5, 'xy')` -> `xyxhi` |
| `ltrim ( string text [, characters text ] )` -> `text` | Entfernt am Anfang von `string` die längste Zeichenfolge, die nur Zeichen aus `characters` enthält; voreingestellt ist ein Leerzeichen. | `ltrim('zzzytest', 'xyz')` -> `test` |
| `normalize ( text [, form ] )` -> `text` | Wandelt die Zeichenkette in die angegebene Unicode-Normalisierungsform um. `form` kann `NFC` (Voreinstellung), `NFD`, `NFKC` oder `NFKD` sein. Nur bei Serverkodierung `UTF8` verwendbar. | `normalize(U&'\0061\0308bc', NFC)` -> `U&'\00E4bc'` |
| `octet_length ( text )` -> `integer` | Gibt die Anzahl der Bytes in der Zeichenkette zurück. | `octet_length('josé')` -> `5` bei Serverkodierung `UTF8` |
| `octet_length ( character )` -> `integer` | Gibt die Anzahl der Bytes in der Zeichenkette zurück. Diese Variante akzeptiert `character` direkt und entfernt deshalb keine nachgestellten Leerzeichen. | `octet_length('abc '::character(4))` -> `4` |
| `overlay ( string text PLACING newsubstring text FROM start integer [ FOR count integer ] )` -> `text` | Ersetzt den Teilstring von `string`, der beim `start`-ten Zeichen beginnt und `count` Zeichen lang ist, durch `newsubstring`. Fehlt `count`, wird die Länge von `newsubstring` verwendet. | `overlay('Txxxxas' placing 'hom' from 2 for 4)` -> `Thomas` |
| `position ( substring text IN string text )` -> `integer` | Gibt die erste Startposition von `substring` in `string` zurück, oder null, wenn der Teilstring nicht vorkommt. | `position('om' in 'Thomas')` -> `3` |
| `rpad ( string text, length integer [, fill text ] )` -> `text` | Verlängert `string` auf `length`, indem rechts Zeichen aus `fill` angehängt werden; voreingestellt ist ein Leerzeichen. Ist `string` bereits länger, wird abgeschnitten. | `rpad('hi', 5, 'xy')` -> `hixyx` |
| `rtrim ( string text [, characters text ] )` -> `text` | Entfernt am Ende von `string` die längste Zeichenfolge, die nur Zeichen aus `characters` enthält; voreingestellt ist ein Leerzeichen. | `rtrim('testxxzx', 'xyz')` -> `test` |
| `substring ( string text [ FROM start integer ] [ FOR count integer ] )` -> `text` | Extrahiert den Teilstring von `string`, beginnend beim `start`-ten Zeichen, falls angegeben, und höchstens `count` Zeichen lang, falls angegeben. Mindestens eines von `start` und `count` muss angegeben werden. | `substring('Thomas' from 2 for 3)` -> `hom`<br>`substring('Thomas' from 3)` -> `omas`<br>`substring('Thomas' for 2)` -> `Th` |
| `substring ( string text FROM pattern text )` -> `text` | Extrahiert den ersten Teilstring, der auf einen POSIX-regulären Ausdruck passt; siehe Abschnitt 9.7.3. | `substring('Thomas' from '...$')` -> `mas` |
| `substring ( string text SIMILAR pattern text ESCAPE escape text )` -> `text`<br>`substring ( string text FROM pattern text FOR escape text )` -> `text` | Extrahiert den ersten Teilstring, der auf einen SQL-regulären Ausdruck passt; siehe Abschnitt 9.7.2. Die erste Form ist seit SQL:2003 spezifiziert; die zweite stammt nur aus SQL:1999 und gilt als veraltet. | `substring('Thomas' similar '%#"o_a#"_' escape '#')` -> `oma` |
| `trim ( [ LEADING \| TRAILING \| BOTH ] [ characters text ] FROM string text )` -> `text` | Entfernt am Anfang, Ende oder an beiden Enden von `string` die längste Zeichenfolge, die nur Zeichen aus `characters` enthält; voreingestellt sind `BOTH` und Leerzeichen. | `trim(both 'xyz' from 'yxTomxx')` -> `Tom` |
| `trim ( [ LEADING \| TRAILING \| BOTH ] [ FROM ] string text [, characters text ] )` -> `text` | Nicht standardisierte Syntax für `trim()`. | `trim(both from 'yxTomxx', 'xyz')` -> `Tom` |
| `unicode_assigned ( text )` -> `boolean` | Gibt `true` zurück, wenn allen Zeichen der Zeichenkette Unicode-Codepunkte zugewiesen sind, sonst `false`. Nur bei Serverkodierung `UTF8` verwendbar. |  |
| `upper ( text )` -> `text` | Wandelt die Zeichenkette nach den Regeln der Datenbank-Locale in Großbuchstaben um. | `upper('tom')` -> `TOM` |

Zusätzliche Zeichenkettenfunktionen und -operatoren sind in Tabelle 9.10 aufgeführt. Einige davon werden intern verwendet, um die SQL-Standardfunktionen aus Tabelle 9.9 zu implementieren. Außerdem gibt es Mustervergleichsoperatoren, die in [Abschnitt 9.7](09_Funktionen_und_Operatoren.md#97-mustervergleich) beschrieben werden, und Operatoren für Volltextsuche, die in [Kapitel 12](12_Volltextsuche.md) beschrieben werden.

**Tabelle 9.10. Weitere Zeichenkettenfunktionen und -operatoren**

| Funktion/Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `text ^@ text` -> `boolean` | Gibt `true` zurück, wenn die erste Zeichenkette mit der zweiten beginnt; entspricht `starts_with()`. | `'alphabet' ^@ 'alph'` -> `t` |
| `ascii ( text )` -> `integer` | Gibt den numerischen Code des ersten Zeichens zurück. Bei `UTF8` ist das der Unicode-Codepunkt; bei anderen Mehrbytekodierungen muss das Argument ein ASCII-Zeichen sein. | `ascii('x')` -> `120` |
| `chr ( integer )` -> `text` | Gibt das Zeichen mit dem angegebenen Code zurück. Bei `UTF8` wird das Argument als Unicode-Codepunkt behandelt. `chr(0)` ist nicht erlaubt, weil `text` dieses Zeichen nicht speichern kann. | `chr(65)` -> `A` |
| `concat ( val1 "any" [, val2 "any" [, ...] ] )` -> `text` | Verkettet die Textdarstellungen aller Argumente. `NULL`-Argumente werden ignoriert. | `concat('abcde', 2, NULL, 22)` -> `abcde222` |
| `concat_ws ( sep text, val1 "any" [, val2 "any" [, ...] ] )` -> `text` | Verkettet alle Argumente außer dem ersten mit Trennzeichen. Das erste Argument ist das Trennzeichen und sollte nicht `NULL` sein. Weitere `NULL`-Argumente werden ignoriert. | `concat_ws(',', 'abcde', 2, NULL, 22)` -> `abcde,2,22` |
| `format ( formatstr text [, formatarg "any" [, ...] ] )` -> `text` | Formatiert Argumente nach einer Formatzeichenkette; siehe [Abschnitt 9.4.1](#941-format). Ähnlich der C-Funktion `sprintf`. | `format('Hello %s, %1$s', 'World')` -> `Hello World, World` |
| `initcap ( text )` -> `text` | Wandelt den ersten Buchstaben jedes Wortes in Großbuchstaben und den Rest in Kleinbuchstaben um. | `initcap('hi THOMAS')` -> `Hi Thomas` |
| `casefold ( text )` -> `text` | Führt Case Folding nach der Kollation aus. Das erleichtert groß-/kleinschreibungsunabhängigen Vergleich. Nur bei Serverkodierung `UTF8` verwendbar. Case Folding kann die Länge der Zeichenkette ändern; in der Kollation `PG_UNICODE_FAST` wird etwa `ß` zu `ss`. Beim Provider `libc` ist `casefold` identisch mit `lower`. |  |
| `left ( string text, n integer )` -> `text` | Gibt die ersten `n` Zeichen zurück; ist `n` negativ, alle außer den letzten `|n|` Zeichen. | `left('abcde', 2)` -> `ab` |
| `length ( text )` -> `integer` | Gibt die Anzahl der Zeichen in der Zeichenkette zurück. | `length('jose')` -> `4` |
| `md5 ( text )` -> `text` | Berechnet den MD5-Hash des Arguments und gibt ihn hexadezimal aus. | `md5('abc')` -> `900150983cd24fb0d6963f7d28e17f72` |
| `parse_ident ( qualified_identifier text [, strict_mode boolean DEFAULT true ] )` -> `text[]` | Zerlegt `qualified_identifier` in ein Array von Bezeichnern und entfernt dabei Quoting. Standardmäßig sind zusätzliche Zeichen nach dem letzten Bezeichner ein Fehler; mit `false` werden sie ignoriert. Überlange Bezeichner werden nicht gekürzt. | `parse_ident('"SomeSchema".someTable')` -> `{SomeSchema,sometable}` |
| `pg_client_encoding ( )` -> `name` | Gibt den Namen der aktuellen Client-Kodierung zurück. | `pg_client_encoding()` -> `UTF8` |
| `quote_ident ( text )` -> `text` | Quotiert die Zeichenkette passend als SQL-Bezeichner. Anführungszeichen werden nur hinzugefügt, wenn nötig; eingebettete Anführungszeichen werden verdoppelt. Siehe auch Beispiel 41.1. | `quote_ident('Foo bar')` -> `"Foo bar"` |
| `quote_literal ( text )` -> `text` | Quotiert die Zeichenkette passend als SQL-Zeichenkettenliteral. Eingebettete einfache Anführungszeichen und Backslashes werden verdoppelt. Gibt bei `NULL`-Eingabe `NULL` zurück; oft ist dann `quote_nullable` geeigneter. | `quote_literal(E'O\'Reilly')` -> `'O''Reilly'` |
| `quote_literal ( anyelement )` -> `text` | Wandelt den Wert in `text` um und quotiert ihn als Literal. | `quote_literal(42.5)` -> `'42.5'` |
| `quote_nullable ( text )` -> `text` | Quotiert die Zeichenkette als SQL-Zeichenkettenliteral oder gibt bei `NULL`-Argument die Zeichenkette `NULL` zurück. | `quote_nullable(NULL)` -> `NULL` |
| `quote_nullable ( anyelement )` -> `text` | Wandelt den Wert in `text` um und quotiert ihn als Literal oder gibt bei `NULL`-Argument `NULL` zurück. | `quote_nullable(42.5)` -> `'42.5'` |
| `regexp_count ( string text, pattern text [, start integer [, flags text ] ] )` -> `integer` | Gibt zurück, wie oft der POSIX-reguläre Ausdruck in `string` passt; siehe Abschnitt 9.7.3. | `regexp_count('123456789012', '\d\d\d', 2)` -> `3` |
| `regexp_instr ( string text, pattern text [, start integer [, N integer [, endoption integer [, flags text [, subexpr integer ] ] ] ] ] )` -> `integer` | Gibt die Position zurück, an der der `N`-te Treffer des POSIX-regulären Ausdrucks in `string` vorkommt, oder null, wenn es keinen Treffer gibt; siehe Abschnitt 9.7.3. | `regexp_instr('ABCDEF', 'c(.)(..)', 1, 1, 0, 'i')` -> `3`<br>`regexp_instr('ABCDEF', 'c(.)(..)', 1, 1, 0, 'i', 2)` -> `5` |
| `regexp_like ( string text, pattern text [, flags text ] )` -> `boolean` | Prüft, ob der POSIX-reguläre Ausdruck in `string` vorkommt; siehe Abschnitt 9.7.3. | `regexp_like('Hello World', 'world$', 'i')` -> `t` |
| `regexp_match ( string text, pattern text [, flags text ] )` -> `text[]` | Gibt Teilzeichenketten des ersten Treffers des POSIX-regulären Ausdrucks zurück; siehe Abschnitt 9.7.3. | `regexp_match('foobarbequebaz', '(bar)(beque)')` -> `{bar,beque}` |
| `regexp_matches ( string text, pattern text [, flags text ] )` -> `setof text[]` | Gibt Teilzeichenketten des ersten Treffers oder, mit Flag `g`, aller Treffer zurück; siehe Abschnitt 9.7.3. | `regexp_matches('foobarbequebaz', 'ba.', 'g')` -> `{bar}` und `{baz}` |
| `regexp_replace ( string text, pattern text, replacement text [, flags text ] )` -> `text` | Ersetzt den ersten Treffer des POSIX-regulären Ausdrucks oder, mit Flag `g`, alle Treffer; siehe Abschnitt 9.7.3. | `regexp_replace('Thomas', '.[mN]a.', 'M')` -> `ThM` |
| `regexp_replace ( string text, pattern text, replacement text, start integer [, N integer [, flags text ] ] )` -> `text` | Ersetzt den `N`-ten Treffer, beginnend beim `start`-ten Zeichen von `string`; wenn `N` null ist, werden alle Treffer ersetzt. | `regexp_replace('Thomas', '.', 'X', 3, 2)` -> `ThoXas`<br>`regexp_replace(string=>'hello world', pattern=>'l', replacement=>'XX', start=>1, "N"=>2)` -> `helXXo world` |
| `regexp_split_to_array ( string text, pattern text [, flags text ] )` -> `text[]` | Teilt `string` mit einem POSIX-regulären Ausdruck als Trennmuster und liefert ein Array; siehe Abschnitt 9.7.3. | `regexp_split_to_array('hello world', '\s+')` -> `{hello,world}` |
| `regexp_split_to_table ( string text, pattern text [, flags text ] )` -> `setof text` | Teilt `string` mit einem POSIX-regulären Ausdruck als Trennmuster und liefert eine Ergebnismenge; siehe Abschnitt 9.7.3. | `regexp_split_to_table('hello world', '\s+')` -> Zeilen `hello`, `world` |
| `regexp_substr ( string text, pattern text [, start integer [, N integer [, flags text [, subexpr integer ] ] ] ] )` -> `text` | Gibt den Teilstring zurück, der dem `N`-ten Treffer des POSIX-regulären Ausdrucks entspricht, oder `NULL`; siehe Abschnitt 9.7.3. | `regexp_substr('ABCDEF', 'c(.)(..)', 1, 1, 'i')` -> `CDEF`<br>`regexp_substr('ABCDEF', 'c(.)(..)', 1, 1, 'i', 2)` -> `EF` |
| `repeat ( string text, number integer )` -> `text` | Wiederholt `string` die angegebene Anzahl von Malen. | `repeat('Pg', 4)` -> `PgPgPgPg` |
| `replace ( string text, from text, to text )` -> `text` | Ersetzt alle Vorkommen von `from` in `string` durch `to`. | `replace('abcdefabcdef', 'cd', 'XX')` -> `abXXefabXXef` |
| `reverse ( text )` -> `text` | Kehrt die Reihenfolge der Zeichen um. | `reverse('abcde')` -> `edcba` |
| `right ( string text, n integer )` -> `text` | Gibt die letzten `n` Zeichen zurück; ist `n` negativ, alle außer den ersten `|n|` Zeichen. | `right('abcde', 2)` -> `de` |
| `split_part ( string text, delimiter text, n integer )` -> `text` | Teilt `string` an Vorkommen von `delimiter` und gibt das `n`-te Feld zurück, beginnend bei 1. Ist `n` negativ, wird von rechts gezählt. | `split_part('abc~@~def~@~ghi', '~@~', 2)` -> `def`<br>`split_part('abc,def,ghi,jkl', ',', -2)` -> `ghi` |
| `starts_with ( string text, prefix text )` -> `boolean` | Gibt `true` zurück, wenn `string` mit `prefix` beginnt. | `starts_with('alphabet', 'alph')` -> `t` |
| `string_to_array ( string text, delimiter text [, null_string text ] )` -> `text[]` | Teilt `string` an `delimiter` und bildet daraus ein `text`-Array. Ist `delimiter` `NULL`, wird jedes Zeichen ein eigenes Element; ist er leer, wird die gesamte Zeichenkette ein Feld. Felder gleich `null_string` werden zu `NULL`. | `string_to_array('xx~~yy~~zz', '~~', 'yy')` -> `{xx,NULL,zz}` |
| `string_to_table ( string text, delimiter text [, null_string text ] )` -> `setof text` | Teilt `string` an `delimiter` und liefert die Felder als Zeilen vom Typ `text`. Die Regeln für `delimiter` und `null_string` entsprechen `string_to_array`. | `string_to_table('xx~^~yy~^~zz', '~^~', 'yy')` -> Zeilen `xx`, `NULL`, `zz` |
| `strpos ( string text, substring text )` -> `integer` | Gibt die erste Startposition von `substring` in `string` zurück, oder null. Entspricht `position(substring in string)`, aber mit umgekehrter Argumentreihenfolge. | `strpos('high', 'ig')` -> `2` |
| `substr ( string text, start integer [, count integer ] )` -> `text` | Extrahiert den Teilstring von `string`, beginnend beim `start`-ten Zeichen und optional `count` Zeichen lang. Entspricht `substring(string from start for count)`. | `substr('alphabet', 3)` -> `phabet`<br>`substr('alphabet', 3, 2)` -> `ph` |
| `to_ascii ( string text )` -> `text`<br>`to_ascii ( string text, encoding name )` -> `text`<br>`to_ascii ( string text, encoding integer )` -> `text` | Wandelt `string` aus einer anderen Kodierung nach ASCII um. Ohne `encoding` wird die Datenbankkodierung angenommen. Die Umwandlung entfernt hauptsächlich Akzente und unterstützt nur `LATIN1`, `LATIN2`, `LATIN9` und `WIN1250`. | `to_ascii('Karél')` -> `Karel` |
| `to_bin ( integer )` -> `text`<br>`to_bin ( bigint )` -> `text` | Wandelt die Zahl in ihre Zweierkomplement-Binärdarstellung um. | `to_bin(2147483647)` -> `1111111111111111111111111111111`<br>`to_bin(-1234)` -> `11111111111111111111101100101110` |
| `to_hex ( integer )` -> `text`<br>`to_hex ( bigint )` -> `text` | Wandelt die Zahl in ihre Zweierkomplement-Hexadezimaldarstellung um. | `to_hex(2147483647)` -> `7fffffff`<br>`to_hex(-1234)` -> `fffffb2e` |
| `to_oct ( integer )` -> `text`<br>`to_oct ( bigint )` -> `text` | Wandelt die Zahl in ihre Zweierkomplement-Oktaldarstellung um. | `to_oct(2147483647)` -> `17777777777`<br>`to_oct(-1234)` -> `37777775456` |
| `translate ( string text, from text, to text )` -> `text` | Ersetzt jedes Zeichen in `string`, das in `from` vorkommt, durch das entsprechende Zeichen aus `to`. Ist `from` länger als `to`, werden die zusätzlichen Zeichen gelöscht. | `translate('12345', '143', 'ax')` -> `a2x5` |
| `unistr ( text )` -> `text` | Wertet Unicode-Escapes im Argument aus. Zeichen können als `\XXXX`, `\+XXXXXX`, `\uXXXX` oder `\UXXXXXXXX` angegeben werden; ein Backslash wird als doppelter Backslash geschrieben. Nicht standardisierte Alternative zu Unicode-Escape-Zeichenketten. | `unistr('d\0061t\+000061')` -> `data`<br>`unistr('d\u0061t\U00000061')` -> `data` |

Die Funktionen `concat`, `concat_ws` und `format` sind variadisch. Deshalb können die zu verkettenden oder zu formatierenden Werte als Array mit dem Schlüsselwort `VARIADIC` übergeben werden (siehe Abschnitt 36.5.6). Die Array-Elemente werden behandelt, als wären sie einzelne normale Funktionsargumente. Ist das variadische Arrayargument `NULL`, geben `concat` und `concat_ws` `NULL` zurück; `format` behandelt `NULL` als leeres Array.

Siehe auch die Aggregatfunktion `string_agg` in [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen) sowie die Funktionen zur Umwandlung zwischen Zeichenketten und dem Typ `bytea` in Tabelle 9.13.

### 9.4.1. `format`

Die Funktion `format` erzeugt eine nach einer Formatzeichenkette formatierte Ausgabe, ähnlich der C-Funktion `sprintf`.

```text
format(formatstr text [, formatarg "any" [, ...] ])
```

`formatstr` ist eine Formatzeichenkette, die festlegt, wie das Ergebnis formatiert wird. Text in der Formatzeichenkette wird direkt in das Ergebnis kopiert, außer an Stellen mit Formatspezifizierern. Formatspezifizierer dienen als Platzhalter und legen fest, wie nachfolgende Funktionsargumente formatiert und in das Ergebnis eingefügt werden. Jedes Argument `formatarg` wird nach den üblichen Ausgaberegeln seines Datentyps in `text` umgewandelt und danach entsprechend dem Formatspezifizierer eingefügt.

Formatspezifizierer beginnen mit `%` und haben die Form:

```text
%[position][flags][width]type
```

Die Bestandteile sind:

- `position` (optional): Eine Zeichenkette der Form `n$`, wobei `n` der Index des auszugebenden Arguments ist. Index 1 bezeichnet das erste Argument nach `formatstr`. Ohne Position wird das nächste Argument in Reihenfolge verwendet.
- `flags` (optional): Zusätzliche Optionen für die Ausgabe. Derzeit wird nur das Minuszeichen (`-`) unterstützt; es richtet die Ausgabe linksbündig aus. Das wirkt nur, wenn auch `width` angegeben ist.
- `width` (optional): Die Mindestzahl von Zeichen für die Ausgabe. Die Ausgabe wird links oder rechts mit Leerzeichen aufgefüllt. Eine zu kleine Breite schneidet nicht ab, sondern wird ignoriert. Die Breite kann als positive Ganzzahl, als Sternchen (`*`) für das nächste Funktionsargument oder als `*n$` für das `n`-te Funktionsargument angegeben werden.
- `type` (erforderlich): Unterstützt werden `s` für einfache Zeichenketten (`NULL` wird als leere Zeichenkette behandelt), `I` für SQL-Bezeichner mit nötigem Double-Quoting (entspricht `quote_ident`) und `L` für SQL-Literale (`NULL` erscheint als unquotierte Zeichenkette `NULL`, entspricht `quote_nullable`).

Zusätzlich zu diesen Formatspezifizierern kann `%%` verwendet werden, um ein literales Prozentzeichen auszugeben.

Beispiele für grundlegende Formatumwandlungen:

```sql
SELECT format('Hello %s', 'World');
```

```text
Result: Hello World
```

```sql
SELECT format('Testing %s, %s, %s, %%', 'one', 'two', 'three');
```

```text
Result: Testing one, two, three, %
```

```sql
SELECT format('INSERT INTO %I VALUES(%L)', 'Foo bar', E'O\'Reilly');
```

```text
Result: INSERT INTO "Foo bar" VALUES('O''Reilly')
```

```sql
SELECT format('INSERT INTO %I VALUES(%L)', 'locations', 'C:\Program Files');
```

```text
Result: INSERT INTO locations VALUES('C:\Program Files')
```

Beispiele mit Breitenangaben und dem Flag `-`:

```sql
SELECT format('|%10s|', 'foo');
SELECT format('|%-10s|', 'foo');
SELECT format('|%*s|', 10, 'foo');
SELECT format('|%*s|', -10, 'foo');
SELECT format('|%-*s|', 10, 'foo');
SELECT format('|%-*s|', -10, 'foo');
```

```text
Result: |       foo|
Result: |foo       |
Result: |       foo|
Result: |foo       |
Result: |foo       |
Result: |foo       |
```

Beispiele mit Positionsfeldern:

```sql
SELECT format('Testing %3$s, %2$s, %1$s', 'one', 'two', 'three');
SELECT format('|%*2$s|', 'foo', 10, 'bar');
SELECT format('|%1$*2$s|', 'foo', 10, 'bar');
```

```text
Result: Testing three, two, one
Result: |       bar|
Result: |       foo|
```

Anders als die Standard-C-Funktion `sprintf` erlaubt PostgreSQLs Funktion `format`, Formatspezifizierer mit und ohne Positionsfelder in derselben Formatzeichenkette zu mischen. Ein Formatspezifizierer ohne Positionsfeld verwendet immer das nächste Argument nach dem zuletzt verbrauchten Argument. Außerdem müssen nicht alle Funktionsargumente in der Formatzeichenkette verwendet werden:

```sql
SELECT format('Testing %3$s, %2$s, %s', 'one', 'two', 'three');
```

```text
Result: Testing three, two, three
```

Die Formatspezifizierer `%I` und `%L` sind besonders nützlich, um dynamische SQL-Anweisungen sicher zu konstruieren. Siehe Beispiel 41.1.

## 9.5. Binäre Zeichenkettenfunktionen und -operatoren

Dieser Abschnitt beschreibt Funktionen und Operatoren zum Untersuchen und Bearbeiten binärer Zeichenketten, also von Werten des Typs `bytea`. Viele davon entsprechen in Zweck und Syntax den Text-Zeichenkettenfunktionen aus dem vorherigen Abschnitt.

SQL definiert einige Zeichenkettenfunktionen, die Schlüsselwörter statt Kommata verwenden, um Argumente zu trennen. Details stehen in Tabelle 9.11. PostgreSQL stellt außerdem Varianten dieser Funktionen mit regulärer Funktionsaufrufsyntax bereit; diese sind in Tabelle 9.12 aufgeführt.

**Tabelle 9.11. SQL-Funktionen und -Operatoren für binäre Zeichenketten**

| Funktion/Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `bytea || bytea` -> `bytea` | Verkettet die beiden binären Zeichenketten. | `E'\\x123456'::bytea || E'\\x789a00bcde'::bytea` -> `\x123456789a00bcde` |
| `bit_length ( bytea )` -> `integer` | Gibt die Anzahl der Bits in der binären Zeichenkette zurück (`8` mal `octet_length`). | `bit_length(E'\\x123456'::bytea)` -> `24` |
| `btrim ( bytes bytea, bytesremoved bytea )` -> `bytea` | Entfernt am Anfang und Ende von `bytes` die längste Zeichenkette, die nur aus Bytes besteht, die in `bytesremoved` vorkommen. | `btrim(E'\\x1234567890'::bytea, E'\\x9012'::bytea)` -> `\x345678` |
| `ltrim ( bytes bytea, bytesremoved bytea )` -> `bytea` | Entfernt am Anfang von `bytes` die längste Zeichenkette, die nur aus Bytes besteht, die in `bytesremoved` vorkommen. | `ltrim(E'\\x1234567890'::bytea, E'\\x9012'::bytea)` -> `\x34567890` |
| `octet_length ( bytea )` -> `integer` | Gibt die Anzahl der Bytes in der binären Zeichenkette zurück. | `octet_length(E'\\x123456'::bytea)` -> `3` |
| `overlay ( bytes bytea PLACING newsubstring bytea FROM start integer [ FOR count integer ] )` -> `bytea` | Ersetzt den Teilstring von `bytes`, der beim `start`-ten Byte beginnt und sich über `count` Bytes erstreckt, durch `newsubstring`. Wenn `count` weggelassen wird, wird standardmäßig die Länge von `newsubstring` verwendet. | `overlay(E'\\x1234567890'::bytea placing E'\\002\\003'::bytea from 2 for 3)` -> `\x12020390` |
| `position ( substring bytea IN bytes bytea )` -> `integer` | Gibt den ersten Startindex des angegebenen Teilstrings innerhalb von `bytes` zurück, oder null, wenn er nicht vorhanden ist. | `position(E'\\x5678'::bytea in E'\\x1234567890'::bytea)` -> `3` |
| `rtrim ( bytes bytea, bytesremoved bytea )` -> `bytea` | Entfernt am Ende von `bytes` die längste Zeichenkette, die nur aus Bytes besteht, die in `bytesremoved` vorkommen. | `rtrim(E'\\x1234567890'::bytea, E'\\x9012'::bytea)` -> `\x12345678` |
| `substring ( bytes bytea [ FROM start integer ] [ FOR count integer ] )` -> `bytea` | Extrahiert den Teilstring von `bytes`, beginnend beim `start`-ten Byte, falls angegeben, und endend nach `count` Bytes, falls angegeben. Mindestens eines von `start` und `count` muss angegeben werden. | `substring(E'\\x1234567890'::bytea from 3 for 2)` -> `\x5678` |
| `trim ( [ LEADING | TRAILING | BOTH ] bytesremoved bytea FROM bytes bytea )` -> `bytea` | Entfernt am Anfang, am Ende oder an beiden Enden (`BOTH` ist die Voreinstellung) von `bytes` die längste Zeichenkette, die nur aus Bytes besteht, die in `bytesremoved` vorkommen. | `trim(E'\\x9012'::bytea from E'\\x1234567890'::bytea)` -> `\x345678` |
| `trim ( [ LEADING | TRAILING | BOTH ] [ FROM ] bytes bytea, bytesremoved bytea )` -> `bytea` | Dies ist eine nicht standardkonforme Syntax für `trim()`. | `trim(both from E'\\x1234567890'::bytea, E'\\x9012'::bytea)` -> `\x345678` |

Weitere Funktionen zur Bearbeitung binärer Zeichenketten sind in Tabelle 9.12 aufgeführt. Einige davon werden intern verwendet, um die SQL-Standardfunktionen aus Tabelle 9.11 zu implementieren.

**Tabelle 9.12. Weitere Funktionen für binäre Zeichenketten**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `bit_count ( bytes bytea )` -> `bigint` | Gibt die Anzahl der gesetzten Bits in der binären Zeichenkette zurück; dies wird auch „Popcount“ genannt. | `bit_count(E'\\x1234567890'::bytea)` -> `15` |
| `crc32 ( bytea )` -> `bigint` | Berechnet den CRC-32-Wert der binären Zeichenkette. | `crc32('abc'::bytea)` -> `891568578` |
| `crc32c ( bytea )` -> `bigint` | Berechnet den CRC-32C-Wert der binären Zeichenkette. | `crc32c('abc'::bytea)` -> `910901175` |
| `get_bit ( bytes bytea, n bigint )` -> `integer` | Extrahiert das `n`-te Bit aus der binären Zeichenkette. | `get_bit(E'\\x1234567890'::bytea, 30)` -> `1` |
| `get_byte ( bytes bytea, n integer )` -> `integer` | Extrahiert das `n`-te Byte aus der binären Zeichenkette. | `get_byte(E'\\x1234567890'::bytea, 4)` -> `144` |
| `length ( bytea )` -> `integer` | Gibt die Anzahl der Bytes in der binären Zeichenkette zurück. | `length(E'\\x1234567890'::bytea)` -> `5` |
| `length ( bytes bytea, encoding name )` -> `integer` | Gibt die Anzahl der Zeichen in der binären Zeichenkette zurück, wobei angenommen wird, dass sie Text in der angegebenen Kodierung enthält. | `length('jose'::bytea, 'UTF8')` -> `4` |
| `md5 ( bytea )` -> `text` | Berechnet den MD5-Hash der binären Zeichenkette; das Ergebnis wird hexadezimal geschrieben. | `md5(E'Th\\000omas'::bytea)` -> `8ab2d3c9689aaf18b4958c334c82d8b1` |
| `reverse ( bytea )` -> `bytea` | Kehrt die Reihenfolge der Bytes in der binären Zeichenkette um. | `reverse(E'\\xabcd'::bytea)` -> `\xcdab` |
| `set_bit ( bytes bytea, n bigint, newvalue integer )` -> `bytea` | Setzt das `n`-te Bit in der binären Zeichenkette auf `newvalue`. | `set_bit(E'\\x1234567890'::bytea, 30, 0)` -> `\x1234563890` |
| `set_byte ( bytes bytea, n integer, newvalue integer )` -> `bytea` | Setzt das `n`-te Byte in der binären Zeichenkette auf `newvalue`. | `set_byte(E'\\x1234567890'::bytea, 4, 64)` -> `\x1234567840` |
| `sha224 ( bytea )` -> `bytea` | Berechnet den SHA-224-Hash der binären Zeichenkette. | `sha224('abc'::bytea)` -> `\x23097d223405d8228642a477bda255b32aadbce4bda0b3f7e36c9da7` |
| `sha256 ( bytea )` -> `bytea` | Berechnet den SHA-256-Hash der binären Zeichenkette. | `sha256('abc'::bytea)` -> `\xba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad` |
| `sha384 ( bytea )` -> `bytea` | Berechnet den SHA-384-Hash der binären Zeichenkette. | `sha384('abc'::bytea)` -> `\xcb00753f45a35e8bb5a03d699ac65007272c32ab0eded1631a8b605a43ff5bed8086072ba1e7cc2358baeca134c825a7` |
| `sha512 ( bytea )` -> `bytea` | Berechnet den SHA-512-Hash der binären Zeichenkette. | `sha512('abc'::bytea)` -> `\xddaf35a193617abacc417349ae20413112e6fa4e89a97ea20a9eeee64b55d39a2192992a274fc1a836ba3c23a3feebbd454d4423643ce80e2a9ac94fa54ca49f` |
| `substr ( bytes bytea, start integer [, count integer ] )` -> `bytea` | Extrahiert den Teilstring von `bytes`, beginnend beim `start`-ten Byte und, falls angegeben, über `count` Bytes. Dies entspricht `substring(bytes from start for count)`. | `substr(E'\\x1234567890'::bytea, 3, 2)` -> `\x5678` |

Die Funktionen `get_byte` und `set_byte` nummerieren das erste Byte einer binären Zeichenkette als Byte 0. Die Funktionen `get_bit` und `set_bit` nummerieren Bits innerhalb jedes Bytes von rechts; Bit 0 ist also das niederwertigste Bit des ersten Bytes, und Bit 15 ist das höchstwertige Bit des zweiten Bytes.

Aus historischen Gründen gibt die Funktion `md5` einen hexadezimal kodierten Wert des Typs `text` zurück, während die SHA-2-Funktionen den Typ `bytea` zurückgeben. Verwenden Sie die Funktionen `encode` und `decode`, um zwischen beiden Darstellungen umzuwandeln. Schreiben Sie zum Beispiel `encode(sha256('abc'), 'hex')`, um eine hexadezimal kodierte Textdarstellung zu erhalten, oder `decode(md5('abc'), 'hex')`, um einen `bytea`-Wert zu erhalten.

Funktionen zum Umwandeln von Zeichenketten zwischen verschiedenen Zeichensätzen (Kodierungen) und zum Darstellen beliebiger Binärdaten in Textform sind in Tabelle 9.13 gezeigt. Bei diesen Funktionen wird ein Argument oder Ergebnis des Typs `text` in der Standardkodierung der Datenbank ausgedrückt, während Argumente oder Ergebnisse des Typs `bytea` in einer Kodierung vorliegen, die durch ein anderes Argument benannt wird.

**Tabelle 9.13. Umwandlungsfunktionen für Text und binäre Zeichenketten**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `convert ( bytes bytea, src_encoding name, dest_encoding name )` -> `bytea` | Wandelt eine binäre Zeichenkette, die Text in der Kodierung `src_encoding` darstellt, in eine binäre Zeichenkette in der Kodierung `dest_encoding` um; verfügbare Umwandlungen siehe [Abschnitt 23.3.4](23_Lokalisierung.md#2334-verfügbare-zeichensatzumsetzungen). | `convert('text_in_utf8', 'UTF8', 'LATIN1')` -> `\x746578745f696e5f75746638` |
| `convert_from ( bytes bytea, src_encoding name )` -> `text` | Wandelt eine binäre Zeichenkette, die Text in der Kodierung `src_encoding` darstellt, in Text in der Datenbankkodierung um; verfügbare Umwandlungen siehe [Abschnitt 23.3.4](23_Lokalisierung.md#2334-verfügbare-zeichensatzumsetzungen). | `convert_from('text_in_utf8', 'UTF8')` -> `text_in_utf8` |
| `convert_to ( string text, dest_encoding name )` -> `bytea` | Wandelt eine Textzeichenkette in der Datenbankkodierung in eine binäre Zeichenkette in der Kodierung `dest_encoding` um; verfügbare Umwandlungen siehe [Abschnitt 23.3.4](23_Lokalisierung.md#2334-verfügbare-zeichensatzumsetzungen). | `convert_to('some_text', 'UTF8')` -> `\x736f6d655f74657874` |
| `encode ( bytes bytea, format text )` -> `text` | Kodiert Binärdaten in eine Textdarstellung; unterstützte Werte für `format` sind `base64`, `escape` und `hex`. | `encode(E'123\\000\\001', 'base64')` -> `MTIzAAE=` |
| `decode ( string text, format text )` -> `bytea` | Dekodiert Binärdaten aus einer Textdarstellung; die unterstützten Werte für `format` sind dieselben wie bei `encode`. | `decode('MTIzAAE=', 'base64')` -> `\x3132330001` |

Die Funktionen `encode` und `decode` unterstützen die folgenden Textformate:

`base64`

Das Format `base64` entspricht [RFC 2045 Abschnitt 6.8](https://datatracker.ietf.org/doc/html/rfc2045#section-6.8). Gemäß RFC werden kodierte Zeilen bei 76 Zeichen umbrochen. Statt des MIME-Zeilenendezeichens CRLF wird jedoch nur ein Zeilenumbruch verwendet. Die Funktion `decode` ignoriert Wagenrücklauf-, Zeilenumbruch-, Leerzeichen- und Tabulatorzeichen. Ansonsten wird ein Fehler ausgelöst, wenn `decode` ungültige `base64`-Daten erhält, auch bei falscher abschließender Auffüllung.

`escape`

Das Format `escape` wandelt Nullbytes und Bytes mit gesetztem höchstem Bit in oktale Escape-Sequenzen (`\nnn`) um und verdoppelt Backslashes. Andere Bytewerte werden wörtlich dargestellt. Die Funktion `decode` löst einen Fehler aus, wenn auf einen Backslash weder ein zweiter Backslash noch drei oktale Ziffern folgen; andere Bytewerte akzeptiert sie unverändert.

`hex`

Das Format `hex` stellt je 4 Datenbits als eine hexadezimale Ziffer von `0` bis `f` dar, wobei die höherwertige Ziffer jedes Bytes zuerst geschrieben wird. Die Funktion `encode` gibt die Hexziffern `a` bis `f` in Kleinbuchstaben aus. Da die kleinste Dateneinheit 8 Bits umfasst, gibt `encode` immer eine gerade Anzahl von Zeichen zurück. Die Funktion `decode` akzeptiert die Zeichen `a` bis `f` sowohl in Groß- als auch in Kleinbuchstaben. Ein Fehler wird ausgelöst, wenn `decode` ungültige Hexdaten erhält, auch bei einer ungeraden Anzahl von Zeichen.

Zusätzlich ist es möglich, ganzzahlige Werte in den Typ `bytea` und zurück zu casten. Das Casten einer Ganzzahl nach `bytea` erzeugt je nach Breite des Ganzzahltyps 2, 4 oder 8 Bytes. Das Ergebnis ist die Zweierkomplementdarstellung der Ganzzahl, mit dem höchstwertigen Byte zuerst. Einige Beispiele:

```text
1234::smallint::bytea                             \x04d2
cast(1234 as bytea)                               \x000004d2
cast(-1234 as bytea)                              \xfffffb2e
'\x8000'::bytea::smallint                         -32768
'\x8000'::bytea::integer                          32768
```

Das Casten eines `bytea`-Werts in eine Ganzzahl löst einen Fehler aus, wenn die Länge des `bytea`-Werts die Breite des Ganzzahltyps überschreitet.

Siehe auch die Aggregatfunktion `string_agg` in [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen) und die Large-Object-Funktionen in [Abschnitt 33.4](33_Große_Objekte.md#334-serverseitige-funktionen).

## 9.6. Bit-String-Funktionen und -operatoren

Dieser Abschnitt beschreibt Funktionen und Operatoren zum Untersuchen und Bearbeiten von Bit-Strings, also von Werten der Typen `bit` und `bit varying`. Obwohl in diesen Tabellen nur der Typ `bit` genannt wird, können Werte des Typs `bit varying` austauschbar verwendet werden. Bit-Strings unterstützen die üblichen Vergleichsoperatoren aus Tabelle 9.1 sowie die in Tabelle 9.14 gezeigten Operatoren.

**Tabelle 9.14. Bit-String-Operatoren**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `bit || bit` -> `bit` | Verkettung | `B'10001' || B'011'` -> `10001011` |
| `bit & bit` -> `bit` | Bitweises UND; die Eingaben müssen gleich lang sein. | `B'10001' & B'01101'` -> `00001` |
| `bit | bit` -> `bit` | Bitweises ODER; die Eingaben müssen gleich lang sein. | `B'10001' | B'01101'` -> `11101` |
| `bit # bit` -> `bit` | Bitweises exklusives ODER; die Eingaben müssen gleich lang sein. | `B'10001' # B'01101'` -> `11100` |
| `~ bit` -> `bit` | Bitweises NICHT | `~ B'10001'` -> `01110` |
| `bit << integer` -> `bit` | Bitweise Verschiebung nach links; die Länge der Zeichenkette bleibt erhalten. | `B'10001' << 3` -> `01000` |
| `bit >> integer` -> `bit` | Bitweise Verschiebung nach rechts; die Länge der Zeichenkette bleibt erhalten. | `B'10001' >> 2` -> `00100` |

Einige der für binäre Zeichenketten verfügbaren Funktionen sind auch für Bit-Strings verfügbar, wie Tabelle 9.15 zeigt.

**Tabelle 9.15. Bit-String-Funktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `bit_count ( bit )` -> `bigint` | Gibt die Anzahl der gesetzten Bits im Bit-String zurück; dies wird auch „Popcount“ genannt. | `bit_count(B'10111')` -> `4` |
| `bit_length ( bit )` -> `integer` | Gibt die Anzahl der Bits im Bit-String zurück. | `bit_length(B'10111')` -> `5` |
| `length ( bit )` -> `integer` | Gibt die Anzahl der Bits im Bit-String zurück. | `length(B'10111')` -> `5` |
| `octet_length ( bit )` -> `integer` | Gibt die Anzahl der Bytes im Bit-String zurück. | `octet_length(B'1011111011')` -> `2` |
| `overlay ( bits bit PLACING newsubstring bit FROM start integer [ FOR count integer ] )` -> `bit` | Ersetzt den Teilstring von `bits`, der beim `start`-ten Bit beginnt und sich über `count` Bits erstreckt, durch `newsubstring`. Wenn `count` weggelassen wird, wird standardmäßig die Länge von `newsubstring` verwendet. | `overlay(B'01010101010101010' placing B'11111' from 2 for 3)` -> `0111110101010101010` |
| `position ( substring bit IN bits bit )` -> `integer` | Gibt den ersten Startindex des angegebenen Teilstrings innerhalb von `bits` zurück, oder null, wenn er nicht vorhanden ist. | `position(B'010' in B'000001101011')` -> `8` |
| `substring ( bits bit [ FROM start integer ] [ FOR count integer ] )` -> `bit` | Extrahiert den Teilstring von `bits`, beginnend beim `start`-ten Bit, falls angegeben, und endend nach `count` Bits, falls angegeben. Mindestens eines von `start` und `count` muss angegeben werden. | `substring(B'110010111111' from 3 for 2)` -> `00` |
| `get_bit ( bits bit, n integer )` -> `integer` | Extrahiert das `n`-te Bit aus dem Bit-String; das erste, linke Bit ist Bit 0. | `get_bit(B'101010101010101010', 6)` -> `1` |
| `set_bit ( bits bit, n integer, newvalue integer )` -> `bit` | Setzt das `n`-te Bit im Bit-String auf `newvalue`; das erste, linke Bit ist Bit 0. | `set_bit(B'101010101010101010', 6, 0)` -> `101010001010101010` |

Zusätzlich ist es möglich, ganzzahlige Werte in den Typ `bit` und zurück zu casten. Das Casten einer Ganzzahl nach `bit(n)` kopiert die rechten `n` Bits. Wird eine Ganzzahl in einen Bit-String gecastet, dessen Breite größer ist als die der Ganzzahl selbst, wird links mit dem Vorzeichen erweitert. Einige Beispiele:

```text
44::bit(10)                                       0000101100
44::bit(3)                                        100
cast(-44 as bit(12))                              111111010100
'1110'::bit(4)::integer                           14
```

Beachten Sie, dass ein Cast nur nach `bit` einem Cast nach `bit(1)` entspricht und daher nur das niederwertigste Bit der Ganzzahl liefert.

## 9.7. Mustervergleich

PostgreSQL stellt drei verschiedene Ansätze für den Mustervergleich bereit: den traditionellen SQL-Operator `LIKE`, den neueren Operator `SIMILAR TO` (eingeführt in SQL:1999) und reguläre Ausdrücke im POSIX-Stil. Neben den grundlegenden Operatoren der Art „passt diese Zeichenkette zu diesem Muster?“ gibt es Funktionen, um passende Teilstrings zu extrahieren oder zu ersetzen und um eine Zeichenkette an passenden Stellen aufzuteilen.

> Tipp: Wenn Sie Anforderungen an den Mustervergleich haben, die darüber hinausgehen, sollten Sie erwägen, eine benutzerdefinierte Funktion in Perl oder Tcl zu schreiben.

> Achtung: Die meisten Suchen mit regulären Ausdrücken können sehr schnell ausgeführt werden, aber reguläre Ausdrücke lassen sich so konstruieren, dass ihre Verarbeitung beliebig viel Zeit und Speicher benötigt. Seien Sie vorsichtig, wenn Sie Suchmuster für reguläre Ausdrücke aus nicht vertrauenswürdigen Quellen akzeptieren. Wenn Sie das tun müssen, ist es ratsam, ein Statement-Timeout zu setzen.
>
> Suchen mit `SIMILAR TO`-Mustern bergen dieselben Sicherheitsrisiken, da `SIMILAR TO` viele der Fähigkeiten regulärer Ausdrücke im POSIX-Stil bereitstellt.
>
> `LIKE`-Suchen sind wesentlich einfacher als die beiden anderen Optionen und daher sicherer, wenn die Muster möglicherweise aus feindlichen Quellen stammen.

`SIMILAR TO` und reguläre Ausdrücke im POSIX-Stil unterstützen keine nichtdeterministischen Collations. Verwenden Sie bei Bedarf `LIKE` oder wenden Sie eine andere Collation auf den Ausdruck an, um diese Einschränkung zu umgehen.

### 9.7.1. `LIKE`

```sql
string LIKE pattern [ESCAPE escape-character]
string NOT LIKE pattern [ESCAPE escape-character]
```

Der Ausdruck `LIKE` gibt wahr zurück, wenn die Zeichenkette zum angegebenen Muster passt. Wie erwartet gibt `NOT LIKE` falsch zurück, wenn `LIKE` wahr zurückgibt, und umgekehrt. Ein äquivalenter Ausdruck ist `NOT (string LIKE pattern)`.

Wenn `pattern` weder Prozentzeichen noch Unterstriche enthält, steht das Muster nur für die Zeichenkette selbst; in diesem Fall verhält sich `LIKE` wie der Gleichheitsoperator. Ein Unterstrich (`_`) in `pattern` steht für ein beliebiges einzelnes Zeichen; ein Prozentzeichen (`%`) passt auf jede Folge von null oder mehr Zeichen.

Einige Beispiele:

```text
'abc' LIKE 'abc'               true
'abc' LIKE 'a%'                true
'abc' LIKE '_b_'               true
'abc' LIKE 'c'                 false
```

Der `LIKE`-Mustervergleich unterstützt nichtdeterministische Collations (siehe [Abschnitt 23.2.2.4](23_Lokalisierung.md#23224-nichtdeterministische-collations)), etwa Collations ohne Beachtung der Groß-/Kleinschreibung oder Collations, die beispielsweise Satzzeichen ignorieren. Mit einer Collation ohne Beachtung der Groß-/Kleinschreibung könnte man also Folgendes haben:

```text
'AbC' LIKE 'abc' COLLATE case_insensitive                             true
'AbC' LIKE 'a%' COLLATE case_insensitive                              true
```

Bei Collations, die bestimmte Zeichen ignorieren oder allgemein Zeichenketten unterschiedlicher Länge als gleich betrachten, kann die Semantik etwas komplizierter werden. Betrachten Sie diese Beispiele:

```text
'.foo.' LIKE 'foo' COLLATE ign_punct                          true
'.foo.' LIKE 'f_o' COLLATE ign_punct                          true
'.foo.' LIKE '_oo' COLLATE ign_punct                          false
```

Der Abgleich funktioniert so, dass das Muster in Folgen aus Wildcards und Nicht-Wildcard-Zeichenketten zerlegt wird; Wildcards sind `_` und `%`. Zum Beispiel wird das Muster `f_o` in `f`, `_`, `o` zerlegt, das Muster `_oo` in `_`, `oo`. Die Eingabezeichenkette passt zum Muster, wenn sie so zerlegt werden kann, dass die Wildcards jeweils ein Zeichen beziehungsweise beliebig viele Zeichen treffen und die Nicht-Wildcard-Teile unter der anwendbaren Collation gleich sind. Daher ist zum Beispiel `'.foo.' LIKE 'f_o' COLLATE ign_punct` wahr, weil man `.foo.` in `.f`, `o`, `o.` zerlegen kann und dann `'.f' = 'f' COLLATE ign_punct`, `o` zur Wildcard `_` passt und `'o.' = 'o' COLLATE ign_punct` gilt. Dagegen ist `'.foo.' LIKE '_oo' COLLATE ign_punct` falsch, weil `.foo.` nicht so zerlegt werden kann, dass das erste Zeichen ein beliebiges Zeichen ist und der Rest der Zeichenkette gleich `oo` vergleicht. Beachten Sie, dass die Ein-Zeichen-Wildcard unabhängig von der Collation immer genau ein Zeichen trifft. In diesem Beispiel würde `_` also `.` treffen, aber der Rest der Eingabezeichenkette passt dann nicht mehr zum Rest des Musters.

Der `LIKE`-Mustervergleich deckt immer die gesamte Zeichenkette ab. Soll eine Folge irgendwo innerhalb einer Zeichenkette gefunden werden, muss das Muster daher mit einem Prozentzeichen beginnen und enden.

Um einen wörtlichen Unterstrich oder ein wörtliches Prozentzeichen zu treffen, ohne andere Zeichen zu erfassen, muss dem jeweiligen Zeichen in `pattern` das Escape-Zeichen vorangestellt werden. Das Standard-Escape-Zeichen ist der Backslash, aber mit der `ESCAPE`-Klausel kann ein anderes gewählt werden. Um das Escape-Zeichen selbst zu treffen, schreiben Sie zwei Escape-Zeichen.

> Hinweis: Wenn `standard_conforming_strings` ausgeschaltet ist, müssen alle Backslashes, die Sie in literalen Zeichenkettenkonstanten schreiben, verdoppelt werden. Weitere Informationen finden Sie in [Abschnitt 4.1.2.1](04_SQL_Syntax.md#4121-zeichenkettenkonstanten).

Es ist auch möglich, kein Escape-Zeichen auszuwählen, indem `ESCAPE ''` geschrieben wird. Das deaktiviert den Escape-Mechanismus effektiv und macht es unmöglich, die besondere Bedeutung von Unterstrich und Prozentzeichen im Muster auszuschalten.

Nach dem SQL-Standard bedeutet das Weglassen von `ESCAPE`, dass es kein Escape-Zeichen gibt, statt standardmäßig einen Backslash zu verwenden; außerdem ist ein `ESCAPE`-Wert der Länge null nicht erlaubt. Das Verhalten von PostgreSQL ist in dieser Hinsicht daher leicht nicht standardkonform.

Das Schlüsselwort `ILIKE` kann anstelle von `LIKE` verwendet werden, um den Abgleich gemäß der aktiven Locale ohne Beachtung der Groß-/Kleinschreibung durchzuführen. Nichtdeterministische Collations werden dadurch jedoch nicht unterstützt. Dies gehört nicht zum SQL-Standard, sondern ist eine PostgreSQL-Erweiterung.

Der Operator `~~` ist äquivalent zu `LIKE`, und `~~*` entspricht `ILIKE`. Außerdem gibt es die Operatoren `!~~` und `!~~*`, die `NOT LIKE` beziehungsweise `NOT ILIKE` darstellen. Alle diese Operatoren sind PostgreSQL-spezifisch. Sie können diese Operatornamen in `EXPLAIN`-Ausgaben und ähnlichen Stellen sehen, weil der Parser `LIKE` und Verwandtes tatsächlich in diese Operatoren übersetzt.

Die Ausdrücke `LIKE`, `ILIKE`, `NOT LIKE` und `NOT ILIKE` werden in der PostgreSQL-Syntax im Allgemeinen als Operatoren behandelt; sie können zum Beispiel in Konstrukten der Form `expression operator ANY (subquery)` verwendet werden, wobei dort keine `ESCAPE`-Klausel enthalten sein kann. In einigen seltenen Fällen kann es nötig sein, stattdessen die zugrunde liegenden Operatornamen zu verwenden.

Siehe auch den Starts-with-Operator `^@` und die entsprechende Funktion `starts_with()`, die nützlich sind, wenn nur der Anfang einer Zeichenkette abgeglichen werden soll.

### 9.7.2. Reguläre Ausdrücke mit `SIMILAR TO`

```sql
string SIMILAR TO pattern [ESCAPE escape-character]
string NOT SIMILAR TO pattern [ESCAPE escape-character]
```

Der Operator `SIMILAR TO` gibt wahr oder falsch zurück, je nachdem, ob sein Muster zur angegebenen Zeichenkette passt. Er ähnelt `LIKE`, interpretiert das Muster aber nach der Definition regulärer Ausdrücke des SQL-Standards. SQL-reguläre Ausdrücke sind eine eigentümliche Mischung aus `LIKE`-Notation und üblicher POSIX-Notation für reguläre Ausdrücke.

Wie `LIKE` ist `SIMILAR TO` nur erfolgreich, wenn sein Muster zur gesamten Zeichenkette passt; das unterscheidet sich vom üblichen Verhalten regulärer Ausdrücke, bei dem ein Muster auf einen beliebigen Teil der Zeichenkette passen kann. Ebenfalls wie `LIKE` verwendet `SIMILAR TO` `_` und `%` als Wildcard-Zeichen für ein beliebiges einzelnes Zeichen beziehungsweise eine beliebige Zeichenkette. Diese sind vergleichbar mit `.` und `.*` in regulären POSIX-Ausdrücken.

Zusätzlich zu diesen von `LIKE` übernommenen Fähigkeiten unterstützt `SIMILAR TO` die folgenden Metazeichen für den Mustervergleich, die von regulären POSIX-Ausdrücken übernommen sind:

- `|` bezeichnet eine Alternative, also eine von zwei Möglichkeiten.
- `*` bezeichnet die Wiederholung des vorherigen Elements null- oder mehrmals.
- `+` bezeichnet die Wiederholung des vorherigen Elements ein- oder mehrmals.
- `?` bezeichnet die Wiederholung des vorherigen Elements null- oder einmal.
- `{m}` bezeichnet die Wiederholung des vorherigen Elements genau `m`-mal.
- `{m,}` bezeichnet die Wiederholung des vorherigen Elements `m`- oder mehrmals.
- `{m,n}` bezeichnet die Wiederholung des vorherigen Elements mindestens `m`- und höchstens `n`-mal.
- Klammern `()` können verwendet werden, um Elemente zu einem einzelnen logischen Element zu gruppieren.
- Ein Klammerausdruck `[...]` gibt eine Zeichenklasse an, genau wie in regulären POSIX-Ausdrücken.

Beachten Sie, dass der Punkt (`.`) für `SIMILAR TO` kein Metazeichen ist.

Wie bei `LIKE` hebt ein Backslash die besondere Bedeutung jedes dieser Metazeichen auf. Mit `ESCAPE` kann ein anderes Escape-Zeichen angegeben werden, oder die Escape-Fähigkeit kann durch `ESCAPE ''` deaktiviert werden.

Nach dem SQL-Standard bedeutet das Weglassen von `ESCAPE`, dass es kein Escape-Zeichen gibt, statt standardmäßig einen Backslash zu verwenden; außerdem ist ein `ESCAPE`-Wert der Länge null nicht erlaubt. Das Verhalten von PostgreSQL ist in dieser Hinsicht daher leicht nicht standardkonform.

Eine weitere nicht standardkonforme Erweiterung besteht darin, dass ein Buchstabe oder eine Ziffer nach dem Escape-Zeichen Zugriff auf die Escape-Sequenzen bietet, die für reguläre POSIX-Ausdrücke definiert sind; siehe Tabelle 9.20, Tabelle 9.21 und Tabelle 9.22 weiter unten.

Einige Beispiele:

```text
'abc' SIMILAR TO 'abc'                            true
'abc' SIMILAR TO 'a'                              false
'abc' SIMILAR TO '%(b|d)%'                        true
'abc' SIMILAR TO '(b|c)%'                         false
'-abc-' SIMILAR TO '%\mabc\M%'                    true
'xabcy' SIMILAR TO '%\mabc\M%'                    false
```

Die Funktion `substring` mit drei Parametern extrahiert einen Teilstring, der zu einem SQL-Muster für reguläre Ausdrücke passt. Die Funktion kann in der SQL-Standardsyntax geschrieben werden:

```sql
substring(string similar pattern escape escape-character)
```

oder mit der inzwischen veralteten SQL:1999-Syntax:

```sql
substring(string from pattern for escape-character)
```

oder als gewöhnliche Funktion mit drei Argumenten:

```sql
substring(string, pattern, escape-character)
```

Wie bei `SIMILAR TO` muss das angegebene Muster zur gesamten Datenzeichenkette passen, sonst schlägt die Funktion fehl und gibt `null` zurück. Um anzugeben, welcher Teil des Musters für den passenden Daten-Teilstring von Interesse ist, sollte das Muster zwei Vorkommen des Escape-Zeichens gefolgt von einem doppelten Anführungszeichen (`"`) enthalten. Der Text, der zu dem Musterteil zwischen diesen Trennern passt, wird bei erfolgreichem Abgleich zurückgegeben.

Die Escape-Doppelquote-Trenner teilen das Muster von `substring` tatsächlich in drei unabhängige reguläre Ausdrücke; ein senkrechter Strich (`|`) in einem der drei Abschnitte wirkt zum Beispiel nur in diesem Abschnitt. Außerdem sind der erste und dritte dieser regulären Ausdrücke so definiert, dass sie bei Mehrdeutigkeit darüber, welcher Teil der Datenzeichenkette zu welchem Muster passt, die kleinstmögliche Textmenge treffen, nicht die größte. In POSIX-Terminologie werden der erste und dritte reguläre Ausdruck also auf nicht-gieriges Verhalten gezwungen.

Als Erweiterung zum SQL-Standard erlaubt PostgreSQL, dass es nur einen Escape-Doppelquote-Trenner gibt; in diesem Fall wird der dritte reguläre Ausdruck als leer angenommen. Gibt es keine Trenner, werden der erste und der dritte reguläre Ausdruck als leer angenommen.

Einige Beispiele, wobei `#"` die zurückzugebende Zeichenkette begrenzt:

```text
substring('foobar' similar '%#"o_b#"%' escape '#')                                  oob
substring('foobar' similar '#"o_b#"%' escape '#')                                   NULL
```

### 9.7.3. Reguläre POSIX-Ausdrücke

Tabelle 9.16 listet die verfügbaren Operatoren für den Mustervergleich mit regulären POSIX-Ausdrücken auf.

**Tabelle 9.16. Match-Operatoren für reguläre Ausdrücke**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `text ~ text` -> `boolean` | Die Zeichenkette passt unter Beachtung der Groß-/Kleinschreibung zum regulären Ausdruck. | `'thomas' ~ 't.*ma'` -> `t` |
| `text ~* text` -> `boolean` | Die Zeichenkette passt ohne Beachtung der Groß-/Kleinschreibung zum regulären Ausdruck. | `'thomas' ~* 'T.*ma'` -> `t` |
| `text !~ text` -> `boolean` | Die Zeichenkette passt unter Beachtung der Groß-/Kleinschreibung nicht zum regulären Ausdruck. | `'thomas' !~ 't.*max'` -> `t` |
| `text !~* text` -> `boolean` | Die Zeichenkette passt ohne Beachtung der Groß-/Kleinschreibung nicht zum regulären Ausdruck. | `'thomas' !~* 'T.*ma'` -> `f` |

Reguläre POSIX-Ausdrücke bieten mächtigere Mittel zum Mustervergleich als die Operatoren `LIKE` und `SIMILAR TO`. Viele Unix-Werkzeuge wie `egrep`, `sed` oder `awk` verwenden eine Sprache für den Mustervergleich, die der hier beschriebenen ähnelt.

Ein regulärer Ausdruck ist eine Zeichenfolge, die eine abgekürzte Definition einer Menge von Zeichenketten, einer regulären Menge, darstellt. Eine Zeichenkette passt zu einem regulären Ausdruck, wenn sie Element der durch den regulären Ausdruck beschriebenen regulären Menge ist. Wie bei `LIKE` passen Musterzeichen exakt zu Zeichen der Zeichenkette, sofern sie keine Sonderzeichen in der Sprache der regulären Ausdrücke sind. Reguläre Ausdrücke verwenden jedoch andere Sonderzeichen als `LIKE`. Anders als `LIKE`-Muster darf ein regulärer Ausdruck irgendwo innerhalb einer Zeichenkette passen, sofern er nicht ausdrücklich am Anfang oder Ende der Zeichenkette verankert ist.

Einige Beispiele:

```text
'abcd' ~ 'bc'        true
'abcd' ~ 'a.c'       true  -- Punkt passt auf jedes Zeichen
'abcd' ~ 'a.*d'      true  -- * wiederholt das vorherige Musterelement
'abcd' ~ '(b|x)'     true  -- | bedeutet ODER, Klammern gruppieren
'abcd' ~ '^a'        true  -- ^ verankert am Anfang der Zeichenkette
'abcd' ~ '^(b|c)'    false -- würde ohne Verankerung passen
```

Die POSIX-Mustersprache wird unten wesentlich ausführlicher beschrieben.

Die Funktion `substring` mit zwei Parametern, `substring(string from pattern)`, extrahiert einen Teilstring, der zu einem Muster eines regulären POSIX-Ausdrucks passt. Sie gibt `null` zurück, wenn es keinen Treffer gibt; andernfalls gibt sie den ersten Textteil zurück, der zum Muster gepasst hat. Wenn das Muster jedoch Klammern enthält, wird der Textteil zurückgegeben, der zum ersten geklammerten Teilausdruck passt, also zu dem Teilausdruck, dessen öffnende Klammer zuerst vorkommt. Sie können Klammern um den gesamten Ausdruck setzen, wenn Sie innerhalb des Musters Klammern verwenden möchten, ohne diese Ausnahme auszulösen. Wenn Sie im Muster Klammern vor dem Teilausdruck benötigen, den Sie extrahieren möchten, beachten Sie die unten beschriebenen nicht erfassenden Klammern.

Einige Beispiele:

```text
substring('foobar' from 'o.b')                         oob
substring('foobar' from 'o(.)b')                       o
```

Die Funktion `regexp_count` zählt, an wie vielen Stellen ein Muster eines regulären POSIX-Ausdrucks zu einer Zeichenkette passt. Sie hat die Syntax `regexp_count(string, pattern [, start [, flags ]])`. `pattern` wird in `string` gesucht, normalerweise vom Anfang der Zeichenkette an; wenn der Parameter `start` angegeben ist, beginnt die Suche jedoch bei diesem Zeichenindex. Der Parameter `flags` ist eine optionale Textzeichenkette mit null oder mehr einbuchstabigen Flags, die das Verhalten der Funktion ändern. Zum Beispiel bewirkt `i` in `flags` einen Abgleich ohne Beachtung der Groß-/Kleinschreibung. Die unterstützten Flags sind in Tabelle 9.24 beschrieben.

Einige Beispiele:

```text
regexp_count('ABCABCAXYaxy', 'A.')                                   3
regexp_count('ABCABCAXYaxy', 'A.', 1, 'i')                           4
```

Die Funktion `regexp_instr` gibt die Start- oder Endposition des `N`-ten Treffers eines Musters eines regulären POSIX-Ausdrucks in einer Zeichenkette zurück, oder null, wenn es keinen solchen Treffer gibt. Sie hat die Syntax `regexp_instr(string, pattern [, start [, N [, endoption [, flags [, subexpr ]]]]])`. `pattern` wird in `string` gesucht, normalerweise vom Anfang der Zeichenkette an; wenn der Parameter `start` angegeben ist, beginnt die Suche jedoch bei diesem Zeichenindex. Wenn `N` angegeben ist, wird der `N`-te Treffer des Musters gefunden, andernfalls der erste Treffer. Wenn der Parameter `endoption` weggelassen oder als null angegeben wird, gibt die Funktion die Position des ersten Zeichens des Treffers zurück. Andernfalls muss `endoption` eins sein, und die Funktion gibt die Position des Zeichens nach dem Treffer zurück. Der Parameter `flags` ist eine optionale Textzeichenkette mit null oder mehr einbuchstabigen Flags, die das Verhalten der Funktion ändern. Die unterstützten Flags sind in Tabelle 9.24 beschrieben. Bei einem Muster mit geklammerten Teilausdrücken ist `subexpr` eine Ganzzahl, die angibt, welcher Teilausdruck von Interesse ist; das Ergebnis identifiziert die Position des Teilstrings, der zu diesem Teilausdruck passt. Teilausdrücke werden in der Reihenfolge ihrer öffnenden Klammern nummeriert. Wenn `subexpr` weggelassen oder null ist, identifiziert das Ergebnis unabhängig von geklammerten Teilausdrücken die Position des gesamten Treffers.

Einige Beispiele:

```text
regexp_instr('number of your street, town zip, FR', '[^,]+', 1, 2)        24
regexp_instr(string=>'ABCDEFGHI', pattern=>'(c..)(...)', start=>1,
             "N"=>1, endoption=>0, flags=>'i', subexpr=>2)              6
```

Die Funktion `regexp_like` prüft, ob ein Muster eines regulären POSIX-Ausdrucks irgendwo innerhalb einer Zeichenkette passt, und gibt entsprechend den booleschen Wert `true` oder `false` zurück. Sie hat die Syntax `regexp_like(string, pattern [, flags ])`. Der Parameter `flags` ist eine optionale Textzeichenkette mit null oder mehr einbuchstabigen Flags, die das Verhalten der Funktion ändern. Die unterstützten Flags sind in Tabelle 9.24 beschrieben. Ohne angegebene Flags liefert diese Funktion dieselben Ergebnisse wie der Operator `~`. Wenn nur das Flag `i` angegeben wird, liefert sie dieselben Ergebnisse wie der Operator `~*`.

Einige Beispiele:

```text
regexp_like('Hello World', 'world')                               false
regexp_like('Hello World', 'world', 'i')                          true
```

Die Funktion `regexp_match` gibt ein Textarray mit den passenden Teilstrings innerhalb des ersten Treffers eines Musters eines regulären POSIX-Ausdrucks in einer Zeichenkette zurück. Sie hat die Syntax `regexp_match(string, pattern [, flags ])`. Wenn es keinen Treffer gibt, ist das Ergebnis `NULL`. Wenn ein Treffer gefunden wird und das Muster keine geklammerten Teilausdrücke enthält, ist das Ergebnis ein einelementiges Textarray mit dem Teilstring, der zum gesamten Muster passt. Wenn ein Treffer gefunden wird und das Muster geklammerte Teilausdrücke enthält, ist das Ergebnis ein Textarray, dessen `n`-tes Element der Teilstring ist, der zum `n`-ten geklammerten Teilausdruck des Musters passt; nicht erfassende Klammern werden dabei nicht gezählt, Details siehe unten. Der Parameter `flags` ist eine optionale Textzeichenkette mit null oder mehr einbuchstabigen Flags, die das Verhalten der Funktion ändern. Die unterstützten Flags sind in Tabelle 9.24 beschrieben.

Einige Beispiele:

```sql
SELECT regexp_match('foobarbequebaz', 'bar.*que');
```

```text
 regexp_match
--------------
 {barbeque}
(1 row)
```

```sql
SELECT regexp_match('foobarbequebaz', '(bar)(beque)');
```

```text
 regexp_match
--------------
 {bar,beque}
(1 row)
```

> Tipp: Im häufigen Fall, in dem Sie nur den gesamten passenden Teilstring oder `NULL` bei keinem Treffer möchten, ist `regexp_substr()` die beste Lösung. `regexp_substr()` existiert jedoch erst ab PostgreSQL-Version 15. In älteren Versionen können Sie das erste Element des Ergebnisses von `regexp_match()` extrahieren, zum Beispiel:
>
> ```sql
> SELECT (regexp_match('foobarbequebaz', 'bar.*que'))[1];
> ```
>
> ```text
>  regexp_match
> --------------
>  barbeque
> (1 row)
> ```

Die Funktion `regexp_matches` gibt eine Menge von Textarrays mit passenden Teilstrings innerhalb von Treffern eines Musters eines regulären POSIX-Ausdrucks in einer Zeichenkette zurück. Sie hat dieselbe Syntax wie `regexp_match`. Diese Funktion gibt keine Zeilen zurück, wenn es keinen Treffer gibt, eine Zeile, wenn es einen Treffer gibt und das Flag `g` nicht angegeben ist, oder `N` Zeilen, wenn es `N` Treffer gibt und das Flag `g` angegeben ist. Jede zurückgegebene Zeile ist ein Textarray, das wie oben für `regexp_match` beschrieben den gesamten passenden Teilstring oder die zu geklammerten Teilausdrücken des Musters passenden Teilstrings enthält. `regexp_matches` akzeptiert alle in Tabelle 9.24 gezeigten Flags sowie zusätzlich das Flag `g`, das bewirkt, dass alle Treffer zurückgegeben werden, nicht nur der erste.

Einige Beispiele:

```sql
SELECT regexp_matches('foo', 'not there');
```

```text
 regexp_matches
----------------
(0 rows)
```

```sql
SELECT regexp_matches('foobarbequebazilbarfbonk', '(b[^b]+)(b[^b]+)', 'g');
```

```text
 regexp_matches
----------------
 {bar,beque}
 {bazil,barf}
(2 rows)
```

> Tipp: In den meisten Fällen sollte `regexp_matches()` mit dem Flag `g` verwendet werden, denn wenn Sie nur den ersten Treffer benötigen, ist `regexp_match()` einfacher und effizienter. `regexp_match()` existiert jedoch erst ab PostgreSQL-Version 10. In älteren Versionen ist ein gängiger Trick, einen Aufruf von `regexp_matches()` in eine Unterabfrage zu setzen, zum Beispiel:
>
> ```sql
> SELECT col1, (SELECT regexp_matches(col2, '(bar)(beque)'))
> FROM tab;
> ```
>
> Dies erzeugt bei einem Treffer ein Textarray oder andernfalls `NULL`, also dasselbe, was `regexp_match()` tun würde. Ohne die Unterabfrage würde diese Abfrage für Tabellenzeilen ohne Treffer überhaupt keine Ausgabe erzeugen, was üblicherweise nicht das gewünschte Verhalten ist.

Die Funktion `regexp_replace` ersetzt Teilstrings, die zu Mustern regulärer POSIX-Ausdrücke passen, durch neuen Text. Sie hat die Syntax `regexp_replace(string, pattern, replacement [, flags ])` oder `regexp_replace(string, pattern, replacement, start [, N [, flags ]])`. Die Quellzeichenkette wird unverändert zurückgegeben, wenn es keinen Treffer zum Muster gibt. Wenn es einen Treffer gibt, wird die Zeichenkette zurückgegeben, wobei die Ersatzzeichenkette den passenden Teilstring ersetzt. Die Ersatzzeichenkette kann `\n` enthalten, wobei `n` von 1 bis 9 reicht, um anzugeben, dass der Quellteilstring eingefügt werden soll, der zum `n`-ten geklammerten Teilausdruck des Musters passt; außerdem kann sie `\&` enthalten, um anzugeben, dass der Teilstring eingefügt werden soll, der zum gesamten Muster passt. Schreiben Sie `\\`, wenn Sie einen wörtlichen Backslash in den Ersatztext setzen müssen. `pattern` wird in `string` gesucht, normalerweise vom Anfang der Zeichenkette an; wenn der Parameter `start` angegeben ist, beginnt die Suche jedoch bei diesem Zeichenindex. Standardmäßig wird nur der erste Treffer des Musters ersetzt. Wenn `N` angegeben und größer als null ist, wird der `N`-te Treffer des Musters ersetzt. Wenn das Flag `g` angegeben ist oder wenn `N` angegeben und null ist, werden alle Treffer an oder nach der Startposition ersetzt. Das Flag `g` wird ignoriert, wenn `N` angegeben ist. Der Parameter `flags` ist eine optionale Textzeichenkette mit null oder mehr einbuchstabigen Flags, die das Verhalten der Funktion ändern. Die unterstützten Flags, außer `g`, sind in Tabelle 9.24 beschrieben.

Einige Beispiele:

```text
regexp_replace('foobarbaz', 'b..', 'X')                                  fooXbaz
regexp_replace('foobarbaz', 'b..', 'X', 'g')                             fooXX
regexp_replace('foobarbaz', 'b(..)', 'X\1Y', 'g')                        fooXarYXazY
regexp_replace('A PostgreSQL function', 'a|e|i|o|u', 'X', 1, 0, 'i')      X PXstgrXSQL fXnctXXn
regexp_replace(string=>'A PostgreSQL function', pattern=>'a|e|i|o|u',
               replacement=>'X', start=>1, "N"=>3, flags=>'i')           A PostgrXSQL function
```

Die Funktion `regexp_split_to_table` teilt eine Zeichenkette mit einem Muster eines regulären POSIX-Ausdrucks als Trennzeichen. Sie hat die Syntax `regexp_split_to_table(string, pattern [, flags ])`. Wenn es keinen Treffer zum Muster gibt, gibt die Funktion die Zeichenkette zurück. Wenn es mindestens einen Treffer gibt, gibt sie für jeden Treffer den Text vom Ende des letzten Treffers (oder vom Anfang der Zeichenkette) bis zum Anfang des Treffers zurück. Wenn es keine weiteren Treffer mehr gibt, gibt sie den Text vom Ende des letzten Treffers bis zum Ende der Zeichenkette zurück. Der Parameter `flags` ist eine optionale Textzeichenkette mit null oder mehr einbuchstabigen Flags, die das Verhalten der Funktion ändern. `regexp_split_to_table` unterstützt die in Tabelle 9.24 beschriebenen Flags.

Die Funktion `regexp_split_to_array` verhält sich genauso wie `regexp_split_to_table`, außer dass `regexp_split_to_array` ihr Ergebnis als Textarray zurückgibt. Sie hat die Syntax `regexp_split_to_array(string, pattern [, flags ])`. Die Parameter sind dieselben wie bei `regexp_split_to_table`.

Einige Beispiele:

```sql
SELECT foo
FROM regexp_split_to_table('the quick brown fox jumps over the lazy dog', '\s+') AS foo;
```

```text
  foo
-------
 the
 quick
 brown
 fox
 jumps
 over
 the
 lazy
 dog
(9 rows)
```

```sql
SELECT regexp_split_to_array('the quick brown fox jumps over the lazy dog', '\s+');
```

```text
              regexp_split_to_array
-----------------------------------------------
 {the,quick,brown,fox,jumps,over,the,lazy,dog}
(1 row)
```

```sql
SELECT foo
FROM regexp_split_to_table('the quick brown fox', '\s*') AS foo;
```

```text
 foo
-----
 t
 h
 e
 q
 u
 i
 c
 k
 b
 r
 o
 w
 n
 f
 o
 x
(16 rows)
```

Wie das letzte Beispiel zeigt, ignorieren die Regexp-Split-Funktionen Treffer der Länge null, die am Anfang oder Ende der Zeichenkette oder unmittelbar nach einem vorherigen Treffer auftreten. Das steht im Gegensatz zur strengen Definition des Regexp-Abgleichs, die von den anderen Regexp-Funktionen implementiert wird, ist in der Praxis aber meist das bequemste Verhalten. Andere Softwaresysteme wie Perl verwenden ähnliche Definitionen.

Die Funktion `regexp_substr` gibt den Teilstring zurück, der zu einem Muster eines regulären POSIX-Ausdrucks passt, oder `NULL`, wenn es keinen Treffer gibt. Sie hat die Syntax `regexp_substr(string, pattern [, start [, N [, flags [, subexpr ]]]])`. `pattern` wird in `string` gesucht, normalerweise vom Anfang der Zeichenkette an; wenn der Parameter `start` angegeben ist, beginnt die Suche jedoch bei diesem Zeichenindex. Wenn `N` angegeben ist, wird der `N`-te Treffer des Musters zurückgegeben, andernfalls der erste Treffer. Der Parameter `flags` ist eine optionale Textzeichenkette mit null oder mehr einbuchstabigen Flags, die das Verhalten der Funktion ändern. Die unterstützten Flags sind in Tabelle 9.24 beschrieben. Bei einem Muster mit geklammerten Teilausdrücken ist `subexpr` eine Ganzzahl, die angibt, welcher Teilausdruck von Interesse ist; das Ergebnis ist der Teilstring, der zu diesem Teilausdruck passt. Teilausdrücke werden in der Reihenfolge ihrer öffnenden Klammern nummeriert. Wenn `subexpr` weggelassen oder null ist, ist das Ergebnis unabhängig von geklammerten Teilausdrücken der gesamte Treffer.

Einige Beispiele:

```text
regexp_substr('number of your street, town zip, FR', '[^,]+', 1, 2)        town zip
regexp_substr('ABCDEFGHI', '(c..)(...)', 1, 1, 'i', 2)                    FGH
```

#### 9.7.3.1. Details regulärer Ausdrücke

Die regulären Ausdrücke von PostgreSQL sind mit einem Softwarepaket implementiert, das von Henry Spencer geschrieben wurde. Ein großer Teil der folgenden Beschreibung regulärer Ausdrücke ist wörtlich seinem Handbuch entnommen.

Reguläre Ausdrücke (REs), wie sie in POSIX 1003.2 definiert sind, gibt es in zwei Formen: erweiterte reguläre Ausdrücke oder EREs, ungefähr die von `egrep`, und grundlegende reguläre Ausdrücke oder BREs, ungefähr die von `ed`. PostgreSQL unterstützt beide Formen und implementiert außerdem einige Erweiterungen, die nicht im POSIX-Standard enthalten sind, aber durch ihre Verfügbarkeit in Programmiersprachen wie Perl und Tcl verbreitet wurden. REs, die diese Nicht-POSIX-Erweiterungen verwenden, heißen in dieser Dokumentation fortgeschrittene reguläre Ausdrücke oder AREs. AREs sind fast eine echte Obermenge von EREs; BREs weisen jedoch mehrere Notationsinkompatibilitäten auf und sind zudem deutlich eingeschränkter. Zunächst werden die Formen ARE und ERE beschrieben, wobei Merkmale genannt werden, die nur für AREs gelten; danach wird beschrieben, worin sich BREs unterscheiden.

> Hinweis: PostgreSQL nimmt zunächst immer an, dass ein regulärer Ausdruck den ARE-Regeln folgt. Die eingeschränkteren ERE- oder BRE-Regeln können jedoch gewählt werden, indem dem RE-Muster eine eingebettete Option vorangestellt wird, wie in Abschnitt 9.7.3.4 beschrieben. Das kann nützlich sein, um Kompatibilität mit Anwendungen herzustellen, die exakt die POSIX-1003.2-Regeln erwarten.

Ein regulärer Ausdruck ist als ein oder mehrere Zweige definiert, die durch `|` getrennt sind. Er passt auf alles, was zu einem der Zweige passt.

Ein Zweig besteht aus null oder mehr quantifizierten Atomen oder Einschränkungen, die aneinandergereiht sind. Er passt auf einen Treffer des ersten Elements, gefolgt von einem Treffer des zweiten Elements usw.; ein leerer Zweig passt auf die leere Zeichenkette.

Ein quantifiziertes Atom ist ein Atom, dem optional ein einzelner Quantifizierer folgt. Ohne Quantifizierer passt es auf einen Treffer des Atoms. Mit Quantifizierer kann es auf eine bestimmte Anzahl von Treffern des Atoms passen. Ein Atom kann eine der in Tabelle 9.17 gezeigten Möglichkeiten sein. Die möglichen Quantifizierer und ihre Bedeutungen zeigt Tabelle 9.18.

Eine Einschränkung passt auf eine leere Zeichenkette, aber nur, wenn bestimmte Bedingungen erfüllt sind. Eine Einschränkung kann dort verwendet werden, wo auch ein Atom verwendet werden könnte, darf jedoch nicht von einem Quantifizierer gefolgt werden. Die einfachen Einschränkungen zeigt Tabelle 9.19; weitere Einschränkungen werden später beschrieben.

**Tabelle 9.17. Atome regulärer Ausdrücke**

| Atom | Beschreibung |
| --- | --- |
| `(re)` | Wobei `re` ein beliebiger regulärer Ausdruck ist: passt auf einen Treffer für `re`, wobei der Treffer für mögliche Ausgabezwecke vermerkt wird. |
| `(?:re)` | Wie oben, aber der Treffer wird nicht für Ausgabezwecke vermerkt; dies ist ein nicht erfassendes Klammerpaar, nur in AREs. |
| `.` | Passt auf ein beliebiges einzelnes Zeichen. |
| `[chars]` | Ein Klammerausdruck, der auf eines der Zeichen in `chars` passt; weitere Details siehe Abschnitt 9.7.3.2. |
| `\k` | Wobei `k` ein nicht alphanumerisches Zeichen ist: passt auf dieses Zeichen als gewöhnliches Zeichen, zum Beispiel passt `\\` auf einen Backslash. |
| `\c` | Wobei `c` alphanumerisch ist und eventuell von weiteren Zeichen gefolgt wird: ein Escape, siehe Abschnitt 9.7.3.3; nur in AREs. In EREs und BREs passt dies auf `c`. |
| `{` | Wenn darauf ein Zeichen folgt, das keine Ziffer ist, passt es auf die linke geschweifte Klammer `{`; wenn darauf eine Ziffer folgt, beginnt damit eine Grenze, siehe unten. |
| `x` | Wobei `x` ein einzelnes Zeichen ohne sonstige besondere Bedeutung ist: passt auf dieses Zeichen. |

Ein RE darf nicht mit einem Backslash (`\`) enden.

> Hinweis: Wenn `standard_conforming_strings` ausgeschaltet ist, müssen alle Backslashes, die Sie in literalen Zeichenkettenkonstanten schreiben, verdoppelt werden. Weitere Informationen finden Sie in [Abschnitt 4.1.2.1](04_SQL_Syntax.md#4121-zeichenkettenkonstanten).

**Tabelle 9.18. Quantifizierer regulärer Ausdrücke**

| Quantifizierer | Passt auf |
| --- | --- |
| `*` | Eine Folge von 0 oder mehr Treffern des Atoms. |
| `+` | Eine Folge von 1 oder mehr Treffern des Atoms. |
| `?` | Eine Folge von 0 oder 1 Treffern des Atoms. |
| `{m}` | Eine Folge von genau `m` Treffern des Atoms. |
| `{m,}` | Eine Folge von `m` oder mehr Treffern des Atoms. |
| `{m,n}` | Eine Folge von `m` bis einschließlich `n` Treffern des Atoms; `m` darf nicht größer als `n` sein. |
| `*?` | Nicht-gierige Version von `*`. |
| `+?` | Nicht-gierige Version von `+`. |
| `??` | Nicht-gierige Version von `?`. |
| `{m}?` | Nicht-gierige Version von `{m}`. |
| `{m,}?` | Nicht-gierige Version von `{m,}`. |
| `{m,n}?` | Nicht-gierige Version von `{m,n}`. |

Die Formen mit `{...}` heißen Grenzen. Die Zahlen `m` und `n` innerhalb einer Grenze sind vorzeichenlose Dezimalzahlen mit zulässigen Werten von 0 bis einschließlich 255.

Nicht-gierige Quantifizierer, die nur in AREs verfügbar sind, passen auf dieselben Möglichkeiten wie ihre entsprechenden normalen, gierigen Gegenstücke, bevorzugen aber die kleinste statt der größten Anzahl von Treffern. Weitere Details finden Sie in Abschnitt 9.7.3.5.

> Hinweis: Ein Quantifizierer darf nicht unmittelbar auf einen anderen Quantifizierer folgen; `**` ist zum Beispiel ungültig. Ein Quantifizierer darf außerdem keinen Ausdruck oder Teilausdruck beginnen und nicht auf `^` oder `|` folgen.

**Tabelle 9.19. Einschränkungen regulärer Ausdrücke**

| Einschränkung | Beschreibung |
| --- | --- |
| `^` | Passt am Anfang der Zeichenkette. |
| `$` | Passt am Ende der Zeichenkette. |
| `(?=re)` | Positiver Lookahead: passt an jeder Stelle, an der ein Teilstring beginnt, der zu `re` passt; nur in AREs. |
| `(?!re)` | Negativer Lookahead: passt an jeder Stelle, an der kein Teilstring beginnt, der zu `re` passt; nur in AREs. |
| `(?<=re)` | Positiver Lookbehind: passt an jeder Stelle, an der ein Teilstring endet, der zu `re` passt; nur in AREs. |
| `(?<!re)` | Negativer Lookbehind: passt an jeder Stelle, an der kein Teilstring endet, der zu `re` passt; nur in AREs. |

Lookahead- und Lookbehind-Einschränkungen dürfen keine Rückverweise enthalten, siehe Abschnitt 9.7.3.3; alle Klammern darin gelten als nicht erfassend.

#### 9.7.3.2. Klammerausdrücke

Ein Klammerausdruck ist eine in `[]` eingeschlossene Liste von Zeichen. Normalerweise passt er auf ein einzelnes Zeichen aus der Liste, mit den unten beschriebenen Ausnahmen. Wenn die Liste mit `^` beginnt, passt er auf ein einzelnes Zeichen, das nicht im Rest der Liste vorkommt. Wenn zwei Zeichen in der Liste durch `-` getrennt sind, ist dies eine Kurzschreibweise für den vollständigen Bereich von Zeichen zwischen diesen beiden Zeichen einschließlich der Endpunkte in der Sortierreihenfolge; zum Beispiel passt `[0-9]` in ASCII auf jede Dezimalziffer. Es ist unzulässig, dass zwei Bereiche einen Endpunkt teilen, etwa `a-c-e`. Bereiche hängen stark von der Sortierreihenfolge ab; portable Programme sollten sich daher nicht auf sie verlassen.

Um ein wörtliches `]` in die Liste aufzunehmen, machen Sie es zum ersten Zeichen, nach `^`, falls dieses verwendet wird. Um ein wörtliches `-` aufzunehmen, machen Sie es zum ersten oder letzten Zeichen oder zum zweiten Endpunkt eines Bereichs. Um ein wörtliches `-` als ersten Endpunkt eines Bereichs zu verwenden, schließen Sie es in `[.` und `.]` ein, damit es zu einem Sortierelement wird. Mit Ausnahme dieser Zeichen, einiger Kombinationen mit `[` und Escapes, nur in AREs, verlieren alle anderen Sonderzeichen innerhalb eines Klammerausdrucks ihre besondere Bedeutung. Insbesondere ist `\` nach ERE- oder BRE-Regeln nicht besonders, während es in AREs als Einleitung eines Escapes besonders ist.

Innerhalb eines Klammerausdrucks steht ein in `[.` und `.]` eingeschlossenes Sortierelement, also ein Zeichen, eine Mehrzeichenfolge, die wie ein einzelnes Zeichen sortiert, oder ein Name in der Sortierreihenfolge, für die Zeichenfolge dieses Sortierelements. Die Folge wird als ein einzelnes Element der Liste des Klammerausdrucks behandelt. Dadurch kann ein Klammerausdruck, der ein Mehrzeichen-Sortierelement enthält, auf mehr als ein Zeichen passen; wenn die Sortierreihenfolge zum Beispiel ein Sortierelement `ch` enthält, passt der RE `[[.ch.]]*c` auf die ersten fünf Zeichen von `chchcc`.

> Hinweis: PostgreSQL unterstützt derzeit keine Mehrzeichen-Sortierelemente. Diese Information beschreibt mögliches zukünftiges Verhalten.

Innerhalb eines Klammerausdrucks ist ein in `[=` und `=]` eingeschlossenes Sortierelement eine Äquivalenzklasse; sie steht für die Zeichenfolgen aller Sortierelemente, die diesem Element äquivalent sind, einschließlich des Elements selbst. Wenn es keine anderen äquivalenten Sortierelemente gibt, wird dies behandelt, als wären die einschließenden Trenner `[.` und `.]`. Wenn zum Beispiel `o` und `^` Mitglieder einer Äquivalenzklasse sind, sind `[[=o=]]`, `[[=^=]]` und `[o^]` synonym. Eine Äquivalenzklasse kann kein Endpunkt eines Bereichs sein.

Innerhalb eines Klammerausdrucks steht der in `[:` und `:]` eingeschlossene Name einer Zeichenklasse für die Liste aller Zeichen, die zu dieser Klasse gehören. Eine Zeichenklasse kann nicht als Endpunkt eines Bereichs verwendet werden. Der POSIX-Standard definiert diese Zeichenklassennamen: `alnum` (Buchstaben und Ziffern), `alpha` (Buchstaben), `blank` (Leerzeichen und Tabulator), `cntrl` (Steuerzeichen), `digit` (Ziffern), `graph` (druckbare Zeichen außer Leerzeichen), `lower` (Kleinbuchstaben), `print` (druckbare Zeichen einschließlich Leerzeichen), `punct` (Satzzeichen), `space` (beliebiger Leerraum), `upper` (Großbuchstaben) und `xdigit` (hexadezimale Ziffern). Das Verhalten dieser Standardzeichenklassen ist für Zeichen im 7-Bit-ASCII-Satz im Allgemeinen plattformübergreifend konsistent. Ob ein bestimmtes Nicht-ASCII-Zeichen als Mitglied einer dieser Klassen gilt, hängt von der Collation ab, die für die Funktion oder den Operator mit regulärem Ausdruck verwendet wird (siehe [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung)), oder standardmäßig von der Datenbank-Locale `LC_CTYPE` (siehe [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung)). Die Klassifizierung von Nicht-ASCII-Zeichen kann sogar zwischen ähnlich benannten Locales plattformabhängig variieren. Die Locale `C` betrachtet jedoch nie Nicht-ASCII-Zeichen als Mitglieder einer dieser Klassen. Zusätzlich zu diesen Standardzeichenklassen definiert PostgreSQL die Zeichenklasse `word`, die `alnum` plus dem Unterstrich (`_`) entspricht, und die Zeichenklasse `ascii`, die genau den 7-Bit-ASCII-Satz enthält.

Es gibt zwei Sonderfälle von Klammerausdrücken: Die Klammerausdrücke `[[:<:]]` und `[[:>:]]` sind Einschränkungen, die auf leere Zeichenketten am Anfang beziehungsweise Ende eines Wortes passen. Ein Wort ist als Folge von Wortzeichen definiert, der weder ein Wortzeichen vorausgeht noch folgt. Ein Wortzeichen ist jedes Zeichen aus der Zeichenklasse `word`, also jeder Buchstabe, jede Ziffer oder der Unterstrich. Dies ist eine Erweiterung, die mit POSIX 1003.2 kompatibel, aber dort nicht spezifiziert ist; in Software, die auf andere Systeme portierbar sein soll, sollte sie mit Vorsicht verwendet werden. Die unten beschriebenen Einschränkungs-Escapes sind meist vorzuziehen; sie sind nicht standardkonformer, aber einfacher zu tippen.

#### 9.7.3.3. Escapes regulärer Ausdrücke

Escapes sind besondere Folgen, die mit `\` beginnen und von einem alphanumerischen Zeichen gefolgt werden. Es gibt mehrere Arten von Escapes: Zeicheneingabe, Klassenkurzformen, Einschränkungs-Escapes und Rückverweise. Ein `\`, dem ein alphanumerisches Zeichen folgt, das aber kein gültiges Escape bildet, ist in AREs unzulässig. In EREs gibt es keine Escapes: Außerhalb eines Klammerausdrucks steht ein `\` gefolgt von einem alphanumerischen Zeichen lediglich für dieses gewöhnliche Zeichen, und innerhalb eines Klammerausdrucks ist `\` ein gewöhnliches Zeichen. Letzteres ist die einzige echte Inkompatibilität zwischen EREs und AREs.

Zeicheneingabe-Escapes dienen dazu, nichtdruckbare oder sonst unhandliche Zeichen in REs leichter anzugeben. Sie sind in Tabelle 9.20 gezeigt.

Klassenkurzform-Escapes bieten Kurzschreibweisen für bestimmte häufig verwendete Zeichenklassen. Sie sind in Tabelle 9.21 gezeigt.

Ein Einschränkungs-Escape ist eine als Escape geschriebene Einschränkung, die auf die leere Zeichenkette passt, wenn bestimmte Bedingungen erfüllt sind. Sie sind in Tabelle 9.22 gezeigt.

Ein Rückverweis (`\n`) passt auf dieselbe Zeichenkette, die durch den vorherigen geklammerten Teilausdruck mit der Nummer `n` getroffen wurde (siehe Tabelle 9.23). Zum Beispiel passt `([bc])\1` auf `bb` oder `cc`, aber nicht auf `bc` oder `cb`. Der Teilausdruck muss dem Rückverweis im RE vollständig vorausgehen. Teilausdrücke werden in der Reihenfolge ihrer öffnenden Klammern nummeriert. Nicht erfassende Klammern definieren keine Teilausdrücke. Der Rückverweis betrachtet nur die Zeichen der Zeichenkette, die vom referenzierten Teilausdruck getroffen wurden, nicht darin enthaltene Einschränkungen. Zum Beispiel passt `(^\d)\1` auf `22`.

**Tabelle 9.20. Zeicheneingabe-Escapes regulärer Ausdrücke**

| Escape | Beschreibung |
| --- | --- |
| `\a` | Alarmzeichen (Bell), wie in C. |
| `\b` | Backspace, wie in C. |
| `\B` | Synonym für Backslash (`\`), um die Notwendigkeit zu verringern, Backslashes zu verdoppeln. |
| `\cX` | Wobei `X` ein beliebiges Zeichen ist: das Zeichen, dessen niederwertige 5 Bits denen von `X` entsprechen und dessen übrige Bits null sind. |
| `\e` | Das Zeichen, dessen Name in der Sortierreihenfolge `ESC` ist, oder andernfalls das Zeichen mit dem oktalen Wert `033`. |
| `\f` | Seitenvorschub, wie in C. |
| `\n` | Zeilenumbruch, wie in C. |
| `\r` | Wagenrücklauf, wie in C. |
| `\t` | Horizontaler Tabulator, wie in C. |
| `\uwxyz` | Wobei `wxyz` genau vier hexadezimale Ziffern sind: das Zeichen mit dem Hexadezimalwert `0xwxyz`. |
| `\Ustuvwxyz` | Wobei `stuvwxyz` genau acht hexadezimale Ziffern sind: das Zeichen mit dem Hexadezimalwert `0xstuvwxyz`. |
| `\v` | Vertikaler Tabulator, wie in C. |
| `\xhhh` | Wobei `hhh` eine beliebige Folge hexadezimaler Ziffern ist: das Zeichen mit dem Hexadezimalwert `0xhhh`; es ist ein einzelnes Zeichen, unabhängig davon, wie viele hexadezimale Ziffern verwendet werden. |
| `\0` | Das Zeichen mit dem Wert 0, also das Nullbyte. |
| `\xy` | Wobei `xy` genau zwei oktale Ziffern sind und kein Rückverweis vorliegt: das Zeichen mit dem oktalen Wert `0xy`. |
| `\xyz` | Wobei `xyz` genau drei oktale Ziffern sind und kein Rückverweis vorliegt: das Zeichen mit dem oktalen Wert `0xyz`. |

Hexadezimale Ziffern sind `0` bis `9`, `a` bis `f` und `A` bis `F`. Oktale Ziffern sind `0` bis `7`.

Numerische Zeicheneingabe-Escapes, die Werte außerhalb des ASCII-Bereichs (0-127) angeben, haben eine von der Datenbankkodierung abhängige Bedeutung. Bei der Kodierung UTF-8 entsprechen Escape-Werte Unicode-Codepoints; `\u1234` bedeutet zum Beispiel das Zeichen `U+1234`. Bei anderen Mehrbyte-Kodierungen geben Zeicheneingabe-Escapes normalerweise nur die Verkettung der Bytewerte für das Zeichen an. Wenn der Escape-Wert keinem gültigen Zeichen in der Datenbankkodierung entspricht, wird kein Fehler ausgelöst, aber er wird nie auf Daten passen.

Zeicheneingabe-Escapes werden immer als gewöhnliche Zeichen behandelt. Zum Beispiel ist `\135` in ASCII `]`, aber `\135` beendet keinen Klammerausdruck.

**Tabelle 9.21. Klassenkurzform-Escapes regulärer Ausdrücke**

| Escape | Beschreibung |
| --- | --- |
| `\d` | Passt auf eine beliebige Ziffer, wie `[[:digit:]]`. |
| `\s` | Passt auf ein beliebiges Leerraumzeichen, wie `[[:space:]]`. |
| `\w` | Passt auf ein beliebiges Wortzeichen, wie `[[:word:]]`. |
| `\D` | Passt auf ein beliebiges Nicht-Ziffernzeichen, wie `[^[:digit:]]`. |
| `\S` | Passt auf ein beliebiges Nicht-Leerraumzeichen, wie `[^[:space:]]`. |
| `\W` | Passt auf ein beliebiges Nicht-Wortzeichen, wie `[^[:word:]]`. |

Die Klassenkurzform-Escapes funktionieren auch innerhalb von Klammerausdrücken, obwohl die oben gezeigten Definitionen in diesem Kontext syntaktisch nicht ganz gültig sind. Zum Beispiel ist `[a-c\d]` äquivalent zu `[a-c[:digit:]]`.

**Tabelle 9.22. Einschränkungs-Escapes regulärer Ausdrücke**

| Escape | Beschreibung |
| --- | --- |
| `\A` | Passt nur am Anfang der Zeichenkette; zum Unterschied zu `^` siehe Abschnitt 9.7.3.5. |
| `\m` | Passt nur am Anfang eines Wortes. |
| `\M` | Passt nur am Ende eines Wortes. |
| `\y` | Passt nur am Anfang oder Ende eines Wortes. |
| `\Y` | Passt nur an einer Stelle, die nicht Anfang oder Ende eines Wortes ist. |
| `\Z` | Passt nur am Ende der Zeichenkette; zum Unterschied zu `$` siehe Abschnitt 9.7.3.5. |

Ein Wort ist wie bei der Spezifikation von `[[:<:]]` und `[[:>:]]` oben definiert. Einschränkungs-Escapes sind innerhalb von Klammerausdrücken unzulässig.

**Tabelle 9.23. Rückverweise regulärer Ausdrücke**

| Escape | Beschreibung |
| --- | --- |
| `\m` | Wobei `m` eine von null verschiedene Ziffer ist: ein Rückverweis auf den `m`-ten Teilausdruck. |
| `\mnn` | Wobei `m` eine von null verschiedene Ziffer ist, `nn` weitere Ziffern sind und der Dezimalwert `mnn` nicht größer ist als die Anzahl der bisher gesehenen schließenden erfassenden Klammern: ein Rückverweis auf den `mnn`-ten Teilausdruck. |

> Hinweis: Zwischen oktalen Zeicheneingabe-Escapes und Rückverweisen gibt es eine inhärente Mehrdeutigkeit, die durch die oben angedeuteten Heuristiken aufgelöst wird. Eine führende Null bezeichnet immer ein oktales Escape. Eine einzelne von null verschiedene Ziffer, der keine weitere Ziffer folgt, wird immer als Rückverweis verstanden. Eine mehrstellige Folge, die nicht mit null beginnt, wird als Rückverweis verstanden, wenn sie nach einem passenden Teilausdruck steht, also wenn die Zahl im zulässigen Bereich für einen Rückverweis liegt; andernfalls wird sie als oktal verstanden.

#### 9.7.3.4. Metasyntax regulärer Ausdrücke

Zusätzlich zur oben beschriebenen Hauptsyntax gibt es einige besondere Formen und verschiedene syntaktische Hilfsmittel.

Ein RE kann mit einem von zwei speziellen Direktivenpräfixen beginnen. Wenn ein RE mit `***:` beginnt, wird der Rest des RE als ARE behandelt. In PostgreSQL hat dies normalerweise keine Wirkung, da REs ohnehin als AREs angenommen werden; es wirkt sich aber aus, wenn durch den Parameter `flags` einer Regex-Funktion der ERE- oder BRE-Modus angegeben wurde. Wenn ein RE mit `***=` beginnt, wird der Rest des RE als wörtliche Zeichenkette behandelt, in der alle Zeichen gewöhnliche Zeichen sind.

Ein ARE kann mit eingebetteten Optionen beginnen: Eine Folge `(?xyz)`, wobei `xyz` aus einem oder mehreren alphabetischen Zeichen besteht, legt Optionen fest, die den Rest des RE beeinflussen. Diese Optionen überschreiben alle zuvor bestimmten Optionen; insbesondere können sie das durch einen Regex-Operator oder den Parameter `flags` einer Regex-Funktion implizierte Verhalten zur Beachtung der Groß-/Kleinschreibung überschreiben. Die verfügbaren Optionsbuchstaben zeigt Tabelle 9.24. Beachten Sie, dass dieselben Optionsbuchstaben in den Parametern `flags` von Regex-Funktionen verwendet werden.

**Tabelle 9.24. Eingebettete Optionsbuchstaben für AREs**

| Option | Beschreibung |
| --- | --- |
| `b` | Der Rest des RE ist ein BRE. |
| `c` | Abgleich mit Beachtung der Groß-/Kleinschreibung; überschreibt den Operatortyp. |
| `e` | Der Rest des RE ist ein ERE. |
| `i` | Abgleich ohne Beachtung der Groß-/Kleinschreibung; siehe Abschnitt 9.7.3.5; überschreibt den Operatortyp. |
| `m` | Historisches Synonym für `n`. |
| `n` | Newline-sensitiver Abgleich; siehe Abschnitt 9.7.3.5. |
| `p` | Teilweise Newline-sensitiver Abgleich; siehe Abschnitt 9.7.3.5. |
| `q` | Der Rest des RE ist eine wörtliche, „quotierte“ Zeichenkette; alle Zeichen sind gewöhnlich. |
| `s` | Nicht-Newline-sensitiver Abgleich; Voreinstellung. |
| `t` | Enge Syntax; Voreinstellung, siehe unten. |
| `w` | Invers teilweise Newline-sensitiver, „weird“, Abgleich; siehe Abschnitt 9.7.3.5. |
| `x` | Erweiterte Syntax; siehe unten. |

Eingebettete Optionen werden an der `)` wirksam, die die Folge beendet. Sie dürfen nur am Anfang eines ARE auftreten, nach der Direktive `***:`, falls vorhanden.

Neben der üblichen, engen RE-Syntax, in der alle Zeichen signifikant sind, gibt es eine erweiterte Syntax, die durch Angabe der eingebetteten Option `x` verfügbar ist. In der erweiterten Syntax werden Leerraumzeichen im RE ignoriert, ebenso alle Zeichen zwischen `#` und dem folgenden Zeilenumbruch oder dem Ende des RE. Das erlaubt eine absatzartige Gliederung und Kommentierung eines komplexen RE. Es gibt drei Ausnahmen von dieser Grundregel:

- Ein Leerraumzeichen oder `#`, dem `\` vorausgeht, bleibt erhalten.
- Leerraum oder `#` innerhalb eines Klammerausdrucks bleibt erhalten.
- Leerraum und Kommentare dürfen nicht innerhalb mehrzeichiger Symbole wie `(?:` auftreten.

Für diesen Zweck sind Leerraumzeichen Leerzeichen, Tabulator, Zeilenumbruch und jedes Zeichen, das zur Zeichenklasse `space` gehört.

Schließlich ist in einem ARE außerhalb von Klammerausdrücken die Folge `(?#ttt)`, wobei `ttt` beliebiger Text ohne `)` ist, ein Kommentar und wird vollständig ignoriert. Auch dies ist zwischen den Zeichen mehrzeichiger Symbole wie `(?:` nicht erlaubt. Solche Kommentare sind eher ein historisches Artefakt als eine nützliche Einrichtung, und ihre Verwendung ist veraltet; verwenden Sie stattdessen die erweiterte Syntax.

Keine dieser Metasyntax-Erweiterungen ist verfügbar, wenn eine anfängliche Direktive `***=` festgelegt hat, dass die Eingabe des Benutzers als wörtliche Zeichenkette und nicht als RE behandelt werden soll.

#### 9.7.3.5. Abgleichregeln regulärer Ausdrücke

Wenn ein RE auf mehr als einen Teilstring einer gegebenen Zeichenkette passen könnte, passt der RE auf den Teilstring, der in der Zeichenkette am frühesten beginnt. Wenn der RE auf mehr als einen Teilstring passen könnte, der an dieser Stelle beginnt, wird je nachdem, ob der RE gierig oder nicht-gierig ist, der längstmögliche oder der kürzestmögliche Treffer genommen.

Ob ein RE gierig ist, wird durch folgende Regeln bestimmt:

- Die meisten Atome und alle Einschränkungen haben kein Gierigkeitsattribut, weil sie ohnehin nicht auf variable Textmengen passen können.
- Klammern um einen RE ändern seine Gierigkeit nicht.
- Ein quantifiziertes Atom mit einem Quantifizierer fester Wiederholung (`{m}` oder `{m}?`) hat dieselbe Gierigkeit, möglicherweise keine, wie das Atom selbst.
- Ein quantifiziertes Atom mit anderen normalen Quantifizierern, einschließlich `{m,n}` mit `m` gleich `n`, ist gierig und bevorzugt den längsten Treffer.
- Ein quantifiziertes Atom mit einem nicht-gierigen Quantifizierer, einschließlich `{m,n}?` mit `m` gleich `n`, ist nicht-gierig und bevorzugt den kürzesten Treffer.
- Ein Zweig, also ein RE ohne `|`-Operator auf oberster Ebene, hat dieselbe Gierigkeit wie das erste quantifizierte Atom darin, das ein Gierigkeitsattribut hat.
- Ein RE, der aus zwei oder mehr durch `|` verbundenen Zweigen besteht, ist immer gierig.

Diese Regeln ordnen Gierigkeitsattribute nicht nur einzelnen quantifizierten Atomen zu, sondern auch Zweigen und ganzen REs, die quantifizierte Atome enthalten. Das bedeutet, dass der Abgleich so erfolgt, dass der Zweig oder der ganze RE als Ganzes auf den längsten oder kürzesten möglichen Teilstring passt. Sobald die Länge des gesamten Treffers bestimmt ist, wird der Teil davon, der zu einem bestimmten Teilausdruck passt, auf Grundlage des Gierigkeitsattributs dieses Teilausdrucks bestimmt, wobei Teilausdrücke, die im RE früher beginnen, Vorrang vor später beginnenden haben.

Ein Beispiel dafür:

```sql
SELECT SUBSTRING('XY1234Z', 'Y*([0-9]{1,3})');
```

```text
Result: 123
```

```sql
SELECT SUBSTRING('XY1234Z', 'Y*?([0-9]{1,3})');
```

```text
Result: 1
```

Im ersten Fall ist der RE als Ganzes gierig, weil `Y*` gierig ist. Er kann beim `Y` beginnen und passt auf die längstmögliche dort beginnende Zeichenkette, also `Y123`. Die Ausgabe ist der geklammerte Teil davon, also `123`. Im zweiten Fall ist der RE als Ganzes nicht-gierig, weil `Y*?` nicht-gierig ist. Er kann beim `Y` beginnen und passt auf die kürzestmögliche dort beginnende Zeichenkette, also `Y1`. Der Teilausdruck `[0-9]{1,3}` ist gierig, kann aber die Entscheidung über die Gesamtlänge des Treffers nicht ändern; daher wird er gezwungen, nur `1` zu treffen.

Kurz gesagt: Wenn ein RE sowohl gierige als auch nicht-gierige Teilausdrücke enthält, ist die Gesamtlänge des Treffers je nach Attribut des gesamten RE entweder so lang wie möglich oder so kurz wie möglich. Die den Teilausdrücken zugeordneten Attribute beeinflussen nur, wie viel von diesem Treffer sie relativ zueinander „essen“ dürfen.

Die Quantifizierer `{1,1}` und `{1,1}?` können verwendet werden, um Gierigkeit beziehungsweise Nicht-Gierigkeit für einen Teilausdruck oder einen ganzen RE zu erzwingen. Das ist nützlich, wenn der gesamte RE ein anderes Gierigkeitsattribut haben soll, als aus seinen Elementen abgeleitet wird. Angenommen, wir möchten eine Zeichenkette, die Ziffern enthält, in die Ziffern und die Teile davor und danach zerlegen. Wir könnten es so versuchen:

```sql
SELECT regexp_match('abc01234xyz', '(.*)(\d+)(.*)');
```

```text
Result: {abc0123,4,xyz}
```

Das hat nicht funktioniert: Das erste `.*` ist gierig und „isst“ daher so viel wie möglich, sodass `\d+` an der spätestmöglichen Stelle passt, der letzten Ziffer. Wir könnten versuchen, das zu beheben, indem wir es nicht-gierig machen:

```sql
SELECT regexp_match('abc01234xyz', '(.*?)(\d+)(.*)');
```

```text
Result: {abc,0,""}
```

Auch das hat nicht funktioniert, weil der RE als Ganzes nun nicht-gierig ist und den Gesamttreffer daher so früh wie möglich beendet. Wir erhalten das gewünschte Ergebnis, indem wir den RE als Ganzes auf gierig zwingen:

```sql
SELECT regexp_match('abc01234xyz', '(?:(.*?)(\d+)(.*)){1,1}');
```

```text
Result: {abc,01234,xyz}
```

Die getrennte Steuerung der Gesamtgierigkeit des RE und der Gierigkeit seiner Bestandteile erlaubt große Flexibilität beim Umgang mit Mustern variabler Länge.

Bei der Entscheidung, was ein längerer oder kürzerer Treffer ist, werden Trefferlängen in Zeichen gemessen, nicht in Sortierelementen. Eine leere Zeichenkette gilt als länger als gar kein Treffer. Zum Beispiel passt `bb*` auf die drei mittleren Zeichen von `abbbc`; `(week|wee)(night|knights)` passt auf alle zehn Zeichen von `weeknights`; wenn `(.*).*` gegen `abc` abgeglichen wird, passt der geklammerte Teilausdruck auf alle drei Zeichen; und wenn `(a*)*` gegen `bc` abgeglichen wird, passen sowohl der gesamte RE als auch der geklammerte Teilausdruck auf eine leere Zeichenkette.

Wenn ein Abgleich ohne Beachtung der Groß-/Kleinschreibung angegeben ist, wirkt dies ungefähr so, als wären alle Groß-/Kleinschreibungsunterschiede aus dem Alphabet verschwunden. Wenn ein Buchstabe, den es in mehreren Schreibweisen gibt, als gewöhnliches Zeichen außerhalb eines Klammerausdrucks erscheint, wird er effektiv in einen Klammerausdruck umgewandelt, der beide Schreibweisen enthält; `x` wird zum Beispiel zu `[xX]`. Wenn er innerhalb eines Klammerausdrucks erscheint, werden alle Schreibweisen dieses Zeichens zum Klammerausdruck hinzugefügt; `[x]` wird zum Beispiel zu `[xX]` und `[^x]` zu `[^xX]`.

Wenn Newline-sensitiver Abgleich angegeben ist, passen `.` und Klammerausdrücke mit `^` nie auf das Zeilenumbruchzeichen, sodass Treffer keine Zeilen überschreiten, außer der RE enthält ausdrücklich einen Zeilenumbruch; außerdem passen `^` und `$` zusätzlich zu Anfang und Ende der Zeichenkette auch auf die leere Zeichenkette nach beziehungsweise vor einem Zeilenumbruch. Die ARE-Escapes `\A` und `\Z` passen dagegen weiterhin nur auf Anfang beziehungsweise Ende der Zeichenkette. Außerdem passen die Zeichenklassenkurzformen `\D` und `\W` unabhängig von diesem Modus auf einen Zeilenumbruch. Vor PostgreSQL 14 passten sie im Newline-sensitiven Modus nicht auf Zeilenumbrüche. Schreiben Sie `[^[:digit:]]` oder `[^[:word:]]`, um das alte Verhalten zu erhalten.

Wenn teilweise Newline-sensitiver Abgleich angegeben ist, wirkt sich dies wie beim Newline-sensitiven Abgleich auf `.` und Klammerausdrücke aus, aber nicht auf `^` und `$`.

Wenn invers teilweise Newline-sensitiver Abgleich angegeben ist, wirkt sich dies wie beim Newline-sensitiven Abgleich auf `^` und `$` aus, aber nicht auf `.` und Klammerausdrücke. Das ist nicht sehr nützlich, wird aber der Symmetrie wegen bereitgestellt.

#### 9.7.3.6. Grenzen und Kompatibilität

Diese Implementierung legt keine bestimmte Grenze für die Länge von REs fest. Programme, die besonders portabel sein sollen, sollten jedoch keine REs mit mehr als 256 Bytes verwenden, da eine POSIX-konforme Implementierung solche REs ablehnen darf.

Das einzige ARE-Merkmal, das tatsächlich mit POSIX-EREs inkompatibel ist, besteht darin, dass `\` innerhalb von Klammerausdrücken seine besondere Bedeutung nicht verliert. Alle anderen ARE-Merkmale verwenden Syntax, die in POSIX-EREs unzulässig ist oder undefinierte beziehungsweise nicht spezifizierte Auswirkungen hat; die `***`-Syntax der Direktiven liegt ebenfalls außerhalb der POSIX-Syntax sowohl für BREs als auch für EREs.

Viele ARE-Erweiterungen sind von Perl übernommen, aber einige wurden bereinigt verändert, und einige Perl-Erweiterungen fehlen. Nennenswerte Inkompatibilitäten sind `\b`, `\B`, das Fehlen einer Sonderbehandlung für einen abschließenden Zeilenumbruch, die Ergänzung komplementierter Klammerausdrücke zu den Dingen, die durch Newline-sensitiven Abgleich beeinflusst werden, die Einschränkungen für Klammern und Rückverweise in Lookahead-/Lookbehind-Einschränkungen sowie die Semantik des längsten/kürzesten Treffers statt einer First-Match-Semantik.

#### 9.7.3.7. Grundlegende reguläre Ausdrücke

BREs unterscheiden sich in mehreren Punkten von EREs. In BREs sind `|`, `+` und `?` gewöhnliche Zeichen, und es gibt kein Äquivalent zu ihrer Funktionalität. Die Trenner für Grenzen sind `\{` und `\}`, während `{` und `}` für sich gewöhnliche Zeichen sind. Die Klammern für verschachtelte Teilausdrücke sind `\(` und `\)`, während `(` und `)` für sich gewöhnliche Zeichen sind. `^` ist ein gewöhnliches Zeichen, außer am Anfang des RE oder am Anfang eines geklammerten Teilausdrucks; `$` ist ein gewöhnliches Zeichen, außer am Ende des RE oder am Ende eines geklammerten Teilausdrucks; und `*` ist ein gewöhnliches Zeichen, wenn es am Anfang des RE oder am Anfang eines geklammerten Teilausdrucks erscheint, nach einem möglichen führenden `^`. Schließlich sind einstellige Rückverweise verfügbar, und `\<` und `\>` sind Synonyme für `[[:<:]]` beziehungsweise `[[:>:]]`; andere Escapes sind in BREs nicht verfügbar.

#### 9.7.3.8. Unterschiede zum SQL-Standard und zu XQuery

Seit SQL:2008 enthält der SQL-Standard Operatoren und Funktionen für reguläre Ausdrücke, die Mustervergleich nach dem Standard für reguläre XQuery-Ausdrücke durchführen:

- `LIKE_REGEX`
- `OCCURRENCES_REGEX`
- `POSITION_REGEX`
- `SUBSTRING_REGEX`
- `TRANSLATE_REGEX`

PostgreSQL implementiert diese Operatoren und Funktionen derzeit nicht. Ungefähr äquivalente Funktionalität können Sie in jedem Fall wie in Tabelle 9.25 gezeigt erhalten. Verschiedene optionale Klauseln auf beiden Seiten wurden in dieser Tabelle weggelassen.

**Tabelle 9.25. Entsprechungen für Funktionen regulärer Ausdrücke**

| SQL-Standard | PostgreSQL |
| --- | --- |
| `string LIKE_REGEX pattern` | `regexp_like(string, pattern)` oder `string ~ pattern` |
| `OCCURRENCES_REGEX(pattern IN string)` | `regexp_count(string, pattern)` |
| `POSITION_REGEX(pattern IN string)` | `regexp_instr(string, pattern)` |
| `SUBSTRING_REGEX(pattern IN string)` | `regexp_substr(string, pattern)` |
| `TRANSLATE_REGEX(pattern IN string WITH replacement)` | `regexp_replace(string, pattern, replacement)` |

Funktionen für reguläre Ausdrücke, die denen von PostgreSQL ähneln, sind auch in einigen anderen SQL-Implementierungen verfügbar, während die SQL-Standardfunktionen weniger breit implementiert sind. Einige Details der Syntax regulärer Ausdrücke werden sich wahrscheinlich in jeder Implementierung unterscheiden.

Die Operatoren und Funktionen des SQL-Standards verwenden reguläre XQuery-Ausdrücke, die der oben beschriebenen ARE-Syntax recht nahekommen. Nennenswerte Unterschiede zwischen der vorhandenen POSIX-basierten Funktionalität für reguläre Ausdrücke und regulären XQuery-Ausdrücken sind:

- Die Subtraktion von XQuery-Zeichenklassen wird nicht unterstützt. Ein Beispiel dafür wäre `[a-z-[aeiou]]`, um nur englische Konsonanten zu treffen.
- Die XQuery-Zeichenklassenkurzformen `\c`, `\C`, `\i` und `\I` werden nicht unterstützt.
- XQuery-Zeichenklassenelemente mit `\p{UnicodeProperty}` oder dem inversen `\P{UnicodeProperty}` werden nicht unterstützt.
- POSIX interpretiert Zeichenklassen wie `\w` (siehe Tabelle 9.21) gemäß der jeweils geltenden Locale, die Sie steuern können, indem Sie dem Operator oder der Funktion eine `COLLATE`-Klausel anhängen. XQuery definiert diese Klassen anhand von Unicode-Zeicheneigenschaften; äquivalentes Verhalten erhalten Sie daher nur mit einer Locale, die den Unicode-Regeln folgt.
- Der SQL-Standard, nicht XQuery selbst, versucht mehr Varianten von „Newline“ abzudecken als POSIX. Die oben beschriebenen Newline-sensitiven Abgleichoptionen betrachten nur ASCII-NL (`\n`) als Zeilenumbruch, während SQL verlangt, auch CR (`\r`), CRLF (`\r\n`), also einen Windows-Zeilenumbruch, und einige nur in Unicode vorkommende Zeichen wie LINE SEPARATOR (`U+2028`) als Zeilenumbrüche zu behandeln. Insbesondere sollen `.` und `\s` nach SQL `\r\n` als ein Zeichen zählen, nicht als zwei.
- Von den in Tabelle 9.20 beschriebenen Zeicheneingabe-Escapes unterstützt XQuery nur `\n`, `\r` und `\t`.
- XQuery unterstützt die Syntax `[:name:]` für Zeichenklassen innerhalb von Klammerausdrücken nicht.
- XQuery kennt weder Lookahead- noch Lookbehind-Einschränkungen und auch keine der in Tabelle 9.22 beschriebenen Einschränkungs-Escapes.
- Die in Abschnitt 9.7.3.4 beschriebenen Metasyntax-Formen existieren in XQuery nicht.
- Die von XQuery definierten Flag-Buchstaben sind verwandt mit den POSIX-Optionsbuchstaben (Tabelle 9.24), aber nicht dieselben. Während sich die Optionen `i` und `q` gleich verhalten, gilt das für andere nicht.
- Die XQuery-Flags `s` (Punkt darf auf Newline passen) und `m` (`^` und `$` dürfen an Newlines passen) bieten Zugriff auf dieselben Verhaltensweisen wie die POSIX-Flags `n`, `p` und `w`, entsprechen aber nicht dem Verhalten der POSIX-Flags `s` und `m`. Beachten Sie insbesondere, dass „Punkt passt auf Newline“ in POSIX das Standardverhalten ist, in XQuery aber nicht.
- Das XQuery-Flag `x` (Leerraum im Muster ignorieren) unterscheidet sich merklich vom erweiterten POSIX-Modus. Das POSIX-Flag `x` erlaubt außerdem, dass `#` im Muster einen Kommentar beginnt, und POSIX ignoriert ein Leerraumzeichen nach einem Backslash nicht.

## 9.8. Formatierungsfunktionen für Datentypen

Die Formatierungsfunktionen von PostgreSQL stellen einen leistungsfähigen Werkzeugsatz bereit, um verschiedene Datentypen (Datum/Zeit, Ganzzahlen, Gleitkommazahlen, `numeric`) in formatierte Zeichenketten umzuwandeln und um aus formatierten Zeichenketten in bestimmte Datentypen umzuwandeln. Tabelle 9.26 listet diese Funktionen auf. Alle diese Funktionen folgen einer gemeinsamen Aufrufkonvention: Das erste Argument ist der zu formatierende Wert, und das zweite Argument ist eine Vorlage, die das Ausgabe- oder Eingabeformat definiert.

**Tabelle 9.26. Formatierungsfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `to_char ( timestamp, text )` -> `text`<br>`to_char ( timestamp with time zone, text )` -> `text` | Wandelt einen Zeitstempel gemäß dem angegebenen Format in eine Zeichenkette um. | `to_char(timestamp '2002-04-20 17:31:12.66', 'HH12:MI:SS')` -> `05:31:12` |
| `to_char ( interval, text )` -> `text` | Wandelt ein Intervall gemäß dem angegebenen Format in eine Zeichenkette um. | `to_char(interval '15h 2m 12s', 'HH24:MI:SS')` -> `15:02:12` |
| `to_char ( numeric_type, text )` -> `text` | Wandelt eine Zahl gemäß dem angegebenen Format in eine Zeichenkette um; verfügbar für `integer`, `bigint`, `numeric`, `real` und `double precision`. | `to_char(125, '999')` -> `125`<br>`to_char(125.8::real, '999D9')` -> `125.8`<br>`to_char(-125.8, '999D99S')` -> `125.80-` |
| `to_date ( text, text )` -> `date` | Wandelt eine Zeichenkette gemäß dem angegebenen Format in ein Datum um. | `to_date('05 Dec 2000', 'DD Mon YYYY')` -> `2000-12-05` |
| `to_number ( text, text )` -> `numeric` | Wandelt eine Zeichenkette gemäß dem angegebenen Format in `numeric` um. | `to_number('12,454.8-', '99G999D9S')` -> `-12454.8` |
| `to_timestamp ( text, text )` -> `timestamp with time zone` | Wandelt eine Zeichenkette gemäß dem angegebenen Format in einen Zeitstempel um. Siehe auch `to_timestamp(double precision)` in Tabelle 9.33. | `to_timestamp('05 Dec 2000', 'DD Mon YYYY')` -> `2000-12-05 00:00:00-05` |

> Tipp: `to_timestamp` und `to_date` existieren, um Eingabeformate zu behandeln, die nicht durch einfaches Casten umgewandelt werden können. Für die meisten Standardformate für Datum/Zeit funktioniert es, die Quellzeichenkette einfach in den benötigten Datentyp zu casten, und das ist wesentlich einfacher. Ebenso ist `to_number` für standardmäßige numerische Darstellungen nicht nötig.

In einer Ausgabevorlage für `to_char` gibt es bestimmte Muster, die erkannt und anhand des angegebenen Werts durch passend formatierte Daten ersetzt werden. Text, der kein Vorlagenmuster ist, wird einfach wörtlich kopiert. Entsprechend identifizieren Vorlagenmuster in einer Eingabevorlage für die anderen Funktionen die Werte, die durch die Eingabezeichenkette geliefert werden sollen. Wenn die Vorlagenzeichenkette Zeichen enthält, die keine Vorlagenmuster sind, werden die entsprechenden Zeichen in der Eingabezeichenkette einfach übersprungen, unabhängig davon, ob sie den Zeichen der Vorlagenzeichenkette entsprechen.

Tabelle 9.27 zeigt die verfügbaren Vorlagenmuster zum Formatieren von Datums- und Zeitwerten.

**Tabelle 9.27. Vorlagenmuster für Datums-/Zeitformatierung**

| Muster | Beschreibung |
| --- | --- |
| `HH` | Stunde des Tages (01-12). |
| `HH12` | Stunde des Tages (01-12). |
| `HH24` | Stunde des Tages (00-23). |
| `MI` | Minute (00-59). |
| `SS` | Sekunde (00-59). |
| `MS` | Millisekunde (000-999). |
| `US` | Mikrosekunde (000000-999999). |
| `FF1` | Zehntelsekunde (0-9). |
| `FF2` | Hundertstelsekunde (00-99). |
| `FF3` | Millisekunde (000-999). |
| `FF4` | Zehntel einer Millisekunde (0000-9999). |
| `FF5` | Hundertstel einer Millisekunde (00000-99999). |
| `FF6` | Mikrosekunde (000000-999999). |
| `SSSS`, `SSSSS` | Sekunden seit Mitternacht (0-86399). |
| `AM`, `am`, `PM` oder `pm` | Meridiem-Anzeige ohne Punkte. |
| `A.M.`, `a.m.`, `P.M.` oder `p.m.` | Meridiem-Anzeige mit Punkten. |
| `Y,YYY` | Jahr mit Komma, 4 oder mehr Ziffern. |
| `YYYY` | Jahr, 4 oder mehr Ziffern. |
| `YYY` | Letzte 3 Ziffern des Jahres. |
| `YY` | Letzte 2 Ziffern des Jahres. |
| `Y` | Letzte Ziffer des Jahres. |
| `IYYY` | ISO-8601-Wochennummerierungsjahr, 4 oder mehr Ziffern. |
| `IYY` | Letzte 3 Ziffern des ISO-8601-Wochennummerierungsjahres. |
| `IY` | Letzte 2 Ziffern des ISO-8601-Wochennummerierungsjahres. |
| `I` | Letzte Ziffer des ISO-8601-Wochennummerierungsjahres. |
| `BC`, `bc`, `AD` oder `ad` | Ära-Anzeige ohne Punkte. |
| `B.C.`, `b.c.`, `A.D.` oder `a.d.` | Ära-Anzeige mit Punkten. |
| `MONTH` | Vollständiger Monatsname in Großbuchstaben, mit Leerzeichen auf 9 Zeichen aufgefüllt. |
| `Month` | Vollständiger Monatsname mit großem Anfangsbuchstaben, mit Leerzeichen auf 9 Zeichen aufgefüllt. |
| `month` | Vollständiger Monatsname in Kleinbuchstaben, mit Leerzeichen auf 9 Zeichen aufgefüllt. |
| `MON` | Abgekürzter Monatsname in Großbuchstaben; 3 Zeichen im Englischen, lokalisierte Längen variieren. |
| `Mon` | Abgekürzter Monatsname mit großem Anfangsbuchstaben; 3 Zeichen im Englischen, lokalisierte Längen variieren. |
| `mon` | Abgekürzter Monatsname in Kleinbuchstaben; 3 Zeichen im Englischen, lokalisierte Längen variieren. |
| `MM` | Monatsnummer (01-12). |
| `DAY` | Vollständiger Tagesname in Großbuchstaben, mit Leerzeichen auf 9 Zeichen aufgefüllt. |
| `Day` | Vollständiger Tagesname mit großem Anfangsbuchstaben, mit Leerzeichen auf 9 Zeichen aufgefüllt. |
| `day` | Vollständiger Tagesname in Kleinbuchstaben, mit Leerzeichen auf 9 Zeichen aufgefüllt. |
| `DY` | Abgekürzter Tagesname in Großbuchstaben; 3 Zeichen im Englischen, lokalisierte Längen variieren. |
| `Dy` | Abgekürzter Tagesname mit großem Anfangsbuchstaben; 3 Zeichen im Englischen, lokalisierte Längen variieren. |
| `dy` | Abgekürzter Tagesname in Kleinbuchstaben; 3 Zeichen im Englischen, lokalisierte Längen variieren. |
| `DDD` | Tag des Jahres (001-366). |
| `IDDD` | Tag des ISO-8601-Wochennummerierungsjahres (001-371; Tag 1 des Jahres ist Montag der ersten ISO-Woche). |
| `DD` | Tag des Monats (01-31). |
| `D` | Tag der Woche, Sonntag (1) bis Samstag (7). |
| `ID` | ISO-8601-Tag der Woche, Montag (1) bis Sonntag (7). |
| `W` | Woche des Monats (1-5); die erste Woche beginnt am ersten Tag des Monats. |
| `WW` | Wochennummer des Jahres (1-53); die erste Woche beginnt am ersten Tag des Jahres. |
| `IW` | Wochennummer des ISO-8601-Wochennummerierungsjahres (01-53; der erste Donnerstag des Jahres liegt in Woche 1). |
| `CC` | Jahrhundert, 2 Ziffern; das 21. Jahrhundert beginnt am 2001-01-01. |
| `J` | Julianisches Datum, ganzzahlige Tage seit dem 24. November 4714 v. Chr. um lokale Mitternacht; siehe Abschnitt B.7. |
| `Q` | Quartal. |
| `RM` | Monat in römischen Großbuchstaben (I-XII; I=Januar). |
| `rm` | Monat in römischen Kleinbuchstaben (i-xii; i=Januar). |
| `TZ` | Zeitzonenabkürzung in Großbuchstaben. |
| `tz` | Zeitzonenabkürzung in Kleinbuchstaben. |
| `TZH` | Zeitzonenstunden. |
| `TZM` | Zeitzonenminuten. |
| `OF` | Zeitzonenversatz gegenüber UTC (`HH` oder `HH:MM`). |

Modifikatoren können auf jedes Vorlagenmuster angewendet werden, um sein Verhalten zu ändern. Zum Beispiel ist `FMMonth` das Muster `Month` mit dem Modifikator `FM`. Tabelle 9.28 zeigt die Modifikatormuster für die Datums-/Zeitformatierung.

**Tabelle 9.28. Modifikatoren für Datums-/Zeitformatierung**

| Modifikator | Beschreibung | Beispiel |
| --- | --- | --- |
| Präfix `FM` | Fill-Modus: unterdrückt führende Nullen und auffüllende Leerzeichen. | `FMMonth` |
| Suffix `TH` | Ordinalzahlsuffix in Großbuchstaben. | `DDTH`, z. B. `12TH` |
| Suffix `th` | Ordinalzahlsuffix in Kleinbuchstaben. | `DDth`, z. B. `12th` |
| Präfix `FX` | Globale Option für festes Format; siehe Hinweise zur Verwendung. | `FX Month DD Day` |
| Präfix `TM` | Übersetzungsmodus: verwendet lokalisierte Tages- und Monatsnamen auf Basis von `lc_time`. | `TMMonth` |
| Suffix `SP` | Spell-Modus; nicht implementiert. | `DDSP` |

Hinweise zur Verwendung der Datums-/Zeitformatierung:

- `FM` unterdrückt führende Nullen und nachgestellte Leerzeichen, die sonst hinzugefügt würden, um die Ausgabe eines Musters auf feste Breite zu bringen. In PostgreSQL verändert `FM` nur die nächste Spezifikation, während `FM` in Oracle alle folgenden Spezifikationen beeinflusst und wiederholte `FM`-Modifikatoren den Fill-Modus ein- und ausschalten.
- `TM` unterdrückt nachgestellte Leerzeichen unabhängig davon, ob `FM` angegeben ist.
- `to_timestamp` und `to_date` ignorieren die Groß-/Kleinschreibung in der Eingabe; daher akzeptieren zum Beispiel `MON`, `Mon` und `mon` dieselben Zeichenketten. Bei Verwendung des Modifikators `TM` erfolgt die Groß-/Kleinschreibungsfaltung nach den Regeln der Eingabe-Collation der Funktion (siehe [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung)).
- `to_timestamp` und `to_date` überspringen mehrere Leerzeichen am Anfang der Eingabezeichenkette und um Datums- und Zeitwerte herum, sofern die Option `FX` nicht verwendet wird. Zum Beispiel funktionieren `to_timestamp(' 2000         JUN', 'YYYY MON')` und `to_timestamp('2000 - JUN', 'YYYY-MON')`, aber `to_timestamp('2000               JUN', 'FXYYYY MON')` gibt einen Fehler zurück, weil `to_timestamp` nur ein einzelnes Leerzeichen erwartet. `FX` muss als erstes Element der Vorlage angegeben werden.
- Ein Trennzeichen, also ein Leerzeichen oder ein Nicht-Buchstaben-/Nicht-Ziffernzeichen, in der Vorlagenzeichenkette von `to_timestamp` und `to_date` passt auf ein beliebiges einzelnes Trennzeichen in der Eingabezeichenkette oder wird übersprungen, sofern die Option `FX` nicht verwendet wird. Zum Beispiel funktionieren `to_timestamp('2000JUN', 'YYYY///MON')` und `to_timestamp('2000/JUN', 'YYYY MON')`, aber `to_timestamp('2000//JUN', 'YYYY/MON')` gibt einen Fehler zurück, weil die Anzahl der Trennzeichen in der Eingabezeichenkette die Anzahl der Trennzeichen in der Vorlage übersteigt.
- Wenn `FX` angegeben ist, passt ein Trennzeichen in der Vorlagenzeichenkette exakt auf ein Zeichen in der Eingabezeichenkette. Beachten Sie aber, dass das Zeichen aus der Eingabezeichenkette nicht dasselbe sein muss wie das Trennzeichen aus der Vorlagenzeichenkette. Zum Beispiel funktioniert `to_timestamp('2000/JUN', 'FXYYYY MON')`, während `to_timestamp('2000/JUN', 'FXYYYY  MON')` einen Fehler zurückgibt, weil das zweite Leerzeichen in der Vorlage den Buchstaben `J` aus der Eingabezeichenkette verbraucht.
- Ein `TZH`-Vorlagenmuster kann auf eine vorzeichenbehaftete Zahl passen. Ohne die Option `FX` können Minuszeichen mehrdeutig sein und als Trennzeichen interpretiert werden. Diese Mehrdeutigkeit wird wie folgt aufgelöst: Wenn die Anzahl der Trennzeichen vor `TZH` in der Vorlagenzeichenkette kleiner ist als die Anzahl der Trennzeichen vor dem Minuszeichen in der Eingabezeichenkette, wird das Minuszeichen als Teil von `TZH` interpretiert. Andernfalls gilt das Minuszeichen als Trennzeichen zwischen Werten.
- Gewöhnlicher Text ist in `to_char`-Vorlagen erlaubt und wird wörtlich ausgegeben. Sie können einen Teilstring in doppelte Anführungszeichen setzen, um zu erzwingen, dass er als wörtlicher Text interpretiert wird, selbst wenn er Vorlagenmuster enthält. Zum Beispiel wird in `"Hello Year "YYYY` das `YYYY` durch die Jahresdaten ersetzt, das einzelne `Y` in `Year` jedoch nicht. In `to_date`, `to_number` und `to_timestamp` führen wörtlicher Text und doppelt gequotete Zeichenketten dazu, dass die entsprechende Anzahl von Eingabezeichen übersprungen wird; zum Beispiel überspringt `"XX"` zwei Eingabezeichen, unabhängig davon, ob sie `XX` sind.

> Tipp: Vor PostgreSQL 12 war es möglich, beliebigen Text in der Eingabezeichenkette mit Nicht-Buchstaben- oder Nicht-Ziffernzeichen zu überspringen. Zum Beispiel funktionierte früher `to_timestamp('2000y6m1d', 'yyyy-MM-DD')`. Jetzt können Sie für diesen Zweck nur Buchstaben verwenden. Zum Beispiel überspringen `to_timestamp('2000y6m1d', 'yyyytMMtDDt')` und `to_timestamp('2000y6m1d', 'yyyy"y"MM"m"DD"d"')` die Zeichen `y`, `m` und `d`.

- Wenn Sie in der Ausgabe ein doppeltes Anführungszeichen haben möchten, müssen Sie ihm einen Backslash voranstellen, zum Beispiel `'\"YYYY Month\"'`. Backslashes sind außerhalb doppelt gequoteter Zeichenketten sonst nicht besonders. Innerhalb einer doppelt gequoteten Zeichenkette bewirkt ein Backslash, dass das nächste Zeichen wörtlich genommen wird, welches es auch ist; eine besondere Wirkung hat das aber nur, wenn das nächste Zeichen ein doppeltes Anführungszeichen oder ein weiterer Backslash ist.
- Wenn in `to_timestamp` und `to_date` die Jahresformatspezifikation weniger als vier Ziffern hat, zum Beispiel `YYY`, und das gelieferte Jahr weniger als vier Ziffern enthält, wird das Jahr auf das Jahr eingestellt, das 2020 am nächsten liegt; zum Beispiel wird `95` zu `1995`.
- In `to_timestamp` und `to_date` werden negative Jahre als v. Chr. behandelt. Wenn Sie sowohl ein negatives Jahr als auch ein explizites `BC`-Feld schreiben, erhalten Sie wieder n. Chr. Eine Eingabe mit Jahr null wird als 1 v. Chr. behandelt.
- In `to_timestamp` und `to_date` hat die Umwandlung `YYYY` eine Einschränkung bei der Verarbeitung von Jahren mit mehr als vier Ziffern. Nach `YYYY` müssen Sie ein Nicht-Ziffernzeichen oder eine Vorlage verwenden, sonst wird das Jahr immer als vierstellig interpretiert. Zum Beispiel wird `to_date('200001130', 'YYYYMMDD')` als vierstelliges Jahr interpretiert; verwenden Sie stattdessen nach dem Jahr ein Nicht-Ziffern-Trennzeichen, etwa `to_date('20000-1130', 'YYYY-MMDD')`, oder `to_date('20000Nov30', 'YYYYMonDD')`.
- In `to_timestamp` und `to_date` wird das Feld `CC` (Jahrhundert) akzeptiert, aber ignoriert, wenn ein Feld `YYY`, `YYYY` oder `Y,YYY` vorhanden ist. Wenn `CC` mit `YY` oder `Y` verwendet wird, wird das Ergebnis als dieses Jahr im angegebenen Jahrhundert berechnet. Wenn das Jahrhundert angegeben ist, aber das Jahr nicht, wird das erste Jahr des Jahrhunderts angenommen.
- In `to_timestamp` und `to_date` werden Wochentagsnamen oder -nummern (`DAY`, `D` und verwandte Feldtypen) akzeptiert, aber für die Berechnung des Ergebnisses ignoriert. Dasselbe gilt für Quartalsfelder (`Q`).
- In `to_timestamp` und `to_date` kann ein ISO-8601-Wochennummerierungsdatum, im Unterschied zu einem gregorianischen Datum, auf zwei Arten angegeben werden: Jahr, Wochennummer und Wochentag, zum Beispiel gibt `to_date('2006-42-4', 'IYYY-IW-ID')` das Datum `2006-10-19` zurück; wenn Sie den Wochentag weglassen, wird 1 (Montag) angenommen. Oder Jahr und Tag des Jahres, zum Beispiel gibt `to_date('2006-291', 'IYYY-IDDD')` ebenfalls `2006-10-19` zurück.
- Der Versuch, ein Datum mit einer Mischung aus ISO-8601-Wochennummerierungsfeldern und gregorianischen Datumsfeldern einzugeben, ist unsinnig und verursacht einen Fehler. Im Kontext eines ISO-8601-Wochennummerierungsjahres hat das Konzept „Monat“ oder „Tag des Monats“ keine Bedeutung. Im Kontext eines gregorianischen Jahres hat die ISO-Woche keine Bedeutung.

> Achtung: Während `to_date` eine Mischung aus gregorianischen und ISO-Wochennummerierungs-Datumsfeldern ablehnt, tut `to_char` das nicht, da Ausgabespezifikationen wie `YYYY-MM-DD (IYYY-IDDD)` nützlich sein können. Vermeiden Sie aber Schreibweisen wie `IYYY-MM-DD`; das würde nahe am Jahresanfang überraschende Ergebnisse liefern. Weitere Informationen finden Sie in Abschnitt 9.9.1.

- In `to_timestamp` werden Millisekundenfelder (`MS`) oder Mikrosekundenfelder (`US`) als Sekundenziffern nach dem Dezimalpunkt verwendet. Zum Beispiel ist `to_timestamp('12.3', 'SS.MS')` nicht 3 Millisekunden, sondern 300, weil die Umwandlung dies als 12 + 0,3 Sekunden behandelt. Für das Format `SS.MS` geben die Eingabewerte `12.3`, `12.30` und `12.300` daher dieselbe Anzahl von Millisekunden an. Um drei Millisekunden zu erhalten, muss man `12.003` schreiben, was die Umwandlung als 12 + 0,003 = 12,003 Sekunden behandelt.
- Ein komplexeres Beispiel: `to_timestamp('15:12:02.020.001230', 'HH24:MI:SS.MS.US')` ist 15 Stunden, 12 Minuten und 2 Sekunden + 20 Millisekunden + 1230 Mikrosekunden = 2,021230 Sekunden.
- Die Wochentagsnummerierung von `to_char(..., 'ID')` entspricht der Funktion `extract(isodow from ...)`, die von `to_char(..., 'D')` entspricht aber nicht der Nummerierung von `extract(dow from ...)`.
- `to_char(interval)` formatiert `HH` und `HH12` wie auf einer 12-Stunden-Uhr; zum Beispiel werden sowohl null Stunden als auch 36 Stunden als `12` ausgegeben, während `HH24` den vollen Stundenwert ausgibt, der in einem Intervallwert größer als 23 sein kann.

Tabelle 9.29 zeigt die verfügbaren Vorlagenmuster zum Formatieren numerischer Werte.

**Tabelle 9.29. Vorlagenmuster für numerische Formatierung**

| Muster | Beschreibung |
| --- | --- |
| `9` | Ziffernposition; kann weggelassen werden, wenn nicht signifikant. |
| `0` | Ziffernposition; wird nicht weggelassen, auch wenn nicht signifikant. |
| `.` (Punkt) | Dezimalpunkt. |
| `,` (Komma) | Gruppierungs-, Tausender-, Trennzeichen. |
| `PR` | Negativer Wert in spitzen Klammern. |
| `S` | Am Zahlenwert verankertes Vorzeichen; verwendet die Locale. |
| `L` | Währungssymbol; verwendet die Locale. |
| `D` | Dezimalpunkt; verwendet die Locale. |
| `G` | Gruppierungstrennzeichen; verwendet die Locale. |
| `MI` | Minuszeichen an angegebener Position, wenn die Zahl < 0 ist. |
| `PL` | Pluszeichen an angegebener Position, wenn die Zahl > 0 ist. |
| `SG` | Plus-/Minuszeichen an angegebener Position. |
| `RN` oder `rn` | Römische Zahl; Werte zwischen 1 und 3999. |
| `TH` oder `th` | Ordinalzahlsuffix. |
| `V` | Verschiebt die angegebene Anzahl von Ziffern; siehe Hinweise. |
| `EEEE` | Exponent für wissenschaftliche Notation. |

Hinweise zur Verwendung der numerischen Formatierung:

- `0` gibt eine Ziffernposition an, die immer ausgegeben wird, selbst wenn sie eine führende oder nachgestellte Null enthält. `9` gibt ebenfalls eine Ziffernposition an; eine führende Null wird jedoch durch ein Leerzeichen ersetzt, während eine nachgestellte Null bei angegebenem Fill-Modus gelöscht wird. Für `to_number()` sind diese beiden Musterzeichen äquivalent.
- Wenn das Format weniger Nachkommastellen bereitstellt als die zu formatierende Zahl, rundet `to_char()` die Zahl auf die angegebene Anzahl von Nachkommastellen.
- Die Musterzeichen `S`, `L`, `D` und `G` stehen für die durch die aktuelle Locale definierten Zeichen für Vorzeichen, Währungssymbol, Dezimalpunkt und Tausendertrennzeichen, siehe `lc_monetary` und `lc_numeric`. Punkt und Komma als Musterzeichen stehen unabhängig von der Locale für genau diese Zeichen mit der Bedeutung Dezimalpunkt beziehungsweise Tausendertrennzeichen.
- Wenn im Muster von `to_char()` keine ausdrückliche Stelle für ein Vorzeichen vorgesehen ist, wird eine Spalte für das Vorzeichen reserviert, und es wird am Zahlenwert verankert, erscheint also direkt links von der Zahl. Wenn `S` direkt links von einigen `9` steht, wird es ebenfalls am Zahlenwert verankert.
- Ein mit `SG`, `PL` oder `MI` formatiertes Vorzeichen ist nicht am Zahlenwert verankert; zum Beispiel erzeugt `to_char(-12, 'MI9999')` den Wert `'- 12'`, aber `to_char(-12, 'S9999')` erzeugt `' -12'`. Die Oracle-Implementierung erlaubt `MI` nicht vor `9`, sondern verlangt, dass `9` vor `MI` steht.
- `TH` wandelt keine Werte kleiner als null und keine Bruchzahlen um.
- `PL`, `SG` und `TH` sind PostgreSQL-Erweiterungen.
- Wenn in `to_number` Nicht-Daten-Vorlagenmuster wie `L` oder `TH` verwendet werden, wird die entsprechende Anzahl von Eingabezeichen übersprungen, unabhängig davon, ob sie zum Vorlagenmuster passen, sofern sie keine Datenzeichen sind, also Ziffern, Vorzeichen, Dezimalpunkt oder Komma. Zum Beispiel würde `TH` zwei Nicht-Datenzeichen überspringen.
- `V` mit `to_char` multipliziert die Eingabewerte mit `10^n`, wobei `n` die Anzahl der Ziffern nach `V` ist. `V` mit `to_number` dividiert entsprechend. Man kann `V` als Markierung der Position eines impliziten Dezimalpunkts in der Eingabe- oder Ausgabezeichenkette verstehen. `to_char` und `to_number` unterstützen nicht die Verwendung von `V` zusammen mit einem Dezimalpunkt; `99.9V99` ist zum Beispiel nicht erlaubt.
- `EEEE` (wissenschaftliche Notation) darf nicht mit anderen Formatierungsmustern oder Modifikatoren kombiniert werden, außer mit Ziffern- und Dezimalpunktmustern, und muss am Ende der Formatzeichenkette stehen; `9.99EEEE` ist zum Beispiel ein gültiges Muster.
- In `to_number()` wandelt das Muster `RN` römische Zahlen in Standardform in Zahlen um. Die Eingabe ignoriert Groß-/Kleinschreibung, daher sind `RN` und `rn` äquivalent. `RN` kann nicht mit anderen Formatierungsmustern oder Modifikatoren kombiniert werden, außer mit `FM`, das nur in `to_char()` anwendbar ist und in `to_number()` ignoriert wird.

Bestimmte Modifikatoren können auf jedes Vorlagenmuster angewendet werden, um sein Verhalten zu ändern. Zum Beispiel ist `FM99.99` das Muster `99.99` mit dem Modifikator `FM`. Tabelle 9.30 zeigt die Modifikatormuster für die numerische Formatierung.

**Tabelle 9.30. Modifikatoren für numerische Formatierung**

| Modifikator | Beschreibung | Beispiel |
| --- | --- | --- |
| Präfix `FM` | Fill-Modus: unterdrückt nachgestellte Nullen und auffüllende Leerzeichen. | `FM99.99` |
| Suffix `TH` | Ordinalzahlsuffix in Großbuchstaben. | `999TH` |
| Suffix `th` | Ordinalzahlsuffix in Kleinbuchstaben. | `999th` |

Tabelle 9.31 zeigt einige Beispiele für die Verwendung der Funktion `to_char`.

**Tabelle 9.31. Beispiele für `to_char`**

| Ausdruck | Ergebnis |
| --- | --- |
| `to_char(current_timestamp, 'Day, DD HH12:MI:SS')` | `'Tuesday      , 06    05:39:18'` |
| `to_char(current_timestamp, 'FMDay, FMDD HH12:MI:SS')` | `'Tuesday, 6        05:39:18'` |
| `to_char(current_timestamp AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"')` | `'2022-12-06T05:39:18Z'`, erweitertes ISO-8601-Format |
| `to_char(-0.1, '99.99')` | `'   -.10'` |
| `to_char(-0.1, 'FM9.99')` | `'-.1'` |
| `to_char(-0.1, 'FM90.99')` | `'-0.1'` |
| `to_char(0.1, '0.9')` | `' 0.1'` |
| `to_char(12, '9990999.9')` | `'        0012.0'` |
| `to_char(12, 'FM9990999.9')` | `'0012.'` |
| `to_char(485, '999')` | `' 485'` |
| `to_char(-485, '999')` | `'-485'` |
| `to_char(485, '9 9 9')` | `' 4 8 5'` |
| `to_char(1485, '9,999')` | `' 1,485'` |
| `to_char(1485, '9G999')` | `' 1 485'` |
| `to_char(148.5, '999.999')` | `' 148.500'` |
| `to_char(148.5, 'FM999.999')` | `'148.5'` |
| `to_char(148.5, 'FM999.990')` | `'148.500'` |
| `to_char(148.5, '999D999')` | `' 148,500'` |
| `to_char(3148.5, '9G999D999')` | `' 3 148,500'` |
| `to_char(-485, '999S')` | `'485-'` |
| `to_char(-485, '999MI')` | `'485-'` |
| `to_char(485, '999MI')` | `'485 '` |
| `to_char(485, 'FM999MI')` | `'485'` |
| `to_char(485, 'PL999')` | `'+485'` |
| `to_char(485, 'SG999')` | `'+485'` |
| `to_char(-485, 'SG999')` | `'-485'` |
| `to_char(-485, '9SG99')` | `'4-85'` |
| `to_char(-485, '999PR')` | `'<485>'` |
| `to_char(485, 'L999')` | `'DM 485'` |
| `to_char(485, 'RN')` | `'            CDLXXXV'` |
| `to_char(485, 'FMRN')` | `'CDLXXXV'` |
| `to_char(5.2, 'FMRN')` | `'V'` |
| `to_char(482, '999th')` | `' 482nd'` |
| `to_char(485, '"Good number:"999')` | `'Good number: 485'` |
| `to_char(485.8, '"Pre:"999" Post:" .999')` | `'Pre: 485 Post: .800'` |
| `to_char(12, '99V999')` | `' 12000'` |
| `to_char(12.4, '99V999')` | `' 12400'` |
| `to_char(12.45, '99V9')` | `' 125'` |
| `to_char(0.0004859, '9.99EEEE')` | `' 4.86e-04'` |

## 9.9. Datums-/Zeitfunktionen und -operatoren

Tabelle 9.33 zeigt die verfügbaren Funktionen zur Verarbeitung von Datums-/Zeitwerten; Details erscheinen in den folgenden Unterabschnitten. Tabelle 9.32 veranschaulicht das Verhalten der grundlegenden arithmetischen Operatoren (`+`, `*` usw.). Formatierungsfunktionen finden Sie in [Abschnitt 9.8](09_Funktionen_und_Operatoren.md#98-formatierungsfunktionen-für-datentypen). Sie sollten mit den Hintergrundinformationen zu Datums-/Zeitdatentypen aus [Abschnitt 8.5](08_Datentypen.md#85-datumszeittypen) vertraut sein.

Zusätzlich stehen für Datums-/Zeittypen die üblichen Vergleichsoperatoren aus Tabelle 9.1 zur Verfügung. Datumswerte und Zeitstempel (mit oder ohne Zeitzone) sind alle miteinander vergleichbar, während Zeiten (mit oder ohne Zeitzone) und Intervalle nur mit anderen Werten desselben Datentyps verglichen werden können. Beim Vergleich eines Zeitstempels ohne Zeitzone mit einem Zeitstempel mit Zeitzone wird angenommen, dass der erstere Wert in der durch den Konfigurationsparameter `TimeZone` angegebenen Zeitzone vorliegt; für den Vergleich mit dem letzteren Wert, der intern bereits in UTC vorliegt, wird er nach UTC umgerechnet. Entsprechend wird ein Datumswert beim Vergleich mit einem Zeitstempel als Mitternacht in der Zone `TimeZone` angenommen.

Alle unten beschriebenen Funktionen und Operatoren, die Eingaben vom Typ `time` oder `timestamp` annehmen, gibt es tatsächlich in zwei Varianten: eine für `time with time zone` beziehungsweise `timestamp with time zone` und eine für `time without time zone` beziehungsweise `timestamp without time zone`. Der Kürze halber werden diese Varianten nicht separat gezeigt. Auch die Operatoren `+` und `*` gibt es in kommutativen Paaren, etwa sowohl `date + integer` als auch `integer + date`; es wird jeweils nur eine Form gezeigt.

**Tabelle 9.32. Datums-/Zeitoperatoren**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `date + integer` -> `date` | Addiert eine Anzahl von Tagen zu einem Datum. | `date '2001-09-28' + 7` -> `2001-10-05` |
| `date + interval` -> `timestamp` | Addiert ein Intervall zu einem Datum. | `date '2001-09-28' + interval '1 hour'` -> `2001-09-28 01:00:00` |
| `date + time` -> `timestamp` | Addiert eine Tageszeit zu einem Datum. | `date '2001-09-28' + time '03:00'` -> `2001-09-28 03:00:00` |
| `interval + interval` -> `interval` | Addiert Intervalle. | `interval '1 day' + interval '1 hour'` -> `1 day 01:00:00` |
| `timestamp + interval` -> `timestamp` | Addiert ein Intervall zu einem Zeitstempel. | `timestamp '2001-09-28 01:00' + interval '23 hours'` -> `2001-09-29 00:00:00` |
| `time + interval` -> `time` | Addiert ein Intervall zu einer Uhrzeit. | `time '01:00' + interval '3 hours'` -> `04:00:00` |
| `- interval` -> `interval` | Negiert ein Intervall. | `- interval '23 hours'` -> `-23:00:00` |
| `date - date` -> `integer` | Subtrahiert Datumswerte und erzeugt die Anzahl der vergangenen Tage. | `date '2001-10-01' - date '2001-09-28'` -> `3` |
| `date - integer` -> `date` | Subtrahiert eine Anzahl von Tagen von einem Datum. | `date '2001-10-01' - 7` -> `2001-09-24` |
| `date - interval` -> `timestamp` | Subtrahiert ein Intervall von einem Datum. | `date '2001-09-28' - interval '1 hour'` -> `2001-09-27 23:00:00` |
| `time - time` -> `interval` | Subtrahiert Uhrzeiten. | `time '05:00' - time '03:00'` -> `02:00:00` |
| `time - interval` -> `time` | Subtrahiert ein Intervall von einer Uhrzeit. | `time '05:00' - interval '2 hours'` -> `03:00:00` |
| `timestamp - interval` -> `timestamp` | Subtrahiert ein Intervall von einem Zeitstempel. | `timestamp '2001-09-28 23:00' - interval '23 hours'` -> `2001-09-28 00:00:00` |
| `interval - interval` -> `interval` | Subtrahiert Intervalle. | `interval '1 day' - interval '1 hour'` -> `1 day -01:00:00` |
| `timestamp - timestamp` -> `interval` | Subtrahiert Zeitstempel; 24-Stunden-Intervalle werden ähnlich wie bei `justify_hours()` in Tage umgewandelt. | `timestamp '2001-09-29 03:00' - timestamp '2001-07-27 12:00'` -> `63 days 15:00:00` |
| `interval * double precision` -> `interval` | Multipliziert ein Intervall mit einem Skalar. | `interval '1 second' * 900` -> `00:15:00`<br>`interval '1 day' * 21` -> `21 days`<br>`interval '1 hour' * 3.5` -> `03:30:00` |
| `interval / double precision` -> `interval` | Dividiert ein Intervall durch einen Skalar. | `interval '1 hour' / 1.5` -> `00:40:00` |

**Tabelle 9.33. Datums-/Zeitfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `age ( timestamp, timestamp )` -> `interval` | Subtrahiert die Argumente und erzeugt ein „symbolisches“ Ergebnis, das Jahre und Monate verwendet, nicht nur Tage. | `age(timestamp '2001-04-10', timestamp '1957-06-13')` -> `43 years 9 mons 27 days` |
| `age ( timestamp )` -> `interval` | Subtrahiert das Argument von `current_date` um Mitternacht. | `age(timestamp '1957-06-13')` -> `62 years 6 mons 10 days` |
| `clock_timestamp ( )` -> `timestamp with time zone` | Aktuelles Datum und aktuelle Uhrzeit; ändert sich während der Ausführung einer Anweisung. Siehe Abschnitt 9.9.5. | `clock_timestamp()` -> `2019-12-23 14:39:53.662522-05` |
| `current_date` -> `date` | Aktuelles Datum. Siehe Abschnitt 9.9.5. | `current_date` -> `2019-12-23` |
| `current_time` -> `time with time zone` | Aktuelle Tageszeit. Siehe Abschnitt 9.9.5. | `current_time` -> `14:39:53.662522-05` |
| `current_time ( integer )` -> `time with time zone` | Aktuelle Tageszeit mit begrenzter Genauigkeit. Siehe Abschnitt 9.9.5. | `current_time(2)` -> `14:39:53.66-05` |
| `current_timestamp` -> `timestamp with time zone` | Aktuelles Datum und aktuelle Uhrzeit, bezogen auf den Start der aktuellen Transaktion. Siehe Abschnitt 9.9.5. | `current_timestamp` -> `2019-12-23 14:39:53.662522-05` |
| `current_timestamp ( integer )` -> `timestamp with time zone` | Aktuelles Datum und aktuelle Uhrzeit, bezogen auf den Start der aktuellen Transaktion, mit begrenzter Genauigkeit. Siehe Abschnitt 9.9.5. | `current_timestamp(0)` -> `2019-12-23 14:39:53-05` |
| `date_add ( timestamp with time zone, interval [, text ] )` -> `timestamp with time zone` | Addiert ein Intervall zu einem `timestamp with time zone`; Tageszeiten und Sommerzeit-Anpassungen werden gemäß der im dritten Argument benannten Zeitzone berechnet, oder gemäß der aktuellen Einstellung `TimeZone`, wenn dieses Argument weggelassen wird. Die Form mit zwei Argumenten entspricht dem Operator `timestamp with time zone + interval`. | `date_add('2021-10-31 00:00:00+02'::timestamptz, '1 day'::interval, 'Europe/Warsaw')` -> `2021-10-31 23:00:00+00` |
| `date_bin ( interval, timestamp, timestamp )` -> `timestamp` | Ordnet die Eingabe einem angegebenen Intervall zu, ausgerichtet an einem angegebenen Ursprung. Siehe Abschnitt 9.9.3. | `date_bin('15 minutes', timestamp '2001-02-16 20:38:40', timestamp '2001-02-16 20:05:00')` -> `2001-02-16 20:35:00` |
| `date_part ( text, timestamp )` -> `double precision` | Holt ein Unterfeld aus einem Zeitstempel; entspricht `extract`. Siehe Abschnitt 9.9.1. | `date_part('hour', timestamp '2001-02-16 20:38:40')` -> `20` |
| `date_part ( text, interval )` -> `double precision` | Holt ein Unterfeld aus einem Intervall; entspricht `extract`. Siehe Abschnitt 9.9.1. | `date_part('month', interval '2 years 3 months')` -> `3` |
| `date_subtract ( timestamp with time zone, interval [, text ] )` -> `timestamp with time zone` | Subtrahiert ein Intervall von einem `timestamp with time zone`; Tageszeiten und Sommerzeit-Anpassungen werden gemäß der im dritten Argument benannten Zeitzone berechnet, oder gemäß der aktuellen Einstellung `TimeZone`, wenn dieses Argument weggelassen wird. Die Form mit zwei Argumenten entspricht dem Operator `timestamp with time zone - interval`. | `date_subtract('2021-11-01 00:00:00+01'::timestamptz, '1 day'::interval, 'Europe/Warsaw')` -> `2021-10-30 22:00:00+00` |
| `date_trunc ( text, timestamp )` -> `timestamp` | Schneidet auf die angegebene Genauigkeit ab. Siehe Abschnitt 9.9.2. | `date_trunc('hour', timestamp '2001-02-16 20:38:40')` -> `2001-02-16 20:00:00` |
| `date_trunc ( text, timestamp with time zone, text )` -> `timestamp with time zone` | Schneidet in der angegebenen Zeitzone auf die angegebene Genauigkeit ab. Siehe Abschnitt 9.9.2. | `date_trunc('day', timestamptz '2001-02-16 20:38:40+00', 'Australia/Sydney')` -> `2001-02-16 13:00:00+00` |
| `date_trunc ( text, interval )` -> `interval` | Schneidet auf die angegebene Genauigkeit ab. Siehe Abschnitt 9.9.2. | `date_trunc('hour', interval '2 days 3 hours 40 minutes')` -> `2 days 03:00:00` |
| `extract ( field from timestamp )` -> `numeric` | Holt ein Unterfeld aus einem Zeitstempel. Siehe Abschnitt 9.9.1. | `extract(hour from timestamp '2001-02-16 20:38:40')` -> `20` |
| `extract ( field from interval )` -> `numeric` | Holt ein Unterfeld aus einem Intervall. Siehe Abschnitt 9.9.1. | `extract(month from interval '2 years 3 months')` -> `3` |
| `isfinite ( date )` -> `boolean` | Prüft auf ein endliches Datum, also nicht `+/-infinity`. | `isfinite(date '2001-02-16')` -> `true` |
| `isfinite ( timestamp )` -> `boolean` | Prüft auf einen endlichen Zeitstempel, also nicht `+/-infinity`. | `isfinite(timestamp 'infinity')` -> `false` |
| `isfinite ( interval )` -> `boolean` | Prüft auf ein endliches Intervall, also nicht `+/-infinity`. | `isfinite(interval '4 hours')` -> `true` |
| `justify_days ( interval )` -> `interval` | Passt ein Intervall an und wandelt 30-Tage-Zeiträume in Monate um. | `justify_days(interval '1 year 65 days')` -> `1 year 2 mons 5 days` |
| `justify_hours ( interval )` -> `interval` | Passt ein Intervall an und wandelt 24-Stunden-Zeiträume in Tage um. | `justify_hours(interval '50 hours 10 minutes')` -> `2 days 02:10:00` |
| `justify_interval ( interval )` -> `interval` | Passt ein Intervall mit `justify_days` und `justify_hours` an, mit zusätzlichen Vorzeichenanpassungen. | `justify_interval(interval '1 mon -1 hour')` -> `29 days 23:00:00` |
| `localtime` -> `time` | Aktuelle Tageszeit. Siehe Abschnitt 9.9.5. | `localtime` -> `14:39:53.662522` |
| `localtime ( integer )` -> `time` | Aktuelle Tageszeit mit begrenzter Genauigkeit. Siehe Abschnitt 9.9.5. | `localtime(0)` -> `14:39:53` |
| `localtimestamp` -> `timestamp` | Aktuelles Datum und aktuelle Uhrzeit, bezogen auf den Start der aktuellen Transaktion. Siehe Abschnitt 9.9.5. | `localtimestamp` -> `2019-12-23 14:39:53.662522` |
| `localtimestamp ( integer )` -> `timestamp` | Aktuelles Datum und aktuelle Uhrzeit, bezogen auf den Start der aktuellen Transaktion, mit begrenzter Genauigkeit. Siehe Abschnitt 9.9.5. | `localtimestamp(2)` -> `2019-12-23 14:39:53.66` |
| `make_date ( year int, month int, day int )` -> `date` | Erzeugt ein Datum aus Jahr-, Monats- und Tagesfeldern; negative Jahre bedeuten v. Chr. | `make_date(2013, 7, 15)` -> `2013-07-15` |
| `make_interval ( [ years int [, months int [, weeks int [, days int [, hours int [, mins int [, secs double precision ]]]]]]] )` -> `interval` | Erzeugt ein Intervall aus Jahr-, Monats-, Wochen-, Tages-, Stunden-, Minuten- und Sekundenfeldern, die jeweils standardmäßig null sein können. | `make_interval(days => 10)` -> `10 days` |
| `make_time ( hour int, min int, sec double precision )` -> `time` | Erzeugt eine Uhrzeit aus Stunden-, Minuten- und Sekundenfeldern. | `make_time(8, 15, 23.5)` -> `08:15:23.5` |
| `make_timestamp ( year int, month int, day int, hour int, min int, sec double precision )` -> `timestamp` | Erzeugt einen Zeitstempel aus Jahr-, Monats-, Tages-, Stunden-, Minuten- und Sekundenfeldern; negative Jahre bedeuten v. Chr. | `make_timestamp(2013, 7, 15, 8, 15, 23.5)` -> `2013-07-15 08:15:23.5` |
| `make_timestamptz ( year int, month int, day int, hour int, min int, sec double precision [, timezone text ] )` -> `timestamp with time zone` | Erzeugt einen Zeitstempel mit Zeitzone aus Jahr-, Monats-, Tages-, Stunden-, Minuten- und Sekundenfeldern; negative Jahre bedeuten v. Chr. Wenn `timezone` nicht angegeben ist, wird die aktuelle Zeitzone verwendet; die Beispiele nehmen an, dass die Sitzungszeitzone `Europe/London` ist. | `make_timestamptz(2013, 7, 15, 8, 15, 23.5)` -> `2013-07-15 08:15:23.5+01`<br>`make_timestamptz(2013, 7, 15, 8, 15, 23.5, 'America/New_York')` -> `2013-07-15 13:15:23.5+01` |
| `now ( )` -> `timestamp with time zone` | Aktuelles Datum und aktuelle Uhrzeit, bezogen auf den Start der aktuellen Transaktion. Siehe Abschnitt 9.9.5. | `now()` -> `2019-12-23 14:39:53.662522-05` |
| `statement_timestamp ( )` -> `timestamp with time zone` | Aktuelles Datum und aktuelle Uhrzeit, bezogen auf den Start der aktuellen Anweisung. Siehe Abschnitt 9.9.5. | `statement_timestamp()` -> `2019-12-23 14:39:53.662522-05` |
| `timeofday ( )` -> `text` | Aktuelles Datum und aktuelle Uhrzeit, wie `clock_timestamp`, aber als Textzeichenkette. Siehe Abschnitt 9.9.5. | `timeofday()` -> `Mon Dec 23 14:39:53.662522 2019 EST` |
| `transaction_timestamp ( )` -> `timestamp with time zone` | Aktuelles Datum und aktuelle Uhrzeit, bezogen auf den Start der aktuellen Transaktion. Siehe Abschnitt 9.9.5. | `transaction_timestamp()` -> `2019-12-23 14:39:53.662522-05` |
| `to_timestamp ( double precision )` -> `timestamp with time zone` | Wandelt die Unix-Epoche, Sekunden seit `1970-01-01 00:00:00+00`, in einen Zeitstempel mit Zeitzone um. | `to_timestamp(1284352323)` -> `2010-09-13 04:32:03+00` |

Zusätzlich zu diesen Funktionen wird der SQL-Operator `OVERLAPS` unterstützt:

```sql
(start1, end1) OVERLAPS (start2, end2)
(start1, length1) OVERLAPS (start2, length2)
```

Dieser Ausdruck ergibt `true`, wenn zwei Zeiträume, definiert durch ihre Endpunkte, sich überlappen, und `false`, wenn sie sich nicht überlappen. Die Endpunkte können als Paare von Datumswerten, Uhrzeiten oder Zeitstempeln angegeben werden; oder als Datum, Uhrzeit oder Zeitstempel gefolgt von einem Intervall. Wenn ein Wertepaar angegeben wird, kann entweder der Anfang oder das Ende zuerst geschrieben werden; `OVERLAPS` verwendet automatisch den früheren Wert des Paars als Anfang. Jeder Zeitraum gilt als halboffenes Intervall `start <= time < end`, außer wenn `start` und `end` gleich sind; dann repräsentiert er diesen einzelnen Zeitpunkt. Das bedeutet zum Beispiel, dass sich zwei Zeiträume, die nur einen Endpunkt gemeinsam haben, nicht überlappen.

```sql
SELECT (DATE '2001-02-16', DATE '2001-12-21') OVERLAPS
       (DATE '2001-10-30', DATE '2002-10-30');
```

```text
Result: true
```

```sql
SELECT (DATE '2001-02-16', INTERVAL '100 days') OVERLAPS
       (DATE '2001-10-30', DATE '2002-10-30');
```

```text
Result: false
```

```sql
SELECT (DATE '2001-10-29', DATE '2001-10-30') OVERLAPS
       (DATE '2001-10-30', DATE '2001-10-31');
```

```text
Result: false
```

```sql
SELECT (DATE '2001-10-30', DATE '2001-10-30') OVERLAPS
       (DATE '2001-10-30', DATE '2001-10-31');
```

```text
Result: true
```

Wenn ein Intervallwert zu einem Wert vom Typ `timestamp` oder `timestamp with time zone` addiert oder davon subtrahiert wird, werden die Felder Monate, Tage und Mikrosekunden des Intervallwerts der Reihe nach behandelt. Zunächst verschiebt ein von null verschiedenes Monatsfeld das Datum des Zeitstempels um die angegebene Anzahl von Monaten vor oder zurück; der Tag des Monats bleibt gleich, außer wenn er nach dem Ende des neuen Monats läge, dann wird der letzte Tag dieses Monats verwendet. Zum Beispiel wird der 31. März plus 1 Monat zum 30. April, aber der 31. März plus 2 Monate zum 31. Mai. Danach verschiebt das Tagesfeld das Datum des Zeitstempels um die angegebene Anzahl von Tagen vor oder zurück. In beiden Schritten bleibt die lokale Tageszeit gleich. Schließlich wird ein von null verschiedenes Mikrosekundenfeld wörtlich addiert oder subtrahiert. Bei Arithmetik mit einem Wert vom Typ `timestamp with time zone` in einer Zeitzone, die Sommerzeit kennt, bedeutet das, dass das Addieren oder Subtrahieren von zum Beispiel `interval '1 day'` nicht unbedingt dasselbe Ergebnis liefert wie das Addieren oder Subtrahieren von `interval '24 hours'`. Zum Beispiel bei Sitzungszeitzone `America/Denver`:

```sql
SELECT timestamp with time zone '2005-04-02 12:00:00-07' + interval '1 day';
```

```text
Result: 2005-04-03 12:00:00-06
```

```sql
SELECT timestamp with time zone '2005-04-02 12:00:00-07' + interval '24 hours';
```

```text
Result: 2005-04-03 13:00:00-06
```

Das geschieht, weil durch den Wechsel zur Sommerzeit am `2005-04-03 02:00:00` in der Zeitzone `America/Denver` eine Stunde übersprungen wurde.

Beachten Sie, dass das von `age` zurückgegebene Monatsfeld mehrdeutig sein kann, weil verschiedene Monate unterschiedlich viele Tage haben. PostgreSQL verwendet bei der Berechnung von Teilmonaten den Monat des früheren der beiden Datumswerte. Zum Beispiel verwendet `age('2004-06-01', '2004-04-30')` den April und ergibt `1 mon 1 day`; würde Mai verwendet, ergäbe sich `1 mon 2 days`, weil der Mai 31 Tage hat, der April aber nur 30.

Auch die Subtraktion von Datumswerten und Zeitstempeln kann komplex sein. Eine konzeptionell einfache Möglichkeit zur Subtraktion besteht darin, jeden Wert mit `EXTRACT(EPOCH FROM ...)` in eine Anzahl von Sekunden umzuwandeln und dann die Ergebnisse zu subtrahieren; das ergibt die Anzahl von Sekunden zwischen den beiden Werten. Dabei werden die Anzahl der Tage in jedem Monat, Zeitzonenänderungen und Sommerzeit-Anpassungen berücksichtigt. Die Subtraktion von `date`- oder `timestamp`-Werten mit dem Operator `-` gibt die Anzahl der Tage (zu 24 Stunden) und Stunden/Minuten/Sekunden zwischen den Werten zurück und nimmt dieselben Anpassungen vor. Die Funktion `age` gibt Jahre, Monate, Tage und Stunden/Minuten/Sekunden zurück, führt eine feldweise Subtraktion aus und passt anschließend negative Feldwerte an. Die folgenden Abfragen veranschaulichen die Unterschiede zwischen diesen Ansätzen. Die Beispielergebnisse wurden mit `timezone = 'US/Eastern'` erzeugt; zwischen den beiden verwendeten Datumswerten liegt ein Sommerzeitwechsel:

```sql
SELECT EXTRACT(EPOCH FROM timestamptz '2013-07-01 12:00:00') -
       EXTRACT(EPOCH FROM timestamptz '2013-03-01 12:00:00');
```

```text
Result: 10537200.000000
```

```sql
SELECT (EXTRACT(EPOCH FROM timestamptz '2013-07-01 12:00:00') -
        EXTRACT(EPOCH FROM timestamptz '2013-03-01 12:00:00'))
        / 60 / 60 / 24;
```

```text
Result: 121.9583333333333333
```

```sql
SELECT timestamptz '2013-07-01 12:00:00' - timestamptz '2013-03-01 12:00:00';
```

```text
Result: 121 days 23:00:00
```

```sql
SELECT age(timestamptz '2013-07-01 12:00:00', timestamptz '2013-03-01 12:00:00');
```

```text
Result: 4 mons
```

### 9.9.1. `EXTRACT`, `date_part`

```sql
EXTRACT(field FROM source)
```

Die Funktion `extract` ruft Unterfelder wie Jahr oder Stunde aus Datums-/Zeitwerten ab. `source` muss ein Wertausdruck des Typs `timestamp`, `date`, `time` oder `interval` sein. Zeitstempel und Uhrzeiten können mit oder ohne Zeitzone vorliegen. `field` ist ein Bezeichner oder eine Zeichenkette, die auswählt, welches Feld aus dem Quellwert extrahiert werden soll. Nicht alle Felder sind für jeden Eingabedatentyp gültig; zum Beispiel können Felder kleiner als ein Tag nicht aus einem `date` extrahiert werden, während Felder von einem Tag oder größer nicht aus einer `time` extrahiert werden können. Die Funktion `extract` gibt Werte des Typs `numeric` zurück.

Die folgenden Feldnamen sind gültig:

`century`

Das Jahrhundert; bei Intervallwerten das Jahresfeld dividiert durch 100.

```sql
SELECT EXTRACT(CENTURY FROM TIMESTAMP '2000-12-16 12:21:13');
SELECT EXTRACT(CENTURY FROM TIMESTAMP '2001-02-16 20:38:40');
SELECT EXTRACT(CENTURY FROM DATE '0001-01-01 AD');
SELECT EXTRACT(CENTURY FROM DATE '0001-12-31 BC');
SELECT EXTRACT(CENTURY FROM INTERVAL '2001 years');
```

```text
Result: 20
Result: 21
Result: 1
Result: -1
Result: 20
```

`day`

Der Tag des Monats (1-31); bei Intervallwerten die Anzahl der Tage.

```sql
SELECT EXTRACT(DAY FROM TIMESTAMP '2001-02-16 20:38:40');
SELECT EXTRACT(DAY FROM INTERVAL '40 days 1 minute');
```

```text
Result: 16
Result: 40
```

`decade`

Das Jahresfeld dividiert durch 10.

```sql
SELECT EXTRACT(DECADE FROM TIMESTAMP '2001-02-16 20:38:40');
```

```text
Result: 200
```

`dow`

Der Wochentag als Sonntag (0) bis Samstag (6).

```sql
SELECT EXTRACT(DOW FROM TIMESTAMP '2001-02-16 20:38:40');
```

```text
Result: 5
```

Beachten Sie, dass die Wochentagsnummerierung von `extract` von der Funktion `to_char(..., 'D')` abweicht.

`doy`

Der Tag des Jahres (1-365/366).

```sql
SELECT EXTRACT(DOY FROM TIMESTAMP '2001-02-16 20:38:40');
```

```text
Result: 47
```

`epoch`

Bei Werten vom Typ `timestamp with time zone` die Anzahl der Sekunden seit `1970-01-01 00:00:00 UTC`, negativ für Zeitstempel davor; bei `date`- und `timestamp`-Werten die nominelle Anzahl von Sekunden seit `1970-01-01 00:00:00` ohne Berücksichtigung von Zeitzonen- oder Sommerzeitregeln; bei `interval`-Werten die Gesamtzahl der Sekunden im Intervall.

```sql
SELECT EXTRACT(EPOCH FROM TIMESTAMP WITH TIME ZONE '2001-02-16 20:38:40.12-08');
SELECT EXTRACT(EPOCH FROM TIMESTAMP '2001-02-16 20:38:40.12');
SELECT EXTRACT(EPOCH FROM INTERVAL '5 days 3 hours');
```

```text
Result: 982384720.120000
Result: 982355920.120000
Result: 442800.000000
```

Sie können einen Epoch-Wert mit `to_timestamp` wieder in einen Zeitstempel mit Zeitzone umwandeln:

```sql
SELECT to_timestamp(982384720.12);
```

```text
Result: 2001-02-17 04:38:40.12+00
```

Seien Sie vorsichtig, wenn Sie `to_timestamp` auf eine Epoche anwenden, die aus einem `date`- oder `timestamp`-Wert extrahiert wurde: Das Ergebnis nimmt effektiv an, dass der ursprüngliche Wert in UTC angegeben war, was nicht der Fall gewesen sein muss.

`hour`

Das Stundenfeld (0-23 in Zeitstempeln, unbegrenzt in Intervallen).

```sql
SELECT EXTRACT(HOUR FROM TIMESTAMP '2001-02-16 20:38:40');
```

```text
Result: 20
```

`isodow`

Der Wochentag als Montag (1) bis Sonntag (7).

```sql
SELECT EXTRACT(ISODOW FROM TIMESTAMP '2001-02-18 20:38:40');
```

```text
Result: 7
```

Dies ist identisch mit `dow`, außer für Sonntag. Es entspricht der ISO-8601-Wochentagsnummerierung.

`isoyear`

Das ISO-8601-Wochennummerierungsjahr, in das das Datum fällt.

```sql
SELECT EXTRACT(ISOYEAR FROM DATE '2006-01-01');
SELECT EXTRACT(ISOYEAR FROM DATE '2006-01-02');
```

```text
Result: 2005
Result: 2006
```

Jedes ISO-8601-Wochennummerierungsjahr beginnt mit dem Montag der Woche, die den 4. Januar enthält; daher kann das ISO-Jahr Anfang Januar oder Ende Dezember vom gregorianischen Jahr abweichen. Weitere Informationen finden Sie beim Feld `week`.

`julian`

Das julianische Datum, das dem Datum oder Zeitstempel entspricht. Zeitstempel, die nicht lokale Mitternacht sind, ergeben einen Bruchwert. Weitere Informationen finden Sie in Abschnitt B.7.

```sql
SELECT EXTRACT(JULIAN FROM DATE '2006-01-01');
SELECT EXTRACT(JULIAN FROM TIMESTAMP '2006-01-01 12:00');
```

```text
Result: 2453737
Result: 2453737.50000000000000000000
```

`microseconds`

Das Sekundenfeld einschließlich Bruchteilen, multipliziert mit 1 000 000; beachten Sie, dass volle Sekunden eingeschlossen sind.

```sql
SELECT EXTRACT(MICROSECONDS FROM TIME '17:12:28.5');
```

```text
Result: 28500000
```

`millennium`

Das Jahrtausend; bei Intervallwerten das Jahresfeld dividiert durch 1000.

```sql
SELECT EXTRACT(MILLENNIUM FROM TIMESTAMP '2001-02-16 20:38:40');
SELECT EXTRACT(MILLENNIUM FROM INTERVAL '2001 years');
```

```text
Result: 3
Result: 2
```

Jahre in den 1900ern liegen im zweiten Jahrtausend. Das dritte Jahrtausend begann am 1. Januar 2001.

`milliseconds`

Das Sekundenfeld einschließlich Bruchteilen, multipliziert mit 1000. Beachten Sie, dass volle Sekunden eingeschlossen sind.

```sql
SELECT EXTRACT(MILLISECONDS FROM TIME '17:12:28.5');
```

```text
Result: 28500.000
```

`minute`

Das Minutenfeld (0-59).

```sql
SELECT EXTRACT(MINUTE FROM TIMESTAMP '2001-02-16 20:38:40');
```

```text
Result: 38
```

`month`

Die Nummer des Monats innerhalb des Jahres (1-12); bei Intervallwerten die Anzahl der Monate modulo 12 (0-11).

```sql
SELECT EXTRACT(MONTH FROM TIMESTAMP '2001-02-16 20:38:40');
SELECT EXTRACT(MONTH FROM INTERVAL '2 years 3 months');
SELECT EXTRACT(MONTH FROM INTERVAL '2 years 13 months');
```

```text
Result: 2
Result: 3
Result: 1
```

`quarter`

Das Quartal des Jahres (1-4), in dem das Datum liegt; bei Intervallwerten das Monatsfeld dividiert durch 3 plus 1.

```sql
SELECT EXTRACT(QUARTER FROM TIMESTAMP '2001-02-16 20:38:40');
SELECT EXTRACT(QUARTER FROM INTERVAL '1 year 6 months');
```

```text
Result: 1
Result: 3
```

`second`

Das Sekundenfeld einschließlich beliebiger Sekundenbruchteile.

```sql
SELECT EXTRACT(SECOND FROM TIMESTAMP '2001-02-16 20:38:40');
SELECT EXTRACT(SECOND FROM TIME '17:12:28.5');
```

```text
Result: 40.000000
Result: 28.500000
```

`timezone`

Der Zeitzonenversatz gegenüber UTC, gemessen in Sekunden. Positive Werte entsprechen Zeitzonen östlich von UTC, negative Werten Zonen westlich von UTC. Technisch verwendet PostgreSQL nicht UTC, weil Schaltsekunden nicht behandelt werden.

`timezone_hour`

Die Stundenkomponente des Zeitzonenversatzes.

`timezone_minute`

Die Minutenkomponente des Zeitzonenversatzes.

`week`

Die Nummer der ISO-8601-Wochennummerierungswoche des Jahres. Per Definition beginnen ISO-Wochen am Montag, und die erste Woche eines Jahres enthält den 4. Januar dieses Jahres. Anders gesagt: Der erste Donnerstag eines Jahres liegt in Woche 1 dieses Jahres.

Im ISO-Wochennummerierungssystem können Datumswerte Anfang Januar Teil der 52. oder 53. Woche des vorherigen Jahres sein, und Datumswerte Ende Dezember können Teil der ersten Woche des nächsten Jahres sein. Zum Beispiel ist `2005-01-01` Teil der 53. Woche des Jahres 2004, und `2006-01-01` ist Teil der 52. Woche des Jahres 2005, während `2012-12-31` Teil der ersten Woche von 2013 ist. Es wird empfohlen, das Feld `isoyear` zusammen mit `week` zu verwenden, um konsistente Ergebnisse zu erhalten.

Bei Intervallwerten ist das Feld `week` einfach die Anzahl ganzer Tage dividiert durch 7.

```sql
SELECT EXTRACT(WEEK FROM TIMESTAMP '2001-02-16 20:38:40');
SELECT EXTRACT(WEEK FROM INTERVAL '13 days 24 hours');
```

```text
Result: 7
Result: 1
```

`year`

Das Jahresfeld. Denken Sie daran, dass es kein Jahr 0 n. Chr. gibt; das Subtrahieren von v.-Chr.-Jahren von n.-Chr.-Jahren sollte daher mit Vorsicht erfolgen.

```sql
SELECT EXTRACT(YEAR FROM TIMESTAMP '2001-02-16 20:38:40');
```

```text
Result: 2001
```

Beim Verarbeiten eines Intervallwerts erzeugt die Funktion `extract` Feldwerte, die der Interpretation der Intervall-Ausgabefunktion entsprechen. Das kann überraschende Ergebnisse erzeugen, wenn man mit einer nicht normalisierten Intervalldarstellung beginnt, zum Beispiel:

```sql
SELECT INTERVAL '80 minutes';
SELECT EXTRACT(MINUTES FROM INTERVAL '80 minutes');
```

```text
Result: 01:20:00
Result: 20
```

> Hinweis: Wenn der Eingabewert `+/-Infinity` ist, gibt `extract` für monoton steigende Felder `+/-Infinity` zurück (`epoch`, `julian`, `year`, `isoyear`, `decade`, `century` und `millennium` bei `timestamp`-Eingaben; `epoch`, `hour`, `day`, `year`, `decade`, `century` und `millennium` bei `interval`-Eingaben). Für andere Felder wird `NULL` zurückgegeben. PostgreSQL-Versionen vor 9.6 gaben für alle Fälle unendlicher Eingaben null zurück.

Die Funktion `extract` ist hauptsächlich für rechnerische Verarbeitung gedacht. Zum Formatieren von Datums-/Zeitwerten für die Anzeige siehe [Abschnitt 9.8](09_Funktionen_und_Operatoren.md#98-formatierungsfunktionen-für-datentypen).

Die Funktion `date_part` ist dem traditionellen Ingres-Äquivalent zur SQL-Standardfunktion `extract` nachempfunden:

```sql
date_part('field', source)
```

Beachten Sie, dass der Parameter `field` hier ein Zeichenkettenwert sein muss, kein Name. Die gültigen Feldnamen für `date_part` sind dieselben wie für `extract`. Aus historischen Gründen gibt `date_part` Werte des Typs `double precision` zurück. Das kann in bestimmten Verwendungen zu Präzisionsverlust führen. Es wird empfohlen, stattdessen `extract` zu verwenden.

```sql
SELECT date_part('day', TIMESTAMP '2001-02-16 20:38:40');
SELECT date_part('hour', INTERVAL '4 hours 3 minutes');
```

```text
Result: 16
Result: 4
```

### 9.9.2. `date_trunc`

Die Funktion `date_trunc` ähnelt konzeptionell der Funktion `trunc` für Zahlen.

```sql
date_trunc(field, source [, time_zone ])
```

`source` ist ein Wertausdruck des Typs `timestamp`, `timestamp with time zone` oder `interval`. Werte der Typen `date` und `time` werden automatisch in `timestamp` beziehungsweise `interval` gecastet. `field` wählt aus, auf welche Genauigkeit der Eingabewert abgeschnitten wird. Der Rückgabewert ist entsprechend vom Typ `timestamp`, `timestamp with time zone` oder `interval`, und alle Felder, die weniger signifikant als das gewählte Feld sind, werden auf null gesetzt, beziehungsweise bei Tag und Monat auf eins.

Gültige Werte für `field` sind:

```text
microseconds
milliseconds
second
minute
hour
day
week
month
quarter
year
decade
century
millennium
```

Wenn der Eingabewert vom Typ `timestamp with time zone` ist, wird das Abschneiden bezogen auf eine bestimmte Zeitzone ausgeführt; das Abschneiden auf `day` erzeugt zum Beispiel einen Wert, der Mitternacht in dieser Zone ist. Standardmäßig erfolgt das Abschneiden bezogen auf die aktuelle Einstellung `TimeZone`, aber das optionale Argument `time_zone` kann angegeben werden, um eine andere Zeitzone festzulegen. Der Zeitzonenname kann auf jede der in Abschnitt 8.5.3 beschriebenen Arten angegeben werden.

Bei Eingaben vom Typ `timestamp without time zone` oder `interval` kann keine Zeitzone angegeben werden. Diese Werte werden immer wörtlich genommen.

Beispiele, unter der Annahme, dass die lokale Zeitzone `America/New_York` ist:

```sql
SELECT date_trunc('hour', TIMESTAMP '2001-02-16 20:38:40');
SELECT date_trunc('year', TIMESTAMP '2001-02-16 20:38:40');
SELECT date_trunc('day', TIMESTAMP WITH TIME ZONE '2001-02-16 20:38:40+00');
SELECT date_trunc('day', TIMESTAMP WITH TIME ZONE '2001-02-16 20:38:40+00', 'Australia/Sydney');
SELECT date_trunc('hour', INTERVAL '3 days 02:47:33');
```

```text
Result: 2001-02-16 20:00:00
Result: 2001-01-01 00:00:00
Result: 2001-02-16 00:00:00-05
Result: 2001-02-16 08:00:00-05
Result: 3 days 02:00:00
```

### 9.9.3. `date_bin`

Die Funktion `date_bin` ordnet den Eingabezeitstempel einem angegebenen Intervall, dem Schrittmaß, zu, ausgerichtet an einem angegebenen Ursprung.

```sql
date_bin(stride, source, origin)
```

`source` ist ein Wertausdruck des Typs `timestamp` oder `timestamp with time zone`. Werte vom Typ `date` werden automatisch in `timestamp` gecastet. `stride` ist ein Wertausdruck des Typs `interval`.

Der Rückgabewert ist ebenfalls vom Typ `timestamp` oder `timestamp with time zone` und markiert den Anfang des Zeitfensters, in das `source` fällt.

Beispiele:

```sql
SELECT date_bin('15 minutes', TIMESTAMP '2020-02-11 15:44:17',
                TIMESTAMP '2001-01-01');
SELECT date_bin('15 minutes', TIMESTAMP '2020-02-11 15:44:17',
                TIMESTAMP '2001-01-01 00:02:30');
```

```text
Result: 2020-02-11 15:30:00
Result: 2020-02-11 15:32:30
```

Bei vollen Einheiten (1 Minute, 1 Stunde usw.) liefert dies dasselbe Ergebnis wie der entsprechende Aufruf von `date_trunc`; der Unterschied besteht darin, dass `date_bin` auf ein beliebiges Intervall kürzen kann.

Das Intervall `stride` muss größer als null sein und darf keine Einheiten von Monat oder größer enthalten.

### 9.9.4. `AT TIME ZONE` und `AT LOCAL`

Der Operator `AT TIME ZONE` wandelt Zeitstempel ohne Zeitzone in Zeitstempel mit Zeitzone und umgekehrt um und wandelt Werte vom Typ `time with time zone` in andere Zeitzonen um. Tabelle 9.34 zeigt seine Varianten.

**Tabelle 9.34. Varianten von `AT TIME ZONE` und `AT LOCAL`**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `timestamp without time zone AT TIME ZONE zone` -> `timestamp with time zone` | Wandelt den angegebenen Zeitstempel ohne Zeitzone in einen Zeitstempel mit Zeitzone um, wobei angenommen wird, dass der angegebene Wert in der benannten Zeitzone liegt. | `timestamp '2001-02-16 20:38:40' at time zone 'America/Denver'` -> `2001-02-17 03:38:40+00` |
| `timestamp without time zone AT LOCAL` -> `timestamp with time zone` | Wandelt den angegebenen Zeitstempel ohne Zeitzone in einen Zeitstempel mit dem `TimeZone`-Wert der Sitzung als Zeitzone um. | `timestamp '2001-02-16 20:38:40' at local` -> `2001-02-17 03:38:40+00` |
| `timestamp with time zone AT TIME ZONE zone` -> `timestamp without time zone` | Wandelt den angegebenen Zeitstempel mit Zeitzone in einen Zeitstempel ohne Zeitzone um, so wie die Zeit in dieser Zone erscheinen würde. | `timestamp with time zone '2001-02-16 20:38:40-05' at time zone 'America/Denver'` -> `2001-02-16 18:38:40` |
| `timestamp with time zone AT LOCAL` -> `timestamp without time zone` | Wandelt den angegebenen Zeitstempel mit Zeitzone in einen Zeitstempel ohne Zeitzone um, so wie die Zeit mit dem `TimeZone`-Wert der Sitzung als Zeitzone erscheinen würde. | `timestamp with time zone '2001-02-16 20:38:40-05' at local` -> `2001-02-16 18:38:40` |
| `time with time zone AT TIME ZONE zone` -> `time with time zone` | Wandelt die angegebene Uhrzeit mit Zeitzone in eine neue Zeitzone um. Da kein Datum angegeben ist, wird der aktuell aktive UTC-Versatz für die benannte Zielzone verwendet. | `time with time zone '05:34:17-05' at time zone 'UTC'` -> `10:34:17+00` |
| `time with time zone AT LOCAL` -> `time with time zone` | Wandelt die angegebene Uhrzeit mit Zeitzone in eine neue Zeitzone um. Da kein Datum angegeben ist, wird der aktuell aktive UTC-Versatz für den `TimeZone`-Wert der Sitzung verwendet. Unter der Annahme, dass die Sitzungszeitzone `UTC` ist. | `time with time zone '05:34:17-05' at local` -> `10:34:17+00` |

In diesen Ausdrücken kann die gewünschte Zeitzone `zone` entweder als Textwert, zum Beispiel `'America/Los_Angeles'`, oder als Intervall, zum Beispiel `INTERVAL '-08:00'`, angegeben werden. Im Textfall kann ein Zeitzonenname auf jede der in Abschnitt 8.5.3 beschriebenen Arten angegeben werden. Der Intervallfall ist nur für Zonen mit festem Versatz gegenüber UTC nützlich und daher in der Praxis nicht sehr verbreitet.

Die Syntax `AT LOCAL` kann als Kurzschreibweise für `AT TIME ZONE local` verwendet werden, wobei `local` der `TimeZone`-Wert der Sitzung ist.

Beispiele unter der Annahme, dass die aktuelle Einstellung `TimeZone` `America/Los_Angeles` ist:

```sql
SELECT TIMESTAMP '2001-02-16 20:38:40' AT TIME ZONE 'America/Denver';
SELECT TIMESTAMP WITH TIME ZONE '2001-02-16 20:38:40-05' AT TIME ZONE 'America/Denver';
SELECT TIMESTAMP '2001-02-16 20:38:40' AT TIME ZONE 'Asia/Tokyo' AT TIME ZONE 'America/Chicago';
SELECT TIMESTAMP WITH TIME ZONE '2001-02-16 20:38:40-05' AT LOCAL;
SELECT TIMESTAMP WITH TIME ZONE '2001-02-16 20:38:40-05' AT TIME ZONE '+05';
SELECT TIME WITH TIME ZONE '20:38:40-05' AT LOCAL;
```

```text
Result: 2001-02-16 19:38:40-08
Result: 2001-02-16 18:38:40
Result: 2001-02-16 05:38:40
Result: 2001-02-16 17:38:40
Result: 2001-02-16 20:38:40
Result: 17:38:40
```

Das erste Beispiel fügt einem Wert, der keine Zeitzone besitzt, eine Zeitzone hinzu und zeigt den Wert mit der aktuellen Einstellung `TimeZone` an. Das zweite Beispiel verschiebt den Wert vom Typ `timestamp with time zone` in die angegebene Zeitzone und gibt den Wert ohne Zeitzone zurück. Dadurch können Werte gespeichert und angezeigt werden, die von der aktuellen Einstellung `TimeZone` abweichen. Das dritte Beispiel wandelt Tokio-Zeit in Chicago-Zeit um. Das vierte Beispiel verschiebt den Wert vom Typ `timestamp with time zone` in die aktuell durch `TimeZone` angegebene Zeitzone und gibt den Wert ohne Zeitzone zurück. Das fünfte Beispiel zeigt, dass das Vorzeichen in einer Zeitzonenspezifikation im POSIX-Stil die entgegengesetzte Bedeutung zum Vorzeichen in einem ISO-8601-Datums-/Zeitliteral hat, wie in Abschnitt 8.5.3 und Anhang B beschrieben.

Das sechste Beispiel ist ein warnendes Beispiel. Da dem Eingabewert kein Datum zugeordnet ist, wird die Umwandlung mit dem aktuellen Datum der Sitzung durchgeführt. Daher kann dieses statische Beispiel je nach Jahreszeit, in der es betrachtet wird, ein falsches Ergebnis zeigen, weil `'America/Los_Angeles'` Sommerzeit verwendet.

Die Funktion `timezone(zone, timestamp)` ist äquivalent zum SQL-konformen Konstrukt `timestamp AT TIME ZONE zone`.

Die Funktion `timezone(zone, time)` ist äquivalent zum SQL-konformen Konstrukt `time AT TIME ZONE zone`.

Die Funktion `timezone(timestamp)` ist äquivalent zum SQL-konformen Konstrukt `timestamp AT LOCAL`.

Die Funktion `timezone(time)` ist äquivalent zum SQL-konformen Konstrukt `time AT LOCAL`.

### 9.9.5. Aktuelles Datum/aktuelle Uhrzeit

PostgreSQL stellt eine Reihe von Funktionen bereit, die Werte zum aktuellen Datum und zur aktuellen Uhrzeit zurückgeben. Diese SQL-Standardfunktionen geben alle Werte zurück, die auf der Startzeit der aktuellen Transaktion basieren:

```sql
CURRENT_DATE
CURRENT_TIME
CURRENT_TIMESTAMP
CURRENT_TIME(precision)
CURRENT_TIMESTAMP(precision)
LOCALTIME
LOCALTIMESTAMP
LOCALTIME(precision)
LOCALTIMESTAMP(precision)
```

`CURRENT_TIME` und `CURRENT_TIMESTAMP` liefern Werte mit Zeitzone; `LOCALTIME` und `LOCALTIMESTAMP` liefern Werte ohne Zeitzone.

`CURRENT_TIME`, `CURRENT_TIMESTAMP`, `LOCALTIME` und `LOCALTIMESTAMP` können optional einen Genauigkeitsparameter annehmen, der bewirkt, dass das Ergebnis auf diese Anzahl von Nachkommastellen im Sekundenfeld gerundet wird. Ohne Genauigkeitsparameter wird das Ergebnis mit der vollen verfügbaren Genauigkeit angegeben.

Einige Beispiele:

```sql
SELECT CURRENT_TIME;
SELECT CURRENT_DATE;
SELECT CURRENT_TIMESTAMP;
SELECT CURRENT_TIMESTAMP(2);
SELECT LOCALTIMESTAMP;
```

```text
Result: 14:39:53.662522-05
Result: 2019-12-23
Result: 2019-12-23 14:39:53.662522-05
Result: 2019-12-23 14:39:53.66-05
Result: 2019-12-23 14:39:53.662522
```

Da diese Funktionen die Startzeit der aktuellen Transaktion zurückgeben, ändern sich ihre Werte während der Transaktion nicht. Das gilt als Feature: Die Absicht ist, einer einzelnen Transaktion eine konsistente Vorstellung der „aktuellen“ Zeit zu geben, sodass mehrere Änderungen innerhalb derselben Transaktion denselben Zeitstempel tragen.

> Hinweis: Andere Datenbanksysteme aktualisieren diese Werte möglicherweise häufiger.

PostgreSQL stellt außerdem Funktionen bereit, die die Startzeit der aktuellen Anweisung sowie die tatsächliche aktuelle Zeit zum Zeitpunkt des Funktionsaufrufs zurückgeben. Die vollständige Liste der nicht zum SQL-Standard gehörenden Zeitfunktionen lautet:

```sql
transaction_timestamp()
statement_timestamp()
clock_timestamp()
timeofday()
now()
```

`transaction_timestamp()` ist äquivalent zu `CURRENT_TIMESTAMP`, ist aber so benannt, dass deutlich wird, was zurückgegeben wird. `statement_timestamp()` gibt die Startzeit der aktuellen Anweisung zurück, genauer den Zeitpunkt des Empfangs der neuesten Befehlsnachricht vom Client. `statement_timestamp()` und `transaction_timestamp()` geben während der ersten Anweisung einer Transaktion denselben Wert zurück, können sich aber bei folgenden Anweisungen unterscheiden. `clock_timestamp()` gibt die tatsächliche aktuelle Zeit zurück; sein Wert ändert sich daher sogar innerhalb einer einzelnen SQL-Anweisung. `timeofday()` ist eine historische PostgreSQL-Funktion. Wie `clock_timestamp()` gibt sie die tatsächliche aktuelle Zeit zurück, aber als formatierte Textzeichenkette statt als Wert vom Typ `timestamp with time zone`. `now()` ist ein traditionelles PostgreSQL-Äquivalent zu `transaction_timestamp()`.

Alle Datums-/Zeitdatentypen akzeptieren außerdem den besonderen Literalwert `now`, um aktuelles Datum und aktuelle Uhrzeit anzugeben, wiederum als Transaktionsstartzeit interpretiert. Daher geben die folgenden drei Formen alle dasselbe Ergebnis zurück:

```sql
SELECT CURRENT_TIMESTAMP;
SELECT now();
SELECT TIMESTAMP 'now'; -- siehe aber den Tipp unten
```

> Tipp: Verwenden Sie die dritte Form nicht, wenn ein Wert angegeben wird, der später ausgewertet werden soll, zum Beispiel in einer `DEFAULT`-Klausel für eine Tabellenspalte. Das System wandelt `now` in einen Zeitstempel um, sobald die Konstante geparst wird; wenn der Vorgabewert später benötigt wird, würde also die Zeit der Tabellenerstellung verwendet. Die ersten beiden Formen werden erst ausgewertet, wenn der Vorgabewert verwendet wird, weil sie Funktionsaufrufe sind. Dadurch liefern sie das gewünschte Verhalten, nämlich standardmäßig den Zeitpunkt des Einfügens der Zeile zu verwenden. Siehe auch Abschnitt 8.5.1.4.

### 9.9.6. Ausführung verzögern

Die folgenden Funktionen stehen zur Verfügung, um die Ausführung des Serverprozesses zu verzögern:

```sql
pg_sleep ( double precision )
pg_sleep_for ( interval )
pg_sleep_until ( timestamp with time zone )
```

`pg_sleep` lässt den Prozess der aktuellen Sitzung schlafen, bis die angegebene Anzahl von Sekunden verstrichen ist. Verzögerungen mit Sekundenbruchteilen können angegeben werden. `pg_sleep_for` ist eine Komfortfunktion, mit der die Schlafzeit als Intervall angegeben werden kann. `pg_sleep_until` ist eine Komfortfunktion für Fälle, in denen eine bestimmte Aufwachzeit gewünscht ist. Zum Beispiel:

```sql
SELECT pg_sleep(1.5);
SELECT pg_sleep_for('5 minutes');
SELECT pg_sleep_until('tomorrow 03:00');
```

> Hinweis: Die effektive Auflösung des Schlafintervalls ist plattformspezifisch; 0,01 Sekunden ist ein häufiger Wert. Die Schlafverzögerung ist mindestens so lang wie angegeben. Abhängig von Faktoren wie Serverlast kann sie länger sein. Insbesondere garantiert `pg_sleep_until` nicht, exakt zur angegebenen Zeit aufzuwachen, aber es wacht nicht früher auf.

> Warnung: Stellen Sie sicher, dass Ihre Sitzung beim Aufruf von `pg_sleep` oder seinen Varianten nicht mehr Sperren hält als nötig. Andernfalls müssten andere Sitzungen möglicherweise auf Ihren schlafenden Prozess warten, wodurch das gesamte System verlangsamt wird.

## 9.10. Enum-Unterstützungsfunktionen

Für Enum-Typen, die in [Abschnitt 8.7](08_Datentypen.md#87-aufzählungstypen) beschrieben werden, gibt es mehrere Funktionen, die sauberere Programmierung ermöglichen, ohne bestimmte Werte eines Enum-Typs fest zu kodieren. Diese Funktionen sind in Tabelle 9.35 aufgeführt. Die Beispiele setzen einen Enum-Typ voraus, der so erstellt wurde:

```sql
CREATE TYPE rainbow AS ENUM ('red', 'orange', 'yellow', 'green',
 'blue', 'purple');
```

**Tabelle 9.35. Enum-Unterstützungsfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `enum_first ( anyenum )` -> `anyenum` | Gibt den ersten Wert des eingegebenen Enum-Typs zurück. | `enum_first(null::rainbow)` -> `red` |
| `enum_last ( anyenum )` -> `anyenum` | Gibt den letzten Wert des eingegebenen Enum-Typs zurück. | `enum_last(null::rainbow)` -> `purple` |
| `enum_range ( anyenum )` -> `anyarray` | Gibt alle Werte des eingegebenen Enum-Typs als geordnetes Array zurück. | `enum_range(null::rainbow)` -> `{red,orange,yellow,green,blue,purple}` |
| `enum_range ( anyenum, anyenum )` -> `anyarray` | Gibt den Bereich zwischen den beiden angegebenen Enum-Werten als geordnetes Array zurück. Die Werte müssen aus demselben Enum-Typ stammen. Wenn der erste Parameter `null` ist, beginnt das Ergebnis mit dem ersten Wert des Enum-Typs. Wenn der zweite Parameter `null` ist, endet das Ergebnis mit dem letzten Wert des Enum-Typs. | `enum_range('orange'::rainbow, 'green'::rainbow)` -> `{orange,yellow,green}`<br>`enum_range(NULL, 'green'::rainbow)` -> `{red,orange,yellow,green}`<br>`enum_range('orange'::rainbow, NULL)` -> `{orange,yellow,green,blue,purple}` |

Beachten Sie, dass diese Funktionen, mit Ausnahme der zweistelligen Form von `enum_range`, den konkreten Wert ignorieren, der ihnen übergeben wird; für sie zählt nur dessen deklarierter Datentyp. Es kann entweder `null` oder ein konkreter Wert des Typs übergeben werden, mit demselben Ergebnis. Häufiger werden diese Funktionen auf eine Tabellenspalte oder ein Funktionsargument angewendet als auf einen fest eingetragenen Typnamen wie in den Beispielen.

## 9.11. Geometrische Funktionen und Operatoren

Die geometrischen Typen `point`, `box`, `lseg`, `line`, `path`, `polygon` und `circle` besitzen einen großen Satz nativer Unterstützungsfunktionen und -operatoren, die in Tabelle 9.36, Tabelle 9.37 und Tabelle 9.38 gezeigt werden.

**Tabelle 9.36. Geometrische Operatoren**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `geometric_type + point` -> `geometric_type` | Addiert die Koordinaten des zweiten Punkts zu denen jedes Punkts des ersten Arguments und führt damit eine Verschiebung aus. Verfügbar für `point`, `box`, `path`, `circle`. | `box '(1,1),(0,0)' + point '(2,0)'` -> `(3,1),(2,0)` |
| `path + path` -> `path` | Verkettet zwei offene Pfade; gibt `NULL` zurück, wenn einer der Pfade geschlossen ist. | `path '[(0,0),(1,1)]' + path '[(2,2),(3,3),(4,4)]'` -> `[(0,0),(1,1),(2,2),(3,3),(4,4)]` |
| `geometric_type - point` -> `geometric_type` | Subtrahiert die Koordinaten des zweiten Punkts von denen jedes Punkts des ersten Arguments und führt damit eine Verschiebung aus. Verfügbar für `point`, `box`, `path`, `circle`. | `box '(1,1),(0,0)' - point '(2,0)'` -> `(-1,1),(-2,0)` |
| `geometric_type * point` -> `geometric_type` | Multipliziert jeden Punkt des ersten Arguments mit dem zweiten Punkt; ein Punkt wird dabei als komplexe Zahl mit Real- und Imaginärteil behandelt, und es wird eine normale komplexe Multiplikation ausgeführt. Interpretiert man den zweiten Punkt als Vektor, entspricht dies einer Skalierung von Größe und Abstand des Objekts vom Ursprung um die Länge des Vektors und einer Drehung gegen den Uhrzeigersinn um den Ursprung um den Winkel des Vektors zur x-Achse. Verfügbar für `point`, `box`, `path`, `circle`. | `path '((0,0),(1,0),(1,1))' * point '(3.0,0)'` -> `((0,0),(3,0),(3,3))`<br>`path '((0,0),(1,0),(1,1))' * point(cosd(45), sind(45))` -> `((0,0),(0.7071067811865475,0.7071067811865475),(0,1.414213562373095))` |
| `geometric_type / point` -> `geometric_type` | Dividiert jeden Punkt des ersten Arguments durch den zweiten Punkt; ein Punkt wird dabei als komplexe Zahl mit Real- und Imaginärteil behandelt, und es wird eine normale komplexe Division ausgeführt. Interpretiert man den zweiten Punkt als Vektor, entspricht dies einer Herunterskalierung von Größe und Abstand des Objekts vom Ursprung um die Länge des Vektors und einer Drehung im Uhrzeigersinn um den Ursprung um den Winkel des Vektors zur x-Achse. Verfügbar für `point`, `box`, `path`, `circle`. | `path '((0,0),(1,0),(1,1))' / point '(2.0,0)'` -> `((0,0),(0.5,0),(0.5,0.5))`<br>`path '((0,0),(1,0),(1,1))' / point(cosd(45), sind(45))` -> `((0,0),(0.7071067811865476,-0.7071067811865476),(1.4142135623730951,0))` |
| `@-@ geometric_type` -> `double precision` | Berechnet die Gesamtlänge. Verfügbar für `lseg`, `path`. | `@-@ path '[(0,0),(1,0),(1,1)]'` -> `2` |
| `@@ geometric_type` -> `point` | Berechnet den Mittelpunkt. Verfügbar für `box`, `lseg`, `polygon`, `circle`. | `@@ box '(2,2),(0,0)'` -> `(1,1)` |
| `# geometric_type` -> `integer` | Gibt die Anzahl der Punkte zurück. Verfügbar für `path`, `polygon`. | `# path '((1,0),(0,1),(-1,0))'` -> `3` |
| `geometric_type # geometric_type` -> `point` | Berechnet den Schnittpunkt oder `NULL`, wenn es keinen gibt. Verfügbar für `lseg`, `line`. | `lseg '[(0,0),(1,1)]' # lseg '[(1,0),(0,1)]'` -> `(0.5,0.5)` |
| `box # box` -> `box` | Berechnet die Schnittmenge zweier Boxen oder `NULL`, wenn es keine gibt. | `box '(2,2),(-1,-1)' # box '(1,1),(-2,-2)'` -> `(1,1),(-1,-1)` |
| `geometric_type ## geometric_type` -> `point` | Berechnet den Punkt auf dem zweiten Objekt, der dem ersten Objekt am nächsten liegt. Verfügbar für die Typpaare `(point, box)`, `(point, lseg)`, `(point, line)`, `(lseg, box)`, `(lseg, lseg)`, `(line, lseg)`. | `point '(0,0)' ## lseg '[(2,0),(0,2)]'` -> `(1,1)` |
| `geometric_type <-> geometric_type` -> `double precision` | Berechnet den Abstand zwischen den Objekten. Verfügbar für alle sieben geometrischen Typen, für alle Kombinationen von `point` mit einem anderen geometrischen Typ sowie für die zusätzlichen Typpaare `(box, lseg)`, `(lseg, line)`, `(polygon, circle)` und die Kommutatorfälle. | `circle '<(0,0),1>' <-> circle '<(5,0),1>'` -> `3` |
| `geometric_type @> geometric_type` -> `boolean` | Enthält das erste Objekt das zweite? Verfügbar für die Typpaare `(box, point)`, `(box, box)`, `(path, point)`, `(polygon, point)`, `(polygon, polygon)`, `(circle, point)`, `(circle, circle)`. | `circle '<(0,0),2>' @> point '(1,1)'` -> `t` |
| `geometric_type <@ geometric_type` -> `boolean` | Ist das erste Objekt im zweiten enthalten oder liegt darauf? Verfügbar für die Typpaare `(point, box)`, `(point, lseg)`, `(point, line)`, `(point, path)`, `(point, polygon)`, `(point, circle)`, `(box, box)`, `(lseg, box)`, `(lseg, line)`, `(polygon, polygon)`, `(circle, circle)`. | `point '(1,1)' <@ circle '<(0,0),2>'` -> `t` |
| `geometric_type && geometric_type` -> `boolean` | Überlappen sich diese Objekte? Ein gemeinsamer Punkt genügt dafür. Verfügbar für `box`, `polygon`, `circle`. | `box '(1,1),(0,0)' && box '(2,2),(0,0)'` -> `t` |
| `geometric_type << geometric_type` -> `boolean` | Liegt das erste Objekt strikt links vom zweiten? Verfügbar für `point`, `box`, `polygon`, `circle`. | `circle '<(0,0),1>' << circle '<(5,0),1>'` -> `t` |
| `geometric_type >> geometric_type` -> `boolean` | Liegt das erste Objekt strikt rechts vom zweiten? Verfügbar für `point`, `box`, `polygon`, `circle`. | `circle '<(5,0),1>' >> circle '<(0,0),1>'` -> `t` |
| `geometric_type &< geometric_type` -> `boolean` | Erstreckt sich das erste Objekt nicht rechts über das zweite hinaus? Verfügbar für `box`, `polygon`, `circle`. | `box '(1,1),(0,0)' &< box '(2,2),(0,0)'` -> `t` |
| `geometric_type &> geometric_type` -> `boolean` | Erstreckt sich das erste Objekt nicht links über das zweite hinaus? Verfügbar für `box`, `polygon`, `circle`. | `box '(3,3),(0,0)' &> box '(2,2),(0,0)'` -> `t` |
| `geometric_type <<| geometric_type` -> `boolean` | Liegt das erste Objekt strikt unterhalb des zweiten? Verfügbar für `point`, `box`, `polygon`, `circle`. | `box '(3,3),(0,0)' <<| box '(5,5),(3,4)'` -> `t` |
| `geometric_type |>> geometric_type` -> `boolean` | Liegt das erste Objekt strikt oberhalb des zweiten? Verfügbar für `point`, `box`, `polygon`, `circle`. | `box '(5,5),(3,4)' |>> box '(3,3),(0,0)'` -> `t` |
| `geometric_type &<| geometric_type` -> `boolean` | Erstreckt sich das erste Objekt nicht oberhalb des zweiten? Verfügbar für `box`, `polygon`, `circle`. | `box '(1,1),(0,0)' &<| box '(2,2),(0,0)'` -> `t` |
| `geometric_type |&> geometric_type` -> `boolean` | Erstreckt sich das erste Objekt nicht unterhalb des zweiten? Verfügbar für `box`, `polygon`, `circle`. | `box '(3,3),(0,0)' |&> box '(2,2),(0,0)'` -> `t` |
| `box <^ box` -> `boolean` | Liegt das erste Objekt unterhalb des zweiten; Berührung an Kanten ist erlaubt. | `box '((1,1),(0,0))' <^ box '((2,2),(1,1))'` -> `t` |
| `box >^ box` -> `boolean` | Liegt das erste Objekt oberhalb des zweiten; Berührung an Kanten ist erlaubt. | `box '((2,2),(1,1))' >^ box '((1,1),(0,0))'` -> `t` |
| `geometric_type ?# geometric_type` -> `boolean` | Schneiden sich diese Objekte? Verfügbar für die Typpaare `(box, box)`, `(lseg, box)`, `(lseg, lseg)`, `(lseg, line)`, `(line, box)`, `(line, line)`, `(path, path)`. | `lseg '[(-1,0),(1,0)]' ?# box '(2,2),(-2,-2)'` -> `t` |
| `?- line` -> `boolean`<br>`?- lseg` -> `boolean` | Ist die Gerade beziehungsweise das Liniensegment horizontal? | `?- lseg '[(-1,0),(1,0)]'` -> `t` |
| `point ?- point` -> `boolean` | Sind die Punkte horizontal ausgerichtet, haben also dieselbe y-Koordinate? | `point '(1,0)' ?- point '(0,0)'` -> `t` |
| `?| line` -> `boolean`<br>`?| lseg` -> `boolean` | Ist die Gerade beziehungsweise das Liniensegment vertikal? | `?| lseg '[(-1,0),(1,0)]'` -> `f` |
| `point ?| point` -> `boolean` | Sind die Punkte vertikal ausgerichtet, haben also dieselbe x-Koordinate? | `point '(0,1)' ?| point '(0,0)'` -> `t` |
| `line ?-| line` -> `boolean`<br>`lseg ?-| lseg` -> `boolean` | Sind die Geraden beziehungsweise Liniensegmente senkrecht zueinander? | `lseg '[(0,0),(0,1)]' ?-| lseg '[(0,0),(1,0)]'` -> `t` |
| `line ?|| line` -> `boolean`<br>`lseg ?|| lseg` -> `boolean` | Sind die Geraden beziehungsweise Liniensegmente parallel? | `lseg '[(-1,0),(1,0)]' ?|| lseg '[(-1,2),(1,2)]'` -> `t` |
| `geometric_type ~= geometric_type` -> `boolean` | Sind diese Objekte gleich? Verfügbar für `point`, `box`, `polygon`, `circle`. | `polygon '((0,0),(1,1))' ~= polygon '((1,1),(0,0))'` -> `t` |

> Hinweis: Das „Drehen“ einer Box mit diesen Operatoren bewegt nur ihre Eckpunkte; die Box gilt weiterhin als eine Box mit achsenparallelen Seiten. Daher bleibt die Größe der Box nicht erhalten, wie es bei einer echten Rotation der Fall wäre.

> Achtung: Der Operator `~=`, „same as“, steht für den üblichen Gleichheitsbegriff der Typen `point`, `box`, `polygon` und `circle`. Einige geometrische Typen besitzen außerdem einen Operator `=`, aber `=` vergleicht nur auf gleiche Flächen. Die anderen skalaren Vergleichsoperatoren (`<=` usw.) vergleichen für diese Typen, sofern verfügbar, ebenfalls Flächen.

> Hinweis: Vor PostgreSQL 14 hießen die Operatoren für den Vergleich „Punkt liegt strikt unterhalb/oberhalb“ `point <<| point` und `point |>> point` entsprechend `<^` und `>^`. Diese Namen sind noch verfügbar, aber veraltet und werden irgendwann entfernt.

**Tabelle 9.37. Geometrische Funktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `area ( geometric_type )` -> `double precision` | Berechnet die Fläche. Verfügbar für `box`, `path`, `circle`. Eine `path`-Eingabe muss geschlossen sein, sonst wird `NULL` zurückgegeben. Wenn sich der Pfad selbst schneidet, kann das Ergebnis außerdem bedeutungslos sein. | `area(box '(2,2),(0,0)')` -> `4` |
| `center ( geometric_type )` -> `point` | Berechnet den Mittelpunkt. Verfügbar für `box`, `circle`. | `center(box '(1,2),(0,0)')` -> `(0.5,1)` |
| `diagonal ( box )` -> `lseg` | Extrahiert die Diagonale der Box als Liniensegment; entspricht `lseg(box)`. | `diagonal(box '(1,2),(0,0)')` -> `[(1,2),(0,0)]` |
| `diameter ( circle )` -> `double precision` | Berechnet den Durchmesser des Kreises. | `diameter(circle '<(0,0),2>')` -> `4` |
| `height ( box )` -> `double precision` | Berechnet die vertikale Größe der Box. | `height(box '(1,2),(0,0)')` -> `2` |
| `isclosed ( path )` -> `boolean` | Ist der Pfad geschlossen? | `isclosed(path '((0,0),(1,1),(2,0))')` -> `t` |
| `isopen ( path )` -> `boolean` | Ist der Pfad offen? | `isopen(path '[(0,0),(1,1),(2,0)]')` -> `t` |
| `length ( geometric_type )` -> `double precision` | Berechnet die Gesamtlänge. Verfügbar für `lseg`, `path`. | `length(path '((-1,0),(1,0))')` -> `4` |
| `npoints ( geometric_type )` -> `integer` | Gibt die Anzahl der Punkte zurück. Verfügbar für `path`, `polygon`. | `npoints(path '[(0,0),(1,1),(2,0)]')` -> `3` |
| `pclose ( path )` -> `path` | Wandelt einen Pfad in geschlossene Form um. | `pclose(path '[(0,0),(1,1),(2,0)]')` -> `((0,0),(1,1),(2,0))` |
| `popen ( path )` -> `path` | Wandelt einen Pfad in offene Form um. | `popen(path '((0,0),(1,1),(2,0))')` -> `[(0,0),(1,1),(2,0)]` |
| `radius ( circle )` -> `double precision` | Berechnet den Radius des Kreises. | `radius(circle '<(0,0),2>')` -> `2` |
| `slope ( point, point )` -> `double precision` | Berechnet die Steigung einer Geraden durch die beiden Punkte. | `slope(point '(0,0)', point '(2,1)')` -> `0.5` |
| `width ( box )` -> `double precision` | Berechnet die horizontale Größe der Box. | `width(box '(1,2),(0,0)')` -> `1` |

**Tabelle 9.38. Umwandlungsfunktionen für geometrische Typen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `box ( circle )` -> `box` | Berechnet die in den Kreis eingeschriebene Box. | `box(circle '<(0,0),2>')` -> `(1.414213562373095,1.414213562373095),(-1.414213562373095,-1.414213562373095)` |
| `box ( point )` -> `box` | Wandelt einen Punkt in eine leere Box um. | `box(point '(1,0)')` -> `(1,0),(1,0)` |
| `box ( point, point )` -> `box` | Wandelt zwei beliebige Eckpunkte in eine Box um. | `box(point '(0,1)', point '(1,0)')` -> `(1,1),(0,0)` |
| `box ( polygon )` -> `box` | Berechnet die Bounding Box eines Polygons. | `box(polygon '((0,0),(1,1),(2,0))')` -> `(2,1),(0,0)` |
| `bound_box ( box, box )` -> `box` | Berechnet die Bounding Box zweier Boxen. | `bound_box(box '(1,1),(0,0)', box '(4,4),(3,3)')` -> `(4,4),(0,0)` |
| `circle ( box )` -> `circle` | Berechnet den kleinsten Kreis, der die Box einschließt. | `circle(box '(1,1),(0,0)')` -> `<(0.5,0.5),0.7071067811865476>` |
| `circle ( point, double precision )` -> `circle` | Konstruiert einen Kreis aus Mittelpunkt und Radius. | `circle(point '(0,0)', 2.0)` -> `<(0,0),2>` |
| `circle ( polygon )` -> `circle` | Wandelt ein Polygon in einen Kreis um. Der Mittelpunkt des Kreises ist der Mittelwert der Positionen der Polygonpunkte, und der Radius ist der durchschnittliche Abstand der Polygonpunkte von diesem Mittelpunkt. | `circle(polygon '((0,0),(1,3),(2,0))')` -> `<(1,1),1.6094757082487299>` |
| `line ( point, point )` -> `line` | Wandelt zwei Punkte in die durch sie verlaufende Gerade um. | `line(point '(-1,0)', point '(1,0)')` -> `{0,-1,0}` |
| `lseg ( box )` -> `lseg` | Extrahiert die Diagonale der Box als Liniensegment. | `lseg(box '(1,0),(-1,0)')` -> `[(1,0),(-1,0)]` |
| `lseg ( point, point )` -> `lseg` | Konstruiert ein Liniensegment aus zwei Endpunkten. | `lseg(point '(-1,0)', point '(1,0)')` -> `[(-1,0),(1,0)]` |
| `path ( polygon )` -> `path` | Wandelt ein Polygon in einen geschlossenen Pfad mit derselben Punktliste um. | `path(polygon '((0,0),(1,1),(2,0))')` -> `((0,0),(1,1),(2,0))` |
| `point ( double precision, double precision )` -> `point` | Konstruiert einen Punkt aus seinen Koordinaten. | `point(23.4, -44.5)` -> `(23.4,-44.5)` |
| `point ( box )` -> `point` | Berechnet den Mittelpunkt der Box. | `point(box '(1,0),(-1,0)')` -> `(0,0)` |
| `point ( circle )` -> `point` | Berechnet den Mittelpunkt des Kreises. | `point(circle '<(0,0),2>')` -> `(0,0)` |
| `point ( lseg )` -> `point` | Berechnet den Mittelpunkt des Liniensegments. | `point(lseg '[(-1,0),(1,0)]')` -> `(0,0)` |
| `point ( polygon )` -> `point` | Berechnet den Mittelpunkt des Polygons, also den Mittelwert der Positionen seiner Punkte. | `point(polygon '((0,0),(1,1),(2,0))')` -> `(1,0.3333333333333333)` |
| `polygon ( box )` -> `polygon` | Wandelt eine Box in ein Polygon mit vier Punkten um. | `polygon(box '(1,1),(0,0)')` -> `((0,0),(0,1),(1,1),(1,0))` |
| `polygon ( circle )` -> `polygon` | Wandelt einen Kreis in ein Polygon mit zwölf Punkten um. | `polygon(circle '<(0,0),2>')` -> `((-2,0),(-1.7320508075688774,0.9999999999999999),(-1.0000000000000002,1.7320508075688772),(-1.2246063538223773e-16,2),(0.9999999999999996,1.7320508075688774),(1.732050807568877,1.0000000000000007),(2,2.4492127076447545e-16),(1.7320508075688776,-0.9999999999999994),(1.0000000000000009,-1.7320508075688767),(3.673819061467132e-16,-2),(-0.9999999999999987,-1.732050807568878),(-1.7320508075688767,-1.0000000000000009))` |
| `polygon ( integer, circle )` -> `polygon` | Wandelt einen Kreis in ein Polygon mit `n` Punkten um. | `polygon(4, circle '<(3,0),1>')` -> `((2,0),(3,1),(4,1.2246063538223773e-16),(3,-1))` |
| `polygon ( path )` -> `polygon` | Wandelt einen geschlossenen Pfad in ein Polygon mit derselben Punktliste um. | `polygon(path '((0,0),(1,1),(2,0))')` -> `((0,0),(1,1),(2,0))` |

Es ist möglich, auf die beiden Komponenten eines Punkts zuzugreifen, als wäre der Punkt ein Array mit den Indizes 0 und 1. Wenn `t.p` zum Beispiel eine Punktspalte ist, ruft `SELECT p[0] FROM t` die X-Koordinate ab, und `UPDATE t SET p[1] = ...` ändert die Y-Koordinate. Auf dieselbe Weise kann ein Wert des Typs `box` oder `lseg` als Array aus zwei Punktwerten behandelt werden.

## 9.12. Netzwerkadressfunktionen und -operatoren

Die IP-Netzwerkadresstypen `cidr` und `inet` unterstützen die üblichen Vergleichsoperatoren aus Tabelle 9.1 sowie die spezialisierten Operatoren und Funktionen aus Tabelle 9.39 und Tabelle 9.40.

Jeder `cidr`-Wert kann implizit nach `inet` gecastet werden; daher funktionieren die unten als auf `inet` arbeitend gezeigten Operatoren und Funktionen auch mit `cidr`-Werten. Wo es separate Funktionen für `inet` und `cidr` gibt, liegt das daran, dass das Verhalten in beiden Fällen unterschiedlich sein soll. Außerdem ist es erlaubt, einen `inet`-Wert nach `cidr` zu casten. Dabei werden alle Bits rechts der Netzmaske stillschweigend auf null gesetzt, um einen gültigen `cidr`-Wert zu erzeugen.

**Tabelle 9.39. IP-Adressoperatoren**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `inet << inet` -> `boolean` | Ist das Subnetz strikt im anderen Subnetz enthalten? Dieser Operator und die nächsten vier testen Subnetz-Einschluss. Sie betrachten nur die Netzwerkteile der beiden Adressen, ignorieren also alle Bits rechts der Netzmasken, und bestimmen, ob ein Netzwerk identisch mit dem anderen oder ein Subnetz davon ist. | `inet '192.168.1.5' << inet '192.168.1/24'` -> `t`<br>`inet '192.168.0.5' << inet '192.168.1/24'` -> `f`<br>`inet '192.168.1/24' << inet '192.168.1/24'` -> `f` |
| `inet <<= inet` -> `boolean` | Ist das Subnetz im anderen Subnetz enthalten oder ihm gleich? | `inet '192.168.1/24' <<= inet '192.168.1/24'` -> `t` |
| `inet >> inet` -> `boolean` | Enthält das Subnetz das andere Subnetz strikt? | `inet '192.168.1/24' >> inet '192.168.1.5'` -> `t` |
| `inet >>= inet` -> `boolean` | Enthält das Subnetz das andere Subnetz oder ist ihm gleich? | `inet '192.168.1/24' >>= inet '192.168.1/24'` -> `t` |
| `inet && inet` -> `boolean` | Enthält eines der Subnetze das andere oder ist ihm gleich? | `inet '192.168.1/24' && inet '192.168.1.80/28'` -> `t`<br>`inet '192.168.1/24' && inet '192.168.2.0/28'` -> `f` |
| `~ inet` -> `inet` | Berechnet bitweises NOT. | `~ inet '192.168.1.6'` -> `63.87.254.249` |
| `inet & inet` -> `inet` | Berechnet bitweises AND. | `inet '192.168.1.6' & inet '0.0.0.255'` -> `0.0.0.6` |
| `inet | inet` -> `inet` | Berechnet bitweises OR. | `inet '192.168.1.6' | inet '0.0.0.255'` -> `192.168.1.255` |
| `inet + bigint` -> `inet` | Addiert einen Versatz zu einer Adresse. | `inet '192.168.1.6' + 25` -> `192.168.1.31` |
| `bigint + inet` -> `inet` | Addiert einen Versatz zu einer Adresse. | `200 + inet '::ffff:fff0:1'` -> `::ffff:255.240.0.201` |
| `inet - bigint` -> `inet` | Subtrahiert einen Versatz von einer Adresse. | `inet '192.168.1.43' - 36` -> `192.168.1.7` |
| `inet - inet` -> `bigint` | Berechnet die Differenz zweier Adressen. | `inet '192.168.1.43' - inet '192.168.1.19'` -> `24`<br>`inet '::1' - inet '::ffff:1'` -> `-4294901760` |

**Tabelle 9.40. IP-Adressfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `abbrev ( inet )` -> `text` | Erzeugt ein abgekürztes Anzeigeformat als Text. Das Ergebnis ist dasselbe wie das der `inet`-Ausgabefunktion; „abgekürzt“ ist es nur im Vergleich zum Ergebnis eines expliziten Casts nach `text`, der aus historischen Gründen den Netzmaskenteil nie unterdrückt. | `abbrev(inet '10.1.0.0/32')` -> `10.1.0.0` |
| `abbrev ( cidr )` -> `text` | Erzeugt ein abgekürztes Anzeigeformat als Text. Die Abkürzung besteht darin, rechts der Netzmaske vollständig nullwertige Oktette wegzulassen; weitere Beispiele stehen in Tabelle 8.22. | `abbrev(cidr '10.1.0.0/16')` -> `10.1/16` |
| `broadcast ( inet )` -> `inet` | Berechnet die Broadcast-Adresse für das Netzwerk der Adresse. | `broadcast(inet '192.168.1.5/24')` -> `192.168.1.255/24` |
| `family ( inet )` -> `integer` | Gibt die Adressfamilie zurück: 4 für IPv4, 6 für IPv6. | `family(inet '::1')` -> `6` |
| `host ( inet )` -> `text` | Gibt die IP-Adresse als Text zurück und ignoriert die Netzmaske. | `host(inet '192.168.1.0/24')` -> `192.168.1.0` |
| `hostmask ( inet )` -> `inet` | Berechnet die Hostmaske für das Netzwerk der Adresse. | `hostmask(inet '192.168.23.20/30')` -> `0.0.0.3` |
| `inet_merge ( inet, inet )` -> `cidr` | Berechnet das kleinste Netzwerk, das beide angegebenen Netzwerke umfasst. | `inet_merge(inet '192.168.1.5/24', inet '192.168.2.5/24')` -> `192.168.0.0/22` |
| `inet_same_family ( inet, inet )` -> `boolean` | Prüft, ob die Adressen zur selben IP-Familie gehören. | `inet_same_family(inet '192.168.1.5/24', inet '::1')` -> `f` |
| `masklen ( inet )` -> `integer` | Gibt die Länge der Netzmaske in Bits zurück. | `masklen(inet '192.168.1.5/24')` -> `24` |
| `netmask ( inet )` -> `inet` | Berechnet die Netzmaske für das Netzwerk der Adresse. | `netmask(inet '192.168.1.5/24')` -> `255.255.255.0` |
| `network ( inet )` -> `cidr` | Gibt den Netzwerkteil der Adresse zurück und setzt alles rechts der Netzmaske auf null. Dies entspricht dem Casten des Werts nach `cidr`. | `network(inet '192.168.1.5/24')` -> `192.168.1.0/24` |
| `set_masklen ( inet, integer )` -> `inet` | Setzt die Netzmaskenlänge für einen `inet`-Wert. Der Adressteil ändert sich nicht. | `set_masklen(inet '192.168.1.5/24', 16)` -> `192.168.1.5/16` |
| `set_masklen ( cidr, integer )` -> `cidr` | Setzt die Netzmaskenlänge für einen `cidr`-Wert. Adressbits rechts der neuen Netzmaske werden auf null gesetzt. | `set_masklen(cidr '192.168.1.0/24', 16)` -> `192.168.0.0/16` |
| `text ( inet )` -> `text` | Gibt die unabgekürzte IP-Adresse und Netzmaskenlänge als Text zurück. Dies hat dasselbe Ergebnis wie ein expliziter Cast nach `text`. | `text(inet '192.168.1.5')` -> `192.168.1.5/32` |

> Tipp: Die Funktionen `abbrev`, `host` und `text` sind hauptsächlich dafür gedacht, alternative Anzeigeformate für IP-Adressen bereitzustellen.

Die MAC-Adresstypen `macaddr` und `macaddr8` unterstützen die üblichen Vergleichsoperatoren aus Tabelle 9.1 sowie die spezialisierten Funktionen aus Tabelle 9.41. Außerdem unterstützen sie die bitweisen logischen Operatoren `~`, `&` und `|` (NOT, AND und OR), genau wie oben für IP-Adressen gezeigt.

**Tabelle 9.41. MAC-Adressfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `trunc ( macaddr )` -> `macaddr` | Setzt die letzten 3 Bytes der Adresse auf null. Das verbleibende Präfix kann einem bestimmten Hersteller zugeordnet werden, mit Daten, die nicht in PostgreSQL enthalten sind. | `trunc(macaddr '12:34:56:78:90:ab')` -> `12:34:56:00:00:00` |
| `trunc ( macaddr8 )` -> `macaddr8` | Setzt die letzten 5 Bytes der Adresse auf null. Das verbleibende Präfix kann einem bestimmten Hersteller zugeordnet werden, mit Daten, die nicht in PostgreSQL enthalten sind. | `trunc(macaddr8 '12:34:56:78:90:ab:cd:ef')` -> `12:34:56:00:00:00:00:00` |
| `macaddr8_set7bit ( macaddr8 )` -> `macaddr8` | Setzt das siebte Bit der Adresse auf eins und erzeugt damit das sogenannte modified EUI-64 zur Einbindung in eine IPv6-Adresse. | `macaddr8_set7bit(macaddr8 '00:34:56:ab:cd:ef')` -> `02:34:56:ff:fe:ab:cd:ef` |

## 9.13. Volltextsuchfunktionen und -operatoren

Tabelle 9.42, Tabelle 9.43 und Tabelle 9.44 fassen die Funktionen und Operatoren zusammen, die für die Volltextsuche bereitgestellt werden. Eine ausführliche Erklärung der Textsuche in PostgreSQL finden Sie in [Kapitel 12](12_Volltextsuche.md).

**Tabelle 9.42. Textsuchoperatoren**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `tsvector @@ tsquery` -> `boolean`<br>`tsquery @@ tsvector` -> `boolean` | Passt der `tsvector` zur `tsquery`? Die Argumente können in beiden Reihenfolgen angegeben werden. | `to_tsvector('fat cats ate rats') @@ to_tsquery('cat & rat')` -> `t` |
| `text @@ tsquery` -> `boolean` | Passt die Textzeichenkette nach implizitem Aufruf von `to_tsvector()` zur `tsquery`? | `'fat cats ate rats' @@ to_tsquery('cat & rat')` -> `t` |
| `tsvector || tsvector` -> `tsvector` | Verkettet zwei `tsvector`-Werte. Wenn beide Eingaben Lexempositionen enthalten, werden die Positionen der zweiten Eingabe entsprechend angepasst. | `'a:1 b:2'::tsvector || 'c:1 d:2 b:3'::tsvector` -> `'a':1 'b':2,5 'c':3 'd':4` |
| `tsquery && tsquery` -> `tsquery` | Verknüpft zwei `tsquery`-Werte mit AND und erzeugt eine Abfrage, die Dokumente trifft, die zu beiden Eingabeabfragen passen. | `'fat | rat'::tsquery && 'cat'::tsquery` -> `( 'fat' | 'rat' ) & 'cat'` |
| `tsquery || tsquery` -> `tsquery` | Verknüpft zwei `tsquery`-Werte mit OR und erzeugt eine Abfrage, die Dokumente trifft, die zu einer der Eingabeabfragen passen. | `'fat | rat'::tsquery || 'cat'::tsquery` -> `'fat' | 'rat' | 'cat'` |
| `!! tsquery` -> `tsquery` | Negiert eine `tsquery` und erzeugt eine Abfrage, die Dokumente trifft, die nicht zur Eingabeabfrage passen. | `!! 'cat'::tsquery` -> `!'cat'` |
| `tsquery <-> tsquery` -> `tsquery` | Konstruiert eine Phrasenabfrage, die passt, wenn die beiden Eingabeabfragen an aufeinanderfolgenden Lexemen passen. | `to_tsquery('fat') <-> to_tsquery('rat')` -> `'fat' <-> 'rat'` |
| `tsquery @> tsquery` -> `boolean` | Enthält die erste `tsquery` die zweite? Dabei wird nur betrachtet, ob alle Lexeme, die in einer Abfrage vorkommen, auch in der anderen vorkommen; Verknüpfungsoperatoren werden ignoriert. | `'cat'::tsquery @> 'cat & rat'::tsquery` -> `f` |
| `tsquery <@ tsquery` -> `boolean` | Ist die erste `tsquery` in der zweiten enthalten? Dabei wird nur betrachtet, ob alle Lexeme, die in einer Abfrage vorkommen, auch in der anderen vorkommen; Verknüpfungsoperatoren werden ignoriert. | `'cat'::tsquery <@ 'cat & rat'::tsquery` -> `t`<br>`'cat'::tsquery <@ '!cat & rat'::tsquery` -> `t` |

Zusätzlich zu diesen spezialisierten Operatoren stehen für die Typen `tsvector` und `tsquery` die üblichen Vergleichsoperatoren aus Tabelle 9.1 zur Verfügung. Für die Textsuche selbst sind sie nicht sehr nützlich, erlauben aber beispielsweise, eindeutige Indizes auf Spalten dieser Typen zu erstellen.

**Tabelle 9.43. Textsuchfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `array_to_tsvector ( text[] )` -> `tsvector` | Wandelt ein Array von Textzeichenketten in einen `tsvector` um. Die angegebenen Zeichenketten werden unverändert als Lexeme verwendet, ohne weitere Verarbeitung. Array-Elemente dürfen weder leere Zeichenketten noch `NULL` sein. | `array_to_tsvector('{fat,cat,rat}'::text[])` -> `'cat' 'fat' 'rat'` |
| `get_current_ts_config ( )` -> `regconfig` | Gibt die OID der aktuellen Standard-Textsuchkonfiguration zurück, wie durch `default_text_search_config` festgelegt. | `get_current_ts_config()` -> `english` |
| `length ( tsvector )` -> `integer` | Gibt die Anzahl der Lexeme im `tsvector` zurück. | `length('fat:2,4 cat:3 rat:5A'::tsvector)` -> `3` |
| `numnode ( tsquery )` -> `integer` | Gibt die Anzahl der Lexeme plus Operatoren in der `tsquery` zurück. | `numnode('(fat & rat) | cat'::tsquery)` -> `5` |
| `plainto_tsquery ( [ config regconfig, ] query text )` -> `tsquery` | Wandelt Text in eine `tsquery` um und normalisiert Wörter gemäß der angegebenen oder Standardkonfiguration. Zeichensetzung in der Zeichenkette wird ignoriert; sie bestimmt keine Abfrageoperatoren. Die resultierende Abfrage trifft Dokumente, die alle Nicht-Stoppwörter im Text enthalten. | `plainto_tsquery('english', 'The Fat Rats')` -> `'fat' & 'rat'` |
| `phraseto_tsquery ( [ config regconfig, ] query text )` -> `tsquery` | Wandelt Text in eine `tsquery` um und normalisiert Wörter gemäß der angegebenen oder Standardkonfiguration. Zeichensetzung wird ignoriert; sie bestimmt keine Abfrageoperatoren. Die resultierende Abfrage trifft Phrasen, die alle Nicht-Stoppwörter im Text enthalten. | `phraseto_tsquery('english', 'The Fat Rats')` -> `'fat' <-> 'rat'`<br>`phraseto_tsquery('english', 'The Cat and Rats')` -> `'cat' <2> 'rat'` |
| `websearch_to_tsquery ( [ config regconfig, ] query text )` -> `tsquery` | Wandelt Text in eine `tsquery` um und normalisiert Wörter gemäß der angegebenen oder Standardkonfiguration. Gequotete Wortfolgen werden in Phrasentests umgewandelt. Das Wort `or` wird als OR-Operator verstanden, und ein Bindestrich erzeugt einen NOT-Operator; andere Zeichensetzung wird ignoriert. Das nähert das Verhalten verbreiteter Websuchwerkzeuge an. | `websearch_to_tsquery('english', '"fat rat" or cat dog')` -> `'fat' <-> 'rat' | 'cat' & 'dog'` |
| `querytree ( tsquery )` -> `text` | Erzeugt eine Darstellung des indexierbaren Teils einer `tsquery`. Ein leeres Ergebnis oder nur `T` zeigt eine nicht indexierbare Abfrage an. | `querytree('foo & ! bar'::tsquery)` -> `'foo'` |
| `setweight ( vector tsvector, weight "char" )` -> `tsvector` | Weist jedem Element des Vektors das angegebene Gewicht zu. | `setweight('fat:2,4 cat:3 rat:5B'::tsvector, 'A')` -> `'cat':3A 'fat':2A,4A 'rat':5A` |
| `setweight ( vector tsvector, weight "char", lexemes text[] )` -> `tsvector` | Weist den Elementen des Vektors, die in `lexemes` aufgeführt sind, das angegebene Gewicht zu. Die Zeichenketten in `lexemes` werden unverändert als Lexeme verwendet, ohne weitere Verarbeitung. Zeichenketten, die zu keinem Lexem in `vector` passen, werden ignoriert. | `setweight('fat:2,4 cat:3 rat:5,6B'::tsvector, 'A', '{cat,rat}')` -> `'cat':3A 'fat':2,4 'rat':5A,6A` |
| `strip ( tsvector )` -> `tsvector` | Entfernt Positionen und Gewichte aus dem `tsvector`. | `strip('fat:2,4 cat:3 rat:5A'::tsvector)` -> `'cat' 'fat' 'rat'` |
| `to_tsquery ( [ config regconfig, ] query text )` -> `tsquery` | Wandelt Text in eine `tsquery` um und normalisiert Wörter gemäß der angegebenen oder Standardkonfiguration. Die Wörter müssen durch gültige `tsquery`-Operatoren kombiniert sein. | `to_tsquery('english', 'The & Fat & Rats')` -> `'fat' & 'rat'` |
| `to_tsvector ( [ config regconfig, ] document text )` -> `tsvector` | Wandelt Text in einen `tsvector` um und normalisiert Wörter gemäß der angegebenen oder Standardkonfiguration. Positionsinformationen sind im Ergebnis enthalten. | `to_tsvector('english', 'The Fat Rats')` -> `'fat':2 'rat':3` |
| `to_tsvector ( [ config regconfig, ] document json )` -> `tsvector`<br>`to_tsvector ( [ config regconfig, ] document jsonb )` -> `tsvector` | Wandelt jeden Zeichenkettenwert im JSON-Dokument in einen `tsvector` um und normalisiert Wörter gemäß der angegebenen oder Standardkonfiguration. Die Ergebnisse werden anschließend in Dokumentreihenfolge verkettet. Positionsinformationen werden so erzeugt, als gäbe es zwischen jedem Paar von Zeichenkettenwerten ein Stoppwort. Beachten Sie, dass die „Dokumentreihenfolge“ der Felder eines JSON-Objekts bei Eingabe vom Typ `jsonb` implementierungsabhängig ist; der Unterschied ist in den Beispielen sichtbar. | `to_tsvector('english', '{"aa": "The Fat Rats", "b": "dog"}'::json)` -> `'dog':5 'fat':2 'rat':3`<br>`to_tsvector('english', '{"aa": "The Fat Rats", "b": "dog"}'::jsonb)` -> `'dog':1 'fat':4 'rat':5` |
| `json_to_tsvector ( [ config regconfig, ] document json, filter jsonb )` -> `tsvector`<br>`jsonb_to_tsvector ( [ config regconfig, ] document jsonb, filter jsonb )` -> `tsvector` | Wählt jedes vom Filter angeforderte Element im JSON-Dokument aus und wandelt es in einen `tsvector` um; Wörter werden gemäß der angegebenen oder Standardkonfiguration normalisiert. Die Ergebnisse werden in Dokumentreihenfolge verkettet. Positionsinformationen werden so erzeugt, als gäbe es zwischen jedem Paar ausgewählter Elemente ein Stoppwort. Der Filter muss ein `jsonb`-Array mit null oder mehr dieser Schlüsselwörter sein: `"string"`, `"numeric"`, `"boolean"`, `"key"` oder `"all"`. Als Sonderfall kann der Filter auch ein einfacher JSON-Wert sein, der eines dieser Schlüsselwörter ist. | `json_to_tsvector('english', '{"a": "The Fat Rats", "b": 123}'::json, '["string", "numeric"]')` -> `'123':5 'fat':2 'rat':3`<br>`json_to_tsvector('english', '{"cat": "The Fat Rats", "dog": 123}'::json, '"all"')` -> `'123':9 'cat':1 'dog':7 'fat':4 'rat':5` |
| `ts_delete ( vector tsvector, lexeme text )` -> `tsvector` | Entfernt jedes Vorkommen des angegebenen Lexems aus dem Vektor. Die Lexemzeichenkette wird unverändert als Lexem behandelt, ohne weitere Verarbeitung. | `ts_delete('fat:2,4 cat:3 rat:5A'::tsvector, 'fat')` -> `'cat':3 'rat':5A` |
| `ts_delete ( vector tsvector, lexemes text[] )` -> `tsvector` | Entfernt jedes Vorkommen der Lexeme in `lexemes` aus dem Vektor. Die Zeichenketten in `lexemes` werden unverändert als Lexeme verwendet, ohne weitere Verarbeitung. Zeichenketten, die zu keinem Lexem in `vector` passen, werden ignoriert. | `ts_delete('fat:2,4 cat:3 rat:5A'::tsvector, ARRAY['fat','rat'])` -> `'cat':3` |
| `ts_filter ( vector tsvector, weights "char"[] )` -> `tsvector` | Wählt nur Elemente mit den angegebenen Gewichten aus dem Vektor aus. | `ts_filter('fat:2,4 cat:3b,7c rat:5A'::tsvector, '{a,b}')` -> `'cat':3B 'rat':5A` |
| `ts_headline ( [ config regconfig, ] document text, query tsquery [, options text ] )` -> `text` | Zeigt in abgekürzter Form die Treffer der Abfrage im Dokument an; das Dokument muss Rohtext sein, kein `tsvector`. Wörter im Dokument werden gemäß der angegebenen oder Standardkonfiguration normalisiert, bevor sie gegen die Abfrage abgeglichen werden. Die Verwendung dieser Funktion wird in Abschnitt 12.3.4 besprochen, der auch die verfügbaren Optionen beschreibt. | `ts_headline('The fat cat ate the rat.', 'cat')` -> `The fat <b>cat</b> ate the rat.` |
| `ts_headline ( [ config regconfig, ] document json, query tsquery [, options text ] )` -> `text`<br>`ts_headline ( [ config regconfig, ] document jsonb, query tsquery [, options text ] )` -> `text` | Zeigt in abgekürzter Form Treffer der Abfrage an, die in Zeichenkettenwerten innerhalb des JSON-Dokuments auftreten. Weitere Details siehe Abschnitt 12.3.4. | `ts_headline('{"cat":"raining cats and dogs"}'::jsonb, 'cat')` -> `{"cat": "raining <b>cats</b> and dogs"}` |
| `ts_rank ( [ weights real[], ] vector tsvector, query tsquery [, normalization integer ] )` -> `real` | Berechnet eine Bewertung, die zeigt, wie gut der Vektor zur Abfrage passt. Details siehe Abschnitt 12.3.3. | `ts_rank(to_tsvector('raining cats and dogs'), 'cat')` -> `0.06079271` |
| `ts_rank_cd ( [ weights real[], ] vector tsvector, query tsquery [, normalization integer ] )` -> `real` | Berechnet mit einem Cover-Density-Algorithmus eine Bewertung, die zeigt, wie gut der Vektor zur Abfrage passt. Details siehe Abschnitt 12.3.3. | `ts_rank_cd(to_tsvector('raining cats and dogs'), 'cat')` -> `0.1` |
| `ts_rewrite ( query tsquery, target tsquery, substitute tsquery )` -> `tsquery` | Ersetzt Vorkommen von `target` innerhalb der Abfrage durch `substitute`. Details siehe Abschnitt 12.4.2.1. | `ts_rewrite('a & b'::tsquery, 'a'::tsquery, 'foo|bar'::tsquery)` -> `'b' & ( 'foo' | 'bar' )` |
| `ts_rewrite ( query tsquery, select text )` -> `tsquery` | Ersetzt Teile der Abfrage gemäß Ziel- und Ersatzwerten, die durch Ausführen eines `SELECT`-Befehls gewonnen werden. Details siehe Abschnitt 12.4.2.1. | `SELECT ts_rewrite('a & b'::tsquery, 'SELECT t,s FROM aliases')` -> `'b' & ( 'foo' | 'bar' )` |
| `tsquery_phrase ( query1 tsquery, query2 tsquery )` -> `tsquery` | Konstruiert eine Phrasenabfrage, die nach Treffern von `query1` und `query2` an aufeinanderfolgenden Lexemen sucht; entspricht dem Operator `<->`. | `tsquery_phrase(to_tsquery('fat'), to_tsquery('cat'))` -> `'fat' <-> 'cat'` |
| `tsquery_phrase ( query1 tsquery, query2 tsquery, distance integer )` -> `tsquery` | Konstruiert eine Phrasenabfrage, die nach Treffern von `query1` und `query2` sucht, die genau `distance` Lexeme auseinanderliegen. | `tsquery_phrase(to_tsquery('fat'), to_tsquery('cat'), 10)` -> `'fat' <10> 'cat'` |
| `tsvector_to_array ( tsvector )` -> `text[]` | Wandelt einen `tsvector` in ein Array von Lexemen um. | `tsvector_to_array('fat:2,4 cat:3 rat:5A'::tsvector)` -> `{cat,fat,rat}` |
| `unnest ( tsvector )` -> `setof record ( lexeme text, positions smallint[], weights text )` | Erweitert einen `tsvector` in eine Menge von Zeilen, eine pro Lexem. | Siehe Beispiel unten. |

```sql
SELECT * FROM unnest('cat:3 fat:2,4 rat:5A'::tsvector);
```

```text
 lexeme | positions | weights
--------+-----------+---------
 cat    | {3}       | {D}
 fat    | {2,4}     | {D,D}
 rat    | {5}       | {A}
```

> Hinweis: Alle Textsuchfunktionen, die ein optionales Argument `regconfig` akzeptieren, verwenden die durch `default_text_search_config` angegebene Konfiguration, wenn dieses Argument weggelassen wird.

Die Funktionen in Tabelle 9.44 werden separat aufgeführt, weil sie normalerweise nicht in alltäglichen Textsuchoperationen verwendet werden. Sie sind hauptsächlich für Entwicklung und Debugging neuer Textsuchkonfigurationen hilfreich.

**Tabelle 9.44. Debugging-Funktionen für Textsuche**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `ts_debug ( [ config regconfig, ] document text )` -> `setof record ( alias text, description text, token text, dictionaries regdictionary[], dictionary regdictionary, lexemes text[] )` | Extrahiert und normalisiert Tokens aus dem Dokument gemäß der angegebenen oder Standard-Textsuchkonfiguration und gibt Informationen darüber zurück, wie jedes Token verarbeitet wurde. Details siehe Abschnitt 12.8.1. | `ts_debug('english', 'The Brightest supernovaes')` -> `(asciiword,"Word, all ASCII",The,{english_stem},english_stem,{}) ...` |
| `ts_lexize ( dict regdictionary, token text )` -> `text[]` | Gibt ein Array von Ersatzlexemen zurück, wenn das Eingabetoken dem Wörterbuch bekannt ist, ein leeres Array, wenn das Token zwar bekannt, aber ein Stoppwort ist, oder `NULL`, wenn es kein bekanntes Wort ist. Details siehe Abschnitt 12.8.3. | `ts_lexize('english_stem', 'stars')` -> `{star}` |
| `ts_parse ( parser_name text, document text )` -> `setof record ( tokid integer, token text )` | Extrahiert Tokens aus dem Dokument mit dem benannten Parser. Details siehe Abschnitt 12.8.2. | `ts_parse('default', 'foo - bar')` -> `(1,foo) ...` |
| `ts_parse ( parser_oid oid, document text )` -> `setof record ( tokid integer, token text )` | Extrahiert Tokens aus dem Dokument mit einem durch OID angegebenen Parser. Details siehe Abschnitt 12.8.2. | `ts_parse(3722, 'foo - bar')` -> `(1,foo) ...` |
| `ts_token_type ( parser_name text )` -> `setof record ( tokid integer, alias text, description text )` | Gibt eine Tabelle zurück, die jeden Tokentyp beschreibt, den der benannte Parser erkennen kann. Details siehe Abschnitt 12.8.2. | `ts_token_type('default')` -> `(1,asciiword,"Word, all ASCII") ...` |
| `ts_token_type ( parser_oid oid )` -> `setof record ( tokid integer, alias text, description text )` | Gibt eine Tabelle zurück, die jeden Tokentyp beschreibt, den ein durch OID angegebener Parser erkennen kann. Details siehe Abschnitt 12.8.2. | `ts_token_type(3722)` -> `(1,asciiword,"Word, all ASCII") ...` |
| `ts_stat ( sqlquery text [, weights text ] )` -> `setof record ( word text, ndoc integer, nentry integer )` | Führt `sqlquery` aus, die eine einzelne `tsvector`-Spalte zurückgeben muss, und gibt Statistiken über jedes unterschiedliche Lexem zurück, das in den Daten enthalten ist. Details siehe Abschnitt 12.4.4. | `ts_stat('SELECT vector FROM apod')` -> `(foo,10,15) ...` |

## 9.14. UUID-Funktionen

Tabelle 9.45 zeigt die PostgreSQL-Funktionen, mit denen UUIDs erzeugt werden können.

**Tabelle 9.45. UUID-Erzeugungsfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `gen_random_uuid ( )` -> `uuid`<br>`uuidv4 ( )` -> `uuid` | Erzeugt eine UUID der Version 4, also eine zufällige UUID. | `gen_random_uuid()` -> `5b30857f-0bfa-48b5-ac0b-5c64e28078d1`<br>`uuidv4()` -> `b42410ee-132f-42ee-9e4f-09a6485c95b8` |
| `uuidv7 ( [ shift interval ] )` -> `uuid` | Erzeugt eine UUID der Version 7, also eine zeitlich geordnete UUID. Der Zeitstempel wird aus Unix-Zeitstempel mit Millisekundengenauigkeit, Sub-Millisekunden-Zeitstempel und Zufall berechnet. Der optionale Parameter `shift` verschiebt den berechneten Zeitstempel um das angegebene Intervall. | `uuidv7()` -> `019535d9-3df7-79fb-b466-fa907fa17f9e` |

> Hinweis: Das Modul `uuid-ossp` stellt zusätzliche Funktionen bereit, die andere Standardalgorithmen zum Erzeugen von UUIDs implementieren.

Tabelle 9.46 zeigt die PostgreSQL-Funktionen, mit denen Informationen aus UUIDs extrahiert werden können.

**Tabelle 9.46. UUID-Extraktionsfunktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `uuid_extract_timestamp ( uuid )` -> `timestamp with time zone` | Extrahiert einen Zeitstempel mit Zeitzone aus einer UUID der Version 1 oder 7. Bei anderen Versionen gibt diese Funktion `null` zurück. Beachten Sie, dass der extrahierte Zeitstempel nicht notwendigerweise exakt der Erzeugungszeit der UUID entspricht; das hängt von der Implementierung ab, die die UUID erzeugt hat. | `uuid_extract_timestamp('019535d9-3df7-79fb-b466-fa907fa17f9e'::uuid)` -> `2025-02-23 21:46:24.503-05` |
| `uuid_extract_version ( uuid )` -> `smallint` | Extrahiert die Version aus einer UUID einer der in [RFC 9562](https://datatracker.ietf.org/doc/html/rfc9562) beschriebenen Varianten. Bei anderen Varianten gibt diese Funktion `null` zurück. Für eine von `gen_random_uuid()` erzeugte UUID gibt diese Funktion zum Beispiel `4` zurück. | `uuid_extract_version('41db1265-8bc1-4ab3-992f-885799a4af1d'::uuid)` -> `4`<br>`uuid_extract_version('019535d9-3df7-79fb-b466-fa907fa17f9e'::uuid)` -> `7` |

PostgreSQL stellt für UUIDs außerdem die üblichen Vergleichsoperatoren aus Tabelle 9.1 bereit.

Details zum Datentyp `uuid` in PostgreSQL finden Sie in [Abschnitt 8.12](08_Datentypen.md#812-uuidtyp).

## 9.15. XML-Funktionen

Die in diesem Abschnitt beschriebenen Funktionen und funktionsähnlichen Ausdrücke arbeiten mit Werten des Typs `xml`. Informationen zum Typ `xml` finden Sie in [Abschnitt 8.13](08_Datentypen.md#813-xmltyp). Die funktionsähnlichen Ausdrücke `xmlparse` und `xmlserialize` zum Konvertieren in den Typ `xml` und aus dem Typ `xml` heraus werden dort dokumentiert, nicht in diesem Abschnitt.

Die Verwendung der meisten dieser Funktionen setzt voraus, dass PostgreSQL mit `configure --with-libxml` gebaut wurde.

### 9.15.1. XML-Inhalt erzeugen

Es gibt eine Reihe von Funktionen und funktionsähnlichen Ausdrücken, mit denen XML-Inhalt aus SQL-Daten erzeugt werden kann. Sie eignen sich besonders dazu, Abfrageergebnisse als XML-Dokumente zu formatieren, die anschließend von Client-Anwendungen weiterverarbeitet werden.

#### 9.15.1.1. `xmltext`

```text
xmltext ( text ) -> xml
```

Die Funktion `xmltext` gibt einen XML-Wert mit einem einzelnen Textknoten zurück, der das Eingabeargument als Inhalt enthält. Vordefinierte Entitäten wie das kaufmännische Und (`&`), spitze Klammern (`< >`) und Anführungszeichen (`"`) werden maskiert.

Beispiel:

```sql
SELECT xmltext('< foo & bar >');
```

```text
         xmltext
-------------------------
 &lt; foo &amp; bar &gt;
```

#### 9.15.1.2. `xmlcomment`

```text
xmlcomment ( text ) -> xml
```

Die Funktion `xmlcomment` erzeugt einen XML-Wert, der einen XML-Kommentar mit dem angegebenen Text als Inhalt enthält. Der Text darf weder `--` enthalten noch mit `-` enden, weil das Ergebnis sonst kein gültiger XML-Kommentar wäre. Ist das Argument null, ist auch das Ergebnis null.

Beispiel:

```sql
SELECT xmlcomment('hello');
```

```text
  xmlcomment
--------------
 <!--hello-->
```

#### 9.15.1.3. `xmlconcat`

```text
xmlconcat ( xml [, ...] ) -> xml
```

Die Funktion `xmlconcat` verkettet eine Liste einzelner XML-Werte zu einem einzigen Wert, der ein XML-Inhaltsfragment enthält. Nullwerte werden ausgelassen; das Ergebnis ist nur dann null, wenn es keine von null verschiedenen Argumente gibt.

Beispiel:

```sql
SELECT xmlconcat('<abc/>', '<bar>foo</bar>');
```

```text
      xmlconcat
----------------------
 <abc/><bar>foo</bar>
```

XML-Deklarationen werden, falls vorhanden, folgendermaßen zusammengeführt: Wenn alle Argumentwerte dieselbe XML-Versionsdeklaration haben, wird diese Version im Ergebnis verwendet, andernfalls wird keine Version verwendet. Wenn alle Argumentwerte für die `standalone`-Deklaration den Wert `yes` haben, wird dieser Wert im Ergebnis verwendet. Wenn alle Argumentwerte eine `standalone`-Deklaration haben und mindestens eine davon `no` lautet, wird `no` verwendet. Andernfalls hat das Ergebnis keine `standalone`-Deklaration. Wenn für das Ergebnis eine `standalone`-Deklaration erforderlich ist, aber keine Versionsdeklaration vorliegt, wird eine Versionsdeklaration mit Version 1.0 verwendet, weil XML verlangt, dass eine XML-Deklaration eine Versionsdeklaration enthält. Encoding-Deklarationen werden in allen Fällen ignoriert und entfernt.

Beispiel:

```sql
SELECT xmlconcat('<?xml version="1.1"?><foo/>', '<?xml
 version="1.1" standalone="no"?><bar/>');
```

```text
             xmlconcat
-----------------------------------
 <?xml version="1.1"?><foo/><bar/>
```

#### 9.15.1.4. `xmlelement`

```text
xmlelement ( NAME name [, XMLATTRIBUTES ( attvalue [ AS attname ]
 [, ...] ) ] [, content [, ...]] ) -> xml
```

Der Ausdruck `xmlelement` erzeugt ein XML-Element mit dem angegebenen Namen, den angegebenen Attributen und dem angegebenen Inhalt. Die in der Syntax gezeigten Elemente `name` und `attname` sind einfache Bezeichner, keine Werte. Die Elemente `attvalue` und `content` sind Ausdrücke und können jeden PostgreSQL-Datentyp liefern. Die Argumente innerhalb von `XMLATTRIBUTES` erzeugen Attribute des XML-Elements; die `content`-Werte werden zu seinem Inhalt verkettet.

Beispiele:

```sql
SELECT xmlelement(name foo);
```

```text
 xmlelement
------------
 <foo/>
```

```sql
SELECT xmlelement(name foo, xmlattributes('xyz' as bar));
```

```text
    xmlelement
------------------
 <foo bar="xyz"/>
```

```sql
SELECT xmlelement(name foo, xmlattributes(current_date as bar),
 'cont', 'ent');
```

```text
             xmlelement
-------------------------------------
 <foo bar="2007-01-26">content</foo>
```

Element- und Attributnamen, die keine gültigen XML-Namen sind, werden maskiert, indem die betreffenden Zeichen durch die Sequenz `_xHHHH_` ersetzt werden; `HHHH` ist dabei der Unicode-Codepoint des Zeichens in hexadezimaler Schreibweise. Beispiel:

```sql
SELECT xmlelement(name "foo$bar", xmlattributes('xyz' as "a&b"));
```

```text
            xmlelement
----------------------------------
 <foo_x0024_bar a_x0026_b="xyz"/>
```

Ein expliziter Attributname muss nicht angegeben werden, wenn der Attributwert eine Spaltenreferenz ist; dann wird standardmäßig der Spaltenname als Attributname verwendet. In anderen Fällen muss das Attribut ausdrücklich benannt werden. Dieses Beispiel ist daher gültig:

```sql
CREATE TABLE test (a xml, b xml);
SELECT xmlelement(name test, xmlattributes(a, b)) FROM test;
```

Diese Beispiele sind dagegen nicht gültig:

```sql
SELECT xmlelement(name test, xmlattributes('constant'), a, b) FROM test;
SELECT xmlelement(name test, xmlattributes(func(a, b))) FROM test;
```

Elementinhalt wird, sofern angegeben, gemäß seinem Datentyp formatiert. Wenn der Inhalt selbst den Typ `xml` hat, können komplexe XML-Dokumente aufgebaut werden. Beispiel:

```sql
SELECT xmlelement(name foo, xmlattributes('xyz' as bar),
                            xmlelement(name abc),
                            xmlcomment('test'),
                            xmlelement(name xyz));
```

```text
                  xmlelement
----------------------------------------------
 <foo bar="xyz"><abc/><!--test--><xyz/></foo>
```

Inhalt anderer Typen wird als gültige XML-Zeichendaten formatiert. Das bedeutet insbesondere, dass die Zeichen `<`, `>` und `&` in Entitäten umgewandelt werden. Binärdaten vom Typ `bytea` werden je nach Einstellung des Konfigurationsparameters `xmlbinary` in Base64- oder Hex-Kodierung dargestellt. Das genaue Verhalten einzelner Datentypen kann sich weiterentwickeln, um die PostgreSQL-Abbildungen an die in SQL:2006 und später beschriebenen Abbildungen anzugleichen, wie in Abschnitt D.3.1.3 erläutert.

#### 9.15.1.5. `xmlforest`

```text
xmlforest ( content [ AS name ] [, ...] ) -> xml
```

Der Ausdruck `xmlforest` erzeugt einen XML-Forest, also eine Folge von Elementen, mit den angegebenen Namen und Inhalten. Wie bei `xmlelement` muss jeder Name ein einfacher Bezeichner sein; die Inhaltsausdrücke können jeden Datentyp haben.

Beispiele:

```sql
SELECT xmlforest('abc' AS foo, 123 AS bar);
```

```text
          xmlforest
------------------------------
 <foo>abc</foo><bar>123</bar>
```

```sql
SELECT xmlforest(table_name, column_name)
FROM information_schema.columns
WHERE table_schema = 'pg_catalog';
```

```text
                              xmlforest
---------------------------------------------------------------------
 <table_name>pg_authid</table_name><column_name>rolname</column_name>
 <table_name>pg_authid</table_name><column_name>rolsuper</column_name>
 ...
```

Wie das zweite Beispiel zeigt, kann der Elementname weggelassen werden, wenn der Inhaltswert eine Spaltenreferenz ist; in diesem Fall wird standardmäßig der Spaltenname verwendet. Andernfalls muss ein Name angegeben werden.

Elementnamen, die keine gültigen XML-Namen sind, werden wie oben für `xmlelement` gezeigt maskiert. Ebenso werden Inhaltsdaten maskiert, damit gültiger XML-Inhalt entsteht, sofern sie nicht bereits den Typ `xml` haben.

Beachten Sie, dass XML-Forests keine gültigen XML-Dokumente sind, wenn sie aus mehr als einem Element bestehen. Es kann daher sinnvoll sein, `xmlforest`-Ausdrücke in `xmlelement` einzuschließen.

#### 9.15.1.6. `xmlpi`

```text
xmlpi ( NAME name [, content ] ) -> xml
```

Der Ausdruck `xmlpi` erzeugt eine XML-Verarbeitungsanweisung. Wie bei `xmlelement` muss der Name ein einfacher Bezeichner sein, während der Inhaltsausdruck jeden Datentyp haben kann. Der Inhalt darf, falls vorhanden, die Zeichenfolge `?>` nicht enthalten.

Beispiel:

```sql
SELECT xmlpi(name php, 'echo "hello world";');
```

```text
            xmlpi
-----------------------------
 <?php echo "hello world";?>
```

#### 9.15.1.7. `xmlroot`

```text
xmlroot ( xml, VERSION {text|NO VALUE} [, STANDALONE {YES|NO|NO VALUE} ] ) -> xml
```

Der Ausdruck `xmlroot` ändert die Eigenschaften des Wurzelknotens eines XML-Wertes. Wenn eine Version angegeben ist, ersetzt sie den Wert in der Versionsdeklaration des Wurzelknotens; wenn eine `standalone`-Einstellung angegeben ist, ersetzt sie den Wert in dessen `standalone`-Deklaration.

```sql
SELECT xmlroot(xmlparse(document '<?xml version="1.1"?>
<content>abc</content>'),
               version '1.0', standalone yes);
```

```text
                xmlroot
----------------------------------------
 <?xml version="1.0" standalone="yes"?>
 <content>abc</content>
```

#### 9.15.1.8. `xmlagg`

```text
xmlagg ( xml ) -> xml
```

Die Funktion `xmlagg` ist, anders als die übrigen hier beschriebenen Funktionen, eine Aggregatfunktion. Sie verkettet die Eingabewerte des Aggregataufrufs ähnlich wie `xmlconcat`; die Verkettung erfolgt jedoch über Zeilen hinweg und nicht über Ausdrücke innerhalb einer einzelnen Zeile. Weitere Informationen zu Aggregatfunktionen finden Sie in [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen).

Beispiel:

```sql
CREATE TABLE test (y int, x xml);
INSERT INTO test VALUES (1, '<foo>abc</foo>');
INSERT INTO test VALUES (2, '<bar/>');
SELECT xmlagg(x) FROM test;
```

```text
        xmlagg
----------------------
 <foo>abc</foo><bar/>
```

Um die Reihenfolge der Verkettung festzulegen, kann dem Aggregataufruf eine `ORDER BY`-Klausel hinzugefügt werden, wie in Abschnitt 4.2.7 beschrieben. Beispiel:

```sql
SELECT xmlagg(x ORDER BY y DESC) FROM test;
```

```text
        xmlagg
----------------------
 <bar/><foo>abc</foo>
```

Der folgende nicht standardkonforme Ansatz wurde in früheren Versionen empfohlen und kann in bestimmten Fällen weiterhin nützlich sein:

```sql
SELECT xmlagg(x) FROM (SELECT * FROM test ORDER BY y DESC) AS tab;
```

```text
        xmlagg
----------------------
 <bar/><foo>abc</foo>
```

### 9.15.2. XML-Prädikate

Die in diesem Abschnitt beschriebenen Ausdrücke prüfen Eigenschaften von `xml`-Werten.

#### 9.15.2.1. `IS DOCUMENT`

```text
xml IS DOCUMENT -> boolean
```

Der Ausdruck `IS DOCUMENT` gibt true zurück, wenn der XML-Wert des Arguments ein korrektes XML-Dokument ist, false, wenn dies nicht der Fall ist, also wenn es sich um ein Inhaltsfragment handelt, und null, wenn das Argument null ist. Den Unterschied zwischen Dokumenten und Inhaltsfragmenten erläutert [Abschnitt 8.13](08_Datentypen.md#813-xmltyp).

#### 9.15.2.2. `IS NOT DOCUMENT`

```text
xml IS NOT DOCUMENT -> boolean
```

Der Ausdruck `IS NOT DOCUMENT` gibt false zurück, wenn der XML-Wert des Arguments ein korrektes XML-Dokument ist, true, wenn dies nicht der Fall ist, also wenn es sich um ein Inhaltsfragment handelt, und null, wenn das Argument null ist.

#### 9.15.2.3. `XMLEXISTS`

```text
XMLEXISTS ( text PASSING [BY {REF|VALUE}] xml [BY {REF|VALUE}] ) -> boolean
```

Die Funktion `xmlexists` wertet einen XPath-1.0-Ausdruck, das erste Argument, mit dem übergebenen XML-Wert als Kontextobjekt aus. Die Funktion gibt false zurück, wenn die Auswertung eine leere Knotenmenge ergibt, und true, wenn sie irgendeinen anderen Wert ergibt. Die Funktion gibt null zurück, wenn ein Argument null ist. Ein von null verschiedener Wert, der als Kontextobjekt übergeben wird, muss ein XML-Dokument sein, kein Inhaltsfragment und kein Nicht-XML-Wert.

Beispiel:

```sql
SELECT xmlexists('//town[text() = ''Toronto'']' PASSING BY VALUE
 '<towns><town>Toronto</town><town>Ottawa</town></towns>');
```

```text
 xmlexists
------------
 t
(1 row)
```

Die Klauseln `BY REF` und `BY VALUE` werden von PostgreSQL akzeptiert, aber ignoriert, wie in Abschnitt D.3.2 beschrieben.

Im SQL-Standard wertet die Funktion `xmlexists` einen Ausdruck in der XML-Query-Sprache aus; PostgreSQL erlaubt jedoch nur einen XPath-1.0-Ausdruck, wie in Abschnitt D.3.1 beschrieben.

#### 9.15.2.4. `xml_is_well_formed`

```text
xml_is_well_formed ( text ) -> boolean
xml_is_well_formed_document ( text ) -> boolean
xml_is_well_formed_content ( text ) -> boolean
```

Diese Funktionen prüfen, ob eine Textzeichenkette wohlgeformtes XML darstellt, und geben ein boolesches Ergebnis zurück. `xml_is_well_formed_document` prüft auf ein wohlgeformtes Dokument, während `xml_is_well_formed_content` auf wohlgeformten Inhalt prüft. `xml_is_well_formed` führt ersteres aus, wenn der Konfigurationsparameter `xmloption` auf `DOCUMENT` gesetzt ist, und letzteres, wenn er auf `CONTENT` gesetzt ist. Das bedeutet, dass `xml_is_well_formed` nützlich ist, um zu prüfen, ob eine einfache Typumwandlung in den Typ `xml` gelingen wird, während die beiden anderen Funktionen nützlich sind, um zu prüfen, ob die entsprechenden Varianten von `XMLPARSE` gelingen werden.

Beispiele:

```sql
SET xmloption TO DOCUMENT;
SELECT xml_is_well_formed('<>');
```

```text
 xml_is_well_formed
--------------------
 f
(1 row)
```

```sql
SELECT xml_is_well_formed('<abc/>');
```

```text
 xml_is_well_formed
--------------------
 t
(1 row)
```

```sql
SET xmloption TO CONTENT;
SELECT xml_is_well_formed('abc');
```

```text
 xml_is_well_formed
--------------------
 t
(1 row)
```

```sql
SELECT xml_is_well_formed_document('<pg:foo xmlns:pg="http://postgresql.org/stuff">bar</pg:foo>');
```

```text
 xml_is_well_formed_document
-----------------------------
 t
(1 row)
```

```sql
SELECT xml_is_well_formed_document('<pg:foo xmlns:pg="http://postgresql.org/stuff">bar</my:foo>');
```

```text
 xml_is_well_formed_document
-----------------------------
 f
(1 row)
```

Das letzte Beispiel zeigt, dass die Prüfungen auch einschließen, ob Namespaces korrekt zusammenpassen.

### 9.15.3. XML verarbeiten

Zur Verarbeitung von Werten des Datentyps `xml` bietet PostgreSQL die Funktionen `xpath` und `xpath_exists`, die XPath-1.0-Ausdrücke auswerten, sowie die Tabellenfunktion `XMLTABLE`.

#### 9.15.3.1. `xpath`

```text
xpath ( xpath text, xml xml [, nsarray text[] ] ) -> xml[]
```

Die Funktion `xpath` wertet den als `text` angegebenen XPath-1.0-Ausdruck `xpath` gegen den XML-Wert `xml` aus. Sie gibt ein Array von XML-Werten zurück, das der vom XPath-Ausdruck erzeugten Knotenmenge entspricht. Wenn der XPath-Ausdruck einen Skalarwert statt einer Knotenmenge zurückgibt, wird ein Array mit einem einzigen Element zurückgegeben.

Das zweite Argument muss ein wohlgeformtes XML-Dokument sein. Insbesondere muss es genau ein Wurzelelement haben.

Das optionale dritte Argument ist ein Array von Namespace-Abbildungen. Dieses Array sollte ein zweidimensionales `text`-Array sein, dessen zweite Achse die Länge 2 hat, also ein Array von Arrays, die jeweils genau zwei Elemente enthalten. Das erste Element jedes Eintrags ist der Namespace-Name beziehungsweise Alias, das zweite die Namespace-URI. Die in diesem Array angegebenen Aliase müssen nicht mit den Aliasen übereinstimmen, die im XML-Dokument selbst verwendet werden; sowohl im XML-Dokument als auch im Kontext der Funktion `xpath` sind Aliase lokal.

Beispiel:

```sql
SELECT xpath('/my:a/text()', '<my:a xmlns:my="http://example.com">test</my:a>',
             ARRAY[ARRAY['my', 'http://example.com']]);
```

```text
 xpath
--------
 {test}
(1 row)
```

Mit Standard-Namespaces, also anonymen Namespaces, kann man zum Beispiel so umgehen:

```sql
SELECT xpath('//mydefns:b/text()', '<a xmlns="http://example.com"><b>test</b></a>',
             ARRAY[ARRAY['mydefns', 'http://example.com']]);
```

```text
 xpath
--------
 {test}
(1 row)
```

#### 9.15.3.2. `xpath_exists`

```text
xpath_exists ( xpath text, xml xml [, nsarray text[] ] ) -> boolean
```

Die Funktion `xpath_exists` ist eine spezialisierte Form der Funktion `xpath`. Statt die einzelnen XML-Werte zurückzugeben, die den XPath-1.0-Ausdruck erfüllen, gibt sie einen booleschen Wert zurück, der angibt, ob die Abfrage erfüllt wurde, genauer: ob sie irgendeinen anderen Wert als eine leere Knotenmenge erzeugt hat. Diese Funktion entspricht dem Prädikat `XMLEXISTS`, bietet aber zusätzlich Unterstützung für ein Namespace-Abbildungsargument.

Beispiel:

```sql
SELECT xpath_exists('/my:a/text()', '<my:a xmlns:my="http://example.com">test</my:a>',
                   ARRAY[ARRAY['my', 'http://example.com']]);
```

```text
 xpath_exists
--------------
 t
(1 row)
```

#### 9.15.3.3. `xmltable`

```text
XMLTABLE (
    [ XMLNAMESPACES ( namespace_uri AS namespace_name [, ...] ), ]
    row_expression PASSING [BY {REF|VALUE}] document_expression [BY {REF|VALUE}]
    COLUMNS name { type [PATH column_expression]
                 [DEFAULT default_expression] [NOT NULL | NULL]
                 | FOR ORDINALITY }
            [, ...]
) -> setof record
```

Der Ausdruck `xmltable` erzeugt eine Tabelle aus einem XML-Wert, einem XPath-Filter zum Extrahieren von Zeilen und einer Menge von Spaltendefinitionen. Obwohl er syntaktisch einer Funktion ähnelt, kann er nur als Tabelle in der `FROM`-Klausel einer Abfrage erscheinen.

Die optionale Klausel `XMLNAMESPACES` gibt eine kommaseparierte Liste von Namespace-Definitionen an. Dabei ist jede `namespace_uri` ein Textausdruck und jeder `namespace_name` ein einfacher Bezeichner. Die Klausel legt die im Dokument verwendeten XML-Namespaces und deren Aliase fest. Eine Angabe für einen Standard-Namespace wird derzeit nicht unterstützt.

Das erforderliche Argument `row_expression` ist ein als Text angegebener XPath-1.0-Ausdruck. Er wird mit dem XML-Wert `document_expression` als Kontextobjekt ausgewertet, um eine Menge von XML-Knoten zu erhalten. Diese Knoten wandelt `xmltable` in Ausgabezeilen um. Es werden keine Zeilen erzeugt, wenn `document_expression` null ist, wenn `row_expression` eine leere Knotenmenge erzeugt oder wenn es irgendeinen anderen Wert als eine Knotenmenge erzeugt.

`document_expression` stellt das Kontextobjekt für `row_expression` bereit. Es muss ein wohlgeformtes XML-Dokument sein; Fragmente beziehungsweise Forests werden nicht akzeptiert. Die Klauseln `BY REF` und `BY VALUE` werden akzeptiert, aber ignoriert, wie in Abschnitt D.3.2 beschrieben.

Im SQL-Standard wertet die Funktion `xmltable` Ausdrücke in der XML-Query-Sprache aus; PostgreSQL erlaubt jedoch nur XPath-1.0-Ausdrücke, wie in Abschnitt D.3.1 beschrieben.

Die erforderliche Klausel `COLUMNS` gibt die Spalten an, die in der Ausgabetabelle erzeugt werden. Für jede Spalte ist ein Name erforderlich, ebenso ein Datentyp, außer bei `FOR ORDINALITY`; dort ist der Typ `integer` implizit. Die Klauseln `PATH`, `DEFAULT` und zur Nullbarkeit sind optional.

Eine mit `FOR ORDINALITY` markierte Spalte wird mit Zeilennummern gefüllt, beginnend bei 1, in der Reihenfolge der Knoten, die aus der Ergebnisknotenmenge von `row_expression` gelesen werden. Höchstens eine Spalte darf mit `FOR ORDINALITY` markiert sein.

> **Hinweis:** XPath 1.0 legt für Knoten in einer Knotenmenge keine Reihenfolge fest. Code, der sich auf eine bestimmte Ergebnisreihenfolge verlässt, ist daher implementierungsabhängig. Einzelheiten stehen in Abschnitt D.3.1.2.

Der `column_expression` einer Spalte ist ein XPath-1.0-Ausdruck, der für jede Zeile mit dem aktuellen Knoten aus dem Ergebnis von `row_expression` als Kontextobjekt ausgewertet wird, um den Spaltenwert zu finden. Wenn kein `column_expression` angegeben ist, wird der Spaltenname als impliziter Pfad verwendet.

Wenn der XPath-Ausdruck einer Spalte einen Nicht-XML-Wert zurückgibt, in XPath 1.0 also eine Zeichenkette, einen booleschen Wert oder einen Double-Wert, und die Spalte einen anderen PostgreSQL-Typ als `xml` hat, wird die Spalte so gesetzt, als würde die Zeichenkettendarstellung des Wertes dem PostgreSQL-Typ zugewiesen. Bei booleschen Werten wird als Zeichenkettendarstellung `1` oder `0` verwendet, wenn die Typkategorie der Ausgabespalte numerisch ist, andernfalls `true` oder `false`.

Wenn der XPath-Ausdruck einer Spalte eine nicht leere Menge von XML-Knoten zurückgibt und der PostgreSQL-Typ der Spalte `xml` ist, wird das Ausdrucksergebnis genau zugewiesen, sofern es Dokument- oder Inhaltsform hat. Ein Ergebnis mit mehr als einem Elementknoten auf oberster Ebene oder mit Nicht-Leerraum-Text außerhalb eines Elements ist ein Beispiel für Inhaltsform. Ein XPath-Ergebnis kann auch weder Dokument- noch Inhaltsform haben, zum Beispiel wenn es einen Attributknoten aus dem umgebenden Element auswählt; ein solches Ergebnis wird in Inhaltsform gebracht, wobei jeder unzulässige Knoten durch seinen Zeichenkettenwert nach der XPath-1.0-Funktion `string` ersetzt wird.

Ein Nicht-XML-Ergebnis, das einer `xml`-Ausgabespalte zugewiesen wird, erzeugt Inhalt: einen einzelnen Textknoten mit dem Zeichenkettenwert des Ergebnisses. Ein XML-Ergebnis, das einer Spalte eines anderen Typs zugewiesen wird, darf nicht mehr als einen Knoten haben, sonst wird ein Fehler gemeldet. Gibt es genau einen Knoten, wird die Spalte so gesetzt, als würde der Zeichenkettenwert dieses Knotens dem PostgreSQL-Typ zugewiesen.

Der Zeichenkettenwert eines XML-Elements ist die Verkettung aller Textknoten, die in diesem Element und seinen Nachkommen enthalten sind, in Dokumentreihenfolge. Der Zeichenkettenwert eines Elements ohne nachgeordnete Textknoten ist die leere Zeichenkette, nicht `NULL`. Etwaige `xsi:nil`-Attribute werden ignoriert. Beachten Sie, dass ein nur aus Leerraum bestehender `text()`-Knoten zwischen zwei Nicht-Text-Elementen erhalten bleibt und dass führender Leerraum eines `text()`-Knotens nicht geglättet wird. Für die Regeln zum Zeichenkettenwert anderer XML-Knotentypen und Nicht-XML-Werte kann die XPath-1.0-Funktion `string` herangezogen werden.

Die hier beschriebenen Konvertierungsregeln entsprechen nicht exakt denen des SQL-Standards, wie in Abschnitt D.3.1.3 erläutert.

Wenn der Pfadausdruck für eine bestimmte Zeile eine leere Knotenmenge zurückgibt, typischerweise weil er nicht passt, wird die Spalte auf `NULL` gesetzt, sofern kein `default_expression` angegeben ist. Ist ein Default angegeben, wird der Wert verwendet, der aus der Auswertung dieses Ausdrucks entsteht.

Ein `default_expression` wird nicht sofort ausgewertet, wenn `xmltable` aufgerufen wird, sondern jedes Mal, wenn für die Spalte ein Default benötigt wird. Wenn der Ausdruck als stabil oder unveränderlich gilt, kann diese wiederholte Auswertung übersprungen werden. Daher können volatile Funktionen wie `nextval` sinnvoll in `default_expression` verwendet werden.

Spalten können als `NOT NULL` markiert werden. Wenn der `column_expression` einer `NOT NULL`-Spalte nichts findet und kein `DEFAULT` vorhanden ist oder auch `default_expression` zu null ausgewertet wird, wird ein Fehler gemeldet.

Beispiele:

```sql
CREATE TABLE xmldata AS SELECT
xml $$
<ROWS>
  <ROW id="1">
    <COUNTRY_ID>AU</COUNTRY_ID>
    <COUNTRY_NAME>Australia</COUNTRY_NAME>
  </ROW>
  <ROW id="5">
    <COUNTRY_ID>JP</COUNTRY_ID>
    <COUNTRY_NAME>Japan</COUNTRY_NAME>
    <PREMIER_NAME>Shinzo Abe</PREMIER_NAME>
    <SIZE unit="sq_mi">145935</SIZE>
  </ROW>
  <ROW id="6">
    <COUNTRY_ID>SG</COUNTRY_ID>
    <COUNTRY_NAME>Singapore</COUNTRY_NAME>
    <SIZE unit="sq_km">697</SIZE>
  </ROW>
</ROWS>
$$ AS data;

SELECT xmltable.*
  FROM xmldata,
       XMLTABLE('//ROWS/ROW'
                PASSING data
                COLUMNS id int PATH '@id',
                        ordinality FOR ORDINALITY,
                        "COUNTRY_NAME" text,
                        country_id text PATH 'COUNTRY_ID',
                        size_sq_km float PATH 'SIZE[@unit = "sq_km"]',
                        size_other text PATH 'concat(SIZE[@unit!="sq_km"], " ", SIZE[@unit!="sq_km"]/@unit)',
                        premier_name text PATH 'PREMIER_NAME' DEFAULT 'not specified');
```

```text
 id | ordinality | COUNTRY_NAME | country_id | size_sq_km | size_other  | premier_name
----+------------+--------------+------------+------------+-------------+---------------
  1 |          1 | Australia    | AU         |            |             | not specified
  5 |          2 | Japan        | JP         |            | 145935 sq_mi | Shinzo Abe
  6 |          3 | Singapore    | SG         |        697 |             | not specified
```

Das folgende Beispiel zeigt die Verkettung mehrerer `text()`-Knoten, die Verwendung des Spaltennamens als XPath-Filter und die Behandlung von Leerraum, XML-Kommentaren und Verarbeitungsanweisungen:

```sql
CREATE TABLE xmlelements AS SELECT
xml $$
  <root>
   <element> Hello<!-- xyxxz -->2a2<?aaaaa?> <!--x--> bbb<x>xxx</x>CC </element>
  </root>
$$ AS data;

SELECT xmltable.*
  FROM xmlelements, XMLTABLE('/root' PASSING data COLUMNS element text);
```

```text
         element
-------------------------
  Hello2a2   bbbxxxCC
```

Das folgende Beispiel zeigt, wie mit der Klausel `XMLNAMESPACES` eine Liste von Namespaces angegeben werden kann, die sowohl im XML-Dokument als auch in den XPath-Ausdrücken verwendet werden:

```sql
WITH xmldata(data) AS (VALUES ('
<example xmlns="http://example.com/myns" xmlns:B="http://example.com/b">
  <item foo="1" B:bar="2"/>
  <item foo="3" B:bar="4"/>
  <item foo="4" B:bar="5"/>
</example>'::xml)
)
SELECT xmltable.*
   FROM XMLTABLE(XMLNAMESPACES('http://example.com/myns' AS x,
                               'http://example.com/b' AS "B"),
              '/x:example/x:item'
                 PASSING (SELECT data FROM xmldata)
                 COLUMNS foo int PATH '@foo',
                         bar int PATH '@B:bar');
```

```text
 foo | bar
-----+-----
   1 |   2
   3 |   4
   4 |   5
(3 rows)
```

### 9.15.4. Tabellen auf XML abbilden

Die folgenden Funktionen bilden den Inhalt relationaler Tabellen auf XML-Werte ab. Man kann sie als XML-Exportfunktionalität verstehen:

```text
table_to_xml ( table regclass, nulls boolean,
               tableforest boolean, targetns text ) -> xml
query_to_xml ( query text, nulls boolean,
               tableforest boolean, targetns text ) -> xml
cursor_to_xml ( cursor refcursor, count integer, nulls boolean,
                tableforest boolean, targetns text ) -> xml
```

`table_to_xml` bildet den Inhalt der benannten Tabelle ab, die als Parameter `table` übergeben wird. Der Typ `regclass` akzeptiert Zeichenketten, die Tabellen in der üblichen Schreibweise bezeichnen, einschließlich optionaler Schemaqualifizierung und doppelter Anführungszeichen; Einzelheiten stehen in [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen). `query_to_xml` führt die als Parameter `query` übergebene Abfrage aus und bildet die Ergebnismenge ab. `cursor_to_xml` liest die angegebene Anzahl von Zeilen aus dem durch den Parameter `cursor` bezeichneten Cursor. Diese Variante wird empfohlen, wenn große Tabellen abgebildet werden müssen, weil jede Funktion den Ergebniswert im Speicher aufbaut.

Wenn `tableforest` false ist, sieht das resultierende XML-Dokument so aus:

```xml
<tablename>
  <row>
    <columnname1>data</columnname1>
    <columnname2>data</columnname2>
  </row>

  <row>
    ...
  </row>

  ...
</tablename>
```

Wenn `tableforest` true ist, ist das Ergebnis ein XML-Inhaltsfragment dieser Form:

```xml
<tablename>
  <columnname1>data</columnname1>
  <columnname2>data</columnname2>
</tablename>

<tablename>
  ...
</tablename>

...
```

Wenn kein Tabellenname verfügbar ist, also beim Abbilden einer Abfrage oder eines Cursors, wird im ersten Format die Zeichenkette `table` und im zweiten Format die Zeichenkette `row` verwendet.

Die Wahl zwischen diesen Formaten liegt beim Benutzer. Das erste Format ist ein korrektes XML-Dokument, was für viele Anwendungen wichtig ist. Das zweite Format ist bei der Funktion `cursor_to_xml` meist nützlicher, wenn die Ergebniswerte später wieder zu einem Dokument zusammengesetzt werden sollen. Die oben besprochenen Funktionen zum Erzeugen von XML-Inhalt, insbesondere `xmlelement`, können verwendet werden, um die Ergebnisse nach Bedarf zu verändern.

Die Datenwerte werden genauso abgebildet wie oben für die Funktion `xmlelement` beschrieben.

Der Parameter `nulls` bestimmt, ob Nullwerte in die Ausgabe aufgenommen werden. Ist er true, werden Nullwerte in Spalten so dargestellt:

```xml
<columnname xsi:nil="true"/>
```

Dabei ist `xsi` das XML-Namespace-Präfix für XML Schema Instance. Eine passende Namespace-Deklaration wird dem Ergebniswert hinzugefügt. Ist `nulls` false, werden Spalten mit Nullwerten einfach aus der Ausgabe ausgelassen.

Der Parameter `targetns` gibt den gewünschten XML-Namespace des Ergebnisses an. Wenn kein bestimmter Namespace gewünscht ist, sollte eine leere Zeichenkette übergeben werden.

Die folgenden Funktionen geben XML-Schema-Dokumente zurück, welche die von den entsprechenden obigen Funktionen vorgenommenen Abbildungen beschreiben:

```text
table_to_xmlschema ( table regclass, nulls boolean,
                     tableforest boolean, targetns text ) -> xml
query_to_xmlschema ( query text, nulls boolean,
                     tableforest boolean, targetns text ) -> xml
cursor_to_xmlschema ( cursor refcursor, nulls boolean,
                      tableforest boolean, targetns text ) -> xml
```

Es ist wesentlich, dieselben Parameter zu übergeben, damit zueinander passende XML-Datenabbildungen und XML-Schema-Dokumente entstehen.

Die folgenden Funktionen erzeugen XML-Datenabbildungen und das zugehörige XML-Schema gemeinsam in einem Dokument oder Forest und verknüpfen sie miteinander. Sie sind nützlich, wenn eigenständige und selbstbeschreibende Ergebnisse gewünscht werden:

```text
table_to_xml_and_xmlschema ( table regclass, nulls boolean,
                             tableforest boolean, targetns text ) -> xml
query_to_xml_and_xmlschema ( query text, nulls boolean,
                             tableforest boolean, targetns text ) -> xml
```

Außerdem sind die folgenden Funktionen verfügbar, um analoge Abbildungen ganzer Schemata oder der gesamten aktuellen Datenbank zu erzeugen:

```text
schema_to_xml ( schema name, nulls boolean,
                tableforest boolean, targetns text ) -> xml
schema_to_xmlschema ( schema name, nulls boolean,
                      tableforest boolean, targetns text ) -> xml
schema_to_xml_and_xmlschema ( schema name, nulls boolean,
                              tableforest boolean, targetns text ) -> xml

database_to_xml ( nulls boolean,
                  tableforest boolean, targetns text ) -> xml
database_to_xmlschema ( nulls boolean,
                        tableforest boolean, targetns text ) -> xml
database_to_xml_and_xmlschema ( nulls boolean,
                                tableforest boolean, targetns text ) -> xml
```

Diese Funktionen ignorieren Tabellen, die für den aktuellen Benutzer nicht lesbar sind. Die datenbankweiten Funktionen ignorieren außerdem Schemata, für die der aktuelle Benutzer kein `USAGE`-Privileg, also kein Lookup-Privileg, besitzt.

Beachten Sie, dass diese Funktionen potenziell sehr viele Daten erzeugen, die im Speicher aufgebaut werden müssen. Wenn Inhaltsabbildungen großer Schemata oder Datenbanken angefordert werden, kann es sinnvoll sein, die Tabellen stattdessen einzeln abzubilden, möglicherweise sogar über einen Cursor.

Das Ergebnis einer Inhaltsabbildung für ein Schema sieht so aus:

```xml
<schemaname>

table1-mapping

table2-mapping

...

</schemaname>
```

Das Format einer Tabellenabbildung hängt dabei vom Parameter `tableforest` ab, wie oben erläutert.

Das Ergebnis einer Inhaltsabbildung für eine Datenbank sieht so aus:

```xml
<dbname>

<schema1name>
  ...
</schema1name>

<schema2name>
  ...
</schema2name>

...

</dbname>
```

Dabei entspricht die Schemaabbildung der oben beschriebenen Form.

Als Beispiel für die Verwendung der Ausgabe dieser Funktionen zeigt Beispiel 9.1 ein XSLT-Stylesheet, das die Ausgabe von `table_to_xml_and_xmlschema` in ein HTML-Dokument umwandelt, das die Tabellendaten tabellarisch darstellt. Auf ähnliche Weise können die Ergebnisse dieser Funktionen in andere XML-basierte Formate umgewandelt werden.

**Beispiel 9.1. XSLT-Stylesheet zum Konvertieren von SQL/XML-Ausgabe nach HTML**

```xml
<?xml version="1.0"?>
<xsl:stylesheet version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns="http://www.w3.org/1999/xhtml"
>

  <xsl:output method="xml"
      doctype-system="http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd"
      doctype-public="-//W3C//DTD XHTML 1.0 Strict//EN"
      indent="yes"/>

  <xsl:template match="/*">
    <xsl:variable name="schema" select="//xsd:schema"/>
    <xsl:variable name="tabletypename"
                  select="$schema/xsd:element[@name=name(current())]/@type"/>
    <xsl:variable name="rowtypename"
                  select="$schema/xsd:complexType[@name=$tabletypename]/xsd:sequence/xsd:element[@name='row']/@type"/>

    <html>
      <head>
        <title><xsl:value-of select="name(current())"/></title>
      </head>
      <body>
        <table>
          <tr>
            <xsl:for-each select="$schema/xsd:complexType[@name=$rowtypename]/xsd:sequence/xsd:element/@name">
              <th><xsl:value-of select="."/></th>
            </xsl:for-each>
          </tr>

          <xsl:for-each select="row">
            <tr>
              <xsl:for-each select="*">
                <td><xsl:value-of select="."/></td>
              </xsl:for-each>
            </tr>
          </xsl:for-each>
        </table>
      </body>
    </html>
  </xsl:template>

</xsl:stylesheet>
```

## 9.16. JSON-Funktionen und -Operatoren

Dieser Abschnitt beschreibt:

- Funktionen und Operatoren zum Verarbeiten und Erzeugen von JSON-Daten,
- die SQL/JSON-Pfad-Sprache,
- die SQL/JSON-Abfragefunktionen.

Um JSON-Datentypen in der SQL-Umgebung nativ zu unterstützen, implementiert PostgreSQL das SQL/JSON-Datenmodell. Dieses Modell besteht aus Sequenzen von Elementen. Jedes Element kann SQL-Skalarwerte enthalten, ergänzt um einen SQL/JSON-Nullwert, sowie zusammengesetzte Datenstrukturen, die JSON-Arrays und JSON-Objekte verwenden. Das Modell formalisiert das in der JSON-Spezifikation [RFC 7159](https://datatracker.ietf.org/doc/html/rfc7159) implizierte Datenmodell.

SQL/JSON erlaubt es, JSON-Daten zusammen mit gewöhnlichen SQL-Daten und Transaktionsunterstützung zu behandeln, unter anderem für:

- das Hochladen von JSON-Daten in die Datenbank und das Speichern in normalen SQL-Spalten als Zeichen- oder Binärzeichenketten,
- das Erzeugen von JSON-Objekten und -Arrays aus relationalen Daten,
- das Abfragen von JSON-Daten mit SQL/JSON-Abfragefunktionen und Ausdrücken der SQL/JSON-Pfad-Sprache.

Weitere Informationen zum SQL/JSON-Standard finden Sie in [sqltr-19075-6]. Details zu den von PostgreSQL unterstützten JSON-Typen stehen in [Abschnitt 8.14](08_Datentypen.md#814-jsontypen).

### 9.16.1. JSON-Daten verarbeiten und erzeugen

Tabelle 9.47 zeigt die Operatoren, die mit JSON-Datentypen verwendet werden können; siehe [Abschnitt 8.14](08_Datentypen.md#814-jsontypen). Zusätzlich stehen die in Tabelle 9.1 gezeigten gewöhnlichen Vergleichsoperatoren für `jsonb` zur Verfügung, jedoch nicht für `json`. Die Vergleichsoperatoren folgen den Sortierregeln für B-Tree-Operationen, die in Abschnitt 8.14.4 umrissen werden. Siehe auch [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen) zu den Aggregatfunktionen `json_agg`, die Datensatzwerte als JSON aggregiert, `json_object_agg`, die Wertepaare zu einem JSON-Objekt aggregiert, sowie deren `jsonb`-Entsprechungen `jsonb_agg` und `jsonb_object_agg`.

**Tabelle 9.47. `json`- und `jsonb`-Operatoren**

| Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `json -> integer -> json`<br>`jsonb -> integer -> jsonb` | Extrahiert das n-te Element eines JSON-Arrays. Array-Elemente werden ab null indiziert; negative ganze Zahlen zählen vom Ende. | `'[{"a":"foo"},{"b":"bar"},{"c":"baz"}]'::json -> 2 -> {"c":"baz"}`<br>`'[{"a":"foo"},{"b":"bar"},{"c":"baz"}]'::json -> -3 -> {"a":"foo"}` |
| `json -> text -> json`<br>`jsonb -> text -> jsonb` | Extrahiert das JSON-Objektfeld mit dem angegebenen Schlüssel. | `'{"a": {"b":"foo"}}'::json -> 'a' -> {"b":"foo"}` |
| `json ->> integer -> text`<br>`jsonb ->> integer -> text` | Extrahiert das n-te Element eines JSON-Arrays als Text. | `'[1,2,3]'::json ->> 2 -> 3` |
| `json ->> text -> text`<br>`jsonb ->> text -> text` | Extrahiert das JSON-Objektfeld mit dem angegebenen Schlüssel als Text. | `'{"a":1,"b":2}'::json ->> 'b' -> 2` |
| `json #> text[] -> json`<br>`jsonb #> text[] -> jsonb` | Extrahiert das JSON-Unterobjekt am angegebenen Pfad; Pfadelemente können Feldschlüssel oder Array-Indizes sein. | `'{"a": {"b": ["foo","bar"]}}'::json #> '{a,b,1}' -> "bar"` |
| `json #>> text[] -> text`<br>`jsonb #>> text[] -> text` | Extrahiert das JSON-Unterobjekt am angegebenen Pfad als Text. | `'{"a": {"b": ["foo","bar"]}}'::json #>> '{a,b,1}' -> bar` |

> **Hinweis:** Die Operatoren zur Feld-, Element- und Pfadextraktion geben `NULL` zurück, statt fehlzuschlagen, wenn die JSON-Eingabe nicht die passende Struktur für die Anfrage hat, zum Beispiel wenn ein solcher Schlüssel oder ein solches Array-Element nicht existiert.

Einige weitere Operatoren gibt es nur für `jsonb`, wie Tabelle 9.48 zeigt. Abschnitt 8.14.4 beschreibt, wie diese Operatoren zur effizienten Suche in indizierten `jsonb`-Daten verwendet werden können.

**Tabelle 9.48. Zusätzliche `jsonb`-Operatoren**

| Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `jsonb @> jsonb -> boolean` | Enthält der erste JSON-Wert den zweiten? Siehe Abschnitt 8.14.3 für Details zum Enthaltensein. | `'{"a":1, "b":2}'::jsonb @> '{"b":2}'::jsonb -> t` |
| `jsonb <@ jsonb -> boolean` | Ist der erste JSON-Wert im zweiten enthalten? | `'{"b":2}'::jsonb <@ '{"a":1, "b":2}'::jsonb -> t` |
| `jsonb ? text -> boolean` | Existiert die Textzeichenkette als Top-Level-Schlüssel oder Array-Element innerhalb des JSON-Werts? | `'{"a":1, "b":2}'::jsonb ? 'b' -> t`<br>`'["a", "b", "c"]'::jsonb ? 'b' -> t` |
| `jsonb ?| text[] -> boolean` | Existiert irgendeine der Zeichenketten im Textarray als Top-Level-Schlüssel oder Array-Element? | `'{"a":1, "b":2, "c":3}'::jsonb ?| array['b', 'd'] -> t` |
| `jsonb ?& text[] -> boolean` | Existieren alle Zeichenketten im Textarray als Top-Level-Schlüssel oder Array-Element? | `'["a", "b", "c"]'::jsonb ?& array['a', 'b'] -> t` |
| `jsonb || jsonb -> jsonb` | Verkettet zwei `jsonb`-Werte. Zwei Arrays ergeben ein Array mit allen Elementen beider Eingaben. Zwei Objekte ergeben ein Objekt mit der Vereinigung ihrer Schlüssel; bei doppelten Schlüsseln gewinnt der Wert des zweiten Objekts. Alle anderen Fälle werden behandelt, indem eine Nicht-Array-Eingabe in ein einelementiges Array umgewandelt wird. Die Operation ist nicht rekursiv; nur die Top-Level-Array- oder Objektstruktur wird zusammengeführt. | `'["a", "b"]'::jsonb || '["a", "d"]'::jsonb -> ["a", "b", "a", "d"]`<br>`'{"a": "b"}'::jsonb || '{"c": "d"}'::jsonb -> {"a": "b", "c": "d"}`<br>`'[1, 2]'::jsonb || '3'::jsonb -> [1, 2, 3]`<br>`'{"a": "b"}'::jsonb || '42'::jsonb -> [{"a": "b"}, 42]`<br>Um ein Array als einzelnen Eintrag an ein anderes Array anzuhängen, umhüllen Sie es mit einer zusätzlichen Array-Schicht: `'[1, 2]'::jsonb || jsonb_build_array('[3, 4]'::jsonb) -> [1, 2, [3, 4]]` |
| `jsonb - text -> jsonb` | Löscht einen Schlüssel samt Wert aus einem JSON-Objekt oder passende Zeichenkettenwerte aus einem JSON-Array. | `'{"a": "b", "c": "d"}'::jsonb - 'a' -> {"c": "d"}`<br>`'["a", "b", "c", "b"]'::jsonb - 'b' -> ["a", "c"]` |
| `jsonb - text[] -> jsonb` | Löscht alle passenden Schlüssel oder Array-Elemente aus dem linken Operanden. | `'{"a": "b", "c": "d"}'::jsonb - '{a,c}'::text[] -> {}` |
| `jsonb - integer -> jsonb` | Löscht das Array-Element mit dem angegebenen Index; negative ganze Zahlen zählen vom Ende. Löst einen Fehler aus, wenn der JSON-Wert kein Array ist. | `'["a", "b"]'::jsonb - 1 -> ["a"]` |
| `jsonb #- text[] -> jsonb` | Löscht das Feld oder Array-Element am angegebenen Pfad; Pfadelemente können Feldschlüssel oder Array-Indizes sein. | `'["a", {"b":1}]'::jsonb #- '{1,b}' -> ["a", {}]` |
| `jsonb @? jsonpath -> boolean` | Gibt der JSON-Pfad für den angegebenen JSON-Wert irgendein Element zurück? Dies ist nur bei SQL-standardkonformen JSON-Pfadausdrücken nützlich, nicht bei Prädikatprüfausdrücken, da diese immer einen Wert zurückgeben. | `'{"a":[1,2,3,4,5]}'::jsonb @? '$.a[*] ? (@ > 2)' -> t` |
| `jsonb @@ jsonpath -> boolean` | Gibt das Ergebnis einer JSON-Pfad-Prädikatprüfung für den angegebenen JSON-Wert zurück. Dies ist nur bei Prädikatprüfausdrücken nützlich, nicht bei SQL-standardkonformen JSON-Pfadausdrücken, weil `NULL` zurückgegeben wird, wenn das Pfadergebnis kein einzelner boolescher Wert ist. | `'{"a":[1,2,3,4,5]}'::jsonb @@ '$.a[*] > 2' -> t` |

> **Hinweis:** Die `jsonpath`-Operatoren `@?` und `@@` unterdrücken die folgenden Fehler: fehlendes Objektfeld oder Array-Element, unerwarteter JSON-Elementtyp sowie Datums-/Zeit- und numerische Fehler. Die unten beschriebenen `jsonpath`-bezogenen Funktionen können ebenfalls angewiesen werden, solche Fehler zu unterdrücken. Dieses Verhalten kann hilfreich sein, wenn JSON-Dokumentsammlungen mit unterschiedlicher Struktur durchsucht werden.

Tabelle 9.49 zeigt die Funktionen zum Erzeugen von `json`- und `jsonb`-Werten. Einige Funktionen in dieser Tabelle haben eine `RETURNING`-Klausel, die den zurückgegebenen Datentyp angibt. Er muss einer von `json`, `jsonb`, `bytea`, ein Zeichenkettentyp (`text`, `char` oder `varchar`) oder ein Typ sein, der in `json` umgewandelt werden kann. Standardmäßig wird der Typ `json` zurückgegeben.

**Tabelle 9.49. JSON-Erzeugungsfunktionen**

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `to_json ( anyelement ) -> json`<br>`to_jsonb ( anyelement ) -> jsonb` | Wandelt jeden SQL-Wert in `json` oder `jsonb` um. Arrays und zusammengesetzte Werte werden rekursiv in Arrays und Objekte umgewandelt; mehrdimensionale Arrays werden in JSON zu Arrays von Arrays. Gibt es eine Typumwandlung vom SQL-Datentyp nach `json`, wird diese verwendet; andernfalls wird ein skalarer JSON-Wert erzeugt. Für jeden Skalar außer Zahl, Boolean oder Nullwert wird die Textdarstellung verwendet und bei Bedarf maskiert. | `to_json('Fred said "Hi."'::text) -> "Fred said \"Hi.\""`<br>`to_jsonb(row(42, 'Fred said "Hi."'::text)) -> {"f1": 42, "f2": "Fred said \"Hi.\""}` |
| `array_to_json ( anyarray [, boolean ] ) -> json` | Wandelt ein SQL-Array in ein JSON-Array um. Das Verhalten entspricht `to_json`, außer dass zwischen Top-Level-Array-Elementen Zeilenumbrüche eingefügt werden, wenn der optionale boolesche Parameter true ist. | `array_to_json('{{1,5},{99,100}}'::int[]) -> [[1,5],[99,100]]` |
| `json_array ( [ { value_expression [ FORMAT JSON ] } [, ...] ] [ { NULL | ABSENT } ON NULL ] [ RETURNING data_type [ FORMAT JSON [ ENCODING UTF8 ] ] ])`<br>`json_array ( [ query_expression ] [ RETURNING data_type [ FORMAT JSON [ ENCODING UTF8 ] ] ])` | Konstruiert ein JSON-Array entweder aus einer Reihe von `value_expression`-Parametern oder aus den Ergebnissen von `query_expression`, einer `SELECT`-Abfrage mit genau einer Spalte. Bei `ABSENT ON NULL` werden Nullwerte ignoriert. Das ist immer der Fall, wenn `query_expression` verwendet wird. | `json_array(1,true,json '{"a":null}') -> [1, true, {"a":null}]`<br>`json_array(SELECT * FROM (VALUES(1),(2)) t) -> [1, 2]` |
| `row_to_json ( record [, boolean ] ) -> json` | Wandelt einen zusammengesetzten SQL-Wert in ein JSON-Objekt um. Das Verhalten entspricht `to_json`, außer dass zwischen Top-Level-Elementen Zeilenumbrüche eingefügt werden, wenn der optionale boolesche Parameter true ist. | `row_to_json(row(1,'foo')) -> {"f1":1,"f2":"foo"}` |
| `json_build_array ( VARIADIC "any" ) -> json`<br>`jsonb_build_array ( VARIADIC "any" ) -> jsonb` | Baut aus einer variadischen Argumentliste ein JSON-Array, dessen Elemente unterschiedliche Typen haben können. Jedes Argument wird wie bei `to_json` oder `to_jsonb` konvertiert. | `json_build_array(1, 2, 'foo', 4, 5) -> [1, 2, "foo", 4, 5]` |
| `json_build_object ( VARIADIC "any" ) -> json`<br>`jsonb_build_object ( VARIADIC "any" ) -> jsonb` | Baut aus einer variadischen Argumentliste ein JSON-Objekt. Konventionsgemäß besteht die Argumentliste aus abwechselnden Schlüsseln und Werten. Schlüsselargumente werden zu Text gezwungen; Wertargumente werden wie bei `to_json` oder `to_jsonb` konvertiert. | `json_build_object('foo', 1, 2, row(3,'bar')) -> {"foo" : 1, "2" : {"f1":3,"f2":"bar"}}` |
| `json_object ( [ { key_expression { VALUE | ':' } value_expression [ FORMAT JSON [ ENCODING UTF8 ] ] }[, ...] ] [ { NULL | ABSENT } ON NULL ] [ { WITH | WITHOUT } UNIQUE [ KEYS ] ] [ RETURNING data_type [ FORMAT JSON [ ENCODING UTF8 ] ] ])` | Konstruiert ein JSON-Objekt aus allen angegebenen Schlüssel/Wert-Paaren oder ein leeres Objekt, wenn keine angegeben sind. `key_expression` ist ein skalarer Ausdruck für den JSON-Schlüssel und wird in `text` umgewandelt. Er darf weder `NULL` sein noch einem Typ angehören, der eine Typumwandlung nach `json` hat. Bei `WITH UNIQUE KEYS` darf es keine doppelten `key_expression`-Werte geben. Paare, deren `value_expression` `NULL` ergibt, werden bei `ABSENT ON NULL` ausgelassen; bei `NULL ON NULL` oder ohne Klausel wird der Schlüssel mit dem Wert `NULL` aufgenommen. | `json_object('code' VALUE 'P123', 'title': 'Jaws') -> {"code" : "P123", "title" : "Jaws"}` |
| `json_object ( text[] ) -> json`<br>`jsonb_object ( text[] ) -> jsonb` | Baut ein JSON-Objekt aus einem Textarray. Das Array muss entweder genau eine Dimension mit gerader Anzahl von Elementen haben, die als abwechselnde Schlüssel/Wert-Paare interpretiert werden, oder zwei Dimensionen, bei denen jedes innere Array genau zwei Elemente als Schlüssel/Wert-Paar enthält. Alle Werte werden in JSON-Zeichenketten umgewandelt. | `json_object('{a, 1, b, "def", c, 3.5}') -> {"a" : "1", "b" : "def", "c" : "3.5"}`<br>`json_object('{{a, 1}, {b, "def"}, {c, 3.5}}') -> {"a" : "1", "b" : "def", "c" : "3.5"}` |
| `json_object ( keys text[], values text[] ) -> json`<br>`jsonb_object ( keys text[], values text[] ) -> jsonb` | Diese Form von `json_object` nimmt Schlüssel und Werte paarweise aus getrennten Textarrays. Ansonsten ist sie mit der Ein-Argument-Form identisch. | `json_object('{a,b}', '{1,2}') -> {"a": "1", "b": "2"}` |
| `json ( expression [ FORMAT JSON [ ENCODING UTF8 ]] [ { WITH | WITHOUT } UNIQUE [ KEYS ]] ) -> json` | Wandelt den angegebenen Ausdruck, der als Text- oder `bytea`-Zeichenkette in UTF8 vorliegt, in einen JSON-Wert um. Ist `expression` `NULL`, wird ein SQL-Nullwert zurückgegeben. Bei `WITH UNIQUE` darf der Ausdruck keine doppelten Objektschlüssel enthalten. | `json('{"a":123, "b":[true,"foo"], "a":"bar"}') -> {"a":123, "b":[true,"foo"], "a":"bar"}` |
| `json_scalar ( expression )` | Wandelt einen SQL-Skalarwert in einen JSON-Skalarwert um. Ist die Eingabe `NULL`, wird SQL-Null zurückgegeben. Bei Zahlen oder booleschen Werten wird eine entsprechende JSON-Zahl oder ein JSON-Boolean zurückgegeben. Für jeden anderen Wert wird eine JSON-Zeichenkette zurückgegeben. | `json_scalar(123.45) -> 123.45`<br>`json_scalar(CURRENT_TIMESTAMP) -> "2022-05-10T10:51:04.62128-04:00"` |
| `json_serialize ( expression [ FORMAT JSON [ ENCODING UTF8 ] ] [ RETURNING data_type [ FORMAT JSON [ ENCODING UTF8 ] ] ] )` | Wandelt einen SQL/JSON-Ausdruck in eine Zeichen- oder Binärzeichenkette um. Der Ausdruck kann einen beliebigen JSON-Typ, einen beliebigen Zeichenkettentyp oder `bytea` in UTF8-Kodierung haben. Der in `RETURNING` verwendete Rückgabetyp kann jeder Zeichenkettentyp oder `bytea` sein. Standard ist `text`. | `json_serialize('{ "a" : 1 } ' RETURNING bytea) -> \x7b20226122203a2031207d20` |

> Zum Beispiel stellt die Erweiterung `hstore` eine Typumwandlung von `hstore` nach `json` bereit, sodass `hstore`-Werte, die mit den JSON-Erzeugungsfunktionen konvertiert werden, als JSON-Objekte und nicht als primitive Zeichenkettenwerte dargestellt werden.

Tabelle 9.50 beschreibt SQL/JSON-Einrichtungen zum Testen von JSON.

**Tabelle 9.50. SQL/JSON-Testfunktionen**

| Funktionssignatur | Beschreibung | Beispiele |
| --- | --- | --- |
| `expression IS [ NOT ] JSON [ { VALUE | SCALAR | ARRAY | OBJECT } ] [ { WITH | WITHOUT } UNIQUE [ KEYS ] ]` | Dieses Prädikat prüft, ob `expression` als JSON geparst werden kann, optional mit angegebenem Typ. Bei `SCALAR`, `ARRAY` oder `OBJECT` wird geprüft, ob das JSON diesen konkreten Typ hat. Bei `WITH UNIQUE KEYS` wird zusätzlich jedes Objekt im Ausdruck auf doppelte Schlüssel geprüft. | Siehe die folgenden SQL-Beispiele. |

```sql
SELECT js,
  js IS JSON "json?",
  js IS JSON SCALAR "scalar?",
  js IS JSON OBJECT "object?",
  js IS JSON ARRAY "array?"
FROM (VALUES
       ('123'), ('"abc"'), ('{"a": "b"}'), ('[1,2]'),
       ('abc')) foo(js);
```

```text
     js      | json? | scalar? | object? | array?
------------+-------+---------+---------+--------
 123        | t     | t       | f       | f
 "abc"     | t     | t       | f       | f
 {"a": "b"}| t    | f       | t       | f
 [1,2]      | t     | f       | f       | t
 abc        | f     | f       | f       | f
```

```sql
SELECT js,
   js IS JSON OBJECT "object?",
   js IS JSON ARRAY "array?",
   js IS JSON ARRAY WITH UNIQUE KEYS "array w. UK?",
   js IS JSON ARRAY WITHOUT UNIQUE KEYS "array w/o UK?"
FROM (VALUES ('[{"a":"1"},
 {"b":"2","b":"3"}]')) foo(js);
```

```text
-[ RECORD 1 ]-+--------------------
js            | [{"a":"1"},       +
              | {"b":"2","b":"3"}]
object?       | f
array?        | t
array w. UK?  | f
array w/o UK? | t
```

Tabelle 9.51 zeigt die Funktionen zum Verarbeiten von `json`- und `jsonb`-Werten.

**Tabelle 9.51. JSON-Verarbeitungsfunktionen**

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `json_array_elements ( json ) -> setof json`<br>`jsonb_array_elements ( jsonb ) -> setof jsonb` | Erweitert das Top-Level-JSON-Array zu einer Menge von JSON-Werten. | `select * from json_array_elements('[1,true, [2,false]]')` gibt `true` und `[2,false]` zurück. |
| `json_array_elements_text ( json ) -> setof text`<br>`jsonb_array_elements_text ( jsonb ) -> setof text` | Erweitert das Top-Level-JSON-Array zu einer Menge von Textwerten. | `select * from json_array_elements_text('["foo", "bar"]')` gibt `foo` und `bar` zurück. |
| `json_array_length ( json ) -> integer`<br>`jsonb_array_length ( jsonb ) -> integer` | Gibt die Anzahl der Elemente im Top-Level-JSON-Array zurück. | `json_array_length('[1,2,3,{"f1":1,"f2":[5,6]},4]') -> 5`<br>`jsonb_array_length('[]') -> 0` |
| `json_each ( json ) -> setof record ( key text, value json )`<br>`jsonb_each ( jsonb ) -> setof record ( key text, value jsonb )` | Erweitert das Top-Level-JSON-Objekt zu einer Menge von Schlüssel/Wert-Paaren. | `select * from json_each('{"a":"foo", "b":"bar"}')` gibt die Paare `a | "foo"` und `b | "bar"` zurück. |
| `json_each_text ( json ) -> setof record ( key text, value text )`<br>`jsonb_each_text ( jsonb ) -> setof record ( key text, value text )` | Erweitert das Top-Level-JSON-Objekt zu einer Menge von Schlüssel/Wert-Paaren; die zurückgegebenen Werte haben den Typ `text`. | `select * from json_each_text('{"a":"foo", "b":"bar"}')` gibt `a | foo` und `b | bar` zurück. |
| `json_extract_path ( from_json json, VARIADIC path_elems text[] ) -> json`<br>`jsonb_extract_path ( from_json jsonb, VARIADIC path_elems text[] ) -> jsonb` | Extrahiert das JSON-Unterobjekt am angegebenen Pfad. Funktional entspricht dies dem Operator `#>`, aber der Pfad kann als variadische Liste bequemer zu schreiben sein. | `json_extract_path('{"f2":{"f3":1},"f4":{"f5":99,"f6":"foo"}}', 'f4', 'f6') -> "foo"` |
| `json_extract_path_text ( from_json json, VARIADIC path_elems text[] ) -> text`<br>`jsonb_extract_path_text ( from_json jsonb, VARIADIC path_elems text[] ) -> text` | Extrahiert das JSON-Unterobjekt am angegebenen Pfad als Text. Funktional entspricht dies dem Operator `#>>`. | `json_extract_path_text('{"f2":{"f3":1},"f4":{"f5":99,"f6":"foo"}}', 'f4', 'f6') -> foo` |
| `json_object_keys ( json ) -> setof text`<br>`jsonb_object_keys ( jsonb ) -> setof text` | Gibt die Schlüsselmenge des Top-Level-JSON-Objekts zurück. | `select * from json_object_keys('{"f1":"abc","f2":{"f3":"a", "f4":"b"}}')` gibt `f1` und `f2` zurück. |
| `json_populate_record ( base anyelement, from_json json ) -> anyelement`<br>`jsonb_populate_record ( base anyelement, from_json jsonb ) -> anyelement` | Erweitert das Top-Level-JSON-Objekt zu einer Zeile mit dem zusammengesetzten Typ des Arguments `base`. JSON-Felder, deren Namen Spaltennamen des Ausgabetyps entsprechen, werden in diese Spalten übernommen; andere Felder werden ignoriert. Typischerweise ist `base` `NULL`, sodass nicht passende Ausgabespalten mit null gefüllt werden. Ist `base` nicht `NULL`, werden seine Werte für nicht passende Spalten verwendet. | Siehe die Regeln und das Beispiel unten. |

Für die Umwandlung eines JSON-Werts in den SQL-Typ einer Ausgabespalte werden nacheinander diese Regeln angewendet:

- Ein JSON-Nullwert wird immer in SQL-Null umgewandelt.
- Hat die Ausgabespalte den Typ `json` oder `jsonb`, wird der JSON-Wert exakt wiedergegeben.
- Ist die Ausgabespalte ein zusammengesetzter Zeilentyp und der JSON-Wert ein Objekt, werden die Felder des Objekts rekursiv nach diesen Regeln in Spalten des Ausgabetyps umgewandelt.
- Ist die Ausgabespalte ein Arraytyp und der JSON-Wert ein Array, werden die Array-Elemente rekursiv nach diesen Regeln in Elemente des Ausgabe-Arrays umgewandelt.
- Ist der JSON-Wert sonst eine Zeichenkette, wird der Inhalt der Zeichenkette an die Eingabekonvertierungsfunktion des Spaltendatentyps übergeben.
- Andernfalls wird die gewöhnliche Textdarstellung des JSON-Werts an die Eingabekonvertierungsfunktion des Spaltendatentyps übergeben.

Obwohl das folgende Beispiel einen konstanten JSON-Wert verwendet, würde man in typischer Verwendung eine `json`- oder `jsonb`-Spalte lateral aus einer anderen Tabelle in der `FROM`-Klausel referenzieren. `json_populate_record` in die `FROM`-Klausel zu schreiben, ist gute Praxis, weil dann alle extrahierten Spalten ohne doppelte Funktionsaufrufe verfügbar sind.

```sql
CREATE TYPE subrowtype AS (d int, e text);
CREATE TYPE myrowtype AS (a int, b text[], c subrowtype);
SELECT *
FROM json_populate_record(NULL::myrowtype,
  '{"a": 1, "b": ["2", "a b"], "c": {"d": 4, "e": "a b c"}, "x": "foo"}');
```

```text
 a |     b     |      c
---+-----------+-------------
 1 | {2,"a b"} | (4,"a b c")
```

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `jsonb_populate_record_valid ( base anyelement, from_json json ) -> boolean` | Testfunktion für `jsonb_populate_record`. Gibt true zurück, wenn der entsprechende Aufruf von `jsonb_populate_record` mit dem gegebenen JSON-Eingabeobjekt ohne Fehler beendet würde, andernfalls false. | Siehe Beispiel unten. |

```sql
CREATE TYPE jsb_char2 AS (a char(2));
SELECT jsonb_populate_record_valid(NULL::jsb_char2, '{"a": "aaa"}');
```

```text
 jsonb_populate_record_valid
-----------------------------
 f
(1 row)
```

```sql
SELECT * FROM jsonb_populate_record(NULL::jsb_char2, '{"a": "aaa"}') q;
```

```text
ERROR:  value too long for type character(2)
```

```sql
SELECT jsonb_populate_record_valid(NULL::jsb_char2, '{"a": "aa"}');
```

```text
 jsonb_populate_record_valid
-----------------------------
 t
(1 row)
```

```sql
SELECT * FROM jsonb_populate_record(NULL::jsb_char2, '{"a": "aa"}') q;
```

```text
 a
----
 aa
(1 row)
```

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `json_populate_recordset ( base anyelement, from_json json ) -> setof anyelement`<br>`jsonb_populate_recordset ( base anyelement, from_json jsonb ) -> setof anyelement` | Erweitert das Top-Level-JSON-Array von Objekten zu einer Menge von Zeilen mit dem zusammengesetzten Typ des Arguments `base`. Jedes Element des JSON-Arrays wird wie oben für `json[b]_populate_record` verarbeitet. | `CREATE TYPE twoints AS (a int, b int); SELECT * FROM json_populate_recordset(NULL::twoints, '[{"a":1,"b":2}, {"a":3,"b":4}]')` gibt die Zeilen `1 | 2` und `3 | 4` zurück. |
| `json_to_record ( json ) -> record`<br>`jsonb_to_record ( jsonb ) -> record` | Erweitert das Top-Level-JSON-Objekt zu einer Zeile mit dem durch eine `AS`-Klausel definierten zusammengesetzten Typ. Wie bei allen Funktionen, die `record` zurückgeben, muss die aufrufende Abfrage die Struktur des Datensatzes mit einer `AS`-Klausel ausdrücklich definieren. Das Ausgaberecord wird wie bei `json[b]_populate_record` aus den Feldern des JSON-Objekts gefüllt. Da es keinen Eingabedatensatzwert gibt, werden nicht passende Spalten immer mit null gefüllt. | `CREATE TYPE myrowtype AS (a int, b text); SELECT * FROM json_to_record('{"a":1,"b":[1,2,3],"c":[1,2,3],"e":"bar","r": {"a": 123, "b": "a b c"}}') AS x(a int, b text, c int[], d text, r myrowtype)` gibt `1 | [1,2,3] | {1,2,3} |   | (123,"a b c")` zurück. |
| `json_to_recordset ( json ) -> setof record`<br>`jsonb_to_recordset ( jsonb ) -> setof record` | Erweitert das Top-Level-JSON-Array von Objekten zu einer Menge von Zeilen mit dem durch eine `AS`-Klausel definierten zusammengesetzten Typ. Jedes Element des JSON-Arrays wird wie oben für `json[b]_populate_record` verarbeitet. | `select * from json_to_recordset('[{"a":1,"b":"foo"}, {"a":"2","c":"bar"}]') as x(a int, b text)` gibt `1 | foo` und `2 |` zurück. |
| `jsonb_set ( target jsonb, path text[], new_value jsonb [, create_if_missing boolean ] ) -> jsonb` | Gibt `target` zurück, wobei das durch `path` bezeichnete Element durch `new_value` ersetzt wird, oder `new_value` hinzugefügt wird, wenn `create_if_missing` true ist (Standard) und das bezeichnete Element nicht existiert. Alle früheren Pfadschritte müssen existieren, sonst wird `target` unverändert zurückgegeben. Negative ganze Zahlen im Pfad zählen vom Ende von JSON-Arrays. Ist der letzte Pfadschritt ein Array-Index außerhalb des gültigen Bereichs und `create_if_missing` true, wird der neue Wert bei negativem Index am Anfang, bei positivem Index am Ende des Arrays hinzugefügt. | `jsonb_set('[{"f1":1,"f2":null},2,null,3]', '{0,f1}', '[2,3,4]', false) -> [{"f1": [2, 3, 4], "f2": null}, 2, null, 3]`<br>`jsonb_set('[{"f1":1,"f2":null},2]', '{0,f3}', '[2,3,4]') -> [{"f1": 1, "f2": null, "f3": [2, 3, 4]}, 2]` |
| `jsonb_set_lax ( target jsonb, path text[], new_value jsonb [, create_if_missing boolean [, null_value_treatment text ]] ) -> jsonb` | Ist `new_value` nicht `NULL`, verhält sich die Funktion identisch zu `jsonb_set`. Andernfalls richtet sich das Verhalten nach `null_value_treatment`, das einen der Werte `raise_exception`, `use_json_null`, `delete_key` oder `return_target` haben muss. Standard ist `use_json_null`. | `jsonb_set_lax('[{"f1":1,"f2":null},2,null,3]', '{0,f1}', null) -> [{"f1": null, "f2": null}, 2, null, 3]`<br>`jsonb_set_lax('[{"f1":99,"f2":null},2]', '{0,f3}', null, true, 'return_target') -> [{"f1": 99, "f2": null}, 2]` |
| `jsonb_insert ( target jsonb, path text[], new_value jsonb [, insert_after boolean ] ) -> jsonb` | Gibt `target` mit eingefügtem `new_value` zurück. Bezeichnet der Pfad ein Array-Element, wird `new_value` vor diesem Element eingefügt, wenn `insert_after` false ist (Standard), oder danach, wenn `insert_after` true ist. Bezeichnet der Pfad ein Objektfeld, wird `new_value` nur eingefügt, wenn das Objekt diesen Schlüssel noch nicht enthält. Alle früheren Pfadschritte müssen existieren, sonst wird `target` unverändert zurückgegeben. Negative Indizes zählen vom Ende. Ist der letzte Pfadschritt ein Array-Index außerhalb des gültigen Bereichs, wird der neue Wert bei negativem Index am Anfang, bei positivem Index am Ende des Arrays hinzugefügt. | `jsonb_insert('{"a": [0,1,2]}', '{a, 1}', '"new_value"') -> {"a": [0, "new_value", 1, 2]}`<br>`jsonb_insert('{"a": [0,1,2]}', '{a, 1}', '"new_value"', true) -> {"a": [0, 1, "new_value", 2]}` |
| `json_strip_nulls ( target json [, strip_in_arrays boolean ] ) -> json`<br>`jsonb_strip_nulls ( target jsonb [, strip_in_arrays boolean ] ) -> jsonb` | Löscht rekursiv alle Objektfelder mit Nullwerten aus dem angegebenen JSON-Wert. Ist `strip_in_arrays` true (Standard ist false), werden auch Null-Array-Elemente entfernt. Bloße Nullwerte werden nie entfernt. | `json_strip_nulls('[{"f1":1, "f2":null}, 2, null, 3]') -> [{"f1":1},2,null,3]`<br>`jsonb_strip_nulls('[1,2,null,3,4]', true) -> [1,2,3,4]` |
| `jsonb_path_exists ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> boolean` | Prüft, ob der JSON-Pfad für den angegebenen JSON-Wert irgendein Element zurückgibt. Dies ist nur für SQL-standardkonforme JSON-Pfadausdrücke nützlich, nicht für Prädikatprüfausdrücke, da diese immer einen Wert zurückgeben. Wenn `vars` angegeben ist, muss es ein JSON-Objekt sein, dessen Felder benannte Werte zur Ersetzung im `jsonpath`-Ausdruck liefern. Ist `silent` angegeben und true, unterdrückt die Funktion dieselben Fehler wie die Operatoren `@?` und `@@`. | `jsonb_path_exists('{"a":[1,2,3,4,5]}', '$.a[*] ? (@ >= $min && @ <= $max)', '{"min":2, "max":4}') -> t` |
| `jsonb_path_match ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> boolean` | Gibt das SQL-boolesche Ergebnis einer JSON-Pfad-Prädikatprüfung für den angegebenen JSON-Wert zurück. Dies ist nur für Prädikatprüfausdrücke nützlich, nicht für SQL-standardkonforme JSON-Pfadausdrücke, weil die Funktion andernfalls fehlschlägt oder `NULL` zurückgibt, wenn das Pfadergebnis kein einzelner boolescher Wert ist. Die optionalen Argumente `vars` und `silent` verhalten sich wie bei `jsonb_path_exists`. | `jsonb_path_match('{"a":[1,2,3,4,5]}', 'exists($.a[*] ? (@ >= $min && @ <= $max))', '{"min":2, "max":4}') -> t` |
| `jsonb_path_query ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> setof jsonb` | Gibt alle JSON-Elemente zurück, die der JSON-Pfad für den angegebenen JSON-Wert zurückgibt. Bei SQL-standardkonformen JSON-Pfadausdrücken sind dies die aus `target` ausgewählten JSON-Werte. Bei Prädikatprüfausdrücken ist es das Ergebnis der Prädikatprüfung: true, false oder null. `vars` und `silent` verhalten sich wie bei `jsonb_path_exists`. | Siehe Beispiel unten. |

```sql
SELECT *
FROM jsonb_path_query('{"a":[1,2,3,4,5]}',
  '$.a[*] ? (@ >= $min && @ <= $max)', '{"min":2, "max":4}');
```

```text
 jsonb_path_query
------------------
 2
 3
 4
```

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `jsonb_path_query_array ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> jsonb` | Gibt alle vom JSON-Pfad zurückgegebenen JSON-Elemente als JSON-Array zurück. Die Parameter entsprechen `jsonb_path_query`. | `jsonb_path_query_array('{"a":[1,2,3,4,5]}', '$.a[*] ? (@ >= $min && @ <= $max)', '{"min":2, "max":4}') -> [2, 3, 4]` |
| `jsonb_path_query_first ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> jsonb` | Gibt das erste vom JSON-Pfad zurückgegebene JSON-Element für den angegebenen JSON-Wert zurück oder `NULL`, wenn es keine Ergebnisse gibt. Die Parameter entsprechen `jsonb_path_query`. | `jsonb_path_query_first('{"a":[1,2,3,4,5]}', '$.a[*] ? (@ >= $min && @ <= $max)', '{"min":2, "max":4}') -> 2` |
| `jsonb_path_exists_tz ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> boolean`<br>`jsonb_path_match_tz ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> boolean`<br>`jsonb_path_query_tz ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> setof jsonb`<br>`jsonb_path_query_array_tz ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> jsonb`<br>`jsonb_path_query_first_tz ( target jsonb, path jsonpath [, vars jsonb [, silent boolean ]] ) -> jsonb` | Diese Funktionen verhalten sich wie die oben beschriebenen Gegenstücke ohne Suffix `_tz`, unterstützen aber Vergleiche von Datums-/Zeitwerten, die zeitzonenbewusste Konvertierungen erfordern. Das Beispiel unten erfordert die Interpretation des reinen Datumswerts `2015-08-02` als Zeitstempel mit Zeitzone; das Ergebnis hängt daher von der aktuellen `TimeZone`-Einstellung ab. Wegen dieser Abhängigkeit sind diese Funktionen als stabil markiert und können nicht in Indizes verwendet werden. Ihre Gegenstücke sind unveränderlich und können daher in Indizes verwendet werden, lösen aber Fehler aus, wenn solche Vergleiche verlangt werden. | `jsonb_path_exists_tz('["2015-08-01 12:00:00-05"]', '$[*] ? (@.datetime() < "2015-08-02".datetime())') -> t` |
| `jsonb_pretty ( jsonb ) -> text` | Wandelt den angegebenen JSON-Wert in hübsch formatierten, eingerückten Text um. | Siehe Beispiel unten. |
| `json_typeof ( json ) -> text`<br>`jsonb_typeof ( jsonb ) -> text` | Gibt den Typ des Top-Level-JSON-Werts als Textzeichenkette zurück. Mögliche Typen sind `object`, `array`, `string`, `number`, `boolean` und `null`. Das Ergebnis `null` darf nicht mit SQL-`NULL` verwechselt werden; siehe Beispiele. | `json_typeof('-123.4') -> number`<br>`json_typeof('null'::json) -> null`<br>`json_typeof(NULL::json) IS NULL -> t` |

```sql
SELECT jsonb_pretty('[{"f1":1,"f2":null}, 2]'::jsonb);
```

```text
[
    {
        "f1": 1,
        "f2": null
    },
    2
]
```

### 9.16.2. Die SQL/JSON-Pfad-Sprache

SQL/JSON-Pfadausdrücke geben Elemente an, die aus einem JSON-Wert gelesen werden sollen, ähnlich wie XPath-Ausdrücke beim Zugriff auf XML-Inhalt. In PostgreSQL werden Pfadausdrücke als Datentyp `jsonpath` implementiert und können alle in Abschnitt 8.14.7 beschriebenen Elemente verwenden.

JSON-Abfragefunktionen und -operatoren übergeben den angegebenen Pfadausdruck zur Auswertung an die Pfad-Engine. Wenn der Ausdruck zu den abgefragten JSON-Daten passt, wird das entsprechende JSON-Element oder eine Menge von Elementen zurückgegeben. Gibt es keine Übereinstimmung, ist das Ergebnis je nach Funktion `NULL`, false oder ein Fehler. Pfadausdrücke werden in der SQL/JSON-Pfad-Sprache geschrieben und können arithmetische Ausdrücke und Funktionen enthalten.

Ein Pfadausdruck besteht aus einer Folge von Elementen, die der Datentyp `jsonpath` erlaubt. Normalerweise wird der Pfadausdruck von links nach rechts ausgewertet, aber mit Klammern kann die Reihenfolge der Operationen geändert werden. Bei erfolgreicher Auswertung entsteht eine Sequenz von JSON-Elementen; dieses Auswertungsergebnis wird an die JSON-Abfragefunktion zurückgegeben, die die angegebene Berechnung abschließt.

Um auf den abgefragten JSON-Wert, das Kontextobjekt, zu verweisen, verwenden Sie im Pfadausdruck die Variable `$`. Das erste Element eines Pfads muss immer `$` sein. Darauf können ein oder mehrere Accessor-Operatoren folgen, die Ebene für Ebene in die JSON-Struktur hinabsteigen, um Unterelemente des Kontextobjekts zu lesen. Jeder Accessor-Operator arbeitet auf den Ergebnissen des vorherigen Auswertungsschritts und erzeugt null, ein oder mehrere Ausgabeelemente je Eingabeelement.

Angenommen, Sie haben JSON-Daten eines GPS-Trackers, die Sie analysieren möchten:

```sql
SELECT '{
  "track": {
    "segments": [
      {
         "location":   [ 47.763, 13.4034 ],
         "start time": "2018-10-14 10:05:14",
         "HR": 73
      },
      {
         "location":   [ 47.706, 13.2635 ],
         "start time": "2018-10-14 10:39:21",
         "HR": 135
      }
    ]
  }
}' AS json \gset
```

Das obige Beispiel kann in `psql` kopiert werden, um die folgenden Beispiele vorzubereiten. Anschließend ersetzt `psql` `:'json'` durch eine passend quotierte Zeichenkettenkonstante, die den JSON-Wert enthält.

Um die verfügbaren Track-Segmente zu lesen, verwenden Sie den Accessor-Operator `.key`, um durch die umgebenden JSON-Objekte abzusteigen:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments');
```

```text
                                                   jsonb_path_query
---------------------------------------------------------------------------------------------------------------------
 [{"HR": 73, "location": [47.763, 13.4034], "start time": "2018-10-14 10:05:14"}, {"HR": 135, "location": [47.706, 13.2635], "start time": "2018-10-14 10:39:21"}]
```

Um den Inhalt eines Arrays zu lesen, verwenden Sie typischerweise den Operator `[*]`. Das folgende Beispiel gibt die Standortkoordinaten aller verfügbaren Track-Segmente zurück:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments[*].location');
```

```text
 jsonb_path_query
-------------------
 [47.763, 13.4034]
 [47.706, 13.2635]
```

Hier wurde mit dem gesamten JSON-Eingabewert (`$`) begonnen. Der Accessor `.track` wählte dann das JSON-Objekt zum Objektschlüssel `track`, `.segments` das JSON-Array zum Schlüssel `segments` innerhalb dieses Objekts, `[*]` jedes Element dieses Arrays und `.location` schließlich das JSON-Array zum Schlüssel `location` innerhalb jedes dieser Objekte. In diesem Beispiel hatten alle Objekte einen Schlüssel `location`; wäre dies bei einem Objekt nicht der Fall, hätte der Accessor `.location` für dieses Eingabeelement einfach keine Ausgabe erzeugt.

Um nur die Koordinaten des ersten Segments zurückzugeben, geben Sie den entsprechenden Index im Accessor-Operator `[]` an. Denken Sie daran, dass JSON-Array-Indizes bei 0 beginnen:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments[0].location');
```

```text
 jsonb_path_query
-------------------
 [47.763, 13.4034]
```

Das Ergebnis jedes Pfadauswertungsschritts kann mit einem oder mehreren der in Abschnitt 9.16.2.3 aufgeführten `jsonpath`-Operatoren und -Methoden weiterverarbeitet werden. Jedem Methodennamen muss ein Punkt vorangestellt werden. Zum Beispiel kann die Größe eines Arrays ermittelt werden:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments.size()');
```

```text
 jsonb_path_query
------------------
 2
```

Weitere Beispiele für `jsonpath`-Operatoren und -Methoden innerhalb von Pfadausdrücken finden Sie unten in Abschnitt 9.16.2.3.

Ein Pfad kann außerdem Filterausdrücke enthalten, die ähnlich wie die `WHERE`-Klausel in SQL arbeiten. Ein Filterausdruck beginnt mit einem Fragezeichen und enthält eine Bedingung in Klammern:

```text
? (condition)
```

Filterausdrücke müssen direkt hinter dem Pfadauswertungsschritt stehen, auf den sie angewendet werden sollen. Das Ergebnis dieses Schritts wird so gefiltert, dass nur die Elemente erhalten bleiben, die die angegebene Bedingung erfüllen. SQL/JSON definiert dreiwertige Logik; die Bedingung kann also true, false oder unknown ergeben. Der Wert unknown spielt dieselbe Rolle wie SQL-`NULL` und kann mit dem Prädikat `is unknown` geprüft werden. Weitere Pfadauswertungsschritte verwenden nur die Elemente, für die der Filterausdruck true zurückgegeben hat.

Die Funktionen und Operatoren, die in Filterausdrücken verwendet werden können, stehen in Tabelle 9.53. Innerhalb eines Filterausdrucks bezeichnet die Variable `@` den gerade betrachteten Wert, also ein Ergebnis des vorherigen Pfadschritts. Hinter `@` können Accessor-Operatoren geschrieben werden, um Bestandteile zu lesen.

Wenn Sie zum Beispiel alle Herzfrequenzwerte über 130 lesen möchten, geht das so:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments[*].HR ? (@ > 130)');
```

```text
 jsonb_path_query
------------------
 135
```

Um die Startzeiten von Segmenten mit solchen Werten zu erhalten, müssen die irrelevanten Segmente vor der Auswahl der Startzeiten herausgefiltert werden. Der Filterausdruck wird daher auf den vorherigen Schritt angewendet, und der in der Bedingung verwendete Pfad ist ein anderer:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments[*] ? (@.HR > 130)."start time"');
```

```text
   jsonb_path_query
-----------------------
 "2018-10-14 10:39:21"
```

Bei Bedarf können mehrere Filterausdrücke nacheinander verwendet werden. Das folgende Beispiel wählt die Startzeiten aller Segmente aus, die relevante Koordinaten und hohe Herzfrequenzwerte enthalten:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments[*] ? (@.location[1] < 13.4) ? (@.HR > 130)."start time"');
```

```text
   jsonb_path_query
-----------------------
 "2018-10-14 10:39:21"
```

Filterausdrücke auf verschiedenen Verschachtelungsebenen sind ebenfalls erlaubt. Das folgende Beispiel filtert zunächst alle Segmente nach Standort und gibt dann, sofern vorhanden, hohe Herzfrequenzwerte für diese Segmente zurück:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments[*] ? (@.location[1] < 13.4).HR ? (@ > 130)');
```

```text
 jsonb_path_query
------------------
 135
```

Filterausdrücke können auch ineinander verschachtelt werden. Dieses Beispiel gibt die Größe des Tracks zurück, wenn er Segmente mit hohen Herzfrequenzwerten enthält, andernfalls eine leere Sequenz:

```sql
SELECT jsonb_path_query(:'json', '$.track ? (exists(@.segments[*] ? (@.HR > 130))).segments.size()');
```

```text
 jsonb_path_query
------------------
 2
```

#### 9.16.2.1. Abweichungen vom SQL-Standard

Die PostgreSQL-Implementierung der SQL/JSON-Pfad-Sprache weist die folgenden Abweichungen vom SQL/JSON-Standard auf.

##### 9.16.2.1.1. Boolesche Prädikatprüfausdrücke

Als Erweiterung des SQL-Standards kann ein PostgreSQL-Pfadausdruck ein boolesches Prädikat sein, während der SQL-Standard Prädikate nur innerhalb von Filtern erlaubt. SQL-standardkonforme Pfadausdrücke geben die passenden Elemente des abgefragten JSON-Werts zurück; Prädikatprüfausdrücke geben dagegen das einzelne dreiwertige `jsonb`-Ergebnis des Prädikats zurück: true, false oder null. Zum Beispiel könnten wir diesen standardkonformen SQL-Filterausdruck schreiben:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments ?(@[*].HR > 130)');
```

```text
                                      jsonb_path_query
--------------------------------------------------------------------------------------------
 {"HR": 135, "location": [47.706, 13.2635], "start time": "2018-10-14 10:39:21"}
```

Der ähnliche Prädikatprüfausdruck gibt einfach true zurück, um anzuzeigen, dass eine Übereinstimmung existiert:

```sql
SELECT jsonb_path_query(:'json', '$.track.segments[*].HR > 130');
```

```text
 jsonb_path_query
------------------
 true
```

> **Hinweis:** Prädikatprüfausdrücke werden im Operator `@@` und in der Funktion `jsonb_path_match` benötigt und sollten nicht mit dem Operator `@?` oder der Funktion `jsonb_path_exists` verwendet werden.

##### 9.16.2.1.2. Interpretation regulärer Ausdrücke

Bei der Interpretation von Mustern regulärer Ausdrücke, die in `like_regex`-Filtern verwendet werden, gibt es kleinere Unterschiede, wie in Abschnitt 9.16.2.4 beschrieben.

#### 9.16.2.2. Strict- und Lax-Modus

Wenn Sie JSON-Daten abfragen, passt der Pfadausdruck möglicherweise nicht zur tatsächlichen JSON-Datenstruktur. Der Versuch, auf ein nicht vorhandenes Mitglied eines Objekts oder ein nicht vorhandenes Element eines Arrays zuzugreifen, wird als Strukturfehler definiert. SQL/JSON-Pfadausdrücke haben zwei Modi zur Behandlung von Strukturfehlern:

- `lax` (Standard): Die Pfad-Engine passt die abgefragten Daten implizit an den angegebenen Pfad an. Strukturfehler, die nicht wie unten beschrieben behoben werden können, werden unterdrückt und erzeugen keine Übereinstimmung.
- `strict`: Tritt ein Strukturfehler auf, wird ein Fehler ausgelöst.

Der Lax-Modus erleichtert den Abgleich von JSON-Dokument und Pfadausdruck, wenn die JSON-Daten nicht dem erwarteten Schema entsprechen. Wenn ein Operand die Anforderungen einer bestimmten Operation nicht erfüllt, kann er automatisch als SQL/JSON-Array umhüllt oder durch Umwandlung seiner Elemente in eine SQL/JSON-Sequenz entpackt werden, bevor die Operation ausgeführt wird. Außerdem entpacken Vergleichsoperatoren ihre Operanden im Lax-Modus automatisch, sodass SQL/JSON-Arrays direkt verglichen werden können. Ein Array der Größe 1 gilt als gleich seinem einzigen Element. Automatisches Entpacken wird nicht ausgeführt, wenn:

- der Pfadausdruck die Methoden `type()` oder `size()` enthält, die den Typ beziehungsweise die Anzahl der Elemente im Array zurückgeben,
- die abgefragten JSON-Daten verschachtelte Arrays enthalten. In diesem Fall wird nur das äußerste Array entpackt, während alle inneren Arrays unverändert bleiben. Implizites Entpacken kann also innerhalb jedes Pfadauswertungsschritts nur eine Ebene tief gehen.

Beim Abfragen der oben gezeigten GPS-Daten können Sie im Lax-Modus zum Beispiel davon abstrahieren, dass die Segmente als Array gespeichert sind:

```sql
SELECT jsonb_path_query(:'json', 'lax $.track.segments.location');
```

```text
 jsonb_path_query
-------------------
 [47.763, 13.4034]
 [47.706, 13.2635]
```

Im Strict-Modus muss der angegebene Pfad genau zur Struktur des abgefragten JSON-Dokuments passen. Dieser Pfadausdruck verursacht daher einen Fehler:

```sql
SELECT jsonb_path_query(:'json', 'strict $.track.segments.location');
```

```text
ERROR:  jsonpath member accessor can only be applied to an object
```

Um dasselbe Ergebnis wie im Lax-Modus zu erhalten, müssen Sie das Array `segments` ausdrücklich entpacken:

```sql
SELECT jsonb_path_query(:'json', 'strict $.track.segments[*].location');
```

```text
 jsonb_path_query
-------------------
 [47.763, 13.4034]
 [47.706, 13.2635]
```

Das Entpackungsverhalten des Lax-Modus kann zu überraschenden Ergebnissen führen. Zum Beispiel wählt die folgende Abfrage mit dem Accessor `.**` jeden `HR`-Wert zweimal aus:

```sql
SELECT jsonb_path_query(:'json', 'lax $.**.HR');
```

```text
 jsonb_path_query
------------------
 73
 135
 73
 135
```

Das passiert, weil der Accessor `.**` sowohl das Array `segments` als auch jedes seiner Elemente auswählt, während der Accessor `.HR` im Lax-Modus Arrays automatisch entpackt. Um überraschende Ergebnisse zu vermeiden, empfiehlt es sich, den Accessor `.**` nur im Strict-Modus zu verwenden. Die folgende Abfrage wählt jeden `HR`-Wert nur einmal aus:

```sql
SELECT jsonb_path_query(:'json', 'strict $.**.HR');
```

```text
 jsonb_path_query
------------------
 73
 135
```

Auch das Entpacken von Arrays kann unerwartete Ergebnisse liefern. Dieses Beispiel wählt alle Standortarrays aus:

```sql
SELECT jsonb_path_query(:'json', 'lax $.track.segments[*].location');
```

```text
 jsonb_path_query
-------------------
 [47.763, 13.4034]
 [47.706, 13.2635]
(2 rows)
```

Wie erwartet gibt es die vollständigen Arrays zurück. Wird jedoch ein Filterausdruck angewendet, werden die Arrays entpackt, um jedes Element auszuwerten, und es werden nur die passenden Elemente zurückgegeben:

```sql
SELECT jsonb_path_query(:'json', 'lax $.track.segments[*].location ?(@[*] > 15)');
```

```text
 jsonb_path_query
------------------
 47.763
 47.706
(2 rows)
```

Das geschieht, obwohl der Pfadausdruck die vollständigen Arrays auswählt. Verwenden Sie den Strict-Modus, um wieder die Arrays auszuwählen:

```sql
SELECT jsonb_path_query(:'json', 'strict $.track.segments[*].location ?(@[*] > 15)');
```

```text
 jsonb_path_query
-------------------
 [47.763, 13.4034]
 [47.706, 13.2635]
(2 rows)
```

#### 9.16.2.3. SQL/JSON-Pfadoperatoren und -methoden

Tabelle 9.52 zeigt die in `jsonpath` verfügbaren Operatoren und Methoden. Beachten Sie, dass unäre Operatoren und Methoden auf mehrere Werte angewendet werden können, die aus einem vorherigen Pfadschritt stammen, während binäre Operatoren wie Addition nur auf einzelne Werte angewendet werden können. Im Lax-Modus werden Methoden, die auf ein Array angewendet werden, für jeden Wert im Array ausgeführt. Ausnahmen sind `.type()` und `.size()`, die auf das Array selbst angewendet werden.

**Tabelle 9.52. `jsonpath`-Operatoren und -Methoden**

| Operator/Methode | Beschreibung | Beispiele |
| --- | --- | --- |
| `number + number -> number` | Addition | `jsonb_path_query('[2]', '$[0] + 3') -> 5` |
| `+ number -> number` | Unäres Plus, keine Operation; anders als Addition kann dies über mehrere Werte iterieren. | `jsonb_path_query_array('{"x": [2,3,4]}', '+ $.x') -> [2, 3, 4]` |
| `number - number -> number` | Subtraktion | `jsonb_path_query('[2]', '7 - $[0]') -> 5` |
| `- number -> number` | Negation; anders als Subtraktion kann dies über mehrere Werte iterieren. | `jsonb_path_query_array('{"x": [2,3,4]}', '- $.x') -> [-2, -3, -4]` |
| `number * number -> number` | Multiplikation | `jsonb_path_query('[4]', '2 * $[0]') -> 8` |
| `number / number -> number` | Division | `jsonb_path_query('[8.5]', '$[0] / 2') -> 4.2500000000000000` |
| `number % number -> number` | Modulo, also Rest | `jsonb_path_query('[32]', '$[0] % 10') -> 2` |
| `value.type() -> string` | Typ des JSON-Elements; siehe `json_typeof`. | `jsonb_path_query_array('[1, "2", {}]', '$[*].type()') -> ["number", "string", "object"]` |
| `value.size() -> number` | Größe des JSON-Elements, also Anzahl der Array-Elemente oder 1, wenn es kein Array ist. | `jsonb_path_query('{"m": [11, 15]}', '$.m.size()') -> 2` |
| `value.boolean() -> boolean` | Boolescher Wert, konvertiert aus einem JSON-Boolean, einer Zahl oder einer Zeichenkette. | `jsonb_path_query_array('[1, "yes", false]', '$[*].boolean()') -> [true, true, false]` |
| `value.string() -> string` | Zeichenkettenwert, konvertiert aus einem JSON-Boolean, einer Zahl, einer Zeichenkette oder einem Datums-/Zeitwert. | `jsonb_path_query_array('[1.23, "xyz", false]', '$[*].string()') -> ["1.23", "xyz", "false"]`<br>`jsonb_path_query('"2023-08-15 12:34:56"', '$.timestamp().string()') -> "2023-08-15T12:34:56"` |
| `value.double() -> number` | Ungefähre Gleitkommazahl, konvertiert aus einer JSON-Zahl oder -Zeichenkette. | `jsonb_path_query('{"len": "1.9"}', '$.len.double() * 2') -> 3.8` |
| `number.ceiling() -> number` | Nächste ganze Zahl größer oder gleich der angegebenen Zahl. | `jsonb_path_query('{"h": 1.3}', '$.h.ceiling()') -> 2` |
| `number.floor() -> number` | Nächste ganze Zahl kleiner oder gleich der angegebenen Zahl. | `jsonb_path_query('{"h": 1.7}', '$.h.floor()') -> 1` |
| `number.abs() -> number` | Absolutwert der angegebenen Zahl. | `jsonb_path_query('{"z": -0.3}', '$.z.abs()') -> 0.3` |
| `value.bigint() -> bigint` | Big-Integer-Wert, konvertiert aus einer JSON-Zahl oder -Zeichenkette. | `jsonb_path_query('{"len": "9876543219"}', '$.len.bigint()') -> 9876543219` |
| `value.decimal( [ precision [ , scale ] ] ) -> decimal` | Gerundeter Dezimalwert, konvertiert aus einer JSON-Zahl oder -Zeichenkette; `precision` und `scale` müssen ganzzahlige Werte sein. | `jsonb_path_query('1234.5678', '$.decimal(6, 2)') -> 1234.57` |
| `value.integer() -> integer` | Integer-Wert, konvertiert aus einer JSON-Zahl oder -Zeichenkette. | `jsonb_path_query('{"len": "12345"}', '$.len.integer()') -> 12345` |
| `value.number() -> numeric` | Numerischer Wert, konvertiert aus einer JSON-Zahl oder -Zeichenkette. | `jsonb_path_query('{"len": "123.45"}', '$.len.number()') -> 123.45` |
| `string.datetime() -> datetime_type` | Datums-/Zeitwert, konvertiert aus einer Zeichenkette; siehe Hinweis unten. | `jsonb_path_query('["2015-8-1", "2015-08-12"]', '$[*] ? (@.datetime() < "2015-08-2".datetime())') -> "2015-8-1"` |
| `string.datetime(template) -> datetime_type` | Datums-/Zeitwert, konvertiert aus einer Zeichenkette mit der angegebenen `to_timestamp`-Vorlage. | `jsonb_path_query_array('["12:30", "18:40"]', '$[*].datetime("HH24:MI")') -> ["12:30:00", "18:40:00"]` |
| `string.date() -> date` | Datumswert, konvertiert aus einer Zeichenkette. | `jsonb_path_query('"2023-08-15"', '$.date()') -> "2023-08-15"` |
| `string.time() -> time without time zone` | Uhrzeit ohne Zeitzone, konvertiert aus einer Zeichenkette. | `jsonb_path_query('"12:34:56"', '$.time()') -> "12:34:56"` |
| `string.time(precision) -> time without time zone` | Uhrzeit ohne Zeitzone, konvertiert aus einer Zeichenkette, mit auf die angegebene Genauigkeit angepassten Sekundenbruchteilen. | `jsonb_path_query('"12:34:56.789"', '$.time(2)') -> "12:34:56.79"` |
| `string.time_tz() -> time with time zone` | Uhrzeit mit Zeitzone, konvertiert aus einer Zeichenkette. | `jsonb_path_query('"12:34:56 +05:30"', '$.time_tz()') -> "12:34:56+05:30"` |
| `string.time_tz(precision) -> time with time zone` | Uhrzeit mit Zeitzone, konvertiert aus einer Zeichenkette, mit auf die angegebene Genauigkeit angepassten Sekundenbruchteilen. | `jsonb_path_query('"12:34:56.789 +05:30"', '$.time_tz(2)') -> "12:34:56.79+05:30"` |
| `string.timestamp() -> timestamp without time zone` | Zeitstempel ohne Zeitzone, konvertiert aus einer Zeichenkette. | `jsonb_path_query('"2023-08-15 12:34:56"', '$.timestamp()') -> "2023-08-15T12:34:56"` |
| `string.timestamp(precision) -> timestamp without time zone` | Zeitstempel ohne Zeitzone, konvertiert aus einer Zeichenkette, mit auf die angegebene Genauigkeit angepassten Sekundenbruchteilen. | `jsonb_path_query('"2023-08-15 12:34:56.789"', '$.timestamp(2)') -> "2023-08-15T12:34:56.79"` |
| `string.timestamp_tz() -> timestamp with time zone` | Zeitstempel mit Zeitzone, konvertiert aus einer Zeichenkette. | `jsonb_path_query('"2023-08-15 12:34:56 +05:30"', '$.timestamp_tz()') -> "2023-08-15T12:34:56+05:30"` |
| `string.timestamp_tz(precision) -> timestamp with time zone` | Zeitstempel mit Zeitzone, konvertiert aus einer Zeichenkette, mit auf die angegebene Genauigkeit angepassten Sekundenbruchteilen. | `jsonb_path_query('"2023-08-15 12:34:56.789 +05:30"', '$.timestamp_tz(2)') -> "2023-08-15T12:34:56.79+05:30"` |
| `object.keyvalue() -> array` | Die Schlüssel/Wert-Paare des Objekts, dargestellt als Array von Objekten mit den drei Feldern `key`, `value` und `id`; `id` ist ein eindeutiger Bezeichner des Objekts, zu dem das Schlüssel/Wert-Paar gehört. | `jsonb_path_query_array('{"x": "20", "y": 32}', '$.keyvalue()') -> [{"id": 0, "key": "x", "value": "20"}, {"id": 0, "key": "y", "value": 32}]` |

> **Hinweis:** Der Ergebnistyp der Methoden `datetime()` und `datetime(template)` kann `date`, `timetz`, `time`, `timestamptz` oder `timestamp` sein. Beide Methoden bestimmen ihren Ergebnistyp dynamisch.
>
> Die Methode `datetime()` versucht nacheinander, die Eingabezeichenkette gegen die ISO-Formate für `date`, `timetz`, `time`, `timestamptz` und `timestamp` abzugleichen. Sie stoppt beim ersten passenden Format und gibt den entsprechenden Datentyp aus.
>
> Die Methode `datetime(template)` bestimmt den Ergebnistyp anhand der Felder, die in der angegebenen Vorlagenzeichenkette verwendet werden.
>
> Die Methoden `datetime()` und `datetime(template)` verwenden dieselben Parseregeln wie die SQL-Funktion `to_timestamp`; siehe [Abschnitt 9.8](09_Funktionen_und_Operatoren.md#98-formatierungsfunktionen-für-datentypen). Es gibt drei Ausnahmen: Erstens erlauben diese Methoden keine nicht passenden Vorlagenmuster. Zweitens sind in der Vorlagenzeichenkette nur die folgenden Trennzeichen erlaubt: Minuszeichen, Punkt, Schrägstrich, Komma, Apostroph, Semikolon, Doppelpunkt und Leerzeichen. Drittens müssen Trennzeichen in der Vorlagenzeichenkette exakt zur Eingabezeichenkette passen.
>
> Wenn unterschiedliche Datums-/Zeittypen verglichen werden müssen, wird eine implizite Typumwandlung angewendet. Ein `date`-Wert kann in `timestamp` oder `timestamptz` umgewandelt werden, `timestamp` in `timestamptz` und `time` in `timetz`. Bis auf die erste hängen diese Umwandlungen jedoch von der aktuellen `TimeZone`-Einstellung ab und können daher nur innerhalb zeitzonenbewusster `jsonpath`-Funktionen ausgeführt werden. Dasselbe gilt für andere Datums-/Zeitmethoden, die Zeichenketten in Datums-/Zeittypen umwandeln und dabei die aktuelle `TimeZone`-Einstellung einbeziehen können.

Tabelle 9.53 zeigt die verfügbaren Elemente für Filterausdrücke.

**Tabelle 9.53. Elemente für `jsonpath`-Filterausdrücke**

| Prädikat/Wert | Beschreibung | Beispiele |
| --- | --- | --- |
| `value == value -> boolean` | Gleichheitsvergleich; dieser und die anderen Vergleichsoperatoren arbeiten auf allen JSON-Skalarwerten. | `jsonb_path_query_array('[1, "a", 1, 3]', '$[*] ? (@ == 1)') -> [1, 1]`<br>`jsonb_path_query_array('[1, "a", 1, 3]', '$[*] ? (@ == "a")') -> ["a"]` |
| `value != value -> boolean`<br>`value <> value -> boolean` | Ungleichheitsvergleich. | `jsonb_path_query_array('[1, 2, 1, 3]', '$[*] ? (@ != 1)') -> [2, 3]`<br>`jsonb_path_query_array('["a", "b", "c"]', '$[*] ? (@ <> "b")') -> ["a", "c"]` |
| `value < value -> boolean` | Kleiner-als-Vergleich. | `jsonb_path_query_array('[1, 2, 3]', '$[*] ? (@ < 2)') -> [1]` |
| `value <= value -> boolean` | Kleiner-gleich-Vergleich. | `jsonb_path_query_array('["a", "b", "c"]', '$[*] ? (@ <= "b")') -> ["a", "b"]` |
| `value > value -> boolean` | Größer-als-Vergleich. | `jsonb_path_query_array('[1, 2, 3]', '$[*] ? (@ > 2)') -> [3]` |
| `value >= value -> boolean` | Größer-gleich-Vergleich. | `jsonb_path_query_array('[1, 2, 3]', '$[*] ? (@ >= 2)') -> [2, 3]` |
| `true -> boolean` | JSON-Konstante true. | `jsonb_path_query('[{"name": "John", "parent": false}, {"name": "Chris", "parent": true}]', '$[*] ? (@.parent == true)') -> {"name": "Chris", "parent": true}` |
| `false -> boolean` | JSON-Konstante false. | `jsonb_path_query('[{"name": "John", "parent": false}, {"name": "Chris", "parent": true}]', '$[*] ? (@.parent == false)') -> {"name": "John", "parent": false}` |
| `null -> value` | JSON-Konstante null. Anders als in SQL funktioniert der Vergleich mit null normal. | `jsonb_path_query('[{"name": "Mary", "job": null}, {"name": "Michael", "job": "driver"}]', '$[*] ? (@.job == null) .name') -> "Mary"` |
| `boolean && boolean -> boolean` | Boolesches UND. | `jsonb_path_query('[1, 3, 7]', '$[*] ? (@ > 1 && @ < 5)') -> 3` |
| `boolean || boolean -> boolean` | Boolesches ODER. | `jsonb_path_query('[1, 3, 7]', '$[*] ? (@ < 1 || @ > 5)') -> 7` |
| `! boolean -> boolean` | Boolesches NICHT. | `jsonb_path_query('[1, 3, 7]', '$[*] ? (!(@ < 5))') -> 7` |
| `boolean is unknown -> boolean` | Prüft, ob eine boolesche Bedingung unknown ist. | `jsonb_path_query('[-1, 2, 7, "foo"]', '$[*] ? ((@ > 0) is unknown)') -> "foo"` |
| `string like_regex string [ flag string ] -> boolean` | Prüft, ob der erste Operand zum durch den zweiten Operanden angegebenen regulären Ausdruck passt, optional mit den durch eine Flag-Zeichenkette beschriebenen Änderungen; siehe Abschnitt 9.16.2.4. | `jsonb_path_query_array('["abc", "abd", "aBdC", "abdacb", "babc"]', '$[*] ? (@ like_regex "^ab.*c")') -> ["abc", "abdacb"]`<br>`jsonb_path_query_array('["abc", "abd", "aBdC", "abdacb", "babc"]', '$[*] ? (@ like_regex "^ab.*c" flag "i")') -> ["abc", "aBdC", "abdacb"]` |
| `string starts with string -> boolean` | Prüft, ob der zweite Operand ein Anfangsteilstring des ersten Operanden ist. | `jsonb_path_query('["John Smith", "Mary Stone", "Bob Johnson"]', '$[*] ? (@ starts with "John")') -> "John Smith"` |
| `exists ( path_expression ) -> boolean` | Prüft, ob ein Pfadausdruck auf mindestens ein SQL/JSON-Element passt. Gibt unknown zurück, wenn der Pfadausdruck zu einem Fehler führen würde; das zweite Beispiel nutzt dies, um im Strict-Modus einen Fehler wegen eines fehlenden Schlüssels zu vermeiden. | `jsonb_path_query('{"x": [1, 2], "y": [2, 4]}', 'strict $.* ? (exists (@ ? (@[*] > 2)))') -> [2, 4]`<br>`jsonb_path_query_array('{"value": 41}', 'strict $ ? (exists (@.name)) .name') -> []` |

#### 9.16.2.4. SQL/JSON-reguläre Ausdrücke

SQL/JSON-Pfadausdrücke erlauben mit dem Filter `like_regex`, Text gegen einen regulären Ausdruck zu prüfen. Die folgende SQL/JSON-Pfadabfrage würde zum Beispiel alle Zeichenketten in einem Array finden, die unabhängig von Groß-/Kleinschreibung mit einem englischen Vokal beginnen:

```text
$[*] ? (@ like_regex "^[aeiou]" flag "i")
```

Die optionale Flag-Zeichenkette kann eines oder mehrere der Zeichen `i`, `m`, `s` und `q` enthalten: `i` für Vergleich ohne Beachtung von Groß-/Kleinschreibung, `m`, damit `^` und `$` an Zeilenumbrüchen passen, `s`, damit `.` auf einen Zeilenumbruch passt, und `q`, um das gesamte Muster zu quoten, was das Verhalten auf eine einfache Teilzeichenkettensuche reduziert.

Der SQL/JSON-Standard entlehnt seine Definition regulärer Ausdrücke vom Operator `LIKE_REGEX`, der wiederum den XQuery-Standard verwendet. PostgreSQL unterstützt den Operator `LIKE_REGEX` derzeit nicht. Daher wird der Filter `like_regex` mit der in Abschnitt 9.7.3 beschriebenen POSIX-Regular-Expression-Engine implementiert. Das führt zu verschiedenen kleineren Abweichungen vom SQL/JSON-Standardverhalten, die in Abschnitt 9.7.3.8 katalogisiert sind. Beachten Sie jedoch, dass die dort beschriebenen Inkompatibilitäten bei Flag-Buchstaben nicht für SQL/JSON gelten, da SQL/JSON die XQuery-Flag-Buchstaben passend zur POSIX-Engine übersetzt.

Denken Sie daran, dass das Musterargument von `like_regex` ein JSON-Pfad-Stringliteral ist, das nach den Regeln aus Abschnitt 8.14.7 geschrieben wird. Das bedeutet insbesondere, dass Backslashes, die im regulären Ausdruck verwendet werden sollen, verdoppelt werden müssen. Um zum Beispiel Zeichenkettenwerte des Wurzeldokuments zu finden, die nur Ziffern enthalten:

```text
$.* ? (@ like_regex "^\\d+$")
```

### 9.16.3. SQL/JSON-Abfragefunktionen

Die in Tabelle 9.54 beschriebenen SQL/JSON-Funktionen `JSON_EXISTS()`, `JSON_QUERY()` und `JSON_VALUE()` können zum Abfragen von JSON-Dokumenten verwendet werden. Jede dieser Funktionen wendet einen `path_expression`, also eine SQL/JSON-Pfadabfrage, auf ein `context_item`, das Dokument, an. Weitere Details dazu, was `path_expression` enthalten kann, stehen in Abschnitt 9.16.2. Der `path_expression` kann außerdem Variablen referenzieren, deren Werte mit den jeweiligen Namen in der von jeder Funktion unterstützten `PASSING`-Klausel angegeben werden. `context_item` kann ein `jsonb`-Wert oder eine Zeichenkette sein, die erfolgreich nach `jsonb` umgewandelt werden kann.

**Tabelle 9.54. SQL/JSON-Abfragefunktionen**

| Funktionssignatur | Beschreibung | Beispiele |
| --- | --- | --- |
| `JSON_EXISTS (`<br>`context_item, path_expression`<br>`[ PASSING { value AS varname } [, ...]]`<br>`[{ TRUE | FALSE | UNKNOWN | ERROR } ON ERROR ]) -> boolean` | Gibt true zurück, wenn der auf `context_item` angewendete SQL/JSON-`path_expression` irgendwelche Elemente ergibt, andernfalls false. Die Klausel `ON ERROR` legt fest, was passiert, wenn bei der Auswertung von `path_expression` ein Fehler auftritt. `ERROR` löst den Fehler mit der passenden Meldung aus. Weitere Optionen sind die booleschen Werte `FALSE` oder `TRUE` sowie `UNKNOWN`, was tatsächlich SQL-`NULL` ist. Ohne `ON ERROR` wird standardmäßig der boolesche Wert false zurückgegeben. | `JSON_EXISTS(jsonb '{"key1": [1,2,3]}', 'strict $.key1[*] ? (@ > $x)' PASSING 2 AS x) -> t`<br>`JSON_EXISTS(jsonb '{"a": [1,2,3]}', 'lax $.a[5]' ERROR ON ERROR) -> f`<br>Siehe Fehlerbeispiel unten. |

```sql
SELECT JSON_EXISTS(jsonb '{"a": [1,2,3]}', 'strict $.a[5]' ERROR ON ERROR);
```

```text
ERROR:  jsonpath array subscript is out of bounds
```

| Funktionssignatur | Beschreibung | Beispiele |
| --- | --- | --- |
| `JSON_QUERY (`<br>`context_item, path_expression`<br>`[ PASSING { value AS varname } [, ...]]`<br>`[ RETURNING data_type [ FORMAT JSON [ ENCODING UTF8 ] ] ]`<br>`[ { WITHOUT | WITH { CONDITIONAL | [UNCONDITIONAL] } } [ ARRAY ] WRAPPER ]`<br>`[ { KEEP | OMIT } QUOTES [ ON SCALAR STRING ] ]`<br>`[ { ERROR | NULL | EMPTY { [ ARRAY ] | OBJECT } | DEFAULT expression } ON EMPTY ]`<br>`[ { ERROR | NULL | EMPTY { [ ARRAY ] | OBJECT } | DEFAULT expression } ON ERROR ]) -> jsonb` | Gibt das Ergebnis zurück, das entsteht, wenn der SQL/JSON-`path_expression` auf `context_item` angewendet wird. Standardmäßig wird das Ergebnis als `jsonb` zurückgegeben; mit `RETURNING` kann ein anderer Typ gewählt werden, in den das Ergebnis erfolgreich gezwungen werden kann. Wenn der Pfadausdruck mehrere Werte zurückgeben kann, müssen diese eventuell mit `WITH WRAPPER` umhüllt werden, damit eine gültige JSON-Zeichenkette entsteht; standardmäßig wird nicht umhüllt, also wie bei `WITHOUT WRAPPER`. `WITH WRAPPER` bedeutet standardmäßig `WITH UNCONDITIONAL WRAPPER`, sodass auch ein einzelner Ergebniswert umhüllt wird. Mit `WITH CONDITIONAL WRAPPER` wird nur bei mehreren Werten umhüllt. Mehrere Ergebniswerte sind bei `WITHOUT WRAPPER` ein Fehler. Bei skalaren Zeichenketten wird der Rückgabewert standardmäßig in Anführungszeichen gesetzt, damit er ein gültiger JSON-Wert ist; dies kann mit `KEEP QUOTES` ausdrücklich gemacht oder mit `OMIT QUOTES` unterdrückt werden. `OMIT QUOTES` darf nicht zusammen mit `WITH WRAPPER` angegeben werden, wenn das Ergebnis gültiges JSON bleiben soll. `ON EMPTY` legt das Verhalten bei leerer Ergebnismenge fest, `ON ERROR` das Verhalten bei Fehlern während Pfadauswertung, Typzwang oder Auswertung des `ON EMPTY`-Ausdrucks. `ERROR` löst einen Fehler aus; andere Optionen geben SQL-`NULL`, ein leeres Array, ein leeres Objekt oder einen benutzerdefinierten Default zurück. Ohne `ON EMPTY` oder `ON ERROR` wird SQL-`NULL` zurückgegeben. | `JSON_QUERY(jsonb '[1,[2,3],null]', 'lax $[*][$off]' PASSING 1 AS off WITH CONDITIONAL WRAPPER) -> 3`<br>`JSON_QUERY(jsonb '{"a": "[1, 2]"}', 'lax $.a' OMIT QUOTES) -> [1, 2]`<br>Siehe Fehlerbeispiel unten. |

```sql
SELECT JSON_QUERY(jsonb '{"a": "[1, 2]"}', 'lax $.a' RETURNING int[] OMIT QUOTES ERROR ON ERROR);
```

```text
ERROR:  malformed array literal: "[1, 2]"
DETAIL: Missing "]" after array dimensions.
```

| Funktionssignatur | Beschreibung | Beispiele |
| --- | --- | --- |
| `JSON_VALUE (`<br>`context_item, path_expression`<br>`[ PASSING { value AS varname } [, ...]]`<br>`[ RETURNING data_type ]`<br>`[ { ERROR | NULL | DEFAULT expression } ON EMPTY ]`<br>`[ { ERROR | NULL | DEFAULT expression } ON ERROR ]) -> text` | Gibt das Ergebnis zurück, das entsteht, wenn der SQL/JSON-`path_expression` auf `context_item` angewendet wird. Verwenden Sie `JSON_VALUE()` nur, wenn der extrahierte Wert erwartungsgemäß ein einzelnes skalares SQL/JSON-Element ist; mehrere Werte sind ein Fehler. Wenn der extrahierte Wert ein Objekt oder Array sein kann, verwenden Sie stattdessen `JSON_QUERY`. Standardmäßig wird das einzelne skalare Ergebnis als `text` zurückgegeben; mit `RETURNING` kann ein anderer Typ gewählt werden, in den das Ergebnis erfolgreich gezwungen werden kann. Die Klauseln `ON ERROR` und `ON EMPTY` haben ähnliche Semantik wie bei `JSON_QUERY`, unterscheiden sich aber in der Menge der Werte, die anstelle eines Fehlers zurückgegeben werden können. Von `JSON_VALUE` zurückgegebene skalare Zeichenketten haben ihre Anführungszeichen immer entfernt, äquivalent zu `OMIT QUOTES` in `JSON_QUERY`. | `JSON_VALUE(jsonb '"123.45"', '$' RETURNING float) -> 123.45`<br>`JSON_VALUE(jsonb '"03:04 2015-02-01"', '$.datetime("HH24:MI YYYY-MM-DD")' RETURNING date) -> 2015-02-01`<br>`JSON_VALUE(jsonb '[1,2]', 'strict $[$off]' PASSING 1 AS off) -> 2`<br>`JSON_VALUE(jsonb '[1,2]', 'strict $[*]' DEFAULT 9 ON ERROR) -> 9` |

> **Hinweis:** Der Ausdruck `context_item` wird durch eine implizite Typumwandlung in `jsonb` konvertiert, wenn er nicht bereits vom Typ `jsonb` ist. Beachten Sie jedoch, dass Parsing-Fehler, die während dieser Konvertierung auftreten, bedingungslos ausgelöst werden, also nicht gemäß der ausdrücklich angegebenen oder impliziten `ON ERROR`-Klausel behandelt werden.

> **Hinweis:** `JSON_VALUE()` gibt SQL-`NULL` zurück, wenn `path_expression` JSON-Null zurückgibt, während `JSON_QUERY()` JSON-Null unverändert zurückgibt.

### 9.16.4. `JSON_TABLE`

`JSON_TABLE` ist eine SQL/JSON-Funktion, die JSON-Daten abfragt und die Ergebnisse als relationale Sicht darstellt, auf die wie auf eine normale SQL-Tabelle zugegriffen werden kann. Sie können `JSON_TABLE` in der `FROM`-Klausel von `SELECT`, `UPDATE` oder `DELETE` sowie als Datenquelle in einer `MERGE`-Anweisung verwenden.

Ausgehend von JSON-Daten als Eingabe verwendet `JSON_TABLE` einen JSON-Pfadausdruck, um einen Teil der angegebenen Daten als Zeilenmuster für die konstruierte Sicht zu extrahieren. Jeder SQL/JSON-Wert, der durch das Zeilenmuster geliefert wird, dient als Quelle für eine eigene Zeile in der konstruierten Sicht.

Um das Zeilenmuster in Spalten aufzuteilen, stellt `JSON_TABLE` die Klausel `COLUMNS` bereit, welche das Schema der erzeugten Sicht definiert. Für jede Spalte kann ein eigener JSON-Pfadausdruck angegeben werden, der gegen das Zeilenmuster ausgewertet wird, um einen SQL/JSON-Wert zu erhalten, der zum Wert der angegebenen Spalte in einer bestimmten Ausgabezeile wird.

JSON-Daten, die auf einer verschachtelten Ebene des Zeilenmusters liegen, können mit der Klausel `NESTED PATH` extrahiert werden. Jede `NESTED PATH`-Klausel kann verwendet werden, um eine oder mehrere Spalten mit Daten aus einer verschachtelten Ebene des Zeilenmusters zu erzeugen. Diese Spalten werden mit einer `COLUMNS`-Klausel angegeben, die der obersten `COLUMNS`-Klausel ähnelt. Zeilen, die aus `NESTED COLUMNS` konstruiert werden, heißen Kindzeilen und werden mit der Zeile verbunden, die aus den Spalten der übergeordneten `COLUMNS`-Klausel konstruiert wurde, um die Zeile in der endgültigen Sicht zu erhalten. Kindspalten können wiederum eine `NESTED PATH`-Angabe enthalten, sodass Daten aus beliebigen Verschachtelungsebenen extrahiert werden können. Spalten, die von mehreren `NESTED PATH`-Klauseln auf derselben Ebene erzeugt werden, gelten als Geschwister; ihre Zeilen werden nach dem Join mit der übergeordneten Zeile per `UNION` kombiniert.

Die von `JSON_TABLE` erzeugten Zeilen werden lateral mit der Zeile verbunden, die sie erzeugt hat. Daher müssen Sie die konstruierte Sicht nicht ausdrücklich mit der ursprünglichen Tabelle verbinden, welche die JSON-Daten enthält.

Die Syntax lautet:

```text
JSON_TABLE (
     context_item, path_expression [ AS json_path_name ]
     [ PASSING { value AS varname } [, ...] ]
     COLUMNS ( json_table_column [, ...] )
     [ { ERROR | EMPTY [ARRAY]} ON ERROR ]
)
```

Dabei ist `json_table_column`:

```text
name FOR ORDINALITY
| name type
      [ FORMAT JSON [ENCODING UTF8]]
      [ PATH path_expression ]
      [ { WITHOUT | WITH { CONDITIONAL | [UNCONDITIONAL] } } [ ARRAY ] WRAPPER ]
      [ { KEEP | OMIT } QUOTES [ ON SCALAR STRING ] ]
      [ { ERROR | NULL | EMPTY { [ARRAY] | OBJECT } | DEFAULT expression } ON EMPTY ]
      [ { ERROR | NULL | EMPTY { [ARRAY] | OBJECT } | DEFAULT expression } ON ERROR ]
| name type EXISTS [ PATH path_expression ]
      [ { ERROR | TRUE | FALSE | UNKNOWN } ON ERROR ]
| NESTED [ PATH ] path_expression [ AS json_path_name ]
      COLUMNS ( json_table_column [, ...] )
```

Jedes Syntaxelement wird im Folgenden genauer beschrieben.

**`context_item, path_expression [ AS json_path_name ] [ PASSING { value AS varname } [, ...]]`**

`context_item` gibt das abzufragende Eingabedokument an, `path_expression` ist ein SQL/JSON-Pfadausdruck, der die Abfrage definiert, und `json_path_name` ist ein optionaler Name für `path_expression`. Die optionale `PASSING`-Klausel stellt Datenwerte für die Variablen bereit, die in `path_expression` erwähnt werden. Das Ergebnis der Eingabedatenauswertung mit diesen Elementen heißt Zeilenmuster und wird als Quelle für Zeilenwerte in der konstruierten Sicht verwendet.

**`COLUMNS ( json_table_column [, ...] )`**

Die Klausel `COLUMNS` definiert das Schema der konstruierten Sicht. Darin können Sie jede Spalte angeben, die mit einem SQL/JSON-Wert gefüllt werden soll, der durch Anwenden eines JSON-Pfadausdrucks auf das Zeilenmuster entsteht. `json_table_column` hat die folgenden Varianten.

**`name FOR ORDINALITY`**

Diese Variante fügt eine Ordinalitätsspalte hinzu, die fortlaufende Zeilennummern ab 1 bereitstellt. Jede `NESTED PATH`-Klausel erhält ihren eigenen Zähler für verschachtelte Ordinalitätsspalten.

**`name type [FORMAT JSON [ENCODING UTF8]] [ PATH path_expression ]`**

Diese Variante fügt einen SQL/JSON-Wert, der durch Anwenden von `path_expression` auf das Zeilenmuster entsteht, nach Umwandlung in den angegebenen `type` in die Ausgabezeile der Sicht ein.

Mit `FORMAT JSON` machen Sie ausdrücklich, dass Sie einen gültigen JSON-Wert erwarten. `FORMAT JSON` ist nur sinnvoll, wenn `type` einer von `bpchar`, `bytea`, `character varying`, `name`, `json`, `jsonb`, `text` oder eine Domain über einem dieser Typen ist.

Optional können Sie `WRAPPER`- und `QUOTES`-Klauseln angeben, um die Ausgabe zu formatieren. Beachten Sie, dass `OMIT QUOTES` ein ebenfalls angegebenes `FORMAT JSON` übersteuert, weil nicht quotierte Literale keine gültigen `json`-Werte darstellen.

Optional können Sie mit `ON EMPTY` und `ON ERROR` festlegen, ob ein Fehler ausgelöst oder der angegebene Wert zurückgegeben wird, wenn die JSON-Pfadauswertung leer ist beziehungsweise wenn während der JSON-Pfadauswertung oder beim Umwandeln des SQL/JSON-Werts in den angegebenen Typ ein Fehler auftritt. Standardmäßig wird in beiden Fällen `NULL` zurückgegeben.

> **Hinweis:** Diese Klausel wird intern in `JSON_VALUE` oder `JSON_QUERY` umgewandelt und hat dieselbe Semantik. `JSON_QUERY` wird verwendet, wenn der angegebene Typ kein Skalartyp ist oder wenn `FORMAT JSON`, eine `WRAPPER`-Klausel oder eine `QUOTES`-Klausel vorhanden ist.

**`name type EXISTS [ PATH path_expression ]`**

Diese Variante fügt einen booleschen Wert, der durch Anwenden von `path_expression` auf das Zeilenmuster entsteht, nach Umwandlung in den angegebenen Typ in die Ausgabezeile der Sicht ein. Der Wert entspricht der Aussage, ob die Anwendung des `PATH`-Ausdrucks auf das Zeilenmuster irgendwelche Werte ergibt. Der angegebene Typ sollte eine Typumwandlung vom Typ `boolean` besitzen.

Optional können Sie mit `ON ERROR` festlegen, ob bei einem Fehler während der JSON-Pfadauswertung oder beim Umwandeln des SQL/JSON-Werts in den angegebenen Typ ein Fehler ausgelöst oder der angegebene Wert zurückgegeben wird. Standard ist der boolesche Wert false.

> **Hinweis:** Diese Klausel wird intern in `JSON_EXISTS` umgewandelt und hat dieselbe Semantik.

**`NESTED [ PATH ] path_expression [ AS json_path_name ] COLUMNS ( json_table_column [, ...] )`**

Diese Variante extrahiert SQL/JSON-Werte aus verschachtelten Ebenen des Zeilenmusters, erzeugt eine oder mehrere Spalten gemäß der Unterklausel `COLUMNS` und fügt die extrahierten SQL/JSON-Werte in diese Spalten ein. Der Ausdruck `json_table_column` in der `COLUMNS`-Unterklausel verwendet dieselbe Syntax wie in der übergeordneten `COLUMNS`-Klausel.

Die Syntax von `NESTED PATH` ist rekursiv. Sie können also mehrere verschachtelte Ebenen hinabsteigen, indem Sie mehrere `NESTED PATH`-Unterklauseln ineinander angeben. So kann die Hierarchie von JSON-Objekten und -Arrays in einem einzigen Funktionsaufruf entpackt werden, statt mehrere `JSON_TABLE`-Ausdrücke in einer SQL-Anweisung zu verketten.

> **Hinweis:** In jeder oben beschriebenen Variante von `json_table_column` wird, wenn die `PATH`-Klausel fehlt, der Pfadausdruck `$.name` verwendet, wobei `name` der angegebene Spaltenname ist.

**`AS json_path_name`**

Der optionale `json_path_name` dient als Bezeichner des angegebenen `path_expression`. Der Name muss eindeutig sein und sich von den Spaltennamen unterscheiden.

**`{ ERROR | EMPTY } ON ERROR`**

Mit dem optionalen `ON ERROR` kann festgelegt werden, wie Fehler bei der Auswertung des obersten `path_expression` behandelt werden. Verwenden Sie `ERROR`, wenn Fehler ausgelöst werden sollen, und `EMPTY`, um eine leere Tabelle zurückzugeben, also eine Tabelle mit 0 Zeilen. Beachten Sie, dass diese Klausel keine Fehler betrifft, die bei der Auswertung von Spalten auftreten; dort hängt das Verhalten davon ab, ob für die betreffende Spalte eine `ON ERROR`-Klausel angegeben wurde.

Beispiele

In den folgenden Beispielen wird die folgende Tabelle mit JSON-Daten verwendet:

```sql
CREATE TABLE my_films ( js jsonb );

INSERT INTO my_films VALUES (
'{ "favorites" : [
   { "kind" : "comedy", "films" : [
     { "title" : "Bananas",
       "director" : "Woody Allen"},
     { "title" : "The Dinner Game",
       "director" : "Francis Veber" } ] },
   { "kind" : "horror", "films" : [
     { "title" : "Psycho",
       "director" : "Alfred Hitchcock" } ] },
   { "kind" : "thriller", "films" : [
     { "title" : "Vertigo",
       "director" : "Alfred Hitchcock" } ] },
   { "kind" : "drama", "films" : [
     { "title" : "Yojimbo",
       "director" : "Akira Kurosawa" } ] }
  ] }');
```

Die folgende Abfrage zeigt, wie `JSON_TABLE` verwendet wird, um die JSON-Objekte in der Tabelle `my_films` in eine Sicht mit Spalten für die im ursprünglichen JSON enthaltenen Schlüssel `kind`, `title` und `director` sowie eine Ordinalitätsspalte umzuwandeln:

```sql
SELECT jt.* FROM
 my_films,
 JSON_TABLE (js, '$.favorites[*]' COLUMNS (
   id FOR ORDINALITY,
   kind text PATH '$.kind',
   title text PATH '$.films[*].title' WITH WRAPPER,
   director text PATH '$.films[*].director' WITH WRAPPER)) AS jt;
```

```text
 id |   kind   |                     title                      |              director
----+----------+------------------------------------------------+-----------------------------------
  1 | comedy   | ["Bananas", "The Dinner Game"]               | ["Woody Allen", "Francis Veber"]
  2 | horror   | ["Psycho"]                                     | ["Alfred Hitchcock"]
  3 | thriller | ["Vertigo"]                                    | ["Alfred Hitchcock"]
  4 | drama    | ["Yojimbo"]                                    | ["Akira Kurosawa"]
(4 rows)
```

Die folgende Variante der obigen Abfrage zeigt die Verwendung von `PASSING`-Argumenten im Filter des obersten JSON-Pfadausdrucks sowie verschiedene Optionen für die einzelnen Spalten:

```sql
SELECT jt.* FROM
 my_films,
 JSON_TABLE (js, '$.favorites[*] ? (@.films[*].director == $filter)'
   PASSING 'Alfred Hitchcock' AS filter
     COLUMNS (
     id FOR ORDINALITY,
     kind text PATH '$.kind',
     title text FORMAT JSON PATH '$.films[*].title' OMIT QUOTES,
     director text PATH '$.films[*].director' KEEP QUOTES)) AS jt;
```

```text
 id |   kind   |  title  |      director
----+----------+---------+--------------------
  1 | horror   | Psycho  | "Alfred Hitchcock"
  2 | thriller | Vertigo | "Alfred Hitchcock"
(2 rows)
```

Die folgende Variante der obigen Abfrage zeigt die Verwendung von `NESTED PATH` zum Füllen der Spalten `title` und `director`. Sie veranschaulicht, wie diese Spalten mit den übergeordneten Spalten `id` und `kind` verbunden werden:

```sql
SELECT jt.* FROM
 my_films,
 JSON_TABLE ( js, '$.favorites[*] ? (@.films[*].director == $filter)'
   PASSING 'Alfred Hitchcock' AS filter
   COLUMNS (
    id FOR ORDINALITY,
    kind text PATH '$.kind',
    NESTED PATH '$.films[*]' COLUMNS (
      title text FORMAT JSON PATH '$.title' OMIT QUOTES,
      director text PATH '$.director' KEEP QUOTES))) AS jt;
```

```text
 id |   kind   |  title  |      director
----+----------+---------+--------------------
  1 | horror   | Psycho  | "Alfred Hitchcock"
  2 | thriller | Vertigo | "Alfred Hitchcock"
(2 rows)
```

Die folgende Abfrage ist dieselbe Abfrage, jedoch ohne den Filter im Wurzelpfad:

```sql
SELECT jt.* FROM
 my_films,
 JSON_TABLE ( js, '$.favorites[*]'
   COLUMNS (
    id FOR ORDINALITY,
    kind text PATH '$.kind',
    NESTED PATH '$.films[*]' COLUMNS (
      title text FORMAT JSON PATH '$.title' OMIT QUOTES,
      director text PATH '$.director' KEEP QUOTES))) AS jt;
```

```text
 id |   kind   |      title      |      director
----+----------+-----------------+--------------------
  1 | comedy   | Bananas         | "Woody Allen"
  1 | comedy   | The Dinner Game | "Francis Veber"
  2 | horror   | Psycho          | "Alfred Hitchcock"
  3 | thriller | Vertigo         | "Alfred Hitchcock"
  4 | drama    | Yojimbo         | "Akira Kurosawa"
(5 rows)
```

Das folgende Beispiel zeigt eine weitere Abfrage mit einem anderen JSON-Objekt als Eingabe. Sie zeigt den `UNION`-„Geschwister-Join“ zwischen den verschachtelten Pfaden `$.movies[*]` und `$.books[*]` sowie die Verwendung von `FOR ORDINALITY`-Spalten auf verschachtelten Ebenen, nämlich `movie_id`, `book_id` und `author_id`:

```sql
SELECT * FROM JSON_TABLE (
'{"favorites":
    [{"movies":
      [{"name": "One", "director": "John Doe"},
       {"name": "Two", "director": "Don Joe"}],
     "books":
      [{"name": "Mystery", "authors": [{"name": "Brown Dan"}]},
       {"name": "Wonder", "authors": [{"name": "Jun Murakami"},
 {"name":"Craig Doe"}]}]
}]}'::json, '$.favorites[*]'
COLUMNS (
  user_id FOR ORDINALITY,
  NESTED '$.movies[*]'
    COLUMNS (
    movie_id FOR ORDINALITY,
    mname text PATH '$.name',
    director text),
  NESTED '$.books[*]'
    COLUMNS (
      book_id FOR ORDINALITY,
      bname text PATH '$.name',
      NESTED '$.authors[*]'
        COLUMNS (
          author_id FOR ORDINALITY,
          author_name text PATH '$.name'))));
```

```text
 user_id | movie_id | mname | director | book_id |  bname  | author_id | author_name
---------+----------+-------+----------+---------+---------+-----------+--------------
       1 |        1 | One   | John Doe |         |         |           |
       1 |        2 | Two   | Don Joe  |         |         |           |
       1 |          |       |          |       1 | Mystery |         1 | Brown Dan
       1 |          |       |          |       2 | Wonder  |         1 | Jun Murakami
       1 |          |       |          |       2 | Wonder  |         2 | Craig Doe
(5 rows)
```

## 9.17. Sequenzmanipulationsfunktionen

Dieser Abschnitt beschreibt Funktionen zum Arbeiten mit Sequenzobjekten, die auch Sequenzgeneratoren oder einfach Sequenzen genannt werden. Sequenzobjekte sind spezielle einzeilige Tabellen, die mit `CREATE SEQUENCE` erzeugt werden. Sie werden häufig verwendet, um eindeutige Bezeichner für Tabellenzeilen zu erzeugen. Die in Tabelle 9.55 aufgeführten Sequenzfunktionen stellen einfache, mehrbenutzersichere Methoden bereit, um aufeinanderfolgende Sequenzwerte aus Sequenzobjekten zu erhalten.

**Tabelle 9.55. Sequenzfunktionen**

| Funktion | Beschreibung |
| --- | --- |
| `nextval ( regclass ) -> bigint` | Setzt das Sequenzobjekt auf seinen nächsten Wert und gibt diesen Wert zurück. Das geschieht atomar: Selbst wenn mehrere Sitzungen gleichzeitig `nextval` ausführen, erhält jede Sitzung sicher einen eigenen Sequenzwert. Wenn das Sequenzobjekt mit Standardparametern erzeugt wurde, geben aufeinanderfolgende `nextval`-Aufrufe aufeinanderfolgende Werte beginnend mit 1 zurück. Anderes Verhalten kann durch passende Parameter im Befehl `CREATE SEQUENCE` erreicht werden. Diese Funktion erfordert das Privileg `USAGE` oder `UPDATE` auf der Sequenz. |
| `setval ( regclass, bigint [, boolean ] ) -> bigint` | Setzt den aktuellen Wert des Sequenzobjekts und optional sein Flag `is_called`. Die Zwei-Parameter-Form setzt das Feld `last_value` der Sequenz auf den angegebenen Wert und `is_called` auf true; der nächste `nextval`-Aufruf erhöht die Sequenz also, bevor er einen Wert zurückgibt. Auch der von `currval` gemeldete Wert wird auf den angegebenen Wert gesetzt. In der Drei-Parameter-Form kann `is_called` auf true oder false gesetzt werden. true hat denselben Effekt wie die Zwei-Parameter-Form. Bei false gibt der nächste `nextval`-Aufruf genau den angegebenen Wert zurück; die Sequenzfortschreibung beginnt mit dem folgenden `nextval`. Außerdem wird der von `currval` gemeldete Wert in diesem Fall nicht geändert. Das von `setval` zurückgegebene Ergebnis ist einfach der Wert des zweiten Arguments. Diese Funktion erfordert das Privileg `UPDATE` auf der Sequenz. |
| `currval ( regclass ) -> bigint` | Gibt den Wert zurück, der in der aktuellen Sitzung zuletzt mit `nextval` für diese Sequenz erhalten wurde. Wenn `nextval` für diese Sequenz in der aktuellen Sitzung noch nie aufgerufen wurde, wird ein Fehler gemeldet. Da ein sitzungslokaler Wert zurückgegeben wird, ist die Antwort vorhersagbar, unabhängig davon, ob andere Sitzungen seitdem `nextval` ausgeführt haben. Diese Funktion erfordert das Privileg `USAGE` oder `SELECT` auf der Sequenz. |
| `lastval () -> bigint` | Gibt den Wert zurück, der in der aktuellen Sitzung zuletzt von `nextval` zurückgegeben wurde. Diese Funktion ist mit `currval` identisch, außer dass sie keinen Sequenznamen als Argument nimmt, sondern auf die Sequenz verweist, auf die `nextval` zuletzt in der aktuellen Sitzung angewendet wurde. Es ist ein Fehler, `lastval` aufzurufen, wenn `nextval` in der aktuellen Sitzung noch nicht aufgerufen wurde. Diese Funktion erfordert das Privileg `USAGE` oder `SELECT` auf der zuletzt verwendeten Sequenz. |

Beispiele für `setval`:

```sql
SELECT setval('myseq', 42);
SELECT setval('myseq', 42, true);
SELECT setval('myseq', 42, false);
```

```text
-- Der nächste nextval-Aufruf gibt 43 zurück.
-- Wie oben.
-- Der nächste nextval-Aufruf gibt 42 zurück.
```

> **Achtung:** Um gleichzeitige Transaktionen, die Nummern aus derselben Sequenz beziehen, nicht zu blockieren, wird ein von `nextval` erhaltener Wert nicht zur Wiederverwendung freigegeben, wenn die aufrufende Transaktion später abbricht. Das bedeutet, dass Transaktionsabbrüche oder Datenbankabstürze zu Lücken in der Folge zugewiesener Werte führen können. Das kann sogar ohne Transaktionsabbruch passieren. Zum Beispiel berechnet ein `INSERT` mit einer `ON CONFLICT`-Klausel das einzufügende Tupel, einschließlich aller erforderlichen `nextval`-Aufrufe, bevor ein Konflikt erkannt wird, der stattdessen zur `ON CONFLICT`-Regel führt. PostgreSQL-Sequenzobjekte können daher nicht verwendet werden, um „lückenlose“ Sequenzen zu erhalten.
>
> Ebenso sind durch `setval` vorgenommene Änderungen des Sequenzzustands sofort für andere Transaktionen sichtbar und werden nicht rückgängig gemacht, wenn die aufrufende Transaktion zurückgerollt wird.
>
> Wenn der Datenbankcluster abstürzt, bevor eine Transaktion mit einem `nextval`- oder `setval`-Aufruf bestätigt wurde, hat die Änderung des Sequenzzustands möglicherweise noch keinen dauerhaften Speicher erreicht. Dann ist ungewiss, ob die Sequenz nach dem Neustart des Clusters ihren ursprünglichen oder aktualisierten Zustand hat. Für die Nutzung der Sequenz innerhalb der Datenbank ist das harmlos, weil andere Effekte nicht bestätigter Transaktionen ebenfalls nicht sichtbar werden. Wenn Sie einen Sequenzwert jedoch dauerhaft außerhalb der Datenbank verwenden möchten, stellen Sie sicher, dass der `nextval`-Aufruf vorher bestätigt wurde.

Die Sequenz, auf die eine Sequenzfunktion angewendet werden soll, wird durch ein Argument vom Typ `regclass` angegeben; das ist einfach die OID der Sequenz im Systemkatalog `pg_class`. Sie müssen die OID jedoch nicht von Hand nachschlagen, weil der Eingabekonverter des Datentyps `regclass` diese Arbeit übernimmt. Details stehen in [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen).

## 9.18. Bedingte Ausdrücke

Dieser Abschnitt beschreibt die SQL-konformen bedingten Ausdrücke, die in PostgreSQL verfügbar sind.

> **Tipp:** Wenn Ihre Anforderungen über die Möglichkeiten dieser bedingten Ausdrücke hinausgehen, sollten Sie erwägen, eine serverseitige Funktion in einer ausdrucksstärkeren Programmiersprache zu schreiben.

> **Hinweis:** Obwohl `COALESCE`, `GREATEST` und `LEAST` syntaktisch Funktionen ähneln, sind sie keine gewöhnlichen Funktionen und können daher nicht mit expliziten `VARIADIC`-Array-Argumenten verwendet werden.

### 9.18.1. `CASE`

Der SQL-Ausdruck `CASE` ist ein allgemeiner bedingter Ausdruck, ähnlich wie `if`/`else`-Anweisungen in anderen Programmiersprachen:

```sql
CASE WHEN condition THEN result
     [WHEN ...]
     [ELSE result]
END
```

`CASE`-Klauseln können überall verwendet werden, wo ein Ausdruck gültig ist. Jede `condition` ist ein Ausdruck, der ein boolesches Ergebnis zurückgibt. Wenn das Ergebnis der Bedingung true ist, ist der Wert des `CASE`-Ausdrucks das `result`, das auf die Bedingung folgt; der Rest des `CASE`-Ausdrucks wird nicht verarbeitet. Ist das Ergebnis der Bedingung nicht true, werden nachfolgende `WHEN`-Klauseln auf dieselbe Weise geprüft. Wenn keine `WHEN`-Bedingung true ergibt, ist der Wert des `CASE`-Ausdrucks das Ergebnis der `ELSE`-Klausel. Wenn die `ELSE`-Klausel fehlt und keine Bedingung true ist, ist das Ergebnis null.

Ein Beispiel:

```sql
SELECT * FROM test;
```

```text
 a
---
 1
 2
 3
```

```sql
SELECT a,
       CASE WHEN a=1 THEN 'one'
            WHEN a=2 THEN 'two'
            ELSE 'other'
       END
FROM test;
```

```text
 a | case
---+-------
 1 | one
 2 | two
 3 | other
```

Die Datentypen aller Ergebnisausdrücke müssen in einen gemeinsamen Ausgabetyp konvertierbar sein. Weitere Details stehen in [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte).

Es gibt eine „einfache“ Form des `CASE`-Ausdrucks, eine Variante der allgemeinen Form oben:

```sql
CASE expression
    WHEN value THEN result
    [WHEN ...]
    [ELSE result]
END
```

Der erste Ausdruck wird berechnet und dann mit jedem der `value`-Ausdrücke in den `WHEN`-Klauseln verglichen, bis ein gleicher Wert gefunden wird. Wenn keine Übereinstimmung gefunden wird, wird das Ergebnis der `ELSE`-Klausel oder ein Nullwert zurückgegeben. Das ähnelt der `switch`-Anweisung in C.

Das obige Beispiel kann mit der einfachen `CASE`-Syntax so geschrieben werden:

```sql
SELECT a,
       CASE a WHEN 1 THEN 'one'
              WHEN 2 THEN 'two'
              ELSE 'other'
       END
FROM test;
```

```text
 a | case
---+-------
 1 | one
 2 | two
 3 | other
```

Ein `CASE`-Ausdruck wertet keine Teilausdrücke aus, die zur Bestimmung des Ergebnisses nicht benötigt werden. Dies ist zum Beispiel eine mögliche Möglichkeit, einen Fehler wegen Division durch null zu vermeiden:

```sql
SELECT ... WHERE CASE WHEN x <> 0 THEN y/x > 1.5 ELSE false END;
```

> **Hinweis:** Wie in Abschnitt 4.2.14 beschrieben, gibt es verschiedene Situationen, in denen Teilausdrücke eines Ausdrucks zu unterschiedlichen Zeitpunkten ausgewertet werden. Das Prinzip, dass `CASE` nur notwendige Teilausdrücke auswertet, ist daher nicht unumstößlich. Zum Beispiel führt ein konstanter Teilausdruck `1/0` normalerweise schon zur Planungszeit zu einem Fehler wegen Division durch null, selbst wenn er in einem `CASE`-Zweig steht, der zur Laufzeit nie betreten würde.

### 9.18.2. `COALESCE`

```sql
COALESCE(value [, ...])
```

Die Funktion `COALESCE` gibt das erste ihrer Argumente zurück, das nicht null ist. Null wird nur zurückgegeben, wenn alle Argumente null sind. Sie wird häufig verwendet, um beim Abrufen von Daten zur Anzeige einen Vorgabewert für Nullwerte einzusetzen, zum Beispiel:

```sql
SELECT COALESCE(description, short_description, '(none)') ...
```

Dies gibt `description` zurück, wenn es nicht null ist, andernfalls `short_description`, wenn dieses nicht null ist, andernfalls `(none)`.

Die Argumente müssen alle in einen gemeinsamen Datentyp konvertierbar sein; dieser wird der Typ des Ergebnisses. Details stehen in [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte).

Wie ein `CASE`-Ausdruck wertet `COALESCE` nur die Argumente aus, die zur Bestimmung des Ergebnisses benötigt werden; Argumente rechts vom ersten nicht-null-Argument werden also nicht ausgewertet. Diese SQL-Standardfunktion bietet ähnliche Möglichkeiten wie `NVL` und `IFNULL`, die in einigen anderen Datenbanksystemen verwendet werden.

### 9.18.3. `NULLIF`

```sql
NULLIF(value1, value2)
```

Die Funktion `NULLIF` gibt einen Nullwert zurück, wenn `value1` gleich `value2` ist; andernfalls gibt sie `value1` zurück. Damit kann die umgekehrte Operation zum oben gezeigten `COALESCE`-Beispiel ausgeführt werden:

```sql
SELECT NULLIF(value, '(none)') ...
```

Wenn in diesem Beispiel `value` gleich `(none)` ist, wird null zurückgegeben, andernfalls der Wert von `value`.

Die beiden Argumente müssen vergleichbare Typen haben. Genauer gesagt werden sie exakt so verglichen, als hätten Sie `value1 = value2` geschrieben; daher muss ein geeigneter `=`-Operator verfügbar sein.

Das Ergebnis hat denselben Typ wie das erste Argument, allerdings mit einer Feinheit: Tatsächlich zurückgegeben wird das erste Argument des impliziten `=`-Operators, und in manchen Fällen wurde dieses auf den Typ des zweiten Arguments angehoben. Zum Beispiel ergibt `NULLIF(1, 2.2)` den Typ `numeric`, weil es keinen Operator `integer = numeric`, sondern nur `numeric = numeric` gibt.

### 9.18.4. `GREATEST` und `LEAST`

```sql
GREATEST(value [, ...])
LEAST(value [, ...])
```

Die Funktionen `GREATEST` und `LEAST` wählen den größten beziehungsweise kleinsten Wert aus einer Liste beliebig vieler Ausdrücke. Die Ausdrücke müssen alle in einen gemeinsamen Datentyp konvertierbar sein; dieser wird der Typ des Ergebnisses. Details stehen in [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte).

Nullwerte in der Argumentliste werden ignoriert. Das Ergebnis ist nur dann `NULL`, wenn alle Ausdrücke zu `NULL` ausgewertet werden. Dies ist eine Abweichung vom SQL-Standard. Nach dem Standard ist der Rückgabewert `NULL`, wenn irgendein Argument `NULL` ist. Einige andere Datenbanken verhalten sich so.

## 9.19. Array-Funktionen und -Operatoren

Tabelle 9.56 zeigt die spezialisierten Operatoren, die für Arraytypen verfügbar sind. Zusätzlich stehen für Arrays die in Tabelle 9.1 gezeigten gewöhnlichen Vergleichsoperatoren zur Verfügung. Die Vergleichsoperatoren vergleichen Array-Inhalte Element für Element mit der Standard-B-Tree-Vergleichsfunktion des Elementdatentyps und sortieren anhand des ersten Unterschieds. In mehrdimensionalen Arrays werden die Elemente in zeilenweiser Reihenfolge besucht, wobei der letzte Index am schnellsten variiert. Wenn die Inhalte zweier Arrays gleich sind, sich aber die Dimensionalität unterscheidet, bestimmt der erste Unterschied in der Dimensionsinformation die Sortierreihenfolge.

**Tabelle 9.56. Array-Operatoren**

| Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `anyarray @> anyarray -> boolean` | Enthält das erste Array das zweite, das heißt: Ist jedes Element, das im zweiten Array vorkommt, gleich irgendeinem Element des ersten Arrays? Duplikate werden nicht besonders behandelt; daher gelten `ARRAY[1]` und `ARRAY[1,1]` jeweils als einander enthaltend. | `ARRAY[1,4,3] @> ARRAY[3,1,3] -> t` |
| `anyarray <@ anyarray -> boolean` | Ist das erste Array im zweiten enthalten? | `ARRAY[2,2,7] <@ ARRAY[1,7,4,2,6] -> t` |
| `anyarray && anyarray -> boolean` | Überlappen die Arrays, haben sie also irgendwelche Elemente gemeinsam? | `ARRAY[1,4,3] && ARRAY[2,1] -> t` |
| `anycompatiblearray || anycompatiblearray -> anycompatiblearray` | Verkettet zwei Arrays. Die Verkettung mit einem Null- oder leeren Array ist wirkungslos; andernfalls müssen die Arrays dieselbe Anzahl von Dimensionen haben, wie im ersten Beispiel, oder sich in der Anzahl der Dimensionen um eins unterscheiden, wie im zweiten Beispiel. Wenn die Arrays nicht identische Elementtypen haben, werden sie in einen gemeinsamen Typ gezwungen; siehe [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte). | `ARRAY[1,2,3] || ARRAY[4,5,6,7] -> {1,2,3,4,5,6,7}`<br>`ARRAY[1,2,3] || ARRAY[[4,5,6],[7,8,9.9]] -> {{1,2,3},{4,5,6},{7,8,9.9}}` |
| `anycompatible || anycompatiblearray -> anycompatiblearray` | Verkettet ein Element mit dem Anfang eines Arrays, das leer oder eindimensional sein muss. | `3 || ARRAY[4,5,6] -> {3,4,5,6}` |
| `anycompatiblearray || anycompatible -> anycompatiblearray` | Verkettet ein Element mit dem Ende eines Arrays, das leer oder eindimensional sein muss. | `ARRAY[4,5,6] || 7 -> {4,5,6,7}` |

Weitere Details zum Verhalten von Array-Operatoren stehen in [Abschnitt 8.15](08_Datentypen.md#815-arrays). Weitere Details dazu, welche Operatoren indizierte Operationen unterstützen, stehen in [Abschnitt 11.2](11_Indizes.md#112-indextypen).

Tabelle 9.57 zeigt die Funktionen, die für Arraytypen verfügbar sind. Weitere Informationen und Beispiele zur Verwendung dieser Funktionen finden Sie in [Abschnitt 8.15](08_Datentypen.md#815-arrays).

**Tabelle 9.57. Array-Funktionen**

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `array_append ( anycompatiblearray, anycompatible ) -> anycompatiblearray` | Hängt ein Element an das Ende eines Arrays an; entspricht dem Operator `anycompatiblearray || anycompatible`. | `array_append(ARRAY[1,2], 3) -> {1,2,3}` |
| `array_cat ( anycompatiblearray, anycompatiblearray ) -> anycompatiblearray` | Verkettet zwei Arrays; entspricht dem Operator `anycompatiblearray || anycompatiblearray`. | `array_cat(ARRAY[1,2,3], ARRAY[4,5]) -> {1,2,3,4,5}` |
| `array_dims ( anyarray ) -> text` | Gibt eine Textdarstellung der Array-Dimensionen zurück. | `array_dims(ARRAY[[1,2,3], [4,5,6]]) -> [1:2][1:3]` |
| `array_fill ( anyelement, integer[] [, integer[] ] ) -> anyarray` | Gibt ein Array zurück, das mit Kopien des angegebenen Wertes gefüllt ist und Dimensionen mit den durch das zweite Argument angegebenen Längen hat. Das optionale dritte Argument liefert die unteren Grenzwerte für jede Dimension; standardmäßig sind sie alle 1. | `array_fill(11, ARRAY[2,3]) -> {{11,11,11},{11,11,11}}`<br>`array_fill(7, ARRAY[3], ARRAY[2]) -> [2:4]={7,7,7}` |
| `array_length ( anyarray, integer ) -> integer` | Gibt die Länge der angeforderten Array-Dimension zurück. Für leere oder fehlende Array-Dimensionen wird `NULL` statt 0 erzeugt. | `array_length(array[1,2,3], 1) -> 3`<br>`array_length(array[]::int[], 1) -> NULL`<br>`array_length(array['text'], 2) -> NULL` |
| `array_lower ( anyarray, integer ) -> integer` | Gibt die untere Grenze der angeforderten Array-Dimension zurück. | `array_lower('[0:2]={1,2,3}'::integer[], 1) -> 0` |
| `array_ndims ( anyarray ) -> integer` | Gibt die Anzahl der Dimensionen des Arrays zurück. | `array_ndims(ARRAY[[1,2,3], [4,5,6]]) -> 2` |
| `array_position ( anycompatiblearray, anycompatible [, integer ] ) -> integer` | Gibt den Index des ersten Vorkommens des zweiten Arguments im Array zurück oder `NULL`, wenn es nicht vorhanden ist. Wenn das dritte Argument angegeben ist, beginnt die Suche bei diesem Index. Das Array muss eindimensional sein. Vergleiche erfolgen mit `IS NOT DISTINCT FROM`-Semantik, sodass nach `NULL` gesucht werden kann. | `array_position(ARRAY['sun', 'mon', 'tue', 'wed', 'thu', 'fri', 'sat'], 'mon') -> 2` |
| `array_positions ( anycompatiblearray, anycompatible ) -> integer[]` | Gibt ein Array mit den Indizes aller Vorkommen des zweiten Arguments im als erstem Argument angegebenen Array zurück. Das Array muss eindimensional sein. Vergleiche erfolgen mit `IS NOT DISTINCT FROM`-Semantik, sodass nach `NULL` gesucht werden kann. `NULL` wird nur zurückgegeben, wenn das Array selbst `NULL` ist; wenn der Wert nicht im Array gefunden wird, wird ein leeres Array zurückgegeben. | `array_positions(ARRAY['A','A','B','A'], 'A') -> {1,2,4}` |
| `array_prepend ( anycompatible, anycompatiblearray ) -> anycompatiblearray` | Stellt ein Element an den Anfang eines Arrays; entspricht dem Operator `anycompatible || anycompatiblearray`. | `array_prepend(1, ARRAY[2,3]) -> {1,2,3}` |
| `array_remove ( anycompatiblearray, anycompatible ) -> anycompatiblearray` | Entfernt alle Elemente aus dem Array, die dem angegebenen Wert gleich sind. Das Array muss eindimensional sein. Vergleiche erfolgen mit `IS NOT DISTINCT FROM`-Semantik, sodass `NULL`-Werte entfernt werden können. | `array_remove(ARRAY[1,2,3,2], 2) -> {1,3}` |
| `array_replace ( anycompatiblearray, anycompatible, anycompatible ) -> anycompatiblearray` | Ersetzt jedes Array-Element, das dem zweiten Argument gleich ist, durch das dritte Argument. | `array_replace(ARRAY[1,2,5,4], 5, 3) -> {1,2,3,4}` |
| `array_reverse ( anyarray ) -> anyarray` | Kehrt die erste Dimension des Arrays um. | `array_reverse(ARRAY[[1,2],[3,4],[5,6]]) -> {{5,6},{3,4},{1,2}}` |
| `array_sample ( array anyarray, n integer ) -> anyarray` | Gibt ein Array aus `n` zufällig ausgewählten Elementen aus `array` zurück. `n` darf die Länge der ersten Dimension von `array` nicht überschreiten. Wenn `array` mehrdimensional ist, ist ein „Element“ ein Slice mit einem gegebenen ersten Index. | `array_sample(ARRAY[1,2,3,4,5,6], 3) -> {2,6,1}`<br>`array_sample(ARRAY[[1,2],[3,4],[5,6]], 2) -> {{5,6},{1,2}}` |
| `array_shuffle ( anyarray ) -> anyarray` | Mischt die erste Dimension des Arrays zufällig. | `array_shuffle(ARRAY[[1,2],[3,4],[5,6]]) -> {{5,6},{1,2},{3,4}}` |
| `array_sort ( array anyarray [, descending boolean [, nulls_first boolean ]] ) -> anyarray` | Sortiert die erste Dimension des Arrays. Die Sortierreihenfolge wird durch die Standardsortierung des Elementtyps bestimmt. Wenn der Elementtyp kollatierbar ist, kann die zu verwendende Kollation jedoch mit einer `COLLATE`-Klausel am Arrayargument angegeben werden. Ist `descending` true, wird absteigend sortiert, sonst aufsteigend; ohne Angabe ist aufsteigend der Standard. Ist `nulls_first` true, erscheinen Nullwerte vor Nicht-Nullwerten, andernfalls danach. Ohne Angabe erhält `nulls_first` denselben Wert wie `descending`. | `array_sort(ARRAY[[2,4],[2,1],[6,5]]) -> {{2,1},{2,4},{6,5}}` |
| `array_to_string ( array anyarray, delimiter text [, null_string text ] ) -> text` | Wandelt jedes Array-Element in seine Textdarstellung um und verkettet die Werte, getrennt durch die Trennzeichenkette. Wenn `null_string` angegeben und nicht `NULL` ist, werden `NULL`-Arrayeinträge durch diese Zeichenkette dargestellt; andernfalls werden sie ausgelassen. Siehe auch `string_to_array`. | `array_to_string(ARRAY[1, 2, 3, NULL, 5], ',', '*') -> 1,2,3,*,5` |
| `array_upper ( anyarray, integer ) -> integer` | Gibt die obere Grenze der angeforderten Array-Dimension zurück. | `array_upper(ARRAY[1,8,3,7], 1) -> 4` |
| `cardinality ( anyarray ) -> integer` | Gibt die Gesamtzahl der Elemente im Array zurück oder 0, wenn das Array leer ist. | `cardinality(ARRAY[[1,2],[3,4]]) -> 4` |
| `trim_array ( array anyarray, n integer ) -> anyarray` | Kürzt ein Array, indem die letzten `n` Elemente entfernt werden. Wenn das Array mehrdimensional ist, wird nur die erste Dimension gekürzt. | `trim_array(ARRAY[1,2,3,4,5,6], 2) -> {1,2,3,4}` |
| `unnest ( anyarray ) -> setof anyelement` | Erweitert ein Array zu einer Menge von Zeilen. Die Array-Elemente werden in Speicherreihenfolge ausgelesen. | Siehe Beispiele unten. |
| `unnest ( anyarray, anyarray [, ... ] ) -> setof anyelement, anyelement [, ... ]` | Erweitert mehrere Arrays, die unterschiedliche Datentypen haben können, zu einer Menge von Zeilen. Wenn die Arrays nicht alle dieselbe Länge haben, werden die kürzeren mit `NULL` aufgefüllt. Diese Form ist nur in der `FROM`-Klausel einer Abfrage erlaubt; siehe Abschnitt 7.2.1.4. | Siehe Beispiel unten. |

Beispiele für `unnest`:

```sql
SELECT * FROM unnest(ARRAY[1,2]);
```

```text
 unnest
--------
      1
      2
```

```sql
SELECT * FROM unnest(ARRAY[['foo','bar'],['baz','quux']]);
```

```text
 unnest
--------
 foo
 bar
 baz
 quux
```

```sql
SELECT *
FROM unnest(ARRAY[1,2], ARRAY['foo','bar','baz']) AS x(a,b);
```

```text
 a |  b
---+-----
 1 | foo
 2 | bar
   | baz
```

Siehe auch [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen) zur Aggregatfunktion `array_agg` für die Verwendung mit Arrays.

## 9.20. Bereichs-/Multibereichsfunktionen und -operatoren

Einen Überblick über Bereichstypen finden Sie in [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen).

Tabelle 9.58 zeigt die spezialisierten Operatoren für Bereichstypen. Tabelle 9.59 zeigt die spezialisierten Operatoren für Multibereichstypen. Zusätzlich stehen für Bereichs- und Multibereichstypen die in Tabelle 9.1 gezeigten gewöhnlichen Vergleichsoperatoren zur Verfügung. Die Vergleichsoperatoren sortieren zuerst nach den unteren Bereichsgrenzen und vergleichen nur dann die oberen Grenzen, wenn die unteren gleich sind. Multibereichsoperatoren vergleichen die einzelnen Bereiche, bis einer ungleich ist. Daraus ergibt sich normalerweise keine sinnvolle Gesamtsortierung; die Operatoren werden aber bereitgestellt, damit eindeutige Indizes auf Bereichen konstruiert werden können.

**Tabelle 9.58. Bereichsoperatoren**

| Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `anyrange @> anyrange -> boolean` | Enthält der erste Bereich den zweiten? | `int4range(2,4) @> int4range(2,3) -> t` |
| `anyrange @> anyelement -> boolean` | Enthält der Bereich das Element? | `'[2011-01-01,2011-03-01)'::tsrange @> '2011-01-10'::timestamp -> t` |
| `anyrange <@ anyrange -> boolean` | Ist der erste Bereich im zweiten enthalten? | `int4range(2,4) <@ int4range(1,7) -> t` |
| `anyelement <@ anyrange -> boolean` | Ist das Element im Bereich enthalten? | `42 <@ int4range(1,7) -> f` |
| `anyrange && anyrange -> boolean` | Überlappen die Bereiche, haben sie also irgendwelche Elemente gemeinsam? | `int8range(3,7) && int8range(4,12) -> t` |
| `anyrange << anyrange -> boolean` | Liegt der erste Bereich strikt links vom zweiten? | `int8range(1,10) << int8range(100,110) -> t` |
| `anyrange >> anyrange -> boolean` | Liegt der erste Bereich strikt rechts vom zweiten? | `int8range(50,60) >> int8range(20,30) -> t` |
| `anyrange &< anyrange -> boolean` | Reicht der erste Bereich nicht rechts über den zweiten hinaus? | `int8range(1,20) &< int8range(18,20) -> t` |
| `anyrange &> anyrange -> boolean` | Reicht der erste Bereich nicht links über den zweiten hinaus? | `int8range(7,20) &> int8range(5,10) -> t` |
| `anyrange -|- anyrange -> boolean` | Sind die Bereiche benachbart? | `numrange(1.1,2.2) -|- numrange(2.2,3.3) -> t` |
| `anyrange + anyrange -> anyrange` | Berechnet die Vereinigung der Bereiche. Die Bereiche müssen überlappen oder benachbart sein, sodass die Vereinigung ein einzelner Bereich ist; siehe aber `range_merge()`. | `numrange(5,15) + numrange(10,20) -> [5,20)` |
| `anyrange * anyrange -> anyrange` | Berechnet den Schnitt der Bereiche. | `int8range(5,15) * int8range(10,20) -> [10,15)` |
| `anyrange - anyrange -> anyrange` | Berechnet die Differenz der Bereiche. Der zweite Bereich darf nicht so im ersten enthalten sein, dass die Differenz kein einzelner Bereich mehr wäre. | `int8range(5,15) - int8range(10,20) -> [5,10)` |

**Tabelle 9.59. Multibereichsoperatoren**

| Operator | Beschreibung | Beispiele |
| --- | --- | --- |
| `anymultirange @> anymultirange -> boolean` | Enthält der erste Multibereich den zweiten? | `'{[2,4)}'::int4multirange @> '{[2,3)}'::int4multirange -> t` |
| `anymultirange @> anyrange -> boolean` | Enthält der Multibereich den Bereich? | `'{[2,4)}'::int4multirange @> int4range(2,3) -> t` |
| `anymultirange @> anyelement -> boolean` | Enthält der Multibereich das Element? | `'{[2011-01-01,2011-03-01)}'::tsmultirange @> '2011-01-10'::timestamp -> t` |
| `anyrange @> anymultirange -> boolean` | Enthält der Bereich den Multibereich? | `'[2,4)'::int4range @> '{[2,3)}'::int4multirange -> t` |
| `anymultirange <@ anymultirange -> boolean` | Ist der erste Multibereich im zweiten enthalten? | `'{[2,4)}'::int4multirange <@ '{[1,7)}'::int4multirange -> t` |
| `anymultirange <@ anyrange -> boolean` | Ist der Multibereich im Bereich enthalten? | `'{[2,4)}'::int4multirange <@ int4range(1,7) -> t` |
| `anyrange <@ anymultirange -> boolean` | Ist der Bereich im Multibereich enthalten? | `int4range(2,4) <@ '{[1,7)}'::int4multirange -> t` |
| `anyelement <@ anymultirange -> boolean` | Ist das Element im Multibereich enthalten? | `4 <@ '{[1,7)}'::int4multirange -> t` |
| `anymultirange && anymultirange -> boolean` | Überlappen die Multibereiche, haben sie also irgendwelche Elemente gemeinsam? | `'{[3,7)}'::int8multirange && '{[4,12)}'::int8multirange -> t` |
| `anymultirange && anyrange -> boolean` | Überlappt der Multibereich den Bereich? | `'{[3,7)}'::int8multirange && int8range(4,12) -> t` |
| `anyrange && anymultirange -> boolean` | Überlappt der Bereich den Multibereich? | `int8range(3,7) && '{[4,12)}'::int8multirange -> t` |
| `anymultirange << anymultirange -> boolean` | Liegt der erste Multibereich strikt links vom zweiten? | `'{[1,10)}'::int8multirange << '{[100,110)}'::int8multirange -> t` |
| `anymultirange << anyrange -> boolean` | Liegt der Multibereich strikt links vom Bereich? | `'{[1,10)}'::int8multirange << int8range(100,110) -> t` |
| `anyrange << anymultirange -> boolean` | Liegt der Bereich strikt links vom Multibereich? | `int8range(1,10) << '{[100,110)}'::int8multirange -> t` |
| `anymultirange >> anymultirange -> boolean` | Liegt der erste Multibereich strikt rechts vom zweiten? | `'{[50,60)}'::int8multirange >> '{[20,30)}'::int8multirange -> t` |
| `anymultirange >> anyrange -> boolean` | Liegt der Multibereich strikt rechts vom Bereich? | `'{[50,60)}'::int8multirange >> int8range(20,30) -> t` |
| `anyrange >> anymultirange -> boolean` | Liegt der Bereich strikt rechts vom Multibereich? | `int8range(50,60) >> '{[20,30)}'::int8multirange -> t` |
| `anymultirange &< anymultirange -> boolean` | Reicht der erste Multibereich nicht rechts über den zweiten hinaus? | `'{[1,20)}'::int8multirange &< '{[18,20)}'::int8multirange -> t` |
| `anymultirange &< anyrange -> boolean` | Reicht der Multibereich nicht rechts über den Bereich hinaus? | `'{[1,20)}'::int8multirange &< int8range(18,20) -> t` |
| `anyrange &< anymultirange -> boolean` | Reicht der Bereich nicht rechts über den Multibereich hinaus? | `int8range(1,20) &< '{[18,20)}'::int8multirange -> t` |
| `anymultirange &> anymultirange -> boolean` | Reicht der erste Multibereich nicht links über den zweiten hinaus? | `'{[7,20)}'::int8multirange &> '{[5,10)}'::int8multirange -> t` |
| `anymultirange &> anyrange -> boolean` | Reicht der Multibereich nicht links über den Bereich hinaus? | `'{[7,20)}'::int8multirange &> int8range(5,10) -> t` |
| `anyrange &> anymultirange -> boolean` | Reicht der Bereich nicht links über den Multibereich hinaus? | `int8range(7,20) &> '{[5,10)}'::int8multirange -> t` |
| `anymultirange -|- anymultirange -> boolean` | Sind die Multibereiche benachbart? | `'{[1.1,2.2)}'::nummultirange -|- '{[2.2,3.3)}'::nummultirange -> t` |
| `anymultirange -|- anyrange -> boolean` | Ist der Multibereich zum Bereich benachbart? | `'{[1.1,2.2)}'::nummultirange -|- numrange(2.2,3.3) -> t` |
| `anyrange -|- anymultirange -> boolean` | Ist der Bereich zum Multibereich benachbart? | `numrange(1.1,2.2) -|- '{[2.2,3.3)}'::nummultirange -> t` |
| `anymultirange + anymultirange -> anymultirange` | Berechnet die Vereinigung der Multibereiche. Die Multibereiche müssen nicht überlappen oder benachbart sein. | `'{[5,10)}'::nummultirange + '{[15,20)}'::nummultirange -> {[5,10), [15,20)}` |
| `anymultirange * anymultirange -> anymultirange` | Berechnet den Schnitt der Multibereiche. | `'{[5,15)}'::int8multirange * '{[10,20)}'::int8multirange -> {[10,15)}` |
| `anymultirange - anymultirange -> anymultirange` | Berechnet die Differenz der Multibereiche. | `'{[5,20)}'::int8multirange - '{[10,15)}'::int8multirange -> {[5,10), [15,20)}` |

Die Operatoren für links-von, rechts-von und benachbart geben immer false zurück, wenn ein leerer Bereich oder Multibereich beteiligt ist; ein leerer Bereich gilt also weder als vor noch als nach irgendeinem anderen Bereich liegend.

An anderer Stelle werden leere Bereiche und Multibereiche als additive Identität behandelt: Alles, was mit einem leeren Wert vereinigt wird, bleibt unverändert. Alles, wovon ein leerer Wert abgezogen wird, bleibt unverändert. Ein leerer Multibereich hat genau dieselben Punkte wie ein leerer Bereich. Jeder Bereich enthält den leeren Bereich. Jeder Multibereich enthält beliebig viele leere Bereiche.

Die Operatoren für Bereichsvereinigung und -differenz schlagen fehl, wenn der entstehende Bereich zwei getrennte Teilbereiche enthalten müsste, weil ein solcher Bereich nicht darstellbar ist. Es gibt separate Operatoren für Vereinigung und Differenz, die Multibereichsparameter entgegennehmen und einen Multibereich zurückgeben; diese schlagen auch bei getrennten Argumenten nicht fehl. Wenn Sie also eine Vereinigung oder Differenz für möglicherweise getrennte Bereiche benötigen, können Sie Fehler vermeiden, indem Sie Ihre Bereiche zuerst in Multibereiche umwandeln.

Tabelle 9.60 zeigt die Funktionen, die für Bereichstypen verfügbar sind. Tabelle 9.61 zeigt die Funktionen für Multibereichstypen.

**Tabelle 9.60. Bereichsfunktionen**

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `lower ( anyrange ) -> anyelement` | Extrahiert die untere Grenze des Bereichs; `NULL`, wenn der Bereich leer ist oder keine untere Grenze hat. | `lower(numrange(1.1,2.2)) -> 1.1` |
| `upper ( anyrange ) -> anyelement` | Extrahiert die obere Grenze des Bereichs; `NULL`, wenn der Bereich leer ist oder keine obere Grenze hat. | `upper(numrange(1.1,2.2)) -> 2.2` |
| `isempty ( anyrange ) -> boolean` | Ist der Bereich leer? | `isempty(numrange(1.1,2.2)) -> f` |
| `lower_inc ( anyrange ) -> boolean` | Ist die untere Grenze des Bereichs inklusive? | `lower_inc(numrange(1.1,2.2)) -> t` |
| `upper_inc ( anyrange ) -> boolean` | Ist die obere Grenze des Bereichs inklusive? | `upper_inc(numrange(1.1,2.2)) -> f` |
| `lower_inf ( anyrange ) -> boolean` | Hat der Bereich keine untere Grenze? Eine untere Grenze von `-Infinity` ergibt false. | `lower_inf('(,)'::daterange) -> t` |
| `upper_inf ( anyrange ) -> boolean` | Hat der Bereich keine obere Grenze? Eine obere Grenze von `Infinity` ergibt false. | `upper_inf('(,)'::daterange) -> t` |
| `range_merge ( anyrange, anyrange ) -> anyrange` | Berechnet den kleinsten Bereich, der beide angegebenen Bereiche einschließt. | `range_merge('[1,2)'::int4range, '[3,4)'::int4range) -> [1,4)` |

**Tabelle 9.61. Multibereichsfunktionen**

| Funktion | Beschreibung | Beispiele |
| --- | --- | --- |
| `lower ( anymultirange ) -> anyelement` | Extrahiert die untere Grenze des Multibereichs; `NULL`, wenn der Multibereich leer ist oder keine untere Grenze hat. | `lower('{[1.1,2.2)}'::nummultirange) -> 1.1` |
| `upper ( anymultirange ) -> anyelement` | Extrahiert die obere Grenze des Multibereichs; `NULL`, wenn der Multibereich leer ist oder keine obere Grenze hat. | `upper('{[1.1,2.2)}'::nummultirange) -> 2.2` |
| `isempty ( anymultirange ) -> boolean` | Ist der Multibereich leer? | `isempty('{[1.1,2.2)}'::nummultirange) -> f` |
| `lower_inc ( anymultirange ) -> boolean` | Ist die untere Grenze des Multibereichs inklusive? | `lower_inc('{[1.1,2.2)}'::nummultirange) -> t` |
| `upper_inc ( anymultirange ) -> boolean` | Ist die obere Grenze des Multibereichs inklusive? | `upper_inc('{[1.1,2.2)}'::nummultirange) -> f` |
| `lower_inf ( anymultirange ) -> boolean` | Hat der Multibereich keine untere Grenze? Eine untere Grenze von `-Infinity` ergibt false. | `lower_inf('{(,)}'::datemultirange) -> t` |
| `upper_inf ( anymultirange ) -> boolean` | Hat der Multibereich keine obere Grenze? Eine obere Grenze von `Infinity` ergibt false. | `upper_inf('{(,)}'::datemultirange) -> t` |
| `range_merge ( anymultirange ) -> anyrange` | Berechnet den kleinsten Bereich, der den gesamten Multibereich einschließt. | `range_merge('{[1,2), [3,4)}'::int4multirange) -> [1,4)` |
| `multirange ( anyrange ) -> anymultirange` | Gibt einen Multibereich zurück, der nur den angegebenen Bereich enthält. | `multirange('[1,2)'::int4range) -> {[1,2)}` |
| `unnest ( anymultirange ) -> setof anyrange` | Erweitert einen Multibereich zu einer Menge von Bereichen in aufsteigender Reihenfolge. | Siehe Beispiel unten. |

Beispiel für `unnest` mit einem Multibereich:

```sql
SELECT * FROM unnest('{[1,2), [3,4)}'::int4multirange);
```

```text
 unnest
--------
 [1,2)
 [3,4)
```

Die Funktionen `lower_inc`, `upper_inc`, `lower_inf` und `upper_inf` geben für einen leeren Bereich oder Multibereich alle false zurück.

## 9.21. Aggregatfunktionen

Aggregatfunktionen berechnen aus einer Menge von Eingabewerten ein einzelnes Ergebnis. Die eingebauten allgemeinen Aggregatfunktionen sind in Tabelle 9.62 aufgeführt, statistische Aggregate in Tabelle 9.63. Die eingebauten geordneten Aggregatfunktionen mit `WITHIN GROUP` stehen in Tabelle 9.64, die eingebauten hypothetischen Aggregatfunktionen mit `WITHIN GROUP` in Tabelle 9.65. Gruppierungsoperationen, die eng mit Aggregatfunktionen verwandt sind, sind in Tabelle 9.66 aufgeführt. Die besondere Syntax für Aggregatfunktionen wird in Abschnitt 4.2.7 erläutert. Weitere einführende Informationen finden Sie in [Abschnitt 2.7](02_Die_SQL_Sprache.md#27-aggregatfunktionen).

Aggregatfunktionen, die den partiellen Modus unterstützen, können an verschiedenen Optimierungen teilnehmen, etwa an paralleler Aggregation.

Obwohl alle unten aufgeführten Aggregate eine optionale `ORDER BY`-Klausel akzeptieren, wie in Abschnitt 4.2.7 beschrieben, wurde die Klausel nur bei den Aggregaten aufgeführt, deren Ausgabe von der Reihenfolge abhängt.

**Tabelle 9.62. Allgemeine Aggregatfunktionen**

| Funktion | Beschreibung | Partieller Modus |
|---|---|---|
| `any_value ( anyelement )` -> gleicher Typ wie Eingabe | Gibt einen beliebigen Wert aus den Nicht-Null-Eingabewerten zurück. | Ja |
| `array_agg ( anynonarray ORDER BY input_sort_columns )` -> `anyarray` | Sammelt alle Eingabewerte einschließlich Nullwerten in einem Array. | Ja |
| `array_agg ( anyarray ORDER BY input_sort_columns )` -> `anyarray` | Verkettet alle Eingabearrays zu einem Array mit einer Dimension mehr. Die Eingaben müssen alle dieselbe Dimensionalität haben und dürfen weder leer noch null sein. | Ja |
| `avg ( smallint )` -> `numeric`<br>`avg ( integer )` -> `numeric`<br>`avg ( bigint )` -> `numeric`<br>`avg ( numeric )` -> `numeric`<br>`avg ( real )` -> `double precision`<br>`avg ( double precision )` -> `double precision`<br>`avg ( interval )` -> `interval` | Berechnet den Durchschnitt, also das arithmetische Mittel, aller Nicht-Null-Eingabewerte. | Ja |
| `bit_and ( smallint )` -> `smallint`<br>`bit_and ( integer )` -> `integer`<br>`bit_and ( bigint )` -> `bigint`<br>`bit_and ( bit )` -> `bit` | Berechnet das bitweise UND aller Nicht-Null-Eingabewerte. | Ja |
| `bit_or ( smallint )` -> `smallint`<br>`bit_or ( integer )` -> `integer`<br>`bit_or ( bigint )` -> `bigint`<br>`bit_or ( bit )` -> `bit` | Berechnet das bitweise ODER aller Nicht-Null-Eingabewerte. | Ja |
| `bit_xor ( smallint )` -> `smallint`<br>`bit_xor ( integer )` -> `integer`<br>`bit_xor ( bigint )` -> `bigint`<br>`bit_xor ( bit )` -> `bit` | Berechnet das bitweise exklusive ODER aller Nicht-Null-Eingabewerte. Das kann als Prüfsumme für eine ungeordnete Menge von Werten nützlich sein. | Ja |
| `bool_and ( boolean )` -> `boolean` | Gibt `true` zurück, wenn alle Nicht-Null-Eingabewerte `true` sind, andernfalls `false`. | Ja |
| `bool_or ( boolean )` -> `boolean` | Gibt `true` zurück, wenn irgendein Nicht-Null-Eingabewert `true` ist, andernfalls `false`. | Ja |
| `count ( * )` -> `bigint` | Berechnet die Anzahl der Eingabezeilen. | Ja |
| `count ( "any" )` -> `bigint` | Berechnet die Anzahl der Eingabezeilen, in denen der Eingabewert nicht null ist. | Ja |
| `every ( boolean )` -> `boolean` | Dies ist das SQL-Standardäquivalent zu `bool_and`. | Ja |
| `json_agg ( anyelement ORDER BY input_sort_columns )` -> `json`<br>`jsonb_agg ( anyelement ORDER BY input_sort_columns )` -> `jsonb` | Sammelt alle Eingabewerte einschließlich Nullwerten in einem JSON-Array. Werte werden wie bei `to_json` beziehungsweise `to_jsonb` in JSON umgewandelt. | Nein |
| `json_agg_strict ( anyelement )` -> `json`<br>`jsonb_agg_strict ( anyelement )` -> `jsonb` | Sammelt alle Eingabewerte unter Überspringen von Nullwerten in einem JSON-Array. Werte werden wie bei `to_json` beziehungsweise `to_jsonb` in JSON umgewandelt. | Nein |
| `json_arrayagg ( [ value_expression ] [ ORDER BY sort_expression ] [ { NULL \| ABSENT } ON NULL ] [ RETURNING data_type [ FORMAT JSON [ ENCODING UTF8 ] ] ] )` | Verhält sich wie `json_array`, jedoch als Aggregatfunktion und nimmt deshalb nur einen Parameter `value_expression` entgegen. Wenn `ABSENT ON NULL` angegeben ist, werden alle `NULL`-Werte ausgelassen. Wenn `ORDER BY` angegeben ist, erscheinen die Elemente in dieser Reihenfolge statt in der Eingabereihenfolge.<br><br>`SELECT json_arrayagg(v) FROM (VALUES(2),(1)) t(v)` -> `[2, 1]` | Nein |
| `json_objectagg ( [ { key_expression { VALUE \| ':' } value_expression } ] [ { NULL \| ABSENT } ON NULL ] [ { WITH \| WITHOUT } UNIQUE [ KEYS ] ] [ RETURNING data_type [ FORMAT JSON [ ENCODING UTF8 ] ] ] )` | Verhält sich wie `json_object`, jedoch als Aggregatfunktion und nimmt deshalb nur je einen Parameter `key_expression` und `value_expression` entgegen.<br><br>`SELECT json_objectagg(k:v) FROM (VALUES ('a'::text,current_date),('b',current_date + 1)) AS t(k,v)` -> `{ "a" : "2022-05-10", "b" : "2022-05-11" }` | Nein |
| `json_object_agg ( key "any", value "any" ORDER BY input_sort_columns )` -> `json`<br>`jsonb_object_agg ( key "any", value "any" ORDER BY input_sort_columns )` -> `jsonb` | Sammelt alle Schlüssel/Wert-Paare in einem JSON-Objekt. Schlüsselargumente werden nach `text` umgewandelt; Wertargumente werden wie bei `to_json` beziehungsweise `to_jsonb` umgewandelt. Werte dürfen null sein, Schlüssel nicht. | Nein |
| `json_object_agg_strict ( key "any", value "any" )` -> `json`<br>`jsonb_object_agg_strict ( key "any", value "any" )` -> `jsonb` | Sammelt alle Schlüssel/Wert-Paare in einem JSON-Objekt. Schlüsselargumente werden nach `text` umgewandelt; Wertargumente werden wie bei `to_json` beziehungsweise `to_jsonb` umgewandelt. Der Schlüssel darf nicht null sein. Wenn der Wert null ist, wird der Eintrag übersprungen. | Nein |
| `json_object_agg_unique ( key "any", value "any" )` -> `json`<br>`jsonb_object_agg_unique ( key "any", value "any" )` -> `jsonb` | Sammelt alle Schlüssel/Wert-Paare in einem JSON-Objekt. Schlüsselargumente werden nach `text` umgewandelt; Wertargumente werden wie bei `to_json` beziehungsweise `to_jsonb` umgewandelt. Werte dürfen null sein, Schlüssel nicht. Bei einem doppelten Schlüssel wird ein Fehler ausgelöst. | Nein |
| `json_object_agg_unique_strict ( key "any", value "any" )` -> `json`<br>`jsonb_object_agg_unique_strict ( key "any", value "any" )` -> `jsonb` | Sammelt alle Schlüssel/Wert-Paare in einem JSON-Objekt. Schlüsselargumente werden nach `text` umgewandelt; Wertargumente werden wie bei `to_json` beziehungsweise `to_jsonb` umgewandelt. Der Schlüssel darf nicht null sein. Wenn der Wert null ist, wird der Eintrag übersprungen. Bei einem doppelten Schlüssel wird ein Fehler ausgelöst. | Nein |
| `max ( siehe Text )` -> gleicher Typ wie Eingabe | Berechnet das Maximum der Nicht-Null-Eingabewerte. Verfügbar für jeden numerischen Typ, Zeichenketten-, Datums-/Zeit- oder Enum-Typ sowie für `bytea`, `inet`, `interval`, `money`, `oid`, `pg_lsn`, `tid`, `xid8` und außerdem für Arrays und zusammengesetzte Typen, die sortierbare Datentypen enthalten. | Ja |
| `min ( siehe Text )` -> gleicher Typ wie Eingabe | Berechnet das Minimum der Nicht-Null-Eingabewerte. Verfügbar für jeden numerischen Typ, Zeichenketten-, Datums-/Zeit- oder Enum-Typ sowie für `bytea`, `inet`, `interval`, `money`, `oid`, `pg_lsn`, `tid`, `xid8` und außerdem für Arrays und zusammengesetzte Typen, die sortierbare Datentypen enthalten. | Ja |
| `range_agg ( value anyrange )` -> `anymultirange`<br>`range_agg ( value anymultirange )` -> `anymultirange` | Berechnet die Vereinigung der Nicht-Null-Eingabewerte. | Nein |
| `range_intersect_agg ( value anyrange )` -> `anyrange`<br>`range_intersect_agg ( value anymultirange )` -> `anymultirange` | Berechnet die Schnittmenge der Nicht-Null-Eingabewerte. | Nein |
| `string_agg ( value text, delimiter text )` -> `text`<br>`string_agg ( value bytea, delimiter bytea ORDER BY input_sort_columns )` -> `bytea` | Verkettet die Nicht-Null-Eingabewerte zu einer Zeichenkette. Jedem Wert nach dem ersten wird das entsprechende Trennzeichen vorangestellt, sofern es nicht null ist. | Ja |
| `sum ( smallint )` -> `bigint`<br>`sum ( integer )` -> `bigint`<br>`sum ( bigint )` -> `numeric`<br>`sum ( numeric )` -> `numeric`<br>`sum ( real )` -> `real`<br>`sum ( double precision )` -> `double precision`<br>`sum ( interval )` -> `interval`<br>`sum ( money )` -> `money` | Berechnet die Summe der Nicht-Null-Eingabewerte. | Ja |
| `xmlagg ( xml ORDER BY input_sort_columns )` -> `xml` | Verkettet die Nicht-Null-XML-Eingabewerte; siehe Abschnitt 9.15.1.8. | Nein |

Es ist zu beachten, dass diese Funktionen mit Ausnahme von `count` einen Nullwert zurückgeben, wenn keine Zeilen ausgewählt werden. Insbesondere liefert `sum` über keine Zeilen null und nicht, wie man vielleicht erwarten würde, 0; `array_agg` liefert null statt eines leeren Arrays, wenn es keine Eingabezeilen gibt. Die Funktion `coalesce` kann verwendet werden, um bei Bedarf null durch 0, ein leeres Array oder einen anderen Ersatzwert zu ersetzen.

Die Aggregatfunktionen `array_agg`, `json_agg`, `jsonb_agg`, `json_agg_strict`, `jsonb_agg_strict`, `json_object_agg`, `jsonb_object_agg`, `json_object_agg_strict`, `jsonb_object_agg_strict`, `json_object_agg_unique`, `jsonb_object_agg_unique`, `json_object_agg_unique_strict`, `jsonb_object_agg_unique_strict`, `string_agg` und `xmlagg` sowie ähnliche benutzerdefinierte Aggregatfunktionen erzeugen je nach Reihenfolge der Eingabewerte sinnvoll unterschiedliche Ergebnisse. Diese Reihenfolge ist standardmäßig nicht festgelegt, kann aber durch eine `ORDER BY`-Klausel innerhalb des Aggregataufrufs gesteuert werden, wie in Abschnitt 4.2.7 gezeigt. Alternativ funktioniert es in der Regel, die Eingabewerte aus einer sortierten Unterabfrage zu liefern. Zum Beispiel:

```sql
SELECT xmlagg(x) FROM (SELECT x FROM test ORDER BY y DESC) AS tab;
```

Beachten Sie, dass dieser Ansatz fehlschlagen kann, wenn die äußere Abfrageebene zusätzliche Verarbeitung enthält, etwa einen Join, weil dadurch die Ausgabe der Unterabfrage neu geordnet werden kann, bevor das Aggregat berechnet wird.

> **Hinweis**
> Die booleschen Aggregate `bool_and` und `bool_or` entsprechen den SQL-Standardaggregaten `every` sowie `any` beziehungsweise `some`. PostgreSQL unterstützt `every`, aber nicht `any` oder `some`, weil die Standardsyntax eine Mehrdeutigkeit enthält:
>
> ```sql
> SELECT b1 = ANY((SELECT b2 FROM t2 ...)) FROM t1 ...;
> ```
>
> Hier kann `ANY` entweder als Einleitung einer Unterabfrage verstanden werden oder als Aggregatfunktion, falls die Unterabfrage eine Zeile mit einem booleschen Wert zurückgibt. Daher kann der Standardname diesen Aggregaten nicht gegeben werden.

> **Hinweis**
> Benutzer, die mit anderen SQL-Datenbankmanagementsystemen arbeiten, könnten von der Leistung des Aggregats `count` enttäuscht sein, wenn es auf die gesamte Tabelle angewendet wird. Eine Abfrage wie:
>
> ```sql
> SELECT count(*) FROM sometable;
> ```
>
> erfordert Aufwand proportional zur Tabellengröße: PostgreSQL muss entweder die gesamte Tabelle oder einen vollständigen Index durchsuchen, der alle Tabellenzeilen enthält.

Tabelle 9.63 zeigt Aggregatfunktionen, die typischerweise in der statistischen Analyse verwendet werden. Sie sind nur getrennt aufgeführt, um die Liste der häufiger genutzten Aggregate übersichtlich zu halten. Funktionen, die `numeric_type` akzeptieren, sind für die Typen `smallint`, `integer`, `bigint`, `numeric`, `real` und `double precision` verfügbar. Wenn die Beschreibung `N` erwähnt, ist damit die Anzahl der Eingabezeilen gemeint, für die alle Eingabeausdrücke nicht null sind. In allen Fällen wird null zurückgegeben, wenn die Berechnung nicht sinnvoll ist, zum Beispiel wenn `N` null ist.

**Tabelle 9.63. Aggregatfunktionen für Statistik**

| Funktion | Beschreibung | Partieller Modus |
|---|---|---|
| `corr ( Y double precision, X double precision )` -> `double precision` | Berechnet den Korrelationskoeffizienten. | Ja |
| `covar_pop ( Y double precision, X double precision )` -> `double precision` | Berechnet die Populationskovarianz. | Ja |
| `covar_samp ( Y double precision, X double precision )` -> `double precision` | Berechnet die Stichprobenkovarianz. | Ja |
| `regr_avgx ( Y double precision, X double precision )` -> `double precision` | Berechnet den Durchschnitt der unabhängigen Variablen, `sum(X)/N`. | Ja |
| `regr_avgy ( Y double precision, X double precision )` -> `double precision` | Berechnet den Durchschnitt der abhängigen Variablen, `sum(Y)/N`. | Ja |
| `regr_count ( Y double precision, X double precision )` -> `bigint` | Berechnet die Anzahl der Zeilen, in denen beide Eingaben nicht null sind. | Ja |
| `regr_intercept ( Y double precision, X double precision )` -> `double precision` | Berechnet den y-Achsenabschnitt der durch die Paare `(X, Y)` bestimmten linearen Ausgleichsgeraden nach der Methode der kleinsten Quadrate. | Ja |
| `regr_r2 ( Y double precision, X double precision )` -> `double precision` | Berechnet das Quadrat des Korrelationskoeffizienten. | Ja |
| `regr_slope ( Y double precision, X double precision )` -> `double precision` | Berechnet die Steigung der durch die Paare `(X, Y)` bestimmten linearen Ausgleichsgeraden nach der Methode der kleinsten Quadrate. | Ja |
| `regr_sxx ( Y double precision, X double precision )` -> `double precision` | Berechnet die Quadratsumme der unabhängigen Variablen, `sum(X^2) - sum(X)^2/N`. | Ja |
| `regr_sxy ( Y double precision, X double precision )` -> `double precision` | Berechnet die Summe der Produkte aus unabhängigen und abhängigen Variablen, `sum(X*Y) - sum(X) * sum(Y)/N`. | Ja |
| `regr_syy ( Y double precision, X double precision )` -> `double precision` | Berechnet die Quadratsumme der abhängigen Variablen, `sum(Y^2) - sum(Y)^2/N`. | Ja |
| `stddev ( numeric_type )` -> `double precision` für `real` oder `double precision`, sonst `numeric` | Dies ist ein historischer Alias für `stddev_samp`. | Ja |
| `stddev_pop ( numeric_type )` -> `double precision` für `real` oder `double precision`, sonst `numeric` | Berechnet die Standardabweichung der Population aus den Eingabewerten. | Ja |
| `stddev_samp ( numeric_type )` -> `double precision` für `real` oder `double precision`, sonst `numeric` | Berechnet die Stichprobenstandardabweichung der Eingabewerte. | Ja |
| `variance ( numeric_type )` -> `double precision` für `real` oder `double precision`, sonst `numeric` | Dies ist ein historischer Alias für `var_samp`. | Ja |
| `var_pop ( numeric_type )` -> `double precision` für `real` oder `double precision`, sonst `numeric` | Berechnet die Populationsvarianz der Eingabewerte, also das Quadrat der Populationsstandardabweichung. | Ja |
| `var_samp ( numeric_type )` -> `double precision` für `real` oder `double precision`, sonst `numeric` | Berechnet die Stichprobenvarianz der Eingabewerte, also das Quadrat der Stichprobenstandardabweichung. | Ja |

Tabelle 9.64 zeigt einige Aggregatfunktionen, die die Syntax für geordnete Aggregate verwenden. Diese Funktionen werden manchmal als inverse Verteilungsfunktionen bezeichnet. Ihre aggregierte Eingabe wird durch `ORDER BY` eingeführt; außerdem können sie ein direktes Argument annehmen, das nicht aggregiert, sondern nur einmal berechnet wird. Alle diese Funktionen ignorieren Nullwerte in ihrer aggregierten Eingabe. Bei Funktionen mit einem Parameter `fraction` muss dieser Wert zwischen 0 und 1 liegen; andernfalls wird ein Fehler ausgelöst. Ein Nullwert für `fraction` erzeugt jedoch einfach ein Nullergebnis.

**Tabelle 9.64. Geordnete Aggregatfunktionen**

| Funktion | Beschreibung | Partieller Modus |
|---|---|---|
| `mode () WITHIN GROUP ( ORDER BY anyelement )` -> `anyelement` | Berechnet den Modalwert, also den häufigsten Wert des aggregierten Arguments. Bei mehreren gleich häufigen Werten wird willkürlich der erste gewählt. Das aggregierte Argument muss von einem sortierbaren Typ sein. | Nein |
| `percentile_cont ( fraction double precision ) WITHIN GROUP ( ORDER BY double precision )` -> `double precision`<br>`percentile_cont ( fraction double precision ) WITHIN GROUP ( ORDER BY interval )` -> `interval` | Berechnet das stetige Perzentil, also einen Wert, der dem angegebenen Anteil innerhalb der geordneten Menge aggregierter Argumentwerte entspricht. Bei Bedarf wird zwischen benachbarten Eingabeelementen interpoliert. | Nein |
| `percentile_cont ( fractions double precision[] ) WITHIN GROUP ( ORDER BY double precision )` -> `double precision[]`<br>`percentile_cont ( fractions double precision[] ) WITHIN GROUP ( ORDER BY interval )` -> `interval[]` | Berechnet mehrere stetige Perzentile. Das Ergebnis ist ein Array mit denselben Dimensionen wie der Parameter `fractions`; jedes Nicht-Null-Element wird durch den, gegebenenfalls interpolierten, Wert ersetzt, der diesem Perzentil entspricht. | Nein |
| `percentile_disc ( fraction double precision ) WITHIN GROUP ( ORDER BY anyelement )` -> `anyelement` | Berechnet das diskrete Perzentil, also den ersten Wert innerhalb der geordneten Menge aggregierter Argumentwerte, dessen Position in der Sortierung dem angegebenen Anteil entspricht oder ihn überschreitet. Das aggregierte Argument muss von einem sortierbaren Typ sein. | Nein |
| `percentile_disc ( fractions double precision[] ) WITHIN GROUP ( ORDER BY anyelement )` -> `anyarray` | Berechnet mehrere diskrete Perzentile. Das Ergebnis ist ein Array mit denselben Dimensionen wie der Parameter `fractions`; jedes Nicht-Null-Element wird durch den Eingabewert ersetzt, der diesem Perzentil entspricht. Das aggregierte Argument muss von einem sortierbaren Typ sein. | Nein |

Jedes der in Tabelle 9.65 aufgeführten hypothetischen Aggregate ist mit einer gleichnamigen Fensterfunktion verbunden, die in [Abschnitt 9.22](09_Funktionen_und_Operatoren.md#922-fensterfunktionen) definiert ist. In jedem Fall ist das Ergebnis des Aggregats der Wert, den die zugehörige Fensterfunktion für die aus `args` konstruierte hypothetische Zeile zurückgegeben hätte, wenn eine solche Zeile zur sortierten Zeilengruppe hinzugefügt worden wäre, die durch `sorted_args` dargestellt wird. Für jede dieser Funktionen muss die Liste der direkten Argumente in `args` Anzahl und Typen der aggregierten Argumente in `sorted_args` entsprechen. Anders als die meisten eingebauten Aggregate sind diese Aggregate nicht strikt, das heißt, sie verwerfen keine Eingabezeilen, die Nullwerte enthalten. Nullwerte werden nach der in der `ORDER BY`-Klausel angegebenen Regel sortiert.

**Tabelle 9.65. Hypothetische Aggregatfunktionen**

| Funktion | Beschreibung | Partieller Modus |
|---|---|---|
| `rank ( args ) WITHIN GROUP ( ORDER BY sorted_args )` -> `bigint` | Berechnet den Rang der hypothetischen Zeile mit Lücken, also die Zeilennummer der ersten Zeile in ihrer Peer-Gruppe. | Nein |
| `dense_rank ( args ) WITHIN GROUP ( ORDER BY sorted_args )` -> `bigint` | Berechnet den Rang der hypothetischen Zeile ohne Lücken; diese Funktion zählt effektiv Peer-Gruppen. | Nein |
| `percent_rank ( args ) WITHIN GROUP ( ORDER BY sorted_args )` -> `double precision` | Berechnet den relativen Rang der hypothetischen Zeile, also `(rank - 1) / (total rows - 1)`. Der Wert liegt damit einschließlich zwischen 0 und 1. | Nein |
| `cume_dist ( args ) WITHIN GROUP ( ORDER BY sorted_args )` -> `double precision` | Berechnet die kumulative Verteilung, also `(Anzahl der vorangehenden Zeilen oder Peers mit der hypothetischen Zeile) / (Gesamtzahl der Zeilen)`. Der Wert liegt damit zwischen `1/N` und 1. | Nein |

**Tabelle 9.66. Gruppierungsoperationen**

| Funktion | Beschreibung |
|---|---|
| `GROUPING ( group_by_expression(s) )` -> `integer` | Gibt eine Bitmaske zurück, die angibt, welche `GROUP BY`-Ausdrücke nicht in der aktuellen Gruppierungsmenge enthalten sind. Die Bits werden so zugeordnet, dass das rechteste Argument dem niederwertigsten Bit entspricht. Jedes Bit ist 0, wenn der entsprechende Ausdruck in den Gruppierungskriterien der Gruppierungsmenge enthalten ist, die die aktuelle Ergebniszeile erzeugt, und 1, wenn er nicht enthalten ist. |

Die in Tabelle 9.66 gezeigten Gruppierungsoperationen werden zusammen mit Gruppierungsmengen verwendet, siehe Abschnitt 7.2.4, um Ergebniszeilen zu unterscheiden. Die Argumente der Funktion `GROUPING` werden nicht tatsächlich ausgewertet, müssen aber exakt mit Ausdrücken übereinstimmen, die in der `GROUP BY`-Klausel der zugehörigen Abfrageebene angegeben sind. Zum Beispiel:

```sql
SELECT * FROM items_sold;
```

```text
 make | model | sales
------+-------+-------
 Foo  | GT    |    10
 Foo  | Tour  |    20
 Bar  | City  |    15
 Bar  | Sport |     5
(4 rows)
```

```sql
SELECT make, model, GROUPING(make,model), sum(sales)
FROM items_sold
GROUP BY ROLLUP(make,model);
```

```text
 make | model | grouping | sum
------+-------+----------+-----
 Foo  | GT    |        0 |  10
 Foo  | Tour  |        0 |  20
 Bar  | City  |        0 |  15
 Bar  | Sport |        0 |   5
 Foo  |       |        1 |  30
 Bar  |       |        1 |  20
      |       |        3 |  50
(7 rows)
```

Hier zeigt der Gruppierungswert 0 in den ersten vier Zeilen, dass diese normal über beide Gruppierungsspalten gruppiert wurden. Der Wert 1 bedeutet, dass `model` in den vorletzten beiden Zeilen nicht gruppiert wurde, und der Wert 3 bedeutet, dass weder `make` noch `model` in der letzten Zeile gruppiert wurden. Diese letzte Zeile ist daher ein Aggregat über alle Eingabezeilen.

## 9.22. Fensterfunktionen

Fensterfunktionen ermöglichen Berechnungen über Mengen von Zeilen, die mit der aktuellen Abfragezeile zusammenhängen. Eine Einführung in diese Funktionalität finden Sie in [Abschnitt 3.5](03_Fortgeschrittene_Funktionen.md#35-windowfunktionen); Syntaxdetails stehen in Abschnitt 4.2.8.

Die eingebauten Fensterfunktionen sind in Tabelle 9.67 aufgeführt. Beachten Sie, dass diese Funktionen mit Fensterfunktionssyntax aufgerufen werden müssen, das heißt, eine `OVER`-Klausel ist erforderlich.

Zusätzlich zu diesen Funktionen kann jedes eingebaute oder benutzerdefinierte normale Aggregat, also kein geordnetes oder hypothetisches Aggregat, als Fensterfunktion verwendet werden. Eine Liste der eingebauten Aggregate finden Sie in [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen). Aggregatfunktionen verhalten sich nur dann als Fensterfunktionen, wenn dem Aufruf eine `OVER`-Klausel folgt; andernfalls verhalten sie sich als einfache Aggregate und geben eine einzige Zeile für die gesamte Menge zurück.

**Tabelle 9.67. Allgemeine Fensterfunktionen**

| Funktion | Beschreibung |
|---|---|
| `row_number ()` -> `bigint` | Gibt die Nummer der aktuellen Zeile innerhalb ihrer Partition zurück, gezählt ab 1. |
| `rank ()` -> `bigint` | Gibt den Rang der aktuellen Zeile mit Lücken zurück, also den `row_number`-Wert der ersten Zeile in ihrer Peer-Gruppe. |
| `dense_rank ()` -> `bigint` | Gibt den Rang der aktuellen Zeile ohne Lücken zurück; diese Funktion zählt effektiv Peer-Gruppen. |
| `percent_rank ()` -> `double precision` | Gibt den relativen Rang der aktuellen Zeile zurück, also `(rank - 1) / (total partition rows - 1)`. Der Wert liegt damit einschließlich zwischen 0 und 1. |
| `cume_dist ()` -> `double precision` | Gibt die kumulative Verteilung zurück, also `(Anzahl der vorangehenden Partitionszeilen oder Peers mit der aktuellen Zeile) / (Gesamtzahl der Partitionszeilen)`. Der Wert liegt damit zwischen `1/N` und 1. |
| `ntile ( num_buckets integer )` -> `integer` | Gibt eine Ganzzahl von 1 bis zum Argumentwert zurück und teilt die Partition so gleichmäßig wie möglich auf. |
| `lag ( value anycompatible [, offset integer [, default anycompatible ]] )` -> `anycompatible` | Gibt `value` ausgewertet an der Zeile zurück, die `offset` Zeilen vor der aktuellen Zeile innerhalb der Partition liegt. Gibt es keine solche Zeile, wird stattdessen `default` zurückgegeben, dessen Typ mit `value` kompatibel sein muss. Sowohl `offset` als auch `default` werden bezogen auf die aktuelle Zeile ausgewertet. Wenn ausgelassen, ist `offset` standardmäßig 1 und `default` standardmäßig `NULL`. |
| `lead ( value anycompatible [, offset integer [, default anycompatible ]] )` -> `anycompatible` | Gibt `value` ausgewertet an der Zeile zurück, die `offset` Zeilen nach der aktuellen Zeile innerhalb der Partition liegt. Gibt es keine solche Zeile, wird stattdessen `default` zurückgegeben, dessen Typ mit `value` kompatibel sein muss. Sowohl `offset` als auch `default` werden bezogen auf die aktuelle Zeile ausgewertet. Wenn ausgelassen, ist `offset` standardmäßig 1 und `default` standardmäßig `NULL`. |
| `first_value ( value anyelement )` -> `anyelement` | Gibt `value` ausgewertet an der ersten Zeile des Fensterrahmens zurück. |
| `last_value ( value anyelement )` -> `anyelement` | Gibt `value` ausgewertet an der letzten Zeile des Fensterrahmens zurück. |
| `nth_value ( value anyelement, n integer )` -> `anyelement` | Gibt `value` ausgewertet an der n-ten Zeile des Fensterrahmens zurück, gezählt ab 1. Gibt `NULL` zurück, wenn es keine solche Zeile gibt. |

Alle in Tabelle 9.67 aufgeführten Funktionen hängen von der Sortierreihenfolge ab, die durch die `ORDER BY`-Klausel der zugehörigen Fensterdefinition angegeben ist. Zeilen, die bei Betrachtung nur der `ORDER BY`-Spalten nicht unterscheidbar sind, werden Peers genannt. Die vier Rangfunktionen einschließlich `cume_dist` sind so definiert, dass sie für alle Zeilen einer Peer-Gruppe dieselbe Antwort liefern.

Beachten Sie, dass `first_value`, `last_value` und `nth_value` nur die Zeilen innerhalb des Fensterrahmens berücksichtigen. Standardmäßig enthält dieser die Zeilen vom Beginn der Partition bis zum letzten Peer der aktuellen Zeile. Das führt bei `last_value` und manchmal auch bei `nth_value` wahrscheinlich zu wenig hilfreichen Ergebnissen. Sie können den Rahmen neu definieren, indem Sie der `OVER`-Klausel eine passende Rahmenspezifikation hinzufügen, also `RANGE`, `ROWS` oder `GROUPS`. Weitere Informationen zu Rahmenspezifikationen finden Sie in Abschnitt 4.2.8.

Wenn eine Aggregatfunktion als Fensterfunktion verwendet wird, aggregiert sie über die Zeilen innerhalb des Fensterrahmens der aktuellen Zeile. Ein Aggregat, das mit `ORDER BY` und der standardmäßigen Fensterrahmendefinition verwendet wird, erzeugt ein Verhalten wie eine laufende Summe, was erwünscht sein kann oder auch nicht. Um über die gesamte Partition zu aggregieren, lassen Sie `ORDER BY` weg oder verwenden Sie `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`. Andere Rahmenspezifikationen können verwendet werden, um andere Effekte zu erzielen.

> **Hinweis**
> Der SQL-Standard definiert für `lead`, `lag`, `first_value`, `last_value` und `nth_value` eine Option `RESPECT NULLS` oder `IGNORE NULLS`. Diese ist in PostgreSQL nicht implementiert: Das Verhalten entspricht immer der Voreinstellung des Standards, nämlich `RESPECT NULLS`. Ebenso ist die Standardoption `FROM FIRST` oder `FROM LAST` für `nth_value` nicht implementiert: Es wird nur das Standardverhalten `FROM FIRST` unterstützt. Das Ergebnis von `FROM LAST` können Sie erreichen, indem Sie die `ORDER BY`-Sortierung umkehren.

## 9.23. MERGE-Unterstützungsfunktionen

PostgreSQL enthält eine MERGE-Unterstützungsfunktion, die in der `RETURNING`-Liste eines `MERGE`-Befehls verwendet werden kann, um die für jede Zeile ausgeführte Aktion zu identifizieren; siehe Tabelle 9.68.

**Tabelle 9.68. MERGE-Unterstützungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `merge_action ( )` -> `text` | Gibt den Merge-Aktionsbefehl zurück, der für die aktuelle Zeile ausgeführt wurde. Dies ist `INSERT`, `UPDATE` oder `DELETE`. |

Beispiel:

```sql
MERGE INTO products p
  USING stock s ON p.product_id = s.product_id
  WHEN MATCHED AND s.quantity > 0 THEN
    UPDATE SET in_stock = true, quantity = s.quantity
  WHEN MATCHED THEN
    UPDATE SET in_stock = false, quantity = 0
  WHEN NOT MATCHED THEN
    INSERT (product_id, in_stock, quantity)
      VALUES (s.product_id, true, s.quantity)
  RETURNING merge_action(), p.*;
```

```text
 merge_action | product_id | in_stock | quantity
--------------+------------+----------+----------
 UPDATE       |       1001 | t        |       50
 UPDATE       |       1002 | f        |        0
 INSERT       |       1003 | t        |       10
```

Beachten Sie, dass diese Funktion nur in der `RETURNING`-Liste eines `MERGE`-Befehls verwendet werden kann. Ihre Verwendung in jedem anderen Teil einer Abfrage ist ein Fehler.

## 9.24. Unterabfrageausdrücke

Dieser Abschnitt beschreibt die SQL-konformen Unterabfrageausdrücke, die in PostgreSQL verfügbar sind. Alle in diesem Abschnitt dokumentierten Ausdrucksformen geben boolesche Ergebnisse zurück, also `true` oder `false`.

### 9.24.1. EXISTS

```sql
EXISTS (subquery)
```

Das Argument von `EXISTS` ist eine beliebige `SELECT`-Anweisung, also eine Unterabfrage. Die Unterabfrage wird ausgewertet, um festzustellen, ob sie Zeilen zurückgibt. Wenn sie mindestens eine Zeile zurückgibt, ist das Ergebnis von `EXISTS` `true`; wenn die Unterabfrage keine Zeilen zurückgibt, ist das Ergebnis `false`.

Die Unterabfrage kann auf Variablen aus der umgebenden Abfrage verweisen; diese wirken während einer einzelnen Auswertung der Unterabfrage wie Konstanten.

Die Unterabfrage wird im Allgemeinen nur so lange ausgeführt, bis feststeht, ob mindestens eine Zeile zurückgegeben wird, nicht notwendigerweise bis zum Ende. Es ist unklug, eine Unterabfrage zu schreiben, die Nebeneffekte hat, etwa Sequenzfunktionen aufruft; ob die Nebeneffekte eintreten, kann unvorhersehbar sein.

Da das Ergebnis nur davon abhängt, ob Zeilen zurückgegeben werden, und nicht vom Inhalt dieser Zeilen, ist die Ausgabeliste der Unterabfrage normalerweise unwichtig. Eine verbreitete Schreibkonvention ist, alle `EXISTS`-Tests in der Form `EXISTS(SELECT 1 WHERE ...)` zu schreiben. Es gibt jedoch Ausnahmen von dieser Regel, etwa Unterabfragen, die `INTERSECT` verwenden.

Dieses einfache Beispiel verhält sich wie ein Inner Join über `col2`, erzeugt aber höchstens eine Ausgabezeile für jede Zeile aus `tab1`, auch wenn mehrere passende Zeilen in `tab2` vorhanden sind:

```sql
SELECT col1
FROM tab1
WHERE EXISTS (SELECT 1 FROM tab2 WHERE col2 = tab1.col2);
```

### 9.24.2. IN

```sql
expression IN (subquery)
```

Die rechte Seite ist eine geklammerte Unterabfrage, die genau eine Spalte zurückgeben muss. Der linke Ausdruck wird ausgewertet und mit jeder Zeile des Unterabfrageergebnisses verglichen. Das Ergebnis von `IN` ist `true`, wenn eine gleiche Unterabfragezeile gefunden wird. Das Ergebnis ist `false`, wenn keine gleiche Zeile gefunden wird, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt.

Beachten Sie, dass das Ergebnis des `IN`-Konstrukts null und nicht `false` ist, wenn der linke Ausdruck null ergibt oder wenn es keine gleichen rechten Werte gibt und mindestens eine rechte Zeile null ergibt. Das entspricht den normalen SQL-Regeln für boolesche Kombinationen mit Nullwerten.

Wie bei `EXISTS` ist es unklug anzunehmen, dass die Unterabfrage vollständig ausgewertet wird.

```sql
row_constructor IN (subquery)
```

Die linke Seite dieser Form von `IN` ist ein Zeilenkonstruktor, wie in Abschnitt 4.2.13 beschrieben. Die rechte Seite ist eine geklammerte Unterabfrage, die genau so viele Spalten zurückgeben muss, wie Ausdrücke in der linken Zeile stehen. Die linken Ausdrücke werden ausgewertet und zeilenweise mit jeder Zeile des Unterabfrageergebnisses verglichen. Das Ergebnis von `IN` ist `true`, wenn eine gleiche Unterabfragezeile gefunden wird. Das Ergebnis ist `false`, wenn keine gleiche Zeile gefunden wird, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt.

Wie üblich werden Nullwerte in den Zeilen nach den normalen Regeln boolescher SQL-Ausdrücke kombiniert. Zwei Zeilen gelten als gleich, wenn alle entsprechenden Elemente nicht null und gleich sind. Die Zeilen sind ungleich, wenn irgendwelche entsprechenden Elemente nicht null und ungleich sind; andernfalls ist das Ergebnis dieses Zeilenvergleichs unbekannt, also null. Wenn alle zeilenweisen Ergebnisse entweder ungleich oder null sind und mindestens ein null dabei ist, ist das Ergebnis von `IN` null.

### 9.24.3. NOT IN

```sql
expression NOT IN (subquery)
```

Die rechte Seite ist eine geklammerte Unterabfrage, die genau eine Spalte zurückgeben muss. Der linke Ausdruck wird ausgewertet und mit jeder Zeile des Unterabfrageergebnisses verglichen. Das Ergebnis von `NOT IN` ist `true`, wenn nur ungleiche Unterabfragezeilen gefunden werden, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt. Das Ergebnis ist `false`, wenn irgendeine gleiche Zeile gefunden wird.

Beachten Sie, dass das Ergebnis des `NOT IN`-Konstrukts null und nicht `true` ist, wenn der linke Ausdruck null ergibt oder wenn es keine gleichen rechten Werte gibt und mindestens eine rechte Zeile null ergibt. Das entspricht den normalen SQL-Regeln für boolesche Kombinationen mit Nullwerten.

Wie bei `EXISTS` ist es unklug anzunehmen, dass die Unterabfrage vollständig ausgewertet wird.

```sql
row_constructor NOT IN (subquery)
```

Die linke Seite dieser Form von `NOT IN` ist ein Zeilenkonstruktor, wie in Abschnitt 4.2.13 beschrieben. Die rechte Seite ist eine geklammerte Unterabfrage, die genau so viele Spalten zurückgeben muss, wie Ausdrücke in der linken Zeile stehen. Die linken Ausdrücke werden ausgewertet und zeilenweise mit jeder Zeile des Unterabfrageergebnisses verglichen. Das Ergebnis von `NOT IN` ist `true`, wenn nur ungleiche Unterabfragezeilen gefunden werden, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt. Das Ergebnis ist `false`, wenn irgendeine gleiche Zeile gefunden wird.

Wie üblich werden Nullwerte in den Zeilen nach den normalen Regeln boolescher SQL-Ausdrücke kombiniert. Zwei Zeilen gelten als gleich, wenn alle entsprechenden Elemente nicht null und gleich sind. Die Zeilen sind ungleich, wenn irgendwelche entsprechenden Elemente nicht null und ungleich sind; andernfalls ist das Ergebnis dieses Zeilenvergleichs unbekannt, also null. Wenn alle zeilenweisen Ergebnisse entweder ungleich oder null sind und mindestens ein null dabei ist, ist das Ergebnis von `NOT IN` null.

### 9.24.4. ANY/SOME

```sql
expression operator ANY (subquery)
expression operator SOME (subquery)
```

Die rechte Seite ist eine geklammerte Unterabfrage, die genau eine Spalte zurückgeben muss. Der linke Ausdruck wird ausgewertet und mit jeder Zeile des Unterabfrageergebnisses unter Verwendung des angegebenen Operators verglichen; dieser muss ein boolesches Ergebnis liefern. Das Ergebnis von `ANY` ist `true`, wenn irgendein Ergebnis `true` ist. Das Ergebnis ist `false`, wenn kein wahres Ergebnis gefunden wird, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt.

`SOME` ist ein Synonym für `ANY`. `IN` entspricht `= ANY`.

Beachten Sie, dass das Ergebnis des `ANY`-Konstrukts null und nicht `false` ist, wenn es keine erfolgreichen Vergleiche gibt und mindestens eine rechte Zeile für das Operatorergebnis null liefert. Das entspricht den normalen SQL-Regeln für boolesche Kombinationen mit Nullwerten.

Wie bei `EXISTS` ist es unklug anzunehmen, dass die Unterabfrage vollständig ausgewertet wird.

```sql
row_constructor operator ANY (subquery)
row_constructor operator SOME (subquery)
```

Die linke Seite dieser Form von `ANY` ist ein Zeilenkonstruktor, wie in Abschnitt 4.2.13 beschrieben. Die rechte Seite ist eine geklammerte Unterabfrage, die genau so viele Spalten zurückgeben muss, wie Ausdrücke in der linken Zeile stehen. Die linken Ausdrücke werden ausgewertet und zeilenweise mit jeder Zeile des Unterabfrageergebnisses unter Verwendung des angegebenen Operators verglichen. Das Ergebnis von `ANY` ist `true`, wenn der Vergleich für irgendeine Unterabfragezeile `true` zurückgibt. Das Ergebnis ist `false`, wenn der Vergleich für jede Unterabfragezeile `false` zurückgibt, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt. Das Ergebnis ist `NULL`, wenn kein Vergleich mit einer Unterabfragezeile `true` zurückgibt und mindestens ein Vergleich `NULL` zurückgibt.

Details zur Bedeutung eines Zeilenkonstruktorvergleichs finden Sie in Abschnitt 9.25.5.

### 9.24.5. ALL

```sql
expression operator ALL (subquery)
```

Die rechte Seite ist eine geklammerte Unterabfrage, die genau eine Spalte zurückgeben muss. Der linke Ausdruck wird ausgewertet und mit jeder Zeile des Unterabfrageergebnisses unter Verwendung des angegebenen Operators verglichen; dieser muss ein boolesches Ergebnis liefern. Das Ergebnis von `ALL` ist `true`, wenn alle Zeilen `true` liefern, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt. Das Ergebnis ist `false`, wenn irgendein falsches Ergebnis gefunden wird. Das Ergebnis ist `NULL`, wenn kein Vergleich mit einer Unterabfragezeile `false` zurückgibt und mindestens ein Vergleich `NULL` zurückgibt.

`NOT IN` entspricht `<> ALL`.

Wie bei `EXISTS` ist es unklug anzunehmen, dass die Unterabfrage vollständig ausgewertet wird.

```sql
row_constructor operator ALL (subquery)
```

Die linke Seite dieser Form von `ALL` ist ein Zeilenkonstruktor, wie in Abschnitt 4.2.13 beschrieben. Die rechte Seite ist eine geklammerte Unterabfrage, die genau so viele Spalten zurückgeben muss, wie Ausdrücke in der linken Zeile stehen. Die linken Ausdrücke werden ausgewertet und zeilenweise mit jeder Zeile des Unterabfrageergebnisses unter Verwendung des angegebenen Operators verglichen. Das Ergebnis von `ALL` ist `true`, wenn der Vergleich für alle Unterabfragezeilen `true` zurückgibt, auch dann, wenn die Unterabfrage keine Zeilen zurückgibt. Das Ergebnis ist `false`, wenn der Vergleich für irgendeine Unterabfragezeile `false` zurückgibt. Das Ergebnis ist `NULL`, wenn kein Vergleich mit einer Unterabfragezeile `false` zurückgibt und mindestens ein Vergleich `NULL` zurückgibt.

Details zur Bedeutung eines Zeilenkonstruktorvergleichs finden Sie in Abschnitt 9.25.5.

### 9.24.6. Vergleich mit Einzelzeile

```sql
row_constructor operator (subquery)
```

Die linke Seite ist ein Zeilenkonstruktor, wie in Abschnitt 4.2.13 beschrieben. Die rechte Seite ist eine geklammerte Unterabfrage, die genau so viele Spalten zurückgeben muss, wie Ausdrücke in der linken Zeile stehen. Außerdem darf die Unterabfrage nicht mehr als eine Zeile zurückgeben. Wenn sie keine Zeilen zurückgibt, wird das Ergebnis als null angenommen. Die linke Seite wird ausgewertet und zeilenweise mit der einzelnen Ergebniszeile der Unterabfrage verglichen.

Details zur Bedeutung eines Zeilenkonstruktorvergleichs finden Sie in Abschnitt 9.25.5.

## 9.25. Zeilen- und Arrayvergleiche

Dieser Abschnitt beschreibt mehrere spezialisierte Konstrukte, mit denen mehrere Vergleiche zwischen Wertgruppen durchgeführt werden können. Diese Formen sind syntaktisch mit den Unterabfrageformen aus dem vorherigen Abschnitt verwandt, verwenden aber keine Unterabfragen. Die Formen mit Array-Teilausdrücken sind PostgreSQL-Erweiterungen; die übrigen sind SQL-konform. Alle in diesem Abschnitt dokumentierten Ausdrucksformen geben boolesche Ergebnisse zurück, also `true` oder `false`.

### 9.25.1. IN

```sql
expression IN (value [, ...])
```

Die rechte Seite ist eine geklammerte Liste von Ausdrücken. Das Ergebnis ist `true`, wenn das Ergebnis des linken Ausdrucks gleich irgendeinem der rechten Ausdrücke ist. Dies ist eine Kurzschreibweise für:

```sql
expression = value1
OR
expression = value2
OR
...
```

Beachten Sie, dass das Ergebnis des `IN`-Konstrukts null und nicht `false` ist, wenn der linke Ausdruck null ergibt oder wenn es keine gleichen rechten Werte gibt und mindestens ein rechter Ausdruck null ergibt. Das entspricht den normalen SQL-Regeln für boolesche Kombinationen mit Nullwerten.

### 9.25.2. NOT IN

```sql
expression NOT IN (value [, ...])
```

Die rechte Seite ist eine geklammerte Liste von Ausdrücken. Das Ergebnis ist `true`, wenn das Ergebnis des linken Ausdrucks ungleich allen rechten Ausdrücken ist. Dies ist eine Kurzschreibweise für:

```sql
expression <> value1
AND
expression <> value2
AND
...
```

Beachten Sie, dass das Ergebnis des `NOT IN`-Konstrukts null und nicht, wie man naiv erwarten könnte, `true` ist, wenn der linke Ausdruck null ergibt oder wenn es keine gleichen rechten Werte gibt und mindestens ein rechter Ausdruck null ergibt. Das entspricht den normalen SQL-Regeln für boolesche Kombinationen mit Nullwerten.

> **Tipp**
> `x NOT IN y` ist in allen Fällen äquivalent zu `NOT (x IN y)`. Nullwerte bringen Einsteiger bei `NOT IN` jedoch deutlich leichter ins Stolpern als bei `IN`. Wenn möglich, ist es besser, die Bedingung positiv zu formulieren.

### 9.25.3. ANY/SOME (Array)

```sql
expression operator ANY (array expression)
expression operator SOME (array expression)
```

Die rechte Seite ist ein geklammerter Ausdruck, der einen Arraywert ergeben muss. Der linke Ausdruck wird ausgewertet und mit jedem Element des Arrays unter Verwendung des angegebenen Operators verglichen; dieser muss ein boolesches Ergebnis liefern. Das Ergebnis von `ANY` ist `true`, wenn irgendein wahres Ergebnis erzielt wird. Das Ergebnis ist `false`, wenn kein wahres Ergebnis gefunden wird, auch dann, wenn das Array keine Elemente enthält.

Wenn der Arrayausdruck ein Null-Array ergibt, ist das Ergebnis von `ANY` null. Wenn der linke Ausdruck null ergibt, ist das Ergebnis von `ANY` normalerweise null, auch wenn ein nicht-strikter Vergleichsoperator möglicherweise ein anderes Ergebnis liefern könnte. Enthält das rechte Array Null-Elemente und wird kein wahrer Vergleich erzielt, ist das Ergebnis von `ANY` ebenfalls null und nicht `false`, wiederum unter Annahme eines strikten Vergleichsoperators. Das entspricht den normalen SQL-Regeln für boolesche Kombinationen mit Nullwerten.

`SOME` ist ein Synonym für `ANY`.

### 9.25.4. ALL (Array)

```sql
expression operator ALL (array expression)
```

Die rechte Seite ist ein geklammerter Ausdruck, der einen Arraywert ergeben muss. Der linke Ausdruck wird ausgewertet und mit jedem Element des Arrays unter Verwendung des angegebenen Operators verglichen; dieser muss ein boolesches Ergebnis liefern. Das Ergebnis von `ALL` ist `true`, wenn alle Vergleiche `true` liefern, auch dann, wenn das Array keine Elemente enthält. Das Ergebnis ist `false`, wenn irgendein falsches Ergebnis gefunden wird.

Wenn der Arrayausdruck ein Null-Array ergibt, ist das Ergebnis von `ALL` null. Wenn der linke Ausdruck null ergibt, ist das Ergebnis von `ALL` normalerweise null, auch wenn ein nicht-strikter Vergleichsoperator möglicherweise ein anderes Ergebnis liefern könnte. Enthält das rechte Array Null-Elemente und wird kein falscher Vergleich erzielt, ist das Ergebnis von `ALL` null und nicht `true`, wiederum unter Annahme eines strikten Vergleichsoperators. Das entspricht den normalen SQL-Regeln für boolesche Kombinationen mit Nullwerten.

### 9.25.5. Zeilenkonstruktorvergleich

```sql
row_constructor operator row_constructor
```

Jede Seite ist ein Zeilenkonstruktor, wie in Abschnitt 4.2.13 beschrieben. Die beiden Zeilenkonstruktoren müssen dieselbe Anzahl von Feldern haben. Der angegebene Operator wird auf jedes Paar entsprechender Felder angewendet. Da die Felder unterschiedliche Typen haben können, kann für jedes Paar ein anderer konkreter Operator ausgewählt werden. Alle ausgewählten Operatoren müssen Mitglieder irgendeiner B-Tree-Operatorklasse sein oder der Negator eines `=`-Mitglieds einer B-Tree-Operatorklasse. Das bedeutet, dass ein Zeilenkonstruktorvergleich nur möglich ist, wenn der Operator `=`, `<>`, `<`, `<=`, `>` oder `>=` ist oder eine ähnliche Semantik hat.

Die Fälle `=` und `<>` verhalten sich etwas anders als die übrigen. Zwei Zeilen gelten als gleich, wenn alle entsprechenden Elemente nicht null und gleich sind. Die Zeilen sind ungleich, wenn irgendwelche entsprechenden Elemente nicht null und ungleich sind; andernfalls ist das Ergebnis des Zeilenvergleichs unbekannt, also null.

Bei `<`, `<=`, `>` und `>=` werden die Zeilenelemente von links nach rechts verglichen; der Vergleich stoppt, sobald ein ungleiches oder nullwertiges Elementpaar gefunden wird. Wenn eines der beiden Elemente dieses Paares null ist, ist das Ergebnis des Zeilenvergleichs unbekannt, also null. Andernfalls bestimmt der Vergleich dieses Elementpaares das Ergebnis. Zum Beispiel ergibt `ROW(1,2,NULL) < ROW(1,3,0)` `true` und nicht null, weil das dritte Elementpaar nicht betrachtet wird.

```sql
row_constructor IS DISTINCT FROM row_constructor
```

Dieses Konstrukt ähnelt einem `<>`-Zeilenvergleich, liefert bei Null-Eingaben aber nicht null. Stattdessen gilt jeder Nullwert als ungleich, also verschieden von jedem Nicht-Null-Wert, und zwei Nullwerte gelten als gleich, also nicht verschieden. Das Ergebnis ist daher immer entweder `true` oder `false`, niemals null.

```sql
row_constructor IS NOT DISTINCT FROM row_constructor
```

Dieses Konstrukt ähnelt einem `=`-Zeilenvergleich, liefert bei Null-Eingaben aber nicht null. Stattdessen gilt jeder Nullwert als ungleich, also verschieden von jedem Nicht-Null-Wert, und zwei Nullwerte gelten als gleich, also nicht verschieden. Das Ergebnis ist daher immer entweder `true` oder `false`, niemals null.

### 9.25.6. Vergleich zusammengesetzter Typen

```sql
record operator record
```

Die SQL-Spezifikation verlangt, dass zeilenweise Vergleiche `NULL` zurückgeben, wenn das Ergebnis vom Vergleich zweier `NULL`-Werte oder eines `NULL`- und eines Nicht-`NULL`-Werts abhängt. PostgreSQL tut dies nur beim Vergleich der Ergebnisse zweier Zeilenkonstruktoren, wie in Abschnitt 9.25.5, oder beim Vergleich eines Zeilenkonstruktors mit der Ausgabe einer Unterabfrage, wie in [Abschnitt 9.24](09_Funktionen_und_Operatoren.md#924-unterabfrageausdrücke). In anderen Kontexten, in denen zwei Werte zusammengesetzter Typen verglichen werden, gelten zwei `NULL`-Feldwerte als gleich, und ein `NULL` gilt als größer als ein Nicht-`NULL`. Das ist notwendig, um konsistentes Sortier- und Indexverhalten für zusammengesetzte Typen zu erhalten.

Jede Seite wird ausgewertet, und beide werden zeilenweise verglichen. Vergleiche zusammengesetzter Typen sind erlaubt, wenn der Operator `=`, `<>`, `<`, `<=`, `>` oder `>=` ist oder eine ähnliche Semantik hat. Genauer gesagt kann ein Operator ein Zeilenvergleichsoperator sein, wenn er Mitglied einer B-Tree-Operatorklasse ist oder der Negator des `=`-Mitglieds einer B-Tree-Operatorklasse ist. Das Standardverhalten der oben genannten Operatoren entspricht dem Verhalten von `IS [ NOT ] DISTINCT FROM` für Zeilenkonstruktoren; siehe Abschnitt 9.25.5.

Um das Vergleichen von Zeilen zu unterstützen, die Elemente ohne standardmäßige B-Tree-Operatorklasse enthalten, sind für den Vergleich zusammengesetzter Typen die folgenden Operatoren definiert: `*=`, `*<>`, `*<`, `*<=`, `*>` und `*>=`. Diese Operatoren vergleichen die interne binäre Darstellung der beiden Zeilen. Zwei Zeilen können eine unterschiedliche binäre Darstellung haben, obwohl der Vergleich der beiden Zeilen mit dem Gleichheitsoperator `true` ergibt. Die Ordnung von Zeilen unter diesen Vergleichsoperatoren ist deterministisch, aber ansonsten nicht sinnvoll. Diese Operatoren werden intern für materialisierte Sichten verwendet und können für andere Spezialzwecke nützlich sein, etwa Replikation und B-Tree-Deduplizierung, siehe Abschnitt 65.1.4.3. Sie sind jedoch nicht dafür gedacht, beim Schreiben allgemeiner Abfragen nützlich zu sein.

## 9.26. Mengenliefernde Funktionen

Dieser Abschnitt beschreibt Funktionen, die möglicherweise mehr als eine Zeile zurückgeben. Die meistverwendeten Funktionen dieser Klasse sind Funktionen zum Erzeugen von Reihen, wie in Tabelle 9.69 und Tabelle 9.70 beschrieben. Andere, spezialisiertere mengenliefernde Funktionen werden an anderer Stelle in diesem Handbuch beschrieben. Möglichkeiten, mehrere mengenliefernde Funktionen zu kombinieren, finden Sie in Abschnitt 7.2.1.4.

**Tabelle 9.69. Funktionen zum Erzeugen von Reihen**

| Funktion | Beschreibung |
|---|---|
| `generate_series ( start integer, stop integer [, step integer ] )` -> `setof integer`<br>`generate_series ( start bigint, stop bigint [, step bigint ] )` -> `setof bigint`<br>`generate_series ( start numeric, stop numeric [, step numeric ] )` -> `setof numeric` | Erzeugt eine Reihe von Werten von `start` bis `stop` mit der Schrittweite `step`. `step` ist standardmäßig 1. |
| `generate_series ( start timestamp, stop timestamp, step interval )` -> `setof timestamp`<br>`generate_series ( start timestamp with time zone, stop timestamp with time zone, step interval [, timezone text ] )` -> `setof timestamp with time zone` | Erzeugt eine Reihe von Werten von `start` bis `stop` mit der Schrittweite `step`. In der zeitzonenbewussten Form werden Tageszeiten und Sommerzeitanpassungen gemäß der durch das Argument `timezone` benannten Zeitzone berechnet, oder gemäß der aktuellen Einstellung `TimeZone`, wenn das Argument ausgelassen wird. |

Wenn `step` positiv ist, werden null Zeilen zurückgegeben, falls `start` größer als `stop` ist. Umgekehrt werden bei negativem `step` null Zeilen zurückgegeben, wenn `start` kleiner als `stop` ist. Null Zeilen werden auch zurückgegeben, wenn irgendeine Eingabe `NULL` ist. Ein `step` von null ist ein Fehler. Es folgen einige Beispiele:

```sql
SELECT * FROM generate_series(2,4);
```

```text
 generate_series
-----------------
               2
               3
               4
(3 rows)
```

```sql
SELECT * FROM generate_series(5,1,-2);
```

```text
 generate_series
-----------------
               5
               3
               1
(3 rows)
```

```sql
SELECT * FROM generate_series(4,3);
```

```text
 generate_series
-----------------
(0 rows)
```

```sql
SELECT generate_series(1.1, 4, 1.3);
```

```text
 generate_series
-----------------
             1.1
             2.4
             3.7
(3 rows)
```

```sql
-- Dieses Beispiel stützt sich auf den Operator Datum-plus-Ganzzahl:
SELECT current_date + s.a AS dates
FROM generate_series(0,14,7) AS s(a);
```

```text
   dates
------------
 2004-02-05
 2004-02-12
 2004-02-19
(3 rows)
```

```sql
SELECT *
FROM generate_series('2008-03-01 00:00'::timestamp,
                     '2008-03-04 12:00', '10 hours');
```

```text
   generate_series
---------------------
 2008-03-01 00:00:00
 2008-03-01 10:00:00
 2008-03-01 20:00:00
 2008-03-02 06:00:00
 2008-03-02 16:00:00
 2008-03-03 02:00:00
 2008-03-03 12:00:00
 2008-03-03 22:00:00
 2008-03-04 08:00:00
(9 rows)
```

```sql
-- Dieses Beispiel nimmt an, dass TimeZone auf UTC gesetzt ist;
-- beachten Sie den Übergang zur Winterzeit:
SELECT *
FROM generate_series('2001-10-22 00:00 -04:00'::timestamptz,
                     '2001-11-01 00:00 -05:00'::timestamptz,
                     '1 day'::interval, 'America/New_York');
```

```text
    generate_series
------------------------
 2001-10-22 04:00:00+00
 2001-10-23 04:00:00+00
 2001-10-24 04:00:00+00
 2001-10-25 04:00:00+00
 2001-10-26 04:00:00+00
 2001-10-27 04:00:00+00
 2001-10-28 04:00:00+00
 2001-10-29 05:00:00+00
 2001-10-30 05:00:00+00
 2001-10-31 05:00:00+00
 2001-11-01 05:00:00+00
(11 rows)
```

**Tabelle 9.70. Funktionen zum Erzeugen von Indizes**

| Funktion | Beschreibung |
|---|---|
| `generate_subscripts ( array anyarray, dim integer )` -> `setof integer` | Erzeugt eine Reihe, die die gültigen Indizes der `dim`-ten Dimension des gegebenen Arrays umfasst. |
| `generate_subscripts ( array anyarray, dim integer, reverse boolean )` -> `setof integer` | Erzeugt eine Reihe, die die gültigen Indizes der `dim`-ten Dimension des gegebenen Arrays umfasst. Wenn `reverse` true ist, wird die Reihe in umgekehrter Reihenfolge zurückgegeben. |

`generate_subscripts` ist eine Hilfsfunktion, die die Menge gültiger Indizes für die angegebene Dimension des gegebenen Arrays erzeugt. Null Zeilen werden zurückgegeben, wenn Arrays die angeforderte Dimension nicht haben oder wenn irgendeine Eingabe `NULL` ist. Es folgen einige Beispiele:

```sql
-- Grundlegende Verwendung:
SELECT generate_subscripts('{NULL,1,NULL,2}'::int[], 1) AS s;
```

```text
 s
---
 1
 2
 3
 4
(4 rows)
```

```sql
-- Um ein Array, den Index und den indizierten Wert anzuzeigen,
-- ist eine Unterabfrage erforderlich:
SELECT * FROM arrays;
```

```text
         a
--------------------
 {-1,-2}
 {100,200,300}
(2 rows)
```

```sql
SELECT a AS array, s AS subscript, a[s] AS value
FROM (SELECT generate_subscripts(a, 1) AS s, a FROM arrays) foo;
```

```text
     array     | subscript | value
---------------+-----------+-------
 {-1,-2}       |         1 |    -1
 {-1,-2}       |         2 |    -2
 {100,200,300} |         1 |   100
 {100,200,300} |         2 |   200
 {100,200,300} |         3 |   300
(5 rows)
```

```sql
-- 2D-Array entpacken:
CREATE OR REPLACE FUNCTION unnest2(anyarray)
RETURNS SETOF anyelement AS $$
SELECT $1[i][j]
   FROM generate_subscripts($1,1) g1(i),
        generate_subscripts($1,2) g2(j);
$$ LANGUAGE sql IMMUTABLE;
```

```text
CREATE FUNCTION
```

```sql
SELECT * FROM unnest2(ARRAY[[1,2],[3,4]]);
```

```text
 unnest2
---------
       1
       2
       3
       4
(4 rows)
```

Wenn eine Funktion in der `FROM`-Klausel mit `WITH ORDINALITY` versehen wird, wird an die Ausgabespalte oder Ausgabespalten der Funktion eine `bigint`-Spalte angehängt. Diese beginnt bei 1 und wird für jede Zeile der Funktionsausgabe um 1 erhöht. Das ist besonders nützlich bei mengenliefernden Funktionen wie `unnest()`.

```sql
-- Mengenliefernde Funktion mit WITH ORDINALITY:
SELECT * FROM pg_ls_dir('.') WITH ORDINALITY AS t(ls,n);
```

```text
        ls       | n
-----------------+----
 pg_serial       | 1
 pg_twophase     | 2
 postmaster.opts | 3
 pg_notify       | 4
 postgresql.conf | 5
 pg_tblspc       | 6
 logfile         | 7
 base            | 8
 postmaster.pid  | 9
 pg_ident.conf   | 10
 global          | 11
 pg_xact         | 12
 pg_snapshots    | 13
 pg_multixact    | 14
 PG_VERSION      | 15
 pg_wal          | 16
 pg_hba.conf     | 17
 pg_stat_tmp     | 18
 pg_subtrans     | 19
(19 rows)
```

## 9.27. Systeminformationsfunktionen und -operatoren

Die in diesem Abschnitt beschriebenen Funktionen werden verwendet, um verschiedene Informationen über eine PostgreSQL-Installation zu erhalten.

### 9.27.1. Sitzungsinformationsfunktionen

Tabelle 9.71 zeigt mehrere Funktionen, die Sitzungs- und Systeminformationen ermitteln.

Zusätzlich zu den in diesem Abschnitt aufgeführten Funktionen gibt es eine Reihe von Funktionen im Zusammenhang mit dem Statistiksystem, die ebenfalls Systeminformationen bereitstellen. Weitere Informationen finden Sie in Abschnitt 27.2.26.

**Tabelle 9.71. Sitzungsinformationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `current_catalog` -> `name`<br>`current_database ()` -> `name` | Gibt den Namen der aktuellen Datenbank zurück. Im SQL-Standard heißen Datenbanken „Kataloge“, daher ist `current_catalog` die Schreibweise des Standards. |
| `current_query ()` -> `text` | Gibt den Text der aktuell ausgeführten Abfrage zurück, so wie sie vom Client übermittelt wurde. Dieser Text kann mehr als eine Anweisung enthalten. |
| `current_role` -> `name` | Entspricht `current_user`. |
| `current_schema` -> `name`<br>`current_schema ()` -> `name` | Gibt den Namen des Schemas zurück, das im Suchpfad an erster Stelle steht, oder null, wenn der Suchpfad leer ist. Dieses Schema wird für Tabellen und andere benannte Objekte verwendet, die ohne Zielschema erstellt werden. |
| `current_schemas ( include_implicit boolean )` -> `name[]` | Gibt ein Array mit den Namen aller Schemas zurück, die derzeit im effektiven Suchpfad stehen, und zwar in ihrer Prioritätsreihenfolge. Einträge der aktuellen `search_path`-Einstellung, die keinem vorhandenen, durchsuchbaren Schema entsprechen, werden ausgelassen. Wenn das boolesche Argument true ist, werden implizit durchsuchte Systemschemas wie `pg_catalog` einbezogen. |
| `current_user` -> `name` | Gibt den Benutzernamen des aktuellen Ausführungskontexts zurück. |
| `inet_client_addr ()` -> `inet` | Gibt die IP-Adresse des aktuellen Clients zurück oder `NULL`, wenn die aktuelle Verbindung über einen Unix-Domain-Socket läuft. |
| `inet_client_port ()` -> `integer` | Gibt die IP-Portnummer des aktuellen Clients zurück oder `NULL`, wenn die aktuelle Verbindung über einen Unix-Domain-Socket läuft. |
| `inet_server_addr ()` -> `inet` | Gibt die IP-Adresse zurück, auf der der Server die aktuelle Verbindung angenommen hat, oder `NULL`, wenn die aktuelle Verbindung über einen Unix-Domain-Socket läuft. |
| `inet_server_port ()` -> `integer` | Gibt die IP-Portnummer zurück, auf der der Server die aktuelle Verbindung angenommen hat, oder `NULL`, wenn die aktuelle Verbindung über einen Unix-Domain-Socket läuft. |
| `pg_backend_pid ()` -> `integer` | Gibt die Prozess-ID des Serverprozesses zurück, der der aktuellen Sitzung zugeordnet ist. |
| `pg_blocking_pids ( integer )` -> `integer[]` | Gibt ein Array mit den Prozess-IDs der Sitzungen zurück, die den Serverprozess mit der angegebenen Prozess-ID daran hindern, eine Sperre zu erhalten, oder ein leeres Array, wenn es keinen solchen Serverprozess gibt oder er nicht blockiert ist. Ein Serverprozess blockiert einen anderen entweder, weil er eine Sperre hält, die mit der angeforderten Sperre kollidiert, oder weil er selbst auf eine Sperre wartet, die mit der angeforderten Sperre kollidieren würde, und in der Warteschlange vor ihm steht. Bei parallelen Abfragen enthält das Ergebnis immer client-sichtbare Prozess-IDs, auch wenn die eigentliche Sperre von einem Workerprozess gehalten oder erwartet wird. Dadurch können doppelte PIDs auftreten. Eine vorbereitete Transaktion mit kollidierender Sperre wird als Prozess-ID null dargestellt. Häufige Aufrufe können die Datenbankleistung beeinflussen, weil kurz exklusiver Zugriff auf den gemeinsamen Zustand des Sperrmanagers benötigt wird. |
| `pg_conf_load_time ()` -> `timestamp with time zone` | Gibt den Zeitpunkt zurück, zu dem die Serverkonfigurationsdateien zuletzt geladen wurden. Wenn die aktuelle Sitzung zu diesem Zeitpunkt bereits existierte, ist dies der Zeitpunkt, zu dem die Sitzung selbst die Konfigurationsdateien erneut gelesen hat; der Wert kann daher zwischen Sitzungen etwas variieren. Andernfalls ist es der Zeitpunkt, zu dem der Postmasterprozess die Konfigurationsdateien erneut gelesen hat. |
| `pg_current_logfile ( [ text ] )` -> `text` | Gibt den Pfadnamen der aktuell vom Logging-Collector verwendeten Logdatei zurück. Der Pfad enthält `log_directory` und den einzelnen Logdateinamen. Das Ergebnis ist `NULL`, wenn der Logging-Collector deaktiviert ist. Wenn mehrere Logdateien in unterschiedlichen Formaten existieren, gibt `pg_current_logfile` ohne Argument den Pfad der Datei mit dem ersten Format in der geordneten Liste `stderr`, `csvlog`, `jsonlog` zurück. `NULL` wird zurückgegeben, wenn keine Logdatei eines dieser Formate vorhanden ist. Um ein bestimmtes Logformat anzufordern, übergeben Sie `csvlog`, `jsonlog` oder `stderr`. Das Ergebnis ist `NULL`, wenn das angeforderte Format in `log_destination` nicht konfiguriert ist. Das Ergebnis spiegelt den Inhalt der Datei `current_logfiles` wider. Standardmäßig ist diese Funktion auf Superuser und Rollen mit Rechten der Rolle `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_get_loaded_modules ()` -> `setof record ( module_name text, version text, file_name text )` | Gibt eine Liste der ladbaren Module zurück, die in der aktuellen Serversitzung geladen sind. Die Felder `module_name` und `version` sind `NULL`, sofern der Modulauthor keine Werte über das Makro `PG_MODULE_MAGIC_EXT` angegeben hat. Das Feld `file_name` enthält den Dateinamen des Moduls, also der Shared Library. |
| `pg_my_temp_schema ()` -> `oid` | Gibt die OID des temporären Schemas der aktuellen Sitzung zurück oder null, wenn die Sitzung keines hat, weil sie keine temporären Tabellen erstellt hat. |
| `pg_is_other_temp_schema ( oid )` -> `boolean` | Gibt true zurück, wenn die angegebene OID die OID des temporären Schemas einer anderen Sitzung ist. Das kann zum Beispiel nützlich sein, um temporäre Tabellen anderer Sitzungen aus einer Kataloganzeige auszuschließen. |
| `pg_jit_available ()` -> `boolean` | Gibt true zurück, wenn eine JIT-Compiler-Erweiterung verfügbar ist, siehe [Kapitel 30](30_Just_in_Time_Kompilierung_JIT.md), und der Konfigurationsparameter `jit` auf `on` gesetzt ist. |
| `pg_numa_available ()` -> `boolean` | Gibt true zurück, wenn der Server mit NUMA-Unterstützung kompiliert wurde. |
| `pg_listening_channels ()` -> `setof text` | Gibt die Namen der asynchronen Benachrichtigungskanäle zurück, auf die die aktuelle Sitzung hört. |
| `pg_notification_queue_usage ()` -> `double precision` | Gibt den Anteil von 0 bis 1 der maximalen Größe der asynchronen Benachrichtigungswarteschlange zurück, der aktuell durch noch zu verarbeitende Benachrichtigungen belegt ist. Weitere Informationen finden Sie bei `LISTEN` und `NOTIFY`. |
| `pg_postmaster_start_time ()` -> `timestamp with time zone` | Gibt den Zeitpunkt zurück, zu dem der Server gestartet wurde. |
| `pg_safe_snapshot_blocking_pids ( integer )` -> `integer[]` | Gibt ein Array mit den Prozess-IDs der Sitzungen zurück, die den Serverprozess mit der angegebenen Prozess-ID daran hindern, einen sicheren Snapshot zu erhalten, oder ein leeres Array, wenn es keinen solchen Serverprozess gibt oder er nicht blockiert ist. Eine Sitzung mit einer `SERIALIZABLE`-Transaktion blockiert eine `SERIALIZABLE READ ONLY DEFERRABLE`-Transaktion beim Erhalt eines Snapshots, bis letztere festgestellt hat, dass sie ohne Prädikatsperren sicher arbeiten kann. Weitere Informationen zu serialisierbaren und aufschiebbaren Transaktionen finden Sie in Abschnitt 13.2.3. Häufige Aufrufe können die Datenbankleistung beeinflussen, weil kurz Zugriff auf den gemeinsamen Zustand des Prädikatsperrenmanagers benötigt wird. |
| `pg_trigger_depth ()` -> `integer` | Gibt die aktuelle Verschachtelungstiefe von PostgreSQL-Triggern zurück, oder 0, wenn der Aufruf nicht direkt oder indirekt aus einem Trigger erfolgt. |
| `session_user` -> `name` | Gibt den Namen des Sitzungsbenutzers zurück. |
| `system_user` -> `text` | Gibt die Authentifizierungsmethode und die Identität zurück, die der Benutzer während der Authentifizierung präsentiert hat, bevor ihm eine Datenbankrolle zugewiesen wurde. Die Darstellung lautet `auth_method:identity`; `NULL` wird zurückgegeben, wenn der Benutzer nicht authentifiziert wurde, etwa bei Trust-Authentifizierung. |
| `user` -> `name` | Entspricht `current_user`. |

> **Hinweis**
> `current_catalog`, `current_role`, `current_schema`, `current_user`, `session_user` und `user` haben in SQL einen besonderen syntaktischen Status: Sie müssen ohne nachfolgende Klammern aufgerufen werden. In PostgreSQL können bei `current_schema` optional Klammern verwendet werden, bei den anderen jedoch nicht.

`session_user` ist normalerweise der Benutzer, der die aktuelle Datenbankverbindung initiiert hat; Superuser können diese Einstellung jedoch mit `SET SESSION AUTHORIZATION` ändern. `current_user` ist der Benutzerbezeichner, der für Berechtigungsprüfungen maßgeblich ist. Normalerweise entspricht er dem Sitzungsbenutzer, kann aber mit `SET ROLE` geändert werden. Er ändert sich außerdem während der Ausführung von Funktionen mit dem Attribut `SECURITY DEFINER`. In Unix-Begriffen ist der Sitzungsbenutzer der „reale Benutzer“ und der aktuelle Benutzer der „effektive Benutzer“. `current_role` und `user` sind Synonyme für `current_user`. Der SQL-Standard unterscheidet zwischen `current_role` und `current_user`; PostgreSQL tut dies nicht, weil es Benutzer und Rollen zu einer einzigen Entitätsart zusammenfasst.

### 9.27.2. Funktionen zur Abfrage von Zugriffsrechten

Tabelle 9.72 führt Funktionen auf, mit denen Objektzugriffsrechte programmatisch abgefragt werden können. Weitere Informationen zu Berechtigungen finden Sie in [Abschnitt 5.8](05_Datendefinition.md#58-privilegien). In diesen Funktionen kann der Benutzer, dessen Rechte abgefragt werden, per Name oder per OID (`pg_authid.oid`) angegeben werden; wenn der Name `public` lautet, werden die Rechte der Pseudorolle `PUBLIC` geprüft. Das Benutzerargument kann auch vollständig ausgelassen werden; dann wird `current_user` angenommen. Auch das abgefragte Objekt kann per Name oder per OID angegeben werden. Bei Namensangaben kann, sofern relevant, ein Schemaname enthalten sein. Das interessierende Zugriffsrecht wird als Textzeichenkette angegeben, die zu einem der passenden Privileg-Schlüsselwörter für den Objekttyp auswerten muss, zum Beispiel `SELECT`. Optional kann `WITH GRANT OPTION` an einen Privilegtyp angehängt werden, um zu prüfen, ob das Recht mit Grant-Option gehalten wird. Außerdem können mehrere Privilegtypen durch Kommas getrennt aufgeführt werden; dann ist das Ergebnis true, wenn irgendeines der aufgeführten Rechte vorhanden ist. Die Groß-/Kleinschreibung der Privilegzeichenkette ist unerheblich, und zusätzlicher Leerraum ist zwischen, aber nicht innerhalb von Privilegnamen erlaubt. Einige Beispiele:

```sql
SELECT has_table_privilege('myschema.mytable', 'select');
SELECT has_table_privilege('joe', 'mytable', 'INSERT, SELECT WITH GRANT OPTION');
```

**Tabelle 9.72. Funktionen zur Abfrage von Zugriffsrechten**

| Funktion | Beschreibung |
|---|---|
| `has_any_column_privilege ( [ user name or oid, ] table text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für irgendeine Spalte der Tabelle? Dies ist erfolgreich, wenn das Recht für die ganze Tabelle besteht oder wenn es ein spaltenbezogenes Grant dieses Rechts für mindestens eine Spalte gibt. Zulässige Privilegtypen sind `SELECT`, `INSERT`, `UPDATE` und `REFERENCES`. |
| `has_column_privilege ( [ user name or oid, ] table text or oid, column text or smallint, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für die angegebene Tabellenspalte? Dies ist erfolgreich, wenn das Recht für die ganze Tabelle besteht oder wenn es ein spaltenbezogenes Grant dieses Rechts für die Spalte gibt. Die Spalte kann per Name oder Attributnummer (`pg_attribute.attnum`) angegeben werden. Zulässige Privilegtypen sind `SELECT`, `INSERT`, `UPDATE` und `REFERENCES`. |
| `has_database_privilege ( [ user name or oid, ] database text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für die Datenbank? Zulässige Privilegtypen sind `CREATE`, `CONNECT`, `TEMPORARY` und `TEMP`, wobei `TEMP` `TEMPORARY` entspricht. |
| `has_foreign_data_wrapper_privilege ( [ user name or oid, ] fdw text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für den Foreign-Data Wrapper? Der einzige zulässige Privilegtyp ist `USAGE`. |
| `has_function_privilege ( [ user name or oid, ] function text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für die Funktion? Der einzige zulässige Privilegtyp ist `EXECUTE`. Wenn eine Funktion per Name statt per OID angegeben wird, entspricht die erlaubte Eingabe dem Datentyp `regprocedure`, siehe [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen). Beispiel: `SELECT has_function_privilege('joeuser', 'myfunc(int, text)', 'execute');` |
| `has_language_privilege ( [ user name or oid, ] language text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für die Sprache? Der einzige zulässige Privilegtyp ist `USAGE`. |
| `has_largeobject_privilege ( [ user name or oid, ] largeobject oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für das Large Object? Zulässige Privilegtypen sind `SELECT` und `UPDATE`. |
| `has_parameter_privilege ( [ user name or oid, ] parameter text, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für den Konfigurationsparameter? Der Parametername ist unabhängig von Groß-/Kleinschreibung. Zulässige Privilegtypen sind `SET` und `ALTER SYSTEM`. |
| `has_schema_privilege ( [ user name or oid, ] schema text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für das Schema? Zulässige Privilegtypen sind `CREATE` und `USAGE`. |
| `has_sequence_privilege ( [ user name or oid, ] sequence text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für die Sequenz? Zulässige Privilegtypen sind `USAGE`, `SELECT` und `UPDATE`. |
| `has_server_privilege ( [ user name or oid, ] server text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für den fremden Server? Der einzige zulässige Privilegtyp ist `USAGE`. |
| `has_table_privilege ( [ user name or oid, ] table text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für die Tabelle? Zulässige Privilegtypen sind `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER` und `MAINTAIN`. |
| `has_tablespace_privilege ( [ user name or oid, ] tablespace text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für den Tablespace? Der einzige zulässige Privilegtyp ist `CREATE`. |
| `has_type_privilege ( [ user name or oid, ] type text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für den Datentyp? Der einzige zulässige Privilegtyp ist `USAGE`. Wenn ein Typ per Name statt per OID angegeben wird, entspricht die erlaubte Eingabe dem Datentyp `regtype`, siehe [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen). |
| `pg_has_role ( [ user name or oid, ] role text or oid, privilege text )` -> `boolean` | Hat der Benutzer ein Recht für die Rolle? Zulässige Privilegtypen sind `MEMBER`, `USAGE` und `SET`. `MEMBER` bezeichnet direkte oder indirekte Mitgliedschaft in der Rolle, unabhängig davon, welche konkreten Rechte verliehen werden. `USAGE` bezeichnet, ob die Rechte der Rolle unmittelbar verfügbar sind, ohne `SET ROLE` auszuführen; `SET` bezeichnet, ob mit `SET ROLE` in die Rolle gewechselt werden kann. `WITH ADMIN OPTION` oder `WITH GRANT OPTION` kann an jeden dieser Privilegtypen angehängt werden, um zu prüfen, ob das Admin-Recht gehalten wird; alle sechs Schreibweisen prüfen dasselbe. Diese Funktion erlaubt den Sonderfall `public` für `user` nicht, weil die Pseudorolle `PUBLIC` nie Mitglied realer Rollen sein kann. |
| `row_security_active ( table text or oid )` -> `boolean` | Ist Row-Level Security für die angegebene Tabelle im Kontext des aktuellen Benutzers und der aktuellen Umgebung aktiv? |

Tabelle 9.73 zeigt die Operatoren, die für den Typ `aclitem` verfügbar sind. Dieser Typ ist die Katalogdarstellung von Zugriffsrechten. Informationen zum Lesen von Zugriffsrechtswerten finden Sie in [Abschnitt 5.8](05_Datendefinition.md#58-privilegien).

**Tabelle 9.73. `aclitem`-Operatoren**

| Operator | Beschreibung | Beispiel |
|---|---|---|
| `aclitem = aclitem` -> `boolean` | Sind die `aclitem`-Werte gleich? Beachten Sie, dass der Typ `aclitem` nicht den üblichen Satz von Vergleichsoperatoren besitzt, sondern nur Gleichheit. Entsprechend können auch `aclitem`-Arrays nur auf Gleichheit verglichen werden. | `'calvin=r*w/hobbes'::aclitem = 'calvin=r*w*/hobbes'::aclitem` -> `f` |
| `aclitem[] @> aclitem` -> `boolean` | Enthält das Array die angegebenen Rechte? Dies ist true, wenn es einen Arrayeintrag gibt, der Berechtigtem und Grantor des `aclitem` entspricht und mindestens die angegebene Menge von Rechten besitzt. | `'{calvin=r*w/hobbes,hobbes=r*w*/postgres}'::aclitem[] @> 'calvin=r*/hobbes'::aclitem` -> `t` |
| `aclitem[] ~ aclitem` -> `boolean` | Dies ist ein veralteter Alias für `@>`. | `'{calvin=r*w/hobbes,hobbes=r*w*/postgres}'::aclitem[] ~ 'calvin=r*/hobbes'::aclitem` -> `t` |

Tabelle 9.74 zeigt einige zusätzliche Funktionen zur Verwaltung des Typs `aclitem`.

**Tabelle 9.74. `aclitem`-Funktionen**

| Funktion | Beschreibung |
|---|---|
| `acldefault ( type "char", ownerId oid )` -> `aclitem[]` | Konstruiert ein `aclitem`-Array mit den Standardzugriffsrechten für ein Objekt des Typs `type`, das der Rolle mit der OID `ownerId` gehört. Dies stellt die Zugriffsrechte dar, die angenommen werden, wenn der ACL-Eintrag eines Objekts null ist. Die Standardzugriffsrechte sind in [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) beschrieben. Der Parameter `type` muss einer der folgenden Werte sein: `'c'` für `COLUMN`, `'r'` für `TABLE` und tabellenartige Objekte, `'s'` für `SEQUENCE`, `'d'` für `DATABASE`, `'f'` für `FUNCTION` oder `PROCEDURE`, `'l'` für `LANGUAGE`, `'L'` für `LARGE OBJECT`, `'n'` für `SCHEMA`, `'p'` für `PARAMETER`, `'t'` für `TABLESPACE`, `'F'` für `FOREIGN DATA WRAPPER`, `'S'` für `FOREIGN SERVER` oder `'T'` für `TYPE` oder `DOMAIN`. |
| `aclexplode ( aclitem[] )` -> `setof record ( grantor oid, grantee oid, privilege_type text, is_grantable boolean )` | Gibt das `aclitem`-Array als Menge von Zeilen zurück. Wenn der Berechtigte die Pseudorolle `PUBLIC` ist, wird er in der Spalte `grantee` als null dargestellt. Jedes gewährte Recht wird als `SELECT`, `INSERT` usw. dargestellt, siehe Tabelle 5.1 für die vollständige Liste. Beachten Sie, dass jedes Recht als eigene Zeile ausgegeben wird, sodass in der Spalte `privilege_type` nur ein Schlüsselwort steht. |
| `makeaclitem ( grantee oid, grantor oid, privileges text, is_grantable boolean )` -> `aclitem` | Konstruiert ein `aclitem` mit den angegebenen Eigenschaften. `privileges` ist eine kommagetrennte Liste von Privilegnamen wie `SELECT`, `INSERT` usw.; alle werden im Ergebnis gesetzt. Die Groß-/Kleinschreibung der Privilegzeichenkette ist unerheblich, und zusätzlicher Leerraum ist zwischen, aber nicht innerhalb von Privilegnamen erlaubt. |

### 9.27.3. Funktionen zur Abfrage der Schemasichtbarkeit

Tabelle 9.75 zeigt Funktionen, die bestimmen, ob ein bestimmtes Objekt im aktuellen Schemasuchpfad sichtbar ist. Eine Tabelle gilt zum Beispiel als sichtbar, wenn ihr enthaltendes Schema im Suchpfad steht und keine Tabelle gleichen Namens früher im Suchpfad erscheint. Das ist gleichbedeutend damit, dass die Tabelle ohne ausdrückliche Schemaqualifikation per Name referenziert werden kann. Um also die Namen aller sichtbaren Tabellen aufzulisten:

```sql
SELECT relname FROM pg_class WHERE pg_table_is_visible(oid);
```

Für Funktionen und Operatoren gilt ein Objekt im Suchpfad als sichtbar, wenn es früher im Pfad kein Objekt mit demselben Namen und denselben Argumentdatentypen gibt. Bei Operatorklassen und -familien werden sowohl der Name als auch die zugehörige Indexzugriffsmethode berücksichtigt.

**Tabelle 9.75. Funktionen zur Abfrage der Schemasichtbarkeit**

| Funktion | Beschreibung |
|---|---|
| `pg_collation_is_visible ( collation oid )` -> `boolean` | Ist die Kollation im Suchpfad sichtbar? |
| `pg_conversion_is_visible ( conversion oid )` -> `boolean` | Ist die Konvertierung im Suchpfad sichtbar? |
| `pg_function_is_visible ( function oid )` -> `boolean` | Ist die Funktion im Suchpfad sichtbar? Dies funktioniert auch für Prozeduren und Aggregate. |
| `pg_opclass_is_visible ( opclass oid )` -> `boolean` | Ist die Operatorklasse im Suchpfad sichtbar? |
| `pg_operator_is_visible ( operator oid )` -> `boolean` | Ist der Operator im Suchpfad sichtbar? |
| `pg_opfamily_is_visible ( opclass oid )` -> `boolean` | Ist die Operatorfamilie im Suchpfad sichtbar? |
| `pg_statistics_obj_is_visible ( stat oid )` -> `boolean` | Ist das Statistikobjekt im Suchpfad sichtbar? |
| `pg_table_is_visible ( table oid )` -> `boolean` | Ist die Tabelle im Suchpfad sichtbar? Dies funktioniert für alle Arten von Relationen, einschließlich Views, materialisierten Views, Indizes, Sequenzen und Fremdtabellen. |
| `pg_ts_config_is_visible ( config oid )` -> `boolean` | Ist die Textsuchkonfiguration im Suchpfad sichtbar? |
| `pg_ts_dict_is_visible ( dict oid )` -> `boolean` | Ist das Textsuchwörterbuch im Suchpfad sichtbar? |
| `pg_ts_parser_is_visible ( parser oid )` -> `boolean` | Ist der Textsuchparser im Suchpfad sichtbar? |
| `pg_ts_template_is_visible ( template oid )` -> `boolean` | Ist die Textsuchvorlage im Suchpfad sichtbar? |
| `pg_type_is_visible ( type oid )` -> `boolean` | Ist der Typ oder die Domain im Suchpfad sichtbar? |

Alle diese Funktionen benötigen Objekt-OIDs, um das zu prüfende Objekt zu identifizieren. Wenn Sie ein Objekt per Name testen möchten, ist die Verwendung der OID-Aliastypen bequem, etwa `regclass`, `regtype`, `regprocedure`, `regoperator`, `regconfig` oder `regdictionary`:

```sql
SELECT pg_type_is_visible('myschema.widget'::regtype);
```

Beachten Sie, dass es wenig sinnvoll wäre, auf diese Weise einen nicht schemaqualifizierten Typnamen zu testen: Wenn der Name überhaupt erkannt werden kann, muss er sichtbar sein.

### 9.27.4. Systemkatalog-Informationsfunktionen

Tabelle 9.76 führt Funktionen auf, die Informationen aus den Systemkatalogen extrahieren.

**Tabelle 9.76. Systemkatalog-Informationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `format_type ( type oid, typemod integer )` -> `text` | Gibt den SQL-Namen für einen Datentyp zurück, der durch seine Typ-OID und möglicherweise einen Typmodifikator identifiziert wird. Übergeben Sie `NULL` für den Typmodifikator, wenn kein spezifischer Modifikator bekannt ist. |
| `pg_basetype ( regtype )` -> `regtype` | Gibt die OID des Basistyps einer Domain zurück, die durch ihre Typ-OID identifiziert wird. Wenn das Argument die OID eines Nicht-Domain-Typs ist, wird das Argument unverändert zurückgegeben. Gibt `NULL` zurück, wenn das Argument keine gültige Typ-OID ist. Gibt es eine Kette von Domain-Abhängigkeiten, wird rekursiv bis zum Basistyp gesucht. Beispiel bei `CREATE DOMAIN mytext AS text`: `pg_basetype('mytext'::regtype)` -> `text`. |
| `pg_char_to_encoding ( encoding name )` -> `integer` | Wandelt den angegebenen Encodingnamen in eine Ganzzahl um, die den internen Bezeichner darstellt, der in einigen Systemkatalogtabellen verwendet wird. Gibt -1 zurück, wenn ein unbekannter Encodingname angegeben wird. |
| `pg_encoding_to_char ( encoding integer )` -> `name` | Wandelt die Ganzzahl, die als interner Encodingbezeichner in einigen Systemkatalogtabellen verwendet wird, in eine menschenlesbare Zeichenkette um. Gibt eine leere Zeichenkette zurück, wenn eine ungültige Encodingnummer angegeben wird. |
| `pg_get_catalog_foreign_keys ()` -> `setof record ( fktable regclass, fkcols text[], pktable regclass, pkcols text[], is_array boolean, is_opt boolean )` | Gibt Datensätze zurück, die die Fremdschlüsselbeziehungen innerhalb der PostgreSQL-Systemkataloge beschreiben. `fktable` enthält den Namen des referenzierenden Katalogs, `fkcols` die Namen der referenzierenden Spalten. Entsprechend enthält `pktable` den referenzierten Katalog und `pkcols` die referenzierten Spalten. Wenn `is_array` true ist, ist die letzte referenzierende Spalte ein Array, dessen Elemente jeweils zu einem Eintrag im referenzierten Katalog passen sollten. Wenn `is_opt` true ist, dürfen die referenzierenden Spalten statt einer gültigen Referenz Nullen enthalten. |
| `pg_get_constraintdef ( constraint oid [, pretty boolean ] )` -> `text` | Rekonstruiert den Erstellungsbefehl für einen Constraint. Dies ist eine dekompilierte Rekonstruktion, nicht der ursprüngliche Befehlstext. |
| `pg_get_expr ( expr pg_node_tree, relation oid [, pretty boolean ] )` -> `text` | Dekompiliert die interne Form eines Ausdrucks, der in den Systemkatalogen gespeichert ist, etwa den Standardwert einer Spalte. Wenn der Ausdruck `Var`s enthalten kann, geben Sie als zweiten Parameter die OID der Relation an, auf die sie sich beziehen; wenn keine `Var`s erwartet werden, genügt null. |
| `pg_get_functiondef ( func oid )` -> `text` | Rekonstruiert den Erstellungsbefehl für eine Funktion oder Prozedur. Das Ergebnis ist eine vollständige `CREATE OR REPLACE FUNCTION`- beziehungsweise `CREATE OR REPLACE PROCEDURE`-Anweisung. |
| `pg_get_function_arguments ( func oid )` -> `text` | Rekonstruiert die Argumentliste einer Funktion oder Prozedur in der Form, in der sie innerhalb von `CREATE FUNCTION` erscheinen müsste, einschließlich Standardwerten. |
| `pg_get_function_identity_arguments ( func oid )` -> `text` | Rekonstruiert die Argumentliste, die zur Identifikation einer Funktion oder Prozedur notwendig ist, in der Form, in der sie in Befehlen wie `ALTER FUNCTION` erscheinen müsste. Diese Form lässt Standardwerte weg. |
| `pg_get_function_result ( func oid )` -> `text` | Rekonstruiert die `RETURNS`-Klausel einer Funktion in der Form, in der sie innerhalb von `CREATE FUNCTION` erscheinen müsste. Gibt für eine Prozedur `NULL` zurück. |
| `pg_get_indexdef ( index oid [, column integer, pretty boolean ] )` -> `text` | Rekonstruiert den Erstellungsbefehl für einen Index. Wenn `column` angegeben und nicht null ist, wird nur die Definition dieser Spalte rekonstruiert. |
| `pg_get_keywords ()` -> `setof record ( word text, catcode "char", barelabel boolean, catdesc text, baredesc text )` | Gibt Datensätze zurück, die die vom Server erkannten SQL-Schlüsselwörter beschreiben. `word` enthält das Schlüsselwort. `catcode` enthält einen Kategoriecode: `U` für nicht reserviert, `C` für ein Schlüsselwort, das Spaltenname sein kann, `T` für ein Schlüsselwort, das Typ- oder Funktionsname sein kann, und `R` für vollständig reserviert. `barelabel` ist true, wenn das Schlüsselwort als bloßes Spaltenlabel in `SELECT`-Listen verwendet werden kann, und false, wenn es nur nach `AS` verwendet werden kann. `catdesc` und `baredesc` enthalten gegebenenfalls lokalisierte Beschreibungen. |
| `pg_get_partkeydef ( table oid )` -> `text` | Rekonstruiert die Definition des Partitionsschlüssels einer partitionierten Tabelle in der Form, die sie in der `PARTITION BY`-Klausel von `CREATE TABLE` hätte. |
| `pg_get_ruledef ( rule oid [, pretty boolean ] )` -> `text` | Rekonstruiert den Erstellungsbefehl für eine Regel. |
| `pg_get_serial_sequence ( table text, column text )` -> `text` | Gibt den Namen der Sequenz zurück, die einer Spalte zugeordnet ist, oder `NULL`, wenn keine Sequenz zugeordnet ist. Bei einer Identity-Spalte ist dies die intern für die Spalte erstellte Sequenz. Bei Spalten, die mit `serial`, `smallserial` oder `bigserial` erstellt wurden, ist es die Sequenz der Serial-Spaltendefinition. Im letzteren Fall kann die Zuordnung mit `ALTER SEQUENCE OWNED BY` geändert oder entfernt werden. Der erste Parameter ist ein Tabellenname mit optionalem Schema und wird nach den üblichen SQL-Regeln geparst; der zweite Parameter ist nur ein Spaltenname und wird wörtlich behandelt. Das Ergebnis ist passend formatiert, um an Sequenzfunktionen übergeben zu werden, siehe [Abschnitt 9.17](09_Funktionen_und_Operatoren.md#917-sequenzmanipulationsfunktionen). Typische Verwendung: `SELECT currval(pg_get_serial_sequence('sometable', 'id'));` |
| `pg_get_statisticsobjdef ( statobj oid )` -> `text` | Rekonstruiert den Erstellungsbefehl für ein erweitertes Statistikobjekt. |
| `pg_get_triggerdef ( trigger oid [, pretty boolean ] )` -> `text` | Rekonstruiert den Erstellungsbefehl für einen Trigger. |
| `pg_get_userbyid ( role oid )` -> `name` | Gibt zu einer Rollen-OID den Rollennamen zurück. |
| `pg_get_viewdef ( view oid [, pretty boolean ] )` -> `text` | Rekonstruiert den zugrunde liegenden `SELECT`-Befehl für eine View oder materialisierte View. |
| `pg_get_viewdef ( view oid, wrap_column integer )` -> `text` | Rekonstruiert den zugrunde liegenden `SELECT`-Befehl für eine View oder materialisierte View. In dieser Form ist Pretty-Printing immer aktiviert, und lange Zeilen werden nach Möglichkeit kürzer als die angegebene Spaltenzahl umbrochen. |
| `pg_get_viewdef ( view text [, pretty boolean ] )` -> `text` | Rekonstruiert den zugrunde liegenden `SELECT`-Befehl für eine View oder materialisierte View ausgehend von einem textuellen Namen statt von der OID. Diese Form ist veraltet; verwenden Sie stattdessen die OID-Variante. |
| `pg_index_column_has_property ( index regclass, column integer, property text )` -> `boolean` | Prüft, ob eine Indexspalte die benannte Eigenschaft hat. Häufige Indexspalteneigenschaften sind in Tabelle 9.77 aufgeführt. Erweiterungszugriffsmethoden können zusätzliche Eigenschaftsnamen definieren. `NULL` wird zurückgegeben, wenn der Eigenschaftsname unbekannt ist oder nicht auf das betreffende Objekt passt, oder wenn OID oder Spaltennummer kein gültiges Objekt identifizieren. |
| `pg_index_has_property ( index regclass, property text )` -> `boolean` | Prüft, ob ein Index die benannte Eigenschaft hat. Häufige Indexeigenschaften sind in Tabelle 9.78 aufgeführt. `NULL` wird zurückgegeben, wenn der Eigenschaftsname unbekannt ist oder nicht auf das betreffende Objekt passt, oder wenn die OID kein gültiges Objekt identifiziert. |
| `pg_indexam_has_property ( am oid, property text )` -> `boolean` | Prüft, ob eine Indexzugriffsmethode die benannte Eigenschaft hat. Eigenschaften von Zugriffsmethoden sind in Tabelle 9.79 aufgeführt. `NULL` wird zurückgegeben, wenn der Eigenschaftsname unbekannt ist oder nicht auf das betreffende Objekt passt, oder wenn die OID kein gültiges Objekt identifiziert. |
| `pg_options_to_table ( options_array text[] )` -> `setof record ( option_name text, option_value text )` | Gibt die Menge von Speicheroptionen zurück, die durch einen Wert aus `pg_class.reloptions` oder `pg_attribute.attoptions` dargestellt werden. |
| `pg_settings_get_flags ( guc text )` -> `text[]` | Gibt ein Array der Flags zurück, die mit dem angegebenen GUC verbunden sind, oder `NULL`, wenn er nicht existiert. Das Ergebnis ist ein leeres Array, wenn der GUC existiert, aber keine anzuzeigenden Flags hat. Nur die nützlichsten in Tabelle 9.80 aufgeführten Flags werden offengelegt. |
| `pg_tablespace_databases ( tablespace oid )` -> `setof oid` | Gibt die OIDs der Datenbanken zurück, die Objekte im angegebenen Tablespace gespeichert haben. Wenn diese Funktion Zeilen zurückgibt, ist der Tablespace nicht leer und kann nicht gelöscht werden. Um die konkreten Objekte zu ermitteln, müssen Sie sich mit den identifizierten Datenbanken verbinden und deren `pg_class`-Kataloge abfragen. |
| `pg_tablespace_location ( tablespace oid )` -> `text` | Gibt den Dateisystempfad zurück, an dem dieser Tablespace liegt. |
| `pg_typeof ( "any" )` -> `regtype` | Gibt die OID des Datentyps des übergebenen Werts zurück. Das kann bei Fehlersuche oder beim dynamischen Aufbau von SQL-Abfragen hilfreich sein. Die Funktion gibt `regtype` zurück, einen OID-Aliastyp, siehe [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen); für Vergleiche entspricht dies einer OID, wird aber als Typname angezeigt. Beispiel: `pg_typeof(33)` -> `integer`. |
| `COLLATION FOR ( "any" )` -> `text` | Gibt den Namen der Kollation des übergebenen Werts zurück. Der Wert wird bei Bedarf quotiert und schemaqualifiziert. Wenn für den Argumentausdruck keine Kollation abgeleitet wurde, wird `NULL` zurückgegeben. Wenn das Argument keinen kollatierbaren Datentyp hat, wird ein Fehler ausgelöst. Beispiele: `collation for ('foo'::text)` -> `"default"`; `collation for ('foo' COLLATE "de_DE")` -> `"de_DE"`. |
| `to_regclass ( text )` -> `regclass` | Übersetzt einen textuellen Relationsnamen in seine OID. Ein ähnliches Ergebnis erhält man durch Cast der Zeichenkette nach `regclass`, siehe [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen); diese Funktion gibt jedoch `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird. |
| `to_regcollation ( text )` -> `regcollation` | Übersetzt einen textuellen Kollationsnamen in seine OID. Gibt `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird. |
| `to_regnamespace ( text )` -> `regnamespace` | Übersetzt einen textuellen Schemanamen in seine OID. Gibt `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird. |
| `to_regoper ( text )` -> `regoper` | Übersetzt einen textuellen Operatornamen in seine OID. Gibt `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird oder mehrdeutig ist. |
| `to_regoperator ( text )` -> `regoperator` | Übersetzt einen textuellen Operatornamen mit Parametertypen in seine OID. Gibt `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird. |
| `to_regproc ( text )` -> `regproc` | Übersetzt einen textuellen Funktions- oder Prozedurnamen in seine OID. Gibt `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird oder mehrdeutig ist. |
| `to_regprocedure ( text )` -> `regprocedure` | Übersetzt einen textuellen Funktions- oder Prozedurnamen mit Argumenttypen in seine OID. Gibt `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird. |
| `to_regrole ( text )` -> `regrole` | Übersetzt einen textuellen Rollennamen in seine OID. Gibt `NULL` zurück, statt einen Fehler auszulösen, wenn der Name nicht gefunden wird. |
| `to_regtype ( text )` -> `regtype` | Parst eine Textzeichenkette, extrahiert daraus einen möglichen Typnamen und übersetzt diesen Namen in eine Typ-OID. Ein Syntaxfehler in der Zeichenkette führt zu einem Fehler; wenn die Zeichenkette jedoch ein syntaktisch gültiger Typname ist, der in den Katalogen nicht gefunden wird, ist das Ergebnis `NULL`. Ein ähnliches Ergebnis erhält man durch Cast nach `regtype`, nur dass dieser bei nicht gefundenem Namen einen Fehler auslöst. |
| `to_regtypemod ( text )` -> `integer` | Parst eine Textzeichenkette, extrahiert daraus einen möglichen Typnamen und übersetzt dessen Typmodifikator, falls vorhanden. Ein Syntaxfehler führt zu einem Fehler; wenn die Zeichenkette jedoch ein syntaktisch gültiger Typname ist, der in den Katalogen nicht gefunden wird, ist das Ergebnis `NULL`. Das Ergebnis ist -1, wenn kein Typmodifikator vorhanden ist. `to_regtypemod` kann mit `to_regtype` kombiniert werden, um passende Eingaben für `format_type` zu erzeugen und eine Zeichenkette mit einem Typnamen zu kanonisieren. Beispiel: `format_type(to_regtype('varchar(32)'), to_regtypemod('varchar(32)'))` -> `character varying(32)`. |

Die meisten Funktionen, die Datenbankobjekte rekonstruieren oder dekompilieren, haben ein optionales Flag `pretty`. Wenn es true ist, wird das Ergebnis „hübsch“ ausgegeben. Pretty-Printing unterdrückt unnötige Klammern und fügt Leerraum für bessere Lesbarkeit hinzu. Das hübsch formatierte Ergebnis ist lesbarer, aber das Standardformat wird von zukünftigen PostgreSQL-Versionen mit höherer Wahrscheinlichkeit gleich interpretiert; vermeiden Sie daher Pretty-Printed-Ausgaben für Dump-Zwecke. `false` für den Parameter `pretty` zu übergeben liefert dasselbe Ergebnis wie das Auslassen des Parameters.

**Tabelle 9.77. Indexspalteneigenschaften**

| Name | Beschreibung |
|---|---|
| `asc` | Sortiert die Spalte bei einem Vorwärtsscan aufsteigend? |
| `desc` | Sortiert die Spalte bei einem Vorwärtsscan absteigend? |
| `nulls_first` | Sortiert die Spalte bei einem Vorwärtsscan Nullwerte zuerst? |
| `nulls_last` | Sortiert die Spalte bei einem Vorwärtsscan Nullwerte zuletzt? |
| `orderable` | Besitzt die Spalte irgendeine definierte Sortierreihenfolge? |
| `distance_orderable` | Kann die Spalte nach einem Distanzoperator gescannt werden, zum Beispiel `ORDER BY col <-> constant`? |
| `returnable` | Kann der Spaltenwert durch einen Index-Only-Scan zurückgegeben werden? |
| `search_array` | Unterstützt die Spalte nativ Suchen der Form `col = ANY(array)`? |
| `search_nulls` | Unterstützt die Spalte Suchen mit `IS NULL` und `IS NOT NULL`? |

**Tabelle 9.78. Indexeigenschaften**

| Name | Beschreibung |
|---|---|
| `clusterable` | Kann der Index in einem `CLUSTER`-Befehl verwendet werden? |
| `index_scan` | Unterstützt der Index einfache Scans, also keine Bitmap-Scans? |
| `bitmap_scan` | Unterstützt der Index Bitmap-Scans? |
| `backward_scan` | Kann die Scanrichtung während des Scans geändert werden, um `FETCH BACKWARD` auf einem Cursor ohne Materialisierung zu unterstützen? |

**Tabelle 9.79. Eigenschaften von Indexzugriffsmethoden**

| Name | Beschreibung |
|---|---|
| `can_order` | Unterstützt die Zugriffsmethode `ASC`, `DESC` und verwandte Schlüsselwörter in `CREATE INDEX`? |
| `can_unique` | Unterstützt die Zugriffsmethode eindeutige Indizes? |
| `can_multi_col` | Unterstützt die Zugriffsmethode Indizes mit mehreren Spalten? |
| `can_exclude` | Unterstützt die Zugriffsmethode Ausschluss-Constraints? |
| `can_include` | Unterstützt die Zugriffsmethode die `INCLUDE`-Klausel von `CREATE INDEX`? |

**Tabelle 9.80. GUC-Flags**

| Flag | Beschreibung |
|---|---|
| `EXPLAIN` | Parameter mit diesem Flag werden in `EXPLAIN (SETTINGS)`-Befehlen einbezogen. |
| `NO_SHOW_ALL` | Parameter mit diesem Flag werden aus `SHOW ALL`-Befehlen ausgeschlossen. |
| `NO_RESET` | Parameter mit diesem Flag unterstützen keine `RESET`-Befehle. |
| `NO_RESET_ALL` | Parameter mit diesem Flag werden aus `RESET ALL`-Befehlen ausgeschlossen. |
| `NOT_IN_SAMPLE` | Parameter mit diesem Flag werden standardmäßig nicht in `postgresql.conf` aufgenommen. |
| `RUNTIME_COMPUTED` | Parameter mit diesem Flag werden zur Laufzeit berechnet. |

### 9.27.5. Objektinformations- und Adressierungsfunktionen

Tabelle 9.81 führt Funktionen auf, die mit Identifikation und Adressierung von Datenbankobjekten zusammenhängen.

**Tabelle 9.81. Objektinformations- und Adressierungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_get_acl ( classid oid, objid oid, objsubid integer )` -> `aclitem[]` | Gibt die ACL für ein Datenbankobjekt zurück, das durch Katalog-OID, Objekt-OID und Unterobjekt-ID angegeben ist. Für undefinierte Objekte gibt diese Funktion Nullwerte zurück. |
| `pg_describe_object ( classid oid, objid oid, objsubid integer )` -> `text` | Gibt eine textuelle Beschreibung eines Datenbankobjekts zurück, das durch Katalog-OID, Objekt-OID und Unterobjekt-ID identifiziert wird, etwa eine Spaltennummer innerhalb einer Tabelle. Die Unterobjekt-ID ist null, wenn ein ganzes Objekt gemeint ist. Die Beschreibung ist für Menschen gedacht und kann abhängig von der Serverkonfiguration übersetzt sein. Besonders nützlich ist dies, um die Identität eines Objekts zu bestimmen, auf das im Katalog `pg_depend` verwiesen wird. Für undefinierte Objekte gibt diese Funktion Nullwerte zurück. |
| `pg_identify_object ( classid oid, objid oid, objsubid integer )` -> `record ( type text, schema text, name text, identity text )` | Gibt eine Zeile mit genügend Informationen zurück, um das Datenbankobjekt eindeutig zu identifizieren. Diese Information ist für Maschinen gedacht und wird nie übersetzt. `type` identifiziert die Objektart; `schema` ist der Schemaname oder `NULL` bei Objekttypen ohne Schema; `name` ist der bei Bedarf quotierte Name, wenn Name und gegebenenfalls Schema zur eindeutigen Identifikation ausreichen, andernfalls `NULL`; `identity` ist die vollständige Objektidentität, deren genaues Format vom Objekttyp abhängt. Namen in dieser Darstellung werden bei Bedarf schemaqualifiziert und quotiert. Undefinierte Objekte werden mit Nullwerten identifiziert. |
| `pg_identify_object_as_address ( classid oid, objid oid, objsubid integer )` -> `record ( type text, object_names text[], object_args text[] )` | Gibt eine Zeile mit genügend Informationen zurück, um das angegebene Datenbankobjekt eindeutig zu identifizieren. Die zurückgegebene Information ist unabhängig vom aktuellen Server und kann verwendet werden, um ein gleichnamiges Objekt auf einem anderen Server zu identifizieren. `type` bezeichnet die Objektart; `object_names` und `object_args` sind Textarrays, die zusammen eine Referenz auf das Objekt bilden. Diese drei Werte können an `pg_get_object_address` übergeben werden, um die interne Adresse des Objekts zu erhalten. |
| `pg_get_object_address ( type text, object_names text[], object_args text[] )` -> `record ( classid oid, objid oid, objsubid integer )` | Gibt eine Zeile mit genügend Informationen zurück, um das durch einen Typcode sowie Objektname- und Argumentarrays angegebene Datenbankobjekt eindeutig zu identifizieren. Die zurückgegebenen Werte entsprechen denen, die in Systemkatalogen wie `pg_depend` verwendet werden; sie können an andere Systemfunktionen wie `pg_describe_object` oder `pg_identify_object` übergeben werden. `classid` ist die OID des Systemkatalogs, der das Objekt enthält, `objid` die OID des Objekts selbst und `objsubid` die Unterobjekt-ID oder null, wenn es keine gibt. Diese Funktion ist die Umkehrung von `pg_identify_object_as_address`. Undefinierte Objekte werden mit Nullwerten identifiziert. |

`pg_get_acl` ist nützlich, um die mit Datenbankobjekten verbundenen Rechte abzurufen und zu untersuchen, ohne spezifische Kataloge anzusehen. Um zum Beispiel alle gewährten Rechte für Objekte in der aktuellen Datenbank abzurufen:

```sql
SELECT
     (pg_identify_object(s.classid,s.objid,s.objsubid)).*,
     pg_catalog.pg_get_acl(s.classid,s.objid,s.objsubid) AS acl
FROM pg_catalog.pg_shdepend AS s
JOIN pg_catalog.pg_database AS d
     ON d.datname = current_database() AND
        d.oid = s.dbid
JOIN pg_catalog.pg_authid AS a
     ON a.oid = s.refobjid AND
        s.refclassid = 'pg_authid'::regclass
WHERE s.deptype = 'a';
```

```text
-[ RECORD 1 ]-----------------------------------------
type     | table
schema   | public
name     | testtab
identity | public.testtab
acl      | {postgres=arwdDxtm/postgres,foo=r/postgres}
```

### 9.27.6. Kommentarinformationsfunktionen

Die in Tabelle 9.82 gezeigten Funktionen extrahieren Kommentare, die zuvor mit dem Befehl `COMMENT` gespeichert wurden. Ein Nullwert wird zurückgegeben, wenn für die angegebenen Parameter kein Kommentar gefunden wurde.

**Tabelle 9.82. Kommentarinformationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `col_description ( table oid, column integer )` -> `text` | Gibt den Kommentar für eine Tabellenspalte zurück, die durch die OID ihrer Tabelle und ihre Spaltennummer angegeben ist. `obj_description` kann für Tabellenspalten nicht verwendet werden, weil Spalten keine eigenen OIDs haben. |
| `obj_description ( object oid, catalog name )` -> `text` | Gibt den Kommentar für ein Datenbankobjekt zurück, das durch seine OID und den Namen des enthaltenden Systemkatalogs angegeben ist. Zum Beispiel würde `obj_description(123456, 'pg_class')` den Kommentar für die Tabelle mit der OID 123456 abrufen. |
| `obj_description ( object oid )` -> `text` | Gibt den Kommentar für ein Datenbankobjekt zurück, das nur durch seine OID angegeben ist. Diese Form ist veraltet, weil nicht garantiert ist, dass OIDs über verschiedene Systemkataloge hinweg eindeutig sind; daher könnte der falsche Kommentar zurückgegeben werden. |
| `shobj_description ( object oid, catalog name )` -> `text` | Gibt den Kommentar für ein gemeinsam genutztes Datenbankobjekt zurück, das durch seine OID und den Namen des enthaltenden Systemkatalogs angegeben ist. Dies entspricht `obj_description`, wird aber für Kommentare zu gemeinsam genutzten Objekten verwendet, also Datenbanken, Rollen und Tablespaces. Einige Systemkataloge sind clusterweit global, und Beschreibungen für Objekte darin werden ebenfalls global gespeichert. |

### 9.27.7. Funktionen zur Gültigkeitsprüfung von Daten

Die in Tabelle 9.83 gezeigten Funktionen können hilfreich sein, um die Gültigkeit vorgeschlagener Eingabedaten zu prüfen.

**Tabelle 9.83. Funktionen zur Gültigkeitsprüfung von Daten**

| Funktion | Beschreibung | Beispiele |
|---|---|---|
| `pg_input_is_valid ( string text, type text )` -> `boolean` | Prüft, ob die angegebene Zeichenkette eine gültige Eingabe für den angegebenen Datentyp ist, und gibt true oder false zurück. Diese Funktion arbeitet nur wie gewünscht, wenn die Eingabefunktion des Datentyps so aktualisiert wurde, dass sie ungültige Eingaben als „weichen“ Fehler meldet. Andernfalls bricht eine ungültige Eingabe die Transaktion ab, genau wie bei einem direkten Cast der Zeichenkette auf den Typ. | `pg_input_is_valid('42', 'integer')` -> `t`<br>`pg_input_is_valid('42000000000', 'integer')` -> `f`<br>`pg_input_is_valid('1234.567', 'numeric(7,4)')` -> `f` |
| `pg_input_error_info ( string text, type text )` -> `record ( message text, detail text, hint text, sql_error_code text )` | Prüft, ob die angegebene Zeichenkette eine gültige Eingabe für den angegebenen Datentyp ist. Falls nicht, werden Details des Fehlers zurückgegeben, der ausgelöst worden wäre. Wenn die Eingabe gültig ist, sind die Ergebnisse `NULL`. Die Eingaben entsprechen denen von `pg_input_is_valid`. Diese Funktion arbeitet nur wie gewünscht, wenn die Eingabefunktion des Datentyps so aktualisiert wurde, dass sie ungültige Eingaben als „weichen“ Fehler meldet. Andernfalls bricht eine ungültige Eingabe die Transaktion ab, genau wie bei einem direkten Cast. | Siehe Beispiel unten. |

```sql
SELECT * FROM pg_input_error_info('42000000000', 'integer');
```

```text
                 message                         | detail | hint | sql_error_code
-------------------------------------------------+--------+------+----------------
 value "42000000000" is out of range for type integer |        |      | 22003
```

### 9.27.8. Transaktions-ID- und Snapshot-Informationsfunktionen

Die in Tabelle 9.84 gezeigten Funktionen stellen Server-Transaktionsinformationen in exportierbarer Form bereit. Die Hauptverwendung dieser Funktionen besteht darin, zu bestimmen, welche Transaktionen zwischen zwei Snapshots committet wurden.

**Tabelle 9.84. Transaktions-ID- und Snapshot-Informationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `age ( xid )` -> `integer` | Gibt die Anzahl der Transaktionen zwischen der angegebenen Transaktions-ID und dem aktuellen Transaktionszähler zurück. |
| `mxid_age ( xid )` -> `integer` | Gibt die Anzahl der Multixact-IDs zwischen der angegebenen Multixact-ID und dem aktuellen Multixact-Zähler zurück. |
| `pg_current_xact_id ()` -> `xid8` | Gibt die ID der aktuellen Transaktion zurück. Falls die aktuelle Transaktion noch keine hat, weil sie noch keine Datenbankupdates ausgeführt hat, wird eine neue zugewiesen; Details siehe Abschnitt 67.1. Wird die Funktion in einer Subtransaktion ausgeführt, wird die Transaktions-ID der obersten Ebene zurückgegeben, siehe [Abschnitt 67.3](67_Transaktionsverarbeitung.md#673-subtransaktionen). |
| `pg_current_xact_id_if_assigned ()` -> `xid8` | Gibt die ID der aktuellen Transaktion zurück oder `NULL`, wenn noch keine ID zugewiesen wurde. Diese Variante ist vorzuziehen, wenn die Transaktion sonst möglicherweise nur lesend wäre, um unnötigen XID-Verbrauch zu vermeiden. In einer Subtransaktion wird die Transaktions-ID der obersten Ebene zurückgegeben. |
| `pg_xact_status ( xid8 )` -> `text` | Meldet den Commit-Status einer jüngeren Transaktion. Das Ergebnis ist `in progress`, `committed` oder `aborted`, sofern die Transaktion jung genug ist, dass das System ihren Commit-Status noch vorhält. Ist sie so alt, dass keine Referenzen auf die Transaktion im System verbleiben und die Commit-Statusinformation verworfen wurde, ist das Ergebnis `NULL`. Anwendungen können damit zum Beispiel bestimmen, ob ihre Transaktion committet oder abgebrochen wurde, nachdem Anwendung und Datenbankserver während eines laufenden `COMMIT` getrennt wurden. Vorbereitete Transaktionen werden als `in progress` gemeldet; Anwendungen müssen `pg_prepared_xacts` prüfen, wenn sie feststellen müssen, ob eine Transaktions-ID zu einer vorbereiteten Transaktion gehört. |
| `pg_current_snapshot ()` -> `pg_snapshot` | Gibt einen aktuellen Snapshot zurück, eine Datenstruktur, die zeigt, welche Transaktions-IDs gerade laufen. Nur Transaktions-IDs der obersten Ebene sind enthalten; Subtransaktions-IDs werden nicht gezeigt, siehe [Abschnitt 67.3](67_Transaktionsverarbeitung.md#673-subtransaktionen). |
| `pg_snapshot_xip ( pg_snapshot )` -> `setof xid8` | Gibt die Menge der laufenden Transaktions-IDs zurück, die in einem Snapshot enthalten sind. |
| `pg_snapshot_xmax ( pg_snapshot )` -> `xid8` | Gibt das `xmax` eines Snapshots zurück. |
| `pg_snapshot_xmin ( pg_snapshot )` -> `xid8` | Gibt das `xmin` eines Snapshots zurück. |
| `pg_visible_in_snapshot ( xid8, pg_snapshot )` -> `boolean` | Ist die angegebene Transaktions-ID gemäß diesem Snapshot sichtbar, wurde sie also vor Erstellung des Snapshots abgeschlossen? Diese Funktion liefert für eine Subtransaktions-ID (`subxid`) keine korrekte Antwort; siehe [Abschnitt 67.3](67_Transaktionsverarbeitung.md#673-subtransaktionen). |
| `pg_get_multixact_members ( multixid xid )` -> `setof record ( xid xid, mode text )` | Gibt Transaktions-ID und Sperrmodus für jedes Mitglied der angegebenen Multixact-ID zurück. Die Sperrmodi `forupd`, `fornokeyupd`, `sh` und `keysh` entsprechen den Zeilensperren `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` beziehungsweise `FOR KEY SHARE`, wie in Abschnitt 13.3.2 beschrieben. Zwei zusätzliche Modi sind Multixact-spezifisch: `nokeyupd` für Updates, die keine Schlüsselspalten ändern, und `upd` für Updates oder Deletes, die Schlüsselspalten ändern. |

Der interne Transaktions-ID-Typ `xid` ist 32 Bit breit und läuft alle 4 Milliarden Transaktionen über. Die in Tabelle 9.84 gezeigten Funktionen verwenden jedoch, mit Ausnahme von `age`, `mxid_age` und `pg_get_multixact_members`, den 64-Bit-Typ `xid8`, der während der Lebensdauer einer Installation nicht überläuft und bei Bedarf durch Cast in `xid` umgewandelt werden kann; siehe [Abschnitt 67.1](67_Transaktionsverarbeitung.md#671-transaktionen-und-identifikatoren). Der Datentyp `pg_snapshot` speichert Informationen über die Sichtbarkeit von Transaktions-IDs zu einem bestimmten Zeitpunkt. Seine Bestandteile werden in Tabelle 9.85 beschrieben. Die textuelle Darstellung von `pg_snapshot` lautet `xmin:xmax:xip_list`. Beispielsweise bedeutet `10:20:10,14,15`: `xmin=10`, `xmax=20`, `xip_list=10, 14, 15`.

**Tabelle 9.85. Snapshot-Bestandteile**

| Name | Beschreibung |
|---|---|
| `xmin` | Niedrigste Transaktions-ID, die noch aktiv war. Alle Transaktions-IDs kleiner als `xmin` sind entweder committet und sichtbar oder zurückgerollt und tot. |
| `xmax` | Eins größer als die höchste abgeschlossene Transaktions-ID. Alle Transaktions-IDs größer oder gleich `xmax` waren zum Zeitpunkt des Snapshots noch nicht abgeschlossen und sind daher unsichtbar. |
| `xip_list` | Transaktionen, die zum Zeitpunkt des Snapshots liefen. Eine Transaktions-ID mit `xmin <= X < xmax`, die nicht in dieser Liste steht, war zum Zeitpunkt des Snapshots bereits abgeschlossen und ist daher je nach Commit-Status entweder sichtbar oder tot. Diese Liste enthält keine Transaktions-IDs von Subtransaktionen (`subxids`). |

In PostgreSQL-Versionen vor 13 gab es keinen Typ `xid8`, daher wurden Varianten dieser Funktionen bereitgestellt, die `bigint` zur Darstellung einer 64-Bit-XID verwendeten, zusammen mit einem entsprechend eigenen Snapshot-Datentyp `txid_snapshot`. Diese älteren Funktionen tragen `txid` im Namen. Sie werden aus Gründen der Abwärtskompatibilität noch unterstützt, könnten aber in einer zukünftigen Version entfernt werden. Siehe Tabelle 9.86.

**Tabelle 9.86. Veraltete Transaktions-ID- und Snapshot-Informationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `txid_current ()` -> `bigint` | Siehe `pg_current_xact_id()`. |
| `txid_current_if_assigned ()` -> `bigint` | Siehe `pg_current_xact_id_if_assigned()`. |
| `txid_current_snapshot ()` -> `txid_snapshot` | Siehe `pg_current_snapshot()`. |
| `txid_snapshot_xip ( txid_snapshot )` -> `setof bigint` | Siehe `pg_snapshot_xip()`. |
| `txid_snapshot_xmax ( txid_snapshot )` -> `bigint` | Siehe `pg_snapshot_xmax()`. |
| `txid_snapshot_xmin ( txid_snapshot )` -> `bigint` | Siehe `pg_snapshot_xmin()`. |
| `txid_visible_in_snapshot ( bigint, txid_snapshot )` -> `boolean` | Siehe `pg_visible_in_snapshot()`. |
| `txid_status ( bigint )` -> `text` | Siehe `pg_xact_status()`. |

### 9.27.9. Funktionen zu committeten Transaktionen

Die in Tabelle 9.87 gezeigten Funktionen liefern Informationen darüber, wann vergangene Transaktionen committet wurden. Sie liefern nur nützliche Daten, wenn die Konfigurationsoption `track_commit_timestamp` aktiviert ist, und nur für Transaktionen, die nach ihrer Aktivierung committet wurden. Commit-Zeitstempelinformationen werden routinemäßig während `VACUUM` entfernt.

**Tabelle 9.87. Funktionen zu committeten Transaktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_xact_commit_timestamp ( xid )` -> `timestamp with time zone` | Gibt den Commit-Zeitstempel einer Transaktion zurück. |
| `pg_xact_commit_timestamp_origin ( xid )` -> `record ( timestamp timestamp with time zone, roident oid )` | Gibt Commit-Zeitstempel und Replikationsursprung einer Transaktion zurück. |
| `pg_last_committed_xact ()` -> `record ( xid xid, timestamp timestamp with time zone, roident oid )` | Gibt Transaktions-ID, Commit-Zeitstempel und Replikationsursprung der zuletzt committeten Transaktion zurück. |

### 9.27.10. Kontrolldatenfunktionen

Die in Tabelle 9.88 gezeigten Funktionen geben Informationen aus, die während `initdb` initialisiert wurden, etwa die Katalogversion. Sie zeigen außerdem Informationen über Write-Ahead Logging und Checkpoint-Verarbeitung. Diese Informationen gelten clusterweit und nicht nur für eine einzelne Datenbank. Die Funktionen liefern größtenteils dieselben Informationen aus derselben Quelle wie das Programm `pg_controldata`.

**Tabelle 9.88. Kontrolldatenfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_control_checkpoint ()` -> `record` | Gibt Informationen über den aktuellen Checkpoint-Zustand zurück, wie in Tabelle 9.89 gezeigt. |
| `pg_control_system ()` -> `record` | Gibt Informationen über den aktuellen Zustand der Kontrolldatei zurück, wie in Tabelle 9.90 gezeigt. |
| `pg_control_init ()` -> `record` | Gibt Informationen über den Initialisierungszustand des Clusters zurück, wie in Tabelle 9.91 gezeigt. |
| `pg_control_recovery ()` -> `record` | Gibt Informationen über den Recovery-Zustand zurück, wie in Tabelle 9.92 gezeigt. |

**Tabelle 9.89. Ausgabespalten von `pg_control_checkpoint`**

| Spaltenname | Datentyp |
|---|---|
| `checkpoint_lsn` | `pg_lsn` |
| `redo_lsn` | `pg_lsn` |
| `redo_wal_file` | `text` |
| `timeline_id` | `integer` |
| `prev_timeline_id` | `integer` |
| `full_page_writes` | `boolean` |
| `next_xid` | `text` |
| `next_oid` | `oid` |
| `next_multixact_id` | `xid` |
| `next_multi_offset` | `xid` |
| `oldest_xid` | `xid` |
| `oldest_xid_dbid` | `oid` |
| `oldest_active_xid` | `xid` |
| `oldest_multi_xid` | `xid` |
| `oldest_multi_dbid` | `oid` |
| `oldest_commit_ts_xid` | `xid` |
| `newest_commit_ts_xid` | `xid` |
| `checkpoint_time` | `timestamp with time zone` |

**Tabelle 9.90. Ausgabespalten von `pg_control_system`**

| Spaltenname | Datentyp |
|---|---|
| `pg_control_version` | `integer` |
| `catalog_version_no` | `integer` |
| `system_identifier` | `bigint` |
| `pg_control_last_modified` | `timestamp with time zone` |

**Tabelle 9.91. Ausgabespalten von `pg_control_init`**

| Spaltenname | Datentyp |
|---|---|
| `max_data_alignment` | `integer` |
| `database_block_size` | `integer` |
| `blocks_per_segment` | `integer` |
| `wal_block_size` | `integer` |
| `bytes_per_wal_segment` | `integer` |
| `max_identifier_length` | `integer` |
| `max_index_columns` | `integer` |
| `max_toast_chunk_size` | `integer` |
| `large_object_chunk_size` | `integer` |
| `float8_pass_by_value` | `boolean` |
| `data_page_checksum_version` | `integer` |
| `default_char_signedness` | `boolean` |

**Tabelle 9.92. Ausgabespalten von `pg_control_recovery`**

| Spaltenname | Datentyp |
|---|---|
| `min_recovery_end_lsn` | `pg_lsn` |
| `min_recovery_end_timeline` | `integer` |
| `backup_start_lsn` | `pg_lsn` |
| `backup_end_lsn` | `pg_lsn` |
| `end_of_backup_record_required` | `boolean` |

### 9.27.11. Versionsinformationsfunktionen

Die in Tabelle 9.93 gezeigten Funktionen geben Versionsinformationen aus.

**Tabelle 9.93. Versionsinformationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `version ()` -> `text` | Gibt eine Zeichenkette zurück, die die Version des PostgreSQL-Servers beschreibt. Diese Information erhalten Sie auch aus `server_version`; für eine maschinenlesbare Version verwenden Sie `server_version_num`. Softwareentwickler sollten `server_version_num`, verfügbar seit 8.2, oder `PQserverVersion` verwenden, statt den Versionstext zu parsen. |
| `unicode_version ()` -> `text` | Gibt eine Zeichenkette zurück, die die von PostgreSQL verwendete Unicode-Version darstellt. |
| `icu_unicode_version ()` -> `text` | Gibt eine Zeichenkette zurück, die die von ICU verwendete Unicode-Version darstellt, wenn der Server mit ICU-Unterstützung gebaut wurde; andernfalls wird `NULL` zurückgegeben. |

### 9.27.12. WAL-Zusammenfassungs-Informationsfunktionen

Die in Tabelle 9.94 gezeigten Funktionen geben Informationen über den Status der WAL-Zusammenfassung aus. Siehe `summarize_wal`.

**Tabelle 9.94. WAL-Zusammenfassungs-Informationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_available_wal_summaries ()` -> `setof record ( tli bigint, start_lsn pg_lsn, end_lsn pg_lsn )` | Gibt Informationen über WAL-Zusammenfassungsdateien zurück, die im Datenverzeichnis unter `pg_wal/summaries` vorhanden sind. Pro WAL-Zusammenfassungsdatei wird eine Zeile zurückgegeben. Jede Datei fasst WAL auf der angegebenen TLI innerhalb des angegebenen LSN-Bereichs zusammen. Diese Funktion kann nützlich sein, um zu bestimmen, ob auf dem Server genügend WAL-Zusammenfassungen vorhanden sind, um ein inkrementelles Backup auf Basis eines früheren Backups mit bekannter Start-LSN zu erstellen. |
| `pg_wal_summary_contents ( tli bigint, start_lsn pg_lsn, end_lsn pg_lsn )` -> `setof record ( relfilenode oid, reltablespace oid, reldatabase oid, relforknumber smallint, relblocknumber bigint, is_limit_block boolean )` | Gibt Informationen über den Inhalt einer einzelnen WAL-Zusammenfassungsdatei zurück, die durch TLI sowie Start- und End-LSN identifiziert wird. Jede Zeile mit `is_limit_block` false bedeutet, dass der durch die übrigen Ausgabespalten identifizierte Block durch mindestens einen WAL-Eintrag im zusammengefassten Bereich geändert wurde. Jede Zeile mit `is_limit_block` true bedeutet entweder, dass der Relation-Fork innerhalb des relevanten WAL-Bereichs auf die durch `relblocknumber` angegebene Länge gekürzt wurde, oder dass der Relation-Fork innerhalb dieses Bereichs erstellt oder gelöscht wurde. In solchen Fällen ist `relblocknumber` null. |
| `pg_get_wal_summarizer_state ()` -> `record ( summarized_tli bigint, summarized_lsn pg_lsn, pending_lsn pg_lsn, summarizer_pid int )` | Gibt Informationen über den Fortschritt des WAL-Summarizers zurück. Wenn der WAL-Summarizer seit dem Start der Instanz noch nie gelaufen ist, sind `summarized_tli` und `summarized_lsn` 0 beziehungsweise `0/0`; andernfalls sind es TLI und End-LSN der letzten auf Platte geschriebenen WAL-Zusammenfassungsdatei. Wenn der WAL-Summarizer gerade läuft, ist `pending_lsn` die End-LSN des letzten von ihm konsumierten Eintrags und muss immer größer oder gleich `summarized_lsn` sein; wenn er nicht läuft, ist sie gleich `summarized_lsn`. `summarizer_pid` ist die PID des WAL-Summarizer-Prozesses, wenn er läuft, andernfalls `NULL`. Als besondere Ausnahme weigert sich der WAL-Summarizer, WAL-Zusammenfassungsdateien zu erzeugen, wenn er auf WAL läuft, das unter `wal_level=minimal` erzeugt wurde, weil solche Zusammenfassungen als Basis für ein inkrementelles Backup unsicher wären. In diesem Fall schreiten die oben genannten Felder weiter voran, als würden Zusammenfassungen erzeugt, aber es wird nichts auf Platte geschrieben. Sobald der Summarizer WAL erreicht, das erzeugt wurde, während `wal_level` auf `replica` oder höher gesetzt war, setzt er das Schreiben von Zusammenfassungen auf Platte fort. |

## 9.28. Systemadministrationsfunktionen

Die in diesem Abschnitt beschriebenen Funktionen werden verwendet, um eine PostgreSQL-Installation zu steuern und zu überwachen.

### 9.28.1. Funktionen für Konfigurationseinstellungen

Tabelle 9.95 zeigt die Funktionen, die zum Abfragen und Ändern von Laufzeit-Konfigurationsparametern verfügbar sind.

**Tabelle 9.95. Funktionen für Konfigurationseinstellungen**

| Funktion | Beschreibung | Beispiele |
|---|---|---|
| `current_setting ( setting_name text [, missing_ok boolean ] )` -> `text` | Gibt den aktuellen Wert der Einstellung `setting_name` zurück. Wenn es keine solche Einstellung gibt, löst `current_setting` einen Fehler aus, außer `missing_ok` ist angegeben und true; in diesem Fall wird `NULL` zurückgegeben. Diese Funktion entspricht dem SQL-Befehl `SHOW`. | `current_setting('datestyle')` -> `ISO, MDY` |
| `set_config ( setting_name text, new_value text, is_local boolean )` -> `text` | Setzt den Parameter `setting_name` auf `new_value` und gibt diesen Wert zurück. Wenn `is_local` true ist, gilt der neue Wert nur während der aktuellen Transaktion. Soll der neue Wert für den Rest der aktuellen Sitzung gelten, verwenden Sie stattdessen false. Diese Funktion entspricht dem SQL-Befehl `SET`. `set_config` akzeptiert `NULL` für `new_value`; da Einstellungen nicht null sein können, wird dies als Aufforderung interpretiert, die Einstellung auf ihren Standardwert zurückzusetzen. | `set_config('log_statement_stats', 'off', false)` -> `off` |

### 9.28.2. Server-Signalisierungsfunktionen

Die in Tabelle 9.96 gezeigten Funktionen senden Steuersignale an andere Serverprozesse. Standardmäßig ist ihre Verwendung auf Superuser beschränkt, der Zugriff kann aber mit `GRANT` anderen Benutzern gewährt werden, soweit nicht anders vermerkt.

Jede dieser Funktionen gibt true zurück, wenn das Signal erfolgreich gesendet wurde, und false, wenn das Senden fehlgeschlagen ist.

**Tabelle 9.96. Server-Signalisierungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_cancel_backend ( pid integer )` -> `boolean` | Bricht die aktuelle Abfrage der Sitzung ab, deren Backendprozess die angegebene Prozess-ID hat. Dies ist auch erlaubt, wenn die aufrufende Rolle Mitglied der Rolle ist, deren Backend abgebrochen wird, oder wenn sie Rechte von `pg_signal_backend` besitzt; nur Superuser können jedoch Superuser-Backends abbrechen. Als Ausnahme dürfen Rollen mit Rechten von `pg_signal_autovacuum_worker` Autovacuum-Workerprozesse abbrechen, die sonst als Superuser-Backends gelten. |
| `pg_log_backend_memory_contexts ( pid integer )` -> `boolean` | Fordert an, die Speicherkontexte des Backends mit der angegebenen Prozess-ID zu protokollieren. Diese Funktion kann die Anforderung an Backends und Hilfsprozesse außer dem Logger senden. Die Speicherkontexte werden auf Log-Level `LOG` protokolliert. Sie erscheinen abhängig von der Logkonfiguration im Serverlog, siehe [Abschnitt 19.8](19_Serverkonfiguration.md#198-fehlerberichte-und-logging), werden aber unabhängig von `client_min_messages` nicht an den Client gesendet. |
| `pg_reload_conf ()` -> `boolean` | Veranlasst alle Prozesse des PostgreSQL-Servers, ihre Konfigurationsdateien neu zu laden. Dazu wird ein `SIGHUP`-Signal an den Postmasterprozess gesendet, der wiederum `SIGHUP` an seine Kindprozesse sendet. Sie können die Sichten `pg_file_settings`, `pg_hba_file_rules` und `pg_ident_file_mappings` verwenden, um die Konfigurationsdateien vor dem Neuladen auf mögliche Fehler zu prüfen. |
| `pg_rotate_logfile ()` -> `boolean` | Signalisiert dem Logdateimanager, sofort auf eine neue Ausgabedatei umzuschalten. Dies funktioniert nur, wenn der eingebaute Log-Collector läuft, da es sonst keinen Logdateimanager-Unterprozess gibt. |
| `pg_terminate_backend ( pid integer, timeout bigint DEFAULT 0 )` -> `boolean` | Beendet die Sitzung, deren Backendprozess die angegebene Prozess-ID hat. Dies ist auch erlaubt, wenn die aufrufende Rolle Mitglied der Rolle ist, deren Backend beendet wird, oder wenn sie Rechte von `pg_signal_backend` besitzt; nur Superuser können jedoch Superuser-Backends beenden. Als Ausnahme dürfen Rollen mit Rechten von `pg_signal_autovacuum_worker` Autovacuum-Workerprozesse beenden. Wenn `timeout` nicht angegeben oder null ist, gibt diese Funktion true zurück, unabhängig davon, ob der Prozess tatsächlich beendet wird; sie zeigt nur an, dass das Signal erfolgreich gesendet wurde. Ist `timeout` in Millisekunden angegeben und größer als null, wartet die Funktion, bis der Prozess tatsächlich beendet ist oder die Zeit abgelaufen ist. Bei erfolgreicher Beendigung gibt sie true zurück; bei Timeout wird eine Warnung ausgegeben und false zurückgegeben. |

`pg_cancel_backend` und `pg_terminate_backend` senden Signale, `SIGINT` beziehungsweise `SIGTERM`, an Backendprozesse, die durch Prozess-ID identifiziert werden. Die Prozess-ID eines aktiven Backends finden Sie in der Spalte `pid` der Sicht `pg_stat_activity` oder durch Auflisten der `postgres`-Prozesse auf dem Server, unter Unix mit `ps`, unter Windows mit dem Task-Manager. Die Rolle eines aktiven Backends finden Sie in der Spalte `usename` von `pg_stat_activity`.

`pg_log_backend_memory_contexts` kann verwendet werden, um die Speicherkontexte eines Backendprozesses zu protokollieren. Zum Beispiel:

```sql
SELECT pg_log_backend_memory_contexts(pg_backend_pid());
```

```text
 pg_log_backend_memory_contexts
--------------------------------
 t
(1 row)
```

Für jeden Speicherkontext wird eine Nachricht protokolliert. Zum Beispiel:

```text
LOG: logging memory contexts of PID 10377
STATEMENT: SELECT pg_log_backend_memory_contexts(pg_backend_pid());
LOG: level: 1; TopMemoryContext: 80800 total in 6 blocks; 14432 free (5 chunks); 66368 used
LOG: level: 2; pgstat TabStatusArray lookup hash table: 8192 total in 1 blocks; 1408 free (0 chunks); 6784 used
LOG: level: 2; TopTransactionContext: 8192 total in 1 blocks; 7720 free (1 chunks); 472 used
LOG: level: 2; RowDescriptionContext: 8192 total in 1 blocks; 6880 free (0 chunks); 1312 used
LOG: level: 2; MessageContext: 16384 total in 2 blocks; 5152 free (0 chunks); 11232 used
LOG: level: 2; Operator class cache: 8192 total in 1 blocks; 512 free (0 chunks); 7680 used
LOG: level: 2; smgr relation table: 16384 total in 2 blocks; 4544 free (3 chunks); 11840 used
LOG: level: 2; TransactionAbortContext: 32768 total in 1 blocks; 32504 free (0 chunks); 264 used
...
LOG: level: 2; ErrorContext: 8192 total in 1 blocks; 7928 free (3 chunks); 264 used
LOG: Grand total: 1651920 bytes in 201 blocks; 622360 free (88 chunks); 1029560 used
```

Wenn es unter demselben Elternkontext mehr als 100 Kindkontexte gibt, werden die ersten 100 Kindkontexte protokolliert, zusammen mit einer Zusammenfassung der verbleibenden Kontexte. Häufige Aufrufe dieser Funktion können erheblichen Overhead verursachen, weil sie eine große Zahl von Logmeldungen erzeugen kann.

### 9.28.3. Backup-Steuerungsfunktionen

Die in Tabelle 9.97 gezeigten Funktionen unterstützen Online-Backups. Diese Funktionen können während der Recovery nicht ausgeführt werden, mit Ausnahme von `pg_backup_start`, `pg_backup_stop` und `pg_wal_lsn_diff`.

Details zur korrekten Verwendung dieser Funktionen finden Sie in [Abschnitt 25.3](25_Sicherung_und_Wiederherstellung.md#253-kontinuierliche-archivierung-und-pointintimerecovery-pitr).

**Tabelle 9.97. Backup-Steuerungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_create_restore_point ( name text )` -> `pg_lsn` | Erstellt einen benannten Markerdatensatz im Write-Ahead Log, der später als Recovery-Ziel verwendet werden kann, und gibt die entsprechende WAL-Position zurück. Der angegebene Name kann anschließend mit `recovery_target_name` verwendet werden, um den Punkt anzugeben, bis zu dem Recovery laufen soll. Vermeiden Sie mehrere Restore Points mit demselben Namen, da Recovery beim ersten passenden Namen anhält. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_current_wal_flush_lsn ()` -> `pg_lsn` | Gibt die aktuelle Flush-Position des Write-Ahead Logs zurück, siehe Hinweise unten. |
| `pg_current_wal_insert_lsn ()` -> `pg_lsn` | Gibt die aktuelle Insert-Position des Write-Ahead Logs zurück, siehe Hinweise unten. |
| `pg_current_wal_lsn ()` -> `pg_lsn` | Gibt die aktuelle Schreibposition des Write-Ahead Logs zurück, siehe Hinweise unten. |
| `pg_backup_start ( label text [, fast boolean ] )` -> `pg_lsn` | Bereitet den Server darauf vor, ein Online-Backup zu beginnen. Der einzige Pflichtparameter ist ein beliebiges benutzerdefiniertes Label für das Backup, typischerweise der Name, unter dem die Backup-Dump-Datei gespeichert wird. Wenn der optionale zweite Parameter true ist, wird `pg_backup_start` so schnell wie möglich ausgeführt. Dadurch wird ein sofortiger Checkpoint erzwungen, was zu einer Spitze bei I/O-Operationen führt und gleichzeitig laufende Abfragen verlangsamen kann. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_backup_stop ( [wait_for_archive boolean ] )` -> `record ( lsn pg_lsn, labelfile text, spcmapfile text )` | Beendet ein Online-Backup. Die gewünschten Inhalte der Backup-Label-Datei und der Tablespace-Map-Datei werden als Teil des Funktionsergebnisses zurückgegeben und müssen in Dateien im Backupbereich geschrieben werden. Diese Dateien dürfen nicht in das Live-Datenverzeichnis geschrieben werden, da PostgreSQL sonst nach einem Absturz möglicherweise nicht neu startet. Wenn der optionale boolesche Parameter false ist, kehrt die Funktion unmittelbar nach Abschluss des Backups zurück, ohne auf die Archivierung von WAL zu warten. Das ist nur mit Backupsoftware sinnvoll, die die WAL-Archivierung unabhängig überwacht. Standardmäßig oder bei true wartet `pg_backup_stop` bei aktivierter Archivierung auf WAL-Archivierung. Auf einem Standby wartet sie nur, wenn `archive_mode = always` ist. Auf einem Primary erzeugt die Funktion außerdem eine Backup-History-Datei im WAL-Archivbereich. Das Ergebnis enthält in `lsn` die Endposition des Backups, in der zweiten Spalte den Inhalt der Backup-Label-Datei und in der dritten Spalte den Inhalt der Tablespace-Map-Datei. Diese müssen als Teil des Backups gespeichert werden und sind für die Wiederherstellung erforderlich. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_switch_wal ()` -> `pg_lsn` | Erzwingt, dass der Server auf eine neue WAL-Datei umschaltet, sodass die aktuelle Datei archiviert werden kann, sofern Continuous Archiving verwendet wird. Das Ergebnis ist die Endposition plus 1 innerhalb der gerade abgeschlossenen WAL-Datei. Wenn seit dem letzten WAL-Wechsel keine WAL-Aktivität stattgefunden hat, tut `pg_switch_wal` nichts und gibt die Startposition der aktuell verwendeten WAL-Datei zurück. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_walfile_name ( lsn pg_lsn )` -> `text` | Wandelt eine WAL-Position in den Namen der WAL-Datei um, die diese Position enthält. |
| `pg_walfile_name_offset ( lsn pg_lsn )` -> `record ( file_name text, file_offset integer )` | Wandelt eine WAL-Position in einen WAL-Dateinamen und einen Byteoffset innerhalb dieser Datei um. |
| `pg_split_walfile_name ( file_name text )` -> `record ( segment_number numeric, timeline_id bigint )` | Extrahiert die Sequenznummer und die Timeline-ID aus einem WAL-Dateinamen. |
| `pg_wal_lsn_diff ( lsn1 pg_lsn, lsn2 pg_lsn )` -> `numeric` | Berechnet die Differenz in Bytes, `lsn1 - lsn2`, zwischen zwei WAL-Positionen. Dies kann mit `pg_stat_replication` oder einigen der in Tabelle 9.97 gezeigten Funktionen verwendet werden, um Replikationsverzögerung zu ermitteln. |

`pg_current_wal_lsn` zeigt die aktuelle Schreibposition des Write-Ahead Logs im selben Format wie die obigen Funktionen. Entsprechend zeigt `pg_current_wal_insert_lsn` die aktuelle Insert-Position und `pg_current_wal_flush_lsn` die aktuelle Flush-Position. Die Insert-Position ist zu jedem Zeitpunkt das „logische“ Ende des WAL; die Schreibposition ist das Ende dessen, was tatsächlich aus den internen Serverpuffern herausgeschrieben wurde; die Flush-Position ist die letzte Position, von der bekannt ist, dass sie dauerhaft gespeichert wurde. Die Schreibposition ist das Ende dessen, was von außerhalb des Servers untersucht werden kann, und ist normalerweise das, was Sie bei der Archivierung teilweise vollständiger WAL-Dateien benötigen. Insert- und Flush-Positionen dienen hauptsächlich Debuggingzwecken des Servers. Diese Operationen sind alle nur lesend und benötigen keine Superuser-Rechte.

Sie können `pg_walfile_name_offset` verwenden, um den zugehörigen WAL-Dateinamen und Byteoffset aus einem `pg_lsn`-Wert zu extrahieren. Zum Beispiel:

```sql
SELECT * FROM pg_walfile_name_offset((pg_backup_stop()).lsn);
```

```text
        file_name         | file_offset
--------------------------+-------------
 00000001000000000000000D |     4039624
(1 row)
```

Entsprechend extrahiert `pg_walfile_name` nur den WAL-Dateinamen.

`pg_split_walfile_name` ist nützlich, um aus Dateioffset und WAL-Dateiname eine LSN zu berechnen, zum Beispiel:

```sql
\set file_name '000000010000000100C000AB'
\set offset 256
SELECT '0/0'::pg_lsn + pd.segment_number * ps.setting::int + :offset AS lsn
FROM pg_split_walfile_name(:'file_name') pd,
     pg_show_all_settings() ps
WHERE ps.name = 'wal_segment_size';
```

```text
      lsn
---------------
 C001/AB000100
(1 row)
```

### 9.28.4. Recovery-Steuerungsfunktionen

Die in Tabelle 9.98 gezeigten Funktionen liefern Informationen über den aktuellen Status eines Standby-Servers. Diese Funktionen können sowohl während der Recovery als auch im normalen Betrieb ausgeführt werden.

**Tabelle 9.98. Recovery-Informationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_is_in_recovery ()` -> `boolean` | Gibt true zurück, wenn Recovery noch läuft. |
| `pg_last_wal_receive_lsn ()` -> `pg_lsn` | Gibt die letzte WAL-Position zurück, die durch Streaming-Replikation empfangen und auf Platte synchronisiert wurde. Während Streaming-Replikation läuft, steigt dieser Wert monoton. Wenn Recovery abgeschlossen ist, bleibt er an der Position des letzten während der Recovery empfangenen und synchronisierten WAL-Eintrags stehen. Wenn Streaming-Replikation deaktiviert ist oder noch nicht begonnen hat, gibt die Funktion `NULL` zurück. |
| `pg_last_wal_replay_lsn ()` -> `pg_lsn` | Gibt die letzte WAL-Position zurück, die während der Recovery wiedergegeben wurde. Wenn Recovery noch läuft, steigt dieser Wert monoton. Wenn Recovery abgeschlossen ist, bleibt er an der Position des letzten während der Recovery angewendeten WAL-Eintrags stehen. Wurde der Server normal ohne Recovery gestartet, gibt die Funktion `NULL` zurück. |
| `pg_last_xact_replay_timestamp ()` -> `timestamp with time zone` | Gibt den Zeitstempel der letzten während der Recovery wiedergegebenen Transaktion zurück. Dies ist der Zeitpunkt, zu dem der Commit- oder Abort-WAL-Eintrag dieser Transaktion auf dem Primary erzeugt wurde. Wenn während der Recovery keine Transaktionen wiedergegeben wurden, gibt die Funktion `NULL` zurück. Läuft Recovery noch, steigt der Wert monoton; nach abgeschlossener Recovery bleibt er beim Zeitpunkt der letzten angewendeten Transaktion stehen. Bei normalem Start ohne Recovery wird `NULL` zurückgegeben. |
| `pg_get_wal_resource_managers ()` -> `setof record ( rm_id integer, rm_name text, rm_builtin boolean )` | Gibt die derzeit im System geladenen WAL-Resource-Manager zurück. Die Spalte `rm_builtin` zeigt an, ob es sich um einen eingebauten Resource-Manager oder um einen von einer Erweiterung geladenen benutzerdefinierten Resource-Manager handelt. |

Die in Tabelle 9.99 gezeigten Funktionen steuern den Fortschritt der Recovery. Diese Funktionen können nur während der Recovery ausgeführt werden.

**Tabelle 9.99. Recovery-Steuerungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_is_wal_replay_paused ()` -> `boolean` | Gibt true zurück, wenn eine Recovery-Pause angefordert wurde. |
| `pg_get_wal_replay_pause_state ()` -> `text` | Gibt den Zustand der Recovery-Pause zurück. Die Rückgabewerte sind `not paused`, wenn keine Pause angefordert wurde, `pause requested`, wenn eine Pause angefordert wurde, die Recovery aber noch nicht pausiert ist, und `paused`, wenn die Recovery tatsächlich pausiert ist. |
| `pg_promote ( wait boolean DEFAULT true, wait_seconds integer DEFAULT 60 )` -> `boolean` | Befördert einen Standby-Server zum Primary. Wenn `wait` true ist, der Standard, wartet die Funktion, bis die Beförderung abgeschlossen ist oder `wait_seconds` Sekunden vergangen sind, und gibt true zurück, wenn die Beförderung erfolgreich war, andernfalls false. Wenn `wait` false ist, gibt die Funktion unmittelbar nach dem Senden eines `SIGUSR1`-Signals an den Postmaster zur Auslösung der Beförderung true zurück. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_wal_replay_pause ()` -> `void` | Fordert an, Recovery zu pausieren. Eine Anforderung bedeutet nicht, dass Recovery sofort stoppt. Wenn Sie sicherstellen möchten, dass Recovery tatsächlich pausiert ist, prüfen Sie den von `pg_get_wal_replay_pause_state()` zurückgegebenen Zustand. `pg_is_wal_replay_paused()` zeigt nur an, ob eine Anforderung gestellt wurde. Während Recovery pausiert ist, werden keine weiteren Datenbankänderungen angewendet. Wenn Hot Standby aktiv ist, sehen alle neuen Abfragen denselben konsistenten Snapshot der Datenbank, und bis zur Fortsetzung der Recovery entstehen keine weiteren Abfragekonflikte. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_wal_replay_resume ()` -> `void` | Setzt Recovery fort, wenn sie pausiert war. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |

`pg_wal_replay_pause` und `pg_wal_replay_resume` können nicht ausgeführt werden, während eine Beförderung läuft. Wenn eine Beförderung ausgelöst wird, während Recovery pausiert ist, endet der Pausenzustand und die Beförderung läuft weiter.

Wenn Streaming-Replikation deaktiviert ist, kann der Pausenzustand unbegrenzt ohne Problem bestehen bleiben. Läuft Streaming-Replikation, werden weiterhin WAL-Einträge empfangen; dies kann abhängig von Pausendauer, WAL-Erzeugungsrate und verfügbarem Plattenplatz irgendwann den verfügbaren Speicher füllen.

### 9.28.5. Snapshot-Synchronisationsfunktionen

PostgreSQL erlaubt Datenbanksitzungen, ihre Snapshots zu synchronisieren. Ein Snapshot bestimmt, welche Daten für die Transaktion sichtbar sind, die den Snapshot verwendet. Synchronisierte Snapshots sind nötig, wenn zwei oder mehr Sitzungen identische Inhalte in der Datenbank sehen müssen. Wenn zwei Sitzungen ihre Transaktionen einfach unabhängig starten, besteht immer die Möglichkeit, dass eine dritte Transaktion zwischen den beiden `START TRANSACTION`-Befehlen committet, sodass eine Sitzung die Effekte dieser Transaktion sieht und die andere nicht.

Zur Lösung dieses Problems erlaubt PostgreSQL einer Transaktion, den verwendeten Snapshot zu exportieren. Solange die exportierende Transaktion offen bleibt, können andere Transaktionen ihren Snapshot importieren und haben dadurch die Garantie, genau dieselbe Sicht auf die Datenbank zu sehen wie die erste Transaktion. Beachten Sie aber, dass Datenbankänderungen, die eine dieser Transaktionen selbst vornimmt, für die anderen Transaktionen unsichtbar bleiben, wie es für Änderungen nicht committeter Transaktionen üblich ist. Die Transaktionen sind also bezogen auf vorher vorhandene Daten synchronisiert, verhalten sich aber normal für Änderungen, die sie selbst vornehmen.

Snapshots werden mit der in Tabelle 9.100 gezeigten Funktion `pg_export_snapshot` exportiert und mit dem Befehl `SET TRANSACTION` importiert.

**Tabelle 9.100. Snapshot-Synchronisationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_export_snapshot ()` -> `text` | Speichert den aktuellen Snapshot der Transaktion und gibt eine Textzeichenkette zurück, die den Snapshot identifiziert. Diese Zeichenkette muss außerhalb der Datenbank an Clients weitergegeben werden, die den Snapshot importieren möchten. Der Snapshot steht nur bis zum Ende der Transaktion, die ihn exportiert hat, für den Import zur Verfügung. Eine Transaktion kann bei Bedarf mehr als einen Snapshot exportieren. Das ist nur bei `READ COMMITTED`-Transaktionen sinnvoll, da Transaktionen auf Isolationsebene `REPEATABLE READ` und höher während ihrer gesamten Lebensdauer denselben Snapshot verwenden. Sobald eine Transaktion Snapshots exportiert hat, kann sie nicht mit `PREPARE TRANSACTION` vorbereitet werden. |
| `pg_log_standby_snapshot ()` -> `pg_lsn` | Erstellt einen Snapshot laufender Transaktionen und schreibt ihn in WAL, ohne auf bgwriter oder checkpointer warten zu müssen. Dies ist für logisches Decoding auf Standbys nützlich, da die Erstellung eines logischen Slots warten muss, bis ein solcher Eintrag auf dem Standby wiedergegeben wurde. |

### 9.28.6. Replikationsverwaltungsfunktionen

Die in Tabelle 9.101 gezeigten Funktionen steuern Replikationsfunktionen oder interagieren mit ihnen. Informationen zu den zugrunde liegenden Funktionen finden Sie in Abschnitt 26.2.5, Abschnitt 26.2.6 und [Kapitel 48](48_Fortschritt_der_Replikation_verfolgen.md). Funktionen für Replication Origins dürfen standardmäßig nur von Superusern verwendet werden, können aber mit `GRANT` anderen Benutzern erlaubt werden. Funktionen für Replikationsslots sind auf Superuser und Benutzer mit `REPLICATION`-Recht beschränkt.

Viele dieser Funktionen haben entsprechende Befehle im Replikationsprotokoll; siehe [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll).

Die in Abschnitt 9.28.3, Abschnitt 9.28.4 und Abschnitt 9.28.5 beschriebenen Funktionen sind ebenfalls für Replikation relevant.

**Tabelle 9.101. Replikationsverwaltungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_create_physical_replication_slot ( slot_name name [, immediately_reserve boolean, temporary boolean ] )` -> `record ( slot_name name, lsn pg_lsn )` | Erstellt einen neuen physischen Replikationsslot namens `slot_name`. Wenn der optionale zweite Parameter true ist, wird die LSN für diesen Slot sofort reserviert; andernfalls bei der ersten Verbindung eines Streaming-Replikationsclients. Streaming-Änderungen aus einem physischen Slot sind nur mit dem Streaming-Replikationsprotokoll möglich, siehe [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll). Wenn der optionale dritte Parameter `temporary` true ist, wird der Slot nicht dauerhaft auf Platte gespeichert und ist nur für die aktuelle Sitzung gedacht. Temporäre Slots werden auch bei Fehlern freigegeben. Diese Funktion entspricht dem Replikationsprotokollbefehl `CREATE_REPLICATION_SLOT ... PHYSICAL`. |
| `pg_drop_replication_slot ( slot_name name )` -> `void` | Löscht den physischen oder logischen Replikationsslot namens `slot_name`. Entspricht dem Replikationsprotokollbefehl `DROP_REPLICATION_SLOT`. |
| `pg_create_logical_replication_slot ( slot_name name, plugin name [, temporary boolean, twophase boolean, failover boolean ] )` -> `record ( slot_name name, lsn pg_lsn )` | Erstellt einen neuen logischen Decoding-Replikationsslot namens `slot_name` mit dem Ausgabeplugin `plugin`. `temporary` legt fest, dass der Slot nicht dauerhaft gespeichert wird und nur für die aktuelle Sitzung gedacht ist. `twophase` aktiviert das Decoding vorbereiteter Transaktionen für diesen Slot. `failover` aktiviert die Synchronisierung dieses Slots zu Standbys, damit logische Replikation nach einem Failover fortgesetzt werden kann. Der Aufruf entspricht `CREATE_REPLICATION_SLOT ... LOGICAL`. |
| `pg_copy_physical_replication_slot ( src_slot_name name, dst_slot_name name [, temporary boolean ] )` -> `record ( slot_name name, lsn pg_lsn )` | Kopiert einen vorhandenen physischen Replikationsslot `src_slot_name` in einen physischen Slot `dst_slot_name`. Der kopierte Slot beginnt, WAL ab derselben LSN wie der Quellslot zu reservieren. Ist `temporary` ausgelassen, wird derselbe Wert wie beim Quellslot verwendet. Das Kopieren eines invalidierten Slots ist nicht erlaubt. |
| `pg_copy_logical_replication_slot ( src_slot_name name, dst_slot_name name [, temporary boolean [, plugin name ]] )` -> `record ( slot_name name, lsn pg_lsn )` | Kopiert einen vorhandenen logischen Replikationsslot `src_slot_name` in einen logischen Slot `dst_slot_name`, optional mit Änderung von Ausgabeplugin und Persistenz. Der kopierte logische Slot beginnt bei derselben LSN wie der Quellslot. Sind `temporary` und `plugin` ausgelassen, werden die Werte des Quellslots verwendet. Die Failover-Option des Quellslots wird nicht kopiert und standardmäßig auf false gesetzt, um das Risiko zu vermeiden, nach Failover auf einen Standby, auf dem der Slot synchronisiert wird, die logische Replikation nicht fortsetzen zu können. Das Kopieren eines invalidierten Slots ist nicht erlaubt. |
| `pg_logical_slot_get_changes ( slot_name name, upto_lsn pg_lsn, upto_nchanges integer, VARIADIC options text[] )` -> `setof record ( lsn pg_lsn, xid xid, data text )` | Gibt Änderungen im Slot `slot_name` zurück, beginnend an der Stelle, bis zu der zuletzt Änderungen konsumiert wurden. Wenn `upto_lsn` und `upto_nchanges` `NULL` sind, läuft logisches Decoding bis zum Ende von WAL. Ist `upto_lsn` nicht `NULL`, werden nur Transaktionen einbezogen, die vor der angegebenen LSN committen. Ist `upto_nchanges` nicht `NULL`, stoppt Decoding, wenn die Zahl der erzeugten Zeilen diesen Wert überschreitet. Tatsächlich können mehr Zeilen zurückgegeben werden, weil die Grenze erst nach dem Hinzufügen der Zeilen eines neu decodierten Commits geprüft wird. Bei einem logischen Failover-Slot kehrt die Funktion erst zurück, wenn alle in `synchronized_standby_slots` angegebenen physischen Slots den WAL-Empfang bestätigt haben. |
| `pg_logical_slot_peek_changes ( slot_name name, upto_lsn pg_lsn, upto_nchanges integer, VARIADIC options text[] )` -> `setof record ( lsn pg_lsn, xid xid, data text )` | Verhält sich wie `pg_logical_slot_get_changes()`, konsumiert die Änderungen aber nicht; sie werden bei zukünftigen Aufrufen erneut zurückgegeben. |
| `pg_logical_slot_get_binary_changes ( slot_name name, upto_lsn pg_lsn, upto_nchanges integer, VARIADIC options text[] )` -> `setof record ( lsn pg_lsn, xid xid, data bytea )` | Verhält sich wie `pg_logical_slot_get_changes()`, gibt Änderungen aber als `bytea` zurück. |
| `pg_logical_slot_peek_binary_changes ( slot_name name, upto_lsn pg_lsn, upto_nchanges integer, VARIADIC options text[] )` -> `setof record ( lsn pg_lsn, xid xid, data bytea )` | Verhält sich wie `pg_logical_slot_peek_changes()`, gibt Änderungen aber als `bytea` zurück. |
| `pg_replication_slot_advance ( slot_name name, upto_lsn pg_lsn )` -> `record ( slot_name name, end_lsn pg_lsn )` | Verschiebt die aktuell bestätigte Position des Replikationsslots `slot_name` vorwärts. Der Slot wird weder rückwärts noch über die aktuelle Insert-Position hinaus verschoben. Zurückgegeben werden der Slotname und die tatsächliche erreichte Position. Bei einer Verschiebung werden die aktualisierten Positionsinformationen beim nächsten Checkpoint geschrieben; nach einem Absturz kann der Slot daher zu einer früheren Position zurückkehren. Bei einem logischen Failover-Slot kehrt die Funktion erst zurück, wenn alle in `synchronized_standby_slots` angegebenen physischen Slots den WAL-Empfang bestätigt haben. |
| `pg_replication_origin_create ( node_name text )` -> `oid` | Erstellt einen Replication Origin mit dem angegebenen externen Namen und gibt die intern zugewiesene ID zurück. Der Name darf höchstens 512 Bytes lang sein. |
| `pg_replication_origin_drop ( node_name text )` -> `void` | Löscht einen zuvor erstellten Replication Origin einschließlich des zugehörigen Replay-Fortschritts. |
| `pg_replication_origin_oid ( node_name text )` -> `oid` | Sucht einen Replication Origin per Name und gibt die interne ID zurück. Wird keiner gefunden, wird `NULL` zurückgegeben. |
| `pg_replication_origin_session_setup ( node_name text )` -> `void` | Markiert die aktuelle Sitzung als Replay vom angegebenen Origin, sodass Replay-Fortschritt verfolgt werden kann. Dies kann nur verwendet werden, wenn aktuell kein Origin ausgewählt ist. Mit `pg_replication_origin_session_reset` wird es rückgängig gemacht. |
| `pg_replication_origin_session_reset ()` -> `void` | Hebt die Wirkung von `pg_replication_origin_session_setup()` auf. |
| `pg_replication_origin_session_is_setup ()` -> `boolean` | Gibt true zurück, wenn in der aktuellen Sitzung ein Replication Origin ausgewählt wurde. |
| `pg_replication_origin_session_progress ( flush boolean )` -> `pg_lsn` | Gibt die Replay-Position für den in der aktuellen Sitzung ausgewählten Replication Origin zurück. Der Parameter `flush` bestimmt, ob garantiert sein muss, dass die entsprechende lokale Transaktion auf Platte geflusht wurde. |
| `pg_replication_origin_xact_setup ( origin_lsn pg_lsn, origin_timestamp timestamp with time zone )` -> `void` | Markiert die aktuelle Transaktion als Replay einer Transaktion, die an der angegebenen LSN und zum angegebenen Zeitstempel committet hat. Kann nur aufgerufen werden, wenn mit `pg_replication_origin_session_setup` ein Replication Origin ausgewählt wurde. |
| `pg_replication_origin_xact_reset ()` -> `void` | Hebt die Wirkung von `pg_replication_origin_xact_setup()` auf. |
| `pg_replication_origin_advance ( node_name text, lsn pg_lsn )` -> `void` | Setzt den Replikationsfortschritt für den angegebenen Knoten auf die angegebene Position. Das ist vor allem zum Setzen einer Anfangsposition oder einer neuen Position nach Konfigurationsänderungen nützlich. Achtloser Gebrauch dieser Funktion kann zu inkonsistent replizierten Daten führen. |
| `pg_replication_origin_progress ( node_name text, flush boolean )` -> `pg_lsn` | Gibt die Replay-Position für den angegebenen Replication Origin zurück. Der Parameter `flush` bestimmt, ob garantiert sein muss, dass die entsprechende lokale Transaktion auf Platte geflusht wurde. |
| `pg_logical_emit_message ( transactional boolean, prefix text, content text [, flush boolean DEFAULT false] )` -> `pg_lsn`<br>`pg_logical_emit_message ( transactional boolean, prefix text, content bytea [, flush boolean DEFAULT false] )` -> `pg_lsn` | Gibt eine logische Decoding-Nachricht aus. Damit können allgemeine Nachrichten über WAL an logische Decoding-Plugins übergeben werden. `transactional` legt fest, ob die Nachricht Teil der aktuellen Transaktion sein oder sofort geschrieben und decodiert werden soll, sobald der logische Decoder den Eintrag liest. `prefix` ist ein Textpräfix, mit dem Plugins interessante Nachrichten erkennen können. `content` ist der Inhalt der Nachricht in Text- oder Binärform. `flush`, standardmäßig false, steuert, ob die Nachricht sofort nach WAL geflusht wird. Bei `transactional` hat `flush` keine Wirkung, weil der WAL-Eintrag der Nachricht zusammen mit der Transaktion geflusht wird. |
| `pg_sync_replication_slots ()` -> `void` | Synchronisiert logische Failover-Replikationsslots vom Primary-Server zum Standby-Server. Diese Funktion kann nur auf dem Standby ausgeführt werden. Temporär synchronisierte Slots können nicht für logisches Decoding verwendet werden und müssen nach einer Beförderung gelöscht werden. Details siehe Abschnitt 47.2.3. Die Funktion ist vor allem für Tests und Debugging gedacht und sollte vorsichtig verwendet werden. Außerdem kann sie nicht ausgeführt werden, wenn `sync_replication_slots` aktiviert ist und der Slotsync-Worker bereits zur Synchronisierung von Slots läuft. |

> **Vorsicht**
> Wenn nach Ausführung der Funktion `hot_standby_feedback` auf dem Standby deaktiviert wird oder der in `primary_slot_name` konfigurierte physische Slot entfernt wird, können die für den synchronisierten Slot nötigen Zeilen durch `VACUUM` auf dem Primary entfernt werden. Dadurch kann der synchronisierte Slot invalidiert werden.

### 9.28.7. Funktionen zur Verwaltung von Datenbankobjekten

Die in Tabelle 9.102 gezeigten Funktionen berechnen den Speicherplatzverbrauch von Datenbankobjekten oder helfen bei der Darstellung beziehungsweise beim Verständnis der Ergebnisse. `bigint`-Ergebnisse werden in Bytes gemessen. Wenn eine OID übergeben wird, die kein existierendes Objekt darstellt, geben diese Funktionen `NULL` zurück.

**Tabelle 9.102. Größenfunktionen für Datenbankobjekte**

| Funktion | Beschreibung |
|---|---|
| `pg_column_size ( "any" )` -> `integer` | Zeigt die Anzahl der Bytes, die zum Speichern eines einzelnen Datenwerts verwendet werden. Direkt auf einen Tabellenspaltenwert angewendet, berücksichtigt dies eine eventuell vorgenommene Kompression. |
| `pg_column_compression ( "any" )` -> `text` | Zeigt den Kompressionsalgorithmus, mit dem ein einzelner variabel langer Wert komprimiert wurde. Gibt `NULL` zurück, wenn der Wert nicht komprimiert ist. |
| `pg_column_toast_chunk_id ( "any" )` -> `oid` | Zeigt die `chunk_id` eines auf Platte befindlichen TOAST-Werts. Gibt `NULL` zurück, wenn der Wert nicht getoastet ist oder nicht auf Platte liegt. Weitere Informationen zu TOAST finden Sie in [Abschnitt 66.2](66_Physische_Speicherung_der_Datenbank.md#662-toast). |
| `pg_database_size ( name )` -> `bigint`<br>`pg_database_size ( oid )` -> `bigint` | Berechnet den gesamten von der Datenbank mit dem angegebenen Namen oder der angegebenen OID belegten Plattenplatz. Um diese Funktion zu verwenden, benötigen Sie `CONNECT`-Rechte auf der angegebenen Datenbank, die standardmäßig gewährt werden, oder Rechte der Rolle `pg_read_all_stats`. |
| `pg_indexes_size ( regclass )` -> `bigint` | Berechnet den gesamten von Indizes belegten Plattenplatz, die an die angegebene Tabelle angehängt sind. |
| `pg_relation_size ( relation regclass [, fork text ] )` -> `bigint` | Berechnet den Plattenplatz, der von einem Fork der angegebenen Relation belegt wird. Für die meisten Zwecke sind die höherstufigen Funktionen `pg_total_relation_size` oder `pg_table_size` bequemer, weil sie die Größen aller Forks aufsummieren. Mit einem Argument wird die Größe des Hauptdatenforks zurückgegeben. Das zweite Argument kann `main`, `fsm`, `vm` oder `init` sein: `main` ist der Hauptdatenfork, `fsm` die Free Space Map, siehe [Abschnitt 66.3](66_Physische_Speicherung_der_Datenbank.md#663-free-space-map), `vm` die Visibility Map, siehe [Abschnitt 66.4](66_Physische_Speicherung_der_Datenbank.md#664-visibility-map), und `init` der Initialisierungsfork, sofern vorhanden. |
| `pg_size_bytes ( text )` -> `bigint` | Wandelt eine Größe in menschenlesbarem Format, wie von `pg_size_pretty` zurückgegeben, in Bytes um. Gültige Einheiten sind `bytes`, `B`, `kB`, `MB`, `GB`, `TB` und `PB`. |
| `pg_size_pretty ( bigint )` -> `text`<br>`pg_size_pretty ( numeric )` -> `text` | Wandelt eine Größe in Bytes in ein leichter lesbares Format mit Größeneinheiten um, also `bytes`, `kB`, `MB`, `GB`, `TB` oder `PB`. Die Einheiten sind Potenzen von 2 statt Potenzen von 10: 1 kB sind 1024 Bytes, 1 MB ist 1024^2 = 1048576 Bytes usw. |
| `pg_table_size ( regclass )` -> `bigint` | Berechnet den von der angegebenen Tabelle belegten Plattenplatz ohne Indizes, aber einschließlich ihrer TOAST-Tabelle, sofern vorhanden, sowie Free Space Map und Visibility Map. |
| `pg_tablespace_size ( name )` -> `bigint`<br>`pg_tablespace_size ( oid )` -> `bigint` | Berechnet den gesamten im Tablespace mit dem angegebenen Namen oder der angegebenen OID belegten Plattenplatz. Um diese Funktion zu verwenden, benötigen Sie `CREATE`-Rechte auf dem angegebenen Tablespace oder Rechte der Rolle `pg_read_all_stats`, außer es handelt sich um den Standardtablespace der aktuellen Datenbank. |
| `pg_total_relation_size ( regclass )` -> `bigint` | Berechnet den gesamten von der angegebenen Tabelle belegten Plattenplatz einschließlich aller Indizes und TOAST-Daten. Das Ergebnis entspricht `pg_table_size + pg_indexes_size`. |

Die obigen Funktionen, die mit Tabellen oder Indizes arbeiten, akzeptieren ein `regclass`-Argument, also einfach die OID der Tabelle oder des Index im Systemkatalog `pg_class`. Sie müssen die OID jedoch nicht von Hand nachschlagen, weil der Eingabekonverter des Datentyps `regclass` dies übernimmt. Details finden Sie in [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen).

Die in Tabelle 9.103 gezeigten Funktionen helfen dabei, die spezifischen Plattendateien zu identifizieren, die Datenbankobjekten zugeordnet sind.

**Tabelle 9.103. Speicherortfunktionen für Datenbankobjekte**

| Funktion | Beschreibung |
|---|---|
| `pg_relation_filenode ( relation regclass )` -> `oid` | Gibt die Filenode-Nummer zurück, die der angegebenen Relation aktuell zugewiesen ist. Die Filenode ist der Basisteil des oder der Dateinamen der Relation, siehe [Abschnitt 66.1](66_Physische_Speicherung_der_Datenbank.md#661-dateilayout-der-datenbank). Für die meisten Relationen entspricht das Ergebnis `pg_class.relfilenode`; bei bestimmten Systemkatalogen ist `relfilenode` jedoch null, und diese Funktion muss verwendet werden. Gibt `NULL` zurück, wenn die Relation keinen Speicher hat, etwa bei einer View. |
| `pg_relation_filepath ( relation regclass )` -> `text` | Gibt den vollständigen Dateipfad der Relation relativ zum Datenverzeichnis des Datenbankclusters, `PGDATA`, zurück. |
| `pg_filenode_relation ( tablespace oid, filenode oid )` -> `regclass` | Gibt zu Tablespace-OID und Filenode die OID einer Relation zurück. Dies ist im Wesentlichen die Umkehrabbildung von `pg_relation_filepath`. Für eine Relation im Standardtablespace der Datenbank kann der Tablespace als null angegeben werden. Gibt `NULL` zurück, wenn in der aktuellen Datenbank keine Relation mit den angegebenen Werten verknüpft ist oder wenn es sich um eine temporäre Relation handelt. |

Tabelle 9.104 führt Funktionen zur Verwaltung von Kollationen auf.

**Tabelle 9.104. Kollationsverwaltungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_collation_actual_version ( oid )` -> `text` | Gibt die tatsächliche Version des Kollationsobjekts zurück, wie sie aktuell im Betriebssystem installiert ist. Wenn diese von `pg_collation.collversion` abweicht, müssen von der Kollation abhängige Objekte möglicherweise neu erstellt werden. Siehe auch `ALTER COLLATION`. |
| `pg_database_collation_actual_version ( oid )` -> `text` | Gibt die tatsächliche Version der Kollation der Datenbank zurück, wie sie aktuell im Betriebssystem installiert ist. Wenn diese von `pg_database.datcollversion` abweicht, müssen von der Kollation abhängige Objekte möglicherweise neu erstellt werden. Siehe auch `ALTER DATABASE`. |
| `pg_import_system_collations ( schema regnamespace )` -> `integer` | Fügt dem Systemkatalog `pg_collation` Kollationen hinzu, basierend auf allen Locales, die im Betriebssystem gefunden werden. Dies verwendet auch `initdb`; Details siehe Abschnitt 23.2.2. Wenn später zusätzliche Locales im Betriebssystem installiert werden, kann diese Funktion erneut ausgeführt werden, um Kollationen für die neuen Locales hinzuzufügen. Locales, die vorhandenen Einträgen in `pg_collation` entsprechen, werden übersprungen. Kollationsobjekte für Locales, die im Betriebssystem nicht mehr vorhanden sind, werden jedoch nicht entfernt. Der Parameter `schema` ist typischerweise `pg_catalog`, muss es aber nicht sein. Die Funktion gibt die Anzahl der neu erstellten Kollationsobjekte zurück. Ihre Verwendung ist auf Superuser beschränkt. |

Tabelle 9.105 führt Funktionen zur Manipulation von Statistiken auf. Diese Funktionen können während der Recovery nicht ausgeführt werden.

> **Warnung**
> Änderungen, die durch diese Statistikmanipulationsfunktionen vorgenommen werden, werden wahrscheinlich durch Autovacuum oder manuelles `VACUUM` beziehungsweise `ANALYZE` überschrieben und sollten als temporär betrachtet werden.

**Tabelle 9.105. Funktionen zur Manipulation von Datenbankobjektstatistiken**

| Funktion | Beschreibung |
|---|---|
| `pg_restore_relation_stats ( VARIADIC kwargs "any" )` -> `boolean` | Aktualisiert tabellenbezogene Statistiken. Normalerweise werden diese Statistiken automatisch gesammelt oder als Teil von `VACUUM` oder `ANALYZE` aktualisiert, sodass ein Aufruf nicht nötig ist. Nach einer Wiederherstellung kann die Funktion jedoch nützlich sein, damit der Optimizer bessere Pläne wählen kann, falls `ANALYZE` noch nicht ausgeführt wurde. Da sich verfolgte Statistiken zwischen Versionen ändern können, werden Argumente als Paare aus Argumentname und Argumentwert übergeben, etwa wie im Beispiel unten. Erforderlich sind `schemaname` und `relname`; weitere Argumente entsprechen bestimmten Spalten in `pg_class`. Derzeit unterstützt werden `relpages` als `integer`, `reltuples` als `real`, `relallvisible` als `integer` und `relallfrozen` als `integer`. Zusätzlich akzeptiert die Funktion `version` als `integer`, um die Serverversion anzugeben, aus der die Statistiken stammen. Kleinere Fehler werden als `WARNING` gemeldet und ignoriert; verbleibende Statistiken werden trotzdem wiederhergestellt. Wenn alle angegebenen Statistiken erfolgreich wiederhergestellt werden, gibt die Funktion true zurück, andernfalls false. Der Aufrufer muss das `MAINTAIN`-Recht auf der Tabelle besitzen oder Eigentümer der Datenbank sein. |
| `pg_clear_relation_stats ( schemaname text, relname text )` -> `void` | Löscht tabellenbezogene Statistiken für die angegebene Relation, als wäre die Tabelle gerade neu erstellt worden. Der Aufrufer muss das `MAINTAIN`-Recht auf der Tabelle besitzen oder Eigentümer der Datenbank sein. |
| `pg_restore_attribute_stats ( VARIADIC kwargs "any" )` -> `boolean` | Erstellt oder aktualisiert spaltenbezogene Statistiken. Normalerweise werden diese Statistiken automatisch gesammelt oder als Teil von `VACUUM` oder `ANALYZE` aktualisiert. Nach einer Wiederherstellung kann die Funktion jedoch nützlich sein, wenn `ANALYZE` noch nicht ausgeführt wurde. Argumente werden als Paare aus Argumentname und Argumentwert übergeben. Erforderlich sind `schemaname` und `relname` als `text`, außerdem entweder `attname` als `text` oder `attnum` als `smallint` zur Angabe der Spalte, sowie `inherited`, das angibt, ob die Statistiken Werte aus Kindtabellen einschließen. Weitere Argumente entsprechen Spalten in `pg_stats`. Zusätzlich akzeptiert die Funktion `version` als `integer`. Kleinere Fehler werden als `WARNING` gemeldet und ignoriert. Wenn alle angegebenen Statistiken erfolgreich wiederhergestellt werden, gibt die Funktion true zurück, andernfalls false. Der Aufrufer muss das `MAINTAIN`-Recht auf der Tabelle besitzen oder Eigentümer der Datenbank sein. |
| `pg_clear_attribute_stats ( schemaname text, relname text, attname text, inherited boolean )` -> `void` | Löscht spaltenbezogene Statistiken für die angegebene Relation und das angegebene Attribut, als wäre die Tabelle gerade neu erstellt worden. Der Aufrufer muss das `MAINTAIN`-Recht auf der Tabelle besitzen oder Eigentümer der Datenbank sein. |

Beispiel für `pg_restore_relation_stats`:

```sql
SELECT pg_restore_relation_stats(
    'schemaname', 'myschema',
    'relname',    'mytable',
    'relpages',   173::integer,
    'reltuples', 10000::real);
```

Beispiel für `pg_restore_attribute_stats`:

```sql
SELECT pg_restore_attribute_stats(
    'schemaname', 'myschema',
    'relname',    'mytable',
    'attname',    'col1',
    'inherited', false,
    'avg_width', 125::integer,
    'null_frac', 0.5::real);
```

Tabelle 9.106 führt Funktionen auf, die Informationen über die Struktur partitionierter Tabellen bereitstellen.

**Tabelle 9.106. Partitionierungsinformationsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_partition_tree ( regclass )` -> `setof record ( relid regclass, parentrelid regclass, isleaf boolean, level integer )` | Listet die Tabellen oder Indizes im Partitionsbaum der angegebenen partitionierten Tabelle oder des angegebenen partitionierten Index auf, mit einer Zeile pro Partition. Die Informationen enthalten die OID der Partition, die OID des unmittelbaren Elternobjekts, einen booleschen Wert, ob die Partition ein Blatt ist, und eine Ganzzahl für ihre Ebene in der Hierarchie. Ebene 0 ist die Eingabetabelle oder der Eingabeindex, Ebene 1 die unmittelbaren Kindpartitionen usw. Gibt keine Zeilen zurück, wenn die Relation nicht existiert oder keine Partition beziehungsweise partitionierte Tabelle ist. |
| `pg_partition_ancestors ( regclass )` -> `setof regclass` | Listet die Vorgängerrelationen der angegebenen Partition einschließlich der Relation selbst auf. Gibt keine Zeilen zurück, wenn die Relation nicht existiert oder keine Partition beziehungsweise partitionierte Tabelle ist. |
| `pg_partition_root ( regclass )` -> `regclass` | Gibt das oberste Elternobjekt des Partitionsbaums zurück, zu dem die angegebene Relation gehört. Gibt `NULL` zurück, wenn die Relation nicht existiert oder keine Partition beziehungsweise partitionierte Tabelle ist. |

Um zum Beispiel die Gesamtgröße der Daten in einer partitionierten Tabelle `measurement` zu prüfen, kann folgende Abfrage verwendet werden:

```sql
SELECT pg_size_pretty(sum(pg_relation_size(relid))) AS total_size
FROM pg_partition_tree('measurement');
```

### 9.28.8. Indexwartungsfunktionen

Tabelle 9.107 zeigt die Funktionen, die für Indexwartungsaufgaben verfügbar sind. Diese Wartungsaufgaben werden normalerweise automatisch durch Autovacuum erledigt; die Verwendung dieser Funktionen ist nur in Sonderfällen erforderlich. Diese Funktionen können während der Recovery nicht ausgeführt werden. Ihre Verwendung ist auf Superuser und den Eigentümer des angegebenen Index beschränkt.

**Tabelle 9.107. Indexwartungsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `brin_summarize_new_values ( index regclass )` -> `integer` | Durchsucht den angegebenen BRIN-Index nach Seitenbereichen in der Basistabelle, die aktuell nicht vom Index zusammengefasst sind; für jeden solchen Bereich wird durch Scannen dieser Tabellenseiten ein neuer Zusammenfassungsindexeintrag erstellt. Gibt die Anzahl der neu in den Index eingefügten Seitenbereichszusammenfassungen zurück. |
| `brin_summarize_range ( index regclass, blockNumber bigint )` -> `integer` | Fasst den Seitenbereich zusammen, der den angegebenen Block enthält, sofern er nicht bereits zusammengefasst ist. Dies entspricht `brin_summarize_new_values`, verarbeitet aber nur den Seitenbereich, der die angegebene Tabellenblocknummer enthält. |
| `brin_desummarize_range ( index regclass, blockNumber bigint )` -> `void` | Entfernt den BRIN-Indexeintrag, der den Seitenbereich des angegebenen Tabellenblocks zusammenfasst, sofern ein solcher Eintrag vorhanden ist. |
| `gin_clean_pending_list ( index regclass )` -> `bigint` | Räumt die Pending-Liste des angegebenen GIN-Index auf, indem Einträge daraus gesammelt in die Hauptdatenstruktur des GIN verschoben werden. Gibt die Anzahl der aus der Pending-Liste entfernten Seiten zurück. Wenn das Argument ein GIN-Index ist, der mit deaktivierter Option `fastupdate` erstellt wurde, geschieht keine Bereinigung und das Ergebnis ist null, weil der Index keine Pending-Liste hat. Details zur Pending-Liste und Option `fastupdate` finden Sie in Abschnitt 65.4.4.1 und Abschnitt 65.4.5. |

### 9.28.9. Allgemeine Dateizugriffsfunktionen

Die in Tabelle 9.108 gezeigten Funktionen bieten nativen Zugriff auf Dateien auf der Maschine, auf der der Server läuft. Nur Dateien innerhalb des Datenbankclusterverzeichnisses und von `log_directory` können gelesen werden, außer der Benutzer ist Superuser oder hat die Rolle `pg_read_server_files`. Verwenden Sie für Dateien im Clusterverzeichnis relative Pfade und für Logdateien einen Pfad, der der Konfigurationseinstellung `log_directory` entspricht.

Beachten Sie, dass das Gewähren des `EXECUTE`-Rechts auf `pg_read_file()` oder verwandte Funktionen Benutzern die Möglichkeit gibt, jede Datei auf dem Server zu lesen, die der Datenbankserverprozess lesen kann. Diese Funktionen umgehen alle datenbankinternen Berechtigungsprüfungen. Das bedeutet zum Beispiel, dass ein Benutzer mit solchem Zugriff den Inhalt der Tabelle `pg_authid`, in der Authentifizierungsinformationen gespeichert sind, sowie beliebige Tabellendaten in der Datenbank lesen kann. Der Zugriff auf diese Funktionen sollte daher sehr sorgfältig gewährt werden.

Wenn Rechte auf diese Funktionen gewährt werden, beachten Sie, dass Tabelleneinträge mit optionalen Parametern meist als mehrere physische Funktionen mit unterschiedlichen Parameterlisten implementiert sind. Rechte müssen separat auf jede solche Funktion gewährt werden, wenn sie verwendet werden soll. Der psql-Befehl `\df` kann hilfreich sein, um die tatsächlichen Funktionssignaturen zu prüfen.

Einige dieser Funktionen akzeptieren einen optionalen Parameter `missing_ok`, der das Verhalten festlegt, wenn die Datei oder das Verzeichnis nicht existiert. Bei true gibt die Funktion je nach Fall `NULL` oder eine leere Ergebnismenge zurück. Bei false wird ein Fehler ausgelöst. Andere Fehlerbedingungen als „Datei nicht gefunden“ werden in jedem Fall als Fehler gemeldet. Der Standard ist false.

**Tabelle 9.108. Allgemeine Dateizugriffsfunktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_ls_dir ( dirname text [, missing_ok boolean, include_dot_dirs boolean ] )` -> `setof text` | Gibt die Namen aller Dateien, Verzeichnisse und anderer Spezialdateien im angegebenen Verzeichnis zurück. `include_dot_dirs` legt fest, ob `.` und `..` in das Ergebnis aufgenommen werden; standardmäßig werden sie ausgelassen. Sie einzuschließen kann nützlich sein, wenn `missing_ok` true ist, um ein leeres von einem nicht existierenden Verzeichnis zu unterscheiden. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_logdir ()` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im Logverzeichnis des Servers zurück. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Rollen mit Rechten von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_waldir ()` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im WAL-Verzeichnis des Servers zurück. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Rollen mit Rechten von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_logicalmapdir ()` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im Verzeichnis `pg_logical/mappings` des Servers zurück. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Mitglieder von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_logicalsnapdir ()` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im Verzeichnis `pg_logical/snapshots` des Servers zurück. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Mitglieder von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_replslotdir ( slot_name text )` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im Verzeichnis `pg_replslot/slot_name` des Servers zurück, wobei `slot_name` der Name des als Eingabe bereitgestellten Replikationsslots ist. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Mitglieder von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_summariesdir ()` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im WAL-Zusammenfassungsverzeichnis des Servers, `pg_wal/summaries`, zurück. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Mitglieder von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_archive_statusdir ()` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im WAL-Archivstatusverzeichnis des Servers, `pg_wal/archive_status`, zurück. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Mitglieder von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_ls_tmpdir ( [ tablespace oid ] )` -> `setof record ( name text, size bigint, modification timestamp with time zone )` | Gibt Name, Größe und letzte Änderungszeit jeder gewöhnlichen Datei im temporären Dateiverzeichnis für den angegebenen Tablespace zurück. Wenn `tablespace` nicht angegeben ist, wird der Tablespace `pg_default` untersucht. Dateinamen, die mit Punkt beginnen, Verzeichnisse und andere Spezialdateien werden ausgelassen. Standardmäßig ist diese Funktion auf Superuser und Mitglieder von `pg_monitor` beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_read_file ( filename text [, offset bigint, length bigint ] [, missing_ok boolean ] )` -> `text` | Gibt eine Textdatei ganz oder teilweise zurück, beginnend am angegebenen Byteoffset und höchstens `length` Bytes lang, weniger, wenn vorher das Dateiende erreicht wird. Wenn `offset` negativ ist, wird er relativ zum Dateiende interpretiert. Wenn `offset` und `length` ausgelassen werden, wird die gesamte Datei zurückgegeben. Die gelesenen Bytes werden als Zeichenkette im Encoding der Datenbank interpretiert; wenn sie in diesem Encoding nicht gültig sind, wird ein Fehler ausgelöst. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |
| `pg_read_binary_file ( filename text [, offset bigint, length bigint ] [, missing_ok boolean ] )` -> `bytea` | Gibt eine Datei ganz oder teilweise zurück. Diese Funktion ist identisch mit `pg_read_file`, außer dass sie beliebige Binärdaten lesen kann und das Ergebnis als `bytea` statt `text` zurückgibt; entsprechend werden keine Encodingprüfungen durchgeführt. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. In Kombination mit `convert_from` kann sie verwendet werden, um eine Textdatei in einem angegebenen Encoding zu lesen und in das Datenbankencoding umzuwandeln. |
| `pg_stat_file ( filename text [, missing_ok boolean ] )` -> `record ( size bigint, access timestamp with time zone, modification timestamp with time zone, change timestamp with time zone, creation timestamp with time zone, isdir boolean )` | Gibt einen Datensatz mit Dateigröße, Zeitpunkt des letzten Zugriffs, Zeitpunkt der letzten Änderung, Zeitpunkt der letzten Statusänderung der Datei auf Unix-Plattformen, Erstellungszeitpunkt auf Windows sowie einem Flag zurück, das angibt, ob es sich um ein Verzeichnis handelt. Standardmäßig ist diese Funktion auf Superuser beschränkt; anderen Benutzern kann jedoch `EXECUTE` gewährt werden. |

Beispiel für `pg_read_binary_file` zusammen mit `convert_from`:

```sql
SELECT convert_from(pg_read_binary_file('file_in_utf8.txt'), 'UTF8');
```

### 9.28.10. Advisory-Lock-Funktionen

Die in Tabelle 9.109 gezeigten Funktionen verwalten Advisory Locks. Details zur korrekten Verwendung dieser Funktionen finden Sie in Abschnitt 13.3.5.

Alle diese Funktionen dienen dazu, anwendungsdefinierte Ressourcen zu sperren, die entweder durch einen einzelnen 64-Bit-Schlüsselwert oder zwei 32-Bit-Schlüsselwerte identifiziert werden können. Diese beiden Schlüsselräume überlappen nicht. Wenn eine andere Sitzung bereits eine kollidierende Sperre auf demselben Ressourcenbezeichner hält, warten die Funktionen entweder, bis die Ressource verfügbar wird, oder geben je nach Funktion false zurück. Sperren können geteilt oder exklusiv sein: Eine geteilte Sperre kollidiert nicht mit anderen geteilten Sperren auf derselben Ressource, sondern nur mit exklusiven Sperren. Sperren können auf Sitzungsebene genommen werden, sodass sie bis zur Freigabe oder bis zum Sitzungsende gehalten werden, oder auf Transaktionsebene, sodass sie bis zum Ende der aktuellen Transaktion gehalten werden. Für Transaktionsebene gibt es keine manuelle Freigabe. Mehrere Sperranforderungen auf Sitzungsebene stapeln sich; wenn derselbe Ressourcenbezeichner dreimal gesperrt wird, sind drei Entsperranforderungen nötig, um die Ressource vor Sitzungsende freizugeben.

**Tabelle 9.109. Advisory-Lock-Funktionen**

| Funktion | Beschreibung |
|---|---|
| `pg_advisory_lock ( key bigint )` -> `void`<br>`pg_advisory_lock ( key1 integer, key2 integer )` -> `void` | Erlangt eine exklusive Advisory-Sperre auf Sitzungsebene und wartet bei Bedarf. |
| `pg_advisory_lock_shared ( key bigint )` -> `void`<br>`pg_advisory_lock_shared ( key1 integer, key2 integer )` -> `void` | Erlangt eine geteilte Advisory-Sperre auf Sitzungsebene und wartet bei Bedarf. |
| `pg_advisory_unlock ( key bigint )` -> `boolean`<br>`pg_advisory_unlock ( key1 integer, key2 integer )` -> `boolean` | Gibt eine zuvor erlangte exklusive Advisory-Sperre auf Sitzungsebene frei. Gibt true zurück, wenn die Sperre erfolgreich freigegeben wird. Wenn die Sperre nicht gehalten wurde, wird false zurückgegeben und zusätzlich eine SQL-Warnung vom Server gemeldet. |
| `pg_advisory_unlock_all ()` -> `void` | Gibt alle Advisory-Sperren auf Sitzungsebene frei, die von der aktuellen Sitzung gehalten werden. Diese Funktion wird beim Sitzungsende implizit aufgerufen, auch wenn der Client unsauber getrennt wird. |
| `pg_advisory_unlock_shared ( key bigint )` -> `boolean`<br>`pg_advisory_unlock_shared ( key1 integer, key2 integer )` -> `boolean` | Gibt eine zuvor erlangte geteilte Advisory-Sperre auf Sitzungsebene frei. Gibt true zurück, wenn die Sperre erfolgreich freigegeben wird. Wenn die Sperre nicht gehalten wurde, wird false zurückgegeben und zusätzlich eine SQL-Warnung vom Server gemeldet. |
| `pg_advisory_xact_lock ( key bigint )` -> `void`<br>`pg_advisory_xact_lock ( key1 integer, key2 integer )` -> `void` | Erlangt eine exklusive Advisory-Sperre auf Transaktionsebene und wartet bei Bedarf. |
| `pg_advisory_xact_lock_shared ( key bigint )` -> `void`<br>`pg_advisory_xact_lock_shared ( key1 integer, key2 integer )` -> `void` | Erlangt eine geteilte Advisory-Sperre auf Transaktionsebene und wartet bei Bedarf. |
| `pg_try_advisory_lock ( key bigint )` -> `boolean`<br>`pg_try_advisory_lock ( key1 integer, key2 integer )` -> `boolean` | Erlangt eine exklusive Advisory-Sperre auf Sitzungsebene, falls verfügbar. Die Funktion erlangt die Sperre sofort und gibt true zurück oder gibt ohne Warten false zurück, wenn die Sperre nicht sofort erlangt werden kann. |
| `pg_try_advisory_lock_shared ( key bigint )` -> `boolean`<br>`pg_try_advisory_lock_shared ( key1 integer, key2 integer )` -> `boolean` | Erlangt eine geteilte Advisory-Sperre auf Sitzungsebene, falls verfügbar. Die Funktion erlangt die Sperre sofort und gibt true zurück oder gibt ohne Warten false zurück, wenn die Sperre nicht sofort erlangt werden kann. |
| `pg_try_advisory_xact_lock ( key bigint )` -> `boolean`<br>`pg_try_advisory_xact_lock ( key1 integer, key2 integer )` -> `boolean` | Erlangt eine exklusive Advisory-Sperre auf Transaktionsebene, falls verfügbar. Die Funktion erlangt die Sperre sofort und gibt true zurück oder gibt ohne Warten false zurück, wenn die Sperre nicht sofort erlangt werden kann. |
| `pg_try_advisory_xact_lock_shared ( key bigint )` -> `boolean`<br>`pg_try_advisory_xact_lock_shared ( key1 integer, key2 integer )` -> `boolean` | Erlangt eine geteilte Advisory-Sperre auf Transaktionsebene, falls verfügbar. Die Funktion erlangt die Sperre sofort und gibt true zurück oder gibt ohne Warten false zurück, wenn die Sperre nicht sofort erlangt werden kann. |

## 9.29. Triggerfunktionen

Obwohl viele Verwendungen von Triggern benutzerdefinierte Triggerfunktionen einsetzen, stellt PostgreSQL einige eingebaute Triggerfunktionen bereit, die direkt in benutzerdefinierten Triggern verwendet werden können. Diese sind in Tabelle 9.110 zusammengefasst. Es gibt weitere eingebaute Triggerfunktionen, die Fremdschlüssel-Constraints und verzögerte Index-Constraints implementieren; sie werden hier nicht dokumentiert, weil Benutzer sie nicht direkt verwenden müssen.

Weitere Informationen zum Erstellen von Triggern finden Sie unter `CREATE TRIGGER`.

**Tabelle 9.110. Eingebaute Triggerfunktionen**

| Funktion | Beschreibung | Beispielverwendung |
|---|---|---|
| `suppress_redundant_updates_trigger ( )` -> `trigger` | Unterdrückt Update-Operationen, die nichts ändern. Details siehe unten. | `CREATE TRIGGER ... suppress_redundant_updates_trigger()` |
| `tsvector_update_trigger ( )` -> `trigger` | Aktualisiert automatisch eine `tsvector`-Spalte aus zugehörigen Dokumentenspalten mit Klartext. Die zu verwendende Textsuchkonfiguration wird namentlich als Triggerargument angegeben. Details siehe Abschnitt 12.4.3. | `CREATE TRIGGER ... tsvector_update_trigger(tsvcol, 'pg_catalog.swedish', title, body)` |
| `tsvector_update_trigger_column ( )` -> `trigger` | Aktualisiert automatisch eine `tsvector`-Spalte aus zugehörigen Dokumentenspalten mit Klartext. Die zu verwendende Textsuchkonfiguration wird aus einer `regconfig`-Spalte der Tabelle entnommen. Details siehe Abschnitt 12.4.3. | `CREATE TRIGGER ... tsvector_update_trigger_column(tsvcol, tsconfigcol, title, body)` |

Die Funktion `suppress_redundant_updates_trigger` verhindert, wenn sie als zeilenbezogener `BEFORE UPDATE`-Trigger eingesetzt wird, jedes Update, das die Daten in der Zeile nicht tatsächlich ändert. Das überschreibt das normale Verhalten, bei dem immer ein physisches Zeilenupdate ausgeführt wird, unabhängig davon, ob sich Daten geändert haben. Dieses normale Verhalten macht Updates schneller, weil keine Prüfung erforderlich ist, und ist außerdem in bestimmten Fällen nützlich.

Idealerweise sollten Sie vermeiden, Updates auszuführen, die die Daten im Datensatz nicht tatsächlich ändern. Redundante Updates können erhebliche unnötige Zeit kosten, insbesondere wenn viele Indizes geändert werden müssen, und sie erzeugen Platzverbrauch durch tote Zeilen, die später von `VACUUM` aufgeräumt werden müssen. Solche Situationen im Clientcode zu erkennen, ist jedoch nicht immer einfach oder überhaupt möglich; entsprechende Ausdrücke können fehleranfällig sein. Eine Alternative ist `suppress_redundant_updates_trigger`, das Updates überspringt, die die Daten nicht ändern. Diese Funktion sollte jedoch mit Bedacht verwendet werden. Der Trigger benötigt für jeden Datensatz eine kleine, aber nicht triviale Zeit; wenn die meisten von Updates betroffenen Datensätze tatsächlich geändert werden, werden Updates durch diesen Trigger im Durchschnitt langsamer.

Die Funktion `suppress_redundant_updates_trigger` kann einer Tabelle wie folgt hinzugefügt werden:

```sql
CREATE TRIGGER z_min_update
BEFORE UPDATE ON tablename
FOR EACH ROW EXECUTE FUNCTION suppress_redundant_updates_trigger();
```

In den meisten Fällen sollte dieser Trigger pro Zeile zuletzt ausgelöst werden, damit er andere Trigger nicht übergeht, die die Zeile möglicherweise ändern möchten. Da Trigger in Namensreihenfolge ausgelöst werden, wählen Sie daher einen Triggernamen, der nach den Namen anderer Trigger auf der Tabelle kommt. Daher das Präfix `z` im Beispiel.

## 9.30. Event-Trigger-Funktionen

PostgreSQL stellt diese Hilfsfunktionen bereit, um Informationen aus Event-Triggern abzurufen.

Weitere Informationen zu Event-Triggern finden Sie in [Kapitel 38](38_Event_Trigger.md).

### 9.30.1. Änderungen am Befehlsende erfassen

```sql
pg_event_trigger_ddl_commands () -> setof record
```

`pg_event_trigger_ddl_commands` gibt eine Liste der DDL-Befehle zurück, die durch jede Benutzeraktion ausgeführt wurden, wenn die Funktion in einer Funktion aufgerufen wird, die an einen `ddl_command_end`-Event-Trigger gebunden ist. In jedem anderen Kontext wird ein Fehler ausgelöst. `pg_event_trigger_ddl_commands` gibt eine Zeile für jeden ausgeführten Basisbefehl zurück; manche Befehle, die ein einzelner SQL-Satz sind, können mehr als eine Zeile zurückgeben. Diese Funktion gibt die folgenden Spalten zurück:

| Name | Typ | Beschreibung |
|---|---|---|
| `classid` | `oid` | OID des Katalogs, zu dem das Objekt gehört. |
| `objid` | `oid` | OID des Objekts selbst. |
| `objsubid` | `integer` | Unterobjekt-ID, zum Beispiel Attributnummer einer Spalte. |
| `command_tag` | `text` | Befehlstag. |
| `object_type` | `text` | Typ des Objekts. |
| `schema_name` | `text` | Name des Schemas, zu dem das Objekt gehört, falls vorhanden; andernfalls `NULL`. Es wird keine Quotierung angewendet. |
| `object_identity` | `text` | Textdarstellung der Objektidentität, schemagebunden qualifiziert. Jeder in der Identität enthaltene Bezeichner wird bei Bedarf quotiert. |
| `in_extension` | `boolean` | `true`, wenn der Befehl Teil eines Erweiterungsskripts ist. |
| `command` | `pg_ddl_command` | Eine vollständige Darstellung des Befehls im internen Format. Diese kann nicht direkt ausgegeben werden, kann aber an andere Funktionen übergeben werden, um verschiedene Informationen über den Befehl zu erhalten. |

### 9.30.2. Von einem DDL-Befehl gelöschte Objekte verarbeiten

```sql
pg_event_trigger_dropped_objects () -> setof record
```

`pg_event_trigger_dropped_objects` gibt eine Liste aller Objekte zurück, die durch den Befehl gelöscht wurden, in dessen `sql_drop`-Event sie aufgerufen wird. In jedem anderen Kontext wird ein Fehler ausgelöst. Diese Funktion gibt die folgenden Spalten zurück:

| Name | Typ | Beschreibung |
|---|---|---|
| `classid` | `oid` | OID des Katalogs, zu dem das Objekt gehörte. |
| `objid` | `oid` | OID des Objekts selbst. |
| `objsubid` | `integer` | Unterobjekt-ID, zum Beispiel Attributnummer einer Spalte. |
| `original` | `boolean` | `true`, wenn dies eines der Wurzelobjekte der Löschung war. |
| `normal` | `boolean` | `true`, wenn es im Abhängigkeitsgraphen eine normale Abhängigkeitsbeziehung gab, die zu diesem Objekt führte. |
| `is_temporary` | `boolean` | `true`, wenn dies ein temporäres Objekt war. |
| `object_type` | `text` | Typ des Objekts. |
| `schema_name` | `text` | Name des Schemas, zu dem das Objekt gehörte, falls vorhanden; andernfalls `NULL`. Es wird keine Quotierung angewendet. |
| `object_name` | `text` | Name des Objekts, wenn die Kombination aus Schema und Name als eindeutiger Bezeichner für das Objekt verwendet werden kann; andernfalls `NULL`. Es wird keine Quotierung angewendet, und der Name ist niemals schemaqualifiziert. |
| `object_identity` | `text` | Textdarstellung der Objektidentität, schemagebunden qualifiziert. Jeder in der Identität enthaltene Bezeichner wird bei Bedarf quotiert. |
| `address_names` | `text[]` | Ein Array, das zusammen mit `object_type` und `address_args` von der Funktion `pg_get_object_address` verwendet werden kann, um die Objektadresse auf einem entfernten Server mit einem gleichnamigen Objekt derselben Art wiederherzustellen. |
| `address_args` | `text[]` | Ergänzung zu `address_names`. |

Die Funktion `pg_event_trigger_dropped_objects` kann in einem Event-Trigger wie folgt verwendet werden:

```sql
CREATE FUNCTION test_event_trigger_for_drops()
  RETURNS event_trigger LANGUAGE plpgsql AS $$
DECLARE
    obj record;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_dropped_objects()
    LOOP
        RAISE NOTICE '% dropped object: % %.% %',
                     tg_tag,
                     obj.object_type,
                     obj.schema_name,
                     obj.object_name,
                     obj.object_identity;
    END LOOP;
END;
$$;

CREATE EVENT TRIGGER test_event_trigger_for_drops
  ON sql_drop
  EXECUTE FUNCTION test_event_trigger_for_drops();
```

### 9.30.3. Ein Tabellen-Rewrite-Ereignis behandeln

Die in Tabelle 9.111 gezeigten Funktionen liefern Informationen über eine Tabelle, für die gerade ein `table_rewrite`-Ereignis ausgelöst wurde. In jedem anderen Kontext wird ein Fehler ausgelöst.

**Tabelle 9.111. Informationsfunktionen für Tabellen-Rewrites**

| Funktion | Beschreibung |
|---|---|
| `pg_event_trigger_table_rewrite_oid ()` -> `oid` | Gibt die OID der Tabelle zurück, die gleich neu geschrieben wird. |
| `pg_event_trigger_table_rewrite_reason ()` -> `integer` | Gibt einen Code zurück, der den Grund oder die Gründe für das Neuschreiben erklärt. Der Wert ist eine Bitmap aus den folgenden Werten: 1, die Persistenz der Tabelle hat sich geändert; 2, der Standardwert einer Spalte hat sich geändert; 4, eine Spalte hat einen neuen Datentyp; und 8, die Tabellenzugriffsmethode hat sich geändert. |

Diese Funktionen können in einem Event-Trigger wie folgt verwendet werden:

```sql
CREATE FUNCTION test_event_trigger_table_rewrite_oid()
RETURNS event_trigger
LANGUAGE plpgsql AS
$$
BEGIN
   RAISE NOTICE 'rewriting table % for reason %',
                 pg_event_trigger_table_rewrite_oid()::regclass,
                 pg_event_trigger_table_rewrite_reason();
END;
$$;

CREATE EVENT TRIGGER test_table_rewrite_oid
  ON table_rewrite
  EXECUTE FUNCTION test_event_trigger_table_rewrite_oid();
```

## 9.31. Statistikinformationsfunktionen

PostgreSQL stellt eine Funktion bereit, um komplexe Statistiken zu untersuchen, die mit dem Befehl `CREATE STATISTICS` definiert wurden.

### 9.31.1. MCV-Listen untersuchen

```sql
pg_mcv_list_items ( pg_mcv_list ) -> setof record
```

`pg_mcv_list_items` gibt eine Menge von Datensätzen zurück, die alle Einträge beschreiben, die in einer mehrspaltigen MCV-Liste gespeichert sind. Die Funktion gibt die folgenden Spalten zurück:

| Name | Typ | Beschreibung |
|---|---|---|
| `index` | `integer` | Index des Eintrags in der MCV-Liste. |
| `values` | `text[]` | Im MCV-Eintrag gespeicherte Werte. |
| `nulls` | `boolean[]` | Flags, die `NULL`-Werte kennzeichnen. |
| `frequency` | `double precision` | Häufigkeit dieses MCV-Eintrags. |
| `base_frequency` | `double precision` | Basishäufigkeit dieses MCV-Eintrags. |

Die Funktion `pg_mcv_list_items` kann wie folgt verwendet werden:

```sql
SELECT m.*
FROM pg_statistic_ext
JOIN pg_statistic_ext_data ON (oid = stxoid),
     pg_mcv_list_items(stxdmcv) m
WHERE stxname = 'stts';
```

Werte des Typs `pg_mcv_list` können nur aus der Spalte `pg_statistic_ext_data.stxdmcv` gewonnen werden.
