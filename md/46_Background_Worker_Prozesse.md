# 46. Background-Worker-Prozesse

PostgreSQL kann erweitert werden, um vom Benutzer bereitgestellten Code in separaten Prozessen auszuführen. Solche Prozesse werden von `postgres` gestartet, gestoppt und überwacht, wodurch ihre Lebensdauer eng mit dem Zustand des Servers verbunden sein kann. Diese Prozesse sind an den Shared-Memory-Bereich von PostgreSQL angebunden und haben die Möglichkeit, intern Verbindungen zu Datenbanken herzustellen; sie können außerdem mehrere Transaktionen seriell ausführen, genau wie ein gewöhnlicher Serverprozess mit Client-Verbindung. Außerdem können sie durch Linken gegen `libpq` eine Verbindung zum Server herstellen und sich wie eine gewöhnliche Client-Anwendung verhalten.

**Warnung:** Die Verwendung von Background-Worker-Prozessen birgt erhebliche Robustheits- und Sicherheitsrisiken, weil sie in C geschrieben sind und uneingeschränkten Zugriff auf Daten haben. Administratoren, die Module aktivieren möchten, die Background-Worker-Prozesse enthalten, sollten äußerste Vorsicht walten lassen. Nur sorgfältig geprüfte Module sollten Background-Worker-Prozesse ausführen dürfen.

Background Worker können beim Start von PostgreSQL initialisiert werden, indem der Modulname in `shared_preload_libraries` aufgenommen wird. Ein Modul, das einen Background Worker ausführen möchte, kann ihn registrieren, indem es aus seiner Funktion `_PG_init()` heraus `RegisterBackgroundWorker(BackgroundWorker *worker)` aufruft. Background Worker können auch gestartet werden, nachdem das System hochgefahren ist und läuft, indem `RegisterDynamicBackgroundWorker(BackgroundWorker *worker, BackgroundWorkerHandle **handle)` aufgerufen wird. Anders als `RegisterBackgroundWorker`, das nur innerhalb des Postmaster-Prozesses aufgerufen werden kann, muss `RegisterDynamicBackgroundWorker` aus einem regulären Backend oder einem anderen Background Worker aufgerufen werden.

Die Struktur `BackgroundWorker` ist wie folgt definiert:

```c
typedef void (*bgworker_main_type)(Datum main_arg);
typedef struct BackgroundWorker
{
    char        bgw_name[BGW_MAXLEN];
    char        bgw_type[BGW_MAXLEN];
    int         bgw_flags;
    BgWorkerStartTime bgw_start_time;
    int         bgw_restart_time;        /* in seconds, or BGW_NEVER_RESTART */
    char        bgw_library_name[MAXPGPATH];
    char        bgw_function_name[BGW_MAXLEN];
    Datum       bgw_main_arg;
    char        bgw_extra[BGW_EXTRALEN];
    pid_t       bgw_notify_pid;
} BackgroundWorker;
```

`bgw_name` und `bgw_type` sind Zeichenketten, die in Logmeldungen, Prozesslisten und ähnlichen Kontexten verwendet werden. `bgw_type` sollte für alle Background Worker desselben Typs gleich sein, damit solche Worker zum Beispiel in einer Prozessliste gruppiert werden können. `bgw_name` kann dagegen zusätzliche Informationen über den konkreten Prozess enthalten. Typischerweise enthält die Zeichenkette für `bgw_name` auf irgendeine Weise den Typ, aber das ist nicht zwingend erforderlich.

`bgw_flags` ist eine mit bitweisem Oder gebildete Bitmaske, die die Fähigkeiten angibt, die das Modul benötigt. Mögliche Werte sind:

- `BGWORKER_SHMEM_ACCESS`
  - Fordert Shared-Memory-Zugriff an. Dieses Flag ist erforderlich.
- `BGWORKER_BACKEND_DATABASE_CONNECTION`
  - Fordert die Fähigkeit an, eine Datenbankverbindung herzustellen, über die später Transaktionen und Abfragen ausgeführt werden können. Ein Background Worker, der `BGWORKER_BACKEND_DATABASE_CONNECTION` verwendet, um eine Verbindung zu einer Datenbank herzustellen, muss außerdem mit `BGWORKER_SHMEM_ACCESS` Shared Memory anbinden, andernfalls schlägt der Start des Workers fehl.

`bgw_start_time` ist der Serverzustand, in dem `postgres` den Prozess starten soll. Er kann einer von `BgWorkerStart_PostmasterStart` sein (Start, sobald `postgres` selbst seine Initialisierung abgeschlossen hat; Prozesse, die dies anfordern, können keine Datenbankverbindungen nutzen), `BgWorkerStart_ConsistentState` (Start, sobald in einem Hot Standby ein konsistenter Zustand erreicht wurde, sodass Prozesse sich mit Datenbanken verbinden und schreibgeschützte Abfragen ausführen können) und `BgWorkerStart_RecoveryFinished` (Start, sobald das System in den normalen Lese-/Schreibzustand eingetreten ist). Beachten Sie, dass die letzten beiden Werte auf einem Server, der kein Hot Standby ist, gleichwertig sind. Beachten Sie außerdem, dass diese Einstellung nur angibt, wann die Prozesse gestartet werden sollen; sie stoppen nicht, wenn ein anderer Zustand erreicht wird.

`bgw_restart_time` ist das Intervall in Sekunden, das `postgres` warten soll, bevor der Prozess nach einem Absturz neu gestartet wird. Es kann ein beliebiger positiver Wert sein oder `BGW_NEVER_RESTART`, was bedeutet, dass der Prozess nach einem Absturz nicht neu gestartet wird.

`bgw_library_name` ist der Name einer Bibliothek, in der der initiale Einstiegspunkt für den Background Worker gesucht werden soll. Die genannte Bibliothek wird vom Worker-Prozess dynamisch geladen, und `bgw_function_name` wird verwendet, um die aufzurufende Funktion zu identifizieren. Wenn eine Funktion im Core-Code aufgerufen wird, muss dies auf `"postgres"` gesetzt werden.

`bgw_function_name` ist der Name der Funktion, die als initialer Einstiegspunkt für den neuen Background Worker verwendet wird. Wenn sich diese Funktion in einer dynamisch geladenen Bibliothek befindet, muss sie als `PGDLLEXPORT` markiert sein und darf nicht `static` sein.

`bgw_main_arg` ist das Argument vom Typ `Datum` für die Hauptfunktion des Background Workers. Diese Hauptfunktion sollte ein einzelnes Argument vom Typ `Datum` entgegennehmen und `void` zurückgeben. `bgw_main_arg` wird als Argument übergeben. Zusätzlich zeigt die globale Variable `MyBgworkerEntry` auf eine Kopie der Struktur `BackgroundWorker`, die bei der Registrierung übergeben wurde; der Worker kann es hilfreich finden, diese Struktur zu untersuchen.

Unter Windows sowie überall dort, wo `EXEC_BACKEND` definiert ist, oder bei dynamischen Background Workern ist es nicht sicher, ein `Datum` per Referenz zu übergeben, sondern nur per Wert. Wenn ein Argument erforderlich ist, ist es am sichersten, ein `int32` oder einen anderen kleinen Wert zu übergeben und ihn als Index in ein im Shared Memory zugewiesenes Array zu verwenden. Wenn ein Wert wie ein `cstring` oder `text` übergeben wird, ist der Zeiger aus dem neuen Background-Worker-Prozess heraus nicht gültig.

`bgw_extra` kann zusätzliche Daten enthalten, die an den Background Worker übergeben werden. Anders als `bgw_main_arg` werden diese Daten nicht als Argument an die Hauptfunktion des Workers übergeben, können aber wie oben beschrieben über `MyBgworkerEntry` abgerufen werden.

`bgw_notify_pid` ist die PID eines PostgreSQL-Backend-Prozesses, an den der Postmaster `SIGUSR1` senden soll, wenn der Prozess gestartet wird oder beendet ist. Sie sollte `0` sein für Worker, die beim Start des Postmasters registriert werden, oder wenn das Backend, das den Worker registriert, nicht auf dessen Start warten möchte. Andernfalls sollte sie mit `MyProcPid` initialisiert werden.

Sobald der Prozess läuft, kann er sich mit einer Datenbank verbinden, indem er `BackgroundWorkerInitializeConnection(char *dbname, char *username, uint32 flags)` oder `BackgroundWorkerInitializeConnectionByOid(Oid dboid, Oid useroid, uint32 flags)` aufruft. Dadurch kann der Prozess Transaktionen und Abfragen über die SPI-Schnittstelle ausführen. Wenn `dbname` `NULL` oder `dboid` `InvalidOid` ist, ist die Sitzung mit keiner bestimmten Datenbank verbunden, aber gemeinsam genutzte Kataloge können angesprochen werden. Wenn `username` `NULL` oder `useroid` `InvalidOid` ist, läuft der Prozess als der während `initdb` erzeugte Superuser. Wenn `BGWORKER_BYPASS_ALLOWCONN` als `flags` angegeben wird, ist es möglich, die Einschränkung zu umgehen, dass keine Verbindung zu Datenbanken hergestellt werden darf, die Benutzerverbindungen nicht erlauben. Wenn `BGWORKER_BYPASS_ROLELOGINCHECK` als `flags` angegeben wird, ist es möglich, die Login-Prüfung für die Rolle zu umgehen, die zur Verbindung mit Datenbanken verwendet wird. Ein Background Worker kann nur eine dieser beiden Funktionen und nur einmal aufrufen. Ein Wechsel der Datenbank ist nicht möglich.

Signale sind zunächst blockiert, wenn die Steuerung die Hauptfunktion des Background Workers erreicht, und müssen von ihr freigegeben werden. Dadurch kann der Prozess seine Signal-Handler bei Bedarf anpassen. Signale können im neuen Prozess durch Aufruf von `BackgroundWorkerUnblockSignals` freigegeben und durch Aufruf von `BackgroundWorkerBlockSignals` blockiert werden.

Wenn `bgw_restart_time` für einen Background Worker als `BGW_NEVER_RESTART` konfiguriert ist, oder wenn er mit Exit-Code `0` beendet oder durch `TerminateBackgroundWorker` terminiert wird, wird er beim Beenden automatisch vom Postmaster deregistriert. Andernfalls wird er nach dem über `bgw_restart_time` konfigurierten Zeitraum neu gestartet, oder sofort, wenn der Postmaster den Cluster wegen eines Backend-Fehlers reinitialisiert. Backends, die ihre Ausführung nur vorübergehend aussetzen müssen, sollten einen unterbrechbaren Schlaf verwenden, statt sich zu beenden; dies kann durch Aufruf von `WaitLatch()` erreicht werden. Stellen Sie sicher, dass beim Aufruf dieser Funktion das Flag `WL_POSTMASTER_DEATH` gesetzt ist, und prüfen Sie den Rückgabecode, um im Notfall, dass `postgres` selbst beendet wurde, umgehend auszusteigen.

Wenn ein Background Worker mit der Funktion `RegisterDynamicBackgroundWorker` registriert wird, kann das Backend, das die Registrierung durchführt, Informationen über den Status des Workers erhalten. Backends, die dies tun möchten, sollten die Adresse eines `BackgroundWorkerHandle *` als zweites Argument an `RegisterDynamicBackgroundWorker` übergeben. Wenn der Worker erfolgreich registriert wird, wird dieser Zeiger mit einem opaken Handle initialisiert, das später an `GetBackgroundWorkerPid(BackgroundWorkerHandle *, pid_t *)` oder `TerminateBackgroundWorker(BackgroundWorkerHandle *)` übergeben werden kann. `GetBackgroundWorkerPid` kann verwendet werden, um den Status des Workers abzufragen: Ein Rückgabewert von `BGWH_NOT_YET_STARTED` zeigt an, dass der Worker noch nicht vom Postmaster gestartet wurde; `BGWH_STOPPED` zeigt an, dass er gestartet wurde, aber nicht mehr läuft; und `BGWH_STARTED` zeigt an, dass er aktuell läuft. Im letzten Fall wird außerdem die PID über das zweite Argument zurückgegeben. `TerminateBackgroundWorker` veranlasst den Postmaster, `SIGTERM` an den Worker zu senden, wenn er läuft, und ihn zu deregistrieren, sobald er nicht mehr läuft.

In manchen Fällen möchte ein Prozess, der einen Background Worker registriert, warten, bis der Worker startet. Dies kann erreicht werden, indem `bgw_notify_pid` auf `MyProcPid` initialisiert und anschließend das bei der Registrierung erhaltene `BackgroundWorkerHandle *` an die Funktion `WaitForBackgroundWorkerStartup(BackgroundWorkerHandle *handle, pid_t *)` übergeben wird. Diese Funktion blockiert, bis der Postmaster versucht hat, den Background Worker zu starten, oder bis der Postmaster stirbt. Wenn der Background Worker läuft, ist der Rückgabewert `BGWH_STARTED`, und die PID wird an die bereitgestellte Adresse geschrieben. Andernfalls ist der Rückgabewert `BGWH_STOPPED` oder `BGWH_POSTMASTER_DIED`.

Ein Prozess kann auch warten, bis ein Background Worker beendet ist, indem er die Funktion `WaitForBackgroundWorkerShutdown(BackgroundWorkerHandle *handle)` verwendet und das bei der Registrierung erhaltene `BackgroundWorkerHandle *` übergibt. Diese Funktion blockiert, bis der Background Worker beendet ist oder der Postmaster stirbt. Wenn der Background Worker beendet wird, ist der Rückgabewert `BGWH_STOPPED`; wenn der Postmaster stirbt, gibt sie `BGWH_POSTMASTER_DIED` zurück.

Background Worker können asynchrone Benachrichtigungen senden, entweder über den Befehl `NOTIFY` via SPI oder direkt über `Async_Notify()`. Solche Benachrichtigungen werden beim Commit der Transaktion gesendet. Background Worker sollten sich nicht mit dem Befehl `LISTEN` für den Empfang asynchroner Benachrichtigungen registrieren, da es keine Infrastruktur gibt, mit der ein Worker solche Benachrichtigungen konsumieren kann.

Das Modul `src/test/modules/worker_spi` enthält ein funktionierendes Beispiel, das einige nützliche Techniken demonstriert.

Die maximale Zahl registrierter Background Worker wird durch `max_worker_processes` begrenzt.
