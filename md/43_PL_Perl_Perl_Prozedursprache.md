# 43. PL/Perl - Perl-Prozedursprache

PL/Perl ist eine ladbare prozedurale Sprache, mit der Sie PostgreSQL-Funktionen und -Prozeduren in der Programmiersprache Perl schreiben können.

Der Hauptvorteil von PL/Perl besteht darin, dass gespeicherte Funktionen und Prozeduren die zahlreichen Operatoren und Funktionen zur Zeichenkettenverarbeitung nutzen können, die Perl bereitstellt. Das Parsen komplexer Zeichenketten kann mit Perl einfacher sein als mit den Zeichenkettenfunktionen und Kontrollstrukturen von PL/pgSQL.

Um PL/Perl in einer bestimmten Datenbank zu installieren, verwenden Sie `CREATE EXTENSION plperl`.

> Wenn eine Sprache in `template1` installiert wird, haben alle danach erzeugten Datenbanken diese Sprache automatisch installiert.

> Benutzer von Quellpaketen müssen den Bau von PL/Perl während des Installationsprozesses ausdrücklich aktivieren. (Weitere Informationen finden Sie in [Kapitel 17](17_Installation_aus_dem_Quellcode.md).) Benutzer von Binärpaketen finden PL/Perl möglicherweise in einem separaten Unterpaket.

## 43.1. PL/Perl-Funktionen und Argumente

Um eine Funktion in der Sprache PL/Perl zu erstellen, verwenden Sie die Standard-Syntax von `CREATE FUNCTION`:

```sql
CREATE FUNCTION funcname (argument-types)
RETURNS return-type
-- function attributes can go here
AS $$
    # PL/Perl function body goes here
$$ LANGUAGE plperl;
```

Der Rumpf der Funktion ist gewöhnlicher Perl-Code. Tatsächlich verpackt der PL/Perl-Verbindungscode ihn in eine Perl-Subroutine. Eine PL/Perl-Funktion wird in einem skalaren Kontext aufgerufen und kann deshalb keine Liste zurückgeben. Nichtskalare Werte (Arrays, Datensätze und Mengen) können Sie zurückgeben, indem Sie eine Referenz zurückgeben, wie unten beschrieben.

In einer PL/Perl-Prozedur wird jeder Rückgabewert aus dem Perl-Code ignoriert.

PL/Perl unterstützt außerdem anonyme Codeblöcke, die mit der Anweisung `DO` aufgerufen werden:

```sql
DO $$
    # PL/Perl code
$$ LANGUAGE plperl;
```

Ein anonymer Codeblock erhält keine Argumente, und jeder Wert, den er eventuell zurückgibt, wird verworfen. Ansonsten verhält er sich genau wie eine Funktion.

> Die Verwendung benannter verschachtelter Subroutinen ist in Perl gefährlich, insbesondere wenn sie auf lexikalische Variablen im umgebenden Geltungsbereich verweisen. Da eine PL/Perl-Funktion in eine Subroutine verpackt wird, ist jede benannte Subroutine, die Sie darin anlegen, verschachtelt. Im Allgemeinen ist es deutlich sicherer, anonyme Subroutinen zu erzeugen, die Sie über eine Coderef aufrufen. Weitere Informationen finden Sie in den Einträgen zu `Variable "%s" will not stay shared` und `Variable "%s" is not available` in der Manpage `perldiag`, oder suchen Sie im Internet nach „perl nested named subroutine“.

Die Syntax des Befehls `CREATE FUNCTION` verlangt, dass der Funktionsrumpf als Zeichenkettenkonstante geschrieben wird. Für diese Zeichenkettenkonstante ist Dollar-Quoting normalerweise am bequemsten (siehe [Abschnitt 4.1.2.4](04_SQL_Syntax.md#4124-dollarquotedzeichenkettenkonstanten)). Wenn Sie stattdessen die Escape-Zeichenkettensyntax `E''` verwenden, müssen Sie alle einfachen Anführungszeichen (`'`) und Backslashes (`\`) im Funktionsrumpf verdoppeln (siehe [Abschnitt 4.1.2.2](04_SQL_Syntax.md#4122-zeichenkettenkonstanten-mit-cartigen-escapes)).

Argumente und Ergebnisse werden wie in jeder anderen Perl-Subroutine behandelt: Argumente werden in `@_` übergeben, und ein Ergebniswert wird mit `return` oder als letzter in der Funktion ausgewerteter Ausdruck zurückgegeben.

Zum Beispiel könnte eine Funktion, die den größeren von zwei ganzzahligen Werten zurückgibt, so definiert werden:

```sql
CREATE FUNCTION perl_max (integer, integer) RETURNS integer AS $$
    if ($_[0] > $_[1]) { return $_[0]; }
    return $_[1];
$$ LANGUAGE plperl;
```

> Argumente werden aus der Kodierung der Datenbank nach UTF-8 konvertiert, damit sie innerhalb von PL/Perl verwendet werden können, und beim Rückgabewert anschließend wieder von UTF-8 in die Datenbankkodierung zurückkonvertiert.

Wenn ein SQL-Nullwert an eine Funktion übergeben wird, erscheint der Argumentwert in Perl als „undefined“. Die obige Funktionsdefinition verhält sich bei Null-Eingaben nicht besonders günstig (tatsächlich behandelt sie sie so, als wären sie Nullen). Wir könnten der Funktionsdefinition `STRICT` hinzufügen, damit PostgreSQL etwas Vernünftigeres tut: Wenn ein Nullwert übergeben wird, wird die Funktion gar nicht aufgerufen, sondern gibt automatisch ein Nullergebnis zurück. Alternativ könnten wir im Funktionsrumpf auf undefinierte Eingaben prüfen. Angenommen zum Beispiel, wir möchten, dass `perl_max` bei einem Null- und einem Nicht-Null-Argument das Nicht-Null-Argument zurückgibt statt eines Nullwerts:

```sql
CREATE FUNCTION perl_max (integer, integer) RETURNS integer AS $$
    my ($x, $y) = @_;
    if (not defined $x) {
        return undef if not defined $y;
        return $y;
    }
    return $x if not defined $y;
    return $x if $x > $y;
    return $y;
$$ LANGUAGE plperl;
```

Wie oben gezeigt, geben Sie aus einer PL/Perl-Funktion einen undefinierten Wert zurück, um einen SQL-Nullwert zurückzugeben. Das ist unabhängig davon möglich, ob die Funktion strict ist oder nicht.

Alles in einem Funktionsargument, das keine Referenz ist, ist eine Zeichenkette in der standardmäßigen externen Textdarstellung von PostgreSQL für den jeweiligen Datentyp. Bei gewöhnlichen numerischen oder Texttypen tut Perl einfach das Richtige, und der Programmierer muss sich normalerweise nicht darum kümmern. In anderen Fällen muss das Argument jedoch in eine Form konvertiert werden, die in Perl besser verwendbar ist. Zum Beispiel kann die Funktion `decode_bytea` verwendet werden, um ein Argument des Typs `bytea` in unescaped Binärdaten zu konvertieren.

Entsprechend müssen Werte, die an PostgreSQL zurückgegeben werden, im Format der externen Textdarstellung vorliegen. Zum Beispiel kann die Funktion `encode_bytea` verwendet werden, um Binärdaten für einen Rückgabewert des Typs `bytea` zu escapen.

Ein besonders wichtiger Fall sind boolesche Werte. Wie gerade beschrieben, ist das Standardverhalten für `bool`-Werte, dass sie an Perl als Text übergeben werden, also entweder als `'t'` oder `'f'`. Das ist problematisch, weil Perl `'f'` nicht als falsch behandelt. Die Situation lässt sich durch einen „Transform“ verbessern (siehe `CREATE TRANSFORM`). Geeignete Transforms werden von der Erweiterung `bool_plperl` bereitgestellt. Um sie zu verwenden, installieren Sie die Erweiterung:

```sql
CREATE EXTENSION bool_plperl;                    -- or bool_plperlu for PL/PerlU
```

Verwenden Sie dann das Funktionsattribut `TRANSFORM` für eine PL/Perl-Funktion, die `bool` annimmt oder zurückgibt, zum Beispiel:

```sql
CREATE FUNCTION perl_and(bool, bool) RETURNS bool
TRANSFORM FOR TYPE bool
AS $$
  my ($a, $b) = @_;
  return $a && $b;
$$ LANGUAGE plperl;
```

Wenn dieser Transform angewendet wird, sieht Perl `bool`-Argumente als `1` oder leer, also korrekt als wahr oder falsch. Wenn das Funktionsergebnis den Typ `bool` hat, ist es wahr oder falsch, je nachdem, ob Perl den zurückgegebenen Wert als wahr auswerten würde. Ähnliche Transformationen werden auch für boolesche Abfrageargumente und Ergebnisse von SPI-Abfragen durchgeführt, die innerhalb der Funktion ausgeführt werden ([Abschnitt 43.3.1](#4331-datenbankzugriff-aus-plperl)).

Perl kann PostgreSQL-Arrays als Referenzen auf Perl-Arrays zurückgeben. Hier ist ein Beispiel:

```sql
CREATE OR REPLACE function returns_array()
RETURNS text[][] AS $$
    return [['a"b','c,d'],['e\\f','g']];
$$ LANGUAGE plperl;

select returns_array();
```

Perl übergibt PostgreSQL-Arrays als blessed `PostgreSQL::InServer::ARRAY`-Objekt. Dieses Objekt kann als Array-Referenz oder als Zeichenkette behandelt werden; dadurch kann Perl-Code, der für PostgreSQL-Versionen vor 9.1 geschrieben wurde, aus Gründen der Rückwärtskompatibilität weiterlaufen. Zum Beispiel:

```sql
CREATE OR REPLACE FUNCTION concat_array_elements(text[]) RETURNS
 TEXT AS $$
    my $arg = shift;
    my $result = "";
    return undef if (!defined $arg);

      # as an array reference
      for (@$arg) {
          $result .= $_;
      }

      # also works as a string
      $result .= $arg;

    return $result;
$$ LANGUAGE plperl;

SELECT concat_array_elements(ARRAY['PL','/','Perl']);
```

> Mehrdimensionale Arrays werden als Referenzen auf niedriger dimensionierte Arrays von Referenzen dargestellt, auf eine Weise, die jedem Perl-Programmierer vertraut ist.

Argumente von zusammengesetzten Typen werden der Funktion als Referenzen auf Hashes übergeben. Die Schlüssel des Hashes sind die Attributnamen des zusammengesetzten Typs. Hier ist ein Beispiel:

```sql
CREATE TABLE employee (
    name text,
    basesalary integer,
    bonus integer
);

CREATE FUNCTION empcomp(employee) RETURNS integer AS $$
    my ($emp) = @_;
    return $emp->{basesalary} + $emp->{bonus};
$$ LANGUAGE plperl;

SELECT name, empcomp(employee.*) FROM employee;
```

Eine PL/Perl-Funktion kann mit demselben Ansatz ein Ergebnis eines zusammengesetzten Typs zurückgeben: Sie gibt eine Referenz auf einen Hash zurück, der die erforderlichen Attribute enthält. Zum Beispiel:

```sql
CREATE TYPE testrowperl AS (f1 integer, f2 text, f3 text);

CREATE OR REPLACE FUNCTION perl_row() RETURNS testrowperl AS $$
    return {f2 => 'hello', f1 => 1, f3 => 'world'};
$$ LANGUAGE plperl;

SELECT * FROM perl_row();
```

Alle Spalten im deklarierten Ergebnisdatentyp, die im Hash nicht vorhanden sind, werden als Nullwerte zurückgegeben.

Entsprechend können Ausgabeparameter von Prozeduren als Hash-Referenz zurückgegeben werden:

```sql
CREATE PROCEDURE perl_triple(INOUT a integer, INOUT b integer) AS $$
    my ($a, $b) = @_;
    return {a => $a * 3, b => $b * 3};
$$ LANGUAGE plperl;

CALL perl_triple(5, 10);
```

PL/Perl-Funktionen können außerdem Mengen skalarer oder zusammengesetzter Typen zurückgeben. Normalerweise werden Sie Zeilen einzeln zurückgeben wollen, sowohl um die Startzeit zu verkürzen als auch um zu vermeiden, dass die gesamte Ergebnismenge im Speicher gepuffert wird. Das können Sie mit `return_next` tun, wie unten gezeigt. Beachten Sie, dass Sie nach dem letzten `return_next` entweder `return` oder (besser) `return undef` schreiben müssen.

```sql
CREATE OR REPLACE FUNCTION perl_set_int(int)
RETURNS SETOF INTEGER AS $$
    foreach (0..$_[0]) {
        return_next($_);
    }
    return undef;
$$ LANGUAGE plperl;

SELECT * FROM perl_set_int(5);

CREATE OR REPLACE FUNCTION perl_set()
RETURNS SETOF testrowperl AS $$
    return_next({ f1 => 1, f2 => 'Hello', f3 => 'World' });
    return_next({ f1 => 2, f2 => 'Hello', f3 => 'PostgreSQL' });
    return_next({ f1 => 3, f2 => 'Hello', f3 => 'PL/Perl' });
    return undef;
$$ LANGUAGE plperl;
```

Für kleine Ergebnismengen können Sie eine Referenz auf ein Array zurückgeben, das für einfache Typen, Array-Typen beziehungsweise zusammengesetzte Typen entweder Skalare, Referenzen auf Arrays oder Referenzen auf Hashes enthält. Hier sind einige einfache Beispiele, die die gesamte Ergebnismenge als Array-Referenz zurückgeben:

```sql
CREATE OR REPLACE FUNCTION perl_set_int(int) RETURNS SETOF INTEGER
 AS $$
    return [0..$_[0]];
$$ LANGUAGE plperl;

SELECT * FROM perl_set_int(5);

CREATE OR REPLACE FUNCTION perl_set() RETURNS SETOF testrowperl AS
 $$
    return [
        { f1 => 1, f2 => 'Hello', f3 => 'World' },
        { f1 => 2, f2 => 'Hello', f3 => 'PostgreSQL' },
        { f1 => 3, f2 => 'Hello', f3 => 'PL/Perl' }
    ];
$$ LANGUAGE plperl;

SELECT * FROM perl_set();
```

Wenn Sie das Pragma `strict` mit Ihrem Code verwenden möchten, haben Sie einige Möglichkeiten. Für eine temporäre globale Verwendung können Sie `plperl.use_strict` mit `SET` auf `true` setzen. Das wirkt sich auf nachfolgende Kompilierungen von PL/Perl-Funktionen aus, nicht aber auf Funktionen, die in der aktuellen Sitzung bereits kompiliert wurden. Für eine dauerhafte globale Verwendung können Sie `plperl.use_strict` in der Datei `postgresql.conf` auf `true` setzen.

Für eine dauerhafte Verwendung in bestimmten Funktionen können Sie einfach Folgendes an den Anfang des Funktionsrumpfs stellen:

```sql
use strict;
```

Das Pragma `feature` kann ebenfalls verwendet werden, wenn Ihre Perl-Version 5.10.0 oder höher ist.

## 43.2. Datenwerte in PL/Perl

Die Argumentwerte, die dem Code einer PL/Perl-Funktion bereitgestellt werden, sind einfach die Eingabeargumente, die in Textform konvertiert wurden (so, als wären sie von einer `SELECT`-Anweisung angezeigt worden). Umgekehrt akzeptieren die Befehle `return` und `return_next` jede Zeichenkette, die ein zulässiges Eingabeformat für den deklarierten Rückgabetyp der Funktion ist.

Wenn dieses Verhalten in einem bestimmten Fall unpraktisch ist, kann es durch einen Transform verbessert werden, wie oben bereits für `bool`-Werte gezeigt. Mehrere Beispiele für Transform-Module sind in der PostgreSQL-Distribution enthalten.

## 43.3. Eingebaute Funktionen

### 43.3.1. Datenbankzugriff aus PL/Perl

Der Zugriff auf die Datenbank selbst kann aus Ihrer Perl-Funktion heraus über die folgenden Funktionen erfolgen:

- `spi_exec_query(query [, limit])`
  - `spi_exec_query` führt einen SQL-Befehl aus und gibt die gesamte Zeilenmenge als Referenz auf ein Array von Hash-Referenzen zurück. Wenn `limit` angegeben und größer als null ist, ruft `spi_exec_query` höchstens `limit` Zeilen ab, ähnlich als enthielte die Abfrage eine `LIMIT`-Klausel. Wird `limit` weggelassen oder als null angegeben, gibt es keine Zeilenbegrenzung.

    Sie sollten diesen Befehl nur verwenden, wenn Sie wissen, dass die Ergebnismenge relativ klein ist. Hier ist ein Beispiel für eine Abfrage (`SELECT`-Befehl) mit der optionalen maximalen Anzahl von Zeilen:

    ```sql
    $rv = spi_exec_query('SELECT * FROM my_table', 5);
    ```

    Dies gibt bis zu 5 Zeilen aus der Tabelle `my_table` zurück. Wenn `my_table` eine Spalte `my_column` hat, können Sie diesen Wert aus Zeile `$i` des Ergebnisses so abrufen:

    ```sql
    $foo = $rv->{rows}[$i]->{my_column};
    ```

    Auf die Gesamtzahl der von einer `SELECT`-Abfrage zurückgegebenen Zeilen können Sie so zugreifen:

    ```sql
    $nrows = $rv->{processed}
    ```

    Hier ist ein Beispiel mit einem anderen Befehlstyp:

    ```sql
    $query = "INSERT INTO my_table VALUES (1, 'test')";
    $rv = spi_exec_query($query);
    ```

    Danach können Sie auf den Befehlsstatus (zum Beispiel `SPI_OK_INSERT`) so zugreifen:

    ```sql
    $res = $rv->{status};
    ```

    Um die Anzahl der betroffenen Zeilen zu ermitteln, verwenden Sie:

    ```sql
    $nrows = $rv->{processed};
    ```

    Hier ist ein vollständiges Beispiel:

    ```sql
    CREATE TABLE test (
        i int,
        v varchar
    );

    INSERT INTO test (i, v) VALUES (1, 'first line');
    INSERT INTO test (i, v) VALUES (2, 'second line');
    INSERT INTO test (i, v) VALUES (3, 'third line');
    INSERT INTO test (i, v) VALUES (4, 'immortal');

    CREATE OR REPLACE FUNCTION test_munge() RETURNS SETOF test AS $$
        my $rv = spi_exec_query('select i, v from test;');
        my $status = $rv->{status};
        my $nrows = $rv->{processed};
        foreach my $rn (0 .. $nrows - 1) {
            my $row = $rv->{rows}[$rn];
            $row->{i} += 200 if defined($row->{i});
            $row->{v} =~ tr/A-Za-z/a-zA-Z/ if (defined($row->{v}));
            return_next($row);
        }
        return undef;
    $$ LANGUAGE plperl;

    SELECT * FROM test_munge();
    ```

- `spi_query(command)`
- `spi_fetchrow(cursor)`
- `spi_cursor_close(cursor)`
  - `spi_query` und `spi_fetchrow` arbeiten als Paar für Zeilenmengen zusammen, die groß sein könnten, oder für Fälle, in denen Sie Zeilen zurückgeben möchten, sobald sie eintreffen. `spi_fetchrow` funktioniert nur mit `spi_query`. Das folgende Beispiel zeigt, wie Sie beide zusammen verwenden:

    ```sql
    CREATE TYPE foo_type AS (the_num INTEGER, the_text TEXT);

    CREATE OR REPLACE FUNCTION lotsa_md5 (INTEGER) RETURNS SETOF
     foo_type AS $$
        use Digest::MD5 qw(md5_hex);
        my $file = '/usr/share/dict/words';
        my $t = localtime;
        elog(NOTICE, "opening file $file at $t" );
        open my $fh, '<', $file # ooh, it's a file access!
            or elog(ERROR, "cannot open $file for reading: $!");
        my @words = <$fh>;
        close $fh;
        $t = localtime;
        elog(NOTICE, "closed file $file at $t");
        chomp(@words);
        my $row;
        my $sth = spi_query("SELECT * FROM generate_series(1,$_[0]) AS b(a)");
        while (defined ($row = spi_fetchrow($sth))) {
            return_next({
                 the_num => $row->{a},
                 the_text => md5_hex($words[rand @words])
            });
        }
        return;
    $$ LANGUAGE plperlu;

    SELECT * from lotsa_md5(500);
    ```

    Normalerweise sollte `spi_fetchrow` wiederholt werden, bis es `undef` zurückgibt und damit anzeigt, dass keine weiteren Zeilen zu lesen sind. Der von `spi_query` zurückgegebene Cursor wird automatisch freigegeben, wenn `spi_fetchrow` `undef` zurückgibt. Wenn Sie nicht alle Zeilen lesen möchten, rufen Sie stattdessen `spi_cursor_close` auf, um den Cursor freizugeben. Andernfalls entstehen Speicherlecks.

- `spi_prepare(command, argument types)`
- `spi_query_prepared(plan, arguments)`
- `spi_exec_prepared(plan [, attributes], arguments)`
- `spi_freeplan(plan)`
  - `spi_prepare`, `spi_query_prepared`, `spi_exec_prepared` und `spi_freeplan` implementieren dieselbe Funktionalität, aber für vorbereitete Abfragen. `spi_prepare` akzeptiert eine Abfragezeichenkette mit nummerierten Argumentplatzhaltern (`$1`, `$2` usw.) und eine Zeichenkettenliste von Argumenttypen:

    ```sql
    $plan = spi_prepare('SELECT * FROM test WHERE id > $1 AND name =
     $2',
                                                         'INTEGER',
     'TEXT');
    ```

    Sobald ein Abfrageplan durch einen Aufruf von `spi_prepare` vorbereitet ist, kann der Plan anstelle der Abfragezeichenkette verwendet werden, entweder in `spi_exec_prepared`, wobei das Ergebnis dasselbe ist wie bei `spi_exec_query`, oder in `spi_query_prepared`, das genau wie `spi_query` einen Cursor zurückgibt, der später an `spi_fetchrow` übergeben werden kann. Der optionale zweite Parameter von `spi_exec_prepared` ist eine Hash-Referenz von Attributen; das einzige derzeit unterstützte Attribut ist `limit`, das die maximale Anzahl der von der Abfrage zurückgegebenen Zeilen festlegt. Wird `limit` weggelassen oder als null angegeben, gibt es keine Zeilenbegrenzung.

    Der Vorteil vorbereiteter Abfragen besteht darin, dass ein vorbereiteter Plan für mehr als eine Abfrageausführung verwendet werden kann. Wenn der Plan nicht mehr benötigt wird, kann er mit `spi_freeplan` freigegeben werden:

    ```sql
    CREATE OR REPLACE FUNCTION init() RETURNS VOID AS $$
            $_SHARED{my_plan} = spi_prepare('SELECT (now() +
     $1)::date AS now',
                                            'INTERVAL');
    $$ LANGUAGE plperl;

    CREATE OR REPLACE FUNCTION add_time( INTERVAL ) RETURNS TEXT AS
     $$
            return spi_exec_prepared(
                    $_SHARED{my_plan},
                    $_[0]
            )->{rows}->[0]->{now};
    $$ LANGUAGE plperl;

    CREATE OR REPLACE FUNCTION done() RETURNS VOID AS $$
            spi_freeplan( $_SHARED{my_plan});
            undef $_SHARED{my_plan};
    $$ LANGUAGE plperl;

    SELECT init();
    SELECT add_time('1 day'), add_time('2 days'), add_time('3 days');
    SELECT done();
    ```

    ```text
      add_time | add_time | add_time
    ------------+------------+------------
     2005-12-10 | 2005-12-11 | 2005-12-12
    ```

    Beachten Sie, dass der Parameterindex in `spi_prepare` über `$1`, `$2`, `$3` usw. definiert wird. Vermeiden Sie deshalb, Abfragezeichenketten in doppelten Anführungszeichen zu deklarieren, da das leicht zu schwer auffindbaren Fehlern führen kann.

    Ein weiteres Beispiel zeigt die Verwendung eines optionalen Parameters in `spi_exec_prepared`:

    ```sql
    CREATE TABLE hosts AS SELECT id, ('192.168.1.'||id)::inet AS
     address
                          FROM generate_series(1,3) AS id;

    CREATE OR REPLACE FUNCTION init_hosts_query() RETURNS VOID AS $$
            $_SHARED{plan} = spi_prepare('SELECT * FROM hosts
                                          WHERE address << $1',
     'inet');
    $$ LANGUAGE plperl;

    CREATE OR REPLACE FUNCTION query_hosts(inet) RETURNS SETOF hosts
     AS $$
            return spi_exec_prepared(
                    $_SHARED{plan},
                    {limit => 2},
                    $_[0]
            )->{rows};
    $$ LANGUAGE plperl;

    CREATE OR REPLACE FUNCTION release_hosts_query() RETURNS VOID AS
     $$
            spi_freeplan($_SHARED{plan});
            undef $_SHARED{plan};
    $$ LANGUAGE plperl;

    SELECT init_hosts_query();
    SELECT query_hosts('192.168.1.0/30');
    SELECT release_hosts_query();
    ```

    ```text
        query_hosts
    -----------------
     (1,192.168.1.1)
     (2,192.168.1.2)
    (2 rows)
    ```

- `spi_commit()`
- `spi_rollback()`
  - Schreibt die aktuelle Transaktion fest oder rollt sie zurück. Dies kann nur in einer Prozedur oder einem anonymen Codeblock (`DO`-Befehl) aufgerufen werden, der von der obersten Ebene aufgerufen wird. (Beachten Sie, dass es nicht möglich ist, die SQL-Befehle `COMMIT` oder `ROLLBACK` über `spi_exec_query` oder Ähnliches auszuführen. Das muss mit diesen Funktionen geschehen.) Nachdem eine Transaktion beendet wurde, wird automatisch eine neue Transaktion gestartet, sodass es dafür keine separate Funktion gibt.

    Hier ist ein Beispiel:

    ```sql
    CREATE PROCEDURE transaction_test1()
          LANGUAGE plperl
          AS $$
          foreach my $i (0..9) {
              spi_exec_query("INSERT INTO test1 (a) VALUES ($i)");
              if ($i % 2 == 0) {
                  spi_commit();
              } else {
                  spi_rollback();
              }
          }
          $$;

          CALL transaction_test1();
    ```

### 43.3.2. Hilfsfunktionen in PL/Perl

- `elog(level, msg)`
  - Gibt eine Log- oder Fehlermeldung aus. Mögliche Stufen sind `DEBUG`, `LOG`, `INFO`, `NOTICE`, `WARNING` und `ERROR`. `ERROR` löst eine Fehlerbedingung aus; wenn diese nicht vom umgebenden Perl-Code abgefangen wird, breitet sich der Fehler bis zur aufrufenden Abfrage aus, wodurch die aktuelle Transaktion oder Subtransaktion abgebrochen wird. Dies ist effektiv dasselbe wie der Perl-Befehl `die`. Die anderen Stufen erzeugen nur Meldungen unterschiedlicher Priorität. Ob Meldungen einer bestimmten Priorität an den Client gemeldet, in das Serverlog geschrieben oder beides werden, wird durch die Konfigurationsvariablen `log_min_messages` und `client_min_messages` gesteuert. Weitere Informationen finden Sie in [Kapitel 19](19_Serverkonfiguration.md).

- `quote_literal(string)`
  - Gibt die angegebene Zeichenkette passend gequotet zurück, sodass sie als Zeichenkettenliteral in einer SQL-Anweisungszeichenkette verwendet werden kann. Eingebettete einfache Anführungszeichen und Backslashes werden korrekt verdoppelt. Beachten Sie, dass `quote_literal` bei `undef`-Eingabe `undef` zurückgibt; wenn das Argument `undef` sein könnte, ist `quote_nullable` oft besser geeignet.

- `quote_nullable(string)`
  - Gibt die angegebene Zeichenkette passend gequotet zurück, sodass sie als Zeichenkettenliteral in einer SQL-Anweisungszeichenkette verwendet werden kann; oder, wenn das Argument `undef` ist, die ungequotete Zeichenkette `"NULL"`. Eingebettete einfache Anführungszeichen und Backslashes werden korrekt verdoppelt.

- `quote_ident(string)`
  - Gibt die angegebene Zeichenkette passend gequotet zurück, sodass sie als Bezeichner in einer SQL-Anweisungszeichenkette verwendet werden kann. Anführungszeichen werden nur bei Bedarf hinzugefügt (also wenn die Zeichenkette Nicht-Bezeichnerzeichen enthält oder durch Kleinschreibung gefaltet würde). Eingebettete Anführungszeichen werden korrekt verdoppelt.

- `decode_bytea(string)`
  - Gibt die unescaped Binärdaten zurück, die durch den Inhalt der angegebenen Zeichenkette dargestellt werden; diese sollte `bytea`-kodiert sein.

- `encode_bytea(string)`
  - Gibt die `bytea`-kodierte Form des Binärdateninhalts der angegebenen Zeichenkette zurück.

- `encode_array_literal(array)`
- `encode_array_literal(array, delimiter)`
  - Gibt den Inhalt des referenzierten Arrays als Zeichenkette im Array-Literalformat zurück (siehe [Abschnitt 8.15.2](08_Datentypen.md#8152-eingabe-von-arraywerten)). Gibt den Argumentwert unverändert zurück, wenn es sich nicht um eine Referenz auf ein Array handelt. Das zwischen Elementen des Array-Literals verwendete Trennzeichen ist standardmäßig `", "`, wenn kein Trennzeichen angegeben wird oder wenn es `undef` ist.

- `encode_typed_literal(value, typename)`
  - Konvertiert eine Perl-Variable in den als zweites Argument übergebenen Datentyp und gibt eine Zeichenkettendarstellung dieses Werts zurück. Verschachtelte Arrays und Werte zusammengesetzter Typen werden korrekt behandelt.

- `encode_array_constructor(array)`
  - Gibt den Inhalt des referenzierten Arrays als Zeichenkette im Array-Konstruktorformat zurück (siehe [Abschnitt 4.2.12](04_SQL_Syntax.md#4212-arraykonstruktoren)). Einzelne Werte werden mit `quote_nullable` gequotet. Gibt den Argumentwert mit `quote_nullable` gequotet zurück, wenn es sich nicht um eine Referenz auf ein Array handelt.

- `looks_like_number(string)`
  - Gibt einen wahren Wert zurück, wenn der Inhalt der angegebenen Zeichenkette nach Perl wie eine Zahl aussieht, andernfalls einen falschen Wert. Gibt `undef` zurück, wenn das Argument `undef` ist. Führender und abschließender Leerraum wird ignoriert. `Inf` und `Infinity` gelten als Zahlen.

- `is_array_ref(argument)`
  - Gibt einen wahren Wert zurück, wenn das angegebene Argument als Array-Referenz behandelt werden kann, also wenn `ref` des Arguments `ARRAY` oder `PostgreSQL::InServer::ARRAY` ist. Andernfalls wird ein falscher Wert zurückgegeben.

## 43.4. Globale Werte in PL/Perl

Sie können den globalen Hash `%_SHARED` verwenden, um Daten, einschließlich Code-Referenzen, zwischen Funktionsaufrufen für die Lebensdauer der aktuellen Sitzung zu speichern.

Hier ist ein einfaches Beispiel für gemeinsam genutzte Daten:

```sql
CREATE OR REPLACE FUNCTION set_var(name text, val text) RETURNS
 text AS $$
    if ($_SHARED{$_[0]} = $_[1]) {
        return 'ok';
    } else {
        return "cannot set shared variable $_[0] to $_[1]";
    }
$$ LANGUAGE plperl;

CREATE OR REPLACE FUNCTION get_var(name text) RETURNS text AS $$
    return $_SHARED{$_[0]};
$$ LANGUAGE plperl;

SELECT set_var('sample', 'Hello, PL/Perl!                           How''s tricks?');
SELECT get_var('sample');
```

Hier ist ein etwas komplizierteres Beispiel, das eine Code-Referenz verwendet:

```sql
CREATE OR REPLACE FUNCTION myfuncs() RETURNS void AS $$
    $_SHARED{myquote} = sub {
        my $arg = shift;
        $arg =~ s/(['\\])/\\$1/g;
        return "'$arg'";
    };
$$ LANGUAGE plperl;

SELECT myfuncs(); /* initializes the function */

/* Set up a function that uses the quote function */

CREATE OR REPLACE FUNCTION use_quote(TEXT) RETURNS text AS $$
    my $text_to_quote = shift;
    my $qfunc = $_SHARED{myquote};
    return &$qfunc($text_to_quote);
$$ LANGUAGE plperl;
```

(Sie hätten das Obige auf Kosten der Lesbarkeit durch den Einzeiler `return $_SHARED{myquote}->($_[0]);` ersetzen können.)

Aus Sicherheitsgründen führt PL/Perl Funktionen, die von einer bestimmten SQL-Rolle aufgerufen werden, in einem separaten Perl-Interpreter für diese Rolle aus. Dadurch wird verhindert, dass ein Benutzer versehentlich oder böswillig das Verhalten der PL/Perl-Funktionen eines anderen Benutzers beeinflusst. Jeder dieser Interpreter hat seinen eigenen Wert der Variablen `%_SHARED` und seinen eigenen weiteren globalen Zustand. Daher teilen sich zwei PL/Perl-Funktionen genau dann denselben Wert von `%_SHARED`, wenn sie von derselben SQL-Rolle ausgeführt werden. In einer Anwendung, in der eine einzelne Sitzung Code unter mehreren SQL-Rollen ausführt (über `SECURITY DEFINER`-Funktionen, die Verwendung von `SET ROLE` usw.), müssen Sie möglicherweise ausdrücklich dafür sorgen, dass PL/Perl-Funktionen Daten über `%_SHARED` gemeinsam nutzen können. Stellen Sie dazu sicher, dass Funktionen, die miteinander kommunizieren sollen, demselben Benutzer gehören, und markieren Sie sie als `SECURITY DEFINER`. Natürlich müssen Sie darauf achten, dass solche Funktionen nicht für unbeabsichtigte Zwecke missbraucht werden können.

## 43.5. Vertrauenswürdiges und nicht vertrauenswürdiges PL/Perl

Normalerweise wird PL/Perl als „vertrauenswürdige“ Programmiersprache namens `plperl` installiert. In dieser Konfiguration sind bestimmte Perl-Operationen deaktiviert, um die Sicherheit zu wahren. Im Allgemeinen sind die eingeschränkten Operationen diejenigen, die mit der Umgebung interagieren. Dazu gehören Datei-Handle-Operationen, `require` und `use` (für externe Module). Es gibt keine Möglichkeit, auf Interna des Datenbankserverprozesses zuzugreifen oder mit den Rechten des Serverprozesses Zugriff auf Betriebssystemebene zu erlangen, wie es eine C-Funktion könnte. Deshalb kann jedem nicht privilegierten Datenbankbenutzer erlaubt werden, diese Sprache zu verwenden.

> Warnung: Vertrauenswürdiges PL/Perl stützt sich auf das Perl-Modul `Opcode`, um Sicherheit zu gewährleisten. Die Perl-Dokumentation weist darauf hin, dass das Modul für den Anwendungsfall des vertrauenswürdigen PL/Perl nicht wirksam ist: <https://perldoc.perl.org/Opcode#WARNING>. Wenn Ihre Sicherheitsanforderungen mit der Unsicherheit in dieser Warnung nicht vereinbar sind, sollten Sie erwägen, `REVOKE USAGE ON LANGUAGE plperl FROM PUBLIC` auszuführen.

Hier ist ein Beispiel für eine Funktion, die nicht funktionieren wird, weil Dateisystemoperationen aus Sicherheitsgründen nicht erlaubt sind:

```sql
CREATE FUNCTION badfunc() RETURNS integer AS $$
    my $tmpfile = "/tmp/badfile";
    open my $fh, '>', $tmpfile
        or elog(ERROR, qq{could not open the file "$tmpfile": $!});
    print $fh "Testing writing to a file\n";
    close $fh or elog(ERROR, qq{could not close the file "$tmpfile": $!});
    return 1;
$$ LANGUAGE plperl;
```

Das Erzeugen dieser Funktion wird fehlschlagen, weil ihre Verwendung einer verbotenen Operation vom Validator erkannt wird.

Manchmal ist es wünschenswert, Perl-Funktionen zu schreiben, die nicht eingeschränkt sind. Zum Beispiel könnte man eine Perl-Funktion wollen, die E-Mails sendet. Für solche Fälle kann PL/Perl auch als „nicht vertrauenswürdige“ Sprache installiert werden (üblicherweise PL/PerlU genannt). In diesem Fall steht die vollständige Perl-Sprache zur Verfügung. Beim Installieren der Sprache wählt der Sprachname `plperlu` die nicht vertrauenswürdige PL/Perl-Variante.

Der Autor einer PL/PerlU-Funktion muss darauf achten, dass die Funktion nicht für unerwünschte Zwecke verwendet werden kann, da sie alles tun kann, was ein als Datenbankadministrator angemeldeter Benutzer tun könnte. Beachten Sie, dass das Datenbanksystem nur Datenbank-Superusern erlaubt, Funktionen in nicht vertrauenswürdigen Sprachen zu erstellen.

Wenn die obige Funktion von einem Superuser mit der Sprache `plperlu` erstellt würde, wäre ihre Ausführung erfolgreich.

Auf dieselbe Weise können anonyme Codeblöcke, die in Perl geschrieben sind, eingeschränkte Operationen verwenden, wenn als Sprache `plperlu` statt `plperl` angegeben wird; der Aufrufer muss jedoch ein Superuser sein.

> Während PL/Perl-Funktionen für jede SQL-Rolle in einem separaten Perl-Interpreter laufen, laufen alle PL/PerlU-Funktionen, die in einer bestimmten Sitzung ausgeführt werden, in einem einzigen Perl-Interpreter (der keiner der für PL/Perl-Funktionen verwendeten Interpreter ist). Dadurch können PL/PerlU-Funktionen Daten frei gemeinsam nutzen, aber zwischen PL/Perl- und PL/PerlU-Funktionen kann keine Kommunikation stattfinden.

> Perl kann mehrere Interpreter innerhalb eines Prozesses nur unterstützen, wenn es mit den passenden Flags gebaut wurde, nämlich entweder `usemultiplicity` oder `useithreads`. (`usemultiplicity` wird bevorzugt, sofern Sie nicht tatsächlich Threads verwenden müssen. Weitere Details finden Sie in der Manpage `perlembed`.) Wenn PL/Perl mit einer Perl-Kopie verwendet wird, die nicht auf diese Weise gebaut wurde, ist nur ein Perl-Interpreter pro Sitzung möglich, und eine Sitzung kann daher entweder PL/PerlU-Funktionen oder PL/Perl-Funktionen ausführen, die alle von derselben SQL-Rolle aufgerufen werden.

## 43.6. PL/Perl-Trigger

PL/Perl kann zum Schreiben von Trigger-Funktionen verwendet werden. In einer Trigger-Funktion enthält die Hash-Referenz `$_TD` Informationen über das aktuelle Trigger-Ereignis. `$_TD` ist eine globale Variable, die für jeden Aufruf des Triggers einen eigenen lokalen Wert erhält. Die Felder der Hash-Referenz `$_TD` sind:

- `$_TD->{new}{foo}`
  - Neuer Wert der Spalte `foo`.

- `$_TD->{old}{foo}`
  - Alter Wert der Spalte `foo`.

- `$_TD->{name}`
  - Name des aufgerufenen Triggers.

- `$_TD->{event}`
  - Trigger-Ereignis: `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE` oder `UNKNOWN`.

- `$_TD->{when}`
  - Zeitpunkt, zu dem der Trigger aufgerufen wurde: `BEFORE`, `AFTER`, `INSTEAD OF` oder `UNKNOWN`.

- `$_TD->{level}`
  - Trigger-Ebene: `ROW`, `STATEMENT` oder `UNKNOWN`.

- `$_TD->{relid}`
  - OID der Tabelle, auf der der Trigger ausgelöst wurde.

- `$_TD->{table_name}`
  - Name der Tabelle, auf der der Trigger ausgelöst wurde.

- `$_TD->{relname}`
  - Name der Tabelle, auf der der Trigger ausgelöst wurde. Dies ist veraltet und könnte in einer zukünftigen Version entfernt werden. Verwenden Sie stattdessen `$_TD->{table_name}`.

- `$_TD->{table_schema}`
  - Name des Schemas, in dem sich die Tabelle befindet, auf der der Trigger ausgelöst wurde.

- `$_TD->{argc}`
  - Anzahl der Argumente der Trigger-Funktion.

- `@{$_TD->{args}}`
  - Argumente der Trigger-Funktion. Existiert nicht, wenn `$_TD->{argc}` den Wert 0 hat.

Trigger auf Zeilenebene können einen der folgenden Werte zurückgeben:

- `return;`
  - Führt die Operation aus.

- `"SKIP"`
  - Führt die Operation nicht aus.

- `"MODIFY"`
  - Zeigt an, dass die `NEW`-Zeile durch die Trigger-Funktion geändert wurde.

Hier ist ein Beispiel für eine Trigger-Funktion, das einige der obigen Punkte veranschaulicht:

```sql
CREATE TABLE test (
    i int,
    v varchar
);

CREATE OR REPLACE FUNCTION valid_id() RETURNS trigger AS $$
    if (($_TD->{new}{i} >= 100) || ($_TD->{new}{i} <= 0)) {
         return "SKIP";    # skip INSERT/UPDATE command
    } elsif ($_TD->{new}{v} ne "immortal") {
         $_TD->{new}{v} .= "(modified by trigger)";
         return "MODIFY"; # modify row and execute INSERT/UPDATE command
    } else {
         return;           # execute INSERT/UPDATE command
    }
$$ LANGUAGE plperl;

CREATE TRIGGER test_valid_id_trig
    BEFORE INSERT OR UPDATE ON test
    FOR EACH ROW EXECUTE FUNCTION valid_id();
```

## 43.7. PL/Perl-Event-Trigger

PL/Perl kann zum Schreiben von Event-Trigger-Funktionen verwendet werden. In einer Event-Trigger-Funktion enthält die Hash-Referenz `$_TD` Informationen über das aktuelle Trigger-Ereignis. `$_TD` ist eine globale Variable, die für jeden Aufruf des Triggers einen eigenen lokalen Wert erhält. Die Felder der Hash-Referenz `$_TD` sind:

- `$_TD->{event}`
  - Der Name des Ereignisses, für das der Trigger ausgelöst wird.

- `$_TD->{tag}`
  - Der Command Tag, für den der Trigger ausgelöst wird.

Der Rückgabewert der Trigger-Funktion wird ignoriert.

Hier ist ein Beispiel für eine Event-Trigger-Funktion, das einige der obigen Punkte veranschaulicht:

```sql
CREATE OR REPLACE FUNCTION perlsnitch() RETURNS event_trigger AS $$
  elog(NOTICE, "perlsnitch: " . $_TD->{event} . " " . $_TD->{tag} . " ");
$$ LANGUAGE plperl;

CREATE EVENT TRIGGER perl_a_snitch
    ON ddl_command_start
    EXECUTE FUNCTION perlsnitch();
```

## 43.8. PL/Perl unter der Haube

### 43.8.1. Konfiguration

Dieser Abschnitt listet Konfigurationsparameter auf, die PL/Perl betreffen.

- `plperl.on_init` (`string`)
  - Gibt Perl-Code an, der ausgeführt wird, wenn ein Perl-Interpreter erstmals initialisiert wird, bevor er für die Verwendung durch `plperl` oder `plperlu` spezialisiert wird. Die SPI-Funktionen sind nicht verfügbar, wenn dieser Code ausgeführt wird. Wenn der Code mit einem Fehler fehlschlägt, bricht er die Initialisierung des Interpreters ab und breitet sich bis zur aufrufenden Abfrage aus, wodurch die aktuelle Transaktion oder Subtransaktion abgebrochen wird.

    Der Perl-Code ist auf eine einzelne Zeichenkette beschränkt. Längerer Code kann in ein Modul gelegt und von der `on_init`-Zeichenkette geladen werden. Beispiele:

    ```text
    plperl.on_init = 'require "plperlinit.pl"'
    plperl.on_init = 'use lib "/my/app"; use MyApp::PgInit;'
    ```

    Alle Module, die von `plperl.on_init` direkt oder indirekt geladen werden, stehen für die Verwendung durch `plperl` zur Verfügung. Dies kann ein Sicherheitsrisiko darstellen. Um zu sehen, welche Module geladen wurden, können Sie Folgendes verwenden:

    ```sql
    DO 'elog(WARNING, join ", ", sort keys %INC)' LANGUAGE plperl;
    ```

    Die Initialisierung erfolgt im Postmaster, wenn die Bibliothek `plperl` in `shared_preload_libraries` enthalten ist; in diesem Fall sollte dem Risiko, den Postmaster zu destabilisieren, besondere Beachtung geschenkt werden. Der Hauptgrund für die Nutzung dieser Funktion ist, dass Perl-Module, die von `plperl.on_init` geladen werden, nur beim Start des Postmasters geladen werden müssen und dann ohne Ladeaufwand in einzelnen Datenbanksitzungen sofort verfügbar sind. Beachten Sie jedoch, dass der Aufwand nur für den ersten Perl-Interpreter vermieden wird, den eine Datenbanksitzung verwendet: entweder PL/PerlU oder PL/Perl für die erste SQL-Rolle, die eine PL/Perl-Funktion aufruft. Jeder zusätzliche Perl-Interpreter, der in einer Datenbanksitzung erzeugt wird, muss `plperl.on_init` erneut ausführen. Außerdem gibt es unter Windows keinerlei Einsparung durch Vorladen, da der im Postmaster-Prozess erzeugte Perl-Interpreter nicht an Kindprozesse weitergegeben wird.

    Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

- `plperl.on_plperl_init` (`string`)
- `plperl.on_plperlu_init` (`string`)
  - Diese Parameter geben Perl-Code an, der ausgeführt wird, wenn ein Perl-Interpreter für `plperl` beziehungsweise `plperlu` spezialisiert wird. Das geschieht, wenn eine PL/Perl- oder PL/PerlU-Funktion erstmals in einer Datenbanksitzung ausgeführt wird oder wenn ein zusätzlicher Interpreter erzeugt werden muss, weil die andere Sprache aufgerufen wird oder eine PL/Perl-Funktion von einer neuen SQL-Rolle aufgerufen wird. Dies folgt auf jede Initialisierung, die durch `plperl.on_init` erfolgt ist. Die SPI-Funktionen sind nicht verfügbar, wenn dieser Code ausgeführt wird. Der Perl-Code in `plperl.on_plperl_init` wird nach dem „Lockdown“ des Interpreters ausgeführt und kann daher nur vertrauenswürdige Operationen ausführen.

    Wenn der Code mit einem Fehler fehlschlägt, bricht er die Initialisierung ab und breitet sich bis zur aufrufenden Abfrage aus, wodurch die aktuelle Transaktion oder Subtransaktion abgebrochen wird. Aktionen, die innerhalb von Perl bereits ausgeführt wurden, werden nicht rückgängig gemacht; dieser Interpreter wird jedoch nicht erneut verwendet. Wenn die Sprache wieder verwendet wird, wird die Initialisierung in einem frischen Perl-Interpreter erneut versucht.

    Nur Superuser können diese Einstellungen ändern. Obwohl diese Einstellungen innerhalb einer Sitzung geändert werden können, wirken sich solche Änderungen nicht auf Perl-Interpreter aus, die bereits zur Ausführung von Funktionen verwendet wurden.

- `plperl.use_strict` (`boolean`)
  - Wenn dies auf `true` gesetzt ist, wird bei nachfolgenden Kompilierungen von PL/Perl-Funktionen das Pragma `strict` aktiviert. Dieser Parameter wirkt sich nicht auf Funktionen aus, die in der aktuellen Sitzung bereits kompiliert wurden.

### 43.8.2. Einschränkungen und fehlende Funktionen

Die folgenden Funktionen fehlen derzeit in PL/Perl, wären aber willkommene Beiträge.

- PL/Perl-Funktionen können einander nicht direkt aufrufen.

- SPI ist noch nicht vollständig implementiert.

- Wenn Sie mit `spi_exec_query` sehr große Datenmengen abrufen, sollten Sie beachten, dass diese vollständig in den Speicher geladen werden. Sie können dies vermeiden, indem Sie `spi_query`/`spi_fetchrow` verwenden, wie weiter oben gezeigt.

  Ein ähnliches Problem tritt auf, wenn eine mengenrückgebende Funktion über `return` eine große Menge von Zeilen an PostgreSQL zurückgibt. Auch dieses Problem können Sie vermeiden, indem Sie stattdessen, wie zuvor gezeigt, für jede zurückgegebene Zeile `return_next` verwenden.

- Wenn eine Sitzung normal endet, also nicht aufgrund eines fatalen Fehlers, werden alle definierten `END`-Blöcke ausgeführt. Derzeit werden keine weiteren Aktionen ausgeführt. Insbesondere werden Datei-Handles nicht automatisch geleert und Objekte nicht automatisch zerstört.
