# Anhang E. Versionshinweise

Die Versionshinweise enthalten die wichtigen Änderungen jedes PostgreSQL-Releases; größere Funktionen und Migrationsfragen stehen jeweils am Anfang. Die Versionshinweise enthalten keine Änderungen, die nur wenige Benutzer betreffen, oder Änderungen, die intern und deshalb für Benutzer nicht sichtbar sind. Der Optimizer wird zum Beispiel in fast jedem Release verbessert, aber Benutzer nehmen diese Verbesserungen gewöhnlich nur als schnellere Abfragen wahr.

Eine vollständige Liste der Änderungen jedes Releases lässt sich den Git-Logs des jeweiligen Releases entnehmen. Die Mailingliste `pgsql-committers` zeichnet ebenfalls alle Quellcodeänderungen auf. Außerdem gibt es ein Webinterface, das Änderungen an bestimmten Dateien zeigt.

Der Name neben jedem Eintrag bezeichnet den wichtigsten Entwickler für diesen Eintrag. Natürlich beruhen alle Änderungen auf Diskussionen in der Community und Patch-Review; jeder Eintrag ist daher tatsächlich eine Gemeinschaftsleistung.

<https://www.postgresql.org/list/pgsql-committers/>
<https://git.postgresql.org/gitweb/?p=postgresql.git;a=summary>

### E.1. Release 18.1

Veröffentlichungsdatum: 2025-11-13

Dieses Release enthält verschiedene Korrekturen gegenüber 18.0. Informationen über neue Funktionen im Hauptrelease 18 finden Sie in [Abschnitt E.2](E_Versionshinweise.md#e2-release-18).

#### E.1.1. Migration auf Version 18.1

Für Installationen, die bereits 18.X ausführen, ist kein Dump/Restore erforderlich.

#### E.1.2. Änderungen

- `CREATE STATISTICS` prüft nun `CREATE`-Privilegien auf dem Schema (Jelte Fennema-Nio)

  Diese ausgelassene Prüfung erlaubte Tabelleneigentümern, Statistiken in beliebigen Schemas anzulegen, was zu unerwarteten Namenskonflikten führen konnte.

  Das PostgreSQL-Projekt dankt Jelte Fennema-Nio für die Meldung dieses Problems. (CVE-2025-12817)

- Integer-Überläufe bei Berechnungen von Allokationsgrößen in `libpq` vermeiden (Jacob Champion)

  An mehreren Stellen in `libpq` wurde die benötigte Größe einer Speicherallokation nicht vorsichtig genug berechnet. Ausreichend große Eingaben konnten Integer-Überläufe auslösen, wodurch ein zu kleiner Puffer entstand und anschließend über dessen Ende hinaus geschrieben wurde.

  Das PostgreSQL-Projekt dankt Aleksey Solovev von Positive Technologies für die Meldung dieses Problems. (CVE-2025-12818)

- Fehler „unrecognized node type“ verhindern, wenn eine SQL/JSON-Funktion wie `JSON_VALUE` eine `DEFAULT`-Klausel mit einem `COLLATE`-Ausdruck enthält (Jian He)

- Falsche Optimierung variablenfreier `HAVING`-Klauseln mit Grouping Sets vermeiden (Richard Guo)

- Parallelität in Hash Right Semi Joins nicht verwenden (Richard Guo)

  Dieser Fall funktioniert wegen einer Race Condition beim Aktualisieren der gemeinsam genutzten Hash-Tabelle des Joins nicht zuverlässig.

- Mögliche Division durch null beim Erstellen von Ordered-Append-Plänen vermeiden (Richard Guo)

  Dieser Fehler konnte zur falschen Auswahl des günstigsten Pfads oder in Debug-Builds zu einem Assertionsfehler führen.

- Planner-Fehler bei Indextypen beheben, die geordneten Zugriff, aber keine Index-Only-Scans unterstützen (Maxime Schoemans)

  Dieses Versehen führte zu Fehlern wie „no data returned for index-only scan“. Bei keinem eingebauten Indextyp tritt dieser Fall auf, aber einige Erweiterungen waren betroffen.

- Fehlerhafte Assertion beim Bereinigen von B-Tree-Indizes entfernen (Peter Geoghegan)

- Mögliche Out-of-Memory- oder „invalid memory alloc request size“-Fehler beim parallelen Aufbau von GIN-Indizes vermeiden (Tomas Vondra)

- Sicherstellen, dass die BRIN-Autosummarization für Indexausdrücke, die einen Snapshot benötigen, einen Snapshot bereitstellt (Álvaro Herrera)

  Bisher schlug die Autosummarization bei solchen Indizes fehl und ließ Platzhalter-Index-Tupel zurück, wodurch der Index mit der Zeit anwuchs.

- Gefahr eines Integer-Überlaufs in BRIN-Indexscans beheben, wenn die Tabelle nahezu 2^32 Seiten enthält (Sunil S)

  Dieses Versehen konnte zu einer Endlosschleife oder zum Scannen nicht benötigter Tabellenseiten führen.

- Falsche Zero-Extension gespeicherter Werte in JIT-generiertem Code zum Zerlegen von Tupeln beheben (David Rowley)

  Ohne JIT verwendet der entsprechende Code Sign-Extension statt Zero-Extension, was zu einer anderen `Datum`-Darstellung kleiner Integer-Datentypen führt. Diese Inkonsistenz blieb in den meisten Fällen verborgen, kann aber bei Verwendung von Memoize-Plan-Knoten zu Fehlern wie „could not find memoization table entry“ führen; weitere Symptome sind möglich.

- Seltenen Absturz beim Verarbeiten gehashter `GROUPING SETS`-Abfragen beheben (David Rowley)

- Fehlerhafte Logik zur Wahl der Hash-Tabellengröße in Hash Joins reparieren (Tomas Vondra)

  Hash Joins verwendeten gelegentlich mehr Speicher als vorgesehen oder teilten ihn nicht effizient auf.

- Logik zur Relationssuche in Statistik-Manipulationsfunktionen verbessern (Nathan Bossart)

  `pg_restore_relation_stats()`, `pg_clear_relation_stats()`, `pg_restore_attribute_stats()` und `pg_clear_attribute_stats()` prüfen Privilegien nun, bevor sie eine Sperre auf der Zielrelation erwerben, statt danach.

- Fehlerhafte Logik beim Caching von Ergebnisrelationsinformationen für Trigger beheben (David Rowley, Amit Langote)

  Wenn die Spaltenmengen von Partitionen physisch nicht mit denen ihrer partitionierten Elterntabellen identisch sind, konnte dieses Versehen zu Abstürzen führen.

- Absturz bei `EvalPlanQual`-Nachprüfungen auf partitionierten Tabellen beheben (David Rowley, Amit Langote)

- `EvalPlanQual`-Behandlung von Foreign oder Custom Joins beheben, für die kein alternativer lokaler Join-Plan für EPQ vorbereitet ist (Masahiko Sawada, Etsuro Fujita)

  In solchen Fällen sollte die Foreign- oder Custom-Access-Methode normal aufgerufen werden; das geschah jedoch nicht und führte typischerweise zu einem Absturz.

- Duplizieren von Hash-Partitions-Constraints während `DETACH CONCURRENTLY` vermeiden (Haiyang Li)

  `ALTER TABLE DETACH PARTITION CONCURRENTLY` fügte bisher der abgetrennten Partition eine Kopie des Partitionierungs-Constraints hinzu. Das war ungünstig: Nicht nebenläufiges `DETACH` tut dies nicht, und bei Hash-Partitionierung enthält der Constraint-Ausdruck Verweise auf die OID der Elterntabelle. Das verursacht Probleme bei Dump/Restore oder wenn die Elterntabelle nach `DETACH` gelöscht wird. Ab Version 19 werden solche kopierten Constraints gar nicht mehr erzeugt. In veröffentlichten Zweigen wird zur Verringerung des Risikos unerwarteter Folgen nur dann auf das Hinzufügen des kopierten Constraints verzichtet, wenn er Hash-Partitionierung betrifft.

- Generierte Spalten in Partitionsschlüsseln verbieten (Jian He, Ashutosh Bapat)

  Dies war bereits nicht erlaubt, aber die Prüfung übersah einige Fälle, etwa wenn der Spaltenverweis implizit in einem Whole-Row-Verweis steckt.

- Generierte Spalten in `COPY ... FROM ... WHERE`-Klauseln verbieten (Peter Eisentraut, Jian He)

  Bisher führte der Versuch, eine solche Spalte zu referenzieren, zu falschem Verhalten oder einer unklaren Fehlermeldung, da generierte Spalten zum Zeitpunkt der `WHERE`-Filterung noch nicht berechnet sind.

- Setzen einer Spalte als Identitätsspalte verhindern, wenn sie einen `NOT NULL`-Constraint hat, dieser aber als ungültig markiert ist (Jian He)

  Identitätsspalten müssen `NOT NULL` sein; die entsprechende Prüfung übersah jedoch diesen Randfall.

- Potenzielles Use-after-free in parallelem Vacuum vermeiden (Kevin Oommen Anish)

  Dieser Fehler scheint in Standard-Builds keine Folgen zu haben, ist theoretisch aber gefährlich.

- Sichtbarkeitsprüfung für Statistikobjekte in `pg_temp` beheben (Noah Misch)

  Ein Statistikobjekt in einem temporären Schema kann nicht ohne Schemaqualifikation benannt werden, aber `pg_statistics_obj_is_visible()` berücksichtigte dies nicht und konnte trotzdem `true` zurückgeben. Dadurch konnten Funktionen wie `pg_describe_object()` den Objektnamen nicht wie erwartet schemaqualifizieren.

- Kleines Speicherleck beim WAL-Replay einer Datenbankerstellung beheben (Nathan Bossart)

- Falsche Meldung des Replikationsrückstands in der Sicht `pg_stat_replication` beheben (Fujii Masao)

  Wenn die Replay-LSN eines Standby-Servers nicht mehr fortschritt, hörten die Spalten `write_lag` und `flush_lag` schließlich auf, aktualisiert zu werden.

- Doppelte Logmeldungen über ungültige `primary_slot_name`-Einstellungen vermeiden (Fujii Masao)

- Fehler vermeiden, wenn `synchronized_standby_slots` auf nicht vorhandene Replikationsslots verweist (Shlok Kyal)

- Unvollständige Slot-Statusdatei entfernen, wenn das Schreiben des Status eines Replikationsslots auf die Platte fehlschlägt (Michael Paquier)

  Bisher blieb bei einem Fehler wie fehlendem Speicherplatz eine temporäre Datei `state.tmp` zurück. Das ist problematisch, weil sie alle späteren Versuche blockiert, den Status zu schreiben, und manuelles Aufräumen erfordert.

- Falsche Behandlung von Lock-Timeout-Signalen in parallelen Apply-Workern der logischen Replikation beheben (Hayato Kuroda)

  Dieselbe Signalnummer wurde sowohl für Worker-Shutdown als auch für Lock-Timeout verwendet, was zu Verwechslungen führte.

- Unerwünschtes Beenden des WAL-Receivers beim Wechsel von Streaming zu Archiv-WAL vermeiden (Xuneng Zhou)

  Während eines Timeline-Wechsels sollte der WAL-Receiver eines Standby-Servers aktiv bleiben und auf einen neuen Startpunkt für WAL-Streaming warten. Stattdessen wurde er wiederholt beendet und sofort neu gestartet, was Statusüberwachungscode verwirren konnte.

- Use-after-free-Problem im Relationssynchronisationscache des Logical-Decoding-Plugins `pgoutput` beheben (Vignesh C, Masahiko Sawada)

  Ein Fehler während der logischen Decodierung konnte bei späteren Decodierungsversuchen in derselben Sitzung zu Abstürzen führen. Der Fall ist nur erreichbar, wenn `pgoutput` über SQL-Funktionen aufgerufen wird.

- Unnötige Invalidierung logischer Replikationsslots vermeiden (Bertrand Drouvot)

- Sonderfall für die `C`-Collation beim Locale-Setup wiederherstellen (Jeff Davis)

  Dies behebt eine Regression beim Zugriff auf gemeinsame Kataloge früh im Backend-Start, bevor eine Datenbank ausgewählt wurde. Für PostgreSQL-Kerncode ist kein Problem bekannt, aber einige Erweiterungen waren betroffen.

- Falsche Ausgabe von Meldungen über Fehler bei der Prüfung beheben, ob der Benutzer Windows-Administratorrechte hat (Bryan Green)

  Dieser Code wäre abgestürzt oder hätte zumindest unsinnige Ausgaben erzeugt. Solche Fälle wurden jedoch nicht gemeldet, was darauf hinweist, dass Fehler dieser Systemaufrufe äußerst selten sind.

- Absturz beim Testen von PostgreSQL mit bestimmten `libsanitizer`-Optionen vermeiden (Emmanuel Sibi, Jacob Champion)

- Falsche Warnungen der Memory-Context-Prüfung in Debug-Builds auf 64-Bit-Windows beheben (David Rowley)

- `GROUP BY DISTINCT` in PL/pgSQL-Zuweisungsanweisungen korrekt behandeln (Tom Lane)

  Der Parser zeichnete die Option `DISTINCT` in diesem Kontext nicht auf, sodass sich der Befehl wie ein einfaches `GROUP BY` verhielt.

- Speicherleck beim Behandeln eines SQL-Fehlers innerhalb von PL/Python vermeiden (Tom Lane)

  Dies behebt ein Speicherleck über die Lebensdauer der Sitzung, das in früheren Minor-Releases eingeführt wurde.

- Behandlung socketbezogener Fehler durch `libpq` unter Windows in der GSSAPI-Logik beheben (Ning Wu, Tom Lane)

  Der Code zum Ver- und Entschlüsseln übertragener Daten mit GSSAPI erkannte Fehlerbedingungen am Verbindungssocket nicht korrekt, weil Windows diese anders meldet als andere Plattformen. Dadurch konnten solche Verbindungen unter Windows fehlschlagen.

- Dumping nicht geerbter `NOT NULL`-Constraints auf geerbten Tabellenspalten beheben (Dilip Kumar)

  `pg_dump` bewahrte solche Constraints beim Dump aus einem Server vor Version 18 nicht.

- Sortierung von Fremdschlüssel-Constraints durch `pg_dump` beheben (Álvaro Herrera)

  Damit wird eine konsistente Reihenfolge dieser Datenbankobjekte sichergestellt, wie sie bereits für andere Objekttypen verwendet wurde.

- Verschiedene Fehler in der Datenkompressionslogik von `pg_dump` und `pg_restore` beheben (Daniel Gustafsson, Tom Lane)

  An mehreren Stellen fehlte die Fehlerprüfung oder sie war falsch; außerdem gab es Portabilitätsprobleme, die sich auf Big-Endian-Hardware zeigten. Diese Probleme waren übersehen worden, weil dieser Code nur zum Lesen komprimierter TOC-Dateien in Dumps im Verzeichnisformat verwendet wird. `pg_dump` erzeugt solche Dumps nie; der Fall ist nur erreichbar, wenn die TOC-Datei nachträglich manuell komprimiert wird. Das wird unterstützt, ist aber sehr ungewöhnlich.

- `pgbench` so korrigieren, dass es sauber mit einem Fehler beendet, wenn eine `COPY`-Operation gestartet wird (Anthonin Bonnefoy)

  `pgbench` soll diesen Fall nicht unterstützen, geriet bisher aber in eine Endlosschleife.

- Meldung mehrerer Fehler durch `pgbench` beheben (Yugo Nagata)

  Wenn zwei aufeinanderfolgende Aufrufe von `PQgetResult` fehlschlugen, konnte `pgbench` die falsche Fehlermeldung ausgeben.

- In `pgbench` fehlerhafte Assertion zu Fehlern im Pipeline-Modus beheben (Yugo Nagata)

- Speicherleck pro Datei in `pg_combinebackup` beheben (Tom Lane)

- Sicherstellen, dass Funktionen von `contrib/pg_buffercache` abgebrochen werden können (Satyanarayana Narlapuram, Yuhang Qiu)

  Einige Codepfade konnten lange laufen, ohne auf Interrupts zu prüfen.

- Privilegprüfungen von `contrib/pg_prewarm` für Indizes beheben (Ayush Vatsa, Nathan Bossart)

  `pg_prewarm()` verlangt `SELECT`-Privilegien auf vorzuwärmenden Relationen. Da Indizes jedoch keine eigenen SQL-Privilegien haben, konnten Nicht-Superuser Indizes nicht vorwärmen. Stattdessen wird nun das `SELECT`-Privileg auf der Tabelle des Index geprüft.

- In `contrib/pg_stat_statements` Absturz vermeiden, wenn zwei oder mehr Konstanten mit derselben Position im SQL-Anweisungstext markiert sind (Sami Imseih, Dmitry Dolgov)

- `contrib/pgstattuple` robuster gegenüber leeren oder ungültigen Indexseiten machen (Nitin Motiani)

  Seiten, die nur Nullen enthalten, werden als freier Speicher gezählt, und Seiten, die nach Prüfung der Größe ihres Special Space ungültig sind, werden ignoriert. Der Code für B-Tree-Indizes zählte Nullseiten bereits als frei, während der Hash- und GiST-Code einen Fehler ausgab, was sich als deutlich weniger benutzerfreundlich erwiesen hat. Entsprechend ignorieren nun alle drei Fälle beschädigte Seiten, statt Fehler zu werfen.

- Lese- und Schreibbarrieren-Makros härten, um Clang zufriedenzustellen (Thomas Munro)

  Es wurde angenommen, dass `__atomic_thread_fence()` eine ausreichende Barriere ist, um den C-Compiler daran zu hindern, Speicherzugriffe darum herum neu zu ordnen. Für Clang scheint das nicht zu stimmen; dadurch konnte zumindest für RISC-V-, MIPS- und LoongArch-Maschinen falscher Code erzeugt werden. Explizite Compiler-Barrieren beheben dies.

- PGXS-Build-Infrastruktur so korrigieren, dass NLS-`po`-Dateien für Erweiterungen gebaut werden können (Ryo Matsumura)

### E.2. Release 18

Veröffentlichungsdatum: 2025-09-25

#### E.2.1. Überblick

PostgreSQL 18 enthält viele neue Funktionen und Verbesserungen, darunter:

- Ein asynchrones I/O-Subsystem (AIO), das die Performance von sequenziellen Scans, Bitmap-Heap-Scans, Vacuums und anderen Operationen verbessern kann.

- `pg_upgrade` erhält nun Optimizer-Statistiken.

- Unterstützung für „Skip Scan“-Suchen, mit denen mehrspaltige B-Tree-Indizes in mehr Fällen genutzt werden können.

- Die Funktion `uuidv7()` zum Erzeugen zeitstempelgeordneter UUIDs.

- Virtuelle generierte Spalten, die ihre Werte beim Lesen berechnen. Dies ist nun die Voreinstellung für generierte Spalten.

- Unterstützung für OAuth-Authentifizierung.

- Unterstützung für `OLD` und `NEW` in `RETURNING`-Klauseln von `INSERT`-, `UPDATE`-, `DELETE`- und `MERGE`-Befehlen.

- Temporale Constraints beziehungsweise Constraints über Bereiche für `PRIMARY KEY`-, `UNIQUE`- und `FOREIGN KEY`-Constraints.

Die oben genannten Punkte und weitere neue Funktionen von PostgreSQL 18 werden in den folgenden Abschnitten ausführlicher erläutert.

#### E.2.2. Migration auf Version 18

Wer Daten aus einer früheren Version migrieren möchte, muss einen Dump/Restore mit `pg_dumpall`, `pg_upgrade` oder logische Replikation verwenden. Allgemeine Informationen zur Migration auf neue Hauptversionen finden Sie in [Abschnitt 18.6](18_Servereinrichtung_und_Betrieb.md#186-einen-postgresqlcluster-aktualisieren).

Version 18 enthält mehrere Änderungen, die die Kompatibilität mit früheren Versionen beeinflussen können. Beachten Sie die folgenden Inkompatibilitäten:

- Voreinstellung von `initdb` ändern, sodass Datenprüfsummen aktiviert werden (Greg Sabino Mullane)

  Prüfsummen können mit der neuen `initdb`-Option `--no-data-checksums` deaktiviert werden. `pg_upgrade` verlangt übereinstimmende Prüfsummeneinstellungen der Cluster; daher kann diese neue Option nützlich sein, um alte Cluster ohne Prüfsummen zu aktualisieren.

- Behandlung von Zeitzonenabkürzungen ändern (Tom Lane)

  Das System bevorzugt nun die Zeitzonenabkürzungen der aktuellen Sitzung, bevor die Servervariable `timezone_abbreviations` geprüft wird. Zuvor wurde `timezone_abbreviations` zuerst geprüft.

- MD5-Passwortauthentifizierung als veraltet kennzeichnen (Nathan Bossart)

  Die Unterstützung für MD5-Passwörter wird in einer zukünftigen Hauptversion entfernt. `CREATE ROLE` und `ALTER ROLE` geben nun Veraltungswarnungen aus, wenn MD5-Passwörter gesetzt werden. Diese Warnungen können deaktiviert werden, indem der Parameter `md5_password_warnings` auf `off` gesetzt wird.

- `VACUUM` und `ANALYZE` so ändern, dass sie Vererbungskinder eines Elternobjekts verarbeiten (Michael Harris)

  Das frühere Verhalten kann mit der neuen Option `ONLY` erreicht werden.

- Verhindern, dass `COPY FROM` beim Lesen von CSV-Dateien `\.` als Dateiende-Markierung behandelt (Daniel Vérité, Tom Lane)

  `psql` behandelt `\.` weiterhin als Dateiende-Markierung, wenn CSV-Dateien von `STDIN` gelesen werden. Ältere `psql`-Clients, die sich mit PostgreSQL-18-Servern verbinden, können Probleme mit `\copy` erfahren. Dieses Release erzwingt außerdem, dass `\.` allein auf einer Zeile stehen muss.

- Unlogged partitionierte Tabellen verbieten (Michael Paquier)

  Bisher tat `ALTER TABLE SET [UN]LOGGED` nichts, und das Erstellen einer unlogged partitionierten Tabelle führte nicht dazu, dass ihre Kinder ebenfalls unlogged wurden.

- `AFTER`-Trigger als die Rolle ausführen, die aktiv war, als die Triggerereignisse eingereiht wurden (Laurenz Albe)

  Bisher wurden solche Trigger als die Rolle ausgeführt, die zum Ausführungszeitpunkt des Triggers aktiv war, zum Beispiel bei `COMMIT`. Das ist wichtig für Fälle, in denen die Rolle zwischen dem Einreihen und dem Commit der Transaktion geändert wird.

- Nicht funktionsfähige Unterstützung für Regelprivilegien in `GRANT`/`REVOKE` entfernen (Fujii Masao)

  Diese funktionieren seit PostgreSQL 8.2 nicht mehr.

- Spalte `pg_backend_memory_contexts.parent` entfernen (Melih Mutlu)

  Sie wird nicht mehr benötigt, seit `pg_backend_memory_contexts.path` hinzugefügt wurde.

- `pg_backend_memory_contexts.level` und `pg_log_backend_memory_contexts()` auf einsbasierte Zählung umstellen (Melih Mutlu, Atsushi Torikoshi, David Rowley, Fujii Masao)

  Diese waren zuvor nullbasiert.

- Volltextsuche so ändern, dass zum Lesen von Konfigurationsdateien und Wörterbüchern der Standard-Collation-Provider des Clusters verwendet wird, statt immer `libc` zu nutzen (Peter Eisentraut)

  Cluster, die standardmäßig Nicht-`libc`-Collation-Provider wie ICU oder `builtin` verwenden und sich bei durch `LC_CTYPE` verarbeiteten Zeichen anders verhalten als `libc`, können Verhaltensänderungen bei einigen Volltextsuchfunktionen sowie bei der Erweiterung `pg_trgm` beobachten. Beim Upgrade solcher Cluster mit `pg_upgrade` wird empfohlen, nach dem Upgrade alle Indizes neu zu indizieren, die mit Volltextsuche und `pg_trgm` zusammenhängen.

#### E.2.3. Änderungen

Im Folgenden finden Sie eine detaillierte Darstellung der Änderungen zwischen PostgreSQL 18 und der vorherigen Hauptversion.

##### E.2.3.1. Server

###### E.2.3.1.1. Optimizer

- Einige unnötige Self-Joins von Tabellen automatisch entfernen (Andrey Lepikhov, Alexander Kuzmenkov, Alexander Korotkov, Alena Rybakina)

  Diese Optimierung kann mit der Servervariable `enable_self_join_elimination` deaktiviert werden.

- Einige `IN (VALUES ...)`-Ausdrücke für bessere Optimizer-Statistiken in `x = ANY ...` umwandeln (Alena Rybakina, Andrei Lepikhov)

- Umwandlung von `OR`-Klauseln in Arrays für schnellere Indexverarbeitung erlauben (Alexander Korotkov, Andrey Lepikhov)

- Verarbeitung von `INTERSECT`, `EXCEPT`, Window-Aggregaten und Spaltenaliasen von Sichten beschleunigen (Tom Lane, David Rowley)

- Interne Umordnung der Schlüssel von `SELECT DISTINCT` erlauben, um Sortierung zu vermeiden (Richard Guo)

  Diese Optimierung kann mit `enable_distinct_reordering` deaktiviert werden.

- `GROUP BY`-Spalten ignorieren, die funktional von anderen Spalten abhängen (Zhang Mingli, Jian He, David Rowley)

  Wenn eine `GROUP BY`-Klausel alle Spalten eines eindeutigen Index sowie weitere Spalten derselben Tabelle enthält, sind diese weiteren Spalten redundant und können aus der Gruppierung entfernt werden. Für nicht zurückstellbare Primärschlüssel galt dies bereits.

- Einige `HAVING`-Klauseln auf `GROUPING SETS` in `WHERE`-Klauseln verschieben lassen (Richard Guo)

  Dadurch können Zeilen früher gefiltert werden. Dieses Release behebt außerdem einige `GROUPING SETS`-Abfragen, die zuvor falsche Ergebnisse lieferten.

- Zeilenschätzungen für `generate_series()` mit `numeric`- und Zeitstempelwerten verbessern (David Rowley, Song Jinzhou)

- Dem Optimizer die Verwendung von Right-Semi-Join-Plänen erlauben (Richard Guo)

  Semi-Joins werden verwendet, wenn geprüft werden muss, ob mindestens ein Treffer existiert.

- Merge Joins erlauben, inkrementelle Sortierungen zu verwenden (Richard Guo)

- Effizienz beim Planen von Abfragen verbessern, die auf viele Partitionen zugreifen (Ashutosh Bapat, Yuya Watari, David Rowley)

- Partitionwise Joins in mehr Fällen erlauben und ihren Speicherverbrauch senken (Richard Guo, Tom Lane, Ashutosh Bapat)

- Kostenschätzungen für Partitionsabfragen verbessern (Nikita Malakhov, Andrei Lepikhov)

- Plan-Caching für Funktionen in SQL-Sprache verbessern (Alexander Pyhalov, Tom Lane)

- Behandlung deaktivierter Optimizer-Funktionen verbessern (Robert Haas)

###### E.2.3.1.2. Indexes

- Skip Scans von B-Tree-Indizes erlauben (Peter Geoghegan)

  Dadurch können mehrspaltige B-Tree-Indizes in mehr Fällen genutzt werden, etwa wenn es keine Einschränkungen für die erste oder frühe indizierte Spalten gibt oder dort Ungleichheitsbedingungen stehen, aber nützliche Einschränkungen für spätere indizierte Spalten vorhanden sind.

- Nicht-B-Tree-Unique-Indizes als Partitionsschlüssel und in materialisierten Sichten erlauben (Mark Dilger)

  Der Indextyp muss weiterhin Gleichheit unterstützen.

- Paralleles Erstellen von GIN-Indizes erlauben (Tomas Vondra, Matthias van de Meent)

- Sortieren von Werten erlauben, um GiST-Indizes für Range-Typen und B-Tree-Indexaufbau zu beschleunigen (Bernd Helmle)

###### E.2.3.1.3. General Performance

- Ein asynchrones I/O-Subsystem hinzufügen (Andres Freund, Thomas Munro, Nazir Bilal Yavuz, Melanie Plageman)

  Diese Funktion erlaubt Backends, mehrere Leseanforderungen einzureihen, was effizientere sequenzielle Scans, Bitmap-Heap-Scans, Vacuums usw. ermöglicht. Aktiviert wird dies über die Servervariable `io_method`; zur Steuerung wurden die Servervariablen `io_combine_limit` und `io_max_combine_limit` hinzugefügt. Außerdem ermöglicht dies Werte größer null für `effective_io_concurrency` und `maintenance_io_concurrency` auf Systemen ohne `fadvise()`-Unterstützung. Die neue Systemsicht `pg_aios` zeigt die für asynchrones I/O verwendeten Datei-Handles.

- Locking-Performance von Abfragen verbessern, die auf viele Relationen zugreifen (Tomas Vondra)

- Performance verbessern und Speicherverbrauch von Hash Joins und `GROUP BY` senken (David Rowley, Jeff Davis)

  Dies verbessert außerdem Hash-Set-Operationen, die von `EXCEPT` verwendet werden, sowie Hash-Lookups von Subplan-Werten.

- Normalem Vacuum erlauben, einige Seiten einzufrieren, obwohl sie all-visible sind (Melanie Plageman)

  Dies verringert den Aufwand späteren Einfrierens der gesamten Relation. Die Aggressivität kann mit der Servervariable und tabellenspezifischen Einstellung `vacuum_max_eager_freeze_failure_rate` gesteuert werden. Bisher verarbeitete Vacuum all-visible Seiten nie, bis Einfrieren erforderlich war.

- Servervariable `vacuum_truncate` hinzufügen, um Dateikürzung während `VACUUM` zu steuern (Nathan Bossart, Gurjeet Singh)

  Ein Storage-Level-Parameter mit demselben Namen und Verhalten existierte bereits.

- Voreinstellungen der Servervariablen `effective_io_concurrency` und `maintenance_io_concurrency` auf 16 erhöhen (Melanie Plageman)

  Dies spiegelt moderne Hardware genauer wider.

###### E.2.3.1.4. Monitoring

- Granularität der Protokollierung der Servervariable `log_connections` erhöhen (Melanie Plageman)

  Diese Servervariable war bisher nur boolesch; das wird weiterhin unterstützt.

- Option `log_connections` hinzufügen, um die Dauer von Verbindungsphasen zu melden (Melanie Plageman)

- Escape `%L` für `log_line_prefix` hinzufügen, um die Client-IP-Adresse auszugeben (Greg Sabino Mullane)

- Servervariable `log_lock_failures` hinzufügen, um fehlgeschlagene Lock-Erwerbe zu protokollieren (Yuki Seino, Fujii Masao)

  Insbesondere meldet sie fehlgeschlagene `SELECT ... NOWAIT`-Locks.

- `pg_stat_all_tables` und Varianten ändern, damit die in `VACUUM`, `ANALYZE` und deren automatischen Varianten verbrachte Zeit gemeldet wird (Sami Imseih)

  Die neuen Spalten sind `total_vacuum_time`, `total_autovacuum_time`, `total_analyze_time` und `total_autoanalyze_time`.

- Meldung von Verzögerungszeiten zu `VACUUM` und `ANALYZE` hinzufügen (Bertrand Drouvot, Nathan Bossart)

  Diese Information erscheint im Serverlog, in den Systemsichten `pg_stat_progress_vacuum` und `pg_stat_progress_analyze` sowie in der Ausgabe von `VACUUM` und `ANALYZE` im `VERBOSE`-Modus; die Erfassung muss mit der Servervariable `track_cost_delay_timing` aktiviert werden.

- Ausgabe von WAL-, CPU- und durchschnittlichen Lesestatistiken zu `ANALYZE VERBOSE` hinzufügen (Anthonin Bonnefoy)

- Vollständige WAL-Pufferzählung zu `VACUUM`/`ANALYZE (VERBOSE)` und zur Autovacuum-Logausgabe hinzufügen (Bertrand Drouvot)

- Meldung von I/O-Statistiken pro Backend hinzufügen (Bertrand Drouvot)

  Auf die Statistiken wird über `pg_stat_get_backend_io()` zugegriffen. I/O-Statistiken pro Backend können mit `pg_stat_reset_backend_stats()` zurückgesetzt werden.

- Spalten zu `pg_stat_io` hinzufügen, um I/O-Aktivität in Bytes zu melden (Nazir Bilal Yavuz)

  Die neuen Spalten sind `read_bytes`, `write_bytes` und `extend_bytes`. Die Spalte `op_bytes`, die immer `BLCKSZ` entsprach, wurde entfernt.

- Zeilen für WAL-I/O-Aktivität zu `pg_stat_io` hinzufügen (Nazir Bilal Yavuz, Bertrand Drouvot, Michael Paquier)

  Dies umfasst WAL-Receiver-Aktivität und ein Warteereignis für solche Schreibvorgänge.

- Servervariable `track_wal_io_timing` ändern, sodass sie WAL-Timing in `pg_stat_io` statt in `pg_stat_wal` steuert (Bertrand Drouvot)

- Read-/Sync-Spalten aus `pg_stat_wal` entfernen (Bertrand Drouvot)

  Dadurch werden die Spalten `wal_write`, `wal_sync`, `wal_write_time` und `wal_sync_time` entfernt.

- Funktion `pg_stat_get_backend_wal()` hinzufügen, um WAL-Statistiken pro Backend zurückzugeben (Bertrand Drouvot)

  WAL-Statistiken pro Backend können mit `pg_stat_reset_backend_stats()` zurückgesetzt werden.

- Funktion `pg_ls_summariesdir()` hinzufügen, um gezielt den Inhalt von `PGDATA/pg_wal/summaries` aufzulisten (Yushi Ogiwara)

- Spalte `pg_stat_checkpointer.num_done` hinzufügen, um die Anzahl abgeschlossener Checkpoints zu melden (Anton A. Melnikov)

  Die Spalten `num_timed` und `num_requested` zählen sowohl abgeschlossene als auch übersprungene Checkpoints.

- Spalte `pg_stat_checkpointer.slru_written` hinzufügen, um geschriebene SLRU-Puffer zu melden (Nitin Jadhav)

  Außerdem wird die Checkpoint-Serverlogmeldung geändert, um getrennte Werte für Shared Buffer und SLRU-Puffer zu melden.

- Spalten zu `pg_stat_database` hinzufügen, um Aktivität paralleler Worker zu melden (Benoit Lobréau)

  Die neuen Spalten sind `parallel_workers_to_launch` und `parallel_workers_launched`.

- Bei der Berechnung von Query-IDs für Konstantenlisten nur die erste und letzte Konstante berücksichtigen (Dmitry Dolgov, Sami Imseih)

  Das Jumbling wird von `pg_stat_statements` verwendet.

- Berechnung von Query-IDs anpassen, um Abfragen mit demselben Relationsnamen zusammenzufassen (Michael Paquier, Sami Imseih)

  Dies gilt auch dann, wenn die Tabellen in verschiedenen Schemas unterschiedliche Spaltennamen haben.

- Spalte `pg_backend_memory_contexts.type` hinzufügen, um den Typ des Memory Context zu melden (David Rowley)

- Spalte `pg_backend_memory_contexts.path` hinzufügen, um Eltern von Memory Contexts anzuzeigen (Melih Mutlu)

###### E.2.3.1.5. Privileges

- Funktion `pg_get_acl()` hinzufügen, um Details der Datenbankzugriffskontrolle abzurufen (Joel Jacobson)

- Funktion `has_largeobject_privilege()` hinzufügen, um Privilegien für Large Objects zu prüfen (Yugo Nagata)

- `ALTER DEFAULT PRIVILEGES` erlauben, Standardprivilegien für Large Objects zu definieren (Takatsuka Haruka, Yugo Nagata, Laurenz Albe)

- Vordefinierte Rolle `pg_signal_autovacuum_worker` hinzufügen (Kirill Reshke)

  Damit können Signale an Autovacuum-Worker gesendet werden.

###### E.2.3.1.6. Server Configuration

- Unterstützung für die OAuth-Authentifizierungsmethode hinzufügen (Jacob Champion, Daniel Gustafsson, Thomas Munro)

  Dies fügt eine `oauth`-Authentifizierungsmethode für `pg_hba.conf`, OAuth-Optionen für `libpq`, eine Servervariable `oauth_validator_libraries` zum Laden von Token-Validierungsbibliotheken und ein Configure-Flag `--withlibcurl` für die benötigten Compilezeit-Bibliotheken hinzu.

- Servervariable `ssl_tls13_ciphers` hinzufügen, um mehrere durch Doppelpunkte getrennte TLSv1.3-Cipher-Suites anzugeben (Erica Zhang, Daniel Gustafsson)

- Voreinstellung der Servervariable `ssl_groups` ändern, sodass die elliptische Kurve X25519 enthalten ist (Daniel Gustafsson, Jacob Champion)

- Rename server variable ssl_ecdh_curve to ssl_groups and allow multiple colon-separated ECDH curves to be specified (Erica Zhang, Daniel Gustafsson)

  The previous name still works.

- Cancel-Request-Schlüssel auf 256 Bit setzen (Heikki Linnakangas, Jelte Fennema-Nio)

  Dies ist nur möglich, wenn Server und Client die in diesem Release eingeführte Wire-Protokollversion 3.2 unterstützen.

- Servervariable `autovacuum_worker_slots` hinzufügen, um die maximale Anzahl von Background Workern festzulegen (Nathan Bossart)

  Wenn diese Variable gesetzt ist, kann `autovacuum_max_workers` zur Laufzeit bis zu diesem Maximum ohne Serverneustart angepasst werden.

- Angabe einer festen Anzahl toter Tupel erlauben, die ein Autovacuum auslöst (Nathan Bossart, Frédéric Yhuel)

  Die Servervariable ist `autovacuum_vacuum_max_threshold`. Prozentwerte werden weiterhin zum Auslösen verwendet.

- Servervariable `max_files_per_process` so ändern, dass nur von einem Backend geöffnete Dateien begrenzt werden (Andres Freund)

  Zuvor wurden auch vom Postmaster geöffnete Dateien auf dieses Limit angerechnet.

- Servervariable `num_os_semaphores` hinzufügen, um die benötigte Anzahl von Semaphoren zu melden (Nathan Bossart)

  Das ist nützlich für die Betriebssystemkonfiguration.

- Servervariable `extension_control_path` hinzufügen, um den Speicherort von Extension-Control-Dateien festzulegen (Peter Eisentraut, Matheus Alcantara)

###### E.2.3.1.7. Streaming Replication and Recovery

- Automatisches Invalidieren inaktiver Replikationsslots über die Servervariable `idle_replication_slot_timeout` erlauben (Nisha Moond, Bharath Rupireddy)

- Servervariable `max_active_replication_origins` hinzufügen, um die maximale Anzahl aktiver Replikationsursprünge zu steuern (Euler Taveira)

  Dies wurde bisher durch `max_replication_slots` gesteuert; die neue Einstellung erlaubt jedoch eine höhere Anzahl von Ursprüngen, wenn weniger Slots benötigt werden.

###### E.2.3.1.8. Logical Replication

- Werte generierter Spalten logisch replizieren lassen (Shubham Khanna, Vignesh C, Zhijie Hou, Shlok Kyal, Peter Smith)

  Wenn die Publication eine Spaltenliste angibt, werden alle angegebenen Spalten, generierte und nicht generierte, veröffentlicht. Ohne Spaltenliste steuert die Publication-Option `publish_generated_columns`, ob generierte Spalten veröffentlicht werden. Bisher wurden generierte Spalten nicht repliziert, und der Subscriber musste die Werte nach Möglichkeit selbst berechnen; das ist besonders nützlich für Nicht-PostgreSQL-Subscriber, denen diese Fähigkeit fehlt.

- Voreinstellung der Streaming-Option von `CREATE SUBSCRIPTION` von `off` auf `parallel` ändern (Vignesh C)

- `ALTER SUBSCRIPTION` erlauben, das Two-Phase-Commit-Verhalten des Replikationsslots zu ändern (Hayato Kuroda, Ajin Cherian, Amit Kapila, Zhijie Hou)

- Konflikte beim Anwenden logischer Replikationsänderungen protokollieren (Zhijie Hou, Nisha Moond)

  Außerdem werden sie in neuen Spalten von `pg_stat_subscription_stats` gemeldet.

##### E.2.3.2. Hilfsbefehle

- Generierte Spalten als virtuell erlauben und zur Voreinstellung machen (Peter Eisentraut, Jian He, Richard Guo, Dean Rasheed)

  Virtuelle generierte Spalten erzeugen ihre Werte beim Lesen der Spalten, nicht beim Schreiben. Das Schreibverhalten kann weiterhin über die Option `STORED` angegeben werden.

- Unterstützung für `OLD`/`NEW` in `RETURNING` bei DML-Abfragen hinzufügen (Dean Rasheed)

  Bisher gab `RETURNING` bei `INSERT` und `UPDATE` nur neue Werte und bei `DELETE` alte Werte zurück; `MERGE` gab den passenden Wert der intern ausgeführten Abfrage zurück. Die neue Syntax erlaubt der `RETURNING`-Liste von `INSERT`/`UPDATE`/`DELETE`/`MERGE`, alte und neue Werte ausdrücklich über die besonderen Aliase `old` und `new` zurückzugeben. Diese Aliase können umbenannt werden, um Bezeichnerkonflikte zu vermeiden.

- Erstellen von Fremdtabellen wie vorhandene lokale Tabellen erlauben (Zhang Mingli)

  Die Syntax lautet `CREATE FOREIGN TABLE ... LIKE`.

- `LIKE` mit nichtdeterministischen Collations erlauben (Peter Eisentraut)

- Textpositions-Suchfunktionen mit nichtdeterministischen Collations erlauben (Peter Eisentraut)

  Diese erzeugten bisher einen Fehler.

- Eingebauten Collation-Provider `PG_UNICODE_FAST` hinzufügen (Jeff Davis)

  Diese Locale unterstützt Case Mapping, sortiert aber in Codepoint-Reihenfolge, nicht in natürlicher Sprachreihenfolge.

- `VACUUM` und `ANALYZE` erlauben, partitionierte Tabellen zu verarbeiten, ohne ihre Kinder zu verarbeiten (Michael Harris)

  Dies wird mit der neuen Option `ONLY` aktiviert. Das ist nützlich, weil Autovacuum partitionierte Tabellen nicht verarbeitet, sondern nur deren Kinder.

- Funktionen zum Ändern von Optimizer-Statistiken pro Relation und pro Spalte hinzufügen (Corey Huinker)

  Die Funktionen sind `pg_restore_relation_stats()`, `pg_restore_attribute_stats()`, `pg_clear_relation_stats()` und `pg_clear_attribute_stats()`.

- Servervariable `file_copy_method` hinzufügen, um die Methode zum Kopieren von Dateien zu steuern (Nazir Bilal Yavuz)

  Dies steuert, ob `CREATE DATABASE ... STRATEGY=FILE_COPY` und `ALTER DATABASE ... SET TABLESPACE` Dateikopien oder Klone verwenden.

###### E.2.3.2.1. Constraints

- Angabe nicht überlappender `PRIMARY KEY`-, `UNIQUE`- und Fremdschlüssel-Constraints erlauben (Paul A. Jungwirth)

  Dies wird mit `WITHOUT OVERLAPS` für `PRIMARY KEY` und `UNIQUE` sowie mit `PERIOD` für Fremdschlüssel angegeben; jeweils angewendet auf die zuletzt angegebene Spalte.

- `CHECK`- und Fremdschlüssel-Constraints als `NOT ENFORCED` angeben lassen (Amul Sul)

  Dies fügt außerdem die Spalte `pg_constraint.conenforced` hinzu.

- Für Primär-/Fremdschlüsselbeziehungen entweder deterministische Collations oder dieselben nichtdeterministischen Collations verlangen (Peter Eisentraut)

  Das Restore eines `pg_dump`, das auch von `pg_upgrade` verwendet wird, schlägt fehl, wenn diese Anforderungen nicht erfüllt sind; für erfolgreiche Upgrades mit diesen Methoden sind Schemaänderungen nötig.

- Spaltenspezifikationen für `NOT NULL` in `pg_constraint` speichern (Álvaro Herrera, Bernd Helmle)

  Dadurch können Namen für `NOT NULL`-Constraints angegeben werden. Außerdem werden `NOT NULL`-Constraints für Fremdtabellen und Vererbungssteuerung für `NOT NULL` bei lokalen Tabellen hinzugefügt.

- `ALTER TABLE` erlauben, das Attribut `NOT VALID` von `NOT NULL`-Constraints zu setzen (Rushabh Lathia, Jian He)

- Änderung der Vererbbarkeit von `NOT NULL`-Constraints erlauben (Suraj Kharage, Álvaro Herrera)

  Die Syntax lautet `ALTER TABLE ... ALTER CONSTRAINT ... [NO] INHERIT`.

- `NOT VALID`-Fremdschlüssel-Constraints auf partitionierten Tabellen erlauben (Amul Sul)

- Löschen von Constraints mit `ONLY` auf partitionierten Tabellen erlauben (Álvaro Herrera)

  Dies war zuvor fälschlicherweise verboten.

###### E.2.3.2.2. COPY

- `REJECT_LIMIT` hinzufügen, um die Anzahl ungültiger Zeilen zu steuern, die `COPY FROM` ignorieren darf (Atsushi Torikoshi)

  Dies ist verfügbar, wenn `ON_ERROR = 'ignore'` gesetzt ist.

- `COPY TO` erlauben, Zeilen aus befüllten materialisierten Sichten zu kopieren (Jian He)

- `COPY LOG_VERBOSITY`-Stufe `silent` hinzufügen, um Logausgaben ignorierter Zeilen zu unterdrücken (Atsushi Torikoshi)

  Diese neue Stufe unterdrückt Ausgaben für verworfene Eingabezeilen, wenn `on_error = 'ignore'` gesetzt ist.

- `COPY FREEZE` auf Fremdtabellen verbieten (Nathan Bossart)

  Bisher funktionierte `COPY`, aber `FREEZE` wurde ignoriert; daher wird dieser Befehl verboten.

###### E.2.3.2.3. EXPLAIN

- `BUFFERS`-Ausgabe automatisch in `EXPLAIN ANALYZE` einschließen (Guillaume Lelarge, David Rowley)

- Vollständige WAL-Pufferzählung zur Ausgabe von `EXPLAIN (WAL)` hinzufügen (Bertrand Drouvot)

- In `EXPLAIN ANALYZE` die Anzahl der verwendeten Index-Lookups pro Index-Scan-Knoten melden (Peter Geoghegan)

- `EXPLAIN` so ändern, dass gebrochene Zeilenzahlen ausgegeben werden (Ibrar Ahmed, Ilia Evdokimov, Robert Haas)

- Details zu Speicher- und Plattennutzung für `Material`-, `Window Aggregate`- und Common-Table-Expression-Knoten zur `EXPLAIN`-Ausgabe hinzufügen (David Rowley, Tatsuo Ishii)

- Details zu Argumenten von Window-Funktionen zur `EXPLAIN`-Ausgabe hinzufügen (Tom Lane)

- Worker-Cache-Statistiken für `Parallel Bitmap Heap Scan` zu `EXPLAIN ANALYZE` hinzufügen (David Geier, Heikki Linnakangas, Donghang Lin, Alena Rybakina, David Rowley)

- Deaktivierte Knoten in der Ausgabe von `EXPLAIN ANALYZE` kennzeichnen (Robert Haas, David Rowley, Laurenz Albe)

##### E.2.3.3. Datentypen

- Vollständiges Unicode-Case-Mapping und Konvertierung verbessern (Jeff Davis)

  Dies fügt die Fähigkeit hinzu, bedingtes und Title-Case-Mapping durchzuführen und einzelne Zeichen auf mehrere Zeichen abzubilden.

- Casts von `jsonb`-Nullwerten zu skalaren Typen als `NULL` erlauben (Tom Lane)

  Bisher erzeugten solche Casts einen Fehler.

- Optionalen Parameter zu `json{b}_strip_nulls` hinzufügen, um das Entfernen von Null-Array-Elementen zu erlauben (Florents Tselai)

- Funktion `array_sort()` hinzufügen, die die erste Dimension eines Arrays sortiert (Junwang Zhao, Jian He)

- Funktion `array_reverse()` hinzufügen, die die erste Dimension eines Arrays umkehrt (Aleksander Alekseev)

- Funktion `reverse()` hinzufügen, um `bytea`-Bytes umzukehren (Aleksander Alekseev)

- Casts zwischen Integer-Typen und `bytea` erlauben (Aleksander Alekseev)

  Die Integer-Werte werden als `bytea`-Werte im Zweierkomplement gespeichert.

- Unicode-Daten auf Unicode 16.0.0 aktualisieren (Peter Eisentraut)

- Stemming für Estnisch in der Volltextsuche hinzufügen (Tom Lane)

- XML-Fehlercodes enger an den SQL-Standard anpassen (Tom Lane)

  Diese Fehler werden über `SQLSTATE` gemeldet.

##### E.2.3.4. Funktionen

- Funktion `casefold()` hinzufügen, um anspruchsvollere Vergleiche ohne Beachtung der Groß-/Kleinschreibung zu ermöglichen (Jeff Davis)

  Dies erlaubt genauere Vergleiche, etwa wenn ein Zeichen mehrere Groß- oder Kleinschreibungsäquivalente haben kann oder die Umwandlung die Anzahl der Zeichen verändert.

- Aggregate `MIN()`/`MAX()` auf Arrays und zusammengesetzten Typen erlauben (Aleksander Alekseev, Marat Buharov)

- Option `WEEK` zu `EXTRACT()` hinzufügen (Tom Lane)

- Ausgabe von `EXTRACT(QUARTER ...)` für negative Werte verbessern (Tom Lane)

- Unterstützung für römische Zahlen zu `to_number()` hinzufügen (Hunaid Sohail)

  Der Zugriff erfolgt über das Muster `RN`.

- Funktion `uuidv7()` zum Erzeugen von UUID Version 7 hinzufügen (Andrey Borodin)

  Dieser UUID-Wert ist zeitlich sortierbar. Der Funktionsalias `uuidv4()` wurde hinzugefügt, um ausdrücklich UUIDs der Version 4 zu erzeugen.

- Funktionen `crc32()` und `crc32c()` zum Berechnen von CRC-Werten hinzufügen (Aleksander Alekseev)

- Mathematische Funktionen `gamma()` und `lgamma()` hinzufügen (Dean Rasheed)

- Syntax `=>` für benannte Cursorargumente in PL/pgSQL erlauben (Pavel Stehule)

  Bisher wurde nur `:=` akzeptiert.

- Benannte Argumente für `regexp_match[es]()`, `regexp_like()`, `regexp_replace()`, `regexp_count()`, `regexp_instr()`, `regexp_substr()`, `regexp_split_to_table()` und `regexp_split_to_array()` erlauben (Jian He)

##### E.2.3.5. libpq

- Funktion `PQfullProtocolVersion()` hinzufügen, um die vollständige Protokollversionsnummer einschließlich Minor-Version zu melden (Jacob Champion, Jelte Fennema-Nio)

- `libpq`-Verbindungsparameter und Umgebungsvariablen hinzufügen, um die minimal und maximal akzeptable Protokollversion für Verbindungen anzugeben (Jelte Fennema-Nio)

- Änderungen von `search_path` an den Client melden (Alexander Kukushkin, Jelte Fennema-Nio, Tomas Vondra)

- `PQtrace()`-Ausgabe für alle Nachrichtentypen einschließlich Authentifizierung hinzufügen (Jelte Fennema-Nio)

- `libpq`-Verbindungsparameter `sslkeylogfile` hinzufügen, der SSL-Schlüsselmaterial ausgibt (Abhishek Chanda, Daniel Gustafsson)

  Das ist nützlich zur Fehlersuche.

- Einige `libpq`-Funktionssignaturen so ändern, dass sie `int64_t` verwenden (Thomas Munro)

  Sie verwendeten zuvor `pg_int64`, das nun veraltet ist.

##### E.2.3.6. psql

- `psql` erlauben, benannte Prepared Statements zu parsen, zu binden und zu schließen (Anthonin Bonnefoy, Michael Paquier)

  Dies geschieht mit den neuen Befehlen `\parse`, `\bind_named` und `\close_prepared`.

- Backslash-Befehle zu `psql` hinzufügen, um Pipeline-Abfragen auszugeben (Anthonin Bonnefoy)

  Die neuen Befehle sind `\startpipeline`, `\syncpipeline`, `\sendpipeline`, `\endpipeline`, `\flushrequest`, `\flush` und `\getresults`.

- Pipeline-Status im `psql`-Prompt erlauben und zugehörige Statusvariablen hinzufügen (Anthonin Bonnefoy)

  Das neue Prompt-Zeichen ist `%P`; die neuen `psql`-Variablen sind `PIPELINE_SYNC_COUNT`, `PIPELINE_COMMAND_COUNT` und `PIPELINE_RESULT_COUNT`.

- Verbindungs-Service-Namen im `psql`-Prompt oder über eine `psql`-Variable zugänglich machen (Michael Banck)

- `psql`-Option hinzufügen, um erweiterten Modus für alle Listenbefehle zu verwenden (Dean Rasheed)

  Das Anhängen des Backslash-Suffixes `x` aktiviert dies.

- `\conninfo` von `psql` auf tabellarisches Format umstellen und mehr Informationen aufnehmen (Álvaro Herrera, Maiquel Grassi, Hunaid Sohail)

- Leakproof-Indikator von Funktionen zu den `psql`-Ausgaben `\df+`, `\do+`, `\dAo+` und `\dC+` hinzufügen (Yugo Nagata)

- Access-Method-Details für partitionierte Relationen in `\dP+` hinzufügen (Justin Pryzby)

- `default_version` zur `psql`-Extension-Ausgabe `\dx` hinzufügen (Magnus Hagander)

- `psql`-Variable `WATCH_INTERVAL` hinzufügen, um die Standardwartezeit von `\watch` festzulegen (Daniel Gustafsson)

##### E.2.3.7. Serveranwendungen

- `initdb` standardmäßig Prüfsummen aktivieren lassen (Greg Sabino Mullane)

  Die neue `initdb`-Option `--no-data-checksums` deaktiviert Prüfsummen.

- `initdb`-Option `--no-sync-data-files` hinzufügen, um das Synchronisieren von Heap-/Indexdateien zu vermeiden (Nathan Bossart)

  Die `initdb`-Option `--no-sync` ist weiterhin verfügbar, um das Synchronisieren aller Dateien zu vermeiden.

- `vacuumdb`-Option `--missing-stats-only` hinzufügen, um nur fehlende Optimizer-Statistiken zu berechnen (Corey Huinker, Nathan Bossart)

  Diese Option kann nur von Superusern ausgeführt und nur mit den Optionen `--analyze-only` und `--analyze-in-stages` verwendet werden.

- `pg_combinebackup`-Option `-k`/`--link` hinzufügen, um Hard Links zu aktivieren (Israel Barth Rubio, Robert Haas)

  Nur einige Dateien können hart verlinkt werden. Dies sollte nicht verwendet werden, wenn die Backups unabhängig voneinander genutzt werden sollen.

- `pg_verifybackup` erlauben, Backups im Tar-Format zu prüfen (Amul Sul)

- Wenn `--source-server` von `pg_rewind` einen Datenbanknamen angibt, diesen in der Ausgabe von `--write-recovery-conf` verwenden (Masahiko Sawada)

- `pg_resetwal`-Option `--char-signedness` hinzufügen, um die Standard-Signedness von `char` zu ändern (Masahiko Sawada)

###### E.2.3.7.1. pg_dump/pg_dumpall/pg_restore

- `pg_dump`-Option `--statistics` hinzufügen (Jeff Davis)

- Optionen `--sequence-data` zu `pg_dump` und `pg_dumpall` hinzufügen, um Sequenzdaten zu dumpen, die normalerweise ausgeschlossen wären (Nathan Bossart)

- Optionen `--statistics-only`, `--no-statistics`, `--no-data` und `--no-schema` zu `pg_dump`, `pg_dumpall` und `pg_restore` hinzufügen (Corey Huinker, Jeff Davis)

- Option `--no-policies` hinzufügen, um die Verarbeitung von Row-Level-Security-Policies in `pg_dump`, `pg_dumpall` und `pg_restore` zu deaktivieren (Nikolay Samokhvalov)

  Das ist nützlich bei der Migration auf Systeme mit anderen Policies.

###### E.2.3.7.2. pg_upgrade

- `pg_upgrade` erlauben, Optimizer-Statistiken zu erhalten (Corey Huinker, Jeff Davis, Nathan Bossart)

  Erweiterte Statistiken werden nicht erhalten. Außerdem wird die `pg_upgrade`-Option `--no-statistics` hinzugefügt, um das Erhalten von Statistiken zu deaktivieren.

- `pg_upgrade` erlauben, Datenbankprüfungen parallel zu verarbeiten (Nathan Bossart)

  Dies wird durch die vorhandene Option `--jobs` gesteuert.

- `pg_upgrade`-Option `--swap` hinzufügen, um Verzeichnisse auszutauschen, statt Dateien zu kopieren, zu klonen oder zu verlinken (Nathan Bossart)

  Dieser Modus ist potenziell der schnellste.

- `pg_upgrade`-Option `--set-char-signedness` hinzufügen, um die Standard-Signedness von `char` des neuen Clusters zu setzen (Masahiko Sawada)

  Dies behandelt Fälle, in denen die Standard-CPU-Signedness eines Clusters vor PostgreSQL 18 nicht zum neuen Cluster passt.

###### E.2.3.7.3. Logical Replication Applications

- `pg_createsubscriber`-Option `--all` hinzufügen, um logische Replikate für alle Datenbanken zu erstellen (Shubham Khanna)

- `pg_createsubscriber`-Option `--clean` hinzufügen, um Publications zu entfernen (Shubham Khanna)

- `pg_createsubscriber`-Option `--enable-two-phase` hinzufügen, um Prepared Transactions zu aktivieren (Shubham Khanna)

- `pg_recvlogical`-Option `--enable-failover` hinzufügen, um Failover-Slots anzugeben (Hayato Kuroda)

  Außerdem wird die Option `--enable-two-phase` als Synonym für `--two-phase` hinzugefügt, und letztere wird als veraltet markiert.

- `pg_recvlogical --drop-slot` erlauben, ohne `--dbname` zu funktionieren (Hayato Kuroda)

##### E.2.3.8. Quellcode

- Laden und Ausführen von Injection Points trennen (Michael Paquier, Heikki Linnakangas)

  Injection Points können nun über `INJECTION_POINT_LOAD()` erstellt, aber nicht ausgeführt werden; solche Injection Points können über `INJECTION_POINT_CACHED()` ausgeführt werden.

- Laufzeitargumente in Injection Points unterstützen (Michael Paquier)

- Inline-Testcode für Injection Points mit `IS_INJECTION_POINT_ATTACHED()` erlauben (Heikki Linnakangas)

- Performance beim Verarbeiten langer JSON-Zeichenketten mit SIMD (Single Instruction Multiple Data) verbessern (David Rowley)

- CRC32C-Berechnungen mit x86-AVX-512-Instruktionen beschleunigen (Raghuveer Devulapalli, Paul Amonson)

- ARM-Neon- und SVE-CPU-Intrinsics für `popcount` (Integer-Bitzählung) hinzufügen (Chiranmoy Bhattacharya, Devanga Susmitha, Rama Malladi)

- Geschwindigkeit numerischer Multiplikation und Division verbessern (Joel Jacobson, Dean Rasheed)

- Configure-Option `--with-libnuma` hinzufügen, um NUMA-Bewusstsein zu aktivieren (Jakub Wartak, Bertrand Drouvot)

  Die Funktion `pg_numa_available()` meldet NUMA-Bewusstsein; die Systemsichten `pg_shmem_allocations_numa` und `pg_buffercache_numa` melden die Verteilung von Shared Memory über NUMA-Knoten.

- TOAST-Tabelle zu `pg_index` hinzufügen, um sehr große Ausdrucksindizes zu erlauben (Nathan Bossart)

- Spalte `pg_attribute.attcacheoff` entfernen (David Rowley)

- Spalte `pg_class.relallfrozen` hinzufügen (Melanie Plageman)

- `amgettreeheight`, `amconsistentequality` und `amconsistentordering` zur API für Index Access Methods hinzufügen (Mark Dilger)

- GiST-Supportfunktion `stratnum()` hinzufügen (Paul A. Jungwirth)

- Standard-CPU-Signedness von `char` in `pg_controldata` aufzeichnen (Masahiko Sawada)

- Unterstützung für die Python „Limited API“ in PL/Python hinzufügen (Peter Eisentraut)

  Dies hilft, Probleme durch Versionsunterschiede bei Python 3.x zu vermeiden.

- Minimal unterstützte Python-Version auf 3.6.8 ändern (Jacob Champion)

- Unterstützung für OpenSSL-Versionen älter als 1.1.1 entfernen (Daniel Gustafsson)

- Wenn LLVM aktiviert ist, Version 14 oder neuer verlangen (Thomas Munro)

- Makro `PG_MODULE_MAGIC_EXT` hinzufügen, damit Erweiterungen ihren Namen und ihre Version melden können (Andrei Lepikhov)

  Auf diese Information kann über die neue Funktion `pg_get_loaded_modules()` zugegriffen werden.

- Dokumentieren, dass `SPI_connect()`/`SPI_connect_ext()` immer Erfolg (`SPI_OK_CONNECT`) zurückgibt (Stepan Neretin)

  Fehler werden immer über `ereport()` gemeldet.

- Dokumentationsabschnitt zur API- und ABI-Kompatibilität hinzufügen (David Wheeler, Peter Eisentraut)

- Kennzeichnung von Meson-Builds unter Windows als experimentell entfernen (Aleksander Alekseev)

- Configure-Optionen `--disable-spinlocks` und `--disable-atomics` entfernen (Thomas Munro)

  32-Bit-Atomics sind nun erforderlich.

- Unterstützung für die HPPA/PA-RISC-Architektur entfernen (Tom Lane)

##### E.2.3.9. Zusätzliche Module

- Erweiterung `pg_logicalinspect` zum Inspizieren logischer Snapshots hinzufügen (Bertrand Drouvot)

- Erweiterung `pg_overexplain` hinzufügen, die Debug-Details zur `EXPLAIN`-Ausgabe ergänzt (Robert Haas)

- Ausgabespalten zu `postgres_fdw_get_connections()` hinzufügen (Hayato Kuroda, Sagar Dilip Shedge)

  Die neue Ausgabespalte `used_in_xact` zeigt an, ob der Foreign Data Wrapper von einer aktuellen Transaktion verwendet wird, `closed` zeigt an, ob er geschlossen ist, `user_name` gibt den Benutzernamen an, und `remote_backend_pid` enthält die Prozess-ID des entfernten Backends.

- SCRAM-Authentifizierung vom Client an `postgres_fdw`-Server weiterreichen lassen (Matheus Alcantara, Peter Eisentraut)

  Dadurch muss `postgres_fdw`-Authentifizierungsinformation nicht in der Datenbank gespeichert werden. Aktiviert wird dies mit der `postgres_fdw`-Verbindungsoption `use_scram_passthrough`. `libpq` verwendet die neuen Verbindungsparameter `scram_client_key` und `scram_server_key`.

- SCRAM-Authentifizierung vom Client an `dblink`-Server weiterreichen lassen (Matheus Alcantara)

- Optionen `on_error` und `log_verbosity` zu `file_fdw` hinzufügen (Atsushi Torikoshi)

  Diese steuern, wie `file_fdw` ungültige Dateizeilen behandelt und meldet.

- `reject_limit` hinzufügen, um die Anzahl ungültiger Zeilen zu steuern, die `file_fdw` ignorieren darf (Atsushi Torikoshi)

  Dies ist aktiv, wenn `ON_ERROR = 'ignore'` gesetzt ist.

- Konfigurierbare Variable `min_password_length` zu `passwordcheck` hinzufügen (Emanuele Musella, Maurizio Boriani)

  Sie steuert die Mindestlänge von Passwörtern.

- `pgbench` die Anzahl fehlgeschlagener, wiederholter oder übersprungener Transaktionen in Berichten pro Skript melden lassen (Yugo Nagata)

- Servervariable `weak` zu `isn` hinzufügen, um die Akzeptanz ungültiger Prüfziffern zu steuern (Viktor Holmberg)

  Dies wurde zuvor nur durch die Funktion `isn_weak()` gesteuert.

- Sortieren von Werten erlauben, um den Aufbau von `btree_gist`-Indizes zu beschleunigen (Bernd Helmle, Andrey Borodin)

- `amcheck`-Prüffunktion `gin_index_check()` zum Überprüfen von GIN-Indizes hinzufügen (Grigory Kryachko, Heikki Linnakangas, Andrey Borodin)

- Funktionen `pg_buffercache_evict_relation()` und `pg_buffercache_evict_all()` hinzufügen, um nicht gepinnte Shared Buffers zu entfernen (Nazir Bilal Yavuz)

  Die vorhandene Funktion `pg_buffercache_evict()` gibt nun den Flush-Status des Puffers zurück.

- Erweiterungen erlauben, eigene `EXPLAIN`-Optionen zu installieren (Robert Haas, Sami Imseih)

- Erweiterungen erlauben, die kumulative Statistik-API des Servers zu verwenden (Michael Paquier)

###### E.2.3.9.1. pg_stat_statements

- Abfragen von `CREATE TABLE AS` und `DECLARE` durch `pg_stat_statements` erfassen lassen (Anthonin Bonnefoy)

  Ihnen werden nun außerdem Query-IDs zugewiesen.

- Parametrisierung von `SET`-Werten in `pg_stat_statements` erlauben (Greg Sabino Mullane, Michael Paquier)

  Dies verringert das Aufblähen durch `SET`-Anweisungen mit unterschiedlichen Konstanten.

- Spalten zu `pg_stat_statements` hinzufügen, um parallele Aktivität zu melden (Guillaume Lelarge)

  Die neuen Spalten sind `parallel_workers_to_launch` und `parallel_workers_launched`.

- `pg_stat_statements.wal_buffers_full` hinzufügen, um volle WAL-Puffer zu melden (Bertrand Drouvot)

###### E.2.3.9.2. pgcrypto

- `pgcrypto`-Algorithmen `sha256crypt` und `sha512crypt` hinzufügen (Bernd Helmle)

- CFB-Modus zur Ver- und Entschlüsselung in `pgcrypto` hinzufügen (Umar Hayat)

- Funktion `fips_mode()` hinzufügen, um den FIPS-Modus des Servers zu melden (Daniel Gustafsson)

- `pgcrypto`-Servervariable `builtin_crypto_enabled` hinzufügen, um eingebaute kryptografische Funktionen außerhalb des FIPS-Modus deaktivieren zu können (Daniel Gustafsson, Joe Conway)

  Das ist nützlich, um Verhalten im FIPS-Modus zu garantieren.

#### E.2.4. Danksagungen

Die folgenden Personen haben zu diesem Release als Patch-Autoren, Committer, Reviewer, Tester oder Melder von Problemen beigetragen; die Liste ist alphabetisch sortiert.

Abhishek Chanda Adam Guo Adam Rauch Aidar Imamov Ajin Cherian Alastair Turner Alec Cozens Aleksander Alekseev Alena Rybakina Alex Friedman Alex Richman Alexander Alehin Alexander Borisov Alexander Korotkov Alexander Kozhemyakin Alexander Kukushkin Alexander Kuzmenkov Alexander Kuznetsov Alexander Lakhin Alexander Pyhalov Alexandra Wang Alexey Dvoichenkov Alexey Makhmutov Alexey Shishkin Ali Akbar Álvaro Herrera Álvaro Mongil Amit Kapila Amit Langote Amul Sul Andreas Karlsson Andreas Scherbaum Andreas Ulbrich Andrei Lepikhov Andres Freund Andrew Andrew Bille Andrew Dunstan Andrew Jackson Andrew Kane Andrew Watkins Andrey Borodin Andrey Chudnovsky Andrey Rachitskiy Andrey Rudometov Andy Alsup Andy Fan Anthonin Bonnefoy Anthony Hsu Anthony Leung Anton Melnikov Anton Voloshin Antonin Houska

Antti Lampinen Arseniy Mukhin Artur Zakirov Arun Thirupathi Ashutosh Bapat Asphator Atsushi Torikoshi Avi Weinberg Aya Iwata Ayush Tiwari Ayush Vatsa Bastien Roucariès Ben Peachey Higdon Benoit Lobréau Bernd Helmle Bernd Reiß Bernhard Wiedemann Bertrand Drouvot Bertrand Mamasam Bharath Rupireddy Bogdan Grigorenko Boyu Yang Braulio Fdo Gonzalez Bruce Momjian Bykov Ivan Cameron Vogt Cary Huang Cédric Villemain Cees van Zeeland ChangAo Chen Chao Li Chapman Flack Charles Samborski Chengwen Wu Chengxi Sun Chiranmoy Bhattacharya Chris Gooch Christian Charukiewicz Christoph Berg Christophe Courtois Christopher Inokuchi Clemens Ruck Corey Huinker Craig Milhiser Crisp Lee Dagfinn Ilmari Mannsåker Daniel Elishakov Daniel Gustafsson Daniel Vérité Daniel Westermann Daniele Varrazzo Daniil Davydov Daria Shanina Dave Cramer Dave Page David Benjamin David Christensen David Fiedler

David G. Johnston David Geier David Rowley David Steele David Wheeler David Zhang Davinder Singh Dean Rasheed Devanga Susmitha Devrim Gündüz Dian Fay Dilip Kumar Dimitrios Apostolou Dipesh Dhameliya Dmitrii Bondar Dmitry Dolgov Dmitry Koval Dmitry Kovalenko Dmitry Yurichev Dominique Devienne Donghang Lin Dorjpalam Batbaatar Drew Callahan Duncan Sands Dwayne Towell Dzmitry Jachnik Egor Chindyaskin Egor Rogov Emanuel Ionescu Emanuele Musella Emre Hasegeli Eric Cyr Erica Zhang Erik Nordström Erik Rijkers Erik Wienhold Erki Eessaar Ethan Mertz Etienne LAFARGE Etsuro Fujita Euler Taveira Evan Si Evgeniy Gorbanev Fabio R. Sluzala Fabrízio de Royes Mello Feike Steenbergen Feliphe Pozzer Felix Fire Emerald Florents Tselai Francesco Degrassi Frank Streitzig Frédéric Yhuel Fredrik Widlert Gabriele Bartolini Gavin Panella Geoff Winkless George MacKerron

Gilles Darold Grant Gryczan Greg Burd Greg Sabino Mullane Greg Stark Grigory Kryachko Guillaume Lelarge Gunnar Morling Gunnar Wagner Gurjeet Singh Haifang Wang Hajime Matsunaga Hamid Akhtar Hannu Krosing Hari Krishna Sunder Haruka Takatsuka Hayato Kuroda Heikki Linnakangas Hironobu Suzuki Holger Jakobs Hubert Lubaczewski Hugo Dubois Hugo Zhang Hunaid Sohail Hywel Carver Ian Barwick Ibrar Ahmed Igor Gnatyuk Igor Korot Ilia Evdokimov Ilya Gladyshev Ilyasov Ian Imran Zaheer Isaac Morland Israel Barth Rubio Ivan Kush Jacob Brazeal Jacob Champion Jaime Casanova Jakob Egger Jakub Wartak James Coleman James Hunter Jan Behrens Japin Li Jason Smith Jayesh Dehankar Jeevan Chalke Jeff Davis Jehan-Guillaume de Rorthais Jelte Fennema-Nio Jian He Jianghua Yang Jiao Shuntian Jim Jones Jim Nasby Jingtang Zhang Jingzhou Fu

Joe Conway Joel Jacobson John Hutchins John Naylor Jonathan Katz Jorge Solórzano José Villanova Josef Šimánek Joseph Koshakow Julien Rouhaud Junwang Zhao Justin Pryzby Kaido Vaikla Kaimeh Karina Litskevich Karthik S Kartyshov Ivan Kashif Zeeshan Keisuke Kuroda Kevin Hale Boyes Kevin K Biju Kirill Reshke Kirill Zdornyy Koen De Groote Koichi Suzuki Koki Nakamura Konstantin Knizhnik Kouhei Sutou Kuntal Ghosh Kyotaro Horiguchi Lakshmi Narayana Velayudam Lars Kanis Laurence Parry Laurenz Albe Lele Gaifax Li Yong Lilian Ontowhee Lingbin Meng Luboslav Špilák Luca Vallisa Lukas Fittl Maciek Sakrejda Magnus Hagander Mahendra Singh Thalor Mahendrakar Srinivasarao Maiquel Grassi Maksim Korotkov Maksim Melnikov Man Zeng Marat Buharov Marc Balmer Marco Nenciarini Marcos Pegoraro Marina Polyakova Mark Callaghan Mark Dilger Marlene Brandstaetter Marlene Reiterer

Martin Rakhmanov Masahiko Sawada Masahiro Ikeda Masao Fujii Mason Mackaman Mat Arye Matheus Alcantara Mats Kindahl Matthew Gabeler-Lee Matthew Kim Matthew Sterrett Matthew Woodcraft Matthias van de Meent Matthieu Denais Maurizio Boriani Max Johnson Max Madden Maxim Boguk Maxim Orlov Maximilian Chrzan Melanie Plageman Melih Mutlu Mert Alev Michael Banck Michael Bondarenko Michael Christofides Michael Guissine Michael Harris Michaël Paquier Michail Nikolaev Michal Kleczek Michel Pelletier Mikaël Gourlaouen Mikhail Gribkov Mikhail Kot Milosz Chmura Muralikrishna Bandaru Murat Efendioglu Mutaamba Maasha Naeem Akhter Nat Makarevitch Nathan Bossart Navneet Kumar Nazir Bilal Yavuz Neil Conway Niccolò Fei Nick Davies Nicolas Maus Niek Brasa Nikhil Raj Nikita Nikita Kalinin Nikita Malakhov Nikolay Samokhvalov Nikolay Shaplov Nisha Moond Nitin Jadhav Nitin Motiani

Noah Misch Noboru Saito Noriyoshi Shinoda Ole Peder Brandtzæg Oleg Sibiryakov Oleg Tselebrovskiy Olleg Samoylov Onder Kalaci Ondrej Navratil Patrick Stählin Paul Amonson Paul Jungwirth Paul Ramsey Pavel Borisov Pavel Luzanov Pavel Nekrasov Pavel Stehule Peter Eisentraut Peter Geoghegan Peter Mittere Peter Smith Phil Eaton Philipp Salvisberg Philippe Beaudoin Pierre Giraud Pixian Shi Polina Bungina Przemyslaw Sztoch Quynh Tran Rafia Sabih Raghuveer Devulapalli Rahila Syed Rama Malladi Ran Benita Ranier Vilela Renan Alves Fonseca Richard Guo Richard Neill Rintaro Ikeda Robert Haas Robert Treat Robins Tharakan Roman Zharkov Ronald Cruz Ronan Dunklau Rui Zhao Rushabh Lathia Rustam Allakov Ryo Kanbayashi Ryohei Takahashi RyotaK Sagar Dilip Shedge Salvatore Dipietro Sam Gabrielsson Sam James Sameer Kumar Sami Imseih Samuel Thibault

Satyanarayana Narlapuram Sebastian Skalacki Senglee Choi Sergei Kornilov Sergey Belyashov Sergey Dudoladov Sergey Prokhorenko Sergey Sargsyan Sergey Soloviev Sergey Tatarintsev Shaik Mohammad Mujeeb Shawn McCoy Shenhao Wang Shihao Zhong Shinya Kato Shlok Kyal Shubham Khanna Shveta Malik Simon Riggs Smolkin Grigory Sofia Kopikova Song Hongyu Song Jinzhou Soumyadeep Chakraborty Sravan Kumar Srinath Reddy Stan Hu Stepan Neretin Stephen Fewer Stephen Frost Steve Chavez Steven Niu Suraj Kharage Sven Klemm Takamichi Osumi Takeshi Ideriha Tatsuo Ishii Ted Yu Tels Tender Wang Teodor Sigaev Thom Brown Thomas Baehler Thomas Krennwallner Thomas Munro Tim Wood Timur Magomedov Tobias Wendorff Todd Cook Tofig Aliev Tom Lane Tomas Vondra Tomasz Rybak Tomasz Szypowski Torsten Foertsch Toshi Harada Tristan Partin Triveni N

Umar Hayat Vallimaharajan G Vasya Boytsov Victor Yegorov Vignesh C Viktor Holmberg Vinícius Abrahão Vinod Sridharan Virender Singla Vitaly Davydov Vladlen Popolitov Vladyslav Nebozhyn Walid Ibrahim Webbo Han Wenhui Qiu Will Mortensen Will Storey Wolfgang Walther Xin Zhang Xing Guo Xuneng Zhou Yan Chengpen Yang Lei Yaroslav Saburov Yaroslav Syrytsia Yasir Hussain Yasuo Honda Yogesh Sharma Yonghao Lee Yoran Heling Yu Liang Yugo Nagata Yuhang Qiu Yuki Seino Yura Sokolov Yurii Rashkovskii Yushi Ogiwara Yusuke Sugie Yuta Katsuragi Yuto Sasaki Yuuki Fujii Yuya Watari Zane Duffield Zeyuan Hu Zhang Mingli Zhihong Yu Zhijie Hou Zsolt Parragi

### E.3. Frühere Releases

Versionshinweise für frühere Release-Zweige finden sich unter <https://www.postgresql.org/docs/release/>.
