# 58. Einen Foreign Data Wrapper schreiben

Alle Operationen auf einer fremden Tabelle werden durch ihren Foreign Data Wrapper abgewickelt. Dieser besteht aus einer Menge von Funktionen, die der Kernserver aufruft. Der Foreign Data Wrapper ist dafür verantwortlich, Daten aus der entfernten Datenquelle zu holen und sie an den PostgreSQL-Executor zurückzugeben. Wenn das Aktualisieren fremder Tabellen unterstützt werden soll, muss der Wrapper auch das behandeln. Dieses Kapitel skizziert, wie ein neuer Foreign Data Wrapper geschrieben wird.

Die in der Standarddistribution enthaltenen Foreign Data Wrapper sind gute Referenzen, wenn Sie einen eigenen schreiben möchten. Sehen Sie im Unterverzeichnis `contrib` des Quellbaums nach. Auch die Referenzseite zu `CREATE FOREIGN DATA WRAPPER` enthält nützliche Details.

> Der SQL-Standard legt eine Schnittstelle zum Schreiben von Foreign Data Wrappern fest. PostgreSQL implementiert diese API jedoch nicht, weil der Aufwand, sie in PostgreSQL einzupassen, groß wäre und die Standard-API ohnehin keine breite Verbreitung gefunden hat.

## 58.1. Foreign-Data-Wrapper-Funktionen

Der FDW-Autor muss eine Handlerfunktion implementieren und kann optional eine Validatorfunktion bereitstellen. Beide Funktionen müssen in einer kompilierten Sprache wie C geschrieben sein und die Version-1-Schnittstelle verwenden. Einzelheiten zu C-Aufrufkonventionen und dynamischem Laden finden Sie in [Abschnitt 36.10](36_SQL_erweitern.md#3610-csprachfunktionen).

Die Handlerfunktion gibt lediglich eine Struktur von Funktionszeigern auf Callback-Funktionen zurück, die vom Planner, Executor und verschiedenen Wartungsbefehlen aufgerufen werden. Der größte Teil der Arbeit beim Schreiben eines FDW besteht darin, diese Callback-Funktionen zu implementieren. Die Handlerfunktion muss in PostgreSQL als Funktion ohne Argumente registriert werden, die den speziellen Pseudotyp `fdw_handler` zurückgibt. Die Callback-Funktionen sind einfache C-Funktionen und auf SQL-Ebene weder sichtbar noch aufrufbar. Sie werden in [Abschnitt 58.2](58_Einen_Foreign_Data_Wrapper_schreiben.md#582-callbackroutinen-für-foreign-data-wrapper) beschrieben.

Die Validatorfunktion ist dafür verantwortlich, Optionen zu prüfen, die in `CREATE`- und `ALTER`-Befehlen für ihren Foreign Data Wrapper sowie für fremde Server, Benutzerzuordnungen und fremde Tabellen angegeben werden, die den Wrapper verwenden. Die Validatorfunktion muss mit zwei Argumenten registriert werden: einem Text-Array mit den zu prüfenden Optionen und einer OID, die den Objekttyp angibt, zu dem die Optionen gehören. Letztere entspricht der OID des Systemkatalogs, in dem das Objekt gespeichert würde, also einem der folgenden Werte:

- `AttributeRelationId`
- `ForeignDataWrapperRelationId`
- `ForeignServerRelationId`
- `ForeignTableRelationId`
- `UserMappingRelationId`

Wenn keine Validatorfunktion angegeben wird, werden Optionen beim Erzeugen oder Ändern von Objekten nicht geprüft.

## 58.2. Callback-Routinen für Foreign Data Wrapper

Die FDW-Handlerfunktion gibt eine mit `palloc` allozierte `FdwRoutine`-Struktur zurück, die Zeiger auf die unten beschriebenen Callback-Funktionen enthält. Die Scan-bezogenen Funktionen sind erforderlich, die übrigen sind optional.

Der Strukturtyp `FdwRoutine` ist in `src/include/foreign/fdwapi.h` deklariert; dort finden Sie weitere Details.

### 58.2.1. FDW-Routinen zum Scannen fremder Tabellen

```c
void
GetForeignRelSize(PlannerInfo *root,
                  RelOptInfo *baserel,
                  Oid foreigntableid);
```

Ermittelt Größenschätzungen für eine fremde Tabelle. Diese Funktion wird zu Beginn der Planung einer Abfrage aufgerufen, die eine fremde Tabelle scannt. `root` enthält die globalen Planner-Informationen zur Abfrage, `baserel` die Planner-Informationen zu dieser Tabelle und `foreigntableid` die `pg_class`-OID der fremden Tabelle. `foreigntableid` könnte auch aus den Planner-Datenstrukturen gewonnen werden, wird aber ausdrücklich übergeben, um Aufwand zu sparen.

Diese Funktion sollte `baserel->rows` auf die erwartete Anzahl von Zeilen setzen, die der Tabellenscan nach Berücksichtigung der Filterung durch Restriktions-Qualifikationen zurückgibt. Der Anfangswert von `baserel->rows` ist nur eine konstante Standardschätzung und sollte nach Möglichkeit ersetzt werden. Die Funktion kann außerdem `baserel->width` aktualisieren, wenn sie eine bessere Schätzung der durchschnittlichen Breite einer Ergebniszeile berechnen kann. Der Anfangswert basiert auf Spaltendatentypen und auf durchschnittlichen Spaltenbreiten, die beim letzten `ANALYZE` gemessen wurden. Ebenso kann diese Funktion `baserel->tuples` aktualisieren, wenn sie eine bessere Schätzung der Gesamtzeilenzahl der fremden Tabelle berechnen kann. Der Anfangswert stammt aus `pg_class.reltuples`, das die beim letzten `ANALYZE` gesehene Gesamtzeilenzahl darstellt; er ist `-1`, wenn auf dieser fremden Tabelle noch kein `ANALYZE` ausgeführt wurde.

Weitere Informationen finden Sie in [Abschnitt 58.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#584-abfrageplanung-für-foreign-data-wrapper).

```c
void
GetForeignPaths(PlannerInfo *root,
                RelOptInfo *baserel,
                Oid foreigntableid);
```

Erzeugt mögliche Zugriffspfade für einen Scan auf einer fremden Tabelle. Diese Funktion wird während der Abfrageplanung aufgerufen. Die Parameter entsprechen denen von `GetForeignRelSize`, das bereits aufgerufen wurde.

Diese Funktion muss mindestens einen Zugriffspfad (`ForeignPath`-Knoten) für einen Scan auf der fremden Tabelle erzeugen und `add_path` aufrufen, um jeden solchen Pfad zu `baserel->pathlist` hinzuzufügen. Es wird empfohlen, `create_foreignscan_path` zu verwenden, um die `ForeignPath`-Knoten zu bauen. Die Funktion kann mehrere Zugriffspfade erzeugen, etwa einen Pfad, der gültige `pathkeys` enthält, um ein vorsortiertes Ergebnis darzustellen. Jeder Zugriffspfad muss Kostenschätzungen enthalten und kann beliebige FDW-private Informationen enthalten, die nötig sind, um die beabsichtigte konkrete Scan-Methode zu identifizieren.

Weitere Informationen finden Sie in [Abschnitt 58.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#584-abfrageplanung-für-foreign-data-wrapper).

```c
ForeignScan *
GetForeignPlan(PlannerInfo *root,
               RelOptInfo *baserel,
               Oid foreigntableid,
               ForeignPath *best_path,
               List *tlist,
               List *scan_clauses,
               Plan *outer_plan);
```

Erzeugt aus dem ausgewählten fremden Zugriffspfad einen `ForeignScan`-Planknoten. Diese Funktion wird am Ende der Abfrageplanung aufgerufen. Die Parameter entsprechen denen von `GetForeignRelSize`, ergänzt um den ausgewählten `ForeignPath`, der zuvor von `GetForeignPaths`, `GetForeignJoinPaths` oder `GetForeignUpperPaths` erzeugt wurde, die vom Planknoten auszugebende Zielliste, die vom Planknoten durchzusetzenden Restriktionsklauseln und den äußeren Unterplan des `ForeignScan`, der für von `RecheckForeignScan` durchgeführte Nachprüfungen verwendet wird. Wenn der Pfad einen Join und keine Basisrelation betrifft, ist `foreigntableid` `InvalidOid`.

Diese Funktion muss einen `ForeignScan`-Planknoten erzeugen und zurückgeben. Es wird empfohlen, `make_foreignscan` zu verwenden, um den `ForeignScan`-Knoten zu bauen.

Weitere Informationen finden Sie in [Abschnitt 58.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#584-abfrageplanung-für-foreign-data-wrapper).

```c
void
BeginForeignScan(ForeignScanState *node,
                 int eflags);
```

Beginnt die Ausführung eines fremden Scans. Diese Funktion wird während des Executor-Starts aufgerufen. Sie sollte alle Initialisierungen durchführen, die nötig sind, bevor der Scan starten kann, aber den eigentlichen Scan noch nicht ausführen; das sollte erst beim ersten Aufruf von `IterateForeignScan` geschehen. Der `ForeignScanState`-Knoten wurde bereits erzeugt, aber sein Feld `fdw_state` ist noch `NULL`. Informationen über die zu scannende Tabelle sind über den `ForeignScanState`-Knoten zugänglich, insbesondere über den zugrunde liegenden `ForeignScan`-Planknoten, der alle von `GetForeignPlan` bereitgestellten FDW-privaten Informationen enthält. `eflags` enthält Flag-Bits, die den Betriebsmodus des Executors für diesen Planknoten beschreiben.

Wenn `(eflags & EXEC_FLAG_EXPLAIN_ONLY)` wahr ist, sollte diese Funktion keine extern sichtbaren Aktionen ausführen. Sie sollte nur das Minimum tun, das erforderlich ist, damit der Knotenzustand für `ExplainForeignScan` und `EndForeignScan` gültig ist.

```c
TupleTableSlot *
IterateForeignScan(ForeignScanState *node);
```

Holt eine Zeile aus der fremden Quelle und gibt sie in einem Tuple-Table-Slot zurück; dafür sollte der `ScanTupleSlot` des Knotens verwendet werden. Gibt `NULL` zurück, wenn keine weiteren Zeilen verfügbar sind. Die Tuple-Table-Slot-Infrastruktur erlaubt die Rückgabe eines physischen oder virtuellen Tupels; in den meisten Fällen ist letzteres aus Performance-Sicht vorzuziehen. Beachten Sie, dass diese Funktion in einem kurzlebigen Speicher-Kontext aufgerufen wird, der zwischen Aufrufen zurückgesetzt wird. Erzeugen Sie in `BeginForeignScan` einen Speicher-Kontext, wenn Sie langlebigere Speicherung benötigen, oder verwenden Sie `es_query_cxt` des `EState` des Knotens.

Die zurückgegebenen Zeilen müssen der Zielliste `fdw_scan_tlist` entsprechen, falls eine angegeben wurde; andernfalls müssen sie dem Zeilentyp der gescannten fremden Tabelle entsprechen. Wenn Sie das Holen nicht benötigter Spalten optimieren und weglassen, sollten Sie an diesen Spaltenpositionen Nullwerte einsetzen oder eine Liste `fdw_scan_tlist` erzeugen, in der diese Spalten ausgelassen sind.

Beachten Sie, dass es dem PostgreSQL-Executor gleichgültig ist, ob die zurückgegebenen Zeilen auf der fremden Tabelle definierte Constraints verletzen. Dem Planner ist das jedoch nicht gleichgültig: Er kann Abfragen falsch optimieren, wenn in der fremden Tabelle Zeilen sichtbar sind, die ein deklariertes Constraint nicht erfüllen. Wenn ein Constraint verletzt wird, obwohl der Benutzer erklärt hat, dass es gelten soll, kann es angemessen sein, einen Fehler auszulösen, so wie es auch bei einem Datentypkonflikt nötig wäre.

```c
void
ReScanForeignScan(ForeignScanState *node);
```

Startet den Scan wieder von vorn. Beachten Sie, dass sich Parameter, von denen der Scan abhängt, geändert haben können; der neue Scan muss daher nicht genau dieselben Zeilen zurückgeben.

```c
void
EndForeignScan(ForeignScanState *node);
```

Beendet den Scan und gibt Ressourcen frei. Es ist normalerweise nicht wichtig, mit `palloc` allozierten Speicher freizugeben, aber zum Beispiel offene Dateien und Verbindungen zu entfernten Servern sollten bereinigt werden.

### 58.2.2. FDW-Routinen zum Scannen fremder Joins

Wenn ein FDW die entfernte Ausführung fremder Joins unterstützt, statt die Daten beider Tabellen zu holen und den Join lokal auszuführen, sollte er diese Callback-Funktion bereitstellen:

```c
void
GetForeignJoinPaths(PlannerInfo *root,
                    RelOptInfo *joinrel,
                    RelOptInfo *outerrel,
                    RelOptInfo *innerrel,
                    JoinType jointype,
                    JoinPathExtraData *extra);
```

Erzeugt mögliche Zugriffspfade für einen Join von zwei oder mehr fremden Tabellen, die alle zum selben fremden Server gehören. Diese optionale Funktion wird während der Abfrageplanung aufgerufen. Wie `GetForeignPaths` sollte diese Funktion `ForeignPath`-Pfade für die übergebene Join-Relation erzeugen, dafür `create_foreign_join_path` verwenden und `add_path` aufrufen, um diese Pfade zu den für den Join betrachteten Pfaden hinzuzufügen. Anders als bei `GetForeignPaths` ist es jedoch nicht erforderlich, dass diese Funktion mindestens einen Pfad erzeugt, da Pfade mit lokalem Join immer möglich sind.

Beachten Sie, dass diese Funktion für dieselbe Join-Relation wiederholt mit verschiedenen Kombinationen von inneren und äußeren Relationen aufgerufen wird. Es liegt in der Verantwortung des FDW, doppelte Arbeit zu minimieren.

Beachten Sie außerdem, dass die Menge der auf den Join anzuwendenden Join-Klauseln, die als `extra->restrictlist` übergeben wird, je nach Kombination von inneren und äußeren Relationen variiert. Ein für `joinrel` erzeugter `ForeignPath`-Pfad muss die Menge der von ihm verwendeten Join-Klauseln enthalten. Diese wird vom Planner verwendet, um den `ForeignPath`-Pfad in einen Plan umzuwandeln, falls er als bester Pfad für `joinrel` ausgewählt wird.

Wenn ein `ForeignPath`-Pfad für den Join ausgewählt wird, repräsentiert er den gesamten Join-Prozess. Für die beteiligten Tabellen und untergeordneten Joins erzeugte Pfade werden dann nicht verwendet. Die weitere Verarbeitung des Join-Pfads verläuft weitgehend wie bei einem Pfad, der eine einzelne fremde Tabelle scannt. Ein Unterschied besteht darin, dass `scanrelid` des resultierenden `ForeignScan`-Planknotens auf null gesetzt werden sollte, weil es keine einzelne Relation gibt, die er repräsentiert. Stattdessen repräsentiert das Feld `fs_relids` des `ForeignScan`-Knotens die Menge der gejointen Relationen. Dieses Feld wird automatisch vom Kerncode des Planners eingerichtet und muss vom FDW nicht gefüllt werden. Ein weiterer Unterschied besteht darin, dass die Spaltenliste für einen entfernten Join nicht aus den Systemkatalogen ermittelt werden kann. Der FDW muss daher `fdw_scan_tlist` mit einer geeigneten Liste von `TargetEntry`-Knoten füllen, die die Menge der Spalten repräsentiert, die er zur Laufzeit in den zurückgegebenen Tupeln liefert.

> Seit PostgreSQL 16 enthält `fs_relids` die Rangtabellenindizes äußerer Joins, wenn solche an diesem Join beteiligt waren. Das neue Feld `fs_base_relids` enthält nur Basisrelationsindizes und bildet daher die frühere Semantik von `fs_relids` nach.

Weitere Informationen finden Sie in [Abschnitt 58.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#584-abfrageplanung-für-foreign-data-wrapper).

### 58.2.3. FDW-Routinen zur Planung der Verarbeitung nach Scan/Join

Wenn ein FDW entfernte Verarbeitung nach Scan oder Join unterstützt, etwa entfernte Aggregation, sollte er diese Callback-Funktion bereitstellen:

```c
void
GetForeignUpperPaths(PlannerInfo *root,
                     UpperRelationKind stage,
                     RelOptInfo *input_rel,
                     RelOptInfo *output_rel,
                     void *extra);
```

Erzeugt mögliche Zugriffspfade für die Verarbeitung oberer Relationen. So nennt der Planner alle Verarbeitungsschritte nach Scan oder Join, etwa Aggregation, Window-Funktionen, Sortierung und Tabellenaktualisierungen. Diese optionale Funktion wird während der Abfrageplanung aufgerufen. Derzeit wird sie nur aufgerufen, wenn alle an der Abfrage beteiligten Basisrelationen zu demselben FDW gehören. Die Funktion sollte `ForeignPath`-Pfade für alle nach Scan oder Join liegenden Verarbeitungsschritte erzeugen, die der FDW entfernt ausführen kann; verwenden Sie dazu `create_foreign_upper_path` und rufen Sie `add_path` auf, um diese Pfade zur angegebenen oberen Relation hinzuzufügen. Wie bei `GetForeignJoinPaths` ist es nicht erforderlich, dass diese Funktion irgendwelche Pfade erzeugt, da Pfade mit lokaler Verarbeitung immer möglich sind.

Der Parameter `stage` identifiziert, welcher Schritt nach Scan oder Join gerade betrachtet wird. `output_rel` ist die obere Relation, die Pfade erhalten soll, welche die Berechnung dieses Schritts repräsentieren, und `input_rel` ist die Relation, die die Eingabe für diesen Schritt repräsentiert. Der Parameter `extra` liefert zusätzliche Details. Derzeit ist er nur für `UPPERREL_PARTIAL_GROUP_AGG` oder `UPPERREL_GROUP_AGG` gesetzt; in diesem Fall zeigt er auf eine `GroupPathExtraData`-Struktur. Für `UPPERREL_FINAL` zeigt er auf eine `FinalPathExtraData`-Struktur. Beachten Sie, dass zu `output_rel` hinzugefügte `ForeignPath`-Pfade typischerweise keine direkte Abhängigkeit von Pfaden von `input_rel` haben, da ihre Verarbeitung extern erfolgen soll. Es kann jedoch nützlich sein, zuvor für den vorherigen Verarbeitungsschritt erzeugte Pfade zu untersuchen, um redundante Planungsarbeit zu vermeiden.

Weitere Informationen finden Sie in [Abschnitt 58.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#584-abfrageplanung-für-foreign-data-wrapper).

### 58.2.4. FDW-Routinen zum Aktualisieren fremder Tabellen

Wenn ein FDW beschreibbare fremde Tabellen unterstützt, sollte er je nach Bedarf und Fähigkeiten einige oder alle der folgenden Callback-Funktionen bereitstellen.

```c
void
AddForeignUpdateTargets(PlannerInfo *root,
                        Index rtindex,
                        RangeTblEntry *target_rte,
                        Relation target_relation);
```

`UPDATE`- und `DELETE`-Operationen werden auf Zeilen ausgeführt, die zuvor von den Tabellenscan-Funktionen geholt wurden. Der FDW kann zusätzliche Informationen benötigen, etwa eine Zeilen-ID oder die Werte von Primärschlüsselspalten, um sicherzustellen, dass er genau die zu aktualisierende oder zu löschende Zeile identifizieren kann. Zur Unterstützung dessen kann diese Funktion zusätzliche verborgene oder »Junk«-Zielspalten zur Liste der Spalten hinzufügen, die während eines `UPDATE` oder `DELETE` aus der fremden Tabelle geholt werden sollen.

Konstruieren Sie dazu eine `Var`, die einen zusätzlich benötigten Wert repräsentiert, und übergeben Sie sie zusammen mit einem Namen für die Junk-Spalte an `add_row_identity_var`. Sie können dies mehrmals tun, wenn mehrere Spalten benötigt werden. Für jede unterschiedliche benötigte `Var` muss ein eigener Junk-Spaltenname gewählt werden, mit der Ausnahme, dass `Var`s, die bis auf das Feld `varno` identisch sind, denselben Spaltennamen gemeinsam verwenden können und sollten. Das Kernsystem verwendet die Junk-Spaltennamen `tableoid` für die `tableoid`-Spalte einer Tabelle, `ctid` oder `ctidN` für `ctid`, `wholerow` für eine Whole-Row-`Var` mit `vartype = RECORD` und `wholerowN` für eine Whole-Row-`Var`, deren `vartype` dem deklarierten Zeilentyp der Tabelle entspricht. Verwenden Sie diese Namen wieder, wenn möglich; der Planner kombiniert doppelte Anforderungen für identische Junk-Spalten. Wenn Sie eine andere Art von Junk-Spalte benötigen, ist es ratsam, einen Namen mit dem Namen Ihrer Erweiterung als Präfix zu wählen, um Konflikte mit anderen FDWs zu vermeiden.

Wenn der Zeiger `AddForeignUpdateTargets` auf `NULL` gesetzt ist, werden keine zusätzlichen Zielausdrücke hinzugefügt. Dadurch wird es unmöglich, `DELETE`-Operationen zu implementieren; `UPDATE` kann jedoch weiterhin möglich sein, wenn der FDW sich auf einen unveränderlichen Primärschlüssel zur Identifikation von Zeilen stützt.

```c
List *
PlanForeignModify(PlannerInfo *root,
                  ModifyTable *plan,
                  Index resultRelation,
                  int subplan_index);
```

Führt alle zusätzlichen Planungsschritte aus, die für ein Insert, Update oder Delete auf einer fremden Tabelle nötig sind. Diese Funktion erzeugt FDW-private Informationen, die an den `ModifyTable`-Planknoten angehängt werden, der die Aktualisierungsaktion ausführt. Diese privaten Informationen müssen die Form einer `List` haben und werden während der Ausführungsphase an `BeginForeignModify` geliefert.

`root` enthält die globalen Planner-Informationen zur Abfrage. `plan` ist der `ModifyTable`-Planknoten, der bis auf das Feld `fdwPrivLists` vollständig ist. `resultRelation` identifiziert die fremde Zieltabelle durch ihren Range-Table-Index. `subplan_index` identifiziert, welches Ziel des `ModifyTable`-Planknotens dies ist, von null an gezählt; verwenden Sie dies, wenn Sie in pro-Zielrelations-Unterstrukturen des Planknotens indizieren möchten.

Weitere Informationen finden Sie in [Abschnitt 58.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#584-abfrageplanung-für-foreign-data-wrapper).

Wenn der Zeiger `PlanForeignModify` auf `NULL` gesetzt ist, werden zur Planungszeit keine zusätzlichen Aktionen ausgeführt, und die an `BeginForeignModify` gelieferte Liste `fdw_private` ist `NIL`.

```c
void
BeginForeignModify(ModifyTableState *mtstate,
                   ResultRelInfo *rinfo,
                   List *fdw_private,
                   int subplan_index,
                   int eflags);
```

Beginnt die Ausführung einer Modifikationsoperation auf einer fremden Tabelle. Diese Routine wird während des Executor-Starts aufgerufen. Sie sollte alle Initialisierungen durchführen, die vor den tatsächlichen Tabellenänderungen nötig sind. Anschließend werden `ExecForeignInsert`/`ExecForeignBatchInsert`, `ExecForeignUpdate` oder `ExecForeignDelete` für einzufügende, zu aktualisierende oder zu löschende Tupel aufgerufen.

`mtstate` ist der Gesamtzustand des ausgeführten `ModifyTable`-Planknotens; globale Daten über Plan und Ausführungszustand sind über diese Struktur verfügbar. `rinfo` ist die `ResultRelInfo`-Struktur, die die fremde Zieltabelle beschreibt. Das Feld `ri_FdwState` von `ResultRelInfo` steht dem FDW zur Verfügung, um privaten Zustand für diese Operation zu speichern. `fdw_private` enthält die von `PlanForeignModify` erzeugten privaten Daten, falls vorhanden. `subplan_index` identifiziert, welches Ziel des `ModifyTable`-Planknotens dies ist. `eflags` enthält Flag-Bits, die den Betriebsmodus des Executors für diesen Planknoten beschreiben.

Wenn `(eflags & EXEC_FLAG_EXPLAIN_ONLY)` wahr ist, sollte diese Funktion keine extern sichtbaren Aktionen ausführen. Sie sollte nur das Minimum tun, das erforderlich ist, damit der Knotenzustand für `ExplainForeignModify` und `EndForeignModify` gültig ist.

Wenn der Zeiger `BeginForeignModify` auf `NULL` gesetzt ist, wird während des Executor-Starts keine Aktion ausgeführt.

```c
TupleTableSlot *
ExecForeignInsert(EState *estate,
                  ResultRelInfo *rinfo,
                  TupleTableSlot *slot,
                  TupleTableSlot *planSlot);
```

Fügt ein Tupel in die fremde Tabelle ein. `estate` ist der globale Ausführungszustand der Abfrage. `rinfo` ist die `ResultRelInfo`-Struktur, die die fremde Zieltabelle beschreibt. `slot` enthält das einzufügende Tupel; es entspricht der Zeilentypdefinition der fremden Tabelle. `planSlot` enthält das Tupel, das vom Unterplan des `ModifyTable`-Planknotens erzeugt wurde; es unterscheidet sich von `slot` dadurch, dass es zusätzliche Junk-Spalten enthalten kann. Bei `INSERT`-Fällen ist `planSlot` typischerweise wenig interessant, wird aber der Vollständigkeit halber bereitgestellt.

Der Rückgabewert ist entweder ein Slot mit den Daten, die tatsächlich eingefügt wurden, was sich zum Beispiel infolge von Triggeraktionen von den übergebenen Daten unterscheiden kann, oder `NULL`, wenn keine Zeile tatsächlich eingefügt wurde, wiederum typischerweise infolge von Triggern. Der übergebene `slot` kann hierfür wiederverwendet werden.

Die Daten im zurückgegebenen Slot werden nur verwendet, wenn die `INSERT`-Anweisung eine `RETURNING`-Klausel hat oder eine View mit `WITH CHECK OPTION` beteiligt ist oder wenn die fremde Tabelle einen `AFTER ROW`-Trigger hat. Trigger benötigen alle Spalten, aber der FDW kann je nach Inhalt der `RETURNING`-Klausel oder der `WITH CHECK OPTION`-Constraints entscheiden, die Rückgabe einiger oder aller Spalten wegzuoptimieren. Unabhängig davon muss ein Slot zurückgegeben werden, um Erfolg anzuzeigen, sonst ist die von der Abfrage gemeldete Zeilenzahl falsch.

Wenn der Zeiger `ExecForeignInsert` auf `NULL` gesetzt ist, schlagen Versuche, in die fremde Tabelle einzufügen, mit einer Fehlermeldung fehl.

Beachten Sie, dass diese Funktion auch aufgerufen wird, wenn geroutete Tupel in eine Fremdtabellenpartition eingefügt werden oder `COPY FROM` auf einer fremden Tabelle ausgeführt wird. In diesem Fall wird sie anders aufgerufen als im `INSERT`-Fall. Siehe die unten beschriebenen Callback-Funktionen, die dem FDW erlauben, dies zu unterstützen.

```c
TupleTableSlot **
ExecForeignBatchInsert(EState *estate,
                       ResultRelInfo *rinfo,
                       TupleTableSlot **slots,
                       TupleTableSlot **planSlots,
                       int *numSlots);
```

Fügt mehrere Tupel gesammelt in die fremde Tabelle ein. Die Parameter entsprechen denen von `ExecForeignInsert`, außer dass `slots` und `planSlots` mehrere Tupel enthalten und `*numSlots` die Anzahl der Tupel in diesen Arrays angibt.

Der Rückgabewert ist ein Array von Slots mit den Daten, die tatsächlich eingefügt wurden. Diese können sich zum Beispiel infolge von Triggeraktionen von den übergebenen Daten unterscheiden. Die übergebenen Slots können hierfür wiederverwendet werden. Die Anzahl erfolgreich eingefügter Tupel wird in `*numSlots` zurückgegeben.

Die Daten im zurückgegebenen Slot werden nur verwendet, wenn die `INSERT`-Anweisung eine View mit `WITH CHECK OPTION` betrifft oder wenn die fremde Tabelle einen `AFTER ROW`-Trigger hat. Trigger benötigen alle Spalten, aber der FDW kann je nach Inhalt der `WITH CHECK OPTION`-Constraints entscheiden, die Rückgabe einiger oder aller Spalten wegzuoptimieren.

Wenn der Zeiger `ExecForeignBatchInsert` oder `GetForeignModifyBatchSize` auf `NULL` gesetzt ist, verwenden Versuche, in die fremde Tabelle einzufügen, `ExecForeignInsert`. Diese Funktion wird nicht verwendet, wenn das `INSERT` eine `RETURNING`-Klausel hat.

Beachten Sie, dass diese Funktion auch aufgerufen wird, wenn geroutete Tupel in eine Fremdtabellenpartition eingefügt werden oder `COPY FROM` auf einer fremden Tabelle ausgeführt wird. In diesem Fall wird sie anders aufgerufen als im `INSERT`-Fall. Siehe die unten beschriebenen Callback-Funktionen, die dem FDW erlauben, dies zu unterstützen.

```c
int
GetForeignModifyBatchSize(ResultRelInfo *rinfo);
```

Meldet die maximale Anzahl von Tupeln, die ein einzelner Aufruf von `ExecForeignBatchInsert` für die angegebene fremde Tabelle verarbeiten kann. Der Executor übergibt höchstens die angegebene Anzahl von Tupeln an `ExecForeignBatchInsert`. `rinfo` ist die `ResultRelInfo`-Struktur, die die fremde Zieltabelle beschreibt. Vom FDW wird erwartet, dass er eine Option für fremde Server und/oder fremde Tabellen bereitstellt, mit der der Benutzer diesen Wert setzen kann, oder dass er einen fest kodierten Wert verwendet.

Wenn der Zeiger `ExecForeignBatchInsert` oder `GetForeignModifyBatchSize` auf `NULL` gesetzt ist, verwenden Versuche, in die fremde Tabelle einzufügen, `ExecForeignInsert`.

```c
TupleTableSlot *
ExecForeignUpdate(EState *estate,
                  ResultRelInfo *rinfo,
                  TupleTableSlot *slot,
                  TupleTableSlot *planSlot);
```

Aktualisiert ein Tupel in der fremden Tabelle. `estate` ist der globale Ausführungszustand der Abfrage. `rinfo` ist die `ResultRelInfo`-Struktur, die die fremde Zieltabelle beschreibt. `slot` enthält die neuen Daten für das Tupel; es entspricht der Zeilentypdefinition der fremden Tabelle. `planSlot` enthält das Tupel, das vom Unterplan des `ModifyTable`-Planknotens erzeugt wurde. Anders als `slot` enthält dieses Tupel nur die neuen Werte für Spalten, die von der Abfrage geändert wurden. Verlassen Sie sich daher nicht auf Attributnummern der fremden Tabelle, um in `planSlot` zu indizieren. Außerdem enthält `planSlot` typischerweise zusätzliche Junk-Spalten. Insbesondere sind alle von `AddForeignUpdateTargets` angeforderten Junk-Spalten in diesem Slot verfügbar.

Der Rückgabewert ist entweder ein Slot mit der Zeile, wie sie tatsächlich aktualisiert wurde, was sich zum Beispiel infolge von Triggeraktionen von den übergebenen Daten unterscheiden kann, oder `NULL`, wenn keine Zeile tatsächlich aktualisiert wurde, wiederum typischerweise infolge von Triggern. Der übergebene `slot` kann hierfür wiederverwendet werden.

Die Daten im zurückgegebenen Slot werden nur verwendet, wenn die `UPDATE`-Anweisung eine `RETURNING`-Klausel hat oder eine View mit `WITH CHECK OPTION` beteiligt ist oder wenn die fremde Tabelle einen `AFTER ROW`-Trigger hat. Trigger benötigen alle Spalten, aber der FDW kann je nach Inhalt der `RETURNING`-Klausel oder der `WITH CHECK OPTION`-Constraints entscheiden, die Rückgabe einiger oder aller Spalten wegzuoptimieren. Unabhängig davon muss ein Slot zurückgegeben werden, um Erfolg anzuzeigen, sonst ist die von der Abfrage gemeldete Zeilenzahl falsch.

Wenn der Zeiger `ExecForeignUpdate` auf `NULL` gesetzt ist, schlagen Versuche, die fremde Tabelle zu aktualisieren, mit einer Fehlermeldung fehl.

```c
TupleTableSlot *
ExecForeignDelete(EState *estate,
                  ResultRelInfo *rinfo,
                  TupleTableSlot *slot,
                  TupleTableSlot *planSlot);
```

Löscht ein Tupel aus der fremden Tabelle. `estate` ist der globale Ausführungszustand der Abfrage. `rinfo` ist die `ResultRelInfo`-Struktur, die die fremde Zieltabelle beschreibt. `slot` enthält beim Aufruf nichts Nützliches, kann aber verwendet werden, um das zurückgegebene Tupel aufzunehmen. `planSlot` enthält das Tupel, das vom Unterplan des `ModifyTable`-Planknotens erzeugt wurde; insbesondere trägt es alle von `AddForeignUpdateTargets` angeforderten Junk-Spalten. Die Junk-Spalte oder -Spalten müssen verwendet werden, um das zu löschende Tupel zu identifizieren.

Der Rückgabewert ist entweder ein Slot mit der gelöschten Zeile oder `NULL`, wenn keine Zeile gelöscht wurde, typischerweise infolge von Triggern. Der übergebene `slot` kann verwendet werden, um das zurückzugebende Tupel aufzunehmen.

Die Daten im zurückgegebenen Slot werden nur verwendet, wenn die `DELETE`-Abfrage eine `RETURNING`-Klausel hat oder die fremde Tabelle einen `AFTER ROW`-Trigger besitzt. Trigger benötigen alle Spalten, aber der FDW kann je nach Inhalt der `RETURNING`-Klausel entscheiden, die Rückgabe einiger oder aller Spalten wegzuoptimieren. Unabhängig davon muss ein Slot zurückgegeben werden, um Erfolg anzuzeigen, sonst ist die von der Abfrage gemeldete Zeilenzahl falsch.

Wenn der Zeiger `ExecForeignDelete` auf `NULL` gesetzt ist, schlagen Versuche, aus der fremden Tabelle zu löschen, mit einer Fehlermeldung fehl.

```c
void
EndForeignModify(EState *estate,
                 ResultRelInfo *rinfo);
```

Beendet die Tabellenaktualisierung und gibt Ressourcen frei. Es ist normalerweise nicht wichtig, mit `palloc` allozierten Speicher freizugeben, aber zum Beispiel offene Dateien und Verbindungen zu entfernten Servern sollten bereinigt werden.

Wenn der Zeiger `EndForeignModify` auf `NULL` gesetzt ist, wird beim Herunterfahren des Executors keine Aktion ausgeführt.

Tupel, die per `INSERT` oder `COPY FROM` in eine partitionierte Tabelle eingefügt werden, werden an Partitionen weitergeleitet. Wenn ein FDW routbare Fremdtabellenpartitionen unterstützt, sollte er auch die folgenden Callback-Funktionen bereitstellen. Diese Funktionen werden außerdem aufgerufen, wenn `COPY FROM` auf einer fremden Tabelle ausgeführt wird.

```c
void
BeginForeignInsert(ModifyTableState *mtstate,
                   ResultRelInfo *rinfo);
```

Beginnt die Ausführung einer Einfügeoperation auf einer fremden Tabelle. Diese Routine wird unmittelbar vor dem ersten Tupel aufgerufen, das in die fremde Tabelle eingefügt wird, sowohl wenn sie die für Tuple Routing ausgewählte Partition ist als auch wenn sie das in einem `COPY FROM`-Befehl angegebene Ziel ist. Sie sollte alle Initialisierungen durchführen, die vor dem tatsächlichen Einfügen nötig sind. Anschließend werden `ExecForeignInsert` oder `ExecForeignBatchInsert` für die in die fremde Tabelle einzufügenden Tupel aufgerufen.

`mtstate` ist der Gesamtzustand des ausgeführten `ModifyTable`-Planknotens; globale Daten über Plan und Ausführungszustand sind über diese Struktur verfügbar. `rinfo` ist die `ResultRelInfo`-Struktur, die die fremde Zieltabelle beschreibt. Das Feld `ri_FdwState` von `ResultRelInfo` steht dem FDW zur Verfügung, um privaten Zustand für diese Operation zu speichern.

Wenn diese Funktion von einem `COPY FROM`-Befehl aufgerufen wird, werden die planbezogenen globalen Daten in `mtstate` nicht bereitgestellt, und der Parameter `planSlot` von `ExecForeignInsert`, das anschließend für jedes eingefügte Tupel aufgerufen wird, ist `NULL`, unabhängig davon, ob die fremde Tabelle die für Tuple Routing ausgewählte Partition oder das im Befehl angegebene Ziel ist.

Wenn der Zeiger `BeginForeignInsert` auf `NULL` gesetzt ist, wird bei der Initialisierung keine Aktion ausgeführt.

Beachten Sie, dass, wenn der FDW routbare Fremdtabellenpartitionen und/oder `COPY FROM` auf fremden Tabellen nicht unterstützt, diese Funktion oder die anschließend aufgerufenen Funktionen `ExecForeignInsert`/`ExecForeignBatchInsert` bei Bedarf einen Fehler werfen müssen.

```c
void
EndForeignInsert(EState *estate,
                 ResultRelInfo *rinfo);
```

Beendet die Einfügeoperation und gibt Ressourcen frei. Es ist normalerweise nicht wichtig, mit `palloc` allozierten Speicher freizugeben, aber zum Beispiel offene Dateien und Verbindungen zu entfernten Servern sollten bereinigt werden.

Wenn der Zeiger `EndForeignInsert` auf `NULL` gesetzt ist, wird bei der Beendigung keine Aktion ausgeführt.

```c
int
IsForeignRelUpdatable(Relation rel);
```

Meldet, welche Aktualisierungsoperationen die angegebene fremde Tabelle unterstützt. Der Rückgabewert sollte eine Bitmaske aus Rule-Event-Nummern sein, die angibt, welche Operationen von der fremden Tabelle unterstützt werden, wobei die Enumeration `CmdType` verwendet wird: `(1 << CMD_UPDATE) = 4` für `UPDATE`, `(1 << CMD_INSERT) = 8` für `INSERT` und `(1 << CMD_DELETE) = 16` für `DELETE`.

Wenn der Zeiger `IsForeignRelUpdatable` auf `NULL` gesetzt ist, wird angenommen, dass fremde Tabellen einfügbar, aktualisierbar oder löschbar sind, wenn der FDW jeweils `ExecForeignInsert`, `ExecForeignUpdate` oder `ExecForeignDelete` bereitstellt. Diese Funktion ist nur erforderlich, wenn der FDW einige Tabellen unterstützt, die aktualisierbar sind, und andere, die es nicht sind. Selbst dann ist es zulässig, in der Ausführungsroutine einen Fehler zu werfen, anstatt dies in dieser Funktion zu prüfen. Diese Funktion wird jedoch verwendet, um die Aktualisierbarkeit für die Anzeige in den `information_schema`-Views zu bestimmen.

Einige Inserts, Updates und Deletes auf fremden Tabellen können durch Implementierung einer alternativen Schnittstellengruppe optimiert werden. Die gewöhnlichen Schnittstellen für Inserts, Updates und Deletes holen Zeilen vom entfernten Server und ändern diese Zeilen dann einzeln. In manchen Fällen ist dieses zeilenweise Vorgehen notwendig, kann aber ineffizient sein. Wenn der fremde Server bestimmen kann, welche Zeilen geändert werden sollen, ohne sie tatsächlich abzurufen, und wenn keine lokalen Strukturen vorhanden sind, die die Operation beeinflussen würden, also lokale Row-Level-Trigger, gespeicherte generierte Spalten oder `WITH CHECK OPTION`-Constraints aus übergeordneten Views, kann die gesamte Operation auf dem entfernten Server ausgeführt werden. Die unten beschriebenen Schnittstellen machen dies möglich.

```c
bool
PlanDirectModify(PlannerInfo *root,
                 ModifyTable *plan,
                 Index resultRelation,
                 int subplan_index);
```

Entscheidet, ob es sicher ist, eine direkte Modifikation auf dem entfernten Server auszuführen. Wenn ja, gibt die Funktion nach Ausführung der dafür nötigen Planungsschritte `true` zurück. Andernfalls gibt sie `false` zurück. Diese optionale Funktion wird während der Abfrageplanung aufgerufen. Wenn sie erfolgreich ist, werden in der Ausführungsphase stattdessen `BeginDirectModify`, `IterateDirectModify` und `EndDirectModify` aufgerufen. Andernfalls wird die Tabellenmodifikation mit den oben beschriebenen Tabellenaktualisierungsfunktionen ausgeführt. Die Parameter entsprechen denen von `PlanForeignModify`.

Um die direkte Modifikation auf dem entfernten Server auszuführen, muss diese Funktion den Ziel-Unterplan mit einem `ForeignScan`-Planknoten umschreiben, der die direkte Modifikation auf dem entfernten Server ausführt. Die Felder `operation` und `resultRelation` des `ForeignScan` müssen passend gesetzt werden. `operation` muss auf den zur Anweisungsart passenden Wert der Enumeration `CmdType` gesetzt werden, also `CMD_UPDATE` für `UPDATE`, `CMD_INSERT` für `INSERT` und `CMD_DELETE` für `DELETE`; das Argument `resultRelation` muss in das Feld `resultRelation` kopiert werden.

Weitere Informationen finden Sie in [Abschnitt 58.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#584-abfrageplanung-für-foreign-data-wrapper).

Wenn der Zeiger `PlanDirectModify` auf `NULL` gesetzt ist, werden keine Versuche unternommen, eine direkte Modifikation auf dem entfernten Server auszuführen.

```c
void
BeginDirectModify(ForeignScanState *node,
                  int eflags);
```

Bereitet die Ausführung einer direkten Modifikation auf dem entfernten Server vor. Diese Funktion wird während des Executor-Starts aufgerufen. Sie sollte alle Initialisierungen durchführen, die vor der direkten Modifikation nötig sind; die eigentliche Modifikation sollte beim ersten Aufruf von `IterateDirectModify` erfolgen. Der `ForeignScanState`-Knoten wurde bereits erzeugt, aber sein Feld `fdw_state` ist noch `NULL`. Informationen über die zu ändernde Tabelle sind über den `ForeignScanState`-Knoten zugänglich, insbesondere über den zugrunde liegenden `ForeignScan`-Planknoten, der alle von `PlanDirectModify` bereitgestellten FDW-privaten Informationen enthält. `eflags` enthält Flag-Bits, die den Betriebsmodus des Executors für diesen Planknoten beschreiben.

Wenn `(eflags & EXEC_FLAG_EXPLAIN_ONLY)` wahr ist, sollte diese Funktion keine extern sichtbaren Aktionen ausführen. Sie sollte nur das Minimum tun, das erforderlich ist, damit der Knotenzustand für `ExplainDirectModify` und `EndDirectModify` gültig ist.

Wenn der Zeiger `BeginDirectModify` auf `NULL` gesetzt ist, werden keine Versuche unternommen, eine direkte Modifikation auf dem entfernten Server auszuführen.

```c
TupleTableSlot *
IterateDirectModify(ForeignScanState *node);
```

Wenn die `INSERT`-, `UPDATE`- oder `DELETE`-Abfrage keine `RETURNING`-Klausel hat, gibt diese Funktion nach einer direkten Modifikation auf dem entfernten Server einfach `NULL` zurück. Wenn die Abfrage eine solche Klausel hat, holt sie ein Ergebnis mit den für die `RETURNING`-Berechnung benötigten Daten und gibt es in einem Tuple-Table-Slot zurück; dafür sollte der `ScanTupleSlot` des Knotens verwendet werden. Die tatsächlich eingefügten, aktualisierten oder gelöschten Daten müssen in `node->resultRelInfo->ri_projectReturning->pi_exprContext->ecxt_scantuple` gespeichert werden. Gibt `NULL` zurück, wenn keine weiteren Zeilen verfügbar sind. Beachten Sie, dass diese Funktion in einem kurzlebigen Speicher-Kontext aufgerufen wird, der zwischen Aufrufen zurückgesetzt wird. Erzeugen Sie in `BeginDirectModify` einen Speicher-Kontext, wenn Sie langlebigere Speicherung benötigen, oder verwenden Sie `es_query_cxt` des `EState` des Knotens.

Die zurückgegebenen Zeilen müssen der Zielliste `fdw_scan_tlist` entsprechen, falls eine angegeben wurde; andernfalls müssen sie dem Zeilentyp der aktualisierten fremden Tabelle entsprechen. Wenn Sie das Holen von Spalten wegoptimieren, die für die `RETURNING`-Berechnung nicht benötigt werden, sollten Sie an diesen Spaltenpositionen Nullwerte einsetzen oder eine Liste `fdw_scan_tlist` erzeugen, in der diese Spalten ausgelassen sind.

Unabhängig davon, ob die Abfrage eine solche Klausel hat oder nicht, muss der FDW selbst die von der Abfrage gemeldete Zeilenzahl erhöhen. Wenn die Abfrage keine solche Klausel hat, muss der FDW im Fall von `EXPLAIN ANALYZE` außerdem die Zeilenzahl für den `ForeignScanState`-Knoten erhöhen.

Wenn der Zeiger `IterateDirectModify` auf `NULL` gesetzt ist, werden keine Versuche unternommen, eine direkte Modifikation auf dem entfernten Server auszuführen.

```c
void
EndDirectModify(ForeignScanState *node);
```

Bereinigt nach einer direkten Modifikation auf dem entfernten Server. Es ist normalerweise nicht wichtig, mit `palloc` allozierten Speicher freizugeben, aber zum Beispiel offene Dateien und Verbindungen zum entfernten Server sollten bereinigt werden.

Wenn der Zeiger `EndDirectModify` auf `NULL` gesetzt ist, werden keine Versuche unternommen, eine direkte Modifikation auf dem entfernten Server auszuführen.

### 58.2.5. FDW-Routinen für TRUNCATE

```c
void
ExecForeignTruncate(List *rels,
                    DropBehavior behavior,
                    bool restart_seqs);
```

Schneidet fremde Tabellen ab. Diese Funktion wird aufgerufen, wenn `TRUNCATE` auf einer fremden Tabelle ausgeführt wird. `rels` ist eine Liste von `Relation`-Datenstrukturen der abzuschneidenden fremden Tabellen.

`behavior` ist entweder `DROP_RESTRICT` oder `DROP_CASCADE` und gibt an, dass im ursprünglichen `TRUNCATE`-Befehl die Option `RESTRICT` beziehungsweise `CASCADE` angefordert wurde.

Wenn `restart_seqs` wahr ist, hat der ursprüngliche `TRUNCATE`-Befehl das Verhalten `RESTART IDENTITY` angefordert; andernfalls wurde `CONTINUE IDENTITY` angefordert.

Beachten Sie, dass die im ursprünglichen `TRUNCATE`-Befehl angegebenen `ONLY`-Optionen nicht an `ExecForeignTruncate` übergeben werden. Dieses Verhalten ähnelt den Callback-Funktionen von `SELECT`, `UPDATE` und `DELETE` auf einer fremden Tabelle.

`ExecForeignTruncate` wird einmal pro fremdem Server aufgerufen, für den fremde Tabellen abgeschnitten werden sollen. Das bedeutet, dass alle in `rels` enthaltenen fremden Tabellen zu demselben Server gehören müssen.

Wenn der Zeiger `ExecForeignTruncate` auf `NULL` gesetzt ist, schlagen Versuche, fremde Tabellen abzuschneiden, mit einer Fehlermeldung fehl.

### 58.2.6. FDW-Routinen für Row Locking

Wenn ein FDW spätes Row Locking unterstützen möchte, wie in [Abschnitt 58.5](58_Einen_Foreign_Data_Wrapper_schreiben.md#585-row-locking-in-foreign-data-wrappern) beschrieben, muss er die folgenden Callback-Funktionen bereitstellen:

```c
RowMarkType
GetForeignRowMarkType(RangeTblEntry *rte,
                      LockClauseStrength strength);
```

Meldet, welche Row-Marking-Option für eine fremde Tabelle zu verwenden ist. `rte` ist der `RangeTblEntry`-Knoten der Tabelle, und `strength` beschreibt die von der relevanten `FOR UPDATE`/`SHARE`-Klausel angeforderte Sperrstärke, falls vorhanden. Das Ergebnis muss ein Element des Enumerationstyps `RowMarkType` sein.

Diese Funktion wird während der Abfrageplanung für jede fremde Tabelle aufgerufen, die in einer `UPDATE`-, `DELETE`- oder `SELECT FOR UPDATE`/`SHARE`-Abfrage erscheint und nicht Ziel von `UPDATE` oder `DELETE` ist.

Wenn der Zeiger `GetForeignRowMarkType` auf `NULL` gesetzt ist, wird immer die Option `ROW_MARK_COPY` verwendet. Dies bedeutet, dass `RefetchForeignRow` nie aufgerufen wird und daher ebenfalls nicht bereitgestellt werden muss.

Weitere Informationen finden Sie in [Abschnitt 58.5](58_Einen_Foreign_Data_Wrapper_schreiben.md#585-row-locking-in-foreign-data-wrappern).

```c
void
RefetchForeignRow(EState *estate,
                  ExecRowMark *erm,
                  Datum rowid,
                  TupleTableSlot *slot,
                  bool *updated);
```

Holt einen Tuple-Slot aus der fremden Tabelle erneut, nachdem er bei Bedarf gesperrt wurde. `estate` ist der globale Ausführungszustand der Abfrage. `erm` ist die `ExecRowMark`-Struktur, die die fremde Zieltabelle und den zu erwerbenden Row-Lock-Typ beschreibt, falls vorhanden. `rowid` identifiziert das zu holende Tupel. `slot` enthält beim Aufruf nichts Nützliches, kann aber verwendet werden, um das zurückgegebene Tupel aufzunehmen. `updated` ist ein Ausgabeparameter.

Diese Funktion sollte das Tupel in den bereitgestellten Slot speichern oder ihn leeren, wenn die Zeilensperre nicht erworben werden konnte. Der zu erwerbende Row-Lock-Typ wird durch `erm->markType` definiert; dies ist der zuvor von `GetForeignRowMarkType` zurückgegebene Wert. `ROW_MARK_REFERENCE` bedeutet, das Tupel lediglich erneut zu holen, ohne eine Sperre zu erwerben; `ROW_MARK_COPY` wird von dieser Routine nie gesehen.

Zusätzlich sollte `*updated` auf `true` gesetzt werden, wenn das geholte Tupel eine aktualisierte Version des Tupels ist und nicht dieselbe Version wie zuvor. Wenn der FDW darüber nicht sicher sein kann, wird empfohlen, immer `true` zurückzugeben.

Beachten Sie, dass das Scheitern beim Erwerb einer Zeilensperre standardmäßig einen Fehler auslösen sollte. Die Rückkehr mit einem leeren Slot ist nur angemessen, wenn die Option `SKIP LOCKED` durch `erm->waitPolicy` angegeben wurde.

Die `rowid` ist der zuvor gelesene `ctid`-Wert der erneut zu holenden Zeile. Obwohl der `rowid`-Wert als `Datum` übergeben wird, kann er derzeit nur ein `tid` sein. Die Funktions-API wurde in der Hoffnung gewählt, dass künftig andere Datentypen für Zeilen-IDs erlaubt werden können.

Wenn der Zeiger `RefetchForeignRow` auf `NULL` gesetzt ist, schlagen Versuche, Zeilen erneut zu holen, mit einer Fehlermeldung fehl.

Weitere Informationen finden Sie in [Abschnitt 58.5](58_Einen_Foreign_Data_Wrapper_schreiben.md#585-row-locking-in-foreign-data-wrappern).

```c
bool
RecheckForeignScan(ForeignScanState *node,
                   TupleTableSlot *slot);
```

Prüft erneut, ob ein zuvor zurückgegebenes Tupel noch zu den relevanten Scan- und Join-Qualifikationen passt, und liefert möglicherweise eine geänderte Version des Tupels. Für Foreign Data Wrapper, die kein Join-Pushdown ausführen, ist es typischerweise bequemer, dies auf `NULL` zu setzen und stattdessen `fdw_recheck_quals` passend zu setzen. Wenn jedoch Outer Joins nach unten verschoben werden, reicht es nicht aus, die für alle Basistabellen relevanten Prüfungen erneut auf das Ergebnistupel anzuwenden, selbst wenn alle benötigten Attribute vorhanden sind. Der Grund ist, dass das Nichtbestehen einer Qualifikation dazu führen kann, dass einige Attribute `NULL` werden, statt dass gar kein Tupel zurückgegeben wird. `RecheckForeignScan` kann Qualifikationen erneut prüfen und `true` zurückgeben, wenn sie weiterhin erfüllt sind, andernfalls `false`; es kann aber auch ein Ersatztupel im übergebenen Slot speichern.

Zur Implementierung von Join-Pushdown konstruiert ein Foreign Data Wrapper typischerweise einen alternativen lokalen Join-Plan, der nur für Nachprüfungen verwendet wird. Dieser wird zum äußeren Unterplan des `ForeignScan`. Wenn eine Nachprüfung erforderlich ist, kann dieser Unterplan ausgeführt und das resultierende Tupel im Slot gespeichert werden. Dieser Plan muss nicht effizient sein, da keine Basistabelle mehr als eine Zeile zurückgibt; er kann zum Beispiel alle Joins als Nested Loops implementieren. Die Funktion `GetExistingLocalJoinPath` kann verwendet werden, um vorhandene Pfade nach einem geeigneten lokalen Join-Pfad zu durchsuchen, der als alternativer lokaler Join-Plan verwendet werden kann. `GetExistingLocalJoinPath` sucht in der Pfadliste der angegebenen Join-Relation nach einem nicht parametrisierten Pfad. Wenn kein solcher Pfad gefunden wird, gibt sie `NULL` zurück; ein Foreign Data Wrapper kann dann selbst den lokalen Pfad bauen oder entscheiden, keine Zugriffspfade für diesen Join zu erzeugen.

### 58.2.7. FDW-Routinen für EXPLAIN

```c
void
ExplainForeignScan(ForeignScanState *node,
                   ExplainState *es);
```

Gibt zusätzliche `EXPLAIN`-Ausgabe für einen Scan einer fremden Tabelle aus. Diese Funktion kann `ExplainPropertyText` und verwandte Funktionen aufrufen, um Felder zur `EXPLAIN`-Ausgabe hinzuzufügen. Die Flag-Felder in `es` können verwendet werden, um zu bestimmen, was ausgegeben wird; der Zustand des `ForeignScanState`-Knotens kann untersucht werden, um im Fall von `EXPLAIN ANALYZE` Laufzeitstatistiken bereitzustellen.

Wenn der Zeiger `ExplainForeignScan` auf `NULL` gesetzt ist, werden während `EXPLAIN` keine zusätzlichen Informationen ausgegeben.

```c
void
ExplainForeignModify(ModifyTableState *mtstate,
                     ResultRelInfo *rinfo,
                     List *fdw_private,
                     int subplan_index,
                     struct ExplainState *es);
```

Gibt zusätzliche `EXPLAIN`-Ausgabe für eine Aktualisierung einer fremden Tabelle aus. Diese Funktion kann `ExplainPropertyText` und verwandte Funktionen aufrufen, um Felder zur `EXPLAIN`-Ausgabe hinzuzufügen. Die Flag-Felder in `es` können verwendet werden, um zu bestimmen, was ausgegeben wird; der Zustand des `ModifyTableState`-Knotens kann untersucht werden, um im Fall von `EXPLAIN ANALYZE` Laufzeitstatistiken bereitzustellen. Die ersten vier Argumente entsprechen denen von `BeginForeignModify`.

Wenn der Zeiger `ExplainForeignModify` auf `NULL` gesetzt ist, werden während `EXPLAIN` keine zusätzlichen Informationen ausgegeben.

```c
void
ExplainDirectModify(ForeignScanState *node,
                    ExplainState *es);
```

Gibt zusätzliche `EXPLAIN`-Ausgabe für eine direkte Modifikation auf dem entfernten Server aus. Diese Funktion kann `ExplainPropertyText` und verwandte Funktionen aufrufen, um Felder zur `EXPLAIN`-Ausgabe hinzuzufügen. Die Flag-Felder in `es` können verwendet werden, um zu bestimmen, was ausgegeben wird; der Zustand des `ForeignScanState`-Knotens kann untersucht werden, um im Fall von `EXPLAIN ANALYZE` Laufzeitstatistiken bereitzustellen.

Wenn der Zeiger `ExplainDirectModify` auf `NULL` gesetzt ist, werden während `EXPLAIN` keine zusätzlichen Informationen ausgegeben.

### 58.2.8. FDW-Routinen für ANALYZE

```c
bool
AnalyzeForeignTable(Relation relation,
                    AcquireSampleRowsFunc *func,
                    BlockNumber *totalpages);
```

Diese Funktion wird aufgerufen, wenn `ANALYZE` auf einer fremden Tabelle ausgeführt wird. Wenn der FDW für diese fremde Tabelle Statistiken sammeln kann, sollte er `true` zurückgeben und in `func` einen Zeiger auf eine Funktion bereitstellen, die Beispielzeilen aus der Tabelle sammelt, sowie in `totalpages` die geschätzte Größe der Tabelle in Seiten. Andernfalls sollte er `false` zurückgeben.

Wenn der FDW das Sammeln von Statistiken für keine Tabellen unterstützt, kann der Zeiger `AnalyzeForeignTable` auf `NULL` gesetzt werden.

Falls bereitgestellt, muss die Funktion zum Sammeln der Stichprobe die folgende Signatur haben:

```c
int
AcquireSampleRowsFunc(Relation relation,
                      int elevel,
                      HeapTuple *rows,
                      int targrows,
                      double *totalrows,
                      double *totaldeadrows);
```

Eine zufällige Stichprobe von bis zu `targrows` Zeilen sollte aus der Tabelle gesammelt und im vom Aufrufer bereitgestellten Array `rows` gespeichert werden. Die tatsächlich gesammelte Anzahl von Zeilen muss zurückgegeben werden. Zusätzlich müssen Schätzungen der Gesamtzahlen lebender und toter Zeilen der Tabelle in den Ausgabeparametern `totalrows` und `totaldeadrows` gespeichert werden. Setzen Sie `totaldeadrows` auf null, wenn der FDW kein Konzept toter Zeilen hat.

### 58.2.9. FDW-Routinen für IMPORT FOREIGN SCHEMA

```c
List *
ImportForeignSchema(ImportForeignSchemaStmt *stmt, Oid serverOid);
```

Ermittelt eine Liste von Befehlen zum Erzeugen fremder Tabellen. Diese Funktion wird beim Ausführen von `IMPORT FOREIGN SCHEMA` aufgerufen und erhält den Parsebaum dieser Anweisung sowie die OID des zu verwendenden fremden Servers. Sie sollte eine Liste von C-Strings zurückgeben, von denen jeder einen `CREATE FOREIGN TABLE`-Befehl enthalten muss. Diese Strings werden vom Kernserver geparst und ausgeführt.

Innerhalb der Struktur `ImportForeignSchemaStmt` ist `remote_schema` der Name des entfernten Schemas, aus dem Tabellen importiert werden sollen. `list_type` identifiziert, wie Tabellennamen gefiltert werden: `FDW_IMPORT_SCHEMA_ALL` bedeutet, dass alle Tabellen im entfernten Schema importiert werden sollen; in diesem Fall ist `table_list` leer. `FDW_IMPORT_SCHEMA_LIMIT_TO` bedeutet, nur die in `table_list` aufgeführten Tabellen einzubeziehen, und `FDW_IMPORT_SCHEMA_EXCEPT` bedeutet, die in `table_list` aufgeführten Tabellen auszuschließen. `options` ist eine Liste von Optionen für den Importvorgang. Die Bedeutung der Optionen liegt beim FDW. Ein FDW könnte zum Beispiel eine Option verwenden, um festzulegen, ob `NOT NULL`-Attribute von Spalten importiert werden sollen. Diese Optionen müssen nichts mit den Optionen zu tun haben, die der FDW als Datenbankobjektoptionen unterstützt.

Der FDW kann das Feld `local_schema` von `ImportForeignSchemaStmt` ignorieren, weil der Kernserver diesen Namen automatisch in die geparsten `CREATE FOREIGN TABLE`-Befehle einsetzt.

Der FDW muss sich auch nicht darum kümmern, die durch `list_type` und `table_list` angegebene Filterung zu implementieren, da der Kernserver automatisch alle zurückgegebenen Befehle für Tabellen überspringt, die gemäß diesen Optionen ausgeschlossen sind. Es ist jedoch oft sinnvoll, die Arbeit zum Erzeugen von Befehlen für ausgeschlossene Tabellen von vornherein zu vermeiden. Die Funktion `IsImportableForeignTable()` kann nützlich sein, um zu prüfen, ob ein gegebener Name einer fremden Tabelle den Filter passiert.

Wenn der FDW den Import von Tabellendefinitionen nicht unterstützt, kann der Zeiger `ImportForeignSchema` auf `NULL` gesetzt werden.

### 58.2.10. FDW-Routinen für parallele Ausführung

Ein `ForeignScan`-Knoten kann optional parallele Ausführung unterstützen. Ein paralleler `ForeignScan` wird in mehreren Prozessen ausgeführt und muss jede Zeile über alle zusammenarbeitenden Prozesse hinweg genau einmal zurückgeben. Dazu können Prozesse sich über feste Blöcke dynamischen Shared Memorys koordinieren. Es ist nicht garantiert, dass dieser Shared Memory in jedem Prozess an derselben Adresse eingeblendet wird; er darf daher keine Zeiger enthalten. Die folgenden Funktionen sind alle optional, aber die meisten sind erforderlich, wenn parallele Ausführung unterstützt werden soll.

```c
bool
IsForeignScanParallelSafe(PlannerInfo *root,
                          RelOptInfo *rel,
                          RangeTblEntry *rte);
```

Prüft, ob ein Scan innerhalb eines parallelen Workers ausgeführt werden kann. Diese Funktion wird nur aufgerufen, wenn der Planner glaubt, dass ein paralleler Plan möglich sein könnte, und sollte `true` zurückgeben, wenn es sicher ist, diesen Scan in einem parallelen Worker auszuführen. Das ist im Allgemeinen nicht der Fall, wenn die entfernte Datenquelle Transaktionssemantik hat, es sei denn, die Verbindung des Workers zu den Daten kann irgendwie denselben Transaktionskontext wie der Leader teilen.

Wenn diese Funktion nicht definiert ist, wird angenommen, dass der Scan im Parallel-Leader stattfinden muss. Beachten Sie, dass `true` nicht bedeutet, dass der Scan selbst parallel erfolgen kann, sondern nur, dass er innerhalb eines parallelen Workers ausgeführt werden kann. Daher kann es sinnvoll sein, diese Methode auch dann zu definieren, wenn parallele Ausführung nicht unterstützt wird.

```c
Size
EstimateDSMForeignScan(ForeignScanState *node,
                       ParallelContext *pcxt);
```

Schätzt die Menge dynamischen Shared Memorys, die für parallelen Betrieb benötigt wird. Der Wert darf höher sein als die tatsächlich verwendete Menge, aber nicht niedriger. Der Rückgabewert ist in Bytes angegeben. Diese Funktion ist optional und kann ausgelassen werden, wenn sie nicht benötigt wird. Wird sie ausgelassen, müssen jedoch auch die nächsten drei Funktionen ausgelassen werden, weil kein Shared Memory zur Nutzung durch den FDW alloziert wird.

```c
void
InitializeDSMForeignScan(ForeignScanState *node,
                         ParallelContext *pcxt,
                         void *coordinate);
```

Initialisiert den dynamischen Shared Memory, der für parallelen Betrieb benötigt wird. `coordinate` zeigt auf einen Shared-Memory-Bereich, dessen Größe dem Rückgabewert von `EstimateDSMForeignScan` entspricht. Diese Funktion ist optional und kann ausgelassen werden, wenn sie nicht benötigt wird.

```c
void
ReInitializeDSMForeignScan(ForeignScanState *node,
                           ParallelContext *pcxt,
                           void *coordinate);
```

Initialisiert den für parallelen Betrieb benötigten dynamischen Shared Memory erneut, wenn der Foreign-Scan-Planknoten gerade erneut gescannt werden soll. Diese Funktion ist optional und kann ausgelassen werden, wenn sie nicht benötigt wird. Empfohlen wird, dass diese Funktion nur gemeinsamen Zustand zurücksetzt, während `ReScanForeignScan` nur lokalen Zustand zurücksetzt. Derzeit wird diese Funktion vor `ReScanForeignScan` aufgerufen, aber es ist besser, sich nicht auf diese Reihenfolge zu verlassen.

```c
void
InitializeWorkerForeignScan(ForeignScanState *node,
                            shm_toc *toc,
                            void *coordinate);
```

Initialisiert den lokalen Zustand eines parallelen Workers anhand des gemeinsamen Zustands, den der Leader während `InitializeDSMForeignScan` eingerichtet hat. Diese Funktion ist optional und kann ausgelassen werden, wenn sie nicht benötigt wird.

```c
void
ShutdownForeignScan(ForeignScanState *node);
```

Gibt Ressourcen frei, wenn erwartet wird, dass der Knoten nicht bis zum Abschluss ausgeführt wird. Diese Funktion wird nicht in allen Fällen aufgerufen; manchmal kann `EndForeignScan` aufgerufen werden, ohne dass diese Funktion zuvor aufgerufen wurde. Da das von parallelen Abfragen verwendete DSM-Segment unmittelbar nach dem Aufruf dieses Callbacks zerstört wird, sollten Foreign Data Wrapper, die vor dem Verschwinden des DSM-Segments noch eine Aktion ausführen wollen, diese Methode implementieren.

### 58.2.11. FDW-Routinen für asynchrone Ausführung

Ein `ForeignScan`-Knoten kann optional asynchrone Ausführung unterstützen, wie in `src/backend/executor/README` beschrieben. Die folgenden Funktionen sind alle optional, aber alle erforderlich, wenn asynchrone Ausführung unterstützt werden soll.

```c
bool
IsForeignPathAsyncCapable(ForeignPath *path);
```

Prüft, ob ein gegebener `ForeignPath`-Pfad die zugrunde liegende fremde Relation asynchron scannen kann. Diese Funktion wird nur am Ende der Abfrageplanung aufgerufen, wenn der gegebene Pfad ein direktes Kind eines `AppendPath`-Pfads ist und wenn der Planner glaubt, dass asynchrone Ausführung die Performance verbessert. Sie sollte `true` zurückgeben, wenn der gegebene Pfad die fremde Relation asynchron scannen kann.

Wenn diese Funktion nicht definiert ist, wird angenommen, dass der gegebene Pfad die fremde Relation mit `IterateForeignScan` scannt. Dies bedeutet, dass die unten beschriebenen Callback-Funktionen nie aufgerufen werden und daher ebenfalls nicht bereitgestellt werden müssen.

```c
void
ForeignAsyncRequest(AsyncRequest *areq);
```

Erzeugt asynchron ein Tupel aus dem `ForeignScan`-Knoten. `areq` ist die `AsyncRequest`-Struktur, die den `ForeignScan`-Knoten und den übergeordneten `Append`-Knoten beschreibt, der das Tupel von ihm angefordert hat. Diese Funktion sollte das Tupel in den durch `areq->result` angegebenen Slot speichern und `areq->request_complete` auf `true` setzen. Wenn sie auf ein Ereignis außerhalb des Kernservers warten muss, etwa Netzwerk-I/O, und nicht sofort ein Tupel erzeugen kann, setzt sie das Flag auf `false` und `areq->callback_pending` auf `true`, damit der `ForeignScan`-Knoten von den unten beschriebenen Callback-Funktionen zurückgerufen wird. Wenn keine weiteren Tupel verfügbar sind, setzt sie den Slot auf `NULL` oder einen leeren Slot und das Flag `areq->request_complete` auf `true`. Es wird empfohlen, `ExecAsyncRequestDone` oder `ExecAsyncRequestPending` zu verwenden, um die Ausgabeparameter in `areq` zu setzen.

```c
void
ForeignAsyncConfigureWait(AsyncRequest *areq);
```

Konfiguriert ein Dateideskriptor-Ereignis, auf das der `ForeignScan`-Knoten warten möchte. Diese Funktion wird nur aufgerufen, wenn beim `ForeignScan`-Knoten das Flag `areq->callback_pending` gesetzt ist, und sollte das Ereignis zum `as_eventset` des durch `areq` beschriebenen übergeordneten `Append`-Knotens hinzufügen. Weitere Informationen finden Sie in den Kommentaren zu `ExecAsyncConfigureWait` in `src/backend/executor/execAsync.c`. Wenn das Dateideskriptor-Ereignis eintritt, wird `ForeignAsyncNotify` aufgerufen.

```c
void
ForeignAsyncNotify(AsyncRequest *areq);
```

Verarbeitet ein relevantes eingetretenes Ereignis und erzeugt anschließend asynchron ein Tupel aus dem `ForeignScan`-Knoten. Diese Funktion sollte die Ausgabeparameter in `areq` auf dieselbe Weise setzen wie `ForeignAsyncRequest`.

### 58.2.12. FDW-Routinen zur Reparametrisierung von Pfaden

```c
List *
ReparameterizeForeignPathByChild(PlannerInfo *root,
                                 List *fdw_private,
                                 RelOptInfo *child_rel);
```

Diese Funktion wird aufgerufen, während ein Pfad, der durch das oberste Elternobjekt der angegebenen Kindrelation `child_rel` parametrisiert ist, so umgewandelt wird, dass er durch die Kindrelation parametrisiert ist. Die Funktion dient dazu, alle Pfade neu zu parametrisieren oder alle Ausdrucksknoten zu übersetzen, die im angegebenen Element `fdw_private` eines `ForeignPath` gespeichert sind. Der Callback kann bei Bedarf `reparameterize_path_by_child`, `adjust_appendrel_attrs` oder `adjust_appendrel_attrs_multilevel` verwenden.

## 58.3. Hilfsfunktionen für Foreign Data Wrapper

Mehrere Hilfsfunktionen werden vom Kernserver exportiert, damit Autoren von Foreign Data Wrappern bequem auf Attribute FDW-bezogener Objekte zugreifen können, etwa auf FDW-Optionen. Um eine dieser Funktionen zu verwenden, müssen Sie die Headerdatei `foreign/foreign.h` in Ihre Quelldatei einbinden. Dieser Header definiert außerdem die Strukturtypen, die von diesen Funktionen zurückgegeben werden.

```c
ForeignDataWrapper *
GetForeignDataWrapperExtended(Oid fdwid, bits16 flags);
```

Diese Funktion gibt ein `ForeignDataWrapper`-Objekt für den Foreign Data Wrapper mit der angegebenen OID zurück. Ein `ForeignDataWrapper`-Objekt enthält Eigenschaften des FDW; Details finden Sie in `foreign/foreign.h`. `flags` ist eine bitweise veroderte Bitmaske, die zusätzliche Optionen angibt. Sie kann den Wert `FDW_MISSING_OK` annehmen; in diesem Fall wird dem Aufrufer bei einem undefinierten Objekt `NULL` zurückgegeben, anstatt einen Fehler zu werfen.

```c
ForeignDataWrapper *
GetForeignDataWrapper(Oid fdwid);
```

Diese Funktion gibt ein `ForeignDataWrapper`-Objekt für den Foreign Data Wrapper mit der angegebenen OID zurück. Ein `ForeignDataWrapper`-Objekt enthält Eigenschaften des FDW; Details finden Sie in `foreign/foreign.h`.

```c
ForeignServer *
GetForeignServerExtended(Oid serverid, bits16 flags);
```

Diese Funktion gibt ein `ForeignServer`-Objekt für den fremden Server mit der angegebenen OID zurück. Ein `ForeignServer`-Objekt enthält Eigenschaften des Servers; Details finden Sie in `foreign/foreign.h`. `flags` ist eine bitweise veroderte Bitmaske, die zusätzliche Optionen angibt. Sie kann den Wert `FSV_MISSING_OK` annehmen; in diesem Fall wird dem Aufrufer bei einem undefinierten Objekt `NULL` zurückgegeben, anstatt einen Fehler zu werfen.

```c
ForeignServer *
GetForeignServer(Oid serverid);
```

Diese Funktion gibt ein `ForeignServer`-Objekt für den fremden Server mit der angegebenen OID zurück. Ein `ForeignServer`-Objekt enthält Eigenschaften des Servers; Details finden Sie in `foreign/foreign.h`.

```c
UserMapping *
GetUserMapping(Oid userid, Oid serverid);
```

Diese Funktion gibt ein `UserMapping`-Objekt für die Benutzerzuordnung der angegebenen Rolle auf dem angegebenen Server zurück. Wenn es keine Zuordnung für den konkreten Benutzer gibt, wird die Zuordnung für `PUBLIC` zurückgegeben oder ein Fehler geworfen, falls keine existiert. Ein `UserMapping`-Objekt enthält Eigenschaften der Benutzerzuordnung; Details finden Sie in `foreign/foreign.h`.

```c
ForeignTable *
GetForeignTable(Oid relid);
```

Diese Funktion gibt ein `ForeignTable`-Objekt für die fremde Tabelle mit der angegebenen OID zurück. Ein `ForeignTable`-Objekt enthält Eigenschaften der fremden Tabelle; Details finden Sie in `foreign/foreign.h`.

```c
List *
GetForeignColumnOptions(Oid relid, AttrNumber attnum);
```

Diese Funktion gibt die spaltenbezogenen FDW-Optionen für die Spalte mit der angegebenen OID der fremden Tabelle und Attributnummer in Form einer Liste von `DefElem` zurück. `NIL` wird zurückgegeben, wenn die Spalte keine Optionen hat.

Einige Objekttypen haben zusätzlich zu den OID-basierten Nachschlagefunktionen namensbasierte Funktionen:

```c
ForeignDataWrapper *
GetForeignDataWrapperByName(const char *name, bool missing_ok);
```

Diese Funktion gibt ein `ForeignDataWrapper`-Objekt für den Foreign Data Wrapper mit dem angegebenen Namen zurück. Wird der Wrapper nicht gefunden, gibt sie `NULL` zurück, wenn `missing_ok` wahr ist; andernfalls wird ein Fehler geworfen.

```c
ForeignServer *
GetForeignServerByName(const char *name, bool missing_ok);
```

Diese Funktion gibt ein `ForeignServer`-Objekt für den fremden Server mit dem angegebenen Namen zurück. Wird der Server nicht gefunden, gibt sie `NULL` zurück, wenn `missing_ok` wahr ist; andernfalls wird ein Fehler geworfen.

## 58.4. Abfrageplanung für Foreign Data Wrapper

Die FDW-Callback-Funktionen `GetForeignRelSize`, `GetForeignPaths`, `GetForeignPlan`, `PlanForeignModify`, `GetForeignJoinPaths`, `GetForeignUpperPaths` und `PlanDirectModify` müssen in die Arbeitsweise des PostgreSQL-Planners passen. Im Folgenden finden Sie einige Hinweise dazu, was sie leisten müssen.

Die Informationen in `root` und `baserel` können verwendet werden, um die Menge der aus der fremden Tabelle zu holenden Informationen zu verringern und damit die Kosten zu senken. Besonders interessant ist `baserel->baserestrictinfo`, da es Restriktions-Qualifikationen, also `WHERE`-Klauseln, enthält, die zum Filtern der zu holenden Zeilen verwendet werden sollten. Der FDW selbst ist nicht verpflichtet, diese Qualifikationen durchzusetzen, da der Kern-Executor sie stattdessen prüfen kann. `baserel->reltarget->exprs` kann verwendet werden, um zu bestimmen, welche Spalten geholt werden müssen. Beachten Sie aber, dass es nur Spalten aufführt, die vom `ForeignScan`-Planknoten ausgegeben werden müssen, nicht Spalten, die bei der Qualifikationsauswertung verwendet, aber von der Abfrage nicht ausgegeben werden.

Den FDW-Planungsfunktionen stehen verschiedene private Felder zur Verfügung, um Informationen aufzubewahren. Im Allgemeinen sollte alles, was Sie in FDW-privaten Feldern speichern, mit `palloc` alloziert werden, damit es am Ende der Planung freigegeben wird.

`baserel->fdw_private` ist ein `void`-Zeiger, der FDW-Planungsfunktionen zur Verfügung steht, um Informationen zu speichern, die für die jeweilige fremde Tabelle relevant sind. Der Kern-Planner berührt ihn nicht, außer ihn beim Erzeugen des `RelOptInfo`-Knotens auf `NULL` zu initialisieren. Er ist nützlich, um Informationen von `GetForeignRelSize` an `GetForeignPaths` und/oder von `GetForeignPaths` an `GetForeignPlan` weiterzugeben und so Neuberechnungen zu vermeiden.

`GetForeignPaths` kann die Bedeutung verschiedener Zugriffspfade identifizieren, indem private Informationen im Feld `fdw_private` von `ForeignPath`-Knoten gespeichert werden. `fdw_private` ist als `List`-Zeiger deklariert, könnte aber tatsächlich alles enthalten, da der Kern-Planner es nicht berührt. Als Best Practice sollte jedoch eine Darstellung verwendet werden, die von `nodeToString` ausgegeben werden kann, damit die im Backend verfügbare Debugging-Unterstützung nutzbar bleibt.

`GetForeignPlan` kann das Feld `fdw_private` des ausgewählten `ForeignPath`-Knotens untersuchen und Listen `fdw_exprs` und `fdw_private` erzeugen, die im `ForeignScan`-Planknoten abgelegt werden und zur Ausführungszeit verfügbar sind. Beide Listen müssen in einer Form dargestellt werden, die `copyObject` kopieren kann. Die Liste `fdw_private` unterliegt keinen weiteren Einschränkungen und wird vom Kern-Backend in keiner Weise interpretiert. Wenn die Liste `fdw_exprs` nicht `NIL` ist, wird erwartet, dass sie Ausdrucksbäume enthält, die zur Laufzeit ausgeführt werden sollen. Diese Bäume werden vom Planner nachbearbeitet, um vollständig ausführbar zu werden.

In `GetForeignPlan` kann die übergebene Zielliste im Allgemeinen unverändert in den Planknoten kopiert werden. Die übergebene Liste `scan_clauses` enthält dieselben Klauseln wie `baserel->baserestrictinfo`, kann aber für bessere Ausführungseffizienz umgeordnet sein. In einfachen Fällen kann der FDW die `RestrictInfo`-Knoten aus der Liste `scan_clauses` entfernen, etwa mit `extract_actual_clauses`, und alle Klauseln in die Qualifikationsliste des Planknotens aufnehmen. Das bedeutet, dass alle Klauseln zur Laufzeit vom Executor geprüft werden. Komplexere FDWs können einige der Klauseln möglicherweise intern prüfen; diese Klauseln können dann aus der Qualifikationsliste des Planknotens entfernt werden, damit der Executor keine Zeit mit ihrer erneuten Prüfung verschwendet.

Als Beispiel könnte der FDW einige Restriktionsklauseln der Form `foreign_variable = sub_expression` identifizieren, die seiner Einschätzung nach auf dem entfernten Server ausgeführt werden können, wenn der lokal ausgewertete Wert von `sub_expression` bekannt ist. Die eigentliche Identifikation einer solchen Klausel sollte während `GetForeignPaths` erfolgen, da sie die Kostenschätzung für den Pfad beeinflusst. Das Feld `fdw_private` des Pfads würde wahrscheinlich einen Zeiger auf den `RestrictInfo`-Knoten der identifizierten Klausel enthalten. `GetForeignPlan` würde diese Klausel dann aus `scan_clauses` entfernen, aber `sub_expression` zu `fdw_exprs` hinzufügen, um sicherzustellen, dass es in ausführbare Form gebracht wird. Wahrscheinlich würde es außerdem Steuerinformationen in das Feld `fdw_private` des Planknotens legen, um den Ausführungsfunktionen mitzuteilen, was zur Laufzeit zu tun ist. Die an den entfernten Server gesendete Abfrage enthielte etwas wie `WHERE foreign_variable = $1`, wobei der Parameterwert zur Laufzeit aus der Auswertung des Ausdrucksbaums in `fdw_exprs` gewonnen würde.

Alle Klauseln, die aus der Qualifikationsliste des Planknotens entfernt werden, müssen stattdessen zu `fdw_recheck_quals` hinzugefügt oder von `RecheckForeignScan` erneut geprüft werden, um korrektes Verhalten bei der Isolationsstufe `READ COMMITTED` sicherzustellen. Wenn ein gleichzeitiges Update an einer anderen an der Abfrage beteiligten Tabelle erfolgt, muss der Executor möglicherweise prüfen, ob alle ursprünglichen Qualifikationen für das Tupel weiterhin erfüllt sind, möglicherweise gegen eine andere Menge von Parameterwerten. Die Verwendung von `fdw_recheck_quals` ist typischerweise einfacher als die Implementierung von Prüfungen innerhalb von `RecheckForeignScan`; diese Methode reicht jedoch nicht aus, wenn Outer Joins nach unten verschoben wurden, da die Join-Tupel dann einige Felder auf `NULL` setzen können, ohne das Tupel vollständig zurückzuweisen.

Ein weiteres Feld von `ForeignScan`, das von FDWs gefüllt werden kann, ist `fdw_scan_tlist`. Es beschreibt die von diesem Planknoten vom FDW zurückgegebenen Tupel. Für einfache Scans fremder Tabellen kann es auf `NIL` gesetzt werden; dies bedeutet, dass die zurückgegebenen Tupel den für die fremde Tabelle deklarierten Zeilentyp haben. Ein nicht-`NIL`-Wert muss eine Zielliste, also eine Liste von `TargetEntry`s, sein, die `Var`s und/oder Ausdrücke enthält, welche die zurückgegebenen Spalten repräsentieren. Dies könnte zum Beispiel verwendet werden, um zu zeigen, dass der FDW einige Spalten weggelassen hat, von denen er erkannt hat, dass sie für die Abfrage nicht benötigt werden. Wenn der FDW Ausdrücke, die von der Abfrage verwendet werden, günstiger berechnen kann als lokal möglich, könnte er diese Ausdrücke außerdem zu `fdw_scan_tlist` hinzufügen. Beachten Sie, dass Join-Pläne, die aus von `GetForeignJoinPaths` erzeugten Pfaden entstehen, immer `fdw_scan_tlist` bereitstellen müssen, um die Menge der zurückgegebenen Spalten zu beschreiben.

Der FDW sollte immer mindestens einen Pfad konstruieren, der nur von den Restriktionsklauseln der Tabelle abhängt. In Join-Abfragen kann er außerdem Pfade konstruieren, die von Join-Klauseln abhängen, zum Beispiel `foreign_variable = local_variable`. Solche Klauseln befinden sich nicht in `baserel->baserestrictinfo`, sondern müssen in den Join-Listen der Relation gesucht werden. Ein Pfad, der eine solche Klausel verwendet, wird »parametrisierter Pfad« genannt. Er muss die anderen Relationen, die in den ausgewählten Join-Klauseln verwendet werden, mit einem passenden Wert von `param_info` identifizieren; verwenden Sie `get_baserel_parampathinfo`, um diesen Wert zu berechnen. In `GetForeignPlan` würde der Teil `local_variable` der Join-Klausel zu `fdw_exprs` hinzugefügt, und zur Laufzeit verhält sich der Fall dann wie bei einer gewöhnlichen Restriktionsklausel.

Wenn ein FDW entfernte Joins unterstützt, sollte `GetForeignJoinPaths` `ForeignPath`s für mögliche entfernte Joins in ähnlicher Weise erzeugen, wie `GetForeignPaths` für Basistabellen arbeitet. Informationen über den beabsichtigten Join können auf dieselben oben beschriebenen Arten an `GetForeignPlan` weitergegeben werden. `baserestrictinfo` ist für Join-Relationen jedoch nicht relevant; stattdessen werden die relevanten Join-Klauseln für einen bestimmten Join als separater Parameter `extra->restrictlist` an `GetForeignJoinPaths` übergeben.

Ein FDW kann zusätzlich die direkte Ausführung einiger Planaktionen oberhalb der Ebene von Scans und Joins unterstützen, etwa Gruppierung oder Aggregation. Um solche Optionen anzubieten, sollte der FDW Pfade erzeugen und sie in die passende obere Relation einfügen. Ein Pfad, der entfernte Aggregation repräsentiert, sollte zum Beispiel mit `add_path` in die Relation `UPPERREL_GROUP_AGG` eingefügt werden. Dieser Pfad wird kostenbasiert mit lokaler Aggregation verglichen, die durch Lesen eines einfachen Scan-Pfads für die fremde Relation erfolgt. Beachten Sie, dass ein solcher einfacher Pfad ebenfalls bereitgestellt werden muss, sonst tritt zur Planungszeit ein Fehler auf. Gewinnt der entfernte Aggregationspfad, was normalerweise der Fall wäre, wird er auf die übliche Weise durch Aufruf von `GetForeignPlan` in einen Plan umgewandelt. Der empfohlene Ort zum Erzeugen solcher Pfade ist die Callback-Funktion `GetForeignUpperPaths`, die für jede obere Relation aufgerufen wird, also für jeden Verarbeitungsschritt nach Scan oder Join, wenn alle Basisrelationen der Abfrage vom selben FDW stammen.

`PlanForeignModify` und die anderen in [Abschnitt 58.2.4](58_Einen_Foreign_Data_Wrapper_schreiben.md#5824-fdwroutinen-zum-aktualisieren-fremder-tabellen) beschriebenen Callbacks sind um die Annahme herum entworfen, dass die fremde Relation auf die übliche Weise gescannt wird und anschließend einzelne Zeilenaktualisierungen von einem lokalen `ModifyTable`-Planknoten gesteuert werden. Dieser Ansatz ist für den allgemeinen Fall erforderlich, in dem ein Update lokale Tabellen ebenso wie fremde Tabellen lesen muss. Wenn die Operation jedoch vollständig vom fremden Server ausgeführt werden könnte, könnte der FDW einen Pfad erzeugen, der dies repräsentiert, und ihn in die obere Relation `UPPERREL_FINAL` einfügen, wo er gegen den `ModifyTable`-Ansatz konkurrieren würde. Dieser Ansatz könnte auch verwendet werden, um entferntes `SELECT FOR UPDATE` zu implementieren, statt die in [Abschnitt 58.2.6](58_Einen_Foreign_Data_Wrapper_schreiben.md#5826-fdwroutinen-für-row-locking) beschriebenen Row-Locking-Callbacks zu verwenden. Denken Sie daran, dass ein in `UPPERREL_FINAL` eingefügter Pfad für die Implementierung des gesamten Verhaltens der Abfrage verantwortlich ist.

Beim Planen eines `UPDATE` oder `DELETE` können `PlanForeignModify` und `PlanDirectModify` die `RelOptInfo`-Struktur für die fremde Tabelle nachschlagen und die zuvor von den Scan-Planungsfunktionen erzeugten Daten `baserel->fdw_private` verwenden. Bei `INSERT` wird die Zieltabelle jedoch nicht gescannt, daher gibt es dafür keine `RelOptInfo`. Die von `PlanForeignModify` zurückgegebene `List` unterliegt denselben Einschränkungen wie die Liste `fdw_private` eines `ForeignScan`-Planknotens: Sie darf nur Strukturen enthalten, die `copyObject` kopieren kann.

`INSERT` mit einer `ON CONFLICT`-Klausel unterstützt die Angabe des Konfliktziels nicht, da Unique Constraints oder Exclusion Constraints auf entfernten Tabellen lokal nicht bekannt sind. Daraus folgt wiederum, dass `ON CONFLICT DO UPDATE` nicht unterstützt wird, da dort die Angabe zwingend erforderlich ist.

## 58.5. Row Locking in Foreign Data Wrappern

Wenn der zugrunde liegende Speichermechanismus eines FDW ein Konzept zum Sperren einzelner Zeilen besitzt, um gleichzeitige Aktualisierungen dieser Zeilen zu verhindern, lohnt es sich in der Regel, dass der FDW Row-Level-Locking mit einer möglichst genauen Annäherung an die Semantik gewöhnlicher PostgreSQL-Tabellen ausführt. Dabei sind mehrere Aspekte zu berücksichtigen.

Eine zentrale Entscheidung ist, ob frühes oder spätes Locking verwendet wird. Beim frühen Locking wird eine Zeile gesperrt, wenn sie erstmals aus dem zugrunde liegenden Speicher geholt wird. Beim späten Locking wird die Zeile erst gesperrt, wenn bekannt ist, dass sie gesperrt werden muss. Der Unterschied entsteht, weil einige Zeilen durch lokal geprüfte Restriktions- oder Join-Bedingungen verworfen werden können. Frühes Locking ist viel einfacher und vermeidet zusätzliche Roundtrips zu einem entfernten Speicher, kann aber Zeilen sperren, die gar nicht hätten gesperrt werden müssen, was zu geringerer Nebenläufigkeit oder sogar unerwarteten Deadlocks führen kann. Außerdem ist spätes Locking nur möglich, wenn die zu sperrende Zeile später eindeutig wiederidentifiziert werden kann. Vorzugsweise sollte der Zeilenbezeichner eine bestimmte Version der Zeile identifizieren, wie PostgreSQL-TIDs dies tun.

Standardmäßig ignoriert PostgreSQL Locking-Aspekte beim Umgang mit FDWs, aber ein FDW kann frühes Locking ohne ausdrückliche Unterstützung durch den Kerncode durchführen. Die in [Abschnitt 58.2.6](58_Einen_Foreign_Data_Wrapper_schreiben.md#5826-fdwroutinen-für-row-locking) beschriebenen API-Funktionen, die in PostgreSQL 9.5 hinzugefügt wurden, erlauben einem FDW, spätes Locking zu verwenden, wenn er dies möchte.

Ein weiterer Aspekt ist, dass PostgreSQL im Isolationsmodus `READ COMMITTED` Restriktions- und Join-Bedingungen möglicherweise gegen eine aktualisierte Version eines Ziel-Tupels erneut prüfen muss. Das erneute Prüfen von Join-Bedingungen erfordert, Kopien der Nicht-Zielzeilen erneut zu erhalten, die zuvor mit dem Ziel-Tupel gejoint wurden. Bei gewöhnlichen PostgreSQL-Tabellen geschieht dies, indem die TIDs der Nicht-Zieltabellen in die durch den Join projizierte Spaltenliste aufgenommen werden und die Nicht-Zielzeilen bei Bedarf erneut geholt werden. Dieser Ansatz hält die Join-Datenmenge kompakt, erfordert aber eine kostengünstige Fähigkeit zum erneuten Holen sowie eine TID, die die erneut zu holende Zeilenversion eindeutig identifizieren kann. Daher besteht der bei fremden Tabellen standardmäßig verwendete Ansatz darin, eine Kopie der gesamten aus einer fremden Tabelle geholten Zeile in die durch den Join projizierte Spaltenliste aufzunehmen. Das stellt keine besonderen Anforderungen an den FDW, kann aber die Performance von Merge- und Hash-Joins verringern. Ein FDW, der die Anforderungen für erneutes Holen erfüllen kann, kann sich für die erste Vorgehensweise entscheiden.

Für ein `UPDATE` oder `DELETE` auf einer fremden Tabelle wird empfohlen, dass die `ForeignScan`-Operation auf der Zieltabelle frühes Locking für die geholten Zeilen ausführt, etwa über ein Äquivalent von `SELECT FOR UPDATE`. Ein FDW kann zur Planungszeit erkennen, ob eine Tabelle Ziel von `UPDATE`/`DELETE` ist, indem er ihre `relid` mit `root->parse->resultRelation` vergleicht, oder zur Ausführungszeit mit `ExecRelationIsTargetRelation()`. Eine Alternative besteht darin, spätes Locking innerhalb des Callbacks `ExecForeignUpdate` oder `ExecForeignDelete` auszuführen; hierfür wird jedoch keine besondere Unterstützung bereitgestellt.

Für fremde Tabellen, die durch einen Befehl `SELECT FOR UPDATE`/`SHARE` gesperrt werden sollen, kann die `ForeignScan`-Operation wiederum frühes Locking ausführen, indem Tupel mit einem Äquivalent von `SELECT FOR UPDATE`/`SHARE` geholt werden. Um stattdessen spätes Locking auszuführen, stellen Sie die in [Abschnitt 58.2.6](58_Einen_Foreign_Data_Wrapper_schreiben.md#5826-fdwroutinen-für-row-locking) definierten Callback-Funktionen bereit. Wählen Sie in `GetForeignRowMarkType` je nach angeforderter Sperrstärke die Rowmark-Option `ROW_MARK_EXCLUSIVE`, `ROW_MARK_NOKEYEXCLUSIVE`, `ROW_MARK_SHARE` oder `ROW_MARK_KEYSHARE`. Der Kerncode verhält sich unabhängig davon, welche dieser vier Optionen Sie wählen, gleich. An anderer Stelle können Sie erkennen, ob eine fremde Tabelle durch diesen Befehlstyp gesperrt werden sollte, indem Sie zur Planungszeit `get_plan_rowmark` oder zur Ausführungszeit `ExecFindRowMark` verwenden. Sie müssen nicht nur prüfen, ob eine Nicht-Null-Rowmark-Struktur zurückgegeben wird, sondern auch, dass ihr Feld `strength` nicht `LCS_NONE` ist.

Schließlich können Sie für fremde Tabellen, die in einem `UPDATE`-, `DELETE`- oder `SELECT FOR UPDATE`/`SHARE`-Befehl verwendet werden, aber nicht zum Sperren von Zeilen angegeben sind, die Standardwahl, ganze Zeilen zu kopieren, überschreiben, indem `GetForeignRowMarkType` bei Sperrstärke `LCS_NONE` die Option `ROW_MARK_REFERENCE` auswählt. Dadurch wird `RefetchForeignRow` mit diesem Wert für `markType` aufgerufen; es sollte dann die Zeile erneut holen, ohne eine neue Sperre zu erwerben. Wenn Sie eine `GetForeignRowMarkType`-Funktion haben, aber ungesperrte Zeilen nicht erneut holen möchten, wählen Sie für `LCS_NONE` die Option `ROW_MARK_COPY`.

Weitere Informationen finden Sie in `src/include/nodes/lockoptions.h`, in den Kommentaren zu `RowMarkType` und `PlanRowMark` in `src/include/nodes/plannodes.h` sowie in den Kommentaren zu `ExecRowMark` in `src/include/nodes/execnodes.h`.
