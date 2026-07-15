# 23. Lokalisierung

Dieses Kapitel beschreibt die verfügbaren Lokalisierungsfunktionen aus Sicht des Administrators. PostgreSQL unterstützt zwei Einrichtungen zur Lokalisierung:

- Die Locale-Funktionen des Betriebssystems, um locale-spezifische Sortierreihenfolgen, Zahlenformatierung, übersetzte Meldungen und andere Aspekte bereitzustellen. Dies wird in [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung) und [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung) behandelt.

- Die Bereitstellung verschiedener Zeichensätze, um Text in allen möglichen Sprachen speichern zu können, sowie Zeichensatzumsetzung zwischen Client und Server. Dies wird in [Abschnitt 23.3](23_Lokalisierung.md#233-zeichensatzunterstützung) behandelt.

## 23.1. Locale-Unterstützung

Locale-Unterstützung bedeutet, dass eine Anwendung kulturelle Präferenzen hinsichtlich Alphabeten, Sortierung, Zahlenformatierung usw. berücksichtigt. PostgreSQL verwendet die vom Serverbetriebssystem bereitgestellten Standard-Locale-Einrichtungen von ISO C und POSIX. Weitere Informationen finden Sie in der Dokumentation Ihres Systems.

### 23.1.1. Überblick

Die Locale-Unterstützung wird automatisch initialisiert, wenn mit `initdb` ein Datenbankcluster erstellt wird. Standardmäßig initialisiert `initdb` den Datenbankcluster mit der Locale-Einstellung seiner Ausführungsumgebung. Wenn Ihr System also bereits so eingestellt ist, dass es die Locale verwendet, die Sie auch in Ihrem Datenbankcluster nutzen möchten, müssen Sie nichts weiter tun. Wenn Sie eine andere Locale verwenden möchten (oder nicht sicher sind, auf welche Locale Ihr System eingestellt ist), können Sie `initdb` mit der Option `--locale` genau anweisen, welche Locale zu verwenden ist. Zum Beispiel:

```text
initdb --locale=sv_SE
```

Dieses Beispiel für Unix-Systeme setzt die Locale auf Schwedisch (`sv`), wie es in Schweden (`SE`) gesprochen wird. Weitere Möglichkeiten wären etwa `en_US` (US-Englisch) und `fr_CA` (Kanadisches Französisch). Wenn für eine Locale mehr als ein Zeichensatz verwendet werden kann, können Angaben die Form `language_territory.codeset` haben. Zum Beispiel steht `fr_BE.UTF-8` für die französische Sprache (`fr`), wie sie in Belgien (`BE`) gesprochen wird, mit einer UTF-8-Zeichensatzkodierung.

Welche Locales unter welchen Namen auf Ihrem System verfügbar sind, hängt davon ab, was der Betriebssystemhersteller bereitstellt und was installiert wurde. Auf den meisten Unix-Systemen liefert der Befehl `locale -a` eine Liste der verfügbaren Locales. Windows verwendet ausführlichere Locale-Namen wie `German_Germany` oder `Swedish_Sweden.1252`, aber die Prinzipien sind dieselben.

Gelegentlich ist es nützlich, Regeln aus mehreren Locales zu mischen, etwa englische Sortierregeln, aber spanische Meldungen zu verwenden. Dafür gibt es eine Reihe von Locale-Unterkategorien, die nur bestimmte Aspekte der Lokalisierungsregeln steuern:

| Kategorie | Bedeutung |
| --- | --- |
| `LC_COLLATE` | Sortierreihenfolge von Zeichenketten |
| `LC_CTYPE` | Zeichenklassifikation (Was ist ein Buchstabe? Was ist seine Großbuchstaben-Entsprechung?) |
| `LC_MESSAGES` | Sprache der Meldungen |
| `LC_MONETARY` | Formatierung von Währungsbeträgen |
| `LC_NUMERIC` | Formatierung von Zahlen |
| `LC_TIME` | Formatierung von Datum und Uhrzeit |

Die Kategorienamen entsprechen Namen von `initdb`-Optionen, mit denen die Locale-Auswahl für eine bestimmte Kategorie überschrieben werden kann. Um zum Beispiel die Locale auf Kanadisches Französisch zu setzen, aber US-Regeln für die Währungsformatierung zu verwenden, nutzen Sie `initdb --locale=fr_CA --lc-monetary=en_US`.

Wenn sich das System so verhalten soll, als hätte es keine Locale-Unterstützung, verwenden Sie den besonderen Locale-Namen `C` oder gleichwertig `POSIX`.

Einige Locale-Kategorien müssen beim Erstellen der Datenbank festgelegt werden. Sie können für verschiedene Datenbanken unterschiedliche Einstellungen verwenden, aber sobald eine Datenbank erstellt wurde, können Sie diese Werte für diese Datenbank nicht mehr ändern. `LC_COLLATE` und `LC_CTYPE` sind solche Kategorien. Sie beeinflussen die Sortierreihenfolge von Indizes und müssen daher fest bleiben, sonst würden Indizes auf Textspalten beschädigt. (Diese Einschränkung lässt sich jedoch mit Collations abmildern, wie in [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung) erläutert.) Die Standardwerte für diese Kategorien werden beim Ausführen von `initdb` bestimmt, und diese Werte werden beim Erstellen neuer Datenbanken verwendet, sofern im Befehl `CREATE DATABASE` nichts anderes angegeben ist.

Die anderen Locale-Kategorien können jederzeit geändert werden, indem die Serverkonfigurationsparameter mit denselben Namen wie die Locale-Kategorien gesetzt werden (Details siehe Abschnitt 19.11.2). Die von `initdb` gewählten Werte werden tatsächlich nur in die Konfigurationsdatei `postgresql.conf` geschrieben, damit sie beim Serverstart als Standardwerte dienen. Wenn Sie diese Zuweisungen aus `postgresql.conf` entfernen, erbt der Server die Einstellungen aus seiner Ausführungsumgebung.

Beachten Sie, dass das Locale-Verhalten des Servers durch die Umgebungsvariablen bestimmt wird, die der Server sieht, nicht durch die Umgebung irgendeines Clients. Achten Sie deshalb darauf, die richtigen Locale-Einstellungen zu konfigurieren, bevor Sie den Server starten. Eine Folge davon ist, dass Meldungen in unterschiedlichen Sprachen erscheinen können, wenn Client und Server mit unterschiedlichen Locales eingerichtet sind, je nachdem, wo die Meldung entstanden ist.

> Hinweis: Wenn hier davon gesprochen wird, dass die Locale aus der Ausführungsumgebung geerbt wird, bedeutet das auf den meisten Betriebssystemen Folgendes: Für eine gegebene Locale-Kategorie, etwa die Sortierung, werden die folgenden Umgebungsvariablen in dieser Reihenfolge herangezogen, bis eine gesetzte Variable gefunden wird: `LC_ALL`, `LC_COLLATE` (oder die Variable, die der jeweiligen Kategorie entspricht), `LANG`. Wenn keine dieser Umgebungsvariablen gesetzt ist, fällt die Locale auf `C` zurück.
>
> Einige Bibliotheken zur Meldungslokalisierung betrachten außerdem die Umgebungsvariable `LANGUAGE`, die alle anderen Locale-Einstellungen zum Zweck der Meldungssprache überschreibt. Im Zweifel konsultieren Sie bitte die Dokumentation Ihres Betriebssystems, insbesondere die Dokumentation zu `gettext`.

Damit Meldungen in die bevorzugte Sprache des Benutzers übersetzt werden können, muss NLS zur Build-Zeit ausgewählt worden sein (`configure --enable-nls`). Alle übrige Locale-Unterstützung ist automatisch eingebaut.

### 23.1.2. Verhalten

Die Locale-Einstellungen beeinflussen die folgenden SQL-Funktionen:

- Sortierreihenfolge in Abfragen mit `ORDER BY` oder den Standardvergleichsoperatoren auf Textdaten

- die Funktionen `upper`, `lower` und `initcap`

- Mustervergleichsoperatoren (`LIKE`, `SIMILAR TO` und reguläre Ausdrücke im POSIX-Stil); Locales beeinflussen sowohl Groß-/Kleinschreibungs-unabhängige Vergleiche als auch die Zeichenklassifikation durch Zeichenklassen in regulären Ausdrücken

- die Funktionsfamilie `to_char`

- die Möglichkeit, Indizes mit `LIKE`-Klauseln zu verwenden

Der Nachteil der Verwendung anderer Locales als `C` oder `POSIX` in PostgreSQL ist ihre Auswirkung auf die Leistung. Sie verlangsamt die Zeichenverarbeitung und verhindert, dass gewöhnliche Indizes für `LIKE` verwendet werden. Verwenden Sie Locales deshalb nur, wenn Sie sie tatsächlich benötigen.

Als Umgehung, damit PostgreSQL unter einer Nicht-`C`-Locale Indizes mit `LIKE`-Klauseln verwenden kann, gibt es mehrere benutzerdefinierte Operatorklassen. Diese erlauben das Erstellen eines Index, der einen strikten zeichenweisen Vergleich ausführt und Locale-Vergleichsregeln ignoriert. Weitere Informationen finden Sie in [Abschnitt 11.10](11_Indizes.md#1110-operatorklassen-und-operatorfamilien). Ein anderer Ansatz besteht darin, Indizes mit der `C`-Collation zu erstellen, wie in [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung) beschrieben.

### 23.1.3. Locales auswählen

Locales können je nach Anforderung in unterschiedlichen Geltungsbereichen ausgewählt werden. Der Überblick oben hat gezeigt, wie Locales mit `initdb` angegeben werden, um die Standardwerte für den gesamten Cluster zu setzen. Die folgende Liste zeigt, wo Locales ausgewählt werden können. Jeder Eintrag liefert die Standardwerte für die folgenden Einträge, und jeder weiter unten stehende Eintrag erlaubt es, die Standardwerte mit feinerer Granularität zu überschreiben.

1. Wie oben erläutert, stellt die Umgebung des Betriebssystems die Standardwerte für die Locales eines neu initialisierten Datenbankclusters bereit. In vielen Fällen reicht das aus: Wenn das Betriebssystem für die gewünschte Sprache/Region konfiguriert ist, verhält sich PostgreSQL standardmäßig ebenfalls entsprechend dieser Locale.

2. Wie oben gezeigt, geben Kommandozeilenoptionen für `initdb` die Locale-Einstellungen für einen neu initialisierten Datenbankcluster an. Verwenden Sie dies, wenn das Betriebssystem nicht die Locale-Konfiguration hat, die Sie für Ihr Datenbanksystem wünschen.

3. Für jede Datenbank kann eine Locale separat ausgewählt werden. Der SQL-Befehl `CREATE DATABASE` und sein Kommandozeilenäquivalent `createdb` haben dafür Optionen. Verwenden Sie dies zum Beispiel, wenn ein Datenbankcluster Datenbanken für mehrere Mandanten mit unterschiedlichen Anforderungen beherbergt.

4. Locale-Einstellungen können für einzelne Tabellenspalten vorgenommen werden. Dazu wird ein SQL-Objekt namens Collation verwendet; dies wird in [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung) erläutert. Verwenden Sie dies zum Beispiel, um Daten in verschiedenen Sprachen zu sortieren oder die Sortierreihenfolge einer bestimmten Tabelle anzupassen.

5. Schließlich können Locales für eine einzelne Abfrage ausgewählt werden. Auch hierfür werden SQL-Collation-Objekte verwendet. Das kann genutzt werden, um die Sortierreihenfolge auf Basis von Laufzeitentscheidungen zu ändern oder um ad hoc zu experimentieren.

### 23.1.4. Locale-Provider

Ein Locale-Provider gibt an, welche Bibliothek das Locale-Verhalten für Collations und Zeichenklassifikationen definiert.

Die oben beschriebenen Befehle und Werkzeuge, die Locale-Einstellungen auswählen, haben jeweils eine Option zur Auswahl des Locale-Providers. Hier ist ein Beispiel, um einen Datenbankcluster mit dem ICU-Provider zu initialisieren:

```text
initdb --locale-provider=icu --icu-locale=en
```

Details finden Sie in der Beschreibung der jeweiligen Befehle und Programme. Beachten Sie, dass Locale-Provider auf unterschiedlichen Granularitäten gemischt werden können: Man kann zum Beispiel standardmäßig `libc` für den Cluster verwenden, aber eine Datenbank mit dem Provider `icu` haben, und dann innerhalb dieser Datenbanken Collation-Objekte mit einem der beiden Provider verwenden.

Unabhängig vom Locale-Provider wird das Betriebssystem weiterhin für einige locale-bewusste Verhaltensweisen verwendet, etwa für Meldungen (siehe `lc_messages`).

Die verfügbaren Locale-Provider sind unten aufgeführt:

`builtin`

Der Provider `builtin` verwendet eingebaute Operationen. Für diesen Provider werden nur die Locales `C`, `C.UTF-8` und `PG_UNICODE_FAST` unterstützt.

Das Verhalten der Locale `C` ist identisch mit der Locale `C` im Provider `libc`. Bei Verwendung dieser Locale kann das Verhalten von der Datenbankkodierung abhängen.

Die Locale `C.UTF-8` ist nur verfügbar, wenn die Datenbankkodierung UTF-8 ist, und das Verhalten basiert auf Unicode. Die Collation verwendet nur die Codepunktwerte. Die Zeichenklassen regulärer Ausdrücke basieren auf der Semantik „POSIX Compatible“, und die Groß-/Kleinschreibung-Abbildung ist die Variante „simple“.

Die Locale `PG_UNICODE_FAST` ist nur verfügbar, wenn die Datenbankkodierung UTF-8 ist, und das Verhalten basiert auf Unicode. Die Collation verwendet nur die Codepunktwerte. Die Zeichenklassen regulärer Ausdrücke basieren auf der Semantik „Standard“, und die Groß-/Kleinschreibung-Abbildung ist die Variante „full“.

`icu`

Der Provider `icu` verwendet die externe ICU-Bibliothek. PostgreSQL muss mit entsprechender Unterstützung konfiguriert worden sein.

ICU stellt Collation- und Zeichenklassifikationsverhalten bereit, das unabhängig vom Betriebssystem und von der Datenbankkodierung ist. Das ist vorzuziehen, wenn Sie erwarten, auf andere Plattformen umzusteigen, ohne Änderungen in den Ergebnissen zu erhalten. `LC_COLLATE` und `LC_CTYPE` können unabhängig von der ICU-Locale gesetzt werden.

> Hinweis: Beim ICU-Provider können Ergebnisse von der verwendeten Version der ICU-Bibliothek abhängen, da diese im Laufe der Zeit aktualisiert wird, um Änderungen natürlicher Sprachen abzubilden.

`libc`

Der Provider `libc` verwendet die C-Bibliothek des Betriebssystems. Das Collation- und Zeichenklassifikationsverhalten wird durch die Einstellungen `LC_COLLATE` und `LC_CTYPE` gesteuert; sie können daher nicht unabhängig voneinander gesetzt werden.

> Hinweis: Derselbe Locale-Name kann auf verschiedenen Plattformen ein unterschiedliches Verhalten haben, wenn der Provider `libc` verwendet wird.

### 23.1.5. ICU-Locales

#### 23.1.5.1. ICU-Locale-Namen

Das ICU-Format für den Locale-Namen ist ein Sprach-Tag.

```sql
CREATE COLLATION mycollation1 (provider = icu, locale = 'ja-JP');
CREATE COLLATION mycollation2 (provider = icu, locale = 'fr');
```

#### 23.1.5.2. Locale-Kanonisierung und Validierung

Wenn ein neues ICU-Collation-Objekt oder eine Datenbank mit ICU als Provider definiert wird, wird der angegebene Locale-Name in einen Sprach-Tag umgewandelt („kanonisiert“), sofern er nicht bereits in dieser Form vorliegt. Zum Beispiel:

```sql
CREATE COLLATION mycollation3 (provider = icu, locale = 'en-US-u-
kn-true');
```

```text
NOTICE:  using standard form "en-US-u-kn" for locale "en-US-u-kn-
true"
```

```sql
CREATE COLLATION mycollation4 (provider = icu, locale =
 'de_DE.utf8');
```

```text
NOTICE:  using standard form "de-DE" for locale "de_DE.utf8"
```

Wenn Sie diese Notice sehen, stellen Sie sicher, dass Provider und Locale das erwartete Ergebnis sind. Für konsistente Ergebnisse mit dem ICU-Provider geben Sie den kanonischen Sprach-Tag an, statt sich auf die Umwandlung zu verlassen.

Eine Locale ohne Sprachnamen oder mit dem besonderen Sprachnamen `root` wird so umgewandelt, dass sie die Sprache `und` („undefined“) hat.

ICU kann die meisten `libc`-Locale-Namen sowie einige andere Formate in Sprach-Tags umwandeln, um den Übergang zu ICU zu erleichtern. Wenn ein `libc`-Locale-Name in ICU verwendet wird, hat er möglicherweise nicht exakt dasselbe Verhalten wie in `libc`.

Wenn es ein Problem beim Interpretieren des Locale-Namens gibt oder wenn der Locale-Name eine Sprache oder Region repräsentiert, die ICU nicht erkennt, sehen Sie die folgende Warnung:

```sql
CREATE COLLATION nonsense (provider = icu, locale = 'nonsense');
```

```text
WARNING:  ICU locale "nonsense" has unknown language "nonsense"
HINT:  To disable ICU locale validation, set parameter
 icu_validation_level to DISABLED.
CREATE COLLATION
```

`icu_validation_level` steuert, wie die Meldung ausgegeben wird. Sofern der Parameter nicht auf `ERROR` gesetzt ist, wird die Collation trotzdem erstellt, aber das Verhalten entspricht möglicherweise nicht der Absicht des Benutzers.

#### 23.1.5.3. Sprach-Tag

Ein Sprach-Tag, definiert in BCP 47, ist ein standardisierter Bezeichner zur Identifikation von Sprachen, Regionen und weiteren Informationen über eine Locale.

Einfache Sprach-Tags sind einfach `language-region` oder sogar nur `language`. `language` ist ein Sprachcode (zum Beispiel `fr` für Französisch), und `region` ist ein Regionscode (zum Beispiel `CA` für Kanada). Beispiele sind `ja-JP`, `de` oder `fr-CA`.

Collation-Einstellungen können in den Sprach-Tag aufgenommen werden, um das Collation-Verhalten anzupassen. ICU erlaubt umfangreiche Anpassungen, etwa Empfindlichkeit (oder Unempfindlichkeit) gegenüber Akzenten, Groß-/Kleinschreibung und Interpunktion, die Behandlung von Ziffern innerhalb von Text und viele weitere Optionen für unterschiedliche Einsatzzwecke.

Um diese zusätzlichen Collation-Informationen in einen Sprach-Tag aufzunehmen, hängen Sie `-u` an, was anzeigt, dass zusätzliche Collation-Einstellungen folgen, gefolgt von einem oder mehreren Paaren `-key-value`. `key` ist der Schlüssel für eine Collation-Einstellung, und `value` ist ein gültiger Wert für diese Einstellung. Bei booleschen Einstellungen kann `-key` ohne zugehöriges `-value` angegeben werden; das impliziert den Wert `true`.

Zum Beispiel bedeutet der Sprach-Tag `en-US-u-kn-ks-level2` die Locale mit englischer Sprache in der Region USA, mit den Collation-Einstellungen `kn` auf `true` und `ks` auf `level2`. Diese Einstellungen bedeuten, dass die Collation Groß-/Kleinschreibung ignoriert und eine Ziffernfolge als einzelne Zahl behandelt:

```sql
CREATE COLLATION mycollation5 (provider = icu, deterministic =
 false, locale = 'en-US-u-kn-ks-level2');
SELECT 'aB' = 'Ab' COLLATE mycollation5 as result;
```

```text
 result
--------
 t
(1 row)
```

```sql
SELECT 'N-45' < 'N-123' COLLATE mycollation5 as result;
```

```text
 result
--------
 t
(1 row)
```

Details und weitere Beispiele zur Verwendung von Sprach-Tags mit benutzerdefinierten Collation-Informationen für die Locale finden Sie in Abschnitt 23.2.3.

### 23.1.6. Probleme

Wenn die Locale-Unterstützung nicht wie oben beschrieben funktioniert, prüfen Sie, ob die Locale-Unterstützung Ihres Betriebssystems korrekt konfiguriert ist. Um zu prüfen, welche Locales auf Ihrem System installiert sind, können Sie den Befehl `locale -a` verwenden, sofern Ihr Betriebssystem ihn bereitstellt.

Prüfen Sie, ob PostgreSQL tatsächlich die Locale verwendet, die Sie erwarten. Die Einstellungen `LC_COLLATE` und `LC_CTYPE` werden beim Erstellen einer Datenbank bestimmt und können nur durch Erstellen einer neuen Datenbank geändert werden. Andere Locale-Einstellungen einschließlich `LC_MESSAGES` und `LC_MONETARY` werden zunächst durch die Umgebung bestimmt, in der der Server gestartet wird, können aber im laufenden Betrieb geändert werden. Die aktiven Locale-Einstellungen können Sie mit dem Befehl `SHOW` prüfen.

Das Verzeichnis `src/test/locale` in der Quelldistribution enthält eine Testsuite für die Locale-Unterstützung von PostgreSQL.

Client-Anwendungen, die serverseitige Fehler behandeln, indem sie den Text der Fehlermeldung parsen, werden offenkundig Probleme haben, wenn die Meldungen des Servers in einer anderen Sprache ausgegeben werden. Autoren solcher Anwendungen wird empfohlen, stattdessen das Fehlercode-Schema zu verwenden.

Die Pflege von Katalogen mit Meldungsübersetzungen erfordert die fortlaufende Arbeit vieler Freiwilliger, die PostgreSQL in ihrer bevorzugten Sprache gut sprechen sehen möchten. Wenn Meldungen in Ihrer Sprache derzeit nicht verfügbar oder nicht vollständig übersetzt sind, ist Ihre Mithilfe willkommen. Wenn Sie helfen möchten, lesen Sie [Kapitel 56](56_Native_Language_Support.md) oder schreiben Sie an die Entwickler-Mailingliste.

## 23.2. Collation-Unterstützung

Die Collation-Funktion erlaubt es, die Sortierreihenfolge und das Zeichenklassifikationsverhalten von Daten pro Spalte oder sogar pro Operation festzulegen. Damit wird die Einschränkung abgemildert, dass die Einstellungen `LC_COLLATE` und `LC_CTYPE` einer Datenbank nach ihrer Erstellung nicht mehr geändert werden können.

### 23.2.1. Konzepte

Konzeptionell hat jeder Ausdruck eines sortierbaren Datentyps eine Collation. (Die eingebauten sortierbaren Datentypen sind `text`, `varchar` und `char`. Benutzerdefinierte Basistypen können ebenfalls als sortierbar markiert werden, und selbstverständlich ist auch eine Domain über einem sortierbaren Datentyp sortierbar.) Wenn der Ausdruck eine Spaltenreferenz ist, ist die Collation des Ausdrucks die definierte Collation der Spalte. Wenn der Ausdruck eine Konstante ist, ist die Collation die Standard-Collation des Datentyps der Konstante. Die Collation eines komplexeren Ausdrucks wird aus den Collations seiner Eingaben abgeleitet, wie unten beschrieben.

Die Collation eines Ausdrucks kann die „default“-Collation sein, also die für die Datenbank definierten Locale-Einstellungen. Es ist auch möglich, dass die Collation eines Ausdrucks unbestimmt ist. In solchen Fällen schlagen Sortieroperationen und andere Operationen fehl, die die Collation kennen müssen.

Wenn das Datenbanksystem eine Sortierung oder Zeichenklassifikation ausführen muss, verwendet es die Collation des Eingabeausdrucks. Das geschieht zum Beispiel bei `ORDER BY`-Klauseln und bei Funktions- oder Operatoraufrufen wie `<`. Die Collation, die für eine `ORDER BY`-Klausel angewendet wird, ist einfach die Collation des Sortierschlüssels. Die Collation, die für einen Funktions- oder Operatoraufruf angewendet wird, wird aus den Argumenten abgeleitet, wie unten beschrieben. Zusätzlich zu Vergleichsoperatoren werden Collations von Funktionen berücksichtigt, die zwischen Klein- und Großbuchstaben umwandeln, etwa `lower`, `upper` und `initcap`; außerdem von Mustervergleichsoperatoren sowie von `to_char` und verwandten Funktionen.

Bei einem Funktions- oder Operatoraufruf wird die Collation, die aus den Argument-Collations abgeleitet wird, zur Laufzeit verwendet, um die angegebene Operation auszuführen. Wenn das Ergebnis des Funktions- oder Operatoraufrufs einen sortierbaren Datentyp hat, wird die Collation außerdem zur Parse-Zeit als definierte Collation des Funktions- oder Operatorausdrucks verwendet, falls es einen umgebenden Ausdruck gibt, der diese Collation kennen muss.

Die Collation-Ableitung eines Ausdrucks kann implizit oder explizit sein. Diese Unterscheidung beeinflusst, wie Collations kombiniert werden, wenn mehrere verschiedene Collations in einem Ausdruck auftreten. Eine explizite Collation-Ableitung liegt vor, wenn eine `COLLATE`-Klausel verwendet wird; alle anderen Collation-Ableitungen sind implizit. Wenn mehrere Collations kombiniert werden müssen, zum Beispiel in einem Funktionsaufruf, gelten die folgenden Regeln:

1. Wenn irgendein Eingabeausdruck eine explizite Collation-Ableitung hat, müssen alle explizit abgeleiteten Collations unter den Eingabeausdrücken identisch sein; andernfalls wird ein Fehler ausgelöst. Wenn eine explizit abgeleitete Collation vorhanden ist, ist sie das Ergebnis der Collation-Kombination.

2. Andernfalls müssen alle Eingabeausdrücke dieselbe implizite Collation-Ableitung oder die Default-Collation haben. Wenn eine Nicht-Default-Collation vorhanden ist, ist sie das Ergebnis der Collation-Kombination. Andernfalls ist das Ergebnis die Default-Collation.

3. Wenn es widersprüchliche implizite Nicht-Default-Collations unter den Eingabeausdrücken gibt, gilt die Kombination als Ausdruck mit unbestimmter Collation. Das ist kein Fehler, es sei denn, die konkret aufgerufene Funktion muss wissen, welche Collation sie anwenden soll. Falls sie das muss, wird zur Laufzeit ein Fehler ausgelöst.

Betrachten Sie zum Beispiel diese Tabellendefinition:

```sql
CREATE TABLE test1 (
    a text COLLATE "de_DE",
    b text COLLATE "es_ES",
    ...
);
```

Dann wird in

```sql
SELECT a < 'foo' FROM test1;
```

der Vergleich `<` nach den Regeln von `de_DE` ausgeführt, weil der Ausdruck eine implizit abgeleitete Collation mit der Default-Collation kombiniert. In

```sql
SELECT a < ('foo' COLLATE "fr_FR") FROM test1;
```

wird der Vergleich dagegen nach den Regeln von `fr_FR` ausgeführt, weil die explizite Collation-Ableitung die implizite überschreibt. Bei

```sql
SELECT a < b FROM test1;
```

kann der Parser nicht bestimmen, welche Collation anzuwenden ist, da die Spalten `a` und `b` widersprüchliche implizite Collations haben. Da der Operator `<` wissen muss, welche Collation zu verwenden ist, führt dies zu einem Fehler. Der Fehler kann behoben werden, indem an einen der Eingabeausdrücke ein expliziter Collation-Spezifizierer angehängt wird:

```sql
SELECT a < b COLLATE "de_DE" FROM test1;
```

oder gleichwertig:

```sql
SELECT a COLLATE "de_DE" < b FROM test1;
```

Der strukturell ähnliche Fall

```sql
SELECT a || b FROM test1;
```

führt dagegen nicht zu einem Fehler, weil sich der Operator `||` nicht für Collations interessiert: Sein Ergebnis ist unabhängig von der Collation dasselbe.

Die Collation, die den kombinierten Eingabeausdrücken einer Funktion oder eines Operators zugewiesen wird, gilt außerdem als Collation des Ergebnisses dieser Funktion oder dieses Operators, sofern die Funktion oder der Operator ein Ergebnis eines sortierbaren Datentyps liefert. Daher wird in

```sql
SELECT * FROM test1 ORDER BY a || 'foo';
```

die Sortierung nach den Regeln von `de_DE` ausgeführt. Diese Abfrage:

```sql
SELECT * FROM test1 ORDER BY a || b;
```

führt jedoch zu einem Fehler, denn obwohl der Operator `||` keine Collation kennen muss, muss die `ORDER BY`-Klausel sie kennen. Wie zuvor kann der Konflikt mit einem expliziten Collation-Spezifizierer gelöst werden:

```sql
SELECT * FROM test1 ORDER BY a || b COLLATE "fr_FR";
```

### 23.2.2. Collations verwalten

Eine Collation ist ein SQL-Schemaobjekt, das einen SQL-Namen auf Locales abbildet, die von im Betriebssystem installierten Bibliotheken bereitgestellt werden. Eine Collation-Definition hat einen Provider, der angibt, welche Bibliothek die Locale-Daten liefert. Ein Standard-Providername ist `libc`; dieser verwendet die Locales, die von der C-Bibliothek des Betriebssystems bereitgestellt werden. Das sind die Locales, die von den meisten Werkzeugen des Betriebssystems verwendet werden. Ein weiterer Provider ist `icu`, der die externe ICU-Bibliothek verwendet. ICU-Locales können nur verwendet werden, wenn PostgreSQL beim Build mit ICU-Unterstützung konfiguriert wurde.

Ein von `libc` bereitgestelltes Collation-Objekt wird auf eine Kombination von `LC_COLLATE`- und `LC_CTYPE`-Einstellungen abgebildet, wie sie vom Systembibliotheksaufruf `setlocale()` akzeptiert werden. (Wie der Name nahelegt, besteht der Hauptzweck einer Collation darin, `LC_COLLATE` festzulegen, das die Sortierreihenfolge steuert. In der Praxis ist es aber selten nötig, eine `LC_CTYPE`-Einstellung zu haben, die sich von `LC_COLLATE` unterscheidet; daher ist es bequemer, beides unter einem Konzept zusammenzufassen, statt eine weitere Infrastruktur zum Setzen von `LC_CTYPE` pro Ausdruck zu schaffen.) Außerdem ist eine `libc`-Collation an eine Zeichensatzkodierung gebunden (siehe [Abschnitt 23.3](23_Lokalisierung.md#233-zeichensatzunterstützung)). Derselbe Collation-Name kann für verschiedene Kodierungen existieren.

Ein von `icu` bereitgestelltes Collation-Objekt wird auf einen benannten Collator abgebildet, der von der ICU-Bibliothek bereitgestellt wird. ICU unterstützt keine getrennten „collate“- und „ctype“-Einstellungen; sie sind also immer gleich. Außerdem sind ICU-Collations unabhängig von der Kodierung, sodass es in einer Datenbank immer nur eine ICU-Collation eines gegebenen Namens gibt.

#### 23.2.2.1. Standard-Collations

Auf allen Plattformen werden die folgenden Collations unterstützt:

`unicode`

Diese SQL-Standard-Collation sortiert mit dem Unicode Collation Algorithm und der Default Unicode Collation Element Table. Sie ist in allen Kodierungen verfügbar. Um diese Collation zu verwenden, ist ICU-Unterstützung erforderlich, und das Verhalten kann sich ändern, wenn PostgreSQL mit einer anderen ICU-Version gebaut wird. (Diese Collation hat dasselbe Verhalten wie die ICU-Root-Locale; siehe `und-x-icu` für „undefined“.)

`ucs_basic`

Diese SQL-Standard-Collation sortiert nach Unicode-Codepunktwerten statt nach natürlicher Sprachordnung, und nur die ASCII-Buchstaben „A“ bis „Z“ werden als Buchstaben behandelt. Das Verhalten ist effizient und über alle Versionen stabil. Sie ist nur für die Kodierung `UTF8` verfügbar. (Diese Collation hat dasselbe Verhalten wie die `libc`-Locale-Spezifikation `C` in UTF8-Kodierung.)

`pg_unicode_fast`

Diese Collation sortiert nach Unicode-Codepunktwerten statt nach natürlicher Sprachordnung. Für die Funktionen `lower`, `initcap` und `upper` verwendet sie Unicode Full Case Mapping. Für Mustervergleiche (einschließlich regulärer Ausdrücke) verwendet sie die Standard-Variante der Unicode Compatibility Properties. Das Verhalten ist innerhalb einer PostgreSQL-Hauptversion effizient und stabil. Sie ist nur für die Kodierung `UTF8` verfügbar.

`pg_c_utf8`

Diese Collation sortiert nach Unicode-Codepunktwerten statt nach natürlicher Sprachordnung. Für die Funktionen `lower`, `initcap` und `upper` verwendet sie Unicode Simple Case Mapping. Für Mustervergleiche (einschließlich regulärer Ausdrücke) verwendet sie die POSIX-Compatible-Variante der Unicode Compatibility Properties. Das Verhalten ist innerhalb einer PostgreSQL-Hauptversion effizient und stabil. Diese Collation ist nur für die Kodierung `UTF8` verfügbar.

`C` (gleichwertig mit `POSIX`)

Die Collations `C` und `POSIX` beruhen auf dem „traditionellen C“-Verhalten. Sie sortieren nach Bytewerten statt nach natürlicher Sprachordnung, und nur die ASCII-Buchstaben „A“ bis „Z“ werden als Buchstaben behandelt. Das Verhalten ist für eine gegebene Datenbankkodierung über alle Versionen effizient und stabil, kann sich aber zwischen verschiedenen Datenbankkodierungen unterscheiden.

`default`

Die Default-Collation wählt die Locale aus, die beim Erstellen der Datenbank angegeben wurde.

Je nach Unterstützung des Betriebssystems können weitere Collations verfügbar sein. Effizienz und Stabilität dieser zusätzlichen Collations hängen vom Collation-Provider, von der Provider-Version und von der Locale ab.

#### 23.2.2.2. Vordefinierte Collations

Wenn das Betriebssystem Unterstützung dafür bietet, mehrere Locales innerhalb eines einzelnen Programms zu verwenden (`newlocale` und verwandte Funktionen), oder wenn ICU-Unterstützung konfiguriert ist, füllt `initdb` beim Initialisieren eines Datenbankclusters den Systemkatalog `pg_collation` mit Collations, die auf allen zu diesem Zeitpunkt im Betriebssystem gefundenen Locales beruhen.

Um die aktuell verfügbaren Locales zu prüfen, verwenden Sie die Abfrage `SELECT * FROM pg_collation` oder den Befehl `\dOS+` in `psql`.

##### 23.2.2.2.1. libc-Collations

Das Betriebssystem könnte zum Beispiel eine Locale namens `de_DE.utf8` bereitstellen. `initdb` würde dann eine Collation namens `de_DE.utf8` für die Kodierung `UTF8` erstellen, bei der sowohl `LC_COLLATE` als auch `LC_CTYPE` auf `de_DE.utf8` gesetzt sind. Außerdem erstellt es eine Collation, bei der der Zusatz `.utf8` aus dem Namen entfernt wurde. Sie könnten die Collation also auch unter dem Namen `de_DE` verwenden, der weniger umständlich zu schreiben ist und den Namen weniger kodierungsabhängig macht. Beachten Sie jedoch, dass die anfängliche Menge von Collation-Namen trotzdem plattformabhängig ist.

Die von `libc` bereitgestellte Standardmenge von Collations wird direkt auf die im Betriebssystem installierten Locales abgebildet, die mit dem Befehl `locale -a` aufgelistet werden können. Falls eine `libc`-Collation benötigt wird, die unterschiedliche Werte für `LC_COLLATE` und `LC_CTYPE` hat, oder falls nach der Initialisierung des Datenbanksystems neue Locales im Betriebssystem installiert wurden, kann mit dem Befehl `CREATE COLLATION` eine neue Collation erstellt werden. Neue Betriebssystem-Locales können außerdem gesammelt mit der Funktion `pg_import_system_collations()` importiert werden.

Innerhalb einer bestimmten Datenbank sind nur Collations von Interesse, die die Kodierung dieser Datenbank verwenden. Andere Einträge in `pg_collation` werden ignoriert. Daher kann ein gekürzter Collation-Name wie `de_DE` innerhalb einer gegebenen Datenbank als eindeutig betrachtet werden, auch wenn er global nicht eindeutig wäre. Die Verwendung gekürzter Collation-Namen wird empfohlen, weil Sie dadurch eine Sache weniger ändern müssen, wenn Sie sich später entscheiden, zu einer anderen Datenbankkodierung zu wechseln. Beachten Sie jedoch, dass die Collations `default`, `C` und `POSIX` unabhängig von der Datenbankkodierung verwendet werden können.

PostgreSQL betrachtet unterschiedliche Collation-Objekte auch dann als inkompatibel, wenn sie identische Eigenschaften haben. Daher erzeugt zum Beispiel

```sql
SELECT a COLLATE "C" < b COLLATE "POSIX" FROM test1;
```

einen Fehler, obwohl die Collations `C` und `POSIX` identisches Verhalten haben. Das Mischen gekürzter und nicht gekürzter Collation-Namen wird daher nicht empfohlen.

##### 23.2.2.2.2. ICU-Collations

Bei ICU ist es nicht sinnvoll, alle möglichen Locale-Namen aufzuzählen. ICU verwendet ein bestimmtes Namenssystem für Locales, aber es gibt viel mehr Möglichkeiten, eine Locale zu benennen, als es tatsächlich verschiedene Locales gibt. `initdb` verwendet die ICU-APIs, um eine Menge unterschiedlicher Locales zu extrahieren und damit die anfängliche Menge von Collations zu füllen. Von ICU bereitgestellte Collations werden in der SQL-Umgebung mit Namen im BCP-47-Sprach-Tag-Format erstellt; daran wird die „private use“-Erweiterung `-x-icu` angehängt, um sie von `libc`-Locales zu unterscheiden.

Hier sind einige Beispiel-Collations, die erstellt werden könnten:

`de-x-icu`

Deutsche Collation, Standardvariante.

`de-AT-x-icu`

Deutsche Collation für Österreich, Standardvariante.

(Es gibt zum Beispiel auch `de-DE-x-icu` oder `de-CH-x-icu`, aber zum Zeitpunkt dieser Beschreibung sind sie gleichwertig mit `de-x-icu`.)

`und-x-icu` (für „undefined“)

ICU-„Root“-Collation. Verwenden Sie diese, um eine vernünftige sprachunabhängige Sortierreihenfolge zu erhalten.

Einige weniger häufig verwendete Kodierungen werden von ICU nicht unterstützt. Wenn die Datenbankkodierung eine davon ist, werden ICU-Collation-Einträge in `pg_collation` ignoriert. Der Versuch, eine solche Collation zu verwenden, führt zu einem Fehler der Art „collation "de-x-icu" for encoding "WIN874" does not exist“.

#### 23.2.2.3. Neue Collation-Objekte erstellen

Wenn die Standard- und vordefinierten Collations nicht ausreichen, können Benutzer mit dem SQL-Befehl `CREATE COLLATION` eigene Collation-Objekte erstellen.

Die Standard- und vordefinierten Collations liegen wie alle vordefinierten Objekte im Schema `pg_catalog`. Benutzerdefinierte Collations sollten in Benutzerschemata erstellt werden. Dadurch wird auch sichergestellt, dass sie von `pg_dump` gesichert werden.

##### 23.2.2.3.1. libc-Collations

Neue `libc`-Collations können so erstellt werden:

```sql
CREATE COLLATION german (provider = libc, locale = 'de_DE');
```

Welche genauen Werte in der `locale`-Klausel dieses Befehls akzeptiert werden, hängt vom Betriebssystem ab. Auf Unix-artigen Systemen zeigt der Befehl `locale -a` eine Liste.

Da die vordefinierten `libc`-Collations bereits alle Collations enthalten, die beim Initialisieren der Datenbankinstanz im Betriebssystem definiert waren, ist es nicht oft nötig, manuell neue zu erstellen. Gründe dafür könnten ein anderes gewünschtes Namenssystem sein (siehe dazu auch Abschnitt 23.2.2.3.3) oder ein Betriebssystem-Upgrade, das neue Locale-Definitionen bereitstellt (siehe dazu auch `pg_import_system_collations()`).

##### 23.2.2.3.2. ICU-Collations

ICU-Collations können so erstellt werden:

```sql
CREATE COLLATION german (provider = icu, locale = 'de-DE');
```

ICU-Locales werden als BCP-47-Sprach-Tag angegeben, können aber auch die meisten `libc`-artigen Locale-Namen akzeptieren. Wenn möglich, werden `libc`-artige Locale-Namen in Sprach-Tags umgewandelt.

Neue ICU-Collations können das Collation-Verhalten umfassend anpassen, indem Collation-Attribute in den Sprach-Tag aufgenommen werden. Details und Beispiele finden Sie in Abschnitt 23.2.3.

##### 23.2.2.3.3. Collations kopieren

Der Befehl `CREATE COLLATION` kann auch verwendet werden, um eine neue Collation aus einer bestehenden Collation zu erstellen. Das kann nützlich sein, um betriebssystemunabhängige Collation-Namen in Anwendungen zu verwenden, Kompatibilitätsnamen zu erzeugen oder eine von ICU bereitgestellte Collation unter einem besser lesbaren Namen zu verwenden. Zum Beispiel:

```sql
CREATE COLLATION german FROM "de_DE";
CREATE COLLATION french FROM "fr-x-icu";
```

#### 23.2.2.4. Nichtdeterministische Collations

Eine Collation ist entweder deterministisch oder nichtdeterministisch. Eine deterministische Collation verwendet deterministische Vergleiche; das bedeutet, dass sie Zeichenketten nur dann als gleich betrachtet, wenn sie aus derselben Bytefolge bestehen. Ein nichtdeterministischer Vergleich kann Zeichenketten auch dann als gleich bestimmen, wenn sie aus unterschiedlichen Bytes bestehen. Typische Situationen sind Groß-/Kleinschreibungs-unempfindliche Vergleiche, akzentunempfindliche Vergleiche sowie der Vergleich von Zeichenketten in verschiedenen Unicode-Normalformen. Es liegt beim Collation-Provider, solche unempfindlichen Vergleiche tatsächlich zu implementieren; das Flag `deterministic` bestimmt nur, ob Gleichstände durch byteweisen Vergleich aufgelöst werden. Weitere Informationen zur Terminologie finden Sie auch in Unicode Technical Standard 10.

Um eine nichtdeterministische Collation zu erstellen, geben Sie für `CREATE COLLATION` die Eigenschaft `deterministic = false` an, zum Beispiel:

```sql
CREATE COLLATION ndcoll (provider = icu, locale = 'und',
 deterministic = false);
```

Dieses Beispiel würde die Standard-Unicode-Collation auf nichtdeterministische Weise verwenden. Insbesondere könnten dadurch Zeichenketten in verschiedenen Normalformen korrekt verglichen werden. Interessantere Beispiele nutzen die oben erläuterten ICU-Anpassungsmöglichkeiten. Zum Beispiel:

```sql
CREATE COLLATION case_insensitive (provider = icu, locale = 'und-u-
ks-level2', deterministic = false);
CREATE COLLATION ignore_accents (provider = icu, locale = 'und-u-
ks-level1-kc-true', deterministic = false);
```

Alle Standard- und vordefinierten Collations sind deterministisch; alle benutzerdefinierten Collations sind standardmäßig deterministisch. Nichtdeterministische Collations liefern zwar ein „korrekteres“ Verhalten, insbesondere wenn man die ganze Stärke von Unicode und seine vielen Sonderfälle berücksichtigt, sie haben aber auch einige Nachteile. Vor allem führt ihre Verwendung zu Leistungseinbußen. Beachten Sie insbesondere, dass B-Baum-Indizes keine Deduplizierung mit Indizes verwenden können, die eine nichtdeterministische Collation nutzen. Außerdem sind bestimmte Operationen mit nichtdeterministischen Collations nicht möglich, zum Beispiel einige Mustervergleichsoperationen. Daher sollten sie nur in Fällen verwendet werden, in denen sie ausdrücklich gewünscht sind.

> Tipp: Um mit Text in verschiedenen Unicode-Normalformen umzugehen, ist es auch möglich, die Funktionen/Ausdrücke `normalize` und `is normalized` zu verwenden, um Zeichenketten vorzuverarbeiten oder zu prüfen, statt nichtdeterministische Collations zu verwenden. Für jeden Ansatz gibt es unterschiedliche Abwägungen.

### 23.2.3. Benutzerdefinierte ICU-Collations

ICU erlaubt umfangreiche Kontrolle über das Collation-Verhalten, indem neue Collations mit Collation-Einstellungen als Teil des Sprach-Tags definiert werden. Diese Einstellungen können die Sortierreihenfolge an unterschiedliche Anforderungen anpassen. Zum Beispiel:

```sql
-- ignore differences in accents and case
CREATE COLLATION ignore_accent_case (provider = icu, deterministic
 = false, locale = 'und-u-ks-level1');
SELECT 'Å' = 'A' COLLATE ignore_accent_case; -- true
SELECT 'z' = 'Z' COLLATE ignore_accent_case; -- true

-- upper case letters sort before lower case.
CREATE COLLATION upper_first (provider = icu, locale = 'und-u-kf-
upper');
SELECT 'B' < 'b' COLLATE upper_first; -- true

-- treat digits numerically and ignore punctuation
CREATE COLLATION num_ignore_punct (provider = icu, deterministic =
 false, locale = 'und-u-ka-shifted-kn');
SELECT 'id-45' < 'id-123' COLLATE num_ignore_punct; -- true
SELECT 'w;x*y-z' = 'wxyz' COLLATE num_ignore_punct; -- true
```

Viele der verfügbaren Optionen werden in Abschnitt 23.2.3.2 beschrieben; weitere Details finden Sie auch in Abschnitt 23.2.3.5.

#### 23.2.3.1. ICU-Vergleichsebenen

Der Vergleich zweier Zeichenketten (Collation) wird in ICU durch einen mehrstufigen Prozess bestimmt, bei dem Textmerkmale in „Ebenen“ gruppiert werden. Die Behandlung jeder Ebene wird durch die Collation-Einstellungen gesteuert. Höhere Ebenen entsprechen feineren Textmerkmalen.

Tabelle 23.1 zeigt, welche Unterschiede in Textmerkmalen bei der Bestimmung von Gleichheit auf der jeweiligen Ebene als signifikant gelten. Das Unicode-Zeichen `U+2063` ist ein unsichtbarer Trenner und wird, wie in der Tabelle zu sehen, auf allen Vergleichsebenen unterhalb von `identic` ignoriert.

Tabelle 23.1. ICU-Collation-Ebenen

| Ebene | Beschreibung | `'f' = 'f'` | `'ab' = U&'a\2063b'` | `'x-y' = 'x_y'` | `'g' = 'G'` | `'n' = 'ñ'` | `'y' = 'z'` |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `level1` | Grundzeichen | true | true | true | true | true | false |
| `level2` | Akzente | true | true | true | true | false | false |
| `level3` | Groß-/Kleinschreibung/Varianten | true | true | true | false | false | false |
| `level4` | Interpunktion | true | true | false | false | false | false |
| `identic` | Alles | true | false | false | false | false | false |

Bei `level3` gilt die Interpunktionsbehandlung nur mit `ka-shifted`; siehe Tabelle 23.2.

Auf jeder Ebene wird auch bei ausgeschalteter vollständiger Normalisierung eine grundlegende Normalisierung ausgeführt. Zum Beispiel kann `'á'` aus den Codepunkten `U&'\0061\0301'` oder aus dem einzelnen Codepunkt `U&'\00E1'` bestehen, und diese Sequenzen werden selbst auf der Ebene `identic` als gleich betrachtet. Um jeden Unterschied in der Codepunktdarstellung als verschieden zu behandeln, verwenden Sie eine Collation, die mit `deterministic` auf `true` erstellt wurde.

##### 23.2.3.1.1. Beispiele zu Collation-Ebenen

```sql
CREATE COLLATION level3 (provider = icu, deterministic = false,
 locale = 'und-u-ka-shifted-ks-level3');
CREATE COLLATION level4 (provider = icu, deterministic = false,
 locale = 'und-u-ka-shifted-ks-level4');
CREATE COLLATION identic (provider = icu, deterministic = false,
 locale = 'und-u-ka-shifted-ks-identic');

-- invisible separator ignored at all levels except identic
SELECT 'ab' = U&'a\2063b' COLLATE level4; -- true
SELECT 'ab' = U&'a\2063b' COLLATE identic; -- false

-- punctuation ignored at level3 but not at level 4
SELECT 'x-y' = 'x_y' COLLATE level3; -- true
SELECT 'x-y' = 'x_y' COLLATE level4; -- false
```

#### 23.2.3.2. Collation-Einstellungen für eine ICU-Locale

Tabelle 23.2 zeigt die verfügbaren Collation-Einstellungen, die als Teil eines Sprach-Tags verwendet werden können, um eine Collation anzupassen.

Tabelle 23.2. ICU-Collation-Einstellungen

| Schlüssel | Werte | Standard | Beschreibung |
| --- | --- | --- | --- |
| `co` | `emoji`, `phonebk`, `standard`, ... | `standard` | Collation-Typ. Weitere Optionen und Details finden Sie in Abschnitt 23.2.3.5. |
| `ka` | `noignore`, `shifted` | `noignore` | Wenn auf `shifted` gesetzt, werden einige Zeichen (z. B. Interpunktion oder Leerzeichen) beim Vergleich ignoriert. Der Schlüssel `ks` muss auf `level3` oder niedriger gesetzt sein, damit dies wirksam wird. Mit `kv` wird gesteuert, welche Zeichenklassen ignoriert werden. |
| `kb` | `true`, `false` | `false` | Rückwärtsvergleich für Unterschiede auf Ebene 2. Zum Beispiel sortiert die Locale `und-u-kb` `'àe'` vor `'aé'`. |
| `kc` | `true`, `false` | `false` | Trennt Groß-/Kleinschreibung in eine „Ebene 2.5“, die zwischen Akzenten und anderen Merkmalen der Ebene 3 liegt. Wenn auf `true` gesetzt und `ks` auf `level1` steht, werden Akzente ignoriert, Groß-/Kleinschreibung aber berücksichtigt. |
| `kf` | `upper`, `lower`, `false` | `false` | Bei `upper` sortieren Großbuchstaben vor Kleinbuchstaben. Bei `lower` sortieren Kleinbuchstaben vor Großbuchstaben. Bei `false` hängt die Sortierung von den Regeln der Locale ab. |
| `kn` | `true`, `false` | `false` | Wenn auf `true` gesetzt, werden Zahlen innerhalb einer Zeichenkette als einzelner numerischer Wert und nicht als Ziffernfolge behandelt. Zum Beispiel sortiert `'id-45'` vor `'id-123'`. |
| `kk` | `true`, `false` | `false` | Aktiviert vollständige Normalisierung; dies kann die Leistung beeinflussen. Grundlegende Normalisierung wird auch ausgeführt, wenn der Wert `false` ist. Locales für Sprachen, die vollständige Normalisierung benötigen, aktivieren sie typischerweise standardmäßig. |
| `kr` | `space`, `punct`, `symbol`, `currency`, `digit`, `script-id` |  | Setzen Sie den Wert auf einen oder mehrere gültige Werte oder auf eine BCP-47-Script-ID, z. B. `latn` („Latin“) oder `grek` („Greek“). Mehrere Werte werden durch `-` getrennt. Dadurch wird die Reihenfolge von Zeichenklassen neu definiert; Zeichen, die zu einer früher in der Liste stehenden Klasse gehören, sortieren vor Zeichen einer später stehenden Klasse. |
| `ks` | `level1`, `level2`, `level3`, `level4`, `identic` | `level3` | Empfindlichkeit („strength“) bei der Bestimmung von Gleichheit, wobei `level1` am wenigsten empfindlich und `identic` am empfindlichsten gegenüber Unterschieden ist. Details siehe Tabelle 23.1. |
| `kv` | `space`, `punct`, `symbol`, `currency` | `punct` | Klassen von Zeichen, die beim Vergleich auf Ebene 3 ignoriert werden. Ein späterer Wert schließt frühere Werte ein; z. B. schließt `symbol` auch `punct` und `space` in die zu ignorierenden Zeichen ein. `ka` muss auf `shifted` und `ks` auf `level3` oder niedriger gesetzt sein, damit dies wirksam wird. |

Standardwerte können von der Locale abhängen. Die obige Tabelle ist nicht als vollständig zu verstehen. Weitere Optionen und Details finden Sie in Abschnitt 23.2.3.5.

> Hinweis: Bei vielen Collation-Einstellungen müssen Sie die Collation mit `deterministic` auf `false` erstellen, damit die Einstellung die gewünschte Wirkung hat (siehe Abschnitt 23.2.2.4). Außerdem werden manche Einstellungen nur wirksam, wenn der Schlüssel `ka` auf `shifted` gesetzt ist (siehe Tabelle 23.2).

#### 23.2.3.3. Beispiele für Collation-Einstellungen

```sql
CREATE COLLATION "de-u-co-phonebk-x-icu" (provider = icu, locale =
'de-u-co-phonebk');
```

Deutsche Collation mit Telefonbuch-Collation-Typ.

```sql
CREATE COLLATION "und-u-co-emoji-x-icu" (provider = icu, locale =
'und-u-co-emoji');
```

Root-Collation mit Emoji-Collation-Typ, gemäß Unicode Technical Standard #51.

```sql
CREATE COLLATION latinlast (provider = icu, locale = 'en-u-kr-grek-
latn');
```

Sortiert griechische Buchstaben vor lateinischen. (Standardmäßig kommt Latein vor Griechisch.)

```sql
CREATE COLLATION upperfirst (provider = icu, locale = 'en-u-kf-up-
per');
```

Sortiert Großbuchstaben vor Kleinbuchstaben. (Standardmäßig kommen Kleinbuchstaben zuerst.)

```sql
CREATE COLLATION special (provider = icu, locale = 'en-u-kf-upper-kr-
grek-latn');
```

Kombiniert beide obigen Optionen.

#### 23.2.3.4. ICU-Tailoring-Regeln

Wenn die oben gezeigten Collation-Einstellungen nicht ausreichen, kann die Reihenfolge von Collation-Elementen mit Tailoring-Regeln geändert werden. Deren Syntax ist unter <https://unicode-org.github.io/icu/userguide/collation/customization/> ausführlich beschrieben.

Dieses kleine Beispiel erstellt eine Collation auf Basis der Root-Locale mit einer Tailoring-Regel:

```sql
CREATE COLLATION custom (provider = icu, locale = 'und', rules =
 '&V << w <<< W');
```

Mit dieser Regel wird der Buchstabe „W“ nach „V“ sortiert, aber als sekundärer Unterschied ähnlich wie ein Akzent behandelt. Regeln wie diese sind in den Locale-Definitionen mancher Sprachen enthalten. (Wenn eine Locale-Definition die gewünschten Regeln bereits enthält, müssen sie natürlich nicht noch einmal ausdrücklich angegeben werden.)

Hier ist ein komplexeres Beispiel. Die folgende Anweisung richtet eine Collation namens `ebcdic` mit Regeln ein, die US-ASCII-Zeichen in der Reihenfolge der EBCDIC-Kodierung sortieren.

```sql
CREATE COLLATION ebcdic (provider = icu, locale = 'und',
rules = $$
& ' ' < '.' < '<' < '(' < '+' < \|
< '&' < '!' < '$' < '*' < ')' < ';'
< '-' < '/' < ',' < '%' < '_' < '>' < '?'
< '`' < ':' < '#' < '@' < \' < '=' < '"'
<*a-r < '~' <*s-z < '^' < '[' < ']'
< '{' <*A-I < '}' <*J-R < '\' <*S-Z <*0-9
$$);

SELECT c
FROM (VALUES ('a'), ('b'), ('A'), ('B'), ('1'), ('2'), ('!'),
 ('^')) AS x(c)
ORDER BY c COLLATE ebcdic;
```

```text
---
 !
 a
 b
 ^
 A
 B
```

#### 23.2.3.5. Externe Referenzen für ICU

Dieser Abschnitt (Abschnitt 23.2.3) ist nur ein kurzer Überblick über ICU-Verhalten und Sprach-Tags. Technische Details, zusätzliche Optionen und neues Verhalten finden Sie in den folgenden Dokumenten:

- [Unicode Technical Standard #35](https://www.unicode.org/reports/tr35/tr35-collation.html)

- [BCP 47](https://www.rfc-editor.org/info/bcp47)

- [CLDR-Repository](https://github.com/unicode-org/cldr/blob/master/common/bcp47/collation.xml)

- <https://unicode-org.github.io/icu/userguide/locale/>

- <https://unicode-org.github.io/icu/userguide/collation/>

## 23.3. Zeichensatzunterstützung

Die Zeichensatzunterstützung in PostgreSQL erlaubt es, Text in einer Vielzahl von Zeichensätzen (auch Kodierungen genannt) zu speichern, darunter Ein-Byte-Zeichensätze wie die ISO-8859-Reihe und Mehr-Byte-Zeichensätze wie EUC (Extended Unix Code), UTF-8 und Mule Internal Code. Alle unterstützten Zeichensätze können von Clients transparent verwendet werden, einige werden jedoch nicht für die Verwendung innerhalb des Servers unterstützt (also nicht als serverseitige Kodierung). Der Standardzeichensatz wird beim Initialisieren des PostgreSQL-Datenbankclusters mit `initdb` ausgewählt. Er kann beim Erstellen einer Datenbank überschrieben werden, sodass mehrere Datenbanken mit jeweils unterschiedlichen Zeichensätzen möglich sind.

Eine wichtige Einschränkung ist jedoch, dass der Zeichensatz jeder Datenbank mit den Locale-Einstellungen `LC_CTYPE` (Zeichenklassifikation) und `LC_COLLATE` (Sortierreihenfolge von Zeichenketten) der Datenbank kompatibel sein muss. Für die Locale `C` oder `POSIX` ist jeder Zeichensatz erlaubt, aber für andere von `libc` bereitgestellte Locales gibt es jeweils nur einen Zeichensatz, der korrekt funktioniert. (Unter Windows kann die Kodierung UTF-8 jedoch mit jeder Locale verwendet werden.) Wenn ICU-Unterstützung konfiguriert ist, können von ICU bereitgestellte Locales mit den meisten, aber nicht allen serverseitigen Kodierungen verwendet werden.

### 23.3.1. Unterstützte Zeichensätze

Tabelle 23.3 zeigt die Zeichensätze, die in PostgreSQL verwendet werden können.

Tabelle 23.3. PostgreSQL-Zeichensätze

| Name | Beschreibung | Sprache | Server? | ICU? | Bytes/Zeichen | Aliasse |
| --- | --- | --- | --- | --- | --- | --- |
| `BIG5` | Big Five | Traditionelles Chinesisch | Nein | Nein | 1-2 | `WIN950`, `Windows950` |
| `EUC_CN` | Extended UNIX Code-CN | Vereinfachtes Chinesisch | Ja | Ja | 1-3 |  |
| `EUC_JP` | Extended UNIX Code-JP | Japanisch | Ja | Ja | 1-3 |  |
| `EUC_JIS_2004` | Extended UNIX Code-JP, JIS X 0213 | Japanisch | Ja | Nein | 1-3 |  |
| `EUC_KR` | Extended UNIX Code-KR | Koreanisch | Ja | Ja | 1-3 |  |
| `EUC_TW` | Extended UNIX Code-TW | Traditionelles Chinesisch, Taiwanesisch | Ja | Ja | 1-4 |  |
| `GB18030` | National Standard | Chinesisch | Nein | Nein | 1-4 |  |
| `GBK` | Extended National Standard | Vereinfachtes Chinesisch | Nein | Nein | 1-2 | `WIN936`, `Windows936` |
| `ISO_8859_5` | ISO 8859-5, ECMA 113 | Latein/Kyrillisch | Ja | Ja | 1 |  |
| `ISO_8859_6` | ISO 8859-6, ECMA 114 | Latein/Arabisch | Ja | Ja | 1 |  |
| `ISO_8859_7` | ISO 8859-7, ECMA 118 | Latein/Griechisch | Ja | Ja | 1 |  |
| `ISO_8859_8` | ISO 8859-8, ECMA 121 | Latein/Hebräisch | Ja | Ja | 1 |  |
| `JOHAB` | JOHAB | Koreanisch (Hangul) | Nein | Nein | 1-3 |  |
| `KOI8R` | KOI8-R | Kyrillisch (Russisch) | Ja | Ja | 1 | `KOI8` |
| `KOI8U` | KOI8-U | Kyrillisch (Ukrainisch) | Ja | Ja | 1 |  |
| `LATIN1` | ISO 8859-1, ECMA 94 | Westeuropäisch | Ja | Ja | 1 | `ISO88591` |
| `LATIN2` | ISO 8859-2, ECMA 94 | Mitteleuropäisch | Ja | Ja | 1 | `ISO88592` |
| `LATIN3` | ISO 8859-3, ECMA 94 | Südeuropäisch | Ja | Ja | 1 | `ISO88593` |
| `LATIN4` | ISO 8859-4, ECMA 94 | Nordeuropäisch | Ja | Ja | 1 | `ISO88594` |
| `LATIN5` | ISO 8859-9, ECMA 128 | Türkisch | Ja | Ja | 1 | `ISO88599` |
| `LATIN6` | ISO 8859-10, ECMA 144 | Nordisch | Ja | Ja | 1 | `ISO885910` |
| `LATIN7` | ISO 8859-13 | Baltisch | Ja | Ja | 1 | `ISO885913` |
| `LATIN8` | ISO 8859-14 | Keltisch | Ja | Ja | 1 | `ISO885914` |
| `LATIN9` | ISO 8859-15 | `LATIN1` mit Euro und Akzenten | Ja | Ja | 1 | `ISO885915` |
| `LATIN10` | ISO 8859-16, ASRO SR | Rumänisch | Ja | Nein | 1 | `ISO885916` |
| `MULE_INTERNAL` | Mule Internal Code | Multilingual Emacs | Ja | Nein | 1-4 |  |
| `SJIS` | Shift JIS | Japanisch | Nein | Nein | 1-2 | `Mskanji`, `ShiftJIS`, `WIN932`, `Windows932` |
| `SHIFT_JIS_2004` | Shift JIS, JIS X 0213 | Japanisch | Nein | Nein | 1-2 |  |
| `SQL_ASCII` | nicht spezifiziert (siehe Text) | beliebig | Ja | Nein | 1 |  |
| `UHC` | Unified Hangul Code | Koreanisch | Nein | Nein | 1-2 | `WIN949`, `Windows949` |
| `UTF8` | Unicode, 8-bit | alle | Ja | Ja | 1-4 | `Unicode` |
| `WIN866` | Windows CP866 | Kyrillisch | Ja | Ja | 1 | `ALT` |
| `WIN874` | Windows CP874 | Thai | Ja | Nein | 1 |  |
| `WIN1250` | Windows CP1250 | Mitteleuropäisch | Ja | Ja | 1 |  |
| `WIN1251` | Windows CP1251 | Kyrillisch | Ja | Ja | 1 | `WIN` |
| `WIN1252` | Windows CP1252 | Westeuropäisch | Ja | Ja | 1 |  |
| `WIN1253` | Windows CP1253 | Griechisch | Ja | Ja | 1 |  |
| `WIN1254` | Windows CP1254 | Türkisch | Ja | Ja | 1 |  |
| `WIN1255` | Windows CP1255 | Hebräisch | Ja | Ja | 1 |  |
| `WIN1256` | Windows CP1256 | Arabisch | Ja | Ja | 1 |  |
| `WIN1257` | Windows CP1257 | Baltisch | Ja | Ja | 1 |  |
| `WIN1258` | Windows CP1258 | Vietnamesisch | Ja | Ja | 1 | `ABC`, `TCVN`, `TCVN5712`, `VSCII` |

Nicht alle Client-APIs unterstützen alle aufgeführten Zeichensätze. Der PostgreSQL-JDBC-Treiber unterstützt zum Beispiel `MULE_INTERNAL`, `LATIN6`, `LATIN8` und `LATIN10` nicht.

Die Einstellung `SQL_ASCII` verhält sich deutlich anders als die anderen Einstellungen. Wenn der Serverzeichensatz `SQL_ASCII` ist, interpretiert der Server Bytewerte von 0 bis 127 gemäß dem ASCII-Standard, während Bytewerte von 128 bis 255 als uninterpretiert gelten. Bei der Einstellung `SQL_ASCII` wird keine Kodierungsumsetzung vorgenommen. Diese Einstellung ist daher weniger eine Erklärung, dass eine bestimmte Kodierung verwendet wird, sondern eher eine Erklärung der Unkenntnis über die Kodierung. In den meisten Fällen ist es unklug, `SQL_ASCII` zu verwenden, wenn Sie mit Nicht-ASCII-Daten arbeiten, da PostgreSQL dann nicht helfen kann, Nicht-ASCII-Zeichen umzusetzen oder zu validieren.

### 23.3.2. Zeichensatz festlegen

`initdb` legt den Standardzeichensatz (die Kodierung) für einen PostgreSQL-Cluster fest. Zum Beispiel:

```text
initdb -E EUC_JP
```

setzt den Standardzeichensatz auf `EUC_JP` (Extended Unix Code für Japanisch). Sie können statt `-E` auch `--encoding` verwenden, wenn Sie längere Optionsnamen bevorzugen. Wenn weder `-E` noch `--encoding` angegeben wird, versucht `initdb`, die passende Kodierung anhand der angegebenen oder standardmäßigen Locale zu bestimmen.

Sie können beim Erstellen einer Datenbank eine vom Standard abweichende Kodierung angeben, sofern die Kodierung mit der ausgewählten Locale kompatibel ist:

```text
createdb -E EUC_KR -T template0 --lc-collate=ko_KR.euckr --lc-
ctype=ko_KR.euckr korean
```

Dies erstellt eine Datenbank namens `korean`, die den Zeichensatz `EUC_KR` und die Locale `ko_KR` verwendet. Dasselbe lässt sich auch mit diesem SQL-Befehl erreichen:

```sql
CREATE DATABASE korean WITH ENCODING 'EUC_KR'
 LC_COLLATE='ko_KR.euckr' LC_CTYPE='ko_KR.euckr'
 TEMPLATE=template0;
```

Beachten Sie, dass die obigen Befehle angeben, dass die Datenbank `template0` kopiert wird. Beim Kopieren jeder anderen Datenbank können Kodierungs- und Locale-Einstellungen nicht gegenüber denen der Quelldatenbank geändert werden, weil dies zu beschädigten Daten führen könnte. Weitere Informationen finden Sie in [Abschnitt 22.3](22_Datenbanken_verwalten.md#223-templatedatenbanken).

Die Kodierung einer Datenbank wird im Systemkatalog `pg_database` gespeichert. Sie können sie mit der `psql`-Option `-l` oder mit dem Befehl `\l` anzeigen.

```text
$ psql -l
                                         List of databases
   Name    | Owner    | Encoding | Collation |       Ctype     |
      Access Privileges
-----------+----------+-----------+-------------+-------------
+-------------------------------------
 clocaledb | hlinnaka | SQL_ASCII | C           | C            |
 englishdb | hlinnaka | UTF8      | en_GB.UTF8 | en_GB.UTF8 |
 japanese | hlinnaka | UTF8       | ja_JP.UTF8 | ja_JP.UTF8 |
 korean    | hlinnaka | EUC_KR    | ko_KR.euckr | ko_KR.euckr |
 postgres | hlinnaka | UTF8       | fi_FI.UTF8 | fi_FI.UTF8 |
 template0 | hlinnaka | UTF8      | fi_FI.UTF8 | fi_FI.UTF8 |
 {=c/hlinnaka,hlinnaka=CTc/hlinnaka}
 template1 | hlinnaka | UTF8      | fi_FI.UTF8 | fi_FI.UTF8 |
 {=c/hlinnaka,hlinnaka=CTc/hlinnaka}
(7 rows)
```

> Wichtig: Auf den meisten modernen Betriebssystemen kann PostgreSQL bestimmen, welcher Zeichensatz durch die Einstellung `LC_CTYPE` impliziert wird, und erzwingt, dass nur die passende Datenbankkodierung verwendet wird. Auf älteren Systemen liegt es in Ihrer Verantwortung sicherzustellen, dass Sie die Kodierung verwenden, die von der ausgewählten Locale erwartet wird. Ein Fehler in diesem Bereich führt wahrscheinlich zu merkwürdigem Verhalten locale-abhängiger Operationen wie Sortierung.
>
> PostgreSQL erlaubt Superusern, Datenbanken mit der Kodierung `SQL_ASCII` auch dann zu erstellen, wenn `LC_CTYPE` nicht `C` oder `POSIX` ist. Wie oben erwähnt, erzwingt `SQL_ASCII` nicht, dass die in der Datenbank gespeicherten Daten eine bestimmte Kodierung haben; diese Wahl birgt daher Risiken für locale-abhängiges Fehlverhalten. Die Verwendung dieser Einstellungskombination ist veraltet und kann eines Tages vollständig verboten werden.

### 23.3.3. Automatische Zeichensatzumsetzung zwischen Server und Client

PostgreSQL unterstützt für viele Kombinationen von Zeichensätzen die automatische Zeichensatzumsetzung zwischen Server und Client (Abschnitt 23.3.4 zeigt, welche).

Um die automatische Zeichensatzumsetzung zu aktivieren, müssen Sie PostgreSQL mitteilen, welchen Zeichensatz (welche Kodierung) Sie im Client verwenden möchten. Dafür gibt es mehrere Möglichkeiten:

- Verwenden Sie den Befehl `\encoding` in `psql`. Mit `\encoding` können Sie die Client-Kodierung während der Sitzung ändern. Um die Kodierung zum Beispiel auf `SJIS` zu ändern, geben Sie ein:

```text
\encoding SJIS
```

- `libpq` ([Abschnitt 32.11](32_libpq_C_Bibliothek.md#3211-steuerfunktionen)) verfügt über Funktionen zur Steuerung der Client-Kodierung.

- Verwenden Sie `SET client_encoding TO`. Die Client-Kodierung kann mit diesem SQL-Befehl gesetzt werden:

```sql
SET CLIENT_ENCODING TO 'value';
```

Außerdem können Sie dafür die Standard-SQL-Syntax `SET NAMES` verwenden:

```sql
SET NAMES 'value';
```

Um die aktuelle Client-Kodierung abzufragen:

```sql
SHOW client_encoding;
```

Um zur Standardkodierung zurückzukehren:

```sql
RESET client_encoding;
```

- Verwenden Sie `PGCLIENTENCODING`. Wenn die Umgebungsvariable `PGCLIENTENCODING` in der Umgebung des Clients definiert ist, wird diese Client-Kodierung automatisch ausgewählt, wenn eine Verbindung zum Server hergestellt wird. (Sie kann anschließend mit einer der oben genannten Methoden überschrieben werden.)

- Verwenden Sie die Konfigurationsvariable `client_encoding`. Wenn die Variable `client_encoding` gesetzt ist, wird diese Client-Kodierung automatisch ausgewählt, wenn eine Verbindung zum Server hergestellt wird. (Sie kann anschließend mit einer der oben genannten Methoden überschrieben werden.)

Wenn die Umsetzung eines bestimmten Zeichens nicht möglich ist – nehmen wir an, Sie haben `EUC_JP` für den Server und `LATIN1` für den Client gewählt, und es werden japanische Zeichen zurückgegeben, die in `LATIN1` keine Darstellung haben –, wird ein Fehler gemeldet.

Wenn der Client-Zeichensatz als `SQL_ASCII` definiert ist, wird die Kodierungsumsetzung unabhängig vom Zeichensatz des Servers deaktiviert. (Wenn der Zeichensatz des Servers jedoch nicht `SQL_ASCII` ist, prüft der Server weiterhin, ob eingehende Daten für diese Kodierung gültig sind; der Nettoeffekt ist also so, als wäre der Client-Zeichensatz derselbe wie der des Servers.) Wie beim Server ist die Verwendung von `SQL_ASCII` unklug, sofern Sie nicht ausschließlich mit ASCII-Daten arbeiten.

### 23.3.4. Verfügbare Zeichensatzumsetzungen

PostgreSQL erlaubt die Umsetzung zwischen beliebigen zwei Zeichensätzen, für die im Systemkatalog `pg_conversion` eine Umsetzungsfunktion eingetragen ist. PostgreSQL bringt einige vordefinierte Umsetzungen mit, zusammengefasst in Tabelle 23.4 und detaillierter dargestellt in Tabelle 23.5. Sie können mit dem SQL-Befehl `CREATE CONVERSION` eine neue Umsetzung erstellen. (Damit sie für automatische Client/Server-Umsetzungen verwendet werden kann, muss eine Umsetzung für ihr Zeichensatzpaar als „default“ markiert sein.)

Tabelle 23.4. Eingebaute Client/Server-Zeichensatzumsetzungen

| Serverzeichensatz | Verfügbare Client-Zeichensätze |
| --- | --- |
| `BIG5` | nicht als Serverkodierung unterstützt |
| `EUC_CN` | `EUC_CN`, `MULE_INTERNAL`, `UTF8` |
| `EUC_JP` | `EUC_JP`, `MULE_INTERNAL`, `SJIS`, `UTF8` |
| `EUC_JIS_2004` | `EUC_JIS_2004`, `SHIFT_JIS_2004`, `UTF8` |
| `EUC_KR` | `EUC_KR`, `MULE_INTERNAL`, `UTF8` |
| `EUC_TW` | `EUC_TW`, `BIG5`, `MULE_INTERNAL`, `UTF8` |
| `GB18030` | nicht als Serverkodierung unterstützt |
| `GBK` | nicht als Serverkodierung unterstützt |
| `ISO_8859_5` | `ISO_8859_5`, `KOI8R`, `MULE_INTERNAL`, `UTF8`, `WIN866`, `WIN1251` |
| `ISO_8859_6` | `ISO_8859_6`, `UTF8` |
| `ISO_8859_7` | `ISO_8859_7`, `UTF8` |
| `ISO_8859_8` | `ISO_8859_8`, `UTF8` |
| `JOHAB` | nicht als Serverkodierung unterstützt |
| `KOI8R` | `KOI8R`, `ISO_8859_5`, `MULE_INTERNAL`, `UTF8`, `WIN866`, `WIN1251` |
| `KOI8U` | `KOI8U`, `UTF8` |
| `LATIN1` | `LATIN1`, `MULE_INTERNAL`, `UTF8` |
| `LATIN2` | `LATIN2`, `MULE_INTERNAL`, `UTF8`, `WIN1250` |
| `LATIN3` | `LATIN3`, `MULE_INTERNAL`, `UTF8` |
| `LATIN4` | `LATIN4`, `MULE_INTERNAL`, `UTF8` |
| `LATIN5` | `LATIN5`, `UTF8` |
| `LATIN6` | `LATIN6`, `UTF8` |
| `LATIN7` | `LATIN7`, `UTF8` |
| `LATIN8` | `LATIN8`, `UTF8` |
| `LATIN9` | `LATIN9`, `UTF8` |
| `LATIN10` | `LATIN10`, `UTF8` |
| `MULE_INTERNAL` | `MULE_INTERNAL`, `BIG5`, `EUC_CN`, `EUC_JP`, `EUC_KR`, `EUC_TW`, `ISO_8859_5`, `KOI8R`, `LATIN1` bis `LATIN4`, `SJIS`, `WIN866`, `WIN1250`, `WIN1251` |
| `SJIS` | nicht als Serverkodierung unterstützt |
| `SHIFT_JIS_2004` | nicht als Serverkodierung unterstützt |
| `SQL_ASCII` | beliebig (es wird keine Umsetzung durchgeführt) |
| `UHC` | nicht als Serverkodierung unterstützt |
| `UTF8` | alle unterstützten Kodierungen |
| `WIN866` | `WIN866`, `ISO_8859_5`, `KOI8R`, `MULE_INTERNAL`, `UTF8`, `WIN1251` |
| `WIN874` | `WIN874`, `UTF8` |
| `WIN1250` | `WIN1250`, `LATIN2`, `MULE_INTERNAL`, `UTF8` |
| `WIN1251` | `WIN1251`, `ISO_8859_5`, `KOI8R`, `MULE_INTERNAL`, `UTF8`, `WIN866` |
| `WIN1252` | `WIN1252`, `UTF8` |
| `WIN1253` | `WIN1253`, `UTF8` |
| `WIN1254` | `WIN1254`, `UTF8` |
| `WIN1255` | `WIN1255`, `UTF8` |
| `WIN1256` | `WIN1256`, `UTF8` |
| `WIN1257` | `WIN1257`, `UTF8` |
| `WIN1258` | `WIN1258`, `UTF8` |

Tabelle 23.5. Alle eingebauten Zeichensatzumsetzungen

| Umsetzungsname | Quellkodierung | Zielkodierung |
| --- | --- | --- |
| `big5_to_euc_tw` | `BIG5` | `EUC_TW` |
| `big5_to_mic` | `BIG5` | `MULE_INTERNAL` |
| `big5_to_utf8` | `BIG5` | `UTF8` |
| `euc_cn_to_mic` | `EUC_CN` | `MULE_INTERNAL` |
| `euc_cn_to_utf8` | `EUC_CN` | `UTF8` |
| `euc_jp_to_mic` | `EUC_JP` | `MULE_INTERNAL` |
| `euc_jp_to_sjis` | `EUC_JP` | `SJIS` |
| `euc_jp_to_utf8` | `EUC_JP` | `UTF8` |
| `euc_kr_to_mic` | `EUC_KR` | `MULE_INTERNAL` |
| `euc_kr_to_utf8` | `EUC_KR` | `UTF8` |
| `euc_tw_to_big5` | `EUC_TW` | `BIG5` |
| `euc_tw_to_mic` | `EUC_TW` | `MULE_INTERNAL` |
| `euc_tw_to_utf8` | `EUC_TW` | `UTF8` |
| `gb18030_to_utf8` | `GB18030` | `UTF8` |
| `gbk_to_utf8` | `GBK` | `UTF8` |
| `iso_8859_10_to_utf8` | `LATIN6` | `UTF8` |
| `iso_8859_13_to_utf8` | `LATIN7` | `UTF8` |
| `iso_8859_14_to_utf8` | `LATIN8` | `UTF8` |
| `iso_8859_15_to_utf8` | `LATIN9` | `UTF8` |
| `iso_8859_16_to_utf8` | `LATIN10` | `UTF8` |
| `iso_8859_1_to_mic` | `LATIN1` | `MULE_INTERNAL` |
| `iso_8859_1_to_utf8` | `LATIN1` | `UTF8` |
| `iso_8859_2_to_mic` | `LATIN2` | `MULE_INTERNAL` |
| `iso_8859_2_to_utf8` | `LATIN2` | `UTF8` |
| `iso_8859_2_to_windows_1250` | `LATIN2` | `WIN1250` |
| `iso_8859_3_to_mic` | `LATIN3` | `MULE_INTERNAL` |
| `iso_8859_3_to_utf8` | `LATIN3` | `UTF8` |
| `iso_8859_4_to_mic` | `LATIN4` | `MULE_INTERNAL` |
| `iso_8859_4_to_utf8` | `LATIN4` | `UTF8` |
| `iso_8859_5_to_koi8_r` | `ISO_8859_5` | `KOI8R` |
| `iso_8859_5_to_mic` | `ISO_8859_5` | `MULE_INTERNAL` |
| `iso_8859_5_to_utf8` | `ISO_8859_5` | `UTF8` |
| `iso_8859_5_to_windows_1251` | `ISO_8859_5` | `WIN1251` |
| `iso_8859_5_to_windows_866` | `ISO_8859_5` | `WIN866` |
| `iso_8859_6_to_utf8` | `ISO_8859_6` | `UTF8` |
| `iso_8859_7_to_utf8` | `ISO_8859_7` | `UTF8` |
| `iso_8859_8_to_utf8` | `ISO_8859_8` | `UTF8` |
| `iso_8859_9_to_utf8` | `LATIN5` | `UTF8` |
| `johab_to_utf8` | `JOHAB` | `UTF8` |
| `koi8_r_to_iso_8859_5` | `KOI8R` | `ISO_8859_5` |
| `koi8_r_to_mic` | `KOI8R` | `MULE_INTERNAL` |
| `koi8_r_to_utf8` | `KOI8R` | `UTF8` |
| `koi8_r_to_windows_1251` | `KOI8R` | `WIN1251` |
| `koi8_r_to_windows_866` | `KOI8R` | `WIN866` |
| `koi8_u_to_utf8` | `KOI8U` | `UTF8` |
| `mic_to_big5` | `MULE_INTERNAL` | `BIG5` |
| `mic_to_euc_cn` | `MULE_INTERNAL` | `EUC_CN` |
| `mic_to_euc_jp` | `MULE_INTERNAL` | `EUC_JP` |
| `mic_to_euc_kr` | `MULE_INTERNAL` | `EUC_KR` |
| `mic_to_euc_tw` | `MULE_INTERNAL` | `EUC_TW` |
| `mic_to_iso_8859_1` | `MULE_INTERNAL` | `LATIN1` |
| `mic_to_iso_8859_2` | `MULE_INTERNAL` | `LATIN2` |
| `mic_to_iso_8859_3` | `MULE_INTERNAL` | `LATIN3` |
| `mic_to_iso_8859_4` | `MULE_INTERNAL` | `LATIN4` |
| `mic_to_iso_8859_5` | `MULE_INTERNAL` | `ISO_8859_5` |
| `mic_to_koi8_r` | `MULE_INTERNAL` | `KOI8R` |
| `mic_to_sjis` | `MULE_INTERNAL` | `SJIS` |
| `mic_to_windows_1250` | `MULE_INTERNAL` | `WIN1250` |
| `mic_to_windows_1251` | `MULE_INTERNAL` | `WIN1251` |
| `mic_to_windows_866` | `MULE_INTERNAL` | `WIN866` |
| `sjis_to_euc_jp` | `SJIS` | `EUC_JP` |
| `sjis_to_mic` | `SJIS` | `MULE_INTERNAL` |
| `sjis_to_utf8` | `SJIS` | `UTF8` |
| `windows_1258_to_utf8` | `WIN1258` | `UTF8` |
| `uhc_to_utf8` | `UHC` | `UTF8` |
| `utf8_to_big5` | `UTF8` | `BIG5` |
| `utf8_to_euc_cn` | `UTF8` | `EUC_CN` |
| `utf8_to_euc_jp` | `UTF8` | `EUC_JP` |
| `utf8_to_euc_kr` | `UTF8` | `EUC_KR` |
| `utf8_to_euc_tw` | `UTF8` | `EUC_TW` |
| `utf8_to_gb18030` | `UTF8` | `GB18030` |
| `utf8_to_gbk` | `UTF8` | `GBK` |
| `utf8_to_iso_8859_1` | `UTF8` | `LATIN1` |
| `utf8_to_iso_8859_10` | `UTF8` | `LATIN6` |
| `utf8_to_iso_8859_13` | `UTF8` | `LATIN7` |
| `utf8_to_iso_8859_14` | `UTF8` | `LATIN8` |
| `utf8_to_iso_8859_15` | `UTF8` | `LATIN9` |
| `utf8_to_iso_8859_16` | `UTF8` | `LATIN10` |
| `utf8_to_iso_8859_2` | `UTF8` | `LATIN2` |
| `utf8_to_iso_8859_3` | `UTF8` | `LATIN3` |
| `utf8_to_iso_8859_4` | `UTF8` | `LATIN4` |
| `utf8_to_iso_8859_5` | `UTF8` | `ISO_8859_5` |
| `utf8_to_iso_8859_6` | `UTF8` | `ISO_8859_6` |
| `utf8_to_iso_8859_7` | `UTF8` | `ISO_8859_7` |
| `utf8_to_iso_8859_8` | `UTF8` | `ISO_8859_8` |
| `utf8_to_iso_8859_9` | `UTF8` | `LATIN5` |
| `utf8_to_johab` | `UTF8` | `JOHAB` |
| `utf8_to_koi8_r` | `UTF8` | `KOI8R` |
| `utf8_to_koi8_u` | `UTF8` | `KOI8U` |
| `utf8_to_sjis` | `UTF8` | `SJIS` |
| `utf8_to_windows_1258` | `UTF8` | `WIN1258` |
| `utf8_to_uhc` | `UTF8` | `UHC` |
| `utf8_to_windows_1250` | `UTF8` | `WIN1250` |
| `utf8_to_windows_1251` | `UTF8` | `WIN1251` |
| `utf8_to_windows_1252` | `UTF8` | `WIN1252` |
| `utf8_to_windows_1253` | `UTF8` | `WIN1253` |
| `utf8_to_windows_1254` | `UTF8` | `WIN1254` |
| `utf8_to_windows_1255` | `UTF8` | `WIN1255` |
| `utf8_to_windows_1256` | `UTF8` | `WIN1256` |
| `utf8_to_windows_1257` | `UTF8` | `WIN1257` |
| `utf8_to_windows_866` | `UTF8` | `WIN866` |
| `utf8_to_windows_874` | `UTF8` | `WIN874` |
| `windows_1250_to_iso_8859_2` | `WIN1250` | `LATIN2` |
| `windows_1250_to_mic` | `WIN1250` | `MULE_INTERNAL` |
| `windows_1250_to_utf8` | `WIN1250` | `UTF8` |
| `windows_1251_to_iso_8859_5` | `WIN1251` | `ISO_8859_5` |
| `windows_1251_to_koi8_r` | `WIN1251` | `KOI8R` |
| `windows_1251_to_mic` | `WIN1251` | `MULE_INTERNAL` |
| `windows_1251_to_utf8` | `WIN1251` | `UTF8` |
| `windows_1251_to_windows_866` | `WIN1251` | `WIN866` |
| `windows_1252_to_utf8` | `WIN1252` | `UTF8` |
| `windows_1256_to_utf8` | `WIN1256` | `UTF8` |
| `windows_866_to_iso_8859_5` | `WIN866` | `ISO_8859_5` |
| `windows_866_to_koi8_r` | `WIN866` | `KOI8R` |
| `windows_866_to_mic` | `WIN866` | `MULE_INTERNAL` |
| `windows_866_to_utf8` | `WIN866` | `UTF8` |
| `windows_866_to_windows_1251` | `WIN866` | `WIN1251` |
| `windows_874_to_utf8` | `WIN874` | `UTF8` |
| `euc_jis_2004_to_utf8` | `EUC_JIS_2004` | `UTF8` |
| `utf8_to_euc_jis_2004` | `UTF8` | `EUC_JIS_2004` |
| `shift_jis_2004_to_utf8` | `SHIFT_JIS_2004` | `UTF8` |
| `utf8_to_shift_jis_2004` | `UTF8` | `SHIFT_JIS_2004` |
| `euc_jis_2004_to_shift_jis_2004` | `EUC_JIS_2004` | `SHIFT_JIS_2004` |
| `shift_jis_2004_to_euc_jis_2004` | `SHIFT_JIS_2004` | `EUC_JIS_2004` |

Die Umsetzungsnamen folgen einem Standardschema: dem offiziellen Namen der Quellkodierung, wobei alle nicht alphanumerischen Zeichen durch Unterstriche ersetzt werden, gefolgt von `_to_`, gefolgt vom entsprechend verarbeiteten Namen der Zielkodierung. Daher weichen diese Namen manchmal von den gebräuchlichen Kodierungsnamen in Tabelle 23.3 ab.

### 23.3.5. Weiterführende Literatur

Dies sind gute Quellen, um sich in verschiedene Arten von Kodierungssystemen einzuarbeiten.

`CJKV Information Processing: Chinese, Japanese, Korean & Vietnamese Computing`

Enthält ausführliche Erklärungen zu `EUC_JP`, `EUC_CN`, `EUC_KR` und `EUC_TW`.

<https://www.unicode.org/>

Die Website des Unicode Consortium.

[RFC 3629](https://datatracker.ietf.org/doc/html/rfc3629)

UTF-8 (8-bit UCS/Unicode Transformation Format) ist dort definiert.
