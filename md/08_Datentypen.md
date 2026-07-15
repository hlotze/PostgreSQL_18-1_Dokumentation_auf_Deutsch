# 8. Datentypen

PostgreSQL stellt Benutzern einen umfangreichen Satz nativer Datentypen bereit. Benutzer können mit dem Befehl `CREATE TYPE` neue Typen hinzufügen.

Tabelle 8.1 zeigt alle eingebauten Allzweck-Datentypen. Die meisten alternativen Namen in der Spalte "Aliases" sind Namen, die PostgreSQL aus historischen Gründen intern verwendet. Zusätzlich sind einige intern verwendete oder veraltete Typen verfügbar, die hier nicht aufgeführt sind.

**Tabelle 8.1. Datentypen**

| Name | Aliases | Beschreibung |
|---|---|---|
| `bigint` | `int8` | Vorzeichenbehaftete 8-Byte-Ganzzahl |
| `bigserial` | `serial8` | Automatisch hochzählende 8-Byte-Ganzzahl |
| `bit [ (n) ]` |  | Bit-String fester Länge |
| `bit varying [ (n) ]` | `varbit [ (n) ]` | Bit-String variabler Länge |
| `boolean` | `bool` | Logischer boolescher Wert (`true`/`false`) |
| `box` |  | Rechteck in einer Ebene |
| `bytea` |  | Binärdaten ("Byte-Array") |
| `character [ (n) ]` | `char [ (n) ]` | Zeichenkette fester Länge |
| `character varying [ (n) ]` | `varchar [ (n) ]` | Zeichenkette variabler Länge |
| `cidr` |  | IPv4- oder IPv6-Netzwerkadresse |
| `circle` |  | Kreis in einer Ebene |
| `date` |  | Kalenderdatum (Jahr, Monat, Tag) |
| `double precision` | `float`, `float8` | Gleitkommazahl doppelter Genauigkeit (8 Byte) |
| `inet` |  | IPv4- oder IPv6-Hostadresse |
| `integer` | `int`, `int4` | Vorzeichenbehaftete 4-Byte-Ganzzahl |
| `interval [ fields ] [ (p) ]` |  | Zeitspanne |
| `json` |  | Textuelle JSON-Daten |
| `jsonb` |  | Binäre JSON-Daten, zerlegt gespeichert |
| `line` |  | Unendliche Gerade in einer Ebene |
| `lseg` |  | Liniensegment in einer Ebene |
| `macaddr` |  | MAC-Adresse (Media Access Control) |
| `macaddr8` |  | MAC-Adresse im EUI-64-Format |
| `money` |  | Währungsbetrag |
| `numeric [ (p, s) ]` | `decimal [ (p, s) ]` | Exakter numerischer Wert mit wählbarer Genauigkeit |
| `path` |  | Geometrischer Pfad in einer Ebene |
| `pg_lsn` |  | PostgreSQL Log Sequence Number |
| `pg_snapshot` |  | Snapshot von Transaktions-IDs auf Benutzerebene |
| `point` |  | Geometrischer Punkt in einer Ebene |
| `polygon` |  | Geschlossener geometrischer Pfad in einer Ebene |
| `real` | `float4` | Gleitkommazahl einfacher Genauigkeit (4 Byte) |
| `smallint` | `int2` | Vorzeichenbehaftete 2-Byte-Ganzzahl |
| `smallserial` | `serial2` | Automatisch hochzählende 2-Byte-Ganzzahl |
| `serial` | `serial4` | Automatisch hochzählende 4-Byte-Ganzzahl |
| `text` |  | Zeichenkette variabler Länge |
| `time [ (p) ] [ without time zone ]` |  | Uhrzeit ohne Zeitzone |
| `time [ (p) ] with time zone` | `timetz` | Uhrzeit mit Zeitzone |
| `timestamp [ (p) ] [ without time zone ]` |  | Datum und Uhrzeit ohne Zeitzone |
| `timestamp [ (p) ] with time zone` | `timestamptz` | Datum und Uhrzeit mit Zeitzone |
| `tsquery` |  | Textsuchabfrage |
| `tsvector` |  | Textsuchdokument |
| `txid_snapshot` |  | Snapshot von Transaktions-IDs auf Benutzerebene (veraltet; siehe `pg_snapshot`) |
| `uuid` |  | Universally Unique Identifier |
| `xml` |  | XML-Daten |

**Kompatibilität:** Die folgenden Typen oder Schreibweisen davon sind durch SQL spezifiziert: `bigint`, `bit`, `bit varying`, `boolean`, `char`, `character varying`, `character`, `varchar`, `date`, `double precision`, `integer`, `interval`, `numeric`, `decimal`, `real`, `smallint`, `time` (mit oder ohne Zeitzone), `timestamp` (mit oder ohne Zeitzone), `xml`.

Jeder Datentyp hat eine externe Darstellung, die durch seine Eingabe- und Ausgabefunktionen bestimmt wird. Viele eingebaute Typen haben naheliegende externe Formate. Einige Typen sind jedoch entweder PostgreSQL-spezifisch, etwa geometrische Pfade, oder sie besitzen mehrere mögliche Formate, wie die Datums- und Zeittypen. Manche Eingabe- und Ausgabefunktionen sind nicht invertierbar; das Ergebnis einer Ausgabefunktion kann also gegenüber der ursprünglichen Eingabe an Genauigkeit verlieren.

## 8.1. Numerische Typen

Numerische Typen bestehen aus zwei-, vier- und acht Byte großen Ganzzahlen, vier- und acht Byte großen Gleitkommazahlen sowie Dezimalzahlen mit wählbarer Genauigkeit. Tabelle 8.2 listet die verfügbaren Typen auf.

**Tabelle 8.2. Numerische Typen**

| Name | Speichergröße | Beschreibung | Bereich |
|---|---:|---|---|
| `smallint` | 2 Byte | Ganzzahl mit kleinem Wertebereich | -32768 bis +32767 |
| `integer` | 4 Byte | Typische Wahl für Ganzzahlen | -2147483648 bis +2147483647 |
| `bigint` | 8 Byte | Ganzzahl mit großem Wertebereich | -9223372036854775808 bis +9223372036854775807 |
| `decimal` | variabel | Benutzerdefinierte Genauigkeit, exakt | Bis zu 131072 Stellen vor dem Dezimalpunkt; bis zu 16383 Stellen danach |
| `numeric` | variabel | Benutzerdefinierte Genauigkeit, exakt | Bis zu 131072 Stellen vor dem Dezimalpunkt; bis zu 16383 Stellen danach |
| `real` | 4 Byte | Variable Genauigkeit, unexakt | 6 Dezimalstellen Genauigkeit |
| `double precision` | 8 Byte | Variable Genauigkeit, unexakt | 15 Dezimalstellen Genauigkeit |
| `smallserial` | 2 Byte | Kleine automatisch hochzählende Ganzzahl | 1 bis 32767 |
| `serial` | 4 Byte | Automatisch hochzählende Ganzzahl | 1 bis 2147483647 |
| `bigserial` | 8 Byte | Große automatisch hochzählende Ganzzahl | 1 bis 9223372036854775807 |

Die Syntax von Konstanten für numerische Typen ist in Abschnitt 4.1.2 beschrieben. Die numerischen Typen besitzen einen vollständigen Satz entsprechender arithmetischer Operatoren und Funktionen. Weitere Informationen finden Sie in [Kapitel 9](09_Funktionen_und_Operatoren.md). Die folgenden Abschnitte beschreiben die Typen im Detail.

### 8.1.1. Ganzzahltypen

Die Typen `smallint`, `integer` und `bigint` speichern ganze Zahlen, also Zahlen ohne Nachkommanteil, mit unterschiedlichen Wertebereichen. Versuche, Werte außerhalb des erlaubten Bereichs zu speichern, führen zu einem Fehler.

Der Typ `integer` ist die übliche Wahl, weil er das beste Gleichgewicht zwischen Wertebereich, Speichergröße und Performance bietet. Der Typ `smallint` wird im Allgemeinen nur verwendet, wenn Speicherplatz besonders knapp ist. Der Typ `bigint` ist dafür gedacht, verwendet zu werden, wenn der Wertebereich von `integer` nicht ausreicht.

SQL spezifiziert nur die Ganzzahltypen `integer` oder `int`, `smallint` und `bigint`. Die Typnamen `int2`, `int4` und `int8` sind Erweiterungen, die auch von einigen anderen SQL-Datenbanksystemen verwendet werden.

### 8.1.2. Zahlen mit beliebiger Genauigkeit

Der Typ `numeric` kann Zahlen mit sehr vielen Stellen speichern. Er wird besonders für Geldbeträge und andere Größen empfohlen, bei denen Exaktheit erforderlich ist. Berechnungen mit `numeric`-Werten liefern, wo möglich, exakte Ergebnisse, etwa bei Addition, Subtraktion und Multiplikation. Berechnungen mit `numeric`-Werten sind jedoch im Vergleich zu Ganzzahltypen oder zu den im nächsten Abschnitt beschriebenen Gleitkommatypen sehr langsam.

Im Folgenden verwenden wir diese Begriffe: Die Precision eines `numeric`-Werts ist die Gesamtzahl signifikanter Stellen in der gesamten Zahl, also die Anzahl der Stellen auf beiden Seiten des Dezimalpunkts. Die Scale eines `numeric`-Werts ist die Anzahl der Dezimalstellen im Nachkommateil, rechts vom Dezimalpunkt. Die Zahl `23.5141` hat also eine Precision von 6 und eine Scale von 4. Ganzzahlen können als Werte mit Scale 0 betrachtet werden.

Sowohl die maximale Precision als auch die maximale Scale einer `numeric`-Spalte können konfiguriert werden. Um eine Spalte des Typs `numeric` zu deklarieren, verwenden Sie die Syntax:

```sql
NUMERIC(precision, scale)
```

Die Precision muss positiv sein, während die Scale positiv oder negativ sein kann (siehe unten). Alternativ:

```sql
NUMERIC(precision)
```

wählt eine Scale von 0. Die Angabe

```sql
NUMERIC
```

ohne Precision oder Scale erstellt eine "unbeschränkte `numeric`"-Spalte, in der `numeric`-Werte beliebiger Länge bis zu den Implementierungsgrenzen gespeichert werden können. Eine solche Spalte zwingt Eingabewerte nicht auf eine bestimmte Scale, während `numeric`-Spalten mit deklarierter Scale Eingabewerte auf diese Scale bringen. Der SQL-Standard verlangt eine Standard-Scale von 0, also eine Erzwingung ganzzahliger Genauigkeit. Das halten wir für wenig nützlich. Wenn Ihnen Portabilität wichtig ist, geben Sie Precision und Scale immer ausdrücklich an.

> Hinweis: Die maximale Precision, die in einer `numeric`-Typdeklaration ausdrücklich angegeben werden kann, beträgt 1000. Eine unbeschränkte `numeric`-Spalte unterliegt den in Tabelle 8.2 beschriebenen Grenzen.

Wenn die Scale eines zu speichernden Werts größer ist als die deklarierte Scale der Spalte, rundet das System den Wert auf die angegebene Anzahl von Nachkommastellen. Wenn danach die Anzahl der Stellen links vom Dezimalpunkt die deklarierte Precision minus der deklarierten Scale überschreitet, wird ein Fehler ausgelöst. Eine als

```sql
NUMERIC(3, 1)
```

deklarierte Spalte rundet Werte beispielsweise auf eine Nachkommastelle und kann Werte zwischen `-99.9` und `99.9` einschließlich speichern.

Seit PostgreSQL 15 ist es erlaubt, eine `numeric`-Spalte mit negativer Scale zu deklarieren. Dann werden Werte links vom Dezimalpunkt gerundet. Die Precision steht weiterhin für die maximale Anzahl nicht gerundeter Stellen. Eine als

```sql
NUMERIC(2, -3)
```

deklarierte Spalte rundet Werte auf das nächste Tausend und kann Werte zwischen `-99000` und `99000` einschließlich speichern.

Es ist außerdem erlaubt, eine Scale zu deklarieren, die größer ist als die deklarierte Precision. Eine solche Spalte kann nur Bruchwerte aufnehmen und erfordert, dass die Anzahl der Nullstellen direkt rechts vom Dezimalpunkt mindestens der deklarierten Scale minus der deklarierten Precision entspricht. Eine als

```sql
NUMERIC(3, 5)
```

deklarierte Spalte rundet Werte beispielsweise auf fünf Nachkommastellen und kann Werte zwischen `-0.00999` und `0.00999` einschließlich speichern.

> Hinweis: PostgreSQL erlaubt für die Scale in einer `numeric`-Typdeklaration jeden Wert im Bereich von -1000 bis 1000. Der SQL-Standard verlangt jedoch, dass die Scale im Bereich von 0 bis Precision liegt. Die Verwendung von Scales außerhalb dieses Bereichs ist möglicherweise nicht auf andere Datenbanksysteme portierbar.

`numeric`-Werte werden physisch ohne zusätzliche führende oder nachgestellte Nullen gespeichert. Die deklarierte Precision und Scale einer Spalte sind daher Maximalwerte, keine festen Zuweisungen. In diesem Sinn ähnelt der Typ `numeric` eher `varchar(n)` als `char(n)`. Der tatsächliche Speicherbedarf beträgt zwei Byte für jede Gruppe von vier Dezimalstellen, plus drei bis acht Byte Overhead.

Zusätzlich zu gewöhnlichen numerischen Werten besitzt der Typ `numeric` mehrere Sonderwerte:

- `Infinity`
- `-Infinity`
- `NaN`

Diese sind aus dem IEEE-754-Standard übernommen und stehen für "Unendlichkeit", "negative Unendlichkeit" beziehungsweise "not-a-number". Wenn Sie diese Werte als Konstanten in einem SQL-Befehl schreiben, müssen Sie sie in Anführungszeichen setzen, zum Beispiel:

```sql
UPDATE table SET x = '-Infinity';
```

Bei der Eingabe werden diese Zeichenketten ohne Berücksichtigung der Groß-/Kleinschreibung erkannt. Die Unendlichkeitswerte können alternativ als `inf` und `-inf` geschrieben werden.

Die Unendlichkeitswerte verhalten sich entsprechend den mathematischen Erwartungen. Zum Beispiel ergibt `Infinity` plus ein beliebiger endlicher Wert wieder `Infinity`, ebenso `Infinity` plus `Infinity`; aber `Infinity` minus `Infinity` ergibt `NaN` ("not a number"), weil es dafür keine wohldefinierte Interpretation gibt. Beachten Sie, dass ein Unendlichkeitswert nur in einer unbeschränkten `numeric`-Spalte gespeichert werden kann, weil er begrifflich jede endliche Precision-Grenze überschreitet.

Der Wert `NaN` ("not a number") wird verwendet, um undefinierte Berechnungsergebnisse darzustellen. Im Allgemeinen ergibt jede Operation mit einer `NaN`-Eingabe wieder `NaN`. Die einzige Ausnahme gilt, wenn die anderen Eingaben der Operation so sind, dass dasselbe Ergebnis herauskäme, wenn `NaN` durch einen beliebigen endlichen oder unendlichen numerischen Wert ersetzt würde; dann wird dieses Ergebnis auch für `NaN` verwendet. Ein Beispiel für dieses Prinzip ist, dass `NaN` hoch null den Wert eins ergibt.

> Hinweis: In den meisten Implementierungen des "not-a-number"-Konzepts gilt `NaN` nicht als gleich irgendeinem anderen numerischen Wert, einschließlich `NaN` selbst. Damit numerische Werte sortiert und in baumbasierten Indizes verwendet werden können, behandelt PostgreSQL `NaN`-Werte als gleich und größer als alle Nicht-`NaN`-Werte.

Die Typen `decimal` und `numeric` sind gleichbedeutend. Beide Typen sind Teil des SQL-Standards.

Beim Runden von Werten rundet der Typ `numeric` Bindungsfälle von null weg, während die Typen `real` und `double precision` auf den meisten Maschinen Bindungsfälle zur nächsten geraden Zahl runden. Zum Beispiel:

```sql
SELECT x,
  round(x::numeric) AS num_round,
  round(x::double precision) AS dbl_round
FROM generate_series(-3.5, 3.5, 1) as x;
```

```text
  x   | num_round | dbl_round
------+-----------+-----------
 -3.5 |        -4 |        -4
 -2.5 |        -3 |        -2
 -1.5 |        -2 |        -2
 -0.5 |        -1 |        -0
  0.5 |         1 |         0
  1.5 |         2 |         2
  2.5 |         3 |         2
  3.5 |         4 |         4
(8 rows)
```

### 8.1.3. Gleitkommatypen

Die Datentypen `real` und `double precision` sind unexakte numerische Typen mit variabler Genauigkeit. Auf allen derzeit unterstützten Plattformen implementieren diese Typen den IEEE-Standard 754 für binäre Gleitkommaarithmetik, einfache beziehungsweise doppelte Genauigkeit, soweit Prozessor, Betriebssystem und Compiler dies unterstützen.

Unexakt bedeutet, dass manche Werte nicht exakt in das interne Format umgewandelt werden können und daher als Näherungen gespeichert werden. Beim Speichern und erneuten Abrufen eines Werts können also kleine Abweichungen sichtbar werden. Der Umgang mit solchen Fehlern und ihrer Fortpflanzung durch Berechnungen ist Gegenstand eines eigenen Zweigs von Mathematik und Informatik und wird hier nicht behandelt, abgesehen von den folgenden Punkten:

- Wenn Sie exakte Speicherung und exakte Berechnungen benötigen, zum Beispiel für Geldbeträge, verwenden Sie stattdessen den Typ `numeric`.

- Wenn Sie mit diesen Typen komplizierte Berechnungen für wichtige Zwecke durchführen wollen, insbesondere wenn Sie auf ein bestimmtes Verhalten in Grenzfällen wie Unendlichkeit oder Underflow angewiesen sind, sollten Sie die Implementierung sorgfältig prüfen.

- Der Vergleich zweier Gleitkommawerte auf Gleichheit funktioniert möglicherweise nicht immer wie erwartet.

Auf allen derzeit unterstützten Plattformen hat der Typ `real` einen Bereich von ungefähr `1E-37` bis `1E+37` bei einer Genauigkeit von mindestens 6 Dezimalstellen. Der Typ `double precision` hat einen Bereich von ungefähr `1E-307` bis `1E+308` bei einer Genauigkeit von mindestens 15 Stellen. Zu große oder zu kleine Werte führen zu einem Fehler. Wenn die Genauigkeit einer Eingabezahl zu hoch ist, kann gerundet werden. Zahlen, die zu nahe an null liegen und nicht als von null verschieden darstellbar sind, führen zu einem Underflow-Fehler.

Standardmäßig werden Gleitkommawerte in Textform in ihrer kürzesten präzisen Dezimaldarstellung ausgegeben; der erzeugte Dezimalwert liegt näher am tatsächlich gespeicherten Binärwert als jeder andere Wert, der mit derselben binären Genauigkeit darstellbar ist. Der Ausgabewert liegt jedoch derzeit niemals exakt in der Mitte zwischen zwei darstellbaren Werten, um einen verbreiteten Fehler zu vermeiden, bei dem Eingaberoutinen die Regel "round to nearest even" nicht korrekt beachten. Diese Darstellung verwendet höchstens 17 signifikante Dezimalstellen für `float8`-Werte und höchstens 9 Stellen für `float4`-Werte.

> Hinweis: Dieses kürzeste präzise Ausgabeformat lässt sich viel schneller erzeugen als das historische gerundete Format.

Zur Kompatibilität mit Ausgaben älterer PostgreSQL-Versionen und um die Ausgabepräzision zu verringern, kann der Parameter `extra_float_digits` verwendet werden, um stattdessen gerundete Dezimalausgabe auszuwählen. Der Wert `0` stellt die frühere Vorgabe wieder her, bei der der Wert auf 6 signifikante Dezimalstellen für `float4` oder 15 für `float8` gerundet wurde. Ein negativer Wert reduziert die Anzahl der Stellen weiter; zum Beispiel würde `-2` die Ausgabe auf 4 beziehungsweise 13 Stellen runden.

Jeder Wert von `extra_float_digits` größer als 0 wählt das kürzeste präzise Format.

> Hinweis: Anwendungen, die präzise Werte benötigten, mussten historisch `extra_float_digits` auf `3` setzen, um sie zu erhalten. Für maximale Kompatibilität zwischen Versionen sollten sie dies weiterhin tun.

Zusätzlich zu gewöhnlichen numerischen Werten haben die Gleitkommatypen mehrere besondere Werte:

```text
Infinity
-Infinity
NaN
```

Diese stehen für die IEEE-754-Sonderwerte "Unendlichkeit", "negative Unendlichkeit" und "not-a-number". Wenn Sie diese Werte als Konstanten in einem SQL-Befehl schreiben, müssen Sie sie in Anführungszeichen setzen, zum Beispiel:

```sql
UPDATE table SET x = '-Infinity';
```

Bei der Eingabe werden diese Zeichenketten ohne Beachtung der Groß-/Kleinschreibung erkannt. Die Unendlichkeitswerte können alternativ als `inf` und `-inf` geschrieben werden.

> Hinweis: IEEE 754 schreibt vor, dass `NaN` mit keinem anderen Gleitkommawert gleich vergleichen soll, auch nicht mit `NaN`. Damit Gleitkommawerte sortiert und in baumbasierten Indizes verwendet werden können, behandelt PostgreSQL `NaN`-Werte als gleich und größer als alle Nicht-`NaN`-Werte.

PostgreSQL unterstützt außerdem die SQL-Standardschreibweisen `float` und `float(p)` zur Angabe unexakter numerischer Typen. Hier gibt `p` die mindestens akzeptable Genauigkeit in Binärstellen an. PostgreSQL akzeptiert `float(1)` bis `float(24)` als Auswahl des Typs `real`, während `float(25)` bis `float(53)` `double precision` auswählen. Werte von `p` außerhalb des zulässigen Bereichs führen zu einem Fehler. `float` ohne angegebene Genauigkeit bedeutet `double precision`.

### 8.1.4. Serial-Typen

> Hinweis: Dieser Abschnitt beschreibt eine PostgreSQL-spezifische Möglichkeit, eine automatisch hochzählende Spalte zu erzeugen. Eine andere Möglichkeit ist die SQL-Standardfunktion für Identitätsspalten, beschrieben in [Abschnitt 5.3](05_Datendefinition.md#53-identitätsspalten).

Die Datentypen `smallserial`, `serial` und `bigserial` sind keine echten Typen, sondern lediglich eine Schreibvereinfachung zum Erzeugen eindeutiger Bezeichnerspalten, ähnlich der von manchen anderen Datenbanken unterstützten Eigenschaft `AUTO_INCREMENT`. In der aktuellen Implementierung entspricht die Angabe

```sql
CREATE TABLE tablename (
    colname SERIAL
);
```

der Angabe

```sql
CREATE SEQUENCE tablename_colname_seq AS integer;
CREATE TABLE tablename (
    colname integer NOT NULL DEFAULT
        nextval('tablename_colname_seq')
);
ALTER SEQUENCE tablename_colname_seq OWNED BY tablename.colname;
```

Damit wurde eine `integer`-Spalte erzeugt und so eingerichtet, dass ihre Standardwerte aus einem Sequenzgenerator zugewiesen werden. Ein `NOT NULL`-Constraint stellt sicher, dass kein Nullwert eingefügt werden kann. In den meisten Fällen sollten Sie zusätzlich einen `UNIQUE`- oder `PRIMARY KEY`-Constraint hinzufügen, um zu verhindern, dass versehentlich doppelte Werte eingefügt werden; automatisch geschieht das jedoch nicht. Schließlich wird die Sequenz als "owned by" der Spalte markiert, sodass sie gelöscht wird, wenn die Spalte oder Tabelle gelöscht wird.

> Hinweis: Weil `smallserial`, `serial` und `bigserial` mithilfe von Sequenzen implementiert werden, kann es "Löcher" oder Lücken in der Wertfolge geben, die in der Spalte erscheint, selbst wenn niemals Zeilen gelöscht werden. Ein aus der Sequenz zugewiesener Wert ist weiterhin "verbraucht", auch wenn eine Zeile mit diesem Wert nie erfolgreich in die Tabellenspalte eingefügt wird. Das kann zum Beispiel passieren, wenn die einfügende Transaktion zurückgerollt wird. Details finden Sie bei `nextval()` in [Abschnitt 9.17](09_Funktionen_und_Operatoren.md#917-sequenzmanipulationsfunktionen).

Um den nächsten Wert der Sequenz in die `serial`-Spalte einzufügen, geben Sie an, dass der `serial`-Spalte ihr Standardwert zugewiesen werden soll. Das kann entweder dadurch geschehen, dass die Spalte in der Spaltenliste der `INSERT`-Anweisung ausgelassen wird, oder durch Verwendung des Schlüsselworts `DEFAULT`.

Die Typnamen `serial` und `serial4` sind gleichbedeutend: Beide erzeugen `integer`-Spalten. Die Typnamen `bigserial` und `serial8` funktionieren genauso, erzeugen aber eine `bigint`-Spalte. `bigserial` sollte verwendet werden, wenn Sie erwarten, während der Lebensdauer der Tabelle mehr als 2^31 Bezeichner zu verwenden. Die Typnamen `smallserial` und `serial2` funktionieren ebenfalls gleich, erzeugen aber eine `smallint`-Spalte.

Die für eine `serial`-Spalte erzeugte Sequenz wird automatisch gelöscht, wenn die besitzende Spalte gelöscht wird. Sie können die Sequenz löschen, ohne die Spalte zu löschen, dies erzwingt jedoch das Entfernen des Standardausdrucks der Spalte.

## 8.2. Geldtypen

Der Typ `money` speichert einen Währungsbetrag mit fester Nachkommagenauigkeit; siehe Tabelle 8.3. Die Nachkommagenauigkeit wird durch die Datenbankeinstellung `lc_monetary` bestimmt. Der in der Tabelle gezeigte Bereich setzt zwei Nachkommastellen voraus. Eingaben werden in verschiedenen Formaten akzeptiert, darunter Ganzzahl- und Gleitkommaliterale sowie typische Währungsformatierungen wie `'$1,000.00'`. Die Ausgabe erfolgt im Allgemeinen in letzterer Form, hängt aber von der Locale ab.

**Tabelle 8.3. Geldtypen**

| Name | Speichergröße | Beschreibung | Bereich |
|---|---:|---|---|
| `money` | 8 Byte | Währungsbetrag | -92233720368547758.08 bis +92233720368547758.07 |

Da die Ausgabe dieses Datentyps locale-abhängig ist, funktioniert das Laden von `money`-Daten in eine Datenbank mit anderer Einstellung von `lc_monetary` möglicherweise nicht. Um Probleme zu vermeiden, stellen Sie vor dem Wiederherstellen eines Dumps in eine neue Datenbank sicher, dass `lc_monetary` denselben oder einen gleichwertigen Wert hat wie in der Datenbank, aus der der Dump stammt.

Werte der Datentypen `numeric`, `int` und `bigint` können nach `money` gecastet werden. Eine Umwandlung aus den Datentypen `real` und `double precision` kann erfolgen, indem zuerst nach `numeric` gecastet wird, zum Beispiel:

```sql
SELECT '12.34'::float8::numeric::money;
```

Dies wird jedoch nicht empfohlen. Gleitkommazahlen sollten wegen möglicher Rundungsfehler nicht für Geldbeträge verwendet werden.

Ein `money`-Wert kann ohne Genauigkeitsverlust nach `numeric` gecastet werden. Eine Umwandlung in andere Typen kann Genauigkeit verlieren und muss ebenfalls in zwei Schritten erfolgen:

```sql
SELECT '52093.89'::money::numeric::float8;
```

Die Division eines `money`-Werts durch einen ganzzahligen Wert wird mit Abschneiden des Nachkommateils in Richtung null durchgeführt. Um ein gerundetes Ergebnis zu erhalten, teilen Sie durch einen Gleitkommawert oder casten den `money`-Wert vor der Division nach `numeric` und anschließend wieder zurück nach `money`. Letzteres ist vorzuziehen, um keinen Genauigkeitsverlust zu riskieren. Wenn ein `money`-Wert durch einen anderen `money`-Wert geteilt wird, ist das Ergebnis `double precision`, also eine reine Zahl und kein Geldbetrag; die Währungseinheiten kürzen sich bei der Division heraus.

## 8.3. Zeichentypen

Tabelle 8.4 zeigt die in PostgreSQL verfügbaren allgemeinen Zeichentypen.

**Tabelle 8.4. Zeichentypen**

| Name | Beschreibung |
|---|---|
| `character varying(n)`, `varchar(n)` | Variable Länge mit Begrenzung |
| `character(n)`, `char(n)`, `bpchar(n)` | Feste Länge, mit Leerzeichen aufgefüllt |
| `bpchar` | Variable unbegrenzte Länge, nachgestellte Leerzeichen werden entfernt |
| `text` | Variable unbegrenzte Länge |

SQL definiert zwei primäre Zeichentypen: `character varying(n)` und `character(n)`, wobei `n` eine positive Ganzzahl ist. Beide Typen können Zeichenketten mit bis zu `n` Zeichen speichern, nicht Bytes. Der Versuch, eine längere Zeichenkette in eine Spalte dieser Typen zu speichern, führt zu einem Fehler, außer die überschüssigen Zeichen sind ausschließlich Leerzeichen; in diesem Fall wird die Zeichenkette auf die maximale Länge gekürzt. Diese etwas merkwürdige Ausnahme wird vom SQL-Standard verlangt. Wenn ein Wert jedoch ausdrücklich nach `character varying(n)` oder `character(n)` gecastet wird, wird ein zu langer Wert ohne Fehler auf `n` Zeichen gekürzt. Auch das wird vom SQL-Standard verlangt. Ist die zu speichernde Zeichenkette kürzer als die deklarierte Länge, werden Werte des Typs `character` mit Leerzeichen aufgefüllt; Werte des Typs `character varying` speichern einfach die kürzere Zeichenkette.

Zusätzlich stellt PostgreSQL den Typ `text` bereit, der Zeichenketten beliebiger Länge speichert. Obwohl der Typ `text` nicht im SQL-Standard enthalten ist, existiert er auch in mehreren anderen SQL-Datenbankmanagementsystemen. `text` ist der native Zeichenkettentyp von PostgreSQL: Die meisten eingebauten Funktionen, die mit Zeichenketten arbeiten, sind so deklariert, dass sie `text` entgegennehmen oder zurückgeben, nicht `character varying`. Für viele Zwecke verhält sich `character varying` wie eine Domain über `text`.

Der Typname `varchar` ist ein Alias für `character varying`, während `bpchar` mit Längenangabe und `char` Aliase für `character` sind. Die Aliase `varchar` und `char` sind im SQL-Standard definiert; `bpchar` ist eine PostgreSQL-Erweiterung.

Wenn eine Länge `n` angegeben wird, muss sie größer als null sein und darf `10,485,760` nicht überschreiten. Wird `character varying` oder `varchar` ohne Längenangabe verwendet, akzeptiert der Typ Zeichenketten beliebiger Länge. Fehlt bei `bpchar` eine Längenangabe, akzeptiert auch dieser Typ Zeichenketten beliebiger Länge, aber nachgestellte Leerzeichen sind semantisch bedeutungslos. Fehlt bei `character` oder `char` die Angabe, ist der Typ gleichbedeutend mit `character(1)`.

Werte des Typs `character` werden physisch bis zur angegebenen Breite `n` mit Leerzeichen aufgefüllt und auch so gespeichert und angezeigt. Nachgestellte Leerzeichen werden beim Vergleich zweier Werte des Typs `character` jedoch als semantisch bedeutungslos behandelt und ignoriert. In Collations, in denen Leerraum signifikant ist, kann dieses Verhalten unerwartete Ergebnisse erzeugen; zum Beispiel liefert

```sql
SELECT 'a '::CHAR(2) collate "C" < E'a\n'::CHAR(2);
```

`true`, obwohl die C-Locale ein Leerzeichen als größer als einen Zeilenumbruch betrachten würde. Nachgestellte Leerzeichen werden entfernt, wenn ein `character`-Wert in einen der anderen Zeichenkettentypen umgewandelt wird. Beachten Sie, dass nachgestellte Leerzeichen in Werten von `character varying` und `text` semantisch bedeutsam sind, ebenso bei Mustervergleichen, also `LIKE` und regulären Ausdrücken.

Welche Zeichen in einem dieser Datentypen gespeichert werden können, wird durch den Zeichensatz der Datenbank bestimmt, der beim Erstellen der Datenbank ausgewählt wird. Unabhängig vom konkreten Zeichensatz kann das Zeichen mit Code null, manchmal `NUL` genannt, nicht gespeichert werden. Weitere Informationen finden Sie in [Abschnitt 23.3](23_Lokalisierung.md#233-zeichensatzunterstützung).

Der Speicherbedarf für eine kurze Zeichenkette bis zu 126 Byte beträgt 1 Byte plus die eigentliche Zeichenkette, einschließlich der Leerzeichenauffüllung bei `character`. Längere Zeichenketten haben statt 1 Byte einen Overhead von 4 Byte. Lange Zeichenketten werden vom System automatisch komprimiert, sodass der physische Bedarf auf der Platte geringer sein kann. Sehr lange Werte werden außerdem in Hintergrundtabellen gespeichert, damit sie den schnellen Zugriff auf kürzere Spaltenwerte nicht behindern. In jedem Fall ist die längste speicherbare Zeichenkette etwa 1 GB groß. Der maximal zulässige Wert für `n` in der Datentypdeklaration ist kleiner als das. Es wäre nicht nützlich, dies zu ändern, weil sich bei Multibyte-Zeichencodierungen die Anzahl der Zeichen und Bytes stark unterscheiden kann. Wenn Sie lange Zeichenketten ohne feste Obergrenze speichern möchten, verwenden Sie `text` oder `character varying` ohne Längenangabe, statt eine willkürliche Längengrenze zu erfinden.

> Tipp: Zwischen diesen drei Typen gibt es keinen Performance-Unterschied, abgesehen vom erhöhten Speicherbedarf beim leerzeichenaufgefüllten Typ und einigen zusätzlichen CPU-Zyklen zur Längenprüfung beim Speichern in eine längenbeschränkte Spalte. Während `character(n)` in manchen anderen Datenbanksystemen Performance-Vorteile hat, gibt es in PostgreSQL keinen solchen Vorteil; tatsächlich ist `character(n)` wegen der zusätzlichen Speicherkosten normalerweise der langsamste der drei. In den meisten Situationen sollten stattdessen `text` oder `character varying` verwendet werden.

Informationen zur Syntax von Zeichenkettenliteralen finden Sie in [Abschnitt 4.1.2.1](04_SQL_Syntax.md#4121-zeichenkettenkonstanten), Informationen zu verfügbaren Operatoren und Funktionen in [Kapitel 9](09_Funktionen_und_Operatoren.md).

**Beispiel 8.1. Verwendung der Zeichentypen**

```sql
CREATE TABLE test1 (a character(4));
INSERT INTO test1 VALUES ('ok');
SELECT a, char_length(a) FROM test1;
```

```text
  a   | char_length
------+-------------
 ok   |           2
```

```sql
CREATE TABLE test2 (b varchar(5));
INSERT INTO test2 VALUES ('ok');
INSERT INTO test2 VALUES ('good      ');
INSERT INTO test2 VALUES ('too long');
```

```text
ERROR: value too long for type character varying(5)
```

```sql
INSERT INTO test2 VALUES ('too long'::varchar(5)); -- explicit truncation
SELECT b, char_length(b) FROM test2;
```

```text
   b  | char_length
------+-------------
 ok   |           2
 good |           5
 too l |           5
```

Die Funktion `char_length` wird in [Abschnitt 9.4](09_Funktionen_und_Operatoren.md#94-zeichenkettenfunktionen-und-operatoren) besprochen.

Es gibt in PostgreSQL zwei weitere Zeichentypen fester Länge, die in Tabelle 8.5 gezeigt werden. Sie sind nicht für allgemeine Verwendung gedacht, sondern nur für die internen Systemkataloge. Der Typ `name` wird zum Speichern von Bezeichnern verwendet. Seine Länge ist derzeit als 64 Byte definiert, also 63 nutzbare Zeichen plus Terminator, sollte im C-Quellcode aber über die Konstante `NAMEDATALEN` referenziert werden. Die Länge wird zur Kompilierzeit festgelegt und ist daher für Sonderverwendungen anpassbar; die standardmäßige Maximallänge könnte sich in einer zukünftigen Version ändern. Der Typ `"char"` (beachten Sie die Anführungszeichen) unterscheidet sich von `char(1)`, weil er nur ein Byte Speicher verwendet und daher nur ein einzelnes ASCII-Zeichen speichern kann. Er wird in den Systemkatalogen als einfacher Enumerationstyp verwendet.

**Tabelle 8.5. Spezielle Zeichentypen**

| Name | Speichergröße | Beschreibung |
|---|---:|---|
| `"char"` | 1 Byte | Interner Ein-Byte-Typ |
| `name` | 64 Byte | Interner Typ für Objektnamen |

## 8.4. Binärdatentypen

Der Datentyp `bytea` erlaubt das Speichern binärer Zeichenketten; siehe Tabelle 8.6.

**Tabelle 8.6. Binärdatentypen**

| Name | Speichergröße | Beschreibung |
|---|---:|---|
| `bytea` | 1 oder 4 Byte plus die eigentliche binäre Zeichenkette | Binäre Zeichenkette variabler Länge |

Eine binäre Zeichenkette ist eine Folge von Oktetten oder Bytes. Binäre Zeichenketten unterscheiden sich von Zeichenketten in zwei Punkten. Erstens erlauben binäre Zeichenketten ausdrücklich das Speichern von Oktetten mit dem Wert null und anderen "nicht druckbaren" Oktetten, üblicherweise Oktetten außerhalb des Dezimalbereichs 32 bis 126. Zeichenketten verbieten Null-Oktette und außerdem alle anderen Oktettwerte oder Folgen von Oktettwerten, die nach der für die Datenbank gewählten Zeichencodierung ungültig sind. Zweitens verarbeiten Operationen auf binären Zeichenketten die tatsächlichen Bytes, während die Verarbeitung von Zeichenketten von Locale-Einstellungen abhängt. Kurz gesagt: Binäre Zeichenketten eignen sich zum Speichern von Daten, die der Programmierer als "rohe Bytes" betrachtet, während Zeichenketten zum Speichern von Text geeignet sind.

Der Typ `bytea` unterstützt zwei Formate für Eingabe und Ausgabe: das "Hex"-Format und das historische PostgreSQL-"Escape"-Format. Beide werden bei der Eingabe immer akzeptiert. Das Ausgabeformat hängt vom Konfigurationsparameter `bytea_output` ab; die Vorgabe ist `hex`. Beachten Sie, dass das Hex-Format in PostgreSQL 9.0 eingeführt wurde; ältere Versionen und manche Werkzeuge verstehen es nicht.

Der SQL-Standard definiert einen anderen Binärzeichentyp namens `BLOB` oder `BINARY LARGE OBJECT`. Das Eingabeformat unterscheidet sich von `bytea`, aber die bereitgestellten Funktionen und Operatoren sind größtenteils dieselben.

### 8.4.1. `bytea`-Hexformat

Das "Hex"-Format codiert binäre Daten als zwei hexadezimale Ziffern pro Byte, mit dem höchstwertigen Nibble zuerst. Der gesamten Zeichenkette wird die Sequenz `\x` vorangestellt, um sie vom Escape-Format zu unterscheiden. In manchen Kontexten muss der anfängliche Backslash durch Verdopplung escaped werden (siehe [Abschnitt 4.1.2.1](04_SQL_Syntax.md#4121-zeichenkettenkonstanten)). Bei der Eingabe können die hexadezimalen Ziffern groß- oder kleingeschrieben sein, und Leerraum zwischen Ziffernpaaren ist erlaubt, aber nicht innerhalb eines Ziffernpaars und nicht in der anfänglichen Sequenz `\x`. Das Hex-Format ist mit einer großen Bandbreite externer Anwendungen und Protokolle kompatibel und wird tendenziell schneller umgewandelt als das Escape-Format; seine Verwendung wird daher bevorzugt.

Beispiel:

```sql
SET bytea_output = 'hex';

SELECT '\xDEADBEEF'::bytea;
```

```text
   bytea
------------
 \xdeadbeef
```

### 8.4.2. `bytea`-Escape-Format

Das "Escape"-Format ist das traditionelle PostgreSQL-Format für den Typ `bytea`. Es stellt eine binäre Zeichenkette als Folge von ASCII-Zeichen dar und wandelt Bytes, die nicht als ASCII-Zeichen dargestellt werden können, in besondere Escape-Sequenzen um. Wenn es aus Sicht der Anwendung sinnvoll ist, Bytes als Zeichen darzustellen, kann diese Darstellung praktisch sein.

In der Praxis ist sie jedoch meist verwirrend, weil sie die Unterscheidung zwischen binären Zeichenketten und Zeichenketten verwischt; außerdem ist der gewählte Escape-Mechanismus etwas umständlich. Dieses Format sollte daher für die meisten neuen Anwendungen eher vermieden werden.

Bei der Eingabe von `bytea`-Werten im Escape-Format müssen Oktette bestimmter Werte escaped werden, während alle Oktettwerte escaped werden können. Im Allgemeinen wird ein Oktett escaped, indem es in seinen dreistelligen Oktalwert umgewandelt und mit einem vorangestellten Backslash geschrieben wird. Der Backslash selbst, Oktett-Dezimalwert 92, kann alternativ durch zwei Backslashes dargestellt werden. Tabelle 8.7 zeigt die Zeichen, die escaped werden müssen, und nennt gegebenenfalls alternative Escape-Sequenzen.

**Tabelle 8.7. Escaped-Oktette in `bytea`-Literalen**

| Dezimaler Oktettwert | Beschreibung | Escaped-Eingabedarstellung | Beispiel | Hex-Darstellung |
|---:|---|---|---|---|
| 0 | Null-Oktett | `'\000'` | `'\000'::bytea` | `\x00` |
| 39 | Einfaches Anführungszeichen | `''''` oder `'\047'` | `''''::bytea` | `\x27` |
| 92 | Backslash | `'\\'` oder `'\134'` | `'\\'::bytea` | `\x5c` |
| 0 bis 31 und 127 bis 255 | "Nicht druckbare" Oktette | `'\xxx'` (Oktalwert) | `'\001'::bytea` | `\x01` |

Die Notwendigkeit, nicht druckbare Oktette zu escapen, variiert je nach Locale-Einstellungen. In manchen Fällen kommt man damit davon, sie nicht zu escapen.

Der Grund dafür, dass einfache Anführungszeichen wie in Tabelle 8.7 verdoppelt werden müssen, ist, dass dies für jedes Zeichenkettenliteral in einem SQL-Befehl gilt. Der allgemeine Parser für Zeichenkettenliterale verbraucht die äußersten einfachen Anführungszeichen und reduziert jedes Paar einfacher Anführungszeichen auf ein Datenzeichen. Die Eingabefunktion von `bytea` sieht also nur ein einzelnes einfaches Anführungszeichen und behandelt es als gewöhnliches Datenzeichen. Backslashes behandelt die Eingabefunktion von `bytea` dagegen als besonders; die übrigen in Tabelle 8.7 gezeigten Verhaltensweisen werden von dieser Funktion implementiert.

In manchen Kontexten müssen Backslashes gegenüber der oben gezeigten Form verdoppelt werden, weil der allgemeine Parser für Zeichenkettenliterale auch Paare von Backslashes auf ein Datenzeichen reduziert; siehe [Abschnitt 4.1.2.1](04_SQL_Syntax.md#4121-zeichenkettenkonstanten).

`bytea`-Oktette werden standardmäßig im Hex-Format ausgegeben. Wenn Sie `bytea_output` auf `escape` ändern, werden "nicht druckbare" Oktette in ihren entsprechenden dreistelligen Oktalwert umgewandelt und mit einem Backslash vorangestellt. Die meisten "druckbaren" Oktette werden in ihrer Standarddarstellung im Client-Zeichensatz ausgegeben, zum Beispiel:

```sql
SET bytea_output = 'escape';

SELECT 'abc \153\154\155 \052\251\124'::bytea;
```

```text
     bytea
----------------
 abc klm *\251T
```

Das Oktett mit dem Dezimalwert 92, also der Backslash, wird in der Ausgabe verdoppelt. Details stehen in Tabelle 8.8.

**Tabelle 8.8. Escaped-Oktette in der `bytea`-Ausgabe**

| Dezimaler Oktettwert | Beschreibung | Escaped-Ausgabedarstellung | Beispiel | Ausgabeergebnis |
|---:|---|---|---|---|
| 92 | Backslash | `\\` | `'\134'::bytea` | `\\` |
| 0 bis 31 und 127 bis 255 | "Nicht druckbare" Oktette | `\xxx` (Oktalwert) | `'\001'::bytea` | `\001` |
| 32 bis 126 | "Druckbare" Oktette | Darstellung im Client-Zeichensatz | `'\176'::bytea` | `~` |

Je nach verwendetem Frontend zu PostgreSQL müssen Sie beim Escapen und Unescapen von `bytea`-Zeichenketten möglicherweise zusätzliche Arbeit leisten. Zum Beispiel müssen Sie eventuell auch Zeilenvorschübe und Wagenrückläufe escapen, wenn Ihre Schnittstelle diese automatisch übersetzt.

## 8.5. Datums-/Zeittypen

PostgreSQL unterstützt die vollständige Menge der SQL-Datums- und Zeittypen, die in Tabelle 8.9 gezeigt werden. Die für diese Datentypen verfügbaren Operationen werden in [Abschnitt 9.9](09_Funktionen_und_Operatoren.md#99-datumszeitfunktionen-und-operatoren) beschrieben. Datumswerte werden nach dem gregorianischen Kalender gezählt, auch in Jahren, bevor dieser Kalender eingeführt wurde; weitere Informationen finden Sie in Abschnitt B.6.

**Tabelle 8.9. Datums-/Zeittypen**

| Name | Speichergröße | Beschreibung | Kleinster Wert | Größter Wert | Auflösung |
|---|---:|---|---|---|---|
| `timestamp [ (p) ] [ without time zone ]` | 8 Byte | Datum und Uhrzeit, ohne Zeitzone | 4713 BC | 294276 AD | 1 Mikrosekunde |
| `timestamp [ (p) ] with time zone` | 8 Byte | Datum und Uhrzeit, mit Zeitzone | 4713 BC | 294276 AD | 1 Mikrosekunde |
| `date` | 4 Byte | Datum, ohne Tageszeit | 4713 BC | 5874897 AD | 1 Tag |
| `time [ (p) ] [ without time zone ]` | 8 Byte | Tageszeit, ohne Datum | 00:00:00 | 24:00:00 | 1 Mikrosekunde |
| `time [ (p) ] with time zone` | 12 Byte | Tageszeit, ohne Datum, mit Zeitzone | 00:00:00+1559 | 24:00:00-1559 | 1 Mikrosekunde |
| `interval [ fields ] [ (p) ]` | 16 Byte | Zeitintervall | -178000000 Jahre | 178000000 Jahre | 1 Mikrosekunde |

> Hinweis: Der SQL-Standard verlangt, dass die Schreibweise `timestamp` allein gleichbedeutend mit `timestamp without time zone` ist, und PostgreSQL hält sich an dieses Verhalten. `timestamptz` wird als Abkürzung für `timestamp with time zone` akzeptiert; dies ist eine PostgreSQL-Erweiterung.

`time`, `timestamp` und `interval` akzeptieren einen optionalen Precision-Wert `p`, der die Anzahl der Nachkommastellen im Sekundenfeld angibt. Standardmäßig gibt es keine ausdrückliche Begrenzung der Precision. Der zulässige Bereich von `p` reicht von 0 bis 6.

Der Typ `interval` hat eine zusätzliche Option, mit der die Menge der gespeicherten Felder eingeschränkt werden kann. Dazu wird eine der folgenden Phrasen geschrieben:

```text
YEAR
MONTH
DAY
HOUR
MINUTE
SECOND
YEAR TO MONTH
DAY TO HOUR
DAY TO MINUTE
DAY TO SECOND
HOUR TO MINUTE
HOUR TO SECOND
MINUTE TO SECOND
```

Beachten Sie: Wenn sowohl `fields` als auch `p` angegeben werden, müssen die Felder `SECOND` enthalten, da die Precision nur für Sekunden gilt.

Der Typ `time with time zone` ist im SQL-Standard definiert, aber seine Definition hat Eigenschaften, die seinen Nutzen fragwürdig machen. In den meisten Fällen sollte eine Kombination aus `date`, `time`, `timestamp without time zone` und `timestamp with time zone` den vollständigen Umfang an Datums-/Zeitfunktionalität bereitstellen, den eine Anwendung benötigt.

### 8.5.1. Datums-/Zeiteingabe

Datums- und Zeiteingaben werden in nahezu jedem vernünftigen Format akzeptiert, darunter ISO 8601, SQL-kompatible Formate, traditionelles POSTGRES und andere. Bei manchen Formaten ist die Reihenfolge von Tag, Monat und Jahr in Datumseingaben mehrdeutig; PostgreSQL unterstützt daher die Angabe der erwarteten Reihenfolge dieser Felder. Setzen Sie den Parameter `DateStyle` auf `MDY`, um Monat-Tag-Jahr-Interpretation zu wählen, auf `DMY` für Tag-Monat-Jahr oder auf `YMD` für Jahr-Monat-Tag.

PostgreSQL ist bei der Verarbeitung von Datums-/Zeiteingaben flexibler, als es der SQL-Standard verlangt. Anhang B enthält die genauen Parseregeln für Datums-/Zeiteingaben und die erkannten Textfelder, darunter Monate, Wochentage und Zeitzonen.

Denken Sie daran, dass jede Datums- oder Zeitliteral-Eingabe wie eine Textzeichenkette in einfache Anführungszeichen gesetzt werden muss. Weitere Informationen finden Sie in [Abschnitt 4.1.2.7](04_SQL_Syntax.md#4127-konstanten-anderer-typen). SQL verlangt die folgende Syntax:

```text
type [ (p) ] 'value'
```

Dabei ist `p` eine optionale Precision-Angabe, welche die Anzahl der Nachkommastellen im Sekundenfeld festlegt. Precision kann für die Typen `time`, `timestamp` und `interval` angegeben werden und reicht von 0 bis 6. Wenn in einer Konstantenspezifikation keine Precision angegeben wird, übernimmt sie standardmäßig die Precision des Literalwerts, höchstens jedoch 6 Stellen.

#### 8.5.1.1. Datumswerte

Tabelle 8.10 zeigt einige mögliche Eingaben für den Typ `date`.

**Tabelle 8.10. Datumseingabe**

| Beispiel | Beschreibung |
|---|---|
| `1999-01-08` | ISO 8601; 8. Januar in jedem Modus, empfohlenes Format |
| `January 8, 1999` | In jedem `datestyle`-Eingabemodus eindeutig |
| `1/8/1999` | 8. Januar im MDY-Modus; 1. August im DMY-Modus |
| `1/18/1999` | 18. Januar im MDY-Modus; in anderen Modi abgewiesen |
| `01/02/03` | 2. Januar 2003 im MDY-Modus; 1. Februar 2003 im DMY-Modus; 3. Februar 2001 im YMD-Modus |
| `1999-Jan-08` | 8. Januar in jedem Modus |
| `Jan-08-1999` | 8. Januar in jedem Modus |
| `08-Jan-1999` | 8. Januar in jedem Modus |
| `99-Jan-08` | 8. Januar im YMD-Modus, sonst Fehler |
| `08-Jan-99` | 8. Januar, außer Fehler im YMD-Modus |
| `Jan-08-99` | 8. Januar, außer Fehler im YMD-Modus |
| `19990108` | ISO 8601; 8. Januar 1999 in jedem Modus |
| `990108` | ISO 8601; 8. Januar 1999 in jedem Modus |
| `1999.008` | Jahr und Tag des Jahres |
| `J2451187` | Julianisches Datum |
| `January 8, 99 BC` | Jahr 99 v. Chr. |

#### 8.5.1.2. Uhrzeiten

Die Tageszeittypen sind `time [ (p) ] without time zone` und `time [ (p) ] with time zone`. `time` allein ist gleichbedeutend mit `time without time zone`.

Gültige Eingaben für diese Typen bestehen aus einer Tageszeit, gefolgt von einer optionalen Zeitzone; siehe Tabelle 8.11 und Tabelle 8.12. Wenn bei der Eingabe für `time without time zone` eine Zeitzone angegeben wird, wird sie stillschweigend ignoriert. Sie können auch ein Datum angeben, aber es wird ebenfalls ignoriert, außer wenn Sie einen Zeitzonennamen verwenden, der eine Sommerzeitregel enthält, zum Beispiel `America/New_York`. In diesem Fall ist das Datum erforderlich, um zu bestimmen, ob Standardzeit oder Sommerzeit gilt. Der passende Zeitzonen-Offset wird im Wert `time with time zone` aufgezeichnet und so ausgegeben, wie er gespeichert wurde; er wird nicht an die aktive Zeitzone angepasst.

**Tabelle 8.11. Zeiteingabe**

| Beispiel | Beschreibung |
|---|---|
| `04:05:06.789` | ISO 8601 |
| `04:05:06` | ISO 8601 |
| `04:05` | ISO 8601 |
| `040506` | ISO 8601 |
| `04:05 AM` | Dasselbe wie `04:05`; `AM` beeinflusst den Wert nicht |
| `04:05 PM` | Dasselbe wie `16:05`; Eingabestunde muss `<= 12` sein |
| `04:05:06.789-8` | ISO 8601, mit Zeitzone als UTC-Offset |
| `04:05:06-08:00` | ISO 8601, mit Zeitzone als UTC-Offset |
| `04:05-08:00` | ISO 8601, mit Zeitzone als UTC-Offset |
| `040506-08` | ISO 8601, mit Zeitzone als UTC-Offset |
| `040506+0730` | ISO 8601, mit Zeitzone mit Bruchteilstunde als UTC-Offset |
| `040506+07:30:00` | UTC-Offset bis auf Sekunden angegeben, in ISO 8601 nicht erlaubt |
| `04:05:06 PST` | Zeitzone durch Abkürzung angegeben |
| `2003-04-12 04:05:06 America/New_York` | Zeitzone durch vollständigen Namen angegeben |

**Tabelle 8.12. Zeitzoneneingabe**

| Beispiel | Beschreibung |
|---|---|
| `PST` | Abkürzung, für Pacific Standard Time |
| `America/New_York` | Vollständiger Zeitzonenname |
| `PST8PDT` | POSIX-artige Zeitzonenspezifikation |
| `-8:00:00` | UTC-Offset für PST |
| `-8:00` | UTC-Offset für PST, erweitertes ISO-8601-Format |
| `-800` | UTC-Offset für PST, einfaches ISO-8601-Format |
| `-8` | UTC-Offset für PST, einfaches ISO-8601-Format |
| `zulu` | Militärische Abkürzung für UTC |
| `z` | Kurzform von `zulu`, auch in ISO 8601 |

Weitere Informationen zur Angabe von Zeitzonen finden Sie in [Abschnitt 8.5.3](#853-zeitzonen).

#### 8.5.1.3. Zeitstempel

Gültige Eingaben für die Zeitstempeltypen bestehen aus der Verkettung eines Datums und einer Uhrzeit, gefolgt von einer optionalen Zeitzone und einem optionalen `AD` oder `BC`. Alternativ kann `AD`/`BC` vor der Zeitzone erscheinen, dies ist jedoch nicht die bevorzugte Reihenfolge. Daher sind

```text
1999-01-08 04:05:06
```

und

```text
1999-01-08 04:05:06 -8:00
```

gültige Werte, die dem ISO-8601-Standard folgen. Zusätzlich wird das verbreitete Format

```text
January 8 04:05:06 1999 PST
```

unterstützt.

Der SQL-Standard unterscheidet Literale von `timestamp without time zone` und `timestamp with time zone` anhand eines `+`- oder `-`-Symbols und eines Zeitzonen-Offsets nach der Uhrzeit. Nach dem Standard ist daher

```sql
TIMESTAMP '2004-10-19 10:23:54'
```

ein Zeitstempel ohne Zeitzone, während

```sql
TIMESTAMP '2004-10-19 10:23:54+02'
```

ein Zeitstempel mit Zeitzone ist. PostgreSQL untersucht den Inhalt einer Literalzeichenkette niemals, bevor es ihren Typ bestimmt, und behandelt daher beide obigen Werte als `timestamp without time zone`. Um sicherzustellen, dass ein Literal als `timestamp with time zone` behandelt wird, geben Sie den richtigen expliziten Typ an:

```sql
TIMESTAMP WITH TIME ZONE '2004-10-19 10:23:54+02'
```

Bei einem Wert, der als `timestamp without time zone` bestimmt wurde, ignoriert PostgreSQL jede Zeitzonenangabe stillschweigend. Der resultierende Wert wird also aus den Datums-/Zeitfeldern der Eingabezeichenkette abgeleitet und nicht wegen der Zeitzone angepasst.

Bei `timestamp with time zone`-Werten wird eine Eingabezeichenkette mit ausdrücklich angegebener Zeitzone anhand des passenden Offsets für diese Zeitzone in UTC (Universal Coordinated Time) umgewandelt. Wenn in der Eingabezeichenkette keine Zeitzone genannt ist, wird angenommen, dass sie in der durch den Systemparameter `TimeZone` angegebenen Zeitzone liegt, und sie wird mit dem Offset dieser Zeitzone nach UTC umgewandelt. In beiden Fällen wird der Wert intern als UTC gespeichert; die ursprünglich angegebene oder angenommene Zeitzone bleibt nicht erhalten.

Wenn ein Wert `timestamp with time zone` ausgegeben wird, wird er immer von UTC in die aktuelle Zeitzone umgewandelt und als lokale Zeit in dieser Zone angezeigt. Um die Zeit in einer anderen Zeitzone zu sehen, ändern Sie entweder `TimeZone` oder verwenden das Konstrukt `AT TIME ZONE` (siehe [Abschnitt 9.9.4](09_Funktionen_und_Operatoren.md#994-at-time-zone-und-at-local)).

Umwandlungen zwischen `timestamp without time zone` und `timestamp with time zone` nehmen normalerweise an, dass der Wert `timestamp without time zone` als lokale Zeit der Zeitzone zu nehmen oder zu liefern ist. Eine andere Zeitzone kann für die Umwandlung mit `AT TIME ZONE` angegeben werden.

#### 8.5.1.4. Sonderwerte

PostgreSQL unterstützt der Bequemlichkeit halber mehrere besondere Eingabewerte für Datum/Zeit, wie in Tabelle 8.13 gezeigt. Die Werte `infinity` und `-infinity` werden im System besonders dargestellt und unverändert angezeigt; die anderen sind lediglich Schreibabkürzungen, die beim Lesen in gewöhnliche Datums-/Zeitwerte umgewandelt werden. Insbesondere werden `now` und verwandte Zeichenketten sofort beim Lesen in einen bestimmten Zeitwert umgewandelt. Alle diese Werte müssen in einfache Anführungszeichen gesetzt werden, wenn sie als Konstanten in SQL-Befehlen verwendet werden.

**Tabelle 8.13. Besondere Datums-/Zeiteingaben**

| Eingabezeichenkette | Gültige Typen | Beschreibung |
|---|---|---|
| `epoch` | `date`, `timestamp` | `1970-01-01 00:00:00+00`, Unix-Systemzeit null |
| `infinity` | `date`, `timestamp`, `interval` | Später als alle anderen Zeitstempel |
| `-infinity` | `date`, `timestamp`, `interval` | Früher als alle anderen Zeitstempel |
| `now` | `date`, `time`, `timestamp` | Startzeit der aktuellen Transaktion |
| `today` | `date`, `timestamp` | Mitternacht, `00:00`, heute |
| `tomorrow` | `date`, `timestamp` | Mitternacht, `00:00`, morgen |
| `yesterday` | `date`, `timestamp` | Mitternacht, `00:00`, gestern |
| `allballs` | `time` | `00:00:00.00 UTC` |

Die folgenden SQL-kompatiblen Funktionen können ebenfalls verwendet werden, um den aktuellen Zeitwert für den entsprechenden Datentyp zu erhalten: `CURRENT_DATE`, `CURRENT_TIME`, `CURRENT_TIMESTAMP`, `LOCALTIME`, `LOCALTIMESTAMP` (siehe [Abschnitt 9.9.5](09_Funktionen_und_Operatoren.md#995-aktuelles-datumaktuelle-uhrzeit)). Beachten Sie, dass dies SQL-Funktionen sind und nicht in Dateneingabezeichenketten erkannt werden.

> Achtung: Die Eingabezeichenketten `now`, `today`, `tomorrow` und `yesterday` sind für interaktive SQL-Befehle in Ordnung, können aber überraschendes Verhalten zeigen, wenn der Befehl für spätere Ausführung gespeichert wird, zum Beispiel in vorbereiteten Anweisungen, Views und Funktionsdefinitionen. Die Zeichenkette kann in einen bestimmten Zeitwert umgewandelt werden, der noch lange weiterverwendet wird, nachdem er veraltet ist. Verwenden Sie in solchen Kontexten stattdessen eine der SQL-Funktionen. Zum Beispiel ist `CURRENT_DATE + 1` sicherer als `'tomorrow'::date`.

### 8.5.2. Datums-/Zeitausgabe

Das Ausgabeformat der Datums-/Zeittypen kann auf einen der vier Stile ISO 8601, SQL (Ingres), traditionelles POSTGRES (Unix-Datumsformat) oder German gesetzt werden. Die Vorgabe ist das ISO-Format. Der SQL-Standard verlangt die Verwendung des ISO-8601-Formats; der Name des Ausgabeformats "SQL" ist ein historischer Zufall. Tabelle 8.14 zeigt Beispiele für jeden Ausgabestil. Die Ausgabe der Datums- und Zeittypen enthält im Allgemeinen nur den Datums- oder Zeitteil entsprechend den gezeigten Beispielen. Der POSTGRES-Stil gibt jedoch reine Datumswerte im ISO-Format aus.

**Tabelle 8.14. Datums-/Zeitausgabestile**

| Stilangabe | Beschreibung | Beispiel |
|---|---|---|
| `ISO` | ISO 8601, SQL-Standard | `1997-12-17 07:37:16-08` |
| `SQL` | Traditioneller Stil | `12/17/1997 07:37:16.00 PST` |
| `Postgres` | Ursprünglicher Stil | `Wed Dec 17 07:37:16 1997 PST` |
| `German` | Regionaler Stil | `17.12.1997 07:37:16.00 PST` |

> Hinweis: ISO 8601 spezifiziert den Großbuchstaben `T` als Trenner zwischen Datum und Uhrzeit. PostgreSQL akzeptiert dieses Format bei der Eingabe, verwendet bei der Ausgabe aber ein Leerzeichen statt `T`, wie oben gezeigt. Das dient der Lesbarkeit und der Konsistenz mit [RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339) sowie einigen anderen Datenbanksystemen.

In den Stilen SQL und POSTGRES erscheint der Tag vor dem Monat, wenn die Feldreihenfolge `DMY` angegeben wurde; andernfalls erscheint der Monat vor dem Tag. Wie diese Einstellung auch die Interpretation von Eingabewerten beeinflusst, steht in [Abschnitt 8.5.1](#851-datumszeiteingabe). Tabelle 8.15 zeigt Beispiele.

**Tabelle 8.15. Konventionen zur Datumsreihenfolge**

| `datestyle`-Einstellung | Eingabereihenfolge | Beispielausgabe |
|---|---|---|
| `SQL, DMY` | Tag/Monat/Jahr | `17/12/1997 15:37:16.00 CET` |
| `SQL, MDY` | Monat/Tag/Jahr | `12/17/1997 07:37:16.00 PST` |
| `Postgres, DMY` | Tag/Monat/Jahr | `Wed 17 Dec 07:37:16 1997 PST` |

Im ISO-Stil wird die Zeitzone immer als vorzeichenbehafteter numerischer Offset von UTC angezeigt, wobei das positive Vorzeichen für Zonen östlich von Greenwich verwendet wird. Der Offset wird als `hh` angezeigt, wenn er eine ganze Zahl von Stunden ist, sonst als `hh:mm`, wenn er eine ganze Zahl von Minuten ist, sonst als `hh:mm:ss`. Der dritte Fall ist mit keinem modernen Zeitzonenstandard möglich, kann aber bei Zeitstempeln auftreten, die vor der Einführung standardisierter Zeitzonen liegen. In den anderen Datumsstilen wird die Zeitzone als alphabetische Abkürzung angezeigt, wenn in der aktuellen Zone eine allgemein verwendete Abkürzung existiert. Andernfalls erscheint sie als vorzeichenbehafteter numerischer Offset im einfachen ISO-8601-Format (`hh` oder `hhmm`). Die alphabetischen Abkürzungen in diesen Stilen stammen aus dem IANA-Zeitzonendatenbankeintrag, der aktuell durch den Laufzeitparameter `TimeZone` ausgewählt ist; sie werden nicht durch die Einstellung `timezone_abbreviations` beeinflusst.

Der Datums-/Zeitstil kann vom Benutzer mit dem Befehl `SET datestyle`, mit dem Parameter `DateStyle` in der Konfigurationsdatei `postgresql.conf` oder mit der Umgebungsvariable `PGDATESTYLE` auf Server oder Client ausgewählt werden.

Die Formatierungsfunktion `to_char` (siehe [Abschnitt 9.8](09_Funktionen_und_Operatoren.md#98-formatierungsfunktionen-für-datentypen)) steht ebenfalls als flexiblere Möglichkeit zur Formatierung von Datums-/Zeitausgaben zur Verfügung.

### 8.5.3. Zeitzonen

Zeitzonen und Zeitzonenkonventionen werden von politischen Entscheidungen beeinflusst, nicht nur von der Geometrie der Erde. Zeitzonen auf der ganzen Welt wurden im 20. Jahrhundert einigermaßen standardisiert, bleiben aber anfällig für willkürliche Änderungen, insbesondere bei Sommerzeitregeln. PostgreSQL verwendet die weit verbreitete IANA-(Olson-)Zeitzonendatenbank für Informationen über historische Zeitzonenregeln. Für Zeiten in der Zukunft wird angenommen, dass die zuletzt bekannten Regeln für eine gegebene Zeitzone beliebig weit in die Zukunft weitergelten.

PostgreSQL bemüht sich, bei typischer Verwendung mit den Definitionen des SQL-Standards kompatibel zu sein. Der SQL-Standard enthält jedoch eine eigentümliche Mischung aus Datums- und Zeittypen sowie Fähigkeiten. Zwei offensichtliche Probleme sind:

- Obwohl der Typ `date` keine zugeordnete Zeitzone haben kann, kann der Typ `time` eine haben. In der realen Welt haben Zeitzonen wenig Bedeutung, wenn sie nicht sowohl mit einem Datum als auch mit einer Uhrzeit verbunden sind, weil der Offset über das Jahr hinweg an Sommerzeitgrenzen variieren kann.

- Die Standardzeitzone wird als konstanter numerischer Offset von UTC angegeben. Dadurch ist es unmöglich, sich bei Datums-/Zeitarithmetik über Sommerzeitgrenzen hinweg an die Sommerzeit anzupassen.

Um diese Schwierigkeiten zu vermeiden, empfehlen wir bei Verwendung von Zeitzonen Datums-/Zeittypen, die sowohl Datum als auch Uhrzeit enthalten. Wir empfehlen den Typ `time with time zone` nicht, obwohl PostgreSQL ihn für Legacy-Anwendungen und zur Einhaltung des SQL-Standards unterstützt. PostgreSQL nimmt für jeden Typ, der nur Datum oder nur Uhrzeit enthält, Ihre lokale Zeitzone an.

Alle zeitzonenbewussten Datums- und Zeitwerte werden intern in UTC gespeichert. Bevor sie dem Client angezeigt werden, werden sie in die lokale Zeit der Zone umgewandelt, die durch den Konfigurationsparameter `TimeZone` angegeben ist.

PostgreSQL erlaubt die Angabe von Zeitzonen in drei verschiedenen Formen:

- Als vollständiger Zeitzonenname, zum Beispiel `America/New_York`. Die erkannten Zeitzonennamen sind in der View `pg_timezone_names` aufgeführt (siehe [Abschnitt 53.34](53_System_Views.md#5334-pgtimezonenames)). PostgreSQL verwendet dafür die weit verbreiteten IANA-Zeitzonendaten, sodass dieselben Zeitzonennamen auch von anderer Software erkannt werden.

- Als Zeitzonenabkürzung, zum Beispiel `PST`. Eine solche Angabe definiert lediglich einen bestimmten Offset von UTC, im Gegensatz zu vollständigen Zeitzonennamen, die auch eine Menge von Sommerzeit-Übergangsregeln implizieren können. Die erkannten Abkürzungen sind in der View `pg_timezone_abbrevs` aufgeführt (siehe [Abschnitt 53.33](53_System_Views.md#5333-pgtimezoneabbrevs)). Sie können die Konfigurationsparameter `TimeZone` oder `log_timezone` nicht auf eine Zeitzonenabkürzung setzen, aber Sie können Abkürzungen in Datums-/Zeiteingabewerten und mit dem Operator `AT TIME ZONE` verwenden.

- Zusätzlich zu Zeitzonennamen und Abkürzungen akzeptiert PostgreSQL POSIX-artige Zeitzonenspezifikationen, wie in Abschnitt B.5 beschrieben. Diese Option ist normalerweise nicht der Verwendung einer benannten Zeitzone vorzuziehen, kann aber nötig sein, wenn kein passender IANA-Zeitzoneneintrag verfügbar ist.

Kurz gesagt ist dies der Unterschied zwischen Abkürzungen und vollständigen Namen: Abkürzungen stellen einen bestimmten Offset von UTC dar, während viele vollständige Namen eine lokale Sommerzeitregel implizieren und daher zwei mögliche UTC-Offsets haben. Zum Beispiel steht `2014-06-04 12:00 America/New_York` für Mittag Ortszeit in New York; an diesem Datum war das Eastern Daylight Time (UTC-4). `2014-06-04 12:00 EDT` bezeichnet also denselben Zeitpunkt. `2014-06-04 12:00 EST` bezeichnet dagegen Mittag Eastern Standard Time (UTC-5), unabhängig davon, ob an diesem Datum nominell Sommerzeit galt.

> Hinweis: Das Vorzeichen in POSIX-artigen Zeitzonenspezifikationen hat die entgegengesetzte Bedeutung zum Vorzeichen in ISO-8601-Datums-/Zeitwerten. Die POSIX-Zeitzone für `2014-06-04 12:00+04` wäre zum Beispiel UTC-4.

Erschwerend kommt hinzu, dass manche Rechtsgebiete dieselbe Zeitzonenabkürzung zu verschiedenen Zeiten für unterschiedliche UTC-Offsets verwendet haben; in Moskau bedeutete `MSK` zum Beispiel in manchen Jahren UTC+3 und in anderen UTC+4. PostgreSQL interpretiert solche Abkürzungen entsprechend ihrer Bedeutung am angegebenen Datum oder ihrer zuletzt bekannten Bedeutung; wie beim EST-Beispiel oben ist dies jedoch nicht zwingend dasselbe wie die lokale zivile Zeit an diesem Datum.

In allen Fällen werden Zeitzonennamen und Abkürzungen ohne Beachtung der Groß-/Kleinschreibung erkannt. Dies ist eine Änderung gegenüber PostgreSQL-Versionen vor 8.2, die in manchen Kontexten groß-/kleinschreibungssensitiv waren und in anderen nicht.

Weder Zeitzonennamen noch Abkürzungen sind fest im Server verdrahtet; sie werden aus Konfigurationsdateien unter `.../share/timezone/` und `.../share/timezonesets/` des Installationsverzeichnisses bezogen (siehe Abschnitt B.4).

Der Konfigurationsparameter `TimeZone` kann in der Datei `postgresql.conf` oder auf eine der anderen in [Kapitel 19](19_Serverkonfiguration.md) beschriebenen Standardarten gesetzt werden. Es gibt außerdem einige besondere Möglichkeiten:

- Der SQL-Befehl `SET TIME ZONE` setzt die Zeitzone für die Sitzung. Dies ist eine alternative Schreibweise von `SET TIMEZONE TO` mit einer stärker SQL-standardkompatiblen Syntax.

- Die Umgebungsvariable `PGTZ` wird von `libpq`-Clients verwendet, um beim Verbindungsaufbau einen Befehl `SET TIME ZONE` an den Server zu senden.

### 8.5.4. Intervalleingabe

`interval`-Werte können mit der folgenden ausführlichen Syntax geschrieben werden:

```text
[@] quantity unit [quantity unit...] [direction]
```

Dabei ist `quantity` eine Zahl, möglicherweise mit Vorzeichen; `unit` ist `microsecond`, `millisecond`, `second`, `minute`, `hour`, `day`, `week`, `month`, `year`, `decade`, `century`, `millennium` oder eine Abkürzung beziehungsweise Pluralform dieser Einheiten; `direction` kann `ago` oder leer sein. Das At-Zeichen (`@`) ist optionales Füllmaterial. Die Beträge der verschiedenen Einheiten werden unter Berücksichtigung ihrer Vorzeichen implizit addiert. `ago` negiert alle Felder. Diese Syntax wird auch für die Intervallausgabe verwendet, wenn `IntervalStyle` auf `postgres_verbose` gesetzt ist.

Mengen von Tagen, Stunden, Minuten und Sekunden können ohne ausdrückliche Einheitenmarkierungen angegeben werden. Zum Beispiel wird `'1 12:59:10'` genauso gelesen wie `'1 day 12 hours 59 min 10 sec'`. Außerdem kann eine Kombination aus Jahren und Monaten mit einem Bindestrich angegeben werden; zum Beispiel wird `'200-10'` genauso gelesen wie `'200 years 10 months'`. Diese kürzeren Formen sind tatsächlich die einzigen, die der SQL-Standard erlaubt, und sie werden für die Ausgabe verwendet, wenn `IntervalStyle` auf `sql_standard` gesetzt ist.

Intervallwerte können außerdem als ISO-8601-Zeitintervalle geschrieben werden, entweder im "Format mit Designatoren" aus Abschnitt 4.4.3.2 des Standards oder im "alternativen Format" aus Abschnitt 4.4.3.3. Das Format mit Designatoren sieht so aus:

```text
P quantity unit [ quantity unit ...] [ T [ quantity unit ...]]
```

Die Zeichenkette muss mit `P` beginnen und kann ein `T` enthalten, das die Tageszeiteinheiten einleitet. Die verfügbaren Einheitenabkürzungen stehen in Tabelle 8.16. Einheiten können ausgelassen und in beliebiger Reihenfolge angegeben werden, aber Einheiten kleiner als ein Tag müssen nach `T` erscheinen. Insbesondere hängt die Bedeutung von `M` davon ab, ob es vor oder nach `T` steht.

**Tabelle 8.16. ISO-8601-Intervall-Einheitenabkürzungen**

| Abkürzung | Bedeutung |
|---|---|
| `Y` | Jahre |
| `M` | Monate, im Datumsteil |
| `W` | Wochen |
| `D` | Tage |
| `H` | Stunden |
| `M` | Minuten, im Zeitteil |
| `S` | Sekunden |

Im alternativen Format

```text
P [ years-months-days ] [ T hours:minutes:seconds ]
```

muss die Zeichenkette mit `P` beginnen, und ein `T` trennt den Datums- und den Zeitteil des Intervalls. Die Werte werden als Zahlen ähnlich ISO-8601-Datumswerten angegeben.

Wenn eine `interval`-Konstante mit einer Feldspezifikation geschrieben wird oder wenn eine Zeichenkette einer `interval`-Spalte zugewiesen wird, die mit einer Feldspezifikation definiert wurde, hängt die Interpretation unmarkierter Mengen von den Feldern ab. Zum Beispiel wird `INTERVAL '1' YEAR` als 1 Jahr gelesen, während `INTERVAL '1'` 1 Sekunde bedeutet. Außerdem werden Feldwerte "rechts" vom kleinsten durch die Feldspezifikation erlaubten Feld stillschweigend verworfen. Zum Beispiel führt `INTERVAL '1 day 2:03:04' HOUR TO MINUTE` dazu, dass das Sekundenfeld verworfen wird, nicht aber das Tagesfeld.

Nach dem SQL-Standard müssen alle Felder eines Intervallwerts dasselbe Vorzeichen haben, sodass ein führendes Minuszeichen auf alle Felder wirkt. Das Minuszeichen im Intervallliteral `'-1 2:03:04'` gilt zum Beispiel sowohl für den Tages- als auch für den Stunden-/Minuten-/Sekundenteil. PostgreSQL erlaubt Felder mit unterschiedlichen Vorzeichen und behandelt traditionell jedes Feld in der textuellen Darstellung als unabhängig vorzeichenbehaftet, sodass der Stunden-/Minuten-/Sekundenteil in diesem Beispiel als positiv betrachtet wird. Wenn `IntervalStyle` auf `sql_standard` gesetzt ist, gilt ein führendes Vorzeichen für alle Felder, aber nur wenn keine zusätzlichen Vorzeichen erscheinen. Andernfalls wird die traditionelle PostgreSQL-Interpretation verwendet. Um Mehrdeutigkeiten zu vermeiden, empfiehlt es sich, jedes Feld ausdrücklich mit einem Vorzeichen zu versehen, wenn irgendein Feld negativ ist.

Intern werden `interval`-Werte als drei ganzzahlige Felder gespeichert: Monate, Tage und Mikrosekunden. Diese Felder bleiben getrennt, weil die Anzahl der Tage in einem Monat variiert und ein Tag 23 oder 25 Stunden haben kann, wenn ein Sommerzeitübergang beteiligt ist. Eine Intervall-Eingabezeichenkette, die andere Einheiten verwendet, wird in dieses Format normalisiert und anschließend für die Ausgabe in standardisierter Weise wieder aufgebaut, zum Beispiel:

```sql
SELECT '2 years 15 months 100 weeks 99 hours 123456789
 milliseconds'::interval;
```

```text
              interval
---------------------------------------
 3 years 3 mons 700 days 133:17:36.789
```

Hier wurden Wochen, die als "7 Tage" verstanden werden, getrennt gehalten, während kleinere und größere Zeiteinheiten kombiniert und normalisiert wurden.

Eingabefeldwerte können Bruchteile haben, zum Beispiel `'1.5 weeks'` oder `'01:02:03.45'`. Da `interval` intern jedoch nur ganzzahlige Felder speichert, müssen Bruchwerte in kleinere Einheiten umgewandelt werden. Bruchteile von Einheiten größer als Monate werden auf eine ganzzahlige Anzahl von Monaten gerundet; zum Beispiel wird `'1.5 years'` zu `'1 year 6 mons'`. Bruchteile von Wochen und Tagen werden unter Annahme von 30 Tagen pro Monat und 24 Stunden pro Tag als ganzzahlige Anzahl von Tagen und Mikrosekunden berechnet; zum Beispiel wird `'1.75 months'` zu `1 mon 22 days 12:00:00`. In der Ausgabe werden nur Sekunden jemals als Bruchwert angezeigt.

Tabelle 8.17 zeigt einige Beispiele gültiger Intervalleingaben.

**Tabelle 8.17. Intervalleingabe**

| Beispiel | Beschreibung |
|---|---|
| `1-2` | SQL-Standardformat: 1 Jahr 2 Monate |
| `3 4:05:06` | SQL-Standardformat: 3 Tage 4 Stunden 5 Minuten 6 Sekunden |
| `1 year 2 months 3 days 4 hours 5 minutes 6 seconds` | Traditionelles Postgres-Format: 1 Jahr 2 Monate 3 Tage 4 Stunden 5 Minuten 6 Sekunden |
| `P1Y2M3DT4H5M6S` | ISO-8601-"Format mit Designatoren": dieselbe Bedeutung wie oben |
| `P0001-02-03T04:05:06` | ISO-8601-"alternatives Format": dieselbe Bedeutung wie oben |

### 8.5.5. Intervallausgabe

Wie zuvor erklärt, speichert PostgreSQL `interval`-Werte als Monate, Tage und Mikrosekunden. Für die Ausgabe wird das Monatsfeld durch Division durch 12 in Jahre und Monate umgewandelt. Das Tagesfeld wird unverändert angezeigt. Das Mikrosekundenfeld wird in Stunden, Minuten, Sekunden und Sekundenbruchteile umgewandelt. Deshalb werden Monate, Minuten und Sekunden niemals außerhalb der Bereiche 0-11, 0-59 beziehungsweise 0-59 angezeigt, während die angezeigten Felder für Jahre, Tage und Stunden sehr groß sein können. Die Funktionen `justify_days` und `justify_hours` können verwendet werden, wenn große Tages- oder Stundenwerte in das nächsthöhere Feld übertragen werden sollen.

Das Ausgabeformat des Typs `interval` kann mit dem Befehl `SET intervalstyle` auf einen der vier Stile `sql_standard`, `postgres`, `postgres_verbose` oder `iso_8601` gesetzt werden. Die Vorgabe ist das Format `postgres`. Tabelle 8.18 zeigt Beispiele für jeden Ausgabestil.

Der Stil `sql_standard` erzeugt eine Ausgabe, die der Spezifikation des SQL-Standards für Intervallliteral-Zeichenketten entspricht, sofern der Intervallwert die Einschränkungen des Standards erfüllt, also entweder nur Jahr-Monat oder nur Tag-Zeit ist und keine Mischung positiver und negativer Komponenten enthält. Andernfalls sieht die Ausgabe wie eine Standard-Jahr-Monat-Literalzeichenkette gefolgt von einer Tag-Zeit-Literalzeichenkette aus, wobei ausdrückliche Vorzeichen hinzugefügt werden, um Intervalle mit gemischten Vorzeichen eindeutig zu machen.

Die Ausgabe des Stils `postgres` entspricht der Ausgabe von PostgreSQL-Versionen vor 8.4, wenn der Parameter `DateStyle` auf `ISO` gesetzt war.

Die Ausgabe des Stils `postgres_verbose` entspricht der Ausgabe von PostgreSQL-Versionen vor 8.4, wenn der Parameter `DateStyle` auf eine Nicht-ISO-Ausgabe gesetzt war.

Die Ausgabe des Stils `iso_8601` entspricht dem "Format mit Designatoren", das in Abschnitt 4.4.3.2 des ISO-8601-Standards beschrieben wird.

**Tabelle 8.18. Beispiele für Intervall-Ausgabestile**

| Stilangabe | Jahr-Monat-Intervall | Tag-Zeit-Intervall | Gemischtes Intervall |
|---|---|---|---|
| `sql_standard` | `1-2` | `3 4:05:06` | `-1-2 +3 -4:05:06` |
| `postgres` | `1 year 2 mons` | `3 days 04:05:06` | `-1 year -2 mons +3 days -04:05:06` |
| `postgres_verbose` | `@ 1 year 2 mons` | `@ 3 days 4 hours 5 mins 6 secs` | `@ 1 year 2 mons -3 days 4 hours 5 mins 6 secs ago` |
| `iso_8601` | `P1Y2M` | `P3DT4H5M6S` | `P-1Y-2M3DT-4H-5M-6S` |

## 8.6. Boolescher Typ

PostgreSQL stellt den SQL-Standardtyp `boolean` bereit; siehe Tabelle 8.19. Der Typ `boolean` kann mehrere Zustände haben: "true", "false" und einen dritten Zustand "unknown", der durch den SQL-Nullwert dargestellt wird.

**Tabelle 8.19. Boolescher Datentyp**

| Name | Speichergröße | Beschreibung |
|---|---:|---|
| `boolean` | 1 Byte | Zustand wahr oder falsch |

Boolesche Konstanten können in SQL-Abfragen durch die SQL-Schlüsselwörter `TRUE`, `FALSE` und `NULL` dargestellt werden.

Die Datentyp-Eingabefunktion für den Typ `boolean` akzeptiert diese Zeichenkettendarstellungen für den Zustand "true":

```text
true
yes
on
```

und diese Darstellungen für den Zustand "false":

```text
false
no
off
```

Eindeutige Präfixe dieser Zeichenketten werden ebenfalls akzeptiert, zum Beispiel `t` oder `n`. Führender oder nachgestellter Leerraum wird ignoriert, und Groß-/Kleinschreibung spielt keine Rolle.

Die Datentyp-Ausgabefunktion für `boolean` gibt immer entweder `t` oder `f` aus, wie Beispiel 8.2 zeigt.

**Beispiel 8.2. Verwendung des booleschen Typs**

```sql
CREATE TABLE test1 (a boolean, b text);
INSERT INTO test1 VALUES (TRUE, 'sic est');
INSERT INTO test1 VALUES (FALSE, 'non est');
SELECT * FROM test1;
```

```text
 a |    b
---+---------
 t | sic est
 f | non est
```

```sql
SELECT * FROM test1 WHERE a;
```

```text
 a |    b
---+---------
 t | sic est
```

Die Schlüsselwörter `TRUE` und `FALSE` sind die bevorzugte, SQL-konforme Methode, boolesche Konstanten in SQL-Abfragen zu schreiben. Sie können aber auch die Zeichenkettendarstellungen verwenden, indem Sie der allgemeinen Syntax für Zeichenkettenliteral-Konstanten aus [Abschnitt 4.1.2.7](04_SQL_Syntax.md#4127-konstanten-anderer-typen) folgen, zum Beispiel `'yes'::boolean`.

Beachten Sie, dass der Parser automatisch versteht, dass `TRUE` und `FALSE` vom Typ `boolean` sind. Für `NULL` gilt das nicht, weil `NULL` jeden Typ haben kann. In manchen Kontexten müssen Sie `NULL` daher ausdrücklich nach `boolean` casten, zum Beispiel `NULL::boolean`. Umgekehrt kann der Cast bei einem booleschen Zeichenkettenliteral in Kontexten weggelassen werden, in denen der Parser ableiten kann, dass das Literal vom Typ `boolean` sein muss.

## 8.7. Aufzählungstypen

Aufzählungstypen, oder Enum-Typen, sind Datentypen, die aus einer statischen, geordneten Menge von Werten bestehen. Sie entsprechen den Enum-Typen, die in vielen Programmiersprachen unterstützt werden. Beispiele für einen Enum-Typ wären die Wochentage oder eine Menge von Statuswerten für ein Datenelement.

### 8.7.1. Deklaration von Aufzählungstypen

Enum-Typen werden mit dem Befehl `CREATE TYPE` erstellt, zum Beispiel:

```sql
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
```

Nach der Erstellung kann der Enum-Typ in Tabellen- und Funktionsdefinitionen ähnlich wie jeder andere Typ verwendet werden:

```sql
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
CREATE TABLE person (
    name text,
    current_mood mood
);
INSERT INTO person VALUES ('Moe', 'happy');
SELECT * FROM person WHERE current_mood = 'happy';
```

```text
 name | current_mood
------+--------------
 Moe  | happy
(1 row)
```

### 8.7.2. Sortierung

Die Reihenfolge der Werte in einem Enum-Typ ist die Reihenfolge, in der die Werte beim Erstellen des Typs aufgelistet wurden. Alle Standardvergleichsoperatoren und zugehörigen Aggregatfunktionen werden für Enums unterstützt. Zum Beispiel:

```sql
INSERT INTO person VALUES ('Larry', 'sad');
INSERT INTO person VALUES ('Curly', 'ok');
SELECT * FROM person WHERE current_mood > 'sad';
```

```text
 name  | current_mood
-------+--------------
 Moe   | happy
 Curly | ok
(2 rows)
```

```sql
SELECT * FROM person WHERE current_mood > 'sad' ORDER BY current_mood;
```

```text
 name  | current_mood
-------+--------------
 Curly | ok
 Moe   | happy
(2 rows)
```

```sql
SELECT name
FROM person
WHERE current_mood = (SELECT MIN(current_mood) FROM person);
```

```text
 name
-------
 Larry
(1 row)
```

### 8.7.3. Typsicherheit

Jeder Aufzählungsdatentyp ist eigenständig und kann nicht mit anderen Aufzählungstypen verglichen werden. Betrachten Sie dieses Beispiel:

```sql
CREATE TYPE happiness AS ENUM ('happy', 'very happy', 'ecstatic');
CREATE TABLE holidays (
     num_weeks integer,
     happiness happiness
);
INSERT INTO holidays(num_weeks,happiness) VALUES (4, 'happy');
INSERT INTO holidays(num_weeks,happiness) VALUES (6, 'very happy');
INSERT INTO holidays(num_weeks,happiness) VALUES (8, 'ecstatic');
INSERT INTO holidays(num_weeks,happiness) VALUES (2, 'sad');
```

```text
ERROR: invalid input value for enum happiness: "sad"
```

```sql
SELECT person.name, holidays.num_weeks FROM person, holidays
   WHERE person.current_mood = holidays.happiness;
```

```text
ERROR: operator does not exist: mood = happiness
```

Wenn Sie so etwas wirklich tun müssen, können Sie entweder einen benutzerdefinierten Operator schreiben oder Ihrer Abfrage explizite Casts hinzufügen:

```sql
SELECT person.name, holidays.num_weeks FROM person, holidays
  WHERE person.current_mood::text = holidays.happiness::text;
```

```text
 name | num_weeks
------+-----------
 Moe  |         4
(1 row)
```

### 8.7.4. Implementierungsdetails

Enum-Labels unterscheiden Groß- und Kleinschreibung, daher ist `'happy'` nicht dasselbe wie `'HAPPY'`. Auch Leerraum in Labels ist signifikant.

Obwohl Enum-Typen hauptsächlich für statische Wertemengen gedacht sind, wird das Hinzufügen neuer Werte zu einem vorhandenen Enum-Typ sowie das Umbenennen von Werten unterstützt (siehe `ALTER TYPE`). Vorhandene Werte können nicht aus einem Enum-Typ entfernt werden, und die Sortierreihenfolge solcher Werte kann nur durch Löschen und erneutes Erstellen des Enum-Typs geändert werden.

Ein Enum-Wert belegt auf der Platte vier Byte. Die Länge des textuellen Labels eines Enum-Werts wird durch die in PostgreSQL einkompilierte Einstellung `NAMEDATALEN` begrenzt; in Standard-Builds bedeutet das höchstens 63 Byte.

Die Zuordnungen von internen Enum-Werten zu textuellen Labels werden im Systemkatalog `pg_enum` gespeichert. Direktes Abfragen dieses Katalogs kann nützlich sein.

## 8.8. Geometrische Typen

Geometrische Datentypen stellen zweidimensionale räumliche Objekte dar. Tabelle 8.20 zeigt die in PostgreSQL verfügbaren geometrischen Typen.

**Tabelle 8.20. Geometrische Typen**

| Name | Speichergröße | Beschreibung | Darstellung |
|---|---:|---|---|
| `point` | 16 Byte | Punkt in einer Ebene | `(x,y)` |
| `line` | 24 Byte | Unendliche Gerade | `{A,B,C}` |
| `lseg` | 32 Byte | Endliches Liniensegment | `[(x1,y1),(x2,y2)]` |
| `box` | 32 Byte | Rechteckige Box | `(x1,y1),(x2,y2)` |
| `path` | 16+16n Byte | Geschlossener Pfad, ähnlich einem Polygon | `((x1,y1),...)` |
| `path` | 16+16n Byte | Offener Pfad | `[(x1,y1),...]` |
| `polygon` | 40+16n Byte | Polygon, ähnlich einem geschlossenen Pfad | `((x1,y1),...)` |
| `circle` | 24 Byte | Kreis | `<(x,y),r>` (Mittelpunkt und Radius) |

In all diesen Typen werden die einzelnen Koordinaten als Zahlen vom Typ `double precision` (`float8`) gespeichert.

Für verschiedene geometrische Operationen wie Skalierung, Verschiebung, Rotation und die Bestimmung von Schnittmengen steht eine reichhaltige Menge von Funktionen und Operatoren zur Verfügung. Sie werden in [Abschnitt 9.11](09_Funktionen_und_Operatoren.md#911-geometrische-funktionen-und-operatoren) erklärt.

### 8.8.1. Punkte

Punkte sind die grundlegenden zweidimensionalen Bausteine für geometrische Typen. Werte des Typs `point` werden mit einer der folgenden Syntaxformen angegeben:

```text
( x , y )
  x , y
```

Dabei sind `x` und `y` die jeweiligen Koordinaten als Gleitkommazahlen.

Punkte werden mit der ersten Syntax ausgegeben.

### 8.8.2. Geraden

Geraden werden durch die lineare Gleichung `Ax + By + C = 0` dargestellt, wobei `A` und `B` nicht beide null sind. Werte des Typs `line` werden in der folgenden Form ein- und ausgegeben:

```text
{ A, B, C }
```

Alternativ kann bei der Eingabe jede der folgenden Formen verwendet werden:

```text
[ ( x1 , y1 ) , ( x2 , y2 ) ]
( ( x1 , y1 ) , ( x2 , y2 ) )
  ( x1 , y1 ) , ( x2 , y2 )
    x1 , y1   ,   x2 , y2
```

Dabei sind `(x1,y1)` und `(x2,y2)` zwei unterschiedliche Punkte auf der Geraden.

### 8.8.3. Liniensegmente

Liniensegmente werden durch Punktepaare dargestellt, die die Endpunkte des Segments sind. Werte des Typs `lseg` werden mit einer der folgenden Syntaxformen angegeben:

```text
[ ( x1 , y1 ) , ( x2 , y2 ) ]
( ( x1 , y1 ) , ( x2 , y2 ) )
  ( x1 , y1 ) , ( x2 , y2 )
    x1 , y1   ,   x2 , y2
```

Dabei sind `(x1,y1)` und `(x2,y2)` die Endpunkte des Liniensegments.

Liniensegmente werden mit der ersten Syntax ausgegeben.

### 8.8.4. Boxen

Boxen werden durch Punktepaare dargestellt, die gegenüberliegende Ecken der Box sind. Werte des Typs `box` werden mit einer der folgenden Syntaxformen angegeben:

```text
( ( x1 , y1 ) , ( x2 , y2 ) )
  ( x1 , y1 ) , ( x2 , y2 )
    x1 , y1   ,   x2 , y2
```

Dabei sind `(x1,y1)` und `(x2,y2)` beliebige zwei gegenüberliegende Ecken der Box.

Boxen werden mit der zweiten Syntax ausgegeben.

Bei der Eingabe können beliebige zwei gegenüberliegende Ecken angegeben werden; die Werte werden nach Bedarf umgeordnet, sodass die obere rechte und die untere linke Ecke in dieser Reihenfolge gespeichert werden.

### 8.8.5. Pfade

Pfade werden durch Listen verbundener Punkte dargestellt. Pfade können offen sein, wobei der erste und der letzte Punkt der Liste als nicht verbunden gelten, oder geschlossen, wobei der erste und der letzte Punkt als verbunden gelten.

Werte des Typs `path` werden mit einer der folgenden Syntaxformen angegeben:

```text
[ ( x1 , y1 ) , ... , ( xn , yn ) ]
( ( x1 , y1 ) , ... , ( xn , yn ) )
  ( x1 , y1 ) , ... , ( xn , yn )
  ( x1 , y1   , ... ,   xn , yn )
    x1 , y1   , ... ,   xn , yn
```

Dabei sind die Punkte die Endpunkte der Liniensegmente, aus denen der Pfad besteht. Eckige Klammern (`[]`) kennzeichnen einen offenen Pfad, während runde Klammern (`()`) einen geschlossenen Pfad kennzeichnen. Wenn die äußersten Klammern wie in der dritten bis fünften Syntax fehlen, wird ein geschlossener Pfad angenommen.

Pfade werden je nach Fall mit der ersten oder zweiten Syntax ausgegeben.

### 8.8.6. Polygone

Polygone werden durch Listen von Punkten dargestellt, den Eckpunkten des Polygons. Polygone sind geschlossenen Pfaden sehr ähnlich; der wesentliche semantische Unterschied besteht darin, dass ein Polygon die von ihm eingeschlossene Fläche umfasst, ein Pfad dagegen nicht.

Ein wichtiger Implementierungsunterschied zwischen Polygonen und Pfaden besteht darin, dass die gespeicherte Darstellung eines Polygons seine kleinste umschließende Box enthält. Das beschleunigt bestimmte Suchoperationen, obwohl die Berechnung dieser Box beim Erstellen neuer Polygone zusätzlichen Aufwand verursacht.

Werte des Typs `polygon` werden mit einer der folgenden Syntaxformen angegeben:

```text
( ( x1 , y1 ) , ... , ( xn , yn ) )
  ( x1 , y1 ) , ... , ( xn , yn )
  ( x1 , y1   , ... ,   xn , yn )
    x1 , y1   , ... ,   xn , yn
```

Dabei sind die Punkte die Endpunkte der Liniensegmente, aus denen die Begrenzung des Polygons besteht.

Polygone werden mit der ersten Syntax ausgegeben.

### 8.8.7. Kreise

Kreise werden durch einen Mittelpunkt und einen Radius dargestellt. Werte des Typs `circle` werden mit einer der folgenden Syntaxformen angegeben:

```text
< ( x , y ) , r >
( ( x , y ) , r )
  ( x , y ) , r
    x , y   , r
```

Dabei ist `(x,y)` der Mittelpunkt und `r` der Radius des Kreises.

Kreise werden mit der ersten Syntax ausgegeben.

## 8.9. Netzwerkadresstypen

PostgreSQL bietet Datentypen zum Speichern von IPv4-, IPv6- und MAC-Adressen, wie in Tabelle 8.21 gezeigt. Es ist besser, diese Typen statt einfacher Texttypen zum Speichern von Netzwerkadressen zu verwenden, weil sie Eingabefehler prüfen und spezialisierte Operatoren und Funktionen bereitstellen (siehe [Abschnitt 9.12](09_Funktionen_und_Operatoren.md#912-netzwerkadressfunktionen-und-operatoren)).

**Tabelle 8.21. Netzwerkadresstypen**

| Name | Speichergröße | Beschreibung |
|---|---:|---|
| `cidr` | 7 oder 19 Byte | IPv4- und IPv6-Netzwerke |
| `inet` | 7 oder 19 Byte | IPv4- und IPv6-Hosts und -Netzwerke |
| `macaddr` | 6 Byte | MAC-Adressen |
| `macaddr8` | 8 Byte | MAC-Adressen im EUI-64-Format |

Beim Sortieren der Datentypen `inet` oder `cidr` werden IPv4-Adressen immer vor IPv6-Adressen sortiert, einschließlich IPv4-Adressen, die in IPv6-Adressen eingekapselt oder auf sie abgebildet sind, etwa `::10.2.3.4` oder `::ffff:10.4.3.2`.

### 8.9.1. `inet`

Der Typ `inet` enthält eine IPv4- oder IPv6-Hostadresse und optional ihr Subnetz, alles in einem Feld. Das Subnetz wird durch die Anzahl der Netzwerkadressbits dargestellt, die in der Hostadresse vorhanden sind, also durch die "Netzmaske". Wenn die Netzmaske 32 ist und die Adresse IPv4 ist, zeigt der Wert kein Subnetz an, sondern nur einen einzelnen Host. Bei IPv6 beträgt die Adresslänge 128 Bit, daher geben 128 Bit eine eindeutige Hostadresse an. Beachten Sie: Wenn Sie nur Netzwerke akzeptieren möchten, sollten Sie den Typ `cidr` statt `inet` verwenden.

Das Eingabeformat für diesen Typ ist `address/y`, wobei `address` eine IPv4- oder IPv6-Adresse und `y` die Anzahl der Bits in der Netzmaske ist. Wenn der Teil `/y` ausgelassen wird, gilt die Netzmaske als 32 für IPv4 oder 128 für IPv6, sodass der Wert nur einen einzelnen Host darstellt. Bei der Anzeige wird der Teil `/y` unterdrückt, wenn die Netzmaske einen einzelnen Host angibt.

### 8.9.2. `cidr`

Der Typ `cidr` enthält eine IPv4- oder IPv6-Netzwerkspezifikation. Eingabe- und Ausgabeformate folgen den Konventionen des Classless Internet Domain Routing. Das Format zur Angabe von Netzwerken ist `address/y`, wobei `address` die niedrigste Adresse des Netzwerks als IPv4- oder IPv6-Adresse ist und `y` die Anzahl der Bits in der Netzmaske. Wenn `y` ausgelassen wird, wird es anhand von Annahmen aus dem älteren klassenbasierten Netzwerknummerierungssystem berechnet, ist aber mindestens groß genug, um alle in der Eingabe geschriebenen Oktette einzuschließen. Es ist ein Fehler, eine Netzwerkadresse anzugeben, bei der rechts der angegebenen Netzmaske Bits gesetzt sind.

Tabelle 8.22 zeigt einige Beispiele.

**Tabelle 8.22. Eingabebeispiele für den Typ `cidr`**

| `cidr`-Eingabe | `cidr`-Ausgabe | `abbrev(cidr)` |
|---|---|---|
| `192.168.100.128/25` | `192.168.100.128/25` | `192.168.100.128/25` |
| `192.168/24` | `192.168.0.0/24` | `192.168.0/24` |
| `192.168/25` | `192.168.0.0/25` | `192.168.0.0/25` |
| `192.168.1` | `192.168.1.0/24` | `192.168.1/24` |
| `192.168` | `192.168.0.0/24` | `192.168.0/24` |
| `128.1` | `128.1.0.0/16` | `128.1/16` |
| `128` | `128.0.0.0/16` | `128.0/16` |
| `128.1.2` | `128.1.2.0/24` | `128.1.2/24` |
| `10.1.2` | `10.1.2.0/24` | `10.1.2/24` |
| `10.1` | `10.1.0.0/16` | `10.1/16` |
| `10` | `10.0.0.0/8` | `10/8` |
| `10.1.2.3/32` | `10.1.2.3/32` | `10.1.2.3/32` |
| `2001:4f8:3:ba::/64` | `2001:4f8:3:ba::/64` | `2001:4f8:3:ba/64` |
| `2001:4f8:3:ba:2e0:81ff:fe22:d1f1/128` | `2001:4f8:3:ba:2e0:81ff:fe22:d1f1/128` | `2001:4f8:3:ba:2e0:81ff:fe22:d1f1/128` |
| `::ffff:1.2.3.0/120` | `::ffff:1.2.3.0/120` | `::ffff:1.2.3/120` |
| `::ffff:1.2.3.0/128` | `::ffff:1.2.3.0/128` | `::ffff:1.2.3.0/128` |

### 8.9.3. `inet` vs. `cidr`

Der wesentliche Unterschied zwischen den Datentypen `inet` und `cidr` besteht darin, dass `inet` Werte mit Nicht-Null-Bits rechts der Netzmaske akzeptiert, `cidr` dagegen nicht. Zum Beispiel ist `192.168.0.1/24` für `inet` gültig, aber nicht für `cidr`.

> Tipp: Wenn Ihnen das Ausgabeformat für `inet`- oder `cidr`-Werte nicht gefällt, probieren Sie die Funktionen `host`, `text` und `abbrev`.

### 8.9.4. `macaddr`

Der Typ `macaddr` speichert MAC-Adressen, wie man sie zum Beispiel von Hardwareadressen von Ethernet-Karten kennt, obwohl MAC-Adressen auch für andere Zwecke verwendet werden. Eingaben werden in den folgenden Formaten akzeptiert:

```text
'08:00:2b:01:02:03'
'08-00-2b-01-02-03'
'08002b:010203'
'08002b-010203'
'0800.2b01.0203'
'0800-2b01-0203'
'08002b010203'
```

Diese Beispiele geben alle dieselbe Adresse an. Für die Ziffern `a` bis `f` werden Groß- und Kleinbuchstaben akzeptiert. Die Ausgabe erfolgt immer in der ersten der gezeigten Formen.

IEEE-Standard 802-2001 spezifiziert die zweite gezeigte Form mit Bindestrichen als kanonische Form für MAC-Adressen und die erste Form mit Doppelpunkten als Verwendung mit bitumgekehrter MSB-first-Notation, sodass `08-00-2b-01-02-03 = 10:00:D4:80:40:C0`. Diese Konvention wird heute weithin ignoriert und ist nur für veraltete Netzwerkprotokolle wie Token Ring relevant. PostgreSQL trifft keine Vorkehrungen für Bitumkehrung; alle akzeptierten Formate verwenden die kanonische LSB-Reihenfolge.

Die verbleibenden fünf Eingabeformate gehören zu keinem Standard.

### 8.9.5. `macaddr8`

Der Typ `macaddr8` speichert MAC-Adressen im EUI-64-Format, wie man sie zum Beispiel von Hardwareadressen von Ethernet-Karten kennt, obwohl MAC-Adressen auch für andere Zwecke verwendet werden. Dieser Typ kann MAC-Adressen mit 6 oder 8 Byte Länge akzeptieren und speichert sie im 8-Byte-Format. MAC-Adressen im 6-Byte-Format werden im 8-Byte-Format gespeichert, wobei das 4. und 5. Byte auf `FF` beziehungsweise `FE` gesetzt werden. Beachten Sie, dass IPv6 ein modifiziertes EUI-64-Format verwendet, bei dem nach der Umwandlung aus EUI-48 das 7. Bit auf eins gesetzt werden sollte. Die Funktion `macaddr8_set7bit` wird bereitgestellt, um diese Änderung vorzunehmen. Allgemein wird jede Eingabe akzeptiert, die aus Paaren hexadezimaler Ziffern auf Bytegrenzen besteht, optional konsistent getrennt durch eines von `:`, `-` oder `.`. Die Anzahl der hexadezimalen Ziffern muss entweder 16 (8 Byte) oder 12 (6 Byte) sein. Führender und nachgestellter Leerraum wird ignoriert. Die folgenden Eingabeformate werden akzeptiert:

```text
'08:00:2b:01:02:03:04:05'
'08-00-2b-01-02-03-04-05'
'08002b:0102030405'
'08002b-0102030405'
'0800.2b01.0203.0405'
'0800-2b01-0203-0405'
'08002b01:02030405'
'08002b0102030405'
```

Diese Beispiele geben alle dieselbe Adresse an. Für die Ziffern `a` bis `f` werden Groß- und Kleinbuchstaben akzeptiert. Die Ausgabe erfolgt immer in der ersten der gezeigten Formen.

Die letzten sechs oben gezeigten Eingabeformate gehören zu keinem Standard.

Um eine traditionelle 48-Bit-MAC-Adresse im EUI-48-Format in das modifizierte EUI-64-Format umzuwandeln, damit sie als Host-Anteil einer IPv6-Adresse verwendet werden kann, verwenden Sie `macaddr8_set7bit` wie gezeigt:

```sql
SELECT macaddr8_set7bit('08:00:2b:01:02:03');
```

```text
    macaddr8_set7bit
-------------------------
 0a:00:2b:ff:fe:01:02:03
(1 row)
```

## 8.10. Bit-String-Typen

Bit-Strings sind Zeichenketten aus Einsen und Nullen. Sie können verwendet werden, um Bitmasken zu speichern oder sichtbar zu machen. Es gibt zwei SQL-Bittypen: `bit(n)` und `bit varying(n)`, wobei `n` eine positive Ganzzahl ist.

Daten des Typs `bit` müssen exakt der Länge `n` entsprechen; der Versuch, kürzere oder längere Bit-Strings zu speichern, ist ein Fehler. Daten des Typs `bit varying` haben variable Länge bis zur Maximallänge `n`; längere Zeichenketten werden abgewiesen. Die Schreibweise `bit` ohne Länge ist gleichbedeutend mit `bit(1)`, während `bit varying` ohne Längenangabe unbegrenzte Länge bedeutet.

> Hinweis: Wenn ein Bit-String-Wert ausdrücklich nach `bit(n)` gecastet wird, wird er rechts abgeschnitten oder mit Nullen aufgefüllt, sodass er exakt `n` Bits lang ist, ohne einen Fehler auszulösen. Wenn ein Bit-String-Wert ausdrücklich nach `bit varying(n)` gecastet wird, wird er entsprechend rechts abgeschnitten, falls er mehr als `n` Bits hat.

Informationen zur Syntax von Bit-String-Konstanten finden Sie in [Abschnitt 4.1.2.5](04_SQL_Syntax.md#4125-bitstringkonstanten). Bit-logische Operatoren und Zeichenkettenmanipulationsfunktionen sind verfügbar; siehe [Abschnitt 9.6](09_Funktionen_und_Operatoren.md#96-bitstringfunktionen-und-operatoren).

**Beispiel 8.3. Verwendung der Bit-String-Typen**

```sql
CREATE TABLE test (a BIT(3), b BIT VARYING(5));
INSERT INTO test VALUES (B'101', B'00');
INSERT INTO test VALUES (B'10', B'101');
```

```text
ERROR:      bit string length 2 does not match type bit(3)
```

```sql
INSERT INTO test VALUES (B'10'::bit(3), B'101');
SELECT * FROM test;
```

```text
  a  |  b
-----+-----
 101 | 00
 100 | 101
```

Ein Bit-String-Wert benötigt 1 Byte für jede Gruppe von 8 Bits, plus 5 oder 8 Byte Overhead je nach Länge der Zeichenkette. Lange Werte können jedoch komprimiert oder ausgelagert werden, wie in [Abschnitt 8.3](#83-zeichentypen) für Zeichenketten erklärt.

## 8.11. Volltextsuchtypen

PostgreSQL stellt zwei Datentypen bereit, die für Volltextsuche gedacht sind: das Durchsuchen einer Sammlung natürlichsprachlicher Dokumente nach den Treffern, die am besten zu einer Abfrage passen. Der Typ `tsvector` stellt ein Dokument in einer für die Textsuche optimierten Form dar; der Typ `tsquery` stellt entsprechend eine Textsuchabfrage dar. [Kapitel 12](12_Volltextsuche.md) erklärt diese Funktionalität ausführlich, und [Abschnitt 9.13](09_Funktionen_und_Operatoren.md#913-volltextsuchfunktionen-und-operatoren) fasst die zugehörigen Funktionen und Operatoren zusammen.

### 8.11.1. `tsvector`

Ein `tsvector`-Wert ist eine sortierte Liste unterschiedlicher Lexeme. Lexeme sind Wörter, die normalisiert wurden, um verschiedene Varianten desselben Wortes zusammenzuführen (siehe [Kapitel 12](12_Volltextsuche.md)). Sortierung und Entfernen von Duplikaten erfolgen bei der Eingabe automatisch:

```sql
SELECT 'a fat cat sat on a mat and ate a fat rat'::tsvector;
```

```text
                      tsvector
----------------------------------------------------
 'a' 'and' 'ate' 'cat' 'fat' 'mat' 'on' 'rat' 'sat'
```

Lexeme mit Leerraum oder Satzzeichen werden in Anführungszeichen gesetzt:

```sql
SELECT $$the lexeme '     ' contains spaces$$::tsvector;
```

```text
                  tsvector
-------------------------------------------
 '    ' 'contains' 'lexeme' 'spaces' 'the'
```

In diesem und dem nächsten Beispiel werden dollar-quotierte Zeichenkettenliterale verwendet, damit Anführungszeichen innerhalb der Literale nicht zusätzlich verdoppelt werden müssen. Eingebettete Anführungszeichen und Backslashes müssen verdoppelt werden:

```sql
SELECT $$the lexeme 'Joe''s' contains a quote$$::tsvector;
```

```text
                    tsvector
------------------------------------------------
 'Joe''s' 'a' 'contains' 'lexeme' 'quote' 'the'
```

Optional können Lexemen ganzzahlige Positionen angehängt werden:

```sql
SELECT 'a:1 fat:2 cat:3 sat:4 on:5 a:6 mat:7 and:8 ate:9 a:10 fat:11 rat:12'::tsvector;
```

```text
                                  tsvector
-------------------------------------------------------------------------------
 'a':1,6,10 'and':8 'ate':9 'cat':3 'fat':2,11 'mat':7 'on':5 'rat':12 'sat':4
```

Eine Position gibt normalerweise die Stelle des Quellworts im Dokument an. Positionsinformationen können für Nähe-Rankings genutzt werden. Positionswerte reichen von 1 bis 16383; größere Zahlen werden stillschweigend auf 16383 gesetzt. Doppelte Positionen desselben Lexems werden verworfen.

Lexeme mit Positionen können außerdem mit einer Gewichtung `A`, `B`, `C` oder `D` versehen werden. `D` ist die Voreinstellung und wird deshalb in der Ausgabe nicht angezeigt:

```sql
SELECT 'a:1A fat:2B,4C cat:5D'::tsvector;
```

```text
          tsvector
----------------------------
 'a':1A 'cat':5 'fat':2B,4C
```

Gewichtungen werden typischerweise genutzt, um die Struktur eines Dokuments abzubilden, zum Beispiel indem Wörter aus Überschriften anders markiert werden als Wörter aus dem Fließtext. Ranking-Funktionen der Volltextsuche können den verschiedenen Gewichtungen unterschiedliche Prioritäten geben.

Wichtig ist: Der Typ `tsvector` normalisiert Wörter nicht selbst. Er setzt voraus, dass die übergebenen Wörter bereits passend zur Anwendung normalisiert sind. Beispiel:

```sql
SELECT 'The Fat Rats'::tsvector;
```

```text
      tsvector
--------------------
 'Fat' 'Rats' 'The'
```

Für die meisten englischen Volltextsuchanwendungen wären diese Wörter nicht normalisiert, aber `tsvector` bewertet das nicht. Roher Dokumenttext sollte normalerweise durch `to_tsvector` geleitet werden, um die Wörter suchgerecht zu normalisieren:

```sql
SELECT to_tsvector('english', 'The Fat Rats');
```

```text
   to_tsvector
-----------------
 'fat':2 'rat':3
```

Weitere Einzelheiten stehen in [Kapitel 12](12_Volltextsuche.md).

### 8.11.2. `tsquery`

Ein `tsquery`-Wert speichert Lexeme, nach denen gesucht werden soll, und kann sie mit den booleschen Operatoren `&` (AND), `|` (OR) und `!` (NOT) sowie mit dem Phrasensuchoperator `<->` (FOLLOWED BY) kombinieren. Außerdem gibt es die Variante `<N>` des FOLLOWED-BY-Operators, wobei `N` eine ganzzahlige Konstante ist, die den Abstand zwischen den beiden gesuchten Lexemen angibt. `<->` entspricht `<1>`.

Klammern erzwingen die Gruppierung dieser Operatoren. Ohne Klammern bindet `!` am stärksten, danach `<->`, danach `&`; `|` bindet am schwächsten.

Beispiele:

```sql
SELECT 'fat & rat'::tsquery;
SELECT 'fat & (rat | cat)'::tsquery;
SELECT 'fat & rat & ! cat'::tsquery;
```

```text
    tsquery
---------------
 'fat' & 'rat'

          tsquery
---------------------------
 'fat' & ( 'rat' | 'cat' )

        tsquery
------------------------
 'fat' & 'rat' & !'cat'
```

Optional können Lexeme in einem `tsquery` mit einem oder mehreren Gewichtungsbuchstaben markiert werden. Dadurch passen sie nur auf `tsvector`-Lexeme mit einer dieser Gewichtungen:

```sql
SELECT 'fat:ab & cat'::tsquery;
```

```text
    tsquery
------------------
 'fat':AB & 'cat'
```

Mit `*` kann außerdem Präfixsuche angefordert werden:

```sql
SELECT 'super:*'::tsquery;
```

```text
  tsquery
-----------
 'super':*
```

Diese Abfrage passt auf jedes Wort in einem `tsvector`, das mit `super` beginnt.

Die Quoting-Regeln für Lexeme entsprechen den oben für `tsvector` beschriebenen Regeln. Auch bei `tsquery` muss jede notwendige Wortnormalisierung vor der Umwandlung in den Typ `tsquery` erfolgen. Dafür ist `to_tsquery` praktisch:

```sql
SELECT to_tsquery('Fat:ab & Cats');
```

```text
    to_tsquery
------------------
 'fat':AB & 'cat'
```

`to_tsquery` verarbeitet Präfixe genauso wie andere Wörter. Deshalb liefert dieser Vergleich `true`:

```sql
SELECT to_tsvector('postgraduate') @@ to_tsquery('postgres:*');
```

```text
 ?column?
----------
 t
```

Der Grund ist, dass `postgres` zu `postgr` gestemmt wird:

```sql
SELECT to_tsvector('postgraduate'), to_tsquery('postgres:*');
```

```text
  to_tsvector | to_tsquery
---------------+------------
 'postgradu':1 | 'postgr':*
```

Das passt auf die gestemmte Form von `postgraduate`.

## 8.12. UUID-Typ

Der Datentyp `uuid` speichert Universally Unique Identifiers (UUIDs), wie sie in RFC 9562, ISO/IEC 9834-8:2005 und verwandten Standards definiert sind. Manche Systeme nennen diesen Datentyp auch Globally Unique Identifier (GUID). Ein UUID ist ein 128-Bit-Wert, der durch einen Algorithmus erzeugt wird, der Kollisionen extrem unwahrscheinlich machen soll. Für verteilte Systeme bieten solche Bezeichner daher eine stärkere Eindeutigkeitsgarantie als Sequenzgeneratoren, die nur innerhalb einer einzelnen Datenbank eindeutig sind.

RFC 9562 definiert acht UUID-Versionen. Jede Version stellt eigene Anforderungen an die Erzeugung neuer UUID-Werte und hat eigene Vor- und Nachteile. PostgreSQL unterstützt die Erzeugung von UUIDs mit den Algorithmen UUIDv4 und UUIDv7 nativ. Alternativ können UUID-Werte außerhalb der Datenbank mit einem beliebigen Algorithmus erzeugt werden. Der Typ `uuid` kann jede UUID speichern, unabhängig von Herkunft und Version.

Eine UUID wird als Folge kleingeschriebener hexadezimaler Ziffern geschrieben, aufgeteilt in Gruppen, die durch Bindestriche getrennt sind: 8 Ziffern, drei Gruppen zu je 4 Ziffern und eine Gruppe zu 12 Ziffern. Das ergibt insgesamt 32 Ziffern für 128 Bit. Beispiel:

```text
a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11
```

PostgreSQL akzeptiert bei der Eingabe außerdem Großbuchstaben, geschweifte Klammern um das Standardformat, das Weglassen einzelner oder aller Bindestriche sowie zusätzliche Bindestriche nach beliebigen Vierergruppen:

```text
A0EEBC99-9C0B-4EF8-BB6D-6BB9BD380A11
{a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11}
a0eebc999c0b4ef8bb6d6bb9bd380a11
a0ee-bc99-9c0b-4ef8-bb6d-6bb9-bd38-0a11
{a0eebc99-9c0b4ef8-bb6d6bb9-bd380a11}
```

Die Ausgabe erfolgt immer im Standardformat. Wie UUIDs in PostgreSQL erzeugt werden, beschreibt [Abschnitt 9.14](09_Funktionen_und_Operatoren.md#914-uuidfunktionen).

## 8.13. XML-Typ

Der Datentyp `xml` kann XML-Daten speichern. Gegenüber der Speicherung in einem `text`-Feld hat er den Vorteil, dass Eingabewerte auf Wohlgeformtheit geprüft werden und dass unterstützende Funktionen für typsichere Operationen vorhanden sind; siehe [Abschnitt 9.15](09_Funktionen_und_Operatoren.md#915-xmlfunktionen). Die Verwendung dieses Datentyps setzt voraus, dass PostgreSQL mit `configure --with-libxml` gebaut wurde.

Der Typ `xml` kann wohlgeformte Dokumente im Sinne des XML-Standards speichern, aber auch Inhaltsfragmente, die sich auf den allgemeineren Dokumentknoten des XQuery- und XPath-Datenmodells beziehen. Grob gesagt können Inhaltsfragmente mehr als ein Element oder mehr als einen Zeichenknoten auf oberster Ebene enthalten. Mit dem Ausdruck `xmlvalue IS DOCUMENT` lässt sich prüfen, ob ein bestimmter `xml`-Wert ein vollständiges Dokument oder nur ein Inhaltsfragment ist.

Grenzen und Kompatibilitätshinweise zum Datentyp `xml` finden sich in Abschnitt D.3.

### 8.13.1. XML-Werte erzeugen

Um aus Zeichendaten einen Wert des Typs `xml` zu erzeugen, verwenden Sie `xmlparse`:

```sql
XMLPARSE ( { DOCUMENT | CONTENT } value )
```

Beispiele:

```sql
XMLPARSE (DOCUMENT '<?xml version="1.0"?><book><title>Manual</title><chapter>...</chapter></book>')
XMLPARSE (CONTENT 'abc<foo>bar</foo><bar>foo</bar>')
```

Nach dem SQL-Standard ist dies die einzige Möglichkeit, Zeichenketten in XML-Werte umzuwandeln. PostgreSQL erlaubt zusätzlich diese PostgreSQL-spezifischen Schreibweisen:

```sql
xml '<foo>bar</foo>'
'<foo>bar</foo>'::xml
```

Der Typ `xml` validiert Eingabewerte nicht gegen eine Dokumenttypdeklaration (DTD), selbst wenn der Eingabewert eine DTD angibt. Auch für andere XML-Schemasprachen wie XML Schema gibt es derzeit keine eingebaute Validierungsunterstützung.

Die Gegenrichtung, also das Erzeugen einer Zeichenkette aus `xml`, verwendet `xmlserialize`:

```sql
XMLSERIALIZE ( { DOCUMENT | CONTENT } value AS type [ [ NO ] INDENT ] )
```

`type` kann `character`, `character varying` oder `text` sein, oder ein Alias für einen dieser Typen. Auch hier ist dies nach dem SQL-Standard die einzige Umwandlung zwischen `xml` und Zeichentypen, PostgreSQL erlaubt aber auch einfache Casts.

Die Option `INDENT` formatiert das Ergebnis lesbarer; `NO INDENT` ist die Voreinstellung und gibt die ursprüngliche Eingabezeichenkette aus. Ein Cast in einen Zeichentyp erzeugt ebenfalls die ursprüngliche Zeichenkette.

Wenn ein Zeichenkettenwert ohne `XMLPARSE` oder `XMLSERIALIZE` in `xml` oder aus `xml` gecastet wird, entscheidet der Sitzungsparameter `XML OPTION`, ob `DOCUMENT` oder `CONTENT` verwendet wird:

```sql
SET XML OPTION { DOCUMENT | CONTENT };
```

oder in PostgreSQL-typischer Schreibweise:

```sql
SET xmloption TO { DOCUMENT | CONTENT };
```

Die Voreinstellung ist `CONTENT`; damit sind alle Formen von XML-Daten zulässig.

### 8.13.2. Kodierungsbehandlung

Beim Umgang mit mehreren Zeichenkodierungen auf Client, Server und in den durchgereichten XML-Daten ist Vorsicht nötig. Im Textmodus, dem normalen Modus für Abfragen und Ergebnisse, wandelt PostgreSQL alle Zeichendaten zwischen Client und Server jeweils in die Kodierung der Gegenseite um; siehe [Abschnitt 23.3](23_Lokalisierung.md#233-zeichensatzunterstützung). Dazu gehören auch Zeichenkettendarstellungen von XML-Werten. Eingebettete Kodierungsdeklarationen in XML-Daten können dadurch ungültig werden, weil sie bei der Umkodierung nicht angepasst werden. Deshalb ignoriert PostgreSQL Kodierungsdeklarationen in Zeichenketten, die als Eingabe für den Typ `xml` dienen, und nimmt an, dass der Inhalt in der aktuellen Serverkodierung vorliegt. Für korrekte Verarbeitung müssen XML-Zeichenketten vom Client in der aktuellen Client-Kodierung gesendet werden. Der Client ist dafür verantwortlich, Dokumente vorher umzuwandeln oder die Client-Kodierung passend einzustellen. Bei der Ausgabe enthalten `xml`-Werte keine Kodierungsdeklaration; Clients sollten die aktuelle Client-Kodierung annehmen.

Im Binärmodus für Parameter und Ergebnisse findet keine Kodierungsumwandlung statt. Dann wird eine Kodierungsdeklaration in den XML-Daten beachtet; fehlt sie, wird UTF-8 angenommen, wie vom XML-Standard gefordert. PostgreSQL unterstützt dabei kein UTF-16. Bei der Ausgabe erhalten die Daten eine Kodierungsdeklaration mit der Client-Kodierung, außer wenn diese UTF-8 ist; dann wird sie weggelassen.

XML-Verarbeitung mit PostgreSQL ist weniger fehleranfällig und effizienter, wenn XML-Datenkodierung, Client-Kodierung und Serverkodierung übereinstimmen. Da XML-Daten intern in UTF-8 verarbeitet werden, ist die Verarbeitung am effizientesten, wenn auch die Serverkodierung UTF-8 ist.

> **Vorsicht**
> Einige XML-bezogene Funktionen funktionieren bei Nicht-ASCII-Daten möglicherweise gar nicht, wenn die Serverkodierung nicht UTF-8 ist. Das betrifft insbesondere `xmltable()` und `xpath()`.

### 8.13.3. Zugriff auf XML-Werte

Der Datentyp `xml` ist insofern ungewöhnlich, als er keine Vergleichsoperatoren bereitstellt. Es gibt keinen wohldefinierten und allgemein nützlichen Vergleichsalgorithmus für XML-Daten. Eine Folge davon ist, dass Zeilen nicht durch den Vergleich einer `xml`-Spalte mit einem Suchwert abgerufen werden können. XML-Werte sollten deshalb typischerweise von einem separaten Schlüsselfeld begleitet werden, etwa einer ID. Alternativ kann man XML-Werte zuerst in Zeichenketten umwandeln, aber Zeichenkettenvergleich hat wenig mit einem sinnvollen XML-Vergleich zu tun.

Da es für `xml` keine Vergleichsoperatoren gibt, kann auf einer Spalte dieses Typs kein Index direkt erstellt werden. Für schnelle Suchen in XML-Daten kommen Ausweichlösungen infrage: die Umwandlung in einen Zeichentyp und Indexierung dieses Ausdrucks oder die Indexierung eines XPath-Ausdrucks. Die eigentliche Abfrage muss dann entsprechend auf den indexierten Ausdruck zugreifen.

Auch die Volltextsuche von PostgreSQL kann vollständige Dokumentensuchen in XML-Daten beschleunigen. Die dafür nötige Vorverarbeitungsunterstützung ist allerdings derzeit nicht Bestandteil der PostgreSQL-Distribution.

## 8.14. JSON-Typen

JSON-Datentypen speichern JSON-Daten (JavaScript Object Notation), wie in RFC 7159 beschrieben. Solche Daten können auch als `text` gespeichert werden; JSON-Datentypen haben jedoch den Vorteil, dass jeder gespeicherte Wert auf Gültigkeit nach den JSON-Regeln geprüft wird. Außerdem gibt es JSON-spezifische Funktionen und Operatoren; siehe [Abschnitt 9.16](09_Funktionen_und_Operatoren.md#916-jsonfunktionen-und-operatoren).

PostgreSQL bietet zwei Typen zur Speicherung von JSON-Daten: `json` und `jsonb`. Für effiziente Abfragemechanismen stellt PostgreSQL außerdem den in [Abschnitt 8.14.7](#8147-jsonpathtyp) beschriebenen Typ `jsonpath` bereit.

`json` und `jsonb` akzeptieren nahezu dieselben Eingabewerte. Der wichtigste praktische Unterschied ist die Effizienz. `json` speichert eine exakte Kopie des Eingabetexts; verarbeitende Funktionen müssen ihn bei jeder Ausführung erneut parsen. `jsonb` speichert Daten in einem zerlegten binären Format. Die Eingabe ist wegen zusätzlicher Konvertierung etwas langsamer, die Verarbeitung aber deutlich schneller, weil kein erneutes Parsen nötig ist. Außerdem unterstützt `jsonb` Indexierung.

Da `json` eine exakte Kopie des Eingabetexts speichert, bleiben semantisch unbedeutende Leerzeichen zwischen Tokens ebenso erhalten wie die Reihenfolge der Schlüssel in JSON-Objekten. Enthält ein Objekt denselben Schlüssel mehrfach, bleiben alle Schlüssel/Wert-Paare erhalten; die verarbeitenden Funktionen betrachten den letzten Wert als maßgeblich. `jsonb` bewahrt Leerraum und Schlüsselreihenfolge nicht und speichert doppelte Objektschlüssel nicht mehrfach. Bei doppelten Schlüsseln wird nur der letzte Wert gespeichert.

Im Allgemeinen sollten die meisten Anwendungen JSON-Daten als `jsonb` speichern, außer es gibt spezielle Anforderungen wie Altannahmen über die Reihenfolge von Objektschlüsseln.

RFC 7159 legt fest, dass JSON-Zeichenketten in UTF-8 kodiert sein sollen. JSON-Typen können der JSON-Spezifikation daher nur dann streng entsprechen, wenn die Datenbankkodierung UTF-8 ist. Zeichen, die in der Datenbankkodierung nicht darstellbar sind, können nicht direkt eingeschlossen werden; umgekehrt werden Zeichen erlaubt, die in der Datenbankkodierung darstellbar sind, aber nicht in UTF-8.

JSON-Zeichenketten dürfen Unicode-Escape-Sequenzen der Form `\uXXXX` enthalten. Die Eingabefunktion von `json` erlaubt solche Escapes unabhängig von der Datenbankkodierung und prüft nur die Syntax. Die Eingabefunktion von `jsonb` ist strenger: Sie verbietet Unicode-Escapes für Zeichen, die in der Datenbankkodierung nicht darstellbar sind, verbietet `\u0000`, weil es im PostgreSQL-Typ `text` nicht darstellbar ist, und verlangt korrekte Surrogatpaare für Zeichen außerhalb der Basic Multilingual Plane. Gültige Unicode-Escapes werden für die Speicherung in das entsprechende einzelne Zeichen umgewandelt.

> **Hinweis**
> Viele der in [Abschnitt 9.16](09_Funktionen_und_Operatoren.md#916-jsonfunktionen-und-operatoren) beschriebenen JSON-Verarbeitungsfunktionen wandeln Unicode-Escapes in normale Zeichen um und können deshalb dieselben Fehler auslösen, auch wenn ihre Eingabe vom Typ `json` und nicht `jsonb` ist. Dass die `json`-Eingabefunktion diese Prüfungen nicht ausführt, ist historisch bedingt, erlaubt aber die einfache Speicherung von JSON-Unicode-Escapes in Datenbankkodierungen, die die dargestellten Zeichen nicht unterstützen.

Beim Umwandeln textueller JSON-Eingabe in `jsonb` werden die primitiven JSON-Typen im Wesentlichen auf native PostgreSQL-Typen abgebildet, wie Tabelle 8.23 zeigt. Dadurch gelten für gültige `jsonb`-Daten einige zusätzliche Einschränkungen, die weder für `json` noch für abstraktes JSON gelten. Insbesondere lehnt `jsonb` Zahlen außerhalb des Bereichs von PostgreSQL-`numeric` ab, während `json` dies nicht tut. Solche implementierungsabhängigen Einschränkungen sind nach RFC 7159 erlaubt. Beim Austausch mit Systemen, die JSON-Zahlen etwa als IEEE-754-`double precision` darstellen, sollte das Risiko des Verlusts numerischer Genauigkeit beachtet werden.

Umgekehrt gibt es, wie die Tabelle zeigt, kleinere Einschränkungen beim Eingabeformat primitiver JSON-Typen, die nicht für die entsprechenden PostgreSQL-Typen gelten.

**Tabelle 8.23. Primitive JSON-Typen und entsprechende PostgreSQL-Typen**

| Primitiver JSON-Typ | PostgreSQL-Typ | Hinweise |
| --- | --- | --- |
| `string` | `text` | `\u0000` ist verboten, ebenso Unicode-Escapes für Zeichen, die in der Datenbankkodierung nicht verfügbar sind. |
| `number` | `numeric` | `NaN` und Unendlichkeitswerte sind verboten. |
| `boolean` | `boolean` | Nur die kleingeschriebenen Formen `true` und `false` werden akzeptiert. |
| `null` | keiner | SQL-`NULL` ist ein anderes Konzept. |

### 8.14.1. JSON-Eingabe- und Ausgabesyntax

Die Eingabe-/Ausgabesyntax der JSON-Datentypen entspricht RFC 7159.

Die folgenden Ausdrücke sind gültige `json`- oder `jsonb`-Ausdrücke:

```sql
-- Einfacher skalarer/primitiver Wert.
-- Primitive Werte können Zahlen, Zeichenketten, true, false oder null sein.
SELECT '5'::json;

-- Array mit null oder mehr Elementen; die Elemente müssen nicht denselben Typ haben.
SELECT '[1, 2, "foo", null]'::json;

-- Objekt mit Schlüssel/Wert-Paaren.
-- Objektschlüssel müssen immer Zeichenketten in Anführungszeichen sein.
SELECT '{"bar": "baz", "balance": 7.77, "active": false}'::json;

-- Arrays und Objekte können beliebig verschachtelt sein.
SELECT '{"foo": [true, "bar"], "tags": {"a": 1, "b": null}}'::json;
```

Wie oben erwähnt, gibt `json` ohne weitere Verarbeitung denselben Text aus, der eingegeben wurde, während `jsonb` semantisch unbedeutende Details wie Leerraum nicht bewahrt:

```sql
SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::json;
SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::jsonb;
```

```text
                         json
-------------------------------------------------
 {"bar": "baz", "balance": 7.77, "active":false}

                         jsonb
--------------------------------------------------
 {"bar": "baz", "active": false, "balance": 7.77}
```

Bei `jsonb` werden Zahlen entsprechend dem Verhalten des zugrunde liegenden Typs `numeric` ausgegeben. Zahlen, die mit Exponentialnotation eingegeben wurden, erscheinen daher ohne diese Schreibweise:

```sql
SELECT '{"reading": 1.230e-5}'::json, '{"reading": 1.230e-5}'::jsonb;
```

```text
         json          |          jsonb
-----------------------+-------------------------
 {"reading": 1.230e-5} | {"reading": 0.00001230}
```

Nachgestellte Nullen im Bruchteil bleiben bei `jsonb` jedoch erhalten, obwohl sie für Gleichheitsvergleiche semantisch unbedeutend sind. Eine Liste der eingebauten Funktionen und Operatoren für JSON-Werte steht in [Abschnitt 9.16](09_Funktionen_und_Operatoren.md#916-jsonfunktionen-und-operatoren).

### 8.14.2. JSON-Dokumente entwerfen

Daten als JSON darzustellen ist wesentlich flexibler als das traditionelle relationale Datenmodell; das ist besonders in Umgebungen attraktiv, in denen Anforderungen häufig wechseln. Beide Ansätze können in derselben Anwendung nebeneinander bestehen und sich ergänzen. Selbst wenn maximale Flexibilität gewünscht ist, sollten JSON-Dokumente aber eine einigermaßen feste Struktur haben. Diese Struktur wird normalerweise nicht erzwungen, obwohl sich manche Geschäftsregeln deklarativ erzwingen lassen. Eine vorhersehbare Struktur erleichtert Abfragen, die eine Menge von Dokumenten in einer Tabelle sinnvoll zusammenfassen.

JSON-Daten unterliegen beim Speichern in Tabellen denselben Überlegungen zur Nebenläufigkeitskontrolle wie andere Datentypen. Große Dokumente können zwar gespeichert werden, aber jede Aktualisierung erwirbt eine Zeilensperre auf die ganze Zeile. Es ist daher sinnvoll, JSON-Dokumente auf eine handhabbare Größe zu begrenzen, um Sperrkonflikte zwischen aktualisierenden Transaktionen zu verringern. Idealerweise stellt jedes JSON-Dokument ein atomares Datum dar, das nach den Geschäftsregeln nicht sinnvoll in kleinere, unabhängig änderbare Daten zerlegt werden kann.

### 8.14.3. `jsonb`-Enthaltensein und Existenz

Enthaltensein zu testen ist eine wichtige Fähigkeit von `jsonb`; für `json` gibt es keine entsprechende Funktionsmenge. Ein Enthaltenseinstest prüft, ob ein `jsonb`-Dokument ein anderes enthält. Die folgenden Beispiele liefern `true`, sofern nichts anderes angegeben ist:

```sql
-- Einfache skalare/primitive Werte enthalten nur denselben Wert.
SELECT '"foo"'::jsonb @> '"foo"'::jsonb;

-- Das Array rechts ist im Array links enthalten.
SELECT '[1, 2, 3]'::jsonb @> '[1, 3]'::jsonb;

-- Die Reihenfolge von Array-Elementen spielt keine Rolle.
SELECT '[1, 2, 3]'::jsonb @> '[3, 1]'::jsonb;

-- Doppelte Array-Elemente spielen ebenfalls keine Rolle.
SELECT '[1, 2, 3]'::jsonb @> '[1, 2, 2]'::jsonb;

-- Das Objekt rechts mit einem einzelnen Paar ist im Objekt links enthalten.
SELECT '{"product": "PostgreSQL", "version": 9.4, "jsonb": true}'::jsonb @> '{"version": 9.4}'::jsonb;

-- Dieses Array ist nicht enthalten, obwohl ein ähnliches Array verschachtelt vorkommt.
SELECT '[1, 2, [1, 3]]'::jsonb @> '[1, 3]'::jsonb; -- ergibt false

-- Mit einer Verschachtelungsebene ist es enthalten.
SELECT '[1, 2, [1, 3]]'::jsonb @> '[[1, 3]]'::jsonb;

-- Ebenso wird hier kein Enthaltensein gemeldet.
SELECT '{"foo": {"bar": "baz"}}'::jsonb @> '{"bar": "baz"}'::jsonb; -- ergibt false

-- Ein Schlüssel auf oberster Ebene mit leerem Objekt ist enthalten.
SELECT '{"foo": {"bar": "baz"}}'::jsonb @> '{"foo": {}}'::jsonb;
```

Allgemein muss das enthaltene Objekt in Struktur und Dateninhalt zum enthaltenden Objekt passen, möglicherweise nachdem nicht passende Array-Elemente oder Objekt-Schlüssel/Wert-Paare aus dem enthaltenden Objekt entfernt wurden. Die Reihenfolge von Array-Elementen ist dabei unbedeutend, und doppelte Array-Elemente werden effektiv nur einmal betrachtet.

Als besondere Ausnahme darf ein Array einen primitiven Wert enthalten:

```sql
-- Dieses Array enthält die primitive Zeichenkette.
SELECT '["foo", "bar"]'::jsonb @> '"bar"'::jsonb;

-- Die Ausnahme gilt nicht umgekehrt.
SELECT '"bar"'::jsonb @> '["bar"]'::jsonb; -- ergibt false
```

`jsonb` hat außerdem einen Existenzoperator. Er prüft, ob eine als `text` angegebene Zeichenkette auf oberster Ebene eines `jsonb`-Werts als Objektschlüssel oder Array-Element vorkommt. Diese Beispiele liefern `true`, sofern nichts anderes angegeben ist:

```sql
-- Zeichenkette existiert als Array-Element.
SELECT '["foo", "bar", "baz"]'::jsonb ? 'bar';

-- Zeichenkette existiert als Objektschlüssel.
SELECT '{"foo": "bar"}'::jsonb ? 'foo';

-- Objektwerte werden nicht berücksichtigt.
SELECT '{"foo": "bar"}'::jsonb ? 'bar'; -- ergibt false

-- Wie beim Enthaltensein muss die Existenz auf oberster Ebene passen.
SELECT '{"foo": {"bar": "baz"}}'::jsonb ? 'bar'; -- ergibt false

-- Eine Zeichenkette gilt als vorhanden, wenn sie einer primitiven JSON-Zeichenkette entspricht.
SELECT '"foo"'::jsonb ? 'foo';
```

JSON-Objekte eignen sich bei vielen Schlüsseln oder Elementen besser für Enthaltenseins- und Existenztests als Arrays, weil sie intern für Suche optimiert sind und nicht linear durchsucht werden müssen.

> **Tipp**
> Da JSON-Enthaltensein verschachtelt ist, kann eine passende Abfrage auf die ausdrückliche Auswahl von Unterobjekten verzichten. Angenommen, eine Spalte `doc` enthält Objekte auf oberster Ebene, von denen die meisten ein Feld `tags` mit Arrays aus Unterobjekten enthalten. Diese Abfrage findet Einträge, in denen Unterobjekte mit sowohl `"term":"paris"` als auch `"term":"food"` vorkommen, und ignoriert solche Schlüssel außerhalb des Arrays `tags`:
>
> ```sql
> SELECT doc->'site_name' FROM websites
>   WHERE doc @> '{"tags":[{"term":"paris"}, {"term":"food"}]}';
> ```
>
> Dasselbe ließe sich etwa so ausdrücken:
>
> ```sql
> SELECT doc->'site_name' FROM websites
>   WHERE doc->'tags' @> '[{"term":"paris"}, {"term":"food"}]';
> ```
>
> Dieser Ansatz ist aber weniger flexibel und oft auch weniger effizient. Der JSON-Existenzoperator ist dagegen nicht verschachtelt; er sucht den angegebenen Schlüssel oder das Array-Element nur auf oberster Ebene des JSON-Werts.

Die Operatoren für Enthaltensein und Existenz sowie alle weiteren JSON-Operatoren und -Funktionen sind in [Abschnitt 9.16](09_Funktionen_und_Operatoren.md#916-jsonfunktionen-und-operatoren) dokumentiert.

### 8.14.4. `jsonb`-Indexierung

GIN-Indizes können Schlüssel oder Schlüssel/Wert-Paare in vielen `jsonb`-Dokumenten effizient suchen. PostgreSQL stellt zwei GIN-Operatorklassen mit unterschiedlichen Leistungs- und Flexibilitätsmerkmalen bereit.

Die voreingestellte GIN-Operatorklasse für `jsonb` unterstützt Abfragen mit den Schlüssel-Existenzoperatoren `?`, `?|` und `?&`, dem Enthaltenseinsoperator `@>` sowie den `jsonpath`-Matchoperatoren `@?` und `@@`. Ein Index mit dieser Operatorklasse kann so erzeugt werden:

```sql
CREATE INDEX idxgin ON api USING GIN (jdoc);
```

Die nicht voreingestellte Operatorklasse `jsonb_path_ops` unterstützt die Schlüssel-Existenzoperatoren nicht, aber `@>`, `@?` und `@@`:

```sql
CREATE INDEX idxginp ON api USING GIN (jdoc jsonb_path_ops);
```

Angenommen, eine Tabelle speichert JSON-Dokumente aus einem Webdienst eines Drittanbieters in der Spalte `jdoc`:

```json
{
  "guid": "9c36adc1-7fb5-4d5b-83b4-90356a46061a",
  "name": "Angela Barton",
  "is_active": true,
  "company": "Magnafone",
  "address": "178 Howard Place, Gulf, Washington, 702",
  "registered": "2009-11-07T08:53:22 +08:00",
  "latitude": 19.793713,
  "longitude": 86.513373,
  "tags": ["enim", "aliquip", "qui"]
}
```

Ein GIN-Index auf dieser Spalte kann Abfragen wie diese unterstützen:

```sql
-- Dokumente finden, in denen der Schlüssel "company" den Wert "Magnafone" hat.
SELECT jdoc->'guid', jdoc->'name' FROM api
WHERE jdoc @> '{"company": "Magnafone"}';
```

Diese Abfrage kann denselben Index dagegen nicht verwenden, weil der indexierbare Operator `?` nicht direkt auf die indexierte Spalte `jdoc` angewendet wird:

```sql
-- Dokumente finden, in denen "tags" das Array-Element "qui" enthält.
SELECT jdoc->'guid', jdoc->'name' FROM api
WHERE jdoc -> 'tags' ? 'qui';
```

Mit einem passenden Ausdrucksindex kann diese Abfrage dennoch indexiert werden:

```sql
CREATE INDEX idxgintags ON api USING GIN ((jdoc -> 'tags'));
```

Dann wird `jdoc -> 'tags' ? 'qui'` als Anwendung des indexierbaren Operators `?` auf den indexierten Ausdruck erkannt. Weitere Informationen zu Ausdrucksindizes stehen in [Abschnitt 11.7](11_Indizes.md#117-indizes-auf-ausdrücken).

Eine andere Abfragestrategie nutzt Enthaltensein:

```sql
-- Dokumente finden, in denen "tags" das Array-Element "qui" enthält.
SELECT jdoc->'guid', jdoc->'name' FROM api
WHERE jdoc @> '{"tags": ["qui"]}';
```

Ein einfacher GIN-Index auf `jdoc` kann diese Abfrage unterstützen. Er speichert jedoch Kopien jedes Schlüssels und jedes Werts aus `jdoc`, während der Ausdrucksindex nur Daten unterhalb des Schlüssels `tags` speichert. Ein einfacher Index ist flexibler, gezielte Ausdrucksindizes sind aber wahrscheinlich kleiner und schneller.

GIN-Indizes unterstützen außerdem die Operatoren `@?` und `@@`, die `jsonpath`-Matching ausführen:

```sql
SELECT jdoc->'guid', jdoc->'name' FROM api
WHERE jdoc @? '$.tags[*] ? (@ == "qui")';

SELECT jdoc->'guid', jdoc->'name' FROM api
WHERE jdoc @@ '$.tags[*] == "qui"';
```

Für diese Operatoren extrahiert ein GIN-Index Klauseln der Form `accessors_chain == constant` aus dem `jsonpath`-Muster und sucht anhand der darin genannten Schlüssel und Werte. Die Zugriffskette kann `.key`, `[*]` und `[index]` enthalten. Die Operatorklasse `jsonb_ops` unterstützt zusätzlich `.*` und `.**`; `jsonb_path_ops` tut dies nicht.

Obwohl `jsonb_path_ops` nur Abfragen mit `@>`, `@?` und `@@` unterstützt, hat diese Operatorklasse deutliche Leistungsvorteile gegenüber `jsonb_ops`: Der Index ist meist viel kleiner und die Suche spezifischer, besonders wenn Abfragen Schlüssel enthalten, die in den Daten häufig vorkommen.

Technisch erzeugt `jsonb_ops` unabhängige Indexeinträge für jeden Schlüssel und jeden Wert, während `jsonb_path_ops` nur für jeden Wert Indexeinträge erzeugt. Jeder `jsonb_path_ops`-Indexeintrag ist im Wesentlichen ein Hash aus dem Wert und den Schlüsseln, die zu ihm führen. Für `{"foo": {"bar": "baz"}}` würde also ein einzelner Indexeintrag entstehen, der `foo`, `bar` und `baz` in den Hash einbezieht. Eine Enthaltenseinsabfrage auf genau diese Struktur wird dadurch sehr spezifisch; es gibt aber keine Möglichkeit, nur nach dem Vorkommen des Schlüssels `foo` zu suchen. Ein `jsonb_ops`-Index würde getrennte Einträge für `foo`, `bar` und `baz` erzeugen und für die Enthaltenseinsabfrage nach Zeilen suchen, die alle drei Einträge enthalten.

Ein Nachteil von `jsonb_path_ops` ist, dass für JSON-Strukturen ohne Werte, etwa `{"a": {}}`, keine Indexeinträge erzeugt werden. Suchen nach Dokumenten, die eine solche Struktur enthalten, erfordern einen vollständigen Indexscan und sind langsam. Für Anwendungen, die solche Suchen häufig ausführen, ist `jsonb_path_ops` daher ungeeignet.

`jsonb` unterstützt auch B-Tree- und Hash-Indizes. Diese sind meist nur dann nützlich, wenn vollständige JSON-Dokumente auf Gleichheit geprüft werden sollen. Die B-Tree-Sortierung für `jsonb`-Werte ist selten praktisch relevant, der Vollständigkeit halber gilt:

```text
Object > Array > Boolean > Number > String > null
Object mit n Paaren > Objekt mit n - 1 Paaren
Array mit n Elementen > Array mit n - 1 Elementen
```

Eine historische Ausnahme ist, dass ein leeres Array auf oberster Ebene kleiner als `null` sortiert. Objekte mit gleicher Paaranzahl werden in Speicherreihenfolge der Schlüssel verglichen:

```text
key-1, value-1, key-2, ...
```

Kürzere Schlüssel werden vor längeren gespeichert; dadurch können unintuitive Ergebnisse entstehen:

```text
{ "aa": 1, "c": 1} > {"b": 1, "d": 1}
```

Arrays mit gleicher Elementanzahl werden in Elementreihenfolge verglichen. Primitive JSON-Werte werden nach denselben Vergleichsregeln wie die zugrunde liegenden PostgreSQL-Datentypen verglichen; Zeichenketten verwenden die Standardkollation der Datenbank.

### 8.14.5. `jsonb`-Subskripte

Der Datentyp `jsonb` unterstützt arrayartige Subskriptausdrücke zum Extrahieren und Ändern von Elementen. Verschachtelte Werte werden durch verkettete Subskriptausdrücke bezeichnet, nach denselben Regeln wie das Pfadargument von `jsonb_set`. Ist ein `jsonb`-Wert ein Array, beginnen numerische Subskripte bei null; negative ganze Zahlen zählen vom letzten Array-Element rückwärts. Slice-Ausdrücke werden nicht unterstützt. Das Ergebnis eines Subskriptausdrucks hat immer den Typ `jsonb`.

`UPDATE`-Anweisungen können Subskripte in der `SET`-Klausel verwenden, um `jsonb`-Werte zu ändern. Subskriptpfade müssen für alle betroffenen Werte durchlaufbar sein, soweit sie existieren. Der Pfad `val['a']['b']['c']` kann beispielsweise bis `c` durchlaufen werden, wenn `val`, `val['a']` und `val['a']['b']` jeweils Objekte sind. Nicht vorhandene Zwischenobjekte werden als leere Objekte angelegt. Ist ein vorhandener Zwischenwert dagegen kein Objekt, sondern etwa eine Zeichenkette, Zahl oder JSON-`null`, wird ein Fehler ausgelöst und die Transaktion abgebrochen.

Beispiele:

```sql
-- Objektwert nach Schlüssel extrahieren.
SELECT ('{"a": 1}'::jsonb)['a'];

-- Verschachtelten Objektwert über einen Schlüsselpfad extrahieren.
SELECT ('{"a": {"b": {"c": 1}}}'::jsonb)['a']['b']['c'];

-- Array-Element nach Index extrahieren.
SELECT ('[1, "2", null]'::jsonb)[1];

-- Objektwert nach Schlüssel aktualisieren. Der zugewiesene Wert muss ebenfalls jsonb sein.
UPDATE table_name SET jsonb_field['key'] = '1';

-- Fehler, wenn jsonb_field['a']['b'] in einer Zeile kein Objekt ist.
UPDATE table_name SET jsonb_field['a']['b']['c'] = '1';

-- Datensätze mit Subskripten in WHERE filtern.
SELECT * FROM table_name WHERE jsonb_field['key'] = '"value"';
```

Zuweisung per Subskript behandelt einige Randfälle anders als `jsonb_set`. Ist der Quellwert `NULL`, wird er so behandelt, als wäre er ein leerer JSON-Wert des durch den Subskriptschlüssel implizierten Typs:

```sql
-- War jsonb_field NULL, ist es danach {"a": 1}.
UPDATE table_name SET jsonb_field['a'] = '1';

-- War jsonb_field NULL, ist es danach [1].
UPDATE table_name SET jsonb_field[0] = '1';
```

Ist ein Array für den angegebenen Index zu kurz, werden `null`-Elemente angehängt, bis der Index erreichbar ist:

```sql
-- War jsonb_field [], ist es danach [null, null, 2];
-- war jsonb_field [0], ist es danach [0, null, 2].
UPDATE table_name SET jsonb_field[2] = '2';
```

Ein `jsonb`-Wert akzeptiert Zuweisungen an nicht vorhandene Subskriptpfade, solange das letzte vorhandene durchlaufene Element ein Objekt oder Array ist, wie es der entsprechende Subskript nahelegt. Verschachtelte Array- und Objektstrukturen werden angelegt; Arrays werden dabei bei Bedarf mit `null` aufgefüllt:

```sql
-- War jsonb_field {}, ist es danach {"a": [{"b": 1}]}.
UPDATE table_name SET jsonb_field['a'][0]['b'] = '1';

-- War jsonb_field [], ist es danach [null, {"a": 1}].
UPDATE table_name SET jsonb_field[1]['a'] = '1';
```

### 8.14.6. Transformationen

Zusätzliche Erweiterungen implementieren Transformationen für den Typ `jsonb` in verschiedenen prozeduralen Sprachen.

Die Erweiterungen für PL/Perl heißen `jsonb_plperl` und `jsonb_plperlu`. Mit ihnen werden `jsonb`-Werte passend auf Perl-Arrays, Hashes und Skalare abgebildet.

Die Erweiterung für PL/Python heißt `jsonb_plpython3u`. Mit ihr werden `jsonb`-Werte passend auf Python-Dictionaries, Listen und Skalare abgebildet.

Von diesen Erweiterungen gilt `jsonb_plperl` als vertrauenswürdig, kann also von Nicht-Superusern installiert werden, die das Privileg `CREATE` auf der aktuellen Datenbank haben. Die übrigen Erweiterungen erfordern Superuser-Rechte.

### 8.14.7. `jsonpath`-Typ

Der Typ `jsonpath` implementiert in PostgreSQL Unterstützung für die SQL/JSON-Pfadsprache, um JSON-Daten effizient abzufragen. Er stellt eine binäre Darstellung eines geparsten SQL/JSON-Pfadausdrucks bereit, der festlegt, welche Elemente die Pfad-Engine aus den JSON-Daten für weitere Verarbeitung mit SQL/JSON-Abfragefunktionen abrufen soll.

Die Semantik von SQL/JSON-Pfadprädikaten und -Operatoren folgt im Allgemeinen SQL. Gleichzeitig verwendet die SQL/JSON-Pfadsyntax einige JavaScript-Konventionen, um den Umgang mit JSON-Daten natürlicher zu machen:

- Punkt (`.`) wird für Member-Zugriff verwendet.
- Eckige Klammern (`[]`) werden für Array-Zugriff verwendet.
- SQL/JSON-Arrays sind nullbasiert, anders als reguläre SQL-Arrays, die bei 1 beginnen.

Numerische Literale in SQL/JSON-Pfadausdrücken folgen JavaScript-Regeln, die sich in einigen Details sowohl von SQL als auch von JSON unterscheiden. SQL/JSON-Pfad erlaubt etwa `.1` und `1.`, die in JSON ungültig sind. Nichtdezimale Ganzzahlliterale und Unterstrich-Trennzeichen werden unterstützt, zum Beispiel `1_000_000`, `0x1EEE_FFFF`, `0o273` und `0b100101`. Direkt nach dem Radix-Präfix darf in SQL/JSON-Pfad kein Unterstrich stehen.

Ein SQL/JSON-Pfadausdruck wird in einer SQL-Abfrage normalerweise als SQL-Zeichenkettenliteral geschrieben. Er muss daher in einfache Anführungszeichen eingeschlossen werden; einfache Anführungszeichen innerhalb des Werts müssen verdoppelt werden (siehe Abschnitt 4.1.2.1). Einige Pfadausdrücke enthalten ihrerseits Zeichenkettenliterale. Diese eingebetteten Literale folgen JavaScript/ECMAScript-Konventionen: Sie stehen in doppelten Anführungszeichen, und Backslash-Escapes können Zeichen darstellen. Ein doppeltes Anführungszeichen wird als `\"` geschrieben, ein Backslash als `\\`. Weitere Sequenzen sind `\b`, `\f`, `\n`, `\r`, `\t`, `\v`, `\xNN`, `\uNNNN` und `\u{N...}`.

Ein Pfadausdruck besteht aus einer Folge von Pfadelementen. Dazu gehören:

- Pfadliterale primitiver JSON-Typen: Unicode-Text, Zahlen, `true`, `false` oder `null`.
- Pfadvariablen aus Tabelle 8.24.
- Zugriffoperatoren aus Tabelle 8.25.
- `jsonpath`-Operatoren und -Methoden aus Abschnitt 9.16.2.3.
- Klammern, mit denen Filterausdrücke angegeben oder die Auswertungsreihenfolge festgelegt wird.

Details zur Verwendung von `jsonpath`-Ausdrücken mit SQL/JSON-Abfragefunktionen stehen in Abschnitt 9.16.2.

**Tabelle 8.24. `jsonpath`-Variablen**

| Variable | Beschreibung |
| --- | --- |
| `$` | Variable für den abgefragten JSON-Wert, das Kontextelement. |
| `$varname` | Benannte Variable. Ihr Wert kann über den Parameter `vars` mehrerer JSON-Verarbeitungsfunktionen gesetzt werden; siehe Tabelle 9.51. |
| `@` | Variable für das Ergebnis der Pfadauswertung in Filterausdrücken. |

**Tabelle 8.25. `jsonpath`-Zugriffoperatoren**

| Zugriffoperator | Beschreibung |
| --- | --- |
| `.key` / `."$varname"` | Member-Zugriff auf den Objekteintrag mit dem angegebenen Schlüssel. Entspricht der Schlüsselname einer benannten Variablen, die mit `$` beginnt, oder nicht den JavaScript-Regeln für Bezeichner, muss er als Zeichenkettenliteral in doppelte Anführungszeichen gesetzt werden. |
| `.*` | Wildcard-Member-Zugriff, der alle Werte der Member auf oberster Ebene des aktuellen Objekts zurückgibt. |
| `.**` | Rekursiver Wildcard-Member-Zugriff über alle Ebenen der JSON-Hierarchie. PostgreSQL-Erweiterung des SQL/JSON-Standards. |
| `.**{level}` / `.**{start_level to end_level}` | Wie `.**`, aber nur für angegebene Hierarchieebenen. Ebene 0 entspricht dem aktuellen Objekt; mit `last` kann die tiefste Ebene adressiert werden. PostgreSQL-Erweiterung. |
| `[subscript, ...]` | Array-Elementzugriff. `subscript` kann ein einzelner Index oder ein Bereich `start_index to end_index` sein. Index 0 bezeichnet das erste Array-Element; `last` bezeichnet das letzte. |
| `[*]` | Wildcard-Array-Zugriff, der alle Array-Elemente zurückgibt. |

## 8.15. Arrays

PostgreSQL erlaubt Tabellenspalten als variable mehrdimensionale Arrays. Arrays können für eingebaute und benutzerdefinierte Basistypen, Aufzählungstypen, zusammengesetzte Typen, Bereichstypen und Domänen erzeugt werden.

### 8.15.1. Deklaration von Array-Typen

Zur Veranschaulichung wird eine Tabelle mit Array-Spalten erzeugt:

```sql
CREATE TABLE sal_emp (
    name            text,
    pay_by_quarter  integer[],
    schedule        text[][]
);
```

Ein Array-Datentyp entsteht, indem an den Typnamen der Elemente eckige Klammern (`[]`) angehängt werden. Die Tabelle `sal_emp` enthält eine Spalte `name` vom Typ `text`, ein eindimensionales Array von `integer` namens `pay_by_quarter` für das Gehalt nach Quartalen und ein zweidimensionales `text`-Array namens `schedule` für den Wochenplan.

Die Syntax von `CREATE TABLE` erlaubt es, Array-Größen anzugeben:

```sql
CREATE TABLE tictactoe (
    squares   integer[3][3]
);
```

Die aktuelle Implementierung ignoriert solche Größenbeschränkungen allerdings; das Verhalten entspricht Arrays ohne angegebene Länge. Auch die deklarierte Dimensionszahl wird nicht erzwungen. Arrays eines bestimmten Elementtyps gelten unabhängig von Größe oder Dimensionszahl als derselbe Typ. Angaben zu Größe oder Dimensionszahl sind daher nur Dokumentation und beeinflussen das Laufzeitverhalten nicht.

Für eindimensionale Arrays kann auch die SQL-Standard-Syntax mit `ARRAY` verwendet werden:

```sql
pay_by_quarter integer ARRAY[4],
pay_by_quarter integer ARRAY,
```

PostgreSQL erzwingt die Größenangabe auch in dieser Schreibweise nicht.

### 8.15.2. Eingabe von Array-Werten

Ein Array-Literal wird geschrieben, indem die Elementwerte in geschweifte Klammern eingeschlossen und durch Trennzeichen getrennt werden. Werte können in doppelte Anführungszeichen gesetzt werden; sie müssen es, wenn sie Kommas oder geschweifte Klammern enthalten. Das allgemeine Format lautet:

```text
'{ val1 delim val2 delim ... }'
```

`delim` ist das Trennzeichen des Elementtyps aus dessen `pg_type`-Eintrag. Fast alle Standardtypen verwenden ein Komma, der Typ `box` verwendet ein Semikolon. Jeder `val` ist entweder eine Konstante des Elementtyps oder ein Unterarray. Beispiel für ein zweidimensionales 3-mal-3-Array:

```text
'{{1,2,3},{4,5,6},{7,8,9}}'
```

Um ein Array-Element auf `NULL` zu setzen, schreiben Sie `NULL` als Elementwert. Soll dagegen die tatsächliche Zeichenkette `NULL` gespeichert werden, muss sie in doppelte Anführungszeichen gesetzt werden.

Diese Array-Konstanten sind ein Spezialfall der generischen Typkonstanten aus Abschnitt 4.1.2.7: Die Konstante wird zunächst als Zeichenkette behandelt und an die Eingabekonvertierung des Arrays übergeben. Eine explizite Typangabe kann nötig sein.

Beispiele für `INSERT`:

```sql
INSERT INTO sal_emp
    VALUES ('Bill',
    '{10000, 10000, 10000, 10000}',
    '{{"meeting", "lunch"}, {"training", "presentation"}}');

INSERT INTO sal_emp
    VALUES ('Carol',
    '{20000, 25000, 25000, 25000}',
    '{{"breakfast", "consulting"}, {"meeting", "lunch"}}');
```

Das Ergebnis sieht so aus:

```sql
SELECT * FROM sal_emp;
```

```text
 name  |       pay_by_quarter       |                 schedule
-------+----------------------------+-------------------------------------------
 Bill  | {10000,10000,10000,10000}  | {{meeting,lunch},{training,presentation}}
 Carol | {20000,25000,25000,25000}  | {{breakfast,consulting},{meeting,lunch}}
(2 rows)
```

Mehrdimensionale Arrays müssen in jeder Dimension passende Ausdehnungen haben. Ein Fehler entsteht zum Beispiel hier:

```sql
INSERT INTO sal_emp
    VALUES ('Bill',
    '{10000, 10000, 10000, 10000}',
    '{{"meeting", "lunch"}, {"meeting"}}');
```

```text
ERROR:  malformed array literal: "{{"meeting", "lunch"}, {"meeting"}}"
DETAIL:  Multidimensional arrays must have sub-arrays with matching dimensions.
```

Auch die `ARRAY`-Konstruktorsyntax kann verwendet werden:

```sql
INSERT INTO sal_emp
    VALUES ('Bill',
    ARRAY[10000, 10000, 10000, 10000],
    ARRAY[['meeting', 'lunch'], ['training', 'presentation']]);

INSERT INTO sal_emp
    VALUES ('Carol',
    ARRAY[20000, 25000, 25000, 25000],
    ARRAY[['breakfast', 'consulting'], ['meeting', 'lunch']]);
```

Array-Elemente sind dabei normale SQL-Konstanten oder -Ausdrücke. Zeichenkettenliterale stehen also in einfachen Anführungszeichen, nicht in doppelten wie beim Array-Literal. Die `ARRAY`-Konstruktorsyntax wird in Abschnitt 4.2.12 genauer beschrieben.

### 8.15.3. Zugriff auf Arrays

Ein einzelnes Array-Element wird mit einem Subskript in eckigen Klammern gelesen. Diese Abfrage findet Mitarbeitende, deren Gehalt sich im zweiten Quartal geändert hat:

```sql
SELECT name FROM sal_emp WHERE pay_by_quarter[1] <> pay_by_quarter[2];
```

```text
 name
-------
 Carol
(1 row)
```

Standardmäßig verwendet PostgreSQL für Arrays eine einsbasierte Nummerierung: Ein Array mit `n` Elementen beginnt bei `array[1]` und endet bei `array[n]`.

Das Gehalt im dritten Quartal wird so abgerufen:

```sql
SELECT pay_by_quarter[3] FROM sal_emp;
```

Rechteckige Slices oder Unterarrays werden mit `lower-bound:upper-bound` angegeben:

```sql
SELECT schedule[1:2][1:1] FROM sal_emp WHERE name = 'Bill';
```

```text
        schedule
------------------------
 {{meeting},{training}}
(1 row)
```

Wenn eine Dimension als Slice geschrieben wird, also einen Doppelpunkt enthält, werden alle Dimensionen als Slices behandelt. Eine Dimension mit nur einer einzelnen Zahl wird dann als Bereich von 1 bis zu dieser Zahl interpretiert:

```sql
SELECT schedule[1:2][2] FROM sal_emp WHERE name = 'Bill';
```

```text
                 schedule
-------------------------------------------
 {{meeting,lunch},{training,presentation}}
(1 row)
```

Um Verwechslungen mit dem Nicht-Slice-Fall zu vermeiden, ist es am besten, Slice-Syntax für alle Dimensionen zu verwenden, also etwa `[1:2][1:1]` statt `[2][1:1]`.

Untere oder obere Grenzen eines Slices können weggelassen werden; die fehlende Grenze wird durch die entsprechende Subskriptgrenze des Arrays ersetzt:

```sql
SELECT schedule[:2][2:] FROM sal_emp WHERE name = 'Bill';
SELECT schedule[:][1:1] FROM sal_emp WHERE name = 'Bill';
```

Ein Array-Subskriptausdruck liefert `NULL`, wenn das Array selbst oder einer der Subskriptausdrücke `NULL` ist. `NULL` wird auch zurückgegeben, wenn ein Subskript außerhalb der Array-Grenzen liegt; das ist kein Fehler. Ein Array-Verweis mit falscher Subskriptanzahl liefert ebenfalls `NULL`.

Ein Array-Slice-Ausdruck liefert ebenfalls `NULL`, wenn das Array oder ein Subskriptausdruck `NULL` ist. In anderen Fällen, etwa wenn der gewünschte Slice vollständig außerhalb der aktuellen Array-Grenzen liegt, liefert ein Slice-Ausdruck ein leeres, nulldimensionales Array statt `NULL`. Überlappt der gewünschte Slice teilweise, wird er stillschweigend auf den überlappenden Bereich reduziert.

Die aktuellen Dimensionen eines Arrays liefert `array_dims`:

```sql
SELECT array_dims(schedule) FROM sal_emp WHERE name = 'Carol';
```

```text
 array_dims
------------
 [1:2][1:2]
(1 row)
```

`array_upper` und `array_lower` liefern die obere bzw. untere Grenze einer angegebenen Dimension, `array_length` deren Länge, und `cardinality` die Gesamtzahl der Elemente über alle Dimensionen hinweg:

```sql
SELECT array_upper(schedule, 1) FROM sal_emp WHERE name = 'Carol';
SELECT array_length(schedule, 1) FROM sal_emp WHERE name = 'Carol';
SELECT cardinality(schedule) FROM sal_emp WHERE name = 'Carol';
```

### 8.15.4. Arrays ändern

Ein Array-Wert kann vollständig ersetzt werden:

```sql
UPDATE sal_emp SET pay_by_quarter = '{25000,25000,27000,27000}'
    WHERE name = 'Carol';

UPDATE sal_emp SET pay_by_quarter = ARRAY[25000,25000,27000,27000]
    WHERE name = 'Carol';
```

Ein einzelnes Element oder ein Slice kann aktualisiert werden:

```sql
UPDATE sal_emp SET pay_by_quarter[4] = 15000
    WHERE name = 'Bill';

UPDATE sal_emp SET pay_by_quarter[1:2] = '{27000,27000}'
    WHERE name = 'Carol';
```

Slice-Syntax mit ausgelassener unterer oder oberer Grenze ist auch bei Aktualisierungen möglich, aber nur bei Array-Werten, die weder `NULL` noch nulldimensional sind.

Ein gespeichertes Array kann vergrößert werden, indem einem noch nicht vorhandenen Element ein Wert zugewiesen wird. Dazwischenliegende Positionen werden mit `NULL` gefüllt. Wird zum Beispiel `myarray[6]` zugewiesen, während `myarray` vier Elemente hat, enthält `myarray[5]` anschließend `NULL`. Diese Vergrößerung ist derzeit nur für eindimensionale Arrays erlaubt.

Subskriptzuweisungen können Arrays mit nicht einsbasierten Subskripten erzeugen, etwa durch Zuweisung an `myarray[-2:7]`.

Neue Array-Werte können auch mit dem Verkettungsoperator `||` erzeugt werden:

```sql
SELECT ARRAY[1,2] || ARRAY[3,4];
SELECT ARRAY[5,6] || ARRAY[[1,2],[3,4]];
```

```text
 ?column?
-----------
 {1,2,3,4}

      ?column?
---------------------
 {{5,6},{1,2},{3,4}}
```

Der Verkettungsoperator kann ein einzelnes Element an Anfang oder Ende eines eindimensionalen Arrays setzen. Er akzeptiert außerdem zwei N-dimensionale Arrays oder ein N-dimensionales und ein (N+1)-dimensionales Array. Beim Anhängen eines einzelnen Elements behält das Ergebnis die untere Grenze des Array-Operanden:

```sql
SELECT array_dims(1 || '[0:1]={2,3}'::int[]);
SELECT array_dims(ARRAY[1,2] || 3);
```

Bei der Verkettung zweier Arrays gleicher Dimensionszahl behält das Ergebnis die untere Grenze der äußeren Dimension des linken Operanden. Wird ein N-dimensionales Array an ein (N+1)-dimensionales Array angehängt, entspricht das dem Element-Array-Fall.

Alternativ können `array_prepend`, `array_append` und `array_cat` verwendet werden:

```sql
SELECT array_prepend(1, ARRAY[2,3]);
SELECT array_append(ARRAY[1,2], 3);
SELECT array_cat(ARRAY[1,2], ARRAY[3,4]);
SELECT array_cat(ARRAY[[1,2],[3,4]], ARRAY[5,6]);
```

In einfachen Fällen ist der Operator `||` diesen Funktionen vorzuziehen. Weil `||` jedoch für mehrere Fälle überladen ist, kann eine Funktion helfen, Mehrdeutigkeit zu vermeiden. Der Parser interpretiert eine untypisierte Konstante neben einem `integer[]` zum Beispiel bevorzugt ebenfalls als `integer[]`:

```sql
SELECT ARRAY[1, 2] || '{3, 4}'; -- das untypisierte Literal gilt als Array
SELECT ARRAY[1, 2] || '7';      -- ebenso dieses, daher Fehler
SELECT ARRAY[1, 2] || NULL;     -- auch ein undecorated NULL
SELECT array_append(ARRAY[1, 2], NULL);
```

Ist diese Heuristik unerwünscht, kann die Konstante auf den Elementtyp gecastet oder direkt `array_append` verwendet werden.

### 8.15.5. In Arrays suchen

Um in einem Array nach einem Wert zu suchen, muss jeder Wert geprüft werden. Bei bekannter Array-Größe kann das manuell geschehen:

```sql
SELECT * FROM sal_emp WHERE pay_by_quarter[1] = 10000 OR
                            pay_by_quarter[2] = 10000 OR
                            pay_by_quarter[3] = 10000 OR
                            pay_by_quarter[4] = 10000;
```

Für große Arrays ist das umständlich und bei unbekannter Größe unbrauchbar. [Abschnitt 9.25](09_Funktionen_und_Operatoren.md#925-zeilen-und-arrayvergleiche) beschreibt Alternativen. Die obige Abfrage kann so geschrieben werden:

```sql
SELECT * FROM sal_emp WHERE 10000 = ANY (pay_by_quarter);
```

Zeilen, bei denen alle Werte `10000` sind, finden Sie mit:

```sql
SELECT * FROM sal_emp WHERE 10000 = ALL (pay_by_quarter);
```

Auch `generate_subscripts` kann verwendet werden:

```sql
SELECT * FROM
   (SELECT pay_by_quarter,
           generate_subscripts(pay_by_quarter, 1) AS s
      FROM sal_emp) AS foo
 WHERE pay_by_quarter[s] = 10000;
```

Die Funktion ist in Tabelle 9.70 beschrieben.

Mit dem Operator `&&` kann geprüft werden, ob sich zwei Arrays überlappen:

```sql
SELECT * FROM sal_emp WHERE pay_by_quarter && ARRAY[10000];
```

Dieser und weitere Array-Operatoren sind in [Abschnitt 9.19](09_Funktionen_und_Operatoren.md#919-arrayfunktionen-und-operatoren) beschrieben. Passende Indizes können solche Abfragen beschleunigen; siehe [Abschnitt 11.2](11_Indizes.md#112-indextypen).

Mit `array_position` und `array_positions` kann ebenfalls nach bestimmten Werten gesucht werden. Die erste Funktion liefert den Subskript des ersten Vorkommens, die zweite ein Array mit allen Subskripten:

```sql
SELECT array_position(ARRAY['sun','mon','tue','wed','thu','fri','sat'], 'mon');
SELECT array_positions(ARRAY[1, 4, 3, 1, 3, 4, 2, 1], 1);
```

```text
 array_position
----------------
              2

 array_positions
-----------------
 {1,4,8}
```

> **Tipp**
> Arrays sind keine Mengen. Die Suche nach bestimmten Array-Elementen kann ein Hinweis auf ein ungünstiges Datenbankdesign sein. Erwägen Sie eine separate Tabelle mit einer Zeile pro Element. Das lässt sich leichter durchsuchen und skaliert bei vielen Elementen wahrscheinlich besser.

### 8.15.6. Array-Eingabe- und Ausgabesyntax

Die externe Textdarstellung eines Array-Werts besteht aus Elementen, die nach den Ein-/Ausgaberegeln des Elementtyps interpretiert werden, sowie Markierung der Array-Struktur. Geschweifte Klammern umschließen den Array-Wert, Trennzeichen stehen zwischen benachbarten Elementen. Das Trennzeichen ist meist ein Komma; der Elementtyp legt es über `typdelim` fest. In mehrdimensionalen Arrays erhält jede Dimension eine eigene Ebene geschweifter Klammern.

Die Array-Ausgaberoutine setzt Elementwerte in doppelte Anführungszeichen, wenn sie leere Zeichenketten sind, geschweifte Klammern, Trennzeichen, doppelte Anführungszeichen, Backslashes oder Leerraum enthalten oder dem Wort `NULL` entsprechen. Eingebettete doppelte Anführungszeichen und Backslashes werden mit Backslash geschützt.

Standardmäßig ist die untere Grenze jeder Array-Dimension 1. Andere untere Grenzen werden durch explizite Subskriptbereiche vor dem Array-Inhalt dargestellt:

```sql
SELECT f1[1][-2][3] AS e1, f1[1][-1][5] AS e2
FROM (SELECT '[1:1][-2:-1][3:5]={{{1,2,3},{4,5,6}}}'::int[] AS f1) AS ss;
```

```text
 e1 | e2
----+----
  1 |  6
(1 row)
```

Explizite Dimensionen erscheinen in der Ausgabe nur, wenn mindestens eine untere Grenze von 1 abweicht.

Wird für ein Element `NULL` geschrieben, gilt es als `NULL`. Anführungszeichen oder Backslashes verhindern diese Interpretation und erlauben die tatsächliche Zeichenkette `NULL`. Aus Kompatibilitätsgründen mit PostgreSQL-Versionen vor 8.2 kann der Parameter `array_nulls` ausgeschaltet werden, um die Erkennung von `NULL` als Nullwert zu unterdrücken.

Beim Schreiben eines Array-Werts müssen Elemente in doppelte Anführungszeichen gesetzt werden, wenn sie sonst den Parser verwirren würden, etwa wegen geschweifter Klammern, Kommas, Trennzeichen des Datentyps, doppelter Anführungszeichen, Backslashes oder führendem bzw. nachgestelltem Leerraum. Leere Zeichenketten und die Zeichenkette `NULL` müssen ebenfalls gequotet werden. Innerhalb eines gequoteten Array-Elements wird ein doppeltes Anführungszeichen oder ein Backslash durch vorangestellten Backslash geschrieben. Alternativ kann Backslash-Escaping auch ohne äußere Anführungszeichen verwendet werden.

Leerraum vor einer linken oder nach einer rechten geschweiften Klammer sowie vor oder nach einzelnen Elementzeichenketten wird ignoriert. Leerraum innerhalb doppelt gequoteter Elemente oder zwischen anderen Nicht-Leerraumzeichen eines Elements wird nicht ignoriert.

> **Tipp**
> Die `ARRAY`-Konstruktorsyntax (siehe Abschnitt 4.2.12) ist in SQL-Befehlen oft einfacher zu verwenden als Array-Literal-Syntax. In `ARRAY` werden einzelne Elementwerte genauso geschrieben, als wären sie keine Array-Elemente.

## 8.16. Zusammengesetzte Typen

Ein zusammengesetzter Typ stellt die Struktur einer Zeile oder eines Records dar: im Wesentlichen eine Liste von Feldnamen und deren Datentypen. PostgreSQL erlaubt zusammengesetzte Typen in vielen Situationen, in denen einfache Typen verwendet werden können; zum Beispiel kann eine Tabellenspalte einen zusammengesetzten Typ haben.

### 8.16.1. Deklaration zusammengesetzter Typen

Zwei einfache Beispiele:

```sql
CREATE TYPE complex AS (
    r       double precision,
    i       double precision
);

CREATE TYPE inventory_item AS (
    name            text,
    supplier_id     integer,
    price           numeric
);
```

Die Syntax ähnelt `CREATE TABLE`, aber es können nur Feldnamen und Typen angegeben werden; Einschränkungen wie `NOT NULL` sind derzeit nicht möglich. Das Schlüsselwort `AS` ist erforderlich.

Die Typen können in Tabellen und Funktionen verwendet werden:

```sql
CREATE TABLE on_hand (
    item      inventory_item,
    count     integer
);

INSERT INTO on_hand VALUES (ROW('fuzzy dice', 42, 1.99), 1000);

CREATE FUNCTION price_extension(inventory_item, integer) RETURNS numeric
AS 'SELECT $1.price * $2' LANGUAGE SQL;

SELECT price_extension(item, 10) FROM on_hand;
```

Beim Erzeugen einer Tabelle wird automatisch auch ein zusammengesetzter Typ gleichen Namens erzeugt, der den Zeilentyp der Tabelle darstellt. Einschränkungen der Tabellendefinition gelten jedoch nicht für Werte dieses zusammengesetzten Typs außerhalb der Tabelle. Soll das erzwungen werden, kann eine Domäne über dem zusammengesetzten Typ mit entsprechenden `CHECK`-Constraints angelegt werden.

### 8.16.2. Zusammengesetzte Werte konstruieren

Ein zusammengesetztes Literal wird geschrieben, indem Feldwerte in Klammern eingeschlossen und durch Kommas getrennt werden:

```text
'( val1 , val2 , ... )'
```

Beispiel:

```text
'("fuzzy dice",42,1.99)'
```

Ein Feld wird zu `NULL`, indem an seiner Position nichts geschrieben wird:

```text
'("fuzzy dice",42,)'
```

Eine leere Zeichenkette statt `NULL` wird mit doppelten Anführungszeichen geschrieben:

```text
'("",42,)'
```

Die `ROW`-Ausdruckssyntax ist meistens einfacher, weil weniger Quote-Ebenen nötig sind:

```sql
ROW('fuzzy dice', 42, 1.99)
ROW('', 42, NULL)
```

Bei mehr als einem Feld ist das Schlüsselwort `ROW` optional:

```sql
('fuzzy dice', 42, 1.99)
('', 42, NULL)
```

Die `ROW`-Syntax wird in Abschnitt 4.2.13 genauer beschrieben.

### 8.16.3. Zugriff auf zusammengesetzte Typen

Auf ein Feld einer zusammengesetzten Spalte wird mit Punkt und Feldname zugegriffen. Häufig sind Klammern nötig, damit der Parser die Spalte nicht mit einem Tabellennamen verwechselt:

```sql
SELECT (item).name FROM on_hand WHERE (item).price > 9.99;
```

Mit Tabellenname:

```sql
SELECT (on_hand.item).name FROM on_hand WHERE (on_hand.item).price > 9.99;
```

Dasselbe gilt für zusammengesetzte Ergebnisse von Funktionen:

```sql
SELECT (my_func(...)).field FROM ...;
```

Ohne zusätzliche Klammern entsteht ein Syntaxfehler. Der spezielle Feldname `*` bedeutet „alle Felder“, wie in [Abschnitt 8.16.5](#8165-zusammengesetzte-typen-in-abfragen-verwenden) erklärt.

### 8.16.4. Zusammengesetzte Typen ändern

Eine ganze zusammengesetzte Spalte kann eingefügt oder aktualisiert werden:

```sql
INSERT INTO mytab (complex_col) VALUES((1.1,2.2));
UPDATE mytab SET complex_col = ROW(1.1,2.2) WHERE ...;
```

Ein einzelnes Unterfeld kann aktualisiert werden:

```sql
UPDATE mytab SET complex_col.r = (complex_col).r + 1 WHERE ...;
```

Direkt nach `SET` steht der Spaltenname ohne Klammern; in Ausdrücken rechts vom Gleichheitszeichen sind Klammern um die zusammengesetzte Spalte nötig.

Auch `INSERT` kann Unterfelder als Ziel angeben:

```sql
INSERT INTO mytab (complex_col.r, complex_col.i) VALUES(1.1, 2.2);
```

Nicht angegebene Unterfelder werden mit `NULL` gefüllt.

### 8.16.5. Zusammengesetzte Typen in Abfragen verwenden

In PostgreSQL ist ein Verweis auf einen Tabellennamen oder Alias in einer Abfrage effektiv ein Verweis auf den zusammengesetzten Wert der aktuellen Tabellenzeile:

```sql
SELECT c FROM inventory_item c;
```

Das Ergebnis ist eine einzelne zusammengesetzte Spalte:

```text
------------------------
 ("fuzzy dice",42,1.99)
(1 row)
```

Einfache Namen werden zuerst mit Spaltennamen abgeglichen und erst danach mit Tabellennamen; das Beispiel funktioniert also nur, wenn keine Spalte `c` existiert.

Die Schreibweise `table_name.column_name` kann als Feldauswahl aus dem zusammengesetzten Wert der aktuellen Zeile verstanden werden. Bei `c.*` wird der zusammengesetzte Wert in einzelne Spalten expandiert:

```sql
SELECT c.* FROM inventory_item c;
```

```text
    name    | supplier_id | price
------------+-------------+-------
 fuzzy dice |          42 |  1.99
(1 row)
```

Das entspricht:

```sql
SELECT c.name, c.supplier_id, c.price FROM inventory_item c;
```

PostgreSQL wendet diese Expansion auf jeden zusammengesetzten Ausdruck an. Wenn `myfunc()` einen zusammengesetzten Typ mit den Spalten `a`, `b` und `c` zurückgibt, sind diese Abfragen gleichwertig:

```sql
SELECT (myfunc(x)).* FROM some_table;
SELECT (myfunc(x)).a, (myfunc(x)).b, (myfunc(x)).c FROM some_table;
```

> **Tipp**
> PostgreSQL transformiert Spaltenexpansion tatsächlich in die zweite Form. In diesem Beispiel würde `myfunc()` also pro Zeile dreimal aufgerufen. Bei teuren Funktionen lässt sich das vermeiden:
>
> ```sql
> SELECT m.* FROM some_table, LATERAL myfunc(x) AS m;
> ```
>
> Die Funktion steht dann als `LATERAL`-Eintrag in `FROM` und wird pro Zeile nur einmal aufgerufen. Das Schlüsselwort `LATERAL` ist hier optional, macht aber deutlich, dass die Funktion `x` aus `some_table` erhält.

`composite_value.*` führt nur auf oberster Ebene einer `SELECT`-Ausgabeliste, einer `RETURNING`-Liste, einer `VALUES`-Klausel oder eines Zeilenkonstruktors zur Spaltenexpansion. In anderen Kontexten bedeutet `.*` einfach „alle Spalten“ und ergibt denselben zusammengesetzten Wert:

```sql
SELECT somefunc(c.*) FROM inventory_item c;
SELECT somefunc(c) FROM inventory_item c;
```

Beide Abfragen übergeben die aktuelle Zeile als einzelnes zusammengesetztes Argument. `.*` ist hier zwar wirkungslos, macht aber die Absicht klar und vermeidet Mehrdeutigkeit.

Diese Abfragen sortieren alle nach dem zusammengesetzten Zeilenwert:

```sql
SELECT * FROM inventory_item c ORDER BY c;
SELECT * FROM inventory_item c ORDER BY c.*;
SELECT * FROM inventory_item c ORDER BY ROW(c.*);
```

Enthielte `inventory_item` eine Spalte `c`, wäre die erste Form anders. Mit den gezeigten Spaltennamen sind diese Formen ebenfalls gleichwertig:

```sql
SELECT * FROM inventory_item c ORDER BY ROW(c.name, c.supplier_id, c.price);
SELECT * FROM inventory_item c ORDER BY (c.name, c.supplier_id, c.price);
```

Für zusammengesetzte Werte kann auch funktionale Schreibweise zur Feldauswahl verwendet werden: `field(table)` und `table.field` sind austauschbar.

```sql
SELECT c.name FROM inventory_item c WHERE c.price > 1000;
SELECT name(c) FROM inventory_item c WHERE price(c) > 1000;
```

Eine Funktion mit einem einzelnen Argument eines zusammengesetzten Typs kann ebenfalls in beiden Schreibweisen aufgerufen werden:

```sql
SELECT somefunc(c) FROM inventory_item c;
SELECT somefunc(c.*) FROM inventory_item c;
SELECT c.somefunc FROM inventory_item c;
```

Diese Gleichwertigkeit ermöglicht „berechnete Felder“ über Funktionen auf zusammengesetzten Typen.

> **Tipp**
> Wegen dieses Verhaltens ist es unklug, einer Funktion mit einem einzelnen zusammengesetzten Argument denselben Namen zu geben wie einem Feld dieses Typs. Bei Mehrdeutigkeit gewinnt in Feldsyntax die Feldnameninterpretation; in Funktionsaufrufsyntax gewinnt die Funktion. In älteren PostgreSQL-Versionen vor 11 konnte man die Funktionsinterpretation erzwingen, indem der Funktionsname schemaqualifiziert wurde, also `schema.func(compositevalue)`.

### 8.16.6. Eingabe- und Ausgabesyntax zusammengesetzter Typen

Die externe Textdarstellung eines zusammengesetzten Werts besteht aus den Feldwerten nach den I/O-Regeln ihrer jeweiligen Typen sowie der Strukturmarkierung: Klammern um den gesamten Wert und Kommas zwischen benachbarten Feldern. Leerraum außerhalb der Klammern wird ignoriert; innerhalb der Klammern gehört er zum Feldwert und kann je nach Feldtyp bedeutsam sein.

Beim Schreiben können einzelne Feldwerte in doppelte Anführungszeichen gesetzt werden. Das ist nötig, wenn der Feldwert sonst den Parser verwirren würde, etwa wegen Klammern, Kommas, doppelter Anführungszeichen oder Backslashes. In einem gequoteten Feldwert wird ein doppeltes Anführungszeichen oder ein Backslash durch vorangestellten Backslash geschrieben. Alternativ können solche Zeichen mit Backslash-Escaping ohne äußere Anführungszeichen geschützt werden.

Ein vollständig leeres Feld steht für `NULL`. Eine leere Zeichenkette wird als `""` geschrieben.

Die Ausgaberoutine setzt Feldwerte in doppelte Anführungszeichen, wenn sie leere Zeichenketten sind oder Klammern, Kommas, doppelte Anführungszeichen, Backslashes oder Leerraum enthalten. Eingebettete doppelte Anführungszeichen und Backslashes werden verdoppelt.

> **Hinweis**
> Was Sie in einem SQL-Befehl schreiben, wird zuerst als Zeichenkettenliteral und danach als zusammengesetzter Wert interpretiert. Dadurch verdoppelt sich die nötige Zahl der Backslashes, sofern Escape-String-Syntax verwendet wird. Um ein Textfeld mit einem doppelten Anführungszeichen und einem Backslash einzufügen, müsste man zum Beispiel schreiben:
>
> ```sql
> INSERT ... VALUES ('("\"\\")');
> ```
>
> Dollar-Quoting (siehe Abschnitt 4.1.2.4) kann helfen, zusätzliche Backslashes zu vermeiden.

> **Tipp**
> Die `ROW`-Konstruktorsyntax ist in SQL-Befehlen meist einfacher als die zusammengesetzte Literal-Syntax. In `ROW` werden einzelne Feldwerte so geschrieben, als wären sie keine Teile eines zusammengesetzten Werts.

## 8.17. Bereichstypen

Bereichstypen stellen einen Wertebereich eines Elementtyps dar, der als Subtyp des Bereichs bezeichnet wird. Bereiche von `timestamp` können beispielsweise Reservierungszeiten eines Besprechungsraums darstellen. Der Typ heißt dann `tsrange`, und `timestamp` ist der Subtyp. Der Subtyp muss eine totale Ordnung haben, damit eindeutig ist, ob Werte innerhalb, vor oder nach dem Bereich liegen.

Bereichstypen sind nützlich, weil sie viele Elementwerte in einem einzelnen Bereichswert darstellen und Konzepte wie überlappende Bereiche klar ausdrücken. Zeit- und Datumsbereiche für Terminplanung sind das offensichtlichste Beispiel; Preisbereiche, Messbereiche und Ähnliches können ebenfalls sinnvoll sein.

Zu jedem Bereichstyp gibt es einen Multibereichstyp. Ein Multibereich ist eine geordnete Liste nicht zusammenhängender, nicht leerer und nicht nullwertiger Bereiche. Die meisten Bereichsoperatoren funktionieren auch für Multibereiche; zusätzlich gibt es eigene Funktionen für Multibereiche.

### 8.17.1. Eingebaute Bereichs- und Multibereichstypen

PostgreSQL enthält diese eingebauten Bereichstypen:

- `int4range`: Bereich von `integer`; `int4multirange`: entsprechender Multibereich.
- `int8range`: Bereich von `bigint`; `int8multirange`: entsprechender Multibereich.
- `numrange`: Bereich von `numeric`; `nummultirange`: entsprechender Multibereich.
- `tsrange`: Bereich von `timestamp without time zone`; `tsmultirange`: entsprechender Multibereich.
- `tstzrange`: Bereich von `timestamp with time zone`; `tstzmultirange`: entsprechender Multibereich.
- `daterange`: Bereich von `date`; `datemultirange`: entsprechender Multibereich.

Eigene Bereichstypen können mit `CREATE TYPE` definiert werden.

### 8.17.2. Beispiele

```sql
CREATE TABLE reservation (room int, during tsrange);
INSERT INTO reservation VALUES
    (1108, '[2010-01-01 14:30, 2010-01-01 15:30)');

-- Enthaltensein
SELECT int4range(10, 20) @> 3;

-- Überlappung
SELECT numrange(11.1, 22.2) && numrange(20.0, 30.0);

-- Obere Grenze extrahieren
SELECT upper(int8range(15, 25));

-- Schnittmenge berechnen
SELECT int4range(10, 20) * int4range(15, 25);

-- Ist der Bereich leer?
SELECT isempty(numrange(1, 5));
```

Vollständige Listen der Operatoren und Funktionen für Bereichstypen stehen in Tabelle 9.58 und Tabelle 9.60.

### 8.17.3. Inklusive und exklusive Grenzen

Jeder nicht leere Bereich hat zwei Grenzen: eine untere und eine obere. Alle Punkte zwischen diesen Werten gehören zum Bereich. Eine inklusive Grenze bedeutet, dass der Grenzpunkt selbst eingeschlossen ist; eine exklusive Grenze schließt ihn aus.

In der Textform eines Bereichs steht `[` für eine inklusive untere Grenze und `(` für eine exklusive untere Grenze. Entsprechend steht `]` für eine inklusive obere Grenze und `)` für eine exklusive obere Grenze. Weitere Details stehen in [Abschnitt 8.17.5](#8175-bereichseingabe-und-ausgabe).

Die Funktionen `lower_inc` und `upper_inc` prüfen, ob die untere bzw. obere Grenze eines Bereichswerts inklusiv ist.

### 8.17.4. Unendliche oder unbegrenzte Bereiche

Die untere Grenze eines Bereichs kann weggelassen werden; dann enthält der Bereich alle Werte kleiner als die obere Grenze, etwa `(,3]`. Entsprechend enthält ein Bereich ohne obere Grenze alle Werte größer als die untere Grenze. Fehlen beide Grenzen, umfasst der Bereich alle Werte des Elementtyps. Eine fehlende Grenze wird automatisch exklusiv behandelt.

Unendliche Werte eines Elementtyps sind nicht dasselbe wie fehlende Bereichsgrenzen. Manche Elementtypen, etwa `timestamp`, haben besondere Werte wie `infinity`; diese können explizit als Grenzen verwendet werden. Ein Bereich mit fehlender oberer Grenze enthält einen solchen Wert `infinity`; ein Bereich mit oberer Grenze `infinity` enthält ihn nicht, wenn die Grenze exklusiv ist.

### 8.17.5. Bereichseingabe und -ausgabe

Die Eingabe für einen Bereichswert muss einem der folgenden Muster entsprechen:

```text
(lower-bound,upper-bound)
(lower-bound,upper-bound]
[lower-bound,upper-bound)
[lower-bound,upper-bound]
empty
```

Die Klammern geben an, ob die untere und obere Grenze exklusiv oder inklusiv sind. `empty` bezeichnet einen leeren Bereich.

`lower-bound` und `upper-bound` können Zeichenketten sein, die für den Subtyp gültig sind, oder leer bleiben, um eine fehlende Grenze anzugeben. Jede Grenze kann mit doppelten Anführungszeichen gequotet werden. Das ist nötig, wenn der Wert Klammern, Kommas, doppelte Anführungszeichen, Backslashes oder führenden bzw. nachgestellten Leerraum enthält. Innerhalb gequoteter Werte werden doppelte Anführungszeichen und Backslashes mit Backslash geschützt. Alternativ kann Backslash-Escaping ohne äußere Anführungszeichen verwendet werden.

Leerraum vor und nach dem Bereichswert wird ignoriert. Leerraum innerhalb der Klammern gilt als Teil der unteren oder oberen Grenze.

Beispiele:

```text
-- umfasst 3 nicht
(,3)

-- umfasst 3
(,3]

-- umfasst nur den einzelnen Punkt 4
[4,4]

-- enthält keine Punkte und wird als empty normalisiert
[4,4)
```

### 8.17.6. Bereiche und Multibereiche konstruieren

Jeder Bereichstyp hat eine Konstruktorfunktion mit demselben Namen wie der Bereichstyp. Bei zwei Argumenten werden untere und obere Grenze angegeben, wobei die Form `[)` verwendet wird:

```sql
SELECT numrange(1.0, 14.0);
```

Mit einem dritten Argument kann die Grenzform gewählt werden:

```sql
SELECT numrange(1.0, 14.0, '(]');
```

Fehlt eine Grenze, kann `NULL` übergeben werden:

```sql
SELECT numrange(NULL, 2.2);
```

Jeder Multibereichstyp hat ebenfalls eine Konstruktorfunktion gleichen Namens. Sie nimmt null oder mehr Bereichswerte entgegen:

```sql
SELECT nummultirange();
SELECT nummultirange(numrange(1.0, 14.0));
SELECT nummultirange(numrange(1.0, 14.0), numrange(20.0, 25.0));
```

### 8.17.7. Diskrete Bereichstypen

Ein diskreter Bereichstyp hat einen klaren Schritt zwischen aufeinanderfolgenden Werten, etwa `integer` oder `date`. Bei kontinuierlichen Typen wie `numeric` oder `timestamp` gibt es keinen solchen nächsten Wert. PostgreSQL kann bei diskreten Bereichstypen unterschiedliche Schreibweisen in eine kanonische Form bringen. Beispielsweise sind `[1,7]`, `[1,8)` und `(0,8)` für `int4range` gleichwertig und werden in dieselbe kanonische Form normalisiert.

Beim Definieren eigener diskreter Bereichstypen sollte eine Kanonisierungsfunktion angegeben werden, damit semantisch gleiche Werte identisch dargestellt werden. Außerdem sollte eine Differenzfunktion angegeben werden, damit GiST- und SP-GiST-Indizes effizient arbeiten können.

### 8.17.8. Neue Bereichstypen definieren

Benutzer können eigene Bereichstypen definieren. Meist genügt es, den Namen des Bereichstyps und den Subtyp anzugeben:

```sql
CREATE TYPE floatrange AS RANGE (
    subtype = float8,
    subtype_diff = float8mi
);
```

Da `float8` keine sinnvolle diskrete Schrittweite hat, ist keine Kanonisierungsfunktion nötig. Wenn der Subtyp keine standardmäßige B-Tree-Operatorklasse besitzt, muss eine passende Subtyp-Operatorklasse angegeben werden. Weitere Informationen stehen bei `CREATE TYPE`.

### 8.17.9. Indexierung

Für Tabellenspalten mit Bereichstypen können GiST- und SP-GiST-Indizes erstellt werden. Für Multibereichsspalten können GiST-Indizes erstellt werden:

```sql
CREATE INDEX reservation_idx ON reservation USING GIST (during);
```

Ein GiST- oder SP-GiST-Index auf Bereichen kann Abfragen mit den Bereichsoperatoren `=`, `&&`, `<@`, `@>`, `<<`, `>>`, `-|-`, `&<` und `&>` beschleunigen. GiST-Indizes auf Multibereichen unterstützen die entsprechenden Multibereichsoperatoren sowie passende Operatoren zwischen Bereichen und Multibereichen. Weitere Informationen stehen in Tabelle 9.58.

B-Tree- und Hash-Indizes können ebenfalls für Bereichsspalten erstellt werden. Praktisch nützlich ist dabei hauptsächlich Gleichheit. Es gibt zwar eine B-Tree-Sortierung mit `<` und `>`, sie ist aber eher intern nützlich und normalerweise fachlich wenig sinnvoll.

### 8.17.10. Constraints auf Bereichen

Während `UNIQUE` eine natürliche Einschränkung für skalare Werte ist, eignet es sich für Bereichstypen meist nicht. Häufig sind Ausschluss-Constraints passender, mit denen sich etwa „nicht überlappend“ ausdrücken lässt:

```sql
CREATE TABLE reservation (
    during tsrange,
    EXCLUDE USING GIST (during WITH &&)
);
```

Dieser Constraint verhindert gleichzeitige überlappende Werte:

```sql
INSERT INTO reservation VALUES
    ('[2010-01-01 11:30, 2010-01-01 15:00)');

INSERT INTO reservation VALUES
    ('[2010-01-01 14:45, 2010-01-01 15:45)');
```

```text
ERROR:  conflicting key value violates exclusion constraint "reservation_during_excl"
DETAIL:  Key (during)=(['2010-01-01 14:45:00','2010-01-01 15:45:00')) conflicts with existing key (during)=(['2010-01-01 11:30:00','2010-01-01 15:00:00')).
```

Mit der Erweiterung `btree_gist` können Ausschluss-Constraints auch auf einfache skalare Datentypen angewendet und mit Bereichsausschlüssen kombiniert werden:

```sql
CREATE EXTENSION btree_gist;
CREATE TABLE room_reservation (
    room text,
    during tsrange,
    EXCLUDE USING GIST (room WITH =, during WITH &&)
);

INSERT INTO room_reservation VALUES
    ('123A', '[2010-01-01 14:00, 2010-01-01 15:00)');

INSERT INTO room_reservation VALUES
    ('123A', '[2010-01-01 14:30, 2010-01-01 15:30)');

INSERT INTO room_reservation VALUES
    ('123B', '[2010-01-01 14:30, 2010-01-01 15:30)');
```

Die zweite Einfügung scheitert, weil Raum und Zeitraum kollidieren; die dritte ist zulässig, weil der Raum ein anderer ist.

## 8.18. Domänentypen

Eine Domäne ist ein benutzerdefinierter Datentyp, der auf einem anderen zugrunde liegenden Typ basiert. Optional kann sie Constraints haben, die gültige Werte auf eine Teilmenge dessen beschränken, was der zugrunde liegende Typ erlauben würde. Andernfalls verhält sie sich wie der zugrunde liegende Typ; Operatoren und Funktionen für den Basistyp funktionieren also auch mit der Domäne. Der zugrunde liegende Typ kann ein eingebauter oder benutzerdefinierter Basistyp, ein Aufzählungstyp, Array-Typ, zusammengesetzter Typ, Bereichstyp oder eine andere Domäne sein.

Beispiel für eine Domäne über Ganzzahlen, die nur positive Werte erlaubt:

```sql
CREATE DOMAIN posint AS integer CHECK (VALUE > 0);
CREATE TABLE mytable (id posint);
INSERT INTO mytable VALUES(1);   -- funktioniert
INSERT INTO mytable VALUES(-1);  -- schlägt fehl
```

Wenn ein Operator oder eine Funktion des zugrunde liegenden Typs auf einen Domänenwert angewendet wird, wird der Domänenwert automatisch auf den zugrunde liegenden Typ herabgestuft. Das Ergebnis von `mytable.id - 1` gilt daher als `integer`, nicht als `posint`. Mit `(mytable.id - 1)::posint` kann das Ergebnis zurück in `posint` gecastet werden; dabei werden die Domänenconstraints erneut geprüft. Das Zuweisen eines Werts des zugrunde liegenden Typs an ein Feld oder eine Variable des Domänentyps ist ohne expliziten Cast erlaubt, prüft aber ebenfalls die Constraints.

Weitere Informationen stehen bei `CREATE DOMAIN`.

## 8.19. Objektbezeichnertypen

Objektbezeichner (OIDs) werden intern von PostgreSQL als Primärschlüssel verschiedener Systemtabellen verwendet. Der Typ `oid` stellt einen Objektbezeichner dar. Daneben gibt es mehrere Aliastypen für `oid`, deren Namen mit `reg` beginnen. Tabelle 8.26 gibt einen Überblick.

Der Typ `oid` ist derzeit als vorzeichenlose 4-Byte-Ganzzahl implementiert. Er reicht daher nicht aus, um datenbankweite Eindeutigkeit in großen Datenbanken oder sogar in sehr großen einzelnen Tabellen zu garantieren.

`oid` selbst bietet außer Vergleich nur wenige Operationen. Er kann jedoch in `integer` gecastet und dann mit den Standard-Ganzzahloperatoren bearbeitet werden. Dabei ist Vorsicht wegen möglicher Verwechslungen zwischen vorzeichenbehafteter und vorzeichenloser Darstellung nötig.

OID-Aliastypen haben außer spezialisierten Ein- und Ausgaberoutinen keine eigenen Operationen. Diese Routinen können symbolische Namen für Systemobjekte akzeptieren und anzeigen statt der rohen numerischen Werte von `oid`. Beispiel:

```sql
SELECT * FROM pg_attribute WHERE attrelid = 'mytable'::regclass;
```

statt:

```sql
SELECT * FROM pg_attribute
  WHERE attrelid = (SELECT oid FROM pg_class WHERE relname = 'mytable');
```

Die zweite Form ist nur scheinbar einfach; bei mehreren Tabellen namens `mytable` in verschiedenen Schemas wäre eine deutlich kompliziertere Unterabfrage nötig. Die Eingabekonvertierung von `regclass` berücksichtigt den Schemasuchpfad und tut automatisch das Richtige. Ebenso ist ein Cast einer Tabellen-OID nach `regclass` praktisch für symbolische Ausgabe.

**Tabelle 8.26. Objektbezeichnertypen**

| Name | Verweist auf | Beschreibung | Beispielwert |
| --- | --- | --- | --- |
| `oid` | beliebig | numerischer Objektbezeichner | `564182` |
| `regclass` | `pg_class` | Relationsname | `pg_type` |
| `regcollation` | `pg_collation` | Kollationsname | `"POSIX"` |
| `regconfig` | `pg_ts_config` | Textsuchkonfiguration | `english` |
| `regdictionary` | `pg_ts_dict` | Textsuchwörterbuch | `simple` |
| `regnamespace` | `pg_namespace` | Namensraumname | `pg_catalog` |
| `regoper` | `pg_operator` | Operatorname | `+` |
| `regoperator` | `pg_operator` | Operator mit Argumenttypen | `*(integer,integer)` oder `-(NONE,integer)` |
| `regproc` | `pg_proc` | Funktionsname | `sum` |
| `regprocedure` | `pg_proc` | Funktion mit Argumenttypen | `sum(int4)` |
| `regrole` | `pg_authid` | Rollenname | `smithee` |
| `regtype` | `pg_type` | Datentypname | `integer` |

Alle OID-Aliastypen für Objekte in Namensräumen akzeptieren schemaqualifizierte Namen. Bei der Ausgabe wird ein schemaqualifizierter Name verwendet, wenn das Objekt ohne Qualifikation im aktuellen Suchpfad nicht gefunden würde. `regproc` und `regoper` akzeptieren nur eindeutige, nicht überladene Namen; meist sind `regprocedure` oder `regoperator` sinnvoller. Bei `regoperator` werden unäre Operatoren mit `NONE` für den unbenutzten Operanden bezeichnet.

Die Eingabefunktionen erlauben Leerraum zwischen Tokens und falten Großbuchstaben außerhalb doppelter Anführungszeichen in Kleinbuchstaben, um SQL-Bezeichnerregeln zu ähneln. Die Ausgabefunktionen setzen bei Bedarf doppelte Anführungszeichen, damit das Ergebnis ein gültiger SQL-Bezeichner ist.

Viele eingebaute PostgreSQL-Funktionen akzeptieren die OID einer Tabelle oder eines anderen Datenbankobjekts und sind der Bequemlichkeit halber mit `regclass` oder dem passenden OID-Aliastyp deklariert. Sie müssen die OID also nicht selbst nachschlagen:

```text
nextval('foo')            arbeitet auf Sequenz foo
nextval('FOO')            ebenso
nextval('"Foo"')          arbeitet auf Sequenz Foo
nextval('myschema.foo')   arbeitet auf myschema.foo
nextval('"myschema".foo') ebenso
nextval('foo')            sucht foo im Suchpfad
```

> **Hinweis**
> Ein undekoriertes Zeichenkettenliteral als Argument einer solchen Funktion wird zu einer Konstante des Typs `regclass` oder eines passenden Aliastyps. Da dies letztlich eine OID ist, verweist sie auch nach späterem Umbenennen oder Verschieben des Objekts weiter auf dasselbe Objekt. Dieses frühe Binden ist für Spaltenvorgaben und Views meist erwünscht. Für spätes Binden zur Laufzeit erzwingen Sie eine `text`-Konstante:
>
> ```sql
> nextval('foo'::text)
> ```
>
> Die Funktion `to_regclass()` und verwandte Funktionen können ebenfalls Laufzeit-Lookups durchführen; siehe Tabelle 9.76.

Ein praktisches Beispiel für `regclass` ist das Nachschlagen der OID einer Tabelle aus `information_schema`, dessen Views solche OIDs nicht direkt liefern. Für `pg_relation_size()` ist dies korrekt:

```sql
SELECT table_schema, table_name,
       pg_relation_size((quote_ident(table_schema) || '.' ||
                         quote_ident(table_name))::regclass)
FROM information_schema.tables
WHERE ...;
```

`quote_ident()` sorgt bei Bedarf für doppelte Anführungszeichen. Die scheinbar einfachere Form

```sql
SELECT pg_relation_size(table_name)
FROM information_schema.tables
WHERE ...;
```

ist nicht empfehlenswert, weil sie für Tabellen außerhalb des Suchpfads oder mit quotierungsbedürftigen Namen fehlschlägt.

Eine weitere Eigenschaft der meisten OID-Aliastypen ist das Erzeugen von Abhängigkeiten. Kommt eine Konstante eines solchen Typs in einem gespeicherten Ausdruck vor, etwa einer Spaltenvorgabe oder View, entsteht eine Abhängigkeit zum referenzierten Objekt. `nextval('my_seq'::regclass)` verhindert daher, dass die Sequenz `my_seq` gelöscht wird, solange die Vorgabe existiert. `nextval('my_seq'::text)` erzeugt keine solche Abhängigkeit. `regrole` ist eine Ausnahme: Konstanten dieses Typs sind in gespeicherten Ausdrücken nicht erlaubt.

Ein weiterer vom System verwendeter Bezeichnertyp ist `xid`, der Transaktionsbezeichner. Er ist der Datentyp der Systemspalten `xmin` und `xmax`. Transaktionsbezeichner sind 32-Bit-Werte. In manchen Kontexten wird die 64-Bit-Variante `xid8` verwendet; anders als `xid` steigt `xid8` innerhalb der Lebensdauer eines Datenbankclusters streng monoton und wird nicht wiederverwendet. Siehe [Abschnitt 67.1](67_Transaktionsverarbeitung.md#671-transaktionen-und-identifikatoren).

`cid` ist der Befehlsbezeichner und Datentyp der Systemspalten `cmin` und `cmax`; auch er ist 32 Bit groß. `tid` ist der Tupelbezeichner oder Zeilenbezeichner und Datentyp der Systemspalte `ctid`. Eine Tuple-ID ist ein Paar aus Blocknummer und Tupelindex innerhalb des Blocks und bezeichnet die physische Position einer Zeile in ihrer Tabelle.

Systemspalten werden in [Abschnitt 5.6](05_Datendefinition.md#56-systemspalten) genauer erklärt.

## 8.20. `pg_lsn`-Typ

Der Datentyp `pg_lsn` speichert LSN-Daten (Log Sequence Number), also Zeiger auf Positionen im WAL. Er ist eine Darstellung von `XLogRecPtr` und ein interner Systemtyp von PostgreSQL.

Intern ist eine LSN eine 64-Bit-Ganzzahl, die eine Byteposition im Write-Ahead-Log-Strom darstellt. Ausgegeben wird sie als zwei hexadezimale Zahlen mit jeweils bis zu 8 Stellen, getrennt durch einen Schrägstrich, zum Beispiel `16/B374D848`. Der Typ `pg_lsn` unterstützt die Standardvergleichsoperatoren wie `=` und `>`. Zwei LSNs können mit `-` subtrahiert werden; das Ergebnis ist die Zahl der Bytes zwischen diesen WAL-Positionen. Außerdem können Bytes mit `+(pg_lsn,numeric)` und `-(pg_lsn,numeric)` zu einer LSN addiert bzw. von ihr subtrahiert werden. Die berechnete LSN muss im Bereich des Typs `pg_lsn` liegen, also zwischen `0/0` und `FFFFFFFF/FFFFFFFF`.

## 8.21. Pseudotypen

Das PostgreSQL-Typsystem enthält mehrere besondere Einträge, die gemeinsam Pseudotypen genannt werden. Ein Pseudotyp kann nicht als Spaltendatentyp verwendet werden, aber er kann den Argument- oder Ergebnistyp einer Funktion deklarieren. Jeder verfügbare Pseudotyp ist in Situationen nützlich, in denen das Verhalten einer Funktion nicht einfach dem Entgegennehmen oder Zurückgeben eines Werts eines bestimmten SQL-Datentyps entspricht. Tabelle 8.27 listet die vorhandenen Pseudotypen.

**Tabelle 8.27. Pseudotypen**

| Name | Beschreibung |
| --- | --- |
| `any` | Funktion akzeptiert einen beliebigen Eingabedatentyp. |
| `anyelement` | Funktion akzeptiert einen beliebigen Datentyp; siehe Abschnitt 36.2.5. |
| `anyarray` | Funktion akzeptiert einen beliebigen Array-Datentyp; siehe Abschnitt 36.2.5. |
| `anynonarray` | Funktion akzeptiert einen beliebigen Nicht-Array-Datentyp; siehe Abschnitt 36.2.5. |
| `anyenum` | Funktion akzeptiert einen beliebigen Aufzählungstyp; siehe Abschnitt 36.2.5 und [Abschnitt 8.7](08_Datentypen.md#87-aufzählungstypen). |
| `anyrange` | Funktion akzeptiert einen beliebigen Bereichstyp; siehe Abschnitt 36.2.5 und [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen). |
| `anymultirange` | Funktion akzeptiert einen beliebigen Multibereichstyp; siehe Abschnitt 36.2.5 und [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen). |
| `anycompatible` | Funktion akzeptiert einen beliebigen Datentyp, wobei mehrere Argumente automatisch auf einen gemeinsamen Datentyp befördert werden; siehe Abschnitt 36.2.5. |
| `anycompatiblearray` | Funktion akzeptiert einen beliebigen Array-Datentyp mit automatischer Beförderung mehrerer Argumente auf einen gemeinsamen Datentyp; siehe Abschnitt 36.2.5. |
| `anycompatiblenonarray` | Funktion akzeptiert einen beliebigen Nicht-Array-Datentyp mit automatischer Beförderung mehrerer Argumente auf einen gemeinsamen Datentyp; siehe Abschnitt 36.2.5. |
| `anycompatiblerange` | Funktion akzeptiert einen beliebigen Bereichstyp mit automatischer Beförderung mehrerer Argumente auf einen gemeinsamen Datentyp; siehe Abschnitt 36.2.5 und [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen). |
| `anycompatiblemultirange` | Funktion akzeptiert einen beliebigen Multibereichstyp mit automatischer Beförderung mehrerer Argumente auf einen gemeinsamen Datentyp; siehe Abschnitt 36.2.5 und [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen). |
| `cstring` | Funktion akzeptiert oder gibt eine nullterminierte C-Zeichenkette zurück. |
| `internal` | Funktion akzeptiert oder gibt einen serverinternen Datentyp zurück. |
| `language_handler` | Call-Handler einer prozeduralen Sprache wird mit Rückgabetyp `language_handler` deklariert. |
| `fdw_handler` | Handler eines Foreign Data Wrappers wird mit Rückgabetyp `fdw_handler` deklariert. |
| `table_am_handler` | Handler einer Tabellenzugriffsmethode wird mit Rückgabetyp `table_am_handler` deklariert. |
| `index_am_handler` | Handler einer Indexzugriffsmethode wird mit Rückgabetyp `index_am_handler` deklariert. |
| `tsm_handler` | Handler einer Tablesample-Methode wird mit Rückgabetyp `tsm_handler` deklariert. |
| `record` | Funktion nimmt einen nicht näher bestimmten Zeilentyp entgegen oder gibt ihn zurück. |
| `trigger` | Triggerfunktion wird mit Rückgabetyp `trigger` deklariert. |
| `event_trigger` | Event-Trigger-Funktion wird mit Rückgabetyp `event_trigger` deklariert. |
| `pg_ddl_command` | Darstellung von DDL-Befehlen, die Event-Triggern zur Verfügung steht. |
| `void` | Funktion gibt keinen Wert zurück. |
| `unknown` | Noch nicht aufgelöster Typ, zum Beispiel eines undekorierten Zeichenkettenliterals. |

In C geschriebene Funktionen, ob eingebaut oder dynamisch geladen, können so deklariert werden, dass sie einen dieser Pseudotypen akzeptieren oder zurückgeben. Der Autor der Funktion muss sicherstellen, dass sie bei Verwendung eines Pseudotyps sicher arbeitet.

Funktionen in prozeduralen Sprachen können Pseudotypen nur verwenden, soweit die jeweilige Sprache es erlaubt. Derzeit verbieten die meisten prozeduralen Sprachen Pseudotypen als Argumenttypen und erlauben nur `void` und `record` als Ergebnistypen, zusätzlich `trigger` oder `event_trigger`, wenn die Funktion als Trigger oder Event-Trigger verwendet wird. Manche Sprachen unterstützen außerdem polymorphe Funktionen mit den oben gezeigten polymorphen Pseudotypen; diese werden in Abschnitt 36.2.5 ausführlich beschrieben.

Der Pseudotyp `internal` deklariert Funktionen, die nur intern vom Datenbanksystem aufgerufen werden sollen, nicht direkt aus SQL. Hat eine Funktion mindestens ein Argument des Typs `internal`, kann sie nicht aus SQL aufgerufen werden. Um die Typsicherheit dieser Einschränkung zu erhalten, gilt die wichtige Regel: Deklarieren Sie keine Funktion mit Rückgabetyp `internal`, wenn sie nicht mindestens ein Argument vom Typ `internal` hat.
