# 6. Datenmanipulation

Das vorige Kapitel hat beschrieben, wie Tabellen und andere Strukturen zum Speichern von Daten erstellt werden. Nun ist es Zeit, die Tabellen mit Daten zu füllen. Dieses Kapitel behandelt, wie Tabellendaten eingefügt, aktualisiert und gelöscht werden. Das nächste Kapitel erklärt schließlich, wie Sie Ihre längst vermissten Daten wieder aus der Datenbank herausholen.

## 6.1. Daten einfügen

Wenn eine Tabelle erstellt wird, enthält sie noch keine Daten. Bevor eine Datenbank wirklich nützlich sein kann, müssen zunächst Daten eingefügt werden. Daten werden zeilenweise eingefügt. Sie können auch mehr als eine Zeile in einem einzigen Befehl einfügen, aber es ist nicht möglich, etwas einzufügen, das keine vollständige Zeile ist. Selbst wenn Sie nur einige Spaltenwerte kennen, muss eine vollständige Zeile erzeugt werden.

Zum Erzeugen einer neuen Zeile verwenden Sie den Befehl `INSERT`. Der Befehl benötigt den Tabellennamen und die Spaltenwerte. Betrachten wir zum Beispiel die Tabelle `products` aus [Kapitel 5](05_Datendefinition.md):

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);
```

Ein Beispielbefehl zum Einfügen einer Zeile wäre:

```sql
INSERT INTO products VALUES (1, 'Cheese', 9.99);
```

Die Datenwerte werden in der Reihenfolge aufgelistet, in der die Spalten in der Tabelle erscheinen, getrennt durch Kommas. Normalerweise sind die Datenwerte Literale oder Konstanten, aber skalare Ausdrücke sind ebenfalls erlaubt.

Diese Syntax hat den Nachteil, dass Sie die Reihenfolge der Spalten in der Tabelle kennen müssen. Um das zu vermeiden, können Sie die Spalten auch ausdrücklich auflisten. Die folgenden beiden Befehle haben beispielsweise dieselbe Wirkung wie der obige:

```sql
INSERT INTO products (product_no, name, price) VALUES (1, 'Cheese', 9.99);
INSERT INTO products (name, price, product_no) VALUES ('Cheese', 9.99, 1);
```

Viele Benutzer halten es für gute Praxis, die Spaltennamen immer anzugeben.

Wenn Sie nicht für alle Spalten Werte haben, können Sie einige Spalten weglassen. In diesem Fall werden die Spalten mit ihren Vorgabewerten gefüllt. Zum Beispiel:

```sql
INSERT INTO products (product_no, name) VALUES (1, 'Cheese');
INSERT INTO products VALUES (1, 'Cheese');
```

Die zweite Form ist eine PostgreSQL-Erweiterung. Sie füllt die Spalten von links mit so vielen Werten, wie angegeben wurden; der Rest erhält Vorgabewerte.

Der Klarheit halber können Sie Vorgabewerte auch ausdrücklich anfordern, für einzelne Spalten oder für die gesamte Zeile:

```sql
INSERT INTO products (product_no, name, price) VALUES (1, 'Cheese', DEFAULT);

INSERT INTO products DEFAULT VALUES;
```

Sie können mehrere Zeilen mit einem einzigen Befehl einfügen:

```sql
INSERT INTO products (product_no, name, price) VALUES
    (1, 'Cheese', 9.99),
    (2, 'Bread', 1.99),
    (3, 'Milk', 2.99);
```

Es ist auch möglich, das Ergebnis einer Abfrage einzufügen; dieses Ergebnis kann keine, eine oder viele Zeilen enthalten:

```sql
INSERT INTO products (product_no, name, price)
  SELECT product_no, name, price FROM new_products
    WHERE release_date = 'today';
```

Damit steht die volle Leistungsfähigkeit des SQL-Abfragemechanismus ([Kapitel 7](07_Abfragen.md)) zur Berechnung der einzufügenden Zeilen zur Verfügung.

> **Tipp:** Wenn Sie viele Daten auf einmal einfügen, sollten Sie den Befehl `COPY` in Betracht ziehen. Er ist nicht so flexibel wie `INSERT`, aber effizienter. Weitere Informationen zur Verbesserung der Leistung beim Massenladen finden Sie in [Abschnitt 14.4](14_Hinweise_zur_Performance.md#144-eine-datenbank-befüllen).

## 6.2. Daten aktualisieren

Das Ändern von Daten, die bereits in der Datenbank vorhanden sind, wird als Aktualisieren bezeichnet. Sie können einzelne Zeilen, alle Zeilen einer Tabelle oder eine Teilmenge aller Zeilen aktualisieren. Jede Spalte kann separat aktualisiert werden; die übrigen Spalten bleiben davon unberührt.

Zum Aktualisieren vorhandener Zeilen verwenden Sie den Befehl `UPDATE`. Dafür werden drei Informationen benötigt:

1. Der Name der Tabelle und der zu aktualisierenden Spalte
2. Der neue Wert der Spalte
3. Welche Zeile oder Zeilen aktualisiert werden sollen

Wie aus [Kapitel 5](05_Datendefinition.md) bekannt, stellt SQL im Allgemeinen keinen eindeutigen Bezeichner für Zeilen bereit. Deshalb ist es nicht immer möglich, direkt anzugeben, welche Zeile aktualisiert werden soll. Stattdessen geben Sie Bedingungen an, die eine Zeile erfüllen muss, um aktualisiert zu werden. Nur wenn Sie in der Tabelle einen Primärschlüssel haben, unabhängig davon, ob Sie ihn deklariert haben oder nicht, können Sie einzelne Zeilen zuverlässig ansprechen, indem Sie eine Bedingung wählen, die dem Primärschlüssel entspricht. Grafische Datenbankwerkzeuge stützen sich darauf, um das einzelne Aktualisieren von Zeilen zu ermöglichen.

Dieser Befehl aktualisiert zum Beispiel alle Produkte, deren Preis 5 beträgt, auf einen Preis von 10:

```sql
UPDATE products SET price = 10 WHERE price = 5;
```

Dadurch können null, eine oder viele Zeilen aktualisiert werden. Es ist kein Fehler, eine Aktualisierung zu versuchen, die auf keine Zeile passt.

Schauen wir uns den Befehl genauer an. Zuerst steht das Schlüsselwort `UPDATE`, gefolgt vom Tabellennamen. Wie üblich kann der Tabellenname schemaqualifiziert sein; andernfalls wird er im Suchpfad gesucht. Danach folgt das Schlüsselwort `SET`, gefolgt vom Spaltennamen, einem Gleichheitszeichen und dem neuen Spaltenwert. Der neue Spaltenwert kann ein beliebiger skalarer Ausdruck sein, nicht nur eine Konstante. Wenn Sie zum Beispiel den Preis aller Produkte um 10 Prozent erhöhen möchten, könnten Sie schreiben:

```sql
UPDATE products SET price = price * 1.10;
```

Wie zu sehen ist, kann der Ausdruck für den neuen Wert auf vorhandene Werte der Zeile verweisen. Außerdem haben wir die `WHERE`-Klausel weggelassen. Wird sie weggelassen, bedeutet das, dass alle Zeilen der Tabelle aktualisiert werden. Ist sie vorhanden, werden nur die Zeilen aktualisiert, die die `WHERE`-Bedingung erfüllen. Beachten Sie, dass das Gleichheitszeichen in der `SET`-Klausel eine Zuweisung ist, während das Gleichheitszeichen in der `WHERE`-Klausel ein Vergleich ist; dadurch entsteht aber keine Mehrdeutigkeit. Die `WHERE`-Bedingung muss natürlich kein Gleichheitstest sein. Viele andere Operatoren sind verfügbar (siehe [Kapitel 9](09_Funktionen_und_Operatoren.md)). Der Ausdruck muss jedoch ein boolesches Ergebnis liefern.

Sie können in einem `UPDATE`-Befehl mehr als eine Spalte aktualisieren, indem Sie in der `SET`-Klausel mehrere Zuweisungen auflisten. Zum Beispiel:

```sql
UPDATE mytable SET a = 5, b = 3, c = 1 WHERE a > 0;
```

## 6.3. Daten löschen

Bisher haben wir erklärt, wie Daten zu Tabellen hinzugefügt und wie Daten geändert werden. Nun bleibt zu besprechen, wie nicht mehr benötigte Daten entfernt werden. So wie Daten nur als ganze Zeilen hinzugefügt werden können, können Sie auch nur ganze Zeilen aus einer Tabelle entfernen. Im vorigen Abschnitt wurde erklärt, dass SQL keine Möglichkeit bietet, einzelne Zeilen direkt anzusprechen. Deshalb können Zeilen nur entfernt werden, indem Bedingungen angegeben werden, denen die zu entfernenden Zeilen entsprechen müssen. Wenn die Tabelle einen Primärschlüssel hat, können Sie die exakte Zeile angeben. Sie können aber auch Gruppen von Zeilen entfernen, die eine Bedingung erfüllen, oder alle Zeilen der Tabelle auf einmal entfernen.

Zum Entfernen von Zeilen verwenden Sie den Befehl `DELETE`; die Syntax ähnelt stark dem Befehl `UPDATE`. Um beispielsweise alle Zeilen aus der Tabelle `products` zu entfernen, deren Preis 10 beträgt, verwenden Sie:

```sql
DELETE FROM products WHERE price = 10;
```

Wenn Sie einfach schreiben:

```sql
DELETE FROM products;
```

dann werden alle Zeilen der Tabelle gelöscht. Programmierer, sei wachsam.

## 6.4. Daten aus geänderten Zeilen zurückgeben

Manchmal ist es nützlich, Daten aus geänderten Zeilen zu erhalten, während diese manipuliert werden. Die Befehle `INSERT`, `UPDATE`, `DELETE` und `MERGE` besitzen alle eine optionale `RETURNING`-Klausel, die dies unterstützt. Die Verwendung von `RETURNING` vermeidet eine zusätzliche Datenbankabfrage zum Einsammeln der Daten und ist besonders wertvoll, wenn es sonst schwierig wäre, die geänderten Zeilen zuverlässig zu identifizieren.

Der zulässige Inhalt einer `RETURNING`-Klausel entspricht der Ausgabeliste eines `SELECT`-Befehls (siehe [Abschnitt 7.3](07_Abfragen.md#73-auswahllisten)). Sie kann Spaltennamen der Zieltabelle des Befehls enthalten oder Wertausdrücke, die diese Spalten verwenden. Eine übliche Kurzform ist `RETURNING *`, wodurch alle Spalten der Zieltabelle in ihrer Reihenfolge ausgewählt werden.

Bei einem `INSERT` sind die für `RETURNING` standardmäßig verfügbaren Daten die Zeile, wie sie eingefügt wurde. Bei trivialen Einfügungen ist das nicht besonders nützlich, da es nur die vom Client gelieferten Daten wiederholt. Es kann aber sehr praktisch sein, wenn berechnete Vorgabewerte verwendet werden. Wenn zum Beispiel eine `serial`-Spalte eindeutige Kennungen erzeugt, kann `RETURNING` die einer neuen Zeile zugewiesene ID zurückgeben:

```sql
CREATE TABLE users (firstname text, lastname text, id serial primary key);

INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool')
 RETURNING id;
```

Die `RETURNING`-Klausel ist auch bei `INSERT ... SELECT` sehr nützlich.

Bei einem `UPDATE` sind die für `RETURNING` standardmäßig verfügbaren Daten der neue Inhalt der geänderten Zeile. Zum Beispiel:

```sql
UPDATE products SET price = price * 1.10
  WHERE price <= 99.99
  RETURNING name, price AS new_price;
```

Bei einem `DELETE` sind die für `RETURNING` standardmäßig verfügbaren Daten der Inhalt der gelöschten Zeile. Zum Beispiel:

```sql
DELETE FROM products
  WHERE obsoletion_date = 'today'
  RETURNING *;
```

Bei einem `MERGE` sind die für `RETURNING` standardmäßig verfügbaren Daten der Inhalt der Quellzeile plus der Inhalt der eingefügten, aktualisierten oder gelöschten Zielzeile. Da Quelle und Ziel häufig viele gleichnamige Spalten haben, kann `RETURNING *` zu vielen doppelten Spalten führen. Es ist daher oft nützlicher, die Angabe so zu qualifizieren, dass nur die Quell- oder Zielzeile zurückgegeben wird. Zum Beispiel:

```sql
MERGE INTO products p USING new_products n ON p.product_no = n.product_no
  WHEN NOT MATCHED THEN INSERT VALUES (n.product_no, n.name, n.price)
  WHEN MATCHED THEN UPDATE SET name = n.name, price = n.price
  RETURNING p.*;
```

In jedem dieser Befehle ist es auch möglich, ausdrücklich den alten und den neuen Inhalt der geänderten Zeile zurückzugeben. Zum Beispiel:

```sql
UPDATE products SET price = price * 1.10
  WHERE price <= 99.99
  RETURNING name, old.price AS old_price, new.price AS new_price,
            new.price - old.price AS price_change;
```

In diesem Beispiel ist `new.price` dasselbe wie nur `price`; die Schreibweise macht die Bedeutung aber klarer.

Diese Syntax zum Zurückgeben alter und neuer Werte ist in den Befehlen `INSERT`, `UPDATE`, `DELETE` und `MERGE` verfügbar. Typischerweise sind alte Werte bei einem `INSERT` jedoch `NULL`, und neue Werte bei einem `DELETE` ebenfalls `NULL`. Es gibt dennoch Situationen, in denen sie auch für diese Befehle nützlich sein können. Bei einem `INSERT` mit einer `ON CONFLICT DO UPDATE`-Klausel sind die alten Werte für konfliktierende Zeilen zum Beispiel nicht `NULL`. Ebenso können neue Werte nicht `NULL` sein, wenn ein `DELETE` durch eine Rewrite-Regel in ein `UPDATE` umgewandelt wird.

Wenn auf der Zieltabelle Trigger ([Kapitel 37](37_Trigger.md)) vorhanden sind, sind die für `RETURNING` verfügbaren Daten die Zeile, wie sie von den Triggern geändert wurde. Das Inspizieren von durch Trigger berechneten Spalten ist daher ein weiterer häufiger Anwendungsfall für `RETURNING`.
