# 61. Genetischer Query-Optimizer

Geschrieben von Martin Utesch (`<utesch@aut.tu-freiberg.de>`) für das Institut für Automatisierungstechnik an der Technischen Universität Bergakademie Freiberg.

## 61.1. Abfrageverarbeitung als komplexes Optimierungsproblem

Unter allen relationalen Operatoren ist der Join der am schwierigsten zu verarbeitende und zu optimierende. Die Anzahl möglicher Abfragepläne wächst exponentiell mit der Anzahl der Joins in der Abfrage. Zusätzlicher Optimierungsaufwand entsteht durch die Unterstützung verschiedener Join-Methoden, etwa Nested Loop, Hash Join und Merge Join in PostgreSQL, sowie durch unterschiedliche Indizes, etwa B-Tree, Hash, GiST und GIN in PostgreSQL, die als Zugriffspfade für Relationen dienen.

Der normale PostgreSQL-Query-Optimizer führt eine nahezu erschöpfende Suche im Raum alternativer Strategien aus. Dieser Algorithmus, der zuerst in IBMs Datenbanksystem System R eingeführt wurde, erzeugt eine nahezu optimale Join-Reihenfolge, kann aber enorme Mengen an Zeit und Speicher benötigen, wenn die Anzahl der Joins in der Abfrage groß wird. Dadurch ist der gewöhnliche PostgreSQL-Query-Optimizer für Abfragen ungeeignet, die eine große Anzahl von Tabellen joinen.

Das Institut für Automatisierungstechnik an der Technischen Universität Bergakademie Freiberg stieß auf Probleme, als PostgreSQL als Backend für ein wissensbasiertes Entscheidungsunterstützungssystem zur Wartung eines elektrischen Stromnetzes verwendet werden sollte. Das DBMS musste große Join-Abfragen für die Inferenzmaschine des wissensbasierten Systems verarbeiten. Die Anzahl der Joins in diesen Abfragen machte den Einsatz des normalen Query-Optimizers unpraktikabel.

Im Folgenden beschreiben wir die Implementierung eines genetischen Algorithmus, der das Problem der Join-Reihenfolge auf eine Weise löst, die für Abfragen mit vielen Joins effizient ist.

## 61.2. Genetische Algorithmen

Der genetische Algorithmus (GA) ist ein heuristisches Optimierungsverfahren, das mit randomisierter Suche arbeitet. Die Menge möglicher Lösungen des Optimierungsproblems wird als Population von Individuen betrachtet. Der Grad der Anpassung eines Individuums an seine Umgebung wird durch seine Fitness angegeben.

Die Koordinaten eines Individuums im Suchraum werden durch Chromosomen dargestellt, im Wesentlichen durch Mengen von Zeichenketten. Ein Gen ist ein Abschnitt eines Chromosoms, der den Wert eines einzelnen zu optimierenden Parameters kodiert. Typische Kodierungen für ein Gen können binär oder ganzzahlig sein.

Durch die Simulation der evolutionären Operationen Rekombination, Mutation und Selektion werden neue Generationen von Suchpunkten gefunden, die im Durchschnitt eine höhere Fitness aufweisen als ihre Vorfahren. Abbildung 61.1 veranschaulicht diese Schritte.

**Abbildung 61.1. Struktur eines genetischen Algorithmus**

```text
INITIALIZE t := 0
P(t): Generation der Vorfahren zum Zeitpunkt t
P''(t): Generation der Nachkommen zum Zeitpunkt t

INITIALIZE P(t)

evaluate FITNESS of P(t)

STOPPING CRITERION

false

P'(t) := RECOMBINATION{P(t)}

P''(t) := MUTATION{P'(t)}

true

P(t+1) := SELECTION{P''(t) + P(t)}

evaluate FITNESS of P''(t)

t := t + 1

end
```

Nach der FAQ von `comp.ai.genetic` kann nicht stark genug betont werden, dass ein GA keine rein zufällige Suche nach einer Problemlösung ist. Ein GA verwendet stochastische Prozesse, aber das Ergebnis ist deutlich nicht-zufällig, also besser als zufällig.

## 61.3. Genetische Query-Optimierung (GEQO) in PostgreSQL

Das GEQO-Modul behandelt das Problem der Abfrageoptimierung so, als wäre es das bekannte Travelling-Salesman-Problem (TSP). Mögliche Abfragepläne werden als Integer-Strings kodiert. Jeder String repräsentiert die Join-Reihenfolge von einer Relation der Abfrage zur nächsten. Der Join-Baum

```text
   /\
  /\ 2
 /\ 3
4 1
```

wird zum Beispiel durch den Integer-String `4-1-3-2` kodiert. Das bedeutet: zuerst Relation `4` und `1` joinen, dann `3` und schließlich `2`, wobei 1, 2, 3 und 4 Relations-IDs innerhalb des PostgreSQL-Optimizers sind.

Besondere Merkmale der GEQO-Implementierung in PostgreSQL sind:

- Die Verwendung eines Steady-State-GA, bei dem die am wenigsten fitten Individuen einer Population ersetzt werden und nicht eine ganze Generation, ermöglicht schnelle Konvergenz zu verbesserten Abfrageplänen. Das ist wesentlich für Abfrageverarbeitung in angemessener Zeit.

- Die Verwendung von Edge-Recombination-Crossover ist besonders geeignet, um beim Lösen des TSP mit einem GA den Verlust von Kanten gering zu halten.

- Mutation als genetischer Operator ist veraltet, sodass keine Reparaturmechanismen benötigt werden, um gültige TSP-Touren zu erzeugen.

Teile des GEQO-Moduls sind aus D. Whitleys Genitor-Algorithmus abgeleitet.

Das GEQO-Modul erlaubt dem PostgreSQL-Query-Optimizer, große Join-Abfragen durch nicht-erschöpfende Suche effektiv zu unterstützen.

### 61.3.1. Mögliche Pläne mit GEQO erzeugen

Der GEQO-Planungsprozess verwendet den Standard-Planner-Code, um Pläne für Scans einzelner Relationen zu erzeugen. Anschließend werden Join-Pläne mit dem genetischen Ansatz entwickelt. Wie oben gezeigt, wird jeder Kandidat für einen Join-Plan durch eine Sequenz repräsentiert, in der die Basisrelationen gejoint werden. In der Anfangsphase erzeugt der GEQO-Code einfach einige mögliche Join-Sequenzen zufällig. Für jede betrachtete Join-Sequenz wird der Standard-Planner-Code aufgerufen, um die Kosten der Ausführung der Abfrage mit dieser Join-Sequenz zu schätzen. Für jeden Schritt der Join-Sequenz werden alle drei möglichen Join-Strategien betrachtet; außerdem stehen alle anfänglich bestimmten Relations-Scan-Pläne zur Verfügung. Die geschätzten Kosten sind die günstigsten dieser Möglichkeiten.

Join-Sequenzen mit niedrigeren geschätzten Kosten gelten als »fitter« als solche mit höheren Kosten. Der genetische Algorithmus verwirft die am wenigsten fitten Kandidaten. Danach werden neue Kandidaten erzeugt, indem Gene fitterer Kandidaten kombiniert werden, also zufällig gewählte Teile bekannter Join-Sequenzen mit niedrigen Kosten verwendet werden, um neue zu betrachtende Sequenzen zu erzeugen. Dieser Prozess wird wiederholt, bis eine voreingestellte Anzahl von Join-Sequenzen betrachtet wurde. Danach wird die während der Suche irgendwann gefundene beste Sequenz verwendet, um den fertigen Plan zu erzeugen.

Dieser Prozess ist inhärent nichtdeterministisch, weil sowohl bei der Auswahl der Anfangspopulation als auch bei der anschließenden »Mutation« der besten Kandidaten randomisierte Entscheidungen getroffen werden. Um überraschende Änderungen des ausgewählten Plans zu vermeiden, startet jeder Lauf des GEQO-Algorithmus seinen Zufallszahlengenerator mit der aktuellen Einstellung des Parameters `geqo_seed` neu. Solange `geqo_seed` und die anderen GEQO-Parameter unverändert bleiben, wird für eine gegebene Abfrage und andere Planner-Eingaben wie Statistiken derselbe Plan erzeugt. Um mit verschiedenen Suchpfaden zu experimentieren, ändern Sie `geqo_seed`.

### 61.3.2. Künftige Implementierungsaufgaben für PostgreSQL GEQO

Es ist weiterhin Arbeit nötig, um die Parametereinstellungen des genetischen Algorithmus zu verbessern. In der Datei `src/backend/optimizer/geqo/geqo_main.c`, in den Routinen `gimme_pool_size` und `gimme_number_generations`, muss ein Kompromiss bei den Parametereinstellungen gefunden werden, um zwei konkurrierende Anforderungen zu erfüllen:

- Optimalität des Abfrageplans
- Rechenzeit

In der aktuellen Implementierung wird die Fitness jeder Kandidaten-Join-Sequenz geschätzt, indem die Join-Auswahl und Kostenschätzung des Standard-Planners von Grund auf ausgeführt werden. Soweit verschiedene Kandidaten ähnliche Teilsequenzen von Joins verwenden, wird viel Arbeit wiederholt. Dies könnte deutlich schneller gemacht werden, indem Kostenschätzungen für Teil-Joins aufbewahrt werden. Das Problem besteht darin, keinen unangemessen großen Speicheraufwand für das Vorhalten dieses Zustands zu verursachen.

Auf einer grundlegenderen Ebene ist nicht klar, ob es angemessen ist, die Abfrageoptimierung mit einem GA-Algorithmus zu lösen, der für TSP entworfen wurde. Im TSP-Fall sind die mit einem beliebigen Teilstring, also einer Teiltour, verbundenen Kosten unabhängig vom Rest der Tour. Für die Abfrageoptimierung trifft das eindeutig nicht zu. Daher ist fraglich, ob Edge-Recombination-Crossover das effektivste Mutationsverfahren ist.

## 61.4. Weiterführende Literatur

Die folgenden Ressourcen enthalten zusätzliche Informationen über genetische Algorithmen:

- The Hitch-Hiker's Guide to Evolutionary Computation, FAQ für `news://comp.ai.genetic`
- Evolutionary Computation and its application to art and design, von Craig Reynolds
- `[elma04]`
- `[fong]`

```text
http://www.faqs.org/faqs/ai-faq/genetic/part1/
https://www.red3d.com/cwr/evolve.html
```
