# Anhang M. Glossar

Dies ist eine Liste von Begriffen und ihrer Bedeutung im Kontext von PostgreSQL und relationalen Datenbanksystemen allgemein.

- **ACID**
  - Atomicity, Consistency, Isolation und Durability. Diese Eigenschaften von Datenbanktransaktionen sollen gültige Ergebnisse bei nebenläufiger Ausführung sowie auch bei Fehlern, Stromausfällen und ähnlichen Ereignissen gewährleisten.

- **Aggregatfunktion (Routine)**
  - Eine Funktion, die mehrere Eingabewerte kombiniert, etwa durch Zählen, Mitteln oder Addieren, und daraus einen einzelnen Ausgabewert erzeugt.
  - Weitere Informationen finden Sie in [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen).
  - Siehe auch: Window-Funktion (Routine).

- **Access Method**
  - Schnittstellen, über die PostgreSQL auf Daten in Tabellen und Indizes zugreift. Diese Abstraktion ermöglicht Unterstützung für neue Arten der Datenspeicherung.
  - Weitere Informationen finden Sie in [Kapitel 62](62_Schnittstellendefinition_für_Table_Access_Methods.md) und [Kapitel 63](63_Schnittstellendefinition_für_Index_Access_Methods.md).

- **Analytische Funktion**
  - Siehe Window-Funktion (Routine).

- **Analyze (Operation)**
  - Das Sammeln von Statistiken aus Tabellen und anderen Relationen, damit der Query-Planner entscheiden kann, wie Abfragen ausgeführt werden sollen.
  - Nicht zu verwechseln mit der Option `ANALYZE` des Befehls `EXPLAIN`.
  - Weitere Informationen finden Sie bei `ANALYZE`.

- **Asynchronous I/O (AIO)**
  - Asynchronous I/O beschreibt nicht blockierende, also asynchrone I/O. Im Gegensatz dazu blockiert synchrone I/O für die gesamte Dauer der I/O-Operation.
  - Bei AIO ist das Starten einer I/O-Operation vom Warten auf ihr Ergebnis getrennt. Dadurch können mehrere I/O-Operationen gleichzeitig angestoßen werden; außerdem können CPU-intensive Operationen parallel zur I/O laufen. Der Preis dieser höheren Nebenläufigkeit ist größere Komplexität.
  - Siehe auch: Input/Output.

- **Atomic**
  - Bezogen auf ein Datum: Die Eigenschaft, dass sein Wert nicht in kleinere Bestandteile zerlegt werden kann.
  - Bezogen auf eine Datenbanktransaktion: siehe Atomicity.

- **Atomicity**
  - Die Eigenschaft einer Transaktion, dass entweder alle ihre Operationen als eine Einheit abgeschlossen werden oder keine davon. Tritt während der Ausführung einer Transaktion ein Systemausfall auf, sind nach der Wiederherstellung außerdem keine Teilergebnisse sichtbar. Dies ist eine der ACID-Eigenschaften.

- **Attribut**
  - Ein Element mit einem bestimmten Namen und Datentyp innerhalb eines Tupels.

- **Autovacuum (Prozess)**
  - Eine Gruppe von Hintergrundprozessen, die regelmäßig `VACUUM`- und `ANALYZE`-Operationen ausführen. Der ständig vorhandene Hilfsprozess, der diese Arbeit koordiniert, sofern Autovacuum nicht deaktiviert ist, heißt Autovacuum Launcher. Die Prozesse, die die eigentliche Arbeit ausführen, heißen Autovacuum Worker.
  - Weitere Informationen finden Sie in [Abschnitt 24.1.6](24_Regelmäßige_Datenbankwartung.md#2416-der-autovacuumdaemon).

- **Hilfsprozess**
  - Ein Prozess innerhalb einer Instanz, der für eine bestimmte Hintergrundaufgabe der Instanz zuständig ist. Zu den Hilfsprozessen gehören der Autovacuum Launcher, aber nicht die Autovacuum Worker, außerdem Background Writer, Checkpointer, Logger, Startup-Prozess, WAL Archiver, WAL Receiver, aber nicht die WAL Sender, WAL Summarizer und WAL Writer.

- **Backend (Prozess)**
  - Prozess einer Instanz, der im Auftrag einer Client-Sitzung handelt und deren Anfragen bearbeitet.
  - Nicht zu verwechseln mit den ähnlichen Begriffen Background Worker oder Background Writer.

- **Background Worker (Prozess)**
  - Prozess innerhalb einer Instanz, der vom System oder vom Benutzer bereitgestellten Code ausführt. Dient als Infrastruktur für mehrere Funktionen in PostgreSQL, etwa logische Replikation und parallele Abfragen. Außerdem können Erweiterungen eigene Background-Worker-Prozesse hinzufügen.
  - Weitere Informationen finden Sie in [Kapitel 46](46_Background_Worker_Prozesse.md).

- **Background Writer (Prozess)**
  - Ein Hilfsprozess, der schmutzige Datenseiten aus dem gemeinsamen Speicher in das Dateisystem schreibt. Er wacht periodisch auf, arbeitet aber jeweils nur kurz, um seine teure I/O-Aktivität über die Zeit zu verteilen und größere I/O-Spitzen zu vermeiden, die andere Prozesse blockieren könnten.
  - Weitere Informationen finden Sie in [Abschnitt 19.4.4](19_Serverkonfiguration.md#1944-hintergrundschreiber).

- **Base Backup**
  - Eine binäre Kopie aller Dateien eines Datenbank-Clusters. Sie wird vom Werkzeug `pg_basebackup` erzeugt. Zusammen mit WAL-Dateien kann sie als Ausgangspunkt für Wiederherstellung, Log Shipping oder Streaming Replication verwendet werden.

- **Bloat**
  - Speicherplatz in Datenseiten, der keine aktuellen Zeilenversionen enthält, etwa ungenutzter freier Platz oder veraltete Zeilenversionen.

- **Bootstrap-Superuser**
  - Der erste Benutzer, der in einem Datenbank-Cluster initialisiert wird. Dieser Benutzer besitzt alle Systemkatalogtabellen in jeder Datenbank. Er ist außerdem die Rolle, von der alle gewährten Berechtigungen ausgehen. Aus diesen Gründen darf diese Rolle nicht gelöscht werden.
  - Diese Rolle verhält sich außerdem wie ein normaler Datenbank-Superuser; ihr Superuser-Status kann nicht entfernt werden.

- **Buffer Access Strategy**
  - Manche Operationen greifen auf sehr viele Seiten zu. Eine Buffer Access Strategy hilft zu verhindern, dass solche Operationen zu viele Seiten aus den Shared Buffers verdrängen.
  - Eine Buffer Access Strategy richtet Verweise auf eine begrenzte Zahl von Shared Buffers ein und verwendet sie zyklisch wieder. Wenn die Operation eine neue Seite benötigt, wird ein Opfer-Buffer aus den Buffern im Strategie-Ring gewählt. Dabei kann es nötig sein, die schmutzigen Daten der Seite und eventuell noch nicht geschriebene WAL-Daten auf dauerhaften Speicher zu schreiben.
  - Buffer Access Strategies werden für verschiedene Operationen verwendet, etwa sequentielle Scans großer Tabellen, `VACUUM`, `COPY`, `CREATE TABLE AS SELECT`, `ALTER TABLE`, `CREATE DATABASE`, `CREATE INDEX` und `CLUSTER`.

- **Cast**
  - Die Umwandlung eines Datums von seinem aktuellen Datentyp in einen anderen Datentyp.
  - Weitere Informationen finden Sie bei `CREATE CAST`.

- **Catalog**
  - Der SQL-Standard verwendet diesen Begriff für das, was in der PostgreSQL-Terminologie Datenbank heißt.
  - Nicht zu verwechseln mit Systemkatalog.
  - Weitere Informationen finden Sie in [Abschnitt 22.1](22_Datenbanken_verwalten.md#221-überblick).

- **Check-Constraint**
  - Eine Art Constraint, der für eine Relation definiert wird und die zulässigen Werte in einem oder mehreren Attributen einschränkt. Ein Check-Constraint kann sich auf jedes Attribut derselben Zeile der Relation beziehen, aber nicht auf andere Zeilen derselben Relation oder auf andere Relationen.
  - Weitere Informationen finden Sie in [Abschnitt 5.5](05_Datendefinition.md#55-integritätsbedingungen-constraints).

- **Checkpoint**
  - Ein Punkt in der WAL-Sequenz, an dem garantiert ist, dass Heap- und Index-Datendateien mit allen Informationen aktualisiert wurden, die vor diesem Checkpoint im gemeinsamen Speicher geändert waren. Ein Checkpoint-Datensatz wird geschrieben und nach WAL geflusht, um diesen Punkt zu markieren.
  - Checkpoint bezeichnet auch die Ausführung aller Aktionen, die nötig sind, um einen solchen Punkt zu erreichen. Dieser Prozess wird gestartet, wenn vordefinierte Bedingungen erfüllt sind, etwa wenn eine bestimmte Zeit vergangen ist oder ein bestimmtes Volumen von Datensätzen geschrieben wurde. Er kann auch durch den Benutzer mit dem Befehl `CHECKPOINT` ausgelöst werden.
  - Weitere Informationen finden Sie in [Abschnitt 28.5](28_Zuverlässigkeit_und_Write_Ahead_Log.md#285-walkonfiguration).

- **Checkpointer (Prozess)**
  - Ein Hilfsprozess, der für das Ausführen von Checkpoints verantwortlich ist.

- **Class (veraltet)**
  - Siehe Relation.

- **Client (Prozess)**
  - Ein beliebiger, möglicherweise entfernter Prozess, der durch Verbindung mit einer Instanz eine Sitzung herstellt, um mit einer Datenbank zu interagieren.

- **Cluster-Eigentümer**
  - Der Betriebssystembenutzer, dem das Datenverzeichnis gehört und unter dem der Prozess `postgres` ausgeführt wird. Dieser Benutzer muss existieren, bevor ein neuer Datenbank-Cluster erzeugt wird.
  - Auf Betriebssystemen mit einem Benutzer `root` darf dieser Benutzer nicht Cluster-Eigentümer sein.

- **Spalte**
  - Ein Attribut in einer Tabelle oder Sicht.

- **Commit**
  - Das Abschließen einer Transaktion innerhalb der Datenbank, wodurch sie für andere Transaktionen sichtbar wird und ihre Dauerhaftigkeit zugesichert ist.
  - Weitere Informationen finden Sie bei `COMMIT`.

- **Nebenläufigkeit**
  - Das Konzept, dass mehrere unabhängige Operationen gleichzeitig innerhalb der Datenbank stattfinden. In PostgreSQL wird Nebenläufigkeit durch Multiversion Concurrency Control gesteuert.

- **Verbindung**
  - Eine eingerichtete Kommunikationsverbindung zwischen einem Client-Prozess und einem Backend-Prozess, normalerweise über ein Netzwerk, die eine Sitzung unterstützt. Der Begriff wird manchmal als Synonym für Sitzung verwendet.
  - Weitere Informationen finden Sie in [Abschnitt 19.3](19_Serverkonfiguration.md#193-verbindungen-und-authentifizierung).

- **Consistency**
  - Die Eigenschaft, dass die Daten in der Datenbank stets den Integritäts-Constraints entsprechen. Transaktionen dürfen manche Constraints vorübergehend verletzen; wenn diese Verletzungen bis zum Commit nicht behoben sind, wird die Transaktion automatisch zurückgerollt. Dies ist eine der ACID-Eigenschaften.

- **Constraint**
  - Eine Einschränkung der Werte, die innerhalb einer Tabelle oder in Attributen einer Domain zulässig sind.
  - Weitere Informationen finden Sie in [Abschnitt 5.5](05_Datendefinition.md#55-integritätsbedingungen-constraints).

- **Kumulatives Statistiksystem**
  - Ein System, das, sofern aktiviert, statistische Informationen über die Aktivitäten der Instanz sammelt.
  - Weitere Informationen finden Sie in [Abschnitt 27.2](27_Datenbankaktivität_überwachen.md#272-das-kumulative-statistiksystem).

- **Datenbereich**
  - Siehe Datenverzeichnis.

- **Datenbank**
  - Eine benannte Sammlung lokaler SQL-Objekte.
  - Weitere Informationen finden Sie in [Abschnitt 22.1](22_Datenbanken_verwalten.md#221-überblick).

- **Datenbank-Cluster**
  - Eine Sammlung von Datenbanken und globalen SQL-Objekten sowie deren gemeinsamen statischen und dynamischen Metadaten. Wird manchmal kurz Cluster genannt. Ein Datenbank-Cluster wird mit dem Programm `initdb` erzeugt.
  - In PostgreSQL wird der Begriff Cluster gelegentlich auch für eine Instanz verwendet. Nicht zu verwechseln mit dem SQL-Befehl `CLUSTER`.
  - Siehe auch Cluster-Eigentümer, den Betriebssystemeigentümer eines Clusters, und Bootstrap-Superuser, den PostgreSQL-Eigentümer eines Clusters.

- **Datenbankserver**
  - Siehe Instanz.

- **Datenbank-Superuser**
  - Eine Rolle mit Superuser-Status; siehe [Abschnitt 21.2](21_Datenbankrollen.md#212-rollenattribute).
  - Häufig kurz Superuser genannt.

- **Datenverzeichnis**
  - Das Basisverzeichnis im Dateisystem eines Servers, das alle Datendateien und Unterverzeichnisse enthält, die zu einem Datenbank-Cluster gehören, mit Ausnahme von Tablespaces und optional WAL. Die Umgebungsvariable `PGDATA` wird häufig verwendet, um auf das Datenverzeichnis zu verweisen.
  - Der Speicherplatz eines Clusters besteht aus dem Datenverzeichnis plus zusätzlichen Tablespaces.
  - Weitere Informationen finden Sie in [Abschnitt 66.1](66_Physische_Speicherung_der_Datenbank.md#661-dateilayout-der-datenbank).

- **Datenseite**
  - Die grundlegende Struktur zum Speichern von Relationsdaten. Alle Seiten haben dieselbe Größe. Datenseiten werden normalerweise auf der Festplatte in einer bestimmten Datei gespeichert und können in Shared Buffers gelesen werden, wo sie geändert und dadurch schmutzig werden. Sie werden sauber, wenn sie auf die Festplatte geschrieben werden. Neue Seiten, die zunächst nur im Speicher existieren, sind ebenfalls schmutzig, bis sie geschrieben wurden.

- **Datum**
  - Die interne Darstellung eines Werts eines SQL-Datentyps.

- **Delete**
  - Ein SQL-Befehl, der Zeilen aus einer bestimmten Tabelle oder Relation entfernt.
  - Weitere Informationen finden Sie bei `DELETE`.

- **Domain**
  - Ein benutzerdefinierter Datentyp, der auf einem anderen zugrunde liegenden Datentyp basiert. Er verhält sich wie der zugrunde liegende Typ, kann aber die Menge der zulässigen Werte einschränken.
  - Weitere Informationen finden Sie in [Abschnitt 8.18](08_Datentypen.md#818-domänentypen).

- **Durability**
  - Die Zusicherung, dass Änderungen nach dem Commit einer Transaktion auch nach einem Systemausfall oder Absturz erhalten bleiben. Dies ist eine der ACID-Eigenschaften.

- **Epoch**
  - Siehe Transaction ID.

- **Extension**
  - Ein Software-Zusatzpaket, das auf einer Instanz installiert werden kann, um zusätzliche Funktionen bereitzustellen.
  - Weitere Informationen finden Sie in [Abschnitt 36.17](36_SQL_erweitern.md#3617-zusammengehörige-objekte-in-einer-erweiterung-paketieren).

- **File segment**
  - Eine physische Datei, die Daten für eine bestimmte Relation speichert. File Segments sind durch einen Konfigurationswert in ihrer Größe begrenzt, typischerweise 1 Gigabyte. Überschreitet eine Relation diese Größe, wird sie in mehrere Segmente aufgeteilt.
  - Weitere Informationen finden Sie in [Abschnitt 66.1](66_Physische_Speicherung_der_Datenbank.md#661-dateilayout-der-datenbank).
  - Nicht zu verwechseln mit dem ähnlichen Begriff WAL-Segment.

- **Foreign Data Wrapper**
  - Ein Mittel, um Daten darzustellen, die nicht in der lokalen Datenbank enthalten sind, sodass sie wie lokale Tabellen erscheinen. Mit einem Foreign Data Wrapper können ein Foreign Server und Foreign Tables definiert werden.
  - Weitere Informationen finden Sie bei `CREATE FOREIGN DATA WRAPPER`.

- **Foreign Key**
  - Eine Art Constraint, der für eine oder mehrere Spalten einer Tabelle definiert wird und verlangt, dass die Werte in diesen Spalten null oder eine Zeile in einer anderen, selten auch derselben, Tabelle identifizieren.

- **Foreign Server**
  - Eine benannte Sammlung von Foreign Tables, die alle denselben Foreign Data Wrapper verwenden und weitere gemeinsame Konfigurationswerte besitzen.
  - Weitere Informationen finden Sie bei `CREATE SERVER`.

- **Foreign Table (Relation)**
  - Eine Relation, die Zeilen und Spalten ähnlich einer normalen Tabelle zu haben scheint, Datenanforderungen aber über ihren Foreign Data Wrapper weiterleitet. Dieser liefert Ergebnismengen zurück, die entsprechend der Definition der Foreign Table strukturiert sind.
  - Weitere Informationen finden Sie bei `CREATE FOREIGN TABLE`.

- **Fork**
  - Jede der getrennten, segmentierten Dateigruppen, in denen eine Relation gespeichert wird. Der Main Fork enthält die eigentlichen Daten. Daneben gibt es zwei sekundäre Forks für Metadaten: die Free Space Map und die Visibility Map. Unlogged Relations besitzen außerdem einen Init Fork.

- **Free Space Map (Fork)**
  - Eine Speicherstruktur, die Metadaten zu jeder Datenseite des Main Fork einer Tabelle hält. Der Free-Space-Map-Eintrag für jede Seite speichert die Menge freien Platzes, die für künftige Tupel verfügbar ist, und ist so strukturiert, dass verfügbarer Platz für ein neues Tupel bestimmter Größe effizient gesucht werden kann.
  - Weitere Informationen finden Sie in [Abschnitt 66.3](66_Physische_Speicherung_der_Datenbank.md#663-free-space-map).

- **Funktion (Routine)**
  - Eine Art Routine, die null oder mehr Argumente entgegennimmt, null oder mehr Ausgabewerte zurückgibt und auf die Ausführung innerhalb einer Transaktion beschränkt ist. Funktionen werden als Teil einer Abfrage aufgerufen, zum Beispiel über `SELECT`. Bestimmte Funktionen können Mengen zurückgeben; sie heißen Set-Returning Functions.
  - Funktionen können außerdem von Triggern aufgerufen werden.
  - Weitere Informationen finden Sie bei `CREATE FUNCTION`.

- **GMT**
  - Siehe UTC.

- **Grant**
  - Ein SQL-Befehl, mit dem einem Benutzer oder einer Rolle Zugriff auf bestimmte Objekte innerhalb der Datenbank erlaubt wird.
  - Weitere Informationen finden Sie bei `GRANT`.

- **Heap**
  - Enthält die Werte der Zeilenattribute, also die Daten, einer Relation. Der Heap wird in einem oder mehreren File Segments im Main Fork der Relation realisiert.

- **Host**
  - Ein Computer, der über ein Netzwerk mit anderen Computern kommuniziert. Der Begriff wird manchmal als Synonym für Server verwendet. Er wird auch für einen Computer verwendet, auf dem Client-Prozesse laufen.

- **Index (Relation)**
  - Eine Relation, die aus einer Tabelle oder materialisierten Sicht abgeleitete Daten enthält. Ihre interne Struktur unterstützt schnellen Abruf und schnellen Zugriff auf die ursprünglichen Daten.
  - Weitere Informationen finden Sie bei `CREATE INDEX`.

- **Inkrementelles Backup**
  - Ein spezielles Base Backup, das für manche Dateien nur diejenigen Seiten enthalten kann, die seit einem vorherigen Backup geändert wurden, statt den vollständigen Inhalt jeder Datei. Wie Base Backups wird es vom Werkzeug `pg_basebackup` erzeugt.
  - Zur Wiederherstellung inkrementeller Backups wird das Werkzeug `pg_combinebackup` verwendet. Es kombiniert inkrementelle Backups mit einem Base Backup. Anschließend kann die Wiederherstellung WAL verwenden, um den Datenbank-Cluster in einen konsistenten Zustand zu bringen.
  - Weitere Informationen finden Sie in [Abschnitt 25.3.3](25_Sicherung_und_Wiederherstellung.md#2533-eine-inkrementelle-sicherung-erstellen).

- **Input/Output (I/O)**
  - Input/Output beschreibt die Kommunikation zwischen einem Programm und Peripheriegeräten. Im Kontext von Datenbanksystemen bezeichnet I/O häufig, aber nicht ausschließlich, die Interaktion mit Speichergeräten oder dem Netzwerk.
  - Siehe auch: Asynchronous I/O.

- **Insert**
  - Ein SQL-Befehl, mit dem neue Daten in eine Tabelle eingefügt werden.
  - Weitere Informationen finden Sie bei `INSERT`.

- **Instanz**
  - Eine Gruppe von Backend- und Hilfsprozessen, die über einen gemeinsamen Shared-Memory-Bereich kommunizieren. Ein Postmaster-Prozess verwaltet die Instanz; eine Instanz verwaltet genau einen Datenbank-Cluster mit allen seinen Datenbanken. Auf demselben Server können mehrere Instanzen laufen, solange ihre TCP-Ports nicht kollidieren.
  - Die Instanz behandelt alle wesentlichen Funktionen eines DBMS: Lese- und Schreibzugriff auf Dateien und Shared Memory, Sicherstellung der ACID-Eigenschaften, Verbindungen zu Client-Prozessen, Rechteprüfung, Crash Recovery, Replikation und mehr.

- **Isolation**
  - Die Eigenschaft, dass die Auswirkungen einer Transaktion für nebenläufige Transaktionen vor ihrem Commit nicht sichtbar sind. Dies ist eine der ACID-Eigenschaften.
  - Weitere Informationen finden Sie in [Abschnitt 13.2](13_Nebenläufigkeitskontrolle.md#132-transaktionsisolation).

- **Join**
  - Eine Operation und ein SQL-Schlüsselwort, die in Abfragen zum Kombinieren von Daten aus mehreren Relationen verwendet werden.

- **Key**
  - Ein Mittel, um eine Zeile innerhalb einer Tabelle oder anderen Relation anhand von Werten zu identifizieren, die in einem oder mehreren Attributen dieser Relation enthalten sind.

- **Lock**
  - Ein Mechanismus, mit dem ein Prozess gleichzeitigen Zugriff auf eine Ressource einschränken oder verhindern kann.

- **Logdatei**
  - Logdateien enthalten menschenlesbare Textzeilen über Ereignisse, zum Beispiel fehlgeschlagene Anmeldungen oder lange laufende Abfragen.
  - Weitere Informationen finden Sie in [Abschnitt 24.3](24_Regelmäßige_Datenbankwartung.md#243-logdateiwartung).

- **Logged**
  - Eine Tabelle gilt als logged, wenn Änderungen an ihr an WAL gesendet werden. Standardmäßig sind alle normalen Tabellen logged. Eine Tabelle kann bei der Erstellung oder per `ALTER TABLE` als unlogged festgelegt werden.

- **Logger (Prozess)**
  - Ein Hilfsprozess, der, sofern aktiviert, Informationen über Datenbankereignisse in die aktuelle Logdatei schreibt. Wenn bestimmte zeit- oder volumenabhängige Kriterien erreicht sind, wird eine neue Logdatei erzeugt. Wird auch `syslogger` genannt.
  - Weitere Informationen finden Sie in [Abschnitt 19.8](19_Serverkonfiguration.md#198-fehlerberichte-und-logging).

- **Logischer Replikations-Cluster**
  - Eine Menge von Publisher- und Subscriber-Instanzen, wobei die Publisher-Instanz Änderungen an die Subscriber-Instanz repliziert.

- **Log record**
  - Veralteter Begriff für WAL Record.

- **Log Sequence Number (LSN)**
  - Byte-Offset in WAL, der mit jedem neuen WAL Record monoton steigt.
  - Weitere Informationen finden Sie bei `pg_lsn` und in [Abschnitt 28.6](28_Zuverlässigkeit_und_Write_Ahead_Log.md#286-walinterna).

- **LSN**
  - Siehe Log Sequence Number.

- **Master (Server)**
  - Siehe Primary (Server).

- **Materialized**
  - Die Eigenschaft, dass bestimmte Informationen vorab berechnet und für spätere Verwendung gespeichert wurden, statt sie bei Bedarf jedes Mal neu zu berechnen.
  - Der Begriff wird bei materialisierten Sichten verwendet und bedeutet dort, dass die aus der Abfrage der Sicht abgeleiteten Daten getrennt von ihren Quelldaten auf der Festplatte gespeichert werden.
  - Er wird außerdem bei manchen mehrstufigen Abfragen verwendet und bedeutet dann, dass die aus einem bestimmten Schritt resultierenden Daten im Speicher gehalten werden, mit möglichem Auslagern auf die Festplatte, sodass ein anderer Schritt sie mehrfach lesen kann.

- **Materialisierte Sicht (Relation)**
  - Eine Relation, die wie eine Sicht durch eine `SELECT`-Anweisung definiert wird, ihre Daten aber wie eine Tabelle speichert. Sie kann nicht mit `INSERT`, `UPDATE`, `DELETE` oder `MERGE` geändert werden.
  - Weitere Informationen finden Sie bei `CREATE MATERIALIZED VIEW`.

- **Merge**
  - Ein SQL-Befehl, mit dem Zeilen in einer bestimmten Tabelle abhängig von Daten aus einer Quellrelation bedingt hinzugefügt, geändert oder entfernt werden.
  - Weitere Informationen finden Sie bei `MERGE`.

- **Multi-Version Concurrency Control (MVCC)**
  - Ein Mechanismus, der mehreren Transaktionen erlauben soll, dieselben Zeilen zu lesen und zu schreiben, ohne dass ein Prozess andere Prozesse zum Stillstand bringt. In PostgreSQL wird MVCC umgesetzt, indem bei Änderungen Kopien, also Versionen, von Tupeln erzeugt werden. Nachdem Transaktionen beendet sind, die die alten Versionen noch sehen konnten, müssen diese alten Versionen entfernt werden.

- **NULL**
  - Ein Konzept der Nicht-Existenz, das ein zentraler Bestandteil der relationalen Datenbanktheorie ist. Es steht für das Fehlen eines bestimmten Werts.

- **Optimizer**
  - Siehe Query Planner.

- **Parallele Abfrage**
  - Die Fähigkeit, Teile der Ausführung einer Abfrage auf Servern mit mehreren CPUs durch parallele Prozesse bearbeiten zu lassen.

- **Partition**
  - Eine von mehreren disjunkten, also nicht überlappenden, Teilmengen einer größeren Menge.
  - Bezogen auf eine partitionierte Tabelle: eine der Tabellen, die jeweils einen Teil der Daten der partitionierten Tabelle enthalten, die als Parent bezeichnet wird. Die Partition ist selbst eine Tabelle und kann daher auch direkt abgefragt werden. Gleichzeitig kann eine Partition gelegentlich selbst eine partitionierte Tabelle sein, wodurch Hierarchien möglich werden.
  - Bezogen auf eine Window-Funktion in einer Abfrage: ein vom Benutzer definiertes Kriterium, das festlegt, welche benachbarten Zeilen der Ergebnismenge der Abfrage von der Funktion berücksichtigt werden können.

- **Partitionierte Tabelle (Relation)**
  - Eine Relation, die semantisch einer Tabelle entspricht, deren Speicherung aber über mehrere Partitionen verteilt ist.

- **Postmaster (Prozess)**
  - Der allererste Prozess einer Instanz. Er startet und verwaltet die Hilfsprozesse und erzeugt bei Bedarf Backend-Prozesse.
  - Weitere Informationen finden Sie in [Abschnitt 18.3](18_Servereinrichtung_und_Betrieb.md#183-den-datenbankserver-starten).

- **Primary Key**
  - Ein Sonderfall eines Unique Constraint, der für eine Tabelle oder andere Relation definiert wird und zusätzlich garantiert, dass keines der Attribute im Primary Key `NULL`-Werte enthält. Wie der Name andeutet, kann es pro Tabelle nur einen Primary Key geben, obwohl mehrere Unique Constraints möglich sind, deren Attribute ebenfalls keine `NULL`-Werte zulassen.

- **Primary (Server)**
  - Wenn zwei oder mehr Datenbanken über Replikation verbunden sind, heißt der Server, der als maßgebliche Informationsquelle gilt, Primary, auch Master genannt.

- **Procedure (Routine)**
  - Eine Art Routine. Ihre unterscheidenden Eigenschaften sind, dass sie keine Werte zurückgibt und Transaktionsanweisungen wie `COMMIT` und `ROLLBACK` ausführen darf. Sie wird mit dem Befehl `CALL` aufgerufen.
  - Weitere Informationen finden Sie bei `CREATE PROCEDURE`.

- **Query**
  - Eine Anfrage, die von einem Client an ein Backend gesendet wird, normalerweise um Ergebnisse zurückzugeben oder Daten in der Datenbank zu ändern.

- **Query Planner**
  - Der Teil von PostgreSQL, der die effizienteste Ausführungsweise für Abfragen bestimmt, also plant. Auch Query Optimizer, Optimizer oder schlicht Planner genannt.

- **Record**
  - Siehe Tupel.

- **Recycling**
  - Siehe WAL-Datei.

- **Referentielle Integrität**
  - Ein Mittel, um Daten in einer Relation durch einen Foreign Key so einzuschränken, dass passende Daten in einer anderen Relation vorhanden sein müssen.

- **Relation**
  - Der allgemeine Begriff für alle Objekte in einer Datenbank, die einen Namen und eine Liste von Attributen in einer bestimmten Reihenfolge besitzen. Tabellen, Sequenzen, Sichten, Foreign Tables, materialisierte Sichten, zusammengesetzte Typen und Indizes sind alles Relationen.
  - Allgemeiner ist eine Relation eine Menge von Tupeln; zum Beispiel ist auch das Ergebnis einer Abfrage eine Relation.
  - In PostgreSQL ist Class ein veraltetes Synonym für Relation.

- **Replica (Server)**
  - Eine Datenbank, die mit einer Primary-Datenbank gepaart ist und eine Kopie einiger oder aller Daten der Primary-Datenbank pflegt. Die wichtigsten Gründe dafür sind besserer Zugriff auf diese Daten und die Aufrechterhaltung der Verfügbarkeit, falls die Primary nicht verfügbar wird.

- **Replikation**
  - Das Reproduzieren von Daten eines Servers auf einem anderen Server, der Replica genannt wird. Dies kann physische Replikation sein, bei der alle Dateiänderungen eines Servers unverändert kopiert werden, oder logische Replikation, bei der eine definierte Teilmenge von Datenänderungen in einer höherwertigen Darstellung übertragen wird.

- **Restartpoint**
  - Eine Variante eines Checkpoints, die auf einer Replica ausgeführt wird.
  - Weitere Informationen finden Sie in [Abschnitt 28.5](28_Zuverlässigkeit_und_Write_Ahead_Log.md#285-walkonfiguration).

- **Ergebnismenge**
  - Eine Relation, die nach Abschluss eines SQL-Befehls von einem Backend-Prozess an einen Client übertragen wird. Meist handelt es sich um `SELECT`, aber auch `INSERT`, `UPDATE`, `DELETE` oder `MERGE` können eine Ergebnismenge liefern, wenn die Klausel `RETURNING` angegeben ist.
  - Dass eine Ergebnismenge eine Relation ist, bedeutet, dass eine Abfrage in der Definition einer anderen Abfrage verwendet werden kann und dadurch zur Unterabfrage wird.

- **Revoke**
  - Ein Befehl, der den Zugriff auf eine benannte Menge von Datenbankobjekten für eine benannte Liste von Rollen verhindert.
  - Weitere Informationen finden Sie bei `REVOKE`.

- **Role**
  - Eine Sammlung von Zugriffsrechten auf die Instanz. Rollen sind selbst ein Privileg, das anderen Rollen gewährt werden kann. Dies geschieht oft aus Bequemlichkeit oder um Vollständigkeit sicherzustellen, wenn mehrere Benutzer dieselben Rechte benötigen.
  - Weitere Informationen finden Sie bei `CREATE ROLE`.

- **Rollback**
  - Ein Befehl, der alle Operationen rückgängig macht, die seit Beginn einer Transaktion ausgeführt wurden.
  - Weitere Informationen finden Sie bei `ROLLBACK`.

- **Routine**
  - Eine definierte Menge von Anweisungen, die im Datenbanksystem gespeichert ist und zur Ausführung aufgerufen werden kann. Eine Routine kann in verschiedenen Programmiersprachen geschrieben sein. Routinen können Funktionen, einschließlich Set-Returning Functions und Triggerfunktionen, Aggregatfunktionen und Prozeduren sein.
  - Viele Routinen sind bereits in PostgreSQL selbst definiert; benutzerdefinierte Routinen können ebenfalls hinzugefügt werden.

- **Row**
  - Siehe Tupel.

- **Savepoint**
  - Eine besondere Marke in der Abfolge der Schritte einer Transaktion. Datenänderungen nach diesem Zeitpunkt können auf den Zustand des Savepoints zurückgesetzt werden.
  - Weitere Informationen finden Sie bei `SAVEPOINT`.

- **Schema**
  - Ein Schema ist ein Namensraum für SQL-Objekte, die alle in derselben Datenbank liegen. Jedes SQL-Objekt muss in genau einem Schema liegen.
  - Alle systemdefinierten SQL-Objekte liegen im Schema `pg_catalog`.
  - Allgemeiner wird der Begriff Schema für alle Datenbeschreibungen, etwa Tabellendefinitionen, Constraints und Kommentare, einer bestimmten Datenbank oder eines Teils davon verwendet.
  - Weitere Informationen finden Sie in [Abschnitt 5.10](05_Datendefinition.md#510-schemata).

- **Segment**
  - Siehe File Segment.

- **Select**
  - Der SQL-Befehl, mit dem Daten aus einer Datenbank angefordert werden. Normalerweise wird von `SELECT`-Befehlen nicht erwartet, dass sie die Datenbank in irgendeiner Weise ändern; allerdings können innerhalb der Abfrage aufgerufene Funktionen Nebeneffekte haben, die Daten verändern.
  - Weitere Informationen finden Sie bei `SELECT`.

- **Sequenz (Relation)**
  - Eine Art Relation, die zum Erzeugen von Werten verwendet wird. Typischerweise sind die erzeugten Werte fortlaufende, nicht wiederholte Zahlen. Sie werden häufig verwendet, um Surrogate Primary Keys zu erzeugen.

- **Server**
  - Ein Computer, auf dem PostgreSQL-Instanzen laufen. Der Begriff Server bezeichnet echte Hardware, einen Container oder eine virtuelle Maschine.
  - Der Begriff wird manchmal auch für eine Instanz oder einen Host verwendet.

- **Sitzung**
  - Ein Zustand, der einem Client und einem Backend erlaubt, über eine Verbindung miteinander zu interagieren.

- **Shared Memory**
  - RAM, der von den Prozessen einer Instanz gemeinsam verwendet wird. Er spiegelt Teile von Datenbankdateien, stellt einen flüchtigen Bereich für WAL Records bereit und speichert zusätzliche gemeinsame Informationen. Shared Memory gehört zur gesamten Instanz, nicht zu einer einzelnen Datenbank.
  - Der größte Teil des Shared Memory heißt Shared Buffers und spiegelt einen Teil der Datendateien, organisiert in Seiten. Wenn eine Seite geändert wird, heißt sie dirty page, bis sie zurück in das Dateisystem geschrieben wird.
  - Weitere Informationen finden Sie in Abschnitt 19.4.1.

- **SQL-Objekt**
  - Jedes Objekt, das mit einem `CREATE`-Befehl erzeugt werden kann. Die meisten Objekte gehören zu einer bestimmten Datenbank und werden allgemein lokale Objekte genannt.
  - Die meisten lokalen Objekte liegen in einem bestimmten Schema ihrer Datenbank, etwa Relationen aller Art, Routinen aller Art und Datentypen. Namen solcher Objekte desselben Typs im selben Schema müssen eindeutig sein.
  - Es gibt auch lokale Objekte, die nicht in Schemas liegen, etwa Extensions, Datentyp-Casts und Foreign Data Wrappers. Namen solcher Objekte desselben Typs müssen innerhalb der Datenbank eindeutig sein.
  - Andere Objekttypen, etwa Rollen, Tablespaces, Replication Origins, Subscriptions für logische Replikation und Datenbanken selbst, sind keine lokalen SQL-Objekte, weil sie vollständig außerhalb einer bestimmten Datenbank existieren. Sie heißen globale Objekte. Namen solcher Objekte müssen im gesamten Datenbank-Cluster eindeutig sein.
  - Weitere Informationen finden Sie in [Abschnitt 22.1](22_Datenbanken_verwalten.md#221-überblick).

- **SQL-Standard**
  - Eine Reihe von Dokumenten, die die Sprache SQL definieren.

- **Standby (Server)**
  - Siehe Replica (Server).

- **Startup-Prozess**
  - Ein Hilfsprozess, der während Crash Recovery und in einer physischen Replica WAL wiedergibt.
  - Der Name ist historisch: Der Startup-Prozess erhielt seinen Namen, bevor Replikation implementiert wurde. Er bezieht sich auf seine Aufgabe beim Serverstart nach einem Absturz.

- **Superuser**
  - In dieser Dokumentation ein Synonym für Datenbank-Superuser.

- **Systemkatalog**
  - Eine Sammlung von Tabellen, die die Struktur aller SQL-Objekte der Instanz beschreiben. Der Systemkatalog liegt im Schema `pg_catalog`. Diese Tabellen enthalten Daten in interner Darstellung und gelten normalerweise nicht als besonders nützlich für direkte Benutzerabfragen. Mehrere benutzerfreundlichere Sichten, ebenfalls im Schema `pg_catalog`, bieten bequemeren Zugriff auf Teile dieser Informationen. Weitere Tabellen und Sichten liegen im Schema `information_schema`; siehe [Kapitel 35](35_Das_Information_Schema.md). Sie stellen einige gleiche und zusätzliche Informationen bereit, wie vom SQL-Standard gefordert.
  - Weitere Informationen finden Sie in [Abschnitt 5.10](05_Datendefinition.md#510-schemata).

- **Tabelle**
  - Eine Sammlung von Tupeln mit gemeinsamer Datenstruktur, also gleicher Anzahl von Attributen in derselben Reihenfolge mit demselben Namen und Typ an jeder Position. Eine Tabelle ist die häufigste Form von Relation in PostgreSQL.
  - Weitere Informationen finden Sie bei `CREATE TABLE`.

- **Tablespace**
  - Ein benannter Ort im Dateisystem des Servers. Alle SQL-Objekte, die über ihre Definition im Systemkatalog hinaus Speicher benötigen, müssen zu genau einem Tablespace gehören. Anfangs enthält ein Datenbank-Cluster einen verwendbaren Tablespace, der als Standard für alle SQL-Objekte dient: `pg_default`.
  - Weitere Informationen finden Sie in [Abschnitt 22.6](22_Datenbanken_verwalten.md#226-tablespaces).

- **Temporäre Tabelle**
  - Tabellen, die je nach Festlegung bei ihrer Erstellung entweder für die Lebensdauer einer Sitzung oder einer Transaktion existieren. Ihre Daten sind für andere Sitzungen nicht sichtbar und werden nicht geloggt. Temporäre Tabellen werden oft verwendet, um Zwischendaten für eine mehrstufige Operation zu speichern.
  - Weitere Informationen finden Sie bei `CREATE TABLE`.

- **TOAST**
  - Ein Mechanismus, mit dem große Attribute von Tabellenzeilen aufgeteilt und in einer sekundären Tabelle gespeichert werden, der TOAST-Tabelle. Jede Relation mit großen Attributen besitzt ihre eigene TOAST-Tabelle.
  - Weitere Informationen finden Sie in [Abschnitt 66.2](66_Physische_Speicherung_der_Datenbank.md#662-toast).

- **Transaktion**
  - Eine Kombination von Befehlen, die als ein einzelner atomarer Befehl wirken müssen: Entweder alle haben als Einheit Erfolg, oder alle schlagen als Einheit fehl. Ihre Auswirkungen sind für andere Sitzungen erst sichtbar, wenn die Transaktion abgeschlossen ist, abhängig von der Isolationsstufe eventuell auch später.
  - Weitere Informationen finden Sie in [Abschnitt 13.2](13_Nebenläufigkeitskontrolle.md#132-transaktionsisolation).

- **Transaction ID**
  - Der numerische, eindeutige und fortlaufend zugewiesene Bezeichner, den jede Transaktion erhält, wenn sie erstmals eine Datenbankänderung verursacht. Häufig als `xid` abgekürzt. Auf der Festplatte sind XIDs nur 32 Bit breit, sodass nur ungefähr vier Milliarden Schreib-Transaktions-IDs erzeugt werden können. Damit das System länger laufen kann, werden ebenfalls 32 Bit breite Epochs verwendet. Wenn der Zähler den maximalen XID-Wert erreicht, beginnt er wieder bei 3, weil kleinere Werte reserviert sind, und der Epoch-Wert wird um eins erhöht. In manchen Kontexten werden Epoch und XID zusammen als ein einzelner 64-Bit-Wert betrachtet; weitere Einzelheiten finden Sie in [Abschnitt 67.1](67_Transaktionsverarbeitung.md#671-transaktionen-und-identifikatoren).
  - Weitere Informationen finden Sie in [Abschnitt 8.19](08_Datentypen.md#819-objektbezeichnertypen).

- **Transactions per Second (TPS)**
  - Durchschnittliche Anzahl von Transaktionen, die pro Sekunde ausgeführt werden, aufsummiert über alle Sitzungen, die während eines gemessenen Laufs aktiv sind. Dies dient als Maß für die Performance-Eigenschaften einer Instanz.

- **Trigger**
  - Eine Funktion, die so definiert werden kann, dass sie ausgeführt wird, wenn eine bestimmte Operation, etwa `INSERT`, `UPDATE`, `DELETE` oder `TRUNCATE`, auf eine Relation angewendet wird. Ein Trigger läuft innerhalb derselben Transaktion wie die Anweisung, die ihn ausgelöst hat. Wenn die Funktion fehlschlägt, schlägt auch die auslösende Anweisung fehl.
  - Weitere Informationen finden Sie bei `CREATE TRIGGER`.

- **Tupel**
  - Eine Sammlung von Attributen in fester Reihenfolge. Diese Reihenfolge kann durch die Tabelle oder andere Relation definiert sein, in der das Tupel enthalten ist; in diesem Fall wird das Tupel oft Zeile genannt. Sie kann auch durch die Struktur einer Ergebnismenge definiert sein; in diesem Fall wird das Tupel manchmal Record genannt.

- **Unique Constraint**
  - Eine Art Constraint, der für eine Relation definiert wird und die zulässigen Werte in einer oder einer Kombination von Spalten so einschränkt, dass jeder Wert oder jede Wertekombination nur einmal in der Relation vorkommen kann. Mit anderen Worten: Keine andere Zeile in der Relation enthält gleiche Werte.
  - Da `NULL`-Werte nicht als gleich zueinander gelten, dürfen mehrere Zeilen mit `NULL`-Werten existieren, ohne den Unique Constraint zu verletzen.

- **Unlogged**
  - Die Eigenschaft bestimmter Relationen, dass Änderungen an ihnen nicht in WAL erscheinen. Dadurch sind Replikation und Crash Recovery für diese Relationen deaktiviert.
  - Der Hauptzweck von unlogged tables ist das Speichern flüchtiger Arbeitsdaten, die prozessübergreifend geteilt werden müssen.
  - Temporäre Tabellen sind immer unlogged.

- **Update**
  - Ein SQL-Befehl, mit dem bereits vorhandene Zeilen in einer bestimmten Tabelle geändert werden. Er kann keine Zeilen erzeugen oder entfernen.
  - Weitere Informationen finden Sie bei `UPDATE`.

- **User**
  - Eine Rolle mit Login-Privileg; siehe [Abschnitt 21.2](21_Datenbankrollen.md#212-rollenattribute).

- **User Mapping**
  - Die Übersetzung von Login-Zugangsdaten in der lokalen Datenbank in Zugangsdaten eines entfernten Datensystems, das durch einen Foreign Data Wrapper definiert ist.
  - Weitere Informationen finden Sie bei `CREATE USER MAPPING`.

- **UTC**
  - Universal Coordinated Time, die primäre globale Zeitreferenz, ungefähr die am Nullmeridian geltende Zeit. Oft, aber ungenau, als GMT (Greenwich Mean Time) bezeichnet.

- **Vacuum**
  - Der Prozess des Entfernens veralteter Tupelversionen aus Tabellen oder materialisierten Sichten sowie weiterer eng verwandter Verarbeitung, die durch PostgreSQLs MVCC-Implementierung erforderlich ist. Dies kann mit dem Befehl `VACUUM` ausgelöst werden, aber auch automatisch durch Autovacuum-Prozesse erfolgen.
  - Weitere Informationen finden Sie in [Abschnitt 24.1](24_Regelmäßige_Datenbankwartung.md#241-routinemäßiges-vacuuming).

- **View**
  - Eine Relation, die durch eine `SELECT`-Anweisung definiert wird, aber keinen eigenen Speicher besitzt. Immer wenn eine Abfrage eine Sicht referenziert, wird die Definition der Sicht in die Abfrage eingesetzt, als hätte der Benutzer sie als Unterabfrage statt als Namen der Sicht geschrieben.
  - Weitere Informationen finden Sie bei `CREATE VIEW`.

- **Visibility Map (Fork)**
  - Eine Speicherstruktur, die Metadaten zu jeder Datenseite des Main Fork einer Tabelle hält. Der Visibility-Map-Eintrag für jede Seite speichert zwei Bits: Das erste, all-visible, zeigt an, dass alle Tupel auf der Seite für alle Transaktionen sichtbar sind. Das zweite, all-frozen, zeigt an, dass alle Tupel auf der Seite als eingefroren markiert sind.

- **WAL**
  - Siehe Write-Ahead Log.

- **WAL Archiver (Prozess)**
  - Ein Hilfsprozess, der, sofern aktiviert, Kopien von WAL-Dateien speichert, um Backups zu erstellen oder Replicas aktuell zu halten.
  - Weitere Informationen finden Sie in [Abschnitt 25.3](25_Sicherung_und_Wiederherstellung.md#253-kontinuierliche-archivierung-und-pointintimerecovery-pitr).

- **WAL-Datei**
  - Auch WAL-Segment oder WAL-Segmentdatei genannt. Jede der fortlaufend nummerierten Dateien, die Speicherplatz für WAL bereitstellen. Die Dateien haben alle dieselbe vordefinierte Größe und werden der Reihe nach geschrieben, wobei Änderungen aus mehreren gleichzeitigen Sitzungen ineinandergreifen. Wenn das System abstürzt, werden die Dateien der Reihe nach gelesen, und jede Änderung wird wiedergegeben, um den Zustand vor dem Absturz wiederherzustellen.
  - Jede WAL-Datei kann freigegeben werden, nachdem ein Checkpoint alle in ihr enthaltenen Änderungen in die entsprechenden Datendateien geschrieben hat. Das Freigeben kann durch Löschen der Datei oder durch Umbenennen geschehen, sodass sie künftig wiederverwendet wird; dies heißt Recycling.
  - Weitere Informationen finden Sie in [Abschnitt 28.6](28_Zuverlässigkeit_und_Write_Ahead_Log.md#286-walinterna).

- **WAL Record**
  - Eine Low-Level-Beschreibung einer einzelnen Datenänderung. Sie enthält genügend Informationen, um die Datenänderung erneut auszuführen, also wiederzugeben, falls ein Systemausfall dazu führt, dass die Änderung verloren geht. WAL Records verwenden ein nicht druckbares Binärformat.
  - Weitere Informationen finden Sie in [Abschnitt 28.6](28_Zuverlässigkeit_und_Write_Ahead_Log.md#286-walinterna).

- **WAL Receiver (Prozess)**
  - Ein Hilfsprozess, der auf einer Replica läuft, um WAL vom Primary-Server zur Wiedergabe durch den Startup-Prozess zu empfangen.
  - Weitere Informationen finden Sie in [Abschnitt 26.2](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#262-logshippingstandbyserver).

- **WAL-Segment**
  - Siehe WAL-Datei.

- **WAL Sender (Prozess)**
  - Ein besonderer Backend-Prozess, der WAL über ein Netzwerk streamt. Das empfangende Ende kann ein WAL Receiver in einer Replica, `pg_receivewal` oder jedes andere Clientprogramm sein, das das Replikationsprotokoll spricht.

- **WAL Summarizer (Prozess)**
  - Ein Hilfsprozess, der WAL-Daten für inkrementelle Backups zusammenfasst.
  - Weitere Informationen finden Sie in Abschnitt 19.5.7.

- **WAL Writer (Prozess)**
  - Ein Hilfsprozess, der WAL Records aus Shared Memory in WAL-Dateien schreibt.
  - Weitere Informationen finden Sie in [Abschnitt 19.5](19_Serverkonfiguration.md#195-writeahead-log).

- **Window-Funktion (Routine)**
  - Eine Art Funktion, die in einer Abfrage auf eine Partition der Ergebnismenge der Abfrage angewendet wird. Das Ergebnis der Funktion basiert auf Werten, die in Zeilen derselben Partition oder desselben Frames gefunden werden.
  - Alle Aggregatfunktionen können als Window-Funktionen verwendet werden, aber Window-Funktionen können zum Beispiel auch dazu dienen, jeder Zeile in der Partition einen Rang zuzuweisen. Sie heißen auch analytische Funktionen.
  - Weitere Informationen finden Sie in [Abschnitt 3.5](03_Fortgeschrittene_Funktionen.md#35-windowfunktionen).

- **Write-Ahead Log**
  - Das Journal, das die Änderungen im Datenbank-Cluster verfolgt, während von Benutzern oder vom System ausgelöste Operationen stattfinden. Es besteht aus vielen einzelnen WAL Records, die fortlaufend in WAL-Dateien geschrieben werden.
