# Anhang F. Weitere mitgelieferte Module und Erweiterungen

Dieser Anhang und der nächste enthalten Informationen zu den optionalen Komponenten im Verzeichnis `contrib` der PostgreSQL-Distribution. Dazu gehören Portierungswerkzeuge, Analysehilfen und Plug-in-Funktionen, die nicht Teil des PostgreSQL-Kernsystems sind. Sie sind vor allem deshalb getrennt, weil sie sich an eine begrenzte Zielgruppe richten oder zu experimentell sind, um Teil des Hauptquellbaums zu sein. Das mindert ihren Nutzen nicht.

Dieser Anhang behandelt Erweiterungen und andere Server-Plug-in-Modulbibliotheken aus `contrib`. [Anhang G](G_Weitere_mitgelieferte_Programme.md) behandelt Hilfsprogramme.

Beim Bau aus der Quelldistribution werden diese optionalen Komponenten nicht automatisch gebaut, es sei denn, Sie bauen das Ziel `world`. Sie können alle Komponenten bauen und installieren, indem Sie im Verzeichnis `contrib` eines konfigurierten Quellbaums Folgendes ausführen:

```sh
make
make install
```

Um nur ein einzelnes Modul zu bauen und zu installieren, führen Sie dieselben Befehle im Unterverzeichnis dieses Moduls aus. Viele Module besitzen Regressionstests, die vor der Installation mit folgendem Befehl ausgeführt werden können:

```sh
make check
```

Nach der Installation und bei laufendem PostgreSQL-Server können Sie stattdessen ausführen:

```sh
make installcheck
```

Wenn Sie eine vorkonfektionierte PostgreSQL-Version verwenden, werden diese Komponenten typischerweise als separates Unterpaket bereitgestellt, zum Beispiel als `postgresql-contrib`.

Viele Komponenten stellen neue benutzerdefinierte Funktionen, Operatoren oder Typen bereit, die als Erweiterungen paketiert sind. Um eine solche Erweiterung zu verwenden, müssen Sie nach der Installation des Codes die neuen SQL-Objekte im Datenbanksystem registrieren. Dazu führen Sie einen `CREATE EXTENSION`-Befehl aus. In einer frischen Datenbank genügt zum Beispiel:

```sql
CREATE EXTENSION extension_name;
```

Dieser Befehl registriert die neuen SQL-Objekte nur in der aktuellen Datenbank. Sie müssen ihn daher in jeder Datenbank ausführen, in der die Funktionen der Erweiterung verfügbar sein sollen. Alternativ können Sie ihn in der Datenbank `template1` ausführen, sodass die Erweiterung standardmäßig in anschließend erzeugte Datenbanken kopiert wird.

Für alle Erweiterungen muss der Befehl `CREATE EXTENSION` von einem Datenbank-Superuser ausgeführt werden, sofern die Erweiterung nicht als „trusted“ gilt. Vertrauenswürdige Erweiterungen können von jedem Benutzer installiert werden, der das Privileg `CREATE` in der aktuellen Datenbank besitzt. Welche Erweiterungen vertrauenswürdig sind, ist in den folgenden Abschnitten angegeben. Im Allgemeinen sind vertrauenswürdige Erweiterungen solche, die keinen Zugriff auf Funktionalität außerhalb der Datenbank ermöglichen.

Die folgenden Erweiterungen sind in einer Standardinstallation vertrauenswürdig:

| Erweiterung | Erweiterung | Erweiterung | Erweiterung |
| --- | --- | --- | --- |
| `btree_gin` | `fuzzystrmatch` | `ltree` | `tcn` |
| `btree_gist` | `hstore` | `pgcrypto` | `tsm_system_rows` |
| `citext` | `intarray` | `pg_trgm` | `tsm_system_time` |
| `cube` | `isn` | `seg` | `unaccent` |
| `dict_int` | `lo` | `tablefunc` | `uuid-ossp` |

Viele Erweiterungen erlauben es, ihre Objekte in einem Schema Ihrer Wahl zu installieren. Fügen Sie dazu `SCHEMA schema_name` zum Befehl `CREATE EXTENSION` hinzu. Standardmäßig werden die Objekte in Ihrem aktuellen Zielschema für neue Objekte abgelegt, das wiederum standardmäßig `public` ist.

Beachten Sie jedoch, dass einige dieser Komponenten keine „Erweiterungen“ in diesem Sinne sind, sondern auf andere Weise in den Server geladen werden, zum Beispiel über `shared_preload_libraries`. Einzelheiten finden Sie in der Dokumentation der jeweiligen Komponente.

### F.1. amcheck — Werkzeuge zur Prüfung der Konsistenz von Tabellen und Indizes

Das Modul `amcheck` stellt Funktionen bereit, mit denen Sie die logische Konsistenz der Struktur von Relationen prüfen können.

Die B-Tree-Prüffunktionen prüfen verschiedene Invarianten in der Struktur der Darstellung bestimmter Relationen. Die Korrektheit der Zugriffsmethodenfunktionen hinter Index-Scans und anderen wichtigen Operationen hängt davon ab, dass diese Invarianten immer gelten. Einige Funktionen prüfen zum Beispiel unter anderem, dass alle B-Tree-Seiten Einträge in „logischer“ Reihenfolge enthalten. Bei B-Tree-Indizes auf `text` sollten Indextupel etwa in collierter lexikalischer Reihenfolge stehen. Wenn diese Invariante verletzt ist, können binäre Suchen auf der betroffenen Seite Index-Scans falsch leiten, was zu falschen Ergebnissen von SQL-Abfragen führen kann. Wenn die Struktur gültig erscheint, wird kein Fehler ausgelöst. Während diese Prüffunktionen laufen, wird der `search_path` vorübergehend auf `pg_catalog, pg_temp` geändert.

Die Prüfung erfolgt mit denselben Verfahren, die auch Index-Scans selbst verwenden. Dabei kann auch benutzerdefinierter Code aus Operatorklassen beteiligt sein. Die B-Tree-Indexprüfung stützt sich zum Beispiel auf Vergleiche mit einer oder mehreren B-Tree-Supportroutinen der Supportfunktion 1. Einzelheiten zu Supportfunktionen von Operatorklassen finden Sie in [Abschnitt 36.16.3](36_SQL_erweitern.md#36163-supportroutinen-von-indexmethoden).

Anders als die B-Tree-Prüffunktionen, die Beschädigungen durch Fehler melden, prüft die Heap-Prüffunktion `verify_heapam` eine Tabelle und versucht, eine Ergebnismenge zurückzugeben, mit einer Zeile pro gefundener Beschädigung. Wenn jedoch Einrichtungen, von denen `verify_heapam` abhängt, selbst beschädigt sind, kann die Funktion möglicherweise nicht fortfahren und stattdessen einen Fehler auslösen.

Die Berechtigung zum Ausführen von `amcheck`-Funktionen kann an Nicht-Superuser vergeben werden. Vor dem Erteilen solcher Berechtigungen sollten jedoch Sicherheits- und Datenschutzaspekte sorgfältig geprüft werden. Die von diesen Funktionen erzeugten Beschädigungsberichte konzentrieren sich zwar weniger auf die Inhalte der beschädigten Daten als auf deren Struktur und die Art der gefundenen Beschädigungen. Ein Angreifer, der die Berechtigung zum Ausführen dieser Funktionen erhält, könnte dennoch, insbesondere wenn er auch Beschädigungen herbeiführen kann, aus solchen Meldungen Rückschlüsse auf die Daten selbst ziehen.

#### F.1.1. Funktionen

```text
bt_index_check(index regclass, heapallindexed boolean, checkunique boolean) returns void
```

`bt_index_check` prüft, ob das Ziel, ein B-Tree-Index, verschiedene Invarianten einhält. Beispielverwendung:

```sql
SELECT bt_index_check(index => c.oid, heapallindexed => i.indisunique),
       c.relname,
       c.relpages
FROM pg_index i
JOIN pg_opclass op ON i.indclass[0] = op.oid
JOIN pg_am am ON op.opcmethod = am.oid
JOIN pg_class c ON i.indexrelid = c.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE am.amname = 'btree' AND n.nspname = 'pg_catalog'
-- Don't check temp tables, which may be from another session:
AND c.relpersistence != 't'
-- Function may throw an error when this is omitted:
AND c.relkind = 'i' AND i.indisready AND i.indisvalid
ORDER BY c.relpages DESC LIMIT 10;
```

Dies kann eine Ausgabe wie die folgende erzeugen:

```text
 bt_index_check |             relname             | relpages
----------------+---------------------------------+----------
                | pg_depend_reference_index       |       43
                | pg_depend_depender_index        |       40
                | pg_proc_proname_args_nsp_index  |       31
                | pg_description_o_c_o_index      |       21
                | pg_attribute_relid_attnam_index |       14
                | pg_proc_oid_index               |       10
                | pg_attribute_relid_attnum_index |        9
                | pg_amproc_fam_proc_index        |        5
                | pg_amop_opr_fam_index           |        5
                | pg_amop_fam_strat_index         |        5
(10 rows)
```

Dieses Beispiel zeigt eine Sitzung, die die zehn größten Katalogindizes in der Datenbank `test` prüft. Für die Teilmenge der eindeutigen Indizes wird außerdem geprüft, ob Heap-Tupel als Indextupel vorhanden sind. Da kein Fehler ausgelöst wird, erscheinen alle geprüften Indizes logisch konsistent. Natürlich ließe sich diese Abfrage leicht so ändern, dass `bt_index_check` für jeden Index in der Datenbank aufgerufen wird, für den die Prüfung unterstützt wird.

`bt_index_check` erwirbt einen `AccessShareLock` auf dem Zielindex und auf der zugehörigen Heap-Relation. Das ist derselbe Sperrmodus, den einfache `SELECT`-Anweisungen auf Relationen erwerben. `bt_index_check` prüft keine Invarianten, die Kind-/Eltern-Beziehungen überspannen. Wenn `heapallindexed` `true` ist, prüft die Funktion aber, ob alle Heap-Tupel im Index als Indextupel vorhanden sind. Wenn `checkunique` `true` ist, prüft `bt_index_check`, dass bei doppelten Einträgen in einem eindeutigen Index nicht mehr als ein Eintrag sichtbar ist. Wenn in einer laufenden Produktionsumgebung eine routinemäßige, leichtgewichtige Prüfung auf Beschädigungen benötigt wird, bietet `bt_index_check` oft den besten Kompromiss zwischen Prüftiefe und begrenzter Auswirkung auf Anwendungsleistung und Verfügbarkeit.

```text
bt_index_parent_check(index regclass, heapallindexed boolean, rootdescend boolean, checkunique boolean) returns void
```

`bt_index_parent_check` prüft, ob das Ziel, ein B-Tree-Index, verschiedene Invarianten einhält. Optional prüft die Funktion, wenn das Argument `heapallindexed` `true` ist, ob alle Heap-Tupel, die im Index zu finden sein sollten, dort vorhanden sind. Wenn `checkunique` `true` ist, prüft `bt_index_parent_check`, dass bei doppelten Einträgen in einem eindeutigen Index nicht mehr als ein Eintrag sichtbar ist. Wenn das optionale Argument `rootdescend` `true` ist, sucht die Prüfung Tupel auf Blattebene erneut, indem sie für jedes Tupel eine neue Suche von der Wurzelseite aus durchführt.

Die Prüfungen, die `bt_index_parent_check` durchführen kann, sind eine Obermenge der Prüfungen von `bt_index_check`. `bt_index_parent_check` kann als gründlichere Variante von `bt_index_check` verstanden werden: Anders als `bt_index_check` prüft `bt_index_parent_check` auch Invarianten, die Eltern-/Kind-Beziehungen überspannen, einschließlich der Prüfung, dass in der Indexstruktur keine Downlinks fehlen. `bt_index_parent_check` folgt der allgemeinen Konvention, einen Fehler auszulösen, wenn eine logische Inkonsistenz oder ein anderes Problem gefunden wird.

`bt_index_parent_check` benötigt einen `ShareLock` auf dem Zielindex; außerdem wird ein `ShareLock` auf der Heap-Relation erworben. Diese Sperren verhindern gleichzeitige Datenänderungen durch `INSERT`-, `UPDATE`- und `DELETE`-Befehle. Sie verhindern außerdem, dass die zugrunde liegende Relation gleichzeitig von `VACUUM` oder anderen Utility-Befehlen verarbeitet wird. Beachten Sie, dass die Funktion Sperren nur während ihrer Laufzeit hält, nicht während der gesamten Transaktion.

Die zusätzliche Prüfung von `bt_index_parent_check` erkennt mit höherer Wahrscheinlichkeit verschiedene pathologische Fälle. Dazu können eine fehlerhaft implementierte B-Tree-Operatorklasse gehören, die vom geprüften Index verwendet wird, oder hypothetisch bislang unentdeckte Fehler im zugrunde liegenden Code der B-Tree-Indexzugriffsmethode. Beachten Sie, dass `bt_index_parent_check` im Hot-Standby-Modus, also auf schreibgeschützten physischen Replikaten, anders als `bt_index_check` nicht verwendet werden kann.

```text
gin_index_check(index regclass) returns void
```

`gin_index_check` prüft, ob der Ziel-GIN-Index konsistente Eltern-/Kind-Tupelbeziehungen besitzt, also keine Elterntupel eine Tupelanpassung benötigen, und ob der Seitengraph Invarianten balancierter Bäume einhält. Interne Seiten dürfen dabei nur auf Blattseiten oder nur auf interne Seiten verweisen.

> **Tipp:** `bt_index_check` und `bt_index_parent_check` geben beide Protokollmeldungen über den Prüfprozess mit den Schweregraden `DEBUG1` und `DEBUG2` aus. Diese Meldungen liefern detaillierte Informationen über den Prüfprozess, die für PostgreSQL-Entwickler interessant sein können. Auch fortgeschrittene Benutzer können diese Informationen nützlich finden, da sie zusätzlichen Kontext liefern, falls die Prüfung tatsächlich eine Inkonsistenz erkennt.
>
> Wenn Sie vor einer Prüfungsabfrage in einer interaktiven `psql`-Sitzung Folgendes ausführen, werden Fortschrittsmeldungen der Prüfung mit gut handhabbarer Detailtiefe angezeigt:
>
> ```sql
> SET client_min_messages = DEBUG1;
> ```

```text
verify_heapam(relation regclass, on_error_stop boolean, check_toast boolean, skip text, startblock bigint, endblock bigint, blkno OUT bigint, offnum OUT integer, attnum OUT integer, msg OUT text) returns setof record
```

`verify_heapam` prüft eine Tabelle, Sequenz oder materialisierte Sicht auf strukturelle Beschädigungen, bei denen Seiten in der Relation ungültig formatierte Daten enthalten, sowie auf logische Beschädigungen, bei denen Seiten strukturell gültig, aber mit dem Rest des Datenbankclusters inkonsistent sind.

Die folgenden optionalen Argumente werden erkannt:

- `on_error_stop`
  - Wenn `true`, endet die Beschädigungsprüfung am Ende des ersten Blocks, in dem Beschädigungen gefunden wurden.
  - Standardwert ist `false`.

- `check_toast`
  - Wenn `true`, werden ausgelagerte TOAST-Werte gegen die TOAST-Tabelle der Zielrelation geprüft.
  - Diese Option ist bekanntermaßen langsam. Wenn außerdem die TOAST-Tabelle oder ihr Index beschädigt ist, könnte die Prüfung gegen TOAST-Werte unter Umständen den Server abstürzen lassen, obwohl in vielen Fällen lediglich ein Fehler erzeugt wird.
  - Standardwert ist `false`.

- `skip`
  - Wenn nicht `none`, überspringt die Beschädigungsprüfung Blöcke, die wie angegeben als all-visible oder all-frozen markiert sind. Gültige Optionen sind `all-visible`, `all-frozen` und `none`.
  - Standardwert ist `none`.

- `startblock`
  - Wenn angegeben, beginnt die Beschädigungsprüfung am angegebenen Block und überspringt alle vorherigen Blöcke. Es ist ein Fehler, `startblock` außerhalb des Blockbereichs der Zieltabelle anzugeben.
  - Standardmäßig beginnt die Prüfung beim ersten Block.

- `endblock`
  - Wenn angegeben, endet die Beschädigungsprüfung am angegebenen Block und überspringt alle übrigen Blöcke. Es ist ein Fehler, `endblock` außerhalb des Blockbereichs der Zieltabelle anzugeben.
  - Standardmäßig werden alle Blöcke geprüft.

Für jede gefundene Beschädigung gibt `verify_heapam` eine Zeile mit den folgenden Spalten zurück:

- `blkno`
  - Die Nummer des Blocks, der die beschädigte Seite enthält.

- `offnum`
  - Die `OffsetNumber` des beschädigten Tupels.

- `attnum`
  - Die Attributnummer der beschädigten Spalte im Tupel, wenn die Beschädigung eine bestimmte Spalte und nicht das Tupel als Ganzes betrifft.

- `msg`
  - Eine Meldung, die das erkannte Problem beschreibt.

#### F.1.2. Optionale `heapallindexed`-Prüfung

Wenn das Argument `heapallindexed` der B-Tree-Prüffunktionen `true` ist, wird eine zusätzliche Prüfphase gegen die Tabelle ausgeführt, die zur Zielindexrelation gehört. Diese besteht aus einer „Dummy“-Operation `CREATE INDEX`, die das Vorhandensein aller hypothetischen neuen Indextupel gegen eine temporäre, im Arbeitsspeicher gehaltene Zusammenfassungsstruktur prüft. Diese Struktur wird bei Bedarf während der grundlegenden ersten Prüfphase aufgebaut. Die Zusammenfassungsstruktur erzeugt einen „Fingerabdruck“ jedes Tupels, das im Zielindex gefunden wird. Das übergeordnete Prinzip der `heapallindexed`-Prüfung lautet: Ein neuer Index, der dem vorhandenen Zielindex entspricht, darf nur Einträge enthalten, die in der vorhandenen Struktur gefunden werden können.

Die zusätzliche `heapallindexed`-Phase verursacht erheblichen Zusatzaufwand; die Prüfung dauert typischerweise ein Mehrfaches länger. Die Sperren auf Relationsebene ändern sich jedoch nicht, wenn die `heapallindexed`-Prüfung ausgeführt wird.

Die Größe der Zusammenfassungsstruktur wird durch `maintenance_work_mem` begrenzt. Um sicherzustellen, dass die Wahrscheinlichkeit, eine Inkonsistenz für jedes im Index zu repräsentierende Heap-Tupel nicht zu erkennen, höchstens 2 % beträgt, werden ungefähr 2 Byte Speicher pro Tupel benötigt. Wenn weniger Speicher pro Tupel verfügbar ist, steigt die Wahrscheinlichkeit, eine Inkonsistenz zu übersehen, langsam an. Dieser Ansatz begrenzt den Prüfaufwand erheblich und senkt die Wahrscheinlichkeit, ein Problem zu erkennen, nur geringfügig, besonders in Installationen, in denen die Prüfung als routinemäßige Wartungsaufgabe behandelt wird. Jedes einzelne fehlende oder fehlerhaft geformte Tupel hat bei jedem neuen Prüfversuch eine neue Chance, erkannt zu werden.

#### F.1.3. `amcheck` effektiv verwenden

`amcheck` kann verschiedene Arten von Fehlermodi erkennen, die Datenprüfsummen nicht erfassen können. Dazu gehören:

- Strukturelle Inkonsistenzen, die durch falsche Implementierungen von Operatorklassen verursacht werden.

  Dazu zählen Probleme, die entstehen, wenn sich die Vergleichsregeln von Betriebssystem-Collations ändern. Vergleiche von Werten eines collierbaren Typs wie `text` müssen unveränderlich sein, genauso wie alle Vergleiche, die für B-Tree-Index-Scans verwendet werden, unveränderlich sein müssen. Daraus folgt, dass sich Collation-Regeln des Betriebssystems niemals ändern dürfen. Obwohl selten, können Aktualisierungen der Collation-Regeln des Betriebssystems solche Probleme verursachen. Häufiger ist eine Inkonsistenz der Collation-Reihenfolge zwischen einem Primärserver und einem Standby-Server beteiligt, möglicherweise weil unterschiedliche Hauptversionen des Betriebssystems verwendet werden. Solche Inkonsistenzen treten im Allgemeinen nur auf Standby-Servern auf und können daher meist auch nur dort erkannt werden.

  Wenn ein solches Problem auftritt, betrifft es möglicherweise nicht jeden einzelnen Index, der mit einer betroffenen Collation sortiert ist, einfach weil die indizierten Werte zufällig dieselbe absolute Reihenfolge haben können, unabhängig von der Verhaltensinkonsistenz. Weitere Einzelheiten dazu, wie PostgreSQL Betriebssystem-Locales und Collations verwendet, finden Sie in [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung) und [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung).

- Strukturelle Inkonsistenzen zwischen Indizes und den Heap-Relationen, die indiziert werden, wenn die `heapallindexed`-Prüfung ausgeführt wird.

  Während des normalen Betriebs findet kein Gegenprüfen von Indizes gegen ihre Heap-Relation statt. Symptome einer Heap-Beschädigung können subtil sein.

- Beschädigungen, die durch hypothetische, bislang unentdeckte Fehler im zugrunde liegenden PostgreSQL-Code der Zugriffsmethoden, im Sortiercode oder im Transaktionsverwaltungscode verursacht werden.

  Die automatische Prüfung der strukturellen Integrität von Indizes spielt eine Rolle beim allgemeinen Testen neuer oder vorgeschlagener PostgreSQL-Funktionen, die plausibel eine logische Inkonsistenz einführen könnten. Die Prüfung der Tabellenstruktur sowie der zugehörigen Sichtbarkeits- und Transaktionsstatusinformationen spielt eine ähnliche Rolle. Eine naheliegende Teststrategie besteht darin, `amcheck`-Funktionen fortlaufend aufzurufen, während die Standard-Regressionstests laufen. Einzelheiten zum Ausführen der Tests finden Sie in [Abschnitt 31.1](31_Regressionstests.md#311-tests-ausführen).

- Fehler im Dateisystem oder Speichersubsystem, wenn Datenprüfsummen deaktiviert sind.

  Beachten Sie, dass `amcheck` eine Seite so untersucht, wie sie zum Prüfzeitpunkt in einem Shared-Memory-Puffer vorliegt, wenn beim Zugriff auf den Block nur ein Shared-Buffer-Treffer erfolgt. Folglich untersucht `amcheck` nicht notwendigerweise Daten, die zum Prüfzeitpunkt aus dem Dateisystem gelesen wurden. Wenn Prüfsummen aktiviert sind, kann `amcheck` beim Einlesen eines beschädigten Blocks in einen Puffer aufgrund eines Prüfsummenfehlers einen Fehler auslösen.

- Beschädigungen, die durch fehlerhaften RAM oder das breitere Speichersubsystem verursacht werden.

  PostgreSQL schützt nicht gegen korrigierbare Speicherfehler. Es wird angenommen, dass Sie RAM verwenden, der branchenübliche Error Correcting Codes (ECC) oder besseren Schutz bietet. ECC-Speicher ist jedoch typischerweise nur gegen Einzelbitfehler immun und sollte nicht als absoluter Schutz vor Ausfällen verstanden werden, die zu Speicherbeschädigung führen.

  Wenn die `heapallindexed`-Prüfung ausgeführt wird, besteht im Allgemeinen eine deutlich erhöhte Chance, Einzelbitfehler zu erkennen, da strikte binäre Gleichheit geprüft wird und die indizierten Attribute im Heap geprüft werden.

Strukturelle Beschädigungen können durch fehlerhafte Speicherhardware entstehen oder dadurch, dass Relationsdateien von nicht zugehöriger Software überschrieben oder verändert werden. Diese Art von Beschädigung kann auch mit Datenprüfsummen erkannt werden.

Relationsseiten, die korrekt formatiert, intern konsistent und relativ zu ihren eigenen internen Prüfsummen korrekt sind, können dennoch logische Beschädigungen enthalten. Solche Beschädigungen lassen sich daher nicht mit Prüfsummen erkennen. Beispiele sind ausgelagerte TOAST-Werte in der Haupttabelle, denen ein entsprechender Eintrag in der TOAST-Tabelle fehlt, oder Tupel in der Haupttabelle mit einer Transaktions-ID, die älter ist als die älteste gültige Transaktions-ID in der Datenbank oder im Cluster.

In Produktionssystemen wurden mehrere Ursachen logischer Beschädigungen beobachtet, darunter Fehler in der PostgreSQL-Server-Software, fehlerhafte oder schlecht konzipierte Backup- und Restore-Werkzeuge sowie Benutzerfehler.

Beschädigte Relationen sind gerade in laufenden Produktionsumgebungen besonders besorgniserregend, also genau dort, wo risikoreiche Aktivitäten am wenigsten willkommen sind. Aus diesem Grund wurde `verify_heapam` so entworfen, dass Beschädigungen ohne unnötiges Risiko diagnostiziert werden können. Die Funktion kann nicht gegen alle Ursachen von Backend-Abstürzen schützen, da bereits das Ausführen der aufrufenden Abfrage auf einem stark beschädigten System unsicher sein könnte. Auf Katalogtabellen wird zugegriffen; das kann problematisch sein, wenn die Kataloge selbst beschädigt sind.

Im Allgemeinen kann `amcheck` nur das Vorhandensein von Beschädigungen beweisen, nicht deren Abwesenheit.

#### F.1.4. Beschädigungen reparieren

Kein von `amcheck` ausgelöster Fehler, der eine Beschädigung betrifft, sollte jemals ein falsch positiver Treffer sein. `amcheck` löst Fehler bei Bedingungen aus, die per Definition niemals auftreten sollten. Deshalb ist bei `amcheck`-Fehlern oft eine sorgfältige Analyse erforderlich.

Es gibt keine allgemeine Methode zur Reparatur von Problemen, die `amcheck` erkennt. Die Ursache einer Invariantenverletzung sollte ermittelt werden. `pageinspect` kann bei der Diagnose von durch `amcheck` erkannten Beschädigungen nützlich sein. Ein `REINDEX` repariert Beschädigungen möglicherweise nicht wirksam.


### F.2. auth_delay — Pause bei Authentifizierungsfehlern

`auth_delay` veranlasst den Server, vor dem Melden eines Authentifizierungsfehlers kurz zu warten, um Brute-Force-Angriffe auf Datenbankpasswörter zu erschweren. Beachten Sie, dass dies nichts gegen Denial-of-Service-Angriffe ausrichtet und sie sogar verschärfen kann, da Prozesse, die vor dem Melden eines Authentifizierungsfehlers warten, weiterhin Verbindungs-Slots belegen.

Damit das Modul funktioniert, muss es über `shared_preload_libraries` in `postgresql.conf` geladen werden.

#### F.2.1. Konfigurationsparameter

- `auth_delay.milliseconds` (`integer`)
  - Die Anzahl der Millisekunden, die vor dem Melden eines Authentifizierungsfehlers gewartet wird. Der Standardwert ist `0`.

Dieser Parameter muss in `postgresql.conf` gesetzt werden. Eine typische Verwendung könnte so aussehen:

```conf
# postgresql.conf
shared_preload_libraries = 'auth_delay'

auth_delay.milliseconds = '500'
```

#### F.2.2. Autor

KaiGai Kohei `<kaigai@ak.jp.nec.com>`

### F.3. auto_explain — Ausführungspläne langsamer Abfragen protokollieren

Das Modul `auto_explain` bietet eine Möglichkeit, Ausführungspläne langsamer Anweisungen automatisch zu protokollieren, ohne `EXPLAIN` von Hand ausführen zu müssen. Das ist besonders hilfreich, um nicht optimierte Abfragen in großen Anwendungen aufzuspüren.

Das Modul stellt keine über SQL zugänglichen Funktionen bereit. Um es zu verwenden, laden Sie es einfach in den Server. Sie können es in einer einzelnen Sitzung laden:

```sql
LOAD 'auto_explain';
```

Dazu müssen Sie Superuser sein. Üblicher ist es, `auto_explain` für einige oder alle Sitzungen vorzuladen, indem Sie es in `session_preload_libraries` oder `shared_preload_libraries` in `postgresql.conf` aufnehmen. Dann können Sie unerwartet langsame Abfragen unabhängig davon verfolgen, wann sie auftreten. Natürlich entsteht dadurch ein gewisser Overhead.

#### F.3.1. Konfigurationsparameter

Mehrere Konfigurationsparameter steuern das Verhalten von `auto_explain`. Beachten Sie, dass das Standardverhalten darin besteht, nichts zu tun. Sie müssen also mindestens `auto_explain.log_min_duration` setzen, wenn Sie Ergebnisse erhalten möchten.

- `auto_explain.log_min_duration` (`integer`)
  - `auto_explain.log_min_duration` ist die minimale Ausführungszeit einer Anweisung in Millisekunden, ab der der Plan der Anweisung protokolliert wird. Der Wert `0` protokolliert alle Pläne. `-1`, der Standardwert, deaktiviert das Protokollieren von Plänen. Wenn Sie den Wert zum Beispiel auf `250ms` setzen, werden alle Anweisungen protokolliert, die mindestens `250ms` laufen. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_parameter_max_length` (`integer`)
  - `auto_explain.log_parameter_max_length` steuert das Protokollieren von Abfrageparameterwerten. Der Wert `-1`, der Standardwert, protokolliert Parameterwerte vollständig. `0` deaktiviert das Protokollieren von Parameterwerten. Ein Wert größer als null kürzt jeden Parameterwert auf diese Anzahl Bytes. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_analyze` (`boolean`)
  - `auto_explain.log_analyze` bewirkt, dass `EXPLAIN ANALYZE`-Ausgabe statt nur `EXPLAIN`-Ausgabe ausgegeben wird, wenn ein Ausführungsplan protokolliert wird. Dieser Parameter ist standardmäßig ausgeschaltet. Nur Superuser können diese Einstellung ändern.

  > **Hinweis:** Wenn dieser Parameter eingeschaltet ist, erfolgt für alle ausgeführten Anweisungen eine Zeitmessung pro Planknoten, unabhängig davon, ob sie lange genug laufen, um tatsächlich protokolliert zu werden. Das kann die Performance extrem negativ beeinflussen. Das Ausschalten von `auto_explain.log_timing` verringert die Performancekosten, allerdings auf Kosten weniger detaillierter Informationen.

- `auto_explain.log_buffers` (`boolean`)
  - `auto_explain.log_buffers` steuert, ob Statistiken zur Puffernutzung ausgegeben werden, wenn ein Ausführungsplan protokolliert wird. Das entspricht der Option `BUFFERS` von `EXPLAIN`. Dieser Parameter hat keine Wirkung, sofern `auto_explain.log_analyze` nicht aktiviert ist. Er ist standardmäßig ausgeschaltet. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_wal` (`boolean`)
  - `auto_explain.log_wal` steuert, ob Statistiken zur WAL-Nutzung ausgegeben werden, wenn ein Ausführungsplan protokolliert wird. Das entspricht der Option `WAL` von `EXPLAIN`. Dieser Parameter hat keine Wirkung, sofern `auto_explain.log_analyze` nicht aktiviert ist. Er ist standardmäßig ausgeschaltet. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_timing` (`boolean`)
  - `auto_explain.log_timing` steuert, ob Zeitinformationen pro Knoten ausgegeben werden, wenn ein Ausführungsplan protokolliert wird. Das entspricht der Option `TIMING` von `EXPLAIN`. Das wiederholte Lesen der Systemuhr kann Abfragen auf manchen Systemen erheblich verlangsamen. Daher kann es sinnvoll sein, diesen Parameter auf `off` zu setzen, wenn nur tatsächliche Zeilenzahlen und keine exakten Zeiten benötigt werden. Dieser Parameter hat keine Wirkung, sofern `auto_explain.log_analyze` nicht aktiviert ist. Er ist standardmäßig eingeschaltet. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_triggers` (`boolean`)
  - `auto_explain.log_triggers` bewirkt, dass Statistiken zur Triggerausführung einbezogen werden, wenn ein Ausführungsplan protokolliert wird. Dieser Parameter hat keine Wirkung, sofern `auto_explain.log_analyze` nicht aktiviert ist. Er ist standardmäßig ausgeschaltet. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_verbose` (`boolean`)
  - `auto_explain.log_verbose` steuert, ob ausführliche Details ausgegeben werden, wenn ein Ausführungsplan protokolliert wird. Das entspricht der Option `VERBOSE` von `EXPLAIN`. Dieser Parameter ist standardmäßig ausgeschaltet. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_settings` (`boolean`)
  - `auto_explain.log_settings` steuert, ob Informationen über geänderte Konfigurationsoptionen ausgegeben werden, wenn ein Ausführungsplan protokolliert wird. In die Ausgabe werden nur Optionen aufgenommen, die die Abfrageplanung beeinflussen und deren Wert vom eingebauten Standardwert abweicht. Dieser Parameter ist standardmäßig ausgeschaltet. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_format` (`enum`)
  - `auto_explain.log_format` wählt das zu verwendende Ausgabeformat von `EXPLAIN`. Zulässige Werte sind `text`, `xml`, `json` und `yaml`. Der Standardwert ist `text`. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_level` (`enum`)
  - `auto_explain.log_level` wählt die Protokollstufe, auf der `auto_explain` den Abfrageplan protokolliert. Gültige Werte sind `DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1`, `INFO`, `NOTICE`, `WARNING` und `LOG`. Der Standardwert ist `LOG`. Nur Superuser können diese Einstellung ändern.

- `auto_explain.log_nested_statements` (`boolean`)
  - `auto_explain.log_nested_statements` bewirkt, dass verschachtelte Anweisungen, also in einer Funktion ausgeführte Anweisungen, für die Protokollierung berücksichtigt werden. Wenn der Parameter ausgeschaltet ist, werden nur Pläne von Top-Level-Abfragen protokolliert. Dieser Parameter ist standardmäßig ausgeschaltet. Nur Superuser können diese Einstellung ändern.

- `auto_explain.sample_rate` (`real`)
  - `auto_explain.sample_rate` bewirkt, dass `auto_explain` nur einen Anteil der Anweisungen in jeder Sitzung erklärt. Der Standardwert ist `1`, was bedeutet, dass alle Abfragen erklärt werden. Bei verschachtelten Anweisungen werden entweder alle oder keine erklärt. Nur Superuser können diese Einstellung ändern.

In der üblichen Verwendung werden diese Parameter in `postgresql.conf` gesetzt, obwohl Superuser sie innerhalb ihrer eigenen Sitzungen auch zur Laufzeit ändern können. Eine typische Verwendung könnte so aussehen:

```conf
# postgresql.conf
session_preload_libraries = 'auto_explain'

auto_explain.log_min_duration = '3s'
```

#### F.3.2. Beispiel

```sql
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 0;
SET auto_explain.log_analyze = true;
SELECT count(*)
FROM pg_class, pg_index
WHERE oid = indrelid AND indisunique;
```

Dies könnte eine Protokollausgabe wie die folgende erzeugen:

```text
LOG: duration: 3.651 ms plan:
  Query Text: SELECT count(*)
              FROM pg_class, pg_index
              WHERE oid = indrelid AND indisunique;
  Aggregate (cost=16.79..16.80 rows=1 width=0) (actual time=3.626..3.627 rows=1.00 loops=1)
    -> Hash Join (cost=4.17..16.55 rows=92 width=0) (actual time=3.349..3.594 rows=92.00 loops=1)
          Hash Cond: (pg_class.oid = pg_index.indrelid)
          -> Seq Scan on pg_class (cost=0.00..9.55 rows=255 width=4) (actual time=0.016..0.140 rows=255.00 loops=1)
          -> Hash (cost=3.02..3.02 rows=92 width=4) (actual time=3.238..3.238 rows=92.00 loops=1)
                Buckets: 1024 Batches: 1 Memory Usage: 4kB
                -> Seq Scan on pg_index (cost=0.00..3.02 rows=92 width=4) (actual time=0.008..3.187 rows=92.00 loops=1)
                      Filter: indisunique
```

#### F.3.3. Autor

Takahiro Itagaki `<itagaki.takahiro@oss.ntt.co.jp>`

### F.4. basebackup_to_shell — Beispiel für ein `pg_basebackup`-Modul mit „shell“-Ziel

`basebackup_to_shell` fügt ein benutzerdefiniertes Basebackup-Ziel namens `shell` hinzu. Dadurch kann `pg_basebackup --target=shell` oder, abhängig von der Konfiguration dieses Moduls, `pg_basebackup --target=shell:DETAIL_STRING` ausgeführt werden. Für jedes vom Backup-Prozess erzeugte Tar-Archiv wird dann ein vom Serveradministrator gewählter Serverbefehl ausgeführt. Der Befehl erhält den Inhalt des Archivs über die Standardeingabe.

Dieses Modul ist hauptsächlich als Beispiel dafür gedacht, wie ein neues Backup-Ziel über ein Erweiterungsmodul erstellt werden kann. In manchen Szenarien kann es aber auch für sich genommen nützlich sein. Damit es funktioniert, muss es über `shared_preload_libraries` oder `local_preload_libraries` geladen werden.

#### F.4.1. Konfigurationsparameter

- `basebackup_to_shell.command` (`string`)
  - Der Befehl, den der Server für jedes vom Backup-Prozess erzeugte Archiv ausführen soll. Wenn `%f` in der Befehlszeichenkette vorkommt, wird es durch den Namen des Archivs ersetzt, zum Beispiel `base.tar`. Wenn `%d` in der Befehlszeichenkette vorkommt, wird es durch das vom Benutzer bereitgestellte Ziel-Detail ersetzt. Ein Ziel-Detail ist erforderlich, wenn `%d` in der Befehlszeichenkette verwendet wird, und andernfalls verboten. Aus Sicherheitsgründen darf es nur alphanumerische Zeichen enthalten. Wenn `%%` in der Befehlszeichenkette vorkommt, wird es durch ein einzelnes `%` ersetzt. Wenn `%` gefolgt von einem anderen Zeichen oder am Ende der Zeichenkette vorkommt, tritt ein Fehler auf.

- `basebackup_to_shell.required_role` (`string`)
  - Die Rolle, die erforderlich ist, um das Backup-Ziel `shell` zu verwenden. Wenn dieser Parameter nicht gesetzt ist, darf jeder Replikationsbenutzer das Backup-Ziel `shell` verwenden.

#### F.4.2. Autor

Robert Haas `<rhaas@postgresql.org>`

### F.5. basic_archive — Beispiel für ein WAL-Archivmodul

`basic_archive` ist ein Beispiel für ein Archivmodul. Dieses Modul kopiert abgeschlossene WAL-Segmentdateien in das angegebene Verzeichnis. Das ist möglicherweise nicht besonders nützlich, kann aber als Ausgangspunkt für die Entwicklung eines eigenen Archivmoduls dienen. Weitere Informationen zu Archivmodulen finden Sie in [Kapitel 49](49_Archivmodule.md).

Damit dieses Modul funktioniert, muss es über `archive_library` geladen und `archive_mode` aktiviert werden.

#### F.5.1. Konfigurationsparameter

- `basic_archive.archive_directory` (`string`)
  - Das Verzeichnis, in das der Server WAL-Segmentdateien kopieren soll. Dieses Verzeichnis muss bereits existieren. Der Standardwert ist eine leere Zeichenkette, was die WAL-Archivierung effektiv anhält. Wenn `archive_mode` aktiviert ist, sammelt der Server jedoch WAL-Segmentdateien in der Erwartung, dass bald ein Wert bereitgestellt wird.

Diese Parameter müssen in `postgresql.conf` gesetzt werden. Eine typische Verwendung könnte so aussehen:

```conf
# postgresql.conf
archive_mode = 'on'
archive_library = 'basic_archive'
basic_archive.archive_directory = '/path/to/archive/directory'
```

#### F.5.2. Hinweise

Serverabstürze können temporäre Dateien mit dem Präfix `archtemp` im Archivverzeichnis hinterlassen. Es wird empfohlen, solche Dateien zu löschen, bevor der Server nach einem Absturz neu gestartet wird. Es ist sicher, solche Dateien bei laufendem Server zu entfernen, solange sie nichts mit einer noch laufenden Archivierung zu tun haben. Benutzer sollten dabei jedoch besonders vorsichtig sein.

#### F.5.3. Autor

Nathan Bossart


### F.6. bloom — Indexzugriffsmethode mit Bloom-Filtern

`bloom` stellt eine Indexzugriffsmethode bereit, die auf Bloom-Filtern basiert.

<https://en.wikipedia.org/wiki/Bloom_filter>

Ein Bloom-Filter ist eine platzsparende Datenstruktur, mit der geprüft werden kann, ob ein Element zu einer Menge gehört. Im Fall einer Indexzugriffsmethode erlaubt er den schnellen Ausschluss nicht passender Tupel über Signaturen, deren Größe beim Erstellen des Index festgelegt wird.

Eine Signatur ist eine verlustbehaftete Darstellung der indizierten Attribute und neigt deshalb zu falsch positiven Treffern. Das heißt, es kann gemeldet werden, dass ein Element in der Menge enthalten ist, obwohl das nicht stimmt. Suchergebnisse aus dem Index müssen daher immer anhand der tatsächlichen Attributwerte aus dem Heap-Eintrag erneut geprüft werden. Größere Signaturen verringern die Wahrscheinlichkeit falsch positiver Treffer und damit die Zahl nutzloser Heap-Zugriffe, machen den Index aber natürlich auch größer und damit langsamer zu scannen.

Dieser Indextyp ist besonders nützlich, wenn eine Tabelle viele Attribute besitzt und Abfragen beliebige Kombinationen davon prüfen. Ein traditioneller B-Tree-Index ist schneller als ein Bloom-Index, aber zur Unterstützung aller möglichen Abfragen können viele B-Tree-Indizes erforderlich sein, während ein einziger Bloom-Index genügt. Beachten Sie jedoch, dass Bloom-Indizes nur Gleichheitsabfragen unterstützen, während B-Tree-Indizes auch Ungleichheits- und Bereichssuchen ausführen können.

#### F.6.1. Parameter

Ein Bloom-Index akzeptiert die folgenden Parameter in seiner `WITH`-Klausel:

- `length`
  - Länge jeder Signatur, also jedes Indexeintrags, in Bits. Der Wert wird auf das nächste Vielfache von 16 aufgerundet. Der Standardwert beträgt 80 Bit, der Maximalwert 4096.

- `col1` bis `col32`
  - Anzahl der Bits, die für jede Indexspalte erzeugt werden. Der Name jedes Parameters bezieht sich auf die Nummer der Indexspalte, die er steuert. Der Standardwert beträgt 2 Bit, der Maximalwert 4095. Parameter für Indexspalten, die tatsächlich nicht verwendet werden, werden ignoriert.

#### F.6.2. Beispiele

Dies ist ein Beispiel für das Erstellen eines Bloom-Index:

```sql
CREATE INDEX bloomidx ON tbloom USING bloom (i1,i2,i3)
       WITH (length=80, col1=2, col2=2, col3=4);
```

Der Index wird mit einer Signaturlänge von 80 Bit erstellt. Die Attribute `i1` und `i2` werden auf jeweils 2 Bit abgebildet, das Attribut `i3` auf 4 Bit. Die Angaben `length`, `col1` und `col2` hätten weggelassen werden können, da sie den Standardwerten entsprechen.

Das folgende umfangreichere Beispiel zeigt die Definition und Verwendung eines Bloom-Index sowie einen Vergleich mit entsprechenden B-Tree-Indizes. Der Bloom-Index ist deutlich kleiner als der B-Tree-Index und kann bessere Leistung liefern.

```sql
CREATE TABLE tbloom AS
   SELECT
     (random() * 1000000)::int as i1,
     (random() * 1000000)::int as i2,
     (random() * 1000000)::int as i3,
     (random() * 1000000)::int as i4,
     (random() * 1000000)::int as i5,
     (random() * 1000000)::int as i6
   FROM generate_series(1,10000000);
```

```text
SELECT 10000000
```

Ein sequenzieller Scan über diese große Tabelle dauert lange:

```sql
EXPLAIN ANALYZE SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
```

```text
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on tbloom (cost=0.00..213744.00 rows=250 width=24) (actual time=357.059..357.059 rows=0.00 loops=1)
   Filter: ((i2 = 898732) AND (i5 = 123451))
   Rows Removed by Filter: 10000000
   Buffers: shared hit=63744
 Planning Time: 0.346 ms
 Execution Time: 357.076 ms
(6 rows)
```

Auch mit einem definierten B-Tree-Index bleibt das Ergebnis ein sequenzieller Scan:

```sql
CREATE INDEX btreeidx ON tbloom (i1, i2, i3, i4, i5, i6);
SELECT pg_size_pretty(pg_relation_size('btreeidx'));
EXPLAIN ANALYZE SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
```

```text
CREATE INDEX
 pg_size_pretty
----------------
 386 MB
(1 row)

                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on tbloom (cost=0.00..213744.00 rows=2 width=24) (actual time=351.016..351.017 rows=0.00 loops=1)
   Filter: ((i2 = 898732) AND (i5 = 123451))
   Rows Removed by Filter: 10000000
   Buffers: shared hit=63744
 Planning Time: 0.138 ms
 Execution Time: 351.035 ms
(6 rows)
```

Wenn für die Tabelle ein Bloom-Index definiert ist, wird diese Art von Suche besser behandelt als mit dem B-Tree-Index:

```sql
CREATE INDEX bloomidx ON tbloom USING bloom (i1, i2, i3, i4, i5, i6);
SELECT pg_size_pretty(pg_relation_size('bloomidx'));
EXPLAIN ANALYZE SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
```

```text
CREATE INDEX
 pg_size_pretty
----------------
 153 MB
(1 row)

                                                      QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on tbloom (cost=1792.00..1799.69 rows=2 width=24) (actual time=22.605..22.606 rows=0.00 loops=1)
   Recheck Cond: ((i2 = 898732) AND (i5 = 123451))
   Rows Removed by Index Recheck: 2300
   Heap Blocks: exact=2256
   Buffers: shared hit=21864
   -> Bitmap Index Scan on bloomidx (cost=0.00..178436.00 rows=1 width=0) (actual time=20.005..20.005 rows=2300.00 loops=1)
         Index Cond: ((i2 = 898732) AND (i5 = 123451))
         Index Searches: 1
         Buffers: shared hit=19608
 Planning Time: 0.099 ms
 Execution Time: 22.632 ms
(11 rows)
```

Das Hauptproblem bei der B-Tree-Suche ist, dass B-Tree ineffizient ist, wenn die Suchbedingungen die führenden Indexspalten nicht einschränken. Eine bessere Strategie für B-Tree besteht darin, für jede Spalte einen eigenen Index zu erstellen. Dann wählt der Planner etwa Folgendes:

```sql
CREATE INDEX btreeidx1 ON tbloom (i1);
CREATE INDEX btreeidx2 ON tbloom (i2);
CREATE INDEX btreeidx3 ON tbloom (i3);
CREATE INDEX btreeidx4 ON tbloom (i4);
CREATE INDEX btreeidx5 ON tbloom (i5);
CREATE INDEX btreeidx6 ON tbloom (i6);
EXPLAIN ANALYZE SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
```

```text
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
                                                         QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on tbloom (cost=9.29..13.30 rows=1 width=24) (actual time=0.032..0.033 rows=0.00 loops=1)
   Recheck Cond: ((i5 = 123451) AND (i2 = 898732))
   Buffers: shared read=6
   -> BitmapAnd (cost=9.29..9.29 rows=1 width=0) (actual time=0.047..0.047 rows=0.00 loops=1)
         Buffers: shared hit=6
         -> Bitmap Index Scan on btreeidx5 (cost=0.00..4.52 rows=11 width=0) (actual time=0.026..0.026 rows=7.00 loops=1)
               Index Cond: (i5 = 123451)
               Index Searches: 1
               Buffers: shared hit=3
         -> Bitmap Index Scan on btreeidx2 (cost=0.00..4.52 rows=11 width=0) (actual time=0.007..0.007 rows=8.00 loops=1)
               Index Cond: (i2 = 898732)
               Index Searches: 1
               Buffers: shared hit=3
 Planning Time: 0.264 ms
 Execution Time: 0.047 ms
(15 rows)
```

Obwohl diese Abfrage deutlich schneller läuft als mit einem der einzelnen mehrspaltigen Indizes, zahlen wir mit größerem Indexspeicherplatz. Jeder der einspaltigen B-Tree-Indizes belegt 88,5 MB, sodass insgesamt 531 MB benötigt werden, also mehr als dreimal so viel Speicherplatz wie beim Bloom-Index.

#### F.6.3. Operator-Klassen-Schnittstelle

Eine Operatorklasse für Bloom-Indizes benötigt nur eine Hash-Funktion für den indizierten Datentyp und einen Gleichheitsoperator für die Suche. Dieses Beispiel zeigt die Definition der Operatorklasse für den Datentyp `text`:

```sql
CREATE OPERATOR CLASS text_ops
DEFAULT FOR TYPE text USING bloom AS
    OPERATOR    1   =(text, text),
    FUNCTION    1   hashtext(text);
```

#### F.6.4. Einschränkungen

- Mit dem Modul werden nur Operatorklassen für `int4` und `text` mitgeliefert.

- Für die Suche wird nur der Operator `=` unterstützt. Es ist jedoch möglich, zukünftig Unterstützung für Arrays mit Vereinigungs- und Schnittmengenoperationen hinzuzufügen.

- Die Zugriffsmethode `bloom` unterstützt keine `UNIQUE`-Indizes.

- Die Zugriffsmethode `bloom` unterstützt keine Suche nach `NULL`-Werten.

#### F.6.5. Autoren

Teodor Sigaev `<teodor@postgrespro.ru>`, Postgres Professional, Moskau, Russland

Alexander Korotkov `<a.korotkov@postgrespro.ru>`, Postgres Professional, Moskau, Russland

Oleg Bartunov `<obartunov@postgrespro.ru>`, Postgres Professional, Moskau, Russland


### F.7. btree_gin — GIN-Operatorklassen mit B-Tree-Verhalten

`btree_gin` stellt GIN-Operatorklassen bereit, die B-Tree-äquivalentes Verhalten für die Datentypen `int2`, `int4`, `int8`, `float4`, `float8`, `timestamp with time zone`, `timestamp without time zone`, `time with time zone`, `time without time zone`, `date`, `interval`, `oid`, `money`, `"char"`, `varchar`, `text`, `bytea`, `bit`, `varbit`, `macaddr`, `macaddr8`, `inet`, `cidr`, `uuid`, `name`, `bool`, `bpchar` und alle Enum-Typen implementieren.

Im Allgemeinen sind diese Operatorklassen nicht schneller als die entsprechenden Standard-B-Tree-Indexmethoden. Außerdem fehlt ihnen ein wichtiges Merkmal des Standard-B-Tree-Codes: die Fähigkeit, Eindeutigkeit zu erzwingen. Sie sind jedoch nützlich zum Testen von GIN und als Grundlage für die Entwicklung anderer GIN-Operatorklassen. Außerdem kann es bei Abfragen, die sowohl eine GIN-indizierbare Spalte als auch eine B-Tree-indizierbare Spalte prüfen, effizienter sein, einen mehrspaltigen GIN-Index mit einer dieser Operatorklassen zu erstellen, statt zwei getrennte Indizes zu erstellen, die anschließend per Bitmap-AND kombiniert werden müssten.

Dieses Modul gilt als „trusted“, das heißt, es kann von Nicht-Superusern installiert werden, die das Privileg `CREATE` in der aktuellen Datenbank besitzen.

#### F.7.1. Beispielverwendung

```sql
CREATE TABLE test (a int4);
-- create index
CREATE INDEX testidx ON test USING GIN (a);
-- query
SELECT * FROM test WHERE a < 10;
```

#### F.7.2. Autoren

Teodor Sigaev `<teodor@stack.net>` und Oleg Bartunov `<oleg@sai.msu.su>`.

Weitere Informationen finden Sie unter:

<http://www.sai.msu.su/~megera/oddmuse/index.cgi/Gin>

### F.8. btree_gist — GiST-Operatorklassen mit B-Tree-Verhalten

`btree_gist` stellt GiST-Index-Operatorklassen bereit, die B-Tree-äquivalentes Verhalten für die Datentypen `int2`, `int4`, `int8`, `float4`, `float8`, `numeric`, `timestamp with time zone`, `timestamp without time zone`, `time with time zone`, `time without time zone`, `date`, `interval`, `oid`, `money`, `char`, `varchar`, `text`, `bytea`, `bit`, `varbit`, `macaddr`, `macaddr8`, `inet`, `cidr`, `uuid`, `bool` und alle Enum-Typen implementieren.

Im Allgemeinen sind diese Operatorklassen nicht schneller als die entsprechenden Standard-B-Tree-Indexmethoden. Außerdem fehlt ihnen ein wichtiges Merkmal des Standard-B-Tree-Codes: die Fähigkeit, Eindeutigkeit zu erzwingen. Sie stellen jedoch einige andere Funktionen bereit, die mit einem B-Tree-Index nicht verfügbar sind, wie unten beschrieben. Außerdem sind diese Operatorklassen nützlich, wenn ein mehrspaltiger GiST-Index benötigt wird, bei dem einige Spalten Datentypen verwenden, die nur mit GiST indizierbar sind, während andere Spalten einfache Datentypen sind. Schließlich sind diese Operatorklassen nützlich zum Testen von GiST und als Grundlage für die Entwicklung anderer GiST-Operatorklassen.

Zusätzlich zu den typischen B-Tree-Suchoperatoren stellt `btree_gist` auch Indexunterstützung für `<>` („ungleich“) bereit. Dies kann in Kombination mit einem Ausschluss-Constraint nützlich sein, wie unten beschrieben.

Für Datentypen, für die es eine natürliche Distanzmetrik gibt, definiert `btree_gist` außerdem einen Distanzoperator `<->` und stellt GiST-Indexunterstützung für Nächste-Nachbarn-Suchen mit diesem Operator bereit. Distanzoperatoren werden für `int2`, `int4`, `int8`, `float4`, `float8`, `timestamp with time zone`, `timestamp without time zone`, `time without time zone`, `date`, `interval`, `oid` und `money` bereitgestellt.

Standardmäßig baut `btree_gist` GiST-Indizes mit Sortierunterstützung im sortierten Modus. Das führt normalerweise zu einem deutlich schnelleren Indexaufbau. Es ist weiterhin möglich, zur gepufferten Aufbaustrategie zurückzukehren, indem beim Erstellen des Index der Parameter `buffering` verwendet wird.

Dieses Modul gilt als „trusted“, das heißt, es kann von Nicht-Superusern installiert werden, die das Privileg `CREATE` in der aktuellen Datenbank besitzen.

#### F.8.1. Beispielverwendung

Ein einfaches Beispiel, das `btree_gist` statt `btree` verwendet:

```sql
CREATE TABLE test (a int4);
-- create index
CREATE INDEX testidx ON test USING GIST (a);
-- query
SELECT * FROM test WHERE a < 10;
-- nearest-neighbor search: find the ten entries closest to "42"
SELECT *, a <-> 42 AS dist FROM test ORDER BY a <-> 42 LIMIT 10;
```

Ein Ausschluss-Constraint kann verwendet werden, um die Regel zu erzwingen, dass ein Käfig in einem Zoo nur eine Tierart enthalten darf:

```sql
CREATE TABLE zoo (
   cage   INTEGER,
   animal TEXT,
   EXCLUDE USING GIST (cage WITH =, animal WITH <>)
);

INSERT INTO zoo VALUES(123, 'zebra');
INSERT INTO zoo VALUES(123, 'zebra');
INSERT INTO zoo VALUES(123, 'lion');
INSERT INTO zoo VALUES(124, 'lion');
```

Die dritte `INSERT`-Anweisung schlägt fehl, weil sie mit dem vorhandenen Eintrag für denselben Käfig und ein anderes Tier kollidiert:

```text
INSERT 0 1
INSERT 0 1
ERROR: conflicting key value violates exclusion constraint "zoo_cage_animal_excl"
DETAIL: Key (cage, animal)=(123, lion) conflicts with existing key (cage, animal)=(123, zebra).
INSERT 0 1
```

#### F.8.2. Autoren

Teodor Sigaev `<teodor@stack.net>`, Oleg Bartunov `<oleg@sai.msu.su>`, Janko Richter `<jankorichter@yahoo.de>` und Paul Jungwirth `<pj@illuminatedcomputing.com>`.

Weitere Informationen finden Sie unter:

<http://www.sai.msu.su/~megera/postgres/gist/>


### F.9. citext — ein zeichenkettenbasierter Typ ohne Beachtung der Groß-/Kleinschreibung

Das Modul `citext` stellt einen Zeichenkettentyp `citext` bereit, der Groß- und Kleinschreibung nicht unterscheidet. Im Wesentlichen ruft er intern `lower` auf, wenn Werte verglichen werden. Ansonsten verhält er sich fast genau wie `text`.

> **Tipp:** Ziehen Sie statt dieses Moduls nichtdeterministische Collations in Betracht; siehe [Abschnitt 23.2.2.4](23_Lokalisierung.md#23224-nichtdeterministische-collations). Sie können für Vergleiche ohne Beachtung der Groß-/Kleinschreibung, für akzentunempfindliche Vergleiche und andere Kombinationen verwendet werden und behandeln mehr Unicode-Sonderfälle korrekt.

Dieses Modul gilt als „trusted“, das heißt, es kann von Nicht-Superusern installiert werden, die das Privileg `CREATE` in der aktuellen Datenbank besitzen.

#### F.9.1. Begründung

Der Standardansatz für Vergleiche ohne Beachtung der Groß-/Kleinschreibung in PostgreSQL bestand darin, beim Vergleichen von Werten die Funktion `lower` zu verwenden, zum Beispiel:

```sql
SELECT * FROM tab WHERE lower(col) = LOWER(?);
```

Das funktioniert recht gut, hat aber mehrere Nachteile:

- Es macht SQL-Anweisungen ausführlicher, und Sie müssen immer daran denken, `lower` sowohl auf die Spalte als auch auf den Abfragewert anzuwenden.

- Es verwendet keinen Index, sofern Sie nicht einen funktionalen Index mit `lower` erstellen.

- Wenn Sie eine Spalte als `UNIQUE` oder `PRIMARY KEY` deklarieren, ist der implizit erzeugte Index abhängig von Groß-/Kleinschreibung. Er ist daher für Suchen ohne Beachtung der Groß-/Kleinschreibung nutzlos und erzwingt Eindeutigkeit nicht unabhängig von Groß-/Kleinschreibung.

Der Datentyp `citext` erlaubt es, Aufrufe von `lower` in SQL-Abfragen zu vermeiden, und ermöglicht einen Primärschlüssel ohne Beachtung der Groß-/Kleinschreibung. `citext` ist wie `text` locale-bewusst. Das bedeutet, dass die Zuordnung von Groß- und Kleinbuchstaben von den Regeln der Datenbankeinstellung `LC_CTYPE` abhängt. Dieses Verhalten ist wiederum identisch mit der Verwendung von `lower` in Abfragen. Da es aber transparent durch den Datentyp erfolgt, müssen Sie in Ihren Abfragen an nichts Besonderes denken.

#### F.9.2. Verwendung

Hier ist ein einfaches Verwendungsbeispiel:

```sql
CREATE TABLE users (
    nick CITEXT PRIMARY KEY,
    pass TEXT   NOT NULL
);

INSERT INTO users VALUES ('larry', sha256(random()::text::bytea));
INSERT INTO users VALUES ('Tom', sha256(random()::text::bytea));
INSERT INTO users VALUES ('Damian', sha256(random()::text::bytea));
INSERT INTO users VALUES ('NEAL', sha256(random()::text::bytea));
INSERT INTO users VALUES ('Bjørn', sha256(random()::text::bytea));

SELECT * FROM users WHERE nick = 'Larry';
```

Die `SELECT`-Anweisung gibt ein Tupel zurück, obwohl die Spalte `nick` auf `larry` gesetzt wurde und die Abfrage nach `Larry` sucht.

#### F.9.3. Verhalten beim Zeichenkettenvergleich

`citext` führt Vergleiche aus, indem jede Zeichenkette in Kleinbuchstaben umgewandelt wird, so als wäre `lower` aufgerufen worden, und die Ergebnisse anschließend normal verglichen werden. Zwei Zeichenketten gelten also zum Beispiel als gleich, wenn `lower` für beide identische Ergebnisse erzeugen würde.

Um eine Collation ohne Beachtung der Groß-/Kleinschreibung möglichst genau nachzubilden, gibt es `citext`-spezifische Versionen mehrerer Operatoren und Funktionen zur Zeichenkettenverarbeitung. So zeigen zum Beispiel die regulären Ausdrucksoperatoren `~` und `~*` dasselbe Verhalten, wenn sie auf `citext` angewendet werden: Beide vergleichen ohne Beachtung der Groß-/Kleinschreibung. Dasselbe gilt für `!~` und `!~*` sowie für die `LIKE`-Operatoren `~~` und `~~*` sowie `!~~` und `!~~*`. Wenn Sie abhängig von Groß-/Kleinschreibung vergleichen möchten, können Sie die Argumente des Operators nach `text` casten.

Entsprechend führen alle folgenden Funktionen Vergleiche ohne Beachtung der Groß-/Kleinschreibung aus, wenn ihre Argumente vom Typ `citext` sind:

- `regexp_match()`
- `regexp_matches()`
- `regexp_replace()`
- `regexp_split_to_array()`
- `regexp_split_to_table()`
- `replace()`
- `split_part()`
- `strpos()`
- `translate()`

Bei den `regexp`-Funktionen können Sie das Flag `c` angeben, um einen Vergleich mit Beachtung der Groß-/Kleinschreibung zu erzwingen. Andernfalls müssen Sie nach `text` casten, bevor Sie eine dieser Funktionen verwenden, wenn Sie abhängig von Groß-/Kleinschreibung arbeiten möchten.

#### F.9.4. Einschränkungen

- Das Case-Folding-Verhalten von `citext` hängt von der Einstellung `LC_CTYPE` Ihrer Datenbank ab. Wie Werte verglichen werden, wird daher beim Erstellen der Datenbank festgelegt. Im Sinne des Unicode-Standards ist es nicht wirklich unabhängig von Groß-/Kleinschreibung. Praktisch bedeutet das: Solange Sie mit Ihrer Collation zufrieden sind, sollten Sie auch mit den Vergleichen von `citext` zufrieden sein. Wenn Sie jedoch Daten in verschiedenen Sprachen in Ihrer Datenbank speichern, können Benutzer einer Sprache unerwartete Abfrageergebnisse erhalten, wenn die Collation für eine andere Sprache bestimmt ist.

- Seit PostgreSQL 9.1 können Sie an `citext`-Spalten oder -Datenwerte eine `COLLATE`-Spezifikation anhängen. Derzeit berücksichtigen `citext`-Operatoren beim Vergleichen in Kleinbuchstaben umgewandelter Zeichenketten eine nicht standardmäßige `COLLATE`-Spezifikation. Das anfängliche Umwandeln in Kleinbuchstaben erfolgt jedoch immer gemäß der Datenbankeinstellung `LC_CTYPE`, also so, als wäre `COLLATE "default"` angegeben. Dies kann in einer zukünftigen Version geändert werden, sodass beide Schritte der `COLLATE`-Spezifikation der Eingabe folgen.

- `citext` ist nicht so effizient wie `text`, weil die Operatorfunktionen und die B-Tree-Vergleichsfunktionen Kopien der Daten erstellen und sie für Vergleiche in Kleinbuchstaben umwandeln müssen. Außerdem kann nur `text` B-Tree-Deduplizierung unterstützen. `citext` ist jedoch etwas effizienter, als mit `lower` Vergleiche ohne Beachtung der Groß-/Kleinschreibung zu erzwingen.

- `citext` hilft wenig, wenn Daten in manchen Kontexten abhängig von Groß-/Kleinschreibung und in anderen Kontexten unabhängig davon verglichen werden müssen. Die Standardlösung besteht darin, den Typ `text` zu verwenden und `lower` manuell einzusetzen, wenn ein Vergleich ohne Beachtung der Groß-/Kleinschreibung benötigt wird. Das funktioniert gut, wenn solche Vergleiche nur selten erforderlich sind. Wenn Sie meistens unabhängiges und nur selten abhängiges Verhalten benötigen, können Sie die Daten als `citext` speichern und die Spalte explizit nach `text` casten, wenn Sie abhängig von Groß-/Kleinschreibung vergleichen möchten. In beiden Fällen benötigen Sie zwei Indizes, wenn beide Sucharten schnell sein sollen.

- Das Schema, das die `citext`-Operatoren enthält, muss im aktuellen `search_path` stehen, typischerweise `public`. Andernfalls werden stattdessen die normalen, von Groß-/Kleinschreibung abhängigen `text`-Operatoren aufgerufen.

- Der Ansatz, Zeichenketten zum Vergleich in Kleinbuchstaben umzuwandeln, behandelt einige Unicode-Sonderfälle nicht korrekt, zum Beispiel wenn ein Großbuchstabe zwei Kleinbuchstaben-Entsprechungen besitzt. Unicode unterscheidet aus diesem Grund zwischen Case Mapping und Case Folding. Verwenden Sie statt `citext` nichtdeterministische Collations, um dies korrekt zu behandeln.

#### F.9.5. Autor

David E. Wheeler `<david@kineticode.com>`

Inspiriert vom ursprünglichen `citext`-Modul von Donald Fraser.


### F.10. cube — ein mehrdimensionaler Cube-Datentyp

Dieses Modul implementiert den Datentyp `cube` zur Darstellung mehrdimensionaler Cubes.

Dieses Modul gilt als „trusted“, das heißt, es kann von Nicht-Superusern installiert werden, die das Privileg `CREATE` in der aktuellen Datenbank besitzen.

#### F.10.1. Syntax

Tabelle F.1 zeigt die gültigen externen Darstellungen für den Typ `cube`. `x`, `y` usw. bezeichnen Gleitkommazahlen.

**Tabelle F.1. Externe Darstellungen von Cubes**

| Externe Syntax | Bedeutung |
| --- | --- |
| `x` | Ein eindimensionaler Punkt oder ein eindimensionales Intervall der Länge null. |
| `(x)` | Dasselbe wie oben. |
| `x1,x2,...,xn` | Ein Punkt im n-dimensionalen Raum, intern als Cube mit Volumen null dargestellt. |
| `(x1,x2,...,xn)` | Dasselbe wie oben. |
| `(x),(y)` | Ein eindimensionales Intervall, das bei `x` beginnt und bei `y` endet oder umgekehrt; die Reihenfolge spielt keine Rolle. |
| `[(x),(y)]` | Dasselbe wie oben. |
| `(x1,...,xn),(y1,...,yn)` | Ein n-dimensionaler Cube, dargestellt durch ein Paar diagonal gegenüberliegender Ecken. |
| `[(x1,...,xn),(y1,...,yn)]` | Dasselbe wie oben. |

Es spielt keine Rolle, in welcher Reihenfolge die gegenüberliegenden Ecken eines Cubes eingegeben werden. Die Cube-Funktionen vertauschen Werte bei Bedarf automatisch, um eine einheitliche interne Darstellung „unten links bis oben rechts“ zu erzeugen. Wenn die Ecken zusammenfallen, speichert `cube` nur eine Ecke zusammen mit einem „is point“-Flag, um keinen Speicherplatz zu verschwenden.

Leerraum wird bei der Eingabe ignoriert, daher ist `[(x),(y)]` dasselbe wie `[ ( x ), ( y ) ]`.

#### F.10.2. Genauigkeit

Werte werden intern als 64-Bit-Gleitkommazahlen gespeichert. Das bedeutet, dass Zahlen mit mehr als ungefähr 16 signifikanten Stellen abgeschnitten werden.

#### F.10.3. Verwendung

Tabelle F.2 zeigt die spezialisierten Operatoren, die für den Typ `cube` bereitgestellt werden.

**Tabelle F.2. Cube-Operatoren**

| Operator | Beschreibung |
| --- | --- |
| `cube && cube` → `boolean` | Überlappen sich die Cubes? |
| `cube @> cube` → `boolean` | Enthält der erste Cube den zweiten? |
| `cube <@ cube` → `boolean` | Ist der erste Cube im zweiten enthalten? |
| `cube -> integer` → `float8` | Extrahiert die n-te Koordinate des Cubes, gezählt ab 1. |
| `cube ~> integer` → `float8` | Extrahiert die n-te Koordinate des Cubes mit folgender Zählweise: `n = 2 * k - 1` bezeichnet die untere Grenze der k-ten Dimension, `n = 2 * k` die obere Grenze der k-ten Dimension. Ein negatives `n` bezeichnet den inversen Wert der entsprechenden positiven Koordinate. Dieser Operator ist für KNN-GiST-Unterstützung gedacht. |
| `cube <-> cube` → `float8` | Berechnet die euklidische Entfernung zwischen zwei Cubes. |
| `cube <#> cube` → `float8` | Berechnet die Taxicab-Entfernung, also die L1-Metrik, zwischen zwei Cubes. |
| `cube <=> cube` → `float8` | Berechnet die Tschebyscheff-Entfernung, also die L∞-Metrik, zwischen zwei Cubes. |

Zusätzlich zu den oben genannten Operatoren stehen für den Typ `cube` die üblichen Vergleichsoperatoren aus Tabelle 9.1 zur Verfügung. Diese Operatoren vergleichen zuerst die ersten Koordinaten, und wenn diese gleich sind, die zweiten Koordinaten usw. Sie existieren hauptsächlich zur Unterstützung der B-Tree-Index-Operatorklasse für `cube`, was zum Beispiel nützlich sein kann, wenn Sie einen `UNIQUE`-Constraint auf einer `cube`-Spalte möchten. Ansonsten ist diese Sortierreihenfolge praktisch wenig nützlich.

Das Modul `cube` stellt außerdem eine GiST-Index-Operatorklasse für `cube`-Werte bereit. Ein GiST-Index auf `cube` kann verwendet werden, um in `WHERE`-Klauseln mit den Operatoren `=`, `&&`, `@>` und `<@` nach Werten zu suchen.

Außerdem kann ein GiST-Index auf `cube` verwendet werden, um mit den metrischen Operatoren `<->`, `<#>` und `<=>` in `ORDER BY`-Klauseln nächste Nachbarn zu finden. Der nächste Nachbar des 3-D-Punkts `(0.5, 0.5, 0.5)` kann zum Beispiel effizient so gefunden werden:

```sql
SELECT c FROM test ORDER BY c <-> cube(array[0.5,0.5,0.5]) LIMIT 1;
```

Der Operator `~>` kann ebenfalls auf diese Weise verwendet werden, um effizient die ersten Werte nach einer ausgewählten Koordinate sortiert abzurufen. Um zum Beispiel die ersten Cubes aufsteigend nach der ersten Koordinate, also der unteren linken Ecke, zu erhalten, können Sie folgende Abfrage verwenden:

```sql
SELECT c FROM test ORDER BY c ~> 1 LIMIT 5;
```

Und um 2-D-Cubes absteigend nach der ersten Koordinate der oberen rechten Ecke zu erhalten:

```sql
SELECT c FROM test ORDER BY c ~> 3 DESC LIMIT 5;
```

Tabelle F.3 zeigt die verfügbaren Funktionen.

**Tabelle F.3. Cube-Funktionen**

| Funktion | Beschreibung | Beispiel |
| --- | --- | --- |
| `cube(float8)` → `cube` | Erzeugt einen eindimensionalen Cube, bei dem beide Koordinaten gleich sind. | `cube(1)` → `(1)` |
| `cube(float8, float8)` → `cube` | Erzeugt einen eindimensionalen Cube. | `cube(1, 2)` → `(1),(2)` |
| `cube(float8[])` → `cube` | Erzeugt einen Cube mit Volumen null aus den im Array definierten Koordinaten. | `cube(ARRAY[1,2,3])` → `(1, 2, 3)` |
| `cube(float8[], float8[])` → `cube` | Erzeugt einen Cube mit oberer rechter und unterer linker Koordinate, wie sie durch die zwei Arrays definiert sind. Die Arrays müssen dieselbe Länge haben. | `cube(ARRAY[1,2], ARRAY[3,4])` → `(1, 2),(3, 4)` |
| `cube(cube, float8)` → `cube` | Erzeugt einen neuen Cube, indem einem vorhandenen Cube eine Dimension hinzugefügt wird; beide Endpunkte der neuen Koordinate erhalten denselben Wert. Das ist nützlich, um Cubes Stück für Stück aus berechneten Werten aufzubauen. | `cube('(1,2),(3,4)'::cube, 5)` → `(1, 2, 5),(3, 4, 5)` |
| `cube(cube, float8, float8)` → `cube` | Erzeugt einen neuen Cube, indem einem vorhandenen Cube eine Dimension hinzugefügt wird. Das ist nützlich, um Cubes Stück für Stück aus berechneten Werten aufzubauen. | `cube('(1,2),(3,4)'::cube, 5, 6)` → `(1, 2, 5),(3, 4, 6)` |
| `cube_dim(cube)` → `integer` | Gibt die Anzahl der Dimensionen des Cubes zurück. | `cube_dim('(1,2),(3,4)')` → `2` |
| `cube_ll_coord(cube, integer)` → `float8` | Gibt den n-ten Koordinatenwert der unteren linken Ecke des Cubes zurück. | `cube_ll_coord('(1,2),(3,4)', 2)` → `2` |
| `cube_ur_coord(cube, integer)` → `float8` | Gibt den n-ten Koordinatenwert der oberen rechten Ecke des Cubes zurück. | `cube_ur_coord('(1,2),(3,4)', 2)` → `4` |
| `cube_is_point(cube)` → `boolean` | Gibt `true` zurück, wenn der Cube ein Punkt ist, also wenn die beiden definierenden Ecken gleich sind. | `cube_is_point(cube(1,1))` → `t` |
| `cube_distance(cube, cube)` → `float8` | Gibt die Entfernung zwischen zwei Cubes zurück. Wenn beide Cubes Punkte sind, ist dies die normale Distanzfunktion. | `cube_distance('(1,2)', '(3,4)')` → `2.8284271247461903` |
| `cube_subset(cube, integer[])` → `cube` | Erzeugt aus einem vorhandenen Cube einen neuen Cube und verwendet dabei eine Liste von Dimensionsindizes aus einem Array. Damit können die Endpunkte einer einzelnen Dimension extrahiert, Dimensionen entfernt oder Dimensionen nach Wunsch neu angeordnet werden. | `cube_subset(cube('(1,3,5),(6,7,8)'), ARRAY[2])` → `(3),(7)`; `cube_subset(cube('(1,3,5),(6,7,8)'), ARRAY[3,2,1,1])` → `(5, 3, 1, 1),(8, 7, 6, 6)` |
| `cube_union(cube, cube)` → `cube` | Erzeugt die Vereinigung zweier Cubes. | `cube_union('(1,2)', '(3,4)')` → `(1, 2),(3, 4)` |
| `cube_inter(cube, cube)` → `cube` | Erzeugt die Schnittmenge zweier Cubes. | `cube_inter('(1,2)', '(3,4)')` → `(3, 4),(1, 2)` |
| `cube_enlarge(c cube, r double, n integer)` → `cube` | Vergrößert den Cube um den angegebenen Radius `r` in mindestens `n` Dimensionen. Wenn der Radius negativ ist, wird der Cube stattdessen verkleinert. Alle definierten Dimensionen werden um den Radius `r` geändert: Untere linke Koordinaten werden um `r` verringert und obere rechte Koordinaten um `r` erhöht. Wenn eine untere linke Koordinate über die entsprechende obere rechte Koordinate hinaus erhöht wird, was nur bei `r < 0` passieren kann, werden beide Koordinaten auf ihren Durchschnitt gesetzt. Wenn `n` größer ist als die Anzahl definierter Dimensionen und der Cube vergrößert wird (`r > 0`), werden zusätzliche Dimensionen hinzugefügt, bis insgesamt `n` vorhanden sind; `0` wird als Anfangswert für die zusätzlichen Koordinaten verwendet. Diese Funktion ist nützlich, um Bounding Boxes um einen Punkt zu erzeugen und nach nahen Punkten zu suchen. | `cube_enlarge('(1,2),(3,4)', 0.5, 3)` → `(0.5, 1.5, -0.5),(3.5, 4.5, 0.5)` |

#### F.10.4. Voreinstellungen

Diese Vereinigung:

```sql
SELECT cube_union('(0,5,2),(2,3,1)', '0');
```

```text
     cube_union
-------------------
 (0, 0, 0),(2, 5, 2)
(1 row)
```

widerspricht nicht dem gesunden Menschenverstand, ebenso wenig wie die Schnittmenge:

```sql
SELECT cube_inter('(0,-1),(1,1)', '(-2),(2)');
```

```text
 cube_inter
-------------
 (0, 0),(1, 0)
(1 row)
```

Bei allen binären Operationen auf Cubes unterschiedlicher Dimension wird angenommen, dass der niedriger dimensionierte Cube eine kartesische Projektion ist, also Nullen an den Stellen besitzt, deren Koordinaten in der Zeichenkettendarstellung ausgelassen wurden. Die obigen Beispiele entsprechen:

```sql
SELECT cube_union('(0,5,2),(2,3,1)','(0,0,0),(0,0,0)');
SELECT cube_inter('(0,-1),(1,1)','(-2,0),(2,0)');
```

Das folgende Enthaltensein-Prädikat verwendet die Punktsyntax, obwohl das zweite Argument intern tatsächlich durch eine Box dargestellt wird. Diese Syntax macht es unnötig, einen eigenen Punkttyp und Funktionen für `(box, point)`-Prädikate zu definieren.

```sql
SELECT cube_contains('(0,0),(1,1)', '0.5,0.5');
```

```text
 cube_contains
--------------
 t
(1 row)
```

#### F.10.5. Hinweise

Beispiele zur Verwendung finden Sie im Regressionstest `sql/cube.sql`.

Um es schwerer zu machen, Dinge kaputtzumachen, gibt es eine Grenze von 100 für die Anzahl der Dimensionen von Cubes. Diese Grenze ist in `cubedata.h` gesetzt, falls Sie etwas Größeres benötigen.

#### F.10.6. Danksagung

Ursprünglicher Autor: Gene Selkov, Jr. `<selkovjr@mcs.anl.gov>`, Mathematics and Computer Science Division, Argonne National Laboratory.

Mein Dank gilt vor allem Prof. Joe Hellerstein (https://dsf.berkeley.edu/jmh/) für die Erläuterung des Kerns von GiST (http://gist.cs.berkeley.edu/) und seinem ehemaligen Studenten Andy Dong für sein für Illustra geschriebenes Beispiel. Außerdem bin ich allen PostgreSQL-Entwicklern, gegenwärtigen wie früheren, dankbar, weil sie es mir ermöglicht haben, meine eigene Welt zu erschaffen und ungestört darin zu leben. Ebenso möchte ich Argonne Lab und dem U.S. Department of Energy meinen Dank für die jahrelange verlässliche Unterstützung meiner Datenbankforschung aussprechen.

Kleinere Aktualisierungen an diesem Paket wurden im August/September 2002 von Bruno Wolff III `<bruno@wolff.to>` vorgenommen. Dazu gehörten die Umstellung der Genauigkeit von einfacher auf doppelte Genauigkeit und das Hinzufügen einiger neuer Funktionen.

Weitere Aktualisierungen wurden im Juli 2006 von Joshua Reich `<josh@root.net>` vorgenommen. Dazu gehörten `cube(float8[], float8[])` und eine Bereinigung des Codes, sodass das V1-Aufrufprotokoll statt des veralteten V0-Protokolls verwendet wird.


### F.11. dblink — Verbindung zu anderen PostgreSQL-Datenbanken

`dblink` ist ein Modul, das innerhalb einer Datenbanksitzung Verbindungen zu anderen PostgreSQL-Datenbanken unterstützt.

`dblink` kann die folgenden Wait Events unter dem Wait-Event-Typ `Extension` melden:

- `DblinkConnect`
  - Warten darauf, eine Verbindung zu einem entfernten Server herzustellen.

- `DblinkGetConnect`
  - Warten darauf, eine Verbindung zu einem entfernten Server herzustellen, wenn sie nicht in der Liste bereits geöffneter Verbindungen gefunden wurde.

- `DblinkGetResult`
  - Warten darauf, die Ergebnisse einer Abfrage von einem entfernten Server zu empfangen.

Siehe auch `postgres_fdw`, das ungefähr dieselbe Funktionalität auf einer moderneren und stärker standardkonformen Infrastruktur bereitstellt.

#### F.11.1. dblink_connect

`dblink_connect` öffnet eine persistente Verbindung zu einer entfernten Datenbank.

##### Synopsis

```text
dblink_connect(text connstr) returns text
dblink_connect(text connname, text connstr) returns text
```

##### Beschreibung

`dblink_connect()` stellt eine Verbindung zu einer entfernten PostgreSQL-Datenbank her. Der zu kontaktierende Server und die Datenbank werden über eine Standard-Verbindungszeichenkette von `libpq` identifiziert. Optional kann der Verbindung ein Name zugewiesen werden. Mehrere benannte Verbindungen können gleichzeitig geöffnet sein, aber es ist jeweils nur eine unbenannte Verbindung erlaubt. Die Verbindung bleibt bestehen, bis sie geschlossen oder die Datenbanksitzung beendet wird.

Die Verbindungszeichenkette kann auch der Name eines vorhandenen Foreign Servers sein. Beim Definieren des Foreign Servers wird empfohlen, den Foreign Data Wrapper `dblink_fdw` zu verwenden. Siehe dazu das Beispiel unten sowie `CREATE SERVER` und `CREATE USER MAPPING`.

##### Argumente

- `connname`
  - Der für diese Verbindung zu verwendende Name. Wird er weggelassen, wird eine unbenannte Verbindung geöffnet und eine eventuell vorhandene unbenannte Verbindung ersetzt.

- `connstr`
  - Eine Verbindungsinformationszeichenkette im Stil von `libpq`, zum Beispiel `hostaddr=127.0.0.1 port=5432 dbname=mydb user=postgres password=mypasswd options=-csearch_path=`. Einzelheiten finden Sie in Abschnitt 32.1.1. Alternativ kann dies der Name eines Foreign Servers sein.

##### Rückgabewert

Gibt den Status zurück, der immer `OK` ist, da jeder Fehler dazu führt, dass die Funktion einen Fehler auslöst, statt zurückzukehren.

##### Hinweise

Wenn nicht vertrauenswürdige Benutzer Zugriff auf eine Datenbank haben, die kein sicheres Schema-Nutzungsmuster übernommen hat, sollte jede Sitzung damit beginnen, öffentlich beschreibbare Schemas aus dem `search_path` zu entfernen. Man könnte zum Beispiel `options=-csearch_path=` zu `connstr` hinzufügen. Diese Überlegung ist nicht `dblink`-spezifisch; sie gilt für jede Schnittstelle, die beliebige SQL-Befehle ausführt.

Der Foreign Data Wrapper `dblink_fdw` besitzt die zusätzliche boolesche Option `use_scram_passthrough`. Sie steuert, ob `dblink` die SCRAM-Pass-through-Authentifizierung verwendet, um sich mit der entfernten Datenbank zu verbinden. Bei SCRAM-Pass-through verwendet `dblink` SCRAM-gehashte Secrets statt Klartextpasswörtern, um sich mit dem entfernten Server zu verbinden. Dadurch werden Klartextpasswörter in PostgreSQL-Systemkatalogen vermieden. Weitere Einzelheiten und Einschränkungen finden Sie in der Dokumentation zur gleichnamigen Option von `postgres_fdw`.

Nur Superuser dürfen `dblink_connect` verwenden, um Verbindungen zu erstellen, die weder Passwortauthentifizierung noch SCRAM-Pass-through noch GSSAPI-Authentifizierung verwenden. Wenn Nicht-Superuser diese Fähigkeit benötigen, verwenden Sie stattdessen `dblink_connect_u`.

Es ist unklug, Verbindungsnamen zu wählen, die Gleichheitszeichen enthalten, da dies eine Verwechslungsgefahr mit Verbindungsinformationszeichenketten in anderen `dblink`-Funktionen eröffnet.

##### Beispiele

```sql
SELECT dblink_connect('dbname=postgres options=-csearch_path=');
SELECT dblink_connect('myconn', 'dbname=postgres options=-csearch_path=');
```

Beispiel mit Foreign-Data-Wrapper-Funktionalität:

```sql
CREATE SERVER fdtest FOREIGN DATA WRAPPER dblink_fdw
  OPTIONS (hostaddr '127.0.0.1', dbname 'contrib_regression');

CREATE USER regress_dblink_user WITH PASSWORD 'secret';
CREATE USER MAPPING FOR regress_dblink_user SERVER fdtest
  OPTIONS (user 'regress_dblink_user', password 'secret');
GRANT USAGE ON FOREIGN SERVER fdtest TO regress_dblink_user;
GRANT SELECT ON TABLE foo TO regress_dblink_user;

\set ORIGINAL_USER :USER
\c - regress_dblink_user
SELECT dblink_connect('myconn', 'fdtest');
SELECT * FROM dblink('myconn', 'SELECT * FROM foo') AS t(a int, b text, c text[]);

\c - :ORIGINAL_USER
REVOKE USAGE ON FOREIGN SERVER fdtest FROM regress_dblink_user;
REVOKE SELECT ON TABLE foo FROM regress_dblink_user;
DROP USER MAPPING FOR regress_dblink_user SERVER fdtest;
DROP USER regress_dblink_user;
DROP SERVER fdtest;
```

#### F.11.2. dblink_connect_u

`dblink_connect_u` öffnet unsicher eine persistente Verbindung zu einer entfernten Datenbank.

##### Synopsis

```text
dblink_connect_u(text connstr) returns text
dblink_connect_u(text connname, text connstr) returns text
```

##### Beschreibung

`dblink_connect_u()` ist identisch mit `dblink_connect()`, erlaubt Nicht-Superusern aber, sich mit jeder Authentifizierungsmethode zu verbinden.

Wenn der entfernte Server eine Authentifizierungsmethode auswählt, die kein Passwort umfasst, können Identitätsannahme und anschließende Rechteausweitung auftreten, da die Sitzung so erscheint, als stamme sie von dem Benutzer, unter dem der lokale PostgreSQL-Server läuft. Selbst wenn der entfernte Server ein Passwort verlangt, kann dieses aus der Serverumgebung geliefert werden, etwa aus einer Datei `~/.pgpass` des Serverbenutzers. Das eröffnet nicht nur ein Risiko der Identitätsannahme, sondern auch die Möglichkeit, ein Passwort einem nicht vertrauenswürdigen entfernten Server offenzulegen.

Deshalb wird `dblink_connect_u()` zunächst mit allen Privilegien für `PUBLIC` widerrufen installiert, sodass die Funktion außer für Superuser nicht aufrufbar ist. In manchen Situationen kann es angemessen sein, bestimmten als vertrauenswürdig geltenden Benutzern `EXECUTE` auf `dblink_connect_u()` zu gewähren; dies sollte aber mit Vorsicht geschehen. Außerdem wird empfohlen, dass eine Datei `~/.pgpass` des Serverbenutzers keine Einträge mit Wildcard-Hostnamen enthält.

Weitere Einzelheiten finden Sie bei `dblink_connect()`.

#### F.11.3. dblink_disconnect

`dblink_disconnect` schließt eine persistente Verbindung zu einer entfernten Datenbank.

##### Synopsis

```text
dblink_disconnect() returns text
dblink_disconnect(text connname) returns text
```

##### Beschreibung

`dblink_disconnect()` schließt eine zuvor mit `dblink_connect()` geöffnete Verbindung. Die Form ohne Argumente schließt eine unbenannte Verbindung.

##### Argumente

- `connname`
  - Der Name einer benannten Verbindung, die geschlossen werden soll.

##### Rückgabewert

Gibt den Status zurück, der immer `OK` ist, da jeder Fehler dazu führt, dass die Funktion einen Fehler auslöst, statt zurückzukehren.

##### Beispiele

```sql
SELECT dblink_disconnect();
SELECT dblink_disconnect('myconn');
```

#### F.11.4. dblink

`dblink` führt eine Abfrage in einer entfernten Datenbank aus.

##### Synopsis

```text
dblink(text connname, text sql [, bool fail_on_error]) returns setof record
dblink(text connstr, text sql [, bool fail_on_error]) returns setof record
dblink(text sql [, bool fail_on_error]) returns setof record
```

##### Beschreibung

`dblink` führt eine Abfrage in einer entfernten Datenbank aus. Meist ist dies eine `SELECT`-Abfrage, es kann aber jede SQL-Anweisung sein, die Zeilen zurückgibt.

Wenn zwei `text`-Argumente angegeben werden, wird das erste zunächst als Name einer persistenten Verbindung gesucht. Wenn eine solche Verbindung gefunden wird, wird der Befehl auf dieser Verbindung ausgeführt. Andernfalls wird das erste Argument wie bei `dblink_connect` als Verbindungsinformationszeichenkette behandelt, und die angegebene Verbindung wird nur für die Dauer dieses Befehls hergestellt.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung. Lassen Sie diesen Parameter weg, um die unbenannte Verbindung zu verwenden.

- `connstr`
  - Eine Verbindungsinformationszeichenkette, wie oben für `dblink_connect` beschrieben.

- `sql`
  - Die SQL-Abfrage, die in der entfernten Datenbank ausgeführt werden soll, zum Beispiel `select * from foo`.

- `fail_on_error`
  - Wenn `true`, der Standardwert bei Weglassen, führt ein Fehler auf der entfernten Seite auch lokal zu einem Fehler. Wenn `false`, wird der entfernte Fehler lokal als `NOTICE` gemeldet und die Funktion gibt keine Zeilen zurück.

##### Rückgabewert

Die Funktion gibt die von der Abfrage erzeugten Zeilen zurück. Da `dblink` mit jeder Abfrage verwendet werden kann, ist die Funktion als Rückgabe von `record` deklariert, statt eine bestimmte Spaltenmenge festzulegen. Deshalb müssen Sie die erwarteten Spalten in der aufrufenden Abfrage angeben:

```sql
SELECT *
FROM dblink('dbname=mydb options=-csearch_path=',
            'select proname, prosrc from pg_proc')
  AS t1(proname name, prosrc text)
WHERE proname LIKE 'bytea%';
```

Der Alias-Teil der `FROM`-Klausel muss die Namen und Typen der Spalten angeben, die die Funktion zurückgibt. Zur Laufzeit wird ein Fehler ausgelöst, wenn das tatsächliche Abfrageergebnis der entfernten Datenbank nicht dieselbe Spaltenzahl besitzt wie in der `FROM`-Klausel angegeben. Die Spaltennamen müssen jedoch nicht übereinstimmen, und `dblink` verlangt auch keine exakte Typgleichheit. Es genügt, wenn die zurückgegebenen Datenzeichenketten gültige Eingaben für die in der `FROM`-Klausel deklarierten Spaltentypen sind.

##### Hinweise

Eine bequeme Methode, `dblink` mit vorher festgelegten Abfragen zu verwenden, ist das Erstellen einer Sicht. Dadurch werden die Spaltentypinformationen in der Sicht verborgen und müssen nicht in jeder Abfrage erneut ausgeschrieben werden.

```sql
CREATE VIEW myremote_pg_proc AS
  SELECT *
  FROM dblink('dbname=postgres options=-csearch_path=',
              'select proname, prosrc from pg_proc')
    AS t1(proname name, prosrc text);

SELECT * FROM myremote_pg_proc WHERE proname LIKE 'bytea%';
```

##### Beispiele

```sql
SELECT * FROM dblink('dbname=postgres options=-csearch_path=',
                     'select proname, prosrc from pg_proc')
  AS t1(proname name, prosrc text)
WHERE proname LIKE 'bytea%';

SELECT dblink_connect('dbname=postgres options=-csearch_path=');

SELECT * FROM dblink('select proname, prosrc from pg_proc')
  AS t1(proname name, prosrc text)
WHERE proname LIKE 'bytea%';

SELECT dblink_connect('myconn', 'dbname=regression options=-csearch_path=');

SELECT * FROM dblink('myconn', 'select proname, prosrc from pg_proc')
  AS t1(proname name, prosrc text)
WHERE proname LIKE 'bytea%';
```

#### F.11.5. dblink_exec

`dblink_exec` führt einen Befehl in einer entfernten Datenbank aus.

##### Synopsis

```text
dblink_exec(text connname, text sql [, bool fail_on_error]) returns text
dblink_exec(text connstr, text sql [, bool fail_on_error]) returns text
dblink_exec(text sql [, bool fail_on_error]) returns text
```

##### Beschreibung

`dblink_exec` führt einen Befehl, also eine SQL-Anweisung ohne Zeilenrückgabe, in einer entfernten Datenbank aus. Wenn zwei `text`-Argumente angegeben werden, wird das erste wie bei `dblink` zuerst als Verbindungsname und andernfalls als Verbindungszeichenkette interpretiert.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

- `connstr`
  - Eine Verbindungsinformationszeichenkette, wie oben für `dblink_connect` beschrieben.

- `sql`
  - Der SQL-Befehl, der in der entfernten Datenbank ausgeführt werden soll, zum Beispiel `insert into foo values(0, 'a', '{"a0","b0","c0"}')`.

- `fail_on_error`
  - Wenn `true`, der Standardwert bei Weglassen, führt ein Fehler auf der entfernten Seite auch lokal zu einem Fehler. Wenn `false`, wird der entfernte Fehler lokal als `NOTICE` gemeldet, und der Rückgabewert der Funktion wird auf `ERROR` gesetzt.

##### Rückgabewert

Gibt den Status zurück, entweder die Statuszeichenkette des Befehls oder `ERROR`.

##### Beispiele

```sql
SELECT dblink_connect('dbname=dblink_test_standby');
SELECT dblink_exec('insert into foo values(21, ''z'', ''{"a0","b0","c0"}'');');
SELECT dblink_connect('myconn', 'dbname=regression');
SELECT dblink_exec('myconn', 'insert into foo values(21, ''z'', ''{"a0","b0","c0"}'');');
SELECT dblink_exec('myconn', 'insert into pg_class values (''foo'')', false);
```

Bei der letzten Abfrage kann der entfernte Fehler als `NOTICE` gemeldet werden und der Rückgabewert `ERROR` sein.

#### F.11.6. dblink_open

`dblink_open` öffnet einen Cursor in einer entfernten Datenbank.

##### Synopsis

```text
dblink_open(text cursorname, text sql [, bool fail_on_error]) returns text
dblink_open(text connname, text cursorname, text sql [, bool fail_on_error]) returns text
```

##### Beschreibung

`dblink_open()` öffnet einen Cursor in einer entfernten Datenbank. Der Cursor kann anschließend mit `dblink_fetch()` und `dblink_close()` verwendet werden.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

- `cursorname`
  - Der Name, der diesem Cursor zugewiesen werden soll.

- `sql`
  - Die `SELECT`-Anweisung, die in der entfernten Datenbank ausgeführt werden soll.

- `fail_on_error`
  - Wenn `true`, der Standardwert bei Weglassen, führt ein Fehler auf der entfernten Seite auch lokal zu einem Fehler. Wenn `false`, wird der entfernte Fehler lokal als `NOTICE` gemeldet und der Rückgabewert der Funktion auf `ERROR` gesetzt.

##### Rückgabewert

Gibt den Status zurück, entweder `OK` oder `ERROR`.

##### Hinweise

Da ein Cursor nur innerhalb einer Transaktion bestehen kann, startet `dblink_open` auf der entfernten Seite einen expliziten Transaktionsblock (`BEGIN`), falls dort noch keine Transaktion läuft. Diese Transaktion wird wieder geschlossen, wenn das passende `dblink_close` ausgeführt wird. Wenn Sie zwischen `dblink_open` und `dblink_close` mit `dblink_exec` Daten ändern und dann ein Fehler auftritt oder Sie `dblink_disconnect` vor `dblink_close` verwenden, geht Ihre Änderung verloren, weil die Transaktion abgebrochen wird.

##### Beispiele

```sql
SELECT dblink_connect('dbname=postgres options=-csearch_path=');
SELECT dblink_open('foo', 'select proname, prosrc from pg_proc');
```

#### F.11.7. dblink_fetch

`dblink_fetch` gibt Zeilen aus einem offenen Cursor in einer entfernten Datenbank zurück.

##### Synopsis

```text
dblink_fetch(text cursorname, int howmany [, bool fail_on_error]) returns setof record
dblink_fetch(text connname, text cursorname, int howmany [, bool fail_on_error]) returns setof record
```

##### Beschreibung

`dblink_fetch` holt Zeilen aus einem zuvor mit `dblink_open` eingerichteten Cursor.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

- `cursorname`
  - Der Name des Cursors, aus dem gelesen werden soll.

- `howmany`
  - Die maximale Anzahl abzurufender Zeilen. Ab der aktuellen Cursorposition werden die nächsten `howmany` Zeilen vorwärts geholt. Sobald der Cursor sein Ende erreicht hat, werden keine weiteren Zeilen erzeugt.

- `fail_on_error`
  - Wenn `true`, der Standardwert bei Weglassen, führt ein Fehler auf der entfernten Seite auch lokal zu einem Fehler. Wenn `false`, wird der entfernte Fehler lokal als `NOTICE` gemeldet und die Funktion gibt keine Zeilen zurück.

##### Rückgabewert

Die Funktion gibt die aus dem Cursor geholten Zeilen zurück. Um diese Funktion zu verwenden, müssen Sie die erwarteten Spalten angeben, wie zuvor für `dblink` beschrieben.

##### Hinweise

Wenn die Zahl der in der `FROM`-Klausel angegebenen Rückgabespalten nicht zur tatsächlichen Zahl der vom entfernten Cursor zurückgegebenen Spalten passt, wird ein Fehler ausgelöst. In diesem Fall wird der entfernte Cursor dennoch um so viele Zeilen weiterbewegt, wie ohne Fehler geholt worden wären. Dasselbe gilt für jeden anderen Fehler, der in der lokalen Abfrage nach dem entfernten `FETCH` auftritt.

##### Beispiele

```sql
SELECT dblink_connect('dbname=postgres options=-csearch_path=');
SELECT dblink_open('foo', 'select proname, prosrc from pg_proc where proname like ''bytea%''');
SELECT * FROM dblink_fetch('foo', 5) AS (funcname name, source text);
```

#### F.11.8. dblink_close

`dblink_close` schließt einen Cursor in einer entfernten Datenbank.

##### Synopsis

```text
dblink_close(text cursorname [, bool fail_on_error]) returns text
dblink_close(text connname, text cursorname [, bool fail_on_error]) returns text
```

##### Beschreibung

`dblink_close` schließt einen zuvor mit `dblink_open` geöffneten Cursor.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

- `cursorname`
  - Der Name des zu schließenden Cursors.

- `fail_on_error`
  - Wenn `true`, der Standardwert bei Weglassen, führt ein Fehler auf der entfernten Seite auch lokal zu einem Fehler. Wenn `false`, wird der entfernte Fehler lokal als `NOTICE` gemeldet und der Rückgabewert der Funktion auf `ERROR` gesetzt.

##### Rückgabewert

Gibt den Status zurück, entweder `OK` oder `ERROR`.

##### Hinweise

Wenn `dblink_open` einen expliziten Transaktionsblock gestartet hat und dies der letzte verbleibende offene Cursor in dieser Verbindung ist, führt `dblink_close` das passende `COMMIT` aus.

##### Beispiele

```sql
SELECT dblink_connect('dbname=postgres options=-csearch_path=');
SELECT dblink_open('foo', 'select proname, prosrc from pg_proc');
SELECT dblink_close('foo');
```

#### F.11.9. dblink_get_connections

`dblink_get_connections` gibt die Namen aller offenen benannten `dblink`-Verbindungen zurück.

##### Synopsis

```text
dblink_get_connections() returns text[]
```

##### Beschreibung und Rückgabewert

`dblink_get_connections` gibt ein Textarray mit den Namen aller offenen benannten `dblink`-Verbindungen zurück oder `NULL`, wenn keine vorhanden sind.

##### Beispiel

```sql
SELECT dblink_get_connections();
```

#### F.11.10. dblink_error_message

`dblink_error_message` ruft die letzte Fehlermeldung der benannten Verbindung ab.

##### Synopsis

```text
dblink_error_message(text connname) returns text
```

##### Beschreibung

`dblink_error_message` holt die jüngste entfernte Fehlermeldung für eine gegebene Verbindung.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

##### Rückgabewert

Gibt die letzte Fehlermeldung zurück oder `OK`, wenn in dieser Verbindung kein Fehler aufgetreten ist.

##### Hinweise

Wenn asynchrone Abfragen mit `dblink_send_query` gestartet werden, wird die mit der Verbindung verknüpfte Fehlermeldung möglicherweise erst aktualisiert, wenn die Antwortmeldung des Servers konsumiert wurde. Das bedeutet normalerweise, dass zuerst `dblink_is_busy` oder `dblink_get_result` aufgerufen werden sollte, damit ein durch die asynchrone Abfrage erzeugter Fehler sichtbar wird.

##### Beispiel

```sql
SELECT dblink_error_message('dtest1');
```

#### F.11.11. dblink_send_query

`dblink_send_query` sendet eine asynchrone Abfrage an eine entfernte Datenbank.

##### Synopsis

```text
dblink_send_query(text connname, text sql) returns int
```

##### Beschreibung

`dblink_send_query` sendet eine Abfrage zur asynchronen Ausführung, also ohne sofort auf das Ergebnis zu warten. Auf der Verbindung darf nicht bereits eine asynchrone Abfrage laufen.

Nach erfolgreichem Absenden einer asynchronen Abfrage kann der Abschlussstatus mit `dblink_is_busy` geprüft werden; die Ergebnisse werden schließlich mit `dblink_get_result` gesammelt. Es ist auch möglich, mit `dblink_cancel_query` zu versuchen, eine aktive asynchrone Abfrage abzubrechen.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

- `sql`
  - Die SQL-Anweisung, die in der entfernten Datenbank ausgeführt werden soll.

##### Rückgabewert

Gibt `1` zurück, wenn die Abfrage erfolgreich abgesendet wurde, andernfalls `0`.

##### Beispiel

```sql
SELECT dblink_send_query('dtest1', 'SELECT * FROM foo WHERE f1 < 3');
```

#### F.11.12. dblink_is_busy

`dblink_is_busy` prüft, ob eine Verbindung mit einer asynchronen Abfrage beschäftigt ist.

##### Synopsis

```text
dblink_is_busy(text connname) returns int
```

##### Beschreibung und Rückgabewert

`dblink_is_busy` prüft, ob eine asynchrone Abfrage läuft. Die Funktion gibt `1` zurück, wenn die Verbindung beschäftigt ist, und `0`, wenn sie nicht beschäftigt ist. Wenn diese Funktion `0` zurückgibt, ist garantiert, dass `dblink_get_result` nicht blockiert.

##### Argumente

- `connname`
  - Name der zu prüfenden Verbindung.

##### Beispiel

```sql
SELECT dblink_is_busy('dtest1');
```

#### F.11.13. dblink_get_notify

`dblink_get_notify` ruft asynchrone Benachrichtigungen auf einer Verbindung ab.

##### Synopsis

```text
dblink_get_notify() returns setof (notify_name text, be_pid int, extra text)
dblink_get_notify(text connname) returns setof (notify_name text, be_pid int, extra text)
```

##### Beschreibung

`dblink_get_notify` ruft Benachrichtigungen entweder auf der unbenannten Verbindung oder, wenn angegeben, auf einer benannten Verbindung ab. Um Benachrichtigungen über `dblink` zu empfangen, muss zuerst mit `dblink_exec` `LISTEN` ausgeführt werden. Einzelheiten finden Sie bei `LISTEN` und `NOTIFY`.

##### Argumente

- `connname`
  - Der Name einer benannten Verbindung, auf der Benachrichtigungen abgeholt werden sollen.

##### Rückgabewert

Gibt `setof (notify_name text, be_pid int, extra text)` zurück oder eine leere Menge, wenn keine Benachrichtigungen vorhanden sind.

##### Beispiel

```sql
SELECT dblink_exec('LISTEN virtual');
SELECT * FROM dblink_get_notify();
NOTIFY virtual;
SELECT * FROM dblink_get_notify();
```

#### F.11.14. dblink_get_result

`dblink_get_result` ruft das Ergebnis einer asynchronen Abfrage ab.

##### Synopsis

```text
dblink_get_result(text connname [, bool fail_on_error]) returns setof record
```

##### Beschreibung

`dblink_get_result` sammelt die Ergebnisse einer zuvor mit `dblink_send_query` gesendeten asynchronen Abfrage. Wenn die Abfrage noch nicht abgeschlossen ist, wartet `dblink_get_result`, bis sie abgeschlossen ist.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

- `fail_on_error`
  - Wenn `true`, der Standardwert bei Weglassen, führt ein Fehler auf der entfernten Seite auch lokal zu einem Fehler. Wenn `false`, wird der entfernte Fehler lokal als `NOTICE` gemeldet und die Funktion gibt keine Zeilen zurück.

##### Rückgabewert

Für eine asynchrone Abfrage, also eine SQL-Anweisung mit Zeilenrückgabe, gibt die Funktion die von der Abfrage erzeugten Zeilen zurück. Dafür müssen die erwarteten Spalten angegeben werden, wie zuvor für `dblink` beschrieben.

Für einen asynchronen Befehl, also eine SQL-Anweisung ohne Zeilenrückgabe, gibt die Funktion eine einzelne Zeile mit einer einzelnen Textspalte zurück, die die Statuszeichenkette des Befehls enthält. Auch dafür muss in der aufrufenden `FROM`-Klausel angegeben werden, dass das Ergebnis eine einzelne Textspalte besitzt.

##### Hinweise

Diese Funktion muss aufgerufen werden, wenn `dblink_send_query` `1` zurückgegeben hat. Sie muss einmal für jede gesendete Abfrage aufgerufen werden und danach ein weiteres Mal, um eine leere Ergebnismenge zu erhalten, bevor die Verbindung wieder verwendet werden kann.

Bei der Verwendung von `dblink_send_query` und `dblink_get_result` holt `dblink` das gesamte entfernte Abfrageergebnis, bevor irgendetwas davon an den lokalen Abfrageprozessor zurückgegeben wird. Wenn die Abfrage viele Zeilen zurückgibt, kann dies vorübergehend viel Speicher in der lokalen Sitzung belegen. Es kann besser sein, eine solche Abfrage mit `dblink_open` als Cursor zu öffnen und dann jeweils eine handhabbare Anzahl Zeilen zu holen. Alternativ kann das einfache `dblink()` verwendet werden, das Speicherausweitung vermeidet, indem es große Ergebnismengen auf Platte zwischenspeichert.

##### Beispiel

```sql
SELECT dblink_connect('dtest1', 'dbname=contrib_regression');
SELECT * FROM dblink_send_query('dtest1', 'select * from foo where f1 < 3') AS t1;
SELECT * FROM dblink_get_result('dtest1') AS t1(f1 int, f2 text, f3 text[]);
SELECT * FROM dblink_get_result('dtest1') AS t1(f1 int, f2 text, f3 text[]);
```

Für mehrere SQL-Anweisungen in einer asynchronen Abfrage muss `dblink_get_result` entsprechend mehrfach aufgerufen werden, bis eine leere Ergebnismenge zurückkommt.

#### F.11.15. dblink_cancel_query

`dblink_cancel_query` bricht eine aktive Abfrage auf der benannten Verbindung ab.

##### Synopsis

```text
dblink_cancel_query(text connname) returns text
```

##### Beschreibung

`dblink_cancel_query` versucht, eine auf der benannten Verbindung laufende Abfrage abzubrechen. Der Erfolg ist nicht garantiert, da die entfernte Abfrage zum Beispiel bereits beendet sein kann. Eine Abbruchanforderung erhöht nur die Wahrscheinlichkeit, dass die Abfrage bald fehlschlägt. Das normale Abfrageprotokoll muss dennoch abgeschlossen werden, zum Beispiel durch Aufruf von `dblink_get_result`.

##### Argumente

- `connname`
  - Name der zu verwendenden Verbindung.

##### Rückgabewert

Gibt `OK` zurück, wenn die Abbruchanforderung gesendet wurde, andernfalls den Text einer Fehlermeldung.

##### Beispiel

```sql
SELECT dblink_cancel_query('dtest1');
```

#### F.11.16. dblink_get_pkey

`dblink_get_pkey` gibt Positionen und Feldnamen der Primärschlüsselfelder einer Relation zurück.

##### Synopsis

```text
dblink_get_pkey(text relname) returns setof dblink_pkey_results
```

##### Beschreibung

`dblink_get_pkey` stellt Informationen über den Primärschlüssel einer Relation in der lokalen Datenbank bereit. Das ist manchmal nützlich, wenn Abfragen erzeugt werden, die an entfernte Datenbanken gesendet werden sollen.

##### Argumente

- `relname`
  - Name einer lokalen Relation, zum Beispiel `foo` oder `myschema.mytab`. Verwenden Sie doppelte Anführungszeichen, wenn der Name gemischte Groß-/Kleinschreibung oder Sonderzeichen enthält, zum Beispiel `"FooBar"`; ohne Anführungszeichen wird die Zeichenkette in Kleinbuchstaben gefaltet.

##### Rückgabewert

Gibt eine Zeile pro Primärschlüsselfeld zurück oder keine Zeilen, wenn die Relation keinen Primärschlüssel besitzt. Der Ergebnistyp ist definiert als:

```sql
CREATE TYPE dblink_pkey_results AS (position int, colname text);
```

Die Spalte `position` läuft einfach von 1 bis N. Sie ist die Nummer des Felds innerhalb des Primärschlüssels, nicht die Nummer innerhalb der Tabellenspalten.

##### Beispiel

```sql
CREATE TABLE foobar (
    f1 int,
    f2 int,
    f3 int,
    PRIMARY KEY (f1, f2, f3)
);

SELECT * FROM dblink_get_pkey('foobar');
```

#### F.11.17. dblink_build_sql_insert

`dblink_build_sql_insert` erzeugt eine `INSERT`-Anweisung aus einem lokalen Tupel und ersetzt dabei die Primärschlüsselwerte durch alternativ bereitgestellte Werte.

##### Synopsis

```text
dblink_build_sql_insert(text relname,
                        int2vector primary_key_attnums,
                        integer num_primary_key_atts,
                        text[] src_pk_att_vals_array,
                        text[] tgt_pk_att_vals_array) returns text
```

##### Beschreibung

`dblink_build_sql_insert` kann bei selektiver Replikation einer lokalen Tabelle in eine entfernte Datenbank nützlich sein. Die Funktion wählt eine Zeile aus der lokalen Tabelle anhand des Primärschlüssels aus und baut dann einen SQL-`INSERT`-Befehl, der diese Zeile dupliziert, jedoch die Primärschlüsselwerte durch die Werte im letzten Argument ersetzt. Um eine exakte Kopie der Zeile zu erzeugen, geben Sie für die letzten beiden Argumente dieselben Werte an.

##### Argumente

- `relname`
  - Name einer lokalen Relation, zum Beispiel `foo` oder `myschema.mytab`. Verwenden Sie doppelte Anführungszeichen für gemischte Groß-/Kleinschreibung oder Sonderzeichen.

- `primary_key_attnums`
  - Attributnummern der Primärschlüsselfelder, 1-basiert, zum Beispiel `1 2`.

- `num_primary_key_atts`
  - Die Anzahl der Primärschlüsselfelder.

- `src_pk_att_vals_array`
  - Werte der Primärschlüsselfelder, mit denen das lokale Tupel gesucht wird. Jedes Feld wird in Textform dargestellt. Es wird ein Fehler ausgelöst, wenn es keine lokale Zeile mit diesen Primärschlüsselwerten gibt.

- `tgt_pk_att_vals_array`
  - Werte der Primärschlüsselfelder, die in den erzeugten `INSERT`-Befehl eingesetzt werden. Jedes Feld wird in Textform dargestellt.

##### Rückgabewert

Gibt die angeforderte SQL-Anweisung als `text` zurück.

##### Hinweise

Seit PostgreSQL 9.0 werden die Attributnummern in `primary_key_attnums` als logische Spaltennummern interpretiert, entsprechend der Spaltenposition in `SELECT * FROM relname`. Frühere Versionen interpretierten die Nummern als physische Spaltenpositionen. Ein Unterschied besteht, wenn links von der angegebenen Spalte während der Lebensdauer der Tabelle Spalten gelöscht wurden.

##### Beispiel

```sql
SELECT dblink_build_sql_insert('foo', '1 2', 2, '{"1", "a"}', '{"1", "b''a"}');
```

```text
             dblink_build_sql_insert
--------------------------------------------------
 INSERT INTO foo(f1,f2,f3) VALUES('1','b''a','1')
(1 row)
```

#### F.11.18. dblink_build_sql_delete

`dblink_build_sql_delete` erzeugt eine `DELETE`-Anweisung unter Verwendung bereitgestellter Werte für Primärschlüsselfelder.

##### Synopsis

```text
dblink_build_sql_delete(text relname,
                        int2vector primary_key_attnums,
                        integer num_primary_key_atts,
                        text[] tgt_pk_att_vals_array) returns text
```

##### Beschreibung

`dblink_build_sql_delete` kann bei selektiver Replikation einer lokalen Tabelle in eine entfernte Datenbank nützlich sein. Die Funktion erzeugt einen SQL-`DELETE`-Befehl, der die Zeile mit den angegebenen Primärschlüsselwerten löscht.

##### Argumente

- `relname`
  - Name einer lokalen Relation, zum Beispiel `foo` oder `myschema.mytab`. Verwenden Sie doppelte Anführungszeichen für gemischte Groß-/Kleinschreibung oder Sonderzeichen.

- `primary_key_attnums`
  - Attributnummern der Primärschlüsselfelder, 1-basiert, zum Beispiel `1 2`.

- `num_primary_key_atts`
  - Die Anzahl der Primärschlüsselfelder.

- `tgt_pk_att_vals_array`
  - Werte der Primärschlüsselfelder, die im erzeugten `DELETE`-Befehl verwendet werden. Jedes Feld wird in Textform dargestellt.

##### Rückgabewert und Hinweise

Gibt die angeforderte SQL-Anweisung als `text` zurück. Für `primary_key_attnums` gilt dieselbe logische-Spaltennummer-Regel wie bei `dblink_build_sql_insert`.

##### Beispiel

```sql
SELECT dblink_build_sql_delete('"MyFoo"', '1 2', 2, '{"1", "b"}');
```

```text
              dblink_build_sql_delete
---------------------------------------------
 DELETE FROM "MyFoo" WHERE f1='1' AND f2='b'
(1 row)
```

#### F.11.19. dblink_build_sql_update

`dblink_build_sql_update` erzeugt eine `UPDATE`-Anweisung aus einem lokalen Tupel und ersetzt dabei die Primärschlüsselwerte durch alternativ bereitgestellte Werte.

##### Synopsis

```text
dblink_build_sql_update(text relname,
                        int2vector primary_key_attnums,
                        integer num_primary_key_atts,
                        text[] src_pk_att_vals_array,
                        text[] tgt_pk_att_vals_array) returns text
```

##### Beschreibung

`dblink_build_sql_update` kann bei selektiver Replikation einer lokalen Tabelle in eine entfernte Datenbank nützlich sein. Die Funktion wählt eine Zeile aus der lokalen Tabelle anhand des Primärschlüssels aus und baut dann einen SQL-`UPDATE`-Befehl, der diese Zeile dupliziert, jedoch die Primärschlüsselwerte durch die Werte im letzten Argument ersetzt. Um eine exakte Kopie der Zeile zu erzeugen, geben Sie für die letzten beiden Argumente dieselben Werte an. Der `UPDATE`-Befehl weist immer alle Felder der Zeile zu. Der Hauptunterschied zu `dblink_build_sql_insert` ist, dass angenommen wird, dass die Zielzeile in der entfernten Tabelle bereits existiert.

##### Argumente

- `relname`
  - Name einer lokalen Relation, zum Beispiel `foo` oder `myschema.mytab`. Verwenden Sie doppelte Anführungszeichen für gemischte Groß-/Kleinschreibung oder Sonderzeichen.

- `primary_key_attnums`
  - Attributnummern der Primärschlüsselfelder, 1-basiert, zum Beispiel `1 2`.

- `num_primary_key_atts`
  - Die Anzahl der Primärschlüsselfelder.

- `src_pk_att_vals_array`
  - Werte der Primärschlüsselfelder, mit denen das lokale Tupel gesucht wird. Jedes Feld wird in Textform dargestellt. Es wird ein Fehler ausgelöst, wenn es keine lokale Zeile mit diesen Primärschlüsselwerten gibt.

- `tgt_pk_att_vals_array`
  - Werte der Primärschlüsselfelder, die im erzeugten `UPDATE`-Befehl verwendet werden. Jedes Feld wird in Textform dargestellt.

##### Rückgabewert und Hinweise

Gibt die angeforderte SQL-Anweisung als `text` zurück. Für `primary_key_attnums` gilt dieselbe logische-Spaltennummer-Regel wie bei `dblink_build_sql_insert`.

##### Beispiel

```sql
SELECT dblink_build_sql_update('foo', '1 2', 2, '{"1", "a"}', '{"1", "b"}');
```

```text
                   dblink_build_sql_update
-------------------------------------------------------------
 UPDATE foo SET f1='1',f2='b',f3='1' WHERE f1='1' AND f2='b'
(1 row)
```


### F.12. dict_int — Beispielwörterbuch für Volltextsuche mit ganzen Zahlen

`dict_int` ist ein Beispiel für eine zusätzliche Wörterbuchvorlage für die Volltextsuche. Der Zweck dieses Beispielwörterbuchs besteht darin, die Indizierung ganzer Zahlen mit oder ohne Vorzeichen zu steuern. Dadurch können solche Zahlen indiziert werden, ohne dass die Anzahl unterschiedlicher Wörter übermäßig wächst, was die Suchleistung stark beeinflusst.

Dieses Modul gilt als „trusted“. Es kann also von Nicht-Superusern installiert werden, die das `CREATE`-Privileg für die aktuelle Datenbank besitzen.

#### F.12.1. Konfiguration

Das Wörterbuch akzeptiert drei Optionen:

- `maxlen`
  - Gibt die maximale Anzahl von Ziffern an, die in einem ganzzahligen Wort erlaubt sind. Der Standardwert ist `6`.

- `rejectlong`
  - Legt fest, ob eine zu lange ganze Zahl abgeschnitten oder ignoriert werden soll. Wenn `rejectlong` `false` ist, was dem Standard entspricht, gibt das Wörterbuch die ersten `maxlen` Ziffern der ganzen Zahl zurück. Wenn `rejectlong` `true` ist, behandelt das Wörterbuch eine zu lange ganze Zahl als Stoppwort, sodass sie nicht indiziert wird. Das bedeutet auch, dass nach einer solchen ganzen Zahl nicht gesucht werden kann.

- `absval`
  - Legt fest, ob führende Zeichen `+` oder `-` aus ganzzahligen Wörtern entfernt werden sollen. Der Standardwert ist `false`. Bei `true` wird das Vorzeichen entfernt, bevor `maxlen` angewendet wird.

#### F.12.2. Verwendung

Die Installation der Erweiterung `dict_int` erzeugt eine Textsuchvorlage `intdict_template` und ein darauf basierendes Wörterbuch `intdict` mit den Standardparametern. Sie können die Parameter ändern, zum Beispiel:

```sql
ALTER TEXT SEARCH DICTIONARY intdict (MAXLEN = 4, REJECTLONG = true);
```

```text
ALTER TEXT SEARCH DICTIONARY
```

Oder Sie können neue Wörterbücher auf Basis der Vorlage erstellen.

Zum Testen des Wörterbuchs können Sie Folgendes versuchen:

```sql
SELECT ts_lexize('intdict', '12345678');
```

```text
 ts_lexize
-----------
 {123456}
```

In der Praxis wird das Wörterbuch in eine Textsuchkonfiguration eingebunden, wie in [Kapitel 12](12_Volltextsuche.md) beschrieben. Das könnte zum Beispiel so aussehen:

```sql
ALTER TEXT SEARCH CONFIGURATION english
    ALTER MAPPING FOR int, uint WITH intdict;
```


### F.13. dict_xsyn — Beispiel-Synonymwörterbuch für die Volltextsuche

`dict_xsyn` (Extended Synonym Dictionary) ist ein Beispiel für eine zusätzliche Wörterbuchvorlage für die Volltextsuche. Dieser Wörterbuchtyp ersetzt Wörter durch Gruppen ihrer Synonyme und ermöglicht dadurch, nach einem Wort über beliebige seiner Synonyme zu suchen.

#### F.13.1. Konfiguration

Ein `dict_xsyn`-Wörterbuch akzeptiert die folgenden Optionen:

- `matchorig`
  - Steuert, ob das ursprüngliche Wort vom Wörterbuch akzeptiert wird. Der Standardwert ist `true`.

- `matchsynonyms`
  - Steuert, ob Synonyme vom Wörterbuch akzeptiert werden. Der Standardwert ist `false`.

- `keeporig`
  - Steuert, ob das ursprüngliche Wort in der Ausgabe des Wörterbuchs enthalten ist. Der Standardwert ist `true`.

- `keepsynonyms`
  - Steuert, ob die Synonyme in der Ausgabe des Wörterbuchs enthalten sind. Der Standardwert ist `true`.

- `rules`
  - Gibt den Basisnamen der Datei an, die die Synonymliste enthält. Diese Datei muss in `$SHAREDIR/tsearch_data/` liegen, wobei `$SHAREDIR` das Verzeichnis für gemeinsam genutzte Daten der PostgreSQL-Installation bezeichnet. Der Dateiname muss auf `.rules` enden; diese Endung wird im Parameter `rules` nicht mit angegeben.

Die Regeldatei hat das folgende Format:

- Jede Zeile stellt eine Gruppe von Synonymen für ein einzelnes Wort dar. Dieses Wort steht als erstes in der Zeile. Synonyme werden durch Leerraum getrennt:

  ```text
  word syn1 syn2 syn3
  ```

- Das Zeichen `#` leitet einen Kommentar ein. Es kann an jeder Position einer Zeile stehen; der Rest der Zeile wird ignoriert.

Ein Beispiel finden Sie in `xsyn_sample.rules`, das in `$SHAREDIR/tsearch_data/` installiert wird.

#### F.13.2. Verwendung

Die Installation der Erweiterung `dict_xsyn` erzeugt eine Textsuchvorlage `xsyn_template` und ein darauf basierendes Wörterbuch `xsyn` mit Standardparametern. Sie können die Parameter ändern, zum Beispiel:

```sql
ALTER TEXT SEARCH DICTIONARY xsyn (RULES='my_rules', KEEPORIG=false);
```

```text
ALTER TEXT SEARCH DICTIONARY
```

Oder Sie können neue Wörterbücher auf Basis der Vorlage erstellen.

Zum Testen des Wörterbuchs können Sie Folgendes versuchen:

```sql
SELECT ts_lexize('xsyn', 'word');
```

```text
      ts_lexize
-----------------------
 {syn1,syn2,syn3}
```

```sql
ALTER TEXT SEARCH DICTIONARY xsyn (RULES='my_rules', KEEPORIG=true);
SELECT ts_lexize('xsyn', 'word');
```

```text
      ts_lexize
-----------------------
 {word,syn1,syn2,syn3}
```

```sql
ALTER TEXT SEARCH DICTIONARY xsyn (RULES='my_rules', KEEPORIG=false, MATCHSYNONYMS=true);
SELECT ts_lexize('xsyn', 'syn1');
```

```text
      ts_lexize
-----------------------
 {syn1,syn2,syn3}
```

```sql
ALTER TEXT SEARCH DICTIONARY xsyn (RULES='my_rules', KEEPORIG=true, MATCHORIG=false, KEEPSYNONYMS=false);
SELECT ts_lexize('xsyn', 'syn1');
```

```text
      ts_lexize
-----------------------
 {word}
```

In der Praxis wird das Wörterbuch in eine Textsuchkonfiguration eingebunden, wie in [Kapitel 12](12_Volltextsuche.md) beschrieben. Das könnte zum Beispiel so aussehen:

```sql
ALTER TEXT SEARCH CONFIGURATION english
    ALTER MAPPING FOR word, asciiword WITH xsyn, english_stem;
```

### F.14. earthdistance — Großkreisentfernungen berechnen

Das Modul `earthdistance` stellt zwei verschiedene Ansätze bereit, um Großkreisentfernungen auf der Erdoberfläche zu berechnen. Der zuerst beschriebene Ansatz hängt vom Modul `cube` ab. Der zweite Ansatz basiert auf dem eingebauten Datentyp `point` und verwendet Längen- und Breitengrad als Koordinaten.

In diesem Modul wird angenommen, dass die Erde eine perfekte Kugel ist. Wenn das zu ungenau ist, ist das Projekt PostGIS unter <https://postgis.net/> möglicherweise geeigneter.

Das Modul `cube` muss installiert sein, bevor `earthdistance` installiert werden kann. Alternativ können Sie mit der Option `CASCADE` von `CREATE EXTENSION` beide Module in einem Befehl installieren.

> **Achtung:** Es wird dringend empfohlen, `earthdistance` und `cube` im selben Schema zu installieren und für dieses Schema keinem nicht vertrauenswürdigen Benutzer das `CREATE`-Privileg zu gewähren. Andernfalls können bei der Installation Sicherheitsrisiken entstehen, wenn das Schema von `earthdistance` Objekte enthält, die von einem böswilligen Benutzer definiert wurden. Außerdem sollte der gesamte `search_path` bei Verwendung der Funktionen von `earthdistance` nur vertrauenswürdige Schemas enthalten.

#### F.14.1. Cube-basierte Erd-Entfernungen

Die Daten werden als Cubes gespeichert, die Punkte darstellen: Beide Ecken sind gleich, und drei Koordinaten beschreiben den Abstand in `x`-, `y`- und `z`-Richtung vom Erdmittelpunkt. Dazu wird eine Domain `earth` über dem Typ `cube` bereitgestellt. Sie enthält Constraint-Prüfungen, die sicherstellen, dass der Wert diese Einschränkungen erfüllt und hinreichend nahe an der tatsächlichen Erdoberfläche liegt.

Der Erdradius wird aus der Funktion `earth()` bezogen und in Metern angegeben. Durch Ändern dieser einen Funktion können Sie das Modul jedoch auf andere Einheiten umstellen oder einen anderen Radiuswert verwenden.

Dieses Paket kann auch für astronomische Datenbanken verwendet werden. Astronomen werden `earth()` vermutlich so ändern wollen, dass die Funktion einen Radius von `180/pi()` zurückgibt, sodass Entfernungen in Grad angegeben werden.

Das Modul stellt Funktionen bereit, um Eingaben als Breiten- und Längengrad in Grad zu unterstützen, Breiten- und Längengrad wieder auszugeben, Großkreisentfernungen zwischen zwei Punkten zu berechnen und einfach eine Bounding Box für Indexsuchen anzugeben.

Die bereitgestellten Funktionen sind in Tabelle F.4 aufgeführt.

**Tabelle F.4. Cube-basierte Funktionen von `earthdistance`**

| Funktion | Beschreibung |
| --- | --- |
| `earth()` → `float8` | Gibt den angenommenen Erdradius zurück. |
| `sec_to_gc(float8)` → `float8` | Wandelt die normale geradlinige Entfernung (Sekante) zwischen zwei Punkten auf der Erdoberfläche in die Großkreisentfernung zwischen ihnen um. |
| `gc_to_sec(float8)` → `float8` | Wandelt die Großkreisentfernung zwischen zwei Punkten auf der Erdoberfläche in die normale geradlinige Entfernung (Sekante) zwischen ihnen um. |
| `ll_to_earth(float8, float8)` → `earth` | Gibt die Position eines Punkts auf der Erdoberfläche anhand von Breitengrad (Argument 1) und Längengrad (Argument 2) in Grad zurück. |
| `latitude(earth)` → `float8` | Gibt den Breitengrad eines Punkts auf der Erdoberfläche in Grad zurück. |
| `longitude(earth)` → `float8` | Gibt den Längengrad eines Punkts auf der Erdoberfläche in Grad zurück. |
| `earth_distance(earth, earth)` → `float8` | Gibt die Großkreisentfernung zwischen zwei Punkten auf der Erdoberfläche zurück. |
| `earth_box(earth, float8)` → `cube` | Gibt eine Box zurück, die für eine indizierte Suche mit dem Operator `@>` von `cube` nach Punkten innerhalb einer angegebenen Großkreisentfernung von einem Ort geeignet ist. Einige Punkte in dieser Box liegen weiter entfernt als die angegebene Großkreisentfernung; daher sollte die Abfrage zusätzlich mit `earth_distance` prüfen. |

#### F.14.2. Point-basierte Erd-Entfernungen

Der zweite Teil des Moduls stellt Orte auf der Erde als Werte des Typs `point` dar. Die erste Komponente steht dabei für den Längengrad in Grad, die zweite für den Breitengrad in Grad. Punkte werden also als `(longitude, latitude)` interpretiert und nicht umgekehrt, weil der Längengrad der intuitiven Vorstellung der `x`-Achse näherkommt und der Breitengrad der `y`-Achse.

Es wird ein einzelner Operator bereitgestellt, siehe Tabelle F.5.

**Tabelle F.5. Point-basierte Operatoren von `earthdistance`**

| Operator | Beschreibung |
| --- | --- |
| `point <@> point` → `float8` | Berechnet die Entfernung in englischen Meilen zwischen zwei Punkten auf der Erdoberfläche. |

Beachten Sie, dass die Einheiten hier im Gegensatz zum cube-basierten Teil des Moduls fest verdrahtet sind: Eine Änderung der Funktion `earth()` wirkt sich nicht auf die Ergebnisse dieses Operators aus.

Ein Nachteil der Längen-/Breitengrad-Darstellung besteht darin, dass Sie auf Randbedingungen in der Nähe der Pole und bei etwa `+/- 180` Grad Länge achten müssen. Die cube-basierte Darstellung vermeidet diese Diskontinuitäten.

### F.15. file_fdw — Zugriff auf Datendateien im Dateisystem des Servers

Das Modul `file_fdw` stellt den Foreign Data Wrapper `file_fdw` bereit. Er kann verwendet werden, um auf Datendateien im Dateisystem des Servers zuzugreifen oder Programme auf dem Server auszuführen und deren Ausgabe zu lesen. Die Datendatei oder Programmausgabe muss in einem Format vorliegen, das von `COPY FROM` gelesen werden kann; Einzelheiten finden Sie bei `COPY`. Der Zugriff auf Datendateien ist derzeit nur lesend möglich.

Eine mit diesem Wrapper erstellte Foreign Table kann die folgenden Optionen besitzen:

- `filename`
  - Gibt die zu lesende Datei an. Relative Pfade beziehen sich auf das Datenverzeichnis. Entweder `filename` oder `program` muss angegeben werden, aber nicht beide.

- `program`
  - Gibt den auszuführenden Befehl an. Die Standardausgabe dieses Befehls wird gelesen, als ob `COPY FROM PROGRAM` verwendet würde. Entweder `program` oder `filename` muss angegeben werden, aber nicht beide.

- `format`
  - Gibt das Datenformat an, entsprechend der Option `FORMAT` von `COPY`.

- `header`
  - Gibt an, ob die Daten eine Kopfzeile besitzen, entsprechend der Option `HEADER` von `COPY`.

- `delimiter`
  - Gibt das Trennzeichen der Daten an, entsprechend der Option `DELIMITER` von `COPY`.

- `quote`
  - Gibt das Quote-Zeichen der Daten an, entsprechend der Option `QUOTE` von `COPY`.

- `escape`
  - Gibt das Escape-Zeichen der Daten an, entsprechend der Option `ESCAPE` von `COPY`.

- `null`
  - Gibt die Zeichenkette für NULL-Werte an, entsprechend der Option `NULL` von `COPY`.

- `encoding`
  - Gibt die Datenkodierung an, entsprechend der Option `ENCODING` von `COPY`.

- `on_error`
  - Gibt an, wie sich der Import verhalten soll, wenn beim Umwandeln eines Eingabewerts in den Datentyp einer Spalte ein Fehler auftritt, entsprechend der Option `ON_ERROR` von `COPY`.

- `reject_limit`
  - Gibt die maximale Anzahl tolerierter Fehler beim Umwandeln eines Eingabewerts in den Datentyp einer Spalte an, entsprechend der Option `REJECT_LIMIT` von `COPY`.

- `log_verbosity`
  - Gibt den Umfang der von `file_fdw` ausgegebenen Meldungen an, entsprechend der Option `LOG_VERBOSITY` von `COPY`.

Beachten Sie, dass `COPY` Optionen wie `HEADER` ohne zugehörigen Wert erlaubt, die Optionssyntax für Foreign Tables aber in allen Fällen einen Wert erfordert. Um `COPY`-Optionen zu aktivieren, die normalerweise ohne Wert geschrieben werden, können Sie den Wert `TRUE` übergeben, da alle diese Optionen boolesch sind.

Eine Spalte einer mit diesem Wrapper erstellten Foreign Table kann die folgenden Optionen besitzen:

- `force_not_null`
  - Boolesche Option. Wenn sie `true` ist, werden Werte der Spalte nicht mit der NULL-Zeichenkette abgeglichen, also mit der Tabellenoption `null`. Dies hat denselben Effekt, als würde die Spalte in der Option `FORCE_NOT_NULL` von `COPY` aufgeführt.

- `force_null`
  - Boolesche Option. Wenn sie `true` ist, werden Werte der Spalte, die der NULL-Zeichenkette entsprechen, auch dann als `NULL` zurückgegeben, wenn der Wert gequotet ist. Ohne diese Option werden nur nicht gequotete Werte, die der NULL-Zeichenkette entsprechen, als `NULL` zurückgegeben. Dies hat denselben Effekt, als würde die Spalte in der Option `FORCE_NULL` von `COPY` aufgeführt.

Die Option `FORCE_QUOTE` von `COPY` wird von `file_fdw` derzeit nicht unterstützt.

Diese Optionen können nur für eine Foreign Table oder ihre Spalten angegeben werden, nicht in den Optionen des Foreign Data Wrappers `file_fdw` selbst und auch nicht in den Optionen eines Servers oder User Mappings, die diesen Wrapper verwenden.

Das Ändern von Optionen auf Tabellenebene erfordert aus Sicherheitsgründen Superuser-Rechte oder die Privilegien der Rolle `pg_read_server_files` zur Verwendung von `filename` beziehungsweise der Rolle `pg_execute_server_program` zur Verwendung von `program`. Nur bestimmte Benutzer sollten steuern können, welche Datei gelesen oder welches Programm ausgeführt wird. Grundsätzlich könnten normale Benutzer berechtigt werden, die anderen Optionen zu ändern; derzeit wird das jedoch nicht unterstützt.

Wenn Sie die Option `program` angeben, denken Sie daran, dass die Optionszeichenkette von der Shell ausgeführt wird. Falls Argumente an den Befehl übergeben werden müssen, die aus einer nicht vertrauenswürdigen Quelle stammen, müssen Sie sorgfältig alle Zeichen entfernen oder maskieren, die für die Shell eine besondere Bedeutung haben könnten. Aus Sicherheitsgründen ist es am besten, eine feste Befehlszeichenkette zu verwenden oder zumindest keine Benutzereingaben darin weiterzugeben.

Für eine Foreign Table, die `file_fdw` verwendet, zeigt `EXPLAIN` den Namen der zu lesenden Datei oder des auszuführenden Programms an. Bei einer Datei wird außerdem die Dateigröße in Byte angezeigt, sofern nicht `COSTS OFF` angegeben ist.

**Beispiel F.1. Eine Foreign Table für PostgreSQL-CSV-Logs erstellen**

Eine naheliegende Verwendung von `file_fdw` besteht darin, das PostgreSQL-Aktivitätslog als abfragbare Tabelle verfügbar zu machen. Dazu müssen Sie zunächst in eine CSV-Datei protokollieren, die hier `pglog.csv` heißt. Installieren Sie zuerst `file_fdw` als Erweiterung:

```sql
CREATE EXTENSION file_fdw;
```

Erstellen Sie dann einen Foreign Server:

```sql
CREATE SERVER pglog FOREIGN DATA WRAPPER file_fdw;
```

Nun können Sie die Foreign Table erstellen. Mit `CREATE FOREIGN TABLE` definieren Sie die Spalten der Tabelle, den Namen der CSV-Datei und ihr Format:

```sql
CREATE FOREIGN TABLE pglog (
  log_time timestamp(3) with time zone,
  user_name text,
  database_name text,
  process_id integer,
  connection_from text,
  session_id text,
  session_line_num bigint,
  command_tag text,
  session_start_time timestamp with time zone,
  virtual_transaction_id text,
  transaction_id bigint,
  error_severity text,
  sql_state_code text,
  message text,
  detail text,
  hint text,
  internal_query text,
  internal_query_pos integer,
  context text,
  query text,
  query_pos integer,
  location text,
  application_name text,
  backend_type text,
  leader_pid integer,
  query_id bigint
) SERVER pglog
OPTIONS ( filename 'log/pglog.csv', format 'csv' );
```

Damit können Sie das Log direkt abfragen. In Produktionsumgebungen müssten Sie natürlich eine geeignete Behandlung der Logrotation definieren.

**Beispiel F.2. Eine Foreign Table mit einer Option an einer Spalte erstellen**

Um die Option `force_null` für eine Spalte zu setzen, verwenden Sie das Schlüsselwort `OPTIONS`.

```sql
CREATE FOREIGN TABLE films (
 code char(5) NOT NULL,
 title text NOT NULL,
 rating text OPTIONS (force_null 'true')
) SERVER film_server
OPTIONS ( filename 'films/db.csv', format 'csv' );
```

### F.16. fuzzystrmatch — Ähnlichkeiten und Abstände von Zeichenketten bestimmen

Das Modul `fuzzystrmatch` stellt mehrere Funktionen bereit, um Ähnlichkeiten und Abstände zwischen Zeichenketten zu bestimmen.

> **Achtung:** Derzeit funktionieren die Funktionen `soundex`, `metaphone`, `dmetaphone` und `dmetaphone_alt` nicht gut mit Multibyte-Kodierungen wie UTF-8. Verwenden Sie für solche Daten `daitch_mokotoff` oder `levenshtein`.

Dieses Modul gilt als „trusted“. Es kann also von Nicht-Superusern installiert werden, die das `CREATE`-Privileg für die aktuelle Datenbank besitzen.

#### F.16.1. Soundex

Das Soundex-System ist eine Methode zum Abgleich ähnlich klingender Namen, indem diese in denselben Code umgewandelt werden. Es wurde ursprünglich bei den Volkszählungen der Vereinigten Staaten in den Jahren 1880, 1900 und 1910 verwendet. Beachten Sie, dass Soundex für nicht englische Namen wenig nützlich ist.

Das Modul `fuzzystrmatch` stellt zwei Funktionen für die Arbeit mit Soundex-Codes bereit:

```text
soundex(text) returns text
difference(text, text) returns int
```

Die Funktion `soundex` wandelt eine Zeichenkette in ihren Soundex-Code um. Die Funktion `difference` wandelt zwei Zeichenketten in ihre Soundex-Codes um und gibt dann die Anzahl übereinstimmender Codepositionen aus. Da Soundex-Codes vier Zeichen lang sind, liegt das Ergebnis zwischen null und vier; null bedeutet keine Übereinstimmung, vier eine exakte Übereinstimmung. Der Funktionsname ist daher etwas unglücklich gewählt; `similarity` wäre passender gewesen.

Einige Verwendungsbeispiele:

```sql
SELECT soundex('hello world!');

SELECT soundex('Anne'), soundex('Ann'), difference('Anne', 'Ann');
SELECT soundex('Anne'), soundex('Andrew'), difference('Anne', 'Andrew');
SELECT soundex('Anne'), soundex('Margaret'), difference('Anne', 'Margaret');

CREATE TABLE s (nm text);

INSERT INTO s VALUES ('john');
INSERT INTO s VALUES ('joan');
INSERT INTO s VALUES ('wobbly');
INSERT INTO s VALUES ('jack');

SELECT * FROM s WHERE soundex(nm) = soundex('john');

SELECT * FROM s WHERE difference(s.nm, 'john') > 2;
```

#### F.16.2. Daitch-Mokotoff-Soundex

Wie das ursprüngliche Soundex-System gleicht Daitch-Mokotoff-Soundex ähnlich klingende Namen ab, indem es sie in denselben Code umwandelt. Daitch-Mokotoff-Soundex ist jedoch für nicht englische Namen deutlich nützlicher als das ursprüngliche System. Wichtige Verbesserungen gegenüber dem Original sind:

- Der Code basiert auf den ersten sechs bedeutungstragenden Buchstaben statt auf vier.
- Ein Buchstabe oder eine Buchstabenkombination wird auf zehn mögliche Codes statt auf sieben abgebildet.
- Wenn zwei aufeinanderfolgende Buchstaben einen einzelnen Laut bilden, werden sie als eine einzelne Zahl codiert.
- Wenn ein Buchstabe oder eine Buchstabenkombination unterschiedlich ausgesprochen werden kann, werden mehrere Codes erzeugt, um alle Möglichkeiten abzudecken.

Diese Funktion erzeugt die Daitch-Mokotoff-Soundex-Codes für ihre Eingabe:

```text
daitch_mokotoff(source text) returns text[]
```

Das Ergebnis kann je nach Anzahl plausibler Aussprachen einen oder mehrere Codes enthalten und wird deshalb als Array dargestellt.

Da ein Daitch-Mokotoff-Soundex-Code nur aus sechs Ziffern besteht, sollte `source` vorzugsweise ein einzelnes Wort oder ein einzelner Name sein.

Beispiele:

```sql
SELECT daitch_mokotoff('George');
```

```text
 daitch_mokotoff
-----------------
 {595000}
```

```sql
SELECT daitch_mokotoff('John');
```

```text
 daitch_mokotoff
-----------------
 {160000,460000}
```

```sql
SELECT daitch_mokotoff('Bierschbach');
```

```text
                       daitch_mokotoff
-----------------------------------------------------------
 {794575,794574,794750,794740,745750,745740,747500,747400}
```

```sql
SELECT daitch_mokotoff('Schwartzenegger');
```

```text
 daitch_mokotoff
-----------------
 {479465}
```

Beim Abgleich einzelner Namen können die zurückgegebenen `text`-Arrays direkt mit dem Operator `&&` verglichen werden: Jede Überlappung kann als Treffer betrachtet werden. Für Effizienz kann ein GIN-Index verwendet werden; siehe [Abschnitt 65.4](65_Eingebaute_Index_Access_Methods.md#654-ginindizes) und das folgende Beispiel:

```sql
CREATE TABLE s (nm text);
CREATE INDEX ix_s_dm ON s USING gin (daitch_mokotoff(nm)) WITH (fastupdate = off);

INSERT INTO s (nm) VALUES
  ('Schwartzenegger'),
  ('John'),
  ('James'),
  ('Steinman'),
  ('Steinmetz');

SELECT * FROM s WHERE daitch_mokotoff(nm) && daitch_mokotoff('Swartzenegger');
SELECT * FROM s WHERE daitch_mokotoff(nm) && daitch_mokotoff('Jane');
SELECT * FROM s WHERE daitch_mokotoff(nm) && daitch_mokotoff('Jens');
```

Für die Indizierung und den Abgleich beliebig vieler Namen in beliebiger Reihenfolge können Funktionen der Volltextsuche verwendet werden. Siehe [Kapitel 12](12_Volltextsuche.md) und dieses Beispiel:

```sql
CREATE FUNCTION soundex_tsvector(v_name text) RETURNS tsvector
BEGIN ATOMIC
  SELECT to_tsvector('simple',
                     string_agg(array_to_string(daitch_mokotoff(n), ' '), ' '))
  FROM regexp_split_to_table(v_name, '\s+') AS n;
END;

CREATE FUNCTION soundex_tsquery(v_name text) RETURNS tsquery
BEGIN ATOMIC
  SELECT string_agg('(' || array_to_string(daitch_mokotoff(n), '|') || ')', '&')::tsquery
  FROM regexp_split_to_table(v_name, '\s+') AS n;
END;

CREATE TABLE s (nm text);
CREATE INDEX ix_s_txt ON s USING gin (soundex_tsvector(nm)) WITH (fastupdate = off);

INSERT INTO s (nm) VALUES
  ('John Doe'),
  ('Jane Roe'),
  ('Public John Q.'),
  ('George Best'),
  ('John Yamson');

SELECT * FROM s WHERE soundex_tsvector(nm) @@ soundex_tsquery('john');
SELECT * FROM s WHERE soundex_tsvector(nm) @@ soundex_tsquery('jane doe');
SELECT * FROM s WHERE soundex_tsvector(nm) @@ soundex_tsquery('john public');
SELECT * FROM s WHERE soundex_tsvector(nm) @@ soundex_tsquery('besst, giorgio');
SELECT * FROM s WHERE soundex_tsvector(nm) @@ soundex_tsquery('Jameson John');
```

Wenn die Neuberechnung der Soundex-Codes bei Index-Rechecks vermieden werden soll, kann statt eines Index auf einem Ausdruck ein Index auf einer separaten Spalte verwendet werden. Dafür kann eine gespeicherte generierte Spalte genutzt werden; siehe [Abschnitt 5.4](05_Datendefinition.md#54-generierte-spalten).

#### F.16.3. Levenshtein

Diese Funktion berechnet die Levenshtein-Distanz zwischen zwei Zeichenketten:

```text
levenshtein(source text, target text, ins_cost int, del_cost int,
            sub_cost int) returns int
levenshtein(source text, target text) returns int
levenshtein_less_equal(source text, target text, ins_cost int,
                       del_cost int, sub_cost int, max_d int) returns int
levenshtein_less_equal(source text, target text, max_d int) returns int
```

`source` und `target` können beliebige nicht-NULL-Zeichenketten mit höchstens 255 Zeichen sein. Die Kostenparameter geben an, wie viel für das Einfügen, Löschen beziehungsweise Ersetzen eines Zeichens berechnet wird. Sie können die Kostenparameter wie in der zweiten Funktionsvariante weglassen; dann ist ihr Standardwert jeweils `1`.

`levenshtein_less_equal` ist eine beschleunigte Variante der Levenshtein-Funktion für Fälle, in denen nur kleine Distanzen interessieren. Wenn die tatsächliche Distanz kleiner oder gleich `max_d` ist, gibt `levenshtein_less_equal` die korrekte Distanz zurück; andernfalls gibt die Funktion einen Wert größer als `max_d` zurück. Wenn `max_d` negativ ist, entspricht das Verhalten dem von `levenshtein`.

Beispiele:

```sql
SELECT levenshtein('GUMBO', 'GAMBOL');
```

```text
 levenshtein
-------------
           2
(1 row)
```

```sql
SELECT levenshtein('GUMBO', 'GAMBOL', 2, 1, 1);
```

```text
 levenshtein
-------------
           3
(1 row)
```

```sql
SELECT levenshtein_less_equal('extensive', 'exhaustive', 2);
```

```text
 levenshtein_less_equal
------------------------
                      3
(1 row)
```

```sql
SELECT levenshtein_less_equal('extensive', 'exhaustive', 4);
```

```text
 levenshtein_less_equal
------------------------
                      4
(1 row)
```

#### F.16.4. Metaphone

Metaphone basiert wie Soundex auf der Idee, für eine Eingabezeichenkette einen repräsentativen Code zu erzeugen. Zwei Zeichenketten gelten dann als ähnlich, wenn sie denselben Code haben.

Diese Funktion berechnet den Metaphone-Code einer Eingabezeichenkette:

```text
metaphone(source text, max_output_length int) returns text
```

`source` muss eine nicht-NULL-Zeichenkette mit höchstens 255 Zeichen sein. `max_output_length` legt die maximale Länge des ausgegebenen Metaphone-Codes fest; längere Ausgaben werden auf diese Länge gekürzt.

Beispiel:

```sql
SELECT metaphone('GUMBO', 4);
```

```text
 metaphone
-----------
 KM
(1 row)
```

#### F.16.5. Double Metaphone

Das Double-Metaphone-System berechnet für eine gegebene Eingabezeichenkette zwei „klingt wie“-Zeichenketten: eine primäre und eine alternative. In den meisten Fällen sind sie gleich; besonders bei nicht englischen Namen können sie sich je nach Aussprache jedoch etwas unterscheiden. Diese Funktionen berechnen den primären und den alternativen Code:

```text
dmetaphone(source text) returns text
dmetaphone_alt(source text) returns text
```

Für die Eingabezeichenketten gibt es keine Längenbeschränkung.

Beispiel:

```sql
SELECT dmetaphone('gumbo');
```

```text
 dmetaphone
------------
 KMP
(1 row)
```


### F.17. hstore — Schlüssel/Wert-Datentyp `hstore`

Dieses Modul implementiert den Datentyp `hstore`, mit dem Mengen von Schlüssel/Wert-Paaren in einem einzelnen PostgreSQL-Wert gespeichert werden können. Das kann in verschiedenen Szenarien nützlich sein, etwa bei Zeilen mit vielen selten abgefragten Attributen oder bei semistrukturierten Daten. Schlüssel und Werte sind einfache Textzeichenketten.

Dieses Modul gilt als „trusted“. Es kann also von Nicht-Superusern installiert werden, die das `CREATE`-Privileg für die aktuelle Datenbank besitzen.

#### F.17.1. Externe Darstellung von `hstore`

Die Textdarstellung eines `hstore`, die für Eingabe und Ausgabe verwendet wird, enthält null oder mehr durch Kommas getrennte Schlüssel/Wert-Paare. Einige Beispiele:

```text
k => v
foo => bar, baz => whatever
"1-a" => "anything at all"
```

Die Reihenfolge der Paare ist nicht bedeutend und wird bei der Ausgabe möglicherweise nicht beibehalten. Leerraum zwischen Paaren oder um das Zeichen `=>` wird ignoriert. Schlüssel und Werte, die Leerraum, Kommas, Gleichheitszeichen oder Größer-als-Zeichen enthalten, müssen in doppelte Anführungszeichen gesetzt werden. Ein doppeltes Anführungszeichen oder ein Backslash in einem Schlüssel oder Wert wird mit einem Backslash maskiert.

Jeder Schlüssel in einem `hstore` ist eindeutig. Wenn ein `hstore` mit doppelten Schlüsseln angegeben wird, wird nur einer davon gespeichert; es gibt keine Garantie, welcher erhalten bleibt:

```sql
SELECT 'a=>1,a=>2'::hstore;
```

```text
  hstore
----------
 "a"=>"1"
```

Ein Wert, aber kein Schlüssel, kann ein SQL-`NULL` sein. Beispiel:

```text
key => NULL
```

Das Schlüsselwort `NULL` ist unabhängig von Groß-/Kleinschreibung. Setzen Sie `NULL` in doppelte Anführungszeichen, um es als normale Zeichenkette `"NULL"` zu behandeln.

> **Hinweis:** Das Textformat von `hstore` wird bei der Eingabe angewendet, bevor eventuell erforderliches Quoting oder Escaping greift. Wenn Sie ein `hstore`-Literal über einen Parameter übergeben, ist keine zusätzliche Verarbeitung nötig. Wenn Sie es jedoch als gequotete Literal-Konstante übergeben, müssen einfache Anführungszeichen und je nach Einstellung des Konfigurationsparameters `standard_conforming_strings` auch Backslashes korrekt maskiert werden. Weitere Informationen zur Behandlung von Zeichenkettenkonstanten finden Sie in Abschnitt 4.1.2.1.

Bei der Ausgabe werden Schlüssel und Werte immer in doppelte Anführungszeichen gesetzt, auch wenn das streng genommen nicht nötig wäre.

#### F.17.2. `hstore`-Operatoren und -Funktionen

Die vom Modul `hstore` bereitgestellten Operatoren sind in Tabelle F.6 aufgeführt, die Funktionen in Tabelle F.7.

**Tabelle F.6. `hstore`-Operatoren**

| Operator | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `hstore -> text` → `text` | Gibt den zu einem Schlüssel gehörenden Wert zurück oder `NULL`, wenn er nicht vorhanden ist. | `'a=>x, b=>y'::hstore -> 'a'` → `x` |
| `hstore -> text[]` → `text[]` | Gibt die Werte zu den angegebenen Schlüsseln zurück oder `NULL`, wenn sie nicht vorhanden sind. | `'a=>x, b=>y, c=>z'::hstore -> ARRAY['c','a']` → `{"z","x"}` |
| `hstore || hstore` → `hstore` | Verkettet zwei `hstore`-Werte. | `'a=>b, c=>d'::hstore || 'c=>x, d=>q'::hstore` → `"a"=>"b", "c"=>"x", "d"=>"q"` |
| `hstore ? text` → `boolean` | Enthält der `hstore` den Schlüssel? | `'a=>1'::hstore ? 'a'` → `t` |
| `hstore ?& text[]` → `boolean` | Enthält der `hstore` alle angegebenen Schlüssel? | `'a=>1,b=>2'::hstore ?& ARRAY['a','b']` → `t` |
| `hstore ?| text[]` → `boolean` | Enthält der `hstore` einen der angegebenen Schlüssel? | `'a=>1,b=>2'::hstore ?| ARRAY['b','c']` → `t` |
| `hstore @> hstore` → `boolean` | Enthält der linke Operand den rechten? | `'a=>b, b=>1, c=>NULL'::hstore @> 'b=>1'` → `t` |
| `hstore <@ hstore` → `boolean` | Ist der linke Operand im rechten enthalten? | `'a=>c'::hstore <@ 'a=>b, b=>1, c=>NULL'` → `f` |
| `hstore - text` → `hstore` | Löscht einen Schlüssel aus dem linken Operanden. | `'a=>1, b=>2, c=>3'::hstore - 'b'::text` → `"a"=>"1", "c"=>"3"` |
| `hstore - text[]` → `hstore` | Löscht die angegebenen Schlüssel aus dem linken Operanden. | `'a=>1, b=>2, c=>3'::hstore - ARRAY['a','b']` → `"c"=>"3"` |
| `hstore - hstore` → `hstore` | Löscht Paare aus dem linken Operanden, die mit Paaren im rechten Operanden übereinstimmen. | `'a=>1, b=>2, c=>3'::hstore - 'a=>4, b=>2'::hstore` → `"a"=>"1", "c"=>"3"` |
| `anyelement #= hstore` → `anyelement` | Ersetzt Felder im linken Operanden, der ein zusammengesetzter Typ sein muss, durch passende Werte aus dem `hstore`. | `ROW(1,3) #= 'f1=>11'::hstore` → `(11,3)` |
| `%% hstore` → `text[]` | Wandelt einen `hstore` in ein Array mit abwechselnden Schlüsseln und Werten um. | `%% 'a=>foo, b=>bar'::hstore` → `{a,foo,b,bar}` |
| `%# hstore` → `text[]` | Wandelt einen `hstore` in ein zweidimensionales Schlüssel/Wert-Array um. | `%# 'a=>foo, b=>bar'::hstore` → `{{a,foo},{b,bar}}` |

**Tabelle F.7. `hstore`-Funktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `hstore(record)` → `hstore` | Erzeugt einen `hstore` aus einem Record oder einer Zeile. | `hstore(ROW(1,2))` → `"f1"=>"1", "f2"=>"2"` |
| `hstore(text[])` → `hstore` | Erzeugt einen `hstore` aus einem Array, entweder aus einem Schlüssel/Wert-Array oder einem zweidimensionalen Array. | `hstore(ARRAY['a','1','b','2'])` → `"a"=>"1", "b"=>"2"` |
| `hstore(text[], text[])` → `hstore` | Erzeugt einen `hstore` aus getrennten Schlüssel- und Wert-Arrays. | `hstore(ARRAY['a','b'], ARRAY['1','2'])` → `"a"=>"1", "b"=>"2"` |
| `hstore(text, text)` → `hstore` | Erzeugt einen `hstore` mit einem einzelnen Eintrag. | `hstore('a', 'b')` → `"a"=>"b"` |
| `akeys(hstore)` → `text[]` | Extrahiert die Schlüssel eines `hstore` als Array. | `akeys('a=>1,b=>2')` → `{a,b}` |
| `skeys(hstore)` → `setof text` | Extrahiert die Schlüssel eines `hstore` als Menge. | `skeys('a=>1,b=>2')` → `a`, `b` |
| `avals(hstore)` → `text[]` | Extrahiert die Werte eines `hstore` als Array. | `avals('a=>1,b=>2')` → `{1,2}` |
| `svals(hstore)` → `setof text` | Extrahiert die Werte eines `hstore` als Menge. | `svals('a=>1,b=>2')` |
| `hstore_to_array(hstore)` → `text[]` | Extrahiert Schlüssel und Werte als Array mit abwechselnden Schlüsseln und Werten. | `hstore_to_array('a=>1,b=>2')` → `{a,1,b,2}` |
| `hstore_to_matrix(hstore)` → `text[]` | Extrahiert Schlüssel und Werte als zweidimensionales Array. | `hstore_to_matrix('a=>1,b=>2')` → `{{a,1},{b,2}}` |
| `hstore_to_json(hstore)` → `json` | Wandelt einen `hstore` in einen `json`-Wert um und konvertiert alle Nicht-NULL-Werte in JSON-Zeichenketten. Diese Funktion wird implizit verwendet, wenn ein `hstore`-Wert nach `json` gecastet wird. | `hstore_to_json('"a key"=>1, b=>t, c=>null')` |
| `hstore_to_jsonb(hstore)` → `jsonb` | Wandelt einen `hstore` in einen `jsonb`-Wert um und konvertiert alle Nicht-NULL-Werte in JSON-Zeichenketten. Diese Funktion wird implizit verwendet, wenn ein `hstore`-Wert nach `jsonb` gecastet wird. | `hstore_to_jsonb('"a key"=>1, b=>t, c=>null')` |
| `hstore_to_json_loose(hstore)` → `json` | Wandelt einen `hstore` in einen `json`-Wert um, versucht aber numerische und boolesche Werte zu erkennen, sodass sie im JSON nicht gequotet werden. | `hstore_to_json_loose('"a key"=>1, b=>t, c=>null')` |
| `hstore_to_jsonb_loose(hstore)` → `jsonb` | Wandelt einen `hstore` in einen `jsonb`-Wert um, versucht aber numerische und boolesche Werte zu erkennen, sodass sie im JSON nicht gequotet werden. | `hstore_to_jsonb_loose('"a key"=>1, b=>t, c=>null')` |
| `slice(hstore, text[])` → `hstore` | Extrahiert eine Teilmenge eines `hstore`, die nur die angegebenen Schlüssel enthält. | `slice('a=>1,b=>2,c=>3'::hstore, ARRAY['b','c','x'])` → `"b"=>"2", "c"=>"3"` |
| `each(hstore)` → `setof record (key text, value text)` | Extrahiert Schlüssel und Werte eines `hstore` als Menge von Records. | `SELECT * FROM each('a=>1,b=>2')` |
| `exist(hstore, text)` → `boolean` | Enthält der `hstore` den Schlüssel? | `exist('a=>1', 'a')` → `t` |
| `defined(hstore, text)` → `boolean` | Enthält der `hstore` für den Schlüssel einen Nicht-NULL-Wert? | `defined('a=>NULL', 'a')` → `f` |
| `delete(hstore, text)` → `hstore` | Löscht das Paar mit passendem Schlüssel. | `delete('a=>1,b=>2', 'b')` → `"a"=>"1"` |
| `delete(hstore, text[])` → `hstore` | Löscht Paare mit passenden Schlüsseln. | `delete('a=>1,b=>2,c=>3', ARRAY['a','b'])` → `"c"=>"3"` |
| `delete(hstore, hstore)` → `hstore` | Löscht Paare, die mit denen im zweiten Argument übereinstimmen. | `delete('a=>1,b=>2', 'a=>4,b=>2'::hstore)` → `"a"=>"1"` |
| `populate_record(anyelement, hstore)` → `anyelement` | Ersetzt Felder im linken Operanden, der ein zusammengesetzter Typ sein muss, durch passende Werte aus dem `hstore`. | `populate_record(ROW(1,2), 'f1=>42'::hstore)` → `(42,2)` |

Zusätzlich zu diesen Operatoren und Funktionen können Werte des Typs `hstore` indiziert werden, sodass sie sich wie assoziative Arrays verhalten. Es kann nur ein einzelner Subscript des Typs `text` angegeben werden; er wird als Schlüssel interpretiert, und der entsprechende Wert wird gelesen oder gespeichert. Beispiel:

```sql
CREATE TABLE mytable (h hstore);
INSERT INTO mytable VALUES ('a=>b, c=>d');
SELECT h['a'] FROM mytable;
```

```text
 h
---
 b
(1 row)
```

```sql
UPDATE mytable SET h['c'] = 'new';
SELECT h FROM mytable;
```

```text
          h
----------------------
 "a"=>"b", "c"=>"new"
(1 row)
```

Ein indizierter Abruf gibt `NULL` zurück, wenn der Subscript `NULL` ist oder der Schlüssel im `hstore` nicht existiert. Damit unterscheidet sich ein indizierter Abruf kaum vom Operator `->`. Eine indizierte Aktualisierung schlägt fehl, wenn der Subscript `NULL` ist; andernfalls ersetzt sie den Wert für diesen Schlüssel und fügt einen Eintrag zum `hstore` hinzu, wenn der Schlüssel noch nicht existiert.

#### F.17.3. Indizes

`hstore` unterstützt GiST- und GIN-Indizes für die Operatoren `@>`, `?`, `?&` und `?|`. Beispiel:

```sql
CREATE INDEX hidx ON testhstore USING GIST (h);

CREATE INDEX hidx ON testhstore USING GIN (h);
```

Die GiST-Operatorklasse `gist_hstore_ops` approximiert eine Menge von Schlüssel/Wert-Paaren als Bitmap-Signatur. Ihr optionaler ganzzahliger Parameter `siglen` bestimmt die Signaturlänge in Byte. Der Standardwert beträgt 16 Byte. Zulässige Signaturlängen liegen zwischen 1 und 2024 Byte. Längere Signaturen führen zu einer präziseren Suche, bei der ein kleinerer Anteil des Index und weniger Heap-Seiten gelesen werden, allerdings auf Kosten eines größeren Index.

Beispiel für einen solchen Index mit einer Signaturlänge von 32 Byte:

```sql
CREATE INDEX hidx ON testhstore USING GIST (h
  gist_hstore_ops(siglen=32));
```

`hstore` unterstützt außerdem B-Tree- oder Hash-Indizes für den Operator `=`. Dadurch können `hstore`-Spalten als `UNIQUE` deklariert oder in Ausdrücken mit `GROUP BY`, `ORDER BY` oder `DISTINCT` verwendet werden. Die Sortierreihenfolge von `hstore`-Werten ist nicht besonders nützlich, aber diese Indizes können für Gleichheitsabfragen hilfreich sein. Indizes für `=`-Vergleiche erstellen Sie so:

```sql
CREATE INDEX hidx ON testhstore USING BTREE (h);

CREATE INDEX hidx ON testhstore USING HASH (h);
```

#### F.17.4. Beispiele

Einen Schlüssel hinzufügen oder einen vorhandenen Schlüssel mit einem neuen Wert aktualisieren:

```sql
UPDATE tab SET h['c'] = '3';
```

Eine andere Möglichkeit, dasselbe zu tun:

```sql
UPDATE tab SET h = h || hstore('c', '3');
```

Wenn mehrere Schlüssel in einer Operation hinzugefügt oder geändert werden sollen, ist der Verkettungsansatz effizienter als die Verwendung von Subscripts:

```sql
UPDATE tab SET h = h || hstore(array['q', 'w'], array['11', '12']);
```

Einen Schlüssel löschen:

```sql
UPDATE tab SET h = delete(h, 'k1');
```

Einen Record in einen `hstore` umwandeln:

```sql
CREATE TABLE test (col1 integer, col2 text, col3 text);
INSERT INTO test VALUES (123, 'foo', 'bar');

SELECT hstore(t) FROM test AS t;
```

```text
                   hstore
---------------------------------------------
 "col1"=>"123", "col2"=>"foo", "col3"=>"bar"
(1 row)
```

Einen `hstore` in einen vordefinierten Record-Typ umwandeln:

```sql
CREATE TABLE test (col1 integer, col2 text, col3 text);

SELECT * FROM populate_record(null::test,
                              '"col1"=>"456", "col2"=>"zzz"');
```

```text
 col1 | col2 | col3
------+------+------
  456 | zzz  |
(1 row)
```

Einen vorhandenen Record mit den Werten aus einem `hstore` ändern:

```sql
CREATE TABLE test (col1 integer, col2 text, col3 text);
INSERT INTO test VALUES (123, 'foo', 'bar');

SELECT (r).* FROM (SELECT t #= '"col3"=>"baz"' AS r FROM test t) s;
```

```text
 col1 | col2 | col3
------+------+------
  123 | foo  | baz
(1 row)
```

#### F.17.5. Statistiken

Der Typ `hstore` kann aufgrund seiner großen Freiheit viele unterschiedliche Schlüssel enthalten. Die Prüfung gültiger Schlüssel ist Aufgabe der Anwendung. Die folgenden Beispiele zeigen mehrere Techniken zum Prüfen von Schlüsseln und zum Ermitteln von Statistiken.

Ein einfaches Beispiel:

```sql
SELECT * FROM each('aaa=>bq, b=>NULL, ""=>1');
```

Mit einer Tabelle:

```sql
CREATE TABLE stat AS SELECT (each(h)).key, (each(h)).value FROM
  testhstore;
```

Online-Statistiken:

```sql
SELECT key, count(*) FROM
  (SELECT (each(h)).key FROM testhstore) AS stat
  GROUP BY key
  ORDER BY count DESC, key;
```

```text
   key   | count
---------+-------
 line    |   883
 query   |   207
 pos     |   203
 node    |   202
 space   |   197
 status  |   195
 public  |   194
 title   |   190
 org     |   189
 ...
```

#### F.17.6. Kompatibilität

Seit PostgreSQL 9.0 verwendet `hstore` eine andere interne Darstellung als frühere Versionen. Das ist für Upgrades per Dump/Restore kein Hindernis, da die Textdarstellung, die im Dump verwendet wird, unverändert ist.

Bei einem binären Upgrade bleibt die Aufwärtskompatibilität dadurch erhalten, dass der neue Code Daten im alten Format erkennt. Das führt zu einem kleinen Performance-Nachteil, wenn Daten verarbeitet werden, die noch nicht durch den neuen Code geändert wurden. Ein Upgrade aller Werte in einer Tabellenspalte kann mit einer `UPDATE`-Anweisung erzwungen werden:

```sql
UPDATE tablename SET hstorecol = hstorecol || '';
```

Eine andere Möglichkeit:

```sql
ALTER TABLE tablename ALTER hstorecol TYPE hstore USING hstorecol
  || '';
```

Die Methode mit `ALTER TABLE` erfordert eine `ACCESS EXCLUSIVE`-Sperre auf der Tabelle, führt aber nicht dazu, dass die Tabelle durch alte Zeilenversionen aufgebläht wird.

#### F.17.7. Transformationen

Zusätzliche Erweiterungen implementieren Transformationen für den Typ `hstore` für die Sprachen PL/Perl und PL/Python. Die Erweiterungen für PL/Perl heißen `hstore_plperl` und `hstore_plperlu`, für vertrauenswürdiges beziehungsweise nicht vertrauenswürdiges PL/Perl. Wenn Sie diese Transformationen installieren und beim Erstellen einer Funktion angeben, werden `hstore`-Werte auf Perl-Hashes abgebildet. Die Erweiterung für PL/Python heißt `hstore_plpython3u`; bei ihrer Verwendung werden `hstore`-Werte auf Python-Dictionaries abgebildet.

#### F.17.8. Autoren

Oleg Bartunov <oleg@sai.msu.su>, Moscow, Moscow University, Russia

Teodor Sigaev <teodor@sigaev.ru>, Moscow, Delta-Soft Ltd., Russia

Zusätzliche Verbesserungen von Andrew Gierth <andrew@tao11.riddles.org.uk>, United Kingdom


### F.18. intagg — Aggregator und Enumerator für ganze Zahlen

Das Modul `intagg` stellt einen Aggregator und einen Enumerator für ganze Zahlen bereit. `intagg` ist inzwischen veraltet, weil eingebaute Funktionen einen Oberumfang seiner Fähigkeiten bereitstellen. Das Modul wird jedoch weiterhin als Kompatibilitäts-Wrapper um die eingebauten Funktionen mitgeliefert.

#### F.18.1. Funktionen

Der Aggregator ist die Aggregatfunktion `int_array_aggregate(integer)`. Sie erzeugt ein Integer-Array, das genau die ihr übergebenen ganzen Zahlen enthält. Dies ist ein Wrapper um `array_agg`, das dasselbe für beliebige Array-Typen leistet.

Der Enumerator ist die Funktion `int_array_enum(integer[])`, die `setof integer` zurückgibt. Sie ist im Wesentlichen die Umkehrung des Aggregators: Aus einem Array ganzer Zahlen wird eine Menge von Zeilen erzeugt. Dies ist ein Wrapper um `unnest`, das dasselbe für beliebige Array-Typen leistet.

#### F.18.2. Beispielverwendungen

Viele Datenbanksysteme kennen das Konzept einer Many-to-Many-Tabelle. Eine solche Tabelle steht typischerweise zwischen zwei indizierten Tabellen, zum Beispiel:

```sql
CREATE TABLE left_table (id INT PRIMARY KEY, ...);
CREATE TABLE right_table (id INT PRIMARY KEY, ...);
CREATE TABLE many_to_many(id_left INT REFERENCES left_table,
                          id_right INT REFERENCES right_table);
```

Sie wird typischerweise so verwendet:

```sql
SELECT right_table.*
FROM right_table JOIN many_to_many ON (right_table.id =
                                       many_to_many.id_right)
WHERE many_to_many.id_left = item;
```

Dies gibt alle Einträge in der rechten Tabelle für einen Eintrag in der linken Tabelle zurück. Das ist ein sehr häufiges SQL-Konstrukt.

Bei sehr vielen Einträgen in der Tabelle `many_to_many` kann diese Methode jedoch umständlich werden. Ein solcher Join führt häufig zu einem Index-Scan und einem Fetch für jeden rechten Tabelleneintrag, der zu einem bestimmten linken Eintrag gehört. In einem sehr dynamischen System lässt sich daran wenig ändern. Wenn die Daten jedoch relativ statisch sind, können Sie mit dem Aggregator eine Zusammenfassungstabelle erstellen.

```sql
CREATE TABLE summary AS
  SELECT id_left, int_array_aggregate(id_right) AS rights
  FROM many_to_many
  GROUP BY id_left;
```

Dadurch entsteht eine Tabelle mit einer Zeile pro linkem Eintrag und einem Array rechter Einträge. Ohne eine Möglichkeit, das Array zu verwenden, wäre das wenig nützlich; deshalb gibt es den Array-Enumerator. Sie können schreiben:

```sql
SELECT id_left, int_array_enum(rights) FROM summary WHERE id_left
  = item;
```

Die obige Abfrage mit `int_array_enum` erzeugt dieselben Ergebnisse wie:

```sql
SELECT id_left, id_right FROM many_to_many WHERE id_left = item;
```

Der Unterschied besteht darin, dass die Abfrage gegen die Zusammenfassungstabelle nur eine Zeile aus der Tabelle lesen muss, während die direkte Abfrage gegen `many_to_many` für jeden Eintrag einen Index-Scan und Fetch ausführen muss.

Auf einem System zeigte `EXPLAIN`, dass eine Abfrage mit Kosten von `8488` auf Kosten von `329` reduziert wurde. Die ursprüngliche Abfrage war ein Join mit der Tabelle `many_to_many`, der durch Folgendes ersetzt wurde:

```sql
SELECT id_right, count(id_right) FROM
  ( SELECT id_left, int_array_enum(rights) AS id_right
    FROM summary
    JOIN (SELECT id FROM left_table
          WHERE id = item) AS lefts
    ON (summary.id_left = lefts.id)
  ) AS list
  GROUP BY id_right
  ORDER BY count DESC;
```


### F.19. intarray — Arrays ganzer Zahlen bearbeiten

Das Modul `intarray` stellt eine Reihe nützlicher Funktionen und Operatoren zum Bearbeiten von NULL-freien Arrays ganzer Zahlen bereit. Außerdem gibt es Unterstützung für indizierte Suchen mit einigen dieser Operatoren.

Alle diese Operationen lösen einen Fehler aus, wenn ein übergebenes Array `NULL`-Elemente enthält.

Viele dieser Operationen sind nur für eindimensionale Arrays sinnvoll. Obwohl sie Eingabearrays mit mehr Dimensionen akzeptieren, werden die Daten so behandelt, als handele es sich in Speicherreihenfolge um ein lineares Array.

Dieses Modul gilt als „trusted“. Es kann also von Nicht-Superusern installiert werden, die das `CREATE`-Privileg für die aktuelle Datenbank besitzen.

#### F.19.1. `intarray`-Funktionen und -Operatoren

Die vom Modul `intarray` bereitgestellten Funktionen sind in Tabelle F.8 aufgeführt, die Operatoren in Tabelle F.9.

**Tabelle F.8. `intarray`-Funktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `icount(integer[])` → `integer` | Gibt die Anzahl der Elemente im Array zurück. | `icount('{1,2,3}'::integer[])` → `3` |
| `sort(integer[], dir text)` → `integer[]` | Sortiert das Array aufsteigend oder absteigend. `dir` muss `asc` oder `desc` sein. | `sort('{1,3,2}'::integer[], 'desc')` → `{3,2,1}` |
| `sort(integer[])` → `integer[]`; `sort_asc(integer[])` → `integer[]` | Sortiert aufsteigend. | `sort(array[11,77,44])` → `{11,44,77}` |
| `sort_desc(integer[])` → `integer[]` | Sortiert absteigend. | `sort_desc(array[11,77,44])` → `{77,44,11}` |
| `uniq(integer[])` → `integer[]` | Entfernt benachbarte Duplikate. Wird oft zusammen mit `sort` verwendet, um alle Duplikate zu entfernen. | `uniq('{1,2,2,3,1,1}'::integer[])` → `{1,2,3,1}`; `uniq(sort('{1,2,3,2,1}'::integer[]))` → `{1,2,3}` |
| `idx(integer[], item integer)` → `integer` | Gibt den Index des ersten Array-Elements zurück, das `item` entspricht, oder `0`, wenn es keinen Treffer gibt. | `idx(array[11,22,33,22,11], 22)` → `2` |
| `subarray(integer[], start integer, len integer)` → `integer[]` | Extrahiert den Teil des Arrays ab Position `start` mit `len` Elementen. | `subarray('{1,2,3,2,1}'::integer[], 2, 3)` → `{2,3,2}` |
| `subarray(integer[], start integer)` → `integer[]` | Extrahiert den Teil des Arrays ab Position `start`. | `subarray('{1,2,3,2,1}'::integer[], 2)` → `{2,3,2,1}` |
| `intset(integer)` → `integer[]` | Erzeugt ein Array mit einem einzelnen Element. | `intset(42)` → `{42}` |

**Tabelle F.9. `intarray`-Operatoren**

| Operator | Beschreibung |
| --- | --- |
| `integer[] && integer[]` → `boolean` | Überlappen sich die Arrays, haben sie also mindestens ein Element gemeinsam? |
| `integer[] @> integer[]` → `boolean` | Enthält das linke Array das rechte Array? |
| `integer[] <@ integer[]` → `boolean` | Ist das linke Array im rechten enthalten? |
| `# integer[]` → `integer` | Gibt die Anzahl der Elemente im Array zurück. |
| `integer[] # integer` → `integer` | Gibt den Index des ersten Array-Elements zurück, das dem rechten Argument entspricht, oder `0`, wenn es keinen Treffer gibt. Entspricht der Funktion `idx`. |
| `integer[] + integer` → `integer[]` | Fügt ein Element am Ende des Arrays hinzu. |
| `integer[] + integer[]` → `integer[]` | Verkettet die Arrays. |
| `integer[] - integer` → `integer[]` | Entfernt Einträge aus dem Array, die dem rechten Argument entsprechen. |
| `integer[] - integer[]` → `integer[]` | Entfernt Elemente des rechten Arrays aus dem linken Array. |
| `integer[] \| integer` → `integer[]` | Bildet die Vereinigung der Argumente. |
| `integer[] \| integer[]` → `integer[]` | Bildet die Vereinigung der Argumente. |
| `integer[] & integer[]` → `integer[]` | Bildet den Schnitt der Argumente. |
| `integer[] @@ query_int` → `boolean` | Erfüllt das Array die Abfrage? Siehe unten. |
| `query_int ~~ integer[]` → `boolean` | Erfüllt das Array die Abfrage? Kommutator von `@@`. |

Die Operatoren `&&`, `@>` und `<@` entsprechen den eingebauten PostgreSQL-Operatoren gleichen Namens, arbeiten aber nur mit Integer-Arrays, die keine NULL-Werte enthalten. Die eingebauten Operatoren funktionieren dagegen mit beliebigen Array-Typen. Diese Einschränkung macht die Operatoren in vielen Fällen schneller als die eingebauten Varianten.

Die Operatoren `@@` und `~~` prüfen, ob ein Array eine Abfrage erfüllt, die als Wert des spezialisierten Datentyps `query_int` ausgedrückt wird. Eine Abfrage besteht aus ganzzahligen Werten, die gegen die Elemente des Arrays geprüft werden, optional kombiniert mit den Operatoren `&` (AND), `|` (OR) und `!` (NOT). Klammern können nach Bedarf verwendet werden. Die Abfrage `1&(2|3)` passt zum Beispiel auf Arrays, die `1` und außerdem entweder `2` oder `3` enthalten.

#### F.19.2. Indexunterstützung

`intarray` stellt Indexunterstützung für die Operatoren `&&`, `@>` und `@@` sowie für normale Array-Gleichheit bereit.

Es werden zwei parametrisierte GiST-Index-Operatorklassen bereitgestellt: `gist__int_ops`, die standardmäßig verwendet wird, eignet sich für kleine bis mittelgroße Datenmengen. `gist__intbig_ops` verwendet eine größere Signatur und eignet sich besser für große Datenmengen, also Spalten mit vielen unterschiedlichen Array-Werten. Die Implementierung verwendet eine RD-Tree-Datenstruktur mit eingebauter verlustbehafteter Kompression.

`gist__int_ops` approximiert eine Menge ganzer Zahlen als Array von Integer-Bereichen. Der optionale ganzzahlige Parameter `numranges` bestimmt die maximale Anzahl von Bereichen in einem Indexschlüssel. Der Standardwert von `numranges` ist `100`. Gültige Werte liegen zwischen `1` und `253`. Größere Arrays als GiST-Indexschlüssel führen zu einer präziseren Suche, bei der ein kleinerer Anteil des Index und weniger Heap-Seiten gelesen werden, allerdings auf Kosten eines größeren Index.

`gist__intbig_ops` approximiert eine Menge ganzer Zahlen als Bitmap-Signatur. Der optionale ganzzahlige Parameter `siglen` bestimmt die Signaturlänge in Byte. Die Standardsignaturlänge beträgt 16 Byte. Gültige Werte liegen zwischen `1` und `2024` Byte. Längere Signaturen führen zu einer präziseren Suche, bei der ein kleinerer Anteil des Index und weniger Heap-Seiten gelesen werden, allerdings auf Kosten eines größeren Index.

Außerdem gibt es die nicht standardmäßige GIN-Operatorklasse `gin__int_ops`, die diese Operatoren sowie `<@` unterstützt.

Die Wahl zwischen GiST- und GIN-Indizierung hängt von den relativen Performance-Eigenschaften von GiST und GIN ab, die an anderer Stelle besprochen werden.

#### F.19.3. Beispiel

```sql
-- a message can be in one or more "sections"
CREATE TABLE message (mid INT PRIMARY KEY, sections INT[], ...);

-- create specialized index with signature length of 32 bytes
CREATE INDEX message_rdtree_idx ON message USING GIST (sections
  gist__intbig_ops (siglen = 32));

-- select messages in section 1 OR 2 - OVERLAP operator
SELECT message.mid FROM message WHERE message.sections && '{1,2}';

-- select messages in sections 1 AND 2 - CONTAINS operator
SELECT message.mid FROM message WHERE message.sections @> '{1,2}';

-- the same, using QUERY operator
SELECT message.mid FROM message WHERE message.sections @@
  '1&2'::query_int;
```

#### F.19.4. Benchmark

Das Quellverzeichnis `contrib/intarray/bench` enthält eine Benchmark-Test-Suite, die gegen einen installierten PostgreSQL-Server ausgeführt werden kann. Dafür muss außerdem `DBD::Pg` installiert sein. Zum Ausführen:

```text
cd .../contrib/intarray/bench
createdb TEST

psql -c "CREATE EXTENSION intarray" TEST
./create_test.pl | psql TEST
./bench.pl
```

Das Skript `bench.pl` besitzt zahlreiche Optionen, die angezeigt werden, wenn es ohne Argumente ausgeführt wird.

#### F.19.5. Autoren

Die gesamte Arbeit wurde von Teodor Sigaev (<teodor@sigaev.ru>) und Oleg Bartunov (<oleg@sai.msu.su>) durchgeführt. Weitere Informationen finden Sie unter <http://www.sai.msu.su/~megera/postgres/gist/>. Andrey Oktyabrski leistete wichtige Arbeit beim Hinzufügen neuer Funktionen und Operationen.


### F.20. isn — Datentypen für internationale Standardnummern (ISBN, EAN, UPC usw.)

Das Modul `isn` stellt Datentypen für die folgenden internationalen Produktnummerierungsstandards bereit: `EAN13`, `UPC`, `ISBN` für Bücher, `ISMN` für Musik und `ISSN` für fortlaufende Veröffentlichungen. Nummern werden bei der Eingabe anhand einer fest einkompilierten Präfixliste validiert; dieselbe Präfixliste wird auch verwendet, um Nummern bei der Ausgabe mit Bindestrichen zu formatieren. Da von Zeit zu Zeit neue Präfixe vergeben werden, kann diese Liste veraltet sein. Eine künftige Version dieses Moduls könnte die Präfixliste aus einer oder mehreren Tabellen beziehen, die Benutzer bei Bedarf aktualisieren können; derzeit kann die Liste jedoch nur durch Ändern des Quellcodes und erneutes Kompilieren aktualisiert werden. Alternativ könnten Präfixvalidierung und Bindestrichunterstützung in einer künftigen Version dieses Moduls entfallen.

Dieses Modul gilt als „trusted“. Es kann also von Nicht-Superusern installiert werden, die das `CREATE`-Privileg für die aktuelle Datenbank besitzen.

#### F.20.1. Datentypen

Tabelle F.10 zeigt die vom Modul `isn` bereitgestellten Datentypen.

**Tabelle F.10. `isn`-Datentypen**

| Datentyp | Beschreibung |
| --- | --- |
| `EAN13` | European Article Numbers, immer im Anzeigeformat `EAN13`. |
| `ISBN13` | International Standard Book Numbers, angezeigt im neuen `EAN13`-Anzeigeformat. |
| `ISMN13` | International Standard Music Numbers, angezeigt im neuen `EAN13`-Anzeigeformat. |
| `ISSN13` | International Standard Serial Numbers, angezeigt im neuen `EAN13`-Anzeigeformat. |
| `ISBN` | International Standard Book Numbers, angezeigt im alten kurzen Anzeigeformat. |
| `ISMN` | International Standard Music Numbers, angezeigt im alten kurzen Anzeigeformat. |
| `ISSN` | International Standard Serial Numbers, angezeigt im alten kurzen Anzeigeformat. |
| `UPC` | Universal Product Codes. |

Einige Hinweise:

1. `ISBN13`-, `ISMN13`- und `ISSN13`-Nummern sind alle `EAN13`-Nummern.
2. `EAN13`-Nummern sind nicht immer `ISBN13`-, `ISMN13`- oder `ISSN13`-Nummern; manche sind es.
3. Einige `ISBN13`-Nummern können als `ISBN` angezeigt werden.
4. Einige `ISMN13`-Nummern können als `ISMN` angezeigt werden.
5. Einige `ISSN13`-Nummern können als `ISSN` angezeigt werden.
6. `UPC`-Nummern sind eine Teilmenge der `EAN13`-Nummern; im Wesentlichen sind sie `EAN13` ohne die erste Ziffer `0`.
7. Alle `UPC`-, `ISBN`-, `ISMN`- und `ISSN`-Nummern können als `EAN13`-Nummern dargestellt werden.

Intern verwenden alle diese Typen dieselbe Repräsentation, nämlich einen 64-Bit-Integer, und sind untereinander austauschbar. Mehrere Typen werden bereitgestellt, um die Anzeigeformatierung zu steuern und eine strengere Gültigkeitsprüfung für Eingaben zu ermöglichen, die eine bestimmte Art von Nummer bezeichnen sollen.

Die Typen `ISBN`, `ISMN` und `ISSN` zeigen, wann immer möglich, die kurze Version der Nummer (`ISxN 10`) an; für Nummern, die nicht in die Kurzversion passen, zeigen sie das Format `ISxN 13`. Die Typen `EAN13`, `ISBN13`, `ISMN13` und `ISSN13` zeigen immer die lange Version der `ISxN`, also `EAN13`, an.

#### F.20.2. Typumwandlungen

Das Modul `isn` stellt die folgenden Paare von Typumwandlungen bereit:

- `ISBN13` <=> `EAN13`
- `ISMN13` <=> `EAN13`
- `ISSN13` <=> `EAN13`
- `ISBN` <=> `EAN13`
- `ISMN` <=> `EAN13`
- `ISSN` <=> `EAN13`
- `UPC` <=> `EAN13`
- `ISBN` <=> `ISBN13`
- `ISMN` <=> `ISMN13`
- `ISSN` <=> `ISSN13`

Beim Cast von `EAN13` in einen anderen Typ wird zur Laufzeit geprüft, ob der Wert im Wertebereich des anderen Typs liegt; andernfalls wird ein Fehler ausgelöst. Die anderen Casts sind lediglich Umbenennungen und gelingen immer.

#### F.20.3. Funktionen und Operatoren

Das Modul `isn` stellt die Standardvergleichsoperatoren sowie B-Tree- und Hash-Indexunterstützung für alle diese Datentypen bereit. Zusätzlich gibt es mehrere spezialisierte Funktionen, siehe Tabelle F.11. In dieser Tabelle steht `isn` für einen beliebigen Datentyp des Moduls.

**Tabelle F.11. `isn`-Funktionen**

| Funktion | Beschreibung |
| --- | --- |
| `make_valid(isn)` → `isn` | Löscht das Kennzeichen für eine ungültige Prüfziffer des Werts. |
| `is_valid(isn)` → `boolean` | Prüft, ob das Kennzeichen für eine ungültige Prüfziffer vorhanden ist. |
| `isn_weak(boolean)` → `boolean` | Setzt den schwachen Eingabemodus und gibt die neue Einstellung zurück. Diese Funktion bleibt aus Gründen der Rückwärtskompatibilität erhalten. Empfohlen ist die Einstellung über den Konfigurationsparameter `isn.weak`. |
| `isn_weak()` → `boolean` | Gibt den aktuellen Status des schwachen Modus zurück. Diese Funktion bleibt aus Gründen der Rückwärtskompatibilität erhalten. Empfohlen ist die Prüfung über den Konfigurationsparameter `isn.weak`. |

#### F.20.4. Konfigurationsparameter

`isn.weak` (`boolean`)

`isn.weak` aktiviert den schwachen Eingabemodus. In diesem Modus werden ISN-Eingabewerte auch dann akzeptiert, wenn ihre Prüfziffer falsch ist. Der Standardwert ist `false`; damit werden ungültige Prüfziffern zurückgewiesen.

Warum sollte man den schwachen Modus verwenden? Vielleicht haben Sie eine große Sammlung von ISBN-Nummern, und einige davon besitzen aus irgendwelchen Gründen eine falsche Prüfziffer. Vielleicht wurden die Nummern aus einer gedruckten Liste gescannt und die OCR hat Fehler gemacht, vielleicht wurden sie manuell erfasst. In jedem Fall möchten Sie die Daten möglicherweise bereinigen, aber zunächst alle Nummern in der Datenbank halten und eventuell ein externes Werkzeug verwenden, um ungültige Nummern in der Datenbank zu finden, die Informationen zu prüfen und sie leichter zu validieren. Dazu könnten Sie zum Beispiel alle ungültigen Nummern in der Tabelle auswählen.

Wenn Sie im schwachen Modus ungültige Nummern in eine Tabelle einfügen, wird die Nummer mit korrigierter Prüfziffer eingefügt, aber mit einem Ausrufezeichen `!` am Ende angezeigt, zum Beispiel `0-11-000322-5!`. Dieses Ungültigkeitskennzeichen kann mit der Funktion `is_valid` geprüft und mit `make_valid` gelöscht werden.

Sie können auch außerhalb des schwachen Modus das Einfügen als ungültig markierter Nummern erzwingen, indem Sie das Zeichen `!` an das Ende der Nummer anhängen.

Eine weitere Besonderheit ist, dass Sie bei der Eingabe anstelle der Prüfziffer `?` schreiben können; die korrekte Prüfziffer wird dann automatisch eingesetzt.

#### F.20.5. Beispiele

Die Typen direkt verwenden:

```sql
SELECT isbn('978-0-393-04002-9');
SELECT isbn13('0901690546');
SELECT issn('1436-4522');
```

Typen umwandeln:

```sql
-- note that you can only cast from ean13 to another type when the
-- number would be valid in the realm of the target type;
-- thus, the following will NOT work:
isbn(ean13('0220356483481'));
-- but these will:
SELECT upc(ean13('0220356483481'));
SELECT ean13(upc('220356483481'));
```

Eine Tabelle mit einer einzelnen Spalte für ISBN-Nummern erstellen:

```sql
CREATE TABLE test (id isbn);
INSERT INTO test VALUES('9780393040029');
```

Prüfziffern automatisch berechnen; beachten Sie das `?`:

```sql
INSERT INTO test VALUES('220500896?');
INSERT INTO test VALUES('978055215372?');

SELECT issn('3251231?');

SELECT ismn('979047213542?');
```

Den schwachen Modus verwenden:

```sql
SET isn.weak TO true;
INSERT INTO test VALUES('978-0-11-000533-4');
INSERT INTO test VALUES('9780141219307');
INSERT INTO test VALUES('2-205-00876-X');
SET isn.weak TO false;

SELECT id FROM test WHERE NOT is_valid(id);
UPDATE test SET id = make_valid(id) WHERE id = '2-205-00876-X!';

SELECT * FROM test;

SELECT isbn13(id) FROM test;
```

#### F.20.6. Bibliographie

Die Informationen zur Implementierung dieses Moduls wurden aus mehreren Quellen gesammelt, darunter:

- <https://www.isbn-international.org/>
- <https://www.issn.org/>
- <https://www.ismn-international.org/>
- <https://www.wikipedia.org/>

Die für die Bindestrichsetzung verwendeten Präfixe wurden außerdem aus folgenden Quellen zusammengestellt:

- <https://www.gs1.org/standards/id-keys>
- <https://en.wikipedia.org/wiki/List_of_ISBN_registration_groups>
- <https://www.isbn-international.org/content/isbn-users-manual/29>
- <https://en.wikipedia.org/wiki/International_Standard_Music_Number>
- <https://www.ismn-international.org/ranges/tools>

Bei der Erstellung der Algorithmen wurde sorgfältig vorgegangen; sie wurden genau gegen die vorgeschlagenen Algorithmen in den offiziellen Benutzerhandbüchern für ISBN, ISMN und ISSN geprüft.

#### F.20.7. Autor

Germán Méndez Bravo (Kronuz), 2004–2006

Dieses Modul wurde von Garrett A. Wollmans Code `isbn_issn` inspiriert.


### F.21. lo — Large Objects verwalten

Das Modul `lo` unterstützt die Verwaltung von Large Objects, auch LOs oder BLOBs genannt. Dazu gehören ein Datentyp `lo` und ein Trigger `lo_manage`.

Dieses Modul gilt als „trusted“. Es kann also von Nicht-Superusern installiert werden, die das `CREATE`-Privileg für die aktuelle Datenbank besitzen.

#### F.21.1. Begründung

Eines der Probleme mit dem JDBC-Treiber, das auch den ODBC-Treiber betrifft, besteht darin, dass die Spezifikation annimmt, dass Verweise auf BLOBs (Binary Large Objects) in einer Tabelle gespeichert werden und dass beim Ändern eines solchen Eintrags das zugehörige BLOB aus der Datenbank gelöscht wird.

In PostgreSQL geschieht das nicht. Large Objects werden als eigenständige Objekte behandelt. Ein Tabelleneintrag kann über eine OID auf ein Large Object verweisen, aber mehrere Tabelleneinträge können dieselbe Large-Object-OID referenzieren. Deshalb löscht das System das Large Object nicht nur deshalb, weil ein solcher Eintrag geändert oder entfernt wird.

Für PostgreSQL-spezifische Anwendungen ist das in Ordnung. Standardcode, der JDBC oder ODBC verwendet, löscht die Objekte jedoch nicht. Dadurch entstehen verwaiste Objekte, die von nichts mehr referenziert werden und lediglich Speicherplatz belegen.

Das Modul `lo` ermöglicht eine Korrektur, indem ein Trigger an Tabellen angehängt wird, die LO-Referenzspalten enthalten. Der Trigger führt im Wesentlichen ein `lo_unlink` aus, wenn ein Wert gelöscht oder geändert wird, der auf ein Large Object verweist. Wenn Sie diesen Trigger verwenden, nehmen Sie an, dass es für jedes Large Object, das in einer triggergesteuerten Spalte referenziert wird, nur eine Datenbankreferenz gibt.

Das Modul stellt außerdem den Datentyp `lo` bereit, der eigentlich nur eine Domain über dem Typ `oid` ist. Das ist nützlich, um Datenbankspalten, die Large-Object-Referenzen enthalten, von Spalten zu unterscheiden, die OIDs anderer Dinge enthalten. Sie müssen den Typ `lo` nicht verwenden, um den Trigger zu nutzen; er kann aber praktisch sein, um nachzuvollziehen, welche Spalten in Ihrer Datenbank Large Objects repräsentieren, die Sie mit dem Trigger verwalten. Außerdem heißt es, dass der ODBC-Treiber durcheinandergerät, wenn `lo` für BLOB-Spalten nicht verwendet wird.

#### F.21.2. Verwendung

Ein einfaches Verwendungsbeispiel:

```sql
CREATE TABLE image (title text, raster lo);

CREATE TRIGGER t_raster BEFORE UPDATE OR DELETE ON image
FOR EACH ROW EXECUTE FUNCTION lo_manage(raster);
```

Erstellen Sie für jede Spalte, die eindeutige Referenzen auf Large Objects enthalten soll, einen `BEFORE UPDATE OR DELETE`-Trigger und geben Sie den Spaltennamen als einziges Triggerargument an. Sie können den Trigger mit `BEFORE UPDATE OF column_name` auch so einschränken, dass er nur bei Aktualisierungen dieser Spalte ausgeführt wird. Wenn Sie mehrere `lo`-Spalten in derselben Tabelle benötigen, erstellen Sie für jede einen eigenen Trigger und geben Sie jedem Trigger in derselben Tabelle einen anderen Namen.

#### F.21.3. Einschränkungen

- Beim Löschen einer Tabelle werden die darin enthaltenen Objekte weiterhin verwaisen, da der Trigger nicht ausgeführt wird. Sie können das vermeiden, indem Sie vor `DROP TABLE` ein `DELETE FROM table` ausführen.

  `TRUNCATE` hat dasselbe Risiko.

  Wenn Sie bereits verwaiste Large Objects haben oder vermuten, dass Sie solche haben, hilft das Modul `vacuumlo` beim Aufräumen. Es ist sinnvoll, `vacuumlo` gelegentlich als Absicherung zum Trigger `lo_manage` auszuführen.

- Einige Frontends erstellen möglicherweise eigene Tabellen und legen die zugehörigen Trigger nicht an. Außerdem erinnern sich Benutzer möglicherweise nicht daran oder wissen nicht, dass sie die Trigger erstellen müssen.

#### F.21.4. Autor

Peter Mount <peter@retep.org.uk>


### F.22. ltree — hierarchischer baumartiger Datentyp

Dieses Modul implementiert den Datentyp `ltree` zur Darstellung von Labels für Daten, die in einer hierarchischen, baumartigen Struktur gespeichert sind. Es stellt umfangreiche Möglichkeiten bereit, solche Label-Bäume zu durchsuchen.

Dieses Modul gilt als „trusted“. Es kann also von Nicht-Superusern installiert werden, die das `CREATE`-Privileg für die aktuelle Datenbank besitzen.

#### F.22.1. Definitionen

Ein Label ist eine Folge alphanumerischer Zeichen, Unterstriche und Bindestriche. Welche alphanumerischen Zeichen gültig sind, hängt von der Locale der Datenbank ab. In der Locale `C` sind zum Beispiel die Zeichen `A-Z`, `a-z`, `0-9`, `_` und `-` erlaubt. Labels dürfen höchstens 1000 Zeichen lang sein.

Beispiele:

```text
42
Personal_Services
```

Ein Label-Pfad ist eine Folge von null oder mehr Labels, die durch Punkte getrennt sind, zum Beispiel `L1.L2.L3`. Er stellt einen Pfad von der Wurzel eines hierarchischen Baums zu einem bestimmten Knoten dar. Die Länge eines Label-Pfads darf 65535 Labels nicht überschreiten.

Beispiel:

```text
Top.Countries.Europe.Russia
```

Das Modul `ltree` stellt mehrere Datentypen bereit:

- `ltree`
  - Speichert einen Label-Pfad.

- `lquery`
  - Stellt ein regulärem Ausdruck ähnliches Muster zum Abgleich von `ltree`-Werten dar. Ein einfaches Wort passt auf dieses Label innerhalb eines Pfads. Ein Sternsymbol `*` passt auf null oder mehr Labels. Solche Elemente können mit Punkten verbunden werden, um ein Muster zu bilden, das auf den gesamten Label-Pfad passen muss.

Beispiele für `lquery`:

| Muster | Bedeutung |
| --- | --- |
| `foo` | Passt exakt auf den Label-Pfad `foo`. |
| `*.foo.*` | Passt auf jeden Label-Pfad, der das Label `foo` enthält. |
| `*.foo` | Passt auf jeden Label-Pfad, dessen letztes Label `foo` ist. |

Sternsymbole und einfache Wörter können quantifiziert werden, um einzuschränken, auf wie viele Labels sie passen dürfen:

| Muster | Bedeutung |
| --- | --- |
| `*{n}` | Passt auf genau `n` Labels. |
| `*{n,}` | Passt auf mindestens `n` Labels. |
| `*{n,m}` | Passt auf mindestens `n`, aber höchstens `m` Labels. |
| `*{,m}` | Passt auf höchstens `m` Labels; entspricht `*{0,m}`. |
| `foo{n,m}` | Passt auf mindestens `n`, aber höchstens `m` Vorkommen von `foo`. |
| `foo{,}` | Passt auf beliebig viele Vorkommen von `foo`, einschließlich null. |

Ohne expliziten Quantifizierer passt ein Sternsymbol standardmäßig auf beliebig viele Labels, also `{,}`. Ein Nicht-Stern-Element passt standardmäßig genau einmal, also `{1}`.

An ein Nicht-Stern-Element in `lquery` können mehrere Modifikatoren angehängt werden, damit es nicht nur auf exakte Übereinstimmungen passt:

| Modifikator | Bedeutung |
| --- | --- |
| `@` | Vergleich ohne Beachtung der Groß-/Kleinschreibung, zum Beispiel passt `a@` auf `A`. |
| `*` | Passt auf jedes Label mit diesem Präfix, zum Beispiel passt `foo*` auf `foobar`. |
| `%` | Passt auf anfängliche durch Unterstriche getrennte Wörter. |

Das Verhalten von `%` ist etwas kompliziert. Es versucht Wörter statt des gesamten Labels abzugleichen. Zum Beispiel passt `foo_bar%` auf `foo_bar_baz`, aber nicht auf `foo_barbaz`. In Kombination mit `*` wird der Präfixabgleich auf jedes Wort separat angewendet; `foo_bar%*` passt zum Beispiel auf `foo1_bar2_baz`, aber nicht auf `foo1_br2_baz`.

Außerdem können mehrere möglicherweise modifizierte Nicht-Stern-Elemente mit `|` (OR) getrennt werden, um auf eines dieser Elemente zu passen. Ein `!` (NOT) am Anfang einer Nicht-Stern-Gruppe passt auf jedes Label, das auf keine der Alternativen passt. Ein Quantifizierer, falls vorhanden, steht am Ende der Gruppe und bezieht sich auf eine Anzahl von Treffern für die Gruppe als Ganzes.

Ein kommentiertes Beispiel für `lquery`:

```text
Top.*{0,2}.sport*@.!football|tennis{1,}.Russ*|Spain
 a. b.      c.      d.                   e.
```

Diese Abfrage passt auf jeden Label-Pfad, der:

- `a.` mit dem Label `Top` beginnt,
- `b.` danach null bis zwei Labels enthält,
- `c.` dann ein Label mit dem Groß-/Kleinschreibung ignorierenden Präfix `sport` enthält,
- `d.` danach ein oder mehrere Labels enthält, von denen keines auf `football` oder `tennis` passt,
- `e.` und anschließend mit einem Label endet, das mit `Russ` beginnt oder exakt `Spain` ist.

- `ltxtquery`
  - Stellt ein Volltextsuche-ähnliches Muster zum Abgleich von `ltree`-Werten dar. Ein `ltxtquery`-Wert enthält Wörter, optional mit den Modifikatoren `@`, `*` und `%` am Ende. Diese Modifikatoren haben dieselbe Bedeutung wie in `lquery`. Wörter können mit `&` (AND), `|` (OR), `!` (NOT) und Klammern kombiniert werden. Der wesentliche Unterschied zu `lquery` besteht darin, dass `ltxtquery` Wörter unabhängig von ihrer Position im Label-Pfad abgleicht.

Beispiel für `ltxtquery`:

```text
Europe & Russia*@ & !Transportation
```

Das passt auf Pfade, die das Label `Europe` und ein Label enthalten, das ohne Beachtung der Groß-/Kleinschreibung mit `Russia` beginnt, aber nicht auf Pfade, die das Label `Transportation` enthalten. Die Position dieser Wörter im Pfad ist nicht wichtig. Wenn `%` verwendet wird, kann das Wort außerdem auf jedes durch Unterstriche getrennte Wort innerhalb eines Labels passen, unabhängig von seiner Position.

> **Hinweis:** `ltxtquery` erlaubt Leerraum zwischen Symbolen; `ltree` und `lquery` erlauben das nicht.

#### F.22.2. Operatoren und Funktionen

Der Typ `ltree` besitzt die üblichen Vergleichsoperatoren `=`, `<>`, `<`, `>`, `<=` und `>=`. Der Vergleich sortiert in der Reihenfolge einer Baumdurchquerung, wobei die Kinder eines Knotens nach Label-Text sortiert werden. Zusätzlich stehen die spezialisierten Operatoren aus Tabelle F.12 zur Verfügung.

**Tabelle F.12. `ltree`-Operatoren**

| Operator | Beschreibung |
| --- | --- |
| `ltree @> ltree` → `boolean` | Ist das linke Argument ein Vorfahr des rechten oder gleich diesem? |
| `ltree <@ ltree` → `boolean` | Ist das linke Argument ein Nachkomme des rechten oder gleich diesem? |
| `ltree ~ lquery` → `boolean`; `lquery ~ ltree` → `boolean` | Passt der `ltree` auf die `lquery`? |
| `ltree ? lquery[]` → `boolean`; `lquery[] ? ltree` → `boolean` | Passt der `ltree` auf irgendeine `lquery` im Array? |
| `ltree @ ltxtquery` → `boolean`; `ltxtquery @ ltree` → `boolean` | Passt der `ltree` auf die `ltxtquery`? |
| `ltree || ltree` → `ltree` | Verkettet `ltree`-Pfade. |
| `ltree || text` → `ltree`; `text || ltree` → `ltree` | Wandelt `text` nach `ltree` um und verkettet. |
| `ltree[] @> ltree` → `boolean`; `ltree <@ ltree[]` → `boolean` | Enthält das Array einen Vorfahren des `ltree`? |
| `ltree[] <@ ltree` → `boolean`; `ltree @> ltree[]` → `boolean` | Enthält das Array einen Nachkommen des `ltree`? |
| `ltree[] ~ lquery` → `boolean`; `lquery ~ ltree[]` → `boolean` | Enthält das Array einen Pfad, der auf die `lquery` passt? |
| `ltree[] ? lquery[]` → `boolean`; `lquery[] ? ltree[]` → `boolean` | Enthält das `ltree`-Array einen Pfad, der auf irgendeine `lquery` passt? |
| `ltree[] @ ltxtquery` → `boolean`; `ltxtquery @ ltree[]` → `boolean` | Enthält das Array einen Pfad, der auf die `ltxtquery` passt? |
| `ltree[] ?@> ltree` → `ltree` | Gibt den ersten Array-Eintrag zurück, der ein Vorfahr des `ltree` ist, oder `NULL`, wenn es keinen gibt. |
| `ltree[] ?<@ ltree` → `ltree` | Gibt den ersten Array-Eintrag zurück, der ein Nachkomme des `ltree` ist, oder `NULL`, wenn es keinen gibt. |
| `ltree[] ?~ lquery` → `ltree` | Gibt den ersten Array-Eintrag zurück, der auf die `lquery` passt, oder `NULL`, wenn es keinen gibt. |
| `ltree[] ?@ ltxtquery` → `ltree` | Gibt den ersten Array-Eintrag zurück, der auf die `ltxtquery` passt, oder `NULL`, wenn es keinen gibt. |

Die Operatoren `<@`, `@>`, `@` und `~` haben die Analoga `^<@`, `^@>`, `^@` und `^~`. Diese verhalten sich gleich, verwenden aber keine Indizes. Sie sind nur für Testzwecke nützlich.

Die verfügbaren Funktionen sind in Tabelle F.13 aufgeführt.

**Tabelle F.13. `ltree`-Funktionen**

| Funktion | Beschreibung | Beispiel(e) |
| --- | --- | --- |
| `subltree(ltree, start integer, end integer)` → `ltree` | Gibt den Teilpfad von Position `start` bis `end - 1` zurück, gezählt ab `0`. | `subltree('Top.Child1.Child2', 1, 2)` → `Child1` |
| `subpath(ltree, offset integer, len integer)` → `ltree` | Gibt den Teilpfad ab Position `offset` mit Länge `len` zurück. Bei negativem `offset` beginnt der Teilpfad entsprechend viele Labels vom Ende entfernt. Bei negativem `len` werden entsprechend viele Labels am Ende ausgelassen. | `subpath('Top.Child1.Child2', 0, 2)` → `Top.Child1` |
| `subpath(ltree, offset integer)` → `ltree` | Gibt den Teilpfad ab Position `offset` bis zum Ende zurück. Bei negativem `offset` beginnt der Teilpfad entsprechend viele Labels vom Ende entfernt. | `subpath('Top.Child1.Child2', 1)` → `Child1.Child2` |
| `nlevel(ltree)` → `integer` | Gibt die Anzahl der Labels im Pfad zurück. | `nlevel('Top.Child1.Child2')` → `3` |
| `index(a ltree, b ltree)` → `integer` | Gibt die Position des ersten Vorkommens von `b` in `a` zurück oder `-1`, wenn es nicht gefunden wird. | `index('0.1.2.3.5.4.5.6.8.5.6.8', '5.6')` → `6` |
| `index(a ltree, b ltree, offset integer)` → `integer` | Gibt die Position des ersten Vorkommens von `b` in `a` zurück oder `-1`, wenn es nicht gefunden wird. Die Suche beginnt bei `offset`; ein negativer `offset` bedeutet, dass `-offset` Labels vom Ende des Pfads aus gestartet wird. | `index('0.1.2.3.5.4.5.6.8.5.6.8', '5.6', -4)` → `9` |
| `text2ltree(text)` → `ltree` | Castet `text` nach `ltree`. | |
| `ltree2text(ltree)` → `text` | Castet `ltree` nach `text`. | |
| `lca(ltree [, ltree [, ...]])` → `ltree` | Berechnet den längsten gemeinsamen Vorfahren der Pfade. Es werden bis zu acht Argumente unterstützt. | `lca('1.2.3', '1.2.3.4.5.6')` → `1.2` |
| `lca(ltree[])` → `ltree` | Berechnet den längsten gemeinsamen Vorfahren der Pfade im Array. | `lca(array['1.2.3'::ltree,'1.2.3.4'])` → `1.2` |

#### F.22.3. Indizes

`ltree` unterstützt mehrere Indextypen, die die angegebenen Operatoren beschleunigen können:

- B-Tree-Index über `ltree`: `<`, `<=`, `=`, `>=`, `>`
- Hash-Index über `ltree`: `=`
- GiST-Index über `ltree` mit der Operatorklasse `gist_ltree_ops`: `<`, `<=`, `=`, `>=`, `>`, `@>`, `<@`, `@`, `~`, `?`

Die GiST-Operatorklasse `gist_ltree_ops` approximiert eine Menge von Pfad-Labels als Bitmap-Signatur. Ihr optionaler ganzzahliger Parameter `siglen` bestimmt die Signaturlänge in Byte. Die Standardsignaturlänge beträgt 8 Byte. Die Länge muss ein positives Vielfaches der `int`-Ausrichtung sein, auf den meisten Maschinen also 4 Byte, und darf höchstens 2024 Byte betragen. Längere Signaturen führen zu einer präziseren Suche, bei der ein kleinerer Anteil des Index und weniger Heap-Seiten gelesen werden, allerdings auf Kosten eines größeren Index.

Beispiel für einen solchen Index mit der Standardsignaturlänge von 8 Byte:

```sql
CREATE INDEX path_gist_idx ON test USING GIST (path);
```

Beispiel für einen solchen Index mit einer Signaturlänge von 100 Byte:

```sql
CREATE INDEX path_gist_idx ON test USING GIST (path
  gist_ltree_ops(siglen=100));
```

- GiST-Index über `ltree[]` mit der Operatorklasse `gist__ltree_ops`: `ltree[] <@ ltree`, `ltree @> ltree[]`, `@`, `~`, `?`

Die GiST-Operatorklasse `gist__ltree_ops` funktioniert ähnlich wie `gist_ltree_ops` und akzeptiert ebenfalls die Signaturlänge als Parameter. Der Standardwert von `siglen` in `gist__ltree_ops` beträgt 28 Byte.

Beispiel für einen solchen Index mit der Standardsignaturlänge von 28 Byte:

```sql
CREATE INDEX path_gist_idx ON test USING GIST (array_path);
```

Beispiel für einen solchen Index mit einer Signaturlänge von 100 Byte:

```sql
CREATE INDEX path_gist_idx ON test USING GIST (array_path
  gist__ltree_ops(siglen=100));
```

> **Hinweis:** Dieser Indextyp ist verlustbehaftet.

#### F.22.4. Beispiel

Dieses Beispiel verwendet die folgenden Daten, die auch in der Datei `contrib/ltree/ltreetest.sql` der Quellcode-Distribution verfügbar sind:

```sql
CREATE TABLE test (path ltree);
INSERT INTO test VALUES ('Top');
INSERT INTO test VALUES ('Top.Science');
INSERT INTO test VALUES ('Top.Science.Astronomy');
INSERT INTO test VALUES ('Top.Science.Astronomy.Astrophysics');
INSERT INTO test VALUES ('Top.Science.Astronomy.Cosmology');
INSERT INTO test VALUES ('Top.Hobbies');
INSERT INTO test VALUES ('Top.Hobbies.Amateurs_Astronomy');
INSERT INTO test VALUES ('Top.Collections');
INSERT INTO test VALUES ('Top.Collections.Pictures');
INSERT INTO test VALUES ('Top.Collections.Pictures.Astronomy');
INSERT INTO test VALUES ('Top.Collections.Pictures.Astronomy.Stars');
INSERT INTO test VALUES ('Top.Collections.Pictures.Astronomy.Galaxies');
INSERT INTO test VALUES ('Top.Collections.Pictures.Astronomy.Astronauts');

CREATE INDEX path_gist_idx ON test USING GIST (path);
CREATE INDEX path_idx ON test USING BTREE (path);
CREATE INDEX path_hash_idx ON test USING HASH (path);
```

Damit enthält die Tabelle `test` Daten, die die folgende Hierarchie beschreiben:

```text
Top
/   | \
Science Hobbies Collections
/      |               \
Astronomy   Amateurs_Astronomy Pictures
/ \                             |
Astrophysics Cosmology          Astronomy
                                / | \
                         Galaxies Stars Astronauts
```

Vererbung kann so abgefragt werden:

```sql
SELECT path FROM test WHERE path <@ 'Top.Science';
```

```text
path
------------------------------------
Top.Science
Top.Science.Astronomy
Top.Science.Astronomy.Astrophysics
Top.Science.Astronomy.Cosmology
(4 rows)
```

Beispiele für Pfadabgleich:

```sql
SELECT path FROM test WHERE path ~ '*.Astronomy.*';
```

```text
path
-----------------------------------------------
Top.Science.Astronomy
Top.Science.Astronomy.Astrophysics
Top.Science.Astronomy.Cosmology
Top.Collections.Pictures.Astronomy
Top.Collections.Pictures.Astronomy.Stars
Top.Collections.Pictures.Astronomy.Galaxies
Top.Collections.Pictures.Astronomy.Astronauts
(7 rows)
```

```sql
SELECT path FROM test WHERE path ~ '*.!pictures@.Astronomy.*';
```

```text
path
------------------------------------
Top.Science.Astronomy
Top.Science.Astronomy.Astrophysics
Top.Science.Astronomy.Cosmology
(3 rows)
```

Beispiele für Volltextsuche:

```sql
SELECT path FROM test WHERE path @ 'Astro*% & !pictures@';
```

```text
path
------------------------------------
Top.Science.Astronomy
Top.Science.Astronomy.Astrophysics
Top.Science.Astronomy.Cosmology
Top.Hobbies.Amateurs_Astronomy
(4 rows)
```

```sql
SELECT path FROM test WHERE path @ 'Astro* & !pictures@';
```

```text
path
------------------------------------
Top.Science.Astronomy
Top.Science.Astronomy.Astrophysics
Top.Science.Astronomy.Cosmology
(3 rows)
```

Pfadkonstruktion mit Funktionen:

```sql
SELECT subpath(path,0,2)||'Space'||subpath(path,2) FROM
  test WHERE path <@ 'Top.Science.Astronomy';
```

```text
?column?
------------------------------------------
Top.Science.Space.Astronomy
Top.Science.Space.Astronomy.Astrophysics
Top.Science.Space.Astronomy.Cosmology
(3 rows)
```

Dies kann durch eine SQL-Funktion vereinfacht werden, die ein Label an einer angegebenen Position in einen Pfad einfügt:

```sql
CREATE FUNCTION ins_label(ltree, int, text) RETURNS ltree
AS 'select subpath($1,0,$2) || $3 || subpath($1,$2);'
LANGUAGE SQL IMMUTABLE;

SELECT ins_label(path,2,'Space') FROM test WHERE path
  <@ 'Top.Science.Astronomy';
```

```text
ins_label
------------------------------------------
Top.Science.Space.Astronomy
Top.Science.Space.Astronomy.Astrophysics
Top.Science.Space.Astronomy.Cosmology
(3 rows)
```

#### F.22.5. Transformationen

Die Erweiterung `ltree_plpython3u` implementiert Transformationen für den Typ `ltree` für PL/Python. Wenn sie installiert und beim Erstellen einer Funktion angegeben wird, werden `ltree`-Werte auf Python-Listen abgebildet. Die umgekehrte Richtung wird derzeit jedoch nicht unterstützt.

#### F.22.6. Autoren

Die gesamte Arbeit wurde von Teodor Sigaev (<teodor@stack.net>) und Oleg Bartunov (<oleg@sai.msu.su>) durchgeführt. Weitere Informationen finden Sie unter <http://www.sai.msu.su/~megera/postgres/gist/>. Die Autoren danken Eugeny Rodichev für hilfreiche Diskussionen. Kommentare und Fehlerberichte sind willkommen.


### F.23. pageinspect — Low-Level-Inspektion von Datenbankseiten

Das Modul `pageinspect` stellt Funktionen bereit, mit denen Sie den Inhalt von Datenbankseiten auf niedriger Ebene untersuchen können. Das ist vor allem für Debugging-Zwecke nützlich. Alle diese Funktionen dürfen nur von Superusern verwendet werden.

#### F.23.1. Allgemeine Funktionen

```text
get_raw_page(relname text, fork text, blkno bigint) returns bytea
```

`get_raw_page` liest den angegebenen Block der benannten Relation und gibt eine Kopie als `bytea`-Wert zurück. Dadurch kann eine einzelne zeitkonsistente Kopie des Blocks erhalten werden. `fork` sollte `main` für den Hauptdaten-Fork, `fsm` für die Free Space Map, `vm` für die Visibility Map oder `init` für den Initialisierungs-Fork sein.

```text
get_raw_page(relname text, blkno bigint) returns bytea
```

Dies ist eine Kurzform von `get_raw_page` zum Lesen aus dem Haupt-Fork. Sie entspricht `get_raw_page(relname, 'main', blkno)`.

```text
page_header(page bytea) returns record
```

`page_header` zeigt Felder, die allen PostgreSQL-Heap- und Indexseiten gemeinsam sind.

Als Argument wird ein mit `get_raw_page` erhaltenes Seitenabbild übergeben. Beispiel:

```sql
SELECT * FROM page_header(get_raw_page('pg_class', 0));
```

```text
    lsn    | checksum | flags | lower | upper | special | pagesize | version | prune_xid
-----------+----------+-------+-------+-------+---------+----------+---------+-----------
 0/24A1B50 |        0 |     1 |   232 |   368 |    8192 |     8192 |       4 |         0
```

Die zurückgegebenen Spalten entsprechen den Feldern der Struktur `PageHeaderData`. Einzelheiten finden Sie in `src/include/storage/bufpage.h`.

Das Feld `checksum` ist die auf der Seite gespeicherte Prüfsumme; sie kann falsch sein, wenn die Seite auf irgendeine Weise beschädigt ist. Wenn Datenprüfsummen für diese Instanz deaktiviert sind, ist der gespeicherte Wert bedeutungslos.

```text
page_checksum(page bytea, blkno bigint) returns smallint
```

`page_checksum` berechnet die Prüfsumme für die Seite so, als läge sie am angegebenen Block.

Als Argument wird ein mit `get_raw_page` erhaltenes Seitenabbild übergeben. Beispiel:

```sql
SELECT page_checksum(get_raw_page('pg_class', 0), 0);
```

```text
 page_checksum
---------------
```

Beachten Sie, dass die Prüfsumme von der Blocknummer abhängt. Deshalb sollten passende Blocknummern übergeben werden, außer bei sehr speziellen Debugging-Aufgaben.

Die mit dieser Funktion berechnete Prüfsumme kann mit dem Ergebnisfeld `checksum` der Funktion `page_header` verglichen werden. Wenn Datenprüfsummen für diese Instanz aktiviert sind, sollten die beiden Werte gleich sein.

```text
fsm_page_contents(page bytea) returns text
```

`fsm_page_contents` zeigt die interne Knotenstruktur einer FSM-Seite. Beispiel:

```sql
SELECT fsm_page_contents(get_raw_page('pg_class', 'fsm', 0));
```

Die Ausgabe ist eine mehrzeilige Zeichenkette mit einer Zeile pro Knoten im binären Baum innerhalb der Seite. Nur Knoten, die nicht null sind, werden ausgegeben. Außerdem wird der sogenannte „next“-Zeiger ausgegeben, der auf den nächsten Slot zeigt, der von der Seite zurückgegeben wird.

Weitere Informationen zur Struktur einer FSM-Seite finden Sie in `src/backend/storage/freespace/README`.

#### F.23.2. Heap-Funktionen

```text
heap_page_items(page bytea) returns setof record
```

`heap_page_items` zeigt alle Line Pointer auf einer Heap-Seite. Für Line Pointer, die in Verwendung sind, werden außerdem Tupel-Header und Rohdaten des Tupels angezeigt. Alle Tupel werden angezeigt, unabhängig davon, ob sie zum Zeitpunkt des Kopierens der Rohseite für einen MVCC-Snapshot sichtbar waren.

Als Argument wird ein mit `get_raw_page` erhaltenes Heap-Seitenabbild übergeben. Beispiel:

```sql
SELECT * FROM heap_page_items(get_raw_page('pg_class', 0));
```

Erläuterungen zu den zurückgegebenen Feldern finden Sie in `src/include/storage/itemid.h` und `src/include/access/htup_details.h`.

Die Funktion `heap_tuple_infomask_flags` kann verwendet werden, um die Flag-Bits von `t_infomask` und `t_infomask2` für Heap-Tupel zu entpacken.

```text
tuple_data_split(rel_oid oid, t_data bytea, t_infomask integer,
                 t_infomask2 integer, t_bits text [, do_detoast bool]) returns bytea[]
```

`tuple_data_split` zerlegt Tupeldaten auf dieselbe Weise in Attribute wie die Backend-Interna.

```sql
SELECT tuple_data_split('pg_class'::regclass,
                        t_data, t_infomask, t_infomask2, t_bits)
FROM heap_page_items(get_raw_page('pg_class', 0));
```

Diese Funktion sollte mit denselben Argumenten aufgerufen werden wie die Rückgabeattribute von `heap_page_items`.

Wenn `do_detoast` `true` ist, werden Attribute bei Bedarf detoastet. Der Standardwert ist `false`.

```text
heap_page_item_attrs(page bytea, rel_oid regclass [, do_detoast bool]) returns setof record
```

`heap_page_item_attrs` entspricht `heap_page_items`, gibt die Rohdaten der Tupel aber als Array von Attributen zurück, die optional durch `do_detoast` detoastet werden können. `do_detoast` ist standardmäßig `false`.

Als Argument wird ein mit `get_raw_page` erhaltenes Heap-Seitenabbild übergeben. Beispiel:

```sql
SELECT *
FROM heap_page_item_attrs(get_raw_page('pg_class', 0),
                          'pg_class'::regclass);
```

```text
heap_tuple_infomask_flags(t_infomask integer, t_infomask2 integer) returns record
```

`heap_tuple_infomask_flags` dekodiert die von `heap_page_items` zurückgegebenen Werte `t_infomask` und `t_infomask2` in eine menschenlesbare Menge von Arrays mit Flag-Namen. Eine Spalte enthält alle Flags, eine weitere kombinierte Flags. Beispiel:

```sql
SELECT t_ctid, raw_flags, combined_flags
FROM heap_page_items(get_raw_page('pg_class', 0)),
LATERAL heap_tuple_infomask_flags(t_infomask,
                                  t_infomask2)
WHERE t_infomask IS NOT NULL OR t_infomask2 IS NOT NULL;
```

Diese Funktion sollte mit denselben Argumenten aufgerufen werden wie die Rückgabeattribute von `heap_page_items`.

Kombinierte Flags werden für Makros auf Quellcodeebene angezeigt, die mehr als ein Rohbit berücksichtigen, zum Beispiel `HEAP_XMIN_FROZEN`.

Erläuterungen zu den zurückgegebenen Flag-Namen finden Sie in `src/include/access/htup_details.h`.

#### F.23.3. B-Tree-Funktionen

```text
bt_metap(relname text) returns record
```

`bt_metap` gibt Informationen über die Metaseite eines B-Tree-Index zurück. Beispiel:

```sql
SELECT * FROM bt_metap('pg_cast_oid_index');
```

```text
-[ RECORD 1 ]-------------+-------
magic                     | 340322
version                   | 4
root                      | 1
level                     | 0
fastroot                  | 1
fastlevel                 | 0
last_cleanup_num_delpages | 0
last_cleanup_num_tuples   | 230
allequalimage             | f
```

```text
bt_page_stats(relname text, blkno bigint) returns record
```

`bt_page_stats` gibt zusammenfassende Informationen über eine Datenseite eines B-Tree-Index zurück. Beispiel:

```sql
SELECT * FROM bt_page_stats('pg_cast_oid_index', 1);
```

```text
-[ RECORD 1 ]-+-----
blkno         | 1
type          | l
live_items    | 224
dead_items    | 0
avg_item_size | 16
page_size     | 8192
free_size     | 3668
btpo_prev     | 0
btpo_next     | 0
btpo_level    | 0
btpo_flags    | 3
```

```text
bt_multi_page_stats(relname text, blkno bigint, blk_count bigint) returns setof record
```

`bt_multi_page_stats` gibt dieselben Informationen wie `bt_page_stats` zurück, jedoch für jede Seite im Bereich, der bei `blkno` beginnt und sich über `blk_count` Seiten erstreckt. Wenn `blk_count` negativ ist, werden alle Seiten von `blkno` bis zum Ende des Index gemeldet. Beispiel:

```sql
SELECT * FROM bt_multi_page_stats('pg_proc_oid_index', 5, 2);
```

```text
-[ RECORD 1 ]-+-----
blkno         | 5
type          | l
live_items    | 367
...
-[ RECORD 2 ]-+-----
blkno         | 6
type          | l
live_items    | 367
...
```

```text
bt_page_items(relname text, blkno bigint) returns setof record
```

`bt_page_items` gibt detaillierte Informationen über alle Items auf einer B-Tree-Indexseite zurück. Beispiel:

```sql
SELECT itemoffset, ctid, itemlen, nulls, vars, data,
       dead, htid, tids[0:2] AS some_tids
FROM bt_page_items('tenk2_hundred', 5);
```

```text
itemoffset |   ctid    | itemlen | nulls | vars | data        | dead | htid  | some_tids
-----------+-----------+---------+-------+------+-------------+------+-------+-----------------------
1          | (16,1)    |      16 | f     | f    | 30 00 ...   |      |       |
2          | (16,8292) |     616 | f     | f    | 24 00 ...   | f    | (1,6) | {"(1,6)","(10,22)"}
...
(13 rows)
```

Dies ist eine B-Tree-Blattseite. Alle Tupel, die auf die Tabelle zeigen, sind hier Posting-List-Tupel. Außerdem gibt es ein „High Key“-Tupel bei `itemoffset` 1. In diesem Beispiel wird `ctid` verwendet, um kodierte Informationen zu jedem Tupel zu speichern; Blattseitentupel speichern häufig stattdessen direkt eine Heap-TID im Feld `ctid`. `tids` ist die Liste der als Posting List gespeicherten TIDs.

Auf einer internen Seite ist der Blocknummernteil von `ctid` ein Downlink, also eine Blocknummer einer anderen Seite im Index. Der Offset-Teil, die zweite Zahl, speichert kodierte Informationen über das Tupel, etwa die Anzahl vorhandener Spalten. Abgeschnittene Spalten werden so behandelt, als hätten sie den Wert „minus infinity“.

`htid` zeigt eine Heap-TID für das Tupel, unabhängig von der zugrunde liegenden Tupeldarstellung. Dieser Wert kann mit `ctid` übereinstimmen oder aus alternativen Darstellungen dekodiert werden, die von Posting-List-Tupeln und Tupeln interner Seiten verwendet werden. Bei Tupeln interner Seiten ist die Heap-TID-Spalte auf Implementierungsebene normalerweise abgeschnitten; dies wird als `NULL`-Wert in `htid` dargestellt.

Beachten Sie, dass das erste Item auf jeder nicht ganz rechten Seite, also auf jeder Seite mit einem von null verschiedenen Wert in `btpo_next`, der „High Key“ der Seite ist. Seine Daten dienen als obere Grenze für alle Items auf der Seite, während sein `ctid`-Feld nicht auf einen anderen Block zeigt. Auf internen Seiten sind beim ersten echten Dateneintrag, also dem ersten Eintrag, der kein High Key ist, zuverlässig alle Spalten abgeschnitten; sein Datenfeld enthält daher keinen tatsächlichen Wert. Ein solches Item besitzt aber einen gültigen Downlink im Feld `ctid`.

Weitere Einzelheiten zur Struktur von B-Tree-Indizes finden Sie in Abschnitt 65.1.4.1. Weitere Einzelheiten zu Deduplizierung und Posting Lists finden Sie in Abschnitt 65.1.4.3.

```text
bt_page_items(page bytea) returns setof record
```

Es ist auch möglich, `bt_page_items` eine Seite als `bytea`-Wert zu übergeben. Als Argument sollte ein mit `get_raw_page` erhaltenes Seitenabbild verwendet werden. Das letzte Beispiel könnte also auch so geschrieben werden:

```sql
SELECT itemoffset, ctid, itemlen, nulls, vars, data,
       dead, htid, tids[0:2] AS some_tids
FROM bt_page_items(get_raw_page('tenk2_hundred', 5));
```

Alle weiteren Details entsprechen der vorherigen Variante.

#### F.23.4. BRIN-Funktionen

```text
brin_page_type(page bytea) returns text
```

`brin_page_type` gibt den Seitentyp der angegebenen BRIN-Indexseite zurück oder löst einen Fehler aus, wenn die Seite keine gültige BRIN-Seite ist. Beispiel:

```sql
SELECT brin_page_type(get_raw_page('brinidx', 0));
```

```text
 brin_page_type
----------------
 meta
```

```text
brin_metapage_info(page bytea) returns record
```

`brin_metapage_info` gibt verschiedene Informationen über eine BRIN-Index-Metaseite zurück. Beispiel:

```sql
SELECT * FROM brin_metapage_info(get_raw_page('brinidx', 0));
```

```text
   magic    | version | pagesperrange | lastrevmappage
------------+---------+---------------+----------------
 0xA8109CFA |       1 |             4 |              2
```

```text
brin_revmap_data(page bytea) returns setof tid
```

`brin_revmap_data` gibt die Liste der Tupel-IDs in einer BRIN-Range-Map-Seite zurück. Beispiel:

```sql
SELECT * FROM brin_revmap_data(get_raw_page('brinidx', 2)) LIMIT 5;
```

```text
 pages
---------
 (6,137)
 (6,138)
 (6,139)
 (6,140)
 (6,141)
```

```text
brin_page_items(page bytea, index oid) returns setof record
```

`brin_page_items` gibt die auf der BRIN-Datenseite gespeicherten Daten zurück. Beispiel:

```sql
SELECT * FROM brin_page_items(get_raw_page('brinidx', 5), 'brinidx')
ORDER BY blknum, attnum LIMIT 6;
```

```text
itemoffset | blknum | attnum | allnulls | hasnulls | placeholder | empty | value
-----------+--------+--------+----------+----------+-------------+-------+-----------
137        |      0 |      1 | t        | f        | f           | f     |
137        |      0 |      2 | f        | f        | f           | f     | {1 .. 88}
...
```

Die zurückgegebenen Spalten entsprechen den Feldern in den Strukturen `BrinMemTuple` und `BrinValues`. Einzelheiten finden Sie in `src/include/access/brin_tuple.h`.

#### F.23.5. GIN-Funktionen

```text
gin_metapage_info(page bytea) returns record
```

`gin_metapage_info` gibt Informationen über eine GIN-Index-Metaseite zurück. Beispiel:

```sql
SELECT * FROM gin_metapage_info(get_raw_page('gin_index', 0));
```

```text
-[ RECORD 1 ]----+-----------
pending_head     | 4294967295
pending_tail     | 4294967295
tail_free_size   | 0
n_pending_pages  | 0
n_pending_tuples | 0
n_total_pages    | 7
n_entry_pages    | 6
n_data_pages     | 0
n_entries        | 693
version          | 2
```

```text
gin_page_opaque_info(page bytea) returns record
```

`gin_page_opaque_info` gibt Informationen über den Opaque-Bereich einer GIN-Indexseite zurück, etwa den Seitentyp. Beispiel:

```sql
SELECT * FROM gin_page_opaque_info(get_raw_page('gin_index', 2));
```

```text
 rightlink | maxoff |         flags
-----------+--------+------------------------
         5 |      0 | {data,leaf,compressed}
(1 row)
```

```text
gin_leafpage_items(page bytea) returns setof record
```

`gin_leafpage_items` gibt Informationen über die auf einer komprimierten GIN-Blattseite gespeicherten Daten zurück. Beispiel:

```sql
SELECT first_tid, nbytes, tids[0:5] AS some_tids
FROM gin_leafpage_items(get_raw_page('gin_test_idx', 2));
```

```text
 first_tid | nbytes |                         some_tids
-----------+--------+----------------------------------------------------------
 (8,41)    |    244 | {"(8,41)","(8,43)","(8,44)","(8,45)","(8,46)"}
 (10,45)   |    248 | {"(10,45)","(10,46)","(10,47)","(10,48)","(10,49)"}
 ...
(7 rows)
```

#### F.23.6. GiST-Funktionen

```text
gist_page_opaque_info(page bytea) returns record
```

`gist_page_opaque_info` gibt Informationen aus dem Opaque-Bereich einer GiST-Indexseite zurück, etwa NSN, Rightlink und Seitentyp. Beispiel:

```sql
SELECT * FROM gist_page_opaque_info(get_raw_page('test_gist_idx', 2));
```

```text
 lsn | nsn | rightlink | flags
-----+-----+-----------+--------
 0/1 | 0/0 |         1 | {leaf}
(1 row)
```

```text
gist_page_items(page bytea, index_oid regclass) returns setof record
```

`gist_page_items` gibt Informationen über die auf einer Seite eines GiST-Index gespeicherten Daten zurück. Beispiel:

```sql
SELECT * FROM gist_page_items(get_raw_page('test_gist_idx', 0), 'test_gist_idx');
```

```text
 itemoffset |   ctid    | itemlen | dead |              keys
------------+-----------+---------+------+-------------------------------
          1 | (1,65535) |      40 | f    | (p)=("(185,185),(1,1)")
          2 | (2,65535) |      40 | f    | (p)=("(370,370),(186,186)")
 ...
(6 rows)
```

```text
gist_page_items_bytea(page bytea) returns setof record
```

`gist_page_items_bytea` entspricht `gist_page_items`, gibt die Schlüsseldaten aber als rohes `bytea`-Blob zurück. Da nicht versucht wird, den Schlüssel zu dekodieren, muss die Funktion nicht wissen, welcher Index betroffen ist. Beispiel:

```sql
SELECT * FROM gist_page_items_bytea(get_raw_page('test_gist_idx', 0));
```

```text
 itemoffset |   ctid    | itemlen | dead | key_data
------------+-----------+---------+------+-----------------------------------------
          1 | (1,65535) |      40 | f    | \x00000100ffff28000000000000c06440...
          2 | (2,65535) |      40 | f    | \x00000200ffff28000000000000c07440...
 ...
(7 rows)
```

#### F.23.7. Hash-Funktionen

```text
hash_page_type(page bytea) returns text
```

`hash_page_type` gibt den Seitentyp der angegebenen Hash-Indexseite zurück. Beispiel:

```sql
SELECT hash_page_type(get_raw_page('con_hash_index', 0));
```

```text
 hash_page_type
----------------
 metapage
```

```text
hash_page_stats(page bytea) returns setof record
```

`hash_page_stats` gibt Informationen über eine Bucket- oder Overflow-Seite eines Hash-Index zurück. Beispiel:

```sql
SELECT * FROM hash_page_stats(get_raw_page('con_hash_index', 1));
```

```text
-[ RECORD 1 ]---+-----------
live_items      | 407
dead_items      | 0
page_size       | 8192
free_size       | 8
hasho_prevblkno | 4096
hasho_nextblkno | 8474
hasho_bucket    | 0
hasho_flag      | 66
hasho_page_id   | 65408
```

```text
hash_page_items(page bytea) returns setof record
```

`hash_page_items` gibt Informationen über die auf einer Bucket- oder Overflow-Seite eines Hash-Index gespeicherten Daten zurück. Beispiel:

```sql
SELECT * FROM hash_page_items(get_raw_page('con_hash_index', 1)) LIMIT 5;
```

```text
 itemoffset |   ctid    |    data
------------+-----------+------------
          1 | (899,77) | 1053474816
          2 | (897,29) | 1053474816
          3 | (894,207) | 1053474816
          4 | (892,159) | 1053474816
          5 | (890,111) | 1053474816
```

```text
hash_bitmap_info(index oid, blkno bigint) returns record
```

`hash_bitmap_info` zeigt den Status eines Bits auf der Bitmap-Seite für eine bestimmte Overflow-Seite eines Hash-Index. Beispiel:

```sql
SELECT * FROM hash_bitmap_info('con_hash_index', 2052);
```

```text
 bitmapblkno | bitmapbit | bitstatus
-------------+-----------+-----------
          65 |         3 | t
```

```text
hash_metapage_info(page bytea) returns record
```

`hash_metapage_info` gibt Informationen zurück, die in der Metaseite eines Hash-Index gespeichert sind. Beispiel:

```sql
SELECT magic, version, ntuples, ffactor, bsize, bmsize,
       bmshift, maxbucket, highmask, lowmask, ovflpoint,
       firstfree, nmaps, procid,
       regexp_replace(spares::text, '(,0)*}', '}') AS spares,
       regexp_replace(mapp::text, '(,0)*}', '}') AS mapp
FROM hash_metapage_info(get_raw_page('con_hash_index', 0));
```

```text
-[ RECORD 1 ]-----------------------------------------------
magic     | 105121344
version   | 4
ntuples   | 500500
ffactor   | 40
bsize     | 8152
bmsize    | 4096
bmshift   | 15
maxbucket | 12512
...
spares    | {0,0,0,0,0,0,1,1,...,1204}
mapp      | {65}
```


### F.24. passwordcheck — Passwortstärke prüfen

Das Modul `passwordcheck` prüft Benutzerpasswörter immer dann, wenn sie mit `CREATE ROLE` oder `ALTER ROLE` gesetzt werden. Wird ein Passwort als zu schwach eingestuft, wird es zurückgewiesen und der Befehl endet mit einem Fehler.

Um dieses Modul zu aktivieren, fügen Sie `'$libdir/passwordcheck'` zu `shared_preload_libraries` in `postgresql.conf` hinzu und starten Sie den Server neu.

Sie können das Modul an Ihre Anforderungen anpassen, indem Sie den Quellcode ändern. Beispielsweise können Sie CrackLib2 zur Passwortprüfung verwenden; dazu müssen nur zwei Zeilen im Makefile einkommentiert und das Modul neu gebaut werden. Aus Lizenzgründen kann CrackLib nicht standardmäßig eingebunden werden. Ohne CrackLib erzwingt das Modul einige einfache Regeln für Passwortstärke, die Sie nach Bedarf ändern oder erweitern können.

> **Achtung:**
> Um zu verhindern, dass unverschlüsselte Passwörter über das Netzwerk übertragen, in das Server-Log geschrieben oder anderweitig von einem Datenbankadministrator abgegriffen werden, erlaubt PostgreSQL die Übergabe bereits verschlüsselter Passwörter. Viele Client-Programme nutzen diese Möglichkeit und verschlüsseln das Passwort, bevor sie es an den Server senden.

Das schränkt den Nutzen von `passwordcheck` ein, weil das Modul in diesem Fall das Passwort nur zu erraten versuchen kann. Daher wird `passwordcheck` bei hohen Sicherheitsanforderungen nicht empfohlen. Sicherer ist es, eine externe Authentifizierungsmethode wie GSSAPI zu verwenden (siehe [Kapitel 20](20_Client_Authentifizierung.md)), statt sich auf Passwörter innerhalb der Datenbank zu verlassen.

Alternativ könnten Sie `passwordcheck` so ändern, dass bereits verschlüsselte Passwörter abgelehnt werden. Benutzer dazu zu zwingen, Passwörter im Klartext zu setzen, bringt jedoch eigene Sicherheitsrisiken mit sich.

#### F.24.1. Konfigurationsparameter

`passwordcheck.min_password_length` (`integer`)

Die minimale zulässige Passwortlänge in Byte. Der Standardwert ist `8`. Nur Superuser können diese Einstellung ändern.

> **Hinweis:**
> Dieser Parameter hat keine Wirkung, wenn ein Benutzer ein bereits verschlüsseltes Passwort übergibt.

Üblicherweise wird dieser Parameter in `postgresql.conf` gesetzt, Superuser können ihn aber auch innerhalb ihrer eigenen Sitzungen zur Laufzeit ändern. Eine typische Verwendung wäre:

```text
# postgresql.conf
passwordcheck.min_password_length = 12
```

CrackLib: <https://github.com/cracklib/cracklib>


### F.25. pg_buffercache — Zustand des PostgreSQL-Puffercaches untersuchen

Das Modul `pg_buffercache` ermöglicht es, den Inhalt des Shared Buffer Cache in Echtzeit zu untersuchen. Außerdem bietet es zu Testzwecken eine Low-Level-Möglichkeit, Daten daraus zu verdrängen.

Das Modul stellt folgende Funktionen und Sichten bereit:

- `pg_buffercache_pages()`, gekapselt durch die Sicht `pg_buffercache`
- `pg_buffercache_numa_pages()`, gekapselt durch die Sicht `pg_buffercache_numa`
- `pg_buffercache_summary()`
- `pg_buffercache_usage_counts()`
- `pg_buffercache_evict()`
- `pg_buffercache_evict_relation()`
- `pg_buffercache_evict_all()`

`pg_buffercache_pages()` gibt eine Menge von Datensätzen zurück; jede Zeile beschreibt den Zustand eines Shared-Buffer-Eintrags. Die Sicht `pg_buffercache` kapselt diese Funktion für eine bequemere Verwendung.

`pg_buffercache_numa_pages()` liefert NUMA-Node-Zuordnungen für Shared-Buffer-Einträge. Diese Information ist nicht Teil von `pg_buffercache_pages()`, weil ihre Ermittlung deutlich langsamer ist. Die Sicht `pg_buffercache_numa` kapselt diese Funktion für eine bequemere Verwendung.

`pg_buffercache_summary()` gibt eine einzelne Zeile zurück, die den Zustand des Shared Buffer Cache zusammenfasst.

`pg_buffercache_usage_counts()` gibt eine Menge von Datensätzen zurück; jede Zeile beschreibt die Anzahl der Puffer mit einem bestimmten Usage Count.

Standardmäßig ist die Verwendung dieser Funktionen auf Superuser und Rollen mit den Rechten der Rolle `pg_monitor` beschränkt. Zugriff kann anderen Rollen mit `GRANT` erteilt werden.

`pg_buffercache_evict()` erlaubt es, anhand einer Buffer-ID einen Block aus dem Buffer Pool zu verdrängen. Diese Funktion darf nur von Superusern verwendet werden.

`pg_buffercache_evict_relation()` erlaubt es, anhand einer Relationskennung alle nicht gepinnten Shared Buffers der Relation aus dem Buffer Pool zu verdrängen. Diese Funktion darf nur von Superusern verwendet werden.

`pg_buffercache_evict_all()` erlaubt es, alle nicht gepinnten Shared Buffers im Buffer Pool zu verdrängen. Diese Funktion darf nur von Superusern verwendet werden.

#### F.25.1. Die Sicht `pg_buffercache`

Die Spalten der Sicht sind in Tabelle F.14 beschrieben.

**Tabelle F.14. Spalten von `pg_buffercache`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `bufferid` | `integer` | ID im Bereich `1..shared_buffers` |
| `relfilenode` | `oid` (referenziert `pg_class.relfilenode`) | Filenode-Nummer der Relation |
| `reltablespace` | `oid` (referenziert `pg_tablespace.oid`) | Tablespace-OID der Relation |
| `reldatabase` | `oid` (referenziert `pg_database.oid`) | Datenbank-OID der Relation |
| `relforknumber` | `smallint` | Fork-Nummer innerhalb der Relation; siehe `common/relpath.h` |
| `relblocknumber` | `bigint` | Seitennummer innerhalb der Relation |
| `isdirty` | `boolean` | Gibt an, ob die Seite dirty ist |
| `usagecount` | `smallint` | Clock-Sweep-Zugriffszähler |
| `pinning_backends` | `integer` | Anzahl der Backends, die diesen Buffer pinnen |

Für jeden Buffer im Shared Cache gibt es eine Zeile. Unbenutzte Buffer werden mit `NULL` in allen Feldern außer `bufferid` angezeigt. Geteilte Systemkataloge werden als zur Datenbank `0` gehörend angezeigt.

Da der Cache von allen Datenbanken gemeinsam verwendet wird, enthält er normalerweise Seiten aus Relationen, die nicht zur aktuellen Datenbank gehören. Dadurch gibt es für manche Zeilen möglicherweise keine passende Join-Zeile in `pg_class`, oder es können sogar falsche Joins entstehen. Wenn Sie gegen `pg_class` joinen, sollten Sie den Join auf Zeilen einschränken, deren `reldatabase` der OID der aktuellen Datenbank oder `0` entspricht.

Beim Kopieren der Buffer-Zustandsdaten, die die Sicht anzeigt, werden keine Buffer-Manager-Sperren genommen. Der Zugriff auf `pg_buffercache` beeinträchtigt normale Buffer-Aktivität daher weniger, liefert aber keinen über alle Buffer hinweg konsistenten Ergebnissatz. Die Informationen jedes einzelnen Buffers sind jedoch in sich konsistent.

#### F.25.2. Die Sicht `pg_buffercache_numa`

Die Spalten der Sicht sind in Tabelle F.15 beschrieben.

**Tabelle F.15. Spalten von `pg_buffercache_numa`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `bufferid` | `integer` | ID im Bereich `1..shared_buffers` |
| `os_page_num` | `bigint` | Nummer der Betriebssystem-Speicherseite für diesen Buffer |
| `numa_node` | `int` | ID des NUMA-Nodes |

Da die NUMA-Node-Abfrage für jede Seite erfordert, dass Speicherseiten eingelagert werden, kann die erste Ausführung dieser Funktion spürbar lange dauern. Auch danach ist das Abrufen dieser Information teuer; häufige Abfragen der Sicht werden nicht empfohlen.

> **Achtung:**
> Beim Bestimmen des NUMA-Nodes berührt die Sicht alle Speicherseiten des Shared-Memory-Segments. Dadurch wird Shared Memory zwangsweise alloziert, falls dies noch nicht geschehen ist, und der Speicher kann abhängig von der Systemkonfiguration vollständig auf einem einzigen NUMA-Node alloziert werden.

#### F.25.3. Die Funktion `pg_buffercache_summary()`

Die von der Funktion ausgegebenen Spalten sind in Tabelle F.16 beschrieben.

**Tabelle F.16. Ausgabespalten von `pg_buffercache_summary()`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `buffers_used` | `int4` | Anzahl der verwendeten Shared Buffers |
| `buffers_unused` | `int4` | Anzahl der unbenutzten Shared Buffers |
| `buffers_dirty` | `int4` | Anzahl der dirty Shared Buffers |
| `buffers_pinned` | `int4` | Anzahl der gepinnten Shared Buffers |
| `usagecount_avg` | `float8` | Durchschnittlicher Usage Count der verwendeten Shared Buffers |

`pg_buffercache_summary()` gibt eine einzelne Zeile zurück, die den Zustand aller Shared Buffers zusammenfasst. Ähnliche und detailliertere Informationen liefert die Sicht `pg_buffercache`, aber `pg_buffercache_summary()` ist deutlich günstiger.

Wie die Sicht `pg_buffercache` nimmt auch `pg_buffercache_summary()` keine Buffer-Manager-Sperren. Gleichzeitige Aktivität kann daher zu kleinen Ungenauigkeiten im Ergebnis führen.

#### F.25.4. Die Funktion `pg_buffercache_usage_counts()`

Die von der Funktion ausgegebenen Spalten sind in Tabelle F.17 beschrieben.

**Tabelle F.17. Ausgabespalten von `pg_buffercache_usage_counts()`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `usage_count` | `int4` | Ein möglicher Buffer Usage Count |
| `buffers` | `int4` | Anzahl der Buffer mit diesem Usage Count |
| `dirty` | `int4` | Anzahl der dirty Buffer mit diesem Usage Count |
| `pinned` | `int4` | Anzahl der gepinnten Buffer mit diesem Usage Count |

`pg_buffercache_usage_counts()` gibt eine Menge von Zeilen zurück, die den Zustand aller Shared Buffers aggregiert nach den möglichen Usage-Count-Werten zusammenfasst. Ähnliche und detailliertere Informationen liefert die Sicht `pg_buffercache`, aber `pg_buffercache_usage_counts()` ist deutlich günstiger.

Wie die Sicht `pg_buffercache` nimmt auch `pg_buffercache_usage_counts()` keine Buffer-Manager-Sperren. Gleichzeitige Aktivität kann daher zu kleinen Ungenauigkeiten im Ergebnis führen.

#### F.25.5. Die Funktion `pg_buffercache_evict()`

`pg_buffercache_evict()` nimmt eine Buffer-ID entgegen, wie sie in der Spalte `bufferid` der Sicht `pg_buffercache` angezeigt wird. Die Funktion gibt Informationen darüber zurück, ob der Buffer verdrängt und geschrieben wurde. Die Spalte `buffer_evicted` ist bei Erfolg `true` und `false`, wenn der Buffer ungültig war, wegen eines Pins nicht verdrängt werden konnte oder nach einem Schreibversuch wieder dirty wurde.

Die Spalte `buffer_flushed` ist `true`, wenn der Buffer geschrieben wurde. Das bedeutet nicht zwingend, dass die Funktion selbst ihn geschrieben hat; dies kann auch durch eine andere Aktivität geschehen sein. Das Ergebnis ist direkt nach der Rückgabe bereits potenziell veraltet, da der Buffer durch gleichzeitige Aktivität jederzeit wieder gültig werden kann. Die Funktion ist nur für Entwicklertests gedacht.

#### F.25.6. Die Funktion `pg_buffercache_evict_relation()`

`pg_buffercache_evict_relation()` ist `pg_buffercache_evict()` sehr ähnlich. Der Unterschied besteht darin, dass `pg_buffercache_evict_relation()` statt einer Buffer-ID eine Relationskennung entgegennimmt. Die Funktion versucht, alle Buffer für alle Forks dieser Relation zu verdrängen. Sie gibt die Anzahl der verdrängten Buffer, der geschriebenen Buffer und der nicht verdrängbaren Buffer zurück. Geschriebene Buffer wurden nicht notwendigerweise von dieser Funktion geschrieben; dies kann auch durch andere Aktivität geschehen sein. Das Ergebnis ist direkt nach der Rückgabe bereits potenziell veraltet, weil Buffer durch gleichzeitige Aktivität sofort wieder eingelesen werden können. Die Funktion ist nur für Entwicklertests gedacht.

#### F.25.7. Die Funktion `pg_buffercache_evict_all()`

`pg_buffercache_evict_all()` ist `pg_buffercache_evict()` sehr ähnlich. Der Unterschied besteht darin, dass `pg_buffercache_evict_all()` kein Argument entgegennimmt, sondern versucht, alle Buffer im Buffer Pool zu verdrängen. Die Funktion gibt die Anzahl der verdrängten Buffer, der geschriebenen Buffer und der nicht verdrängbaren Buffer zurück. Geschriebene Buffer wurden nicht notwendigerweise von dieser Funktion geschrieben; dies kann auch durch andere Aktivität geschehen sein. Das Ergebnis ist direkt nach der Rückgabe bereits potenziell veraltet, weil Buffer durch gleichzeitige Aktivität sofort wieder eingelesen werden können. Die Funktion ist nur für Entwicklertests gedacht.

#### F.25.8. Beispielausgabe

```sql
regression=# SELECT n.nspname, c.relname, count(*) AS buffers
FROM pg_buffercache b JOIN pg_class c
ON b.relfilenode = pg_relation_filenode(c.oid) AND
b.reldatabase IN (0, (SELECT oid FROM pg_database
WHERE datname =
current_database()))
JOIN pg_namespace n ON n.oid = c.relnamespace
GROUP BY n.nspname, c.relname
ORDER BY 3 DESC
LIMIT 10;
```

```text
nspname   |        relname         | buffers
------------+------------------------+---------
public     | delete_test_table      |     593
public     | delete_test_table_pkey |     494
pg_catalog | pg_attribute           |     472
public     | quad_poly_tbl          |     353
public     | tenk2                  |     349
public     | tenk1                  |     349
public     | gin_test_idx           |     306
pg_catalog | pg_largeobject         |     206
public     | gin_test_tbl           |     188
public     | spgist_text_tbl        |     182
(10 rows)
```

```sql
regression=# SELECT * FROM pg_buffercache_summary();
```

```text
buffers_used | buffers_unused | buffers_dirty | buffers_pinned |
usagecount_avg
--------------+----------------+---------------+----------------
+----------------
248 |        2096904 |            39 |              0 |
3.141129
(1 row)
```

```sql
regression=# SELECT * FROM pg_buffercache_usage_counts();
```

```text
usage_count | buffers | dirty | pinned
-------------+---------+-------+--------
0 |   14650 |     0 |      0
1 |    1436 |   671 |      0
2 |     102 |    88 |      0
3 |      23 |    21 |      0
4 |       9 |     7 |      0
5 |     164 |   106 |      0
(6 rows)
```

#### F.25.9. Autoren

Mark Kirkwood <markir@paradise.net.nz>

Designvorschläge: Neil Conway <neilc@samurai.com>

Debugging-Hinweise: Tom Lane <tgl@sss.pgh.pa.us>


### F.26. pgcrypto — kryptographische Funktionen

Das Modul `pgcrypto` stellt kryptographische Funktionen für PostgreSQL bereit.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

`pgcrypto` benötigt OpenSSL und wird nicht installiert, wenn PostgreSQL ohne OpenSSL-Unterstützung gebaut wurde.

#### F.26.1. Allgemeine Hash-Funktionen

##### F.26.1.1. digest()

```text
digest(data text, type text) returns bytea
digest(data bytea, type text) returns bytea
```

Berechnet einen binären Hash der angegebenen Daten. `type` ist der zu verwendende Algorithmus. Standardalgorithmen sind `md5`, `sha1`, `sha224`, `sha256`, `sha384` und `sha512`. Außerdem werden alle Digest-Algorithmen automatisch übernommen, die OpenSSL unterstützt.

Wenn Sie den Digest als hexadezimale Zeichenkette benötigen, verwenden Sie `encode()` auf dem Ergebnis. Beispiel:

```sql
CREATE OR REPLACE FUNCTION sha1(bytea) returns text AS $$
SELECT encode(digest($1, 'sha1'), 'hex')
$$ LANGUAGE SQL STRICT IMMUTABLE;
```

##### F.26.1.2. hmac()

```text
hmac(data text, key text, type text) returns bytea
hmac(data bytea, key bytea, type text) returns bytea
```

Berechnet einen gehashten MAC für `data` mit dem Schlüssel `key`. `type` hat dieselbe Bedeutung wie bei `digest()`.

Dies ähnelt `digest()`, aber der Hash kann nur mit Kenntnis des Schlüssels neu berechnet werden. Dadurch wird verhindert, dass jemand Daten verändert und zugleich den Hash passend dazu austauscht.

Wenn der Schlüssel größer als die Blockgröße des Hashs ist, wird er zuerst gehasht und das Ergebnis als Schlüssel verwendet.

#### F.26.2. Passwort-Hash-Funktionen

Die Funktionen `crypt()` und `gen_salt()` sind speziell für das Hashen von Passwörtern ausgelegt. `crypt()` führt das Hashing aus, und `gen_salt()` bereitet die Algorithmusparameter dafür vor.

Die Algorithmen in `crypt()` unterscheiden sich von üblichen MD5- oder SHA-1-Hash-Algorithmen in folgenden Punkten:

1. Sie sind langsam. Da die Datenmenge sehr klein ist, ist dies die einzige Möglichkeit, Brute-Force-Angriffe auf Passwörter zu erschweren.
2. Sie verwenden einen Zufallswert, den Salt, sodass Benutzer mit demselben Passwort unterschiedliche verschlüsselte Passwörter erhalten. Das erschwert zusätzlich das Umkehren des Algorithmus.
3. Sie enthalten den Algorithmustyp im Ergebnis, sodass mit unterschiedlichen Algorithmen gehashte Passwörter nebeneinander existieren können.
4. Einige sind adaptiv. Wenn Computer schneller werden, kann der Algorithmus langsamer eingestellt werden, ohne bestehende Passwort-Hashes inkompatibel zu machen.

Tabelle F.18 zeigt die von `crypt()` unterstützten Algorithmen.

**Tabelle F.18. Unterstützte Algorithmen für `crypt()`**

| Algorithmus | Maximale Passwortlänge | Adaptiv? | Salt-Bits | Ausgabelänge | Beschreibung |
|---|---:|---|---:|---:|---|
| `bf` | 72 | ja | 128 | 60 | Blowfish-basiert, Variante 2a |
| `md5` | unbegrenzt | nein | 48 | 34 | MD5-basiertes `crypt` |
| `xdes` | 8 | ja | 24 | 20 | Erweitertes DES |
| `des` | 8 | nein | 12 | 13 | Ursprüngliches UNIX-`crypt` |
| `sha256crypt` | unbegrenzt | ja | bis zu 32 | 80 | Aus öffentlich verfügbarer UNIX-`crypt`-Referenzimplementierung mit SHA-256/SHA-512 angepasst |
| `sha512crypt` | unbegrenzt | ja | bis zu 32 | 123 | Aus öffentlich verfügbarer UNIX-`crypt`-Referenzimplementierung mit SHA-256/SHA-512 angepasst |

##### F.26.2.1. crypt()

```text
crypt(password text, salt text) returns text
```

Berechnet einen Hash im Stil von `crypt(3)` für `password`. Beim Speichern eines neuen Passworts verwenden Sie `gen_salt()`, um einen neuen Salt-Wert zu erzeugen. Zum Prüfen eines Passworts übergeben Sie den gespeicherten Hash-Wert als `salt` und testen, ob das Ergebnis dem gespeicherten Wert entspricht.

Beispiel zum Setzen eines neuen Passworts:

```sql
UPDATE ... SET pswhash = crypt('new password', gen_salt('md5'));
```

Beispiel für die Authentifizierung:

```sql
SELECT (pswhash = crypt('entered password', pswhash)) AS pswmatch
FROM ... ;
```

Dies gibt `true` zurück, wenn das eingegebene Passwort korrekt ist.

##### F.26.2.2. gen_salt()

```text
gen_salt(type text [, iter_count integer ]) returns text
```

Erzeugt eine neue zufällige Salt-Zeichenkette für `crypt()`. Die Salt-Zeichenkette teilt `crypt()` auch mit, welcher Algorithmus verwendet werden soll.

Der Parameter `type` gibt den Hashing-Algorithmus an. Zulässige Typen sind `des`, `xdes`, `md5`, `bf`, `sha256crypt` und `sha512crypt`. Die letzten beiden sind moderne SHA-2-basierte Passwort-Hashes. Siehe <https://www.akkadia.org/drepper/SHA-crypt.txt>.

Der Parameter `iter_count` erlaubt es, für Algorithmen mit Iterationszählung die Anzahl der Iterationen festzulegen. Je höher der Wert, desto länger dauert das Hashen des Passworts und desto länger dauert ein Angriff. Ist der Wert allerdings zu hoch, kann das Berechnen eines Hashs mehrere Jahre dauern, was wenig praktikabel ist. Wird `iter_count` weggelassen, wird der Standardwert verwendet. Die zulässigen Werte hängen vom Algorithmus ab und sind in Tabelle F.19 gezeigt.

**Tabelle F.19. Iterationszahlen für `crypt()`**

| Algorithmus | Standard | Minimum | Maximum |
|---|---:|---:|---:|
| `xdes` | 725 | 1 | 16777215 |
| `bf` | 6 | 4 | 31 |
| `sha256crypt`, `sha512crypt` | 5000 | 1000 | 999999999 |

Für `xdes` gilt zusätzlich, dass die Iterationszahl ungerade sein muss.

Zur Wahl einer passenden Iterationszahl sollten Sie bedenken, dass das ursprüngliche DES-`crypt` auf damaliger Hardware auf etwa vier Hashes pro Sekunde ausgelegt war. Langsamer als vier Hashes pro Sekunde dürfte die Benutzbarkeit beeinträchtigen; schneller als 100 Hashes pro Sekunde ist vermutlich zu schnell.

Tabelle F.20 gibt einen Überblick über die relative Langsamkeit verschiedener Hashing-Algorithmen. Die Tabelle zeigt, wie lange es dauern würde, alle Zeichenkombinationen eines acht Zeichen langen Passworts zu testen, einmal nur mit Kleinbuchstaben und einmal mit Groß- und Kleinbuchstaben sowie Ziffern. Bei den `crypt-bf`-Einträgen ist die Zahl nach dem Schrägstrich der Parameter `iter_count` von `gen_salt()`.

Der Standardwert `5000` für `sha256crypt` und `sha512crypt` gilt für moderne Hardware als zu niedrig, kann aber angepasst werden, um stärkere Passwort-Hashes zu erzeugen. Ansonsten gelten sowohl `sha256crypt` als auch `sha512crypt` als sicher.

**Tabelle F.20. Geschwindigkeit von Hash-Algorithmen**

| Algorithmus | Hashes/Sek. | Für `[a-z]` | Für `[A-Za-z0-9]` | Dauer relativ zu MD5-Hash |
|---|---:|---:|---:|---:|
| `crypt-bf/8` | 1792 | 4 Jahre | 3927 Jahre | 100k |
| `crypt-bf/7` | 3648 | 2 Jahre | 1929 Jahre | 50k |
| `crypt-bf/6` | 7168 | 1 Jahr | 982 Jahre | 25k |
| `crypt-bf/5` | 13504 | 188 Tage | 521 Jahre | 12.5k |
| `crypt-md5` | 171584 | 15 Tage | 41 Jahre | 1k |
| `crypt-des` | 23221568 | 157,5 Minuten | 108 Tage | 7 |
| `sha1` | 37774272 | 90 Minuten | 68 Tage | 4 |
| `md5` (Hash) | 150085504 | 22,5 Minuten | 17 Tage | 1 |

Hinweise:

- Die verwendete Maschine war ein Intel Mobile Core i3.
- Die Werte für `crypt-des` und `crypt-md5` stammen aus der Ausgabe von `john -test` aus John the Ripper v1.6.38.
- Die MD5-Hash-Werte stammen aus `mdcrack` 1.2.
- Die SHA-1-Werte stammen aus `lcrack-20031130-beta`.
- Die `crypt-bf`-Werte stammen aus einem einfachen Programm, das über 1000 Passwörter mit acht Zeichen iteriert. So lässt sich die Geschwindigkeit bei unterschiedlichen Iterationszahlen zeigen. Zum Vergleich: `john -test` zeigt 13506 Loops/Sek. für `crypt-bf/5`. Der sehr kleine Unterschied passt dazu, dass die `crypt-bf`-Implementierung in `pgcrypto` dieselbe ist wie in John the Ripper.

Beachten Sie, dass „alle Kombinationen ausprobieren“ kein realistisches Szenario ist. Passwort-Cracking geschieht normalerweise mithilfe von Wörterbüchern, die reguläre Wörter und verschiedene Mutationen enthalten. Auch halbwegs wortähnliche Passwörter können daher deutlich schneller geknackt werden, als die obigen Zahlen nahelegen; ein sechs Zeichen langes, nicht wortähnliches Passwort kann dagegen möglicherweise standhalten. Oder auch nicht.

#### F.26.3. PGP-Verschlüsselungsfunktionen

Die Funktionen in diesem Abschnitt implementieren den Verschlüsselungsteil des OpenPGP-Standards ([RFC 4880](https://datatracker.ietf.org/doc/html/rfc4880)). Unterstützt werden sowohl symmetrische Verschlüsselung als auch Public-Key-Verschlüsselung.

Eine verschlüsselte PGP-Nachricht besteht aus zwei Paketen:

- einem Paket mit einem Sitzungsschlüssel, der entweder symmetrisch oder mit einem öffentlichen Schlüssel verschlüsselt ist
- einem Paket mit den Daten, die mit dem Sitzungsschlüssel verschlüsselt sind

Bei der Verschlüsselung mit einem symmetrischen Schlüssel, also einem Passwort:

1. Das angegebene Passwort wird mit einem String2Key-Algorithmus (S2K) gehasht. Das ähnelt `crypt()`-Algorithmen: absichtlich langsam und mit zufälligem Salt, erzeugt aber einen binären Schlüssel voller Länge.
2. Wenn ein separater Sitzungsschlüssel angefordert wird, wird ein neuer Zufallsschlüssel erzeugt. Andernfalls wird der S2K-Schlüssel direkt als Sitzungsschlüssel verwendet.
3. Wird der S2K-Schlüssel direkt verwendet, werden nur die S2K-Einstellungen in das Sitzungsschlüsselpaket geschrieben. Andernfalls wird der Sitzungsschlüssel mit dem S2K-Schlüssel verschlüsselt und in das Sitzungsschlüsselpaket geschrieben.

Bei der Verschlüsselung mit einem öffentlichen Schlüssel:

1. Es wird ein neuer zufälliger Sitzungsschlüssel erzeugt.
2. Er wird mit dem öffentlichen Schlüssel verschlüsselt und in das Sitzungsschlüsselpaket geschrieben.

In beiden Fällen werden die zu verschlüsselnden Daten wie folgt verarbeitet:

1. Optionale Datenmanipulation: Komprimierung, Konvertierung nach UTF-8 und/oder Konvertierung der Zeilenenden.
2. Den Daten wird ein Block zufälliger Bytes vorangestellt. Das entspricht der Verwendung eines zufälligen IV.
3. Ein SHA-1-Hash des zufälligen Präfixes und der Daten wird angehängt.
4. Alles zusammen wird mit dem Sitzungsschlüssel verschlüsselt und in das Datenpaket geschrieben.

##### F.26.3.1. pgp_sym_encrypt()

```text
pgp_sym_encrypt(data text, psw text [, options text ]) returns bytea
pgp_sym_encrypt_bytea(data bytea, psw text [, options text ]) returns bytea
```

Verschlüsselt Daten mit einem symmetrischen PGP-Schlüssel `psw`. Der Parameter `options` kann Optionen enthalten, wie unten beschrieben.

##### F.26.3.2. pgp_sym_decrypt()

```text
pgp_sym_decrypt(msg bytea, psw text [, options text ]) returns text
pgp_sym_decrypt_bytea(msg bytea, psw text [, options text ]) returns bytea
```

Entschlüsselt eine symmetrisch verschlüsselte PGP-Nachricht.

Das Entschlüsseln von `bytea`-Daten mit `pgp_sym_decrypt()` ist nicht erlaubt, um die Ausgabe ungültiger Zeichendaten zu vermeiden. Ursprünglich textuelle Daten mit `pgp_sym_decrypt_bytea()` zu entschlüsseln ist in Ordnung.

Der Parameter `options` kann Optionen enthalten, wie unten beschrieben.

##### F.26.3.3. pgp_pub_encrypt()

```text
pgp_pub_encrypt(data text, key bytea [, options text ]) returns bytea
pgp_pub_encrypt_bytea(data bytea, key bytea [, options text ]) returns bytea
```

Verschlüsselt Daten mit einem öffentlichen PGP-Schlüssel `key`. Wird dieser Funktion ein geheimer Schlüssel übergeben, entsteht ein Fehler.

Der Parameter `options` kann Optionen enthalten, wie unten beschrieben.

##### F.26.3.4. pgp_pub_decrypt()

```text
pgp_pub_decrypt(msg bytea, key bytea [, psw text [, options text ]]) returns text
pgp_pub_decrypt_bytea(msg bytea, key bytea [, psw text [, options text ]]) returns bytea
```

Entschlüsselt eine mit öffentlichem Schlüssel verschlüsselte Nachricht. `key` muss der geheime Schlüssel sein, der zu dem öffentlichen Schlüssel gehört, mit dem verschlüsselt wurde. Wenn der geheime Schlüssel passwortgeschützt ist, müssen Sie das Passwort in `psw` angeben. Gibt es kein Passwort, Sie möchten aber Optionen angeben, müssen Sie ein leeres Passwort übergeben.

Das Entschlüsseln von `bytea`-Daten mit `pgp_pub_decrypt()` ist nicht erlaubt, um die Ausgabe ungültiger Zeichendaten zu vermeiden. Ursprünglich textuelle Daten mit `pgp_pub_decrypt_bytea()` zu entschlüsseln ist in Ordnung.

Der Parameter `options` kann Optionen enthalten, wie unten beschrieben.

##### F.26.3.5. pgp_key_id()

```text
pgp_key_id(bytea) returns text
```

`pgp_key_id()` extrahiert die Schlüssel-ID eines öffentlichen oder geheimen PGP-Schlüssels. Wird eine verschlüsselte Nachricht übergeben, liefert die Funktion die Schlüssel-ID, die zur Verschlüsselung verwendet wurde.

Die Funktion kann zwei besondere Schlüssel-IDs zurückgeben:

- `SYMKEY`: Die Nachricht ist mit einem symmetrischen Schlüssel verschlüsselt.
- `ANYKEY`: Die Nachricht ist mit einem öffentlichen Schlüssel verschlüsselt, aber die Schlüssel-ID wurde entfernt. Sie müssen dann alle Ihre geheimen Schlüssel ausprobieren, um den passenden zu finden. `pgcrypto` selbst erzeugt solche Nachrichten nicht.

Beachten Sie, dass verschiedene Schlüssel dieselbe ID haben können. Das ist selten, aber normal. Die Client-Anwendung sollte dann mit jedem Schlüssel zu entschlüsseln versuchen, ähnlich wie bei `ANYKEY`.

##### F.26.3.6. armor(), dearmor()

```text
armor(data bytea [ , keys text[], values text[] ]) returns text
dearmor(data text) returns bytea
```

Diese Funktionen verpacken binäre Daten in das PGP-ASCII-Armor-Format oder entpacken sie daraus. Dabei handelt es sich im Wesentlichen um Base64 mit CRC und zusätzlicher Formatierung.

Wenn die Arrays `keys` und `values` angegeben werden, wird für jedes Schlüssel/Wert-Paar ein Armor-Header hinzugefügt. Beide Arrays müssen eindimensional und gleich lang sein. Schlüssel und Werte dürfen keine Nicht-ASCII-Zeichen enthalten.

##### F.26.3.7. pgp_armor_headers

```text
pgp_armor_headers(data text, key out text, value out text) returns setof record
```

`pgp_armor_headers()` extrahiert die Armor-Header aus `data`. Der Rückgabewert ist eine Menge von Zeilen mit den Spalten `key` und `value`. Enthalten Schlüssel oder Werte Nicht-ASCII-Zeichen, werden sie als UTF-8 behandelt.

##### F.26.3.8. Optionen für PGP-Funktionen

Die Optionsnamen sind an GnuPG angelehnt. Der Wert einer Option wird nach einem Gleichheitszeichen angegeben; mehrere Optionen werden durch Kommas getrennt. Beispiel:

```sql
pgp_sym_encrypt(data, psw, 'compress-algo=1, cipher-algo=aes256')
```

Alle Optionen außer `convert-crlf` gelten nur für Verschlüsselungsfunktionen. Entschlüsselungsfunktionen beziehen die Parameter aus den PGP-Daten.

Die interessantesten Optionen sind vermutlich `compress-algo` und `unicode-mode`. Die übrigen sollten sinnvolle Standardwerte haben.

###### F.26.3.8.1. cipher-algo

Legt den zu verwendenden Verschlüsselungsalgorithmus fest.

- Werte: `bf`, `aes128`, `aes192`, `aes256`, `3des`, `cast5`
- Standard: `aes128`
- Gilt für: `pgp_sym_encrypt`, `pgp_pub_encrypt`

###### F.26.3.8.2. compress-algo

Legt den zu verwendenden Komprimierungsalgorithmus fest. Nur verfügbar, wenn PostgreSQL mit `zlib` gebaut wurde.

- Werte:
  - `0`: keine Komprimierung
  - `1`: ZIP-Komprimierung
  - `2`: ZLIB-Komprimierung, also ZIP plus Metadaten und Block-CRCs
- Standard: `0`
- Gilt für: `pgp_sym_encrypt`, `pgp_pub_encrypt`

###### F.26.3.8.3. compress-level

Legt fest, wie stark komprimiert wird. Höhere Stufen komprimieren stärker, sind aber langsamer. `0` deaktiviert die Komprimierung.

- Werte: `0`, `1` bis `9`
- Standard: `6`
- Gilt für: `pgp_sym_encrypt`, `pgp_pub_encrypt`

###### F.26.3.8.4. convert-crlf

Legt fest, ob beim Verschlüsseln `\n` in `\r\n` und beim Entschlüsseln `\r\n` in `\n` konvertiert wird. RFC 4880 schreibt vor, dass Textdaten mit `\r\n`-Zeilenenden gespeichert werden sollen. Verwenden Sie diese Option für vollständig RFC-konformes Verhalten.

- Werte: `0`, `1`
- Standard: `0`
- Gilt für: `pgp_sym_encrypt`, `pgp_pub_encrypt`, `pgp_sym_decrypt`, `pgp_pub_decrypt`

###### F.26.3.8.5. disable-mdc

Schützt Daten nicht mit SHA-1. Der einzige gute Grund für diese Option ist Kompatibilität mit sehr alten PGP-Produkten, die älter sind als die Einführung SHA-1-geschützter Pakete in RFC 4880. Aktuelle Software von `gnupg.org` und `pgp.com` unterstützt diese Pakete problemlos.

- Werte: `0`, `1`
- Standard: `0`
- Gilt für: `pgp_sym_encrypt`, `pgp_pub_encrypt`

###### F.26.3.8.6. sess-key

Verwendet einen separaten Sitzungsschlüssel. Public-Key-Verschlüsselung verwendet immer einen separaten Sitzungsschlüssel; diese Option ist für symmetrische Verschlüsselung gedacht, die standardmäßig direkt den S2K-Schlüssel verwendet.

- Werte: `0`, `1`
- Standard: `0`
- Gilt für: `pgp_sym_encrypt`

###### F.26.3.8.7. s2k-mode

Legt fest, welcher S2K-Algorithmus verwendet wird.

- Werte:
  - `0`: ohne Salt. Gefährlich!
  - `1`: mit Salt, aber fester Iterationszahl
  - `3`: variable Iterationszahl
- Standard: `3`
- Gilt für: `pgp_sym_encrypt`

###### F.26.3.8.8. s2k-count

Die Anzahl der Iterationen des S2K-Algorithmus. Der Wert muss zwischen `1024` und `65011712` einschließlich liegen.

- Standard: ein Zufallswert zwischen `65536` und `253952`
- Gilt für: `pgp_sym_encrypt`, nur mit `s2k-mode=3`

###### F.26.3.8.9. s2k-digest-algo

Legt den Digest-Algorithmus für die S2K-Berechnung fest.

- Werte: `md5`, `sha1`
- Standard: `sha1`
- Gilt für: `pgp_sym_encrypt`

###### F.26.3.8.10. s2k-cipher-algo

Legt fest, welcher Cipher zur Verschlüsselung eines separaten Sitzungsschlüssels verwendet wird.

- Werte: `bf`, `aes`, `aes128`, `aes192`, `aes256`
- Standard: `cipher-algo` verwenden
- Gilt für: `pgp_sym_encrypt`

###### F.26.3.8.11. unicode-mode

Legt fest, ob Textdaten von der internen Datenbankkodierung nach UTF-8 und zurück konvertiert werden. Wenn Ihre Datenbank bereits UTF-8 verwendet, findet keine Konvertierung statt, aber die Nachricht wird als UTF-8 markiert. Ohne diese Option geschieht das nicht.

- Werte: `0`, `1`
- Standard: `0`
- Gilt für: `pgp_sym_encrypt`, `pgp_pub_encrypt`

##### F.26.3.9. PGP-Schlüssel mit GnuPG erzeugen

Einen neuen Schlüssel erzeugen:

```text
gpg --gen-key
```

Der bevorzugte Schlüsseltyp ist „DSA and Elgamal“.

Für RSA-Verschlüsselung müssen Sie entweder einen DSA- oder RSA-Schlüssel nur zum Signieren als Hauptschlüssel erzeugen und danach mit `gpg --edit-key` einen RSA-Verschlüsselungs-Subkey hinzufügen.

Schlüssel auflisten:

```text
gpg --list-secret-keys
```

Einen öffentlichen Schlüssel im ASCII-Armor-Format exportieren:

```text
gpg -a --export KEYID > public.key
```

Einen geheimen Schlüssel im ASCII-Armor-Format exportieren:

```text
gpg -a --export-secret-keys KEYID > secret.key
```

Sie müssen `dearmor()` auf diese Schlüssel anwenden, bevor Sie sie den PGP-Funktionen übergeben. Wenn Sie binäre Daten verarbeiten können, können Sie alternativ `-a` im Befehl weglassen.

Weitere Einzelheiten finden Sie in `man gpg`, im [GNU Privacy Handbook](https://www.gnupg.org/gph/en/manual.html) und in weiterer Dokumentation auf <https://www.gnupg.org/>.

##### F.26.3.10. Einschränkungen des PGP-Codes

- Keine Unterstützung für Signaturen. Das bedeutet auch, dass nicht geprüft wird, ob der Verschlüsselungs-Subkey zum Hauptschlüssel gehört.
- Keine Unterstützung für einen Verschlüsselungsschlüssel als Hauptschlüssel. Da diese Praxis allgemein nicht empfohlen wird, sollte das kein Problem sein.
- Keine Unterstützung für mehrere Subkeys. Das kann problematisch erscheinen, weil diese Praxis üblich ist. Andererseits sollten Sie Ihre normalen GPG-/PGP-Schlüssel nicht mit `pgcrypto` verwenden, sondern wegen des anderen Einsatzszenarios neue Schlüssel erzeugen.

#### F.26.4. Roh-Verschlüsselungsfunktionen

Diese Funktionen wenden lediglich einen Cipher auf Daten an; sie besitzen keine erweiterten Eigenschaften der PGP-Verschlüsselung. Dadurch haben sie einige wesentliche Probleme:

1. Sie verwenden den Benutzerschlüssel direkt als Cipher-Schlüssel.
2. Sie bieten keine Integritätsprüfung, mit der sich erkennen ließe, ob verschlüsselte Daten verändert wurden.
3. Sie erwarten, dass Benutzer alle Verschlüsselungsparameter einschließlich IV selbst verwalten.
4. Sie behandeln keinen Text.

Mit der Einführung der PGP-Verschlüsselung wird die Verwendung der Roh-Verschlüsselungsfunktionen daher nicht empfohlen.

```text
encrypt(data bytea, key bytea, type text) returns bytea
decrypt(data bytea, key bytea, type text) returns bytea

encrypt_iv(data bytea, key bytea, iv bytea, type text) returns bytea
decrypt_iv(data bytea, key bytea, iv bytea, type text) returns bytea
```

Verschlüsselt oder entschlüsselt Daten mit der durch `type` angegebenen Cipher-Methode. Die Syntax der `type`-Zeichenkette ist:

```text
algorithm [ - mode ] [ /pad: padding ]
```

`algorithm` ist einer der folgenden Werte:

- `bf`: Blowfish
- `aes`: AES (Rijndael-128, -192 oder -256)

`mode` ist einer der folgenden Werte:

- `cbc`: der nächste Block hängt vom vorherigen ab (Standard)
- `cfb`: der nächste Block hängt vom vorherigen verschlüsselten Block ab
- `ecb`: jeder Block wird separat verschlüsselt (nur zum Testen)

`padding` ist einer der folgenden Werte:

- `pkcs`: Daten können beliebige Länge haben (Standard)
- `none`: Daten müssen ein Vielfaches der Cipher-Blockgröße sein

Die folgenden Aufrufe sind also äquivalent:

```sql
encrypt(data, 'fooz', 'bf')
encrypt(data, 'fooz', 'bf-cbc/pad:pkcs')
```

In `encrypt_iv()` und `decrypt_iv()` ist der Parameter `iv` der Initialwert für die Modi CBC und CFB; bei ECB wird er ignoriert. Wenn er nicht genau Blockgröße hat, wird er abgeschnitten oder mit Nullen aufgefüllt. In den Funktionen ohne diesen Parameter ist der Standardwert vollständig aus Nullen zusammengesetzt.

#### F.26.5. Zufallsdatenfunktionen

```text
gen_random_bytes(count integer) returns bytea
```

Gibt `count` kryptographisch starke Zufallsbytes zurück. Es können höchstens 1024 Bytes auf einmal entnommen werden, um den Pool des Zufallsgenerators nicht zu erschöpfen.

```text
gen_random_uuid() returns uuid
```

Gibt eine Version-4-UUID, also eine zufällige UUID, zurück. Diese Funktion ist veraltet; intern ruft sie die gleichnamige Kernfunktion auf.

#### F.26.6. OpenSSL-Unterstützungsfunktionen

```text
fips_mode() returns boolean
```

Gibt `true` zurück, wenn OpenSSL im FIPS-Modus läuft, andernfalls `false`.

#### F.26.7. Konfigurationsparameter

Es gibt einen Konfigurationsparameter, der das Verhalten von `pgcrypto` steuert.

`pgcrypto.builtin_crypto_enabled` (`enum`)

`pgcrypto.builtin_crypto_enabled` legt fest, ob die eingebauten Kryptofunktionen `gen_salt()` und `crypt()` verfügbar sind. Der Wert `off` deaktiviert diese Funktionen. `on` (Standard) aktiviert sie normal. `fips` deaktiviert sie, wenn erkannt wird, dass OpenSSL im FIPS-Modus arbeitet.

Üblicherweise wird dieser Parameter in `postgresql.conf` gesetzt; Superuser können ihn jedoch in ihren eigenen Sitzungen zur Laufzeit ändern.

#### F.26.8. Hinweise

##### F.26.8.1. Konfiguration

`pgcrypto` konfiguriert sich anhand der Ergebnisse des Haupt-`configure`-Skripts von PostgreSQL. Die relevanten Optionen sind `--with-zlib` und `--with-ssl=openssl`.

Wenn PostgreSQL mit `zlib` gebaut wurde, können PGP-Verschlüsselungsfunktionen Daten vor dem Verschlüsseln komprimieren.

`pgcrypto` benötigt OpenSSL. Andernfalls wird es nicht gebaut oder installiert.

Bei Kompilierung gegen OpenSSL 3.0.0 und neuere Versionen muss der Legacy-Provider in der Konfigurationsdatei `openssl.cnf` aktiviert werden, um ältere Cipher wie DES oder Blowfish zu verwenden.

##### F.26.8.2. Umgang mit NULL

Wie in SQL üblich geben alle Funktionen `NULL` zurück, wenn eines der Argumente `NULL` ist. Bei unvorsichtiger Verwendung kann dies Sicherheitsrisiken erzeugen.

##### F.26.8.3. Sicherheitseinschränkungen

Alle `pgcrypto`-Funktionen laufen innerhalb des Datenbankservers. Das bedeutet, dass alle Daten und Passwörter zwischen `pgcrypto` und Client-Anwendungen im Klartext übertragen werden. Daher müssen Sie:

1. lokal verbinden oder SSL-Verbindungen verwenden
2. sowohl dem Systemadministrator als auch dem Datenbankadministrator vertrauen

Wenn Sie das nicht können, sollten Sie Kryptographie besser in der Client-Anwendung durchführen.

Die Implementierung widersteht keinen Side-Channel-Angriffen. Beispielsweise variiert die Zeit, die eine `pgcrypto`-Entschlüsselungsfunktion benötigt, zwischen Ciphertexts gleicher Größe.

Siehe auch <https://en.wikipedia.org/wiki/Side-channel_attack>.

#### F.26.9. Autor

Marko Kreen <markokr@gmail.com>

`pgcrypto` verwendet Code aus folgenden Quellen:

| Algorithmus | Autor | Herkunft |
|---|---|---|
| DES `crypt` | David Burren und andere | FreeBSD `libcrypt` |
| MD5 `crypt` | Poul-Henning Kamp | FreeBSD `libcrypt` |
| Blowfish `crypt` | Solar Designer | <https://www.openwall.com> |


### F.27. pg_freespacemap — Free Space Map untersuchen

Das Modul `pg_freespacemap` stellt Mittel bereit, um die Free Space Map (FSM) zu untersuchen. Genauer gesagt stellt es zwei überladene Varianten der Funktion `pg_freespace` bereit. Die Funktionen zeigen den in der Free Space Map gespeicherten Wert für eine bestimmte Seite oder für alle Seiten einer Relation.

Standardmäßig ist die Verwendung auf Superuser und Rollen mit den Rechten der Rolle `pg_stat_scan_tables` beschränkt. Zugriff kann anderen Rollen mit `GRANT` erteilt werden.

#### F.27.1. Funktionen

```text
pg_freespace(rel regclass IN, blkno bigint IN) returns int2
```

Gibt gemäß FSM die Menge freien Speicherplatzes auf der durch `blkno` angegebenen Seite der Relation zurück.

```text
pg_freespace(rel regclass IN, blkno OUT bigint, avail OUT int2)
```

Zeigt gemäß FSM die Menge freien Speicherplatzes auf jeder Seite der Relation an. Zurückgegeben wird eine Menge von Tupeln `(blkno bigint, avail int2)`, ein Tupel für jede Seite der Relation.

Die in der Free Space Map gespeicherten Werte sind nicht exakt. Sie werden auf eine Genauigkeit von 1/256 von `BLCKSZ` gerundet (32 Bytes beim Standardwert von `BLCKSZ`) und beim Einfügen und Aktualisieren von Tupeln nicht vollständig aktuell gehalten.

Bei Indizes werden vollständig unbenutzte Seiten verfolgt, nicht freier Platz innerhalb von Seiten. Die Werte sind daher nicht als Speicherplatzangaben sinnvoll, sondern zeigen nur, ob eine Seite verwendet wird oder leer ist.

#### F.27.2. Beispielausgabe

```sql
postgres=# SELECT * FROM pg_freespace('foo');
```

```text
blkno | avail
-------+-------
0 |     0
1 |     0
2 |     0
3 |    32
4 |   704
5 |   704
6 |   704
7 | 1216
8 |   704
9 |   704
10 |   704
11 |   704
12 |   704
13 |   704
14 |   704
15 |   704
16 |   704
17 |   704
18 |   704
19 | 3648
(20 rows)
```

```sql
postgres=# SELECT * FROM pg_freespace('foo', 7);
```

```text
pg_freespace
--------------
         1216
(1 row)
```

#### F.27.3. Autor

Theodore Wilk <theodorewilk@alumni.ucla.edu>


### F.28. pg_logicalinspect — Komponenten der logischen Decodierung untersuchen

Das Modul `pg_logicalinspect` stellt SQL-Funktionen bereit, mit denen sich Inhalte von Komponenten der logischen Decodierung untersuchen lassen. Es erlaubt insbesondere die Untersuchung serialisierter logischer Snapshots eines laufenden PostgreSQL-Datenbankclusters, was für Debugging und Lernzwecke nützlich ist.

Standardmäßig ist die Verwendung dieser Funktionen auf Superuser und Mitglieder der Rolle `pg_read_server_files` beschränkt. Superuser können anderen Rollen mit `GRANT` Zugriff erteilen.

#### F.28.1. Funktionen

```text
pg_get_logical_snapshot_meta(filename text) returns record
```

Liefert Metadaten zu einer logischen Snapshot-Datei im Serververzeichnis `pg_logical/snapshots`. Das Argument `filename` bezeichnet den Namen der Snapshot-Datei. Beispiel:

```sql
postgres=# SELECT * FROM pg_ls_logicalsnapdir();
```

```text
-[ RECORD 1 ]+-----------------------
name         | 0-40796E18.snap
size         | 152
modification | 2024-08-14 16:36:32+00
```

```sql
postgres=# SELECT * FROM pg_get_logical_snapshot_meta('0-40796E18.snap');
```

```text
-[ RECORD 1 ]--------
magic    | 1369563137
checksum | 1028045905
version  | 6
```

```sql
postgres=# SELECT ss.name, meta.*
FROM pg_ls_logicalsnapdir() AS ss,
     pg_get_logical_snapshot_meta(ss.name) AS meta;
```

```text
-[ RECORD 1 ]-------------
name     | 0-40796E18.snap
magic    | 1369563137
checksum | 1028045905
version  | 6
```

Wenn `filename` keiner Snapshot-Datei entspricht, löst die Funktion einen Fehler aus.

```text
pg_get_logical_snapshot_info(filename text) returns record
```

Liefert Informationen zu einer logischen Snapshot-Datei im Serververzeichnis `pg_logical/snapshots`. Das Argument `filename` bezeichnet den Namen der Snapshot-Datei. Beispiel:

```sql
postgres=# SELECT * FROM pg_ls_logicalsnapdir();
```

```text
-[ RECORD 1 ]+-----------------------
name         | 0-40796E18.snap
size         | 152
modification | 2024-08-14 16:36:32+00
```

```sql
postgres=# SELECT * FROM pg_get_logical_snapshot_info('0-40796E18.snap');
```

```text
-[ RECORD 1 ]------------+-----------
state                    | consistent
xmin                     | 751
xmax                     | 751
start_decoding_at        | 0/40796AF8
two_phase_at             | 0/40796AF8
initial_xmin_horizon     | 0
building_full_snapshot   | f
in_slot_creation         | f
last_serialized_snapshot | 0/0
next_phase_at            | 0
committed_count          | 0
committed_xip            |
catchange_count          | 2
catchange_xip            | {751,752}
```

```sql
postgres=# SELECT ss.name, info.*
FROM pg_ls_logicalsnapdir() AS ss,
     pg_get_logical_snapshot_info(ss.name) AS info;
```

```text
-[ RECORD 1 ]------------+----------------
name                     | 0-40796E18.snap
state                    | consistent
xmin                     | 751
xmax                     | 751
start_decoding_at        | 0/40796AF8
two_phase_at             | 0/40796AF8
initial_xmin_horizon     | 0
building_full_snapshot   | f
in_slot_creation         | f
last_serialized_snapshot | 0/0
next_phase_at            | 0
committed_count          | 0
committed_xip            |
catchange_count          | 2
catchange_xip            | {751,752}
```

Wenn `filename` keiner Snapshot-Datei entspricht, löst die Funktion einen Fehler aus.

#### F.28.2. Autor

Bertrand Drouvot <bertranddrouvot.pg@gmail.com>


### F.29. pg_overexplain — zusätzliche Details in EXPLAIN ausgeben

Das Modul `pg_overexplain` erweitert `EXPLAIN` um neue Optionen, die zusätzliche Ausgaben erzeugen. Es ist vor allem zur Unterstützung beim Debugging und bei der Entwicklung des Planners gedacht, nicht für den allgemeinen Gebrauch. Da das Modul interne Details von Planner-Datenstrukturen anzeigt, kann zum Verständnis der Ausgabe ein Blick in den Quellcode nötig sein. Außerdem kann sich die Ausgabe immer dann ändern, wenn sich diese Datenstrukturen ändern.

Zur Verwendung laden Sie das Modul einfach in den Server. Für eine einzelne Sitzung geht das so:

```sql
LOAD 'pg_overexplain';
```

Sie können es auch für manche oder alle Sitzungen vorladen, indem Sie `pg_overexplain` in `session_preload_libraries` oder `shared_preload_libraries` in `postgresql.conf` aufnehmen.

#### F.29.1. EXPLAIN (DEBUG)

Die Option `DEBUG` zeigt verschiedene Informationen aus dem Planbaum an, die normalerweise nicht ausgegeben werden, weil sie nicht von allgemeinem Interesse sind. Für jeden einzelnen Planknoten werden die folgenden Felder angezeigt. Weitere Dokumentation zu diesen Feldern finden Sie bei `Plan` in `nodes/plannodes.h`.

- `Disabled Nodes`: Normales `EXPLAIN` bestimmt, ob ein Knoten deaktiviert ist, indem es prüft, ob die Anzahl deaktivierter Knoten größer ist als die Summe der Zähler für die darunterliegenden Knoten. Diese Option zeigt den rohen Zählerwert.
- `Parallel Safe`: Gibt an, ob ein Knoten eines Planbaums sicher unterhalb eines `Gather`- oder `Gather Merge`-Knotens erscheinen könnte, unabhängig davon, ob er tatsächlich darunter liegt.
- `Plan Node ID`: Eine interne ID-Nummer, die für jeden Knoten im Planbaum eindeutig sein sollte. Sie dient zur Koordination paralleler Abfrageaktivität.
- `extParam` und `allParam`: Informationen darüber, welche nummerierten Parameter diesen Planknoten oder seine Kinder beeinflussen. Im Textmodus werden diese Felder nur angezeigt, wenn sie nicht leere Mengen sind.

Einmal pro Abfrage zeigt `DEBUG` die folgenden Felder an. Weitere Details finden Sie bei `PlannedStmt` in `nodes/plannodes.h`.

- `Command Type`: Zum Beispiel `select` oder `update`.
- `Flags`: Eine kommaseparierte Liste boolescher Strukturmembernamen aus `PlannedStmt`, die auf `true` gesetzt sind. Abgedeckt sind `hasReturning`, `hasModifyingCTE`, `canSetTag`, `transientPlan`, `dependsOnRole` und `parallelModeNeeded`.
- `Subplans Needing Rewind`: Ganzzahlige IDs von Subplänen, die vom Executor möglicherweise zurückgespult werden müssen.
- `Relation OIDs`: OIDs der Relationen, von denen dieser Plan abhängt.
- `Executor Parameter Types`: Typ-OID für jeden Executor-Parameter, etwa wenn ein Nested Loop gewählt wurde und ein Parameter verwendet wird, um einen Wert an einen inneren Index-Scan weiterzugeben. Parameter, die vom Benutzer an ein Prepared Statement übergeben werden, sind nicht enthalten.
- `Parse Location`: Position innerhalb der an den Planner übergebenen Abfragezeichenkette, an der der Text dieser Abfrage gefunden werden kann. In manchen Kontexten kann dies `Unknown` sein; andernfalls etwa `NNN to end` oder `NNN for MMM bytes`.

#### F.29.2. EXPLAIN (RANGE_TABLE)

Die Option `RANGE_TABLE` zeigt Informationen aus dem Planbaum an, die speziell die Range Table der Abfrage betreffen. Range-Table-Einträge entsprechen grob den Elementen der `FROM`-Klausel, allerdings mit zahlreichen Ausnahmen. Beispielsweise können als unnötig erkannte Unterabfragen vollständig aus der Range Table entfernt werden, während Vererbungs-Expansion Range-Table-Einträge für Kindtabellen hinzufügt, die in der Abfrage nicht direkt genannt sind.

Range-Table-Einträge werden im Abfrageplan normalerweise über einen Range-Table-Index (RTI) referenziert. Planknoten, die einen oder mehrere RTIs referenzieren, werden entsprechend markiert, zum Beispiel mit `Scan RTI`, `Nominal RTI`, `Exclude Relation RTI` oder `Append RTIs`.

Zusätzlich kann die Abfrage als Ganzes Listen von Range-Table-Indizes verwalten, die für verschiedene Zwecke benötigt werden. Diese Listen werden einmal pro Abfrage mit passenden Bezeichnungen wie `Unprunable RTIs` oder `Result RTIs` ausgegeben. Im Textmodus werden sie nur angezeigt, wenn sie nicht leer sind.

Am wichtigsten ist, dass `RANGE_TABLE` einen Dump der gesamten Range Table der Abfrage ausgibt. Jeder Range-Table-Eintrag wird mit dem passenden Range-Table-Index, der Art des Eintrags, etwa Relation, Unterabfrage oder Join, sowie mit verschiedenen Feldern des Range-Table-Eintrags beschriftet, die normalerweise nicht Teil der `EXPLAIN`-Ausgabe sind. Manche dieser Felder werden nur für bestimmte Arten von Range-Table-Einträgen angezeigt. Beispielsweise wird `Eref` für alle Typen angezeigt, `CTE Name` aber nur für Einträge vom Typ `cte`.

Weitere Informationen über Range-Table-Einträge finden Sie in der Definition von `RangeTblEntry` in `nodes/plannodes.h`.

#### F.29.3. Autor

Robert Haas <rhaas@postgresql.org>


### F.30. pg_prewarm — Relationsdaten in Puffercaches vorladen

Das Modul `pg_prewarm` bietet eine bequeme Möglichkeit, Relationsdaten entweder in den Betriebssystem-Buffer-Cache oder in den PostgreSQL-Buffer-Cache zu laden. Das Vorwärmen kann manuell mit der Funktion `pg_prewarm()` erfolgen oder automatisch, indem `pg_prewarm` in `shared_preload_libraries` aufgenommen wird. Im zweiten Fall startet das System einen Background Worker, der periodisch den Inhalt der Shared Buffers in einer Datei namens `autoprewarm.blocks` aufzeichnet und diese Blöcke nach einem Neustart mithilfe von zwei Background Workern wieder lädt.

#### F.30.1. Funktionen

```text
pg_prewarm(regclass, mode text default 'buffer', fork text default 'main',
           first_block int8 default null,
           last_block int8 default null) RETURNS int8
```

Das erste Argument ist die vorzuwärmende Relation. Das zweite Argument ist die zu verwendende Vorwärmmethode, die weiter unten beschrieben wird. Das dritte ist der vorzuwärmende Relations-Fork, normalerweise `main`. Das vierte Argument ist die erste vorzuwärmende Blocknummer; `NULL` wird als Synonym für null akzeptiert. Das fünfte Argument ist die letzte vorzuwärmende Blocknummer; `NULL` bedeutet bis zum letzten Block der Relation. Der Rückgabewert ist die Anzahl der vorgewärmten Blöcke.

Es gibt drei verfügbare Vorwärmmethoden:

- `prefetch`: sendet asynchrone Prefetch-Anforderungen an das Betriebssystem, sofern dies unterstützt wird; andernfalls wird ein Fehler ausgelöst.
- `read`: liest den angeforderten Blockbereich. Im Gegensatz zu `prefetch` ist dies synchron und auf allen Plattformen und Builds unterstützt, kann aber langsamer sein.
- `buffer`: liest den angeforderten Blockbereich in den Datenbank-Buffer-Cache.

Beachten Sie, dass bei jeder dieser Methoden der Versuch, mehr Blöcke vorzuwärmen, als im Cache gehalten werden können, wahrscheinlich dazu führt, dass Blöcke mit niedrigeren Nummern verdrängt werden, während Blöcke mit höheren Nummern eingelesen werden. Das gilt für den Betriebssystemcache bei `prefetch` oder `read` und für PostgreSQL bei `buffer`. Vorgewärmte Daten genießen keinen besonderen Schutz vor Cache-Verdrängung; andere Systemaktivität kann die frisch vorgewärmten Blöcke kurz nach dem Lesen wieder verdrängen. Umgekehrt kann das Vorwärmen selbst andere Daten aus dem Cache verdrängen. Aus diesen Gründen ist Vorwärmen typischerweise beim Start am nützlichsten, wenn die Caches weitgehend leer sind.

```text
autoprewarm_start_worker() RETURNS void
```

Startet den Haupt-`autoprewarm`-Worker. Normalerweise geschieht dies automatisch, ist aber nützlich, wenn automatisches Vorwärmen beim Serverstart nicht konfiguriert war und der Worker später gestartet werden soll.

```text
autoprewarm_dump_now() RETURNS int8
```

Aktualisiert `autoprewarm.blocks` sofort. Das kann nützlich sein, wenn der `autoprewarm`-Worker nicht läuft, Sie ihn aber nach dem nächsten Neustart verwenden möchten. Der Rückgabewert ist die Anzahl der in `autoprewarm.blocks` geschriebenen Datensätze.

#### F.30.2. Konfigurationsparameter

`pg_prewarm.autoprewarm` (`boolean`)

Steuert, ob der Server den `autoprewarm`-Worker ausführen soll. Dies ist standardmäßig `on`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`pg_prewarm.autoprewarm_interval` (`integer`)

Das Intervall zwischen Aktualisierungen von `autoprewarm.blocks`. Der Standardwert ist 300 Sekunden. Bei `0` wird die Datei nicht in regelmäßigen Abständen geschrieben, sondern nur beim Herunterfahren des Servers.

Diese Parameter müssen in `postgresql.conf` gesetzt werden. Eine typische Verwendung wäre:

```text
# postgresql.conf
shared_preload_libraries = 'pg_prewarm'

pg_prewarm.autoprewarm = true
pg_prewarm.autoprewarm_interval = 300s
```

#### F.30.3. Autor

Robert Haas <rhaas@postgresql.org>


### F.31. pgrowlocks — Zeilensperrinformationen einer Tabelle anzeigen

Das Modul `pgrowlocks` stellt eine Funktion bereit, die Zeilensperrinformationen für eine angegebene Tabelle anzeigt.

Standardmäßig ist die Verwendung auf Superuser, Rollen mit den Rechten der Rolle `pg_stat_scan_tables` und Benutzer mit `SELECT`-Rechten auf der Tabelle beschränkt.

#### F.31.1. Überblick

```text
pgrowlocks(text) returns setof record
```

Der Parameter ist der Name einer Tabelle. Das Ergebnis ist eine Menge von Datensätzen mit je einer Zeile pro gesperrter Zeile innerhalb der Tabelle. Die Ausgabespalten sind in Tabelle F.21 gezeigt.

**Tabelle F.21. Ausgabespalten von `pgrowlocks`**

| Name | Typ | Beschreibung |
|---|---|---|
| `locked_row` | `tid` | Tuple-ID (TID) der gesperrten Zeile |
| `locker` | `xid` | Transaktions-ID des Sperrenden oder MultiXact-ID bei Multitransaktion; siehe [Abschnitt 67.1](67_Transaktionsverarbeitung.md#671-transaktionen-und-identifikatoren) |
| `multi` | `boolean` | `true`, wenn `locker` eine Multitransaktion ist |
| `xids` | `xid[]` | Transaktions-IDs der Sperrenden, bei Multitransaktion mehr als eine |
| `modes` | `text[]` | Sperrmodi der Sperrenden, bei Multitransaktion mehr als einer; Array aus `For Key Share`, `For Share`, `For No Key Update`, `No Key Update`, `For Update`, `Update` |
| `pids` | `integer[]` | Prozess-IDs der sperrenden Backends, bei Multitransaktion mehr als eine |

`pgrowlocks` nimmt eine `AccessShareLock` auf die Zieltabelle und liest jede Zeile einzeln, um die Zeilensperrinformationen zu sammeln. Für große Tabellen ist das nicht besonders schnell. Beachten Sie:

1. Wenn eine `ACCESS EXCLUSIVE`-Sperre auf der Tabelle gehalten wird, wird `pgrowlocks` blockiert.
2. `pgrowlocks` garantiert keinen in sich konsistenten Snapshot. Während der Ausführung kann eine neue Zeilensperre genommen oder eine alte freigegeben werden.

`pgrowlocks` zeigt nicht den Inhalt der gesperrten Zeilen. Wenn Sie gleichzeitig den Zeileninhalt sehen möchten, können Sie zum Beispiel Folgendes tun:

```sql
SELECT * FROM accounts AS a, pgrowlocks('accounts') AS p
WHERE p.locked_row = a.ctid;
```

Beachten Sie aber, dass eine solche Abfrage sehr ineffizient ist.

#### F.31.2. Beispielausgabe

```sql
=# SELECT * FROM pgrowlocks('t1');
```

```text
locked_row | locker | multi | xids  |      modes      | pids
------------+--------+-------+-------+----------------+--------
(0,1)      |    609 | f     | {609} | {"For Share"} | {3161}
(0,2)      |    609 | f     | {609} | {"For Share"} | {3161}
(0,3)      |    607 | f     | {607} | {"For Update"} | {3107}
(0,4)      |    607 | f     | {607} | {"For Update"} | {3107}
(4 rows)
```

#### F.31.3. Autor

Tatsuo Ishii


### F.32. pg_stat_statements — Statistiken zu SQL-Planung und -Ausführung verfolgen

Das Modul `pg_stat_statements` bietet eine Möglichkeit, Planungs- und Ausführungsstatistiken aller SQL-Anweisungen zu verfolgen, die von einem Server ausgeführt werden.

Das Modul muss durch Aufnahme von `pg_stat_statements` in `shared_preload_libraries` in `postgresql.conf` geladen werden, weil es zusätzlichen Shared Memory benötigt. Zum Hinzufügen oder Entfernen des Moduls ist daher ein Serverneustart nötig. Außerdem muss die Berechnung von Abfragekennungen aktiviert sein. Das geschieht automatisch, wenn `compute_query_id` auf `auto` oder `on` gesetzt ist oder wenn ein Drittanbieter-Modul geladen ist, das Abfragekennungen berechnet.

Wenn `pg_stat_statements` aktiv ist, verfolgt es Statistiken über alle Datenbanken des Servers hinweg. Zum Zugriff auf diese Statistiken und zu ihrer Verwaltung stellt das Modul die Sichten `pg_stat_statements` und `pg_stat_statements_info` sowie die Hilfsfunktionen `pg_stat_statements_reset()` und `pg_stat_statements()` bereit. Diese Objekte sind nicht global verfügbar, sondern können mit `CREATE EXTENSION pg_stat_statements` für eine bestimmte Datenbank aktiviert werden.

#### F.32.1. Die Sicht `pg_stat_statements`

Die vom Modul gesammelten Statistiken werden über die Sicht `pg_stat_statements` verfügbar gemacht. Die Sicht enthält eine Zeile für jede eindeutige Kombination aus Datenbank-ID, Benutzer-ID, Query-ID und der Information, ob es sich um eine Top-Level-Anweisung handelt, bis zur maximalen Zahl unterschiedlicher Anweisungen, die das Modul verfolgen kann. Die Spalten der Sicht sind in Tabelle F.22 dargestellt.

**Tabelle F.22. Spalten von `pg_stat_statements`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `userid` | `oid (referenziert pg_authid.oid)` | OID des Benutzers, der die Anweisung ausgeführt hat |
| `dbid` | `oid (referenziert pg_database.oid)` | OID der Datenbank, in der die Anweisung ausgeführt wurde |
| `toplevel` | `bool` | True, wenn die Abfrage als Top-Level-Anweisung ausgeführt wurde |
| `queryid` | `bigint` | Hash-Code zur Identifikation identischer normalisierter Abfragen |
| `query` | `text` | Text einer repräsentativen Anweisung |
| `plans` | `bigint` | Anzahl der Planungen der Anweisung, falls pg_stat_statements.track_planning aktiv ist |
| `total_plan_time` | `double precision` | Gesamtzeit für die Planung in Millisekunden |
| `min_plan_time` | `double precision` | Minimale Planungszeit in Millisekunden |
| `max_plan_time` | `double precision` | Maximale Planungszeit in Millisekunden |
| `mean_plan_time` | `double precision` | Mittlere Planungszeit in Millisekunden |
| `stddev_plan_time` | `double precision` | Populationsstandardabweichung der Planungszeit |
| `calls` | `bigint` | Anzahl der Ausführungen der Anweisung |
| `total_exec_time` | `double precision` | Gesamtzeit für die Ausführung in Millisekunden |
| `min_exec_time` | `double precision` | Minimale Ausführungszeit in Millisekunden |
| `max_exec_time` | `double precision` | Maximale Ausführungszeit in Millisekunden |
| `mean_exec_time` | `double precision` | Mittlere Ausführungszeit in Millisekunden |
| `stddev_exec_time` | `double precision` | Populationsstandardabweichung der Ausführungszeit |
| `rows` | `bigint` | Gesamtzahl der gelesenen oder betroffenen Zeilen |
| `shared_blks_hit` | `bigint` | Gesamtzahl der Shared-Block-Cache-Hits |
| `shared_blks_read` | `bigint` | Gesamtzahl der gelesenen Shared Blocks |
| `shared_blks_dirtied` | `bigint` | Gesamtzahl der dirty gemachten Shared Blocks |
| `shared_blks_written` | `bigint` | Gesamtzahl der geschriebenen Shared Blocks |
| `local_blks_hit` | `bigint` | Gesamtzahl der Local-Block-Cache-Hits |
| `local_blks_read` | `bigint` | Gesamtzahl der gelesenen Local Blocks |
| `local_blks_dirtied` | `bigint` | Gesamtzahl der dirty gemachten Local Blocks |
| `local_blks_written` | `bigint` | Gesamtzahl der geschriebenen Local Blocks |
| `temp_blks_read` | `bigint` | Gesamtzahl der gelesenen temporären Blöcke |
| `temp_blks_written` | `bigint` | Gesamtzahl der geschriebenen temporären Blöcke |
| `shared_blk_read_time` | `double precision` | Zeit zum Lesen von Shared Blocks in Millisekunden, falls track_io_timing aktiv ist |
| `shared_blk_write_time` | `double precision` | Zeit zum Schreiben von Shared Blocks in Millisekunden, falls track_io_timing aktiv ist |
| `local_blk_read_time` | `double precision` | Zeit zum Lesen von Local Blocks in Millisekunden, falls track_io_timing aktiv ist |
| `local_blk_write_time` | `double precision` | Zeit zum Schreiben von Local Blocks in Millisekunden, falls track_io_timing aktiv ist |
| `temp_blk_read_time` | `double precision` | Zeit zum Lesen temporärer Dateiblöcke in Millisekunden, falls track_io_timing aktiv ist |
| `temp_blk_write_time` | `double precision` | Zeit zum Schreiben temporärer Dateiblöcke in Millisekunden, falls track_io_timing aktiv ist |
| `wal_records` | `bigint` | Gesamtzahl der von der Anweisung erzeugten WAL-Records |
| `wal_fpi` | `bigint` | Gesamtzahl der erzeugten WAL-Full-Page-Images |
| `wal_bytes` | `numeric` | Gesamtmenge des erzeugten WAL in Bytes |
| `wal_buffers_full` | `bigint` | Anzahl der Fälle, in denen die WAL-Buffer voll wurden |
| `jit_functions` | `bigint` | Gesamtzahl der JIT-kompilierten Funktionen |
| `jit_generation_time` | `double precision` | Zeit für das Erzeugen von JIT-Code in Millisekunden |
| `jit_inlining_count` | `bigint` | Anzahl der Inlining-Vorgänge |
| `jit_inlining_time` | `double precision` | Zeit für JIT-Inlining in Millisekunden |
| `jit_optimization_count` | `bigint` | Anzahl der Optimierungsvorgänge |
| `jit_optimization_time` | `double precision` | Zeit für JIT-Optimierung in Millisekunden |
| `jit_emission_count` | `bigint` | Anzahl der Code-Emissionen |
| `jit_emission_time` | `double precision` | Zeit für Code-Emission in Millisekunden |
| `jit_deform_count` | `bigint` | Gesamtzahl der JIT-kompilierten Tuple-Deform-Funktionen |
| `jit_deform_time` | `double precision` | Zeit für JIT-Kompilierung von Tuple-Deform-Funktionen |
| `parallel_workers_to_launch` | `bigint` | Anzahl der geplanten parallelen Worker |
| `parallel_workers_launched` | `bigint` | Anzahl der tatsächlich gestarteten parallelen Worker |
| `stats_since` | `timestamp with time zone` | Zeitpunkt, seit dem Statistiken für diese Anweisung gesammelt werden |
| `minmax_stats_since` | `timestamp with time zone` | Zeitpunkt, seit dem Min/Max-Statistiken für diese Anweisung gesammelt werden |

Aus Sicherheitsgründen dürfen nur Superuser und Rollen mit den Rechten der Rolle `pg_read_all_stats` den SQL-Text und die `queryid` von Abfragen sehen, die von anderen Benutzern ausgeführt wurden. Andere Benutzer können die Statistiken sehen, sofern die Sicht in ihrer Datenbank installiert wurde.

Planbare Abfragen, also `SELECT`, `INSERT`, `UPDATE`, `DELETE` und `MERGE`, sowie Utility-Befehle werden zu einem einzelnen `pg_stat_statements`-Eintrag zusammengefasst, wenn sie gemäß einer internen Hash-Berechnung identische Abfragestrukturen besitzen. Typischerweise gelten zwei Abfragen dafür als gleich, wenn sie semantisch äquivalent sind und sich nur in den Werten literaler Konstanten unterscheiden.

> **Hinweis:**
> Die folgenden Details zur Ersetzung von Konstanten und zu `queryid` gelten nur, wenn `compute_query_id` aktiviert ist. Wenn Sie stattdessen ein externes Modul zur Berechnung von `queryid` verwenden, entnehmen Sie die Details dessen Dokumentation.

Wenn der Wert einer Konstante beim Abgleich mit anderen Abfragen ignoriert wurde, wird die Konstante in der Anzeige von `pg_stat_statements` durch ein Parametersymbol wie `$1` ersetzt. Der übrige Abfragetext stammt von der ersten Abfrage, die den betreffenden `queryid`-Hash-Wert für den `pg_stat_statements`-Eintrag hatte.

Abfragen, auf die Normalisierung angewendet werden kann, können in `pg_stat_statements` dennoch mit Konstantenwerten sichtbar sein, insbesondere wenn Einträge häufig freigegeben werden. Um die Wahrscheinlichkeit zu reduzieren, können Sie `pg_stat_statements.max` erhöhen. Die unten beschriebene Sicht `pg_stat_statements_info` liefert Statistiken über solche Freigaben.

In manchen Fällen können sichtbar unterschiedliche Abfragetexte zu einem einzigen Eintrag zusammengeführt werden. Zusätzlich wird eine Liste von Konstanten auf ein einzelnes Element reduziert, wenn der einzige Unterschied zwischen Abfragen in der Anzahl der Listenelemente besteht; dies wird mit einem auskommentierten Listenhinweis angezeigt:

```sql
=# SELECT pg_stat_statements_reset();
=# SELECT * FROM test WHERE a IN (1, 2, 3, 4, 5, 6, 7);
=# SELECT * FROM test WHERE a IN (1, 2, 3, 4, 5, 6, 7, 8);
=# SELECT query, calls FROM pg_stat_statements
   WHERE query LIKE 'SELECT%';
```

```text
-[ RECORD 1 ]------------------------------
query | SELECT * FROM test WHERE a IN ($1 /*, ... */)
calls | 2
```

Zusätzlich besteht eine geringe Wahrscheinlichkeit, dass Hash-Kollisionen nicht zusammengehörige Abfragen in einem Eintrag zusammenführen. Für Abfragen verschiedener Benutzer oder Datenbanken kann dies jedoch nicht passieren.

Da der `queryid`-Hash-Wert auf der Darstellung nach der Parse-Analyse berechnet wird, ist auch das Gegenteil möglich: Abfragen mit identischem Text können als separate Einträge erscheinen, wenn sie aufgrund unterschiedlicher Faktoren, etwa verschiedener `search_path`-Einstellungen, unterschiedliche Bedeutungen haben.

Verbraucher von `pg_stat_statements` können `queryid`, gegebenenfalls zusammen mit `dbid` und `userid`, als stabileren Bezeichner eines Eintrags verwenden als den Abfragetext. Es gibt jedoch nur begrenzte Garantien für die Stabilität des `queryid`-Hash-Werts. Er hängt unter anderem von internen Objektkennungen ab, die in der Darstellung nach der Parse-Analyse vorkommen. Daher können scheinbar identische Abfragen als verschieden gelten, wenn sie etwa eine Funktion referenzieren, die zwischen den Ausführungen gelöscht und neu erzeugt wurde. Umgekehrt können scheinbar identische Abfragen trotz neu erzeugter Tabelle als gleich gelten. Unterschiedliche Tabellenaliasnamen können wiederum zu unterschiedlichen Einträgen führen. Auch Maschinenarchitektur und Plattformdetails können den Hash beeinflussen, und `queryid` sollte nicht als stabil über PostgreSQL-Hauptversionen hinweg angenommen werden.

Zwei Server, die per physischem WAL-Replay replizieren, sollten für dieselbe Abfrage identische `queryid`-Werte haben. Logische Replikation garantiert dies nicht. Im Zweifel empfiehlt sich ein direkter Test.

Im Allgemeinen kann angenommen werden, dass `queryid`-Werte zwischen Minor-Versionen stabil sind, sofern dieselbe Maschinenarchitektur verwendet wird und die Katalogmetadaten übereinstimmen. Die Kompatibilität wird zwischen Minor-Versionen nur als letztes Mittel gebrochen.

Die Parametersymbole, die Konstanten in repräsentativen Abfragetexten ersetzen, beginnen mit der nächsten Nummer nach dem höchsten `$n`-Parameter im ursprünglichen Abfragetext oder mit `$1`, wenn es keinen gab. In manchen Fällen können versteckte Parametersymbole diese Nummerierung beeinflussen. PL/pgSQL verwendet zum Beispiel versteckte Parametersymbole, um Werte lokaler Funktionsvariablen in Abfragen einzufügen; eine PL/pgSQL-Anweisung wie `SELECT i + 1 INTO j` könnte daher als repräsentativen Text `SELECT i + $2` haben.

Die repräsentativen Abfragetexte werden in einer externen Datei gespeichert und verbrauchen keinen Shared Memory. Auch sehr lange Abfragetexte können daher gespeichert werden. Wenn viele lange Texte gesammelt werden, kann diese Datei allerdings unhandlich groß werden. Als Wiederherstellungsmaßnahme kann `pg_stat_statements` die Abfragetexte verwerfen; bestehende Einträge zeigen dann `NULL` in den `query`-Feldern, während die Statistiken zu jeder `queryid` erhalten bleiben. In diesem Fall sollten Sie eine Verringerung von `pg_stat_statements.max` erwägen.

`plans` und `calls` müssen nicht übereinstimmen, weil Planungs- und Ausführungsstatistiken jeweils am Ende ihrer Phase und nur für erfolgreiche Operationen aktualisiert werden. Wenn eine Anweisung erfolgreich geplant wird, aber während der Ausführung fehlschlägt, werden nur die Planungsstatistiken aktualisiert. Wird die Planung wegen eines gecachten Plans übersprungen, werden nur die Ausführungsstatistiken aktualisiert.

#### F.32.2. Die Sicht `pg_stat_statements_info`

Die Statistiken des Moduls `pg_stat_statements` selbst werden über die Sicht `pg_stat_statements_info` verfügbar gemacht. Diese Sicht enthält nur eine einzelne Zeile. Die Spalten sind in Tabelle F.23 dargestellt.

**Tabelle F.23. Spalten von `pg_stat_statements_info`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `dealloc` | `bigint` | Gesamtzahl der Fälle, in denen Einträge zu den am seltensten ausgeführten Anweisungen freigegeben wurden, weil mehr unterschiedliche Anweisungen als `pg_stat_statements.max` beobachtet wurden |
| `stats_reset` | `timestamp with time zone` | Zeitpunkt, zu dem alle Statistiken in der Sicht `pg_stat_statements` zuletzt zurückgesetzt wurden |

#### F.32.3. Funktionen

```text
pg_stat_statements_reset(userid Oid, dbid Oid, queryid bigint, minmax_only boolean) returns timestamp with time zone
```

`pg_stat_statements_reset()` verwirft die bisher von `pg_stat_statements` gesammelten Statistiken, die zu `userid`, `dbid` und `queryid` passen. Nicht angegebene Parameter erhalten den Standardwert `0` (ungültig); zurückgesetzt werden dann Statistiken, die zu den übrigen Parametern passen. Sind keine Parameter angegeben oder sind alle angegebenen Parameter `0`, werden alle Statistiken verworfen. Werden alle Statistiken der Sicht `pg_stat_statements` verworfen, werden auch die Statistiken in `pg_stat_statements_info` zurückgesetzt. Ist `minmax_only` `true`, werden nur die Minimal- und Maximalwerte für Planungs- und Ausführungszeit zurückgesetzt. Standardmäßig ist `minmax_only` `false`. Die Funktion gibt den Zeitpunkt des Resets zurück. Standardmäßig kann sie nur von Superusern ausgeführt werden; Zugriff kann mit `GRANT` erteilt werden.

```text
pg_stat_statements(showtext boolean) returns setof record
```

Die Sicht `pg_stat_statements` ist über eine gleichnamige Funktion definiert. Clients können diese Funktion direkt aufrufen und mit `showtext := false` den Abfragetext auslassen; das OUT-Argument zur Spalte `query` gibt dann `NULL` zurück. Dies unterstützt externe Werkzeuge, die den Aufwand vermeiden möchten, wiederholt Abfragetexte unbestimmter Länge abzurufen. Solche Werkzeuge können den ersten beobachteten Abfragetext eines Eintrags selbst zwischenspeichern und Texte nur bei Bedarf abrufen. Da der Server Abfragetexte in einer Datei speichert, kann dieser Ansatz wiederholte physische I/O beim Untersuchen von `pg_stat_statements` reduzieren.

#### F.32.4. Konfigurationsparameter

`pg_stat_statements.max` (`integer`)

Maximale Anzahl von Anweisungen, die das Modul verfolgt, also maximale Zeilenzahl der Sicht `pg_stat_statements`. Werden mehr unterschiedliche Anweisungen beobachtet, werden Informationen zu den am seltensten ausgeführten Anweisungen verworfen. Wie oft dies geschehen ist, ist in `pg_stat_statements_info` sichtbar. Standardwert ist `5000`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`pg_stat_statements.track` (`enum`)

Steuert, welche Anweisungen gezählt werden. `top` verfolgt Top-Level-Anweisungen, `all` zusätzlich verschachtelte Anweisungen, etwa aus Funktionen heraus, und `none` deaktiviert die Statistik-Erfassung. Standardwert ist `top`. Nur Superuser können diese Einstellung ändern.

`pg_stat_statements.track_utility` (`boolean`)

Steuert, ob Utility-Befehle verfolgt werden. Utility-Befehle sind alle anderen Befehle als `SELECT`, `INSERT`, `UPDATE`, `DELETE` und `MERGE`. Standardwert ist `on`. Nur Superuser können diese Einstellung ändern.

`pg_stat_statements.track_planning` (`boolean`)

Steuert, ob Planungsvorgänge und deren Dauer verfolgt werden. Das Aktivieren dieses Parameters kann spürbare Performance-Kosten verursachen, insbesondere wenn viele konkurrierende Verbindungen Anweisungen mit identischer Abfragestruktur ausführen und dieselben wenigen `pg_stat_statements`-Einträge aktualisieren. Standardwert ist `off`. Nur Superuser können diese Einstellung ändern.

`pg_stat_statements.save` (`boolean`)

Gibt an, ob Anweisungsstatistiken über Server-Shutdowns hinweg gespeichert werden. Bei `off` werden Statistiken beim Shutdown nicht gespeichert und beim Serverstart nicht neu geladen. Standardwert ist `on`. Dieser Parameter kann nur in `postgresql.conf` oder auf der Serverbefehlszeile gesetzt werden.

Das Modul benötigt zusätzlichen Shared Memory proportional zu `pg_stat_statements.max`. Dieser Speicher wird verbraucht, sobald das Modul geladen ist, selbst wenn `pg_stat_statements.track` auf `none` gesetzt ist.

Diese Parameter müssen in `postgresql.conf` gesetzt werden. Eine typische Verwendung wäre:

```text
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'

compute_query_id = on
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

#### F.32.5. Beispielausgabe

```sql
bench=# SELECT pg_stat_statements_reset();

$ pgbench -i bench
$ pgbench -c10 -t300 bench

bench=# \x
bench=# SELECT query, calls, total_exec_time, rows,
       100.0 * shared_blks_hit /
       nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

```text
-[ RECORD 1 ]---+----------------------------------------------------
query           | UPDATE pgbench_branches SET bbalance = bbalance + $1 WHERE bid = $2
calls           | 3000
total_exec_time | 25565.855387
rows            | 3000
hit_percent     | 100.0000000000000000
...
```

Ein einzelner Eintrag kann anhand seiner `queryid` zurückgesetzt werden:

```sql
bench=# SELECT pg_stat_statements_reset(0,0,s.queryid)
FROM pg_stat_statements AS s
WHERE s.query = 'UPDATE pgbench_branches SET bbalance = bbalance + $1 WHERE bid = $2';
```

Danach verschwindet dieser Eintrag aus der Spitzengruppe der Statistik. Ein vollständiger Reset erfolgt mit:

```sql
bench=# SELECT pg_stat_statements_reset(0,0,0);
```

#### F.32.6. Autoren

Takahiro Itagaki <itagaki.takahiro@oss.ntt.co.jp>. Query-Normalisierung ergänzt von Peter Geoghegan <peter@2ndquadrant.com>.


### F.33. pgstattuple — Statistiken auf Tuple-Ebene ermitteln

Das Modul `pgstattuple` stellt verschiedene Funktionen bereit, um Statistiken auf Tuple-Ebene zu ermitteln.

Da diese Funktionen detaillierte Informationen auf Seitenebene zurückgeben, ist der Zugriff standardmäßig eingeschränkt. Standardmäßig besitzt nur die Rolle `pg_stat_scan_tables` das `EXECUTE`-Recht. Superuser umgehen diese Einschränkung natürlich. Nach der Installation der Erweiterung können Benutzer mit `GRANT` die Rechte der Funktionen ändern, um anderen die Ausführung zu erlauben. Oft ist es jedoch vorzuziehen, diese Benutzer stattdessen der Rolle `pg_stat_scan_tables` hinzuzufügen.

#### F.33.1. Funktionen

```text
pgstattuple(regclass) returns record
```

`pgstattuple` gibt die physische Länge einer Relation, den Anteil „toter“ Tupel und weitere Informationen zurück. Das kann helfen zu entscheiden, ob `VACUUM` nötig ist. Das Argument ist der Name der Zielrelation, optional schemaqualifiziert, oder ihre OID. Beispiel:

```sql
test=> SELECT * FROM pgstattuple('pg_catalog.pg_proc');
```

```text
-[ RECORD 1 ]------+-------
table_len          | 458752
tuple_count        | 1470
tuple_len          | 438896
tuple_percent      | 95.67
dead_tuple_count   | 11
dead_tuple_len     | 3157
dead_tuple_percent | 0.69
free_space         | 8932
free_percent       | 1.95
```

Die Ausgabespalten sind in Tabelle F.24 beschrieben.

**Tabelle F.24. Ausgabespalten von `pgstattuple`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `table_len` | `bigint` | Physische Relationslänge in Bytes |
| `tuple_count` | `bigint` | Anzahl lebender Tupel |
| `tuple_len` | `bigint` | Gesamtlänge lebender Tupel in Bytes |
| `tuple_percent` | `float8` | Prozentualer Anteil lebender Tupel |
| `dead_tuple_count` | `bigint` | Anzahl toter Tupel |
| `dead_tuple_len` | `bigint` | Gesamtlänge toter Tupel in Bytes |
| `dead_tuple_percent` | `float8` | Prozentualer Anteil toter Tupel |
| `free_space` | `bigint` | Gesamter freier Speicherplatz in Bytes |
| `free_percent` | `float8` | Prozentualer Anteil freien Speicherplatzes |

> **Hinweis:**
> `table_len` ist immer größer als die Summe aus `tuple_len`, `dead_tuple_len` und `free_space`. Die Differenz entsteht durch festen Seiten-Overhead, die seitenspezifische Zeigertabelle auf Tupel und Padding zur korrekten Ausrichtung von Tupeln.

`pgstattuple` nimmt nur eine Lesesperre auf die Relation. Die Ergebnisse spiegeln daher keinen momentanen Snapshot wider; gleichzeitige Aktualisierungen beeinflussen sie.

`pgstattuple` betrachtet ein Tupel als „tot“, wenn `HeapTupleSatisfiesDirty` `false` zurückgibt.

```text
pgstattuple(text) returns record
```

Dies entspricht `pgstattuple(regclass)`, nur dass die Zielrelation als `text` angegeben wird. Diese Funktion bleibt bisher aus Gründen der Abwärtskompatibilität erhalten und wird in einer zukünftigen Version veraltet sein.

```text
pgstatindex(regclass) returns record
```

`pgstatindex` gibt einen Datensatz mit Informationen über einen B-Tree-Index zurück. Beispiel:

```sql
test=> SELECT * FROM pgstatindex('pg_cast_oid_index');
```

```text
-[ RECORD 1 ]------+------
version            | 2
tree_level         | 0
index_size         | 16384
root_block_no      | 1
internal_pages     | 0
leaf_pages         | 1
empty_pages        | 0
deleted_pages      | 0
avg_leaf_density   | 54.27
leaf_fragmentation | 0
```

Ausgabespalten:

| Spalte | Typ | Beschreibung |
|---|---|---|
| `version` | `integer` | B-Tree-Versionsnummer |
| `tree_level` | `integer` | Baumebene der Root-Seite |
| `index_size` | `bigint` | Gesamte Indexgröße in Bytes |
| `root_block_no` | `bigint` | Position der Root-Seite, null falls keine vorhanden ist |
| `internal_pages` | `bigint` | Anzahl interner Seiten oberer Ebenen |
| `leaf_pages` | `bigint` | Anzahl Blattseiten |
| `empty_pages` | `bigint` | Anzahl leerer Seiten |
| `deleted_pages` | `bigint` | Anzahl gelöschter Seiten |
| `avg_leaf_density` | `float8` | Durchschnittliche Dichte der Blattseiten |
| `leaf_fragmentation` | `float8` | Fragmentierung der Blattseiten |

Die gemeldete `index_size` entspricht normalerweise einer Seite mehr, als durch `internal_pages + leaf_pages + empty_pages + deleted_pages` erklärt wird, weil auch die Metaseite des Index enthalten ist.

Wie bei `pgstattuple` werden die Ergebnisse seitenweise akkumuliert und sollten nicht als momentaner Snapshot des gesamten Index verstanden werden.

```text
pgstatindex(text) returns record
```

Dies entspricht `pgstatindex(regclass)`, nur dass der Zielindex als `text` angegeben wird. Diese Funktion bleibt bisher aus Gründen der Abwärtskompatibilität erhalten und wird in einer zukünftigen Version veraltet sein.

```text
pgstatginindex(regclass) returns record
```

`pgstatginindex` gibt Informationen über einen GIN-Index zurück. Beispiel:

```sql
test=> SELECT * FROM pgstatginindex('test_gin_index');
```

```text
-[ RECORD 1 ]--+--
version        | 1
pending_pages  | 0
pending_tuples | 0
```

Ausgabespalten:

| Spalte | Typ | Beschreibung |
|---|---|---|
| `version` | `integer` | GIN-Versionsnummer |
| `pending_pages` | `integer` | Anzahl der Seiten in der Pending List |
| `pending_tuples` | `bigint` | Anzahl der Tupel in der Pending List |

```text
pgstathashindex(regclass) returns record
```

`pgstathashindex` gibt Informationen über einen Hash-Index zurück. Beispiel:

```sql
test=> SELECT * FROM pgstathashindex('con_hash_index');
```

```text
-[ RECORD 1 ]--+-----------------
version        | 4
bucket_pages   | 33081
overflow_pages | 0
bitmap_pages   | 1
unused_pages   | 32455
live_items     | 10204006
dead_items     | 0
free_percent   | 61.8005949100872
```

Ausgabespalten:

| Spalte | Typ | Beschreibung |
|---|---|---|
| `version` | `integer` | Hash-Versionsnummer |
| `bucket_pages` | `bigint` | Anzahl Bucket-Seiten |
| `overflow_pages` | `bigint` | Anzahl Overflow-Seiten |
| `bitmap_pages` | `bigint` | Anzahl Bitmap-Seiten |
| `unused_pages` | `bigint` | Anzahl unbenutzter Seiten |
| `live_items` | `bigint` | Anzahl lebender Tupel |
| `dead_tuples` | `bigint` | Anzahl toter Tupel |
| `free_percent` | `float` | Prozentualer Anteil freien Speicherplatzes |

```text
pg_relpages(regclass) returns bigint
```

`pg_relpages` gibt die Anzahl der Seiten in der Relation zurück.

```text
pg_relpages(text) returns bigint
```

Dies entspricht `pg_relpages(regclass)`, nur dass die Zielrelation als `text` angegeben wird. Diese Funktion bleibt bisher aus Gründen der Abwärtskompatibilität erhalten und wird in einer zukünftigen Version veraltet sein.

```text
pgstattuple_approx(regclass) returns record
```

`pgstattuple_approx` ist eine schnellere Alternative zu `pgstattuple`, die Näherungswerte zurückgibt. Das Argument ist der Name oder die OID der Zielrelation. Beispiel:

```sql
test=> SELECT * FROM pgstattuple_approx('pg_catalog.pg_proc'::regclass);
```

```text
-[ RECORD 1 ]--------+-------
table_len            | 573440
scanned_percent      | 2
approx_tuple_count   | 2740
approx_tuple_len     | 561210
approx_tuple_percent | 97.87
dead_tuple_count     | 0
dead_tuple_len       | 0
dead_tuple_percent   | 0
approx_free_space    | 11996
approx_free_percent  | 2.09
```

Die Ausgabespalten sind in Tabelle F.25 beschrieben.

**Tabelle F.25. Ausgabespalten von `pgstattuple_approx`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `table_len` | `bigint` | Physische Relationslänge in Bytes, exakt |
| `scanned_percent` | `float8` | Prozentualer Anteil der Tabelle, der gescannt wurde |
| `approx_tuple_count` | `bigint` | Anzahl lebender Tupel, geschätzt |
| `approx_tuple_len` | `bigint` | Gesamtlänge lebender Tupel in Bytes, geschätzt |
| `approx_tuple_percent` | `float8` | Prozentualer Anteil lebender Tupel |
| `dead_tuple_count` | `bigint` | Anzahl toter Tupel, exakt |
| `dead_tuple_len` | `bigint` | Gesamtlänge toter Tupel in Bytes, exakt |
| `dead_tuple_percent` | `float8` | Prozentualer Anteil toter Tupel |
| `approx_free_space` | `bigint` | Gesamter freier Speicherplatz in Bytes, geschätzt |
| `approx_free_percent` | `float8` | Prozentualer Anteil freien Speicherplatzes |

Während `pgstattuple` immer einen vollständigen Tabellenscan ausführt und exakte Zählungen lebender und toter Tupel sowie freien Speicherplatzes liefert, versucht `pgstattuple_approx`, den vollständigen Scan zu vermeiden. Es liefert exakte Statistiken über tote Tupel und Näherungswerte für Anzahl und Größe lebender Tupel sowie freien Speicherplatz.

Dazu überspringt es Seiten, die laut Visibility Map nur sichtbare Tupel enthalten. Für solche Seiten wird der freie Speicherplatz aus der Free Space Map abgeleitet, und der restliche Platz wird als von lebenden Tupeln belegt angenommen. Seiten, die nicht übersprungen werden können, werden Tupel für Tupel gescannt; am Ende wird die Gesamtzahl lebender Tupel ähnlich geschätzt wie `VACUUM` `pg_class.reltuples` schätzt.

In der obigen Ausgabe stimmen die Werte zum freien Speicherplatz möglicherweise nicht exakt mit der Ausgabe von `pgstattuple` überein, weil die Free Space Map zwar einen konkreten Wert liefert, aber keine bytegenaue Genauigkeit garantiert.

#### F.33.2. Autoren

Tatsuo Ishii, Satoshi Nagayasu und Abhijit Menon-Sen


### F.34. pg_surgery — Low-Level-Eingriffe an Relationsdaten durchführen

Das Modul `pg_surgery` stellt verschiedene Funktionen bereit, um Eingriffe an einer beschädigten Relation vorzunehmen. Diese Funktionen sind absichtlich unsicher, und ihre Verwendung kann die Datenbank beschädigen oder weiter beschädigen. Sie können zum Beispiel leicht dazu verwendet werden, eine Tabelle inkonsistent zu ihren eigenen Indizes zu machen, `UNIQUE`- oder `FOREIGN KEY`-Verletzungen zu erzeugen oder Tupel sichtbar zu machen, deren Lesen einen Serverabsturz auslöst. Sie sollten nur mit großer Vorsicht und als letztes Mittel verwendet werden.

#### F.34.1. Funktionen

```text
heap_force_kill(regclass, tid[]) returns void
```

`heap_force_kill` markiert „verwendete“ Line Pointer als „dead“, ohne die Tupel zu untersuchen. Gedacht ist die Funktion dazu, Tupel zwangsweise zu entfernen, die sonst nicht zugänglich sind. Beispiel:

```sql
test=> SELECT * FROM t1 WHERE ctid = '(0, 1)';
```

```text
ERROR: could not access status of transaction 4007513275
DETAIL: Could not open file "pg_xact/0EED": No such file or directory.
```

```sql
test=# SELECT heap_force_kill('t1'::regclass, ARRAY['(0,1)']::tid[]);
```

```text
heap_force_kill
-----------------

(1 row)
```

```sql
test=# SELECT * FROM t1 WHERE ctid = '(0, 1)';
```

```text
(0 rows)
```

```text
heap_force_freeze(regclass, tid[]) returns void
```

`heap_force_freeze` markiert Tupel als eingefroren, ohne die Tupeldaten zu untersuchen. Gedacht ist die Funktion dazu, Tupel zugänglich zu machen, die wegen beschädigter Sichtbarkeitsinformationen unzugänglich sind oder verhindern, dass die Tabelle erfolgreich per `VACUUM` bearbeitet wird. Beispiel:

```sql
test=> VACUUM t1;
```

```text
ERROR: found xmin 507 from before relfrozenxid 515
CONTEXT: while scanning block 0 of relation "public.t1"
```

```sql
test=# SELECT ctid FROM t1 WHERE xmin = 507;
```

```text
ctid
-------
(0,3)
(1 row)
```

```sql
test=# SELECT heap_force_freeze('t1'::regclass, ARRAY['(0,3)']::tid[]);
```

```text
heap_force_freeze
-------------------

(1 row)
```

```sql
test=# SELECT ctid FROM t1 WHERE xmin = 2;
```

```text
ctid
-------
(0,3)
(1 row)
```

#### F.34.2. Autoren

Ashutosh Sharma <ashu.coek88@gmail.com>


### F.35. pg_trgm — Textähnlichkeit mit Trigramm-Abgleich

Das Modul `pg_trgm` stellt Funktionen und Operatoren bereit, um die Ähnlichkeit alphanumerischer Texte anhand von Trigramm-Abgleich zu bestimmen. Außerdem enthält es Index-Operator-Klassen, die schnelle Suchen nach ähnlichen Zeichenketten unterstützen.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

#### F.35.1. Trigramm- oder Trigraph-Konzepte

Ein Trigramm ist eine Gruppe von drei aufeinanderfolgenden Zeichen aus einer Zeichenkette. Die Ähnlichkeit zweier Zeichenketten lässt sich messen, indem man zählt, wie viele Trigramme sie gemeinsam haben. Diese einfache Idee erweist sich in vielen natürlichen Sprachen als sehr wirksam zur Messung der Wortähnlichkeit.

> **Hinweis:**
> `pg_trgm` ignoriert beim Extrahieren von Trigrammen Nicht-Wort-Zeichen, also nicht alphanumerische Zeichen. Jedes Wort wird so behandelt, als hätte es zwei vorangestellte Leerzeichen und ein angehängtes Leerzeichen. Die Trigramme der Zeichenkette `cat` sind zum Beispiel `"  c"`, `" ca"`, `"cat"` und `"at "`. Die Trigramme der Zeichenkette `foo|bar` sind `"  f"`, `" fo"`, `"foo"`, `"oo "`, `"  b"`, `" ba"`, `"bar"` und `"ar "`.

#### F.35.2. Funktionen und Operatoren

Die vom Modul `pg_trgm` bereitgestellten Funktionen sind in Tabelle F.26 gezeigt, die Operatoren in Tabelle F.27.

**Tabelle F.26. Funktionen von `pg_trgm`**

| Funktion | Beschreibung |
|---|---|
| `similarity(text, text) → real` | Gibt eine Zahl zurück, die angibt, wie ähnlich die beiden Argumente sind. Der Ergebnisbereich reicht von `0` für völlig unähnlich bis `1` für identisch. |
| `show_trgm(text) → text[]` | Gibt ein Array aller Trigramme in der angegebenen Zeichenkette zurück. In der Praxis ist dies vor allem zum Debugging nützlich. |
| `word_similarity(text, text) → real` | Gibt die größte Ähnlichkeit zwischen der Trigramm-Menge der ersten Zeichenkette und einem zusammenhängenden Abschnitt einer geordneten Trigramm-Menge der zweiten Zeichenkette zurück. |
| `strict_word_similarity(text, text) → real` | Wie `word_similarity`, erzwingt aber, dass Abschnittsgrenzen Wortgrenzen entsprechen. Da es keine wortübergreifenden Trigramme gibt, ist dies die größte Ähnlichkeit zwischen der ersten Zeichenkette und einem zusammenhängenden Wortabschnitt der zweiten Zeichenkette. |
| `show_limit() → real` | Gibt den aktuellen Ähnlichkeitsschwellwert des Operators `%` zurück. Veraltet; verwenden Sie stattdessen `SHOW pg_trgm.similarity_threshold`. |
| `set_limit(real) → real` | Setzt den aktuellen Ähnlichkeitsschwellwert für `%`. Der Wert muss zwischen `0` und `1` liegen, Standard ist `0.3`. Veraltet; verwenden Sie stattdessen `SET pg_trgm.similarity_threshold`. |

Beispiel:

```sql
SELECT word_similarity('word', 'two words');
```

```text
 word_similarity
-----------------
             0.8
(1 row)
```

In der ersten Zeichenkette ist die Trigramm-Menge `{ "  w", " wo", "wor", "ord", "rd " }`. In der zweiten Zeichenkette ist die geordnete Trigramm-Menge `{ "  t", " tw", "two", "wo ", "  w", " wo", "wor", "ord", "rds", "ds " }`. Der ähnlichste Abschnitt der geordneten Trigramm-Menge der zweiten Zeichenkette ist `{ "  w", " wo", "wor", "ord" }`, und die Ähnlichkeit beträgt `0.8`.

Diese Funktion liefert einen Wert, der näherungsweise als größte Ähnlichkeit zwischen der ersten Zeichenkette und irgendeinem Teilstring der zweiten Zeichenkette verstanden werden kann. Allerdings fügt sie an den Grenzen des Abschnitts kein Padding hinzu. Zusätzliche Zeichen in der zweiten Zeichenkette werden daher nicht berücksichtigt, außer bei nicht passenden Wortgrenzen.

Gleichzeitig wählt `strict_word_similarity` einen Wortabschnitt in der zweiten Zeichenkette. Im obigen Beispiel würde `strict_word_similarity` den Abschnitt des einzelnen Worts `words` wählen, dessen Trigramm-Menge `{ "  w", " wo", "wor", "ord", "rds", "ds " }` ist.

```sql
SELECT strict_word_similarity('word', 'two words'), similarity('word', 'words');
```

```text
 strict_word_similarity | similarity
------------------------+------------
               0.571429 |   0.571429
(1 row)
```

`strict_word_similarity` ist daher nützlich, um Ähnlichkeit zu ganzen Wörtern zu finden, während `word_similarity` besser für Ähnlichkeit zu Wortteilen geeignet ist.

**Tabelle F.27. Operatoren von `pg_trgm`**

| Operator | Beschreibung |
|---|---|
| `text % text → boolean` | Gibt `true` zurück, wenn die Argumente eine Ähnlichkeit oberhalb des aktuellen Schwellwerts `pg_trgm.similarity_threshold` haben. |
| `text <% text → boolean` | Gibt `true` zurück, wenn die Ähnlichkeit zwischen der Trigramm-Menge des ersten Arguments und einem zusammenhängenden Abschnitt der geordneten Trigramm-Menge des zweiten Arguments oberhalb von `pg_trgm.word_similarity_threshold` liegt. |
| `text %> text → boolean` | Kommutator des Operators `<%`. |
| `text <<% text → boolean` | Gibt `true` zurück, wenn das zweite Argument einen zusammenhängenden Abschnitt einer geordneten Trigramm-Menge mit Wortgrenzen besitzt, dessen Ähnlichkeit zur Trigramm-Menge des ersten Arguments oberhalb von `pg_trgm.strict_word_similarity_threshold` liegt. |
| `text %>> text → boolean` | Kommutator des Operators `<<%`. |
| `text <-> text → real` | Gibt die „Distanz“ zwischen den Argumenten zurück, also `1 - similarity()`. |
| `text <<-> text → real` | Gibt `1 - word_similarity()` zurück. |
| `text <->> text → real` | Kommutator des Operators `<<->`. |
| `text <<<-> text → real` | Gibt `1 - strict_word_similarity()` zurück. |
| `text <->>> text → real` | Kommutator des Operators `<<<->`. |

#### F.35.3. GUC-Parameter

`pg_trgm.similarity_threshold` (`real`)

Setzt den aktuellen Ähnlichkeitsschwellwert für den Operator `%`. Der Wert muss zwischen `0` und `1` liegen; Standard ist `0.3`.

`pg_trgm.word_similarity_threshold` (`real`)

Setzt den aktuellen Wortähnlichkeitsschwellwert für die Operatoren `<%` und `%>`. Der Wert muss zwischen `0` und `1` liegen; Standard ist `0.6`.

`pg_trgm.strict_word_similarity_threshold` (`real`)

Setzt den aktuellen strikten Wortähnlichkeitsschwellwert für die Operatoren `<<%` und `%>>`. Der Wert muss zwischen `0` und `1` liegen; Standard ist `0.5`.

#### F.35.4. Indexunterstützung

Das Modul `pg_trgm` stellt GiST- und GIN-Index-Operator-Klassen bereit, mit denen Sie einen Index auf einer `text`-Spalte für sehr schnelle Ähnlichkeitssuchen anlegen können. Diese Indextypen unterstützen die oben beschriebenen Ähnlichkeitsoperatoren sowie trigrammbasierte Indexsuchen für `LIKE`, `ILIKE`, `~`, `~*` und `=`. In einem Standard-Build von `pg_trgm` sind Ähnlichkeitsvergleiche nicht case-sensitiv. Ungleichheitsoperatoren werden nicht unterstützt. Beachten Sie, dass diese Indizes für Gleichheitsvergleiche möglicherweise weniger effizient sind als reguläre B-Tree-Indizes.

Beispiel:

```sql
CREATE TABLE test_trgm (t text);
CREATE INDEX trgm_idx ON test_trgm USING GIST (t gist_trgm_ops);
```

oder:

```sql
CREATE INDEX trgm_idx ON test_trgm USING GIN (t gin_trgm_ops);
```

Die GiST-Operator-Klasse `gist_trgm_ops` approximiert eine Menge von Trigrammen als Bitmap-Signatur. Ihr optionaler Integer-Parameter `siglen` bestimmt die Signaturlänge in Bytes. Der Standardwert beträgt 12 Bytes. Gültige Signaturlängen liegen zwischen 1 und 2024 Bytes. Längere Signaturen führen zu präziseren Suchen, also zu einem kleineren gescannten Indexanteil und weniger Heap-Seiten, auf Kosten eines größeren Index.

Beispiel für einen Index mit 32 Byte Signaturlänge:

```sql
CREATE INDEX trgm_idx ON test_trgm USING GIST (t gist_trgm_ops(siglen=32));
```

Damit existiert ein Index auf der Spalte `t`, der für Ähnlichkeitssuchen verwendet werden kann. Eine typische Abfrage lautet:

```sql
SELECT t, similarity(t, 'word') AS sml
FROM test_trgm
WHERE t % 'word'
ORDER BY sml DESC, t;
```

Dies gibt alle Werte der Textspalte zurück, die `word` ausreichend ähnlich sind, sortiert vom besten zum schlechtesten Treffer. Der Index macht diese Operation auch auf sehr großen Datenmengen schnell.

Eine Variante ist:

```sql
SELECT t, t <-> 'word' AS dist
FROM test_trgm
ORDER BY dist LIMIT 10;
```

Dies kann von GiST-Indizes recht effizient umgesetzt werden, nicht aber von GIN-Indizes. Wenn nur wenige nächste Treffer benötigt werden, ist diese Formulierung normalerweise schneller als die erste.

Sie können einen Index auf `t` auch für Wortähnlichkeit oder strikte Wortähnlichkeit verwenden. Typische Abfragen sind:

```sql
SELECT t, word_similarity('word', t) AS sml
FROM test_trgm
WHERE 'word' <% t
ORDER BY sml DESC, t;
```

und:

```sql
SELECT t, strict_word_similarity('word', t) AS sml
FROM test_trgm
WHERE 'word' <<% t
ORDER BY sml DESC, t;
```

Dies gibt alle Werte der Textspalte zurück, für die es in der entsprechenden geordneten Trigramm-Menge einen zusammenhängenden Abschnitt gibt, der der Trigramm-Menge von `word` ausreichend ähnlich ist, sortiert vom besten zum schlechtesten Treffer.

Mögliche Varianten sind:

```sql
SELECT t, 'word' <<-> t AS dist
FROM test_trgm
ORDER BY dist LIMIT 10;
```

und:

```sql
SELECT t, 'word' <<<-> t AS dist
FROM test_trgm
ORDER BY dist LIMIT 10;
```

Auch dies kann von GiST-Indizes recht effizient umgesetzt werden, nicht aber von GIN-Indizes.

Seit PostgreSQL 9.1 unterstützen diese Indextypen auch Indexsuchen für `LIKE` und `ILIKE`, zum Beispiel:

```sql
SELECT * FROM test_trgm WHERE t LIKE '%foo%bar';
```

Die Indexsuche extrahiert Trigramme aus der Suchzeichenkette und schlägt sie im Index nach. Je mehr Trigramme die Suchzeichenkette enthält, desto wirksamer ist die Indexsuche. Anders als bei B-Tree-basierten Suchen muss die Suchzeichenkette nicht links verankert sein.

Seit PostgreSQL 9.3 unterstützen diese Indextypen auch Indexsuchen für reguläre Ausdrücke mit den Operatoren `~` und `~*`, zum Beispiel:

```sql
SELECT * FROM test_trgm WHERE t ~ '(foo|bar)';
```

Die Indexsuche extrahiert Trigramme aus dem regulären Ausdruck und schlägt sie im Index nach. Je mehr Trigramme extrahiert werden können, desto wirksamer ist die Indexsuche. Auch hier muss die Suchzeichenkette nicht links verankert sein.

Bei `LIKE`- und regulären Ausdruckssuchen ist zu beachten, dass ein Muster ohne extrahierbare Trigramme zu einem vollständigen Indexscan degeneriert.

Die Wahl zwischen GiST- und GIN-Indizierung hängt von den relativen Performance-Eigenschaften von GiST und GIN ab, die an anderer Stelle behandelt werden.

#### F.35.5. Integration in die Textsuche

Trigramm-Abgleich ist in Verbindung mit einem Volltextindex sehr nützlich. Insbesondere kann er helfen, falsch geschriebene Eingabewörter zu erkennen, die vom Volltextsuchmechanismus nicht direkt gefunden würden.

Der erste Schritt besteht darin, eine Hilfstabelle mit allen eindeutigen Wörtern der Dokumente zu erzeugen:

```sql
CREATE TABLE words AS
SELECT word
FROM ts_stat('SELECT to_tsvector(''simple'', bodytext) FROM documents');
```

Dabei ist `documents` eine Tabelle mit dem Textfeld `bodytext`, das durchsucht werden soll. Die Konfiguration `simple` wird mit `to_tsvector` verwendet, damit eine Liste der ursprünglichen, nicht gestemmten Wörter entsteht.

Als Nächstes wird ein Trigramm-Index auf der Spalte `word` erzeugt:

```sql
CREATE INDEX words_idx ON words USING GIN (word gin_trgm_ops);
```

Nun kann eine `SELECT`-Abfrage ähnlich wie im vorherigen Beispiel verwendet werden, um Schreibvorschläge für falsch geschriebene Wörter in Benutzersuchbegriffen zu erzeugen. Ein nützlicher zusätzlicher Test ist, zu verlangen, dass die ausgewählten Wörter eine ähnliche Länge wie das falsch geschriebene Wort haben.

> **Hinweis:**
> Da die Tabelle `words` als separate, statische Tabelle erzeugt wurde, muss sie regelmäßig neu erzeugt werden, damit sie hinreichend aktuell zur Dokumentensammlung bleibt. Exakte Aktualität ist normalerweise nicht nötig.

#### F.35.6. Referenzen

- GiST Development Site: <http://www.sai.msu.su/~megera/postgres/gist/>
- Tsearch2 Development Site: <http://www.sai.msu.su/~megera/postgres/gist/tsearch/V2/>

#### F.35.7. Autoren

Oleg Bartunov <oleg@sai.msu.su>, Moskau, Moscow University, Russland

Teodor Sigaev <teodor@sigaev.ru>, Moskau, Delta-Soft Ltd., Russland

Alexander Korotkov <a.korotkov@postgrespro.ru>, Moskau, Postgres Professional, Russland

Dokumentation: Christopher Kings-Lynne

Dieses Modul wird von Delta-Soft Ltd., Moskau, Russland, gesponsert.


### F.36. pg_visibility — Informationen und Hilfsfunktionen zur Visibility Map

Das Modul `pg_visibility` ermöglicht es, die Visibility Map (VM) und Informationen zur Sichtbarkeit auf Seitenebene einer Tabelle zu untersuchen. Außerdem stellt es Funktionen bereit, um die Integrität einer Visibility Map zu prüfen und ihren Neuaufbau zu erzwingen.

Drei verschiedene Bits speichern Informationen zur Sichtbarkeit auf Seitenebene. Das `all-visible`-Bit in der Visibility Map zeigt an, dass jedes Tupel auf der entsprechenden Seite der Relation für jede aktuelle und zukünftige Transaktion sichtbar ist. Das `all-frozen`-Bit in der Visibility Map zeigt an, dass jedes Tupel auf der Seite eingefroren ist; ein zukünftiges `VACUUM` muss die Seite also nicht verändern, bis dort ein Tupel eingefügt, aktualisiert, gelöscht oder gesperrt wird. Das Bit `PD_ALL_VISIBLE` im Seiten-Header hat dieselbe Bedeutung wie das `all-visible`-Bit in der Visibility Map, wird aber innerhalb der Datenseite selbst gespeichert. Diese beiden Bits stimmen normalerweise überein, können nach Crash Recovery aber vorübergehend voneinander abweichen. Auch Änderungen zwischen der Untersuchung der Visibility Map und der Datenseite können abweichende Werte erzeugen. Jede Art von Datenkorruption kann ebenfalls dazu führen, dass diese Bits nicht übereinstimmen.

Funktionen, die Informationen über `PD_ALL_VISIBLE`-Bits anzeigen, sind deutlich teurer als Funktionen, die nur die Visibility Map konsultieren, weil sie die Datenblöcke der Relation lesen müssen, nicht nur die viel kleinere Visibility Map. Funktionen, die Datenblöcke der Relation prüfen, sind ähnlich teuer.

#### F.36.1. Funktionen

```text
pg_visibility_map(relation regclass, blkno bigint,
                  all_visible OUT boolean, all_frozen OUT boolean) returns record
```

Gibt die `all-visible`- und `all-frozen`-Bits in der Visibility Map für den angegebenen Block der angegebenen Relation zurück.

```text
pg_visibility(relation regclass, blkno bigint,
              all_visible OUT boolean, all_frozen OUT boolean,
              pd_all_visible OUT boolean) returns record
```

Gibt die `all-visible`- und `all-frozen`-Bits in der Visibility Map für den angegebenen Block der angegebenen Relation sowie das `PD_ALL_VISIBLE`-Bit dieses Blocks zurück.

```text
pg_visibility_map(relation regclass, blkno OUT bigint,
                  all_visible OUT boolean, all_frozen OUT boolean) returns setof record
```

Gibt die `all-visible`- und `all-frozen`-Bits in der Visibility Map für jeden Block der angegebenen Relation zurück.

```text
pg_visibility(relation regclass, blkno OUT bigint,
              all_visible OUT boolean, all_frozen OUT boolean,
              pd_all_visible OUT boolean) returns setof record
```

Gibt die `all-visible`- und `all-frozen`-Bits in der Visibility Map für jeden Block der angegebenen Relation sowie das `PD_ALL_VISIBLE`-Bit jedes Blocks zurück.

```text
pg_visibility_map_summary(relation regclass,
                          all_visible OUT bigint, all_frozen OUT bigint) returns record
```

Gibt gemäß Visibility Map die Anzahl der all-visible-Seiten und die Anzahl der all-frozen-Seiten in der Relation zurück.

```text
pg_check_frozen(relation regclass, t_ctid OUT tid) returns setof tid
```

Gibt die TIDs nicht eingefrorener Tupel zurück, die auf Seiten gespeichert sind, die in der Visibility Map als all-frozen markiert sind. Wenn diese Funktion eine nicht leere Menge von TIDs zurückgibt, ist die Visibility Map beschädigt.

```text
pg_check_visible(relation regclass, t_ctid OUT tid) returns setof tid
```

Gibt die TIDs nicht vollständig sichtbarer Tupel zurück, die auf Seiten gespeichert sind, die in der Visibility Map als all-visible markiert sind. Wenn diese Funktion eine nicht leere Menge von TIDs zurückgibt, ist die Visibility Map beschädigt.

```text
pg_truncate_visibility_map(relation regclass) returns void
```

Kürzt die Visibility Map für die angegebene Relation. Diese Funktion ist nützlich, wenn Sie vermuten, dass die Visibility Map einer Relation beschädigt ist, und ihren Neuaufbau erzwingen möchten. Das erste `VACUUM` auf der Relation nach Ausführung dieser Funktion scannt jede Seite der Relation und baut die Visibility Map neu auf. Bis dahin behandeln Abfragen die Visibility Map so, als enthielte sie nur Nullen.

Standardmäßig können diese Funktionen nur von Superusern und Rollen mit den Rechten der Rolle `pg_stat_scan_tables` ausgeführt werden. Die Ausnahme ist `pg_truncate_visibility_map(relation regclass)`, das nur von Superusern ausgeführt werden kann.

#### F.36.2. Autor

Robert Haas <rhaas@postgresql.org>


### F.37. pg_walinspect — Low-Level-Inspektion des WAL

Das Modul `pg_walinspect` stellt SQL-Funktionen bereit, mit denen Sie den Inhalt des Write-Ahead Logs eines laufenden PostgreSQL-Datenbankclusters auf niedriger Ebene untersuchen können. Das ist für Debugging, Analyse, Reporting und Lernzwecke nützlich. Es ähnelt `pg_waldump`, ist aber über SQL statt als separates Hilfsprogramm zugänglich.

Alle Funktionen dieses Moduls liefern WAL-Informationen mit der aktuellen Timeline-ID des Servers.

> **Hinweis:**
> Die Funktionen von `pg_walinspect` werden oft mit einem LSN-Argument aufgerufen, das den Beginn eines bekannten interessanten WAL-Records angibt. Einige Funktionen, etwa `pg_logical_emit_message`, geben jedoch den LSN nach dem gerade eingefügten Record zurück.

> **Tipp:**
> Alle `pg_walinspect`-Funktionen, die Informationen über Records in einem LSN-Bereich anzeigen, akzeptieren großzügig `end_lsn`-Argumente, die nach dem aktuellen LSN des Servers liegen. Ein `end_lsn` „aus der Zukunft“ löst keinen Fehler aus. Praktisch kann auch `FFFFFFFF/FFFFFFFF`, der maximal gültige `pg_lsn`-Wert, als `end_lsn` verwendet werden; dies entspricht dem aktuellen LSN des Servers.

Standardmäßig ist die Verwendung dieser Funktionen auf Superuser und Mitglieder der Rolle `pg_read_server_files` beschränkt. Superuser können anderen Rollen mit `GRANT` Zugriff erteilen.

#### F.37.1. Allgemeine Funktionen

```text
pg_get_wal_record_info(in_lsn pg_lsn) returns record
```

Liefert Informationen über einen WAL-Record, der sich an oder nach `in_lsn` befindet. Beispiel:

```sql
postgres=# SELECT * FROM pg_get_wal_record_info('0/E419E28');
```

```text
-[ RECORD 1 ]----+-------------------------------------------------
start_lsn        | 0/E419E28
end_lsn          | 0/E419E68
prev_lsn         | 0/E419D78
xid              | 0
resource_manager | Heap2
record_type      | VACUUM
record_length    | 58
main_data_length | 2
fpi_length       | 0
description      | nunused: 5, unused: [1, 2, 3, 4, 5]
block_ref        | blkref #0: rel 1663/16385/1249 fork main blk
```

Wenn `in_lsn` nicht am Anfang eines WAL-Records liegt, werden stattdessen Informationen über den nächsten gültigen WAL-Record angezeigt. Gibt es keinen nächsten gültigen WAL-Record, löst die Funktion einen Fehler aus.

```text
pg_get_wal_records_info(start_lsn pg_lsn, end_lsn pg_lsn) returns setof record
```

Liefert Informationen über alle gültigen WAL-Records zwischen `start_lsn` und `end_lsn`. Es wird eine Zeile pro WAL-Record zurückgegeben. Beispiel:

```sql
postgres=# SELECT * FROM pg_get_wal_records_info('0/1E913618', '0/1E913740') LIMIT 1;
```

```text
-[ RECORD 1 ]----+--------------------------------------------------------------
start_lsn        | 0/1E913618
end_lsn          | 0/1E913650
prev_lsn         | 0/1E9135A0
xid              | 0
resource_manager | Standby
record_type      | RUNNING_XACTS
record_length    | 50
main_data_length | 24
fpi_length       | 0
description      | nextXid 33775 latestCompletedXid 33774 oldestRunningXid 33775
block_ref        |
```

Die Funktion löst einen Fehler aus, wenn `start_lsn` nicht verfügbar ist.

```text
pg_get_wal_block_info(start_lsn pg_lsn, end_lsn pg_lsn,
                      show_data boolean DEFAULT true) returns setof record
```

Liefert Informationen zu jeder Blockreferenz aus allen gültigen WAL-Records zwischen `start_lsn` und `end_lsn`, die eine oder mehrere Blockreferenzen enthalten. Es wird eine Zeile pro Blockreferenz pro WAL-Record zurückgegeben. Beispiel:

```sql
postgres=# SELECT * FROM pg_get_wal_block_info('0/1230278', '0/12302B8');
```

```text
-[ RECORD 1 ]-----+-----------------------------------
start_lsn         | 0/1230278
end_lsn           | 0/12302B8
prev_lsn          | 0/122FD40
block_id          | 0
reltablespace     | 1663
reldatabase       | 1
relfilenode       | 2658
relforknumber     | 0
relblocknumber    | 11
xid               | 341
resource_manager  | Btree
record_type       | INSERT_LEAF
record_length     | 64
main_data_length  | 2
block_data_length | 16
block_fpi_length  | 0
block_fpi_info    |
description       | off: 46
block_data        | \x00002a00070010402630000070696400
block_fpi_data    |
```

Dieses Beispiel betrifft einen WAL-Record mit nur einer Blockreferenz; viele WAL-Records enthalten aber mehrere Blockreferenzen. Von `pg_get_wal_block_info` ausgegebene Zeilen haben garantiert eine eindeutige Kombination aus `start_lsn` und `block_id`.

Ein großer Teil der angezeigten Informationen entspricht der Ausgabe von `pg_get_wal_records_info` für dieselben Argumente. `pg_get_wal_block_info` entfaltet die Informationen jedes WAL-Records jedoch in eine erweiterte Form mit einer Zeile pro Blockreferenz. Manche Details werden dadurch auf Blockreferenzebene statt auf Record-Ebene verfolgt. Das ist nützlich für Abfragen, die nachverfolgen, wie sich einzelne Blöcke über die Zeit verändert haben. Records ohne Blockreferenzen, etwa `COMMIT`-WAL-Records, liefern keine Zeilen; `pg_get_wal_block_info` kann daher weniger Zeilen zurückgeben als `pg_get_wal_records_info`.

Die Parameter `reltablespace`, `reldatabase` und `relfilenode` referenzieren `pg_tablespace.oid`, `pg_database.oid` beziehungsweise `pg_class.relfilenode`. Das Feld `relforknumber` ist die Fork-Nummer innerhalb der Relation für die Blockreferenz; Einzelheiten finden Sie in `common/relpath.h`.

> **Tipp:**
> Die Funktion `pg_filenode_relation` (siehe Tabelle 9.103) kann helfen zu bestimmen, welche Relation bei der ursprünglichen Ausführung geändert wurde.

Clients können den Aufwand für das Materialisieren von Blockdaten vermeiden. Dadurch kann die Funktionsausführung erheblich schneller werden. Wenn `show_data` auf `false` gesetzt ist, werden `block_data` und `block_fpi_data` ausgelassen, das heißt, die entsprechenden OUT-Argumente sind für alle zurückgegebenen Zeilen `NULL`. Diese Optimierung ist natürlich nur sinnvoll, wenn Blockdaten nicht wirklich benötigt werden.

Die Funktion löst einen Fehler aus, wenn `start_lsn` nicht verfügbar ist.

```text
pg_get_wal_stats(start_lsn pg_lsn, end_lsn pg_lsn,
                 per_record boolean DEFAULT false) returns setof record
```

Liefert Statistiken über alle gültigen WAL-Records zwischen `start_lsn` und `end_lsn`. Standardmäßig wird eine Zeile pro `resource_manager`-Typ zurückgegeben. Wenn `per_record` auf `true` gesetzt ist, wird eine Zeile pro `record_type` zurückgegeben. Beispiel:

```sql
postgres=# SELECT * FROM pg_get_wal_stats('0/1E847D00', '0/1E84F500')
WHERE count > 0 AND "resource_manager/record_type" = 'Transaction'
LIMIT 1;
```

```text
-[ RECORD 1 ]----------------+-------------------
resource_manager/record_type | Transaction
count                        | 2
count_percentage             | 8
record_size                  | 875
record_size_percentage       | 41.23468426013195
fpi_size                     | 0
fpi_size_percentage          | 0
combined_size                | 875
combined_size_percentage     | 2.8634072910530795
```

Die Funktion löst einen Fehler aus, wenn `start_lsn` nicht verfügbar ist.

#### F.37.2. Autor

Bharath Rupireddy <bharath.rupireddyforpostgres@gmail.com>


### F.38. postgres_fdw — Zugriff auf Daten in externen PostgreSQL-Servern

Das Modul `postgres_fdw` stellt den Foreign-Data Wrapper `postgres_fdw` bereit, mit dem auf Daten in externen PostgreSQL-Servern zugegriffen werden kann.

Die Funktionalität dieses Moduls überschneidet sich erheblich mit dem älteren Modul `dblink`. `postgres_fdw` bietet jedoch eine transparentere und stärker standardkonforme Syntax für den Zugriff auf entfernte Tabellen und kann in vielen Fällen bessere Performance liefern.

Zur Vorbereitung eines entfernten Zugriffs mit `postgres_fdw`:

1. Installieren Sie die Erweiterung `postgres_fdw` mit `CREATE EXTENSION`.
2. Erzeugen Sie mit `CREATE SERVER` ein Foreign-Server-Objekt für jede entfernte Datenbank, zu der Sie eine Verbindung herstellen möchten. Geben Sie Verbindungsinformationen außer Benutzer und Passwort als Optionen des Serverobjekts an.
3. Erzeugen Sie mit `CREATE USER MAPPING` ein User Mapping für jeden Datenbankbenutzer, dem Sie Zugriff auf den jeweiligen Foreign Server erlauben möchten. Geben Sie den entfernten Benutzernamen und das Passwort als Optionen `user` und `password` des User Mappings an.
4. Erzeugen Sie mit `CREATE FOREIGN TABLE` oder `IMPORT FOREIGN SCHEMA` eine Fremdtabelle für jede entfernte Tabelle, auf die Sie zugreifen möchten. Die Spalten der Fremdtabelle müssen zur referenzierten entfernten Tabelle passen. Tabellen- und Spaltennamen dürfen lokal abweichen, wenn die korrekten entfernten Namen als Optionen des Fremdtabellenobjekts angegeben werden.

Danach genügt ein `SELECT` aus einer Fremdtabelle, um auf die Daten der zugrunde liegenden entfernten Tabelle zuzugreifen. Sie können die entfernte Tabelle auch mit `INSERT`, `UPDATE`, `DELETE`, `COPY` oder `TRUNCATE` ändern, sofern der im User Mapping angegebene entfernte Benutzer die entsprechenden Rechte besitzt.

Die Option `ONLY` in `SELECT`, `UPDATE`, `DELETE` oder `TRUNCATE` hat beim Zugriff auf oder beim Ändern der entfernten Tabelle keine Wirkung.

`postgres_fdw` unterstützt derzeit keine `INSERT`-Anweisungen mit `ON CONFLICT DO UPDATE`. `ON CONFLICT DO NOTHING` wird jedoch unterstützt, sofern keine eindeutige Index-Inferenzspezifikation angegeben wird. Außerdem unterstützt `postgres_fdw` Zeilenverschiebung durch `UPDATE`-Anweisungen auf partitionierten Tabellen, behandelt aber derzeit nicht den Fall, dass eine entfernte Partition, in die eine verschobene Zeile eingefügt werden soll, zugleich eine `UPDATE`-Zielpartition ist, die an anderer Stelle im selben Befehl aktualisiert wird.

Es wird allgemein empfohlen, die Spalten einer Fremdtabelle mit genau denselben Datentypen und gegebenenfalls Collations zu deklarieren wie die referenzierten Spalten der entfernten Tabelle. Obwohl `postgres_fdw` Datentypumwandlungen bei Bedarf derzeit recht großzügig ausführt, können überraschende semantische Anomalien entstehen, wenn Typen oder Collations nicht übereinstimmen, weil der entfernte Server Abfragebedingungen anders interpretiert als der lokale Server.

Eine Fremdtabelle darf weniger Spalten oder eine andere Spaltenreihenfolge haben als die zugrunde liegende entfernte Tabelle. Die Zuordnung der Spalten zur entfernten Tabelle erfolgt nach Namen, nicht nach Position.

#### F.38.1. FDW-Optionen von `postgres_fdw`

##### F.38.1.1. Verbindungsoptionen

Ein Foreign Server, der den Foreign-Data Wrapper `postgres_fdw` verwendet, kann dieselben Optionen haben, die `libpq` in Verbindungszeichenketten akzeptiert, wie in Abschnitt 32.1.2 beschrieben. Ausnahmen sind Optionen, die nicht erlaubt sind oder besonders behandelt werden:

- `user`, `password` und `sslpassword`: Stattdessen in einem User Mapping angeben oder eine Service-Datei verwenden.
- `client_encoding`: Wird automatisch aus der lokalen Serverkodierung gesetzt.
- `application_name`: Kann in der Verbindung und/oder in `postgres_fdw.application_name` vorkommen. Sind beide vorhanden, überschreibt `postgres_fdw.application_name` die Verbindungseinstellung. Anders als `libpq` erlaubt `postgres_fdw` in `application_name` Escape-Sequenzen; Details siehe `postgres_fdw.application_name`.
- `fallback_application_name`: Immer auf `postgres_fdw` gesetzt.
- `sslkey` und `sslcert`: Können in der Verbindung und/oder in einem User Mapping vorkommen. Sind beide vorhanden, überschreibt die Einstellung im User Mapping die Verbindungseinstellung.

Nur Superuser dürfen User Mappings mit den Einstellungen `sslcert` oder `sslkey` erzeugen oder ändern.

Nicht-Superuser dürfen Foreign Server mit Passwortauthentifizierung oder delegierten GSSAPI-Anmeldedaten verwenden. Geben Sie daher die Option `password` für User Mappings von Nicht-Superusern an, wenn Passwortauthentifizierung erforderlich ist.

Ein Superuser kann diese Prüfung pro User Mapping übergehen, indem er die User-Mapping-Option `password_required 'false'` setzt, zum Beispiel:

```sql
ALTER USER MAPPING FOR some_non_superuser SERVER loopback_nopw
OPTIONS (ADD password_required 'false');
```

Damit unprivilegierte Benutzer nicht die Authentifizierungsrechte des Unix-Benutzers ausnutzen können, unter dem der `postgres`-Server läuft, und so Superuser-Rechte erlangen, darf nur ein Superuser diese Option auf einem User Mapping setzen.

Dabei ist Vorsicht nötig, damit der zugeordnete Benutzer nicht gemäß CVE-2007-3278 und CVE-2007-6601 als Superuser zur zugeordneten Datenbank verbinden kann. Setzen Sie `password_required=false` nicht auf der Rolle `public`. Bedenken Sie, dass der zugeordnete Benutzer potenziell alle Client-Zertifikate, `.pgpass`, `.pg_service.conf` usw. im Unix-Home-Verzeichnis des Systembenutzers verwenden kann, unter dem der `postgres`-Server läuft. Details zur Suche nach Home-Verzeichnissen finden Sie in [Abschnitt 32.16](32_libpq_C_Bibliothek.md#3216-die-passwortdatei). Ebenso können Trust-Beziehungen aus Authentifizierungsmodi wie Peer oder Ident genutzt werden.

##### F.38.1.2. Objektnamenoptionen

Diese Optionen steuern die Namen, die in SQL-Anweisungen an den entfernten PostgreSQL-Server gesendet werden. Sie werden benötigt, wenn eine Fremdtabelle mit anderen Namen als die zugrunde liegende entfernte Tabelle erzeugt wird.

- `schema_name` (`string`)
  - Gibt für eine Fremdtabelle den Schemanamen an, der auf dem entfernten Server zu verwenden ist. Wird die Option weggelassen, wird der Schemaname der Fremdtabelle verwendet.

- `table_name` (`string`)
  - Gibt für eine Fremdtabelle den Tabellennamen an, der auf dem entfernten Server zu verwenden ist. Wird die Option weggelassen, wird der Name der Fremdtabelle verwendet.

- `column_name` (`string`)
  - Gibt für eine Spalte einer Fremdtabelle den Spaltennamen an, der auf dem entfernten Server zu verwenden ist. Wird die Option weggelassen, wird der Name der Spalte verwendet.

##### F.38.1.3. Kostenschätzungsoptionen

`postgres_fdw` ruft entfernte Daten durch Ausführen von Abfragen auf entfernten Servern ab. Idealerweise sollten die geschätzten Kosten eines Scans einer Fremdtabelle daher den Kosten auf dem entfernten Server plus Kommunikations-Overhead entsprechen. Am zuverlässigsten erhält man eine solche Schätzung, indem man den entfernten Server fragt und etwas Overhead hinzufügt. Für einfache Abfragen ist eine zusätzliche entfernte Anfrage zur Kostenschätzung aber möglicherweise zu teuer. Daher bietet `postgres_fdw` folgende Optionen zur Steuerung der Kostenschätzung:

- `use_remote_estimate` (`boolean`)
  - Steuert, ob `postgres_fdw` entfernte `EXPLAIN`-Befehle ausführt, um Kostenschätzungen zu erhalten. Die Einstellung einer Fremdtabelle überschreibt die Einstellung ihres Servers, aber nur für diese Tabelle. Standard ist `false`.

- `fdw_startup_cost` (`floating point`)
  - Ein für einen Foreign Server angebbarer Gleitkommawert, der zu den geschätzten Startkosten jedes Fremdtabellenscans auf diesem Server addiert wird. Er repräsentiert zusätzlichen Overhead für Verbindungsaufbau, Parsen und Planen auf der entfernten Seite usw. Standardwert ist `100`.

- `fdw_tuple_cost` (`floating point`)
  - Ein für einen Foreign Server angebbarer Gleitkommawert, der als zusätzliche Kosten pro Tupel für Fremdtabellenscans auf diesem Server verwendet wird. Er repräsentiert den Overhead der Datenübertragung zwischen Servern. Sie können diesen Wert erhöhen oder verringern, um höhere oder niedrigere Netzwerklatenz zum entfernten Server abzubilden. Standardwert ist `0.2`.

Wenn `use_remote_estimate` `true` ist, erhält `postgres_fdw` Zeilenzahl- und Kostenschätzungen vom entfernten Server und addiert `fdw_startup_cost` und `fdw_tuple_cost`. Wenn `use_remote_estimate` `false` ist, führt `postgres_fdw` lokale Zeilenzahl- und Kostenschätzungen aus und addiert dieselben Kosten. Diese lokale Schätzung ist wahrscheinlich ungenau, sofern keine lokalen Kopien der Statistik der entfernten Tabelle verfügbar sind. `ANALYZE` auf der Fremdtabelle aktualisiert die lokalen Statistiken, indem die entfernte Tabelle gescannt und Statistiken berechnet und gespeichert werden, als wäre die Tabelle lokal. Lokale Statistiken können den Planungs-Overhead pro Abfrage reduzieren, werden aber schnell veraltet, wenn die entfernte Tabelle häufig geändert wird.

Die folgende Option steuert das Verhalten einer solchen `ANALYZE`-Operation:

- `analyze_sampling` (`string`)
  - Legt fest, ob `ANALYZE` auf einer Fremdtabelle die Daten auf der entfernten Seite sampelt oder alle Daten liest und überträgt und das Sampling lokal ausführt. Unterstützte Werte sind `off`, `random`, `system`, `bernoulli` und `auto`. `off` deaktiviert entferntes Sampling. `random` verwendet auf der entfernten Seite die Funktion `random()`, während `system` und `bernoulli` die eingebauten `TABLESAMPLE`-Methoden gleichen Namens verwenden. `random` funktioniert mit allen entfernten Serverversionen; `TABLESAMPLE` wird erst ab 9.5 unterstützt. `auto` (Standard) wählt automatisch die empfohlene Sampling-Methode, derzeit je nach entfernter Serverversion `bernoulli` oder `random`.

##### F.38.1.4. Optionen für entfernte Ausführung

Standardmäßig werden nur `WHERE`-Klauseln mit eingebauten Operatoren und Funktionen für die Ausführung auf dem entfernten Server in Betracht gezogen. Klauseln mit nicht eingebauten Funktionen werden lokal geprüft, nachdem die Zeilen abgerufen wurden. Wenn solche Funktionen auf dem entfernten Server verfügbar sind und dort zuverlässig dieselben Ergebnisse liefern wie lokal, kann die Performance verbessert werden, indem solche `WHERE`-Klauseln zur entfernten Ausführung gesendet werden. Dieses Verhalten steuert folgende Option:

- `extensions` (`string`)
  - Eine kommaseparierte Liste von Namen von PostgreSQL-Erweiterungen, die in kompatiblen Versionen sowohl auf dem lokalen als auch auf dem entfernten Server installiert sind. Immutable Funktionen und Operatoren, die zu einer gelisteten Erweiterung gehören, gelten als zum entfernten Server sendbar. Diese Option kann nur für Foreign Server angegeben werden, nicht pro Tabelle.

Bei Verwendung von `extensions` ist der Benutzer dafür verantwortlich, dass die gelisteten Erweiterungen auf beiden Servern existieren und sich identisch verhalten. Andernfalls können entfernte Abfragen fehlschlagen oder unerwartete Ergebnisse liefern.

- `fetch_size` (`integer`)
  - Gibt an, wie viele Zeilen `postgres_fdw` in jeder Fetch-Operation abrufen soll. Die Option kann für eine Fremdtabelle oder einen Foreign Server angegeben werden; eine Tabellenoption überschreibt die Serveroption. Standard ist `100`.

- `batch_size` (`integer`)
  - Gibt an, wie viele Zeilen `postgres_fdw` in jeder Insert-Operation einfügen soll. Die Option kann für eine Fremdtabelle oder einen Foreign Server angegeben werden; eine Tabellenoption überschreibt die Serveroption. Standard ist `1`.

Die tatsächliche Anzahl der auf einmal eingefügten Zeilen hängt von der Spaltenzahl und `batch_size` ab. Der Batch wird als einzelne Abfrage ausgeführt, und das `libpq`-Protokoll begrenzt die Anzahl der Parameter in einer Abfrage auf `65535`. Überschreitet `Spaltenzahl * batch_size` diese Grenze, wird `batch_size` angepasst, um einen Fehler zu vermeiden.

Diese Option gilt auch beim Kopieren in Fremdtabellen. In diesem Fall wird die tatsächliche Zeilenzahl ähnlich bestimmt wie beim Insert, ist wegen Implementierungsbeschränkungen des `COPY`-Befehls jedoch auf höchstens `1000` begrenzt.

##### F.38.1.5. Optionen für asynchrone Ausführung

`postgres_fdw` unterstützt asynchrone Ausführung, bei der mehrere Teile eines `Append`-Knotens gleichzeitig statt seriell laufen, um die Performance zu verbessern. Diese Ausführung steuert folgende Option:

- `async_capable` (`boolean`)
  - Steuert, ob `postgres_fdw` Fremdtabellen für asynchrone Ausführung gleichzeitig scannen darf. Die Option kann für eine Fremdtabelle oder einen Foreign Server angegeben werden; eine Tabellenoption überschreibt die Serveroption. Standard ist `false`.

Um konsistente Daten von einem Foreign Server sicherzustellen, öffnet `postgres_fdw` nur eine Verbindung pro Foreign Server und führt alle Abfragen gegen diesen Server seriell aus, selbst wenn mehrere Fremdtabellen beteiligt sind, sofern diese Tabellen nicht unterschiedlichen User Mappings unterliegen. In einem solchen Fall kann es performanter sein, diese Option zu deaktivieren, um den Overhead asynchroner Abfragen zu vermeiden.

Asynchrone Ausführung wird auch angewendet, wenn ein `Append`-Knoten sowohl synchron als auch asynchron ausgeführte Unterpläne enthält. Wenn die asynchronen Unterpläne mit `postgres_fdw` verarbeitet werden, werden Tupel aus diesen Unterplänen erst zurückgegeben, nachdem mindestens ein synchroner Unterplan alle Tupel geliefert hat. Dieses Verhalten kann sich in einer zukünftigen Version ändern.

##### F.38.1.6. Optionen für Transaktionsverwaltung

Wie im Abschnitt zur Transaktionsverwaltung beschrieben, verwaltet `postgres_fdw` Transaktionen, indem entsprechende entfernte Transaktionen erzeugt werden. Subtransaktionen werden durch entsprechende entfernte Subtransaktionen verwaltet. Sind mehrere entfernte Transaktionen oder Subtransaktionen beteiligt, werden sie standardmäßig seriell committed oder abgebrochen. Die Performance kann mit folgenden Optionen verbessert werden:

- `parallel_commit` (`boolean`)
  - Steuert, ob `postgres_fdw` entfernte Transaktionen auf einem Foreign Server parallel committet, wenn die lokale Transaktion committed wird. Diese Einstellung gilt auch für entfernte und lokale Subtransaktionen. Die Option kann nur für Foreign Server angegeben werden, nicht pro Tabelle. Standard ist `false`.

- `parallel_abort` (`boolean`)
  - Steuert, ob `postgres_fdw` entfernte Transaktionen auf einem Foreign Server parallel abbricht, wenn die lokale Transaktion abgebrochen wird. Diese Einstellung gilt auch für entfernte und lokale Subtransaktionen. Die Option kann nur für Foreign Server angegeben werden, nicht pro Tabelle. Standard ist `false`.

Sind mehrere Foreign Server mit diesen Optionen an einer lokalen Transaktion beteiligt, werden die entfernten Transaktionen auf diesen Servern beim lokalen Commit oder Abort parallel über die Foreign Server hinweg committed oder abgebrochen.

Bei aktivierten Optionen kann ein Foreign Server mit vielen entfernten Transaktionen beim lokalen Commit oder Abort eine negative Performance-Auswirkung sehen.

##### F.38.1.7. Aktualisierbarkeitsoptionen

Standardmäßig gelten alle Fremdtabellen, die `postgres_fdw` verwenden, als aktualisierbar. Dies kann mit folgender Option überschrieben werden:

- `updatable` (`boolean`)
  - Steuert, ob `postgres_fdw` Änderungen an Fremdtabellen mit `INSERT`, `UPDATE` und `DELETE` erlaubt. Die Option kann für eine Fremdtabelle oder einen Foreign Server angegeben werden; eine Tabellenoption überschreibt die Serveroption. Standard ist `true`.

Wenn die entfernte Tabelle tatsächlich nicht aktualisierbar ist, tritt natürlich trotzdem ein Fehler auf. Der Nutzen dieser Option liegt vor allem darin, den Fehler lokal auszulösen, ohne den entfernten Server abzufragen. Die Sichten in `information_schema` melden eine `postgres_fdw`-Fremdtabelle gemäß dieser Option als aktualisierbar oder nicht, ohne den entfernten Server zu prüfen.

##### F.38.1.8. TRUNCATE-Optionen

Standardmäßig gelten alle Fremdtabellen, die `postgres_fdw` verwenden, als mit `TRUNCATE` kürzbar. Dies kann mit folgender Option überschrieben werden:

- `truncatable` (`boolean`)
  - Steuert, ob `postgres_fdw` `TRUNCATE` auf Fremdtabellen erlaubt. Die Option kann für eine Fremdtabelle oder einen Foreign Server angegeben werden; eine Tabellenoption überschreibt die Serveroption. Standard ist `true`.

Wenn die entfernte Tabelle tatsächlich nicht kürzbar ist, tritt natürlich trotzdem ein Fehler auf. Der Nutzen dieser Option liegt vor allem darin, den Fehler lokal auszulösen, ohne den entfernten Server abzufragen.

##### F.38.1.9. Importoptionen

`postgres_fdw` kann Fremdtabellendefinitionen mit `IMPORT FOREIGN SCHEMA` importieren. Dieser Befehl erzeugt auf dem lokalen Server Fremdtabellendefinitionen, die zu Tabellen oder Sichten auf dem entfernten Server passen. Wenn die zu importierenden entfernten Tabellen Spalten benutzerdefinierter Datentypen haben, muss der lokale Server kompatible Typen gleichen Namens besitzen.

Das Importverhalten kann mit folgenden Optionen angepasst werden, die im Befehl `IMPORT FOREIGN SCHEMA` angegeben werden:

- `import_collate` (`boolean`)
  - Steuert, ob `COLLATE`-Optionen von Spalten in die Definitionen importierter Fremdtabellen aufgenommen werden. Standard ist `true`. Sie müssen dies eventuell deaktivieren, wenn der entfernte Server andere Collation-Namen hat als der lokale, etwa wegen eines anderen Betriebssystems. Bei Deaktivierung besteht jedoch ein erhebliches Risiko, dass die Collations der importierten Tabellenspalten nicht zu den zugrunde liegenden Daten passen.

Auch bei `true` kann der Import von Spalten riskant sein, deren Collation die Standard-Collation des entfernten Servers ist. Sie werden mit `COLLATE "default"` importiert, was die Standard-Collation des lokalen Servers auswählt; diese kann abweichen.

- `import_default` (`boolean`)
  - Steuert, ob `DEFAULT`-Ausdrücke von Spalten in die Definitionen importierter Fremdtabellen aufgenommen werden. Standard ist `false`. Bei Aktivierung sollten Sie auf Defaults achten, die lokal anders berechnet werden könnten als entfernt; `nextval()` ist eine häufige Problemquelle. Der Import schlägt vollständig fehl, wenn ein importierter Default-Ausdruck eine lokal nicht vorhandene Funktion oder einen lokal nicht vorhandenen Operator verwendet.

- `import_generated` (`boolean`)
  - Steuert, ob `GENERATED`-Ausdrücke von Spalten in die Definitionen importierter Fremdtabellen aufgenommen werden. Standard ist `true`. Der Import schlägt vollständig fehl, wenn ein importierter Generated-Ausdruck eine lokal nicht vorhandene Funktion oder einen lokal nicht vorhandenen Operator verwendet.

- `import_not_null` (`boolean`)
  - Steuert, ob `NOT NULL`-Constraints von Spalten in die Definitionen importierter Fremdtabellen aufgenommen werden. Standard ist `true`.

Andere Constraints als `NOT NULL` werden nie aus entfernten Tabellen importiert. PostgreSQL unterstützt zwar Check-Constraints auf Fremdtabellen, importiert sie aber nicht automatisch, weil ein Constraint-Ausdruck lokal und entfernt unterschiedlich ausgewertet werden könnte. Solche Inkonsistenzen können schwer erkennbare Fehler in der Abfrageoptimierung verursachen. Wenn Sie Check-Constraints importieren möchten, müssen Sie dies manuell tun und die Semantik jedes einzelnen sorgfältig prüfen. Weitere Details zur Behandlung von Check-Constraints auf Fremdtabellen finden Sie bei `CREATE FOREIGN TABLE`.

Tabellen oder Fremdtabellen, die Partitionen einer anderen Tabelle sind, werden nur importiert, wenn sie ausdrücklich in der `LIMIT TO`-Klausel genannt werden. Andernfalls werden sie automatisch von `IMPORT FOREIGN SCHEMA` ausgeschlossen. Da alle Daten über die partitionierte Root-Tabelle zugänglich sind, sollte der Import nur partitionierter Tabellen den Zugriff auf alle Daten ohne zusätzliche Objekte ermöglichen.

##### F.38.1.10. Optionen für Verbindungsverwaltung

Standardmäßig hält `postgres_fdw` alle Verbindungen zu Foreign Servern in der lokalen Sitzung für eine Wiederverwendung offen.

- `keep_connections` (`boolean`)
  - Steuert, ob `postgres_fdw` Verbindungen zum Foreign Server offen hält, damit spätere Abfragen sie wiederverwenden können. Diese Option kann nur für einen Foreign Server angegeben werden. Standard ist `on`. Bei `off` werden alle Verbindungen zu diesem Foreign Server am Ende jeder Transaktion verworfen.

- `use_scram_passthrough` (`boolean`)
  - Steuert, ob `postgres_fdw` SCRAM-Pass-Through-Authentifizierung zur Verbindung mit dem Foreign Server verwendet. Dabei verwendet `postgres_fdw` SCRAM-gehashte Secrets statt Klartextpasswörtern, um sich am entfernten Server zu authentifizieren. So wird vermieden, Klartextpasswörter in PostgreSQL-Systemkatalogen zu speichern.

Zur Verwendung von SCRAM-Pass-Through-Authentifizierung:

- Der entfernte Server muss die Authentifizierungsmethode `scram-sha-256` anfordern, sonst schlägt die Verbindung fehl.
- Der entfernte Server kann jede PostgreSQL-Version verwenden, die SCRAM unterstützt. Unterstützung für `use_scram_passthrough` wird nur auf der Client-Seite, also der FDW-Seite, benötigt.
- Das Passwort im User Mapping wird nicht verwendet.
- Der Server, auf dem `postgres_fdw` läuft, und der entfernte Server müssen identische SCRAM-Secrets, also verschlüsselte Passwörter, für den Benutzer haben, der sich über `postgres_fdw` am Foreign Server authentifiziert. Identisch bedeutet gleicher Salt und gleiche Iterationen, nicht nur dasselbe Passwort.
- Wenn FDW-Verbindungen zu mehreren Hosts hergestellt werden, etwa für partitionierte Fremdtabellen oder Sharding, müssen alle Hosts identische SCRAM-Secrets für die beteiligten Benutzer besitzen.
- Die aktuelle Sitzung auf der PostgreSQL-Instanz, die die ausgehenden FDW-Verbindungen herstellt, muss auch für ihre eingehende Client-Verbindung SCRAM-Authentifizierung verwenden. Daher „Pass-Through“: SCRAM muss hinein und hinaus verwendet werden. Dies ist eine technische Anforderung des SCRAM-Protokolls.

#### F.38.2. Funktionen

```text
postgres_fdw_get_connections(IN check_conn boolean DEFAULT false,
                             OUT server_name text,
                             OUT user_name text,
                             OUT valid boolean,
                             OUT used_in_xact boolean,
                             OUT closed boolean,
                             OUT remote_backend_pid int4) returns setof record
```

Diese Funktion gibt Informationen über alle offenen Verbindungen zurück, die `postgres_fdw` von der lokalen Sitzung zu Foreign Servern aufgebaut hat. Gibt es keine offenen Verbindungen, werden keine Datensätze zurückgegeben.

Wenn `check_conn` auf `true` gesetzt ist, prüft die Funktion den Status jeder Verbindung und zeigt das Ergebnis in der Spalte `closed` an. Diese Funktionalität ist derzeit nur auf Systemen verfügbar, die die nicht standardisierte Erweiterung `POLLRDHUP` des Systemaufrufs `poll` unterstützen, etwa Linux. Das ist nützlich, um zu prüfen, ob alle in einer Transaktion verwendeten Verbindungen noch offen sind. Sind Verbindungen geschlossen, kann die Transaktion nicht erfolgreich committed werden; es ist daher besser, sofort zurückzurollen, sobald eine geschlossene Verbindung erkannt wird.

Beispiel:

```sql
postgres=# SELECT * FROM postgres_fdw_get_connections(true);
```

```text
server_name | user_name | valid | used_in_xact | closed | remote_backend_pid
-------------+-----------+-------+--------------+--------+--------------------
loopback1   | postgres  | t     | t            | f      |
loopback2   | public    | t     | t            | f      |
loopback3   |           | f     | t            | f      |
```

Die Ausgabespalten sind in Tabelle F.28 beschrieben.

**Tabelle F.28. Ausgabespalten von `postgres_fdw_get_connections`**

| Spalte | Typ | Beschreibung |
|---|---|---|
| `server_name` | `text` | Name des Foreign Servers dieser Verbindung. Wenn der Server gelöscht wurde, die Verbindung aber offen bleibt und als ungültig markiert ist, ist dieser Wert `NULL`. |
| `user_name` | `text` | Name des lokalen Benutzers, der diesem Foreign Server zugeordnet ist, oder `public`, wenn ein öffentliches Mapping verwendet wird. Wenn das User Mapping gelöscht wurde, die Verbindung aber offen bleibt und als ungültig markiert ist, ist dieser Wert `NULL`. |
| `valid` | `boolean` | `false`, wenn die Verbindung ungültig ist, also in der aktuellen Transaktion verwendet wird, ihr Foreign Server oder User Mapping aber geändert oder gelöscht wurde. Die ungültige Verbindung wird am Ende der Transaktion geschlossen. Andernfalls wird `true` zurückgegeben. |
| `used_in_xact` | `boolean` | `true`, wenn diese Verbindung in der aktuellen Transaktion verwendet wird. |
| `closed` | `boolean` | `true`, wenn diese Verbindung geschlossen ist, andernfalls `false`. `NULL`, wenn `check_conn` `false` ist oder die Statusprüfung auf dieser Plattform nicht verfügbar ist. |
| `remote_backend_pid` | `int4` | Prozess-ID des entfernten Backends auf dem Foreign Server, das die Verbindung bearbeitet. Wenn das entfernte Backend beendet und die Verbindung geschlossen wurde, wird die Prozess-ID des beendeten Backends weiterhin angezeigt. |

```text
postgres_fdw_disconnect(server_name text) returns boolean
```

Diese Funktion verwirft offene Verbindungen, die `postgres_fdw` von der lokalen Sitzung zum Foreign Server mit dem angegebenen Namen aufgebaut hat. Es kann mehrere Verbindungen zu demselben Server über verschiedene User Mappings geben. Werden die Verbindungen in der aktuellen lokalen Transaktion verwendet, werden sie nicht getrennt und Warnmeldungen ausgegeben. Die Funktion gibt `true` zurück, wenn mindestens eine Verbindung getrennt wurde, andernfalls `false`. Wird kein Foreign Server mit dem angegebenen Namen gefunden, wird ein Fehler gemeldet. Beispiel:

```sql
postgres=# SELECT postgres_fdw_disconnect('loopback1');
```

```text
 postgres_fdw_disconnect
-------------------------
 t
```

```text
postgres_fdw_disconnect_all() returns boolean
```

Diese Funktion verwirft alle offenen Verbindungen, die `postgres_fdw` von der lokalen Sitzung zu Foreign Servern aufgebaut hat. Werden die Verbindungen in der aktuellen lokalen Transaktion verwendet, werden sie nicht getrennt und Warnmeldungen ausgegeben. Die Funktion gibt `true` zurück, wenn mindestens eine Verbindung getrennt wurde, andernfalls `false`. Beispiel:

```sql
postgres=# SELECT postgres_fdw_disconnect_all();
```

```text
 postgres_fdw_disconnect_all
-----------------------------
 t
```

#### F.38.3. Verbindungsverwaltung

`postgres_fdw` baut während der ersten Abfrage, die eine Fremdtabelle eines Foreign Servers verwendet, eine Verbindung zu diesem Foreign Server auf. Standardmäßig wird diese Verbindung für spätere Abfragen in derselben Sitzung beibehalten und wiederverwendet. Dieses Verhalten kann mit der Option `keep_connections` eines Foreign Servers gesteuert werden. Wenn mehrere Benutzeridentitäten, also User Mappings, für den Zugriff auf den Foreign Server verwendet werden, wird pro User Mapping eine Verbindung aufgebaut.

Beim Ändern der Definition oder beim Entfernen eines Foreign Servers oder User Mappings werden die zugehörigen Verbindungen geschlossen. Wenn Verbindungen in der aktuellen lokalen Transaktion verwendet werden, bleiben sie jedoch bis zum Ende der Transaktion bestehen. Geschlossene Verbindungen werden bei Bedarf durch zukünftige Abfragen mit einer Fremdtabelle neu aufgebaut.

Sobald eine Verbindung zu einem Foreign Server aufgebaut wurde, bleibt sie standardmäßig bestehen, bis die lokale oder entsprechende entfernte Sitzung endet. Zum expliziten Trennen kann die Option `keep_connections` eines Foreign Servers deaktiviert oder `postgres_fdw_disconnect()` beziehungsweise `postgres_fdw_disconnect_all()` verwendet werden. Das ist beispielsweise nützlich, um nicht mehr benötigte Verbindungen zu schließen und dadurch Verbindungen auf dem Foreign Server freizugeben.

#### F.38.4. Transaktionsverwaltung

Während einer Abfrage, die entfernte Tabellen auf einem Foreign Server referenziert, öffnet `postgres_fdw` auf dem entfernten Server eine Transaktion, sofern noch keine zur aktuellen lokalen Transaktion passende entfernte Transaktion offen ist. Die entfernte Transaktion wird committed oder abgebrochen, wenn die lokale Transaktion committed oder abgebrochen wird. Savepoints werden entsprechend durch entfernte Savepoints verwaltet.

Die entfernte Transaktion verwendet `SERIALIZABLE`, wenn die lokale Transaktion die Isolationsstufe `SERIALIZABLE` hat; andernfalls verwendet sie `REPEATABLE READ`. Diese Wahl stellt sicher, dass eine Abfrage mit mehreren Tabellenscans auf dem entfernten Server snapshot-konsistente Ergebnisse für alle Scans erhält. Eine Folge ist, dass aufeinanderfolgende Abfragen innerhalb einer einzelnen Transaktion dieselben Daten vom entfernten Server sehen, auch wenn dort gleichzeitig Änderungen durch andere Aktivitäten stattfinden. Bei lokalen Transaktionen mit `SERIALIZABLE` oder `REPEATABLE READ` wäre dieses Verhalten ohnehin zu erwarten, kann aber bei lokalen `READ COMMITTED`-Transaktionen überraschen. Eine zukünftige PostgreSQL-Version könnte diese Regeln ändern.

`postgres_fdw` unterstützt derzeit nicht, die entfernte Transaktion für Two-Phase Commit vorzubereiten.

#### F.38.5. Optimierung entfernter Abfragen

`postgres_fdw` versucht, entfernte Abfragen zu optimieren, um die vom Foreign Server übertragene Datenmenge zu reduzieren. Dazu werden `WHERE`-Klauseln zur Ausführung an den entfernten Server gesendet und Tabellenspalten, die für die aktuelle Abfrage nicht benötigt werden, nicht abgerufen. Um Fehlverhalten zu vermeiden, werden `WHERE`-Klauseln nur dann an den entfernten Server gesendet, wenn sie ausschließlich eingebaute Datentypen, Operatoren und Funktionen verwenden oder solche, die zu einer in der Option `extensions` des Foreign Servers gelisteten Erweiterung gehören. Operatoren und Funktionen müssen außerdem `IMMUTABLE` sein.

Bei `UPDATE`- oder `DELETE`-Abfragen versucht `postgres_fdw`, die gesamte Abfrage an den entfernten Server zu senden, wenn keine nicht sendbaren `WHERE`-Klauseln vorhanden sind, keine lokalen Joins beteiligt sind, keine lokalen Row-Level-`BEFORE`- oder `AFTER`-Trigger oder gespeicherten Generated Columns auf der Zieltabelle existieren und keine `CHECK OPTION`-Constraints aus übergeordneten Sichten vorliegen. In `UPDATE` müssen Ausdrücke zur Zuweisung an Zielspalten ausschließlich eingebaute Datentypen, `IMMUTABLE`-Operatoren oder `IMMUTABLE`-Funktionen verwenden.

Wenn `postgres_fdw` einen Join zwischen Fremdtabellen auf demselben Foreign Server erkennt, sendet es den gesamten Join an den Foreign Server, sofern es nicht aus irgendeinem Grund annimmt, dass das einzelne Abrufen der Zeilen aus jeder Tabelle effizienter wäre, oder sofern die beteiligten Tabellenreferenzen nicht unterschiedlichen User Mappings unterliegen. Für `JOIN`-Klauseln gelten dieselben Vorsichtsmaßnahmen wie für `WHERE`-Klauseln.

Die tatsächlich zur Ausführung an den entfernten Server gesendete Abfrage kann mit `EXPLAIN VERBOSE` untersucht werden.

#### F.38.6. Ausführungsumgebung entfernter Abfragen

In den von `postgres_fdw` geöffneten entfernten Sitzungen wird `search_path` auf nur `pg_catalog` gesetzt, sodass nur eingebaute Objekte ohne Schemaqualifikation sichtbar sind. Für Abfragen, die `postgres_fdw` selbst erzeugt, ist das kein Problem, weil es immer qualifizierte Namen verwendet. Es kann aber für Funktionen gefährlich sein, die auf dem entfernten Server über Trigger oder Regeln auf entfernten Tabellen ausgeführt werden. Wenn eine entfernte Tabelle tatsächlich eine Sicht ist, werden alle in dieser Sicht verwendeten Funktionen mit dem eingeschränkten Suchpfad ausgeführt. Es wird empfohlen, alle Namen in solchen Funktionen schemaqualifiziert zu schreiben oder diesen Funktionen `SET search_path`-Optionen anzuhängen (siehe `CREATE FUNCTION`), um die erwartete Suchpfadumgebung festzulegen.

`postgres_fdw` setzt außerdem entfernte Sitzungseinstellungen für verschiedene Parameter:

- `TimeZone` wird auf `UTC` gesetzt.
- `DateStyle` wird auf `ISO` gesetzt.
- `IntervalStyle` wird auf `postgres` gesetzt.
- `extra_float_digits` wird für entfernte Server ab 9.0 auf `3` gesetzt und für ältere Versionen auf `2`.

Diese Einstellungen sind weniger wahrscheinlich problematisch als `search_path`, können aber bei Bedarf mit `SET`-Optionen von Funktionen behandelt werden.

Es wird nicht empfohlen, dieses Verhalten durch Änderung der Sitzungseinstellungen dieser Parameter zu übergehen; dies kann dazu führen, dass `postgres_fdw` nicht korrekt funktioniert.

#### F.38.7. Versionsübergreifende Kompatibilität

`postgres_fdw` kann mit entfernten Servern bis zurück zu PostgreSQL 8.3 verwendet werden. Nur-Lese-Fähigkeit ist bis 8.1 zurück verfügbar.

Eine Einschränkung ist, dass `postgres_fdw` im Allgemeinen annimmt, immutable eingebaute Funktionen und Operatoren seien sicher zur entfernten Ausführung sendbar, wenn sie in einer `WHERE`-Klausel für eine Fremdtabelle erscheinen. Eine eingebaute Funktion, die erst nach der Version des entfernten Servers hinzugefügt wurde, könnte daher zur Ausführung dorthin gesendet werden und einen Fehler wie „function does not exist“ verursachen. Dies lässt sich umgehen, indem die Abfrage umgeschrieben wird, etwa indem die Fremdtabellenreferenz in eine Unterabfrage mit `OFFSET 0` als Optimierungsbarriere eingebettet und die problematische Funktion oder der Operator außerhalb dieser Unterabfrage platziert wird.

Eine weitere Einschränkung ist, dass bei `INSERT`-Anweisungen mit `ON CONFLICT DO NOTHING` auf einer Fremdtabelle der entfernte Server PostgreSQL 9.5 oder neuer ausführen muss, da frühere Versionen diese Funktion nicht unterstützen.

#### F.38.8. Wait Events

`postgres_fdw` kann folgende Wait Events unter dem Wait-Event-Typ `Extension` melden:

| Wait Event | Bedeutung |
|---|---|
| `PostgresFdwCleanupResult` | Warten auf Transaktionsabbruch auf dem entfernten Server. |
| `PostgresFdwConnect` | Warten auf den Aufbau einer Verbindung zu einem entfernten Server. |
| `PostgresFdwGetResult` | Warten auf den Empfang der Ergebnisse einer Abfrage vom entfernten Server. |

#### F.38.9. Konfigurationsparameter

`postgres_fdw.application_name` (`string`)

Gibt einen Wert für den Konfigurationsparameter `application_name` an, der verwendet wird, wenn `postgres_fdw` eine Verbindung zu einem Foreign Server aufbaut. Dies überschreibt die Option `application_name` des Serverobjekts. Änderungen dieses Parameters wirken sich erst auf bestehende Verbindungen aus, wenn diese neu aufgebaut werden.

`postgres_fdw.application_name` kann eine Zeichenkette beliebiger Länge sein und auch Nicht-ASCII-Zeichen enthalten. Wird sie jedoch als `application_name` an einen Foreign Server übergeben, wird sie auf weniger als `NAMEDATALEN` Zeichen gekürzt. Alles außer druckbaren ASCII-Zeichen wird durch C-artige hexadezimale Escapes ersetzt; Details siehe `application_name`.

`%`-Zeichen beginnen Escape-Sequenzen, die durch Statusinformationen ersetzt werden. Nicht erkannte Escapes werden ignoriert. Andere Zeichen werden direkt in den Anwendungsnamen kopiert. Zwischen `%` und der Option darf kein Plus-/Minuszeichen oder numerisches Literal für Ausrichtung oder Padding angegeben werden.

| Escape | Wirkung |
|---|---|
| `%a` | Anwendungsname auf dem lokalen Server |
| `%c` | Sitzungs-ID auf dem lokalen Server; Details siehe `log_line_prefix` |
| `%C` | Clustername auf dem lokalen Server; Details siehe `cluster_name` |
| `%u` | Benutzername auf dem lokalen Server |
| `%d` | Datenbankname auf dem lokalen Server |
| `%p` | Prozess-ID des Backends auf dem lokalen Server |
| `%%` | Literales `%` |

Wenn zum Beispiel der Benutzer `local_user` aus der Datenbank `local_db` eine Verbindung zu `foreign_db` als Benutzer `foreign_user` aufbaut, wird die Einstellung `'db=%d, user=%u'` zu `'db=local_db, user=local_user'` ersetzt.

#### F.38.10. Beispiele

Hier ist ein Beispiel zum Erzeugen einer Fremdtabelle mit `postgres_fdw`. Zuerst wird die Erweiterung installiert:

```sql
CREATE EXTENSION postgres_fdw;
```

Dann wird mit `CREATE SERVER` ein Foreign Server erzeugt. In diesem Beispiel soll eine Verbindung zu einem PostgreSQL-Server auf Host `192.83.123.89` hergestellt werden, der auf Port `5432` lauscht. Die entfernte Datenbank heißt `foreign_db`:

```sql
CREATE SERVER foreign_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '192.83.123.89', port '5432', dbname 'foreign_db');
```

Ein mit `CREATE USER MAPPING` definiertes User Mapping ist ebenfalls nötig, um die Rolle zu bestimmen, die auf dem entfernten Server verwendet wird:

```sql
CREATE USER MAPPING FOR local_user
SERVER foreign_server
OPTIONS (user 'foreign_user', password 'password');
```

Nun kann mit `CREATE FOREIGN TABLE` eine Fremdtabelle erzeugt werden. In diesem Beispiel soll auf die Tabelle `some_schema.some_table` auf dem entfernten Server zugegriffen werden. Ihr lokaler Name ist `foreign_table`:

```sql
CREATE FOREIGN TABLE foreign_table (
    id integer NOT NULL,
    data text
)
SERVER foreign_server
OPTIONS (schema_name 'some_schema', table_name 'some_table');
```

Es ist wesentlich, dass die Datentypen und sonstigen Eigenschaften der in `CREATE FOREIGN TABLE` deklarierten Spalten zur tatsächlichen entfernten Tabelle passen. Auch Spaltennamen müssen übereinstimmen, sofern Sie nicht `column_name`-Optionen an einzelne Spalten anhängen, um deren Namen in der entfernten Tabelle anzugeben. In vielen Fällen ist `IMPORT FOREIGN SCHEMA` dem manuellen Aufbau von Fremdtabellendefinitionen vorzuziehen.

#### F.38.11. Autor

Shigeru Hanada <shigeru.hanada@gmail.com>


### F.39. seg — Datentyp für Liniensegmente oder Gleitkomma-Intervalle

Dieses Modul implementiert den Datentyp `seg` zur Darstellung von Liniensegmenten oder Gleitkomma-Intervallen. `seg` kann Unsicherheit an Intervallgrenzen darstellen und eignet sich dadurch besonders für Labormessungen.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

#### F.39.1. Begründung

Die Geometrie von Messungen ist meist komplexer als ein Punkt auf einem numerischen Kontinuum. Eine Messung ist gewöhnlich ein Abschnitt dieses Kontinuums mit etwas unscharfen Grenzen. Messwerte erscheinen als Intervalle wegen Unsicherheit und Zufälligkeit, aber auch weil der gemessene Wert selbst natürlicherweise ein Intervall sein kann, etwa der Temperaturbereich, in dem ein Protein stabil ist.

Schon mit gesundem Menschenverstand wirkt es praktischer, solche Daten als Intervalle statt als Zahlenpaare zu speichern. In der Praxis erweist sich das in den meisten Anwendungen sogar als effizienter.

Die Unschärfe der Grenzen legt außerdem nahe, dass traditionelle numerische Datentypen zu Informationsverlust führen. Betrachten Sie Folgendes: Ihr Instrument liest `6.50`, und Sie geben diesen Wert in die Datenbank ein. Was erhalten Sie beim Abrufen?

```sql
test=> SELECT 6.50::float8 AS "pH";
```

```text
 pH
---
6.5
(1 row)
```

In der Welt der Messungen ist `6.50` nicht dasselbe wie `6.5`. Manchmal kann dieser Unterschied kritisch sein. Experimentatoren notieren und veröffentlichen üblicherweise die Ziffern, denen sie vertrauen. `6.50` ist tatsächlich ein unscharfes Intervall innerhalb eines größeren und noch unschärferen Intervalls `6.5`; gemeinsam ist ihnen vermutlich nur der Mittelpunkt. Solche unterschiedlichen Daten sollten nicht gleich erscheinen.

Es ist daher nützlich, einen speziellen Datentyp zu haben, der Intervallgrenzen mit beliebig variabler Genauigkeit speichern kann, wobei jedes Datenelement seine eigene Genauigkeit aufzeichnet.

Beispiel:

```sql
test=> SELECT '6.25 .. 6.50'::seg AS "pH";
```

```text
     pH
------------
6.25 .. 6.50
(1 row)
```

#### F.39.2. Syntax

Die externe Darstellung eines Intervalls wird aus einer oder zwei Gleitkommazahlen gebildet, die mit dem Bereichsoperator `..` oder `...` verbunden sind. Alternativ kann sie als Mittelpunkt plus/minus Abweichung angegeben werden. Optionale Sicherheitsindikatoren (`<`, `>` oder `~`) können ebenfalls gespeichert werden. Diese Indikatoren werden von allen eingebauten Operatoren ignoriert. Tabelle F.29 gibt einen Überblick über erlaubte Darstellungen; Tabelle F.30 zeigt Beispiele.

In Tabelle F.29 bezeichnen `x`, `y` und `delta` Gleitkommazahlen. `x` und `y`, aber nicht `delta`, können mit einem Sicherheitsindikator beginnen.

**Tabelle F.29. Externe Darstellungen von `seg`**

| Darstellung | Bedeutung |
|---|---|
| `x` | Einzelwert, also Intervall der Länge null |
| `x .. y` | Intervall von `x` bis `y` |
| `x (+-) delta` | Intervall von `x - delta` bis `x + delta` |
| `x ..` | Offenes Intervall mit unterer Grenze `x` |
| `.. x` | Offenes Intervall mit oberer Grenze `x` |

**Tabelle F.30. Beispiele gültiger `seg`-Eingaben**

| Eingabe | Bedeutung |
|---|---|
| `5.0` | Erzeugt ein Segment der Länge null, also einen Punkt. |
| `~5.0` | Erzeugt ein Segment der Länge null und speichert `~` in den Daten. `~` wird von `seg`-Operationen ignoriert, aber als Kommentar erhalten. |
| `<5.0` | Erzeugt einen Punkt bei `5.0`. `<` wird ignoriert, aber als Kommentar erhalten. |
| `>5.0` | Erzeugt einen Punkt bei `5.0`. `>` wird ignoriert, aber als Kommentar erhalten. |
| `5(+-)0.3` | Erzeugt das Intervall `4.7 .. 5.3`. Die Schreibweise `(+-)` wird nicht erhalten. |
| `50 ..` | Alles, was größer oder gleich `50` ist. |
| `.. 0` | Alles, was kleiner oder gleich `0` ist. |
| `1.5e-2 .. 2E-2` | Erzeugt das Intervall `0.015 .. 0.02`. |
| `1 ... 2` | Dasselbe wie `1...2`, `1 .. 2` oder `1..2`; Leerzeichen um den Bereichsoperator werden ignoriert. |

Da der Operator `...` in Datenquellen weit verbreitet ist, ist er als alternative Schreibweise für `..` erlaubt. Dadurch entsteht allerdings eine Parse-Mehrdeutigkeit: Bei `0...23` ist nicht klar, ob die obere Grenze `23` oder `0.23` gemeint ist. Dies wird aufgelöst, indem in allen Zahlen in `seg`-Eingaben mindestens eine Ziffer vor dem Dezimalpunkt verlangt wird.

Als Plausibilitätsprüfung lehnt `seg` Intervalle ab, deren untere Grenze größer als die obere ist, zum Beispiel `5 .. 2`.

#### F.39.3. Genauigkeit

`seg`-Werte werden intern als Paare von 32-Bit-Gleitkommazahlen gespeichert. Zahlen mit mehr als sieben signifikanten Ziffern werden daher abgeschnitten.

Zahlen mit sieben oder weniger signifikanten Ziffern behalten ihre ursprüngliche Genauigkeit. Wenn Ihre Abfrage also `0.00` zurückgibt, können Sie sicher sein, dass die nachgestellten Nullen keine Formatierungsartefakte sind, sondern die Genauigkeit der ursprünglichen Daten widerspiegeln. Führende Nullen beeinflussen die Genauigkeit nicht: Der Wert `0.0067` gilt als Zahl mit nur zwei signifikanten Ziffern.

#### F.39.4. Verwendung

Das Modul `seg` enthält eine GiST-Index-Operator-Klasse für `seg`-Werte. Die von der GiST-Operator-Klasse unterstützten Operatoren sind in Tabelle F.31 gezeigt.

**Tabelle F.31. GiST-Operatoren für `seg`**

| Operator | Beschreibung |
|---|---|
| `seg << seg → boolean` | Liegt das erste `seg` vollständig links vom zweiten? `[a, b] << [c, d]` ist wahr, wenn `b < c`. |
| `seg >> seg → boolean` | Liegt das erste `seg` vollständig rechts vom zweiten? `[a, b] >> [c, d]` ist wahr, wenn `a > d`. |
| `seg &< seg → boolean` | Erstreckt sich das erste `seg` nicht rechts über das zweite hinaus? `[a, b] &< [c, d]` ist wahr, wenn `b <= d`. |
| `seg &> seg → boolean` | Erstreckt sich das erste `seg` nicht links über das zweite hinaus? `[a, b] &> [c, d]` ist wahr, wenn `a >= c`. |
| `seg = seg → boolean` | Sind die beiden `seg`-Werte gleich? |
| `seg && seg → boolean` | Überlappen sich die beiden `seg`-Werte? |
| `seg @> seg → boolean` | Enthält das erste `seg` das zweite? |
| `seg <@ seg → boolean` | Ist das erste `seg` im zweiten enthalten? |

Zusätzlich stehen für den Typ `seg` die üblichen Vergleichsoperatoren aus Tabelle 9.1 zur Verfügung. Diese Operatoren vergleichen zuerst `a` mit `c` und, falls diese gleich sind, `b` mit `d`. Das ergibt in den meisten Fällen eine brauchbare Sortierung, was für `ORDER BY` mit diesem Typ nützlich ist.

#### F.39.5. Hinweise

Beispiele zur Verwendung finden Sie im Regressionstest `sql/seg.sql`.

Der Mechanismus, der `(+-)` in reguläre Bereiche umwandelt, bestimmt die Anzahl signifikanter Ziffern für die Grenzen nicht vollständig exakt. Er fügt zum Beispiel der unteren Grenze eine zusätzliche Ziffer hinzu, wenn das resultierende Intervall eine Zehnerpotenz enthält:

```sql
postgres=> SELECT '10(+-)1'::seg AS seg;
```

```text
   seg
---------
9.0 .. 11             -- sollte sein: 9 .. 11
```

Die Performance eines R-Tree-Index kann stark von der ursprünglichen Reihenfolge der Eingabewerte abhängen. Es kann sehr hilfreich sein, die Eingabetabelle nach der `seg`-Spalte zu sortieren; ein Beispiel finden Sie im Skript `sort-segments.pl`.

#### F.39.6. Danksagung

Ursprünglicher Autor: Gene Selkov, Jr. <selkovjr@mcs.anl.gov>, Mathematics and Computer Science Division, Argonne National Laboratory.

Mein Dank gilt vor allem Prof. Joe Hellerstein (<https://dsf.berkeley.edu/jmh/>) für die Erläuterung des Kerns von GiST (<http://gist.cs.berkeley.edu/>). Außerdem danke ich allen gegenwärtigen und früheren Postgres-Entwicklern, die es mir ermöglicht haben, meine eigene Welt zu schaffen und unbehelligt darin zu leben. Ebenso danke ich dem Argonne Lab und dem U.S. Department of Energy für die langjährige verlässliche Unterstützung meiner Datenbankforschung.


### F.40. sepgsql — SELinux-basierte Mandatory Access Control (MAC) mit Labels

`sepgsql` ist ein ladbares Modul, das labelbasierte Mandatory Access Control (MAC) auf Basis der SELinux-Sicherheitsrichtlinie unterstützt.

> **Achtung:**
> Die aktuelle Implementierung hat erhebliche Einschränkungen und erzwingt Mandatory Access Control nicht für alle Aktionen. Siehe Abschnitt F.40.7.

#### F.40.1. Überblick

Dieses Modul integriert PostgreSQL mit SELinux und stellt eine zusätzliche Sicherheitsschicht über die normalen PostgreSQL-Prüfungen hinaus bereit. Aus Sicht von SELinux kann PostgreSQL dadurch als User-Space Object Manager fungieren. Jeder Zugriff auf Tabellen oder Funktionen, der durch eine DML-Abfrage ausgelöst wird, wird gegen die Sicherheitsrichtlinie des Systems geprüft. Diese Prüfung erfolgt zusätzlich zur üblichen SQL-Rechteprüfung durch PostgreSQL.

SELinux-Zugriffsentscheidungen beruhen auf Sicherheitslabels, die als Zeichenketten wie `system_u:object_r:sepgsql_table_t:s0` dargestellt werden. Jede Zugriffsentscheidung umfasst zwei Labels: das Label des Subjekts, das die Aktion ausführen möchte, und das Label des Objekts, auf dem die Operation ausgeführt werden soll. Da solche Labels auf beliebige Objektarten angewendet werden können, können Zugriffsentscheidungen für Datenbankobjekte denselben allgemeinen Kriterien unterworfen werden wie Entscheidungen für Dateien oder andere Objekte. Dieses Design soll eine zentrale Sicherheitsrichtlinie ermöglichen, die Informationswerte unabhängig davon schützt, wie diese gespeichert sind.

Die Anweisung `SECURITY LABEL` erlaubt das Zuweisen eines Sicherheitslabels zu einem Datenbankobjekt.

#### F.40.2. Installation

`sepgsql` kann nur unter Linux 2.6.28 oder höher mit aktiviertem SELinux verwendet werden. Auf anderen Plattformen ist es nicht verfügbar. Außerdem benötigen Sie `libselinux` 2.1.10 oder höher und `selinux-policy` 3.9.13 oder höher, wobei manche Distributionen die nötigen Regeln in ältere Policy-Versionen zurückportieren.

Mit dem Befehl `sestatus` können Sie den Status von SELinux prüfen. Eine typische Ausgabe ist:

```text
$ sestatus
SELinux status:                                    enabled
SELinuxfs mount:                                   /selinux
Current mode:                                      enforcing
Mode from config file:                             enforcing
Policy version:                                    24
Policy from config file:                           targeted
```

Wenn SELinux deaktiviert oder nicht installiert ist, müssen Sie es zuerst einrichten, bevor Sie dieses Modul installieren.

Zum Bauen dieses Moduls geben Sie `--with-selinux` an, wenn Sie `make` und Autoconf verwenden, oder `-Dselinux={ auto | enabled | disabled }` bei Meson. Stellen Sie sicher, dass das RPM `libselinux-devel` zur Build-Zeit installiert ist.

Zur Verwendung müssen Sie `sepgsql` in den Parameter `shared_preload_libraries` in `postgresql.conf` aufnehmen. Das Modul funktioniert nicht korrekt, wenn es auf andere Weise geladen wird. Nach dem Laden sollten Sie `sepgsql.sql` in jeder Datenbank ausführen. Dadurch werden die für Sicherheitslabel-Verwaltung benötigten Funktionen installiert und anfängliche Sicherheitslabels vergeben.

Das folgende Beispiel zeigt, wie ein frischer Datenbankcluster mit `sepgsql`-Funktionen und Sicherheitslabels initialisiert wird. Passen Sie die Pfade an Ihre Installation an:

```text
$ export PGDATA=/path/to/data/directory
$ initdb
$ vi $PGDATA/postgresql.conf
```

Ändern Sie:

```text
#shared_preload_libraries = ''                # (change requires restart)
```

in:

```text
shared_preload_libraries = 'sepgsql'          # (change requires restart)
```

und führen Sie aus:

```text
$ for DBNAME in template0 template1 postgres; do
postgres --single -F -c exit_on_error=true $DBNAME \
</usr/local/pgsql/share/contrib/sepgsql.sql >/dev/null
done
```

Je nach Version von `libselinux` und `selinux-policy` können einige oder alle der folgenden Hinweise erscheinen:

```text
/etc/selinux/targeted/contexts/sepgsql_contexts: line 33 has invalid object type db_blobs
/etc/selinux/targeted/contexts/sepgsql_contexts: line 36 has invalid object type db_language
/etc/selinux/targeted/contexts/sepgsql_contexts: line 37 has invalid object type db_language
/etc/selinux/targeted/contexts/sepgsql_contexts: line 38 has invalid object type db_language
/etc/selinux/targeted/contexts/sepgsql_contexts: line 39 has invalid object type db_language
/etc/selinux/targeted/contexts/sepgsql_contexts: line 40 has invalid object type db_language
```

Diese Meldungen sind harmlos und können ignoriert werden. Wenn die Installation ohne Fehler abgeschlossen wurde, können Sie den Server normal starten.

#### F.40.3. Regressionstests

Die Testsuite von `sepgsql` wird ausgeführt, wenn `PG_TEST_EXTRA` `sepgsql` enthält (siehe Abschnitt 31.1.3). Diese Methode eignet sich während der PostgreSQL-Entwicklung. Alternativ gibt es eine Möglichkeit, Tests auszuführen, die prüfen, ob eine Datenbankinstanz korrekt für `sepgsql` eingerichtet wurde.

Wegen der Natur von SELinux erfordert das Ausführen der Regressionstests für `sepgsql` mehrere zusätzliche Konfigurationsschritte, von denen einige als `root` ausgeführt werden müssen.

Die manuellen Tests müssen im Verzeichnis `contrib/sepgsql` eines konfigurierten PostgreSQL-Build-Trees ausgeführt werden. Obwohl sie einen Build-Tree benötigen, sind sie für die Ausführung gegen einen installierten Server gedacht; sie entsprechen also eher `make installcheck` als `make check`.

Richten Sie zuerst `sepgsql` in einer Arbeitsdatenbank gemäß Abschnitt F.40.2 ein. Der aktuelle Betriebssystembenutzer muss sich ohne Passwortauthentifizierung als Superuser mit der Datenbank verbinden können.

Bauen und installieren Sie danach das Policy-Paket für den Regressionstest. Die Policy `sepgsql-regtest` ist ein spezielles Policy-Paket mit Regeln, die während der Regressionstests erlaubt werden müssen. Es wird aus der Policy-Quelldatei `sepgsql-regtest.te` mit `make` und einem von SELinux bereitgestellten Makefile gebaut. Den passenden Makefile-Pfad müssen Sie auf Ihrem System finden; der folgende Pfad ist nur ein Beispiel. Nach dem Build installieren Sie das Policy-Paket mit `semodule`. Wenn es korrekt installiert ist, sollte `semodule -l` `sepgsql-regtest` als verfügbares Policy-Paket auflisten:

```text
$ cd .../contrib/sepgsql
$ make -f /usr/share/selinux/devel/Makefile
$ sudo semodule -u sepgsql-regtest.pp
$ sudo semodule -l | grep sepgsql
sepgsql-regtest 1.07
```

Aktivieren Sie anschließend `sepgsql_regression_test_mode`. Aus Sicherheitsgründen sind die Regeln in `sepgsql-regtest` standardmäßig nicht aktiv. Der Parameter `sepgsql_regression_test_mode` aktiviert die für die Regressionstests nötigen Regeln und kann mit `setsebool` eingeschaltet werden:

```text
$ sudo setsebool sepgsql_regression_test_mode on
$ getsebool sepgsql_regression_test_mode
sepgsql_regression_test_mode --> on
```

Prüfen Sie dann, ob Ihre Shell in der Domäne `unconfined_t` läuft:

```text
$ id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

Details zur Anpassung Ihrer Arbeitsdomäne finden Sie in Abschnitt F.40.8.

Führen Sie schließlich das Regressionstestskript aus:

```text
$ ./test_sepgsql
```

Dieses Skript prüft, ob alle Konfigurationsschritte korrekt ausgeführt wurden, und führt dann die Regressionstests für `sepgsql` aus.

Nach Abschluss der Tests wird empfohlen, den Parameter `sepgsql_regression_test_mode` wieder zu deaktivieren:

```text
$ sudo setsebool sepgsql_regression_test_mode off
```

Sie können auch die Policy `sepgsql-regtest` vollständig entfernen:

```text
$ sudo semodule -r sepgsql-regtest
```

#### F.40.4. GUC-Parameter

`sepgsql.permissive` (`boolean`)

Dieser Parameter ermöglicht es `sepgsql`, unabhängig von der Systemeinstellung im permissiven Modus zu arbeiten. Standard ist `off`. Der Parameter kann nur in `postgresql.conf` oder auf der Serverbefehlszeile gesetzt werden. Bei `on` arbeitet `sepgsql` im permissiven Modus, selbst wenn SELinux allgemein im enforcing-Modus läuft. Der Parameter ist vor allem für Tests nützlich.

`sepgsql.debug_audit` (`boolean`)

Dieser Parameter aktiviert das Ausgeben von Audit-Meldungen unabhängig von den System-Policy-Einstellungen. Standard ist `off`; dann werden Meldungen gemäß Systemeinstellungen ausgegeben. Die SELinux-Sicherheitsrichtlinie enthält ebenfalls Regeln, die steuern, ob bestimmte Zugriffe protokolliert werden. Standardmäßig werden Zugriffsverletzungen protokolliert, erlaubte Zugriffe jedoch nicht. Dieser Parameter erzwingt vollständige Protokollierung unabhängig von der System-Policy.

#### F.40.5. Funktionen

##### F.40.5.1. Kontrollierte Objektklassen

Das Sicherheitsmodell von SELinux beschreibt alle Zugriffskontrollregeln als Beziehungen zwischen einem Subjekt, typischerweise einem Datenbankclient, und einem Objekt, etwa einem Datenbankobjekt. Beide werden durch Sicherheitslabels identifiziert. Wird auf ein Objekt ohne Label zugegriffen, wird es behandelt, als hätte es das Label `unlabeled_t`.

Derzeit erlaubt `sepgsql`, Sicherheitslabels an Schemata, Tabellen, Spalten, Sequenzen, Sichten und Funktionen zu vergeben. Wenn `sepgsql` verwendet wird, werden unterstützten Datenbankobjekten bei ihrer Erstellung automatisch Sicherheitslabels zugewiesen. Dieses Label heißt Default Security Label und wird gemäß der System-Sicherheitsrichtlinie bestimmt, die das Label des Erstellers, das Label des Elternobjekts und optional den Namen des erzeugten Objekts berücksichtigt.

Ein neues Datenbankobjekt erbt grundsätzlich das Sicherheitslabel seines Elternobjekts, außer wenn die Sicherheitsrichtlinie besondere Regeln, sogenannte Type-Transition-Regeln, enthält. In diesem Fall kann ein anderes Label angewendet werden. Für Schemata ist das Elternobjekt die aktuelle Datenbank; für Tabellen, Sequenzen, Sichten und Funktionen das enthaltende Schema; für Spalten die enthaltende Tabelle.

##### F.40.5.2. DML-Berechtigungen

Für Tabellen werden je nach Anweisung `db_table:select`, `db_table:insert`, `db_table:update` oder `db_table:delete` für alle referenzierten Zieltabelle geprüft. Zusätzlich wird `db_table:select` für alle Tabellen geprüft, die Spalten enthalten, die in `WHERE` oder `RETURNING` referenziert werden, etwa als Datenquelle für `UPDATE`.

Spaltenberechtigungen werden ebenfalls für jede referenzierte Spalte geprüft. `db_column:select` wird nicht nur für Spalten geprüft, die mit `SELECT` gelesen werden, sondern auch für Spalten, die in anderen DML-Anweisungen referenziert werden. `db_column:update` oder `db_column:insert` werden für Spalten geprüft, die durch `UPDATE` oder `INSERT` verändert werden.

Beispiel:

```sql
UPDATE t1 SET x = 2, y = func1(y) WHERE z = 100;
```

Hier wird `db_column:update` für `t1.x` geprüft, weil die Spalte aktualisiert wird. Für `t1.y` wird `db_column:{select update}` geprüft, weil die Spalte sowohl aktualisiert als auch referenziert wird. Für `t1.z` wird `db_column:select` geprüft, weil die Spalte nur referenziert wird. Auf Tabellenebene wird außerdem `db_table:{select update}` geprüft.

Für Sequenzen wird `db_sequence:get_value` geprüft, wenn ein Sequenzobjekt mit `SELECT` referenziert wird. Derzeit werden jedoch keine Berechtigungen für die Ausführung entsprechender Funktionen wie `lastval()` geprüft.

Für Sichten wird `db_view:expand` geprüft; anschließend werden alle sonst nötigen Berechtigungen auf den aus der Sicht expandierten Objekten einzeln geprüft.

Für Funktionen wird `db_procedure:{execute}` geprüft, wenn der Benutzer versucht, eine Funktion als Teil einer Abfrage oder über Fast-Path Invocation auszuführen. Wenn die Funktion eine vertrauenswürdige Prozedur ist, wird zusätzlich `db_procedure:{entrypoint}` geprüft.

Für den Zugriff auf ein beliebiges Schemaobjekt ist `db_schema:search` auf dem enthaltenden Schema erforderlich. Wird ein Objekt ohne Schemaqualifikation referenziert, werden Schemata ohne diese Berechtigung nicht durchsucht, ähnlich wie bei fehlendem `USAGE`-Recht. Bei expliziter Schemaqualifikation entsteht ein Fehler, wenn der Benutzer die nötige Berechtigung auf dem genannten Schema nicht besitzt.

Der Client muss auf alle referenzierten Tabellen und Spalten zugreifen dürfen, auch wenn sie aus Sichten stammen, die anschließend expandiert wurden. So werden konsistente Zugriffskontrollregeln unabhängig davon angewendet, wie Tabelleninhalte referenziert werden.

Das normale Datenbank-Rechtesystem erlaubt Datenbank-Superusern, Systemkataloge mit DML-Befehlen zu ändern und TOAST-Tabellen zu referenzieren oder zu ändern. Diese Operationen sind bei aktiviertem `sepgsql` verboten.

##### F.40.5.3. DDL-Berechtigungen

SELinux definiert mehrere Berechtigungen zur Kontrolle üblicher Operationen für jeden Objekttyp, etwa Erstellen, Ändern, Löschen und Umlabeln eines Sicherheitslabels. Zusätzlich besitzen manche Objekttypen besondere Berechtigungen zur Kontrolle charakteristischer Operationen, etwa Hinzufügen oder Entfernen von Namenseinträgen in einem bestimmten Schema.

Das Erzeugen eines neuen Datenbankobjekts erfordert die Berechtigung `create`. SELinux erlaubt oder verweigert diese Berechtigung anhand des Sicherheitslabels des Clients und des vorgeschlagenen Sicherheitslabels für das neue Objekt. In manchen Fällen sind zusätzliche Berechtigungen nötig:

- `CREATE DATABASE` erfordert zusätzlich `getattr` für die Quell- oder Template-Datenbank.
- Das Erzeugen eines Schemaobjekts erfordert zusätzlich `add_name` auf dem Elternschema.
- Das Erzeugen einer Tabelle erfordert zusätzlich die Berechtigung, jede einzelne Tabellenspalte zu erzeugen, als wäre jede Spalte ein separates Top-Level-Objekt.
- Das Erzeugen einer als `LEAKPROOF` markierten Funktion erfordert zusätzlich `install`. Diese Berechtigung wird auch geprüft, wenn `LEAKPROOF` für eine bestehende Funktion gesetzt wird.

Bei `DROP` wird `drop` auf dem zu entfernenden Objekt geprüft. Berechtigungen werden auch für indirekt über `CASCADE` entfernte Objekte geprüft. Das Löschen von Objekten innerhalb eines bestimmten Schemas, etwa Tabellen, Sichten, Sequenzen und Prozeduren, erfordert zusätzlich `remove_name` auf dem Schema.

Bei `ALTER` wird `setattr` auf dem geänderten Objekt geprüft, außer bei untergeordneten Objekten wie Indizes oder Triggern einer Tabelle; dort werden Berechtigungen stattdessen am Elternobjekt geprüft. In manchen Fällen sind zusätzliche Berechtigungen nötig:

- Das Verschieben eines Objekts in ein neues Schema erfordert zusätzlich `remove_name` auf dem alten und `add_name` auf dem neuen Schema.
- Das Setzen des Attributs `LEAKPROOF` auf einer Funktion erfordert `install`.
- `SECURITY LABEL` auf einem Objekt erfordert zusätzlich `relabelfrom` für das Objekt in Verbindung mit seinem alten Sicherheitslabel und `relabelto` in Verbindung mit seinem neuen Sicherheitslabel. Wenn mehrere Label-Provider installiert sind und der Benutzer ein Sicherheitslabel setzen möchte, das nicht von SELinux verwaltet wird, sollte hier nur `setattr` geprüft werden. Dies geschieht derzeit wegen Implementierungsbeschränkungen nicht.

##### F.40.5.4. Vertrauenswürdige Prozeduren

Vertrauenswürdige Prozeduren ähneln Security-Definer-Funktionen oder `setuid`-Befehlen. SELinux erlaubt, vertrauenswürdigen Code mit einem anderen Sicherheitslabel als dem des Clients auszuführen, meist um hoch kontrollierten Zugriff auf sensible Daten bereitzustellen, etwa durch Auslassen von Zeilen oder Reduzieren der Genauigkeit gespeicherter Werte. Ob eine Funktion als vertrauenswürdige Prozedur wirkt, wird durch ihr Sicherheitslabel und die Betriebssystem-Sicherheitsrichtlinie gesteuert. Beispiel:

```sql
postgres=# CREATE TABLE customer (
    cid     int primary key,
    cname   text,
    credit  text
);
CREATE TABLE
postgres=# SECURITY LABEL ON COLUMN customer.credit
IS 'system_u:object_r:sepgsql_secret_table_t:s0';
SECURITY LABEL
postgres=# CREATE FUNCTION show_credit(int) RETURNS text
AS 'SELECT regexp_replace(credit, ''-[0-9]+$'', ''-xxxx'', ''g'')
FROM customer WHERE cid = $1'
LANGUAGE sql;
CREATE FUNCTION
postgres=# SECURITY LABEL ON FUNCTION show_credit(int)
IS 'system_u:object_r:sepgsql_trusted_proc_exec_t:s0';
SECURITY LABEL
```

Die obigen Operationen sollten von einem administrativen Benutzer ausgeführt werden.

```sql
postgres=# SELECT * FROM customer;
```

```text
ERROR: SELinux: security policy violation
```

```sql
postgres=# SELECT cid, cname, show_credit(cid) FROM customer;
```

```text
 cid | cname  |      show_credit
-----+--------+---------------------
 1   | taro   | 1111-2222-3333-xxxx
 2   | hanako | 5555-6666-7777-xxxx
(2 rows)
```

In diesem Fall kann ein normaler Benutzer `customer.credit` nicht direkt referenzieren. Die vertrauenswürdige Prozedur `show_credit` erlaubt aber, Kreditkartennummern der Kunden mit teilweise maskierten Ziffern auszugeben.

##### F.40.5.5. Dynamische Domänenübergänge

Es ist möglich, mit der dynamischen Domänenübergangsfunktion von SELinux das Sicherheitslabel des Client-Prozesses, also die Client-Domäne, auf einen neuen Kontext umzuschalten, wenn die Sicherheitsrichtlinie dies erlaubt. Die Client-Domäne benötigt die Berechtigung `setcurrent` sowie `dyntransition` vom alten zur neuen Domäne.

Dynamische Domänenübergänge sollten sorgfältig betrachtet werden, weil sie Benutzern erlauben, ihr Label und damit ihre Rechte nach eigener Wahl zu ändern, statt wie bei einer vertrauenswürdigen Prozedur durch das System vorgeschrieben. Die Berechtigung `dyntransition` gilt daher nur dann als sicher, wenn sie in eine Domäne mit weniger Rechten als die ursprüngliche wechselt. Beispiel:

```sql
regression=# SELECT sepgsql_getcon();
```

```text
sepgsql_getcon
-------------------------------------------------------
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
(1 row)
```

```sql
regression=# SELECT sepgsql_setcon('unconfined_u:unconfined_r:unconfined_t:s0-s0:c1.c4');
```

```text
sepgsql_setcon
----------------
t
(1 row)
```

```sql
regression=# SELECT sepgsql_setcon('unconfined_u:unconfined_r:unconfined_t:s0-s0:c1.c1023');
```

```text
ERROR: SELinux: security policy violation
```

In diesem Beispiel war der Wechsel vom größeren MCS-Bereich `c1.c1023` zum kleineren Bereich `c1.c4` erlaubt, der Wechsel zurück wurde aber verweigert.

Eine Kombination aus dynamischem Domänenübergang und vertrauenswürdiger Prozedur ermöglicht einen interessanten Anwendungsfall, der zum typischen Lebenszyklus von Connection-Pooling-Software passt. Auch wenn Ihre Connection-Pooling-Software die meisten SQL-Befehle nicht ausführen darf, können Sie ihr erlauben, das Sicherheitslabel des Clients mit `sepgsql_setcon()` innerhalb einer vertrauenswürdigen Prozedur zu ändern. Diese Prozedur sollte Anmeldedaten verlangen, um die Labeländerung zu autorisieren. Danach hat die Sitzung die Rechte des Zielbenutzers statt die des Connection Poolers. Der Connection Pooler kann die Änderung später wieder rückgängig machen, indem er erneut `sepgsql_setcon()` mit `NULL` aufruft, ebenfalls innerhalb einer vertrauenswürdigen Prozedur mit geeigneten Prüfungen. Entscheidend ist, dass nur die vertrauenswürdige Prozedur die Berechtigung besitzt, das effektive Sicherheitslabel zu ändern, und dies nur bei passenden Anmeldedaten tut. Für sicheren Betrieb muss der Speicher dieser Anmeldedaten, also Tabelle, Prozedurdefinition oder Ähnliches, vor unautorisiertem Zugriff geschützt werden.

##### F.40.5.6. Verschiedenes

Der Befehl `LOAD` wird generell abgelehnt, weil jedes geladene Modul die Durchsetzung der Sicherheitsrichtlinie leicht umgehen könnte.

#### F.40.6. `sepgsql`-Funktionen

Tabelle F.32 zeigt die verfügbaren Funktionen.

**Tabelle F.32. Funktionen von `sepgsql`**

| Funktion | Beschreibung |
|---|---|
| `sepgsql_getcon() → text` | Gibt die Client-Domäne zurück, also das aktuelle Sicherheitslabel des Clients. |
| `sepgsql_setcon(text) → boolean` | Wechselt die Client-Domäne der aktuellen Sitzung zur neuen Domäne, wenn die Sicherheitsrichtlinie dies erlaubt. `NULL` bedeutet eine Anfrage zum Übergang in die ursprüngliche Client-Domäne. |
| `sepgsql_mcstrans_in(text) → text` | Übersetzt den angegebenen qualifizierten MLS-/MCS-Bereich in das Rohformat, wenn der `mcstrans`-Daemon läuft. |
| `sepgsql_mcstrans_out(text) → text` | Übersetzt den angegebenen rohen MLS-/MCS-Bereich in das qualifizierte Format, wenn der `mcstrans`-Daemon läuft. |
| `sepgsql_restorecon(text) → boolean` | Richtet anfängliche Sicherheitslabels für alle Objekte in der aktuellen Datenbank ein. Das Argument kann `NULL` sein oder der Name einer Spezifikationsdatei, die statt der Systemvorgabe verwendet wird. |

#### F.40.7. Einschränkungen

- Data Definition Language (DDL) Permissions
  - Wegen Implementierungsbeschränkungen prüfen manche DDL-Operationen keine Berechtigungen.

- Data Control Language (DCL) Permissions
  - Wegen Implementierungsbeschränkungen prüfen DCL-Operationen keine Berechtigungen.

- Row-Level Access Control
  - PostgreSQL unterstützt Zugriffskontrolle auf Zeilenebene, `sepgsql` jedoch nicht.

- Verdeckte Kanäle
  - `sepgsql` versucht nicht, die Existenz eines bestimmten Objekts zu verbergen, selbst wenn der Benutzer es nicht referenzieren darf. Beispielsweise kann man aus Primärschlüsselkonflikten, Fremdschlüsselverletzungen usw. auf die Existenz eines unsichtbaren Objekts schließen, auch wenn dessen Inhalt nicht zugänglich ist. Die Existenz einer streng geheimen Tabelle kann nicht verborgen werden; man kann nur hoffen, ihren Inhalt zu verbergen.

#### F.40.8. Externe Ressourcen

- SE-PostgreSQL Introduction: <https://wiki.postgresql.org/wiki/SEPostgreSQL_Introduction>
  - Diese Wiki-Seite bietet einen kurzen Überblick über Sicherheitsdesign, Architektur, Administration und geplante Funktionen.

- SELinux User's and Administrator's Guide: <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/index>
  - Dieses Dokument vermittelt breites Wissen zur Administration von SELinux-Systemen. Es bezieht sich vor allem auf Red-Hat-Betriebssysteme, ist aber nicht darauf beschränkt.

- Fedora SELinux FAQ: <https://fedoraproject.org/wiki/SELinux_FAQ>
  - Dieses Dokument beantwortet häufige Fragen zu SELinux. Es bezieht sich vor allem auf Fedora, ist aber nicht darauf beschränkt.

#### F.40.9. Autor

KaiGai Kohei <kaigai@ak.jp.nec.com>


### F.41. spi — Funktionen und Beispiele zum Server Programming Interface

Das Modul `spi` stellt mehrere funktionsfähige Beispiele für die Verwendung des Server Programming Interface (SPI) und von Triggern bereit. Die Funktionen sind auch für sich genommen nützlich, vor allem aber als Beispiele, die Sie für eigene Zwecke anpassen können. Sie sind allgemein genug, um mit beliebigen Tabellen verwendet zu werden; Tabellen- und Feldnamen müssen jedoch beim Erzeugen des Triggers angegeben werden.

Jede der unten beschriebenen Funktionsgruppen wird als separat installierbare Erweiterung bereitgestellt.

#### F.41.1. refint — Funktionen zur Implementierung referenzieller Integrität

`check_primary_key()` und `check_foreign_key()` werden verwendet, um Fremdschlüssel-Constraints zu prüfen. Diese Funktionalität ist natürlich längst durch den eingebauten Fremdschlüsselmechanismus ersetzt, das Modul bleibt aber als Beispiel nützlich.

`check_primary_key()` prüft die referenzierende Tabelle. Zur Verwendung erzeugen Sie einen `AFTER INSERT OR UPDATE`-Trigger mit dieser Funktion auf einer Tabelle, die eine andere Tabelle referenziert. Als Triggerargumente geben Sie die Spaltennamen der referenzierenden Tabelle an, die den Fremdschlüssel bilden, den Namen der referenzierten Tabelle und die Spaltennamen in der referenzierten Tabelle, die den Primär- oder Unique-Key bilden. Für mehrere Fremdschlüssel erzeugen Sie je Referenz einen Trigger.

`check_foreign_key()` prüft die referenzierte Tabelle. Zur Verwendung erzeugen Sie einen `AFTER DELETE OR UPDATE`-Trigger mit dieser Funktion auf einer Tabelle, die von anderen Tabellen referenziert wird. Als Triggerargumente geben Sie die Anzahl der referenzierenden Tabellen an, für die die Funktion prüfen soll, die Aktion bei gefundenem referenzierendem Schlüssel (`cascade`, `restrict` oder `setnull`), die Spaltennamen der getriggerten Tabelle, die den Primär- oder Unique-Key bilden, und danach den Namen und die Spaltennamen der referenzierenden Tabelle. Letzteres wird so oft wiederholt, wie im ersten Argument referenzierende Tabellen angegeben wurden. Die Primär- oder Unique-Key-Spalten sollten `NOT NULL` sein und einen eindeutigen Index besitzen.

Wenn diese Trigger aus einem anderen `BEFORE`-Trigger heraus ausgeführt werden, können sie unerwartet fehlschlagen. Fügt ein Benutzer beispielsweise `row1` ein und der `BEFORE`-Trigger fügt `row2` ein und ruft einen Trigger mit `check_foreign_key()` auf, sieht `check_foreign_key()` `row1` nicht und schlägt fehl.

Beispiele finden Sie in `refint.example`.

#### F.41.2. autoinc — Funktionen für automatisch inkrementierte Felder

`autoinc()` ist ein Trigger, der den nächsten Wert einer Sequenz in einem Integer-Feld speichert. Das überschneidet sich mit der eingebauten Funktionalität von „Serial Columns“, ist aber nicht identisch. Der Trigger ersetzt den Feldwert nur, wenn dieser Wert anfänglich null oder `NULL` ist, also nach der Wirkung der SQL-Anweisung, die die Zeile eingefügt oder aktualisiert hat. Wenn der nächste Sequenzwert null ist, wird `nextval()` ein zweites Mal aufgerufen, um einen Nicht-Null-Wert zu erhalten.

Zur Verwendung erzeugen Sie einen `BEFORE INSERT`-Trigger, optional `BEFORE INSERT OR UPDATE`, mit dieser Funktion. Geben Sie zwei Triggerargumente an: den Namen der zu ändernden Integer-Spalte und den Namen des Sequenzobjekts, das Werte liefert. Sie können auch beliebig viele solcher Namenspaare angeben, wenn Sie mehr als eine automatisch inkrementierte Spalte aktualisieren möchten.

Ein Beispiel finden Sie in `autoinc.example`.

#### F.41.3. insert_username — Funktionen zum Nachverfolgen, wer eine Tabelle geändert hat

`insert_username()` ist ein Trigger, der den Namen des aktuellen Benutzers in einem Textfeld speichert. Das kann nützlich sein, um nachzuverfolgen, wer eine bestimmte Zeile zuletzt geändert hat.

Zur Verwendung erzeugen Sie einen `BEFORE INSERT`- und/oder `UPDATE`-Trigger mit dieser Funktion. Geben Sie als einziges Triggerargument den Namen der zu ändernden Textspalte an.

Ein Beispiel finden Sie in `insert_username.example`.

#### F.41.4. moddatetime — Funktionen zum Nachverfolgen der letzten Änderungszeit

`moddatetime()` ist ein Trigger, der die aktuelle Zeit in einem Zeitstempelfeld speichert. Das kann nützlich sein, um die letzte Änderungszeit einer bestimmten Zeile in einer Tabelle nachzuverfolgen.

Zur Verwendung erzeugen Sie einen `BEFORE UPDATE`-Trigger mit dieser Funktion. Geben Sie als einziges Triggerargument den Namen der zu ändernden Spalte an. Die Spalte muss vom Typ `timestamp` oder `timestamp with time zone` sein.

Ein Beispiel finden Sie in `moddatetime.example`.


### F.42. sslinfo — SSL-Informationen des Clients abrufen

Das Modul `sslinfo` liefert Informationen über das SSL-Zertifikat, das der aktuelle Client beim Verbinden mit PostgreSQL bereitgestellt hat. Das Modul ist nutzlos, wenn die aktuelle Verbindung kein SSL verwendet; die meisten Funktionen geben dann `NULL` zurück.

Ein Teil der über dieses Modul verfügbaren Informationen kann auch über die eingebaute Systemsicht `pg_stat_ssl` erhalten werden.

Diese Erweiterung wird nur gebaut, wenn die Installation mit `--with-ssl=openssl` konfiguriert wurde.

#### F.42.1. Bereitgestellte Funktionen

```text
ssl_is_used() returns boolean
```

Gibt `true` zurück, wenn die aktuelle Verbindung zum Server SSL verwendet, andernfalls `false`.

```text
ssl_version() returns text
```

Gibt den Namen des für die SSL-Verbindung verwendeten Protokolls zurück, etwa `TLSv1.0`, `TLSv1.1`, `TLSv1.2` oder `TLSv1.3`.

```text
ssl_cipher() returns text
```

Gibt den Namen des für die SSL-Verbindung verwendeten Ciphers zurück, etwa `DHE-RSA-AES256-SHA`.

```text
ssl_client_cert_present() returns boolean
```

Gibt `true` zurück, wenn der aktuelle Client dem Server ein gültiges SSL-Client-Zertifikat präsentiert hat, andernfalls `false`. Der Server kann so konfiguriert sein, dass er ein Client-Zertifikat verlangt, muss es aber nicht.

```text
ssl_client_serial() returns numeric
```

Gibt die Seriennummer des aktuellen Client-Zertifikats zurück. Die Kombination aus Zertifikatseriennummer und Zertifikatsaussteller identifiziert ein Zertifikat eindeutig, nicht aber seinen Besitzer; Besitzer sollten ihre Schlüssel regelmäßig wechseln und neue Zertifikate vom Aussteller beziehen.

Wenn Sie also Ihre eigene CA betreiben und dem Server nur Zertifikate dieser CA erlauben, ist die Seriennummer das zuverlässigste, wenn auch wenig einprägsame Mittel zur Identifikation eines Benutzers.

```text
ssl_client_dn() returns text
```

Gibt den vollständigen Subject-Namen des aktuellen Client-Zertifikats zurück und konvertiert Zeichendaten in die aktuelle Datenbankkodierung. Wenn Sie Nicht-ASCII-Zeichen in Zertifikatsnamen verwenden, wird angenommen, dass Ihre Datenbank diese Zeichen ebenfalls darstellen kann. Bei `SQL_ASCII` werden Nicht-ASCII-Zeichen im Namen als UTF-8-Sequenzen dargestellt.

Das Ergebnis sieht etwa so aus: `/CN=Somebody /C=Some country/O=Some organization`.

```text
ssl_issuer_dn() returns text
```

Gibt den vollständigen Ausstellernamen des aktuellen Client-Zertifikats zurück und konvertiert Zeichendaten in die aktuelle Datenbankkodierung. Kodierungskonvertierungen werden wie bei `ssl_client_dn()` behandelt.

Die Kombination des Rückgabewerts dieser Funktion mit der Zertifikatseriennummer identifiziert das Zertifikat eindeutig. Diese Funktion ist vor allem dann nützlich, wenn mehr als ein vertrauenswürdiges CA-Zertifikat in der Zertifizierungsstellen-Datei des Servers vorhanden ist oder wenn diese CA Zwischenzertifizierungsstellenzertifikate ausgestellt hat.

```text
ssl_client_dn_field(fieldname text) returns text
```

Diese Funktion gibt den Wert des angegebenen Felds im Zertifikatssubjekt zurück oder `NULL`, wenn das Feld nicht vorhanden ist. Feldnamen sind Zeichenkettenkonstanten, die über die OpenSSL-Objektdatenbank in ASN.1-Objektkennungen konvertiert werden. Zulässig sind unter anderem:

- `commonName` (Alias `CN`)
- `surname` (Alias `SN`)
- `name`
- `givenName` (Alias `GN`)
- `countryName` (Alias `C`)
- `localityName` (Alias `L`)
- `stateOrProvinceName` (Alias `ST`)
- `organizationName` (Alias `O`)
- `organizationalUnitName` (Alias `OU`)
- `title`
- `description`
- `initials`
- `postalCode`
- `streetAddress`
- `generationQualifier`
- `dnQualifier`
- `x500UniqueIdentifier`
- `pseudonym`
- `role`
- `emailAddress`

Alle diese Felder außer `commonName` sind optional. Welche davon enthalten sind, hängt vollständig von der Policy Ihrer CA ab. Die Bedeutung der Felder ist jedoch durch X.500- und X.509-Standards strikt definiert; Sie sollten ihnen daher keine beliebige Bedeutung zuweisen.

```text
ssl_issuer_field(fieldname text) returns text
```

Entspricht `ssl_client_dn_field()`, jedoch für den Zertifikatsaussteller statt für das Zertifikatssubjekt.

```text
ssl_extension_info() returns setof record
```

Liefert Informationen über Erweiterungen des Client-Zertifikats: Name der Erweiterung, Wert der Erweiterung und ob es sich um eine kritische Erweiterung handelt.

#### F.42.2. Autoren

Victor Wagner <vitus@cryptocom.ru>, Cryptocom LTD

Dmitry Voronin <carriingfate92@yandex.ru>

E-Mail der Cryptocom-OpenSSL-Entwicklungsgruppe: <openssl@cryptocom.ru>


### F.43. tablefunc — Funktionen, die Tabellen zurückgeben (crosstab und andere)

Das Modul `tablefunc` enthält verschiedene Funktionen, die Tabellen zurückgeben, also mehrere Zeilen. Diese Funktionen sind sowohl für sich genommen nützlich als auch als Beispiele dafür, wie C-Funktionen geschrieben werden, die mehrere Zeilen zurückgeben.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

#### F.43.1. Bereitgestellte Funktionen

Tabelle F.33 fasst die vom Modul `tablefunc` bereitgestellten Funktionen zusammen.

**Tabelle F.33. Funktionen von `tablefunc`**

| Funktion | Beschreibung |
|---|---|
| `normal_rand(numvals integer, mean float8, stddev float8) → setof float8` | Erzeugt eine Menge normalverteilter Zufallswerte. |
| `crosstab(sql text) → setof record` | Erzeugt eine Pivot-Tabelle mit Zeilennamen und `N` Wertspalten; `N` wird durch den in der aufrufenden Abfrage angegebenen Zeilentyp bestimmt. |
| `crosstabN(sql text) → setof table_crosstab_N` | Erzeugt eine Pivot-Tabelle mit Zeilennamen und `N` Wertspalten. `crosstab2`, `crosstab3` und `crosstab4` sind vordefiniert; weitere `crosstabN`-Funktionen können wie unten beschrieben erzeugt werden. |
| `crosstab(source_sql text, category_sql text) → setof record` | Erzeugt eine Pivot-Tabelle, deren Wertspalten durch eine zweite Abfrage angegeben werden. |
| `crosstab(sql text, N integer) → setof record` | Veraltete Variante von `crosstab(text)`. Der Parameter `N` wird ignoriert, da die Zahl der Wertspalten immer durch die aufrufende Abfrage bestimmt wird. |
| `connectby(relname text, keyid_fld text, parent_keyid_fld text [, orderby_fld text ], start_with text, max_depth integer [, branch_delim text]) → setof record` | Erzeugt eine Darstellung einer hierarchischen Baumstruktur. |

##### F.43.1.1. normal_rand

```text
normal_rand(int numvals, float8 mean, float8 stddev) returns setof float8
```

`normal_rand` erzeugt eine Menge normalverteilter Zufallswerte, also Werte aus einer Gauß-Verteilung.

`numvals` ist die Anzahl der zurückzugebenden Werte. `mean` ist der Mittelwert der Normalverteilung, `stddev` ihre Standardabweichung.

Beispiel für 1000 Werte mit Mittelwert 5 und Standardabweichung 3:

```sql
test=# SELECT * FROM normal_rand(1000, 5, 3);
```

```text
normal_rand
----------------------
1.56556322244898
9.10040991424657
5.36957140345079
-0.369151492880995
0.283600703686639
...
4.82992125404908
9.71308014517282
2.49639286969028
(1000 rows)
```

##### F.43.1.2. crosstab(text)

```text
crosstab(text sql)
crosstab(text sql, int N)
```

Die Funktion `crosstab` erzeugt Pivot-Darstellungen, bei denen Daten quer über die Seite statt untereinander aufgelistet werden. Aus Daten wie:

```text
row1       val11
row1       val12
row1       val13
...
row2       val21
row2       val22
row2       val23
...
```

soll eine Darstellung wie diese entstehen:

```text
row1       val11       val12       val13       ...
row2       val21       val22       val23       ...
...
```

`crosstab` nimmt einen `text`-Parameter entgegen, der eine SQL-Abfrage enthält, die Rohdaten in der ersten Form erzeugt, und gibt eine Tabelle in der zweiten Form zurück.

Der Parameter `sql` ist eine SQL-Anweisung, die die Quelldatenmenge erzeugt. Sie muss eine Spalte `row_name`, eine Kategorie-Spalte und eine Wertspalte zurückgeben. `N` ist ein veralteter Parameter und wird ignoriert.

Eine Quellabfrage könnte beispielsweise Folgendes liefern:

```text
row_name | cat  | value
---------+------+------
row1     | cat1 | val1
row1     | cat2 | val2
row1     | cat3 | val3
row1     | cat4 | val4
row2     | cat1 | val5
row2     | cat2 | val6
row2     | cat3 | val7
row2     | cat4 | val8
```

Da `crosstab` als `setof record` deklariert ist, müssen die tatsächlichen Namen und Typen der Ausgabespalten in der `FROM`-Klausel der aufrufenden `SELECT`-Anweisung definiert werden:

```sql
SELECT *
FROM crosstab('...') AS ct(row_name text, category_1 text, category_2 text);
```

Das ergibt etwa:

```text
row_name | category_1 | category_2
---------+------------+------------
row1     | val1       | val2
row2     | val5       | val6
```

Die `FROM`-Klausel muss die Ausgabe als eine `row_name`-Spalte definieren, deren Datentyp dem der ersten Ergebnisspalte der SQL-Abfrage entspricht, gefolgt von `N` Wertspalten, deren Datentyp dem der dritten Ergebnisspalte entspricht. Die Namen der Ausgabespalten sind frei wählbar.

`crosstab` erzeugt eine Ausgabezeile für jede aufeinanderfolgende Gruppe von Eingabezeilen mit demselben `row_name`-Wert. Die Wertspalten werden von links nach rechts mit den Wertfeldern dieser Zeilen gefüllt. Gibt es weniger Zeilen in einer Gruppe als Ausgabewertspalten, werden die zusätzlichen Ausgabespalten mit `NULL` gefüllt; gibt es mehr Zeilen, werden die zusätzlichen Eingabezeilen übersprungen.

In der Praxis sollte die SQL-Abfrage immer `ORDER BY 1,2` angeben, damit Eingabezeilen korrekt sortiert sind: Werte mit demselben `row_name` stehen zusammen und sind innerhalb der Zeile richtig geordnet. `crosstab` selbst beachtet die zweite Spalte des Abfrageergebnisses nicht; sie dient nur zum Sortieren, um die Reihenfolge zu steuern, in der die Werte der dritten Spalte quer ausgegeben werden.

Vollständiges Beispiel:

```sql
CREATE TABLE ct(id SERIAL, rowid TEXT, attribute TEXT, value TEXT);
INSERT INTO ct(rowid, attribute, value) VALUES('test1','att1','val1');
INSERT INTO ct(rowid, attribute, value) VALUES('test1','att2','val2');
INSERT INTO ct(rowid, attribute, value) VALUES('test1','att3','val3');
INSERT INTO ct(rowid, attribute, value) VALUES('test1','att4','val4');
INSERT INTO ct(rowid, attribute, value) VALUES('test2','att1','val5');
INSERT INTO ct(rowid, attribute, value) VALUES('test2','att2','val6');
INSERT INTO ct(rowid, attribute, value) VALUES('test2','att3','val7');
INSERT INTO ct(rowid, attribute, value) VALUES('test2','att4','val8');

SELECT *
FROM crosstab(
  'select rowid, attribute, value
   from ct
   where attribute = ''att2'' or attribute = ''att3''
   order by 1,2')
AS ct(row_name text, category_1 text, category_2 text, category_3 text);
```

```text
row_name | category_1 | category_2 | category_3
---------+------------+------------+------------
test1    | val2       | val3       |
test2    | val6       | val7       |
(2 rows)
```

Sie können vermeiden, jedes Mal eine `FROM`-Klausel zur Definition der Ausgabespalten zu schreiben, indem Sie eine benutzerdefinierte `crosstab`-Funktion mit fest verdrahtetem Ausgabetyp einrichten. Das wird im nächsten Abschnitt beschrieben. Eine weitere Möglichkeit ist, die erforderliche `FROM`-Klausel in eine View-Definition einzubetten.

> **Hinweis:**
> Siehe auch den Befehl `\crosstabview` in `psql`, der ähnliche Funktionalität wie `crosstab()` bereitstellt.

##### F.43.1.3. crosstabN(text)

```text
crosstabN(text sql)
```

Die Funktionen `crosstabN` sind Beispiele dafür, wie benutzerdefinierte Wrapper für die allgemeine Funktion `crosstab` eingerichtet werden können, damit in der aufrufenden `SELECT`-Abfrage keine Spaltennamen und -typen ausgeschrieben werden müssen. Das Modul `tablefunc` enthält `crosstab2`, `crosstab3` und `crosstab4`; ihre Ausgabezeilentypen sind nach folgendem Muster definiert:

```sql
CREATE TYPE tablefunc_crosstab_N AS (
    row_name TEXT,
    category_1 TEXT,
    category_2 TEXT,
    ...
    category_N TEXT
);
```

Diese Funktionen können direkt verwendet werden, wenn die Eingabeabfrage `row_name`- und Wertspalten vom Typ `text` erzeugt und zwei, drei oder vier Ausgabewertspalten gewünscht sind. Ansonsten verhalten sie sich genau wie die allgemeine Funktion `crosstab`.

Das Beispiel aus dem vorherigen Abschnitt funktioniert daher auch so:

```sql
SELECT *
FROM crosstab3(
  'select rowid, attribute, value
   from ct
   where attribute = ''att2'' or attribute = ''att3''
   order by 1,2');
```

Diese Funktionen dienen hauptsächlich der Illustration. Sie können eigene Rückgabetypen und Funktionen auf Basis der zugrunde liegenden Funktion `crosstab()` erzeugen. Es gibt zwei Wege:

- Einen zusammengesetzten Typ definieren, der die gewünschten Ausgabespalten beschreibt, ähnlich den Beispielen in `contrib/tablefunc/tablefunc--1.0.sql`. Danach definieren Sie eine eindeutige Funktion, die einen `text`-Parameter akzeptiert und `setof your_type_name` zurückgibt, aber auf dieselbe zugrunde liegende C-Funktion `crosstab` verweist.

```sql
CREATE TYPE my_crosstab_float8_5_cols AS (
    my_row_name text,
    my_category_1 float8,
    my_category_2 float8,
    my_category_3 float8,
    my_category_4 float8,
    my_category_5 float8
);

CREATE OR REPLACE FUNCTION crosstab_float8_5_cols(text)
RETURNS setof my_crosstab_float8_5_cols
AS '$libdir/tablefunc','crosstab' LANGUAGE C STABLE STRICT;
```

- `OUT`-Parameter verwenden, um den Rückgabetyp implizit zu definieren:

```sql
CREATE OR REPLACE FUNCTION crosstab_float8_5_cols(
    IN text,
    OUT my_row_name text,
    OUT my_category_1 float8,
    OUT my_category_2 float8,
    OUT my_category_3 float8,
    OUT my_category_4 float8,
    OUT my_category_5 float8)
RETURNS setof record
AS '$libdir/tablefunc','crosstab' LANGUAGE C STABLE STRICT;
```

##### F.43.1.4. crosstab(text, text)

```text
crosstab(text source_sql, text category_sql)
```

Die wichtigste Einschränkung der einparametrigen Form von `crosstab` besteht darin, dass sie alle Werte einer Gruppe gleich behandelt und jeden Wert in die erste verfügbare Spalte einfügt. Wenn Wertspalten bestimmten Datenkategorien entsprechen sollen und manche Gruppen für manche Kategorien keine Daten haben, funktioniert das schlecht. Die zweiparametrige Form löst diesen Fall, indem sie eine explizite Liste der Kategorien bereitstellt, die den Ausgabespalten entsprechen.

`source_sql` ist eine SQL-Anweisung, die die Quelldatenmenge erzeugt. Sie muss eine `row_name`-Spalte, eine Kategorie-Spalte und eine Wertspalte zurückgeben. Zusätzlich darf sie eine oder mehrere „Extra“-Spalten enthalten. `row_name` muss die erste Spalte sein. Kategorie und Wert müssen die letzten beiden Spalten in dieser Reihenfolge sein. Alle Spalten zwischen `row_name` und Kategorie gelten als „Extra“-Spalten und sollten für alle Zeilen mit demselben `row_name`-Wert gleich sein.

Beispiel für eine mögliche Quellabfrage:

```sql
SELECT row_name, extra_col, cat, value FROM foo ORDER BY 1;
```

```text
row_name | extra_col | cat  | value
---------+-----------+------+------
row1     | extra1    | cat1 | val1
row1     | extra1    | cat2 | val2
row1     | extra1    | cat4 | val4
row2     | extra2    | cat1 | val5
row2     | extra2    | cat2 | val6
row2     | extra2    | cat3 | val7
row2     | extra2    | cat4 | val8
```

`category_sql` ist eine SQL-Anweisung, die die Kategorienmenge erzeugt. Sie darf nur eine Spalte zurückgeben, muss mindestens eine Zeile erzeugen und darf keine Duplikate liefern. Beispiel:

```sql
SELECT DISTINCT cat FROM foo ORDER BY 1;
```

```text
 cat
------
cat1
cat2
cat3
cat4
```

Auch diese `crosstab`-Form ist als `setof record` deklariert, daher müssen Namen und Typen der Ausgabespalten in der `FROM`-Klausel angegeben werden:

```sql
SELECT * FROM crosstab('...', '...')
AS ct(row_name text, extra text, cat1 text, cat2 text, cat3 text, cat4 text);
```

Die `FROM`-Klausel muss die richtige Anzahl von Ausgabespalten mit passenden Datentypen definieren. Wenn das Ergebnis von `source_sql` `N` Spalten enthält, müssen die ersten `N-2` davon zu den ersten `N-2` Ausgabespalten passen. Die übrigen Ausgabespalten müssen den Typ der letzten Spalte aus `source_sql` haben, und es muss genau so viele von ihnen geben, wie `category_sql` Zeilen liefert.

`crosstab` erzeugt eine Ausgabezeile für jede aufeinanderfolgende Gruppe von Eingabezeilen mit demselben `row_name`-Wert. Die `row_name`-Spalte und etwaige Extra-Spalten werden aus der ersten Zeile der Gruppe kopiert. Die Wertspalten werden mit den Werten aus Zeilen gefüllt, deren Kategorie zu einer Ausgabekategorie passt. Kategorien ohne passende Eingabezeile ergeben `NULL`; nicht passende Eingabekategorien werden ignoriert.

In der Praxis sollte `source_sql` immer `ORDER BY 1` angeben, damit Werte mit demselben `row_name` zusammenstehen. Die Reihenfolge der Kategorien innerhalb einer Gruppe ist nicht wichtig. Entscheidend ist aber, dass die Ausgabe von `category_sql` in derselben Reihenfolge vorliegt wie die angegebenen Ausgabespalten.

Beispiel mit Monatsumsätzen:

```sql
CREATE TABLE sales(year int, month int, qty int);
INSERT INTO sales VALUES(2007, 1, 1000);
INSERT INTO sales VALUES(2007, 2, 1500);
INSERT INTO sales VALUES(2007, 7, 500);
INSERT INTO sales VALUES(2007, 11, 1500);
INSERT INTO sales VALUES(2007, 12, 2000);
INSERT INTO sales VALUES(2008, 1, 1000);

SELECT * FROM crosstab(
  'select year, month, qty from sales order by 1',
  'select m from generate_series(1,12) m'
) AS (
  year int,
  "Jan" int, "Feb" int, "Mar" int, "Apr" int,
  "May" int, "Jun" int, "Jul" int, "Aug" int,
  "Sep" int, "Oct" int, "Nov" int, "Dec" int
);
```

```text
year | Jan  | Feb  | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | Nov  | Dec
-----+------+------+-----+-----+-----+-----+-----+-----+-----+-----+------+------
2007 | 1000 | 1500 |     |     |     |     | 500 |     |     |     | 1500 | 2000
2008 | 1000 |      |     |     |     |     |     |     |     |     |      |
(2 rows)
```

Beispiel mit Extra-Spalten:

```sql
CREATE TABLE cth(rowid text, rowdt timestamp, attribute text, val text);
INSERT INTO cth VALUES('test1','01 March 2003','temperature','42');
INSERT INTO cth VALUES('test1','01 March 2003','test_result','PASS');
INSERT INTO cth VALUES('test1','01 March 2003','volts','2.6987');
INSERT INTO cth VALUES('test2','02 March 2003','temperature','53');
INSERT INTO cth VALUES('test2','02 March 2003','test_result','FAIL');
INSERT INTO cth VALUES('test2','02 March 2003','test_startdate','01 March 2003');
INSERT INTO cth VALUES('test2','02 March 2003','volts','3.1234');

SELECT * FROM crosstab(
  'SELECT rowid, rowdt, attribute, val FROM cth ORDER BY 1',
  'SELECT DISTINCT attribute FROM cth ORDER BY 1'
) AS (
  rowid text,
  rowdt timestamp,
  temperature int4,
  test_result text,
  test_startdate timestamp,
  volts float8
);
```

```text
rowid |          rowdt           | temperature | test_result |      test_startdate      | volts
------+--------------------------+-------------+-------------+--------------------------+--------
test1 | Sat Mar 01 00:00:00 2003 |          42 | PASS        |                          | 2.6987
test2 | Sun Mar 02 00:00:00 2003 |          53 | FAIL        | Sat Mar 01 00:00:00 2003 | 3.1234
(2 rows)
```

Sie können vordefinierte Funktionen erzeugen, um die Spaltennamen und -typen des Ergebnisses nicht in jeder Abfrage ausschreiben zu müssen. Beispiele stehen im vorherigen Abschnitt. Die zugrunde liegende C-Funktion für diese Form von `crosstab` heißt `crosstab_hash`.

##### F.43.1.5. connectby

```text
connectby(text relname, text keyid_fld, text parent_keyid_fld
          [, text orderby_fld ], text start_with, int max_depth
          [, text branch_delim ])
```

Die Funktion `connectby` erzeugt eine Darstellung hierarchischer Daten, die in einer Tabelle gespeichert sind. Die Tabelle muss ein Schlüsselfeld besitzen, das Zeilen eindeutig identifiziert, und ein Elternschlüsselfeld, das auf den Elternknoten einer Zeile verweist, sofern vorhanden. `connectby` kann den Teilbaum unterhalb einer beliebigen Zeile anzeigen.

Tabelle F.34 beschreibt die Parameter.

**Tabelle F.34. Parameter von `connectby`**

| Parameter | Beschreibung |
|---|---|
| `relname` | Name der Quellrelation |
| `keyid_fld` | Name des Schlüsselfelds |
| `parent_keyid_fld` | Name des Elternschlüsselfelds |
| `orderby_fld` | Name des Felds, nach dem Geschwister sortiert werden sollen; optional |
| `start_with` | Schlüsselwert der Startzeile |
| `max_depth` | Maximale Tiefe, bis zu der abgestiegen wird, oder null für unbegrenzte Tiefe |
| `branch_delim` | Zeichenkette zur Trennung von Schlüsseln in der Branch-Ausgabe; optional |

Schlüssel- und Elternschlüsselfeld können beliebige Datentypen haben, müssen aber denselben Typ besitzen. Der Wert `start_with` muss unabhängig vom Typ des Schlüsselfelds als Textzeichenkette eingegeben werden.

Da `connectby` als `setof record` deklariert ist, müssen Namen und Typen der Ausgabespalten in der `FROM`-Klausel der aufrufenden `SELECT`-Anweisung definiert werden:

```sql
SELECT * FROM connectby('connectby_tree', 'keyid', 'parent_keyid',
                        'pos', 'row2', 0, '~')
AS t(keyid text, parent_keyid text, level int, branch text, pos int);
```

Die ersten beiden Ausgabespalten werden für den Schlüssel der aktuellen Zeile und den Schlüssel ihrer Elternzeile verwendet und müssen zum Typ des Tabellenschlüsselfelds passen. Die dritte Ausgabespalte ist die Tiefe im Baum und muss vom Typ `integer` sein. Wenn `branch_delim` angegeben wurde, folgt die Branch-Ausgabe als `text`. Wenn `orderby_fld` angegeben wurde, ist die letzte Ausgabespalte eine laufende Nummer vom Typ `integer`.

Die Spalte `branch` zeigt den Pfad von Schlüsseln bis zur aktuellen Zeile, getrennt durch `branch_delim`. Wenn keine Branch-Ausgabe gewünscht ist, lassen Sie sowohl den Parameter `branch_delim` als auch die Spalte `branch` in der Ausgabespaltenliste weg.

Wenn die Reihenfolge von Geschwistern wichtig ist, geben Sie `orderby_fld` an. Dieses Feld kann einen beliebigen sortierbaren Datentyp haben. Die Ausgabespaltenliste muss genau dann eine letzte Integer-Spalte für die laufende Nummer enthalten, wenn `orderby_fld` angegeben ist.

Tabellen- und Feldnamen werden unverändert in die SQL-Abfragen kopiert, die `connectby` intern erzeugt. Verwenden Sie daher doppelte Anführungszeichen, wenn Namen gemischte Groß-/Kleinschreibung oder Sonderzeichen enthalten. Gegebenenfalls muss auch der Tabellenname schemaqualifiziert werden.

Bei großen Tabellen ist die Performance schlecht, sofern kein Index auf dem Elternschlüsselfeld vorhanden ist.

Wichtig ist, dass die Zeichenkette `branch_delim` in keinem Schlüsselwert vorkommt, sonst kann `connectby` fälschlich eine Endlosrekursion melden. Wenn `branch_delim` nicht angegeben wird, wird zur Rekursionserkennung der Standardwert `~` verwendet.

Beispiel:

```sql
CREATE TABLE connectby_tree(keyid text, parent_keyid text, pos int);

INSERT INTO connectby_tree VALUES('row1',NULL, 0);
INSERT INTO connectby_tree VALUES('row2','row1', 0);
INSERT INTO connectby_tree VALUES('row3','row1', 0);
INSERT INTO connectby_tree VALUES('row4','row2', 1);
INSERT INTO connectby_tree VALUES('row5','row2', 0);
INSERT INTO connectby_tree VALUES('row6','row4', 0);
INSERT INTO connectby_tree VALUES('row7','row3', 0);
INSERT INTO connectby_tree VALUES('row8','row6', 0);
INSERT INTO connectby_tree VALUES('row9','row5', 0);
```

Mit Branch-Ausgabe, ohne `orderby_fld`; die Ergebnisreihenfolge ist nicht garantiert:

```sql
SELECT * FROM connectby('connectby_tree', 'keyid', 'parent_keyid', 'row2', 0, '~')
AS t(keyid text, parent_keyid text, level int, branch text);
```

```text
keyid | parent_keyid | level |       branch
------+--------------+-------+---------------------
row2  |              |     0 | row2
row4  | row2         |     1 | row2~row4
row6  | row4         |     2 | row2~row4~row6
row8  | row6         |     3 | row2~row4~row6~row8
row5  | row2         |     1 | row2~row5
row9  | row5         |     2 | row2~row5~row9
(6 rows)
```

Ohne Branch-Ausgabe, ohne `orderby_fld`:

```sql
SELECT * FROM connectby('connectby_tree', 'keyid', 'parent_keyid', 'row2', 0)
AS t(keyid text, parent_keyid text, level int);
```

```text
keyid | parent_keyid | level
------+--------------+------
row2  |              |     0
row4  | row2         |     1
row6  | row4         |     2
row8  | row6         |     3
row5  | row2         |     1
row9  | row5         |     2
(6 rows)
```

Mit Branch-Ausgabe und `orderby_fld`, wobei `row5` vor `row4` kommt:

```sql
SELECT * FROM connectby('connectby_tree', 'keyid', 'parent_keyid',
                        'pos', 'row2', 0, '~')
AS t(keyid text, parent_keyid text, level int, branch text, pos int);
```

```text
keyid | parent_keyid | level |       branch        | pos
------+--------------+-------+---------------------+----
row2  |              |     0 | row2                |   1
row5  | row2         |     1 | row2~row5           |   2
row9  | row5         |     2 | row2~row5~row9      |   3
row4  | row2         |     1 | row2~row4           |   4
row6  | row4         |     2 | row2~row4~row6      |   5
row8  | row6         |     3 | row2~row4~row6~row8 |   6
(6 rows)
```

Ohne Branch-Ausgabe, mit `orderby_fld`:

```sql
SELECT * FROM connectby('connectby_tree', 'keyid', 'parent_keyid',
                        'pos', 'row2', 0)
AS t(keyid text, parent_keyid text, level int, pos int);
```

```text
keyid | parent_keyid | level | pos
------+--------------+-------+----
row2  |              |     0 |   1
row5  | row2         |     1 |   2
row9  | row5         |     2 |   3
row4  | row2         |     1 |   4
row6  | row4         |     2 |   5
row8  | row6         |     3 |   6
(6 rows)
```

#### F.43.2. Autor

Joe Conway


### F.44. tcn — Triggerfunktion für Benachrichtigungen über Änderungen an Tabelleninhalten

Das Modul `tcn` stellt eine Triggerfunktion bereit, die Listener über Änderungen an jeder Tabelle benachrichtigt, an die sie angehängt ist. Sie muss als `AFTER`-Trigger `FOR EACH ROW` verwendet werden.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

In einer `CREATE TRIGGER`-Anweisung darf der Funktion höchstens ein Parameter übergeben werden; dieser ist optional. Wenn er angegeben wird, dient er als Kanalname für die Benachrichtigungen. Wird er weggelassen, wird `tcn` als Kanalname verwendet.

Die Payload der Benachrichtigungen besteht aus dem Tabellennamen, einem Buchstaben für die Art der Operation und Spaltenname/Wert-Paaren für die Primärschlüsselspalten. Die Teile werden jeweils durch Kommas getrennt. Zur einfacheren Verarbeitung mit regulären Ausdrücken werden Tabellen- und Spaltennamen immer in doppelte Anführungszeichen gesetzt, Datenwerte immer in einfache Anführungszeichen. Eingebettete Anführungszeichen werden verdoppelt.

Kurzes Beispiel:

```sql
test=# CREATE TABLE tcndata
       (
          a int not null,
          b date not null,
          c text,
          primary key (a, b)
       );
test=# CREATE TRIGGER tcndata_tcn_trigger
       AFTER INSERT OR UPDATE OR DELETE ON tcndata
       FOR EACH ROW EXECUTE FUNCTION triggered_change_notification();
test=# LISTEN tcn;
test=# INSERT INTO tcndata VALUES (1, date '2012-12-22', 'one'),
                                  (1, date '2012-12-23', 'another'),
                                  (2, date '2012-12-23', 'two');
```

```text
CREATE TABLE
CREATE TRIGGER
LISTEN
INSERT 0 3
Asynchronous notification "tcn" with payload ""tcndata",I,"a"='1',"b"='2012-12-22'" received from server process with PID 22770.
Asynchronous notification "tcn" with payload ""tcndata",I,"a"='1',"b"='2012-12-23'" received from server process with PID 22770.
Asynchronous notification "tcn" with payload ""tcndata",I,"a"='2',"b"='2012-12-23'" received from server process with PID 22770.
```

```sql
test=# UPDATE tcndata SET c = 'uno' WHERE a = 1;
test=# DELETE FROM tcndata WHERE a = 1 AND b = date '2012-12-22';
```

```text
UPDATE 2
Asynchronous notification "tcn" with payload ""tcndata",U,"a"='1',"b"='2012-12-22'" received from server process with PID 22770.
Asynchronous notification "tcn" with payload ""tcndata",U,"a"='1',"b"='2012-12-23'" received from server process with PID 22770.
DELETE 1
Asynchronous notification "tcn" with payload ""tcndata",D,"a"='1',"b"='2012-12-22'" received from server process with PID 22770.
```


### F.45. test_decoding — SQL-basiertes Test- und Beispielmodul für logische WAL-Decodierung

`test_decoding` ist ein Beispiel für ein Ausgabe-Plugin der logischen Decodierung. Es tut nichts besonders Nützliches, kann aber als Ausgangspunkt für die Entwicklung eigener Ausgabe-Plugins dienen.

`test_decoding` empfängt WAL über den Mechanismus der logischen Decodierung und decodiert es in Textdarstellungen der ausgeführten Operationen.

Typische Ausgabe dieses Plugins über die SQL-Schnittstelle der logischen Decodierung könnte so aussehen:

```sql
postgres=# SELECT * FROM pg_logical_slot_get_changes('test_slot',
           NULL, NULL, 'include-xids', '0');
```

```text
    lsn    | xid |                       data
-----------+-----+--------------------------------------------------
0/16D30F8 | 691 | BEGIN
0/16D32A0 | 691 | table public.data: INSERT: id[int4]:2 data[text]:'arg'
0/16D32A0 | 691 | table public.data: INSERT: id[int4]:3 data[text]:'demo'
0/16D32A0 | 691 | COMMIT
0/16D32D8 | 692 | BEGIN
0/16D3398 | 692 | table public.data: DELETE: id[int4]:2
0/16D3398 | 692 | table public.data: DELETE: id[int4]:3
0/16D3398 | 692 | COMMIT
(8 rows)
```

Auch Änderungen einer laufenden Transaktion können abgefragt werden. Eine typische Ausgabe könnte sein:

```sql
postgres[33712]=#* SELECT *
FROM pg_logical_slot_get_changes('test_slot', NULL, NULL, 'streamchanges', '1');
```

```text
   lsn    | xid |                       data
----------+-----+--------------------------------------------------
0/16B21F8 | 503 | opening a streamed block for transaction TXN 503
0/16B21F8 | 503 | streaming change for TXN 503
0/16B2300 | 503 | streaming change for TXN 503
0/16B2408 | 503 | streaming change for TXN 503
0/16BEBA0 | 503 | closing a streamed block for transaction TXN 503
...
(10 rows)
```


### F.46. tsm_system_rows — Sampling-Methode SYSTEM_ROWS für TABLESAMPLE

Das Modul `tsm_system_rows` stellt die Tabellen-Sampling-Methode `SYSTEM_ROWS` bereit, die in der `TABLESAMPLE`-Klausel eines `SELECT`-Befehls verwendet werden kann.

Diese Sampling-Methode akzeptiert ein einzelnes Integer-Argument: die maximale Anzahl zu lesender Zeilen. Das resultierende Sample enthält immer genau so viele Zeilen, sofern die Tabelle genügend Zeilen enthält; andernfalls wird die gesamte Tabelle ausgewählt.

Wie die eingebaute Sampling-Methode `SYSTEM` arbeitet `SYSTEM_ROWS` auf Blockebene. Das Sample ist daher nicht vollständig zufällig und kann Cluster-Effekten unterliegen, insbesondere wenn nur wenige Zeilen angefordert werden.

`SYSTEM_ROWS` unterstützt die Klausel `REPEATABLE` nicht.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

#### F.46.1. Beispiele

Beispiel für das Auswählen eines Samples mit `SYSTEM_ROWS`. Zuerst wird die Erweiterung installiert:

```sql
CREATE EXTENSION tsm_system_rows;
```

Danach kann sie in einem `SELECT`-Befehl verwendet werden:

```sql
SELECT * FROM my_table TABLESAMPLE SYSTEM_ROWS(100);
```

Dieser Befehl gibt ein Sample von 100 Zeilen aus `my_table` zurück, sofern die Tabelle mindestens 100 sichtbare Zeilen enthält; andernfalls werden alle Zeilen zurückgegeben.


### F.47. tsm_system_time — Sampling-Methode SYSTEM_TIME für TABLESAMPLE

Das Modul `tsm_system_time` stellt die Tabellen-Sampling-Methode `SYSTEM_TIME` bereit, die in der `TABLESAMPLE`-Klausel eines `SELECT`-Befehls verwendet werden kann.

Diese Sampling-Methode akzeptiert ein einzelnes Gleitkommaargument: die maximale Anzahl von Millisekunden, die zum Lesen der Tabelle aufgewendet werden soll. Dadurch steuern Sie direkt die Abfragedauer, allerdings wird die Größe des Samples schwer vorhersehbar. Das resultierende Sample enthält so viele Zeilen, wie in der angegebenen Zeit gelesen werden konnten, sofern nicht vorher die ganze Tabelle gelesen wurde.

Wie die eingebaute Sampling-Methode `SYSTEM` arbeitet `SYSTEM_TIME` auf Blockebene. Das Sample ist daher nicht vollständig zufällig und kann Cluster-Effekten unterliegen, insbesondere wenn nur wenige Zeilen ausgewählt werden.

`SYSTEM_TIME` unterstützt die Klausel `REPEATABLE` nicht.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

#### F.47.1. Beispiele

Beispiel für das Auswählen eines Samples mit `SYSTEM_TIME`. Zuerst wird die Erweiterung installiert:

```sql
CREATE EXTENSION tsm_system_time;
```

Danach kann sie in einem `SELECT`-Befehl verwendet werden:

```sql
SELECT * FROM my_table TABLESAMPLE SYSTEM_TIME(1000);
```

Dieser Befehl gibt ein möglichst großes Sample aus `my_table` zurück, das innerhalb von einer Sekunde, also 1000 Millisekunden, gelesen werden kann. Wenn die gesamte Tabelle in weniger als einer Sekunde gelesen werden kann, werden alle Zeilen zurückgegeben.


### F.48. unaccent — Textsuche-Wörterbuch zum Entfernen diakritischer Zeichen

`unaccent` ist ein Textsuche-Wörterbuch, das Akzente, also diakritische Zeichen, aus Lexemen entfernt. Es ist ein Filterwörterbuch: Seine Ausgabe wird immer an das nächste Wörterbuch weitergereicht, sofern eines vorhanden ist. Dadurch wird akzentunempfindliche Verarbeitung für die Volltextsuche möglich.

Die aktuelle Implementierung von `unaccent` kann nicht als normalisierendes Wörterbuch für das Thesaurus-Wörterbuch verwendet werden.

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

#### F.48.1. Konfiguration

Ein `unaccent`-Wörterbuch akzeptiert folgende Option:

- `RULES`: Basisname der Datei mit der Liste der Übersetzungsregeln. Diese Datei muss in `$SHAREDIR/tsearch_data/` gespeichert sein, wobei `$SHAREDIR` das Shared-Data-Verzeichnis der PostgreSQL-Installation bezeichnet. Ihr Name muss mit `.rules` enden; diese Endung wird im Parameter `RULES` nicht angegeben.

Die Regeldatei hat folgendes Format:

- Jede Zeile beschreibt eine Übersetzungsregel, bestehend aus einem Zeichen mit Akzent, gefolgt von einem Zeichen ohne Akzent. Das erste wird in das zweite übersetzt. Beispiel:

```text
À             A
Á             A
Â             A
Ã             A
Ä             A
Å             A
Æ             AE
```

- Die beiden Zeichen müssen durch Whitespace getrennt sein; führender und nachgestellter Whitespace wird ignoriert.
- Wenn nur ein Zeichen in einer Zeile angegeben wird, werden Vorkommen dieses Zeichens gelöscht. Das ist für Sprachen nützlich, in denen Akzente durch separate Zeichen dargestellt werden.
- Ein „Zeichen“ kann tatsächlich jede Zeichenkette ohne Whitespace sein. `unaccent`-Wörterbücher können daher auch für andere Arten von Teilzeichenkettenersetzungen verwendet werden.
- Manche Zeichen, etwa numerische Symbole, benötigen Whitespace in ihrer Übersetzungsregel. In diesem Fall können doppelte Anführungszeichen um die übersetzten Zeichen verwendet werden. Ein doppeltes Anführungszeichen innerhalb der Übersetzung wird durch ein zweites doppeltes Anführungszeichen escaped.

```text
¼          " 1/4"
½          " 1/2"
¾          " 3/4"
“          """"
”          """"
```

Wie andere PostgreSQL-Textsuche-Konfigurationsdateien muss die Regeldatei in UTF-8 gespeichert sein. Die Daten werden beim Laden automatisch in die aktuelle Datenbankkodierung übersetzt. Zeilen mit nicht übersetzbaren Zeichen werden stillschweigend ignoriert, sodass Regeldateien auch Regeln enthalten können, die für die aktuelle Kodierung nicht gelten.

Ein vollständigeres Beispiel, das für die meisten europäischen Sprachen direkt nützlich ist, findet sich in `unaccent.rules`. Diese Datei wird bei Installation des Moduls `unaccent` in `$SHAREDIR/tsearch_data/` installiert. Sie übersetzt Zeichen mit Akzenten in dieselben Zeichen ohne Akzente und erweitert außerdem Ligaturen in entsprechende Folgen einfacher Zeichen, etwa `Æ` zu `AE`.

#### F.48.2. Verwendung

Die Installation der Erweiterung `unaccent` erzeugt ein Textsuche-Template `unaccent` und ein darauf basierendes Wörterbuch `unaccent`. Das Wörterbuch hat standardmäßig `RULES='unaccent'` und ist damit sofort mit der Standarddatei `unaccent.rules` verwendbar. Bei Bedarf können Sie den Parameter ändern:

```sql
mydb=# ALTER TEXT SEARCH DICTIONARY unaccent (RULES='my_rules');
```

oder neue Wörterbücher auf Basis des Templates erzeugen.

Zum Testen des Wörterbuchs:

```sql
mydb=# SELECT ts_lexize('unaccent','Hôtel');
```

```text
 ts_lexize
-----------
 {Hotel}
(1 row)
```

Beispiel zum Einfügen des Wörterbuchs `unaccent` in eine Textsuche-Konfiguration:

```sql
mydb=# CREATE TEXT SEARCH CONFIGURATION fr ( COPY = french );
mydb=# ALTER TEXT SEARCH CONFIGURATION fr
       ALTER MAPPING FOR hword, hword_part, word
       WITH unaccent, french_stem;
mydb=# SELECT to_tsvector('fr','Hôtels de la Mer');
```

```text
   to_tsvector
-------------------
'hotel':1 'mer':4
(1 row)
```

```sql
mydb=# SELECT to_tsvector('fr','Hôtel de la Mer') @@ to_tsquery('fr','Hotels');
```

```text
 ?column?
----------
 t
(1 row)
```

```sql
mydb=# SELECT ts_headline('fr','Hôtel de la Mer',to_tsquery('fr','Hotels'));
```

```text
      ts_headline
------------------------
<b>Hôtel</b> de la Mer
(1 row)
```

#### F.48.3. Funktionen

Die Funktion `unaccent()` entfernt Akzente, also diakritische Zeichen, aus einer Zeichenkette. Im Kern ist sie ein Wrapper um Wörterbücher des Typs `unaccent`, kann aber auch außerhalb normaler Textsuche-Kontexte verwendet werden.

```text
unaccent([dictionary regdictionary, ] string text) returns text
```

Wenn das Argument `dictionary` weggelassen wird, wird das Textsuche-Wörterbuch namens `unaccent` verwendet, das im selben Schema wie die Funktion `unaccent()` selbst liegt.

Beispiel:

```sql
SELECT unaccent('unaccent', 'Hôtel');
SELECT unaccent('Hôtel');
```


### F.49. uuid-ossp — UUID-Generator

Das Modul `uuid-ossp` stellt Funktionen bereit, um Universally Unique Identifiers (UUIDs) mit mehreren Standardalgorithmen zu erzeugen. Außerdem gibt es Funktionen zur Erzeugung bestimmter spezieller UUID-Konstanten. Dieses Modul ist nur für spezielle Anforderungen nötig, die über die Kernfunktionalität von PostgreSQL hinausgehen. Eingebaute Möglichkeiten zur UUID-Erzeugung finden Sie in [Abschnitt 9.14](09_Funktionen_und_Operatoren.md#914-uuidfunktionen).

Dieses Modul gilt als „trusted“, das heißt: Es kann von Nicht-Superusern installiert werden, die das `CREATE`-Recht in der aktuellen Datenbank besitzen.

#### F.49.1. `uuid-ossp`-Funktionen

Tabelle F.35 zeigt die verfügbaren Funktionen zur UUID-Erzeugung. Die relevanten Standards ITU-T Rec. X.667, ISO/IEC 9834-8:2005 und [RFC 4122](https://datatracker.ietf.org/doc/html/rfc4122) beschreiben vier Algorithmen zur UUID-Erzeugung mit den Versionsnummern 1, 3, 4 und 5. Es gibt keinen Version-2-Algorithmus. Jeder dieser Algorithmen kann für andere Anwendungen geeignet sein.

**Tabelle F.35. Funktionen zur UUID-Erzeugung**

| Funktion | Beschreibung |
|---|---|
| `uuid_generate_v1() → uuid` | Erzeugt eine Version-1-UUID. Dabei werden die MAC-Adresse des Computers und ein Zeitstempel verwendet. Solche UUIDs offenbaren die Identität des erzeugenden Computers und den Erzeugungszeitpunkt und können daher für sicherheitssensible Anwendungen ungeeignet sein. |
| `uuid_generate_v1mc() → uuid` | Erzeugt eine Version-1-UUID, verwendet aber statt der echten MAC-Adresse des Computers eine zufällige Multicast-MAC-Adresse. |
| `uuid_generate_v3(namespace uuid, name text) → uuid` | Erzeugt eine Version-3-UUID im angegebenen Namespace mit dem angegebenen Namen. Der Namespace sollte eine der von `uuid_ns_*()` erzeugten Konstanten aus Tabelle F.36 sein, kann theoretisch aber jede UUID sein. Der Name ist ein Bezeichner im gewählten Namespace. Der Name wird per MD5 gehasht, sodass der Klartext nicht aus der erzeugten UUID abgeleitet werden kann. UUIDs dieser Methode sind reproduzierbar und enthalten kein Zufalls- oder Umgebungselement. |
| `uuid_generate_v4() → uuid` | Erzeugt eine Version-4-UUID, die vollständig aus Zufallszahlen abgeleitet wird. |
| `uuid_generate_v5(namespace uuid, name text) → uuid` | Erzeugt eine Version-5-UUID. Sie funktioniert wie Version 3, verwendet aber SHA-1 als Hashmethode. Version 5 sollte Version 3 vorgezogen werden, weil SHA-1 als sicherer gilt als MD5. |

Beispiel:

```sql
SELECT uuid_generate_v3(uuid_ns_url(), 'http://www.postgresql.org');
```

**Tabelle F.36. Funktionen, die UUID-Konstanten zurückgeben**

| Funktion | Beschreibung |
|---|---|
| `uuid_nil() → uuid` | Gibt eine „Nil“-UUID-Konstante zurück, die nicht als echte UUID vorkommt. |
| `uuid_ns_dns() → uuid` | Gibt eine Konstante zurück, die den DNS-Namespace für UUIDs bezeichnet. |
| `uuid_ns_url() → uuid` | Gibt eine Konstante zurück, die den URL-Namespace für UUIDs bezeichnet. |
| `uuid_ns_oid() → uuid` | Gibt eine Konstante zurück, die den ISO-Object-Identifier-Namespace (OID) für UUIDs bezeichnet. Dies bezieht sich auf ASN.1-OIDs, die nichts mit den in PostgreSQL verwendeten OIDs zu tun haben. |
| `uuid_ns_x500() → uuid` | Gibt eine Konstante zurück, die den X.500-Distinguished-Name-Namespace (DN) für UUIDs bezeichnet. |

#### F.49.2. `uuid-ossp` bauen

Historisch hing dieses Modul von der OSSP-UUID-Bibliothek ab, woraus sein Name stammt. Die OSSP-UUID-Bibliothek ist weiterhin unter <http://www.ossp.org/pkg/lib/uuid/> zu finden, wird aber nicht gut gepflegt und lässt sich zunehmend schwer auf neuere Plattformen portieren. `uuid-ossp` kann auf manchen Plattformen inzwischen ohne die OSSP-Bibliothek gebaut werden. Unter FreeBSD und einigen anderen BSD-abgeleiteten Plattformen sind geeignete UUID-Erzeugungsfunktionen in der Kernbibliothek `libc` enthalten. Unter Linux, macOS und einigen anderen Plattformen werden geeignete Funktionen von der Bibliothek `libuuid` bereitgestellt, die ursprünglich aus dem Projekt `e2fsprogs` stammt und auf modernem Linux als Teil von `util-linux-ng` gilt.

Beim Aufruf von `configure` geben Sie `--with-uuid=bsd` an, um BSD-Funktionen zu verwenden, `--with-uuid=e2fs`, um `libuuid` aus `e2fsprogs` zu verwenden, oder `--with-uuid=ossp`, um die OSSP-UUID-Bibliothek zu verwenden. Auf einem bestimmten Rechner können mehrere dieser Bibliotheken verfügbar sein; `configure` wählt daher nicht automatisch eine aus.

#### F.49.3. Autor

Peter Eisentraut <peter_e@gmx.net>


### F.50. xml2 — XPath-Abfragen und XSLT-Funktionalität

Das Modul `xml2` stellt XPath-Abfragen und XSLT-Funktionalität bereit.

#### F.50.1. Hinweis zur Veralterung

Seit PostgreSQL 8.3 gibt es im Kernserver XML-bezogene Funktionalität auf Basis des SQL/XML-Standards. Diese Funktionalität deckt XML-Syntaxprüfung und XPath-Abfragen ab, also das, was dieses Modul tut, und mehr. Die API ist jedoch nicht kompatibel. Es ist geplant, dieses Modul in einer zukünftigen PostgreSQL-Version zugunsten der neueren Standard-API zu entfernen. Daher sollten Sie versuchen, Ihre Anwendungen umzustellen. Wenn Sie feststellen, dass Funktionalität dieses Moduls in der neuen API nicht angemessen verfügbar ist, schildern Sie das Problem bitte an <pgsql-hackers@lists.postgresql.org>, damit die Lücke geschlossen werden kann.

#### F.50.2. Beschreibung der Funktionen

Tabelle F.37 zeigt die von diesem Modul bereitgestellten Funktionen. Sie bieten einfache XML-Parsing- und XPath-Abfragen.

**Tabelle F.37. Funktionen von `xml2`**

| Funktion | Beschreibung |
|---|---|
| `xml_valid(document text) → boolean` | Parst das angegebene Dokument und gibt `true` zurück, wenn es wohlgeformtes XML ist. Dies ist ein Alias für die Standardfunktion `xml_is_well_formed()`. Der Name `xml_valid()` ist technisch ungenau, da Gültigkeit und Wohlgeformtheit in XML unterschiedliche Bedeutungen haben. |
| `xpath_string(document text, query text) → text` | Wertet die XPath-Abfrage auf dem Dokument aus und wandelt das Ergebnis in `text` um. |
| `xpath_number(document text, query text) → real` | Wertet die XPath-Abfrage auf dem Dokument aus und wandelt das Ergebnis in `real` um. |
| `xpath_bool(document text, query text) → boolean` | Wertet die XPath-Abfrage auf dem Dokument aus und wandelt das Ergebnis in `boolean` um. |
| `xpath_nodeset(document text, query text, toptag text, itemtag text) → text` | Wertet die Abfrage aus und verpackt das Ergebnis in XML-Tags. Bei mehrwertigem Ergebnis werden mehrere `itemtag`-Elemente unter `toptag` erzeugt. Ist `toptag` oder `itemtag` leer, wird das jeweilige Tag ausgelassen. |
| `xpath_nodeset(document text, query text, itemtag text) → text` | Wie `xpath_nodeset(document, query, toptag, itemtag)`, lässt aber `toptag` aus. |
| `xpath_nodeset(document text, query text) → text` | Wie die obige Funktion, lässt aber beide Tags aus. |
| `xpath_list(document text, query text, separator text) → text` | Wertet die Abfrage aus und gibt mehrere Werte getrennt durch `separator` zurück, zum Beispiel `Value 1,Value 2,Value 3` bei `,` als Separator. |
| `xpath_list(document text, query text) → text` | Wrapper für die obige Funktion mit `,` als Separator. |

Beispiel für die Ausgabe von `xpath_nodeset`:

```xml
<toptag>
  <itemtag>Value 1 which could be an XML fragment</itemtag>
  <itemtag>Value 2....</itemtag>
</toptag>
```

#### F.50.3. xpath_table

```text
xpath_table(text key, text document, text relation, text xpaths,
            text criteria) returns setof record
```

`xpath_table` ist eine Tabellenfunktion, die eine Menge von XPath-Abfragen auf jedes Dokument einer Dokumentmenge anwendet und die Ergebnisse als Tabelle zurückgibt. Das Primärschlüsselfeld aus der ursprünglichen Dokumenttabelle wird als erste Spalte des Ergebnisses zurückgegeben, damit das Ergebnis einfach in Joins verwendet werden kann. Die Parameter sind in Tabelle F.38 beschrieben.

**Tabelle F.38. Parameter von `xpath_table`**

| Parameter | Beschreibung |
|---|---|
| `key` | Name des Schlüsselfelds. Dieses Feld wird als erste Spalte der Ausgabetabelle verwendet und identifiziert den Datensatz, aus dem die jeweilige Ausgabezeile stammt. |
| `document` | Name des Felds, das das XML-Dokument enthält. |
| `relation` | Name der Tabelle oder Sicht, die die Dokumente enthält. |
| `xpaths` | Eine oder mehrere XPath-Ausdrücke, getrennt durch `|`. |
| `criteria` | Inhalt der `WHERE`-Klausel. Dieser Parameter kann nicht weggelassen werden; verwenden Sie `true` oder `1=1`, wenn alle Zeilen der Relation verarbeitet werden sollen. |

Diese Parameter, mit Ausnahme der XPath-Zeichenketten, werden einfach in eine normale SQL-`SELECT`-Anweisung eingesetzt:

```sql
SELECT <key>, <document> FROM <relation> WHERE <criteria>
```

Sie haben dabei eine gewisse Flexibilität; die Werte müssen nur an diesen Positionen gültig sein. Das Ergebnis dieses `SELECT` muss genau zwei Spalten zurückgeben. Dieser einfache Ansatz bedeutet auch, dass benutzerbereitgestellte Werte validiert werden müssen, um SQL-Injection-Angriffe zu vermeiden.

Die Funktion muss in einem `FROM`-Ausdruck mit einer `AS`-Klausel verwendet werden, die die Ausgabespalten angibt:

```sql
SELECT * FROM
xpath_table('article_id',
            'article_xml',
            'articles',
            '/article/author|/article/pages|/article/title',
            'date_entered > ''2003-01-01'' ')
AS t(article_id integer, author text, page_count integer, title text);
```

Die `AS`-Klausel definiert Namen und Typen der Ausgabespalten. Die erste ist das Schlüsselfeld, die übrigen entsprechen den XPath-Abfragen. Gibt es mehr XPath-Abfragen als Ergebnisspalten, werden die zusätzlichen Abfragen ignoriert. Gibt es mehr Ergebnisspalten als XPath-Abfragen, werden die zusätzlichen Spalten `NULL`.

Wenn Sie etwa `page_count` als `integer` definieren, wandelt die Funktion intern die Zeichenkettendarstellung des XPath-Ergebnisses mit den PostgreSQL-Eingabefunktionen in `integer` um. Wenn dies nicht möglich ist, entsteht ein Fehler. Bei problematischen Daten kann es daher sinnvoll sein, `text` als Spaltentyp zu verwenden.

Die aufrufende `SELECT`-Anweisung muss nicht nur `SELECT *` sein. Sie kann Ausgabespalten namentlich referenzieren oder mit anderen Tabellen joinen:

```sql
SELECT t.title, p.fullname, p.email
FROM xpath_table('article_id', 'article_xml', 'articles',
                 '/article/title|/article/author/@id',
                 'xpath_string(article_xml,''/article/@date'') > ''2003-03-20'' ')
     AS t(article_id integer, title text, author_id integer),
     tblPeopleInfo AS p
WHERE t.author_id = p.person_id;
```

Natürlich können Sie dies zur bequemeren Verwendung in eine Sicht einbetten.

##### F.50.3.1. Mehrwertige Ergebnisse

`xpath_table` nimmt an, dass die Ergebnisse jeder XPath-Abfrage mehrwertig sein können. Die Zahl der von der Funktion zurückgegebenen Zeilen muss daher nicht der Zahl der Eingabedokumente entsprechen. Die erste zurückgegebene Zeile enthält das erste Ergebnis jeder Abfrage, die zweite das zweite Ergebnis jeder Abfrage usw. Hat eine Abfrage weniger Werte als die anderen, werden stattdessen `NULL`-Werte zurückgegeben.

Wenn ein Benutzer weiß, dass eine bestimmte XPath-Abfrage nur ein einzelnes Ergebnis zurückgibt, etwa einen eindeutigen Dokumentbezeichner, erscheint dieses einwertige Ergebnis neben einer mehrwertigen XPath-Abfrage nur in der ersten Ergebniszeile. Die Lösung besteht darin, das Schlüsselfeld als Join-Bestandteil gegen eine einfachere XPath-Abfrage zu verwenden.

Beispiel:

```sql
CREATE TABLE test (
    id int PRIMARY KEY,
    xml text
);

INSERT INTO test VALUES (1, '<doc num="C1">
<line num="L1"><a>1</a><b>2</b><c>3</c></line>
<line num="L2"><a>11</a><b>22</b><c>33</c></line>
</doc>');

SELECT * FROM
xpath_table('id','xml','test',
            '/doc/@num|/doc/line/@num|/doc/line/a|/doc/line/b|/doc/line/c',
            'true')
AS t(id int, doc_num varchar(10), line_num varchar(10), val1 int, val2 int, val3 int)
WHERE id = 1
ORDER BY doc_num, line_num;
```

```text
id | doc_num | line_num | val1 | val2 | val3
---+---------+----------+------+------+------
1  | C1      | L1       |    1 |    2 |    3
1  |         | L2       |   11 |   22 |   33
```

Um `doc_num` in jeder Zeile zu erhalten, verwenden Sie zwei Aufrufe von `xpath_table` und joinen deren Ergebnisse:

```sql
SELECT t.*, i.doc_num
FROM xpath_table('id', 'xml', 'test',
                 '/doc/line/@num|/doc/line/a|/doc/line/b|/doc/line/c',
                 'true')
     AS t(id int, line_num varchar(10), val1 int, val2 int, val3 int),
     xpath_table('id', 'xml', 'test', '/doc/@num', 'true')
     AS i(id int, doc_num varchar(10))
WHERE i.id = t.id AND i.id = 1
ORDER BY doc_num, line_num;
```

```text
id | line_num | val1 | val2 | val3 | doc_num
---+----------+------+------+------+---------
1  | L1       |    1 |    2 |    3 | C1
1  | L2       |   11 |   22 |   33 | C1
(2 rows)
```

#### F.50.4. XSLT-Funktionen

Die folgenden Funktionen sind verfügbar, wenn `libxslt` installiert ist.

##### F.50.4.1. xslt_process

```text
xslt_process(text document, text stylesheet, text paramlist) returns text
```

Diese Funktion wendet das XSL-Stylesheet auf das Dokument an und gibt das transformierte Ergebnis zurück. `paramlist` ist eine Liste von Parameterzuweisungen für die Transformation im Format `a=1,b=2`. Die Parameterverarbeitung ist sehr einfach: Parameterwerte dürfen keine Kommas enthalten.

Es gibt auch eine zweiparametrige Variante von `xslt_process`, die keine Parameter an die Transformation übergibt.

#### F.50.5. Autor

John Gray <jgray@azuli.co.uk>

Die Entwicklung dieses Moduls wurde von Torchbox Ltd. (<https://www.torchbox.com>) gesponsert. Es hat dieselbe BSD-Lizenz wie PostgreSQL.
