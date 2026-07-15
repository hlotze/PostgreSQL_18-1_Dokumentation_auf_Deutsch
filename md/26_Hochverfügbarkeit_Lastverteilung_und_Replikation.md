# 26. Hochverfügbarkeit, Lastverteilung und Replikation

Datenbankserver können zusammenarbeiten, damit ein zweiter Server bei Ausfall des Primärservers schnell übernehmen kann (Hochverfügbarkeit), oder damit mehrere Rechner dieselben Daten bereitstellen können (Lastverteilung). Im Idealfall könnten Datenbankserver nahtlos zusammenarbeiten. Webserver, die statische Webseiten ausliefern, lassen sich recht einfach kombinieren, indem Webanfragen auf mehrere Maschinen verteilt werden. Tatsächlich lassen sich auch schreibgeschützte Datenbankserver relativ einfach kombinieren. Leider haben die meisten Datenbankserver eine Mischung aus Lese- und Schreibanfragen, und Read/Write-Server sind deutlich schwerer zu kombinieren. Der Grund ist: Schreibgeschützte Daten müssen nur einmal auf jeden Server gebracht werden, aber ein Schreibvorgang auf einem beliebigen Server muss an alle Server weitergegeben werden, damit spätere Leseanfragen an diese Server konsistente Ergebnisse liefern.

Dieses Synchronisationsproblem ist die grundlegende Schwierigkeit beim Zusammenarbeiten von Servern. Da es keine einzelne Lösung gibt, die die Auswirkungen des Synchronisationsproblems für alle Anwendungsfälle beseitigt, gibt es mehrere Lösungen. Jede Lösung geht dieses Problem anders an und minimiert seine Auswirkungen für eine bestimmte Arbeitslast.

Einige Lösungen behandeln Synchronisation dadurch, dass nur ein Server die Daten ändern darf. Server, die Daten ändern können, werden Read/Write-, Master- oder Primärserver genannt. Server, die Änderungen des Primärservers nachvollziehen, heißen Standby- oder Sekundärserver. Ein Standby-Server, zu dem erst Verbindungen aufgebaut werden können, nachdem er zum Primärserver hochgestuft wurde, heißt Warm-Standby-Server; ein Standby, der Verbindungen annehmen und schreibgeschützte Abfragen bedienen kann, heißt Hot-Standby-Server.

Einige Lösungen sind synchron. Das bedeutet, dass eine datenändernde Transaktion erst dann als committet gilt, wenn alle Server die Transaktion committet haben. Dadurch wird garantiert, dass bei einem Failover keine Daten verloren gehen und alle lastverteilten Server konsistente Ergebnisse liefern, egal welcher Server abgefragt wird. Asynchrone Lösungen erlauben dagegen eine Verzögerung zwischen Commit und Weitergabe an die anderen Server. Dadurch können beim Wechsel auf einen Backup-Server einige Transaktionen verloren gehen, und lastverteilte Server können leicht veraltete Ergebnisse liefern. Asynchrone Kommunikation wird verwendet, wenn synchrone Kommunikation zu langsam wäre.

Lösungen lassen sich auch nach ihrer Granularität einteilen. Einige Lösungen können nur mit einem ganzen Datenbankserver umgehen, andere erlauben Steuerung auf Tabellen- oder Datenbankebene.

Bei jeder Auswahl muss Performance berücksichtigt werden. Üblicherweise gibt es einen Kompromiss zwischen Funktionalität und Performance. Eine vollständig synchrone Lösung über ein langsames Netzwerk kann die Performance beispielsweise um mehr als die Hälfte verringern, während eine asynchrone Lösung nur minimale Performance-Auswirkungen haben kann.

Der Rest dieses Abschnitts skizziert verschiedene Lösungen für Failover, Replikation und Lastverteilung.

## 26.1. Vergleich verschiedener Lösungen

**Shared-Disk-Failover**

Shared-Disk-Failover vermeidet Synchronisationsaufwand, indem nur eine Kopie der Datenbank existiert. Es verwendet ein einzelnes Platten-Array, das von mehreren Servern gemeinsam genutzt wird. Wenn der Hauptdatenbankserver ausfällt, kann der Standby-Server die Datenbank einhängen und starten, als würde er sich von einem Datenbankabsturz erholen. Das erlaubt schnellen Failover ohne Datenverlust.

Gemeinsam genutzte Hardwarefunktionen sind bei Netzwerkspeichergeräten üblich. Auch die Verwendung eines Netzwerkdateisystems ist möglich, wobei sorgfältig darauf geachtet werden muss, dass das Dateisystem vollständiges POSIX-Verhalten besitzt; siehe Abschnitt 18.2.2.1. Eine wesentliche Einschränkung dieser Methode ist, dass sowohl Primär- als auch Standby-Server funktionsunfähig sind, wenn das gemeinsame Platten-Array ausfällt oder beschädigt wird. Ein weiteres Problem ist, dass der Standby-Server niemals auf den gemeinsamen Speicher zugreifen darf, während der Primärserver läuft.

**Dateisystem- bzw. Blockgeräte-Replikation**

Eine abgewandelte Form gemeinsam genutzter Hardwarefunktionalität ist Dateisystemreplikation, bei der alle Änderungen an einem Dateisystem auf ein Dateisystem gespiegelt werden, das sich auf einem anderen Rechner befindet. Die einzige Einschränkung ist, dass die Spiegelung so erfolgen muss, dass der Standby-Server eine konsistente Kopie des Dateisystems besitzt. Insbesondere müssen Schreibvorgänge auf dem Standby in derselben Reihenfolge erfolgen wie auf dem Primary. DRBD ist eine verbreitete Lösung für Dateisystemreplikation unter Linux.

**Write-Ahead-Log-Shipping**

Warm- und Hot-Standby-Server können aktuell gehalten werden, indem sie einen Strom von Write-Ahead-Log-Datensätzen (WAL) lesen. Wenn der Hauptserver ausfällt, enthält der Standby fast alle Daten des Hauptservers und kann schnell zum neuen primären Datenbankserver gemacht werden. Dies kann synchron oder asynchron erfolgen und ist nur für den gesamten Datenbankserver möglich.

Ein Standby-Server kann mit dateibasiertem Log-Shipping ([Abschnitt 26.2](#262-logshippingstandbyserver)) oder Streaming-Replikation (siehe [Abschnitt 26.2.5](#2625-streamingreplikation)) oder einer Kombination aus beidem implementiert werden. Informationen zu Hot Standby finden Sie in [Abschnitt 26.4](#264-hot-standby).

**Logische Replikation**

Logische Replikation erlaubt einem Datenbankserver, einen Strom von Datenänderungen an einen anderen Server zu senden. Die logische Replikation von PostgreSQL konstruiert aus dem WAL einen Strom logischer Datenänderungen. Logische Replikation erlaubt die Replikation von Datenänderungen auf Tabellenebene. Außerdem kann ein Server, der eigene Änderungen veröffentlicht, auch Änderungen von einem anderen Server abonnieren, sodass Daten in mehrere Richtungen fließen können. Weitere Informationen zur logischen Replikation finden Sie in [Kapitel 29](29_Logische_Replikation.md). Über die Schnittstelle für logische Decodierung ([Kapitel 47](47_Logische_Decodierung.md)) können Erweiterungen von Drittanbietern ähnliche Funktionen bereitstellen.

**Trigger-basierte Primary-Standby-Replikation**

Eine triggerbasierte Replikationskonfiguration leitet Datenänderungsabfragen typischerweise an einen bestimmten Primärserver. Auf Tabellenebene sendet der Primärserver Datenänderungen üblicherweise asynchron an die Standby-Server. Standby-Server können Abfragen beantworten, während der Primary läuft, und können lokale Datenänderungen oder Schreibaktivität erlauben. Diese Replikationsform wird häufig verwendet, um große Analyse- oder Data-Warehouse-Abfragen auszulagern.

Slony-I ist ein Beispiel für diese Art von Replikation, mit Tabellen-Granularität und Unterstützung für mehrere Standby-Server. Weil der Standby-Server asynchron und in Batches aktualisiert wird, ist bei einem Failover Datenverlust möglich.

**SQL-basierte Replikations-Middleware**

Bei SQL-basierter Replikations-Middleware fängt ein Programm jede SQL-Abfrage ab und sendet sie an einen oder alle Server. Jeder Server arbeitet unabhängig. Read/Write-Abfragen müssen an alle Server gesendet werden, damit jeder Server alle Änderungen erhält. Schreibgeschützte Abfragen können dagegen nur an einen Server gesendet werden, sodass die Leselast auf die Server verteilt werden kann.

Wenn Abfragen einfach unverändert verbreitet werden, können Funktionen wie `random()`, `CURRENT_TIMESTAMP` und Sequenzen auf verschiedenen Servern unterschiedliche Werte haben. Das liegt daran, dass jeder Server unabhängig arbeitet und SQL-Abfragen statt tatsächlicher Datenänderungen verbreitet werden. Wenn das nicht akzeptabel ist, muss entweder die Middleware oder die Anwendung solche Werte aus einer einzigen Quelle bestimmen und diese Werte dann in Schreibabfragen verwenden. Außerdem muss darauf geachtet werden, dass alle Transaktionen auf allen Servern entweder committen oder abbrechen, möglicherweise mit Two-Phase Commit (`PREPARE TRANSACTION` und `COMMIT PREPARED`). Pgpool-II und Continuent Tungsten sind Beispiele für diese Art von Replikation.

**Asynchrone Multimaster-Replikation**

Bei Servern, die nicht regelmäßig verbunden sind oder langsame Kommunikationsstrecken haben, etwa Laptops oder entfernte Server, ist es schwierig, Daten zwischen Servern konsistent zu halten. Bei asynchroner Multimaster-Replikation arbeitet jeder Server unabhängig und kommuniziert regelmäßig mit den anderen Servern, um widersprüchliche Transaktionen zu erkennen. Die Konflikte können von Benutzern oder durch Konfliktlösungsregeln aufgelöst werden. Bucardo ist ein Beispiel für diese Art von Replikation.

**Synchrone Multimaster-Replikation**

Bei synchroner Multimaster-Replikation kann jeder Server Schreibanforderungen annehmen, und geänderte Daten werden vor dem Commit jeder Transaktion vom ursprünglichen Server an jeden anderen Server übertragen. Intensive Schreibaktivität kann übermäßige Sperren und Commit-Verzögerungen verursachen, was zu schlechter Performance führt. Leseanfragen können an jeden Server gesendet werden. Einige Implementierungen verwenden Shared Disk, um den Kommunikationsaufwand zu verringern. Synchrone Multimaster-Replikation eignet sich am besten für überwiegend lesende Arbeitslasten. Ihr großer Vorteil ist jedoch, dass jeder Server Schreibanforderungen annehmen kann; Arbeitslasten müssen nicht zwischen Primär- und Standby-Servern aufgeteilt werden, und da die Datenänderungen von einem Server an einen anderen gesendet werden, gibt es kein Problem mit nichtdeterministischen Funktionen wie `random()`.

PostgreSQL bietet diese Art von Replikation nicht an, auch wenn PostgreSQLs Two-Phase Commit (`PREPARE TRANSACTION` und `COMMIT PREPARED`) verwendet werden kann, um sie in Anwendungscode oder Middleware zu implementieren.

Tabelle 26.1 fasst die Fähigkeiten der oben genannten Lösungen zusammen.

**Tabelle 26.1. Funktionsmatrix für Hochverfügbarkeit, Lastverteilung und Replikation**

| Merkmal | Shared Disk | Dateisystem-Repl. | Write-Ahead-Log-Shipping | Logische Repl. | Trigger-basierte Repl. | SQL-Repl.-Middleware | Async. MM-Repl. | Sync. MM-Repl. |
|---|---|---|---|---|---|---|---|---|
| Verbreitete Beispiele | NAS | DRBD | eingebaute Streaming-Replikation | eingebaute logische Replikation, pglogical | Londiste, Slony | pgpool-II | Bucardo | |
| Kommunikationsmethode | Shared Disk | Plattenblöcke | WAL | Logical Decoding | Tabellenzeilen | SQL | Tabellenzeilen | Tabellenzeilen und Zeilensperren |
| Keine Spezialhardware erforderlich | | ja | ja | ja | ja | ja | ja | ja |
| Erlaubt mehrere Primärserver | | | | ja | | ja | ja | ja |
| Kein Zusatzaufwand auf dem Primary | ja | | ja | ja | | ja | | |
| Kein Warten auf mehrere Server | ja | | bei ausgeschalteter Synchronität | bei ausgeschalteter Synchronität | ja | | ja | |
| Ausfall des Primary verliert nie Daten | ja | ja | bei eingeschalteter Synchronität | bei eingeschalteter Synchronität | | ja | | ja |
| Replikate akzeptieren schreibgeschützte Abfragen | | | mit Hot Standby | ja | ja | ja | ja | ja |
| Tabellen-Granularität | | | | ja | ja | | ja | ja |
| Keine Konfliktauflösung nötig | ja | ja | ja | | ja | ja | | ja |

Es gibt einige Lösungen, die nicht in die obigen Kategorien passen.

**Datenpartitionierung**

Datenpartitionierung teilt Tabellen in Datenmengen auf. Jede Menge kann nur von einem Server geändert werden. Daten können beispielsweise nach Niederlassungen partitioniert werden, etwa London und Paris, mit je einem Server in jeder Niederlassung. Wenn Abfragen erforderlich sind, die Londoner und Pariser Daten kombinieren, kann eine Anwendung beide Server abfragen, oder Primary/Standby-Replikation kann verwendet werden, um auf jedem Server eine schreibgeschützte Kopie der Daten der jeweils anderen Niederlassung vorzuhalten.

**Parallele Abfrageausführung über mehrere Server**

Viele der oben genannten Lösungen erlauben mehreren Servern, mehrere Abfragen zu bearbeiten, aber keine erlaubt einer einzelnen Abfrage, mehrere Server zu nutzen, um schneller fertig zu werden. Diese Lösung erlaubt mehreren Servern, gemeinsam an einer einzelnen Abfrage zu arbeiten. Üblicherweise wird dies erreicht, indem die Daten auf Server aufgeteilt werden und jeder Server seinen Teil der Abfrage ausführt und Ergebnisse an einen zentralen Server zurückgibt, wo sie kombiniert und an den Benutzer zurückgegeben werden. Dies kann mit dem PL/Proxy-Werkzeugsatz implementiert werden.

Außerdem ist zu beachten, dass PostgreSQL Open Source und leicht erweiterbar ist. Daher haben mehrere Unternehmen PostgreSQL genommen und kommerzielle Closed-Source-Lösungen mit eigenen Failover-, Replikations- und Lastverteilungsfähigkeiten geschaffen. Diese werden hier nicht behandelt.

## 26.2. Log-Shipping-Standby-Server

Kontinuierliche Archivierung kann verwendet werden, um eine Hochverfügbarkeits-Cluster-Konfiguration (HA) mit einem oder mehreren Standby-Servern zu erstellen, die bereit sind, den Betrieb zu übernehmen, wenn der Primärserver ausfällt. Diese Fähigkeit wird weithin als Warm Standby oder Log Shipping bezeichnet.

Primär- und Standby-Server arbeiten zusammen, um diese Fähigkeit bereitzustellen, sind aber nur lose gekoppelt. Der Primärserver arbeitet im Modus kontinuierlicher Archivierung, während jeder Standby-Server im Modus kontinuierlicher Recovery arbeitet und die WAL-Dateien des Primary liest. Zur Aktivierung dieser Fähigkeit sind keine Änderungen an den Datenbanktabellen erforderlich, sodass der Administrationsaufwand im Vergleich zu einigen anderen Replikationslösungen gering ist. Außerdem hat diese Konfiguration relativ geringe Performance-Auswirkungen auf den Primärserver.

Das direkte Verschieben von WAL-Datensätzen von einem Datenbankserver zu einem anderen wird typischerweise als Log Shipping bezeichnet. PostgreSQL implementiert dateibasiertes Log Shipping, indem WAL-Datensätze jeweils dateiweise, also pro WAL-Segment, übertragen werden. WAL-Dateien von 16 MB können einfach und kostengünstig über beliebige Entfernungen verschickt werden, ob zu einem benachbarten System, zu einem anderen System am selben Standort oder zu einem System auf der anderen Seite der Welt. Die für diese Technik erforderliche Bandbreite hängt von der Transaktionsrate des Primärservers ab. Datensatzbasiertes Log Shipping ist feingranularer und streamt WAL-Änderungen inkrementell über eine Netzwerkverbindung; siehe [Abschnitt 26.2.5](#2625-streamingreplikation).

Es ist zu beachten, dass Log Shipping asynchron ist, das heißt, die WAL-Datensätze werden nach dem Commit der Transaktion verschickt. Dadurch gibt es ein Zeitfenster für Datenverlust, falls der Primärserver katastrophal ausfällt; noch nicht verschickte Transaktionen gehen verloren. Die Größe dieses Datenverlustfensters kann beim dateibasierten Log Shipping durch den Parameter `archive_timeout` begrenzt werden, der auf wenige Sekunden gesetzt werden kann. Eine so niedrige Einstellung erhöht jedoch die für File Shipping benötigte Bandbreite erheblich. Streaming-Replikation (siehe [Abschnitt 26.2.5](#2625-streamingreplikation)) erlaubt ein wesentlich kleineres Datenverlustfenster.

Die Recovery-Performance ist gut genug, dass der Standby typischerweise nur Augenblicke von voller Verfügbarkeit entfernt ist, sobald er aktiviert wurde. Daher spricht man von einer Warm-Standby-Konfiguration, die Hochverfügbarkeit bietet. Einen Server aus einer archivierten Basissicherung wiederherzustellen und per Rollforward nachzuziehen, dauert deutlich länger; diese Technik bietet daher nur eine Lösung für Disaster Recovery, nicht für Hochverfügbarkeit. Ein Standby-Server kann auch für schreibgeschützte Abfragen verwendet werden; in diesem Fall heißt er Hot-Standby-Server. Weitere Informationen finden Sie in [Abschnitt 26.4](#264-hot-standby).

### 26.2.1. Planung

Es ist normalerweise sinnvoll, Primär- und Standby-Server so ähnlich wie möglich aufzubauen, zumindest aus Sicht des Datenbankservers. Insbesondere werden die mit Tablespaces verbundenen Pfadnamen unverändert weitergegeben; daher müssen Primär- und Standby-Server dieselben Mount-Pfade für Tablespaces haben, wenn diese Funktion genutzt wird. Denken Sie daran: Wenn `CREATE TABLESPACE` auf dem Primary ausgeführt wird, muss jeder dafür benötigte neue Mount-Punkt auf dem Primary und auf allen Standby-Servern erstellt sein, bevor der Befehl ausgeführt wird. Die Hardware muss nicht exakt identisch sein, aber die Erfahrung zeigt, dass zwei identische Systeme über die Lebensdauer der Anwendung und des Systems leichter zu warten sind als zwei unterschiedliche. In jedem Fall muss die Hardwarearchitektur dieselbe sein; Shipping etwa von einem 32-Bit- auf ein 64-Bit-System funktioniert nicht.

Im Allgemeinen ist Log Shipping zwischen Servern mit unterschiedlichen PostgreSQL-Hauptversionen nicht möglich. Die PostgreSQL Global Development Group verfolgt die Politik, bei Minor-Release-Upgrades keine Änderungen an Plattenformaten vorzunehmen. Daher ist es wahrscheinlich, dass unterschiedliche Minor-Release-Stände auf Primär- und Standby-Servern funktionieren. Formale Unterstützung dafür gibt es jedoch nicht, und es wird empfohlen, Primär- und Standby-Server möglichst auf demselben Release-Stand zu halten. Beim Aktualisieren auf ein neues Minor Release ist es am sichersten, zuerst die Standby-Server zu aktualisieren; ein neues Minor Release kann WAL-Dateien aus einem vorherigen Minor Release eher lesen als umgekehrt.

### 26.2.2. Betrieb eines Standby-Servers

Ein Server wechselt in den Standby-Modus, wenn beim Start eine Datei `standby.signal` im Datenverzeichnis vorhanden ist.

Im Standby-Modus wendet der Server kontinuierlich WAL an, das vom Primärserver empfangen wurde. Der Standby-Server kann WAL aus einem WAL-Archiv (siehe `restore_command`) oder direkt vom Primary über eine TCP-Verbindung lesen (Streaming-Replikation). Der Standby-Server versucht außerdem, jedes WAL wiederherzustellen, das im Verzeichnis `pg_wal` des Standby-Clusters gefunden wird. Das geschieht typischerweise nach einem Serverneustart, wenn der Standby erneut WAL abspielt, das vor dem Neustart vom Primary gestreamt wurde. Sie können aber auch jederzeit Dateien manuell nach `pg_wal` kopieren, damit sie abgespielt werden.

Beim Start beginnt der Standby damit, alles im Archiv verfügbare WAL wiederherzustellen, indem er `restore_command` aufruft. Sobald er das Ende des dort verfügbaren WAL erreicht und `restore_command` fehlschlägt, versucht er, verfügbares WAL im Verzeichnis `pg_wal` wiederherzustellen. Wenn auch das fehlschlägt und Streaming-Replikation konfiguriert wurde, versucht der Standby, sich mit dem Primärserver zu verbinden und WAL ab dem letzten gültigen Datensatz zu streamen, der im Archiv oder in `pg_wal` gefunden wurde. Wenn das fehlschlägt, Streaming-Replikation nicht konfiguriert ist oder die Verbindung später getrennt wird, kehrt der Standby zu Schritt 1 zurück und versucht erneut, die Datei aus dem Archiv wiederherzustellen. Diese Schleife aus Wiederholungen aus dem Archiv, aus `pg_wal` und per Streaming-Replikation läuft weiter, bis der Server gestoppt oder hochgestuft wird.

Der Standby-Modus wird verlassen und der Server wechselt in den normalen Betrieb, wenn `pg_ctl promote` ausgeführt oder `pg_promote()` aufgerufen wird. Vor dem Failover wird jedes unmittelbar im Archiv oder in `pg_wal` verfügbare WAL wiederhergestellt, aber es wird nicht versucht, eine Verbindung zum Primary aufzubauen.

### 26.2.3. Den Primary für Standby-Server vorbereiten

Richten Sie auf dem Primary kontinuierliche Archivierung in ein Archivverzeichnis ein, das vom Standby aus zugänglich ist, wie in [Abschnitt 25.3](25_Sicherung_und_Wiederherstellung.md#253-kontinuierliche-archivierung-und-pointintimerecovery-pitr) beschrieben. Der Archivort sollte vom Standby aus auch dann erreichbar sein, wenn der Primary heruntergefahren ist. Er sollte also auf dem Standby-Server selbst oder auf einem anderen vertrauenswürdigen Server liegen, nicht auf dem Primärserver.

Wenn Sie Streaming-Replikation verwenden möchten, richten Sie auf dem Primärserver Authentifizierung ein, um Replikationsverbindungen von den Standby-Servern zu erlauben. Erstellen Sie also eine Rolle und passende Einträge in `pg_hba.conf`, wobei das Datenbankfeld auf `replication` gesetzt wird. Stellen Sie außerdem sicher, dass `max_wal_senders` in der Konfigurationsdatei des Primärservers auf einen ausreichend großen Wert gesetzt ist. Wenn Replikations-Slots verwendet werden, stellen Sie auch sicher, dass `max_replication_slots` ausreichend hoch gesetzt ist.

Erstellen Sie eine Basissicherung, wie in [Abschnitt 25.3.2](25_Sicherung_und_Wiederherstellung.md#2532-eine-basissicherung-erstellen) beschrieben, um den Standby-Server zu initialisieren.

### 26.2.4. Einen Standby-Server einrichten

Um den Standby-Server einzurichten, stellen Sie die vom Primärserver erstellte Basissicherung wieder her; siehe [Abschnitt 25.3.5](25_Sicherung_und_Wiederherstellung.md#2535-wiederherstellung-mit-einer-sicherung-aus-kontinuierlichem-archiv). Erstellen Sie im Cluster-Datenverzeichnis des Standby eine Datei `standby.signal`. Setzen Sie `restore_command` auf einen einfachen Befehl, der Dateien aus dem WAL-Archiv kopiert. Wenn Sie mehrere Standby-Server für Hochverfügbarkeit einrichten, stellen Sie sicher, dass `recovery_target_timeline` auf `latest` gesetzt ist, was der Standardwert ist, damit der Standby-Server der Timeline-Änderung folgt, die beim Failover auf einen anderen Standby entsteht.

> **Hinweis**
>
> `restore_command` sollte sofort zurückkehren, wenn die Datei nicht existiert; der Server wiederholt den Befehl bei Bedarf.

Wenn Sie Streaming-Replikation verwenden möchten, füllen Sie `primary_conninfo` mit einer `libpq`-Verbindungszeichenkette, einschließlich Hostname oder IP-Adresse und aller weiteren Angaben, die für die Verbindung zum Primärserver nötig sind. Wenn der Primary zur Authentifizierung ein Passwort benötigt, muss das Passwort ebenfalls in `primary_conninfo` angegeben werden.

Wenn Sie den Standby-Server für Hochverfügbarkeit einrichten, konfigurieren Sie WAL-Archivierung, Verbindungen und Authentifizierung wie auf dem Primärserver, weil der Standby-Server nach einem Failover als Primärserver arbeiten wird.

Wenn Sie ein WAL-Archiv verwenden, kann dessen Größe mit dem Parameter `archive_cleanup_command` minimiert werden, indem Dateien entfernt werden, die der Standby-Server nicht mehr benötigt. Das Hilfsprogramm `pg_archivecleanup` ist speziell für die Verwendung mit `archive_cleanup_command` in typischen Einzel-Standby-Konfigurationen gedacht; siehe `pg_archivecleanup`. Beachten Sie jedoch: Wenn Sie das Archiv für Sicherungszwecke verwenden, müssen Sie Dateien aufbewahren, die zur Wiederherstellung mindestens ab der neuesten Basissicherung erforderlich sind, auch wenn der Standby sie nicht mehr benötigt.

Ein einfaches Konfigurationsbeispiel:

```text
primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=foopass options=''-c wal_sender_timeout=5000'''
restore_command = 'cp /path/to/archive/%f %p'
archive_cleanup_command = 'pg_archivecleanup /path/to/archive %r'
```

Sie können beliebig viele Standby-Server haben. Wenn Sie jedoch Streaming-Replikation verwenden, achten Sie darauf, `max_wal_senders` auf dem Primary hoch genug zu setzen, damit alle gleichzeitig verbunden sein können.

### 26.2.5. Streaming-Replikation

Streaming-Replikation erlaubt einem Standby-Server, aktueller zu bleiben, als es mit dateibasiertem Log Shipping möglich ist. Der Standby verbindet sich mit dem Primary, der WAL-Datensätze an den Standby streamt, sobald sie entstehen, ohne zu warten, bis die WAL-Datei gefüllt ist.

Streaming-Replikation ist standardmäßig asynchron (siehe [Abschnitt 26.2.8](#2628-synchrone-replikation)). In diesem Fall gibt es eine kleine Verzögerung zwischen dem Commit einer Transaktion auf dem Primary und dem Sichtbarwerden der Änderungen auf dem Standby. Diese Verzögerung ist jedoch deutlich kleiner als beim dateibasierten Log Shipping, typischerweise unter einer Sekunde, sofern der Standby leistungsfähig genug ist, um mit der Last mitzuhalten. Bei Streaming-Replikation ist `archive_timeout` nicht erforderlich, um das Datenverlustfenster zu verringern.

Wenn Sie Streaming-Replikation ohne dateibasierte kontinuierliche Archivierung verwenden, kann der Server alte WAL-Segmente recyceln, bevor der Standby sie empfangen hat. In diesem Fall muss der Standby aus einer neuen Basissicherung neu initialisiert werden. Sie können dies vermeiden, indem Sie `wal_keep_size` auf einen Wert setzen, der groß genug ist, damit WAL-Segmente nicht zu früh recycelt werden, oder indem Sie einen Replikations-Slot für den Standby konfigurieren. Wenn Sie ein vom Standby erreichbares WAL-Archiv einrichten, sind diese Lösungen nicht erforderlich, da der Standby immer das Archiv zum Aufholen verwenden kann, sofern es genügend Segmente aufbewahrt.

Um Streaming-Replikation zu verwenden, richten Sie einen dateibasierten Log-Shipping-Standby-Server wie in [Abschnitt 26.2](#262-logshippingstandbyserver) beschrieben ein. Der Schritt, der einen dateibasierten Log-Shipping-Standby in einen Streaming-Replikations-Standby verwandelt, besteht darin, `primary_conninfo` auf den Primärserver zeigen zu lassen. Setzen Sie `listen_addresses` und die Authentifizierungsoptionen (siehe `pg_hba.conf`) auf dem Primary so, dass der Standby-Server eine Verbindung zur Replikations-Pseudodatenbank auf dem Primärserver herstellen kann; siehe [Abschnitt 26.2.5.1](#26251-authentifizierung).

Auf Systemen, die die Socket-Option Keepalive unterstützen, hilft das Setzen von `tcp_keepalives_idle`, `tcp_keepalives_interval` und `tcp_keepalives_count` dem Primary, eine unterbrochene Verbindung zeitnah zu bemerken.

Setzen Sie die maximale Anzahl gleichzeitiger Verbindungen von Standby-Servern; Details finden Sie bei `max_wal_senders`.

Wenn der Standby gestartet wird und `primary_conninfo` korrekt gesetzt ist, verbindet sich der Standby nach dem Abspielen aller im Archiv verfügbaren WAL-Dateien mit dem Primary. Wenn die Verbindung erfolgreich hergestellt wurde, sehen Sie auf dem Standby einen `walreceiver` und auf dem Primary einen entsprechenden `walsender`-Prozess.

#### 26.2.5.1. Authentifizierung

Es ist sehr wichtig, die Zugriffsrechte für Replikation so einzurichten, dass nur vertrauenswürdige Benutzer den WAL-Strom lesen können, denn daraus lassen sich leicht privilegierte Informationen extrahieren. Standby-Server müssen sich gegenüber dem Primary als Konto authentifizieren, das das Privileg `REPLICATION` besitzt oder Superuser ist. Es wird empfohlen, ein eigenes Benutzerkonto mit den Privilegien `REPLICATION` und `LOGIN` für die Replikation zu erstellen. Das Privileg `REPLICATION` verleiht sehr weitgehende Berechtigungen, erlaubt dem Benutzer aber im Unterschied zum Privileg `SUPERUSER` nicht, Daten auf dem Primary-System zu ändern.

Die Client-Authentifizierung für Replikation wird durch einen `pg_hba.conf`-Eintrag gesteuert, der im Datenbankfeld `replication` angibt. Wenn der Standby beispielsweise auf Host-IP `192.168.1.100` läuft und der Kontoname für die Replikation `foo` ist, kann der Administrator die folgende Zeile zur Datei `pg_hba.conf` auf dem Primary hinzufügen:

```text
# Allow the user "foo" from host 192.168.1.100 to connect to the primary
# as a replication standby if the user's password is correctly supplied.
#
# TYPE  DATABASE      USER  ADDRESS           METHOD
host    replication   foo   192.168.1.100/32  md5
```

Hostname und Portnummer des Primary, Verbindungsbenutzername und Passwort werden in `primary_conninfo` angegeben. Das Passwort kann auch in der Datei `~/.pgpass` auf dem Standby gesetzt werden; geben Sie dabei `replication` im Datenbankfeld an. Wenn der Primary beispielsweise auf Host-IP `192.168.1.50`, Port `5432`, läuft, der Kontoname für die Replikation `foo` ist und das Passwort `foopass` lautet, kann der Administrator die folgende Zeile zur Datei `postgresql.conf` auf dem Standby hinzufügen:

```text
# The standby connects to the primary that is running on host 192.168.1.50
# and port 5432 as the user "foo" whose password is "foopass".
primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=foopass'
```

#### 26.2.5.2. Überwachung

Ein wichtiger Gesundheitsindikator der Streaming-Replikation ist die Menge an WAL-Datensätzen, die auf dem Primary erzeugt, aber auf dem Standby noch nicht angewendet wurde. Diese Verzögerung können Sie berechnen, indem Sie die aktuelle WAL-Schreibposition auf dem Primary mit der letzten vom Standby empfangenen WAL-Position vergleichen. Diese Positionen können mit `pg_current_wal_lsn` auf dem Primary beziehungsweise `pg_last_wal_receive_lsn` auf dem Standby abgerufen werden; Details finden Sie in Tabelle 9.97 und Tabelle 9.98. Die letzte WAL-Empfangsposition auf dem Standby wird außerdem im Prozessstatus des WAL-Receiver-Prozesses angezeigt, der mit dem Befehl `ps` sichtbar ist; Details siehe [Abschnitt 27.1](27_Datenbankaktivität_überwachen.md#271-standardunixwerkzeuge).

Eine Liste der WAL-Sender-Prozesse können Sie über die Sicht `pg_stat_replication` abrufen. Große Unterschiede zwischen `pg_current_wal_lsn` und dem Feld `sent_lsn` der Sicht können darauf hinweisen, dass der Primärserver stark belastet ist. Unterschiede zwischen `sent_lsn` und `pg_last_wal_receive_lsn` auf dem Standby können auf Netzwerkverzögerungen oder eine hohe Last auf dem Standby hinweisen.

Auf einem Hot Standby kann der Status des WAL-Receiver-Prozesses über die Sicht `pg_stat_wal_receiver` abgerufen werden. Ein großer Unterschied zwischen `pg_last_wal_replay_lsn` und dem Feld `flushed_lsn` der Sicht zeigt an, dass WAL schneller empfangen wird, als es abgespielt werden kann.

### 26.2.6. Replikations-Slots

Replikations-Slots stellen einen automatisierten Weg bereit, um sicherzustellen, dass der Primärserver WAL-Segmente erst entfernt, nachdem sie von allen Standbys empfangen wurden, und dass der Primary keine Zeilen entfernt, die einen Recovery-Konflikt verursachen könnten, auch wenn der Standby getrennt ist.

Anstelle von Replikations-Slots ist es möglich, das Entfernen alter WAL-Segmente mit `wal_keep_size` oder durch Speichern der Segmente in einem Archiv mit `archive_command` oder `archive_library` zu verhindern. Ein Nachteil dieser Methoden ist, dass sie oft dazu führen, dass mehr WAL-Segmente aufbewahrt werden als nötig, während Replikations-Slots nur die Anzahl von Segmenten behalten, die bekanntermaßen benötigt wird.

Ähnlich schützt `hot_standby_feedback` allein, ohne zusätzlich einen Replikations-Slot zu verwenden, zwar davor, dass relevante Zeilen durch Vacuum entfernt werden, bietet aber keinen Schutz während der Zeiträume, in denen der Standby nicht verbunden ist.

> **Vorsicht**
>
> Beachten Sie, dass Replikations-Slots dazu führen können, dass der Server so viele WAL-Segmente zurückhält, dass der für `pg_wal` vorgesehene Platz vollläuft. Mit `max_slot_wal_keep_size` kann die Größe der von Replikations-Slots zurückgehaltenen WAL-Dateien begrenzt werden.

#### 26.2.6.1. Replikations-Slots abfragen und manipulieren

Jeder Replikations-Slot hat einen Namen, der Kleinbuchstaben, Ziffern und den Unterstrich enthalten kann.

Vorhandene Replikations-Slots und ihr Zustand sind in der Sicht `pg_replication_slots` sichtbar.

Slots können entweder über das Streaming-Replikationsprotokoll (siehe [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll)) oder über SQL-Funktionen erstellt und gelöscht werden; siehe Abschnitt 9.28.6.

#### 26.2.6.2. Konfigurationsbeispiel

Sie können einen Replikations-Slot so erstellen:

```text
postgres=# SELECT * FROM pg_create_physical_replication_slot('node_a_slot');
 slot_name   | lsn
-------------+-----
 node_a_slot |

postgres=# SELECT slot_name, slot_type, active FROM pg_replication_slots;
 slot_name   | slot_type | active
-------------+-----------+--------
 node_a_slot | physical  | f
(1 row)
```

Um den Standby so zu konfigurieren, dass er diesen Slot verwendet, sollte `primary_slot_name` auf dem Standby konfiguriert werden. Ein einfaches Beispiel:

```text
primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=foopass'
primary_slot_name = 'node_a_slot'
```

### 26.2.7. Kaskadierende Replikation

Die Funktion der kaskadierenden Replikation erlaubt einem Standby-Server, Replikationsverbindungen anzunehmen und WAL-Datensätze an andere Standbys zu streamen, also als Relay zu dienen. Dies kann genutzt werden, um die Anzahl direkter Verbindungen zum Primary zu verringern und außerdem den Bandbreitenaufwand zwischen Standorten zu minimieren.

Ein Standby, der sowohl als Empfänger als auch als Sender dient, heißt kaskadierender Standby. Standbys, die direkter mit dem Primary verbunden sind, heißen Upstream-Server; weiter entfernte Standby-Server heißen Downstream-Server. Kaskadierende Replikation setzt keine Grenzen für Anzahl oder Anordnung von Downstream-Servern, obwohl jeder Standby nur mit einem Upstream-Server verbunden ist, der letztlich zu einem einzigen Primärserver führt.

Ein kaskadierender Standby sendet nicht nur WAL-Datensätze, die er vom Primary empfangen hat, sondern auch solche, die aus dem Archiv wiederhergestellt wurden. Selbst wenn also die Replikationsverbindung in einer Upstream-Verbindung beendet wird, läuft die Streaming-Replikation downstream weiter, solange neue WAL-Datensätze verfügbar sind.

Kaskadierende Replikation ist derzeit asynchron. Einstellungen für synchrone Replikation (siehe [Abschnitt 26.2.8](#2628-synchrone-replikation)) haben derzeit keine Wirkung auf kaskadierende Replikation.

Hot-Standby-Feedback wird upstream weitergegeben, unabhängig von der kaskadierten Anordnung.

Wenn ein Upstream-Standby-Server zum neuen Primary hochgestuft wird, streamen Downstream-Server weiter vom neuen Primary, wenn `recovery_target_timeline` auf `latest` gesetzt ist, was der Standardwert ist.

Um kaskadierende Replikation zu verwenden, richten Sie den kaskadierenden Standby so ein, dass er Replikationsverbindungen annehmen kann. Setzen Sie also `max_wal_senders` und `hot_standby` und konfigurieren Sie hostbasierte Authentifizierung. Außerdem müssen Sie `primary_conninfo` im Downstream-Standby so setzen, dass es auf den kaskadierenden Standby zeigt.

### 26.2.8. Synchrone Replikation

PostgreSQL-Streaming-Replikation ist standardmäßig asynchron. Wenn der Primärserver abstürzt, können einige committete Transaktionen noch nicht auf den Standby-Server repliziert worden sein, was zu Datenverlust führt. Die Menge verlorener Daten ist proportional zur Replikationsverzögerung zum Zeitpunkt des Failovers.

Synchrone Replikation bietet die Möglichkeit zu bestätigen, dass alle von einer Transaktion vorgenommenen Änderungen an einen oder mehrere synchrone Standby-Server übertragen wurden. Dies erweitert das normale Dauerhaftigkeitsniveau, das ein Transaktions-Commit bietet. Dieses Schutzniveau wird in der Informatiktheorie als 2-safe replication bezeichnet und als group-1-safe (group-safe und 1-safe), wenn `synchronous_commit` auf `remote_write` gesetzt ist.

Bei angeforderter synchroner Replikation wartet jeder Commit einer Schreibtransaktion, bis die Bestätigung eingegangen ist, dass der Commit sowohl auf dem Primary als auch auf dem Standby-Server in das Write-Ahead-Log auf Platte geschrieben wurde. Daten können nur verloren gehen, wenn Primary und Standby gleichzeitig abstürzen. Dies kann ein deutlich höheres Dauerhaftigkeitsniveau bieten, allerdings nur, wenn der Systemadministrator bei Platzierung und Verwaltung der beiden Server sorgfältig ist. Das Warten auf Bestätigung erhöht das Vertrauen des Benutzers, dass Änderungen bei Serverabstürzen nicht verloren gehen, erhöht aber notwendigerweise auch die Antwortzeit der anfordernden Transaktion. Die minimale Wartezeit ist die Roundtrip-Zeit zwischen Primary und Standby.

Schreibgeschützte Transaktionen und Transaktions-Rollbacks müssen nicht auf Antworten von Standby-Servern warten. Commits von Subtransaktionen warten nicht auf Antworten von Standby-Servern, nur Top-Level-Commits. Lang laufende Aktionen wie Datenladen oder Indexaufbau warten erst bei der allerletzten Commit-Nachricht. Alle Two-Phase-Commit-Aktionen benötigen Commit-Wartezeiten, einschließlich Prepare und Commit.

Ein synchroner Standby kann ein Standby für physische Replikation oder ein Subscriber für logische Replikation sein. Er kann auch jeder andere Konsument eines physischen oder logischen WAL-Replikationsstroms sein, der die passenden Feedback-Nachrichten senden kann. Neben den eingebauten Systemen für physische und logische Replikation umfasst dies Spezialprogramme wie `pg_receivewal` und `pg_recvlogical` sowie einige Replikationssysteme von Drittanbietern und eigene Programme. Details zur Unterstützung synchroner Replikation finden Sie in der jeweiligen Dokumentation.

#### 26.2.8.1. Grundkonfiguration

Sobald Streaming-Replikation konfiguriert ist, erfordert die Konfiguration synchroner Replikation nur einen zusätzlichen Konfigurationsschritt: `synchronous_standby_names` muss auf einen nicht leeren Wert gesetzt werden. `synchronous_commit` muss ebenfalls auf `on` stehen; da dies der Standardwert ist, ist typischerweise keine Änderung erforderlich. Siehe Abschnitt 19.5.1 und Abschnitt 19.6.2. Diese Konfiguration bewirkt, dass jeder Commit auf die Bestätigung wartet, dass der Standby den Commit-Datensatz in dauerhaften Speicher geschrieben hat. `synchronous_commit` kann von einzelnen Benutzern gesetzt werden, sodass es in der Konfigurationsdatei, für bestimmte Benutzer oder Datenbanken oder dynamisch durch Anwendungen konfiguriert werden kann, um die Dauerhaftigkeitsgarantie pro Transaktion zu steuern.

Nachdem ein Commit-Datensatz auf dem Primary auf Platte geschrieben wurde, wird der WAL-Datensatz an den Standby gesendet. Der Standby sendet Antwortnachrichten, wann immer ein neuer Batch von WAL-Daten auf Platte geschrieben wurde, sofern `wal_receiver_status_interval` auf dem Standby nicht auf null gesetzt ist. Falls `synchronous_commit` auf `remote_apply` gesetzt ist, sendet der Standby Antwortnachrichten, wenn der Commit-Datensatz abgespielt wurde und die Transaktion sichtbar ist. Wenn der Standby gemäß der Einstellung `synchronous_standby_names` auf dem Primary als synchroner Standby ausgewählt ist, werden die Antwortnachrichten dieses Standby zusammen mit denen anderer synchroner Standbys berücksichtigt, um zu entscheiden, wann Transaktionen freigegeben werden, die auf die Bestätigung des empfangenen Commit-Datensatzes warten. Diese Parameter erlauben dem Administrator festzulegen, welche Standby-Server synchrone Standbys sein sollen. Beachten Sie, dass die Konfiguration synchroner Replikation hauptsächlich auf dem Primary erfolgt. Benannte Standbys müssen direkt mit dem Primary verbunden sein; der Primary weiß nichts über Downstream-Standby-Server, die kaskadierte Replikation verwenden.

Wenn `synchronous_commit` auf `remote_write` gesetzt wird, wartet jeder Commit auf die Bestätigung, dass der Standby den Commit-Datensatz empfangen und an sein eigenes Betriebssystem geschrieben hat, aber nicht darauf, dass die Daten auf dem Standby auf Platte geflusht wurden. Diese Einstellung bietet eine schwächere Dauerhaftigkeitsgarantie als `on`: Der Standby könnte die Daten bei einem Betriebssystemabsturz verlieren, nicht jedoch bei einem PostgreSQL-Absturz. In der Praxis ist diese Einstellung trotzdem nützlich, weil sie die Antwortzeit der Transaktion verringern kann. Datenverlust könnte nur eintreten, wenn sowohl Primary als auch Standby abstürzen und gleichzeitig die Datenbank des Primary beschädigt wird.

Wenn `synchronous_commit` auf `remote_apply` gesetzt wird, wartet jeder Commit, bis die aktuellen synchronen Standbys melden, dass sie die Transaktion abgespielt haben und sie für Benutzerabfragen sichtbar ist. In einfachen Fällen erlaubt dies Lastverteilung mit kausaler Konsistenz.

Benutzer hören auf zu warten, wenn ein schneller Shutdown angefordert wird. Wie bei asynchroner Replikation fährt der Server jedoch erst vollständig herunter, wenn alle ausstehenden WAL-Datensätze an die derzeit verbundenen Standby-Server übertragen wurden.

#### 26.2.8.2. Mehrere synchrone Standbys

Synchrone Replikation unterstützt einen oder mehrere synchrone Standby-Server; Transaktionen warten, bis alle als synchron betrachteten Standby-Server den Empfang ihrer Daten bestätigen. Die Anzahl synchroner Standbys, von denen Transaktionen Antworten erwarten müssen, wird in `synchronous_standby_names` angegeben. Dieser Parameter legt außerdem eine Liste von Standby-Namen und die Methode (`FIRST` und `ANY`) fest, mit der synchrone Standbys aus den aufgeführten Standbys ausgewählt werden.

Die Methode `FIRST` legt prioritätsbasierte synchrone Replikation fest und sorgt dafür, dass Transaktions-Commits warten, bis ihre WAL-Datensätze auf die geforderte Anzahl synchroner Standbys repliziert wurden, die nach Priorität ausgewählt werden. Standbys, deren Namen früher in der Liste erscheinen, erhalten höhere Priorität und werden als synchron betrachtet. Andere später in dieser Liste erscheinende Standby-Server sind mögliche synchrone Standbys. Wenn einer der aktuellen synchronen Standbys aus irgendeinem Grund die Verbindung trennt, wird er sofort durch den Standby mit der nächsthöheren Priorität ersetzt.

Ein Beispiel für `synchronous_standby_names` für mehrere prioritätsbasierte synchrone Standbys:

```text
synchronous_standby_names = 'FIRST 2 (s1, s2, s3)'
```

Wenn in diesem Beispiel vier Standby-Server `s1`, `s2`, `s3` und `s4` laufen, werden die beiden Standbys `s1` und `s2` als synchrone Standbys ausgewählt, weil ihre Namen früh in der Liste der Standby-Namen stehen. `s3` ist ein möglicher synchroner Standby und übernimmt die Rolle eines synchronen Standby, wenn entweder `s1` oder `s2` ausfällt. `s4` ist ein asynchroner Standby, weil sein Name nicht in der Liste steht.

Die Methode `ANY` legt quorum-basierte synchrone Replikation fest und sorgt dafür, dass Transaktions-Commits warten, bis ihre WAL-Datensätze auf mindestens die geforderte Anzahl synchroner Standbys in der Liste repliziert wurden.

Ein Beispiel für `synchronous_standby_names` für mehrere quorum-basierte synchrone Standbys:

```text
synchronous_standby_names = 'ANY 2 (s1, s2, s3)'
```

Wenn in diesem Beispiel vier Standby-Server `s1`, `s2`, `s3` und `s4` laufen, warten Transaktions-Commits auf Antworten von mindestens zwei beliebigen Standbys aus `s1`, `s2` und `s3`. `s4` ist ein asynchroner Standby, weil sein Name nicht in der Liste steht.

Die Synchronzustände der Standby-Server können mit der Sicht `pg_stat_replication` angezeigt werden.

#### 26.2.8.3. Performance-Planung

Synchrone Replikation erfordert normalerweise sorgfältig geplante und platzierte Standby-Server, damit Anwendungen akzeptable Performance erreichen. Warten verbraucht keine Systemressourcen, aber Transaktionssperren bleiben gehalten, bis die Übertragung bestätigt wurde. Daher verringert unvorsichtige Verwendung synchroner Replikation die Performance von Datenbankanwendungen durch höhere Antwortzeiten und stärkere Konkurrenz.

PostgreSQL erlaubt dem Anwendungsentwickler, das durch Replikation benötigte Dauerhaftigkeitsniveau festzulegen. Dies kann für das Gesamtsystem angegeben werden, aber auch für bestimmte Benutzer oder Verbindungen oder sogar für einzelne Transaktionen.

Eine Anwendungsarbeitslast könnte beispielsweise so aussehen: 10 % der Änderungen betreffen wichtige Kundendaten, während 90 % weniger wichtige Daten sind, deren Verlust das Unternehmen leichter verkraften kann, etwa Chatnachrichten zwischen Benutzern.

Mit auf Anwendungsebene angegebenen Optionen für synchrone Replikation, auf dem Primary, können wir synchrone Replikation für die wichtigsten Änderungen anbieten, ohne den Großteil der gesamten Arbeitslast zu verlangsamen. Optionen auf Anwendungsebene sind ein wichtiges und praktisches Werkzeug, um die Vorteile synchroner Replikation für Hochleistungsanwendungen nutzbar zu machen.

Sie sollten berücksichtigen, dass die Netzwerkbandbreite höher sein muss als die Erzeugungsrate von WAL-Daten.

#### 26.2.8.4. Hochverfügbarkeitsplanung

`synchronous_standby_names` gibt Anzahl und Namen synchroner Standbys an, von denen Transaktions-Commits, die bei `synchronous_commit` gleich `on`, `remote_apply` oder `remote_write` ausgeführt werden, Antworten erwarten. Solche Transaktions-Commits werden möglicherweise nie abgeschlossen, wenn einer der synchronen Standbys abstürzt.

Die beste Lösung für Hochverfügbarkeit ist sicherzustellen, dass Sie so viele synchrone Standbys vorhalten, wie angefordert werden. Dies kann erreicht werden, indem mehrere mögliche synchrone Standbys mit `synchronous_standby_names` benannt werden.

Bei prioritätsbasierter synchroner Replikation werden Standbys, deren Namen früher in der Liste erscheinen, als synchrone Standbys verwendet. Danach aufgeführte Standbys übernehmen die Rolle des synchronen Standby, wenn einer der aktuellen ausfällt.

Bei quorum-basierter synchroner Replikation werden alle in der Liste aufgeführten Standbys als Kandidaten für synchrone Standbys verwendet. Selbst wenn einer von ihnen ausfällt, übernehmen die anderen Standbys weiterhin die Rolle von Kandidaten für synchrone Standbys.

Wenn ein Standby sich zum ersten Mal mit dem Primary verbindet, ist er noch nicht ordnungsgemäß synchronisiert. Dies wird als Catchup-Modus bezeichnet. Sobald die Verzögerung zwischen Standby und Primary erstmals null erreicht, wechseln wir in den Echtzeit-Streaming-Zustand. Die Catch-up-Dauer kann unmittelbar nach dem Erstellen des Standby lang sein. Wenn der Standby heruntergefahren ist, verlängert sich die Catch-up-Phase entsprechend der Dauer, in der der Standby heruntergefahren war. Der Standby kann erst synchroner Standby werden, wenn er den Streaming-Zustand erreicht hat. Dieser Zustand ist über die Sicht `pg_stat_replication` sichtbar.

Wenn der Primary neu startet, während Commits auf Bestätigung warten, werden diese wartenden Transaktionen nach der Recovery der Primary-Datenbank als vollständig committet markiert. Es gibt keine Möglichkeit, sicher zu wissen, ob alle Standbys zum Zeitpunkt des Absturzes des Primary alle ausstehenden WAL-Daten erhalten haben. Einige Transaktionen erscheinen auf dem Standby möglicherweise nicht als committet, obwohl sie auf dem Primary als committet erscheinen. Die angebotene Garantie lautet: Die Anwendung erhält keine explizite Bestätigung eines erfolgreichen Transaktions-Commits, bevor bekannt ist, dass die WAL-Daten sicher von allen synchronen Standbys empfangen wurden.

Wenn Sie wirklich nicht so viele synchrone Standbys vorhalten können, wie angefordert werden, sollten Sie die Anzahl synchroner Standbys verringern, von denen Transaktions-Commits in `synchronous_standby_names` Antworten erwarten müssen, oder die Einstellung deaktivieren, und die Konfigurationsdatei auf dem Primärserver neu laden.

Wenn der Primary von den verbleibenden Standby-Servern isoliert ist, sollten Sie auf den besten Kandidaten unter diesen übrigen Standby-Servern failovern.

Wenn Sie einen Standby-Server neu erstellen müssen, während Transaktionen warten, stellen Sie sicher, dass die Funktionen `pg_backup_start()` und `pg_backup_stop()` in einer Sitzung mit `synchronous_commit = off` ausgeführt werden; andernfalls warten diese Anforderungen ewig darauf, dass der Standby erscheint.

### 26.2.9. Kontinuierliche Archivierung im Standby

Wenn kontinuierliche WAL-Archivierung in einem Standby verwendet wird, gibt es zwei verschiedene Szenarien: Das WAL-Archiv kann vom Primary und vom Standby gemeinsam genutzt werden, oder der Standby kann ein eigenes WAL-Archiv haben. Wenn der Standby ein eigenes WAL-Archiv hat, setzen Sie `archive_mode` auf `always`; dann ruft der Standby für jedes empfangene WAL-Segment den Archivierungsbefehl auf, egal ob es durch Wiederherstellen aus dem Archiv oder durch Streaming-Replikation empfangen wurde. Das gemeinsame Archiv kann ähnlich behandelt werden, aber `archive_command` oder `archive_library` muss testen, ob die zu archivierende Datei bereits existiert und ob die vorhandene Datei identische Inhalte hat. Dies erfordert mehr Sorgfalt in `archive_command` oder `archive_library`, weil darauf geachtet werden muss, keine vorhandene Datei mit anderen Inhalten zu überschreiben, aber Erfolg zurückzugeben, wenn exakt dieselbe Datei zweimal archiviert wird. All das muss frei von Race Conditions erfolgen, wenn zwei Server gleichzeitig versuchen, dieselbe Datei zu archivieren.

Wenn `archive_mode` auf `on` gesetzt ist, ist der Archiver während Recovery oder Standby-Modus nicht aktiviert. Wenn der Standby-Server hochgestuft wird, beginnt er nach der Hochstufung mit der Archivierung, archiviert aber keine WAL- oder Timeline-History-Dateien, die er nicht selbst erzeugt hat. Um eine vollständige Folge von WAL-Dateien im Archiv zu erhalten, müssen Sie sicherstellen, dass alles WAL archiviert ist, bevor es den Standby erreicht. Bei dateibasiertem Log Shipping ist dies inhärent der Fall, da der Standby nur Dateien wiederherstellen kann, die im Archiv gefunden werden, aber nicht, wenn Streaming-Replikation aktiviert ist. Wenn sich ein Server nicht im Recovery-Modus befindet, gibt es keinen Unterschied zwischen den Modi `on` und `always`.

## 26.3. Failover

Wenn der Primärserver ausfällt, sollte der Standby-Server Failover-Verfahren einleiten.

Wenn der Standby-Server ausfällt, muss kein Failover stattfinden. Wenn der Standby-Server neu gestartet werden kann, auch erst später, kann der Recovery-Prozess ebenfalls sofort neu gestartet werden und die wiederaufnehmbare Recovery nutzen. Wenn der Standby-Server nicht neu gestartet werden kann, sollte eine vollständige neue Standby-Server-Instanz erstellt werden.

Wenn der Primärserver ausfällt und der Standby-Server zum neuen Primary wird und der alte Primary danach neu startet, müssen Sie einen Mechanismus haben, um dem alten Primary mitzuteilen, dass er nicht mehr der Primary ist. Dies wird manchmal STONITH genannt (Shoot The Other Node In The Head) und ist notwendig, um Situationen zu vermeiden, in denen beide Systeme glauben, der Primary zu sein, was zu Verwirrung und letztlich Datenverlust führt.

Viele Failover-Systeme verwenden nur zwei Systeme, den Primary und den Standby, die durch eine Art Heartbeat-Mechanismus verbunden sind, um die Konnektivität zwischen beiden und die Funktionsfähigkeit des Primary ständig zu prüfen. Es ist auch möglich, ein drittes System, einen sogenannten Witness-Server, zu verwenden, um einige Fälle unangemessenen Failovers zu verhindern. Die zusätzliche Komplexität lohnt sich aber möglicherweise nur, wenn dies mit ausreichender Sorgfalt und gründlichen Tests eingerichtet wird.

PostgreSQL stellt nicht die Systemsoftware bereit, die erforderlich ist, um einen Ausfall des Primary zu erkennen und den Standby-Datenbankserver zu benachrichtigen. Es gibt viele solche Werkzeuge, die gut in Betriebssystemfunktionen integriert sind, die für erfolgreiches Failover erforderlich sind, etwa IP-Adressmigration.

Sobald ein Failover auf den Standby erfolgt ist, ist nur noch ein einzelner Server in Betrieb. Dies heißt degenerierter Zustand. Der frühere Standby ist nun der Primary, aber der frühere Primary ist heruntergefahren und bleibt möglicherweise heruntergefahren. Um zum normalen Betrieb zurückzukehren, muss ein Standby-Server neu erstellt werden, entweder auf dem früheren Primary-System, wenn es wieder hochkommt, oder auf einem dritten, möglicherweise neuen System. Das Hilfsprogramm `pg_rewind` kann diesen Prozess bei großen Clustern beschleunigen. Sobald dies abgeschlossen ist, können Primary und Standby als rollengewechselt betrachtet werden. Manche verwenden einen dritten Server, um dem neuen Primary Schutz zu bieten, bis der neue Standby-Server neu erstellt ist, doch das verkompliziert offensichtlich Systemkonfiguration und Betriebsabläufe.

Der Wechsel vom Primary zum Standby-Server kann also schnell sein, erfordert aber etwas Zeit, um den Failover-Cluster wieder vorzubereiten. Regelmäßiger Wechsel von Primary zu Standby ist nützlich, weil dadurch regelmäßige Ausfallzeiten auf jedem System für Wartung möglich sind. Das dient außerdem als Test des Failover-Mechanismus, damit er wirklich funktioniert, wenn Sie ihn brauchen. Schriftliche Administrationsverfahren werden empfohlen.

Wenn Sie sich für Synchronisation logischer Replikations-Slots entschieden haben (siehe Abschnitt 47.2.3), wird empfohlen, vor dem Wechsel zum Standby-Server zu prüfen, ob die auf dem Standby-Server synchronisierten logischen Slots für Failover bereit sind. Dies kann durch die in [Abschnitt 29.3](29_Logische_Replikation.md#293-failover-bei-logischer-replikation) beschriebenen Schritte erfolgen.

Um Failover eines Log-Shipping-Standby-Servers auszulösen, führen Sie `pg_ctl promote` aus oder rufen `pg_promote()` auf. Wenn Sie Reporting-Server einrichten, die nur zur Auslagerung schreibgeschützter Abfragen vom Primary dienen und nicht für Hochverfügbarkeit, müssen Sie nicht hochstufen.

## 26.4. Hot Standby

Hot Standby bezeichnet die Fähigkeit, sich mit dem Server zu verbinden und schreibgeschützte Abfragen auszuführen, während sich der Server in Archive-Recovery oder im Standby-Modus befindet. Das ist sowohl für Replikationszwecke nützlich als auch dafür, eine Sicherung mit hoher Präzision auf einen gewünschten Zustand zurückzubringen. Der Begriff Hot Standby bezeichnet außerdem die Fähigkeit des Servers, aus der Recovery in den normalen Betrieb zu wechseln, während Benutzer weiterhin Abfragen ausführen und/oder ihre Verbindungen offen halten.

Abfragen im Hot-Standby-Modus auszuführen ähnelt dem normalen Abfragebetrieb, es gibt jedoch mehrere unten erläuterte Nutzungs- und Administrationsunterschiede.

### 26.4.1. Überblick für Benutzer

Wenn der Parameter `hot_standby` auf einem Standby-Server auf `true` gesetzt ist, beginnt der Server Verbindungen anzunehmen, sobald die Recovery das System in einen konsistenten Zustand gebracht hat und es für Hot Standby bereit ist. Alle diese Verbindungen sind strikt schreibgeschützt; nicht einmal temporäre Tabellen dürfen geschrieben werden.

Die Daten auf dem Standby benötigen etwas Zeit, um vom Primärserver anzukommen. Daher gibt es eine messbare Verzögerung zwischen Primary und Standby. Dieselbe Abfrage nahezu gleichzeitig auf Primary und Standby auszuführen, kann deshalb unterschiedliche Ergebnisse liefern. Man sagt, dass Daten auf dem Standby letztlich mit dem Primary konsistent werden. Sobald der Commit-Datensatz einer Transaktion auf dem Standby abgespielt wurde, sind die von dieser Transaktion vorgenommenen Änderungen für alle neuen Snapshots auf dem Standby sichtbar. Snapshots können je nach aktueller Transaktionsisolationsstufe zu Beginn jeder Abfrage oder zu Beginn jeder Transaktion genommen werden. Weitere Details finden Sie in [Abschnitt 13.2](13_Nebenläufigkeitskontrolle.md#132-transaktionsisolation).

Transaktionen, die während Hot Standby gestartet werden, dürfen die folgenden Befehle ausführen:

- Abfragezugriff: `SELECT`, `COPY TO`
- Cursor-Befehle: `DECLARE`, `FETCH`, `CLOSE`
- Einstellungen: `SHOW`, `SET`, `RESET`
- Transaktionsverwaltungsbefehle: `BEGIN`, `END`, `ABORT`, `START TRANSACTION`, `SAVEPOINT`, `RELEASE`, `ROLLBACK TO SAVEPOINT`, `EXCEPTION`-Blöcke und andere interne Subtransaktionen
- `LOCK TABLE`, jedoch nur wenn ausdrücklich einer dieser Modi verwendet wird: `ACCESS SHARE`, `ROW SHARE` oder `ROW EXCLUSIVE`
- Pläne und Ressourcen: `PREPARE`, `EXECUTE`, `DEALLOCATE`, `DISCARD`
- Plugins und Erweiterungen: `LOAD`
- `UNLISTEN`

Transaktionen, die während Hot Standby gestartet werden, erhalten niemals eine Transaktions-ID und können nicht in das systemweite Write-Ahead-Log schreiben. Daher erzeugen die folgenden Aktionen Fehlermeldungen:

- Data Manipulation Language (DML): `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `COPY FROM`, `TRUNCATE`. Beachten Sie, dass keine erlaubten Aktionen existieren, die während Recovery zur Ausführung eines Triggers führen. Diese Einschränkung gilt sogar für temporäre Tabellen, weil Tabellenzeilen ohne Zuweisung einer Transaktions-ID nicht gelesen oder geschrieben werden können, was in einer Hot-Standby-Umgebung derzeit nicht möglich ist.
- Data Definition Language (DDL): `CREATE`, `DROP`, `ALTER`, `COMMENT`. Diese Einschränkung gilt sogar für temporäre Tabellen, weil die Ausführung dieser Operationen Aktualisierungen der Systemkatalogtabellen erfordern würde.
- `SELECT ... FOR SHARE | UPDATE`, weil Zeilensperren nicht genommen werden können, ohne die zugrunde liegenden Datendateien zu aktualisieren.
- Regeln auf `SELECT`-Anweisungen, die DML-Befehle erzeugen.
- `LOCK`, wenn ausdrücklich ein Modus höher als `ROW EXCLUSIVE MODE` angefordert wird.
- `LOCK` in der kurzen Standardform, da dadurch `ACCESS EXCLUSIVE MODE` angefordert wird.
- Transaktionsverwaltungsbefehle, die ausdrücklich einen nicht schreibgeschützten Zustand setzen: `BEGIN READ WRITE`, `START TRANSACTION READ WRITE`, `SET TRANSACTION READ WRITE`, `SET SESSION CHARACTERISTICS AS TRANSACTION READ WRITE`, `SET transaction_read_only = off`
- Two-Phase-Commit-Befehle: `PREPARE TRANSACTION`, `COMMIT PREPARED`, `ROLLBACK PREPARED`, weil selbst schreibgeschützte Transaktionen in der Prepare-Phase, der ersten Phase von Two-Phase Commit, WAL schreiben müssen.
- Sequenzaktualisierungen: `nextval()`, `setval()`
- `LISTEN`, `NOTIFY`

Im normalen Betrieb dürfen schreibgeschützte Transaktionen `LISTEN` und `NOTIFY` verwenden. Hot-Standby-Sitzungen arbeiten daher unter etwas strengeren Einschränkungen als gewöhnliche schreibgeschützte Sitzungen. Einige dieser Einschränkungen könnten in einer künftigen Version gelockert werden.

Während Hot Standby ist der Parameter `transaction_read_only` immer `true` und darf nicht geändert werden. Solange jedoch kein Versuch unternommen wird, die Datenbank zu ändern, verhalten sich Verbindungen während Hot Standby weitgehend wie jede andere Datenbankverbindung. Wenn Failover oder Switchover erfolgt, wechselt die Datenbank in den normalen Verarbeitungsmodus. Sitzungen bleiben verbunden, während der Server den Modus wechselt. Sobald Hot Standby endet, ist es möglich, Read/Write-Transaktionen zu starten, auch aus einer Sitzung heraus, die während Hot Standby begonnen wurde.

Benutzer können feststellen, ob Hot Standby für ihre Sitzung derzeit aktiv ist, indem sie `SHOW in_hot_standby` ausführen. In Serverversionen vor 14 existierte der Parameter `in_hot_standby` nicht; eine brauchbare Ersatzmethode für ältere Server ist `SHOW transaction_read_only`. Zusätzlich erlaubt eine Reihe von Funktionen (Tabelle 9.98), Informationen über den Standby-Server abzurufen. Damit können Sie Programme schreiben, die den aktuellen Zustand der Datenbank kennen. Diese Funktionen können verwendet werden, um den Fortschritt der Recovery zu überwachen oder komplexe Programme zu schreiben, die die Datenbank in bestimmte Zustände zurückführen.

### 26.4.2. Abfragekonflikte behandeln

Primär- und Standby-Server sind in vieler Hinsicht lose gekoppelt. Aktionen auf dem Primary wirken sich auf den Standby aus. Dadurch besteht die Möglichkeit negativer Interaktionen oder Konflikte zwischen ihnen. Der am einfachsten zu verstehende Konflikt ist Performance: Wenn auf dem Primary ein großer Datenladevorgang stattfindet, erzeugt das einen ähnlichen Strom von WAL-Datensätzen auf dem Standby, sodass Standby-Abfragen um Systemressourcen wie I/O konkurrieren können.

Es gibt außerdem weitere Konflikttypen, die mit Hot Standby auftreten können. Diese Konflikte sind harte Konflikte in dem Sinne, dass Abfragen möglicherweise abgebrochen und in einigen Fällen Sitzungen getrennt werden müssen, um sie aufzulösen. Dem Benutzer stehen mehrere Möglichkeiten zur Verfügung, mit diesen Konflikten umzugehen. Konfliktfälle umfassen:

- `Access Exclusive`-Sperren, die auf dem Primärserver genommen werden, einschließlich ausdrücklicher `LOCK`-Befehle und verschiedener DDL-Aktionen, stehen mit Tabellenzugriffen in Standby-Abfragen im Konflikt.
- Das Löschen eines Tablespace auf dem Primary steht mit Standby-Abfragen im Konflikt, die diesen Tablespace für temporäre Arbeitsdateien verwenden.
- Das Löschen einer Datenbank auf dem Primary steht mit Sitzungen im Konflikt, die auf dem Standby mit dieser Datenbank verbunden sind.
- Die Anwendung eines Vacuum-Cleanup-Datensatzes aus WAL steht mit Standby-Transaktionen im Konflikt, deren Snapshots noch eine der zu entfernenden Zeilen sehen können.
- Die Anwendung eines Vacuum-Cleanup-Datensatzes aus WAL steht mit Abfragen im Konflikt, die auf dem Standby auf die Zielseite zugreifen, unabhängig davon, ob die zu entfernenden Daten sichtbar sind.

Auf dem Primärserver führen diese Fälle einfach zu Warten; der Benutzer kann entscheiden, eine der widersprüchlichen Aktionen abzubrechen. Auf dem Standby gibt es jedoch keine Wahl: Die im WAL protokollierte Aktion ist auf dem Primary bereits geschehen, daher darf der Standby ihre Anwendung nicht verweigern. Außerdem kann es sehr unerwünscht sein, WAL-Anwendung unbegrenzt warten zu lassen, weil der Zustand des Standby immer weiter hinter den Primary zurückfällt. Daher gibt es einen Mechanismus, der Standby-Abfragen zwangsweise abbricht, wenn sie mit anzuwendenden WAL-Datensätzen in Konflikt stehen.

Ein Beispiel für die Problemsituation ist ein Administrator auf dem Primärserver, der `DROP TABLE` auf einer Tabelle ausführt, die gerade auf dem Standby-Server abgefragt wird. Offensichtlich kann die Standby-Abfrage nicht fortgesetzt werden, wenn `DROP TABLE` auf dem Standby angewendet wird. Wenn diese Situation auf dem Primary aufträte, würde `DROP TABLE` warten, bis die andere Abfrage beendet ist. Wenn `DROP TABLE` aber auf dem Primary ausgeführt wird, hat der Primary keine Informationen darüber, welche Abfragen auf dem Standby laufen, und wartet daher nicht auf solche Standby-Abfragen. Die WAL-Änderungsdatensätze erreichen den Standby, während die Standby-Abfrage noch läuft, und verursachen einen Konflikt. Der Standby-Server muss entweder die Anwendung der WAL-Datensätze und aller folgenden verzögern oder die widersprüchliche Abfrage abbrechen, damit `DROP TABLE` angewendet werden kann.

Wenn eine widersprüchliche Abfrage kurz ist, ist es typischerweise wünschenswert, ihr durch eine kurze Verzögerung der WAL-Anwendung den Abschluss zu erlauben. Eine lange Verzögerung der WAL-Anwendung ist jedoch normalerweise nicht wünschenswert. Deshalb hat der Abbruchmechanismus die Parameter `max_standby_archive_delay` und `max_standby_streaming_delay`, die die maximal erlaubte Verzögerung der WAL-Anwendung definieren. Widersprüchliche Abfragen werden abgebrochen, sobald es länger als die relevante Verzögerungseinstellung dauert, neu empfangene WAL-Daten anzuwenden. Es gibt zwei Parameter, damit unterschiedliche Verzögerungswerte für das Lesen von WAL-Daten aus einem Archiv, also anfängliche Recovery aus einer Basissicherung oder Aufholen eines stark zurückgefallenen Standby-Servers, und für das Lesen von WAL-Daten per Streaming-Replikation angegeben werden können.

Bei einem Standby-Server, der hauptsächlich für Hochverfügbarkeit existiert, ist es am besten, die Verzögerungsparameter relativ kurz zu setzen, damit der Server wegen Verzögerungen durch Standby-Abfragen nicht weit hinter den Primary zurückfallen kann. Wenn der Standby-Server dagegen für lang laufende Abfragen gedacht ist, kann ein hoher oder sogar unendlicher Verzögerungswert vorzuziehen sein. Denken Sie jedoch daran, dass eine lang laufende Abfrage dazu führen kann, dass andere Sitzungen auf dem Standby-Server aktuelle Änderungen auf dem Primary nicht sehen, wenn sie die Anwendung von WAL-Datensätzen verzögert.

Sobald die durch `max_standby_archive_delay` oder `max_standby_streaming_delay` angegebene Verzögerung überschritten wurde, werden widersprüchliche Abfragen abgebrochen. Das führt normalerweise nur zu einem Abbruchfehler, obwohl beim Abspielen eines `DROP DATABASE` die gesamte widersprüchliche Sitzung beendet wird. Wenn der Konflikt über eine Sperre entsteht, die von einer inaktiven Transaktion gehalten wird, wird die widersprüchliche Sitzung ebenfalls beendet; dieses Verhalten könnte sich in Zukunft ändern.

Abgebrochene Abfragen können sofort erneut versucht werden, natürlich nach Beginn einer neuen Transaktion. Da der Abbruch einer Abfrage von der Art der abgespielten WAL-Datensätze abhängt, kann eine abgebrochene Abfrage durchaus erfolgreich sein, wenn sie erneut ausgeführt wird.

Beachten Sie, dass die Verzögerungsparameter mit der seit dem Empfang der WAL-Daten durch den Standby-Server verstrichenen Zeit verglichen werden. Die Schonfrist für eine einzelne Abfrage auf dem Standby beträgt daher nie mehr als den Verzögerungsparameter und kann deutlich kürzer sein, wenn der Standby bereits infolge des Wartens auf frühere Abfragen oder aufgrund einer hohen Aktualisierungslast zurückgefallen ist.

Der häufigste Grund für Konflikte zwischen Standby-Abfragen und WAL-Replay ist frühes Aufräumen. Normalerweise erlaubt PostgreSQL das Aufräumen alter Zeilenversionen, wenn keine Transaktionen existieren, die sie noch sehen müssen, um korrekte Datensichtbarkeit nach MVCC-Regeln zu gewährleisten. Diese Regel kann jedoch nur für Transaktionen angewendet werden, die auf dem Primary ausgeführt werden. Daher ist es möglich, dass Cleanup auf dem Primary Zeilenversionen entfernt, die für eine Transaktion auf dem Standby noch sichtbar sind.

Das Aufräumen von Zeilenversionen ist nicht die einzige mögliche Ursache für Konflikte mit Standby-Abfragen. Alle Index-Only-Scans, auch solche auf Standbys, müssen einen MVCC-Snapshot verwenden, der mit der Visibility Map übereinstimmt. Konflikte sind daher immer dann erforderlich, wenn `VACUUM` eine Seite in der Visibility Map als all-visible markiert, die eine oder mehrere Zeilen enthält, die nicht für alle Standby-Abfragen sichtbar sind. Selbst `VACUUM` auf einer Tabelle ohne aktualisierte oder gelöschte Zeilen, die Cleanup benötigen, kann daher zu Konflikten führen.

Benutzern sollte klar sein, dass Tabellen, die auf dem Primärserver regelmäßig und stark aktualisiert werden, schnell zum Abbruch länger laufender Abfragen auf dem Standby führen. In solchen Fällen kann das Setzen eines endlichen Werts für `max_standby_archive_delay` oder `max_standby_streaming_delay` ähnlich betrachtet werden wie das Setzen von `statement_timeout`.

Es gibt Abhilfemöglichkeiten, wenn die Zahl der Standby-Abfrageabbrüche als inakzeptabel empfunden wird. Die erste Option ist, den Parameter `hot_standby_feedback` zu setzen, der verhindert, dass `VACUUM` kürzlich tote Zeilen entfernt, sodass Cleanup-Konflikte nicht auftreten. Wenn Sie dies tun, sollten Sie beachten, dass dies das Aufräumen toter Zeilen auf dem Primary verzögert, was zu unerwünschter Tabellenaufblähung führen kann. Die Cleanup-Situation ist jedoch nicht schlechter, als wenn die Standby-Abfragen direkt auf dem Primärserver liefen, und Sie profitieren weiterhin von der Auslagerung der Ausführung auf den Standby. Wenn Standby-Server häufig verbinden und trennen, möchten Sie möglicherweise Anpassungen für Zeiträume vornehmen, in denen kein `hot_standby_feedback` geliefert wird. Ziehen Sie beispielsweise in Betracht, `max_standby_archive_delay` zu erhöhen, damit Abfragen während getrennter Zeiträume nicht schnell durch Konflikte in WAL-Archivdateien abgebrochen werden. Außerdem sollten Sie erwägen, `max_standby_streaming_delay` zu erhöhen, um schnelle Abbrüche durch neu eintreffende Streaming-WAL-Einträge nach Wiederverbindung zu vermeiden.

Die Zahl der Abfrageabbrüche und der Grund dafür können über die Systemsicht `pg_stat_database_conflicts` auf dem Standby-Server angezeigt werden. Die Systemsicht `pg_stat_database` enthält ebenfalls zusammenfassende Informationen.

Benutzer können steuern, ob eine Logmeldung erzeugt wird, wenn WAL-Replay länger als `deadlock_timeout` auf Konflikte wartet. Dies wird durch den Parameter `log_recovery_conflict_waits` gesteuert.

### 26.4.3. Überblick für Administratoren

Wenn `hot_standby` in `postgresql.conf` eingeschaltet ist, was der Standardwert ist, und eine Datei `standby.signal` vorhanden ist, läuft der Server im Hot-Standby-Modus. Es kann jedoch einige Zeit dauern, bis Hot-Standby-Verbindungen erlaubt werden, weil der Server keine Verbindungen annimmt, bis er genügend Recovery abgeschlossen hat, um einen konsistenten Zustand bereitzustellen, gegen den Abfragen laufen können. Während dieser Zeit werden Clients, die Verbindungen aufzubauen versuchen, mit einer Fehlermeldung abgewiesen. Um zu bestätigen, dass der Server hochgekommen ist, versuchen Sie entweder in einer Schleife aus der Anwendung heraus zu verbinden oder suchen Sie diese Meldungen in den Server-Logs:

```text
LOG:  entering standby mode

... then some time later ...

LOG:  consistent recovery state reached
LOG:  database system is ready to accept read-only connections
```

Konsistenzinformationen werden auf dem Primary einmal pro Checkpoint aufgezeichnet. Hot Standby kann nicht aktiviert werden, wenn WAL gelesen wird, das während einer Zeit geschrieben wurde, in der `wal_level` auf dem Primary nicht auf `replica` oder `logical` gesetzt war. Selbst nach Erreichen eines konsistenten Zustands ist der Recovery-Snapshot möglicherweise noch nicht für Hot Standby bereit, wenn beide der folgenden Bedingungen erfüllt sind; dadurch verzögert sich die Annahme schreibgeschützter Verbindungen. Um Hot Standby zu ermöglichen, müssen langlebige Schreibtransaktionen mit mehr als 64 Subtransaktionen auf dem Primary abgeschlossen sein.

- Eine Schreibtransaktion hat mehr als 64 Subtransaktionen.
- Sehr langlebige Schreibtransaktionen existieren.

Wenn Sie dateibasiertes Log Shipping (Warm Standby) ausführen, müssen Sie möglicherweise warten, bis die nächste WAL-Datei eintrifft, was so lange dauern kann wie die Einstellung `archive_timeout` auf dem Primary.

Die Einstellungen einiger Parameter bestimmen die Größe des Shared Memory zur Nachverfolgung von Transaktions-IDs, Sperren und vorbereiteten Transaktionen. Diese Shared-Memory-Strukturen dürfen auf einem Standby nicht kleiner sein als auf dem Primary, damit sichergestellt ist, dass dem Standby während der Recovery nicht der Shared Memory ausgeht. Wenn der Primary beispielsweise eine vorbereitete Transaktion verwendet hat, der Standby aber keinen Shared Memory zum Nachverfolgen vorbereiteter Transaktionen reserviert hat, könnte Recovery erst fortfahren, wenn die Konfiguration des Standby geändert wurde. Betroffen sind die Parameter:

- `max_connections`
- `max_prepared_transactions`
- `max_locks_per_transaction`
- `max_wal_senders`
- `max_worker_processes`

Der einfachste Weg, dieses Problem zu vermeiden, ist, diese Parameter auf den Standbys auf Werte zu setzen, die gleich oder größer sind als auf dem Primary. Wenn Sie diese Werte erhöhen möchten, sollten Sie das daher zuerst auf allen Standby-Servern tun, bevor Sie die Änderungen auf dem Primärserver anwenden. Umgekehrt sollten Sie, wenn Sie diese Werte verringern möchten, dies zuerst auf dem Primärserver tun, bevor Sie die Änderungen auf allen Standby-Servern anwenden. Denken Sie daran: Wenn ein Standby hochgestuft wird, wird er zur neuen Referenz für die erforderlichen Parametereinstellungen der ihm folgenden Standbys. Um zu vermeiden, dass dies bei Switchover oder Failover zum Problem wird, wird empfohlen, diese Einstellungen auf allen Standby-Servern gleich zu halten.

Das WAL verfolgt Änderungen an diesen Parametern auf dem Primary. Wenn ein Hot Standby WAL verarbeitet, das anzeigt, dass der aktuelle Wert auf dem Primary höher ist als sein eigener Wert, protokolliert er eine Warnung und pausiert Recovery, zum Beispiel:

```text
WARNING:  hot standby is not possible because of insufficient parameter settings
DETAIL:  max_connections = 80 is a lower setting than on the primary server, where its value was 100.
LOG:  recovery has paused
DETAIL:  If recovery is unpaused, the server will shut down.
HINT:  You can then restart the server after making the necessary configuration changes.
```

An diesem Punkt müssen die Einstellungen auf dem Standby aktualisiert und die Instanz neu gestartet werden, bevor Recovery fortgesetzt werden kann. Wenn der Standby kein Hot Standby ist, fährt er beim Auftreten der inkompatiblen Parameteränderung sofort herunter, ohne zu pausieren, da es dann keinen Nutzen hat, ihn weiterlaufen zu lassen.

Es ist wichtig, dass der Administrator passende Einstellungen für `max_standby_archive_delay` und `max_standby_streaming_delay` wählt. Die besten Werte hängen von geschäftlichen Prioritäten ab. Wenn der Server beispielsweise vor allem als Hochverfügbarkeitsserver dient, möchten Sie niedrige Verzögerungseinstellungen, vielleicht sogar null, obwohl das eine sehr aggressive Einstellung ist. Wenn der Standby-Server als zusätzlicher Server für Decision-Support-Abfragen dient, kann es akzeptabel sein, die maximalen Verzögerungswerte auf viele Stunden oder sogar auf `-1` zu setzen, was bedeutet, für den Abschluss von Abfragen unbegrenzt zu warten.

Auf dem Primary geschriebene Transaktionsstatus-Hint-Bits werden nicht im WAL protokolliert, daher schreiben Daten auf dem Standby diese Hinweise wahrscheinlich erneut auf dem Standby. Der Standby-Server führt also weiterhin Plattenschreibvorgänge aus, obwohl alle Benutzer schreibgeschützt sind; an den Datenwerten selbst ändern sich nichts. Benutzer schreiben weiterhin große temporäre Sortierdateien und erzeugen Relcache-Informationsdateien neu, sodass kein Teil der Datenbank im Hot-Standby-Modus wirklich schreibgeschützt ist. Beachten Sie außerdem, dass Schreibvorgänge auf entfernte Datenbanken über das Modul `dblink` und andere Operationen außerhalb der Datenbank über PL-Funktionen weiterhin möglich sind, obwohl die Transaktion lokal schreibgeschützt ist.

Die folgenden Arten von Administrationsbefehlen werden im Recovery-Modus nicht akzeptiert:

- Data Definition Language (DDL), z. B. `CREATE INDEX`
- Privilegien und Eigentümerschaft: `GRANT`, `REVOKE`, `REASSIGN`
- Wartungsbefehle: `ANALYZE`, `VACUUM`, `CLUSTER`, `REINDEX`

Beachten Sie erneut, dass einige dieser Befehle in schreibgeschützten Transaktionen auf dem Primary tatsächlich erlaubt sind.

Daher können Sie keine zusätzlichen Indizes erstellen, die nur auf dem Standby existieren, und keine Statistiken, die nur auf dem Standby existieren. Wenn diese Administrationsbefehle benötigt werden, sollten sie auf dem Primary ausgeführt werden; schließlich propagieren diese Änderungen zum Standby.

`pg_cancel_backend()` und `pg_terminate_backend()` funktionieren für Benutzer-Backends, aber nicht für den Startup-Prozess, der Recovery ausführt. `pg_stat_activity` zeigt Recovery-Transaktionen nicht als aktiv an. Daher ist `pg_prepared_xacts` während Recovery immer leer. Wenn Sie zweifelhafte vorbereitete Transaktionen auflösen möchten, betrachten Sie `pg_prepared_xacts` auf dem Primary und führen dort Befehle zur Auflösung von Transaktionen aus, oder lösen Sie sie nach dem Ende der Recovery auf.

`pg_locks` zeigt wie üblich von Backends gehaltene Sperren. `pg_locks` zeigt außerdem eine virtuelle Transaktion, die vom Startup-Prozess verwaltet wird und alle `AccessExclusiveLocks` besitzt, die von durch Recovery abgespielten Transaktionen gehalten werden. Beachten Sie, dass der Startup-Prozess keine Sperren nimmt, um Datenbankänderungen vorzunehmen, und daher andere Sperren als `AccessExclusiveLocks` für den Startup-Prozess nicht in `pg_locks` erscheinen; es wird lediglich angenommen, dass sie existieren.

Das Nagios-Plugin `check_pgsql` funktioniert, weil die einfachen Informationen, die es prüft, vorhanden sind. Das Monitoring-Skript `check_postgres` funktioniert ebenfalls, auch wenn einige gemeldete Werte abweichende oder verwirrende Ergebnisse liefern können. Beispielsweise wird die letzte Vacuum-Zeit nicht gepflegt, da auf dem Standby kein Vacuum stattfindet. Auf dem Primary laufende Vacuums senden ihre Änderungen dennoch an den Standby.

WAL-Dateisteuerungsbefehle wie `pg_backup_start`, `pg_switch_wal` usw. funktionieren während Recovery nicht.

Dynamisch ladbare Module funktionieren, einschließlich `pg_stat_statements`.

Advisory Locks funktionieren während Recovery normal, einschließlich Deadlock-Erkennung. Beachten Sie, dass Advisory Locks niemals im WAL protokolliert werden. Daher ist es unmöglich, dass ein Advisory Lock auf Primary oder Standby mit WAL-Replay in Konflikt steht. Ebenso ist es nicht möglich, einen Advisory Lock auf dem Primary zu erwerben und dadurch einen ähnlichen Advisory Lock auf dem Standby auszulösen. Advisory Locks beziehen sich nur auf den Server, auf dem sie erworben werden.

Triggerbasierte Replikationssysteme wie Slony, Londiste und Bucardo laufen auf dem Standby überhaupt nicht, laufen aber problemlos auf dem Primärserver, solange die Änderungen nicht an Standby-Server gesendet und dort angewendet werden. WAL-Replay ist nicht triggerbasiert, daher können Sie vom Standby nicht an ein System weiterleiten, das zusätzliche Datenbankschreibvorgänge erfordert oder auf Trigger angewiesen ist.

Neue OIDs können nicht zugewiesen werden, obwohl einige UUID-Generatoren weiterhin funktionieren können, solange sie nicht darauf angewiesen sind, neuen Status in die Datenbank zu schreiben.

Derzeit ist das Erstellen temporärer Tabellen während schreibgeschützter Transaktionen nicht erlaubt, sodass vorhandene Skripte in einigen Fällen nicht korrekt laufen. Diese Einschränkung könnte in einer späteren Version gelockert werden. Dies ist sowohl eine Frage der SQL-Standardkonformität als auch ein technisches Problem.

`DROP TABLESPACE` kann nur erfolgreich sein, wenn der Tablespace leer ist. Einige Standby-Benutzer verwenden den Tablespace möglicherweise aktiv über ihren Parameter `temp_tablespaces`. Wenn im Tablespace temporäre Dateien vorhanden sind, werden alle aktiven Abfragen abgebrochen, um sicherzustellen, dass temporäre Dateien entfernt werden, sodass der Tablespace entfernt und WAL-Replay fortgesetzt werden kann.

Das Ausführen von `DROP DATABASE` oder `ALTER DATABASE ... SET TABLESPACE` auf dem Primary erzeugt einen WAL-Eintrag, der bewirkt, dass alle Benutzer, die auf dem Standby mit dieser Datenbank verbunden sind, zwangsweise getrennt werden. Diese Aktion erfolgt sofort, unabhängig von der Einstellung von `max_standby_streaming_delay`. Beachten Sie, dass `ALTER DATABASE ... RENAME` Benutzer nicht trennt, was in den meisten Fällen unbemerkt bleibt, aber in einigen Fällen ein Programm verwirren kann, wenn es in irgendeiner Weise vom Datenbanknamen abhängt.

Im normalen Nicht-Recovery-Modus passiert nichts mit verbundenen Benutzern, wenn Sie `DROP USER` oder `DROP ROLE` für eine Rolle mit Login-Fähigkeit ausführen, während dieser Benutzer noch verbunden ist; er bleibt verbunden. Der Benutzer kann sich jedoch nicht erneut verbinden. Dieses Verhalten gilt auch in Recovery, sodass ein `DROP USER` auf dem Primary diesen Benutzer auf dem Standby nicht trennt.

Das kumulative Statistiksystem ist während Recovery aktiv. Alle Scans, Reads, Blöcke, Indexnutzung usw. werden auf dem Standby normal aufgezeichnet. WAL-Replay erhöht jedoch keine relations- und datenbankspezifischen Zähler. Das heißt, Replay erhöht keine Spalten von `pg_stat_all_tables` wie `n_tup_ins`; vom Startup-Prozess durchgeführte Reads oder Writes werden nicht in den `pg_statio_`-Sichten nachverfolgt, und zugehörige Spalten von `pg_stat_database` werden nicht erhöht.

Autovacuum ist während Recovery nicht aktiv. Es startet normal am Ende der Recovery.

Der Checkpointer-Prozess und der Background-Writer-Prozess sind während Recovery aktiv. Der Checkpointer-Prozess führt Restartpoints durch, ähnlich wie Checkpoints auf dem Primary, und der Background-Writer-Prozess führt normale Blockreinigungsaktivitäten aus. Dies kann Aktualisierungen der auf dem Standby-Server gespeicherten Hint-Bit-Informationen umfassen. Der Befehl `CHECKPOINT` wird während Recovery akzeptiert, führt aber einen Restartpoint statt eines neuen Checkpoints aus.

### 26.4.4. Parameterreferenz für Hot Standby

Verschiedene Parameter wurden oben in [Abschnitt 26.4.2](#2642-abfragekonflikte-behandeln) und [Abschnitt 26.4.3](#2643-überblick-für-administratoren) erwähnt.

Auf dem Primary kann der Parameter `wal_level` verwendet werden. `max_standby_archive_delay` und `max_standby_streaming_delay` haben keine Wirkung, wenn sie auf dem Primary gesetzt werden.

Auf dem Standby können die Parameter `hot_standby`, `max_standby_archive_delay` und `max_standby_streaming_delay` verwendet werden.

### 26.4.5. Einschränkungen

Hot Standby hat mehrere Einschränkungen. Diese können und werden wahrscheinlich in künftigen Versionen behoben:

- Vollständige Kenntnis laufender Transaktionen ist erforderlich, bevor Snapshots genommen werden können. Transaktionen, die eine große Anzahl von Subtransaktionen verwenden, derzeit mehr als 64, verzögern den Start schreibgeschützter Verbindungen bis zum Abschluss der am längsten laufenden Schreibtransaktion. Wenn diese Situation eintritt, werden erklärende Meldungen an das Server-Log gesendet.

- Gültige Startpunkte für Standby-Abfragen werden bei jedem Checkpoint auf dem Primary erzeugt. Wenn der Standby heruntergefahren wird, während der Primary in einem Shutdown-Zustand ist, ist es möglicherweise nicht möglich, wieder in Hot Standby einzutreten, bis der Primary gestartet wurde und weitere Startpunkte in den WAL-Logs erzeugt. Diese Situation ist in den häufigsten Fällen, in denen sie auftreten kann, kein Problem. Wenn der Primary heruntergefahren und nicht mehr verfügbar ist, liegt dies im Allgemeinen wahrscheinlich an einem schweren Fehler, der ohnehin erfordert, den Standby zum neuen Primary zu machen. Und in Situationen, in denen der Primary absichtlich heruntergefahren wird, gehört es ebenfalls zum Standardverfahren, dafür zu sorgen, dass der Standby reibungslos zum neuen Primary wird.

- Am Ende der Recovery benötigen von vorbereiteten Transaktionen gehaltene `AccessExclusiveLocks` doppelt so viele Lock-Tabellen-Einträge wie normal. Wenn Sie planen, entweder eine große Anzahl gleichzeitiger vorbereiteter Transaktionen auszuführen, die normalerweise `AccessExclusiveLocks` nehmen, oder eine große Transaktion zu verwenden, die viele `AccessExclusiveLocks` nimmt, wird empfohlen, einen größeren Wert für `max_locks_per_transaction` zu wählen, möglicherweise bis zum Doppelten des Werts auf dem Primärserver. Sie müssen dies überhaupt nicht berücksichtigen, wenn Ihre Einstellung von `max_prepared_transactions` `0` ist.

- Die Transaktionsisolationsstufe Serializable ist in Hot Standby noch nicht verfügbar. Details finden Sie in Abschnitt 13.2.3 und Abschnitt 13.4.1. Der Versuch, eine Transaktion im Hot-Standby-Modus auf die Isolationsstufe Serializable zu setzen, erzeugt einen Fehler.
