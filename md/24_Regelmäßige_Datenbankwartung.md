# 24. Regelmäßige Datenbankwartung

PostgreSQL erfordert, wie jede Datenbanksoftware, dass bestimmte Aufgaben regelmäßig ausgeführt werden, um eine optimale Leistung zu erreichen. Die hier besprochenen Aufgaben sind notwendig, aber wiederkehrender Natur und lassen sich leicht mit Standardwerkzeugen wie `cron`-Skripten oder dem Windows-Taskplaner automatisieren. Es liegt in der Verantwortung des Datenbankadministrators, geeignete Skripte einzurichten und zu prüfen, ob sie erfolgreich ausgeführt werden.

Eine offensichtliche Wartungsaufgabe ist das regelmäßige Erstellen von Sicherungskopien der Daten. Ohne eine aktuelle Sicherung haben Sie nach einer Katastrophe (Plattenausfall, Feuer, versehentliches Löschen einer wichtigen Tabelle usw.) keine Möglichkeit zur Wiederherstellung. Die in PostgreSQL verfügbaren Sicherungs- und Wiederherstellungsmechanismen werden ausführlich in [Kapitel 25](25_Sicherung_und_Wiederherstellung.md) behandelt.

Die andere Hauptkategorie von Wartungsaufgaben ist das regelmäßige „Vacuuming“ der Datenbank. Diese Tätigkeit wird in [Abschnitt 24.1](24_Regelmäßige_Datenbankwartung.md#241-routinemäßiges-vacuuming) besprochen. Eng damit verbunden ist das Aktualisieren der Statistiken, die vom Query-Planner verwendet werden; dies wird in [Abschnitt 24.1.3](#2413-plannerstatistiken-aktualisieren) erläutert.

Eine weitere Aufgabe, die regelmäßige Aufmerksamkeit erfordern kann, ist die Verwaltung von Logdateien. Sie wird in [Abschnitt 24.3](#243-logdateiwartung) behandelt.

[`check_postgres`](https://bucardo.org/check_postgres/) ist verfügbar, um den Zustand der Datenbank zu überwachen und ungewöhnliche Zustände zu melden. `check_postgres` integriert sich in Nagios und MRTG, kann aber auch eigenständig ausgeführt werden.

PostgreSQL ist im Vergleich zu manchen anderen Datenbankmanagementsystemen wartungsarm. Dennoch trägt angemessene Aufmerksamkeit für diese Aufgaben wesentlich dazu bei, eine angenehme und produktive Arbeit mit dem System sicherzustellen.

## 24.1. Routinemäßiges Vacuuming

PostgreSQL-Datenbanken benötigen regelmäßige Wartung, die als Vacuuming bekannt ist. Für viele Installationen reicht es aus, das Vacuuming vom Autovacuum-Daemon ausführen zu lassen, der in [Abschnitt 24.1.6](#2416-der-autovacuumdaemon) beschrieben wird. Möglicherweise müssen Sie die dort beschriebenen Autovacuum-Parameter anpassen, um für Ihre Situation die besten Ergebnisse zu erhalten. Einige Datenbankadministratoren möchten die Aktivitäten des Daemons durch manuell verwaltete `VACUUM`-Befehle ergänzen oder ersetzen, die typischerweise nach Zeitplan durch `cron`- oder Taskplaner-Skripte ausgeführt werden. Um manuell verwaltetes Vacuuming richtig einzurichten, ist es wesentlich, die in den nächsten Unterabschnitten besprochenen Fragen zu verstehen. Administratoren, die sich auf Autovacuum verlassen, sollten dieses Material dennoch überfliegen, um Autovacuum besser zu verstehen und anzupassen.

### 24.1.1. Grundlagen des Vacuuming

Der PostgreSQL-Befehl `VACUUM` muss jede Tabelle aus mehreren Gründen regelmäßig verarbeiten:

1. Um Plattenplatz zurückzugewinnen oder wiederzuverwenden, der von aktualisierten oder gelöschten Zeilen belegt wird.

2. Um Datenstatistiken zu aktualisieren, die vom PostgreSQL-Query-Planner verwendet werden.

3. Um die Visibility Map zu aktualisieren, die Index-Only-Scans beschleunigt.

4. Um vor dem Verlust sehr alter Daten durch Transaction-ID-Wraparound oder Multixact-ID-Wraparound zu schützen.

Jeder dieser Gründe erfordert `VACUUM`-Operationen mit unterschiedlicher Häufigkeit und unterschiedlichem Umfang, wie in den folgenden Unterabschnitten erläutert.

Es gibt zwei Varianten von `VACUUM`: das Standard-`VACUUM` und `VACUUM FULL`. `VACUUM FULL` kann mehr Plattenplatz zurückgewinnen, läuft aber wesentlich langsamer. Außerdem kann die Standardform von `VACUUM` parallel zu produktiven Datenbankoperationen laufen. (Befehle wie `SELECT`, `INSERT`, `UPDATE` und `DELETE` funktionieren weiterhin normal, obwohl Sie die Definition einer Tabelle nicht mit Befehlen wie `ALTER TABLE` ändern können, während sie vacuumed wird.) `VACUUM FULL` benötigt eine `ACCESS EXCLUSIVE`-Sperre auf der bearbeiteten Tabelle und kann daher nicht parallel zu anderer Nutzung der Tabelle ausgeführt werden. Im Allgemeinen sollten Administratoren deshalb versuchen, Standard-`VACUUM` zu verwenden und `VACUUM FULL` zu vermeiden.

`VACUUM` erzeugt erheblichen E/A-Verkehr, was für andere aktive Sitzungen schlechte Leistung verursachen kann. Es gibt Konfigurationsparameter, die angepasst werden können, um die Auswirkungen von Hintergrund-Vacuuming auf die Leistung zu verringern; siehe Abschnitt 19.10.2.

### 24.1.2. Plattenplatz zurückgewinnen

In PostgreSQL entfernt ein `UPDATE` oder `DELETE` einer Zeile die alte Version der Zeile nicht sofort. Dieser Ansatz ist nötig, um die Vorteile der Multiversion Concurrency Control (MVCC, siehe [Kapitel 13](13_Nebenläufigkeitskontrolle.md)) zu erhalten: Die Zeilenversion darf nicht gelöscht werden, solange sie für andere Transaktionen noch potenziell sichtbar ist. Irgendwann ist eine veraltete oder gelöschte Zeilenversion jedoch für keine Transaktion mehr von Interesse. Der von ihr belegte Platz muss dann zur Wiederverwendung durch neue Zeilen zurückgewonnen werden, damit der Bedarf an Plattenplatz nicht unbegrenzt wächst. Dies geschieht durch Ausführen von `VACUUM`.

Die Standardform von `VACUUM` entfernt tote Zeilenversionen in Tabellen und Indizes und markiert den Platz als für künftige Wiederverwendung verfügbar. Sie gibt den Platz jedoch nicht an das Betriebssystem zurück, außer in dem Sonderfall, dass eine oder mehrere Seiten am Ende einer Tabelle vollständig frei werden und eine exklusive Tabellensperre leicht erhalten werden kann. Im Gegensatz dazu kompaktiert `VACUUM FULL` Tabellen aktiv, indem es eine vollständig neue Version der Tabellendatei ohne toten Platz schreibt. Das minimiert die Größe der Tabelle, kann aber lange dauern. Außerdem benötigt es bis zum Abschluss der Operation zusätzlichen Plattenplatz für die neue Kopie der Tabelle.

Das übliche Ziel routinemäßigen Vacuumings ist es, Standard-`VACUUM` oft genug auszuführen, um `VACUUM FULL` nicht zu benötigen. Der Autovacuum-Daemon versucht genau so zu arbeiten und gibt tatsächlich niemals `VACUUM FULL` aus. Bei diesem Ansatz geht es nicht darum, Tabellen auf ihrer Minimalgröße zu halten, sondern um einen stabilen Plattenplatzverbrauch: Jede Tabelle belegt Platz entsprechend ihrer Minimalgröße plus dem Platz, der zwischen zwei Vacuum-Läufen verbraucht wird. Obwohl `VACUUM FULL` verwendet werden kann, um eine Tabelle wieder auf ihre Minimalgröße zu schrumpfen und den Plattenplatz an das Betriebssystem zurückzugeben, ist das wenig sinnvoll, wenn die Tabelle in Zukunft einfach wieder wächst. Daher sind mäßig häufige Standard-`VACUUM`-Läufe für stark aktualisierte Tabellen besser als seltene `VACUUM FULL`-Läufe.

Einige Administratoren ziehen es vor, Vacuuming selbst zu planen, etwa indem sie die gesamte Arbeit nachts bei geringer Last erledigen. Die Schwierigkeit bei Vacuuming nach festem Zeitplan besteht darin, dass eine Tabelle bei einer unerwarteten Spitze der Aktualisierungsaktivität so stark aufblähen kann, dass `VACUUM FULL` wirklich nötig wird, um Platz zurückzugewinnen. Die Verwendung des Autovacuum-Daemons mildert dieses Problem, da der Daemon Vacuuming dynamisch als Reaktion auf Aktualisierungsaktivität plant. Es ist unklug, den Daemon vollständig zu deaktivieren, sofern Sie keine extrem vorhersehbare Arbeitslast haben. Ein möglicher Kompromiss besteht darin, die Parameter des Daemons so einzustellen, dass er nur auf ungewöhnlich starke Aktualisierungsaktivität reagiert und damit verhindert, dass die Dinge außer Kontrolle geraten, während geplante `VACUUM`-Läufe bei typischer Last den Großteil der Arbeit erledigen.

Für Installationen ohne Autovacuum besteht ein typischer Ansatz darin, einmal täglich während einer Phase geringer Nutzung ein datenbankweites `VACUUM` einzuplanen und dieses bei Bedarf durch häufigeres Vacuuming stark aktualisierter Tabellen zu ergänzen. (Einige Installationen mit extrem hohen Aktualisierungsraten vacuumieren ihre aktivsten Tabellen sogar alle paar Minuten.) Wenn Sie mehrere Datenbanken in einem Cluster haben, vergessen Sie nicht, jede einzelne zu vacuumieren; das Programm `vacuumdb` kann dabei hilfreich sein.

> Tipp: Einfaches `VACUUM` ist möglicherweise nicht ausreichend, wenn eine Tabelle als Folge massiver Aktualisierungs- oder Löschaktivität sehr viele tote Zeilenversionen enthält. Wenn Sie eine solche Tabelle haben und den übermäßigen Plattenplatz zurückgewinnen müssen, benötigen Sie `VACUUM FULL` oder alternativ `CLUSTER` oder eine der tabellenneuschreibenden Varianten von `ALTER TABLE`. Diese Befehle schreiben eine vollständig neue Kopie der Tabelle und bauen neue Indizes dafür. Alle diese Optionen erfordern eine `ACCESS EXCLUSIVE`-Sperre. Beachten Sie, dass sie außerdem temporär zusätzlichen Plattenplatz ungefähr in der Größe der Tabelle verwenden, da die alten Kopien der Tabelle und Indizes erst freigegeben werden können, wenn die neuen vollständig sind.

> Tipp: Wenn Sie eine Tabelle haben, deren gesamter Inhalt regelmäßig gelöscht wird, sollten Sie erwägen, dafür `TRUNCATE` statt `DELETE` gefolgt von `VACUUM` zu verwenden. `TRUNCATE` entfernt den gesamten Inhalt der Tabelle sofort, ohne dass ein anschließendes `VACUUM` oder `VACUUM FULL` nötig ist, um den nun ungenutzten Plattenplatz zurückzugewinnen. Der Nachteil ist, dass strenge MVCC-Semantik verletzt wird.

### 24.1.3. Plannerstatistiken aktualisieren

Der PostgreSQL-Query-Planner verlässt sich auf statistische Informationen über den Inhalt von Tabellen, um gute Pläne für Abfragen zu erzeugen. Diese Statistiken werden vom Befehl `ANALYZE` gesammelt, der allein oder als optionaler Schritt in `VACUUM` aufgerufen werden kann. Es ist wichtig, hinreichend genaue Statistiken zu haben, sonst können schlechte Planauswahlen die Datenbankleistung verschlechtern.

Der Autovacuum-Daemon gibt, sofern aktiviert, automatisch `ANALYZE`-Befehle aus, wenn sich der Inhalt einer Tabelle ausreichend geändert hat. Administratoren ziehen es jedoch möglicherweise vor, sich auf manuell geplante `ANALYZE`-Operationen zu verlassen, insbesondere wenn bekannt ist, dass Aktualisierungsaktivität an einer Tabelle die Statistiken der „interessanten“ Spalten nicht beeinflusst. Der Daemon plant `ANALYZE` streng als Funktion der Anzahl eingefügter oder aktualisierter Zeilen; er weiß nicht, ob dies zu bedeutenden statistischen Änderungen führt.

In Partitionen und Vererbungskindern geänderte Tupel lösen kein Analyze auf der Elterntabelle aus. Wenn die Elterntabelle leer ist oder selten geändert wird, wird sie möglicherweise nie von Autovacuum verarbeitet, und die Statistiken für den Vererbungsbaum insgesamt werden nicht gesammelt. Es ist notwendig, `ANALYZE` auf der Elterntabelle manuell auszuführen, um die Statistiken aktuell zu halten.

Wie beim Vacuuming zur Platzrückgewinnung sind häufige Statistikaktualisierungen für stark aktualisierte Tabellen nützlicher als für selten aktualisierte. Aber selbst bei einer stark aktualisierten Tabelle sind Statistikaktualisierungen möglicherweise nicht nötig, wenn sich die statistische Verteilung der Daten kaum ändert. Eine einfache Faustregel ist, darüber nachzudenken, wie stark sich die Minimal- und Maximalwerte der Spalten in der Tabelle ändern. Eine Zeitstempelspalte, die den Zeitpunkt der Zeilenaktualisierung enthält, wird zum Beispiel einen ständig steigenden Maximalwert haben, wenn Zeilen hinzugefügt und aktualisiert werden; eine solche Spalte benötigt wahrscheinlich häufigere Statistikaktualisierungen als etwa eine Spalte mit URLs für auf einer Website abgerufene Seiten. Die URL-Spalte kann genauso häufig Änderungen erhalten, aber die statistische Verteilung ihrer Werte ändert sich wahrscheinlich relativ langsam.

Es ist möglich, `ANALYZE` für bestimmte Tabellen und sogar nur für bestimmte Spalten einer Tabelle auszuführen. Dadurch besteht die Flexibilität, einige Statistiken häufiger zu aktualisieren als andere, wenn Ihre Anwendung dies erfordert. In der Praxis ist es jedoch normalerweise am besten, einfach die gesamte Datenbank zu analysieren, da dies eine schnelle Operation ist. `ANALYZE` verwendet eine statistisch zufällige Stichprobe der Zeilen einer Tabelle, statt jede einzelne Zeile zu lesen.

> Tipp: Obwohl eine spaltenweise Feinabstimmung der `ANALYZE`-Häufigkeit möglicherweise nicht sehr produktiv ist, kann es sich lohnen, den Detailgrad der von `ANALYZE` gesammelten Statistiken spaltenweise anzupassen. Spalten, die intensiv in `WHERE`-Klauseln verwendet werden und stark unregelmäßige Datenverteilungen haben, benötigen möglicherweise ein feiner aufgelöstes Datenhistogramm als andere Spalten. Siehe `ALTER TABLE SET STATISTICS`, oder ändern Sie den datenbankweiten Standard mit dem Konfigurationsparameter `default_statistics_target`.
>
> Außerdem sind standardmäßig nur begrenzte Informationen über die Selektivität von Funktionen verfügbar. Wenn Sie jedoch ein Statistikobjekt oder einen Ausdrucksindex erstellen, der einen Funktionsaufruf verwendet, werden nützliche Statistiken über die Funktion gesammelt, was Abfragepläne, die den Ausdrucksindex verwenden, stark verbessern kann.

> Tipp: Der Autovacuum-Daemon gibt keine `ANALYZE`-Befehle für Fremdtabellen aus, da er nicht feststellen kann, wie oft dies nützlich wäre. Wenn Ihre Abfragen Statistiken über Fremdtabellen für eine ordnungsgemäße Planung benötigen, ist es sinnvoll, manuell verwaltete `ANALYZE`-Befehle für diese Tabellen nach einem geeigneten Zeitplan auszuführen.

> Tipp: Der Autovacuum-Daemon gibt keine `ANALYZE`-Befehle für partitionierte Tabellen aus. Vererbungseltern werden nur analysiert, wenn die Elterntabelle selbst geändert wird; Änderungen an Kindtabellen lösen kein Autoanalyze auf der Elterntabelle aus. Wenn Ihre Abfragen Statistiken über Elterntabellen für eine ordnungsgemäße Planung benötigen, ist es notwendig, regelmäßig manuelles `ANALYZE` auf diesen Tabellen auszuführen, um die Statistiken aktuell zu halten.

### 24.1.4. Visibility Map aktualisieren

Vacuum pflegt für jede Tabelle eine Visibility Map, um zu verfolgen, welche Seiten nur Tupel enthalten, die bekanntermaßen für alle aktiven Transaktionen sichtbar sind (und für alle künftigen Transaktionen, bis die Seite erneut geändert wird). Dies hat zwei Zwecke. Erstens kann Vacuum selbst solche Seiten beim nächsten Lauf überspringen, da dort nichts zu bereinigen ist.

Zweitens erlaubt es PostgreSQL, manche Abfragen ausschließlich mithilfe des Index zu beantworten, ohne auf die zugrunde liegende Tabelle zuzugreifen. Da PostgreSQL-Indizes keine Tupelsichtbarkeitsinformationen enthalten, holt ein normaler Indexscan für jeden passenden Indexeintrag das Heap-Tupel, um zu prüfen, ob es von der aktuellen Transaktion gesehen werden soll. Ein Index-Only-Scan dagegen prüft zuerst die Visibility Map. Wenn bekannt ist, dass alle Tupel auf der Seite sichtbar sind, kann der Heap-Zugriff übersprungen werden. Dies ist besonders nützlich bei großen Datenmengen, bei denen die Visibility Map Plattenzugriffe verhindern kann. Die Visibility Map ist sehr viel kleiner als der Heap, sodass sie leicht im Cache gehalten werden kann, selbst wenn der Heap sehr groß ist.

### 24.1.5. Transaction-ID-Wraparound-Fehler verhindern

Die MVCC-Transaktionssemantik von PostgreSQL hängt davon ab, Transaction-ID-Nummern (XIDs) vergleichen zu können: Eine Zeilenversion mit einer Einfüge-XID, die größer ist als die XID der aktuellen Transaktion, liegt „in der Zukunft“ und sollte für die aktuelle Transaktion nicht sichtbar sein. Da Transaction-IDs aber eine begrenzte Größe haben (32 Bit), würde ein Cluster, der lange läuft (mehr als 4 Milliarden Transaktionen), einen Transaction-ID-Wraparound erleiden: Der XID-Zähler läuft auf null zurück, und plötzlich erscheinen Transaktionen, die in der Vergangenheit lagen, in der Zukunft. Das bedeutet, dass ihre Ergebnisse unsichtbar werden. Kurz gesagt: katastrophaler Datenverlust. (Tatsächlich sind die Daten noch vorhanden, aber das ist ein schwacher Trost, wenn man nicht mehr an sie herankommt.) Um dies zu vermeiden, muss jede Tabelle in jeder Datenbank mindestens einmal alle zwei Milliarden Transaktionen vacuumed werden.

Der Grund, warum regelmäßiges Vacuuming das Problem löst, besteht darin, dass `VACUUM` Zeilen als eingefroren markiert. Das zeigt an, dass sie von einer Transaktion eingefügt wurden, die so weit in der Vergangenheit committed wurde, dass die Effekte der einfügenden Transaktion sicher für alle aktuellen und zukünftigen Transaktionen sichtbar sind.

Normale XIDs werden mit Modulo-2^32-Arithmetik verglichen. Das bedeutet, dass es für jede normale XID zwei Milliarden XIDs gibt, die „älter“ sind, und zwei Milliarden, die „neuer“ sind; anders ausgedrückt ist der normale XID-Raum kreisförmig und hat keinen Endpunkt. Sobald also eine Zeilenversion mit einer bestimmten normalen XID erzeugt wurde, erscheint diese Zeilenversion für die nächsten zwei Milliarden Transaktionen „in der Vergangenheit“, ganz gleich, von welcher normalen XID die Rede ist. Wenn die Zeilenversion nach mehr als zwei Milliarden Transaktionen noch existiert, erscheint sie plötzlich in der Zukunft. Um dies zu verhindern, reserviert PostgreSQL eine besondere XID, `FrozenTransactionId`, die nicht den normalen XID-Vergleichsregeln folgt und immer als älter als jede normale XID betrachtet wird. Eingefrorene Zeilenversionen werden behandelt, als wäre die einfügende XID `FrozenTransactionId`, sodass sie unabhängig von Wraparound-Fragen für alle normalen Transaktionen „in der Vergangenheit“ erscheinen und solche Zeilenversionen gültig bleiben, bis sie gelöscht werden, egal wie lange das dauert.

> Hinweis: In PostgreSQL-Versionen vor 9.4 wurde das Einfrieren implementiert, indem die Einfüge-XID einer Zeile tatsächlich durch `FrozenTransactionId` ersetzt wurde, was in der Systemspalte `xmin` der Zeile sichtbar war. Neuere Versionen setzen nur ein Flag-Bit und bewahren das ursprüngliche `xmin` der Zeile für mögliche forensische Zwecke. In Datenbanken, die mit `pg_upgrade` von Versionen vor 9.4 aktualisiert wurden, können jedoch weiterhin Zeilen mit `xmin` gleich `FrozenTransactionId` (2) gefunden werden.
>
> Außerdem können Systemkataloge Zeilen mit `xmin` gleich `BootstrapTransactionId` (1) enthalten, was anzeigt, dass sie während der ersten Phase von `initdb` eingefügt wurden. Wie `FrozenTransactionId` wird diese besondere XID als älter als jede normale XID behandelt.

`vacuum_freeze_min_age` steuert, wie alt ein XID-Wert sein muss, bevor Zeilen mit dieser XID eingefroren werden. Eine Erhöhung dieser Einstellung kann unnötige Arbeit vermeiden, wenn die Zeilen, die andernfalls eingefroren würden, bald wieder geändert werden; eine Verringerung dieser Einstellung erhöht jedoch die Anzahl der Transaktionen, die vergehen können, bevor die Tabelle erneut vacuumed werden muss.

`VACUUM` verwendet die Visibility Map, um zu bestimmen, welche Seiten einer Tabelle gescannt werden müssen. Normalerweise überspringt es Seiten, die keine toten Zeilenversionen haben, selbst wenn diese Seiten noch Zeilenversionen mit alten XID-Werten enthalten könnten. Daher frieren normale `VACUUM`-Läufe nicht immer jede alte Zeilenversion in der Tabelle ein. Wenn das geschieht, muss `VACUUM` schließlich ein aggressives Vacuum ausführen, das alle dafür geeigneten nicht eingefrorenen XID- und MXID-Werte einfriert, einschließlich solcher von Seiten, die all-visible, aber nicht all-frozen sind.

Wenn eine Tabelle einen Rückstand von Seiten aufbaut, die all-visible, aber nicht all-frozen sind, kann ein normales Vacuum beschließen, überspringbare Seiten zu scannen, um sie einzufrieren. Dies verringert die Anzahl der Seiten, die das nächste aggressive Vacuum scannen muss. Diese Seiten werden als eagerly scanned pages bezeichnet. Eager Scanning kann so abgestimmt werden, dass versucht wird, mehr all-visible-Seiten einzufrieren, indem `vacuum_max_eager_freeze_failure_rate` erhöht wird. Selbst wenn Eager Scanning die Anzahl der all-visible, aber nicht all-frozen Seiten minimiert hat, benötigen die meisten Tabellen weiterhin regelmäßiges aggressives Vacuuming. Erfolgreich eager eingefrorene Seiten können jedoch während eines aggressiven Vacuum übersprungen werden, sodass Eager Freezing den Aufwand aggressiver Vacuums minimieren kann.

`vacuum_freeze_table_age` steuert, wann eine Tabelle aggressiv vacuumed wird. Alle all-visible, aber nicht all-frozen Seiten werden gescannt, wenn die Anzahl der seit dem letzten solchen Scan vergangenen Transaktionen größer ist als `vacuum_freeze_table_age` minus `vacuum_freeze_min_age`. Wenn `vacuum_freeze_table_age` auf 0 gesetzt wird, erzwingt dies, dass `VACUUM` immer seine aggressive Strategie verwendet.

Die maximale Zeit, die eine Tabelle ohne Vacuum bleiben kann, beträgt zwei Milliarden Transaktionen minus dem Wert von `vacuum_freeze_min_age` zum Zeitpunkt des letzten aggressiven Vacuum. Wenn sie länger unvacuumed bliebe, könnte Datenverlust die Folge sein. Um sicherzustellen, dass dies nicht geschieht, wird Autovacuum auf jeder Tabelle aufgerufen, die nicht eingefrorene Zeilen mit XIDs enthalten könnte, die älter sind als das vom Konfigurationsparameter `autovacuum_freeze_max_age` angegebene Alter. (Dies geschieht sogar dann, wenn Autovacuum deaktiviert ist.)

Das bedeutet, dass Autovacuum auf einer Tabelle, die sonst nicht vacuumed wird, ungefähr einmal alle `autovacuum_freeze_max_age` minus `vacuum_freeze_min_age` Transaktionen aufgerufen wird.

Für Tabellen, die regelmäßig zur Platzrückgewinnung vacuumed werden, ist dies von geringer Bedeutung. Für statische Tabellen (einschließlich Tabellen, die Einfügungen, aber keine Aktualisierungen oder Löschungen erhalten) besteht jedoch kein Bedarf an Vacuuming zur Platzrückgewinnung; daher kann es nützlich sein, das Intervall zwischen erzwungenen Autovacuums bei sehr großen statischen Tabellen zu maximieren. Offensichtlich kann man dies entweder durch Erhöhen von `autovacuum_freeze_max_age` oder durch Verringern von `vacuum_freeze_min_age` tun.

Das effektive Maximum für `vacuum_freeze_table_age` ist `0.95 * autovacuum_freeze_max_age`; eine höhere Einstellung wird auf dieses Maximum begrenzt. Ein Wert über `autovacuum_freeze_max_age` ergäbe keinen Sinn, da an diesem Punkt ohnehin ein Anti-Wraparound-Autovacuum ausgelöst würde, und der Faktor 0,95 lässt etwas Spielraum, um vorher ein manuelles `VACUUM` auszuführen. Als Faustregel sollte `vacuum_freeze_table_age` etwas unter `autovacuum_freeze_max_age` liegen, mit genügend Abstand, damit in diesem Fenster ein regelmäßig geplantes `VACUUM` oder ein durch normale Lösch- und Aktualisierungsaktivität ausgelöstes Autovacuum läuft. Eine zu nahe Einstellung kann zu Anti-Wraparound-Autovacuums führen, obwohl die Tabelle kürzlich zur Platzrückgewinnung vacuumed wurde; niedrigere Werte führen zu häufigerem aggressivem Vacuuming.

Der einzige Nachteil einer Erhöhung von `autovacuum_freeze_max_age` (und zusammen damit `vacuum_freeze_table_age`) besteht darin, dass die Unterverzeichnisse `pg_xact` und `pg_commit_ts` des Datenbankclusters mehr Platz benötigen, weil sie den Commit-Status und (falls `track_commit_timestamp` aktiviert ist) den Zeitstempel aller Transaktionen bis zurück zum Horizont von `autovacuum_freeze_max_age` speichern müssen. Der Commit-Status verwendet zwei Bits pro Transaktion. Wenn `autovacuum_freeze_max_age` also auf den maximal zulässigen Wert von zwei Milliarden gesetzt wird, kann erwartet werden, dass `pg_xact` auf etwa ein halbes Gigabyte und `pg_commit_ts` auf etwa 20 GB wächst. Wenn dies im Vergleich zur Gesamtgröße Ihrer Datenbank unerheblich ist, wird empfohlen, `autovacuum_freeze_max_age` auf den maximal zulässigen Wert zu setzen. Andernfalls wählen Sie den Wert danach, welchen Speicherverbrauch für `pg_xact` und `pg_commit_ts` Sie zulassen wollen. (Der Standardwert von 200 Millionen Transaktionen entspricht etwa 50 MB `pg_xact`-Speicher und etwa 2 GB `pg_commit_ts`-Speicher.)

Ein Nachteil der Verringerung von `vacuum_freeze_min_age` besteht darin, dass `VACUUM` dadurch nutzlose Arbeit leisten kann: Eine Zeilenversion einzufrieren ist Zeitverschwendung, wenn die Zeile kurz darauf geändert wird (wodurch sie eine neue XID erhält). Die Einstellung sollte also groß genug sein, dass Zeilen erst eingefroren werden, wenn sie wahrscheinlich nicht mehr geändert werden.

Um das Alter der ältesten nicht eingefrorenen XIDs in einer Datenbank zu verfolgen, speichert `VACUUM` XID-Statistiken in den Systemtabellen `pg_class` und `pg_database`. Insbesondere enthält die Spalte `relfrozenxid` der `pg_class`-Zeile einer Tabelle die älteste verbleibende nicht eingefrorene XID am Ende des letzten `VACUUM`, das `relfrozenxid` erfolgreich vorangebracht hat (typischerweise das letzte aggressive `VACUUM`). Entsprechend ist die Spalte `datfrozenxid` der `pg_database`-Zeile einer Datenbank eine untere Grenze für die nicht eingefrorenen XIDs, die in dieser Datenbank auftreten; sie ist einfach das Minimum der `relfrozenxid`-Werte pro Tabelle innerhalb der Datenbank. Eine bequeme Möglichkeit, diese Informationen zu untersuchen, sind Abfragen wie:

```sql
SELECT c.oid::regclass as table_name,
       greatest(age(c.relfrozenxid),age(t.relfrozenxid)) as age
FROM pg_class c
LEFT JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relkind IN ('r', 'm');

SELECT datname, age(datfrozenxid) FROM pg_database;
```

Die Spalte `age` misst die Anzahl der Transaktionen von der Cutoff-XID bis zur XID der aktuellen Transaktion.

> Tipp: Wenn der Parameter `VERBOSE` des Befehls `VACUUM` angegeben wird, gibt `VACUUM` verschiedene Statistiken über die Tabelle aus. Dazu gehören Informationen darüber, wie `relfrozenxid` und `relminmxid` vorangekommen sind, sowie die Anzahl neu eingefrorener Seiten. Dieselben Details erscheinen im Serverlog, wenn Autovacuum-Logging (gesteuert durch `log_autovacuum_min_duration`) über eine von Autovacuum ausgeführte `VACUUM`-Operation berichtet.

Während `VACUUM` hauptsächlich Seiten scannt, die seit dem letzten Vacuum geändert wurden, kann es außerdem einige all-visible, aber nicht all-frozen Seiten eager scannen, um sie einzufrieren. `relfrozenxid` wird jedoch nur vorangebracht, wenn jede Seite der Tabelle gescannt wurde, die nicht eingefrorene XIDs enthalten könnte. Dies geschieht, wenn `relfrozenxid` mehr als `vacuum_freeze_table_age` Transaktionen alt ist, wenn die Option `FREEZE` von `VACUUM` verwendet wird oder wenn alle Seiten, die noch nicht all-frozen sind, ohnehin vacuumed werden müssen, um tote Zeilenversionen zu entfernen. Wenn `VACUUM` jede Seite der Tabelle scannt, die nicht bereits all-frozen ist, sollte es `age(relfrozenxid)` auf einen Wert setzen, der nur etwas größer ist als die verwendete Einstellung `vacuum_freeze_min_age` (größer um die Anzahl der seit Beginn des `VACUUM` gestarteten Transaktionen). `VACUUM` setzt `relfrozenxid` auf die älteste XID, die in der Tabelle verbleibt, sodass der endgültige Wert viel jünger sein kann, als streng erforderlich wäre. Wenn kein `VACUUM`, das `relfrozenxid` voranbringt, auf der Tabelle ausgeführt wird, bis `autovacuum_freeze_max_age` erreicht ist, wird bald ein Autovacuum für die Tabelle erzwungen.

Wenn Autovacuum aus irgendeinem Grund alte XIDs nicht aus einer Tabelle entfernen kann, beginnt das System Warnmeldungen wie diese auszugeben, wenn die ältesten XIDs der Datenbank bis auf vierzig Millionen Transaktionen an den Wraparound-Punkt herankommen:

```text
WARNING: database "mydb" must be vacuumed within 39985967
 transactions
HINT: To avoid XID assignment failures, execute a database-wide
 VACUUM in that database.
```

(Ein manuelles `VACUUM` sollte das Problem beheben, wie der Hinweis vorschlägt; beachten Sie aber, dass `VACUUM` von einem Superuser ausgeführt werden sollte, da es sonst die Systemkataloge nicht verarbeiten kann, was verhindert, dass `datfrozenxid` der Datenbank vorangebracht wird.) Wenn diese Warnungen ignoriert werden, weigert sich das System, neue XIDs zuzuweisen, sobald weniger als drei Millionen Transaktionen bis zum Wraparound verbleiben:

```text
ERROR: database is not accepting commands that assign new XIDs to
 avoid wraparound data loss in database "mydb"
HINT: Execute a database-wide VACUUM in that database.
```

In diesem Zustand können bereits laufende Transaktionen fortgesetzt werden, aber es können nur noch schreibgeschützte Transaktionen gestartet werden. Operationen, die Datenbankeinträge ändern oder Relationen truncaten, schlagen fehl. Der Befehl `VACUUM` kann weiterhin normal ausgeführt werden. Entgegen mancher Empfehlungen aus früheren Versionen ist es nicht nötig oder wünschenswert, den Postmaster zu stoppen oder in den Single-User-Modus zu wechseln, um den normalen Betrieb wiederherzustellen. Folgen Sie stattdessen diesen Schritten:

1. Lösen Sie alte vorbereitete Transaktionen auf. Sie finden diese, indem Sie `pg_prepared_xacts` auf Zeilen prüfen, bei denen `age(transactionid)` groß ist. Solche Transaktionen sollten committed oder zurückgerollt werden.

2. Beenden Sie lang laufende offene Transaktionen. Sie finden diese, indem Sie `pg_stat_activity` auf Zeilen prüfen, bei denen `age(backend_xid)` oder `age(backend_xmin)` groß ist. Solche Transaktionen sollten committed oder zurückgerollt werden, oder die Sitzung kann mit `pg_terminate_backend` beendet werden.

3. Löschen Sie alte Replikationsslots. Verwenden Sie `pg_stat_replication`, um Slots zu finden, bei denen `age(xmin)` oder `age(catalog_xmin)` groß ist. In vielen Fällen wurden solche Slots für Replikation zu Servern erstellt, die nicht mehr existieren oder lange ausgefallen sind. Wenn Sie einen Slot für einen Server löschen, der noch existiert und möglicherweise noch versucht, sich mit diesem Slot zu verbinden, muss diese Replik möglicherweise neu aufgebaut werden.

4. Führen Sie `VACUUM` in der Zieldatenbank aus. Ein datenbankweites `VACUUM` ist am einfachsten; um die benötigte Zeit zu verringern, ist es auch möglich, manuelle `VACUUM`-Befehle auf den Tabellen auszugeben, bei denen `relminxid` am ältesten ist. Verwenden Sie in diesem Szenario nicht `VACUUM FULL`, weil es eine XID benötigt und daher fehlschlägt, außer im Superuser-Modus, wo es stattdessen eine XID verbraucht und damit das Risiko eines Transaction-ID-Wraparound erhöht. Verwenden Sie auch nicht `VACUUM FREEZE`, weil es mehr als die minimale Arbeit ausführt, die zur Wiederherstellung des normalen Betriebs nötig ist.

5. Sobald der normale Betrieb wiederhergestellt ist, stellen Sie sicher, dass Autovacuum in der Zieldatenbank richtig konfiguriert ist, um künftige Probleme zu vermeiden.

> Hinweis: In früheren Versionen war es manchmal nötig, den Postmaster zu stoppen und die Datenbank im Single-User-Modus zu vacuumieren. In typischen Szenarien ist dies nicht mehr nötig und sollte nach Möglichkeit vermieden werden, da es einen Systemausfall bedeutet. Es ist außerdem riskanter, weil es Transaction-ID-Wraparound-Schutzmechanismen deaktiviert, die Datenverlust verhindern sollen. Der einzige Grund, in diesem Szenario den Single-User-Modus zu verwenden, besteht darin, unnötige Tabellen mit `TRUNCATE` oder `DROP` zu entfernen, um sie nicht vacuumieren zu müssen. Die Sicherheitsmarge von drei Millionen Transaktionen existiert, damit der Administrator dies tun kann. Details zur Verwendung des Single-User-Modus finden Sie auf der Referenzseite zu `postgres`.

#### 24.1.5.1. Multixacts und Wraparound

Multixact-IDs werden verwendet, um Zeilensperren durch mehrere Transaktionen zu unterstützen. Da im Tupelkopf nur begrenzter Platz zum Speichern von Sperrinformationen vorhanden ist, werden diese Informationen als „Multiple Transaction ID“, kurz Multixact-ID, kodiert, wenn mehr als eine Transaktion gleichzeitig eine Zeile sperrt. Informationen darüber, welche Transaction-IDs in einer bestimmten Multixact-ID enthalten sind, werden separat im Unterverzeichnis `pg_multixact` gespeichert; im Feld `xmax` des Tupelkopfs erscheint nur die Multixact-ID. Wie Transaction-IDs sind Multixact-IDs als 32-Bit-Zähler und zugehöriger Speicher implementiert; all dies erfordert sorgfältige Alterungsverwaltung, Speicherbereinigung und Wraparound-Behandlung. Es gibt einen separaten Speicherbereich, der die Liste der Mitglieder jeder Multixact enthält; auch dieser verwendet einen 32-Bit-Zähler und muss ebenfalls verwaltet werden. Die in Tabelle 9.84 beschriebene Systemfunktion `pg_get_multixact_members()` kann verwendet werden, um die mit einer Multixact-ID verbundenen Transaction-IDs zu untersuchen.

Immer wenn `VACUUM` einen Teil einer Tabelle scannt, ersetzt es jede Multixact-ID, auf die es trifft und die älter als `vacuum_multixact_freeze_min_age` ist, durch einen anderen Wert. Das kann der Nullwert, eine einzelne Transaction-ID oder eine neuere Multixact-ID sein. Für jede Tabelle speichert `pg_class.relminmxid` die älteste mögliche Multixact-ID, die noch in irgendeinem Tupel dieser Tabelle erscheint. Wenn dieser Wert älter als `vacuum_multixact_freeze_table_age` ist, wird ein aggressives Vacuum erzwungen. Wie im vorigen Abschnitt erläutert, bedeutet ein aggressives Vacuum, dass nur Seiten übersprungen werden, die bekanntermaßen all-frozen sind. `mxid_age()` kann auf `pg_class.relminmxid` verwendet werden, um dessen Alter zu ermitteln.

Aggressive `VACUUM`-Läufe können unabhängig von ihrer Ursache garantiert `relminmxid` der Tabelle voranbringen. Schließlich können, wenn alle Tabellen in allen Datenbanken gescannt und ihre ältesten Multixact-Werte vorangebracht wurden, auf der Platte gespeicherte ältere Multixacts entfernt werden.

Als Sicherheitsvorkehrung erfolgt ein aggressiver Vacuum-Scan für jede Tabelle, deren Multixact-Alter größer ist als `autovacuum_multixact_freeze_max_age`. Wenn außerdem der von Multixact-Mitgliedern belegte Speicher etwa 10 GB überschreitet, erfolgen aggressive Vacuum-Scans häufiger für alle Tabellen, beginnend mit denen, die das älteste Multixact-Alter haben. Beide Arten aggressiver Scans erfolgen auch dann, wenn Autovacuum nominell deaktiviert ist. Der Speicherbereich für Mitglieder kann bis auf etwa 20 GB wachsen, bevor Wraparound erreicht wird.

Ähnlich wie im XID-Fall beginnt das System Warnmeldungen auszugeben, wenn Autovacuum alte MXIDs nicht aus einer Tabelle entfernen kann und die ältesten MXIDs der Datenbank bis auf vierzig Millionen Transaktionen an den Wraparound-Punkt herankommen. Und genau wie im XID-Fall weigert sich das System, neue MXIDs zu erzeugen, sobald weniger als drei Millionen bis zum Wraparound verbleiben, wenn diese Warnungen ignoriert werden.

Der normale Betrieb bei erschöpften MXIDs kann weitgehend auf dieselbe Weise wiederhergestellt werden wie bei erschöpften XIDs. Folgen Sie denselben Schritten wie im vorigen Abschnitt, mit folgenden Unterschieden:

1. Laufende Transaktionen und vorbereitete Transaktionen können ignoriert werden, wenn keine Chance besteht, dass sie in einer Multixact erscheinen.

2. MXID-Informationen sind in Systemsichten wie `pg_stat_activity` nicht direkt sichtbar; das Suchen nach alten XIDs ist jedoch weiterhin eine gute Möglichkeit zu bestimmen, welche Transaktionen MXID-Wraparound-Probleme verursachen.

3. XID-Erschöpfung blockiert alle schreibenden Transaktionen, MXID-Erschöpfung blockiert dagegen nur eine Teilmenge schreibender Transaktionen, nämlich solche, die Zeilensperren betreffen, die eine MXID erfordern.

### 24.1.6. Der Autovacuum-Daemon

PostgreSQL hat eine optionale, aber dringend empfohlene Funktion namens Autovacuum, deren Zweck darin besteht, die Ausführung von `VACUUM`- und `ANALYZE`-Befehlen zu automatisieren. Wenn aktiviert, prüft Autovacuum auf Tabellen, bei denen viele Tupel eingefügt, aktualisiert oder gelöscht wurden. Diese Prüfungen verwenden die Statistik-Sammelfunktion; daher kann Autovacuum nur verwendet werden, wenn `track_counts` auf `true` gesetzt ist. In der Standardkonfiguration ist Autovacuum aktiviert, und die zugehörigen Konfigurationsparameter sind passend gesetzt.

Der „Autovacuum-Daemon“ besteht tatsächlich aus mehreren Prozessen. Es gibt einen persistenten Daemon-Prozess namens Autovacuum Launcher, der dafür zuständig ist, Autovacuum-Worker-Prozesse für alle Datenbanken zu starten. Der Launcher verteilt die Arbeit über die Zeit und versucht, alle `autovacuum_naptime` Sekunden einen Worker innerhalb jeder Datenbank zu starten. (Wenn die Installation also `N` Datenbanken hat, wird alle `autovacuum_naptime/N` Sekunden ein neuer Worker gestartet.) Höchstens `autovacuum_max_workers` Worker-Prozesse dürfen gleichzeitig laufen. Wenn mehr als `autovacuum_max_workers` Datenbanken zu verarbeiten sind, wird die nächste Datenbank verarbeitet, sobald der erste Worker fertig ist. Jeder Worker-Prozess prüft jede Tabelle innerhalb seiner Datenbank und führt nach Bedarf `VACUUM` und/oder `ANALYZE` aus. `log_autovacuum_min_duration` kann gesetzt werden, um die Aktivität von Autovacuum-Workern zu überwachen.

Wenn mehrere große Tabellen innerhalb kurzer Zeit alle für Vacuuming infrage kommen, können alle Autovacuum-Worker für längere Zeit mit dem Vacuuming dieser Tabellen beschäftigt sein. Dadurch würden andere Tabellen und Datenbanken erst vacuumed, wenn ein Worker verfügbar wird. Es gibt keine Begrenzung dafür, wie viele Worker in einer einzelnen Datenbank sein können, aber Worker versuchen, Arbeit nicht zu wiederholen, die bereits von anderen Workern erledigt wurde. Beachten Sie, dass die Anzahl laufender Worker nicht gegen die Grenzen `max_connections` oder `superuser_reserved_connections` zählt.

Tabellen, deren Wert `relfrozenxid` mehr als `autovacuum_freeze_max_age` Transaktionen alt ist, werden immer vacuumed (dies gilt auch für Tabellen, deren Freeze-Max-Age über Speicherparameter geändert wurde; siehe unten). Andernfalls wird die Tabelle vacuumed, wenn die Anzahl der seit dem letzten `VACUUM` veralteten Tupel den „Vacuum-Schwellenwert“ überschreitet. Der Vacuum-Schwellenwert ist definiert als:

```text
vacuum threshold = Minimum(vacuum max threshold, vacuum base
 threshold + vacuum scale factor * number of tuples)
```

Dabei ist der Vacuum-Max-Schwellenwert `autovacuum_vacuum_max_threshold`, der Vacuum-Basisschwellenwert `autovacuum_vacuum_threshold`, der Vacuum-Skalierungsfaktor `autovacuum_vacuum_scale_factor` und die Anzahl der Tupel `pg_class.reltuples`.

Die Tabelle wird außerdem vacuumed, wenn die Anzahl der seit dem letzten Vacuum eingefügten Tupel den definierten Insert-Schwellenwert überschritten hat, der wie folgt definiert ist:

```text
vacuum insert threshold = vacuum base insert threshold + vacuum
 insert scale factor * number of tuples
```

Dabei ist der Vacuum-Insert-Basisschwellenwert `autovacuum_vacuum_insert_threshold`, und der Vacuum-Insert-Skalierungsfaktor ist `autovacuum_vacuum_insert_scale_factor`. Solche Vacuums können dazu führen, dass Teile der Tabelle als all-visible markiert werden, und außerdem Tupel einfrieren, was die Arbeit bei späteren Vacuums verringern kann. Für Tabellen, die `INSERT`-Operationen erhalten, aber keine oder fast keine `UPDATE`-/`DELETE`-Operationen, kann es vorteilhaft sein, `autovacuum_freeze_min_age` der Tabelle zu senken, da dadurch Tupel durch frühere Vacuums eingefroren werden können. Die Anzahl veralteter Tupel und die Anzahl eingefügter Tupel stammen aus dem kumulativen Statistiksystem; es handelt sich um eine letztlich konsistente Zählung, die durch jede `UPDATE`-, `DELETE`- und `INSERT`-Operation aktualisiert wird. Wenn der Wert `relfrozenxid` der Tabelle mehr als `vacuum_freeze_table_age` Transaktionen alt ist, wird ein aggressives Vacuum ausgeführt, um alte Tupel einzufrieren und `relfrozenxid` voranzubringen.

Für Analyze wird eine ähnliche Bedingung verwendet: Der Schwellenwert,

```text
analyze threshold = analyze base threshold + analyze scale factor *
 number of tuples
```

wird mit der Gesamtzahl der Tupel verglichen, die seit dem letzten `ANALYZE` eingefügt, aktualisiert oder gelöscht wurden.

Partitionierte Tabellen speichern selbst keine Tupel und werden folglich nicht von Autovacuum verarbeitet. (Autovacuum verarbeitet Tabellenpartitionen wie andere Tabellen.) Leider bedeutet dies, dass Autovacuum kein `ANALYZE` auf partitionierten Tabellen ausführt, was zu suboptimalen Plänen für Abfragen führen kann, die Statistiken partitionierter Tabellen verwenden. Sie können dieses Problem umgehen, indem Sie `ANALYZE` manuell auf partitionierten Tabellen ausführen, wenn sie zuerst befüllt werden, und erneut, wenn sich die Datenverteilung in ihren Partitionen erheblich ändert.

Temporäre Tabellen können von Autovacuum nicht erreicht werden. Daher sollten geeignete Vacuum- und Analyze-Operationen über SQL-Befehle der Sitzung ausgeführt werden.

Die Standardschwellenwerte und Skalierungsfaktoren stammen aus `postgresql.conf`, können aber (zusammen mit vielen anderen Autovacuum-Steuerparametern) pro Tabelle überschrieben werden; weitere Informationen finden Sie unter Speicherparameter. Wenn eine Einstellung über die Speicherparameter einer Tabelle geändert wurde, wird dieser Wert bei der Verarbeitung dieser Tabelle verwendet; andernfalls werden die globalen Einstellungen verwendet. Weitere Details zu den globalen Einstellungen finden Sie in Abschnitt 19.10.1.

Wenn mehrere Worker laufen, werden die Parameter für die Autovacuum-Kostenverzögerung (siehe Abschnitt 19.10.2) unter allen laufenden Workern „ausbalanciert“, sodass die gesamte E/A-Auswirkung auf das System unabhängig von der tatsächlich laufenden Anzahl von Workern gleich bleibt. Worker, die Tabellen verarbeiten, deren tabellenspezifische Speicherparameter `autovacuum_vacuum_cost_delay` oder `autovacuum_vacuum_cost_limit` gesetzt wurden, werden im Balancing-Algorithmus jedoch nicht berücksichtigt.

Autovacuum-Worker blockieren andere Befehle normalerweise nicht. Wenn ein Prozess versucht, eine Sperre zu erwerben, die mit der von Autovacuum gehaltenen `SHARE UPDATE EXCLUSIVE`-Sperre kollidiert, unterbricht der Sperrerwerb das Autovacuum. Für kollidierende Sperrmodi siehe Tabelle 13.2. Wenn das Autovacuum jedoch läuft, um Transaction-ID-Wraparound zu verhindern (d. h. wenn der Autovacuum-Abfragename in der Sicht `pg_stat_activity` mit `(to prevent wraparound)` endet), wird das Autovacuum nicht automatisch unterbrochen.

> Warnung: Regelmäßig ausgeführte Befehle, die Sperren erwerben, die mit einer `SHARE UPDATE EXCLUSIVE`-Sperre kollidieren (z. B. `ANALYZE`), können effektiv verhindern, dass Autovacuums jemals abgeschlossen werden.

## 24.2. Routinemäßiges Reindexing

In manchen Situationen lohnt es sich, Indizes regelmäßig mit dem Befehl `REINDEX` oder mit einer Reihe einzelner Neuaufbauschritte neu aufzubauen.

B-Baum-Indexseiten, die vollständig leer geworden sind, werden zur Wiederverwendung zurückgewonnen. Es besteht jedoch weiterhin die Möglichkeit ineffizienter Platzausnutzung: Wenn alle bis auf wenige Indexschlüssel auf einer Seite gelöscht wurden, bleibt die Seite dennoch zugewiesen. Daher führt ein Nutzungsmuster, bei dem in jedem Bereich die meisten, aber nicht alle Schlüssel schließlich gelöscht werden, zu schlechter Platzausnutzung. Für solche Nutzungsmuster wird regelmäßiges Reindexing empfohlen.

Das Potenzial für Bloat in Nicht-B-Baum-Indizes ist nicht gut erforscht. Es ist sinnvoll, bei Verwendung eines Nicht-B-Baum-Indextyps regelmäßig die physische Größe des Index zu überwachen.

Außerdem ist bei B-Baum-Indizes ein frisch erstellter Index etwas schneller zugänglich als einer, der viele Male aktualisiert wurde, weil logisch benachbarte Seiten in einem neu aufgebauten Index normalerweise auch physisch benachbart sind. (Diese Überlegung gilt nicht für Nicht-B-Baum-Indizes.) Es kann sich lohnen, regelmäßig nur zur Verbesserung der Zugriffsgeschwindigkeit neu zu indizieren.

`REINDEX` kann in allen Fällen sicher und einfach verwendet werden. Dieser Befehl erfordert standardmäßig eine `ACCESS EXCLUSIVE`-Sperre; daher ist es oft vorzuziehen, ihn mit seiner Option `CONCURRENTLY` auszuführen, die nur eine `SHARE UPDATE EXCLUSIVE`-Sperre benötigt.

## 24.3. Logdateiwartung

Es ist eine gute Idee, die Logausgabe des Datenbankservers irgendwo zu speichern, statt sie einfach über `/dev/null` zu verwerfen. Die Logausgabe ist bei der Diagnose von Problemen von unschätzbarem Wert.

> Hinweis: Das Serverlog kann sensible Informationen enthalten und muss geschützt werden, unabhängig davon, wie oder wo es gespeichert wird oder an welches Ziel es weitergeleitet wird. Zum Beispiel können manche DDL-Anweisungen Klartextpasswörter oder andere Authentifizierungsdetails enthalten. Auf der Ebene `ERROR` geloggte Anweisungen können den SQL-Quellcode von Anwendungen zeigen und außerdem Teile von Datenzeilen enthalten. Das Aufzeichnen von Daten, Ereignissen und zugehörigen Informationen ist die beabsichtigte Funktion dieser Einrichtung, daher ist dies kein Leak und kein Fehler. Stellen Sie sicher, dass Serverlogs nur für entsprechend autorisierte Personen sichtbar sind.

Logausgaben sind tendenziell umfangreich (besonders bei höheren Debug-Stufen), daher werden Sie sie nicht unbegrenzt speichern wollen. Sie müssen die Logdateien rotieren, sodass nach einer angemessenen Zeit neue Logdateien begonnen und alte entfernt werden.

Wenn Sie einfach `stderr` von `postgres` in eine Datei umleiten, erhalten Sie Logausgabe, aber die einzige Möglichkeit, die Logdatei zu kürzen, besteht darin, den Server zu stoppen und neu zu starten. Das kann akzeptabel sein, wenn Sie PostgreSQL in einer Entwicklungsumgebung verwenden, aber nur wenige Produktionsserver würden dieses Verhalten akzeptabel finden.

Ein besserer Ansatz ist es, die `stderr`-Ausgabe des Servers an ein Logrotationsprogramm zu senden. Es gibt eine eingebaute Logrotationsfunktion, die Sie verwenden können, indem Sie den Konfigurationsparameter `logging_collector` in `postgresql.conf` auf `true` setzen. Die Steuerparameter für dieses Programm werden in Abschnitt 19.8.1 beschrieben. Mit diesem Ansatz können Sie die Logdaten auch in maschinenlesbarem CSV-Format (comma-separated values) erfassen.

Alternativ möchten Sie vielleicht ein externes Logrotationsprogramm verwenden, wenn Sie bereits eines mit anderer Serversoftware einsetzen. Zum Beispiel kann das in der Apache-Distribution enthaltene Werkzeug `rotatelogs` mit PostgreSQL verwendet werden. Eine Möglichkeit dafür besteht darin, die `stderr`-Ausgabe des Servers an das gewünschte Programm zu pipen. Wenn Sie den Server mit `pg_ctl` starten, ist `stderr` bereits nach `stdout` umgeleitet, sodass Sie nur einen Pipe-Befehl benötigen, zum Beispiel:

```text
pg_ctl start | rotatelogs /var/log/pgsql_log 86400
```

Sie können diese Ansätze kombinieren, indem Sie `logrotate` so einrichten, dass es von PostgreSQLs eingebautem Logging Collector erzeugte Logdateien einsammelt. In diesem Fall definiert der Logging Collector die Namen und den Speicherort der Logdateien, während `logrotate` diese Dateien regelmäßig archiviert. Beim Einleiten der Logrotation muss `logrotate` sicherstellen, dass die Anwendung weitere Ausgaben an die neue Datei sendet. Dies geschieht üblicherweise mit einem `postrotate`-Skript, das ein `SIGHUP`-Signal an die Anwendung sendet, die daraufhin die Logdatei neu öffnet. In PostgreSQL können Sie stattdessen `pg_ctl` mit der Option `logrotate` ausführen. Wenn der Server diesen Befehl erhält, wechselt er je nach Logging-Konfiguration entweder zu einer neuen Logdatei oder öffnet die vorhandene Datei erneut (siehe Abschnitt 19.8.1).

> Hinweis: Bei Verwendung statischer Logdateinamen kann der Server die Logdatei möglicherweise nicht erneut öffnen, wenn die maximale Anzahl offener Dateien erreicht ist oder ein File-Table-Overflow auftritt. In diesem Fall werden Logmeldungen bis zu einer erfolgreichen Logrotation in die alte Logdatei geschrieben. Wenn `logrotate` so konfiguriert ist, dass es die Logdatei komprimiert und löscht, kann der Server die in diesem Zeitraum geloggten Meldungen verlieren. Um dieses Problem zu vermeiden, können Sie den Logging Collector so konfigurieren, dass er Logdateinamen dynamisch vergibt, und ein `prerotate`-Skript verwenden, das offene Logdateien ignoriert.

Ein weiterer produktionstauglicher Ansatz zur Verwaltung der Logausgabe besteht darin, sie an `syslog` zu senden und `syslog` die Dateirotation überlassen. Setzen Sie dazu den Konfigurationsparameter `log_destination` in `postgresql.conf` auf `syslog` (um nur nach `syslog` zu loggen). Dann können Sie dem `syslog`-Daemon ein `SIGHUP`-Signal senden, wann immer Sie erzwingen möchten, dass er eine neue Logdatei beginnt. Wenn Sie die Logrotation automatisieren möchten, kann das Programm `logrotate` so konfiguriert werden, dass es mit Logdateien von `syslog` arbeitet.

Auf vielen Systemen ist `syslog` jedoch nicht sehr zuverlässig, besonders bei großen Logmeldungen; es kann Meldungen kürzen oder verwerfen, gerade wenn Sie sie am dringendsten benötigen. Außerdem schreibt `syslog` unter Linux jede Meldung auf die Platte, was schlechte Leistung ergibt. (Sie können ein „-“ am Anfang des Dateinamens in der `syslog`-Konfigurationsdatei verwenden, um das Synchronisieren zu deaktivieren.)

Beachten Sie, dass alle oben beschriebenen Lösungen dafür sorgen, in konfigurierbaren Intervallen neue Logdateien zu beginnen, aber nicht das Löschen alter, nicht mehr nützlicher Logdateien übernehmen. Wahrscheinlich werden Sie einen Batch-Job einrichten wollen, der alte Logdateien regelmäßig löscht. Eine andere Möglichkeit besteht darin, das Rotationsprogramm so zu konfigurieren, dass alte Logdateien zyklisch überschrieben werden.

[`pgBadger`](https://pgbadger.darold.net/) ist ein externes Projekt, das anspruchsvolle Logdateianalyse bietet. [`check_postgres`](https://bucardo.org/check_postgres/) stellt Nagios-Warnungen bereit, wenn wichtige Meldungen in den Logdateien erscheinen, sowie die Erkennung vieler anderer außergewöhnlicher Zustände.
