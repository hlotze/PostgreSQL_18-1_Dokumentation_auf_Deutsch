# 53. Systemsichten

Zusätzlich zu den Systemkatalogen stellt PostgreSQL eine Reihe eingebauter Sichten bereit. Einige Systemsichten bieten bequemen Zugriff auf häufig verwendete Abfragen der Systemkataloge. Andere Sichten ermöglichen Zugriff auf den internen Zustand des Servers.

Das Information Schema ([Kapitel 35](35_Das_Information_Schema.md)) bietet einen alternativen Satz von Sichten, deren Funktionalität sich mit den Systemsichten überschneidet. Da das Information Schema dem SQL-Standard entspricht, während die hier beschriebenen Sichten PostgreSQL-spezifisch sind, ist es normalerweise besser, das Information Schema zu verwenden, wenn es alle benötigten Informationen bereitstellt.

Tabelle 53.1 listet die hier beschriebenen Systemsichten auf. Eine ausführlichere Dokumentation jeder Sicht folgt weiter unten. Es gibt einige zusätzliche Sichten, die Zugriff auf gesammelte Statistiken bieten; sie werden in Tabelle 27.2 beschrieben.

## 53.1. Überblick

Tabelle 53.1 listet die Systemsichten auf. Eine ausführlichere Dokumentation jedes Katalogs folgt weiter unten. Soweit nicht anders angegeben, sind alle hier beschriebenen Sichten schreibgeschützt.

**Tabelle 53.1. Systemsichten**

| Sichtname | Zweck |
| --- | --- |
| `pg_aios` | verwendete asynchrone I/O-Handles |
| `pg_available_extensions` | verfügbare Erweiterungen |
| `pg_available_extension_versions` | verfügbare Versionen von Erweiterungen |
| `pg_backend_memory_contexts` | Speicher-Kontexte des Backends |
| `pg_config` | Konfigurationsparameter zur Kompilierzeit |
| `pg_cursors` | offene Cursor |
| `pg_file_settings` | Zusammenfassung der Inhalte von Konfigurationsdateien |
| `pg_group` | Gruppen von Datenbankbenutzern |
| `pg_hba_file_rules` | Zusammenfassung der Inhalte der Konfigurationsdatei für Client-Authentifizierung |
| `pg_ident_file_mappings` | Zusammenfassung der Inhalte der Konfigurationsdatei für Zuordnungen von Client-Benutzernamen |
| `pg_indexes` | Indizes |
| `pg_locks` | aktuell gehaltene oder erwartete Sperren |
| `pg_matviews` | materialisierte Sichten |
| `pg_policies` | Policies |
| `pg_prepared_statements` | vorbereitete Anweisungen |
| `pg_prepared_xacts` | vorbereitete Transaktionen |
| `pg_publication_tables` | Publikationen und Informationen über ihre zugehörigen Tabellen |
| `pg_replication_origin_status` | Informationen über Replikationsursprünge, einschließlich des Replikationsfortschritts |
| `pg_replication_slots` | Informationen zu Replikations-Slots |
| `pg_roles` | Datenbankrollen |
| `pg_rules` | Regeln |
| `pg_seclabels` | Sicherheitslabels |
| `pg_sequences` | Sequenzen |
| `pg_settings` | Parametereinstellungen |
| `pg_shadow` | Datenbankbenutzer |
| `pg_shmem_allocations` | Shared-Memory-Allokationen |
| `pg_shmem_allocations_numa` | NUMA-Knotenzuordnungen für Shared-Memory-Allokationen |
| `pg_stats` | Planner-Statistiken |
| `pg_stats_ext` | erweiterte Planner-Statistiken |
| `pg_stats_ext_exprs` | erweiterte Planner-Statistiken für Ausdrücke |
| `pg_tables` | Tabellen |
| `pg_timezone_abbrevs` | Zeitzonenabkürzungen |
| `pg_timezone_names` | Zeitzonennamen |
| `pg_user` | Datenbankbenutzer |
| `pg_user_mappings` | Benutzerzuordnungen |
| `pg_views` | Sichten |
| `pg_wait_events` | Warteereignisse |

## 53.2. pg_aios

Die Sicht `pg_aios` listet alle aktuell verwendeten asynchronen I/O-Handles auf. Ein I/O-Handle wird verwendet, um auf eine I/O-Operation zu verweisen, die vorbereitet, ausgeführt oder gerade abgeschlossen wird. `pg_aios` enthält eine Zeile für jedes I/O-Handle.

Diese Sicht ist hauptsächlich für Entwickler von PostgreSQL nützlich, kann aber auch beim Tuning von PostgreSQL hilfreich sein.

**Tabelle 53.2. Spalten von `pg_aios`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pid` | `int4` | Prozess-ID des Serverprozesses, der diese I/O-Operation ausgibt. |
| `io_id` | `int4` | Bezeichner des I/O-Handles. Handles werden wiederverwendet, sobald die I/O-Operation abgeschlossen ist oder wenn das Handle freigegeben wird, bevor I/O gestartet wurde. Bei Wiederverwendung wird `pg_aios.io_generation` erhöht. |
| `io_generation` | `int8` | Generation des I/O-Handles. |
| `state` | `text` | Zustand des I/O-Handles: `HANDED_OUT`, von Code referenziert, aber noch nicht verwendet; `DEFINED`, die für die Ausführung nötigen Informationen sind bekannt; `STAGED`, bereit zur Ausführung; `SUBMITTED`, zur Ausführung eingereicht; `COMPLETED_IO`, abgeschlossen, aber das Ergebnis wurde noch nicht verarbeitet; `COMPLETED_SHARED`, gemeinsame Abschlussverarbeitung abgeschlossen; `COMPLETED_LOCAL`, Backend-lokale Abschlussverarbeitung abgeschlossen. |
| `operation` | `text` | Operation, die mit dem I/O-Handle ausgeführt wird: `invalid`, noch nicht bekannt; `readv`, ein vektorisierter Lesevorgang; `writev`, ein vektorisierter Schreibvorgang. |
| `off` | `int8` | Offset der I/O-Operation. |
| `length` | `int8` | Länge der I/O-Operation. |
| `target` | `text` | Art des Objekts, auf das die I/O-Operation zielt: `smgr`, I/O auf Relationen. |
| `handle_data_len` | `int2` | Länge der Daten, die mit der I/O-Operation verbunden sind. Bei I/O von oder nach `shared_buffers` und `temp_buffers` gibt dies die Anzahl der Puffer an, auf denen die I/O-Operation arbeitet. |
| `raw_result` | `int4` | Low-Level-Ergebnis der I/O-Operation oder `NULL`, wenn die Operation noch nicht abgeschlossen ist. |
| `result` | `text` | High-Level-Ergebnis der I/O-Operation: `UNKNOWN` bedeutet, dass das Ergebnis der Operation noch nicht bekannt ist; `OK` bedeutet, dass die I/O-Operation erfolgreich abgeschlossen wurde; `PARTIAL` bedeutet, dass die I/O-Operation ohne Fehler abgeschlossen wurde, aber nicht alle Daten verarbeitet hat. Üblicherweise müssen Aufrufer erneut versuchen, den verbleibenden Teil der Arbeit in einer separaten I/O-Operation auszuführen. `WARNING` bedeutet, dass die I/O-Operation ohne Fehler abgeschlossen wurde, die Ausführung der I/O-Operation aber eine Warnung ausgelöst hat, etwa beim Auftreten eines beschädigten Puffers mit aktiviertem `zero_damaged_pages`. `ERROR` bedeutet, dass die I/O-Operation mit einem Fehler fehlgeschlagen ist. |
| `target_desc` | `text` | Beschreibung dessen, worauf die I/O-Operation zielt. |
| `f_sync` | `bool` | Kennzeichen, ob die I/O-Operation synchron ausgeführt wird. |
| `f_localmem` | `bool` | Kennzeichen, ob die I/O-Operation prozesslokalen Speicher referenziert. |
| `f_buffered` | `bool` | Kennzeichen, ob es sich um gepufferte I/O handelt. |

Die Sicht `pg_aios` ist schreibgeschützt.

Standardmäßig kann die Sicht `pg_aios` nur von Superusern oder Rollen mit den Rechten der Rolle `pg_read_all_stats` gelesen werden.

## 53.3. pg_available_extensions

Die Sicht `pg_available_extensions` listet die Erweiterungen auf, die zur Installation verfügbar sind. Siehe auch den Katalog `pg_extension`, der die aktuell installierten Erweiterungen zeigt.

**Tabelle 53.3. Spalten von `pg_available_extensions`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `name` | Name der Erweiterung |
| `default_version` | `text` | Name der Standardversion oder `NULL`, wenn keine angegeben ist |
| `installed_version` | `text` | Aktuell installierte Version der Erweiterung oder `NULL`, wenn sie nicht installiert ist |
| `comment` | `text` | Kommentarzeichenkette aus der Kontrolldatei der Erweiterung |

Die Sicht `pg_available_extensions` ist schreibgeschützt.

## 53.4. pg_available_extension_versions

Die Sicht `pg_available_extension_versions` listet die konkreten Erweiterungsversionen auf, die zur Installation verfügbar sind. Siehe auch den Katalog `pg_extension`, der die aktuell installierten Erweiterungen zeigt.

**Tabelle 53.4. Spalten von `pg_available_extension_versions`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `name` | Name der Erweiterung |
| `version` | `text` | Versionsname |
| `installed` | `bool` | Wahr, wenn diese Version dieser Erweiterung derzeit installiert ist |
| `superuser` | `bool` | Wahr, wenn nur Superuser diese Erweiterung installieren dürfen; siehe aber `trusted` |
| `trusted` | `bool` | Wahr, wenn die Erweiterung von Nicht-Superusern mit passenden Rechten installiert werden kann |
| `relocatable` | `bool` | Wahr, wenn die Erweiterung in ein anderes Schema verschoben werden kann |
| `schema` | `name` | Name des Schemas, in das die Erweiterung installiert werden muss, oder `NULL`, wenn sie teilweise oder vollständig verschiebbar ist |
| `requires` | `name[]` | Namen vorausgesetzter Erweiterungen oder `NULL`, wenn es keine gibt |
| `comment` | `text` | Kommentarzeichenkette aus der Kontrolldatei der Erweiterung |

Die Sicht `pg_available_extension_versions` ist schreibgeschützt.

## 53.5. pg_backend_memory_contexts

Die Sicht `pg_backend_memory_contexts` zeigt alle Speicher-Kontexte des Serverprozesses an, der mit der aktuellen Sitzung verbunden ist.

`pg_backend_memory_contexts` enthält eine Zeile für jeden Speicher-Kontext.

**Tabelle 53.5. Spalten von `pg_backend_memory_contexts`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Name des Speicher-Kontexts |
| `ident` | `text` | Identifikationsinformationen des Speicher-Kontexts. Dieses Feld wird bei 1024 Byte abgeschnitten. |
| `type` | `text` | Typ des Speicher-Kontexts |
| `level` | `int4` | 1-basierte Ebene des Kontexts in der Speicher-Kontext-Hierarchie. Die Ebene eines Kontexts zeigt auch die Position dieses Kontexts in der Spalte `path`. |
| `path` | `int4[]` | Array temporärer numerischer Bezeichner zur Beschreibung der Speicher-Kontext-Hierarchie. Das erste Element steht für `TopMemoryContext`, nachfolgende Elemente enthalten Zwischen-Eltern, und das letzte Element enthält den Bezeichner des aktuellen Kontexts. |
| `total_bytes` | `int8` | Insgesamt für diesen Speicher-Kontext allozierte Bytes |
| `total_nblocks` | `int8` | Gesamtzahl der für diesen Speicher-Kontext allozierten Blöcke |
| `free_bytes` | `int8` | Freier Speicherplatz in Byte |
| `free_chunks` | `int8` | Gesamtzahl freier Chunks |
| `used_bytes` | `int8` | Verwendeter Speicherplatz in Byte |

Standardmäßig kann die Sicht `pg_backend_memory_contexts` nur von Superusern oder Rollen mit den Rechten der Rolle `pg_read_all_stats` gelesen werden.

Da Speicher-Kontexte während der Ausführung einer Abfrage erzeugt und zerstört werden, können die in der Spalte `path` gespeicherten Bezeichner zwischen mehreren Aufrufen der Sicht in derselben Abfrage instabil sein. Das folgende Beispiel zeigt eine wirksame Verwendung dieser Spalte und berechnet die Gesamtzahl der Bytes, die von `CacheMemoryContext` und allen seinen Kindern verwendet werden:

```sql
WITH memory_contexts AS (
    SELECT * FROM pg_backend_memory_contexts
)
SELECT sum(c1.total_bytes)
FROM memory_contexts c1, memory_contexts c2
WHERE c2.name = 'CacheMemoryContext'
AND c1.path[c2.level] = c2.path[c2.level];
```

Der Common Table Expression wird verwendet, um sicherzustellen, dass die Kontext-IDs in der Spalte `path` zwischen beiden Auswertungen der Sicht übereinstimmen.

## 53.6. pg_config

Die Sicht `pg_config` beschreibt die Konfigurationsparameter zur Kompilierzeit der aktuell installierten Version von PostgreSQL. Sie ist beispielsweise für Softwarepakete gedacht, die an PostgreSQL anbinden und das Auffinden der erforderlichen Header-Dateien und Bibliotheken erleichtern wollen. Sie stellt dieselben grundlegenden Informationen bereit wie die PostgreSQL-Client-Anwendung `pg_config`.

Standardmäßig kann die Sicht `pg_config` nur von Superusern gelesen werden.

**Tabelle 53.6. Spalten von `pg_config`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Parametername |
| `setting` | `text` | Parameterwert |

## 53.7. pg_cursors

Die Sicht `pg_cursors` listet die Cursor auf, die derzeit verfügbar sind. Cursor können auf verschiedene Arten definiert werden:

- über die SQL-Anweisung `DECLARE`
- über die `Bind`-Nachricht im Frontend/Backend-Protokoll, wie in Abschnitt 54.2.3 beschrieben
- über das Server Programming Interface (SPI), wie in [Abschnitt 45.1](45_Server_Programming_Interface.md#451-schnittstellenfunktionen) beschrieben

Die Sicht `pg_cursors` zeigt Cursor an, die auf irgendeine dieser Arten erzeugt wurden. Cursor existieren nur für die Dauer der Transaktion, die sie definiert, sofern sie nicht mit `WITH HOLD` deklariert wurden. Daher sind nicht haltbare Cursor nur bis zum Ende ihrer erzeugenden Transaktion in der Sicht vorhanden.

> **Hinweis**
>
> Cursor werden intern verwendet, um einige Komponenten von PostgreSQL zu implementieren, etwa prozedurale Sprachen. Daher kann die Sicht `pg_cursors` Cursor enthalten, die vom Benutzer nicht ausdrücklich erzeugt wurden.

**Tabelle 53.7. Spalten von `pg_cursors`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Name des Cursors |
| `statement` | `text` | Wörtliche Abfragezeichenkette, die zum Deklarieren dieses Cursors übermittelt wurde |
| `is_holdable` | `bool` | Wahr, wenn der Cursor haltbar ist, also nach dem Commit der Transaktion, die ihn deklariert hat, weiter zugreifbar ist; andernfalls falsch |
| `is_binary` | `bool` | Wahr, wenn der Cursor als `BINARY` deklariert wurde; andernfalls falsch |
| `is_scrollable` | `bool` | Wahr, wenn der Cursor scrollfähig ist, also Zeilen in nicht sequenzieller Weise abgerufen werden können; andernfalls falsch |
| `creation_time` | `timestamptz` | Zeitpunkt, zu dem der Cursor deklariert wurde |

Die Sicht `pg_cursors` ist schreibgeschützt.

## 53.8. pg_file_settings

Die Sicht `pg_file_settings` bietet eine Zusammenfassung der Inhalte der Konfigurationsdatei oder -dateien des Servers. Für jeden Eintrag der Form `name = value`, der in den Dateien vorkommt, erscheint eine Zeile in dieser Sicht, versehen mit Anmerkungen dazu, ob der Wert erfolgreich angewendet werden konnte. Zusätzliche Zeilen können für Probleme erscheinen, die nicht mit einem `name = value`-Eintrag verknüpft sind, etwa Syntaxfehler in den Dateien.

Diese Sicht ist hilfreich, um zu prüfen, ob geplante Änderungen in den Konfigurationsdateien funktionieren werden, oder um einen früheren Fehler zu diagnostizieren. Beachten Sie, dass diese Sicht über die aktuellen Inhalte der Dateien berichtet, nicht darüber, was zuletzt vom Server angewendet wurde. Dafür ist die Sicht `pg_settings` normalerweise ausreichend.

Standardmäßig kann die Sicht `pg_file_settings` nur von Superusern gelesen werden.

**Tabelle 53.8. Spalten von `pg_file_settings`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `sourcefile` | `text` | Vollständiger Pfadname der Konfigurationsdatei |
| `sourceline` | `int4` | Zeilennummer innerhalb der Konfigurationsdatei, in der der Eintrag erscheint |
| `seqno` | `int4` | Reihenfolge, in der die Einträge verarbeitet werden (`1..n`) |
| `name` | `text` | Name des Konfigurationsparameters |
| `setting` | `text` | Wert, der dem Parameter zugewiesen werden soll |
| `applied` | `bool` | Wahr, wenn der Wert erfolgreich angewendet werden kann |
| `error` | `text` | Wenn nicht `NULL`, eine Fehlermeldung, die angibt, warum dieser Eintrag nicht angewendet werden konnte |

Wenn die Konfigurationsdatei Syntaxfehler oder ungültige Parameternamen enthält, versucht der Server nicht, irgendwelche Einstellungen daraus anzuwenden; daher werden alle `applied`-Felder als falsch angezeigt. In einem solchen Fall gibt es eine oder mehrere Zeilen mit nicht-`NULL`-Werten in den `error`-Feldern, die das Problem oder die Probleme angeben. Andernfalls werden einzelne Einstellungen angewendet, sofern das möglich ist. Wenn eine einzelne Einstellung nicht angewendet werden kann, etwa wegen eines ungültigen Werts oder weil die Einstellung nach dem Serverstart nicht geändert werden kann, enthält sie eine passende Meldung im Feld `error`. Eine weitere Möglichkeit für `applied = false` ist, dass der Eintrag durch einen späteren Eintrag für denselben Parameternamen überschrieben wird; dieser Fall gilt nicht als Fehler, daher erscheint im Feld `error` nichts.

Weitere Informationen zu den verschiedenen Möglichkeiten, Laufzeitparameter zu ändern, finden Sie in [Abschnitt 19.1](19_Serverkonfiguration.md#191-parameter-setzen).

## 53.9. pg_group

Die Sicht `pg_group` existiert aus Gründen der Rückwärtskompatibilität: Sie emuliert einen Katalog, der in PostgreSQL vor Version 8.1 existierte. Sie zeigt die Namen und Mitglieder aller Rollen, die als nicht `rolcanlogin` markiert sind; das ist eine Annäherung an die Menge der Rollen, die als Gruppen verwendet werden.

**Tabelle 53.9. Spalten von `pg_group`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `groname` | `name` (referenziert `pg_authid.rolname`) | Name der Gruppe |
| `grosysid` | `oid` (referenziert `pg_authid.oid`) | ID dieser Gruppe |
| `grolist` | `oid[]` (referenziert `pg_authid.oid`) | Ein Array, das die IDs der Rollen in dieser Gruppe enthält |

## 53.10. pg_hba_file_rules

Die Sicht `pg_hba_file_rules` bietet eine Zusammenfassung der Inhalte der Client-Authentifizierungs-Konfigurationsdatei `pg_hba.conf`. Für jede nicht leere Zeile ohne Kommentar in der Datei erscheint eine Zeile in dieser Sicht, versehen mit Anmerkungen dazu, ob die Regel erfolgreich angewendet werden konnte.

Diese Sicht kann hilfreich sein, um zu prüfen, ob geplante Änderungen an der Authentifizierungs-Konfigurationsdatei funktionieren werden, oder um einen früheren Fehler zu diagnostizieren. Beachten Sie, dass diese Sicht über die aktuellen Inhalte der Datei berichtet, nicht darüber, was zuletzt vom Server geladen wurde.

Standardmäßig kann die Sicht `pg_hba_file_rules` nur von Superusern gelesen werden.

**Tabelle 53.10. Spalten von `pg_hba_file_rules`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `rule_number` | `int4` | Nummer dieser Regel, wenn gültig, andernfalls `NULL`. Dies gibt die Reihenfolge an, in der jede Regel betrachtet wird, bis während der Authentifizierung eine Übereinstimmung gefunden wird. |
| `file_name` | `text` | Name der Datei, die diese Regel enthält |
| `line_number` | `int4` | Zeilennummer dieser Regel in `file_name` |
| `type` | `text` | Typ der Verbindung |
| `database` | `text[]` | Liste der Datenbanknamen, für die diese Regel gilt |
| `user_name` | `text[]` | Liste der Benutzer- und Gruppennamen, für die diese Regel gilt |
| `address` | `text` | Hostname oder IP-Adresse oder eines von `all`, `samehost` oder `samenet`, oder `NULL` für lokale Verbindungen |
| `netmask` | `text` | IP-Adressmaske oder `NULL`, wenn nicht anwendbar |
| `auth_method` | `text` | Authentifizierungsmethode |
| `options` | `text[]` | Optionen, die für die Authentifizierungsmethode angegeben wurden, falls vorhanden |
| `error` | `text` | Wenn nicht `NULL`, eine Fehlermeldung, die angibt, warum diese Zeile nicht verarbeitet werden konnte |

Üblicherweise enthält eine Zeile, die einen fehlerhaften Eintrag widerspiegelt, nur Werte für die Felder `line_number` und `error`.

Weitere Informationen zur Konfiguration der Client-Authentifizierung finden Sie in [Kapitel 20](20_Client_Authentifizierung.md).

## 53.11. pg_ident_file_mappings

Die Sicht `pg_ident_file_mappings` bietet eine Zusammenfassung der Inhalte der Konfigurationsdatei `pg_ident.conf` für Zuordnungen von Client-Benutzernamen. Für jede nicht leere Zeile ohne Kommentar in der Datei erscheint eine Zeile in dieser Sicht, versehen mit Anmerkungen dazu, ob die Zuordnung erfolgreich angewendet werden konnte.

Diese Sicht kann hilfreich sein, um zu prüfen, ob geplante Änderungen an der Authentifizierungs-Konfigurationsdatei funktionieren werden, oder um einen früheren Fehler zu diagnostizieren. Beachten Sie, dass diese Sicht über die aktuellen Inhalte der Datei berichtet, nicht darüber, was zuletzt vom Server geladen wurde.

Standardmäßig kann die Sicht `pg_ident_file_mappings` nur von Superusern gelesen werden.

**Tabelle 53.11. Spalten von `pg_ident_file_mappings`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `map_number` | `int4` | Nummer dieser Zuordnung in Prioritätsreihenfolge, wenn gültig, andernfalls `NULL` |
| `file_name` | `text` | Name der Datei, die diese Zuordnung enthält |
| `line_number` | `int4` | Zeilennummer dieser Zuordnung in `file_name` |
| `map_name` | `text` | Name der Zuordnung |
| `sys_name` | `text` | Erkannter Benutzername des Clients |
| `pg_username` | `text` | Angeforderter PostgreSQL-Benutzername |
| `error` | `text` | Wenn nicht `NULL`, eine Fehlermeldung, die angibt, warum diese Zeile nicht verarbeitet werden konnte |

Üblicherweise enthält eine Zeile, die einen fehlerhaften Eintrag widerspiegelt, nur Werte für die Felder `line_number` und `error`.

Weitere Informationen zur Konfiguration der Client-Authentifizierung finden Sie in [Kapitel 20](20_Client_Authentifizierung.md).

## 53.12. pg_indexes

Die Sicht `pg_indexes` bietet Zugriff auf nützliche Informationen über jeden Index in der Datenbank.

**Tabelle 53.12. Spalten von `pg_indexes`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das Tabelle und Index enthält |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle, zu der der Index gehört |
| `indexname` | `name` (referenziert `pg_class.relname`) | Name des Index |
| `tablespace` | `name` (referenziert `pg_tablespace.spcname`) | Name des Tablespace, der den Index enthält, oder `NULL`, wenn der Standard der Datenbank verwendet wird |
| `indexdef` | `text` | Indexdefinition, also ein rekonstruiertes `CREATE INDEX`-Kommando |

## 53.13. pg_locks

Die Sicht `pg_locks` bietet Zugriff auf Informationen über Sperren, die von aktiven Prozessen innerhalb des Datenbankservers gehalten werden. Weitere Erläuterungen zu Sperren finden Sie in [Kapitel 13](13_Nebenläufigkeitskontrolle.md).

`pg_locks` enthält eine Zeile pro aktivem sperrbarem Objekt, angefordertem Sperrmodus und relevantem Prozess. Daher kann dasselbe sperrbare Objekt mehrfach erscheinen, wenn mehrere Prozesse Sperren darauf halten oder auf Sperren dafür warten. Ein Objekt, auf dem derzeit keine Sperren liegen, erscheint dagegen überhaupt nicht.

Es gibt mehrere unterschiedliche Arten sperrbarer Objekte: ganze Relationen, beispielsweise Tabellen, einzelne Seiten von Relationen, einzelne Tupel von Relationen, Transaktions-IDs, sowohl virtuelle als auch permanente IDs, und allgemeine Datenbankobjekte, die über Klassen-OID und Objekt-OID identifiziert werden, auf dieselbe Weise wie in `pg_description` oder `pg_depend`. Außerdem wird das Recht, eine Relation zu erweitern, als eigenes sperrbares Objekt dargestellt, ebenso das Recht, `pg_database.datfrozenxid` zu aktualisieren. Außerdem können Advisory Locks auf Zahlen genommen werden, denen Benutzer eigene Bedeutungen geben.

**Tabelle 53.13. Spalten von `pg_locks`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `locktype` | `text` | Typ des sperrbaren Objekts: `relation`, `extend`, `frozenid`, `page`, `tuple`, `transactionid`, `virtualxid`, `spectoken`, `object`, `userlock`, `advisory` oder `applytransaction`. Siehe auch Tabelle 27.11. |
| `database` | `oid` (referenziert `pg_database.oid`) | OID der Datenbank, in der das Sperrziel existiert, oder `0`, wenn das Ziel ein gemeinsames Objekt ist, oder `NULL`, wenn das Ziel eine Transaktions-ID ist |
| `relation` | `oid` (referenziert `pg_class.oid`) | OID der Relation, auf die die Sperre zielt, oder `NULL`, wenn das Ziel keine Relation oder kein Teil einer Relation ist |
| `page` | `int4` | Seitennummer innerhalb der Relation, auf die die Sperre zielt, oder `NULL`, wenn das Ziel keine Relationsseite und kein Tupel ist |
| `tuple` | `int2` | Tupelnummer innerhalb der Seite, auf die die Sperre zielt, oder `NULL`, wenn das Ziel kein Tupel ist |
| `virtualxid` | `text` | Virtuelle ID der Transaktion, auf die die Sperre zielt, oder `NULL`, wenn das Ziel keine virtuelle Transaktions-ID ist; siehe [Kapitel 67](67_Transaktionsverarbeitung.md) |
| `transactionid` | `xid` | ID der Transaktion, auf die die Sperre zielt, oder `NULL`, wenn das Ziel keine Transaktions-ID ist; siehe [Kapitel 67](67_Transaktionsverarbeitung.md) |
| `classid` | `oid` (referenziert `pg_class.oid`) | OID des Systemkatalogs, der das Sperrziel enthält, oder `NULL`, wenn das Ziel kein allgemeines Datenbankobjekt ist |
| `objid` | `oid` (referenziert eine beliebige OID-Spalte) | OID des Sperrziels innerhalb seines Systemkatalogs oder `NULL`, wenn das Ziel kein allgemeines Datenbankobjekt ist |
| `objsubid` | `int2` | Spaltennummer, auf die die Sperre zielt; `classid` und `objid` beziehen sich dabei auf die Tabelle selbst. Null, wenn das Ziel ein anderes allgemeines Datenbankobjekt ist, oder `NULL`, wenn das Ziel kein allgemeines Datenbankobjekt ist |
| `virtualtransaction` | `text` | Virtuelle ID der Transaktion, die diese Sperre hält oder auf sie wartet |
| `pid` | `int4` | Prozess-ID des Serverprozesses, der diese Sperre hält oder auf sie wartet, oder `NULL`, wenn die Sperre von einer vorbereiteten Transaktion gehalten wird |
| `mode` | `text` | Name des Sperrmodus, den dieser Prozess hält oder anfordert; siehe Abschnitt 13.3.1 und Abschnitt 13.2.3 |
| `granted` | `bool` | Wahr, wenn die Sperre gehalten wird; falsch, wenn auf die Sperre gewartet wird |
| `fastpath` | `bool` | Wahr, wenn die Sperre über den Fast Path genommen wurde; falsch, wenn sie über die Haupt-Sperrtabelle genommen wurde |
| `waitstart` | `timestamptz` | Zeitpunkt, zu dem der Serverprozess begonnen hat, auf diese Sperre zu warten, oder `NULL`, wenn die Sperre gehalten wird. Beachten Sie, dass dies für sehr kurze Zeit nach Beginn des Wartens `NULL` sein kann, obwohl `granted` falsch ist. |

`granted` ist in einer Zeile wahr, die eine vom angegebenen Prozess gehaltene Sperre darstellt. Falsch bedeutet, dass dieser Prozess derzeit darauf wartet, diese Sperre zu erwerben. Das impliziert, dass mindestens ein anderer Prozess einen kollidierenden Sperrmodus auf demselben sperrbaren Objekt hält oder darauf wartet. Der wartende Prozess schläft, bis die andere Sperre freigegeben wird oder eine Deadlock-Situation erkannt wird. Ein einzelner Prozess kann immer nur auf den Erwerb höchstens einer Sperre warten.

Während der Ausführung einer Transaktion hält ein Serverprozess eine exklusive Sperre auf die virtuelle Transaktions-ID der Transaktion. Wenn der Transaktion eine permanente ID zugewiesen wird, was normalerweise nur geschieht, wenn die Transaktion den Zustand der Datenbank ändert, hält sie bis zu ihrem Ende auch eine exklusive Sperre auf ihre permanente Transaktions-ID. Wenn ein Prozess gezielt darauf warten muss, dass eine andere Transaktion endet, versucht er, eine Share-Sperre auf die ID der anderen Transaktion zu erwerben, je nach Situation auf die virtuelle oder permanente ID. Das gelingt erst, wenn die andere Transaktion endet und ihre Sperren freigibt.

Obwohl Tupel eine sperrbare Objektart sind, werden Informationen über Sperren auf Zeilenebene auf der Platte gespeichert, nicht im Arbeitsspeicher; daher erscheinen Zeilensperren normalerweise nicht in dieser Sicht. Wenn ein Prozess auf eine Sperre auf Zeilenebene wartet, erscheint er in der Sicht üblicherweise so, als warte er auf die permanente Transaktions-ID des aktuellen Besitzers dieser Zeilensperre.

Eine Sperre für spekulative Einfügungen besteht aus einer Transaktions-ID und einem Token für spekulative Einfügungen. Das Token für spekulative Einfügungen wird in der Spalte `objid` angezeigt.

Advisory Locks können auf Schlüssel genommen werden, die entweder aus einem einzelnen `bigint`-Wert oder aus zwei Integer-Werten bestehen. Ein `bigint`-Schlüssel wird mit seiner höherwertigen Hälfte in der Spalte `classid`, seiner niederwertigen Hälfte in der Spalte `objid` und `objsubid` gleich 1 angezeigt. Der ursprüngliche `bigint`-Wert kann mit dem Ausdruck `(classid::bigint << 32) | objid::bigint` wieder zusammengesetzt werden. Integer-Schlüssel werden mit dem ersten Schlüssel in der Spalte `classid`, dem zweiten Schlüssel in der Spalte `objid` und `objsubid` gleich 2 angezeigt. Die tatsächliche Bedeutung der Schlüssel liegt beim Benutzer. Advisory Locks sind jeweils datenbanklokal, daher ist die Spalte `database` für eine Advisory Lock aussagekräftig.

Apply-Transaction-Sperren werden im parallelen Modus verwendet, um die Transaktion in der logischen Replikation anzuwenden. Die entfernte Transaktions-ID wird in der Spalte `transactionid` angezeigt. `objsubid` zeigt den Sperrsubtyp an: 0 für die Sperre, die zur Synchronisierung der Änderungsmenge verwendet wird, und 1 für die Sperre, mit der auf das Ende der Transaktion gewartet wird, um die Commit-Reihenfolge sicherzustellen.

`pg_locks` bietet eine globale Sicht auf alle Sperren im Datenbankcluster, nicht nur auf diejenigen, die für die aktuelle Datenbank relevant sind. Obwohl die Spalte `relation` mit `pg_class.oid` verknüpft werden kann, um gesperrte Relationen zu identifizieren, funktioniert dies nur für Relationen in der aktuellen Datenbank korrekt, also für solche, bei denen die Spalte `database` entweder die OID der aktuellen Datenbank oder `0` ist.

Die Spalte `pid` kann mit der Spalte `pid` der Sicht `pg_stat_activity` verknüpft werden, um weitere Informationen über die Sitzung zu erhalten, die jede Sperre hält oder auf sie wartet, zum Beispiel:

```sql
SELECT * FROM pg_locks pl LEFT JOIN pg_stat_activity psa
    ON pl.pid = psa.pid;
```

Wenn Sie vorbereitete Transaktionen verwenden, kann außerdem die Spalte `virtualtransaction` mit der Spalte `transaction` der Sicht `pg_prepared_xacts` verknüpft werden, um weitere Informationen über vorbereitete Transaktionen zu erhalten, die Sperren halten. Eine vorbereitete Transaktion kann niemals auf eine Sperre warten, hält aber weiterhin die Sperren, die sie während ihrer Ausführung erworben hat. Zum Beispiel:

```sql
SELECT * FROM pg_locks pl LEFT JOIN pg_prepared_xacts ppx
    ON pl.virtualtransaction = '-1/' || ppx.transaction;
```

Zwar ist es möglich, Informationen darüber zu erhalten, welche Prozesse welche anderen Prozesse blockieren, indem man `pg_locks` mit sich selbst verknüpft, doch ist dies im Detail sehr schwer korrekt hinzubekommen. Eine solche Abfrage müsste Wissen darüber kodieren, welche Sperrmodi mit welchen anderen kollidieren. Erschwerend kommt hinzu, dass die Sicht `pg_locks` weder Informationen darüber offenlegt, welche Prozesse in Sperr-Warteschlangen vor welchen anderen stehen, noch Informationen darüber, welche Prozesse parallele Worker sind, die im Auftrag welcher anderen Client-Sitzungen laufen. Es ist besser, die Funktion `pg_blocking_pids()` zu verwenden (siehe Tabelle 9.71), um zu ermitteln, hinter welchem Prozess oder welchen Prozessen ein wartender Prozess blockiert ist.

Die Sicht `pg_locks` zeigt Daten sowohl aus dem regulären Lock Manager als auch aus dem Predicate Lock Manager an; dies sind getrennte Systeme. Außerdem unterteilt der reguläre Lock Manager seine Sperren in reguläre Sperren und Fast-Path-Sperren. Es ist nicht garantiert, dass diese Daten vollständig konsistent sind. Wenn die Sicht abgefragt wird, werden Daten zu Fast-Path-Sperren mit `fastpath = true` nacheinander von jedem Backend gesammelt, ohne den Zustand des gesamten Lock Managers einzufrieren; daher ist es möglich, dass Sperren genommen oder freigegeben werden, während Informationen gesammelt werden. Beachten Sie jedoch, dass diese Sperren bekanntermaßen mit keiner anderen aktuell vorhandenen Sperre kollidieren. Nachdem alle Backends nach Fast-Path-Sperren abgefragt wurden, wird der Rest des regulären Lock Managers als Einheit gesperrt, und ein konsistenter Snapshot aller verbleibenden Sperren wird als atomare Aktion gesammelt. Nach dem Entsperren des regulären Lock Managers wird der Predicate Lock Manager auf ähnliche Weise gesperrt, und alle Predicate Locks werden als atomare Aktion gesammelt. Mit Ausnahme von Fast-Path-Sperren liefert daher jeder Lock Manager eine konsistente Ergebnismenge; da wir aber nicht beide Lock Manager gleichzeitig sperren, ist es möglich, dass Sperren genommen oder freigegeben werden, nachdem der reguläre Lock Manager abgefragt wurde und bevor der Predicate Lock Manager abgefragt wird.

Das Sperren des regulären und/oder des Predicate Lock Managers könnte Auswirkungen auf die Datenbankleistung haben, wenn diese Sicht sehr häufig abgefragt wird. Die Sperren werden nur für die minimale Zeit gehalten, die nötig ist, um Daten aus den Lock Managern zu gewinnen, aber das schließt die Möglichkeit einer Leistungsauswirkung nicht vollständig aus.

## 53.14. pg_matviews

Die Sicht `pg_matviews` bietet Zugriff auf nützliche Informationen über jede materialisierte Sicht in der Datenbank.

**Tabelle 53.14. Spalten von `pg_matviews`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die materialisierte Sicht enthält |
| `matviewname` | `name` (referenziert `pg_class.relname`) | Name der materialisierten Sicht |
| `matviewowner` | `name` (referenziert `pg_authid.rolname`) | Name des Besitzers der materialisierten Sicht |
| `tablespace` | `name` (referenziert `pg_tablespace.spcname`) | Name des Tablespace, der die materialisierte Sicht enthält, oder `NULL`, wenn der Standard der Datenbank verwendet wird |
| `hasindexes` | `bool` | Wahr, wenn die materialisierte Sicht Indizes hat oder kürzlich hatte |
| `ispopulated` | `bool` | Wahr, wenn die materialisierte Sicht derzeit gefüllt ist |
| `definition` | `text` | Definition der materialisierten Sicht, also eine rekonstruierte `SELECT`-Abfrage |

## 53.15. pg_policies

Die Sicht `pg_policies` bietet Zugriff auf nützliche Informationen über jede Row-Level-Security-Policy in der Datenbank.

**Tabelle 53.15. Spalten von `pg_policies`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Tabelle enthält, für die die Policy gilt |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle, für die die Policy gilt |
| `policyname` | `name` (referenziert `pg_policy.polname`) | Name der Policy |
| `permissive` | `text` | Gibt an, ob die Policy permissiv oder restriktiv ist |
| `roles` | `name[]` | Rollen, für die diese Policy gilt |
| `cmd` | `text` | Befehlstyp, auf den die Policy angewendet wird |
| `qual` | `text` | Ausdruck, der zu den Security-Barrier-Qualifikationen für Abfragen hinzugefügt wird, auf die diese Policy angewendet wird |
| `with_check` | `text` | Ausdruck, der zu den `WITH CHECK`-Qualifikationen für Abfragen hinzugefügt wird, die versuchen, Zeilen zu dieser Tabelle hinzuzufügen |

## 53.16. pg_prepared_statements

Die Sicht `pg_prepared_statements` zeigt alle vorbereiteten Anweisungen an, die in der aktuellen Sitzung verfügbar sind. Weitere Informationen zu vorbereiteten Anweisungen finden Sie unter `PREPARE`.

`pg_prepared_statements` enthält eine Zeile für jede vorbereitete Anweisung. Zeilen werden der Sicht hinzugefügt, wenn eine neue vorbereitete Anweisung erzeugt wird, und entfernt, wenn eine vorbereitete Anweisung freigegeben wird, beispielsweise mit dem Kommando `DEALLOCATE`.

**Tabelle 53.16. Spalten von `pg_prepared_statements`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Bezeichner der vorbereiteten Anweisung |
| `statement` | `text` | Abfragezeichenkette, die vom Client übermittelt wurde, um diese vorbereitete Anweisung zu erzeugen. Bei vorbereiteten Anweisungen, die über SQL erzeugt wurden, ist dies die vom Client übermittelte `PREPARE`-Anweisung. Bei vorbereiteten Anweisungen, die über das Frontend/Backend-Protokoll erzeugt wurden, ist dies der Text der vorbereiteten Anweisung selbst. |
| `prepare_time` | `timestamptz` | Zeitpunkt, zu dem die vorbereitete Anweisung erzeugt wurde |
| `parameter_types` | `regtype[]` | Erwartete Parametertypen für die vorbereitete Anweisung in Form eines Arrays von `regtype`. Die OID, die einem Element dieses Arrays entspricht, kann durch Casten des `regtype`-Werts nach `oid` erhalten werden. |
| `result_types` | `regtype[]` | Typen der Spalten, die von der vorbereiteten Anweisung zurückgegeben werden, in Form eines Arrays von `regtype`. Die OID, die einem Element dieses Arrays entspricht, kann durch Casten des `regtype`-Werts nach `oid` erhalten werden. Wenn die vorbereitete Anweisung kein Ergebnis liefert, etwa eine DML-Anweisung, ist dieses Feld `NULL`. |
| `from_sql` | `bool` | Wahr, wenn die vorbereitete Anweisung über das SQL-Kommando `PREPARE` erzeugt wurde; falsch, wenn die Anweisung über das Frontend/Backend-Protokoll vorbereitet wurde |
| `generic_plans` | `int8` | Anzahl der Male, in denen ein generischer Plan gewählt wurde |
| `custom_plans` | `int8` | Anzahl der Male, in denen ein benutzerdefinierter Plan gewählt wurde |

Die Sicht `pg_prepared_statements` ist schreibgeschützt.

## 53.17. pg_prepared_xacts

Die Sicht `pg_prepared_xacts` zeigt Informationen über Transaktionen an, die derzeit für Two-Phase Commit vorbereitet sind. Details finden Sie unter `PREPARE TRANSACTION`.

`pg_prepared_xacts` enthält eine Zeile pro vorbereiteter Transaktion. Ein Eintrag wird entfernt, wenn die Transaktion committet oder zurückgerollt wird.

**Tabelle 53.17. Spalten von `pg_prepared_xacts`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `transaction` | `xid` | Numerischer Transaktionsbezeichner der vorbereiteten Transaktion |
| `gid` | `text` | Globaler Transaktionsbezeichner, der der Transaktion zugewiesen wurde |
| `prepared` | `timestamptz` | Zeitpunkt, zu dem die Transaktion für den Commit vorbereitet wurde |
| `owner` | `name` (referenziert `pg_authid.rolname`) | Name des Benutzers, der die Transaktion ausgeführt hat |
| `database` | `name` (referenziert `pg_database.datname`) | Name der Datenbank, in der die Transaktion ausgeführt wurde |

Wenn auf die Sicht `pg_prepared_xacts` zugegriffen wird, werden die internen Datenstrukturen des Transaktionsmanagers kurzzeitig gesperrt, und für die Anzeige in der Sicht wird eine Kopie erstellt. Dadurch wird sichergestellt, dass die Sicht eine konsistente Ergebnismenge erzeugt, ohne normale Operationen länger als nötig zu blockieren. Dennoch kann es Auswirkungen auf die Datenbankleistung geben, wenn diese Sicht häufig abgefragt wird.

## 53.18. pg_publication_tables

Die Sicht `pg_publication_tables` liefert Informationen über die Zuordnung zwischen Publikationen und den Tabellen, die sie enthalten. Anders als der zugrunde liegende Katalog `pg_publication_rel` erweitert diese Sicht Publikationen, die als `FOR ALL TABLES` und `FOR TABLES IN SCHEMA` definiert sind; für solche Publikationen gibt es daher eine Zeile für jede geeignete Tabelle.

**Tabelle 53.18. Spalten von `pg_publication_tables`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `pubname` | `name` (referenziert `pg_publication.pubname`) | Name der Publikation |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Tabelle enthält |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle |
| `attnames` | `name[]` (referenziert `pg_attribute.attname`) | Namen der Tabellenspalten, die in der Publikation enthalten sind. Dies enthält alle Spalten der Tabelle, wenn der Benutzer keine Spaltenliste für die Tabelle angegeben hat. |
| `rowfilter` | `text` | Ausdruck für die qualifizierende Bedingung der Tabellenpublikation |

## 53.19. pg_replication_origin_status

Die Sicht `pg_replication_origin_status` enthält Informationen darüber, wie weit die Wiedergabe für einen bestimmten Ursprung fortgeschritten ist. Weitere Informationen zu Replikationsursprüngen finden Sie in [Kapitel 48](48_Fortschritt_der_Replikation_verfolgen.md).

**Tabelle 53.19. Spalten von `pg_replication_origin_status`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `local_id` | `oid` (referenziert `pg_replication_origin.roident`) | Interner Knotenbezeichner |
| `external_id` | `text` (referenziert `pg_replication_origin.roname`) | Externer Knotenbezeichner |
| `remote_lsn` | `pg_lsn` | LSN des Ursprungsknotens, bis zu der Daten repliziert wurden |
| `local_lsn` | `pg_lsn` | LSN dieses Knotens, bei der `remote_lsn` repliziert wurde. Wird verwendet, um Commit-Datensätze auf die Platte zu schreiben, bevor Daten bei asynchronen Commits dauerhaft gespeichert werden. |

## 53.20. pg_replication_slots

Die Sicht `pg_replication_slots` listet alle Replikations-Slots auf, die derzeit im Datenbankcluster existieren, zusammen mit ihrem aktuellen Zustand.

Weitere Informationen zu Replikations-Slots finden Sie in Abschnitt 26.2.6 und [Kapitel 47](47_Logische_Decodierung.md).

**Tabelle 53.20. Spalten von `pg_replication_slots`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `slot_name` | `name` | Ein eindeutiger, clusterweiter Bezeichner für den Replikations-Slot |
| `plugin` | `name` | Basisname des Shared Objects, das das Ausgabe-Plugin enthält, das dieser logische Slot verwendet, oder `NULL` für physische Slots |
| `slot_type` | `text` | Slot-Typ: `physical` oder `logical` |
| `datoid` | `oid` (referenziert `pg_database.oid`) | OID der Datenbank, mit der dieser Slot verbunden ist, oder `NULL`. Nur logische Slots haben eine zugehörige Datenbank. |
| `database` | `name` (referenziert `pg_database.datname`) | Name der Datenbank, mit der dieser Slot verbunden ist, oder `NULL`. Nur logische Slots haben eine zugehörige Datenbank. |
| `temporary` | `bool` | Wahr, wenn dies ein temporärer Replikations-Slot ist. Temporäre Slots werden nicht auf Platte gespeichert und bei Fehlern oder beim Ende der Sitzung automatisch entfernt. |
| `active` | `bool` | Wahr, wenn dieser Slot derzeit gestreamt wird |
| `active_pid` | `int4` | Prozess-ID der Sitzung, die Daten für diesen Slot streamt. `NULL`, wenn inaktiv. |
| `xmin` | `xid` | Älteste Transaktion, die dieser Slot von der Datenbank zurückhalten lassen muss. `VACUUM` kann keine Tupel entfernen, die von späteren Transaktionen gelöscht wurden. |
| `catalog_xmin` | `xid` | Älteste Transaktion, die die Systemkataloge betrifft und die dieser Slot von der Datenbank zurückhalten lassen muss. `VACUUM` kann keine Katalogtupel entfernen, die von späteren Transaktionen gelöscht wurden. |
| `restart_lsn` | `pg_lsn` | Adresse (LSN) des ältesten WAL, das vom Verbraucher dieses Slots noch benötigt werden könnte und daher während Checkpoints nicht automatisch entfernt wird, sofern diese LSN nicht um mehr als `max_slot_wal_keep_size` hinter die aktuelle LSN zurückfällt. `NULL`, wenn für diesen Slot noch nie eine LSN reserviert wurde. |
| `confirmed_flush_lsn` | `pg_lsn` | Adresse (LSN), bis zu der der Verbraucher des logischen Slots den Empfang von Daten bestätigt hat. Daten zu Transaktionen, die vor dieser LSN committet wurden, sind nicht mehr verfügbar. `NULL` für physische Slots. |
| `wal_status` | `text` | Verfügbarkeit der von diesem Slot beanspruchten WAL-Dateien. Mögliche Werte sind: `reserved`, die beanspruchten Dateien liegen innerhalb von `max_wal_size`; `extended`, `max_wal_size` ist überschritten, die Dateien werden aber weiterhin entweder durch den Replikations-Slot oder durch `wal_keep_size` zurückgehalten; `unreserved`, der Slot hält die erforderlichen WAL-Dateien nicht mehr zurück, und einige davon sollen beim nächsten Checkpoint entfernt werden. Dies tritt typischerweise auf, wenn `max_slot_wal_keep_size` auf einen nicht negativen Wert gesetzt ist. Dieser Zustand kann zu `reserved` oder `extended` zurückkehren; `lost`, dieser Slot ist nicht mehr verwendbar. |
| `safe_wal_size` | `int8` | Anzahl der Bytes, die in das WAL geschrieben werden können, ohne dass dieser Slot Gefahr läuft, in den Zustand `lost` zu geraten. Für verlorene Slots ist dies `NULL`, ebenso wenn `max_slot_wal_keep_size` `-1` ist. |
| `two_phase` | `bool` | Wahr, wenn der Slot für das Decoding vorbereiteter Transaktionen aktiviert ist. Für physische Slots immer falsch. |
| `two_phase_at` | `pg_lsn` | Adresse (LSN), ab der das Decoding vorbereiteter Transaktionen aktiviert ist. `NULL` für logische Slots, bei denen `two_phase` falsch ist, und für physische Slots. |
| `inactive_since` | `timestamptz` | Zeitpunkt, zu dem der Slot inaktiv wurde. `NULL`, wenn der Slot derzeit gestreamt wird. Wenn der Slot ungültig wird, wird dieser Wert nie mehr aktualisiert. Für Standby-Slots, die von einem Primärserver synchronisiert werden und deren Feld `synced` wahr ist, gibt `inactive_since` den Zeitpunkt an, zu dem die Slot-Synchronisierung (siehe Abschnitt 47.2.3) zuletzt beendet wurde. `NULL`, wenn der Slot immer synchronisiert war. Dies hilft Standby-Slots nachzuverfolgen, wann die Synchronisierung unterbrochen wurde. |
| `conflicting` | `bool` | Wahr, wenn dieser logische Slot mit der Recovery kollidiert ist und deshalb ungültig gemacht wurde. Wenn diese Spalte wahr ist, enthält die Spalte `invalidation_reason` den Konfliktgrund. Für physische Slots immer `NULL`. |
| `invalidation_reason` | `text` | Grund für die Ungültigmachung des Slots. Er wird sowohl für logische als auch für physische Slots gesetzt. `NULL`, wenn der Slot nicht ungültig gemacht wurde. Mögliche Werte sind: `wal_removed`, das erforderliche WAL wurde entfernt; `rows_removed`, die erforderlichen Zeilen wurden entfernt, nur für logische Slots; `wal_level_insufficient`, der Primärserver hat kein für logisches Decoding ausreichendes `wal_level`, nur für logische Slots; `idle_timeout`, der Slot blieb länger inaktiv als die konfigurierte Dauer `idle_replication_slot_timeout`. |
| `failover` | `bool` | Wahr, wenn dies ein logischer Slot ist, der für die Synchronisierung auf Standbys aktiviert ist, sodass die logische Replikation nach einem Failover vom neuen Primärserver aus fortgesetzt werden kann. Für physische Slots immer falsch. |
| `synced` | `bool` | Wahr, wenn dies ein logischer Slot ist, der von einem Primärserver synchronisiert wurde. Auf einem Hot Standby können Slots, deren Spalte `synced` als wahr markiert ist, weder für logisches Decoding verwendet noch manuell entfernt werden. Der Wert dieser Spalte hat auf dem Primärserver keine Bedeutung; auf dem Primärserver ist der Spaltenwert standardmäßig für alle Slots falsch, kann aber, falls von einem beförderten Standby übrig geblieben, auch wahr sein. |

## 53.21. pg_roles

Die Sicht `pg_roles` bietet Zugriff auf Informationen über Datenbankrollen. Sie ist einfach eine öffentlich lesbare Sicht auf `pg_authid`, bei der das Passwortfeld ausgeblendet wird.

**Tabelle 53.21. Spalten von `pg_roles`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `rolname` | `name` | Rollenname |
| `rolsuper` | `bool` | Die Rolle hat Superuser-Rechte |
| `rolinherit` | `bool` | Die Rolle erbt automatisch Rechte von Rollen, deren Mitglied sie ist |
| `rolcreaterole` | `bool` | Die Rolle kann weitere Rollen erstellen |
| `rolcreatedb` | `bool` | Die Rolle kann Datenbanken erstellen |
| `rolcanlogin` | `bool` | Die Rolle kann sich anmelden. Das heißt, diese Rolle kann als anfänglicher Sitzungsautorisierungsbezeichner angegeben werden. |
| `rolreplication` | `bool` | Die Rolle ist eine Replikationsrolle. Eine Replikationsrolle kann Replikationsverbindungen initiieren sowie Replikations-Slots erstellen und entfernen. |
| `rolconnlimit` | `int4` | Für Rollen, die sich anmelden können, legt dies die maximale Anzahl gleichzeitiger Verbindungen fest, die diese Rolle herstellen kann. `-1` bedeutet keine Begrenzung. |
| `rolpassword` | `text` | Nicht das Passwort; liest sich immer als `********` |
| `rolvaliduntil` | `timestamptz` | Ablaufzeit des Passworts, nur für Passwortauthentifizierung verwendet; `NULL`, wenn es keinen Ablauf gibt |
| `rolbypassrls` | `bool` | Die Rolle umgeht jede Row-Level-Security-Policy; weitere Informationen finden Sie in [Abschnitt 5.9](05_Datendefinition.md#59-rowsecuritypolicies). |
| `rolconfig` | `text[]` | Rollenspezifische Standardwerte für Laufzeit-Konfigurationsvariablen |
| `oid` | `oid` (referenziert `pg_authid.oid`) | ID der Rolle |

## 53.22. pg_rules

Die Sicht `pg_rules` bietet Zugriff auf nützliche Informationen über Query-Rewrite-Regeln.

**Tabelle 53.22. Spalten von `pg_rules`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Tabelle enthält |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle, für die die Regel gilt |
| `rulename` | `name` (referenziert `pg_rewrite.rulename`) | Name der Regel |
| `definition` | `text` | Regeldefinition, also ein rekonstruiertes Erstellungskommando |

Die Sicht `pg_rules` schließt die `ON SELECT`-Regeln von Sichten und materialisierten Sichten aus; diese sind in `pg_views` und `pg_matviews` sichtbar.

## 53.23. pg_seclabels

Die Sicht `pg_seclabels` liefert Informationen über Sicherheitslabels. Sie ist eine leichter abfragbare Version des Katalogs `pg_seclabel`.

**Tabelle 53.23. Spalten von `pg_seclabels`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `objoid` | `oid` (referenziert eine beliebige OID-Spalte) | OID des Objekts, zu dem dieses Sicherheitslabel gehört |
| `classoid` | `oid` (referenziert `pg_class.oid`) | OID des Systemkatalogs, in dem dieses Objekt erscheint |
| `objsubid` | `int4` | Bei einem Sicherheitslabel auf einer Tabellenspalte ist dies die Spaltennummer; `objoid` und `classoid` beziehen sich auf die Tabelle selbst. Für alle anderen Objekttypen ist diese Spalte null. |
| `objtype` | `text` | Typ des Objekts, auf das dieses Label angewendet wird, als Text |
| `objnamespace` | `oid` (referenziert `pg_namespace.oid`) | OID des Namespace für dieses Objekt, falls anwendbar; andernfalls `NULL` |
| `objname` | `text` | Name des Objekts, auf das dieses Label angewendet wird, als Text |
| `provider` | `text` (referenziert `pg_seclabel.provider`) | Label-Provider, der mit diesem Label verbunden ist |
| `label` | `text` (referenziert `pg_seclabel.label`) | Sicherheitslabel, das auf dieses Objekt angewendet wurde |

## 53.24. pg_sequences

Die Sicht `pg_sequences` bietet Zugriff auf nützliche Informationen über jede Sequenz in der Datenbank.

**Tabelle 53.24. Spalten von `pg_sequences`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Sequenz enthält |
| `sequencename` | `name` (referenziert `pg_class.relname`) | Name der Sequenz |
| `sequenceowner` | `name` (referenziert `pg_authid.rolname`) | Name des Besitzers der Sequenz |
| `data_type` | `regtype` (referenziert `pg_type.oid`) | Datentyp der Sequenz |
| `start_value` | `int8` | Startwert der Sequenz |
| `min_value` | `int8` | Minimalwert der Sequenz |
| `max_value` | `int8` | Maximalwert der Sequenz |
| `increment_by` | `int8` | Schrittweite der Sequenz |
| `cycle` | `bool` | Gibt an, ob die Sequenz zyklisch ist |
| `cache_size` | `int8` | Cache-Größe der Sequenz |
| `last_value` | `int8` | Letzter auf Platte geschriebener Sequenzwert. Wenn Caching verwendet wird, kann dieser Wert größer sein als der letzte von der Sequenz ausgegebene Wert. |

Die Spalte `last_value` liest sich als `NULL`, wenn eine der folgenden Bedingungen zutrifft:

- Die Sequenz wurde bisher noch nicht gelesen.
- Der aktuelle Benutzer hat kein Recht `USAGE` oder `SELECT` auf der Sequenz.
- Die Sequenz ist unlogged und der Server ist ein Standby.

## 53.25. pg_settings

Die Sicht `pg_settings` bietet Zugriff auf die Laufzeitparameter des Servers. Sie ist im Wesentlichen eine alternative Schnittstelle zu den Kommandos `SHOW` und `SET`. Außerdem bietet sie Zugriff auf einige Fakten über jeden Parameter, die über `SHOW` nicht direkt verfügbar sind, etwa Minimal- und Maximalwerte.

**Tabelle 53.25. Spalten von `pg_settings`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Name des Laufzeit-Konfigurationsparameters |
| `setting` | `text` | Aktueller Wert des Parameters |
| `unit` | `text` | Implizite Einheit des Parameters |
| `category` | `text` | Logische Gruppe des Parameters |
| `short_desc` | `text` | Kurze Beschreibung des Parameters |
| `extra_desc` | `text` | Zusätzliche, ausführlichere Beschreibung des Parameters |
| `context` | `text` | Kontext, der erforderlich ist, um den Wert des Parameters zu setzen; siehe unten |
| `vartype` | `text` | Parametertyp: `bool`, `enum`, `integer`, `real` oder `string` |
| `source` | `text` | Quelle des aktuellen Parameterwerts |
| `min_val` | `text` | Minimal zulässiger Wert des Parameters; `NULL` für nicht numerische Werte |
| `max_val` | `text` | Maximal zulässiger Wert des Parameters; `NULL` für nicht numerische Werte |
| `enumvals` | `text[]` | Zulässige Werte eines Enum-Parameters; `NULL` für Nicht-Enum-Werte |
| `boot_val` | `text` | Parameterwert, der beim Serverstart angenommen wird, wenn der Parameter nicht anderweitig gesetzt ist |
| `reset_val` | `text` | Wert, auf den `RESET` den Parameter in der aktuellen Sitzung zurücksetzen würde |
| `sourcefile` | `text` | Konfigurationsdatei, in der der aktuelle Wert gesetzt wurde; `NULL` für Werte aus anderen Quellen als Konfigurationsdateien oder wenn ein Benutzer nachsieht, der weder Superuser ist noch Rechte von `pg_read_all_settings` hat. Dies ist hilfreich bei der Verwendung von Include-Direktiven in Konfigurationsdateien. |
| `sourceline` | `int4` | Zeilennummer innerhalb der Konfigurationsdatei, an der der aktuelle Wert gesetzt wurde; `NULL` für Werte aus anderen Quellen als Konfigurationsdateien oder wenn ein Benutzer nachsieht, der weder Superuser ist noch Rechte von `pg_read_all_settings` hat. |
| `pending_restart` | `bool` | Wahr, wenn der Wert in der Konfigurationsdatei geändert wurde, aber einen Neustart benötigt; andernfalls falsch |

Für `context` gibt es mehrere mögliche Werte. In absteigender Schwierigkeit der Änderung der Einstellung sind dies:

- `internal`
  - Diese Einstellungen können nicht direkt geändert werden; sie spiegeln intern bestimmte Werte wider. Einige davon können möglicherweise durch Neubauen des Servers mit anderen Konfigurationsoptionen oder durch Ändern der an `initdb` übergebenen Optionen angepasst werden.

- `postmaster`
  - Diese Einstellungen können nur angewendet werden, wenn der Server startet; jede Änderung erfordert daher einen Neustart des Servers. Werte für diese Einstellungen werden typischerweise in der Datei `postgresql.conf` gespeichert oder beim Start des Servers auf der Kommandozeile übergeben. Natürlich können auch Einstellungen mit einem der niedrigeren Kontexttypen beim Serverstart gesetzt werden.

- `sighup`
  - Änderungen an diesen Einstellungen können in `postgresql.conf` vorgenommen werden, ohne den Server neu zu starten. Senden Sie ein `SIGHUP`-Signal an den Postmaster, damit er `postgresql.conf` erneut liest und die Änderungen anwendet. Der Postmaster leitet das `SIGHUP`-Signal auch an seine Kindprozesse weiter, sodass alle den neuen Wert übernehmen.

- `superuser-backend`
  - Änderungen an diesen Einstellungen können in `postgresql.conf` vorgenommen werden, ohne den Server neu zu starten. Sie können auch für eine bestimmte Sitzung im Verbindungsanforderungspaket gesetzt werden, zum Beispiel über die libpq-Umgebungsvariable `PGOPTIONS`, aber nur, wenn der verbindende Benutzer ein Superuser ist oder das passende `SET`-Recht erhalten hat. Diese Einstellungen ändern sich jedoch innerhalb einer Sitzung nie, nachdem sie gestartet wurde. Wenn Sie sie in `postgresql.conf` ändern, senden Sie ein `SIGHUP`-Signal an den Postmaster, damit er `postgresql.conf` erneut liest. Die neuen Werte wirken sich nur auf danach gestartete Sitzungen aus.

- `backend`
  - Änderungen an diesen Einstellungen können in `postgresql.conf` vorgenommen werden, ohne den Server neu zu starten. Sie können auch für eine bestimmte Sitzung im Verbindungsanforderungspaket gesetzt werden, zum Beispiel über die libpq-Umgebungsvariable `PGOPTIONS`; jeder Benutzer kann eine solche Änderung für seine Sitzung vornehmen. Diese Einstellungen ändern sich jedoch innerhalb einer Sitzung nie, nachdem sie gestartet wurde. Wenn Sie sie in `postgresql.conf` ändern, senden Sie ein `SIGHUP`-Signal an den Postmaster, damit er `postgresql.conf` erneut liest. Die neuen Werte wirken sich nur auf danach gestartete Sitzungen aus.

- `superuser`
  - Diese Einstellungen können aus `postgresql.conf` oder innerhalb einer Sitzung mit dem Kommando `SET` gesetzt werden; über `SET` können sie aber nur Superuser und Benutzer mit dem passenden `SET`-Recht ändern. Änderungen in `postgresql.conf` wirken sich nur dann auf bestehende Sitzungen aus, wenn kein sitzungslokaler Wert mit `SET` festgelegt wurde.

- `user`
  - Diese Einstellungen können aus `postgresql.conf` oder innerhalb einer Sitzung mit dem Kommando `SET` gesetzt werden. Jeder Benutzer darf seinen sitzungslokalen Wert ändern. Änderungen in `postgresql.conf` wirken sich nur dann auf bestehende Sitzungen aus, wenn kein sitzungslokaler Wert mit `SET` festgelegt wurde.

Weitere Informationen zu den verschiedenen Möglichkeiten, diese Parameter zu ändern, finden Sie in [Abschnitt 19.1](19_Serverkonfiguration.md#191-parameter-setzen).

In diese Sicht kann nicht eingefügt und aus ihr kann nicht gelöscht werden, aber sie kann aktualisiert werden. Ein `UPDATE`, das auf eine Zeile von `pg_settings` angewendet wird, entspricht der Ausführung des Kommandos `SET` für den genannten Parameter. Die Änderung betrifft nur den Wert, der von der aktuellen Sitzung verwendet wird. Wenn ein `UPDATE` innerhalb einer Transaktion ausgeführt wird, die später abgebrochen wird, verschwinden die Auswirkungen des `UPDATE`-Kommandos beim Zurückrollen der Transaktion. Sobald die umgebende Transaktion committet ist, bleiben die Auswirkungen bis zum Ende der Sitzung bestehen, sofern sie nicht durch ein anderes `UPDATE` oder `SET` überschrieben werden.

Diese Sicht zeigt keine angepassten Optionen an, es sei denn, das Erweiterungsmodul, das sie definiert, wurde von dem Backend-Prozess geladen, der die Abfrage ausführt, zum Beispiel durch eine Erwähnung in `shared_preload_libraries`, einen Aufruf einer C-Funktion in der Erweiterung oder das Kommando `LOAD`. Da Archivmodule normalerweise nur vom Archiver-Prozess und nicht von regulären Sitzungen geladen werden, zeigt diese Sicht beispielsweise keine angepassten Optionen an, die von solchen Modulen definiert wurden, sofern nicht besondere Maßnahmen getroffen werden, um sie in den Backend-Prozess zu laden, der die Abfrage ausführt.

## 53.26. pg_shadow

Die Sicht `pg_shadow` existiert aus Gründen der Rückwärtskompatibilität: Sie emuliert einen Katalog, der in PostgreSQL vor Version 8.1 existierte. Sie zeigt Eigenschaften aller Rollen, die in `pg_authid` als `rolcanlogin` markiert sind.

Der Name stammt daher, dass diese Tabelle nicht öffentlich lesbar sein sollte, da sie Passwörter enthält. `pg_user` ist eine öffentlich lesbare Sicht auf `pg_shadow`, bei der das Passwortfeld ausgeblendet wird.

**Tabelle 53.26. Spalten von `pg_shadow`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `usename` | `name` (referenziert `pg_authid.rolname`) | Benutzername |
| `usesysid` | `oid` (referenziert `pg_authid.oid`) | ID dieses Benutzers |
| `usecreatedb` | `bool` | Der Benutzer kann Datenbanken erstellen |
| `usesuper` | `bool` | Der Benutzer ist ein Superuser |
| `userepl` | `bool` | Der Benutzer kann Streaming-Replikation initiieren und das System in den Backup-Modus versetzen sowie daraus zurückholen |
| `usebypassrls` | `bool` | Der Benutzer umgeht jede Row-Level-Security-Policy; weitere Informationen finden Sie in [Abschnitt 5.9](05_Datendefinition.md#59-rowsecuritypolicies). |
| `passwd` | `text` | Verschlüsseltes Passwort oder `NULL`, wenn keines vorhanden ist. Details dazu, wie verschlüsselte Passwörter gespeichert werden, finden Sie bei `pg_authid`. |
| `valuntil` | `timestamptz` | Ablaufzeit des Passworts, nur für Passwortauthentifizierung verwendet |
| `useconfig` | `text[]` | Sitzungs-Standardwerte für Laufzeit-Konfigurationsvariablen |

## 53.27. pg_shmem_allocations

Die Sicht `pg_shmem_allocations` zeigt Allokationen aus dem Haupt-Shared-Memory-Segment des Servers. Dazu gehört sowohl Speicher, der von PostgreSQL selbst alloziert wurde, als auch Speicher, den Erweiterungen mit den in Abschnitt 36.10.11 beschriebenen Mechanismen alloziert haben.

Beachten Sie, dass diese Sicht keinen Speicher enthält, der über die Infrastruktur für dynamischen Shared Memory alloziert wurde.

**Tabelle 53.27. Spalten von `pg_shmem_allocations`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Name der Shared-Memory-Allokation. `NULL` für ungenutzten Speicher und `<anonymous>` für anonyme Allokationen. |
| `off` | `int8` | Offset, an dem die Allokation beginnt. `NULL` für anonyme Allokationen, da die zugehörigen Details nicht bekannt sind. |
| `size` | `int8` | Größe der Allokation in Byte |
| `allocated_size` | `int8` | Größe der Allokation in Byte einschließlich Padding. Für anonyme Allokationen sind keine Informationen über Padding verfügbar, daher sind die Spalten `size` und `allocated_size` immer gleich. Für freien Speicher ist Padding ebenfalls nicht relevant, daher sind die Spalten auch in diesem Fall gleich. |

Anonyme Allokationen sind Allokationen, die direkt mit `ShmemAlloc()` vorgenommen wurden, nicht über `ShmemInitStruct()` oder `ShmemInitHash()`.

Standardmäßig kann die Sicht `pg_shmem_allocations` nur von Superusern oder Rollen mit den Rechten der Rolle `pg_read_all_stats` gelesen werden.

## 53.28. pg_shmem_allocations_numa

Die Sicht `pg_shmem_allocations_numa` zeigt, wie Shared-Memory-Allokationen im Haupt-Shared-Memory-Segment des Servers über NUMA-Knoten verteilt sind. Dazu gehört sowohl Speicher, der von PostgreSQL selbst alloziert wurde, als auch Speicher, den Erweiterungen mit den in Abschnitt 36.10.11 beschriebenen Mechanismen alloziert haben. Diese Sicht gibt mehrere Zeilen für jedes Shared-Memory-Segment aus, sofern es über mehrere NUMA-Knoten verteilt ist. Diese Sicht sollte nicht von Monitoring-Systemen abgefragt werden, da sie sehr langsam ist und unter Umständen Shared Memory alloziert, falls dieser zuvor nicht verwendet wurde. Eine aktuelle Einschränkung dieser Sicht ist, dass sie keine anonymen Shared-Memory-Allokationen anzeigt.

Beachten Sie, dass diese Sicht keinen Speicher enthält, der über die Infrastruktur für dynamischen Shared Memory alloziert wurde.

> **Warnung**
>
> Beim Bestimmen des NUMA-Knotens berührt die Sicht alle Speicherseiten des Shared-Memory-Segments. Dadurch wird die Allokation des Shared Memory erzwungen, falls er nicht bereits alloziert war, und der Speicher kann, abhängig von der Systemkonfiguration, auf einem einzelnen NUMA-Knoten alloziert werden.

**Tabelle 53.28. Spalten von `pg_shmem_allocations_numa`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Name der Shared-Memory-Allokation |
| `numa_node` | `int4` | ID des NUMA-Knotens |
| `size` | `int8` | Größe der Allokation auf diesem bestimmten NUMA-Speicherknoten in Byte |

Standardmäßig kann die Sicht `pg_shmem_allocations_numa` nur von Superusern oder Rollen mit den Rechten der Rolle `pg_read_all_stats` gelesen werden.

## 53.29. pg_stats

Die Sicht `pg_stats` bietet Zugriff auf die Informationen, die im Katalog `pg_statistic` gespeichert sind. Diese Sicht erlaubt nur Zugriff auf Zeilen von `pg_statistic`, die zu Tabellen gehören, für die der Benutzer Leserechte hat; daher ist es sicher, öffentlichen Lesezugriff auf diese Sicht zu erlauben.

`pg_stats` ist außerdem darauf ausgelegt, die Informationen in einem besser lesbaren Format darzustellen als der zugrunde liegende Katalog. Der Preis dafür ist, dass ihr Schema erweitert werden muss, sobald neue Slot-Typen für `pg_statistic` definiert werden.

**Tabelle 53.29. Spalten von `pg_stats`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Tabelle enthält |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle |
| `attname` | `name` (referenziert `pg_attribute.attname`) | Name der Spalte, die durch diese Zeile beschrieben wird |
| `inherited` | `bool` | Wenn wahr, enthält diese Zeile Werte aus Kindtabellen, nicht nur die Werte in der angegebenen Tabelle |
| `null_frac` | `float4` | Anteil der Spalteneinträge, die `NULL` sind |
| `avg_width` | `int4` | Durchschnittliche Breite der Spalteneinträge in Byte |
| `n_distinct` | `float4` | Wenn größer als null, die geschätzte Anzahl unterschiedlicher Werte in der Spalte. Wenn kleiner als null, der negative Wert der Anzahl unterschiedlicher Werte geteilt durch die Anzahl der Zeilen. Die negierte Form wird verwendet, wenn `ANALYZE` annimmt, dass die Anzahl unterschiedlicher Werte wahrscheinlich mit dem Wachstum der Tabelle zunimmt; die positive Form wird verwendet, wenn die Spalte anscheinend eine feste Anzahl möglicher Werte hat. Beispielsweise gibt `-1` eine eindeutige Spalte an, in der die Anzahl unterschiedlicher Werte der Anzahl der Zeilen entspricht. |
| `most_common_vals` | `anyarray` | Liste der häufigsten Werte in der Spalte. `NULL`, wenn keine Werte häufiger zu sein scheinen als andere. |
| `most_common_freqs` | `float4[]` | Liste der Häufigkeiten der häufigsten Werte, also Anzahl der Vorkommen jedes Werts geteilt durch die Gesamtzahl der Zeilen. `NULL`, wenn `most_common_vals` `NULL` ist. |
| `histogram_bounds` | `anyarray` | Liste von Werten, die die Werte der Spalte in Gruppen mit ungefähr gleicher Besetzung aufteilen. Werte in `most_common_vals` werden, falls vorhanden, bei dieser Histogrammberechnung ausgelassen. Diese Spalte ist `NULL`, wenn der Datentyp der Spalte keinen Operator `<` hat oder wenn die Liste `most_common_vals` die gesamte Population abdeckt. |
| `correlation` | `float4` | Statistische Korrelation zwischen physischer Zeilenreihenfolge und logischer Reihenfolge der Spaltenwerte. Der Bereich reicht von `-1` bis `+1`. Wenn der Wert nahe `-1` oder `+1` liegt, wird ein Index Scan auf der Spalte wegen weniger zufälliger Plattenzugriffe günstiger geschätzt als bei einem Wert nahe null. Diese Spalte ist `NULL`, wenn der Datentyp der Spalte keinen Operator `<` hat. |
| `most_common_elems` | `anyarray` | Liste der Nicht-`NULL`-Elementwerte, die innerhalb der Werte der Spalte am häufigsten vorkommen. `NULL` für skalare Typen. |
| `most_common_elem_freqs` | `float4[]` | Liste der Häufigkeiten der häufigsten Elementwerte, also der Anteil der Zeilen, die mindestens ein Vorkommen des jeweiligen Werts enthalten. Zwei oder drei zusätzliche Werte folgen den Häufigkeiten pro Element; dies sind Minimum und Maximum der vorhergehenden Häufigkeiten pro Element und optional die Häufigkeit von `NULL`-Elementen. `NULL`, wenn `most_common_elems` `NULL` ist. |
| `elem_count_histogram` | `float4[]` | Histogramm der Anzahlen unterschiedlicher Nicht-`NULL`-Elementwerte innerhalb der Werte der Spalte, gefolgt von der durchschnittlichen Anzahl unterschiedlicher Nicht-`NULL`-Elemente. `NULL` für skalare Typen. |
| `range_length_histogram` | `anyarray` | Histogramm der Längen nicht leerer und nicht `NULL`-Range-Werte einer Spalte mit Range-Typ. `NULL` für Nicht-Range-Typen. Dieses Histogramm wird unabhängig davon, ob Range-Grenzen inklusiv sind, mit der Range-Funktion `subtype_diff` berechnet. |
| `range_empty_frac` | `float4` | Anteil der Spalteneinträge, deren Werte leere Ranges sind. `NULL` für Nicht-Range-Typen. |
| `range_bounds_histogram` | `anyarray` | Histogramm der unteren und oberen Grenzen nicht leerer und nicht `NULL`-Range-Werte. `NULL` für Nicht-Range-Typen. Diese beiden Histogramme werden als einzelnes Array von Ranges dargestellt, deren untere Grenzen das Histogramm der unteren Grenzen und deren obere Grenzen das Histogramm der oberen Grenzen darstellen. |

Die maximale Anzahl von Einträgen in den Array-Feldern kann spaltenweise mit dem Kommando `ALTER TABLE SET STATISTICS` oder global durch Setzen des Laufzeitparameters `default_statistics_target` gesteuert werden.

## 53.30. pg_stats_ext

Die Sicht `pg_stats_ext` bietet Zugriff auf Informationen über jedes erweiterte Statistikobjekt in der Datenbank, indem sie Informationen kombiniert, die in den Katalogen `pg_statistic_ext` und `pg_statistic_ext_data` gespeichert sind. Diese Sicht erlaubt nur Zugriff auf Zeilen von `pg_statistic_ext` und `pg_statistic_ext_data`, die zu Tabellen gehören, deren Besitzer der Benutzer ist; daher ist es sicher, öffentlichen Lesezugriff auf diese Sicht zu erlauben.

`pg_stats_ext` ist außerdem darauf ausgelegt, die Informationen in einem besser lesbaren Format darzustellen als die zugrunde liegenden Kataloge. Der Preis dafür ist, dass ihr Schema erweitert werden muss, sobald neue Typen erweiterter Statistiken zu `pg_statistic_ext` hinzugefügt werden.

**Tabelle 53.30. Spalten von `pg_stats_ext`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Tabelle enthält |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle |
| `statistics_schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das das erweiterte Statistikobjekt enthält |
| `statistics_name` | `name` (referenziert `pg_statistic_ext.stxname`) | Name des erweiterten Statistikobjekts |
| `statistics_owner` | `name` (referenziert `pg_authid.rolname`) | Besitzer des erweiterten Statistikobjekts |
| `attnames` | `name[]` (referenziert `pg_attribute.attname`) | Namen der Spalten, die im erweiterten Statistikobjekt enthalten sind |
| `exprs` | `text[]` | Ausdrücke, die im erweiterten Statistikobjekt enthalten sind |
| `kinds` | `char[]` | Typen erweiterter Statistikobjekte, die für diesen Datensatz aktiviert sind |
| `inherited` | `bool` (referenziert `pg_statistic_ext_data.stxdinherit`) | Wenn wahr, enthalten die Statistiken Werte aus Kindtabellen, nicht nur die Werte in der angegebenen Relation |
| `n_distinct` | `pg_ndistinct` | N-distinct-Zählungen für Kombinationen von Spaltenwerten. Wenn größer als null, die geschätzte Anzahl unterschiedlicher Werte in der Kombination. Wenn kleiner als null, der negative Wert der Anzahl unterschiedlicher Werte geteilt durch die Anzahl der Zeilen. Die negierte Form wird verwendet, wenn `ANALYZE` annimmt, dass die Anzahl unterschiedlicher Werte wahrscheinlich mit dem Wachstum der Tabelle zunimmt; die positive Form wird verwendet, wenn die Spalte anscheinend eine feste Anzahl möglicher Werte hat. Beispielsweise gibt `-1` eine eindeutige Spaltenkombination an, in der die Anzahl unterschiedlicher Kombinationen der Anzahl der Zeilen entspricht. |
| `dependencies` | `pg_dependencies` | Statistiken zu funktionalen Abhängigkeiten |
| `most_common_vals` | `text[]` | Liste der häufigsten Wertkombinationen in den Spalten. `NULL`, wenn keine Kombinationen häufiger zu sein scheinen als andere. |
| `most_common_val_nulls` | `bool[]` | Liste von `NULL`-Kennzeichen für die häufigsten Wertkombinationen. `NULL`, wenn `most_common_vals` `NULL` ist. |
| `most_common_freqs` | `float8[]` | Liste der Häufigkeiten der häufigsten Kombinationen, also Anzahl der Vorkommen jeder Kombination geteilt durch die Gesamtzahl der Zeilen. `NULL`, wenn `most_common_vals` `NULL` ist. |
| `most_common_base_freqs` | `float8[]` | Liste der Basis-Häufigkeiten der häufigsten Kombinationen, also Produkt der Häufigkeiten pro Wert. `NULL`, wenn `most_common_vals` `NULL` ist. |

Die maximale Anzahl von Einträgen in den Array-Feldern kann spaltenweise mit dem Kommando `ALTER TABLE SET STATISTICS` oder global durch Setzen des Laufzeitparameters `default_statistics_target` gesteuert werden.

## 53.31. pg_stats_ext_exprs

Die Sicht `pg_stats_ext_exprs` bietet Zugriff auf Informationen über alle Ausdrücke, die in erweiterten Statistikobjekten enthalten sind, indem sie Informationen kombiniert, die in den Katalogen `pg_statistic_ext` und `pg_statistic_ext_data` gespeichert sind. Diese Sicht erlaubt nur Zugriff auf Zeilen von `pg_statistic_ext` und `pg_statistic_ext_data`, die zu Tabellen gehören, deren Besitzer der Benutzer ist; daher ist es sicher, öffentlichen Lesezugriff auf diese Sicht zu erlauben.

`pg_stats_ext_exprs` ist außerdem darauf ausgelegt, die Informationen in einem besser lesbaren Format darzustellen als die zugrunde liegenden Kataloge. Der Preis dafür ist, dass ihr Schema erweitert werden muss, sobald sich die Struktur der Statistiken in `pg_statistic_ext` ändert.

**Tabelle 53.31. Spalten von `pg_stats_ext_exprs`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Tabelle enthält |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle, für die das Statistikobjekt definiert ist |
| `statistics_schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das das erweiterte Statistikobjekt enthält |
| `statistics_name` | `name` (referenziert `pg_statistic_ext.stxname`) | Name des erweiterten Statistikobjekts |
| `statistics_owner` | `name` (referenziert `pg_authid.rolname`) | Besitzer des erweiterten Statistikobjekts |
| `expr` | `text` | Ausdruck, der im erweiterten Statistikobjekt enthalten ist |
| `inherited` | `bool` (referenziert `pg_statistic_ext_data.stxdinherit`) | Wenn wahr, enthalten die Statistiken Werte aus Kindtabellen, nicht nur die Werte in der angegebenen Relation |
| `null_frac` | `float4` | Anteil der Ausdruckseinträge, die `NULL` sind |
| `avg_width` | `int4` | Durchschnittliche Breite der Ausdruckseinträge in Byte |
| `n_distinct` | `float4` | Wenn größer als null, die geschätzte Anzahl unterschiedlicher Werte im Ausdruck. Wenn kleiner als null, der negative Wert der Anzahl unterschiedlicher Werte geteilt durch die Anzahl der Zeilen. Die negierte Form wird verwendet, wenn `ANALYZE` annimmt, dass die Anzahl unterschiedlicher Werte wahrscheinlich mit dem Wachstum der Tabelle zunimmt; die positive Form wird verwendet, wenn der Ausdruck anscheinend eine feste Anzahl möglicher Werte hat. Beispielsweise gibt `-1` einen eindeutigen Ausdruck an, in dem die Anzahl unterschiedlicher Werte der Anzahl der Zeilen entspricht. |
| `most_common_vals` | `anyarray` | Liste der häufigsten Werte im Ausdruck. `NULL`, wenn keine Werte häufiger zu sein scheinen als andere. |
| `most_common_freqs` | `float4[]` | Liste der Häufigkeiten der häufigsten Werte, also Anzahl der Vorkommen jedes Werts geteilt durch die Gesamtzahl der Zeilen. `NULL`, wenn `most_common_vals` `NULL` ist. |
| `histogram_bounds` | `anyarray` | Liste von Werten, die die Werte des Ausdrucks in Gruppen mit ungefähr gleicher Besetzung aufteilen. Werte in `most_common_vals` werden, falls vorhanden, bei dieser Histogrammberechnung ausgelassen. Dieser Ausdruck ist `NULL`, wenn der Datentyp des Ausdrucks keinen Operator `<` hat oder wenn die Liste `most_common_vals` die gesamte Population abdeckt. |
| `correlation` | `float4` | Statistische Korrelation zwischen physischer Zeilenreihenfolge und logischer Reihenfolge der Ausdruckswerte. Der Bereich reicht von `-1` bis `+1`. Wenn der Wert nahe `-1` oder `+1` liegt, wird ein Index Scan auf dem Ausdruck wegen weniger zufälliger Plattenzugriffe günstiger geschätzt als bei einem Wert nahe null. Dieser Ausdruck ist `NULL`, wenn der Datentyp des Ausdrucks keinen Operator `<` hat. |
| `most_common_elems` | `anyarray` | Liste der Nicht-`NULL`-Elementwerte, die innerhalb der Werte des Ausdrucks am häufigsten vorkommen. `NULL` für skalare Typen. |
| `most_common_elem_freqs` | `float4[]` | Liste der Häufigkeiten der häufigsten Elementwerte, also der Anteil der Zeilen, die mindestens ein Vorkommen des jeweiligen Werts enthalten. Zwei oder drei zusätzliche Werte folgen den Häufigkeiten pro Element; dies sind Minimum und Maximum der vorhergehenden Häufigkeiten pro Element und optional die Häufigkeit von `NULL`-Elementen. `NULL`, wenn `most_common_elems` `NULL` ist. |
| `elem_count_histogram` | `float4[]` | Histogramm der Anzahlen unterschiedlicher Nicht-`NULL`-Elementwerte innerhalb der Werte des Ausdrucks, gefolgt von der durchschnittlichen Anzahl unterschiedlicher Nicht-`NULL`-Elemente. `NULL` für skalare Typen. |

Die maximale Anzahl von Einträgen in den Array-Feldern kann spaltenweise mit dem Kommando `ALTER TABLE SET STATISTICS` oder global durch Setzen des Laufzeitparameters `default_statistics_target` gesteuert werden.

## 53.32. pg_tables

Die Sicht `pg_tables` bietet Zugriff auf nützliche Informationen über jede Tabelle in der Datenbank.

**Tabelle 53.32. Spalten von `pg_tables`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Tabelle enthält |
| `tablename` | `name` (referenziert `pg_class.relname`) | Name der Tabelle |
| `tableowner` | `name` (referenziert `pg_authid.rolname`) | Name des Besitzers der Tabelle |
| `tablespace` | `name` (referenziert `pg_tablespace.spcname`) | Name des Tablespace, der die Tabelle enthält, oder `NULL`, wenn der Standard der Datenbank verwendet wird |
| `hasindexes` | `bool` (referenziert `pg_class.relhasindex`) | Wahr, wenn die Tabelle Indizes hat oder kürzlich hatte |
| `hasrules` | `bool` (referenziert `pg_class.relhasrules`) | Wahr, wenn die Tabelle Regeln hat oder einmal hatte |
| `hastriggers` | `bool` (referenziert `pg_class.relhastriggers`) | Wahr, wenn die Tabelle Trigger hat oder einmal hatte |
| `rowsecurity` | `bool` (referenziert `pg_class.relrowsecurity`) | Wahr, wenn Row Security für die Tabelle aktiviert ist |

## 53.33. pg_timezone_abbrevs

Die Sicht `pg_timezone_abbrevs` stellt eine Liste der Zeitzonenabkürzungen bereit, die derzeit von den Eingaberoutinen für Datum und Uhrzeit erkannt werden. Der Inhalt dieser Sicht ändert sich, wenn die Laufzeitparameter `TimeZone` oder `timezone_abbreviations` geändert werden.

**Tabelle 53.33. Spalten von `pg_timezone_abbrevs`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `abbrev` | `text` | Zeitzonenabkürzung |
| `utc_offset` | `interval` | Offset von UTC; positive Werte bedeuten östlich von Greenwich |
| `is_dst` | `bool` | Wahr, wenn dies eine Sommerzeitabkürzung ist |

Während die meisten Zeitzonenabkürzungen feste Offsets von UTC darstellen, gibt es einige, deren Wert historisch variiert hat; weitere Informationen finden Sie in Abschnitt B.4. In solchen Fällen zeigt diese Sicht ihre aktuelle Bedeutung.

## 53.34. pg_timezone_names

Die Sicht `pg_timezone_names` stellt eine Liste der Zeitzonennamen bereit, die von `SET TIMEZONE` erkannt werden, zusammen mit den zugehörigen Abkürzungen, UTC-Offsets und dem Sommerzeitstatus. Technisch gesehen verwendet PostgreSQL nicht UTC, weil Schaltsekunden nicht behandelt werden. Anders als die in `pg_timezone_abbrevs` gezeigten Abkürzungen implizieren viele dieser Namen eine Menge von Regeln für Sommerzeit-Übergangsdaten. Daher ändern sich die zugehörigen Informationen über lokale DST-Grenzen hinweg. Die angezeigten Informationen werden anhand des aktuellen Werts von `CURRENT_TIMESTAMP` berechnet.

**Tabelle 53.34. Spalten von `pg_timezone_names`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `name` | `text` | Zeitzonenname |
| `abbrev` | `text` | Zeitzonenabkürzung |
| `utc_offset` | `interval` | Offset von UTC; positive Werte bedeuten östlich von Greenwich |
| `is_dst` | `bool` | Wahr, wenn derzeit Sommerzeit gilt |

## 53.35. pg_user

Die Sicht `pg_user` bietet Zugriff auf Informationen über Datenbankbenutzer. Sie ist einfach eine öffentlich lesbare Sicht auf `pg_shadow`, bei der das Passwortfeld ausgeblendet wird.

**Tabelle 53.35. Spalten von `pg_user`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `usename` | `name` | Benutzername |
| `usesysid` | `oid` | ID dieses Benutzers |
| `usecreatedb` | `bool` | Der Benutzer kann Datenbanken erstellen |
| `usesuper` | `bool` | Der Benutzer ist ein Superuser |
| `userepl` | `bool` | Der Benutzer kann Streaming-Replikation initiieren und das System in den Backup-Modus versetzen sowie daraus zurückholen |
| `usebypassrls` | `bool` | Der Benutzer umgeht jede Row-Level-Security-Policy; weitere Informationen finden Sie in [Abschnitt 5.9](05_Datendefinition.md#59-rowsecuritypolicies). |
| `passwd` | `text` | Nicht das Passwort; liest sich immer als `********` |
| `valuntil` | `timestamptz` | Ablaufzeit des Passworts, nur für Passwortauthentifizierung verwendet |
| `useconfig` | `text[]` | Sitzungs-Standardwerte für Laufzeit-Konfigurationsvariablen |

## 53.36. pg_user_mappings

Die Sicht `pg_user_mappings` bietet Zugriff auf Informationen über Benutzerzuordnungen. Sie ist im Wesentlichen eine öffentlich lesbare Sicht auf `pg_user_mapping`, die das Optionsfeld auslässt, wenn der Benutzer keine Rechte hat, es zu verwenden.

**Tabelle 53.36. Spalten von `pg_user_mappings`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `umid` | `oid` (referenziert `pg_user_mapping.oid`) | OID der Benutzerzuordnung |
| `srvid` | `oid` (referenziert `pg_foreign_server.oid`) | OID des Foreign Servers, der diese Zuordnung enthält |
| `srvname` | `name` (referenziert `pg_foreign_server.srvname`) | Name des Foreign Servers |
| `umuser` | `oid` (referenziert `pg_authid.oid`) | OID der lokalen Rolle, die zugeordnet wird, oder `0`, wenn die Benutzerzuordnung öffentlich ist |
| `usename` | `name` | Name des lokalen Benutzers, der zugeordnet wird |
| `umoptions` | `text[]` | Optionen, die spezifisch für die Benutzerzuordnung sind, als Zeichenketten der Form `keyword=value` |

Um Passwortinformationen zu schützen, die als Option einer Benutzerzuordnung gespeichert sind, liest sich die Spalte `umoptions` als `NULL`, sofern nicht eine der folgenden Bedingungen zutrifft:

- Der aktuelle Benutzer ist der zugeordnete Benutzer und besitzt den Server oder hat das Recht `USAGE` darauf.
- Der aktuelle Benutzer ist der Besitzer des Servers, und die Zuordnung gilt für `PUBLIC`.
- Der aktuelle Benutzer ist ein Superuser.

## 53.37. pg_views

Die Sicht `pg_views` bietet Zugriff auf nützliche Informationen über jede Sicht in der Datenbank.

**Tabelle 53.37. Spalten von `pg_views`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `schemaname` | `name` (referenziert `pg_namespace.nspname`) | Name des Schemas, das die Sicht enthält |
| `viewname` | `name` (referenziert `pg_class.relname`) | Name der Sicht |
| `viewowner` | `name` (referenziert `pg_authid.rolname`) | Name des Besitzers der Sicht |
| `definition` | `text` | Definition der Sicht, also eine rekonstruierte `SELECT`-Abfrage |

## 53.38. pg_wait_events

Die Sicht `pg_wait_events` stellt Beschreibungen der Warteereignisse bereit.

**Tabelle 53.38. Spalten von `pg_wait_events`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `type` | `text` | Typ des Warteereignisses |
| `name` | `text` | Name des Warteereignisses |
| `description` | `text` | Beschreibung des Warteereignisses |
