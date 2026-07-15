# 66. Physische Speicherung der Datenbank

Dieses Kapitel gibt einen Überblick über das physische Speicherformat, das von PostgreSQL-Datenbanken verwendet wird.

## 66.1. Dateilayout der Datenbank

Dieser Abschnitt beschreibt das Speicherformat auf der Ebene von Dateien und Verzeichnissen.

Traditionell werden die Konfigurations- und Datendateien, die ein Datenbank-Cluster verwendet, gemeinsam im Datenverzeichnis des Clusters gespeichert. Dieses Verzeichnis wird üblicherweise als `PGDATA` bezeichnet, nach dem Namen der Umgebungsvariable, mit der es definiert werden kann. Ein typischer Ort für `PGDATA` ist `/var/lib/pgsql/data`. Auf derselben Maschine können mehrere Cluster existieren, die von unterschiedlichen Serverinstanzen verwaltet werden.

Das Verzeichnis `PGDATA` enthält mehrere Unterverzeichnisse und Steuerdateien, wie in Tabelle 66.1 gezeigt. Zusätzlich zu diesen erforderlichen Elementen werden die Cluster-Konfigurationsdateien `postgresql.conf`, `pg_hba.conf` und `pg_ident.conf` traditionell in `PGDATA` gespeichert, obwohl sie auch an anderer Stelle liegen können.

**Tabelle 66.1. Inhalt von `PGDATA`**

| Element | Beschreibung |
| --- | --- |
| `PG_VERSION` | Datei mit der Hauptversionsnummer von PostgreSQL |
| `base` | Unterverzeichnis mit je einem Unterverzeichnis pro Datenbank |
| `current_logfiles` | Datei, die die aktuell vom Logging Collector geschriebenen Logdateien aufzeichnet |
| `global` | Unterverzeichnis mit clusterweiten Tabellen, etwa `pg_database` |
| `pg_commit_ts` | Unterverzeichnis mit Daten zu Transaktions-Commit-Zeitstempeln |
| `pg_dynshmem` | Unterverzeichnis mit Dateien, die vom Subsystem für dynamischen Shared Memory verwendet werden |
| `pg_logical` | Unterverzeichnis mit Statusdaten für logische Decodierung |
| `pg_multixact` | Unterverzeichnis mit Multitransaktions-Statusdaten, verwendet für gemeinsam genutzte Row-Locks |
| `pg_notify` | Unterverzeichnis mit Statusdaten für `LISTEN`/`NOTIFY` |
| `pg_replslot` | Unterverzeichnis mit Daten zu Replikations-Slots |
| `pg_serial` | Unterverzeichnis mit Informationen über committete serialisierbare Transaktionen |
| `pg_snapshots` | Unterverzeichnis mit exportierten Snapshots |
| `pg_stat` | Unterverzeichnis mit dauerhaften Dateien für das Statistik-Subsystem |
| `pg_stat_tmp` | Unterverzeichnis mit temporären Dateien für das Statistik-Subsystem |
| `pg_subtrans` | Unterverzeichnis mit Subtransaktions-Statusdaten |
| `pg_tblspc` | Unterverzeichnis mit symbolischen Links auf Tablespaces |
| `pg_twophase` | Unterverzeichnis mit Zustandsdateien für vorbereitete Transaktionen |
| `pg_wal` | Unterverzeichnis mit WAL-Dateien (Write-Ahead Log) |
| `pg_xact` | Unterverzeichnis mit Transaktions-Commit-Statusdaten |
| `postgresql.auto.conf` | Datei zum Speichern von Konfigurationsparametern, die mit `ALTER SYSTEM` gesetzt wurden |
| `postmaster.opts` | Datei, die die Befehlszeilenoptionen aufzeichnet, mit denen der Server zuletzt gestartet wurde |
| `postmaster.pid` | Lock-Datei mit der aktuellen Postmaster-Prozess-ID (PID), dem Pfad des Cluster-Datenverzeichnisses, dem Startzeitstempel des Postmasters, der Portnummer, dem Pfad des Unix-Domain-Socket-Verzeichnisses, der auch leer sein kann, der ersten gültigen `listen_address`, also IP-Adresse oder `*` oder leer, wenn nicht auf TCP gelauscht wird, und der Shared-Memory-Segment-ID. Diese Datei ist nach dem Herunterfahren des Servers nicht vorhanden. |

Für jede Datenbank im Cluster gibt es innerhalb von `PGDATA/base` ein Unterverzeichnis, benannt nach der OID der Datenbank in `pg_database`. Dieses Unterverzeichnis ist der Standardort für die Dateien der Datenbank; insbesondere werden dort ihre Systemkataloge gespeichert.

Die folgenden Abschnitte beschreiben das Verhalten der eingebauten Heap-Table-Access-Method und der eingebauten Index-Access-Methods. Wegen der erweiterbaren Natur von PostgreSQL können andere Access Methods anders funktionieren.

Jede Tabelle und jeder Index wird in einer separaten Datei gespeichert. Bei gewöhnlichen Relationen sind diese Dateien nach der Filenode-Nummer der Tabelle oder des Index benannt, die in `pg_class.relfilenode` zu finden ist. Bei temporären Relationen hat der Dateiname die Form `tBBB_FFF`, wobei `BBB` die Prozessnummer des Backends ist, das die Datei angelegt hat, und `FFF` die Filenode-Nummer ist. In beiden Fällen hat jede Tabelle und jeder Index zusätzlich zur Hauptdatei, auch Main Fork genannt, eine Free Space Map (siehe [Abschnitt 66.3](66_Physische_Speicherung_der_Datenbank.md#663-free-space-map)), die Informationen über freien Platz in der Relation speichert. Die Free Space Map liegt in einer Datei, deren Name aus der Filenode-Nummer plus dem Suffix `_fsm` besteht. Tabellen haben außerdem eine Visibility Map, gespeichert in einer Fork mit dem Suffix `_vm`, um zu verfolgen, welche Seiten bekanntermaßen keine toten Tupel enthalten. Die Visibility Map wird in [Abschnitt 66.4](66_Physische_Speicherung_der_Datenbank.md#664-visibility-map) weiter beschrieben. Unlogged Tabellen und Indizes haben eine dritte Fork, die Initialization Fork, die in einer Fork mit dem Suffix `_init` gespeichert wird (siehe [Abschnitt 66.5](66_Physische_Speicherung_der_Datenbank.md#665-initialization-fork)).

> **Achtung:** Obwohl die Filenode einer Tabelle häufig ihrer OID entspricht, ist das nicht notwendigerweise der Fall. Einige Operationen, etwa `TRUNCATE`, `REINDEX`, `CLUSTER` und manche Formen von `ALTER TABLE`, können die Filenode ändern, während die OID erhalten bleibt. Vermeiden Sie die Annahme, dass Filenode und Tabellen-OID identisch sind. Außerdem enthält `pg_class.relfilenode` für bestimmte Systemkataloge, einschließlich `pg_class` selbst, den Wert null. Die tatsächliche Filenode-Nummer dieser Kataloge ist in einer tieferliegenden Datenstruktur gespeichert und kann mit der Funktion `pg_relation_filenode()` ermittelt werden.

Wenn eine Tabelle oder ein Index 1 GB überschreitet, wird sie oder er in gigabytegroße Segmente aufgeteilt. Der Dateiname des ersten Segments ist identisch mit der Filenode; nachfolgende Segmente heißen `filenode.1`, `filenode.2` usw. Diese Anordnung vermeidet Probleme auf Plattformen mit Dateigrößenbeschränkungen. Tatsächlich ist 1 GB nur die Standardsegmentgröße. Die Segmentgröße kann beim Bauen von PostgreSQL mit der Konfigurationsoption `--with-segsize` angepasst werden. Grundsätzlich könnten auch Free-Space-Map- und Visibility-Map-Forks mehrere Segmente benötigen, auch wenn das in der Praxis unwahrscheinlich ist.

Eine Tabelle mit Spalten, die potenziell große Einträge enthalten, hat eine zugehörige TOAST-Tabelle, die zur Out-of-Line-Speicherung von Feldwerten verwendet wird, die zu groß sind, um direkt in den Tabellenzeilen gehalten zu werden. `pg_class.reltoastrelid` verknüpft eine Tabelle mit ihrer TOAST-Tabelle, falls es eine gibt. Siehe [Abschnitt 66.2](66_Physische_Speicherung_der_Datenbank.md#662-toast) für weitere Informationen.

Der Inhalt von Tabellen und Indizes wird in [Abschnitt 66.6](66_Physische_Speicherung_der_Datenbank.md#666-layout-von-datenbankseiten) weiter besprochen.

Tablespaces machen das Szenario komplizierter. Jeder benutzerdefinierte Tablespace hat innerhalb des Verzeichnisses `PGDATA/pg_tblspc` einen symbolischen Link, der auf das physische Tablespace-Verzeichnis zeigt, also auf den Ort, der im Befehl `CREATE TABLESPACE` des Tablespace angegeben wurde. Dieser symbolische Link ist nach der OID des Tablespace benannt. Innerhalb des physischen Tablespace-Verzeichnisses gibt es ein Unterverzeichnis mit einem Namen, der von der PostgreSQL-Serverversion abhängt, zum Beispiel `PG_9.0_201008051`. Dieser Unterordner wird verwendet, damit aufeinanderfolgende Versionen der Datenbank denselben Wert für den Speicherort in `CREATE TABLESPACE` verwenden können, ohne in Konflikt zu geraten. Innerhalb des versionsspezifischen Unterverzeichnisses gibt es für jede Datenbank, die Elemente im Tablespace hat, ein Unterverzeichnis, benannt nach der OID der Datenbank. Tabellen und Indizes werden dort nach dem Filenode-Namensschema gespeichert. Auf den Tablespace `pg_default` wird nicht über `pg_tblspc` zugegriffen; er entspricht `PGDATA/base`. Entsprechend wird auf den Tablespace `pg_global` nicht über `pg_tblspc` zugegriffen; er entspricht `PGDATA/global`.

Die Funktion `pg_relation_filepath()` zeigt den vollständigen Pfad einer Relation relativ zu `PGDATA`. Sie ist oft ein nützlicher Ersatz dafür, sich viele der oben genannten Regeln zu merken. Beachten Sie jedoch, dass diese Funktion nur den Namen des ersten Segments der Main Fork der Relation liefert. Möglicherweise müssen Sie eine Segmentnummer und/oder `_fsm`, `_vm` oder `_init` anhängen, um alle Dateien zu finden, die zu der Relation gehören.

Temporäre Dateien, etwa für Operationen wie Sortieren von mehr Daten, als in den Arbeitsspeicher passen, werden in `PGDATA/base/pgsql_tmp` angelegt oder in einem Unterverzeichnis `pgsql_tmp` eines Tablespace-Verzeichnisses, wenn dafür ein anderer Tablespace als `pg_default` angegeben ist. Der Name einer temporären Datei hat die Form `pgsql_tmpPPP.NNN`, wobei `PPP` die PID des besitzenden Backends ist und `NNN` verschiedene temporäre Dateien dieses Backends unterscheidet.

## 66.2. TOAST

Dieser Abschnitt gibt einen Überblick über TOAST, die „Oversized-Attribute Storage Technique“.

PostgreSQL verwendet eine feste Seitengröße, üblicherweise 8 kB, und erlaubt nicht, dass Tupel über mehrere Seiten reichen. Deshalb ist es nicht möglich, sehr große Feldwerte direkt zu speichern. Um diese Einschränkung zu umgehen, werden große Feldwerte komprimiert und/oder in mehrere physische Zeilen zerlegt. Das geschieht transparent für den Benutzer und hat nur geringen Einfluss auf den meisten Backend-Code. Die Technik ist liebevoll als TOAST bekannt. Die TOAST-Infrastruktur wird außerdem verwendet, um die Behandlung großer Datenwerte im Speicher zu verbessern.

Nur bestimmte Datentypen unterstützen TOAST; es ist nicht sinnvoll, Datentypen mit Verwaltungsaufwand zu belasten, die keine großen Feldwerte erzeugen können. Um TOAST zu unterstützen, muss ein Datentyp eine Darstellung variabler Länge, also eine `varlena`-Darstellung, haben. Dabei enthält normalerweise das erste Vier-Byte-Wort jedes gespeicherten Werts die Gesamtlänge des Werts in Byte einschließlich dieses Längenworts. TOAST schränkt die übrige Darstellung des Datentyps nicht ein. Die besonderen Darstellungen, die zusammen als TOASTed Values bezeichnet werden, funktionieren dadurch, dass sie dieses anfängliche Längenwort verändern oder neu interpretieren. Deshalb müssen C-Funktionen, die einen TOAST-fähigen Datentyp unterstützen, vorsichtig damit umgehen, wie sie potenziell TOASTed Eingabewerte behandeln: Eine Eingabe besteht möglicherweise erst nach dem Detoasting tatsächlich aus einem Vier-Byte-Längenwort und Inhalt. Normalerweise geschieht das durch Aufruf von `PG_DETOAST_DATUM`, bevor etwas mit einem Eingabewert getan wird; in manchen Fällen sind jedoch effizientere Ansätze möglich. Siehe Abschnitt 36.13.1 für weitere Details.

TOAST verwendet zwei Bits des `varlena`-Längenworts für sich, auf Big-Endian-Maschinen die höchstwertigen Bits und auf Little-Endian-Maschinen die niederwertigen Bits. Dadurch wird die logische Größe jedes Werts eines TOAST-fähigen Datentyps auf 1 GB begrenzt, also `2^30 - 1` Byte. Wenn beide Bits null sind, ist der Wert ein gewöhnlicher, nicht TOASTed Wert des Datentyps, und die übrigen Bits des Längenworts geben die Gesamtdatengröße einschließlich Längenwort in Byte an. Wenn das höchstwertige oder niederwertige Bit gesetzt ist, hat der Wert statt des normalen Vier-Byte-Headers nur einen Ein-Byte-Header, und die übrigen Bits dieses Bytes geben die Gesamtdatengröße einschließlich Längenbyte in Byte an. Diese Alternative unterstützt eine platzsparende Speicherung von Werten unter 127 Byte, erlaubt aber weiterhin, dass der Datentyp bei Bedarf bis 1 GB wächst. Werte mit Ein-Byte-Headern sind nicht an einer bestimmten Grenze ausgerichtet, während Werte mit Vier-Byte-Headern mindestens an einer Vier-Byte-Grenze ausgerichtet sind; der Wegfall von Alignment-Padding spart bei kurzen Werten zusätzlich erheblich Platz. Wenn als Sonderfall die übrigen Bits eines Ein-Byte-Headers alle null sind, was für eine Länge einschließlich sich selbst unmöglich wäre, ist der Wert ein Zeiger auf Out-of-Line-Daten. Dafür gibt es mehrere mögliche Varianten, die unten beschrieben werden. Typ und Größe eines solchen TOAST-Zeigers werden durch einen Code im zweiten Byte des Datums bestimmt. Wenn schließlich das höchstwertige oder niederwertige Bit frei ist, aber das benachbarte Bit gesetzt ist, wurde der Inhalt des Datums komprimiert und muss vor der Verwendung dekomprimiert werden. In diesem Fall geben die übrigen Bits des Vier-Byte-Längenworts die Gesamtgröße des komprimierten Datums an, nicht die ursprünglichen Daten. Beachten Sie, dass auch Out-of-Line-Daten komprimiert sein können; der `varlena`-Header zeigt dies jedoch nicht an, sondern der Inhalt des TOAST-Zeigers.

Die Kompressionstechnik für inline oder out-of-line komprimierte Daten kann pro Spalte gewählt werden, indem die Spaltenoption `COMPRESSION` in `CREATE TABLE` oder `ALTER TABLE` gesetzt wird. Für Spalten ohne explizite Einstellung wird beim Einfügen von Daten der Parameter `default_toast_compression` herangezogen.

Wie erwähnt gibt es mehrere Arten von TOAST-Zeiger-Datums. Die älteste und häufigste Art ist ein Zeiger auf Out-of-Line-Daten, die in einer TOAST-Tabelle gespeichert sind, die von der Tabelle getrennt, aber ihr zugeordnet ist, die das TOAST-Zeiger-Datum selbst enthält. Diese On-Disk-Zeiger-Datums werden vom TOAST-Verwaltungscode in `access/common/toast_internals.c` erzeugt, wenn ein auf Platte zu speicherndes Tupel zu groß ist, um unverändert gespeichert zu werden. Weitere Details stehen in [Abschnitt 66.2.1](66_Physische_Speicherung_der_Datenbank.md#6621-outoflineondisktoastspeicherung). Alternativ kann ein TOAST-Zeiger-Datum einen Zeiger auf Out-of-Line-Daten enthalten, die sich an anderer Stelle im Speicher befinden. Solche Datums sind notwendigerweise kurzlebig und erscheinen niemals auf Platte, sind aber sehr nützlich, um Kopieren und redundante Verarbeitung großer Datenwerte zu vermeiden. Weitere Details stehen in [Abschnitt 66.2.2](66_Physische_Speicherung_der_Datenbank.md#6622-outoflineinmemorytoastspeicherung).

### 66.2.1. Out-of-Line-On-Disk-TOAST-Speicherung

Wenn eine der Spalten einer Tabelle TOAST-fähig ist, hat die Tabelle eine zugehörige TOAST-Tabelle, deren OID im Eintrag `pg_class.reltoastrelid` der Tabelle gespeichert ist. On-Disk-TOASTed-Werte werden in der TOAST-Tabelle gehalten, wie unten genauer beschrieben.

Out-of-Line-Werte werden, nach Kompression falls verwendet, in Chunks von höchstens `TOAST_MAX_CHUNK_SIZE` Byte aufgeteilt. Standardmäßig wird dieser Wert so gewählt, dass vier Chunk-Zeilen auf eine Seite passen, also ungefähr 2000 Byte. Jeder Chunk wird als separate Zeile in der TOAST-Tabelle der besitzenden Tabelle gespeichert. Jede TOAST-Tabelle hat die Spalten `chunk_id`, eine OID, die den jeweiligen TOASTed Wert identifiziert, `chunk_seq`, eine Sequenznummer des Chunks innerhalb seines Werts, und `chunk_data`, die eigentlichen Daten des Chunks. Ein eindeutiger Index auf `chunk_id` und `chunk_seq` ermöglicht schnellen Zugriff. Ein Zeiger-Datum, das einen Out-of-Line-On-Disk-TOASTed-Wert repräsentiert, muss daher die OID der TOAST-Tabelle speichern, in der gesucht werden soll, sowie die OID des konkreten Werts, also seine `chunk_id`. Aus Bequemlichkeit speichern Zeiger-Datums außerdem die logische Datumgröße, also die ursprüngliche unkomprimierte Datenlänge, die physisch gespeicherte Größe, die bei Kompression abweicht, und die verwendete Kompressionsmethode, falls vorhanden. Einschließlich der `varlena`-Header-Bytes beträgt die Gesamtgröße eines On-Disk-TOAST-Zeiger-Datums daher immer 18 Byte, unabhängig von der tatsächlichen Größe des repräsentierten Werts.

Der TOAST-Verwaltungscode wird nur ausgelöst, wenn ein in einer Tabelle zu speichernder Zeilenwert breiter ist als `TOAST_TUPLE_THRESHOLD` Byte, normalerweise 2 kB. Der TOAST-Code komprimiert Feldwerte und/oder verschiebt sie out-of-line, bis der Zeilenwert kürzer ist als `TOAST_TUPLE_TARGET` Byte, ebenfalls normalerweise 2 kB und anpassbar, oder bis keine weiteren Gewinne möglich sind. Bei einer `UPDATE`-Operation bleiben Werte unveränderter Felder normalerweise unverändert erhalten; ein `UPDATE` einer Zeile mit Out-of-Line-Werten verursacht daher keine TOAST-Kosten, wenn sich keiner dieser Out-of-Line-Werte ändert.

Der TOAST-Verwaltungscode kennt vier verschiedene Strategien zur Speicherung TOAST-fähiger Spalten auf Platte:

- `PLAIN`
  - Verhindert sowohl Kompression als auch Out-of-Line-Speicherung. Dies ist die einzige mögliche Strategie für Spalten nicht TOAST-fähiger Datentypen.

- `EXTENDED`
  - Erlaubt sowohl Kompression als auch Out-of-Line-Speicherung. Dies ist die Standardeinstellung für die meisten TOAST-fähigen Datentypen. Zuerst wird Kompression versucht, anschließend Out-of-Line-Speicherung, falls die Zeile immer noch zu groß ist.

- `EXTERNAL`
  - Erlaubt Out-of-Line-Speicherung, aber keine Kompression. Die Verwendung von `EXTERNAL` macht Teilstring-Operationen auf breiten `text`- und `bytea`-Spalten schneller, zum Preis eines erhöhten Speicherverbrauchs, weil diese Operationen darauf optimiert sind, nur die benötigten Teile des Out-of-Line-Werts zu holen, wenn er nicht komprimiert ist.

- `MAIN`
  - Erlaubt Kompression, aber keine Out-of-Line-Speicherung. Tatsächlich wird Out-of-Line-Speicherung für solche Spalten dennoch durchgeführt, aber nur als letzter Ausweg, wenn es keine andere Möglichkeit gibt, die Zeile klein genug für eine Seite zu machen.

Jeder TOAST-fähige Datentyp legt eine Standardstrategie für Spalten dieses Datentyps fest, aber die Strategie für eine bestimmte Tabellenspalte kann mit `ALTER TABLE ... SET STORAGE` geändert werden.

`TOAST_TUPLE_TARGET` kann pro Tabelle mit `ALTER TABLE ... SET (toast_tuple_target = N)` angepasst werden.

Dieses Schema hat gegenüber einem einfacheren Ansatz, etwa dem Erlauben von Zeilenwerten über Seitengrenzen hinweg, mehrere Vorteile. Wenn Abfragen normalerweise durch Vergleiche mit relativ kleinen Schlüsselwerten eingeschränkt werden, findet der größte Teil der Executor-Arbeit mit dem Haupteintrag der Zeile statt. Die großen Werte TOASTed Attribute werden erst geholt, wenn die Ergebnismenge an den Client gesendet wird, sofern sie überhaupt ausgewählt wurden. Dadurch ist die Haupttabelle deutlich kleiner, und mehr ihrer Zeilen passen in den Shared-Buffer-Cache als ohne Out-of-Line-Speicherung. Auch Sortiermengen schrumpfen, und Sortierungen können häufiger vollständig im Speicher ausgeführt werden. Ein kleiner Test zeigte, dass eine Tabelle mit typischen HTML-Seiten und ihren URLs einschließlich TOAST-Tabelle in etwa der Hälfte der Rohdatengröße gespeichert wurde und dass die Haupttabelle nur ungefähr 10 % der gesamten Daten enthielt, nämlich die URLs und einige kleine HTML-Seiten. Es gab keinen Laufzeitunterschied im Vergleich zu einer nicht TOASTed Vergleichstabelle, in der alle HTML-Seiten auf 7 kB gekürzt wurden, damit sie passen.

### 66.2.2. Out-of-Line-In-Memory-TOAST-Speicherung

TOAST-Zeiger können auf Daten zeigen, die nicht auf Platte liegen, sondern an anderer Stelle im Speicher des aktuellen Serverprozesses. Solche Zeiger können offensichtlich nicht langlebig sein, sind aber dennoch nützlich. Derzeit gibt es zwei Unterfälle: Zeiger auf indirekte Daten und Zeiger auf expandierte Daten.

Indirekte TOAST-Zeiger zeigen einfach auf einen nicht indirekten `varlena`-Wert, der irgendwo im Speicher liegt. Dieser Fall wurde ursprünglich nur als Machbarkeitsnachweis geschaffen, wird derzeit aber bei logischer Decodierung verwendet, um möglicherweise vermeiden zu können, physische Tupel mit mehr als 1 GB zu erzeugen, was passieren könnte, wenn alle Out-of-Line-Feldwerte in das Tupel gezogen würden. Der Fall ist nur begrenzt nutzbar, weil der Erzeuger des Zeiger-Datums vollständig dafür verantwortlich ist, dass die referenzierten Daten so lange überleben, wie der Zeiger existieren könnte; es gibt keine Infrastruktur, die dabei hilft.

Expandierte TOAST-Zeiger sind nützlich für komplexe Datentypen, deren On-Disk-Darstellung für Berechnungen nicht besonders gut geeignet ist. Die Standard-`varlena`-Darstellung eines PostgreSQL-Arrays enthält zum Beispiel Dimensionsinformationen, eine Null-Bitmap, falls es Null-Elemente gibt, und danach die Werte aller Elemente in Reihenfolge. Wenn der Elementtyp selbst variabel lang ist, kann das N-te Element nur gefunden werden, indem alle vorhergehenden Elemente durchsucht werden. Diese Darstellung ist wegen ihrer Kompaktheit für die On-Disk-Speicherung geeignet, aber für Berechnungen mit dem Array ist eine expandierte oder zerlegte Darstellung viel angenehmer, in der die Startpositionen aller Elemente bereits identifiziert wurden. Der TOAST-Zeigermechanismus unterstützt dieses Bedürfnis, indem er erlaubt, dass ein als Referenz übergebenes `Datum` entweder auf einen Standard-`varlena`-Wert, also die On-Disk-Darstellung, zeigt oder auf einen TOAST-Zeiger, der irgendwo im Speicher auf eine expandierte Darstellung zeigt. Die Details dieser expandierten Darstellung liegen beim Datentyp, sie muss aber einen Standardheader haben und die übrigen API-Anforderungen aus `src/include/utils/expandeddatum.h` erfüllen. C-Funktionen, die mit dem Datentyp arbeiten, können beide Darstellungen behandeln. Funktionen, die die expandierte Darstellung nicht kennen und nur `PG_DETOAST_DATUM` auf ihre Eingaben anwenden, erhalten automatisch die traditionelle `varlena`-Darstellung; Unterstützung für eine expandierte Darstellung kann also schrittweise eingeführt werden, Funktion für Funktion.

TOAST-Zeiger auf expandierte Werte werden weiter in Lese-Schreib- und Nur-Lese-Zeiger unterteilt. Die referenzierte Darstellung ist in beiden Fällen dieselbe, aber eine Funktion, die einen Lese-Schreib-Zeiger erhält, darf den referenzierten Wert in-place ändern, während eine Funktion mit Nur-Lese-Zeiger dies nicht darf; sie muss zuerst eine Kopie anlegen, wenn sie eine veränderte Version des Werts erzeugen will. Diese Unterscheidung und einige zugehörige Konventionen machen es möglich, unnötiges Kopieren expandierter Werte während der Abfrageausführung zu vermeiden.

Für alle Arten von In-Memory-TOAST-Zeigern stellt der TOAST-Verwaltungscode sicher, dass ein solches Zeiger-Datum nicht versehentlich auf Platte gespeichert werden kann. In-Memory-TOAST-Zeiger werden vor der Speicherung automatisch zu normalen Inline-`varlena`-Werten expandiert und anschließend möglicherweise in On-Disk-TOAST-Zeiger umgewandelt, falls das enthaltende Tupel sonst zu groß wäre.

## 66.3. Free Space Map

Jede Heap- und Indexrelation, mit Ausnahme von Hash-Indizes, hat eine Free Space Map (FSM), um den verfügbaren Platz in der Relation zu verfolgen. Sie wird neben den Hauptdaten der Relation in einer separaten Relation-Fork gespeichert, benannt nach der Filenode-Nummer der Relation plus dem Suffix `_fsm`. Wenn die Filenode einer Relation zum Beispiel `12345` ist, wird die FSM in einer Datei namens `12345_fsm` im selben Verzeichnis wie die Hauptdatei der Relation gespeichert.

Die Free Space Map ist als Baum von FSM-Seiten organisiert. Die FSM-Seiten auf der untersten Ebene speichern den verfügbaren freien Platz auf jeder Heap- oder Indexseite, wobei für jede solche Seite ein Byte verwendet wird. Die oberen Ebenen aggregieren Informationen aus den unteren Ebenen.

Innerhalb jeder FSM-Seite befindet sich ein binärer Baum, gespeichert in einem Array mit einem Byte je Knoten. Jeder Blattknoten repräsentiert eine Heap-Seite oder eine FSM-Seite einer niedrigeren Ebene. In jedem inneren Knoten wird der höhere Wert seiner beiden Kinder gespeichert. Der Maximalwert der Blattknoten steht daher an der Wurzel.

Weitere Details zur Struktur der FSM sowie dazu, wie sie aktualisiert und durchsucht wird, finden sich in `src/backend/storage/freespace/README`. Das Modul `pg_freespacemap` kann verwendet werden, um die in Free Space Maps gespeicherten Informationen zu untersuchen.

## 66.4. Visibility Map

Jede Heap-Relation hat eine Visibility Map (VM), um zu verfolgen, welche Seiten nur Tupel enthalten, von denen bekannt ist, dass sie für alle aktiven Transaktionen sichtbar sind; außerdem verfolgt sie, welche Seiten nur eingefrorene Tupel enthalten. Sie wird neben den Hauptdaten der Relation in einer separaten Relation-Fork gespeichert, benannt nach der Filenode-Nummer der Relation plus dem Suffix `_vm`. Wenn die Filenode einer Relation zum Beispiel `12345` ist, wird die VM in einer Datei namens `12345_vm` im selben Verzeichnis wie die Hauptdatei der Relation gespeichert. Indizes haben keine VMs.

Die Visibility Map speichert zwei Bits pro Heap-Seite. Das erste Bit zeigt, wenn es gesetzt ist, an, dass die Seite all-visible ist, also keine Tupel enthält, die vacuumed werden müssen. Diese Information kann auch von Index-only Scans genutzt werden, um Abfragen ausschließlich mit dem Indextupel zu beantworten. Das zweite Bit bedeutet, wenn es gesetzt ist, dass alle Tupel auf der Seite eingefroren wurden. Das heißt, nicht einmal ein Anti-Wraparound-Vacuum muss die Seite erneut besuchen.

Die Map ist konservativ: PostgreSQL stellt sicher, dass eine gesetzte Bit-Aussage wahr ist; wenn ein Bit nicht gesetzt ist, kann die Aussage wahr sein oder nicht. Visibility-Map-Bits werden nur von Vacuum gesetzt, aber von jeder datenverändernden Operation auf einer Seite gelöscht.

Das Modul `pg_visibility` kann verwendet werden, um die in der Visibility Map gespeicherten Informationen zu untersuchen.

## 66.5. Initialization Fork

Jede unlogged Tabelle und jeder Index auf einer unlogged Tabelle hat eine Initialization Fork. Die Initialization Fork ist eine leere Tabelle oder ein leerer Index des passenden Typs. Wenn eine unlogged Tabelle wegen eines Absturzes auf leer zurückgesetzt werden muss, wird die Initialization Fork über die Main Fork kopiert, und alle anderen Forks werden gelöscht. Sie werden bei Bedarf automatisch neu erstellt.

## 66.6. Layout von Datenbankseiten

Dieser Abschnitt gibt einen Überblick über das Seitenformat, das innerhalb von PostgreSQL-Tabellen und -Indizes verwendet wird. Sequenzen und TOAST-Tabellen sind genauso formatiert wie gewöhnliche Tabellen.

In der folgenden Erklärung wird angenommen, dass ein Byte 8 Bit enthält. Außerdem bezeichnet der Begriff Item einen einzelnen Datenwert, der auf einer Seite gespeichert ist. In einer Tabelle ist ein Item eine Zeile; in einem Index ist ein Item ein Indexeintrag.

Jede Tabelle und jeder Index wird als Array von Seiten fester Größe gespeichert, normalerweise 8 kB, wobei beim Kompilieren des Servers eine andere Seitengröße gewählt werden kann. In einer Tabelle sind alle Seiten logisch gleichwertig, sodass ein bestimmtes Item, also eine Zeile, auf jeder Seite gespeichert werden kann. In Indizes ist die erste Seite im Allgemeinen als Metaseite reserviert, die Steuerinformationen enthält, und es kann je nach Index-Access-Method unterschiedliche Arten von Seiten innerhalb des Index geben.

Die Verwendung dieses Seitenformats ist für Tabellen- oder Index-Access-Methods nicht zwingend. Die Heap-Table-Access-Method verwendet dieses Format immer. Alle vorhandenen Indexmethoden verwenden ebenfalls das Grundformat, aber die auf Index-Metaseiten gespeicherten Daten folgen normalerweise nicht den Item-Layout-Regeln.

Tabelle 66.2 zeigt das Gesamtlayout einer Seite. Jede Seite besteht aus fünf Teilen.

**Tabelle 66.2. Gesamtlayout einer Seite**

| Element | Beschreibung |
| --- | --- |
| `PageHeaderData` | 24 Byte lang. Enthält allgemeine Informationen über die Seite, einschließlich Zeigern auf freien Platz. |
| `ItemIdData` | Array von Item-Identifikatoren, die auf die eigentlichen Items zeigen. Jeder Eintrag ist ein Paar aus Offset und Länge. 4 Byte je Item. |
| Freier Platz | Der nicht zugewiesene Platz. Neue Item-Identifikatoren werden vom Anfang dieses Bereichs aus zugewiesen, neue Items vom Ende aus. |
| Items | Die eigentlichen Items selbst. |
| Spezialbereich | Daten, die spezifisch für die Index-Access-Method sind. Verschiedene Methoden speichern unterschiedliche Daten. In gewöhnlichen Tabellen leer. |

Die ersten 24 Byte jeder Seite bestehen aus einem Seitenheader (`PageHeaderData`). Sein Format ist in Tabelle 66.3 dargestellt. Das erste Feld verfolgt den jüngsten WAL-Eintrag, der mit dieser Seite zusammenhängt. Das zweite Feld enthält die Seitenprüfsumme, wenn Prüfsummen aktiviert sind. Danach kommt ein Zwei-Byte-Feld mit Flag-Bits. Es folgen drei Zwei-Byte-Integerfelder (`pd_lower`, `pd_upper` und `pd_special`). Diese enthalten Byte-Offsets vom Seitenanfang zum Beginn des nicht zugewiesenen Platzes, zum Ende des nicht zugewiesenen Platzes und zum Beginn des Spezialbereichs. Die nächsten 2 Byte des Seitenheaders, `pd_pagesize_version`, speichern sowohl die Seitengröße als auch einen Versionsindikator. Seit PostgreSQL 8.3 ist die Versionsnummer 4; PostgreSQL 8.1 und 8.2 verwendeten Version 3, PostgreSQL 8.0 Version 2, PostgreSQL 7.3 und 7.4 Version 1 und frühere Releases Version 0. Das grundlegende Seitenlayout und Headerformat hat sich in den meisten dieser Versionen nicht geändert, wohl aber das Layout der Heap-Zeilenheader. Die Seitengröße dient im Wesentlichen nur als Plausibilitätsprüfung; es gibt keine Unterstützung dafür, in einer Installation mehr als eine Seitengröße zu haben. Das letzte Feld ist ein Hinweis darauf, ob Pruning der Seite wahrscheinlich lohnend ist: Es verfolgt die älteste noch nicht geprunte `XMAX` auf der Seite.

**Tabelle 66.3. Layout von `PageHeaderData`**

| Feld | Typ | Länge | Beschreibung |
| --- | --- | --- | --- |
| `pd_lsn` | `PageXLogRecPtr` | 8 Byte | LSN: nächstes Byte nach dem letzten Byte des WAL-Records für die letzte Änderung an dieser Seite |
| `pd_checksum` | `uint16` | 2 Byte | Seitenprüfsumme |
| `pd_flags` | `uint16` | 2 Byte | Flag-Bits |
| `pd_lower` | `LocationIndex` | 2 Byte | Offset zum Beginn des freien Platzes |
| `pd_upper` | `LocationIndex` | 2 Byte | Offset zum Ende des freien Platzes |
| `pd_special` | `LocationIndex` | 2 Byte | Offset zum Beginn des Spezialbereichs |
| `pd_pagesize_version` | `uint16` | 2 Byte | Informationen zu Seitengröße und Layoutversionsnummer |
| `pd_prune_xid` | `TransactionId` | 4 Byte | Älteste ungeprunte `XMAX` auf der Seite oder null, wenn keine vorhanden ist |

Alle Details stehen in `src/include/storage/bufpage.h`.

Nach dem Seitenheader folgen Item-Identifikatoren (`ItemIdData`), die jeweils vier Byte benötigen. Ein Item-Identifikator enthält einen Byte-Offset zum Beginn eines Items, seine Länge in Byte und einige Attributbits, die seine Interpretation beeinflussen. Neue Item-Identifikatoren werden bei Bedarf vom Anfang des nicht zugewiesenen Platzes aus zugewiesen. Die Anzahl vorhandener Item-Identifikatoren kann aus `pd_lower` bestimmt werden, das zur Zuweisung eines neuen Identifikators erhöht wird. Da ein Item-Identifikator erst verschoben wird, wenn er freigegeben ist, kann sein Index langfristig verwendet werden, um ein Item zu referenzieren, selbst wenn das Item selbst auf der Seite verschoben wird, um freien Platz zu verdichten. Tatsächlich besteht jeder Zeiger auf ein Item (`ItemPointer`, auch als `CTID` bekannt), den PostgreSQL erzeugt, aus einer Seitennummer und dem Index eines Item-Identifikators.

Die Items selbst werden in Platz gespeichert, der rückwärts vom Ende des nicht zugewiesenen Platzes aus zugewiesen wird. Die genaue Struktur hängt davon ab, was die Tabelle enthalten soll. Tabellen und Sequenzen verwenden beide eine Struktur namens `HeapTupleHeaderData`, die unten beschrieben wird.

Der letzte Abschnitt ist der Spezialbereich, der alles enthalten kann, was die Access Method speichern möchte. B-Tree-Indizes speichern dort zum Beispiel Links auf die linken und rechten Nachbarseiten sowie weitere Daten, die für die Indexstruktur relevant sind. Gewöhnliche Tabellen verwenden überhaupt keinen Spezialbereich; das wird dadurch angezeigt, dass `pd_special` gleich der Seitengröße gesetzt wird.

Abbildung 66.1 veranschaulicht, wie diese Teile auf einer Seite angeordnet sind.

**Abbildung 66.1. Seitenlayout**

```text
+----------------+--------+--------+--------------+--------+---------+
| PageHeaderData | ItemId | ItemId | Freier Platz |  Item  | Spezial |
+----------------+--------+--------+--------------+--------+---------+
```

### 66.6.1. Layout von Tabellenzeilen

Alle Tabellenzeilen sind gleich strukturiert. Es gibt einen Header fester Größe, auf den meisten Maschinen 23 Byte, gefolgt von einer optionalen Null-Bitmap, einem optionalen Objekt-ID-Feld und den Benutzerdaten. Der Header ist in Tabelle 66.4 dargestellt. Die eigentlichen Benutzerdaten, also die Spalten der Zeile, beginnen bei dem Offset, der durch `t_hoff` angegeben wird; dieser muss immer ein Vielfaches der `MAXALIGN`-Distanz der Plattform sein. Die Null-Bitmap ist nur vorhanden, wenn das Bit `HEAP_HASNULL` in `t_infomask` gesetzt ist. Wenn sie vorhanden ist, beginnt sie direkt nach dem festen Header und belegt genug Bytes, um ein Bit pro Datenspalte zu haben, also so viele Bits wie der Attributzähler in `t_infomask2` angibt. In dieser Bitliste bedeutet ein 1-Bit not-null und ein 0-Bit null. Wenn die Bitmap nicht vorhanden ist, werden alle Spalten als not-null angenommen. Die Objekt-ID ist nur vorhanden, wenn das Bit `HEAP_HASOID_OLD` in `t_infomask` gesetzt ist. Wenn sie vorhanden ist, erscheint sie direkt vor der `t_hoff`-Grenze. Padding, das nötig ist, um `t_hoff` auf ein `MAXALIGN`-Vielfaches zu bringen, erscheint zwischen Null-Bitmap und Objekt-ID. Dadurch wird wiederum sichergestellt, dass die Objekt-ID passend ausgerichtet ist.

**Tabelle 66.4. Layout von `HeapTupleHeaderData`**

| Feld | Typ | Länge | Beschreibung |
| --- | --- | --- | --- |
| `t_xmin` | `TransactionId` | 4 Byte | Insert-XID-Stempel |
| `t_xmax` | `TransactionId` | 4 Byte | Delete-XID-Stempel |
| `t_cid` | `CommandId` | 4 Byte | Insert- und/oder Delete-CID-Stempel, überlagert mit `t_xvac` |
| `t_xvac` | `TransactionId` | 4 Byte | XID für eine VACUUM-Operation, die eine Zeilenversion verschiebt |
| `t_ctid` | `ItemPointerData` | 6 Byte | Aktuelle TID dieser oder einer neueren Zeilenversion |
| `t_infomask2` | `uint16` | 2 Byte | Anzahl der Attribute plus verschiedene Flag-Bits |
| `t_infomask` | `uint16` | 2 Byte | Verschiedene Flag-Bits |
| `t_hoff` | `uint8` | 1 Byte | Offset zu den Benutzerdaten |

Alle Details stehen in `src/include/access/htup_details.h`.

Die eigentlichen Daten können nur mit Informationen interpretiert werden, die aus anderen Tabellen stammen, vor allem aus `pg_attribute`. Die wichtigsten Werte zur Bestimmung von Feldpositionen sind `attlen` und `attalign`. Es gibt keine Möglichkeit, ein bestimmtes Attribut direkt zu holen, außer wenn nur Felder fester Breite und keine Nullwerte vorhanden sind. Diese ganze Technik ist in den Funktionen `heap_getattr`, `fastgetattr` und `heap_getsysattr` gekapselt.

Um die Daten zu lesen, muss jedes Attribut nacheinander untersucht werden. Zuerst wird anhand der Null-Bitmap geprüft, ob das Feld `NULL` ist. Wenn ja, geht es mit dem nächsten Feld weiter. Danach muss die richtige Ausrichtung hergestellt werden. Wenn das Feld feste Breite hat, liegen einfach alle Bytes dort. Wenn es ein Feld variabler Länge ist (`attlen = -1`), ist es etwas komplizierter. Alle Datentypen variabler Länge teilen sich die gemeinsame Headerstruktur `struct varlena`, die die Gesamtlänge des gespeicherten Werts und einige Flag-Bits enthält. Abhängig von den Flags können die Daten inline oder in einer TOAST-Tabelle liegen; sie können außerdem komprimiert sein (siehe [Abschnitt 66.2](66_Physische_Speicherung_der_Datenbank.md#662-toast)).

## 66.7. Heap-Only Tuples (HOT)

Für hohe Nebenläufigkeit verwendet PostgreSQL Multiversion Concurrency Control (MVCC), um Zeilen zu speichern. MVCC hat jedoch Nachteile bei Update-Abfragen. Insbesondere erfordern Updates, dass neue Versionen von Zeilen zu Tabellen hinzugefügt werden. Das kann auch neue Indexeinträge für jede aktualisierte Zeile erfordern, und das Entfernen alter Versionen von Zeilen und ihrer Indexeinträge kann teuer sein.

Um den Aufwand von Updates zu reduzieren, hat PostgreSQL eine Optimierung namens Heap-Only Tuples (HOT). Diese Optimierung ist möglich, wenn:

- Das Update keine Spalten verändert, auf die sich die Indizes der Tabelle beziehen, zusammenfassende Indizes ausgenommen. Die einzige zusammenfassende Indexmethode in der PostgreSQL-Kerndistribution ist BRIN.

- Auf der Seite, die die alte Zeile enthält, genügend freier Platz für die aktualisierte Zeile vorhanden ist.

In solchen Fällen bieten Heap-Only Tuples zwei Optimierungen:

- Neue Indexeinträge sind nicht erforderlich, um aktualisierte Zeilen zu repräsentieren; Zusammenfassungsindizes müssen jedoch möglicherweise trotzdem aktualisiert werden.

- Wenn eine Zeile mehrfach aktualisiert wird, können Zeilenversionen außer der ältesten und der neuesten während des normalen Betriebs vollständig entfernt werden, auch durch `SELECT`s, statt periodische Vacuum-Operationen zu erfordern. Indizes verweisen immer auf den Page-Item-Identifikator der ursprünglichen Zeilenversion. Die zu dieser Zeilenversion gehörenden Tupeldaten werden entfernt, und ihr Item-Identifikator wird in eine Umleitung umgewandelt, die auf die älteste Version zeigt, die für eine gleichzeitige Transaktion noch sichtbar sein könnte. Zwischenversionen, die für niemanden mehr sichtbar sind, werden vollständig entfernt, und die zugehörigen Page-Item-Identifikatoren werden wieder zur Wiederverwendung verfügbar gemacht.

Sie können die Wahrscheinlichkeit ausreichenden Seitenplatzes für HOT-Updates erhöhen, indem Sie den `fillfactor` einer Tabelle senken. Auch ohne das werden HOT-Updates auftreten, weil neue Zeilen natürlicherweise auf neue Seiten wandern und vorhandene Seiten genügend freien Platz für neue Zeilenversionen haben können. Die System-View `pg_stat_all_tables` erlaubt es, das Auftreten von HOT- und Nicht-HOT-Updates zu überwachen.
