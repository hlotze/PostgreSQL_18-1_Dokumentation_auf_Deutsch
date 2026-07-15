# 59. Eine Table-Sampling-Methode schreiben

Die PostgreSQL-Implementierung der `TABLESAMPLE`-Klausel unterstützt zusätzlich zu den vom SQL-Standard geforderten Methoden `BERNOULLI` und `SYSTEM` auch benutzerdefinierte Table-Sampling-Methoden. Die Sampling-Methode bestimmt, welche Zeilen einer Tabelle ausgewählt werden, wenn die `TABLESAMPLE`-Klausel verwendet wird.

Auf SQL-Ebene wird eine Table-Sampling-Methode durch eine einzelne SQL-Funktion dargestellt, die typischerweise in C implementiert ist und folgende Signatur hat:

```sql
method_name(internal) RETURNS tsm_handler
```

Der Name der Funktion ist derselbe Methodenname, der in der `TABLESAMPLE`-Klausel erscheint. Das Argument `internal` ist ein Dummy-Argument (immer mit dem Wert null), das nur dazu dient, den direkten Aufruf dieser Funktion aus einem SQL-Befehl zu verhindern. Das Ergebnis der Funktion muss eine mit `palloc` allozierte Struktur vom Typ `TsmRoutine` sein, die Zeiger auf Support-Funktionen für die Sampling-Methode enthält. Diese Support-Funktionen sind einfache C-Funktionen und auf SQL-Ebene weder sichtbar noch aufrufbar. Die Support-Funktionen werden in [Abschnitt 59.1](59_Eine_Table_Sampling_Methode_schreiben.md#591-supportfunktionen-für-samplingmethoden) beschrieben.

Neben Funktionszeigern muss die `TsmRoutine`-Struktur diese zusätzlichen Felder bereitstellen:

- `List *parameterTypes`
  - Dies ist eine OID-Liste mit den OIDs der Datentypen der Parameter, die von der `TABLESAMPLE`-Klausel akzeptiert werden, wenn diese Sampling-Methode verwendet wird. Bei den eingebauten Methoden enthält diese Liste zum Beispiel ein einzelnes Element mit dem Wert `FLOAT4OID`, das den Sampling-Prozentsatz darstellt. Benutzerdefinierte Sampling-Methoden können mehr oder andere Parameter haben.

- `bool repeatable_across_queries`
  - Wenn `true`, kann die Sampling-Methode über aufeinanderfolgende Abfragen hinweg identische Stichproben liefern, sofern jedes Mal dieselben Parameter und derselbe `REPEATABLE`-Seed-Wert angegeben werden und sich der Tabelleninhalt nicht geändert hat. Wenn dieses Feld `false` ist, wird die `REPEATABLE`-Klausel für diese Sampling-Methode nicht akzeptiert.

- `bool repeatable_across_scans`
  - Wenn `true`, kann die Sampling-Methode innerhalb derselben Abfrage über aufeinanderfolgende Scans hinweg identische Stichproben liefern, vorausgesetzt Parameter, Seed-Wert und Snapshot bleiben unverändert. Wenn dieses Feld `false` ist, wählt der Planner keine Pläne aus, die erfordern würden, die beprobte Tabelle mehr als einmal zu scannen, da dies zu inkonsistenten Abfrageergebnissen führen könnte.

Der Strukturtyp `TsmRoutine` ist in `src/include/access/tsmapi.h` deklariert; dort finden Sie weitere Details.

Die in der Standarddistribution enthaltenen Table-Sampling-Methoden sind gute Referenzen, wenn Sie eine eigene Methode schreiben möchten. Die eingebauten Sampling-Methoden befinden sich im Unterverzeichnis `src/backend/access/tablesample` des Quellbaums; zusätzliche Methoden finden Sie im Unterverzeichnis `contrib`.

## 59.1. Support-Funktionen für Sampling-Methoden

Die TSM-Handlerfunktion gibt eine mit `palloc` allozierte `TsmRoutine`-Struktur zurück, die Zeiger auf die unten beschriebenen Support-Funktionen enthält. Die meisten Funktionen sind erforderlich, einige sind jedoch optional; die entsprechenden Zeiger können dann `NULL` sein.

```c
void
SampleScanGetSampleSize(PlannerInfo *root,
                        RelOptInfo *baserel,
                        List *paramexprs,
                        BlockNumber *pages,
                        double *tuples);
```

Diese Funktion wird während der Planung aufgerufen. Sie muss die Anzahl der Relationsseiten schätzen, die während eines Sample-Scans gelesen werden, sowie die Anzahl der Tupel, die der Scan auswählen wird. Diese Werte könnten zum Beispiel bestimmt werden, indem die Sampling-Fraktion geschätzt und anschließend mit `baserel->pages` und `baserel->tuples` multipliziert wird; die Ergebnisse müssen dabei auf ganzzahlige Werte gerundet werden. Die Liste `paramexprs` enthält die Ausdrücke, die als Parameter der `TABLESAMPLE`-Klausel dienen. Es wird empfohlen, mit `estimate_expression_value()` zu versuchen, diese Ausdrücke zu Konstanten zu reduzieren, falls ihre Werte für die Schätzung benötigt werden. Die Funktion muss aber auch dann Größenschätzungen liefern, wenn die Ausdrücke nicht reduziert werden können, und sollte nicht fehlschlagen, selbst wenn die Werte ungültig erscheinen. Denken Sie daran, dass es nur Schätzungen der Laufzeitwerte sind. Die Parameter `pages` und `tuples` sind Ausgaben.

```c
void
InitSampleScan(SampleScanState *node,
               int eflags);
```

Initialisiert die Ausführung eines `SampleScan`-Planknotens. Diese Funktion wird während des Executor-Starts aufgerufen. Sie sollte alle Initialisierungen durchführen, die vor Beginn der Verarbeitung nötig sind. Der `SampleScanState`-Knoten wurde bereits erzeugt, aber sein Feld `tsm_state` ist `NULL`. `InitSampleScan` kann beliebige interne Zustandsdaten mit `palloc` allozieren und einen Zeiger darauf in `node->tsm_state` speichern. Informationen über die zu scannende Tabelle sind über andere Felder des `SampleScanState`-Knotens zugänglich. Beachten Sie aber, dass der Scan-Deskriptor `node->ss.ss_currentScanDesc` noch nicht eingerichtet ist. `eflags` enthält Flag-Bits, die den Betriebsmodus des Executors für diesen Planknoten beschreiben.

Wenn `(eflags & EXEC_FLAG_EXPLAIN_ONLY)` wahr ist, wird der Scan nicht tatsächlich ausgeführt. Die Funktion sollte dann nur das Minimum tun, das erforderlich ist, damit der Knotenzustand für `EXPLAIN` und `EndSampleScan` gültig ist.

Diese Funktion kann ausgelassen werden, indem der Zeiger auf `NULL` gesetzt wird. In diesem Fall muss `BeginSampleScan` alle für die Sampling-Methode nötigen Initialisierungen durchführen.

```c
void
BeginSampleScan(SampleScanState *node,
                Datum *params,
                int nparams,
                uint32 seed);
```

Beginnt die Ausführung eines Sampling-Scans. Diese Funktion wird unmittelbar vor dem ersten Versuch aufgerufen, ein Tupel zu holen, und kann erneut aufgerufen werden, wenn der Scan neu gestartet werden muss. Informationen über die zu scannende Tabelle sind über Felder des `SampleScanState`-Knotens zugänglich. Beachten Sie aber, dass der Scan-Deskriptor `node->ss.ss_currentScanDesc` noch nicht eingerichtet ist. Das Array `params` mit der Länge `nparams` enthält die Werte der Parameter, die in der `TABLESAMPLE`-Klausel angegeben wurden. Diese haben Anzahl und Typen gemäß der Liste `parameterTypes` der Sampling-Methode und wurden darauf geprüft, nicht null zu sein. `seed` enthält einen Seed für alle innerhalb der Sampling-Methode erzeugten Zufallszahlen; er ist entweder ein aus dem `REPEATABLE`-Wert abgeleiteter Hash oder, wenn kein solcher Wert angegeben wurde, das Ergebnis von `random()`.

Diese Funktion kann die Felder `node->use_bulkread` und `node->use_pagemode` anpassen. Wenn `node->use_bulkread` wahr ist, was der Standard ist, verwendet der Scan eine Buffer-Zugriffsstrategie, die das Wiederverwenden von Buffern nach ihrer Nutzung begünstigt. Es kann sinnvoll sein, diesen Wert auf `false` zu setzen, wenn der Scan nur einen kleinen Anteil der Tabellenseiten besucht. Wenn `node->use_pagemode` wahr ist, was ebenfalls der Standard ist, führt der Scan die Sichtbarkeitsprüfung für alle Tupel auf jeder besuchten Seite in einem Durchgang aus. Es kann sinnvoll sein, diesen Wert auf `false` zu setzen, wenn der Scan nur einen kleinen Anteil der Tupel auf jeder besuchten Seite auswählt. Dadurch werden weniger Tupel-Sichtbarkeitsprüfungen durchgeführt, allerdings ist jede einzelne Prüfung teurer, weil sie mehr Sperren erfordert.

Wenn die Sampling-Methode als `repeatable_across_scans` markiert ist, muss sie bei einem Rescan dieselbe Menge von Tupeln auswählen können wie ursprünglich. Das heißt, ein frischer Aufruf von `BeginSampleScan` muss zur Auswahl derselben Tupel führen wie zuvor, sofern sich `TABLESAMPLE`-Parameter und Seed nicht ändern.

```c
BlockNumber
NextSampleBlock(SampleScanState *node, BlockNumber nblocks);
```

Gibt die Blocknummer der nächsten zu scannenden Seite zurück oder `InvalidBlockNumber`, wenn keine weiteren Seiten zu scannen sind.

Diese Funktion kann ausgelassen werden, indem der Zeiger auf `NULL` gesetzt wird. In diesem Fall führt der Kerncode einen sequenziellen Scan der gesamten Relation aus. Ein solcher Scan kann synchronisiertes Scannen verwenden, sodass die Sampling-Methode nicht annehmen darf, dass die Relationsseiten bei jedem Scan in derselben Reihenfolge besucht werden.

```c
OffsetNumber
NextSampleTuple(SampleScanState *node,
                BlockNumber blockno,
                OffsetNumber maxoffset);
```

Gibt die Offsetnummer des nächsten auf der angegebenen Seite zu beprobenden Tupels zurück oder `InvalidOffsetNumber`, wenn keine weiteren Tupel zu beproben sind. `maxoffset` ist die größte auf der Seite verwendete Offsetnummer.

> `NextSampleTuple` wird nicht ausdrücklich mitgeteilt, welche Offsetnummern im Bereich von 1 bis `maxoffset` tatsächlich gültige Tupel enthalten. Normalerweise ist das kein Problem, da der Kerncode Anforderungen ignoriert, fehlende oder unsichtbare Tupel zu beproben; dadurch sollte keine Verzerrung der Stichprobe entstehen. Falls nötig, kann die Funktion jedoch `node->donetuples` verwenden, um zu untersuchen, wie viele der von ihr zurückgegebenen Tupel gültig und sichtbar waren.

> `NextSampleTuple` darf nicht annehmen, dass `blockno` dieselbe Seitennummer ist, die vom jüngsten Aufruf von `NextSampleBlock` zurückgegeben wurde. Sie wurde von einem früheren Aufruf von `NextSampleBlock` zurückgegeben, aber der Kerncode darf `NextSampleBlock` im Voraus aufrufen, bevor Seiten tatsächlich gescannt werden, um Prefetching zu unterstützen. Es ist zulässig anzunehmen, dass, sobald das Sampling einer bestimmten Seite begonnen hat, aufeinanderfolgende Aufrufe von `NextSampleTuple` dieselbe Seite betreffen, bis `InvalidOffsetNumber` zurückgegeben wird.

```c
void
EndSampleScan(SampleScanState *node);
```

Beendet den Scan und gibt Ressourcen frei. Es ist normalerweise nicht wichtig, mit `palloc` allozierten Speicher freizugeben, aber alle extern sichtbaren Ressourcen sollten bereinigt werden. Diese Funktion kann im üblichen Fall, dass keine solchen Ressourcen existieren, ausgelassen werden, indem der Zeiger auf `NULL` gesetzt wird.
