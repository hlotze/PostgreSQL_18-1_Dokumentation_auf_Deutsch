# 44. PL/Python - Python-Prozedursprache

Die prozedurale Sprache PL/Python erlaubt es, PostgreSQL-Funktionen und -Prozeduren in der Sprache Python zu schreiben.

Um PL/Python in einer bestimmten Datenbank zu installieren, verwenden Sie `CREATE EXTENSION plpython3u`.

> Wenn eine Sprache in `template1` installiert wird, haben alle danach erzeugten Datenbanken diese Sprache automatisch installiert.

PL/Python ist nur als „nicht vertrauenswürdige“ Sprache verfügbar. Das bedeutet, dass es keine Möglichkeit bietet, einzuschränken, was Benutzer darin tun können; deshalb heißt es `plpython3u`. Eine vertrauenswürdige Variante `plpython` könnte in Zukunft verfügbar werden, falls in Python ein sicherer Ausführungsmechanismus entwickelt wird. Der Autor einer Funktion in nicht vertrauenswürdigem PL/Python muss darauf achten, dass die Funktion nicht für unerwünschte Zwecke verwendet werden kann, da sie alles tun kann, was ein als Datenbankadministrator angemeldeter Benutzer tun könnte. Nur Superuser können Funktionen in nicht vertrauenswürdigen Sprachen wie `plpython3u` erstellen.

> Benutzer von Quellpaketen müssen den Bau von PL/Python während des Installationsprozesses ausdrücklich aktivieren. (Weitere Informationen finden Sie in den Installationsanweisungen.) Benutzer von Binärpaketen finden PL/Python möglicherweise in einem separaten Unterpaket.

## 44.1. PL/Python-Funktionen

Funktionen in PL/Python werden mit der Standard-Syntax von `CREATE FUNCTION` deklariert:

```sql
CREATE FUNCTION funcname (argument-list)
  RETURNS return-type
AS $$
  # PL/Python function body
$$ LANGUAGE plpython3u;
```

Der Rumpf einer Funktion ist einfach ein Python-Skript. Wenn die Funktion aufgerufen wird, werden ihre Argumente als Elemente der Liste `args` übergeben; benannte Argumente werden außerdem als gewöhnliche Variablen an das Python-Skript übergeben. Die Verwendung benannter Argumente ist normalerweise besser lesbar. Das Ergebnis wird aus dem Python-Code auf die übliche Weise mit `return` oder `yield` zurückgegeben (im Fall einer Ergebnismengenanweisung). Wenn Sie keinen Rückgabewert angeben, gibt Python standardmäßig `None` zurück. PL/Python übersetzt Pythons `None` in den SQL-Nullwert. In einer Prozedur muss das Ergebnis des Python-Codes `None` sein (typischerweise erreicht, indem die Prozedur ohne `return`-Anweisung endet oder indem eine `return`-Anweisung ohne Argument verwendet wird); andernfalls wird ein Fehler ausgelöst.

Zum Beispiel kann eine Funktion, die den größeren von zwei Ganzzahlen zurückgibt, so definiert werden:

```sql
CREATE FUNCTION pymax (a integer, b integer)
  RETURNS integer
AS $$
  if a > b:
    return a
  return b
$$ LANGUAGE plpython3u;
```

Der Python-Code, der als Rumpf der Funktionsdefinition angegeben wird, wird in eine Python-Funktion umgewandelt. Das obige Beispiel ergibt zum Beispiel:

```sql
def __plpython_procedure_pymax_23456():
  if a > b:
    return a
  return b
```

wobei angenommen wird, dass `23456` die OID ist, die PostgreSQL der Funktion zugewiesen hat.

Die Argumente werden als globale Variablen gesetzt. Aufgrund der Geltungsbereichsregeln von Python hat dies die subtile Folge, dass eine Argumentvariable innerhalb der Funktion nicht einem Ausdruckswert zugewiesen werden kann, der den Variablennamen selbst verwendet, es sei denn, die Variable wird im Block erneut als `global` deklariert. Zum Beispiel funktioniert Folgendes nicht:

```sql
CREATE FUNCTION pystrip(x text)
  RETURNS text
AS $$
  x = x.strip() # error
  return x
$$ LANGUAGE plpython3u;
```

Denn die Zuweisung an `x` macht `x` für den gesamten Block zu einer lokalen Variablen, sodass sich das `x` auf der rechten Seite der Zuweisung auf eine noch nicht zugewiesene lokale Variable `x` bezieht, nicht auf den PL/Python-Funktionsparameter. Mit der Anweisung `global` kann dies zum Laufen gebracht werden:

```sql
CREATE FUNCTION pystrip(x text)
  RETURNS text
AS $$
  global x
  x = x.strip() # ok now
  return x
$$ LANGUAGE plpython3u;
```

Es ist jedoch ratsam, sich nicht auf dieses Implementierungsdetail von PL/Python zu verlassen. Es ist besser, Funktionsparameter als schreibgeschützt zu behandeln.

## 44.2. Datenwerte

Allgemein gesprochen ist das Ziel von PL/Python, eine „natürliche“ Abbildung zwischen der PostgreSQL- und der Python-Welt bereitzustellen. Daraus ergeben sich die unten beschriebenen Regeln für die Datenabbildung.

### 44.2.1. Datentypabbildung

Wenn eine PL/Python-Funktion aufgerufen wird, werden ihre Argumente von ihrem PostgreSQL-Datentyp in einen entsprechenden Python-Typ konvertiert:

- PostgreSQL `boolean` wird in Python `bool` konvertiert.

- PostgreSQL `smallint`, `int`, `bigint` und `oid` werden in Python `int` konvertiert.

- PostgreSQL `real` und `double` werden in Python `float` konvertiert.

- PostgreSQL `numeric` wird in Python `Decimal` konvertiert. Dieser Typ wird aus dem Paket `cdecimal` importiert, falls dieses verfügbar ist. Andernfalls wird `decimal.Decimal` aus der Standardbibliothek verwendet. `cdecimal` ist deutlich schneller als `decimal`. In Python 3.3 und höher wurde `cdecimal` jedoch unter dem Namen `decimal` in die Standardbibliothek integriert, sodass kein Unterschied mehr besteht.

- PostgreSQL `bytea` wird in Python `bytes` konvertiert.

- Alle anderen Datentypen, einschließlich der PostgreSQL-Zeichenkettentypen, werden in Python `str` konvertiert (in Unicode, wie alle Python-Zeichenketten).

- Informationen zu nichtskalaren Datentypen finden Sie unten.

Wenn eine PL/Python-Funktion zurückkehrt, wird ihr Rückgabewert wie folgt in den deklarierten PostgreSQL-Rückgabetyp der Funktion konvertiert:

- Wenn der PostgreSQL-Rückgabetyp `boolean` ist, wird der Rückgabewert nach den Python-Regeln auf Wahrheitswert ausgewertet. Das heißt, `0` und die leere Zeichenkette sind falsch, aber insbesondere ist `'f'` wahr.

- Wenn der PostgreSQL-Rückgabetyp `bytea` ist, wird der Rückgabewert mit den jeweiligen Python-Built-ins in Python `bytes` konvertiert; das Ergebnis wird anschließend in `bytea` konvertiert.

- Für alle anderen PostgreSQL-Rückgabetypen wird der Rückgabewert mit dem Python-Built-in `str` in eine Zeichenkette konvertiert, und das Ergebnis wird an die Eingabefunktion des PostgreSQL-Datentyps übergeben. (Wenn der Python-Wert ein `float` ist, wird er mit dem Built-in `repr` statt mit `str` konvertiert, um Genauigkeitsverlust zu vermeiden.)

  Zeichenketten werden automatisch in die Serverkodierung von PostgreSQL konvertiert, wenn sie an PostgreSQL übergeben werden.

- Informationen zu nichtskalaren Datentypen finden Sie unten.

Beachten Sie, dass logische Nichtübereinstimmungen zwischen dem deklarierten PostgreSQL-Rückgabetyp und dem Python-Datentyp des tatsächlich zurückgegebenen Objekts nicht gemeldet werden; der Wert wird in jedem Fall konvertiert.

### 44.2.2. Null, None

Wenn ein SQL-Nullwert an eine Funktion übergeben wird, erscheint der Argumentwert in Python als `None`. Zum Beispiel gibt die in [Abschnitt 44.1](#441-plpythonfunktionen) gezeigte Funktionsdefinition von `pymax` bei Null-Eingaben die falsche Antwort zurück. Wir könnten der Funktionsdefinition `STRICT` hinzufügen, damit PostgreSQL etwas Vernünftigeres tut: Wenn ein Nullwert übergeben wird, wird die Funktion gar nicht aufgerufen, sondern gibt automatisch ein Nullergebnis zurück. Alternativ könnten wir im Funktionsrumpf auf Null-Eingaben prüfen:

```sql
CREATE FUNCTION pymax (a integer, b integer)
  RETURNS integer
AS $$
  if (a is None) or (b is None):
    return None
  if a > b:
    return a
  return b
$$ LANGUAGE plpython3u;
```

Wie oben gezeigt, geben Sie aus einer PL/Python-Funktion den Wert `None` zurück, um einen SQL-Nullwert zurückzugeben. Das ist unabhängig davon möglich, ob die Funktion strict ist oder nicht.

### 44.2.3. Arrays, Listen

SQL-Array-Werte werden an PL/Python als Python-Liste übergeben. Um einen SQL-Array-Wert aus einer PL/Python-Funktion zurückzugeben, geben Sie eine Python-Liste zurück:

```sql
CREATE FUNCTION return_arr()
  RETURNS int[]
AS $$
return [1, 2, 3, 4, 5]
$$ LANGUAGE plpython3u;

SELECT return_arr();
```

```text
 return_arr
-------------
 {1,2,3,4,5}
(1 row)
```

Mehrdimensionale Arrays werden an PL/Python als verschachtelte Python-Listen übergeben. Ein zweidimensionales Array ist zum Beispiel eine Liste von Listen. Wenn eine PL/Python-Funktion ein mehrdimensionales SQL-Array zurückgibt, müssen die inneren Listen auf jeder Ebene alle dieselbe Größe haben. Zum Beispiel:

```sql
CREATE FUNCTION test_type_conversion_array_int4(x int4[]) RETURNS
 int4[] AS $$
plpy.info(x, type(x))
return x
$$ LANGUAGE plpython3u;

SELECT * FROM test_type_conversion_array_int4(ARRAY[[1,2,3],
[4,5,6]]);
```

```text
INFO: ([[1, 2, 3], [4, 5, 6]], <type 'list'>)
 test_type_conversion_array_int4
---------------------------------
 {{1,2,3},{4,5,6}}
(1 row)
```

Andere Python-Sequenzen, etwa Tupel, werden aus Gründen der Rückwärtskompatibilität mit PostgreSQL-Versionen 9.6 und älter ebenfalls akzeptiert, als mehrdimensionale Arrays noch nicht unterstützt wurden. Sie werden jedoch immer als eindimensionale Arrays behandelt, weil sie mit zusammengesetzten Typen mehrdeutig sind. Aus demselben Grund muss ein zusammengesetzter Typ, der in einem mehrdimensionalen Array verwendet wird, durch ein Tupel dargestellt werden, nicht durch eine Liste.

Beachten Sie, dass Zeichenketten in Python Sequenzen sind; das kann unerwünschte Effekte haben, die Python-Programmierern vertraut sein könnten:

```sql
CREATE FUNCTION return_str_arr()
  RETURNS varchar[]
AS $$
return "hello"
$$ LANGUAGE plpython3u;

SELECT return_str_arr();
```

```text
 return_str_arr
----------------
 {h,e,l,l,o}
(1 row)
```

### 44.2.4. Zusammengesetzte Typen

Argumente mit zusammengesetztem Typ werden der Funktion als Python-Mappings übergeben. Die Elementnamen des Mappings sind die Attributnamen des zusammengesetzten Typs. Wenn ein Attribut in der übergebenen Zeile den Nullwert hat, hat es im Mapping den Wert `None`. Hier ist ein Beispiel:

```sql
CREATE TABLE employee (
   name text,
   salary integer,
   age integer
);

CREATE FUNCTION overpaid (e employee)
  RETURNS boolean
AS $$
  if e["salary"] > 200000:
    return True
  if (e["age"] < 30) and (e["salary"] > 100000):
    return True
  return False
$$ LANGUAGE plpython3u;
```

Es gibt mehrere Möglichkeiten, Zeilen- oder zusammengesetzte Typen aus einer Python-Funktion zurückzugeben. Die folgenden Beispiele setzen voraus, dass wir Folgendes haben:

```sql
CREATE TYPE named_value AS (
   name  text,
   value integer
);
```

Ein zusammengesetztes Ergebnis kann zurückgegeben werden als:

- Sequenztyp (ein Tupel oder eine Liste, aber kein Set, weil es nicht indizierbar ist)
  - Zurückgegebene Sequenzobjekte müssen genauso viele Elemente haben, wie der zusammengesetzte Ergebnistyp Felder hat. Das Element mit Index 0 wird dem ersten Feld des zusammengesetzten Typs zugewiesen, 1 dem zweiten und so weiter. Zum Beispiel:

    ```sql
    CREATE FUNCTION make_pair (name text, value integer)
      RETURNS named_value
    AS $$
      return ( name, value )
      # or alternatively, as list: return [ name, value ]
    $$ LANGUAGE plpython3u;
    ```

    Um für eine Spalte einen SQL-Nullwert zurückzugeben, fügen Sie an der entsprechenden Position `None` ein.

    Wenn ein Array zusammengesetzter Typen zurückgegeben wird, kann es nicht als Liste zurückgegeben werden, weil mehrdeutig wäre, ob die Python-Liste einen zusammengesetzten Typ oder eine weitere Array-Dimension darstellt.

- Mapping (Dictionary)
  - Der Wert für jede Spalte des Ergebnistyps wird aus dem Mapping mit dem Spaltennamen als Schlüssel abgerufen. Beispiel:

    ```sql
    CREATE FUNCTION make_pair (name text, value integer)
      RETURNS named_value
    AS $$
      return { "name": name, "value": value }
    $$ LANGUAGE plpython3u;
    ```

    Alle zusätzlichen Dictionary-Schlüssel/Wert-Paare werden ignoriert. Fehlende Schlüssel werden als Fehler behandelt. Um für eine Spalte einen SQL-Nullwert zurückzugeben, fügen Sie `None` mit dem entsprechenden Spaltennamen als Schlüssel ein.

- Objekt (jedes Objekt, das die Methode `__getattr__` bereitstellt)
  - Dies funktioniert genauso wie ein Mapping. Beispiel:

    ```sql
    CREATE FUNCTION make_pair (name text, value integer)
      RETURNS named_value
    AS $$
      class named_value:
        def __init__ (self, n, v):
          self.name = n
          self.value = v
      return named_value(name, value)

      # or simply
      class nv: pass
      nv.name = name
      nv.value = value
      return nv
    $$ LANGUAGE plpython3u;
    ```

Funktionen mit `OUT`-Parametern werden ebenfalls unterstützt. Zum Beispiel:

```sql
CREATE FUNCTION multiout_simple(OUT i integer, OUT j integer) AS $$
return (1, 2)
$$ LANGUAGE plpython3u;

SELECT * FROM multiout_simple();
```

Ausgabeparameter von Prozeduren werden auf dieselbe Weise zurückgegeben. Zum Beispiel:

```sql
CREATE PROCEDURE python_triple(INOUT a integer, INOUT b integer) AS
 $$
return (a * 3, b * 3)
$$ LANGUAGE plpython3u;

CALL python_triple(5, 10);
```

### 44.2.5. Funktionen, die Mengen zurückgeben

Eine PL/Python-Funktion kann außerdem Mengen skalarer oder zusammengesetzter Typen zurückgeben. Dafür gibt es mehrere Möglichkeiten, weil das zurückgegebene Objekt intern in einen Iterator umgewandelt wird. Die folgenden Beispiele setzen voraus, dass wir einen zusammengesetzten Typ haben:

```sql
CREATE TYPE greeting AS (
  how text,
  who text
);
```

Ein Mengenergebnis kann zurückgegeben werden aus:

- Sequenztyp (Tupel, Liste, Set)

  ```sql
  CREATE FUNCTION greet (how text)
    RETURNS SETOF greeting
  AS $$
    # return tuple containing lists as composite types
    # all other combinations work also
    return ( [ how, "World" ], [ how, "PostgreSQL" ], [ how, "PL/Python" ] )
  $$ LANGUAGE plpython3u;
  ```

- Iterator (jedes Objekt, das die Methoden `__iter__` und `__next__` bereitstellt)

  ```sql
  CREATE FUNCTION greet (how text)
    RETURNS SETOF greeting
  AS $$
    class producer:
      def __init__ (self, how, who):
        self.how = how
        self.who = who
        self.ndx = -1

        def __iter__ (self):
          return self

        def __next__(self):
          self.ndx += 1
          if self.ndx == len(self.who):
            raise StopIteration
          return ( self.how, self.who[self.ndx] )

    return producer(how, [ "World", "PostgreSQL", "PL/Python" ])
  $$ LANGUAGE plpython3u;
  ```

- Generator (`yield`)

  ```sql
  CREATE FUNCTION greet (how text)
    RETURNS SETOF greeting
  AS $$
    for who in [ "World", "PostgreSQL", "PL/Python" ]:
      yield ( how, who )
  $$ LANGUAGE plpython3u;
  ```

Mengenrückgebende Funktionen mit `OUT`-Parametern (unter Verwendung von `RETURNS SETOF record`) werden ebenfalls unterstützt. Zum Beispiel:

```sql
CREATE FUNCTION multiout_simple_setof(n integer, OUT integer, OUT
 integer) RETURNS SETOF record AS $$
return [(1, 2)] * n
$$ LANGUAGE plpython3u;

SELECT * FROM multiout_simple_setof(3);
```

## 44.3. Daten gemeinsam nutzen

Das globale Dictionary `SD` steht zur Verfügung, um private Daten zwischen wiederholten Aufrufen derselben Funktion zu speichern. Das globale Dictionary `GD` ist öffentliche Daten, die allen Python-Funktionen innerhalb einer Sitzung zur Verfügung stehen; verwenden Sie es mit Vorsicht.

Jede Funktion erhält ihre eigene Ausführungsumgebung im Python-Interpreter, sodass globale Daten und Funktionsargumente aus `myfunc` für `myfunc2` nicht verfügbar sind. Die Ausnahme sind, wie oben erwähnt, Daten im Dictionary `GD`.

## 44.4. Anonyme Codeblöcke

PL/Python unterstützt außerdem anonyme Codeblöcke, die mit der Anweisung `DO` aufgerufen werden:

```sql
DO $$
    # PL/Python code
$$ LANGUAGE plpython3u;
```

Ein anonymer Codeblock erhält keine Argumente, und jeder Wert, den er eventuell zurückgibt, wird verworfen. Ansonsten verhält er sich genau wie eine Funktion.

## 44.5. Trigger-Funktionen

Wenn eine Funktion als Trigger verwendet wird, enthält das Dictionary `TD` triggerbezogene Werte:

- `TD["event"]`
  - Enthält das Ereignis als Zeichenkette: `INSERT`, `UPDATE`, `DELETE` oder `TRUNCATE`.

- `TD["when"]`
  - Enthält einen der Werte `BEFORE`, `AFTER` oder `INSTEAD OF`.

- `TD["level"]`
  - Enthält `ROW` oder `STATEMENT`.

- `TD["new"]`
- `TD["old"]`
  - Bei einem Trigger auf Zeilenebene enthalten eines oder beide dieser Felder die jeweiligen Trigger-Zeilen, abhängig vom Trigger-Ereignis.

- `TD["name"]`
  - Enthält den Triggernamen.

- `TD["table_name"]`
  - Enthält den Namen der Tabelle, auf der der Trigger aufgetreten ist.

- `TD["table_schema"]`
  - Enthält das Schema der Tabelle, auf der der Trigger aufgetreten ist.

- `TD["relid"]`
  - Enthält die OID der Tabelle, auf der der Trigger aufgetreten ist.

- `TD["args"]`
  - Wenn der Befehl `CREATE TRIGGER` Argumente enthielt, stehen sie in `TD["args"][0]` bis `TD["args"][n-1]` zur Verfügung.

Wenn `TD["when"]` `BEFORE` oder `INSTEAD OF` ist und `TD["level"]` `ROW` ist, können Sie aus der Python-Funktion `None` oder `"OK"` zurückgeben, um anzuzeigen, dass die Zeile unverändert ist, `"SKIP"`, um das Ereignis abzubrechen, oder, wenn `TD["event"]` `INSERT` oder `UPDATE` ist, `"MODIFY"`, um anzuzeigen, dass Sie die neue Zeile geändert haben. Andernfalls wird der Rückgabewert ignoriert.

## 44.6. Datenbankzugriff

Das PL/Python-Sprachmodul importiert automatisch ein Python-Modul namens `plpy`. Die Funktionen und Konstanten in diesem Modul stehen Ihnen im Python-Code als `plpy.foo` zur Verfügung.

### 44.6.1. Datenbankzugriffsfunktionen

Das Modul `plpy` stellt mehrere Funktionen bereit, um Datenbankbefehle auszuführen:

- `plpy.execute(query [, limit])`
  - Der Aufruf von `plpy.execute` mit einer Abfragezeichenkette und einem optionalen Argument für die Zeilenbegrenzung führt diese Abfrage aus und gibt das Ergebnis in einem Ergebnisobjekt zurück.

    Wenn `limit` angegeben und größer als null ist, ruft `plpy.execute` höchstens `limit` Zeilen ab, ähnlich als enthielte die Abfrage eine `LIMIT`-Klausel. Wird `limit` weggelassen oder als null angegeben, gibt es keine Zeilenbegrenzung.

    Das Ergebnisobjekt emuliert ein Listen- oder Dictionary-Objekt. Auf das Ergebnisobjekt kann über Zeilennummer und Spaltenname zugegriffen werden. Zum Beispiel:

    ```sql
    rv = plpy.execute("SELECT * FROM my_table", 5)
    ```

    gibt bis zu 5 Zeilen aus `my_table` zurück. Wenn `my_table` eine Spalte `my_column` hat, würde darauf so zugegriffen:

    ```sql
    foo = rv[i]["my_column"]
    ```

    Die Anzahl der zurückgegebenen Zeilen kann mit der Built-in-Funktion `len` ermittelt werden.

    Das Ergebnisobjekt hat diese zusätzlichen Methoden:

    - `nrows()`
      - Gibt die Anzahl der vom Befehl verarbeiteten Zeilen zurück. Beachten Sie, dass dies nicht unbedingt dieselbe Zahl ist wie die Anzahl der zurückgegebenen Zeilen. Zum Beispiel setzt ein `UPDATE`-Befehl diesen Wert, gibt aber keine Zeilen zurück (sofern nicht `RETURNING` verwendet wird).

    - `status()`
      - Der Rückgabewert von `SPI_execute()`.

    - `colnames()`
    - `coltypes()`
    - `coltypmods()`
      - Geben jeweils eine Liste von Spaltennamen, eine Liste von Spaltentyp-OIDs und eine Liste typspezifischer Typmodifikatoren für die Spalten zurück.

        Diese Methoden lösen eine Exception aus, wenn sie auf einem Ergebnisobjekt eines Befehls aufgerufen werden, der keine Ergebnismenge erzeugt hat, zum Beispiel `UPDATE` ohne `RETURNING` oder `DROP TABLE`. Es ist aber in Ordnung, diese Methoden auf einer Ergebnismenge aufzurufen, die null Zeilen enthält.

    - `__str__()`
      - Die Standardmethode `__str__` ist definiert, sodass es zum Beispiel möglich ist, Abfrageausführungsergebnisse mit `plpy.debug(rv)` zu debuggen.

    Das Ergebnisobjekt kann geändert werden.

    Beachten Sie, dass ein Aufruf von `plpy.execute` dazu führt, dass die gesamte Ergebnismenge in den Speicher gelesen wird. Verwenden Sie diese Funktion nur, wenn Sie sicher sind, dass die Ergebnismenge relativ klein ist. Wenn Sie beim Abrufen großer Ergebnisse keinen übermäßigen Speicherverbrauch riskieren möchten, verwenden Sie `plpy.cursor` statt `plpy.execute`.

- `plpy.prepare(query [, argtypes])`
- `plpy.execute(plan [, arguments [, limit]])`
  - `plpy.prepare` bereitet den Ausführungsplan für eine Abfrage vor. Es wird mit einer Abfragezeichenkette und einer Liste von Parametertypen aufgerufen, wenn Sie Parameterreferenzen in der Abfrage haben. Zum Beispiel:

    ```sql
    plan = plpy.prepare("SELECT last_name FROM my_users WHERE first_name = $1", ["text"])
    ```

    `text` ist der Typ der Variablen, die Sie für `$1` übergeben werden. Das zweite Argument ist optional, wenn Sie keine Parameter an die Abfrage übergeben möchten.

    Nachdem eine Anweisung vorbereitet wurde, verwenden Sie eine Variante der Funktion `plpy.execute`, um sie auszuführen:

    ```sql
    rv = plpy.execute(plan, ["name"], 5)
    ```

    Übergeben Sie den Plan als erstes Argument (anstelle der Abfragezeichenkette) und eine Liste von Werten, die in die Abfrage eingesetzt werden sollen, als zweites Argument. Das zweite Argument ist optional, wenn die Abfrage keine Parameter erwartet. Das dritte Argument ist wie zuvor die optionale Zeilenbegrenzung.

    Alternativ können Sie die Methode `execute` des Planobjekts aufrufen:

    ```sql
    rv = plan.execute(["name"], 5)
    ```

    Abfrageparameter und Ergebniszeilenfelder werden zwischen PostgreSQL- und Python-Datentypen konvertiert, wie in [Abschnitt 44.2](#442-datenwerte) beschrieben.

    Wenn Sie mit dem PL/Python-Modul einen Plan vorbereiten, wird er automatisch gespeichert. Lesen Sie die SPI-Dokumentation ([Kapitel 45](45_Server_Programming_Interface.md)) für eine Beschreibung, was das bedeutet. Um dies über Funktionsaufrufe hinweg effektiv zu nutzen, muss eines der persistenten Speicher-Dictionaries `SD` oder `GD` verwendet werden (siehe [Abschnitt 44.3](#443-daten-gemeinsam-nutzen)). Zum Beispiel:

    ```sql
    CREATE FUNCTION usesavedplan() RETURNS trigger AS $$
    if "plan" in SD:
        plan = SD["plan"]
    else:
        plan = plpy.prepare("SELECT 1")
        SD["plan"] = plan
    # rest of function
    $$ LANGUAGE plpython3u;
    ```

- `plpy.cursor(query)`
- `plpy.cursor(plan [, arguments])`
  - Die Funktion `plpy.cursor` akzeptiert dieselben Argumente wie `plpy.execute` (außer der Zeilenbegrenzung) und gibt ein Cursor-Objekt zurück, mit dem Sie große Ergebnismengen in kleineren Blöcken verarbeiten können. Wie bei `plpy.execute` kann entweder eine Abfragezeichenkette oder ein Planobjekt zusammen mit einer Argumentliste verwendet werden, oder die Cursor-Funktion kann als Methode des Planobjekts aufgerufen werden.

    Das Cursor-Objekt stellt eine Methode `fetch` bereit, die einen Integer-Parameter akzeptiert und ein Ergebnisobjekt zurückgibt. Jedes Mal, wenn Sie `fetch` aufrufen, enthält das zurückgegebene Objekt den nächsten Block von Zeilen, nie mehr als der Parameterwert. Sobald alle Zeilen erschöpft sind, gibt `fetch` leere Ergebnisobjekte zurück. Cursor-Objekte stellen außerdem eine Iterator-Schnittstelle bereit (<https://docs.python.org/library/stdtypes.html#iterator-types>), die jeweils eine Zeile liefert, bis alle Zeilen erschöpft sind. Auf diese Weise abgerufene Daten werden nicht als Ergebnisobjekte zurückgegeben, sondern als Dictionaries, wobei jedes Dictionary einer einzelnen Ergebniszeile entspricht.

    Ein Beispiel für zwei Arten, Daten aus einer großen Tabelle zu verarbeiten:

    ```sql
    CREATE FUNCTION count_odd_iterator() RETURNS integer AS $$
    odd = 0
    for row in plpy.cursor("select num from largetable"):
        if row['num'] % 2:
             odd += 1
    return odd
    $$ LANGUAGE plpython3u;

    CREATE FUNCTION count_odd_fetch(batch_size integer) RETURNS
     integer AS $$
    odd = 0
    cursor = plpy.cursor("select num from largetable")
    while True:
        rows = cursor.fetch(batch_size)
        if not rows:
            break
        for row in rows:
            if row['num'] % 2:
                odd += 1
    return odd
    $$ LANGUAGE plpython3u;

    CREATE FUNCTION count_odd_prepared() RETURNS integer AS $$
    odd = 0
    plan = plpy.prepare("select num from largetable where num % $1 <> 0", ["integer"])
    rows = list(plpy.cursor(plan, [2])) # or: list(plan.cursor([2]))

    return len(rows)
    $$ LANGUAGE plpython3u;
    ```

    Cursor werden automatisch freigegeben. Wenn Sie aber alle von einem Cursor gehaltenen Ressourcen ausdrücklich freigeben möchten, verwenden Sie die Methode `close`. Sobald ein Cursor geschlossen ist, kann nicht mehr daraus gelesen werden.

> Verwechseln Sie mit `plpy.cursor` erzeugte Objekte nicht mit DB-API-Cursors, wie sie von der Python Database API Specification definiert werden: <https://www.python.org/dev/peps/pep-0249/>. Außer dem Namen haben sie nichts gemeinsam.

### 44.6.2. Fehler abfangen

Funktionen, die auf die Datenbank zugreifen, können auf Fehler stoßen, die dazu führen, dass sie abbrechen und eine Exception auslösen. Sowohl `plpy.execute` als auch `plpy.prepare` können eine Instanz einer Unterklasse von `plpy.SPIError` auslösen, die die Funktion standardmäßig beendet. Dieser Fehler kann wie jede andere Python-Exception mit dem Konstrukt `try`/`except` behandelt werden. Zum Beispiel:

```sql
CREATE FUNCTION try_adding_joe() RETURNS text AS $$
    try:
         plpy.execute("INSERT INTO users(username) VALUES ('joe')")
    except plpy.SPIError:
         return "something went wrong"
    else:
         return "Joe added"
$$ LANGUAGE plpython3u;
```

Die tatsächliche Klasse der ausgelösten Exception entspricht der spezifischen Bedingung, die den Fehler verursacht hat. Eine Liste möglicher Bedingungen finden Sie in Tabelle A.1. Das Modul `plpy.spiexceptions` definiert für jede PostgreSQL-Bedingung eine Exception-Klasse und leitet deren Namen aus dem Bedingungsnamen ab. Zum Beispiel wird `division_by_zero` zu `DivisionByZero`, `unique_violation` zu `UniqueViolation`, `fdw_error` zu `FdwError` und so weiter. Jede dieser Exception-Klassen erbt von `SPIError`. Diese Trennung erleichtert die Behandlung bestimmter Fehler, zum Beispiel:

```sql
CREATE FUNCTION insert_fraction(numerator int, denominator int)
 RETURNS text AS $$
from plpy import spiexceptions
try:
     plan = plpy.prepare("INSERT INTO fractions (frac) VALUES ($1 / $2)", ["int", "int"])
     plpy.execute(plan, [numerator, denominator])
except spiexceptions.DivisionByZero:
     return "denominator cannot equal zero"
except spiexceptions.UniqueViolation:
     return "already have that fraction"
except plpy.SPIError as e:
     return "other error, SQLSTATE %s" % e.sqlstate
else:
     return "fraction inserted"
$$ LANGUAGE plpython3u;
```

Beachten Sie, dass alle Exceptions aus dem Modul `plpy.spiexceptions` von `SPIError` erben; eine `except`-Klausel, die `SPIError` behandelt, fängt daher jeden Datenbankzugriffsfehler ab.

Als alternative Möglichkeit, verschiedene Fehlerbedingungen zu behandeln, können Sie die Exception `SPIError` abfangen und die spezifische Fehlerbedingung innerhalb des `except`-Blocks anhand des Attributs `sqlstate` des Exception-Objekts bestimmen. Dieses Attribut ist ein Zeichenkettenwert, der den „SQLSTATE“-Fehlercode enthält. Dieser Ansatz bietet ungefähr dieselbe Funktionalität.

## 44.7. Explizite Subtransaktionen

Das Wiederherstellen nach Fehlern, die durch Datenbankzugriff verursacht wurden, wie in [Abschnitt 44.6.2](#4462-fehler-abfangen) beschrieben, kann zu einer unerwünschten Situation führen, in der einige Operationen erfolgreich sind, bevor eine von ihnen fehlschlägt, und die Daten nach der Wiederherstellung von diesem Fehler in einem inkonsistenten Zustand zurückbleiben. PL/Python bietet für dieses Problem eine Lösung in Form expliziter Subtransaktionen.

### 44.7.1. Subtransaction-Context-Manager

Betrachten Sie eine Funktion, die eine Überweisung zwischen zwei Konten implementiert:

```sql
CREATE FUNCTION transfer_funds() RETURNS void AS $$
try:
     plpy.execute("UPDATE accounts SET balance = balance - 100 WHERE account_name = 'joe'")
     plpy.execute("UPDATE accounts SET balance = balance + 100 WHERE account_name = 'mary'")
except plpy.SPIError as e:
     result = "error transferring funds: %s" % e.args
else:
     result = "funds transferred correctly"
plan = plpy.prepare("INSERT INTO operations (result) VALUES ($1)", ["text"])
plpy.execute(plan, [result])
$$ LANGUAGE plpython3u;
```

Wenn die zweite `UPDATE`-Anweisung dazu führt, dass eine Exception ausgelöst wird, meldet diese Funktion den Fehler, aber das Ergebnis der ersten `UPDATE`-Anweisung wird dennoch committet. Mit anderen Worten: Das Geld wird von Joes Konto abgebucht, aber nicht auf Marys Konto übertragen.

Um solche Probleme zu vermeiden, können Sie Ihre `plpy.execute`-Aufrufe in eine explizite Subtransaktion einschließen. Das Modul `plpy` stellt ein Hilfsobjekt zur Verwaltung expliziter Subtransaktionen bereit, das mit der Funktion `plpy.subtransaction()` erzeugt wird. Objekte, die von dieser Funktion erzeugt werden, implementieren die Context-Manager-Schnittstelle (<https://docs.python.org/library/stdtypes.html#context-manager-types>). Mit expliziten Subtransaktionen können wir unsere Funktion so umschreiben:

```sql
CREATE FUNCTION transfer_funds2() RETURNS void AS $$
try:
     with plpy.subtransaction():
         plpy.execute("UPDATE accounts SET balance = balance - 100 WHERE account_name = 'joe'")
         plpy.execute("UPDATE accounts SET balance = balance + 100 WHERE account_name = 'mary'")
except plpy.SPIError as e:
     result = "error transferring funds: %s" % e.args
else:
     result = "funds transferred correctly"
plan = plpy.prepare("INSERT INTO operations (result) VALUES ($1)", ["text"])
plpy.execute(plan, [result])
$$ LANGUAGE plpython3u;
```

Beachten Sie, dass die Verwendung von `try`/`except` weiterhin erforderlich ist. Andernfalls würde sich die Exception bis an die Spitze des Python-Stacks ausbreiten und die gesamte Funktion mit einem PostgreSQL-Fehler abbrechen, sodass keine Zeile in die Tabelle `operations` eingefügt würde. Der Subtransaction-Context-Manager fängt keine Fehler ab; er stellt nur sicher, dass alle Datenbankoperationen, die innerhalb seines Geltungsbereichs ausgeführt werden, atomar committet oder zurückgerollt werden. Ein Rollback des Subtransaktionsblocks erfolgt bei jeder Art von Exception-Ausstieg, nicht nur bei Fehlern, die aus dem Datenbankzugriff stammen. Auch eine reguläre Python-Exception, die innerhalb eines expliziten Subtransaktionsblocks ausgelöst wird, würde dazu führen, dass die Subtransaktion zurückgerollt wird.

## 44.8. Transaktionsverwaltung

In einer Prozedur, die von der obersten Ebene aufgerufen wird, oder in einem anonymen Codeblock (`DO`-Befehl), der von der obersten Ebene aufgerufen wird, ist es möglich, Transaktionen zu steuern. Um die aktuelle Transaktion zu committen, rufen Sie `plpy.commit()` auf. Um die aktuelle Transaktion zurückzurollen, rufen Sie `plpy.rollback()` auf. (Beachten Sie, dass es nicht möglich ist, die SQL-Befehle `COMMIT` oder `ROLLBACK` über `plpy.execute` oder Ähnliches auszuführen. Dies muss mit diesen Funktionen geschehen.) Nachdem eine Transaktion beendet wurde, wird automatisch eine neue Transaktion gestartet, sodass es dafür keine separate Funktion gibt.

Hier ist ein Beispiel:

```sql
CREATE PROCEDURE transaction_test1()
LANGUAGE plpython3u
AS $$
for i in range(0, 10):
    plpy.execute("INSERT INTO test1 (a) VALUES (%d)" % i)
    if i % 2 == 0:
        plpy.commit()
    else:
        plpy.rollback()
$$;

CALL transaction_test1();
```

Transaktionen können nicht beendet werden, solange eine explizite Subtransaktion aktiv ist.

## 44.9. Hilfsfunktionen

Das Modul `plpy` stellt außerdem die folgenden Funktionen bereit:

- `plpy.debug(msg, **kwargs)`
- `plpy.log(msg, **kwargs)`
- `plpy.info(msg, **kwargs)`
- `plpy.notice(msg, **kwargs)`
- `plpy.warning(msg, **kwargs)`
- `plpy.error(msg, **kwargs)`
- `plpy.fatal(msg, **kwargs)`

`plpy.error` und `plpy.fatal` lösen tatsächlich eine Python-Exception aus, die sich, wenn sie nicht abgefangen wird, bis zur aufrufenden Abfrage ausbreitet und dazu führt, dass die aktuelle Transaktion oder Subtransaktion abgebrochen wird. `raise plpy.Error(msg)` und `raise plpy.Fatal(msg)` entsprechen dem Aufruf von `plpy.error(msg)` beziehungsweise `plpy.fatal(msg)`, aber die `raise`-Form erlaubt keine Übergabe von Schlüsselwortargumenten. Die anderen Funktionen erzeugen nur Meldungen unterschiedlicher Priorität. Ob Meldungen einer bestimmten Priorität an den Client gemeldet, in das Serverlog geschrieben oder beides werden, wird durch die Konfigurationsvariablen `log_min_messages` und `client_min_messages` gesteuert. Weitere Informationen finden Sie in [Kapitel 19](19_Serverkonfiguration.md).

Das Argument `msg` wird als Positionsargument angegeben. Aus Gründen der Rückwärtskompatibilität kann mehr als ein Positionsargument angegeben werden. In diesem Fall wird die Zeichenkettendarstellung des Tupels der Positionsargumente zur Meldung, die an den Client gemeldet wird.

Die folgenden Keyword-only-Argumente werden akzeptiert:

- `detail`
- `hint`
- `sqlstate`
- `schema_name`
- `table_name`
- `column_name`
- `datatype_name`
- `constraint_name`

Die Zeichenkettendarstellung der als Keyword-only-Argumente übergebenen Objekte wird verwendet, um die an den Client gemeldeten Meldungen anzureichern. Zum Beispiel:

```sql
CREATE FUNCTION raise_custom_exception() RETURNS void AS $$
plpy.error("custom exception message",
           detail="some info about exception",
           hint="hint for users")
$$ LANGUAGE plpython3u;
```

```text
=# SELECT raise_custom_exception();
ERROR: plpy.Error: custom exception message
DETAIL: some info about exception
HINT: hint for users
CONTEXT: Traceback (most recent call last):
  PL/Python function "raise_custom_exception", line 4, in <module>
    hint="hint for users")
PL/Python function "raise_custom_exception"
```

Eine weitere Gruppe von Hilfsfunktionen sind `plpy.quote_literal(string)`, `plpy.quote_nullable(string)` und `plpy.quote_ident(string)`. Sie entsprechen den eingebauten Quoting-Funktionen, die in [Abschnitt 9.4](09_Funktionen_und_Operatoren.md#94-zeichenkettenfunktionen-und-operatoren) beschrieben sind. Sie sind nützlich, wenn Ad-hoc-Abfragen konstruiert werden. Ein PL/Python-Äquivalent zu dynamischem SQL aus Beispiel 41.1 wäre:

```sql
plpy.execute("UPDATE tbl SET %s = %s WHERE key = %s" % (
    plpy.quote_ident(colname),
    plpy.quote_nullable(newvalue),
    plpy.quote_literal(keyvalue)))
```

## 44.10. Python 2 versus Python 3

PL/Python unterstützt nur Python 3. Frühere Versionen von PostgreSQL unterstützten Python 2 unter Verwendung der Sprachnamen `plpythonu` und `plpython2u`.

## 44.11. Umgebungsvariablen

Einige der Umgebungsvariablen, die vom Python-Interpreter akzeptiert werden, können auch verwendet werden, um das Verhalten von PL/Python zu beeinflussen. Sie müssen in der Umgebung des Hauptprozesses des PostgreSQL-Servers gesetzt werden, zum Beispiel in einem Startskript. Die verfügbaren Umgebungsvariablen hängen von der Python-Version ab; Details finden Sie in der Python-Dokumentation. Zum Zeitpunkt der Erstellung dieses Textes wirken sich die folgenden Umgebungsvariablen auf PL/Python aus, sofern eine passende Python-Version verwendet wird:

- `PYTHONHOME`
- `PYTHONPATH`
- `PYTHONY2K`
- `PYTHONOPTIMIZE`
- `PYTHONDEBUG`
- `PYTHONVERBOSE`
- `PYTHONCASEOK`
- `PYTHONDONTWRITEBYTECODE`
- `PYTHONIOENCODING`
- `PYTHONUSERBASE`
- `PYTHONHASHSEED`

(Es scheint ein Implementierungsdetail von Python außerhalb der Kontrolle von PL/Python zu sein, dass einige der in der Manpage `python` aufgeführten Umgebungsvariablen nur in einem Kommandozeileninterpreter wirksam sind und nicht in einem eingebetteten Python-Interpreter.)
