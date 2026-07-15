# 51. Überblick über die Interna von PostgreSQL

Dieses Kapitel entstand ursprünglich als Teil von [sim98], der Masterarbeit von Stefan Simkovics, erstellt an der Technischen Universität Wien unter der Betreuung von O.Univ.Prof.Dr. Georg Gottlob und Univ.Ass. Mag. Katrin Seyr.

Dieses Kapitel gibt einen Überblick über die interne Struktur des PostgreSQL-Backends. Nach dem Lesen der folgenden Abschnitte sollten Sie eine Vorstellung davon haben, wie eine Abfrage verarbeitet wird. Das Kapitel soll helfen, die allgemeine Reihenfolge der Operationen zu verstehen, die im Backend stattfinden, vom Eingang einer Abfrage bis zu dem Zeitpunkt, an dem die Ergebnisse an den Client zurückgegeben werden.

## 51.1. Der Weg einer Abfrage

Hier geben wir einen kurzen Überblick über die Phasen, die eine Abfrage durchlaufen muss, um ein Ergebnis zu liefern.

1. Eine Verbindung von einem Anwendungsprogramm zum PostgreSQL-Server muss hergestellt werden. Das Anwendungsprogramm übermittelt eine Abfrage an den Server und wartet darauf, die vom Server zurückgesendeten Ergebnisse zu empfangen.

2. Die Parser-Phase prüft die vom Anwendungsprogramm übermittelte Abfrage auf korrekte Syntax und erstellt einen Abfragebaum.

3. Das Rewrite-System nimmt den von der Parser-Phase erzeugten Abfragebaum und sucht nach Regeln (die in den Systemkatalogen gespeichert sind), die auf den Abfragebaum angewendet werden sollen. Es führt die in den Regelkörpern angegebenen Transformationen aus.

   Eine Anwendung des Rewrite-Systems ist die Realisierung von Views. Immer wenn eine Abfrage gegen eine View (also eine virtuelle Tabelle) gestellt wird, schreibt das Rewrite-System die Abfrage des Benutzers in eine Abfrage um, die stattdessen auf die in der View-Definition angegebenen Basistabellen zugreift.

4. Der Planner/Optimizer nimmt den (umgeschriebenen) Abfragebaum und erzeugt einen Abfrageplan, der die Eingabe für den Executor bildet.

   Dazu erzeugt er zunächst alle möglichen Pfade, die zum gleichen Ergebnis führen. Wenn es beispielsweise einen Index auf einer zu scannenden Relation gibt, existieren zwei Pfade für den Scan. Eine Möglichkeit ist ein einfacher sequenzieller Scan, die andere ist die Verwendung des Indexes. Anschließend werden die Kosten für die Ausführung jedes Pfads geschätzt, und der billigste Pfad wird ausgewählt. Der billigste Pfad wird zu einem vollständigen Plan erweitert, den der Executor verwenden kann.

5. Der Executor durchläuft den Planbaum rekursiv und ruft Zeilen auf die im Plan dargestellte Weise ab. Der Executor nutzt beim Scannen von Relationen das Speichersystem, führt Sortierungen und Joins aus, wertet Qualifikationen aus und gibt schließlich die abgeleiteten Zeilen zurück.

In den folgenden Abschnitten behandeln wir jeden der oben aufgeführten Punkte ausführlicher, um ein besseres Verständnis der internen Kontroll- und Datenstrukturen von PostgreSQL zu vermitteln.

## 51.2. Wie Verbindungen hergestellt werden

PostgreSQL implementiert ein Client/Server-Modell mit einem Prozess pro Benutzer. In diesem Modell verbindet sich jeder Clientprozess mit genau einem Backend-Prozess. Da wir im Voraus nicht wissen, wie viele Verbindungen hergestellt werden, müssen wir einen Supervisor-Prozess verwenden, der jedes Mal einen neuen Backend-Prozess erzeugt, wenn eine Verbindung angefordert wird. Dieser Supervisor-Prozess heißt `postmaster` und lauscht an einem angegebenen TCP/IP-Port auf eingehende Verbindungen. Sobald er eine Verbindungsanforderung erkennt, erzeugt er einen neuen Backend-Prozess. Diese Backend-Prozesse kommunizieren untereinander und mit anderen Prozessen der Instanz über Semaphore und gemeinsam genutzten Speicher, um die Datenintegrität bei gleichzeitigem Datenzugriff sicherzustellen.

Der Clientprozess kann jedes Programm sein, das das in [Kapitel 54](54_Frontend_Backend_Protokoll.md) beschriebene PostgreSQL-Protokoll versteht. Viele Clients basieren auf der C-Sprachbibliothek `libpq`, aber es gibt mehrere unabhängige Implementierungen des Protokolls, etwa den Java-JDBC-Treiber.

Sobald eine Verbindung hergestellt ist, kann der Clientprozess eine Abfrage an den Backend-Prozess senden, mit dem er verbunden ist. Die Abfrage wird als einfacher Text übertragen, das heißt, im Client findet kein Parsing statt. Der Backend-Prozess parst die Abfrage, erzeugt einen Ausführungsplan, führt den Plan aus und gibt die abgerufenen Zeilen an den Client zurück, indem er sie über die hergestellte Verbindung überträgt.

## 51.3. Die Parser-Phase

Die Parser-Phase besteht aus zwei Teilen:

- Der in `gram.y` und `scan.l` definierte Parser wird mit den Unix-Werkzeugen `bison` und `flex` erzeugt.

- Der Transformationsprozess nimmt Änderungen und Ergänzungen an den Datenstrukturen vor, die vom Parser zurückgegeben werden.

### 51.3.1. Parser

Der Parser muss die Abfragezeichenkette (die als einfacher Text eintrifft) auf gültige Syntax prüfen. Wenn die Syntax korrekt ist, wird ein Parse-Baum aufgebaut und zurückgegeben; andernfalls wird ein Fehler zurückgegeben. Parser und Lexer sind mit den bekannten Unix-Werkzeugen `bison` und `flex` implementiert.

Der Lexer ist in der Datei `scan.l` definiert und dafür verantwortlich, Bezeichner, SQL-Schlüsselwörter usw. zu erkennen. Für jedes gefundene Schlüsselwort oder jeden gefundenen Bezeichner wird ein Token erzeugt und an den Parser übergeben.

Der Parser ist in der Datei `gram.y` definiert und besteht aus einer Menge von Grammatikregeln und Aktionen, die ausgeführt werden, wenn eine Regel ausgelöst wird. Der Code der Aktionen (tatsächlich C-Code) wird verwendet, um den Parse-Baum aufzubauen.

Die Datei `scan.l` wird mit dem Programm `flex` in die C-Quelldatei `scan.c` transformiert, und `gram.y` wird mit `bison` in `gram.c` transformiert. Nachdem diese Transformationen stattgefunden haben, kann ein normaler C-Compiler verwendet werden, um den Parser zu erzeugen. Nehmen Sie niemals Änderungen an den erzeugten C-Dateien vor, da sie beim nächsten Aufruf von `flex` oder `bison` überschrieben werden.

> Die genannten Transformationen und Kompilierungen werden normalerweise automatisch mit den Makefiles ausgeführt, die mit der PostgreSQL-Quelldistribution ausgeliefert werden.

Eine ausführliche Beschreibung von `bison` oder der in `gram.y` angegebenen Grammatikregeln würde den Rahmen dieses Handbuchs sprengen. Es gibt viele Bücher und Dokumente zu `flex` und `bison`. Sie sollten mit `bison` vertraut sein, bevor Sie beginnen, die in `gram.y` angegebene Grammatik zu untersuchen; andernfalls werden Sie nicht verstehen, was dort geschieht.

### 51.3.2. Transformationsprozess

Die Parser-Phase erzeugt einen Parse-Baum, der ausschließlich auf festen Regeln über die syntaktische Struktur von SQL beruht. Sie führt kein Nachschlagen in den Systemkatalogen aus, daher gibt es keine Möglichkeit, die detaillierte Semantik der angeforderten Operationen zu verstehen. Nachdem der Parser abgeschlossen ist, nimmt der Transformationsprozess den vom Parser zurückgegebenen Baum als Eingabe und führt die semantische Interpretation durch, die nötig ist, um zu verstehen, welche Tabellen, Funktionen und Operatoren von der Abfrage referenziert werden. Die Datenstruktur, die zur Darstellung dieser Informationen aufgebaut wird, heißt Abfragebaum.

Der Grund für die Trennung von rohem Parsing und semantischer Analyse ist, dass Zugriffe auf Systemkataloge nur innerhalb einer Transaktion erfolgen können und wir nicht unmittelbar beim Empfang einer Abfragezeichenkette eine Transaktion starten möchten. Die rohe Parser-Phase reicht aus, um Transaktionssteuerungsbefehle (`BEGIN`, `ROLLBACK` usw.) zu identifizieren; diese können dann ohne weitere Analyse korrekt ausgeführt werden. Sobald wir wissen, dass wir es mit einer tatsächlichen Abfrage zu tun haben (etwa `SELECT` oder `UPDATE`), ist es in Ordnung, eine Transaktion zu starten, falls wir uns nicht bereits in einer befinden. Erst dann kann der Transformationsprozess aufgerufen werden.

Der vom Transformationsprozess erzeugte Abfragebaum ähnelt dem rohen Parse-Baum an den meisten Stellen strukturell, unterscheidet sich aber in vielen Details. Beispielsweise stellt ein `FuncCall`-Knoten im Parse-Baum etwas dar, das syntaktisch wie ein Funktionsaufruf aussieht. Dies kann in einen `FuncExpr`- oder einen `Aggref`-Knoten transformiert werden, je nachdem, ob sich der referenzierte Name als gewöhnliche Funktion oder als Aggregatfunktion herausstellt. Außerdem werden Informationen über die tatsächlichen Datentypen von Spalten und Ausdrucksergebnissen zum Abfragebaum hinzugefügt.

## 51.4. Das PostgreSQL-Regelsystem

PostgreSQL unterstützt ein leistungsfähiges Regelsystem zur Spezifikation von Views und mehrdeutigen View-Aktualisierungen. Ursprünglich bestand das PostgreSQL-Regelsystem aus zwei Implementierungen:

- Die erste arbeitete mit zeilenweiser Verarbeitung und war tief im Executor implementiert. Das Regelsystem wurde immer dann aufgerufen, wenn auf eine einzelne Zeile zugegriffen worden war. Diese Implementierung wurde 1995 entfernt, als die letzte offizielle Veröffentlichung des Berkeley-Postgres-Projekts in Postgres95 überführt wurde.

- Die zweite Implementierung des Regelsystems ist eine Technik namens Query Rewriting. Das Rewrite-System ist ein Modul, das zwischen Parser-Phase und Planner/Optimizer existiert. Diese Technik ist weiterhin implementiert.

Der Query Rewriter wird in [Kapitel 39](39_Das_Regelsystem.md) ausführlicher behandelt, daher müssen wir ihn hier nicht weiter erläutern. Wir weisen nur darauf hin, dass sowohl Eingabe als auch Ausgabe des Rewriters Abfragebäume sind; es gibt also keine Änderung in der Darstellung oder im semantischen Detaillierungsgrad der Bäume. Rewriting kann als eine Form der Makroexpansion verstanden werden.

## 51.5. Planner/Optimizer

Die Aufgabe des Planners/Optimizers besteht darin, einen optimalen Ausführungsplan zu erzeugen. Eine gegebene SQL-Abfrage (und damit ein Abfragebaum) kann tatsächlich auf viele verschiedene Arten ausgeführt werden, von denen jede dieselbe Ergebnismenge erzeugt. Wenn es rechnerisch machbar ist, untersucht der Query Optimizer jeden dieser möglichen Ausführungspläne und wählt schließlich den Ausführungsplan aus, von dem erwartet wird, dass er am schnellsten läuft.

> In manchen Situationen würde es übermäßig viel Zeit und Speicher benötigen, jede mögliche Ausführungsweise einer Abfrage zu untersuchen. Dies tritt insbesondere bei Abfragen auf, die eine große Anzahl von Join-Operationen enthalten. Um in angemessener Zeit einen vernünftigen (nicht unbedingt optimalen) Abfrageplan zu bestimmen, verwendet PostgreSQL einen Genetic Query Optimizer (siehe [Kapitel 61](61_Genetischer_Query_Optimizer.md)), wenn die Anzahl der Joins einen Schwellenwert überschreitet (siehe `geqo_threshold`).

Das Suchverfahren des Planners arbeitet tatsächlich mit Datenstrukturen namens Pfade, die einfach reduzierte Darstellungen von Plänen sind und nur so viele Informationen enthalten, wie der Planner für seine Entscheidungen benötigt. Nachdem der billigste Pfad bestimmt wurde, wird ein vollständiger Planbaum aufgebaut, der an den Executor übergeben wird. Dieser stellt den gewünschten Ausführungsplan ausreichend detailliert dar, damit der Executor ihn ausführen kann. Im Rest dieses Abschnitts ignorieren wir die Unterscheidung zwischen Pfaden und Plänen.

### 51.5.1. Mögliche Pläne erzeugen

Der Planner/Optimizer beginnt damit, Pläne für das Scannen jeder einzelnen in der Abfrage verwendeten Relation (Tabelle) zu erzeugen. Die möglichen Pläne werden durch die verfügbaren Indizes auf jeder Relation bestimmt. Es besteht immer die Möglichkeit, einen sequenziellen Scan auf einer Relation auszuführen, daher wird immer ein Plan für einen sequenziellen Scan erzeugt. Nehmen wir an, auf einer Relation ist ein Index definiert (zum Beispiel ein B-Tree-Index) und eine Abfrage enthält die Einschränkung `relation.attribute OPR constant`. Wenn `relation.attribute` zufällig zum Schlüssel des B-Tree-Indexes passt und `OPR` einer der in der Operatorklasse des Indexes aufgeführten Operatoren ist, wird ein weiterer Plan erzeugt, der den B-Tree-Index zum Scannen der Relation verwendet. Wenn weitere Indizes vorhanden sind und die Einschränkungen in der Abfrage zufällig zu einem Schlüssel eines Indexes passen, werden weitere Pläne betrachtet. Indexscan-Pläne werden auch für Indizes erzeugt, die eine Sortierreihenfolge besitzen, die zur `ORDER BY`-Klausel der Abfrage (falls vorhanden) passt, oder eine Sortierreihenfolge, die für einen Merge Join nützlich sein könnte (siehe unten).

Wenn die Abfrage das Verbinden von zwei oder mehr Relationen erfordert, werden Pläne zum Verbinden von Relationen betrachtet, nachdem alle durchführbaren Pläne zum Scannen einzelner Relationen gefunden wurden. Die drei verfügbaren Join-Strategien sind:

- Nested-Loop-Join: Die rechte Relation wird für jede in der linken Relation gefundene Zeile einmal gescannt. Diese Strategie ist einfach zu implementieren, kann aber sehr zeitaufwendig sein. Wenn die rechte Relation jedoch mit einem Indexscan gescannt werden kann, kann dies eine gute Strategie sein. Es ist möglich, Werte aus der aktuellen Zeile der linken Relation als Schlüssel für den Indexscan der rechten zu verwenden.

- Merge Join: Jede Relation wird vor Beginn des Joins nach den Join-Attributen sortiert. Dann werden die beiden Relationen parallel gescannt, und passende Zeilen werden zu Join-Zeilen kombiniert. Diese Art von Join ist attraktiv, weil jede Relation nur einmal gescannt werden muss. Die erforderliche Sortierung kann entweder durch einen expliziten Sortierschritt erreicht werden oder indem die Relation mit einem Index auf dem Join-Schlüssel in der richtigen Reihenfolge gescannt wird.

- Hash Join: Die rechte Relation wird zuerst gescannt und in eine Hash-Tabelle geladen, wobei ihre Join-Attribute als Hash-Schlüssel verwendet werden. Anschließend wird die linke Relation gescannt, und die passenden Werte jeder gefundenen Zeile werden als Hash-Schlüssel verwendet, um die passenden Zeilen in der Tabelle zu finden.

Wenn die Abfrage mehr als zwei Relationen betrifft, muss das Endergebnis durch einen Baum von Join-Schritten aufgebaut werden, von denen jeder zwei Eingaben hat. Der Planner untersucht verschiedene mögliche Join-Reihenfolgen, um die billigste zu finden.

Wenn die Abfrage weniger als `geqo_threshold` Relationen verwendet, wird eine nahezu erschöpfende Suche durchgeführt, um die beste Join-Reihenfolge zu finden. Der Planner betrachtet bevorzugt Joins zwischen zwei Relationen, für die es eine entsprechende Join-Klausel in der `WHERE`-Qualifikation gibt, also eine Einschränkung wie `where rel1.attr1=rel2.attr2`. Join-Paare ohne Join-Klausel werden nur betrachtet, wenn es keine andere Wahl gibt, das heißt, wenn eine bestimmte Relation keine verfügbaren Join-Klauseln zu irgendeiner anderen Relation hat. Für jedes vom Planner betrachtete Join-Paar werden alle möglichen Pläne erzeugt, und derjenige, der (geschätzt) der billigste ist, wird ausgewählt.

Wenn `geqo_threshold` überschritten wird, werden die betrachteten Join-Reihenfolgen durch Heuristiken bestimmt, wie in [Kapitel 61](61_Genetischer_Query_Optimizer.md) beschrieben. Ansonsten ist der Prozess derselbe.

Der fertige Planbaum besteht aus sequenziellen Scans oder Indexscans der Basisrelationen, dazu je nach Bedarf Nested-Loop-, Merge- oder Hash-Join-Knoten sowie allen nötigen Hilfsschritten wie Sort-Knoten oder Knoten zur Berechnung von Aggregatfunktionen. Die meisten dieser Planknotentypen haben zusätzlich die Fähigkeit, Selektion (Verwerfen von Zeilen, die eine angegebene boolesche Bedingung nicht erfüllen) und Projektion (Berechnung einer abgeleiteten Spaltenmenge aus gegebenen Spaltenwerten, also Auswertung skalarer Ausdrücke bei Bedarf) auszuführen. Eine der Verantwortlichkeiten des Planners besteht darin, Selektionsbedingungen aus der `WHERE`-Klausel und die Berechnung erforderlicher Ausdrücke der Ausgabe an die am besten geeigneten Knoten des Planbaums anzuhängen.

## 51.6. Executor

Der Executor nimmt den vom Planner/Optimizer erzeugten Plan und verarbeitet ihn rekursiv, um die erforderliche Menge von Zeilen zu extrahieren. Dies ist im Wesentlichen ein Demand-Pull-Pipeline-Mechanismus. Jedes Mal, wenn ein Planknoten aufgerufen wird, muss er eine weitere Zeile liefern oder melden, dass er keine weiteren Zeilen mehr liefert.

Als konkretes Beispiel nehmen wir an, dass der oberste Knoten ein `MergeJoin`-Knoten ist. Bevor irgendein Merge durchgeführt werden kann, müssen zwei Zeilen geholt werden (eine aus jedem Unterplan). Also ruft der Executor sich rekursiv selbst auf, um die Unterpläne zu verarbeiten; er beginnt mit dem Unterplan, der an `lefttree` hängt. Der neue oberste Knoten (der oberste Knoten des linken Unterplans) sei ein `Sort`-Knoten, und erneut ist Rekursion nötig, um eine Eingabezeile zu erhalten. Der Kindknoten des `Sort` könnte ein `SeqScan`-Knoten sein, der das tatsächliche Lesen einer Tabelle darstellt. Die Ausführung dieses Knotens veranlasst den Executor, eine Zeile aus der Tabelle zu holen und sie an den aufrufenden Knoten zurückzugeben. Der `Sort`-Knoten ruft sein Kind wiederholt auf, um alle zu sortierenden Zeilen zu erhalten. Wenn die Eingabe erschöpft ist (angezeigt dadurch, dass der Kindknoten `NULL` statt einer Zeile zurückgibt), führt der `Sort`-Code die Sortierung aus und kann schließlich seine erste Ausgabezeile zurückgeben, nämlich die erste in sortierter Reihenfolge. Er hält die übrigen Zeilen gespeichert, damit er sie bei späteren Anforderungen in sortierter Reihenfolge liefern kann.

Der `MergeJoin`-Knoten fordert ebenso die erste Zeile von seinem rechten Unterplan an. Dann vergleicht er die beiden Zeilen, um zu sehen, ob sie gejoint werden können; wenn ja, gibt er eine Join-Zeile an seinen Aufrufer zurück. Beim nächsten Aufruf, oder unmittelbar, wenn das aktuelle Eingabepaar nicht gejoint werden kann, geht er zur nächsten Zeile der einen oder der anderen Tabelle über (abhängig davon, wie der Vergleich ausgefallen ist) und prüft erneut auf eine Übereinstimmung. Schließlich ist der eine oder andere Unterplan erschöpft, und der `MergeJoin`-Knoten gibt `NULL` zurück, um anzuzeigen, dass keine weiteren Join-Zeilen gebildet werden können.

Komplexe Abfragen können viele Ebenen von Planknoten umfassen, aber der allgemeine Ansatz ist derselbe: Jeder Knoten berechnet seine nächste Ausgabezeile und gibt sie jedes Mal zurück, wenn er aufgerufen wird. Jeder Knoten ist außerdem dafür verantwortlich, alle Selektions- oder Projektionsausdrücke anzuwenden, die ihm vom Planner zugewiesen wurden.

Der Executor-Mechanismus wird zur Auswertung aller fünf grundlegenden SQL-Abfragetypen verwendet: `SELECT`, `INSERT`, `UPDATE`, `DELETE` und `MERGE`. Bei `SELECT` muss der Top-Level-Executor-Code lediglich jede vom Abfrageplanbaum zurückgegebene Zeile an den Client senden. `INSERT ... SELECT`, `UPDATE`, `DELETE` und `MERGE` sind effektiv `SELECT`s unter einem speziellen Top-Level-Planknoten namens `ModifyTable`.

`INSERT ... SELECT` reicht die Zeilen an `ModifyTable` zur Einfügung weiter. Bei `UPDATE` sorgt der Planner dafür, dass jede berechnete Zeile alle aktualisierten Spaltenwerte sowie die TID (Tuple-ID oder Zeilen-ID) der ursprünglichen Zielzeile enthält; diese Daten werden an den `ModifyTable`-Knoten weitergereicht, der sie verwendet, um eine neue aktualisierte Zeile zu erzeugen und die alte Zeile als gelöscht zu markieren. Bei `DELETE` ist die einzige Spalte, die tatsächlich vom Plan zurückgegeben wird, die TID, und der `ModifyTable`-Knoten verwendet die TID einfach, um jede Zielzeile aufzusuchen und als gelöscht zu markieren. Bei `MERGE` verbindet der Planner die Quell- und Zielrelationen und fügt alle Spaltenwerte ein, die von den `WHEN`-Klauseln benötigt werden, sowie die TID der Zielzeile; diese Daten werden an den `ModifyTable`-Knoten weitergereicht, der anhand der Informationen bestimmt, welche `WHEN`-Klausel auszuführen ist, und dann die Zielzeile wie erforderlich einfügt, aktualisiert oder löscht.

Ein einfacher Befehl `INSERT ... VALUES` erzeugt einen trivialen Planbaum, der aus einem einzigen `Result`-Knoten besteht. Dieser berechnet genau eine Ergebniszeile und reicht sie an `ModifyTable` weiter, um die Einfügung auszuführen.
