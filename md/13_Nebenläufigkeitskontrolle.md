# 13. Nebenläufigkeitskontrolle

Dieses Kapitel beschreibt das Verhalten des PostgreSQL-Datenbanksystems, wenn zwei oder mehr Sitzungen gleichzeitig auf dieselben Daten zugreifen wollen. Ziel ist es in dieser Situation, allen Sitzungen effizienten Zugriff zu ermöglichen und zugleich strikte Datenintegrität zu bewahren. Jeder Entwickler von Datenbankanwendungen sollte mit den in diesem Kapitel behandelten Themen vertraut sein.

## 13.1. Einführung

PostgreSQL stellt Entwicklern eine umfangreiche Menge von Werkzeugen bereit, um nebenläufigen Zugriff auf Daten zu verwalten. Intern wird die Datenkonsistenz mithilfe eines Mehrversionenmodells aufrechterhalten, der Multiversion Concurrency Control (MVCC). Das bedeutet, dass jede SQL-Anweisung einen Snapshot der Daten sieht, also eine Datenbankversion aus einem bestimmten früheren Zeitpunkt, unabhängig vom aktuellen Zustand der zugrunde liegenden Daten. Dadurch wird verhindert, dass Anweisungen inkonsistente Daten sehen, die durch nebenläufige Transaktionen entstehen, die dieselben Datenzeilen aktualisieren; jede Datenbanksitzung erhält also Transaktionsisolation. Indem MVCC die Sperrmethoden traditioneller Datenbanksysteme vermeidet, minimiert es Sperrkonflikte und ermöglicht so angemessene Performance in Mehrbenutzerumgebungen.

Der Hauptvorteil des MVCC-Modells gegenüber Sperren besteht darin, dass bei MVCC Sperren, die zum Abfragen, also Lesen, von Daten erworben werden, nicht mit Sperren kollidieren, die zum Schreiben von Daten erworben werden. Lesen blockiert daher nie Schreiben, und Schreiben blockiert nie Lesen. PostgreSQL erhält diese Garantie sogar dann aufrecht, wenn es mit Serializable Snapshot Isolation (SSI) die strengste Stufe der Transaktionsisolation bereitstellt.

PostgreSQL bietet außerdem Sperrmöglichkeiten auf Tabellen- und Zeilenebene für Anwendungen, die normalerweise keine vollständige Transaktionsisolation benötigen und bestimmte Konfliktpunkte ausdrücklich selbst verwalten wollen. Die korrekte Verwendung von MVCC bietet im Allgemeinen jedoch bessere Performance als Sperren. Zusätzlich stellen anwendungsdefinierte Advisory Locks einen Mechanismus bereit, um Sperren zu erwerben, die nicht an eine einzelne Transaktion gebunden sind.

## 13.2. Transaktionsisolation

Der SQL-Standard definiert vier Stufen der Transaktionsisolation. Die strengste ist `Serializable`; sie wird im Standard sinngemäß so definiert, dass jede nebenläufige Ausführung einer Menge serialisierbarer Transaktionen garantiert denselben Effekt hat, als würden diese Transaktionen in irgendeiner Reihenfolge nacheinander ausgeführt. Die anderen drei Stufen werden über Phänomene definiert, die aus der Interaktion nebenläufiger Transaktionen entstehen und auf der jeweiligen Stufe nicht auftreten dürfen. Der Standard merkt an, dass wegen der Definition von `Serializable` keines dieser Phänomene auf dieser Stufe möglich ist. Das überrascht kaum: Wenn die Wirkung der Transaktionen so sein muss, als wären sie nacheinander ausgeführt worden, wie könnte man dann Phänomene sehen, die durch Interaktionen entstehen?

Die auf verschiedenen Stufen verbotenen Phänomene sind:

- Dirty Read: Eine Transaktion liest Daten, die von einer nebenläufigen, noch nicht bestätigten Transaktion geschrieben wurden.
- Nonrepeatable Read: Eine Transaktion liest Daten erneut, die sie zuvor gelesen hat, und stellt fest, dass diese Daten von einer anderen Transaktion geändert wurden, die seit dem ersten Lesen committed hat.
- Phantom Read: Eine Transaktion führt eine Abfrage erneut aus, die eine Menge von Zeilen mit einer Suchbedingung zurückgibt, und stellt fest, dass sich die Menge der Zeilen, die diese Bedingung erfüllen, durch eine andere kürzlich bestätigte Transaktion geändert hat.
- Serialization Anomaly: Das Ergebnis einer erfolgreich bestätigten Gruppe von Transaktionen ist mit keiner möglichen Reihenfolge vereinbar, in der diese Transaktionen nacheinander ausgeführt würden.

Die Transaktionsisolationsstufen des SQL-Standards und die in PostgreSQL implementierten Stufen sind in Tabelle 13.1 beschrieben.

**Tabelle 13.1. Transaktionsisolationsstufen**

| Isolationsstufe | Dirty Read | Nonrepeatable Read | Phantom Read | Serialization Anomaly |
| --- | --- | --- | --- | --- |
| Read Uncommitted | Erlaubt, aber nicht in PG | Möglich | Möglich | Möglich |
| Read Committed | Nicht möglich | Möglich | Möglich | Möglich |
| Repeatable Read | Nicht möglich | Nicht möglich | Erlaubt, aber nicht in PG | Möglich |
| Serializable | Nicht möglich | Nicht möglich | Nicht möglich | Nicht möglich |

In PostgreSQL können Sie alle vier standardisierten Transaktionsisolationsstufen anfordern, intern sind jedoch nur drei unterschiedliche Isolationsstufen implementiert; PostgreSQLs Modus `Read Uncommitted` verhält sich also wie `Read Committed`. Das ist die einzig sinnvolle Abbildung der Standard-Isolationsstufen auf PostgreSQLs MVCC-Architektur.

Die Tabelle zeigt außerdem, dass PostgreSQLs Implementierung von `Repeatable Read` keine Phantom Reads erlaubt. Das ist nach dem SQL-Standard zulässig, weil der Standard festlegt, welche Anomalien auf bestimmten Isolationsstufen nicht auftreten dürfen; stärkere Garantien sind erlaubt. Das Verhalten der verfügbaren Isolationsstufen wird in den folgenden Unterabschnitten genauer beschrieben.

Um die Transaktionsisolationsstufe einer Transaktion festzulegen, verwenden Sie den Befehl `SET TRANSACTION`.

> Wichtig: Einige PostgreSQL-Datentypen und -Funktionen haben besondere Regeln für ihr Transaktionsverhalten. Insbesondere sind Änderungen an einer Sequenz, und damit am Zähler einer mit `serial` deklarierten Spalte, sofort für alle anderen Transaktionen sichtbar und werden nicht zurückgerollt, wenn die Transaktion, die die Änderung vorgenommen hat, abbricht. Siehe [Abschnitt 9.17](09_Funktionen_und_Operatoren.md#917-sequenzmanipulationsfunktionen) und [Abschnitt 8.1.4](08_Datentypen.md#814-serialtypen).

### 13.2.1. Isolationsstufe Read Committed

`Read Committed` ist die Standard-Isolationsstufe in PostgreSQL. Wenn eine Transaktion diese Isolationsstufe verwendet, sieht eine `SELECT`-Abfrage ohne `FOR UPDATE`-/`SHARE`-Klausel nur Daten, die vor Beginn der Abfrage committed wurden. Sie sieht nie uncommitted Daten oder Änderungen, die von nebenläufigen Transaktionen während der Ausführung der Abfrage committed werden. Effektiv sieht eine `SELECT`-Abfrage einen Snapshot der Datenbank zu dem Zeitpunkt, an dem die Abfrage zu laufen beginnt. `SELECT` sieht jedoch die Effekte früherer Aktualisierungen innerhalb derselben Transaktion, auch wenn diese noch nicht committed sind. Beachten Sie außerdem, dass zwei aufeinanderfolgende `SELECT`-Befehle innerhalb einer einzigen Transaktion unterschiedliche Daten sehen können, wenn andere Transaktionen Änderungen committen, nachdem das erste `SELECT` gestartet wurde und bevor das zweite `SELECT` startet.

Die Befehle `UPDATE`, `DELETE`, `SELECT FOR UPDATE` und `SELECT FOR SHARE` verhalten sich beim Suchen nach Zielzeilen genauso wie `SELECT`: Sie finden nur Zielzeilen, die zum Startzeitpunkt des Befehls committed waren. Eine solche Zielzeile kann jedoch bereits von einer anderen nebenläufigen Transaktion aktualisiert, gelöscht oder gesperrt worden sein, wenn sie gefunden wird. In diesem Fall wartet der mögliche Aktualisierer darauf, dass die erste aktualisierende Transaktion committed oder zurückrollt, falls sie noch läuft. Wenn der erste Aktualisierer zurückrollt, werden seine Effekte aufgehoben und der zweite Aktualisierer kann mit dem Aktualisieren der ursprünglich gefundenen Zeile fortfahren. Wenn der erste Aktualisierer committed, ignoriert der zweite Aktualisierer die Zeile, falls sie vom ersten gelöscht wurde; andernfalls versucht er, seine Operation auf die aktualisierte Version der Zeile anzuwenden. Die Suchbedingung des Befehls, also die `WHERE`-Klausel, wird erneut ausgewertet, um zu prüfen, ob die aktualisierte Version der Zeile weiterhin zur Suchbedingung passt. Ist dies der Fall, fährt der zweite Aktualisierer mit seiner Operation auf der aktualisierten Version der Zeile fort. Bei `SELECT FOR UPDATE` und `SELECT FOR SHARE` bedeutet dies, dass die aktualisierte Version der Zeile gesperrt und an den Client zurückgegeben wird.

`INSERT` mit einer `ON CONFLICT DO UPDATE`-Klausel verhält sich ähnlich. Im Modus `Read Committed` wird jede zur Einfügung vorgeschlagene Zeile entweder eingefügt oder aktualisiert. Sofern keine unabhängigen Fehler auftreten, ist eines dieser beiden Ergebnisse garantiert. Wenn ein Konflikt aus einer anderen Transaktion stammt, deren Effekte für das `INSERT` noch nicht sichtbar sind, wirkt die `UPDATE`-Klausel auf diese Zeile, obwohl möglicherweise keine Version dieser Zeile auf herkömmliche Weise für den Befehl sichtbar ist.

Bei `INSERT` mit einer `ON CONFLICT DO NOTHING`-Klausel kann das Einfügen einer Zeile aufgrund des Ergebnisses einer anderen Transaktion unterbleiben, deren Effekte im `INSERT`-Snapshot nicht sichtbar sind. Auch dies gilt nur im Modus `Read Committed`.

`MERGE` erlaubt dem Benutzer, verschiedene Kombinationen von `INSERT`-, `UPDATE`- und `DELETE`-Unterbefehlen anzugeben. Ein `MERGE`-Befehl mit sowohl `INSERT`- als auch `UPDATE`-Unterbefehlen ähnelt einem `INSERT` mit `ON CONFLICT DO UPDATE`, garantiert aber nicht, dass entweder `INSERT` oder `UPDATE` ausgeführt wird. Wenn `MERGE` ein `UPDATE` oder `DELETE` versucht und die Zeile nebenläufig aktualisiert wird, die Join-Bedingung aber weiterhin für das aktuelle Ziel- und Quelltupel erfüllt ist, verhält sich `MERGE` wie die Befehle `UPDATE` oder `DELETE` und führt seine Aktion auf der aktualisierten Version der Zeile aus. Da `MERGE` jedoch mehrere Aktionen angeben kann und diese bedingt sein können, werden die Bedingungen für jede Aktion auf der aktualisierten Version der Zeile erneut ausgewertet, beginnend mit der ersten Aktion, selbst wenn die ursprünglich passende Aktion später in der Aktionsliste steht. Wenn die Zeile dagegen nebenläufig so aktualisiert wird, dass die Join-Bedingung nicht mehr erfüllt ist, wertet `MERGE` als Nächstes die Aktionen `NOT MATCHED BY SOURCE` und `NOT MATCHED [BY TARGET]` des Befehls aus und führt die erste erfolgreiche Aktion jeder Art aus. Wenn die Zeile nebenläufig gelöscht wird, wertet `MERGE` die Aktionen `NOT MATCHED [BY TARGET]` des Befehls aus und führt die erste erfolgreiche aus. Wenn `MERGE` ein `INSERT` versucht, ein eindeutiger Index vorhanden ist und nebenläufig eine doppelte Zeile eingefügt wird, wird ein Eindeutigkeitsverletzungsfehler ausgelöst; `MERGE` versucht nicht, solche Fehler durch erneutes Auswerten von `MATCHED`-Bedingungen zu vermeiden.

Aufgrund der obigen Regeln ist es möglich, dass ein aktualisierender Befehl einen inkonsistenten Snapshot sieht: Er kann die Effekte nebenläufiger Aktualisierungsbefehle auf denselben Zeilen sehen, die er selbst zu aktualisieren versucht, sieht aber nicht die Effekte dieser Befehle auf anderen Zeilen der Datenbank. Dieses Verhalten macht den Modus `Read Committed` ungeeignet für Befehle mit komplexen Suchbedingungen; für einfachere Fälle ist es dagegen genau richtig. Betrachten Sie zum Beispiel die Überweisung von 100 Dollar von einem Konto auf ein anderes:

```sql
BEGIN;
UPDATE accounts SET balance = balance + 100.00 WHERE acctnum = 12345;
UPDATE accounts SET balance = balance - 100.00 WHERE acctnum = 7534;
COMMIT;
```

Wenn eine andere Transaktion nebenläufig versucht, den Saldo des Kontos `7534` zu ändern, wollen wir offensichtlich, dass die zweite Anweisung mit der aktualisierten Version der Kontenzeile beginnt. Da jeder Befehl nur eine vorab festgelegte Zeile betrifft, erzeugt es keine problematische Inkonsistenz, wenn er die aktualisierte Version dieser Zeile sieht.

Komplexere Nutzung kann im Modus `Read Committed` unerwünschte Ergebnisse erzeugen. Betrachten Sie zum Beispiel einen `DELETE`-Befehl, der auf Daten arbeitet, die durch einen anderen Befehl zugleich in seine Einschränkungskriterien hinein- und herausbewegt werden. Nehmen Sie etwa an, `website` sei eine Tabelle mit zwei Zeilen, in denen `website.hits` den Werten `9` und `10` entspricht:

```sql
BEGIN;
UPDATE website SET hits = hits + 1;
-- run from another session: DELETE FROM website WHERE hits = 10;
COMMIT;
```

Das `DELETE` hat keine Wirkung, obwohl es vor und nach dem `UPDATE` eine Zeile mit `website.hits = 10` gibt. Das geschieht, weil der Zeilenwert `9` vor dem Update übersprungen wird und, sobald das `UPDATE` abgeschlossen ist und `DELETE` eine Sperre erhält, der neue Zeilenwert nicht mehr `10`, sondern `11` ist und damit nicht mehr zu den Kriterien passt.

Da der Modus `Read Committed` jeden Befehl mit einem neuen Snapshot beginnt, der alle bis zu diesem Zeitpunkt committed Transaktionen enthält, sehen spätere Befehle in derselben Transaktion in jedem Fall die Effekte der committed nebenläufigen Transaktion. Der strittige Punkt oben ist, ob ein einzelner Befehl eine absolut konsistente Sicht auf die Datenbank sieht.

Die teilweise Transaktionsisolation des Modus `Read Committed` reicht für viele Anwendungen aus, und dieser Modus ist schnell und einfach zu verwenden; er ist jedoch nicht für alle Fälle ausreichend. Anwendungen mit komplexen Abfragen und Aktualisierungen benötigen möglicherweise eine strengere konsistente Sicht auf die Datenbank, als `Read Committed` bietet.

### 13.2.2. Isolationsstufe Repeatable Read

Die Isolationsstufe `Repeatable Read` sieht nur Daten, die vor Beginn der Transaktion committed wurden. Sie sieht weder uncommitted Daten noch Änderungen, die von nebenläufigen Transaktionen während der Ausführung der Transaktion committed werden. Jede Abfrage sieht jedoch die Effekte früherer Aktualisierungen innerhalb derselben Transaktion, auch wenn diese noch nicht committed sind. Das ist eine stärkere Garantie, als der SQL-Standard für diese Isolationsstufe verlangt, und verhindert alle in Tabelle 13.1 beschriebenen Phänomene mit Ausnahme von Serialization Anomalies. Wie oben erwähnt, ist dies ausdrücklich vom Standard erlaubt, der nur die Mindestschutzmaßnahmen jeder Isolationsstufe beschreibt.

Diese Stufe unterscheidet sich von `Read Committed` dadurch, dass eine Abfrage in einer Repeatable-Read-Transaktion einen Snapshot vom Beginn der ersten Nicht-Transaktionssteuerungsanweisung der Transaktion sieht, nicht vom Beginn der aktuellen Anweisung innerhalb der Transaktion. Daher sehen aufeinanderfolgende `SELECT`-Befehle innerhalb einer einzigen Transaktion dieselben Daten; sie sehen also keine Änderungen, die von anderen Transaktionen committed wurden, nachdem ihre eigene Transaktion begonnen hat.

Anwendungen, die diese Stufe verwenden, müssen darauf vorbereitet sein, Transaktionen wegen Serialisierungsfehlern erneut auszuführen.

Die Befehle `UPDATE`, `DELETE`, `MERGE`, `SELECT FOR UPDATE` und `SELECT FOR SHARE` verhalten sich beim Suchen nach Zielzeilen genauso wie `SELECT`: Sie finden nur Zielzeilen, die zum Startzeitpunkt der Transaktion committed waren. Eine solche Zielzeile kann jedoch bereits von einer anderen nebenläufigen Transaktion aktualisiert, gelöscht oder gesperrt worden sein, wenn sie gefunden wird. In diesem Fall wartet die Repeatable-Read-Transaktion darauf, dass die erste aktualisierende Transaktion committed oder zurückrollt, falls sie noch läuft. Wenn der erste Aktualisierer zurückrollt, werden seine Effekte aufgehoben und die Repeatable-Read-Transaktion kann mit dem Aktualisieren der ursprünglich gefundenen Zeile fortfahren. Wenn der erste Aktualisierer jedoch committed und die Zeile tatsächlich aktualisiert oder gelöscht hat, nicht nur gesperrt, wird die Repeatable-Read-Transaktion mit dieser Meldung zurückgerollt:

```text
ERROR:  could not serialize access due to concurrent update
```

Der Grund ist, dass eine Repeatable-Read-Transaktion keine Zeilen verändern oder sperren kann, die von anderen Transaktionen geändert wurden, nachdem die Repeatable-Read-Transaktion begonnen hat.

Wenn eine Anwendung diese Fehlermeldung erhält, sollte sie die aktuelle Transaktion abbrechen und die gesamte Transaktion von Anfang an erneut versuchen. Beim zweiten Durchlauf sieht die Transaktion die zuvor committed Änderung als Teil ihrer anfänglichen Sicht auf die Datenbank, sodass es keinen logischen Konflikt gibt, die neue Version der Zeile als Ausgangspunkt für das Update der neuen Transaktion zu verwenden.

Beachten Sie, dass nur aktualisierende Transaktionen möglicherweise erneut versucht werden müssen; reine Lesetransaktionen haben niemals Serialisierungskonflikte.

Der Modus `Repeatable Read` garantiert streng, dass jede Transaktion eine vollständig stabile Sicht auf die Datenbank sieht. Diese Sicht muss jedoch nicht immer mit irgendeiner seriellen, also nacheinander ausgeführten, Ausführung nebenläufiger Transaktionen derselben Stufe konsistent sein. Beispielsweise kann sogar eine schreibgeschützte Transaktion auf dieser Stufe einen Steuerdatensatz sehen, der anzeigt, dass ein Batch abgeschlossen wurde, aber einen der Detaildatensätze nicht sehen, der logisch zu diesem Batch gehört, weil sie eine frühere Revision des Steuerdatensatzes gelesen hat. Versuche, Geschäftsregeln durch Transaktionen auf dieser Isolationsstufe durchzusetzen, funktionieren ohne sorgfältigen Einsatz expliziter Sperren zum Blockieren konfliktreicher Transaktionen wahrscheinlich nicht korrekt.

Die Isolationsstufe `Repeatable Read` wird mit einer Technik implementiert, die in der akademischen Datenbankliteratur und in einigen anderen Datenbankprodukten als Snapshot Isolation bekannt ist. Im Vergleich zu Systemen, die eine traditionelle Sperrtechnik verwenden, die Nebenläufigkeit reduziert, können Unterschiede in Verhalten und Performance beobachtet werden. Einige andere Systeme können `Repeatable Read` und Snapshot Isolation sogar als unterschiedliche Isolationsstufen mit unterschiedlichem Verhalten anbieten. Die erlaubten Phänomene, die diese beiden Techniken unterscheiden, wurden von Datenbankforschern erst formalisiert, nachdem der SQL-Standard entwickelt worden war, und liegen außerhalb des Umfangs dieses Handbuchs. Eine vollständige Behandlung finden Sie in [berenson95].

> Hinweis: Vor PostgreSQL-Version 9.1 bot eine Anforderung der Transaktionsisolationsstufe `Serializable` genau das hier beschriebene Verhalten. Um das alte Serializable-Verhalten beizubehalten, sollte heute `Repeatable Read` angefordert werden.

### 13.2.3. Isolationsstufe Serializable

Die Isolationsstufe `Serializable` bietet die strengste Transaktionsisolation. Diese Stufe emuliert für alle committed Transaktionen eine serielle Transaktionsausführung, als wären die Transaktionen nacheinander und nicht nebenläufig ausgeführt worden. Wie bei `Repeatable Read` müssen Anwendungen, die diese Stufe verwenden, jedoch darauf vorbereitet sein, Transaktionen wegen Serialisierungsfehlern erneut auszuführen. Tatsächlich funktioniert diese Isolationsstufe genau wie `Repeatable Read`, überwacht aber zusätzlich Bedingungen, die dazu führen könnten, dass die Ausführung einer nebenläufigen Menge serialisierbarer Transaktionen sich auf eine Weise verhält, die mit keiner möglichen seriellen Ausführung dieser Transaktionen konsistent ist. Diese Überwachung führt zu keiner zusätzlichen Blockierung gegenüber Repeatable Read, verursacht aber einen gewissen Overhead; das Erkennen von Bedingungen, die eine Serialization Anomaly verursachen könnten, löst einen Serialisierungsfehler aus.

Betrachten Sie als Beispiel eine Tabelle `mytab`, die anfangs Folgendes enthält:

```text
 class | value
-------+-------
     1 |    10
     1 |    20
     2 |   100
     2 |   200
```

Angenommen, die serialisierbare Transaktion A berechnet:

```sql
SELECT SUM(value) FROM mytab WHERE class = 1;
```

und fügt anschließend das Ergebnis `30` als Wert in einer neuen Zeile mit `class = 2` ein. Gleichzeitig berechnet die serialisierbare Transaktion B:

```sql
SELECT SUM(value) FROM mytab WHERE class = 2;
```

Sie erhält das Ergebnis `300`, das sie in einer neuen Zeile mit `class = 1` einfügt. Danach versuchen beide Transaktionen zu committen. Würde eine der beiden Transaktionen auf der Isolationsstufe `Repeatable Read` laufen, dürften beide committen; da es jedoch keine serielle Ausführungsreihenfolge gibt, die mit diesem Ergebnis konsistent ist, lässt die Verwendung serialisierbarer Transaktionen eine Transaktion committen und rollt die andere mit dieser Meldung zurück:

```text
ERROR:  could not serialize access due to read/write dependencies
        among transactions
```

Denn wenn A vor B ausgeführt worden wäre, hätte B die Summe `330` berechnet, nicht `300`; in der anderen Reihenfolge hätte entsprechend A eine andere Summe berechnet.

Wenn man sich auf serialisierbare Transaktionen verlässt, um Anomalien zu verhindern, ist wichtig, dass Daten, die aus einer permanenten Benutzertabelle gelesen wurden, erst dann als gültig betrachtet werden, wenn die Transaktion, die sie gelesen hat, erfolgreich committed wurde. Das gilt sogar für schreibgeschützte Transaktionen, mit der Ausnahme, dass Daten, die innerhalb einer deferrable schreibgeschützten Transaktion gelesen wurden, als gültig bekannt sind, sobald sie gelesen wurden, weil eine solche Transaktion vor dem Lesen wartet, bis sie einen Snapshot erhalten kann, der garantiert frei von solchen Problemen ist. In allen anderen Fällen dürfen Anwendungen nicht von Ergebnissen abhängen, die während einer Transaktion gelesen wurden, die später abbricht; stattdessen sollten sie die Transaktion erneut versuchen, bis sie erfolgreich ist.

Um echte Serialisierbarkeit zu garantieren, verwendet PostgreSQL Predicate Locking. Das bedeutet, dass Sperren gehalten werden, mit denen PostgreSQL erkennen kann, wann ein Schreibvorgang das Ergebnis eines früheren Lesevorgangs einer nebenläufigen Transaktion beeinflusst hätte, wenn er zuerst gelaufen wäre. In PostgreSQL verursachen diese Sperren keine Blockierung und können daher auch nicht zu einem Deadlock beitragen. Sie werden verwendet, um Abhängigkeiten zwischen nebenläufigen serialisierbaren Transaktionen zu identifizieren und zu markieren, die in bestimmten Kombinationen zu Serialization Anomalies führen können. Im Gegensatz dazu muss eine `Read Committed`- oder `Repeatable Read`-Transaktion, die Datenkonsistenz sicherstellen will, möglicherweise eine ganze Tabelle sperren, was andere Benutzer blockieren kann, die diese Tabelle verwenden wollen, oder sie verwendet `SELECT FOR UPDATE` oder `SELECT FOR SHARE`, was andere Transaktionen nicht nur blockieren, sondern auch Plattenzugriffe verursachen kann.

Predicate Locks in PostgreSQL beruhen, wie in den meisten anderen Datenbanksystemen, auf den Daten, auf die eine Transaktion tatsächlich zugreift. Sie erscheinen in der System-View `pg_locks` mit dem Modus `SIReadLock`. Welche konkreten Sperren während der Ausführung einer Abfrage erworben werden, hängt vom Plan ab, den die Abfrage verwendet. Mehrere feingranulare Sperren, etwa Tupelsperren, können während der Transaktion zu weniger, gröber granularen Sperren, etwa Seitensperren, zusammengefasst werden, um zu verhindern, dass der Speicher zur Nachverfolgung der Sperren erschöpft wird. Eine `READ ONLY`-Transaktion kann ihre `SIRead`-Sperren vor Abschluss freigeben, wenn sie erkennt, dass keine Konflikte mehr auftreten können, die zu einer Serialization Anomaly führen würden. Tatsächlich können `READ ONLY`-Transaktionen dies oft schon beim Start feststellen und vermeiden, überhaupt Predicate Locks zu nehmen. Wenn Sie ausdrücklich eine `SERIALIZABLE READ ONLY DEFERRABLE`-Transaktion anfordern, blockiert sie, bis sie dies feststellen kann. Das ist der einzige Fall, in dem Serializable-Transaktionen blockieren, Repeatable-Read-Transaktionen aber nicht. Andererseits müssen `SIRead`-Sperren häufig über den Commit der Transaktion hinaus behalten werden, bis überlappende Read-Write-Transaktionen abgeschlossen sind.

Die konsequente Verwendung serialisierbarer Transaktionen kann die Entwicklung vereinfachen. Die Garantie, dass jede Menge erfolgreich committed nebenläufiger serialisierbarer Transaktionen denselben Effekt hat, als wären sie nacheinander ausgeführt worden, bedeutet: Wenn Sie zeigen können, dass eine einzelne Transaktion, so wie sie geschrieben ist, allein ausgeführt das Richtige tut, können Sie darauf vertrauen, dass sie in jeder Mischung serialisierbarer Transaktionen ebenfalls das Richtige tut, ohne etwas darüber wissen zu müssen, was diese anderen Transaktionen tun könnten, oder dass sie nicht erfolgreich committed. Wichtig ist, dass eine Umgebung, die diese Technik verwendet, einen allgemeinen Umgang mit Serialisierungsfehlern besitzt, die immer mit dem SQLSTATE-Wert `40001` zurückgegeben werden. Es ist nämlich sehr schwer vorherzusagen, welche Transaktionen genau zu den Read/Write-Abhängigkeiten beitragen und zurückgerollt werden müssen, um Serialization Anomalies zu verhindern. Die Überwachung von Read/Write-Abhängigkeiten hat Kosten, ebenso der Neustart von Transaktionen, die mit einem Serialisierungsfehler beendet werden. Verglichen mit den Kosten und Blockierungen expliziter Sperren und von `SELECT FOR UPDATE` oder `SELECT FOR SHARE` sind serialisierbare Transaktionen in manchen Umgebungen jedoch die leistungsfähigste Wahl.

Obwohl PostgreSQLs Isolationsstufe `Serializable` nebenläufige Transaktionen nur committen lässt, wenn es beweisen kann, dass es eine serielle Ausführungsreihenfolge mit derselben Wirkung gibt, verhindert sie nicht immer Fehler, die bei echter serieller Ausführung nicht auftreten würden. Insbesondere können Eindeutigkeitsverletzungen auftreten, die durch Konflikte mit überlappenden serialisierbaren Transaktionen verursacht werden, selbst wenn ausdrücklich geprüft wurde, dass der Schlüssel nicht vorhanden ist, bevor versucht wird, ihn einzufügen. Das lässt sich vermeiden, indem sichergestellt wird, dass alle serialisierbaren Transaktionen, die potenziell konfliktreiche Schlüssel einfügen, ausdrücklich zuerst prüfen, ob sie dies dürfen. Stellen Sie sich zum Beispiel eine Anwendung vor, die den Benutzer nach einem neuen Schlüssel fragt und dann durch vorheriges Auswählen prüft, ob er noch nicht existiert, oder einen neuen Schlüssel erzeugt, indem sie den größten vorhandenen Schlüssel auswählt und eins addiert. Wenn einige serialisierbare Transaktionen neue Schlüssel direkt einfügen, ohne dieses Protokoll einzuhalten, können Eindeutigkeitsverletzungen gemeldet werden, selbst in Fällen, in denen sie bei serieller Ausführung der nebenläufigen Transaktionen nicht auftreten könnten.

Für optimale Performance, wenn Sie sich bei der Nebenläufigkeitskontrolle auf serialisierbare Transaktionen verlassen, sollten diese Punkte berücksichtigt werden:

- Deklarieren Sie Transaktionen nach Möglichkeit als `READ ONLY`.
- Begrenzen Sie die Anzahl aktiver Verbindungen, bei Bedarf mithilfe eines Connection Pools. Das ist immer ein wichtiger Performanceaspekt, kann aber in einem ausgelasteten System mit serialisierbaren Transaktionen besonders wichtig sein.
- Packen Sie nicht mehr in eine einzelne Transaktion, als für Integritätszwecke nötig ist.
- Lassen Sie Verbindungen nicht länger als nötig `idle in transaction` hängen. Der Konfigurationsparameter `idle_in_transaction_session_timeout` kann verwendet werden, um solche verbleibenden Sitzungen automatisch zu trennen.
- Entfernen Sie explizite Sperren, `SELECT FOR UPDATE` und `SELECT FOR SHARE`, wo sie aufgrund des automatisch durch serialisierbare Transaktionen bereitgestellten Schutzes nicht mehr nötig sind.
- Wenn das System wegen Speichermangels in der Predicate-Lock-Tabelle gezwungen ist, mehrere Predicate Locks auf Seitenebene zu einer einzigen Predicate Lock auf Relationsebene zusammenzufassen, kann die Rate der Serialisierungsfehler steigen. Sie können dies vermeiden, indem Sie `max_pred_locks_per_transaction`, `max_pred_locks_per_relation` und/oder `max_pred_locks_per_page` erhöhen.
- Ein sequentieller Scan erfordert immer eine Predicate Lock auf Relationsebene. Das kann zu einer erhöhten Rate von Serialisierungsfehlern führen. Es kann hilfreich sein, die Verwendung von Index-Scans zu fördern, indem `random_page_cost` gesenkt und/oder `cpu_tuple_cost` erhöht wird. Wägen Sie jede Verringerung von Transaktionsabbrüchen und Neustarts gegen eine mögliche Gesamtänderung der Abfrageausführungszeit ab.

Die Isolationsstufe `Serializable` wird mit einer Technik implementiert, die in der akademischen Datenbankliteratur als Serializable Snapshot Isolation bekannt ist. Sie baut auf Snapshot Isolation auf und ergänzt Prüfungen auf Serialisierungsanomalien. Im Vergleich zu anderen Systemen, die eine traditionelle Sperrtechnik verwenden, können einige Unterschiede in Verhalten und Performance auftreten. Ausführliche Informationen finden Sie in [ports12].

## 13.3. Explizites Sperren

PostgreSQL stellt verschiedene Sperrmodi bereit, um nebenläufigen Zugriff auf Daten in Tabellen zu steuern. Diese Modi können für anwendungsgesteuertes Sperren in Situationen verwendet werden, in denen MVCC nicht das gewünschte Verhalten liefert. Außerdem erwerben die meisten PostgreSQL-Befehle automatisch Sperren passender Modi, um sicherzustellen, dass referenzierte Tabellen während der Befehlsausführung nicht gelöscht oder auf inkompatible Weise geändert werden. Beispielsweise kann `TRUNCATE` nicht sicher nebenläufig mit anderen Operationen auf derselben Tabelle ausgeführt werden und erhält daher eine `ACCESS EXCLUSIVE`-Sperre auf der Tabelle, um dies durchzusetzen.

Um eine Liste der aktuell ausstehenden Sperren in einem Datenbankserver zu untersuchen, verwenden Sie die System-View `pg_locks`. Weitere Informationen zum Überwachen des Status des Lock-Manager-Subsystems finden Sie in [Kapitel 27](27_Datenbankaktivität_überwachen.md).

### 13.3.1. Sperren auf Tabellenebene

Die folgende Liste zeigt die verfügbaren Sperrmodi und die Kontexte, in denen PostgreSQL sie automatisch verwendet. Sie können jede dieser Sperren auch ausdrücklich mit dem Befehl `LOCK` erwerben. Denken Sie daran, dass alle diese Sperrmodi Sperren auf Tabellenebene sind, auch wenn der Name das Wort „row“ enthält; die Namen der Sperrmodi sind historisch. Bis zu einem gewissen Grad spiegeln die Namen die typische Verwendung der jeweiligen Sperrmodi wider, die Semantik ist aber überall dieselbe. Der einzige echte Unterschied zwischen einem Sperrmodus und einem anderen ist die Menge der Sperrmodi, mit denen er kollidiert (siehe Tabelle 13.2). Zwei Transaktionen können auf derselben Tabelle nicht gleichzeitig Sperren konfliktierender Modi halten. Eine Transaktion kollidiert jedoch nie mit sich selbst; sie könnte zum Beispiel eine `ACCESS EXCLUSIVE`-Sperre und später eine `ACCESS SHARE`-Sperre auf derselben Tabelle erwerben. Nicht konfliktierende Sperrmodi können von vielen Transaktionen gleichzeitig gehalten werden. Beachten Sie insbesondere, dass einige Sperrmodi selbstkonfliktierend sind, zum Beispiel kann eine `ACCESS EXCLUSIVE`-Sperre nicht von mehr als einer Transaktion gleichzeitig gehalten werden, während andere nicht selbstkonfliktierend sind, zum Beispiel kann eine `ACCESS SHARE`-Sperre von mehreren Transaktionen gehalten werden.

**Sperrmodi auf Tabellenebene**

- `ACCESS SHARE` (`AccessShareLock`): Konfligiert nur mit dem Sperrmodus `ACCESS EXCLUSIVE`. Der Befehl `SELECT` erwirbt eine Sperre dieses Modus auf referenzierten Tabellen. Allgemein erwirbt jede Abfrage, die eine Tabelle nur liest und nicht verändert, diesen Sperrmodus.
- `ROW SHARE` (`RowShareLock`): Konfligiert mit den Modi `EXCLUSIVE` und `ACCESS EXCLUSIVE`. Der Befehl `SELECT` erwirbt eine Sperre dieses Modus auf allen Tabellen, für die eine der Optionen `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` oder `FOR KEY SHARE` angegeben ist, zusätzlich zu `ACCESS SHARE`-Sperren auf anderen referenzierten Tabellen ohne ausdrückliche `FOR ...`-Sperroption.
- `ROW EXCLUSIVE` (`RowExclusiveLock`): Konfligiert mit `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` und `ACCESS EXCLUSIVE`. Die Befehle `UPDATE`, `DELETE`, `INSERT` und `MERGE` erwerben diesen Sperrmodus auf der Zieltabelle, zusätzlich zu `ACCESS SHARE`-Sperren auf anderen referenzierten Tabellen. Allgemein wird dieser Sperrmodus von jedem Befehl erworben, der Daten in einer Tabelle verändert.
- `SHARE UPDATE EXCLUSIVE` (`ShareUpdateExclusiveLock`): Konfligiert mit `SHARE UPDATE EXCLUSIVE`, `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` und `ACCESS EXCLUSIVE`. Dieser Modus schützt eine Tabelle vor nebenläufigen Schemaänderungen und `VACUUM`-Läufen. Er wird von `VACUUM` ohne `FULL`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`, `CREATE STATISTICS`, `COMMENT ON`, `REINDEX CONCURRENTLY` sowie bestimmten Varianten von `ALTER INDEX` und `ALTER TABLE` erworben; vollständige Details stehen in der Dokumentation dieser Befehle.
- `SHARE` (`ShareLock`): Konfligiert mit `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` und `ACCESS EXCLUSIVE`. Dieser Modus schützt eine Tabelle vor nebenläufigen Datenänderungen. Er wird von `CREATE INDEX` ohne `CONCURRENTLY` erworben.
- `SHARE ROW EXCLUSIVE` (`ShareRowExclusiveLock`): Konfligiert mit `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` und `ACCESS EXCLUSIVE`. Dieser Modus schützt eine Tabelle vor nebenläufigen Datenänderungen und ist selbstexklusiv, sodass ihn immer nur eine Sitzung halten kann. Er wird von `CREATE TRIGGER` und einigen Formen von `ALTER TABLE` erworben.
- `EXCLUSIVE` (`ExclusiveLock`): Konfligiert mit `ROW SHARE`, `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` und `ACCESS EXCLUSIVE`. Dieser Modus erlaubt nur nebenläufige `ACCESS SHARE`-Sperren; parallel zu einer Transaktion, die diesen Sperrmodus hält, sind also nur Lesezugriffe auf die Tabelle möglich. Er wird von `REFRESH MATERIALIZED VIEW CONCURRENTLY` erworben.
- `ACCESS EXCLUSIVE` (`AccessExclusiveLock`): Konfligiert mit Sperren aller Modi (`ACCESS SHARE`, `ROW SHARE`, `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` und `ACCESS EXCLUSIVE`). Dieser Modus garantiert, dass der Halter die einzige Transaktion ist, die in irgendeiner Weise auf die Tabelle zugreift. Er wird von den Befehlen `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`, `VACUUM FULL` und `REFRESH MATERIALIZED VIEW` ohne `CONCURRENTLY` erworben. Viele Formen von `ALTER INDEX` und `ALTER TABLE` erwerben ebenfalls eine Sperre dieser Stufe. Dies ist auch der Standardsperrmodus für `LOCK TABLE`-Anweisungen, die keinen Modus ausdrücklich angeben.

> Tipp: Nur eine `ACCESS EXCLUSIVE`-Sperre blockiert eine `SELECT`-Anweisung ohne `FOR UPDATE`/`SHARE`.

Nach dem Erwerb wird eine Sperre normalerweise bis zum Ende der Transaktion gehalten. Wenn eine Sperre jedoch nach dem Setzen eines Savepoints erworben wird, wird sie sofort freigegeben, wenn zu diesem Savepoint zurückgerollt wird. Das entspricht dem Prinzip, dass `ROLLBACK` alle Effekte der Befehle seit dem Savepoint aufhebt. Dasselbe gilt für Sperren, die innerhalb eines PL/pgSQL-Exception-Blocks erworben wurden: Ein Fehler, der den Block verlässt, gibt die darin erworbenen Sperren frei.

**Tabelle 13.2. Konfliktierende Sperrmodi**

| Angeforderter Sperrmodus | ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ACCESS SHARE |  |  |  |  |  |  |  | X |
| ROW SHARE |  |  |  |  |  |  | X | X |
| ROW EXCLUSIVE |  |  |  |  | X | X | X | X |
| SHARE UPDATE EXCLUSIVE |  |  |  | X | X | X | X | X |
| SHARE |  |  | X | X |  | X | X | X |
| SHARE ROW EXCLUSIVE |  |  | X | X | X | X | X | X |
| EXCLUSIVE |  | X | X | X | X | X | X | X |
| ACCESS EXCLUSIVE | X | X | X | X | X | X | X | X |

### 13.3.2. Sperren auf Zeilenebene

Zusätzlich zu Sperren auf Tabellenebene gibt es Sperren auf Zeilenebene. Sie sind unten mit den Kontexten aufgeführt, in denen PostgreSQL sie automatisch verwendet. Eine vollständige Tabelle der Konflikte zwischen Sperren auf Zeilenebene finden Sie in Tabelle 13.3. Beachten Sie, dass eine Transaktion konfliktierende Sperren auf derselben Zeile halten kann, sogar in unterschiedlichen Subtransaktionen; abgesehen davon können zwei Transaktionen niemals konfliktierende Sperren auf derselben Zeile halten. Sperren auf Zeilenebene beeinflussen das Abfragen von Daten nicht; sie blockieren nur Schreiber und Sperrer derselben Zeile. Sperren auf Zeilenebene werden wie Sperren auf Tabellenebene am Transaktionsende oder beim Zurückrollen zu einem Savepoint freigegeben.

**Sperrmodi auf Zeilenebene**

- `FOR UPDATE`: Bewirkt, dass die von der `SELECT`-Anweisung abgerufenen Zeilen wie für eine Aktualisierung gesperrt werden. Dadurch wird verhindert, dass andere Transaktionen sie sperren, ändern oder löschen, bis die aktuelle Transaktion endet. Andere Transaktionen, die für diese Zeilen `UPDATE`, `DELETE`, `SELECT FOR UPDATE`, `SELECT FOR NO KEY UPDATE`, `SELECT FOR SHARE` oder `SELECT FOR KEY SHARE` versuchen, werden blockiert, bis die aktuelle Transaktion endet. Umgekehrt wartet `SELECT FOR UPDATE` auf eine nebenläufige Transaktion, die einen dieser Befehle auf derselben Zeile ausgeführt hat, und sperrt und gibt dann die aktualisierte Zeile zurück, oder keine Zeile, wenn die Zeile gelöscht wurde. Innerhalb einer `REPEATABLE READ`- oder `SERIALIZABLE`-Transaktion wird jedoch ein Fehler ausgelöst, wenn sich eine zu sperrende Zeile seit Beginn der Transaktion geändert hat. Weitere Erläuterungen finden Sie in [Abschnitt 13.4](#134-datenkonsistenzprüfungen-auf-anwendungsebene). Der Sperrmodus `FOR UPDATE` wird außerdem von jedem `DELETE` auf einer Zeile erworben sowie von einem `UPDATE`, das die Werte bestimmter Spalten ändert. Derzeit sind dies Spalten, auf denen ein eindeutiger Index liegt, der in einem Fremdschlüssel verwendet werden kann; partielle Indizes und Ausdrucksindizes werden also nicht berücksichtigt. Dies kann sich künftig ändern.
- `FOR NO KEY UPDATE`: Verhält sich ähnlich wie `FOR UPDATE`, aber die erworbene Sperre ist schwächer: Sie blockiert keine `SELECT FOR KEY SHARE`-Befehle, die eine Sperre auf denselben Zeilen erwerben wollen. Dieser Sperrmodus wird außerdem von jedem `UPDATE` erworben, das keine `FOR UPDATE`-Sperre erwirbt.
- `FOR SHARE`: Verhält sich ähnlich wie `FOR NO KEY UPDATE`, erwirbt aber eine geteilte Sperre statt einer exklusiven Sperre auf jeder abgerufenen Zeile. Eine geteilte Sperre blockiert andere Transaktionen daran, `UPDATE`, `DELETE`, `SELECT FOR UPDATE` oder `SELECT FOR NO KEY UPDATE` auf diesen Zeilen auszuführen, verhindert aber nicht `SELECT FOR SHARE` oder `SELECT FOR KEY SHARE`.
- `FOR KEY SHARE`: Verhält sich ähnlich wie `FOR SHARE`, aber die Sperre ist schwächer: `SELECT FOR UPDATE` wird blockiert, `SELECT FOR NO KEY UPDATE` jedoch nicht. Eine Key-Share-Sperre blockiert andere Transaktionen daran, `DELETE` oder ein `UPDATE` auszuführen, das die Schlüsselwerte ändert, aber nicht andere `UPDATE`-Befehle; sie verhindert auch nicht `SELECT FOR NO KEY UPDATE`, `SELECT FOR SHARE` oder `SELECT FOR KEY SHARE`.

PostgreSQL speichert keine Informationen über geänderte Zeilen im Arbeitsspeicher, daher gibt es keine Begrenzung für die Anzahl gleichzeitig gesperrter Zeilen. Das Sperren einer Zeile kann jedoch einen Plattenschreibvorgang verursachen; zum Beispiel verändert `SELECT FOR UPDATE` ausgewählte Zeilen, um sie als gesperrt zu markieren, und führt daher zu Plattenschreibvorgängen.

**Tabelle 13.3. Konfliktierende Sperren auf Zeilenebene**

| Angeforderter Sperrmodus | FOR KEY SHARE | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE |
| --- | --- | --- | --- | --- |
| FOR KEY SHARE |  |  |  | X |
| FOR SHARE |  |  | X | X |
| FOR NO KEY UPDATE |  | X | X | X |
| FOR UPDATE | X | X | X | X |

### 13.3.3. Sperren auf Seitenebene

Zusätzlich zu Tabellen- und Zeilensperren werden geteilte/exklusive Sperren auf Seitenebene verwendet, um Lese-/Schreibzugriff auf Tabellenseiten im gemeinsamen Pufferpool zu steuern. Diese Sperren werden unmittelbar freigegeben, nachdem eine Zeile geholt oder aktualisiert wurde. Anwendungsentwickler müssen sich normalerweise nicht mit Sperren auf Seitenebene befassen; sie werden hier nur der Vollständigkeit halber erwähnt.

### 13.3.4. Deadlocks

Die Verwendung expliziter Sperren kann die Wahrscheinlichkeit von Deadlocks erhöhen, bei denen zwei oder mehr Transaktionen jeweils Sperren halten, die die andere benötigt. Wenn zum Beispiel Transaktion 1 eine exklusive Sperre auf Tabelle A erwirbt und dann versucht, eine exklusive Sperre auf Tabelle B zu erwerben, während Transaktion 2 bereits eine exklusive Sperre auf Tabelle B hält und nun eine exklusive Sperre auf Tabelle A benötigt, kann keine der beiden fortfahren. PostgreSQL erkennt Deadlock-Situationen automatisch und löst sie auf, indem es eine der beteiligten Transaktionen abbricht, sodass die andere oder die anderen abgeschlossen werden können. Welche Transaktion genau abgebrochen wird, ist schwer vorherzusagen und sollte nicht vorausgesetzt werden.

Beachten Sie, dass Deadlocks auch durch Sperren auf Zeilenebene entstehen können und somit auch dann auftreten können, wenn keine expliziten Sperren verwendet werden. Betrachten Sie den Fall, dass zwei nebenläufige Transaktionen eine Tabelle ändern. Die erste Transaktion führt aus:

```sql
UPDATE accounts SET balance = balance + 100.00 WHERE acctnum = 11111;
```

Dies erwirbt eine Sperre auf Zeilenebene auf der Zeile mit der angegebenen Kontonummer. Danach führt die zweite Transaktion aus:

```sql
UPDATE accounts SET balance = balance + 100.00 WHERE acctnum = 22222;
UPDATE accounts SET balance = balance - 100.00 WHERE acctnum = 11111;
```

Die erste `UPDATE`-Anweisung erwirbt erfolgreich eine Sperre auf Zeilenebene auf der angegebenen Zeile und aktualisiert diese Zeile daher erfolgreich. Die zweite `UPDATE`-Anweisung stellt jedoch fest, dass die Zeile, die sie aktualisieren will, bereits gesperrt ist, und wartet daher auf den Abschluss der Transaktion, die die Sperre erworben hat.

Transaktion zwei wartet nun darauf, dass Transaktion eins abgeschlossen wird, bevor sie fortfährt. Nun führt Transaktion eins aus:

```sql
UPDATE accounts SET balance = balance - 100.00 WHERE acctnum = 22222;
```

Transaktion eins versucht, eine Sperre auf Zeilenebene auf der angegebenen Zeile zu erwerben, kann das aber nicht: Transaktion zwei hält bereits eine solche Sperre. Also wartet sie auf den Abschluss von Transaktion zwei. Damit ist Transaktion eins durch Transaktion zwei blockiert, und Transaktion zwei ist durch Transaktion eins blockiert: eine Deadlock-Situation. PostgreSQL erkennt diese Situation und bricht eine der Transaktionen ab.

Die beste Verteidigung gegen Deadlocks besteht im Allgemeinen darin, sie zu vermeiden, indem sichergestellt wird, dass alle Anwendungen, die eine Datenbank verwenden, Sperren auf mehrere Objekte in konsistenter Reihenfolge erwerben. Im obigen Beispiel wäre kein Deadlock aufgetreten, wenn beide Transaktionen die Zeilen in derselben Reihenfolge aktualisiert hätten. Außerdem sollte sichergestellt werden, dass die erste Sperre, die in einer Transaktion auf einem Objekt erworben wird, der restriktivste Modus ist, der für dieses Objekt benötigt wird. Wenn dies nicht im Voraus geprüft werden kann, können Deadlocks zur Laufzeit behandelt werden, indem Transaktionen, die wegen Deadlocks abbrechen, erneut versucht werden.

Solange keine Deadlock-Situation erkannt wird, wartet eine Transaktion, die eine Tabellen- oder Zeilensperre sucht, unbegrenzt darauf, dass konfliktierende Sperren freigegeben werden. Daher ist es für Anwendungen eine schlechte Idee, Transaktionen über längere Zeit offen zu halten, etwa während sie auf Benutzereingaben warten.

### 13.3.5. Advisory Locks

PostgreSQL bietet eine Möglichkeit, Sperren mit anwendungsdefinierter Bedeutung zu erstellen. Diese heißen Advisory Locks, weil das System ihre Verwendung nicht erzwingt; es liegt an der Anwendung, sie korrekt zu verwenden. Advisory Locks können für Sperrstrategien nützlich sein, die schlecht zum MVCC-Modell passen. Eine häufige Verwendung von Advisory Locks besteht zum Beispiel darin, pessimistische Sperrstrategien zu emulieren, wie sie für sogenannte Flat-File-Datenverwaltungssysteme typisch sind. Ein in einer Tabelle gespeichertes Flag könnte zwar für denselben Zweck verwendet werden, Advisory Locks sind jedoch schneller, vermeiden Table Bloat und werden vom Server am Ende der Sitzung automatisch aufgeräumt.

Es gibt in PostgreSQL zwei Möglichkeiten, eine Advisory Lock zu erwerben: auf Sitzungsebene oder auf Transaktionsebene. Nach Erwerb auf Sitzungsebene wird eine Advisory Lock gehalten, bis sie ausdrücklich freigegeben wird oder die Sitzung endet. Anders als normale Sperranforderungen berücksichtigen Advisory-Lock-Anforderungen auf Sitzungsebene keine Transaktionssemantik: Eine Sperre, die während einer Transaktion erworben wurde, die später zurückgerollt wird, bleibt nach dem Rollback weiterhin gehalten; ebenso ist ein Unlock wirksam, selbst wenn die aufrufende Transaktion später fehlschlägt. Eine Sperre kann mehrfach von ihrem besitzenden Prozess erworben werden; für jede abgeschlossene Sperranforderung muss es eine entsprechende Unlock-Anforderung geben, bevor die Sperre tatsächlich freigegeben wird. Sperranforderungen auf Transaktionsebene verhalten sich dagegen eher wie normale Sperranforderungen: Sie werden am Ende der Transaktion automatisch freigegeben, und es gibt keine ausdrückliche Unlock-Operation. Dieses Verhalten ist für die kurzfristige Nutzung einer Advisory Lock oft bequemer als das Verhalten auf Sitzungsebene. Sperranforderungen auf Sitzungs- und Transaktionsebene für denselben Advisory-Lock-Bezeichner blockieren einander in der erwarteten Weise. Wenn eine Sitzung eine bestimmte Advisory Lock bereits hält, sind weitere Anforderungen durch sie immer erfolgreich, selbst wenn andere Sitzungen auf die Sperre warten; diese Aussage gilt unabhängig davon, ob die vorhandene Sperre und die neue Anforderung auf Sitzungs- oder Transaktionsebene liegen.

Wie bei allen Sperren in PostgreSQL findet sich eine vollständige Liste der derzeit von einer Sitzung gehaltenen Advisory Locks in der System-View `pg_locks`.

Sowohl Advisory Locks als auch reguläre Sperren werden in einem gemeinsamen Speicherpool abgelegt, dessen Größe durch die Konfigurationsvariablen `max_locks_per_transaction` und `max_connections` bestimmt wird. Es muss darauf geachtet werden, diesen Speicher nicht zu erschöpfen, sonst kann der Server überhaupt keine Sperren mehr gewähren. Dadurch entsteht eine Obergrenze für die Anzahl der vom Server gewährbaren Advisory Locks, typischerweise im Bereich von Zehntausenden bis Hunderttausenden, abhängig von der Serverkonfiguration.

In bestimmten Fällen bei der Verwendung von Advisory-Lock-Methoden, besonders in Abfragen mit ausdrücklicher Sortierung und `LIMIT`-Klauseln, muss sorgfältig kontrolliert werden, welche Sperren erworben werden, weil SQL-Ausdrücke in einer bestimmten Reihenfolge ausgewertet werden. Zum Beispiel:

```sql
SELECT pg_advisory_lock(id) FROM foo WHERE id = 12345; -- ok
SELECT pg_advisory_lock(id) FROM foo WHERE id > 12345 LIMIT 100; -- danger!
SELECT pg_advisory_lock(q.id) FROM
(
   SELECT id FROM foo WHERE id > 12345 LIMIT 100
) q; -- ok
```

In den obigen Abfragen ist die zweite Form gefährlich, weil nicht garantiert ist, dass `LIMIT` angewendet wird, bevor die Sperrfunktion ausgeführt wird. Dadurch könnten Sperren erworben werden, die die Anwendung nicht erwartet und daher nicht freigeben würde, bis die Sitzung endet. Aus Sicht der Anwendung wären solche Sperren hängengeblieben, obwohl sie weiterhin in `pg_locks` sichtbar sind.

Die Funktionen zum Manipulieren von Advisory Locks sind in [Abschnitt 9.28.10](09_Funktionen_und_Operatoren.md#92810-advisorylockfunktionen) beschrieben.

## 13.4. Datenkonsistenzprüfungen auf Anwendungsebene

Es ist sehr schwierig, Geschäftsregeln zur Datenintegrität mit `Read Committed`-Transaktionen durchzusetzen, weil sich die Sicht auf die Daten mit jeder Anweisung verschiebt und selbst eine einzelne Anweisung sich bei einem Schreibkonflikt möglicherweise nicht auf ihren Anweisungs-Snapshot beschränkt.

Eine `Repeatable Read`-Transaktion hat während ihrer gesamten Ausführung eine stabile Sicht auf die Daten. Bei der Verwendung von MVCC-Snapshots für Datenkonsistenzprüfungen gibt es jedoch ein subtiles Problem, das als Read/Write-Konflikt bekannt ist. Wenn eine Transaktion Daten schreibt und eine nebenläufige Transaktion versucht, dieselben Daten zu lesen, vor oder nach dem Schreiben, kann sie die Arbeit der anderen Transaktion nicht sehen. Der Leser scheint dann zuerst ausgeführt worden zu sein, unabhängig davon, welche Transaktion zuerst begann oder zuerst committed hat. Wenn es dabei bleibt, gibt es kein Problem. Wenn der Leser jedoch ebenfalls Daten schreibt, die von einer nebenläufigen Transaktion gelesen werden, gibt es nun eine Transaktion, die vor beiden zuvor erwähnten Transaktionen gelaufen zu sein scheint. Wenn die Transaktion, die scheinbar zuletzt ausgeführt wurde, tatsächlich zuerst committed, kann sehr leicht ein Zyklus im Graphen der Ausführungsreihenfolge der Transaktionen entstehen. Wenn ein solcher Zyklus entsteht, funktionieren Integritätsprüfungen ohne zusätzliche Hilfe nicht korrekt.

Wie in [Abschnitt 13.2.3](#1323-isolationsstufe-serializable) erwähnt, sind serialisierbare Transaktionen einfach `Repeatable Read`-Transaktionen, die eine nicht blockierende Überwachung gefährlicher Muster von Read/Write-Konflikten hinzufügen. Wenn ein Muster erkannt wird, das einen Zyklus in der scheinbaren Ausführungsreihenfolge verursachen könnte, wird eine der beteiligten Transaktionen zurückgerollt, um den Zyklus zu brechen.

### 13.4.1. Konsistenz mit serialisierbaren Transaktionen erzwingen

Wenn die Transaktionsisolationsstufe `Serializable` für alle Schreibvorgänge und für alle Lesevorgänge verwendet wird, die eine konsistente Sicht auf die Daten benötigen, ist kein weiterer Aufwand erforderlich, um Konsistenz sicherzustellen. Software aus anderen Umgebungen, die so geschrieben wurde, dass sie serialisierbare Transaktionen zur Sicherstellung von Konsistenz verwendet, sollte in dieser Hinsicht in PostgreSQL einfach funktionieren.

Bei dieser Technik vermeiden Sie unnötige Belastung für Anwendungsprogrammierer, wenn die Anwendungssoftware ein Framework verwendet, das Transaktionen automatisch erneut versucht, wenn sie mit einem Serialisierungsfehler zurückgerollt werden. Es kann sinnvoll sein, `default_transaction_isolation` auf `serializable` zu setzen. Außerdem ist es ratsam, Maßnahmen zu ergreifen, die sicherstellen, dass keine andere Transaktionsisolationsstufe verwendet wird, weder versehentlich noch zur Umgehung von Integritätsprüfungen, etwa durch Prüfungen der Transaktionsisolationsstufe in Triggern.

Performancevorschläge finden Sie in [Abschnitt 13.2.3](#1323-isolationsstufe-serializable).

> Warnung: Serialisierbare Transaktionen und Datenreplikation
>
> Diese Stufe des Integritätsschutzes mit serialisierbaren Transaktionen erstreckt sich noch nicht auf den Hot-Standby-Modus ([Abschnitt 26.4](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#264-hot-standby)) oder auf logische Replikate. Daher möchten Benutzer von Hot Standby oder logischer Replikation auf dem Primary möglicherweise `Repeatable Read` und explizite Sperren verwenden.

### 13.4.2. Konsistenz mit explizit blockierenden Sperren erzwingen

Wenn nicht serialisierbare Schreibvorgänge möglich sind, müssen Sie `SELECT FOR UPDATE`, `SELECT FOR SHARE` oder eine passende `LOCK TABLE`-Anweisung verwenden, um die aktuelle Gültigkeit einer Zeile sicherzustellen und sie vor nebenläufigen Aktualisierungen zu schützen. `SELECT FOR UPDATE` und `SELECT FOR SHARE` sperren nur die zurückgegebenen Zeilen gegen nebenläufige Aktualisierungen, während `LOCK TABLE` die ganze Tabelle sperrt. Dies sollte beim Portieren von Anwendungen aus anderen Umgebungen nach PostgreSQL berücksichtigt werden.

Für Umsteiger aus anderen Umgebungen ist außerdem wichtig, dass `SELECT FOR UPDATE` nicht sicherstellt, dass eine nebenläufige Transaktion eine ausgewählte Zeile nicht aktualisieren oder löschen wird. Um das in PostgreSQL sicherzustellen, müssen Sie die Zeile tatsächlich aktualisieren, selbst wenn keine Werte geändert werden müssen. `SELECT FOR UPDATE` blockiert andere Transaktionen vorübergehend daran, dieselbe Sperre zu erwerben oder ein `UPDATE` oder `DELETE` auszuführen, das die gesperrte Zeile betreffen würde. Sobald die Transaktion, die diese Sperre hält, jedoch committed oder zurückrollt, fährt eine blockierte Transaktion mit der konfliktierenden Operation fort, sofern während des Haltens der Sperre kein tatsächliches `UPDATE` der Zeile ausgeführt wurde.

Globale Gültigkeitsprüfungen erfordern unter nicht serialisierbarem MVCC zusätzliche Überlegung. Eine Bankanwendung könnte zum Beispiel prüfen wollen, ob die Summe aller Habenbuchungen in einer Tabelle der Summe der Sollbuchungen in einer anderen Tabelle entspricht, während beide Tabellen aktiv aktualisiert werden. Der Vergleich der Ergebnisse zweier aufeinanderfolgender `SELECT sum(...)`-Befehle funktioniert im Modus `Read Committed` nicht zuverlässig, weil die zweite Abfrage wahrscheinlich Ergebnisse von Transaktionen enthält, die von der ersten nicht gezählt wurden. Die beiden Summen in einer einzigen Repeatable-Read-Transaktion zu berechnen, liefert ein genaues Bild nur der Effekte von Transaktionen, die vor Beginn dieser Repeatable-Read-Transaktion committed wurden; man kann aber berechtigt fragen, ob die Antwort noch relevant ist, wenn sie geliefert wird. Wenn die Repeatable-Read-Transaktion selbst Änderungen vorgenommen hat, bevor sie die Konsistenzprüfung versucht, wird die Nützlichkeit der Prüfung noch fraglicher, weil sie nun einige, aber nicht alle Änderungen nach Transaktionsbeginn enthält. In solchen Fällen möchte eine vorsichtige Person möglicherweise alle für die Prüfung benötigten Tabellen sperren, um ein unbestreitbares Bild der aktuellen Realität zu erhalten. Eine Sperre im Modus `SHARE` oder höher garantiert, dass es in der gesperrten Tabelle außer denen der aktuellen Transaktion keine uncommitted Änderungen gibt.

Beachten Sie außerdem: Wenn man sich auf explizite Sperren verlässt, um nebenläufige Änderungen zu verhindern, sollte man entweder den Modus `Read Committed` verwenden oder im Modus `Repeatable Read` darauf achten, Sperren vor Abfragen zu erwerben. Eine von einer Repeatable-Read-Transaktion erworbene Sperre garantiert, dass keine anderen Transaktionen, die die Tabelle ändern, noch laufen. Wenn der von der Transaktion gesehene Snapshot jedoch vor dem Erwerb der Sperre liegt, könnte er vor einigen inzwischen committed Änderungen in der Tabelle liegen. Der Snapshot einer Repeatable-Read-Transaktion wird tatsächlich beim Start ihrer ersten Abfrage oder Datenänderungsanweisung (`SELECT`, `INSERT`, `UPDATE`, `DELETE` oder `MERGE`) eingefroren; daher ist es möglich, Sperren ausdrücklich zu erwerben, bevor der Snapshot eingefroren wird.

## 13.5. Behandlung von Serialisierungsfehlern

Sowohl die Isolationsstufen `Repeatable Read` als auch `Serializable` können Fehler erzeugen, die Serialisierungsanomalien verhindern sollen. Wie bereits gesagt, müssen Anwendungen, die diese Stufen verwenden, darauf vorbereitet sein, Transaktionen erneut zu versuchen, die wegen Serialisierungsfehlern fehlschlagen. Der Meldungstext eines solchen Fehlers variiert je nach genauer Situation, hat aber immer den SQLSTATE-Code `40001` (`serialization_failure`).

Es kann außerdem ratsam sein, Deadlock-Fehler erneut zu versuchen. Diese haben den SQLSTATE-Code `40P01` (`deadlock_detected`).

In manchen Fällen ist es auch angemessen, Eindeutigkeitsfehler erneut zu versuchen, die den SQLSTATE-Code `23505` (`unique_violation`) haben, sowie Ausschluss-Constraint-Fehler mit SQLSTATE-Code `23P01` (`exclusion_violation`). Wenn die Anwendung zum Beispiel einen neuen Wert für eine Primärschlüsselspalte auswählt, nachdem sie die derzeit gespeicherten Schlüssel geprüft hat, kann sie einen Eindeutigkeitsfehler erhalten, weil eine andere Anwendungsinstanz nebenläufig denselben neuen Schlüssel ausgewählt hat. Das ist effektiv ein Serialisierungsfehler, aber der Server erkennt ihn nicht als solchen, weil er die Verbindung zwischen dem eingefügten Wert und den vorherigen Lesevorgängen nicht „sehen“ kann. Es gibt außerdem einige Randfälle, in denen der Server einen Eindeutigkeits- oder Ausschluss-Constraint-Fehler ausgibt, obwohl er im Prinzip genug Informationen hätte, um zu erkennen, dass ein Serialisierungsproblem die zugrunde liegende Ursache ist. Während es empfehlenswert ist, `serialization_failure`-Fehler bedingungslos erneut zu versuchen, ist bei diesen anderen Fehlercodes mehr Sorgfalt nötig, weil sie dauerhafte Fehlerbedingungen statt vorübergehender Fehler darstellen können.

Wichtig ist, die vollständige Transaktion erneut zu versuchen, einschließlich aller Logik, die entscheidet, welches SQL ausgeführt wird und/oder welche Werte verwendet werden. Daher bietet PostgreSQL keine automatische Retry-Funktion, da dies nicht mit einer Korrektheitsgarantie möglich wäre.

Ein erneuter Transaktionsversuch garantiert nicht, dass die erneut versuchte Transaktion abgeschlossen wird; mehrere Versuche können nötig sein. Bei sehr hoher Konkurrenz ist es möglich, dass der Abschluss einer Transaktion viele Versuche erfordert. Wenn eine konfliktierende vorbereitete Transaktion beteiligt ist, ist möglicherweise kein Fortschritt möglich, bis die vorbereitete Transaktion committed oder zurückgerollt wird.

## 13.6. Einschränkungen und Besonderheiten

Einige DDL-Befehle, derzeit nur `TRUNCATE` und die tabellenumschreibenden Formen von `ALTER TABLE`, sind nicht MVCC-sicher. Das bedeutet, dass die Tabelle nach dem Commit des Truncation- oder Rewrite-Vorgangs für nebenläufige Transaktionen leer erscheint, wenn diese einen Snapshot verwenden, der vor dem Commit des DDL-Befehls aufgenommen wurde. Das ist nur für eine Transaktion ein Problem, die vor Beginn des DDL-Befehls noch nicht auf die betreffende Tabelle zugegriffen hat; jede Transaktion, die dies getan hat, würde mindestens eine `ACCESS SHARE`-Tabellensperre halten, die den DDL-Befehl blockieren würde, bis diese Transaktion abgeschlossen ist. Diese Befehle verursachen also keine sichtbare Inkonsistenz im Tabelleninhalt bei aufeinanderfolgenden Abfragen auf der Zieltabelle, können aber eine sichtbare Inkonsistenz zwischen dem Inhalt der Zieltabelle und anderen Tabellen in der Datenbank verursachen.

Unterstützung für die Transaktionsisolationsstufe `Serializable` wurde noch nicht für Hot-Standby-Replikationsziele hinzugefügt, die in [Abschnitt 26.4](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#264-hot-standby) beschrieben werden. Die strengste Isolationsstufe, die im Hot-Standby-Modus derzeit unterstützt wird, ist `Repeatable Read`. Während die Ausführung aller dauerhaften Datenbankschreibvorgänge innerhalb serialisierbarer Transaktionen auf dem Primary sicherstellt, dass alle Standbys schließlich einen konsistenten Zustand erreichen, kann eine auf dem Standby laufende Repeatable-Read-Transaktion manchmal einen vorübergehenden Zustand sehen, der mit keiner seriellen Ausführung der Transaktionen auf dem Primary konsistent ist.

Interner Zugriff auf die Systemkataloge erfolgt nicht mit der Isolationsstufe der aktuellen Transaktion. Das bedeutet, dass neu erstellte Datenbankobjekte wie Tabellen für nebenläufige `Repeatable Read`- und `Serializable`-Transaktionen sichtbar sind, obwohl die darin enthaltenen Zeilen es nicht sind. Dagegen sehen Abfragen, die die Systemkataloge ausdrücklich untersuchen, auf den höheren Isolationsstufen keine Zeilen, die nebenläufig erstellte Datenbankobjekte repräsentieren.

## 13.7. Sperren und Indizes

Obwohl PostgreSQL nicht blockierenden Lese-/Schreibzugriff auf Tabellendaten bietet, wird nicht blockierender Lese-/Schreibzugriff derzeit nicht für jede in PostgreSQL implementierte Indexzugriffsmethode angeboten. Die verschiedenen Indextypen werden wie folgt behandelt:

- B-tree-, GiST- und SP-GiST-Indizes: Für Lese-/Schreibzugriff werden kurzlebige geteilte/exklusive Sperren auf Seitenebene verwendet. Sperren werden unmittelbar freigegeben, nachdem jede Indexzeile geholt oder eingefügt wurde. Diese Indextypen bieten die höchste Nebenläufigkeit ohne Deadlock-Bedingungen.
- Hash-Indizes: Für Lese-/Schreibzugriff werden geteilte/exklusive Sperren auf Hash-Bucket-Ebene verwendet. Sperren werden freigegeben, nachdem der ganze Bucket verarbeitet wurde. Sperren auf Bucket-Ebene bieten bessere Nebenläufigkeit als Sperren auf Indexebene, aber Deadlocks sind möglich, weil die Sperren länger als eine einzelne Indexoperation gehalten werden.
- GIN-Indizes: Für Lese-/Schreibzugriff werden kurzlebige geteilte/exklusive Sperren auf Seitenebene verwendet. Sperren werden unmittelbar freigegeben, nachdem jede Indexzeile geholt oder eingefügt wurde. Beachten Sie aber, dass das Einfügen eines GIN-indizierten Werts normalerweise mehrere Indexschlüsseleinfügungen pro Zeile erzeugt, sodass GIN für das Einfügen eines einzelnen Werts beträchtliche Arbeit leisten kann.

Derzeit bieten B-tree-Indizes die beste Performance für nebenläufige Anwendungen. Da sie außerdem mehr Funktionen als Hash-Indizes haben, sind sie der empfohlene Indextyp für nebenläufige Anwendungen, die skalare Daten indizieren müssen. Bei nicht skalaren Daten sind B-trees nicht nützlich; stattdessen sollten GiST-, SP-GiST- oder GIN-Indizes verwendet werden.
