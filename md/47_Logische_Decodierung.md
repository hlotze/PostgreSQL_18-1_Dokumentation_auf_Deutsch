# 47. Logische Decodierung

PostgreSQL stellt Infrastruktur bereit, um die über SQL vorgenommenen Änderungen an externe Verbraucher zu streamen. Diese Funktionalität kann für verschiedene Zwecke genutzt werden, darunter Replikationslösungen und Auditing.

Änderungen werden in Streams ausgegeben, die durch logische Replikations-Slots identifiziert werden.

Das Format, in dem diese Änderungen gestreamt werden, wird durch das verwendete Output-Plugin bestimmt. Ein Beispiel-Plugin wird mit der PostgreSQL-Distribution bereitgestellt. Zusätzliche Plugins können geschrieben werden, um die Auswahl verfügbarer Formate zu erweitern, ohne Core-Code zu ändern. Jedes Output-Plugin hat Zugriff auf jede einzelne neue Zeile, die durch `INSERT` erzeugt wird, und auf die neue Zeilenversion, die durch `UPDATE` erzeugt wird. Ob alte Zeilenversionen für `UPDATE` und `DELETE` verfügbar sind, hängt von der konfigurierten Replica Identity ab; siehe `REPLICA IDENTITY`.

Änderungen können entweder über das Streaming-Replikationsprotokoll konsumiert werden (siehe [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll) und Abschnitt 47.3) oder durch den Aufruf von Funktionen über SQL (siehe Abschnitt 47.4). Es ist auch möglich, zusätzliche Methoden zum Konsumieren der Ausgabe eines Replikations-Slots zu schreiben, ohne Core-Code zu ändern (siehe Abschnitt 47.7).

## 47.1. Beispiele für logische Decodierung

Das folgende Beispiel demonstriert die Steuerung der logischen Decodierung über die SQL-Schnittstelle.

Bevor Sie logische Decodierung verwenden können, müssen Sie `wal_level` auf `logical` und `max_replication_slots` auf mindestens 1 setzen. Danach sollten Sie sich als Superuser mit der Zieldatenbank verbinden; im folgenden Beispiel ist das `postgres`.

```text
postgres=# -- Create a slot named 'regression_slot' using the output plugin 'test_decoding'
postgres=# SELECT * FROM
postgres-#     pg_create_logical_replication_slot('regression_slot',
postgres(#                                        'test_decoding', false, true);
    slot_name    |    lsn
-----------------+-----------
 regression_slot | 0/16B1970
(1 row)

postgres=# SELECT slot_name, plugin, slot_type, database, active,
postgres-#        restart_lsn, confirmed_flush_lsn FROM pg_replication_slots;
    slot_name    |    plugin    | slot_type | database | active | restart_lsn | confirmed_flush_lsn
-----------------+--------------+-----------+----------+--------+-------------+---------------------
 regression_slot | test_decoding | logical   | postgres | f      | 0/16A4408   | 0/16A4440
(1 row)

postgres=# -- There are no changes to see yet
postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
 lsn | xid | data
-----+-----+------
(0 rows)

postgres=# CREATE TABLE data(id serial primary key, data text);
CREATE TABLE

postgres=# -- DDL isn't replicated, so all you'll see is the transaction
postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
    lsn    |  xid  |     data
-----------+-------+--------------
 0/BA2DA58 | 10297 | BEGIN 10297
 0/BA5A5A0 | 10297 | COMMIT 10297
(2 rows)

postgres=# -- Once changes are read, they're consumed and not emitted
postgres=# -- in a subsequent call:
postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
 lsn | xid | data
-----+-----+------
(0 rows)

postgres=# BEGIN;
postgres=*# INSERT INTO data(data) VALUES('1');
postgres=*# INSERT INTO data(data) VALUES('2');
postgres=*# COMMIT;

postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
    lsn    |  xid  |                            data
-----------+-------+-------------------------------------------------------------
 0/BA5A688 | 10298 | BEGIN 10298
 0/BA5A6F0 | 10298 | table public.data: INSERT: id[integer]:1 data[text]:'1'
 0/BA5A7F8 | 10298 | table public.data: INSERT: id[integer]:2 data[text]:'2'
 0/BA5A8A8 | 10298 | COMMIT 10298
(4 rows)

postgres=# INSERT INTO data(data) VALUES('3');

postgres=# -- You can also peek ahead in the change stream without consuming changes
postgres=# SELECT * FROM pg_logical_slot_peek_changes('regression_slot', NULL, NULL);
    lsn    |  xid  |                            data
-----------+-------+-------------------------------------------------------------
 0/BA5A8E0 | 10299 | BEGIN 10299
 0/BA5A8E0 | 10299 | table public.data: INSERT: id[integer]:3 data[text]:'3'
 0/BA5A990 | 10299 | COMMIT 10299
(3 rows)

postgres=# -- The next call to pg_logical_slot_peek_changes() returns the same changes again
postgres=# SELECT * FROM pg_logical_slot_peek_changes('regression_slot', NULL, NULL);
    lsn    |  xid  |                            data
-----------+-------+-------------------------------------------------------------
 0/BA5A8E0 | 10299 | BEGIN 10299
 0/BA5A8E0 | 10299 | table public.data: INSERT: id[integer]:3 data[text]:'3'
 0/BA5A990 | 10299 | COMMIT 10299
(3 rows)

postgres=# -- options can be passed to output plugin, to influence the formatting
postgres=# SELECT * FROM pg_logical_slot_peek_changes('regression_slot', NULL, NULL,
postgres(#                                           'include-timestamp', 'on');
    lsn    |  xid  |                            data
-----------+-------+-------------------------------------------------------------
 0/BA5A8E0 | 10299 | BEGIN 10299
 0/BA5A8E0 | 10299 | table public.data: INSERT: id[integer]:3 data[text]:'3'
 0/BA5A990 | 10299 | COMMIT 10299 (at 2017-05-10 12:07:21.272494-04)
(3 rows)

postgres=# -- Remember to destroy a slot you no longer need to stop it consuming
postgres=# -- server resources:
postgres=# SELECT pg_drop_replication_slot('regression_slot');
 pg_drop_replication_slot
--------------------------

(1 row)
```

Die folgenden Beispiele zeigen, wie logische Decodierung über das Streaming-Replikationsprotokoll gesteuert wird, unter Verwendung des Programms `pg_recvlogical`, das in der PostgreSQL-Distribution enthalten ist. Dies setzt voraus, dass die Client-Authentifizierung so eingerichtet ist, dass Replikationsverbindungen erlaubt sind (siehe [Abschnitt 26.2.5.1](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#26251-authentifizierung)) und dass `max_wal_senders` hoch genug gesetzt ist, um eine zusätzliche Verbindung zu erlauben. Das zweite Beispiel zeigt, wie Zwei-Phasen-Transaktionen gestreamt werden. Bevor Sie Zwei-Phasen-Befehle verwenden, müssen Sie `max_prepared_transactions` auf mindestens 1 setzen.

Beispiel 1:

```text
$ pg_recvlogical -d postgres --slot=test --create-slot
$ pg_recvlogical -d postgres --slot=test --start -f -
Control+Z
$ psql -d postgres -c "INSERT INTO data(data) VALUES('4');"
$ fg
BEGIN 693
table public.data: INSERT: id[integer]:4 data[text]:'4'
COMMIT 693
Control+C
$ pg_recvlogical -d postgres --slot=test --drop-slot
```

Beispiel 2:

```text
$ pg_recvlogical -d postgres --slot=test --create-slot --enable-two-phase
$ pg_recvlogical -d postgres --slot=test --start -f -
Control+Z
$ psql -d postgres -c "BEGIN;INSERT INTO data(data) VALUES('5');PREPARE TRANSACTION 'test';"
$ fg
BEGIN 694
table public.data: INSERT: id[integer]:5 data[text]:'5'
PREPARE TRANSACTION 'test', txid 694
Control+Z
$ psql -d postgres -c "COMMIT PREPARED 'test';"
$ fg
COMMIT PREPARED 'test', txid 694
Control+C
$ pg_recvlogical -d postgres --slot=test --drop-slot
```

Das folgende Beispiel zeigt die SQL-Schnittstelle, die zum Decodieren vorbereiteter Transaktionen verwendet werden kann. Bevor Sie Zwei-Phasen-Commit-Befehle verwenden, müssen Sie `max_prepared_transactions` auf mindestens 1 setzen. Außerdem müssen Sie beim Erstellen des Slots mit `pg_create_logical_replication_slot` den Parameter `two_phase` auf `true` gesetzt haben. Beachten Sie, dass die gesamte Transaktion nach dem Commit gestreamt wird, wenn sie nicht bereits decodiert wurde.

```text
postgres=# BEGIN;
postgres=*# INSERT INTO data(data) VALUES('5');
postgres=*# PREPARE TRANSACTION 'test_prepared1';

postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
    lsn    | xid |                          data
-----------+-----+---------------------------------------------------------
 0/1689DC0 | 529 | BEGIN 529
 0/1689DC0 | 529 | table public.data: INSERT: id[integer]:3 data[text]:'5'
 0/1689FC0 | 529 | PREPARE TRANSACTION 'test_prepared1', txid 529
(3 rows)

postgres=# COMMIT PREPARED 'test_prepared1';
postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
    lsn    | xid |                    data
-----------+-----+--------------------------------------------
 0/168A060 | 529 | COMMIT PREPARED 'test_prepared1', txid 529
(1 row)

postgres=# -- you can also rollback a prepared transaction
postgres=# BEGIN;
postgres=*# INSERT INTO data(data) VALUES('6');
postgres=*# PREPARE TRANSACTION 'test_prepared2';
postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
    lsn    | xid |                          data
-----------+-----+---------------------------------------------------------
 0/168A180 | 530 | BEGIN 530
 0/168A1E8 | 530 | table public.data: INSERT: id[integer]:4 data[text]:'6'
 0/168A430 | 530 | PREPARE TRANSACTION 'test_prepared2', txid 530
(3 rows)

postgres=# ROLLBACK PREPARED 'test_prepared2';
postgres=# SELECT * FROM pg_logical_slot_get_changes('regression_slot', NULL, NULL);
    lsn    | xid |                     data
-----------+-----+----------------------------------------------
 0/168A4B8 | 530 | ROLLBACK PREPARED 'test_prepared2', txid 530
(1 row)
```

## 47.2. Konzepte der logischen Decodierung

### 47.2.1. Logische Decodierung

Logische Decodierung ist der Prozess, alle persistenten Änderungen an den Tabellen einer Datenbank in ein kohärentes, leicht verständliches Format zu extrahieren, das ohne detailliertes Wissen über den internen Zustand der Datenbank interpretiert werden kann.

In PostgreSQL wird logische Decodierung dadurch implementiert, dass der Inhalt des Write-Ahead-Logs, der Änderungen auf Speicherebene beschreibt, in eine anwendungsspezifische Form decodiert wird, etwa in einen Strom von Tupeln oder SQL-Anweisungen.

### 47.2.2. Replikations-Slots

Im Kontext der logischen Replikation repräsentiert ein Slot einen Strom von Änderungen, die in der Reihenfolge, in der sie auf dem Ursprungsserver vorgenommen wurden, an einen Client wiedergegeben werden können. Jeder Slot streamt eine Folge von Änderungen aus einer einzelnen Datenbank.

**Hinweis:** PostgreSQL hat auch Streaming-Replikations-Slots; siehe Abschnitt 26.2.5. Dort werden sie jedoch etwas anders verwendet.

Ein Replikations-Slot hat einen Bezeichner, der über alle Datenbanken in einem PostgreSQL-Cluster hinweg eindeutig ist. Slots bleiben unabhängig von der Verbindung bestehen, die sie verwendet, und sind absturzsicher.

Ein logischer Slot gibt jede Änderung im Normalbetrieb genau einmal aus. Die aktuelle Position jedes Slots wird nur bei einem Checkpoint persistent gemacht. Daher kann der Slot nach einem Absturz zu einer früheren LSN zurückkehren, was dazu führt, dass aktuelle Änderungen beim Serverneustart erneut gesendet werden. Clients der logischen Decodierung sind dafür verantwortlich, negative Folgen durch die mehrfache Verarbeitung derselben Nachricht zu vermeiden. Clients können die letzte LSN aufzeichnen, die sie beim Decodieren gesehen haben, und wiederholte Daten überspringen oder, wenn sie das Replikationsprotokoll verwenden, verlangen, dass die Decodierung ab dieser LSN beginnt, statt den Server den Startpunkt bestimmen zu lassen. Die Funktion zum Verfolgen des Replikationsfortschritts ist für diesen Zweck gedacht; siehe Replikations-Ursprünge.

Für eine einzelne Datenbank können mehrere unabhängige Slots existieren. Jeder Slot hat seinen eigenen Zustand, sodass verschiedene Verbraucher Änderungen ab unterschiedlichen Punkten im Änderungsstrom der Datenbank empfangen können. Für die meisten Anwendungen wird für jeden Verbraucher ein eigener Slot benötigt.

Ein logischer Replikations-Slot weiß nichts über den Zustand des oder der Empfänger. Es ist sogar möglich, dass mehrere unterschiedliche Empfänger denselben Slot zu unterschiedlichen Zeiten verwenden; sie erhalten dann einfach die Änderungen ab dem Punkt, an dem der letzte Empfänger aufgehört hat, sie zu konsumieren. Zu einem bestimmten Zeitpunkt darf nur ein Empfänger Änderungen aus einem Slot konsumieren.

Ein logischer Replikations-Slot kann auch auf einem Hot Standby erstellt werden. Um zu verhindern, dass `VACUUM` benötigte Zeilen aus den Systemkatalogen entfernt, sollte auf dem Standby `hot_standby_feedback` gesetzt werden. Wenn dennoch benötigte Zeilen entfernt werden, wird der Slot ungültig. Es wird dringend empfohlen, zwischen Primary und Standby einen physischen Slot zu verwenden. Andernfalls funktioniert `hot_standby_feedback` nur, solange die Verbindung aktiv ist; zum Beispiel würde ein Neustart des Knotens dies unterbrechen. Dann kann der Primary Systemkatalogzeilen löschen, die von der logischen Decodierung auf dem Standby benötigt werden könnten, da er nichts über `catalog_xmin` auf dem Standby weiß. Vorhandene logische Slots auf dem Standby werden außerdem ungültig, wenn `wal_level` auf dem Primary auf weniger als `logical` reduziert wird. Dies geschieht, sobald der Standby eine solche Änderung im WAL-Strom erkennt. Das bedeutet, dass für verzögerte WAL-Sender, falls vorhanden, einige WAL-Datensätze bis zur Änderung des Parameters `wal_level` auf dem Primary nicht decodiert werden.

Das Erstellen eines logischen Slots erfordert Informationen über alle aktuell laufenden Transaktionen. Auf dem Primary sind diese Informationen direkt verfügbar; auf einem Standby müssen sie jedoch vom Primary bezogen werden. Daher muss das Erstellen des Slots möglicherweise warten, bis auf dem Primary Aktivität stattfindet. Wenn der Primary untätig ist, kann das Erstellen eines logischen Slots auf dem Standby spürbar dauern. Dies kann beschleunigt werden, indem auf dem Primary die Funktion `pg_log_standby_snapshot` aufgerufen wird.

**Achtung:** Replikations-Slots überdauern Abstürze und wissen nichts über den Zustand ihrer Verbraucher. Sie verhindern das Entfernen benötigter Ressourcen auch dann, wenn keine Verbindung sie verwendet. Das verbraucht Speicherplatz, weil weder benötigtes WAL noch benötigte Zeilen aus den Systemkatalogen von `VACUUM` entfernt werden können, solange sie von einem Replikations-Slot benötigt werden. In Extremfällen kann dies dazu führen, dass die Datenbank heruntergefahren wird, um einen Transaction-ID-Wraparound zu verhindern; siehe Abschnitt 24.1.5. Wenn ein Slot nicht mehr benötigt wird, sollte er daher gelöscht werden.

### 47.2.3. Synchronisierung von Replikations-Slots

Die logischen Replikations-Slots auf dem Primary können mit dem Hot Standby synchronisiert werden, indem der Parameter `failover` von `pg_create_logical_replication_slot` verwendet wird oder indem bei der Slot-Erstellung die Option `failover` von `CREATE SUBSCRIPTION` verwendet wird. Zusätzlich muss auf dem Standby `sync_replication_slots` aktiviert werden. Durch Aktivieren von `sync_replication_slots` auf dem Standby können die Failover-Slots periodisch im Slotsync-Worker synchronisiert werden. Damit die Synchronisierung funktioniert, muss zwischen Primary und Standby zwingend ein physischer Replikations-Slot vorhanden sein, das heißt `primary_slot_name` sollte auf dem Standby konfiguriert sein, und `hot_standby_feedback` muss auf dem Standby aktiviert sein. Außerdem muss in `primary_conninfo` ein gültiger `dbname` angegeben werden. Es wird dringend empfohlen, den genannten physischen Replikations-Slot in der Liste `synchronized_standby_slots` auf dem Primary aufzuführen, um zu verhindern, dass der Subscriber Änderungen schneller konsumiert als der Hot Standby. Selbst bei korrekter Konfiguration ist beim Senden von Änderungen an logische Subscriber mit einer gewissen Latenz zu rechnen, weil auf Slots gewartet wird, die in `synchronized_standby_slots` genannt sind. Wenn `synchronized_standby_slots` verwendet wird, fährt der Primary-Server erst dann vollständig herunter, wenn die entsprechenden Standbys, die den in `synchronized_standby_slots` angegebenen physischen Replikations-Slots zugeordnet sind, bestätigt haben, dass sie WAL bis zur letzten geflushten Position auf dem Primary-Server empfangen haben.

**Hinweis:** Während das Aktivieren von `sync_replication_slots` die automatische periodische Synchronisierung von Failover-Slots ermöglicht, können sie auch manuell mit der Funktion `pg_sync_replication_slots` auf dem Standby synchronisiert werden. Diese Funktion ist jedoch hauptsächlich für Tests und Debugging gedacht und sollte mit Vorsicht verwendet werden. Anders als die automatische Synchronisierung enthält sie keine zyklischen Wiederholungen und ist daher anfälliger für Synchronisierungsfehler, insbesondere bei initialen Synchronisierungsszenarien, in denen die für den Slot benötigten WAL-Dateien oder Katalogzeilen auf dem Standby bereits entfernt wurden oder von Entfernung bedroht sind. Im Gegensatz dazu stellt die automatische Synchronisierung über `sync_replication_slots` kontinuierliche Slot-Aktualisierungen bereit, ermöglicht nahtloses Failover und unterstützt Hochverfügbarkeit. Daher ist sie die empfohlene Methode zum Synchronisieren von Slots.

Wenn die Slot-Synchronisierung wie empfohlen konfiguriert ist und die initiale Synchronisierung entweder automatisch oder manuell über `pg_sync_replication_slots` ausgeführt wird, kann der Standby den synchronisierten Slot nur dauerhaft speichern, wenn die folgende Bedingung erfüllt ist: Der logische Replikations-Slot auf dem Primary muss WAL und Systemkatalogzeilen vorhalten, die auf dem Standby noch verfügbar sind. Dies stellt Datenintegrität sicher und erlaubt, dass logische Replikation nach einer Promotion reibungslos fortgesetzt wird. Wenn die benötigten WAL-Dateien oder Katalogzeilen bereits vom Standby entfernt wurden, wird der Slot nicht dauerhaft gespeichert, um Datenverlust zu vermeiden. In solchen Fällen kann die folgende Logmeldung erscheinen:

```text
LOG: could not synchronize replication slot "failover_slot"
DETAIL: Synchronization could lead to data loss, because the
 remote slot needs WAL at LSN 0/3003F28 and catalog xmin 754, but
 the standby has LSN 0/3003F28 and catalog xmin 756.
```

Wenn der logische Replikations-Slot aktiv von einem Verbraucher verwendet wird, ist kein manueller Eingriff nötig; der Slot schreitet automatisch voran, und die Synchronisierung wird im nächsten Zyklus fortgesetzt. Wenn jedoch kein Verbraucher konfiguriert ist, ist es ratsam, den Slot auf dem Primary manuell mit `pg_logical_slot_get_changes` oder `pg_logical_slot_get_binary_changes` voranzuschieben, damit die Synchronisierung fortfahren kann.

Die Möglichkeit, logische Replikation nach einem Failover fortzusetzen, hängt vom Wert `pg_replication_slots.synced` der synchronisierten Slots auf dem Standby zum Zeitpunkt des Failovers ab. Nur persistente Slots, die vor dem Failover auf dem Standby den synchronisierten Zustand `true` erreicht haben, können nach dem Failover für logische Replikation verwendet werden. Temporäre synchronisierte Slots können nicht für logische Decodierung verwendet werden; daher kann logische Replikation für diese Slots nicht fortgesetzt werden. Wenn der synchronisierte Slot zum Beispiel wegen einer deaktivierten Subscription auf dem Standby nicht persistent werden konnte, kann die Subscription nach dem Failover nicht fortgesetzt werden, selbst wenn sie aktiviert wird.

Um logische Replikation nach einem Failover von synchronisierten logischen Slots fortzusetzen, muss `conninfo` der Subscription so geändert werden, dass es auf den neuen Primary-Server zeigt. Dies geschieht mit `ALTER SUBSCRIPTION ... CONNECTION`. Es wird empfohlen, Subscriptions zunächst zu deaktivieren, bevor der Standby promoted wird, und sie nach dem Ändern der Verbindungszeichenkette wieder zu aktivieren.

**Achtung:** Es besteht die Möglichkeit, dass der alte Primary während der Promotion wieder verfügbar wird. Wenn Subscriptions nicht deaktiviert sind, können die logischen Subscriber auch nach der Promotion weiterhin Daten vom alten Primary-Server empfangen, bis die Verbindungszeichenkette geändert wird. Dies kann zu Dateninkonsistenzen führen und verhindern, dass die logischen Subscriber die Replikation vom neuen Primary-Server fortsetzen können.

### 47.2.4. Output-Plugins

Output-Plugins transformieren die Daten aus der internen Darstellung des Write-Ahead-Logs in das Format, das der Verbraucher eines Replikations-Slots wünscht.

### 47.2.5. Exportierte Snapshots

Wenn ein neuer Replikations-Slot über die Streaming-Replikationsschnittstelle erstellt wird (siehe `CREATE_REPLICATION_SLOT`), wird ein Snapshot exportiert (siehe [Abschnitt 9.28.5](09_Funktionen_und_Operatoren.md#9285-snapshotsynchronisationsfunktionen)), der exakt den Zustand der Datenbank zeigt, nach dem alle Änderungen in den Änderungsstrom aufgenommen werden. Dies kann verwendet werden, um eine neue Replika zu erzeugen, indem mit `SET TRANSACTION SNAPSHOT` der Zustand der Datenbank zu dem Zeitpunkt gelesen wird, an dem der Slot erstellt wurde. Diese Transaktion kann dann verwendet werden, um den Zustand der Datenbank zu diesem Zeitpunkt zu dumpen; anschließend kann dieser Zustand mit den Inhalten des Slots aktualisiert werden, ohne Änderungen zu verlieren.

Anwendungen, die keinen Snapshot-Export benötigen, können ihn mit der Option `SNAPSHOT 'nothing'` unterdrücken.

## 47.3. Schnittstelle des Streaming-Replikationsprotokolls

Die Befehle

- `CREATE_REPLICATION_SLOT slot_name LOGICAL output_plugin`
- `DROP_REPLICATION_SLOT slot_name [ WAIT ]`
- `START_REPLICATION SLOT slot_name LOGICAL ...`

werden jeweils verwendet, um Änderungen aus einem Replikations-Slot zu erstellen, zu löschen und zu streamen. Diese Befehle sind nur über eine Replikationsverbindung verfügbar; sie können nicht über SQL verwendet werden. Einzelheiten zu diesen Befehlen finden Sie in [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll).

Der Befehl `pg_recvlogical` kann verwendet werden, um logische Decodierung über eine Streaming-Replikationsverbindung zu steuern. Intern verwendet er diese Befehle.

## 47.4. SQL-Schnittstelle für logische Decodierung

Eine ausführliche Dokumentation der SQL-Level-API für die Interaktion mit logischer Decodierung finden Sie in [Abschnitt 9.28.6](09_Funktionen_und_Operatoren.md#9286-replikationsverwaltungsfunktionen).

Synchrone Replikation (siehe [Abschnitt 26.2.8](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#2628-synchrone-replikation)) wird nur für Replikations-Slots unterstützt, die über die Streaming-Replikationsschnittstelle verwendet werden. Die Funktionsschnittstelle und zusätzliche, nicht zum Core gehörende Schnittstellen unterstützen keine synchrone Replikation.

## 47.5. Systemkataloge zu logischer Decodierung

Die Views `pg_replication_slots` und `pg_stat_replication` liefern Informationen über den aktuellen Zustand von Replikations-Slots beziehungsweise Streaming-Replikationsverbindungen. Diese Views gelten sowohl für physische als auch für logische Replikation. Die View `pg_stat_replication_slots` liefert Statistik-Informationen über die logischen Replikations-Slots.

## 47.6. Output-Plugins für logische Decodierung

Ein Beispiel-Output-Plugin befindet sich im Unterverzeichnis `contrib/test_decoding` des PostgreSQL-Quellbaums.

### 47.6.1. Initialisierungsfunktion

Ein Output-Plugin wird geladen, indem eine Shared Library dynamisch geladen wird, deren Basisname dem Namen des Output-Plugins entspricht. Der normale Bibliothekssuchpfad wird verwendet, um die Bibliothek zu finden. Um die erforderlichen Output-Plugin-Callbacks bereitzustellen und anzuzeigen, dass die Bibliothek tatsächlich ein Output-Plugin ist, muss sie eine Funktion namens `_PG_output_plugin_init` bereitstellen. Dieser Funktion wird eine Struktur übergeben, die mit den Callback-Funktionszeigern für die einzelnen Aktionen gefüllt werden muss.

```c
typedef struct OutputPluginCallbacks
{
    LogicalDecodeStartupCB startup_cb;
    LogicalDecodeBeginCB begin_cb;
    LogicalDecodeChangeCB change_cb;
    LogicalDecodeTruncateCB truncate_cb;
    LogicalDecodeCommitCB commit_cb;
    LogicalDecodeMessageCB message_cb;
    LogicalDecodeFilterByOriginCB filter_by_origin_cb;
    LogicalDecodeShutdownCB shutdown_cb;
    LogicalDecodeFilterPrepareCB filter_prepare_cb;
    LogicalDecodeBeginPrepareCB begin_prepare_cb;
    LogicalDecodePrepareCB prepare_cb;
    LogicalDecodeCommitPreparedCB commit_prepared_cb;
    LogicalDecodeRollbackPreparedCB rollback_prepared_cb;
    LogicalDecodeStreamStartCB stream_start_cb;
    LogicalDecodeStreamStopCB stream_stop_cb;
    LogicalDecodeStreamAbortCB stream_abort_cb;
    LogicalDecodeStreamPrepareCB stream_prepare_cb;
    LogicalDecodeStreamCommitCB stream_commit_cb;
    LogicalDecodeStreamChangeCB stream_change_cb;
    LogicalDecodeStreamMessageCB stream_message_cb;
    LogicalDecodeStreamTruncateCB stream_truncate_cb;
} OutputPluginCallbacks;

typedef void (*LogicalOutputPluginInit) (struct OutputPluginCallbacks *cb);
```

Die Callbacks `begin_cb`, `change_cb` und `commit_cb` sind erforderlich, während `startup_cb`, `truncate_cb`, `message_cb`, `filter_by_origin_cb` und `shutdown_cb` optional sind. Wenn `truncate_cb` nicht gesetzt ist, aber ein `TRUNCATE` decodiert werden soll, wird die Aktion ignoriert.

Ein Output-Plugin kann außerdem Funktionen definieren, um das Streaming großer, noch laufender Transaktionen zu unterstützen. `stream_start_cb`, `stream_stop_cb`, `stream_abort_cb`, `stream_commit_cb` und `stream_change_cb` sind erforderlich, während `stream_message_cb` und `stream_truncate_cb` optional sind. `stream_prepare_cb` ist ebenfalls erforderlich, wenn das Output-Plugin auch Zwei-Phasen-Commits unterstützt.

Ein Output-Plugin kann außerdem Funktionen definieren, um Zwei-Phasen-Commits zu unterstützen. Dadurch können Aktionen bei `PREPARE TRANSACTION` decodiert werden. Die Callbacks `begin_prepare_cb`, `prepare_cb`, `commit_prepared_cb` und `rollback_prepared_cb` sind erforderlich, während `filter_prepare_cb` optional ist. `stream_prepare_cb` ist ebenfalls erforderlich, wenn das Output-Plugin auch das Streaming großer laufender Transaktionen unterstützt.

### 47.6.2. Fähigkeiten

Um Änderungen zu decodieren, zu formatieren und auszugeben, können Output-Plugins den größten Teil der normalen Backend-Infrastruktur verwenden, einschließlich des Aufrufs von Ausgabefunktionen. Schreibgeschützter Zugriff auf Relationen ist erlaubt, solange nur auf Relationen zugegriffen wird, die entweder von `initdb` im Schema `pg_catalog` erstellt wurden oder mit folgenden Befehlen als vom Benutzer bereitgestellte Katalogtabellen markiert wurden:

```sql
ALTER TABLE user_catalog_table SET (user_catalog_table = true);
CREATE TABLE another_catalog_table(data text) WITH (user_catalog_table = true);
```

Beachten Sie, dass der Zugriff auf Benutzerkatalogtabellen oder normale Systemkatalogtabellen in Output-Plugins ausschließlich über die Scan-APIs `systable_*` erfolgen muss. Zugriff über die Scan-APIs `heap_*` führt zu einem Fehler. Außerdem sind alle Aktionen verboten, die zur Zuweisung einer Transaktions-ID führen. Dazu gehören unter anderem das Schreiben in Tabellen, das Ausführen von DDL-Änderungen und der Aufruf von `pg_current_xact_id()`.

### 47.6.3. Ausgabemodi

Output-Plugin-Callbacks können Daten in nahezu beliebigen Formaten an den Verbraucher weitergeben. Für einige Anwendungsfälle, etwa das Anzeigen der Änderungen über SQL, ist es umständlich, Daten in einem Datentyp zurückzugeben, der beliebige Daten enthalten kann, zum Beispiel `bytea`. Wenn das Output-Plugin ausschließlich Textdaten in der Servercodierung ausgibt, kann es dies deklarieren, indem es im Startup-Callback `OutputPluginOptions.output_type` auf `OUTPUT_PLUGIN_TEXTUAL_OUTPUT` statt auf `OUTPUT_PLUGIN_BINARY_OUTPUT` setzt. In diesem Fall müssen alle Daten in der Servercodierung vorliegen, damit ein `text`-Datum sie enthalten kann. Dies wird in Builds mit aktivierten Assertions geprüft.

### 47.6.4. Output-Plugin-Callbacks

Ein Output-Plugin wird über verschiedene Callbacks, die es bereitstellen muss, über stattfindende Änderungen benachrichtigt.

Gleichzeitige Transaktionen werden in Commit-Reihenfolge decodiert, und zwischen den Begin- und Commit-Callbacks werden nur Änderungen decodiert, die zu einer bestimmten Transaktion gehören. Transaktionen, die explizit oder implizit zurückgerollt wurden, werden nie decodiert. Erfolgreiche Savepoints werden in der Reihenfolge, in der sie innerhalb der Transaktion ausgeführt wurden, in die enthaltende Transaktion eingefaltet. Eine Transaktion, die mit `PREPARE TRANSACTION` für einen Zwei-Phasen-Commit vorbereitet wird, wird ebenfalls decodiert, wenn die dafür benötigten Output-Plugin-Callbacks bereitgestellt werden. Es ist möglich, dass die aktuell decodierte vorbereitete Transaktion gleichzeitig durch einen Befehl `ROLLBACK PREPARED` abgebrochen wird. In diesem Fall wird auch die logische Decodierung dieser Transaktion abgebrochen. Alle Änderungen einer solchen Transaktion werden übersprungen, sobald der Abbruch erkannt wurde und der Callback `prepare_cb` aufgerufen wird. Dadurch werden dem Output-Plugin selbst bei einem gleichzeitigen Abbruch genügend Informationen bereitgestellt, um mit `ROLLBACK PREPARED` korrekt umzugehen, sobald dies decodiert wird.

**Hinweis:** Nur Transaktionen, die bereits sicher auf Platte geflusht wurden, werden decodiert. Das kann dazu führen, dass ein `COMMIT` nicht sofort in einem unmittelbar folgenden `pg_logical_slot_get_changes()` decodiert wird, wenn `synchronous_commit` auf `off` gesetzt ist.

#### 47.6.4.1. Startup-Callback

Der optionale Callback `startup_cb` wird immer dann aufgerufen, wenn ein Replikations-Slot erstellt wird oder aufgefordert wird, Änderungen zu streamen, unabhängig von der Zahl der Änderungen, die zur Ausgabe bereitstehen.

```c
typedef void (*LogicalDecodeStartupCB) (struct LogicalDecodingContext *ctx,
                                        OutputPluginOptions *options,
                                        bool is_init);
```

Der Parameter `is_init` ist `true`, wenn der Replikations-Slot erstellt wird, andernfalls `false`. `options` zeigt auf eine Optionsstruktur, die Output-Plugins setzen können:

```c
typedef struct OutputPluginOptions
{
    OutputPluginOutputType output_type;
    bool        receive_rewrites;
} OutputPluginOptions;
```

`output_type` muss entweder auf `OUTPUT_PLUGIN_TEXTUAL_OUTPUT` oder auf `OUTPUT_PLUGIN_BINARY_OUTPUT` gesetzt werden. Siehe auch Abschnitt 47.6.3. Wenn `receive_rewrites` `true` ist, wird das Output-Plugin auch für Änderungen aufgerufen, die durch Heap-Rewrites während bestimmter DDL-Operationen vorgenommen wurden. Diese sind für Plugins interessant, die DDL-Replikation behandeln, erfordern aber besondere Behandlung.

Der Startup-Callback sollte die in `ctx->output_plugin_options` vorhandenen Optionen validieren. Wenn das Output-Plugin einen Zustand benötigt, kann es `ctx->output_plugin_private` verwenden, um ihn zu speichern.

#### 47.6.4.2. Shutdown-Callback

Der optionale Callback `shutdown_cb` wird immer dann aufgerufen, wenn ein zuvor aktiver Replikations-Slot nicht mehr verwendet wird. Er kann verwendet werden, um Ressourcen freizugeben, die privat zum Output-Plugin gehören. Der Slot wird dabei nicht notwendigerweise gelöscht; das Streaming wird lediglich beendet.

```c
typedef void (*LogicalDecodeShutdownCB) (struct LogicalDecodingContext *ctx);
```

#### 47.6.4.3. Callback bei Transaktionsbeginn

Der erforderliche Callback `begin_cb` wird immer dann aufgerufen, wenn der Beginn einer committeten Transaktion decodiert wurde. Abgebrochene Transaktionen und ihre Inhalte werden nie decodiert.

```c
typedef void (*LogicalDecodeBeginCB) (struct LogicalDecodingContext *ctx,
                                      ReorderBufferTXN *txn);
```

Der Parameter `txn` enthält Metainformationen über die Transaktion, etwa den Zeitstempel, zu dem sie committet wurde, und ihre XID.

#### 47.6.4.4. Callback bei Transaktionsende

Der erforderliche Callback `commit_cb` wird immer dann aufgerufen, wenn ein Transaktions-Commit decodiert wurde. Die Callbacks `change_cb` für alle geänderten Zeilen wurden zuvor aufgerufen, sofern es geänderte Zeilen gab.

```c
typedef void (*LogicalDecodeCommitCB) (struct LogicalDecodingContext *ctx,
                                       ReorderBufferTXN *txn,
                                       XLogRecPtr commit_lsn);
```

#### 47.6.4.5. Change-Callback

Der erforderliche Callback `change_cb` wird für jede einzelne Zeilenänderung innerhalb einer Transaktion aufgerufen, sei es ein `INSERT`, `UPDATE` oder `DELETE`. Selbst wenn der ursprüngliche Befehl mehrere Zeilen auf einmal geändert hat, wird der Callback für jede Zeile einzeln aufgerufen. Der Callback `change_cb` kann auf System- oder Benutzerkatalogtabellen zugreifen, um die Ausgabe der Details der Zeilenänderung zu unterstützen. Beim Decodieren einer vorbereiteten, aber noch nicht committeten Transaktion oder beim Decodieren einer nicht committeten Transaktion kann dieser Change-Callback wegen eines gleichzeitigen Rollbacks genau dieser Transaktion ebenfalls mit einem Fehler abbrechen. In diesem Fall wird die logische Decodierung dieser abgebrochenen Transaktion geordnet beendet.

```c
typedef void (*LogicalDecodeChangeCB) (struct LogicalDecodingContext *ctx,
                                       ReorderBufferTXN *txn,
                                       Relation relation,
                                       ReorderBufferChange *change);
```

Die Parameter `ctx` und `txn` haben denselben Inhalt wie bei den Callbacks `begin_cb` und `commit_cb`; zusätzlich werden der Relationsdeskriptor `relation`, der auf die Relation zeigt, zu der die Zeile gehört, sowie eine Struktur `change`, die die Zeilenänderung beschreibt, übergeben.

**Hinweis:** Mit logischer Decodierung können nur Änderungen an benutzerdefinierten Tabellen extrahiert werden, die nicht unlogged (siehe `UNLOGGED`) und nicht temporär (siehe `TEMPORARY` oder `TEMP`) sind.

#### 47.6.4.6. Truncate-Callback

Der optionale Callback `truncate_cb` wird für einen `TRUNCATE`-Befehl aufgerufen.

```c
typedef void (*LogicalDecodeTruncateCB) (struct LogicalDecodingContext *ctx,
                                         ReorderBufferTXN *txn,
                                         int nrelations,
                                         Relation relations[],
                                         ReorderBufferChange *change);
```

Die Parameter entsprechen denen des Callbacks `change_cb`. Da `TRUNCATE`-Aktionen auf Tabellen, die durch Fremdschlüssel verbunden sind, jedoch gemeinsam ausgeführt werden müssen, erhält dieser Callback ein Array von Relationen statt nur einer einzelnen Relation. Einzelheiten finden Sie in der Beschreibung der Anweisung `TRUNCATE`.

#### 47.6.4.7. Origin-Filter-Callback

Der optionale Callback `filter_by_origin_cb` wird aufgerufen, um zu bestimmen, ob Daten, die von `origin_id` wiedergegeben wurden, für das Output-Plugin von Interesse sind.

```c
typedef bool (*LogicalDecodeFilterByOriginCB) (struct LogicalDecodingContext *ctx,
                                               RepOriginId origin_id);
```

Der Parameter `ctx` hat denselben Inhalt wie bei den anderen Callbacks. Außer dem Ursprung sind keine Informationen verfügbar. Um zu signalisieren, dass Änderungen, die von dem übergebenen Knoten stammen, irrelevant sind, geben Sie `true` zurück; dadurch werden sie herausgefiltert. Andernfalls geben Sie `false` zurück. Die anderen Callbacks werden für Transaktionen und Änderungen, die herausgefiltert wurden, nicht aufgerufen.

Dies ist nützlich bei der Implementierung kaskadierender oder multidirektionaler Replikationslösungen. Das Filtern nach Ursprung ermöglicht es, in solchen Konstellationen zu verhindern, dass dieselben Änderungen hin und her repliziert werden. Zwar tragen Transaktionen und Änderungen ebenfalls Informationen über den Ursprung, aber das Filtern über diesen Callback ist deutlich effizienter.

#### 47.6.4.8. Generischer Message-Callback

Der optionale Callback `message_cb` wird immer dann aufgerufen, wenn eine Nachricht der logischen Decodierung decodiert wurde.

```c
typedef void (*LogicalDecodeMessageCB) (struct LogicalDecodingContext *ctx,
                                        ReorderBufferTXN *txn,
                                        XLogRecPtr message_lsn,
                                        bool transactional,
                                        const char *prefix,
                                        Size message_size,
                                        const char *message);
```

Der Parameter `txn` enthält Metainformationen über die Transaktion, etwa den Zeitstempel, zu dem sie committet wurde, und ihre XID. Beachten Sie jedoch, dass er `NULL` sein kann, wenn die Nachricht nichttransaktional ist und in der Transaktion, die die Nachricht geloggt hat, noch keine XID zugewiesen wurde. `lsn` enthält die WAL-Position der Nachricht. `transactional` gibt an, ob die Nachricht transaktional gesendet wurde oder nicht. Ähnlich wie beim Change-Callback kann dieser Message-Callback beim Decodieren einer vorbereiteten, aber noch nicht committeten Transaktion oder beim Decodieren einer nicht committeten Transaktion wegen eines gleichzeitigen Rollbacks genau dieser Transaktion ebenfalls mit einem Fehler abbrechen. In diesem Fall wird die logische Decodierung dieser abgebrochenen Transaktion geordnet beendet. `prefix` ist ein beliebiges nullterminiertes Präfix, das verwendet werden kann, um für das aktuelle Plugin interessante Nachrichten zu identifizieren. Schließlich enthält der Parameter `message` die eigentliche Nachricht der Größe `message_size`.

Es sollte besonders darauf geachtet werden, dass das Präfix, das das Output-Plugin als interessant betrachtet, eindeutig ist. Der Name der Erweiterung oder des Output-Plugins selbst ist oft eine gute Wahl.

#### 47.6.4.9. Prepare-Filter-Callback

Der optionale Callback `filter_prepare_cb` wird aufgerufen, um zu bestimmen, ob Daten, die Teil der aktuellen Zwei-Phasen-Commit-Transaktion sind, bereits in dieser Prepare-Phase für die Decodierung berücksichtigt werden sollen oder später als reguläre Ein-Phasen-Transaktion zum Zeitpunkt von `COMMIT PREPARED`. Um zu signalisieren, dass die Decodierung übersprungen werden soll, geben Sie `true` zurück; andernfalls `false`. Wenn der Callback nicht definiert ist, wird `false` angenommen, das heißt keine Filterung; alle Transaktionen, die Zwei-Phasen-Commit verwenden, werden auch in zwei Phasen decodiert.

```c
typedef bool (*LogicalDecodeFilterPrepareCB) (struct LogicalDecodingContext *ctx,
                                              TransactionId xid,
                                              const char *gid);
```

Der Parameter `ctx` hat denselben Inhalt wie bei den anderen Callbacks. Die Parameter `xid` und `gid` bieten zwei verschiedene Möglichkeiten, die Transaktion zu identifizieren. Das spätere `COMMIT PREPARED` oder `ROLLBACK PREPARED` trägt beide Bezeichner, wodurch das Output-Plugin wählen kann, welchen es verwendet.

Der Callback kann pro zu decodierender Transaktion mehrfach aufgerufen werden und muss für ein gegebenes Paar aus `xid` und `gid` bei jedem Aufruf dieselbe statische Antwort liefern.

#### 47.6.4.10. Callback bei Beginn einer vorbereiteten Transaktion

Der erforderliche Callback `begin_prepare_cb` wird immer dann aufgerufen, wenn der Beginn einer vorbereiteten Transaktion decodiert wurde. Das Feld `gid`, das Teil des Parameters `txn` ist, kann in diesem Callback verwendet werden, um zu prüfen, ob das Plugin dieses `PREPARE` bereits erhalten hat. In diesem Fall kann es entweder einen Fehler auslösen oder die verbleibenden Änderungen der Transaktion überspringen.

```c
typedef void (*LogicalDecodeBeginPrepareCB) (struct LogicalDecodingContext *ctx,
                                             ReorderBufferTXN *txn);
```

#### 47.6.4.11. Callback beim Prepare einer Transaktion

Der erforderliche Callback `prepare_cb` wird immer dann aufgerufen, wenn eine Transaktion decodiert wurde, die für Zwei-Phasen-Commit vorbereitet ist. Der Callback `change_cb` wurde zuvor für alle geänderten Zeilen aufgerufen, sofern es geänderte Zeilen gab. Das Feld `gid`, das Teil des Parameters `txn` ist, kann in diesem Callback verwendet werden.

```c
typedef void (*LogicalDecodePrepareCB) (struct LogicalDecodingContext *ctx,
                                        ReorderBufferTXN *txn,
                                        XLogRecPtr prepare_lsn);
```

#### 47.6.4.12. Callback beim Commit einer vorbereiteten Transaktion

Der erforderliche Callback `commit_prepared_cb` wird immer dann aufgerufen, wenn ein `COMMIT PREPARED` einer Transaktion decodiert wurde. Das Feld `gid`, das Teil des Parameters `txn` ist, kann in diesem Callback verwendet werden.

```c
typedef void (*LogicalDecodeCommitPreparedCB) (struct LogicalDecodingContext *ctx,
                                               ReorderBufferTXN *txn,
                                               XLogRecPtr commit_lsn);
```

#### 47.6.4.13. Callback beim Rollback einer vorbereiteten Transaktion

Der erforderliche Callback `rollback_prepared_cb` wird immer dann aufgerufen, wenn ein `ROLLBACK PREPARED` einer Transaktion decodiert wurde. Das Feld `gid`, das Teil des Parameters `txn` ist, kann in diesem Callback verwendet werden. Die Parameter `prepare_end_lsn` und `prepare_time` können verwendet werden, um zu prüfen, ob das Plugin dieses `PREPARE TRANSACTION` erhalten hat; in diesem Fall kann es den Rollback anwenden, andernfalls kann es die Rollback-Operation überspringen. `gid` allein reicht nicht aus, weil der nachgelagerte Knoten eine vorbereitete Transaktion mit demselben Bezeichner haben kann.

```c
typedef void (*LogicalDecodeRollbackPreparedCB) (struct LogicalDecodingContext *ctx,
                                                 ReorderBufferTXN *txn,
                                                 XLogRecPtr prepare_end_lsn,
                                                 TimestampTz prepare_time);
```

#### 47.6.4.14. Stream-Start-Callback

Der erforderliche Callback `stream_start_cb` wird aufgerufen, wenn ein Block gestreamter Änderungen aus einer laufenden Transaktion geöffnet wird.

```c
typedef void (*LogicalDecodeStreamStartCB) (struct LogicalDecodingContext *ctx,
                                            ReorderBufferTXN *txn);
```

#### 47.6.4.15. Stream-Stop-Callback

Der erforderliche Callback `stream_stop_cb` wird aufgerufen, wenn ein Block gestreamter Änderungen aus einer laufenden Transaktion geschlossen wird.

```c
typedef void (*LogicalDecodeStreamStopCB) (struct LogicalDecodingContext *ctx,
                                           ReorderBufferTXN *txn);
```

#### 47.6.4.16. Stream-Abort-Callback

Der erforderliche Callback `stream_abort_cb` wird aufgerufen, um eine zuvor gestreamte Transaktion abzubrechen.

```c
typedef void (*LogicalDecodeStreamAbortCB) (struct LogicalDecodingContext *ctx,
                                            ReorderBufferTXN *txn,
                                            XLogRecPtr abort_lsn);
```

#### 47.6.4.17. Stream-Prepare-Callback

Der Callback `stream_prepare_cb` wird aufgerufen, um eine zuvor gestreamte Transaktion als Teil eines Zwei-Phasen-Commits vorzubereiten. Dieser Callback ist erforderlich, wenn das Output-Plugin sowohl das Streaming großer laufender Transaktionen als auch Zwei-Phasen-Commits unterstützt.

```c
typedef void (*LogicalDecodeStreamPrepareCB) (struct LogicalDecodingContext *ctx,
                                              ReorderBufferTXN *txn,
                                              XLogRecPtr prepare_lsn);
```

#### 47.6.4.18. Stream-Commit-Callback

Der erforderliche Callback `stream_commit_cb` wird aufgerufen, um eine zuvor gestreamte Transaktion zu committen.

```c
typedef void (*LogicalDecodeStreamCommitCB) (struct LogicalDecodingContext *ctx,
                                             ReorderBufferTXN *txn,
                                             XLogRecPtr commit_lsn);
```

#### 47.6.4.19. Stream-Change-Callback

Der erforderliche Callback `stream_change_cb` wird aufgerufen, wenn eine Änderung in einem Block gestreamter Änderungen gesendet wird, der durch Aufrufe von `stream_start_cb` und `stream_stop_cb` abgegrenzt ist. Die tatsächlichen Änderungen werden nicht angezeigt, da die Transaktion zu einem späteren Zeitpunkt abbrechen kann und Änderungen für abgebrochene Transaktionen nicht decodiert werden.

```c
typedef void (*LogicalDecodeStreamChangeCB) (struct LogicalDecodingContext *ctx,
                                             ReorderBufferTXN *txn,
                                             Relation relation,
                                             ReorderBufferChange *change);
```

#### 47.6.4.20. Stream-Message-Callback

Der optionale Callback `stream_message_cb` wird aufgerufen, wenn eine generische Nachricht in einem Block gestreamter Änderungen gesendet wird, der durch Aufrufe von `stream_start_cb` und `stream_stop_cb` abgegrenzt ist. Der Inhalt transaktionaler Nachrichten wird nicht angezeigt, da die Transaktion zu einem späteren Zeitpunkt abbrechen kann und Änderungen für abgebrochene Transaktionen nicht decodiert werden.

```c
typedef void (*LogicalDecodeStreamMessageCB) (struct LogicalDecodingContext *ctx,
                                              ReorderBufferTXN *txn,
                                              XLogRecPtr message_lsn,
                                              bool transactional,
                                              const char *prefix,
                                              Size message_size,
                                              const char *message);
```

#### 47.6.4.21. Stream-Truncate-Callback

Der optionale Callback `stream_truncate_cb` wird für einen `TRUNCATE`-Befehl in einem Block gestreamter Änderungen aufgerufen, der durch Aufrufe von `stream_start_cb` und `stream_stop_cb` abgegrenzt ist.

```c
typedef void (*LogicalDecodeStreamTruncateCB) (struct LogicalDecodingContext *ctx,
                                               ReorderBufferTXN *txn,
                                               int nrelations,
                                               Relation relations[],
                                               ReorderBufferChange *change);
```

Die Parameter entsprechen denen des Callbacks `stream_change_cb`. Da `TRUNCATE`-Aktionen auf Tabellen, die durch Fremdschlüssel verbunden sind, jedoch gemeinsam ausgeführt werden müssen, erhält dieser Callback ein Array von Relationen statt nur einer einzelnen Relation. Einzelheiten finden Sie in der Beschreibung der Anweisung `TRUNCATE`.

### 47.6.5. Funktionen zum Erzeugen von Ausgabe

Um tatsächlich Ausgabe zu erzeugen, können Output-Plugins innerhalb der Callbacks `begin_cb`, `commit_cb` oder `change_cb` Daten in den `StringInfo`-Ausgabepuffer in `ctx->out` schreiben. Vor dem Schreiben in den Ausgabepuffer muss `OutputPluginPrepareWrite(ctx, last_write)` aufgerufen werden, und nach Abschluss des Schreibens in den Puffer muss `OutputPluginWrite(ctx, last_write)` aufgerufen werden, um den Schreibvorgang auszuführen. `last_write` gibt an, ob ein bestimmter Schreibvorgang der letzte Schreibvorgang des Callbacks war.

Das folgende Beispiel zeigt, wie Daten an den Verbraucher eines Output-Plugins ausgegeben werden:

```c
OutputPluginPrepareWrite(ctx, true);
appendStringInfo(ctx->out, "BEGIN %u", txn->xid);
OutputPluginWrite(ctx, true);
```

## 47.7. Output-Writer für logische Decodierung

Es ist möglich, weitere Ausgabemethoden für logische Decodierung hinzuzufügen. Einzelheiten finden Sie in `src/backend/replication/logical/logicalfuncs.c`. Im Wesentlichen müssen drei Funktionen bereitgestellt werden: eine zum Lesen des WAL, eine zum Vorbereiten des Schreibens der Ausgabe und eine zum Schreiben der Ausgabe; siehe Abschnitt 47.6.5.

## 47.8. Unterstützung synchroner Replikation für logische Decodierung

### 47.8.1. Überblick

Logische Decodierung kann verwendet werden, um synchrone Replikationslösungen mit derselben Benutzerschnittstelle wie die synchrone Replikation für Streaming-Replikation zu bauen. Dazu muss die Streaming-Replikationsschnittstelle verwendet werden, um Daten auszugeben; siehe [Abschnitt 47.3](#473-schnittstelle-des-streamingreplikationsprotokolls). Clients müssen, genau wie Streaming-Replikationsclients, Nachrichten vom Typ Standby status update (`F`) senden; siehe [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll).

**Hinweis:** Eine synchrone Replika, die Änderungen über logische Decodierung empfängt, arbeitet im Geltungsbereich einer einzelnen Datenbank. Da `synchronous_standby_names` im Gegensatz dazu derzeit serverweit gilt, funktioniert diese Technik nicht korrekt, wenn mehr als eine Datenbank aktiv verwendet wird.

### 47.8.2. Einschränkungen

In einer Einrichtung mit synchroner Replikation kann ein Deadlock auftreten, wenn die Transaktion Benutzerkatalogtabellen exklusiv gesperrt hat. Informationen zu Benutzerkatalogtabellen finden Sie in Abschnitt 47.6.2. Der Grund ist, dass die logische Decodierung von Transaktionen Katalogtabellen sperren kann, um auf sie zuzugreifen. Um dies zu vermeiden, müssen Benutzer davon absehen, exklusive Sperren auf Benutzerkatalogtabellen zu nehmen. Dies kann auf folgende Weise geschehen:

- Ausführen eines expliziten `LOCK` auf `pg_class` in einer Transaktion.
- Ausführen von `CLUSTER` auf `pg_class` in einer Transaktion.
- `PREPARE TRANSACTION` nach einem `LOCK`-Befehl auf `pg_class` und Zulassen logischer Decodierung von Zwei-Phasen-Transaktionen.
- `PREPARE TRANSACTION` nach einem `CLUSTER`-Befehl auf `pg_trigger` und Zulassen logischer Decodierung von Zwei-Phasen-Transaktionen. Dies führt nur dann zu einem Deadlock, wenn die publizierte Tabelle einen Trigger hat.
- Ausführen von `TRUNCATE` auf einer Benutzerkatalogtabelle in einer Transaktion.

Beachten Sie, dass diese Befehle Deadlocks nicht nur für die oben aufgeführten Systemkatalogtabellen verursachen können, sondern auch für andere Katalogtabellen.

## 47.9. Streaming großer Transaktionen für logische Decodierung

Die grundlegenden Output-Plugin-Callbacks (z. B. `begin_cb`, `change_cb`, `commit_cb` und `message_cb`) werden erst aufgerufen, wenn die Transaktion tatsächlich committet. Die Änderungen werden weiterhin aus dem Transaktionslog decodiert, aber erst beim Commit an das Output-Plugin übergeben (und verworfen, wenn die Transaktion abbricht).

Das bedeutet: Obwohl die Decodierung inkrementell erfolgt und gegebenenfalls auf Platte ausgelagert wird, um den Speicherverbrauch unter Kontrolle zu halten, müssen alle decodierten Änderungen übertragen werden, wenn die Transaktion schließlich committet (genauer gesagt, wenn der Commit aus dem Transaktionslog decodiert wird). Je nach Größe der Transaktion und Netzwerkbandbreite kann die Übertragungszeit den Apply-Lag erheblich erhöhen.

Um den durch große Transaktionen verursachten Apply-Lag zu verringern, kann ein Output-Plugin zusätzliche Callbacks bereitstellen, die inkrementelles Streaming laufender Transaktionen unterstützen. Es gibt mehrere erforderliche Streaming-Callbacks (`stream_start_cb`, `stream_stop_cb`, `stream_abort_cb`, `stream_commit_cb` und `stream_change_cb`) sowie zwei optionale Callbacks (`stream_message_cb` und `stream_truncate_cb`). Wenn außerdem Streaming von Two-Phase-Befehlen unterstützt werden soll, müssen weitere Callbacks bereitgestellt werden. (Einzelheiten finden Sie in [Abschnitt 47.10](47_Logische_Decodierung.md#4710-unterstützung-von-twophasecommits-für-logische-decodierung).)

Beim Streaming einer laufenden Transaktion werden die Änderungen (und Nachrichten) in Blöcken gestreamt, die durch die Callbacks `stream_start_cb` und `stream_stop_cb` begrenzt werden. Sobald alle decodierten Änderungen übertragen wurden, kann die Transaktion mit dem Callback `stream_commit_cb` committet (oder möglicherweise mit dem Callback `stream_abort_cb` abgebrochen) werden. Wenn Two-Phase-Commits unterstützt werden, kann die Transaktion mit dem Callback `stream_prepare_cb` vorbereitet, mit dem Callback `commit_prepared_cb` per `COMMIT PREPARED` committet oder mit `rollback_prepared_cb` abgebrochen werden.

Eine beispielhafte Abfolge von Streaming-Callback-Aufrufen für eine Transaktion könnte so aussehen:

```text
stream_start_cb(...);    <-- start of first block of changes
  stream_change_cb(...);
  stream_change_cb(...);
  stream_message_cb(...);
  stream_change_cb(...);
  ...
  stream_change_cb(...);
stream_stop_cb(...);     <-- end of first block of changes

stream_start_cb(...);    <-- start of second block of changes
  stream_change_cb(...);
  stream_change_cb(...);
  stream_change_cb(...);
  ...
  stream_message_cb(...);
  stream_change_cb(...);
stream_stop_cb(...);     <-- end of second block of changes

[a. when using normal commit]
stream_commit_cb(...);    <-- commit of the streamed transaction

[b. when using two-phase commit]
stream_prepare_cb(...);   <-- prepare the streamed transaction
commit_prepared_cb(...); <-- commit of the prepared transaction
```

Die tatsächliche Abfolge der Callback-Aufrufe kann natürlich komplizierter sein. Es kann Blöcke für mehrere gestreamte Transaktionen geben, manche Transaktionen können abgebrochen werden usw.

Ähnlich wie beim Verhalten des Auslagerns auf Platte wird Streaming ausgelöst, wenn die Gesamtmenge der aus dem WAL decodierten Änderungen (für alle laufenden Transaktionen) den durch die Einstellung `logical_decoding_work_mem` definierten Grenzwert überschreitet. An diesem Punkt wird die größte Top-Level-Transaktion (gemessen an der Menge des aktuell für decodierte Änderungen verwendeten Speichers) ausgewählt und gestreamt. In manchen Fällen müssen wir jedoch auch bei aktiviertem Streaming weiterhin auf Platte auslagern, weil der Speicherschwellwert überschritten wird, das vollständige Tupel aber noch nicht decodiert wurde, z. B. wenn nur der Insert in die TOAST-Tabelle decodiert wurde, aber noch nicht der Insert in die Haupttabelle.

Auch beim Streaming großer Transaktionen werden die Änderungen weiterhin in Commit-Reihenfolge angewendet, sodass dieselben Garantien wie im Nicht-Streaming-Modus erhalten bleiben.

## 47.10. Unterstützung von Two-Phase-Commits für logische Decodierung

Mit den grundlegenden Output-Plugin-Callbacks (z. B. `begin_cb`, `change_cb`, `commit_cb` und `message_cb`) werden Two-Phase-Commit-Befehle wie `PREPARE TRANSACTION`, `COMMIT PREPARED` und `ROLLBACK PREPARED` nicht decodiert. Während `PREPARE TRANSACTION` ignoriert wird, wird `COMMIT PREPARED` als `COMMIT` und `ROLLBACK PREPARED` als `ROLLBACK` decodiert.

Um das Streaming von Two-Phase-Befehlen zu unterstützen, muss ein Output-Plugin zusätzliche Callbacks bereitstellen. Es gibt mehrere erforderliche Two-Phase-Commit-Callbacks (`begin_prepare_cb`, `prepare_cb`, `commit_prepared_cb`, `rollback_prepared_cb` und `stream_prepare_cb`) sowie einen optionalen Callback (`filter_prepare_cb`).

Wenn die Output-Plugin-Callbacks für die Decodierung von Two-Phase-Commit-Befehlen bereitgestellt werden, werden bei `PREPARE TRANSACTION` die Änderungen dieser Transaktion decodiert, an das Output-Plugin übergeben, und der Callback `prepare_cb` wird aufgerufen. Dies unterscheidet sich von der grundlegenden Decodierungseinrichtung, bei der Änderungen erst dann an das Output-Plugin übergeben werden, wenn eine Transaktion committet wird. Der Beginn einer vorbereiteten Transaktion wird durch den Callback `begin_prepare_cb` angezeigt.

Wenn eine vorbereitete Transaktion mit `ROLLBACK PREPARED` zurückgerollt wird, wird der Callback `rollback_prepared_cb` aufgerufen; wenn die vorbereitete Transaktion mit `COMMIT PREPARED` committet wird, wird der Callback `commit_prepared_cb` aufgerufen.

Optional kann das Output-Plugin über `filter_prepare_cb` Filterregeln definieren, um nur bestimmte Transaktionen in zwei Phasen zu decodieren. Dies kann durch Pattern-Matching auf der `gid` oder über Nachschlagen mithilfe der `xid` erreicht werden.

Benutzer, die vorbereitete Transaktionen decodieren möchten, müssen die folgenden Punkte beachten:

- Wenn die vorbereitete Transaktion Benutzerkatalogtabellen exklusiv gesperrt hat, kann die Decodierung des Prepare blockieren, bis die Haupttransaktion committet wird.

- Die logische Replikationslösung, die mit dieser Funktion verteilten Two-Phase-Commit aufbaut, kann in einen Deadlock geraten, wenn die vorbereitete Transaktion Benutzerkatalogtabellen exklusiv gesperrt hat. Um dies zu vermeiden, müssen Benutzer in solchen Transaktionen auf Sperren von Katalogtabellen verzichten (z. B. durch einen expliziten `LOCK`-Befehl). Einzelheiten finden Sie in [Abschnitt 47.8.2](#4782-einschränkungen).
