# 15. Parallele Abfragen

PostgreSQL kann Abfragepläne erstellen, die mehrere CPUs nutzen, um Abfragen schneller zu beantworten. Diese Funktion wird als parallele Abfrage bezeichnet. Viele Abfragen können von paralleler Ausführung nicht profitieren, entweder wegen Einschränkungen der aktuellen Implementierung oder weil es keinen denkbaren Abfrageplan gibt, der schneller wäre als der serielle Plan. Bei Abfragen, die profitieren können, ist die Beschleunigung durch parallele Ausführung jedoch oft sehr deutlich. Viele Abfragen können mit paralleler Ausführung mehr als doppelt so schnell laufen; manche laufen viermal so schnell oder sogar noch schneller. Am meisten profitieren typischerweise Abfragen, die eine große Datenmenge berühren, dem Benutzer aber nur wenige Zeilen zurückgeben. Dieses Kapitel erklärt einige Details dazu, wie parallele Abfragen funktionieren und in welchen Situationen sie verwendet werden können, damit Benutzer einschätzen können, was sie erwarten dürfen.

## 15.1. Wie parallele Abfragen funktionieren

Wenn der Optimizer feststellt, dass eine parallele Abfrage die schnellste Ausführungsstrategie für eine bestimmte Abfrage ist, erstellt er einen Abfrageplan, der einen `Gather`- oder `Gather Merge`-Knoten enthält. Ein einfaches Beispiel:

```sql
EXPLAIN SELECT * FROM pgbench_accounts WHERE filler LIKE '%x%';
```

```text
                                     QUERY PLAN
--------------------------------------------------------------------------------
 Gather (cost=1000.00..217018.43 rows=1 width=97)
   Workers Planned: 2
   -> Parallel Seq Scan on pgbench_accounts (cost=0.00..216018.33 rows=1 width=97)
         Filter: (filler ~~ '%x%'::text)
(4 rows)
```

In allen Fällen hat der `Gather`- oder `Gather Merge`-Knoten genau einen Kindplan. Dieser Kindplan ist der Teil des Plans, der parallel ausgeführt wird. Wenn der `Gather`- oder `Gather Merge`-Knoten ganz oben im Planbaum steht, wird die gesamte Abfrage parallel ausgeführt. Wenn er an einer anderen Stelle im Planbaum steht, läuft nur der Planabschnitt unterhalb dieses Knotens parallel. Im obigen Beispiel greift die Abfrage nur auf eine Tabelle zu. Deshalb gibt es außer dem `Gather`-Knoten selbst nur einen weiteren Planknoten; da dieser Planknoten ein Kind des `Gather`-Knotens ist, wird er parallel ausgeführt.

Mit `EXPLAIN` können Sie sehen, wie viele Worker der Planner gewählt hat. Wenn der `Gather`-Knoten während der Ausführung erreicht wird, fordert der Prozess, der die Sitzung des Benutzers ausführt, so viele Hintergrund-Worker-Prozesse an, wie der Planner gewählt hat. Die Zahl der Hintergrund-Worker, deren Verwendung der Planner erwägt, ist höchstens durch `max_parallel_workers_per_gather` begrenzt. Die Gesamtzahl der Hintergrund-Worker, die zu einem Zeitpunkt existieren können, wird sowohl durch `max_worker_processes` als auch durch `max_parallel_workers` begrenzt. Daher kann eine parallele Abfrage mit weniger Workern als geplant laufen oder sogar ganz ohne Worker. Der optimale Plan kann von der Zahl verfügbarer Worker abhängen, sodass dies zu schlechter Abfrageperformance führen kann. Wenn das häufig vorkommt, sollten Sie erwägen, `max_worker_processes` und `max_parallel_workers` zu erhöhen, damit mehr Worker gleichzeitig laufen können, oder alternativ `max_parallel_workers_per_gather` zu reduzieren, damit der Planner weniger Worker anfordert.

Jeder Hintergrund-Worker-Prozess, der für eine bestimmte parallele Abfrage erfolgreich gestartet wird, führt den parallelen Teil des Plans aus. Auch der Leader führt diesen Planabschnitt aus, hat aber eine zusätzliche Aufgabe: Er muss außerdem alle von den Workern erzeugten Tupel lesen. Wenn der parallele Planabschnitt nur wenige Tupel erzeugt, verhält sich der Leader oft fast wie ein zusätzlicher Worker und beschleunigt die Abfrageausführung. Wenn der parallele Planabschnitt dagegen viele Tupel erzeugt, kann der Leader fast vollständig damit beschäftigt sein, die von den Workern erzeugten Tupel zu lesen und weitere Verarbeitungsschritte auszuführen, die von Planknoten oberhalb des `Gather`- oder `Gather Merge`-Knotens benötigt werden. In solchen Fällen erledigt der Leader nur sehr wenig von der eigentlichen Arbeit im parallelen Planabschnitt.

Wenn der oberste Knoten des parallelen Planabschnitts `Gather Merge` statt `Gather` ist, bedeutet das, dass jeder Prozess, der den parallelen Planabschnitt ausführt, Tupel in sortierter Reihenfolge erzeugt und dass der Leader diese Ergebnisse ordnungserhaltend zusammenführt. Im Gegensatz dazu liest `Gather` Tupel von den Workern in der jeweils passenden Reihenfolge und zerstört dabei jede möglicherweise vorhandene Sortierreihenfolge.

## 15.2. Wann können parallele Abfragen verwendet werden?

Mehrere Einstellungen können dazu führen, dass der Query Planner unter keinen Umständen einen parallelen Abfrageplan erzeugt. Damit überhaupt parallele Abfragepläne erzeugt werden können, müssen die folgenden Einstellungen wie angegeben konfiguriert sein.

- `max_parallel_workers_per_gather` muss auf einen Wert größer als null gesetzt sein. Dies ist ein Spezialfall des allgemeineren Prinzips, dass nicht mehr Worker verwendet werden sollten als die Zahl, die über `max_parallel_workers_per_gather` konfiguriert ist.

Außerdem darf das System nicht im Single-User-Modus laufen. Da in dieser Situation das gesamte Datenbanksystem als einzelner Prozess läuft, stehen keine Hintergrund-Worker zur Verfügung.

Auch wenn parallele Abfragepläne grundsätzlich erzeugt werden können, erzeugt der Planner für eine bestimmte Abfrage keinen solchen Plan, wenn eine der folgenden Bedingungen zutrifft:

- Die Abfrage schreibt Daten oder sperrt Datenbankzeilen. Wenn eine Abfrage eine datenverändernde Operation entweder auf oberster Ebene oder innerhalb einer CTE enthält, werden für diese Abfrage keine parallelen Pläne erzeugt. Als Ausnahme können die folgenden Befehle, die eine neue Tabelle erzeugen und befüllen, für den zugrunde liegenden `SELECT`-Teil der Abfrage einen parallelen Plan verwenden:

  - `CREATE TABLE ... AS`
  - `SELECT INTO`
  - `CREATE MATERIALIZED VIEW`
  - `REFRESH MATERIALIZED VIEW`

- Die Abfrage könnte während der Ausführung angehalten werden. In jeder Situation, in der das System annimmt, dass teilweise oder inkrementelle Ausführung auftreten könnte, wird kein paralleler Plan erzeugt. Ein mit `DECLARE CURSOR` erstellter Cursor verwendet zum Beispiel niemals einen parallelen Plan. Ebenso verwendet eine PL/pgSQL-Schleife der Form `FOR x IN query LOOP .. END LOOP` niemals einen parallelen Plan, weil das System für parallele Abfragen nicht prüfen kann, ob der Code in der Schleife sicher ausgeführt werden kann, während parallele Abfrageausführung aktiv ist.

- Die Abfrage verwendet eine Funktion, die als `PARALLEL UNSAFE` markiert ist. Die meisten systemdefinierten Funktionen sind `PARALLEL SAFE`, aber benutzerdefinierte Funktionen werden standardmäßig als `PARALLEL UNSAFE` markiert. Siehe dazu [Abschnitt 15.4](#154-parallele-sicherheit).

- Die Abfrage läuft innerhalb einer anderen Abfrage, die bereits parallel ist. Wenn zum Beispiel eine von einer parallelen Abfrage aufgerufene Funktion selbst eine SQL-Abfrage ausführt, verwendet diese SQL-Abfrage niemals einen parallelen Plan. Das ist eine Einschränkung der aktuellen Implementierung. Es ist aber möglicherweise nicht wünschenswert, diese Einschränkung zu entfernen, weil sonst eine einzelne Abfrage eine sehr große Zahl von Prozessen verwenden könnte.

Selbst wenn für eine bestimmte Abfrage ein paralleler Abfrageplan erzeugt wird, gibt es mehrere Umstände, unter denen dieser Plan zur Ausführungszeit nicht parallel ausgeführt werden kann. Wenn das geschieht, führt der Leader den Teil des Plans unterhalb des `Gather`-Knotens vollständig allein aus, fast so, als wäre der `Gather`-Knoten nicht vorhanden. Das passiert, wenn eine der folgenden Bedingungen erfüllt ist:

- Es können keine Hintergrund-Worker beschafft werden, weil die Gesamtzahl der Hintergrund-Worker `max_worker_processes` nicht überschreiten darf.
- Es können keine Hintergrund-Worker beschafft werden, weil die Gesamtzahl der für parallele Abfragen gestarteten Hintergrund-Worker `max_parallel_workers` nicht überschreiten darf.
- Der Client sendet eine `Execute`-Nachricht mit einer Fetch-Zahl ungleich null. Siehe dazu die Beschreibung des erweiterten Abfrageprotokolls. Da `libpq` derzeit keine Möglichkeit bietet, eine solche Nachricht zu senden, kann dies nur bei einem Client auftreten, der nicht auf `libpq` beruht. Wenn das häufig vorkommt, kann es sinnvoll sein, `max_parallel_workers_per_gather` in Sitzungen, in denen dies wahrscheinlich ist, auf null zu setzen, damit keine Abfragepläne erzeugt werden, die bei serieller Ausführung suboptimal sein können.

## 15.3. Parallele Pläne

Da jeder Worker den parallelen Teil des Plans vollständig ausführt, kann man nicht einfach einen gewöhnlichen Abfrageplan nehmen und ihn mit mehreren Workern laufen lassen. Jeder Worker würde eine vollständige Kopie der Ausgabemenge erzeugen; die Abfrage wäre also nicht schneller als normal und würde falsche Ergebnisse liefern. Stattdessen muss der parallele Teil des Plans das sein, was intern im Query Optimizer als Partial Plan bezeichnet wird. Das heißt, er muss so konstruiert sein, dass jeder Prozess, der den Plan ausführt, nur eine Teilmenge der Ausgabezeilen erzeugt, und zwar so, dass jede benötigte Ausgabezeile garantiert von genau einem der zusammenarbeitenden Prozesse erzeugt wird. Im Allgemeinen bedeutet dies, dass der Scan auf der treibenden Tabelle der Abfrage ein parallelfähiger Scan sein muss.

### 15.3.1. Parallele Scans

Die folgenden Arten parallelfähiger Tabellenscans werden derzeit unterstützt.

- Bei einem parallelen sequenziellen Scan werden die Blöcke der Tabelle in Bereiche aufgeteilt und unter den zusammenarbeitenden Prozessen geteilt. Jeder Worker-Prozess scannt seinen zugewiesenen Blockbereich vollständig, bevor er einen weiteren Blockbereich anfordert.

- Bei einem parallelen Bitmap-Heap-Scan wird ein Prozess als Leader gewählt. Dieser Prozess scannt einen oder mehrere Indizes und erstellt eine Bitmap, die angibt, welche Tabellenblöcke besucht werden müssen. Diese Blöcke werden anschließend wie bei einem parallelen sequenziellen Scan unter den zusammenarbeitenden Prozessen aufgeteilt. Anders gesagt: Der Heap-Scan wird parallel ausgeführt, der zugrunde liegende Index-Scan jedoch nicht.

- Bei einem parallelen Index-Scan oder parallelen Index-Only-Scan lesen die zusammenarbeitenden Prozesse abwechselnd Daten aus dem Index. Derzeit werden parallele Index-Scans nur für B-Tree-Indizes unterstützt. Jeder Prozess beansprucht einen einzelnen Indexblock, scannt ihn und gibt alle Tupel zurück, auf die dieser Block verweist; gleichzeitig können andere Prozesse Tupel aus anderen Indexblöcken zurückgeben. Die Ergebnisse eines parallelen B-Tree-Scans werden innerhalb jedes Worker-Prozesses in sortierter Reihenfolge zurückgegeben.

Andere Scantypen, etwa Scans von Nicht-B-Tree-Indizes, könnten in Zukunft parallele Scans unterstützen.

### 15.3.2. Parallele Joins

Wie in einem nicht parallelen Plan kann die treibende Tabelle mit einer oder mehreren anderen Tabellen über Nested Loop, Hash Join oder Merge Join verbunden werden. Die innere Seite des Joins kann jede Art nicht parallelen Plans sein, die der Planner sonst unterstützt, sofern sie sicher innerhalb eines parallelen Workers ausgeführt werden kann. Je nach Join-Typ kann auch die innere Seite ein paralleler Plan sein.

- Bei einem Nested-Loop-Join ist die innere Seite immer nicht parallel. Obwohl sie vollständig ausgeführt wird, ist dies effizient, wenn die innere Seite ein Index-Scan ist, weil die äußeren Tupel und damit die Schleifen, die Werte im Index nachschlagen, auf die zusammenarbeitenden Prozesse verteilt werden.

- Bei einem Merge Join ist die innere Seite immer ein nicht paralleler Plan und wird daher vollständig ausgeführt. Das kann ineffizient sein, besonders wenn eine Sortierung ausgeführt werden muss, weil die Arbeit und die resultierenden Daten in jedem zusammenarbeitenden Prozess dupliziert werden.

- Bei einem Hash Join ohne das Präfix `parallel` wird die innere Seite von jedem zusammenarbeitenden Prozess vollständig ausgeführt, um identische Kopien der Hash-Tabelle aufzubauen. Das kann ineffizient sein, wenn die Hash-Tabelle groß oder der Plan teuer ist. Bei einem parallelen Hash Join ist die innere Seite ein paralleler Hash, der die Arbeit zum Aufbau einer gemeinsam genutzten Hash-Tabelle auf die zusammenarbeitenden Prozesse verteilt.

### 15.3.3. Parallele Aggregation

PostgreSQL unterstützt parallele Aggregation durch Aggregation in zwei Stufen. Zunächst führt jeder Prozess, der am parallelen Teil der Abfrage beteiligt ist, einen Aggregationsschritt aus und erzeugt ein Teilergebnis für jede Gruppe, die diesem Prozess bekannt ist. Im Plan erscheint dies als `Partial Aggregate`-Knoten. Anschließend werden die Teilergebnisse über `Gather` oder `Gather Merge` an den Leader übertragen. Schließlich aggregiert der Leader die Ergebnisse über alle Worker hinweg erneut, um das endgültige Ergebnis zu erzeugen. Im Plan erscheint dies als `Finalize Aggregate`-Knoten.

Da der `Finalize Aggregate`-Knoten im Leader-Prozess läuft, erscheinen Abfragen, die im Verhältnis zur Zahl der Eingabezeilen relativ viele Gruppen erzeugen, dem Query Planner weniger attraktiv. Im schlimmsten Fall könnte die Zahl der Gruppen, die der `Finalize Aggregate`-Knoten sieht, genauso groß sein wie die Zahl der Eingabezeilen, die alle Worker-Prozesse in der `Partial Aggregate`-Stufe gesehen haben. In solchen Fällen gibt es offensichtlich keinen Performancevorteil durch parallele Aggregation. Der Query Planner berücksichtigt dies während der Planung und wird in diesem Szenario wahrscheinlich keine parallele Aggregation wählen.

Parallele Aggregation wird nicht in allen Situationen unterstützt. Jedes Aggregat muss parallelitätssicher sein und eine Combine-Funktion besitzen. Wenn das Aggregat einen Transition State vom Typ `internal` hat, muss es Serialisierungs- und Deserialisierungsfunktionen besitzen. Weitere Details finden Sie bei `CREATE AGGREGATE`. Parallele Aggregation wird nicht unterstützt, wenn ein Aggregatfunktionsaufruf eine `DISTINCT`- oder `ORDER BY`-Klausel enthält. Sie wird außerdem nicht für Ordered-Set-Aggregate unterstützt und nicht, wenn die Abfrage `GROUPING SETS` enthält. Sie kann nur verwendet werden, wenn alle an der Abfrage beteiligten Joins ebenfalls Teil des parallelen Planabschnitts sind.

### 15.3.4. Parallel Append

Immer wenn PostgreSQL Zeilen aus mehreren Quellen zu einer einzigen Ergebnismenge kombinieren muss, verwendet es einen `Append`- oder `MergeAppend`-Planknoten. Das passiert häufig bei der Umsetzung von `UNION ALL` oder beim Scannen einer partitionierten Tabelle. Solche Knoten können in parallelen Plänen genauso verwendet werden wie in jedem anderen Plan. In einem parallelen Plan kann der Planner stattdessen jedoch einen `Parallel Append`-Knoten verwenden.

Wenn ein `Append`-Knoten in einem parallelen Plan verwendet wird, führt jeder Prozess die Kindpläne in der Reihenfolge aus, in der sie erscheinen. Alle beteiligten Prozesse arbeiten also gemeinsam am ersten Kindplan, bis er abgeschlossen ist, und wechseln dann ungefähr gleichzeitig zum zweiten Plan. Wird stattdessen `Parallel Append` verwendet, verteilt der Executor die beteiligten Prozesse möglichst gleichmäßig auf die Kindpläne, sodass mehrere Kindpläne gleichzeitig ausgeführt werden. Dadurch werden Konflikte vermieden, und außerdem entstehen die Startkosten eines Kindplans nicht in Prozessen, die ihn nie ausführen.

Außerdem kann ein `Parallel Append`-Knoten, anders als ein normaler `Append`-Knoten, der in einem parallelen Plan nur partielle Kindpläne haben kann, sowohl partielle als auch nicht partielle Kindpläne besitzen. Nicht partielle Kindpläne werden nur von einem einzigen Prozess gescannt, weil mehrfaches Scannen doppelte Ergebnisse erzeugen würde. Pläne, die mehrere Ergebnismengen aneinanderhängen, können daher grobkörnige Parallelität erreichen, selbst wenn keine effizienten partiellen Pläne verfügbar sind. Betrachten Sie zum Beispiel eine Abfrage auf eine partitionierte Tabelle, die nur effizient umgesetzt werden kann, indem ein Index verwendet wird, der keine parallelen Scans unterstützt. Der Planner könnte einen `Parallel Append` regulärer `Index Scan`-Pläne wählen; jeder einzelne Index-Scan müsste von einem einzelnen Prozess vollständig ausgeführt werden, aber verschiedene Scans könnten gleichzeitig von verschiedenen Prozessen ausgeführt werden.

`enable_parallel_append` kann verwendet werden, um diese Funktion zu deaktivieren.

### 15.3.5. Tipps zu parallelen Plänen

Wenn eine Abfrage, von der Sie es erwarten, keinen parallelen Plan erzeugt, können Sie versuchen, `parallel_setup_cost` oder `parallel_tuple_cost` zu reduzieren. Natürlich kann sich dieser Plan als langsamer herausstellen als der serielle Plan, den der Planner bevorzugt hat, aber das ist nicht immer der Fall. Wenn Sie selbst mit sehr kleinen Werten für diese Einstellungen keinen parallelen Plan erhalten, zum Beispiel nachdem Sie beide auf null gesetzt haben, gibt es möglicherweise einen Grund, warum der Query Planner für Ihre Abfrage keinen parallelen Plan erzeugen kann. Informationen dazu, warum dies der Fall sein kann, finden Sie in [Abschnitt 15.2](#152-wann-können-parallele-abfragen-verwendet-werden) und [Abschnitt 15.4](#154-parallele-sicherheit).

Bei der Ausführung eines parallelen Plans können Sie `EXPLAIN (ANALYZE, VERBOSE)` verwenden, um für jeden Planknoten Statistiken pro Worker anzuzeigen. Das kann nützlich sein, um festzustellen, ob die Arbeit gleichmäßig auf alle Planknoten verteilt wird, und allgemeiner, um die Performanceeigenschaften des Plans zu verstehen.

## 15.4. Parallele Sicherheit

Der Planner klassifiziert Operationen, die an einer Abfrage beteiligt sind, als parallel safe, parallel restricted oder parallel unsafe. Eine parallel sichere Operation ist eine Operation, die nicht mit der Verwendung paralleler Abfragen in Konflikt gerät. Eine parallel eingeschränkte Operation kann nicht in einem parallelen Worker ausgeführt werden, kann aber im Leader ausgeführt werden, während parallele Abfrageausführung aktiv ist. Daher können parallel eingeschränkte Operationen niemals unterhalb eines `Gather`- oder `Gather Merge`-Knotens vorkommen, wohl aber an anderer Stelle in einem Plan, der einen solchen Knoten enthält. Eine parallel unsichere Operation kann nicht ausgeführt werden, während parallele Abfrageausführung aktiv ist, nicht einmal im Leader. Wenn eine Abfrage irgendetwas enthält, das parallel unsicher ist, wird parallele Abfrageausführung für diese Abfrage vollständig deaktiviert.

Die folgenden Operationen sind immer parallel eingeschränkt:

- Scans von Common Table Expressions (CTEs).
- Scans temporärer Tabellen.
- Scans fremder Tabellen, sofern der Foreign Data Wrapper nicht über eine `IsForeignScanParallelSafe`-API verfügt, die etwas anderes angibt.
- Planknoten, die einen korrelierten `SubPlan` referenzieren.

### 15.4.1. Parallele Kennzeichnung für Funktionen und Aggregate

Der Planner kann nicht automatisch feststellen, ob eine benutzerdefinierte Funktion oder ein benutzerdefiniertes Aggregat parallel sicher, parallel eingeschränkt oder parallel unsicher ist, weil er dazu jede Operation vorhersagen müsste, die die Funktion möglicherweise ausführen kann. Im Allgemeinen entspricht das dem Halteproblem und ist daher unmöglich. Selbst bei einfachen Funktionen, bei denen dies vorstellbar wäre, wird es nicht versucht, weil es teuer und fehleranfällig wäre. Stattdessen wird für alle benutzerdefinierten Funktionen angenommen, dass sie parallel unsicher sind, sofern sie nicht anders markiert wurden. Bei `CREATE FUNCTION` oder `ALTER FUNCTION` können Markierungen durch Angabe von `PARALLEL SAFE`, `PARALLEL RESTRICTED` oder `PARALLEL UNSAFE` gesetzt werden. Bei `CREATE AGGREGATE` kann die Option `PARALLEL` mit dem entsprechenden Wert `SAFE`, `RESTRICTED` oder `UNSAFE` angegeben werden.

Funktionen und Aggregate müssen als `PARALLEL UNSAFE` markiert werden, wenn sie in die Datenbank schreiben, den Transaktionszustand verändern, außer durch Verwendung einer Subtransaktion zur Fehlerbehandlung, auf Sequenzen zugreifen oder persistente Änderungen an Einstellungen vornehmen. Ebenso müssen Funktionen als `PARALLEL RESTRICTED` markiert werden, wenn sie auf temporäre Tabellen, den Zustand der Client-Verbindung, Cursor, Prepared Statements oder sonstigen backend-lokalen Zustand zugreifen, den das System nicht zwischen Workern synchronisieren kann. Zum Beispiel sind `setseed` und `random` aus diesem letzten Grund parallel eingeschränkt.

Wenn eine Funktion allgemein als sicher markiert ist, obwohl sie eingeschränkt oder unsicher ist, oder wenn sie als eingeschränkt markiert ist, obwohl sie tatsächlich unsicher ist, kann sie bei Verwendung in einer parallelen Abfrage Fehler auslösen oder falsche Ergebnisse erzeugen. C-Funktionen könnten bei falscher Kennzeichnung theoretisch völlig undefiniertes Verhalten zeigen, weil das System sich nicht gegen beliebigen C-Code schützen kann. In den wahrscheinlichsten Fällen wird das Ergebnis aber nicht schlimmer sein als bei anderen Funktionen. Im Zweifel ist es vermutlich am besten, Funktionen als `UNSAFE` zu kennzeichnen.

Wenn eine innerhalb eines parallelen Workers ausgeführte Funktion Sperren erwirbt, die der Leader nicht hält, zum Beispiel durch Abfragen einer Tabelle, die in der Abfrage nicht referenziert wird, werden diese Sperren beim Beenden des Workers freigegeben und nicht erst am Ende der Transaktion. Wenn Sie eine Funktion schreiben, die dies tut, und dieser Verhaltensunterschied für Sie wichtig ist, kennzeichnen Sie solche Funktionen als `PARALLEL RESTRICTED`, damit sie nur im Leader ausgeführt werden.

Beachten Sie, dass der Query Planner nicht erwägt, die Auswertung parallel eingeschränkter Funktionen oder Aggregate in der Abfrage aufzuschieben, um einen besseren Plan zu erhalten. Wenn zum Beispiel eine `WHERE`-Klausel, die auf eine bestimmte Tabelle angewendet wird, parallel eingeschränkt ist, erwägt der Query Planner nicht, diese Tabelle im parallelen Teil des Plans zu scannen. In manchen Fällen wäre es möglich und vielleicht sogar effizient, den Scan dieser Tabelle in den parallelen Teil der Abfrage aufzunehmen und die Auswertung der `WHERE`-Klausel so aufzuschieben, dass sie oberhalb des `Gather`-Knotens geschieht. Der Planner tut dies jedoch nicht.
