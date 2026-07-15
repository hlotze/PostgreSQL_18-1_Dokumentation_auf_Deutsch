# 60. Einen Custom-Scan-Provider schreiben

PostgreSQL unterstützt eine Reihe experimenteller Einrichtungen, die Erweiterungsmodulen erlauben sollen, neue Scan-Typen zum System hinzuzufügen. Anders als ein Foreign Data Wrapper, der nur wissen muss, wie seine eigenen fremden Tabellen gescannt werden, kann ein Custom-Scan-Provider eine alternative Methode zum Scannen beliebiger Relationen im System bereitstellen. Typischerweise besteht die Motivation zum Schreiben eines Custom-Scan-Providers darin, eine Optimierung nutzbar zu machen, die vom Kernsystem nicht unterstützt wird, zum Beispiel Caching oder eine Form von Hardwarebeschleunigung. Dieses Kapitel skizziert, wie ein neuer Custom-Scan-Provider geschrieben wird.

Die Implementierung eines neuen Typs von Custom Scan erfolgt in drei Schritten. Zuerst müssen während der Planung Zugriffspfade erzeugt werden, die einen Scan mit der vorgeschlagenen Strategie repräsentieren. Wenn anschließend einer dieser Zugriffspfade vom Planner als optimale Strategie zum Scannen einer bestimmten Relation ausgewählt wird, muss der Zugriffspfad in einen Plan umgewandelt werden. Schließlich muss der Plan ausgeführt werden können und dieselben Ergebnisse erzeugen, die auch jeder andere Zugriffspfad für dieselbe Relation erzeugt hätte.

## 60.1. Custom-Scan-Pfade erzeugen

Ein Custom-Scan-Provider fügt Pfade für eine Basisrelation typischerweise hinzu, indem er den folgenden Hook setzt. Dieser Hook wird aufgerufen, nachdem der Kerncode alle Zugriffspfade erzeugt hat, die er für die Relation erzeugen kann. Ausgenommen sind `Gather`- und `Gather Merge`-Pfade; diese werden nach diesem Aufruf erstellt, damit sie vom Hook hinzugefügte partielle Pfade verwenden können.

```c
typedef void (*set_rel_pathlist_hook_type) (PlannerInfo *root,
                                            RelOptInfo *rel,
                                            Index rti,
                                            RangeTblEntry *rte);
extern PGDLLIMPORT set_rel_pathlist_hook_type
set_rel_pathlist_hook;
```

Obwohl diese Hook-Funktion verwendet werden kann, um vom Kernsystem erzeugte Pfade zu untersuchen, zu verändern oder zu entfernen, wird sich ein Custom-Scan-Provider typischerweise darauf beschränken, `CustomPath`-Objekte zu erzeugen und sie mit `add_path` zu `rel` hinzuzufügen, oder mit `add_partial_path`, wenn es partielle Pfade sind. Der Custom-Scan-Provider ist dafür verantwortlich, das `CustomPath`-Objekt zu initialisieren. Es ist wie folgt deklariert:

```c
typedef struct CustomPath
{
    Path       path;
    uint32     flags;
    List      *custom_paths;
    List      *custom_restrictinfo;
    List      *custom_private;
    const CustomPathMethods *methods;
} CustomPath;
```

`path` muss wie jeder andere Pfad initialisiert werden, einschließlich der Zeilenschätzung, der Start- und Gesamtkosten sowie der durch diesen Pfad bereitgestellten Sortierreihenfolge. `flags` ist eine Bitmaske, die angibt, ob der Scan-Provider bestimmte optionale Fähigkeiten unterstützt. `flags` sollte `CUSTOMPATH_SUPPORT_BACKWARD_SCAN` enthalten, wenn der Custom Path Rückwärtsscans unterstützt, `CUSTOMPATH_SUPPORT_MARK_RESTORE`, wenn er Markieren und Wiederherstellen unterstützt, und `CUSTOMPATH_SUPPORT_PROJECTION`, wenn er Projektionen ausführen kann. Wenn `CUSTOMPATH_SUPPORT_PROJECTION` nicht gesetzt ist, wird der Scan-Knoten nur aufgefordert, `Var`s der gescannten Relation zu erzeugen. Ist dieses Flag gesetzt, muss der Scan-Knoten skalare Ausdrücke über diesen `Var`s auswerten können.

Das optionale Feld `custom_paths` ist eine Liste von `Path`-Knoten, die von diesem Custom-Path-Knoten verwendet werden; diese werden vom Planner in `Plan`-Knoten umgewandelt. Wie unten beschrieben, können Custom Paths auch für Join-Relationen erzeugt werden. In einem solchen Fall sollte `custom_restrictinfo` verwendet werden, um die Menge der Join-Klauseln zu speichern, die auf den Join anzuwenden sind, den der Custom Path ersetzt. Andernfalls sollte es `NIL` sein. `custom_private` kann zum Speichern privater Daten des Custom Path verwendet werden. Private Daten sollten in einer Form gespeichert werden, die von `nodeToString` verarbeitet werden kann, damit Debugging-Routinen, die den Custom Path ausgeben wollen, wie vorgesehen funktionieren. `methods` muss auf ein üblicherweise statisch alloziertes Objekt zeigen, das die erforderlichen Custom-Path-Methoden implementiert; diese werden weiter unten genauer beschrieben.

Ein Custom-Scan-Provider kann auch Join-Pfade bereitstellen. Wie bei Basisrelationen muss ein solcher Pfad dieselbe Ausgabe erzeugen, die normalerweise von dem Join erzeugt würde, den er ersetzt. Dazu sollte der Join-Provider den folgenden Hook setzen und innerhalb der Hook-Funktion `CustomPath`-Pfade für die Join-Relation erzeugen.

```c
typedef void (*set_join_pathlist_hook_type) (PlannerInfo *root,
                                             RelOptInfo *joinrel,
                                             RelOptInfo *outerrel,
                                             RelOptInfo *innerrel,
                                             JoinType jointype,
                                             JoinPathExtraData *extra);
extern PGDLLIMPORT set_join_pathlist_hook_type
set_join_pathlist_hook;
```

Dieser Hook wird für dieselbe Join-Relation wiederholt mit verschiedenen Kombinationen von inneren und äußeren Relationen aufgerufen. Es liegt in der Verantwortung des Hooks, doppelte Arbeit zu minimieren.

Beachten Sie außerdem, dass die Menge der auf den Join anzuwendenden Join-Klauseln, die als `extra->restrictlist` übergeben wird, je nach Kombination von inneren und äußeren Relationen variiert. Ein für `joinrel` erzeugter `CustomPath`-Pfad muss die Menge der von ihm verwendeten Join-Klauseln enthalten. Der Planner verwendet diese Klauseln, um den `CustomPath`-Pfad in einen Plan umzuwandeln, falls er ihn als besten Pfad für `joinrel` auswählt.

### 60.1.1. Callbacks für Custom-Scan-Pfade

```c
Plan *(*PlanCustomPath)(PlannerInfo *root,
                        RelOptInfo *rel,
                        CustomPath *best_path,
                        List *tlist,
                        List *clauses,
                        List *custom_plans);
```

Wandelt einen Custom Path in einen fertigen Plan um. Der Rückgabewert ist im Allgemeinen ein `CustomScan`-Objekt, das der Callback allozieren und initialisieren muss. Weitere Details finden Sie in [Abschnitt 60.2](60_Einen_Custom_Scan_Provider_schreiben.md#602-customscanpläne-erzeugen).

```c
List *(*ReparameterizeCustomPathByChild)(PlannerInfo *root,
                                         List *custom_private,
                                         RelOptInfo *child_rel);
```

Dieser Callback wird aufgerufen, während ein Pfad, der durch das oberste Elternobjekt der angegebenen Kindrelation `child_rel` parametrisiert ist, so umgewandelt wird, dass er durch die Kindrelation parametrisiert ist. Der Callback dient dazu, alle Pfade neu zu parametrisieren oder alle Ausdrucksknoten zu übersetzen, die im angegebenen `custom_private`-Element eines `CustomPath` gespeichert sind. Der Callback kann bei Bedarf `reparameterize_path_by_child`, `adjust_appendrel_attrs` oder `adjust_appendrel_attrs_multilevel` verwenden.

## 60.2. Custom-Scan-Pläne erzeugen

Ein Custom Scan wird in einem fertigen Planbaum durch die folgende Struktur dargestellt:

```c
typedef struct CustomScan
{
    Scan       scan;
    uint32     flags;
    List      *custom_plans;
    List      *custom_exprs;
    List      *custom_private;
    List      *custom_scan_tlist;
    Bitmapset *custom_relids;
    const CustomScanMethods *methods;
} CustomScan;
```

`scan` muss wie jeder andere Scan initialisiert werden, einschließlich geschätzter Kosten, Ziellisten, Qualifikationen und so weiter. `flags` ist eine Bitmaske mit derselben Bedeutung wie in `CustomPath`. `custom_plans` kann zum Speichern untergeordneter `Plan`-Knoten verwendet werden. `custom_exprs` sollte zum Speichern von Ausdrucksbäumen verwendet werden, die von `setrefs.c` und `subselect.c` angepasst werden müssen, während `custom_private` andere private Daten enthalten sollte, die nur vom Custom-Scan-Provider selbst verwendet werden. `custom_scan_tlist` kann beim Scannen einer Basisrelation `NIL` sein; dies bedeutet, dass der Custom Scan Scan-Tupel zurückgibt, die dem Zeilentyp der Basisrelation entsprechen. Andernfalls ist es eine Zielliste, die die tatsächlichen Scan-Tupel beschreibt. Für Joins muss `custom_scan_tlist` bereitgestellt werden; bei Scans kann sie bereitgestellt werden, wenn der Custom-Scan-Provider einige Nicht-`Var`-Ausdrücke berechnen kann. `custom_relids` wird vom Kerncode auf die Menge der Relationen (Range-Table-Indizes) gesetzt, die dieser Scan-Knoten behandelt. Außer wenn dieser Scan einen Join ersetzt, enthält sie nur ein Element. `methods` muss auf ein üblicherweise statisch alloziertes Objekt zeigen, das die erforderlichen Custom-Scan-Methoden implementiert; diese werden weiter unten genauer beschrieben.

Wenn ein `CustomScan` eine einzelne Relation scannt, muss `scan.scanrelid` der Range-Table-Index der zu scannenden Tabelle sein. Wenn er einen Join ersetzt, sollte `scan.scanrelid` null sein.

Planbäume müssen mit `copyObject` dupliziert werden können. Daher müssen alle Daten, die in den `custom`-Feldern gespeichert werden, aus Knoten bestehen, die diese Funktion verarbeiten kann. Außerdem können Custom-Scan-Provider keine größere Struktur, die einen `CustomScan` einbettet, anstelle der Struktur selbst einsetzen, wie es bei einem `CustomPath` oder `CustomScanState` möglich wäre.

### 60.2.1. Callbacks für Custom-Scan-Pläne

```c
Node *(*CreateCustomScanState)(CustomScan *cscan);
```

Alloziert einen `CustomScanState` für diesen `CustomScan`. Die tatsächliche Allokation ist häufig größer als für einen gewöhnlichen `CustomScanState` erforderlich, da viele Provider diesen als erstes Feld einer größeren Struktur einbetten wollen. Der zurückgegebene Wert muss den Knotentag und die Methoden passend gesetzt haben; andere Felder sollten zu diesem Zeitpunkt jedoch als Nullen belassen werden. Nachdem `ExecInitCustomScan` die grundlegende Initialisierung durchgeführt hat, wird der Callback `BeginCustomScan` aufgerufen, damit der Custom-Scan-Provider alles Weitere tun kann, was nötig ist.

## 60.3. Custom Scans ausführen

Wenn ein `CustomScan` ausgeführt wird, wird sein Ausführungszustand durch einen `CustomScanState` dargestellt, der wie folgt deklariert ist:

```c
typedef struct CustomScanState
{
    ScanState ss;
    uint32    flags;
    const CustomExecMethods *methods;
} CustomScanState;
```

`ss` wird wie jeder andere Scan-Zustand initialisiert, außer dass `ss.ss_currentRelation` auf `NULL` belassen wird, wenn der Scan einen Join und keine Basisrelation betrifft. `flags` ist eine Bitmaske mit derselben Bedeutung wie in `CustomPath` und `CustomScan`. `methods` muss auf ein üblicherweise statisch alloziertes Objekt zeigen, das die erforderlichen Methoden für den Custom-Scan-Zustand implementiert; diese werden weiter unten genauer beschrieben. Typischerweise ist ein `CustomScanState`, der `copyObject` nicht unterstützen muss, tatsächlich eine größere Struktur, die die oben gezeigte Struktur als erstes Element einbettet.

### 60.3.1. Callbacks für die Ausführung von Custom Scans

```c
void (*BeginCustomScan)(CustomScanState *node,
                        EState *estate,
                        int eflags);
```

Schließt die Initialisierung des übergebenen `CustomScanState` ab. Standardfelder wurden von `ExecInitCustomScan` initialisiert, aber private Felder sollten hier initialisiert werden.

```c
TupleTableSlot *(*ExecCustomScan)(CustomScanState *node);
```

Holt das nächste Scan-Tupel. Wenn weitere Tupel vorhanden sind, sollte der Callback `ps_ResultTupleSlot` mit dem nächsten Tupel in der aktuellen Scan-Richtung füllen und dann den Tuple-Slot zurückgeben. Andernfalls sollte `NULL` oder ein leerer Slot zurückgegeben werden.

```c
void (*EndCustomScan)(CustomScanState *node);
```

Bereinigt alle privaten Daten, die zum `CustomScanState` gehören. Diese Methode ist erforderlich, muss aber nichts tun, wenn keine zugehörigen Daten existieren oder diese automatisch bereinigt werden.

```c
void (*ReScanCustomScan)(CustomScanState *node);
```

Setzt den aktuellen Scan auf den Anfang zurück und bereitet einen erneuten Scan der Relation vor.

```c
void (*MarkPosCustomScan)(CustomScanState *node);
```

Speichert die aktuelle Scan-Position, damit sie später durch den Callback `RestrPosCustomScan` wiederhergestellt werden kann. Dieser Callback ist optional und muss nur bereitgestellt werden, wenn das Flag `CUSTOMPATH_SUPPORT_MARK_RESTORE` gesetzt ist.

```c
void (*RestrPosCustomScan)(CustomScanState *node);
```

Stellt die zuvor durch den Callback `MarkPosCustomScan` gespeicherte Scan-Position wieder her. Dieser Callback ist optional und muss nur bereitgestellt werden, wenn das Flag `CUSTOMPATH_SUPPORT_MARK_RESTORE` gesetzt ist.

```c
Size (*EstimateDSMCustomScan)(CustomScanState *node,
                              ParallelContext *pcxt);
```

Schätzt die Menge dynamischen Shared Memorys, die für parallelen Betrieb benötigt wird. Der Wert darf höher sein als die tatsächlich verwendete Menge, aber nicht niedriger. Der Rückgabewert ist in Bytes angegeben. Dieser Callback ist optional und muss nur bereitgestellt werden, wenn dieser Custom-Scan-Provider parallele Ausführung unterstützt.

```c
void (*InitializeDSMCustomScan)(CustomScanState *node,
                                ParallelContext *pcxt,
                                void *coordinate);
```

Initialisiert den dynamischen Shared Memory, der für parallelen Betrieb benötigt wird. `coordinate` zeigt auf einen Shared-Memory-Bereich, dessen Größe dem Rückgabewert von `EstimateDSMCustomScan` entspricht. Dieser Callback ist optional und muss nur bereitgestellt werden, wenn dieser Custom-Scan-Provider parallele Ausführung unterstützt.

```c
void (*ReInitializeDSMCustomScan)(CustomScanState *node,
                                  ParallelContext *pcxt,
                                  void *coordinate);
```

Initialisiert den für parallelen Betrieb benötigten dynamischen Shared Memory erneut, wenn der Custom-Scan-Planknoten gerade erneut gescannt werden soll. Dieser Callback ist optional und muss nur bereitgestellt werden, wenn dieser Custom-Scan-Provider parallele Ausführung unterstützt. Empfohlen wird, dass dieser Callback nur gemeinsamen Zustand zurücksetzt, während `ReScanCustomScan` nur lokalen Zustand zurücksetzt. Derzeit wird dieser Callback vor `ReScanCustomScan` aufgerufen, aber es ist besser, sich nicht auf diese Reihenfolge zu verlassen.

```c
void (*InitializeWorkerCustomScan)(CustomScanState *node,
                                   shm_toc *toc,
                                   void *coordinate);
```

Initialisiert den lokalen Zustand eines parallelen Workers anhand des gemeinsamen Zustands, den der Leader während `InitializeDSMCustomScan` eingerichtet hat. Dieser Callback ist optional und muss nur bereitgestellt werden, wenn dieser Custom-Scan-Provider parallele Ausführung unterstützt.

```c
void (*ShutdownCustomScan)(CustomScanState *node);
```

Gibt Ressourcen frei, wenn erwartet wird, dass der Knoten nicht bis zum Abschluss ausgeführt wird. Diese Funktion wird nicht in allen Fällen aufgerufen; manchmal kann `EndCustomScan` aufgerufen werden, ohne dass diese Funktion zuvor aufgerufen wurde. Da das von parallelen Abfragen verwendete DSM-Segment unmittelbar nach dem Aufruf dieses Callbacks zerstört wird, sollten Custom-Scan-Provider, die vor dem Verschwinden des DSM-Segments noch eine Aktion ausführen wollen, diese Methode implementieren.

```c
void (*ExplainCustomScan)(CustomScanState *node,
                          List *ancestors,
                          ExplainState *es);
```

Gibt zusätzliche Informationen für `EXPLAIN` eines Custom-Scan-Planknotens aus. Dieser Callback ist optional. Gemeinsame Daten, die im `ScanState` gespeichert sind, etwa die Zielliste und die Scan-Relation, werden auch ohne diesen Callback angezeigt; der Callback erlaubt jedoch die Anzeige zusätzlicher privater Zustandsinformationen.
