# 64. Write-Ahead-Logging für Erweiterungen

Bestimmte Erweiterungen, vor allem Erweiterungen, die eigene Zugriffsmethoden implementieren, müssen möglicherweise Write-Ahead-Logging durchführen, um Crash-Sicherheit zu gewährleisten. PostgreSQL stellt Erweiterungen zwei Wege bereit, dieses Ziel zu erreichen.

Erstens können Erweiterungen generisches WAL verwenden. Dabei handelt es sich um einen speziellen Typ von WAL-Record, der Änderungen an Seiten auf generische Weise beschreibt. Diese Methode ist einfach zu implementieren und erfordert nicht, dass eine Erweiterungsbibliothek geladen ist, um die Records anzuwenden. Generische WAL-Records werden bei der logischen Decodierung jedoch ignoriert.

Zweitens können Erweiterungen einen Custom Resource Manager verwenden. Diese Methode ist flexibler, unterstützt logische Decodierung und kann manchmal deutlich kleinere Write-Ahead-Log-Records erzeugen, als es mit generischem WAL möglich wäre. Für eine Erweiterung ist sie jedoch komplexer zu implementieren.

## 64.1. Generische WAL-Records

Obwohl alle eingebauten WAL-gelogten Module eigene Typen von WAL-Records besitzen, gibt es auch einen generischen WAL-Record-Typ, der Änderungen an Seiten auf generische Weise beschreibt.

> Generische WAL-Records werden während der logischen Decodierung ignoriert. Wenn Ihre Erweiterung logische Decodierung benötigt, sollten Sie einen Custom WAL Resource Manager in Betracht ziehen.

Die API zum Konstruieren generischer WAL-Records ist in `access/generic_xlog.h` definiert und in `access/transam/generic_xlog.c` implementiert.

Um eine WAL-geloggte Datenaktualisierung mit der Einrichtung für generische WAL-Records durchzuführen, gehen Sie wie folgt vor:

1. `state = GenericXLogStart(relation)` startet die Konstruktion eines generischen WAL-Records für die angegebene Relation.

2. `page = GenericXLogRegisterBuffer(state, buffer, flags)` registriert einen Buffer, der innerhalb des aktuellen generischen WAL-Records geändert werden soll. Diese Funktion gibt einen Zeiger auf eine temporäre Kopie der Buffer-Seite zurück, in der die Änderungen vorgenommen werden sollen. Ändern Sie den Inhalt des Buffers nicht direkt. Das dritte Argument ist eine Bitmaske von Flags, die auf die Operation anwendbar sind. Derzeit ist das einzige solche Flag `GENERIC_XLOG_FULL_IMAGE`; es gibt an, dass ein vollständiges Seitenabbild statt einer Delta-Aktualisierung in den WAL-Record aufgenommen werden soll. Typischerweise wird dieses Flag gesetzt, wenn die Seite neu ist oder vollständig neu geschrieben wurde. `GenericXLogRegisterBuffer` kann wiederholt werden, wenn die WAL-geloggte Aktion mehrere Seiten ändern muss.

3. Wenden Sie die Änderungen auf die Seitenabbilder an, die im vorherigen Schritt erhalten wurden.

4. `GenericXLogFinish(state)` wendet die Änderungen auf die Buffer an und schreibt den generischen WAL-Record.

Die Konstruktion eines WAL-Records kann zwischen beliebigen der obigen Schritte abgebrochen werden, indem `GenericXLogAbort(state)` aufgerufen wird. Dadurch werden alle Änderungen an den Kopien der Seitenabbilder verworfen.

Beachten Sie bei der Verwendung der Einrichtung für generische WAL-Records die folgenden Punkte:

- Direkte Änderungen an Buffern sind nicht erlaubt. Alle Änderungen müssen in Kopien erfolgen, die von `GenericXLogRegisterBuffer()` erhalten wurden. Anders gesagt: Code, der generische WAL-Records erzeugt, sollte niemals selbst `BufferGetPage()` aufrufen. Es bleibt jedoch Verantwortung des Aufrufers, die Buffer zu geeigneten Zeitpunkten zu pinnen, zu entpinnen, zu sperren und zu entsperren. Auf jedem Zielbuffer muss von vor `GenericXLogRegisterBuffer()` bis nach `GenericXLogFinish()` ein exklusiver Lock gehalten werden.

- Das Registrieren von Buffern (Schritt 2) und das Ändern von Seitenabbildern (Schritt 3) können frei gemischt werden; beide Schritte dürfen also in beliebiger Reihenfolge wiederholt werden. Denken Sie daran, dass Buffer in derselben Reihenfolge registriert werden sollten, in der beim Replay Locks auf ihnen erworben werden sollen.

- Die maximale Anzahl von Buffern, die für einen generischen WAL-Record registriert werden können, ist `MAX_GENERIC_XLOG_PAGES`. Wird diese Grenze überschritten, wird ein Fehler geworfen.

- Generisches WAL nimmt an, dass die zu ändernden Seiten das Standardlayout besitzen und insbesondere zwischen `pd_lower` und `pd_upper` keine nützlichen Daten stehen.

- Da Sie Kopien von Buffer-Seiten ändern, startet `GenericXLogStart()` keinen kritischen Abschnitt. Daher können Sie zwischen `GenericXLogStart()` und `GenericXLogFinish()` sicher Speicher allozieren, Fehler werfen und so weiter. Der einzige tatsächliche kritische Abschnitt befindet sich innerhalb von `GenericXLogFinish()`. Auch um einen Aufruf von `GenericXLogAbort()` während eines Fehlerausstiegs müssen Sie sich nicht kümmern.

- `GenericXLogFinish()` kümmert sich darum, Buffer als dirty zu markieren und ihre LSNs zu setzen. Sie müssen dies nicht ausdrücklich tun.

- Für ungeloggte Relationen funktioniert alles gleich, außer dass kein tatsächlicher WAL-Record geschrieben wird. Daher müssen Sie typischerweise keine ausdrücklichen Prüfungen auf ungeloggte Relationen durchführen.

- Die Redo-Funktion für generisches WAL erwirbt exklusive Locks auf Buffern in derselben Reihenfolge, in der sie registriert wurden. Nachdem alle Änderungen erneut ausgeführt wurden, werden die Locks in derselben Reihenfolge freigegeben.

- Wenn `GENERIC_XLOG_FULL_IMAGE` für einen registrierten Buffer nicht angegeben ist, enthält der generische WAL-Record ein Delta zwischen dem alten und dem neuen Seitenabbild. Dieses Delta basiert auf einem byteweisen Vergleich. Für den Fall, dass Daten innerhalb einer Seite verschoben werden, ist das nicht sehr kompakt und könnte künftig verbessert werden.

## 64.2. Custom WAL Resource Managers

Dieser Abschnitt erklärt die Schnittstelle zwischen dem PostgreSQL-Kernsystem und Custom WAL Resource Managers, die Erweiterungen eine direkte Integration mit dem WAL ermöglichen.

Eine Erweiterung, insbesondere eine Table Access Method oder Index Access Method, muss WAL möglicherweise für Recovery, Replikation und/oder logische Decodierung verwenden.

Um einen neuen Custom WAL Resource Manager zu erstellen, definieren Sie zunächst eine `RmgrData`-Struktur mit Implementierungen für die Methoden des Resource Managers. Siehe `src/backend/access/transam/README` und `src/include/access/xlog_internal.h` im PostgreSQL-Quellcode.

```c
/*
 * Method table for resource managers.
 *
 * This struct must be kept in sync with the PG_RMGR definition in
 * rmgr.c.
 *
 * rm_identify must return a name for the record based on xl_info
 * (without reference to the rmid). For example, XLOG_BTREE_VACUUM
 * would be named "VACUUM". rm_desc can then be called to obtain
 * additional detail for the record, if available (e.g. the last block).
 *
 * rm_mask takes as input a page modified by the resource manager and
 * masks out bits that shouldn't be flagged by wal_consistency_checking.
 *
 * RmgrTable[] is indexed by RmgrId values (see rmgrlist.h). If rm_name
 * is NULL, the corresponding RmgrTable entry is considered invalid.
 */
typedef struct RmgrData
{
    const char *rm_name;
    void        (*rm_redo) (XLogReaderState *record);
    void        (*rm_desc) (StringInfo buf, XLogReaderState *record);
    const char *(*rm_identify) (uint8 info);
    void        (*rm_startup) (void);
    void        (*rm_cleanup) (void);
    void        (*rm_mask) (char *pagedata, BlockNumber blkno);
    void        (*rm_decode) (struct LogicalDecodingContext *ctx,
                              struct XLogRecordBuffer *buf);
} RmgrData;
```

Das Modul `src/test/modules/test_custom_rmgrs` enthält ein funktionsfähiges Beispiel, das die Verwendung von Custom WAL Resource Managers demonstriert.

Registrieren Sie anschließend Ihren neuen Resource Manager.

```c
/*
 * Register a new custom WAL resource manager.
 *
 * Resource manager IDs must be globally unique across all extensions.
 * Refer to https://wiki.postgresql.org/wiki/CustomWALResourceManagers
 * to reserve a unique RmgrId for your extension, to avoid conflicts
 * with other extension developers. During development, use
 * RM_EXPERIMENTAL_ID to avoid needlessly reserving a new ID.
 */
extern void RegisterCustomRmgr(RmgrId rmid, const RmgrData *rmgr);
```

`RegisterCustomRmgr` muss aus der Funktion `_PG_init` des Erweiterungsmoduls aufgerufen werden. Während der Entwicklung einer neuen Erweiterung verwenden Sie `RM_EXPERIMENTAL_ID` für `rmid`. Wenn Sie bereit sind, die Erweiterung für Benutzer freizugeben, reservieren Sie eine neue Resource-Manager-ID auf der Seite Custom WAL Resource Manager.

Platzieren Sie das Erweiterungsmodul, das den Custom Resource Manager implementiert, in `shared_preload_libraries`, damit es früh während des PostgreSQL-Starts geladen wird.

> Die Erweiterung muss in `shared_preload_libraries` bleiben, solange im System Custom-WAL-Records existieren können. Andernfalls kann PostgreSQL die Custom-WAL-Records nicht anwenden oder decodieren, was den Start des Servers verhindern kann.

```text
https://wiki.postgresql.org/wiki/CustomWALResourceManagers
```
