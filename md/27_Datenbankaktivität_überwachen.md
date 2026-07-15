# 27. Datenbankaktivität überwachen

Ein Datenbankadministrator fragt sich häufig: „Was macht das System gerade?“ Dieses Kapitel beschreibt, wie man das herausfindet.

Für die Überwachung der Datenbankaktivität und die Analyse der Performance stehen mehrere Werkzeuge zur Verfügung. Der größte Teil dieses Kapitels beschreibt PostgreSQLs kumulatives Statistiksystem, aber normale Unix-Überwachungsprogramme wie `ps`, `top`, `iostat` und `vmstat` sollte man nicht vernachlässigen. Wenn eine schlecht laufende Abfrage identifiziert wurde, kann außerdem weitere Untersuchung mit PostgreSQLs Befehl `EXPLAIN` nötig sein. [Abschnitt 14.1](14_Hinweise_zur_Performance.md#141-explain-verwenden) behandelt `EXPLAIN` und andere Methoden, um das Verhalten einer einzelnen Abfrage zu verstehen.

## 27.1. Standard-Unix-Werkzeuge

Auf den meisten Unix-Plattformen ändert PostgreSQL den von `ps` gemeldeten Befehlstitel, sodass einzelne Serverprozesse leicht identifiziert werden können. Eine Beispielausgabe ist:

```text
$ ps auxww | grep ^postgres
postgres 15551 0.0 0.1 57536 7132 pts/0          S    18:02                                    0:00 postgres -i
postgres 15554 0.0 0.0 57536 1184 ?              Ss   18:02                                    0:00 postgres: background writer
postgres 15555 0.0 0.0 57536 916 ?               Ss   18:02                                    0:00 postgres: checkpointer
postgres 15556 0.0 0.0 57536 916 ?               Ss   18:02                                    0:00 postgres: walwriter
postgres 15557 0.0 0.0 58504 2244 ?              Ss   18:02                                    0:00 postgres: autovacuum launcher
postgres 15582 0.0 0.0 58772 3080 ?              Ss   18:04                                    0:00 postgres: joe runbug 127.0.0.1 idle
postgres 15606 0.0 0.0 58772 3052 ?              Ss   18:07                                    0:00 postgres: tgl regression [local] SELECT waiting
postgres 15610 0.0 0.0 58772 3056 ?              Ss   18:07                                    0:00 postgres: tgl regression [local] idle in transaction
```

Der passende Aufruf von `ps` unterscheidet sich je nach Plattform, ebenso die Details der angezeigten Informationen. Dieses Beispiel stammt von einem neueren Linux-System. Der erste hier aufgelistete Prozess ist der primäre Serverprozess. Die für ihn angezeigten Befehlsargumente sind dieselben, mit denen er gestartet wurde. Die nächsten vier Prozesse sind Hintergrundprozesse, die automatisch vom primären Prozess gestartet wurden. Der Prozess `autovacuum launcher` ist nicht vorhanden, wenn das System so eingestellt wurde, dass Autovacuum nicht läuft. Jeder der übrigen Prozesse ist ein Serverprozess, der eine Clientverbindung bearbeitet. Jeder dieser Prozesse setzt seine Kommandozeilenanzeige in der Form:

```text
postgres: user database host activity
```

Die Einträge für Benutzer, Datenbank und Client-Host bleiben während der Lebensdauer der Clientverbindung gleich, aber der Aktivitätsindikator ändert sich. Die Aktivität kann `idle` sein, also Warten auf einen Clientbefehl, `idle in transaction`, also Warten auf den Client innerhalb eines `BEGIN`-Blocks, oder ein Befehlstypname wie `SELECT`. Außerdem wird `waiting` angehängt, wenn der Serverprozess derzeit auf eine Sperre wartet, die von einer anderen Sitzung gehalten wird. Im obigen Beispiel lässt sich schließen, dass Prozess 15606 darauf wartet, dass Prozess 15610 seine Transaktion abschließt und dadurch eine Sperre freigibt. Prozess 15610 muss der blockierende Prozess sein, weil es keine andere aktive Sitzung gibt. In komplizierteren Fällen müsste man die Systemsicht `pg_locks` untersuchen, um festzustellen, wer wen blockiert.

Wenn `cluster_name` konfiguriert wurde, erscheint der Clustername ebenfalls in der `ps`-Ausgabe:

```text
$ psql -c 'SHOW cluster_name'
 cluster_name
--------------
 server1
(1 row)

$ ps aux|grep server1
postgres   27093 0.0 0.0 30096 2752 ?  Ss  11:34  0:00 postgres: server1: background writer
...
```

Wenn Sie `update_process_title` ausgeschaltet haben, wird der Aktivitätsindikator nicht aktualisiert; der Prozesstitel wird nur einmal gesetzt, wenn ein neuer Prozess gestartet wird. Auf manchen Plattformen spart dies einen messbaren Betrag an Aufwand pro Befehl, auf anderen ist der Unterschied unbedeutend.

> **Tipp**
>
> Solaris erfordert Sonderbehandlung. Sie müssen `/usr/ucb/ps` statt `/bin/ps` verwenden. Außerdem müssen Sie zwei `w`-Schalter verwenden, nicht nur einen. Zusätzlich muss Ihr ursprünglicher Aufruf des Befehls `postgres` eine kürzere `ps`-Statusanzeige haben als die, die jeder Serverprozess bereitstellt. Wenn eine dieser drei Bedingungen nicht erfüllt ist, zeigt `ps` für jeden Serverprozess die ursprüngliche `postgres`-Kommandozeile an.

## 27.2. Das kumulative Statistiksystem

PostgreSQLs kumulatives Statistiksystem unterstützt das Sammeln und Melden von Informationen über Serveraktivität. Derzeit werden Zugriffe auf Tabellen und Indizes sowohl in Plattenblöcken als auch in einzelnen Zeilen gezählt. Die Gesamtzahl der Zeilen in jeder Tabelle sowie Informationen über Vacuum- und Analyze-Aktionen für jede Tabelle werden ebenfalls gezählt. Wenn aktiviert, werden auch Aufrufe benutzerdefinierter Funktionen und die insgesamt in jeder Funktion verbrachte Zeit gezählt.

PostgreSQL unterstützt außerdem das Melden dynamischer Informationen darüber, was gerade im System geschieht, etwa welcher exakte Befehl gerade von anderen Serverprozessen ausgeführt wird und welche anderen Verbindungen im System existieren. Diese Funktion ist unabhängig vom kumulativen Statistiksystem.

### 27.2.1. Konfiguration der Statistiksammlung

Da das Sammeln von Statistiken zusätzlichen Aufwand bei der Abfrageausführung verursacht, kann das System so konfiguriert werden, dass es Informationen sammelt oder nicht sammelt. Dies wird durch Konfigurationsparameter gesteuert, die normalerweise in `postgresql.conf` gesetzt werden. Details zum Setzen von Konfigurationsparametern finden Sie in [Kapitel 19](19_Serverkonfiguration.md).

Der Parameter `track_activities` aktiviert die Überwachung des aktuellen Befehls, der von einem Serverprozess ausgeführt wird.

Der Parameter `track_cost_delay_timing` aktiviert die Überwachung der kostenbasierten Vacuum-Verzögerung.

Der Parameter `track_counts` steuert, ob kumulative Statistiken über Tabellen- und Indexzugriffe gesammelt werden.

Der Parameter `track_functions` aktiviert das Nachverfolgen der Nutzung benutzerdefinierter Funktionen.

Der Parameter `track_io_timing` aktiviert die Überwachung von Zeiten für Block-Lesen, -Schreiben, -Erweitern und `fsync`.

Der Parameter `track_wal_io_timing` aktiviert die Überwachung von Zeiten für WAL-Lesen, -Schreiben und `fsync`.

Normalerweise werden diese Parameter in `postgresql.conf` gesetzt, sodass sie für alle Serverprozesse gelten. Es ist aber möglich, sie in einzelnen Sitzungen mit dem Befehl `SET` ein- oder auszuschalten. Damit gewöhnliche Benutzer ihre Aktivität nicht vor dem Administrator verbergen können, dürfen nur Superuser diese Parameter mit `SET` ändern.

Kumulative Statistiken werden im Shared Memory gesammelt. Jeder PostgreSQL-Prozess sammelt Statistiken lokal und aktualisiert die gemeinsamen Daten in passenden Intervallen. Wenn ein Server, einschließlich eines physischen Replikats, sauber herunterfährt, wird eine dauerhafte Kopie der Statistikdaten im Unterverzeichnis `pg_stat` gespeichert, sodass Statistiken über Serverneustarts hinweg erhalten bleiben können. Beim Start nach einem unsauberen Herunterfahren, etwa nach einem Immediate Shutdown, einem Serverabsturz, dem Start aus einer Basissicherung oder Point-in-Time-Recovery, werden dagegen alle Statistikzähler zurückgesetzt.

### 27.2.2. Statistiken anzeigen

Mehrere vordefinierte Sichten, die in Tabelle 27.1 aufgeführt sind, zeigen den aktuellen Zustand des Systems. Weitere Sichten, die in Tabelle 27.2 aufgeführt sind, zeigen die gesammelten Statistiken. Alternativ kann man eigene Sichten auf Basis der zugrunde liegenden kumulativen Statistikfunktionen erstellen, wie in [Abschnitt 27.2.26](#27226-statistikfunktionen) erläutert.

Bei der Verwendung kumulativer Statistik-Sichten und -Funktionen zur Überwachung gesammelter Daten ist wichtig zu wissen, dass die Informationen nicht augenblicklich aktualisiert werden. Jeder einzelne Serverprozess schreibt angesammelte Statistiken kurz vor dem Leerlauf in den Shared Memory, jedoch nicht häufiger als einmal pro `PGSTAT_MIN_INTERVAL` Millisekunden, also 1 Sekunde, sofern dies beim Bau des Servers nicht geändert wurde. Eine noch laufende Abfrage oder Transaktion beeinflusst die angezeigten Summen daher nicht, und die angezeigten Informationen hinken der tatsächlichen Aktivität hinterher. Die von `track_activities` gesammelten Informationen zur aktuellen Abfrage sind dagegen immer aktuell.

Ein weiterer wichtiger Punkt ist, dass in der Standardkonfiguration die abgerufenen Werte bis zum Ende der aktuellen Transaktion zwischengespeichert werden, wenn ein Serverprozess gebeten wird, kumulative Statistiken anzuzeigen. Die Statistiken zeigen also statische Informationen, solange die aktuelle Transaktion läuft. Ebenso werden Informationen über die aktuellen Abfragen aller Sitzungen gesammelt, sobald solche Informationen innerhalb einer Transaktion erstmals angefordert werden, und dieselben Informationen werden während der gesamten Transaktion angezeigt. Das ist ein Feature, kein Fehler, weil es erlaubt, mehrere Abfragen auf die Statistiken auszuführen und die Ergebnisse miteinander zu korrelieren, ohne dass sich die Zahlen unter Ihnen verändern. Bei interaktiver Statistik-Analyse oder teuren Abfragen kann die Zeitdifferenz zwischen Zugriffen auf einzelne Statistiken zu deutlicher Verzerrung in den zwischengespeicherten Statistiken führen. Um Verzerrungen zu minimieren, kann `stats_fetch_consistency` auf `snapshot` gesetzt werden, zum Preis erhöhten Speicherverbrauchs für das Zwischenspeichern nicht benötigter Statistikdaten. Umgekehrt ist das Zwischenspeichern abgerufener Statistiken unnötig, wenn bekannt ist, dass Statistiken nur einmal abgerufen werden; es kann vermieden werden, indem `stats_fetch_consistency` auf `none` gesetzt wird. Sie können `pg_stat_clear_snapshot()` aufrufen, um den aktuellen Statistik-Snapshot der Transaktion oder zwischengespeicherte Werte, falls vorhanden, zu verwerfen. Die nächste Verwendung statistischer Informationen erstellt im Snapshot-Modus einen neuen Snapshot oder speichert im Cache-Modus die abgerufenen Statistiken zwischen.

Eine Transaktion kann ihre eigenen, noch nicht in die Shared-Memory-Statistiken geschriebenen Statistiken auch in den Sichten `pg_stat_xact_all_tables`, `pg_stat_xact_sys_tables`, `pg_stat_xact_user_tables` und `pg_stat_xact_user_functions` sehen. Diese Zahlen verhalten sich nicht wie oben beschrieben; stattdessen aktualisieren sie sich kontinuierlich während der Transaktion.

Einige Informationen in den dynamischen Statistik-Sichten aus Tabelle 27.1 sind aus Sicherheitsgründen eingeschränkt. Gewöhnliche Benutzer können alle Informationen nur über ihre eigenen Sitzungen sehen, also Sitzungen, die zu einer Rolle gehören, deren Mitglied sie sind. In Zeilen zu anderen Sitzungen sind viele Spalten `NULL`. Die Existenz einer Sitzung und ihre allgemeinen Eigenschaften wie Sitzungsbenutzer und Datenbank sind jedoch für alle Benutzer sichtbar. Superuser und Rollen mit den Privilegien der eingebauten Rolle `pg_read_all_stats` können alle Informationen über alle Sitzungen sehen.

**Tabelle 27.1. Dynamische Statistik-Sichten**

| Sichtname | Beschreibung |
|---|---|
| `pg_stat_activity` | Eine Zeile pro Serverprozess; zeigt Informationen zur aktuellen Aktivität dieses Prozesses, etwa Zustand und aktuelle Abfrage. Details siehe `pg_stat_activity`. |
| `pg_stat_replication` | Eine Zeile pro WAL-Sender-Prozess; zeigt Statistiken zur Replikation an den mit diesem Sender verbundenen Standby-Server. Details siehe `pg_stat_replication`. |
| `pg_stat_wal_receiver` | Genau eine Zeile; zeigt Statistiken über den WAL-Receiver von dessen verbundenem Server. Details siehe `pg_stat_wal_receiver`. |
| `pg_stat_recovery_prefetch` | Genau eine Zeile; zeigt Statistiken über während der Recovery vorab geladene Blöcke. Details siehe `pg_stat_recovery_prefetch`. |
| `pg_stat_subscription` | Mindestens eine Zeile pro Subscription; zeigt Informationen über die Subscription-Worker. Details siehe `pg_stat_subscription`. |
| `pg_stat_ssl` | Eine Zeile pro Verbindung, regulär oder Replikation; zeigt Informationen über auf dieser Verbindung verwendetes SSL. Details siehe `pg_stat_ssl`. |
| `pg_stat_gssapi` | Eine Zeile pro Verbindung, regulär oder Replikation; zeigt Informationen über GSSAPI-Authentifizierung und -Verschlüsselung auf dieser Verbindung. Details siehe `pg_stat_gssapi`. |
| `pg_stat_progress_analyze` | Eine Zeile für jedes Backend, einschließlich Autovacuum-Worker-Prozessen, das `ANALYZE` ausführt; zeigt den aktuellen Fortschritt. Siehe [Abschnitt 27.4.1](#2741-fortschrittsanzeige-für-analyze). |
| `pg_stat_progress_create_index` | Eine Zeile für jedes Backend, das `CREATE INDEX` oder `REINDEX` ausführt; zeigt den aktuellen Fortschritt. Siehe [Abschnitt 27.4.4](#2744-fortschrittsanzeige-für-create-index). |
| `pg_stat_progress_vacuum` | Eine Zeile für jedes Backend, einschließlich Autovacuum-Worker-Prozessen, das `VACUUM` ausführt; zeigt den aktuellen Fortschritt. Siehe [Abschnitt 27.4.5](#2745-fortschrittsanzeige-für-vacuum). |
| `pg_stat_progress_cluster` | Eine Zeile für jedes Backend, das `CLUSTER` oder `VACUUM FULL` ausführt; zeigt den aktuellen Fortschritt. Siehe [Abschnitt 27.4.2](#2742-fortschrittsanzeige-für-cluster). |
| `pg_stat_progress_basebackup` | Eine Zeile für jeden WAL-Sender-Prozess, der eine Basissicherung streamt; zeigt den aktuellen Fortschritt. Siehe [Abschnitt 27.4.6](#2746-fortschrittsanzeige-für-basissicherungen). |
| `pg_stat_progress_copy` | Eine Zeile für jedes Backend, das `COPY` ausführt; zeigt den aktuellen Fortschritt. Siehe [Abschnitt 27.4.3](#2743-fortschrittsanzeige-für-copy). |

**Tabelle 27.2. Gesammelte Statistik-Sichten**

| Sichtname | Beschreibung |
|---|---|
| `pg_stat_archiver` | Genau eine Zeile; zeigt Statistiken über die Aktivität des WAL-Archiver-Prozesses. Details siehe `pg_stat_archiver`. |
| `pg_stat_bgwriter` | Genau eine Zeile; zeigt Statistiken über die Aktivität des Background-Writer-Prozesses. Details siehe `pg_stat_bgwriter`. |
| `pg_stat_checkpointer` | Genau eine Zeile; zeigt Statistiken über die Aktivität des Checkpointer-Prozesses. Details siehe `pg_stat_checkpointer`. |
| `pg_stat_database` | Eine Zeile pro Datenbank; zeigt datenbankweite Statistiken. Details siehe `pg_stat_database`. |
| `pg_stat_database_conflicts` | Eine Zeile pro Datenbank; zeigt datenbankweite Statistiken über Abfrageabbrüche aufgrund von Konflikten mit Recovery auf Standby-Servern. Details siehe `pg_stat_database_conflicts`. |
| `pg_stat_io` | Eine Zeile für jede Kombination aus Backend-Typ, Kontext und Zielobjekt mit clusterweiten I/O-Statistiken. Details siehe `pg_stat_io`. |
| `pg_stat_replication_slots` | Eine Zeile pro Replikations-Slot; zeigt Statistiken über die Nutzung des Replikations-Slots. Details siehe `pg_stat_replication_slots`. |
| `pg_stat_slru` | Eine Zeile pro SLRU; zeigt Operationsstatistiken. Details siehe `pg_stat_slru`. |
| `pg_stat_subscription_stats` | Eine Zeile pro Subscription; zeigt Statistiken über Fehler und Konflikte. Details siehe `pg_stat_subscription_stats`. |
| `pg_stat_wal` | Genau eine Zeile; zeigt Statistiken über WAL-Aktivität. Details siehe `pg_stat_wal`. |
| `pg_stat_all_tables` | Eine Zeile für jede Tabelle in der aktuellen Datenbank; zeigt Statistiken über Zugriffe auf diese Tabelle. Details siehe `pg_stat_all_tables`. |
| `pg_stat_sys_tables` | Wie `pg_stat_all_tables`, zeigt aber nur Systemtabellen. |
| `pg_stat_user_tables` | Wie `pg_stat_all_tables`, zeigt aber nur Benutzertabellen. |
| `pg_stat_xact_all_tables` | Ähnlich wie `pg_stat_all_tables`, zählt aber Aktionen, die bisher innerhalb der aktuellen Transaktion ausgeführt wurden und noch nicht in `pg_stat_all_tables` und verwandten Sichten enthalten sind. Die Spalten für die Anzahl lebender und toter Zeilen sowie Vacuum- und Analyze-Aktionen sind in dieser Sicht nicht vorhanden. |
| `pg_stat_xact_sys_tables` | Wie `pg_stat_xact_all_tables`, zeigt aber nur Systemtabellen. |
| `pg_stat_xact_user_tables` | Wie `pg_stat_xact_all_tables`, zeigt aber nur Benutzertabellen. |
| `pg_stat_all_indexes` | Eine Zeile für jeden Index in der aktuellen Datenbank; zeigt Statistiken über Zugriffe auf diesen Index. Details siehe `pg_stat_all_indexes`. |
| `pg_stat_sys_indexes` | Wie `pg_stat_all_indexes`, zeigt aber nur Indizes auf Systemtabellen. |
| `pg_stat_user_indexes` | Wie `pg_stat_all_indexes`, zeigt aber nur Indizes auf Benutzertabellen. |
| `pg_stat_user_functions` | Eine Zeile für jede nachverfolgte Funktion; zeigt Statistiken über Ausführungen dieser Funktion. Details siehe `pg_stat_user_functions`. |
| `pg_stat_xact_user_functions` | Ähnlich wie `pg_stat_user_functions`, zählt aber nur Aufrufe während der aktuellen Transaktion, die noch nicht in `pg_stat_user_functions` enthalten sind. |
| `pg_statio_all_tables` | Eine Zeile für jede Tabelle in der aktuellen Datenbank; zeigt I/O-Statistiken zu dieser Tabelle. Details siehe `pg_statio_all_tables`. |
| `pg_statio_sys_tables` | Wie `pg_statio_all_tables`, zeigt aber nur Systemtabellen. |
| `pg_statio_user_tables` | Wie `pg_statio_all_tables`, zeigt aber nur Benutzertabellen. |
| `pg_statio_all_indexes` | Eine Zeile für jeden Index in der aktuellen Datenbank; zeigt I/O-Statistiken zu diesem Index. Details siehe `pg_statio_all_indexes`. |
| `pg_statio_sys_indexes` | Wie `pg_statio_all_indexes`, zeigt aber nur Indizes auf Systemtabellen. |
| `pg_statio_user_indexes` | Wie `pg_statio_all_indexes`, zeigt aber nur Indizes auf Benutzertabellen. |
| `pg_statio_all_sequences` | Eine Zeile für jede Sequenz in der aktuellen Datenbank; zeigt I/O-Statistiken zu dieser Sequenz. Details siehe `pg_statio_all_sequences`. |
| `pg_statio_sys_sequences` | Wie `pg_statio_all_sequences`, zeigt aber nur Systemsequenzen. Derzeit sind keine Systemsequenzen definiert, daher ist diese Sicht immer leer. |
| `pg_statio_user_sequences` | Wie `pg_statio_all_sequences`, zeigt aber nur Benutzersequenzen. |

Die Statistiken pro Index sind besonders nützlich, um zu bestimmen, welche Indizes verwendet werden und wie effektiv sie sind.

Die Sichten `pg_stat_io` und die `pg_statio_`-Sichten sind nützlich, um die Wirksamkeit des Buffer Cache zu bestimmen. Sie können verwendet werden, um eine Cache-Hit-Rate zu berechnen. Beachten Sie, dass PostgreSQLs I/O-Statistiken zwar die meisten Fälle erfassen, in denen der Kernel zur Durchführung von I/O aufgerufen wurde, aber nicht zwischen Daten unterscheiden, die von der Platte gelesen werden mussten, und Daten, die bereits im Kernel-Page-Cache lagen. Benutzer sollten die PostgreSQL-Statistiksichten zusammen mit Betriebssystemwerkzeugen verwenden, um ein vollständigeres Bild der I/O-Performance ihrer Datenbank zu erhalten.

### 27.2.3. pg_stat_activity

Die Sicht `pg_stat_activity` enthält eine Zeile pro Serverprozess und zeigt Informationen zur aktuellen Aktivität dieses Prozesses.

**Tabelle 27.3. Sicht `pg_stat_activity`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `datid` | `oid` | OID der Datenbank, mit der dieses Backend verbunden ist. |
| `datname` | `name` | Name der Datenbank, mit der dieses Backend verbunden ist. |
| `pid` | `integer` | Prozess-ID dieses Backends. |
| `leader_pid` | `integer` | Prozess-ID des Parallel-Group-Leaders, wenn dieser Prozess ein Parallel-Query-Worker ist, oder Prozess-ID des Leader-Apply-Workers, wenn dieser Prozess ein Parallel-Apply-Worker ist. `NULL` bedeutet, dass dieser Prozess ein Parallel-Group-Leader oder Leader-Apply-Worker ist oder an keiner parallelen Operation teilnimmt. |
| `usesysid` | `oid` | OID des Benutzers, der in diesem Backend angemeldet ist. |
| `usename` | `name` | Name des Benutzers, der in diesem Backend angemeldet ist. |
| `application_name` | `text` | Name der Anwendung, die mit diesem Backend verbunden ist. |
| `client_addr` | `inet` | IP-Adresse des Clients, der mit diesem Backend verbunden ist. Ist dieses Feld `NULL`, ist der Client entweder über einen Unix-Socket auf der Servermaschine verbunden, oder es handelt sich um einen internen Prozess wie Autovacuum. |
| `client_hostname` | `text` | Hostname des verbundenen Clients, wie er durch einen Reverse-DNS-Lookup von `client_addr` gemeldet wird. Dieses Feld ist nur bei IP-Verbindungen ungleich `NULL` und nur dann, wenn `log_hostname` aktiviert ist. |
| `client_port` | `integer` | TCP-Portnummer, die der Client für die Kommunikation mit diesem Backend verwendet, oder `-1`, wenn ein Unix-Socket verwendet wird. Ist dieses Feld `NULL`, handelt es sich um einen internen Serverprozess. |
| `backend_start` | `timestamp with time zone` | Zeitpunkt, zu dem dieser Prozess gestartet wurde. Bei Client-Backends ist dies der Zeitpunkt, zu dem sich der Client mit dem Server verbunden hat. |
| `xact_start` | `timestamp with time zone` | Zeitpunkt, zu dem die aktuelle Transaktion dieses Prozesses gestartet wurde, oder `NULL`, wenn keine Transaktion aktiv ist. Wenn die aktuelle Abfrage die erste ihrer Transaktion ist, entspricht diese Spalte der Spalte `query_start`. |
| `query_start` | `timestamp with time zone` | Zeitpunkt, zu dem die derzeit aktive Abfrage gestartet wurde, oder, wenn `state` nicht `active` ist, der Zeitpunkt, zu dem die letzte Abfrage gestartet wurde. |
| `state_change` | `timestamp with time zone` | Zeitpunkt, zu dem sich der Zustand zuletzt geändert hat. |
| `wait_event_type` | `text` | Typ des Ereignisses, auf das das Backend wartet, falls vorhanden; andernfalls `NULL`. Siehe Tabelle 27.4. |
| `wait_event` | `text` | Name des Warteereignisses, wenn das Backend derzeit wartet; andernfalls `NULL`. Siehe Tabellen 27.5 bis 27.13. |
| `state` | `text` | Aktueller Gesamtzustand dieses Backends. Mögliche Werte sind `starting` (Backend im initialen Start; Client-Authentifizierung findet in dieser Phase statt), `active` (Backend führt eine Abfrage aus), `idle` (Backend wartet auf einen neuen Clientbefehl), `idle in transaction` (Backend ist in einer Transaktion, führt aber derzeit keine Abfrage aus), `idle in transaction (aborted)` (ähnlich wie `idle in transaction`, aber eine Anweisung in der Transaktion hat einen Fehler verursacht), `fastpath function call` (Backend führt eine Fast-Path-Funktion aus) und `disabled` (wird gemeldet, wenn `track_activities` in diesem Backend deaktiviert ist). |
| `backend_xid` | `xid` | Top-Level-Transaktionskennung dieses Backends, falls vorhanden; siehe [Abschnitt 67.1](67_Transaktionsverarbeitung.md#671-transaktionen-und-identifikatoren). |
| `backend_xmin` | `xid` | Aktueller `xmin`-Horizont dieses Backends. |
| `query_id` | `bigint` | Kennung der jüngsten Abfrage dieses Backends. Wenn `state` `active` ist, zeigt dieses Feld die Kennung der derzeit ausgeführten Abfrage. In allen anderen Zuständen zeigt es die Kennung der zuletzt ausgeführten Abfrage. Query-Identifier werden standardmäßig nicht berechnet; dieses Feld ist daher `NULL`, sofern der Parameter `compute_query_id` nicht aktiviert ist oder ein Drittanbietermodul konfiguriert wurde, das Query-Identifier berechnet. |
| `query` | `text` | Text der jüngsten Abfrage dieses Backends. Wenn `state` `active` ist, zeigt dieses Feld die derzeit ausgeführte Abfrage. In allen anderen Zuständen zeigt es die zuletzt ausgeführte Abfrage. Standardmäßig wird der Abfragetext bei 1024 Bytes abgeschnitten; dieser Wert kann über den Parameter `track_activity_query_size` geändert werden. |
| `backend_type` | `text` | Typ des aktuellen Backends. Mögliche Typen sind `autovacuum launcher`, `autovacuum worker`, `logical replication launcher`, `logical replication worker`, `parallel worker`, `background writer`, `client backend`, `checkpointer`, `archiver`, `standalone backend`, `startup`, `walreceiver`, `walsender`, `walwriter` und `walsummarizer`. Außerdem können von Erweiterungen registrierte Background Worker zusätzliche Typen haben. |

> **Hinweis**
>
> Die Spalten `wait_event` und `state` sind unabhängig voneinander. Wenn ein Backend im Zustand `active` ist, kann es auf ein Ereignis warten oder auch nicht. Ist `state` `active` und `wait_event` nicht `NULL`, bedeutet das, dass eine Abfrage ausgeführt wird, aber irgendwo im System blockiert ist. Um den Meldeaufwand gering zu halten, versucht das System nicht, verschiedene Aspekte der Aktivitätsdaten eines Backends zu synchronisieren. Daher können kurzlebige Abweichungen zwischen den Spalten der Sicht auftreten.

**Tabelle 27.4. Warteereignistypen**

| Warteereignistyp | Beschreibung |
|---|---|
| `Activity` | Der Serverprozess ist untätig. Dieser Ereignistyp zeigt an, dass ein Prozess in seiner Hauptverarbeitungsschleife auf Aktivität wartet. `wait_event` identifiziert den konkreten Wartepunkt; siehe Tabelle 27.5. |
| `BufferPin` | Der Serverprozess wartet auf exklusiven Zugriff auf einen Datenpuffer. Buffer-Pin-Wartezeiten können länger dauern, wenn ein anderer Prozess einen offenen Cursor hält, der zuletzt Daten aus dem betreffenden Puffer gelesen hat. Siehe Tabelle 27.6. |
| `Client` | Der Serverprozess wartet auf Aktivität auf einem Socket, der mit einer Benutzeranwendung verbunden ist. Der Server erwartet also, dass etwas passiert, das unabhängig von seinen internen Prozessen ist. `wait_event` identifiziert den konkreten Wartepunkt; siehe Tabelle 27.7. |
| `Extension` | Der Serverprozess wartet auf eine von einem Erweiterungsmodul definierte Bedingung. Siehe Tabelle 27.8. |
| `InjectionPoint` | Der Serverprozess wartet darauf, dass ein Injection Point ein in einem Test definiertes Ergebnis erreicht. Weitere Details finden Sie in Abschnitt 36.10.14. Dieser Typ hat keine vordefinierten Wartepunkte. |
| `IO` | Der Serverprozess wartet auf den Abschluss einer I/O-Operation. `wait_event` identifiziert den konkreten Wartepunkt; siehe Tabelle 27.9. |
| `IPC` | Der Serverprozess wartet auf eine Interaktion mit einem anderen Serverprozess. `wait_event` identifiziert den konkreten Wartepunkt; siehe Tabelle 27.10. |
| `Lock` | Der Serverprozess wartet auf eine Heavyweight Lock. Heavyweight Locks, auch Lock-Manager-Locks oder einfach Locks genannt, schützen vor allem SQL-sichtbare Objekte wie Tabellen. Sie werden aber auch verwendet, um wechselseitigen Ausschluss für bestimmte interne Operationen wie Relation-Erweiterung sicherzustellen. `wait_event` identifiziert den erwarteten Lock-Typ; siehe Tabelle 27.11. |
| `LWLock` | Der Serverprozess wartet auf eine Lightweight Lock. Die meisten dieser Locks schützen eine bestimmte Datenstruktur im Shared Memory. `wait_event` enthält einen Namen, der den Zweck der Lightweight Lock identifiziert. Manche Locks haben konkrete Namen; andere gehören zu einer Gruppe von Locks mit ähnlichem Zweck. Siehe Tabelle 27.12. |
| `Timeout` | Der Serverprozess wartet darauf, dass ein Timeout abläuft. `wait_event` identifiziert den konkreten Wartepunkt; siehe Tabelle 27.13. |

**Tabelle 27.5. Warteereignisse vom Typ `Activity`**

| Activity-Warteereignis | Beschreibung |
|---|---|
| `ArchiverMain` | Warten in der Hauptschleife des Archiver-Prozesses. |
| `AutovacuumMain` | Warten in der Hauptschleife des Autovacuum-Launcher-Prozesses. |
| `BgwriterHibernate` | Warten im Background-Writer-Prozess im Ruhezustand. |
| `BgwriterMain` | Warten in der Hauptschleife des Background-Writer-Prozesses. |
| `CheckpointerMain` | Warten in der Hauptschleife des Checkpointer-Prozesses. |
| `CheckpointerShutdown` | Warten darauf, dass der Checkpointer-Prozess beendet wird. |
| `IoWorkerMain` | Warten in der Hauptschleife des IO-Worker-Prozesses. |
| `LogicalApplyMain` | Warten in der Hauptschleife des Apply-Prozesses für logische Replikation. |
| `LogicalLauncherMain` | Warten in der Hauptschleife des Launchers für logische Replikation. |
| `LogicalParallelApplyMain` | Warten in der Hauptschleife des parallelen Apply-Prozesses für logische Replikation. |
| `RecoveryWalStream` | Warten in der Hauptschleife des Startup-Prozesses darauf, dass während Streaming-Recovery WAL eintrifft. |
| `ReplicationSlotsyncMain` | Warten in der Hauptschleife des Slot-Sync-Workers. |
| `ReplicationSlotsyncShutdown` | Warten darauf, dass der Slot-Sync-Worker herunterfährt. |
| `SysloggerMain` | Warten in der Hauptschleife des Syslogger-Prozesses. |
| `WalReceiverMain` | Warten in der Hauptschleife des WAL-Receiver-Prozesses. |
| `WalSenderMain` | Warten in der Hauptschleife des WAL-Sender-Prozesses. |
| `WalSummarizerWal` | Warten im WAL-Summarizer darauf, dass weiteres WAL erzeugt wird. |
| `WalWriterMain` | Warten in der Hauptschleife des WAL-Writer-Prozesses. |

**Tabelle 27.6. Warteereignisse vom Typ `BufferPin`**

| BufferPin-Warteereignis | Beschreibung |
|---|---|
| `BufferPin` | Warten darauf, einen exklusiven Pin auf einem Puffer zu erwerben. |

**Tabelle 27.7. Warteereignisse vom Typ `Client`**

| Client-Warteereignis | Beschreibung |
|---|---|
| `ClientRead` | Warten darauf, Daten vom Client zu lesen. |
| `ClientWrite` | Warten darauf, Daten an den Client zu schreiben. |
| `GssOpenServer` | Warten darauf, Daten vom Client zu lesen, während eine GSSAPI-Sitzung aufgebaut wird. |
| `LibpqwalreceiverConnect` | Warten im WAL-Receiver darauf, eine Verbindung zum Remote-Server aufzubauen. |
| `LibpqwalreceiverReceive` | Warten im WAL-Receiver darauf, Daten vom Remote-Server zu empfangen. |
| `SslOpenServer` | Warten auf SSL beim Verbindungsversuch. |
| `WaitForStandbyConfirmation` | Warten darauf, dass WAL vom physischen Standby empfangen und geflusht wird. |
| `WalSenderWaitForWal` | Warten im WAL-Sender-Prozess darauf, dass WAL geflusht wird. |
| `WalSenderWriteData` | Warten auf irgendeine Aktivität beim Verarbeiten von Antworten des WAL-Receivers im WAL-Sender-Prozess. |

**Tabelle 27.8. Warteereignisse vom Typ `Extension`**

| Extension-Warteereignis | Beschreibung |
|---|---|
| `Extension` | Warten in einer Erweiterung. |

**Tabelle 27.9. Warteereignisse vom Typ `IO`**

| IO-Warteereignis | Beschreibung |
|---|---|
| `AioIoCompletion` | Warten darauf, dass ein anderer Prozess I/O abschließt. |
| `AioIoUringExecution` | Warten auf I/O-Ausführung über `io_uring`. |
| `AioIoUringSubmit` | Warten auf I/O-Submission über `io_uring`. |
| `BasebackupRead` | Warten darauf, dass eine Basissicherung aus einer Datei liest. |
| `BasebackupSync` | Warten darauf, dass von einer Basissicherung geschriebene Daten dauerhaften Speicher erreichen. |
| `BasebackupWrite` | Warten darauf, dass eine Basissicherung in eine Datei schreibt. |
| `BuffileRead` | Warten auf einen Lesevorgang aus einer gepufferten Datei. |
| `BuffileTruncate` | Warten darauf, dass eine gepufferte Datei gekürzt wird. |
| `BuffileWrite` | Warten auf einen Schreibvorgang in eine gepufferte Datei. |
| `ControlFileRead` | Warten auf einen Lesevorgang aus der Datei `pg_control`. |
| `ControlFileSync` | Warten darauf, dass die Datei `pg_control` dauerhaften Speicher erreicht. |
| `ControlFileSyncUpdate` | Warten darauf, dass eine Aktualisierung der Datei `pg_control` dauerhaften Speicher erreicht. |
| `ControlFileWrite` | Warten auf einen Schreibvorgang in die Datei `pg_control`. |
| `ControlFileWriteUpdate` | Warten auf einen Schreibvorgang zur Aktualisierung der Datei `pg_control`. |
| `CopyFileCopy` | Warten auf eine Dateikopieroperation. |
| `CopyFileRead` | Warten auf einen Lesevorgang während einer Dateikopieroperation. |
| `CopyFileWrite` | Warten auf einen Schreibvorgang während einer Dateikopieroperation. |
| `DataFileExtend` | Warten darauf, dass eine Relationsdatendatei erweitert wird. |
| `DataFileFlush` | Warten darauf, dass eine Relationsdatendatei dauerhaften Speicher erreicht. |
| `DataFileImmediateSync` | Warten auf eine sofortige Synchronisierung einer Relationsdatendatei in dauerhaften Speicher. |
| `DataFilePrefetch` | Warten auf asynchrones Prefetching aus einer Relationsdatendatei. |
| `DataFileRead` | Warten auf einen Lesevorgang aus einer Relationsdatendatei. |
| `DataFileSync` | Warten darauf, dass Änderungen an einer Relationsdatendatei dauerhaften Speicher erreichen. |
| `DataFileTruncate` | Warten darauf, dass eine Relationsdatendatei gekürzt wird. |
| `DataFileWrite` | Warten auf einen Schreibvorgang in eine Relationsdatendatei. |
| `DsmAllocate` | Warten darauf, dass ein dynamisches Shared-Memory-Segment zugewiesen wird. |
| `DsmFillZeroWrite` | Warten darauf, eine Backing-Datei für dynamisches Shared Memory mit Nullen zu füllen. |
| `LockFileAddtodatadirRead` | Warten auf einen Lesevorgang beim Hinzufügen einer Zeile zur Datenverzeichnis-Lockdatei. |
| `LockFileAddtodatadirSync` | Warten darauf, dass Daten beim Hinzufügen einer Zeile zur Datenverzeichnis-Lockdatei dauerhaften Speicher erreichen. |
| `LockFileAddtodatadirWrite` | Warten auf einen Schreibvorgang beim Hinzufügen einer Zeile zur Datenverzeichnis-Lockdatei. |
| `LockFileCreateRead` | Warten auf einen Lesevorgang beim Erstellen der Datenverzeichnis-Lockdatei. |
| `LockFileCreateSync` | Warten darauf, dass Daten beim Erstellen der Datenverzeichnis-Lockdatei dauerhaften Speicher erreichen. |
| `LockFileCreateWrite` | Warten auf einen Schreibvorgang beim Erstellen der Datenverzeichnis-Lockdatei. |
| `LockFileRecheckdatadirRead` | Warten auf einen Lesevorgang beim erneuten Prüfen der Datenverzeichnis-Lockdatei. |
| `LogicalRewriteCheckpointSync` | Warten darauf, dass Logical-Rewrite-Mappings während eines Checkpoints dauerhaften Speicher erreichen. |
| `LogicalRewriteMappingSync` | Warten darauf, dass Mapping-Daten während eines Logical Rewrite dauerhaften Speicher erreichen. |
| `LogicalRewriteMappingWrite` | Warten auf das Schreiben von Mapping-Daten während eines Logical Rewrite. |
| `LogicalRewriteSync` | Warten darauf, dass Logical-Rewrite-Mappings dauerhaften Speicher erreichen. |
| `LogicalRewriteTruncate` | Warten auf das Kürzen von Mapping-Daten während eines Logical Rewrite. |
| `LogicalRewriteWrite` | Warten auf das Schreiben von Logical-Rewrite-Mappings. |
| `RelationMapRead` | Warten auf einen Lesevorgang aus der Relation-Map-Datei. |
| `RelationMapReplace` | Warten auf die dauerhafte Ersetzung einer Relation-Map-Datei. |
| `RelationMapWrite` | Warten auf einen Schreibvorgang in die Relation-Map-Datei. |
| `ReorderBufferRead` | Warten auf einen Lesevorgang während der Reorder-Buffer-Verwaltung. |
| `ReorderBufferWrite` | Warten auf einen Schreibvorgang während der Reorder-Buffer-Verwaltung. |
| `ReorderLogicalMappingRead` | Warten auf einen Lesevorgang aus einem logischen Mapping während der Reorder-Buffer-Verwaltung. |
| `ReplicationSlotRead` | Warten auf einen Lesevorgang aus einer Steuerdatei eines Replikations-Slots. |
| `ReplicationSlotRestoreSync` | Warten darauf, dass eine Steuerdatei eines Replikations-Slots beim Wiederherstellen in den Speicher dauerhaften Speicher erreicht. |
| `ReplicationSlotSync` | Warten darauf, dass eine Steuerdatei eines Replikations-Slots dauerhaften Speicher erreicht. |
| `ReplicationSlotWrite` | Warten auf einen Schreibvorgang in eine Steuerdatei eines Replikations-Slots. |
| `SlruFlushSync` | Warten darauf, dass SLRU-Daten während eines Checkpoints oder Datenbank-Shutdowns dauerhaften Speicher erreichen. |
| `SlruRead` | Warten auf einen Lesevorgang einer SLRU-Seite. |
| `SlruSync` | Warten darauf, dass SLRU-Daten nach einem Seitenschreibvorgang dauerhaften Speicher erreichen. |
| `SlruWrite` | Warten auf einen Schreibvorgang einer SLRU-Seite. |
| `SnapbuildRead` | Warten auf einen Lesevorgang eines serialisierten historischen Katalog-Snapshots. |
| `SnapbuildSync` | Warten darauf, dass ein serialisierter historischer Katalog-Snapshot dauerhaften Speicher erreicht. |
| `SnapbuildWrite` | Warten auf einen Schreibvorgang eines serialisierten historischen Katalog-Snapshots. |
| `TimelineHistoryFileSync` | Warten darauf, dass eine per Streaming-Replikation empfangene Timeline-History-Datei dauerhaften Speicher erreicht. |
| `TimelineHistoryFileWrite` | Warten auf das Schreiben einer per Streaming-Replikation empfangenen Timeline-History-Datei. |
| `TimelineHistoryRead` | Warten auf einen Lesevorgang aus einer Timeline-History-Datei. |
| `TimelineHistorySync` | Warten darauf, dass eine neu erzeugte Timeline-History-Datei dauerhaften Speicher erreicht. |
| `TimelineHistoryWrite` | Warten auf das Schreiben einer neu erzeugten Timeline-History-Datei. |
| `TwophaseFileRead` | Warten auf einen Lesevorgang aus einer Two-Phase-State-Datei. |
| `TwophaseFileSync` | Warten darauf, dass eine Two-Phase-State-Datei dauerhaften Speicher erreicht. |
| `TwophaseFileWrite` | Warten auf einen Schreibvorgang in eine Two-Phase-State-Datei. |
| `VersionFileSync` | Warten darauf, dass die Versionsdatei beim Erstellen einer Datenbank dauerhaften Speicher erreicht. |
| `VersionFileWrite` | Warten darauf, dass die Versionsdatei beim Erstellen einer Datenbank geschrieben wird. |
| `WalsenderTimelineHistoryRead` | Warten auf einen Lesevorgang aus einer Timeline-History-Datei während eines Walsender-Timeline-Befehls. |
| `WalBootstrapSync` | Warten darauf, dass WAL während des Bootstrappings dauerhaften Speicher erreicht. |
| `WalBootstrapWrite` | Warten auf das Schreiben einer WAL-Seite während des Bootstrappings. |
| `WalCopyRead` | Warten auf einen Lesevorgang beim Erzeugen eines neuen WAL-Segments durch Kopieren eines vorhandenen Segments. |
| `WalCopySync` | Warten darauf, dass ein durch Kopieren eines vorhandenen Segments erzeugtes neues WAL-Segment dauerhaften Speicher erreicht. |
| `WalCopyWrite` | Warten auf einen Schreibvorgang beim Erzeugen eines neuen WAL-Segments durch Kopieren eines vorhandenen Segments. |
| `WalInitSync` | Warten darauf, dass eine neu initialisierte WAL-Datei dauerhaften Speicher erreicht. |
| `WalInitWrite` | Warten auf einen Schreibvorgang beim Initialisieren einer neuen WAL-Datei. |
| `WalRead` | Warten auf einen Lesevorgang aus einer WAL-Datei. |
| `WalSummaryRead` | Warten auf einen Lesevorgang aus einer WAL-Summary-Datei. |
| `WalSummaryWrite` | Warten auf einen Schreibvorgang in eine WAL-Summary-Datei. |
| `WalSync` | Warten darauf, dass eine WAL-Datei dauerhaften Speicher erreicht. |
| `WalSyncMethodAssign` | Warten darauf, dass Daten beim Zuweisen einer neuen WAL-Sync-Methode dauerhaften Speicher erreichen. |
| `WalWrite` | Warten auf einen Schreibvorgang in eine WAL-Datei. |

**Tabelle 27.10. Warteereignisse vom Typ `IPC`**

| IPC-Warteereignis | Beschreibung |
|---|---|
| `AppendReady` | Warten darauf, dass Subplan-Knoten eines `Append`-Planknotens bereit sind. |
| `ArchiveCleanupCommand` | Warten auf den Abschluss von `archive_cleanup_command`. |
| `ArchiveCommand` | Warten auf den Abschluss von `archive_command`. |
| `BackendTermination` | Warten auf die Beendigung eines anderen Backends. |
| `BackupWaitWalArchive` | Warten darauf, dass für eine Sicherung erforderliche WAL-Dateien erfolgreich archiviert werden. |
| `BgworkerShutdown` | Warten darauf, dass ein Background Worker herunterfährt. |
| `BgworkerStartup` | Warten darauf, dass ein Background Worker startet. |
| `BtreePage` | Warten darauf, dass die für die Fortsetzung eines parallelen B-Tree-Scans benötigte Seitennummer verfügbar wird. |
| `BufferIo` | Warten auf den Abschluss von Puffer-I/O. |
| `CheckpointDelayComplete` | Warten auf ein Backend, das den Abschluss eines Checkpoints blockiert. |
| `CheckpointDelayStart` | Warten auf ein Backend, das den Start eines Checkpoints blockiert. |
| `CheckpointDone` | Warten auf den Abschluss eines Checkpoints. |
| `CheckpointStart` | Warten auf den Start eines Checkpoints. |
| `ExecuteGather` | Warten auf Aktivität eines Child-Prozesses während der Ausführung eines `Gather`-Planknotens. |
| `HashBatchAllocate` | Warten darauf, dass ein gewählter Parallel-Hash-Teilnehmer eine Hashtabelle zuweist. |
| `HashBatchElect` | Warten darauf, einen Parallel-Hash-Teilnehmer für die Zuweisung einer Hashtabelle zu wählen. |
| `HashBatchLoad` | Warten darauf, dass andere Parallel-Hash-Teilnehmer das Laden einer Hashtabelle abschließen. |
| `HashBuildAllocate` | Warten darauf, dass ein gewählter Parallel-Hash-Teilnehmer die initiale Hashtabelle zuweist. |
| `HashBuildElect` | Warten darauf, einen Parallel-Hash-Teilnehmer für die Zuweisung der initialen Hashtabelle zu wählen. |
| `HashBuildHashInner` | Warten darauf, dass andere Parallel-Hash-Teilnehmer das Hashing der inneren Relation abschließen. |
| `HashBuildHashOuter` | Warten darauf, dass andere Parallel-Hash-Teilnehmer die äußere Relation partitionieren. |
| `HashGrowBatchesDecide` | Warten darauf, einen Parallel-Hash-Teilnehmer zu wählen, der über künftiges Batch-Wachstum entscheidet. |
| `HashGrowBatchesElect` | Warten darauf, einen Parallel-Hash-Teilnehmer für die Zuweisung weiterer Batches zu wählen. |
| `HashGrowBatchesFinish` | Warten darauf, dass ein gewählter Parallel-Hash-Teilnehmer über künftiges Batch-Wachstum entscheidet. |
| `HashGrowBatchesReallocate` | Warten darauf, dass ein gewählter Parallel-Hash-Teilnehmer weitere Batches zuweist. |
| `HashGrowBatchesRepartition` | Warten darauf, dass andere Parallel-Hash-Teilnehmer die Repartitionierung abschließen. |
| `HashGrowBucketsElect` | Warten darauf, einen Parallel-Hash-Teilnehmer für die Zuweisung weiterer Buckets zu wählen. |
| `HashGrowBucketsReallocate` | Warten darauf, dass ein gewählter Parallel-Hash-Teilnehmer die Zuweisung weiterer Buckets abschließt. |
| `HashGrowBucketsReinsert` | Warten darauf, dass andere Parallel-Hash-Teilnehmer Tupel in neue Buckets einfügen. |
| `LogicalApplySendData` | Warten darauf, dass ein Leader-Apply-Prozess der logischen Replikation Daten an einen parallelen Apply-Prozess sendet. |
| `LogicalParallelApplyStateChange` | Warten darauf, dass ein paralleler Apply-Prozess der logischen Replikation den Zustand ändert. |
| `LogicalSyncData` | Warten darauf, dass ein Remote-Server der logischen Replikation Daten für die anfängliche Tabellensynchronisierung sendet. |
| `LogicalSyncStateChange` | Warten darauf, dass ein Remote-Server der logischen Replikation den Zustand ändert. |
| `MessageQueueInternal` | Warten darauf, dass ein anderer Prozess an eine gemeinsame Nachrichtenwarteschlange angehängt wird. |
| `MessageQueuePutMessage` | Warten darauf, eine Protokollnachricht in eine gemeinsame Nachrichtenwarteschlange zu schreiben. |
| `MessageQueueReceive` | Warten darauf, Bytes aus einer gemeinsamen Nachrichtenwarteschlange zu empfangen. |
| `MessageQueueSend` | Warten darauf, Bytes an eine gemeinsame Nachrichtenwarteschlange zu senden. |
| `MultixactCreation` | Warten auf den Abschluss einer Multixact-Erzeugung. |
| `ParallelBitmapScan` | Warten darauf, dass ein paralleler Bitmap-Scan initialisiert wird. |
| `ParallelCreateIndexScan` | Warten darauf, dass parallele `CREATE INDEX`-Worker den Heap-Scan abschließen. |
| `ParallelFinish` | Warten darauf, dass parallele Worker ihre Berechnungen abschließen. |
| `ProcarrayGroupUpdate` | Warten darauf, dass der Gruppenleiter die Transaktions-ID am Transaktionsende löscht. |
| `ProcSignalBarrier` | Warten darauf, dass ein Barrier-Ereignis von allen Backends verarbeitet wird. |
| `Promote` | Warten auf Standby-Hochstufung. |
| `RecoveryConflictSnapshot` | Warten auf die Auflösung eines Recovery-Konflikts für ein Vacuum-Cleanup. |
| `RecoveryConflictTablespace` | Warten auf die Auflösung eines Recovery-Konflikts beim Löschen eines Tablespace. |
| `RecoveryEndCommand` | Warten auf den Abschluss von `recovery_end_command`. |
| `RecoveryPause` | Warten darauf, dass Recovery fortgesetzt wird. |
| `ReplicationOriginDrop` | Warten darauf, dass ein Replikationsursprung inaktiv wird, damit er gelöscht werden kann. |
| `ReplicationSlotDrop` | Warten darauf, dass ein Replikations-Slot inaktiv wird, damit er gelöscht werden kann. |
| `RestoreCommand` | Warten auf den Abschluss von `restore_command`. |
| `SafeSnapshot` | Warten darauf, einen gültigen Snapshot für eine `READ ONLY DEFERRABLE`-Transaktion zu erhalten. |
| `SyncRep` | Warten auf Bestätigung von einem Remote-Server während synchroner Replikation. |
| `WalReceiverExit` | Warten darauf, dass der WAL-Receiver beendet wird. |
| `WalReceiverWaitStart` | Warten darauf, dass der Startup-Prozess initiale Daten für Streaming-Replikation sendet. |
| `WalSummaryReady` | Warten darauf, dass eine neue WAL-Summary erzeugt wird. |
| `XactGroupUpdate` | Warten darauf, dass der Gruppenleiter den Transaktionsstatus am Transaktionsende aktualisiert. |

**Tabelle 27.11. Warteereignisse vom Typ `Lock`**

| Lock-Warteereignis | Beschreibung |
|---|---|
| `advisory` | Warten darauf, eine Advisory User Lock zu erwerben. |
| `applytransaction` | Warten darauf, eine Sperre auf einer Remote-Transaktion zu erwerben, die von einem Subscriber logischer Replikation angewendet wird. |
| `extend` | Warten darauf, eine Relation zu erweitern. |
| `frozenid` | Warten darauf, `pg_database.datfrozenxid` und `pg_database.datminmxid` zu aktualisieren. |
| `object` | Warten darauf, eine Sperre auf einem Nicht-Relation-Datenbankobjekt zu erwerben. |
| `page` | Warten darauf, eine Sperre auf einer Seite einer Relation zu erwerben. |
| `relation` | Warten darauf, eine Sperre auf einer Relation zu erwerben. |
| `spectoken` | Warten darauf, eine spekulative Insertionssperre zu erwerben. |
| `transactionid` | Warten darauf, dass eine Transaktion beendet wird. |
| `tuple` | Warten darauf, eine Sperre auf einem Tupel zu erwerben. |
| `userlock` | Warten darauf, eine User Lock zu erwerben. |
| `virtualxid` | Warten darauf, eine virtuelle Transaktions-ID-Sperre zu erwerben; siehe [Abschnitt 67.1](67_Transaktionsverarbeitung.md#671-transaktionen-und-identifikatoren). |

**Tabelle 27.12. Warteereignisse vom Typ `LWLock`**

| LWLock-Warteereignis | Beschreibung |
|---|---|
| `AddinShmemInit` | Warten darauf, die Speicherzuweisung einer Erweiterung im Shared Memory zu verwalten. |
| `AioUringCompletion` | Warten darauf, dass ein anderer Prozess I/O über `io_uring` abschließt. |
| `AioWorkerSubmissionQueue` | Warten auf Zugriff auf die AIO-Worker-Submission-Queue. |
| `AutoFile` | Warten darauf, die Datei `postgresql.auto.conf` zu aktualisieren. |
| `Autovacuum` | Warten darauf, den aktuellen Zustand von Autovacuum-Workern zu lesen oder zu aktualisieren. |
| `AutovacuumSchedule` | Warten darauf sicherzustellen, dass eine für Autovacuum ausgewählte Tabelle weiterhin Vacuuming benötigt. |
| `BackgroundWorker` | Warten darauf, den Zustand von Background Workern zu lesen oder zu aktualisieren. |
| `BtreeVacuum` | Warten darauf, Vacuum-bezogene Informationen für einen B-Tree-Index zu lesen oder zu aktualisieren. |
| `BufferContent` | Warten auf Zugriff auf eine Datenseite im Speicher. |
| `BufferMapping` | Warten darauf, einen Datenblock mit einem Puffer im Buffer Pool zu verknüpfen. |
| `CheckpointerComm` | Warten darauf, `fsync`-Anforderungen zu verwalten. |
| `CommitTs` | Warten darauf, den zuletzt gesetzten Wert für einen Transaktions-Commit-Zeitstempel zu lesen oder zu aktualisieren. |
| `CommitTsBuffer` | Warten auf I/O auf einem Commit-Timestamp-SLRU-Puffer. |
| `CommitTsSLRU` | Warten auf Zugriff auf den Commit-Timestamp-SLRU-Cache. |
| `ControlFile` | Warten darauf, die Datei `pg_control` zu lesen oder zu aktualisieren oder eine neue WAL-Datei zu erzeugen. |
| `DSMRegistry` | Warten darauf, die Registry für dynamisches Shared Memory zu lesen oder zu aktualisieren. |
| `DSMRegistryDSA` | Warten auf Zugriff auf den Dynamic-Shared-Memory-Allocator der DSM-Registry. |
| `DSMRegistryHash` | Warten auf Zugriff auf die Shared-Hash-Tabelle der DSM-Registry. |
| `DynamicSharedMemoryControl` | Warten darauf, Informationen zur Zuweisung von dynamischem Shared Memory zu lesen oder zu aktualisieren. |
| `InjectionPoint` | Warten darauf, Informationen zu Injection Points zu lesen oder zu aktualisieren. |
| `LockFastPath` | Warten darauf, Fast-Path-Lock-Informationen eines Prozesses zu lesen oder zu aktualisieren. |
| `LockManager` | Warten darauf, Informationen über Heavyweight Locks zu lesen oder zu aktualisieren. |
| `LogicalRepLauncherDSA` | Warten auf Zugriff auf den Dynamic-Shared-Memory-Allocator des Launchers für logische Replikation. |
| `LogicalRepLauncherHash` | Warten auf Zugriff auf die Shared-Hash-Tabelle des Launchers für logische Replikation. |
| `LogicalRepWorker` | Warten darauf, den Zustand von Workern für logische Replikation zu lesen oder zu aktualisieren. |
| `MultiXactGen` | Warten darauf, den gemeinsamen Multixact-Zustand zu lesen oder zu aktualisieren. |
| `MultiXactMemberBuffer` | Warten auf I/O auf einem Multixact-Member-SLRU-Puffer. |
| `MultiXactMemberSLRU` | Warten auf Zugriff auf den Multixact-Member-SLRU-Cache. |
| `MultiXactOffsetBuffer` | Warten auf I/O auf einem Multixact-Offset-SLRU-Puffer. |
| `MultiXactOffsetSLRU` | Warten auf Zugriff auf den Multixact-Offset-SLRU-Cache. |
| `MultiXactTruncation` | Warten darauf, Multixact-Informationen zu lesen oder zu kürzen. |
| `NotifyBuffer` | Warten auf I/O auf einem NOTIFY-Message-SLRU-Puffer. |
| `NotifyQueue` | Warten darauf, `NOTIFY`-Nachrichten zu lesen oder zu aktualisieren. |
| `NotifyQueueTail` | Warten darauf, die Grenze für NOTIFY-Nachrichtenspeicher zu aktualisieren. |
| `NotifySLRU` | Warten auf Zugriff auf den NOTIFY-Message-SLRU-Cache. |
| `OidGen` | Warten darauf, eine neue OID zuzuweisen. |
| `ParallelAppend` | Warten darauf, während der Ausführung eines Parallel-Append-Plans den nächsten Subplan auszuwählen. |
| `ParallelBtreeScan` | Warten darauf, Worker während der Ausführung eines parallelen B-Tree-Scan-Plans zu synchronisieren. |
| `ParallelHashJoin` | Warten darauf, Worker während der Ausführung eines Parallel-Hash-Join-Plans zu synchronisieren. |
| `ParallelQueryDSA` | Warten auf Zuweisung von dynamischem Shared Memory für parallele Abfragen. |
| `ParallelVacuumDSA` | Warten auf Zuweisung von dynamischem Shared Memory für paralleles Vacuum. |
| `PerSessionDSA` | Warten auf Zuweisung von dynamischem Shared Memory für parallele Abfragen. |
| `PerSessionRecordType` | Warten auf Zugriff auf Informationen einer parallelen Abfrage über zusammengesetzte Typen. |
| `PerSessionRecordTypmod` | Warten auf Zugriff auf Informationen einer parallelen Abfrage über Typmodifikatoren, die anonyme Record-Typen identifizieren. |
| `PerXactPredicateList` | Warten auf Zugriff auf die Liste der von der aktuellen serialisierbaren Transaktion gehaltenen Predicate Locks während einer parallelen Abfrage. |
| `PgStatsData` | Warten auf Zugriff auf Statistikdaten im Shared Memory. |
| `PgStatsDSA` | Warten auf Zugriff auf den Dynamic-Shared-Memory-Allocator für Statistiken. |
| `PgStatsHash` | Warten auf Zugriff auf die Shared-Memory-Hash-Tabelle für Statistiken. |
| `PredicateLockManager` | Warten auf Zugriff auf Predicate-Lock-Informationen, die von serialisierbaren Transaktionen verwendet werden. |
| `ProcArray` | Warten auf Zugriff auf die gemeinsamen prozessbezogenen Datenstrukturen, typischerweise um einen Snapshot zu erhalten oder die Transaktions-ID einer Sitzung zu melden. |
| `RelationMapping` | Warten darauf, eine `pg_filenode.map`-Datei zu lesen oder zu aktualisieren, die die Filenode-Zuweisungen bestimmter Systemkataloge verfolgt. |
| `RelCacheInit` | Warten darauf, eine Initialisierungsdatei `pg_internal.init` für den Relation Cache zu lesen oder zu aktualisieren. |
| `ReplicationOrigin` | Warten darauf, einen Replikationsursprung zu erzeugen, zu löschen oder zu verwenden. |
| `ReplicationOriginState` | Warten darauf, den Fortschritt eines Replikationsursprungs zu lesen oder zu aktualisieren. |
| `ReplicationSlotAllocation` | Warten darauf, einen Replikations-Slot zuzuweisen oder freizugeben. |
| `ReplicationSlotControl` | Warten darauf, den Zustand eines Replikations-Slots zu lesen oder zu aktualisieren. |
| `ReplicationSlotIO` | Warten auf I/O auf einem Replikations-Slot. |
| `SerialBuffer` | Warten auf I/O auf einem SLRU-Puffer für Konflikte serialisierbarer Transaktionen. |
| `SerialControl` | Warten darauf, den gemeinsamen `pg_serial`-Zustand zu lesen oder zu aktualisieren. |
| `SerializableFinishedList` | Warten auf Zugriff auf die Liste abgeschlossener serialisierbarer Transaktionen. |
| `SerializablePredicateList` | Warten auf Zugriff auf die Liste der von serialisierbaren Transaktionen gehaltenen Predicate Locks. |
| `SerializableXactHash` | Warten darauf, Informationen über serialisierbare Transaktionen zu lesen oder zu aktualisieren. |
| `SerialSLRU` | Warten auf Zugriff auf den SLRU-Cache für Konflikte serialisierbarer Transaktionen. |
| `SharedTidBitmap` | Warten auf Zugriff auf eine gemeinsame TID-Bitmap während eines parallelen Bitmap-Index-Scans. |
| `SharedTupleStore` | Warten auf Zugriff auf einen gemeinsamen Tuple Store während einer parallelen Abfrage. |
| `ShmemIndex` | Warten darauf, Platz im Shared Memory zu finden oder zuzuweisen. |
| `SInvalRead` | Warten darauf, Nachrichten aus der gemeinsamen Katalog-Invalidierungswarteschlange abzurufen. |
| `SInvalWrite` | Warten darauf, eine Nachricht zur gemeinsamen Katalog-Invalidierungswarteschlange hinzuzufügen. |
| `SubtransBuffer` | Warten auf I/O auf einem Subtransaction-SLRU-Puffer. |
| `SubtransSLRU` | Warten auf Zugriff auf den Subtransaction-SLRU-Cache. |
| `SyncRep` | Warten darauf, Informationen über den Zustand synchroner Replikation zu lesen oder zu aktualisieren. |
| `SyncScan` | Warten darauf, die Startposition eines synchronisierten Tabellenscans auszuwählen. |
| `TablespaceCreate` | Warten darauf, einen Tablespace zu erzeugen oder zu löschen. |
| `TwoPhaseState` | Warten darauf, den Zustand vorbereiteter Transaktionen zu lesen oder zu aktualisieren. |
| `WaitEventCustom` | Warten darauf, Informationen über eigene Warteereignisse zu lesen oder zu aktualisieren. |
| `WALBufMapping` | Warten darauf, eine Seite in WAL-Puffern zu ersetzen. |
| `WALInsert` | Warten darauf, WAL-Daten in einen Speicherpuffer einzufügen. |
| `WALSummarizer` | Warten darauf, den Zustand der WAL-Summarization zu lesen oder zu aktualisieren. |
| `WALWrite` | Warten darauf, dass WAL-Puffer auf Platte geschrieben werden. |
| `WrapLimitsVacuum` | Warten darauf, Grenzen für Transaction-ID- und Multixact-Verbrauch zu aktualisieren. |
| `XactBuffer` | Warten auf I/O auf einem SLRU-Puffer für Transaktionsstatus. |
| `XactSLRU` | Warten auf Zugriff auf den SLRU-Cache für Transaktionsstatus. |
| `XactTruncation` | Warten darauf, `pg_xact_status` auszuführen oder die älteste dafür verfügbare Transaktions-ID zu aktualisieren. |
| `XidGen` | Warten darauf, eine neue Transaktions-ID zuzuweisen. |

**Tabelle 27.13. Warteereignisse vom Typ `Timeout`**

| Timeout-Warteereignis | Beschreibung |
|---|---|
| `BaseBackupThrottle` | Warten während einer Basissicherung wegen Drosselung der Aktivität. |
| `CheckpointWriteDelay` | Warten zwischen Schreibvorgängen während eines Checkpoints. |
| `PgSleep` | Warten wegen eines Aufrufs von `pg_sleep` oder einer verwandten Funktion. |
| `RecoveryApplyDelay` | Warten darauf, WAL während Recovery wegen einer Verzögerungseinstellung anzuwenden. |
| `RecoveryRetrieveRetryInterval` | Warten während Recovery, wenn WAL-Daten aus keiner Quelle verfügbar sind (`pg_wal`, Archiv oder Stream). |
| `RegisterSyncRequest` | Warten beim Senden von Synchronisierungsanforderungen an den Checkpointer, weil die Anforderungswarteschlange voll ist. |
| `SpinDelay` | Warten beim Erwerb eines umkämpften Spinlocks. |
| `VacuumDelay` | Warten an einem kostenbasierten Vacuum-Verzögerungspunkt. |
| `VacuumTruncate` | Warten darauf, eine exklusive Sperre zu erwerben, um leere Seiten am Ende einer vacuumten Tabelle abzuschneiden. |
| `WalSummarizerError` | Warten nach einem Fehler des WAL-Summarizers. |

Hier sind Beispiele dafür, wie Warteereignisse angezeigt werden können:

```sql
SELECT pid, wait_event_type, wait_event FROM pg_stat_activity WHERE
 wait_event is NOT NULL;
```

```text
 pid  | wait_event_type | wait_event
------+-----------------+------------
 2540 | Lock            | relation
 6644 | LWLock          | ProcArray
(2 rows)
```

```sql
SELECT a.pid, a.wait_event, w.description
  FROM pg_stat_activity a JOIN
       pg_wait_events w ON (a.wait_event_type = w.type AND
                            a.wait_event = w.name)
  WHERE a.wait_event is NOT NULL and a.state = 'active';
```

```text
-[ RECORD 1 ]------------------------------------------------------
pid         | 686674
wait_event  | WALInitSync
description | Waiting for a newly initialized WAL file to reach durable storage
```

> **Hinweis**
>
> Erweiterungen können den in Tabelle 27.8 und Tabelle 27.12 gezeigten Listen Ereignisse der Typen `Extension`, `InjectionPoint` und `LWLock` hinzufügen. In manchen Fällen ist der Name einer von einer Erweiterung zugewiesenen `LWLock` nicht in allen Serverprozessen verfügbar. Er kann dann nur als `extension` statt unter dem von der Erweiterung zugewiesenen Namen gemeldet werden.

### 27.2.4. pg_stat_replication

Die Sicht `pg_stat_replication` enthält eine Zeile pro WAL-Sender-Prozess und zeigt Statistiken zur Replikation auf den mit diesem Sender verbundenen Standby-Server. Aufgeführt werden nur direkt verbundene Standbys; über nachgelagerte Standby-Server sind keine Informationen verfügbar.

**Tabelle 27.14. Sicht `pg_stat_replication`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID eines WAL-Sender-Prozesses. |
| `usesysid` | `oid` | OID des Benutzers, der an diesem WAL-Sender-Prozess angemeldet ist. |
| `usename` | `name` | Name des Benutzers, der an diesem WAL-Sender-Prozess angemeldet ist. |
| `application_name` | `text` | Name der Anwendung, die mit diesem WAL-Sender verbunden ist. |
| `client_addr` | `inet` | IP-Adresse des Clients, der mit diesem WAL-Sender verbunden ist. Ist dieses Feld null, ist der Client über einen Unix-Socket auf dem Serverrechner verbunden. |
| `client_hostname` | `text` | Hostname des verbundenen Clients, wie er durch eine Reverse-DNS-Abfrage von `client_addr` ermittelt wurde. Dieses Feld ist nur bei IP-Verbindungen ungleich null und nur dann, wenn `log_hostname` aktiviert ist. |
| `client_port` | `integer` | TCP-Portnummer, die der Client für die Kommunikation mit diesem WAL-Sender verwendet, oder `-1`, wenn ein Unix-Socket verwendet wird. |
| `backend_start` | `timestamp with time zone` | Zeitpunkt, zu dem dieser Prozess gestartet wurde, also wann der Client die Verbindung zu diesem WAL-Sender hergestellt hat. |
| `backend_xmin` | `xid` | Der von diesem Standby über `hot_standby_feedback` gemeldete `xmin`-Horizont. |
| `state` | `text` | Aktueller Zustand des WAL-Senders. Mögliche Werte sind `startup` (der WAL-Sender startet), `catchup` (der verbundene Standby holt den Primärserver ein), `streaming` (der WAL-Sender streamt Änderungen, nachdem der verbundene Standby den Primärserver eingeholt hat), `backup` (der WAL-Sender sendet eine Sicherung) und `stopping` (der WAL-Sender wird beendet). |
| `sent_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die über diese Verbindung gesendet wurde. |
| `write_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die von diesem Standby-Server auf Platte geschrieben wurde. |
| `flush_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die von diesem Standby-Server auf Platte geflusht wurde. |
| `replay_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die auf diesem Standby-Server in der Datenbank wiedergegeben wurde. |
| `write_lag` | `interval` | Verstrichene Zeit zwischen dem lokalen Flushen aktueller WAL-Daten und der Benachrichtigung, dass dieser Standby-Server sie geschrieben hat, aber noch nicht geflusht oder angewendet hat. Damit lässt sich die Verzögerung abschätzen, die die Stufe `remote_write` von `synchronous_commit` beim Commit verursacht hat, wenn dieser Server als synchroner Standby konfiguriert war. |
| `flush_lag` | `interval` | Verstrichene Zeit zwischen dem lokalen Flushen aktueller WAL-Daten und der Benachrichtigung, dass dieser Standby-Server sie geschrieben und geflusht, aber noch nicht angewendet hat. Damit lässt sich die Verzögerung abschätzen, die die Stufe `on` von `synchronous_commit` beim Commit verursacht hat, wenn dieser Server als synchroner Standby konfiguriert war. |
| `replay_lag` | `interval` | Verstrichene Zeit zwischen dem lokalen Flushen aktueller WAL-Daten und der Benachrichtigung, dass dieser Standby-Server sie geschrieben, geflusht und angewendet hat. Damit lässt sich die Verzögerung abschätzen, die die Stufe `remote_apply` von `synchronous_commit` beim Commit verursacht hat, wenn dieser Server als synchroner Standby konfiguriert war. |
| `sync_priority` | `integer` | Priorität dieses Standby-Servers für die Auswahl als synchroner Standby bei prioritätsbasierter synchroner Replikation. Bei quorum-basierter synchroner Replikation hat dies keine Wirkung. |
| `sync_state` | `text` | Synchronisationszustand dieses Standby-Servers. Mögliche Werte sind `async` (asynchron), `potential` (der Standby ist derzeit asynchron, kann aber synchron werden, falls einer der aktuellen synchronen Standbys ausfällt), `sync` (synchron) und `quorum` (der Standby gilt als Kandidat für Quorum-Standbys). |
| `reply_time` | `timestamp with time zone` | Sendezeitpunkt der letzten Antwortnachricht, die vom Standby-Server empfangen wurde. |

Die in der Sicht `pg_stat_replication` gemeldeten Verzögerungszeiten messen, wie lange es dauert, aktuelle WAL-Daten zu schreiben, zu flushen und wiederzugeben und bis der Sender davon erfährt. Diese Zeiten stellen die Commit-Verzögerung dar, die jede Stufe des synchronen Commits verursacht hat oder verursacht hätte, wenn der entfernte Server als synchroner Standby konfiguriert gewesen wäre. Bei einem asynchronen Standby nähert die Spalte `replay_lag` die Verzögerung an, nach der aktuelle Transaktionen für Abfragen sichtbar wurden. Wenn der Standby-Server den sendenden Server vollständig eingeholt hat und keine weitere WAL-Aktivität vorhanden ist, werden die zuletzt gemessenen Verzögerungszeiten noch kurz angezeigt und wechseln dann zu `NULL`.

Verzögerungszeiten funktionieren bei physischer Replikation automatisch. Plugins für logisches Decoding können optional Nachverfolgungsnachrichten ausgeben; tun sie das nicht, zeigt der Nachverfolgungsmechanismus einfach `NULL` als Verzögerung an.

> **Hinweis**
>
> Die gemeldeten Verzögerungszeiten sind keine Vorhersage, wie lange der Standby bei aktueller Wiedergabegeschwindigkeit braucht, um den sendenden Server einzuholen. Ein solches System würde ähnliche Zeiten anzeigen, solange neues WAL erzeugt wird, sich aber unterscheiden, wenn der Sender untätig wird. Insbesondere zeigt `pg_stat_replication`, wenn der Standby vollständig aufgeholt hat, die Zeit zum Schreiben, Flushen und Wiedergeben der zuletzt gemeldeten WAL-Position an, nicht null, wie manche Benutzer erwarten könnten. Das entspricht dem Ziel, Verzögerungen beim synchronen Commit und bei der Transaktionssichtbarkeit für aktuelle Schreibtransaktionen zu messen. Um Verwirrung bei Benutzern zu verringern, die ein anderes Verzögerungsmodell erwarten, wechseln die Verzögerungsspalten auf einem vollständig wiedergegebenen, untätigen System nach kurzer Zeit wieder zu `NULL`. Überwachungssysteme sollten entscheiden, ob sie dies als fehlende Daten, als null oder als letzten bekannten Wert darstellen.

### 27.2.5. pg_stat_replication_slots

Die Sicht `pg_stat_replication_slots` enthält eine Zeile pro logischem Replikationsslot und zeigt Statistiken zu dessen Nutzung.

**Tabelle 27.15. Sicht `pg_stat_replication_slots`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `slot_name` | `text` | Ein clusterweit eindeutiger Bezeichner für den Replikationsslot. |
| `spill_txns` | `bigint` | Anzahl der Transaktionen, die auf Platte ausgelagert wurden, sobald der von logischem Decoding zum Decodieren von Änderungen aus WAL verwendete Speicher `logical_decoding_work_mem` überschritten hat. Der Zähler wird sowohl für Top-Level-Transaktionen als auch für Subtransaktionen erhöht. |
| `spill_count` | `bigint` | Anzahl der Auslagerungen von Transaktionen auf Platte beim Decodieren von Änderungen aus WAL für diesen Slot. Dieser Zähler wird jedes Mal erhöht, wenn eine Transaktion ausgelagert wird; dieselbe Transaktion kann mehrfach ausgelagert werden. |
| `spill_bytes` | `bigint` | Menge der decodierten Transaktionsdaten, die beim Decodieren von Änderungen aus WAL für diesen Slot auf Platte ausgelagert wurde. Dieser und andere Spill-Zähler können verwendet werden, um die beim logischen Decoding aufgetretene I/O abzuschätzen und `logical_decoding_work_mem` abzustimmen. |
| `stream_txns` | `bigint` | Anzahl laufender Transaktionen, die an das Decoding-Ausgabeplugin gestreamt wurden, nachdem der von logischem Decoding zum Decodieren von Änderungen aus WAL für diesen Slot verwendete Speicher `logical_decoding_work_mem` überschritten hat. Streaming funktioniert nur mit Top-Level-Transaktionen; Subtransaktionen können nicht unabhängig gestreamt werden, daher wird der Zähler für Subtransaktionen nicht erhöht. |
| `stream_count` | `bigint` | Anzahl der Streaming-Vorgänge laufender Transaktionen an das Decoding-Ausgabeplugin beim Decodieren von Änderungen aus WAL für diesen Slot. Dieser Zähler wird bei jedem Streaming einer Transaktion erhöht; dieselbe Transaktion kann mehrfach gestreamt werden. |
| `stream_bytes` | `bigint` | Menge der Transaktionsdaten, die beim Decodieren von Änderungen aus WAL für diesen Slot für das Streaming laufender Transaktionen an das Decoding-Ausgabeplugin decodiert wurde. Dieser und andere Streaming-Zähler für diesen Slot können zur Abstimmung von `logical_decoding_work_mem` verwendet werden. |
| `total_txns` | `bigint` | Anzahl decodierter Transaktionen, die für diesen Slot an das Decoding-Ausgabeplugin gesendet wurden. Gezählt werden nur Top-Level-Transaktionen; Subtransaktionen erhöhen den Zähler nicht. Enthalten sind auch Transaktionen, die gestreamt und/oder ausgelagert wurden. |
| `total_bytes` | `bigint` | Menge der Transaktionsdaten, die beim Decodieren von Änderungen aus WAL für diesen Slot zum Senden von Transaktionen an das Decoding-Ausgabeplugin decodiert wurde. Enthalten sind auch Daten, die gestreamt und/oder ausgelagert wurden. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

### 27.2.6. pg_stat_wal_receiver

Die Sicht `pg_stat_wal_receiver` enthält nur eine Zeile und zeigt Statistiken zum WAL-Empfänger von dem Server, mit dem dieser Empfänger verbunden ist.

**Tabelle 27.16. Sicht `pg_stat_wal_receiver`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID des WAL-Empfänger-Prozesses. |
| `status` | `text` | Aktivitätsstatus des WAL-Empfänger-Prozesses. |
| `receive_start_lsn` | `pg_lsn` | Erste Write-Ahead-Log-Position, die beim Start des WAL-Empfängers verwendet wird. |
| `receive_start_tli` | `integer` | Erste Timeline-Nummer, die beim Start des WAL-Empfängers verwendet wird. |
| `written_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die bereits empfangen und auf Platte geschrieben, aber noch nicht geflusht wurde. Dies sollte nicht für Datenintegritätsprüfungen verwendet werden. |
| `flushed_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die bereits empfangen und auf Platte geflusht wurde; Anfangswert dieses Feldes ist die erste Log-Position, die beim Start des WAL-Empfängers verwendet wird. |
| `received_tli` | `integer` | Timeline-Nummer der letzten Write-Ahead-Log-Position, die empfangen und auf Platte geflusht wurde; Anfangswert dieses Feldes ist die Timeline-Nummer der ersten Log-Position, die beim Start des WAL-Empfängers verwendet wird. |
| `last_msg_send_time` | `timestamp with time zone` | Sendezeitpunkt der letzten Nachricht, die vom ursprünglichen WAL-Sender empfangen wurde. |
| `last_msg_receipt_time` | `timestamp with time zone` | Empfangszeitpunkt der letzten Nachricht, die vom ursprünglichen WAL-Sender empfangen wurde. |
| `latest_end_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die dem ursprünglichen WAL-Sender gemeldet wurde. |
| `latest_end_time` | `timestamp with time zone` | Zeitpunkt der letzten Write-Ahead-Log-Position, die dem ursprünglichen WAL-Sender gemeldet wurde. |
| `slot_name` | `text` | Name des Replikationsslots, der von diesem WAL-Empfänger verwendet wird. |
| `sender_host` | `text` | Host der PostgreSQL-Instanz, mit der dieser WAL-Empfänger verbunden ist. Dies kann ein Hostname, eine IP-Adresse oder ein Verzeichnispfad sein, wenn die Verbindung über einen Unix-Socket läuft. Der Pfadfall ist daran zu erkennen, dass er immer ein absoluter Pfad ist, der mit `/` beginnt. |
| `sender_port` | `integer` | Portnummer der PostgreSQL-Instanz, mit der dieser WAL-Empfänger verbunden ist. |
| `conninfo` | `text` | Verbindungszeichenkette, die von diesem WAL-Empfänger verwendet wird; sicherheitssensitive Felder sind unkenntlich gemacht. |

### 27.2.7. pg_stat_recovery_prefetch

Die Sicht `pg_stat_recovery_prefetch` enthält nur eine Zeile. Die Spalten `wal_distance`, `block_distance` und `io_depth` zeigen aktuelle Werte; die übrigen Spalten zeigen kumulative Zähler, die mit der Funktion `pg_stat_reset_shared` zurückgesetzt werden können.

**Tabelle 27.17. Sicht `pg_stat_recovery_prefetch`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |
| `prefetch` | `bigint` | Anzahl der Blöcke, die vorab geladen wurden, weil sie sich nicht im Buffer-Pool befanden. |
| `hit` | `bigint` | Anzahl der Blöcke, die nicht vorab geladen wurden, weil sie sich bereits im Buffer-Pool befanden. |
| `skip_init` | `bigint` | Anzahl der Blöcke, die nicht vorab geladen wurden, weil sie mit Nullen initialisiert würden. |
| `skip_new` | `bigint` | Anzahl der Blöcke, die nicht vorab geladen wurden, weil sie noch nicht existierten. |
| `skip_fpw` | `bigint` | Anzahl der Blöcke, die nicht vorab geladen wurden, weil im WAL ein Full-Page-Image enthalten war. |
| `skip_rep` | `bigint` | Anzahl der Blöcke, die nicht vorab geladen wurden, weil sie bereits kurz zuvor vorab geladen worden waren. |
| `wal_distance` | `int` | Wie viele Bytes der Prefetcher vorausschaut. |
| `block_distance` | `int` | Wie viele Blöcke der Prefetcher vorausschaut. |
| `io_depth` | `int` | Wie viele Prefetches gestartet wurden, deren Abschluss aber noch nicht bekannt ist. |

### 27.2.8. pg_stat_subscription

**Tabelle 27.18. Sicht `pg_stat_subscription`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `subid` | `oid` | OID der Subskription. |
| `subname` | `name` | Name der Subskription. |
| `worker_type` | `text` | Typ des Subskriptions-Worker-Prozesses. Mögliche Typen sind `apply`, `parallel apply` und `table synchronization`. |
| `pid` | `integer` | Prozess-ID des Subskriptions-Worker-Prozesses. |
| `leader_pid` | `integer` | Prozess-ID des führenden Apply-Workers, wenn dieser Prozess ein paralleler Apply-Worker ist; `NULL`, wenn dieser Prozess ein führender Apply-Worker oder ein Tabellensynchronisations-Worker ist. |
| `relid` | `oid` | OID der Relation, die der Worker synchronisiert; `NULL` für den führenden Apply-Worker und parallele Apply-Worker. |
| `received_lsn` | `pg_lsn` | Letzte empfangene Write-Ahead-Log-Position; Anfangswert dieses Feldes ist `0`; `NULL` für parallele Apply-Worker. |
| `last_msg_send_time` | `timestamp with time zone` | Sendezeitpunkt der letzten Nachricht, die vom ursprünglichen WAL-Sender empfangen wurde; `NULL` für parallele Apply-Worker. |
| `last_msg_receipt_time` | `timestamp with time zone` | Empfangszeitpunkt der letzten Nachricht, die vom ursprünglichen WAL-Sender empfangen wurde; `NULL` für parallele Apply-Worker. |
| `latest_end_lsn` | `pg_lsn` | Letzte Write-Ahead-Log-Position, die dem ursprünglichen WAL-Sender gemeldet wurde; `NULL` für parallele Apply-Worker. |
| `latest_end_time` | `timestamp with time zone` | Zeitpunkt der letzten Write-Ahead-Log-Position, die dem ursprünglichen WAL-Sender gemeldet wurde; `NULL` für parallele Apply-Worker. |

### 27.2.9. pg_stat_subscription_stats

Die Sicht `pg_stat_subscription_stats` enthält eine Zeile pro Subskription.

**Tabelle 27.19. Sicht `pg_stat_subscription_stats`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `subid` | `oid` | OID der Subskription. |
| `subname` | `name` | Name der Subskription. |
| `apply_error_count` | `bigint` | Anzahl der Fehler beim Anwenden von Änderungen. Beachten Sie, dass jeder Konflikt, der zu einem Apply-Fehler führt, sowohl in `apply_error_count` als auch im entsprechenden Konfliktzähler gezählt wird, etwa `confl_*`. |
| `sync_error_count` | `bigint` | Anzahl der Fehler während der anfänglichen Tabellensynchronisierung. |
| `confl_insert_exists` | `bigint` | Anzahl der Fälle, in denen ein Zeileneinfügen beim Anwenden von Änderungen eine nicht aufschiebbare eindeutige Einschränkung verletzte. Einzelheiten zu diesem Konflikt finden Sie bei `insert_exists`. |
| `confl_update_origin_differs` | `bigint` | Anzahl der Fälle, in denen beim Anwenden von Änderungen ein Update auf eine Zeile angewendet wurde, die zuvor von einer anderen Quelle geändert worden war. Einzelheiten zu diesem Konflikt finden Sie bei `update_origin_differs`. |
| `confl_update_exists` | `bigint` | Anzahl der Fälle, in denen ein aktualisierter Zeilenwert beim Anwenden von Änderungen eine nicht aufschiebbare eindeutige Einschränkung verletzte. Einzelheiten zu diesem Konflikt finden Sie bei `update_exists`. |
| `confl_update_missing` | `bigint` | Anzahl der Fälle, in denen das zu aktualisierende Tupel beim Anwenden von Änderungen nicht gefunden wurde. Einzelheiten zu diesem Konflikt finden Sie bei `update_missing`. |
| `confl_delete_origin_differs` | `bigint` | Anzahl der Fälle, in denen beim Anwenden von Änderungen eine Löschoperation auf eine Zeile angewendet wurde, die zuvor von einer anderen Quelle geändert worden war. Einzelheiten zu diesem Konflikt finden Sie bei `delete_origin_differs`. |
| `confl_delete_missing` | `bigint` | Anzahl der Fälle, in denen das zu löschende Tupel beim Anwenden von Änderungen nicht gefunden wurde. Einzelheiten zu diesem Konflikt finden Sie bei `delete_missing`. |
| `confl_multiple_unique_conflicts` | `bigint` | Anzahl der Fälle, in denen ein Zeileneinfügen oder ein aktualisierter Zeilenwert beim Anwenden von Änderungen mehrere nicht aufschiebbare eindeutige Einschränkungen verletzte. Einzelheiten zu diesem Konflikt finden Sie bei `multiple_unique_conflicts`. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

### 27.2.10. pg_stat_ssl

Die Sicht `pg_stat_ssl` enthält eine Zeile pro Backend- oder WAL-Sender-Prozess und zeigt Statistiken zur SSL-Nutzung auf dieser Verbindung. Sie kann über die Spalte `pid` mit `pg_stat_activity` oder `pg_stat_replication` verknüpft werden, um weitere Details zur Verbindung zu erhalten.

**Tabelle 27.20. Sicht `pg_stat_ssl`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID eines Backend- oder WAL-Sender-Prozesses. |
| `ssl` | `boolean` | Wahr, wenn SSL auf dieser Verbindung verwendet wird. |
| `version` | `text` | Verwendete SSL-Version oder `NULL`, wenn SSL auf dieser Verbindung nicht verwendet wird. |
| `cipher` | `text` | Name der verwendeten SSL-Chiffre oder `NULL`, wenn SSL auf dieser Verbindung nicht verwendet wird. |
| `bits` | `integer` | Anzahl der Bits im verwendeten Verschlüsselungsalgorithmus oder `NULL`, wenn SSL auf dieser Verbindung nicht verwendet wird. |
| `client_dn` | `text` | Feld Distinguished Name (DN) aus dem verwendeten Clientzertifikat oder `NULL`, wenn kein Clientzertifikat bereitgestellt wurde oder SSL auf dieser Verbindung nicht verwendet wird. Dieses Feld wird abgeschnitten, wenn das DN-Feld länger als `NAMEDATALEN` ist (64 Zeichen in einer Standardkompilierung). |
| `client_serial` | `numeric` | Seriennummer des Clientzertifikats oder `NULL`, wenn kein Clientzertifikat bereitgestellt wurde oder SSL auf dieser Verbindung nicht verwendet wird. Die Kombination aus Zertifikatseriennummer und Zertifikatsaussteller identifiziert ein Zertifikat eindeutig, sofern der Aussteller Seriennummern nicht irrtümlich wiederverwendet. |
| `issuer_dn` | `text` | DN des Ausstellers des Clientzertifikats oder `NULL`, wenn kein Clientzertifikat bereitgestellt wurde oder SSL auf dieser Verbindung nicht verwendet wird. Dieses Feld wird wie `client_dn` abgeschnitten. |

### 27.2.11. pg_stat_gssapi

Die Sicht `pg_stat_gssapi` enthält eine Zeile pro Backend und zeigt Informationen zur GSSAPI-Nutzung auf dieser Verbindung. Sie kann über die Spalte `pid` mit `pg_stat_activity` oder `pg_stat_replication` verknüpft werden, um weitere Details zur Verbindung zu erhalten.

**Tabelle 27.21. Sicht `pg_stat_gssapi`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID eines Backends. |
| `gss_authenticated` | `boolean` | Wahr, wenn für diese Verbindung GSSAPI-Authentifizierung verwendet wurde. |
| `principal` | `text` | Principal, der zur Authentifizierung dieser Verbindung verwendet wurde, oder `NULL`, wenn GSSAPI nicht zur Authentifizierung dieser Verbindung verwendet wurde. Dieses Feld wird abgeschnitten, wenn der Principal länger als `NAMEDATALEN` ist (64 Zeichen in einer Standardkompilierung). |
| `encrypted` | `boolean` | Wahr, wenn auf dieser Verbindung GSSAPI-Verschlüsselung verwendet wird. |
| `credentials_delegated` | `boolean` | Wahr, wenn auf dieser Verbindung GSSAPI-Anmeldeinformationen delegiert wurden. |

### 27.2.12. pg_stat_archiver

Die Sicht `pg_stat_archiver` hat immer eine einzelne Zeile und enthält Daten zum Archiver-Prozess des Clusters.

**Tabelle 27.22. Sicht `pg_stat_archiver`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `archived_count` | `bigint` | Anzahl der WAL-Dateien, die erfolgreich archiviert wurden. |
| `last_archived_wal` | `text` | Name der zuletzt erfolgreich archivierten WAL-Datei. |
| `last_archived_time` | `timestamp with time zone` | Zeitpunkt der letzten erfolgreichen Archivierungsoperation. |
| `failed_count` | `bigint` | Anzahl fehlgeschlagener Versuche, WAL-Dateien zu archivieren. |
| `last_failed_wal` | `text` | Name der WAL-Datei der letzten fehlgeschlagenen Archivierungsoperation. |
| `last_failed_time` | `timestamp with time zone` | Zeitpunkt der letzten fehlgeschlagenen Archivierungsoperation. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

Normalerweise werden WAL-Dateien der Reihe nach archiviert, von der ältesten zur neuesten. Das ist jedoch nicht garantiert und gilt unter besonderen Umständen nicht, etwa beim Promoten eines Standbys oder nach einer Wiederherstellung nach einem Absturz. Deshalb ist es nicht sicher anzunehmen, dass alle Dateien, die älter als `last_archived_wal` sind, ebenfalls erfolgreich archiviert wurden.

### 27.2.13. pg_stat_io

Die Sicht `pg_stat_io` enthält eine Zeile für jede Kombination aus Backend-Typ, Ziel-I/O-Objekt und I/O-Kontext und zeigt clusterweite I/O-Statistiken. Kombinationen, die keinen Sinn ergeben, werden ausgelassen.

Derzeit werden I/O auf Relationen, etwa Tabellen und Indizes, sowie WAL-Aktivität erfasst. Relationen-I/O, die Shared Buffers umgeht, etwa beim Verschieben einer Tabelle von einem Tablespace in einen anderen, wird derzeit jedoch nicht erfasst.

**Tabelle 27.23. Sicht `pg_stat_io`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `backend_type` | `text` | Typ des Backends, zum Beispiel Background Worker oder Autovacuum Worker. Weitere Informationen zu `backend_type` finden Sie bei `pg_stat_activity`. Manche Backend-Typen sammeln keine Statistiken zu I/O-Operationen und erscheinen daher nicht in der Sicht. |
| `object` | `text` | Zielobjekt einer I/O-Operation. Mögliche Werte sind `relation` für permanente Relationen, `temp relation` für temporäre Relationen und `wal` für Write-Ahead-Logs. |
| `context` | `text` | Kontext einer I/O-Operation. Mögliche Werte sind `normal` (Standardkontext für eine Art von I/O-Operation; Relationen werden standardmäßig aus Shared Buffers gelesen und dorthin geschrieben), `init` (I/O-Operationen beim Erzeugen von WAL-Segmenten), `vacuum` (I/O-Operationen außerhalb von Shared Buffers beim Vacuuming und Analysieren permanenter Relationen), `bulkread` (bestimmte große Lese-I/O-Operationen außerhalb von Shared Buffers, etwa ein sequenzieller Scan einer großen Tabelle) und `bulkwrite` (bestimmte große Schreib-I/O-Operationen außerhalb von Shared Buffers, etwa `COPY`). Vacuums temporärer Tabellen verwenden denselben lokalen Buffer-Pool wie andere I/O-Operationen temporärer Tabellen und werden im Kontext `normal` erfasst. |
| `reads` | `bigint` | Anzahl der Leseoperationen. |
| `read_bytes` | `numeric` | Gesamtgröße der Leseoperationen in Byte. |
| `read_time` | `double precision` | Zeit in Millisekunden, die auf Leseoperationen gewartet wurde, wenn `track_io_timing` aktiviert ist und `object` nicht `wal` ist, oder wenn `track_wal_io_timing` aktiviert ist und `object` `wal` ist; andernfalls 0. |
| `writes` | `bigint` | Anzahl der Schreiboperationen. |
| `write_bytes` | `numeric` | Gesamtgröße der Schreiboperationen in Byte. |
| `write_time` | `double precision` | Zeit in Millisekunden, die auf Schreiboperationen gewartet wurde, wenn `track_io_timing` aktiviert ist und `object` nicht `wal` ist, oder wenn `track_wal_io_timing` aktiviert ist und `object` `wal` ist; andernfalls 0. |
| `writebacks` | `bigint` | Anzahl der Einheiten der Größe `BLCKSZ` (typischerweise 8 kB), für die der Prozess den Kernel angewiesen hat, sie in dauerhaften Speicher auszuschreiben. |
| `writeback_time` | `double precision` | Zeit in Millisekunden, die auf Writeback-Operationen gewartet wurde, wenn `track_io_timing` aktiviert ist; andernfalls 0. Dazu gehört die Zeit in der Warteschlange für Schreibanforderungen und möglicherweise die Zeit zum Ausschreiben der Dirty Data. |
| `extends` | `bigint` | Anzahl der Relation-Extend-Operationen. |
| `extend_bytes` | `numeric` | Gesamtgröße der Relation-Extend-Operationen in Byte. |
| `extend_time` | `double precision` | Zeit in Millisekunden, die auf Extend-Operationen gewartet wurde, wenn `track_io_timing` aktiviert ist und `object` nicht `wal` ist, oder wenn `track_wal_io_timing` aktiviert ist und `object` `wal` ist; andernfalls 0. |
| `hits` | `bigint` | Anzahl der Fälle, in denen ein gewünschter Block in einem Shared Buffer gefunden wurde. |
| `evictions` | `bigint` | Anzahl der Fälle, in denen ein Block aus einem Shared oder Local Buffer ausgeschrieben wurde, um ihn für eine andere Verwendung verfügbar zu machen. Im Kontext `normal` zählt dies, wie oft ein Block aus einem Buffer verdrängt und durch einen anderen Block ersetzt wurde. In den Kontexten `bulkwrite`, `bulkread` und `vacuum` zählt dies, wie oft ein Block aus Shared Buffers verdrängt wurde, um den Shared Buffer einem separaten, größenbegrenzten Ringbuffer für eine Bulk-I/O-Operation hinzuzufügen. |
| `reuses` | `bigint` | Anzahl der Fälle, in denen ein vorhandener Buffer in einem größenbegrenzten Ringbuffer außerhalb von Shared Buffers als Teil einer I/O-Operation in den Kontexten `bulkread`, `bulkwrite` oder `vacuum` wiederverwendet wurde. |
| `fsyncs` | `bigint` | Anzahl der `fsync`-Aufrufe. Diese werden nur im Kontext `normal` erfasst. |
| `fsync_time` | `double precision` | Zeit in Millisekunden, die auf `fsync`-Operationen gewartet wurde, wenn `track_io_timing` aktiviert ist und `object` nicht `wal` ist, oder wenn `track_wal_io_timing` aktiviert ist und `object` `wal` ist; andernfalls 0. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

Manche Backend-Typen führen niemals I/O-Operationen auf bestimmten I/O-Objekten und/oder in bestimmten I/O-Kontexten aus. Diese Zeilen werden in der Sicht ausgelassen. Beispielsweise führt der Checkpointer keine Checkpoints für temporäre Tabellen aus; daher gibt es keine Zeilen mit `backend_type` `checkpointer` und `object` `temp relation`.

Außerdem werden manche I/O-Operationen von bestimmten Backend-Typen oder auf bestimmten I/O-Objekten und/oder in bestimmten I/O-Kontexten niemals ausgeführt. Diese Zellen enthalten dann `NULL`. Temporäre Tabellen werden beispielsweise nicht per `fsync` synchronisiert, daher ist `fsyncs` für `object` `temp relation` `NULL`. Außerdem führt der Background Writer keine Leseoperationen aus, daher ist `reads` in Zeilen mit `backend_type` `background writer` `NULL`.

Für das Objekt `wal` erfassen `fsyncs` und `fsync_time` die `fsync`-Aktivität von WAL-Dateien, die in `issue_xlog_fsync` ausgeführt wird. `writes` und `write_time` erfassen die Schreibaktivität von WAL-Dateien, die in `XLogWrite` ausgeführt wird. Weitere Informationen finden Sie in [Abschnitt 28.5](28_Zuverlässigkeit_und_Write_Ahead_Log.md#285-walkonfiguration).

`pg_stat_io` kann Hinweise für die Datenbankabstimmung liefern. Zum Beispiel:

- Eine hohe Anzahl von `evictions` kann darauf hinweisen, dass Shared Buffers vergrößert werden sollten.
- Client-Backends verlassen sich auf den Checkpointer, damit Daten dauerhaft gespeichert werden. Eine große Anzahl von `fsyncs` durch Client-Backends kann auf eine Fehlkonfiguration der Shared Buffers oder des Checkpointers hinweisen. Weitere Informationen zur Konfiguration des Checkpointers finden Sie in [Abschnitt 28.5](28_Zuverlässigkeit_und_Write_Ahead_Log.md#285-walkonfiguration).
- Normalerweise sollten Client-Backends sich darauf verlassen können, dass Hilfsprozesse wie der Checkpointer und der Background Writer Dirty Data so weit wie möglich ausschreiben. Eine große Anzahl von Schreibvorgängen durch Client-Backends kann auf eine Fehlkonfiguration der Shared Buffers oder des Checkpointers hinweisen. Weitere Informationen zur Konfiguration des Checkpointers finden Sie in [Abschnitt 28.5](28_Zuverlässigkeit_und_Write_Ahead_Log.md#285-walkonfiguration).

> **Hinweis**
>
> Spalten, die I/O-Wartezeiten erfassen, sind nur dann ungleich null, wenn `track_io_timing` aktiviert ist. Bei der gemeinsamen Betrachtung dieser Spalten und der zugehörigen I/O-Operationen ist Vorsicht geboten, falls `track_io_timing` nicht während der gesamten Zeit seit dem letzten Zurücksetzen der Statistiken aktiviert war.

### 27.2.14. pg_stat_bgwriter

Die Sicht `pg_stat_bgwriter` hat immer eine einzelne Zeile und enthält Daten zum Background Writer des Clusters.

**Tabelle 27.24. Sicht `pg_stat_bgwriter`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `buffers_clean` | `bigint` | Anzahl der Buffer, die vom Background Writer geschrieben wurden. |
| `maxwritten_clean` | `bigint` | Anzahl der Fälle, in denen der Background Writer einen Cleaning-Scan abgebrochen hat, weil er zu viele Buffer geschrieben hatte. |
| `buffers_alloc` | `bigint` | Anzahl der zugewiesenen Buffer. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

### 27.2.15. pg_stat_checkpointer

Die Sicht `pg_stat_checkpointer` hat immer eine einzelne Zeile und enthält Daten zum Checkpointer-Prozess des Clusters.

**Tabelle 27.25. Sicht `pg_stat_checkpointer`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `num_timed` | `bigint` | Anzahl der geplanten Checkpoints aufgrund eines Timeouts. |
| `num_requested` | `bigint` | Anzahl der angeforderten Checkpoints. |
| `num_done` | `bigint` | Anzahl der Checkpoints, die ausgeführt wurden. |
| `restartpoints_timed` | `bigint` | Anzahl der geplanten Restartpoints aufgrund eines Timeouts oder nach einem fehlgeschlagenen Versuch, sie auszuführen. |
| `restartpoints_req` | `bigint` | Anzahl der angeforderten Restartpoints. |
| `restartpoints_done` | `bigint` | Anzahl der Restartpoints, die ausgeführt wurden. |
| `write_time` | `double precision` | Gesamtzeit in Millisekunden, die beim Verarbeiten von Checkpoints und Restartpoints in dem Teil verbracht wurde, in dem Dateien auf Platte geschrieben werden. |
| `sync_time` | `double precision` | Gesamtzeit in Millisekunden, die beim Verarbeiten von Checkpoints und Restartpoints in dem Teil verbracht wurde, in dem Dateien auf Platte synchronisiert werden. |
| `buffers_written` | `bigint` | Anzahl der Shared Buffers, die während Checkpoints und Restartpoints geschrieben wurden. |
| `slru_written` | `bigint` | Anzahl der SLRU-Buffer, die während Checkpoints und Restartpoints geschrieben wurden. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

Checkpoints können übersprungen werden, wenn der Server seit dem letzten Checkpoint untätig war. `num_timed` und `num_requested` zählen sowohl abgeschlossene als auch übersprungene Checkpoints, während `num_done` nur die abgeschlossenen verfolgt. Ebenso können Restartpoints übersprungen werden, wenn der zuletzt wiedergegebene Checkpoint-Datensatz bereits der letzte Restartpoint ist. `restartpoints_timed` und `restartpoints_req` zählen sowohl abgeschlossene als auch übersprungene Restartpoints, während `restartpoints_done` nur die abgeschlossenen verfolgt.

### 27.2.16. pg_stat_wal

Die Sicht `pg_stat_wal` hat immer eine einzelne Zeile und enthält Daten zur WAL-Aktivität des Clusters.

**Tabelle 27.26. Sicht `pg_stat_wal`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `wal_records` | `bigint` | Gesamtanzahl der erzeugten WAL-Datensätze. |
| `wal_fpi` | `bigint` | Gesamtanzahl der erzeugten WAL-Full-Page-Images. |
| `wal_bytes` | `numeric` | Gesamtmenge des erzeugten WAL in Byte. |
| `wal_buffers_full` | `bigint` | Anzahl der Fälle, in denen WAL-Daten auf Platte geschrieben wurden, weil die WAL-Buffer voll wurden. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

### 27.2.17. pg_stat_database

Die Sicht `pg_stat_database` enthält eine Zeile für jede Datenbank im Cluster sowie eine zusätzliche Zeile für gemeinsam genutzte Objekte und zeigt datenbankweite Statistiken.

**Tabelle 27.27. Sicht `pg_stat_database`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `datid` | `oid` | OID dieser Datenbank oder `0` für Objekte, die zu einer gemeinsam genutzten Relation gehören. |
| `datname` | `name` | Name dieser Datenbank oder `NULL` für gemeinsam genutzte Objekte. |
| `numbackends` | `integer` | Anzahl der Backends, die derzeit mit dieser Datenbank verbunden sind, oder `NULL` für gemeinsam genutzte Objekte. Dies ist die einzige Spalte dieser Sicht, die einen Wert zum aktuellen Zustand zurückgibt; alle anderen Spalten liefern seit dem letzten Zurücksetzen aufsummierte Werte. |
| `xact_commit` | `bigint` | Anzahl der Transaktionen in dieser Datenbank, die committet wurden. |
| `xact_rollback` | `bigint` | Anzahl der Transaktionen in dieser Datenbank, die zurückgerollt wurden. |
| `blks_read` | `bigint` | Anzahl der in dieser Datenbank gelesenen Plattenblöcke. |
| `blks_hit` | `bigint` | Anzahl der Fälle, in denen Plattenblöcke bereits im Buffer-Cache gefunden wurden, sodass kein Lesen nötig war. Dies umfasst nur Treffer im PostgreSQL-Buffer-Cache, nicht im Dateisystem-Cache des Betriebssystems. |
| `tup_returned` | `bigint` | Anzahl der lebenden Zeilen, die durch sequenzielle Scans geholt wurden, und der Indexeinträge, die durch Indexscans in dieser Datenbank zurückgegeben wurden. |
| `tup_fetched` | `bigint` | Anzahl der lebenden Zeilen, die durch Indexscans in dieser Datenbank geholt wurden. |
| `tup_inserted` | `bigint` | Anzahl der Zeilen, die durch Abfragen in diese Datenbank eingefügt wurden. |
| `tup_updated` | `bigint` | Anzahl der Zeilen, die durch Abfragen in dieser Datenbank aktualisiert wurden. |
| `tup_deleted` | `bigint` | Anzahl der Zeilen, die durch Abfragen in dieser Datenbank gelöscht wurden. |
| `conflicts` | `bigint` | Anzahl der Abfragen, die in dieser Datenbank aufgrund von Konflikten mit der Wiederherstellung abgebrochen wurden. Konflikte treten nur auf Standby-Servern auf; Einzelheiten finden Sie bei `pg_stat_database_conflicts`. |
| `temp_files` | `bigint` | Anzahl der temporären Dateien, die durch Abfragen in dieser Datenbank erzeugt wurden. Alle temporären Dateien werden gezählt, unabhängig davon, warum sie erzeugt wurden, etwa zum Sortieren oder Hashing, und unabhängig von der Einstellung `log_temp_files`. |
| `temp_bytes` | `bigint` | Gesamtmenge der Daten, die durch Abfragen in dieser Datenbank in temporäre Dateien geschrieben wurden. Alle temporären Dateien werden gezählt, unabhängig davon, warum sie erzeugt wurden, und unabhängig von der Einstellung `log_temp_files`. |
| `deadlocks` | `bigint` | Anzahl der in dieser Datenbank erkannten Deadlocks. |
| `checksum_failures` | `bigint` | Anzahl der Prüfsummenfehler auf Datenseiten, die in dieser Datenbank oder an einem gemeinsam genutzten Objekt erkannt wurden, oder `NULL`, wenn Datenprüfsummen deaktiviert sind. |
| `checksum_last_failure` | `timestamp with time zone` | Zeitpunkt, zu dem der letzte Prüfsummenfehler auf einer Datenseite in dieser Datenbank oder an einem gemeinsam genutzten Objekt erkannt wurde, oder `NULL`, wenn Datenprüfsummen deaktiviert sind. |
| `blk_read_time` | `double precision` | Zeit in Millisekunden, die Backends in dieser Datenbank mit dem Lesen von Datendateiblöcken verbracht haben, wenn `track_io_timing` aktiviert ist; andernfalls 0. |
| `blk_write_time` | `double precision` | Zeit in Millisekunden, die Backends in dieser Datenbank mit dem Schreiben von Datendateiblöcken verbracht haben, wenn `track_io_timing` aktiviert ist; andernfalls 0. |
| `session_time` | `double precision` | Zeit in Millisekunden, die Datenbanksitzungen in dieser Datenbank verbracht haben. Beachten Sie, dass Statistiken nur aktualisiert werden, wenn sich der Zustand einer Sitzung ändert; wenn Sitzungen lange untätig waren, ist diese Leerlaufzeit daher nicht enthalten. |
| `active_time` | `double precision` | Zeit in Millisekunden, die in dieser Datenbank mit der Ausführung von SQL-Anweisungen verbracht wurde. Dies entspricht den Zuständen `active` und `fastpath function call` in `pg_stat_activity`. |
| `idle_in_transaction_time` | `double precision` | Zeit in Millisekunden, die in dieser Datenbank untätig innerhalb einer Transaktion verbracht wurde. Dies entspricht den Zuständen `idle in transaction` und `idle in transaction (aborted)` in `pg_stat_activity`. |
| `sessions` | `bigint` | Gesamtanzahl der zu dieser Datenbank aufgebauten Sitzungen. |
| `sessions_abandoned` | `bigint` | Anzahl der Datenbanksitzungen zu dieser Datenbank, die beendet wurden, weil die Verbindung zum Client verloren ging. |
| `sessions_fatal` | `bigint` | Anzahl der Datenbanksitzungen zu dieser Datenbank, die durch fatale Fehler beendet wurden. |
| `sessions_killed` | `bigint` | Anzahl der Datenbanksitzungen zu dieser Datenbank, die durch Eingriff eines Operators beendet wurden. |
| `parallel_workers_to_launch` | `bigint` | Anzahl der parallelen Worker, deren Start von Abfragen auf dieser Datenbank geplant war. |
| `parallel_workers_launched` | `bigint` | Anzahl der parallelen Worker, die von Abfragen auf dieser Datenbank gestartet wurden. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

### 27.2.18. pg_stat_database_conflicts

Die Sicht `pg_stat_database_conflicts` enthält eine Zeile pro Datenbank und zeigt datenbankweite Statistiken zu Abfrageabbrüchen, die durch Konflikte mit der Wiederherstellung auf Standby-Servern entstehen. Diese Sicht enthält nur Informationen auf Standby-Servern, da Konflikte auf Primärservern nicht auftreten.

**Tabelle 27.28. Sicht `pg_stat_database_conflicts`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `datid` | `oid` | OID einer Datenbank. |
| `datname` | `name` | Name dieser Datenbank. |
| `confl_tablespace` | `bigint` | Anzahl der Abfragen in dieser Datenbank, die aufgrund gelöschter Tablespaces abgebrochen wurden. |
| `confl_lock` | `bigint` | Anzahl der Abfragen in dieser Datenbank, die aufgrund von Sperrzeitüberschreitungen abgebrochen wurden. |
| `confl_snapshot` | `bigint` | Anzahl der Abfragen in dieser Datenbank, die aufgrund alter Snapshots abgebrochen wurden. |
| `confl_bufferpin` | `bigint` | Anzahl der Abfragen in dieser Datenbank, die aufgrund gepinnter Buffer abgebrochen wurden. |
| `confl_deadlock` | `bigint` | Anzahl der Abfragen in dieser Datenbank, die aufgrund von Deadlocks abgebrochen wurden. |
| `confl_active_logicalslot` | `bigint` | Anzahl der Nutzungen logischer Slots in dieser Datenbank, die aufgrund alter Snapshots oder eines zu niedrigen `wal_level` auf dem Primärserver abgebrochen wurden. |

### 27.2.19. pg_stat_all_tables

Die Sicht `pg_stat_all_tables` enthält eine Zeile für jede Tabelle in der aktuellen Datenbank, einschließlich TOAST-Tabellen, und zeigt Statistiken zu Zugriffen auf diese bestimmte Tabelle. Die Sichten `pg_stat_user_tables` und `pg_stat_sys_tables` enthalten dieselben Informationen, sind aber so gefiltert, dass sie nur Benutzer- beziehungsweise Systemtabellen anzeigen.

**Tabelle 27.29. Sicht `pg_stat_all_tables`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `relid` | `oid` | OID einer Tabelle. |
| `schemaname` | `name` | Name des Schemas, in dem sich diese Tabelle befindet. |
| `relname` | `name` | Name dieser Tabelle. |
| `seq_scan` | `bigint` | Anzahl der auf dieser Tabelle gestarteten sequenziellen Scans. |
| `last_seq_scan` | `timestamp with time zone` | Zeitpunkt des letzten sequenziellen Scans auf dieser Tabelle, basierend auf der jüngsten Transaktionsendezeit. |
| `seq_tup_read` | `bigint` | Anzahl der lebenden Zeilen, die durch sequenzielle Scans geholt wurden. |
| `idx_scan` | `bigint` | Anzahl der auf dieser Tabelle gestarteten Indexscans. |
| `last_idx_scan` | `timestamp with time zone` | Zeitpunkt des letzten Indexscans auf dieser Tabelle, basierend auf der jüngsten Transaktionsendezeit. |
| `idx_tup_fetch` | `bigint` | Anzahl der lebenden Zeilen, die durch Indexscans geholt wurden. |
| `n_tup_ins` | `bigint` | Gesamtanzahl der eingefügten Zeilen. |
| `n_tup_upd` | `bigint` | Gesamtanzahl der aktualisierten Zeilen. Dazu gehören Zeilenaktualisierungen, die in `n_tup_hot_upd` und `n_tup_newpage_upd` gezählt werden, sowie verbleibende Nicht-HOT-Aktualisierungen. |
| `n_tup_del` | `bigint` | Gesamtanzahl der gelöschten Zeilen. |
| `n_tup_hot_upd` | `bigint` | Anzahl der per HOT aktualisierten Zeilen. Das sind Aktualisierungen, bei denen in Indizes keine Nachfolgeversionen erforderlich sind. |
| `n_tup_newpage_upd` | `bigint` | Anzahl der aktualisierten Zeilen, bei denen die Nachfolgeversion auf eine neue Heap-Seite geht und eine ursprüngliche Version mit einem `t_ctid`-Feld zurücklässt, das auf eine andere Heap-Seite zeigt. Dies sind immer Nicht-HOT-Aktualisierungen. |
| `n_live_tup` | `bigint` | Geschätzte Anzahl lebender Zeilen. |
| `n_dead_tup` | `bigint` | Geschätzte Anzahl toter Zeilen. |
| `n_mod_since_analyze` | `bigint` | Geschätzte Anzahl der Zeilen, die seit der letzten Analyse dieser Tabelle geändert wurden. |
| `n_ins_since_vacuum` | `bigint` | Geschätzte Anzahl der Zeilen, die seit dem letzten Vacuum dieser Tabelle eingefügt wurden, ohne `VACUUM FULL`. |
| `last_vacuum` | `timestamp with time zone` | Letzter Zeitpunkt, zu dem diese Tabelle manuell bereinigt wurde, ohne `VACUUM FULL`. |
| `last_autovacuum` | `timestamp with time zone` | Letzter Zeitpunkt, zu dem diese Tabelle vom Autovacuum-Daemon bereinigt wurde. |
| `last_analyze` | `timestamp with time zone` | Letzter Zeitpunkt, zu dem diese Tabelle manuell analysiert wurde. |
| `last_autoanalyze` | `timestamp with time zone` | Letzter Zeitpunkt, zu dem diese Tabelle vom Autovacuum-Daemon analysiert wurde. |
| `vacuum_count` | `bigint` | Anzahl der manuellen Vacuum-Läufe für diese Tabelle, ohne `VACUUM FULL`. |
| `autovacuum_count` | `bigint` | Anzahl der Vacuum-Läufe für diese Tabelle durch den Autovacuum-Daemon. |
| `analyze_count` | `bigint` | Anzahl der manuellen Analyse-Läufe für diese Tabelle. |
| `autoanalyze_count` | `bigint` | Anzahl der Analyse-Läufe für diese Tabelle durch den Autovacuum-Daemon. |
| `total_vacuum_time` | `double precision` | Gesamtzeit in Millisekunden, die diese Tabelle manuell bereinigt wurde, ohne `VACUUM FULL`. Enthalten ist auch die Zeit, die wegen kostenbasierter Verzögerungen im Schlafzustand verbracht wurde. |
| `total_autovacuum_time` | `double precision` | Gesamtzeit in Millisekunden, die diese Tabelle vom Autovacuum-Daemon bereinigt wurde. Enthalten ist auch die Zeit, die wegen kostenbasierter Verzögerungen im Schlafzustand verbracht wurde. |
| `total_analyze_time` | `double precision` | Gesamtzeit in Millisekunden, die diese Tabelle manuell analysiert wurde. Enthalten ist auch die Zeit, die wegen kostenbasierter Verzögerungen im Schlafzustand verbracht wurde. |
| `total_autoanalyze_time` | `double precision` | Gesamtzeit in Millisekunden, die diese Tabelle vom Autovacuum-Daemon analysiert wurde. Enthalten ist auch die Zeit, die wegen kostenbasierter Verzögerungen im Schlafzustand verbracht wurde. |

### 27.2.20. pg_stat_all_indexes

Die Sicht `pg_stat_all_indexes` enthält eine Zeile für jeden Index in der aktuellen Datenbank und zeigt Statistiken zu Zugriffen auf diesen bestimmten Index. Die Sichten `pg_stat_user_indexes` und `pg_stat_sys_indexes` enthalten dieselben Informationen, sind aber so gefiltert, dass sie nur Benutzer- beziehungsweise Systemindizes anzeigen.

**Tabelle 27.30. Sicht `pg_stat_all_indexes`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `relid` | `oid` | OID der Tabelle für diesen Index. |
| `indexrelid` | `oid` | OID dieses Index. |
| `schemaname` | `name` | Name des Schemas, in dem sich dieser Index befindet. |
| `relname` | `name` | Name der Tabelle für diesen Index. |
| `indexrelname` | `name` | Name dieses Index. |
| `idx_scan` | `bigint` | Anzahl der auf diesem Index gestarteten Indexscans. |
| `last_idx_scan` | `timestamp with time zone` | Zeitpunkt des letzten Scans auf diesem Index, basierend auf der jüngsten Transaktionsendezeit. |
| `idx_tup_read` | `bigint` | Anzahl der Indexeinträge, die durch Scans auf diesem Index zurückgegeben wurden. |
| `idx_tup_fetch` | `bigint` | Anzahl der lebenden Tabellenzeilen, die durch einfache Indexscans mit diesem Index geholt wurden. |

Indizes können von einfachen Indexscans, Bitmap-Indexscans und vom Optimizer verwendet werden. Bei einem Bitmap-Scan kann die Ausgabe mehrerer Indizes über `AND`- oder `OR`-Regeln kombiniert werden; daher ist es schwierig, einzelne Heap-Zeilenabrufe bestimmten Indizes zuzuordnen, wenn ein Bitmap-Scan verwendet wird. Deshalb erhöht ein Bitmap-Scan den oder die Zähler `pg_stat_all_indexes.idx_tup_read` für die verwendeten Indizes und den Zähler `pg_stat_all_tables.idx_tup_fetch` für die Tabelle, beeinflusst aber `pg_stat_all_indexes.idx_tup_fetch` nicht. Der Optimizer greift außerdem auf Indizes zu, um bereitgestellte Konstanten zu prüfen, deren Werte außerhalb des aufgezeichneten Bereichs der Optimizer-Statistiken liegen, weil die Optimizer-Statistiken veraltet sein könnten.

> **Hinweis**
>
> Die Zähler `idx_tup_read` und `idx_tup_fetch` können sich auch ohne Bitmap-Scans unterscheiden, weil `idx_tup_read` die aus dem Index geholten Indexeinträge zählt, während `idx_tup_fetch` die aus der Tabelle geholten lebenden Zeilen zählt. Letzterer Wert ist kleiner, wenn tote oder noch nicht committete Zeilen über den Index geholt werden oder wenn Heap-Abrufe durch einen Index-Only-Scan vermieden werden.

> **Hinweis**
>
> Indexscans können bei einer Ausführung manchmal mehrere Indexsuchen durchführen. Jede Indexsuche erhöht `pg_stat_all_indexes.idx_scan`, sodass die Anzahl der Indexscans die Gesamtanzahl der Ausführungen von Indexscan-Executor-Knoten deutlich übersteigen kann.
>
> Dies kann bei Abfragen passieren, die bestimmte SQL-Konstrukte verwenden, um nach Zeilen zu suchen, die einem beliebigen Wert aus einer Liste oder einem Array mehrerer skalarer Werte entsprechen (siehe [Abschnitt 9.25](09_Funktionen_und_Operatoren.md#925-zeilen-und-arrayvergleiche)). Es kann auch bei Abfragen mit einem Konstrukt der Form `column_name = value1 OR column_name = value2 ...` passieren, allerdings nur, wenn der Optimizer das Konstrukt in eine äquivalente mehrwertige Array-Darstellung umformt. Ebenso wird bei B-Tree-Indexscans mit Skip-Scan-Optimierung jedes Mal eine Indexsuche ausgeführt, wenn der Scan auf die nächste Index-Leaf-Page umpositioniert wird, die passende Tupel enthalten könnte (siehe [Abschnitt 11.3](11_Indizes.md#113-mehrspaltige-indizes)).

> **Tipp**
>
> `EXPLAIN ANALYZE` gibt die Gesamtanzahl der Indexsuchen aus, die von jedem Indexscan-Knoten durchgeführt wurden. Ein Beispiel dafür finden Sie in [Abschnitt 14.1.2](14_Hinweise_zur_Performance.md#1412-explain-analyze).

### 27.2.21. pg_statio_all_tables

Die Sicht `pg_statio_all_tables` enthält eine Zeile für jede Tabelle in der aktuellen Datenbank, einschließlich TOAST-Tabellen, und zeigt Statistiken zur I/O auf dieser bestimmten Tabelle. Die Sichten `pg_statio_user_tables` und `pg_statio_sys_tables` enthalten dieselben Informationen, sind aber so gefiltert, dass sie nur Benutzer- beziehungsweise Systemtabellen anzeigen.

**Tabelle 27.31. Sicht `pg_statio_all_tables`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `relid` | `oid` | OID einer Tabelle. |
| `schemaname` | `name` | Name des Schemas, in dem sich diese Tabelle befindet. |
| `relname` | `name` | Name dieser Tabelle. |
| `heap_blks_read` | `bigint` | Anzahl der von dieser Tabelle gelesenen Plattenblöcke. |
| `heap_blks_hit` | `bigint` | Anzahl der Buffer-Treffer in dieser Tabelle. |
| `idx_blks_read` | `bigint` | Anzahl der aus allen Indizes dieser Tabelle gelesenen Plattenblöcke. |
| `idx_blks_hit` | `bigint` | Anzahl der Buffer-Treffer in allen Indizes dieser Tabelle. |
| `toast_blks_read` | `bigint` | Anzahl der aus der TOAST-Tabelle dieser Tabelle gelesenen Plattenblöcke, falls vorhanden. |
| `toast_blks_hit` | `bigint` | Anzahl der Buffer-Treffer in der TOAST-Tabelle dieser Tabelle, falls vorhanden. |
| `tidx_blks_read` | `bigint` | Anzahl der aus den Indizes der TOAST-Tabelle dieser Tabelle gelesenen Plattenblöcke, falls vorhanden. |
| `tidx_blks_hit` | `bigint` | Anzahl der Buffer-Treffer in den Indizes der TOAST-Tabelle dieser Tabelle, falls vorhanden. |

### 27.2.22. pg_statio_all_indexes

Die Sicht `pg_statio_all_indexes` enthält eine Zeile für jeden Index in der aktuellen Datenbank und zeigt Statistiken zur I/O auf diesem bestimmten Index. Die Sichten `pg_statio_user_indexes` und `pg_statio_sys_indexes` enthalten dieselben Informationen, sind aber so gefiltert, dass sie nur Benutzer- beziehungsweise Systemindizes anzeigen.

**Tabelle 27.32. Sicht `pg_statio_all_indexes`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `relid` | `oid` | OID der Tabelle für diesen Index. |
| `indexrelid` | `oid` | OID dieses Index. |
| `schemaname` | `name` | Name des Schemas, in dem sich dieser Index befindet. |
| `relname` | `name` | Name der Tabelle für diesen Index. |
| `indexrelname` | `name` | Name dieses Index. |
| `idx_blks_read` | `bigint` | Anzahl der von diesem Index gelesenen Plattenblöcke. |
| `idx_blks_hit` | `bigint` | Anzahl der Buffer-Treffer in diesem Index. |

### 27.2.23. pg_statio_all_sequences

Die Sicht `pg_statio_all_sequences` enthält eine Zeile für jede Sequenz in der aktuellen Datenbank und zeigt Statistiken zur I/O auf dieser bestimmten Sequenz.

**Tabelle 27.33. Sicht `pg_statio_all_sequences`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `relid` | `oid` | OID einer Sequenz. |
| `schemaname` | `name` | Name des Schemas, in dem sich diese Sequenz befindet. |
| `relname` | `name` | Name dieser Sequenz. |
| `blks_read` | `bigint` | Anzahl der von dieser Sequenz gelesenen Plattenblöcke. |
| `blks_hit` | `bigint` | Anzahl der Buffer-Treffer in dieser Sequenz. |

### 27.2.24. pg_stat_user_functions

Die Sicht `pg_stat_user_functions` enthält eine Zeile für jede nachverfolgte Funktion und zeigt Statistiken über Ausführungen dieser Funktion. Der Parameter `track_functions` steuert genau, welche Funktionen nachverfolgt werden.

**Tabelle 27.34. Sicht `pg_stat_user_functions`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `funcid` | `oid` | OID einer Funktion. |
| `schemaname` | `name` | Name des Schemas, in dem sich diese Funktion befindet. |
| `funcname` | `name` | Name dieser Funktion. |
| `calls` | `bigint` | Anzahl der Aufrufe dieser Funktion. |
| `total_time` | `double precision` | Gesamtzeit in Millisekunden, die in dieser Funktion und allen von ihr aufgerufenen Funktionen verbracht wurde. |
| `self_time` | `double precision` | Gesamtzeit in Millisekunden, die in dieser Funktion selbst verbracht wurde, ohne von ihr aufgerufene Funktionen. |

### 27.2.25. pg_stat_slru

PostgreSQL greift auf bestimmte Informationen auf Platte über SLRU-Caches zu (simple least-recently-used). Die Sicht `pg_stat_slru` enthält eine Zeile für jeden nachverfolgten SLRU-Cache und zeigt Statistiken über den Zugriff auf zwischengespeicherte Seiten.

Für jeden SLRU-Cache, der Teil des Kernservers ist, gibt es einen Konfigurationsparameter, der seine Größe steuert; an den Namen wird das Suffix `_buffers` angehängt.

**Tabelle 27.35. Sicht `pg_stat_slru`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Name des SLRU. |
| `blks_zeroed` | `bigint` | Anzahl der Blöcke, die bei Initialisierungen mit Nullen gefüllt wurden. |
| `blks_hit` | `bigint` | Anzahl der Fälle, in denen Plattenblöcke bereits im SLRU gefunden wurden, sodass kein Lesen nötig war. Dies umfasst nur Treffer im SLRU, nicht im Dateisystem-Cache des Betriebssystems. |
| `blks_read` | `bigint` | Anzahl der für dieses SLRU gelesenen Plattenblöcke. |
| `blks_written` | `bigint` | Anzahl der für dieses SLRU geschriebenen Plattenblöcke. |
| `blks_exists` | `bigint` | Anzahl der Blöcke, deren Existenz für dieses SLRU geprüft wurde. |
| `flushes` | `bigint` | Anzahl der Flushes von Dirty Data für dieses SLRU. |
| `truncates` | `bigint` | Anzahl der Truncates für dieses SLRU. |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem diese Statistiken zuletzt zurückgesetzt wurden. |

### 27.2.26. Statistikfunktionen

Weitere Sichtweisen auf die Statistiken lassen sich durch Abfragen einrichten, die dieselben zugrunde liegenden Statistik-Zugriffsfunktionen verwenden wie die oben gezeigten Standardsichten. Details wie die Funktionsnamen finden Sie in den Definitionen der Standardsichten. In `psql` könnten Sie zum Beispiel `\d+ pg_stat_activity` ausführen. Die Zugriffsfunktionen für datenbankspezifische Statistiken erwarten eine Datenbank-OID als Argument, um festzulegen, über welche Datenbank berichtet werden soll. Die Funktionen für tabellen- und indexspezifische Statistiken erwarten eine Tabellen- oder Index-OID. Die Funktionen für funktionsspezifische Statistiken erwarten eine Funktions-OID. Beachten Sie, dass mit diesen Funktionen nur Tabellen, Indizes und Funktionen in der aktuellen Datenbank sichtbar sind.

Zusätzliche Funktionen im Zusammenhang mit dem kumulativen Statistiksystem sind in Tabelle 27.36 aufgeführt.

**Tabelle 27.36. Zusätzliche Statistikfunktionen**

| Funktion | Beschreibung |
| --- | --- |
| `pg_backend_pid () → integer` | Gibt die Prozess-ID des Serverprozesses zurück, der mit der aktuellen Sitzung verbunden ist. |
| `pg_stat_get_backend_io ( integer ) → setof record` | Gibt I/O-Statistiken über das Backend mit der angegebenen Prozess-ID zurück. Die Ausgabefelder entsprechen genau denen der Sicht `pg_stat_io`. Die Funktion gibt keine I/O-Statistiken für Checkpointer, Background Writer, Startup-Prozess und Autovacuum Launcher zurück, da diese bereits in `pg_stat_io` sichtbar sind und es von jedem nur einen gibt. |
| `pg_stat_get_activity ( integer ) → setof record` | Gibt einen Datensatz mit Informationen über das Backend mit der angegebenen Prozess-ID zurück, oder je einen Datensatz für jedes aktive Backend im System, wenn `NULL` angegeben wird. Die zurückgegebenen Felder sind eine Teilmenge der Felder in der Sicht `pg_stat_activity`. |
| `pg_stat_get_backend_wal ( integer ) → record` | Gibt WAL-Statistiken über das Backend mit der angegebenen Prozess-ID zurück. Die Ausgabefelder entsprechen genau denen der Sicht `pg_stat_wal`. Die Funktion gibt keine WAL-Statistiken für Checkpointer, Background Writer, Startup-Prozess und Autovacuum Launcher zurück. |
| `pg_stat_get_snapshot_timestamp () → timestamp with time zone` | Gibt den Zeitstempel des aktuellen Statistik-Snapshots zurück oder `NULL`, wenn kein Statistik-Snapshot erstellt wurde. Ein Snapshot wird beim ersten Zugriff auf kumulative Statistiken in einer Transaktion erstellt, wenn `stats_fetch_consistency` auf `snapshot` gesetzt ist. |
| `pg_stat_get_xact_blocks_fetched ( oid ) → bigint` | Gibt die Anzahl der Block-Leseanforderungen für eine Tabelle oder einen Index in der aktuellen Transaktion zurück. Dieser Wert minus `pg_stat_get_xact_blocks_hit` ergibt die Anzahl der Kernel-Aufrufe von `read()`; die Anzahl der tatsächlichen physischen Lesevorgänge ist wegen Kernel-Pufferung üblicherweise kleiner. |
| `pg_stat_get_xact_blocks_hit ( oid ) → bigint` | Gibt die Anzahl der Block-Leseanforderungen für eine Tabelle oder einen Index in der aktuellen Transaktion zurück, die im Cache gefunden wurden und daher keine Kernel-Aufrufe von `read()` auslösten. |
| `pg_stat_clear_snapshot () → void` | Verwirft den aktuellen Statistik-Snapshot oder zwischengespeicherte Informationen. |
| `pg_stat_reset () → void` | Setzt alle Statistikzähler für die aktuelle Datenbank auf null zurück. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden, um die Funktion auszuführen. |
| `pg_stat_reset_shared ( [ target text DEFAULT NULL ] ) → void` | Setzt abhängig vom Argument einige clusterweite Statistikzähler auf null zurück. `target` kann `archiver`, `bgwriter`, `checkpointer`, `io`, `recovery_prefetch`, `slru`, `wal` oder `NULL` beziehungsweise nicht angegeben sein. Die genannten Werte setzen jeweils die Zähler der gleichnamigen Sicht zurück; `NULL` oder kein Argument setzt alle Zähler der oben aufgelisteten Sichten zurück. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden. |
| `pg_stat_reset_single_table_counters ( oid ) → void` | Setzt Statistiken für eine einzelne Tabelle oder einen einzelnen Index in der aktuellen Datenbank oder gemeinsam über alle Datenbanken im Cluster auf null zurück. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden. |
| `pg_stat_reset_backend_stats ( integer ) → void` | Setzt Statistiken für ein einzelnes Backend mit der angegebenen Prozess-ID auf null zurück. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden. |
| `pg_stat_reset_single_function_counters ( oid ) → void` | Setzt Statistiken für eine einzelne Funktion in der aktuellen Datenbank auf null zurück. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden. |
| `pg_stat_reset_slru ( [ target text DEFAULT NULL ] ) → void` | Setzt Statistiken für einen einzelnen SLRU-Cache oder für alle SLRUs im Cluster auf null zurück. Wenn `target` `NULL` ist oder nicht angegeben wird, werden alle in `pg_stat_slru` angezeigten Zähler für alle SLRU-Caches zurückgesetzt. Das Argument kann `commit_timestamp`, `multixact_member`, `multixact_offset`, `notify`, `serializable`, `subtransaction` oder `transaction` sein, um nur diesen Eintrag zurückzusetzen. Ist das Argument `other` oder ein nicht erkannter Name, werden die Zähler aller übrigen SLRU-Caches zurückgesetzt, etwa von Erweiterungen definierter Caches. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden. |
| `pg_stat_reset_replication_slot ( text ) → void` | Setzt die Statistiken des durch das Argument angegebenen Replikationsslots zurück. Wenn das Argument `NULL` ist, werden die Statistiken aller Replikationsslots zurückgesetzt. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden. |
| `pg_stat_reset_subscription_stats ( oid ) → void` | Setzt die in `pg_stat_subscription_stats` angezeigten Statistiken für eine einzelne Subskription auf null zurück. Wenn das Argument `NULL` ist, werden die Statistiken aller Subskriptionen zurückgesetzt. Diese Funktion ist standardmäßig auf Superuser beschränkt, anderen Benutzern kann aber `EXECUTE` gewährt werden. |

> **Warnung**
>
> Die Verwendung von `pg_stat_reset()` setzt auch Zähler zurück, die Autovacuum verwendet, um zu bestimmen, wann ein Vacuum oder Analyze ausgelöst werden soll. Das Zurücksetzen dieser Zähler kann dazu führen, dass Autovacuum notwendige Arbeit nicht ausführt, was Probleme wie Table Bloat oder veraltete Tabellenstatistiken verursachen kann. Nach dem Zurücksetzen der Statistiken wird ein datenbankweites `ANALYZE` empfohlen.

`pg_stat_get_activity`, die zugrunde liegende Funktion der Sicht `pg_stat_activity`, gibt eine Menge von Datensätzen mit allen verfügbaren Informationen über jeden Backend-Prozess zurück. Manchmal ist es bequemer, nur eine Teilmenge dieser Informationen zu erhalten. In solchen Fällen kann eine weitere Gruppe backendbezogener Statistik-Zugriffsfunktionen verwendet werden; sie ist in Tabelle 27.37 dargestellt. Diese Zugriffsfunktionen verwenden die Backend-ID-Nummer der Sitzung, eine kleine Ganzzahl (`>= 0`), die sich von der Backend-ID jeder gleichzeitig laufenden Sitzung unterscheidet, obwohl die ID einer Sitzung wiederverwendet werden kann, sobald sie beendet ist. Die Backend-ID wird unter anderem verwendet, um das temporäre Schema der Sitzung zu identifizieren, falls sie eines hat. Die Funktion `pg_stat_get_backend_idset` bietet eine bequeme Möglichkeit, die ID-Nummern aller aktiven Backends für den Aufruf dieser Funktionen aufzulisten. Um zum Beispiel die PIDs und aktuellen Abfragen aller Backends anzuzeigen:

```sql
SELECT pg_stat_get_backend_pid(backendid) AS pid,
       pg_stat_get_backend_activity(backendid) AS query
FROM pg_stat_get_backend_idset() AS backendid;
```

**Tabelle 27.37. Backendbezogene Statistikfunktionen**

| Funktion | Beschreibung |
| --- | --- |
| `pg_stat_get_backend_activity ( integer ) → text` | Gibt den Text der letzten Abfrage dieses Backends zurück. |
| `pg_stat_get_backend_activity_start ( integer ) → timestamp with time zone` | Gibt den Zeitpunkt zurück, zu dem die letzte Abfrage des Backends gestartet wurde. |
| `pg_stat_get_backend_client_addr ( integer ) → inet` | Gibt die IP-Adresse des Clients zurück, der mit diesem Backend verbunden ist. |
| `pg_stat_get_backend_client_port ( integer ) → integer` | Gibt die TCP-Portnummer zurück, die der Client für die Kommunikation verwendet. |
| `pg_stat_get_backend_dbid ( integer ) → oid` | Gibt die OID der Datenbank zurück, mit der dieses Backend verbunden ist. |
| `pg_stat_get_backend_idset () → setof integer` | Gibt die Menge der derzeit aktiven Backend-ID-Nummern zurück. |
| `pg_stat_get_backend_pid ( integer ) → integer` | Gibt die Prozess-ID dieses Backends zurück. |
| `pg_stat_get_backend_start ( integer ) → timestamp with time zone` | Gibt den Zeitpunkt zurück, zu dem dieser Prozess gestartet wurde. |
| `pg_stat_get_backend_subxact ( integer ) → record` | Gibt einen Datensatz mit Informationen über die Subtransaktionen des Backends mit der angegebenen ID zurück. Die zurückgegebenen Felder sind `subxact_count`, die Anzahl der Subtransaktionen im Subtransaction-Cache des Backends, und `subxact_overflow`, das angibt, ob der Subtransaction-Cache des Backends übergelaufen ist. |
| `pg_stat_get_backend_userid ( integer ) → oid` | Gibt die OID des Benutzers zurück, der an diesem Backend angemeldet ist. |
| `pg_stat_get_backend_wait_event ( integer ) → text` | Gibt den Namen des Warteereignisses zurück, wenn dieses Backend derzeit wartet, andernfalls `NULL`. Siehe Tabelle 27.5 bis Tabelle 27.13. |
| `pg_stat_get_backend_wait_event_type ( integer ) → text` | Gibt den Namen des Warteereignistyps zurück, wenn dieses Backend derzeit wartet, andernfalls `NULL`. Details finden Sie in Tabelle 27.4. |
| `pg_stat_get_backend_xact_start ( integer ) → timestamp with time zone` | Gibt den Zeitpunkt zurück, zu dem die aktuelle Transaktion des Backends gestartet wurde. |

## 27.3. Sperren anzeigen

Ein weiteres nützliches Werkzeug zur Überwachung der Datenbankaktivität ist die Systemtabelle `pg_locks`. Sie ermöglicht Datenbankadministratoren, Informationen über ausstehende Sperren im Lock-Manager einzusehen. Diese Fähigkeit kann zum Beispiel verwendet werden, um:

- alle derzeit ausstehenden Sperren anzuzeigen, alle Sperren auf Relationen in einer bestimmten Datenbank, alle Sperren auf einer bestimmten Relation oder alle Sperren, die von einer bestimmten PostgreSQL-Sitzung gehalten werden,
- die Relation in der aktuellen Datenbank mit den meisten nicht gewährten Sperren zu bestimmen, die eine Ursache für Konkurrenz zwischen Datenbankclients sein könnte,
- den Einfluss von Sperrkonflikten auf die gesamte Datenbank-Performance zu bestimmen und zu erkennen, in welchem Umfang die Konkurrenz mit dem gesamten Datenbankverkehr variiert.

Details zur Sicht `pg_locks` finden Sie in [Abschnitt 53.13](53_System_Views.md#5313-pglocks). Weitere Informationen zu Sperren und zur Verwaltung von Nebenläufigkeit mit PostgreSQL finden Sie in [Kapitel 13](13_Nebenläufigkeitskontrolle.md).

## 27.4. Fortschrittsanzeige

PostgreSQL kann den Fortschritt bestimmter Befehle während ihrer Ausführung melden. Derzeit unterstützen nur `ANALYZE`, `CLUSTER`, `CREATE INDEX`, `VACUUM`, `COPY` und `BASE_BACKUP` diese Fortschrittsanzeige. `BASE_BACKUP` ist dabei der Replikationsbefehl, den `pg_basebackup` ausgibt, um eine Basissicherung zu erstellen. Dies kann in Zukunft erweitert werden.

### 27.4.1. Fortschrittsanzeige für ANALYZE

Wenn `ANALYZE` läuft, enthält die Sicht `pg_stat_progress_analyze` eine Zeile für jedes Backend, das diesen Befehl gerade ausführt. Die folgenden Tabellen beschreiben die gemeldeten Informationen und helfen bei ihrer Interpretation.

**Tabelle 27.38. Sicht `pg_stat_progress_analyze`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID des Backends. |
| `datid` | `oid` | OID der Datenbank, mit der dieses Backend verbunden ist. |
| `datname` | `name` | Name der Datenbank, mit der dieses Backend verbunden ist. |
| `relid` | `oid` | OID der Tabelle, die analysiert wird. |
| `phase` | `text` | Aktuelle Verarbeitungsphase. Siehe Tabelle 27.39. |
| `sample_blks_total` | `bigint` | Gesamtanzahl der Heap-Blöcke, aus denen Stichproben entnommen werden. |
| `sample_blks_scanned` | `bigint` | Anzahl der gescannten Heap-Blöcke. |
| `ext_stats_total` | `bigint` | Anzahl der erweiterten Statistiken. |
| `ext_stats_computed` | `bigint` | Anzahl der berechneten erweiterten Statistiken. Dieser Zähler schreitet nur in der Phase `computing extended statistics` voran. |
| `child_tables_total` | `bigint` | Anzahl der Kindtabellen. |
| `child_tables_done` | `bigint` | Anzahl der gescannten Kindtabellen. Dieser Zähler schreitet nur in der Phase `acquiring inherited sample rows` voran. |
| `current_child_table_relid` | `oid` | OID der Kindtabelle, die gerade gescannt wird. Dieses Feld ist nur in der Phase `acquiring inherited sample rows` gültig. |
| `delay_time` | `double precision` | Gesamtzeit in Millisekunden, die wegen kostenbasierter Verzögerung geschlafen wurde (siehe Abschnitt 19.10.2), wenn `track_cost_delay_timing` aktiviert ist; andernfalls 0. |

**Tabelle 27.39. ANALYZE-Phasen**

| Phase | Beschreibung |
| --- | --- |
| `initializing` | Der Befehl bereitet den Scan des Heaps vor. Diese Phase ist voraussichtlich sehr kurz. |
| `acquiring sample rows` | Der Befehl scannt derzeit die durch `relid` angegebene Tabelle, um Stichprobenzeilen zu erhalten. |
| `acquiring inherited sample rows` | Der Befehl scannt derzeit Kindtabellen, um Stichprobenzeilen zu erhalten. Die Spalten `child_tables_total`, `child_tables_done` und `current_child_table_relid` enthalten die Fortschrittsinformationen für diese Phase. |
| `computing statistics` | Der Befehl berechnet Statistiken aus den Stichprobenzeilen, die während des Tabellenscans gewonnen wurden. |
| `computing extended statistics` | Der Befehl berechnet erweiterte Statistiken aus den Stichprobenzeilen, die während des Tabellenscans gewonnen wurden. |
| `finalizing analyze` | Der Befehl aktualisiert `pg_class`. Wenn diese Phase abgeschlossen ist, endet `ANALYZE`. |

> **Hinweis**
>
> Wenn `ANALYZE` auf einer partitionierten Tabelle ohne das Schlüsselwort `ONLY` ausgeführt wird, werden alle Partitionen ebenfalls rekursiv analysiert. In diesem Fall wird der Fortschritt von `ANALYZE` zuerst für die übergeordnete Tabelle gemeldet, wobei deren Vererbungsstatistiken gesammelt werden, und anschließend für jede Partition.

### 27.4.2. Fortschrittsanzeige für CLUSTER

Wenn `CLUSTER` oder `VACUUM FULL` läuft, enthält die Sicht `pg_stat_progress_cluster` eine Zeile für jedes Backend, das gerade einen der beiden Befehle ausführt. Die folgenden Tabellen beschreiben die gemeldeten Informationen und helfen bei ihrer Interpretation.

**Tabelle 27.40. Sicht `pg_stat_progress_cluster`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID des Backends. |
| `datid` | `oid` | OID der Datenbank, mit der dieses Backend verbunden ist. |
| `datname` | `name` | Name der Datenbank, mit der dieses Backend verbunden ist. |
| `relid` | `oid` | OID der Tabelle, die geclustert wird. |
| `command` | `text` | Der ausgeführte Befehl: entweder `CLUSTER` oder `VACUUM FULL`. |
| `phase` | `text` | Aktuelle Verarbeitungsphase. Siehe Tabelle 27.41. |
| `cluster_index_relid` | `oid` | Wenn die Tabelle mit einem Index gescannt wird, ist dies die OID des verwendeten Index; andernfalls 0. |
| `heap_tuples_scanned` | `bigint` | Anzahl der gescannten Heap-Tupel. Dieser Zähler schreitet nur in den Phasen `seq scanning heap`, `index scanning heap` oder `writing new heap` voran. |
| `heap_tuples_written` | `bigint` | Anzahl der geschriebenen Heap-Tupel. Dieser Zähler schreitet nur in den Phasen `seq scanning heap`, `index scanning heap` oder `writing new heap` voran. |
| `heap_blks_total` | `bigint` | Gesamtanzahl der Heap-Blöcke in der Tabelle. Dieser Wert wird zu Beginn von `seq scanning heap` gemeldet. |
| `heap_blks_scanned` | `bigint` | Anzahl der gescannten Heap-Blöcke. Dieser Zähler schreitet nur in der Phase `seq scanning heap` voran. |
| `index_rebuild_count` | `bigint` | Anzahl der neu aufgebauten Indizes. Dieser Zähler schreitet nur in der Phase `rebuilding index` voran. |

**Tabelle 27.41. CLUSTER- und VACUUM-FULL-Phasen**

| Phase | Beschreibung |
| --- | --- |
| `initializing` | Der Befehl bereitet den Scan des Heaps vor. Diese Phase ist voraussichtlich sehr kurz. |
| `seq scanning heap` | Der Befehl scannt die Tabelle derzeit mit einem sequenziellen Scan. |
| `index scanning heap` | `CLUSTER` scannt die Tabelle derzeit mit einem Indexscan. |
| `sorting tuples` | `CLUSTER` sortiert derzeit Tupel. |
| `writing new heap` | `CLUSTER` schreibt derzeit den neuen Heap. |
| `swapping relation files` | Der Befehl tauscht derzeit neu erstellte Dateien ein. |
| `rebuilding index` | Der Befehl baut derzeit einen Index neu auf. |
| `performing final cleanup` | Der Befehl führt die abschließende Bereinigung durch. Wenn diese Phase abgeschlossen ist, endet `CLUSTER` oder `VACUUM FULL`. |

### 27.4.3. Fortschrittsanzeige für COPY

Wenn `COPY` läuft, enthält die Sicht `pg_stat_progress_copy` eine Zeile für jedes Backend, das gerade einen `COPY`-Befehl ausführt. Die folgende Tabelle beschreibt die gemeldeten Informationen und hilft bei ihrer Interpretation.

**Tabelle 27.42. Sicht `pg_stat_progress_copy`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID des Backends. |
| `datid` | `oid` | OID der Datenbank, mit der dieses Backend verbunden ist. |
| `datname` | `name` | Name der Datenbank, mit der dieses Backend verbunden ist. |
| `relid` | `oid` | OID der Tabelle, auf der der `COPY`-Befehl ausgeführt wird. Beim Kopieren aus einer `SELECT`-Abfrage wird dies auf 0 gesetzt. |
| `command` | `text` | Der ausgeführte Befehl: `COPY FROM` oder `COPY TO`. |
| `type` | `text` | Der I/O-Typ, aus dem Daten gelesen oder in den Daten geschrieben werden: `FILE`, `PROGRAM`, `PIPE` (für `COPY FROM STDIN` und `COPY TO STDOUT`) oder `CALLBACK` (zum Beispiel während der anfänglichen Tabellensynchronisierung bei logischer Replikation). |
| `bytes_processed` | `bigint` | Anzahl der vom `COPY`-Befehl bereits verarbeiteten Bytes. |
| `bytes_total` | `bigint` | Größe der Quelldatei für einen `COPY FROM`-Befehl in Byte. Ist sie nicht verfügbar, wird dies auf 0 gesetzt. |
| `tuples_processed` | `bigint` | Anzahl der vom `COPY`-Befehl bereits verarbeiteten Tupel. |
| `tuples_excluded` | `bigint` | Anzahl der Tupel, die nicht verarbeitet wurden, weil sie durch die `WHERE`-Klausel des `COPY`-Befehls ausgeschlossen wurden. |
| `tuples_skipped` | `bigint` | Anzahl der Tupel, die übersprungen wurden, weil sie fehlerhafte Daten enthalten. Dieser Zähler schreitet nur voran, wenn für die Option `ON_ERROR` ein anderer Wert als `stop` angegeben ist. |

### 27.4.4. Fortschrittsanzeige für CREATE INDEX

Wenn `CREATE INDEX` oder `REINDEX` läuft, enthält die Sicht `pg_stat_progress_create_index` eine Zeile für jedes Backend, das gerade Indizes erstellt. Die folgenden Tabellen beschreiben die gemeldeten Informationen und helfen bei ihrer Interpretation.

**Tabelle 27.43. Sicht `pg_stat_progress_create_index`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID des Backends, das Indizes erstellt. |
| `datid` | `oid` | OID der Datenbank, mit der dieses Backend verbunden ist. |
| `datname` | `name` | Name der Datenbank, mit der dieses Backend verbunden ist. |
| `relid` | `oid` | OID der Tabelle, auf der der Index erstellt wird. |
| `index_relid` | `oid` | OID des Index, der erstellt oder reindiziert wird. Während eines nicht nebenläufigen `CREATE INDEX` ist dies 0. |
| `command` | `text` | Konkreter Befehlstyp: `CREATE INDEX`, `CREATE INDEX CONCURRENTLY`, `REINDEX` oder `REINDEX CONCURRENTLY`. |
| `phase` | `text` | Aktuelle Verarbeitungsphase der Indexerstellung. Siehe Tabelle 27.44. |
| `lockers_total` | `bigint` | Gesamtanzahl der Locker, auf die gegebenenfalls gewartet werden muss. |
| `lockers_done` | `bigint` | Anzahl der Locker, auf die bereits gewartet wurde. |
| `current_locker_pid` | `bigint` | Prozess-ID des Lockers, auf den gerade gewartet wird. |
| `blocks_total` | `bigint` | Gesamtanzahl der Blöcke, die in der aktuellen Phase verarbeitet werden sollen. |
| `blocks_done` | `bigint` | Anzahl der Blöcke, die in der aktuellen Phase bereits verarbeitet wurden. |
| `tuples_total` | `bigint` | Gesamtanzahl der Tupel, die in der aktuellen Phase verarbeitet werden sollen. |
| `tuples_done` | `bigint` | Anzahl der Tupel, die in der aktuellen Phase bereits verarbeitet wurden. |
| `partitions_total` | `bigint` | Gesamtanzahl der Partitionen, auf denen der Index erstellt oder angehängt werden soll, einschließlich direkter und indirekter Partitionen. Während eines `REINDEX` oder wenn der Index nicht partitioniert ist, ist dies 0. |
| `partitions_done` | `bigint` | Anzahl der Partitionen, auf denen der Index bereits erstellt oder angehängt wurde, einschließlich direkter und indirekter Partitionen. Während eines `REINDEX` oder wenn der Index nicht partitioniert ist, ist dies 0. |

**Tabelle 27.44. CREATE-INDEX-Phasen**

| Phase | Beschreibung |
| --- | --- |
| `initializing` | `CREATE INDEX` oder `REINDEX` bereitet die Indexerstellung vor. Diese Phase ist voraussichtlich sehr kurz. |
| `waiting for writers before build` | `CREATE INDEX CONCURRENTLY` oder `REINDEX CONCURRENTLY` wartet darauf, dass Transaktionen mit Schreibsperren, die die Tabelle potenziell sehen können, beendet werden. Diese Phase wird außerhalb des nebenläufigen Modus übersprungen. Die Spalten `lockers_total`, `lockers_done` und `current_locker_pid` enthalten die Fortschrittsinformationen. |
| `building index` | Der Index wird vom zugriffsmethodenspezifischen Code erstellt. In dieser Phase tragen Zugriffsmethoden, die Fortschrittsmeldungen unterstützen, eigene Fortschrittsdaten ein; die Unterphase wird in dieser Spalte angegeben. Typischerweise enthalten `blocks_total` und `blocks_done` Fortschrittsdaten, möglicherweise auch `tuples_total` und `tuples_done`. |
| `waiting for writers before validation` | `CREATE INDEX CONCURRENTLY` oder `REINDEX CONCURRENTLY` wartet darauf, dass Transaktionen mit Schreibsperren, die potenziell in die Tabelle schreiben können, beendet werden. Diese Phase wird außerhalb des nebenläufigen Modus übersprungen. Die Spalten `lockers_total`, `lockers_done` und `current_locker_pid` enthalten die Fortschrittsinformationen. |
| `index validation: scanning index` | `CREATE INDEX CONCURRENTLY` scannt den Index und sucht nach Tupeln, die validiert werden müssen. Diese Phase wird außerhalb des nebenläufigen Modus übersprungen. Die Spalten `blocks_total` (auf die Gesamtgröße des Index gesetzt) und `blocks_done` enthalten die Fortschrittsinformationen. |
| `index validation: sorting tuples` | `CREATE INDEX CONCURRENTLY` sortiert die Ausgabe der Indexscan-Phase. |
| `index validation: scanning table` | `CREATE INDEX CONCURRENTLY` scannt die Tabelle, um die in den vorherigen beiden Phasen gesammelten Indextupel zu validieren. Diese Phase wird außerhalb des nebenläufigen Modus übersprungen. Die Spalten `blocks_total` (auf die Gesamtgröße der Tabelle gesetzt) und `blocks_done` enthalten die Fortschrittsinformationen. |
| `waiting for old snapshots` | `CREATE INDEX CONCURRENTLY` oder `REINDEX CONCURRENTLY` wartet darauf, dass Transaktionen, die die Tabelle potenziell sehen können, ihre Snapshots freigeben. Diese Phase wird außerhalb des nebenläufigen Modus übersprungen. Die Spalten `lockers_total`, `lockers_done` und `current_locker_pid` enthalten die Fortschrittsinformationen. |
| `waiting for readers before marking dead` | `REINDEX CONCURRENTLY` wartet darauf, dass Transaktionen mit Lesesperren auf der Tabelle beendet werden, bevor der alte Index als tot markiert wird. Diese Phase wird außerhalb des nebenläufigen Modus übersprungen. Die Spalten `lockers_total`, `lockers_done` und `current_locker_pid` enthalten die Fortschrittsinformationen. |
| `waiting for readers before dropping` | `REINDEX CONCURRENTLY` wartet darauf, dass Transaktionen mit Lesesperren auf der Tabelle beendet werden, bevor der alte Index gelöscht wird. Diese Phase wird außerhalb des nebenläufigen Modus übersprungen. Die Spalten `lockers_total`, `lockers_done` und `current_locker_pid` enthalten die Fortschrittsinformationen. |

### 27.4.5. Fortschrittsanzeige für VACUUM

Wenn `VACUUM` läuft, enthält die Sicht `pg_stat_progress_vacuum` eine Zeile für jedes Backend, einschließlich Autovacuum-Worker-Prozessen, das gerade vacuumiert. Die folgenden Tabellen beschreiben die gemeldeten Informationen und helfen bei ihrer Interpretation. Der Fortschritt von `VACUUM FULL`-Befehlen wird über `pg_stat_progress_cluster` gemeldet, weil sowohl `VACUUM FULL` als auch `CLUSTER` die Tabelle neu schreiben, während normales `VACUUM` sie nur an Ort und Stelle ändert. Siehe [Abschnitt 27.4.2](#2742-fortschrittsanzeige-für-cluster).

**Tabelle 27.45. Sicht `pg_stat_progress_vacuum`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID des Backends. |
| `datid` | `oid` | OID der Datenbank, mit der dieses Backend verbunden ist. |
| `datname` | `name` | Name der Datenbank, mit der dieses Backend verbunden ist. |
| `relid` | `oid` | OID der Tabelle, die vacuumiert wird. |
| `phase` | `text` | Aktuelle Verarbeitungsphase von Vacuum. Siehe Tabelle 27.46. |
| `heap_blks_total` | `bigint` | Gesamtanzahl der Heap-Blöcke in der Tabelle. Dieser Wert wird zu Beginn des Scans gemeldet; später hinzugefügte Blöcke werden von diesem `VACUUM` nicht besucht und müssen es auch nicht. |
| `heap_blks_scanned` | `bigint` | Anzahl der gescannten Heap-Blöcke. Da die Visibility Map zur Optimierung von Scans verwendet wird, werden manche Blöcke ohne Inspektion übersprungen; übersprungene Blöcke sind in dieser Summe enthalten, sodass dieser Wert am Ende von Vacuum `heap_blks_total` entspricht. Dieser Zähler schreitet nur in der Phase `scanning heap` voran. |
| `heap_blks_vacuumed` | `bigint` | Anzahl der vacuumierten Heap-Blöcke. Sofern die Tabelle Indizes hat, schreitet dieser Zähler nur in der Phase `vacuuming heap` voran. Blöcke ohne tote Tupel werden übersprungen, daher kann der Zähler manchmal in größeren Sprüngen voranschreiten. |
| `index_vacuum_count` | `bigint` | Anzahl abgeschlossener Index-Vacuum-Zyklen. |
| `max_dead_tuple_bytes` | `bigint` | Menge an Daten toter Tupel, die gespeichert werden kann, bevor ein Index-Vacuum-Zyklus nötig ist, basierend auf `maintenance_work_mem`. |
| `dead_tuple_bytes` | `bigint` | Menge an Daten toter Tupel, die seit dem letzten Index-Vacuum-Zyklus gesammelt wurde. |
| `num_dead_item_ids` | `bigint` | Anzahl der Bezeichner toter Elemente, die seit dem letzten Index-Vacuum-Zyklus gesammelt wurden. |
| `indexes_total` | `bigint` | Gesamtanzahl der Indizes, die vacuumiert oder bereinigt werden. Dieser Wert wird zu Beginn der Phase `vacuuming indexes` oder `cleaning up indexes` gemeldet. |
| `indexes_processed` | `bigint` | Anzahl der verarbeiteten Indizes. Dieser Zähler schreitet nur in den Phasen `vacuuming indexes` oder `cleaning up indexes` voran. |
| `delay_time` | `double precision` | Gesamtzeit in Millisekunden, die wegen kostenbasierter Verzögerung geschlafen wurde (siehe Abschnitt 19.10.2), wenn `track_cost_delay_timing` aktiviert ist; andernfalls 0. Enthalten ist die Schlafzeit zugehöriger paralleler Worker. Da parallele Worker ihre Schlafzeit jedoch höchstens einmal pro Sekunde melden, kann der gemeldete Wert leicht veraltet sein. |

**Tabelle 27.46. VACUUM-Phasen**

| Phase | Beschreibung |
| --- | --- |
| `initializing` | `VACUUM` bereitet den Scan des Heaps vor. Diese Phase ist voraussichtlich sehr kurz. |
| `scanning heap` | `VACUUM` scannt derzeit den Heap. Es entfernt und defragmentiert bei Bedarf jede Seite und kann Freeze-Aktivität durchführen. Die Spalte `heap_blks_scanned` kann zur Fortschrittsüberwachung des Scans verwendet werden. |
| `vacuuming indexes` | `VACUUM` vacuumiert derzeit die Indizes. Wenn eine Tabelle Indizes hat, geschieht dies mindestens einmal pro Vacuum, nachdem der Heap vollständig gescannt wurde. Es kann mehrfach pro Vacuum geschehen, wenn `maintenance_work_mem` oder bei Autovacuum `autovacuum_work_mem`, sofern gesetzt, nicht ausreicht, um die Anzahl der gefundenen toten Tupel zu speichern. |
| `vacuuming heap` | `VACUUM` vacuumiert derzeit den Heap. Das Vacuumieren des Heaps unterscheidet sich vom Scannen des Heaps und erfolgt nach jeder Instanz von `vacuuming indexes`. Wenn `heap_blks_scanned` kleiner als `heap_blks_total` ist, kehrt das System nach Abschluss dieser Phase zum Scannen des Heaps zurück; andernfalls beginnt es anschließend mit der Indexbereinigung. |
| `cleaning up indexes` | `VACUUM` bereinigt derzeit Indizes. Dies geschieht, nachdem der Heap vollständig gescannt wurde und alle Vacuum-Arbeiten an Indizes und Heap abgeschlossen sind. |
| `truncating heap` | `VACUUM` kürzt derzeit den Heap, um leere Seiten am Ende der Relation an das Betriebssystem zurückzugeben. Dies geschieht nach der Indexbereinigung. |
| `performing final cleanup` | `VACUUM` führt die abschließende Bereinigung durch. In dieser Phase vacuumiert `VACUUM` die Free Space Map, aktualisiert Statistiken in `pg_class` und meldet Statistiken an das kumulative Statistiksystem. Wenn diese Phase abgeschlossen ist, endet `VACUUM`. |

### 27.4.6. Fortschrittsanzeige für Basissicherungen

Wenn eine Anwendung wie `pg_basebackup` eine Basissicherung erstellt, enthält die Sicht `pg_stat_progress_basebackup` eine Zeile für jeden WAL-Sender-Prozess, der gerade den Replikationsbefehl `BASE_BACKUP` ausführt und die Sicherung streamt. Die folgenden Tabellen beschreiben die gemeldeten Informationen und helfen bei ihrer Interpretation.

**Tabelle 27.47. Sicht `pg_stat_progress_basebackup`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `integer` | Prozess-ID eines WAL-Sender-Prozesses. |
| `phase` | `text` | Aktuelle Verarbeitungsphase. Siehe Tabelle 27.48. |
| `backup_total` | `bigint` | Gesamtmenge der Daten, die gestreamt werden. Dieser Wert wird geschätzt und zu Beginn der Phase `streaming database files` gemeldet. Er ist nur eine Näherung, da sich die Datenbank während dieser Phase ändern kann und später WAL-Log in die Sicherung einbezogen werden kann. Sobald die gestreamte Datenmenge die geschätzte Gesamtgröße überschreitet, entspricht dieser Wert immer `backup_streamed`. Wenn die Schätzung in `pg_basebackup` deaktiviert ist, etwa mit `--no-estimate-size`, ist dieser Wert `NULL`. |
| `backup_streamed` | `bigint` | Menge der gestreamten Daten. Dieser Zähler schreitet nur in den Phasen `streaming database files` oder `transferring wal files` voran. |
| `tablespaces_total` | `bigint` | Gesamtanzahl der Tablespaces, die gestreamt werden. |
| `tablespaces_streamed` | `bigint` | Anzahl der gestreamten Tablespaces. Dieser Zähler schreitet nur in der Phase `streaming database files` voran. |

**Tabelle 27.48. Basissicherungsphasen**

| Phase | Beschreibung |
| --- | --- |
| `initializing` | Der WAL-Sender-Prozess bereitet den Beginn der Sicherung vor. Diese Phase ist voraussichtlich sehr kurz. |
| `waiting for checkpoint to finish` | Der WAL-Sender-Prozess führt derzeit `pg_backup_start` aus, um die Basissicherung vorzubereiten, und wartet darauf, dass der Start-of-Backup-Checkpoint abgeschlossen wird. |
| `estimating backup size` | Der WAL-Sender-Prozess schätzt derzeit die Gesamtmenge der Datenbankdateien, die als Basissicherung gestreamt werden. |
| `streaming database files` | Der WAL-Sender-Prozess streamt derzeit Datenbankdateien als Basissicherung. |
| `waiting for wal archiving to finish` | Der WAL-Sender-Prozess führt derzeit `pg_backup_stop` aus, um die Sicherung abzuschließen, und wartet darauf, dass alle für die Basissicherung erforderlichen WAL-Dateien erfolgreich archiviert werden. Wenn in `pg_basebackup` entweder `--wal-method=none` oder `--wal-method=stream` angegeben ist, endet die Sicherung nach Abschluss dieser Phase. |
| `transferring wal files` | Der WAL-Sender-Prozess überträgt derzeit alle während der Sicherung erzeugten WAL-Logs. Diese Phase tritt nach `waiting for wal archiving to finish` auf, wenn in `pg_basebackup` `--wal-method=fetch` angegeben ist. Die Sicherung endet, wenn diese Phase abgeschlossen ist. |

## 27.5. Dynamisches Tracing

PostgreSQL stellt Einrichtungen bereit, um dynamisches Tracing des Datenbankservers zu unterstützen. Dadurch kann ein externes Werkzeug an bestimmten Stellen im Code aufgerufen werden und so die Ausführung verfolgen.

Eine Reihe von Probes oder Trace Points ist bereits in den Quellcode eingefügt. Diese Probes sind für Datenbankentwickler und Administratoren gedacht. Standardmäßig werden die Probes nicht in PostgreSQL einkompiliert; der Benutzer muss dem Konfigurationsskript ausdrücklich mitteilen, dass die Probes verfügbar gemacht werden sollen.

Derzeit wird das Werkzeug DTrace unterstützt, das zum Zeitpunkt dieser Dokumentation auf Solaris, macOS, FreeBSD, NetBSD und Oracle Linux verfügbar ist. Das SystemTap-Projekt für Linux stellt ein DTrace-Äquivalent bereit und kann ebenfalls verwendet werden. Die Unterstützung anderer dynamischer Tracing-Werkzeuge ist theoretisch möglich, indem die Definitionen der Makros in `src/include/utils/probes.h` geändert werden.

### 27.5.1. Für dynamisches Tracing kompilieren

Standardmäßig sind Probes nicht verfügbar; Sie müssen dem Konfigurationsskript also ausdrücklich mitteilen, dass die Probes in PostgreSQL verfügbar gemacht werden sollen. Um DTrace-Unterstützung einzubeziehen, geben Sie `--enable-dtrace` an `configure` weiter. Weitere Informationen finden Sie in [Abschnitt 17.3.3.6](17_Installation_aus_dem_Quellcode.md#17336-entwickleroptionen).

### 27.5.2. Eingebaute Probes

Im Quellcode werden mehrere Standard-Probes bereitgestellt, wie in Tabelle 27.49 gezeigt; Tabelle 27.50 zeigt die in den Probes verwendeten Typen. Weitere Probes können natürlich hinzugefügt werden, um die Beobachtbarkeit von PostgreSQL zu verbessern.

- <https://en.wikipedia.org/wiki/DTrace>
- <https://sourceware.org/systemtap/>

**Tabelle 27.49. Eingebaute DTrace-Probes**

| Name | Parameter | Beschreibung |
| --- | --- | --- |
| `transaction-start` | `(LocalTransactionId)` | Wird beim Start einer neuen Transaktion ausgelöst. `arg0` ist die Transaktions-ID. |
| `transaction-commit` | `(LocalTransactionId)` | Wird ausgelöst, wenn eine Transaktion erfolgreich abgeschlossen wird. `arg0` ist die Transaktions-ID. |
| `transaction-abort` | `(LocalTransactionId)` | Wird ausgelöst, wenn eine Transaktion erfolglos abgeschlossen wird. `arg0` ist die Transaktions-ID. |
| `query-start` | `(const char *)` | Wird ausgelöst, wenn die Verarbeitung einer Abfrage beginnt. `arg0` ist die Abfragezeichenkette. |
| `query-done` | `(const char *)` | Wird ausgelöst, wenn die Verarbeitung einer Abfrage abgeschlossen ist. `arg0` ist die Abfragezeichenkette. |
| `query-parse-start` | `(const char *)` | Wird ausgelöst, wenn das Parsen einer Abfrage beginnt. `arg0` ist die Abfragezeichenkette. |
| `query-parse-done` | `(const char *)` | Wird ausgelöst, wenn das Parsen einer Abfrage abgeschlossen ist. `arg0` ist die Abfragezeichenkette. |
| `query-rewrite-start` | `(const char *)` | Wird ausgelöst, wenn das Umschreiben einer Abfrage beginnt. `arg0` ist die Abfragezeichenkette. |
| `query-rewrite-done` | `(const char *)` | Wird ausgelöst, wenn das Umschreiben einer Abfrage abgeschlossen ist. `arg0` ist die Abfragezeichenkette. |
| `query-plan-start` | `()` | Wird ausgelöst, wenn die Planung einer Abfrage beginnt. |
| `query-plan-done` | `()` | Wird ausgelöst, wenn die Planung einer Abfrage abgeschlossen ist. |
| `query-execute-start` | `()` | Wird ausgelöst, wenn die Ausführung einer Abfrage beginnt. |
| `query-execute-done` | `()` | Wird ausgelöst, wenn die Ausführung einer Abfrage abgeschlossen ist. |
| `statement-status` | `(const char *)` | Wird jedes Mal ausgelöst, wenn der Serverprozess seinen Status in `pg_stat_activity.status` aktualisiert. `arg0` ist die neue Statuszeichenkette. |
| `checkpoint-start` | `(int)` | Wird ausgelöst, wenn ein Checkpoint beginnt. `arg0` enthält bitweise Flags, die verschiedene Checkpoint-Typen wie Shutdown, Immediate oder Force unterscheiden. |
| `checkpoint-done` | `(int, int, int, int, int)` | Wird ausgelöst, wenn ein Checkpoint abgeschlossen ist. `arg0` ist die Anzahl geschriebener Buffer, `arg1` die Gesamtanzahl der Buffer, und `arg2`, `arg3` sowie `arg4` enthalten die Anzahl hinzugefügter, entfernter beziehungsweise recycelter WAL-Dateien. |
| `clog-checkpoint-start` | `(bool)` | Wird ausgelöst, wenn der CLOG-Teil eines Checkpoints beginnt. `arg0` ist wahr für einen normalen Checkpoint und falsch für einen Shutdown-Checkpoint. |
| `clog-checkpoint-done` | `(bool)` | Wird ausgelöst, wenn der CLOG-Teil eines Checkpoints abgeschlossen ist. `arg0` hat dieselbe Bedeutung wie bei `clog-checkpoint-start`. |
| `subtrans-checkpoint-start` | `(bool)` | Wird ausgelöst, wenn der SUBTRANS-Teil eines Checkpoints beginnt. `arg0` ist wahr für einen normalen Checkpoint und falsch für einen Shutdown-Checkpoint. |
| `subtrans-checkpoint-done` | `(bool)` | Wird ausgelöst, wenn der SUBTRANS-Teil eines Checkpoints abgeschlossen ist. `arg0` hat dieselbe Bedeutung wie bei `subtrans-checkpoint-start`. |
| `multixact-checkpoint-start` | `(bool)` | Wird ausgelöst, wenn der MultiXact-Teil eines Checkpoints beginnt. `arg0` ist wahr für einen normalen Checkpoint und falsch für einen Shutdown-Checkpoint. |
| `multixact-checkpoint-done` | `(bool)` | Wird ausgelöst, wenn der MultiXact-Teil eines Checkpoints abgeschlossen ist. `arg0` hat dieselbe Bedeutung wie bei `multixact-checkpoint-start`. |
| `buffer-checkpoint-start` | `(int)` | Wird ausgelöst, wenn der Buffer-Writing-Teil eines Checkpoints beginnt. `arg0` enthält bitweise Flags zur Unterscheidung von Checkpoint-Typen wie Shutdown, Immediate oder Force. |
| `buffer-sync-start` | `(int, int)` | Wird ausgelöst, wenn während eines Checkpoints mit dem Schreiben Dirty Buffers begonnen wird, nachdem bestimmt wurde, welche Buffer geschrieben werden müssen. `arg0` ist die Gesamtanzahl der Buffer, `arg1` die Anzahl der derzeit Dirty Buffers, die geschrieben werden müssen. |
| `buffer-sync-written` | `(int)` | Wird nach jedem während eines Checkpoints geschriebenen Buffer ausgelöst. `arg0` ist die ID-Nummer des Buffers. |
| `buffer-sync-done` | `(int, int, int)` | Wird ausgelöst, wenn alle Dirty Buffers geschrieben wurden. `arg0` ist die Gesamtanzahl der Buffer, `arg1` die vom Checkpoint-Prozess tatsächlich geschriebene Anzahl, und `arg2` die erwartete Anzahl; Unterschiede spiegeln wider, dass andere Prozesse während des Checkpoints Buffer geflusht haben. |
| `buffer-checkpoint-sync-start` | `()` | Wird ausgelöst, nachdem Dirty Buffers in den Kernel geschrieben wurden und bevor `fsync`-Anforderungen ausgegeben werden. |
| `buffer-checkpoint-done` | `()` | Wird ausgelöst, wenn das Synchronisieren der Buffer auf Platte abgeschlossen ist. |
| `twophase-checkpoint-start` | `()` | Wird ausgelöst, wenn der Two-Phase-Teil eines Checkpoints beginnt. |
| `twophase-checkpoint-done` | `()` | Wird ausgelöst, wenn der Two-Phase-Teil eines Checkpoints abgeschlossen ist. |
| `buffer-extend-start` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int, unsigned int)` | Wird ausgelöst, wenn eine Relationserweiterung beginnt. Die Argumente enthalten Fork und Relation-Identität; `arg4` ist die ID des Backends für temporäre lokale Buffer oder `INVALID_PROC_NUMBER` für Shared Buffer, `arg5` die gewünschte Erweiterung in Blöcken. |
| `buffer-extend-done` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int, unsigned int, BlockNumber)` | Wird ausgelöst, wenn eine Relationserweiterung abgeschlossen ist. Die Argumente entsprechen `buffer-extend-start`; `arg5` ist die tatsächlich erfolgte Erweiterung in Blöcken und `arg6` die Blocknummer des ersten neuen Blocks. |
| `buffer-read-start` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int)` | Wird ausgelöst, wenn ein Buffer-Lesen beginnt. Die Argumente enthalten Fork, Blocknummer, Tablespace-, Datenbank- und Relations-OIDs sowie die Backend-ID für lokale temporäre Relationen oder `INVALID_PROC_NUMBER` für Shared Buffer. |
| `buffer-read-done` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int, bool)` | Wird ausgelöst, wenn ein Buffer-Lesen abgeschlossen ist. Die ersten Argumente entsprechen `buffer-read-start`; `arg6` ist wahr, wenn der Buffer im Pool gefunden wurde, andernfalls falsch. |
| `buffer-flush-start` | `(ForkNumber, BlockNumber, Oid, Oid, Oid)` | Wird ausgelöst, bevor eine Schreibanforderung für einen Shared Buffer ausgegeben wird. Die Argumente enthalten Fork, Blocknummer sowie Tablespace-, Datenbank- und Relations-OIDs. |
| `buffer-flush-done` | `(ForkNumber, BlockNumber, Oid, Oid, Oid)` | Wird ausgelöst, wenn eine Schreibanforderung abgeschlossen ist. Dies spiegelt nur die Zeit wider, um die Daten an den Kernel zu übergeben; üblicherweise sind sie zu diesem Zeitpunkt noch nicht tatsächlich auf Platte geschrieben. |
| `wal-buffer-write-dirty-start` | `()` | Wird ausgelöst, wenn ein Serverprozess beginnt, einen Dirty WAL Buffer zu schreiben, weil kein WAL-Buffer-Platz mehr verfügbar ist. Wenn dies häufig geschieht, ist `wal_buffers` vermutlich zu klein. |
| `wal-buffer-write-dirty-done` | `()` | Wird ausgelöst, wenn das Schreiben eines Dirty WAL Buffers abgeschlossen ist. |
| `wal-insert` | `(unsigned char, unsigned char)` | Wird ausgelöst, wenn ein WAL-Datensatz eingefügt wird. `arg0` ist der Resource Manager (`rmid`) für den Datensatz, `arg1` enthält die Info-Flags. |
| `wal-switch` | `()` | Wird ausgelöst, wenn ein WAL-Segmentwechsel angefordert wird. |
| `smgr-md-read-start` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int)` | Wird ausgelöst, wenn begonnen wird, einen Block aus einer Relation zu lesen. Die Argumente identifizieren Fork, Block, Tablespace, Datenbank, Relation und Backend. |
| `smgr-md-read-done` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int, int, int)` | Wird ausgelöst, wenn ein Blocklesen abgeschlossen ist. `arg6` ist die tatsächlich gelesene Bytezahl, `arg7` die angeforderte Bytezahl; ein Unterschied weist auf ein kurzes Lesen hin. |
| `smgr-md-write-start` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int)` | Wird ausgelöst, wenn begonnen wird, einen Block in eine Relation zu schreiben. Die Argumente identifizieren Fork, Block, Tablespace, Datenbank, Relation und Backend. |
| `smgr-md-write-done` | `(ForkNumber, BlockNumber, Oid, Oid, Oid, int, int, int)` | Wird ausgelöst, wenn ein Blockschreiben abgeschlossen ist. `arg6` ist die tatsächlich geschriebene Bytezahl, `arg7` die angeforderte Bytezahl; ein Unterschied weist auf ein kurzes Schreiben hin. |
| `sort-start` | `(int, bool, int, int, bool, int)` | Wird ausgelöst, wenn eine Sortieroperation beginnt. `arg0` gibt Heap-, Index- oder Datum-Sortierung an, `arg1` ist wahr bei Unique-Erzwingung, `arg2` ist die Anzahl der Schlüsselspalten, `arg3` der erlaubte Arbeitsspeicher in Kilobyte, `arg4` zeigt an, ob zufälliger Zugriff auf das Sortierergebnis erforderlich ist, und `arg5` unterscheidet seriell, paralleler Worker oder paralleler Leader. |
| `sort-done` | `(bool, long)` | Wird ausgelöst, wenn eine Sortierung abgeschlossen ist. `arg0` ist wahr für externe Sortierung und falsch für interne Sortierung; `arg1` ist die Anzahl der für eine externe Sortierung verwendeten Plattenblöcke oder die für eine interne Sortierung verwendeten Kilobyte Speicher. |
| `lwlock-acquire` | `(char *, LWLockMode)` | Wird ausgelöst, wenn ein LWLock erworben wurde. `arg0` ist die Tranche des LWLocks, `arg1` der angeforderte Sperrmodus, exklusiv oder geteilt. |
| `lwlock-release` | `(char *)` | Wird ausgelöst, wenn ein LWLock freigegeben wurde; freigegebene Wartende wurden dabei noch nicht zwingend geweckt. `arg0` ist die Tranche des LWLocks. |
| `lwlock-wait-start` | `(char *, LWLockMode)` | Wird ausgelöst, wenn ein LWLock nicht sofort verfügbar war und ein Serverprozess begonnen hat, auf die Sperre zu warten. |
| `lwlock-wait-done` | `(char *, LWLockMode)` | Wird ausgelöst, wenn ein Serverprozess aus seinem Warten auf einen LWLock entlassen wurde; er besitzt die Sperre zu diesem Zeitpunkt noch nicht tatsächlich. |
| `lwlock-condacquire` | `(char *, LWLockMode)` | Wird ausgelöst, wenn ein LWLock erfolgreich erworben wurde, obwohl der Aufrufer kein Warten angegeben hatte. |
| `lwlock-condacquire-fail` | `(char *, LWLockMode)` | Wird ausgelöst, wenn ein LWLock nicht erfolgreich erworben wurde, obwohl der Aufrufer kein Warten angegeben hatte. |
| `lock-wait-start` | `(unsigned int, unsigned int, unsigned int, unsigned int, unsigned int, LOCKMODE)` | Wird ausgelöst, wenn eine Anforderung für eine Heavyweight-Sperre (`lmgr`-Sperre) zu warten beginnt, weil die Sperre nicht verfügbar ist. `arg0` bis `arg3` sind Tag-Felder des gesperrten Objekts, `arg4` gibt den Objekttyp an und `arg5` den angeforderten Sperrtyp. |
| `lock-wait-done` | `(unsigned int, unsigned int, unsigned int, unsigned int, unsigned int, LOCKMODE)` | Wird ausgelöst, wenn eine Anforderung für eine Heavyweight-Sperre mit dem Warten fertig ist, also die Sperre erworben hat. Die Argumente entsprechen `lock-wait-start`. |
| `deadlock-found` | `()` | Wird ausgelöst, wenn der Deadlock-Detektor einen Deadlock findet. |

**Tabelle 27.50. Definierte Typen in Probe-Parametern**

| Typ | Definition |
| --- | --- |
| `LocalTransactionId` | `unsigned int` |
| `LWLockMode` | `int` |
| `LOCKMODE` | `int` |
| `BlockNumber` | `unsigned int` |
| `Oid` | `unsigned int` |
| `ForkNumber` | `int` |
| `bool` | `unsigned char` |

### 27.5.3. Probes verwenden

Das folgende Beispiel zeigt ein DTrace-Skript zum Analysieren der Transaktionszahlen im System, als Alternative zum Erstellen von Snapshots von `pg_stat_database` vor und nach einem Performance-Test:

```text
#!/usr/sbin/dtrace -qs

postgresql$1:::transaction-start
{
      @start["Start"] = count();
      self->ts = timestamp;
}

postgresql$1:::transaction-abort
{
      @abort["Abort"] = count();
}

postgresql$1:::transaction-commit
/self->ts/
{
      @commit["Commit"] = count();
      @time["Total time (ns)"] = sum(timestamp - self->ts);
      self->ts=0;
}
```

Beim Ausführen erzeugt das D-Skript beispielsweise folgende Ausgabe:

```text
# ./txn_count.d `pgrep -n postgres` or ./txn_count.d <PID>
^C

Start                                                             71
Commit                                                            70
Total time (ns)                                           2312105013
```

> **Hinweis**
>
> SystemTap verwendet für Trace-Skripte eine andere Notation als DTrace, auch wenn die zugrunde liegenden Trace Points kompatibel sind. Beachtenswert ist, dass SystemTap-Skripte zum Zeitpunkt dieser Dokumentation Probe-Namen mit doppelten Unterstrichen statt Bindestrichen referenzieren müssen. Es wird erwartet, dass dies in zukünftigen SystemTap-Versionen behoben wird.

Denken Sie daran, dass DTrace-Skripte sorgfältig geschrieben und debuggt werden müssen; andernfalls können die gesammelten Trace-Informationen bedeutungslos sein. In den meisten Fällen, in denen Probleme gefunden werden, liegt der Fehler in der Instrumentierung, nicht im zugrunde liegenden System. Wenn Informationen diskutiert werden, die mit dynamischem Tracing gefunden wurden, sollten Sie das verwendete Skript beifügen, damit auch dieses geprüft und besprochen werden kann.

### 27.5.4. Neue Probes definieren

Neue Probes können überall im Code definiert werden, wo der Entwickler sie benötigt; dies erfordert jedoch eine Neukompilierung. Die Schritte zum Einfügen neuer Probes sind:

1. Legen Sie Probe-Namen und die Daten fest, die über die Probes verfügbar gemacht werden sollen.
2. Fügen Sie die Probe-Definitionen zu `src/backend/utils/probes.d` hinzu.
3. Binden Sie `pg_trace.h` ein, falls es in den Modulen mit den Probe-Punkten noch nicht vorhanden ist, und fügen Sie an den gewünschten Stellen im Quellcode `TRACE_POSTGRESQL`-Probe-Makros ein.
4. Kompilieren Sie neu und prüfen Sie, ob die neuen Probes verfügbar sind.

Beispiel: So würden Sie eine Probe hinzufügen, um alle neuen Transaktionen anhand der Transaktions-ID zu verfolgen.

1. Entscheiden Sie, dass die Probe `transaction-start` heißen und einen Parameter vom Typ `LocalTransactionId` benötigen soll.
2. Fügen Sie die Probe-Definition zu `src/backend/utils/probes.d` hinzu:

```text
probe transaction__start(LocalTransactionId);
```

Beachten Sie die Verwendung des doppelten Unterstrichs im Probe-Namen. In einem DTrace-Skript, das die Probe verwendet, muss der doppelte Unterstrich durch einen Bindestrich ersetzt werden; daher ist `transaction-start` der Name, der für Benutzer dokumentiert werden sollte.

Zur Kompilierzeit wird `transaction__start` in ein Makro namens `TRACE_POSTGRESQL_TRANSACTION_START` umgewandelt. Beachten Sie, dass die Unterstriche hier einfach sind. Dieses Makro ist durch Einbinden von `pg_trace.h` verfügbar. Fügen Sie den Makroaufruf an der passenden Stelle im Quellcode hinzu. In diesem Fall sieht er so aus:

```c
TRACE_POSTGRESQL_TRANSACTION_START(vxid.localTransactionId);
```

Nach dem Neukompilieren und Starten der neuen Binärdatei prüfen Sie mit dem folgenden DTrace-Befehl, ob die neu hinzugefügte Probe verfügbar ist. Die Ausgabe sollte ähnlich aussehen:

```text
# dtrace -ln transaction-start
   ID    PROVIDER          MODULE                              FUNCTION NAME
18705 postgresql49878     postgres                        StartTransactionCommand
 transaction-start
18755 postgresql49877     postgres                        StartTransactionCommand
 transaction-start
18805 postgresql49876     postgres                        StartTransactionCommand
 transaction-start
18855 postgresql49875     postgres                        StartTransactionCommand
 transaction-start
18986 postgresql49873     postgres                        StartTransactionCommand
 transaction-start
```

Beim Hinzufügen von Trace-Makros zu C-Code sind einige Dinge zu beachten:

- Achten Sie darauf, dass die für die Parameter einer Probe angegebenen Datentypen mit den Datentypen der im Makro verwendeten Variablen übereinstimmen. Andernfalls erhalten Sie Kompilierungsfehler.
- Auf den meisten Plattformen werden, wenn PostgreSQL mit `--enable-dtrace` gebaut wurde, die Argumente eines Trace-Makros immer dann ausgewertet, wenn der Kontrollfluss durch das Makro läuft, selbst wenn kein Tracing stattfindet. Das ist normalerweise kein Problem, wenn nur Werte einiger lokaler Variablen gemeldet werden. Vermeiden Sie aber teure Funktionsaufrufe in den Argumenten. Wenn Sie solche Aufrufe benötigen, sollten Sie das Makro mit einer Prüfung schützen, ob das Tracing tatsächlich aktiviert ist:

```c
if (TRACE_POSTGRESQL_TRANSACTION_START_ENABLED())
    TRACE_POSTGRESQL_TRANSACTION_START(some_function(...));
```

Jedes Trace-Makro hat ein entsprechendes `ENABLED`-Makro.

## 27.6. Plattennutzung überwachen

Dieser Abschnitt behandelt, wie man die Plattennutzung eines PostgreSQL-Datenbanksystems überwacht.

### 27.6.1. Plattennutzung bestimmen

Jede Tabelle hat eine primäre Heap-Datei auf Platte, in der die meisten Daten gespeichert werden. Wenn die Tabelle Spalten mit potenziell breiten Werten hat, kann außerdem eine TOAST-Datei mit der Tabelle verbunden sein, die Werte speichert, die zu breit sind, um bequem in die Haupttabelle zu passen (siehe [Abschnitt 66.2](66_Physische_Speicherung_der_Datenbank.md#662-toast)). Falls vorhanden, gibt es einen gültigen Index auf der TOAST-Tabelle. Außerdem können mit der Basistabelle Indizes verbunden sein. Jede Tabelle und jeder Index wird in einer separaten Plattendatei gespeichert, möglicherweise in mehr als einer Datei, wenn die Datei ein Gigabyte überschreiten würde. Die Namenskonventionen für diese Dateien werden in [Abschnitt 66.1](66_Physische_Speicherung_der_Datenbank.md#661-dateilayout-der-datenbank) beschrieben.

Sie können Plattenplatz auf drei Arten überwachen: mit den in Tabelle 9.102 aufgeführten SQL-Funktionen, mit dem Modul `oid2name` oder durch manuelle Untersuchung der Systemkataloge. Die SQL-Funktionen sind am einfachsten zu verwenden und werden im Allgemeinen empfohlen. Der Rest dieses Abschnitts zeigt die Vorgehensweise über die Untersuchung der Systemkataloge.

Mit `psql` können Sie auf einer kürzlich vacuumierten oder analysierten Datenbank Abfragen ausführen, um die Plattennutzung einer beliebigen Tabelle zu sehen:

```sql
SELECT pg_relation_filepath(oid), relpages FROM pg_class WHERE
 relname = 'customer';
```

```text
 pg_relation_filepath | relpages
----------------------+----------
 base/16384/16806     |       60
(1 row)
```

Jede Seite ist typischerweise 8 Kilobyte groß. Denken Sie daran, dass `relpages` nur durch `VACUUM`, `ANALYZE` und einige DDL-Befehle wie `CREATE INDEX` aktualisiert wird. Der Dateipfad ist interessant, wenn Sie die Plattendatei der Tabelle direkt untersuchen möchten.

Um den von TOAST-Tabellen verwendeten Platz anzuzeigen, verwenden Sie eine Abfrage wie die folgende:

```sql
SELECT relname, relpages
FROM pg_class,
     (SELECT reltoastrelid
      FROM pg_class
      WHERE relname = 'customer') AS ss
WHERE oid = ss.reltoastrelid OR
      oid = (SELECT indexrelid
             FROM pg_index
             WHERE indrelid = ss.reltoastrelid)
 ORDER BY relname;
```

```text
        relname        | relpages
----------------------+----------
 pg_toast_16806       |        0
 pg_toast_16806_index |        1
```

Auch Indexgrößen lassen sich leicht anzeigen:

```sql
SELECT c2.relname, c2.relpages
FROM pg_class c, pg_class c2, pg_index i
WHERE c.relname = 'customer' AND
      c.oid = i.indrelid AND
      c2.oid = i.indexrelid
ORDER BY c2.relname;
```

```text
      relname      | relpages
-------------------+----------
 customer_id_index |       26
```

Mit diesen Informationen lassen sich die größten Tabellen und Indizes leicht finden:

```sql
SELECT relname, relpages
FROM pg_class
ORDER BY relpages DESC;
```

```text
       relname        | relpages
----------------------+----------
 bigtable             |     3290
 customer             |     3144
```

### 27.6.2. Fehler durch volle Platte

Die wichtigste Aufgabe eines Datenbankadministrators bei der Plattenüberwachung besteht darin sicherzustellen, dass die Platte nicht voll läuft. Eine volle Datenplatte führt nicht zu Datenkorruption, kann aber nützliche Aktivität verhindern. Wenn die Platte mit den WAL-Dateien voll läuft, kann es zu einer Panic des Datenbankservers und anschließendem Herunterfahren kommen.

Wenn Sie auf der Platte keinen zusätzlichen Platz freigeben können, indem Sie andere Dinge löschen, können Sie mithilfe von Tablespaces einige Datenbankdateien auf andere Dateisysteme verschieben. Weitere Informationen dazu finden Sie in [Abschnitt 22.6](22_Datenbanken_verwalten.md#226-tablespaces).

> **Tipp**
>
> Einige Dateisysteme verhalten sich schlecht, wenn sie fast voll sind. Warten Sie daher nicht, bis die Platte vollständig voll ist, bevor Sie handeln.

Wenn Ihr System Plattenkontingente pro Benutzer unterstützt, unterliegt die Datenbank natürlich dem Kontingent, das für den Benutzer gesetzt ist, unter dem der Server läuft. Das Überschreiten des Kontingents hat dieselben schlechten Auswirkungen wie ein vollständig erschöpfter Plattenplatz.
