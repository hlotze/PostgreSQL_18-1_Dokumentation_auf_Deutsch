# 68. Systemkatalog-Deklarationen und Anfangsinhalte

PostgreSQL verwendet viele verschiedene Systemkataloge, um die Existenz und Eigenschaften von Datenbankobjekten wie Tabellen und Funktionen zu verfolgen. Physisch gibt es keinen Unterschied zwischen einem Systemkatalog und einer gewöhnlichen Benutzertabelle, aber der Backend-C-Code kennt Struktur und Eigenschaften jedes Katalogs und kann ihn direkt auf niedriger Ebene manipulieren. Deshalb ist es zum Beispiel nicht ratsam, die Struktur eines Katalogs im laufenden Betrieb zu ändern; das würde Annahmen verletzen, die im C-Code über das Layout der Katalogzeilen eingebaut sind. Die Struktur der Kataloge kann sich jedoch zwischen Hauptversionen ändern.

Die Strukturen der Kataloge werden in besonders formatierten C-Headerdateien im Verzeichnis `src/include/catalog/` des Quellbaums deklariert. Für jeden Katalog gibt es eine Headerdatei, die nach dem Katalog benannt ist, zum Beispiel `pg_class.h` für `pg_class`. Sie definiert die Spalten des Katalogs sowie einige weitere grundlegende Eigenschaften, etwa seine OID.

Viele Kataloge haben Anfangsdaten, die während der Bootstrap-Phase von `initdb` in sie geladen werden müssen, damit das System an einen Punkt kommt, an dem es SQL-Befehle ausführen kann. Beispielsweise muss `pg_class.h` einen Eintrag für sich selbst sowie einen für jeden anderen Systemkatalog und Index enthalten. Diese Anfangsdaten werden in bearbeitbarer Form in Datendateien gehalten, die ebenfalls im Verzeichnis `src/include/catalog/` gespeichert sind. Zum Beispiel beschreibt `pg_proc.dat` alle Anfangszeilen, die in den Katalog `pg_proc` eingefügt werden müssen.

Um die Katalogdateien zu erzeugen und diese Anfangsdaten zu laden, liest ein im Bootstrap-Modus laufendes Backend eine BKI-Datei (Backend Interface), die Befehle und Anfangsdaten enthält. Die in diesem Modus verwendete Datei `postgres.bki` wird beim Bauen einer PostgreSQL-Distribution aus den genannten Header- und Datendateien durch ein Perl-Skript namens `genbki.pl` vorbereitet. Obwohl sie spezifisch für ein bestimmtes PostgreSQL-Release ist, ist `postgres.bki` plattformunabhängig und wird im Unterverzeichnis `share` des Installationsbaums installiert.

`genbki.pl` erzeugt außerdem für jeden Katalog eine abgeleitete Headerdatei, zum Beispiel `pg_class_d.h` für den Katalog `pg_class`. Diese Datei enthält automatisch erzeugte Makrodefinitionen und kann weitere Makros, `enum`-Deklarationen und Ähnliches enthalten, die für C-Code nützlich sind, der einen bestimmten Katalog liest.

Die meisten PostgreSQL-Entwickler müssen sich nicht direkt mit der BKI-Datei beschäftigen, aber fast jede nichttriviale Feature-Erweiterung im Backend erfordert Änderungen an Katalog-Headerdateien und/oder Anfangsdatendateien. Der Rest dieses Kapitels gibt dazu einige Informationen und beschreibt der Vollständigkeit halber das Format der BKI-Datei.

## 68.1. Deklarationsregeln für Systemkataloge

Der zentrale Teil einer Katalog-Headerdatei ist eine C-Strukturdefinition, die das Layout jeder Zeile des Katalogs beschreibt. Sie beginnt mit einem `CATALOG`-Makro, das aus Sicht des C-Compilers nur eine Kurzform für `typedef struct FormData_catalogname` ist. Jedes Feld in der Struktur erzeugt eine Katalogspalte. Felder können mit den in `genbki.h` beschriebenen BKI-Property-Makros annotiert werden, etwa um einen Standardwert für ein Feld zu definieren oder es als nullable oder not nullable zu markieren. Auch die `CATALOG`-Zeile kann mit einigen anderen in `genbki.h` beschriebenen BKI-Property-Makros annotiert werden, um Eigenschaften des Katalogs als Ganzes zu definieren, zum Beispiel ob es eine Shared Relation ist.

Der Systemkatalog-Cache-Code, und allgemein der meiste Code, der Kataloge bearbeitet, nimmt an, dass die Fixed-Length-Teile aller Systemkatalogtupel tatsächlich vorhanden sind, weil er diese C-Strukturdeklaration auf sie abbildet. Deshalb müssen alle Felder variabler Länge und nullable Felder am Ende stehen, und sie dürfen nicht als Strukturfelder angesprochen werden. Wenn Sie zum Beispiel versuchen würden, `pg_type.typrelid` auf `NULL` zu setzen, würde das fehlschlagen, sobald irgendein Code versucht, `typetup->typrelid` zu referenzieren, oder schlimmer noch `typetup->typelem`, weil es nach `typrelid` folgt. Das würde zu zufälligen Fehlern oder sogar zu Segmentation Faults führen.

Als teilweiser Schutz gegen diese Art von Fehler sollten Felder variabler Länge oder nullable Felder dem C-Compiler nicht direkt sichtbar gemacht werden. Das geschieht, indem sie in `#ifdef CATALOG_VARLEN ... #endif` eingeschlossen werden, wobei `CATALOG_VARLEN` ein Symbol ist, das niemals definiert wird. Dadurch wird verhindert, dass C-Code unvorsichtig versucht, auf Felder zuzugreifen, die möglicherweise nicht vorhanden sind oder an einem anderen Offset liegen. Als unabhängiger Schutz gegen das Erzeugen falscher Zeilen wird verlangt, dass alle Spalten, die not nullable sein sollen, in `pg_attribute` entsprechend markiert sind. Der Bootstrap-Code markiert Katalogspalten automatisch als `NOT NULL`, wenn sie Fixed-Width sind und ihnen keine nullable oder Variable-Width-Spalte vorausgeht. Wo diese Regel nicht ausreicht, kann die korrekte Markierung mit den Annotationen `BKI_FORCE_NOT_NULL` und `BKI_FORCE_NULL` erzwungen werden.

Frontend-Code sollte keine `pg_xxx.h`-Katalog-Headerdatei einbinden, weil diese Dateien C-Code enthalten können, der außerhalb des Backends nicht kompiliert. Typischerweise passiert das, weil diese Dateien auch Deklarationen für Funktionen in Dateien unter `src/backend/catalog/` enthalten. Stattdessen kann Frontend-Code den entsprechenden generierten Header `pg_xxx_d.h` einbinden, der OID-`#define`s und andere Daten enthält, die auf Client-Seite nützlich sein können. Wenn Makros oder anderer Code aus einem Katalog-Header für Frontend-Code sichtbar sein sollen, setzen Sie um diesen Abschnitt `#ifdef EXPOSE_TO_CLIENT_CODE ... #endif`, damit `genbki.pl` ihn in den Header `pg_xxx_d.h` kopiert.

Einige wenige Kataloge sind so fundamental, dass sie nicht einmal mit dem BKI-Befehl `create` erzeugt werden können, der für die meisten Kataloge verwendet wird, weil dieser Befehl Informationen in genau diese Kataloge schreiben muss, um den neuen Katalog zu beschreiben. Diese heißen Bootstrap-Kataloge, und die Definition eines solchen Katalogs erfordert viel zusätzliche Arbeit: Sie müssen passende Einträge für sie manuell in den vorgeladenen Inhalten von `pg_class` und `pg_type` vorbereiten, und diese Einträge müssen bei späteren Strukturänderungen des Katalogs aktualisiert werden. Bootstrap-Kataloge brauchen außerdem vorgeladene Einträge in `pg_attribute`, aber glücklicherweise erledigt `genbki.pl` diese Arbeit heutzutage. Vermeiden Sie möglichst, neue Kataloge zu Bootstrap-Katalogen zu machen.

## 68.2. Anfangsdaten von Systemkatalogen

Jeder Katalog, der manuell erstellte Anfangsdaten hat, manche haben keine, besitzt eine entsprechende `.dat`-Datei, die seine Anfangsdaten in einem bearbeitbaren Format enthält.

### 68.2.1. Datendateiformat

Jede `.dat`-Datei enthält Perl-Datenstruktur-Literale, die einfach mit `eval` ausgewertet werden, um eine In-Memory-Datenstruktur zu erzeugen, die aus einem Array von Hash-Referenzen besteht, eine je Katalogzeile. Ein leicht geänderter Ausschnitt aus `pg_database.dat` zeigt die wichtigsten Merkmale:

```perl
[

  # A comment could appear here.
{ oid => '1', oid_symbol => 'Template1DbOid',
  descr => 'database\'s default template',
  datname => 'template1', encoding => 'ENCODING',
  datlocprovider => 'LOCALE_PROVIDER', datistemplate => 't',
  datallowconn => 't', dathasloginevt => 'f', datconnlimit => '-1',
  datfrozenxid => '0',
  datminmxid => '1', dattablespace => 'pg_default', datcollate => 'LC_COLLATE',
  datctype => 'LC_CTYPE', datlocale => 'DATLOCALE', datacl => '_null_' },

]
```

Dabei ist Folgendes zu beachten:

- Das Gesamtlayout der Datei lautet: öffnende eckige Klammer, eine oder mehrere Mengen geschweifter Klammern, von denen jede eine Katalogzeile repräsentiert, schließende eckige Klammer. Nach jeder schließenden geschweiften Klammer steht ein Komma.

- Innerhalb jeder Katalogzeile werden durch Kommas getrennte Paare `key => value` geschrieben. Erlaubte Schlüssel sind die Namen der Katalogspalten plus die Metadatenschlüssel `oid`, `oid_symbol`, `array_type_oid` und `descr`. Die Verwendung von `oid` und `oid_symbol` wird unten in [Abschnitt 68.2.2](68_Systemkatalog_Deklarationen_und_Anfangsinhalte.md#6822-oidzuweisung) beschrieben, `array_type_oid` in [Abschnitt 68.2.4](68_Systemkatalog_Deklarationen_und_Anfangsinhalte.md#6824-automatische-erzeugung-von-arraytypen). `descr` liefert eine Beschreibungszeichenkette für das Objekt, die je nach Fall in `pg_description` oder `pg_shdescription` eingefügt wird. Die Metadatenschlüssel sind optional, aber alle definierten Spalten des Katalogs müssen angegeben werden, außer wenn die `.h`-Datei des Katalogs einen Standardwert für die Spalte festlegt. Im obigen Beispiel wurde das Feld `datdba` weggelassen, weil `pg_database.h` dafür einen passenden Standardwert liefert.

- Alle Werte müssen in einfache Anführungszeichen gesetzt werden. Einfache Anführungszeichen innerhalb eines Werts werden mit einem Backslash maskiert. Backslashes, die Daten sein sollen, können verdoppelt werden, müssen es aber nicht; das folgt den Perl-Regeln für einfache String-Literale. Beachten Sie, dass Backslashes, die als Daten erscheinen, vom Bootstrap-Scanner nach denselben Regeln wie Escape-String-Konstanten behandelt werden (siehe Abschnitt 4.1.2.2); zum Beispiel wird `\t` in ein Tabulatorzeichen umgewandelt. Wenn Sie tatsächlich einen Backslash im endgültigen Wert haben wollen, müssen Sie vier davon schreiben: Perl entfernt zwei, sodass `\\` für den Bootstrap-Scanner übrig bleibt.

- Nullwerte werden durch `_null_` dargestellt. Es gibt keine Möglichkeit, einen Wert zu erzeugen, der genau dieser String ist.

- Kommentare beginnen mit `#` und müssen auf eigenen Zeilen stehen.

- Feldwerte, die OIDs anderer Katalogeinträge sind, sollten durch symbolische Namen statt durch tatsächliche numerische OIDs dargestellt werden. Im obigen Beispiel enthält `dattablespace` eine solche Referenz. Das wird unten in [Abschnitt 68.2.3](68_Systemkatalog_Deklarationen_und_Anfangsinhalte.md#6823-oidreferenzauflösung) beschrieben.

- Da Hashes ungeordnete Datenstrukturen sind, sind Feldreihenfolge und Zeilenlayout semantisch nicht wichtig. Um dennoch ein einheitliches Erscheinungsbild zu erhalten, gibt es einige Regeln, die vom Formatierungsskript `reformat_dat_file.pl` angewendet werden:
  - Innerhalb jedes geschweiften Klammerpaars stehen die Metadatenfelder `oid`, `oid_symbol`, `array_type_oid` und `descr`, falls vorhanden, in dieser Reihenfolge zuerst; danach folgen die katalogeigenen Felder in ihrer definierten Reihenfolge.
  - Zeilenumbrüche werden zwischen Feldern nach Bedarf eingefügt, um die Zeilenlänge möglichst auf 80 Zeichen zu begrenzen. Außerdem wird zwischen den Metadatenfeldern und den regulären Feldern ein Zeilenumbruch eingefügt.
  - Wenn die `.h`-Datei des Katalogs einen Standardwert für eine Spalte festlegt und ein Dateneintrag denselben Wert hat, lässt `reformat_dat_file.pl` ihn aus der Datendatei weg. Das hält die Datenrepräsentation kompakt.
  - `reformat_dat_file.pl` erhält Leerzeilen und Kommentarzeilen unverändert.

Es wird empfohlen, `reformat_dat_file.pl` auszuführen, bevor Patches für Katalogdaten eingereicht werden. Zur Bequemlichkeit können Sie einfach nach `src/include/catalog/` wechseln und `make reformat-dat-files` ausführen.

Wenn Sie eine neue Methode hinzufügen wollen, um die Datenrepräsentation kleiner zu machen, müssen Sie sie in `reformat_dat_file.pl` implementieren und außerdem `Catalog::ParseData()` beibringen, wie die Daten wieder zur vollständigen Repräsentation expandiert werden.

### 68.2.2. OID-Zuweisung

Einer Katalogzeile in den Anfangsdaten kann eine manuell zugewiesene OID gegeben werden, indem ein Metadatenfeld `oid => nnnn` geschrieben wird. Wenn eine OID zugewiesen wird, kann außerdem ein C-Makro für diese OID erzeugt werden, indem ein Metadatenfeld `oid_symbol => name` geschrieben wird.

Vorgeladene Katalogzeilen müssen vorab zugewiesene OIDs haben, wenn es in anderen vorgeladenen Zeilen OID-Referenzen auf sie gibt. Eine vorab zugewiesene OID ist auch nötig, wenn die OID der Zeile aus C-Code referenziert werden muss. Wenn keiner der beiden Fälle zutrifft, kann das Metadatenfeld `oid` weggelassen werden; dann weist der Bootstrap-Code automatisch eine OID zu. In der Praxis werden für alle oder für keine der vorgeladenen Zeilen in einem bestimmten Katalog OIDs vorab zugewiesen, selbst wenn nur einige tatsächlich querreferenziert werden.

Den tatsächlichen numerischen Wert einer OID in C-Code zu schreiben, gilt als sehr schlechter Stil; verwenden Sie stattdessen immer ein Makro. Direkte Referenzen auf `pg_proc`-OIDs sind häufig genug, dass es einen speziellen Mechanismus gibt, um die nötigen Makros automatisch zu erzeugen; siehe `src/backend/utils/Gen_fmgrtab.pl`. Ähnlich gibt es, aus historischen Gründen jedoch anders umgesetzt, eine automatische Methode zum Erzeugen von Makros für `pg_type`-OIDs. `oid_symbol`-Einträge sind daher in diesen beiden Katalogen nicht nötig. Ebenso werden Makros für die `pg_class`-OIDs von Systemkatalogen und Indizes automatisch eingerichtet. Für alle anderen Systemkataloge müssen benötigte Makros manuell über `oid_symbol`-Einträge angegeben werden.

Um eine verfügbare OID für eine neue vorgeladene Zeile zu finden, führen Sie das Skript `src/include/catalog/unused_oids` aus. Es gibt inklusive Bereiche unbenutzter OIDs aus; zum Beispiel bedeutet die Ausgabezeile `45-900`, dass die OIDs 45 bis 900 noch nicht vergeben wurden. Derzeit sind OIDs 1 bis 9999 für manuelle Zuweisung reserviert; das Skript `unused_oids` durchsucht einfach die Katalog-Header und `.dat`-Dateien danach, welche nicht vorkommen. Mit dem Skript `duplicate_oids` können Sie außerdem Fehler prüfen. `genbki.pl` weist OIDs für alle Zeilen zu, die keine manuell zugewiesene OID erhalten haben, und erkennt doppelte OIDs zur Compile-Zeit.

Bei der Wahl von OIDs für einen Patch, der voraussichtlich nicht sofort committed wird, ist es Best Practice, eine Gruppe mehr oder weniger aufeinanderfolgender OIDs zu verwenden, die mit einer zufälligen Wahl im Bereich 8000 bis 9999 beginnt. Das minimiert das Risiko von OID-Kollisionen mit anderen parallel entwickelten Patches. Um den Bereich 8000 bis 9999 für Entwicklungszwecke freizuhalten, sollten die OIDs eines Patches nach dessen Commit in das Master-Git-Repository in verfügbaren Raum unterhalb dieses Bereichs umnummeriert werden. Typischerweise geschieht das gegen Ende jedes Entwicklungszyklus, wobei alle OIDs, die von in diesem Zyklus committeten Patches verbraucht wurden, gleichzeitig verschoben werden. Das Skript `renumber_oids.pl` kann dafür verwendet werden. Wenn ein nicht committeter Patch OID-Konflikte mit einem kürzlich committeten Patch hat, kann `renumber_oids.pl` ebenfalls nützlich sein, um die Situation zu bereinigen.

Wegen dieser Konvention möglicher Umnummerierung von durch Patches zugewiesenen OIDs sollten die einem Patch zugewiesenen OIDs erst dann als stabil gelten, wenn der Patch in ein offizielles Release aufgenommen wurde. Manuell zugewiesene Objekt-OIDs werden nach einem Release jedoch nicht geändert, weil das verschiedene Kompatibilitätsprobleme verursachen würde.

Wenn `genbki.pl` einer Katalogzeile eine OID zuweisen muss, die keine manuell zugewiesene OID hat, verwendet es einen Wert im Bereich 10000 bis 11999. Der OID-Zähler des Servers wird zu Beginn eines Bootstrap-Laufs auf 10000 gesetzt, sodass Objekte, die während der Bootstrap-Verarbeitung ad hoc erzeugt werden, ebenfalls OIDs aus diesem Bereich erhalten. Der normale OID-Zuweisungsmechanismus sorgt dafür, Konflikte zu verhindern.

Objekte mit OIDs unter `FirstUnpinnedObjectId` (12000) gelten als „pinned“ und können nicht gelöscht werden. Es gibt eine kleine Zahl von Ausnahmen, die in `IsPinnedObject()` fest verdrahtet sind. `initdb` zwingt den OID-Zähler auf `FirstUnpinnedObjectId`, sobald es bereit ist, unpinned Objekte zu erzeugen. Daher sind Objekte, die während späterer Phasen von `initdb` erzeugt werden, etwa beim Ausführen des Skripts `information_schema.sql`, nicht pinned, während alle `genbki.pl` bekannten Objekte pinned sind.

OIDs, die während des normalen Datenbankbetriebs vergeben werden, sind auf 16384 oder höher beschränkt. Dadurch bleibt der Bereich 10000 bis 16383 frei für OIDs, die automatisch von `genbki.pl` oder während `initdb` vergeben werden. Diese automatisch vergebenen OIDs gelten nicht als stabil und können von Installation zu Installation variieren.

### 68.2.3. OID-Referenzauflösung

Grundsätzlich könnten Querverweise von einer Anfangskatalogzeile auf eine andere einfach dadurch geschrieben werden, dass die vorab zugewiesene OID der referenzierten Zeile in das referenzierende Feld geschrieben wird. Das widerspricht jedoch der Projektpolitik, weil es fehleranfällig, schwer lesbar und anfällig für Brüche ist, wenn eine neu zugewiesene OID umnummeriert wird. Deshalb stellt `genbki.pl` Mechanismen bereit, um stattdessen symbolische Referenzen zu schreiben. Die Regeln lauten:

- Die Verwendung symbolischer Referenzen wird für eine bestimmte Katalogspalte aktiviert, indem `BKI_LOOKUP(lookuprule)` an die Spaltendefinition angehängt wird. Dabei ist `lookuprule` der Name des referenzierten Katalogs, zum Beispiel `pg_proc`. `BKI_LOOKUP` kann an Spalten der Typen `Oid`, `regproc`, `oidvector` oder `Oid[]` angehängt werden; in den letzten beiden Fällen bedeutet es, dass jedes Element des Arrays nachgeschlagen wird.

- Es ist auch erlaubt, `BKI_LOOKUP(encoding)` an Integer-Spalten anzuhängen, um Zeichensatzcodierungen zu referenzieren, die derzeit nicht als Katalog-OIDs repräsentiert sind, aber eine `genbki.pl` bekannte Wertemenge haben.

- In einigen Katalogspalten dürfen Einträge null sein, statt eine gültige Referenz zu enthalten. Wenn das erlaubt ist, schreiben Sie `BKI_LOOKUP_OPT` statt `BKI_LOOKUP`. Dann können Sie für einen Eintrag `0` schreiben. Wenn die Spalte als `regproc` deklariert ist, können Sie optional `-` statt `0` schreiben. Abgesehen von diesem Sonderfall müssen alle Einträge in einer `BKI_LOOKUP`-Spalte symbolische Referenzen sein. `genbki.pl` warnt vor unbekannten Namen.

- Die meisten Arten von Katalogobjekten werden einfach durch ihre Namen referenziert. Beachten Sie, dass Typnamen exakt mit dem `typname`-Eintrag des referenzierten `pg_type` übereinstimmen müssen; Aliase wie `integer` für `int4` dürfen nicht verwendet werden.

- Eine Funktion kann durch ihren `proname` dargestellt werden, wenn dieser unter den Einträgen in `pg_proc.dat` eindeutig ist; das funktioniert wie `regproc`-Eingabe. Andernfalls wird sie wie `regprocedure` als `proname(argtypename,argtypename,...)` geschrieben. Die Argumenttypnamen müssen exakt so geschrieben werden, wie sie im Feld `proargtypes` des Eintrags in `pg_proc.dat` stehen. Fügen Sie keine Leerzeichen ein.

- Operatoren werden als `oprname(lefttype,righttype)` dargestellt, wobei die Typnamen exakt so geschrieben werden, wie sie in den Feldern `oprleft` und `oprright` des Eintrags in `pg_operator.dat` stehen. Für den fehlenden Operanden eines unären Operators schreiben Sie `0`.

- Die Namen von Opclasses und Opfamilies sind nur innerhalb einer Access Method eindeutig, deshalb werden sie als `access_method_name/object_name` dargestellt.

- In keinem dieser Fälle gibt es eine Möglichkeit zur Schema-Qualifikation; alle während des Bootstrap erzeugten Objekte sollen im Schema `pg_catalog` liegen.

`genbki.pl` löst alle symbolischen Referenzen während seiner Ausführung auf und schreibt einfache numerische OIDs in die erzeugte BKI-Datei. Das Bootstrap-Backend muss sich daher nicht mit symbolischen Referenzen befassen.

Es ist wünschenswert, OID-Referenzspalten mit `BKI_LOOKUP` oder `BKI_LOOKUP_OPT` zu markieren, selbst wenn der Katalog keine Anfangsdaten hat, die einen Lookup erfordern. Dadurch kann `genbki.pl` die Fremdschlüsselbeziehungen aufzeichnen, die in den Systemkatalogen existieren. Diese Informationen werden in den Regressionstests verwendet, um fehlerhafte Einträge zu prüfen. Siehe auch die Makros `DECLARE_FOREIGN_KEY`, `DECLARE_FOREIGN_KEY_OPT`, `DECLARE_ARRAY_FOREIGN_KEY` und `DECLARE_ARRAY_FOREIGN_KEY_OPT`, mit denen Fremdschlüsselbeziehungen deklariert werden, die für `BKI_LOOKUP` zu komplex sind, typischerweise mehrspaltige Fremdschlüssel.

### 68.2.4. Automatische Erzeugung von Array-Typen

Die meisten skalaren Datentypen sollten einen entsprechenden Array-Typ haben, also einen Standard-`varlena`-Array-Typ, dessen Elementtyp der skalare Typ ist und der vom Feld `typarray` des `pg_type`-Eintrags des skalaren Typs referenziert wird. `genbki.pl` kann den `pg_type`-Eintrag für den Array-Typ in den meisten Fällen automatisch erzeugen.

Um diese Funktion zu nutzen, schreiben Sie einfach ein Metadatenfeld `array_type_oid => nnnn` in den `pg_type`-Eintrag des skalaren Typs und geben damit die OID an, die für den Array-Typ verwendet werden soll. Danach können Sie das Feld `typarray` weglassen, weil es automatisch mit dieser OID gefüllt wird.

Der Name des erzeugten Array-Typs ist der Name des skalaren Typs mit vorangestelltem Unterstrich. Die übrigen Felder des Array-Eintrags werden aus `BKI_ARRAY_DEFAULT(value)`-Annotationen in `pg_type.h` gefüllt oder, wenn es keine gibt, vom skalaren Typ kopiert. Für `typalign` gibt es außerdem einen Sonderfall. Danach werden die Felder `typelem` und `typarray` der beiden Einträge so gesetzt, dass sie einander querreferenzieren.

### 68.2.5. Rezepte zum Bearbeiten von Datendateien

Hier folgen einige Vorschläge für einfache Vorgehensweisen bei häufigen Aufgaben, wenn Katalogdatendateien aktualisiert werden.

- Eine neue Spalte mit Standardwert zu einem Katalog hinzufügen:
  - Fügen Sie die Spalte mit einer Annotation `BKI_DEFAULT(value)` zur Headerdatei hinzu. Die Datendatei muss nur dort angepasst werden, wo in bestehenden Zeilen ein vom Standard abweichender Wert benötigt wird.

- Einen Standardwert zu einer bestehenden Spalte hinzufügen, die noch keinen hat:
  - Fügen Sie der Headerdatei eine Annotation `BKI_DEFAULT` hinzu und führen Sie danach `make reformat-dat-files` aus, um nun redundante Feldeinträge zu entfernen.

- Eine Spalte entfernen, unabhängig davon, ob sie einen Standardwert hat:
  - Entfernen Sie die Spalte aus dem Header und führen Sie danach `make reformat-dat-files` aus, um nun nutzlose Feldeinträge zu entfernen.

- Einen bestehenden Standardwert ändern oder entfernen:
  - Sie können nicht einfach die Headerdatei ändern, weil dadurch die aktuellen Daten falsch interpretiert würden. Führen Sie zuerst `make expand-dat-files` aus, um die Datendateien mit allen Standardwerten explizit eingefügt umzuschreiben. Ändern oder entfernen Sie danach die Annotation `BKI_DEFAULT` und führen Sie anschließend `make reformat-dat-files` aus, um überflüssige Felder wieder zu entfernen.

- Ad-hoc-Massenbearbeitung:
  - `reformat_dat_file.pl` kann für viele Arten von Massenänderungen angepasst werden. Suchen Sie nach seinen Blockkommentaren, die zeigen, wo Einmalcode eingefügt werden kann. Im folgenden Beispiel werden zwei boolesche Felder in `pg_proc` zu einem `char`-Feld zusammengeführt.

1. Fügen Sie die neue Spalte mit Standardwert zu `pg_proc.h` hinzu:

```diff
+       /* see PROKIND_ categories below */
+       char        prokind BKI_DEFAULT(f);
```

2. Erzeugen Sie ein neues Skript auf Basis von `reformat_dat_file.pl`, um passende Werte on the fly einzufügen:

```diff
-                 # At this point we have the full row in memory as a hash
-                 # and can do any operations we want. As written, it only
-                 # removes default values, but this script can be adapted to
-                 # do one-off bulk-editing.
+            # One-off change to migrate to prokind
+            # Default has already been filled in by now, so change to other
+            # values as appropriate
+            if ($values{proisagg} eq 't')
+            {
+                 $values{prokind} = 'a';
+            }
+            elsif ($values{proiswindow} eq 't')
+            {
+                 $values{prokind} = 'w';
+            }
```

3. Führen Sie das neue Skript aus:

```text
$ cd src/include/catalog
$ perl rewrite_dat_with_prokind.pl pg_proc.dat
```

An diesem Punkt enthält `pg_proc.dat` alle drei Spalten, `prokind`, `proisagg` und `proiswindow`, auch wenn sie nur in Zeilen erscheinen, in denen sie vom Standardwert abweichen.

4. Entfernen Sie die alten Spalten aus `pg_proc.h`:

```diff
-       /* is it an aggregate? */
-       bool        proisagg BKI_DEFAULT(f);
-
-       /* is it a window function? */
-       bool        proiswindow BKI_DEFAULT(f);
```

5. Führen Sie abschließend `make reformat-dat-files` aus, um die nutzlosen alten Einträge aus `pg_proc.dat` zu entfernen.

Weitere Beispiele für Skripte zur Massenbearbeitung finden sich in `convert_oid2name.pl` und `remove_pg_type_oid_symbols.pl`, die an diese Nachricht angehängt sind: `https://www.postgresql.org/message-id/CAJVSVGVX8gXnPm+Xa=DxR7kFYprcQ1tNcCT5D0O3ShfnM6jehA@mail.gmail.com`

## 68.3. BKI-Dateiformat

Dieser Abschnitt beschreibt, wie das PostgreSQL-Backend BKI-Dateien interpretiert. Die Beschreibung ist leichter zu verstehen, wenn die Datei `postgres.bki` als Beispiel bereitliegt.

BKI-Eingabe besteht aus einer Folge von Befehlen. Befehle bestehen je nach Syntax des Befehls aus einer Anzahl von Tokens. Tokens werden normalerweise durch Whitespace getrennt, müssen es aber nicht, wenn keine Mehrdeutigkeit entsteht. Es gibt keinen besonderen Befehlstrenner; das nächste Token, das syntaktisch nicht mehr zum vorhergehenden Befehl gehören kann, beginnt einen neuen Befehl. Der Klarheit halber wird normalerweise jeder neue Befehl in eine neue Zeile geschrieben. Tokens können bestimmte Schlüsselwörter, Sonderzeichen wie Klammern und Kommas, Identifikatoren, Zahlen oder einfach gequotete Strings sein. Alles ist case-sensitive.

Zeilen, die mit `#` beginnen, werden ignoriert.

## 68.4. BKI-Befehle

- `create tablename tableoid [bootstrap] [shared_relation] [rowtype_oid oid] (name1 = type1 [FORCE NOT NULL | FORCE NULL ] [, name2 = type2 [FORCE NOT NULL | FORCE NULL ], ...])`
  - Erzeugt eine Tabelle namens `tablename` mit der OID `tableoid` und den in Klammern angegebenen Spalten.
  - Die folgenden Spaltentypen werden von `bootstrap.c` direkt unterstützt: `bool`, `bytea`, `char` (1 Byte), `name`, `int2`, `int4`, `regproc`, `regclass`, `regtype`, `text`, `oid`, `tid`, `xid`, `cid`, `int2vector`, `oidvector`, `_int4` (Array), `_text` (Array), `_oid` (Array), `_char` (Array), `_aclitem` (Array). Es ist zwar möglich, Tabellen mit Spalten anderer Typen zu erzeugen, aber das geht erst, nachdem `pg_type` erzeugt und mit passenden Einträgen gefüllt wurde. Effektiv bedeutet das, dass in Bootstrap-Katalogen nur diese Spaltentypen verwendet werden können; Nicht-Bootstrap-Kataloge können dagegen jeden eingebauten Typ enthalten.
  - Wenn `bootstrap` angegeben ist, wird die Tabelle nur auf Platte erzeugt; es wird nichts in `pg_class`, `pg_attribute` usw. für sie eingetragen. Die Tabelle ist daher für gewöhnliche SQL-Operationen nicht zugänglich, bis solche Einträge auf die harte Weise mit `insert`-Befehlen angelegt werden. Diese Option wird verwendet, um `pg_class` und ähnliche Kataloge selbst zu erzeugen.
  - Die Tabelle wird als shared erzeugt, wenn `shared_relation` angegeben ist. Die Zeilentyp-OID der Tabelle, also die `pg_type`-OID, kann optional mit der Klausel `rowtype_oid` angegeben werden; wenn sie nicht angegeben ist, wird automatisch eine OID dafür erzeugt. Die Klausel `rowtype_oid` ist nutzlos, wenn `bootstrap` angegeben ist, kann aber zur Dokumentation trotzdem angegeben werden.

- `open tablename`
  - Öffnet die Tabelle namens `tablename` zum Einfügen von Daten. Jede aktuell geöffnete Tabelle wird geschlossen.

- `close tablename`
  - Schließt die offene Tabelle. Der Name der Tabelle muss als Gegenprüfung angegeben werden.

- `insert ( [oid_value] value1 value2 ... )`
  - Fügt eine neue Zeile in die offene Tabelle ein und verwendet `value1`, `value2` usw. als Spaltenwerte.
  - `NULL`-Werte können mit dem besonderen Schlüsselwort `_null_` angegeben werden. Werte, die nicht wie Identifikatoren oder Ziffernfolgen aussehen, müssen in einfache Anführungszeichen gesetzt werden. Um ein einfaches Anführungszeichen in einen Wert aufzunehmen, schreiben Sie es doppelt. Backslash-Escapes im Stil von Escape-Strings sind ebenfalls erlaubt.

- `declare [unique] index indexname indexoid on tablename using amname ( opclass1 name1 [, ...] )`
  - Erzeugt einen Index namens `indexname` mit der OID `indexoid` auf der Tabelle `tablename` unter Verwendung der Access Method `amname`. Die zu indizierenden Felder heißen `name1`, `name2` usw., und die zu verwendenden Operatorklassen heißen entsprechend `opclass1`, `opclass2` usw. Die Indexdatei wird erzeugt und passende Katalogeinträge werden angelegt, aber der Indexinhalt wird durch diesen Befehl nicht initialisiert.

- `declare toast toasttableoid toastindexoid on tablename`
  - Erzeugt eine TOAST-Tabelle für die Tabelle `tablename`. Die TOAST-Tabelle erhält die OID `toasttableoid`, und ihr Index erhält die OID `toastindexoid`. Wie bei `declare index` wird das Füllen des Index aufgeschoben.

- `build indices`
  - Füllt die zuvor deklarierten Indizes.

## 68.5. Struktur der Bootstrap-BKI-Datei

Der Befehl `open` kann erst verwendet werden, wenn die Tabellen existieren, die er benutzt, und Einträge für die zu öffnende Tabelle vorhanden sind. Diese Minimalmenge von Tabellen ist `pg_class`, `pg_attribute`, `pg_proc` und `pg_type`. Damit diese Tabellen selbst gefüllt werden können, öffnet `create` mit der Option `bootstrap` die erzeugte Tabelle implizit für Dateneinfügungen.

Auch die Befehle `declare index` und `declare toast` können erst verwendet werden, nachdem die Systemkataloge, die sie benötigen, erzeugt und gefüllt wurden.

Die Datei `postgres.bki` muss daher folgende Struktur haben:

1. Einen der kritischen Tabellen mit `create bootstrap` erzeugen.
2. Daten einfügen, die mindestens die kritischen Tabellen beschreiben.
3. `close`
4. Für die anderen kritischen Tabellen wiederholen.
5. Eine nichtkritische Tabelle mit `create` ohne `bootstrap` erzeugen.
6. `open`
7. Gewünschte Daten einfügen.
8. `close`
9. Für die anderen nichtkritischen Tabellen wiederholen.
10. Indizes und TOAST-Tabellen definieren.
11. `build indices`

Es gibt zweifellos weitere, undokumentierte Ordnungsabhängigkeiten.

## 68.6. BKI-Beispiel

Die folgende Befehlsfolge erzeugt die Tabelle `test_table` mit der OID 420, die drei Spalten `oid`, `cola` und `colb` der Typen `oid`, `int4` und `text` hat, und fügt zwei Zeilen in die Tabelle ein:

```text
create test_table 420 (oid = oid, cola = int4, colb = text)
open test_table
insert ( 421 1 'value 1' )
insert ( 422 2 _null_ )
close test_table
```
