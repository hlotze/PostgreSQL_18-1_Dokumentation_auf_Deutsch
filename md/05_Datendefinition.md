# 5. Datendefinition

Dieses Kapitel behandelt, wie die Datenbankstrukturen erstellt werden, die Ihre Daten aufnehmen. In einer relationalen Datenbank werden die Rohdaten in Tabellen gespeichert; daher widmet sich der größte Teil dieses Kapitels der Erklärung, wie Tabellen erstellt und geändert werden und welche Funktionen verfügbar sind, um zu steuern, welche Daten in den Tabellen gespeichert werden. Anschließend besprechen wir, wie Tabellen in Schemas organisiert werden können und wie Privilegien für Tabellen vergeben werden. Zum Schluss werfen wir einen kurzen Blick auf weitere Funktionen, die die Datenspeicherung beeinflussen, etwa Vererbung, Tabellenpartitionierung, Sichten, Funktionen und Trigger.

## 5.1. Tabellengrundlagen

Eine Tabelle in einer relationalen Datenbank ähnelt einer Tabelle auf Papier: Sie besteht aus Zeilen und Spalten. Anzahl und Reihenfolge der Spalten sind festgelegt, und jede Spalte hat einen Namen. Die Anzahl der Zeilen ist veränderlich; sie spiegelt wider, wie viele Daten zu einem bestimmten Zeitpunkt gespeichert sind. SQL garantiert keine Reihenfolge der Zeilen in einer Tabelle. Beim Lesen einer Tabelle erscheinen die Zeilen in einer nicht festgelegten Reihenfolge, sofern nicht ausdrücklich eine Sortierung angefordert wird. Das wird in [Kapitel 7](07_Abfragen.md) behandelt. Außerdem weist SQL Zeilen keine eindeutigen Bezeichner zu, sodass mehrere vollständig identische Zeilen in einer Tabelle möglich sind. Das ist eine Folge des mathematischen Modells, auf dem SQL beruht, ist in der Praxis aber meist unerwünscht. Weiter unten in diesem Kapitel wird gezeigt, wie man damit umgeht.

Jede Spalte hat einen Datentyp. Der Datentyp beschränkt die Menge möglicher Werte, die einer Spalte zugewiesen werden können, und gibt den in der Spalte gespeicherten Daten eine Bedeutung, sodass sie für Berechnungen verwendet werden können. Eine als numerisch deklarierte Spalte akzeptiert beispielsweise keine beliebigen Textzeichenketten, und die darin gespeicherten Daten können für mathematische Berechnungen genutzt werden. Eine als Zeichenkettentyp deklarierte Spalte akzeptiert dagegen nahezu beliebige Daten, eignet sich aber nicht für mathematische Berechnungen, auch wenn andere Operationen wie Zeichenkettenverkettung verfügbar sind.

PostgreSQL enthält eine umfangreiche Menge eingebauter Datentypen, die für viele Anwendungen passen. Benutzer können auch eigene Datentypen definieren. Die meisten eingebauten Datentypen haben naheliegende Namen und Bedeutungen; eine ausführliche Erklärung steht daher in [Kapitel 8](08_Datentypen.md). Häufig verwendete Datentypen sind `integer` für ganze Zahlen, `numeric` für Zahlen mit möglichen Nachkommastellen, `text` für Zeichenketten, `date` für Datumswerte, `time` für Uhrzeiten und `timestamp` für Werte, die Datum und Uhrzeit enthalten.

Zum Erstellen einer Tabelle verwenden Sie den passend benannten Befehl `CREATE TABLE`. In diesem Befehl geben Sie mindestens einen Namen für die neue Tabelle, die Namen der Spalten und den Datentyp jeder Spalte an. Zum Beispiel:

```sql
CREATE TABLE my_first_table (
    first_column text,
    second_column integer
);
```

Dies erstellt eine Tabelle namens `my_first_table` mit zwei Spalten. Die erste Spalte heißt `first_column` und hat den Datentyp `text`; die zweite Spalte heißt `second_column` und hat den Typ `integer`. Tabellen- und Spaltennamen folgen der in [Abschnitt 4.1.1](04_SQL_Syntax.md#411-bezeichner-und-schlüsselwörter) erläuterten Bezeichnersyntax. Die Typnamen sind normalerweise ebenfalls Bezeichner, es gibt aber einige Ausnahmen. Beachten Sie, dass die Spaltenliste durch Kommas getrennt und in Klammern eingeschlossen wird.

Natürlich war das vorherige Beispiel stark konstruiert. Normalerweise geben Sie Tabellen und Spalten Namen, aus denen hervorgeht, welche Art von Daten sie speichern. Betrachten wir also ein realistischeres Beispiel:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);
```

Der Typ `numeric` kann Nachkommastellen speichern, wie es für Geldbeträge typisch ist.

> **Tipp:** Wenn Sie viele miteinander verknüpfte Tabellen erstellen, ist es sinnvoll, ein konsistentes Benennungsschema für Tabellen und Spalten zu wählen. Bei Tabellennamen kann man sich beispielsweise für Singular- oder Pluralformen entscheiden; für beide Varianten gibt es Befürworter.

Die Anzahl der Spalten, die eine Tabelle enthalten kann, ist begrenzt. Abhängig von den Spaltentypen liegt die Grenze zwischen 250 und 1600. Eine Tabelle mit annähernd so vielen Spalten zu definieren, ist jedoch sehr ungewöhnlich und oft ein fragwürdiges Design.

Wenn Sie eine Tabelle nicht mehr benötigen, können Sie sie mit dem Befehl `DROP TABLE` entfernen. Zum Beispiel:

```sql
DROP TABLE my_first_table;
DROP TABLE products;
```

Der Versuch, eine nicht vorhandene Tabelle zu löschen, ist ein Fehler. Dennoch ist es in SQL-Skriptdateien üblich, vor dem Erstellen jede Tabelle bedingungslos zu löschen und etwaige Fehlermeldungen zu ignorieren, damit das Skript unabhängig davon funktioniert, ob die Tabelle bereits existiert. Wenn Sie möchten, können Sie die Variante `DROP TABLE IF EXISTS` verwenden, um solche Fehlermeldungen zu vermeiden; sie gehört jedoch nicht zum SQL-Standard.

Wenn Sie eine bereits vorhandene Tabelle ändern müssen, siehe [Abschnitt 5.7](#57-tabellen-ändern) weiter unten in diesem Kapitel.

Mit den bisher besprochenen Werkzeugen können Sie vollständig funktionsfähige Tabellen erstellen. Der Rest dieses Kapitels behandelt zusätzliche Eigenschaften der Tabellendefinition, die Datenintegrität, Sicherheit oder Komfort sicherstellen. Wenn Sie Ihre Tabellen jetzt sofort mit Daten füllen möchten, können Sie zu [Kapitel 6](06_Datenmanipulation.md) springen und den Rest dieses Kapitels später lesen.

## 5.2. Vorgabewerte

Einer Spalte kann ein Vorgabewert zugewiesen werden. Wenn eine neue Zeile erstellt wird und für einige Spalten keine Werte angegeben sind, werden diese Spalten mit ihren jeweiligen Vorgabewerten gefüllt. Ein Datenmanipulationsbefehl kann außerdem ausdrücklich anfordern, dass eine Spalte auf ihren Vorgabewert gesetzt wird, ohne diesen Wert kennen zu müssen. Details zu Datenmanipulationsbefehlen stehen in [Kapitel 6](06_Datenmanipulation.md).

Wenn kein Vorgabewert ausdrücklich deklariert wird, ist der Vorgabewert der Nullwert. Das ist in der Regel sinnvoll, weil ein Nullwert als unbekannter Wert verstanden werden kann.

In einer Tabellendefinition werden Vorgabewerte nach dem Datentyp der Spalte angegeben. Zum Beispiel:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric DEFAULT 9.99
);
```

Der Vorgabewert kann ein Ausdruck sein, der immer dann ausgewertet wird, wenn der Vorgabewert eingefügt wird, nicht bereits beim Erstellen der Tabelle. Ein häufiges Beispiel ist eine Zeitstempelspalte mit dem Vorgabewert `CURRENT_TIMESTAMP`, sodass sie auf den Zeitpunkt des Einfügens der Zeile gesetzt wird. Ein weiteres häufiges Beispiel ist das Erzeugen einer „Seriennummer“ für jede Zeile. In PostgreSQL geschieht das typischerweise etwa so:

```sql
CREATE TABLE products (
    product_no integer DEFAULT nextval('products_product_no_seq'),
    ...
);
```

wobei die Funktion `nextval()` fortlaufende Werte aus einem Sequenzobjekt liefert (siehe [Abschnitt 9.17](09_Funktionen_und_Operatoren.md#917-sequenzmanipulationsfunktionen)). Diese Konstruktion ist so häufig, dass es dafür eine spezielle Kurzform gibt:

```sql
CREATE TABLE products (
    product_no SERIAL,
    ...
);
```

Die Kurzform `SERIAL` wird in [Abschnitt 8.1.4](08_Datentypen.md#814-serialtypen) ausführlicher behandelt.

## 5.3. Identitätsspalten

Eine Identitätsspalte ist eine besondere Spalte, die automatisch aus einer impliziten Sequenz erzeugt wird. Sie kann verwendet werden, um Schlüsselwerte zu generieren.

Um eine Identitätsspalte zu erstellen, verwenden Sie die Klausel `GENERATED ... AS IDENTITY` in `CREATE TABLE`, zum Beispiel:

```sql
CREATE TABLE people (
    id bigint GENERATED ALWAYS AS IDENTITY,
    ...,
);
```

oder alternativ:

```sql
CREATE TABLE people (
    id bigint GENERATED BY DEFAULT AS IDENTITY,
    ...,
);
```

Weitere Details finden Sie bei `CREATE TABLE`.

Wenn ein `INSERT`-Befehl auf der Tabelle mit der Identitätsspalte ausgeführt wird und für die Identitätsspalte kein Wert ausdrücklich angegeben ist, wird ein von der impliziten Sequenz erzeugter Wert eingefügt. Bei den obigen Definitionen und weiteren passenden Spalten würde zum Beispiel

```sql
INSERT INTO people (name, address) VALUES ('A', 'foo');
INSERT INTO people (name, address) VALUES ('B', 'bar');
```

Werte für die Spalte `id` beginnend bei 1 erzeugen und zu folgenden Tabellendaten führen:

| id | name | address |
|---:|---|---|
| 1 | A | foo |
| 2 | B | bar |

Alternativ kann das Schlüsselwort `DEFAULT` anstelle eines Werts angegeben werden, um den sequenzerzeugten Wert ausdrücklich anzufordern, etwa so:

```sql
INSERT INTO people (id, name, address) VALUES (DEFAULT, 'C',
 'baz');
```

Entsprechend kann das Schlüsselwort `DEFAULT` auch in `UPDATE`-Befehlen verwendet werden.

In vielerlei Hinsicht verhält sich eine Identitätsspalte daher wie eine Spalte mit einem Vorgabewert.

Die Klauseln `ALWAYS` und `BY DEFAULT` in der Spaltendefinition bestimmen, wie ausdrücklich vom Benutzer angegebene Werte in `INSERT`- und `UPDATE`-Befehlen behandelt werden. Ist bei einem `INSERT`-Befehl `ALWAYS` gewählt, wird ein vom Benutzer angegebener Wert nur akzeptiert, wenn die `INSERT`-Anweisung `OVERRIDING SYSTEM VALUE` angibt. Ist `BY DEFAULT` gewählt, hat der vom Benutzer angegebene Wert Vorrang. `BY DEFAULT` verhält sich damit eher wie ein Vorgabewert, der durch einen expliziten Wert überschrieben werden kann, während `ALWAYS` etwas mehr Schutz gegen versehentlich explizit eingefügte Werte bietet.

Der Datentyp einer Identitätsspalte muss einer der von Sequenzen unterstützten Datentypen sein. Siehe `CREATE SEQUENCE`. Die Eigenschaften der zugehörigen Sequenz können beim Erstellen einer Identitätsspalte angegeben (siehe `CREATE TABLE`) oder später geändert werden (siehe `ALTER TABLE`).

Eine Identitätsspalte wird automatisch als `NOT NULL` markiert. Eine Identitätsspalte garantiert jedoch keine Eindeutigkeit. Eine Sequenz liefert normalerweise eindeutige Werte, sie kann aber zurückgesetzt werden, oder Werte können, wie oben beschrieben, manuell in die Identitätsspalte eingefügt werden. Eindeutigkeit muss mit einem `PRIMARY KEY`- oder `UNIQUE`-Constraint erzwungen werden.

In Tabellenvererbungshierarchien sind Identitätsspalten und ihre Eigenschaften in einer Kindtabelle unabhängig von denen der Elterntabellen. Eine Kindtabelle erbt Identitätsspalten oder deren Eigenschaften nicht automatisch vom Elternobjekt. Bei `INSERT` oder `UPDATE` wird eine Spalte als Identitätsspalte behandelt, wenn sie in der in der Anweisung genannten Tabelle eine Identitätsspalte ist; dann werden die entsprechenden Identitätseigenschaften angewendet.

Partitionen erben Identitätsspalten von der partitionierten Tabelle. Sie können keine eigenen Identitätsspalten haben. Die Eigenschaften einer bestimmten Identitätsspalte sind in der gesamten Partitionshierarchie konsistent.

## 5.4. Generierte Spalten

Eine generierte Spalte ist eine besondere Spalte, die immer aus anderen Spalten berechnet wird. Sie ist für Spalten also das, was eine Sicht für Tabellen ist. Es gibt zwei Arten generierter Spalten: gespeicherte und virtuelle. Eine gespeicherte generierte Spalte wird beim Schreiben, also beim Einfügen oder Aktualisieren, berechnet und belegt Speicherplatz wie eine normale Spalte. Eine virtuelle generierte Spalte belegt keinen Speicherplatz und wird beim Lesen berechnet. Eine virtuelle generierte Spalte ähnelt damit einer Sicht, und eine gespeicherte generierte Spalte ähnelt einer materialisierten Sicht, mit dem Unterschied, dass sie stets automatisch aktualisiert wird.

Um eine generierte Spalte zu erstellen, verwenden Sie die Klausel `GENERATED ALWAYS AS` in `CREATE TABLE`, zum Beispiel:

```sql
CREATE TABLE people (
    ...,
    height_cm numeric,
    height_in numeric GENERATED ALWAYS AS (height_cm / 2.54)
);
```

Eine generierte Spalte ist standardmäßig virtuell. Verwenden Sie die Schlüsselwörter `VIRTUAL` oder `STORED`, um die Wahl ausdrücklich zu machen. Weitere Details finden Sie bei `CREATE TABLE`.

In eine generierte Spalte kann nicht direkt geschrieben werden. In `INSERT`- oder `UPDATE`-Befehlen kann für eine generierte Spalte kein Wert angegeben werden; das Schlüsselwort `DEFAULT` ist jedoch zulässig.

Beachten Sie die Unterschiede zwischen einer Spalte mit Vorgabewert und einer generierten Spalte. Der Vorgabewert einer Spalte wird einmal ausgewertet, wenn die Zeile erstmals eingefügt wird und kein anderer Wert angegeben wurde; eine generierte Spalte wird bei jeder Änderung der Zeile aktualisiert und kann nicht überschrieben werden. Ein Spaltenvorgabewert darf sich nicht auf andere Spalten der Tabelle beziehen; ein Generierungsausdruck tut dies normalerweise gerade. Ein Spaltenvorgabewert kann volatile Funktionen verwenden, zum Beispiel `random()` oder Funktionen, die sich auf die aktuelle Zeit beziehen; für generierte Spalten ist das nicht erlaubt.

Für die Definition generierter Spalten und von Tabellen mit generierten Spalten gelten mehrere Einschränkungen:

- Der Generierungsausdruck darf nur immutable Funktionen verwenden und darf keine Unterabfragen nutzen oder sich in irgendeiner Weise auf etwas anderes als die aktuelle Zeile beziehen.

- Ein Generierungsausdruck darf sich nicht auf eine andere generierte Spalte beziehen.

- Ein Generierungsausdruck darf sich auf keine Systemspalte beziehen, mit Ausnahme von `tableoid`.

- Eine virtuelle generierte Spalte darf keinen benutzerdefinierten Typ haben, und der Generierungsausdruck einer virtuellen generierten Spalte darf sich nicht auf benutzerdefinierte Funktionen oder Typen beziehen; er darf also nur eingebaute Funktionen oder Typen verwenden. Das gilt auch indirekt, etwa für Funktionen oder Typen, auf denen Operatoren oder Casts beruhen. Diese Einschränkung gibt es für gespeicherte generierte Spalten nicht.

- Eine generierte Spalte darf keinen Spaltenvorgabewert und keine Identitätsdefinition haben.

- Eine generierte Spalte darf nicht Teil eines Partitionsschlüssels sein.

- Fremdtabellen können generierte Spalten haben. Details finden Sie bei `CREATE FOREIGN TABLE`.

- Für Vererbung und Partitionierung gilt:

  - Wenn eine Elternspalte eine generierte Spalte ist, muss ihre Kindspalte ebenfalls eine generierte Spalte derselben Art sein, also gespeichert oder virtuell; die Kindspalte kann jedoch einen anderen Generierungsausdruck haben.

Für gespeicherte generierte Spalten ist der beim Einfügen oder Aktualisieren einer Zeile tatsächlich angewendete Generierungsausdruck derjenige, der zu der Tabelle gehört, in der sich die Zeile physisch befindet. Das unterscheidet sich vom Verhalten bei Spaltenvorgabewerten: Dort gilt der Vorgabewert der in der Abfrage genannten Tabelle. Für virtuelle generierte Spalten gilt beim Lesen einer Tabelle der Generierungsausdruck der in der Abfrage genannten Tabelle.

  - Wenn eine Elternspalte keine generierte Spalte ist, darf auch ihre Kindspalte nicht generiert sein.

  - Bei geerbten Tabellen wird, wenn Sie in `CREATE TABLE ... INHERITS` eine Kindspaltendefinition ohne `GENERATED`-Klausel schreiben, die `GENERATED`-Klausel automatisch vom Elternobjekt kopiert. `ALTER TABLE ... INHERIT` verlangt, dass Eltern- und Kindspalten hinsichtlich ihres Generierungsstatus bereits übereinstimmen, verlangt aber nicht, dass ihre Generierungsausdrücke übereinstimmen.

  - Entsprechend wird bei partitionierten Tabellen, wenn Sie in `CREATE TABLE ... PARTITION OF` eine Kindspaltendefinition ohne `GENERATED`-Klausel schreiben, die `GENERATED`-Klausel automatisch vom Elternobjekt kopiert. `ALTER TABLE ... ATTACH PARTITION` verlangt, dass Eltern- und Kindspalten hinsichtlich ihres Generierungsstatus bereits übereinstimmen, verlangt aber nicht, dass ihre Generierungsausdrücke übereinstimmen.

  - Bei Mehrfachvererbung müssen, wenn eine Elternspalte eine generierte Spalte ist, alle Elternspalten generierte Spalten sein. Wenn sie nicht alle denselben Generierungsausdruck haben, muss der gewünschte Ausdruck für das Kind ausdrücklich angegeben werden.

Für die Verwendung generierter Spalten gelten zusätzliche Überlegungen.

- Generierte Spalten verwalten Zugriffsprivilegien getrennt von den zugrunde liegenden Basisspalten. Es ist also möglich, es so einzurichten, dass eine bestimmte Rolle aus einer generierten Spalte lesen darf, nicht aber aus den zugrunde liegenden Basisspalten.

Für virtuelle generierte Spalten ist dies nur dann vollständig sicher, wenn der Generierungsausdruck ausschließlich leakproof Funktionen verwendet (siehe `CREATE FUNCTION`); das System erzwingt dies jedoch nicht.

- Privilegien von Funktionen, die in Generierungsausdrücken verwendet werden, werden geprüft, wenn der Ausdruck tatsächlich ausgeführt wird, also je nach Fall beim Schreiben oder Lesen, so als wäre der Generierungsausdruck direkt aus der Abfrage aufgerufen worden, die die generierte Spalte verwendet. Der Benutzer einer generierten Spalte muss die Berechtigung haben, alle vom Generierungsausdruck verwendeten Funktionen aufzurufen. Funktionen im Generierungsausdruck werden mit den Privilegien des ausführenden Benutzers oder des Funktionsbesitzers ausgeführt, abhängig davon, ob die Funktionen als `SECURITY INVOKER` oder `SECURITY DEFINER` definiert sind.

- Generierte Spalten werden konzeptionell nach der Ausführung von `BEFORE`-Triggern aktualisiert. Änderungen, die ein `BEFORE`-Trigger an Basisspalten vornimmt, spiegeln sich daher in generierten Spalten wider. Umgekehrt ist es jedoch nicht erlaubt, in `BEFORE`-Triggern auf generierte Spalten zuzugreifen.

- Generierte Spalten dürfen bei der logischen Replikation gemäß dem Parameter `publish_generated_columns` von `CREATE PUBLICATION` oder durch Aufnahme in die Spaltenliste des Befehls `CREATE PUBLICATION` repliziert werden. Dies wird derzeit nur für gespeicherte generierte Spalten unterstützt. Details finden Sie in [Abschnitt 29.6](29_Logische_Replikation.md#296-replikation-generierter-spalten).

## 5.5. Integritätsbedingungen (Constraints)

Datentypen sind eine Möglichkeit, die Art der Daten einzuschränken, die in einer Tabelle gespeichert werden können. Für viele Anwendungen ist diese Einschränkung jedoch zu grob. Eine Spalte mit einem Produktpreis sollte wahrscheinlich nur positive Werte akzeptieren. Es gibt aber keinen Standarddatentyp, der ausschließlich positive Zahlen zulässt. Außerdem möchte man Spaltendaten manchmal in Bezug auf andere Spalten oder Zeilen einschränken. In einer Tabelle mit Produktinformationen sollte es zum Beispiel nur eine Zeile für jede Produktnummer geben.

Zu diesem Zweck erlaubt SQL, Constraints für Spalten und Tabellen zu definieren. Constraints geben Ihnen so viel Kontrolle über die Daten in Ihren Tabellen, wie Sie benötigen. Wenn ein Benutzer versucht, Daten in einer Spalte zu speichern, die einen Constraint verletzen, wird ein Fehler ausgelöst. Das gilt auch dann, wenn der Wert aus einer Vorgabewertdefinition stammt.

### 5.5.1. CHECK-Constraints

Ein CHECK-Constraint ist der allgemeinste Constraint-Typ. Er erlaubt festzulegen, dass der Wert in einer bestimmten Spalte einen booleschen Ausdruck erfüllen muss. Um zum Beispiel positive Produktpreise zu verlangen, könnten Sie schreiben:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0)
);
```

Wie zu sehen ist, steht die Constraint-Definition nach dem Datentyp, genau wie Vorgabewertdefinitionen. Vorgabewerte und Constraints können in beliebiger Reihenfolge aufgeführt werden. Ein CHECK-Constraint besteht aus dem Schlüsselwort `CHECK`, gefolgt von einem Ausdruck in Klammern. Der Ausdruck des CHECK-Constraints sollte die dadurch eingeschränkte Spalte einbeziehen, da der Constraint sonst wenig Sinn ergäbe.

Sie können dem Constraint auch einen eigenen Namen geben. Das macht Fehlermeldungen klarer und erlaubt, den Constraint später gezielt zu referenzieren, wenn er geändert werden soll. Die Syntax lautet:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CONSTRAINT positive_price CHECK (price > 0)
);
```

Um also einen benannten Constraint anzugeben, verwenden Sie das Schlüsselwort `CONSTRAINT`, gefolgt von einem Bezeichner und anschließend der Constraint-Definition. Wenn Sie keinen Constraint-Namen angeben, wählt das System einen Namen für Sie.

Ein CHECK-Constraint kann sich auch auf mehrere Spalten beziehen. Angenommen, Sie speichern einen regulären Preis und einen rabattierten Preis und möchten sicherstellen, dass der rabattierte Preis niedriger ist als der reguläre Preis:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric CHECK (discounted_price > 0),
    CHECK (price > discounted_price)
);
```

Die ersten beiden Constraints sollten vertraut aussehen. Der dritte verwendet eine neue Syntax. Er ist nicht an eine bestimmte Spalte angehängt, sondern erscheint als eigenständiges Element in der durch Kommas getrennten Spaltenliste. Spaltendefinitionen und solche Constraint-Definitionen können gemischt aufgeführt werden.

Die ersten beiden Constraints sind Spalten-Constraints, während der dritte ein Tabellen-Constraint ist, weil er getrennt von jeder einzelnen Spaltendefinition geschrieben wird. Spalten-Constraints können auch als Tabellen-Constraints geschrieben werden; umgekehrt ist das nicht unbedingt möglich, da ein Spalten-Constraint eigentlich nur auf die Spalte verweisen sollte, an die er angehängt ist. PostgreSQL erzwingt diese Regel nicht, aber Sie sollten sie beachten, wenn Ihre Tabellendefinitionen auch mit anderen Datenbanksystemen funktionieren sollen. Das obige Beispiel könnte auch so geschrieben werden:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0),
    CHECK (price > discounted_price)
);
```

Oder sogar so:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0 AND price > discounted_price)
);
```

Das ist Geschmackssache.

Tabellen-Constraints können auf dieselbe Weise Namen erhalten wie Spalten-Constraints:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0),
    CONSTRAINT valid_discount CHECK (price > discounted_price)
);
```

Beachten Sie, dass ein CHECK-Constraint erfüllt ist, wenn der Prüfausdruck entweder `true` oder den Nullwert ergibt. Da die meisten Ausdrücke den Nullwert ergeben, wenn irgendein Operand null ist, verhindern sie keine Nullwerte in den eingeschränkten Spalten. Um sicherzustellen, dass eine Spalte keine Nullwerte enthält, kann der im nächsten Abschnitt beschriebene NOT-NULL-Constraint verwendet werden.

> Hinweis: PostgreSQL unterstützt keine CHECK-Constraints, die auf Tabellendaten außerhalb der neu eingefügten oder aktualisierten Zeile verweisen, die gerade geprüft wird. Ein CHECK-Constraint, der diese Regel verletzt, mag in einfachen Tests zu funktionieren scheinen, kann aber nicht garantieren, dass die Datenbank nie in einen Zustand gelangt, in dem die Constraint-Bedingung falsch ist, etwa durch spätere Änderungen an den beteiligten anderen Zeilen. Das würde dazu führen, dass ein Datenbank-Dump und eine Wiederherstellung fehlschlagen. Die Wiederherstellung kann sogar dann scheitern, wenn der vollständige Datenbankzustand mit dem Constraint konsistent ist, weil Zeilen nicht in einer Reihenfolge geladen werden, die den Constraint erfüllt. Verwenden Sie, wenn möglich, `UNIQUE`-, `EXCLUDE`- oder `FOREIGN KEY`-Constraints, um Einschränkungen über Zeilen oder Tabellen hinweg auszudrücken.
>
> Wenn Sie nur beim Einfügen einer Zeile eine einmalige Prüfung gegen andere Zeilen wünschen, statt eine dauerhaft gepflegte Konsistenzgarantie zu benötigen, kann dafür ein eigener Trigger verwendet werden. Dieser Ansatz vermeidet das Dump-/Restore-Problem, weil `pg_dump` Trigger erst nach dem Wiederherstellen der Daten erneut installiert, sodass die Prüfung während Dump und Restore nicht erzwungen wird.

> Hinweis: PostgreSQL nimmt an, dass Bedingungen von CHECK-Constraints unveränderlich sind, also für dieselbe Eingabezeile immer dasselbe Ergebnis liefern. Diese Annahme rechtfertigt, CHECK-Constraints nur beim Einfügen oder Aktualisieren von Zeilen zu prüfen und nicht zu anderen Zeitpunkten. Die obige Warnung, nicht auf andere Tabellendaten zu verweisen, ist eigentlich ein Spezialfall dieser Einschränkung.
>
> Eine häufige Möglichkeit, diese Annahme zu verletzen, besteht darin, in einem CHECK-Ausdruck eine benutzerdefinierte Funktion zu referenzieren und anschließend das Verhalten dieser Funktion zu ändern. PostgreSQL verbietet das nicht, bemerkt aber auch nicht, wenn dadurch Zeilen in der Tabelle den CHECK-Constraint verletzen. Ein späterer Datenbank-Dump mit Wiederherstellung würde dann fehlschlagen. Die empfohlene Vorgehensweise bei einer solchen Änderung ist, den Constraint zu entfernen, etwa mit `ALTER TABLE`, die Funktionsdefinition anzupassen und den Constraint erneut hinzuzufügen, wodurch er gegen alle Tabellenzeilen neu geprüft wird.

### 5.5.2. NOT-NULL-Constraints

Ein NOT-NULL-Constraint legt schlicht fest, dass eine Spalte nicht den Nullwert annehmen darf. Ein Syntaxbeispiel:

```sql
CREATE TABLE products (
    product_no integer NOT NULL,
    name text NOT NULL,
    price numeric
);
```

Ein expliziter Constraint-Name kann ebenfalls angegeben werden, zum Beispiel:

```sql
CREATE TABLE products (
    product_no integer NOT NULL,
    name text CONSTRAINT products_name_not_null NOT NULL,
    price numeric
);
```

Ein NOT-NULL-Constraint wird normalerweise als Spalten-Constraint geschrieben. Die Syntax, um ihn als Tabellen-Constraint zu schreiben, lautet:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    NOT NULL product_no,
    NOT NULL name
);
```

Diese Syntax ist jedoch nicht standardkonform und hauptsächlich für die Verwendung durch `pg_dump` gedacht.

Ein NOT-NULL-Constraint ist funktional gleichbedeutend mit einem CHECK-Constraint `CHECK (column_name IS NOT NULL)`, aber in PostgreSQL ist ein expliziter NOT-NULL-Constraint effizienter.

Natürlich kann eine Spalte mehr als einen Constraint haben. Schreiben Sie die Constraints einfach hintereinander:

```sql
CREATE TABLE products (
    product_no integer NOT NULL,
    name text NOT NULL,
    price numeric NOT NULL CHECK (price > 0)
);
```

Die Reihenfolge spielt keine Rolle. Sie bestimmt nicht notwendigerweise, in welcher Reihenfolge die Constraints geprüft werden.

Eine Spalte kann jedoch höchstens einen expliziten NOT-NULL-Constraint haben.

Der `NOT NULL`-Constraint hat ein Gegenstück: den `NULL`-Constraint. Das bedeutet nicht, dass die Spalte null sein muss, was offensichtlich wenig nützlich wäre. Stattdessen wird damit einfach das Standardverhalten ausgewählt, dass die Spalte null sein darf. Der `NULL`-Constraint ist nicht im SQL-Standard enthalten und sollte in portablen Anwendungen nicht verwendet werden. Er wurde nur in PostgreSQL ergänzt, um mit einigen anderen Datenbanksystemen kompatibel zu sein. Manche Benutzer mögen ihn dennoch, weil sich der Constraint dadurch in einer Skriptdatei leicht umschalten lässt. Sie könnten zum Beispiel beginnen mit:

```sql
CREATE TABLE products (
    product_no integer NULL,
    name text NULL,
    price numeric NULL
);
```

und dann dort, wo gewünscht, das Schlüsselwort `NOT` einfügen.

> Tipp: In den meisten Datenbankentwürfen sollte die Mehrzahl der Spalten als NOT NULL markiert sein.

### 5.5.3. UNIQUE-Constraints

UNIQUE-Constraints stellen sicher, dass die in einer Spalte oder in einer Gruppe von Spalten enthaltenen Daten unter allen Zeilen der Tabelle eindeutig sind. Als Spalten-Constraint lautet die Syntax:

```sql
CREATE TABLE products (
    product_no integer UNIQUE,
    name text,
    price numeric
);
```

Als Tabellen-Constraint lautet sie:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    UNIQUE (product_no)
);
```

Um einen UNIQUE-Constraint für eine Gruppe von Spalten zu definieren, schreiben Sie ihn als Tabellen-Constraint mit durch Kommas getrennten Spaltennamen:

```sql
CREATE TABLE example (
    a integer,
    b integer,
    c integer,
    UNIQUE (a, c)
);
```

Dies legt fest, dass die Wertekombination in den angegebenen Spalten über die gesamte Tabelle hinweg eindeutig ist, auch wenn keine der einzelnen Spalten für sich genommen eindeutig sein muss und es normalerweise auch nicht ist.

Sie können einem UNIQUE-Constraint wie üblich einen eigenen Namen geben:

```sql
CREATE TABLE products (
    product_no integer CONSTRAINT must_be_different UNIQUE,
    name text,
    price numeric
);
```

Das Hinzufügen eines UNIQUE-Constraints erzeugt automatisch einen eindeutigen B-Tree-Index auf der im Constraint aufgeführten Spalte oder Spaltengruppe. Eine Eindeutigkeitsbeschränkung, die nur einige Zeilen betrifft, kann nicht als UNIQUE-Constraint geschrieben werden; sie kann aber durch einen eindeutigen partiellen Index erzwungen werden.

Im Allgemeinen ist ein UNIQUE-Constraint verletzt, wenn es mehr als eine Zeile in der Tabelle gibt, in der die Werte aller im Constraint enthaltenen Spalten gleich sind. Standardmäßig gelten zwei Nullwerte bei diesem Vergleich nicht als gleich. Das bedeutet, dass trotz UNIQUE-Constraint doppelte Zeilen gespeichert werden können, die in mindestens einer der eingeschränkten Spalten einen Nullwert enthalten. Dieses Verhalten kann durch die Klausel `NULLS NOT DISTINCT` geändert werden, etwa so:

```sql
CREATE TABLE products (
    product_no integer UNIQUE NULLS NOT DISTINCT,
    name text,
    price numeric
);
```

Oder so:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    UNIQUE NULLS NOT DISTINCT (product_no)
);
```

Das Standardverhalten kann mit `NULLS DISTINCT` ausdrücklich angegeben werden. Die Standardbehandlung von Nullwerten in UNIQUE-Constraints ist laut SQL-Standard implementierungsabhängig, und andere Implementierungen verhalten sich anders. Seien Sie daher vorsichtig, wenn Sie Anwendungen entwickeln, die portabel sein sollen.

### 5.5.4. Primärschlüssel

Ein Primärschlüssel-Constraint zeigt an, dass eine Spalte oder eine Gruppe von Spalten als eindeutiger Bezeichner für Zeilen in der Tabelle verwendet werden kann. Dafür müssen die Werte sowohl eindeutig als auch nicht null sein. Die folgenden beiden Tabellendefinitionen akzeptieren daher dieselben Daten:

```sql
CREATE TABLE products (
    product_no integer UNIQUE NOT NULL,
    name text,
    price numeric
);

CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);
```

Primärschlüssel können mehr als eine Spalte umfassen; die Syntax ähnelt UNIQUE-Constraints:

```sql
CREATE TABLE example (
    a integer,
    b integer,
    c integer,
    PRIMARY KEY (a, c)
);
```

Das Hinzufügen eines Primärschlüssels erzeugt automatisch einen eindeutigen B-Tree-Index auf der im Primärschlüssel aufgeführten Spalte oder Spaltengruppe und erzwingt, dass die Spalte oder Spalten als `NOT NULL` markiert werden.

Eine Tabelle kann höchstens einen Primärschlüssel haben. Es kann beliebig viele UNIQUE-Constraints geben, die in Kombination mit NOT-NULL-Constraints funktional fast dasselbe leisten; aber nur einer kann als Primärschlüssel ausgewiesen werden. Die relationale Datenbanktheorie verlangt, dass jede Tabelle einen Primärschlüssel hat. PostgreSQL erzwingt diese Regel nicht, aber es ist normalerweise am besten, ihr zu folgen.

Primärschlüssel sind sowohl für die Dokumentation als auch für Client-Anwendungen nützlich. Eine GUI-Anwendung, die das Ändern von Zeilenwerten erlaubt, muss zum Beispiel wahrscheinlich den Primärschlüssel einer Tabelle kennen, um Zeilen eindeutig identifizieren zu können. Auch das Datenbanksystem nutzt einen deklarierten Primärschlüssel auf verschiedene Weise; der Primärschlüssel definiert beispielsweise die Standardzielspalte oder -spalten für Fremdschlüssel, die auf seine Tabelle verweisen.

### 5.5.5. Fremdschlüssel

Ein Fremdschlüssel-Constraint legt fest, dass die Werte in einer Spalte oder in einer Gruppe von Spalten mit Werten übereinstimmen müssen, die in irgendeiner Zeile einer anderen Tabelle vorkommen. Dadurch wird die referenzielle Integrität zwischen zwei verwandten Tabellen gewahrt.

Nehmen wir die Produkttabelle, die wir schon mehrfach verwendet haben:

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);
```

Angenommen, es gibt außerdem eine Tabelle, die Bestellungen dieser Produkte speichert. Wir möchten sicherstellen, dass die Bestelltabelle nur Bestellungen von Produkten enthält, die tatsächlich existieren. Daher definieren wir in der Tabelle `orders` einen Fremdschlüssel-Constraint, der die Tabelle `products` referenziert:

```sql
CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products (product_no),
    quantity integer
);
```

Nun ist es unmöglich, Bestellungen mit nicht-nulligen `product_no`-Einträgen anzulegen, die in der Tabelle `products` nicht vorkommen.

In dieser Situation heißt die Tabelle `orders` die referenzierende Tabelle und die Tabelle `products` die referenzierte Tabelle. Entsprechend gibt es referenzierende und referenzierte Spalten.

Der obige Befehl kann auch verkürzt werden:

```sql
CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products,
    quantity integer
);
```

Wenn keine Spaltenliste angegeben ist, wird der Primärschlüssel der referenzierten Tabelle als referenzierte Spalte oder Spalten verwendet.

Sie können einem Fremdschlüssel-Constraint wie üblich einen eigenen Namen geben.

Ein Fremdschlüssel kann auch eine Gruppe von Spalten einschränken und referenzieren. Wie üblich muss er dann in der Form eines Tabellen-Constraints geschrieben werden. Hier ist ein künstliches Syntaxbeispiel:

```sql
CREATE TABLE t1 (
   a integer PRIMARY KEY,
   b integer,
   c integer,
   FOREIGN KEY (b, c) REFERENCES other_table (c1, c2)
);
```

Natürlich müssen Anzahl und Typ der eingeschränkten Spalten mit Anzahl und Typ der referenzierten Spalten übereinstimmen.

Manchmal ist es nützlich, wenn die "andere Tabelle" eines Fremdschlüssel-Constraints dieselbe Tabelle ist. Das nennt man einen selbstreferenziellen Fremdschlüssel. Wenn Zeilen einer Tabelle beispielsweise Knoten einer Baumstruktur darstellen sollen, könnten Sie schreiben:

```sql
CREATE TABLE tree (
    node_id integer PRIMARY KEY,
    parent_id integer REFERENCES tree,
    name text,
    ...
);
```

Ein oberster Knoten hätte `NULL` als `parent_id`, während nicht-nullige `parent_id`-Einträge darauf beschränkt wären, gültige Zeilen der Tabelle zu referenzieren.

Eine Tabelle kann mehr als einen Fremdschlüssel-Constraint haben. Das wird verwendet, um Viele-zu-viele-Beziehungen zwischen Tabellen umzusetzen. Angenommen, Sie haben Tabellen für Produkte und Bestellungen, möchten nun aber erlauben, dass eine Bestellung möglicherweise viele Produkte enthält, was die obige Struktur nicht erlaubte. Sie könnten diese Tabellenstruktur verwenden:

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    shipping_address text,
    ...
);

CREATE TABLE order_items (
    product_no integer REFERENCES products,
    order_id integer REFERENCES orders,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
```

Beachten Sie, dass sich der Primärschlüssel in der letzten Tabelle mit den Fremdschlüsseln überschneidet.

Wir wissen, dass die Fremdschlüssel das Anlegen von Bestellungen verhindern, die sich auf keine Produkte beziehen. Was aber geschieht, wenn ein Produkt entfernt wird, nachdem eine Bestellung angelegt wurde, die darauf verweist? SQL erlaubt auch dafür eine Behandlung. Intuitiv gibt es mehrere Möglichkeiten:

- Das Löschen eines referenzierten Produkts verbieten
- Die Bestellungen ebenfalls löschen
- Etwas anderes tun

Zur Veranschaulichung setzen wir in der obigen Viele-zu-viele-Beziehung folgende Regel um: Wenn jemand ein Produkt entfernen möchte, das noch von einer Bestellung referenziert wird, nämlich über `order_items`, verbieten wir dies. Wenn jemand eine Bestellung entfernt, werden die zugehörigen Bestellpositionen ebenfalls entfernt:

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    shipping_address text,
    ...
);

CREATE TABLE order_items (
    product_no integer REFERENCES products ON DELETE RESTRICT,
    order_id integer REFERENCES orders ON DELETE CASCADE,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
```

Die Standardaktion für `ON DELETE` ist `ON DELETE NO ACTION`; sie muss nicht angegeben werden. Das bedeutet, dass das Löschen in der referenzierten Tabelle zunächst fortgesetzt werden darf. Der Fremdschlüssel-Constraint muss aber weiterhin erfüllt sein, sodass diese Operation normalerweise zu einem Fehler führt. Die Prüfung von Fremdschlüssel-Constraints kann jedoch auch auf später in der Transaktion verschoben werden; das wird in diesem Kapitel nicht behandelt. In diesem Fall würde `NO ACTION` anderen Befehlen erlauben, die Situation vor der Constraint-Prüfung zu "reparieren", etwa durch Einfügen einer anderen passenden Zeile in die referenzierte Tabelle oder durch Löschen der nun verwaisten Zeilen aus der referenzierenden Tabelle.

`RESTRICT` ist strenger als `NO ACTION`. Es verhindert das Löschen einer referenzierten Zeile und erlaubt nicht, die Prüfung auf später in der Transaktion zu verschieben.

`CASCADE` gibt an, dass beim Löschen einer referenzierten Zeile auch die Zeile oder Zeilen, die darauf verweisen, automatisch gelöscht werden sollen.

Es gibt zwei weitere Optionen: `SET NULL` und `SET DEFAULT`. Sie bewirken, dass die referenzierende Spalte oder Spalten in den referenzierenden Zeilen auf null beziehungsweise auf ihre Standardwerte gesetzt werden, wenn die referenzierte Zeile gelöscht wird. Beachten Sie, dass Sie dadurch nicht von der Einhaltung anderer Constraints befreit sind. Wenn eine Aktion beispielsweise `SET DEFAULT` angibt, der Standardwert aber den Fremdschlüssel-Constraint nicht erfüllen würde, schlägt die Operation fehl.

Die passende Wahl der `ON DELETE`-Aktion hängt davon ab, welche Arten von Objekten die verbundenen Tabellen darstellen. Wenn die referenzierende Tabelle etwas darstellt, das Bestandteil dessen ist, was die referenzierte Tabelle darstellt, und nicht unabhängig existieren kann, kann `CASCADE` passend sein. Wenn die beiden Tabellen unabhängige Objekte darstellen, sind `RESTRICT` oder `NO ACTION` passender; eine Anwendung, die tatsächlich beide Objekte löschen möchte, müsste dies dann ausdrücklich tun und zwei Löschbefehle ausführen. Im obigen Beispiel sind Bestellpositionen Teil einer Bestellung, und es ist praktisch, wenn sie automatisch gelöscht werden, sobald eine Bestellung gelöscht wird. Produkte und Bestellungen sind jedoch unterschiedliche Dinge; daher könnte es problematisch sein, wenn das Löschen eines Produkts automatisch das Löschen einiger Bestellpositionen auslöst. Die Aktionen `SET NULL` oder `SET DEFAULT` können passend sein, wenn eine Fremdschlüsselbeziehung optionale Informationen darstellt. Wenn die Tabelle `products` beispielsweise einen Verweis auf einen Produktmanager enthielte und der Eintrag dieses Produktmanagers gelöscht würde, könnte es sinnvoll sein, den Produktmanager des Produkts auf null oder einen Standardwert zu setzen.

Die Aktionen `SET NULL` und `SET DEFAULT` können eine Spaltenliste erhalten, um anzugeben, welche Spalten gesetzt werden sollen. Normalerweise werden alle Spalten des Fremdschlüssel-Constraints gesetzt; nur eine Teilmenge zu setzen, ist in einigen Sonderfällen nützlich. Betrachten Sie folgendes Beispiel:

```sql
CREATE TABLE tenants (
    tenant_id integer PRIMARY KEY
);

CREATE TABLE users (
    tenant_id integer REFERENCES tenants ON DELETE CASCADE,
    user_id integer NOT NULL,
    PRIMARY KEY (tenant_id, user_id)
);

CREATE TABLE posts (
    tenant_id integer REFERENCES tenants ON DELETE CASCADE,
    post_id integer NOT NULL,
    author_id integer,
    PRIMARY KEY (tenant_id, post_id),
    FOREIGN KEY (tenant_id, author_id) REFERENCES users ON DELETE
 SET NULL (author_id)
);
```

Ohne Angabe der Spalte würde der Fremdschlüssel auch die Spalte `tenant_id` auf null setzen, obwohl diese Spalte weiterhin als Teil des Primärschlüssels benötigt wird.

Analog zu `ON DELETE` gibt es auch `ON UPDATE`, das aufgerufen wird, wenn eine referenzierte Spalte geändert wird. Die möglichen Aktionen sind dieselben, außer dass für `SET NULL` und `SET DEFAULT` keine Spaltenlisten angegeben werden können. In diesem Fall bedeutet `CASCADE`, dass die aktualisierten Werte der referenzierten Spalte oder Spalten in die referenzierenden Zeilen kopiert werden sollen. Es gibt außerdem einen merklichen Unterschied zwischen `ON UPDATE NO ACTION`, der Voreinstellung, und `ON UPDATE RESTRICT`. Ersteres lässt die Aktualisierung zu, und der Fremdschlüssel-Constraint wird gegen den Zustand nach der Aktualisierung geprüft. Letzteres verhindert die Aktualisierung bereits dann, wenn der Zustand nach der Aktualisierung den Constraint weiterhin erfüllen würde. Das verhindert, dass eine referenzierte Zeile auf einen Wert aktualisiert wird, der zwar verschieden ist, aber als gleich vergleicht, zum Beispiel eine Zeichenkette mit anderer Groß-/Kleinschreibung, wenn ein Zeichenkettentyp mit case-insensitive Collation verwendet wird.

Normalerweise muss eine referenzierende Zeile den Fremdschlüssel-Constraint nicht erfüllen, wenn irgendeine ihrer referenzierenden Spalten null ist. Wenn der Fremdschlüsseldeklaration `MATCH FULL` hinzugefügt wird, entkommt eine referenzierende Zeile dem Constraint nur dann, wenn alle ihre referenzierenden Spalten null sind; eine Mischung aus Null- und Nicht-Null-Werten schlägt bei einem `MATCH FULL`-Constraint also garantiert fehl. Wenn referenzierende Zeilen die Erfüllung des Fremdschlüssel-Constraints nicht auf diese Weise umgehen können sollen, deklarieren Sie die referenzierende Spalte oder die referenzierenden Spalten als `NOT NULL`.

Ein Fremdschlüssel muss Spalten referenzieren, die entweder ein Primärschlüssel sind, einen UNIQUE-Constraint bilden oder Spalten aus einem nicht partiellen eindeutigen Index sind. Das bedeutet, dass die referenzierten Spalten immer einen Index haben, der effiziente Suchvorgänge ermöglicht, um zu prüfen, ob eine referenzierende Zeile eine passende Zeile hat. Da ein `DELETE` einer Zeile aus der referenzierten Tabelle oder ein `UPDATE` einer referenzierten Spalte einen Scan der referenzierenden Tabelle nach Zeilen mit dem alten Wert erfordert, ist es oft sinnvoll, auch die referenzierenden Spalten zu indizieren. Weil das nicht immer nötig ist und es viele Möglichkeiten gibt, wie indiziert werden kann, erzeugt die Deklaration eines Fremdschlüssel-Constraints nicht automatisch einen Index auf den referenzierenden Spalten.

Weitere Informationen zum Aktualisieren und Löschen von Daten finden Sie in [Kapitel 6](06_Datenmanipulation.md). Siehe außerdem die Beschreibung der Fremdschlüssel-Constraint-Syntax in der Referenzdokumentation zu `CREATE TABLE`.

### 5.5.6. Exclusion-Constraints

Exclusion-Constraints stellen sicher, dass beim Vergleich zweier beliebiger Zeilen über die angegebenen Spalten oder Ausdrücke mit den angegebenen Operatoren mindestens einer dieser Operatorvergleiche `false` oder `null` ergibt. Die Syntax lautet:

```sql
CREATE TABLE circles (
    c circle,
    EXCLUDE USING gist (c WITH &&)
);
```

Siehe auch `CREATE TABLE ... CONSTRAINT ... EXCLUDE` für Details.

Das Hinzufügen eines Exclusion-Constraints erzeugt automatisch einen Index des in der Constraint-Deklaration angegebenen Typs.

## 5.6. Systemspalten

Jede Tabelle besitzt mehrere Systemspalten, die implizit vom System definiert werden. Diese Namen können daher nicht als Namen benutzerdefinierter Spalten verwendet werden. Beachten Sie, dass diese Einschränkungen unabhängig davon gelten, ob der Name ein Schlüsselwort ist; auch das Quoten eines Namens hebt diese Einschränkungen nicht auf. Im Alltag müssen Sie sich um diese Spalten meist nicht kümmern, sollten aber wissen, dass es sie gibt.

- `tableoid`
  - Die OID der Tabelle, die diese Zeile enthält. Diese Spalte ist besonders nützlich für Abfragen, die aus partitionierten Tabellen (siehe [Abschnitt 5.12](05_Datendefinition.md#512-tabellenpartitionierung)) oder Vererbungshierarchien (siehe [Abschnitt 5.11](05_Datendefinition.md#511-vererbung)) auswählen, weil sonst schwer zu erkennen ist, aus welcher konkreten Tabelle eine Zeile stammt. `tableoid` kann mit der Spalte `oid` von `pg_class` verknüpft werden, um den Tabellennamen zu erhalten.

- `xmin`
  - Die Identität, also Transaktions-ID, der einfügenden Transaktion für diese Zeilenversion. Eine Zeilenversion ist ein einzelner Zustand einer Zeile; jede Aktualisierung einer Zeile erzeugt eine neue Zeilenversion für dieselbe logische Zeile.

- `cmin`
  - Die Befehlskennung innerhalb der einfügenden Transaktion, beginnend bei null.

- `xmax`
  - Die Identität, also Transaktions-ID, der löschenden Transaktion oder null für eine nicht gelöschte Zeilenversion. Diese Spalte kann in einer sichtbaren Zeilenversion ungleich null sein. Das weist normalerweise darauf hin, dass die löschende Transaktion noch nicht bestätigt wurde oder dass ein Löschversuch zurückgerollt wurde.

- `cmax`
  - Die Befehlskennung innerhalb der löschenden Transaktion oder null.

- `ctid`
  - Der physische Speicherort der Zeilenversion innerhalb ihrer Tabelle. Obwohl `ctid` verwendet werden kann, um die Zeilenversion sehr schnell zu finden, ändert sich die `ctid` einer Zeile, wenn sie aktualisiert oder durch `VACUUM FULL` verschoben wird. Daher ist `ctid` als langfristiger Zeilenbezeichner ungeeignet. Zur Identifikation logischer Zeilen sollte ein Primärschlüssel verwendet werden.

Transaktionskennungen sind ebenfalls 32-Bit-Größen. In einer langlebigen Datenbank können Transaktions-IDs umlaufen. Bei geeigneter Wartung ist das kein schwerwiegendes Problem; Details stehen in [Kapitel 24](24_Regelmäßige_Datenbankwartung.md). Es ist jedoch unklug, sich langfristig, also über mehr als eine Milliarde Transaktionen hinweg, auf die Eindeutigkeit von Transaktions-IDs zu verlassen.

Befehlskennungen sind ebenfalls 32-Bit-Größen. Daraus ergibt sich eine harte Grenze von 2^32, also 4 Milliarden, SQL-Befehlen innerhalb einer einzelnen Transaktion. In der Praxis ist diese Grenze kein Problem; beachten Sie, dass die Grenze die Anzahl der SQL-Befehle betrifft, nicht die Anzahl der verarbeiteten Zeilen. Außerdem verbrauchen nur Befehle, die den Datenbankinhalt tatsächlich ändern, eine Befehlskennung.

## 5.7. Tabellen ändern

Wenn Sie beim Erstellen einer Tabelle feststellen, dass Sie einen Fehler gemacht haben, oder wenn sich die Anforderungen der Anwendung ändern, können Sie die Tabelle löschen und neu erstellen. Das ist jedoch keine bequeme Option, wenn die Tabelle bereits Daten enthält oder von anderen Datenbankobjekten referenziert wird, etwa durch einen Fremdschlüssel-Constraint. Deshalb stellt PostgreSQL eine Reihe von Befehlen bereit, um bestehende Tabellen zu ändern. Beachten Sie, dass dies konzeptionell etwas anderes ist als das Ändern der in der Tabelle enthaltenen Daten: Hier geht es darum, die Definition oder Struktur der Tabelle zu ändern.

Sie können:

- Spalten hinzufügen
- Spalten entfernen
- Constraints hinzufügen
- Constraints entfernen
- Vorgabewerte ändern
- Spaltendatentypen ändern
- Spalten umbenennen
- Tabellen umbenennen

Alle diese Aktionen werden mit dem Befehl `ALTER TABLE` ausgeführt; seine Referenzseite enthält weitere Details, die hier nicht behandelt werden.

### 5.7.1. Eine Spalte hinzufügen

Um eine Spalte hinzuzufügen, verwenden Sie einen Befehl wie:

```sql
ALTER TABLE products ADD COLUMN description text;
```

Die neue Spalte wird zunächst mit dem angegebenen Vorgabewert gefüllt, oder mit null, wenn Sie keine `DEFAULT`-Klausel angeben.

> Tipp: Das Hinzufügen einer Spalte mit einem konstanten Vorgabewert erfordert nicht, dass jede Zeile der Tabelle aktualisiert wird, wenn die `ALTER TABLE`-Anweisung ausgeführt wird. Stattdessen wird der Vorgabewert beim nächsten Zugriff auf die Zeile zurückgegeben und angewendet, wenn die Tabelle neu geschrieben wird. Dadurch ist `ALTER TABLE` auch bei großen Tabellen sehr schnell.
>
> Wenn der Vorgabewert volatil ist, etwa `clock_timestamp()`, muss jede Zeile mit dem Wert aktualisiert werden, der zum Zeitpunkt der Ausführung von `ALTER TABLE` berechnet wird. Um eine möglicherweise lang laufende Aktualisierung zu vermeiden, insbesondere wenn Sie die Spalte ohnehin überwiegend mit Nicht-Standardwerten füllen wollen, kann es besser sein, die Spalte ohne Vorgabewert hinzuzufügen, die richtigen Werte mit `UPDATE` einzutragen und anschließend den gewünschten Vorgabewert wie unten beschrieben zu ergänzen.

Sie können zugleich Constraints für die Spalte definieren, mit der üblichen Syntax:

```sql
ALTER TABLE products ADD COLUMN description text CHECK (description
 <> '');
```

Tatsächlich können hier alle Optionen verwendet werden, die auch auf eine Spaltenbeschreibung in `CREATE TABLE` angewendet werden können. Beachten Sie jedoch, dass der Vorgabewert die angegebenen Constraints erfüllen muss, sonst schlägt `ADD` fehl. Alternativ können Sie Constraints später hinzufügen, nachdem Sie die neue Spalte korrekt gefüllt haben.

### 5.7.2. Eine Spalte entfernen

Um eine Spalte zu entfernen, verwenden Sie einen Befehl wie:

```sql
ALTER TABLE products DROP COLUMN description;
```

Alle Daten in dieser Spalte verschwinden. Tabellen-Constraints, die die Spalte betreffen, werden ebenfalls gelöscht. Wenn die Spalte jedoch von einem Fremdschlüssel-Constraint einer anderen Tabelle referenziert wird, löscht PostgreSQL diesen Constraint nicht stillschweigend. Sie können das Löschen aller von der Spalte abhängigen Objekte erlauben, indem Sie `CASCADE` hinzufügen:

```sql
ALTER TABLE products DROP COLUMN description CASCADE;
```

Eine Beschreibung des allgemeinen Mechanismus dahinter finden Sie in [Abschnitt 5.15](05_Datendefinition.md#515-abhängigkeitsverfolgung).

### 5.7.3. Einen Constraint hinzufügen

Um einen Constraint hinzuzufügen, wird die Tabellen-Constraint-Syntax verwendet. Zum Beispiel:

```sql
ALTER TABLE products ADD CHECK (name <> '');
ALTER TABLE products ADD CONSTRAINT some_name UNIQUE (product_no);
ALTER TABLE products ADD FOREIGN KEY (product_group_id) REFERENCES
 product_groups;
```

Zum Hinzufügen eines NOT-NULL-Constraints, der normalerweise nicht als Tabellen-Constraint geschrieben wird, gibt es diese besondere Syntax:

```sql
ALTER TABLE products ALTER COLUMN product_no SET NOT NULL;
```

Dieser Befehl tut stillschweigend nichts, wenn die Spalte bereits einen NOT-NULL-Constraint hat.

Der Constraint wird sofort geprüft; die Tabellendaten müssen den Constraint also bereits erfüllen, bevor er hinzugefügt werden kann.

### 5.7.4. Einen Constraint entfernen

Um einen Constraint zu entfernen, müssen Sie seinen Namen kennen. Wenn Sie ihm selbst einen Namen gegeben haben, ist das einfach. Andernfalls hat das System einen generierten Namen vergeben, den Sie herausfinden müssen. Der `psql`-Befehl `\d table_name` kann dabei hilfreich sein; andere Schnittstellen bieten möglicherweise ebenfalls eine Möglichkeit, Tabellendetails anzuzeigen. Der Befehl lautet dann:

```sql
ALTER TABLE products DROP CONSTRAINT some_name;
```

Wie beim Löschen einer Spalte müssen Sie `CASCADE` hinzufügen, wenn Sie einen Constraint löschen möchten, von dem etwas anderes abhängt. Ein Beispiel ist ein Fremdschlüssel-Constraint, der von einem UNIQUE- oder Primärschlüssel-Constraint auf den referenzierten Spalten abhängt.

Für das Entfernen eines NOT-NULL-Constraints gibt es eine vereinfachte Syntax:

```sql
ALTER TABLE products ALTER COLUMN product_no DROP NOT NULL;
```

Dies spiegelt die Syntax `SET NOT NULL` zum Hinzufügen eines NOT-NULL-Constraints wider. Der Befehl tut stillschweigend nichts, wenn die Spalte keinen NOT-NULL-Constraint hat. Denken Sie daran, dass eine Spalte höchstens einen NOT-NULL-Constraint haben kann; daher ist nie mehrdeutig, auf welchen Constraint dieser Befehl wirkt.

### 5.7.5. Den Vorgabewert einer Spalte ändern

Um einen neuen Vorgabewert für eine Spalte zu setzen, verwenden Sie einen Befehl wie:

```sql
ALTER TABLE products ALTER COLUMN price SET DEFAULT 7.77;
```

Beachten Sie, dass dies keine vorhandenen Zeilen in der Tabelle betrifft; es ändert nur den Vorgabewert für zukünftige `INSERT`-Befehle.

Um einen Vorgabewert zu entfernen, verwenden Sie:

```sql
ALTER TABLE products ALTER COLUMN price DROP DEFAULT;
```

Das ist effektiv dasselbe, als würde der Vorgabewert auf null gesetzt. Daher ist es kein Fehler, einen Vorgabewert zu löschen, der gar nicht definiert war, weil der Vorgabewert implizit der Nullwert ist.

### 5.7.6. Den Datentyp einer Spalte ändern

Um eine Spalte in einen anderen Datentyp umzuwandeln, verwenden Sie einen Befehl wie:

```sql
ALTER TABLE products ALTER COLUMN price TYPE numeric(10,2);
```

Das ist nur erfolgreich, wenn jeder vorhandene Eintrag in der Spalte durch einen impliziten Cast in den neuen Typ umgewandelt werden kann. Wenn eine komplexere Umwandlung nötig ist, können Sie eine `USING`-Klausel hinzufügen, die angibt, wie die neuen Werte aus den alten berechnet werden.

PostgreSQL versucht außerdem, den Vorgabewert der Spalte, sofern vorhanden, sowie alle Constraints, die die Spalte betreffen, in den neuen Typ umzuwandeln. Diese Umwandlungen können jedoch fehlschlagen oder überraschende Ergebnisse erzeugen. Es ist oft am besten, alle Constraints auf der Spalte zu entfernen, bevor ihr Typ geändert wird, und anschließend passend geänderte Constraints wieder hinzuzufügen.

### 5.7.7. Eine Spalte umbenennen

So benennen Sie eine Spalte um:

```sql
ALTER TABLE products RENAME COLUMN product_no TO product_number;
```

### 5.7.8. Eine Tabelle umbenennen

So benennen Sie eine Tabelle um:

```sql
ALTER TABLE products RENAME TO items;
```

## 5.8. Privilegien

Wenn ein Objekt erzeugt wird, erhält es einen Eigentümer. Der Eigentümer ist normalerweise die Rolle, die die Erzeugungsanweisung ausgeführt hat. Bei den meisten Objektarten ist der Anfangszustand so, dass nur der Eigentümer oder ein Superuser etwas mit dem Objekt tun kann. Damit andere Rollen es verwenden können, müssen Privilegien erteilt werden.

Es gibt verschiedene Arten von Privilegien: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER`, `CREATE`, `CONNECT`, `TEMPORARY`, `EXECUTE`, `USAGE`, `SET`, `ALTER SYSTEM` und `MAINTAIN`. Welche Privilegien für ein bestimmtes Objekt gelten, hängt vom Objekttyp ab, etwa Tabelle oder Funktion. Die Bedeutung der Privilegien wird unten genauer beschrieben. Die folgenden Abschnitte und Kapitel zeigen außerdem, wie diese Privilegien verwendet werden.

Das Recht, ein Objekt zu ändern oder zu zerstören, ist Teil der Eigentümerschaft und kann nicht für sich genommen erteilt oder entzogen werden. Wie alle Privilegien kann dieses Recht jedoch an Mitglieder der Eigentümerrolle vererbt werden; siehe [Abschnitt 21.3](21_Datenbankrollen.md#213-rollenmitgliedschaft).

Ein Objekt kann mit einem passenden `ALTER`-Befehl einem neuen Eigentümer zugewiesen werden, zum Beispiel:

```sql
ALTER TABLE table_name OWNER TO new_owner;
```

Superuser können das immer tun. Gewöhnliche Rollen können es nur, wenn sie sowohl aktueller Eigentümer des Objekts sind oder die Privilegien der Eigentümerrolle erben, als auch mit `SET ROLE` zur neuen Eigentümerrolle wechseln dürfen.

Zum Zuweisen von Privilegien wird der Befehl `GRANT` verwendet. Wenn `joe` eine vorhandene Rolle und `accounts` eine vorhandene Tabelle ist, kann das Recht zum Aktualisieren der Tabelle zum Beispiel so erteilt werden:

```sql
GRANT UPDATE ON accounts TO joe;
```

Wenn statt eines bestimmten Privilegs `ALL` geschrieben wird, werden alle Privilegien erteilt, die für den Objekttyp relevant sind.

Der besondere Rollenname `PUBLIC` kann verwendet werden, um ein Privileg jeder Rolle im System zu erteilen. Außerdem können Gruppenrollen eingerichtet werden, um Privilegien bei vielen Datenbankbenutzern leichter zu verwalten; Details stehen in [Kapitel 21](21_Datenbankrollen.md).

Um ein zuvor erteiltes Privileg wieder zu entziehen, verwenden Sie den passend benannten Befehl `REVOKE`:

```sql
REVOKE ALL ON accounts FROM PUBLIC;
```

Normalerweise kann nur der Eigentümer eines Objekts oder ein Superuser Privilegien auf einem Objekt erteilen oder entziehen. Es ist jedoch möglich, ein Privileg "with grant option" zu erteilen; dadurch erhält der Empfänger das Recht, es seinerseits an andere weiterzugeben. Wenn die Grant-Option später entzogen wird, verlieren alle, die das Privileg von diesem Empfänger erhalten haben, direkt oder über eine Kette von Grant-Vorgängen, ebenfalls das Privileg. Details finden Sie auf den Referenzseiten zu `GRANT` und `REVOKE`.

Der Eigentümer eines Objekts kann eigene gewöhnliche Privilegien entziehen, etwa um eine Tabelle auch für sich selbst schreibgeschützt zu machen. Eigentümer werden jedoch immer so behandelt, als hätten sie alle Grant-Optionen, sodass sie sich ihre eigenen Privilegien jederzeit erneut erteilen können.

Die verfügbaren Privilegien sind:

- `SELECT`: Erlaubt `SELECT` aus beliebigen oder bestimmten Spalten einer Tabelle, View, materialisierten View oder eines anderen tabellenartigen Objekts. Erlaubt außerdem `COPY TO`. Dieses Privileg wird auch benötigt, um vorhandene Spaltenwerte in `UPDATE`, `DELETE` oder `MERGE` zu referenzieren. Bei Sequenzen erlaubt es außerdem die Verwendung der Funktion `currval`. Bei Large Objects erlaubt es, das Objekt zu lesen.
- `INSERT`: Erlaubt `INSERT` einer neuen Zeile in eine Tabelle, View usw. Es kann auf bestimmte Spalten erteilt werden; dann dürfen im `INSERT`-Befehl nur diese Spalten belegt werden, während andere Spalten ihre Vorgabewerte erhalten. Erlaubt außerdem `COPY FROM`.
- `UPDATE`: Erlaubt `UPDATE` beliebiger oder bestimmter Spalten einer Tabelle, View usw. In der Praxis benötigt jeder nichttriviale `UPDATE`-Befehl zusätzlich das `SELECT`-Privileg, weil er Tabellenspalten referenzieren muss, um zu bestimmen, welche Zeilen aktualisiert werden, oder um neue Spaltenwerte zu berechnen. `SELECT ... FOR UPDATE` und `SELECT ... FOR SHARE` benötigen zusätzlich zum `SELECT`-Privileg ebenfalls dieses Privileg auf mindestens einer Spalte. Bei Sequenzen erlaubt es die Verwendung der Funktionen `nextval` und `setval`. Bei Large Objects erlaubt es, das Objekt zu schreiben oder abzuschneiden.
- `DELETE`: Erlaubt `DELETE` einer Zeile aus einer Tabelle, View usw. In der Praxis benötigt jeder nichttriviale `DELETE`-Befehl zusätzlich das `SELECT`-Privileg, weil er Tabellenspalten referenzieren muss, um zu bestimmen, welche Zeilen gelöscht werden.
- `TRUNCATE`: Erlaubt `TRUNCATE` auf einer Tabelle.
- `REFERENCES`: Erlaubt das Erzeugen eines Fremdschlüssel-Constraints, der eine Tabelle oder bestimmte Spalten einer Tabelle referenziert.
- `TRIGGER`: Erlaubt das Erzeugen eines Triggers auf einer Tabelle, View usw.
- `CREATE`: Bei Datenbanken erlaubt es, neue Schemas und Publikationen innerhalb der Datenbank zu erzeugen, und erlaubt, vertrauenswürdige Erweiterungen in der Datenbank zu installieren. Bei Schemas erlaubt es, neue Objekte im Schema zu erzeugen. Um ein vorhandenes Objekt umzubenennen, müssen Sie Eigentümer des Objekts sein und dieses Privileg für das enthaltende Schema besitzen. Bei Tablespaces erlaubt es, Tabellen, Indizes und temporäre Dateien im Tablespace zu erzeugen, und erlaubt, Datenbanken zu erzeugen, die diesen Tablespace als Standard-Tablespace haben. Das Entziehen dieses Privilegs verändert die Existenz oder den Speicherort vorhandener Objekte nicht.
- `CONNECT`: Erlaubt dem Empfänger, sich mit der Datenbank zu verbinden. Dieses Privileg wird beim Verbindungsaufbau geprüft, zusätzlich zu den durch `pg_hba.conf` auferlegten Einschränkungen.
- `TEMPORARY`: Erlaubt das Erzeugen temporärer Tabellen während der Verwendung der Datenbank.
- `EXECUTE`: Erlaubt den Aufruf einer Funktion oder Prozedur, einschließlich der Verwendung von Operatoren, die auf dieser Funktion aufbauen. Dies ist der einzige Privilegtyp, der auf Funktionen und Prozeduren anwendbar ist.
- `USAGE`: Bei prozeduralen Sprachen erlaubt es die Verwendung der Sprache zur Erzeugung von Funktionen in dieser Sprache. Bei Schemas erlaubt es den Zugriff auf im Schema enthaltene Objekte, vorausgesetzt, die eigenen Privilegienanforderungen dieser Objekte sind ebenfalls erfüllt. Bei Sequenzen erlaubt es die Verwendung der Funktionen `currval` und `nextval`. Bei Typen und Domains erlaubt es die Verwendung des Typs oder der Domain beim Erzeugen von Tabellen, Funktionen und anderen Schemaobjekten; es steuert nicht jede "Verwendung" des Typs, etwa Werte des Typs in Abfragen. Bei Foreign Data Wrappers erlaubt es das Erzeugen neuer Server mit diesem Foreign Data Wrapper. Bei Foreign Servers erlaubt es das Erzeugen von Fremdtabellen mit diesem Server; Empfänger dürfen außerdem eigene User Mappings ändern oder löschen, die mit diesem Server verbunden sind.
- `SET`: Erlaubt, einen Server-Konfigurationsparameter innerhalb der aktuellen Sitzung auf einen neuen Wert zu setzen. Dieses Privileg kann zwar auf jedem Parameter erteilt werden, ist aber nur bei Parametern sinnvoll, für deren Setzen normalerweise Superuser-Privilegien erforderlich wären.
- `ALTER SYSTEM`: Erlaubt, einen Server-Konfigurationsparameter mit dem Befehl `ALTER SYSTEM` auf einen neuen Wert zu konfigurieren.
- `MAINTAIN`: Erlaubt `VACUUM`, `ANALYZE`, `CLUSTER`, `REFRESH MATERIALIZED VIEW`, `REINDEX`, `LOCK TABLE` und Funktionen zur Manipulation von Datenbankobjektstatistiken (siehe Tabelle 9.105) auf einer Relation.

Die von anderen Befehlen benötigten Privilegien sind auf der Referenzseite des jeweiligen Befehls aufgeführt.

PostgreSQL erteilt beim Erzeugen einiger Objekttypen standardmäßig Privilegien an `PUBLIC`. Standardmäßig werden `PUBLIC` keine Privilegien auf Tabellen, Tabellenspalten, Sequenzen, Foreign Data Wrappers, Foreign Servers, Large Objects, Schemas, Tablespaces oder Konfigurationsparameter erteilt. Für andere Objekttypen sind die standardmäßig an `PUBLIC` erteilten Privilegien: `CONNECT` und `TEMPORARY` für Datenbanken, `EXECUTE` für Funktionen und Prozeduren sowie `USAGE` für Sprachen und Datentypen einschließlich Domains. Der Objekteigentümer kann selbstverständlich sowohl Standardprivilegien als auch ausdrücklich erteilte Privilegien mit `REVOKE` entziehen. Für maximale Sicherheit sollte `REVOKE` in derselben Transaktion ausgeführt werden, die das Objekt erzeugt; dann gibt es kein Zeitfenster, in dem ein anderer Benutzer das Objekt verwenden kann. Außerdem können diese Standardprivilegien mit `ALTER DEFAULT PRIVILEGES` überschrieben werden.

Tabelle 5.1 zeigt die Ein-Buchstaben-Abkürzungen, die für diese Privilegtypen in ACL-Werten verwendet werden. Diese Buchstaben erscheinen in der Ausgabe der unten genannten `psql`-Befehle oder beim Betrachten von ACL-Spalten in Systemkatalogen.

**Tabelle 5.1. ACL-Privilegabkürzungen**

| Privileg | Abkürzung | Anwendbare Objekttypen |
|---|---|---|
| `SELECT` | `r` ("read") | `LARGE OBJECT`, `SEQUENCE`, `TABLE` und tabellenartige Objekte, Tabellenspalte |
| `INSERT` | `a` ("append") | `TABLE`, Tabellenspalte |
| `UPDATE` | `w` ("write") | `LARGE OBJECT`, `SEQUENCE`, `TABLE`, Tabellenspalte |
| `DELETE` | `d` | `TABLE` |
| `TRUNCATE` | `D` | `TABLE` |
| `REFERENCES` | `x` | `TABLE`, Tabellenspalte |
| `TRIGGER` | `t` | `TABLE` |
| `CREATE` | `C` | `DATABASE`, `SCHEMA`, `TABLESPACE` |
| `CONNECT` | `c` | `DATABASE` |
| `TEMPORARY` | `T` | `DATABASE` |
| `EXECUTE` | `X` | `FUNCTION`, `PROCEDURE` |
| `USAGE` | `U` | `DOMAIN`, `FOREIGN DATA WRAPPER`, `FOREIGN SERVER`, `LANGUAGE`, `SCHEMA`, `SEQUENCE`, `TYPE` |
| `SET` | `s` | `PARAMETER` |
| `ALTER SYSTEM` | `A` | `PARAMETER` |
| `MAINTAIN` | `m` | `TABLE` |

Tabelle 5.2 fasst die für jeden SQL-Objekttyp verfügbaren Privilegien zusammen und verwendet dabei die oben gezeigten Abkürzungen. Sie zeigt außerdem den `psql`-Befehl, mit dem die Privileg-Einstellungen für jeden Objekttyp geprüft werden können.

**Tabelle 5.2. Zusammenfassung der Zugriffsprivilegien**

| Objekttyp | Alle Privilegien | Standardprivilegien für `PUBLIC` | `psql`-Befehl |
|---|---|---|---|
| `DATABASE` | `CTc` | `Tc` | `\l` |
| `DOMAIN` | `U` | `U` | `\dD+` |
| `FUNCTION` oder `PROCEDURE` | `X` | `X` | `\df+` |
| `FOREIGN DATA WRAPPER` | `U` | keine | `\dew+` |
| `FOREIGN SERVER` | `U` | keine | `\des+` |
| `LANGUAGE` | `U` | `U` | `\dL+` |
| `LARGE OBJECT` | `rw` | keine | `\dl+` |
| `PARAMETER` | `sA` | keine | `\dconfig+` |
| `SCHEMA` | `UC` | keine | `\dn+` |
| `SEQUENCE` | `rwU` | keine | `\dp` |
| `TABLE` und tabellenartige Objekte | `arwdDxtm` | keine | `\dp` |
| Tabellenspalte | `arwx` | keine | `\dp` |
| `TABLESPACE` | `C` | keine | `\db+` |
| `TYPE` | `U` | `U` | `\dT+` |

Die für ein bestimmtes Objekt erteilten Privilegien werden als Liste von `aclitem`-Einträgen angezeigt, jeweils in diesem Format:

```sql
grantee=privilege-abbreviation[*].../grantor
```

Jeder `aclitem` listet alle Berechtigungen eines Empfängers auf, die von einem bestimmten Erteiler vergeben wurden. Einzelne Privilegien werden durch die Ein-Buchstaben-Abkürzungen aus Tabelle 5.1 dargestellt; ein angehängtes `*` bedeutet, dass das Privileg mit Grant-Option erteilt wurde. Zum Beispiel bedeutet `calvin=r*w/hobbes`, dass die Rolle `calvin` das Privileg `SELECT` (`r`) mit Grant-Option (`*`) sowie das nicht weiter vergebbare Privileg `UPDATE` (`w`) besitzt, beide erteilt durch die Rolle `hobbes`. Wenn `calvin` weitere Privilegien auf demselben Objekt von einem anderen Erteiler hat, erscheinen diese als separater `aclitem`-Eintrag. Ein leeres Empfängerfeld in einem `aclitem` steht für `PUBLIC`.

Als Beispiel: Angenommen, Benutzer `miriam` erzeugt die Tabelle `mytable` und führt aus:

```sql
GRANT SELECT ON mytable TO PUBLIC;
GRANT SELECT, UPDATE, INSERT ON mytable TO admin;
GRANT SELECT (col1), UPDATE (col1) ON mytable TO miriam_rw;
```

Dann würde der `psql`-Befehl `\dp` etwa Folgendes anzeigen:

```sql
=> \dp mytable
                                  Access privileges
 Schema | Name    | Type  |    Access privileges     |  Column privileges  | Policies
--------+---------+-------+--------------------------+---------------------+----------
 public | mytable | table | miriam=arwdDxtm/miriam+ | col1:              |
        |         |       | =r/miriam              + | miriam_rw=rw/miriam |
        |         |       | admin=arw/miriam         |                     |
(1 row)
```

Wenn die Spalte "Access privileges" für ein bestimmtes Objekt leer ist, bedeutet das, dass das Objekt Standardprivilegien hat, also dass sein Privileg-Eintrag im betreffenden Systemkatalog null ist. Standardprivilegien umfassen immer alle Privilegien für den Eigentümer und können je nach Objekttyp auch einige Privilegien für `PUBLIC` enthalten, wie oben erläutert. Das erste `GRANT` oder `REVOKE` auf einem Objekt materialisiert die Standardprivilegien, erzeugt also zum Beispiel `miriam=arwdDxt/miriam`, und verändert sie anschließend gemäß der angegebenen Anforderung. Entsprechend werden Einträge in "Column privileges" nur für Spalten mit nicht standardmäßigen Privilegien angezeigt. Für diesen Zweck bedeutet "Standardprivilegien" immer die eingebauten Standardprivilegien für den Objekttyp. Ein Objekt, dessen Privilegien durch `ALTER DEFAULT PRIVILEGES` beeinflusst wurden, wird immer mit einem ausdrücklichen Privileg-Eintrag angezeigt, der die Auswirkungen von `ALTER` enthält.

Beachten Sie, dass die impliziten Grant-Optionen des Eigentümers in der Privileg-Anzeige nicht markiert werden. Ein `*` erscheint nur, wenn Grant-Optionen ausdrücklich an jemanden erteilt wurden.

Die Spalte "Access privileges" zeigt `(none)`, wenn der Privileg-Eintrag des Objekts nicht null, aber leer ist. Das bedeutet, dass überhaupt keine Privilegien erteilt sind, nicht einmal an den Eigentümer; das ist eine seltene Situation. Der Eigentümer hat in diesem Fall weiterhin implizite Grant-Optionen und könnte sich seine Privilegien erneut erteilen, besitzt sie im Moment aber nicht.

## 5.9. Row-Security-Policies

Zusätzlich zum SQL-Standard-Privilegiensystem über `GRANT` können Tabellen Row-Security-Policies besitzen, die pro Benutzer einschränken, welche Zeilen von normalen Abfragen zurückgegeben oder durch Datenänderungsbefehle eingefügt, aktualisiert oder gelöscht werden dürfen. Dieses Feature ist auch als Row-Level Security bekannt. Standardmäßig haben Tabellen keine Policies; wenn ein Benutzer nach dem SQL-Privilegiensystem Zugriffsrechte auf eine Tabelle hat, stehen ihm daher alle Zeilen der Tabelle gleichermaßen zum Abfragen oder Aktualisieren zur Verfügung.

Wenn Row Security für eine Tabelle aktiviert ist, etwa mit `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`, muss jeder normale Zugriff auf die Tabelle zum Auswählen oder Ändern von Zeilen durch eine Row-Security-Policy erlaubt werden. Der Tabelleneigentümer unterliegt Row-Security-Policies allerdings typischerweise nicht. Wenn für die Tabelle keine Policy existiert, wird eine Default-Deny-Policy verwendet: Es sind dann keine Zeilen sichtbar oder änderbar. Operationen, die sich auf die gesamte Tabelle beziehen, etwa `TRUNCATE` und `REFERENCES`, unterliegen nicht der Row Security.

Row-Security-Policies können auf bestimmte Befehle, auf Rollen oder auf beides beschränkt sein. Eine Policy kann für `ALL`-Befehle oder für `SELECT`, `INSERT`, `UPDATE` oder `DELETE` gelten. Einer Policy können mehrere Rollen zugeordnet werden; dabei gelten die normalen Regeln für Rollenmitgliedschaft und Vererbung.

Um festzulegen, welche Zeilen gemäß einer Policy sichtbar oder änderbar sind, wird ein Ausdruck benötigt, der ein boolesches Ergebnis liefert. Dieser Ausdruck wird für jede Zeile ausgewertet, bevor Bedingungen oder Funktionen aus der Benutzerabfrage angewendet werden. Die einzigen Ausnahmen sind `LEAKPROOF`-Funktionen, die garantiert keine Informationen preisgeben; der Optimierer kann solche Funktionen vor der Row-Security-Prüfung anwenden. Zeilen, für die der Ausdruck nicht `true` ergibt, werden nicht verarbeitet. Es können getrennte Ausdrücke angegeben werden, um unabhängig zu steuern, welche Zeilen sichtbar sind und welche Zeilen geändert werden dürfen. Policy-Ausdrücke werden als Teil der Abfrage und mit den Privilegien des Benutzers ausgeführt, der die Abfrage ausführt; allerdings können Security-Definer-Funktionen verwendet werden, um auf Daten zuzugreifen, die dem aufrufenden Benutzer nicht zur Verfügung stehen.

Superuser und Rollen mit dem Attribut `BYPASSRLS` umgehen das Row-Security-System beim Zugriff auf eine Tabelle immer. Tabelleneigentümer umgehen Row Security normalerweise ebenfalls, können aber mit `ALTER TABLE ... FORCE ROW LEVEL SECURITY` wählen, selbst der Row Security zu unterliegen.

Row Security zu aktivieren oder zu deaktivieren sowie Policies zu einer Tabelle hinzuzufügen, ist immer ausschließlich dem Tabelleneigentümer vorbehalten.

Policies werden mit `CREATE POLICY` erzeugt, mit `ALTER POLICY` geändert und mit `DROP POLICY` gelöscht. Um Row Security für eine bestimmte Tabelle zu aktivieren oder zu deaktivieren, verwenden Sie `ALTER TABLE`.

Jede Policy hat einen Namen, und für eine Tabelle können mehrere Policies definiert werden. Da Policies tabellenspezifisch sind, muss jede Policy einer Tabelle einen eindeutigen Namen haben. Verschiedene Tabellen können Policies mit demselben Namen besitzen.

Wenn mehrere Policies auf eine bestimmte Abfrage anwendbar sind, werden sie entweder mit `OR` kombiniert, bei permissiven Policies, die der Standard sind, oder mit `AND`, bei restriktiven Policies. Das `OR`-Verhalten ähnelt der Regel, dass eine Rolle die Privilegien aller Rollen besitzt, in denen sie Mitglied ist. Permissive und restriktive Policies werden weiter unten genauer besprochen.

Als einfaches Beispiel wird hier eine Policy auf der Relation `account` erzeugt, die nur Mitgliedern der Rolle `managers` Zugriff auf Zeilen erlaubt, und zwar nur auf die Zeilen ihrer eigenen Konten:

```sql
CREATE TABLE accounts (manager text, company text, contact_email text);

ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY account_managers ON accounts TO managers
    USING (manager = current_user);
```

Die obige Policy stellt implizit eine `WITH CHECK`-Klausel bereit, die mit ihrer `USING`-Klausel identisch ist. Dadurch gilt die Einschränkung sowohl für Zeilen, die von einem Befehl ausgewählt werden, sodass ein Manager keine vorhandenen Zeilen eines anderen Managers mit `SELECT`, `UPDATE` oder `DELETE` bearbeiten kann, als auch für Zeilen, die durch einen Befehl geändert werden, sodass Zeilen eines anderen Managers nicht per `INSERT` oder `UPDATE` erzeugt werden können.

Wenn keine Rolle angegeben ist oder der besondere Benutzername `PUBLIC` verwendet wird, gilt die Policy für alle Benutzer im System. Um allen Benutzern nur Zugriff auf ihre eigene Zeile in einer Tabelle `users` zu erlauben, kann eine einfache Policy verwendet werden:

```sql
CREATE POLICY user_policy ON users
    USING (user_name = current_user);
```

Dies funktioniert ähnlich wie das vorige Beispiel.

Um für Zeilen, die einer Tabelle hinzugefügt werden, eine andere Policy zu verwenden als für sichtbare Zeilen, können mehrere Policies kombiniert werden. Dieses Policy-Paar würde allen Benutzern erlauben, alle Zeilen der Tabelle `users` zu sehen, aber nur ihre eigenen zu ändern:

```sql
CREATE POLICY user_sel_policy ON users
    FOR SELECT
    USING (true);
CREATE POLICY user_mod_policy ON users
    USING (user_name = current_user);
```

In einem `SELECT`-Befehl werden diese beiden Policies mit `OR` kombiniert, sodass im Ergebnis alle Zeilen ausgewählt werden können. Bei anderen Befehlsarten gilt nur die zweite Policy, sodass die Wirkung dieselbe ist wie zuvor.

Row Security kann ebenfalls mit `ALTER TABLE` deaktiviert werden. Das Deaktivieren von Row Security entfernt keine Policies, die auf der Tabelle definiert sind; sie werden lediglich ignoriert. Dann sind alle Zeilen der Tabelle sichtbar und änderbar, vorbehaltlich des Standard-SQL-Privilegiensystems.

Das folgende größere Beispiel zeigt, wie dieses Feature in Produktionsumgebungen verwendet werden kann. Die Tabelle `passwd` bildet eine Unix-Passwortdatei nach:

```sql
-- Einfaches Beispiel auf Basis einer passwd-Datei
CREATE TABLE passwd (
   user_name            text UNIQUE NOT NULL,
   pwhash               text,
   uid                  int PRIMARY KEY,
   gid                  int NOT NULL,
   real_name            text NOT NULL,
   home_phone           text,
   extra_info           text,
   home_dir             text NOT NULL,
   shell                text NOT NULL
);

CREATE ROLE admin;            -- Administrator
CREATE ROLE bob;              -- Normaler Benutzer
CREATE ROLE alice;            -- Normaler Benutzer

-- Tabelle befüllen
INSERT INTO passwd VALUES
  ('admin','xxx',0,0,'Admin','111-222-3333',null,'/root','/bin/dash');
INSERT INTO passwd VALUES
  ('bob','xxx',1,1,'Bob','123-456-7890',null,'/home/bob','/bin/zsh');
INSERT INTO passwd VALUES
  ('alice','xxx',2,1,'Alice','098-765-4321',null,'/home/alice','/bin/zsh');

-- Row-Level Security auf der Tabelle aktivieren
ALTER TABLE passwd ENABLE ROW LEVEL SECURITY;

-- Policies anlegen
-- Der Administrator kann alle Zeilen sehen und beliebige Zeilen hinzufügen
CREATE POLICY admin_all ON passwd TO admin USING (true) WITH CHECK (true);
-- Normale Benutzer können alle Zeilen sehen
CREATE POLICY all_view ON passwd FOR SELECT USING (true);
-- Normale Benutzer können ihre eigenen Datensätze ändern;
-- die erlaubten Shells werden aber eingeschränkt
CREATE POLICY user_mod ON passwd FOR UPDATE
  USING (current_user = user_name)
  WITH CHECK (
     current_user = user_name AND
     shell IN ('/bin/bash','/bin/sh','/bin/dash','/bin/zsh','/bin/tcsh')
  );

-- Dem Administrator alle normalen Rechte erlauben
GRANT SELECT, INSERT, UPDATE, DELETE ON passwd TO admin;
-- Benutzer erhalten nur SELECT-Zugriff auf öffentliche Spalten
GRANT SELECT
  (user_name, uid, gid, real_name, home_phone, extra_info, home_dir, shell)
  ON passwd TO public;
-- Benutzer dürfen bestimmte Spalten aktualisieren
GRANT UPDATE
  (pwhash, real_name, home_phone, extra_info, shell)
  ON passwd TO public;
```

Wie bei allen Sicherheitseinstellungen ist es wichtig zu testen und sicherzustellen, dass sich das System wie erwartet verhält. Mit dem obigen Beispiel lässt sich zeigen, dass das Berechtigungssystem korrekt arbeitet.

```sql
-- admin kann alle Zeilen und Felder sehen
postgres=> set role admin;
SET
postgres=> table passwd;
 user_name | pwhash | uid | gid | real_name | home_phone  | extra_info | home_dir    |   shell
-----------+--------+-----+-----+-----------+-------------+------------+-------------+-----------
 admin     | xxx    |   0 |   0 | Admin     | 111-222-3333|            | /root       | /bin/dash
 bob       | xxx    |   1 |   1 | Bob       | 123-456-7890|            | /home/bob   | /bin/zsh
 alice     | xxx    |   2 |   1 | Alice     | 098-765-4321|            | /home/alice | /bin/zsh
(3 rows)

-- Testen, was Alice tun kann
postgres=> set role alice;
SET
postgres=> table passwd;
ERROR: permission denied for table passwd
postgres=> select user_name,real_name,home_phone,extra_info,home_dir,shell from passwd;
 user_name | real_name | home_phone  | extra_info | home_dir    |   shell
-----------+-----------+-------------+------------+-------------+-----------
 admin     | Admin     | 111-222-3333|            | /root       | /bin/dash
 bob       | Bob       | 123-456-7890|            | /home/bob   | /bin/zsh
 alice     | Alice     | 098-765-4321|            | /home/alice | /bin/zsh
(3 rows)

postgres=> update passwd set user_name = 'joe';
ERROR: permission denied for table passwd
-- Alice darf ihren eigenen real_name ändern, aber keine anderen
postgres=> update passwd set real_name = 'Alice Doe';
UPDATE 1
postgres=> update passwd set real_name = 'John Doe' where user_name = 'admin';
UPDATE 0
postgres=> update passwd set shell = '/bin/xx';
ERROR: new row violates WITH CHECK OPTION for "passwd"
postgres=> delete from passwd;
ERROR: permission denied for table passwd
postgres=> insert into passwd (user_name) values ('xxx');
ERROR: permission denied for table passwd
-- Alice kann ihr eigenes Passwort ändern; RLS verhindert stillschweigend
-- Aktualisierungen anderer Zeilen
postgres=> update passwd set pwhash = 'abc';
UPDATE 1
```

Alle bisher konstruierten Policies waren permissive Policies. Das bedeutet, dass mehrere angewendete Policies mit dem booleschen Operator `OR` kombiniert werden. Permissive Policies können so aufgebaut werden, dass sie Zugriff nur in den beabsichtigten Fällen erlauben; oft ist es aber einfacher, permissive Policies mit restriktiven Policies zu kombinieren. Restriktive Policies müssen von den Datensätzen erfüllt werden und werden mit dem booleschen Operator `AND` kombiniert. Aufbauend auf dem obigen Beispiel fügen wir eine restriktive Policy hinzu, die verlangt, dass der Administrator über einen lokalen Unix-Socket verbunden ist, um auf die Datensätze der Tabelle `passwd` zugreifen zu können:

```sql
CREATE POLICY admin_local_only ON passwd AS RESTRICTIVE TO admin
    USING (pg_catalog.inet_client_addr() IS NULL);
```

Dann sieht man, dass ein Administrator, der über das Netzwerk verbunden ist, wegen der restriktiven Policy keine Datensätze sieht:

```sql
=> SELECT current_user;
 current_user
--------------
 admin
(1 row)

=> select inet_client_addr();
 inet_client_addr
------------------
 127.0.0.1
(1 row)

=> TABLE passwd;
 user_name | pwhash | uid | gid | real_name | home_phone | extra_info | home_dir | shell
-----------+--------+-----+-----+-----------+------------+------------+----------+-------
(0 rows)

=> UPDATE passwd set pwhash = NULL;
UPDATE 0
```

Prüfungen der referenziellen Integrität, etwa UNIQUE- oder Primärschlüssel-Constraints und Fremdschlüsselreferenzen, umgehen Row Security immer, damit die Datenintegrität erhalten bleibt. Beim Entwerfen von Schemas und Row-Level-Policies muss darauf geachtet werden, Informationslecks über solche Prüfungen der referenziellen Integrität zu vermeiden.

In manchen Kontexten ist es wichtig, sicher zu sein, dass Row Security nicht angewendet wird. Beim Erstellen eines Backups wäre es zum Beispiel verheerend, wenn Row Security stillschweigend dazu führen würde, dass einige Zeilen im Backup fehlen. In einer solchen Situation können Sie den Konfigurationsparameter `row_security` auf `off` setzen. Das umgeht Row Security nicht selbst; es löst vielmehr einen Fehler aus, wenn die Ergebnisse einer Abfrage durch eine Policy gefiltert würden. Der Grund für den Fehler kann dann untersucht und behoben werden.

In den obigen Beispielen betrachten die Policy-Ausdrücke nur die aktuellen Werte der Zeile, auf die zugegriffen wird oder die aktualisiert wird. Das ist der einfachste und leistungsfähigste Fall; wenn möglich, sollten Row-Security-Anwendungen so entworfen werden. Wenn für eine Policy-Entscheidung andere Zeilen oder andere Tabellen herangezogen werden müssen, kann dies mit Unter-`SELECT`s oder Funktionen geschehen, die `SELECT`s enthalten. Beachten Sie jedoch, dass solche Zugriffe Race Conditions erzeugen können, die bei fehlender Sorgfalt Informationslecks ermöglichen. Betrachten Sie als Beispiel den folgenden Tabellenentwurf:

```sql
-- Definition der Privileggruppen
CREATE TABLE groups (group_id int PRIMARY KEY,
                     group_name text NOT NULL);

INSERT INTO groups VALUES
  (1, 'low'),
  (2, 'medium'),
  (5, 'high');

GRANT ALL ON groups TO alice; -- alice ist die Administratorin
GRANT SELECT ON groups TO public;

-- Definition der Privilegstufen der Benutzer
CREATE TABLE users (user_name text PRIMARY KEY,
                    group_id int NOT NULL REFERENCES groups);

INSERT INTO users VALUES
  ('alice', 5),
  ('bob', 2),
  ('mallory', 2);

GRANT ALL ON users TO alice;
GRANT SELECT ON users TO public;

-- Tabelle mit den zu schützenden Informationen
CREATE TABLE information (info text,
                          group_id int NOT NULL REFERENCES groups);

INSERT INTO information VALUES
  ('barely secret', 1),
  ('slightly secret', 2),
  ('very secret', 5);

ALTER TABLE information ENABLE ROW LEVEL SECURITY;

-- Eine Zeile soll für Benutzer sichtbar/aktualisierbar sein,
-- deren Sicherheits-group_id größer oder gleich der group_id der Zeile ist
CREATE POLICY fp_s ON information FOR SELECT
  USING (group_id <= (SELECT group_id FROM users WHERE user_name = current_user));
CREATE POLICY fp_u ON information FOR UPDATE
  USING (group_id <= (SELECT group_id FROM users WHERE user_name = current_user));

-- Zum Schutz der Tabelle information verlassen wir uns nur auf RLS
GRANT ALL ON information TO public;
```

Angenommen nun, `alice` möchte die Information "slightly secret" ändern, entscheidet aber, dass `mallory` dem neuen Inhalt dieser Zeile nicht vertrauen soll. Sie führt daher aus:

```sql
BEGIN;
UPDATE users SET group_id = 1 WHERE user_name = 'mallory';
UPDATE information SET info = 'secret from mallory' WHERE group_id = 2;
COMMIT;
```

Das sieht sicher aus; es scheint kein Zeitfenster zu geben, in dem `mallory` die Zeichenkette "secret from mallory" sehen könnte. Es gibt hier jedoch eine Race Condition. Wenn `mallory` gleichzeitig zum Beispiel Folgendes ausführt:

```sql
SELECT * FROM information WHERE group_id = 2 FOR UPDATE;
```

und ihre Transaktion im Modus `READ COMMITTED` läuft, kann sie "secret from mallory" sehen. Das geschieht, wenn ihre Transaktion die Zeile in `information` kurz nach `alice` erreicht. Sie blockiert, bis `alice`' Transaktion bestätigt ist, und holt dann dank der `FOR UPDATE`-Klausel den aktualisierten Zeileninhalt. Sie holt jedoch keine aktualisierte Zeile für das implizite `SELECT` aus `users`, weil dieses Unter-`SELECT` kein `FOR UPDATE` hatte; stattdessen wird die Zeile aus `users` mit dem Snapshot gelesen, der zu Beginn der Abfrage genommen wurde. Daher testet der Policy-Ausdruck den alten Wert von `mallory`' Privilegstufe und erlaubt ihr, die aktualisierte Zeile zu sehen.

Es gibt mehrere Möglichkeiten, dieses Problem zu umgehen. Eine einfache Antwort ist, `SELECT ... FOR SHARE` in Unter-`SELECT`s von Row-Security-Policies zu verwenden. Das erfordert jedoch, den betroffenen Benutzern das `UPDATE`-Privileg auf der referenzierten Tabelle, hier `users`, zu erteilen, was unerwünscht sein kann. Eine weitere Row-Security-Policy könnte sie allerdings daran hindern, dieses Privileg tatsächlich auszuüben; oder das Unter-`SELECT` könnte in eine Security-Definer-Funktion eingebettet werden. Außerdem kann intensive gleichzeitige Nutzung von Row-Share-Locks auf der referenzierten Tabelle ein Leistungsproblem darstellen, besonders wenn sie häufig aktualisiert wird. Eine weitere Lösung, praktikabel bei seltenen Aktualisierungen der referenzierten Tabelle, besteht darin, beim Aktualisieren der referenzierten Tabelle einen `ACCESS EXCLUSIVE`-Lock darauf zu nehmen, sodass keine gleichzeitigen Transaktionen alte Zeilenwerte untersuchen können. Oder man wartet nach dem Bestätigen einer Aktualisierung der referenzierten Tabelle einfach, bis alle gleichzeitigen Transaktionen beendet sind, bevor Änderungen vorgenommen werden, die sich auf die neue Sicherheitssituation stützen.

Weitere Details finden Sie unter `CREATE POLICY` und `ALTER TABLE`.

## 5.10. Schemata

Ein PostgreSQL-Datenbank-Cluster enthält eine oder mehrere benannte Datenbanken. Rollen und einige andere Objekttypen werden clusterweit gemeinsam genutzt. Eine Client-Verbindung zum Server kann nur auf Daten in einer einzigen Datenbank zugreifen, nämlich auf die Datenbank, die in der Verbindungsanfrage angegeben wurde.

> **Hinweis:** Benutzer eines Clusters haben nicht notwendigerweise das Privileg, auf jede Datenbank des Clusters zuzugreifen. Weil Rollennamen gemeinsam genutzt werden, kann es nicht zwei unterschiedliche Rollen mit demselben Namen, etwa `joe`, in zwei Datenbanken desselben Clusters geben. Das System kann aber so konfiguriert werden, dass `joe` nur auf einige Datenbanken zugreifen darf.

Eine Datenbank enthält ein oder mehrere benannte Schemata, die wiederum Tabellen enthalten. Schemata enthalten außerdem andere Arten benannter Objekte, darunter Datentypen, Funktionen und Operatoren. Innerhalb eines Schemas können zwei Objekte desselben Typs nicht denselben Namen haben. Außerdem teilen sich Tabellen, Sequenzen, Indizes, Views, materialisierte Views und Fremdtabellen denselben Namensraum; deshalb müssen zum Beispiel ein Index und eine Tabelle unterschiedliche Namen haben, wenn sie sich im selben Schema befinden. Derselbe Objektname kann in unterschiedlichen Schemata ohne Konflikt verwendet werden; beispielsweise können sowohl `schema1` als auch `myschema` Tabellen namens `mytable` enthalten. Anders als Datenbanken sind Schemata nicht strikt voneinander getrennt: Ein Benutzer kann auf Objekte in jedem Schema der Datenbank zugreifen, mit der er verbunden ist, sofern er die dafür nötigen Privilegien besitzt.

Es gibt mehrere Gründe, Schemata zu verwenden:

- Sie erlauben vielen Benutzern, eine Datenbank zu verwenden, ohne sich gegenseitig zu stören.
- Sie organisieren Datenbankobjekte in logische Gruppen und machen sie damit leichter verwaltbar.
- Anwendungen von Drittanbietern können in eigene Schemata gelegt werden, damit ihre Namen nicht mit denen anderer Objekte kollidieren.

Schemata sind auf Betriebssystemebene mit Verzeichnissen vergleichbar, können jedoch nicht verschachtelt werden.

### 5.10.1. Ein Schema erstellen

Um ein Schema zu erstellen, verwenden Sie den Befehl `CREATE SCHEMA`. Geben Sie dem Schema einen Namen Ihrer Wahl. Zum Beispiel:

```sql
CREATE SCHEMA myschema;
```

Um Objekte in einem Schema zu erstellen oder darauf zuzugreifen, schreiben Sie einen qualifizierten Namen, der aus dem Schemanamen und dem Tabellennamen besteht, getrennt durch einen Punkt:

```sql
schema.table
```

Das funktioniert überall dort, wo ein Tabellenname erwartet wird, einschließlich der Befehle zur Tabellenänderung und der Datenzugriffsbefehle, die in den folgenden Kapiteln besprochen werden. Der Kürze halber sprechen wir hier nur von Tabellen; dieselben Ideen gelten aber auch für andere Arten benannter Objekte, etwa Typen und Funktionen.

Auch die noch allgemeinere Syntax

```sql
database.schema.table
```

kann verwendet werden, derzeit jedoch nur zur formalen Übereinstimmung mit dem SQL-Standard. Wenn Sie einen Datenbanknamen angeben, muss er mit der Datenbank übereinstimmen, mit der Sie verbunden sind.

Um also eine Tabelle im neuen Schema zu erstellen, verwenden Sie:

```sql
CREATE TABLE myschema.mytable (
 ...
);
```

Um ein Schema zu löschen, wenn es leer ist, also wenn alle darin enthaltenen Objekte gelöscht wurden, verwenden Sie:

```sql
DROP SCHEMA myschema;
```

Um ein Schema einschließlich aller enthaltenen Objekte zu löschen, verwenden Sie:

```sql
DROP SCHEMA myschema CASCADE;
```

Eine Beschreibung des allgemeinen Mechanismus dahinter finden Sie in [Abschnitt 5.15](05_Datendefinition.md#515-abhängigkeitsverfolgung).

Oft möchten Sie ein Schema erstellen, das jemand anderem gehört, da dies eine Möglichkeit ist, die Aktivitäten Ihrer Benutzer auf genau definierte Namensräume zu beschränken. Die Syntax dafür lautet:

```sql
CREATE SCHEMA schema_name AUTHORIZATION user_name;
```

Sie können sogar den Schemanamen weglassen; dann ist der Schemaname derselbe wie der Benutzername. Siehe [Abschnitt 5.10.6](#5106-nutzungsmuster), warum das nützlich sein kann.

Schemanamen, die mit `pg_` beginnen, sind für Systemzwecke reserviert und können von Benutzern nicht erstellt werden.

### 5.10.2. Das Schema `public`

In den vorigen Abschnitten haben wir Tabellen erstellt, ohne Schemanamen anzugeben. Standardmäßig werden solche Tabellen und andere Objekte automatisch in ein Schema namens `public` gelegt. Jede neue Datenbank enthält ein solches Schema. Daher sind die folgenden Befehle gleichwertig:

```sql
CREATE TABLE products ( ... );
```

und:

```sql
CREATE TABLE public.products ( ... );
```

### 5.10.3. Der Schema-Suchpfad

Qualifizierte Namen sind mühsam zu schreiben, und häufig ist es ohnehin besser, keinen bestimmten Schemanamen fest in Anwendungen einzubauen. Deshalb werden Tabellen oft über unqualifizierte Namen angesprochen, die nur aus dem Tabellennamen bestehen. Das System bestimmt anhand eines Suchpfads, welche Tabelle gemeint ist. Dieser Suchpfad ist eine Liste von Schemata, in denen gesucht wird. Die erste passende Tabelle im Suchpfad wird als die gewünschte Tabelle verwendet. Wenn im Suchpfad keine Übereinstimmung gefunden wird, wird ein Fehler gemeldet, auch wenn es in anderen Schemata der Datenbank passende Tabellennamen gibt.

Die Möglichkeit, gleichnamige Objekte in unterschiedlichen Schemata zu erstellen, erschwert es, eine Abfrage zu schreiben, die jedes Mal exakt dieselben Objekte referenziert. Außerdem eröffnet sie Benutzern die Möglichkeit, das Verhalten der Abfragen anderer Benutzer absichtlich oder versehentlich zu verändern. Weil unqualifizierte Namen in Abfragen und intern in PostgreSQL weit verbreitet sind, bedeutet das Hinzufügen eines Schemas zu `search_path` effektiv, dass allen Benutzern mit `CREATE`-Privileg auf diesem Schema vertraut wird. Wenn Sie eine gewöhnliche Abfrage ausführen, kann ein böswilliger Benutzer, der Objekte in einem Schema Ihres Suchpfads erstellen darf, die Kontrolle übernehmen und beliebige SQL-Funktionen ausführen, als hätten Sie sie selbst ausgeführt.

Das erste im Suchpfad genannte Schema heißt aktuelles Schema. Es ist nicht nur das erste Schema, in dem gesucht wird, sondern auch das Schema, in dem neue Tabellen erstellt werden, wenn der Befehl `CREATE TABLE` keinen Schemanamen angibt.

Um den aktuellen Suchpfad anzuzeigen, verwenden Sie:

```sql
SHOW search_path;
```

In der Standardkonfiguration liefert dies:

```sql
 search_path
--------------
 "$user", public
```

Das erste Element gibt an, dass ein Schema mit demselben Namen wie der aktuelle Benutzer gesucht werden soll. Wenn ein solches Schema nicht existiert, wird der Eintrag ignoriert. Das zweite Element verweist auf das Schema `public`, das wir bereits kennengelernt haben.

Das erste Schema im Suchpfad, das existiert, ist der Standardort für neu erstellte Objekte. Deshalb werden Objekte standardmäßig im Schema `public` erstellt. Wenn Objekte in einem anderen Kontext ohne Schemakennzeichnung referenziert werden, etwa bei Tabellenänderungen, Datenänderungen oder Abfragen, wird der Suchpfad durchlaufen, bis ein passendes Objekt gefunden wird. In der Standardkonfiguration kann daher jeder unqualifizierte Zugriff wiederum nur auf das Schema `public` verweisen.

Um unser neues Schema in den Suchpfad aufzunehmen, verwenden wir:

```sql
SET search_path TO myschema,public;
```

Wir lassen `$user` hier weg, weil wir es im Moment nicht brauchen. Danach können wir ohne Schemakennzeichnung auf die Tabelle zugreifen:

```sql
DROP TABLE mytable;
```

Da `myschema` außerdem das erste Element im Suchpfad ist, würden neue Objekte standardmäßig darin erstellt.

Wir hätten auch schreiben können:

```sql
SET search_path TO myschema;
```

Dann hätten wir ohne ausdrückliche Qualifizierung keinen Zugriff mehr auf das Schema `public`. Am Schema `public` ist nichts Besonderes, außer dass es standardmäßig existiert. Es kann ebenfalls gelöscht werden.

Weitere Möglichkeiten zur Manipulation des Schema-Suchpfads finden Sie auch in [Abschnitt 9.27](09_Funktionen_und_Operatoren.md#927-systeminformationsfunktionen-und-operatoren).

Der Suchpfad funktioniert für Datentypnamen, Funktionsnamen und Operatornamen genauso wie für Tabellennamen. Namen von Datentypen und Funktionen können genauso qualifiziert werden wie Tabellennamen. Wenn Sie in einem Ausdruck einen qualifizierten Operatornamen schreiben müssen, gibt es eine besondere Schreibweise:

```sql
OPERATOR(schema.operator)
```

Dies ist nötig, um syntaktische Mehrdeutigkeit zu vermeiden. Ein Beispiel:

```sql
SELECT 3 OPERATOR(pg_catalog.+) 4;
```

In der Praxis verlässt man sich bei Operatoren normalerweise auf den Suchpfad, um nicht so etwas Unhandliches schreiben zu müssen.

### 5.10.4. Schemata und Privilegien

Standardmäßig können Benutzer nicht auf Objekte in Schemata zugreifen, die ihnen nicht gehören. Damit dies möglich wird, muss der Eigentümer des Schemas das Privileg `USAGE` auf dem Schema erteilen. Standardmäßig hat jeder dieses Privileg auf dem Schema `public`. Damit Benutzer die Objekte in einem Schema tatsächlich verwenden können, müssen je nach Objekttyp eventuell weitere Privilegien erteilt werden.

Einem Benutzer kann außerdem erlaubt werden, Objekte im Schema eines anderen Benutzers zu erstellen. Dazu muss das Privileg `CREATE` auf dem Schema erteilt werden. In Datenbanken, die von PostgreSQL 14 oder früher aktualisiert wurden, besitzt jeder dieses Privileg auf dem Schema `public`. Einige Nutzungsmuster sehen vor, dieses Privileg zu entziehen:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

Das erste `public` ist das Schema, das zweite `PUBLIC` bedeutet „jeder Benutzer“. Im ersten Sinn ist es ein Bezeichner, im zweiten ein Schlüsselwort; daher die unterschiedliche Großschreibung. Vergleichen Sie dazu die Richtlinien aus [Abschnitt 4.1.1](04_SQL_Syntax.md#411-bezeichner-und-schlüsselwörter).

### 5.10.5. Das Systemkatalog-Schema

Zusätzlich zu `public` und benutzerdefinierten Schemata enthält jede Datenbank ein Schema `pg_catalog`, das die Systemtabellen und alle eingebauten Datentypen, Funktionen und Operatoren enthält. `pg_catalog` ist immer faktisch Teil des Suchpfads. Wenn es im Pfad nicht ausdrücklich genannt ist, wird es implizit vor den Schemata des Suchpfads durchsucht. Dadurch ist sichergestellt, dass eingebaute Namen immer gefunden werden. Sie können `pg_catalog` jedoch ausdrücklich ans Ende Ihres Suchpfads setzen, wenn Sie möchten, dass benutzerdefinierte Namen eingebaute Namen übersteuern.

Da Systemtabellennamen mit `pg_` beginnen, sollten solche Namen vermieden werden, damit kein Konflikt entsteht, falls eine zukünftige Version eine Systemtabelle mit demselben Namen wie Ihre Tabelle definiert. Beim Standard-Suchpfad würde ein unqualifizierter Verweis auf Ihren Tabellennamen dann stattdessen als Systemtabelle aufgelöst. Systemtabellen werden weiterhin der Konvention folgen, Namen mit `pg_` zu verwenden, sodass sie nicht mit unqualifizierten Benutzertabellennamen kollidieren, solange Benutzer das Präfix `pg_` vermeiden.

### 5.10.6. Nutzungsmuster

Schemata können verwendet werden, um Daten auf viele Arten zu organisieren. Ein sicheres Schema-Nutzungsmuster verhindert, dass nicht vertrauenswürdige Benutzer das Verhalten der Abfragen anderer Benutzer verändern. Wenn eine Datenbank kein sicheres Schema-Nutzungsmuster verwendet, sollten Benutzer, die diese Datenbank sicher abfragen möchten, zu Beginn jeder Sitzung Schutzmaßnahmen ergreifen. Konkret würden sie jede Sitzung damit beginnen, `search_path` auf die leere Zeichenkette zu setzen oder anderweitig Schemata aus `search_path` zu entfernen, die von Nicht-Superusern beschreibbar sind. Es gibt einige Nutzungsmuster, die von der Standardkonfiguration leicht unterstützt werden:

- Gewöhnliche Benutzer auf benutzerprivate Schemata beschränken. Stellen Sie zur Umsetzung dieses Musters zunächst sicher, dass keine Schemata öffentliche `CREATE`-Privilegien haben. Erstellen Sie dann für jeden Benutzer, der nichttemporäre Objekte erstellen muss, ein Schema mit demselben Namen wie dieser Benutzer, zum Beispiel:

```sql
CREATE SCHEMA alice AUTHORIZATION alice;
```

  Denken Sie daran, dass der Standardsuchpfad mit `$user` beginnt, was zum Benutzernamen aufgelöst wird. Wenn jeder Benutzer ein eigenes Schema hat, greift er daher standardmäßig auf sein eigenes Schema zu. Dieses Muster ist ein sicheres Schema-Nutzungsmuster, sofern kein nicht vertrauenswürdiger Benutzer Eigentümer der Datenbank ist oder `ADMIN OPTION` auf einer relevanten Rolle erhalten hat; in diesem Fall gibt es kein sicheres Schema-Nutzungsmuster.

  In PostgreSQL 15 und später unterstützt die Standardkonfiguration dieses Nutzungsmuster. In früheren Versionen oder bei einer Datenbank, die aus einer früheren Version aktualisiert wurde, müssen Sie das öffentliche `CREATE`-Privileg vom Schema `public` entfernen:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

  Anschließend sollten Sie erwägen, das Schema `public` auf Objekte zu prüfen, deren Namen Objekten im Schema `pg_catalog` ähneln.

- Das Schema `public` aus dem Standardsuchpfad entfernen, indem Sie `postgresql.conf` ändern oder Folgendes ausführen:

```sql
ALTER ROLE ALL SET search_path = "$user";
```

  Erteilen Sie dann Privilegien zum Erstellen im Schema `public`. Nur qualifizierte Namen wählen dann Objekte aus dem Schema `public`. Qualifizierte Tabellenverweise sind unproblematisch, aber Aufrufe von Funktionen im Schema `public` sind unsicher oder unzuverlässig. Wenn Sie Funktionen oder Erweiterungen im Schema `public` erstellen, verwenden Sie stattdessen das erste Muster. Andernfalls ist dies wie das erste Muster sicher, sofern kein nicht vertrauenswürdiger Benutzer Eigentümer der Datenbank ist oder `ADMIN OPTION` auf einer relevanten Rolle erhalten hat.

- Den Standardsuchpfad beibehalten und Privilegien zum Erstellen im Schema `public` erteilen. Alle Benutzer greifen implizit auf das Schema `public` zu. Das simuliert eine Situation, in der Schemata gar nicht verfügbar sind, und erleichtert den Übergang aus einer Welt ohne Schemata. Dieses Muster ist jedoch niemals sicher. Es ist nur akzeptabel, wenn die Datenbank einen einzelnen Benutzer oder wenige Benutzer hat, die einander gegenseitig vertrauen. In Datenbanken, die von PostgreSQL 14 oder früher aktualisiert wurden, ist dies die Voreinstellung.

Für jedes Muster gilt: Um gemeinsam genutzte Anwendungen zu installieren, etwa Tabellen, die von allen verwendet werden, oder zusätzliche Funktionen von Drittanbietern, legen Sie diese in eigene Schemata. Denken Sie daran, passende Privilegien zu erteilen, damit andere Benutzer darauf zugreifen können. Benutzer können diese zusätzlichen Objekte dann referenzieren, indem sie die Namen mit einem Schemanamen qualifizieren, oder sie können die zusätzlichen Schemata nach Bedarf in ihren Suchpfad aufnehmen.

### 5.10.7. Portabilität

Im SQL-Standard gibt es nicht die Vorstellung, dass Objekte im selben Schema unterschiedlichen Benutzern gehören. Außerdem erlauben einige Implementierungen nicht, Schemata zu erstellen, die einen anderen Namen haben als ihr Eigentümer. Tatsächlich sind die Konzepte Schema und Benutzer in einem Datenbanksystem, das nur die grundlegende Schemaunterstützung des Standards implementiert, nahezu gleichwertig. Deshalb betrachten viele Benutzer qualifizierte Namen eigentlich als `user_name.table_name`. PostgreSQL verhält sich effektiv so, wenn Sie für jeden Benutzer ein eigenes Schema erstellen.

Außerdem gibt es im SQL-Standard kein Konzept eines Schemas `public`. Für maximale Standardkonformität sollten Sie das Schema `public` nicht verwenden.

Einige SQL-Datenbanksysteme implementieren Schemata natürlich gar nicht oder bieten Namensraumunterstützung über einen gegebenenfalls eingeschränkten datenbankübergreifenden Zugriff. Wenn Sie mit solchen Systemen arbeiten müssen, erreichen Sie maximale Portabilität, indem Sie Schemata überhaupt nicht verwenden.

## 5.11. Vererbung

PostgreSQL implementiert Tabellenvererbung, die für Datenbankdesigner ein nützliches Werkzeug sein kann. SQL:1999 und spätere Versionen definieren eine Typvererbung, die sich in vieler Hinsicht von den hier beschriebenen Funktionen unterscheidet.

Beginnen wir mit einem Beispiel: Angenommen, wir wollen ein Datenmodell für Städte erstellen. Jeder Bundesstaat hat viele Städte, aber nur eine Hauptstadt. Wir möchten die Hauptstadt eines bestimmten Bundesstaats schnell abfragen können. Dazu könnten wir zwei Tabellen erstellen: eine für Hauptstädte und eine für Städte, die keine Hauptstädte sind. Was passiert aber, wenn wir Daten über eine Stadt abfragen möchten, unabhängig davon, ob sie eine Hauptstadt ist oder nicht? Die Vererbungsfunktion kann dieses Problem lösen. Wir definieren die Tabelle `capitals` so, dass sie von `cities` erbt:

```sql
CREATE TABLE cities (
    name            text,
    population      float,
    elevation       int                    -- in feet
);

CREATE TABLE capitals (
    state            char(2)
) INHERITS (cities);
```

In diesem Fall erbt die Tabelle `capitals` alle Spalten ihrer Elterntabelle `cities`. Hauptstädte haben zusätzlich die Spalte `state`, die den Bundesstaat angibt.

In PostgreSQL kann eine Tabelle von keiner, einer oder mehreren anderen Tabellen erben. Eine Abfrage kann entweder alle Zeilen einer Tabelle referenzieren oder alle Zeilen einer Tabelle einschließlich aller Zeilen ihrer abgeleiteten Tabellen. Letzteres ist das Standardverhalten. Die folgende Abfrage findet zum Beispiel die Namen aller Städte, einschließlich der Hauptstädte, die auf einer Höhe von mehr als 500 Fuß liegen:

```sql
SELECT name, elevation
FROM cities
WHERE elevation > 500;
```

Mit den Beispieldaten aus dem PostgreSQL-Tutorial (siehe [Abschnitt 2.1](02_Die_SQL_Sprache.md#21-einführung)) liefert dies:

```sql
   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
 Madison   |       845
```

Die folgende Abfrage findet dagegen alle Städte, die keine Hauptstädte sind und auf einer Höhe von mehr als 500 Fuß liegen:

```sql
SELECT name, elevation
    FROM ONLY cities
    WHERE elevation > 500;
```

```sql
   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
```

Das Schlüsselwort `ONLY` gibt hier an, dass die Abfrage nur auf `cities` angewendet werden soll, nicht auf Tabellen unterhalb von `cities` in der Vererbungshierarchie. Viele der bereits besprochenen Befehle, darunter `SELECT`, `UPDATE` und `DELETE`, unterstützen das Schlüsselwort `ONLY`.

Sie können den Tabellennamen auch mit einem nachgestellten `*` schreiben, um ausdrücklich anzugeben, dass abgeleitete Tabellen eingeschlossen werden:

```sql
SELECT name, elevation
    FROM cities*
    WHERE elevation > 500;
```

Das Schreiben von `*` ist nicht notwendig, da dieses Verhalten immer der Standard ist. Die Syntax wird jedoch weiterhin aus Kompatibilitätsgründen mit älteren Versionen unterstützt, in denen die Voreinstellung geändert werden konnte.

In manchen Fällen möchten Sie wissen, aus welcher Tabelle eine bestimmte Zeile stammt. Dafür gibt es in jeder Tabelle eine Systemspalte namens `tableoid`, die die Ursprungstabelle angibt:

```sql
SELECT c.tableoid, c.name, c.elevation
FROM cities c
WHERE c.elevation > 500;
```

Dies liefert:

```sql
 tableoid |   name    | elevation
----------+-----------+-----------
   139793 | Las Vegas |      2174
   139793 | Mariposa  |      1953
   139798 | Madison   |       845
```

Wenn Sie dieses Beispiel nachvollziehen, werden Sie wahrscheinlich andere numerische OIDs erhalten. Durch einen Join mit `pg_class` können Sie die tatsächlichen Tabellennamen sehen:

```sql
SELECT p.relname, c.name, c.elevation
FROM cities c, pg_class p
WHERE c.elevation > 500 AND c.tableoid = p.oid;
```

Dies liefert:

```sql
 relname  |   name    | elevation
----------+-----------+-----------
 cities   | Las Vegas |      2174
 cities   | Mariposa  |      1953
 capitals | Madison   |       845
```

Eine weitere Möglichkeit, denselben Effekt zu erzielen, besteht darin, den Alias-Typ `regclass` zu verwenden, der die Tabellen-OID symbolisch ausgibt:

```sql
SELECT c.tableoid::regclass, c.name, c.elevation
FROM cities c
WHERE c.elevation > 500;
```

Vererbung leitet Daten aus `INSERT`- oder `COPY`-Befehlen nicht automatisch an andere Tabellen in der Vererbungshierarchie weiter. In unserem Beispiel schlägt die folgende `INSERT`-Anweisung fehl:

```sql
INSERT INTO cities (name, population, elevation, state)
VALUES ('Albany', NULL, NULL, 'NY');
```

Man könnte erwarten, dass die Daten irgendwie an die Tabelle `capitals` weitergeleitet werden; das geschieht jedoch nicht. `INSERT` fügt immer genau in die angegebene Tabelle ein. In manchen Fällen ist es möglich, das Einfügen mithilfe einer Regel umzuleiten (siehe [Kapitel 39](39_Das_Regelsystem.md)). Im obigen Fall hilft das jedoch nicht, weil die Tabelle `cities` die Spalte `state` nicht enthält und der Befehl daher abgewiesen wird, bevor die Regel angewendet werden kann.

Alle Check-Constraints und Not-Null-Constraints einer Elterntabelle werden automatisch von ihren Kindtabellen geerbt, sofern dies nicht ausdrücklich mit `NO INHERIT`-Klauseln anders angegeben wird. Andere Constraint-Arten, also Unique-, Primärschlüssel- und Fremdschlüssel-Constraints, werden nicht geerbt.

Eine Tabelle kann von mehr als einer Elterntabelle erben. In diesem Fall besitzt sie die Vereinigung der in den Elterntabellen definierten Spalten. Spalten, die in der Definition der Kindtabelle deklariert sind, werden hinzugefügt. Wenn derselbe Spaltenname in mehreren Elterntabellen oder sowohl in einer Elterntabelle als auch in der Definition der Kindtabelle vorkommt, werden diese Spalten „zusammengeführt“, sodass es in der Kindtabelle nur eine solche Spalte gibt. Damit Spalten zusammengeführt werden können, müssen sie denselben Datentyp haben; andernfalls wird ein Fehler ausgelöst. Vererbbare Check-Constraints und Not-Null-Constraints werden auf ähnliche Weise zusammengeführt. Eine zusammengeführte Spalte wird beispielsweise als Not-Null markiert, wenn irgendeine der Spaltendefinitionen, aus denen sie stammt, als Not-Null markiert ist. Check-Constraints werden zusammengeführt, wenn sie denselben Namen haben; die Zusammenführung schlägt fehl, wenn ihre Bedingungen unterschiedlich sind.

Tabellenvererbung wird typischerweise beim Erstellen der Kindtabelle eingerichtet, indem die Klausel `INHERITS` der Anweisung `CREATE TABLE` verwendet wird. Alternativ kann einer bereits passend definierten Tabelle mit der Variante `INHERIT` von `ALTER TABLE` nachträglich eine neue Elternbeziehung hinzugefügt werden. Dazu muss die neue Kindtabelle bereits Spalten mit denselben Namen und Typen wie die Spalten der Elterntabelle enthalten. Außerdem muss sie Check-Constraints mit denselben Namen und Prüfausdrücken wie die Elterntabelle enthalten. Ebenso kann eine Vererbungsverknüpfung mit der Variante `NO INHERIT` von `ALTER TABLE` von einer Kindtabelle entfernt werden. Das dynamische Hinzufügen und Entfernen solcher Vererbungsverknüpfungen kann nützlich sein, wenn die Vererbungsbeziehung zur Tabellenpartitionierung verwendet wird (siehe [Abschnitt 5.12](05_Datendefinition.md#512-tabellenpartitionierung)).

Eine bequeme Möglichkeit, eine kompatible Tabelle zu erstellen, die später zu einer neuen Kindtabelle gemacht werden soll, ist die Klausel `LIKE` in `CREATE TABLE`. Sie erzeugt eine neue Tabelle mit denselben Spalten wie die Quelltabelle. Wenn auf der Quelltabelle `CHECK`-Constraints definiert sind, sollte die Option `INCLUDING CONSTRAINTS` zu `LIKE` angegeben werden, da die neue Kindtabelle zum Elternobjekt passende Constraints haben muss, um als kompatibel zu gelten.

Eine Elterntabelle kann nicht gelöscht werden, solange eine ihrer Kindtabellen vorhanden ist. Ebenso können Spalten oder Check-Constraints von Kindtabellen nicht gelöscht oder geändert werden, wenn sie von einer Elterntabelle geerbt wurden. Wenn Sie eine Tabelle und alle ihre abgeleiteten Tabellen entfernen möchten, besteht eine einfache Möglichkeit darin, die Elterntabelle mit der Option `CASCADE` zu löschen (siehe [Abschnitt 5.15](05_Datendefinition.md#515-abhängigkeitsverfolgung)).

`ALTER TABLE` gibt Änderungen an Spaltendefinitionen und Check-Constraints in der Vererbungshierarchie nach unten weiter. Auch hier ist das Löschen von Spalten, von denen andere Tabellen abhängen, nur mit der Option `CASCADE` möglich. `ALTER TABLE` folgt denselben Regeln für das Zusammenführen und Zurückweisen doppelter Spalten, die auch bei `CREATE TABLE` gelten.

Vererbte Abfragen prüfen Zugriffsrechte nur auf der Elterntabelle. Wird zum Beispiel das Privileg `UPDATE` auf der Tabelle `cities` erteilt, schließt das die Berechtigung ein, Zeilen in der Tabelle `capitals` zu aktualisieren, wenn diese über `cities` angesprochen werden. Dadurch bleibt der Eindruck erhalten, dass sich die Daten auch in der Elterntabelle befinden. Die Tabelle `capitals` könnte jedoch ohne zusätzliche Erteilung nicht direkt aktualisiert werden. Entsprechend werden die Row-Security-Policies der Elterntabelle (siehe [Abschnitt 5.9](05_Datendefinition.md#59-rowsecuritypolicies)) bei einer vererbten Abfrage auf Zeilen angewendet, die aus Kindtabellen stammen. Policies einer Kindtabelle werden, falls vorhanden, nur angewendet, wenn diese Tabelle ausdrücklich in der Abfrage genannt wird; in diesem Fall werden Policies ihrer Elterntabellen ignoriert.

Fremdtabellen (siehe [Abschnitt 5.13](05_Datendefinition.md#513-fremddaten)) können ebenfalls Teil von Vererbungshierarchien sein, als Eltern- oder Kindtabellen, genau wie reguläre Tabellen. Wenn eine Fremdtabelle Teil einer Vererbungshierarchie ist, werden Operationen, die von der Fremdtabelle nicht unterstützt werden, auch für die gesamte Hierarchie nicht unterstützt.

### 5.11.1. Einschränkungen

Beachten Sie, dass nicht alle SQL-Befehle mit Vererbungshierarchien arbeiten können. Befehle zur Datenabfrage, Datenänderung oder Schemaänderung, zum Beispiel `SELECT`, `UPDATE`, `DELETE` und die meisten Varianten von `ALTER TABLE`, aber nicht `INSERT` oder `ALTER TABLE ... RENAME`, schließen Kindtabellen typischerweise standardmäßig ein und unterstützen die Schreibweise `ONLY`, um sie auszuschließen. Die meisten Befehle zur Datenbankwartung und -optimierung, zum Beispiel `REINDEX`, arbeiten nur auf einzelnen physischen Tabellen und unterstützen keine Rekursion über Vererbungshierarchien. Die Befehle `VACUUM` und `ANALYZE` schließen Kindtabellen jedoch standardmäßig ein; die Schreibweise `ONLY` wird unterstützt, um sie auszuschließen. Das jeweilige Verhalten der einzelnen Befehle ist auf ihrer Referenzseite dokumentiert (SQL-Befehle).

Eine ernsthafte Einschränkung der Vererbungsfunktion besteht darin, dass Indizes, einschließlich Unique Constraints, und Fremdschlüssel-Constraints nur auf einzelne Tabellen angewendet werden, nicht auf deren Vererbungskinder. Das gilt sowohl für die referenzierende als auch für die referenzierte Seite eines Fremdschlüssel-Constraints. Bezogen auf das obige Beispiel bedeutet das:

- Wenn wir `cities.name` als `UNIQUE` oder als Primärschlüssel deklarieren, würde das die Tabelle `capitals` nicht daran hindern, Zeilen mit Namen zu enthalten, die Zeilen in `cities` duplizieren. Diese doppelten Zeilen würden standardmäßig in Abfragen von `cities` erscheinen. Tatsächlich hätte `capitals` standardmäßig überhaupt keinen Unique Constraint und könnte daher mehrere Zeilen mit demselben Namen enthalten. Sie könnten `capitals` einen Unique Constraint hinzufügen, aber das würde Duplikate gegenüber `cities` nicht verhindern.
- Ebenso würde ein Constraint der Form `cities.name REFERENCES` auf eine andere Tabelle nicht automatisch an `capitals` weitergegeben. In diesem Fall könnten Sie das umgehen, indem Sie denselben `REFERENCES`-Constraint manuell zu `capitals` hinzufügen.
- Wenn die Spalte einer anderen Tabelle `REFERENCES cities(name)` angibt, dürfte diese andere Tabelle Städtenamen enthalten, aber keine Hauptstadtnamen. Für diesen Fall gibt es keine gute Umgehung.

Einige Funktionen, die für Vererbungshierarchien nicht implementiert sind, sind für deklarative Partitionierung implementiert. Es ist daher erhebliche Sorgfalt nötig, wenn Sie entscheiden, ob Partitionierung mit klassischer Vererbung für Ihre Anwendung sinnvoll ist.

## 5.12. Tabellenpartitionierung

PostgreSQL unterstützt grundlegende Tabellenpartitionierung. Dieser Abschnitt beschreibt, warum und wie Partitionierung als Teil des Datenbankdesigns eingesetzt werden kann.

### 5.12.1. Überblick

Partitionierung bedeutet, eine logisch große Tabelle in kleinere physische Teile aufzuteilen. Das kann mehrere Vorteile haben:

- Die Abfrageleistung kann sich in bestimmten Situationen deutlich verbessern, besonders wenn die meisten stark genutzten Zeilen der Tabelle in einer einzelnen Partition oder in wenigen Partitionen liegen. Partitionierung ersetzt gewissermaßen die oberen Ebenen von Indexbäumen, sodass die stark genutzten Teile der Indizes eher im Speicher liegen.
- Wenn Abfragen oder Aktualisierungen einen großen Anteil einer einzelnen Partition betreffen, kann ein sequenzieller Scan dieser Partition schneller sein als die Verwendung eines Indexes, der zufällige Lesezugriffe über die gesamte Tabelle hinweg erfordern würde.
- Massenlade- und Massenlöschvorgänge können durch Hinzufügen oder Entfernen von Partitionen erledigt werden, wenn das Nutzungsmuster beim Partitionsdesign berücksichtigt wurde. Eine einzelne Partition mit `DROP TABLE` zu löschen oder mit `ALTER TABLE DETACH PARTITION` abzutrennen, ist deutlich schneller als eine Massenoperation. Außerdem vermeiden diese Befehle den `VACUUM`-Overhead, der durch ein großes `DELETE` entsteht.
- Selten genutzte Daten können auf günstigere und langsamere Speichermedien verschoben werden.

Diese Vorteile lohnen sich normalerweise nur, wenn eine Tabelle andernfalls sehr groß wäre. Der genaue Punkt, ab dem eine Tabelle von Partitionierung profitiert, hängt von der Anwendung ab; als Faustregel gilt jedoch, dass die Tabellengröße den physischen Speicher des Datenbankservers übersteigen sollte.

PostgreSQL bietet eingebaute Unterstützung für folgende Formen der Partitionierung:

**Range-Partitionierung**

Die Tabelle wird in Bereiche partitioniert, die durch eine Schlüsselspalte oder eine Menge von Spalten definiert werden. Die Wertebereiche verschiedener Partitionen überlappen sich nicht. Man kann zum Beispiel nach Datumsbereichen oder nach Bereichen von Kennungen bestimmter Geschäftsobjekte partitionieren. Bereichsgrenzen gelten unten als inklusive und oben als exklusiv. Wenn der Bereich einer Partition von 1 bis 10 reicht und der Bereich der nächsten von 10 bis 20, gehört der Wert 10 also zur zweiten Partition, nicht zur ersten.

**List-Partitionierung**

Die Tabelle wird partitioniert, indem ausdrücklich aufgelistet wird, welche Schlüsselwerte in welcher Partition erscheinen.

**Hash-Partitionierung**

Die Tabelle wird partitioniert, indem für jede Partition ein Modulus und ein Rest angegeben werden. Jede Partition enthält die Zeilen, deren Hashwert des Partitionsschlüssels, geteilt durch den angegebenen Modulus, den angegebenen Rest ergibt.

Wenn Ihre Anwendung andere Formen der Partitionierung benötigt, können stattdessen alternative Methoden wie Vererbung und `UNION ALL`-Views verwendet werden. Diese Methoden bieten Flexibilität, haben aber nicht alle Leistungsvorteile der eingebauten deklarativen Partitionierung.

### 5.12.2. Deklarative Partitionierung

PostgreSQL erlaubt es, eine Tabelle als in Partitionen aufgeteilt zu deklarieren. Die aufgeteilte Tabelle heißt partitionierte Tabelle. Die Deklaration enthält die oben beschriebene Partitionierungsmethode sowie eine Liste von Spalten oder Ausdrücken, die als Partitionsschlüssel verwendet werden.

Die partitionierte Tabelle selbst ist eine „virtuelle“ Tabelle ohne eigenen Speicher. Der Speicher gehört stattdessen den Partitionen, die ansonsten gewöhnliche Tabellen sind, die der partitionierten Tabelle zugeordnet sind. Jede Partition speichert eine Teilmenge der Daten entsprechend ihren Partitionsgrenzen. Alle Zeilen, die in eine partitionierte Tabelle eingefügt werden, werden anhand der Werte der Partitionsschlüsselspalten an die passende Partition weitergeleitet. Wird der Partitionsschlüssel einer Zeile aktualisiert, wird die Zeile in eine andere Partition verschoben, wenn sie die Grenzen ihrer ursprünglichen Partition nicht mehr erfüllt.

Partitionen können selbst als partitionierte Tabellen definiert werden, wodurch Unterpartitionierung entsteht. Obwohl alle Partitionen dieselben Spalten wie ihre partitionierte Elterntabelle haben müssen, können Partitionen eigene Indizes, Constraints und Vorgabewerte besitzen, die sich von denen anderer Partitionen unterscheiden. Weitere Details zum Erstellen partitionierter Tabellen und Partitionen finden Sie unter `CREATE TABLE`.

Es ist nicht möglich, eine reguläre Tabelle in eine partitionierte Tabelle umzuwandeln oder umgekehrt. Es ist jedoch möglich, eine vorhandene reguläre oder partitionierte Tabelle als Partition einer partitionierten Tabelle hinzuzufügen oder eine Partition aus einer partitionierten Tabelle zu entfernen und dadurch in eine eigenständige Tabelle umzuwandeln. Das kann viele Wartungsprozesse vereinfachen und beschleunigen. Weitere Informationen zu den Unterbefehlen `ATTACH PARTITION` und `DETACH PARTITION` finden Sie unter `ALTER TABLE`.

Partitionen können auch Fremdtabellen sein. Dabei ist allerdings erhebliche Sorgfalt nötig, denn dann ist der Benutzer dafür verantwortlich, dass der Inhalt der Fremdtabelle die Partitionierungsregel erfüllt. Es gibt außerdem weitere Einschränkungen. Weitere Informationen finden Sie unter `CREATE FOREIGN TABLE`.

#### 5.12.2.1. Beispiel

Angenommen, wir bauen eine Datenbank für ein großes Eiscremeunternehmen. Das Unternehmen misst täglich Höchsttemperaturen und Eisverkäufe in jeder Region. Konzeptionell möchten wir eine Tabelle wie diese:

```sql
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
);
```

Wir wissen, dass die meisten Abfragen nur auf Daten der letzten Woche, des letzten Monats oder des letzten Quartals zugreifen, da die Tabelle vor allem für Online-Berichte an das Management verwendet wird. Um weniger alte Daten speichern zu müssen, entscheiden wir uns, nur die Daten der letzten drei Jahre aufzubewahren. Zu Beginn jedes Monats entfernen wir die Daten des ältesten Monats. In dieser Situation kann Partitionierung helfen, die verschiedenen Anforderungen an die Tabelle `measurement` zu erfüllen.

Um hier deklarative Partitionierung zu verwenden, gehen Sie wie folgt vor:

1. Erstellen Sie die Tabelle `measurement` als partitionierte Tabelle, indem Sie die Klausel `PARTITION BY` angeben. Diese enthält die Partitionierungsmethode, hier `RANGE`, und die Spalten, die als Partitionsschlüssel dienen.

```sql
CREATE TABLE measurement (
      city_id                   int not null,
      logdate                   date not null,
      peaktemp                  int,
      unitsales                 int
) PARTITION BY RANGE (logdate);
```

2. Erstellen Sie Partitionen. Jede Partitionsdefinition muss Grenzen angeben, die zur Partitionierungsmethode und zum Partitionsschlüssel der Elterntabelle passen. Grenzen, durch die sich Werte der neuen Partition mit Werten einer oder mehrerer vorhandener Partitionen überschneiden würden, lösen einen Fehler aus.

Die so erstellten Partitionen sind in jeder Hinsicht normale PostgreSQL-Tabellen oder möglicherweise Fremdtabellen. Für jede Partition können Tablespace und Speicherparameter getrennt angegeben werden.

In unserem Beispiel soll jede Partition Daten für einen Monat enthalten, damit jeweils ein Monatsbestand gelöscht werden kann. Die Befehle könnten so aussehen:

```sql
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');

CREATE TABLE measurement_y2006m03 PARTITION OF measurement
    FOR VALUES FROM ('2006-03-01') TO ('2006-04-01');

...
CREATE TABLE measurement_y2007m11 PARTITION OF measurement
    FOR VALUES FROM ('2007-11-01') TO ('2007-12-01');

CREATE TABLE measurement_y2007m12 PARTITION OF measurement
    FOR VALUES FROM ('2007-12-01') TO ('2008-01-01')
    TABLESPACE fasttablespace;

CREATE TABLE measurement_y2008m01 PARTITION OF measurement
    FOR VALUES FROM ('2008-01-01') TO ('2008-02-01')
    WITH (parallel_workers = 4)
    TABLESPACE fasttablespace;
```

Benachbarte Partitionen können denselben Grenzwert teilen, weil obere Grenzen von Bereichen als exklusiv behandelt werden.

Wenn Sie Unterpartitionierung implementieren möchten, geben Sie in den Befehlen zum Erstellen einzelner Partitionen erneut die Klausel `PARTITION BY` an, zum Beispiel:

```sql
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01')
    PARTITION BY RANGE (peaktemp);
```

Nachdem Partitionen von `measurement_y2006m02` erstellt wurden, werden Daten, die in `measurement` eingefügt und `measurement_y2006m02` zugeordnet werden, anhand der Spalte `peaktemp` weiter an eine ihrer Partitionen geleitet. Dasselbe gilt für Daten, die direkt in `measurement_y2006m02` eingefügt werden, sofern deren Partitions-Constraint erfüllt ist. Der angegebene Partitionsschlüssel darf sich mit dem Partitionsschlüssel der Elterntabelle überschneiden. Beim Festlegen der Grenzen einer Unterpartition muss jedoch darauf geachtet werden, dass die von ihr akzeptierten Daten eine Teilmenge dessen bilden, was die eigenen Grenzen der Partition erlauben; das System versucht nicht zu prüfen, ob das tatsächlich der Fall ist.

Wenn Daten in die Elterntabelle eingefügt werden, die keiner vorhandenen Partition zugeordnet werden können, entsteht ein Fehler; eine passende Partition muss manuell hinzugefügt werden.

Es ist nicht nötig, Tabellen-Constraints manuell zu erstellen, die die Partitionsgrenzen beschreiben. Solche Constraints werden automatisch erzeugt.

3. Erstellen Sie auf der partitionierten Tabelle einen Index auf den Schlüsselspalten sowie alle weiteren gewünschten Indizes. Der Schlüsselindex ist nicht zwingend notwendig, aber in den meisten Szenarien hilfreich. Dadurch wird automatisch ein passender Index auf jeder Partition erstellt; auch Partitionen, die Sie später erstellen oder anhängen, erhalten einen solchen Index. Ein auf einer partitionierten Tabelle deklarierter Index oder Unique Constraint ist genauso „virtuell“ wie die partitionierte Tabelle selbst: Die eigentlichen Daten liegen in Kindindizes auf den einzelnen Partitionstabellen.

```sql
CREATE INDEX ON measurement (logdate);
```

4. Stellen Sie sicher, dass der Konfigurationsparameter `enable_partition_pruning` in `postgresql.conf` nicht deaktiviert ist. Andernfalls werden Abfragen nicht wie gewünscht optimiert.

Im obigen Beispiel würden wir jeden Monat eine neue Partition erstellen; daher kann es sinnvoll sein, ein Skript zu schreiben, das die nötigen DDL-Befehle automatisch erzeugt.

#### 5.12.2.2. Partitionspflege

Normalerweise soll die beim ersten Definieren der Tabelle eingerichtete Menge von Partitionen nicht statisch bleiben. Häufig sollen Partitionen mit alten Daten entfernt und regelmäßig neue Partitionen für neue Daten hinzugefügt werden. Einer der wichtigsten Vorteile der Partitionierung besteht gerade darin, dass diese sonst mühsame Aufgabe durch Manipulation der Partitionsstruktur nahezu sofort ausgeführt werden kann, statt große Datenmengen physisch zu verschieben.

Die einfachste Option zum Entfernen alter Daten ist das Löschen der nicht mehr benötigten Partition:

```sql
DROP TABLE measurement_y2006m02;
```

Damit lassen sich Millionen von Datensätzen sehr schnell löschen, weil nicht jeder Datensatz einzeln entfernt werden muss. Beachten Sie jedoch, dass der obige Befehl einen `ACCESS EXCLUSIVE`-Lock auf der Elterntabelle benötigt.

Eine oft vorzuziehende Alternative ist, die Partition aus der partitionierten Tabelle zu entfernen, den Zugriff auf sie als eigenständige Tabelle aber zu behalten. Dafür gibt es zwei Formen:

```sql
ALTER TABLE measurement DETACH PARTITION measurement_y2006m02;
ALTER TABLE measurement DETACH PARTITION measurement_y2006m02
 CONCURRENTLY;
```

So können weitere Operationen auf den Daten ausgeführt werden, bevor sie gelöscht werden. Beispielsweise ist dies oft ein guter Zeitpunkt, die Daten mit `COPY`, `pg_dump` oder ähnlichen Werkzeugen zu sichern. Auch Aggregation in kleinere Formate, weitere Datenmanipulationen oder Berichte können hier sinnvoll sein. Die erste Befehlsform benötigt einen `ACCESS EXCLUSIVE`-Lock auf der Elterntabelle. Der Zusatz `CONCURRENTLY` in der zweiten Form erlaubt es, dass die Abtrennung nur einen `SHARE UPDATE EXCLUSIVE`-Lock auf der Elterntabelle benötigt; Einzelheiten zu den Einschränkungen finden Sie unter `ALTER TABLE ... DETACH PARTITION`.

Ebenso können wir eine neue Partition für neue Daten hinzufügen. Eine leere Partition in der partitionierten Tabelle wird genauso erstellt wie die ursprünglichen Partitionen:

```sql
CREATE TABLE measurement_y2008m02 PARTITION OF measurement
    FOR VALUES FROM ('2008-02-01') TO ('2008-03-01')
    TABLESPACE fasttablespace;
```

Alternativ ist es manchmal praktischer, eine neue Tabelle außerhalb der Partitionsstruktur zu erstellen und sie später als Partition anzuhängen. Dadurch können neue Daten geladen, geprüft und transformiert werden, bevor sie in der partitionierten Tabelle erscheinen. Außerdem benötigt `ATTACH PARTITION` nur einen `SHARE UPDATE EXCLUSIVE`-Lock auf der partitionierten Tabelle, nicht den `ACCESS EXCLUSIVE`-Lock, den `CREATE TABLE ... PARTITION OF` benötigt. Das ist freundlicher gegenüber gleichzeitigen Operationen auf der partitionierten Tabelle; weitere Einzelheiten finden Sie unter `ALTER TABLE ... ATTACH PARTITION`. Die Option `CREATE TABLE ... LIKE` kann helfen, die Definition der Elterntabelle nicht mühsam wiederholen zu müssen:

```sql
CREATE TABLE measurement_y2008m02
  (LIKE measurement INCLUDING DEFAULTS INCLUDING CONSTRAINTS)
  TABLESPACE fasttablespace;

ALTER TABLE measurement_y2008m02 ADD CONSTRAINT y2008m02
   CHECK (logdate >= DATE '2008-02-01' AND logdate < DATE '2008-03-01');

\copy measurement_y2008m02 from 'measurement_y2008m02'
-- möglicherweise weitere Datenvorbereitung

ALTER TABLE measurement ATTACH PARTITION measurement_y2008m02
    FOR VALUES FROM ('2008-02-01') TO ('2008-03-01');
```

Beim Ausführen von `ATTACH PARTITION` wird die Tabelle gescannt, um den Partitions-Constraint zu validieren, während auf dieser Partition ein `ACCESS EXCLUSIVE`-Lock gehalten wird. Wie oben gezeigt, empfiehlt es sich, diesen Scan zu vermeiden, indem vor dem Anhängen ein `CHECK`-Constraint erstellt wird, der dem erwarteten Partitions-Constraint entspricht. Sobald `ATTACH PARTITION` abgeschlossen ist, sollte der nun redundante `CHECK`-Constraint entfernt werden. Wenn die anzuhängende Tabelle selbst eine partitionierte Tabelle ist, werden ihre Unterpartitionen rekursiv gesperrt und gescannt, bis entweder ein passender `CHECK`-Constraint gefunden wird oder die Blattpartitionen erreicht sind.

Wenn die partitionierte Tabelle eine `DEFAULT`-Partition hat, empfiehlt es sich ebenso, einen `CHECK`-Constraint zu erstellen, der den Constraint der anzuhängenden Partition ausschließt. Geschieht das nicht, wird die `DEFAULT`-Partition gescannt, um zu prüfen, dass sie keine Datensätze enthält, die in die anzuhängende Partition gehören. Diese Operation wird ausgeführt, während auf der `DEFAULT`-Partition ein `ACCESS EXCLUSIVE`-Lock gehalten wird. Ist die `DEFAULT`-Partition selbst partitioniert, werden ihre Partitionen auf dieselbe Weise rekursiv geprüft.

Wie bereits erwähnt, können auf partitionierten Tabellen Indizes erstellt werden, sodass sie automatisch auf die gesamte Hierarchie angewendet werden. Das ist sehr bequem, weil sowohl alle bestehenden als auch alle zukünftigen Partitionen indexiert werden. Eine Einschränkung beim Erstellen neuer Indizes auf partitionierten Tabellen ist jedoch, dass der Zusatz `CONCURRENTLY` nicht verwendet werden kann, was zu langen Sperrzeiten führen kann. Um das zu vermeiden, kann `CREATE INDEX ON ONLY` auf der partitionierten Tabelle verwendet werden. Dadurch wird der neue Index als ungültig markiert erstellt, was die automatische Anwendung auf bestehende Partitionen verhindert. Stattdessen können Indizes auf jeder Partition einzeln mit `CONCURRENTLY` erstellt und anschließend mit `ALTER INDEX ... ATTACH PARTITION` an den partitionierten Index der Elterntabelle angehängt werden. Sobald Indizes für alle Partitionen angehängt sind, wird der Elternindex automatisch als gültig markiert. Beispiel:

```sql
CREATE INDEX measurement_usls_idx ON ONLY measurement (unitsales);

CREATE INDEX CONCURRENTLY measurement_usls_200602_idx
    ON measurement_y2006m02 (unitsales);
ALTER INDEX measurement_usls_idx
    ATTACH PARTITION measurement_usls_200602_idx;
...
```

Diese Technik kann auch mit `UNIQUE`- und `PRIMARY KEY`-Constraints verwendet werden; die Indizes werden beim Erstellen des Constraints implizit erzeugt. Beispiel:

```sql
ALTER TABLE ONLY measurement ADD UNIQUE (city_id, logdate);

ALTER TABLE measurement_y2006m02 ADD UNIQUE (city_id, logdate);
ALTER INDEX measurement_city_id_logdate_key
    ATTACH PARTITION measurement_y2006m02_city_id_logdate_key;
...
```

#### 5.12.2.3. Einschränkungen

Für partitionierte Tabellen gelten die folgenden Einschränkungen:

- Um einen Unique Constraint oder Primärschlüssel-Constraint auf einer partitionierten Tabelle zu erstellen, dürfen die Partitionsschlüssel keine Ausdrücke oder Funktionsaufrufe enthalten, und die Constraint-Spalten müssen alle Partitionsschlüsselspalten enthalten. Diese Einschränkung existiert, weil die einzelnen Indizes, aus denen der Constraint besteht, Eindeutigkeit nur innerhalb ihrer jeweiligen Partition direkt erzwingen können. Die Partitionsstruktur selbst muss daher garantieren, dass es keine Duplikate in unterschiedlichen Partitionen gibt.
- Ebenso muss ein Exclusion-Constraint alle Partitionsschlüsselspalten enthalten. Außerdem muss der Constraint diese Spalten auf Gleichheit vergleichen, nicht zum Beispiel mit `&&`. Auch diese Einschränkung entsteht daraus, dass partitionsübergreifende Einschränkungen nicht erzwungen werden können. Der Constraint darf zusätzliche Spalten enthalten, die nicht Teil des Partitionsschlüssels sind, und diese mit beliebigen Operatoren vergleichen.
- `BEFORE ROW`-Trigger auf `INSERT` können nicht ändern, welche Partition das endgültige Ziel für eine neue Zeile ist.
- Temporäre und permanente Relationen dürfen nicht im selben Partitionsbaum gemischt werden. Ist die partitionierte Tabelle permanent, müssen auch ihre Partitionen permanent sein; ist sie temporär, müssen auch ihre Partitionen temporär sein. Bei temporären Relationen müssen alle Mitglieder des Partitionsbaums aus derselben Sitzung stammen.

Einzelne Partitionen werden im Hintergrund per Vererbung mit ihrer partitionierten Tabelle verbunden. Dennoch können nicht alle allgemeinen Vererbungsfunktionen mit deklarativ partitionierten Tabellen oder ihren Partitionen verwendet werden. Insbesondere kann eine Partition keine anderen Eltern haben als die partitionierte Tabelle, deren Partition sie ist, und eine Tabelle kann nicht zugleich von einer partitionierten Tabelle und einer regulären Tabelle erben. Partitionierte Tabellen und ihre Partitionen teilen daher nie eine Vererbungshierarchie mit regulären Tabellen.

Da eine Partitionshierarchie aus partitionierter Tabelle und Partitionen weiterhin eine Vererbungshierarchie ist, gelten `tableoid` und alle normalen Vererbungsregeln wie in [Abschnitt 5.11](05_Datendefinition.md#511-vererbung) beschrieben, mit einigen Ausnahmen:

- Partitionen können keine Spalten besitzen, die in der Elterntabelle nicht vorhanden sind. Beim Erstellen von Partitionen mit `CREATE TABLE` können keine Spalten angegeben werden, und es ist auch nicht möglich, Partitionen nachträglich mit `ALTER TABLE` Spalten hinzuzufügen. Tabellen können mit `ALTER TABLE ... ATTACH PARTITION` nur dann als Partition hinzugefügt werden, wenn ihre Spalten exakt zur Elterntabelle passen.
- Sowohl `CHECK`- als auch `NOT NULL`-Constraints einer partitionierten Tabelle werden immer von allen Partitionen geerbt; es ist nicht erlaubt, `NO INHERIT`-Constraints dieser Typen zu erstellen. Ein Constraint dieser Typen kann nicht gelöscht werden, wenn derselbe Constraint in der Elterntabelle vorhanden ist.
- `ONLY` zum Hinzufügen oder Entfernen eines Constraints nur auf der partitionierten Tabelle wird unterstützt, solange keine Partitionen vorhanden sind. Sobald Partitionen existieren, führt `ONLY` bei allen Constraints außer `UNIQUE` und `PRIMARY KEY` zu einem Fehler. Stattdessen können Constraints auf den Partitionen selbst hinzugefügt und, sofern sie nicht in der Elterntabelle vorhanden sind, gelöscht werden.
- Da eine partitionierte Tabelle selbst keine Daten besitzt, führen Versuche, `TRUNCATE ONLY` auf einer partitionierten Tabelle zu verwenden, immer zu einem Fehler.

### 5.12.3. Partitionierung mit Vererbung

Obwohl die eingebaute deklarative Partitionierung für die meisten häufigen Anwendungsfälle geeignet ist, gibt es Situationen, in denen ein flexiblerer Ansatz nützlich sein kann. Partitionierung kann mit Tabellenvererbung implementiert werden, was mehrere Funktionen ermöglicht, die deklarative Partitionierung nicht unterstützt:

- Bei deklarativer Partitionierung müssen Partitionen exakt dieselbe Spaltenmenge wie die partitionierte Tabelle haben; bei Tabellenvererbung dürfen Kindtabellen zusätzliche Spalten besitzen, die in der Elterntabelle nicht vorhanden sind.
- Tabellenvererbung erlaubt Mehrfachvererbung.
- Deklarative Partitionierung unterstützt nur Range-, List- und Hash-Partitionierung, während Tabellenvererbung Daten auf eine vom Benutzer frei gewählte Weise aufteilen kann. Wenn Constraint Exclusion Kindtabellen nicht effektiv ausschließen kann, kann die Abfrageleistung allerdings schlecht sein.

#### 5.12.3.1. Beispiel

Dieses Beispiel baut eine Partitionsstruktur auf, die dem obigen Beispiel für deklarative Partitionierung entspricht. Gehen Sie wie folgt vor:

1. Erstellen Sie die „Wurzel“-Tabelle, von der alle „Kind“-Tabellen erben. Diese Tabelle enthält keine Daten. Definieren Sie auf ihr keine Check-Constraints, sofern diese nicht für alle Kindtabellen gleichermaßen gelten sollen. Auch Indizes oder Unique Constraints darauf sind nicht sinnvoll. In unserem Beispiel ist die Wurzeltabelle die ursprünglich definierte Tabelle `measurement`:

```sql
CREATE TABLE measurement (
    city_id                 int not null,
    logdate                 date not null,
    peaktemp                int,
    unitsales               int
);
```

2. Erstellen Sie mehrere Kindtabellen, die jeweils von der Wurzeltabelle erben. Normalerweise fügen diese Tabellen der von der Wurzel geerbten Spaltenmenge keine weiteren Spalten hinzu. Wie bei deklarativer Partitionierung sind diese Tabellen in jeder Hinsicht normale PostgreSQL-Tabellen oder Fremdtabellen.

```sql
CREATE TABLE measurement_y2006m02 () INHERITS (measurement);
CREATE TABLE measurement_y2006m03 () INHERITS (measurement);
...
CREATE TABLE measurement_y2007m11 () INHERITS (measurement);
CREATE TABLE measurement_y2007m12 () INHERITS (measurement);
CREATE TABLE measurement_y2008m01 () INHERITS (measurement);
```

3. Fügen Sie den Kindtabellen sich nicht überschneidende Tabellen-Constraints hinzu, um die jeweils erlaubten Schlüsselwerte zu definieren.

Typische Beispiele wären:

```sql
CHECK (x = 1)
CHECK (county IN ('Oxfordshire', 'Buckinghamshire', 'Warwickshire'))
CHECK (outletID >= 100 AND outletID < 200)
```

Stellen Sie sicher, dass die Constraints garantieren, dass es keine Überschneidung zwischen den in verschiedenen Kindtabellen erlaubten Schlüsselwerten gibt. Ein häufiger Fehler ist, Bereichs-Constraints so einzurichten:

```sql
CHECK (outletID BETWEEN 100 AND 200)
CHECK (outletID BETWEEN 200 AND 300)
```

Das ist falsch, weil nicht klar ist, zu welcher Kindtabelle der Schlüsselwert 200 gehört. Bereiche sollten stattdessen in diesem Stil definiert werden:

```sql
CREATE TABLE measurement_y2006m02 (
    CHECK (logdate >= DATE '2006-02-01' AND logdate < DATE '2006-03-01')
) INHERITS (measurement);

CREATE TABLE measurement_y2006m03 (
    CHECK (logdate >= DATE '2006-03-01' AND logdate < DATE '2006-04-01')
) INHERITS (measurement);

...
CREATE TABLE measurement_y2007m11 (
    CHECK (logdate >= DATE '2007-11-01' AND logdate < DATE '2007-12-01')
) INHERITS (measurement);

CREATE TABLE measurement_y2007m12 (
    CHECK (logdate >= DATE '2007-12-01' AND logdate < DATE '2008-01-01')
) INHERITS (measurement);

CREATE TABLE measurement_y2008m01 (
    CHECK (logdate >= DATE '2008-01-01' AND logdate < DATE '2008-02-01')
) INHERITS (measurement);
```

4. Erstellen Sie für jede Kindtabelle einen Index auf den Schlüsselspalten sowie alle weiteren gewünschten Indizes.

```sql
CREATE INDEX measurement_y2006m02_logdate ON
  measurement_y2006m02 (logdate);
CREATE INDEX measurement_y2006m03_logdate ON
  measurement_y2006m03 (logdate);
CREATE INDEX measurement_y2007m11_logdate ON
  measurement_y2007m11 (logdate);
CREATE INDEX measurement_y2007m12_logdate ON
  measurement_y2007m12 (logdate);
CREATE INDEX measurement_y2008m01_logdate ON
  measurement_y2008m01 (logdate);
```

5. Unsere Anwendung soll `INSERT INTO measurement ...` ausführen können, während die Daten in die passende Kindtabelle umgeleitet werden. Das erreichen wir, indem wir eine geeignete Triggerfunktion an die Wurzeltabelle hängen. Wenn Daten nur in die neueste Kindtabelle eingefügt werden, reicht eine sehr einfache Triggerfunktion:

```sql
CREATE OR REPLACE FUNCTION measurement_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
     INSERT INTO measurement_y2008m01 VALUES (NEW.*);
     RETURN NULL;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER insert_measurement_trigger
    BEFORE INSERT ON measurement
    FOR EACH ROW EXECUTE FUNCTION measurement_insert_trigger();
```

Wir müssen die Triggerfunktion jeden Monat neu definieren, damit sie immer in die aktuelle Kindtabelle einfügt. Die Triggerdefinition selbst muss jedoch nicht aktualisiert werden.

Vielleicht möchten wir Daten einfügen und den Server automatisch die Kindtabelle finden lassen, in die die Zeile eingefügt werden soll. Das lässt sich mit einer komplexeren Triggerfunktion erreichen:

```sql
CREATE OR REPLACE FUNCTION measurement_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF (NEW.logdate >= DATE '2006-02-01' AND
        NEW.logdate < DATE '2006-03-01') THEN
        INSERT INTO measurement_y2006m02 VALUES (NEW.*);
    ELSIF (NEW.logdate >= DATE '2006-03-01' AND
           NEW.logdate < DATE '2006-04-01') THEN
        INSERT INTO measurement_y2006m03 VALUES (NEW.*);
    ...
    ELSIF (NEW.logdate >= DATE '2008-01-01' AND
           NEW.logdate < DATE '2008-02-01') THEN
        INSERT INTO measurement_y2008m01 VALUES (NEW.*);
    ELSE
        RAISE EXCEPTION 'Date out of range. Fix the measurement_insert_trigger() function!';
    END IF;
    RETURN NULL;
END;
$$
LANGUAGE plpgsql;
```

Die Triggerdefinition ist dieselbe wie zuvor. Beachten Sie, dass jeder `IF`-Test exakt zum `CHECK`-Constraint seiner Kindtabelle passen muss.

Obwohl diese Funktion komplexer ist als der Fall mit nur einem Monat, muss sie nicht so häufig aktualisiert werden, weil Zweige im Voraus hinzugefügt werden können.

> **Hinweis:** In der Praxis ist es oft am besten, zuerst die neueste Kindtabelle zu prüfen, wenn die meisten Einfügungen in diese Kindtabelle gehen. Der Einfachheit halber wurden die Triggerprüfungen hier in derselben Reihenfolge gezeigt wie in den anderen Teilen dieses Beispiels.

Ein anderer Ansatz zur Umleitung von Einfügungen in die passende Kindtabelle besteht darin, auf der Wurzeltabelle Regeln statt eines Triggers einzurichten:

```sql
CREATE RULE measurement_insert_y2006m02 AS
ON INSERT TO measurement WHERE
    (logdate >= DATE '2006-02-01' AND logdate < DATE '2006-03-01')
DO INSTEAD
    INSERT INTO measurement_y2006m02 VALUES (NEW.*);
...
CREATE RULE measurement_insert_y2008m01 AS
ON INSERT TO measurement WHERE
    (logdate >= DATE '2008-01-01' AND logdate < DATE '2008-02-01')
DO INSTEAD
    INSERT INTO measurement_y2008m01 VALUES (NEW.*);
```

Eine Regel hat deutlich mehr Overhead als ein Trigger, aber dieser Overhead fällt einmal pro Abfrage an und nicht einmal pro Zeile. Für Masseneinfügungen kann diese Methode daher vorteilhaft sein. In den meisten Fällen bietet die Trigger-Methode jedoch bessere Leistung.

Beachten Sie, dass `COPY` Regeln ignoriert. Wenn Sie `COPY` zum Einfügen von Daten verwenden möchten, müssen Sie in die richtige Kindtabelle kopieren, statt direkt in die Wurzel zu kopieren. `COPY` löst Trigger aus, sodass Sie es normal verwenden können, wenn Sie den Trigger-Ansatz nutzen.

Ein weiterer Nachteil des Regelansatzes ist, dass es keinen einfachen Weg gibt, einen Fehler zu erzwingen, wenn die Regeln das Einfügedatum nicht abdecken; die Daten landen dann stillschweigend in der Wurzeltabelle.

6. Stellen Sie sicher, dass der Konfigurationsparameter `constraint_exclusion` in `postgresql.conf` nicht deaktiviert ist; andernfalls werden Kindtabellen möglicherweise unnötig durchsucht.

Wie zu sehen ist, kann eine komplexe Tabellenhierarchie eine beträchtliche Menge DDL erfordern. Im obigen Beispiel würden wir jeden Monat eine neue Kindtabelle erstellen; daher kann es sinnvoll sein, ein Skript zu schreiben, das die nötige DDL automatisch erzeugt.

#### 5.12.3.2. Pflege bei Vererbungspartitionierung

Um alte Daten schnell zu entfernen, löschen Sie einfach die nicht mehr benötigte Kindtabelle:

```sql
DROP TABLE measurement_y2006m02;
```

Um die Kindtabelle aus der Vererbungshierarchie zu entfernen, den Zugriff auf sie als eigenständige Tabelle aber zu behalten:

```sql
ALTER TABLE measurement_y2006m02 NO INHERIT measurement;
```

Um eine neue Kindtabelle für neue Daten hinzuzufügen, erstellen Sie eine leere Kindtabelle genauso wie die ursprünglichen Kindtabellen:

```sql
CREATE TABLE measurement_y2008m02 (
    CHECK (logdate >= DATE '2008-02-01' AND logdate < DATE '2008-03-01')
) INHERITS (measurement);
```

Alternativ kann es sinnvoll sein, die neue Kindtabelle zu erstellen und zu befüllen, bevor sie der Tabellenhierarchie hinzugefügt wird. Dadurch können Daten geladen, geprüft und transformiert werden, bevor sie für Abfragen auf der Elterntabelle sichtbar werden.

```sql
CREATE TABLE measurement_y2008m02
  (LIKE measurement INCLUDING DEFAULTS INCLUDING CONSTRAINTS);
ALTER TABLE measurement_y2008m02 ADD CONSTRAINT y2008m02
   CHECK (logdate >= DATE '2008-02-01' AND logdate < DATE '2008-03-01');
\copy measurement_y2008m02 from 'measurement_y2008m02'
-- möglicherweise weitere Datenvorbereitung
ALTER TABLE measurement_y2008m02 INHERIT measurement;
```

#### 5.12.3.3. Einschränkungen

Für Partitionierung, die mit Vererbung implementiert ist, gelten folgende Einschränkungen:

- Es gibt keinen automatischen Weg zu prüfen, ob alle `CHECK`-Constraints sich gegenseitig ausschließen. Sicherer ist es, Code zu erstellen, der Kindtabellen erzeugt und zugehörige Objekte erstellt oder ändert, statt alles von Hand zu schreiben.
- Indizes und Fremdschlüssel-Constraints gelten für einzelne Tabellen und nicht für deren Vererbungskinder; dabei sind einige Einschränkungen zu beachten.
- Die hier gezeigten Schemata setzen voraus, dass sich die Werte der Schlüsselspalten einer Zeile nie ändern oder zumindest nicht so stark ändern, dass die Zeile in eine andere Partition verschoben werden müsste. Ein `UPDATE`, das dies versucht, schlägt wegen der `CHECK`-Constraints fehl. Wenn Sie solche Fälle behandeln müssen, können Sie passende Update-Trigger auf den Kindtabellen anlegen, was die Verwaltung der Struktur aber deutlich komplizierter macht.
- Manuelle `VACUUM`- und `ANALYZE`-Befehle verarbeiten automatisch alle Vererbungskindtabellen. Wenn das unerwünscht ist, können Sie das Schlüsselwort `ONLY` verwenden:

```sql
ANALYZE ONLY measurement;
```

Dieser Befehl verarbeitet nur die Wurzeltabelle.

- `INSERT`-Anweisungen mit `ON CONFLICT`-Klauseln funktionieren wahrscheinlich nicht wie erwartet, weil die `ON CONFLICT`-Aktion nur bei Unique-Verletzungen auf der angegebenen Zielrelation ausgeführt wird, nicht auf deren Kindrelationen.
- Trigger oder Regeln werden benötigt, um Zeilen an die gewünschte Kindtabelle weiterzuleiten, sofern die Anwendung das Partitionierungsschema nicht ausdrücklich kennt. Trigger können kompliziert zu schreiben sein und sind deutlich langsamer als das interne Tuple-Routing der deklarativen Partitionierung.

### 5.12.4. Partitionsbereinigung (Partition Pruning)

Partition Pruning ist eine Technik zur Abfrageoptimierung, die die Leistung bei deklarativ partitionierten Tabellen verbessert. Ein Beispiel:

```sql
SET enable_partition_pruning = on;                 -- the default
SELECT count(*) FROM measurement WHERE logdate >= DATE '2008-01-01';
```

Ohne Partition Pruning würde die obige Abfrage jede Partition der Tabelle `measurement` durchsuchen. Mit aktiviertem Partition Pruning untersucht der Planer die Definition jeder Partition und beweist, dass eine Partition nicht gescannt werden muss, weil sie keine Zeilen enthalten kann, die die `WHERE`-Klausel der Abfrage erfüllen. Wenn der Planer dies beweisen kann, schließt er die Partition aus dem Abfrageplan aus.

Mit dem Befehl `EXPLAIN` und dem Konfigurationsparameter `enable_partition_pruning` lässt sich der Unterschied zwischen einem Plan zeigen, bei dem Partitionen bereinigt wurden, und einem, bei dem dies nicht geschehen ist. Ein typischer nicht optimierter Plan für diese Tabellenstruktur ist:

```sql
SET enable_partition_pruning = off;
EXPLAIN SELECT count(*) FROM measurement WHERE logdate >= DATE '2008-01-01';
                                      QUERY PLAN
------------------------------------------------------------------------------------
 Aggregate (cost=188.76..188.77 rows=1 width=8)
   -> Append (cost=0.00..181.05 rows=3085 width=0)
        -> Seq Scan on measurement_y2006m02 (cost=0.00..33.12 rows=617 width=0)
              Filter: (logdate >= '2008-01-01'::date)
        -> Seq Scan on measurement_y2006m03 (cost=0.00..33.12 rows=617 width=0)
              Filter: (logdate >= '2008-01-01'::date)
...
        -> Seq Scan on measurement_y2007m11 (cost=0.00..33.12 rows=617 width=0)
              Filter: (logdate >= '2008-01-01'::date)
        -> Seq Scan on measurement_y2007m12 (cost=0.00..33.12 rows=617 width=0)
              Filter: (logdate >= '2008-01-01'::date)
        -> Seq Scan on measurement_y2008m01 (cost=0.00..33.12 rows=617 width=0)
              Filter: (logdate >= '2008-01-01'::date)
```

Einige oder alle Partitionen könnten Index-Scans statt vollständiger sequenzieller Tabellenscans verwenden. Entscheidend ist hier aber, dass die älteren Partitionen zur Beantwortung dieser Abfrage gar nicht gescannt werden müssen. Aktivieren wir Partition Pruning, erhalten wir einen deutlich günstigeren Plan mit demselben Ergebnis:

```sql
SET enable_partition_pruning = on;
EXPLAIN SELECT count(*) FROM measurement WHERE logdate >= DATE '2008-01-01';
                                      QUERY PLAN
------------------------------------------------------------------------------------
 Aggregate (cost=37.75..37.76 rows=1 width=8)
   -> Seq Scan on measurement_y2008m01 (cost=0.00..33.12 rows=617 width=0)
         Filter: (logdate >= '2008-01-01'::date)
```

Beachten Sie, dass Partition Pruning nur durch die implizit durch die Partitionsschlüssel definierten Constraints gesteuert wird, nicht durch vorhandene Indizes. Es ist daher nicht nötig, Indizes auf den Schlüsselspalten zu definieren. Ob für eine bestimmte Partition ein Index erstellt werden muss, hängt davon ab, ob Abfragen, die diese Partition scannen, normalerweise einen großen oder nur einen kleinen Teil der Partition lesen. Im letzteren Fall hilft ein Index, im ersteren nicht.

Partition Pruning kann nicht nur während der Planung einer Abfrage erfolgen, sondern auch während ihrer Ausführung. Das ist nützlich, weil dadurch mehr Partitionen entfernt werden können, wenn Klauseln Ausdrücke enthalten, deren Werte zur Planungszeit noch nicht bekannt sind, zum Beispiel Parameter aus einer `PREPARE`-Anweisung, ein Wert aus einer Unterabfrage oder ein parametrisierter Wert auf der inneren Seite eines Nested-Loop-Joins. Partition Pruning während der Ausführung kann zu folgenden Zeitpunkten stattfinden:

- Während der Initialisierung des Abfrageplans. Hier kann Partition Pruning für Parameterwerte durchgeführt werden, die während der Initialisierungsphase der Ausführung bekannt sind. Partitionen, die in dieser Phase entfernt werden, erscheinen nicht in der Ausgabe von `EXPLAIN` oder `EXPLAIN ANALYZE`. Die Anzahl der in dieser Phase entfernten Partitionen lässt sich an der Eigenschaft „Subplans Removed“ in der `EXPLAIN`-Ausgabe erkennen. Der Abfrageplaner erwirbt Locks für alle Partitionen, die Teil des Plans sind. Wenn der Executor jedoch einen zwischengespeicherten Plan verwendet, werden Locks nur für die Partitionen erworben, die nach dem Partition Pruning während der Initialisierungsphase übrig bleiben, also für die in der `EXPLAIN`-Ausgabe sichtbaren Partitionen und nicht für die durch „Subplans Removed“ genannten.
- Während der tatsächlichen Ausführung des Abfrageplans. Auch hier kann Partition Pruning Partitionen entfernen, wenn Werte erst während der Ausführung bekannt werden. Dazu gehören Werte aus Unterabfragen und Werte aus Ausführungsparametern, etwa aus parametrisierten Nested-Loop-Joins. Da sich solche Parameterwerte während einer Abfrage mehrfach ändern können, wird Partition Pruning immer dann durchgeführt, wenn sich ein verwendeter Ausführungsparameter ändert. Ob Partitionen in dieser Phase entfernt wurden, erfordert eine genaue Betrachtung der Eigenschaft `loops` in der Ausgabe von `EXPLAIN ANALYZE`. Unterpläne für verschiedene Partitionen können unterschiedliche Werte haben, je nachdem, wie oft sie während der Ausführung entfernt wurden. Einige können als `(never executed)` erscheinen, wenn sie jedes Mal entfernt wurden.

Partition Pruning kann mit der Einstellung `enable_partition_pruning` deaktiviert werden.

### 5.12.5. Partitionierung und Constraint Exclusion

Constraint Exclusion ist eine Abfrageoptimierungstechnik, die Partition Pruning ähnelt. Sie wird hauptsächlich für Partitionierung verwendet, die mit der klassischen Vererbungsmethode implementiert wurde, kann aber auch für andere Zwecke genutzt werden, einschließlich deklarativer Partitionierung.

Constraint Exclusion funktioniert sehr ähnlich wie Partition Pruning, verwendet jedoch die `CHECK`-Constraints jeder Tabelle, daher der Name, während Partition Pruning die Partitionsgrenzen der Tabelle verwendet, die nur bei deklarativer Partitionierung existieren. Ein weiterer Unterschied besteht darin, dass Constraint Exclusion nur zur Planungszeit angewendet wird; es gibt keinen Versuch, Partitionen zur Ausführungszeit zu entfernen.

Dass Constraint Exclusion `CHECK`-Constraints verwendet und deshalb im Vergleich zu Partition Pruning langsam ist, kann manchmal vorteilhaft sein: Da Constraints auch auf deklarativ partitionierten Tabellen zusätzlich zu ihren internen Partitionsgrenzen definiert werden können, kann Constraint Exclusion unter Umständen zusätzliche Partitionen aus dem Abfrageplan entfernen.

Die standardmäßige und empfohlene Einstellung von `constraint_exclusion` ist weder `on` noch `off`, sondern die Zwischeneinstellung `partition`. Sie bewirkt, dass die Technik nur auf Abfragen angewendet wird, die wahrscheinlich auf vererbungsbasierte partitionierte Tabellen zugreifen. Die Einstellung `on` veranlasst den Planer, `CHECK`-Constraints in allen Abfragen zu untersuchen, auch in einfachen Abfragen, die davon kaum profitieren.

Für Constraint Exclusion gelten folgende Einschränkungen:

- Constraint Exclusion wird nur während der Abfrageplanung angewendet, anders als Partition Pruning, das auch während der Abfrageausführung angewendet werden kann.
- Constraint Exclusion funktioniert nur, wenn die `WHERE`-Klausel der Abfrage Konstanten oder extern gelieferte Parameter enthält. Ein Vergleich mit einer nicht unveränderlichen Funktion wie `CURRENT_TIMESTAMP` kann zum Beispiel nicht optimiert werden, da der Planer nicht wissen kann, in welche Kindtabelle der Funktionswert zur Laufzeit fallen wird.
- Halten Sie Partitionierungs-Constraints einfach, sonst kann der Planer möglicherweise nicht beweisen, dass Kindtabellen nicht besucht werden müssen. Verwenden Sie einfache Gleichheitsbedingungen für List-Partitionierung oder einfache Bereichstests für Range-Partitionierung, wie in den vorigen Beispielen gezeigt. Eine gute Faustregel ist, dass Partitionierungs-Constraints nur Vergleiche der Partitionierungsspalten mit Konstanten unter Verwendung B-Tree-indexierbarer Operatoren enthalten sollten, weil im Partitionsschlüssel nur B-Tree-indexierbare Spalten erlaubt sind.
- Während Constraint Exclusion werden alle Constraints aller Kindtabellen der Elterntabelle untersucht; viele Kindtabellen können die Abfrageplanungszeit daher erheblich erhöhen. Die klassische vererbungsbasierte Partitionierung funktioniert daher mit vielleicht bis zu hundert Kindtabellen gut; versuchen Sie nicht, viele Tausend Kindtabellen zu verwenden.

### 5.12.6. Best Practices für deklarative Partitionierung

Die Entscheidung, wie eine Tabelle partitioniert wird, sollte sorgfältig getroffen werden, da schlechtes Design die Leistung von Abfrageplanung und -ausführung negativ beeinflussen kann.

Eine der wichtigsten Designentscheidungen betrifft die Spalte oder Spalten, nach denen Daten partitioniert werden. Häufig ist es am besten, nach der Spalte oder Spaltenmenge zu partitionieren, die am häufigsten in `WHERE`-Klauseln von Abfragen auf der partitionierten Tabelle vorkommt. `WHERE`-Klauseln, die mit den Partitionsgrenzen kompatibel sind, können verwendet werden, um nicht benötigte Partitionen zu entfernen. Anforderungen an einen `PRIMARY KEY` oder einen `UNIQUE`-Constraint können Sie allerdings zu anderen Entscheidungen zwingen. Auch das Entfernen unerwünschter Daten ist bei der Planung der Partitionierungsstrategie zu berücksichtigen. Eine ganze Partition kann recht schnell abgetrennt werden; daher kann es vorteilhaft sein, die Partitionsstrategie so zu entwerfen, dass alle auf einmal zu entfernenden Daten in einer einzigen Partition liegen.

Auch die Zielanzahl der Partitionen, in die die Tabelle aufgeteilt werden soll, ist eine kritische Entscheidung. Zu wenige Partitionen können bedeuten, dass Indizes zu groß bleiben und die Datenlokalität schlecht bleibt, was zu niedrigen Cache-Trefferraten führen kann. Zu viele Partitionen können jedoch ebenfalls Probleme verursachen: längere Planungszeiten und höheren Speicherverbrauch während Planung und Ausführung. Bei der Wahl der Partitionierung sollten auch mögliche zukünftige Änderungen bedacht werden. Wenn Sie zum Beispiel eine Partition pro Kunde wählen und derzeit wenige große Kunden haben, bedenken Sie, was passiert, wenn Sie in einigen Jahren viele kleine Kunden haben. In diesem Fall kann es besser sein, per `HASH` mit einer sinnvollen Anzahl von Partitionen zu partitionieren, statt per `LIST` zu partitionieren und zu hoffen, dass die Kundenzahl nicht über ein praktikables Maß hinaus wächst.

Unterpartitionierung kann nützlich sein, um Partitionen weiter aufzuteilen, die voraussichtlich größer werden als andere Partitionen. Eine weitere Möglichkeit ist Range-Partitionierung mit mehreren Spalten im Partitionsschlüssel. Beides kann leicht zu übermäßig vielen Partitionen führen; Zurückhaltung ist daher ratsam.

Es ist wichtig, den Overhead der Partitionierung während Abfrageplanung und -ausführung zu berücksichtigen. Der Abfrageplaner kann Partitionshierarchien mit bis zu einigen Tausend Partitionen im Allgemeinen recht gut handhaben, sofern typische Abfragen dem Planer erlauben, alle bis auf wenige Partitionen zu entfernen. Planungszeiten und Speicherverbrauch steigen, wenn nach dem Partition Pruning mehr Partitionen übrig bleiben. Ein weiterer Grund zur Vorsicht bei vielen Partitionen ist, dass der Speicherverbrauch des Servers im Lauf der Zeit deutlich wachsen kann, besonders wenn viele Sitzungen viele Partitionen berühren. Jede Partition erfordert nämlich, dass ihre Metadaten in den lokalen Speicher jeder Sitzung geladen werden, die sie berührt.

Bei Data-Warehouse-Workloads kann eine größere Anzahl von Partitionen sinnvoller sein als bei OLTP-Workloads. In Data Warehouses ist die Abfrageplanungszeit meist weniger kritisch, da der Großteil der Verarbeitungszeit während der Abfrageausführung anfällt. Bei beiden Workload-Typen ist es wichtig, früh die richtigen Entscheidungen zu treffen, weil eine spätere Neupartitionierung großer Datenmengen sehr langsam sein kann. Simulationen des geplanten Workloads sind oft hilfreich, um die Partitionierungsstrategie zu optimieren. Gehen Sie nie einfach davon aus, dass mehr Partitionen besser sind als weniger Partitionen, oder umgekehrt.

## 5.13. Fremddaten

PostgreSQL implementiert Teile der SQL/MED-Spezifikation und ermöglicht damit den Zugriff auf Daten, die außerhalb von PostgreSQL liegen, mit normalen SQL-Abfragen. Solche Daten werden als Fremddaten bezeichnet. Diese Verwendung ist nicht mit Fremdschlüsseln zu verwechseln, die eine Art Constraint innerhalb der Datenbank sind.

Der Zugriff auf Fremddaten erfolgt mithilfe eines Foreign Data Wrapper. Ein Foreign Data Wrapper ist eine Bibliothek, die mit einer externen Datenquelle kommunizieren kann und die Details der Verbindung zur Datenquelle sowie der Datenbeschaffung verbirgt. Einige Foreign Data Wrapper stehen als contrib-Module zur Verfügung; siehe Anhang F. Andere Arten von Foreign Data Wrappers können als Drittanbieterprodukte verfügbar sein. Wenn keiner der vorhandenen Foreign Data Wrappers Ihren Anforderungen entspricht, können Sie einen eigenen schreiben; siehe [Kapitel 58](58_Einen_Foreign_Data_Wrapper_schreiben.md).

Um auf Fremddaten zuzugreifen, müssen Sie ein Foreign-Server-Objekt erstellen, das beschreibt, wie die Verbindung zu einer bestimmten externen Datenquelle gemäß den Optionen ihres Foreign Data Wrappers hergestellt wird. Danach erstellen Sie eine oder mehrere Fremdtabellen, die die Struktur der entfernten Daten beschreiben. Eine Fremdtabelle kann in Abfragen wie eine normale Tabelle verwendet werden, hat aber keinen Speicher im PostgreSQL-Server. Wann immer sie verwendet wird, fordert PostgreSQL den Foreign Data Wrapper auf, Daten aus der externen Quelle zu holen oder bei Aktualisierungsbefehlen Daten an die externe Quelle zu übertragen.

Der Zugriff auf entfernte Daten kann eine Authentifizierung gegenüber der externen Datenquelle erfordern. Diese Informationen können durch ein User Mapping bereitgestellt werden, das zusätzliche Daten wie Benutzernamen und Passwörter auf Grundlage der aktuellen PostgreSQL-Rolle liefern kann.

Weitere Informationen finden Sie unter `CREATE FOREIGN DATA WRAPPER`, `CREATE SERVER`, `CREATE USER MAPPING`, `CREATE FOREIGN TABLE` und `IMPORT FOREIGN SCHEMA`.

## 5.14. Weitere Datenbankobjekte

Tabellen sind die zentralen Objekte einer relationalen Datenbankstruktur, weil sie Ihre Daten enthalten. Sie sind aber nicht die einzigen Objekte, die in einer Datenbank existieren. Viele weitere Arten von Objekten können erstellt werden, um die Nutzung und Verwaltung der Daten effizienter oder bequemer zu machen. Sie werden in diesem Kapitel nicht behandelt, aber hier ist eine Liste, damit Sie wissen, was möglich ist:

- Views
- Funktionen, Prozeduren und Operatoren
- Datentypen und Domains
- Trigger und Rewrite-Regeln

Detaillierte Informationen zu diesen Themen finden Sie in Teil V.

## 5.15. Abhängigkeitsverfolgung

Wenn Sie komplexe Datenbankstrukturen mit vielen Tabellen, Fremdschlüssel-Constraints, Views, Triggern, Funktionen usw. erstellen, erzeugen Sie implizit ein Netz von Abhängigkeiten zwischen den Objekten. Eine Tabelle mit einem Fremdschlüssel-Constraint hängt zum Beispiel von der Tabelle ab, die sie referenziert.

Um die Integrität der gesamten Datenbankstruktur sicherzustellen, sorgt PostgreSQL dafür, dass Objekte nicht gelöscht werden können, solange andere Objekte noch von ihnen abhängen. Der Versuch, die Tabelle `products` aus [Abschnitt 5.5.5](05_Datendefinition.md#555-fremdschlüssel) zu löschen, während die Tabelle `orders` von ihr abhängt, würde zum Beispiel eine Fehlermeldung wie diese ergeben:

```sql
DROP TABLE products;

ERROR: cannot drop table products because other objects depend on it
DETAIL: constraint orders_product_no_fkey on table orders depends on table products
HINT: Use DROP ... CASCADE to drop the dependent objects too.
```

Die Fehlermeldung enthält einen nützlichen Hinweis: Wenn Sie nicht alle abhängigen Objekte einzeln löschen möchten, können Sie ausführen:

```sql
DROP TABLE products CASCADE;
```

Dann werden alle abhängigen Objekte entfernt, ebenso rekursiv alle Objekte, die von diesen abhängen. In diesem Fall wird die Tabelle `orders` nicht entfernt, sondern nur der Fremdschlüssel-Constraint. Dort endet die Kaskade, weil nichts vom Fremdschlüssel-Constraint abhängt. Wenn Sie prüfen möchten, was `DROP ... CASCADE` tun würde, führen Sie `DROP` ohne `CASCADE` aus und lesen Sie die `DETAIL`-Ausgabe.

Fast alle `DROP`-Befehle in PostgreSQL unterstützen die Angabe von `CASCADE`. Die Art möglicher Abhängigkeiten variiert natürlich je nach Objekttyp. Sie können auch `RESTRICT` statt `CASCADE` schreiben, um das Standardverhalten zu erhalten, das das Löschen von Objekten verhindert, von denen andere Objekte abhängen.

> **Hinweis:** Nach dem SQL-Standard muss in einem `DROP`-Befehl entweder `RESTRICT` oder `CASCADE` angegeben werden. Kein Datenbanksystem erzwingt diese Regel tatsächlich, aber ob das Standardverhalten `RESTRICT` oder `CASCADE` ist, unterscheidet sich zwischen Systemen.

Wenn ein `DROP`-Befehl mehrere Objekte auflistet, ist `CASCADE` nur erforderlich, wenn Abhängigkeiten außerhalb der angegebenen Gruppe bestehen. Bei `DROP TABLE tab1, tab2` würde also ein Fremdschlüssel von `tab2` auf `tab1` nicht bedeuten, dass `CASCADE` nötig ist.

Für eine benutzerdefinierte Funktion oder Prozedur, deren Rumpf als Zeichenkettenliteral definiert ist, verfolgt PostgreSQL Abhängigkeiten, die mit den von außen sichtbaren Eigenschaften der Funktion zusammenhängen, etwa Argument- und Ergebnistypen, aber keine Abhängigkeiten, die nur durch Untersuchung des Funktionsrumpfs bekannt wären. Betrachten Sie dieses Beispiel:

```sql
CREATE TYPE rainbow AS ENUM ('red', 'orange', 'yellow',
                             'green', 'blue', 'purple');

CREATE TABLE my_colors (color rainbow, note text);

CREATE FUNCTION get_color_note (rainbow) RETURNS text AS
  'SELECT note FROM my_colors WHERE color = $1'
  LANGUAGE SQL;
```

Eine Erklärung von SQL-Sprachfunktionen finden Sie in [Abschnitt 36.5](36_SQL_erweitern.md#365-querylanguagesqlfunktionen). PostgreSQL erkennt, dass die Funktion `get_color_note` vom Typ `rainbow` abhängt: Das Löschen des Typs würde das Löschen der Funktion erzwingen, weil ihr Argumenttyp nicht mehr definiert wäre. PostgreSQL betrachtet `get_color_note` jedoch nicht als von der Tabelle `my_colors` abhängig und löscht die Funktion daher nicht, wenn die Tabelle gelöscht wird. Dieser Ansatz hat Nachteile, aber auch Vorteile. Die Funktion ist in gewissem Sinn weiterhin gültig, wenn die Tabelle fehlt, auch wenn ihre Ausführung einen Fehler verursachen würde; das Erstellen einer neuen Tabelle mit demselben Namen würde die Funktion wieder nutzbar machen.

Für eine SQL-Sprachfunktion oder -Prozedur, deren Rumpf im Stil des SQL-Standards geschrieben ist, wird der Rumpf dagegen zur Definitionszeit geparst, und alle vom Parser erkannten Abhängigkeiten werden gespeichert. Wenn wir die obige Funktion also so schreiben:

```sql
CREATE FUNCTION get_color_note (rainbow) RETURNS text
BEGIN ATOMIC
  SELECT note FROM my_colors WHERE color = $1;
END;
```

dann ist die Abhängigkeit der Funktion von der Tabelle `my_colors` bekannt und wird bei `DROP` erzwungen.
