# 41. PL/pgSQL - SQL-Prozedursprache

## 41.1. Überblick

PL/pgSQL ist eine ladbare prozedurale Sprache für das PostgreSQL-Datenbanksystem. Die Entwurfsziele von PL/pgSQL bestanden darin, eine ladbare prozedurale Sprache zu schaffen, die

- zum Erstellen von Funktionen, Prozeduren und Triggern verwendet werden kann,

- der SQL-Sprache Kontrollstrukturen hinzufügt,

- komplexe Berechnungen ausführen kann,

- alle benutzerdefinierten Typen, Funktionen, Prozeduren und Operatoren erbt,

- vom Server als vertrauenswürdig definiert werden kann,

- einfach zu verwenden ist.

Mit PL/pgSQL erstellte Funktionen können überall dort verwendet werden, wo eingebaute Funktionen verwendet werden könnten. Zum Beispiel ist es möglich, komplexe bedingte Berechnungsfunktionen zu erstellen und diese später zum Definieren von Operatoren oder in Indexausdrücken zu verwenden.

In PostgreSQL 9.0 und später ist PL/pgSQL standardmäßig installiert. Es bleibt jedoch ein ladbares Modul, sodass besonders sicherheitsbewusste Administratoren es entfernen können.

### 41.1.1. Vorteile der Verwendung von PL/pgSQL

SQL ist die Sprache, die PostgreSQL und die meisten anderen relationalen Datenbanken als Abfragesprache verwenden. Sie ist portabel und leicht zu lernen. Jede SQL-Anweisung muss jedoch einzeln vom Datenbankserver ausgeführt werden.

Das bedeutet, dass Ihre Client-Anwendung jede Abfrage an den Datenbankserver senden, auf deren Verarbeitung warten, die Ergebnisse empfangen und verarbeiten, Berechnungen durchführen und dann weitere Abfragen an den Server senden muss. All dies verursacht Interprozesskommunikation und zusätzlich Netzwerk-Overhead, wenn sich Ihr Client auf einem anderen Rechner als der Datenbankserver befindet.

Mit PL/pgSQL können Sie einen Berechnungsblock und eine Reihe von Abfragen innerhalb des Datenbankservers bündeln. Dadurch erhalten Sie die Leistungsfähigkeit einer prozeduralen Sprache und die einfache Verwendung von SQL, sparen aber beträchtlich Overhead bei der Client/Server-Kommunikation.

- Zusätzliche Roundtrips zwischen Client und Server entfallen.

- Zwischenergebnisse, die der Client nicht benötigt, müssen nicht zwischen Server und Client marshaled oder übertragen werden.

- Mehrere Durchläufe des Query-Parsings können vermieden werden.

Verglichen mit einer Anwendung, die keine gespeicherten Funktionen verwendet, kann dies zu einer erheblichen Leistungssteigerung führen.

Außerdem können Sie mit PL/pgSQL alle Datentypen, Operatoren und Funktionen von SQL verwenden.

### 41.1.2. Unterstützte Argument- und Ergebnisdatentypen

In PL/pgSQL geschriebene Funktionen können jeden vom Server unterstützten skalaren Datentyp und Array-Datentyp als Argument akzeptieren und ein Ergebnis jedes dieser Typen zurückgeben. Sie können außerdem jeden namentlich angegebenen zusammengesetzten Typ (Zeilentyp) akzeptieren oder zurückgeben. Es ist auch möglich, eine PL/pgSQL-Funktion so zu deklarieren, dass sie `record` akzeptiert; das bedeutet, dass jeder zusammengesetzte Typ als Eingabe zulässig ist. Ebenso kann sie `record` zurückgeben; dann ist das Ergebnis ein Zeilentyp, dessen Spalten durch die Spezifikation in der aufrufenden Abfrage bestimmt werden, wie in Abschnitt 7.2.1.4 beschrieben.

PL/pgSQL-Funktionen können mit dem Marker `VARIADIC` so deklariert werden, dass sie eine variable Anzahl von Argumenten akzeptieren. Das funktioniert genau wie bei SQL-Funktionen, wie in Abschnitt 36.5.6 beschrieben.

PL/pgSQL-Funktionen können außerdem so deklariert werden, dass sie die in Abschnitt 36.2.5 beschriebenen polymorphen Typen akzeptieren und zurückgeben. Dadurch können die tatsächlich von der Funktion verarbeiteten Datentypen von Aufruf zu Aufruf variieren. Beispiele erscheinen in Abschnitt 41.3.1.

PL/pgSQL-Funktionen können außerdem so deklariert werden, dass sie eine „Menge“ (oder Tabelle) eines beliebigen Datentyps zurückgeben, der als einzelne Instanz zurückgegeben werden kann. Eine solche Funktion erzeugt ihre Ausgabe, indem sie für jedes gewünschte Element der Ergebnismenge `RETURN NEXT` ausführt oder mit `RETURN QUERY` das Ergebnis einer ausgewerteten Abfrage ausgibt.

Schließlich kann eine PL/pgSQL-Funktion so deklariert werden, dass sie `void` zurückgibt, wenn sie keinen sinnvollen Rückgabewert besitzt. Alternativ könnte sie in diesem Fall als Prozedur geschrieben werden.

PL/pgSQL-Funktionen können auch mit Ausgabeparametern deklariert werden, statt den Rückgabetyp explizit anzugeben. Das fügt der Sprache keine grundlegend neue Fähigkeit hinzu, ist aber oft praktisch, besonders beim Zurückgeben mehrerer Werte. Die Schreibweise `RETURNS TABLE` kann ebenfalls anstelle von `RETURNS SETOF` verwendet werden.

Konkrete Beispiele erscheinen in Abschnitt 41.3.1 und Abschnitt 41.6.1.

## 41.2. Struktur von PL/pgSQL

In PL/pgSQL geschriebene Funktionen werden dem Server durch Ausführen von `CREATE FUNCTION`-Befehlen definiert. Ein solcher Befehl sähe normalerweise etwa so aus:

```sql
CREATE FUNCTION somefunc(integer, text) RETURNS integer
AS 'function body text'
LANGUAGE plpgsql;
```

Der Funktionsrumpf ist aus Sicht von `CREATE FUNCTION` einfach ein Zeichenkettenliteral. Es ist oft hilfreich, Dollar-Quoting (siehe Abschnitt 4.1.2.4) zu verwenden, um den Funktionsrumpf zu schreiben, statt die normale Syntax mit einfachen Anführungszeichen zu benutzen. Ohne Dollar-Quoting müssen alle einfachen Anführungszeichen oder Backslashes im Funktionsrumpf durch Verdoppeln escaped werden. Fast alle Beispiele in diesem Kapitel verwenden dollar-gequotete Literale für ihre Funktionsrümpfe.

PL/pgSQL ist eine blockstrukturierte Sprache. Der vollständige Text eines Funktionsrumpfs muss ein Block sein. Ein Block ist folgendermaßen definiert:

```text
[ <<label>> ]
[ DECLARE
    declarations ]
BEGIN
    statements
END [ label ];
```

Jede Deklaration und jede Anweisung innerhalb eines Blocks wird mit einem Semikolon abgeschlossen. Ein Block, der innerhalb eines anderen Blocks erscheint, muss nach `END` ein Semikolon haben, wie oben gezeigt; das abschließende `END`, das einen Funktionsrumpf beendet, benötigt jedoch kein Semikolon.

> Tipp: Ein häufiger Fehler ist es, direkt nach `BEGIN` ein Semikolon zu schreiben. Das ist falsch und führt zu einem Syntaxfehler.

Ein Label wird nur benötigt, wenn Sie den Block für die Verwendung in einer `EXIT`-Anweisung identifizieren oder die Namen der im Block deklarierten Variablen qualifizieren möchten. Wenn nach `END` ein Label angegeben wird, muss es mit dem Label am Anfang des Blocks übereinstimmen.

Alle Schlüsselwörter sind unabhängig von der Groß-/Kleinschreibung. Bezeichner werden implizit in Kleinschreibung umgewandelt, sofern sie nicht doppelt gequotet sind, genau wie in normalen SQL-Befehlen.

Kommentare funktionieren in PL/pgSQL-Code genauso wie in normalem SQL. Ein doppelter Bindestrich (`--`) beginnt einen Kommentar, der bis zum Ende der Zeile reicht. Ein `/*` beginnt einen Blockkommentar, der bis zum passenden Auftreten von `*/` reicht. Blockkommentare können verschachtelt werden.

Jede Anweisung im Anweisungsabschnitt eines Blocks kann ein Unterblock sein. Unterblöcke können zur logischen Gruppierung oder dazu verwendet werden, Variablen auf eine kleine Gruppe von Anweisungen zu beschränken. In einem Unterblock deklarierte Variablen verdecken gleichnamige Variablen äußerer Blöcke für die Dauer des Unterblocks; auf die äußeren Variablen können Sie aber dennoch zugreifen, wenn Sie deren Namen mit dem Label ihres Blocks qualifizieren. Beispiel:

```sql
CREATE FUNCTION somefunc() RETURNS integer AS $$
<< outerblock >>
DECLARE
    quantity integer := 30;
BEGIN
    RAISE NOTICE 'Quantity here is %', quantity; -- Prints 30
    quantity := 50;
    --
    -- Create a subblock
    --
    DECLARE
         quantity integer := 80;
    BEGIN
         RAISE NOTICE 'Quantity here is %', quantity; -- Prints 80
         RAISE NOTICE 'Outer quantity here is %',
 outerblock.quantity; -- Prints 50
    END;

      RAISE NOTICE 'Quantity here is %', quantity;                           -- Prints 50

     RETURN quantity;
END;
$$ LANGUAGE plpgsql;
```

> Hinweis: Tatsächlich gibt es einen verborgenen „äußeren Block“ um den Rumpf jeder PL/pgSQL-Funktion. Dieser Block stellt die Deklarationen der Funktionsparameter (falls vorhanden) sowie einige besondere Variablen wie `FOUND` bereit (siehe Abschnitt 41.5.5). Der äußere Block ist mit dem Namen der Funktion gelabelt, sodass Parameter und besondere Variablen mit dem Funktionsnamen qualifiziert werden können.

Es ist wichtig, die Verwendung von `BEGIN`/`END` zum Gruppieren von Anweisungen in PL/pgSQL nicht mit den gleichnamigen SQL-Befehlen zur Transaktionssteuerung zu verwechseln. `BEGIN`/`END` in PL/pgSQL dienen nur der Gruppierung; sie starten oder beenden keine Transaktion. Informationen zum Verwalten von Transaktionen in PL/pgSQL finden Sie in [Abschnitt 41.8](#418-transaktionsverwaltung). Außerdem bildet ein Block, der eine `EXCEPTION`-Klausel enthält, effektiv eine Subtransaktion, die zurückgerollt werden kann, ohne die äußere Transaktion zu beeinflussen. Mehr dazu finden Sie in [Abschnitt 41.6.8](#4168-fehler-abfangen).

## 41.3. Deklarationen

Alle in einem Block verwendeten Variablen müssen im Deklarationsabschnitt des Blocks deklariert werden. Die einzigen Ausnahmen sind die Schleifenvariable einer `FOR`-Schleife, die über einen Bereich ganzzahliger Werte iteriert, sie wird automatisch als `integer`-Variable deklariert, und entsprechend die Schleifenvariable einer `FOR`-Schleife, die über das Ergebnis eines Cursors iteriert, sie wird automatisch als `record`-Variable deklariert.

PL/pgSQL-Variablen können jeden SQL-Datentyp haben, etwa `integer`, `varchar` und `char`.

Hier sind einige Beispiele für Variablendeklarationen:

```sql
user_id integer;
quantity numeric(5);
url varchar;
myrow tablename%ROWTYPE;
myfield tablename.columnname%TYPE;
arow RECORD;
```

Die allgemeine Syntax einer Variablendeklaration lautet:

```text
name [ CONSTANT ] type [ COLLATE collation_name ] [ NOT NULL ]
    [ { DEFAULT | := | = } expression ];
```

Die `DEFAULT`-Klausel gibt, falls sie vorhanden ist, den Anfangswert an, der der Variablen beim Eintritt in den Block zugewiesen wird. Wenn die `DEFAULT`-Klausel nicht angegeben ist, wird die Variable mit dem SQL-Nullwert initialisiert. Die Option `CONSTANT` verhindert, dass der Variablen nach der Initialisierung erneut ein Wert zugewiesen wird, sodass ihr Wert für die Dauer des Blocks konstant bleibt. Die Option `COLLATE` gibt eine Kollation an, die für die Variable verwendet werden soll (siehe Abschnitt 41.3.6). Wenn `NOT NULL` angegeben ist, führt eine Zuweisung eines Nullwerts zu einem Laufzeitfehler. Alle als `NOT NULL` deklarierten Variablen müssen einen nichtnulligen Standardwert besitzen. Das Gleichheitszeichen (`=`) kann anstelle des PL/SQL-kompatiblen `:=` verwendet werden.

Der Standardwert einer Variablen wird jedes Mal ausgewertet und der Variablen zugewiesen, wenn der Block betreten wird, nicht nur einmal pro Funktionsaufruf. Wenn Sie also beispielsweise einer Variablen vom Typ `timestamp` den Wert `now()` zuweisen, enthält die Variable die Zeit des aktuellen Funktionsaufrufs, nicht die Zeit, zu der die Funktion vorkompiliert wurde.

Beispiele:

```sql
quantity integer DEFAULT 32;
url varchar := 'http://mysite.com';
transaction_time CONSTANT timestamp with time zone := now();
```

Nach der Deklaration kann der Wert einer Variablen in späteren Initialisierungsausdrücken im selben Block verwendet werden, zum Beispiel:

```sql
DECLARE
  x integer := 1;
  y integer := x + 1;
```

### 41.3.1. Funktionsparameter deklarieren

An Funktionen übergebene Parameter werden mit den Bezeichnern `$1`, `$2` usw. benannt. Optional können für `$n`-Parameternamen Aliasse deklariert werden, um die Lesbarkeit zu erhöhen. Danach kann entweder der Alias oder der numerische Bezeichner verwendet werden, um auf den Parameterwert zu verweisen.

Es gibt zwei Möglichkeiten, einen Alias zu erstellen. Die bevorzugte Methode besteht darin, dem Parameter im Befehl `CREATE FUNCTION` einen Namen zu geben, zum Beispiel:

```sql
CREATE FUNCTION sales_tax(subtotal real) RETURNS real AS $$
BEGIN
     RETURN subtotal * 0.06;
END;
$$ LANGUAGE plpgsql;
```

Die andere Möglichkeit besteht darin, einen Alias explizit mit der Deklarationssyntax zu deklarieren:

```text
name ALIAS FOR $n;
```

Dasselbe Beispiel sieht in diesem Stil so aus:

```sql
CREATE FUNCTION sales_tax(real) RETURNS real AS $$
DECLARE
     subtotal ALIAS FOR $1;
BEGIN
     RETURN subtotal * 0.06;
END;
$$ LANGUAGE plpgsql;
```

> Hinweis: Diese beiden Beispiele sind nicht vollständig äquivalent. Im ersten Fall könnte auf `subtotal` als `sales_tax.subtotal` verwiesen werden, im zweiten Fall nicht. Hätten wir dem inneren Block ein Label gegeben, könnte `subtotal` stattdessen mit diesem Label qualifiziert werden.

Einige weitere Beispiele:

```sql
CREATE FUNCTION instr(varchar, integer) RETURNS integer AS $$
DECLARE
     v_string ALIAS FOR $1;
     index ALIAS FOR $2;
BEGIN
     -- some computations using v_string and index here
END;
$$ LANGUAGE plpgsql;

CREATE FUNCTION concat_selected_fields(in_t sometablename) RETURNS
 text AS $$
BEGIN
     RETURN in_t.f1 || in_t.f3 || in_t.f5 || in_t.f7;
END;
$$ LANGUAGE plpgsql;
```

Wenn eine PL/pgSQL-Funktion mit Ausgabeparametern deklariert wird, erhalten die Ausgabeparameter `$n`-Namen und optionale Aliasse auf genau dieselbe Weise wie normale Eingabeparameter. Ein Ausgabeparameter ist faktisch eine Variable, die mit `NULL` beginnt; ihr sollte während der Ausführung der Funktion ein Wert zugewiesen werden. Der endgültige Wert des Parameters ist der Rückgabewert. Das Beispiel zur Umsatzsteuer könnte zum Beispiel auch so geschrieben werden:

```sql
CREATE FUNCTION sales_tax(subtotal real, OUT tax real) AS $$
BEGIN
     tax := subtotal * 0.06;
END;
$$ LANGUAGE plpgsql;
```

Beachten Sie, dass wir `RETURNS real` weggelassen haben. Wir hätten es angeben können, es wäre aber redundant.

Um eine Funktion mit `OUT`-Parametern aufzurufen, lassen Sie den oder die Ausgabeparameter im Funktionsaufruf weg:

```sql
SELECT sales_tax(100.00);
```

Ausgabeparameter sind besonders nützlich, wenn mehrere Werte zurückgegeben werden sollen. Ein triviales Beispiel ist:

```sql
CREATE FUNCTION sum_n_product(x int, y int, OUT sum int, OUT prod
 int) AS $$
BEGIN
     sum := x + y;
     prod := x * y;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM sum_n_product(2, 4);
```

```text
 sum | prod
-----+------
   6 |    8
```

Wie in Abschnitt 36.5.4 besprochen, erzeugt dies effektiv einen anonymen Record-Typ für die Ergebnisse der Funktion. Wenn eine `RETURNS`-Klausel angegeben wird, muss sie `RETURNS record` lauten.

Das funktioniert auch mit Prozeduren, zum Beispiel:

```sql
CREATE PROCEDURE sum_n_product(x int, y int, OUT sum int, OUT prod
 int) AS $$
BEGIN
     sum := x + y;
     prod := x * y;
END;
$$ LANGUAGE plpgsql;
```

Bei einem Aufruf einer Prozedur müssen alle Parameter angegeben werden. Für Ausgabeparameter kann beim Aufruf der Prozedur aus einfachem SQL `NULL` angegeben werden:

```sql
CALL sum_n_product(2, 4, NULL, NULL);
```

```text
 sum | prod
-----+------
   6 |    8
```

Wenn Sie eine Prozedur aus PL/pgSQL heraus aufrufen, sollten Sie für jeden Ausgabeparameter stattdessen eine Variable schreiben; die Variable erhält das Ergebnis des Aufrufs. Details finden Sie in Abschnitt 41.6.3.

Eine weitere Möglichkeit, eine PL/pgSQL-Funktion zu deklarieren, ist `RETURNS TABLE`, zum Beispiel:

```sql
CREATE FUNCTION extended_sales(p_itemno int)
RETURNS TABLE(quantity int, total numeric) AS $$
BEGIN
     RETURN QUERY SELECT s.quantity, s.quantity * s.price FROM sales
 AS s
                  WHERE s.itemno = p_itemno;
END;
$$ LANGUAGE plpgsql;
```

Das ist genau äquivalent dazu, einen oder mehrere `OUT`-Parameter zu deklarieren und `RETURNS SETOF sometype` anzugeben.

Wenn der Rückgabetyp einer PL/pgSQL-Funktion als polymorpher Typ deklariert ist (siehe Abschnitt 36.2.5), wird ein besonderer Parameter `$0` erstellt. Sein Datentyp ist der tatsächliche Rückgabetyp der Funktion, wie er aus den tatsächlichen Eingabetypen abgeleitet wird. Dadurch kann die Funktion auf ihren tatsächlichen Rückgabetyp zugreifen, wie in Abschnitt 41.3.3 gezeigt. `$0` wird mit Null initialisiert und kann von der Funktion geändert werden; daher kann er bei Bedarf den Rückgabewert aufnehmen, obwohl das nicht erforderlich ist. `$0` kann auch einen Alias erhalten. Diese Funktion arbeitet zum Beispiel mit jedem Datentyp, der einen `+`-Operator besitzt:

```sql
CREATE FUNCTION add_three_values(v1 anyelement, v2 anyelement, v3
 anyelement)
RETURNS anyelement AS $$
DECLARE
     result ALIAS FOR $0;
BEGIN
     result := v1 + v2 + v3;
     RETURN result;
END;
$$ LANGUAGE plpgsql;
```

Derselbe Effekt kann erzielt werden, indem ein oder mehrere Ausgabeparameter als polymorphe Typen deklariert werden. In diesem Fall wird der besondere Parameter `$0` nicht verwendet; die Ausgabeparameter selbst erfüllen denselben Zweck. Beispiel:

```sql
CREATE FUNCTION add_three_values(v1 anyelement, v2 anyelement, v3
 anyelement,
                                 OUT sum anyelement)
AS $$
BEGIN
     sum := v1 + v2 + v3;
END;
$$ LANGUAGE plpgsql;
```

In der Praxis kann es nützlicher sein, eine polymorphe Funktion mit der Typfamilie `anycompatible` zu deklarieren, sodass die automatische Heraufstufung der Eingabeargumente auf einen gemeinsamen Typ erfolgt. Beispiel:

```sql
CREATE FUNCTION add_three_values(v1 anycompatible, v2
 anycompatible, v3 anycompatible)
RETURNS anycompatible AS $$
BEGIN
     RETURN v1 + v2 + v3;
END;
$$ LANGUAGE plpgsql;
```

Mit diesem Beispiel funktioniert ein Aufruf wie

```sql
SELECT add_three_values(1, 2, 4.7);
```

und stuft die `integer`-Eingaben automatisch auf `numeric` herauf. Die Funktion mit `anyelement` würde verlangen, dass Sie die drei Eingaben manuell auf denselben Typ casten.

### 41.3.2. ALIAS

```text
newname ALIAS FOR oldname;
```

Die `ALIAS`-Syntax ist allgemeiner, als der vorherige Abschnitt nahelegt: Sie können einen Alias für jede Variable deklarieren, nicht nur für Funktionsparameter. Der wichtigste praktische Einsatz besteht darin, Variablen mit vorgegebenen Namen, etwa `NEW` oder `OLD` innerhalb einer Triggerfunktion, einen anderen Namen zu geben.

Beispiele:

```sql
DECLARE
  prior ALIAS FOR old;
  updated ALIAS FOR new;
```

Da `ALIAS` zwei verschiedene Möglichkeiten erzeugt, dasselbe Objekt zu benennen, kann uneingeschränkte Verwendung verwirrend sein. Am besten verwenden Sie es nur dazu, vorgegebene Namen zu überschreiben.

### 41.3.3. Typen kopieren

```text
name table.column%TYPE
name variable%TYPE
```

`%TYPE` liefert den Datentyp einer Tabellenspalte oder einer zuvor deklarierten PL/pgSQL-Variablen. Sie können dies verwenden, um Variablen zu deklarieren, die Datenbankwerte aufnehmen sollen. Angenommen, Sie haben eine Spalte namens `user_id` in Ihrer Tabelle `users`. Um eine Variable mit demselben Datentyp wie `users.user_id` zu deklarieren, schreiben Sie:

```sql
user_id users.user_id%TYPE;
```

Es ist auch möglich, nach `%TYPE` Array-Dekoration zu schreiben und dadurch eine Variable zu erzeugen, die ein Array des referenzierten Typs enthält:

```sql
user_ids users.user_id%TYPE[];
user_ids users.user_id%TYPE ARRAY[4];                        -- equivalent to the above
```

Wie bei der Deklaration von Tabellenspalten als Arrays spielt es keine Rolle, ob Sie mehrere Klammerpaare oder konkrete Array-Dimensionen schreiben: PostgreSQL behandelt alle Arrays eines gegebenen Elementtyps unabhängig von ihrer Dimensionalität als denselben Typ. Siehe Abschnitt 8.15.1.

Durch die Verwendung von `%TYPE` müssen Sie den Datentyp der Struktur, auf die Sie sich beziehen, nicht kennen. Noch wichtiger ist: Wenn sich der Datentyp des referenzierten Elements später ändert, zum Beispiel wenn Sie den Typ von `user_id` von `integer` in `real` ändern, müssen Sie Ihre Funktionsdefinition möglicherweise nicht ändern.

`%TYPE` ist besonders wertvoll in polymorphen Funktionen, weil sich die für interne Variablen benötigten Datentypen von Aufruf zu Aufruf ändern können. Geeignete Variablen können erstellt werden, indem `%TYPE` auf die Argumente der Funktion oder auf Ergebnisplatzhalter angewendet wird.

### 41.3.4. Zeilentypen

```text
name table_name%ROWTYPE;
name composite_type_name;
```

Eine Variable eines zusammengesetzten Typs wird Zeilenvariable (oder Row-Type-Variable) genannt. Eine solche Variable kann eine ganze Zeile eines `SELECT`- oder `FOR`-Abfrageergebnisses aufnehmen, solange die Spaltenmenge dieser Abfrage zum deklarierten Typ der Variablen passt. Auf die einzelnen Felder des Zeilenwerts wird mit der üblichen Punktnotation zugegriffen, zum Beispiel `rowvar.field`.

Eine Zeilenvariable kann mit derselben Struktur wie die Zeilen einer vorhandenen Tabelle oder View deklariert werden, indem die Schreibweise `table_name%ROWTYPE` verwendet wird; oder sie kann deklariert werden, indem der Name eines zusammengesetzten Typs angegeben wird. Da jede Tabelle in PostgreSQL einen zugehörigen zusammengesetzten Typ desselben Namens besitzt, spielt es tatsächlich keine Rolle, ob Sie `%ROWTYPE` schreiben oder nicht. Die Form mit `%ROWTYPE` ist jedoch portabler.

Wie bei `%TYPE` kann auf `%ROWTYPE` Array-Dekoration folgen, um eine Variable zu deklarieren, die ein Array des referenzierten zusammengesetzten Typs enthält.

Parameter einer Funktion können zusammengesetzte Typen sein, also vollständige Tabellenzeilen. In diesem Fall ist der entsprechende Bezeichner `$n` eine Zeilenvariable, und Felder können daraus ausgewählt werden, zum Beispiel `$1.user_id`.

Hier ist ein Beispiel für die Verwendung zusammengesetzter Typen. `table1` und `table2` sind vorhandene Tabellen, die mindestens die genannten Felder besitzen:

```sql
CREATE FUNCTION merge_fields(t_row table1) RETURNS text AS $$
DECLARE
     t2_row table2%ROWTYPE;
BEGIN
     SELECT * INTO t2_row FROM table2 WHERE ... ;
     RETURN t_row.f1 || t2_row.f3 || t_row.f5 || t2_row.f7;
END;
$$ LANGUAGE plpgsql;

SELECT merge_fields(t.*) FROM table1 t WHERE ... ;
```

### 41.3.5. Record-Typen

```text
name RECORD;
```

Record-Variablen ähneln Zeilentyp-Variablen, besitzen aber keine vordefinierte Struktur. Sie nehmen die tatsächliche Zeilenstruktur der Zeile an, die ihnen während eines `SELECT`- oder `FOR`-Befehls zugewiesen wird. Die Unterstruktur einer Record-Variablen kann sich bei jeder Zuweisung ändern. Eine Folge davon ist, dass eine Record-Variable keine Unterstruktur besitzt, bevor ihr erstmals etwas zugewiesen wurde; jeder Versuch, vorher auf ein Feld zuzugreifen, führt zu einem Laufzeitfehler.

Beachten Sie, dass `RECORD` kein echter Datentyp ist, sondern nur ein Platzhalter. Außerdem sollte man sich klarmachen, dass eine PL/pgSQL-Funktion, die mit Rückgabetyp `record` deklariert ist, nicht ganz dasselbe Konzept verwendet wie eine Record-Variable, auch wenn eine solche Funktion eine Record-Variable zum Aufnehmen ihres Ergebnisses verwenden könnte. In beiden Fällen ist die tatsächliche Zeilenstruktur unbekannt, wenn die Funktion geschrieben wird. Bei einer Funktion, die `record` zurückgibt, wird die tatsächliche Struktur jedoch beim Parsen der aufrufenden Abfrage bestimmt, während eine Record-Variable ihre Zeilenstruktur im laufenden Betrieb ändern kann.

### 41.3.6. Kollation von PL/pgSQL-Variablen

Wenn eine PL/pgSQL-Funktion einen oder mehrere Parameter kollationsfähiger Datentypen besitzt, wird für jeden Funktionsaufruf abhängig von den Kollationen, die den tatsächlichen Argumenten zugewiesen sind, eine Kollation bestimmt, wie in [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung) beschrieben. Wenn eine Kollation erfolgreich bestimmt wird, also keine Konflikte impliziter Kollationen zwischen den Argumenten bestehen, werden alle kollationsfähigen Parameter implizit so behandelt, als hätten sie diese Kollation. Das wirkt sich auf das Verhalten kollationssensitiver Operationen innerhalb der Funktion aus. Betrachten Sie zum Beispiel:

```sql
CREATE FUNCTION less_than(a text, b text) RETURNS boolean AS $$
BEGIN
     RETURN a < b;
END;
$$ LANGUAGE plpgsql;

SELECT less_than(text_field_1, text_field_2) FROM table1;
SELECT less_than(text_field_1, text_field_2 COLLATE "C") FROM
 table1;
```

Die erste Verwendung von `less_than` nutzt für den Vergleich die gemeinsame Kollation von `text_field_1` und `text_field_2`, während die zweite Verwendung die Kollation `C` nutzt.

Darüber hinaus wird die bestimmte Kollation auch als Kollation aller lokalen Variablen angenommen, die kollationsfähige Typen haben. Daher würde diese Funktion sich nicht anders verhalten, wenn sie folgendermaßen geschrieben wäre:

```sql
CREATE FUNCTION less_than(a text, b text) RETURNS boolean AS $$
DECLARE
     local_a text := a;
     local_b text := b;
BEGIN
     RETURN local_a < local_b;
END;
$$ LANGUAGE plpgsql;
```

Wenn es keine Parameter kollationsfähiger Datentypen gibt oder für sie keine gemeinsame Kollation bestimmt werden kann, verwenden Parameter und lokale Variablen die Standardkollation ihres Datentyps. Das ist normalerweise die Standardkollation der Datenbank, kann aber bei Variablen von Domänentypen abweichen.

Eine lokale Variable eines kollationsfähigen Datentyps kann mit einer anderen Kollation verbunden werden, indem die Option `COLLATE` in ihrer Deklaration angegeben wird, zum Beispiel:

```sql
DECLARE
    local_a text COLLATE "en_US";
```

Diese Option überschreibt die Kollation, die der Variablen nach den obigen Regeln sonst gegeben würde.

Natürlich können innerhalb einer Funktion auch explizite `COLLATE`-Klauseln geschrieben werden, wenn eine bestimmte Kollation für eine bestimmte Operation erzwungen werden soll. Beispiel:

```sql
CREATE FUNCTION less_than_c(a text, b text) RETURNS boolean AS $$
BEGIN
     RETURN a < b COLLATE "C";
END;
$$ LANGUAGE plpgsql;
```

Dies überschreibt die Kollationen, die mit den in dem Ausdruck verwendeten Tabellenspalten, Parametern oder lokalen Variablen verbunden sind, genau wie es in einem normalen SQL-Befehl geschehen würde.

## 41.4. Ausdrücke

Alle Ausdrücke, die in PL/pgSQL-Anweisungen verwendet werden, werden mit dem Haupt-SQL-Executor des Servers verarbeitet. Wenn Sie zum Beispiel eine PL/pgSQL-Anweisung wie

```sql
IF expression THEN ...
```

schreiben, wertet PL/pgSQL den Ausdruck aus, indem es eine Abfrage wie

```sql
SELECT expression
```

an die Haupt-SQL-Engine übergibt. Beim Bilden des `SELECT`-Befehls werden alle Vorkommen von PL/pgSQL-Variablennamen durch Abfrageparameter ersetzt, wie in Abschnitt 41.11.1 ausführlich beschrieben. Dadurch kann der Abfrageplan für das `SELECT` nur einmal vorbereitet und anschließend bei späteren Auswertungen mit anderen Variablenwerten wiederverwendet werden. Was bei der ersten Verwendung eines Ausdrucks tatsächlich geschieht, entspricht also im Wesentlichen einem `PREPARE`-Befehl. Wenn wir beispielsweise zwei `integer`-Variablen `x` und `y` deklariert haben und schreiben:

```sql
IF x < y THEN ...
```

dann entspricht das, was im Hintergrund geschieht, etwa:

```sql
PREPARE statement_name(integer, integer) AS SELECT $1 < $2;
```

Diese vorbereitete Anweisung wird dann bei jeder Ausführung der `IF`-Anweisung mit den aktuellen Werten der PL/pgSQL-Variablen als Parameterwerte ausgeführt. Normalerweise sind diese Details für PL/pgSQL-Benutzer nicht wichtig, sie sind aber hilfreich, wenn ein Problem diagnostiziert werden soll. Weitere Informationen finden Sie in Abschnitt 41.11.2.

Da ein Ausdruck in einen `SELECT`-Befehl umgewandelt wird, kann er dieselben Klauseln enthalten wie ein gewöhnliches `SELECT`, mit der Ausnahme, dass er keine `UNION`-, `INTERSECT`- oder `EXCEPT`-Klausel auf oberster Ebene enthalten darf. So könnte man zum Beispiel testen, ob eine Tabelle nicht leer ist, mit:

```sql
IF count(*) > 0 FROM my_table THEN ...
```

Der Ausdruck zwischen `IF` und `THEN` wird dabei so geparst, als wäre er `SELECT count(*) > 0 FROM my_table`. Das `SELECT` muss genau eine Spalte und höchstens eine Zeile erzeugen. Wenn es keine Zeilen erzeugt, wird das Ergebnis als `NULL` betrachtet.

## 41.5. Grundlegende Anweisungen

In diesem und den folgenden Abschnitten beschreiben wir alle Anweisungstypen, die PL/pgSQL ausdrücklich versteht. Alles, was nicht als einer dieser Anweisungstypen erkannt wird, gilt als SQL-Befehl und wird zur Ausführung an die Haupt-Datenbank-Engine gesendet, wie in Abschnitt 41.5.2 beschrieben.

### 41.5.1. Zuweisung

Eine Wertzuweisung an eine PL/pgSQL-Variable wird so geschrieben:

```text
variable { := | = } expression;
```

Wie zuvor erklärt, wird der Ausdruck in einer solchen Anweisung mit einem SQL-`SELECT`-Befehl ausgewertet, der an die Haupt-Datenbank-Engine gesendet wird. Der Ausdruck muss einen einzelnen Wert liefern, möglicherweise einen Zeilenwert, wenn die Variable eine Zeilen- oder Record-Variable ist. Die Zielvariable kann eine einfache Variable sein, optional mit einem Blocknamen qualifiziert, ein Feld eines Zeilen- oder Record-Ziels oder ein Element beziehungsweise Slice eines Array-Ziels. Das Gleichheitszeichen (`=`) kann anstelle des PL/SQL-kompatiblen `:=` verwendet werden.

Wenn der Datentyp des Ausdrucksergebnisses nicht zum Datentyp der Variablen passt, wird der Wert so erzwungen, als würde ein Assignment-Cast verwendet (siehe [Abschnitt 10.4](10_Typumwandlung.md#104-wertspeicherung)). Wenn für das beteiligte Datentyppaar kein Assignment-Cast bekannt ist, versucht der PL/pgSQL-Interpreter, den Ergebniswert textuell umzuwandeln, also durch Anwenden der Ausgabefunktion des Ergebnistyps und anschließend der Eingabefunktion des Variablentyps. Beachten Sie, dass dies zu Laufzeitfehlern führen kann, die von der Eingabefunktion erzeugt werden, wenn die Zeichenkettenform des Ergebniswerts für die Eingabefunktion nicht akzeptabel ist.

Beispiele:

```sql
tax := subtotal * 0.06;
my_record.user_id := 20;
my_array[j] := 20;
my_array[1:3] := array[1,2,3];
complex_array[n].realpart = 12.3;
```

### 41.5.2. SQL-Befehle ausführen

Im Allgemeinen kann jeder SQL-Befehl, der keine Zeilen zurückgibt, innerhalb einer PL/pgSQL-Funktion ausgeführt werden, indem man ihn einfach schreibt. Zum Beispiel könnten Sie eine Tabelle erstellen und füllen mit:

```sql
CREATE TABLE mytable (id int primary key, data text);
INSERT INTO mytable VALUES (1,'one'), (2,'two');
```

Wenn der Befehl Zeilen zurückgibt, zum Beispiel `SELECT` oder `INSERT`/`UPDATE`/`DELETE`/`MERGE` mit `RETURNING`, gibt es zwei Vorgehensweisen. Wenn der Befehl höchstens eine Zeile zurückgibt oder Sie nur an der ersten Ausgabezeile interessiert sind, schreiben Sie den Befehl wie gewohnt, fügen aber eine `INTO`-Klausel hinzu, um die Ausgabe zu erfassen, wie in Abschnitt 41.5.3 beschrieben. Um alle Ausgabezeilen zu verarbeiten, schreiben Sie den Befehl als Datenquelle für eine `FOR`-Schleife, wie in Abschnitt 41.6.6 beschrieben.

Normalerweise reicht es nicht aus, nur statisch definierte SQL-Befehle auszuführen. Typischerweise soll ein Befehl unterschiedliche Datenwerte verwenden oder sich sogar grundlegender ändern, etwa indem zu unterschiedlichen Zeiten unterschiedliche Tabellennamen verwendet werden. Auch hier gibt es je nach Situation zwei Vorgehensweisen.

PL/pgSQL-Variablenwerte können automatisch in optimierbare SQL-Befehle eingefügt werden. Dazu gehören `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE` sowie bestimmte Utility-Befehle, die einen dieser Befehle enthalten, etwa `EXPLAIN` und `CREATE TABLE ... AS SELECT`. In diesen Befehlen wird jeder PL/pgSQL-Variablenname, der im Befehlstext erscheint, durch einen Abfrageparameter ersetzt; der aktuelle Wert der Variablen wird dann zur Laufzeit als Parameterwert bereitgestellt. Das ist genau dieselbe Verarbeitung, die zuvor für Ausdrücke beschrieben wurde; Details finden Sie in Abschnitt 41.11.1.

Wenn ein optimierbarer SQL-Befehl auf diese Weise ausgeführt wird, kann PL/pgSQL den Ausführungsplan für den Befehl zwischenspeichern und wiederverwenden, wie in Abschnitt 41.11.2 beschrieben.

Nicht optimierbare SQL-Befehle, auch Utility-Befehle genannt, können keine Abfrageparameter akzeptieren. Daher funktioniert die automatische Ersetzung von PL/pgSQL-Variablen in solchen Befehlen nicht. Um nichtkonstanten Text in einen aus PL/pgSQL ausgeführten Utility-Befehl einzufügen, müssen Sie den Utility-Befehl als Zeichenkette aufbauen und dann mit `EXECUTE` ausführen, wie in Abschnitt 41.5.4 beschrieben.

`EXECUTE` muss auch verwendet werden, wenn Sie den Befehl auf andere Weise verändern möchten, als nur einen Datenwert bereitzustellen, zum Beispiel durch Ändern eines Tabellennamens.

Manchmal ist es nützlich, einen Ausdruck oder eine `SELECT`-Abfrage auszuwerten, das Ergebnis aber zu verwerfen, zum Beispiel beim Aufruf einer Funktion, die Seiteneffekte hat, aber keinen nützlichen Ergebniswert liefert. Verwenden Sie dafür in PL/pgSQL die Anweisung `PERFORM`:

```sql
PERFORM query;
```

Dies führt `query` aus und verwirft das Ergebnis. Schreiben Sie die Abfrage genauso, wie Sie einen SQL-`SELECT`-Befehl schreiben würden, ersetzen Sie aber das einleitende Schlüsselwort `SELECT` durch `PERFORM`. Für `WITH`-Abfragen verwenden Sie `PERFORM` und setzen die Abfrage anschließend in Klammern. In diesem Fall darf die Abfrage nur eine Zeile zurückgeben. PL/pgSQL-Variablen werden in der Abfrage genauso ersetzt wie oben beschrieben, und der Plan wird auf dieselbe Weise zwischengespeichert. Außerdem wird die besondere Variable `FOUND` auf true gesetzt, wenn die Abfrage mindestens eine Zeile erzeugt hat, andernfalls auf false (siehe Abschnitt 41.5.5).

> Hinweis: Man könnte erwarten, dass das direkte Schreiben von `SELECT` dieses Ergebnis erreicht, aber derzeit ist `PERFORM` die einzige akzeptierte Methode dafür. Ein SQL-Befehl, der Zeilen zurückgeben kann, etwa `SELECT`, wird als Fehler abgewiesen, sofern er keine `INTO`-Klausel enthält, wie im nächsten Abschnitt beschrieben.

Ein Beispiel:

```sql
PERFORM create_mv('cs_session_page_requests_mv', my_query);
```

### 41.5.3. Einen Befehl mit einem Ein-Zeilen-Ergebnis ausführen

Das Ergebnis eines SQL-Befehls, der eine einzelne Zeile liefert, möglicherweise mit mehreren Spalten, kann einer Record-Variablen, einer Zeilentyp-Variablen oder einer Liste skalarer Variablen zugewiesen werden. Dazu schreibt man den eigentlichen SQL-Befehl und fügt eine `INTO`-Klausel hinzu. Beispiele:

```sql
SELECT select_expressions INTO [STRICT] target FROM ...;
INSERT ... RETURNING expressions INTO [STRICT] target;
UPDATE ... RETURNING expressions INTO [STRICT] target;
DELETE ... RETURNING expressions INTO [STRICT] target;
MERGE ... RETURNING expressions INTO [STRICT] target;
```

Dabei kann `target` eine Record-Variable, eine Zeilenvariable oder eine kommagetrennte Liste einfacher Variablen und Record-/Zeilenfelder sein. PL/pgSQL-Variablen werden im Rest des Befehls, also überall außer in der `INTO`-Klausel, genauso ersetzt wie oben beschrieben, und der Plan wird auf dieselbe Weise zwischengespeichert. Das funktioniert für `SELECT`, `INSERT`/`UPDATE`/`DELETE`/`MERGE` mit `RETURNING` sowie bestimmte Utility-Befehle, die Zeilenmengen zurückgeben, etwa `EXPLAIN`. Abgesehen von der `INTO`-Klausel ist der SQL-Befehl derselbe, wie er außerhalb von PL/pgSQL geschrieben würde.

> Tipp: Beachten Sie, dass diese Interpretation von `SELECT` mit `INTO` sich deutlich vom regulären PostgreSQL-Befehl `SELECT INTO` unterscheidet, bei dem das `INTO`-Ziel eine neu erstellte Tabelle ist. Wenn Sie innerhalb einer PL/pgSQL-Funktion aus einem `SELECT`-Ergebnis eine Tabelle erstellen möchten, verwenden Sie die Syntax `CREATE TABLE ... AS SELECT`.

Wenn eine Zeilenvariable oder eine Variablenliste als Ziel verwendet wird, müssen die Ergebnisspalten des Befehls in Anzahl und Datentypen exakt zur Struktur des Ziels passen, andernfalls tritt ein Laufzeitfehler auf. Wenn eine Record-Variable das Ziel ist, konfiguriert sie sich automatisch auf den Zeilentyp der Ergebnisspalten des Befehls.

Die `INTO`-Klausel kann fast überall im SQL-Befehl erscheinen. Üblicherweise wird sie bei einem `SELECT`-Befehl entweder direkt vor oder direkt nach der Liste der `select_expressions` geschrieben, bei anderen Befehlstypen am Ende des Befehls. Es wird empfohlen, dieser Konvention zu folgen, falls der PL/pgSQL-Parser in zukünftigen Versionen strenger wird.

Wenn `STRICT` in der `INTO`-Klausel nicht angegeben ist, wird `target` auf die erste vom Befehl zurückgegebene Zeile gesetzt oder auf Nullwerte, wenn der Befehl keine Zeilen zurückgegeben hat. Beachten Sie, dass „die erste Zeile“ nicht wohldefiniert ist, sofern Sie nicht `ORDER BY` verwendet haben. Alle Ergebniszeilen nach der ersten werden verworfen. Sie können die besondere Variable `FOUND` prüfen (siehe Abschnitt 41.5.5), um festzustellen, ob eine Zeile zurückgegeben wurde:

```sql
SELECT * INTO myrec FROM emp WHERE empname = myname;
IF NOT FOUND THEN
    RAISE EXCEPTION 'employee % not found', myname;
END IF;
```

Wenn die Option `STRICT` angegeben ist, muss der Befehl genau eine Zeile zurückgeben, andernfalls wird ein Laufzeitfehler gemeldet, entweder `NO_DATA_FOUND` (keine Zeilen) oder `TOO_MANY_ROWS` (mehr als eine Zeile). Sie können einen Exception-Block verwenden, wenn Sie den Fehler abfangen möchten, zum Beispiel:

```sql
BEGIN
     SELECT * INTO STRICT myrec FROM emp WHERE empname = myname;
     EXCEPTION
         WHEN NO_DATA_FOUND THEN
             RAISE EXCEPTION 'employee % not found', myname;
         WHEN TOO_MANY_ROWS THEN
             RAISE EXCEPTION 'employee % not unique', myname;
END;
```

Die erfolgreiche Ausführung eines Befehls mit `STRICT` setzt `FOUND` immer auf true.

Bei `INSERT`/`UPDATE`/`DELETE`/`MERGE` mit `RETURNING` meldet PL/pgSQL einen Fehler, wenn mehr als eine Zeile zurückgegeben wird, auch wenn `STRICT` nicht angegeben ist. Das liegt daran, dass es keine Option wie `ORDER BY` gibt, mit der bestimmt werden könnte, welche betroffene Zeile zurückgegeben werden soll.

Wenn `print_strict_params` für die Funktion aktiviert ist, enthält der `DETAIL`-Teil der Fehlermeldung Informationen über die an den Befehl übergebenen Parameter, wenn ein Fehler geworfen wird, weil die Anforderungen von `STRICT` nicht erfüllt sind. Sie können die Einstellung `print_strict_params` für alle Funktionen ändern, indem Sie `plpgsql.print_strict_params` setzen; betroffen sind jedoch nur nachfolgende Funktionskompilierungen. Sie können sie auch pro Funktion mit einer Compiler-Option aktivieren, zum Beispiel:

```sql
CREATE FUNCTION get_userid(username text) RETURNS int
AS $$
#print_strict_params on
DECLARE
userid int;
BEGIN
     SELECT users.userid INTO STRICT userid
         FROM users WHERE users.username = get_userid.username;
     RETURN userid;
END;
$$ LANGUAGE plpgsql;
```

Bei einem Fehlschlag könnte diese Funktion eine Fehlermeldung wie diese erzeugen:

```text
ERROR: query returned no rows
DETAIL: parameters: username = 'nosuchuser'
CONTEXT: PL/pgSQL function get_userid(text) line 6 at SQL
 statement
```

> Hinweis: Die Option `STRICT` entspricht dem Verhalten von `SELECT INTO` und verwandten Anweisungen in Oracle PL/SQL.

### 41.5.4. Dynamische Befehle ausführen

Häufig möchten Sie innerhalb Ihrer PL/pgSQL-Funktionen dynamische Befehle erzeugen, also Befehle, die bei jeder Ausführung unterschiedliche Tabellen oder unterschiedliche Datentypen betreffen. Die normalen Versuche von PL/pgSQL, Pläne für Befehle zwischenzuspeichern (wie in Abschnitt 41.11.2 besprochen), funktionieren in solchen Szenarien nicht. Für diese Art von Problem gibt es die Anweisung `EXECUTE`:

```text
EXECUTE command-string [ INTO [STRICT] target ] [ USING expression
    [, ... ] ];
```

Dabei ist `command-string` ein Ausdruck, der eine Zeichenkette vom Typ `text` liefert, die den auszuführenden Befehl enthält. Das optionale `target` ist eine Record-Variable, eine Zeilenvariable oder eine kommagetrennte Liste einfacher Variablen und Record-/Zeilenfelder, in denen die Ergebnisse des Befehls gespeichert werden. Die optionalen `USING`-Ausdrücke liefern Werte, die in den Befehl eingefügt werden sollen.

Auf der berechneten Befehlszeichenkette findet keine Ersetzung von PL/pgSQL-Variablen statt. Alle benötigten Variablenwerte müssen beim Aufbau der Befehlszeichenkette eingefügt werden; alternativ können Sie Parameter verwenden, wie unten beschrieben.

Außerdem gibt es kein Plan-Caching für Befehle, die mit `EXECUTE` ausgeführt werden. Stattdessen wird der Befehl jedes Mal neu geplant, wenn die Anweisung ausgeführt wird. Daher kann die Befehlszeichenkette innerhalb der Funktion dynamisch erstellt werden, um Aktionen auf unterschiedlichen Tabellen und Spalten auszuführen.

Die `INTO`-Klausel gibt an, wohin die Ergebnisse eines SQL-Befehls, der Zeilen zurückgibt, zugewiesen werden sollen. Wenn eine Zeilenvariable oder Variablenliste angegeben ist, muss sie exakt zur Struktur der Befehlsergebnisse passen; wenn eine Record-Variable angegeben ist, konfiguriert sie sich automatisch passend zur Ergebnisstruktur. Wenn mehrere Zeilen zurückgegeben werden, wird nur die erste den `INTO`-Variablen zugewiesen. Wenn keine Zeilen zurückgegeben werden, wird den `INTO`-Variablen `NULL` zugewiesen. Wenn keine `INTO`-Klausel angegeben ist, werden die Befehlsergebnisse verworfen.

Wenn die Option `STRICT` angegeben ist, wird ein Fehler gemeldet, sofern der Befehl nicht genau eine Zeile erzeugt.

Die Befehlszeichenkette kann Parameterwerte verwenden, auf die im Befehl als `$1`, `$2` usw. verwiesen wird. Diese Symbole beziehen sich auf Werte, die in der `USING`-Klausel bereitgestellt werden. Diese Methode ist oft vorzuziehen, statt Datenwerte als Text in die Befehlszeichenkette einzufügen: Sie vermeidet Laufzeitaufwand für die Umwandlung der Werte in Text und zurück und ist deutlich weniger anfällig für SQL-Injection-Angriffe, weil kein Quoting oder Escaping nötig ist. Ein Beispiel:

```sql
EXECUTE 'SELECT count(*) FROM mytable WHERE inserted_by = $1 AND
 inserted <= $2'
   INTO c
   USING checked_user, checked_date;
```

Beachten Sie, dass Parametersymbole nur für Datenwerte verwendet werden können. Wenn Sie dynamisch bestimmte Tabellen- oder Spaltennamen verwenden möchten, müssen Sie sie textuell in die Befehlszeichenkette einfügen. Wenn die vorherige Abfrage beispielsweise gegen eine dynamisch ausgewählte Tabelle ausgeführt werden müsste, könnten Sie Folgendes schreiben:

```sql
EXECUTE 'SELECT count(*) FROM '
    || quote_ident(tabname)
    || ' WHERE inserted_by = $1 AND inserted <= $2'
   INTO c
   USING checked_user, checked_date;
```

Ein saubererer Ansatz besteht darin, die Spezifikation `%I` von `format()` zu verwenden, um Tabellen- oder Spaltennamen automatisch gequotet einzufügen:

```sql
EXECUTE format('SELECT count(*) FROM %I '
   'WHERE inserted_by = $1 AND inserted <= $2', tabname)
   INTO c
   USING checked_user, checked_date;
```

Dieses Beispiel stützt sich auf die SQL-Regel, dass Zeichenkettenliterale, die durch einen Zeilenumbruch getrennt sind, implizit verkettet werden.

Eine weitere Einschränkung für Parametersymbole ist, dass sie nur in optimierbaren SQL-Befehlen funktionieren (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE` und bestimmte Befehle, die einen dieser Befehle enthalten). In anderen Anweisungstypen, allgemein Utility-Anweisungen genannt, müssen Sie Werte textuell einfügen, selbst wenn es nur Datenwerte sind.

Ein `EXECUTE` mit einer einfachen konstanten Befehlszeichenkette und einigen `USING`-Parametern, wie im ersten Beispiel oben, ist funktional äquivalent dazu, den Befehl direkt in PL/pgSQL zu schreiben und die Ersetzung von PL/pgSQL-Variablen automatisch geschehen zu lassen. Der wichtige Unterschied besteht darin, dass `EXECUTE` den Befehl bei jeder Ausführung neu plant und dabei einen Plan erzeugt, der spezifisch für die aktuellen Parameterwerte ist; PL/pgSQL kann andernfalls einen generischen Plan erzeugen und zur Wiederverwendung zwischenspeichern. In Situationen, in denen der beste Plan stark von den Parameterwerten abhängt, kann `EXECUTE` hilfreich sein, um sicherzustellen, dass kein generischer Plan gewählt wird.

`SELECT INTO` wird innerhalb von `EXECUTE` derzeit nicht unterstützt; führen Sie stattdessen einen einfachen `SELECT`-Befehl aus und geben Sie `INTO` als Teil von `EXECUTE` selbst an.

> Hinweis: Die PL/pgSQL-Anweisung `EXECUTE` ist nicht mit der SQL-Anweisung `EXECUTE` verwandt, die vom PostgreSQL-Server unterstützt wird. Die `EXECUTE`-Anweisung des Servers kann nicht direkt innerhalb von PL/pgSQL-Funktionen verwendet werden und wird dort auch nicht benötigt.

#### Beispiel 41.1. Werte in dynamischen Abfragen quoten

Beim Arbeiten mit dynamischen Befehlen müssen Sie häufig das Escaping einfacher Anführungszeichen behandeln. Die empfohlene Methode zum Quoten von festem Text im Funktionsrumpf ist Dollar-Quoting. Wenn Sie alten Code haben, der kein Dollar-Quoting verwendet, lesen Sie die Übersicht in Abschnitt 41.12.1; sie kann beim Überführen dieses Codes in ein vernünftigeres Schema Arbeit sparen.

Dynamische Werte müssen sorgfältig behandelt werden, weil sie Quote-Zeichen enthalten können. Ein Beispiel mit `format()`; dabei wird angenommen, dass Sie den Funktionsrumpf mit Dollar-Quoting schreiben, sodass Anführungszeichen nicht verdoppelt werden müssen:

```sql
EXECUTE format('UPDATE tbl SET %I = $1 '
   'WHERE key = $2', colname) USING newvalue, keyvalue;
```

Es ist auch möglich, die Quoting-Funktionen direkt aufzurufen:

```sql
EXECUTE 'UPDATE tbl SET '
        || quote_ident(colname)
        || ' = '
        || quote_literal(newvalue)
        || ' WHERE key = '
        || quote_literal(keyvalue);
```

Dieses Beispiel demonstriert die Verwendung der Funktionen `quote_ident` und `quote_literal` (siehe [Abschnitt 9.4](09_Funktionen_und_Operatoren.md#94-zeichenkettenfunktionen-und-operatoren)). Aus Sicherheitsgründen sollten Ausdrücke, die Spalten- oder Tabellenbezeichner enthalten, vor dem Einfügen in eine dynamische Abfrage durch `quote_ident` geleitet werden. Ausdrücke, die Werte enthalten, die in dem erzeugten Befehl als Zeichenkettenliterale erscheinen sollen, sollten durch `quote_literal` geleitet werden. Diese Funktionen treffen die passenden Maßnahmen, um den Eingabetext in doppelten beziehungsweise einfachen Anführungszeichen zurückzugeben und eingebettete Sonderzeichen korrekt zu escapen.

Da `quote_literal` als `STRICT` markiert ist, gibt es immer Null zurück, wenn es mit einem Nullargument aufgerufen wird. Im obigen Beispiel würde, falls `newvalue` oder `keyvalue` null wäre, die gesamte dynamische Abfragezeichenkette null werden, was zu einem Fehler von `EXECUTE` führt. Sie können dieses Problem vermeiden, indem Sie die Funktion `quote_nullable` verwenden. Sie funktioniert wie `quote_literal`, gibt bei einem Nullargument aber die Zeichenkette `NULL` zurück. Beispiel:

```sql
EXECUTE 'UPDATE tbl SET '
        || quote_ident(colname)
        || ' = '
        || quote_nullable(newvalue)
        || ' WHERE key = '
        || quote_nullable(keyvalue);
```

Wenn Sie mit Werten umgehen, die null sein könnten, sollten Sie normalerweise `quote_nullable` anstelle von `quote_literal` verwenden.

Wie immer muss darauf geachtet werden, dass Nullwerte in einer Abfrage keine unbeabsichtigten Ergebnisse liefern. Zum Beispiel wird die `WHERE`-Klausel

```sql
'WHERE key = ' || quote_nullable(keyvalue)
```

niemals erfolgreich sein, wenn `keyvalue` null ist, weil das Ergebnis der Verwendung des Gleichheitsoperators `=` mit einem Nulloperanden immer null ist. Wenn null wie ein gewöhnlicher Schlüsselwert funktionieren soll, müssten Sie das obige Beispiel so umschreiben:

```sql
'WHERE key IS NOT DISTINCT FROM ' || quote_nullable(keyvalue)
```

Derzeit wird `IS NOT DISTINCT FROM` deutlich weniger effizient behandelt als `=`, tun Sie dies also nur, wenn Sie es wirklich müssen. Weitere Informationen zu Nullwerten und `IS DISTINCT` finden Sie in [Abschnitt 9.2](09_Funktionen_und_Operatoren.md#92-vergleichsfunktionen-und-operatoren).

Beachten Sie, dass Dollar-Quoting nur zum Quoten von festem Text nützlich ist. Es wäre eine sehr schlechte Idee, dieses Beispiel so zu schreiben:

```sql
EXECUTE 'UPDATE tbl SET '
        || quote_ident(colname)
        || ' = $$'
        || newvalue
        || '$$ WHERE key = '
        || quote_literal(keyvalue);
```

Das würde scheitern, wenn der Inhalt von `newvalue` zufällig `$$` enthält. Derselbe Einwand gilt für jeden anderen Dollar-Quoting-Begrenzer, den Sie wählen könnten. Um Text sicher zu quoten, der nicht im Voraus bekannt ist, müssen Sie daher je nach Bedarf `quote_literal`, `quote_nullable` oder `quote_ident` verwenden.

Dynamische SQL-Anweisungen können auch sicher mit der Funktion `format` konstruiert werden (siehe Abschnitt 9.4.1). Beispiel:

```sql
EXECUTE format('UPDATE tbl SET %I = %L '
   'WHERE key = %L', colname, newvalue, keyvalue);
```

`%I` ist äquivalent zu `quote_ident`, und `%L` ist äquivalent zu `quote_nullable`. Die Funktion `format` kann zusammen mit der `USING`-Klausel verwendet werden:

```sql
EXECUTE format('UPDATE tbl SET %I = $1 WHERE key = $2', colname)
   USING newvalue, keyvalue;
```

Diese Form ist besser, weil die Variablen in ihrem nativen Datentypformat behandelt werden, statt sie bedingungslos in Text umzuwandeln und über `%L` zu quoten. Sie ist außerdem effizienter.

Ein viel größeres Beispiel für einen dynamischen Befehl und `EXECUTE` ist in Beispiel 41.10 zu sehen, das einen `CREATE FUNCTION`-Befehl aufbaut und ausführt, um eine neue Funktion zu definieren.

### 41.5.5. Ergebnisstatus ermitteln

Es gibt mehrere Möglichkeiten, die Wirkung eines Befehls zu bestimmen. Die erste Methode ist die Verwendung des Befehls `GET DIAGNOSTICS`, der folgende Form hat:

```text
GET [ CURRENT ] DIAGNOSTICS variable { = | := } item [ , ... ];
```

Dieser Befehl erlaubt das Abrufen von Systemstatusindikatoren. `CURRENT` ist ein Füllwort; siehe aber auch `GET STACKED DIAGNOSTICS` in Abschnitt 41.6.8.1. Jedes `item` ist ein Schlüsselwort, das einen Statuswert identifiziert, der der angegebenen Variablen zugewiesen werden soll; diese sollte den passenden Datentyp besitzen, um ihn aufzunehmen. Die derzeit verfügbaren Statuseinträge sind in Tabelle 41.1 gezeigt. Das Kolon-Gleich-Zeichen (`:=`) kann anstelle des SQL-standardisierten Tokens `=` verwendet werden. Ein Beispiel:

```sql
GET DIAGNOSTICS integer_var = ROW_COUNT;
```

Tabelle 41.1. Verfügbare Diagnoseeinträge

| Name | Typ | Beschreibung |
| --- | --- | --- |
| `ROW_COUNT` | `bigint` | Anzahl der Zeilen, die vom zuletzt ausgeführten SQL-Befehl verarbeitet wurden |
| `PG_CONTEXT` | `text` | Textzeile(n), die den aktuellen Aufruf-Stack beschreiben (siehe Abschnitt 41.6.9) |
| `PG_ROUTINE_OID` | `oid` | OID der aktuellen Funktion |

Die zweite Methode, die Wirkung eines Befehls zu bestimmen, besteht darin, die besondere Variable `FOUND` zu prüfen, die vom Typ `boolean` ist. `FOUND` beginnt innerhalb jedes PL/pgSQL-Funktionsaufrufs mit false. Sie wird von folgenden Anweisungstypen gesetzt:

- Eine `SELECT INTO`-Anweisung setzt `FOUND` auf true, wenn eine Zeile zugewiesen wird, und auf false, wenn keine Zeile zurückgegeben wird.

- Eine `PERFORM`-Anweisung setzt `FOUND` auf true, wenn sie eine oder mehrere Zeilen erzeugt und verwirft, und auf false, wenn keine Zeile erzeugt wird.

- `UPDATE`-, `INSERT`-, `DELETE`- und `MERGE`-Anweisungen setzen `FOUND` auf true, wenn mindestens eine Zeile betroffen ist, andernfalls auf false.

- Eine `FETCH`-Anweisung setzt `FOUND` auf true, wenn sie eine Zeile zurückgibt, andernfalls auf false.

- Eine `MOVE`-Anweisung setzt `FOUND` auf true, wenn sie den Cursor erfolgreich neu positioniert, andernfalls auf false.

- Eine `FOR`- oder `FOREACH`-Anweisung setzt `FOUND` auf true, wenn sie einmal oder öfter iteriert, andernfalls auf false. `FOUND` wird auf diese Weise gesetzt, wenn die Schleife beendet wird; während der Ausführung der Schleife wird `FOUND` durch die Schleifenanweisung nicht verändert, kann aber durch die Ausführung anderer Anweisungen im Schleifenrumpf geändert werden.

- `RETURN QUERY`- und `RETURN QUERY EXECUTE`-Anweisungen setzen `FOUND` auf true, wenn die Abfrage mindestens eine Zeile zurückgibt, andernfalls auf false.

Andere PL/pgSQL-Anweisungen verändern den Zustand von `FOUND` nicht. Beachten Sie insbesondere, dass `EXECUTE` die Ausgabe von `GET DIAGNOSTICS` verändert, aber nicht `FOUND`.

`FOUND` ist eine lokale Variable innerhalb jeder PL/pgSQL-Funktion; Änderungen daran betreffen nur die aktuelle Funktion.

### 41.5.6. Gar nichts tun

Manchmal ist eine Platzhalteranweisung nützlich, die nichts tut. Zum Beispiel kann sie anzeigen, dass ein Zweig einer `if`/`then`/`else`-Kette absichtlich leer ist. Verwenden Sie dafür die Anweisung `NULL`:

```sql
NULL;
```

Die folgenden beiden Codefragmente sind zum Beispiel äquivalent:

```sql
BEGIN
     y := x / 0;
EXCEPTION
     WHEN division_by_zero THEN
         NULL; -- ignore the error
END;

BEGIN
     y := x / 0;
EXCEPTION
     WHEN division_by_zero THEN                    -- ignore the error
END;
```

Welche Form vorzuziehen ist, ist Geschmackssache.

> Hinweis: In Oracles PL/SQL sind leere Anweisungslisten nicht erlaubt, daher sind `NULL`-Anweisungen in solchen Situationen erforderlich. PL/pgSQL erlaubt es dagegen, einfach nichts zu schreiben.

## 41.6. Kontrollstrukturen

Kontrollstrukturen sind wahrscheinlich der nützlichste und wichtigste Teil von PL/pgSQL. Mit den Kontrollstrukturen von PL/pgSQL können Sie PostgreSQL-Daten sehr flexibel und leistungsfähig bearbeiten.

### 41.6.1. Aus einer Funktion zurückkehren

Es gibt zwei Befehle, mit denen Daten aus einer Funktion zurückgegeben werden können: `RETURN` und `RETURN NEXT`.

#### 41.6.1.1. RETURN

```text
RETURN expression;
```

`RETURN` mit einem Ausdruck beendet die Funktion und gibt den Wert von `expression` an den Aufrufer zurück. Diese Form wird für PL/pgSQL-Funktionen verwendet, die keine Menge zurückgeben.

In einer Funktion, die einen skalaren Typ zurückgibt, wird das Ergebnis des Ausdrucks automatisch in den Rückgabetyp der Funktion gecastet, wie es für Zuweisungen beschrieben wurde. Um jedoch einen zusammengesetzten Wert (Zeilenwert) zurückzugeben, müssen Sie einen Ausdruck schreiben, der exakt die angeforderte Spaltenmenge liefert. Dafür kann explizites Casting nötig sein.

Wenn Sie die Funktion mit Ausgabeparametern deklariert haben, schreiben Sie einfach `RETURN` ohne Ausdruck. Die aktuellen Werte der Ausgabeparameter-Variablen werden zurückgegeben.

Wenn Sie die Funktion so deklariert haben, dass sie `void` zurückgibt, kann eine `RETURN`-Anweisung verwendet werden, um die Funktion vorzeitig zu verlassen; schreiben Sie dann aber keinen Ausdruck nach `RETURN`.

Der Rückgabewert einer Funktion darf nicht undefiniert bleiben. Wenn die Steuerung das Ende des obersten Blocks der Funktion erreicht, ohne auf eine `RETURN`-Anweisung zu treffen, tritt ein Laufzeitfehler auf. Diese Einschränkung gilt jedoch nicht für Funktionen mit Ausgabeparametern und Funktionen, die `void` zurückgeben. In diesen Fällen wird automatisch eine `RETURN`-Anweisung ausgeführt, wenn der oberste Block endet.

Einige Beispiele:

```sql
-- functions returning a scalar type
RETURN 1 + 2;
RETURN scalar_var;

-- functions returning a composite type
RETURN composite_type_var;
RETURN (1, 2, 'three'::text); -- must cast columns to correct
 types
```

#### 41.6.1.2. RETURN NEXT und RETURN QUERY

```text
RETURN NEXT expression;
RETURN QUERY query;
RETURN QUERY EXECUTE command-string [ USING expression [, ... ] ];
```

Wenn eine PL/pgSQL-Funktion so deklariert ist, dass sie `SETOF sometype` zurückgibt, ist das Vorgehen etwas anders. In diesem Fall werden die einzelnen zurückzugebenden Elemente durch eine Folge von `RETURN NEXT`- oder `RETURN QUERY`-Befehlen angegeben; anschließend wird ein abschließender `RETURN`-Befehl ohne Argument verwendet, um anzuzeigen, dass die Ausführung der Funktion beendet ist. `RETURN NEXT` kann sowohl mit skalaren als auch mit zusammengesetzten Datentypen verwendet werden; bei einem zusammengesetzten Ergebnistyp wird eine ganze „Tabelle“ von Ergebnissen zurückgegeben. `RETURN QUERY` hängt die Ergebnisse der Ausführung einer Abfrage an die Ergebnismenge der Funktion an. `RETURN NEXT` und `RETURN QUERY` können in einer einzelnen mengenrückgebenden Funktion frei gemischt werden; ihre Ergebnisse werden dann verkettet.

`RETURN NEXT` und `RETURN QUERY` kehren nicht tatsächlich aus der Funktion zurück, sondern hängen lediglich null oder mehr Zeilen an die Ergebnismenge der Funktion an. Die Ausführung fährt danach mit der nächsten Anweisung in der PL/pgSQL-Funktion fort. Während nacheinander `RETURN NEXT`- oder `RETURN QUERY`-Befehle ausgeführt werden, wird die Ergebnismenge aufgebaut. Ein abschließendes `RETURN`, das kein Argument haben sollte, beendet die Steuerung der Funktion; alternativ können Sie die Steuerung einfach das Ende der Funktion erreichen lassen.

`RETURN QUERY` besitzt die Variante `RETURN QUERY EXECUTE`, die die auszuführende Abfrage dynamisch angibt. Parameterausdrücke können über `USING` in die berechnete Abfragezeichenkette eingefügt werden, genau wie beim Befehl `EXECUTE`.

Wenn Sie die Funktion mit Ausgabeparametern deklariert haben, schreiben Sie einfach `RETURN NEXT` ohne Ausdruck. Bei jeder Ausführung werden die aktuellen Werte der Ausgabeparameter-Variablen für die spätere Rückgabe als Zeile des Ergebnisses gespeichert. Beachten Sie, dass Sie die Funktion als `SETOF record` deklarieren müssen, wenn es mehrere Ausgabeparameter gibt, oder als `SETOF sometype`, wenn es nur einen Ausgabeparameter vom Typ `sometype` gibt, um eine mengenrückgebende Funktion mit Ausgabeparametern zu erstellen.

Hier ist ein Beispiel für eine Funktion, die `RETURN NEXT` verwendet:

```sql
CREATE TABLE foo (fooid INT, foosubid INT, fooname TEXT);

INSERT INTO foo VALUES (1, 2, 'three');
INSERT INTO foo VALUES (4, 5, 'six');

CREATE OR REPLACE FUNCTION get_all_foo() RETURNS SETOF foo AS
$BODY$
DECLARE
     r foo%rowtype;
BEGIN
     FOR r IN
          SELECT * FROM foo WHERE fooid > 0
     LOOP
          -- can do some processing here
          RETURN NEXT r; -- return current row of SELECT
     END LOOP;
     RETURN;
END;
$BODY$
LANGUAGE plpgsql;

SELECT * FROM get_all_foo();
```

Hier ist ein Beispiel für eine Funktion, die `RETURN QUERY` verwendet:

```sql
CREATE FUNCTION get_available_flightid(date) RETURNS SETOF integer
 AS
$BODY$
BEGIN
    RETURN QUERY SELECT flightid
                   FROM flight
                  WHERE flightdate >= $1
                    AND flightdate < ($1 + 1);

    -- Since execution is not finished, we can check whether rows
 were returned
    -- and raise exception if not.
    IF NOT FOUND THEN
        RAISE EXCEPTION 'No flight at %.', $1;
    END IF;

    RETURN;
 END;
$BODY$
LANGUAGE plpgsql;

-- Returns available flights or raises exception if there are no
-- available flights.
SELECT * FROM get_available_flightid(CURRENT_DATE);
```

> Hinweis: Die aktuelle Implementierung von `RETURN NEXT` und `RETURN QUERY` speichert die gesamte Ergebnismenge, bevor sie aus der Funktion zurückkehrt, wie oben besprochen. Das bedeutet: Wenn eine PL/pgSQL-Funktion eine sehr große Ergebnismenge erzeugt, kann die Leistung schlecht sein. Daten werden auf die Festplatte geschrieben, um Speichermangel zu vermeiden, aber die Funktion selbst kehrt erst zurück, wenn die gesamte Ergebnismenge erzeugt wurde. Eine zukünftige Version von PL/pgSQL könnte Benutzern erlauben, mengenrückgebende Funktionen zu definieren, die diese Einschränkung nicht haben. Derzeit wird der Punkt, an dem Daten auf die Festplatte geschrieben werden, durch die Konfigurationsvariable `work_mem` gesteuert. Administratoren, die genügend Speicher haben, um größere Ergebnismengen im Arbeitsspeicher zu halten, sollten erwägen, diesen Parameter zu erhöhen.

### 41.6.2. Aus einer Prozedur zurückkehren

Eine Prozedur hat keinen Rückgabewert. Eine Prozedur kann daher ohne `RETURN`-Anweisung enden. Wenn Sie eine `RETURN`-Anweisung verwenden möchten, um den Code vorzeitig zu verlassen, schreiben Sie einfach `RETURN` ohne Ausdruck.

Wenn die Prozedur Ausgabeparameter besitzt, werden die endgültigen Werte der Ausgabeparameter-Variablen an den Aufrufer zurückgegeben.

### 41.6.3. Eine Prozedur aufrufen

Eine PL/pgSQL-Funktion, -Prozedur oder ein `DO`-Block kann mit `CALL` eine Prozedur aufrufen. Ausgabeparameter werden anders behandelt als bei der Funktionsweise von `CALL` in einfachem SQL. Jeder `OUT`- oder `INOUT`-Parameter der Prozedur muss einer Variablen in der `CALL`-Anweisung entsprechen, und was immer die Prozedur zurückgibt, wird nach ihrer Rückkehr dieser Variablen zugewiesen. Beispiel:

```sql
CREATE PROCEDURE triple(INOUT x int)
LANGUAGE plpgsql
AS $$
BEGIN
     x := x * 3;
END;
$$;

DO $$
DECLARE myvar int := 5;
BEGIN
  CALL triple(myvar);
  RAISE NOTICE 'myvar = %', myvar;                   -- prints 15
END;
$$;
```

Die Variable, die einem Ausgabeparameter entspricht, kann eine einfache Variable oder ein Feld einer Variablen zusammengesetzten Typs sein. Derzeit kann sie kein Element eines Arrays sein.

### 41.6.4. Bedingungen

Mit `IF`- und `CASE`-Anweisungen können Sie abhängig von bestimmten Bedingungen alternative Befehle ausführen. PL/pgSQL kennt drei Formen von `IF`:

- `IF ... THEN ... END IF`

- `IF ... THEN ... ELSE ... END IF`

- `IF ... THEN ... ELSIF ... THEN ... ELSE ... END IF`

und zwei Formen von `CASE`:

- `CASE ... WHEN ... THEN ... ELSE ... END CASE`

- `CASE WHEN ... THEN ... ELSE ... END CASE`

#### 41.6.4.1. IF-THEN

```text
IF boolean-expression THEN
    statements
END IF;
```

`IF`-`THEN`-Anweisungen sind die einfachste Form von `IF`. Die Anweisungen zwischen `THEN` und `END IF` werden ausgeführt, wenn die Bedingung wahr ist. Andernfalls werden sie übersprungen.

Beispiel:

```sql
IF v_user_id <> 0 THEN
    UPDATE users SET email = v_email WHERE user_id = v_user_id;
END IF;
```

#### 41.6.4.2. IF-THEN-ELSE

```text
IF boolean-expression THEN
     statements
ELSE
     statements
END IF;
```

`IF`-`THEN`-`ELSE`-Anweisungen erweitern `IF`-`THEN`, indem sie erlauben, eine alternative Menge von Anweisungen anzugeben, die ausgeführt werden soll, wenn die Bedingung nicht wahr ist. Beachten Sie, dass dies auch den Fall einschließt, in dem die Bedingung zu `NULL` ausgewertet wird.

Beispiele:

```sql
IF parentid IS NULL OR parentid = ''
THEN
     RETURN fullname;
ELSE
     RETURN hp_true_filename(parentid) || '/' || fullname;
END IF;

IF v_count > 0 THEN
     INSERT INTO users_count (count) VALUES (v_count);
     RETURN 't';
ELSE
     RETURN 'f';
END IF;
```

#### 41.6.4.3. IF-THEN-ELSIF

```text
IF boolean-expression THEN
    statements
[ ELSIF boolean-expression THEN
    statements
[ ELSIF boolean-expression THEN
    statements
    ...
]
]
[ ELSE
    statements ]
END IF;
```

Manchmal gibt es mehr als nur zwei Alternativen. `IF`-`THEN`-`ELSIF` bietet eine bequeme Methode, mehrere Alternativen der Reihe nach zu prüfen. Die `IF`-Bedingungen werden nacheinander getestet, bis die erste gefunden wird, die wahr ist. Dann werden die zugehörigen Anweisungen ausgeführt, und anschließend geht die Steuerung zur nächsten Anweisung nach `END IF` über. Nachfolgende `IF`-Bedingungen werden nicht getestet. Wenn keine der `IF`-Bedingungen wahr ist, wird der `ELSE`-Block ausgeführt, falls vorhanden.

Hier ist ein Beispiel:

```sql
IF number = 0 THEN
     result := 'zero';
ELSIF number > 0 THEN
     result := 'positive';
ELSIF number < 0 THEN
     result := 'negative';
ELSE
     -- hmm, the only other possibility is that number is null
     result := 'NULL';
END IF;
```

Das Schlüsselwort `ELSIF` kann auch `ELSEIF` geschrieben werden.

Eine alternative Möglichkeit, dieselbe Aufgabe zu erfüllen, besteht darin, `IF`-`THEN`-`ELSE`-Anweisungen zu verschachteln, wie im folgenden Beispiel:

```sql
IF demo_row.sex = 'm' THEN
     pretty_sex := 'man';
ELSE
     IF demo_row.sex = 'f' THEN
         pretty_sex := 'woman';
     END IF;
END IF;
```

Diese Methode erfordert jedoch ein passendes `END IF` für jedes `IF` und ist daher bei vielen Alternativen deutlich umständlicher als die Verwendung von `ELSIF`.

#### 41.6.4.4. Einfaches CASE

```text
CASE search-expression
    WHEN expression [, expression [ ... ]] THEN
      statements
  [ WHEN expression [, expression [ ... ]] THEN
      statements
    ... ]
  [ ELSE
      statements ]
END CASE;
```

Die einfache Form von `CASE` bietet bedingte Ausführung auf Grundlage der Gleichheit von Operanden. Der `search-expression` wird einmal ausgewertet und nacheinander mit jedem Ausdruck in den `WHEN`-Klauseln verglichen. Wenn eine Übereinstimmung gefunden wird, werden die entsprechenden Anweisungen ausgeführt, und danach geht die Steuerung zur nächsten Anweisung nach `END CASE` über. Nachfolgende `WHEN`-Ausdrücke werden nicht ausgewertet. Wenn keine Übereinstimmung gefunden wird, werden die `ELSE`-Anweisungen ausgeführt; wenn `ELSE` jedoch nicht vorhanden ist, wird eine `CASE_NOT_FOUND`-Exception ausgelöst.

Hier ist ein einfaches Beispiel:

```sql
CASE x
    WHEN 1, 2 THEN
         msg := 'one or two';
    ELSE
         msg := 'other value than one or two';
END CASE;
```

#### 41.6.4.5. Gesuchtes CASE

```text
CASE
    WHEN boolean-expression THEN
      statements
  [ WHEN boolean-expression THEN
      statements
    ... ]
  [ ELSE
      statements ]
END CASE;
```

Die gesuchte Form von `CASE` bietet bedingte Ausführung auf Grundlage der Wahrheit boolescher Ausdrücke. Der `boolean-expression` jeder `WHEN`-Klausel wird der Reihe nach ausgewertet, bis einer gefunden wird, der true ergibt. Dann werden die entsprechenden Anweisungen ausgeführt, und danach geht die Steuerung zur nächsten Anweisung nach `END CASE` über. Nachfolgende `WHEN`-Ausdrücke werden nicht ausgewertet. Wenn kein wahres Ergebnis gefunden wird, werden die `ELSE`-Anweisungen ausgeführt; wenn `ELSE` jedoch nicht vorhanden ist, wird eine `CASE_NOT_FOUND`-Exception ausgelöst.

Hier ist ein Beispiel:

```sql
CASE
    WHEN x BETWEEN 0 AND 10 THEN
        msg := 'value is between zero and ten';
    WHEN x BETWEEN 11 AND 20 THEN
        msg := 'value is between eleven and twenty';
END CASE;
```

Diese Form von `CASE` ist vollständig äquivalent zu `IF`-`THEN`-`ELSIF`, abgesehen von der Regel, dass das Erreichen einer weggelassenen `ELSE`-Klausel zu einem Fehler führt, statt nichts zu tun.

### 41.6.5. Einfache Schleifen

Mit den Anweisungen `LOOP`, `EXIT`, `CONTINUE`, `WHILE`, `FOR` und `FOREACH` können Sie veranlassen, dass Ihre PL/pgSQL-Funktion eine Reihe von Befehlen wiederholt.

#### 41.6.5.1. LOOP

```text
[ <<label>> ]
LOOP
     statements
END LOOP [ label ];
```

`LOOP` definiert eine unbedingte Schleife, die unbegrenzt wiederholt wird, bis sie durch eine `EXIT`- oder `RETURN`-Anweisung beendet wird. Das optionale Label kann von `EXIT`- und `CONTINUE`-Anweisungen innerhalb verschachtelter Schleifen verwendet werden, um anzugeben, auf welche Schleife sich diese Anweisungen beziehen.

#### 41.6.5.2. EXIT

```text
EXIT [ label ] [ WHEN boolean-expression ];
```

Wenn kein Label angegeben ist, wird die innerste Schleife beendet, und als Nächstes wird die Anweisung nach `END LOOP` ausgeführt. Wenn `label` angegeben ist, muss es das Label der aktuellen oder einer äußeren Ebene einer verschachtelten Schleife oder eines Blocks sein. Dann wird die benannte Schleife beziehungsweise der benannte Block beendet, und die Steuerung fährt mit der Anweisung nach dem zugehörigen `END` der Schleife oder des Blocks fort.

Wenn `WHEN` angegeben ist, erfolgt der Schleifenausstieg nur, wenn `boolean-expression` wahr ist. Andernfalls geht die Steuerung zur Anweisung nach `EXIT` über.

`EXIT` kann mit allen Schleifentypen verwendet werden; es ist nicht auf unbedingte Schleifen beschränkt.

Wenn `EXIT` mit einem `BEGIN`-Block verwendet wird, übergibt es die Steuerung an die nächste Anweisung nach dem Ende des Blocks. Beachten Sie, dass hierfür ein Label verwendet werden muss; ein nicht gelabeltes `EXIT` wird nie als passend zu einem `BEGIN`-Block betrachtet. Dies ist eine Änderung gegenüber PostgreSQL-Versionen vor 8.4, die erlaubten, dass ein nicht gelabeltes `EXIT` zu einem `BEGIN`-Block passt.

Beispiele:

```sql
LOOP
    -- some computations
    IF count > 0 THEN
        EXIT; -- exit loop
    END IF;
END LOOP;

LOOP
    -- some computations
    EXIT WHEN count > 0;                 -- same result as previous example
END LOOP;

<<ablock>>
BEGIN
     -- some computations
     IF stocks > 100000 THEN
         EXIT ablock; -- causes exit from the BEGIN block
     END IF;
     -- computations here will be skipped when stocks > 100000
END;
```

#### 41.6.5.3. CONTINUE

```text
CONTINUE [ label ] [ WHEN boolean-expression ];
```

Wenn kein Label angegeben ist, beginnt die nächste Iteration der innersten Schleife. Das heißt, alle verbleibenden Anweisungen im Schleifenrumpf werden übersprungen, und die Steuerung kehrt zum Schleifensteuerungsausdruck zurück, sofern es einen gibt, um zu bestimmen, ob eine weitere Schleifeniteration nötig ist. Wenn `label` vorhanden ist, gibt es das Label der Schleife an, deren Ausführung fortgesetzt werden soll.

Wenn `WHEN` angegeben ist, beginnt die nächste Iteration der Schleife nur, wenn `boolean-expression` wahr ist. Andernfalls geht die Steuerung zur Anweisung nach `CONTINUE` über.

`CONTINUE` kann mit allen Schleifentypen verwendet werden; es ist nicht auf unbedingte Schleifen beschränkt.

Beispiele:

```sql
LOOP
     -- some computations
     EXIT WHEN count > 100;
     CONTINUE WHEN count < 50;
     -- some computations for count IN [50 .. 100]
END LOOP;
```

#### 41.6.5.4. WHILE

```text
[ <<label>> ]
WHILE boolean-expression LOOP
    statements
END LOOP [ label ];
```

Die `WHILE`-Anweisung wiederholt eine Folge von Anweisungen, solange `boolean-expression` zu true ausgewertet wird. Der Ausdruck wird unmittelbar vor jedem Eintritt in den Schleifenrumpf geprüft.

Zum Beispiel:

```sql
WHILE amount_owed > 0 AND gift_certificate_balance > 0 LOOP
    -- some computations here
END LOOP;

WHILE NOT done LOOP
    -- some computations here
END LOOP;
```

#### 41.6.5.5. FOR (Integer-Variante)

```text
[ <<label>> ]
FOR name IN [ REVERSE ] expression .. expression [ BY expression ]
 LOOP
    statements
END LOOP [ label ];
```

Diese Form von `FOR` erzeugt eine Schleife, die über einen Bereich ganzzahliger Werte iteriert. Der Variablenname wird automatisch als Typ `integer` definiert und existiert nur innerhalb der Schleife; jede vorhandene Definition desselben Variablennamens wird innerhalb der Schleife ignoriert. Die beiden Ausdrücke, die die untere und obere Grenze des Bereichs angeben, werden beim Eintritt in die Schleife einmal ausgewertet. Wenn die `BY`-Klausel nicht angegeben ist, beträgt die Iterationsschrittweite 1; andernfalls ist sie der in der `BY`-Klausel angegebene Wert, der ebenfalls einmal beim Schleifeneintritt ausgewertet wird. Wenn `REVERSE` angegeben ist, wird der Schrittwert nach jeder Iteration subtrahiert statt addiert.

Einige Beispiele für Integer-`FOR`-Schleifen:

```sql
FOR i IN 1..10 LOOP
    -- i will take on the values 1,2,3,4,5,6,7,8,9,10 within the
 loop
END LOOP;

FOR i IN REVERSE 10..1 LOOP
    -- i will take on the values 10,9,8,7,6,5,4,3,2,1 within the
 loop
END LOOP;

FOR i IN REVERSE 10..1 BY 2 LOOP
    -- i will take on the values 10,8,6,4,2 within the loop
END LOOP;
```

Wenn die untere Grenze größer als die obere Grenze ist oder im Fall von `REVERSE` kleiner, wird der Schleifenrumpf überhaupt nicht ausgeführt. Es wird kein Fehler ausgelöst.

Wenn der `FOR`-Schleife ein Label angehängt ist, kann die Integer-Schleifenvariable mit einem qualifizierten Namen unter Verwendung dieses Labels referenziert werden.

### 41.6.6. Über Abfrageergebnisse schleifen

Mit einer anderen Art von `FOR`-Schleife können Sie über die Ergebnisse einer Abfrage iterieren und diese Daten entsprechend bearbeiten. Die Syntax lautet:

```text
[ <<label>> ]
FOR target IN query LOOP
    statements
END LOOP [ label ];
```

Das Ziel ist eine Record-Variable, eine Zeilenvariable oder eine kommagetrennte Liste skalarer Variablen. Dem Ziel wird nacheinander jede Zeile zugewiesen, die aus der Abfrage hervorgeht, und der Schleifenrumpf wird für jede Zeile ausgeführt. Hier ist ein Beispiel:

```sql
CREATE FUNCTION refresh_mviews() RETURNS integer AS $$
DECLARE
    mviews RECORD;
BEGIN
    RAISE NOTICE 'Refreshing all materialized views...';

      FOR mviews IN
         SELECT n.nspname AS mv_schema,
                 c.relname AS mv_name,
                 pg_catalog.pg_get_userbyid(c.relowner) AS owner
            FROM pg_catalog.pg_class c
      LEFT JOIN pg_catalog.pg_namespace n ON (n.oid = c.relnamespace)
           WHERE c.relkind = 'm'
       ORDER BY 1
      LOOP

        -- Now "mviews" has one record with information about the
 materialized view

            RAISE NOTICE 'Refreshing materialized view %.% (owner:
 %)...',
                     quote_ident(mviews.mv_schema),
                     quote_ident(mviews.mv_name),
                     quote_ident(mviews.owner);
        EXECUTE format('REFRESH MATERIALIZED VIEW %I.%I',
 mviews.mv_schema, mviews.mv_name);
    END LOOP;

      RAISE NOTICE 'Done refreshing materialized views.';
      RETURN 1;
END;
$$ LANGUAGE plpgsql;
```

Wenn die Schleife durch eine `EXIT`-Anweisung beendet wird, ist der zuletzt zugewiesene Zeilenwert nach der Schleife weiterhin zugänglich.

Die in dieser Art von `FOR`-Anweisung verwendete Abfrage kann jeder SQL-Befehl sein, der Zeilen an den Aufrufer zurückgibt: `SELECT` ist der häufigste Fall, aber Sie können auch `INSERT`, `UPDATE`, `DELETE` oder `MERGE` mit einer `RETURNING`-Klausel verwenden. Einige Utility-Befehle wie `EXPLAIN` funktionieren ebenfalls.

PL/pgSQL-Variablen werden durch Abfrageparameter ersetzt, und der Abfrageplan wird zur möglichen Wiederverwendung zwischengespeichert, wie in Abschnitt 41.11.1 und Abschnitt 41.11.2 ausführlich besprochen.

Die Anweisung `FOR`-`IN`-`EXECUTE` ist eine weitere Möglichkeit, über Zeilen zu iterieren:

```text
[ <<label>> ]
FOR target IN EXECUTE text_expression [ USING expression [, ... ] ]
 LOOP
    statements
END LOOP [ label ];
```

Dies ähnelt der vorherigen Form, außer dass die Quellabfrage als Zeichenkettenausdruck angegeben wird, der bei jedem Eintritt in die `FOR`-Schleife ausgewertet und neu geplant wird. Dadurch kann der Programmierer zwischen der Geschwindigkeit einer vorgeplanten Abfrage und der Flexibilität einer dynamischen Abfrage wählen, genau wie bei einer einfachen `EXECUTE`-Anweisung. Wie bei `EXECUTE` können Parameterwerte über `USING` in den dynamischen Befehl eingefügt werden.

Eine weitere Möglichkeit, die Abfrage anzugeben, über deren Ergebnisse iteriert werden soll, besteht darin, sie als Cursor zu deklarieren. Dies wird in Abschnitt 41.7.4 beschrieben.

### 41.6.7. Über Arrays schleifen

Die `FOREACH`-Schleife ähnelt stark einer `FOR`-Schleife, iteriert aber nicht über die von einer SQL-Abfrage zurückgegebenen Zeilen, sondern über die Elemente eines Array-Werts. Allgemein ist `FOREACH` dafür gedacht, über Bestandteile eines zusammengesetzten Ausdrucks zu schleifen; Varianten für zusammengesetzte Werte außer Arrays könnten in Zukunft hinzukommen. Die `FOREACH`-Anweisung zum Schleifen über ein Array lautet:

```text
[ <<label>> ]
FOREACH target [ SLICE number ] IN ARRAY expression LOOP
    statements
END LOOP [ label ];
```

Ohne `SLICE`, oder wenn `SLICE 0` angegeben ist, iteriert die Schleife über einzelne Elemente des Arrays, das durch Auswerten des Ausdrucks erzeugt wird. Der Zielvariablen wird jedes Element der Reihe nach zugewiesen, und der Schleifenrumpf wird für jedes Element ausgeführt. Hier ist ein Beispiel für das Schleifen über die Elemente eines Integer-Arrays:

```sql
CREATE FUNCTION sum(int[]) RETURNS int8 AS $$
DECLARE
  s int8 := 0;
  x int;
BEGIN
  FOREACH x IN ARRAY $1
  LOOP
     s := s + x;
  END LOOP;
  RETURN s;
END;
$$ LANGUAGE plpgsql;
```

Die Elemente werden unabhängig von der Anzahl der Array-Dimensionen in Speicherreihenfolge besucht. Obwohl das Ziel normalerweise nur eine einzelne Variable ist, kann es beim Schleifen über ein Array zusammengesetzter Werte (Records) eine Liste von Variablen sein. In diesem Fall werden die Variablen für jedes Array-Element aus aufeinanderfolgenden Spalten des zusammengesetzten Werts zugewiesen.

Mit einem positiven `SLICE`-Wert iteriert `FOREACH` über Slices des Arrays statt über einzelne Elemente. Der `SLICE`-Wert muss eine Integer-Konstante sein, die nicht größer ist als die Anzahl der Dimensionen des Arrays. Die Zielvariable muss ein Array sein und erhält nacheinander Slices des Array-Werts, wobei jeder Slice die durch `SLICE` angegebene Anzahl von Dimensionen hat. Hier ist ein Beispiel für das Iterieren über eindimensionale Slices:

```sql
CREATE FUNCTION scan_rows(int[]) RETURNS void AS $$
DECLARE
  x int[];
BEGIN
  FOREACH x SLICE 1 IN ARRAY $1
  LOOP
     RAISE NOTICE 'row = %', x;
  END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT scan_rows(ARRAY[[1,2,3],[4,5,6],[7,8,9],[10,11,12]]);
```

```text
NOTICE:      row = {1,2,3}
NOTICE:      row = {4,5,6}
NOTICE:      row = {7,8,9}
NOTICE:      row = {10,11,12}
```

### 41.6.8. Fehler abfangen

Standardmäßig bricht jeder Fehler, der in einer PL/pgSQL-Funktion auftritt, die Ausführung der Funktion und der umgebenden Transaktion ab. Sie können Fehler abfangen und sich davon erholen, indem Sie einen `BEGIN`-Block mit einer `EXCEPTION`-Klausel verwenden. Die Syntax ist eine Erweiterung der normalen Syntax für einen `BEGIN`-Block:

```text
[ <<label>> ]
[ DECLARE
     declarations ]
BEGIN
     statements
EXCEPTION
     WHEN condition [ OR condition ... ] THEN
         handler_statements
     [ WHEN condition [ OR condition ... ] THEN
           handler_statements
       ... ]
END;
```

Wenn kein Fehler auftritt, führt diese Blockform einfach alle Anweisungen aus, und die Steuerung geht anschließend zur nächsten Anweisung nach `END` über. Wenn jedoch innerhalb der Anweisungen ein Fehler auftritt, wird die weitere Verarbeitung dieser Anweisungen abgebrochen, und die Steuerung geht zur `EXCEPTION`-Liste über. Die Liste wird nach der ersten Bedingung durchsucht, die zum aufgetretenen Fehler passt. Wenn eine Übereinstimmung gefunden wird, werden die entsprechenden `handler_statements` ausgeführt, und anschließend geht die Steuerung zur nächsten Anweisung nach `END` über. Wenn keine Übereinstimmung gefunden wird, wird der Fehler so weitergereicht, als gäbe es die `EXCEPTION`-Klausel überhaupt nicht: Der Fehler kann von einem umgebenden Block mit `EXCEPTION` abgefangen werden, oder, falls es keinen gibt, die Verarbeitung der Funktion abbrechen.

Die Bedingungsnamen können alle in Anhang A gezeigten Namen sein. Ein Kategoriename passt auf jeden Fehler innerhalb seiner Kategorie. Der besondere Bedingungsname `OTHERS` passt auf jeden Fehlertyp außer `QUERY_CANCELED` und `ASSERT_FAILURE`. Es ist möglich, aber oft unklug, diese beiden Fehlertypen namentlich abzufangen. Bedingungsnamen unterscheiden nicht zwischen Groß- und Kleinschreibung. Außerdem kann eine Fehlerbedingung durch einen `SQLSTATE`-Code angegeben werden; zum Beispiel sind diese beiden Formen äquivalent:

```sql
WHEN division_by_zero THEN ...
WHEN SQLSTATE '22012' THEN ...
```

Wenn innerhalb der ausgewählten `handler_statements` ein neuer Fehler auftritt, kann er von dieser `EXCEPTION`-Klausel nicht abgefangen werden, sondern wird weitergereicht. Eine umgebende `EXCEPTION`-Klausel könnte ihn abfangen.

Wenn ein Fehler von einer `EXCEPTION`-Klausel abgefangen wird, bleiben die lokalen Variablen der PL/pgSQL-Funktion so, wie sie zum Zeitpunkt des Fehlers waren, aber alle Änderungen am persistenten Datenbankzustand innerhalb des Blocks werden zurückgerollt. Betrachten Sie als Beispiel dieses Fragment:

```sql
INSERT INTO mytab(firstname, lastname) VALUES('Tom', 'Jones');
BEGIN
     UPDATE mytab SET firstname = 'Joe' WHERE lastname = 'Jones';
     x := x + 1;
     y := x / 0;
EXCEPTION
     WHEN division_by_zero THEN
         RAISE NOTICE 'caught division_by_zero';
         RETURN x;
END;
```

Wenn die Steuerung die Zuweisung an `y` erreicht, schlägt sie mit einem `division_by_zero`-Fehler fehl. Dieser wird von der `EXCEPTION`-Klausel abgefangen. Der in der `RETURN`-Anweisung zurückgegebene Wert ist der inkrementierte Wert von `x`, aber die Auswirkungen des `UPDATE`-Befehls wurden zurückgerollt. Der `INSERT`-Befehl vor dem Block wird jedoch nicht zurückgerollt, sodass das Endergebnis ist, dass die Datenbank Tom Jones enthält, nicht Joe Jones.

> Tipp: Ein Block mit einer `EXCEPTION`-Klausel ist beim Betreten und Verlassen deutlich teurer als ein Block ohne eine solche Klausel. Verwenden Sie `EXCEPTION` daher nicht ohne Not.

#### Beispiel 41.2. Exceptions mit UPDATE/INSERT

Dieses Beispiel verwendet Exception-Behandlung, um je nach Bedarf entweder `UPDATE` oder `INSERT` auszuführen. Es wird empfohlen, dass Anwendungen `INSERT` mit `ON CONFLICT DO UPDATE` verwenden, statt dieses Muster tatsächlich einzusetzen. Dieses Beispiel dient vor allem dazu, die Verwendung von PL/pgSQL-Kontrollflussstrukturen zu veranschaulichen:

```sql
CREATE TABLE db (a INT PRIMARY KEY, b TEXT);

CREATE FUNCTION merge_db(key INT, data TEXT) RETURNS VOID AS
$$
BEGIN
    LOOP
         -- first try to update the key
         UPDATE db SET b = data WHERE a = key;
         IF found THEN
             RETURN;
         END IF;
         -- not there, so try to insert the key
         -- if someone else inserts the same key concurrently,
         -- we could get a unique-key failure
         BEGIN
             INSERT INTO db(a,b) VALUES (key, data);
             RETURN;

         EXCEPTION WHEN unique_violation THEN
              -- Do nothing, and loop to try the UPDATE again.
         END;
    END LOOP;
END;
$$
LANGUAGE plpgsql;

SELECT merge_db(1, 'david');
SELECT merge_db(1, 'dennis');
```

Dieser Code nimmt an, dass der Fehler `unique_violation` vom `INSERT` verursacht wurde und nicht etwa von einem `INSERT` in einer Triggerfunktion auf der Tabelle. Er kann sich auch falsch verhalten, wenn es mehr als einen Unique-Index auf der Tabelle gibt, weil er die Operation unabhängig davon erneut versucht, welcher Index den Fehler verursacht hat. Mehr Sicherheit ließe sich erreichen, indem man die als Nächstes besprochenen Funktionen verwendet, um zu prüfen, ob der abgefangene Fehler der erwartete war.

#### 41.6.8.1. Informationen über einen Fehler ermitteln

Exception-Handler müssen häufig den konkreten aufgetretenen Fehler identifizieren. In PL/pgSQL gibt es zwei Möglichkeiten, Informationen über die aktuelle Exception zu erhalten: besondere Variablen und den Befehl `GET STACKED DIAGNOSTICS`.

Innerhalb eines Exception-Handlers enthält die besondere Variable `SQLSTATE` den Fehlercode, der der ausgelösten Exception entspricht; eine Liste möglicher Fehlercodes finden Sie in Tabelle A.1. Die besondere Variable `SQLERRM` enthält die Fehlermeldung, die mit der Exception verbunden ist. Außerhalb von Exception-Handlern sind diese Variablen undefiniert.

Innerhalb eines Exception-Handlers kann man Informationen über die aktuelle Exception außerdem mit dem Befehl `GET STACKED DIAGNOSTICS` abrufen, der folgende Form hat:

```text
GET STACKED DIAGNOSTICS variable { = | := } item [ , ... ];
```

Jedes `item` ist ein Schlüsselwort, das einen Statuswert identifiziert, der der angegebenen Variablen zugewiesen werden soll; diese sollte den passenden Datentyp haben, um ihn aufzunehmen. Die derzeit verfügbaren Statuseinträge sind in Tabelle 41.2 gezeigt.

Tabelle 41.2. Fehlerdiagnoseeinträge

| Name | Typ | Beschreibung |
| --- | --- | --- |
| `RETURNED_SQLSTATE` | `text` | Der `SQLSTATE`-Fehlercode der Exception |
| `COLUMN_NAME` | `text` | Der Name der Spalte, die mit der Exception zusammenhängt |
| `CONSTRAINT_NAME` | `text` | Der Name des Constraints, der mit der Exception zusammenhängt |
| `PG_DATATYPE_NAME` | `text` | Der Name des Datentyps, der mit der Exception zusammenhängt |
| `MESSAGE_TEXT` | `text` | Der Text der primären Exception-Meldung |
| `TABLE_NAME` | `text` | Der Name der Tabelle, die mit der Exception zusammenhängt |
| `SCHEMA_NAME` | `text` | Der Name des Schemas, das mit der Exception zusammenhängt |
| `PG_EXCEPTION_DETAIL` | `text` | Der Text der Detailmeldung der Exception, falls vorhanden |
| `PG_EXCEPTION_HINT` | `text` | Der Text des Hinweises der Exception, falls vorhanden |
| `PG_EXCEPTION_CONTEXT` | `text` | Textzeile(n), die den Aufruf-Stack zum Zeitpunkt der Exception beschreiben (siehe Abschnitt 41.6.9) |

Wenn die Exception für einen Eintrag keinen Wert gesetzt hat, wird eine leere Zeichenkette zurückgegeben.

Hier ist ein Beispiel:

```sql
DECLARE
  text_var1 text;
  text_var2 text;
  text_var3 text;
BEGIN
  -- some processing which might cause an exception
  ...
EXCEPTION WHEN OTHERS THEN
  GET STACKED DIAGNOSTICS text_var1 = MESSAGE_TEXT,
                           text_var2 = PG_EXCEPTION_DETAIL,
                           text_var3 = PG_EXCEPTION_HINT;
END;
```

### 41.6.9. Informationen über den Ausführungsort ermitteln

Der zuvor in Abschnitt 41.5.5 beschriebene Befehl `GET DIAGNOSTICS` ruft Informationen über den aktuellen Ausführungszustand ab, während der oben besprochene Befehl `GET STACKED DIAGNOSTICS` Informationen über den Ausführungszustand zum Zeitpunkt eines früheren Fehlers meldet. Sein Statuseintrag `PG_CONTEXT` ist nützlich, um den aktuellen Ausführungsort zu identifizieren. `PG_CONTEXT` gibt eine Textzeichenkette mit Textzeilen zurück, die den Aufruf-Stack beschreiben. Die erste Zeile bezieht sich auf die aktuelle Funktion und den gerade ausgeführten Befehl `GET DIAGNOSTICS`. Die zweite und alle weiteren Zeilen beziehen sich auf aufrufende Funktionen weiter oben im Aufruf-Stack. Beispiel:

```sql
CREATE OR REPLACE FUNCTION outer_func() RETURNS integer AS $$
BEGIN
  RETURN inner_func();
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION inner_func() RETURNS integer AS $$
DECLARE
  stack text;
BEGIN
  GET DIAGNOSTICS stack = PG_CONTEXT;
  RAISE NOTICE E'--- Call Stack ---\n%', stack;
  RETURN 1;
END;
$$ LANGUAGE plpgsql;

SELECT outer_func();
```

```text
NOTICE: --- Call Stack ---
PL/pgSQL function inner_func() line 5 at GET DIAGNOSTICS
PL/pgSQL function outer_func() line 3 at RETURN

CONTEXT: PL/pgSQL function outer_func() line 3 at RETURN
 outer_func
 ------------
(1 row)
```

`GET STACKED DIAGNOSTICS ... PG_EXCEPTION_CONTEXT` gibt dieselbe Art von Stacktrace zurück, beschreibt aber den Ort, an dem ein Fehler erkannt wurde, statt den aktuellen Ort.

## 41.7. Cursor

Statt eine ganze Abfrage auf einmal auszuführen, kann man einen Cursor einrichten, der die Abfrage kapselt, und das Abfrageergebnis dann jeweils einige Zeilen auf einmal lesen. Ein Grund dafür ist, Speicherüberläufe zu vermeiden, wenn das Ergebnis sehr viele Zeilen enthält. (PL/pgSQL-Benutzer müssen sich darum allerdings normalerweise nicht kümmern, weil `FOR`-Schleifen intern automatisch einen Cursor verwenden, um Speicherprobleme zu vermeiden.) Interessanter ist es, eine Referenz auf einen Cursor zurückzugeben, den eine Funktion erzeugt hat, sodass der Aufrufer die Zeilen lesen kann. Das ist eine effiziente Möglichkeit, große Zeilenmengen aus Funktionen zurückzugeben.

### 41.7.1. Cursor-Variablen deklarieren

Jeder Zugriff auf Cursor in PL/pgSQL erfolgt über Cursor-Variablen, die immer den speziellen Datentyp `refcursor` haben. Eine Möglichkeit, eine Cursor-Variable zu erzeugen, besteht einfach darin, sie als Variable des Typs `refcursor` zu deklarieren. Eine andere Möglichkeit ist die Cursor-Deklarationssyntax, die allgemein so aussieht:

```text
name [ [ NO ] SCROLL ] CURSOR [ ( arguments ) ] FOR query;
```

(`FOR` kann aus Gründen der Oracle-Kompatibilität durch `IS` ersetzt werden.) Wenn `SCROLL` angegeben ist, kann der Cursor rückwärts scrollen; wenn `NO SCROLL` angegeben ist, werden rückwärts gerichtete Abrufe zurückgewiesen; wenn keines von beiden angegeben ist, hängt es von der Abfrage ab, ob rückwärts gerichtete Abrufe erlaubt sind. `arguments`, falls angegeben, ist eine kommagetrennte Liste von Paaren `name datatype`, die Namen definieren, die in der angegebenen Abfrage durch Parameterwerte ersetzt werden. Die tatsächlich einzusetzenden Werte für diese Namen werden später angegeben, wenn der Cursor geöffnet wird.

Einige Beispiele:

```sql
DECLARE
    curs1 refcursor;
    curs2 CURSOR FOR SELECT * FROM tenk1;
    curs3 CURSOR (key integer) FOR SELECT * FROM tenk1 WHERE unique1 = key;
```

Alle drei Variablen haben den Datentyp `refcursor`, aber die erste kann mit jeder beliebigen Abfrage verwendet werden, während an die zweite bereits eine vollständig angegebene Abfrage gebunden ist und an die letzte eine parametrisierte Abfrage. (`key` wird beim Öffnen des Cursors durch einen ganzzahligen Parameterwert ersetzt.) Die Variable `curs1` wird als ungebunden bezeichnet, weil sie nicht an eine bestimmte Abfrage gebunden ist.

Die Option `SCROLL` kann nicht verwendet werden, wenn die Abfrage des Cursors `FOR UPDATE`/`SHARE` verwendet. Außerdem ist es am besten, `NO SCROLL` mit einer Abfrage zu verwenden, die volatile Funktionen enthält. Die Implementierung von `SCROLL` nimmt an, dass erneutes Lesen der Abfrageausgabe konsistente Ergebnisse liefert, was bei einer volatilen Funktion möglicherweise nicht der Fall ist.

### 41.7.2. Cursor öffnen

Bevor ein Cursor zum Abrufen von Zeilen verwendet werden kann, muss er geöffnet werden. (Das entspricht dem SQL-Befehl `DECLARE CURSOR`.) PL/pgSQL hat drei Formen der `OPEN`-Anweisung; zwei davon verwenden ungebundene Cursor-Variablen, die dritte verwendet eine gebundene Cursor-Variable.

> Gebundene Cursor-Variablen können auch ohne explizites Öffnen des Cursors verwendet werden, und zwar über die in [Abschnitt 41.7.4](#4174-über-das-ergebnis-eines-cursors-schleifen) beschriebene `FOR`-Anweisung. Eine `FOR`-Schleife öffnet den Cursor und schließt ihn wieder, wenn die Schleife beendet ist.

Das Öffnen eines Cursors erzeugt eine serverinterne Datenstruktur namens Portal, die den Ausführungszustand der Cursor-Abfrage hält. Ein Portal hat einen Namen, der während der Lebensdauer des Portals innerhalb der Sitzung eindeutig sein muss. Standardmäßig weist PL/pgSQL jedem erzeugten Portal einen eindeutigen Namen zu. Wenn Sie einer Cursor-Variable jedoch einen nicht-null Zeichenkettenwert zuweisen, wird diese Zeichenkette als Portalname verwendet. Diese Funktion kann wie in [Abschnitt 41.7.3.5](#41735-cursor-zurückgeben) beschrieben genutzt werden.

#### 41.7.2.1. OPEN FOR query

```text
OPEN unbound_cursorvar [ [ NO ] SCROLL ] FOR query;
```

Die Cursor-Variable wird geöffnet und bekommt die angegebene Abfrage zur Ausführung zugewiesen. Der Cursor darf noch nicht geöffnet sein, und er muss als ungebundene Cursor-Variable deklariert worden sein (also als einfache `refcursor`-Variable). Die Abfrage muss ein `SELECT` sein oder etwas anderes, das Zeilen zurückgibt (etwa `EXPLAIN`). Die Abfrage wird genauso behandelt wie andere SQL-Befehle in PL/pgSQL: Namen von PL/pgSQL-Variablen werden ersetzt, und der Abfrageplan wird für mögliche Wiederverwendung zwischengespeichert. Wenn eine PL/pgSQL-Variable in die Cursor-Abfrage eingesetzt wird, ist der eingesetzte Wert derjenige, den sie zum Zeitpunkt von `OPEN` hat; spätere Änderungen an der Variablen beeinflussen das Verhalten des Cursors nicht. Die Optionen `SCROLL` und `NO SCROLL` haben dieselbe Bedeutung wie bei einem gebundenen Cursor.

Ein Beispiel:

```sql
OPEN curs1 FOR SELECT * FROM foo WHERE key = mykey;
```

#### 41.7.2.2. OPEN FOR EXECUTE

```text
OPEN unbound_cursorvar [ [ NO ] SCROLL ] FOR EXECUTE query_string
                                     [ USING expression [, ... ] ];
```

Die Cursor-Variable wird geöffnet und bekommt die angegebene Abfrage zur Ausführung zugewiesen. Der Cursor darf noch nicht geöffnet sein, und er muss als ungebundene Cursor-Variable deklariert worden sein (also als einfache `refcursor`-Variable). Die Abfrage wird als Zeichenkettenausdruck angegeben, genauso wie beim Befehl `EXECUTE`. Wie üblich bietet das Flexibilität, sodass der Abfrageplan von einem Lauf zum nächsten variieren kann (siehe [Abschnitt 41.11.2](#41112-plancaching)), und es bedeutet außerdem, dass auf der Befehlszeichenkette keine Variablenersetzung durchgeführt wird. Wie bei `EXECUTE` können Parameterwerte über `format()` und `USING` in den dynamischen Befehl eingefügt werden. Die Optionen `SCROLL` und `NO SCROLL` haben dieselbe Bedeutung wie bei einem gebundenen Cursor.

Ein Beispiel:

```sql
OPEN curs1 FOR EXECUTE format('SELECT * FROM %I WHERE col1 =
 $1',tabname) USING keyvalue;
```

In diesem Beispiel wird der Tabellenname über `format()` in die Abfrage eingefügt. Der Vergleichswert für `col1` wird über einen `USING`-Parameter eingefügt, sodass er nicht gequotet werden muss.

#### 41.7.2.3. Gebundenen Cursor öffnen

```text
OPEN bound_cursorvar [ ( [ argument_name { := |
 => } ] argument_value [, ...] ) ];
```

Diese Form von `OPEN` wird verwendet, um eine Cursor-Variable zu öffnen, deren Abfrage bei der Deklaration an sie gebunden wurde. Der Cursor darf noch nicht geöffnet sein. Eine Liste tatsächlicher Argumentwert-Ausdrücke muss genau dann erscheinen, wenn der Cursor mit Argumenten deklariert wurde. Diese Werte werden in die Abfrage eingesetzt.

Der Abfrageplan für einen gebundenen Cursor gilt immer als cachebar; es gibt in diesem Fall kein Gegenstück zu `EXECUTE`. Beachten Sie, dass `SCROLL` und `NO SCROLL` in `OPEN` nicht angegeben werden können, da das Scroll-Verhalten des Cursors bereits festgelegt wurde.

Argumentwerte können entweder in Positions- oder in Namensschreibweise übergeben werden. In Positionsschreibweise werden alle Argumente der Reihe nach angegeben. In Namensschreibweise wird der Name jedes Arguments mit `:=` oder `=>` angegeben, um ihn vom Argumentausdruck zu trennen. Ähnlich wie beim Aufruf von Funktionen, der in [Abschnitt 4.3](04_SQL_Syntax.md#43-funktionsaufrufe) beschrieben ist, ist es auch erlaubt, Positions- und Namensschreibweise zu mischen.

Beispiele (sie verwenden die obigen Beispiele für Cursor-Deklarationen):

```sql
OPEN curs2;
OPEN curs3(42);
OPEN curs3(key := 42);
OPEN curs3(key => 42);
```

Da bei der Abfrage eines gebundenen Cursors Variablenersetzung durchgeführt wird, gibt es eigentlich zwei Möglichkeiten, Werte an den Cursor zu übergeben: entweder mit einem expliziten Argument für `OPEN` oder implizit durch Verweis auf eine PL/pgSQL-Variable in der Abfrage. Es werden jedoch nur Variablen ersetzt, die vor der Deklaration des gebundenen Cursors deklariert wurden. In beiden Fällen wird der zu übergebende Wert zum Zeitpunkt von `OPEN` bestimmt. Eine andere Möglichkeit, denselben Effekt wie im obigen Beispiel `curs3` zu erzielen, ist zum Beispiel:

```sql
DECLARE
    key integer;
    curs4 CURSOR FOR SELECT * FROM tenk1 WHERE unique1 = key;
BEGIN
    key := 42;
    OPEN curs4;
```

### 41.7.3. Cursor verwenden

Nachdem ein Cursor geöffnet wurde, kann er mit den hier beschriebenen Anweisungen bearbeitet werden.

Diese Operationen müssen nicht in derselben Funktion stattfinden, die den Cursor ursprünglich geöffnet hat. Sie können einen `refcursor`-Wert aus einer Funktion zurückgeben und den Aufrufer mit dem Cursor arbeiten lassen. (Intern ist ein `refcursor`-Wert einfach der Zeichenkettenname des Portals, das die aktive Abfrage für den Cursor enthält. Dieser Name kann weitergereicht, anderen `refcursor`-Variablen zugewiesen und so weiter werden, ohne das Portal zu stören.)

Alle Portale werden am Ende einer Transaktion implizit geschlossen. Daher kann ein `refcursor`-Wert nur bis zum Ende der Transaktion verwendet werden, um auf einen geöffneten Cursor zu verweisen.

#### 41.7.3.1. FETCH

```text
FETCH [ direction { FROM | IN } ] cursor INTO target;
```

`FETCH` ruft die nächste Zeile (in der angegebenen Richtung) aus dem Cursor in ein Ziel ab. Dieses Ziel kann eine Zeilenvariable, eine Record-Variable oder eine kommagetrennte Liste einfacher Variablen sein, genau wie bei `SELECT INTO`. Wenn es keine passende Zeile gibt, wird das Ziel auf `NULL`-Werte gesetzt. Wie bei `SELECT INTO` kann die spezielle Variable `FOUND` geprüft werden, um festzustellen, ob eine Zeile erhalten wurde oder nicht. Wenn keine Zeile erhalten wird, wird der Cursor je nach Bewegungsrichtung hinter die letzte Zeile oder vor die erste Zeile positioniert.

Die Richtungsklausel kann jede Variante sein, die im SQL-Befehl `FETCH` erlaubt ist, außer den Varianten, die mehr als eine Zeile abrufen können; konkret kann sie `NEXT`, `PRIOR`, `FIRST`, `LAST`, `ABSOLUTE count`, `RELATIVE count`, `FORWARD` oder `BACKWARD` sein. Wird `direction` weggelassen, entspricht das der Angabe von `NEXT`. In den Formen mit einem Zähler kann `count` ein beliebiger ganzzahliger Ausdruck sein (anders als beim SQL-Befehl `FETCH`, der nur eine ganzzahlige Konstante erlaubt). `direction`-Werte, die eine Rückwärtsbewegung erfordern, schlagen wahrscheinlich fehl, sofern der Cursor nicht mit der Option `SCROLL` deklariert oder geöffnet wurde.

`cursor` muss der Name einer `refcursor`-Variablen sein, die auf ein geöffnetes Cursor-Portal verweist.

Beispiele:

```sql
FETCH curs1 INTO rowvar;
FETCH curs2 INTO foo, bar, baz;
FETCH LAST FROM curs3 INTO x, y;
FETCH RELATIVE -2 FROM curs4 INTO x;
```

#### 41.7.3.2. MOVE

```text
MOVE [ direction { FROM | IN } ] cursor;
```

`MOVE` positioniert einen Cursor neu, ohne Daten abzurufen. `MOVE` funktioniert wie der Befehl `FETCH`, außer dass es den Cursor nur neu positioniert und die Zeile, zu der bewegt wurde, nicht zurückgibt. Die Richtungsklausel kann jede Variante sein, die im SQL-Befehl `FETCH` erlaubt ist, einschließlich der Varianten, die mehr als eine Zeile abrufen können; der Cursor wird auf die letzte dieser Zeilen positioniert. (Der Fall, in dem die Richtungsklausel einfach ein Zählerausdruck ohne Schlüsselwort ist, gilt in PL/pgSQL jedoch als veraltet. Diese Syntax ist mehrdeutig mit dem Fall, in dem die Richtungsklausel ganz weggelassen wird, und kann deshalb fehlschlagen, wenn der Zähler keine Konstante ist.) Wie bei `SELECT INTO` kann die spezielle Variable `FOUND` geprüft werden, um festzustellen, ob es eine Zeile gab, zu der bewegt werden konnte. Wenn es keine solche Zeile gibt, wird der Cursor je nach Bewegungsrichtung hinter die letzte Zeile oder vor die erste Zeile positioniert.

Beispiele:

```sql
MOVE curs1;
MOVE LAST FROM curs3;
MOVE RELATIVE -2 FROM curs4;
MOVE FORWARD 2 FROM curs4;
```

#### 41.7.3.3. UPDATE/DELETE WHERE CURRENT OF

```sql
UPDATE table SET ... WHERE CURRENT OF cursor;
DELETE FROM table WHERE CURRENT OF cursor;
```

Wenn ein Cursor auf einer Tabellenzeile positioniert ist, kann diese Zeile aktualisiert oder gelöscht werden, indem der Cursor zur Identifikation der Zeile verwendet wird. Es gibt Einschränkungen dafür, wie die Abfrage des Cursors aussehen darf (insbesondere keine Gruppierung), und es ist am besten, im Cursor `FOR UPDATE` zu verwenden. Weitere Informationen finden Sie auf der Referenzseite zu `DECLARE`.

Ein Beispiel:

```sql
UPDATE foo SET dataval = myval WHERE CURRENT OF curs1;
```

#### 41.7.3.4. CLOSE

```text
CLOSE cursor;
```

`CLOSE` schließt das Portal, das einem geöffneten Cursor zugrunde liegt. Dies kann verwendet werden, um Ressourcen früher als am Transaktionsende freizugeben oder um die Cursor-Variable wieder für ein erneutes Öffnen freizumachen.

Ein Beispiel:

```sql
CLOSE curs1;
```

#### 41.7.3.5. Cursor zurückgeben

PL/pgSQL-Funktionen können Cursor an den Aufrufer zurückgeben. Das ist nützlich, um mehrere Zeilen oder Spalten zurückzugeben, insbesondere bei sehr großen Ergebnismengen. Dazu öffnet die Funktion den Cursor und gibt den Cursornamen an den Aufrufer zurück (oder öffnet den Cursor einfach mit einem Portalnamen, der vom Aufrufer angegeben wurde oder ihm anderweitig bekannt ist). Der Aufrufer kann dann Zeilen aus dem Cursor abrufen. Der Cursor kann vom Aufrufer geschlossen werden, oder er wird automatisch geschlossen, wenn die Transaktion endet.

Der Portalname, der für einen Cursor verwendet wird, kann vom Programmierer angegeben oder automatisch erzeugt werden. Um einen Portalnamen anzugeben, weisen Sie der `refcursor`-Variablen vor dem Öffnen einfach eine Zeichenkette zu. Der Zeichenkettenwert der `refcursor`-Variablen wird von `OPEN` als Name des zugrunde liegenden Portals verwendet. Wenn der Wert der `refcursor`-Variablen jedoch null ist (wie es standardmäßig der Fall ist), erzeugt `OPEN` automatisch einen Namen, der mit keinem bestehenden Portal kollidiert, und weist ihn der `refcursor`-Variablen zu.

> Vor PostgreSQL 16 wurden gebundene Cursor-Variablen so initialisiert, dass sie ihre eigenen Namen enthielten, statt null zu bleiben, sodass der zugrunde liegende Portalname standardmäßig derselbe war wie der Name der Cursor-Variablen. Dies wurde geändert, weil es ein zu großes Risiko von Konflikten zwischen ähnlich benannten Cursorn in verschiedenen Funktionen erzeugte.

Das folgende Beispiel zeigt eine Möglichkeit, wie der Aufrufer einen Cursornamen bereitstellen kann:

```sql
CREATE TABLE test (col text);
INSERT INTO test VALUES ('123');

CREATE FUNCTION reffunc(refcursor) RETURNS refcursor AS '
BEGIN
     OPEN $1 FOR SELECT col FROM test;
     RETURN $1;
END;
' LANGUAGE plpgsql;

BEGIN;
SELECT reffunc('funccursor');
FETCH ALL IN funccursor;
COMMIT;
```

Das folgende Beispiel verwendet die automatische Erzeugung von Cursornamen:

```sql
CREATE FUNCTION reffunc2() RETURNS refcursor AS '
DECLARE
    ref refcursor;
BEGIN
    OPEN ref FOR SELECT col FROM test;
    RETURN ref;
END;
' LANGUAGE plpgsql;

-- need to be in a transaction to use cursors.
BEGIN;
SELECT reffunc2();
```

```text
      reffunc2
--------------------
 <unnamed cursor 1>
(1 row)
```

```sql
FETCH ALL IN "<unnamed cursor 1>";
COMMIT;
```

Das folgende Beispiel zeigt eine Möglichkeit, mehrere Cursor aus einer einzelnen Funktion zurückzugeben:

```sql
CREATE FUNCTION myfunc(refcursor, refcursor) RETURNS SETOF refcursor AS $$
BEGIN
     OPEN $1 FOR SELECT * FROM table_1;
     RETURN NEXT $1;
     OPEN $2 FOR SELECT * FROM table_2;
     RETURN NEXT $2;
END;
$$ LANGUAGE plpgsql;

-- need to be in a transaction to use cursors.
BEGIN;

SELECT * FROM myfunc('a', 'b');

FETCH ALL FROM a;
FETCH ALL FROM b;
COMMIT;
```

### 41.7.4. Über das Ergebnis eines Cursors schleifen

Es gibt eine Variante der `FOR`-Anweisung, mit der über die von einem Cursor zurückgegebenen Zeilen iteriert werden kann. Die Syntax lautet:

```text
[ <<label>> ]
FOR recordvar IN bound_cursorvar [ ( [ argument_name { := |
 => } ] argument_value [, ...] ) ] LOOP
    statements
END LOOP [ label ];
```

Die Cursor-Variable muss bei ihrer Deklaration an eine Abfrage gebunden worden sein, und sie darf noch nicht geöffnet sein. Die `FOR`-Anweisung öffnet den Cursor automatisch und schließt ihn wieder, wenn die Schleife verlassen wird. Eine Liste tatsächlicher Argumentwert-Ausdrücke muss genau dann erscheinen, wenn der Cursor mit Argumenten deklariert wurde. Diese Werte werden auf dieselbe Weise wie bei einem `OPEN` in die Abfrage eingesetzt (siehe [Abschnitt 41.7.2.3](#41723-gebundenen-cursor-öffnen)).

Die Variable `recordvar` wird automatisch als Typ `record` definiert und existiert nur innerhalb der Schleife (eine bereits bestehende Definition des Variablennamens wird innerhalb der Schleife ignoriert). Jede vom Cursor zurückgegebene Zeile wird nacheinander dieser Record-Variablen zugewiesen, und der Schleifenrumpf wird ausgeführt.

## 41.8. Transaktionsverwaltung

In Prozeduren, die mit dem Befehl `CALL` aufgerufen werden, sowie in anonymen Codeblöcken (Befehl `DO`) ist es möglich, Transaktionen mit den Befehlen `COMMIT` und `ROLLBACK` zu beenden. Nachdem eine Transaktion mit diesen Befehlen beendet wurde, wird automatisch eine neue Transaktion gestartet, daher gibt es keinen separaten Befehl `START TRANSACTION`. (Beachten Sie, dass `BEGIN` und `END` in PL/pgSQL eine andere Bedeutung haben.)

Hier ist ein einfaches Beispiel:

```sql
CREATE PROCEDURE transaction_test1()
LANGUAGE plpgsql
AS $$
BEGIN
     FOR i IN 0..9 LOOP
         INSERT INTO test1 (a) VALUES (i);
         IF i % 2 = 0 THEN
              COMMIT;
         ELSE
              ROLLBACK;
         END IF;
     END LOOP;
END;
$$;

CALL transaction_test1();
```

Eine neue Transaktion beginnt mit Standard-Transaktionseigenschaften wie etwa der Transaktionsisolationsstufe. In Fällen, in denen Transaktionen in einer Schleife committet werden, kann es wünschenswert sein, neue Transaktionen automatisch mit denselben Eigenschaften wie die vorherige zu starten. Die Befehle `COMMIT AND CHAIN` und `ROLLBACK AND CHAIN` leisten dies.

Transaktionssteuerung ist nur in `CALL`- oder `DO`-Aufrufen von der obersten Ebene aus möglich oder in verschachtelten `CALL`- oder `DO`-Aufrufen, ohne dass dazwischen ein anderer Befehl liegt. Wenn der Aufrufstack zum Beispiel `CALL proc1()` -> `CALL proc2()` -> `CALL proc3()` lautet, können die zweite und dritte Prozedur Transaktionssteuerungsaktionen ausführen. Wenn der Aufrufstack aber `CALL proc1()` -> `SELECT func2()` -> `CALL proc3()` lautet, kann die letzte Prozedur keine Transaktionssteuerung ausführen, weil dazwischen das `SELECT` steht.

PL/pgSQL unterstützt keine Savepoints (Befehle `SAVEPOINT`/`ROLLBACK TO SAVEPOINT`/`RELEASE SAVEPOINT`). Typische Verwendungsmuster für Savepoints können durch Blöcke mit Exception-Handlern ersetzt werden (siehe [Abschnitt 41.6.8](#4168-fehler-abfangen)). Unter der Haube bildet ein Block mit Exception-Handlern eine Subtransaktion, was bedeutet, dass Transaktionen innerhalb eines solchen Blocks nicht beendet werden können.

Für Cursor-Schleifen gelten besondere Überlegungen. Betrachten Sie dieses Beispiel:

```sql
CREATE PROCEDURE transaction_test2()
LANGUAGE plpgsql
AS $$
DECLARE
     r RECORD;
BEGIN
     FOR r IN SELECT * FROM test2 ORDER BY x LOOP
         INSERT INTO test1 (a) VALUES (r.x);
         COMMIT;
     END LOOP;
END;
$$;

CALL transaction_test2();
```

Normalerweise werden Cursor beim Commit einer Transaktion automatisch geschlossen. Ein Cursor, der als Teil einer solchen Schleife erzeugt wird, wird durch das erste `COMMIT` oder `ROLLBACK` jedoch automatisch in einen holdable Cursor umgewandelt. Das bedeutet, dass der Cursor beim ersten `COMMIT` oder `ROLLBACK` vollständig ausgewertet wird, statt zeilenweise. Der Cursor wird nach der Schleife weiterhin automatisch entfernt, sodass dies für den Benutzer größtenteils unsichtbar ist. Man muss aber beachten, dass Tabellen- oder Zeilensperren, die durch die Abfrage des Cursors genommen wurden, nach dem ersten `COMMIT` oder `ROLLBACK` nicht mehr gehalten werden.

Transaktionsbefehle sind in Cursor-Schleifen nicht erlaubt, die von Befehlen angetrieben werden, die nicht read-only sind (zum Beispiel `UPDATE ... RETURNING`).

## 41.9. Fehler und Meldungen

### 41.9.1. Fehler und Meldungen ausgeben

Verwenden Sie die Anweisung `RAISE`, um Meldungen auszugeben und Fehler auszulösen.

```text
RAISE [ level ] 'format' [, expression [, ... ]] [ USING option { =
 | := } expression [, ... ] ];
RAISE [ level ] condition_name [ USING option { = | := } expression
 [, ... ] ];
RAISE [ level ] SQLSTATE 'sqlstate' [ USING option { =
 | := } expression [, ... ] ];
RAISE [ level ] USING option { = | := } expression [, ... ];
RAISE ;
```

Die Option `level` gibt den Fehlerschweregrad an. Erlaubte Stufen sind `DEBUG`, `LOG`, `INFO`, `NOTICE`, `WARNING` und `EXCEPTION`, wobei `EXCEPTION` die Vorgabe ist. `EXCEPTION` löst einen Fehler aus (der normalerweise die aktuelle Transaktion abbricht); die anderen Stufen erzeugen nur Meldungen unterschiedlicher Priorität. Ob Meldungen einer bestimmten Priorität an den Client gemeldet, in das Serverlog geschrieben oder beides werden, wird durch die Konfigurationsvariablen `log_min_messages` und `client_min_messages` gesteuert. Weitere Informationen finden Sie in [Kapitel 19](19_Serverkonfiguration.md).

In der ersten Syntaxvariante schreiben Sie nach der Stufe, falls eine angegeben ist, eine Formatzeichenkette (die ein einfaches Zeichenkettenliteral sein muss, kein Ausdruck). Die Formatzeichenkette gibt den zu meldenden Fehlermeldungstext an. Auf die Formatzeichenkette können optionale Argumentausdrücke folgen, die in die Meldung eingesetzt werden. Innerhalb der Formatzeichenkette wird `%` durch die Zeichenkettendarstellung des nächsten optionalen Argumentwerts ersetzt. Schreiben Sie `%%`, um ein literales `%` auszugeben. Die Anzahl der Argumente muss mit der Anzahl der `%`-Platzhalter in der Formatzeichenkette übereinstimmen, sonst wird beim Kompilieren der Funktion ein Fehler ausgelöst.

In diesem Beispiel ersetzt der Wert von `v_job_id` das `%` in der Zeichenkette:

```sql
RAISE NOTICE 'Calling cs_create_job(%)', v_job_id;
```

In der zweiten und dritten Syntaxvariante geben `condition_name` beziehungsweise `sqlstate` einen Fehlerbedingungsnamen oder einen fünfstelligen SQLSTATE-Code an. Die gültigen Fehlerbedingungsnamen und die vordefinierten SQLSTATE-Codes finden Sie in [Anhang A](A_PostgreSQL_Fehlercodes.md).

Hier sind Beispiele für die Verwendung von `condition_name` und `sqlstate`:

```sql
RAISE division_by_zero;
RAISE WARNING SQLSTATE '22012';
```

In jeder dieser Syntaxvarianten können Sie dem Fehlerbericht zusätzliche Informationen hinzufügen, indem Sie `USING` gefolgt von Einträgen der Form `option = expression` schreiben. Jeder Ausdruck kann ein beliebiger zeichenkettenwertiger Ausdruck sein. Die erlaubten Optionsschlüsselwörter sind:

- `MESSAGE`
  - Setzt den Fehlermeldungstext. Diese Option kann in der ersten Syntaxvariante nicht verwendet werden, da die Meldung dort bereits angegeben ist.

- `DETAIL`
  - Liefert eine Detailmeldung zum Fehler.

- `HINT`
  - Liefert eine Hinweismeldung.

- `ERRCODE`
  - Gibt den zu meldenden Fehlercode (SQLSTATE) an, entweder über den Fehlerbedingungsnamen wie in [Anhang A](A_PostgreSQL_Fehlercodes.md) gezeigt oder direkt als fünfstelligen SQLSTATE-Code. Diese Option kann in der zweiten oder dritten Syntaxvariante nicht verwendet werden, da der Fehlercode dort bereits angegeben ist.

- `COLUMN`, `CONSTRAINT`, `DATATYPE`, `TABLE`, `SCHEMA`
  - Liefert den Namen eines zugehörigen Objekts.

Dieses Beispiel bricht die Transaktion mit der angegebenen Fehlermeldung und dem angegebenen Hinweis ab:

```sql
RAISE EXCEPTION 'Nonexistent ID --> %', user_id
      USING HINT = 'Please check your user ID';
```

Diese beiden Beispiele zeigen äquivalente Möglichkeiten, den SQLSTATE zu setzen:

```sql
RAISE 'Duplicate user ID: %', user_id USING ERRCODE =
 'unique_violation';
RAISE 'Duplicate user ID: %', user_id USING ERRCODE = '23505';
```

Eine andere Möglichkeit, dasselbe Ergebnis zu erzeugen, ist:

```sql
RAISE unique_violation USING MESSAGE = 'Duplicate user ID: ' ||
 user_id;
```

Wie in der vierten Syntaxvariante gezeigt, ist es auch möglich, `RAISE USING` oder `RAISE level USING` zu schreiben und alles Weitere in die `USING`-Liste zu setzen.

Die letzte Variante von `RAISE` hat überhaupt keine Parameter. Diese Form kann nur innerhalb der `EXCEPTION`-Klausel eines `BEGIN`-Blocks verwendet werden; sie bewirkt, dass der gerade behandelte Fehler erneut ausgelöst wird.

> Vor PostgreSQL 9.1 wurde `RAISE` ohne Parameter so interpretiert, dass der Fehler aus dem Block erneut ausgelöst wurde, der den aktiven Exception-Handler enthält. Daher konnte eine innerhalb dieses Handlers verschachtelte `EXCEPTION`-Klausel ihn nicht abfangen, selbst wenn das `RAISE` innerhalb des Blocks der verschachtelten `EXCEPTION`-Klausel stand. Dies wurde als überraschend und außerdem als inkompatibel mit Oracles PL/SQL angesehen.

Wenn in einem Befehl `RAISE EXCEPTION` weder ein Bedingungsname noch ein SQLSTATE angegeben ist, wird standardmäßig `raise_exception` (`P0001`) verwendet. Wenn kein Meldungstext angegeben ist, wird standardmäßig der Bedingungsname oder der SQLSTATE als Meldungstext verwendet.

> Wenn Sie einen Fehlercode über einen SQLSTATE-Code angeben, sind Sie nicht auf die vordefinierten Fehlercodes beschränkt, sondern können jeden Fehlercode wählen, der aus fünf Ziffern und/oder ASCII-Großbuchstaben besteht, außer `00000`. Es wird empfohlen, keine Fehlercodes auszulösen, die auf drei Nullen enden, weil dies Kategoriecodes sind und nur abgefangen werden können, indem die ganze Kategorie abgefangen wird.

### 41.9.2. Assertions prüfen

Die Anweisung `ASSERT` ist eine praktische Kurzform, um Debugging-Prüfungen in PL/pgSQL-Funktionen einzufügen.

```text
ASSERT condition [ , message ];
```

`condition` ist ein boolescher Ausdruck, von dem erwartet wird, dass er immer zu wahr ausgewertet wird; wenn dies der Fall ist, tut die Anweisung `ASSERT` nichts weiter. Wenn das Ergebnis falsch oder null ist, wird eine `ASSERT_FAILURE`-Exception ausgelöst. (Wenn beim Auswerten der Bedingung ein Fehler auftritt, wird er als normaler Fehler gemeldet.)

Wenn die optionale `message` angegeben wird, ist sie ein Ausdruck, dessen Ergebnis (falls nicht null) den Standard-Fehlermeldungstext „assertion failed“ ersetzt, falls die Bedingung fehlschlägt. Der Meldungsausdruck wird im Normalfall, in dem die Assertion erfolgreich ist, nicht ausgewertet.

Das Prüfen von Assertions kann über den Konfigurationsparameter `plpgsql.check_asserts` aktiviert oder deaktiviert werden; dieser nimmt einen booleschen Wert an, die Vorgabe ist `on`. Wenn dieser Parameter `off` ist, tun `ASSERT`-Anweisungen nichts.

Beachten Sie, dass `ASSERT` dazu gedacht ist, Programmfehler zu erkennen, nicht um gewöhnliche Fehlerbedingungen zu melden. Verwenden Sie dafür die oben beschriebene Anweisung `RAISE`.

## 41.10. Trigger-Funktionen

PL/pgSQL kann verwendet werden, um Trigger-Funktionen für Datenänderungen oder Datenbankereignisse zu definieren. Eine Trigger-Funktion wird mit dem Befehl `CREATE FUNCTION` erzeugt, wobei sie als Funktion ohne Argumente und mit einem Rückgabetyp `trigger` (für Datenänderungs-Trigger) oder `event_trigger` (für Datenbankereignis-Trigger) deklariert wird. Spezielle lokale Variablen mit Namen der Form `TG_something` werden automatisch definiert, um die Bedingung zu beschreiben, die den Aufruf ausgelöst hat.

### 41.10.1. Trigger bei Datenänderungen

Ein Datenänderungs-Trigger wird als Funktion ohne Argumente und mit dem Rückgabetyp `trigger` deklariert. Beachten Sie, dass die Funktion auch dann ohne Argumente deklariert werden muss, wenn sie erwartet, einige in `CREATE TRIGGER` angegebene Argumente zu erhalten; solche Argumente werden über `TG_ARGV` übergeben, wie unten beschrieben.

Wenn eine PL/pgSQL-Funktion als Trigger aufgerufen wird, werden im äußersten Block automatisch mehrere spezielle Variablen erzeugt. Dies sind:

- `NEW record`
  - Neue Datenbankzeile für `INSERT`-/`UPDATE`-Operationen in zeilenbezogenen Triggern. Diese Variable ist in anweisungsbezogenen Triggern und bei `DELETE`-Operationen null.

- `OLD record`
  - Alte Datenbankzeile für `UPDATE`-/`DELETE`-Operationen in zeilenbezogenen Triggern. Diese Variable ist in anweisungsbezogenen Triggern und bei `INSERT`-Operationen null.

- `TG_NAME name`
  - Name des Triggers, der ausgelöst wurde.

- `TG_WHEN text`
  - `BEFORE`, `AFTER` oder `INSTEAD OF`, je nach Definition des Triggers.

- `TG_LEVEL text`
  - `ROW` oder `STATEMENT`, je nach Definition des Triggers.

- `TG_OP text`
  - Operation, für die der Trigger ausgelöst wurde: `INSERT`, `UPDATE`, `DELETE` oder `TRUNCATE`.

- `TG_RELID oid` (verweist auf `pg_class.oid`)
  - Objekt-ID der Tabelle, die den Trigger-Aufruf verursacht hat.

- `TG_RELNAME name`
  - Tabelle, die den Trigger-Aufruf verursacht hat. Dies ist inzwischen veraltet und könnte in einer zukünftigen Version verschwinden. Verwenden Sie stattdessen `TG_TABLE_NAME`.

- `TG_TABLE_NAME name`
  - Tabelle, die den Trigger-Aufruf verursacht hat.

- `TG_TABLE_SCHEMA name`
  - Schema der Tabelle, die den Trigger-Aufruf verursacht hat.

- `TG_NARGS integer`
  - Anzahl der Argumente, die der Trigger-Funktion in der Anweisung `CREATE TRIGGER` übergeben wurden.

- `TG_ARGV text[]`
  - Argumente aus der Anweisung `CREATE TRIGGER`. Der Index zählt ab 0. Ungültige Indizes (kleiner als 0 oder größer als oder gleich `tg_nargs`) führen zu einem Nullwert.

Eine Trigger-Funktion muss entweder `NULL` oder einen Record-/Zeilenwert zurückgeben, der exakt die Struktur der Tabelle hat, für die der Trigger ausgelöst wurde.

Zeilenbezogene Trigger, die `BEFORE` ausgelöst werden, können null zurückgeben, um dem Trigger-Manager zu signalisieren, dass der Rest der Operation für diese Zeile übersprungen werden soll (das heißt, nachfolgende Trigger werden nicht ausgelöst, und das `INSERT`/`UPDATE`/`DELETE` findet für diese Zeile nicht statt). Wenn ein Nicht-null-Wert zurückgegeben wird, wird die Operation mit diesem Zeilenwert fortgesetzt. Die Rückgabe eines Zeilenwerts, der vom ursprünglichen Wert von `NEW` abweicht, verändert die Zeile, die eingefügt oder aktualisiert wird. Wenn die Trigger-Funktion also möchte, dass die auslösende Aktion normal erfolgreich ist, ohne den Zeilenwert zu verändern, muss `NEW` (oder ein dazu gleicher Wert) zurückgegeben werden. Um die zu speichernde Zeile zu verändern, kann man einzelne Werte direkt in `NEW` ersetzen und das geänderte `NEW` zurückgeben, oder einen vollständigen neuen Record-/Zeilenwert für die Rückgabe aufbauen. Bei einem Before-Trigger auf `DELETE` hat der zurückgegebene Wert keine direkte Wirkung, er muss aber nicht-null sein, damit die Trigger-Aktion fortgesetzt werden darf. Beachten Sie, dass `NEW` in `DELETE`-Triggern null ist, sodass seine Rückgabe normalerweise nicht sinnvoll ist. Das übliche Idiom in `DELETE`-Triggern ist, `OLD` zurückzugeben.

`INSTEAD OF`-Trigger (die immer zeilenbezogene Trigger sind und nur auf Views verwendet werden können) können null zurückgeben, um zu signalisieren, dass sie keine Aktualisierungen vorgenommen haben und dass der Rest der Operation für diese Zeile übersprungen werden soll (das heißt, nachfolgende Trigger werden nicht ausgelöst, und die Zeile wird im Status der betroffenen Zeilen für das umgebende `INSERT`/`UPDATE`/`DELETE` nicht mitgezählt). Andernfalls sollte ein Nicht-null-Wert zurückgegeben werden, um zu signalisieren, dass der Trigger die angeforderte Operation ausgeführt hat. Für `INSERT`- und `UPDATE`-Operationen sollte der Rückgabewert `NEW` sein, den die Trigger-Funktion ändern darf, um `INSERT RETURNING` und `UPDATE RETURNING` zu unterstützen (dies beeinflusst auch den Zeilenwert, der an nachfolgende Trigger übergeben wird oder an eine spezielle `EXCLUDED`-Aliasreferenz innerhalb einer `INSERT`-Anweisung mit einer `ON CONFLICT DO UPDATE`-Klausel). Für `DELETE`-Operationen sollte der Rückgabewert `OLD` sein.

Der Rückgabewert eines zeilenbezogenen Triggers, der `AFTER` ausgelöst wird, oder eines anweisungsbezogenen Triggers, der `BEFORE` oder `AFTER` ausgelöst wird, wird immer ignoriert; er kann genauso gut null sein. Jeder dieser Triggertypen kann die gesamte Operation jedoch weiterhin abbrechen, indem er einen Fehler auslöst.

Beispiel 41.3 zeigt ein Beispiel für eine Trigger-Funktion in PL/pgSQL.

**Beispiel 41.3. Eine PL/pgSQL-Trigger-Funktion**

Dieser Beispiel-Trigger sorgt dafür, dass jedes Mal, wenn eine Zeile in die Tabelle eingefügt oder darin aktualisiert wird, der aktuelle Benutzername und die aktuelle Zeit in die Zeile eingetragen werden. Außerdem prüft er, dass ein Mitarbeitername angegeben ist und dass das Gehalt einen positiven Wert hat.

```sql
CREATE TABLE emp (
    empname                       text,
    salary                        integer,
    last_date                     timestamp,
    last_user                     text
);

CREATE FUNCTION emp_stamp() RETURNS trigger AS $emp_stamp$
    BEGIN
        -- Check that empname and salary are given
        IF NEW.empname IS NULL THEN
            RAISE EXCEPTION 'empname cannot be null';
        END IF;
        IF NEW.salary IS NULL THEN
            RAISE EXCEPTION '% cannot have null salary',
 NEW.empname;
        END IF;

        -- Who works for us when they must pay for it?
        IF NEW.salary < 0 THEN
            RAISE EXCEPTION '% cannot have a negative salary',
 NEW.empname;
        END IF;

            -- Remember who changed the payroll when
            NEW.last_date := current_timestamp;
            NEW.last_user := current_user;
            RETURN NEW;
    END;
$emp_stamp$ LANGUAGE plpgsql;

CREATE TRIGGER emp_stamp BEFORE INSERT OR UPDATE ON emp
      FOR EACH ROW EXECUTE FUNCTION emp_stamp();
```

Eine andere Möglichkeit, Änderungen an einer Tabelle zu protokollieren, besteht darin, eine neue Tabelle anzulegen, die für jedes vorkommende Einfügen, Aktualisieren oder Löschen eine Zeile enthält. Dieser Ansatz kann als Auditieren von Änderungen an einer Tabelle verstanden werden. Beispiel 41.4 zeigt ein Beispiel für eine Audit-Trigger-Funktion in PL/pgSQL.

**Beispiel 41.4. Eine PL/pgSQL-Trigger-Funktion zum Auditieren**

Dieser Beispiel-Trigger sorgt dafür, dass jedes Einfügen, Aktualisieren oder Löschen einer Zeile in der Tabelle `emp` in der Tabelle `emp_audit` aufgezeichnet (das heißt auditiert) wird. Die aktuelle Zeit und der Benutzername werden zusammen mit dem Typ der ausgeführten Operation in die Zeile eingetragen.

```sql
CREATE TABLE emp (
    empname                      text NOT NULL,
    salary                       integer
);

CREATE TABLE emp_audit(
    operation         char(1)   NOT NULL,
    stamp             timestamp NOT NULL,
    userid            text      NOT NULL,
    empname           text      NOT NULL,
    salary            integer
);

CREATE OR REPLACE FUNCTION process_emp_audit() RETURNS TRIGGER AS
 $emp_audit$
    BEGIN
         --
         -- Create a row in emp_audit to reflect the operation
 performed on emp,
         -- making use of the special variable TG_OP to work out the
 operation.
         --
         IF (TG_OP = 'DELETE') THEN
             INSERT INTO emp_audit SELECT 'D', now(), current_user,
 OLD.*;
         ELSIF (TG_OP = 'UPDATE') THEN
             INSERT INTO emp_audit SELECT 'U', now(), current_user,
 NEW.*;
         ELSIF (TG_OP = 'INSERT') THEN
             INSERT INTO emp_audit SELECT 'I', now(), current_user,
 NEW.*;
         END IF;
         RETURN NULL; -- result is ignored since this is an AFTER
 trigger
    END;
$emp_audit$ LANGUAGE plpgsql;

CREATE TRIGGER emp_audit
AFTER INSERT OR UPDATE OR DELETE ON emp
    FOR EACH ROW EXECUTE FUNCTION process_emp_audit();
```

Eine Variante des vorherigen Beispiels verwendet einen View, der die Haupttabelle mit der Audit-Tabelle verbindet, um anzuzeigen, wann jeder Eintrag zuletzt geändert wurde. Dieser Ansatz zeichnet weiterhin die vollständige Audit-Historie der Änderungen an der Tabelle auf, präsentiert aber zusätzlich eine vereinfachte Sicht auf die Audit-Historie, die nur den aus der Audit-Historie abgeleiteten Zeitstempel der letzten Änderung für jeden Eintrag zeigt. Beispiel 41.5 zeigt ein Beispiel für einen Audit-Trigger auf einem View in PL/pgSQL.

**Beispiel 41.5. Eine PL/pgSQL-View-Trigger-Funktion zum Auditieren**

Dieses Beispiel verwendet einen Trigger auf dem View, um ihn aktualisierbar zu machen und sicherzustellen, dass jedes Einfügen, Aktualisieren oder Löschen einer Zeile im View in der Tabelle `emp_audit` aufgezeichnet (das heißt auditiert) wird. Die aktuelle Zeit und der Benutzername werden zusammen mit dem Typ der ausgeführten Operation aufgezeichnet, und der View zeigt die letzte Änderungszeit jeder Zeile an.

```sql
CREATE TABLE emp (
    empname                      text PRIMARY KEY,
    salary                       integer
);

CREATE TABLE emp_audit(
    operation         char(1)   NOT NULL,
    userid            text      NOT NULL,
    empname           text      NOT NULL,
    salary            integer,
    stamp             timestamp NOT NULL
);

CREATE VIEW emp_view AS
    SELECT e.empname,
           e.salary,
           max(ea.stamp) AS last_updated
      FROM emp e
      LEFT JOIN emp_audit ea ON ea.empname = e.empname
     GROUP BY 1, 2;

CREATE OR REPLACE FUNCTION update_emp_view() RETURNS TRIGGER AS $$
    BEGIN
        --
        -- Perform the required operation on emp, and create a row
 in emp_audit
        -- to reflect the change made to emp.
        --
        IF (TG_OP = 'DELETE') THEN
            DELETE FROM emp WHERE empname = OLD.empname;
            IF NOT FOUND THEN RETURN NULL; END IF;

            OLD.last_updated = now();
            INSERT INTO emp_audit VALUES('D', current_user, OLD.*);
            RETURN OLD;
        ELSIF (TG_OP = 'UPDATE') THEN
            UPDATE emp SET salary = NEW.salary WHERE empname =
 OLD.empname;
            IF NOT FOUND THEN RETURN NULL; END IF;

                NEW.last_updated = now();
                INSERT INTO emp_audit VALUES('U', current_user, NEW.*);
                RETURN NEW;
            ELSIF (TG_OP = 'INSERT') THEN
                INSERT INTO emp VALUES(NEW.empname, NEW.salary);

                NEW.last_updated = now();
                INSERT INTO emp_audit VALUES('I', current_user, NEW.*);
                RETURN NEW;
            END IF;
    END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER emp_audit
INSTEAD OF INSERT OR UPDATE OR DELETE ON emp_view
    FOR EACH ROW EXECUTE FUNCTION update_emp_view();
```

Eine Verwendung von Triggern besteht darin, eine Summary-Tabelle einer anderen Tabelle zu pflegen. Die daraus entstehende Zusammenfassung kann für bestimmte Abfragen anstelle der ursprünglichen Tabelle verwendet werden, oft mit deutlich kürzeren Laufzeiten. Diese Technik wird häufig im Data Warehousing verwendet, wo die Tabellen mit gemessenen oder beobachteten Daten (sogenannte Faktentabellen) extrem groß sein können. Beispiel 41.6 zeigt ein Beispiel für eine Trigger-Funktion in PL/pgSQL, die eine Summary-Tabelle für eine Faktentabelle in einem Data Warehouse pflegt.

**Beispiel 41.6. Eine PL/pgSQL-Trigger-Funktion zur Pflege einer Summary-Tabelle**

Das hier detaillierte Schema basiert teilweise auf dem Grocery-Store-Beispiel aus *The Data Warehouse Toolkit* von Ralph Kimball.

```sql
--
-- Main tables - time dimension and sales fact.
--
CREATE TABLE time_dimension (
    time_key                    integer NOT NULL,
    day_of_week                 integer NOT NULL,
    day_of_month                integer NOT NULL,
    month                       integer NOT NULL,
    quarter                     integer NOT NULL,
    year                        integer NOT NULL
);
CREATE UNIQUE INDEX time_dimension_key ON time_dimension(time_key);

CREATE TABLE sales_fact (
    time_key                    integer NOT NULL,
    product_key                 integer NOT NULL,
    store_key                   integer NOT NULL,
    amount_sold                 numeric(12,2) NOT NULL,
    units_sold                  integer NOT NULL,
    amount_cost                 numeric(12,2) NOT NULL
);
CREATE INDEX sales_fact_time ON sales_fact(time_key);

--
-- Summary table - sales by time.
--
CREATE TABLE sales_summary_bytime (
    time_key                     integer NOT NULL,
    amount_sold                  numeric(15,2) NOT NULL,
    units_sold                   numeric(12) NOT NULL,
    amount_cost                  numeric(15,2) NOT NULL
);
CREATE UNIQUE INDEX sales_summary_bytime_key ON
 sales_summary_bytime(time_key);

--
-- Function and trigger to amend summarized column(s) on UPDATE,
 INSERT, DELETE.
--
CREATE OR REPLACE FUNCTION maint_sales_summary_bytime() RETURNS
 TRIGGER
AS $maint_sales_summary_bytime$
    DECLARE
         delta_time_key         integer;
         delta_amount_sold      numeric(15,2);
         delta_units_sold       numeric(12);
         delta_amount_cost      numeric(15,2);
    BEGIN

        -- Work out the increment/decrement amount(s).
        IF (TG_OP = 'DELETE') THEN

             delta_time_key = OLD.time_key;
             delta_amount_sold = -1 * OLD.amount_sold;
             delta_units_sold = -1 * OLD.units_sold;
             delta_amount_cost = -1 * OLD.amount_cost;

        ELSIF (TG_OP = 'UPDATE') THEN

             -- forbid updates that change the time_key -
             -- (probably not too onerous, as DELETE + INSERT is how
 most
             -- changes will be made).
             IF ( OLD.time_key != NEW.time_key) THEN
                 RAISE EXCEPTION 'Update of time_key : % -> % not
 allowed',
                                                         OLD.time_key,
 NEW.time_key;
            END IF;

             delta_time_key = OLD.time_key;
             delta_amount_sold = NEW.amount_sold - OLD.amount_sold;
             delta_units_sold = NEW.units_sold - OLD.units_sold;
             delta_amount_cost = NEW.amount_cost - OLD.amount_cost;

        ELSIF (TG_OP = 'INSERT') THEN

             delta_time_key = NEW.time_key;
             delta_amount_sold = NEW.amount_sold;
             delta_units_sold = NEW.units_sold;
             delta_amount_cost = NEW.amount_cost;

        END IF;

        -- Insert or update the summary row with the new values.
        <<insert_update>>
        LOOP
             UPDATE sales_summary_bytime
                 SET amount_sold = amount_sold + delta_amount_sold,
                     units_sold = units_sold + delta_units_sold,
                     amount_cost = amount_cost + delta_amount_cost
                 WHERE time_key = delta_time_key;

             EXIT insert_update WHEN found;

                  BEGIN
                      INSERT INTO sales_summary_bytime (
                                   time_key,
                                   amount_sold,
                                   units_sold,
                                   amount_cost)
                          VALUES (
                                   delta_time_key,
                                   delta_amount_sold,
                                   delta_units_sold,
                                   delta_amount_cost
                                 );

                        EXIT insert_update;

                EXCEPTION
                     WHEN UNIQUE_VIOLATION THEN
                         -- do nothing
                END;
            END LOOP insert_update;

            RETURN NULL;

    END;
$maint_sales_summary_bytime$ LANGUAGE plpgsql;

CREATE TRIGGER maint_sales_summary_bytime
AFTER INSERT OR UPDATE OR DELETE ON sales_fact
    FOR EACH ROW EXECUTE FUNCTION maint_sales_summary_bytime();

INSERT INTO sales_fact VALUES(1,1,1,10,3,15);
INSERT INTO sales_fact VALUES(1,2,1,20,5,35);
INSERT INTO sales_fact VALUES(2,2,1,40,15,135);
INSERT INTO sales_fact VALUES(2,3,1,10,1,13);
SELECT * FROM sales_summary_bytime;
DELETE FROM sales_fact WHERE product_key = 1;
SELECT * FROM sales_summary_bytime;
UPDATE sales_fact SET units_sold = units_sold * 2;
SELECT * FROM sales_summary_bytime;
```

`AFTER`-Trigger können auch Transition Tables verwenden, um die gesamte Menge der Zeilen zu untersuchen, die durch die auslösende Anweisung geändert wurden. Der Befehl `CREATE TRIGGER` weist einer oder beiden Transition Tables Namen zu, und die Funktion kann dann auf diese Namen verweisen, als wären es read-only temporäre Tabellen. Beispiel 41.7 zeigt ein Beispiel.

**Beispiel 41.7. Auditieren mit Transition Tables**

Dieses Beispiel erzeugt dieselben Ergebnisse wie Beispiel 41.4, verwendet aber statt eines Triggers, der für jede Zeile ausgelöst wird, einen Trigger, der einmal pro Anweisung ausgelöst wird, nachdem die relevanten Informationen in einer Transition Table gesammelt wurden. Das kann deutlich schneller sein als der Ansatz mit zeilenbezogenen Triggern, wenn die aufrufende Anweisung viele Zeilen geändert hat. Beachten Sie, dass wir für jede Ereignisart eine eigene Trigger-Deklaration vornehmen müssen, da die `REFERENCING`-Klauseln jeweils unterschiedlich sein müssen. Das hindert uns aber nicht daran, eine einzige Trigger-Funktion zu verwenden, wenn wir das möchten. (In der Praxis kann es besser sein, drei getrennte Funktionen zu verwenden und die Laufzeittests auf `TG_OP` zu vermeiden.)

```sql
CREATE TABLE emp (
    empname                       text NOT NULL,
    salary                        integer
);

CREATE TABLE emp_audit(
    operation         char(1)   NOT NULL,
    stamp             timestamp NOT NULL,
    userid            text      NOT NULL,
    empname           text      NOT NULL,
    salary            integer
);

CREATE OR REPLACE FUNCTION process_emp_audit() RETURNS TRIGGER AS
 $emp_audit$
    BEGIN
         --
         -- Create rows in emp_audit to reflect the operations
 performed on emp,
         -- making use of the special variable TG_OP to work out the
 operation.
         --
         IF (TG_OP = 'DELETE') THEN
             INSERT INTO emp_audit
                 SELECT 'D', now(), current_user, o.* FROM old_table
 o;
         ELSIF (TG_OP = 'UPDATE') THEN
             INSERT INTO emp_audit
                 SELECT 'U', now(), current_user, n.* FROM new_table
 n;
         ELSIF (TG_OP = 'INSERT') THEN
             INSERT INTO emp_audit
                 SELECT 'I', now(), current_user, n.* FROM new_table
 n;
         END IF;
         RETURN NULL; -- result is ignored since this is an AFTER
 trigger
    END;
$emp_audit$ LANGUAGE plpgsql;

CREATE TRIGGER emp_audit_ins
    AFTER INSERT ON emp
    REFERENCING NEW TABLE AS new_table
    FOR EACH STATEMENT EXECUTE FUNCTION process_emp_audit();
CREATE TRIGGER emp_audit_upd
    AFTER UPDATE ON emp
    REFERENCING OLD TABLE AS old_table NEW TABLE AS new_table
    FOR EACH STATEMENT EXECUTE FUNCTION process_emp_audit();
CREATE TRIGGER emp_audit_del
    AFTER DELETE ON emp
    REFERENCING OLD TABLE AS old_table
    FOR EACH STATEMENT EXECUTE FUNCTION process_emp_audit();
```

### 41.10.2. Trigger bei Ereignissen

PL/pgSQL kann verwendet werden, um Event-Trigger zu definieren. PostgreSQL verlangt, dass eine Funktion, die als Event-Trigger aufgerufen werden soll, als Funktion ohne Argumente und mit dem Rückgabetyp `event_trigger` deklariert wird.

Wenn eine PL/pgSQL-Funktion als Event-Trigger aufgerufen wird, werden im äußersten Block automatisch mehrere spezielle Variablen erzeugt. Dies sind:

- `TG_EVENT text`
  - Ereignis, für das der Trigger ausgelöst wird.

- `TG_TAG text`
  - Command Tag, für den der Trigger ausgelöst wird.

Beispiel 41.8 zeigt ein Beispiel für eine Event-Trigger-Funktion in PL/pgSQL.

**Beispiel 41.8. Eine PL/pgSQL-Event-Trigger-Funktion**

Dieser Beispiel-Trigger gibt einfach jedes Mal eine `NOTICE`-Meldung aus, wenn ein unterstützter Befehl ausgeführt wird.

```sql
CREATE OR REPLACE FUNCTION snitch() RETURNS event_trigger AS $$
BEGIN
     RAISE NOTICE 'snitch: % %', tg_event, tg_tag;
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER snitch ON ddl_command_start EXECUTE FUNCTION
 snitch();
```

## 41.11. PL/pgSQL unter der Haube

Dieser Abschnitt behandelt einige Implementierungsdetails, die für PL/pgSQL-Benutzer häufig wichtig zu wissen sind.

### 41.11.1. Variablenersetzung

SQL-Anweisungen und Ausdrücke innerhalb einer PL/pgSQL-Funktion können auf Variablen und Parameter der Funktion verweisen. Im Hintergrund ersetzt PL/pgSQL solche Verweise durch Abfrageparameter. Abfrageparameter werden nur an Stellen eingesetzt, an denen sie syntaktisch zulässig sind. Betrachten Sie als Extremfall dieses Beispiel schlechten Programmierstils:

```sql
INSERT INTO foo (foo) VALUES (foo(foo));
```

Das erste Vorkommen von `foo` muss syntaktisch ein Tabellenname sein und wird daher nicht ersetzt, selbst wenn die Funktion eine Variable namens `foo` hat. Das zweite Vorkommen muss der Name einer Spalte dieser Tabelle sein und wird daher ebenfalls nicht ersetzt. Ebenso muss das dritte Vorkommen ein Funktionsname sein und wird deshalb auch nicht ersetzt. Nur das letzte Vorkommen ist ein Kandidat für einen Verweis auf eine Variable der PL/pgSQL-Funktion.

Eine andere Art, dies zu verstehen: Variablenersetzung kann nur Datenwerte in einen SQL-Befehl einsetzen; sie kann nicht dynamisch ändern, auf welche Datenbankobjekte der Befehl verweist. (Wenn Sie das tun möchten, müssen Sie eine Befehlszeichenkette dynamisch aufbauen, wie in [Abschnitt 41.5.4](#4154-dynamische-befehle-ausführen) erläutert.)

Da sich die Namen von Variablen syntaktisch nicht von den Namen von Tabellenspalten unterscheiden, kann es in Anweisungen, die auch auf Tabellen verweisen, Mehrdeutigkeiten geben: Soll ein bestimmter Name auf eine Tabellenspalte oder auf eine Variable verweisen? Ändern wir das vorherige Beispiel zu:

```sql
INSERT INTO dest (col) SELECT foo + bar FROM src;
```

Hier müssen `dest` und `src` Tabellennamen sein, und `col` muss eine Spalte von `dest` sein, aber `foo` und `bar` könnten sinnvollerweise entweder Variablen der Funktion oder Spalten von `src` sein.

Standardmäßig meldet PL/pgSQL einen Fehler, wenn ein Name in einer SQL-Anweisung entweder auf eine Variable oder auf eine Tabellenspalte verweisen könnte. Sie können ein solches Problem beheben, indem Sie die Variable oder Spalte umbenennen, den mehrdeutigen Verweis qualifizieren oder PL/pgSQL mitteilen, welche Interpretation bevorzugt werden soll.

Die einfachste Lösung ist, die Variable oder Spalte umzubenennen. Eine übliche Codierregel ist, für PL/pgSQL-Variablen eine andere Namenskonvention zu verwenden als für Spaltennamen. Wenn Sie Funktionsvariablen zum Beispiel konsequent `v_something` nennen, während keiner Ihrer Spaltennamen mit `v_` beginnt, treten keine Konflikte auf.

Alternativ können Sie mehrdeutige Verweise qualifizieren, um sie eindeutig zu machen. Im obigen Beispiel wäre `src.foo` ein eindeutiger Verweis auf die Tabellenspalte. Um einen eindeutigen Verweis auf eine Variable zu erzeugen, deklarieren Sie sie in einem markierten Block und verwenden das Label des Blocks (siehe [Abschnitt 41.2](#412-struktur-von-plpgsql)). Zum Beispiel:

```sql
<<block>>
DECLARE
    foo int;
BEGIN
    foo := ...;
    INSERT INTO dest (col) SELECT block.foo + bar FROM src;
```

Hier bedeutet `block.foo` die Variable, selbst wenn es in `src` eine Spalte `foo` gibt. Funktionsparameter sowie spezielle Variablen wie `FOUND` können mit dem Funktionsnamen qualifiziert werden, weil sie implizit in einem äußeren Block deklariert werden, der mit dem Funktionsnamen beschriftet ist.

Manchmal ist es unpraktisch, alle mehrdeutigen Verweise in einem großen PL/pgSQL-Codebestand zu beheben. In solchen Fällen können Sie angeben, dass PL/pgSQL mehrdeutige Verweise als Variable auflösen soll (kompatibel mit dem Verhalten von PL/pgSQL vor PostgreSQL 9.0) oder als Tabellenspalte (kompatibel mit einigen anderen Systemen wie Oracle).

Um dieses Verhalten systemweit zu ändern, setzen Sie den Konfigurationsparameter `plpgsql.variable_conflict` auf einen der Werte `error`, `use_variable` oder `use_column` (wobei `error` die Werkseinstellung ist). Dieser Parameter betrifft nachfolgende Kompilierungen von Anweisungen in PL/pgSQL-Funktionen, aber keine Anweisungen, die in der aktuellen Sitzung bereits kompiliert wurden. Da eine Änderung dieser Einstellung unerwartete Änderungen im Verhalten von PL/pgSQL-Funktionen verursachen kann, kann sie nur von einem Superuser geändert werden.

Sie können das Verhalten auch funktionsweise festlegen, indem Sie einen dieser speziellen Befehle an den Anfang des Funktionstextes einfügen:

```text
#variable_conflict error
#variable_conflict use_variable
#variable_conflict use_column
```

Diese Befehle betreffen nur die Funktion, in der sie geschrieben stehen, und übersteuern die Einstellung von `plpgsql.variable_conflict`. Ein Beispiel ist:

```sql
CREATE FUNCTION stamp_user(id int, comment text) RETURNS void AS $$
    #variable_conflict use_variable
    DECLARE
         curtime timestamp := now();
    BEGIN
         UPDATE users SET last_modified = curtime, comment = comment
           WHERE users.id = id;
    END;
$$ LANGUAGE plpgsql;
```

Im Befehl `UPDATE` verweisen `curtime`, `comment` und `id` auf die Variable beziehungsweise Parameter der Funktion, unabhängig davon, ob `users` Spalten mit diesen Namen hat. Beachten Sie, dass wir den Verweis auf `users.id` in der `WHERE`-Klausel qualifizieren mussten, damit er auf die Tabellenspalte verweist. Den Verweis auf `comment` als Ziel in der `UPDATE`-Liste mussten wir dagegen nicht qualifizieren, weil dies syntaktisch eine Spalte von `users` sein muss. Wir könnten dieselbe Funktion auch ohne Abhängigkeit von der Einstellung `variable_conflict` so schreiben:

```sql
CREATE FUNCTION stamp_user(id int, comment text) RETURNS void AS $$
    <<fn>>
    DECLARE
         curtime timestamp := now();
    BEGIN
         UPDATE users SET last_modified = fn.curtime, comment =
 stamp_user.comment
           WHERE users.id = stamp_user.id;
    END;
$$ LANGUAGE plpgsql;
```

Variablenersetzung findet nicht in einer Befehlszeichenkette statt, die `EXECUTE` oder einer seiner Varianten übergeben wird. Wenn Sie einen veränderlichen Wert in einen solchen Befehl einsetzen müssen, tun Sie dies beim Aufbau der Zeichenkette oder verwenden Sie `USING`, wie in [Abschnitt 41.5.4](#4154-dynamische-befehle-ausführen) gezeigt.

Variablenersetzung funktioniert derzeit nur in `SELECT`, `INSERT`, `UPDATE`, `DELETE` und Befehlen, die einen dieser Befehle enthalten (etwa `EXPLAIN` und `CREATE TABLE ... AS SELECT`), weil die Haupt-SQL-Engine Abfrageparameter nur in diesen Befehlen erlaubt. Um in anderen Anweisungstypen (allgemein Utility-Anweisungen genannt) einen nicht konstanten Namen oder Wert zu verwenden, müssen Sie die Utility-Anweisung als Zeichenkette aufbauen und mit `EXECUTE` ausführen.

### 41.11.2. Plan-Caching

Der PL/pgSQL-Interpreter parst den Quelltext der Funktion und erzeugt beim ersten Aufruf der Funktion (innerhalb jeder Sitzung) einen internen binären Anweisungsbaum. Der Anweisungsbaum übersetzt die PL/pgSQL-Anweisungsstruktur vollständig, aber einzelne SQL-Ausdrücke und SQL-Befehle, die in der Funktion verwendet werden, werden nicht sofort übersetzt.

Wenn jeder Ausdruck und SQL-Befehl erstmals in der Funktion ausgeführt wird, parst und analysiert der PL/pgSQL-Interpreter den Befehl, um mithilfe der Funktion `SPI_prepare` des SPI-Managers eine vorbereitete Anweisung zu erzeugen. Spätere Besuche dieses Ausdrucks oder Befehls verwenden die vorbereitete Anweisung wieder. Daher verursacht eine Funktion mit bedingten Codepfaden, die selten besucht werden, niemals den Aufwand für die Analyse jener Befehle, die in der aktuellen Sitzung nie ausgeführt werden. Ein Nachteil ist, dass Fehler in einem bestimmten Ausdruck oder Befehl erst erkannt werden können, wenn dieser Teil der Funktion in der Ausführung erreicht wird. (Triviale Syntaxfehler werden beim ersten Parse-Durchlauf erkannt, alles Tiefere aber erst bei der Ausführung.)

PL/pgSQL (genauer gesagt der SPI-Manager) kann außerdem versuchen, den Ausführungsplan zu cachen, der zu einer bestimmten vorbereiteten Anweisung gehört. Wenn kein gecachter Plan verwendet wird, wird bei jedem Besuch der Anweisung ein frischer Ausführungsplan erzeugt, und die aktuellen Parameterwerte (also die Werte von PL/pgSQL-Variablen) können verwendet werden, um den ausgewählten Plan zu optimieren. Wenn die Anweisung keine Parameter hat oder sehr häufig ausgeführt wird, erwägt der SPI-Manager, einen generischen Plan zu erzeugen, der nicht von bestimmten Parameterwerten abhängt, und diesen zur Wiederverwendung zu cachen. Typischerweise geschieht dies nur, wenn der Ausführungsplan nicht sehr empfindlich auf die Werte der darin referenzierten PL/pgSQL-Variablen reagiert. Wenn er das doch tut, ist es insgesamt günstiger, jedes Mal einen Plan zu erzeugen. Weitere Informationen zum Verhalten vorbereiteter Anweisungen finden Sie bei `PREPARE`.

Da PL/pgSQL vorbereitete Anweisungen und manchmal Ausführungspläne auf diese Weise speichert, müssen SQL-Befehle, die direkt in einer PL/pgSQL-Funktion erscheinen, bei jeder Ausführung auf dieselben Tabellen und Spalten verweisen; das heißt, Sie können einen Parameter in einem SQL-Befehl nicht als Namen einer Tabelle oder Spalte verwenden. Um diese Einschränkung zu umgehen, können Sie dynamische Befehle mit der PL/pgSQL-Anweisung `EXECUTE` aufbauen, um den Preis, dass bei jeder Ausführung eine neue Parse-Analyse durchgeführt und ein neuer Ausführungsplan erstellt wird.

Die veränderliche Natur von Record-Variablen stellt in diesem Zusammenhang ein weiteres Problem dar. Wenn Felder einer Record-Variablen in Ausdrücken oder Anweisungen verwendet werden, dürfen sich die Datentypen der Felder von einem Funktionsaufruf zum nächsten nicht ändern, da jeder Ausdruck mit dem Datentyp analysiert wird, der beim ersten Erreichen des Ausdrucks vorhanden ist. `EXECUTE` kann bei Bedarf verwendet werden, um dieses Problem zu umgehen.

Wenn dieselbe Funktion als Trigger für mehr als eine Tabelle verwendet wird, bereitet PL/pgSQL Anweisungen unabhängig für jede dieser Tabellen vor und cached sie auch unabhängig, das heißt, es gibt einen Cache für jede Kombination aus Trigger-Funktion und Tabelle, nicht nur für jede Funktion. Das entschärft einige Probleme mit wechselnden Datentypen; zum Beispiel kann eine Trigger-Funktion erfolgreich mit einer Spalte namens `key` arbeiten, auch wenn diese in verschiedenen Tabellen zufällig unterschiedliche Typen hat.

Ebenso haben Funktionen mit polymorphen Argumenttypen einen getrennten Anweisungs-Cache für jede Kombination tatsächlicher Argumenttypen, mit denen sie aufgerufen wurden, sodass Datentypunterschiede keine unerwarteten Fehler verursachen.

Anweisungs-Caching kann manchmal überraschende Auswirkungen auf die Interpretation zeitabhängiger Werte haben. Zum Beispiel gibt es einen Unterschied zwischen dem Verhalten dieser beiden Funktionen:

```sql
CREATE FUNCTION logfunc1(logtxt text) RETURNS void AS $$
    BEGIN
         INSERT INTO logtable VALUES (logtxt, 'now');
    END;
$$ LANGUAGE plpgsql;
```

und:

```sql
CREATE FUNCTION logfunc2(logtxt text) RETURNS void AS $$
    DECLARE
         curtime timestamp;
    BEGIN
         curtime := 'now';
         INSERT INTO logtable VALUES (logtxt, curtime);
    END;
$$ LANGUAGE plpgsql;
```

Im Fall von `logfunc1` weiß der PostgreSQL-Hauptparser beim Analysieren des `INSERT`, dass die Zeichenkette `'now'` als `timestamp` interpretiert werden soll, weil die Zielspalte von `logtable` diesen Typ hat. Daher wird `'now'` beim Analysieren des `INSERT` in eine Timestamp-Konstante umgewandelt und dann in allen Aufrufen von `logfunc1` während der Lebensdauer der Sitzung verwendet. Das ist natürlich nicht das, was der Programmierer wollte. Eine bessere Idee ist, die Funktion `now()` oder `current_timestamp` zu verwenden.

Im Fall von `logfunc2` weiß der PostgreSQL-Hauptparser nicht, welcher Typ aus `'now'` werden soll, und gibt daher einen Datenwert des Typs `text` zurück, der die Zeichenkette `now` enthält. Während der anschließenden Zuweisung an die lokale Variable `curtime` castet der PL/pgSQL-Interpreter diese Zeichenkette in den Typ `timestamp`, indem er für die Umwandlung die Funktionen `textout` und `timestamp_in` aufruft. Der berechnete Zeitstempel wird also bei jeder Ausführung aktualisiert, wie der Programmierer es erwartet. Auch wenn dies zufällig wie erwartet funktioniert, ist es nicht besonders effizient; die Verwendung der Funktion `now()` wäre daher weiterhin die bessere Idee.

## 41.12. Tipps für die Entwicklung in PL/pgSQL

Eine gute Möglichkeit, in PL/pgSQL zu entwickeln, besteht darin, mit einem Texteditor Ihrer Wahl Ihre Funktionen zu erstellen und in einem anderen Fenster `psql` zu verwenden, um diese Funktionen zu laden und zu testen. Wenn Sie so arbeiten, ist es sinnvoll, die Funktion mit `CREATE OR REPLACE FUNCTION` zu schreiben. Dann können Sie die Datei einfach erneut laden, um die Funktionsdefinition zu aktualisieren. Zum Beispiel:

```sql
CREATE OR REPLACE FUNCTION testfunc(integer) RETURNS integer AS $$
          ....
$$ LANGUAGE plpgsql;
```

Während `psql` läuft, können Sie eine solche Datei mit Funktionsdefinitionen laden oder erneut laden mit:

```text
\i filename.sql
```

und anschließend sofort SQL-Befehle absetzen, um die Funktion zu testen.

Eine weitere gute Möglichkeit, in PL/pgSQL zu entwickeln, ist ein GUI-Datenbankzugriffswerkzeug, das die Entwicklung in einer prozeduralen Sprache erleichtert. Ein Beispiel für ein solches Werkzeug ist pgAdmin, es gibt aber auch andere. Diese Werkzeuge bieten oft praktische Funktionen wie das Escapen einfacher Anführungszeichen und erleichtern es, Funktionen neu zu erstellen und zu debuggen.

### 41.12.1. Umgang mit Anführungszeichen

Der Code einer PL/pgSQL-Funktion wird in `CREATE FUNCTION` als Zeichenkettenliteral angegeben. Wenn Sie das Zeichenkettenliteral auf die gewöhnliche Weise mit umgebenden einfachen Anführungszeichen schreiben, müssen alle einfachen Anführungszeichen innerhalb des Funktionsrumpfs verdoppelt werden; ebenso müssen alle Backslashes verdoppelt werden (unter der Annahme, dass Escape-String-Syntax verwendet wird). Das Verdoppeln von Anführungszeichen ist bestenfalls mühsam, und in komplizierteren Fällen kann der Code geradezu unverständlich werden, weil man leicht ein halbes Dutzend oder mehr benachbarte Anführungszeichen benötigt. Es wird empfohlen, den Funktionsrumpf stattdessen als „dollar-quoted“ Zeichenkettenliteral zu schreiben (siehe [Abschnitt 4.1.2.4](04_SQL_Syntax.md#4124-dollarquotedzeichenkettenkonstanten)). Beim Dollar-Quoting verdoppeln Sie niemals Anführungszeichen, sondern achten stattdessen darauf, für jede benötigte Verschachtelungsebene einen anderen Dollar-Quoting-Begrenzer zu wählen. Zum Beispiel könnten Sie den Befehl `CREATE FUNCTION` so schreiben:

```sql
CREATE OR REPLACE FUNCTION testfunc(integer) RETURNS integer AS
 $PROC$
          ....
$PROC$ LANGUAGE plpgsql;
```

Darin könnten Sie Anführungszeichen für einfache Literalzeichenketten in SQL-Befehlen verwenden und `$$`, um Fragmente von SQL-Befehlen abzugrenzen, die Sie als Zeichenketten zusammensetzen. Wenn Sie Text quoten müssen, der `$$` enthält, könnten Sie `$Q$` verwenden und so weiter.

Die folgende Übersicht zeigt, was Sie tun müssen, wenn Sie Anführungszeichen ohne Dollar-Quoting schreiben. Sie kann nützlich sein, wenn Sie Code vor der Dollar-Quoting-Zeit in etwas Verständlicheres übersetzen.

**1 Anführungszeichen**

Zum Beginnen und Beenden des Funktionsrumpfs, zum Beispiel:

```sql
CREATE FUNCTION foo() RETURNS integer AS '
          ....
' LANGUAGE plpgsql;
```

An jeder Stelle innerhalb eines einfach gequoteten Funktionsrumpfs müssen Anführungszeichen paarweise auftreten.

**2 Anführungszeichen**

Für Zeichenkettenliterale innerhalb des Funktionsrumpfs, zum Beispiel:

```sql
a_output := ''Blah'';
SELECT * FROM users WHERE f_name=''foobar'';
```

Beim Dollar-Quoting-Ansatz würden Sie einfach schreiben:

```sql
a_output := 'Blah';
SELECT * FROM users WHERE f_name='foobar';
```

Das ist genau das, was der PL/pgSQL-Parser in beiden Fällen sehen würde.

**4 Anführungszeichen**

Wenn Sie ein einzelnes Anführungszeichen in einer Zeichenkettenkonstante innerhalb des Funktionsrumpfs benötigen, zum Beispiel:

```sql
a_output := a_output || '' AND name LIKE ''''foobar'''' AND
 xyz''
```

Der tatsächlich an `a_output` angehängte Wert wäre:

```text
AND name LIKE 'foobar' AND xyz
```

Beim Dollar-Quoting-Ansatz würden Sie schreiben:

```sql
a_output := a_output || $$ AND name LIKE 'foobar' AND xyz$$
```

Dabei müssen Sie darauf achten, dass etwaige Dollar-Quote-Begrenzer außen herum nicht einfach `$$` sind.

**6 Anführungszeichen**

Wenn ein einzelnes Anführungszeichen in einer Zeichenkette innerhalb des Funktionsrumpfs direkt an das Ende dieser Zeichenkettenkonstante grenzt, zum Beispiel:

```sql
a_output := a_output || '' AND name LIKE ''''foobar''''''
```

Der an `a_output` angehängte Wert wäre dann:

```text
AND name LIKE 'foobar'
```

Beim Dollar-Quoting-Ansatz wird daraus:

```sql
a_output := a_output || $$ AND name LIKE 'foobar'$$
```

**10 Anführungszeichen**

Wenn Sie zwei einfache Anführungszeichen in einer Zeichenkettenkonstante möchten (was 8 Anführungszeichen ausmacht) und dies direkt an das Ende dieser Zeichenkettenkonstante grenzt (2 weitere). Das werden Sie wahrscheinlich nur brauchen, wenn Sie eine Funktion schreiben, die andere Funktionen erzeugt, wie in Beispiel 41.10. Zum Beispiel:

```sql
a_output := a_output || '' if v_'' ||
    referrer_keys.kind || '' like ''''''''''
    || referrer_keys.key_string || ''''''''''
    then return '''''' || referrer_keys.referrer_type
    || ''''''; end if;'';
```

Der Wert von `a_output` wäre dann:

```text
if v_... like ''...'' then return ''...''; end if;
```

Beim Dollar-Quoting-Ansatz wird daraus:

```sql
a_output := a_output || $$ if v_$$ || referrer_keys.kind || $$
 like '$$
    || referrer_keys.key_string || $$'
    then return '$$ || referrer_keys.referrer_type
    || $$'; end if;$$;
```

Dabei nehmen wir an, dass wir nur einfache Anführungszeichen in `a_output` einfügen müssen, weil es vor der Verwendung erneut gequotet wird.

### 41.12.2. Zusätzliche Prüfungen zur Compile- und Laufzeit

Um Benutzer dabei zu unterstützen, einfache, aber häufige Probleme zu finden, bevor sie Schaden verursachen, bietet PL/pgSQL zusätzliche Prüfungen. Wenn sie aktiviert sind, können sie je nach Konfiguration beim Kompilieren einer Funktion entweder eine `WARNING` oder einen `ERROR` ausgeben. Eine Funktion, die eine `WARNING` erhalten hat, kann ohne weitere Meldungen ausgeführt werden; daher empfiehlt es sich, in einer getrennten Entwicklungsumgebung zu testen.

In Entwicklungs- und/oder Testumgebungen wird empfohlen, `plpgsql.extra_warnings` beziehungsweise `plpgsql.extra_errors` passend auf `"all"` zu setzen.

Diese zusätzlichen Prüfungen werden über die Konfigurationsvariablen `plpgsql.extra_warnings` für Warnungen und `plpgsql.extra_errors` für Fehler aktiviert. Beide können entweder auf eine kommagetrennte Liste von Prüfungen, `"none"` oder `"all"` gesetzt werden. Die Vorgabe ist `"none"`. Derzeit umfasst die Liste der verfügbaren Prüfungen:

- `shadowed_variables`
  - Prüft, ob eine Deklaration eine zuvor definierte Variable verdeckt.

- `strict_multi_assignment`
  - Einige PL/pgSQL-Befehle erlauben es, mehreren Variablen gleichzeitig Werte zuzuweisen, etwa `SELECT INTO`. Typischerweise sollten die Anzahl der Zielvariablen und die Anzahl der Quellvariablen übereinstimmen, obwohl PL/pgSQL für fehlende Werte `NULL` verwendet und zusätzliche Variablen ignoriert. Wenn diese Prüfung aktiviert wird, gibt PL/pgSQL immer dann eine `WARNING` oder einen `ERROR` aus, wenn sich die Anzahl der Zielvariablen von der Anzahl der Quellvariablen unterscheidet.

- `too_many_rows`
  - Wenn diese Prüfung aktiviert ist, prüft PL/pgSQL, ob eine gegebene Abfrage bei Verwendung einer `INTO`-Klausel mehr als eine Zeile zurückgibt. Da eine `INTO`-Anweisung immer nur eine Zeile verwendet, ist eine Abfrage, die mehrere Zeilen zurückgibt, im Allgemeinen entweder ineffizient und/oder nicht deterministisch und daher wahrscheinlich ein Fehler.

Das folgende Beispiel zeigt die Wirkung von `plpgsql.extra_warnings`, wenn es auf `shadowed_variables` gesetzt ist:

```sql
SET plpgsql.extra_warnings TO 'shadowed_variables';

CREATE FUNCTION foo(f1 int) RETURNS int AS $$
DECLARE
f1 int;
BEGIN
RETURN f1;
END;
$$ LANGUAGE plpgsql;
```

```text
WARNING: variable "f1" shadows a previously defined variable
LINE 3: f1 int;
        ^
CREATE FUNCTION
```

Das folgende Beispiel zeigt die Auswirkungen, wenn `plpgsql.extra_warnings` auf `strict_multi_assignment` gesetzt wird:

```sql
SET plpgsql.extra_warnings TO 'strict_multi_assignment';

CREATE OR REPLACE FUNCTION public.foo()
 RETURNS void
 LANGUAGE plpgsql
AS $$
DECLARE
  x int;
  y int;
BEGIN
  SELECT 1 INTO x, y;
  SELECT 1, 2 INTO x, y;
  SELECT 1, 2, 3 INTO x, y;
END;
$$;

SELECT foo();
```

```text
WARNING: number of source and target fields in assignment does not
 match
DETAIL: strict_multi_assignment check of extra_warnings is active.
HINT: Make sure the query returns the exact list of columns.
WARNING: number of source and target fields in assignment does not
 match
DETAIL: strict_multi_assignment check of extra_warnings is active.
HINT: Make sure the query returns the exact list of columns.

 foo
-----

(1 row)
```

## 41.13. Portierung von Oracle PL/SQL

Dieser Abschnitt erklärt Unterschiede zwischen PostgreSQLs Sprache PL/pgSQL und Oracles Sprache PL/SQL, um Entwickler zu unterstützen, die Anwendungen von Oracle® nach PostgreSQL portieren.

PL/pgSQL ähnelt PL/SQL in vielerlei Hinsicht. Es ist eine blockstrukturierte, imperative Sprache, und alle Variablen müssen deklariert werden. Zuweisungen, Schleifen und Bedingungen sind ähnlich. Die wichtigsten Unterschiede, die Sie beim Portieren von PL/SQL nach PL/pgSQL im Blick behalten sollten, sind:

- Wenn ein in einem SQL-Befehl verwendeter Name entweder ein Spaltenname einer im Befehl verwendeten Tabelle oder ein Verweis auf eine Variable der Funktion sein könnte, behandelt PL/SQL ihn als Spaltennamen. Standardmäßig wirft PL/pgSQL einen Fehler, der meldet, dass der Name mehrdeutig ist. Sie können `plpgsql.variable_conflict = use_column` angeben, um dieses Verhalten an PL/SQL anzupassen, wie in [Abschnitt 41.11.1](#41111-variablenersetzung) erklärt. Häufig ist es am besten, solche Mehrdeutigkeiten von vornherein zu vermeiden; wenn Sie jedoch eine große Menge Code portieren müssen, die von diesem Verhalten abhängt, kann das Setzen von `variable_conflict` die beste Lösung sein.

- In PostgreSQL muss der Funktionsrumpf als Zeichenkettenliteral geschrieben werden. Daher müssen Sie Dollar-Quoting verwenden oder einfache Anführungszeichen im Funktionsrumpf escapen. (Siehe [Abschnitt 41.12.1](#41121-umgang-mit-anführungszeichen).)

- Datentypnamen müssen häufig übersetzt werden. In Oracle werden Zeichenkettenwerte zum Beispiel üblicherweise als Typ `varchar2` deklariert, der kein SQL-Standardtyp ist. Verwenden Sie in PostgreSQL stattdessen den Typ `varchar` oder `text`. Ersetzen Sie ähnlich den Typ `number` durch `numeric`, oder verwenden Sie einen anderen numerischen Datentyp, falls einer besser passt.

- Verwenden Sie statt Packages Schemas, um Ihre Funktionen in Gruppen zu organisieren.

- Da es keine Packages gibt, gibt es auch keine Variablen auf Package-Ebene. Das ist etwas lästig. Sie können sitzungsbezogenen Zustand stattdessen in temporären Tabellen halten.

- Ganzzahlige `FOR`-Schleifen mit `REVERSE` funktionieren anders: PL/SQL zählt von der zweiten Zahl zur ersten herunter, während PL/pgSQL von der ersten Zahl zur zweiten herunterzählt, sodass beim Portieren die Schleifengrenzen vertauscht werden müssen. Diese Inkompatibilität ist bedauerlich, wird sich aber wahrscheinlich nicht ändern. (Siehe [Abschnitt 41.6.5.5](#41655-for-integervariante).)

- `FOR`-Schleifen über Abfragen (außer Cursorn) funktionieren ebenfalls anders: Die Zielvariable(n) müssen deklariert worden sein, während PL/SQL sie immer implizit deklariert. Ein Vorteil davon ist, dass die Variablenwerte nach dem Verlassen der Schleife noch zugänglich sind.

- Es gibt verschiedene notationsbezogene Unterschiede bei der Verwendung von Cursor-Variablen.

### 41.13.1. Portierungsbeispiele

Beispiel 41.9 zeigt, wie eine einfache Funktion von PL/SQL nach PL/pgSQL portiert wird.

**Beispiel 41.9. Portierung einer einfachen Funktion von PL/SQL nach PL/pgSQL**

Hier ist eine Oracle-PL/SQL-Funktion:

```sql
CREATE OR REPLACE FUNCTION cs_fmt_browser_version(v_name varchar2,
                                                  v_version
  varchar2)
RETURN varchar2 IS
BEGIN
     IF v_version IS NULL THEN
         RETURN v_name;
     END IF;
     RETURN v_name || '/' || v_version;
END;
/
show errors;
```

Gehen wir diese Funktion durch und betrachten die Unterschiede zu PL/pgSQL:

- Der Typname `varchar2` muss in `varchar` oder `text` geändert werden. In den Beispielen dieses Abschnitts verwenden wir `varchar`, aber `text` ist oft die bessere Wahl, wenn Sie keine spezifischen Längenbegrenzungen für Zeichenketten benötigen.

- Das Schlüsselwort `RETURN` im Funktionsprototyp (nicht im Funktionsrumpf) wird in PostgreSQL zu `RETURNS`. Außerdem wird `IS` zu `AS`, und Sie müssen eine `LANGUAGE`-Klausel hinzufügen, weil PL/pgSQL nicht die einzige mögliche Funktionssprache ist.

- In PostgreSQL gilt der Funktionsrumpf als Zeichenkettenliteral, daher müssen Sie Anführungszeichen oder Dollar-Quotes darum setzen. Das ersetzt das abschließende `/` im Oracle-Ansatz.

- Der Befehl `show errors` existiert in PostgreSQL nicht und wird nicht benötigt, weil Fehler automatisch gemeldet werden.

So sähe diese Funktion portiert nach PostgreSQL aus:

```sql
CREATE OR REPLACE FUNCTION cs_fmt_browser_version(v_name varchar,
                                                  v_version
 varchar)
RETURNS varchar AS $$
BEGIN
     IF v_version IS NULL THEN
         RETURN v_name;
     END IF;
     RETURN v_name || '/' || v_version;
END;
$$ LANGUAGE plpgsql;
```

Beispiel 41.10 zeigt, wie eine Funktion portiert wird, die eine andere Funktion erzeugt, und wie die dabei entstehenden Quoting-Probleme behandelt werden.

**Beispiel 41.10. Portierung einer Funktion, die eine andere Funktion erzeugt, von PL/SQL nach PL/pgSQL**

Die folgende Prozedur holt Zeilen aus einer `SELECT`-Anweisung und baut der Effizienz halber aus den Ergebnissen in `IF`-Anweisungen eine große Funktion.

Dies ist die Oracle-Version:

```sql
CREATE OR REPLACE PROCEDURE cs_update_referrer_type_proc IS
    CURSOR referrer_keys IS
        SELECT * FROM cs_referrer_keys
        ORDER BY try_order;
    func_cmd VARCHAR(4000);
BEGIN
    func_cmd := 'CREATE OR REPLACE FUNCTION
 cs_find_referrer_type(v_host IN VARCHAR2,
                 v_domain IN VARCHAR2, v_url IN VARCHAR2) RETURN
 VARCHAR2 IS BEGIN';

      FOR referrer_key IN referrer_keys LOOP
          func_cmd := func_cmd ||
            ' IF v_' || referrer_key.kind
            || ' LIKE ''' || referrer_key.key_string
            || ''' THEN RETURN ''' || referrer_key.referrer_type
            || '''; END IF;';
      END LOOP;

      func_cmd := func_cmd || ' RETURN NULL; END;';

     EXECUTE IMMEDIATE func_cmd;
END;
/
show errors;
```

So würde diese Funktion in PostgreSQL aussehen:

```sql
CREATE OR REPLACE PROCEDURE cs_update_referrer_type_proc() AS $func
$
DECLARE
    referrer_keys CURSOR IS
        SELECT * FROM cs_referrer_keys
        ORDER BY try_order;
    func_body text;
    func_cmd text;
BEGIN
    func_body := 'BEGIN';

    FOR referrer_key IN referrer_keys LOOP
        func_body := func_body ||
          ' IF v_' || referrer_key.kind
          || ' LIKE ' || quote_literal(referrer_key.key_string)
          || ' THEN RETURN ' ||
 quote_literal(referrer_key.referrer_type)
          || '; END IF;' ;
    END LOOP;

      func_body := func_body || ' RETURN NULL; END;';

    func_cmd :=
      'CREATE OR REPLACE FUNCTION cs_find_referrer_type(v_host
 varchar,
                                                        v_domain
 varchar,
                                                        v_url
 varchar)
        RETURNS varchar AS '
      || quote_literal(func_body)
      || ' LANGUAGE plpgsql;' ;

     EXECUTE func_cmd;
END;
$func$ LANGUAGE plpgsql;
```

Beachten Sie, wie der Rumpf der Funktion separat aufgebaut und durch `quote_literal` geleitet wird, um alle darin enthaltenen Anführungszeichen zu verdoppeln. Diese Technik ist nötig, weil wir für die Definition der neuen Funktion nicht sicher Dollar-Quoting verwenden können: Wir wissen nicht sicher, welche Zeichenketten aus dem Feld `referrer_key.key_string` interpoliert werden. (Wir nehmen hier an, dass `referrer_key.kind` vertrauenswürdig ist und immer `host`, `domain` oder `url` ist, aber `referrer_key.key_string` könnte beliebig sein, insbesondere könnte es Dollarzeichen enthalten.) Diese Funktion ist tatsächlich eine Verbesserung gegenüber dem Oracle-Original, weil sie keinen defekten Code erzeugt, wenn `referrer_key.key_string` oder `referrer_key.referrer_type` Anführungszeichen enthalten.

Beispiel 41.11 zeigt, wie eine Funktion mit `OUT`-Parametern und Zeichenkettenmanipulation portiert wird. PostgreSQL hat keine eingebaute Funktion `instr`, aber Sie können eine mit einer Kombination anderer Funktionen erstellen. In [Abschnitt 41.13.3](#41133-anhang) gibt es eine PL/pgSQL-Implementierung von `instr`, die Sie verwenden können, um die Portierung zu erleichtern.

**Beispiel 41.11. Portierung einer Prozedur mit Zeichenkettenmanipulation und OUT-Parametern von PL/SQL nach PL/pgSQL**

Die folgende Oracle-PL/SQL-Prozedur wird verwendet, um eine URL zu parsen und mehrere Elemente zurückzugeben (`host`, `path` und `query`).

Dies ist die Oracle-Version:

```sql
CREATE OR REPLACE PROCEDURE cs_parse_url(
    v_url IN VARCHAR2,
    v_host OUT VARCHAR2, -- This will be passed back
    v_path OUT VARCHAR2, -- This one too
    v_query OUT VARCHAR2) -- And this one
IS
    a_pos1 INTEGER;
    a_pos2 INTEGER;
BEGIN
    v_host := NULL;
    v_path := NULL;
    v_query := NULL;
    a_pos1 := instr(v_url, '//');

      IF a_pos1 = 0 THEN
          RETURN;
      END IF;
      a_pos2 := instr(v_url, '/', a_pos1 + 2);
      IF a_pos2 = 0 THEN
          v_host := substr(v_url, a_pos1 + 2);
          v_path := '/';
          RETURN;
      END IF;

      v_host := substr(v_url, a_pos1 + 2, a_pos2 - a_pos1 - 2);
      a_pos1 := instr(v_url, '?', a_pos2 + 1);

      IF a_pos1 = 0 THEN
          v_path := substr(v_url, a_pos2);
          RETURN;
      END IF;

      v_path := substr(v_url, a_pos2, a_pos1 - a_pos2);
      v_query := substr(v_url, a_pos1 + 1);
END;
/
show errors;
```

Hier ist eine mögliche Übersetzung nach PL/pgSQL:

```sql
CREATE OR REPLACE FUNCTION cs_parse_url(
    v_url IN VARCHAR,
    v_host OUT VARCHAR, -- This will be passed back
    v_path OUT VARCHAR, -- This one too
    v_query OUT VARCHAR) -- And this one
AS $$
DECLARE
    a_pos1 INTEGER;
    a_pos2 INTEGER;
BEGIN
    v_host := NULL;
    v_path := NULL;
    v_query := NULL;
    a_pos1 := instr(v_url, '//');

      IF a_pos1 = 0 THEN
          RETURN;
      END IF;
      a_pos2 := instr(v_url, '/', a_pos1 + 2);
      IF a_pos2 = 0 THEN
          v_host := substr(v_url, a_pos1 + 2);
          v_path := '/';
          RETURN;
      END IF;

      v_host := substr(v_url, a_pos1 + 2, a_pos2 - a_pos1 - 2);
      a_pos1 := instr(v_url, '?', a_pos2 + 1);

      IF a_pos1 = 0 THEN
          v_path := substr(v_url, a_pos2);
          RETURN;
      END IF;

      v_path := substr(v_url, a_pos2, a_pos1 - a_pos2);
      v_query := substr(v_url, a_pos1 + 1);
END;
$$ LANGUAGE plpgsql;
```

Diese Funktion könnte so verwendet werden:

```sql
SELECT * FROM cs_parse_url('http://foobar.com/query.cgi?baz');
```

Beispiel 41.12 zeigt, wie eine Prozedur portiert wird, die zahlreiche Oracle-spezifische Funktionen verwendet.

**Beispiel 41.12. Portierung einer Prozedur von PL/SQL nach PL/pgSQL**

Die Oracle-Version:

```sql
CREATE OR REPLACE PROCEDURE cs_create_job(v_job_id IN INTEGER) IS
    a_running_job_count INTEGER;
BEGIN
    LOCK TABLE cs_jobs IN EXCLUSIVE MODE;

    SELECT count(*) INTO a_running_job_count FROM cs_jobs WHERE
 end_stamp IS NULL;

    IF a_running_job_count > 0 THEN
        COMMIT; -- free lock
        raise_application_error(-20000,
                 'Unable to create a new job: a job is currently
 running.');
    END IF;

      DELETE FROM cs_active_job;
      INSERT INTO cs_active_job(job_id) VALUES (v_job_id);

    BEGIN
         INSERT INTO cs_jobs (job_id, start_stamp) VALUES (v_job_id,
 now());
    EXCEPTION
         WHEN dup_val_on_index THEN NULL; -- don't worry if it
 already exists
    END;
    COMMIT;
END;
/
show errors
```

So könnten wir diese Prozedur nach PL/pgSQL portieren:

```sql
CREATE OR REPLACE PROCEDURE cs_create_job(v_job_id integer) AS $$
DECLARE
    a_running_job_count integer;
BEGIN
    LOCK TABLE cs_jobs IN EXCLUSIVE MODE;

       SELECT count(*) INTO a_running_job_count FROM cs_jobs WHERE
    end_stamp IS NULL;

       IF a_running_job_count > 0 THEN
           COMMIT; -- free lock
           RAISE EXCEPTION 'Unable to create a new job: a job is
    currently running'; -- 1
       END IF;

       DELETE FROM cs_active_job;
       INSERT INTO cs_active_job(job_id) VALUES (v_job_id);

     BEGIN
          INSERT INTO cs_jobs (job_id, start_stamp) VALUES (v_job_id,
 now());
     EXCEPTION
          WHEN unique_violation THEN -- 2
              -- don't worry if it already exists
     END;
     COMMIT;
END;
$$ LANGUAGE plpgsql;
```

1. Die Syntax von `RAISE` unterscheidet sich erheblich von Oracles Anweisung, obwohl der einfache Fall `RAISE exception_name` ähnlich funktioniert.

2. Die von PL/pgSQL unterstützten Exception-Namen unterscheiden sich von denen Oracles. Die Menge der eingebauten Exception-Namen ist viel größer (siehe [Anhang A](A_PostgreSQL_Fehlercodes.md)). Derzeit gibt es keine Möglichkeit, benutzerdefinierte Exception-Namen zu deklarieren, obwohl Sie stattdessen selbst gewählte SQLSTATE-Werte auslösen können.

### 41.13.2. Weitere Dinge, auf die zu achten ist

Dieser Abschnitt erklärt einige weitere Dinge, auf die Sie beim Portieren von Oracle-PL/SQL-Funktionen nach PostgreSQL achten sollten.

#### 41.13.2.1. Implizites Rollback nach Exceptions

Wenn in PL/pgSQL eine Exception von einer `EXCEPTION`-Klausel abgefangen wird, werden alle Datenbankänderungen seit dem `BEGIN` des Blocks automatisch zurückgerollt. Das Verhalten entspricht also dem, was Sie in Oracle mit Folgendem erhalten würden:

```sql
BEGIN
    SAVEPOINT s1;
    ... code here ...
EXCEPTION
    WHEN ... THEN
        ROLLBACK TO s1;
        ... code here ...
    WHEN ... THEN
        ROLLBACK TO s1;
        ... code here ...
END;
```

Wenn Sie eine Oracle-Prozedur übersetzen, die `SAVEPOINT` und `ROLLBACK TO` in diesem Stil verwendet, ist Ihre Aufgabe einfach: Lassen Sie `SAVEPOINT` und `ROLLBACK TO` einfach weg. Wenn Sie eine Prozedur haben, die `SAVEPOINT` und `ROLLBACK TO` auf andere Weise verwendet, ist etwas echtes Nachdenken erforderlich.

#### 41.13.2.2. EXECUTE

Die PL/pgSQL-Version von `EXECUTE` funktioniert ähnlich wie die PL/SQL-Version, aber Sie müssen daran denken, `quote_literal` und `quote_ident` wie in [Abschnitt 41.5.4](#4154-dynamische-befehle-ausführen) beschrieben zu verwenden. Konstrukte der Form `EXECUTE 'SELECT * FROM $1';` funktionieren nicht zuverlässig, wenn Sie diese Funktionen nicht verwenden.

#### 41.13.2.3. PL/pgSQL-Funktionen optimieren

PostgreSQL bietet Ihnen zwei Modifikatoren beim Erzeugen von Funktionen, um die Ausführung zu optimieren: „Volatility“ (ob die Funktion bei gleichen Argumenten immer dasselbe Ergebnis zurückgibt) und „Strictness“ (ob die Funktion null zurückgibt, wenn ein Argument null ist). Einzelheiten finden Sie auf der Referenzseite zu `CREATE FUNCTION`.

Wenn Sie diese Optimierungsattribute verwenden, könnte Ihre Anweisung `CREATE FUNCTION` ungefähr so aussehen:

```sql
CREATE FUNCTION foo(...) RETURNS integer AS $$
...
$$ LANGUAGE plpgsql STRICT IMMUTABLE;
```

### 41.13.3. Anhang

Dieser Abschnitt enthält den Code für eine Menge Oracle-kompatibler `instr`-Funktionen, die Sie verwenden können, um Ihre Portierungsarbeit zu vereinfachen.

```sql
--
-- instr functions that mimic Oracle's counterpart
-- Syntax: instr(string1, string2 [, n [, m]])
-- where [] denotes optional parameters.
--
-- Search string1, beginning at the nth character, for the mth
 occurrence
-- of string2. If n is negative, search backwards, starting at the
 abs(n)'th
-- character from the end of string1.
-- If n is not passed, assume 1 (search starts at first character).
-- If m is not passed, assume 1 (find first occurrence).
-- Returns starting index of string2 in string1, or 0 if string2 is
 not found.
--

CREATE FUNCTION instr(varchar, varchar) RETURNS integer AS $$
BEGIN
    RETURN instr($1, $2, 1);
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;

CREATE FUNCTION instr(string varchar, string_to_search_for varchar,
                       beg_index integer)
RETURNS integer AS $$
DECLARE
    pos integer NOT NULL DEFAULT 0;
    temp_str varchar;
    beg integer;
    length integer;
    ss_length integer;
BEGIN
    IF beg_index > 0 THEN
        temp_str := substring(string FROM beg_index);
        pos := position(string_to_search_for IN temp_str);

        IF pos = 0 THEN
             RETURN 0;
        ELSE
             RETURN pos + beg_index - 1;
        END IF;
    ELSIF beg_index < 0 THEN
        ss_length := char_length(string_to_search_for);
        length := char_length(string);
        beg := length + 1 + beg_index;

        WHILE beg > 0 LOOP
            temp_str := substring(string FROM beg FOR ss_length);
            IF string_to_search_for = temp_str THEN
                RETURN beg;
            END IF;

            beg := beg - 1;
        END LOOP;

        RETURN 0;
    ELSE
        RETURN 0;
    END IF;
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;

CREATE FUNCTION instr(string varchar, string_to_search_for varchar,
                       beg_index integer, occur_index integer)
RETURNS integer AS $$
DECLARE
    pos integer NOT NULL DEFAULT 0;
    occur_number integer NOT NULL DEFAULT 0;
    temp_str varchar;
    beg integer;
    i integer;
    length integer;
    ss_length integer;
BEGIN
    IF occur_index <= 0 THEN
        RAISE 'argument ''%'' is out of range', occur_index
          USING ERRCODE = '22003';
    END IF;

    IF beg_index > 0 THEN
        beg := beg_index - 1;
        FOR i IN 1..occur_index LOOP
            temp_str := substring(string FROM beg + 1);
            pos := position(string_to_search_for IN temp_str);
            IF pos = 0 THEN
                RETURN 0;
            END IF;
            beg := beg + pos;
        END LOOP;

        RETURN beg;
    ELSIF beg_index < 0 THEN
        ss_length := char_length(string_to_search_for);
        length := char_length(string);
        beg := length + 1 + beg_index;

        WHILE beg > 0 LOOP
            temp_str := substring(string FROM beg FOR ss_length);
            IF string_to_search_for = temp_str THEN
                occur_number := occur_number + 1;
                IF occur_number = occur_index THEN
                    RETURN beg;
                END IF;
            END IF;

            beg := beg - 1;
        END LOOP;

        RETURN 0;
    ELSE
        RETURN 0;
    END IF;
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;
```
