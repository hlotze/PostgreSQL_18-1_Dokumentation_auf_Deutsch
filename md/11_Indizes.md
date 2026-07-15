# 11. Indizes

Indizes sind ein verbreitetes Mittel, um die Leistung einer Datenbank zu verbessern. Ein Index erlaubt es dem Datenbankserver, bestimmte Zeilen wesentlich schneller zu finden und abzurufen, als es ohne Index möglich wäre. Indizes verursachen aber auch Aufwand für das Datenbanksystem insgesamt; sie sollten deshalb mit Bedacht eingesetzt werden.

## 11.1. Einführung

Angenommen, es gibt eine Tabelle wie diese:

```sql
CREATE TABLE test1 (
    id integer,
    content varchar
);
```

und die Anwendung führt viele Abfragen dieser Form aus:

```sql
SELECT content FROM test1 WHERE id = constant;
```

Ohne vorbereitende Maßnahmen müsste das System die gesamte Tabelle `test1` Zeile für Zeile durchsuchen, um alle passenden Einträge zu finden. Wenn `test1` viele Zeilen enthält und nur wenige Zeilen, vielleicht auch gar keine oder nur eine, von einer solchen Abfrage zurückgegeben würden, ist das offensichtlich ineffizient. Wenn das System aber angewiesen wurde, einen Index auf der Spalte `id` zu pflegen, kann es eine effizientere Methode zum Auffinden passender Zeilen verwenden. Es muss dann zum Beispiel nur wenige Ebenen eines Suchbaums durchlaufen.

Ein ähnlicher Ansatz wird in den meisten Sachbüchern verwendet: Begriffe und Konzepte, nach denen Leser häufig suchen, werden am Ende des Buches in einem alphabetischen Register gesammelt. Wer einen bestimmten Inhalt sucht, kann dieses Register relativ schnell durchsuchen und zu den passenden Seiten springen, statt das ganze Buch lesen zu müssen. So wie es Aufgabe des Autors ist, vorherzusehen, wonach Leser wahrscheinlich suchen werden, ist es Aufgabe des Datenbankprogrammierers, vorherzusehen, welche Indizes nützlich sein werden.

Der folgende Befehl erzeugt den besprochenen Index auf der Spalte `id`:

```sql
CREATE INDEX test1_id_index ON test1 (id);
```

Der Name `test1_id_index` kann frei gewählt werden, sollte aber so gewählt sein, dass später noch erkennbar ist, wofür der Index gedacht war.

Um einen Index zu entfernen, verwenden Sie den Befehl `DROP INDEX`. Indizes können jederzeit zu Tabellen hinzugefügt und wieder aus ihnen entfernt werden.

Sobald ein Index angelegt ist, ist keine weitere Intervention nötig: Das System aktualisiert den Index, wenn die Tabelle geändert wird, und verwendet ihn in Abfragen, wenn es dies für effizienter hält als einen sequenziellen Tabellenscan. Allerdings müssen Sie unter Umständen regelmäßig den Befehl `ANALYZE` ausführen, damit die Statistiken aktuell bleiben und der Query Planner fundierte Entscheidungen treffen kann. Informationen dazu, wie Sie feststellen, ob ein Index verwendet wird und wann und warum der Planner sich gegen einen Index entscheidet, finden Sie in [Kapitel 14](14_Hinweise_zur_Performance.md).

Indizes können auch `UPDATE`- und `DELETE`-Befehle mit Suchbedingungen beschleunigen. Außerdem können Indizes bei Join-Suchen verwendet werden. Ein Index auf einer Spalte, die Teil einer Join-Bedingung ist, kann Abfragen mit Joins daher ebenfalls erheblich beschleunigen.

Allgemein können PostgreSQL-Indizes verwendet werden, um Abfragen zu optimieren, die eine oder mehrere `WHERE`- oder `JOIN`-Klauseln dieser Form enthalten:

```text
indexed-column indexable-operator comparison-value
```

Dabei ist `indexed-column` die Spalte oder der Ausdruck, auf der bzw. dem der Index definiert wurde. `indexable-operator` ist ein Operator, der zur Operatorklasse des Indexes für die indizierte Spalte gehört. Weitere Einzelheiten dazu folgen weiter unten. `comparison-value` kann jeder Ausdruck sein, der nicht volatil ist und die Tabelle des Indexes nicht referenziert.

In manchen Fällen kann der Query Planner aus einem anderen SQL-Konstrukt eine indizierbare Klausel dieser Form ableiten. Ein einfaches Beispiel ist eine ursprüngliche Klausel der Form

```text
comparison-value operator indexed-column
```

Sie kann in die indizierbare Form umgedreht werden, wenn der ursprüngliche Operator einen Kommutatoroperator besitzt, der zur Operatorklasse des Indexes gehört.

Das Anlegen eines Indexes auf einer großen Tabelle kann lange dauern. Standardmäßig erlaubt PostgreSQL während des Indexaufbaus parallele Lesezugriffe (`SELECT`-Anweisungen) auf die Tabelle, blockiert aber Schreibzugriffe (`INSERT`, `UPDATE`, `DELETE`), bis der Index fertig aufgebaut ist. In Produktionsumgebungen ist das häufig nicht akzeptabel. Es ist möglich, Schreibzugriffe parallel zum Indexaufbau zuzulassen; dabei sind jedoch einige Einschränkungen zu beachten. Weitere Informationen finden Sie unter „Building Indexes Concurrently“.

Nachdem ein Index angelegt wurde, muss das System ihn mit der Tabelle synchron halten. Das verursacht zusätzlichen Aufwand bei Datenmanipulationsoperationen. Indizes können außerdem die Erzeugung von Heap-Only-Tuples verhindern. Indizes, die in Abfragen selten oder nie verwendet werden, sollten deshalb entfernt werden.

## 11.2. Indextypen

PostgreSQL stellt mehrere Indextypen bereit: B-Tree, Hash, GiST, SP-GiST, GIN, BRIN sowie die Erweiterung `bloom`. Jeder Indextyp verwendet einen anderen Algorithmus, der für unterschiedliche Arten indizierbarer Klauseln geeignet ist. Standardmäßig erzeugt `CREATE INDEX` B-Tree-Indizes; diese passen zu den häufigsten Situationen. Andere Indextypen werden gewählt, indem nach dem Schlüsselwort `USING` der Name des Indextyps angegeben wird. Um zum Beispiel einen Hash-Index zu erstellen:

```sql
CREATE INDEX name ON table USING HASH (column);
```

### 11.2.1. B-Tree

B-Trees können Gleichheits- und Bereichsabfragen auf Daten verarbeiten, die sortierbar sind. Der PostgreSQL-Query-Planner zieht insbesondere dann einen B-Tree-Index in Betracht, wenn eine indizierte Spalte in einem Vergleich mit einem dieser Operatoren vorkommt:

```text
<     <=      =     >=      >
```

Konstrukte, die Kombinationen dieser Operatoren entsprechen, etwa `BETWEEN` und `IN`, können ebenfalls mit einer B-Tree-Indexsuche umgesetzt werden. Auch eine `IS NULL`- oder `IS NOT NULL`-Bedingung auf einer Indexspalte kann mit einem B-Tree-Index verwendet werden.

Der Optimierer kann einen B-Tree-Index außerdem für Abfragen mit den Pattern-Matching-Operatoren `LIKE` und `~` verwenden, wenn das Muster konstant ist und am Anfang der Zeichenkette verankert ist, zum Beispiel `col LIKE 'foo%'` oder `col ~ '^foo'`, aber nicht `col LIKE '%bar'`. Wenn Ihre Datenbank nicht die Locale `C` verwendet, müssen Sie den Index mit einer speziellen Operatorklasse anlegen, um Pattern-Matching-Abfragen zu unterstützen; siehe [Abschnitt 11.10](#1110-operatorklassen-und-operatorfamilien) weiter unten. B-Tree-Indizes können auch für `ILIKE` und `~*` verwendet werden, allerdings nur dann, wenn das Muster mit nicht alphabetischen Zeichen beginnt, also mit Zeichen, die nicht von Groß-/Kleinschreibung betroffen sind.

B-Tree-Indizes können außerdem verwendet werden, um Daten in sortierter Reihenfolge abzurufen. Das ist nicht immer schneller als ein einfacher Scan mit anschließendem Sortieren, aber oft hilfreich.

### 11.2.2. Hash

Hash-Indizes speichern einen 32-Bit-Hashcode, der aus dem Wert der indizierten Spalte abgeleitet wird. Deshalb können solche Indizes nur einfache Gleichheitsvergleiche verarbeiten. Der Query Planner zieht einen Hash-Index in Betracht, wenn eine indizierte Spalte mit dem Gleichheitsoperator verglichen wird:

```text
=
```

### 11.2.3. GiST

GiST-Indizes sind nicht ein einzelner Indextyp, sondern eine Infrastruktur, innerhalb der viele unterschiedliche Indexierungsstrategien implementiert werden können. Die Operatoren, mit denen ein GiST-Index verwendet werden kann, hängen daher von der jeweiligen Indexierungsstrategie, also der Operatorklasse, ab. Die Standarddistribution von PostgreSQL enthält zum Beispiel GiST-Operatorklassen für mehrere zweidimensionale geometrische Datentypen, die indizierte Abfragen mit diesen Operatoren unterstützen:

```text
<<     &<      &>      >>      <<|       &<|      |&>      |>>       @>      <@      ~=      &&
```

Die Bedeutung dieser Operatoren ist in [Abschnitt 9.11](09_Funktionen_und_Operatoren.md#911-geometrische-funktionen-und-operatoren) beschrieben. Die in der Standarddistribution enthaltenen GiST-Operatorklassen sind in Tabelle 65.1 dokumentiert. Viele weitere GiST-Operatorklassen sind in der `contrib`-Sammlung oder als separate Projekte verfügbar. Weitere Informationen finden Sie in [Abschnitt 65.2](65_Eingebaute_Index_Access_Methods.md#652-gistindizes).

GiST-Indizes können außerdem „Nearest Neighbor“-Suchen optimieren, zum Beispiel:

```sql
SELECT * FROM places ORDER BY location <-> point '(101,456)' LIMIT 10;
```

Diese Abfrage findet die zehn Orte, die einem gegebenen Zielpunkt am nächsten liegen. Auch diese Fähigkeit hängt von der verwendeten Operatorklasse ab. In Tabelle 65.1 sind Operatoren, die auf diese Weise verwendet werden können, in der Spalte „Ordering Operators“ aufgeführt.

### 11.2.4. SP-GiST

SP-GiST-Indizes bieten wie GiST-Indizes eine Infrastruktur, die verschiedene Arten von Suchen unterstützt. SP-GiST erlaubt die Implementierung vieler unterschiedlicher nicht balancierter, plattenbasierter Datenstrukturen, etwa Quadtrees, k-d-Bäume und Radixbäume (Tries). Die Standarddistribution von PostgreSQL enthält zum Beispiel SP-GiST-Operatorklassen für zweidimensionale Punkte, die indizierte Abfragen mit diesen Operatoren unterstützen:

```text
<<     >>      ~=      <@      <<|       |>>
```

Die Bedeutung dieser Operatoren ist in [Abschnitt 9.11](09_Funktionen_und_Operatoren.md#911-geometrische-funktionen-und-operatoren) beschrieben. Die in der Standarddistribution enthaltenen SP-GiST-Operatorklassen sind in Tabelle 65.2 dokumentiert. Weitere Informationen finden Sie in [Abschnitt 65.3](65_Eingebaute_Index_Access_Methods.md#653-spgistindizes).

Wie GiST unterstützt auch SP-GiST „Nearest Neighbor“-Suchen. Für SP-GiST-Operatorklassen, die Distanzsortierung unterstützen, ist der entsprechende Operator in Tabelle 65.2 in der Spalte „Ordering Operators“ aufgeführt.

### 11.2.5. GIN

GIN-Indizes sind „invertierte Indizes“ und eignen sich für Datenwerte, die mehrere Komponentenwerte enthalten, zum Beispiel Arrays. Ein invertierter Index enthält einen separaten Eintrag für jeden Komponentenwert und kann Abfragen effizient verarbeiten, die auf das Vorhandensein bestimmter Komponentenwerte prüfen.

Wie GiST und SP-GiST kann GIN viele unterschiedliche benutzerdefinierte Indexierungsstrategien unterstützen; die Operatoren, mit denen ein GIN-Index verwendet werden kann, hängen von der jeweiligen Indexierungsstrategie ab. Die Standarddistribution von PostgreSQL enthält zum Beispiel eine GIN-Operatorklasse für Arrays, die indizierte Abfragen mit diesen Operatoren unterstützt:

```text
<@     @>      =     &&
```

Die Bedeutung dieser Operatoren ist in [Abschnitt 9.19](09_Funktionen_und_Operatoren.md#919-arrayfunktionen-und-operatoren) beschrieben. Die in der Standarddistribution enthaltenen GIN-Operatorklassen sind in Tabelle 65.3 dokumentiert. Viele weitere GIN-Operatorklassen sind in der `contrib`-Sammlung oder als separate Projekte verfügbar. Weitere Informationen finden Sie in [Abschnitt 65.4](65_Eingebaute_Index_Access_Methods.md#654-ginindizes).

### 11.2.6. BRIN

BRIN-Indizes, kurz für Block Range INdexes, speichern Zusammenfassungen über die Werte, die in aufeinanderfolgenden physischen Blockbereichen einer Tabelle liegen. Sie sind daher am wirksamsten für Spalten, deren Werte gut mit der physischen Reihenfolge der Tabellenzeilen korrelieren. Wie GiST, SP-GiST und GIN kann BRIN viele unterschiedliche Indexierungsstrategien unterstützen; die Operatoren, mit denen ein BRIN-Index verwendet werden kann, hängen von der jeweiligen Strategie ab. Für Datentypen mit einer linearen Sortierreihenfolge entsprechen die indizierten Daten den Minimal- und Maximalwerten der Spaltenwerte für jeden Blockbereich. Dadurch werden indizierte Abfragen mit diesen Operatoren unterstützt:

```text
<     <=     =     >=      >
```

Die in der Standarddistribution enthaltenen BRIN-Operatorklassen sind in Tabelle 65.4 dokumentiert. Weitere Informationen finden Sie in [Abschnitt 65.5](65_Eingebaute_Index_Access_Methods.md#655-brinindizes).

## 11.3. Mehrspaltige Indizes

Ein Index kann auf mehr als einer Spalte einer Tabelle definiert werden. Wenn Sie zum Beispiel eine Tabelle dieser Form haben:

```sql
CREATE TABLE test2 (
   major int,
   minor int,
   name varchar
);
```

und Sie Ihr `/dev`-Verzeichnis, sagen wir, in einer Datenbank halten und häufig Abfragen wie diese ausführen:

```sql
SELECT name FROM test2 WHERE major = constant AND minor = constant;
```

dann kann es sinnvoll sein, einen gemeinsamen Index auf den Spalten `major` und `minor` zu definieren, zum Beispiel:

```sql
CREATE INDEX test2_mm_idx ON test2 (major, minor);
```

Derzeit unterstützen nur die Indextypen B-Tree, GiST, GIN und BRIN Indizes mit mehreren Schlüsselspalten. Ob mehrere Schlüsselspalten möglich sind, ist unabhängig davon, ob `INCLUDE`-Spalten zum Index hinzugefügt werden können. Indizes können bis zu 32 Spalten enthalten, einschließlich `INCLUDE`-Spalten. Diese Grenze kann beim Bauen von PostgreSQL geändert werden; siehe die Datei `pg_config_manual.h`.

Ein mehrspaltiger B-Tree-Index kann mit Abfragebedingungen verwendet werden, die eine beliebige Teilmenge der Indexspalten betreffen. Am effizientesten ist er jedoch, wenn Einschränkungen auf den führenden, also ganz linken, Spalten vorhanden sind. Die genaue Regel lautet: Gleichheitsbedingungen auf führenden Spalten sowie Ungleichheitsbedingungen auf der ersten Spalte ohne Gleichheitsbedingung werden immer verwendet, um den zu durchsuchenden Teil des Indexes einzuschränken. Bedingungen auf weiter rechts liegenden Spalten werden im Index geprüft und sparen daher immer Zugriffe auf die eigentliche Tabelle, verkleinern aber nicht notwendigerweise den Teil des Indexes, der durchsucht werden muss.

Wenn ein B-Tree-Indexscan die Skip-Scan-Optimierung wirksam anwenden kann, nutzt er beim Navigieren durch den Index jede Spaltenbedingung über wiederholte Indexsuchen. Dadurch kann der zu lesende Teil des Indexes kleiner werden, obwohl eine oder mehrere Spalten vor der am wenigsten signifikanten Indexspalte aus dem Abfrageprädikat keine herkömmliche Gleichheitsbedingung besitzen. Skip Scan erzeugt intern eine dynamische Gleichheitsbedingung, die jeden möglichen Wert in einer Indexspalte abdeckt. Das geschieht nur für eine Spalte ohne Gleichheitsbedingung aus dem Abfrageprädikat und nur dann, wenn die erzeugte Bedingung zusammen mit einer späteren Spaltenbedingung aus dem Abfrageprädikat verwendet werden kann.

Bei einem Index auf `(x, y)` und einer Abfragebedingung `WHERE y = 7700` kann ein B-Tree-Indexscan zum Beispiel die Skip-Scan-Optimierung anwenden. Das geschieht im Allgemeinen dann, wenn der Query Planner erwartet, dass wiederholte Suchen der Form `WHERE x = N AND y = 7700` für jeden möglichen Wert von `N`, oder für jeden tatsächlich im Index gespeicherten `x`-Wert, angesichts der verfügbaren Indizes der schnellste Ansatz sind. Dieser Ansatz wird normalerweise nur gewählt, wenn es so wenige verschiedene `x`-Werte gibt, dass der Planner erwartet, den größten Teil des Indexes überspringen zu können, weil die meisten Leaf-Pages unmöglich relevante Tupel enthalten. Gibt es viele verschiedene `x`-Werte, muss der gesamte Index durchsucht werden; in den meisten Fällen wird der Planner dann einen sequenziellen Tabellenscan bevorzugen.

Die Skip-Scan-Optimierung kann auch selektiv während B-Tree-Scans angewendet werden, die zumindest einige nützliche Bedingungen aus dem Abfrageprädikat haben. Bei einem Index auf `(a, b, c)` und einer Abfragebedingung `WHERE a = 5 AND b >= 42 AND c < 77` muss der Index zum Beispiel möglicherweise vom ersten Eintrag mit `a = 5` und `b = 42` bis zum letzten Eintrag mit `a = 5` durchsucht werden. Indexeinträge mit `c >= 77` müssen nie auf Tabellenebene herausgefiltert werden, aber es kann sich lohnen oder auch nicht, sie schon innerhalb des Indexes zu überspringen. Wenn übersprungen wird, startet der Scan eine neue Indexsuche, um sich vom Ende der aktuellen Gruppierung `a = 5` und `b = N` zur nächsten solchen Gruppierung neu zu positionieren.

Ein mehrspaltiger GiST-Index kann mit Abfragebedingungen verwendet werden, die eine beliebige Teilmenge der Indexspalten betreffen. Bedingungen auf zusätzlichen Spalten schränken die vom Index zurückgegebenen Einträge ein, aber die Bedingung auf der ersten Spalte ist am wichtigsten dafür, wie viel vom Index durchsucht werden muss. Ein GiST-Index ist relativ ineffektiv, wenn seine erste Spalte nur wenige unterschiedliche Werte hat, selbst wenn zusätzliche Spalten viele unterschiedliche Werte enthalten.

Ein mehrspaltiger GIN-Index kann mit Abfragebedingungen verwendet werden, die eine beliebige Teilmenge der Indexspalten betreffen. Anders als bei B-Tree oder GiST ist die Effektivität der Indexsuche unabhängig davon, welche Indexspalten die Abfragebedingungen verwenden.

Ein mehrspaltiger BRIN-Index kann mit Abfragebedingungen verwendet werden, die eine beliebige Teilmenge der Indexspalten betreffen. Wie bei GIN und anders als bei B-Tree oder GiST ist die Effektivität der Indexsuche unabhängig davon, welche Indexspalten die Abfragebedingungen verwenden. Der einzige Grund, mehrere BRIN-Indizes statt eines mehrspaltigen BRIN-Indexes auf einer einzelnen Tabelle zu haben, ist ein unterschiedlicher Speicherparameter `pages_per_range`.

Natürlich muss jede Spalte mit Operatoren verwendet werden, die zum Indextyp passen; Klauseln mit anderen Operatoren werden nicht berücksichtigt.

Mehrspaltige Indizes sollten sparsam eingesetzt werden. In den meisten Situationen reicht ein Index auf einer einzelnen Spalte aus und spart Platz und Zeit. Indizes mit mehr als drei Spalten sind wahrscheinlich nur dann hilfreich, wenn die Nutzung der Tabelle extrem schematisch ist. Siehe auch [Abschnitt 11.5](#115-kombinieren-mehrerer-indizes) und [Abschnitt 11.9](#119-indexonlyscans-und-coveringindizes) für eine Diskussion verschiedener Indexkonfigurationen.

## 11.4. Indizes und ORDER BY

Ein Index kann nicht nur die von einer Abfrage zurückzugebenden Zeilen finden, sondern sie unter Umständen auch in einer bestimmten Sortierreihenfolge liefern. Dadurch kann eine `ORDER BY`-Spezifikation ohne separaten Sortierschritt erfüllt werden. Von den derzeit von PostgreSQL unterstützten Indextypen kann nur B-Tree sortierte Ausgabe erzeugen; die anderen Indextypen geben passende Zeilen in einer nicht spezifizierten, implementierungsabhängigen Reihenfolge zurück.

Der Planner zieht in Betracht, eine `ORDER BY`-Spezifikation entweder durch Scannen eines verfügbaren, passenden Indexes zu erfüllen oder die Tabelle in physischer Reihenfolge zu scannen und explizit zu sortieren. Bei einer Abfrage, die einen großen Teil der Tabelle scannen muss, ist explizites Sortieren wahrscheinlich schneller als die Verwendung eines Indexes, weil wegen des sequenziellen Zugriffsmusters weniger Platten-I/O erforderlich ist. Indizes sind nützlicher, wenn nur wenige Zeilen geholt werden müssen. Ein wichtiger Sonderfall ist `ORDER BY` in Kombination mit `LIMIT n`: Ein expliziter Sortiervorgang muss alle Daten verarbeiten, um die ersten `n` Zeilen zu bestimmen; wenn aber ein zur `ORDER BY`-Klausel passender Index vorhanden ist, können die ersten `n` Zeilen direkt abgerufen werden, ohne den Rest zu durchsuchen.

Standardmäßig speichern B-Tree-Indizes ihre Einträge in aufsteigender Reihenfolge mit Nullwerten zuletzt; die Tabellen-TID wird als Tie-Breaker-Spalte zwischen ansonsten gleichen Einträgen behandelt. Das bedeutet, dass ein Vorwärtsscan eines Indexes auf Spalte `x` eine Ausgabe erzeugt, die `ORDER BY x` erfüllt, genauer `ORDER BY x ASC NULLS LAST`. Der Index kann auch rückwärts gescannt werden und erfüllt dann `ORDER BY x DESC`, genauer `ORDER BY x DESC NULLS FIRST`, da `NULLS FIRST` die Voreinstellung für `ORDER BY DESC` ist.

Sie können die Sortierreihenfolge eines B-Tree-Indexes anpassen, indem Sie beim Erzeugen des Indexes die Optionen `ASC`, `DESC`, `NULLS FIRST` und/oder `NULLS LAST` angeben, zum Beispiel:

```sql
CREATE INDEX test2_info_nulls_low ON test2 (info NULLS FIRST);
CREATE INDEX test3_desc_index ON test3 (id DESC NULLS LAST);
```

Ein Index, der in aufsteigender Reihenfolge mit Nullwerten zuerst gespeichert ist, kann je nach Scanrichtung entweder `ORDER BY x ASC NULLS FIRST` oder `ORDER BY x DESC NULLS LAST` erfüllen.

Man könnte sich fragen, warum alle vier Optionen angeboten werden, wenn zwei Optionen zusammen mit der Möglichkeit eines Rückwärtsscans alle Varianten von `ORDER BY` abdecken würden. Bei einspaltigen Indizes sind die Optionen tatsächlich redundant, bei mehrspaltigen Indizes können sie jedoch nützlich sein. Ein zweispaltiger Index auf `(x, y)` kann bei Vorwärtsscan `ORDER BY x, y` erfüllen oder bei Rückwärtsscan `ORDER BY x DESC, y DESC`. Es kann aber sein, dass die Anwendung häufig `ORDER BY x ASC, y DESC` benötigt. Diese Reihenfolge ist mit einem einfachen Index nicht erreichbar, wohl aber mit einem Index, der als `(x ASC, y DESC)` oder `(x DESC, y ASC)` definiert ist.

Indizes mit nicht standardmäßigen Sortierreihenfolgen sind offensichtlich eine recht spezielle Funktion, können aber bei bestimmten Abfragen enorme Beschleunigungen bringen. Ob es sich lohnt, einen solchen Index zu pflegen, hängt davon ab, wie oft Abfragen mit einer speziellen Sortierreihenfolge verwendet werden.

## 11.5. Kombinieren mehrerer Indizes

Ein einzelner Indexscan kann nur Abfrageklauseln verwenden, die die Spalten des Indexes mit Operatoren seiner Operatorklasse nutzen und mit `AND` verbunden sind. Bei einem Index auf `(a, b)` könnte eine Bedingung wie `WHERE a = 5 AND b = 6` den Index verwenden, eine Abfrage wie `WHERE a = 5 OR b = 6` dagegen nicht direkt.

PostgreSQL kann jedoch mehrere Indizes, einschließlich mehrfacher Verwendungen desselben Indexes, kombinieren, um Fälle zu behandeln, die sich nicht durch einzelne Indexscans umsetzen lassen. Das System kann `AND`- und `OR`-Bedingungen über mehrere Indexscans bilden. Eine Abfrage wie `WHERE x = 42 OR x = 47 OR x = 53 OR x = 99` könnte zum Beispiel in vier separate Scans eines Indexes auf `x` zerlegt werden, wobei jeder Scan eine der Abfrageklauseln verwendet. Die Ergebnisse dieser Scans werden dann mit `OR` zusammengeführt. Ein weiteres Beispiel: Wenn separate Indizes auf `x` und `y` vorhanden sind, kann eine mögliche Umsetzung von `WHERE x = 5 AND y = 6` darin bestehen, jeden Index mit der passenden Klausel zu verwenden und die Indexergebnisse anschließend mit `AND` zusammenzuführen, um die Ergebniszeilen zu bestimmen.

Um mehrere Indizes zu kombinieren, scannt das System jeden benötigten Index und erstellt im Speicher eine Bitmap mit den Positionen der Tabellenzeilen, die laut Indexbedingungen passen. Die Bitmaps werden dann je nach Abfrage mit `AND` bzw. `OR` verknüpft. Schließlich werden die tatsächlichen Tabellenzeilen besucht und zurückgegeben. Die Tabellenzeilen werden in physischer Reihenfolge besucht, weil die Bitmap so aufgebaut ist; dadurch geht jede Sortierung der ursprünglichen Indizes verloren, und für eine Abfrage mit `ORDER BY` ist ein separater Sortierschritt nötig. Aus diesem Grund und weil jeder zusätzliche Indexscan zusätzliche Zeit kostet, entscheidet sich der Planner manchmal für einen einfachen Indexscan, obwohl weitere Indizes verfügbar wären, die ebenfalls hätten verwendet werden können.

In allen außer den einfachsten Anwendungen gibt es verschiedene Indexkombinationen, die nützlich sein könnten, und der Datenbankentwickler muss abwägen, welche Indizes bereitgestellt werden sollen. Manchmal sind mehrspaltige Indizes am besten, manchmal ist es besser, separate Indizes anzulegen und sich auf die Indexkombination zu verlassen. Wenn Ihre Arbeitslast zum Beispiel eine Mischung aus Abfragen enthält, die manchmal nur Spalte `x`, manchmal nur Spalte `y` und manchmal beide Spalten betreffen, könnten Sie zwei separate Indizes auf `x` und `y` anlegen und die Kombination für Abfragen mit beiden Spalten verwenden. Sie könnten auch einen mehrspaltigen Index auf `(x, y)` anlegen. Dieser wäre für Abfragen mit beiden Spalten typischerweise effizienter als Indexkombination, wäre aber, wie in [Abschnitt 11.3](#113-mehrspaltige-indizes) besprochen, weniger nützlich für Abfragen, die nur `y` betreffen. Wie nützlich er ist, hängt davon ab, wie wirksam die Skip-Scan-Optimierung des B-Tree-Indexes ist. Wenn `x` höchstens einige Hundert verschiedene Werte hat, kann Skip Scan Suchen nach bestimmten `y`-Werten recht effizient ausführen. Auch eine Kombination aus einem mehrspaltigen Index auf `(x, y)` und einem separaten Index auf `y` kann sinnvoll sein. Für Abfragen nur auf `x` könnte der mehrspaltige Index verwendet werden, wäre aber größer und daher langsamer als ein Index nur auf `x`. Die letzte Alternative wäre, alle drei Indizes zu erzeugen; das ist aber wahrscheinlich nur sinnvoll, wenn die Tabelle viel häufiger durchsucht als aktualisiert wird und alle drei Abfragetypen häufig sind. Wenn einer der Abfragetypen deutlich seltener ist als die anderen, würde man sich vermutlich auf die zwei Indizes beschränken, die am besten zu den häufigen Typen passen.

## 11.6. Eindeutige Indizes

Indizes können auch verwendet werden, um die Eindeutigkeit eines Spaltenwertes oder die Eindeutigkeit der kombinierten Werte mehrerer Spalten zu erzwingen.

```sql
CREATE UNIQUE INDEX name ON table (column [, ...]) [ NULLS [ NOT ] DISTINCT ];
```

Derzeit können nur B-Tree-Indizes als eindeutig deklariert werden.

Wenn ein Index als eindeutig deklariert ist, sind mehrere Tabellenzeilen mit gleichen indizierten Werten nicht zulässig. Standardmäßig gelten Nullwerte in einer eindeutigen Spalte nicht als gleich, sodass mehrere Nullwerte in der Spalte erlaubt sind. Die Option `NULLS NOT DISTINCT` ändert dies und bewirkt, dass der Index Nullwerte als gleich behandelt. Ein mehrspaltiger eindeutiger Index weist nur Fälle zurück, in denen alle indizierten Spalten in mehreren Zeilen gleich sind.

PostgreSQL erzeugt automatisch einen eindeutigen Index, wenn für eine Tabelle eine Unique-Constraint oder ein Primärschlüssel definiert wird. Der Index umfasst die Spalten, aus denen der Primärschlüssel oder die Unique-Constraint besteht, bei Bedarf als mehrspaltiger Index, und ist der Mechanismus, der die Constraint durchsetzt.

> Hinweis: Es ist nicht nötig, Indizes auf eindeutigen Spalten manuell anzulegen; dadurch würde lediglich der automatisch erzeugte Index dupliziert.

## 11.7. Indizes auf Ausdrücken

Eine Indexspalte muss nicht einfach eine Spalte der zugrunde liegenden Tabelle sein, sondern kann eine Funktion oder ein skalarer Ausdruck sein, der aus einer oder mehreren Spalten der Tabelle berechnet wird. Diese Funktion ist nützlich, um schnellen Zugriff auf Tabellen anhand berechneter Ergebnisse zu erhalten.

Eine verbreitete Methode für Vergleiche ohne Beachtung der Groß-/Kleinschreibung ist zum Beispiel die Verwendung der Funktion `lower`:

```sql
SELECT * FROM test1 WHERE lower(col1) = 'value';
```

Diese Abfrage kann einen Index verwenden, wenn einer auf dem Ergebnis der Funktion `lower(col1)` definiert wurde:

```sql
CREATE INDEX test1_lower_col1_idx ON test1 (lower(col1));
```

Wenn dieser Index als `UNIQUE` deklariert würde, würde er die Erzeugung von Zeilen verhindern, deren `col1`-Werte sich nur in der Groß-/Kleinschreibung unterscheiden, ebenso wie Zeilen, deren `col1`-Werte tatsächlich identisch sind. Indizes auf Ausdrücken können daher Constraints durchsetzen, die sich nicht als einfache Unique-Constraints definieren lassen.

Ein weiteres Beispiel: Wenn häufig Abfragen dieser Art ausgeführt werden:

```sql
SELECT * FROM people WHERE (first_name || ' ' || last_name) = 'John Smith';
```

dann kann es sich lohnen, einen Index wie diesen zu erzeugen:

```sql
CREATE INDEX people_names ON people ((first_name || ' ' || last_name));
```

Die Syntax von `CREATE INDEX` verlangt normalerweise Klammern um Indexausdrücke, wie im zweiten Beispiel. Die Klammern können weggelassen werden, wenn der Ausdruck lediglich ein Funktionsaufruf ist, wie im ersten Beispiel.

Indexausdrücke sind relativ teuer in der Pflege, weil die abgeleiteten Ausdrücke bei jedem Einfügen einer Zeile und bei jedem Nicht-HOT-Update berechnet werden müssen. Während einer indizierten Suche werden die Indexausdrücke jedoch nicht neu berechnet, da sie bereits im Index gespeichert sind. In beiden obigen Beispielen sieht das System die Abfrage als einfache Bedingung `WHERE indexedcolumn = 'constant'`; die Suchgeschwindigkeit entspricht daher jeder anderen einfachen Indexabfrage. Indizes auf Ausdrücken sind also nützlich, wenn Abrufgeschwindigkeit wichtiger ist als Einfüge- und Update-Geschwindigkeit.

## 11.8. Partielle Indizes

Ein partieller Index ist ein Index über eine Teilmenge einer Tabelle; die Teilmenge wird durch einen bedingten Ausdruck definiert, das Prädikat des partiellen Indexes. Der Index enthält nur Einträge für diejenigen Tabellenzeilen, die das Prädikat erfüllen. Partielle Indizes sind eine Spezialfunktion, aber es gibt mehrere Situationen, in denen sie nützlich sind.

Ein wichtiger Grund für einen partiellen Index ist das Vermeiden der Indizierung häufiger Werte. Da eine Abfrage nach einem häufigen Wert, also einem Wert, der mehr als wenige Prozent aller Tabellenzeilen ausmacht, den Index ohnehin nicht verwenden wird, gibt es keinen Grund, diese Zeilen im Index zu halten. Dadurch wird der Index kleiner, was diejenigen Abfragen beschleunigt, die ihn verwenden. Außerdem werden viele Tabellenaktualisierungen schneller, weil der Index nicht in jedem Fall aktualisiert werden muss. Beispiel 11.1 zeigt eine mögliche Anwendung dieser Idee.

**Beispiel 11.1. Einen partiellen Index einrichten, um häufige Werte auszuschließen**

Angenommen, Sie speichern Webserver-Zugriffsprotokolle in einer Datenbank. Die meisten Zugriffe stammen aus dem IP-Adressbereich Ihrer Organisation, einige aber von außerhalb, etwa von Mitarbeitern mit Einwahlverbindungen. Wenn Ihre Suchen nach IP-Adressen vor allem externe Zugriffe betreffen, müssen Sie den IP-Bereich des Organisationssubnetzes wahrscheinlich nicht indizieren.

Nehmen wir eine Tabelle wie diese an:

```sql
CREATE TABLE access_log (
    url varchar,
    client_ip inet,
    ...
);
```

Um einen partiellen Index zu erzeugen, der zu unserem Beispiel passt, verwenden Sie einen Befehl wie diesen:

```sql
CREATE INDEX access_log_client_ip_ix ON access_log (client_ip)
WHERE NOT (client_ip > inet '192.168.100.0' AND
           client_ip < inet '192.168.100.255');
```

Eine typische Abfrage, die diesen Index verwenden kann, wäre:

```sql
SELECT *
FROM access_log
WHERE url = '/index.html' AND client_ip = inet '212.78.10.32';
```

Hier wird die IP-Adresse der Abfrage vom partiellen Index abgedeckt. Die folgende Abfrage kann den partiellen Index nicht verwenden, weil sie eine IP-Adresse verwendet, die aus dem Index ausgeschlossen wurde:

```sql
SELECT *
FROM access_log
WHERE url = '/index.html' AND client_ip = inet '192.168.100.23';
```

Beachten Sie, dass diese Art partieller Index voraussetzt, dass die häufigen Werte im Voraus bestimmt sind. Solche partiellen Indizes eignen sich daher am besten für Datenverteilungen, die sich nicht ändern. Sie können gelegentlich neu erstellt werden, um sich an neue Datenverteilungen anzupassen, aber das verursacht Wartungsaufwand.

Eine andere mögliche Verwendung für einen partiellen Index besteht darin, Werte aus dem Index auszuschließen, die für die typische Abfragelast uninteressant sind; dies zeigt Beispiel 11.2. Das führt zu denselben Vorteilen wie oben, verhindert aber, dass die „uninteressanten“ Werte über diesen Index erreichbar sind, selbst wenn ein Indexscan in diesem Fall lohnend wäre. Das Einrichten partieller Indizes für ein solches Szenario erfordert offensichtlich viel Sorgfalt und Experimentieren.

**Beispiel 11.2. Einen partiellen Index einrichten, um uninteressante Werte auszuschließen**

Wenn Sie eine Tabelle haben, die sowohl berechnete als auch unberechnete Bestellungen enthält, wobei die unberechneten Bestellungen nur einen kleinen Teil der gesamten Tabelle ausmachen, aber die am häufigsten abgefragten Zeilen sind, können Sie die Leistung verbessern, indem Sie einen Index nur auf den unberechneten Zeilen erzeugen. Der Befehl dafür sähe so aus:

```sql
CREATE INDEX orders_unbilled_index ON orders (order_nr)
    WHERE billed is not true;
```

Eine mögliche Abfrage, die diesen Index verwendet, wäre:

```sql
SELECT * FROM orders WHERE billed is not true AND order_nr < 10000;
```

Der Index kann aber auch in Abfragen verwendet werden, die `order_nr` gar nicht betreffen, zum Beispiel:

```sql
SELECT * FROM orders WHERE billed is not true AND amount > 5000.00;
```

Das ist nicht so effizient, wie es ein partieller Index auf der Spalte `amount` wäre, weil das System den gesamten Index scannen muss. Wenn es jedoch relativ wenige unberechnete Bestellungen gibt, kann die Verwendung dieses partiellen Indexes allein zum Auffinden der unberechneten Bestellungen ein Gewinn sein.

Beachten Sie, dass diese Abfrage den Index nicht verwenden kann:

```sql
SELECT * FROM orders WHERE order_nr = 3501;
```

Die Bestellung 3501 könnte zu den berechneten oder den unberechneten Bestellungen gehören.

Beispiel 11.2 zeigt außerdem, dass die indizierte Spalte und die im Prädikat verwendete Spalte nicht übereinstimmen müssen. PostgreSQL unterstützt partielle Indizes mit beliebigen Prädikaten, solange nur Spalten der indizierten Tabelle beteiligt sind. Denken Sie jedoch daran, dass das Prädikat zu den Bedingungen der Abfragen passen muss, die vom Index profitieren sollen. Genau genommen kann ein partieller Index in einer Abfrage nur dann verwendet werden, wenn das System erkennen kann, dass die `WHERE`-Bedingung der Abfrage mathematisch das Prädikat des Indexes impliziert. PostgreSQL besitzt keinen ausgefeilten Theorembeweiser, der mathematisch äquivalente Ausdrücke in unterschiedlichen Schreibweisen erkennt. Ein solcher allgemeiner Theorembeweiser wäre nicht nur extrem schwer zu erstellen, sondern wahrscheinlich auch zu langsam für praktischen Nutzen. Das System kann einfache Implikationen von Ungleichungen erkennen, zum Beispiel dass `x < 1` `x < 2` impliziert. Ansonsten muss die Prädikatbedingung genau einem Teil der `WHERE`-Bedingung der Abfrage entsprechen, sonst wird der Index nicht als nutzbar erkannt. Der Abgleich findet zur Planungszeit statt, nicht zur Laufzeit. Daher funktionieren parametrisierte Abfrageklauseln nicht mit einem partiellen Index. Eine vorbereitete Abfrage mit einem Parameter könnte zum Beispiel `x < ?` angeben, was niemals für alle möglichen Parameterwerte `x < 2` impliziert.

Eine dritte mögliche Verwendung partieller Indizes verlangt gar nicht, dass der Index in Abfragen verwendet wird. Die Idee besteht darin, einen eindeutigen Index über eine Teilmenge einer Tabelle zu erzeugen, wie in Beispiel 11.3. Dadurch wird Eindeutigkeit unter den Zeilen erzwungen, die das Indexprädikat erfüllen, ohne diejenigen einzuschränken, die es nicht erfüllen.

**Beispiel 11.3. Einen partiellen eindeutigen Index einrichten**

Angenommen, es gibt eine Tabelle, die Testergebnisse beschreibt. Es soll sichergestellt werden, dass es nur einen „erfolgreichen“ Eintrag für eine gegebene Kombination aus Subjekt und Ziel gibt, während es beliebig viele „nicht erfolgreiche“ Einträge geben darf. Eine Möglichkeit ist:

```sql
CREATE TABLE tests (
    subject text,
    target text,
    success boolean,
    ...
);

CREATE UNIQUE INDEX tests_success_constraint ON tests (subject, target)
    WHERE success;
```

Das ist ein besonders effizienter Ansatz, wenn es wenige erfolgreiche Tests und viele nicht erfolgreiche gibt. Es ist auch möglich, nur einen Nullwert in einer Spalte zu erlauben, indem ein eindeutiger partieller Index mit einer `IS NULL`-Einschränkung erzeugt wird.

Schließlich kann ein partieller Index auch verwendet werden, um die Planentscheidungen des Systems zu übersteuern. Datenbestände mit ungewöhnlichen Verteilungen können das System dazu veranlassen, einen Index zu verwenden, obwohl es das eigentlich nicht sollte. In diesem Fall kann der Index so eingerichtet werden, dass er für die problematische Abfrage nicht verfügbar ist. Normalerweise trifft PostgreSQL vernünftige Entscheidungen über die Indexverwendung, zum Beispiel vermeidet es Indizes beim Abruf häufiger Werte; das frühere Beispiel spart also tatsächlich nur Indexgröße und ist nicht erforderlich, um Indexverwendung zu vermeiden. Grob falsche Planentscheidungen sind ein Anlass für einen Fehlerbericht.

Behalten Sie im Blick, dass das Einrichten eines partiellen Indexes bedeutet, dass Sie mindestens so viel wissen wie der Query Planner, insbesondere wann ein Index lohnend sein könnte. Dieses Wissen erfordert Erfahrung und Verständnis dafür, wie Indizes in PostgreSQL funktionieren. In den meisten Fällen ist der Vorteil eines partiellen Indexes gegenüber einem regulären Index minimal. Es gibt auch Fälle, in denen sie deutlich kontraproduktiv sind, wie Beispiel 11.4 zeigt.

**Beispiel 11.4. Partielle Indizes nicht als Ersatz für Partitionierung verwenden**

Man könnte versucht sein, eine große Menge nicht überlappender partieller Indizes zu erzeugen, zum Beispiel:

```sql
CREATE INDEX mytable_cat_1 ON mytable (data) WHERE category = 1;
CREATE INDEX mytable_cat_2 ON mytable (data) WHERE category = 2;
CREATE INDEX mytable_cat_3 ON mytable (data) WHERE category = 3;
...
CREATE INDEX mytable_cat_N ON mytable (data) WHERE category = N;
```

Das ist eine schlechte Idee. Fast sicher sind Sie mit einem einzelnen nicht partiellen Index besser bedient, etwa:

```sql
CREATE INDEX mytable_cat_data ON mytable (category, data);
```

Setzen Sie die Spalte `category` aus den in [Abschnitt 11.3](#113-mehrspaltige-indizes) beschriebenen Gründen an den Anfang. Eine Suche in diesem größeren Index muss vielleicht ein paar zusätzliche Baumebenen hinabsteigen, aber das ist mit hoher Wahrscheinlichkeit günstiger als der Planungsaufwand, der nötig ist, um den passenden partiellen Index auszuwählen. Der Kern des Problems ist, dass das System die Beziehung zwischen den partiellen Indizes nicht versteht und jeden einzelnen mühsam testet, um zu sehen, ob er für die aktuelle Abfrage anwendbar ist.

Wenn Ihre Tabelle so groß ist, dass ein einzelner Index tatsächlich eine schlechte Idee ist, sollten Sie stattdessen Partitionierung in Betracht ziehen; siehe [Abschnitt 5.12](05_Datendefinition.md#512-tabellenpartitionierung). Bei diesem Mechanismus versteht das System, dass Tabellen und Indizes nicht überlappen, sodass deutlich bessere Leistung möglich ist.

Weitere Informationen über partielle Indizes finden Sie in [ston89b], [olson93] und [seshadri95].

## 11.9. Index-Only-Scans und Covering-Indizes

Alle Indizes in PostgreSQL sind sekundäre Indizes. Das bedeutet, dass jeder Index getrennt vom Hauptdatenbereich der Tabelle gespeichert wird, der in der PostgreSQL-Terminologie Heap der Tabelle heißt. Bei einem gewöhnlichen Indexscan muss daher für jede abgerufene Zeile sowohl aus dem Index als auch aus dem Heap gelesen werden. Außerdem liegen die Indexeinträge, die zu einer indizierbaren `WHERE`-Bedingung passen, gewöhnlich nahe beieinander im Index, die referenzierten Tabellenzeilen können aber überall im Heap liegen. Der Heap-Zugriffsteil eines Indexscans umfasst daher viele zufällige Zugriffe in den Heap, was insbesondere auf traditionellen rotierenden Medien langsam sein kann. Wie in [Abschnitt 11.5](#115-kombinieren-mehrerer-indizes) beschrieben, versuchen Bitmap-Scans diese Kosten zu mindern, indem sie die Heap-Zugriffe in sortierter Reihenfolge ausführen; das hilft aber nur bis zu einem gewissen Punkt.

Um dieses Leistungsproblem zu lösen, unterstützt PostgreSQL Index-Only-Scans, die Abfragen allein aus einem Index beantworten können, ohne auf den Heap zuzugreifen. Die Grundidee ist, Werte direkt aus jedem Indexeintrag zurückzugeben, statt den zugehörigen Heap-Eintrag zu konsultieren. Dafür gibt es zwei grundlegende Einschränkungen:

1. Der Indextyp muss Index-Only-Scans unterstützen. B-Tree-Indizes tun dies immer. GiST- und SP-GiST-Indizes unterstützen Index-Only-Scans für manche Operatorklassen, aber nicht für andere. Andere Indextypen unterstützen sie nicht. Die zugrunde liegende Anforderung ist, dass der Index den ursprünglichen Datenwert für jeden Indexeintrag physisch speichern oder rekonstruieren können muss. GIN-Indizes können Index-Only-Scans zum Beispiel nicht unterstützen, weil jeder Indexeintrag typischerweise nur einen Teil des ursprünglichen Datenwerts enthält.

2. Die Abfrage darf nur Spalten referenzieren, die im Index gespeichert sind. Bei einem Index auf den Spalten `x` und `y` einer Tabelle, die außerdem eine Spalte `z` hat, könnten diese Abfragen Index-Only-Scans verwenden:

```sql
SELECT x, y FROM tab WHERE x = 'key';
SELECT x FROM tab WHERE x = 'key' AND y < 42;
```

Diese Abfragen könnten es dagegen nicht:

```sql
SELECT x, z FROM tab WHERE x = 'key';
SELECT x FROM tab WHERE x = 'key' AND z < 42;
```

Ausdrucksindizes und partielle Indizes verkomplizieren diese Regel, wie unten besprochen.

Wenn diese beiden grundlegenden Anforderungen erfüllt sind, sind alle von der Abfrage benötigten Datenwerte im Index verfügbar, sodass ein Index-Only-Scan physisch möglich ist. Es gibt aber eine zusätzliche Anforderung für jeden Tabellenscan in PostgreSQL: Er muss prüfen, ob jede abgerufene Zeile für den MVCC-Snapshot der Abfrage „sichtbar“ ist, wie in [Kapitel 13](13_Nebenläufigkeitskontrolle.md) beschrieben. Sichtbarkeitsinformationen werden nicht in Indexeinträgen gespeichert, sondern nur in Heap-Einträgen. Auf den ersten Blick scheint daher jeder Zeilenabruf ohnehin einen Heap-Zugriff zu benötigen. Das ist tatsächlich der Fall, wenn die Tabellenzeile kürzlich geändert wurde. Für selten geänderte Daten gibt es jedoch einen Ausweg. PostgreSQL verfolgt für jede Seite im Heap einer Tabelle, ob alle dort gespeicherten Zeilen alt genug sind, um für alle aktuellen und zukünftigen Transaktionen sichtbar zu sein. Diese Information wird in einem Bit in der Visibility Map der Tabelle gespeichert.

Ein Index-Only-Scan prüft nach dem Finden eines Kandidaten-Indexeintrags das Visibility-Map-Bit der entsprechenden Heap-Seite. Wenn es gesetzt ist, gilt die Zeile als sichtbar und die Daten können ohne weitere Arbeit zurückgegeben werden. Wenn es nicht gesetzt ist, muss der Heap-Eintrag besucht werden, um die Sichtbarkeit festzustellen; dann entsteht kein Leistungsvorteil gegenüber einem normalen Indexscan. Selbst im Erfolgsfall tauscht dieser Ansatz Visibility-Map-Zugriffe gegen Heap-Zugriffe. Da die Visibility Map jedoch um Größenordnungen kleiner ist als der Heap, den sie beschreibt, ist dafür deutlich weniger physisches I/O nötig. In den meisten Situationen bleibt die Visibility Map ständig im Speicher gecacht.

Kurz gesagt: Ein Index-Only-Scan ist zwar möglich, wenn die beiden grundlegenden Anforderungen erfüllt sind, er bringt aber nur dann einen Gewinn, wenn bei einem erheblichen Anteil der Heap-Seiten der Tabelle die All-Visible-Bits gesetzt sind. Tabellen, in denen ein großer Teil der Zeilen unverändert bleibt, sind jedoch häufig genug, um diese Scan-Art in der Praxis sehr nützlich zu machen.

Um Index-Only-Scans wirksam zu nutzen, kann ein Covering-Index erstellt werden, also ein Index, der speziell darauf ausgelegt ist, die von einer häufig ausgeführten Abfrage benötigten Spalten einzuschließen. Da Abfragen typischerweise mehr Spalten abrufen als nur diejenigen, nach denen sie suchen, erlaubt PostgreSQL Indizes, in denen einige Spalten nur „Payload“ sind und nicht zum Suchschlüssel gehören. Dazu wird eine `INCLUDE`-Klausel mit den zusätzlichen Spalten angegeben. Wenn Sie zum Beispiel häufig Abfragen wie diese ausführen:

```sql
SELECT y FROM tab WHERE x = 'key';
```

wäre der traditionelle Ansatz, zur Beschleunigung einen Index nur auf `x` anzulegen. Ein so definierter Index

```sql
CREATE INDEX tab_x_y ON tab(x) INCLUDE (y);
```

könnte diese Abfragen jedoch als Index-Only-Scans behandeln, weil `y` aus dem Index geholt werden kann, ohne den Heap zu besuchen.

Da Spalte `y` nicht Teil des Suchschlüssels des Indexes ist, muss sie keinen Datentyp haben, den der Index verarbeiten kann; sie wird lediglich im Index gespeichert und von der Indexmaschinerie nicht interpretiert. Ist der Index außerdem eindeutig, also

```sql
CREATE UNIQUE INDEX tab_x_y ON tab(x) INCLUDE (y);
```

dann gilt die Eindeutigkeitsbedingung nur für Spalte `x`, nicht für die Kombination aus `x` und `y`. Eine `INCLUDE`-Klausel kann auch in `UNIQUE`- und `PRIMARY KEY`-Constraints geschrieben werden; das bietet eine alternative Syntax zum Einrichten eines solchen Indexes.

Es ist klug, beim Hinzufügen von Nicht-Schlüssel-Payload-Spalten zu einem Index konservativ zu sein, besonders bei breiten Spalten. Wenn ein Indextupel die für den Indextyp zulässige Maximalgröße überschreitet, schlägt das Einfügen von Daten fehl. In jedem Fall duplizieren Nicht-Schlüsselspalten Daten aus der Indextabelle und blähen den Index auf, was Suchen verlangsamen kann. Denken Sie auch daran, dass Payload-Spalten in einem Index wenig bringen, wenn sich die Tabelle nicht langsam genug ändert, damit ein Index-Only-Scan den Heap wahrscheinlich nicht besuchen muss. Wenn das Heap-Tupel ohnehin besucht werden muss, kostet es nichts zusätzlich, den Spaltenwert dort zu holen. Weitere Einschränkungen sind, dass Ausdrücke derzeit nicht als eingeschlossene Spalten unterstützt werden und dass nur B-Tree-, GiST- und SP-GiST-Indizes eingeschlossene Spalten unterstützen.

Bevor PostgreSQL die `INCLUDE`-Funktion hatte, wurden Covering-Indizes manchmal dadurch erzeugt, dass die Payload-Spalten als gewöhnliche Indexspalten geschrieben wurden:

```sql
CREATE INDEX tab_x_y ON tab(x, y);
```

Das funktioniert gut, solange die zusätzlichen Spalten nachlaufende Spalten sind; sie zu führenden Spalten zu machen, ist aus den in [Abschnitt 11.3](#113-mehrspaltige-indizes) erläuterten Gründen unklug. Diese Methode unterstützt jedoch nicht den Fall, dass der Index Eindeutigkeit nur für die Schlüsselspalten erzwingen soll.

Suffix-Truncation entfernt Nicht-Schlüsselspalten immer aus den oberen B-Tree-Ebenen. Als Payload-Spalten werden sie nie verwendet, um Indexscans zu steuern. Der Kürzungsprozess entfernt außerdem eine oder mehrere nachlaufende Schlüsselspalten, wenn der verbleibende Präfix der Schlüsselspalten ausreicht, um Tupel auf der untersten B-Tree-Ebene zu beschreiben. In der Praxis vermeiden Covering-Indizes ohne `INCLUDE`-Klausel häufig, Spalten, die effektiv Payload sind, in den oberen Ebenen zu speichern. Werden Payload-Spalten jedoch ausdrücklich als Nicht-Schlüsselspalten definiert, bleiben Tupel in den oberen Ebenen zuverlässig klein.

Grundsätzlich können Index-Only-Scans mit Ausdrucksindizes verwendet werden. Bei einem Index auf `f(x)`, wobei `x` eine Tabellenspalte ist, sollte es zum Beispiel möglich sein,

```sql
SELECT f(x) FROM tab WHERE f(x) < 1;
```

als Index-Only-Scan auszuführen. Das ist besonders attraktiv, wenn `f()` teuer zu berechnen ist. Der PostgreSQL-Planner ist in solchen Fällen derzeit jedoch nicht sehr schlau. Er betrachtet eine Abfrage nur dann als potenziell per Index-Only-Scan ausführbar, wenn alle von der Abfrage benötigten Spalten im Index verfügbar sind. In diesem Beispiel wird `x` nur im Kontext `f(x)` benötigt, aber der Planner erkennt das nicht und kommt zu dem Schluss, dass ein Index-Only-Scan nicht möglich ist. Wenn sich ein Index-Only-Scan ausreichend lohnt, kann dies umgangen werden, indem `x` als eingeschlossene Spalte hinzugefügt wird, zum Beispiel:

```sql
CREATE INDEX tab_f_x ON tab (f(x)) INCLUDE (x);
```

Ein zusätzlicher Vorbehalt, wenn das Ziel darin besteht, eine Neuberechnung von `f(x)` zu vermeiden: Der Planner ordnet Verwendungen von `f(x)`, die nicht in indizierbaren `WHERE`-Klauseln stehen, nicht unbedingt der Indexspalte zu. In einfachen Abfragen wie der oben gezeigten macht er das in der Regel richtig, nicht aber in Abfragen mit Joins. Diese Defizite könnten in zukünftigen PostgreSQL-Versionen behoben werden.

Partielle Indizes haben ebenfalls interessante Wechselwirkungen mit Index-Only-Scans. Betrachten Sie den partiellen Index aus Beispiel 11.3:

```sql
CREATE UNIQUE INDEX tests_success_constraint ON tests (subject, target)
    WHERE success;
```

Grundsätzlich könnte ein Index-Only-Scan auf diesem Index eine Abfrage wie diese erfüllen:

```sql
SELECT target FROM tests WHERE subject = 'some-subject' AND success;
```

Es gibt aber ein Problem: Die `WHERE`-Klausel referenziert `success`, das nicht als Ergebnisspalte im Index verfügbar ist. Dennoch ist ein Index-Only-Scan möglich, weil der Plan diesen Teil der `WHERE`-Klausel zur Laufzeit nicht erneut prüfen muss: Alle im Index gefundenen Einträge haben notwendigerweise `success = true`, sodass dies im Plan nicht ausdrücklich geprüft werden muss. PostgreSQL-Versionen ab 9.6 erkennen solche Fälle und erlauben die Erzeugung von Index-Only-Scans; ältere Versionen tun dies nicht.

## 11.10. Operatorklassen und Operatorfamilien

Eine Indexdefinition kann für jede Spalte eines Indexes eine Operatorklasse angeben.

```sql
CREATE INDEX name ON table (column opclass [ ( opclass_options ) ] [sort options] [, ...]);
```

Die Operatorklasse identifiziert die Operatoren, die der Index für diese Spalte verwenden soll. Ein B-Tree-Index auf dem Typ `int4` würde zum Beispiel die Klasse `int4_ops` verwenden; diese Operatorklasse enthält Vergleichsfunktionen für Werte des Typs `int4`. In der Praxis reicht die Standardoperatorklasse für den Datentyp der Spalte meistens aus. Der wichtigste Grund für Operatorklassen ist, dass es für manche Datentypen mehr als ein sinnvolles Indexverhalten geben kann. Ein Datentyp für komplexe Zahlen könnte zum Beispiel entweder nach Betrag oder nach Realteil sortiert werden. Das ließe sich umsetzen, indem zwei Operatorklassen für den Datentyp definiert und beim Erzeugen eines Indexes die passende Klasse ausgewählt wird. Die Operatorklasse bestimmt die grundlegende Sortierreihenfolge, die anschließend durch Sortieroptionen wie `COLLATE`, `ASC`/`DESC` und/oder `NULLS FIRST`/`NULLS LAST` verändert werden kann.

Neben den Standardklassen gibt es einige eingebaute Operatorklassen:

- Die Operatorklassen `text_pattern_ops`, `varchar_pattern_ops` und `bpchar_pattern_ops` unterstützen B-Tree-Indizes auf den Typen `text`, `varchar` bzw. `char`. Der Unterschied zu den Standardoperatorklassen besteht darin, dass die Werte strikt Zeichen für Zeichen verglichen werden und nicht nach den locale-spezifischen Kollationsregeln. Dadurch eignen sich diese Operatorklassen für Abfragen mit Pattern-Matching-Ausdrücken (`LIKE` oder POSIX-Regular-Expressions), wenn die Datenbank nicht die Standard-Locale `C` verwendet. Eine `varchar`-Spalte könnten Sie zum Beispiel so indizieren:

```sql
CREATE INDEX test_index ON test_table (col varchar_pattern_ops);
```

Beachten Sie, dass Sie zusätzlich einen Index mit der Standardoperatorklasse erzeugen sollten, wenn Abfragen mit gewöhnlichen Vergleichen wie `<`, `<=`, `>` oder `>=` einen Index verwenden sollen. Solche Abfragen können die `xxx_pattern_ops`-Operatorklassen nicht verwenden. Gewöhnliche Gleichheitsvergleiche können diese Operatorklassen allerdings verwenden. Es ist möglich, mehrere Indizes auf derselben Spalte mit unterschiedlichen Operatorklassen zu erzeugen. Wenn Sie die Locale `C` verwenden, benötigen Sie die `xxx_pattern_ops`-Operatorklassen nicht, weil ein Index mit der Standardoperatorklasse in der Locale `C` für Pattern-Matching-Abfragen nutzbar ist.

Die folgende Abfrage zeigt alle definierten Operatorklassen:

```sql
SELECT am.amname AS index_method,
       opc.opcname AS opclass_name,
       opc.opcintype::regtype AS indexed_type,
       opc.opcdefault AS is_default
    FROM pg_am am, pg_opclass opc
    WHERE opc.opcmethod = am.oid
    ORDER BY index_method, opclass_name;
```

Eine Operatorklasse ist tatsächlich nur eine Teilmenge einer größeren Struktur, die Operatorfamilie heißt. Wenn mehrere Datentypen ähnliches Verhalten haben, ist es häufig nützlich, Operatoren über Datentypgrenzen hinweg zu definieren und sie mit Indizes arbeiten zu lassen. Dazu müssen die Operatorklassen der einzelnen Typen in derselben Operatorfamilie gruppiert werden. Die Cross-Type-Operatoren sind Mitglieder der Familie, aber keiner einzelnen Klasse innerhalb der Familie zugeordnet.

Diese erweiterte Version der vorherigen Abfrage zeigt die Operatorfamilie, zu der jede Operatorklasse gehört:

```sql
SELECT am.amname AS index_method,
       opc.opcname AS opclass_name,
       opf.opfname AS opfamily_name,
       opc.opcintype::regtype AS indexed_type,
       opc.opcdefault AS is_default
    FROM pg_am am, pg_opclass opc, pg_opfamily opf
    WHERE opc.opcmethod = am.oid AND
          opc.opcfamily = opf.oid
    ORDER BY index_method, opclass_name;
```

Diese Abfrage zeigt alle definierten Operatorfamilien und alle Operatoren, die in jeder Familie enthalten sind:

```sql
SELECT am.amname AS index_method,
       opf.opfname AS opfamily_name,
       amop.amopopr::regoperator AS opfamily_operator
    FROM pg_am am, pg_opfamily opf, pg_amop amop
    WHERE opf.opfmethod = am.oid AND
          amop.amopfamily = opf.oid
    ORDER BY index_method, opfamily_name, opfamily_operator;
```

> Tipp: `psql` besitzt die Befehle `\dAc`, `\dAf` und `\dAo`, die etwas ausgefeiltere Versionen dieser Abfragen bereitstellen.

## 11.11. Indizes und Kollationen

Ein Index kann pro Indexspalte nur eine Kollation unterstützen. Wenn mehrere Kollationen relevant sind, können mehrere Indizes nötig sein.

Betrachten Sie diese Anweisungen:

```sql
CREATE TABLE test1c (
    id integer,
    content varchar COLLATE "x"
);

CREATE INDEX test1c_content_index ON test1c (content);
```

Der Index verwendet automatisch die Kollation der zugrunde liegenden Spalte. Eine Abfrage dieser Form

```sql
SELECT * FROM test1c WHERE content > constant;
```

könnte den Index verwenden, weil der Vergleich standardmäßig die Kollation der Spalte verwendet. Dieser Index kann jedoch keine Abfragen beschleunigen, die eine andere Kollation betreffen. Wenn also etwa Abfragen dieser Form ebenfalls relevant sind:

```sql
SELECT * FROM test1c WHERE content > constant COLLATE "y";
```

dann könnte ein zusätzlicher Index erzeugt werden, der die Kollation `"y"` unterstützt:

```sql
CREATE INDEX test1c_content_y_index ON test1c (content COLLATE "y");
```

## 11.12. Indexverwendung untersuchen

Obwohl Indizes in PostgreSQL keine Wartung oder Feinabstimmung benötigen, ist es dennoch wichtig zu prüfen, welche Indizes von der realen Abfragelast tatsächlich verwendet werden. Die Indexverwendung für eine einzelne Abfrage wird mit dem Befehl `EXPLAIN` untersucht; seine Anwendung dafür wird in [Abschnitt 14.1](14_Hinweise_zur_Performance.md#141-explain-verwenden) gezeigt. Es ist auch möglich, Gesamtstatistiken zur Indexverwendung in einem laufenden Server zu sammeln, wie in [Abschnitt 27.2](27_Datenbankaktivität_überwachen.md#272-das-kumulative-statistiksystem) beschrieben.

Es ist schwierig, ein allgemeines Verfahren dafür anzugeben, welche Indizes anzulegen sind. In den Beispielen der vorangegangenen Abschnitte wurden einige typische Fälle gezeigt. Oft ist ein gutes Maß an Experimentieren nötig. Der Rest dieses Abschnitts gibt einige Hinweise dazu:

- Führen Sie immer zuerst `ANALYZE` aus. Dieser Befehl sammelt Statistiken über die Verteilung der Werte in der Tabelle. Diese Informationen werden benötigt, um die Zahl der von einer Abfrage zurückgegebenen Zeilen zu schätzen; der Planner braucht diese Schätzung, um jedem möglichen Abfrageplan realistische Kosten zuzuweisen. Ohne echte Statistiken werden Standardwerte angenommen, die fast sicher ungenau sind. Die Untersuchung der Indexverwendung einer Anwendung, ohne zuvor `ANALYZE` ausgeführt zu haben, ist daher aussichtslos. Weitere Informationen finden Sie in Abschnitt 24.1.3 und Abschnitt 24.1.6.

- Verwenden Sie echte Daten für Experimente. Testdaten zum Einrichten von Indizes sagen Ihnen, welche Indizes Sie für die Testdaten brauchen, aber nicht mehr.

  Besonders fatal ist die Verwendung sehr kleiner Testdatenbestände. Während die Auswahl von 1000 aus 100000 Zeilen ein Kandidat für einen Index sein könnte, wird die Auswahl von 1 aus 100 Zeilen kaum einer sein, weil diese 100 Zeilen vermutlich auf eine einzelne Plattenseite passen und kein Plan besser sein kann als das sequenzielle Lesen dieser einen Seite.

  Seien Sie auch vorsichtig beim Erfinden von Testdaten, was oft unvermeidlich ist, wenn die Anwendung noch nicht in Produktion ist. Werte, die sehr ähnlich, völlig zufällig oder in sortierter Reihenfolge eingefügt sind, verzerren die Statistiken weg von der Verteilung, die echte Daten hätten.

- Wenn Indizes nicht verwendet werden, kann es zum Testen nützlich sein, ihre Verwendung zu erzwingen. Es gibt Laufzeitparameter, die verschiedene Plantypen abschalten können; siehe Abschnitt 19.7.1. Das Abschalten sequenzieller Scans (`enable_seqscan`) und Nested-Loop-Joins (`enable_nestloop`), der grundlegendsten Pläne, zwingt das System zum Beispiel dazu, einen anderen Plan zu verwenden. Wenn das System dennoch einen sequenziellen Scan oder Nested-Loop-Join wählt, gibt es wahrscheinlich einen grundlegenderen Grund dafür, dass der Index nicht verwendet wird, etwa dass die Abfragebedingung nicht zum Index passt. Welche Art von Abfrage welche Art von Index verwenden kann, wurde in den vorangegangenen Abschnitten erklärt.

- Wenn das Erzwingen der Indexverwendung dazu führt, dass der Index verwendet wird, gibt es zwei Möglichkeiten: Entweder hat das System recht und die Verwendung des Indexes ist tatsächlich nicht angemessen, oder die Kostenschätzungen der Abfragepläne spiegeln die Realität nicht wider. Sie sollten Ihre Abfrage daher mit und ohne Indizes zeitlich messen. Der Befehl `EXPLAIN ANALYZE` kann dabei nützlich sein.

- Wenn sich herausstellt, dass die Kostenschätzungen falsch sind, gibt es wiederum zwei Möglichkeiten. Die Gesamtkosten werden aus den Kosten pro Zeile jedes Planknotens multipliziert mit der Selektivitätsschätzung des Planknotens berechnet. Die geschätzten Kosten der Planknoten können über Laufzeitparameter angepasst werden, die in Abschnitt 19.7.2 beschrieben sind. Eine ungenaue Selektivitätsschätzung ist auf unzureichende Statistiken zurückzuführen. Möglicherweise lässt sich dies durch Anpassen der Parameter für die Statistikermittlung verbessern; siehe `ALTER TABLE`.

  Wenn es nicht gelingt, die Kosten angemessener einzustellen, müssen Sie eventuell dazu übergehen, die Indexverwendung ausdrücklich zu erzwingen. Sie können auch die PostgreSQL-Entwickler kontaktieren, damit sie das Problem untersuchen.
