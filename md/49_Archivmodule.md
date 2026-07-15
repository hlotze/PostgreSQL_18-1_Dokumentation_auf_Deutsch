# 49. Archivmodule

PostgreSQL stellt Infrastruktur bereit, um eigene Module für kontinuierliche Archivierung zu erstellen (siehe [Abschnitt 25.3](25_Sicherung_und_Wiederherstellung.md#253-kontinuierliche-archivierung-und-pointintimerecovery-pitr)). Obwohl die Archivierung über einen Shell-Befehl (d. h. `archive_command`) viel einfacher ist, ist ein eigenes Archivmodul oft deutlich robuster und performanter.

Wenn eine eigene `archive_library` konfiguriert ist, übergibt PostgreSQL abgeschlossene WAL-Dateien an das Modul, und der Server vermeidet es, diese WAL-Dateien wiederzuverwenden oder zu entfernen, bis das Modul anzeigt, dass die Dateien erfolgreich archiviert wurden. Letztlich entscheidet das Modul, was mit jeder WAL-Datei geschieht; viele Empfehlungen sind jedoch in [Abschnitt 25.3.1](25_Sicherung_und_Wiederherstellung.md#2531-walarchivierung-einrichten) aufgeführt.

Archivierungsmodule müssen mindestens aus einer Initialisierungsfunktion (siehe [Abschnitt 49.1](49_Archivmodule.md#491-initialisierungsfunktionen)) und den erforderlichen Callbacks (siehe [Abschnitt 49.2](49_Archivmodule.md#492-archivmodulcallbacks)) bestehen. Archivmodule dürfen jedoch auch deutlich mehr tun, zum Beispiel GUCs deklarieren und Background Worker registrieren.

Das Modul `contrib/basic_archive` enthält ein funktionsfähiges Beispiel, das einige nützliche Techniken demonstriert.

## 49.1. Initialisierungsfunktionen

Eine Archivbibliothek wird geladen, indem eine Shared Library dynamisch geladen wird, deren Basisname dem Namen von `archive_library` entspricht. Zum Auffinden der Bibliothek wird der normale Bibliothekssuchpfad verwendet. Um die erforderlichen Archivmodul-Callbacks bereitzustellen und anzuzeigen, dass die Bibliothek tatsächlich ein Archivmodul ist, muss sie eine Funktion mit dem Namen `_PG_archive_module_init` bereitstellen. Das Ergebnis der Funktion muss ein Zeiger auf eine Struktur vom Typ `ArchiveModuleCallbacks` sein, die alles enthält, was der Core-Code wissen muss, um das Archivmodul zu verwenden. Der Rückgabewert muss Server-Lebensdauer haben; typischerweise wird dies erreicht, indem er als `static const`-Variable im globalen Gültigkeitsbereich definiert wird.

```c
typedef struct ArchiveModuleCallbacks
{
    ArchiveStartupCB startup_cb;
    ArchiveCheckConfiguredCB check_configured_cb;
    ArchiveFileCB archive_file_cb;
    ArchiveShutdownCB shutdown_cb;
} ArchiveModuleCallbacks;
typedef const ArchiveModuleCallbacks *(*ArchiveModuleInit) (void);
```

Nur der Callback `archive_file_cb` ist erforderlich. Die anderen sind optional.

## 49.2. Archivmodul-Callbacks

Die Archiv-Callbacks definieren das eigentliche Archivierungsverhalten des Moduls. Der Server ruft sie nach Bedarf auf, um jede einzelne WAL-Datei zu verarbeiten.

### 49.2.1. Startup-Callback

Der Callback `startup_cb` wird kurz nach dem Laden des Moduls aufgerufen. Dieser Callback kann verwendet werden, um zusätzlich erforderliche Initialisierung auszuführen. Wenn das Archivmodul eigenen Zustand hat, kann es `state->private_data` verwenden, um ihn zu speichern.

```c
typedef void (*ArchiveStartupCB) (ArchiveModuleState *state);
```

### 49.2.2. Check-Callback

Der Callback `check_configured_cb` wird aufgerufen, um festzustellen, ob das Modul vollständig konfiguriert und bereit ist, WAL-Dateien anzunehmen (zum Beispiel ob seine Konfigurationsparameter auf gültige Werte gesetzt sind). Wenn kein `check_configured_cb` definiert ist, nimmt der Server immer an, dass das Modul konfiguriert ist.

```c
typedef bool (*ArchiveCheckConfiguredCB) (ArchiveModuleState *state);
```

Wenn `true` zurückgegeben wird, fährt der Server mit der Archivierung der Datei fort, indem er den Callback `archive_file_cb` aufruft. Wenn `false` zurückgegeben wird, wird die Archivierung nicht fortgesetzt, und der Archiver gibt die folgende Meldung in das Serverlog aus:

```text
WARNING:       archive_mode enabled, yet archiving is not configured
```

Im letzteren Fall ruft der Server diese Funktion periodisch auf, und die Archivierung wird erst fortgesetzt, wenn sie `true` zurückgibt.

> Wenn `false` zurückgegeben wird, kann es nützlich sein, der generischen Warnmeldung zusätzliche Informationen anzuhängen. Stellen Sie dazu vor der Rückgabe von `false` eine Meldung für das Makro `arch_module_check_errdetail` bereit. Wie `errdetail()` akzeptiert dieses Makro einen Formatstring, gefolgt von einer optionalen Liste von Argumenten. Die resultierende Zeichenkette wird als `DETAIL`-Zeile der Warnmeldung ausgegeben.

### 49.2.3. Archive-Callback

Der Callback `archive_file_cb` wird aufgerufen, um eine einzelne WAL-Datei zu archivieren.

```c
typedef bool (*ArchiveFileCB) (ArchiveModuleState *state, const char *file, const char *path);
```

Wenn `true` zurückgegeben wird, fährt der Server so fort, als wäre die Datei erfolgreich archiviert worden; dazu kann auch das Wiederverwenden oder Entfernen der ursprünglichen WAL-Datei gehören. Wenn `false` zurückgegeben oder ein Fehler ausgelöst wird, behält der Server die ursprüngliche WAL-Datei und versucht die Archivierung später erneut. `file` enthält nur den Dateinamen der zu archivierenden WAL-Datei, während `path` den vollständigen Pfad der WAL-Datei enthält (einschließlich des Dateinamens).

> Der Callback `archive_file_cb` wird in einem kurzlebigen Speicherkontext aufgerufen, der zwischen Aufrufen zurückgesetzt wird. Wenn Sie längerlebigen Speicher benötigen, erstellen Sie im Callback `startup_cb` des Moduls einen Speicherkontext.

### 49.2.4. Shutdown-Callback

Der Callback `shutdown_cb` wird aufgerufen, wenn der Archiver-Prozess beendet wird (zum Beispiel nach einem Fehler) oder sich der Wert von `archive_library` ändert. Wenn kein `shutdown_cb` definiert ist, wird in diesen Situationen keine besondere Aktion ausgeführt. Wenn das Archivmodul eigenen Zustand hat, sollte dieser Callback ihn freigeben, um Lecks zu vermeiden.

```c
typedef void (*ArchiveShutdownCB) (ArchiveModuleState *state);
```
