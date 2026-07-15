# 45. Server-Programmierschnittstelle

Das Server Programming Interface (SPI) gibt Autoren benutzerdefinierter C-Funktionen die Möglichkeit, SQL-Befehle innerhalb ihrer Funktionen oder Prozeduren auszuführen. SPI ist eine Menge von Schnittstellenfunktionen, die den Zugriff auf Parser, Planner und Executor vereinfachen. SPI übernimmt außerdem einen Teil der Speicherverwaltung.

> Die verfügbaren prozeduralen Sprachen stellen verschiedene Möglichkeiten bereit, SQL-Befehle aus Funktionen heraus auszuführen. Die meisten dieser Einrichtungen basieren auf SPI, daher kann diese Dokumentation auch für Benutzer dieser Sprachen nützlich sein.

Beachten Sie: Wenn ein über SPI aufgerufener Befehl fehlschlägt, wird die Steuerung nicht an Ihre C-Funktion zurückgegeben. Stattdessen wird die Transaktion oder Subtransaktion, in der Ihre C-Funktion ausgeführt wird, zurückgerollt. (Das mag überraschend wirken, da für die SPI-Funktionen meist Fehler-Rückgabekonventionen dokumentiert sind. Diese Konventionen gelten jedoch nur für Fehler, die innerhalb der SPI-Funktionen selbst erkannt werden.) Es ist möglich, nach einem Fehler die Kontrolle wiederzuerlangen, indem Sie eine eigene Subtransaktion um SPI-Aufrufe legen, die fehlschlagen könnten.

SPI-Funktionen geben bei Erfolg ein nichtnegatives Ergebnis zurück (entweder über einen zurückgegebenen Integer-Wert oder in der globalen Variablen `SPI_result`, wie unten beschrieben). Bei einem Fehler wird ein negatives Ergebnis oder `NULL` zurückgegeben.

Quellcodedateien, die SPI verwenden, müssen die Headerdatei `executor/spi.h` einbinden.

## 45.1. Schnittstellenfunktionen

### `SPI_connect`

`SPI_connect`, `SPI_connect_ext` - eine C-Funktion mit dem SPI-Manager verbinden

**Synopsis**

```c
int SPI_connect(void)

int SPI_connect_ext(int options)
```

**Beschreibung**

`SPI_connect` öffnet eine Verbindung von einem C-Funktionsaufruf zum SPI-Manager. Sie müssen diese Funktion aufrufen, wenn Sie Befehle über SPI ausführen möchten. Einige SPI-Hilfsfunktionen können aus nicht verbundenen C-Funktionen heraus aufgerufen werden.

`SPI_connect_ext` tut dasselbe, hat aber ein Argument, das die Übergabe von Options-Flags erlaubt. Derzeit sind die folgenden Optionswerte verfügbar:

- `SPI_OPT_NONATOMIC`
  - Setzt die SPI-Verbindung auf nonatomic, was bedeutet, dass Transaktionssteuerungsaufrufe (`SPI_commit`, `SPI_rollback`) erlaubt sind. Andernfalls führt der Aufruf dieser Funktionen zu einem sofortigen Fehler.

`SPI_connect()` ist äquivalent zu `SPI_connect_ext(0)`.

**Rückgabewert**

- `SPI_OK_CONNECT`
  - Bei Erfolg.

Dass diese Funktionen `int` und nicht `void` zurückgeben, ist historisch bedingt. Alle Fehlerfälle werden über `ereport` oder `elog` gemeldet. (In Versionen vor PostgreSQL v10 wurden einige, aber nicht alle Fehler mit einem Ergebniswert von `SPI_ERROR_CONNECT` gemeldet.)

### `SPI_finish`

`SPI_finish` - eine C-Funktion vom SPI-Manager trennen

**Synopsis**

```c
int SPI_finish(void)
```

**Beschreibung**

`SPI_finish` schließt eine bestehende Verbindung zum SPI-Manager. Sie müssen diese Funktion aufrufen, nachdem Sie die während des aktuellen Aufrufs Ihrer C-Funktion benötigten SPI-Operationen abgeschlossen haben. Sie müssen sich jedoch nicht darum kümmern, wenn Sie die Transaktion über `elog(ERROR)` abbrechen. In diesem Fall räumt SPI automatisch auf.

**Rückgabewert**

- `SPI_OK_FINISH`
  - Wenn die Verbindung ordnungsgemäß getrennt wurde.

- `SPI_ERROR_UNCONNECTED`
  - Wenn die Funktion aus einer nicht verbundenen C-Funktion heraus aufgerufen wurde.

### `SPI_execute`

`SPI_execute` - einen Befehl ausführen

**Synopsis**

```c
int SPI_execute(const char * command, bool read_only, long count)
```

**Beschreibung**

`SPI_execute` führt den angegebenen SQL-Befehl für `count` Zeilen aus. Wenn `read_only` wahr ist, muss der Befehl schreibgeschützt sein, und der Ausführungsaufwand wird etwas reduziert.

Diese Funktion kann nur aus einer verbundenen C-Funktion heraus aufgerufen werden.

Wenn `count` null ist, wird der Befehl für alle Zeilen ausgeführt, auf die er anwendbar ist. Wenn `count` größer als null ist, werden nicht mehr als `count` Zeilen abgerufen; die Ausführung stoppt, wenn `count` erreicht ist, ähnlich wie beim Hinzufügen einer `LIMIT`-Klausel zur Abfrage. Zum Beispiel:

```c
SPI_execute("SELECT * FROM foo", true, 5);
```

ruft höchstens 5 Zeilen aus der Tabelle ab. Beachten Sie, dass eine solche Begrenzung nur wirksam ist, wenn der Befehl tatsächlich Zeilen zurückgibt. Zum Beispiel:

```c
SPI_execute("INSERT INTO foo SELECT * FROM bar", false, 5);
```

fügt alle Zeilen aus `bar` ein und ignoriert den Parameter `count`. Dagegen würden mit

```c
SPI_execute("INSERT INTO foo SELECT * FROM bar RETURNING *", false, 5);
```

höchstens 5 Zeilen eingefügt, da die Ausführung stoppt, nachdem die fünfte `RETURNING`-Ergebniszeile abgerufen wurde.

Sie können mehrere Befehle in einer Zeichenkette übergeben; `SPI_execute` gibt das Ergebnis des zuletzt ausgeführten Befehls zurück. Die Begrenzung durch `count` gilt für jeden Befehl separat (auch wenn tatsächlich nur das letzte Ergebnis zurückgegeben wird). Die Begrenzung wird nicht auf versteckte Befehle angewendet, die von Regeln erzeugt werden.

Wenn `read_only` falsch ist, erhöht `SPI_execute` den Befehlszähler und berechnet vor der Ausführung jedes Befehls in der Zeichenkette einen neuen Snapshot. Der Snapshot ändert sich nicht tatsächlich, wenn die aktuelle Transaktionsisolationsstufe `SERIALIZABLE` oder `REPEATABLE READ` ist; im Modus `READ COMMITTED` erlaubt die Snapshot-Aktualisierung jedoch jedem Befehl, die Ergebnisse neu committeter Transaktionen aus anderen Sitzungen zu sehen. Das ist für konsistentes Verhalten wesentlich, wenn die Befehle die Datenbank ändern.

Wenn `read_only` wahr ist, aktualisiert `SPI_execute` weder den Snapshot noch den Befehlszähler, und es erlaubt nur einfache `SELECT`-Befehle in der Befehlszeichenkette. Die Befehle werden mit dem Snapshot ausgeführt, der zuvor für die umgebende Abfrage etabliert wurde. Dieser Ausführungsmodus ist etwas schneller als der Lese-/Schreibmodus, weil der Aufwand pro Befehl entfällt. Er erlaubt außerdem den Bau tatsächlich stabiler Funktionen: Da aufeinanderfolgende Ausführungen alle denselben Snapshot verwenden, ändern sich die Ergebnisse nicht.

Im Allgemeinen ist es unklug, schreibgeschützte und schreibende Befehle innerhalb einer einzelnen Funktion mit SPI zu mischen. Das könnte zu sehr verwirrendem Verhalten führen, da die schreibgeschützten Abfragen die Ergebnisse von Datenbankaktualisierungen, die von den schreibenden Abfragen durchgeführt wurden, nicht sehen würden.

Die tatsächliche Anzahl der Zeilen, für die der (letzte) Befehl ausgeführt wurde, wird in der globalen Variablen `SPI_processed` zurückgegeben. Wenn der Rückgabewert der Funktion `SPI_OK_SELECT`, `SPI_OK_INSERT_RETURNING`, `SPI_OK_DELETE_RETURNING`, `SPI_OK_UPDATE_RETURNING` oder `SPI_OK_MERGE_RETURNING` ist, können Sie den globalen Zeiger `SPITupleTable *SPI_tuptable` verwenden, um auf die Ergebniszeilen zuzugreifen. Einige Utility-Befehle (etwa `EXPLAIN`) geben ebenfalls Zeilenmengen zurück, und `SPI_tuptable` enthält auch in diesen Fällen das Ergebnis. Einige Utility-Befehle (`COPY`, `CREATE TABLE AS`) geben keine Zeilenmenge zurück, sodass `SPI_tuptable` `NULL` ist, aber sie geben dennoch die Anzahl der verarbeiteten Zeilen in `SPI_processed` zurück.

Die Struktur `SPITupleTable` ist folgendermaßen definiert:

```c
typedef struct SPITupleTable
{
    /* Public members */
    TupleDesc   tupdesc;                       /* tuple descriptor */
    HeapTuple *vals;                           /* array of tuples */
    uint64      numvals;                       /* number of valid tuples */

    /* Private members, not intended for external callers */
    uint64      alloced;                       /* allocated length of vals array */
    MemoryContext tuptabcxt;                   /* memory context of result table */
    slist_node next;                           /* link for internal bookkeeping */
    SubTransactionId subid;                    /* subxact in which tuptable was created */
} SPITupleTable;
```

Die Felder `tupdesc`, `vals` und `numvals` können von SPI-Aufrufern verwendet werden; die übrigen Felder sind intern. `vals` ist ein Array von Zeigern auf Zeilen. Die Anzahl der Zeilen wird durch `numvals` angegeben (aus eher historischen Gründen wird diese Anzahl auch in `SPI_processed` zurückgegeben). `tupdesc` ist ein Zeilendeskriptor, den Sie an SPI-Funktionen übergeben können, die mit Zeilen umgehen.

`SPI_finish` gibt alle `SPITupleTable`s frei, die während der aktuellen C-Funktion angelegt wurden. Sie können eine bestimmte Ergebnistabelle früher freigeben, wenn Sie damit fertig sind, indem Sie `SPI_freetuptable` aufrufen.

**Argumente**

- `const char * command`
  - Zeichenkette, die den auszuführenden Befehl enthält.

- `bool read_only`
  - Wahr für schreibgeschützte Ausführung.

- `long count`
  - Maximale Anzahl zurückzugebender Zeilen oder 0 für keine Begrenzung.

**Rückgabewert**

Wenn die Ausführung des Befehls erfolgreich war, wird einer der folgenden (nichtnegativen) Werte zurückgegeben:

- `SPI_OK_SELECT`
  - Wenn ein `SELECT` (aber kein `SELECT INTO`) ausgeführt wurde.

- `SPI_OK_SELINTO`
  - Wenn ein `SELECT INTO` ausgeführt wurde.

- `SPI_OK_INSERT`
  - Wenn ein `INSERT` ausgeführt wurde.

- `SPI_OK_DELETE`
  - Wenn ein `DELETE` ausgeführt wurde.

- `SPI_OK_UPDATE`
  - Wenn ein `UPDATE` ausgeführt wurde.

- `SPI_OK_MERGE`
  - Wenn ein `MERGE` ausgeführt wurde.

- `SPI_OK_INSERT_RETURNING`
  - Wenn ein `INSERT RETURNING` ausgeführt wurde.

- `SPI_OK_DELETE_RETURNING`
  - Wenn ein `DELETE RETURNING` ausgeführt wurde.

- `SPI_OK_UPDATE_RETURNING`
  - Wenn ein `UPDATE RETURNING` ausgeführt wurde.

- `SPI_OK_MERGE_RETURNING`
  - Wenn ein `MERGE RETURNING` ausgeführt wurde.

- `SPI_OK_UTILITY`
  - Wenn ein Utility-Befehl (zum Beispiel `CREATE TABLE`) ausgeführt wurde.

- `SPI_OK_REWRITTEN`
  - Wenn der Befehl durch eine Regel in eine andere Befehlsart umgeschrieben wurde (zum Beispiel wenn aus `UPDATE` ein `INSERT` wurde).

Bei einem Fehler wird einer der folgenden negativen Werte zurückgegeben:

- `SPI_ERROR_ARGUMENT`
  - Wenn `command` `NULL` ist oder `count` kleiner als 0 ist.

- `SPI_ERROR_COPY`
  - Wenn `COPY TO stdout` oder `COPY FROM stdin` versucht wurde.

- `SPI_ERROR_TRANSACTION`
  - Wenn ein Transaktionsmanipulationsbefehl versucht wurde (`BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `PREPARE TRANSACTION`, `COMMIT PREPARED`, `ROLLBACK PREPARED` oder eine Variante davon).

- `SPI_ERROR_OPUNKNOWN`
  - Wenn der Befehlstyp unbekannt ist (sollte nicht passieren).

- `SPI_ERROR_UNCONNECTED`
  - Wenn die Funktion aus einer nicht verbundenen C-Funktion heraus aufgerufen wurde.

**Hinweise**

Alle SPI-Abfrageausführungsfunktionen setzen sowohl `SPI_processed` als auch `SPI_tuptable` (nur den Zeiger, nicht den Inhalt der Struktur). Speichern Sie diese beiden globalen Variablen in lokalen Variablen der C-Funktion, wenn Sie über spätere Aufrufe hinweg auf die Ergebnistabelle von `SPI_execute` oder einer anderen Abfrageausführungsfunktion zugreifen müssen.

### `SPI_exec`

`SPI_exec` - einen Lese-/Schreibbefehl ausführen

**Synopsis**

```c
int SPI_exec(const char * command, long count)
```

**Beschreibung**

`SPI_exec` ist dasselbe wie `SPI_execute`, wobei dessen Parameter `read_only` immer als falsch angenommen wird.

**Argumente**

- `const char * command`
  - Zeichenkette, die den auszuführenden Befehl enthält.

- `long count`
  - Maximale Anzahl zurückzugebender Zeilen oder 0 für keine Begrenzung.

**Rückgabewert**

Siehe `SPI_execute`.

### `SPI_execute_extended`

`SPI_execute_extended` - einen Befehl mit out-of-line-Parametern ausführen

**Synopsis**

```c
int SPI_execute_extended(const char *command,
                         const SPIExecuteOptions * options)
```

**Beschreibung**

`SPI_execute_extended` führt einen Befehl aus, der Verweise auf extern bereitgestellte Parameter enthalten kann. Der Befehlstext verweist auf einen Parameter als `$n`, und das Objekt `options->params` stellt (falls angegeben) Werte und Typinformationen für jedes solche Symbol bereit. In der Struktur `options` können außerdem verschiedene Ausführungsoptionen angegeben werden.

Das Objekt `options->params` sollte normalerweise jeden Parameter mit dem Flag `PARAM_FLAG_CONST` markieren, da für die Abfrage immer ein One-Shot-Plan verwendet wird.

Wenn `options->dest` nicht `NULL` ist, werden Ergebnistupel an dieses Objekt übergeben, während sie vom Executor erzeugt werden, statt in `SPI_tuptable` angesammelt zu werden. Die Verwendung eines vom Aufrufer bereitgestellten `DestReceiver`-Objekts ist besonders hilfreich für Abfragen, die viele Tupel erzeugen könnten, da die Daten direkt verarbeitet werden können, statt im Speicher gesammelt zu werden.

**Argumente**

- `const char * command`
  - Befehlszeichenkette.

- `const SPIExecuteOptions * options`
  - Struktur, die optionale Argumente enthält.

Aufrufer sollten immer die gesamte Struktur `options` mit Nullen initialisieren und dann die Felder füllen, die sie setzen möchten. Dies stellt die Vorwärtskompatibilität des Codes sicher, da alle Felder, die der Struktur künftig hinzugefügt werden, bei einem Wert von null so definiert werden, dass sie rückwärtskompatibel funktionieren. Die derzeit verfügbaren Optionsfelder sind:

- `ParamListInfo params`
  - Datenstruktur, die Typen und Werte der Abfrageparameter enthält; `NULL`, wenn keine vorhanden sind.

- `bool read_only`
  - Wahr für schreibgeschützte Ausführung.

- `bool allow_nonatomic`
  - Wahr erlaubt die nichtatomare Ausführung von `CALL`- und `DO`-Anweisungen (dieses Feld wird jedoch ignoriert, sofern nicht das Flag `SPI_OPT_NONATOMIC` an `SPI_connect_ext` übergeben wurde).

- `bool must_return_tuples`
  - Wenn wahr, wird ein Fehler ausgelöst, falls die Abfrage nicht von einer Art ist, die Tupel zurückgibt (dies verbietet nicht den Fall, dass sie zufällig null Tupel zurückgibt).

- `uint64 tcount`
  - Maximale Anzahl zurückzugebender Zeilen oder 0 für keine Begrenzung.

- `DestReceiver * dest`
  - `DestReceiver`-Objekt, das alle von der Abfrage ausgegebenen Tupel empfängt; wenn `NULL`, werden Ergebnistupel wie bei `SPI_execute` in einer `SPI_tuptable`-Struktur gesammelt.

- `ResourceOwner owner`
  - Dieses Feld ist aus Konsistenzgründen mit `SPI_execute_plan_extended` vorhanden, wird aber ignoriert, da der von `SPI_execute_extended` verwendete Plan nie gespeichert wird.

**Rückgabewert**

Der Rückgabewert ist derselbe wie bei `SPI_execute`.

Wenn `options->dest` `NULL` ist, werden `SPI_processed` und `SPI_tuptable` wie bei `SPI_execute` gesetzt. Wenn `options->dest` nicht `NULL` ist, wird `SPI_processed` auf null gesetzt und `SPI_tuptable` auf `NULL`. Wenn eine Tupelanzahl benötigt wird, muss das `DestReceiver`-Objekt des Aufrufers sie berechnen.

### `SPI_execute_with_args`

`SPI_execute_with_args` - einen Befehl mit out-of-line-Parametern ausführen

**Synopsis**

```c
int SPI_execute_with_args(const char *command,
                          int nargs, Oid *argtypes,
                          Datum *values, const char *nulls,
                          bool read_only, long count)
```

**Beschreibung**

`SPI_execute_with_args` führt einen Befehl aus, der Verweise auf extern bereitgestellte Parameter enthalten kann. Der Befehlstext verweist auf einen Parameter als `$n`, und der Aufruf gibt Datentypen und Werte für jedes solche Symbol an. `read_only` und `count` haben dieselbe Bedeutung wie bei `SPI_execute`.

Der Hauptvorteil dieser Routine gegenüber `SPI_execute` besteht darin, dass Datenwerte ohne mühsames Quoting/Escaping in den Befehl eingefügt werden können, und damit mit deutlich geringerem Risiko von SQL-Injection-Angriffen.

Ähnliche Ergebnisse können mit `SPI_prepare` gefolgt von `SPI_execute_plan` erzielt werden; bei Verwendung dieser Funktion wird der Abfrageplan jedoch immer an die konkret bereitgestellten Parameterwerte angepasst. Für einmalige Abfrageausführung sollte diese Funktion bevorzugt werden. Wenn derselbe Befehl mit vielen verschiedenen Parametern ausgeführt werden soll, kann jede der beiden Methoden schneller sein, abhängig von den Kosten des erneuten Planens gegenüber dem Nutzen benutzerdefinierter Pläne.

**Argumente**

- `const char * command`
  - Befehlszeichenkette.

- `int nargs`
  - Anzahl der Eingabeparameter (`$1`, `$2` usw.).

- `Oid * argtypes`
  - Array der Länge `nargs`, das die OIDs der Datentypen der Parameter enthält.

- `Datum * values`
  - Array der Länge `nargs`, das die tatsächlichen Parameterwerte enthält.

- `const char * nulls`
  - Array der Länge `nargs`, das beschreibt, welche Parameter null sind.

    Wenn `nulls` `NULL` ist, nimmt `SPI_execute_with_args` an, dass keine Parameter null sind. Andernfalls sollte jeder Eintrag des Arrays `nulls` ein Leerzeichen (`' '`) sein, wenn der entsprechende Parameterwert nicht-null ist, oder `'n'`, wenn der entsprechende Parameterwert null ist. (Im letzteren Fall spielt der tatsächliche Wert im entsprechenden Eintrag von `values` keine Rolle.) Beachten Sie, dass `nulls` keine Textzeichenkette ist, sondern nur ein Array: Es benötigt keinen Terminator `'\0'`.

- `bool read_only`
  - Wahr für schreibgeschützte Ausführung.

- `long count`
  - Maximale Anzahl zurückzugebender Zeilen oder 0 für keine Begrenzung.

**Rückgabewert**

Der Rückgabewert ist derselbe wie bei `SPI_execute`.

`SPI_processed` und `SPI_tuptable` werden bei Erfolg wie bei `SPI_execute` gesetzt.

### `SPI_prepare`

`SPI_prepare` - eine Anweisung vorbereiten, ohne sie bereits auszuführen

**Synopsis**

```c
SPIPlanPtr SPI_prepare(const char * command, int nargs, Oid * argtypes)
```

**Beschreibung**

`SPI_prepare` erzeugt eine vorbereitete Anweisung für den angegebenen Befehl und gibt sie zurück, führt den Befehl aber nicht aus. Die vorbereitete Anweisung kann später mit `SPI_execute_plan` wiederholt ausgeführt werden.

Wenn derselbe oder ein ähnlicher Befehl wiederholt ausgeführt werden soll, ist es im Allgemeinen vorteilhaft, die Parse-Analyse nur einmal durchzuführen, und es kann außerdem vorteilhaft sein, einen Ausführungsplan für den Befehl wiederzuverwenden. `SPI_prepare` konvertiert eine Befehlszeichenkette in eine vorbereitete Anweisung, die die Ergebnisse der Parse-Analyse kapselt. Die vorbereitete Anweisung bietet außerdem einen Ort zum Cachen eines Ausführungsplans, falls sich herausstellt, dass das Erzeugen eines benutzerdefinierten Plans für jede Ausführung nicht hilfreich ist.

Ein vorbereiteter Befehl kann verallgemeinert werden, indem Parameter (`$1`, `$2` usw.) anstelle dessen geschrieben werden, was in einem normalen Befehl Konstanten wären. Die tatsächlichen Werte der Parameter werden dann angegeben, wenn `SPI_execute_plan` aufgerufen wird. Dadurch kann der vorbereitete Befehl in einem größeren Bereich von Situationen verwendet werden, als es ohne Parameter möglich wäre.

Die von `SPI_prepare` zurückgegebene Anweisung kann nur im aktuellen Aufruf der C-Funktion verwendet werden, da `SPI_finish` den für eine solche Anweisung angelegten Speicher freigibt. Die Anweisung kann jedoch mit den Funktionen `SPI_keepplan` oder `SPI_saveplan` länger gespeichert werden.

**Argumente**

- `const char * command`
  - Befehlszeichenkette.

- `int nargs`
  - Anzahl der Eingabeparameter (`$1`, `$2` usw.).

- `Oid * argtypes`
  - Zeiger auf ein Array, das die OIDs der Datentypen der Parameter enthält.

**Rückgabewert**

`SPI_prepare` gibt einen nicht-null Zeiger auf einen `SPIPlan` zurück, eine opake Struktur, die eine vorbereitete Anweisung darstellt. Bei einem Fehler wird `NULL` zurückgegeben, und `SPI_result` wird auf einen der gleichen Fehlercodes gesetzt, die von `SPI_execute` verwendet werden; eine Ausnahme ist `SPI_ERROR_ARGUMENT`, wenn `command` `NULL` ist, wenn `nargs` kleiner als 0 ist oder wenn `nargs` größer als 0 und `argtypes` `NULL` ist.

**Hinweise**

Wenn keine Parameter definiert sind, wird bei der ersten Verwendung von `SPI_execute_plan` ein generischer Plan erzeugt und auch für alle folgenden Ausführungen verwendet. Wenn Parameter vorhanden sind, erzeugen die ersten Verwendungen von `SPI_execute_plan` benutzerdefinierte Pläne, die für die übergebenen Parameterwerte spezifisch sind. Nach genügend Verwendungen derselben vorbereiteten Anweisung baut `SPI_execute_plan` einen generischen Plan; wenn dieser nicht wesentlich teurer ist als die benutzerdefinierten Pläne, beginnt es, den generischen Plan zu verwenden, statt jedes Mal neu zu planen. Wenn dieses Standardverhalten ungeeignet ist, können Sie es ändern, indem Sie das Flag `CURSOR_OPT_GENERIC_PLAN` oder `CURSOR_OPT_CUSTOM_PLAN` an `SPI_prepare_cursor` übergeben, um jeweils die Verwendung generischer oder benutzerdefinierter Pläne zu erzwingen.

Obwohl der Hauptzweck einer vorbereiteten Anweisung darin besteht, wiederholte Parse-Analyse und Planung der Anweisung zu vermeiden, erzwingt PostgreSQL vor der Verwendung der Anweisung eine erneute Analyse und Planung, wenn Datenbankobjekte, die in der Anweisung verwendet werden, seit der vorherigen Verwendung der vorbereiteten Anweisung definitorische Änderungen (DDL) erfahren haben. Außerdem wird die Anweisung mit dem neuen `search_path` erneut geparst, wenn sich der Wert von `search_path` zwischen zwei Verwendungen ändert. (Dieses letztere Verhalten ist seit PostgreSQL 9.3 neu.) Weitere Informationen zum Verhalten vorbereiteter Anweisungen finden Sie unter `PREPARE`.

Diese Funktion sollte nur aus einer verbundenen C-Funktion heraus aufgerufen werden.

`SPIPlanPtr` ist in `spi.h` als Zeiger auf einen opaken Strukturtyp deklariert. Es ist unklug, direkt auf seinen Inhalt zuzugreifen, da Ihr Code dadurch viel wahrscheinlicher in zukünftigen PostgreSQL-Revisionen kaputtgeht.

Der Name `SPIPlanPtr` ist etwas historisch, da die Datenstruktur nicht mehr unbedingt einen Ausführungsplan enthält.

### `SPI_prepare_cursor`

`SPI_prepare_cursor` - eine Anweisung vorbereiten, ohne sie bereits auszuführen

**Synopsis**

```c
SPIPlanPtr SPI_prepare_cursor(const char * command, int nargs,
                              Oid * argtypes, int cursorOptions)
```

**Beschreibung**

`SPI_prepare_cursor` ist identisch mit `SPI_prepare`, außer dass es außerdem die Angabe des Planner-Parameters „cursor options“ erlaubt. Dies ist eine Bitmaske mit den Werten, die in `nodes/parsenodes.h` für das Feld `options` von `DeclareCursorStmt` gezeigt werden. `SPI_prepare` verwendet für die Cursor-Optionen immer null.

Diese Funktion ist inzwischen zugunsten von `SPI_prepare_extended` veraltet.

**Argumente**

- `const char * command`
  - Befehlszeichenkette.

- `int nargs`
  - Anzahl der Eingabeparameter (`$1`, `$2` usw.).

- `Oid * argtypes`
  - Zeiger auf ein Array, das die OIDs der Datentypen der Parameter enthält.

- `int cursorOptions`
  - Integer-Bitmaske von Cursor-Optionen; null erzeugt das Standardverhalten.

**Rückgabewert**

`SPI_prepare_cursor` hat dieselben Rückgabekonventionen wie `SPI_prepare`.

**Hinweise**

Nützliche Bits, die in `cursorOptions` gesetzt werden können, sind `CURSOR_OPT_SCROLL`, `CURSOR_OPT_NO_SCROLL`, `CURSOR_OPT_FAST_PLAN`, `CURSOR_OPT_GENERIC_PLAN` und `CURSOR_OPT_CUSTOM_PLAN`. Beachten Sie insbesondere, dass `CURSOR_OPT_HOLD` ignoriert wird.

### `SPI_prepare_extended`

`SPI_prepare_extended` - eine Anweisung vorbereiten, ohne sie bereits auszuführen

**Synopsis**

```c
SPIPlanPtr SPI_prepare_extended(const char * command,
                                const SPIPrepareOptions * options)
```

**Beschreibung**

`SPI_prepare_extended` erzeugt eine vorbereitete Anweisung für den angegebenen Befehl und gibt sie zurück, führt den Befehl aber nicht aus. Diese Funktion ist äquivalent zu `SPI_prepare`, mit dem Zusatz, dass der Aufrufer Optionen angeben kann, um das Parsen externer Parameterreferenzen sowie andere Aspekte von Abfrageparsing und -planung zu steuern.

**Argumente**

- `const char * command`
  - Befehlszeichenkette.

- `const SPIPrepareOptions * options`
  - Struktur, die optionale Argumente enthält.

Aufrufer sollten immer die gesamte Struktur `options` mit Nullen initialisieren und dann die Felder füllen, die sie setzen möchten. Dies stellt die Vorwärtskompatibilität des Codes sicher, da alle Felder, die der Struktur künftig hinzugefügt werden, bei einem Wert von null so definiert werden, dass sie rückwärtskompatibel funktionieren. Die derzeit verfügbaren Optionsfelder sind:

- `ParserSetupHook parserSetup`
  - Parser-Hook-Setup-Funktion.

- `void * parserSetupArg`
  - Durchgereichtes Argument für `parserSetup`.

- `RawParseMode parseMode`
  - Modus für das rohe Parsen; `RAW_PARSE_DEFAULT` (null) erzeugt das Standardverhalten.

- `int cursorOptions`
  - Integer-Bitmaske von Cursor-Optionen; null erzeugt das Standardverhalten.

**Rückgabewert**

`SPI_prepare_extended` hat dieselben Rückgabekonventionen wie `SPI_prepare`.

### `SPI_prepare_params`

`SPI_prepare_params` - eine Anweisung vorbereiten, ohne sie bereits auszuführen

**Synopsis**

```c
SPIPlanPtr SPI_prepare_params(const char * command,
                              ParserSetupHook parserSetup,
                              void * parserSetupArg,
                              int cursorOptions)
```

**Beschreibung**

`SPI_prepare_params` erzeugt eine vorbereitete Anweisung für den angegebenen Befehl und gibt sie zurück, führt den Befehl aber nicht aus. Diese Funktion ist äquivalent zu `SPI_prepare_cursor`, mit dem Zusatz, dass der Aufrufer Parser-Hook-Funktionen angeben kann, um das Parsen externer Parameterreferenzen zu steuern.

Diese Funktion ist inzwischen zugunsten von `SPI_prepare_extended` veraltet.

**Argumente**

- `const char * command`
  - Befehlszeichenkette.

- `ParserSetupHook parserSetup`
  - Parser-Hook-Setup-Funktion.

- `void * parserSetupArg`
  - Durchgereichtes Argument für `parserSetup`.

- `int cursorOptions`
  - Integer-Bitmaske von Cursor-Optionen; null erzeugt das Standardverhalten.

**Rückgabewert**

`SPI_prepare_params` hat dieselben Rückgabekonventionen wie `SPI_prepare`.

### `SPI_getargcount`

`SPI_getargcount` - die Anzahl der Argumente zurückgeben, die eine mit `SPI_prepare` vorbereitete Anweisung benötigt

**Synopsis**

```c
int SPI_getargcount(SPIPlanPtr plan)
```

**Beschreibung**

`SPI_getargcount` gibt die Anzahl der Argumente zurück, die zum Ausführen einer mit `SPI_prepare` vorbereiteten Anweisung benötigt werden.

**Argumente**

- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).

**Rückgabewert**

Die Anzahl der erwarteten Argumente für den Plan. Wenn der Plan `NULL` oder ungültig ist, wird `SPI_result` auf `SPI_ERROR_ARGUMENT` gesetzt und `-1` zurückgegeben.

### `SPI_getargtypeid`

`SPI_getargtypeid` - die Datentyp-OID für ein Argument einer mit `SPI_prepare` vorbereiteten Anweisung zurückgeben

**Synopsis**

```c
Oid SPI_getargtypeid(SPIPlanPtr plan, int argIndex)
```

**Beschreibung**

`SPI_getargtypeid` gibt die OID zurück, die den Typ des `argIndex`-ten Arguments einer mit `SPI_prepare` vorbereiteten Anweisung darstellt. Das erste Argument hat den Index null.

**Argumente**

- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).

- `int argIndex`
  - Nullbasierter Index des Arguments.

**Rückgabewert**

Die Typ-OID des Arguments am angegebenen Index. Wenn der Plan `NULL` oder ungültig ist, oder wenn `argIndex` kleiner als 0 oder nicht kleiner als die Anzahl der für den Plan deklarierten Argumente ist, wird `SPI_result` auf `SPI_ERROR_ARGUMENT` gesetzt und `InvalidOid` zurückgegeben.

### `SPI_is_cursor_plan`

`SPI_is_cursor_plan` - wahr zurückgeben, wenn eine mit `SPI_prepare` vorbereitete Anweisung mit `SPI_cursor_open` verwendet werden kann

**Synopsis**

```c
bool SPI_is_cursor_plan(SPIPlanPtr plan)
```

**Beschreibung**

`SPI_is_cursor_plan` gibt wahr zurück, wenn eine mit `SPI_prepare` vorbereitete Anweisung als Argument an `SPI_cursor_open` übergeben werden kann, oder falsch, wenn dies nicht der Fall ist. Die Kriterien sind, dass der Plan einen einzelnen Befehl darstellt und dass dieser Befehl Tupel an den Aufrufer zurückgibt; zum Beispiel ist `SELECT` erlaubt, sofern es keine `INTO`-Klausel enthält, und `UPDATE` ist nur erlaubt, wenn es eine `RETURNING`-Klausel enthält.

**Argumente**

- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).

**Rückgabewert**

`true` oder `false`, um anzugeben, ob der Plan einen Cursor erzeugen kann oder nicht, wobei `SPI_result` auf null gesetzt wird. Wenn es nicht möglich ist, die Antwort zu bestimmen (zum Beispiel wenn der Plan `NULL` oder ungültig ist oder wenn der Aufruf nicht mit SPI verbunden ist), wird `SPI_result` auf einen passenden Fehlercode gesetzt und `false` zurückgegeben.

### `SPI_execute_plan`

`SPI_execute_plan` - führt eine mit `SPI_prepare` vorbereitete Anweisung aus

**Synopsis**

```c
int SPI_execute_plan(SPIPlanPtr plan, Datum * values, const char * nulls,
                     bool read_only, long count)
```

**Beschreibung**

`SPI_execute_plan` führt eine Anweisung aus, die mit `SPI_prepare` oder einer ihrer verwandten Funktionen vorbereitet wurde. `read_only` und `count` haben dieselbe Bedeutung wie bei `SPI_execute`.

**Argumente**

- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).
- `Datum * values`
  - Ein Array der tatsächlichen Parameterwerte. Es muss dieselbe Länge haben wie die Zahl der Argumente der Anweisung.
- `const char * nulls`
  - Ein Array, das beschreibt, welche Parameter null sind. Es muss dieselbe Länge haben wie die Zahl der Argumente der Anweisung.
  - Wenn `nulls` `NULL` ist, nimmt `SPI_execute_plan` an, dass keine Parameter null sind. Andernfalls sollte jeder Eintrag des Arrays `nulls` ein Leerzeichen (`' '`) sein, wenn der entsprechende Parameterwert nicht null ist, oder `'n'`, wenn der entsprechende Parameterwert null ist. Im letzteren Fall ist der tatsächliche Wert im entsprechenden Eintrag von `values` ohne Bedeutung. Beachten Sie, dass `nulls` keine Textzeichenkette ist, sondern nur ein Array; es benötigt keinen Terminator `'\0'`.
- `bool read_only`
  - `true` für schreibgeschützte Ausführung.
- `long count`
  - Maximale Zahl der zurückzugebenden Zeilen oder `0` für keine Begrenzung.

**Rückgabewert**

Der Rückgabewert ist derselbe wie bei `SPI_execute`, mit den folgenden zusätzlichen möglichen (negativen) Fehlerergebnissen:

- `SPI_ERROR_ARGUMENT`
  - Wenn `plan` `NULL` oder ungültig ist oder `count` kleiner als 0 ist.
- `SPI_ERROR_PARAM`
  - Wenn `values` `NULL` ist und `plan` mit Parametern vorbereitet wurde.

Bei Erfolg werden `SPI_processed` und `SPI_tuptable` wie bei `SPI_execute` gesetzt.

### `SPI_execute_plan_extended`

`SPI_execute_plan_extended` - führt eine mit `SPI_prepare` vorbereitete Anweisung aus

**Synopsis**

```c
int SPI_execute_plan_extended(SPIPlanPtr plan,
                              const SPIExecuteOptions * options)
```

**Beschreibung**

`SPI_execute_plan_extended` führt eine Anweisung aus, die mit `SPI_prepare` oder einer ihrer verwandten Funktionen vorbereitet wurde. Diese Funktion entspricht `SPI_execute_plan`, außer dass Informationen über die an die Abfrage zu übergebenden Parameterwerte anders dargestellt werden und zusätzliche Optionen zur Ausführungssteuerung übergeben werden können.

Abfrageparameterwerte werden durch eine Struktur `ParamListInfo` repräsentiert. Das ist praktisch, wenn Werte weitergereicht werden, die bereits in diesem Format vorliegen. Dynamische Parametersätze können ebenfalls verwendet werden, und zwar über Hook-Funktionen, die in `ParamListInfo` angegeben sind.

Außerdem können Tupel, statt immer in einer Struktur `SPI_tuptable` gesammelt zu werden, an ein vom Aufrufer bereitgestelltes Objekt `DestReceiver` übergeben werden, während sie vom Executor erzeugt werden. Das ist besonders hilfreich bei Abfragen, die viele Tupel erzeugen könnten, weil die Daten unmittelbar verarbeitet werden können, statt im Speicher angesammelt zu werden.

**Argumente**

- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).
- `const SPIExecuteOptions * options`
  - Struktur, die optionale Argumente enthält.

Aufrufer sollten die gesamte Struktur `options` immer auf null setzen und anschließend die Felder füllen, die sie setzen möchten. Dadurch bleibt Code vorwärtskompatibel, da jedes Feld, das künftig zur Struktur hinzugefügt wird, bei einem Nullwert rückwärtskompatibles Verhalten haben wird. Derzeit sind die folgenden Optionsfelder verfügbar:

- `ParamListInfo params`
  - Datenstruktur, die Abfrageparametertypen und -werte enthält; `NULL`, wenn keine vorhanden sind.
- `bool read_only`
  - `true` für schreibgeschützte Ausführung.
- `bool allow_nonatomic`
  - `true` erlaubt die nichtatomare Ausführung von `CALL`- und `DO`-Anweisungen. Dieses Feld wird jedoch ignoriert, sofern nicht das Flag `SPI_OPT_NONATOMIC` an `SPI_connect_ext` übergeben wurde.
- `bool must_return_tuples`
  - Wenn `true`, wird ein Fehler ausgelöst, falls die Abfrage nicht von einer Art ist, die Tupel zurückgibt. Dies verbietet nicht den Fall, dass sie zufällig null Tupel zurückgibt.
- `uint64 tcount`
  - Maximale Zahl der zurückzugebenden Zeilen oder `0` für keine Begrenzung.
- `DestReceiver * dest`
  - `DestReceiver`-Objekt, das alle von der Abfrage ausgegebenen Tupel erhält. Wenn `NULL`, werden Ergebnistupel wie bei `SPI_execute_plan` in einer Struktur `SPI_tuptable` gesammelt.
- `ResourceOwner owner`
  - Der Resource Owner, der während der Ausführung einen Referenzzähler auf dem Plan hält. Wenn `NULL`, wird `CurrentResourceOwner` verwendet. Für nicht gespeicherte Pläne wird dies ignoriert, da SPI für diese keine Referenzzähler erwirbt.

**Rückgabewert**

Der Rückgabewert ist derselbe wie bei `SPI_execute_plan`.

Wenn `options->dest` `NULL` ist, werden `SPI_processed` und `SPI_tuptable` wie bei `SPI_execute_plan` gesetzt. Wenn `options->dest` nicht `NULL` ist, wird `SPI_processed` auf null gesetzt und `SPI_tuptable` auf `NULL`. Wenn eine Tupelzahl benötigt wird, muss das Objekt `DestReceiver` des Aufrufers sie berechnen.

### `SPI_execute_plan_with_paramlist`

`SPI_execute_plan_with_paramlist` - führt eine mit `SPI_prepare` vorbereitete Anweisung aus

**Synopsis**

```c
int SPI_execute_plan_with_paramlist(SPIPlanPtr plan,
                                    ParamListInfo params,
                                    bool read_only,
                                    long count)
```

**Beschreibung**

`SPI_execute_plan_with_paramlist` führt eine mit `SPI_prepare` vorbereitete Anweisung aus. Diese Funktion entspricht `SPI_execute_plan`, außer dass Informationen über die an die Abfrage zu übergebenden Parameterwerte anders dargestellt werden. Die Darstellung `ParamListInfo` kann praktisch sein, wenn Werte weitergereicht werden, die bereits in diesem Format vorliegen. Sie unterstützt außerdem die Verwendung dynamischer Parametersätze über Hook-Funktionen, die in `ParamListInfo` angegeben sind.

Diese Funktion ist inzwischen zugunsten von `SPI_execute_plan_extended` veraltet.

**Argumente**

- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).
- `ParamListInfo params`
  - Datenstruktur, die Parametertypen und -werte enthält; `NULL`, wenn keine vorhanden sind.
- `bool read_only`
  - `true` für schreibgeschützte Ausführung.
- `long count`
  - Maximale Zahl der zurückzugebenden Zeilen oder `0` für keine Begrenzung.

**Rückgabewert**

Der Rückgabewert ist derselbe wie bei `SPI_execute_plan`.

Bei Erfolg werden `SPI_processed` und `SPI_tuptable` wie bei `SPI_execute_plan` gesetzt.

### `SPI_execp`

`SPI_execp` - führt eine Anweisung im Lese-/Schreibmodus aus

**Synopsis**

```c
int SPI_execp(SPIPlanPtr plan, Datum * values, const char * nulls,
              long count)
```

**Beschreibung**

`SPI_execp` ist dasselbe wie `SPI_execute_plan`, wobei der Parameter `read_only` der letzteren Funktion immer als `false` angenommen wird.

**Argumente**

- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).
- `Datum * values`
  - Ein Array der tatsächlichen Parameterwerte. Es muss dieselbe Länge haben wie die Zahl der Argumente der Anweisung.
- `const char * nulls`
  - Ein Array, das beschreibt, welche Parameter null sind. Es muss dieselbe Länge haben wie die Zahl der Argumente der Anweisung.
  - Wenn `nulls` `NULL` ist, nimmt `SPI_execp` an, dass keine Parameter null sind. Andernfalls sollte jeder Eintrag des Arrays `nulls` ein Leerzeichen (`' '`) sein, wenn der entsprechende Parameterwert nicht null ist, oder `'n'`, wenn der entsprechende Parameterwert null ist. Im letzteren Fall ist der tatsächliche Wert im entsprechenden Eintrag von `values` ohne Bedeutung. Beachten Sie, dass `nulls` keine Textzeichenkette ist, sondern nur ein Array; es benötigt keinen Terminator `'\0'`.
- `long count`
  - Maximale Zahl der zurückzugebenden Zeilen oder `0` für keine Begrenzung.

**Rückgabewert**

Siehe `SPI_execute_plan`.

Bei Erfolg werden `SPI_processed` und `SPI_tuptable` wie bei `SPI_execute` gesetzt.

### `SPI_cursor_open`

`SPI_cursor_open` - richtet einen Cursor mit einer durch `SPI_prepare` erzeugten Anweisung ein

**Synopsis**

```c
Portal SPI_cursor_open(const char * name, SPIPlanPtr plan,
                       Datum * values, const char * nulls,
                       bool read_only)
```

**Beschreibung**

`SPI_cursor_open` richtet einen Cursor ein (intern ein Portal), der eine mit `SPI_prepare` vorbereitete Anweisung ausführt. Die Parameter haben dieselbe Bedeutung wie die entsprechenden Parameter von `SPI_execute_plan`.

Die Verwendung eines Cursors, statt die Anweisung direkt auszuführen, hat zwei Vorteile. Erstens können die Ergebniszeilen in kleinen Gruppen abgerufen werden, was Speicherüberläufe bei Abfragen vermeidet, die viele Zeilen zurückgeben. Zweitens kann ein Portal die aktuelle C-Funktion überdauern; es kann tatsächlich bis zum Ende der aktuellen Transaktion leben. Wenn der Portalname an den Aufrufer der C-Funktion zurückgegeben wird, kann auf diese Weise eine Zeilenmenge als Ergebnis zurückgegeben werden.

Die übergebenen Parameterdaten werden in das Portal des Cursors kopiert, sodass sie freigegeben werden können, während der Cursor noch existiert.

**Argumente**

- `const char * name`
  - Name für das Portal oder `NULL`, damit das System einen Namen auswählt.
- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).
- `Datum * values`
  - Ein Array der tatsächlichen Parameterwerte. Es muss dieselbe Länge haben wie die Zahl der Argumente der Anweisung.
- `const char * nulls`
  - Ein Array, das beschreibt, welche Parameter null sind. Es muss dieselbe Länge haben wie die Zahl der Argumente der Anweisung.
  - Wenn `nulls` `NULL` ist, nimmt `SPI_cursor_open` an, dass keine Parameter null sind. Andernfalls sollte jeder Eintrag des Arrays `nulls` ein Leerzeichen (`' '`) sein, wenn der entsprechende Parameterwert nicht null ist, oder `'n'`, wenn der entsprechende Parameterwert null ist. Im letzteren Fall ist der tatsächliche Wert im entsprechenden Eintrag von `values` ohne Bedeutung. Beachten Sie, dass `nulls` keine Textzeichenkette ist, sondern nur ein Array; es benötigt keinen Terminator `'\0'`.
- `bool read_only`
  - `true` für schreibgeschützte Ausführung.

**Rückgabewert**

Zeiger auf das Portal, das den Cursor enthält. Beachten Sie, dass es keine Fehler-Rückgabekonvention gibt; jeder Fehler wird über `elog` gemeldet.

### `SPI_cursor_open_with_args`

`SPI_cursor_open_with_args` - richtet einen Cursor mit einer Abfrage und Parametern ein

**Synopsis**

```c
Portal SPI_cursor_open_with_args(const char *name,
                                 const char *command,
                                 int nargs, Oid *argtypes,
                                 Datum *values, const char *nulls,
                                 bool read_only, int cursorOptions)
```

**Beschreibung**

`SPI_cursor_open_with_args` richtet einen Cursor ein (intern ein Portal), der die angegebene Abfrage ausführt. Die meisten Parameter haben dieselbe Bedeutung wie die entsprechenden Parameter von `SPI_prepare_cursor` und `SPI_cursor_open`.

Für eine einmalige Abfrageausführung sollte diese Funktion gegenüber `SPI_prepare_cursor`, gefolgt von `SPI_cursor_open`, bevorzugt werden. Wenn derselbe Befehl mit vielen verschiedenen Parametern ausgeführt werden soll, kann je nach Kosten der Neuplanung im Vergleich zum Nutzen maßgeschneiderter Pläne jede der beiden Methoden schneller sein.

Die übergebenen Parameterdaten werden in das Portal des Cursors kopiert, sodass sie freigegeben werden können, während der Cursor noch existiert.

Diese Funktion ist inzwischen zugunsten von `SPI_cursor_parse_open` veraltet, das gleichwertige Funktionalität über eine modernere API zur Behandlung von Abfrageparametern bereitstellt.

**Argumente**

- `const char * name`
  - Name für das Portal oder `NULL`, damit das System einen Namen auswählt.
- `const char * command`
  - Befehlszeichenkette.
- `int nargs`
  - Anzahl der Eingabeparameter (`$1`, `$2` usw.).
- `Oid * argtypes`
  - Ein Array der Länge `nargs`, das die OIDs der Datentypen der Parameter enthält.
- `Datum * values`
  - Ein Array der Länge `nargs`, das die tatsächlichen Parameterwerte enthält.
- `const char * nulls`
  - Ein Array der Länge `nargs`, das beschreibt, welche Parameter null sind.
  - Wenn `nulls` `NULL` ist, nimmt `SPI_cursor_open_with_args` an, dass keine Parameter null sind. Andernfalls sollte jeder Eintrag des Arrays `nulls` ein Leerzeichen (`' '`) sein, wenn der entsprechende Parameterwert nicht null ist, oder `'n'`, wenn der entsprechende Parameterwert null ist. Im letzteren Fall ist der tatsächliche Wert im entsprechenden Eintrag von `values` ohne Bedeutung. Beachten Sie, dass `nulls` keine Textzeichenkette ist, sondern nur ein Array; es benötigt keinen Terminator `'\0'`.
- `bool read_only`
  - `true` für schreibgeschützte Ausführung.
- `int cursorOptions`
  - Ganzzahlige Bitmaske von Cursor-Optionen; null erzeugt das Standardverhalten.

**Rückgabewert**

Zeiger auf das Portal, das den Cursor enthält. Beachten Sie, dass es keine Fehler-Rückgabekonvention gibt; jeder Fehler wird über `elog` gemeldet.

### `SPI_cursor_open_with_paramlist`

`SPI_cursor_open_with_paramlist` - richtet einen Cursor mit Parametern ein

**Synopsis**

```c
Portal SPI_cursor_open_with_paramlist(const char *name,
                                      SPIPlanPtr plan,
                                      ParamListInfo params,
                                      bool read_only)
```

**Beschreibung**

`SPI_cursor_open_with_paramlist` richtet einen Cursor ein (intern ein Portal), der eine mit `SPI_prepare` vorbereitete Anweisung ausführt. Diese Funktion entspricht `SPI_cursor_open`, außer dass Informationen über die an die Abfrage zu übergebenden Parameterwerte anders dargestellt werden. Die Darstellung `ParamListInfo` kann praktisch sein, wenn Werte weitergereicht werden, die bereits in diesem Format vorliegen. Sie unterstützt außerdem die Verwendung dynamischer Parametersätze über Hook-Funktionen, die in `ParamListInfo` angegeben sind.

Die übergebenen Parameterdaten werden in das Portal des Cursors kopiert, sodass sie freigegeben werden können, während der Cursor noch existiert.

**Argumente**

- `const char * name`
  - Name für das Portal oder `NULL`, damit das System einen Namen auswählt.
- `SPIPlanPtr plan`
  - Vorbereitete Anweisung (von `SPI_prepare` zurückgegeben).
- `ParamListInfo params`
  - Datenstruktur, die Parametertypen und -werte enthält; `NULL`, wenn keine vorhanden sind.
- `bool read_only`
  - `true` für schreibgeschützte Ausführung.

**Rückgabewert**

Zeiger auf das Portal, das den Cursor enthält. Beachten Sie, dass es keine Fehler-Rückgabekonvention gibt; jeder Fehler wird über `elog` gemeldet.

### `SPI_cursor_parse_open`

`SPI_cursor_parse_open` - richtet einen Cursor mit einer Abfragezeichenkette und Parametern ein

**Synopsis**

```c
Portal SPI_cursor_parse_open(const char *name,
                             const char *command,
                             const SPIParseOpenOptions * options)
```

**Beschreibung**

`SPI_cursor_parse_open` richtet einen Cursor ein (intern ein Portal), der die angegebene Abfragezeichenkette ausführt. Dies ist vergleichbar mit `SPI_prepare_cursor`, gefolgt von `SPI_cursor_open_with_paramlist`, außer dass Parameterreferenzen innerhalb der Abfragezeichenkette vollständig durch die Bereitstellung eines Objekts `ParamListInfo` behandelt werden.

Für eine einmalige Abfrageausführung sollte diese Funktion gegenüber `SPI_prepare_cursor`, gefolgt von `SPI_cursor_open_with_paramlist`, bevorzugt werden. Wenn derselbe Befehl mit vielen verschiedenen Parametern ausgeführt werden soll, kann je nach Kosten der Neuplanung im Vergleich zum Nutzen maßgeschneiderter Pläne jede der beiden Methoden schneller sein.

Das Objekt `options->params` sollte normalerweise jeden Parameter mit dem Flag `PARAM_FLAG_CONST` markieren, da für die Abfrage immer ein Einmalplan verwendet wird.

Die übergebenen Parameterdaten werden in das Portal des Cursors kopiert, sodass sie freigegeben werden können, während der Cursor noch existiert.

**Argumente**

- `const char * name`
  - Name für das Portal oder `NULL`, damit das System einen Namen auswählt.
- `const char * command`
  - Befehlszeichenkette.
- `const SPIParseOpenOptions * options`
  - Struktur, die optionale Argumente enthält.

Aufrufer sollten die gesamte Struktur `options` immer auf null setzen und anschließend die Felder füllen, die sie setzen möchten. Dadurch bleibt Code vorwärtskompatibel, da jedes Feld, das künftig zur Struktur hinzugefügt wird, bei einem Nullwert rückwärtskompatibles Verhalten haben wird. Derzeit sind die folgenden Optionsfelder verfügbar:

- `ParamListInfo params`
  - Datenstruktur, die Abfrageparametertypen und -werte enthält; `NULL`, wenn keine vorhanden sind.
- `int cursorOptions`
  - Ganzzahlige Bitmaske von Cursor-Optionen; null erzeugt das Standardverhalten.
- `bool read_only`
  - `true` für schreibgeschützte Ausführung.

**Rückgabewert**

Zeiger auf das Portal, das den Cursor enthält. Beachten Sie, dass es keine Fehler-Rückgabekonvention gibt; jeder Fehler wird über `elog` gemeldet.

### `SPI_cursor_find`

`SPI_cursor_find` - findet einen vorhandenen Cursor anhand des Namens

**Synopsis**

```c
Portal SPI_cursor_find(const char * name)
```

**Beschreibung**

`SPI_cursor_find` findet ein vorhandenes Portal anhand seines Namens. Dies ist vor allem nützlich, um einen Cursornamen aufzulösen, der von einer anderen Funktion als Text zurückgegeben wurde.

**Argumente**

- `const char * name`
  - Name des Portals.

**Rückgabewert**

Zeiger auf das Portal mit dem angegebenen Namen oder `NULL`, wenn keines gefunden wurde.

**Hinweise**

Beachten Sie, dass diese Funktion ein Objekt `Portal` zurückgeben kann, das keine cursorartigen Eigenschaften hat; zum Beispiel gibt es möglicherweise keine Tupel zurück. Wenn Sie den Zeiger `Portal` einfach an andere SPI-Funktionen übergeben, können diese sich gegen solche Fälle schützen, aber beim direkten Inspizieren des Portals ist Vorsicht angebracht.

### `SPI_cursor_fetch`

`SPI_cursor_fetch` - holt einige Zeilen aus einem Cursor

**Synopsis**

```c
void SPI_cursor_fetch(Portal portal, bool forward, long count)
```

**Beschreibung**

`SPI_cursor_fetch` holt einige Zeilen aus einem Cursor. Dies entspricht einer Teilmenge des SQL-Befehls `FETCH`; für mehr Funktionalität siehe `SPI_scroll_cursor_fetch`.

**Argumente**

- `Portal portal`
  - Portal, das den Cursor enthält.
- `bool forward`
  - `true` für Vorwärts-Fetch, `false` für Rückwärts-Fetch.
- `long count`
  - Maximale Zahl der abzurufenden Zeilen.

**Rückgabewert**

Bei Erfolg werden `SPI_processed` und `SPI_tuptable` wie bei `SPI_execute` gesetzt.

**Hinweise**

Rückwärts-Fetch kann fehlschlagen, wenn der Plan des Cursors nicht mit der Option `CURSOR_OPT_SCROLL` erstellt wurde.

### `SPI_cursor_move`

`SPI_cursor_move` - bewegt einen Cursor

**Synopsis**

```c
void SPI_cursor_move(Portal portal, bool forward, long count)
```

**Beschreibung**

`SPI_cursor_move` überspringt eine bestimmte Zahl von Zeilen in einem Cursor. Dies entspricht einer Teilmenge des SQL-Befehls `MOVE`; für mehr Funktionalität siehe `SPI_scroll_cursor_move`.

**Argumente**

- `Portal portal`
  - Portal, das den Cursor enthält.
- `bool forward`
  - `true` für Vorwärtsbewegung, `false` für Rückwärtsbewegung.
- `long count`
  - Maximale Zahl der zu überspringenden Zeilen.

**Hinweise**

Rückwärtsbewegung kann fehlschlagen, wenn der Plan des Cursors nicht mit der Option `CURSOR_OPT_SCROLL` erstellt wurde.

### `SPI_scroll_cursor_fetch`

`SPI_scroll_cursor_fetch` - holt einige Zeilen aus einem Cursor

**Synopsis**

```c
void SPI_scroll_cursor_fetch(Portal portal, FetchDirection direction,
                             long count)
```

**Beschreibung**

`SPI_scroll_cursor_fetch` holt einige Zeilen aus einem Cursor. Dies entspricht dem SQL-Befehl `FETCH`.

**Argumente**

- `Portal portal`
  - Portal, das den Cursor enthält.
- `FetchDirection direction`
  - Einer von `FETCH_FORWARD`, `FETCH_BACKWARD`, `FETCH_ABSOLUTE` oder `FETCH_RELATIVE`.
- `long count`
  - Zahl der abzurufenden Zeilen für `FETCH_FORWARD` oder `FETCH_BACKWARD`, absolute Zeilennummer, die für `FETCH_ABSOLUTE` abgerufen werden soll, oder relative Zeilennummer, die für `FETCH_RELATIVE` abgerufen werden soll.

**Rückgabewert**

Bei Erfolg werden `SPI_processed` und `SPI_tuptable` wie bei `SPI_execute` gesetzt.

**Hinweise**

Einzelheiten zur Interpretation der Parameter `direction` und `count` finden Sie beim SQL-Befehl `FETCH`.

Richtungswerte außer `FETCH_FORWARD` können fehlschlagen, wenn der Plan des Cursors nicht mit der Option `CURSOR_OPT_SCROLL` erstellt wurde.

### `SPI_scroll_cursor_move`

`SPI_scroll_cursor_move` - bewegt einen Cursor

**Synopsis**

```c
void SPI_scroll_cursor_move(Portal portal, FetchDirection direction,
                            long count)
```

**Beschreibung**

`SPI_scroll_cursor_move` überspringt eine bestimmte Zahl von Zeilen in einem Cursor. Dies entspricht dem SQL-Befehl `MOVE`.

**Argumente**

- `Portal portal`
  - Portal, das den Cursor enthält.
- `FetchDirection direction`
  - Einer von `FETCH_FORWARD`, `FETCH_BACKWARD`, `FETCH_ABSOLUTE` oder `FETCH_RELATIVE`.
- `long count`
  - Zahl der zu überspringenden Zeilen für `FETCH_FORWARD` oder `FETCH_BACKWARD`, absolute Zeilennummer, zu der bei `FETCH_ABSOLUTE` bewegt werden soll, oder relative Zeilennummer, um die bei `FETCH_RELATIVE` bewegt werden soll.

**Rückgabewert**

Bei Erfolg wird `SPI_processed` wie bei `SPI_execute` gesetzt. `SPI_tuptable` wird auf `NULL` gesetzt, da diese Funktion keine Zeilen zurückgibt.

**Hinweise**

Einzelheiten zur Interpretation der Parameter `direction` und `count` finden Sie beim SQL-Befehl `FETCH`.

Richtungswerte außer `FETCH_FORWARD` können fehlschlagen, wenn der Plan des Cursors nicht mit der Option `CURSOR_OPT_SCROLL` erstellt wurde.

### `SPI_cursor_close`

`SPI_cursor_close` - schließt einen Cursor

**Synopsis**

```c
void SPI_cursor_close(Portal portal)
```

**Beschreibung**

`SPI_cursor_close` schließt einen zuvor erzeugten Cursor und gibt den Portal-Speicher frei.

Alle geöffneten Cursor werden am Ende einer Transaktion automatisch geschlossen. `SPI_cursor_close` muss nur aufgerufen werden, wenn Ressourcen früher freigegeben werden sollen.

**Argumente**

- `Portal portal`
  - Portal, das den Cursor enthält.

### `SPI_keepplan`

`SPI_keepplan` - speichert eine vorbereitete Anweisung

**Synopsis**

```c
int SPI_keepplan(SPIPlanPtr plan)
```

**Beschreibung**

`SPI_keepplan` speichert eine übergebene Anweisung (vorbereitet mit `SPI_prepare`), sodass sie weder von `SPI_finish` noch vom Transaktionsmanager freigegeben wird. Dadurch können Sie vorbereitete Anweisungen bei späteren Aufrufen Ihrer C-Funktion in der aktuellen Sitzung wiederverwenden.

**Argumente**

- `SPIPlanPtr plan`
  - Die zu speichernde vorbereitete Anweisung.

**Rückgabewert**

`0` bei Erfolg; `SPI_ERROR_ARGUMENT`, wenn `plan` `NULL` oder ungültig ist.

**Hinweise**

Die übergebene Anweisung wird durch Zeigeranpassung in dauerhaften Speicher verlagert; es ist kein Kopieren von Daten erforderlich. Wenn Sie sie später löschen möchten, verwenden Sie dafür `SPI_freeplan`.

### `SPI_saveplan`

`SPI_saveplan` - speichert eine vorbereitete Anweisung

**Synopsis**

```c
SPIPlanPtr SPI_saveplan(SPIPlanPtr plan)
```

**Beschreibung**

`SPI_saveplan` kopiert eine übergebene Anweisung (vorbereitet mit `SPI_prepare`) in Speicher, der weder von `SPI_finish` noch vom Transaktionsmanager freigegeben wird, und gibt einen Zeiger auf die kopierte Anweisung zurück. Dadurch können Sie vorbereitete Anweisungen bei späteren Aufrufen Ihrer C-Funktion in der aktuellen Sitzung wiederverwenden.

**Argumente**

- `SPIPlanPtr plan`
  - Die zu speichernde vorbereitete Anweisung.

**Rückgabewert**

Zeiger auf die kopierte Anweisung oder `NULL`, wenn der Aufruf erfolglos war. Bei einem Fehler wird `SPI_result` wie folgt gesetzt:

- `SPI_ERROR_ARGUMENT`
  - Wenn `plan` `NULL` oder ungültig ist.
- `SPI_ERROR_UNCONNECTED`
  - Wenn der Aufruf aus einer nicht verbundenen C-Funktion erfolgt.

**Hinweise**

Die ursprünglich übergebene Anweisung wird nicht freigegeben; Sie möchten daher möglicherweise `SPI_freeplan` darauf anwenden, um zu vermeiden, dass bis zu `SPI_finish` Speicher verloren geht.

In den meisten Fällen ist `SPI_keepplan` dieser Funktion vorzuziehen, da sie weitgehend dasselbe Ergebnis erzielt, ohne die Datenstrukturen der vorbereiteten Anweisung physisch kopieren zu müssen.

### `SPI_register_relation`

`SPI_register_relation` - macht eine flüchtige benannte Relation unter ihrem Namen in SPI-Abfragen verfügbar

**Synopsis**

```c
int SPI_register_relation(EphemeralNamedRelation enr)
```

**Beschreibung**

`SPI_register_relation` macht eine flüchtige benannte Relation mit den zugehörigen Informationen für Abfragen verfügbar, die über die aktuelle SPI-Verbindung geplant und ausgeführt werden.

**Argumente**

- `EphemeralNamedRelation enr`
  - Der Registrierungseintrag der flüchtigen benannten Relation.

**Rückgabewert**

Wenn die Ausführung des Befehls erfolgreich war, wird der folgende (nichtnegative) Wert zurückgegeben:

- `SPI_OK_REL_REGISTER`
  - Wenn die Relation erfolgreich nach Namen registriert wurde.

Bei einem Fehler wird einer der folgenden negativen Werte zurückgegeben:

- `SPI_ERROR_ARGUMENT`
  - Wenn `enr` `NULL` ist oder sein Feld `name` `NULL` ist.
- `SPI_ERROR_UNCONNECTED`
  - Wenn der Aufruf aus einer nicht verbundenen C-Funktion erfolgt.
- `SPI_ERROR_REL_DUPLICATE`
  - Wenn der im Feld `name` von `enr` angegebene Name für diese Verbindung bereits registriert ist.

### `SPI_unregister_relation`

`SPI_unregister_relation` - entfernt eine flüchtige benannte Relation aus der Registrierung

**Synopsis**

```c
int SPI_unregister_relation(const char * name)
```

**Beschreibung**

`SPI_unregister_relation` entfernt eine flüchtige benannte Relation aus der Registrierung für die aktuelle Verbindung.

**Argumente**

- `const char * name`
  - Der Name des Relation-Registrierungseintrags.

**Rückgabewert**

Wenn die Ausführung des Befehls erfolgreich war, wird der folgende (nichtnegative) Wert zurückgegeben:

- `SPI_OK_REL_UNREGISTER`
  - Wenn der Tuplestore erfolgreich aus der Registrierung entfernt wurde.

Bei einem Fehler wird einer der folgenden negativen Werte zurückgegeben:

- `SPI_ERROR_ARGUMENT`
  - Wenn `name` `NULL` ist.
- `SPI_ERROR_UNCONNECTED`
  - Wenn der Aufruf aus einer nicht verbundenen C-Funktion erfolgt.
- `SPI_ERROR_REL_NOT_FOUND`
  - Wenn `name` in der Registrierung für die aktuelle Verbindung nicht gefunden wird.

### `SPI_register_trigger_data`

`SPI_register_trigger_data` - macht flüchtige Trigger-Daten in SPI-Abfragen verfügbar

**Synopsis**

```c
int SPI_register_trigger_data(TriggerData *tdata)
```

**Beschreibung**

`SPI_register_trigger_data` macht alle von einem Trigger erfassten flüchtigen Relationen für Abfragen verfügbar, die über die aktuelle SPI-Verbindung geplant und ausgeführt werden. Derzeit sind damit die Übergangstabellen gemeint, die von einem `AFTER`-Trigger erfasst werden, der mit einer Klausel `REFERENCING OLD/NEW TABLE AS ...` definiert ist. Diese Funktion sollte von einer PL-Trigger-Handler-Funktion nach dem Verbinden aufgerufen werden.

**Argumente**

- `TriggerData *tdata`
  - Das Objekt `TriggerData`, das als `fcinfo->context` an eine Trigger-Handler-Funktion übergeben wurde.

**Rückgabewert**

Wenn die Ausführung des Befehls erfolgreich war, wird der folgende (nichtnegative) Wert zurückgegeben:

- `SPI_OK_TD_REGISTER`
  - Wenn die erfassten Trigger-Daten (falls vorhanden) erfolgreich registriert wurden.

Bei einem Fehler wird einer der folgenden negativen Werte zurückgegeben:

- `SPI_ERROR_ARGUMENT`
  - Wenn `tdata` `NULL` ist.
- `SPI_ERROR_UNCONNECTED`
  - Wenn der Aufruf aus einer nicht verbundenen C-Funktion erfolgt.
- `SPI_ERROR_REL_DUPLICATE`
  - Wenn der Name einer transienten Relation der Trigger-Daten für diese Verbindung bereits registriert ist.

## 45.2. Schnittstellen-Hilfsfunktionen

Die hier beschriebenen Funktionen stellen eine Schnittstelle bereit, um Informationen aus Ergebnismengen zu extrahieren, die von `SPI_execute` und anderen SPI-Funktionen zurückgegeben wurden.

Alle in diesem Abschnitt beschriebenen Funktionen können sowohl von verbundenen als auch von nicht verbundenen C-Funktionen verwendet werden.

### `SPI_fname`

`SPI_fname` - bestimmt den Spaltennamen für die angegebene Spaltennummer

**Synopsis**

```c
char * SPI_fname(TupleDesc rowdesc, int colnumber)
```

**Beschreibung**

`SPI_fname` gibt eine Kopie des Spaltennamens der angegebenen Spalte zurück. Sie können `pfree` verwenden, um die Kopie des Namens freizugeben, wenn Sie sie nicht mehr benötigen.

**Argumente**

- `TupleDesc rowdesc`
  - Eingabe-Zeilenbeschreibung.
- `int colnumber`
  - Spaltennummer; die Zählung beginnt bei 1.

**Rückgabewert**

Der Spaltenname; `NULL`, wenn `colnumber` außerhalb des gültigen Bereichs liegt. Bei einem Fehler wird `SPI_result` auf `SPI_ERROR_NOATTRIBUTE` gesetzt.

### `SPI_fnumber`

`SPI_fnumber` - bestimmt die Spaltennummer für den angegebenen Spaltennamen

**Synopsis**

```c
int SPI_fnumber(TupleDesc rowdesc, const char * colname)
```

**Beschreibung**

`SPI_fnumber` gibt die Spaltennummer für die Spalte mit dem angegebenen Namen zurück.

Wenn `colname` auf eine Systemspalte verweist (zum Beispiel `ctid`), wird die passende negative Spaltennummer zurückgegeben. Der Aufrufer sollte darauf achten, den Rückgabewert auf exakte Gleichheit mit `SPI_ERROR_NOATTRIBUTE` zu prüfen, um einen Fehler zu erkennen; das Prüfen des Ergebnisses auf kleiner oder gleich 0 ist nicht korrekt, sofern Systemspalten nicht abgewiesen werden sollen.

**Argumente**

- `TupleDesc rowdesc`
  - Eingabe-Zeilenbeschreibung.
- `const char * colname`
  - Spaltenname.

**Rückgabewert**

Spaltennummer; die Zählung beginnt für benutzerdefinierte Spalten bei 1. Wenn die benannte Spalte nicht gefunden wurde, wird `SPI_ERROR_NOATTRIBUTE` zurückgegeben.

### `SPI_getvalue`

`SPI_getvalue` - gibt den Zeichenkettenwert der angegebenen Spalte zurück

**Synopsis**

```c
char * SPI_getvalue(HeapTuple row, TupleDesc rowdesc, int colnumber)
```

**Beschreibung**

`SPI_getvalue` gibt die Zeichenkettendarstellung des Werts der angegebenen Spalte zurück.

Das Ergebnis wird in Speicher zurückgegeben, der mit `palloc` zugewiesen wurde. Sie können `pfree` verwenden, um den Speicher freizugeben, wenn Sie ihn nicht mehr benötigen.

**Argumente**

- `HeapTuple row`
  - Zu untersuchende Eingabezeile.
- `TupleDesc rowdesc`
  - Eingabe-Zeilenbeschreibung.
- `int colnumber`
  - Spaltennummer; die Zählung beginnt bei 1.

**Rückgabewert**

Spaltenwert oder `NULL`, wenn die Spalte null ist, `colnumber` außerhalb des gültigen Bereichs liegt (`SPI_result` wird auf `SPI_ERROR_NOATTRIBUTE` gesetzt) oder keine Ausgabefunktion verfügbar ist (`SPI_result` wird auf `SPI_ERROR_NOOUTFUNC` gesetzt).

### `SPI_getbinval`

`SPI_getbinval` - gibt den Binärwert der angegebenen Spalte zurück

**Synopsis**

```c
Datum SPI_getbinval(HeapTuple row, TupleDesc rowdesc, int colnumber,
                    bool * isnull)
```

**Beschreibung**

`SPI_getbinval` gibt den Wert der angegebenen Spalte in interner Form zurück, als Typ `Datum`.

Diese Funktion weist keinen neuen Speicherplatz für das Datum zu. Bei einem by-reference übergebenen Datentyp ist der Rückgabewert ein Zeiger in die übergebene Zeile.

**Argumente**

- `HeapTuple row`
  - Zu untersuchende Eingabezeile.
- `TupleDesc rowdesc`
  - Eingabe-Zeilenbeschreibung.
- `int colnumber`
  - Spaltennummer; die Zählung beginnt bei 1.
- `bool * isnull`
  - Kennzeichen für einen Nullwert in der Spalte.

**Rückgabewert**

Der Binärwert der Spalte wird zurückgegeben. Die Variable, auf die `isnull` zeigt, wird auf `true` gesetzt, wenn die Spalte null ist, sonst auf `false`.

Bei einem Fehler wird `SPI_result` auf `SPI_ERROR_NOATTRIBUTE` gesetzt.

### `SPI_gettype`

`SPI_gettype` - gibt den Datentypnamen der angegebenen Spalte zurück

**Synopsis**

```c
char * SPI_gettype(TupleDesc rowdesc, int colnumber)
```

**Beschreibung**

`SPI_gettype` gibt eine Kopie des Datentypnamens der angegebenen Spalte zurück. Sie können `pfree` verwenden, um die Kopie des Namens freizugeben, wenn Sie sie nicht mehr benötigen.

**Argumente**

- `TupleDesc rowdesc`
  - Eingabe-Zeilenbeschreibung.
- `int colnumber`
  - Spaltennummer; die Zählung beginnt bei 1.

**Rückgabewert**

Der Datentypname der angegebenen Spalte oder `NULL` bei einem Fehler. Bei einem Fehler wird `SPI_result` auf `SPI_ERROR_NOATTRIBUTE` gesetzt.

### `SPI_gettypeid`

`SPI_gettypeid` - gibt die Datentyp-OID der angegebenen Spalte zurück

**Synopsis**

```c
Oid SPI_gettypeid(TupleDesc rowdesc, int colnumber)
```

**Beschreibung**

`SPI_gettypeid` gibt die OID des Datentyps der angegebenen Spalte zurück.

**Argumente**

- `TupleDesc rowdesc`
  - Eingabe-Zeilenbeschreibung.
- `int colnumber`
  - Spaltennummer; die Zählung beginnt bei 1.

**Rückgabewert**

Die OID des Datentyps der angegebenen Spalte oder `InvalidOid` bei einem Fehler. Bei einem Fehler wird `SPI_result` auf `SPI_ERROR_NOATTRIBUTE` gesetzt.

### `SPI_getrelname`

`SPI_getrelname` - gibt den Namen der angegebenen Relation zurück

**Synopsis**

```c
char * SPI_getrelname(Relation rel)
```

**Beschreibung**

`SPI_getrelname` gibt eine Kopie des Namens der angegebenen Relation zurück. Sie können `pfree` verwenden, um die Kopie des Namens freizugeben, wenn Sie sie nicht mehr benötigen.

**Argumente**

- `Relation rel`
  - Eingaberelation.

**Rückgabewert**

Der Name der angegebenen Relation.

### `SPI_getnspname`

`SPI_getnspname` - gibt den Namensraum der angegebenen Relation zurück

**Synopsis**

```c
char * SPI_getnspname(Relation rel)
```

**Beschreibung**

`SPI_getnspname` gibt eine Kopie des Namens des Namensraums zurück, zu dem die angegebene `Relation` gehört. Dies entspricht dem Schema der Relation. Sie sollten den Rückgabewert dieser Funktion mit `pfree` freigeben, wenn Sie damit fertig sind.

**Argumente**

- `Relation rel`
  - Eingaberelation.

**Rückgabewert**

Der Name des Namensraums der angegebenen Relation.

### `SPI_result_code_string`

`SPI_result_code_string` - gibt einen Fehlercode als Zeichenkette zurück

**Synopsis**

```c
const char * SPI_result_code_string(int code);
```

**Beschreibung**

`SPI_result_code_string` gibt eine Zeichenkettendarstellung des Ergebnis-Codes zurück, der von verschiedenen SPI-Funktionen zurückgegeben oder in `SPI_result` gespeichert wird.

**Argumente**

- `int code`
  - Ergebnis-Code.

**Rückgabewert**

Eine Zeichenkettendarstellung des Ergebnis-Codes.

## 45.3. Speicherverwaltung

PostgreSQL weist Speicher innerhalb von Memory Contexts zu. Diese bieten eine praktische Methode, um Speicherzuweisungen zu verwalten, die an vielen verschiedenen Stellen erfolgen und unterschiedlich lange leben müssen. Das Zerstören eines Contexts gibt den gesamten Speicher frei, der darin zugewiesen wurde. Es ist daher nicht nötig, einzelne Objekte zu verfolgen, um Speicherlecks zu vermeiden; stattdessen muss nur eine relativ kleine Zahl von Contexts verwaltet werden. `palloc` und verwandte Funktionen weisen Speicher aus dem "aktuellen" Context zu.

`SPI_connect` erzeugt einen neuen Memory Context und macht ihn zum aktuellen Context. `SPI_finish` stellt den vorherigen aktuellen Memory Context wieder her und zerstört den von `SPI_connect` erzeugten Context. Diese Aktionen stellen sicher, dass vorübergehende Speicherzuweisungen innerhalb Ihrer C-Funktion beim Verlassen der C-Funktion zurückgewonnen werden, wodurch Speicherlecks vermieden werden.

Wenn Ihre C-Funktion jedoch ein Objekt in zugewiesenem Speicher zurückgeben muss (zum Beispiel einen Wert eines by-reference übergebenen Datentyps), können Sie diesen Speicher nicht mit `palloc` zuweisen, zumindest nicht während Sie mit SPI verbunden sind. Wenn Sie es versuchen, wird das Objekt durch `SPI_finish` freigegeben, und Ihre C-Funktion wird nicht zuverlässig funktionieren. Um dieses Problem zu lösen, verwenden Sie `SPI_palloc`, um Speicher für Ihr Rückgabeobjekt zuzuweisen. `SPI_palloc` weist Speicher im "upper executor context" zu, also in dem Memory Context, der aktuell war, als `SPI_connect` aufgerufen wurde. Das ist genau der richtige Context für einen Wert, der von Ihrer C-Funktion zurückgegeben wird. Mehrere der anderen in diesem Abschnitt beschriebenen Hilfsfunktionen geben ebenfalls Objekte zurück, die im upper executor context erzeugt wurden.

Wenn `SPI_connect` aufgerufen wird, wird der private Context der C-Funktion, der von `SPI_connect` erzeugt wird, zum aktuellen Context gemacht. Alle Zuweisungen, die mit `palloc`, `repalloc` oder SPI-Hilfsfunktionen erfolgen (außer wie in diesem Abschnitt beschrieben), werden in diesem Context vorgenommen. Wenn eine C-Funktion die Verbindung zum SPI-Manager trennt (über `SPI_finish`), wird der aktuelle Context auf den upper executor context zurückgesetzt, und alle im Memory Context der C-Funktion vorgenommenen Zuweisungen werden freigegeben und können nicht mehr verwendet werden.

### `SPI_palloc`

`SPI_palloc` - weist Speicher im upper executor context zu

**Synopsis**

```c
void * SPI_palloc(Size size)
```

**Beschreibung**

`SPI_palloc` weist Speicher im upper executor context zu.

Diese Funktion kann nur verwendet werden, während eine Verbindung zu SPI besteht. Andernfalls löst sie einen Fehler aus.

**Argumente**

- `Size size`
  - Größe des zuzuweisenden Speicherbereichs in Bytes.

**Rückgabewert**

Zeiger auf neuen Speicherplatz der angegebenen Größe.

### `SPI_repalloc`

`SPI_repalloc` - weist Speicher im upper executor context neu zu

**Synopsis**

```c
void * SPI_repalloc(void * pointer, Size size)
```

**Beschreibung**

`SPI_repalloc` ändert die Größe eines Speichersegments, das zuvor mit `SPI_palloc` zugewiesen wurde.

Diese Funktion unterscheidet sich nicht mehr von einfachem `repalloc`. Sie wird nur zur Rückwärtskompatibilität bestehenden Codes beibehalten.

**Argumente**

- `void * pointer`
  - Zeiger auf den vorhandenen Speicher, der geändert werden soll.
- `Size size`
  - Größe des zuzuweisenden Speicherbereichs in Bytes.

**Rückgabewert**

Zeiger auf neuen Speicherplatz der angegebenen Größe, wobei der Inhalt aus dem vorhandenen Bereich kopiert wurde.

### `SPI_pfree`

`SPI_pfree` - gibt Speicher im upper executor context frei

**Synopsis**

```c
void SPI_pfree(void * pointer)
```

**Beschreibung**

`SPI_pfree` gibt Speicher frei, der zuvor mit `SPI_palloc` oder `SPI_repalloc` zugewiesen wurde.

Diese Funktion unterscheidet sich nicht mehr von einfachem `pfree`. Sie wird nur zur Rückwärtskompatibilität bestehenden Codes beibehalten.

**Argumente**

- `void * pointer`
  - Zeiger auf den vorhandenen Speicher, der freigegeben werden soll.

### `SPI_copytuple`

`SPI_copytuple` - erzeugt eine Kopie einer Zeile im upper executor context

**Synopsis**

```c
HeapTuple SPI_copytuple(HeapTuple row)
```

**Beschreibung**

`SPI_copytuple` erzeugt eine Kopie einer Zeile im upper executor context. Dies wird normalerweise verwendet, um eine geänderte Zeile aus einem Trigger zurückzugeben. Verwenden Sie stattdessen `SPI_returntuple` in einer Funktion, die so deklariert ist, dass sie einen zusammengesetzten Typ zurückgibt.

Diese Funktion kann nur verwendet werden, während eine Verbindung zu SPI besteht. Andernfalls gibt sie `NULL` zurück und setzt `SPI_result` auf `SPI_ERROR_UNCONNECTED`.

**Argumente**

- `HeapTuple row`
  - Zu kopierende Zeile.

**Rückgabewert**

Die kopierte Zeile oder `NULL` bei einem Fehler; siehe `SPI_result` für eine Fehleranzeige.

### `SPI_returntuple`

`SPI_returntuple` - bereitet die Rückgabe eines Tupels als `Datum` vor

**Synopsis**

```c
HeapTupleHeader SPI_returntuple(HeapTuple row, TupleDesc rowdesc)
```

**Beschreibung**

`SPI_returntuple` erzeugt eine Kopie einer Zeile im upper executor context und gibt sie in Form eines Zeilentyp-`Datum` zurück. Der zurückgegebene Zeiger muss vor der Rückgabe nur mit `PointerGetDatum` in `Datum` umgewandelt werden.

Diese Funktion kann nur verwendet werden, während eine Verbindung zu SPI besteht. Andernfalls gibt sie `NULL` zurück und setzt `SPI_result` auf `SPI_ERROR_UNCONNECTED`.

Beachten Sie, dass dies für Funktionen verwendet werden sollte, die so deklariert sind, dass sie zusammengesetzte Typen zurückgeben. Für Trigger wird sie nicht verwendet; verwenden Sie `SPI_copytuple`, um eine geänderte Zeile in einem Trigger zurückzugeben.

**Argumente**

- `HeapTuple row`
  - Zu kopierende Zeile.
- `TupleDesc rowdesc`
  - Deskriptor für die Zeile. Übergeben Sie für möglichst wirksames Caching jedes Mal denselben Deskriptor.

**Rückgabewert**

`HeapTupleHeader`, der auf die kopierte Zeile zeigt, oder `NULL` bei einem Fehler; siehe `SPI_result` für eine Fehleranzeige.

### `SPI_modifytuple`

`SPI_modifytuple` - erzeugt eine Zeile, indem ausgewählte Felder einer gegebenen Zeile ersetzt werden

**Synopsis**

```c
HeapTuple SPI_modifytuple(Relation rel, HeapTuple row, int ncols,
                          int * colnum, Datum * values,
                          const char * nulls)
```

**Beschreibung**

`SPI_modifytuple` erzeugt eine neue Zeile, indem neue Werte für ausgewählte Spalten eingesetzt und die Spalten der ursprünglichen Zeile an anderen Positionen kopiert werden. Die Eingabezeile wird nicht geändert. Die neue Zeile wird im upper executor context zurückgegeben.

Diese Funktion kann nur verwendet werden, während eine Verbindung zu SPI besteht. Andernfalls gibt sie `NULL` zurück und setzt `SPI_result` auf `SPI_ERROR_UNCONNECTED`.

**Argumente**

- `Relation rel`
  - Wird nur als Quelle des Zeilendeskriptors für die Zeile verwendet. Das Übergeben einer Relation statt eines Zeilendeskriptors ist eine Fehlkonstruktion.
- `HeapTuple row`
  - Zu ändernde Zeile.
- `int ncols`
  - Anzahl der zu ändernden Spalten.
- `int * colnum`
  - Ein Array der Länge `ncols`, das die Nummern der zu ändernden Spalten enthält. Spaltennummern beginnen bei 1.
- `Datum * values`
  - Ein Array der Länge `ncols`, das die neuen Werte für die angegebenen Spalten enthält.
- `const char * nulls`
  - Ein Array der Länge `ncols`, das beschreibt, welche neuen Werte null sind.
  - Wenn `nulls` `NULL` ist, nimmt `SPI_modifytuple` an, dass keine neuen Werte null sind. Andernfalls sollte jeder Eintrag des Arrays `nulls` ein Leerzeichen (`' '`) sein, wenn der entsprechende neue Wert nicht null ist, oder `'n'`, wenn der entsprechende neue Wert null ist. Im letzteren Fall ist der tatsächliche Wert im entsprechenden Eintrag von `values` ohne Bedeutung. Beachten Sie, dass `nulls` keine Textzeichenkette ist, sondern nur ein Array; es benötigt keinen Terminator `'\0'`.

**Rückgabewert**

Neue Zeile mit Änderungen, im upper executor context zugewiesen, oder `NULL` bei einem Fehler; siehe `SPI_result` für eine Fehleranzeige.

Bei einem Fehler wird `SPI_result` wie folgt gesetzt:

- `SPI_ERROR_ARGUMENT`
  - Wenn `rel` `NULL` ist, wenn `row` `NULL` ist, wenn `ncols` kleiner oder gleich 0 ist, wenn `colnum` `NULL` ist oder wenn `values` `NULL` ist.
- `SPI_ERROR_NOATTRIBUTE`
  - Wenn `colnum` eine ungültige Spaltennummer enthält, also kleiner oder gleich 0 oder größer als die Zahl der Spalten in `row`.
- `SPI_ERROR_UNCONNECTED`
  - Wenn SPI nicht aktiv ist.

### `SPI_freetuple`

`SPI_freetuple` - gibt eine im upper executor context zugewiesene Zeile frei

**Synopsis**

```c
void SPI_freetuple(HeapTuple row)
```

**Beschreibung**

`SPI_freetuple` gibt eine Zeile frei, die zuvor im upper executor context zugewiesen wurde.

Diese Funktion unterscheidet sich nicht mehr von einfachem `heap_freetuple`. Sie wird nur zur Rückwärtskompatibilität bestehenden Codes beibehalten.

**Argumente**

- `HeapTuple row`
  - Freizugebende Zeile.

### `SPI_freetuptable`

`SPI_freetuptable` - gibt eine von `SPI_execute` oder einer ähnlichen Funktion erzeugte Zeilenmenge frei

**Synopsis**

```c
void SPI_freetuptable(SPITupleTable * tuptable)
```

**Beschreibung**

`SPI_freetuptable` gibt eine Zeilenmenge frei, die von einer früheren SPI-Befehlsausführungsfunktion wie `SPI_execute` erzeugt wurde. Daher wird diese Funktion häufig mit der globalen Variablen `SPI_tuptable` als Argument aufgerufen.

Diese Funktion ist nützlich, wenn eine SPI-verwendende C-Funktion mehrere Befehle ausführen muss und die Ergebnisse früherer Befehle nicht bis zu ihrem Ende behalten möchte. Beachten Sie, dass alle nicht freigegebenen Zeilenmengen ohnehin bei `SPI_finish` freigegeben werden. Außerdem gibt SPI automatisch alle Zeilenmengen frei, die während einer Subtransaktion erzeugt wurden, wenn innerhalb der Ausführung einer SPI-verwendenden C-Funktion eine Subtransaktion gestartet und dann abgebrochen wird.

Seit PostgreSQL 9.3 enthält `SPI_freetuptable` Schutzlogik gegen doppelte Löschanforderungen für dieselbe Zeilenmenge. In früheren Versionen führten doppelte Löschungen zu Abstürzen.

**Argumente**

- `SPITupleTable * tuptable`
  - Zeiger auf die freizugebende Zeilenmenge oder `NULL`, um nichts zu tun.

### `SPI_freeplan`

`SPI_freeplan` - gibt eine zuvor gespeicherte vorbereitete Anweisung frei

**Synopsis**

```c
int SPI_freeplan(SPIPlanPtr plan)
```

**Beschreibung**

`SPI_freeplan` gibt eine vorbereitete Anweisung frei, die zuvor von `SPI_prepare` zurückgegeben oder von `SPI_keepplan` beziehungsweise `SPI_saveplan` gespeichert wurde.

**Argumente**

- `SPIPlanPtr plan`
  - Zeiger auf die freizugebende Anweisung.

**Rückgabewert**

`0` bei Erfolg; `SPI_ERROR_ARGUMENT`, wenn `plan` `NULL` oder ungültig ist.

## 45.4. Transaktionsverwaltung

Es ist nicht möglich, Transaktionssteuerungsbefehle wie `COMMIT` und `ROLLBACK` über SPI-Funktionen wie `SPI_execute` auszuführen. Es gibt jedoch separate Schnittstellenfunktionen, die Transaktionssteuerung über SPI erlauben.

Es ist im Allgemeinen weder sicher noch sinnvoll, Transaktionen in beliebigen benutzerdefinierten, aus SQL aufrufbaren Funktionen zu beginnen und zu beenden, ohne den Kontext zu berücksichtigen, in dem sie aufgerufen werden. Zum Beispiel führt eine Transaktionsgrenze mitten in einer Funktion, die Teil eines komplexen SQL-Ausdrucks ist, der wiederum Teil eines SQL-Befehls ist, wahrscheinlich zu schwer durchschaubaren internen Fehlern oder Abstürzen. Die hier vorgestellten Schnittstellenfunktionen sind in erster Linie für Implementierungen prozeduraler Sprachen gedacht, um Transaktionsverwaltung in SQL-Prozeduren zu unterstützen, die auf SQL-Ebene durch den Befehl `CALL` aufgerufen werden, wobei der Kontext des `CALL`-Aufrufs berücksichtigt wird. SPI-verwendende, in C implementierte Prozeduren können dieselbe Logik implementieren, aber die Einzelheiten dazu liegen außerhalb des Umfangs dieser Dokumentation.

### `SPI_commit`

`SPI_commit`, `SPI_commit_and_chain` - schreiben die aktuelle Transaktion fest

**Synopsis**

```c
void SPI_commit(void)

void SPI_commit_and_chain(void)
```

**Beschreibung**

`SPI_commit` schreibt die aktuelle Transaktion fest. Dies entspricht ungefähr der Ausführung des SQL-Befehls `COMMIT`. Nachdem die Transaktion festgeschrieben wurde, wird automatisch eine neue Transaktion mit Standard-Transaktionseigenschaften gestartet, sodass der Aufrufer SPI-Einrichtungen weiter verwenden kann. Wenn beim Commit ein Fehler auftritt, wird die aktuelle Transaktion stattdessen zurückgerollt und eine neue Transaktion gestartet; anschließend wird der Fehler auf die übliche Weise ausgelöst.

`SPI_commit_and_chain` ist dasselbe, aber die neue Transaktion wird mit denselben Transaktionseigenschaften wie die gerade beendete gestartet, wie beim SQL-Befehl `COMMIT AND CHAIN`.

Diese Funktionen können nur ausgeführt werden, wenn die SPI-Verbindung beim Aufruf von `SPI_connect_ext` als nichtatomar eingerichtet wurde.

### `SPI_rollback`

`SPI_rollback`, `SPI_rollback_and_chain` - brechen die aktuelle Transaktion ab

**Synopsis**

```c
void SPI_rollback(void)

void SPI_rollback_and_chain(void)
```

**Beschreibung**

`SPI_rollback` rollt die aktuelle Transaktion zurück. Dies entspricht ungefähr der Ausführung des SQL-Befehls `ROLLBACK`. Nachdem die Transaktion zurückgerollt wurde, wird automatisch eine neue Transaktion mit Standard-Transaktionseigenschaften gestartet, sodass der Aufrufer SPI-Einrichtungen weiter verwenden kann.

`SPI_rollback_and_chain` ist dasselbe, aber die neue Transaktion wird mit denselben Transaktionseigenschaften wie die gerade beendete gestartet, wie beim SQL-Befehl `ROLLBACK AND CHAIN`.

Diese Funktionen können nur ausgeführt werden, wenn die SPI-Verbindung beim Aufruf von `SPI_connect_ext` als nichtatomar eingerichtet wurde.

### `SPI_start_transaction`

`SPI_start_transaction` - veraltete Funktion

**Synopsis**

```c
void SPI_start_transaction(void)
```

**Beschreibung**

`SPI_start_transaction` tut nichts und existiert nur für Codekompatibilität mit früheren PostgreSQL-Versionen. Früher war sie nach einem Aufruf von `SPI_commit` oder `SPI_rollback` erforderlich, aber inzwischen starten diese Funktionen automatisch eine neue Transaktion.

## 45.5. Sichtbarkeit von Datenänderungen

Die folgenden Regeln bestimmen die Sichtbarkeit von Datenänderungen in Funktionen, die SPI verwenden, oder in beliebigen anderen C-Funktionen:

- Während der Ausführung eines SQL-Befehls sind alle durch diesen Befehl vorgenommenen Datenänderungen für den Befehl selbst unsichtbar. Zum Beispiel sind bei

  ```sql
  INSERT INTO a SELECT * FROM a;
  ```

  die eingefügten Zeilen für den `SELECT`-Teil unsichtbar.

- Änderungen, die durch einen Befehl `C` vorgenommen werden, sind für alle Befehle sichtbar, die nach `C` gestartet werden, unabhängig davon, ob sie innerhalb von `C` (während der Ausführung von `C`) oder nach Abschluss von `C` gestartet werden.

- Befehle, die über SPI innerhalb einer Funktion ausgeführt werden, die von einem SQL-Befehl aufgerufen wurde (entweder eine gewöhnliche Funktion oder ein Trigger), folgen abhängig vom an SPI übergebenen Lese-/Schreib-Flag der einen oder der anderen obigen Regel. Befehle, die im schreibgeschützten Modus ausgeführt werden, folgen der ersten Regel: Sie können Änderungen des aufrufenden Befehls nicht sehen. Befehle, die im Lese-/Schreibmodus ausgeführt werden, folgen der zweiten Regel: Sie können alle bisher vorgenommenen Änderungen sehen.

- Alle standardmäßigen prozeduralen Sprachen setzen den SPI-Lese-/Schreibmodus abhängig vom Volatilitätsattribut der Funktion. Befehle von `STABLE`- und `IMMUTABLE`-Funktionen werden im schreibgeschützten Modus ausgeführt, während Befehle von `VOLATILE`-Funktionen im Lese-/Schreibmodus ausgeführt werden. Autoren von C-Funktionen können diese Konvention zwar verletzen, aber das ist wahrscheinlich keine gute Idee.

Der nächste Abschnitt enthält ein Beispiel, das die Anwendung dieser Regeln veranschaulicht.

## 45.6. Beispiele

Dieser Abschnitt enthält ein sehr einfaches Beispiel für die SPI-Verwendung. Die C-Funktion `execq` übernimmt als erstes Argument einen SQL-Befehl und als zweites eine Zeilenzahl, führt den Befehl mit `SPI_exec` aus und gibt die Zahl der Zeilen zurück, die vom Befehl verarbeitet wurden. Komplexere Beispiele für SPI finden Sie im Quellbaum in `src/test/regress/regress.c` und im Modul `spi`.

```c
#include "postgres.h"

#include "executor/spi.h"
#include "utils/builtins.h"

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(execq);

Datum
execq(PG_FUNCTION_ARGS)
{
    char *command;
    int cnt;
    int ret;
    uint64 proc;

    /* Convert given text object to a C string */
    command = text_to_cstring(PG_GETARG_TEXT_PP(0));
    cnt = PG_GETARG_INT32(1);

    SPI_connect();

    ret = SPI_exec(command, cnt);

    proc = SPI_processed;

    /*
     * If some rows were fetched, print them via elog(INFO).
     */
    if (ret > 0 && SPI_tuptable != NULL)
    {
        SPITupleTable *tuptable = SPI_tuptable;
        TupleDesc tupdesc = tuptable->tupdesc;
        char buf[8192];
        uint64 j;

        for (j = 0; j < tuptable->numvals; j++)
        {
            HeapTuple tuple = tuptable->vals[j];
            int i;

            for (i = 1, buf[0] = 0; i <= tupdesc->natts; i++)
                snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), " %s%s",
                         SPI_getvalue(tuple, tupdesc, i),
                         (i == tupdesc->natts) ? " " : " |");
            elog(INFO, "EXECQ: %s", buf);
        }
    }

    SPI_finish();
    pfree(command);

    PG_RETURN_INT64(proc);
}
```

So deklarieren Sie die Funktion, nachdem Sie sie in eine Shared Library kompiliert haben; Details stehen in [Abschnitt 36.10.5](36_SQL_erweitern.md#36105-dynamisch-geladene-funktionen-kompilieren-und-linken):

```sql
CREATE FUNCTION execq(text, integer) RETURNS int8
    AS 'filename'
    LANGUAGE C STRICT;
```

Hier ist eine Beispielsitzung:

```text
=> SELECT execq('CREATE TABLE a (x integer)', 0);
 execq
-------
(1 row)

=> INSERT INTO a VALUES (execq('INSERT INTO a VALUES (0)', 0));
INSERT 0 1
=> SELECT execq('SELECT * FROM a', 0);
INFO: EXECQ: 0      -- inserted by execq
INFO: EXECQ: 1      -- returned by execq and inserted by upper INSERT

 execq
-------
(1 row)

=> SELECT execq('INSERT INTO a SELECT x + 2 FROM a RETURNING *', 1);
INFO: EXECQ: 2      -- 0 + 2, then execution was stopped by count
 execq
-------
(1 row)

=> SELECT execq('SELECT * FROM a', 10);
INFO: EXECQ: 0
INFO: EXECQ: 1
INFO: EXECQ: 2

 execq
-------
     3              -- 10 is the max value only, 3 is the real number of rows
(1 row)

=> SELECT execq('INSERT INTO a SELECT x + 10 FROM a', 1);
 execq
-------
     3              -- all rows processed; count does not stop it,
                    -- because nothing is returned
(1 row)

=> SELECT * FROM a;
 x
----
 0
 1
 2
10
11
12
(6 rows)

=> DELETE FROM a;
DELETE 6
=> INSERT INTO a VALUES (execq('SELECT * FROM a', 0) + 1);
INSERT 0 1
=> SELECT * FROM a;
 x
---
 1                  -- 0 (no rows in a) + 1
(1 row)

=> INSERT INTO a VALUES (execq('SELECT * FROM a', 0) + 1);
INFO: EXECQ: 1
INSERT 0 1
=> SELECT * FROM a;
 x
---
 1
 2                  -- 1 (there was one row in a) + 1
(2 rows)

-- This demonstrates the data changes visibility rule.
-- execq is called twice and sees different numbers of rows each time:

=> INSERT INTO a SELECT execq('SELECT * FROM a', 0) * x FROM a;
INFO: EXECQ: 1      -- results from first execq
INFO: EXECQ: 2
INFO: EXECQ: 1      -- results from second execq
INFO: EXECQ: 2
INFO: EXECQ: 2
INSERT 0 2
=> SELECT * FROM a;
 x
---
 1
 2
 2                  -- 2 rows * 1 (x in first row)
 6                  -- 3 rows (2 + 1 just inserted) * 2 (x in second row)
(4 rows)
```
