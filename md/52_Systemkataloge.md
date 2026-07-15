# 52. Systemkataloge

Die Systemkataloge sind der Ort, an dem ein relationales Datenbankmanagementsystem Schema-Metadaten speichert, etwa Informationen über Tabellen und Spalten, sowie interne Verwaltungsinformationen. Die Systemkataloge von PostgreSQL sind gewöhnliche Tabellen. Sie können die Tabellen löschen und neu erstellen, Spalten hinzufügen, Werte einfügen und aktualisieren und Ihr System dadurch massiv beschädigen. Normalerweise sollte man die Systemkataloge nicht von Hand ändern; dafür gibt es üblicherweise SQL-Befehle. Beispielsweise fügt `CREATE DATABASE` eine Zeile in den Katalog `pg_database` ein und erzeugt die Datenbank tatsächlich auf der Platte. Es gibt einige Ausnahmen für besonders esoterische Operationen, aber viele davon wurden im Laufe der Zeit als SQL-Befehle verfügbar gemacht, sodass die Notwendigkeit direkter Manipulation der Systemkataloge immer weiter abnimmt.

## 52.1. Überblick

Tabelle 52.1 listet die Systemkataloge auf. Eine ausführlichere Dokumentation jedes Katalogs folgt weiter unten.

Die meisten Systemkataloge werden beim Erzeugen einer Datenbank aus der Template-Datenbank kopiert und sind danach datenbankspezifisch. Einige wenige Kataloge werden physisch von allen Datenbanken in einem Cluster gemeinsam genutzt; darauf wird in den Beschreibungen der einzelnen Kataloge hingewiesen.

**Tabelle 52.1. Systemkataloge**

| Katalogname | Zweck |
| --- | --- |
| `pg_aggregate` | Aggregatfunktionen |
| `pg_am` | Relation-Access-Methods |
| `pg_amop` | Access-Method-Operatoren |
| `pg_amproc` | Support-Funktionen von Access Methods |
| `pg_attrdef` | Spalten-Default-Werte |
| `pg_attribute` | Tabellenspalten ("Attribute") |
| `pg_authid` | Autorisierungsbezeichner (Rollen) |
| `pg_auth_members` | Mitgliedschaftsbeziehungen zwischen Autorisierungsbezeichnern |
| `pg_cast` | Casts (Datentypumwandlungen) |
| `pg_class` | Tabellen, Indizes, Sequenzen, Views ("Relationen") |
| `pg_collation` | Collations (Locale-Informationen) |
| `pg_constraint` | Check-Constraints, Unique-Constraints, Primary-Key-Constraints, Foreign-Key-Constraints |
| `pg_conversion` | Informationen zur Kodierungsumwandlung |
| `pg_database` | Datenbanken innerhalb dieses Datenbankclusters |
| `pg_db_role_setting` | Einstellungen pro Rolle und pro Datenbank |
| `pg_default_acl` | Default-Privilegien für Objekttypen |
| `pg_depend` | Abhängigkeiten zwischen Datenbankobjekten |
| `pg_description` | Beschreibungen oder Kommentare zu Datenbankobjekten |
| `pg_enum` | Definitionen von Enum-Labels und -Werten |
| `pg_event_trigger` | Event-Trigger |
| `pg_extension` | Installierte Erweiterungen |
| `pg_foreign_data_wrapper` | Foreign-Data-Wrapper-Definitionen |
| `pg_foreign_server` | Foreign-Server-Definitionen |
| `pg_foreign_table` | Zusätzliche Informationen zu Foreign Tables |
| `pg_index` | Zusätzliche Indexinformationen |
| `pg_inherits` | Tabellenvererbungshierarchie |
| `pg_init_privs` | Anfangsprivilegien von Objekten |
| `pg_language` | Sprachen zum Schreiben von Funktionen |
| `pg_largeobject` | Datenseiten für Large Objects |
| `pg_largeobject_metadata` | Metadaten für Large Objects |
| `pg_namespace` | Schemas |
| `pg_opclass` | Access-Method-Operatorklassen |
| `pg_operator` | Operatoren |
| `pg_opfamily` | Access-Method-Operatorfamilien |
| `pg_parameter_acl` | Konfigurationsparameter, für die Privilegien vergeben wurden |
| `pg_partitioned_table` | Informationen über den Partitionsschlüssel von Tabellen |
| `pg_policy` | Row-Security-Policies |
| `pg_proc` | Funktionen und Prozeduren |
| `pg_publication` | Publications für logische Replikation |
| `pg_publication_namespace` | Zuordnung von Schema zu Publication |
| `pg_publication_rel` | Zuordnung von Relation zu Publication |
| `pg_range` | Informationen über Range-Typen |
| `pg_replication_origin` | Registrierte Replication Origins |
| `pg_rewrite` | Query-Rewrite-Regeln |
| `pg_seclabel` | Security Labels auf Datenbankobjekten |
| `pg_sequence` | Informationen über Sequenzen |
| `pg_shdepend` | Abhängigkeiten von gemeinsam genutzten Objekten |
| `pg_shdescription` | Kommentare zu gemeinsam genutzten Objekten |
| `pg_shseclabel` | Security Labels auf gemeinsam genutzten Datenbankobjekten |
| `pg_statistic` | Planner-Statistiken |
| `pg_statistic_ext` | Erweiterte Planner-Statistiken (Definition) |
| `pg_statistic_ext_data` | Erweiterte Planner-Statistiken (berechnete Statistiken) |
| `pg_subscription` | Subscriptions für logische Replikation |
| `pg_subscription_rel` | Relationsstatus für Subscriptions |
| `pg_tablespace` | Tablespaces innerhalb dieses Datenbankclusters |
| `pg_transform` | Transforms (Umwandlungen von Datentypen in prozedurale Sprachen) |
| `pg_trigger` | Trigger |
| `pg_ts_config` | Textsuchkonfigurationen |
| `pg_ts_config_map` | Token-Zuordnungen von Textsuchkonfigurationen |
| `pg_ts_dict` | Textsuchwörterbücher |
| `pg_ts_parser` | Textsuchparser |
| `pg_ts_template` | Textsuchvorlagen |
| `pg_type` | Datentypen |
| `pg_user_mapping` | Zuordnungen von Benutzern zu Foreign Servers |

## 52.2. pg_aggregate

Der Katalog `pg_aggregate` speichert Informationen über Aggregatfunktionen. Eine Aggregatfunktion ist eine Funktion, die auf einer Menge von Werten arbeitet (typischerweise einer Spalte aus jeder Zeile, die eine Abfragebedingung erfüllt) und einen einzelnen Wert zurückgibt, der aus all diesen Werten berechnet wurde. Typische Aggregatfunktionen sind `sum`, `count` und `max`. Jeder Eintrag in `pg_aggregate` ist eine Erweiterung eines Eintrags in `pg_proc`. Der `pg_proc`-Eintrag enthält den Namen des Aggregats, Eingabe- und Ausgabedatentypen sowie weitere Informationen, die denen gewöhnlicher Funktionen ähneln.

**Tabelle 52.2. Spalten von `pg_aggregate`**

| Spalte Typ | Beschreibung |
| --- | --- |
| `aggfnoid` `regproc` (referenziert `pg_proc.oid`) | `pg_proc`-OID der Aggregatfunktion |
| `aggkind` `char` | Aggregatart: `n` für "normale" Aggregate, `o` für "Ordered-Set"-Aggregate oder `h` für "Hypothetical-Set"-Aggregate |
| `aggnumdirectargs` `int2` | Anzahl direkter (nicht aggregierter) Argumente eines Ordered-Set- oder Hypothetical-Set-Aggregats, wobei ein variadisches Array als ein Argument zählt. Wenn der Wert gleich `pronargs` ist, muss das Aggregat variadisch sein, und das variadische Array beschreibt sowohl die aggregierten Argumente als auch die abschließenden direkten Argumente. Für normale Aggregate immer null. |
| `aggtransfn` `regproc` (referenziert `pg_proc.oid`) | Transitionsfunktion |
| `aggfinalfn` `regproc` (referenziert `pg_proc.oid`) | Finalfunktion (null, wenn keine vorhanden ist) |
| `aggcombinefn` `regproc` (referenziert `pg_proc.oid`) | Combine-Funktion (null, wenn keine vorhanden ist) |
| `aggserialfn` `regproc` (referenziert `pg_proc.oid`) | Serialisierungsfunktion (null, wenn keine vorhanden ist) |
| `aggdeserialfn` `regproc` (referenziert `pg_proc.oid`) | Deserialisierungsfunktion (null, wenn keine vorhanden ist) |
| `aggmtransfn` `regproc` (referenziert `pg_proc.oid`) | Vorwärts-Transitionsfunktion für den Moving-Aggregate-Modus (null, wenn keine vorhanden ist) |
| `aggminvtransfn` `regproc` (referenziert `pg_proc.oid`) | Inverse Transitionsfunktion für den Moving-Aggregate-Modus (null, wenn keine vorhanden ist) |
| `aggmfinalfn` `regproc` (referenziert `pg_proc.oid`) | Finalfunktion für den Moving-Aggregate-Modus (null, wenn keine vorhanden ist) |
| `aggfinalextra` `bool` | Wahr, wenn zusätzliche Dummy-Argumente an `aggfinalfn` übergeben werden sollen |
| `aggmfinalextra` `bool` | Wahr, wenn zusätzliche Dummy-Argumente an `aggmfinalfn` übergeben werden sollen |
| `aggfinalmodify` `char` | Gibt an, ob `aggfinalfn` den Wert des Transitionszustands verändert: `r`, wenn sie nur liest, `s`, wenn `aggtransfn` nach `aggfinalfn` nicht angewendet werden kann, oder `w`, wenn sie auf den Wert schreibt |
| `aggmfinalmodify` `char` | Wie `aggfinalmodify`, aber für `aggmfinalfn` |
| `aggsortop` `oid` (referenziert `pg_operator.oid`) | Zugehöriger Sortieroperator (null, wenn keiner vorhanden ist) |
| `aggtranstype` `oid` (referenziert `pg_type.oid`) | Datentyp der internen Transitionsdaten (Zustandsdaten) der Aggregatfunktion |
| `aggtransspace` `int4` | Ungefähre durchschnittliche Größe (in Bytes) der Transitionszustandsdaten oder null, um eine Default-Schätzung zu verwenden |
| `aggmtranstype` `oid` (referenziert `pg_type.oid`) | Datentyp der internen Transitionsdaten (Zustandsdaten) der Aggregatfunktion für den Moving-Aggregate-Modus (null, wenn keiner vorhanden ist) |
| `aggmtransspace` `int4` | Ungefähre durchschnittliche Größe (in Bytes) der Transitionszustandsdaten für den Moving-Aggregate-Modus oder null, um eine Default-Schätzung zu verwenden |
| `agginitval` `text` | Anfangswert des Transitionszustands. Dies ist ein Textfeld, das den Anfangswert in seiner externen Zeichenkettendarstellung enthält. Wenn dieses Feld null ist, beginnt der Transitionszustandswert als null. |
| `aggminitval` `text` | Anfangswert des Transitionszustands für den Moving-Aggregate-Modus. Dies ist ein Textfeld, das den Anfangswert in seiner externen Zeichenkettendarstellung enthält. Wenn dieses Feld null ist, beginnt der Transitionszustandswert als null. |

Neue Aggregatfunktionen werden mit dem Befehl `CREATE AGGREGATE` registriert. Weitere Informationen zum Schreiben von Aggregatfunktionen und zur Bedeutung der Transitionsfunktionen usw. finden Sie in [Abschnitt 36.12](36_SQL_erweitern.md#3612-benutzerdefinierte-aggregate).

## 52.3. pg_am

Der Katalog `pg_am` speichert Informationen über Relation-Access-Methods. Für jede vom System unterstützte Access Method gibt es eine Zeile. Derzeit haben nur Tabellen und Indizes Access Methods. Die Anforderungen an Tabellen- und Index-Access-Methods werden ausführlich in [Kapitel 62](62_Schnittstellendefinition_für_Table_Access_Methods.md) beziehungsweise [Kapitel 63](63_Schnittstellendefinition_für_Index_Access_Methods.md) behandelt.

**Tabelle 52.3. Spalten von `pg_am`**

| Spalte Typ | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `amname` `name` | Name der Access Method |
| `amhandler` `regproc` (referenziert `pg_proc.oid`) | OID einer Handlerfunktion, die dafür verantwortlich ist, Informationen über die Access Method bereitzustellen |
| `amtype` `char` | `t` = Tabelle (einschließlich materialisierter Views), `i` = Index |

> Vor PostgreSQL 9.6 enthielt `pg_am` viele zusätzliche Spalten, die Eigenschaften von Index-Access-Methods darstellten. Diese Daten sind jetzt nur noch direkt auf C-Code-Ebene sichtbar. Allerdings wurden `pg_index_column_has_property()` und verwandte Funktionen hinzugefügt, damit SQL-Abfragen Eigenschaften von Index-Access-Methods untersuchen können; siehe Tabelle 9.76.

## 52.4. pg_amop

Der Katalog `pg_amop` speichert Informationen über Operatoren, die Access-Method-Operatorfamilien zugeordnet sind. Für jeden Operator, der Mitglied einer Operatorfamilie ist, gibt es eine Zeile. Ein Familienmitglied kann entweder ein Suchoperator oder ein Ordnungsoperator sein. Ein Operator kann in mehr als einer Familie vorkommen, darf innerhalb einer Familie aber nicht in mehr als einer Suchposition und nicht in mehr als einer Ordnungsposition vorkommen. Es ist zulässig, wenn auch unwahrscheinlich, dass ein Operator sowohl für Such- als auch für Ordnungszwecke verwendet wird.

**Tabelle 52.4. Spalten von `pg_amop`**

| Spalte Typ | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `amopfamily` `oid` (referenziert `pg_opfamily.oid`) | Die Operatorfamilie, zu der dieser Eintrag gehört |
| `amoplefttype` `oid` (referenziert `pg_type.oid`) | Linker Eingabedatentyp des Operators |
| `amoprighttype` `oid` (referenziert `pg_type.oid`) | Rechter Eingabedatentyp des Operators |
| `amopstrategy` `int2` | Operator-Strategienummer |
| `amoppurpose` `char` | Zweck des Operators, entweder `s` für Suche oder `o` für Ordnung |
| `amopopr` `oid` (referenziert `pg_operator.oid`) | OID des Operators |
| `amopmethod` `oid` (referenziert `pg_am.oid`) | Index-Access-Method, zu der die Operatorfamilie gehört |
| `amopsortfamily` `oid` (referenziert `pg_opfamily.oid`) | Die B-Tree-Operatorfamilie, nach der dieser Eintrag sortiert, falls es sich um einen Ordnungsoperator handelt; null bei einem Suchoperator |

Ein Eintrag für einen "Suchoperator" zeigt an, dass ein Index dieser Operatorfamilie durchsucht werden kann, um alle Zeilen zu finden, die `WHERE indexed_column operator constant` erfüllen. Offensichtlich muss ein solcher Operator `boolean` zurückgeben, und sein linker Eingabetyp muss zum Spaltendatentyp des Indexes passen.

Ein Eintrag für einen "Ordnungsoperator" zeigt an, dass ein Index dieser Operatorfamilie gescannt werden kann, um Zeilen in der durch `ORDER BY indexed_column operator constant` dargestellten Reihenfolge zurückzugeben. Ein solcher Operator könnte jeden sortierbaren Datentyp zurückgeben, wobei auch hier sein linker Eingabetyp zum Spaltendatentyp des Indexes passen muss. Die genaue Semantik von `ORDER BY` wird durch die Spalte `amopsortfamily` angegeben, die eine B-Tree-Operatorfamilie für den Ergebnistyp des Operators referenzieren muss.

> Derzeit wird angenommen, dass die Sortierreihenfolge für einen Ordnungsoperator der Default der referenzierten Operatorfamilie ist, also `ASC NULLS LAST`. Diese Einschränkung könnte eines Tages gelockert werden, indem zusätzliche Spalten hinzugefügt werden, um Sortieroptionen explizit anzugeben.

`amopmethod` eines Eintrags muss mit `opfmethod` der enthaltenden Operatorfamilie übereinstimmen. Dass `amopmethod` hier enthalten ist, ist eine absichtliche Denormalisierung der Katalogstruktur aus Performance-Gründen. Außerdem müssen `amoplefttype` und `amoprighttype` mit den Feldern `oprleft` und `oprright` des referenzierten `pg_operator`-Eintrags übereinstimmen.

## 52.5. pg_amproc

Der Katalog `pg_amproc` speichert Informationen über Support-Funktionen, die Access-Method-Operatorfamilien zugeordnet sind. Für jede Support-Funktion, die zu einer Operatorfamilie gehört, gibt es eine Zeile.

**Tabelle 52.5. Spalten von `pg_amproc`**

| Spalte Typ | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `amprocfamily` `oid` (referenziert `pg_opfamily.oid`) | Die Operatorfamilie, zu der dieser Eintrag gehört |
| `amproclefttype` `oid` (referenziert `pg_type.oid`) | Linker Eingabedatentyp des zugehörigen Operators |
| `amprocrighttype` `oid` (referenziert `pg_type.oid`) | Rechter Eingabedatentyp des zugehörigen Operators |
| `amprocnum` `int2` | Nummer der Support-Funktion |
| `amproc` `regproc` (referenziert `pg_proc.oid`) | OID der Funktion |

Die übliche Interpretation der Felder `amproclefttype` und `amprocrighttype` ist, dass sie den linken und rechten Eingabetyp des Operators oder der Operatoren identifizieren, die eine bestimmte Support-Funktion unterstützt. Bei manchen Access Methods stimmen diese mit den Eingabedatentypen der Support-Funktion selbst überein, bei anderen nicht. Es gibt den Begriff der "Default"-Support-Funktionen für einen Index; das sind diejenigen, bei denen `amproclefttype` und `amprocrighttype` beide gleich `opcintype` der Index-Operatorklasse sind.

## 52.6. pg_attrdef

Der Katalog `pg_attrdef` speichert Spalten-Default-Ausdrücke und Generierungsausdrücke. Die wichtigsten Informationen über Spalten werden in `pg_attribute` gespeichert. Nur Spalten, für die ein Default-Ausdruck oder Generierungsausdruck explizit gesetzt wurde, haben hier einen Eintrag.

**Tabelle 52.6. Spalten von `pg_attrdef`**

| Spalte Typ | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `adrelid` `oid` (referenziert `pg_class.oid`) | Die Tabelle, zu der diese Spalte gehört |
| `adnum` `int2` (referenziert `pg_attribute.attnum`) | Die Nummer der Spalte |
| `adbin` `pg_node_tree` | Der Spalten-Default- oder Generierungsausdruck in `nodeToString()`-Darstellung. Verwenden Sie `pg_get_expr(adbin, adrelid)`, um ihn in einen SQL-Ausdruck umzuwandeln. |

## 52.7. pg_attribute

Der Katalog `pg_attribute` speichert Informationen über Tabellenspalten. Für jede Spalte jeder Tabelle in der Datenbank gibt es genau eine `pg_attribute`-Zeile. Es gibt außerdem Attributeinträge für Indizes und tatsächlich für alle Objekte, die Einträge in `pg_class` haben.

Der Begriff Attribut ist gleichbedeutend mit Spalte und wird aus historischen Gründen verwendet.

**Tabelle 52.7. Spalten von `pg_attribute`**

| Spalte Typ | Beschreibung |
| --- | --- |
| `attrelid` `oid` (referenziert `pg_class.oid`) | Die Tabelle, zu der diese Spalte gehört |
| `attname` `name` | Der Spaltenname |
| `atttypid` `oid` (referenziert `pg_type.oid`) | Der Datentyp dieser Spalte (null bei einer gelöschten Spalte) |
| `attlen` `int2` | Eine Kopie von `pg_type.typlen` des Typs dieser Spalte |
| `attnum` `int2` | Die Nummer der Spalte. Gewöhnliche Spalten werden ab 1 aufwärts nummeriert. Systemspalten wie `ctid` haben (beliebige) negative Nummern. |
| `atttypmod` `int4` | `atttypmod` speichert typspezifische Daten, die beim Erstellen der Tabelle angegeben wurden, zum Beispiel die maximale Länge einer `varchar`-Spalte. Der Wert wird an typspezifische Eingabefunktionen und Längen-Coercion-Funktionen übergeben. Für Typen, die `atttypmod` nicht benötigen, ist der Wert im Allgemeinen `-1`. |
| `attndims` `int2` | Anzahl der Dimensionen, wenn die Spalte ein Array-Typ ist; andernfalls 0. Derzeit wird die Anzahl der Dimensionen eines Arrays nicht erzwungen, sodass jeder Wert ungleich null effektiv bedeutet: "es ist ein Array". |
| `attbyval` `bool` | Eine Kopie von `pg_type.typbyval` des Typs dieser Spalte |
| `attalign` `char` | Eine Kopie von `pg_type.typalign` des Typs dieser Spalte |
| `attstorage` `char` | Normalerweise eine Kopie von `pg_type.typstorage` des Typs dieser Spalte. Für TOAST-fähige Datentypen kann dies nach dem Erstellen der Spalte geändert werden, um die Speicherstrategie zu steuern. |
| `attcompression` `char` | Die aktuelle Kompressionsmethode der Spalte. Typischerweise ist dies `'\0'`, um die aktuelle Default-Einstellung zu verwenden (siehe `default_toast_compression`). Andernfalls wählt `p` die `pglz`-Kompression, während `l` LZ4-Kompression wählt. Dieses Feld wird jedoch ignoriert, wenn `attstorage` keine Kompression erlaubt. |
| `attnotnull` `bool` | Diese Spalte hat einen (möglicherweise ungültigen) Not-Null-Constraint. |
| `atthasdef` `bool` | Diese Spalte hat einen Default-Ausdruck oder Generierungsausdruck; in diesem Fall gibt es einen entsprechenden Eintrag im Katalog `pg_attrdef`, der den Ausdruck tatsächlich definiert. Prüfen Sie `attgenerated`, um festzustellen, ob es sich um einen Default- oder Generierungsausdruck handelt. |
| `atthasmissing` `bool` | Diese Spalte hat einen Wert, der verwendet wird, wenn die Spalte in der Zeile vollständig fehlt, wie es geschieht, wenn eine Spalte mit einem nicht-volatilen `DEFAULT`-Wert hinzugefügt wird, nachdem die Zeile erstellt wurde. Der tatsächlich verwendete Wert wird in der Spalte `attmissingval` gespeichert. |
| `attidentity` `char` | Wenn dies ein Null-Byte (`''`) ist, handelt es sich nicht um eine Identity-Spalte. Andernfalls bedeutet `a` "generated always" und `d` "generated by default". |
| `attgenerated` `char` | Wenn dies ein Null-Byte (`''`) ist, handelt es sich nicht um eine generierte Spalte. Andernfalls bedeutet `s` gespeichert und `v` virtuell. Eine gespeicherte generierte Spalte wird physisch wie eine normale Spalte gespeichert. Eine virtuelle generierte Spalte wird physisch als Nullwert gespeichert; der tatsächliche Wert wird zur Laufzeit berechnet. |
| `attisdropped` `bool` | Diese Spalte wurde gelöscht und ist nicht mehr gültig. Eine gelöschte Spalte ist physisch noch in der Tabelle vorhanden, wird aber vom Parser ignoriert und kann daher nicht über SQL angesprochen werden. |
| `attislocal` `bool` | Diese Spalte ist lokal in der Relation definiert. Beachten Sie, dass eine Spalte gleichzeitig lokal definiert und geerbt sein kann. |
| `attinhcount` `int2` | Die Anzahl direkter Vorfahren, die diese Spalte hat. Eine Spalte mit einer Anzahl von Vorfahren ungleich null kann weder gelöscht noch umbenannt werden. |
| `attcollation` `oid` (referenziert `pg_collation.oid`) | Die definierte Collation der Spalte oder null, wenn die Spalte keinen collatable Datentyp hat |
| `attstattarget` `int2` | `attstattarget` steuert den Detailgrad der Statistiken, die `ANALYZE` für diese Spalte sammelt. Ein Wert von null bedeutet, dass keine Statistiken gesammelt werden sollen. Ein Nullwert bedeutet, dass das systemweite Default-Statistikziel verwendet werden soll. Die genaue Bedeutung positiver Werte hängt vom Datentyp ab. Für skalare Datentypen ist `attstattarget` sowohl die Zielanzahl der zu sammelnden "most common values" als auch die Zielanzahl der zu erzeugenden Histogramm-Bins. |
| `attacl` `aclitem[]` | Zugriffsprivilegien auf Spaltenebene, falls solche speziell für diese Spalte vergeben wurden |
| `attoptions` `text[]` | Optionen auf Attributebene als Zeichenketten der Form `keyword=value` |
| `attfdwoptions` `text[]` | Foreign-Data-Wrapper-Optionen auf Attributebene als Zeichenketten der Form `keyword=value` |
| `attmissingval` `anyarray` | Diese Spalte enthält ein Array mit einem Element, das den Wert enthält, der verwendet wird, wenn die Spalte in der Zeile vollständig fehlt, wie es geschieht, wenn eine Spalte mit einem nicht-volatilen `DEFAULT`-Wert hinzugefügt wird, nachdem die Zeile erstellt wurde. Der Wert wird nur verwendet, wenn `atthasmissing` wahr ist. Wenn kein Wert vorhanden ist, ist die Spalte null. |

Im `pg_attribute`-Eintrag einer gelöschten Spalte wird `atttypid` auf null zurückgesetzt, aber `attlen` und die anderen aus `pg_type` kopierten Felder bleiben gültig. Diese Anordnung ist nötig, um mit der Situation umzugehen, dass der Datentyp der gelöschten Spalte später gelöscht wurde und es daher keine `pg_type`-Zeile mehr gibt. `attlen` und die anderen Felder können verwendet werden, um den Inhalt einer Tabellenzeile zu interpretieren.

## 52.8. pg_authid

Der Katalog `pg_authid` enthält Informationen über Datenbank-Autorisierungskennungen (Rollen). Eine Rolle umfasst die Konzepte "Benutzer" und "Gruppen". Ein Benutzer ist im Wesentlichen nur eine Rolle, bei der das Flag `rolcanlogin` gesetzt ist. Jede Rolle (mit oder ohne `rolcanlogin`) kann andere Rollen als Mitglieder haben; siehe `pg_auth_members`.

Da dieser Katalog Passwörter enthält, darf er nicht öffentlich lesbar sein. `pg_roles` ist eine öffentlich lesbare Sicht auf `pg_authid`, in der das Passwortfeld ausgeblendet wird.

[Kapitel 21](21_Datenbankrollen.md) enthält ausführliche Informationen über Benutzer- und Privilegienverwaltung.

Da Benutzeridentitäten clusterweit gelten, wird `pg_authid` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_authid` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.8. Spalten von `pg_authid`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `rolname` `name` | Rollenname |
| `rolsuper` `bool` | Die Rolle hat Superuser-Privilegien |
| `rolinherit` `bool` | Die Rolle erbt automatisch Privilegien von Rollen, deren Mitglied sie ist |
| `rolcreaterole` `bool` | Die Rolle kann weitere Rollen erstellen |
| `rolcreatedb` `bool` | Die Rolle kann Datenbanken erstellen |
| `rolcanlogin` `bool` | Die Rolle kann sich anmelden. Das heißt, diese Rolle kann als anfängliche Sitzungsautorisierungskennung angegeben werden. |
| `rolreplication` `bool` | Die Rolle ist eine Replikationsrolle. Eine Replikationsrolle kann Replikationsverbindungen starten sowie Replikations-Slots erstellen und löschen. |
| `rolbypassrls` `bool` | Die Rolle umgeht jede Row-Level-Security-Richtlinie; siehe [Abschnitt 5.9](05_Datendefinition.md#59-rowsecuritypolicies) für weitere Informationen. |
| `rolconnlimit` `int4` | Für Rollen, die sich anmelden können, legt dies die maximale Anzahl gleichzeitiger Verbindungen fest, die diese Rolle herstellen kann. `-1` bedeutet keine Begrenzung. |
| `rolpassword` `text` | Verschlüsseltes Passwort; null, wenn keines vorhanden ist. Das Format hängt von der verwendeten Verschlüsselungsform ab. |
| `rolvaliduntil` `timestamptz` | Ablaufzeit des Passworts (nur für Passwortauthentifizierung verwendet); null, wenn es keinen Ablaufzeitpunkt gibt |

Bei einem MD5-verschlüsselten Passwort beginnt die Spalte `rolpassword` mit der Zeichenkette `md5`, gefolgt von einem 32 Zeichen langen hexadezimalen MD5-Hash. Der MD5-Hash wird aus dem Passwort des Benutzers gebildet, an das sein Benutzername angehängt wird. Wenn der Benutzer `joe` beispielsweise das Passwort `xyzzy` hat, speichert PostgreSQL den MD5-Hash von `xyzzyjoe`.

> **Warnung**
>
> Die Unterstützung für MD5-verschlüsselte Passwörter ist veraltet und wird in einer zukünftigen PostgreSQL-Version entfernt. Hinweise zur Migration auf einen anderen Passworttyp finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

Wenn das Passwort mit SCRAM-SHA-256 verschlüsselt ist, hat es das folgende Format:

```text
SCRAM-SHA-256$<iteration count>:<salt>$<StoredKey>:<ServerKey>
```

Dabei sind `salt`, `StoredKey` und `ServerKey` Base64-codiert. Dieses Format entspricht dem in [RFC 5803](https://datatracker.ietf.org/doc/html/rfc5803) angegebenen Format.

## 52.9. pg_auth_members

Der Katalog `pg_auth_members` zeigt die Mitgliedschaftsbeziehungen zwischen Rollen. Jede nicht zirkuläre Menge von Beziehungen ist erlaubt.

Da Benutzeridentitäten clusterweit gelten, wird `pg_auth_members` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_auth_members` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.9. Spalten von `pg_auth_members`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `roleid` `oid` (referenziert `pg_authid.oid`) | ID einer Rolle, die ein Mitglied hat |
| `member` `oid` (referenziert `pg_authid.oid`) | ID einer Rolle, die Mitglied von `roleid` ist |
| `grantor` `oid` (referenziert `pg_authid.oid`) | ID der Rolle, die diese Mitgliedschaft gewährt hat |
| `admin_option` `bool` | Wahr, wenn das Mitglied die Mitgliedschaft in `roleid` an andere vergeben kann |
| `inherit_option` `bool` | Wahr, wenn das Mitglied automatisch die Privilegien der gewährten Rolle erbt |
| `set_option` `bool` | Wahr, wenn das Mitglied mit `SET ROLE` zu der gewährten Rolle wechseln kann |

## 52.10. pg_cast

Der Katalog `pg_cast` speichert sowohl eingebaute als auch benutzerdefinierte Pfade für Datentypumwandlungen.

Dabei ist zu beachten, dass `pg_cast` nicht jede Typumwandlung darstellt, die das System ausführen kann, sondern nur solche, die nicht aus einer generischen Regel abgeleitet werden können. Eine Umwandlung zwischen einer Domain und ihrem Basistyp wird zum Beispiel nicht ausdrücklich in `pg_cast` dargestellt. Eine weitere wichtige Ausnahme sind "automatic I/O conversion casts", also Casts, die mit den eigenen Ein-/Ausgabefunktionen eines Datentyps eine Umwandlung nach oder von `text` oder anderen Zeichenkettentypen durchführen; sie werden nicht ausdrücklich in `pg_cast` dargestellt.

**Tabelle 52.10. Spalten von `pg_cast`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `castsource` `oid` (referenziert `pg_type.oid`) | OID des Quelldatentyps |
| `casttarget` `oid` (referenziert `pg_type.oid`) | OID des Zieldatentyps |
| `castfunc` `oid` (referenziert `pg_proc.oid`) | Die OID der Funktion, mit der dieser Cast ausgeführt wird. Null wird gespeichert, wenn die Cast-Methode keine Funktion benötigt. |
| `castcontext` `char` | Gibt an, in welchen Kontexten der Cast aufgerufen werden kann. `e` bedeutet nur als expliziter Cast (mit `CAST`- oder `::`-Syntax). `a` bedeutet implizit bei der Zuweisung an eine Zielspalte sowie explizit. `i` bedeutet implizit in Ausdrücken sowie in den anderen Fällen. |
| `castmethod` `char` | Gibt an, wie der Cast ausgeführt wird. `f` bedeutet, dass die im Feld `castfunc` angegebene Funktion verwendet wird. `i` bedeutet, dass die Ein-/Ausgabefunktionen verwendet werden. `b` bedeutet, dass die Typen binär-koerzibel sind und daher keine Umwandlung erforderlich ist. |

Die in `pg_cast` aufgeführten Cast-Funktionen müssen immer den Cast-Quelldatentyp als ersten Argumenttyp haben und den Cast-Zieldatentyp als Ergebnistyp zurückgeben. Eine Cast-Funktion kann bis zu drei Argumente haben. Das zweite Argument muss, falls vorhanden, vom Typ `integer` sein; es empfängt den Typmodifikator, der dem Zieldatentyp zugeordnet ist, oder `-1`, wenn keiner vorhanden ist. Das dritte Argument muss, falls vorhanden, vom Typ `boolean` sein; es empfängt `true`, wenn der Cast explizit ist, andernfalls `false`.

Es ist zulässig, einen `pg_cast`-Eintrag zu erstellen, bei dem Quell- und Zieltyp identisch sind, wenn die zugehörige Funktion mehr als ein Argument annimmt. Solche Einträge stellen "length coercion functions" dar, die Werte des Typs so erzwingen, dass sie für einen bestimmten Typmodifikatorwert gültig sind.

Wenn ein `pg_cast`-Eintrag unterschiedliche Quell- und Zieltypen sowie eine Funktion mit mehr als einem Argument hat, stellt er eine Umwandlung von einem Typ in einen anderen und die Anwendung einer Längenkoerzierung in einem einzigen Schritt dar. Wenn kein solcher Eintrag verfügbar ist, umfasst die Koerzierung zu einem Typ, der einen Typmodifikator verwendet, zwei Schritte: einen zur Umwandlung zwischen Datentypen und einen zweiten zur Anwendung des Modifikators.

## 52.11. pg_class

Der Katalog `pg_class` beschreibt Tabellen und andere Objekte, die Spalten haben oder auf andere Weise einer Tabelle ähneln. Dazu gehören Indizes (siehe aber auch `pg_index`), Sequenzen (siehe aber auch `pg_sequence`), Sichten, materialisierte Sichten, zusammengesetzte Typen und TOAST-Tabellen; siehe `relkind`. Wenn im Folgenden all diese Objektarten gemeint sind, sprechen wir von "Relationen". Nicht alle Spalten von `pg_class` sind für alle Arten von Relationen sinnvoll.

**Tabelle 52.11. Spalten von `pg_class`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `relname` `name` | Name der Tabelle, des Index, der Sicht usw. |
| `relnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diese Relation enthält |
| `reltype` `oid` (referenziert `pg_type.oid`) | Die OID des Datentyps, der dem Zeilentyp dieser Tabelle entspricht, falls vorhanden; null für Indizes, Sequenzen und TOAST-Tabellen, die keinen `pg_type`-Eintrag haben |
| `reloftype` `oid` (referenziert `pg_type.oid`) | Bei typisierten Tabellen die OID des zugrunde liegenden zusammengesetzten Typs; null für alle anderen Relationen |
| `relowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Relation |
| `relam` `oid` (referenziert `pg_am.oid`) | Die Zugriffsmethode, die für den Zugriff auf diese Tabelle oder diesen Index verwendet wird. Nicht sinnvoll, wenn die Relation eine Sequenz ist oder keine Datei auf dem Datenträger hat, außer bei partitionierten Tabellen: Wenn dort ein Wert gesetzt ist, hat er Vorrang vor `default_table_access_method`, wenn bestimmt wird, welche Zugriffsmethode für Partitionen verwendet wird, die ohne entsprechende Angabe im Erstellungsbefehl erzeugt werden. |
| `relfilenode` `oid` | Name der Datei dieser Relation auf dem Datenträger; null bedeutet, dass dies eine "mapped" Relation ist, deren Dateiname auf dem Datenträger durch Low-Level-Zustand bestimmt wird |
| `reltablespace` `oid` (referenziert `pg_tablespace.oid`) | Der Tablespace, in dem diese Relation gespeichert ist. Wenn null, wird der Standard-Tablespace der Datenbank impliziert. Nicht sinnvoll, wenn die Relation keine Datei auf dem Datenträger hat, außer bei partitionierten Tabellen: Dort ist dies der Tablespace, in dem Partitionen erstellt werden, wenn im Erstellungsbefehl keiner angegeben wird. |
| `relpages` `int4` | Größe der Darstellung dieser Tabelle auf dem Datenträger in Seiten (der Größe `BLCKSZ`). Dies ist nur eine vom Planner verwendete Schätzung. Sie wird durch `VACUUM`, `ANALYZE` und einige DDL-Befehle wie `CREATE INDEX` aktualisiert. |
| `reltuples` `float4` | Anzahl der lebenden Zeilen in der Tabelle. Dies ist nur eine vom Planner verwendete Schätzung. Sie wird durch `VACUUM`, `ANALYZE` und einige DDL-Befehle wie `CREATE INDEX` aktualisiert. Wenn die Tabelle noch nie mit `VACUUM` bearbeitet oder analysiert wurde, enthält `reltuples` den Wert `-1`, der angibt, dass die Zeilenzahl unbekannt ist. |
| `relallvisible` `int4` | Anzahl der Seiten, die in der Visibility Map der Tabelle als all-visible markiert sind. Dies ist nur eine vom Planner verwendete Schätzung. Sie wird durch `VACUUM`, `ANALYZE` und einige DDL-Befehle wie `CREATE INDEX` aktualisiert. |
| `relallfrozen` `int4` | Anzahl der Seiten, die in der Visibility Map der Tabelle als all-frozen markiert sind. Dies ist nur eine Schätzung, die zum Auslösen von Autovacuum-Läufen verwendet wird. Sie kann zusammen mit `relallvisible` auch zur Planung manueller Vacuum-Läufe und zur Abstimmung des Freezing-Verhaltens von Vacuum verwendet werden. Sie wird durch `VACUUM`, `ANALYZE` und einige DDL-Befehle wie `CREATE INDEX` aktualisiert. |
| `reltoastrelid` `oid` (referenziert `pg_class.oid`) | OID der dieser Tabelle zugeordneten TOAST-Tabelle; null, wenn keine vorhanden ist. Die TOAST-Tabelle speichert große Attribute "out of line" in einer sekundären Tabelle. |
| `relhasindex` `bool` | Wahr, wenn dies eine Tabelle ist und sie Indizes hat (oder vor Kurzem hatte) |
| `relisshared` `bool` | Wahr, wenn diese Tabelle von allen Datenbanken im Cluster gemeinsam genutzt wird. Nur bestimmte Systemkataloge (wie `pg_database`) werden gemeinsam genutzt. |
| `relpersistence` `char` | `p` = permanente Tabelle/Sequenz, `u` = unlogged Tabelle/Sequenz, `t` = temporäre Tabelle/Sequenz |
| `relkind` `char` | `r` = gewöhnliche Tabelle, `i` = Index, `S` = Sequenz, `t` = TOAST-Tabelle, `v` = Sicht, `m` = materialisierte Sicht, `c` = zusammengesetzter Typ, `f` = Fremdtabelle, `p` = partitionierte Tabelle, `I` = partitionierter Index |
| `relnatts` `int2` | Anzahl der Benutzerspalten in der Relation (Systemspalten nicht mitgezählt). Es muss entsprechend viele Einträge in `pg_attribute` geben. Siehe auch `pg_attribute.attnum`. |
| `relchecks` `int2` | Anzahl der `CHECK`-Constraints auf der Tabelle; siehe Katalog `pg_constraint` |
| `relhasrules` `bool` | Wahr, wenn die Tabelle Regeln hat (oder früher hatte); siehe Katalog `pg_rewrite` |
| `relhastriggers` `bool` | Wahr, wenn die Tabelle Trigger hat (oder früher hatte); siehe Katalog `pg_trigger` |
| `relhassubclass` `bool` | Wahr, wenn die Tabelle oder der Index Vererbungsunterobjekte oder Partitionen hat (oder früher hatte) |
| `relrowsecurity` `bool` | Wahr, wenn für die Tabelle Row-Level Security aktiviert ist; siehe Katalog `pg_policy` |
| `relforcerowsecurity` `bool` | Wahr, wenn Row-Level Security (sofern aktiviert) auch auf den Tabelleneigentümer angewendet wird; siehe Katalog `pg_policy` |
| `relispopulated` `bool` | Wahr, wenn die Relation befüllt ist (dies ist für alle Relationen außer manchen materialisierten Sichten wahr) |
| `relreplident` `char` | Spalten, die zur Bildung der "Replica Identity" für Zeilen verwendet werden: `d` = Default (Primärschlüssel, falls vorhanden), `n` = nichts, `f` = alle Spalten, `i` = Index mit gesetztem `indisreplident` (entspricht nichts, wenn der verwendete Index gelöscht wurde) |
| `relispartition` `bool` | Wahr, wenn die Tabelle oder der Index eine Partition ist |
| `relrewrite` `oid` (referenziert `pg_class.oid`) | Bei neuen Relationen, die während einer DDL-Operation geschrieben werden, die ein Neuschreiben der Tabelle erfordert, enthält dies die OID der ursprünglichen Relation; andernfalls null. Dieser Zustand ist nur intern sichtbar; dieses Feld sollte bei einer für Benutzer sichtbaren Relation niemals etwas anderes als null enthalten. |
| `relfrozenxid` `xid` | Alle Transaktions-IDs vor dieser wurden in dieser Tabelle durch eine permanente ("frozen") Transaktions-ID ersetzt. Dies wird verwendet, um zu verfolgen, ob die Tabelle mit `VACUUM` bearbeitet werden muss, damit ein Transaction-ID-Wraparound verhindert oder `pg_xact` verkleinert werden kann. Null (`InvalidTransactionId`), wenn die Relation keine Tabelle ist. |
| `relminmxid` `xid` | Alle Multixact-IDs vor dieser wurden in dieser Tabelle durch eine Transaktions-ID ersetzt. Dies wird verwendet, um zu verfolgen, ob die Tabelle mit `VACUUM` bearbeitet werden muss, damit ein Multixact-ID-Wraparound verhindert oder `pg_multixact` verkleinert werden kann. Null (`InvalidMultiXactId`), wenn die Relation keine Tabelle ist. |
| `relacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |
| `reloptions` `text[]` | Zugriffsmethodenspezifische Optionen als Zeichenketten der Form `keyword=value` |
| `relpartbound` `pg_node_tree` | Wenn die Tabelle eine Partition ist (siehe `relispartition`), die interne Darstellung der Partitionsgrenze |

Mehrere der booleschen Flags in `pg_class` werden verzögert gepflegt: Sie sind garantiert wahr, wenn dies der korrekte Zustand ist, werden aber möglicherweise nicht sofort auf falsch zurückgesetzt, wenn die Bedingung nicht mehr zutrifft. `relhasindex` wird zum Beispiel durch `CREATE INDEX` gesetzt, aber niemals durch `DROP INDEX` gelöscht. Stattdessen löscht `VACUUM` `relhasindex`, wenn es feststellt, dass die Tabelle keine Indizes hat. Diese Anordnung vermeidet Race Conditions und verbessert die Nebenläufigkeit.

## 52.12. pg_collation

Der Katalog `pg_collation` beschreibt die verfügbaren Collations, die im Wesentlichen Zuordnungen von einem SQL-Namen zu Locale-Kategorien des Betriebssystems sind. Weitere Informationen finden Sie in [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung).

**Tabelle 52.12. Spalten von `pg_collation`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `collname` `name` | Collation-Name (eindeutig pro Namespace und Encoding) |
| `collnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diese Collation enthält |
| `collowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Collation |
| `collprovider` `char` | Provider der Collation: `d` = Datenbank-Default, `b` = builtin, `c` = libc, `i` = ICU |
| `collisdeterministic` `bool` | Ist die Collation deterministisch? |
| `collencoding` `int4` | Encoding, in dem die Collation anwendbar ist, oder `-1`, wenn sie für jedes Encoding funktioniert |
| `collcollate` `text` | `LC_COLLATE` für dieses Collation-Objekt. Wenn der Provider nicht `libc` ist, ist `collcollate` `NULL` und stattdessen wird `colllocale` verwendet. |
| `collctype` `text` | `LC_CTYPE` für dieses Collation-Objekt. Wenn der Provider nicht `libc` ist, ist `collctype` `NULL` und stattdessen wird `colllocale` verwendet. |
| `colllocale` `text` | Locale-Name des Collation-Providers für dieses Collation-Objekt. Wenn der Provider `libc` ist, ist `colllocale` `NULL`; stattdessen werden `collcollate` und `collctype` verwendet. |
| `collicurules` `text` | ICU-Collation-Regeln für dieses Collation-Objekt |
| `collversion` `text` | Provider-spezifische Version der Collation. Sie wird beim Erstellen der Collation aufgezeichnet und später bei der Verwendung geprüft, um Änderungen an der Collation-Definition zu erkennen, die zu Datenbeschädigung führen könnten. |

Beachten Sie, dass der eindeutige Schlüssel dieses Katalogs `(collname, collencoding, collnamespace)` ist, nicht nur `(collname, collnamespace)`. PostgreSQL ignoriert im Allgemeinen alle Collations, deren `collencoding` weder dem Encoding der aktuellen Datenbank noch `-1` entspricht, und das Erstellen neuer Einträge mit demselben Namen wie ein Eintrag mit `collencoding = -1` ist verboten. Daher reicht es aus, einen qualifizierten SQL-Namen (`schema.name`) zu verwenden, um eine Collation zu identifizieren, auch wenn dies gemäß der Katalogdefinition nicht eindeutig ist. Der Grund für diese Katalogdefinition ist, dass `initdb` ihn bei der Clusterinitialisierung mit Einträgen für alle auf dem System verfügbaren Locales füllt; daher muss er Einträge für alle Encodings aufnehmen können, die jemals im Cluster verwendet werden könnten.

In der Datenbank `template0` kann es nützlich sein, Collations zu erstellen, deren Encoding nicht mit dem Datenbank-Encoding übereinstimmt, da sie zu den Encodings von Datenbanken passen könnten, die später aus `template0` geklont werden. Dies müsste derzeit manuell erfolgen.

## 52.13. pg_constraint

Der Katalog `pg_constraint` speichert Check-, Not-Null-, Primary-Key-, Unique-, Foreign-Key- und Exclusion-Constraints auf Tabellen. (Spalten-Constraints werden nicht besonders behandelt. Jeder Spalten-Constraint entspricht einem Tabellen-Constraint.)

Benutzerdefinierte Constraint-Trigger (erstellt mit `CREATE CONSTRAINT TRIGGER`) erzeugen ebenfalls einen Eintrag in dieser Tabelle.

Check-Constraints auf Domains werden ebenfalls hier gespeichert.

**Tabelle 52.13. Spalten von `pg_constraint`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `conname` `name` | Constraint-Name (nicht notwendigerweise eindeutig!) |
| `connamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diesen Constraint enthält |
| `contype` `char` | `c` = Check-Constraint, `f` = Foreign-Key-Constraint, `n` = Not-Null-Constraint, `p` = Primary-Key-Constraint, `u` = Unique-Constraint, `t` = Constraint-Trigger, `x` = Exclusion-Constraint |
| `condeferrable` `bool` | Ist der Constraint deferrable? |
| `condeferred` `bool` | Ist der Constraint standardmäßig deferred? |
| `conenforced` `bool` | Wird der Constraint erzwungen? |
| `convalidated` `bool` | Wurde der Constraint validiert? |
| `conrelid` `oid` (referenziert `pg_class.oid`) | Die Tabelle, auf der dieser Constraint liegt; null, wenn es kein Tabellen-Constraint ist |
| `contypid` `oid` (referenziert `pg_type.oid`) | Die Domain, auf der dieser Constraint liegt; null, wenn es kein Domain-Constraint ist |
| `conindid` `oid` (referenziert `pg_class.oid`) | Der Index, der diesen Constraint unterstützt, wenn es ein Unique-, Primary-Key-, Foreign-Key- oder Exclusion-Constraint ist; sonst null |
| `conparentid` `oid` (referenziert `pg_constraint.oid`) | Der entsprechende Constraint der übergeordneten partitionierten Tabelle, wenn dies ein Constraint auf einer Partition ist; sonst null |
| `confrelid` `oid` (referenziert `pg_class.oid`) | Bei einem Foreign Key die referenzierte Tabelle; sonst null |
| `confupdtype` `char` | Aktionscode für Foreign-Key-Aktualisierungen: `a` = no action, `r` = restrict, `c` = cascade, `n` = set null, `d` = set default |
| `confdeltype` `char` | Aktionscode für Foreign-Key-Löschungen: `a` = no action, `r` = restrict, `c` = cascade, `n` = set null, `d` = set default |
| `confmatchtype` `char` | Match-Typ des Foreign Keys: `f` = full, `p` = partial, `s` = simple |
| `conislocal` `bool` | Dieser Constraint ist lokal für die Relation definiert. Beachten Sie, dass ein Constraint zugleich lokal definiert und geerbt sein kann. |
| `coninhcount` `int2` | Die Anzahl direkter Vererbungsvorfahren, die dieser Constraint hat. Ein Constraint mit einer Anzahl von Vorfahren ungleich null kann weder gelöscht noch umbenannt werden. |
| `connoinherit` `bool` | Dieser Constraint ist lokal für die Relation definiert. Es ist ein nicht vererbbarer Constraint. |
| `conperiod` `bool` | Dieser Constraint ist mit `WITHOUT OVERLAPS` (für Primary Keys und Unique Constraints) oder `PERIOD` (für Foreign Keys) definiert. |
| `conkey` `int2[]` (referenziert `pg_attribute.attnum`) | Bei einem Tabellen-Constraint (einschließlich Foreign Keys, aber nicht Constraint-Triggern) die Liste der eingeschränkten Spalten |
| `confkey` `int2[]` (referenziert `pg_attribute.attnum`) | Bei einem Foreign Key die Liste der referenzierten Spalten |
| `conpfeqop` `oid[]` (referenziert `pg_operator.oid`) | Bei einem Foreign Key die Liste der Gleichheitsoperatoren für `PK = FK`-Vergleiche |
| `conppeqop` `oid[]` (referenziert `pg_operator.oid`) | Bei einem Foreign Key die Liste der Gleichheitsoperatoren für `PK = PK`-Vergleiche |
| `conffeqop` `oid[]` (referenziert `pg_operator.oid`) | Bei einem Foreign Key die Liste der Gleichheitsoperatoren für `FK = FK`-Vergleiche |
| `confdelsetcols` `int2[]` (referenziert `pg_attribute.attnum`) | Bei einem Foreign Key mit einer Löschaktion `SET NULL` oder `SET DEFAULT` die Spalten, die aktualisiert werden. Wenn null, werden alle referenzierenden Spalten aktualisiert. |
| `conexclop` `oid[]` (referenziert `pg_operator.oid`) | Bei einem Exclusion-Constraint oder einem `WITHOUT OVERLAPS`-Primary-Key-/Unique-Constraint die Liste der Exclusion-Operatoren pro Spalte. |
| `conbin` `pg_node_tree` | Bei einem Check-Constraint eine interne Darstellung des Ausdrucks. (Es wird empfohlen, `pg_get_constraintdef()` zu verwenden, um die Definition eines Check-Constraints zu extrahieren.) |

Bei einem Exclusion-Constraint ist `conkey` nur für Constraint-Elemente nützlich, die einfache Spaltenreferenzen sind. In anderen Fällen erscheint in `conkey` eine null, und der zugehörige Index muss herangezogen werden, um den eingeschränkten Ausdruck zu ermitteln. (`conkey` hat daher denselben Inhalt wie `pg_index.indkey` für den Index.)

> **Hinweis**
>
> `pg_class.relchecks` muss mit der Anzahl der Check-Constraint-Einträge übereinstimmen, die für jede Relation in dieser Tabelle gefunden werden.

## 52.14. pg_conversion

Der Katalog `pg_conversion` beschreibt Funktionen zur Encoding-Konvertierung. Weitere Informationen finden Sie unter `CREATE CONVERSION`.

**Tabelle 52.14. Spalten von `pg_conversion`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `conname` `name` | Conversion-Name (eindeutig innerhalb eines Namensraums) |
| `connamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diese Conversion enthält |
| `conowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Conversion |
| `conforencoding` `int4` | Quell-Encoding-ID (`pg_encoding_to_char()` kann diese Zahl in den Encoding-Namen übersetzen) |
| `contoencoding` `int4` | Ziel-Encoding-ID (`pg_encoding_to_char()` kann diese Zahl in den Encoding-Namen übersetzen) |
| `conproc` `regproc` (referenziert `pg_proc.oid`) | Conversion-Funktion |
| `condefault` `bool` | Wahr, wenn dies die Default-Conversion ist |

## 52.15. pg_database

Der Katalog `pg_database` speichert Informationen über die verfügbaren Datenbanken. Datenbanken werden mit dem Befehl `CREATE DATABASE` erstellt. Details zur Bedeutung einiger Parameter finden Sie in [Kapitel 22](22_Datenbanken_verwalten.md).

Anders als die meisten Systemkataloge wird `pg_database` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_database` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.15. Spalten von `pg_database`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `datname` `name` | Datenbankname |
| `datdba` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Datenbank, normalerweise der Benutzer, der sie erstellt hat |
| `encoding` `int4` | Zeichencodierung dieser Datenbank (`pg_encoding_to_char()` kann diese Zahl in den Encoding-Namen übersetzen) |
| `datlocprovider` `char` | Locale-Provider für diese Datenbank: `b` = builtin, `c` = libc, `i` = ICU |
| `datistemplate` `bool` | Wenn wahr, kann diese Datenbank von jedem Benutzer mit `CREATEDB`-Privilegien geklont werden; wenn falsch, können nur Superuser oder der Eigentümer der Datenbank sie klonen. |
| `datallowconn` `bool` | Wenn falsch, kann sich niemand mit dieser Datenbank verbinden. Dies wird verwendet, um die Datenbank `template0` vor Änderungen zu schützen. |
| `dathasloginevt` `bool` | Gibt an, dass für diese Datenbank Login-Event-Trigger definiert sind. Dieses Flag wird verwendet, um zusätzliche Nachschlagevorgänge in der Tabelle `pg_event_trigger` während jedes Backend-Starts zu vermeiden. Dieses Flag wird intern von PostgreSQL verwendet und sollte nicht manuell geändert oder für Monitoring-Zwecke gelesen werden. |
| `datconnlimit` `int4` | Legt die maximale Anzahl gleichzeitiger Verbindungen fest, die zu dieser Datenbank hergestellt werden können. `-1` bedeutet keine Begrenzung, `-2` zeigt an, dass die Datenbank ungültig ist. |
| `datfrozenxid` `xid` | Alle Transaktions-IDs vor dieser wurden in dieser Datenbank durch eine permanente ("frozen") Transaktions-ID ersetzt. Dies wird verwendet, um zu verfolgen, ob die Datenbank mit `VACUUM` bearbeitet werden muss, damit ein Transaction-ID-Wraparound verhindert oder `pg_xact` verkleinert werden kann. Es ist das Minimum der tabellenbezogenen `pg_class.relfrozenxid`-Werte. |
| `datminmxid` `xid` | Alle Multixact-IDs vor dieser wurden in dieser Datenbank durch eine Transaktions-ID ersetzt. Dies wird verwendet, um zu verfolgen, ob die Datenbank mit `VACUUM` bearbeitet werden muss, damit ein Multixact-ID-Wraparound verhindert oder `pg_multixact` verkleinert werden kann. Es ist das Minimum der tabellenbezogenen `pg_class.relminmxid`-Werte. |
| `dattablespace` `oid` (referenziert `pg_tablespace.oid`) | Der Standard-Tablespace für die Datenbank. Innerhalb dieser Datenbank werden alle Tabellen, für die `pg_class.reltablespace` null ist, in diesem Tablespace gespeichert; insbesondere befinden sich dort alle nicht gemeinsam genutzten Systemkataloge. |
| `datcollate` `text` | `LC_COLLATE` für diese Datenbank |
| `datctype` `text` | `LC_CTYPE` für diese Datenbank |
| `datlocale` `text` | Locale-Name des Collation-Providers für diese Datenbank. Wenn der Provider `libc` ist, ist `datlocale` `NULL`; stattdessen werden `datcollate` und `datctype` verwendet. |
| `daticurules` `text` | ICU-Collation-Regeln für diese Datenbank |
| `datcollversion` `text` | Provider-spezifische Version der Collation. Sie wird beim Erstellen der Datenbank aufgezeichnet und später bei der Verwendung geprüft, um Änderungen an der Collation-Definition zu erkennen, die zu Datenbeschädigung führen könnten. |
| `datacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

## 52.16. pg_db_role_setting

Der Katalog `pg_db_role_setting` zeichnet die Default-Werte auf, die für Laufzeit-Konfigurationsvariablen gesetzt wurden, jeweils für eine Kombination aus Rolle und Datenbank.

Anders als die meisten Systemkataloge wird `pg_db_role_setting` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_db_role_setting` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.16. Spalten von `pg_db_role_setting`**

| Spalte | Beschreibung |
| --- | --- |
| `setdatabase` `oid` (referenziert `pg_database.oid`) | Die OID der Datenbank, für die die Einstellung gilt, oder null, wenn sie nicht datenbankspezifisch ist |
| `setrole` `oid` (referenziert `pg_authid.oid`) | Die OID der Rolle, für die die Einstellung gilt, oder null, wenn sie nicht rollenspezifisch ist |
| `setconfig` `text[]` | Defaults für Laufzeit-Konfigurationsvariablen |

## 52.17. pg_default_acl

Der Katalog `pg_default_acl` speichert anfängliche Privilegien, die neu erstellten Objekten zugewiesen werden sollen.

**Tabelle 52.17. Spalten von `pg_default_acl`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `defaclrole` `oid` (referenziert `pg_authid.oid`) | Die OID der Rolle, die diesem Eintrag zugeordnet ist |
| `defaclnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diesem Eintrag zugeordnet ist, oder null, wenn keiner vorhanden ist |
| `defaclobjtype` `char` | Objekttyp, für den dieser Eintrag gilt: `r` = Relation (Tabelle, Sicht), `S` = Sequenz, `f` = Funktion, `T` = Typ, `n` = Schema, `L` = Large Object |
| `defaclacl` `aclitem[]` | Zugriffsprivilegien, die dieser Objekttyp beim Erstellen haben soll |

Ein `pg_default_acl`-Eintrag zeigt die anfänglichen Privilegien, die einem Objekt des angegebenen Benutzers zugewiesen werden sollen. Derzeit gibt es zwei Arten von Einträgen: "globale" Einträge mit `defaclnamespace = 0` und "pro-Schema"-Einträge, die ein bestimmtes Schema referenzieren. Wenn ein globaler Eintrag vorhanden ist, überschreibt er die normalen fest eingebauten Default-Privilegien für den Objekttyp. Ein pro-Schema-Eintrag stellt, wenn vorhanden, Privilegien dar, die zu den globalen oder fest eingebauten Default-Privilegien hinzugefügt werden.

Beachten Sie, dass ein nullwertiger ACL-Eintrag in einem anderen Katalog so verstanden wird, dass er die fest eingebauten Default-Privilegien für sein Objekt darstellt, nicht das, was gerade in `pg_default_acl` stehen könnte. `pg_default_acl` wird nur beim Erstellen von Objekten herangezogen.

## 52.18. pg_depend

Der Katalog `pg_depend` zeichnet die Abhängigkeitsbeziehungen zwischen Datenbankobjekten auf. Diese Informationen ermöglichen es `DROP`-Befehlen zu ermitteln, welche anderen Objekte bei `DROP CASCADE` ebenfalls gelöscht werden müssen, oder das Löschen im Fall von `DROP RESTRICT` zu verhindern.

Siehe auch `pg_shdepend`, das eine ähnliche Funktion für Abhängigkeiten erfüllt, an denen Objekte beteiligt sind, die über einen Datenbankcluster hinweg gemeinsam genutzt werden.

**Tabelle 52.18. Spalten von `pg_depend`**

| Spalte | Beschreibung |
| --- | --- |
| `classid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem sich das abhängige Objekt befindet |
| `objid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des konkreten abhängigen Objekts |
| `objsubid` `int4` | Bei einer Tabellenspalte ist dies die Spaltennummer (`objid` und `classid` referenzieren die Tabelle selbst). Für alle anderen Objekttypen ist diese Spalte null. |
| `refclassid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem sich das referenzierte Objekt befindet |
| `refobjid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des konkreten referenzierten Objekts |
| `refobjsubid` `int4` | Bei einer Tabellenspalte ist dies die Spaltennummer (`refobjid` und `refclassid` referenzieren die Tabelle selbst). Für alle anderen Objekttypen ist diese Spalte null. |
| `deptype` `char` | Ein Code, der die konkrete Semantik dieser Abhängigkeitsbeziehung definiert; siehe Text |

In allen Fällen zeigt ein `pg_depend`-Eintrag an, dass das referenzierte Objekt nicht gelöscht werden kann, ohne auch das abhängige Objekt zu löschen. Es gibt jedoch mehrere Unterarten, die durch `deptype` gekennzeichnet werden:

`DEPENDENCY_NORMAL` (`n`)

Eine normale Beziehung zwischen getrennt erstellten Objekten. Das abhängige Objekt kann gelöscht werden, ohne das referenzierte Objekt zu beeinflussen. Das referenzierte Objekt kann nur durch Angabe von `CASCADE` gelöscht werden; in diesem Fall wird auch das abhängige Objekt gelöscht. Beispiel: Eine Tabellenspalte hat eine normale Abhängigkeit von ihrem Datentyp.

`DEPENDENCY_AUTO` (`a`)

Das abhängige Objekt kann getrennt vom referenzierten Objekt gelöscht werden und sollte automatisch gelöscht werden (unabhängig vom Modus `RESTRICT` oder `CASCADE`), wenn das referenzierte Objekt gelöscht wird. Beispiel: Ein benannter Constraint auf einer Tabelle wird auto-abhängig von der Tabelle gemacht, sodass er verschwindet, wenn die Tabelle gelöscht wird.

`DEPENDENCY_INTERNAL` (`i`)

Das abhängige Objekt wurde als Teil der Erstellung des referenzierten Objekts erstellt und ist eigentlich nur Teil seiner internen Implementierung. Ein direktes `DROP` des abhängigen Objekts wird vollständig abgelehnt (der Benutzer wird stattdessen angewiesen, ein `DROP` gegen das referenzierte Objekt auszuführen). Ein `DROP` des referenzierten Objekts führt dazu, dass das abhängige Objekt automatisch gelöscht wird, unabhängig davon, ob `CASCADE` angegeben wurde. Wenn das abhängige Objekt aufgrund einer Abhängigkeit von einem anderen entfernten Objekt gelöscht werden muss, wird sein Löschen in ein Löschen des referenzierten Objekts umgewandelt, sodass `NORMAL`- und `AUTO`-Abhängigkeiten des abhängigen Objekts sich weitgehend so verhalten, als wären sie Abhängigkeiten des referenzierten Objekts. Beispiel: Die `ON SELECT`-Regel einer Sicht wird intern von der Sicht abhängig gemacht, sodass sie nicht gelöscht werden kann, solange die Sicht bestehen bleibt. Abhängigkeiten der Regel (etwa Tabellen, auf die sie verweist) verhalten sich so, als wären sie Abhängigkeiten der Sicht.

`DEPENDENCY_PARTITION_PRI` (`P`)

`DEPENDENCY_PARTITION_SEC` (`S`)

Das abhängige Objekt wurde als Teil der Erstellung des referenzierten Objekts erstellt und ist eigentlich nur Teil seiner internen Implementierung; anders als bei `INTERNAL` gibt es jedoch mehr als ein solches referenziertes Objekt. Das abhängige Objekt darf nur gelöscht werden, wenn mindestens eines dieser referenzierten Objekte gelöscht wird; wenn eines davon gelöscht wird, soll das abhängige Objekt gelöscht werden, unabhängig davon, ob `CASCADE` angegeben wurde. Ebenfalls anders als bei `INTERNAL` führt das Löschen eines anderen Objekts, von dem das abhängige Objekt abhängt, nicht zur automatischen Löschung eines partitionsreferenzierten Objekts. Wenn das Löschen also nicht über einen anderen Pfad zu mindestens einem dieser Objekte kaskadiert, wird es verweigert. (In den meisten Fällen teilt das abhängige Objekt alle seine Nicht-Partitions-Abhängigkeiten mit mindestens einem partitionsreferenzierten Objekt, sodass diese Einschränkung keine kaskadierte Löschung blockiert.) Primäre und sekundäre Partitionsabhängigkeiten verhalten sich identisch, außer dass die primäre Abhängigkeit bevorzugt in Fehlermeldungen verwendet wird; daher sollte ein partitionsabhängiges Objekt eine primäre Partitionsabhängigkeit und eine oder mehrere sekundäre Partitionsabhängigkeiten haben. Beachten Sie, dass Partitionsabhängigkeiten zusätzlich zu den Abhängigkeiten erstellt werden, die das Objekt normalerweise hätte, nicht an deren Stelle. Dies vereinfacht `ATTACH PARTITION`-/`DETACH PARTITION`-Operationen: Die Partitionsabhängigkeiten müssen nur hinzugefügt oder entfernt werden. Beispiel: Ein untergeordneter partitionierter Index wird sowohl von der Partitionstabelle, auf der er liegt, als auch vom übergeordneten partitionierten Index partitionsabhängig gemacht, sodass er verschwindet, wenn eines dieser Objekte gelöscht wird, andernfalls aber nicht. Die Abhängigkeit vom übergeordneten Index ist primär, sodass die Fehlermeldung, wenn der Benutzer versucht, den untergeordneten partitionierten Index zu löschen, vorschlägt, stattdessen den übergeordneten Index zu löschen (nicht die Tabelle).

`DEPENDENCY_EXTENSION` (`e`)

Das abhängige Objekt ist ein Mitglied der Extension, die das referenzierte Objekt ist (siehe `pg_extension`). Das abhängige Objekt kann nur über `DROP EXTENSION` auf dem referenzierten Objekt gelöscht werden. Funktional verhält sich dieser Abhängigkeitstyp wie eine `INTERNAL`-Abhängigkeit, wird aber der Klarheit halber und zur Vereinfachung von `pg_dump` getrennt gehalten.

`DEPENDENCY_AUTO_EXTENSION` (`x`)

Das abhängige Objekt ist kein Mitglied der Extension, die das referenzierte Objekt ist (und sollte daher von `pg_dump` nicht ignoriert werden), kann aber ohne die Extension nicht funktionieren und sollte automatisch gelöscht werden, wenn die Extension gelöscht wird. Das abhängige Objekt kann auch eigenständig gelöscht werden. Funktional verhält sich dieser Abhängigkeitstyp wie eine `AUTO`-Abhängigkeit, wird aber der Klarheit halber und zur Vereinfachung von `pg_dump` getrennt gehalten.

Weitere Abhängigkeitsarten könnten in Zukunft benötigt werden.

Beachten Sie, dass zwei Objekte durchaus durch mehr als einen `pg_depend`-Eintrag verbunden sein können. Ein untergeordneter partitionierter Index hätte zum Beispiel sowohl eine Partitionsabhängigkeit von seiner zugehörigen Partitionstabelle als auch eine automatische Abhängigkeit von jeder Spalte dieser Tabelle, die er indiziert. Eine solche Situation drückt die Vereinigung mehrerer Abhängigkeitssemantiken aus. Ein abhängiges Objekt kann ohne `CASCADE` gelöscht werden, wenn eine seiner Abhängigkeiten ihre Bedingung für automatisches Löschen erfüllt. Umgekehrt müssen alle Einschränkungen der Abhängigkeiten dazu erfüllt sein, welche Objekte gemeinsam gelöscht werden müssen.

Die meisten während `initdb` erstellten Objekte gelten als "pinned", was bedeutet, dass das System selbst von ihnen abhängt. Daher dürfen sie niemals gelöscht werden. Da außerdem bekannt ist, dass gepinnte Objekte nicht gelöscht werden, erzeugt der Abhängigkeitsmechanismus keine `pg_depend`-Einträge, die Abhängigkeiten von ihnen zeigen. So hat zum Beispiel eine Tabellenspalte des Typs `numeric` begrifflich eine `NORMAL`-Abhängigkeit vom Datentyp `numeric`, aber ein solcher Eintrag erscheint tatsächlich nicht in `pg_depend`.

## 52.19. pg_description

Der Katalog `pg_description` speichert optionale Beschreibungen (Kommentare) für jedes Datenbankobjekt. Beschreibungen können mit dem Befehl `COMMENT` bearbeitet und mit den `\d`-Befehlen von `psql` angezeigt werden. Beschreibungen vieler eingebauter Systemobjekte werden im Anfangsinhalt von `pg_description` bereitgestellt.

Siehe auch `pg_shdescription`, das eine ähnliche Funktion für Beschreibungen erfüllt, an denen Objekte beteiligt sind, die über einen Datenbankcluster hinweg gemeinsam genutzt werden.

**Tabelle 52.19. Spalten von `pg_description`**

| Spalte | Beschreibung |
| --- | --- |
| `objoid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des Objekts, zu dem diese Beschreibung gehört |
| `classoid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem dieses Objekt erscheint |
| `objsubid` `int4` | Bei einem Kommentar zu einer Tabellenspalte ist dies die Spaltennummer (`objoid` und `classoid` referenzieren die Tabelle selbst). Für alle anderen Objekttypen ist diese Spalte null. |
| `description` `text` | Beliebiger Text, der als Beschreibung dieses Objekts dient |

## 52.20. pg_enum

Der Katalog `pg_enum` enthält Einträge, die die Werte und Labels für jeden Enum-Typ zeigen. Die interne Darstellung eines gegebenen Enum-Werts ist tatsächlich die OID seiner zugehörigen Zeile in `pg_enum`.

**Tabelle 52.20. Spalten von `pg_enum`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `enumtypid` `oid` (referenziert `pg_type.oid`) | Die OID des `pg_type`-Eintrags, zu dem dieser Enum-Wert gehört |
| `enumsortorder` `float4` | Die Sortierposition dieses Enum-Werts innerhalb seines Enum-Typs |
| `enumlabel` `name` | Das Textlabel für diesen Enum-Wert |

Die OIDs für `pg_enum`-Zeilen folgen einer besonderen Regel: Geradzahlige OIDs sind garantiert in derselben Weise geordnet wie die Sortierreihenfolge ihres Enum-Typs. Das heißt, wenn zwei gerade OIDs zum selben Enum-Typ gehören, muss die kleinere OID den kleineren `enumsortorder`-Wert haben. Ungeradzahlige OID-Werte müssen keine Beziehung zur Sortierreihenfolge haben. Diese Regel erlaubt es den Enum-Vergleichsroutinen, Katalognachschläge in vielen häufigen Fällen zu vermeiden. Die Routinen, die Enum-Typen erstellen und ändern, versuchen nach Möglichkeit, Enum-Werten gerade OIDs zuzuweisen.

Wenn ein Enum-Typ erstellt wird, werden seinen Mitgliedern Sortierpositionen `1..n` zugewiesen. Später hinzugefügte Mitglieder können jedoch negative oder gebrochene Werte für `enumsortorder` erhalten. Die einzige Anforderung an diese Werte ist, dass sie innerhalb jedes Enum-Typs korrekt geordnet und eindeutig sind.

## 52.21. pg_event_trigger

Der Katalog `pg_event_trigger` speichert Event-Trigger. Weitere Informationen finden Sie in [Kapitel 38](38_Event_Trigger.md).

**Tabelle 52.21. Spalten von `pg_event_trigger`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `evtname` `name` | Triggername (muss eindeutig sein) |
| `evtevent` `name` | Identifiziert das Ereignis, für das dieser Trigger ausgelöst wird |
| `evtowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Event-Triggers |
| `evtfoid` `oid` (referenziert `pg_proc.oid`) | Die aufzurufende Funktion |
| `evtenabled` `char` | Steuert, in welchen Modi von `session_replication_role` der Event-Trigger ausgelöst wird. `O` = Trigger wird in den Modi "origin" und "local" ausgelöst, `D` = Trigger ist deaktiviert, `R` = Trigger wird im Modus "replica" ausgelöst, `A` = Trigger wird immer ausgelöst. |
| `evttags` `text[]` | Command-Tags, für die dieser Trigger ausgelöst wird. Wenn `NULL`, ist das Auslösen dieses Triggers nicht anhand des Command-Tags eingeschränkt. |

## 52.22. pg_extension

Der Katalog `pg_extension` speichert Informationen über die installierten Extensions. Details zu Extensions finden Sie in [Abschnitt 36.17](36_SQL_erweitern.md#3617-zusammengehörige-objekte-in-einer-erweiterung-paketieren).

**Tabelle 52.22. Spalten von `pg_extension`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `extname` `name` | Name der Extension |
| `extowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Extension |
| `extnamespace` `oid` (referenziert `pg_namespace.oid`) | Schema, das die exportierten Objekte der Extension enthält |
| `extrelocatable` `bool` | Wahr, wenn die Extension in ein anderes Schema verschoben werden kann |
| `extversion` `text` | Versionsname der Extension |
| `extconfig` `oid[]` (referenziert `pg_class.oid`) | Array von `regclass`-OIDs für die Konfigurationstabelle(n) der Extension oder `NULL`, wenn keine vorhanden sind |
| `extcondition` `text[]` | Array von Filterbedingungen für `WHERE`-Klauseln der Konfigurationstabelle(n) der Extension oder `NULL`, wenn keine vorhanden sind |

Beachten Sie, dass `extnamespace` anders als bei den meisten Katalogen mit einer "namespace"-Spalte nicht bedeuten soll, dass die Extension zu diesem Schema gehört. Extension-Namen werden nie schemaqualifiziert. Stattdessen gibt `extnamespace` das Schema an, das die meisten oder alle Objekte der Extension enthält. Wenn `extrelocatable` wahr ist, muss dieses Schema tatsächlich alle schemaqualifizierbaren Objekte enthalten, die zur Extension gehören.

## 52.23. pg_foreign_data_wrapper

Der Katalog `pg_foreign_data_wrapper` speichert Definitionen von Foreign-Data-Wrappern. Ein Foreign-Data-Wrapper ist der Mechanismus, mit dem auf externe Daten zugegriffen wird, die auf Foreign Servern liegen.

**Tabelle 52.23. Spalten von `pg_foreign_data_wrapper`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `fdwname` `name` | Name des Foreign-Data-Wrappers |
| `fdwowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Foreign-Data-Wrappers |
| `fdwhandler` `oid` (referenziert `pg_proc.oid`) | Referenziert eine Handler-Funktion, die dafür verantwortlich ist, Ausführungsroutinen für den Foreign-Data-Wrapper bereitzustellen. Null, wenn kein Handler bereitgestellt wird |
| `fdwvalidator` `oid` (referenziert `pg_proc.oid`) | Referenziert eine Validator-Funktion, die dafür verantwortlich ist, die Gültigkeit der Optionen zu prüfen, die dem Foreign-Data-Wrapper übergeben werden, sowie Optionen für Foreign Server und User Mappings, die den Foreign-Data-Wrapper verwenden. Null, wenn kein Validator bereitgestellt wird |
| `fdwacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |
| `fdwoptions` `text[]` | Foreign-Data-Wrapper-spezifische Optionen als Zeichenketten der Form `keyword=value` |

## 52.24. pg_foreign_server

Der Katalog `pg_foreign_server` speichert Definitionen von Foreign Servern. Ein Foreign Server beschreibt eine Quelle externer Daten, etwa einen entfernten Server. Auf Foreign Server wird über Foreign-Data-Wrapper zugegriffen.

**Tabelle 52.24. Spalten von `pg_foreign_server`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `srvname` `name` | Name des Foreign Servers |
| `srvowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Foreign Servers |
| `srvfdw` `oid` (referenziert `pg_foreign_data_wrapper.oid`) | OID des Foreign-Data-Wrappers dieses Foreign Servers |
| `srvtype` `text` | Typ des Servers (optional) |
| `srvversion` `text` | Version des Servers (optional) |
| `srvacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |
| `srvoptions` `text[]` | Foreign-Server-spezifische Optionen als Zeichenketten der Form `keyword=value` |

## 52.25. pg_foreign_table

Der Katalog `pg_foreign_table` enthält Hilfsinformationen über Foreign Tables. Eine Foreign Table wird, genau wie eine gewöhnliche Tabelle, hauptsächlich durch einen `pg_class`-Eintrag dargestellt. Ihr `pg_foreign_table`-Eintrag enthält die Informationen, die nur für Foreign Tables und für keine andere Relationsart relevant sind.

**Tabelle 52.25. Spalten von `pg_foreign_table`**

| Spalte | Beschreibung |
| --- | --- |
| `ftrelid` `oid` (referenziert `pg_class.oid`) | Die OID des `pg_class`-Eintrags für diese Foreign Table |
| `ftserver` `oid` (referenziert `pg_foreign_server.oid`) | OID des Foreign Servers für diese Foreign Table |
| `ftoptions` `text[]` | Foreign-Table-Optionen als Zeichenketten der Form `keyword=value` |

## 52.26. pg_index

Der Katalog `pg_index` enthält einen Teil der Informationen über Indizes. Der Rest befindet sich hauptsächlich in `pg_class`.

**Tabelle 52.26. Spalten von `pg_index`**

| Spalte | Beschreibung |
| --- | --- |
| `indexrelid` `oid` (referenziert `pg_class.oid`) | Die OID des `pg_class`-Eintrags für diesen Index |
| `indrelid` `oid` (referenziert `pg_class.oid`) | Die OID des `pg_class`-Eintrags für die Tabelle, zu der dieser Index gehört |
| `indnatts` `int2` | Die Gesamtzahl der Spalten im Index (dupliziert `pg_class.relnatts`); diese Zahl umfasst sowohl Schlüssel- als auch eingeschlossene Attribute |
| `indnkeyatts` `int2` | Die Anzahl der Schlüsselspalten im Index, ohne eingeschlossene Spalten mitzuzählen, die lediglich gespeichert werden und nicht an der Indexsemantik teilnehmen |
| `indisunique` `bool` | Wenn wahr, ist dies ein Unique Index |
| `indnullsnotdistinct` `bool` | Dieser Wert wird nur für Unique Indexes verwendet. Wenn falsch, betrachtet dieser Unique Index Nullwerte als verschieden (der Index kann also mehrere Nullwerte in einer Spalte enthalten, das Standardverhalten von PostgreSQL). Wenn wahr, betrachtet er Nullwerte als gleich (der Index kann also nur einen Nullwert in einer Spalte enthalten). |
| `indisprimary` `bool` | Wenn wahr, stellt dieser Index den Primärschlüssel der Tabelle dar (`indisunique` sollte immer wahr sein, wenn dies wahr ist) |
| `indisexclusion` `bool` | Wenn wahr, unterstützt dieser Index einen Exclusion-Constraint |
| `indimmediate` `bool` | Wenn wahr, wird die Eindeutigkeitsprüfung sofort beim Einfügen erzwungen (irrelevant, wenn `indisunique` nicht wahr ist) |
| `indisclustered` `bool` | Wenn wahr, wurde die Tabelle zuletzt auf diesem Index geclustert |
| `indisvalid` `bool` | Wenn wahr, ist der Index derzeit für Abfragen gültig. Falsch bedeutet, dass der Index möglicherweise unvollständig ist: Er muss weiterhin durch `INSERT`-/`UPDATE`-Operationen geändert werden, kann aber für Abfragen nicht sicher verwendet werden. Wenn er eindeutig ist, ist auch die Eindeutigkeitseigenschaft nicht garantiert wahr. |
| `indcheckxmin` `bool` | Wenn wahr, dürfen Abfragen den Index nicht verwenden, bis die `xmin` dieser `pg_index`-Zeile unter ihrem `TransactionXmin`-Ereignishorizont liegt, weil die Tabelle beschädigte HOT-Ketten mit inkompatiblen Zeilen enthalten kann, die sie sehen können |
| `indisready` `bool` | Wenn wahr, ist der Index derzeit für Einfügungen bereit. Falsch bedeutet, dass der Index von `INSERT`-/`UPDATE`-Operationen ignoriert werden muss. |
| `indislive` `bool` | Wenn falsch, wird der Index gerade gelöscht und sollte für alle Zwecke ignoriert werden (einschließlich HOT-Sicherheitsentscheidungen) |
| `indisreplident` `bool` | Wenn wahr, wurde dieser Index mit `ALTER TABLE ... REPLICA IDENTITY USING INDEX ...` als "Replica Identity" ausgewählt |
| `indkey` `int2vector` (referenziert `pg_attribute.attnum`) | Dies ist ein Array aus `indnatts`-Werten, die angeben, welche Tabellenspalten dieser Index indiziert. Ein Wert von `1 3` würde zum Beispiel bedeuten, dass die erste und die dritte Tabellenspalte die Indexeinträge bilden. Schlüsselspalten stehen vor Nicht-Schlüsselspalten (eingeschlossenen Spalten). Eine null in diesem Array zeigt an, dass das entsprechende Indexattribut ein Ausdruck über den Tabellenspalten ist und keine einfache Spaltenreferenz. |
| `indcollation` `oidvector` (referenziert `pg_collation.oid`) | Enthält für jede Spalte im Indexschlüssel (`indnkeyatts`-Werte) die OID der für den Index zu verwendenden Collation oder null, wenn die Spalte keinen collatable Datentyp hat. |
| `indclass` `oidvector` (referenziert `pg_opclass.oid`) | Enthält für jede Spalte im Indexschlüssel (`indnkeyatts`-Werte) die OID der zu verwendenden Operatorklasse. Details finden Sie unter `pg_opclass`. |
| `indoption` `int2vector` | Dies ist ein Array aus `indnkeyatts`-Werten, das spaltenbezogene Flag-Bits speichert. Die Bedeutung der Bits wird durch die Zugriffsmethode des Index definiert. |
| `indexprs` `pg_node_tree` | Ausdrucksbäume (in `nodeToString()`-Darstellung) für Indexattribute, die keine einfachen Spaltenreferenzen sind. Dies ist eine Liste mit einem Element für jeden null-Eintrag in `indkey`. Null, wenn alle Indexattribute einfache Referenzen sind. |
| `indpred` `pg_node_tree` | Ausdrucksbaum (in `nodeToString()`-Darstellung) für das Prädikat eines partiellen Index. Null, wenn es kein partieller Index ist. |

## 52.27. pg_inherits

Der Katalog `pg_inherits` zeichnet Informationen über Vererbungshierarchien von Tabellen und Indizes auf. Es gibt einen Eintrag für jede direkte Parent-Child-Beziehung zwischen Tabellen oder Indizes in der Datenbank. (Indirekte Vererbung kann durch Verfolgen von Eintragsketten ermittelt werden.)

**Tabelle 52.27. Spalten von `pg_inherits`**

| Spalte | Beschreibung |
| --- | --- |
| `inhrelid` `oid` (referenziert `pg_class.oid`) | Die OID der untergeordneten Tabelle oder des untergeordneten Index |
| `inhparent` `oid` (referenziert `pg_class.oid`) | Die OID der übergeordneten Tabelle oder des übergeordneten Index |
| `inhseqno` `int4` | Wenn es mehr als einen direkten Parent für eine untergeordnete Tabelle gibt (Mehrfachvererbung), gibt diese Zahl die Reihenfolge an, in der die geerbten Spalten angeordnet werden sollen. Die Zählung beginnt bei 1. Indizes können keine Mehrfachvererbung haben, da sie nur bei deklarativer Partitionierung erben können. |
| `inhdetachpending` `bool` | Wahr für eine Partition, die gerade abgetrennt wird; andernfalls falsch. |

## 52.28. pg_init_privs

Der Katalog `pg_init_privs` zeichnet Informationen über die Initialprivilegien von Objekten im System auf. Es gibt einen Eintrag für jedes Objekt in der Datenbank, das eine nicht standardmäßige (nicht `NULL`) anfängliche Privilegienmenge hat.

Objekte können Initialprivilegien entweder dadurch haben, dass diese Privilegien bei der Systeminitialisierung (durch `initdb`) gesetzt werden, oder dadurch, dass das Objekt während eines `CREATE EXTENSION` erstellt wird und das Extension-Skript Initialprivilegien mit dem `GRANT`-System setzt. Beachten Sie, dass das System die Aufzeichnung der Privilegien während des Extension-Skripts automatisch übernimmt und Extension-Autoren in ihrem Skript nur die Anweisungen `GRANT` und `REVOKE` verwenden müssen, damit die Privilegien aufgezeichnet werden. Die Spalte `privtype` gibt an, ob das Initialprivileg durch `initdb` oder während eines `CREATE EXTENSION`-Befehls gesetzt wurde.

Objekte, deren Initialprivilegien durch `initdb` gesetzt wurden, haben Einträge, bei denen `privtype` den Wert `i` hat, während Objekte, deren Initialprivilegien durch `CREATE EXTENSION` gesetzt wurden, Einträge haben, bei denen `privtype` den Wert `e` hat.

**Tabelle 52.28. Spalten von `pg_init_privs`**

| Spalte | Beschreibung |
| --- | --- |
| `objoid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des konkreten Objekts |
| `classoid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem sich das Objekt befindet |
| `objsubid` `int4` | Bei einer Tabellenspalte ist dies die Spaltennummer (`objoid` und `classoid` referenzieren die Tabelle selbst). Für alle anderen Objekttypen ist diese Spalte null. |
| `privtype` `char` | Ein Code, der den Typ des Initialprivilegs dieses Objekts definiert; siehe Text |
| `initprivs` `aclitem[]` | Die anfänglichen Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

## 52.29. pg_language

Der Katalog `pg_language` registriert Sprachen, in denen Funktionen oder Stored Procedures geschrieben werden können. Weitere Informationen zu Language-Handlern finden Sie unter `CREATE LANGUAGE` und in [Kapitel 40](40_Prozedurale_Sprachen.md).

**Tabelle 52.29. Spalten von `pg_language`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `lanname` `name` | Name der Sprache |
| `lanowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Sprache |
| `lanispl` `bool` | Dies ist für interne Sprachen (wie SQL) falsch und für benutzerdefinierte Sprachen wahr. Derzeit verwendet `pg_dump` dies noch, um zu bestimmen, welche Sprachen gedumpt werden müssen, aber dies könnte in Zukunft durch einen anderen Mechanismus ersetzt werden. |
| `lanpltrusted` `bool` | Wahr, wenn dies eine vertrauenswürdige Sprache ist, was bedeutet, dass angenommen wird, dass sie keinen Zugriff auf etwas außerhalb der normalen SQL-Ausführungsumgebung gewährt. Nur Superuser können Funktionen in nicht vertrauenswürdigen Sprachen erstellen. |
| `lanplcallfoid` `oid` (referenziert `pg_proc.oid`) | Bei nicht internen Sprachen referenziert dies den Language-Handler, eine spezielle Funktion, die für die Ausführung aller Funktionen verantwortlich ist, die in der betreffenden Sprache geschrieben sind. Null für interne Sprachen. |
| `laninline` `oid` (referenziert `pg_proc.oid`) | Referenziert eine Funktion, die für die Ausführung von "inline" anonymen Codeblöcken (`DO`-Blöcken) verantwortlich ist. Null, wenn Inline-Blöcke nicht unterstützt werden. |
| `lanvalidator` `oid` (referenziert `pg_proc.oid`) | Referenziert eine Language-Validator-Funktion, die dafür verantwortlich ist, Syntax und Gültigkeit neuer Funktionen bei deren Erstellung zu prüfen. Null, wenn kein Validator bereitgestellt wird. |
| `lanacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

## 52.30. pg_largeobject

Der Katalog `pg_largeobject` enthält die Daten, aus denen "Large Objects" bestehen. Ein Large Object wird durch eine OID identifiziert, die ihm bei der Erstellung zugewiesen wird. Jedes Large Object wird in Segmente oder "Seiten" zerlegt, die klein genug sind, um bequem als Zeilen in `pg_largeobject` gespeichert zu werden. Die Datenmenge pro Seite ist als `LOBLKSIZE` definiert (derzeit `BLCKSZ/4`, typischerweise 2 kB).

Vor PostgreSQL 9.0 gab es keine Berechtigungsstruktur für Large Objects. Deshalb war `pg_largeobject` öffentlich lesbar und konnte verwendet werden, um die OIDs (und Inhalte) aller Large Objects im System zu erhalten. Das ist nicht mehr der Fall; verwenden Sie `pg_largeobject_metadata`, um eine Liste der Large-Object-OIDs zu erhalten.

**Tabelle 52.30. Spalten von `pg_largeobject`**

| Spalte | Beschreibung |
| --- | --- |
| `loid` `oid` (referenziert `pg_largeobject_metadata.oid`) | Bezeichner des Large Objects, das diese Seite enthält |
| `pageno` `int4` | Seitennummer dieser Seite innerhalb ihres Large Objects (von null an gezählt) |
| `data` `bytea` | Tatsächliche Daten, die im Large Object gespeichert sind. Dies sind nie mehr als `LOBLKSIZE` Bytes und können weniger sein. |

Jede Zeile von `pg_largeobject` enthält Daten für eine Seite eines Large Objects, beginnend bei Byte-Offset `(pageno * LOBLKSIZE)` innerhalb des Objekts. Die Implementierung erlaubt sparse Speicherung: Seiten können fehlen und kürzer als `LOBLKSIZE` Bytes sein, auch wenn sie nicht die letzte Seite des Objekts sind. Fehlende Bereiche innerhalb eines Large Objects werden als Nullen gelesen.

## 52.31. pg_largeobject_metadata

Der Katalog `pg_largeobject_metadata` enthält Metadaten, die Large Objects zugeordnet sind. Die eigentlichen Large-Object-Daten werden in `pg_largeobject` gespeichert.

**Tabelle 52.31. Spalten von `pg_largeobject_metadata`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `lomowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Large Objects |
| `lomacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

## 52.32. pg_namespace

Der Katalog `pg_namespace` speichert Namespaces. Ein Namespace ist die Struktur, die SQL-Schemas zugrunde liegt: Jeder Namespace kann eine eigene Sammlung von Relationen, Typen usw. ohne Namenskonflikte haben.

**Tabelle 52.32. Spalten von `pg_namespace`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `nspname` `name` | Name des Namespace |
| `nspowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Namespace |
| `nspacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

## 52.33. pg_opclass

Der Katalog `pg_opclass` definiert Operatorklassen für Indexzugriffsmethoden. Jede Operatorklasse definiert die Semantik für Indexspalten eines bestimmten Datentyps und einer bestimmten Indexzugriffsmethode. Eine Operatorklasse legt im Wesentlichen fest, dass eine bestimmte Operatorfamilie auf einen bestimmten indizierbaren Spaltendatentyp anwendbar ist. Die tatsächlich mit der indizierten Spalte nutzbare Menge von Operatoren aus der Familie besteht aus denjenigen Operatoren, die den Datentyp der Spalte als linken Eingabetyp akzeptieren.

Operatorklassen werden ausführlich in [Abschnitt 36.16](36_SQL_erweitern.md#3616-erweiterungen-an-indizes-anbinden) beschrieben.

**Tabelle 52.33. Spalten von `pg_opclass`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `opcmethod` `oid` (referenziert `pg_am.oid`) | Indexzugriffsmethode, für die diese Operatorklasse gilt |
| `opcname` `name` | Name dieser Operatorklasse |
| `opcnamespace` `oid` (referenziert `pg_namespace.oid`) | Namespace dieser Operatorklasse |
| `opcowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Operatorklasse |
| `opcfamily` `oid` (referenziert `pg_opfamily.oid`) | Operatorfamilie, die die Operatorklasse enthält |
| `opcintype` `oid` (referenziert `pg_type.oid`) | Datentyp, den die Operatorklasse indiziert |
| `opcdefault` `bool` | Wahr, wenn diese Operatorklasse der Default für `opcintype` ist |
| `opckeytype` `oid` (referenziert `pg_type.oid`) | Typ der im Index gespeicherten Daten oder null, wenn er mit `opcintype` identisch ist |

Die `opcmethod` einer Operatorklasse muss zur `opfmethod` ihrer enthaltenden Operatorfamilie passen. Außerdem darf es für eine gegebene Kombination aus `opcmethod` und `opcintype` höchstens eine `pg_opclass`-Zeile geben, bei der `opcdefault` wahr ist.

## 52.34. pg_operator

Der Katalog `pg_operator` speichert Informationen über Operatoren. Weitere Informationen finden Sie unter `CREATE OPERATOR` und in [Abschnitt 36.14](36_SQL_erweitern.md#3614-benutzerdefinierte-operatoren).

**Tabelle 52.34. Spalten von `pg_operator`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `oprname` `name` | Name des Operators |
| `oprnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diesen Operator enthält |
| `oprowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Operators |
| `oprkind` `char` | `b` = Infix-Operator ("both") oder `l` = Präfix-Operator ("left") |
| `oprcanmerge` `bool` | Dieser Operator unterstützt Merge Joins |
| `oprcanhash` `bool` | Dieser Operator unterstützt Hash Joins |
| `oprleft` `oid` (referenziert `pg_type.oid`) | Typ des linken Operanden (null für einen Präfix-Operator) |
| `oprright` `oid` (referenziert `pg_type.oid`) | Typ des rechten Operanden |
| `oprresult` `oid` (referenziert `pg_type.oid`) | Typ des Ergebnisses (null für einen noch nicht definierten "Shell"-Operator) |
| `oprcom` `oid` (referenziert `pg_operator.oid`) | Kommutator dieses Operators (null, wenn keiner vorhanden ist) |
| `oprnegate` `oid` (referenziert `pg_operator.oid`) | Negator dieses Operators (null, wenn keiner vorhanden ist) |
| `oprcode` `regproc` (referenziert `pg_proc.oid`) | Funktion, die diesen Operator implementiert (null für einen noch nicht definierten "Shell"-Operator) |
| `oprrest` `regproc` (referenziert `pg_proc.oid`) | Funktion zur Schätzung der Restriktionsselektivität für diesen Operator (null, wenn keine vorhanden ist) |
| `oprjoin` `regproc` (referenziert `pg_proc.oid`) | Funktion zur Schätzung der Join-Selektivität für diesen Operator (null, wenn keine vorhanden ist) |

## 52.35. pg_opfamily

Der Katalog `pg_opfamily` definiert Operatorfamilien. Jede Operatorfamilie ist eine Sammlung von Operatoren und zugehörigen Support-Routinen, die die für eine bestimmte Indexzugriffsmethode angegebene Semantik implementieren. Außerdem sind die Operatoren in einer Familie in einer durch die Zugriffsmethode festgelegten Weise "kompatibel". Das Konzept der Operatorfamilie erlaubt es, datentypübergreifende Operatoren mit Indizes zu verwenden und sie anhand des Wissens über die Semantik der Zugriffsmethode zu verstehen.

Operatorfamilien werden ausführlich in [Abschnitt 36.16](36_SQL_erweitern.md#3616-erweiterungen-an-indizes-anbinden) beschrieben.

**Tabelle 52.35. Spalten von `pg_opfamily`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `opfmethod` `oid` (referenziert `pg_am.oid`) | Indexzugriffsmethode, für die diese Operatorfamilie gilt |
| `opfname` `name` | Name dieser Operatorfamilie |
| `opfnamespace` `oid` (referenziert `pg_namespace.oid`) | Namespace dieser Operatorfamilie |
| `opfowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Operatorfamilie |

Der Großteil der Informationen, die eine Operatorfamilie definieren, steht nicht in ihrer `pg_opfamily`-Zeile, sondern in den zugehörigen Zeilen in `pg_amop`, `pg_amproc` und `pg_opclass`.

## 52.36. pg_parameter_acl

Der Katalog `pg_parameter_acl` zeichnet Konfigurationsparameter auf, für die einer oder mehreren Rollen Privilegien gewährt wurden. Für Parameter mit Default-Privilegien wird kein Eintrag angelegt.

Anders als die meisten Systemkataloge wird `pg_parameter_acl` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_parameter_acl` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.36. Spalten von `pg_parameter_acl`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `parname` `text` | Der Name eines Konfigurationsparameters, für den Privilegien gewährt werden |
| `paracl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

## 52.37. pg_partitioned_table

Der Katalog `pg_partitioned_table` speichert Informationen darüber, wie Tabellen partitioniert sind.

**Tabelle 52.37. Spalten von `pg_partitioned_table`**

| Spalte | Beschreibung |
| --- | --- |
| `partrelid` `oid` (referenziert `pg_class.oid`) | Die OID des `pg_class`-Eintrags für diese partitionierte Tabelle |
| `partstrat` `char` | Partitionierungsstrategie; `h` = Hash-partitionierte Tabelle, `l` = Listen-partitionierte Tabelle, `r` = Range-partitionierte Tabelle |
| `partnatts` `int2` | Die Anzahl der Spalten im Partitionierungsschlüssel |
| `partdefid` `oid` (referenziert `pg_class.oid`) | Die OID des `pg_class`-Eintrags für die Default-Partition dieser partitionierten Tabelle oder null, wenn diese partitionierte Tabelle keine Default-Partition hat |
| `partattrs` `int2vector` (referenziert `pg_attribute.attnum`) | Dies ist ein Array aus `partnatts`-Werten, die angeben, welche Tabellenspalten Teil des Partitionierungsschlüssels sind. Ein Wert von `1 3` würde zum Beispiel bedeuten, dass die erste und die dritte Tabellenspalte den Partitionierungsschlüssel bilden. Eine null in diesem Array zeigt an, dass die entsprechende Partitionierungsschlüsselspalte ein Ausdruck ist und keine einfache Spaltenreferenz. |
| `partclass` `oidvector` (referenziert `pg_opclass.oid`) | Enthält für jede Spalte im Partitionierungsschlüssel die OID der zu verwendenden Operatorklasse. Details finden Sie unter `pg_opclass`. |
| `partcollation` `oidvector` (referenziert `pg_collation.oid`) | Enthält für jede Spalte im Partitionierungsschlüssel die OID der für die Partitionierung zu verwendenden Collation oder null, wenn die Spalte keinen collatable Datentyp hat. |
| `partexprs` `pg_node_tree` | Ausdrucksbäume (in `nodeToString()`-Darstellung) für Partitionierungsschlüsselspalten, die keine einfachen Spaltenreferenzen sind. Dies ist eine Liste mit einem Element für jeden null-Eintrag in `partattrs`. Null, wenn alle Partitionierungsschlüsselspalten einfache Referenzen sind. |

## 52.38. pg_policy

Der Katalog `pg_policy` speichert Row-Level-Security-Richtlinien für Tabellen. Eine Richtlinie umfasst die Art des Befehls, auf den sie angewendet wird (möglicherweise alle Befehle), die Rollen, für die sie gilt, den Ausdruck, der als Security-Barrier-Qualifikation zu Abfragen hinzugefügt wird, die die Tabelle enthalten, sowie den Ausdruck, der als `WITH CHECK`-Option zu Abfragen hinzugefügt wird, die versuchen, neue Datensätze in die Tabelle einzufügen.

**Tabelle 52.38. Spalten von `pg_policy`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `polname` `name` | Der Name der Richtlinie |
| `polrelid` `oid` (referenziert `pg_class.oid`) | Die Tabelle, für die die Richtlinie gilt |
| `polcmd` `char` | Der Befehlstyp, auf den die Richtlinie angewendet wird: `r` für `SELECT`, `a` für `INSERT`, `w` für `UPDATE`, `d` für `DELETE` oder `*` für alle |
| `polpermissive` `bool` | Ist die Richtlinie permissiv oder restriktiv? |
| `polroles` `oid[]` (referenziert `pg_authid.oid`) | Die Rollen, für die die Richtlinie gilt; null bedeutet `PUBLIC` (und erscheint normalerweise allein im Array) |
| `polqual` `pg_node_tree` | Der Ausdrucksbaum, der zu den Security-Barrier-Qualifikationen für Abfragen hinzugefügt wird, die die Tabelle verwenden |
| `polwithcheck` `pg_node_tree` | Der Ausdrucksbaum, der zu den `WITH CHECK`-Qualifikationen für Abfragen hinzugefügt wird, die versuchen, Zeilen zur Tabelle hinzuzufügen |

> **Hinweis**
>
> In `pg_policy` gespeicherte Richtlinien werden nur angewendet, wenn `pg_class.relrowsecurity` für ihre Tabelle gesetzt ist.

## 52.39. pg_proc

Der Katalog `pg_proc` speichert Informationen über Funktionen, Prozeduren, Aggregatfunktionen und Window-Funktionen (zusammen auch als Routinen bezeichnet). Weitere Informationen finden Sie unter `CREATE FUNCTION`, `CREATE PROCEDURE` und in [Abschnitt 36.3](36_SQL_erweitern.md#363-benutzerdefinierte-funktionen).

Wenn `prokind` angibt, dass der Eintrag eine Aggregatfunktion betrifft, sollte es eine passende Zeile in `pg_aggregate` geben.

**Tabelle 52.39. Spalten von `pg_proc`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `proname` `name` | Name der Funktion |
| `pronamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diese Funktion enthält |
| `proowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Funktion |
| `prolang` `oid` (referenziert `pg_language.oid`) | Implementierungssprache oder Aufrufschnittstelle dieser Funktion |
| `procost` `float4` | Geschätzte Ausführungskosten (in Einheiten von `cpu_operator_cost`); wenn `proretset` gesetzt ist, sind dies die Kosten pro zurückgegebener Zeile |
| `prorows` `float4` | Geschätzte Anzahl von Ergebniszeilen (null, wenn `proretset` nicht gesetzt ist) |
| `provariadic` `oid` (referenziert `pg_type.oid`) | Datentyp der Elemente des variadischen Array-Parameters oder null, wenn die Funktion keinen variadischen Parameter hat |
| `prosupport` `regproc` (referenziert `pg_proc.oid`) | Planner-Support-Funktion für diese Funktion (siehe [Abschnitt 36.11](36_SQL_erweitern.md#3611-informationen-zur-funktionsoptimierung)) oder null, wenn keine vorhanden ist |
| `prokind` `char` | `f` für eine normale Funktion, `p` für eine Prozedur, `a` für eine Aggregatfunktion oder `w` für eine Window-Funktion |
| `prosecdef` `bool` | Die Funktion ist ein Security Definer (d. h. eine "setuid"-Funktion) |
| `proleakproof` `bool` | Die Funktion hat keine Seiteneffekte. Über die Argumente werden außer über den Rückgabewert keine Informationen vermittelt. Jede Funktion, die abhängig von den Werten ihrer Argumente einen Fehler auslösen könnte, ist nicht leakproof. |
| `proisstrict` `bool` | Die Funktion gibt null zurück, wenn irgendein Aufrufargument null ist. In diesem Fall wird die Funktion tatsächlich gar nicht aufgerufen. Funktionen, die nicht "strict" sind, müssen darauf vorbereitet sein, null-Eingaben zu behandeln. |
| `proretset` `bool` | Die Funktion gibt eine Menge zurück (d. h. mehrere Werte des angegebenen Datentyps) |
| `provolatile` `char` | `provolatile` gibt an, ob das Ergebnis der Funktion nur von ihren Eingabeargumenten abhängt oder von äußeren Faktoren beeinflusst wird. `i` steht für "immutable" Funktionen, die bei gleichen Eingaben immer dasselbe Ergebnis liefern. `s` steht für "stable" Funktionen, deren Ergebnisse sich bei festen Eingaben innerhalb eines Scans nicht ändern. `v` steht für "volatile" Funktionen, deren Ergebnisse sich jederzeit ändern können. (Verwenden Sie `v` auch für Funktionen mit Seiteneffekten, damit Aufrufe nicht wegoptimiert werden können.) |
| `proparallel` `char` | `proparallel` gibt an, ob die Funktion sicher im Parallelmodus ausgeführt werden kann. `s` steht für Funktionen, die ohne Einschränkung sicher im Parallelmodus ausgeführt werden können. `r` steht für Funktionen, die im Parallelmodus ausgeführt werden können, deren Ausführung aber auf den Leiter der Parallelgruppe beschränkt ist; parallele Worker-Prozesse können diese Funktionen nicht aufrufen. `u` steht für Funktionen, die im Parallelmodus unsicher sind; das Vorhandensein einer solchen Funktion erzwingt einen seriellen Ausführungsplan. |
| `pronargs` `int2` | Anzahl der Eingabeargumente |
| `pronargdefaults` `int2` | Anzahl der Argumente, die Default-Werte haben |
| `prorettype` `oid` (referenziert `pg_type.oid`) | Datentyp des Rückgabewerts |
| `proargtypes` `oidvector` (referenziert `pg_type.oid`) | Ein Array der Datentypen der Funktionsargumente. Es enthält nur Eingabeargumente (einschließlich `INOUT`- und `VARIADIC`-Argumenten) und stellt daher die Aufrufsignatur der Funktion dar. |
| `proallargtypes` `oid[]` (referenziert `pg_type.oid`) | Ein Array der Datentypen der Funktionsargumente. Es enthält alle Argumente (einschließlich `OUT`- und `INOUT`-Argumenten); wenn jedoch alle Argumente `IN`-Argumente sind, ist dieses Feld null. Beachten Sie, dass die Indizierung 1-basiert ist, während `proargtypes` aus historischen Gründen ab 0 indiziert wird. |
| `proargmodes` `char[]` | Ein Array der Modi der Funktionsargumente, codiert als `i` für `IN`-Argumente, `o` für `OUT`-Argumente, `b` für `INOUT`-Argumente, `v` für `VARIADIC`-Argumente und `t` für `TABLE`-Argumente. Wenn alle Argumente `IN`-Argumente sind, ist dieses Feld null. Beachten Sie, dass die Indizes Positionen in `proallargtypes` entsprechen, nicht in `proargtypes`. |
| `proargnames` `text[]` | Ein Array der Namen der Funktionsargumente. Argumente ohne Namen werden im Array auf leere Zeichenketten gesetzt. Wenn keines der Argumente einen Namen hat, ist dieses Feld null. Beachten Sie, dass die Indizes Positionen in `proallargtypes` entsprechen, nicht in `proargtypes`. |
| `proargdefaults` `pg_node_tree` | Ausdrucksbäume (in `nodeToString()`-Darstellung) für Default-Werte. Dies ist eine Liste mit `pronargdefaults` Elementen, die den letzten N Eingabeargumenten entsprechen (d. h. den letzten N Positionen in `proargtypes`). Wenn keines der Argumente Default-Werte hat, ist dieses Feld null. |
| `protrftypes` `oid[]` (referenziert `pg_type.oid`) | Ein Array der Argument-/Ergebnis-Datentypen, auf die Transforms angewendet werden sollen (aus der `TRANSFORM`-Klausel der Funktion). Null, wenn keine vorhanden sind. |
| `prosrc` `text` | Gibt dem Function Handler an, wie die Funktion aufzurufen ist. Dies kann abhängig von Implementierungssprache oder Aufrufkonvention der tatsächliche Quellcode der Funktion für interpretierte Sprachen, ein Link-Symbol, ein Dateiname oder fast alles andere sein. |
| `probin` `text` | Zusätzliche Informationen darüber, wie die Funktion aufzurufen ist. Auch hier ist die Interpretation sprachspezifisch. |
| `prosqlbody` `pg_node_tree` | Vorab geparster SQL-Funktionskörper. Dies wird für SQL-Sprachfunktionen verwendet, wenn der Körper in SQL-Standardnotation statt als Zeichenkettenliteral angegeben ist. In anderen Fällen ist es null. |
| `proconfig` `text[]` | Lokale Einstellungen der Funktion für Laufzeit-Konfigurationsvariablen |
| `proacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

Bei kompilierten Funktionen, sowohl eingebauten als auch dynamisch geladenen, enthält `prosrc` den C-Sprachnamen der Funktion (Link-Symbol). Bei SQL-Sprachfunktionen enthält `prosrc` den Quelltext der Funktion, wenn dieser als Zeichenkettenliteral angegeben ist; wenn der Funktionskörper aber im SQL-Standardstil angegeben ist, wird `prosrc` nicht verwendet (typischerweise ist es eine leere Zeichenkette), und `prosqlbody` enthält die vorab geparste Definition. Bei allen anderen derzeit bekannten Sprachtypen enthält `prosrc` den Quelltext der Funktion. `probin` ist null, außer bei dynamisch geladenen C-Funktionen; dort enthält es den Namen der Shared-Library-Datei, die die Funktion enthält.

## 52.40. pg_publication

Der Katalog `pg_publication` enthält alle in der Datenbank erstellten Publications. Weitere Informationen zu Publications finden Sie in [Abschnitt 29.1](29_Logische_Replikation.md#291-publication).

**Tabelle 52.40. Spalten von `pg_publication`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `pubname` `name` | Name der Publication |
| `pubowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Publication |
| `puballtables` `bool` | Wenn wahr, enthält diese Publication automatisch alle Tabellen in der Datenbank, einschließlich solcher, die in Zukunft erstellt werden. |
| `pubinsert` `bool` | Wenn wahr, werden `INSERT`-Operationen für Tabellen in der Publication repliziert. |
| `pubupdate` `bool` | Wenn wahr, werden `UPDATE`-Operationen für Tabellen in der Publication repliziert. |
| `pubdelete` `bool` | Wenn wahr, werden `DELETE`-Operationen für Tabellen in der Publication repliziert. |
| `pubtruncate` `bool` | Wenn wahr, werden `TRUNCATE`-Operationen für Tabellen in der Publication repliziert. |
| `pubviaroot` `bool` | Wenn wahr, werden Operationen auf einer Blattpartition unter Verwendung der Identität und des Schemas ihres obersten partitionierten Vorfahren repliziert, der in der Publication genannt ist, statt ihrer eigenen. |
| `pubgencols` `char` | Steuert, wie die Replikation generierter Spalten behandelt wird, wenn es keine Publication-Spaltenliste gibt: `n` = generierte Spalten in den mit der Publication verknüpften Tabellen sollen nicht repliziert werden, `s` = gespeicherte generierte Spalten in den mit der Publication verknüpften Tabellen sollen repliziert werden. |

## 52.41. pg_publication_namespace

Der Katalog `pg_publication_namespace` enthält die Zuordnung zwischen Schemas und Publications in der Datenbank. Dies ist eine Many-to-Many-Zuordnung.

**Tabelle 52.41. Spalten von `pg_publication_namespace`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `pnpubid` `oid` (referenziert `pg_publication.oid`) | Referenz auf die Publication |
| `pnnspid` `oid` (referenziert `pg_namespace.oid`) | Referenz auf das Schema |

## 52.42. pg_publication_rel

Der Katalog `pg_publication_rel` enthält die Zuordnung zwischen Relationen und Publications in der Datenbank. Dies ist eine Many-to-Many-Zuordnung. Siehe auch [Abschnitt 53.18](53_System_Views.md#5318-pgpublicationtables) für eine benutzerfreundlichere Sicht auf diese Informationen.

**Tabelle 52.42. Spalten von `pg_publication_rel`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `prpubid` `oid` (referenziert `pg_publication.oid`) | Referenz auf die Publication |
| `prrelid` `oid` (referenziert `pg_class.oid`) | Referenz auf die Relation |
| `prqual` `pg_node_tree` | Ausdrucksbaum (in `nodeToString()`-Darstellung) für die qualifizierende Bedingung der Relation in der Publication. Null, wenn es keine qualifizierende Publication-Bedingung gibt. |
| `prattrs` `int2vector` (referenziert `pg_attribute.attnum`) | Dies ist ein Array von Werten, das angibt, welche Tabellenspalten Teil der Publication sind. Ein Wert von `1 3` würde zum Beispiel bedeuten, dass die erste und die dritte Tabellenspalte publiziert werden. Ein Nullwert zeigt an, dass alle Spalten publiziert werden. |

## 52.43. pg_range

Der Katalog `pg_range` speichert Informationen über Range-Typen. Dies ergänzt die Einträge der Typen in `pg_type`.

**Tabelle 52.43. Spalten von `pg_range`**

| Spalte | Beschreibung |
| --- | --- |
| `rngtypid` `oid` (referenziert `pg_type.oid`) | OID des Range-Typs |
| `rngsubtype` `oid` (referenziert `pg_type.oid`) | OID des Elementtyps (Subtyps) dieses Range-Typs |
| `rngmultitypid` `oid` (referenziert `pg_type.oid`) | OID des Multirange-Typs für diesen Range-Typ |
| `rngcollation` `oid` (referenziert `pg_collation.oid`) | OID der für Range-Vergleiche verwendeten Collation oder null, wenn keine vorhanden ist |
| `rngsubopc` `oid` (referenziert `pg_opclass.oid`) | OID der Operatorklasse des Subtyps, die für Range-Vergleiche verwendet wird |
| `rngcanonical` `regproc` (referenziert `pg_proc.oid`) | OID der Funktion, die einen Range-Wert in kanonische Form umwandelt, oder null, wenn keine vorhanden ist |
| `rngsubdiff` `regproc` (referenziert `pg_proc.oid`) | OID der Funktion, die die Differenz zwischen zwei Elementwerten als `double precision` zurückgibt, oder null, wenn keine vorhanden ist |

`rngsubopc` (zusammen mit `rngcollation`, wenn der Elementtyp collatable ist) bestimmt die vom Range-Typ verwendete Sortierreihenfolge. `rngcanonical` wird verwendet, wenn der Elementtyp diskret ist. `rngsubdiff` ist optional, sollte aber bereitgestellt werden, um die Performance von GiST-Indizes auf dem Range-Typ zu verbessern.

## 52.44. pg_replication_origin

Der Katalog `pg_replication_origin` enthält alle erstellten Replication Origins. Weitere Informationen zu Replication Origins finden Sie in [Kapitel 48](48_Fortschritt_der_Replikation_verfolgen.md).

Anders als die meisten Systemkataloge wird `pg_replication_origin` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_replication_origin` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.44. Spalten von `pg_replication_origin`**

| Spalte | Beschreibung |
| --- | --- |
| `roident` `oid` | Ein eindeutiger, clusterweiter Bezeichner für die Replication Origin. Sollte das System niemals verlassen. |
| `roname` `text` | Der externe, benutzerdefinierte Name einer Replication Origin. |

## 52.45. pg_rewrite

Der Katalog `pg_rewrite` speichert Rewrite-Regeln für Tabellen und Sichten.

**Tabelle 52.45. Spalten von `pg_rewrite`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `rulename` `name` | Regelname |
| `ev_class` `oid` (referenziert `pg_class.oid`) | Die Tabelle, für die diese Regel gilt |
| `ev_type` `char` | Ereignistyp, für den die Regel gilt: `1` = `SELECT`, `2` = `UPDATE`, `3` = `INSERT`, `4` = `DELETE` |
| `ev_enabled` `char` | Steuert, in welchen Modi von `session_replication_role` die Regel ausgelöst wird. `O` = Regel wird in den Modi "origin" und "local" ausgelöst, `D` = Regel ist deaktiviert, `R` = Regel wird im Modus "replica" ausgelöst, `A` = Regel wird immer ausgelöst. |
| `is_instead` `bool` | Wahr, wenn die Regel eine `INSTEAD`-Regel ist |
| `ev_qual` `pg_node_tree` | Ausdrucksbaum (in Form einer `nodeToString()`-Darstellung) für die qualifizierende Bedingung der Regel |
| `ev_action` `pg_node_tree` | Query-Baum (in Form einer `nodeToString()`-Darstellung) für die Aktion der Regel |

> **Hinweis**
>
> `pg_class.relhasrules` muss wahr sein, wenn eine Tabelle Regeln in diesem Katalog hat.

## 52.46. pg_seclabel

Der Katalog `pg_seclabel` speichert Security Labels auf Datenbankobjekten. Security Labels können mit dem Befehl `SECURITY LABEL` bearbeitet werden. Eine einfachere Möglichkeit, Security Labels anzuzeigen, finden Sie in [Abschnitt 53.23](53_System_Views.md#5323-pgseclabels).

Siehe auch `pg_shseclabel`, das eine ähnliche Funktion für Security Labels von Datenbankobjekten erfüllt, die über einen Datenbankcluster hinweg gemeinsam genutzt werden.

**Tabelle 52.46. Spalten von `pg_seclabel`**

| Spalte | Beschreibung |
| --- | --- |
| `objoid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des Objekts, zu dem dieses Security Label gehört |
| `classoid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem dieses Objekt erscheint |
| `objsubid` `int4` | Bei einem Security Label auf einer Tabellenspalte ist dies die Spaltennummer (`objoid` und `classoid` referenzieren die Tabelle selbst). Für alle anderen Objekttypen ist diese Spalte null. |
| `provider` `text` | Der mit diesem Label verknüpfte Label-Provider. |
| `label` `text` | Das auf dieses Objekt angewendete Security Label. |

## 52.47. pg_sequence

Der Katalog `pg_sequence` enthält Informationen über Sequenzen. Einige Informationen über Sequenzen, etwa Name und Schema, stehen in `pg_class`.

**Tabelle 52.47. Spalten von `pg_sequence`**

| Spalte | Beschreibung |
| --- | --- |
| `seqrelid` `oid` (referenziert `pg_class.oid`) | Die OID des `pg_class`-Eintrags für diese Sequenz |
| `seqtypid` `oid` (referenziert `pg_type.oid`) | Datentyp der Sequenz |
| `seqstart` `int8` | Startwert der Sequenz |
| `seqincrement` `int8` | Inkrementwert der Sequenz |
| `seqmax` `int8` | Maximalwert der Sequenz |
| `seqmin` `int8` | Minimalwert der Sequenz |
| `seqcache` `int8` | Cache-Größe der Sequenz |
| `seqcycle` `bool` | Ob die Sequenz zyklisch ist |

## 52.48. pg_shdepend

Der Katalog `pg_shdepend` zeichnet Abhängigkeitsbeziehungen zwischen Datenbankobjekten und gemeinsam genutzten Objekten wie Rollen auf. Diese Informationen ermöglichen PostgreSQL sicherzustellen, dass diese Objekte nicht mehr referenziert werden, bevor versucht wird, sie zu löschen.

Siehe auch `pg_depend`, das eine ähnliche Funktion für Abhängigkeiten erfüllt, an denen Objekte innerhalb einer einzelnen Datenbank beteiligt sind.

Anders als die meisten Systemkataloge wird `pg_shdepend` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_shdepend` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.48. Spalten von `pg_shdepend`**

| Spalte | Beschreibung |
| --- | --- |
| `dbid` `oid` (referenziert `pg_database.oid`) | Die OID der Datenbank, in der sich das abhängige Objekt befindet, oder null für ein gemeinsam genutztes Objekt |
| `classid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem sich das abhängige Objekt befindet |
| `objid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des konkreten abhängigen Objekts |
| `objsubid` `int4` | Bei einer Tabellenspalte ist dies die Spaltennummer (`objid` und `classid` referenzieren die Tabelle selbst). Für alle anderen Objekttypen ist diese Spalte null. |
| `refclassid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem sich das referenzierte Objekt befindet (muss ein gemeinsam genutzter Katalog sein) |
| `refobjid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des konkreten referenzierten Objekts |
| `deptype` `char` | Ein Code, der die konkrete Semantik dieser Abhängigkeitsbeziehung definiert; siehe Text |

In allen Fällen zeigt ein `pg_shdepend`-Eintrag an, dass das referenzierte Objekt nicht gelöscht werden kann, ohne auch das abhängige Objekt zu löschen. Es gibt jedoch mehrere Unterarten, die durch `deptype` gekennzeichnet werden:

`SHARED_DEPENDENCY_OWNER` (`o`)

Das referenzierte Objekt (das eine Rolle sein muss) ist der Eigentümer des abhängigen Objekts.

`SHARED_DEPENDENCY_ACL` (`a`)

Das referenzierte Objekt (das eine Rolle sein muss) wird in der ACL des abhängigen Objekts erwähnt. (Für den Eigentümer des Objekts wird kein `SHARED_DEPENDENCY_ACL`-Eintrag angelegt, da der Eigentümer ohnehin einen `SHARED_DEPENDENCY_OWNER`-Eintrag hat.)

`SHARED_DEPENDENCY_INITACL` (`i`)

Das referenzierte Objekt (das eine Rolle sein muss) wird in einem `pg_init_privs`-Eintrag für das abhängige Objekt erwähnt.

`SHARED_DEPENDENCY_POLICY` (`r`)

Das referenzierte Objekt (das eine Rolle sein muss) wird als Ziel eines abhängigen Policy-Objekts erwähnt.

`SHARED_DEPENDENCY_TABLESPACE` (`t`)

Das referenzierte Objekt (das ein Tablespace sein muss) wird als Tablespace für eine Relation erwähnt, die keinen Speicher besitzt.

Weitere Abhängigkeitsarten könnten in Zukunft benötigt werden. Beachten Sie insbesondere, dass die aktuelle Definition nur Rollen und Tablespaces als referenzierte Objekte unterstützt.

Wie im Katalog `pg_depend` gelten die meisten während `initdb` erstellten Objekte als "pinned". In `pg_shdepend` werden keine Einträge angelegt, bei denen ein gepinntes Objekt entweder referenziertes oder abhängiges Objekt wäre.

## 52.49. pg_shdescription

Der Katalog `pg_shdescription` speichert optionale Beschreibungen (Kommentare) für gemeinsam genutzte Datenbankobjekte. Beschreibungen können mit dem Befehl `COMMENT` bearbeitet und mit den `\d`-Befehlen von `psql` angezeigt werden.

Siehe auch `pg_description`, das eine ähnliche Funktion für Beschreibungen erfüllt, an denen Objekte innerhalb einer einzelnen Datenbank beteiligt sind.

Anders als die meisten Systemkataloge wird `pg_shdescription` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_shdescription` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.49. Spalten von `pg_shdescription`**

| Spalte | Beschreibung |
| --- | --- |
| `objoid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des Objekts, zu dem diese Beschreibung gehört |
| `classoid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem dieses Objekt erscheint |
| `description` `text` | Beliebiger Text, der als Beschreibung dieses Objekts dient |

## 52.50. pg_shseclabel

Der Katalog `pg_shseclabel` speichert Security Labels auf gemeinsam genutzten Datenbankobjekten. Security Labels können mit dem Befehl `SECURITY LABEL` bearbeitet werden. Eine einfachere Möglichkeit, Security Labels anzuzeigen, finden Sie in [Abschnitt 53.23](53_System_Views.md#5323-pgseclabels).

Siehe auch `pg_seclabel`, das eine ähnliche Funktion für Security Labels erfüllt, an denen Objekte innerhalb einer einzelnen Datenbank beteiligt sind.

Anders als die meisten Systemkataloge wird `pg_shseclabel` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_shseclabel` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.50. Spalten von `pg_shseclabel`**

| Spalte | Beschreibung |
| --- | --- |
| `objoid` `oid` (referenziert eine beliebige OID-Spalte) | Die OID des Objekts, zu dem dieses Security Label gehört |
| `classoid` `oid` (referenziert `pg_class.oid`) | Die OID des Systemkatalogs, in dem dieses Objekt erscheint |
| `provider` `text` | Der mit diesem Label verknüpfte Label-Provider. |
| `label` `text` | Das auf dieses Objekt angewendete Security Label. |

## 52.51. pg_statistic

Der Katalog `pg_statistic` speichert statistische Daten über den Inhalt der Datenbank. Einträge werden durch `ANALYZE` erstellt und anschließend vom Query Planner verwendet. Beachten Sie, dass alle statistischen Daten grundsätzlich Näherungen sind, selbst wenn sie aktuell sind.

Normalerweise gibt es für jede analysierte Tabellenspalte einen Eintrag mit `stainherit = false`. Wenn die Tabelle Vererbungsuntertabellen oder Partitionen hat, wird zusätzlich ein zweiter Eintrag mit `stainherit = true` erstellt. Diese Zeile stellt die Statistiken der Spalte über den Vererbungsbaum dar, also Statistiken für die Daten, die Sie mit `SELECT column FROM table*` sehen würden, während die Zeile mit `stainherit = false` die Ergebnisse von `SELECT column FROM ONLY table` darstellt.

`pg_statistic` speichert auch statistische Daten über die Werte von Indexausdrücken. Diese werden so beschrieben, als wären sie tatsächliche Datenspalten; insbesondere referenziert `starelid` den Index. Für eine gewöhnliche Nicht-Ausdrucks-Indexspalte wird jedoch kein Eintrag angelegt, da er redundant zum Eintrag für die zugrunde liegende Tabellenspalte wäre. Derzeit haben Einträge für Indexausdrücke immer `stainherit = false`.

Da für verschiedene Arten von Daten verschiedene Arten von Statistiken geeignet sein können, ist `pg_statistic` so entworfen, dass es nur wenige Annahmen darüber trifft, welche Art von Statistiken es speichert. Nur sehr allgemeine Statistiken (etwa Nullwerte) erhalten eigene Spalten in `pg_statistic`. Alles andere wird in "Slots" gespeichert, also Gruppen zugehöriger Spalten, deren Inhalt durch eine Codenummer in einer der Slot-Spalten identifiziert wird. Weitere Informationen finden Sie in `src/include/catalog/pg_statistic.h`.

`pg_statistic` sollte nicht öffentlich lesbar sein, da selbst statistische Informationen über Tabelleninhalte als sensibel gelten können. (Beispiel: Minimal- und Maximalwerte einer Gehaltsspalte könnten durchaus interessant sein.) `pg_stats` ist eine öffentlich lesbare Sicht auf `pg_statistic`, die nur Informationen über Tabellen offenlegt, die für den aktuellen Benutzer lesbar sind.

**Tabelle 52.51. Spalten von `pg_statistic`**

| Spalte | Beschreibung |
| --- | --- |
| `starelid` `oid` (referenziert `pg_class.oid`) | Die Tabelle oder der Index, zu dem die beschriebene Spalte gehört |
| `staattnum` `int2` (referenziert `pg_attribute.attnum`) | Die Nummer der beschriebenen Spalte |
| `stainherit` `bool` | Wenn wahr, enthalten die Statistiken Werte aus untergeordneten Tabellen, nicht nur die Werte in der angegebenen Relation |
| `stanullfrac` `float4` | Der Anteil der Spalteneinträge, die null sind |
| `stawidth` `int4` | Die durchschnittliche gespeicherte Breite nicht-nullwertiger Einträge in Bytes |
| `stadistinct` `float4` | Die Anzahl unterschiedlicher nicht-nullwertiger Datenwerte in der Spalte. Ein Wert größer als null ist die tatsächliche Anzahl unterschiedlicher Werte. Ein Wert kleiner als null ist das Negative eines Multiplikators für die Anzahl der Zeilen in der Tabelle; eine Spalte, in der etwa 80% der Werte nicht null sind und jeder nicht-nullwertige Wert im Durchschnitt etwa zweimal vorkommt, könnte beispielsweise durch `stadistinct = -0.4` dargestellt werden. Ein Wert von null bedeutet, dass die Anzahl unterschiedlicher Werte unbekannt ist. |
| `stakindN` `int2` | Eine Codenummer, die die Art der Statistik angibt, die im N-ten "Slot" der `pg_statistic`-Zeile gespeichert ist. |
| `staopN` `oid` (referenziert `pg_operator.oid`) | Ein Operator, der verwendet wird, um die im N-ten "Slot" gespeicherte Statistik abzuleiten. Ein Histogramm-Slot würde zum Beispiel den `<`-Operator zeigen, der die Sortierreihenfolge der Daten definiert. Null, wenn die Statistikart keinen Operator benötigt. |
| `stacollN` `oid` (referenziert `pg_collation.oid`) | Die Collation, die verwendet wird, um die im N-ten "Slot" gespeicherte Statistik abzuleiten. Ein Histogramm-Slot für eine collatable Spalte würde zum Beispiel die Collation zeigen, die die Sortierreihenfolge der Daten definiert. Null für nicht-collatable Daten. |
| `stanumbersN` `float4[]` | Numerische Statistiken der passenden Art für den N-ten "Slot" oder null, wenn die Slot-Art keine numerischen Werte umfasst |
| `stavaluesN` `anyarray` | Spaltendatenwerte der passenden Art für den N-ten "Slot" oder null, wenn die Slot-Art keine Datenwerte speichert. Die Elementwerte jedes Arrays haben tatsächlich den spezifischen Datentyp der Spalte oder einen verwandten Typ wie den Elementtyp eines Arrays; daher gibt es keine Möglichkeit, den Typ dieser Spalten genauer als `anyarray` zu definieren. |

## 52.52. pg_statistic_ext

Der Katalog `pg_statistic_ext` enthält Definitionen erweiterter Planner-Statistiken. Jede Zeile in diesem Katalog entspricht einem Statistikobjekt, das mit `CREATE STATISTICS` erstellt wurde.

**Tabelle 52.52. Spalten von `pg_statistic_ext`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `stxrelid` `oid` (referenziert `pg_class.oid`) | Tabelle, die die durch dieses Objekt beschriebenen Spalten enthält |
| `stxname` `name` | Name des Statistikobjekts |
| `stxnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der dieses Statistikobjekt enthält |
| `stxowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Statistikobjekts |
| `stxkeys` `int2vector` (referenziert `pg_attribute.attnum`) | Ein Array von Attributnummern, das angibt, welche Tabellenspalten von diesem Statistikobjekt abgedeckt werden; ein Wert von `1 3` würde zum Beispiel bedeuten, dass die erste und die dritte Tabellenspalte abgedeckt sind |
| `stxstattarget` `int2` | `stxstattarget` steuert den Detailgrad der Statistiken, die `ANALYZE` für dieses Statistikobjekt sammelt. Ein Wert von null zeigt an, dass keine Statistiken gesammelt werden sollen. Ein Nullwert bedeutet, dass das Maximum der Statistikziele der referenzierten Spalten, sofern gesetzt, oder das systemweite Default-Statistikziel verwendet werden soll. Positive Werte von `stxstattarget` bestimmen die Zielanzahl der zu sammelnden "most common values". |
| `stxkind` `char[]` | Ein Array mit Codes für die aktivierten Statistikarten; gültige Werte sind: `d` für n-distinct-Statistiken, `f` für Statistiken funktionaler Abhängigkeiten, `m` für Most-Common-Values-(MCV)-Listenstatistiken und `e` für Ausdrucksstatistiken |
| `stxexprs` `pg_node_tree` | Ausdrucksbäume (in `nodeToString()`-Darstellung) für Attribute des Statistikobjekts, die keine einfachen Spaltenreferenzen sind. Dies ist eine Liste mit einem Element pro Ausdruck. Null, wenn alle Attribute des Statistikobjekts einfache Referenzen sind. |

Der `pg_statistic_ext`-Eintrag wird während `CREATE STATISTICS` vollständig ausgefüllt, aber die tatsächlichen statistischen Werte werden zu diesem Zeitpunkt nicht berechnet. Nachfolgende `ANALYZE`-Befehle berechnen die gewünschten Werte und füllen einen Eintrag im Katalog `pg_statistic_ext_data`.

## 52.53. pg_statistic_ext_data

Der Katalog `pg_statistic_ext_data` enthält Daten für erweiterte Planner-Statistiken, die in `pg_statistic_ext` definiert sind. Jede Zeile in diesem Katalog entspricht einem Statistikobjekt, das mit `CREATE STATISTICS` erstellt wurde.

Normalerweise gibt es für jedes analysierte Statistikobjekt einen Eintrag mit `stxdinherit = false`. Wenn die Tabelle Vererbungsuntertabellen oder Partitionen hat, wird zusätzlich ein zweiter Eintrag mit `stxdinherit = true` erstellt. Diese Zeile stellt das Statistikobjekt über den Vererbungsbaum dar, also Statistiken für die Daten, die Sie mit `SELECT * FROM table*` sehen würden, während die Zeile mit `stxdinherit = false` die Ergebnisse von `SELECT * FROM ONLY table` darstellt.

Wie `pg_statistic` sollte auch `pg_statistic_ext_data` nicht öffentlich lesbar sein, da die Inhalte als sensibel gelten können. (Beispiel: häufigste Kombinationen von Werten in Spalten könnten durchaus interessant sein.) `pg_stats_ext` ist eine öffentlich lesbare Sicht auf `pg_statistic_ext_data` (nach Join mit `pg_statistic_ext`), die nur Informationen über Tabellen offenlegt, deren Eigentümer der aktuelle Benutzer ist.

**Tabelle 52.53. Spalten von `pg_statistic_ext_data`**

| Spalte | Beschreibung |
| --- | --- |
| `stxoid` `oid` (referenziert `pg_statistic_ext.oid`) | Erweitertes Statistikobjekt, das die Definition für diese Daten enthält |
| `stxdinherit` `bool` | Wenn wahr, enthalten die Statistiken Werte aus untergeordneten Tabellen, nicht nur die Werte in der angegebenen Relation |
| `stxdndistinct` `pg_ndistinct` | N-distinct-Zählungen, serialisiert als Typ `pg_ndistinct` |
| `stxddependencies` `pg_dependencies` | Statistiken funktionaler Abhängigkeiten, serialisiert als Typ `pg_dependencies` |
| `stxdmcv` `pg_mcv_list` | MCV-(most-common values)-Listenstatistiken, serialisiert als Typ `pg_mcv_list` |
| `stxdexpr` `pg_statistic[]` | Statistiken pro Ausdruck, serialisiert als Array des Typs `pg_statistic` |

## 52.54. pg_subscription

Der Katalog `pg_subscription` enthält alle vorhandenen Subscriptions für logische Replikation. Weitere Informationen zur logischen Replikation finden Sie in [Kapitel 29](29_Logische_Replikation.md).

Anders als die meisten Systemkataloge wird `pg_subscription` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_subscription` pro Cluster, nicht eine pro Datenbank.

Der Zugriff auf die Spalte `subconninfo` ist normalen Benutzern entzogen, weil sie Klartextpasswörter enthalten könnte.

**Tabelle 52.54. Spalten von `pg_subscription`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `subdbid` `oid` (referenziert `pg_database.oid`) | OID der Datenbank, in der sich die Subscription befindet |
| `subskiplsn` `pg_lsn` | Abschluss-LSN der Transaktion, deren Änderungen übersprungen werden sollen, wenn es eine gültige LSN ist; andernfalls `0/0`. |
| `subname` `name` | Name der Subscription |
| `subowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Subscription |
| `subenabled` `bool` | Wenn wahr, ist die Subscription aktiviert und sollte replizieren |
| `subbinary` `bool` | Wenn wahr, fordert die Subscription an, dass der Publisher Daten im Binärformat sendet |
| `substream` `char` | Steuert, wie Streaming laufender Transaktionen behandelt wird: `f` = Streaming laufender Transaktionen verbieten, `t` = Änderungen laufender Transaktionen auf die Platte auslagern und nach dem Commit auf dem Publisher und dem Empfang beim Subscriber auf einmal anwenden, `p` = Änderungen direkt mit einem parallelen Apply Worker anwenden, falls verfügbar (entspricht `t`, wenn kein Worker verfügbar ist) |
| `subtwophasestate` `char` | Statuscodes für den Two-Phase-Modus: `d` = deaktiviert, `p` = Aktivierung ausstehend, `e` = aktiviert |
| `subdisableonerr` `bool` | Wenn wahr, wird die Subscription deaktiviert, wenn einer ihrer Worker einen Fehler erkennt |
| `subpasswordrequired` `bool` | Wenn wahr, muss die Subscription ein Passwort zur Authentifizierung angeben |
| `subrunasowner` `bool` | Wenn wahr, wird die Subscription mit den Berechtigungen des Subscription-Eigentümers ausgeführt |
| `subfailover` `bool` | Wenn wahr, können die zugehörigen Replikations-Slots (d. h. der Haupt-Slot und die Tabellensynchronisations-Slots) in der Upstream-Datenbank mit den Standbys synchronisiert werden |
| `subconninfo` `text` | Verbindungszeichenkette zur Upstream-Datenbank |
| `subslotname` `name` | Name des Replikations-Slots in der Upstream-Datenbank (auch für den lokalen Replication-Origin-Namen verwendet); null steht für `NONE` |
| `subsynccommit` `text` | Die Einstellung `synchronous_commit`, die von den Workern der Subscription verwendet wird |
| `subpublications` `text[]` | Array der abonnierten Publication-Namen. Diese referenzieren Publications, die in der Upstream-Datenbank definiert sind. Weitere Informationen zu Publications finden Sie in [Abschnitt 29.1](29_Logische_Replikation.md#291-publication). |
| `suborigin` `text` | Der Origin-Wert muss entweder `none` oder `any` sein. Der Default ist `any`. Bei `none` fordert die Subscription vom Publisher an, nur Änderungen zu senden, die keinen Origin haben. Bei `any` sendet der Publisher Änderungen unabhängig von ihrem Origin. |

## 52.55. pg_subscription_rel

Der Katalog `pg_subscription_rel` enthält den Status jeder replizierten Relation in jeder Subscription. Dies ist eine Many-to-Many-Zuordnung.

Dieser Katalog enthält nur Tabellen, die der Subscription nach Ausführung von `CREATE SUBSCRIPTION` oder `ALTER SUBSCRIPTION ... REFRESH PUBLICATION` bekannt sind.

**Tabelle 52.55. Spalten von `pg_subscription_rel`**

| Spalte | Beschreibung |
| --- | --- |
| `srsubid` `oid` (referenziert `pg_subscription.oid`) | Referenz auf die Subscription |
| `srrelid` `oid` (referenziert `pg_class.oid`) | Referenz auf die Relation |
| `srsubstate` `char` | Statuscode: `i` = initialize, `d` = Daten werden kopiert, `f` = Tabellenkopie abgeschlossen, `s` = synchronisiert, `r` = bereit (normale Replikation) |
| `srsublsn` `pg_lsn` | Remote-LSN der Statusänderung, die zur Synchronisationskoordination verwendet wird, wenn der Status `s` oder `r` ist; andernfalls null |

## 52.56. pg_tablespace

Der Katalog `pg_tablespace` speichert Informationen über die verfügbaren Tablespaces. Tabellen können in bestimmten Tablespaces abgelegt werden, um die Verwaltung des Plattenlayouts zu unterstützen.

Anders als die meisten Systemkataloge wird `pg_tablespace` von allen Datenbanken eines Clusters gemeinsam genutzt: Es gibt nur eine Kopie von `pg_tablespace` pro Cluster, nicht eine pro Datenbank.

**Tabelle 52.56. Spalten von `pg_tablespace`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `spcname` `name` | Tablespace-Name |
| `spcowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Tablespaces, normalerweise der Benutzer, der ihn erstellt hat |
| `spcacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |
| `spcoptions` `text[]` | Optionen auf Tablespace-Ebene als Zeichenketten der Form `keyword=value` |

## 52.57. pg_transform

Der Katalog `pg_transform` speichert Informationen über Transforms, einen Mechanismus zur Anpassung von Datentypen an prozedurale Sprachen. Weitere Informationen finden Sie unter `CREATE TRANSFORM`.

**Tabelle 52.57. Spalten von `pg_transform`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `trftype` `oid` (referenziert `pg_type.oid`) | OID des Datentyps, für den dieser Transform gilt |
| `trflang` `oid` (referenziert `pg_language.oid`) | OID der Sprache, für die dieser Transform gilt |
| `trffromsql` `regproc` (referenziert `pg_proc.oid`) | Die OID der Funktion, die beim Umwandeln des Datentyps für die Eingabe in die prozedurale Sprache (z. B. Funktionsparameter) verwendet wird. Null wird gespeichert, wenn das Default-Verhalten verwendet werden soll. |
| `trftosql` `regproc` (referenziert `pg_proc.oid`) | Die OID der Funktion, die beim Umwandeln der Ausgabe aus der prozeduralen Sprache (z. B. Rückgabewerte) in den Datentyp verwendet wird. Null wird gespeichert, wenn das Default-Verhalten verwendet werden soll. |

## 52.58. pg_trigger

Der Katalog `pg_trigger` speichert Trigger auf Tabellen und Sichten. Weitere Informationen finden Sie unter `CREATE TRIGGER`.

**Tabelle 52.58. Spalten von `pg_trigger`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `tgrelid` `oid` (referenziert `pg_class.oid`) | Die Tabelle, auf der dieser Trigger liegt |
| `tgparentid` `oid` (referenziert `pg_trigger.oid`) | Parent-Trigger, von dem dieser Trigger geklont wurde (dies geschieht, wenn Partitionen erstellt oder an eine partitionierte Tabelle angehängt werden); null, wenn es kein Klon ist |
| `tgname` `name` | Triggername (muss unter den Triggern derselben Tabelle eindeutig sein) |
| `tgfoid` `oid` (referenziert `pg_proc.oid`) | Die aufzurufende Funktion |
| `tgtype` `int2` | Bitmaske, die die Auslösebedingungen des Triggers identifiziert |
| `tgenabled` `char` | Steuert, in welchen Modi von `session_replication_role` der Trigger ausgelöst wird. `O` = Trigger wird in den Modi "origin" und "local" ausgelöst, `D` = Trigger ist deaktiviert, `R` = Trigger wird im Modus "replica" ausgelöst, `A` = Trigger wird immer ausgelöst. |
| `tgisinternal` `bool` | Wahr, wenn der Trigger intern erzeugt wurde (üblicherweise, um den durch `tgconstraint` identifizierten Constraint durchzusetzen) |
| `tgconstrrelid` `oid` (referenziert `pg_class.oid`) | Die Tabelle, auf die ein referenzieller Integritäts-Constraint verweist (null, wenn der Trigger nicht zu einem referenziellen Integritäts-Constraint gehört) |
| `tgconstrindid` `oid` (referenziert `pg_class.oid`) | Der Index, der einen Unique-, Primary-Key-, referenziellen Integritäts- oder Exclusion-Constraint unterstützt (null, wenn der Trigger nicht zu einem dieser Constraint-Typen gehört) |
| `tgconstraint` `oid` (referenziert `pg_constraint.oid`) | Der mit dem Trigger verknüpfte `pg_constraint`-Eintrag (null, wenn der Trigger nicht zu einem Constraint gehört) |
| `tgdeferrable` `bool` | Wahr, wenn der Constraint-Trigger deferrable ist |
| `tginitdeferred` `bool` | Wahr, wenn der Constraint-Trigger anfänglich deferred ist |
| `tgnargs` `int2` | Anzahl der Argumentzeichenketten, die an die Trigger-Funktion übergeben werden |
| `tgattr` `int2vector` (referenziert `pg_attribute.attnum`) | Spaltennummern, wenn der Trigger spaltenspezifisch ist; andernfalls ein leeres Array |
| `tgargs` `bytea` | Argumentzeichenketten, die an den Trigger übergeben werden, jeweils `NULL`-terminiert |
| `tgqual` `pg_node_tree` | Ausdrucksbaum (in `nodeToString()`-Darstellung) für die `WHEN`-Bedingung des Triggers oder null, wenn keine vorhanden ist |
| `tgoldtable` `name` | Name der `REFERENCING`-Klausel für `OLD TABLE` oder null, wenn keiner vorhanden ist |
| `tgnewtable` `name` | Name der `REFERENCING`-Klausel für `NEW TABLE` oder null, wenn keiner vorhanden ist |

Derzeit wird spaltenspezifisches Triggern nur für `UPDATE`-Ereignisse unterstützt; daher ist `tgattr` nur für diesen Ereignistyp relevant. `tgtype` kann auch Bits für andere Ereignistypen enthalten, diese gelten jedoch unabhängig vom Inhalt von `tgattr` als tabellenweit.

> **Hinweis**
>
> Wenn `tgconstraint` ungleich null ist, sind `tgconstrrelid`, `tgconstrindid`, `tgdeferrable` und `tginitdeferred` weitgehend redundant mit dem referenzierten `pg_constraint`-Eintrag. Es ist jedoch möglich, dass ein nicht deferrable Trigger mit einem deferrable Constraint verknüpft ist: Foreign-Key-Constraints können einige deferrable und einige nicht deferrable Trigger haben.

> **Hinweis**
>
> `pg_class.relhastriggers` muss wahr sein, wenn eine Relation Trigger in diesem Katalog hat.

## 52.59. pg_ts_config

Der Katalog `pg_ts_config` enthält Einträge, die Textsuche-Konfigurationen darstellen. Eine Konfiguration gibt einen bestimmten Textsuche-Parser und eine Liste von Wörterbüchern an, die für jeden Ausgabetokentyp des Parsers verwendet werden sollen. Der Parser steht im `pg_ts_config`-Eintrag, die Token-zu-Wörterbuch-Zuordnung wird jedoch durch untergeordnete Einträge in `pg_ts_config_map` definiert.

Die Textsuche-Funktionen von PostgreSQL werden ausführlich in [Kapitel 12](12_Volltextsuche.md) beschrieben.

**Tabelle 52.59. Spalten von `pg_ts_config`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `cfgname` `name` | Name der Textsuche-Konfiguration |
| `cfgnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diese Konfiguration enthält |
| `cfgowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer der Konfiguration |
| `cfgparser` `oid` (referenziert `pg_ts_parser.oid`) | Die OID des Textsuche-Parsers für diese Konfiguration |

## 52.60. pg_ts_config_map

Der Katalog `pg_ts_config_map` enthält Einträge, die zeigen, welche Textsuche-Wörterbücher in welcher Reihenfolge für jeden Ausgabetokentyp des Parsers jeder Textsuche-Konfiguration herangezogen werden sollen.

Die Textsuche-Funktionen von PostgreSQL werden ausführlich in [Kapitel 12](12_Volltextsuche.md) beschrieben.

**Tabelle 52.60. Spalten von `pg_ts_config_map`**

| Spalte | Beschreibung |
| --- | --- |
| `mapcfg` `oid` (referenziert `pg_ts_config.oid`) | Die OID des `pg_ts_config`-Eintrags, dem dieser Map-Eintrag gehört |
| `maptokentype` `int4` | Ein vom Parser der Konfiguration ausgegebener Tokentyp |
| `mapseqno` `int4` | Reihenfolge, in der dieser Eintrag herangezogen wird (niedrigere `mapseqno` zuerst) |
| `mapdict` `oid` (referenziert `pg_ts_dict.oid`) | Die OID des heranzuziehenden Textsuche-Wörterbuchs |

## 52.61. pg_ts_dict

Der Katalog `pg_ts_dict` enthält Einträge, die Textsuche-Wörterbücher definieren. Ein Wörterbuch hängt von einem Textsuche-Template ab, das alle benötigten Implementierungsfunktionen angibt; das Wörterbuch selbst liefert Werte für die vom Template unterstützten, vom Benutzer setzbaren Parameter. Diese Arbeitsteilung erlaubt es nicht privilegierten Benutzern, Wörterbücher zu erstellen. Die Parameter werden durch die Textzeichenkette `dictinitoption` angegeben, deren Format und Bedeutung vom Template abhängen.

Die Textsuche-Funktionen von PostgreSQL werden ausführlich in [Kapitel 12](12_Volltextsuche.md) beschrieben.

**Tabelle 52.61. Spalten von `pg_ts_dict`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `dictname` `name` | Name des Textsuche-Wörterbuchs |
| `dictnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der dieses Wörterbuch enthält |
| `dictowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Wörterbuchs |
| `dicttemplate` `oid` (referenziert `pg_ts_template.oid`) | Die OID des Textsuche-Templates für dieses Wörterbuch |
| `dictinitoption` `text` | Initialisierungsoptionszeichenkette für das Template |

## 52.62. pg_ts_parser

Der Katalog `pg_ts_parser` enthält Einträge, die Textsuche-Parser definieren. Ein Parser ist dafür verantwortlich, Eingabetext in Lexeme aufzuteilen und jedem Lexem einen Tokentyp zuzuweisen. Da ein Parser durch Funktionen auf C-Sprach-Ebene implementiert werden muss, ist das Erstellen neuer Parser auf Datenbank-Superuser beschränkt.

Die Textsuche-Funktionen von PostgreSQL werden ausführlich in [Kapitel 12](12_Volltextsuche.md) beschrieben.

**Tabelle 52.62. Spalten von `pg_ts_parser`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `prsname` `name` | Name des Textsuche-Parsers |
| `prsnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diesen Parser enthält |
| `prsstart` `regproc` (referenziert `pg_proc.oid`) | OID der Startup-Funktion des Parsers |
| `prstoken` `regproc` (referenziert `pg_proc.oid`) | OID der Next-Token-Funktion des Parsers |
| `prsend` `regproc` (referenziert `pg_proc.oid`) | OID der Shutdown-Funktion des Parsers |
| `prsheadline` `regproc` (referenziert `pg_proc.oid`) | OID der Headline-Funktion des Parsers (null, wenn keine vorhanden ist) |
| `prslextype` `regproc` (referenziert `pg_proc.oid`) | OID der LexType-Funktion des Parsers |

## 52.63. pg_ts_template

Der Katalog `pg_ts_template` enthält Einträge, die Textsuche-Templates definieren. Ein Template ist das Implementierungsskelett für eine Klasse von Textsuche-Wörterbüchern. Da ein Template durch Funktionen auf C-Sprach-Ebene implementiert werden muss, ist das Erstellen neuer Templates auf Datenbank-Superuser beschränkt.

Die Textsuche-Funktionen von PostgreSQL werden ausführlich in [Kapitel 12](12_Volltextsuche.md) beschrieben.

**Tabelle 52.63. Spalten von `pg_ts_template`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `tmplname` `name` | Name des Textsuche-Templates |
| `tmplnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der dieses Template enthält |
| `tmplinit` `regproc` (referenziert `pg_proc.oid`) | OID der Initialisierungsfunktion des Templates (null, wenn keine vorhanden ist) |
| `tmpllexize` `regproc` (referenziert `pg_proc.oid`) | OID der Lexize-Funktion des Templates |

## 52.64. pg_type

Der Katalog `pg_type` speichert Informationen über Datentypen. Basistypen und Enum-Typen (skalare Typen) werden mit `CREATE TYPE` erstellt, Domains mit `CREATE DOMAIN`. Für jede Tabelle in der Datenbank wird automatisch ein zusammengesetzter Typ erstellt, der die Zeilenstruktur der Tabelle darstellt. Es ist auch möglich, zusammengesetzte Typen mit `CREATE TYPE AS` zu erstellen.

**Tabelle 52.64. Spalten von `pg_type`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `typname` `name` | Name des Datentyps |
| `typnamespace` `oid` (referenziert `pg_namespace.oid`) | Die OID des Namensraums, der diesen Typ enthält |
| `typowner` `oid` (referenziert `pg_authid.oid`) | Eigentümer des Typs |
| `typlen` `int2` | Bei einem Typ fester Größe ist `typlen` die Anzahl der Bytes in der internen Darstellung des Typs. Bei einem Typ variabler Länge ist `typlen` negativ. `-1` zeigt einen "varlena"-Typ an (einen Typ mit Längenwort), `-2` eine nullterminierte C-Zeichenkette. |
| `typbyval` `bool` | `typbyval` bestimmt, ob interne Routinen einen Wert dieses Typs by value oder by reference übergeben. `typbyval` sollte falsch sein, wenn `typlen` nicht 1, 2 oder 4 ist (oder 8 auf Maschinen, auf denen `Datum` 8 Bytes groß ist). Typen variabler Länge werden immer by reference übergeben. Beachten Sie, dass `typbyval` falsch sein kann, auch wenn die Länge eine Übergabe by value erlauben würde. |
| `typtype` `char` | `typtype` ist `b` für einen Basistyp, `c` für einen zusammengesetzten Typ (z. B. den Zeilentyp einer Tabelle), `d` für eine Domain, `e` für einen Enum-Typ, `p` für einen Pseudotyp, `r` für einen Range-Typ oder `m` für einen Multirange-Typ. Siehe auch `typrelid` und `typbasetype`. |
| `typcategory` `char` | `typcategory` ist eine beliebige Klassifikation von Datentypen, die der Parser verwendet, um zu bestimmen, welche impliziten Casts "bevorzugt" sein sollen. Siehe Tabelle 52.65. |
| `typispreferred` `bool` | Wahr, wenn der Typ innerhalb seiner `typcategory` ein bevorzugtes Cast-Ziel ist |
| `typisdefined` `bool` | Wahr, wenn der Typ definiert ist; falsch, wenn dies ein Platzhaltereintrag für einen noch nicht definierten Typ ist. Wenn `typisdefined` falsch ist, kann man sich auf nichts außer Typnamen, Namespace und OID verlassen. |
| `typdelim` `char` | Zeichen, das beim Parsen von Array-Eingaben zwei Werte dieses Typs trennt. Beachten Sie, dass das Trennzeichen dem Datentyp des Array-Elements zugeordnet ist, nicht dem Array-Datentyp. |
| `typrelid` `oid` (referenziert `pg_class.oid`) | Wenn dies ein zusammengesetzter Typ ist (siehe `typtype`), zeigt diese Spalte auf den `pg_class`-Eintrag, der die entsprechende Tabelle definiert. (Bei einem eigenständigen zusammengesetzten Typ stellt der `pg_class`-Eintrag nicht wirklich eine Tabelle dar, wird aber dennoch benötigt, damit die `pg_attribute`-Einträge des Typs verknüpft werden können.) Null für nicht zusammengesetzte Typen. |
| `typsubscript` `regproc` (referenziert `pg_proc.oid`) | OID der Subscripting-Handler-Funktion oder null, wenn dieser Typ kein Subscripting unterstützt. Typen, die "echte" Array-Typen sind, haben `typsubscript = array_subscript_handler`, aber andere Typen können andere Handler-Funktionen haben, um spezielles Subscripting-Verhalten zu implementieren. |
| `typelem` `oid` (referenziert `pg_type.oid`) | Wenn `typelem` nicht null ist, identifiziert es eine andere Zeile in `pg_type`, die den durch Subscripting gelieferten Typ definiert. Dies sollte null sein, wenn `typsubscript` null ist. Es kann jedoch null sein, obwohl `typsubscript` nicht null ist, wenn der Handler `typelem` nicht benötigt, um den Ergebnistyp des Subscripting zu bestimmen. Beachten Sie, dass eine `typelem`-Abhängigkeit als physische Enthaltung des Elementtyps in diesem Typ verstanden wird; daher können DDL-Änderungen am Elementtyp durch das Vorhandensein dieses Typs eingeschränkt werden. |
| `typarray` `oid` (referenziert `pg_type.oid`) | Wenn `typarray` nicht null ist, identifiziert es eine andere Zeile in `pg_type`, die der "echte" Array-Typ mit diesem Typ als Element ist |
| `typinput` `regproc` (referenziert `pg_proc.oid`) | Eingabekonvertierungsfunktion (Textformat) |
| `typoutput` `regproc` (referenziert `pg_proc.oid`) | Ausgabekonvertierungsfunktion (Textformat) |
| `typreceive` `regproc` (referenziert `pg_proc.oid`) | Eingabekonvertierungsfunktion (Binärformat) oder null, wenn keine vorhanden ist |
| `typsend` `regproc` (referenziert `pg_proc.oid`) | Ausgabekonvertierungsfunktion (Binärformat) oder null, wenn keine vorhanden ist |
| `typmodin` `regproc` (referenziert `pg_proc.oid`) | Eingabefunktion für Typmodifikatoren oder null, wenn der Typ keine Modifikatoren unterstützt |
| `typmodout` `regproc` (referenziert `pg_proc.oid`) | Ausgabefunktion für Typmodifikatoren oder null, um das Standardformat zu verwenden |
| `typanalyze` `regproc` (referenziert `pg_proc.oid`) | Benutzerdefinierte `ANALYZE`-Funktion oder null, um die Standardfunktion zu verwenden |
| `typalign` `char` | `typalign` ist die beim Speichern eines Werts dieses Typs erforderliche Ausrichtung. Sie gilt für die Speicherung auf dem Datenträger ebenso wie für die meisten Darstellungen des Werts innerhalb von PostgreSQL. Wenn mehrere Werte hintereinander gespeichert werden, etwa in der Darstellung einer vollständigen Zeile auf dem Datenträger, wird vor einem Datum dieses Typs Padding eingefügt, damit es an der angegebenen Grenze beginnt. Bezugspunkt der Ausrichtung ist der Beginn des ersten Datums in der Sequenz. Mögliche Werte sind: `c` = char alignment, also keine Ausrichtung nötig; `s` = short alignment (2 Bytes auf den meisten Maschinen); `i` = int alignment (4 Bytes auf den meisten Maschinen); `d` = double alignment (8 Bytes auf vielen Maschinen, aber keineswegs allen). |
| `typstorage` `char` | `typstorage` gibt für varlena-Typen (solche mit `typlen = -1`) an, ob der Typ für TOAST vorbereitet ist und welche Default-Strategie für Attribute dieses Typs gelten soll. Mögliche Werte sind: `p` (plain): Werte müssen immer plain gespeichert werden (Nicht-varlena-Typen verwenden immer diesen Wert); `e` (external): Werte können in einer sekundären "TOAST"-Relation gespeichert werden (falls die Relation eine hat, siehe `pg_class.reltoastrelid`); `m` (main): Werte können komprimiert und inline gespeichert werden; `x` (extended): Werte können komprimiert und/oder in eine sekundäre Relation verschoben werden. `x` ist die übliche Wahl für toastbare Typen. Beachten Sie, dass `m`-Werte ebenfalls in sekundären Speicher verschoben werden können, aber nur als letzte Möglichkeit (`e`- und `x`-Werte werden zuerst verschoben). |
| `typnotnull` `bool` | `typnotnull` stellt einen Not-Null-Constraint auf einem Typ dar. Wird nur für Domains verwendet. |
| `typbasetype` `oid` (referenziert `pg_type.oid`) | Wenn dies eine Domain ist (siehe `typtype`), identifiziert `typbasetype` den Typ, auf dem diese basiert. Null, wenn dieser Typ keine Domain ist. |
| `typtypmod` `int4` | Domains verwenden `typtypmod`, um den Typmodifikator aufzuzeichnen, der auf ihren Basistyp angewendet wird (`-1`, wenn der Basistyp keinen Typmodifikator verwendet). `-1`, wenn dieser Typ keine Domain ist. |
| `typndims` `int4` | `typndims` ist die Anzahl der Array-Dimensionen für eine Domain über einem Array (d. h. `typbasetype` ist ein Array-Typ). Null für andere Typen als Domains über Array-Typen. |
| `typcollation` `oid` (referenziert `pg_collation.oid`) | `typcollation` gibt die Collation des Typs an. Wenn der Typ keine Collations unterstützt, ist dies null. Ein Basistyp, der Collations unterstützt, hat hier einen Nicht-Null-Wert, typischerweise `DEFAULT_COLLATION_OID`. Eine Domain über einem collatable Typ kann eine von ihrem Basistyp abweichende Collation-OID haben, wenn für die Domain eine angegeben wurde. |
| `typdefaultbin` `pg_node_tree` | Wenn `typdefaultbin` nicht null ist, enthält es die `nodeToString()`-Darstellung eines Default-Ausdrucks für den Typ. Dies wird nur für Domains verwendet. |
| `typdefault` `text` | `typdefault` ist null, wenn der Typ keinen zugehörigen Default-Wert hat. Wenn `typdefaultbin` nicht null ist, muss `typdefault` eine menschenlesbare Version des durch `typdefaultbin` dargestellten Default-Ausdrucks enthalten. Wenn `typdefaultbin` null ist und `typdefault` nicht null, dann ist `typdefault` die externe Darstellung des Default-Werts des Typs, die an den Eingabekonverter des Typs übergeben werden kann, um eine Konstante zu erzeugen. |
| `typacl` `aclitem[]` | Zugriffsprivilegien; siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien) für Details |

> **Hinweis**
>
> Für Typen fester Breite, die in Systemtabellen verwendet werden, ist es kritisch, dass die in `pg_type` definierte Größe und Ausrichtung damit übereinstimmen, wie der Compiler die Spalte in einer Struktur anordnet, die eine Tabellenzeile darstellt.

Tabelle 52.65 listet die systemdefinierten Werte von `typcategory` auf. Künftige Ergänzungen dieser Liste werden ebenfalls ASCII-Großbuchstaben sein. Alle anderen ASCII-Zeichen sind für benutzerdefinierte Kategorien reserviert.

**Tabelle 52.65. `typcategory`-Codes**

| Code | Kategorie |
| --- | --- |
| `A` | Array-Typen |
| `B` | Boolean-Typen |
| `C` | Zusammengesetzte Typen |
| `D` | Datum-/Zeit-Typen |
| `E` | Enum-Typen |
| `G` | Geometrische Typen |
| `I` | Netzwerkadresstypen |
| `N` | Numerische Typen |
| `P` | Pseudotypen |
| `R` | Range-Typen |
| `S` | Zeichenkettentypen |
| `T` | Zeitspannentypen |
| `U` | Benutzerdefinierte Typen |
| `V` | Bit-String-Typen |
| `X` | Typ `unknown` |
| `Z` | Typen für interne Verwendung |

## 52.65. pg_user_mapping

Der Katalog `pg_user_mapping` speichert die Zuordnungen von lokalen Benutzern zu entfernten Benutzern. Der Zugriff auf diesen Katalog ist für normale Benutzer eingeschränkt; verwenden Sie stattdessen die Sicht `pg_user_mappings`.

**Tabelle 52.66. Spalten von `pg_user_mapping`**

| Spalte | Beschreibung |
| --- | --- |
| `oid` `oid` | Zeilenbezeichner |
| `umuser` `oid` (referenziert `pg_authid.oid`) | OID der lokalen Rolle, die zugeordnet wird, oder null, wenn das User Mapping öffentlich ist |
| `umserver` `oid` (referenziert `pg_foreign_server.oid`) | Die OID des Foreign Servers, der dieses Mapping enthält |
| `umoptions` `text[]` | User-Mapping-spezifische Optionen als Zeichenketten der Form `keyword=value` |
