# 12. Volltextsuche

## 12.1. Einführung

Volltextsuche, kurz auch Textsuche, bietet die Möglichkeit, natürlichsprachliche Dokumente zu identifizieren, die eine Abfrage erfüllen, und sie optional nach ihrer Relevanz für diese Abfrage zu sortieren. Die häufigste Suchart besteht darin, alle Dokumente zu finden, die bestimmte Suchbegriffe enthalten, und sie nach ihrer Ähnlichkeit mit der Abfrage zurückzugeben. Die Begriffe Abfrage und Ähnlichkeit sind sehr flexibel und hängen von der jeweiligen Anwendung ab. Die einfachste Suche betrachtet eine Abfrage als Menge von Wörtern und Ähnlichkeit als Häufigkeit dieser Wörter im Dokument.

Textuelle Suchoperatoren gibt es in Datenbanken seit vielen Jahren. PostgreSQL besitzt für textuelle Datentypen die Operatoren `~`, `~*`, `LIKE` und `ILIKE`, ihnen fehlen jedoch viele wesentliche Eigenschaften, die moderne Informationssysteme benötigen:

- Es gibt keine linguistische Unterstützung, nicht einmal für Englisch. Reguläre Ausdrücke reichen nicht aus, weil sie abgeleitete Wörter wie `satisfies` und `satisfy` nicht einfach behandeln können. Man könnte Dokumente übersehen, die `satisfies` enthalten, obwohl man sie bei einer Suche nach `satisfy` wahrscheinlich finden möchte. Es ist möglich, mit `OR` nach mehreren abgeleiteten Formen zu suchen, aber das ist mühsam und fehleranfällig; manche Wörter haben mehrere Tausend Ableitungen.
- Es gibt keine Sortierung, also kein Ranking, der Suchergebnisse. Dadurch werden diese Operatoren ineffektiv, wenn Tausende passende Dokumente gefunden werden.
- Sie sind tendenziell langsam, weil es keine Indexunterstützung gibt; daher müssen bei jeder Suche alle Dokumente verarbeitet werden.

Volltextindizierung erlaubt es, Dokumente vorzuverarbeiten und einen Index für spätere schnelle Suchen zu speichern. Die Vorverarbeitung umfasst:

- Das Zerlegen von Dokumenten in Tokens. Es ist nützlich, verschiedene Token-Klassen zu unterscheiden, zum Beispiel Zahlen, Wörter, zusammengesetzte Wörter oder E-Mail-Adressen, damit sie unterschiedlich verarbeitet werden können. Grundsätzlich hängen Token-Klassen von der konkreten Anwendung ab, für die meisten Zwecke reicht jedoch eine vordefinierte Menge von Klassen. PostgreSQL verwendet für diesen Schritt einen Parser. Ein Standardparser wird mitgeliefert, und für besondere Anforderungen können eigene Parser erstellt werden.
- Das Umwandeln von Tokens in Lexeme. Ein Lexem ist wie ein Token eine Zeichenkette, wurde aber normalisiert, sodass verschiedene Formen desselben Wortes gleich werden. Normalisierung umfasst fast immer das Umwandeln von Großbuchstaben in Kleinbuchstaben und häufig das Entfernen von Endungen, etwa `s` oder `es` im Englischen. Dadurch können Suchen Varianten desselben Wortes finden, ohne dass alle Varianten mühsam eingegeben werden müssen. Außerdem entfernt dieser Schritt typischerweise Stoppwörter, also Wörter, die so häufig vorkommen, dass sie für die Suche nutzlos sind. Kurz gesagt: Tokens sind rohe Fragmente des Dokumenttexts, während Lexeme Wörter sind, die für Indexierung und Suche als nützlich gelten. PostgreSQL verwendet für diesen Schritt Wörterbücher. Es werden verschiedene Standardwörterbücher mitgeliefert, und eigene Wörterbücher können bei Bedarf erstellt werden.
- Das Speichern vorverarbeiteter Dokumente in einer für die Suche optimierten Form. Zum Beispiel kann jedes Dokument als sortiertes Array normalisierter Lexeme dargestellt werden. Neben den Lexemen ist es oft sinnvoll, Positionsinformationen zu speichern, um Proximity-Ranking zu ermöglichen, sodass ein Dokument mit einem dichteren Bereich von Suchwörtern höher bewertet wird als eines mit verstreuten Suchwörtern.

Wörterbücher erlauben eine feingranulare Kontrolle darüber, wie Tokens normalisiert werden. Mit passenden Wörterbüchern können Sie:

- Stoppwörter definieren, die nicht indiziert werden sollen.
- Synonyme mithilfe von Ispell auf ein einzelnes Wort abbilden.
- Phrasen mithilfe eines Thesaurus auf ein einzelnes Wort abbilden.
- Verschiedene Varianten eines Wortes mithilfe eines Ispell-Wörterbuchs auf eine kanonische Form abbilden.
- Verschiedene Varianten eines Wortes mithilfe von Snowball-Stemming-Regeln auf eine kanonische Form abbilden.

Für die Speicherung vorverarbeiteter Dokumente stellt PostgreSQL den Datentyp `tsvector` bereit; für verarbeitete Abfragen gibt es den Typ `tsquery` ([Abschnitt 8.11](08_Datentypen.md#811-volltextsuchtypen)). Für diese Datentypen stehen viele Funktionen und Operatoren zur Verfügung ([Abschnitt 9.13](09_Funktionen_und_Operatoren.md#913-volltextsuchfunktionen-und-operatoren)). Der wichtigste davon ist der Match-Operator `@@`, der in [Abschnitt 12.1.2](#1212-einfache-textübereinstimmung) eingeführt wird. Volltextsuchen können mit Indizes beschleunigt werden ([Abschnitt 12.9](#129-bevorzugte-indextypen-für-textsuche)).

### 12.1.1. Was ist ein Dokument?

Ein Dokument ist die Sucheinheit in einem Volltextsuchsystem, zum Beispiel ein Zeitschriftenartikel oder eine E-Mail-Nachricht. Die Textsuchmaschine muss Dokumente parsen und Zuordnungen von Lexemen, also Schlüsselwörtern, zu ihrem jeweiligen Dokument speichern können. Später werden diese Zuordnungen verwendet, um Dokumente zu suchen, die Abfragewörter enthalten.

Für Suchen innerhalb von PostgreSQL ist ein Dokument normalerweise ein Textfeld innerhalb einer Zeile einer Datenbanktabelle oder möglicherweise eine Kombination, also Konkatenation, solcher Felder. Diese Felder können auch in mehreren Tabellen gespeichert oder dynamisch gewonnen werden. Anders gesagt: Ein Dokument kann für die Indexierung aus verschiedenen Teilen zusammengesetzt werden und muss nicht als Ganzes irgendwo gespeichert sein. Zum Beispiel:

```sql
SELECT title || ' ' || author || ' ' || abstract || ' ' || body AS document
FROM messages
WHERE mid = 12;

SELECT m.title || ' ' || m.author || ' ' || m.abstract || ' ' ||
       d.body AS document
FROM messages m, docs d
WHERE m.mid = d.did AND m.mid = 12;
```

> Hinweis: In diesen Beispielabfragen sollte eigentlich `coalesce` verwendet werden, um zu verhindern, dass ein einzelnes `NULL`-Attribut zu einem `NULL`-Ergebnis für das gesamte Dokument führt.

Eine andere Möglichkeit besteht darin, Dokumente als einfache Textdateien im Dateisystem zu speichern. In diesem Fall kann die Datenbank verwendet werden, um den Volltextindex zu speichern und Suchen auszuführen, und ein eindeutiger Bezeichner kann zum Abrufen des Dokuments aus dem Dateisystem dienen. Das Abrufen von Dateien außerhalb der Datenbank erfordert jedoch Superuser-Rechte oder spezielle Funktionsunterstützung und ist daher meist weniger bequem, als alle Daten innerhalb von PostgreSQL zu halten. Außerdem erlaubt es die Speicherung in der Datenbank, einfach auf Dokumentmetadaten zuzugreifen, die bei Indexierung und Anzeige helfen.

Für Zwecke der Textsuche muss jedes Dokument auf das vorverarbeitete Format `tsvector` reduziert werden. Suche und Ranking werden vollständig auf der `tsvector`-Darstellung eines Dokuments ausgeführt; der ursprüngliche Text muss erst abgerufen werden, wenn das Dokument zur Anzeige für einen Benutzer ausgewählt wurde. Daher sprechen wir häufig vom `tsvector` als dem Dokument, obwohl er natürlich nur eine kompakte Darstellung des vollständigen Dokuments ist.

### 12.1.2. Einfache Textübereinstimmung

Die Volltextsuche in PostgreSQL basiert auf dem Match-Operator `@@`, der `true` zurückgibt, wenn ein `tsvector`, also ein Dokument, zu einem `tsquery`, also einer Abfrage, passt. Es spielt keine Rolle, welcher Datentyp zuerst geschrieben wird:

```sql
SELECT 'a fat cat sat on a mat and ate a fat rat'::tsvector @@ 'cat & rat'::tsquery;
```

```text
 ?column?
----------
 t
```

```sql
SELECT 'fat & cow'::tsquery @@ 'a fat cat sat on a mat and ate a fat rat'::tsvector;
```

```text
 ?column?
----------
 f
```

Wie das Beispiel zeigt, ist ein `tsquery` ebenso wenig roher Text wie ein `tsvector`. Ein `tsquery` enthält Suchbegriffe, die bereits normalisierte Lexeme sein müssen, und kann mehrere Begriffe mit den Operatoren `AND`, `OR`, `NOT` und `FOLLOWED BY` kombinieren. Syntaxdetails stehen in [Abschnitt 8.11.2](08_Datentypen.md#8112-tsquery). Die Funktionen `to_tsquery`, `plainto_tsquery` und `phraseto_tsquery` helfen dabei, vom Benutzer geschriebenen Text in einen passenden `tsquery` umzuwandeln, vor allem indem sie die im Text vorkommenden Wörter normalisieren. Entsprechend wird `to_tsvector` verwendet, um eine Dokumentzeichenkette zu parsen und zu normalisieren. In der Praxis sieht eine Textsuche daher eher so aus:

```sql
SELECT to_tsvector('fat cats ate fat rats') @@ to_tsquery('fat & rat');
```

```text
 ?column?
----------
 t
```

Diese Übereinstimmung würde nicht gelingen, wenn sie so geschrieben würde:

```sql
SELECT 'fat cats ate fat rats'::tsvector @@ to_tsquery('fat & rat');
```

```text
 ?column?
----------
 f
```

Hier findet keine Normalisierung des Wortes `rats` statt. Die Elemente eines `tsvector` sind Lexeme, von denen angenommen wird, dass sie bereits normalisiert sind; daher passt `rats` nicht zu `rat`.

Der Operator `@@` unterstützt auch Texteingaben, sodass in einfachen Fällen die explizite Umwandlung einer Textzeichenkette in `tsvector` oder `tsquery` übersprungen werden kann. Die verfügbaren Varianten sind:

```text
tsvector @@ tsquery
tsquery @@ tsvector
text @@ tsquery
text @@ text
```

Die ersten beiden davon haben wir bereits gesehen. Die Form `text @@ tsquery` entspricht `to_tsvector(x) @@ y`. Die Form `text @@ text` entspricht `to_tsvector(x) @@ plainto_tsquery(y)`.

Innerhalb eines `tsquery` legt der Operator `&` (`AND`) fest, dass beide Argumente im Dokument vorkommen müssen, damit eine Übereinstimmung vorliegt. Entsprechend legt `|` (`OR`) fest, dass mindestens eines seiner Argumente vorkommen muss, während `!` (`NOT`) festlegt, dass sein Argument nicht vorkommen darf. Die Abfrage `fat & ! rat` passt zum Beispiel auf Dokumente, die `fat`, aber nicht `rat` enthalten.

Nach Phrasen kann mithilfe des `tsquery`-Operators `<->` (`FOLLOWED BY`) gesucht werden. Er passt nur dann, wenn seine Argumente benachbarte Treffer in der angegebenen Reihenfolge haben. Zum Beispiel:

```sql
SELECT to_tsvector('fatal error') @@ to_tsquery('fatal <-> error');
```

```text
 ?column?
----------
 t
```

```sql
SELECT to_tsvector('error is not fatal') @@ to_tsquery('fatal <-> error');
```

```text
 ?column?
----------
 f
```

Es gibt eine allgemeinere Version des `FOLLOWED BY`-Operators in der Form `<N>`, wobei `N` eine Ganzzahl ist, die für den Abstand zwischen den Positionen der passenden Lexeme steht. `<1>` ist dasselbe wie `<->`, während `<2>` genau ein anderes Lexem zwischen den Treffern erlaubt, und so weiter. Die Funktion `phraseto_tsquery` verwendet diesen Operator, um einen `tsquery` zu erzeugen, der eine Mehrwortphrase auch dann finden kann, wenn einige der Wörter Stoppwörter sind. Zum Beispiel:

```sql
SELECT phraseto_tsquery('cats ate rats');
```

```text
       phraseto_tsquery
-------------------------------
 'cat' <-> 'ate' <-> 'rat'
```

```sql
SELECT phraseto_tsquery('the cats ate the rats');
```

```text
       phraseto_tsquery
-------------------------------
 'cat' <-> 'ate' <2> 'rat'
```

Ein manchmal nützlicher Sonderfall ist `<0>`, womit verlangt werden kann, dass zwei Muster auf dasselbe Wort passen.

Klammern können verwendet werden, um die Verschachtelung der `tsquery`-Operatoren zu steuern. Ohne Klammern bindet `|` am schwächsten, dann `&`, dann `<->`, und `!` am stärksten.

Es ist bemerkenswert, dass die Operatoren `AND`, `OR` und `NOT` innerhalb der Argumente eines `FOLLOWED BY`-Operators eine etwas andere Bedeutung haben als sonst, weil innerhalb von `FOLLOWED BY` die genaue Trefferposition wichtig ist. Normalerweise passt `!x` zum Beispiel nur auf Dokumente, die nirgendwo `x` enthalten. `!x <-> y` passt dagegen auf `y`, wenn es nicht unmittelbar nach einem `x` steht; ein `x` an anderer Stelle im Dokument verhindert die Übereinstimmung nicht. Ein weiteres Beispiel: `x & y` verlangt normalerweise nur, dass `x` und `y` irgendwo im Dokument vorkommen, aber `(x & y) <-> z` verlangt, dass `x` und `y` an derselben Stelle unmittelbar vor einem `z` passen. Diese Abfrage verhält sich daher anders als `x <-> z & y <-> z`, was auf ein Dokument mit zwei getrennten Sequenzen `x z` und `y z` passen würde. Diese konkrete Abfrage ist so geschrieben nutzlos, da `x` und `y` nicht an derselben Stelle passen könnten; bei komplexeren Situationen wie Präfix-Match-Mustern kann eine Abfrage dieser Form jedoch nützlich sein.

### 12.1.3. Konfigurationen

Die bisherigen Beispiele sind einfache Textsuchbeispiele. Wie erwähnt, kann die Volltextsuche viel mehr: bestimmte Wörter, also Stoppwörter, von der Indexierung ausschließen, Synonyme verarbeiten und anspruchsvolleres Parsing verwenden, etwa nicht nur anhand von Leerraum trennen. Diese Funktionalität wird durch Textsuchkonfigurationen gesteuert. PostgreSQL liefert vordefinierte Konfigurationen für viele Sprachen mit, und eigene Konfigurationen lassen sich leicht erstellen. Der `psql`-Befehl `\dF` zeigt alle verfügbaren Konfigurationen.

Bei der Installation wird eine passende Konfiguration ausgewählt und `default_text_search_config` entsprechend in `postgresql.conf` gesetzt. Wenn Sie für den gesamten Cluster dieselbe Textsuchkonfiguration verwenden, können Sie den Wert in `postgresql.conf` nutzen. Um im Cluster unterschiedliche Konfigurationen zu verwenden, aber innerhalb einer einzelnen Datenbank dieselbe Konfiguration, verwenden Sie `ALTER DATABASE ... SET`. Andernfalls können Sie `default_text_search_config` in jeder Sitzung setzen.

Jede Textsuchfunktion, die von einer Konfiguration abhängt, besitzt ein optionales Argument `regconfig`, sodass die zu verwendende Konfiguration ausdrücklich angegeben werden kann. `default_text_search_config` wird nur verwendet, wenn dieses Argument weggelassen wird.

Um das Erstellen eigener Textsuchkonfigurationen zu erleichtern, wird eine Konfiguration aus einfacheren Datenbankobjekten aufgebaut. Die Textsuche von PostgreSQL stellt vier Arten konfigurationsbezogener Datenbankobjekte bereit:

- Textsuchparser zerlegen Dokumente in Tokens und klassifizieren jedes Token, zum Beispiel als Wort oder Zahl.
- Textsuchwörterbücher wandeln Tokens in normalisierte Form um und verwerfen Stoppwörter.
- Textsuchvorlagen stellen die Funktionen bereit, auf denen Wörterbücher beruhen. Ein Wörterbuch gibt lediglich eine Vorlage und eine Menge von Parametern für diese Vorlage an.
- Textsuchkonfigurationen wählen einen Parser und eine Menge von Wörterbüchern aus, mit denen die vom Parser erzeugten Tokens normalisiert werden.

Textsuchparser und -vorlagen werden aus Low-Level-C-Funktionen gebaut; deshalb sind C-Programmierkenntnisse nötig, um neue zu entwickeln, und Superuser-Rechte, um sie in einer Datenbank zu installieren. Beispiele für zusätzliche Parser und Vorlagen befinden sich im Bereich `contrib/` der PostgreSQL-Distribution. Da Wörterbücher und Konfigurationen lediglich einige zugrunde liegende Parser und Vorlagen parametrisieren und verbinden, ist kein besonderes Recht nötig, um ein neues Wörterbuch oder eine neue Konfiguration zu erstellen. Beispiele für eigene Wörterbücher und Konfigurationen erscheinen später in diesem Kapitel.

## 12.2. Tabellen und Indizes

Die Beispiele im vorherigen Abschnitt haben Volltextübereinstimmungen mit einfachen konstanten Zeichenketten gezeigt. Dieser Abschnitt zeigt, wie Tabellendaten durchsucht werden, optional mithilfe von Indizes.

### 12.2.1. Eine Tabelle durchsuchen

Eine Volltextsuche ist auch ohne Index möglich. Eine einfache Abfrage, die den Titel jeder Zeile ausgibt, deren Feld `body` das Wort `friend` enthält, lautet:

```sql
SELECT title
FROM pgweb
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'friend');
```

Diese Abfrage findet auch verwandte Wörter wie `friends` und `friendly`, weil alle auf dasselbe normalisierte Lexem reduziert werden.

Die obige Abfrage gibt an, dass die Konfiguration `english` zum Parsen und Normalisieren der Zeichenketten verwendet werden soll. Alternativ könnten die Konfigurationsparameter weggelassen werden:

```sql
SELECT title
FROM pgweb
WHERE to_tsvector(body) @@ to_tsquery('friend');
```

Diese Abfrage verwendet die durch `default_text_search_config` festgelegte Konfiguration.

Ein komplexeres Beispiel wählt die zehn neuesten Dokumente aus, die `create` und `table` im Titel oder im Textkörper enthalten:

```sql
SELECT title
FROM pgweb
WHERE to_tsvector(title || ' ' || body) @@ to_tsquery('create & table')
ORDER BY last_mod_date DESC
LIMIT 10;
```

Der Übersichtlichkeit halber wurden die `coalesce`-Funktionsaufrufe weggelassen, die nötig wären, um Zeilen zu finden, die in einem der beiden Felder `NULL` enthalten.

Obwohl diese Abfragen ohne Index funktionieren, ist dieser Ansatz für die meisten Anwendungen zu langsam, außer vielleicht bei gelegentlichen Ad-hoc-Suchen. Die praktische Nutzung der Textsuche erfordert normalerweise das Erstellen eines Indexes.

### 12.2.2. Indizes erstellen

Zur Beschleunigung von Textsuchen können wir einen GIN-Index erstellen ([Abschnitt 12.9](#129-bevorzugte-indextypen-für-textsuche)):

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', body));
```

Beachten Sie, dass die Zwei-Argument-Version von `to_tsvector` verwendet wird. In Ausdrucksindizes können nur Textsuchfunktionen verwendet werden, die einen Konfigurationsnamen angeben ([Abschnitt 11.7](11_Indizes.md#117-indizes-auf-ausdrücken)). Der Grund ist, dass der Indexinhalt nicht von `default_text_search_config` beeinflusst werden darf. Wäre das der Fall, könnte der Indexinhalt inkonsistent werden, weil unterschiedliche Einträge `tsvector`-Werte enthalten könnten, die mit unterschiedlichen Textsuchkonfigurationen erzeugt wurden, ohne dass sich später feststellen ließe, welche Konfiguration jeweils verwendet wurde. Ein solcher Index ließe sich nicht korrekt sichern und wiederherstellen.

Da im obigen Index die Zwei-Argument-Version von `to_tsvector` verwendet wurde, kann nur eine Abfragereferenz, die die Zwei-Argument-Version von `to_tsvector` mit demselben Konfigurationsnamen verwendet, diesen Index nutzen. Das heißt, `WHERE to_tsvector('english', body) @@ 'a & b'` kann den Index verwenden, `WHERE to_tsvector(body) @@ 'a & b'` dagegen nicht. Dadurch wird sichergestellt, dass ein Index nur mit derselben Konfiguration verwendet wird, mit der seine Einträge erzeugt wurden.

Es ist möglich, komplexere Ausdrucksindizes einzurichten, bei denen der Konfigurationsname durch eine andere Spalte angegeben wird, zum Beispiel:

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector(config_name, body));
```

Dabei ist `config_name` eine Spalte in der Tabelle `pgweb`. Das erlaubt gemischte Konfigurationen im selben Index und hält zugleich fest, welche Konfiguration für jeden Indexeintrag verwendet wurde. Das wäre zum Beispiel nützlich, wenn die Dokumentensammlung Dokumente in verschiedenen Sprachen enthält. Auch hier müssen Abfragen, die den Index nutzen sollen, passend formuliert sein, etwa `WHERE to_tsvector(config_name, body) @@ 'a & b'`.

Indizes können sogar Spalten konkatenieren:

```sql
CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', title || ' ' || body));
```

Ein anderer Ansatz besteht darin, eine separate `tsvector`-Spalte zu erzeugen, die die Ausgabe von `to_tsvector` enthält. Um diese Spalte automatisch mit ihren Quelldaten aktuell zu halten, verwenden Sie eine gespeicherte generierte Spalte. Dieses Beispiel ist eine Konkatenation von `title` und `body` und verwendet `coalesce`, um sicherzustellen, dass ein Feld auch dann indiziert wird, wenn das andere `NULL` ist:

```sql
ALTER TABLE pgweb
    ADD COLUMN textsearchable_index_col tsvector
               GENERATED ALWAYS AS (to_tsvector('english',
                 coalesce(title, '') || ' ' || coalesce(body, ''))) STORED;
```

Dann erzeugen wir einen GIN-Index, um die Suche zu beschleunigen:

```sql
CREATE INDEX textsearch_idx ON pgweb USING GIN (textsearchable_index_col);
```

Jetzt können wir eine schnelle Volltextsuche ausführen:

```sql
SELECT title
FROM pgweb
WHERE textsearchable_index_col @@ to_tsquery('create & table')
ORDER BY last_mod_date DESC
LIMIT 10;
```

Ein Vorteil des Ansatzes mit separater Spalte gegenüber einem Ausdrucksindex ist, dass in Abfragen nicht ausdrücklich die Textsuchkonfiguration angegeben werden muss, um den Index zu nutzen. Wie das Beispiel zeigt, kann sich die Abfrage auf `default_text_search_config` stützen. Ein weiterer Vorteil ist, dass Suchen schneller sind, weil die `to_tsvector`-Aufrufe zur Prüfung von Indextreffern nicht wiederholt werden müssen. Das ist bei GiST-Indizes wichtiger als bei GIN-Indizes; siehe [Abschnitt 12.9](#129-bevorzugte-indextypen-für-textsuche). Der Ausdrucksindex-Ansatz ist dagegen einfacher einzurichten und benötigt weniger Plattenplatz, weil die `tsvector`-Darstellung nicht ausdrücklich gespeichert wird.

## 12.3. Textsuche steuern

Zur Implementierung von Volltextsuche muss es eine Funktion geben, die aus einem Dokument einen `tsvector` und aus einer Benutzerabfrage einen `tsquery` erzeugt. Außerdem sollen Ergebnisse in einer nützlichen Reihenfolge zurückgegeben werden; dafür wird eine Funktion benötigt, die Dokumente hinsichtlich ihrer Relevanz für die Abfrage vergleicht. Ebenso wichtig ist eine ansprechende Darstellung der Ergebnisse. PostgreSQL unterstützt all diese Funktionen.

### 12.3.1. Dokumente parsen

PostgreSQL stellt die Funktion `to_tsvector` bereit, um ein Dokument in den Datentyp `tsvector` umzuwandeln.

```text
to_tsvector([ config regconfig, ] document text) returns tsvector
```

`to_tsvector` parst ein textuelles Dokument in Tokens, reduziert diese Tokens auf Lexeme und gibt einen `tsvector` zurück, der die Lexeme zusammen mit ihren Positionen im Dokument aufführt. Das Dokument wird nach der angegebenen oder voreingestellten Textsuchkonfiguration verarbeitet. Hier ist ein einfaches Beispiel:

```sql
SELECT to_tsvector('english', 'a fat cat sat on a mat - it ate a fat rats');
```

```text
                  to_tsvector
-----------------------------------------------------
 'ate':9 'cat':3 'fat':2,11 'mat':7 'rat':12 'sat':4
```

Im Beispiel sehen wir, dass der resultierende `tsvector` die Wörter `a`, `on` und `it` nicht enthält, dass aus dem Wort `rats` das Lexem `rat` wurde und dass das Satzzeichen `-` ignoriert wurde.

Die Funktion `to_tsvector` ruft intern einen Parser auf, der den Dokumenttext in Tokens zerlegt und jedem Token einen Typ zuweist. Für jedes Token wird eine Liste von Wörterbüchern ([Abschnitt 12.6](#126-wörterbücher)) konsultiert; diese Liste kann je nach Tokentyp unterschiedlich sein. Das erste Wörterbuch, das das Token erkennt, gibt ein oder mehrere normalisierte Lexeme aus, die das Token repräsentieren. `rats` wurde zum Beispiel zu `rat`, weil eines der Wörterbücher erkannt hat, dass `rats` eine Pluralform von `rat` ist. Einige Wörter werden als Stoppwörter erkannt ([Abschnitt 12.6.1](#1261-stoppwörter)); sie werden ignoriert, weil sie zu häufig vorkommen, um für die Suche nützlich zu sein. In unserem Beispiel sind das `a`, `on` und `it`. Wenn kein Wörterbuch in der Liste das Token erkennt, wird es ebenfalls ignoriert. Im Beispiel geschah das mit dem Satzzeichen `-`, weil seinem Tokentyp, den Leerraumsymbolen, tatsächlich keine Wörterbücher zugeordnet sind; Leerraumtokens werden also nie indiziert. Die Auswahl von Parser, Wörterbüchern und zu indizierenden Tokentypen wird durch die gewählte Textsuchkonfiguration bestimmt ([Abschnitt 12.7](#127-konfigurationsbeispiel)). In derselben Datenbank können viele verschiedene Konfigurationen vorhanden sein, und vordefinierte Konfigurationen gibt es für verschiedene Sprachen. In unserem Beispiel wurde die Standardkonfiguration `english` für die englische Sprache verwendet.

Die Funktion `setweight` kann verwendet werden, um die Einträge eines `tsvector` mit einem bestimmten Gewicht zu markieren, wobei ein Gewicht einer der Buchstaben `A`, `B`, `C` oder `D` ist. Das wird typischerweise genutzt, um Einträge aus unterschiedlichen Teilen eines Dokuments zu kennzeichnen, etwa Titel gegenüber Textkörper. Später kann diese Information beim Ranking von Suchergebnissen verwendet werden.

Da `to_tsvector(NULL)` `NULL` zurückgibt, empfiehlt es sich, `coalesce` zu verwenden, wenn ein Feld null sein könnte. Dies ist die empfohlene Methode, um aus einem strukturierten Dokument einen `tsvector` zu erzeugen:

```sql
UPDATE tt SET ti =
    setweight(to_tsvector(coalesce(title,'')), 'A')    ||
    setweight(to_tsvector(coalesce(keyword,'')), 'B')  ||
    setweight(to_tsvector(coalesce(abstract,'')), 'C') ||
    setweight(to_tsvector(coalesce(body,'')), 'D');
```

Hier haben wir `setweight` verwendet, um die Quelle jedes Lexems im fertigen `tsvector` zu markieren, und die markierten `tsvector`-Werte anschließend mit dem Verkettungsoperator `||` für `tsvector` zusammengeführt. [Abschnitt 12.4.1](#1241-dokumente-manipulieren) enthält Einzelheiten zu diesen Operationen.

### 12.3.2. Abfragen parsen

PostgreSQL stellt die Funktionen `to_tsquery`, `plainto_tsquery`, `phraseto_tsquery` und `websearch_to_tsquery` bereit, um eine Abfrage in den Datentyp `tsquery` umzuwandeln. `to_tsquery` bietet Zugriff auf mehr Funktionen als `plainto_tsquery` oder `phraseto_tsquery`, ist aber weniger nachsichtig mit seiner Eingabe. `websearch_to_tsquery` ist eine vereinfachte Version von `to_tsquery` mit einer alternativen Syntax, ähnlich der Syntax von Websuchmaschinen.

```text
to_tsquery([ config regconfig, ] querytext text) returns tsquery
```

`to_tsquery` erzeugt aus `querytext` einen `tsquery`-Wert. `querytext` muss aus einzelnen Tokens bestehen, die durch die `tsquery`-Operatoren `&` (`AND`), `|` (`OR`), `!` (`NOT`) und `<->` (`FOLLOWED BY`) getrennt sind und optional mit Klammern gruppiert werden. Anders gesagt: Die Eingabe für `to_tsquery` muss bereits den allgemeinen Regeln für `tsquery`-Eingaben folgen, wie in [Abschnitt 8.11.2](08_Datentypen.md#8112-tsquery) beschrieben. Der Unterschied besteht darin, dass einfache `tsquery`-Eingabe die Tokens unverändert übernimmt, während `to_tsquery` jedes Token mit der angegebenen oder voreingestellten Konfiguration zu einem Lexem normalisiert und Tokens verwirft, die nach dieser Konfiguration Stoppwörter sind. Zum Beispiel:

```sql
SELECT to_tsquery('english', 'The & Fat & Rats');
```

```text
  to_tsquery
---------------
 'fat' & 'rat'
```

Wie bei einfacher `tsquery`-Eingabe können einem Lexem Gewichte angehängt werden, um die Übereinstimmung auf `tsvector`-Lexeme mit diesen Gewichten zu beschränken. Zum Beispiel:

```sql
SELECT to_tsquery('english', 'Fat | Rats:AB');
```

```text
    to_tsquery
------------------
 'fat' | 'rat':AB
```

Außerdem kann `*` an ein Lexem angehängt werden, um Präfixsuche anzugeben:

```sql
SELECT to_tsquery('supern:*A & star:A*B');
```

```text
        to_tsquery
--------------------------
 'supern':*A & 'star':*AB
```

Ein solches Lexem passt auf jedes Wort in einem `tsvector`, das mit der angegebenen Zeichenkette beginnt.

`to_tsquery` kann auch einfach gequotete Phrasen akzeptieren. Das ist vor allem nützlich, wenn die Konfiguration ein Thesaurus-Wörterbuch enthält, das auf solche Phrasen reagieren kann. Im folgenden Beispiel enthält ein Thesaurus die Regel `supernovae stars : sn`:

```sql
SELECT to_tsquery('''supernovae stars'' & !crab');
```

```text
  to_tsquery
---------------
 'sn' & !'crab'
```

Ohne Anführungszeichen erzeugt `to_tsquery` einen Syntaxfehler für Tokens, die nicht durch einen `AND`-, `OR`- oder `FOLLOWED BY`-Operator getrennt sind.

```text
plainto_tsquery([ config regconfig, ] querytext text) returns tsquery
```

`plainto_tsquery` wandelt den unformatierten Text `querytext` in einen `tsquery`-Wert um. Der Text wird ähnlich wie bei `to_tsvector` geparst und normalisiert; anschließend wird zwischen den übrig gebliebenen Wörtern der `tsquery`-Operator `&` (`AND`) eingefügt.

Beispiel:

```sql
SELECT plainto_tsquery('english', 'The Fat Rats');
```

```text
 plainto_tsquery
-----------------
 'fat' & 'rat'
```

Beachten Sie, dass `plainto_tsquery` keine `tsquery`-Operatoren, Gewichtungslabels oder Präfix-Match-Labels in seiner Eingabe erkennt:

```sql
SELECT plainto_tsquery('english', 'The Fat & Rats:C');
```

```text
   plainto_tsquery
---------------------
 'fat' & 'rat' & 'c'
```

Hier wurde die gesamte Interpunktion der Eingabe verworfen.

```text
phraseto_tsquery([ config regconfig, ] querytext text) returns tsquery
```

`phraseto_tsquery` verhält sich ähnlich wie `plainto_tsquery`, fügt aber zwischen überlebenden Wörtern den Operator `<->` (`FOLLOWED BY`) statt des Operators `&` (`AND`) ein. Außerdem werden Stoppwörter nicht einfach verworfen, sondern durch das Einfügen von `<N>`-Operatoren berücksichtigt, statt nur `<->`-Operatoren zu setzen. Diese Funktion ist nützlich, wenn nach exakten Lexemfolgen gesucht wird, da die `FOLLOWED BY`-Operatoren nicht nur das Vorhandensein aller Lexeme, sondern auch ihre Reihenfolge prüfen.

Beispiel:

```sql
SELECT phraseto_tsquery('english', 'The Fat Rats');
```

```text
 phraseto_tsquery
------------------
 'fat' <-> 'rat'
```

Wie `plainto_tsquery` erkennt auch `phraseto_tsquery` keine `tsquery`-Operatoren, Gewichtungslabels oder Präfix-Match-Labels in seiner Eingabe:

```sql
SELECT phraseto_tsquery('english', 'The Fat & Rats:C');
```

```text
      phraseto_tsquery
-----------------------------
 'fat' <-> 'rat' <-> 'c'
```

```text
websearch_to_tsquery([ config regconfig, ] querytext text) returns tsquery
```

`websearch_to_tsquery` erzeugt aus `querytext` mit einer alternativen Syntax einen `tsquery`-Wert, in der einfacher unformatierter Text eine gültige Abfrage ist. Anders als `plainto_tsquery` und `phraseto_tsquery` erkennt diese Funktion auch bestimmte Operatoren. Außerdem löst sie niemals Syntaxfehler aus, was die Verwendung roher Benutzereingaben für die Suche erlaubt. Die folgende Syntax wird unterstützt:

- Nicht gequoteter Text: Text außerhalb von Anführungszeichen wird in Begriffe umgewandelt, die durch `&`-Operatoren getrennt sind, als wäre er mit `plainto_tsquery` verarbeitet worden.
- `"quoted text"`: Text in Anführungszeichen wird in Begriffe umgewandelt, die durch `<->`-Operatoren getrennt sind, als wäre er mit `phraseto_tsquery` verarbeitet worden.
- `OR`: Das Wort „or“ wird in den Operator `|` umgewandelt.
- `-`: Ein Bindestrich wird in den Operator `!` umgewandelt.

Andere Interpunktion wird ignoriert. Wie `plainto_tsquery` und `phraseto_tsquery` erkennt `websearch_to_tsquery` daher keine `tsquery`-Operatoren, Gewichtungslabels oder Präfix-Match-Labels in seiner Eingabe.

Beispiele:

```sql
SELECT websearch_to_tsquery('english', 'The fat rats');
```

```text
 websearch_to_tsquery
----------------------
 'fat' & 'rat'
(1 row)
```

```sql
SELECT websearch_to_tsquery('english', '"supernovae stars" -crab');
```

```text
        websearch_to_tsquery
----------------------------------
 'supernova' <-> 'star' & !'crab'
(1 row)
```

```sql
SELECT websearch_to_tsquery('english', '"sad cat" or "fat rat"');
```

```text
       websearch_to_tsquery
-----------------------------------
 'sad' <-> 'cat' | 'fat' <-> 'rat'
(1 row)
```

```sql
SELECT websearch_to_tsquery('english', 'signal -"segmentation fault"');
```

```text
             websearch_to_tsquery
---------------------------------------
 'signal' & !( 'segment' <-> 'fault' )
(1 row)
```

```sql
SELECT websearch_to_tsquery('english', '""" )( dummy \\ query <->');
```

```text
 websearch_to_tsquery
----------------------
 'dummi' & 'queri'
(1 row)
```

### 12.3.3. Suchergebnisse bewerten

Ranking versucht zu messen, wie relevant Dokumente für eine bestimmte Abfrage sind, damit bei vielen Treffern die relevantesten zuerst angezeigt werden können. PostgreSQL stellt zwei vordefinierte Rankingfunktionen bereit, die lexikalische, räumliche und strukturelle Informationen berücksichtigen. Sie betrachten also, wie häufig Abfragebegriffe im Dokument vorkommen, wie nahe diese Begriffe beieinanderstehen und wie wichtig der Teil des Dokuments ist, in dem sie vorkommen. Der Begriff Relevanz ist jedoch unscharf und sehr anwendungsspezifisch. Unterschiedliche Anwendungen können zusätzliche Informationen für das Ranking benötigen, etwa den Änderungszeitpunkt eines Dokuments. Die eingebauten Rankingfunktionen sind nur Beispiele. Sie können eigene Rankingfunktionen schreiben und/oder ihre Ergebnisse mit weiteren Faktoren kombinieren, um sie an Ihre Anforderungen anzupassen.

Die beiden derzeit verfügbaren Rankingfunktionen sind:

```text
ts_rank([ weights float4[], ] vector tsvector, query tsquery [, normalization integer ]) returns float4
```

Bewertet Vektoren anhand der Häufigkeit ihrer passenden Lexeme.

```text
ts_rank_cd([ weights float4[], ] vector tsvector, query tsquery [, normalization integer ]) returns float4
```

Diese Funktion berechnet das Cover-Density-Ranking für den gegebenen Dokumentvektor und die Abfrage, wie in Clarke, Cormack und Tudhope, „Relevance Ranking for One to Three Term Queries“, in „Information Processing and Management“, 1999, beschrieben. Cover Density ähnelt dem Ranking von `ts_rank`, berücksichtigt aber zusätzlich die Nähe passender Lexeme zueinander.

Diese Funktion benötigt Positionsinformationen der Lexeme für ihre Berechnung. Daher ignoriert sie alle „stripped“ Lexeme im `tsvector`. Wenn die Eingabe keine nicht gestripteten Lexeme enthält, ist das Ergebnis null. Weitere Informationen zur Funktion `strip` und zu Positionsinformationen in `tsvector`-Werten finden Sie in [Abschnitt 12.4.1](#1241-dokumente-manipulieren).

Bei beiden Funktionen bietet das optionale Argument `weights` die Möglichkeit, Wortvorkommen je nach ihrem Label stärker oder schwächer zu gewichten. Die Gewichtungsarrays geben in dieser Reihenfolge an, wie stark jede Wortkategorie gewichtet wird:

```text
{D-weight, C-weight, B-weight, A-weight}
```

Wenn keine Gewichte angegeben werden, gelten diese Standardwerte:

```text
{0.1, 0.2, 0.4, 1.0}
```

Typischerweise werden Gewichte verwendet, um Wörter aus besonderen Bereichen des Dokuments, etwa Titel oder einleitender Zusammenfassung, zu markieren, sodass sie wichtiger oder weniger wichtig behandelt werden können als Wörter im eigentlichen Dokumenttext.

Da ein längeres Dokument mit höherer Wahrscheinlichkeit einen Abfragebegriff enthält, ist es sinnvoll, die Dokumentgröße zu berücksichtigen. Ein Dokument mit hundert Wörtern und fünf Vorkommen eines Suchwortes ist wahrscheinlich relevanter als ein Dokument mit tausend Wörtern und ebenfalls fünf Vorkommen. Beide Rankingfunktionen akzeptieren eine ganzzahlige Normalisierungsoption, die festlegt, ob und wie die Länge eines Dokuments seinen Rang beeinflussen soll. Die Ganzzahloption steuert mehrere Verhaltensweisen und ist daher eine Bitmaske: Mit `|` können Sie ein oder mehrere Verhaltensweisen angeben, zum Beispiel `2|4`.

- `0` (Standard) ignoriert die Dokumentlänge.
- `1` teilt den Rang durch `1 +` den Logarithmus der Dokumentlänge.
- `2` teilt den Rang durch die Dokumentlänge.
- `4` teilt den Rang durch die mittlere harmonische Entfernung zwischen Extents; dies ist nur von `ts_rank_cd` implementiert.
- `8` teilt den Rang durch die Anzahl eindeutiger Wörter im Dokument.
- `16` teilt den Rang durch `1 +` den Logarithmus der Anzahl eindeutiger Wörter im Dokument.
- `32` teilt den Rang durch sich selbst plus 1.

Wenn mehr als ein Flag-Bit angegeben ist, werden die Transformationen in der oben aufgeführten Reihenfolge angewendet.

Wichtig ist, dass die Rankingfunktionen keine globalen Informationen verwenden. Daher ist es nicht möglich, eine faire Normalisierung auf 1 Prozent oder 100 Prozent zu erzeugen, wie sie manchmal gewünscht wird. Die Normalisierungsoption `32` (`rank/(rank+1)`) kann verwendet werden, um alle Ränge in den Bereich von null bis eins zu skalieren; das ist jedoch nur eine kosmetische Änderung und beeinflusst die Sortierung der Suchergebnisse nicht.

Das folgende Beispiel wählt nur die zehn am höchsten bewerteten Treffer aus:

```sql
SELECT title, ts_rank_cd(textsearch, query) AS rank
FROM apod, to_tsquery('neutrino|(dark & matter)') query
WHERE query @@ textsearch
ORDER BY rank DESC
LIMIT 10;
```

```text
                      title                    |    rank
-----------------------------------------------+----------
 Neutrinos in the Sun                          |       3.1
 The Sudbury Neutrino Detector                 |       2.4
 A MACHO View of Galactic Dark Matter          | 2.01317
 Hot Gas and Dark Matter                       | 1.91171
 The Virgo Cluster: Hot Plasma and Dark Matter | 1.90953
 Rafting for Solar Neutrinos                   |       1.9
 NGC 4650A: Strange Galaxy and Dark Matter     | 1.85774
 Hot Gas and Dark Matter                       |    1.6123
 Ice Fishing for Cosmic Neutrinos              |       1.6
 Weak Lensing Distorts the Universe            | 0.818218
```

Dies ist dasselbe Beispiel mit normalisiertem Ranking:

```sql
SELECT title, ts_rank_cd(textsearch, query, 32 /* rank/(rank+1) */ ) AS rank
FROM apod, to_tsquery('neutrino|(dark & matter)') query
WHERE query @@ textsearch
ORDER BY rank DESC
LIMIT 10;
```

```text
                      title                    |        rank
-----------------------------------------------+-------------------
 Neutrinos in the Sun                          | 0.756097569485493
 The Sudbury Neutrino Detector                 | 0.705882361190954
 A MACHO View of Galactic Dark Matter          | 0.668123210574724
 Hot Gas and Dark Matter                       | 0.65655958650282
 The Virgo Cluster: Hot Plasma and Dark Matter | 0.656301290640973
 Rafting for Solar Neutrinos                   | 0.655172410958162
 NGC 4650A: Strange Galaxy and Dark Matter     | 0.650072921219637
 Hot Gas and Dark Matter                       | 0.617195790024749
 Ice Fishing for Cosmic Neutrinos              | 0.615384618911517
 Weak Lensing Distorts the Universe            | 0.450010798361481
```

Ranking kann teuer sein, weil dazu der `tsvector` jedes passenden Dokuments konsultiert werden muss. Das kann I/O-gebunden und daher langsam sein. Leider lässt sich das kaum vermeiden, da praktische Abfragen oft sehr viele Treffer ergeben.

### 12.3.4. Ergebnisse hervorheben

Für die Präsentation von Suchergebnissen ist es ideal, einen Teil jedes Dokuments zu zeigen und sichtbar zu machen, wie er zur Abfrage passt. Suchmaschinen zeigen normalerweise Dokumentfragmente mit markierten Suchbegriffen. PostgreSQL stellt die Funktion `ts_headline` bereit, die diese Funktionalität implementiert.

```text
ts_headline([ config regconfig, ] document text, query tsquery [, options text ]) returns text
```

`ts_headline` akzeptiert ein Dokument zusammen mit einer Abfrage und gibt einen Auszug aus dem Dokument zurück, in dem Begriffe aus der Abfrage hervorgehoben sind. Genauer gesagt verwendet die Funktion die Abfrage, um relevante Textfragmente auszuwählen, und hebt dann alle Wörter hervor, die in der Abfrage vorkommen, selbst wenn die Wortpositionen nicht zu den Einschränkungen der Abfrage passen. Die Konfiguration zum Parsen des Dokuments kann mit `config` angegeben werden; wenn `config` weggelassen wird, verwendet die Funktion die Konfiguration `default_text_search_config`.

Wenn eine Optionszeichenkette angegeben wird, muss sie aus einer kommaseparierten Liste von einem oder mehreren Paaren `option=value` bestehen. Verfügbar sind diese Optionen:

- `MaxWords`, `MinWords` (Ganzzahlen): Diese Zahlen bestimmen die längste und kürzeste auszugebende Headline. Die Standardwerte sind 35 und 15.
- `ShortWord` (Ganzzahl): Wörter dieser Länge oder kürzer werden am Anfang und Ende einer Headline entfernt, es sei denn, es handelt sich um Abfragebegriffe. Der Standardwert drei entfernt häufige englische Artikel.
- `HighlightAll` (Boolean): Wenn `true`, wird das gesamte Dokument als Headline verwendet; die drei vorherigen Parameter werden ignoriert. Standard ist `false`.
- `MaxFragments` (Ganzzahl): Maximale Anzahl anzuzeigender Textfragmente. Der Standardwert null wählt eine nicht fragmentbasierte Methode zur Headline-Erzeugung. Ein Wert größer als null wählt fragmentbasierte Headline-Erzeugung, siehe unten.
- `StartSel`, `StopSel` (Zeichenketten): Die Zeichenketten, mit denen im Dokument vorkommende Abfragewörter begrenzt werden, um sie von anderen ausgegebenen Wörtern zu unterscheiden. Die Standardwerte sind `<b>` und `</b>`, was für HTML-Ausgabe geeignet sein kann; beachten Sie jedoch die Warnung unten.
- `FragmentDelimiter` (Zeichenkette): Wenn mehr als ein Fragment angezeigt wird, werden die Fragmente durch diese Zeichenkette getrennt. Standard ist ` ... `.

> Warnung: Cross-Site-Scripting-Sicherheit (XSS)
>
> Die Ausgabe von `ts_headline` ist nicht garantiert sicher für die direkte Einbindung in Webseiten. Wenn `HighlightAll` `false` ist, also standardmäßig, werden einige einfache XML-Tags aus dem Dokument entfernt; es ist jedoch nicht garantiert, dass dadurch sämtliches HTML-Markup entfernt wird. Das bietet daher keinen wirksamen Schutz gegen Angriffe wie Cross-Site Scripting (XSS), wenn mit nicht vertrauenswürdiger Eingabe gearbeitet wird. Zum Schutz vor solchen Angriffen sollte sämtliches HTML-Markup aus dem Eingabedokument entfernt oder ein HTML-Sanitizer auf die Ausgabe angewendet werden.

Diese Optionsnamen werden unabhängig von Groß-/Kleinschreibung erkannt. Zeichenkettenwerte müssen in doppelte Anführungszeichen gesetzt werden, wenn sie Leerzeichen oder Kommata enthalten.

Bei nicht fragmentbasierter Headline-Erzeugung lokalisiert `ts_headline` Treffer für die gegebene Abfrage und wählt einen einzelnen Treffer zur Anzeige aus; bevorzugt werden Treffer, die innerhalb der zulässigen Headline-Länge mehr Abfragewörter enthalten. Bei fragmentbasierter Headline-Erzeugung lokalisiert `ts_headline` die Abfragetreffer und teilt jeden Treffer in „Fragmente“ mit höchstens `MaxWords` Wörtern auf. Bevorzugt werden Fragmente mit mehr Abfragewörtern; wenn möglich, werden Fragmente „gedehnt“, um umgebende Wörter einzuschließen. Der fragmentbasierte Modus ist daher nützlicher, wenn sich die Abfragetreffer über große Teile des Dokuments erstrecken oder wenn mehrere Treffer angezeigt werden sollen. In beiden Modi wird, falls keine Abfragetreffer identifiziert werden können, ein einzelnes Fragment aus den ersten `MinWords` Wörtern des Dokuments angezeigt.

Zum Beispiel:

```sql
SELECT ts_headline('english',
  'The most common type of search
is to find all documents containing given query terms
and return them in order of their similarity to the
query.',
  to_tsquery('english', 'query & similarity'));
```

```text
                        ts_headline
------------------------------------------------------------
 containing given <b>query</b> terms                       +
 and return them in order of their <b>similarity</b> to the+
 <b>query</b>.
```

```sql
SELECT ts_headline('english',
  'Search terms may occur
many times in a document,
requiring ranking of the search matches to decide which
occurrences to display in the result.',
  to_tsquery('english', 'search & term'),
  'MaxFragments=10, MaxWords=7, MinWords=3, StartSel=<<, StopSel=>>');
```

```text
                        ts_headline
------------------------------------------------------------
 <<Search>> <<terms>> may occur                             +
 many times ... ranking of the <<search>> matches to decide
```

`ts_headline` verwendet das ursprüngliche Dokument, nicht eine `tsvector`-Zusammenfassung. Deshalb kann die Funktion langsam sein und sollte mit Bedacht eingesetzt werden.

## 12.4. Weitere Funktionen

Dieser Abschnitt beschreibt zusätzliche Funktionen und Operatoren, die im Zusammenhang mit Textsuche nützlich sind.

### 12.4.1. Dokumente manipulieren

[Abschnitt 12.3.1](#1231-dokumente-parsen) zeigte, wie rohe Textdokumente in `tsvector`-Werte umgewandelt werden können. PostgreSQL stellt außerdem Funktionen und Operatoren bereit, mit denen Dokumente manipuliert werden können, die bereits in `tsvector`-Form vorliegen.

```text
tsvector || tsvector
```

Der Verkettungsoperator für `tsvector` gibt einen Vektor zurück, der die Lexeme und Positionsinformationen der beiden als Argumente übergebenen Vektoren kombiniert. Positionen und Gewichtungslabels bleiben während der Verkettung erhalten. Positionen im rechten Vektor werden um die größte im linken Vektor erwähnte Position verschoben, sodass das Ergebnis nahezu dem Ergebnis entspricht, das `to_tsvector` auf der Verkettung der beiden ursprünglichen Dokumentzeichenketten liefern würde. Die Entsprechung ist nicht exakt, weil Stoppwörter, die am Ende des linken Arguments entfernt wurden, das Ergebnis nicht beeinflussen; bei textueller Verkettung hätten sie dagegen die Positionen der Lexeme im rechten Argument beeinflusst.

Ein Vorteil der Verkettung in Vektorform, statt Text vor der Anwendung von `to_tsvector` zu verketten, besteht darin, dass verschiedene Konfigurationen zum Parsen unterschiedlicher Dokumentabschnitte verwendet werden können. Da die Funktion `setweight` außerdem alle Lexeme eines gegebenen Vektors auf dieselbe Weise markiert, muss der Text zuerst geparst und mit `setweight` versehen werden, bevor verkettet wird, wenn unterschiedliche Dokumentteile mit unterschiedlichen Gewichten markiert werden sollen.

```text
setweight(vector tsvector, weight "char") returns tsvector
```

`setweight` gibt eine Kopie des Eingabevektors zurück, in der jede Position mit dem angegebenen Gewicht markiert wurde, also `A`, `B`, `C` oder `D`. `D` ist die Voreinstellung für neue Vektoren und wird daher in der Ausgabe nicht angezeigt. Diese Labels bleiben erhalten, wenn Vektoren verkettet werden, sodass Wörter aus verschiedenen Teilen eines Dokuments von Rankingfunktionen unterschiedlich gewichtet werden können.

Beachten Sie, dass Gewichtungslabels auf Positionen angewendet werden, nicht auf Lexeme. Wenn der Eingabevektor von Positionen befreit wurde, bewirkt `setweight` nichts.

```text
length(vector tsvector) returns integer
```

Gibt die Anzahl der im Vektor gespeicherten Lexeme zurück.

```text
strip(vector tsvector) returns tsvector
```

Gibt einen Vektor zurück, der dieselben Lexeme wie der gegebene Vektor enthält, aber keine Positions- oder Gewichtungsinformationen. Das Ergebnis ist normalerweise viel kleiner als ein nicht gestrippter Vektor, aber auch weniger nützlich. Relevanzranking funktioniert mit gestrippierten Vektoren nicht so gut wie mit ungestrippten. Außerdem passt der `tsquery`-Operator `<->` (`FOLLOWED BY`) niemals auf gestrippte Eingabe, weil er den Abstand zwischen Lexemvorkommen nicht bestimmen kann.

Eine vollständige Liste der `tsvector`-bezogenen Funktionen steht in Tabelle 9.43.

### 12.4.2. Abfragen manipulieren

[Abschnitt 12.3.2](#1232-abfragen-parsen) zeigte, wie rohe Textabfragen in `tsquery`-Werte umgewandelt werden können. PostgreSQL stellt außerdem Funktionen und Operatoren bereit, mit denen Abfragen manipuliert werden können, die bereits in `tsquery`-Form vorliegen.

```text
tsquery && tsquery
```

Gibt die `AND`-Kombination der beiden gegebenen Abfragen zurück.

```text
tsquery || tsquery
```

Gibt die `OR`-Kombination der beiden gegebenen Abfragen zurück.

```text
!! tsquery
```

Gibt die Negation (`NOT`) der gegebenen Abfrage zurück.

```text
tsquery <-> tsquery
```

Gibt eine Abfrage zurück, die nach einem Treffer für die erste gegebene Abfrage sucht, dem unmittelbar ein Treffer für die zweite gegebene Abfrage folgt. Dafür wird der `tsquery`-Operator `<->` (`FOLLOWED BY`) verwendet. Zum Beispiel:

```sql
SELECT to_tsquery('fat') <-> to_tsquery('cat | rat');
```

```text
          ?column?
----------------------------
 'fat' <-> ( 'cat' | 'rat' )
```

```text
tsquery_phrase(query1 tsquery, query2 tsquery [, distance integer ]) returns tsquery
```

Gibt eine Abfrage zurück, die nach einem Treffer für die erste gegebene Abfrage sucht, gefolgt von einem Treffer für die zweite gegebene Abfrage in einem Abstand von genau `distance` Lexemen. Dafür wird der `tsquery`-Operator `<N>` verwendet. Zum Beispiel:

```sql
SELECT tsquery_phrase(to_tsquery('fat'), to_tsquery('cat'), 10);
```

```text
  tsquery_phrase
------------------
 'fat' <10> 'cat'
```

```text
numnode(query tsquery) returns integer
```

Gibt die Anzahl der Knoten, also Lexeme plus Operatoren, in einem `tsquery` zurück. Diese Funktion ist nützlich, um festzustellen, ob die Abfrage sinnvoll ist, also mehr als 0 zurückgibt, oder nur Stoppwörter enthält, also 0 zurückgibt. Beispiele:

```sql
SELECT numnode(plainto_tsquery('the any'));
```

```text
NOTICE: query contains only stopword(s) or doesn't contain lexeme(s), ignored
 numnode
---------
       0
```

```sql
SELECT numnode('foo & bar'::tsquery);
```

```text
 numnode
---------
       3
```

```text
querytree(query tsquery) returns text
```

Gibt den Teil eines `tsquery` zurück, der für die Suche in einem Index verwendet werden kann. Diese Funktion ist nützlich, um nicht indizierbare Abfragen zu erkennen, zum Beispiel solche, die nur Stoppwörter oder nur negierte Begriffe enthalten. Zum Beispiel:

```sql
SELECT querytree(to_tsquery('defined'));
```

```text
 querytree
-----------
 'defin'
```

```sql
SELECT querytree(to_tsquery('!defined'));
```

```text
 querytree
-----------
 T
```

#### 12.4.2.1. Query-Rewriting

Die Funktionsfamilie `ts_rewrite` durchsucht einen gegebenen `tsquery` nach Vorkommen einer Zielunterabfrage und ersetzt jedes Vorkommen durch eine Ersatzunterabfrage. Im Kern ist diese Operation eine `tsquery`-spezifische Version der Teilzeichenkettenersetzung. Eine Kombination aus Ziel und Ersatz kann als Query-Rewrite-Regel betrachtet werden. Eine Sammlung solcher Rewrite-Regeln kann eine leistungsfähige Suchhilfe sein. Zum Beispiel kann eine Suche mithilfe von Synonymen erweitert werden, etwa `new york`, `big apple`, `nyc`, `gotham`, oder eingeschränkt werden, um Benutzer zu einem besonders relevanten Thema zu führen. Es gibt funktionale Überschneidungen zwischen dieser Funktion und Thesaurus-Wörterbüchern ([Abschnitt 12.6.4](#1264-thesauruswörterbuch)). Eine Menge von Rewrite-Regeln kann jedoch im laufenden Betrieb geändert werden, ohne neu zu indizieren, während eine Änderung am Thesaurus erst nach Reindizierung wirksam wird.

```text
ts_rewrite(query tsquery, target tsquery, substitute tsquery) returns tsquery
```

Diese Form von `ts_rewrite` wendet einfach eine einzelne Rewrite-Regel an: `target` wird überall dort durch `substitute` ersetzt, wo es in `query` vorkommt. Zum Beispiel:

```sql
SELECT ts_rewrite('a & b'::tsquery, 'a'::tsquery, 'c'::tsquery);
```

```text
 ts_rewrite
------------
 'b' & 'c'
```

```text
ts_rewrite(query tsquery, select text) returns tsquery
```

Diese Form von `ts_rewrite` akzeptiert eine Startabfrage und einen SQL-`SELECT`-Befehl, der als Textzeichenkette übergeben wird. Das `SELECT` muss zwei Spalten vom Typ `tsquery` liefern. Für jede Zeile des `SELECT`-Ergebnisses werden Vorkommen des Werts der ersten Spalte, also des Ziels, im aktuellen Abfragewert durch den Wert der zweiten Spalte, also den Ersatz, ersetzt. Zum Beispiel:

```sql
CREATE TABLE aliases (t tsquery PRIMARY KEY, s tsquery);
INSERT INTO aliases VALUES('a', 'c');

SELECT ts_rewrite('a & b'::tsquery, 'SELECT t,s FROM aliases');
```

```text
 ts_rewrite
------------
 'b' & 'c'
```

Beachten Sie, dass die Anwendungsreihenfolge wichtig sein kann, wenn mehrere Rewrite-Regeln auf diese Weise angewendet werden. In der Praxis sollte die Quellabfrage daher nach einem Ordnungsschlüssel sortiert werden.

Betrachten wir ein reales astronomisches Beispiel. Wir erweitern die Abfrage `supernovae` mithilfe tabellengesteuerter Rewrite-Regeln:

```sql
CREATE TABLE aliases (t tsquery primary key, s tsquery);
INSERT INTO aliases VALUES(to_tsquery('supernovae'),
                           to_tsquery('supernovae|sn'));

SELECT ts_rewrite(to_tsquery('supernovae & crab'), 'SELECT * FROM aliases');
```

```text
            ts_rewrite
---------------------------------
 'crab' & ( 'supernova' | 'sn' )
```

Wir können die Rewrite-Regeln einfach ändern, indem wir die Tabelle aktualisieren:

```sql
UPDATE aliases
SET s = to_tsquery('supernovae|sn & !nebulae')
WHERE t = to_tsquery('supernovae');

SELECT ts_rewrite(to_tsquery('supernovae & crab'), 'SELECT * FROM aliases');
```

```text
                 ts_rewrite
---------------------------------------------
 'crab' & ( 'supernova' | 'sn' & !'nebula' )
```

Rewriting kann langsam sein, wenn es viele Rewrite-Regeln gibt, weil jede Regel auf mögliche Übereinstimmungen geprüft wird. Um offensichtliche Nichtkandidaten auszufiltern, können die Enthaltenseinsoperatoren für den Typ `tsquery` verwendet werden. Im folgenden Beispiel wählen wir nur die Regeln aus, die zur ursprünglichen Abfrage passen könnten:

```sql
SELECT ts_rewrite('a & b'::tsquery,
                  'SELECT t,s FROM aliases WHERE ''a & b''::tsquery @> t');
```

```text
 ts_rewrite
------------
 'b' & 'c'
```

### 12.4.3. Trigger für automatische Aktualisierungen

> Hinweis: Die in diesem Abschnitt beschriebene Methode wurde durch die Verwendung gespeicherter generierter Spalten überholt, wie in [Abschnitt 12.2.2](#1222-indizes-erstellen) beschrieben.

Wenn eine separate Spalte verwendet wird, um die `tsvector`-Darstellung Ihrer Dokumente zu speichern, muss ein Trigger erstellt werden, der die `tsvector`-Spalte aktualisiert, wenn sich die Inhalts-Spalten des Dokuments ändern. Dafür stehen zwei eingebaute Triggerfunktionen zur Verfügung; alternativ können Sie eine eigene schreiben.

```text
tsvector_update_trigger(tsvector_column_name, config_name, text_column_name [, ... ])
tsvector_update_trigger_column(tsvector_column_name, config_column_name, text_column_name [, ... ])
```

Diese Triggerfunktionen berechnen automatisch eine `tsvector`-Spalte aus einer oder mehreren Textspalten, gesteuert durch Parameter, die im Befehl `CREATE TRIGGER` angegeben werden. Ein Beispiel für ihre Verwendung:

```sql
CREATE TABLE messages (
    title       text,
    body        text,
    tsv         tsvector
);

CREATE TRIGGER tsvectorupdate BEFORE INSERT OR UPDATE
ON messages FOR EACH ROW EXECUTE FUNCTION
tsvector_update_trigger(tsv, 'pg_catalog.english', title, body);

INSERT INTO messages VALUES('title here', 'the body text is here');

SELECT * FROM messages;
```

```text
   title    |         body          |            tsv
------------+-----------------------+----------------------------
 title here | the body text is here | 'bodi':4 'text':5 'titl':1
```

```sql
SELECT title, body FROM messages WHERE tsv @@ to_tsquery('title & body');
```

```text
   title    |         body
------------+-----------------------
 title here | the body text is here
```

Nachdem dieser Trigger erstellt wurde, spiegelt sich jede Änderung an `title` oder `body` automatisch in `tsv` wider, ohne dass die Anwendung sich darum kümmern muss.

Das erste Triggerargument muss der Name der zu aktualisierenden `tsvector`-Spalte sein. Das zweite Argument gibt die Textsuchkonfiguration an, die für die Umwandlung verwendet werden soll. Bei `tsvector_update_trigger` wird der Konfigurationsname einfach als zweites Triggerargument angegeben. Er muss wie oben gezeigt schemaqualifiziert sein, damit sich das Triggerverhalten nicht durch Änderungen an `search_path` ändert. Bei `tsvector_update_trigger_column` ist das zweite Triggerargument der Name einer anderen Tabellenspalte, die vom Typ `regconfig` sein muss. Damit kann die Konfiguration zeilenweise ausgewählt werden. Die übrigen Argumente sind die Namen textueller Spalten vom Typ `text`, `varchar` oder `char`. Diese werden in der angegebenen Reihenfolge in das Dokument aufgenommen. `NULL`-Werte werden übersprungen, die anderen Spalten werden aber weiterhin indiziert.

Eine Einschränkung dieser eingebauten Trigger ist, dass sie alle Eingabespalten gleich behandeln. Um Spalten unterschiedlich zu verarbeiten, zum Beispiel den Titel anders zu gewichten als den Textkörper, muss ein eigener Trigger geschrieben werden. Hier ist ein Beispiel mit PL/pgSQL als Triggersprache:

```sql
CREATE FUNCTION messages_trigger() RETURNS trigger AS $$
begin
  new.tsv :=
      setweight(to_tsvector('pg_catalog.english',
                            coalesce(new.title,'')), 'A') ||
      setweight(to_tsvector('pg_catalog.english',
                            coalesce(new.body,'')), 'D');
  return new;
end
$$ LANGUAGE plpgsql;

CREATE TRIGGER tsvectorupdate BEFORE INSERT OR UPDATE
    ON messages FOR EACH ROW EXECUTE FUNCTION messages_trigger();
```

Denken Sie daran, dass es wichtig ist, den Konfigurationsnamen beim Erzeugen von `tsvector`-Werten in Triggern ausdrücklich anzugeben, damit der Spalteninhalt nicht von Änderungen an `default_text_search_config` beeinflusst wird. Wird dies versäumt, kann das zu Problemen führen, etwa dazu, dass sich Suchergebnisse nach Dump und Restore ändern.

### 12.4.4. Dokumentstatistiken sammeln

Die Funktion `ts_stat` ist nützlich, um Ihre Konfiguration zu prüfen und Kandidaten für Stoppwörter zu finden.

```text
ts_stat(sqlquery text, [ weights text, ]
        OUT word text, OUT ndoc integer,
        OUT nentry integer) returns setof record
```

`sqlquery` ist ein Textwert, der eine SQL-Abfrage enthält, die eine einzelne `tsvector`-Spalte zurückgeben muss. `ts_stat` führt die Abfrage aus und gibt Statistiken über jedes unterschiedliche Lexem, also jedes Wort, zurück, das in den `tsvector`-Daten enthalten ist. Die zurückgegebenen Spalten sind:

- `word text`: der Wert eines Lexems.
- `ndoc integer`: Anzahl der Dokumente, also `tsvector`-Werte, in denen das Wort vorkam.
- `nentry integer`: Gesamtzahl der Vorkommen des Wortes.

Wenn `weights` angegeben ist, werden nur Vorkommen mit einem dieser Gewichte gezählt.

Um zum Beispiel die zehn häufigsten Wörter in einer Dokumentensammlung zu finden:

```sql
SELECT * FROM ts_stat('SELECT vector FROM apod')
ORDER BY nentry DESC, ndoc DESC, word
LIMIT 10;
```

Dasselbe, aber nur mit Wortvorkommen der Gewichte `A` oder `B`:

```sql
SELECT * FROM ts_stat('SELECT vector FROM apod', 'ab')
ORDER BY nentry DESC, ndoc DESC, word
LIMIT 10;
```

## 12.5. Parser

Textsuchparser sind dafür verantwortlich, rohen Dokumenttext in Tokens zu zerlegen und den Typ jedes Tokens zu bestimmen. Die Menge der möglichen Typen wird dabei vom Parser selbst definiert. Beachten Sie, dass ein Parser den Text überhaupt nicht verändert, sondern lediglich plausible Wortgrenzen erkennt. Wegen dieses begrenzten Aufgabenbereichs besteht weniger Bedarf an anwendungsspezifischen eigenen Parsern als an eigenen Wörterbüchern. Derzeit stellt PostgreSQL nur einen eingebauten Parser bereit, der sich für eine breite Palette von Anwendungen als nützlich erwiesen hat.

Der eingebaute Parser heißt `pg_catalog.default`. Er erkennt 23 Token-Typen, die in Tabelle 12.1 gezeigt werden.

**Tabelle 12.1. Token-Typen des Standardparsers**

| Alias | Beschreibung | Beispiel |
| --- | --- | --- |
| `asciiword` | Wort, ausschließlich ASCII-Buchstaben | `elephant` |
| `word` | Wort, ausschließlich Buchstaben | `mañana` |
| `numword` | Wort, Buchstaben und Ziffern | `beta1` |
| `asciihword` | Wort mit Bindestrich, ausschließlich ASCII | `up-to-date` |
| `hword` | Wort mit Bindestrich, ausschließlich Buchstaben | `lógico-matemática` |
| `numhword` | Wort mit Bindestrich, Buchstaben und Ziffern | `postgresql-beta1` |
| `hword_asciipart` | Teil eines Wortes mit Bindestrich, ausschließlich ASCII | `postgresql` im Kontext `postgresql-beta1` |
| `hword_part` | Teil eines Wortes mit Bindestrich, ausschließlich Buchstaben | `lógico` oder `matemática` im Kontext `lógico-matemática` |
| `hword_numpart` | Teil eines Wortes mit Bindestrich, Buchstaben und Ziffern | `beta1` im Kontext `postgresql-beta1` |
| `email` | E-Mail-Adresse | `foo@example.com` |
| `protocol` | Protokollkopf | `http://` |
| `url` | URL | `example.com/stuff/index.html` |
| `host` | Host | `example.com` |
| `url_path` | URL-Pfad | `/stuff/index.html` im Kontext einer URL |
| `file` | Datei- oder Pfadname | `/usr/local/foo.txt`, wenn nicht innerhalb einer URL |
| `sfloat` | Wissenschaftliche Notation | `-1.234e56` |
| `float` | Dezimalschreibweise | `-1.234` |
| `int` | Ganzzahl mit Vorzeichen | `-1234` |
| `uint` | Ganzzahl ohne Vorzeichen | `1234` |
| `version` | Versionsnummer | `8.3.0` |
| `tag` | XML-Tag | `<a href="dictionaries.html">` |
| `entity` | XML-Entity | `&amp;` |
| `blank` | Leerraumsymbole | beliebiger Leerraum oder sonst nicht erkannte Interpunktion |

> Hinweis: Die Vorstellung des Parsers davon, was ein „Buchstabe“ ist, wird durch die Locale-Einstellung der Datenbank bestimmt, genauer durch `lc_ctype`. Wörter, die nur die einfachen ASCII-Buchstaben enthalten, werden als eigener Token-Typ gemeldet, weil es manchmal nützlich ist, sie zu unterscheiden. In den meisten europäischen Sprachen sollten die Token-Typen `word` und `asciiword` gleich behandelt werden.
>
> `email` unterstützt nicht alle gültigen E-Mail-Zeichen gemäß [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322). Insbesondere werden als nicht alphanumerische Zeichen in E-Mail-Benutzernamen nur Punkt, Bindestrich und Unterstrich unterstützt.
>
> `tag` unterstützt nicht alle gültigen Tag-Namen gemäß der [W3C-Empfehlung XML](https://www.w3.org/TR/xml/). Insbesondere werden nur Tag-Namen unterstützt, die mit einem ASCII-Buchstaben, Unterstrich oder Doppelpunkt beginnen und nur Buchstaben, Ziffern, Bindestriche, Unterstriche, Punkte und Doppelpunkte enthalten. `tag` umfasst außerdem XML-Kommentare, die mit `<!--` beginnen und mit `-->` enden, sowie XML-Deklarationen. Beachten Sie aber, dass dies alles einschließt, was mit `<?x` beginnt und mit `>` endet.

Es ist möglich, dass der Parser aus demselben Textstück überlappende Tokens erzeugt. Ein Wort mit Bindestrich wird zum Beispiel sowohl als ganzes Wort als auch in seinen einzelnen Bestandteilen gemeldet:

```sql
SELECT alias, description, token FROM ts_debug('foo-bar-beta1');
```

```text
     alias       |              description               |     token
-----------------+----------------------------------------+---------------
 numhword        | Hyphenated word, letters and digits    | foo-bar-beta1
 hword_asciipart | Hyphenated word part, all ASCII        | foo
 blank           | Space symbols                          | -
 hword_asciipart | Hyphenated word part, all ASCII        | bar
 blank           | Space symbols                          | -
 hword_numpart   | Hyphenated word part, letters and digits | beta1
```

Dieses Verhalten ist erwünscht, weil Suchen dadurch sowohl für das gesamte zusammengesetzte Wort als auch für seine Bestandteile funktionieren. Hier ist ein weiteres aufschlussreiches Beispiel:

```sql
SELECT alias, description, token FROM ts_debug('http://example.com/stuff/index.html');
```

```text
  alias   | description   |              token
----------+---------------+------------------------------
 protocol | Protocol head | http://
 url      | URL           | example.com/stuff/index.html
 host     | Host          | example.com
 url_path | URL path      | /stuff/index.html
```

## 12.6. Wörterbücher

Wörterbücher werden verwendet, um Wörter zu entfernen, die bei einer Suche nicht berücksichtigt werden sollen, also Stoppwörter, und um Wörter so zu normalisieren, dass verschiedene abgeleitete Formen desselben Wortes übereinstimmen. Ein erfolgreich normalisiertes Wort heißt Lexem. Neben der Verbesserung der Suchqualität verringern Normalisierung und Entfernen von Stoppwörtern die Größe der `tsvector`-Darstellung eines Dokuments und verbessern dadurch die Performance. Normalisierung hat nicht immer eine linguistische Bedeutung und hängt normalerweise von der Semantik der Anwendung ab.

Einige Beispiele für Normalisierung:

- Linguistisch: Ispell-Wörterbücher versuchen, Eingabewörter auf eine normalisierte Form zurückzuführen; Stemmer-Wörterbücher entfernen Wortendungen.
- URL-Orte können kanonisiert werden, damit äquivalente URLs übereinstimmen:
  - `http://www.pgsql.ru/db/mw/index.html`
  - `http://www.pgsql.ru/db/mw/`
  - `http://www.pgsql.ru/db/../db/mw/index.html`
- Farbnamen können durch ihre Hexadezimalwerte ersetzt werden, zum Beispiel `red`, `green`, `blue`, `magenta` -> `FF0000`, `00FF00`, `0000FF`, `FF00FF`.
- Wenn Zahlen indiziert werden, können einige Nachkommastellen entfernt werden, um den Bereich möglicher Zahlen zu reduzieren. So wären zum Beispiel `3.14159265359`, `3.1415926` und `3.14` nach der Normalisierung gleich, wenn nur zwei Stellen nach dem Dezimalpunkt behalten werden.

Ein Wörterbuch ist ein Programm, das ein Token als Eingabe annimmt und Folgendes zurückgibt:

- ein Array von Lexemen, wenn das Eingabe-Token dem Wörterbuch bekannt ist; beachten Sie, dass ein Token mehr als ein Lexem erzeugen kann
- ein einzelnes Lexem mit gesetztem `TSL_FILTER`-Flag, um das ursprüngliche Token durch ein neues Token zu ersetzen, das an nachfolgende Wörterbücher weitergereicht wird; ein Wörterbuch, das dies tut, heißt filterndes Wörterbuch
- ein leeres Array, wenn das Wörterbuch das Token kennt, es aber ein Stoppwort ist
- `NULL`, wenn das Wörterbuch das Eingabe-Token nicht erkennt

PostgreSQL stellt vordefinierte Wörterbücher für viele Sprachen bereit. Außerdem gibt es mehrere vordefinierte Vorlagen, mit denen neue Wörterbücher mit eigenen Parametern erstellt werden können. Jede vordefinierte Wörterbuchvorlage wird im Folgenden beschrieben. Wenn keine vorhandene Vorlage passt, können auch neue erstellt werden; Beispiele finden Sie im Bereich `contrib/` der PostgreSQL-Distribution.

Eine Textsuchkonfiguration verbindet einen Parser mit einer Menge von Wörterbüchern, um die vom Parser ausgegebenen Tokens zu verarbeiten. Für jeden Token-Typ, den der Parser zurückgeben kann, legt die Konfiguration eine eigene Liste von Wörterbüchern fest. Wenn der Parser ein Token dieses Typs findet, wird jedes Wörterbuch in der Liste der Reihe nach befragt, bis ein Wörterbuch es als bekanntes Wort erkennt. Wird es als Stoppwort erkannt oder erkennt kein Wörterbuch das Token, wird es verworfen und weder indiziert noch gesucht. Normalerweise bestimmt das erste Wörterbuch, das eine von `NULL` verschiedene Ausgabe zurückgibt, das Ergebnis, und die übrigen Wörterbücher werden nicht mehr befragt; ein filterndes Wörterbuch kann das angegebene Wort jedoch durch ein verändertes Wort ersetzen, das anschließend an nachfolgende Wörterbücher weitergereicht wird.

Die allgemeine Regel für die Konfiguration einer Wörterbuchliste lautet: zuerst das engste und spezifischste Wörterbuch, danach allgemeinere Wörterbücher und zuletzt ein sehr allgemeines Wörterbuch wie ein Snowball-Stemmer oder `simple`, das alles erkennt. Für eine astronomiespezifische Suche, etwa eine Konfiguration `astro_en`, könnte man den Token-Typ `asciiword` (ASCII-Wort) zum Beispiel an ein Synonymwörterbuch astronomischer Begriffe, ein allgemeines englisches Wörterbuch und einen englischen Snowball-Stemmer binden:

```sql
ALTER TEXT SEARCH CONFIGURATION astro_en
    ADD MAPPING FOR asciiword WITH astrosyn, english_ispell,
    english_stem;
```

Ein filterndes Wörterbuch kann an jeder Stelle der Liste stehen, außer am Ende, wo es nutzlos wäre. Filternde Wörterbücher sind nützlich, um Wörter teilweise zu normalisieren und damit die Aufgabe späterer Wörterbücher zu vereinfachen. Ein filterndes Wörterbuch könnte zum Beispiel verwendet werden, um Akzente aus akzentuierten Buchstaben zu entfernen, wie es das Modul `unaccent` tut.

### 12.6.1. Stoppwörter

Stoppwörter sind Wörter, die sehr häufig sind, in fast jedem Dokument vorkommen und keinen Unterscheidungswert haben. Daher können sie im Kontext der Volltextsuche ignoriert werden. Zum Beispiel enthält jeder englische Text Wörter wie `a` und `the`, daher ist es nutzlos, sie in einem Index zu speichern. Stoppwörter beeinflussen jedoch die Positionen in einem `tsvector`, was wiederum das Ranking beeinflusst:

```sql
SELECT to_tsvector('english', 'in the list of stop words');
```

```text
        to_tsvector
----------------------------
 'list':3 'stop':5 'word':6
```

Die fehlenden Positionen `1`, `2` und `4` entstehen durch Stoppwörter. Ränge, die für Dokumente mit und ohne Stoppwörter berechnet werden, unterscheiden sich deutlich:

```sql
SELECT ts_rank_cd(to_tsvector('english', 'in the list of stop words'),
                  to_tsquery('list & stop'));
```

```text
 ts_rank_cd
------------
       0.05
```

```sql
SELECT ts_rank_cd(to_tsvector('english', 'list stop words'),
                  to_tsquery('list & stop'));
```

```text
 ts_rank_cd
------------
        0.1
```

Wie Stoppwörter behandelt werden, hängt vom jeweiligen Wörterbuch ab. Ispell-Wörterbücher normalisieren Wörter zum Beispiel zuerst und sehen dann in der Stoppwortliste nach, während Snowball-Stemmer zuerst die Stoppwortliste prüfen. Der Grund für dieses unterschiedliche Verhalten ist der Versuch, Rauschen zu verringern.

### 12.6.2. Einfaches Wörterbuch

Die Vorlage für einfache Wörterbücher arbeitet, indem sie das Eingabe-Token in Kleinbuchstaben umwandelt und gegen eine Stoppwortdatei prüft. Wird es in der Datei gefunden, wird ein leeres Array zurückgegeben, wodurch das Token verworfen wird. Andernfalls wird die kleingeschriebene Form des Wortes als normalisiertes Lexem zurückgegeben. Alternativ kann das Wörterbuch so konfiguriert werden, dass es Nicht-Stoppwörter als nicht erkannt meldet, sodass sie an das nächste Wörterbuch in der Liste weitergegeben werden.

Hier ist ein Beispiel für eine Wörterbuchdefinition mit der Vorlage `simple`:

```sql
CREATE TEXT SEARCH DICTIONARY public.simple_dict (
    TEMPLATE = pg_catalog.simple,
    STOPWORDS = english
);
```

Hier ist `english` der Basisname einer Stoppwortdatei. Der vollständige Dateiname lautet `$SHAREDIR/tsearch_data/english.stop`, wobei `$SHAREDIR` das Shared-Data-Verzeichnis der PostgreSQL-Installation meint, häufig `/usr/local/share/postgresql`; verwenden Sie `pg_config --sharedir`, wenn Sie unsicher sind. Das Dateiformat ist einfach eine Liste von Wörtern, eines pro Zeile. Leere Zeilen und nachgestellte Leerzeichen werden ignoriert, Großbuchstaben werden in Kleinbuchstaben umgewandelt, aber darüber hinaus wird der Dateiinhalt nicht verarbeitet.

Nun können wir unser Wörterbuch testen:

```sql
SELECT ts_lexize('public.simple_dict', 'YeS');
```

```text
 ts_lexize
-----------
 {yes}
```

```sql
SELECT ts_lexize('public.simple_dict', 'The');
```

```text
 ts_lexize
-----------
 {}
```

Wir können auch festlegen, dass `NULL` statt des kleingeschriebenen Wortes zurückgegeben wird, wenn das Wort nicht in der Stoppwortdatei vorkommt. Dieses Verhalten wird ausgewählt, indem der Parameter `Accept` des Wörterbuchs auf `false` gesetzt wird. Das Beispiel geht weiter:

```sql
ALTER TEXT SEARCH DICTIONARY public.simple_dict ( Accept = false );

SELECT ts_lexize('public.simple_dict', 'YeS');
```

```text
 ts_lexize
-----------
```

```sql
SELECT ts_lexize('public.simple_dict', 'The');
```

```text
 ts_lexize
-----------
 {}
```

Mit der Voreinstellung `Accept = true` ist es nur sinnvoll, ein einfaches Wörterbuch ans Ende einer Wörterbuchliste zu stellen, da es nie ein Token an ein folgendes Wörterbuch weiterreicht. Umgekehrt ist `Accept = false` nur sinnvoll, wenn mindestens ein folgendes Wörterbuch vorhanden ist.

> Achtung: Die meisten Wörterbuchtypen hängen von Konfigurationsdateien ab, etwa von Stoppwortdateien. Diese Dateien müssen in UTF-8 kodiert sein. Sie werden beim Einlesen in den Server in die tatsächliche Datenbankkodierung übersetzt, falls diese davon abweicht.

> Achtung: Normalerweise liest eine Datenbanksitzung eine Wörterbuch-Konfigurationsdatei nur einmal, nämlich bei ihrer ersten Verwendung in der Sitzung. Wenn Sie eine Konfigurationsdatei ändern und vorhandene Sitzungen dazu zwingen wollen, den neuen Inhalt zu übernehmen, führen Sie für das Wörterbuch einen `ALTER TEXT SEARCH DICTIONARY`-Befehl aus. Das kann ein „Dummy“-Update sein, das tatsächlich keine Parameterwerte ändert.

### 12.6.3. Synonymwörterbuch

Diese Wörterbuchvorlage wird verwendet, um Wörterbücher zu erstellen, die ein Wort durch ein Synonym ersetzen. Phrasen werden nicht unterstützt; verwenden Sie dafür die Thesaurus-Vorlage ([Abschnitt 12.6.4](#1264-thesauruswörterbuch)). Ein Synonymwörterbuch kann verwendet werden, um linguistische Probleme zu umgehen, zum Beispiel um zu verhindern, dass ein englisches Stemmer-Wörterbuch das Wort `Paris` auf `pari` reduziert. Es reicht, im Synonymwörterbuch eine Zeile `Paris paris` zu haben und dieses Wörterbuch vor das Wörterbuch `english_stem` zu setzen. Zum Beispiel:

```sql
SELECT * FROM ts_debug('english', 'Paris');
```

```text
   alias   |   description   | token |  dictionaries  |  dictionary  | lexemes
-----------+-----------------+-------+----------------+--------------+---------
 asciiword | Word, all ASCII | Paris | {english_stem} | english_stem | {pari}
```

```sql
CREATE TEXT SEARCH DICTIONARY my_synonym (
    TEMPLATE = synonym,
    SYNONYMS = my_synonyms
);

ALTER TEXT SEARCH CONFIGURATION english
    ALTER MAPPING FOR asciiword
    WITH my_synonym, english_stem;

SELECT * FROM ts_debug('english', 'Paris');
```

```text
   alias   |   description   | token |        dictionaries       | dictionary | lexemes
-----------+-----------------+-------+---------------------------+------------+---------
 asciiword | Word, all ASCII | Paris | {my_synonym,english_stem} | my_synonym | {paris}
```

Der einzige erforderliche Parameter der Vorlage `synonym` ist `SYNONYMS`, der Basisname ihrer Konfigurationsdatei, im obigen Beispiel `my_synonyms`. Der vollständige Dateiname lautet `$SHAREDIR/tsearch_data/my_synonyms.syn`, wobei `$SHAREDIR` das Shared-Data-Verzeichnis der PostgreSQL-Installation meint. Das Dateiformat besteht einfach aus einer Zeile pro zu ersetzendem Wort, wobei auf das Wort sein Synonym folgt, getrennt durch Leerraum. Leere Zeilen und nachgestellte Leerzeichen werden ignoriert.

Die Vorlage `synonym` hat außerdem den optionalen Parameter `CaseSensitive`, dessen Voreinstellung `false` ist. Wenn `CaseSensitive` `false` ist, werden Wörter in der Synonymdatei ebenso wie Eingabe-Tokens in Kleinbuchstaben umgewandelt. Wenn der Parameter `true` ist, werden Wörter und Tokens nicht in Kleinbuchstaben umgewandelt, sondern unverändert verglichen.

Am Ende eines Synonyms in der Konfigurationsdatei kann ein Sternchen (`*`) stehen. Es zeigt an, dass das Synonym ein Präfix ist. Das Sternchen wird ignoriert, wenn der Eintrag in `to_tsvector()` verwendet wird; bei Verwendung in `to_tsquery()` wird das Ergebnis jedoch ein Abfrageelement mit Präfix-Match-Markierung (siehe [Abschnitt 12.3.2](#1232-abfragen-parsen)). Angenommen, `$SHAREDIR/tsearch_data/synonym_sample.syn` enthält diese Einträge:

```text
postgres                pgsql
postgresql              pgsql
postgre pgsql
gogle    googl
indices index*
```

Dann erhalten wir diese Ergebnisse:

```text
mydb=# CREATE TEXT SEARCH DICTIONARY syn (template=synonym,
 synonyms='synonym_sample');
mydb=# SELECT ts_lexize('syn', 'indices');
 ts_lexize
-----------
 {index}
(1 row)

mydb=# CREATE TEXT SEARCH CONFIGURATION tst (copy=simple);
mydb=# ALTER TEXT SEARCH CONFIGURATION tst ALTER MAPPING FOR
 asciiword WITH syn;
mydb=# SELECT to_tsvector('tst', 'indices');
 to_tsvector
-------------
 'index':1
(1 row)

mydb=# SELECT to_tsquery('tst', 'indices');
 to_tsquery
------------
 'index':*
(1 row)

mydb=# SELECT 'indexes are very useful'::tsvector;
            tsvector
---------------------------------
 'are' 'indexes' 'useful' 'very'
(1 row)

mydb=# SELECT 'indexes are very useful'::tsvector @@
 to_tsquery('tst', 'indices');
 ?column?
----------
 t
(1 row)
```

### 12.6.4. Thesaurus-Wörterbuch

Ein Thesaurus-Wörterbuch, manchmal als TZ abgekürzt, ist eine Sammlung von Wörtern, die Informationen über Beziehungen zwischen Wörtern und Phrasen enthält, also Oberbegriffe (`BT`), Unterbegriffe (`NT`), bevorzugte Begriffe, nicht bevorzugte Begriffe, verwandte Begriffe und so weiter.

Grundsätzlich ersetzt ein Thesaurus-Wörterbuch alle nicht bevorzugten Begriffe durch einen bevorzugten Begriff und behält optional zusätzlich die ursprünglichen Begriffe für die Indizierung bei. Die aktuelle PostgreSQL-Implementierung des Thesaurus-Wörterbuchs ist eine Erweiterung des Synonymwörterbuchs mit zusätzlicher Phrasenunterstützung. Ein Thesaurus-Wörterbuch benötigt eine Konfigurationsdatei im folgenden Format:

```text
# this is a comment
sample word(s) : indexed word(s)
more sample word(s) : more indexed word(s)
...
```

Dabei dient der Doppelpunkt (`:`) als Trennzeichen zwischen einer Phrase und ihrer Ersetzung.

Ein Thesaurus-Wörterbuch verwendet ein Unterwörterbuch, das in der Wörterbuchkonfiguration angegeben wird, um den Eingabetext zu normalisieren, bevor auf Phrasenübereinstimmungen geprüft wird. Es kann nur ein Unterwörterbuch ausgewählt werden. Wenn das Unterwörterbuch ein Wort nicht erkennt, wird ein Fehler gemeldet. In diesem Fall sollten Sie die Verwendung des Wortes entfernen oder dem Unterwörterbuch das Wort beibringen. Am Anfang eines indizierten Wortes kann ein Sternchen (`*`) stehen, um die Anwendung des Unterwörterbuchs auf dieses Wort zu überspringen; alle Beispielwörter müssen dem Unterwörterbuch jedoch bekannt sein.

Das Thesaurus-Wörterbuch wählt die längste Übereinstimmung, wenn mehrere Phrasen zur Eingabe passen; Gleichstände werden zugunsten der letzten Definition aufgelöst.

Bestimmte Stoppwörter, die vom Unterwörterbuch erkannt werden, können nicht angegeben werden. Verwenden Sie stattdessen `?`, um die Stelle zu markieren, an der ein beliebiges Stoppwort vorkommen darf. Angenommen, `a` und `the` sind gemäß Unterwörterbuch Stoppwörter:

```text
? one ? two : swsw
```

Dies passt auf `a one the two` und `the one a two`; beide würden durch `swsw` ersetzt.

Da ein Thesaurus-Wörterbuch Phrasen erkennen kann, muss es seinen Zustand speichern und mit dem Parser zusammenarbeiten. Ein Thesaurus-Wörterbuch verwendet diese Zuordnungen, um zu prüfen, ob es das nächste Wort behandeln oder die Akkumulation beenden soll. Das Thesaurus-Wörterbuch muss sorgfältig konfiguriert werden. Wenn das Thesaurus-Wörterbuch zum Beispiel nur dem Token-Typ `asciiword` zugeordnet ist, funktioniert eine Thesaurus-Definition wie `one 7` nicht, weil der Token-Typ `uint` dem Thesaurus-Wörterbuch nicht zugeordnet ist.

> Achtung: Thesauri werden während der Indizierung verwendet. Jede Änderung der Parameter eines Thesaurus-Wörterbuchs erfordert daher ein Reindexing. Bei den meisten anderen Wörterbuchtypen erzwingen kleine Änderungen, etwa das Hinzufügen oder Entfernen von Stoppwörtern, kein Reindexing.

#### 12.6.4.1. Thesaurus-Konfiguration

Um ein neues Thesaurus-Wörterbuch zu definieren, verwenden Sie die Vorlage `thesaurus`. Zum Beispiel:

```sql
CREATE TEXT SEARCH DICTIONARY thesaurus_simple (
    TEMPLATE = thesaurus,
    DictFile = mythesaurus,
    Dictionary = pg_catalog.english_stem
);
```

Dabei gilt:

- `thesaurus_simple` ist der Name des neuen Wörterbuchs.
- `mythesaurus` ist der Basisname der Thesaurus-Konfigurationsdatei. Ihr vollständiger Name lautet `$SHAREDIR/tsearch_data/mythesaurus.ths`, wobei `$SHAREDIR` das Shared-Data-Verzeichnis der Installation meint.
- `pg_catalog.english_stem` ist das Unterwörterbuch, hier ein englischer Snowball-Stemmer, das für die Thesaurus-Normalisierung verwendet wird. Beachten Sie, dass das Unterwörterbuch seine eigene Konfiguration besitzt, zum Beispiel Stoppwörter, die hier nicht gezeigt wird.

Nun kann das Thesaurus-Wörterbuch `thesaurus_simple` in einer Konfiguration an die gewünschten Token-Typen gebunden werden, zum Beispiel:

```sql
ALTER TEXT SEARCH CONFIGURATION russian
    ALTER MAPPING FOR asciiword, asciihword, hword_asciipart
    WITH thesaurus_simple;
```

#### 12.6.4.2. Thesaurus-Beispiel

Betrachten Sie einen einfachen astronomischen Thesaurus `thesaurus_astro`, der einige astronomische Wortkombinationen enthält:

```text
supernovae stars : sn
crab nebulae : crab
```

Unten erstellen wir ein Wörterbuch und binden einige Token-Typen an einen astronomischen Thesaurus und einen englischen Stemmer:

```sql
CREATE TEXT SEARCH DICTIONARY thesaurus_astro (
    TEMPLATE = thesaurus,
    DictFile = thesaurus_astro,
    Dictionary = english_stem
);

ALTER TEXT SEARCH CONFIGURATION russian
    ALTER MAPPING FOR asciiword, asciihword, hword_asciipart
    WITH thesaurus_astro, english_stem;
```

Jetzt können wir sehen, wie es funktioniert. `ts_lexize` ist zum Testen eines Thesaurus nicht sehr nützlich, weil es seine Eingabe als einzelnes Token behandelt. Stattdessen können wir `plainto_tsquery` und `to_tsvector` verwenden, die ihre Eingabezeichenketten in mehrere Tokens zerlegen:

```sql
SELECT plainto_tsquery('supernova star');
```

```text
 plainto_tsquery
-----------------
 'sn'
```

```sql
SELECT to_tsvector('supernova star');
```

```text
 to_tsvector
-------------
 'sn':1
```

Im Prinzip kann man `to_tsquery` verwenden, wenn man das Argument quotet:

```sql
SELECT to_tsquery('''supernova star''');
```

```text
 to_tsquery
------------
 'sn'
```

Beachten Sie, dass `supernova star` zu `supernovae stars` in `thesaurus_astro` passt, weil wir in der Thesaurus-Definition den Stemmer `english_stem` angegeben haben. Der Stemmer hat das `e` und das `s` entfernt.

Um sowohl die ursprüngliche Phrase als auch den Ersatz zu indizieren, nehmen Sie sie einfach in den rechten Teil der Definition auf:

```text
supernovae stars : sn supernovae stars
```

```sql
SELECT plainto_tsquery('supernova star');
```

```text
       plainto_tsquery
-----------------------------
 'sn' & 'supernova' & 'star'
```

### 12.6.5. Ispell-Wörterbuch

Die Vorlage für Ispell-Wörterbücher unterstützt morphologische Wörterbücher, die viele unterschiedliche sprachliche Formen eines Wortes auf dasselbe Lexem normalisieren können. Ein englisches Ispell-Wörterbuch kann zum Beispiel alle Deklinationen und Konjugationen des Suchbegriffs `bank` erfassen, etwa `banking`, `banked`, `banks`, `banks'` und `bank's`.

Die PostgreSQL-Standarddistribution enthält keine Ispell-Konfigurationsdateien. Wörterbücher für viele Sprachen sind bei [Ispell](https://www.cs.hmc.edu/~geoff/ispell.html) verfügbar. Außerdem werden einige modernere Wörterbuchdateiformate unterstützt: [MySpell](https://en.wikipedia.org/wiki/MySpell) (OO < 2.0.1) und [Hunspell](https://hunspell.github.io/) (OO >= 2.0.2). Eine große Liste von Wörterbüchern ist im [OpenOffice-Wiki](https://wiki.openoffice.org/wiki/Dictionaries) verfügbar.

Zum Erstellen eines Ispell-Wörterbuchs gehen Sie folgendermaßen vor:

- Laden Sie die Wörterbuch-Konfigurationsdateien herunter. OpenOffice-Erweiterungsdateien haben die Erweiterung `.oxt`. Daraus müssen die Dateien `.aff` und `.dic` extrahiert und die Erweiterungen in `.affix` und `.dict` geändert werden. Bei manchen Wörterbuchdateien müssen die Zeichen außerdem in die UTF-8-Kodierung umgewandelt werden, zum Beispiel für ein norwegisches Wörterbuch:

```text
iconv -f ISO_8859-1 -t UTF-8 -o nn_no.affix nn_NO.aff
iconv -f ISO_8859-1 -t UTF-8 -o nn_no.dict nn_NO.dic
```

- Kopieren Sie die Dateien in das Verzeichnis `$SHAREDIR/tsearch_data`.
- Laden Sie die Dateien mit folgendem Befehl in PostgreSQL:

```sql
CREATE TEXT SEARCH DICTIONARY english_hunspell (
    TEMPLATE = ispell,
    DictFile = en_us,
    AffFile = en_us,
    Stopwords = english);
```

Hier geben `DictFile`, `AffFile` und `StopWords` die Basisnamen der Wörterbuch-, Affix- und Stoppwortdateien an. Die Stoppwortdatei hat dasselbe Format, das oben für den einfachen Wörterbuchtyp erläutert wurde. Das Format der anderen Dateien wird hier nicht angegeben, ist aber auf den oben genannten Webseiten verfügbar.

Ispell-Wörterbücher erkennen normalerweise nur eine begrenzte Wortmenge und sollten daher von einem weiteren, breiteren Wörterbuch gefolgt werden, zum Beispiel einem Snowball-Wörterbuch, das alles erkennt.

Die `.affix`-Datei von Ispell hat folgende Struktur:

```text
prefixes
flag *A:
    .                         >    RE        # As in enter > reenter
suffixes
flag T:
    E                         >    ST      # As in late > latest
    [^AEIOU]Y                 >    -Y,IEST # As in dirty > dirtiest
    [AEIOU]Y                  >    EST     # As in gray > grayest
    [^EY]                     >    EST     # As in small > smallest
```

Die `.dict`-Datei hat folgende Struktur:

```text
lapse/ADGRS
lard/DGRS
large/PRTY
lark/MRS
```

Das Format der `.dict`-Datei ist:

```text
basic_form/affix_class_name
```

In der `.affix`-Datei wird jedes Affix-Flag im folgenden Format beschrieben:

```text
condition > [-stripping_letters,] adding_affix
```

Hier hat `condition` ein Format, das dem Format regulärer Ausdrücke ähnelt. Es kann Gruppierungen wie `[...]` und `[^...]` verwenden. Zum Beispiel bedeutet `[AEIOU]Y`, dass der letzte Buchstabe des Wortes `y` ist und der vorletzte Buchstabe `a`, `e`, `i`, `o` oder `u`. `[^EY]` bedeutet, dass der letzte Buchstabe weder `e` noch `y` ist.

Ispell-Wörterbücher unterstützen das Aufteilen zusammengesetzter Wörter, eine nützliche Eigenschaft. Beachten Sie, dass die Affix-Datei mit der Anweisung `compoundwords controlled` ein spezielles Flag angeben sollte, das Wörterbuchwörter markiert, die an der Bildung zusammengesetzter Wörter teilnehmen können:

```text
compoundwords           controlled z
```

Hier sind einige Beispiele für die norwegische Sprache:

```sql
SELECT ts_lexize('norwegian_ispell',
 'overbuljongterningpakkmesterassistent');
```

```text
{over,buljong,terning,pakk,mester,assistent}
```

```sql
SELECT ts_lexize('norwegian_ispell', 'sjokoladefabrikk');
```

```text
{sjokoladefabrikk,sjokolade,fabrikk}
```

Das MySpell-Format ist eine Teilmenge von Hunspell. Die `.affix`-Datei von Hunspell hat folgende Struktur:

```text
PFX A Y 1
PFX A   0             re                .
SFX T N 4
SFX T   0             st                e
SFX T   y             iest              [^aeiou]y
SFX T   0             est               [aeiou]y
SFX T   0             est               [^ey]
```

Die erste Zeile einer Affix-Klasse ist der Header. Nach dem Header werden die Felder einer Affix-Regel aufgelistet:

- Parametername (`PFX` oder `SFX`)
- Flag, also Name der Affix-Klasse
- Zeichen, die am Anfang bei einem Präfix oder am Ende bei einem Suffix vom Wort entfernt werden
- hinzuzufügendes Affix
- Bedingung in einem Format, das dem Format regulärer Ausdrücke ähnelt

Die `.dict`-Datei sieht wie die `.dict`-Datei von Ispell aus:

```text
larder/M
lardy/RT
large/RSPMYT
largehearted
```

> Hinweis: MySpell unterstützt keine zusammengesetzten Wörter. Hunspell bietet ausgefeilte Unterstützung für zusammengesetzte Wörter. Derzeit implementiert PostgreSQL nur die grundlegenden Operationen von Hunspell für zusammengesetzte Wörter.

### 12.6.6. Snowball-Wörterbuch

Die Vorlage für Snowball-Wörterbücher basiert auf einem Projekt von Martin Porter, dem Erfinder des beliebten Porter-Stemming-Algorithmus für die englische Sprache. Snowball stellt inzwischen Stemming-Algorithmen für viele Sprachen bereit; weitere Informationen finden Sie auf der [Snowball-Seite](https://snowballstem.org/). Jeder Algorithmus weiß, wie übliche Variantenformen von Wörtern in seiner Sprache auf eine Grund- oder Stamm-Schreibweise zurückgeführt werden. Ein Snowball-Wörterbuch benötigt einen Sprachparameter, der angibt, welcher Stemmer verwendet werden soll, und kann optional den Namen einer Stoppwortdatei angeben, die eine Liste zu entfernender Wörter enthält. Die Standard-Stoppwortlisten von PostgreSQL werden ebenfalls vom Snowball-Projekt bereitgestellt. Zum Beispiel gibt es eine eingebaute Definition, die Folgendem entspricht:

```sql
CREATE TEXT SEARCH DICTIONARY english_stem (
    TEMPLATE = snowball,
    Language = english,
    StopWords = english
);
```

Das Format der Stoppwortdatei ist dasselbe wie bereits erläutert.

Ein Snowball-Wörterbuch erkennt alles, unabhängig davon, ob es das Wort vereinfachen kann oder nicht. Daher sollte es am Ende der Wörterbuchliste stehen. Vor einem anderen Wörterbuch ist es nutzlos, weil ein Token niemals durch es hindurch an das nächste Wörterbuch weitergereicht wird.

## 12.7. Konfigurationsbeispiel

Eine Textsuchkonfiguration legt alle Optionen fest, die nötig sind, um ein Dokument in einen `tsvector` umzuwandeln: den Parser, der den Text in Tokens zerlegt, und die Wörterbücher, die jedes Token in ein Lexem umwandeln. Jeder Aufruf von `to_tsvector` oder `to_tsquery` benötigt eine Textsuchkonfiguration, um seine Verarbeitung auszuführen. Der Konfigurationsparameter `default_text_search_config` gibt den Namen der Standardkonfiguration an, also jener Konfiguration, die Textsuchfunktionen verwenden, wenn kein expliziter Konfigurationsparameter angegeben wird. Er kann in `postgresql.conf` gesetzt oder mit dem Befehl `SET` für eine einzelne Sitzung festgelegt werden.

Es sind mehrere vordefinierte Textsuchkonfigurationen verfügbar, und eigene Konfigurationen können leicht erstellt werden. Zur Verwaltung von Textsuchobjekten steht eine Reihe von SQL-Befehlen zur Verfügung; außerdem gibt es mehrere `psql`-Befehle, die Informationen über Textsuchobjekte anzeigen ([Abschnitt 12.10](#1210-psqlunterstützung)).

Als Beispiel erstellen wir eine Konfiguration `pg`, indem wir zunächst die eingebaute Konfiguration `english` duplizieren:

```sql
CREATE TEXT SEARCH CONFIGURATION public.pg ( COPY =
 pg_catalog.english );
```

Wir verwenden eine PostgreSQL-spezifische Synonymliste und speichern sie in `$SHAREDIR/tsearch_data/pg_dict.syn`. Der Dateiinhalt sieht so aus:

```text
postgres           pg
pgsql              pg
postgresql         pg
```

Das Synonymwörterbuch definieren wir so:

```sql
CREATE TEXT SEARCH DICTIONARY pg_dict (
    TEMPLATE = synonym,
    SYNONYMS = pg_dict
);
```

Als Nächstes registrieren wir das Ispell-Wörterbuch `english_ispell`, das eigene Konfigurationsdateien besitzt:

```sql
CREATE TEXT SEARCH DICTIONARY english_ispell (
    TEMPLATE = ispell,
    DictFile = english,
    AffFile = english,
    StopWords = english
);
```

Nun können wir die Zuordnungen für Wörter in der Konfiguration `pg` einrichten:

```sql
ALTER TEXT SEARCH CONFIGURATION pg
    ALTER MAPPING FOR asciiword, asciihword, hword_asciipart,
                      word, hword, hword_part
    WITH pg_dict, english_ispell, english_stem;
```

Wir entscheiden uns, einige Token-Typen, die die eingebaute Konfiguration behandelt, nicht zu indizieren oder zu suchen:

```sql
ALTER TEXT SEARCH CONFIGURATION pg
    DROP MAPPING FOR email, url, url_path, sfloat, float;
```

Nun können wir unsere Konfiguration testen:

```sql
SELECT * FROM ts_debug('public.pg', '
PostgreSQL, the highly scalable, SQL compliant, open source object-
relational
database management system, is now undergoing beta testing of the
next
version of our software.
');
```

Der nächste Schritt besteht darin, die Sitzung auf die neue Konfiguration zu setzen, die im Schema `public` erstellt wurde:

```text
=> \dF
   List of text search configurations
 Schema | Name | Description
--------+------+-------------
 public | pg   |

SET default_text_search_config = 'public.pg';
SET

SHOW default_text_search_config;
 default_text_search_config
----------------------------
 public.pg
```

## 12.8. Textsuche testen und debuggen

Das Verhalten einer eigenen Textsuchkonfiguration kann leicht unübersichtlich werden. Die in diesem Abschnitt beschriebenen Funktionen sind nützlich, um Textsuchobjekte zu testen. Sie können eine vollständige Konfiguration testen oder Parser und Wörterbücher getrennt prüfen.

### 12.8.1. Konfiguration testen

Die Funktion `ts_debug` erlaubt ein einfaches Testen einer Textsuchkonfiguration.

```text
ts_debug([ config regconfig, ] document text,
         OUT alias text,
         OUT description text,
         OUT token text,
         OUT dictionaries regdictionary[],
         OUT dictionary regdictionary,
         OUT lexemes text[])
         returns setof record
```

`ts_debug` zeigt Informationen über jedes Token von `document` an, wie es vom Parser erzeugt und von den konfigurierten Wörterbüchern verarbeitet wurde. Die Funktion verwendet die durch `config` angegebene Konfiguration oder `default_text_search_config`, wenn dieses Argument weggelassen wird.

`ts_debug` gibt für jedes Token, das der Parser im Text identifiziert, eine Zeile zurück. Die zurückgegebenen Spalten sind:

- `alias text`: Kurzname des Token-Typs
- `description text`: Beschreibung des Token-Typs
- `token text`: Text des Tokens
- `dictionaries regdictionary[]`: die Wörterbücher, die die Konfiguration für diesen Token-Typ ausgewählt hat
- `dictionary regdictionary`: das Wörterbuch, das das Token erkannt hat, oder `NULL`, wenn keines es erkannt hat
- `lexemes text[]`: die von dem erkennenden Wörterbuch erzeugten Lexeme oder `NULL`, wenn keines es erkannt hat; ein leeres Array (`{}`) bedeutet, dass es als Stoppwort erkannt wurde

Hier ist ein einfaches Beispiel:

```sql
SELECT * FROM ts_debug('english', 'a fat cat sat on a mat - it ate a fat rats');
```

```text
   alias   |   description   | token |  dictionaries  |  dictionary  | lexemes
-----------+-----------------+-------+----------------+--------------+---------
 asciiword | Word, all ASCII | a     | {english_stem} | english_stem | {}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | fat   | {english_stem} | english_stem | {fat}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | cat   | {english_stem} | english_stem | {cat}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | sat   | {english_stem} | english_stem | {sat}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | on    | {english_stem} | english_stem | {}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | a     | {english_stem} | english_stem | {}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | mat   | {english_stem} | english_stem | {mat}
 blank     | Space symbols   |       | {}             |              |
 blank     | Space symbols   | -     | {}             |              |
 asciiword | Word, all ASCII | it    | {english_stem} | english_stem | {}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | ate   | {english_stem} | english_stem | {ate}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | a     | {english_stem} | english_stem | {}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | fat   | {english_stem} | english_stem | {fat}
 blank     | Space symbols   |       | {}             |              |
 asciiword | Word, all ASCII | rats  | {english_stem} | english_stem | {rat}
```

Für eine ausführlichere Demonstration erstellen wir zunächst eine Konfiguration `public.english` und ein Ispell-Wörterbuch für die englische Sprache:

```sql
CREATE TEXT SEARCH CONFIGURATION public.english ( COPY =
 pg_catalog.english );

CREATE TEXT SEARCH DICTIONARY english_ispell (
    TEMPLATE = ispell,
    DictFile = english,
    AffFile = english,
    StopWords = english
);

ALTER TEXT SEARCH CONFIGURATION public.english
   ALTER MAPPING FOR asciiword WITH english_ispell, english_stem;

SELECT * FROM ts_debug('public.english', 'The Brightest
 supernovaes');
```

```text
   alias   |   description   |    token    |         dictionaries          |   dictionary   |  lexemes
-----------+-----------------+-------------+-------------------------------+----------------+-------------
 asciiword | Word, all ASCII | The         | {english_ispell,english_stem} | english_ispell | {}
 blank     | Space symbols   |             | {}                            |                |
 asciiword | Word, all ASCII | Brightest   | {english_ispell,english_stem} | english_ispell | {bright}
 blank     | Space symbols   |             | {}                            |                |
 asciiword | Word, all ASCII | supernovaes | {english_ispell,english_stem} | english_stem   | {supernova}
```

In diesem Beispiel wurde das Wort `Brightest` vom Parser als ASCII-Wort erkannt, also mit Alias `asciiword`. Für diesen Token-Typ lautet die Wörterbuchliste `english_ispell` und `english_stem`. Das Wort wurde von `english_ispell` erkannt, das es auf das Substantiv `bright` reduzierte. Das Wort `supernovaes` ist dem Wörterbuch `english_ispell` unbekannt, daher wurde es an das nächste Wörterbuch weitergereicht und glücklicherweise erkannt. Tatsächlich ist `english_stem` ein Snowball-Wörterbuch, das alles erkennt; deshalb wurde es ans Ende der Wörterbuchliste gesetzt.

Das Wort `The` wurde vom Wörterbuch `english_ispell` als Stoppwort erkannt ([Abschnitt 12.6.1](#1261-stoppwörter)) und wird nicht indiziert. Die Leerzeichen werden ebenfalls verworfen, da die Konfiguration für sie überhaupt keine Wörterbücher bereitstellt.

Sie können die Breite der Ausgabe verringern, indem Sie ausdrücklich angeben, welche Spalten Sie sehen möchten:

```sql
SELECT alias, token, dictionary, lexemes
FROM ts_debug('public.english', 'The Brightest supernovaes');
```

```text
   alias   |    token    |   dictionary   |   lexemes
-----------+-------------+----------------+-------------
 asciiword | The         | english_ispell | {}
 blank     |             |                |
 asciiword | Brightest   | english_ispell | {bright}
 blank     |             |                |
 asciiword | supernovaes | english_stem   | {supernova}
```

### 12.8.2. Parser testen

Die folgenden Funktionen erlauben das direkte Testen eines Textsuchparsers.

```text
ts_parse(parser_name text, document text,
         OUT tokid integer, OUT token text) returns setof record
ts_parse(parser_oid oid, document text,
         OUT tokid integer, OUT token text) returns setof record
```

`ts_parse` parst das angegebene Dokument und gibt eine Reihe von Datensätzen zurück, einen für jedes beim Parsen erzeugte Token. Jeder Datensatz enthält eine `tokid`, die den zugewiesenen Token-Typ zeigt, und ein `token`, das den Text des Tokens enthält. Zum Beispiel:

```sql
SELECT * FROM ts_parse('default', '123 - a number');
```

```text
 tokid | token
-------+--------
    22 | 123
    12 |
    12 | -
     1 | a
    12 |
     1 | number
```

```text
ts_token_type(parser_name text, OUT tokid integer,
              OUT alias text, OUT description text) returns setof record
ts_token_type(parser_oid oid, OUT tokid integer,
              OUT alias text, OUT description text) returns setof record
```

`ts_token_type` gibt eine Tabelle zurück, die jeden Token-Typ beschreibt, den der angegebene Parser erkennen kann. Für jeden Token-Typ enthält die Tabelle die ganzzahlige `tokid`, die der Parser zum Kennzeichnen eines Tokens dieses Typs verwendet, den Alias, der den Token-Typ in Konfigurationsbefehlen benennt, und eine kurze Beschreibung. Zum Beispiel:

```sql
SELECT * FROM ts_token_type('default');
```

```text
 tokid |      alias       |               description
-------+------------------+------------------------------------------
     1 | asciiword        | Word, all ASCII
     2 | word             | Word, all letters
     3 | numword          | Word, letters and digits
     4 | email            | Email address
     5 | url              | URL
     6 | host             | Host
     7 | sfloat           | Scientific notation
     8 | version          | Version number
     9 | hword_numpart    | Hyphenated word part, letters and digits
    10 | hword_part       | Hyphenated word part, all letters
    11 | hword_asciipart  | Hyphenated word part, all ASCII
    12 | blank            | Space symbols
    13 | tag              | XML tag
    14 | protocol         | Protocol head
    15 | numhword         | Hyphenated word, letters and digits
    16 | asciihword       | Hyphenated word, all ASCII
    17 | hword            | Hyphenated word, all letters
    18 | url_path         | URL path
    19 | file             | File or path name
    20 | float            | Decimal notation
    21 | int              | Signed integer
    22 | uint             | Unsigned integer
    23 | entity           | XML entity
```

### 12.8.3. Wörterbuch testen

Die Funktion `ts_lexize` erleichtert das Testen von Wörterbüchern.

```text
ts_lexize(dict regdictionary, token text) returns text[]
```

`ts_lexize` gibt ein Array von Lexemen zurück, wenn das Eingabe-Token dem Wörterbuch bekannt ist, ein leeres Array, wenn das Token dem Wörterbuch bekannt ist, aber ein Stoppwort ist, oder `NULL`, wenn es ein unbekanntes Wort ist.

Beispiele:

```sql
SELECT ts_lexize('english_stem', 'stars');
```

```text
 ts_lexize
-----------
 {star}
```

```sql
SELECT ts_lexize('english_stem', 'a');
```

```text
 ts_lexize
-----------
 {}
```

> Hinweis: Die Funktion `ts_lexize` erwartet ein einzelnes Token, nicht Text. Hier ist ein Fall, in dem das verwirrend sein kann:
>
> ```sql
> SELECT ts_lexize('thesaurus_astro', 'supernovae stars') is null;
> ```
>
> ```text
>  ?column?
> ----------
>  t
> ```
>
> Das Thesaurus-Wörterbuch `thesaurus_astro` kennt die Phrase `supernovae stars`, aber `ts_lexize` schlägt fehl, weil es den Eingabetext nicht parst, sondern als einzelnes Token behandelt. Verwenden Sie zum Testen von Thesaurus-Wörterbüchern `plainto_tsquery` oder `to_tsvector`, zum Beispiel:
>
> ```sql
> SELECT plainto_tsquery('supernovae stars');
> ```
>
> ```text
>  plainto_tsquery
> -----------------
>  'sn'
> ```

## 12.9. Bevorzugte Indextypen für Textsuche

Es gibt zwei Arten von Indizes, mit denen Volltextsuchen beschleunigt werden können: GIN und GiST. Beachten Sie, dass Indizes für die Volltextsuche nicht zwingend erforderlich sind. Wenn eine Spalte jedoch regelmäßig durchsucht wird, ist ein Index normalerweise wünschenswert.

Um einen solchen Index zu erstellen, verwenden Sie eine der folgenden Formen:

```sql
CREATE INDEX name ON table USING GIN (column);
```

Dies erstellt einen GIN-basierten Index (`Generalized Inverted Index`). Die Spalte muss vom Typ `tsvector` sein.

```sql
CREATE INDEX name ON table USING GIST (column [ { DEFAULT | tsvector_ops } (siglen = number) ] );
```

Dies erstellt einen GiST-basierten Index (`Generalized Search Tree`). Die Spalte kann vom Typ `tsvector` oder `tsquery` sein. Der optionale ganzzahlige Parameter `siglen` bestimmt die Signaturlänge in Byte; Details folgen unten.

GIN-Indizes sind der bevorzugte Indextyp für Textsuche. Als invertierte Indizes enthalten sie für jedes Wort, also jedes Lexem, einen Indexeintrag mit einer komprimierten Liste passender Fundorte. Mehrwortsuchen können die erste Übereinstimmung finden und dann den Index verwenden, um Zeilen zu entfernen, denen weitere Wörter fehlen. GIN-Indizes speichern nur die Wörter, also Lexeme, von `tsvector`-Werten, nicht aber ihre Gewichtungskennzeichnungen. Daher ist ein erneutes Prüfen der Tabellenzeile nötig, wenn eine Abfrage Gewichtungen verwendet.

Ein GiST-Index ist verlustbehaftet, was bedeutet, dass der Index falsche Treffer liefern kann und die tatsächliche Tabellenzeile geprüft werden muss, um solche falschen Treffer zu entfernen. PostgreSQL tut dies bei Bedarf automatisch. GiST-Indizes sind verlustbehaftet, weil jedes Dokument im Index durch eine Signatur fester Länge dargestellt wird. Die Signaturlänge in Byte wird durch den Wert des optionalen ganzzahligen Parameters `siglen` bestimmt. Die Standardsignaturlänge, wenn `siglen` nicht angegeben ist, beträgt 124 Byte; die maximale Signaturlänge beträgt 2024 Byte. Die Signatur wird erzeugt, indem jedes Wort auf ein einzelnes Bit in einer `n`-Bit-Zeichenkette gehasht wird und all diese Bits per OR zu einer `n`-Bit-Dokumentsignatur zusammengeführt werden. Wenn zwei Wörter auf dieselbe Bitposition gehasht werden, entsteht ein falscher Treffer. Wenn alle Wörter in der Abfrage Treffer haben, echte oder falsche, muss die Tabellenzeile abgerufen werden, um zu prüfen, ob die Übereinstimmung korrekt ist. Längere Signaturen führen zu einer präziseren Suche, bei der ein kleinerer Teil des Indexes und weniger Heap-Seiten gescannt werden, allerdings auf Kosten eines größeren Indexes.

Ein GiST-Index kann ein Covering-Index sein, also die `INCLUDE`-Klausel verwenden. Enthaltene Spalten können Datentypen ohne GiST-Operator-Klasse haben. Enthaltene Attribute werden unkomprimiert gespeichert.

Die Verlustbehaftung führt zu Performanceeinbußen durch unnötige Zugriffe auf Tabellendatensätze, die sich als falsche Treffer herausstellen. Da wahlfreier Zugriff auf Tabellendatensätze langsam ist, begrenzt dies die Nützlichkeit von GiST-Indizes. Die Wahrscheinlichkeit falscher Treffer hängt von mehreren Faktoren ab, insbesondere von der Anzahl unterschiedlicher Wörter. Daher wird empfohlen, Wörterbücher zu verwenden, um diese Anzahl zu verringern.

Beachten Sie, dass sich die Erstellungszeit von GIN-Indizes häufig durch Erhöhen von `maintenance_work_mem` verbessern lässt, während die Erstellungszeit von GiST-Indizes für diesen Parameter unempfindlich ist.

Partitionierung großer Sammlungen und die richtige Verwendung von GIN- und GiST-Indizes ermöglichen die Umsetzung sehr schneller Suchen mit Online-Aktualisierung. Partitionierung kann auf Datenbankebene mithilfe von Tabellenvererbung erfolgen oder indem Dokumente auf Server verteilt und externe Suchergebnisse gesammelt werden, zum Beispiel über Foreign Data Access. Letzteres ist möglich, weil Ranking-Funktionen nur lokale Informationen verwenden.

## 12.10. psql-Unterstützung

Informationen über Textsuch-Konfigurationsobjekte können in `psql` mit einer Reihe von Befehlen abgerufen werden:

```text
\dF{d,p,t}[+] [PATTERN]
```

Ein optionales `+` erzeugt mehr Details.

Der optionale Parameter `PATTERN` kann der Name eines Textsuchobjekts sein, optional schemaqualifiziert. Wenn `PATTERN` weggelassen wird, werden Informationen über alle sichtbaren Objekte angezeigt. `PATTERN` kann ein regulärer Ausdruck sein und getrennte Muster für Schema- und Objektnamen bereitstellen. Die folgenden Beispiele veranschaulichen dies:

```text
=> \dF *fulltext*
       List of text search configurations
 Schema | Name         | Description
--------+--------------+-------------
 public | fulltext_cfg |

=> \dF *.fulltext*
        List of text search configurations
 Schema   | Name         | Description
----------+--------------+-------------
 fulltext | fulltext_cfg |
 public   | fulltext_cfg |
```

Die verfügbaren Befehle sind:

```text
\dF[+] [PATTERN]
```

Textsuchkonfigurationen auflisten; mit `+` werden mehr Details angezeigt.

```text
=> \dF russian
            List of text search configurations
   Schema   | Name    |            Description
------------+---------+------------------------------------
 pg_catalog | russian | configuration for russian language

=> \dF+ russian
Text search configuration "pg_catalog.russian"
Parser: "pg_catalog.default"
       Token     | Dictionaries
-----------------+--------------
 asciihword      | english_stem
 asciiword       | english_stem
 email           | simple
 file            | simple
 float           | simple
 host            | simple
 hword           | russian_stem
 hword_asciipart | english_stem
 hword_numpart   | simple
 hword_part      | russian_stem
 int             | simple
 numhword        | simple
 numword         | simple
 sfloat          | simple
 uint            | simple
 url             | simple
 url_path        | simple
 version         | simple
 word            | russian_stem
```

```text
\dFd[+] [PATTERN]
```

Textsuchwörterbücher auflisten; mit `+` werden mehr Details angezeigt.

```text
=> \dFd
                            List of text search dictionaries
   Schema   |      Name       |              Description
------------+-----------------+----------------------------------------
 pg_catalog | arabic_stem     | snowball stemmer for arabic language
 pg_catalog | armenian_stem   | snowball stemmer for armenian language
 pg_catalog | basque_stem     | snowball stemmer for basque language
 pg_catalog | catalan_stem    | snowball stemmer for catalan language
 pg_catalog | danish_stem     | snowball stemmer for danish language
 pg_catalog | dutch_stem      | snowball stemmer for dutch language
 pg_catalog | english_stem    | snowball stemmer for english language
 pg_catalog | estonian_stem   | snowball stemmer for estonian language
 pg_catalog | finnish_stem    | snowball stemmer for finnish language
 pg_catalog | french_stem     | snowball stemmer for french language
 pg_catalog | german_stem     | snowball stemmer for german language
 pg_catalog | greek_stem      | snowball stemmer for greek language
 pg_catalog | hindi_stem      | snowball stemmer for hindi language
 pg_catalog | hungarian_stem  | snowball stemmer for hungarian language
 pg_catalog | indonesian_stem | snowball stemmer for indonesian language
 pg_catalog | irish_stem      | snowball stemmer for irish language
 pg_catalog | italian_stem    | snowball stemmer for italian language
 pg_catalog | lithuanian_stem | snowball stemmer for lithuanian language
 pg_catalog | nepali_stem     | snowball stemmer for nepali language
 pg_catalog | norwegian_stem  | snowball stemmer for norwegian language
 pg_catalog | portuguese_stem | snowball stemmer for portuguese language
 pg_catalog | romanian_stem   | snowball stemmer for romanian language
 pg_catalog | russian_stem    | snowball stemmer for russian language
 pg_catalog | serbian_stem    | snowball stemmer for serbian language
 pg_catalog | simple          | simple dictionary: just lower case and check for stopword
 pg_catalog | spanish_stem    | snowball stemmer for spanish language
 pg_catalog | swedish_stem    | snowball stemmer for swedish language
 pg_catalog | tamil_stem      | snowball stemmer for tamil language
 pg_catalog | turkish_stem    | snowball stemmer for turkish language
 pg_catalog | yiddish_stem    | snowball stemmer for yiddish language
```

```text
\dFp[+] [PATTERN]
```

Textsuchparser auflisten; mit `+` werden mehr Details angezeigt.

```text
=> \dFp
         List of text search parsers
   Schema   |  Name   |     Description
------------+---------+---------------------
 pg_catalog | default | default word parser

=> \dFp+
    Text search parser "pg_catalog.default"
     Method       |    Function    | Description
------------------+----------------+-------------
 Start parse      | prsd_start     |
 Get next token   | prsd_nexttoken |
 End parse        | prsd_end       |
 Get headline     | prsd_headline  |
 Get token types  | prsd_lextype   |

              Token types for parser "pg_catalog.default"
      Token name     |                Description
---------------------+------------------------------------------
 asciihword          | Hyphenated word, all ASCII
 asciiword           | Word, all ASCII
 blank               | Space symbols
 email               | Email address
 entity              | XML entity
 file                | File or path name
 float               | Decimal notation
 host                | Host
 hword               | Hyphenated word, all letters
 hword_asciipart     | Hyphenated word part, all ASCII
 hword_numpart       | Hyphenated word part, letters and digits
 hword_part          | Hyphenated word part, all letters
 int                 | Signed integer
 numhword            | Hyphenated word, letters and digits
 numword             | Word, letters and digits
 protocol            | Protocol head
 sfloat              | Scientific notation
 tag                 | XML tag
 uint                | Unsigned integer
 url                 | URL
 url_path            | URL path
 version             | Version number
 word                | Word, all letters
(23 rows)
```

```text
\dFt[+] [PATTERN]
```

Textsuchvorlagen auflisten; mit `+` werden mehr Details angezeigt.

```text
=> \dFt
                         List of text search templates
   Schema   |   Name    |                         Description
------------+-----------+-------------------------------------------------------------
 pg_catalog | ispell    | ispell dictionary
 pg_catalog | simple    | simple dictionary: just lower case and check for stopword
 pg_catalog | snowball  | snowball stemmer
 pg_catalog | synonym   | synonym dictionary: replace word by its synonym
 pg_catalog | thesaurus | thesaurus dictionary: phrase by phrase substitution
```

## 12.11. Einschränkungen

Die aktuellen Einschränkungen der Textsuchfunktionen von PostgreSQL sind:

- Die Länge jedes Lexems muss kleiner als 2 Kilobyte sein.
- Die Länge eines `tsvector` (Lexeme + Positionen) muss kleiner als 1 Megabyte sein.
- Die Anzahl der Lexeme muss kleiner als 2^64 sein.
- Positionswerte in einem `tsvector` müssen größer als 0 und höchstens 16.383 sein.
- Die Trefferentfernung in einem `<N>`-Operator (`FOLLOWED BY`) von `tsquery` darf höchstens 16.384 sein.
- Es sind höchstens 256 Positionen pro Lexem erlaubt.
- Die Anzahl der Knoten (Lexeme + Operatoren) in einem `tsquery` muss kleiner als 32.768 sein.

Zum Vergleich: Die Dokumentation von PostgreSQL 8.1 enthielt 10.441 eindeutige Wörter, insgesamt 335.420 Wörter, und das häufigste Wort `postgresql` wurde 6.127-mal in 655 Dokumenten erwähnt.

Ein weiteres Beispiel: Die Archive der PostgreSQL-Mailinglisten enthielten 910.989 eindeutige Wörter mit 57.491.343 Lexemen in 461.020 Nachrichten.
