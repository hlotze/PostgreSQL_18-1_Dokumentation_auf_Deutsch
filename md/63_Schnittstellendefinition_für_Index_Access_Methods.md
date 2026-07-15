# 63. Schnittstellendefinition für Index Access Methods

Dieses Kapitel definiert die Schnittstelle zwischen dem PostgreSQL-Kernsystem und Index Access Methods, die einzelne Indextypen verwalten. Das Kernsystem weiß über Indizes nur das, was hier festgelegt ist. Daher ist es möglich, durch Zusatzcode vollständig neue Indextypen zu entwickeln.

Alle Indizes in PostgreSQL sind technisch gesehen sekundäre Indizes. Das heißt, der Index ist physisch getrennt von der Tabellendatei, die er beschreibt. Jeder Index wird als eigene physische Relation gespeichert und deshalb durch einen Eintrag im Katalog `pg_class` beschrieben. Der Inhalt eines Index liegt vollständig unter der Kontrolle seiner Index Access Method. In der Praxis teilen alle Index Access Methods Indizes in Seiten standardisierter Größe auf, sodass sie den regulären Storage Manager und Buffer Manager verwenden können, um auf Indexinhalte zuzugreifen. Alle vorhandenen Index Access Methods verwenden außerdem das in [Abschnitt 66.6](66_Physische_Speicherung_der_Datenbank.md#666-layout-von-datenbankseiten) beschriebene Standardseitenlayout, und die meisten verwenden dasselbe Format für Index-Tupel-Header. Diese Entscheidungen werden einer Zugriffsmethode jedoch nicht vorgeschrieben.

Ein Index ist im Wesentlichen eine Abbildung von Daten-Schlüsselwerten auf Tupelbezeichner, also TIDs, von Zeilenversionen (Tupeln) in der übergeordneten Tabelle des Index. Eine TID besteht aus einer Blocknummer und einer Itemnummer innerhalb dieses Blocks; siehe [Abschnitt 66.6](66_Physische_Speicherung_der_Datenbank.md#666-layout-von-datenbankseiten). Diese Informationen reichen aus, um eine bestimmte Zeilenversion aus der Tabelle zu holen. Indizes wissen nicht direkt, dass unter MVCC mehrere vorhandene Versionen derselben logischen Zeile existieren können. Für einen Index ist jedes Tupel ein eigenständiges Objekt, das einen eigenen Indexeintrag benötigt. Daher erzeugt ein Update einer Zeile immer vollständig neue Indexeinträge für die Zeile, selbst wenn sich die Schlüsselwerte nicht geändert haben. HOT-Tupel bilden eine Ausnahme von dieser Aussage, aber auch mit ihnen beschäftigen sich Indizes nicht direkt. Indexeinträge für tote Tupel werden beim Vacuuming zurückgewonnen, wenn die toten Tupel selbst zurückgewonnen werden.

## 63.1. Grundlegende API-Struktur für Indizes

Jede Index Access Method wird durch eine Zeile im Systemkatalog `pg_am` beschrieben. Der `pg_am`-Eintrag gibt einen Namen und eine Handlerfunktion für die Index Access Method an. Diese Einträge können mit den SQL-Befehlen `CREATE ACCESS METHOD` und `DROP ACCESS METHOD` erzeugt und gelöscht werden.

Eine Handlerfunktion für eine Index Access Method muss so deklariert sein, dass sie ein einzelnes Argument vom Typ `internal` akzeptiert und den Pseudotyp `index_am_handler` zurückgibt. Das Argument ist ein Dummy-Wert, der lediglich verhindert, dass Handlerfunktionen direkt aus SQL-Befehlen aufgerufen werden. Das Ergebnis der Funktion muss eine mit `palloc` allozierte Struktur vom Typ `IndexAmRoutine` sein. Diese enthält alles, was der Kerncode wissen muss, um die Index Access Method zu verwenden. Die Struktur `IndexAmRoutine`, auch API-Struktur der Zugriffsmethode genannt, enthält Felder, die verschiedene feste Eigenschaften der Zugriffsmethode angeben, etwa ob sie Mehrspaltenindizes unterstützen kann. Wichtiger ist, dass sie Zeiger auf Support-Funktionen der Zugriffsmethode enthält, die die eigentliche Arbeit beim Zugriff auf Indizes ausführen. Diese Support-Funktionen sind einfache C-Funktionen und auf SQL-Ebene weder sichtbar noch aufrufbar. Die Support-Funktionen werden in [Abschnitt 63.2](63_Schnittstellendefinition_für_Index_Access_Methods.md#632-funktionen-von-index-access-methods) beschrieben.

Die Struktur `IndexAmRoutine` ist wie folgt definiert:

```c
typedef struct IndexAmRoutine
{
    NodeTag     type;

    /*
     * Total number of strategies (operators) by which we can
     * traverse/search this AM. Zero if AM does not have a fixed set
     * of strategy assignments.
     */
    uint16      amstrategies;
    /* total number of support functions that this AM uses */
    uint16      amsupport;
    /* opclass options support function number or 0 */
    uint16      amoptsprocnum;
    /* does AM support ORDER BY indexed column's value? */
    bool        amcanorder;
    /* does AM support ORDER BY result of an operator on indexed column? */
    bool        amcanorderbyop;
    /* does AM support hashing using API consistent with the hash AM? */
    bool        amcanhash;
    /* do operators within an opfamily have consistent equality semantics? */
    bool        amconsistentequality;
    /* do operators within an opfamily have consistent ordering semantics? */
    bool        amconsistentordering;
    /* does AM support backward scanning? */
    bool        amcanbackward;
    /* does AM support UNIQUE indexes? */
    bool        amcanunique;
    /* does AM support multi-column indexes? */
    bool        amcanmulticol;
    /* does AM require scans to have a constraint on the first index column? */
    bool        amoptionalkey;
    /* does AM handle ScalarArrayOpExpr quals? */
    bool        amsearcharray;
    /* does AM handle IS NULL/IS NOT NULL quals? */
    bool        amsearchnulls;
    /* can index storage data type differ from column data type? */
    bool        amstorage;
    /* can an index of this type be clustered on? */
    bool        amclusterable;
    /* does AM handle predicate locks? */
    bool        ampredlocks;
    /* does AM support parallel scan? */
    bool        amcanparallel;
    /* does AM support parallel build? */
    bool        amcanbuildparallel;
    /* does AM support columns included with clause INCLUDE? */
    bool        amcaninclude;
    /* does AM use maintenance_work_mem? */
    bool        amusemaintenanceworkmem;
    /*
     * does AM summarize tuples, with at least all tuples in the block
     * summarized in one summary
     */
    bool        amsummarizing;
    /* OR of parallel vacuum flags */
    uint8       amparallelvacuumoptions;
    /* type of data stored in index, or InvalidOid if variable */
    Oid         amkeytype;

    /* interface functions */
    ambuild_function ambuild;
    ambuildempty_function ambuildempty;
    aminsert_function aminsert;
    aminsertcleanup_function aminsertcleanup;   /* can be NULL */
    ambulkdelete_function ambulkdelete;
    amvacuumcleanup_function amvacuumcleanup;
    amcanreturn_function amcanreturn;           /* can be NULL */
    amcostestimate_function amcostestimate;
    amgettreeheight_function amgettreeheight;   /* can be NULL */
    amoptions_function amoptions;
    amproperty_function amproperty;             /* can be NULL */
    ambuildphasename_function ambuildphasename; /* can be NULL */
    amvalidate_function amvalidate;
    amadjustmembers_function amadjustmembers;   /* can be NULL */
    ambeginscan_function ambeginscan;
    amrescan_function amrescan;
    amgettuple_function amgettuple;             /* can be NULL */
    amgetbitmap_function amgetbitmap;           /* can be NULL */
    amendscan_function amendscan;
    ammarkpos_function ammarkpos;               /* can be NULL */
    amrestrpos_function amrestrpos;             /* can be NULL */

    /* interface functions to support parallel index scans */
    amestimateparallelscan_function amestimateparallelscan; /* can be NULL */
    aminitparallelscan_function aminitparallelscan;         /* can be NULL */
    amparallelrescan_function amparallelrescan;             /* can be NULL */

    /* interface functions to support planning */
    amtranslate_strategy_function amtranslatestrategy;      /* can be NULL */
    amtranslate_cmptype_function amtranslatecmptype;        /* can be NULL */
} IndexAmRoutine;
```

Um nützlich zu sein, muss eine Index Access Method außerdem eine oder mehrere Operatorfamilien und Operatorklassen definieren, und zwar in `pg_opfamily`, `pg_opclass`, `pg_amop` und `pg_amproc`. Diese Einträge erlauben dem Planner zu bestimmen, welche Arten von Abfragequalifikationen mit Indizes dieser Zugriffsmethode verwendet werden können. Operatorfamilien und Operatorklassen werden in [Abschnitt 36.16](36_SQL_erweitern.md#3616-erweiterungen-an-indizes-anbinden) beschrieben; dieser Abschnitt ist Voraussetzung für das Verständnis dieses Kapitels.

Ein einzelner Index wird durch einen `pg_class`-Eintrag definiert, der ihn als physische Relation beschreibt, sowie durch einen `pg_index`-Eintrag, der den logischen Inhalt des Index angibt, also die Menge seiner Indexspalten und die Semantik dieser Spalten, wie sie durch die zugehörigen Operatorklassen erfasst wird. Die Indexspalten (Schlüsselwerte) können entweder einfache Spalten der zugrunde liegenden Tabelle oder Ausdrücke über Tabellenzeilen sein. Die Index Access Method interessiert sich normalerweise nicht dafür, woher die Indexschlüsselwerte stammen, da ihr immer vorberechnete Schlüsselwerte übergeben werden. Sie wird sich jedoch sehr für die Operatorklasseninformationen in `pg_index` interessieren. Beide Katalogeinträge sind als Teil der `Relation`-Datenstruktur zugänglich, die an alle Operationen auf dem Index übergeben wird.

Einige Flag-Felder von `IndexAmRoutine` haben nicht offensichtliche Auswirkungen. Die Anforderungen an `amcanunique` werden in [Abschnitt 63.5](63_Schnittstellendefinition_für_Index_Access_Methods.md#635-indexeindeutigkeitsprüfungen) besprochen. Das Flag `amcanmulticol` sagt aus, dass die Zugriffsmethode Mehrschlüsselspaltenindizes unterstützt, während `amoptionalkey` aussagt, dass sie Scans erlaubt, bei denen keine indexierbare Restriktionsklausel für die erste Indexspalte angegeben ist. Wenn `amcanmulticol` `false` ist, sagt `amoptionalkey` im Wesentlichen, ob die Zugriffsmethode vollständige Indexscans ohne irgendeine Restriktionsklausel unterstützt. Zugriffsmethoden, die mehrere Indexspalten unterstützen, müssen Scans unterstützen, die Restriktionen für beliebige oder alle Spalten nach der ersten auslassen. Sie dürfen jedoch verlangen, dass für die erste Indexspalte irgendeine Restriktion vorhanden ist; dies wird signalisiert, indem `amoptionalkey` auf `false` gesetzt wird. Ein Grund, warum eine Index-AM `amoptionalkey` auf `false` setzen könnte, ist, dass sie keine Nullwerte indiziert.

Da die meisten indexierbaren Operatoren strikt sind und daher bei Null-Eingaben nicht `true` zurückgeben können, erscheint es auf den ersten Blick attraktiv, keine Indexeinträge für Nullwerte zu speichern: Sie könnten ohnehin nie von einem Indexscan zurückgegeben werden. Dieses Argument schlägt jedoch fehl, wenn ein Indexscan für eine gegebene Indexspalte keine Restriktionsklausel hat. In der Praxis bedeutet das, dass Indizes mit `amoptionalkey = true` Nullwerte indizieren müssen, da der Planner entscheiden könnte, einen solchen Index ganz ohne Scan Keys zu verwenden. Eine verwandte Einschränkung lautet, dass eine Index Access Method, die mehrere Indexspalten unterstützt, das Indizieren von Nullwerten in Spalten nach der ersten unterstützen muss, weil der Planner annimmt, dass der Index für Abfragen verwendet werden kann, die diese Spalten nicht einschränken. Betrachten Sie zum Beispiel einen Index auf `(a,b)` und eine Abfrage mit `WHERE a = 4`. Das System wird annehmen, dass der Index verwendet werden kann, um nach Zeilen mit `a = 4` zu scannen. Das wäre falsch, wenn der Index Zeilen auslässt, bei denen `b` null ist. Es ist jedoch in Ordnung, Zeilen auszulassen, bei denen die erste indizierte Spalte null ist. Eine Index Access Method, die Nullwerte indiziert, kann außerdem `amsearchnulls` setzen, um anzugeben, dass sie `IS NULL`- und `IS NOT NULL`-Klauseln als Suchbedingungen unterstützt.

Das Flag `amcaninclude` gibt an, ob die Zugriffsmethode »eingeschlossene« Spalten unterstützt, also zusätzliche Spalten jenseits der Schlüsselspalte oder -spalten speichern kann, ohne sie zu verarbeiten. Die Anforderungen des vorherigen Absatzes gelten nur für die Schlüsselspalten. Insbesondere ist die Kombination `amcanmulticol=false` und `amcaninclude=true` sinnvoll: Sie bedeutet, dass es nur eine Schlüsselspalte geben kann, aber zusätzlich eingeschlossene Spalten vorhanden sein können. Außerdem müssen eingeschlossene Spalten unabhängig von `amoptionalkey` null sein dürfen.

Das Flag `amsummarizing` gibt an, ob die Zugriffsmethode die indizierten Tupel zusammenfasst, mit einer Zusammenfassungsgranularität von mindestens einem Block. Zugriffsmethoden, die nicht auf einzelne Tupel, sondern auf Blockbereiche zeigen, wie BRIN, können die HOT-Optimierung weiterhin zulassen. Das gilt nicht für Attribute, auf die in Indexprädikaten verwiesen wird; ein Update eines solchen Attributs deaktiviert HOT immer.

## 63.2. Funktionen von Index Access Methods

Die Funktionen für Indexkonstruktion und -wartung, die eine Index Access Method in `IndexAmRoutine` bereitstellen muss, sind:

```c
IndexBuildResult *
ambuild(Relation heapRelation,
        Relation indexRelation,
        IndexInfo *indexInfo);
```

Erzeugt einen neuen Index. Die Indexrelation wurde physisch erzeugt, ist aber leer. Sie muss mit allen festen Daten gefüllt werden, die die Zugriffsmethode benötigt, sowie mit Einträgen für alle Tupel, die bereits in der Tabelle existieren. Normalerweise ruft die Funktion `ambuild` `table_index_build_scan()` auf, um die Tabelle nach vorhandenen Tupeln zu scannen und die Schlüssel zu berechnen, die in den Index eingefügt werden müssen. Die Funktion muss eine mit `palloc` allozierte Struktur zurückgeben, die Statistiken über den neuen Index enthält. Das Flag `amcanbuildparallel` gibt an, ob die Zugriffsmethode parallele Index-Builds unterstützt. Wenn es auf `true` gesetzt ist, versucht das System, parallele Worker für den Build zuzuweisen. Zugriffsmethoden, die nur nichtparallele Index-Builds unterstützen, sollten dieses Flag auf `false` belassen.

```c
void
ambuildempty(Relation indexRelation);
```

Erzeugt einen leeren Index und schreibt ihn in den Initialisierungs-Fork (`INIT_FORKNUM`) der angegebenen Relation. Diese Methode wird nur für ungeloggte Indizes aufgerufen; der in den Initialisierungs-Fork geschriebene leere Index wird bei jedem Serverneustart über den Haupt-Fork der Relation kopiert.

```c
bool
aminsert(Relation indexRelation,
         Datum *values,
         bool *isnull,
         ItemPointer heap_tid,
         Relation heapRelation,
         IndexUniqueCheck checkUnique,
         bool indexUnchanged,
         IndexInfo *indexInfo);
```

Fügt ein neues Tupel in einen vorhandenen Index ein. Die Arrays `values` und `isnull` geben die zu indizierenden Schlüsselwerte an, und `heap_tid` ist die zu indizierende TID. Wenn die Zugriffsmethode eindeutige Indizes unterstützt, also ihr Flag `amcanunique` `true` ist, gibt `checkUnique` die Art der auszuführenden Eindeutigkeitsprüfung an. Dies variiert abhängig davon, ob das Unique Constraint deferrable ist; Einzelheiten finden Sie in [Abschnitt 63.5](63_Schnittstellendefinition_für_Index_Access_Methods.md#635-indexeindeutigkeitsprüfungen). Normalerweise benötigt die Zugriffsmethode den Parameter `heapRelation` nur bei Eindeutigkeitsprüfungen, weil sie dann in den Heap schauen muss, um die Lebendigkeit von Tupeln zu verifizieren.

Der boolesche Wert `indexUnchanged` gibt einen Hinweis auf die Art des zu indizierenden Tupels. Wenn er `true` ist, ist das Tupel ein Duplikat eines vorhandenen Tupels im Index. Das neue Tupel ist eine logisch unveränderte nachfolgende MVCC-Tupelversion. Dies geschieht, wenn ein `UPDATE` ausgeführt wird, das keine vom Index abgedeckten Spalten ändert, aber dennoch eine neue Version im Index benötigt. Die Index-AM kann diesen Hinweis verwenden, um in Teilen des Index, in denen sich viele Versionen derselben logischen Zeile ansammeln, Bottom-up Index Deletion anzuwenden. Beachten Sie, dass das Aktualisieren einer Nicht-Schlüsselspalte oder einer Spalte, die nur in einem partiellen Indexprädikat erscheint, den Wert von `indexUnchanged` nicht beeinflusst. Der Kerncode bestimmt den Wert `indexUnchanged` jedes Tupels mit einem Verfahren mit geringem Overhead, das sowohl False Positives als auch False Negatives erlaubt. Index-AMs dürfen `indexUnchanged` nicht als verbindliche Informationsquelle über Tupelsichtbarkeit oder Versionierung behandeln.

Der boolesche Rückgabewert der Funktion ist nur relevant, wenn `checkUnique` den Wert `UNIQUE_CHECK_PARTIAL` hat. In diesem Fall bedeutet ein Ergebnis `true`, dass der neue Eintrag als eindeutig bekannt ist, während `false` bedeutet, dass er möglicherweise nicht eindeutig ist und eine zurückgestellte Eindeutigkeitsprüfung eingeplant werden muss. In anderen Fällen wird ein konstanter Rückgabewert `false` empfohlen.

Einige Indizes indizieren möglicherweise nicht alle Tupel. Wenn das Tupel nicht indiziert werden soll, sollte `aminsert` einfach zurückkehren, ohne etwas zu tun.

Wenn die Index-AM Daten über aufeinanderfolgende Indexeinfügungen innerhalb einer SQL-Anweisung hinweg zwischenspeichern möchte, kann sie Speicher in `indexInfo->ii_Context` allozieren und einen Zeiger auf die Daten in `indexInfo->ii_AmCache` speichern, das anfangs `NULL` ist. Wenn nach Indexeinfügungen Ressourcen außer Speicher freigegeben werden müssen, kann `aminsertcleanup` bereitgestellt werden; diese Funktion wird aufgerufen, bevor der Speicher freigegeben wird.

```c
void
aminsertcleanup(Relation indexRelation,
                IndexInfo *indexInfo);
```

Bereinigt Zustand, der über aufeinanderfolgende Einfügungen hinweg in `indexInfo->ii_AmCache` gehalten wurde. Das ist nützlich, wenn die Daten zusätzliche Bereinigungsschritte erfordern, zum Beispiel das Freigeben gepinnter Buffer, und das bloße Freigeben des Speichers nicht ausreicht.

```c
IndexBulkDeleteResult *
ambulkdelete(IndexVacuumInfo *info,
             IndexBulkDeleteResult *stats,
             IndexBulkDeleteCallback callback,
             void *callback_state);
```

Löscht Tupel aus dem Index. Dies ist eine Bulk-Delete-Operation, die dazu gedacht ist, durch Scannen des gesamten Index und Prüfen jedes Eintrags implementiert zu werden. Die übergebene Callback-Funktion muss im Stil `callback(TID, callback_state) returns bool` aufgerufen werden, um zu bestimmen, ob ein bestimmter Indexeintrag, identifiziert durch seine referenzierte TID, gelöscht werden soll. Die Funktion muss entweder `NULL` oder eine mit `palloc` allozierte Struktur zurückgeben, die Statistiken über die Auswirkungen der Löschoperation enthält. Es ist zulässig, `NULL` zurückzugeben, wenn keine Informationen an `amvacuumcleanup` weitergegeben werden müssen.

Wegen begrenztem `maintenance_work_mem` muss `ambulkdelete` möglicherweise mehrfach aufgerufen werden, wenn viele Tupel zu löschen sind. Das Argument `stats` ist das Ergebnis des vorherigen Aufrufs für diesen Index; beim ersten Aufruf innerhalb einer `VACUUM`-Operation ist es `NULL`. Dadurch kann die AM Statistiken über die gesamte Operation hinweg sammeln. Typischerweise ändert `ambulkdelete` dieselbe Struktur und gibt sie zurück, wenn das übergebene `stats` nicht null ist.

```c
IndexBulkDeleteResult *
amvacuumcleanup(IndexVacuumInfo *info,
                IndexBulkDeleteResult *stats);
```

Bereinigt nach einer `VACUUM`-Operation, also nach null oder mehr `ambulkdelete`-Aufrufen. Diese Funktion muss nichts weiter tun, als Indexstatistiken zurückzugeben, kann aber auch umfangreiche Bereinigung ausführen, etwa leere Indexseiten zurückgewinnen. `stats` ist das, was der letzte `ambulkdelete`-Aufruf zurückgegeben hat, oder `NULL`, wenn `ambulkdelete` nicht aufgerufen wurde, weil keine Tupel gelöscht werden mussten. Wenn das Ergebnis nicht `NULL` ist, muss es eine mit `palloc` allozierte Struktur sein. Die darin enthaltenen Statistiken werden zum Aktualisieren von `pg_class` verwendet und von `VACUUM` gemeldet, wenn `VERBOSE` angegeben ist. Es ist zulässig, `NULL` zurückzugeben, wenn der Index während der `VACUUM`-Operation überhaupt nicht geändert wurde; andernfalls sollten korrekte Statistiken zurückgegeben werden.

`amvacuumcleanup` wird auch beim Abschluss einer `ANALYZE`-Operation aufgerufen. In diesem Fall ist `stats` immer `NULL`, und jeder Rückgabewert wird ignoriert. Dieser Fall kann durch Prüfen von `info->analyze_only` unterschieden werden. Es wird empfohlen, dass die Zugriffsmethode in einem solchen Aufruf außer Post-Insert-Cleanup nichts tut, und das nur in einem Autovacuum-Worker-Prozess.

```c
bool
amcanreturn(Relation indexRelation, int attno);
```

Prüft, ob der Index Index-Only-Scans für die angegebene Spalte unterstützen kann, indem er den ursprünglichen indizierten Wert der Spalte zurückgibt. Die Attributnummer ist 1-basiert, das heißt, `attno` der ersten Spalte ist 1. Gibt `true` zurück, wenn dies unterstützt wird, andernfalls `false`. Diese Funktion sollte für eingeschlossene Spalten immer `true` zurückgeben, sofern solche unterstützt werden, da eine eingeschlossene Spalte, die nicht abgerufen werden kann, wenig Sinn hat. Wenn die Zugriffsmethode Index-Only-Scans überhaupt nicht unterstützt, kann das Feld `amcanreturn` in ihrer `IndexAmRoutine`-Struktur auf `NULL` gesetzt werden.

```c
void
amcostestimate(PlannerInfo *root,
               IndexPath *path,
               double loop_count,
               Cost *indexStartupCost,
               Cost *indexTotalCost,
               Selectivity *indexSelectivity,
               double *indexCorrelation,
               double *indexPages);
```

Schätzt die Kosten eines Indexscans. Diese Funktion wird unten in [Abschnitt 63.6](63_Schnittstellendefinition_für_Index_Access_Methods.md#636-funktionen-zur-indexkostenschätzung) vollständig beschrieben.

```c
int
amgettreeheight(Relation rel);
```

Berechnet die Höhe eines baumförmigen Index. Diese Information wird der Funktion `amcostestimate` in `path->indexinfo->tree_height` bereitgestellt und kann zur Unterstützung der Kostenschätzung verwendet werden. Das Ergebnis wird nirgendwo sonst verwendet, sodass diese Funktion tatsächlich jede Art von Daten über den Index berechnen kann, die in einen Integer passt und die die Kostenschätzungsfunktion kennen möchte. Wenn die Berechnung teuer ist, kann es sinnvoll sein, das Ergebnis als Teil von `RelationData.rd_amcache` zwischenzuspeichern.

```c
bytea *
amoptions(ArrayType *reloptions,
          bool validate);
```

Parst und validiert das Array `reloptions` für einen Index. Dies wird nur aufgerufen, wenn ein nicht-nulles `reloptions`-Array für den Index existiert. `reloptions` ist ein Text-Array mit Einträgen der Form `name=value`. Die Funktion sollte einen `bytea`-Wert konstruieren, der in das Feld `rd_options` des Relcache-Eintrags des Index kopiert wird. Der Dateninhalt des `bytea`-Werts kann von der Zugriffsmethode frei definiert werden; die meisten Standardzugriffsmethoden verwenden `struct StdRdOptions`. Wenn `validate` wahr ist, sollte die Funktion eine geeignete Fehlermeldung ausgeben, falls Optionen unbekannt sind oder ungültige Werte haben. Wenn `validate` falsch ist, sollten ungültige Einträge stillschweigend ignoriert werden. `validate` ist falsch, wenn Optionen geladen werden, die bereits in `pg_catalog` gespeichert sind; ein ungültiger Eintrag könnte nur gefunden werden, wenn die Zugriffsmethode ihre Regeln für Optionen geändert hat, und in diesem Fall ist es angemessen, veraltete Einträge zu ignorieren. Es ist zulässig, `NULL` zurückzugeben, wenn das Standardverhalten gewünscht ist.

```c
bool
amproperty(Oid index_oid, int attno,
           IndexAMProperty prop, const char *propname,
           bool *res, bool *isnull);
```

Die Methode `amproperty` erlaubt Index Access Methods, das Standardverhalten von `pg_index_column_has_property` und verwandten Funktionen zu überschreiben. Wenn die Zugriffsmethode kein besonderes Verhalten für Anfragen nach Indexeigenschaften hat, kann das Feld `amproperty` in ihrer `IndexAmRoutine`-Struktur auf `NULL` gesetzt werden. Andernfalls wird `amproperty` bei Aufrufen von `pg_indexam_has_property` mit `index_oid` und `attno` jeweils null aufgerufen, bei Aufrufen von `pg_index_has_property` mit gültiger `index_oid` und `attno` null, oder bei Aufrufen von `pg_index_column_has_property` mit gültiger `index_oid` und `attno` größer als null. `prop` ist ein Enum-Wert, der die getestete Eigenschaft identifiziert, während `propname` die ursprüngliche Zeichenkette des Eigenschaftsnamens ist. Wenn der Kerncode den Eigenschaftsnamen nicht erkennt, ist `prop` `AMPROP_UNKNOWN`. Zugriffsmethoden können eigene Eigenschaftsnamen definieren, indem sie `propname` auf Übereinstimmung prüfen; verwenden Sie für Konsistenz mit dem Kerncode `pg_strcasecmp`. Für dem Kerncode bekannte Namen ist es besser, `prop` zu untersuchen.

Wenn `amproperty` `true` zurückgibt, hat die Methode das Ergebnis des Eigenschaftstests bestimmt: Sie muss `*res` auf den zurückzugebenden booleschen Wert setzen oder `*isnull` auf `true`, um `NULL` zurückzugeben. Beide referenzierten Variablen werden vor dem Aufruf auf `false` initialisiert. Wenn `amproperty` `false` zurückgibt, fährt der Kerncode mit seiner normalen Logik zur Bestimmung des Ergebnisses fort.

Zugriffsmethoden, die Ordnungsoperatoren unterstützen, sollten die Eigenschaftsprüfung `AMPROP_DISTANCE_ORDERABLE` implementieren, da der Kerncode nicht weiß, wie er das tun soll, und `NULL` zurückgeben wird. Es kann außerdem vorteilhaft sein, die Prüfung `AMPROP_RETURNABLE` zu implementieren, wenn dies günstiger möglich ist, als den Index zu öffnen und `amcanreturn` aufzurufen, was das Standardverhalten des Kerncodes ist. Für alle anderen Standardeigenschaften sollte das Standardverhalten zufriedenstellend sein.

```c
char *
ambuildphasename(int64 phasenum);
```

Gibt den Textnamen der angegebenen Build-Phasennummer zurück. Die Phasennummern sind diejenigen, die während eines Index-Builds über die Schnittstelle `pgstat_progress_update_param` gemeldet werden. Die Phasennamen werden anschließend in der View `pg_stat_progress_create_index` sichtbar gemacht.

```c
bool
amvalidate(Oid opclassoid);
```

Validiert die Katalogeinträge für die angegebene Operatorklasse, soweit die Zugriffsmethode dies vernünftigerweise tun kann. Dazu könnte zum Beispiel gehören, zu prüfen, ob alle erforderlichen Support-Funktionen bereitgestellt werden. Die Funktion `amvalidate` muss `false` zurückgeben, wenn die Operatorklasse ungültig ist. Probleme sollten mit `ereport`-Meldungen gemeldet werden, typischerweise auf Ebene `INFO`.

```c
void
amadjustmembers(Oid opfamilyoid,
                Oid opclassoid,
                List *operators,
                List *functions);
```

Validiert vorgeschlagene neue Operator- und Funktionsmitglieder einer Operatorfamilie, soweit die Zugriffsmethode dies vernünftigerweise tun kann, und setzt ihre Abhängigkeitstypen, wenn der Standard nicht zufriedenstellend ist. Dies wird während `CREATE OPERATOR CLASS` und während `ALTER OPERATOR FAMILY ADD` aufgerufen; im letzteren Fall ist `opclassoid` `InvalidOid`. Die `List`-Argumente sind Listen von `OpFamilyMember`-Strukturen, wie sie in `amapi.h` definiert sind. Die von dieser Funktion ausgeführten Tests sind typischerweise eine Teilmenge der von `amvalidate` ausgeführten Tests, da `amadjustmembers` nicht annehmen kann, dass es eine vollständige Menge von Mitgliedern sieht. Es wäre zum Beispiel vernünftig, die Signatur einer Support-Funktion zu prüfen, aber nicht zu prüfen, ob alle erforderlichen Support-Funktionen bereitgestellt werden.

Probleme können gemeldet werden, indem ein Fehler geworfen wird. Die abhängigkeitbezogenen Felder der `OpFamilyMember`-Strukturen werden vom Kerncode initialisiert, um bei `CREATE OPERATOR CLASS` harte Abhängigkeiten von der Operatorklasse oder bei `ALTER OPERATOR FAMILY ADD` weiche Abhängigkeiten von der Operatorfamilie zu erzeugen. `amadjustmembers` kann diese Felder anpassen, wenn ein anderes Verhalten angemessener ist. GIN, GiST und SP-GiST setzen Operator-Mitglieder zum Beispiel immer so, dass sie weiche Abhängigkeiten von der Operatorfamilie haben, da die Verbindung zwischen einem Operator und einer Operatorklasse bei diesen Indextypen relativ schwach ist. Daher ist es sinnvoll, das freie Hinzufügen und Entfernen von Operator-Mitgliedern zu erlauben. Optionale Support-Funktionen erhalten typischerweise ebenfalls weiche Abhängigkeiten, damit sie bei Bedarf entfernt werden können.

Der Zweck eines Index besteht natürlich darin, Scans nach Tupeln zu unterstützen, die eine indexierbare `WHERE`-Bedingung erfüllen, oft Qualifier oder Scan Key genannt. Die Semantik des Indexscannens wird unten in [Abschnitt 63.3](63_Schnittstellendefinition_für_Index_Access_Methods.md#633-indexscans) ausführlicher beschrieben. Eine Index Access Method kann »einfache« Indexscans, »Bitmap«-Indexscans oder beides unterstützen. Die scanbezogenen Funktionen, die eine Index Access Method bereitstellen muss oder kann, sind:

```c
IndexScanDesc
ambeginscan(Relation indexRelation,
            int nkeys,
            int norderbys);
```

Bereitet einen Indexscan vor. Die Parameter `nkeys` und `norderbys` geben die Anzahl der Qualifikationen und Ordnungsoperatoren an, die im Scan verwendet werden; dies kann für Speicherzuweisungen nützlich sein. Beachten Sie, dass die tatsächlichen Werte der Scan Keys noch nicht bereitgestellt werden. Das Ergebnis muss eine mit `palloc` allozierte Struktur sein. Aus Implementierungsgründen muss die Index Access Method diese Struktur durch Aufruf von `RelationGetIndexScan()` erzeugen. In den meisten Fällen tut `ambeginscan` wenig mehr als diesen Aufruf auszuführen und vielleicht Locks zu erwerben; die interessanten Teile des Indexscan-Starts liegen in `amrescan`.

```c
void
amrescan(IndexScanDesc scan,
         ScanKey keys,
         int nkeys,
         ScanKey orderbys,
         int norderbys);
```

Startet oder startet einen Indexscan neu, möglicherweise mit neuen Scan Keys. Um mit zuvor übergebenen Keys neu zu starten, wird für `keys` und/oder `orderbys` `NULL` übergeben. Beachten Sie, dass die Anzahl der Keys oder Order-by-Operatoren nicht größer sein darf als die an `ambeginscan` übergebene. In der Praxis wird die Neustartfunktion verwendet, wenn bei einem Nested-Loop-Join ein neues äußeres Tupel ausgewählt wird und daher ein neuer Schlüsselvergleichswert benötigt wird, die Scan-Key-Struktur aber gleich bleibt.

```c
bool
amgettuple(IndexScanDesc scan,
           ScanDirection direction);
```

Holt das nächste Tupel im angegebenen Scan und bewegt sich dabei in die angegebene Richtung, also vorwärts oder rückwärts im Index. Gibt `true` zurück, wenn ein Tupel erhalten wurde, und `false`, wenn keine passenden Tupel mehr vorhanden sind. Im Erfolgsfall wird die Tupel-TID in der Scanstruktur gespeichert. Beachten Sie, dass »Erfolg« nur bedeutet, dass der Index einen Eintrag enthält, der zu den Scan Keys passt, nicht dass das Tupel notwendigerweise noch im Heap existiert oder den Snapshot-Test des Aufrufers besteht. Bei Erfolg muss `amgettuple` außerdem `scan->xs_recheck` auf `true` oder `false` setzen. `false` bedeutet, dass sicher ist, dass der Indexeintrag zu den Scan Keys passt. `true` bedeutet, dass dies nicht sicher ist und die durch die Scan Keys repräsentierten Bedingungen nach dem Holen gegen das Heap-Tupel erneut geprüft werden müssen. Diese Einrichtung unterstützt »lossy« Indexoperatoren. Beachten Sie, dass sich die Nachprüfung nur auf die Scanbedingungen erstreckt; ein partielles Indexprädikat, falls vorhanden, wird von Aufrufern von `amgettuple` nie erneut geprüft.

Wenn der Index Index-Only-Scans unterstützt, also `amcanreturn` für eine seiner Spalten `true` zurückgibt, muss die AM bei Erfolg außerdem `scan->xs_want_itup` prüfen. Wenn dieses Feld wahr ist, muss sie die ursprünglich indizierten Daten für den Indexeintrag zurückgeben. Spalten, für die `amcanreturn` `false` zurückgibt, können als null zurückgegeben werden. Die Daten können in Form eines `IndexTuple`-Zeigers zurückgegeben werden, der in `scan->xs_itup` gespeichert ist, mit Tupeldeskriptor `scan->xs_itupdesc`, oder in Form eines `HeapTuple`-Zeigers, der in `scan->xs_hitup` gespeichert ist, mit Tupeldeskriptor `scan->xs_hitupdesc`. Das letztere Format sollte verwendet werden, wenn Daten rekonstruiert werden, die möglicherweise nicht in ein `IndexTuple` passen. In beiden Fällen ist die Zugriffsmethode für die Verwaltung der Daten verantwortlich, auf die der Zeiger verweist. Die Daten müssen mindestens bis zum nächsten Aufruf von `amgettuple`, `amrescan` oder `amendscan` für den Scan gültig bleiben.

Die Funktion `amgettuple` muss nur bereitgestellt werden, wenn die Zugriffsmethode »einfache« Indexscans unterstützt. Wenn nicht, muss das Feld `amgettuple` in ihrer `IndexAmRoutine`-Struktur auf `NULL` gesetzt werden.

```c
int64
amgetbitmap(IndexScanDesc scan,
            TIDBitmap *tbm);
```

Holt alle Tupel im angegebenen Scan und fügt sie der vom Aufrufer bereitgestellten `TIDBitmap` hinzu, also per ODER-Verknüpfung die Menge von Tupel-IDs zu der bereits vorhandenen Menge in der Bitmap. Die Anzahl der geholten Tupel wird zurückgegeben; das kann nur eine ungefähre Zahl sein, zum Beispiel weil manche AMs Duplikate nicht erkennen. Beim Einfügen von Tupel-IDs in die Bitmap kann `amgetbitmap` angeben, dass die Scanbedingungen für bestimmte Tupel-IDs erneut geprüft werden müssen. Dies entspricht dem Ausgabeparameter `xs_recheck` von `amgettuple`. Hinweis: In der aktuellen Implementierung ist die Unterstützung dieser Funktion mit der Unterstützung lossy Speicherung der Bitmap selbst vermischt. Daher prüfen Aufrufer für erneut zu prüfende Tupel sowohl die Scanbedingungen als auch das partielle Indexprädikat, falls vorhanden. Das muss jedoch nicht immer so bleiben. `amgetbitmap` und `amgettuple` können nicht im selben Indexscan verwendet werden; bei Verwendung von `amgetbitmap` gelten außerdem weitere Einschränkungen, wie in [Abschnitt 63.3](63_Schnittstellendefinition_für_Index_Access_Methods.md#633-indexscans) erläutert.

Die Funktion `amgetbitmap` muss nur bereitgestellt werden, wenn die Zugriffsmethode »Bitmap«-Indexscans unterstützt. Wenn nicht, muss das Feld `amgetbitmap` in ihrer `IndexAmRoutine`-Struktur auf `NULL` gesetzt werden.

```c
void
amendscan(IndexScanDesc scan);
```

Beendet einen Scan und gibt Ressourcen frei. Die Scanstruktur selbst sollte nicht freigegeben werden, aber alle intern von der Zugriffsmethode erworbenen Locks oder Pins sowie jeder von `ambeginscan` und anderen scanbezogenen Funktionen allozierte Speicher müssen freigegeben werden.

```c
void
ammarkpos(IndexScanDesc scan);
```

Markiert die aktuelle Scanposition. Die Zugriffsmethode muss nur eine gemerkte Scanposition pro Scan unterstützen.

Die Funktion `ammarkpos` muss nur bereitgestellt werden, wenn die Zugriffsmethode geordnete Scans unterstützt. Wenn nicht, kann das Feld `ammarkpos` in ihrer `IndexAmRoutine`-Struktur auf `NULL` gesetzt werden.

```c
void
amrestrpos(IndexScanDesc scan);
```

Stellt den Scan auf die zuletzt markierte Position wieder her.

Die Funktion `amrestrpos` muss nur bereitgestellt werden, wenn die Zugriffsmethode geordnete Scans unterstützt. Wenn nicht, kann das Feld `amrestrpos` in ihrer `IndexAmRoutine`-Struktur auf `NULL` gesetzt werden.

Zusätzlich zur Unterstützung gewöhnlicher Indexscans möchten manche Indextypen parallele Indexscans unterstützen, bei denen mehrere Backends gemeinsam einen Indexscan ausführen. Die Index Access Method sollte dafür sorgen, dass jeder beteiligte Prozess eine Teilmenge der Tupel zurückgibt, die von einem gewöhnlichen, nichtparallelen Indexscan geliefert würden, und zwar so, dass die Vereinigung dieser Teilmengen der Menge der Tupel entspricht, die von einem gewöhnlichen, nichtparallelen Indexscan zurückgegeben würden. Außerdem muss zwar keine globale Ordnung der von einem parallelen Scan zurückgegebenen Tupel bestehen, aber die Ordnung der innerhalb jedes beteiligten Backends zurückgegebenen Teilmenge von Tupeln muss der angeforderten Ordnung entsprechen. Die folgenden Funktionen können implementiert werden, um parallele Indexscans zu unterstützen:

```c
Size
amestimateparallelscan(Relation indexRelation,
                       int nkeys,
                       int norderbys);
```

Schätzt die Anzahl von Bytes dynamischen Shared Memorys, die die Zugriffsmethode zur Ausführung eines parallelen Scans benötigt, und gibt sie zurück. Diese Zahl kommt zusätzlich zu dem Speicherplatz hinzu, der für AM-unabhängige Daten in `ParallelIndexScanDescData` benötigt wird.

Die Parameter `nkeys` und `norderbys` geben die Anzahl der Qualifikationen und Ordnungsoperatoren an, die im Scan verwendet werden; dieselben Werte werden an `amrescan` übergeben. Beachten Sie, dass die tatsächlichen Werte der Scan Keys noch nicht bereitgestellt werden.

Für Zugriffsmethoden, die keine parallelen Scans unterstützen oder für die die Anzahl zusätzlich benötigter Bytes Speicher null ist, muss diese Funktion nicht implementiert werden.

```c
void
aminitparallelscan(void *target);
```

Diese Funktion wird aufgerufen, um dynamischen Shared Memory zu Beginn eines parallelen Scans zu initialisieren. `target` zeigt auf mindestens so viele Bytes, wie zuvor von `amestimateparallelscan` zurückgegeben wurden; die Funktion kann diesen Speicherplatz verwenden, um beliebige gewünschte Daten zu speichern.

Für Zugriffsmethoden, die keine parallelen Scans unterstützen, oder in Fällen, in denen der benötigte Shared-Memory-Bereich keine Initialisierung benötigt, muss diese Funktion nicht implementiert werden.

```c
void
amparallelrescan(IndexScanDesc scan);
```

Diese Funktion wird, falls implementiert, aufgerufen, wenn ein paralleler Indexscan neu gestartet werden muss. Sie sollte jeden von `aminitparallelscan` eingerichteten gemeinsamen Zustand so zurücksetzen, dass der Scan von vorn neu gestartet wird.

```c
CompareType
amtranslatestrategy(StrategyNumber strategy, Oid opfamily, Oid opcintype);

StrategyNumber
amtranslatecmptype(CompareType cmptype, Oid opfamily, Oid opcintype);
```

Diese Funktionen werden, falls implementiert, vom Planner und Executor aufgerufen, um zwischen festen `CompareType`-Werten und den spezifischen Strategienummern zu übersetzen, die von der Zugriffsmethode verwendet werden. Sie können von Zugriffsmethoden implementiert werden, die eine Funktionalität ähnlich den eingebauten Zugriffsmethoden `btree` oder `hash` bereitstellen. Durch diese Übersetzungen kann das System die Semantik der Operationen der Zugriffsmethode verstehen und sie an verschiedenen Stellen anstelle von `btree`- oder `hash`-Indizes verwenden. Wenn die Funktionalität der Zugriffsmethode diesen eingebauten Zugriffsmethoden nicht ähnelt, müssen diese Funktionen nicht implementiert werden. Sind sie nicht implementiert, wird die Zugriffsmethode für bestimmte Planner- und Executor-Entscheidungen ignoriert, ist aber ansonsten vollständig funktionsfähig.

## 63.3. Indexscans

Bei einem Indexscan ist die Index Access Method dafür verantwortlich, die TIDs aller Tupel zurückzugeben, die ihr bekannt sind und zu den Scan Keys passen. Die Zugriffsmethode ist nicht daran beteiligt, diese Tupel tatsächlich aus der übergeordneten Tabelle des Index zu holen oder zu bestimmen, ob sie den Sichtbarkeitstest des Scans oder andere Bedingungen erfüllen.

Ein Scan Key ist die interne Darstellung einer `WHERE`-Klausel der Form `index_key operator constant`, wobei der Indexschlüssel eine der Spalten des Index ist und der Operator eines der Mitglieder der Operatorfamilie ist, die dieser Indexspalte zugeordnet ist. Ein Indexscan hat null oder mehr Scan Keys, die implizit mit `AND` verknüpft werden. Von den zurückgegebenen Tupeln wird erwartet, dass sie alle angegebenen Bedingungen erfüllen.

Die Zugriffsmethode kann melden, dass der Index für eine bestimmte Abfrage lossy ist oder Nachprüfungen erfordert. Das bedeutet, dass der Indexscan alle Einträge zurückgibt, die den Scan Key passieren, und möglicherweise zusätzliche Einträge, die dies nicht tun. Die Indexscan-Maschinerie des Kernsystems wendet dann die Indexbedingungen erneut auf das Heap-Tupel an, um zu prüfen, ob es wirklich ausgewählt werden sollte. Wenn die Recheck-Option nicht angegeben ist, muss der Indexscan genau die Menge der passenden Einträge zurückgeben.

Beachten Sie, dass es vollständig der Zugriffsmethode überlassen bleibt, sicherzustellen, dass sie genau alle Einträge findet, die alle angegebenen Scan Keys erfüllen, und keine anderen. Außerdem übergibt das Kernsystem einfach alle `WHERE`-Klauseln, die zu Indexschlüsseln und Operatorfamilien passen, ohne semantische Analyse darauf, ob sie redundant oder widersprüchlich sind. Bei `WHERE x > 4 AND x > 14`, wobei `x` eine B-Tree-indizierte Spalte ist, bleibt es zum Beispiel der B-Tree-Funktion `amrescan` überlassen zu erkennen, dass der erste Scan Key redundant ist und verworfen werden kann. Der Umfang der während `amrescan` nötigen Vorverarbeitung hängt davon ab, in welchem Maß die Index Access Method die Scan Keys auf eine »normalisierte« Form reduzieren muss.

Manche Zugriffsmethoden geben Indexeinträge in einer wohldefinierten Reihenfolge zurück, andere nicht. Es gibt tatsächlich zwei verschiedene Arten, wie eine Zugriffsmethode sortierte Ausgabe unterstützen kann:

- Zugriffsmethoden, die Einträge immer in der natürlichen Ordnung ihrer Daten zurückgeben, wie `btree`, sollten `amcanorder` auf `true` setzen. Derzeit müssen solche Zugriffsmethoden für ihre Gleichheits- und Ordnungsoperatoren `btree`-kompatible Strategienummern verwenden.

- Zugriffsmethoden, die Ordnungsoperatoren unterstützen, sollten `amcanorderbyop` auf `true` setzen. Dies zeigt an, dass der Index Einträge in einer Ordnung zurückgeben kann, die `ORDER BY index_key operator constant` erfüllt. Scan-Modifikatoren dieser Form können wie zuvor beschrieben an `amrescan` übergeben werden.

Die Funktion `amgettuple` hat ein Richtungsargument, das entweder `ForwardScanDirection`, der Normalfall, oder `BackwardScanDirection` sein kann. Wenn der erste Aufruf nach `amrescan` `BackwardScanDirection` angibt, soll die Menge der passenden Indexeinträge von hinten nach vorn statt in der normalen Richtung von vorn nach hinten gescannt werden. `amgettuple` muss dann das letzte passende Tupel im Index zurückgeben, nicht wie üblich das erste. Dies tritt nur bei Zugriffsmethoden auf, die `amcanorder` auf `true` setzen. Nach dem ersten Aufruf muss `amgettuple` darauf vorbereitet sein, den Scan von dem zuletzt zurückgegebenen Eintrag aus in beide Richtungen fortzusetzen. Wenn `amcanbackward` jedoch `false` ist, haben alle nachfolgenden Aufrufe dieselbe Richtung wie der erste.

Zugriffsmethoden, die geordnete Scans unterstützen, müssen das »Markieren« einer Position in einem Scan und die spätere Rückkehr zu dieser markierten Position unterstützen. Dieselbe Position kann mehrfach wiederhergestellt werden. Es muss jedoch nur eine Position pro Scan gemerkt werden; ein neuer Aufruf von `ammarkpos` überschreibt die zuvor markierte Position. Eine Zugriffsmethode, die keine geordneten Scans unterstützt, muss in `IndexAmRoutine` keine Funktionen `ammarkpos` und `amrestrpos` bereitstellen; setzen Sie diese Zeiger stattdessen auf `NULL`.

Sowohl die Scanposition als auch die markierte Position, falls vorhanden, müssen bei gleichzeitigen Einfügungen oder Löschungen im Index konsistent gehalten werden. Es ist in Ordnung, wenn ein frisch eingefügter Eintrag nicht von einem Scan zurückgegeben wird, der ihn gefunden hätte, wenn er beim Start des Scans existiert hätte. Ebenso ist es in Ordnung, wenn ein Scan einen solchen Eintrag bei einem Rescan oder beim Zurückgehen zurückgibt, obwohl er ihn beim ersten Durchlauf nicht zurückgegeben hatte. Entsprechend kann eine gleichzeitige Löschung in den Ergebnissen eines Scans sichtbar sein oder nicht. Wichtig ist, dass Einfügungen oder Löschungen nicht dazu führen, dass der Scan Einträge übersieht oder mehrfach zurückgibt, die nicht selbst eingefügt oder gelöscht wurden.

Wenn der Index die ursprünglichen indizierten Datenwerte speichert und nicht irgendeine lossy Darstellung davon, ist es nützlich, Index-Only-Scans zu unterstützen, bei denen der Index die tatsächlichen Daten und nicht nur die TID des Heap-Tupels zurückgibt. Das vermeidet I/O nur, wenn die Visibility Map zeigt, dass die TID auf einer All-Visible-Seite liegt; andernfalls muss das Heap-Tupel ohnehin besucht werden, um die MVCC-Sichtbarkeit zu prüfen. Das ist jedoch nicht Sache der Zugriffsmethode.

Statt `amgettuple` zu verwenden, kann ein Indexscan mit `amgetbitmap` ausgeführt werden, um alle Tupel in einem Aufruf zu holen. Das kann spürbar effizienter sein als `amgettuple`, weil es Lock/Unlock-Zyklen innerhalb der Zugriffsmethode vermeiden kann. Im Prinzip sollte `amgetbitmap` dieselben Auswirkungen haben wie wiederholte Aufrufe von `amgettuple`; wir erzwingen jedoch mehrere Einschränkungen, um die Dinge zu vereinfachen. Zunächst gibt `amgetbitmap` alle Tupel auf einmal zurück, und das Markieren oder Wiederherstellen von Scanpositionen wird nicht unterstützt. Zweitens werden die Tupel in einer Bitmap zurückgegeben, die keine bestimmte Ordnung besitzt; deshalb nimmt `amgetbitmap` kein Richtungsargument. Auch Ordnungsoperatoren werden für einen solchen Scan nie geliefert. Außerdem gibt es keine Unterstützung für Index-Only-Scans mit `amgetbitmap`, weil es keine Möglichkeit gibt, die Inhalte von Indextupeln zurückzugeben. Schließlich garantiert `amgetbitmap` keine Sperrung der zurückgegebenen Tupel; die Auswirkungen werden in [Abschnitt 63.4](63_Schnittstellendefinition_für_Index_Access_Methods.md#634-lockingüberlegungen-für-indizes) erläutert.

Beachten Sie, dass eine Zugriffsmethode nur `amgetbitmap` und nicht `amgettuple` implementieren darf oder umgekehrt, wenn ihre interne Implementierung für die eine oder andere API ungeeignet ist.

## 63.4. Locking-Überlegungen für Indizes

Index Access Methods müssen gleichzeitige Aktualisierungen des Index durch mehrere Prozesse handhaben. Das PostgreSQL-Kernsystem erwirbt während eines Indexscans `AccessShareLock` auf dem Index und beim Aktualisieren des Index, einschließlich einfachem `VACUUM`, `RowExclusiveLock`. Da diese Lock-Typen nicht miteinander kollidieren, ist die Zugriffsmethode dafür verantwortlich, jedes feingranulare Locking zu handhaben, das sie benötigt. Ein `ACCESS EXCLUSIVE`-Lock auf den gesamten Index wird nur während Indexerzeugung, Zerstörung oder `REINDEX` genommen; bei `CONCURRENTLY` wird stattdessen `SHARE UPDATE EXCLUSIVE` genommen.

Der Bau eines Indextyps, der gleichzeitige Aktualisierungen unterstützt, erfordert gewöhnlich eine umfangreiche und subtile Analyse des erforderlichen Verhaltens. Für B-Tree- und Hash-Indextypen können Sie die beteiligten Entwurfsentscheidungen in `src/backend/access/nbtree/README` und `src/backend/access/hash/README` nachlesen.

Neben den eigenen internen Konsistenzanforderungen des Index erzeugen gleichzeitige Aktualisierungen Fragen der Konsistenz zwischen der übergeordneten Tabelle, dem Heap, und dem Index. Da PostgreSQL Zugriffe und Aktualisierungen des Heaps von denen des Index trennt, gibt es Zeitfenster, in denen der Index mit dem Heap inkonsistent sein kann. Wir behandeln dieses Problem mit den folgenden Regeln:

- Ein neuer Heap-Eintrag wird erzeugt, bevor seine Indexeinträge erzeugt werden. Daher sieht ein gleichzeitiger Indexscan den Heap-Eintrag wahrscheinlich nicht. Das ist in Ordnung, weil der Indexleser an einer nicht festgeschriebenen Zeile ohnehin nicht interessiert wäre. Siehe aber [Abschnitt 63.5](63_Schnittstellendefinition_für_Index_Access_Methods.md#635-indexeindeutigkeitsprüfungen).

- Wenn ein Heap-Eintrag gelöscht werden soll, etwa durch `VACUUM`, müssen zuerst alle seine Indexeinträge entfernt werden.

- Ein Indexscan muss einen Pin auf der Indexseite halten, die das zuletzt von `amgettuple` zurückgegebene Item enthält, und `ambulkdelete` darf keine Einträge von Seiten löschen, die von anderen Backends gepinnt sind. Die Notwendigkeit dieser Regel wird unten erklärt.

Ohne die dritte Regel könnte ein Indexleser einen Indexeintrag genau vor dessen Entfernung durch `VACUUM` sehen und anschließend beim zugehörigen Heap-Eintrag ankommen, nachdem dieser bereits durch `VACUUM` entfernt wurde. Das erzeugt keine ernsthaften Probleme, wenn diese Itemnummer noch unbenutzt ist, wenn der Leser sie erreicht, da ein leerer Item-Slot von `heap_fetch()` ignoriert wird. Was aber, wenn ein drittes Backend den Item-Slot bereits für etwas anderes wiederverwendet hat? Bei Verwendung eines MVCC-konformen Snapshots gibt es kein Problem, weil der neue Beleger des Slots sicher zu neu ist, um den Snapshot-Test zu bestehen. Bei einem nicht-MVCC-konformen Snapshot wie `SnapshotAny` wäre es jedoch möglich, eine Zeile zu akzeptieren und zurückzugeben, die tatsächlich nicht zu den Scan Keys passt. Man könnte sich gegen dieses Szenario schützen, indem man verlangt, dass die Scan Keys in allen Fällen gegen die Heap-Zeile erneut geprüft werden, aber das ist zu teuer. Stattdessen verwenden wir einen Pin auf einer Indexseite als Stellvertreter dafür, dass der Leser möglicherweise noch »unterwegs« vom Indexeintrag zum passenden Heap-Eintrag ist. Wenn `ambulkdelete` auf einen solchen Pin blockiert, kann `VACUUM` den Heap-Eintrag nicht löschen, bevor der Leser mit ihm fertig ist. Diese Lösung kostet zur Laufzeit wenig und fügt Blockierungsaufwand nur in den seltenen Fällen hinzu, in denen tatsächlich ein Konflikt besteht.

Diese Lösung erfordert, dass Indexscans »synchron« sind: Wir müssen jedes Heap-Tupel unmittelbar nach dem Scannen des entsprechenden Indexeintrags holen. Das ist aus mehreren Gründen teuer. Ein »asynchroner« Scan, bei dem wir viele TIDs aus dem Index sammeln und die Heap-Tupel erst später besuchen, erfordert deutlich weniger Index-Locking-Aufwand und kann ein effizienteres Heap-Zugriffsmuster ermöglichen. Nach der obigen Analyse müssen wir für nicht-MVCC-konforme Snapshots den synchronen Ansatz verwenden, aber ein asynchroner Scan ist für eine Abfrage mit MVCC-Snapshot praktikabel.

Bei einem `amgetbitmap`-Indexscan hält die Zugriffsmethode keinen Index-Pin auf irgendeinem der zurückgegebenen Tupel. Daher ist es nur sicher, solche Scans mit MVCC-konformen Snapshots zu verwenden.

Wenn das Flag `ampredlocks` nicht gesetzt ist, erwirbt jeder Scan, der diese Index Access Method innerhalb einer serialisierbaren Transaktion verwendet, einen nichtblockierenden Predicate Lock auf dem gesamten Index. Dies erzeugt einen Read-Write-Konflikt mit dem Einfügen irgendeines Tupels in diesen Index durch eine gleichzeitige serialisierbare Transaktion. Wenn bestimmte Muster von Read-Write-Konflikten unter einer Menge gleichzeitiger serialisierbarer Transaktionen erkannt werden, kann eine dieser Transaktionen abgebrochen werden, um die Datenintegrität zu schützen. Wenn das Flag gesetzt ist, zeigt es an, dass die Index Access Method feiner granuliertes Predicate Locking implementiert, was die Häufigkeit solcher Transaktionsabbrüche tendenziell reduziert.

## 63.5. Index-Eindeutigkeitsprüfungen

PostgreSQL erzwingt SQL-Eindeutigkeits-Constraints mit eindeutigen Indizes, also Indizes, die mehrere Einträge mit identischen Schlüsseln nicht zulassen. Eine Zugriffsmethode, die dieses Feature unterstützt, setzt `amcanunique` auf `true`. Derzeit unterstützt nur B-Tree dies. Spalten, die in der `INCLUDE`-Klausel aufgeführt sind, werden beim Erzwingen der Eindeutigkeit nicht berücksichtigt.

Wegen MVCC ist es immer nötig zu erlauben, dass Duplikate physisch in einem Index existieren: Die Einträge können auf aufeinanderfolgende Versionen einer einzelnen logischen Zeile verweisen. Das Verhalten, das wir tatsächlich erzwingen wollen, lautet, dass kein MVCC-Snapshot zwei Zeilen mit gleichen Indexschlüsseln enthalten kann. Dies zerfällt in die folgenden Fälle, die beim Einfügen einer neuen Zeile in einen eindeutigen Index geprüft werden müssen:

- Wenn eine konfliktierende gültige Zeile von der aktuellen Transaktion gelöscht wurde, ist das in Ordnung. Insbesondere löscht ein `UPDATE` immer die alte Zeilenversion, bevor die neue Version eingefügt wird; dies erlaubt ein `UPDATE` auf einer Zeile, ohne den Schlüssel zu ändern.

- Wenn eine konfliktierende Zeile von einer noch nicht festgeschriebenen Transaktion eingefügt wurde, muss der einfügende Prozess abwarten, ob diese Transaktion festgeschrieben wird. Wenn sie zurückrollt, gibt es keinen Konflikt. Wenn sie festschreibt, ohne die konfliktierende Zeile wieder zu löschen, liegt eine Eindeutigkeitsverletzung vor. In der Praxis warten wir einfach, bis die andere Transaktion endet, und wiederholen dann die Sichtbarkeitsprüfung vollständig.

- Entsprechend muss der einfügende Prozess, wenn eine konfliktierende gültige Zeile von einer noch nicht festgeschriebenen Transaktion gelöscht wurde, warten, bis diese Transaktion festschreibt oder abbricht, und dann den Test wiederholen.

Außerdem muss die Zugriffsmethode unmittelbar vor dem Melden einer Eindeutigkeitsverletzung nach den obigen Regeln die Lebendigkeit der einzufügenden Zeile erneut prüfen. Wenn sie festgeschrieben tot ist, sollte keine Verletzung gemeldet werden. Dieser Fall kann beim gewöhnlichen Szenario des Einfügens einer gerade von der aktuellen Transaktion erzeugten Zeile nicht auftreten. Er kann jedoch während `CREATE UNIQUE INDEX CONCURRENTLY` auftreten.

Wir verlangen von der Index Access Method, diese Tests selbst anzuwenden. Das bedeutet, dass sie in den Heap greifen muss, um den Commit-Status jeder Zeile zu prüfen, die nach dem Indexinhalt einen doppelten Schlüssel zu haben scheint. Das ist ohne Zweifel unschön und nicht modular, spart aber redundante Arbeit: Wenn wir eine separate Sondierung durchführten, würde die Indexsuche nach einer konfliktierenden Zeile im Wesentlichen wiederholt, während die Stelle zum Einfügen des neuen Indexeintrags gesucht wird. Außerdem gibt es keinen offensichtlichen Weg, Race Conditions zu vermeiden, wenn die Konfliktprüfung nicht integraler Bestandteil des Einfügens des neuen Indexeintrags ist.

Wenn das Unique Constraint deferrable ist, gibt es zusätzliche Komplexität: Wir müssen einen Indexeintrag für eine neue Zeile einfügen können, die Meldung einer Eindeutigkeitsverletzung aber bis zum Ende der Anweisung oder noch später zurückstellen. Um unnötige wiederholte Suchläufe im Index zu vermeiden, sollte die Index Access Method während des anfänglichen Einfügens eine vorläufige Eindeutigkeitsprüfung durchführen. Wenn diese zeigt, dass sicher kein konfliktierendes lebendes Tupel existiert, sind wir fertig. Andernfalls planen wir eine erneute Prüfung für den Zeitpunkt ein, zu dem das Constraint erzwungen werden soll. Wenn zum Zeitpunkt der erneuten Prüfung sowohl das eingefügte Tupel als auch ein anderes Tupel mit demselben Schlüssel leben, muss der Fehler gemeldet werden. Beachten Sie, dass »lebend« für diesen Zweck tatsächlich bedeutet: irgendein Tupel in der HOT-Kette des Indexeintrags ist lebend. Zur Implementierung erhält die Funktion `aminsert` einen Parameter `checkUnique` mit einem der folgenden Werte:

- `UNIQUE_CHECK_NO`
  - Gibt an, dass keine Eindeutigkeitsprüfung durchgeführt werden soll; dies ist kein eindeutiger Index.

- `UNIQUE_CHECK_YES`
  - Gibt an, dass dies ein nicht-deferrable eindeutiger Index ist und die Eindeutigkeitsprüfung sofort wie oben beschrieben durchgeführt werden muss.

- `UNIQUE_CHECK_PARTIAL`
  - Gibt an, dass das Unique Constraint deferrable ist. PostgreSQL verwendet diesen Modus, um den Indexeintrag jeder Zeile einzufügen. Die Zugriffsmethode muss doppelte Einträge im Index zulassen und mögliche Duplikate melden, indem sie von `aminsert` `false` zurückgibt. Für jede Zeile, für die `false` zurückgegeben wird, wird eine zurückgestellte erneute Prüfung eingeplant.

    Die Zugriffsmethode muss alle Zeilen identifizieren, die das Unique Constraint verletzen könnten, aber es ist kein Fehler, wenn sie False Positives meldet. Dadurch kann die Prüfung durchgeführt werden, ohne auf das Ende anderer Transaktionen zu warten. Hier gemeldete Konflikte werden nicht als Fehler behandelt und später erneut geprüft; bis dahin können sie keine Konflikte mehr sein.

- `UNIQUE_CHECK_EXISTING`
  - Gibt an, dass dies eine zurückgestellte erneute Prüfung einer Zeile ist, die als mögliche Eindeutigkeitsverletzung gemeldet wurde. Obwohl dies durch Aufruf von `aminsert` implementiert ist, darf die Zugriffsmethode in diesem Fall keinen neuen Indexeintrag einfügen. Der Indexeintrag ist bereits vorhanden. Stattdessen muss die Zugriffsmethode prüfen, ob es einen anderen lebenden Indexeintrag gibt. Wenn ja und wenn die Zielzeile ebenfalls noch lebt, muss ein Fehler gemeldet werden.

    Es wird empfohlen, dass die Zugriffsmethode bei einem Aufruf mit `UNIQUE_CHECK_EXISTING` zusätzlich prüft, ob die Zielzeile tatsächlich einen vorhandenen Eintrag im Index hat, und einen Fehler meldet, falls nicht. Das ist sinnvoll, weil die an `aminsert` übergebenen Index-Tupelwerte neu berechnet wurden. Wenn die Indexdefinition Funktionen enthält, die nicht wirklich immutable sind, könnten wir den falschen Bereich des Index prüfen. Die Prüfung, dass die Zielzeile bei der erneuten Prüfung gefunden wird, bestätigt, dass wir nach denselben Tupelwerten scannen, die beim ursprünglichen Einfügen verwendet wurden.

## 63.6. Funktionen zur Indexkostenschätzung

Die Funktion `amcostestimate` erhält Informationen, die einen möglichen Indexscan beschreiben, einschließlich Listen von `WHERE`- und `ORDER BY`-Klauseln, die als mit dem Index verwendbar bestimmt wurden. Sie muss Schätzungen der Kosten des Indexzugriffs und der Selektivität der `WHERE`-Klauseln zurückgeben, also des Anteils der Zeilen der übergeordneten Tabelle, die während des Indexscans geholt werden. In einfachen Fällen kann fast die gesamte Arbeit des Kostenschätzers durch Aufruf von Standardroutinen im Optimizer erledigt werden. Der Zweck einer `amcostestimate`-Funktion besteht darin, Index Access Methods indexspezifisches Wissen bereitstellen zu lassen, falls es möglich ist, die Standardschätzungen zu verbessern.

Jede Funktion `amcostestimate` muss diese Signatur haben:

```c
void
amcostestimate(PlannerInfo *root,
               IndexPath *path,
               double loop_count,
               Cost *indexStartupCost,
               Cost *indexTotalCost,
               Selectivity *indexSelectivity,
               double *indexCorrelation,
               double *indexPages);
```

Die ersten drei Parameter sind Eingaben:

- `root`
  - Die Informationen des Planners über die verarbeitete Abfrage.

- `path`
  - Der betrachtete Indexzugriffspfad. Alle Felder außer Kosten- und Selektivitätswerten sind gültig.

- `loop_count`
  - Die Anzahl der Wiederholungen des Indexscans, die in die Kostenschätzungen einbezogen werden soll. Dieser Wert ist typischerweise größer als eins, wenn ein parametrisierter Scan für die Verwendung im Inneren eines Nested-Loop-Joins betrachtet wird. Beachten Sie, dass sich die Kostenschätzungen dennoch nur auf einen Scan beziehen sollten; ein größeres `loop_count` bedeutet, dass es angemessen sein kann, Cache-Effekte über mehrere Scans hinweg zu berücksichtigen.

Die letzten fünf Parameter sind Ausgaben per Referenz:

- `*indexStartupCost`
  - Auf die Kosten der Startverarbeitung des Index setzen.

- `*indexTotalCost`
  - Auf die Gesamtkosten der Indexverarbeitung setzen.

- `*indexSelectivity`
  - Auf die Indexselektivität setzen.

- `*indexCorrelation`
  - Auf den Korrelationskoeffizienten zwischen Indexscan-Reihenfolge und Reihenfolge der zugrunde liegenden Tabelle setzen.

- `*indexPages`
  - Auf die Anzahl der Leaf-Seiten des Index setzen.

Beachten Sie, dass Kostenschätzungsfunktionen in C geschrieben werden müssen, nicht in SQL oder einer verfügbaren prozeduralen Sprache, weil sie auf interne Datenstrukturen des Planners/Optimizers zugreifen müssen.

Die Indexzugriffskosten sollten mit den Parametern berechnet werden, die von `src/backend/optimizer/path/costsize.c` verwendet werden: Ein sequenzieller Plattenblockzugriff hat Kosten `seq_page_cost`, ein nichtsequenzieller Zugriff hat Kosten `random_page_cost`, und die Kosten für die Verarbeitung einer Indexzeile sollten normalerweise als `cpu_index_tuple_cost` angesetzt werden. Zusätzlich sollte für alle während der Indexverarbeitung aufgerufenen Vergleichsoperatoren, besonders für die Auswertung der `indexquals` selbst, ein passendes Vielfaches von `cpu_operator_cost` berechnet werden.

Die Zugriffskosten sollten alle Platten- und CPU-Kosten enthalten, die mit dem Scannen des Index selbst verbunden sind, aber nicht die Kosten für das Holen oder Verarbeiten der Zeilen der übergeordneten Tabelle, die durch den Index identifiziert werden.

Die »Startkosten« sind der Teil der gesamten Scankosten, der aufgewendet werden muss, bevor die erste Zeile geholt werden kann. Für die meisten Indizes kann dieser Wert als null angenommen werden, aber ein Indextyp mit hohen Startkosten möchte ihn möglicherweise auf einen Nicht-Null-Wert setzen.

`indexSelectivity` sollte auf den geschätzten Anteil der Zeilen der übergeordneten Tabelle gesetzt werden, die während des Indexscans geholt werden. Bei einer lossy Abfrage ist dieser Wert typischerweise höher als der Anteil der Zeilen, die die angegebenen Qualifikationsbedingungen tatsächlich erfüllen.

`indexCorrelation` sollte auf die Korrelation zwischen Indexordnung und Tabellenordnung gesetzt werden, im Bereich von `-1.0` bis `1.0`. Dieser Wert wird verwendet, um die Schätzung der Kosten für das Holen von Zeilen aus der übergeordneten Tabelle anzupassen.

`indexPages` sollte auf die Anzahl der Leaf-Seiten gesetzt werden. Dieser Wert wird verwendet, um die Anzahl der Worker für parallele Indexscans zu schätzen.

Wenn `loop_count` größer als eins ist, sollten die zurückgegebenen Zahlen Durchschnittswerte sein, die für einen beliebigen einzelnen Scan des Index erwartet werden.

Eine typische Kostenschätzungsfunktion geht wie folgt vor:

1. Schätzen Sie den Anteil der Zeilen der übergeordneten Tabelle, die aufgrund der angegebenen Qualifikationsbedingungen besucht werden, und geben Sie ihn zurück. Wenn kein indextypspezifisches Wissen vorliegt, verwenden Sie die Standard-Optimizer-Funktion `clauselist_selectivity()`:

   ```c
   *indexSelectivity = clauselist_selectivity(root, path->indexquals,
                                              path->indexinfo->rel->relid,
                                              JOIN_INNER, NULL);
   ```

2. Schätzen Sie die Anzahl der Indexzeilen, die während des Scans besucht werden. Für viele Indextypen ist dies dasselbe wie `indexSelectivity` multipliziert mit der Anzahl der Zeilen im Index, es kann aber auch mehr sein. Beachten Sie, dass Größe und Zeilenzahl des Index über die Struktur `path->indexinfo` verfügbar sind.

3. Schätzen Sie die Anzahl der Indexseiten, die während des Scans geholt werden. Das kann einfach `indexSelectivity` multipliziert mit der Größe des Index in Seiten sein.

4. Berechnen Sie die Indexzugriffskosten. Ein generischer Schätzer könnte dies tun:

   ```c
   /*
    * Our generic assumption is that the index pages will be read
    * sequentially, so they cost seq_page_cost each, not
    * random_page_cost.
    * Also, we charge for evaluation of the indexquals at each index row.
    * All the costs are assumed to be paid incrementally during the scan.
    */
   cost_qual_eval(&index_qual_cost, path->indexquals, root);

   *indexStartupCost = index_qual_cost.startup;
   *indexTotalCost = seq_page_cost * numIndexPages +
       (cpu_index_tuple_cost + index_qual_cost.per_tuple) *
       numIndexTuples;
   ```

   Allerdings berücksichtigt das obige Beispiel keine Amortisierung von Indexlesevorgängen über wiederholte Indexscans hinweg.

5. Schätzen Sie die Indexkorrelation. Für einen einfachen geordneten Index auf einem einzelnen Feld kann diese aus `pg_statistic` gelesen werden. Wenn die Korrelation nicht bekannt ist, ist die konservative Schätzung null, also keine Korrelation.

Beispiele für Kostenschätzungsfunktionen befinden sich in `src/backend/utils/adt/selfuncs.c`.
