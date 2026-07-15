# 25. Sicherung und Wiederherstellung

Wie alles, was wertvolle Daten enthält, sollten PostgreSQL-Datenbanken regelmäßig gesichert werden. Das Verfahren ist im Kern einfach, aber es ist wichtig, die zugrunde liegenden Techniken und Annahmen klar zu verstehen.

Es gibt drei grundsätzlich verschiedene Ansätze zur Sicherung von PostgreSQL-Daten:

- SQL-Dump
- Sicherung auf Dateisystemebene
- Kontinuierliche Archivierung

Jeder Ansatz hat eigene Stärken und Schwächen; die folgenden Abschnitte behandeln sie nacheinander.

## 25.1. SQL-Dump

Die Idee hinter dieser Dump-Methode ist, eine Datei mit SQL-Befehlen zu erzeugen, die die Datenbank beim erneuten Einspielen auf dem Server in demselben Zustand wiederherstellen, in dem sie sich zum Zeitpunkt des Dumps befand. PostgreSQL stellt dafür das Hilfsprogramm `pg_dump` bereit. Die Grundform dieses Befehls lautet:

```text
pg_dump dbname > dumpfile
```

Wie zu sehen ist, schreibt `pg_dump` sein Ergebnis auf die Standardausgabe. Weiter unten wird deutlich, warum das nützlich sein kann. Der obige Befehl erzeugt zwar eine Textdatei, `pg_dump` kann aber auch Dateien in anderen Formaten erzeugen, die Parallelität und eine feinere Steuerung der Objektwiederherstellung erlauben.

`pg_dump` ist eine normale PostgreSQL-Clientanwendung, wenn auch eine besonders ausgeklügelte. Das bedeutet, dass Sie dieses Sicherungsverfahren von jedem entfernten Rechner aus durchführen können, der Zugriff auf die Datenbank hat. Denken Sie aber daran, dass `pg_dump` nicht mit besonderen Rechten arbeitet. Insbesondere muss es Lesezugriff auf alle Tabellen haben, die Sie sichern möchten. Um die gesamte Datenbank zu sichern, müssen Sie es daher fast immer als Datenbank-Superuser ausführen. Wenn Sie nicht genügend Rechte haben, um die ganze Datenbank zu sichern, können Sie mit Optionen wie `-n schema` oder `-t table` dennoch die Teile der Datenbank sichern, auf die Sie Zugriff haben.

Um anzugeben, welchen Datenbankserver `pg_dump` kontaktieren soll, verwenden Sie die Kommandozeilenoptionen `-h host` und `-p port`. Der Standardhost ist der lokale Rechner oder der Wert der Umgebungsvariablen `PGHOST`. Entsprechend wird der Standardport durch die Umgebungsvariable `PGPORT` oder, falls diese nicht gesetzt ist, durch den einkompilierten Standardwert bestimmt. Praktischerweise hat der Server normalerweise denselben einkompilierten Standardwert.

Wie jede andere PostgreSQL-Clientanwendung verbindet sich `pg_dump` standardmäßig mit dem Datenbankbenutzernamen, der dem aktuellen Betriebssystembenutzernamen entspricht. Um dies zu überschreiben, geben Sie entweder die Option `-U` an oder setzen die Umgebungsvariable `PGUSER`. Beachten Sie, dass `pg_dump`-Verbindungen den normalen Client-Authentifizierungsmechanismen unterliegen, die in [Kapitel 20](20_Client_Authentifizierung.md) beschrieben sind.

Ein wichtiger Vorteil von `pg_dump` gegenüber den später beschriebenen Sicherungsmethoden besteht darin, dass die Ausgabe von `pg_dump` im Allgemeinen in neuere PostgreSQL-Versionen wieder eingespielt werden kann, während Sicherungen auf Dateiebene und kontinuierliche Archivierung beide stark von der Serverversion abhängen. `pg_dump` ist außerdem die einzige Methode, die beim Übertragen einer Datenbank auf eine andere Maschinenarchitektur funktioniert, etwa beim Wechsel von einem 32-Bit- auf einen 64-Bit-Server.

Von `pg_dump` erzeugte Dumps sind intern konsistent. Das heißt, der Dump stellt einen Snapshot der Datenbank zu dem Zeitpunkt dar, an dem `pg_dump` gestartet wurde. `pg_dump` blockiert während seiner Arbeit keine anderen Operationen auf der Datenbank. Ausnahmen sind Operationen, die eine exklusive Sperre benötigen, etwa die meisten Formen von `ALTER TABLE`.

### 25.1.1. Den Dump wiederherstellen

Von `pg_dump` erzeugte Textdateien sind dafür gedacht, mit dem Programm `psql` und dessen Standardeinstellungen gelesen zu werden. Die allgemeine Befehlsform zur Wiederherstellung eines Text-Dumps lautet:

```text
psql -X dbname < dumpfile
```

Dabei ist `dumpfile` die vom Befehl `pg_dump` ausgegebene Datei. Die Datenbank `dbname` wird durch diesen Befehl nicht erzeugt; Sie müssen sie daher vor dem Ausführen von `psql` selbst aus `template0` erstellen, zum Beispiel mit `createdb -T template0 dbname`. Damit `psql` mit seinen Standardeinstellungen läuft, verwenden Sie die Option `-X` (`--no-psqlrc`). `psql` unterstützt ähnliche Optionen wie `pg_dump`, um den Zielserver und den zu verwendenden Benutzernamen anzugeben. Weitere Informationen finden Sie auf der Referenzseite zu `psql`.

Dumps in Nicht-Textformaten sollten mit dem Hilfsprogramm `pg_restore` wiederhergestellt werden.

Vor dem Wiederherstellen eines SQL-Dumps müssen alle Benutzer, die Objekte besitzen oder denen Berechtigungen auf Objekte in der gesicherten Datenbank gewährt wurden, bereits existieren. Andernfalls kann die Wiederherstellung die Objekte nicht mit den ursprünglichen Eigentümern und/oder Berechtigungen neu erzeugen. Manchmal ist genau das gewünscht, normalerweise aber nicht.

Standardmäßig führt das `psql`-Skript seine Ausführung nach einem SQL-Fehler fort. Sie können `psql` mit gesetzter Variable `ON_ERROR_STOP` ausführen, um dieses Verhalten zu ändern und `psql` bei einem SQL-Fehler mit Exit-Status 3 beenden zu lassen:

```text
psql -X --set ON_ERROR_STOP=on dbname < dumpfile
```

In beiden Fällen erhalten Sie andernfalls nur eine teilweise wiederhergestellte Datenbank. Alternativ können Sie festlegen, dass der gesamte Dump als eine einzige Transaktion wiederhergestellt wird, sodass die Wiederherstellung entweder vollständig abgeschlossen oder vollständig zurückgerollt wird. Dieser Modus wird mit den Kommandozeilenoptionen `-1` oder `--single-transaction` von `psql` aktiviert. Beachten Sie dabei, dass schon ein kleiner Fehler eine Wiederherstellung zurückrollen kann, die bereits viele Stunden gelaufen ist. Das kann dennoch besser sein, als eine komplexe Datenbank nach einem teilweise eingespielten Dump manuell aufzuräumen.

Die Fähigkeit von `pg_dump` und `psql`, in Pipes zu schreiben beziehungsweise aus ihnen zu lesen, macht es möglich, eine Datenbank direkt von einem Server auf einen anderen zu dumpen, zum Beispiel:

```text
pg_dump -h host1 dbname | psql -X -h host2 dbname
```

> **Wichtig**
>
> Die von `pg_dump` erzeugten Dumps beziehen sich auf `template0`. Das bedeutet, dass Sprachen, Prozeduren usw., die über `template1` hinzugefügt wurden, ebenfalls von `pg_dump` ausgegeben werden. Wenn Sie also mit einem angepassten `template1` arbeiten, müssen Sie beim Wiederherstellen die leere Datenbank wie im obigen Beispiel aus `template0` erzeugen.

Nach dem Wiederherstellen einer Sicherung ist es sinnvoll, für jede Datenbank `ANALYZE` auszuführen, damit der Query Optimizer brauchbare Statistiken hat; weitere Informationen finden Sie in [Abschnitt 24.1.3](24_Regelmäßige_Datenbankwartung.md#2413-plannerstatistiken-aktualisieren) und [Abschnitt 24.1.6](24_Regelmäßige_Datenbankwartung.md#2416-der-autovacuumdaemon). Weitere Hinweise zum effizienten Laden großer Datenmengen in PostgreSQL finden Sie in [Abschnitt 14.4](14_Hinweise_zur_Performance.md#144-eine-datenbank-befüllen).

### 25.1.2. `pg_dumpall` verwenden

`pg_dump` sichert immer nur eine Datenbank auf einmal und sichert keine Informationen über Rollen oder Tablespaces, da diese clusterweit und nicht datenbankspezifisch sind. Um das bequeme Sichern des gesamten Inhalts eines Datenbankclusters zu ermöglichen, gibt es das Programm `pg_dumpall`. `pg_dumpall` sichert jede Datenbank in einem angegebenen Cluster und bewahrt außerdem clusterweite Daten wie Rollen- und Tablespace-Definitionen. Die Grundform dieses Befehls lautet:

```text
pg_dumpall > dumpfile
```

Der resultierende Dump kann mit `psql` wiederhergestellt werden:

```text
psql -X -f dumpfile postgres
```

Tatsächlich können Sie zum Starten jeden vorhandenen Datenbanknamen angeben; wenn Sie jedoch in einen leeren Cluster laden, sollte normalerweise `postgres` verwendet werden. Beim Wiederherstellen eines `pg_dumpall`-Dumps ist immer Datenbank-Superuser-Zugriff erforderlich, weil dies zum Wiederherstellen der Rollen- und Tablespace-Informationen nötig ist. Wenn Sie Tablespaces verwenden, stellen Sie sicher, dass die Tablespace-Pfade im Dump für die neue Installation geeignet sind.

`pg_dumpall` erzeugt Befehle, mit denen Rollen, Tablespaces und leere Datenbanken neu erstellt werden, und ruft anschließend `pg_dump` für jede Datenbank auf. Das bedeutet: Jede einzelne Datenbank ist intern konsistent, aber die Snapshots verschiedener Datenbanken sind nicht synchronisiert.

Clusterweite Daten können allein mit der Option `pg_dumpall --globals-only` ausgegeben werden. Dies ist erforderlich, um den Cluster vollständig zu sichern, wenn der Befehl `pg_dump` für einzelne Datenbanken verwendet wird.

### 25.1.3. Große Datenbanken handhaben

Einige Betriebssysteme haben maximale Dateigrößen, die beim Erzeugen großer `pg_dump`-Ausgabedateien Probleme verursachen. Glücklicherweise kann `pg_dump` auf die Standardausgabe schreiben, sodass Sie normale Unix-Werkzeuge verwenden können, um dieses mögliche Problem zu umgehen. Es gibt mehrere Methoden.

Komprimierte Dumps verwenden. Sie können Ihr bevorzugtes Kompressionsprogramm verwenden, zum Beispiel `gzip`:

```text
pg_dump dbname | gzip > filename.gz
```

Wieder einspielen mit:

```text
gunzip -c filename.gz | psql dbname
```

oder:

```text
cat filename.gz | gunzip | psql dbname
```

`split` verwenden. Der Befehl `split` ermöglicht es, die Ausgabe in kleinere Dateien aufzuteilen, deren Größe für das zugrunde liegende Dateisystem akzeptabel ist. Um beispielsweise 2-Gigabyte-Stücke zu erzeugen:

```text
pg_dump dbname | split -b 2G - filename
```

Wieder einspielen mit:

```text
cat filename* | psql dbname
```

Wenn GNU `split` verwendet wird, lässt es sich mit `gzip` kombinieren:

```text
pg_dump dbname | split -b 2G --filter='gzip > $FILE.gz'
```

Die Wiederherstellung kann mit `zcat` erfolgen.

Das Custom-Dump-Format von `pg_dump` verwenden. Wenn PostgreSQL auf einem System gebaut wurde, auf dem die Kompressionsbibliothek `zlib` installiert ist, komprimiert das Custom-Dump-Format die Daten beim Schreiben in die Ausgabedatei. Dadurch entstehen Dump-Dateien in ähnlicher Größe wie bei der Verwendung von `gzip`; zusätzlich hat dieses Format den Vorteil, dass Tabellen selektiv wiederhergestellt werden können. Der folgende Befehl dumpt eine Datenbank im Custom-Dump-Format:

```text
pg_dump -Fc dbname > filename
```

Ein Dump im Custom-Format ist kein Skript für `psql`, sondern muss mit `pg_restore` wiederhergestellt werden, zum Beispiel:

```text
pg_restore -d dbname filename
```

Details finden Sie auf den Referenzseiten zu `pg_dump` und `pg_restore`.

Bei sehr großen Datenbanken müssen Sie `split` möglicherweise mit einem der beiden anderen Ansätze kombinieren.

Die parallele Dump-Funktion von `pg_dump` verwenden. Um den Dump einer großen Datenbank zu beschleunigen, können Sie den parallelen Modus von `pg_dump` nutzen. Dabei werden mehrere Tabellen gleichzeitig gedumpt. Den Grad der Parallelität steuern Sie mit dem Parameter `-j`. Parallele Dumps werden nur für das Archivformat `directory` unterstützt.

```text
pg_dump -j num -F d -f out.dir dbname
```

Sie können `pg_restore -j` verwenden, um einen Dump parallel wiederherzustellen. Das funktioniert für jedes Archiv im `custom`- oder `directory`-Archivmodus, unabhängig davon, ob es mit `pg_dump -j` erzeugt wurde.

## 25.2. Sicherung auf Dateisystemebene

Eine alternative Sicherungsstrategie besteht darin, die Dateien direkt zu kopieren, die PostgreSQL zum Speichern der Daten in der Datenbank verwendet; [Abschnitt 18.2](18_Servereinrichtung_und_Betrieb.md#182-einen-datenbankcluster-erstellen) erklärt, wo sich diese Dateien befinden. Sie können jede beliebige Methode für Dateisystemsicherungen verwenden, zum Beispiel:

```text
tar -cf backup.tar /usr/local/pgsql/data
```

Es gibt jedoch zwei Einschränkungen, die diese Methode unpraktisch oder zumindest der `pg_dump`-Methode unterlegen machen:

1. Der Datenbankserver muss heruntergefahren sein, damit eine brauchbare Sicherung entsteht. Halbmaßnahmen wie das Unterbinden aller Verbindungen funktionieren nicht, teilweise weil `tar` und ähnliche Werkzeuge keinen atomaren Snapshot des Dateisystemzustands erstellen, aber auch wegen interner Pufferung im Server. Informationen zum Stoppen des Servers finden Sie in [Abschnitt 18.5](18_Servereinrichtung_und_Betrieb.md#185-den-server-herunterfahren). Selbstverständlich müssen Sie den Server auch vor dem Wiederherstellen der Daten herunterfahren.

2. Wenn Sie sich mit den Details des Dateisystemlayouts der Datenbank beschäftigt haben, könnten Sie versucht sein, nur bestimmte einzelne Tabellen oder Datenbanken aus ihren jeweiligen Dateien oder Verzeichnissen zu sichern oder wiederherzustellen. Das funktioniert nicht, weil die in diesen Dateien enthaltenen Informationen ohne die Commit-Log-Dateien `pg_xact/*`, die den Commit-Status aller Transaktionen enthalten, nicht nutzbar sind. Eine Tabellendatei ist nur mit diesen Informationen verwendbar. Natürlich ist es auch unmöglich, nur eine Tabelle und die zugehörigen `pg_xact`-Daten wiederherzustellen, weil dadurch alle anderen Tabellen im Datenbankcluster unbrauchbar würden. Dateisystemsicherungen funktionieren daher nur für die vollständige Sicherung und Wiederherstellung eines gesamten Datenbankclusters.

Ein alternativer Ansatz für Dateisystemsicherungen besteht darin, einen konsistenten Snapshot des Datenverzeichnisses zu erstellen, sofern das Dateisystem diese Funktion unterstützt und Sie darauf vertrauen, dass sie korrekt implementiert ist. Das typische Verfahren besteht darin, einen eingefrorenen Snapshot des Volumes zu erstellen, das die Datenbank enthält, anschließend das gesamte Datenverzeichnis aus dem Snapshot auf ein Sicherungsmedium zu kopieren, nicht nur Teile davon, und danach den eingefrorenen Snapshot freizugeben. Das funktioniert auch, während der Datenbankserver läuft. Eine auf diese Weise erzeugte Sicherung speichert die Datenbankdateien jedoch in einem Zustand, als wäre der Datenbankserver nicht ordnungsgemäß heruntergefahren worden. Wenn Sie den Datenbankserver auf den gesicherten Daten starten, nimmt er daher an, dass die vorherige Serverinstanz abgestürzt ist, und spielt das WAL-Log erneut ab. Das ist kein Problem; man sollte es nur wissen und sicherstellen, dass die WAL-Dateien in der Sicherung enthalten sind. Sie können vor dem Erzeugen des Snapshots ein `CHECKPOINT` ausführen, um die Recovery-Zeit zu verringern.

Wenn Ihre Datenbank über mehrere Dateisysteme verteilt ist, gibt es möglicherweise keine Möglichkeit, exakt gleichzeitige eingefrorene Snapshots aller Volumes zu erhalten. Wenn beispielsweise Ihre Datendateien und das WAL-Log auf unterschiedlichen Platten liegen oder Tablespaces auf verschiedenen Dateisystemen, kann eine Snapshot-Sicherung unmöglich sein, weil die Snapshots gleichzeitig erfolgen müssen. Lesen Sie die Dokumentation Ihres Dateisystems sehr sorgfältig, bevor Sie in solchen Situationen auf die Technik konsistenter Snapshots vertrauen.

Wenn gleichzeitige Snapshots nicht möglich sind, besteht eine Möglichkeit darin, den Datenbankserver lange genug herunterzufahren, um alle eingefrorenen Snapshots einzurichten. Eine andere Möglichkeit ist eine Basissicherung mit kontinuierlicher Archivierung ([Abschnitt 25.3.2](#2532-eine-basissicherung-erstellen)), weil solche Sicherungen gegenüber Dateisystemänderungen während der Sicherung unempfindlich sind. Dafür muss die kontinuierliche Archivierung nur während des Sicherungsvorgangs aktiviert werden; die Wiederherstellung erfolgt über Recovery aus einem kontinuierlichen Archiv ([Abschnitt 25.3.5](#2535-wiederherstellung-mit-einer-sicherung-aus-kontinuierlichem-archiv)).

Eine weitere Möglichkeit ist, `rsync` für eine Dateisystemsicherung zu verwenden. Dazu wird zuerst `rsync` ausgeführt, während der Datenbankserver läuft, und der Datenbankserver anschließend nur lange genug heruntergefahren, um ein `rsync --checksum` durchzuführen. `--checksum` ist erforderlich, weil `rsync` bei Änderungszeiten von Dateien nur eine Granularität von einer Sekunde hat. Das zweite `rsync` ist schneller als das erste, weil relativ wenig Daten übertragen werden müssen, und das Endergebnis ist konsistent, weil der Server heruntergefahren war. Diese Methode ermöglicht eine Dateisystemsicherung mit minimaler Ausfallzeit.

Beachten Sie, dass eine Dateisystemsicherung typischerweise größer ist als ein SQL-Dump. `pg_dump` muss beispielsweise nicht den Inhalt von Indizes ausgeben, sondern nur die Befehle zu ihrer Neuerzeugung. Eine Dateisystemsicherung kann jedoch schneller sein.

## 25.3. Kontinuierliche Archivierung und Point-in-Time-Recovery (PITR)

PostgreSQL führt jederzeit ein Write-Ahead-Log (WAL) im Unterverzeichnis `pg_wal/` des Datenverzeichnisses des Clusters. Das Log zeichnet jede Änderung an den Datendateien der Datenbank auf. Dieses Log dient in erster Linie der Absturzsicherheit: Wenn das System abstürzt, kann die Datenbank wieder in einen konsistenten Zustand gebracht werden, indem die Log-Einträge seit dem letzten Checkpoint erneut abgespielt werden. Die Existenz dieses Logs ermöglicht aber auch eine dritte Strategie zur Datenbanksicherung: Man kombiniert eine Sicherung auf Dateisystemebene mit der Sicherung der WAL-Dateien. Wenn eine Wiederherstellung erforderlich ist, stellt man die Dateisystemsicherung wieder her und spielt anschließend die gesicherten WAL-Dateien ein, um das System auf einen aktuellen Stand zu bringen. Dieser Ansatz ist administrativ komplexer als die beiden vorherigen, hat aber einige erhebliche Vorteile:

- Wir benötigen keine perfekt konsistente Dateisystemsicherung als Ausgangspunkt. Jede interne Inkonsistenz in der Sicherung wird durch das erneute Abspielen des Logs korrigiert; das unterscheidet sich nicht wesentlich von dem, was bei einer Crash-Recovery geschieht. Daher brauchen wir keine Snapshot-Fähigkeit des Dateisystems, sondern nur `tar` oder ein ähnliches Archivierungswerkzeug.

- Da wir eine beliebig lange Folge von WAL-Dateien zum erneuten Abspielen kombinieren können, lässt sich eine kontinuierliche Sicherung einfach dadurch erreichen, dass die WAL-Dateien fortlaufend archiviert werden. Das ist besonders wertvoll bei großen Datenbanken, bei denen häufige vollständige Sicherungen unpraktisch sein können.

- Es ist nicht notwendig, die WAL-Einträge bis ganz zum Ende erneut abzuspielen. Wir können das Abspielen an einem beliebigen Punkt stoppen und erhalten einen konsistenten Snapshot der Datenbank, wie sie zu diesem Zeitpunkt war. Diese Technik unterstützt also Point-in-Time-Recovery: Die Datenbank kann auf ihren Zustand zu jedem Zeitpunkt seit der Basissicherung zurückgebracht werden.

- Wenn wir die Reihe der WAL-Dateien kontinuierlich an eine andere Maschine liefern, auf der dieselbe Basissicherung geladen wurde, erhalten wir ein Warm-Standby-System: Zu jedem Zeitpunkt können wir die zweite Maschine hochfahren, und sie hat eine nahezu aktuelle Kopie der Datenbank.

> **Hinweis**
>
> `pg_dump` und `pg_dumpall` erzeugen keine Sicherungen auf Dateisystemebene und können nicht als Teil einer Lösung mit kontinuierlicher Archivierung verwendet werden. Solche Dumps sind logisch und enthalten nicht genügend Informationen, um durch WAL-Replay genutzt zu werden.

Wie die einfache Dateisicherungstechnik unterstützt diese Methode nur die Wiederherstellung eines gesamten Datenbankclusters, nicht einer Teilmenge. Außerdem benötigt sie viel Archivspeicher: Die Basissicherung kann umfangreich sein, und ein stark beschäftigtes System erzeugt viele Megabyte WAL-Verkehr, die archiviert werden müssen. Trotzdem ist dies in vielen Situationen, in denen hohe Zuverlässigkeit erforderlich ist, die bevorzugte Sicherungstechnik.

Für eine erfolgreiche Wiederherstellung mit kontinuierlicher Archivierung, von vielen Datenbankanbietern auch Online-Backup genannt, benötigen Sie eine durchgehende Folge archivierter WAL-Dateien, die mindestens bis zum Startzeitpunkt Ihrer Sicherung zurückreicht. Daher sollten Sie zuerst Ihr Verfahren zur Archivierung von WAL-Dateien einrichten und testen, bevor Sie die erste Basissicherung erstellen. Entsprechend behandeln wir zunächst die Mechanik der WAL-Archivierung.

### 25.3.1. WAL-Archivierung einrichten

Abstrakt betrachtet erzeugt ein laufendes PostgreSQL-System eine unbegrenzt lange Folge von WAL-Datensätzen. Das System teilt diese Folge physisch in WAL-Segmentdateien auf, die normalerweise jeweils 16 MB groß sind, wobei die Segmentgröße während `initdb` geändert werden kann. Die Segmentdateien erhalten numerische Namen, die ihre Position in der abstrakten WAL-Folge widerspiegeln. Wenn keine WAL-Archivierung verwendet wird, erzeugt das System normalerweise nur einige Segmentdateien und recycelt sie dann, indem es nicht mehr benötigte Segmentdateien auf höhere Segmentnummern umbenennt. Es wird angenommen, dass Segmentdateien, deren Inhalt vor dem letzten Checkpoint liegt, nicht mehr von Interesse sind und recycelt werden können.

Beim Archivieren von WAL-Daten müssen wir den Inhalt jeder Segmentdatei erfassen, sobald sie gefüllt ist, und diese Daten irgendwo speichern, bevor die Segmentdatei zur Wiederverwendung recycelt wird. Je nach Anwendung und verfügbarer Hardware kann es viele verschiedene Wege geben, die Daten irgendwo zu speichern: Wir könnten die Segmentdateien in ein per NFS eingebundenes Verzeichnis auf einer anderen Maschine kopieren, sie auf ein Bandlaufwerk schreiben, wobei der ursprüngliche Name jeder Datei identifizierbar bleiben muss, sie sammeln und auf CDs brennen oder etwas ganz anderes tun. Um dem Datenbankadministrator Flexibilität zu geben, versucht PostgreSQL, keine Annahmen darüber zu treffen, wie archiviert wird. Stattdessen kann der Administrator einen Shell-Befehl oder eine Archivbibliothek angeben, die ausgeführt wird, um eine abgeschlossene Segmentdatei an ihren Zielort zu kopieren. Das kann ein einfacher Shell-Befehl mit `cp` sein oder eine komplexe C-Funktion aufrufen; die Entscheidung liegt bei Ihnen.

Um WAL-Archivierung zu aktivieren, setzen Sie den Konfigurationsparameter `wal_level` auf `replica` oder höher, `archive_mode` auf `on` und geben entweder den zu verwendenden Shell-Befehl im Konfigurationsparameter `archive_command` oder die zu verwendende Bibliothek im Konfigurationsparameter `archive_library` an. In der Praxis werden diese Einstellungen immer in der Datei `postgresql.conf` stehen.

In `archive_command` wird `%p` durch den Pfadnamen der zu archivierenden Datei ersetzt, während `%f` nur durch den Dateinamen ersetzt wird. Der Pfadname ist relativ zum aktuellen Arbeitsverzeichnis, also zum Datenverzeichnis des Clusters. Verwenden Sie `%%`, wenn Sie ein echtes Prozentzeichen in den Befehl einbetten müssen. Der einfachste nützliche Befehl sieht etwa so aus:

```text
archive_command = 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f' # Unix
archive_command = 'copy "%p" "C:\\server\\archivedir\\%f"' # Windows
```

Damit werden archivierbare WAL-Segmente in das Verzeichnis `/mnt/server/archivedir` kopiert. Dies ist ein Beispiel, keine Empfehlung, und funktioniert möglicherweise nicht auf allen Plattformen. Nachdem die Parameter `%p` und `%f` ersetzt wurden, könnte der tatsächlich ausgeführte Befehl so aussehen:

```text
test ! -f /mnt/server/archivedir/00000001000000A900000065 && cp pg_wal/00000001000000A900000065 /mnt/server/archivedir/00000001000000A900000065
```

Für jede neu zu archivierende Datei wird ein ähnlicher Befehl erzeugt.

Der Archivierungsbefehl wird unter demselben Benutzer ausgeführt, unter dem auch der PostgreSQL-Server läuft. Da die archivierte Folge von WAL-Dateien praktisch alles in Ihrer Datenbank enthält, sollten Sie sicherstellen, dass die archivierten Daten vor neugierigen Blicken geschützt sind, zum Beispiel indem Sie in ein Verzeichnis archivieren, das keine Lesezugriffe für Gruppe oder Welt erlaubt.

Es ist wichtig, dass der Archivierungsbefehl genau dann den Exit-Status null zurückgibt, wenn er erfolgreich war. Bei einem Ergebnis null nimmt PostgreSQL an, dass die Datei erfolgreich archiviert wurde, und entfernt oder recycelt sie. Ein von null verschiedener Status teilt PostgreSQL dagegen mit, dass die Datei nicht archiviert wurde; PostgreSQL versucht es dann regelmäßig erneut, bis es gelingt.

Eine andere Archivierungsmethode besteht darin, ein eigenes Archivmodul als `archive_library` zu verwenden. Da solche Module in C geschrieben werden, kann das Erstellen eines eigenen Moduls deutlich mehr Aufwand erfordern als das Schreiben eines Shell-Befehls. Archivmodule können jedoch leistungsfähiger sein als die Archivierung über die Shell und haben Zugriff auf viele nützliche Serverressourcen. Weitere Informationen über Archivmodule finden Sie in [Kapitel 49](49_Archivmodule.md).

Wenn der Archivierungsbefehl durch ein Signal beendet wird, außer `SIGTERM`, das als Teil eines Server-Shutdowns verwendet wird, oder durch einen Shell-Fehler mit einem Exit-Status größer als 125, etwa weil ein Befehl nicht gefunden wurde, oder wenn die Archivfunktion ein `ERROR` oder `FATAL` ausgibt, bricht der Archiver-Prozess ab und wird vom Postmaster neu gestartet. In solchen Fällen wird der Fehler nicht in `pg_stat_archiver` gemeldet.

Archivierungsbefehle und -bibliotheken sollten im Allgemeinen so entworfen werden, dass sie das Überschreiben bereits vorhandener Archivdateien verweigern. Dies ist eine wichtige Sicherheitsfunktion, um die Integrität Ihres Archivs bei Administratorfehlern zu bewahren, etwa wenn die Ausgabe zweier verschiedener Server in dasselbe Archivverzeichnis geleitet wird. Es ist ratsam, die vorgeschlagene Archivbibliothek zu testen, um sicherzustellen, dass sie keine vorhandene Datei überschreibt.

In seltenen Fällen kann PostgreSQL versuchen, eine WAL-Datei erneut zu archivieren, die zuvor bereits archiviert wurde. Wenn das System beispielsweise abstürzt, bevor der Server den Archivierungserfolg dauerhaft aufgezeichnet hat, versucht der Server nach dem Neustart, die Datei erneut zu archivieren, sofern die Archivierung weiterhin aktiviert ist. Wenn ein Archivierungsbefehl oder eine Archivbibliothek auf eine bereits vorhandene Datei trifft, sollte er beziehungsweise sie den Status null beziehungsweise `true` zurückgeben, wenn die WAL-Datei denselben Inhalt hat wie das vorhandene Archiv und das vorhandene Archiv vollständig dauerhaft gespeichert wurde. Wenn eine vorhandene Datei andere Inhalte enthält als die zu archivierende WAL-Datei, muss der Archivierungsbefehl beziehungsweise die Archivbibliothek einen von null verschiedenen Status beziehungsweise `false` zurückgeben.

Der obige Beispielbefehl für Unix vermeidet das Überschreiben eines vorhandenen Archivs durch einen separaten Testschritt. Auf einigen Unix-Plattformen hat `cp` Schalter wie `-i`, die dasselbe mit weniger Text leisten können. Darauf sollten Sie sich jedoch nicht verlassen, ohne zu prüfen, ob der richtige Exit-Status zurückgegeben wird. Insbesondere gibt GNU `cp` mit `-i` den Status null zurück, wenn die Zieldatei bereits existiert, was nicht das gewünschte Verhalten ist.

Berücksichtigen Sie beim Entwerfen Ihrer Archivierung, was passiert, wenn der Archivierungsbefehl oder die Archivbibliothek wiederholt fehlschlägt, weil ein Aspekt Bedienereingriff erfordert oder das Archiv voll läuft. Dies kann zum Beispiel passieren, wenn auf Band ohne Autoloader geschrieben wird; ist das Band voll, kann nichts weiter archiviert werden, bis das Band gewechselt wurde. Sie sollten sicherstellen, dass jeder Fehlerzustand oder jede Aufforderung an einen menschlichen Operator angemessen gemeldet wird, damit die Situation in vertretbarer Zeit gelöst werden kann. Das Verzeichnis `pg_wal/` füllt sich weiter mit WAL-Segmentdateien, bis die Situation behoben ist. Wenn das Dateisystem, das `pg_wal/` enthält, voll läuft, führt PostgreSQL einen `PANIC`-Shutdown durch. Keine committeten Transaktionen gehen verloren, aber die Datenbank bleibt offline, bis Sie Speicherplatz freigeben.

Die Geschwindigkeit des Archivierungsbefehls oder der Archivbibliothek ist unwichtig, solange sie mit der durchschnittlichen Rate mithalten kann, mit der Ihr Server WAL-Daten erzeugt. Der normale Betrieb geht weiter, auch wenn die Archivierung etwas zurückfällt. Wenn die Archivierung deutlich zurückfällt, erhöht sich die Datenmenge, die im Katastrophenfall verloren ginge. Außerdem bedeutet dies, dass das Verzeichnis `pg_wal/` viele noch nicht archivierte Segmentdateien enthält, was irgendwann den verfügbaren Plattenplatz überschreiten kann. Es wird empfohlen, den Archivierungsprozess zu überwachen, um sicherzustellen, dass er wie vorgesehen funktioniert.

Beim Schreiben Ihres Archivierungsbefehls oder Ihrer Archivbibliothek sollten Sie annehmen, dass die zu archivierenden Dateinamen bis zu 64 Zeichen lang sein und jede Kombination aus ASCII-Buchstaben, Ziffern und Punkten enthalten können. Es ist nicht notwendig, den ursprünglichen relativen Pfad (`%p`) zu erhalten, wohl aber den Dateinamen (`%f`).

Beachten Sie, dass WAL-Archivierung zwar die Wiederherstellung aller Änderungen erlaubt, die an den Daten in Ihrer PostgreSQL-Datenbank vorgenommen wurden, aber keine Änderungen an Konfigurationsdateien wiederherstellt, also an `postgresql.conf`, `pg_hba.conf` und `pg_ident.conf`, da diese manuell und nicht über SQL-Operationen bearbeitet werden. Sie sollten die Konfigurationsdateien eventuell an einem Ort aufbewahren, der durch Ihre regulären Dateisicherungsverfahren gesichert wird. [Abschnitt 19.2](19_Serverkonfiguration.md#192-dateispeicherorte) beschreibt, wie Konfigurationsdateien verlegt werden können.

Der Archivierungsbefehl oder die Archivierungsfunktion wird nur für abgeschlossene WAL-Segmente aufgerufen. Wenn Ihr Server daher nur wenig WAL-Verkehr erzeugt oder ruhige Phasen hat, kann es eine lange Verzögerung zwischen dem Abschluss einer Transaktion und ihrer sicheren Aufzeichnung im Archivspeicher geben. Um zu begrenzen, wie alt nicht archivierte Daten werden können, können Sie `archive_timeout` setzen, damit der Server mindestens in diesem Intervall auf eine neue WAL-Segmentdatei umschaltet. Beachten Sie, dass Dateien, die wegen eines erzwungenen Wechsels früh archiviert werden, trotzdem genauso lang sind wie vollständig gefüllte Dateien. Daher ist es unklug, `archive_timeout` sehr kurz zu setzen; das bläht Ihren Archivspeicher auf. Einstellungen von ungefähr einer Minute sind normalerweise vernünftig.

Außerdem können Sie mit `pg_switch_wal` manuell einen Segmentwechsel erzwingen, wenn Sie sicherstellen möchten, dass eine gerade abgeschlossene Transaktion so schnell wie möglich archiviert wird. Weitere Hilfsfunktionen zur WAL-Verwaltung sind in Tabelle 9.97 aufgeführt.

Wenn `wal_level` auf `minimal` steht, werden einige SQL-Befehle so optimiert, dass WAL-Protokollierung vermieden wird, wie in Abschnitt 14.4.7 beschrieben. Würden Archivierung oder Streaming-Replikation während der Ausführung einer dieser Anweisungen eingeschaltet, enthielte WAL nicht genügend Informationen für eine Archive-Recovery. Crash-Recovery ist davon nicht betroffen. Aus diesem Grund kann `wal_level` nur beim Serverstart geändert werden. `archive_command` und `archive_library` können jedoch durch ein Neuladen der Konfigurationsdatei geändert werden. Wenn Sie über die Shell archivieren und die Archivierung vorübergehend stoppen möchten, besteht eine Möglichkeit darin, `archive_command` auf die leere Zeichenkette (`''`) zu setzen. Dadurch sammeln sich WAL-Dateien in `pg_wal/`, bis wieder ein funktionierender `archive_command` eingerichtet ist.

### 25.3.2. Eine Basissicherung erstellen

Der einfachste Weg, eine Basissicherung zu erstellen, ist das Werkzeug `pg_basebackup`. Es kann eine Basissicherung entweder als normale Dateien oder als `tar`-Archiv erzeugen. Wenn mehr Flexibilität benötigt wird, als `pg_basebackup` bietet, können Sie eine Basissicherung auch über die Low-Level-API erstellen; siehe [Abschnitt 25.3.4](#2534-eine-basissicherung-mit-der-lowlevelapi-erstellen).

Es ist nicht nötig, sich über die Dauer einer Basissicherung Gedanken zu machen. Wenn Sie den Server normalerweise mit deaktiviertem `full_page_writes` betreiben, können Sie jedoch während der Sicherung einen Performance-Rückgang feststellen, weil `full_page_writes` im Sicherungsmodus effektiv erzwungen wird.

Um die Sicherung nutzen zu können, müssen Sie alle WAL-Segmentdateien aufbewahren, die während und nach der Dateisystemsicherung erzeugt werden. Um Sie dabei zu unterstützen, erzeugt der Basissicherungsprozess eine Backup-History-Datei, die sofort im WAL-Archivbereich gespeichert wird. Diese Datei wird nach der ersten WAL-Segmentdatei benannt, die Sie für die Dateisystemsicherung benötigen. Wenn die Start-WAL-Datei beispielsweise `0000000100001234000055CD` ist, heißt die Backup-History-Datei etwa `0000000100001234000055CD.007C9330.backup`. Der zweite Teil des Dateinamens steht für eine genaue Position innerhalb der WAL-Datei und kann normalerweise ignoriert werden. Sobald Sie die Dateisystemsicherung und die während der Sicherung verwendeten WAL-Segmentdateien sicher archiviert haben, wie in der Backup-History-Datei angegeben, werden alle archivierten WAL-Segmente mit numerisch kleineren Namen für die Wiederherstellung dieser Dateisystemsicherung nicht mehr benötigt und können gelöscht werden. Sie sollten jedoch erwägen, mehrere Sicherungssätze aufzubewahren, um absolut sicher zu sein, dass Sie Ihre Daten wiederherstellen können.

Die Backup-History-Datei ist nur eine kleine Textdatei. Sie enthält die Label-Zeichenkette, die Sie `pg_basebackup` gegeben haben, sowie Start- und Endzeiten und WAL-Segmente der Sicherung. Wenn Sie das Label verwendet haben, um die zugehörige Dump-Datei zu identifizieren, reicht die archivierte History-Datei aus, um zu erkennen, welche Dump-Datei wiederhergestellt werden muss.

Da Sie alle archivierten WAL-Dateien bis zurück zur letzten Basissicherung aufbewahren müssen, sollte das Intervall zwischen Basissicherungen normalerweise danach gewählt werden, wie viel Speicher Sie für archivierte WAL-Dateien einsetzen möchten. Außerdem sollten Sie berücksichtigen, wie viel Zeit Sie im Wiederherstellungsfall für Recovery aufwenden möchten: Das System muss alle diese WAL-Segmente erneut abspielen, und das kann dauern, wenn seit der letzten Basissicherung viel Zeit vergangen ist.

### 25.3.3. Eine inkrementelle Sicherung erstellen

Sie können `pg_basebackup` für eine inkrementelle Sicherung verwenden, indem Sie die Option `--incremental` angeben. Als Argument für `--incremental` müssen Sie das Backup-Manifest einer früheren Sicherung desselben Servers angeben. In der resultierenden Sicherung werden Nicht-Relation-Dateien vollständig aufgenommen, aber einige Relation-Dateien können durch kleinere inkrementelle Dateien ersetzt werden, die nur die seit der früheren Sicherung geänderten Blöcke sowie genügend Metadaten enthalten, um die aktuelle Version der Datei zu rekonstruieren.

Um zu ermitteln, welche Blöcke gesichert werden müssen, verwendet der Server WAL-Zusammenfassungen, die im Datenverzeichnis im Verzeichnis `pg_wal/summaries` gespeichert werden. Wenn die erforderlichen Summary-Dateien nicht vorhanden sind, schlägt der Versuch einer inkrementellen Sicherung fehl. Die in diesem Verzeichnis vorhandenen Zusammenfassungen müssen alle LSNs von der Start-LSN der vorherigen Sicherung bis zur Start-LSN der aktuellen Sicherung abdecken. Da der Server kurz nach dem Festlegen der Start-LSN der aktuellen Sicherung nach WAL-Zusammenfassungen sucht, werden die nötigen Summary-Dateien wahrscheinlich nicht sofort auf der Platte vorhanden sein; der Server wartet aber darauf, dass fehlende Dateien erscheinen. Das hilft auch, wenn der WAL-Summarization-Prozess zurückgefallen ist. Wenn die nötigen Dateien jedoch bereits entfernt wurden oder der WAL-Summarizer nicht schnell genug aufholt, schlägt die inkrementelle Sicherung fehl.

Beim Wiederherstellen einer inkrementellen Sicherung benötigen Sie nicht nur die inkrementelle Sicherung selbst, sondern auch alle früheren Sicherungen, die erforderlich sind, um die in der inkrementellen Sicherung ausgelassenen Blöcke bereitzustellen. Weitere Informationen zu dieser Anforderung finden Sie bei `pg_combinebackup`. Beachten Sie, dass es Einschränkungen für die Verwendung von `pg_combinebackup` gibt, wenn sich der Prüfsummenstatus des Clusters geändert hat; siehe die Einschränkungen von `pg_combinebackup`.

Alle Anforderungen für die Nutzung einer vollständigen Sicherung gelten auch für eine inkrementelle Sicherung. Sie benötigen zum Beispiel weiterhin alle WAL-Segmentdateien, die während und nach der Dateisystemsicherung erzeugt wurden, sowie alle relevanten WAL-History-Dateien. Und Sie müssen weiterhin eine Datei `recovery.signal` oder `standby.signal` erstellen und Recovery durchführen, wie in [Abschnitt 25.3.5](#2535-wiederherstellung-mit-einer-sicherung-aus-kontinuierlichem-archiv) beschrieben. Die Anforderung, frühere Sicherungen zur Wiederherstellungszeit verfügbar zu haben und `pg_combinebackup` zu verwenden, kommt zusätzlich zu allem anderen hinzu. Denken Sie daran, dass PostgreSQL keinen eingebauten Mechanismus besitzt, um herauszufinden, welche Sicherungen noch als Grundlage für die Wiederherstellung späterer inkrementeller Sicherungen benötigt werden. Sie müssen die Beziehungen zwischen Ihren vollständigen und inkrementellen Sicherungen selbst nachhalten und sicherstellen, dass frühere Sicherungen nicht entfernt werden, wenn sie zur Wiederherstellung späterer inkrementeller Sicherungen benötigt werden könnten.

Inkrementelle Sicherungen sind typischerweise nur für relativ große Datenbanken sinnvoll, bei denen ein erheblicher Teil der Daten unverändert bleibt oder sich nur langsam ändert. Bei einer kleinen Datenbank ist es einfacher, inkrementelle Sicherungen zu ignorieren und vollständige Sicherungen zu erstellen, die leichter zu verwalten sind. Bei einer großen Datenbank, in der alles stark geändert wird, sind inkrementelle Sicherungen kaum kleiner als vollständige Sicherungen.

Eine inkrementelle Sicherung ist nur möglich, wenn das Replay von einem späteren Checkpoint beginnen würde als bei der vorherigen Sicherung, von der sie abhängt. Wenn Sie die inkrementelle Sicherung auf dem Primary erstellen, ist diese Bedingung immer erfüllt, weil jede Sicherung einen neuen Checkpoint auslöst. Auf einem Standby beginnt das Replay am neuesten Restartpoint. Deshalb kann eine inkrementelle Sicherung eines Standby-Servers fehlschlagen, wenn es seit der vorherigen Sicherung nur sehr wenig Aktivität gab und daher kein neuer Restartpoint erstellt wurde.

### 25.3.4. Eine Basissicherung mit der Low-Level-API erstellen

Anstatt eine vollständige oder inkrementelle Basissicherung mit `pg_basebackup` zu erstellen, können Sie eine Basissicherung über die Low-Level-API durchführen. Dieses Verfahren umfasst etwas mehr Schritte als die Methode mit `pg_basebackup`, ist aber relativ einfach. Es ist sehr wichtig, dass diese Schritte in der angegebenen Reihenfolge ausgeführt werden und dass der Erfolg eines Schritts geprüft wird, bevor der nächste begonnen wird.

Mehrere Sicherungen können gleichzeitig laufen, sowohl solche, die über diese Backup-API gestartet wurden, als auch solche, die mit `pg_basebackup` gestartet wurden.

1. Stellen Sie sicher, dass WAL-Archivierung aktiviert ist und funktioniert.

2. Verbinden Sie sich als Benutzer mit dem Server, egal mit welcher Datenbank, der berechtigt ist, `pg_backup_start` auszuführen, also als Superuser oder als Benutzer, dem `EXECUTE` auf die Funktion gewährt wurde, und führen Sie den Befehl aus:

   ```sql
   SELECT pg_backup_start(label => 'label', fast => false);
   ```

   Dabei ist `label` eine beliebige Zeichenkette, mit der Sie diesen Sicherungsvorgang eindeutig identifizieren möchten. Die Verbindung, die `pg_backup_start` aufruft, muss bis zum Ende der Sicherung bestehen bleiben, sonst wird die Sicherung automatisch abgebrochen.

   Online-Sicherungen beginnen immer am Anfang eines Checkpoints. Standardmäßig wartet `pg_backup_start`, bis der nächste regulär geplante Checkpoint abgeschlossen ist, was lange dauern kann; siehe die Konfigurationsparameter `checkpoint_timeout` und `checkpoint_completion_target`. Das ist normalerweise vorzuziehen, weil es die Auswirkungen auf das laufende System minimiert. Wenn Sie die Sicherung so schnell wie möglich starten möchten, übergeben Sie `true` als zweiten Parameter an `pg_backup_start`; dann wird ein sofortiger Checkpoint angefordert, der so schnell wie möglich und mit so viel I/O wie möglich abgeschlossen wird.

3. Führen Sie die Sicherung mit einem geeigneten Dateisicherungswerkzeug wie `tar` oder `cpio` durch, nicht mit `pg_dump` oder `pg_dumpall`. Es ist weder notwendig noch wünschenswert, den normalen Datenbankbetrieb dafür anzuhalten. [Abschnitt 25.3.4.1](#25341-das-datenverzeichnis-sichern) enthält Punkte, die während dieser Sicherung zu beachten sind.

4. Führen Sie in derselben Verbindung wie zuvor den Befehl aus:

   ```sql
   SELECT * FROM pg_backup_stop(wait_for_archive => true);
   ```

   Dies beendet den Sicherungsmodus. Auf einem Primary bewirkt es außerdem einen automatischen Wechsel zum nächsten WAL-Segment. Auf einem Standby ist ein automatischer WAL-Segmentwechsel nicht möglich; daher möchten Sie möglicherweise auf dem Primary `pg_switch_wal` ausführen, um manuell zu wechseln. Der Grund für den Wechsel ist, dass die letzte während des Sicherungsintervalls geschriebene WAL-Segmentdatei für die Archivierung bereitgestellt werden soll.

   `pg_backup_stop` gibt eine Zeile mit drei Werten zurück. Das zweite dieser Felder sollte in eine Datei namens `backup_label` im Wurzelverzeichnis der Sicherung geschrieben werden. Das dritte Feld sollte in eine Datei namens `tablespace_map` geschrieben werden, sofern das Feld nicht leer ist. Diese Dateien sind entscheidend dafür, dass die Sicherung funktioniert, und müssen bytegenau ohne Änderung geschrieben werden, was möglicherweise das Öffnen der Datei im Binärmodus erfordert.

5. Sobald die während der Sicherung aktiven WAL-Segmentdateien archiviert sind, sind Sie fertig. Die Datei, die durch den ersten Rückgabewert von `pg_backup_stop` identifiziert wird, ist das letzte Segment, das für einen vollständigen Satz von Sicherungsdateien erforderlich ist. Auf einem Primary kehrt `pg_backup_stop` bei aktiviertem `archive_mode` und gesetztem Parameter `wait_for_archive` erst zurück, wenn das letzte Segment archiviert wurde. Auf einem Standby muss `archive_mode` auf `always` stehen, damit `pg_backup_stop` wartet. Die Archivierung dieser Dateien erfolgt automatisch, da Sie `archive_command` oder `archive_library` bereits konfiguriert haben. In den meisten Fällen geschieht dies schnell, aber Sie sollten Ihr Archivsystem überwachen, um sicherzustellen, dass keine Verzögerungen auftreten. Wenn der Archivierungsprozess wegen Fehlern des Archivierungsbefehls oder der Archivbibliothek zurückgefallen ist, versucht er es weiter, bis die Archivierung gelingt und die Sicherung vollständig ist. Wenn Sie die Ausführungszeit von `pg_backup_stop` begrenzen möchten, setzen Sie einen passenden Wert für `statement_timeout`; beachten Sie aber, dass Ihre Sicherung möglicherweise nicht gültig ist, wenn `pg_backup_stop` dadurch beendet wird.

   Wenn der Sicherungsprozess selbst überwacht und sicherstellt, dass alle für die Sicherung erforderlichen WAL-Segmentdateien erfolgreich archiviert wurden, kann der Parameter `wait_for_archive`, der standardmäßig `true` ist, auf `false` gesetzt werden, damit `pg_backup_stop` zurückkehrt, sobald der Stop-Backup-Datensatz ins WAL geschrieben wurde. Standardmäßig wartet `pg_backup_stop`, bis alles WAL archiviert wurde, was einige Zeit dauern kann. Diese Option muss mit Vorsicht verwendet werden: Wenn die WAL-Archivierung nicht korrekt überwacht wird, enthält die Sicherung möglicherweise nicht alle WAL-Dateien und ist daher unvollständig und nicht wiederherstellbar.

#### 25.3.4.1. Das Datenverzeichnis sichern

Einige Dateisicherungswerkzeuge geben Warnungen oder Fehler aus, wenn sich Dateien während des Kopierens ändern. Beim Erstellen einer Basissicherung einer aktiven Datenbank ist diese Situation normal und kein Fehler. Sie müssen jedoch sicherstellen, dass Sie Beschwerden dieser Art von echten Fehlern unterscheiden können. Einige Versionen von `rsync` geben beispielsweise einen eigenen Exit-Code für verschwundene Quelldateien zurück, und Sie können ein Steuerungsskript schreiben, das diesen Exit-Code als Nicht-Fehler akzeptiert. Einige Versionen von GNU `tar` geben außerdem einen Fehlercode zurück, der nicht von einem fatalen Fehler zu unterscheiden ist, wenn eine Datei während des Kopierens durch `tar` gekürzt wurde. Glücklicherweise beendet sich GNU `tar` ab Version 1.16 mit 1, wenn eine Datei während der Sicherung geändert wurde, und mit 2 bei anderen Fehlern. Ab GNU `tar` 1.23 können Sie die Warnoptionen `--warning=no-file-changed --warning=no-file-removed` verwenden, um die entsprechenden Warnmeldungen zu unterdrücken.

Stellen Sie sicher, dass Ihre Sicherung alle Dateien unter dem Datenbankclusterverzeichnis enthält, zum Beispiel `/usr/local/pgsql/data`. Wenn Sie Tablespaces verwenden, die nicht unterhalb dieses Verzeichnisses liegen, achten Sie darauf, auch diese einzuschließen, und stellen Sie sicher, dass Ihre Sicherung symbolische Links als Links archiviert; andernfalls beschädigt die Wiederherstellung Ihre Tablespaces.

Sie sollten jedoch die Dateien im Unterverzeichnis `pg_wal/` des Clusters aus der Sicherung auslassen. Diese kleine Anpassung ist sinnvoll, weil sie das Risiko von Fehlern bei der Wiederherstellung verringert. Das ist leicht einzurichten, wenn `pg_wal/` ein symbolischer Link auf einen Ort außerhalb des Clusterverzeichnisses ist, was aus Performance-Gründen ohnehin eine verbreitete Konfiguration ist. Möglicherweise möchten Sie auch `postmaster.pid` und `postmaster.opts` ausschließen, die Informationen über den laufenden Postmaster aufzeichnen, nicht über den Postmaster, der diese Sicherung später verwenden wird. Diese Dateien können `pg_ctl` verwirren.

Oft ist es auch sinnvoll, die Dateien im Verzeichnis `pg_replslot/` des Clusters aus der Sicherung auszulassen, damit Replikations-Slots, die auf dem Primary existieren, nicht Teil der Sicherung werden. Andernfalls kann die spätere Verwendung der Sicherung zur Erstellung eines Standby dazu führen, dass WAL-Dateien auf dem Standby unbegrenzt zurückgehalten werden, und möglicherweise zu Aufblähung auf dem Primary, wenn Hot-Standby-Feedback aktiviert ist, weil die Clients, die diese Replikations-Slots verwenden, weiterhin eine Verbindung zum Primary herstellen und die Slots dort aktualisieren, nicht auf dem Standby. Selbst wenn die Sicherung nur zur Erstellung eines neuen Primary gedacht ist, wird das Kopieren der Replikations-Slots voraussichtlich nicht besonders nützlich sein, da die Inhalte dieser Slots wahrscheinlich stark veraltet sind, wenn der neue Primary online geht.

Die Inhalte der Verzeichnisse `pg_dynshmem/`, `pg_notify/`, `pg_serial/`, `pg_snapshots/`, `pg_stat_tmp/` und `pg_subtrans/`, nicht aber die Verzeichnisse selbst, können aus der Sicherung ausgelassen werden, da sie beim Start des Postmasters initialisiert werden.

Jede Datei oder jedes Verzeichnis, dessen Name mit `pgsql_tmp` beginnt, kann aus der Sicherung ausgelassen werden. Diese Dateien werden beim Start des Postmasters entfernt, und die Verzeichnisse werden bei Bedarf neu erstellt.

Dateien namens `pg_internal.init` können überall dort ausgelassen werden, wo eine Datei dieses Namens gefunden wird. Diese Dateien enthalten Relation-Cache-Daten, die bei der Wiederherstellung immer neu aufgebaut werden.

Die Backup-Label-Datei enthält die Label-Zeichenkette, die Sie `pg_backup_start` gegeben haben, sowie die Zeit, zu der `pg_backup_start` ausgeführt wurde, und den Namen der Start-WAL-Datei. Im Zweifelsfall ist es daher möglich, in eine Sicherungsdatei hineinzusehen und genau zu bestimmen, aus welcher Sicherungssitzung die Dump-Datei stammt. Die Tablespace-Map-Datei enthält die Namen der symbolischen Links, wie sie im Verzeichnis `pg_tblspc/` existieren, und den vollständigen Pfad jedes symbolischen Links. Diese Dateien dienen nicht nur Ihrer Information; ihre Anwesenheit und ihr Inhalt sind entscheidend für die korrekte Funktion des Recovery-Prozesses des Systems.

Es ist auch möglich, eine Sicherung zu erstellen, während der Server gestoppt ist. In diesem Fall können Sie natürlich weder `pg_backup_start` noch `pg_backup_stop` verwenden und müssen daher selbst nachhalten, welche Sicherung welche ist und wie weit die zugehörigen WAL-Dateien zurückreichen. Im Allgemeinen ist es besser, dem oben beschriebenen Verfahren der kontinuierlichen Archivierung zu folgen.

### 25.3.5. Wiederherstellung mit einer Sicherung aus kontinuierlichem Archiv

Gut, das Schlimmste ist passiert, und Sie müssen aus Ihrer Sicherung wiederherstellen. Das Verfahren ist:

1. Stoppen Sie den Server, falls er läuft.

2. Wenn Sie genug Platz haben, kopieren Sie das gesamte Cluster-Datenverzeichnis und alle Tablespaces an einen temporären Ort, falls Sie sie später benötigen. Beachten Sie, dass diese Vorsichtsmaßnahme voraussetzt, dass auf Ihrem System genug freier Platz für zwei Kopien Ihrer vorhandenen Datenbank vorhanden ist. Wenn Sie nicht genug Platz haben, sollten Sie zumindest den Inhalt des Unterverzeichnisses `pg_wal` des Clusters sichern, da es WAL-Dateien enthalten könnte, die vor dem Systemausfall nicht archiviert wurden.

3. Entfernen Sie alle vorhandenen Dateien und Unterverzeichnisse unter dem Cluster-Datenverzeichnis und unter den Wurzelverzeichnissen aller verwendeten Tablespaces.

4. Wenn Sie eine vollständige Sicherung wiederherstellen, können Sie die Datenbankdateien direkt in die Zielverzeichnisse zurückspielen. Achten Sie darauf, dass sie mit dem richtigen Eigentümer, also dem Datenbanksystembenutzer und nicht `root`, und mit den richtigen Berechtigungen wiederhergestellt werden. Wenn Sie Tablespaces verwenden, sollten Sie prüfen, ob die symbolischen Links in `pg_tblspc/` korrekt wiederhergestellt wurden.

5. Wenn Sie eine inkrementelle Sicherung wiederherstellen, müssen Sie die inkrementelle Sicherung und alle früheren Sicherungen, von denen sie direkt oder indirekt abhängt, auf die Maschine zurückspielen, auf der Sie die Wiederherstellung durchführen. Diese Sicherungen müssen in getrennten Verzeichnissen liegen, nicht in den Zielverzeichnissen, in denen der laufende Server am Ende liegen soll. Danach verwenden Sie `pg_combinebackup`, um Daten aus der vollständigen Sicherung und allen nachfolgenden inkrementellen Sicherungen zusammenzuführen und eine synthetische vollständige Sicherung in die Zielverzeichnisse zu schreiben. Prüfen Sie wie oben, ob Berechtigungen und Tablespace-Links korrekt sind.

6. Entfernen Sie alle in `pg_wal/` vorhandenen Dateien; sie stammen aus der Dateisystemsicherung und sind daher wahrscheinlich veraltet statt aktuell. Wenn Sie `pg_wal/` überhaupt nicht archiviert haben, erstellen Sie es mit den richtigen Berechtigungen neu und achten dabei darauf, es wieder als symbolischen Link einzurichten, falls es zuvor so konfiguriert war.

7. Wenn Sie nicht archivierte WAL-Segmentdateien haben, die Sie in Schritt 2 gesichert haben, kopieren Sie sie nach `pg_wal/`. Es ist am besten, sie zu kopieren und nicht zu verschieben, damit Sie die unveränderten Dateien noch haben, falls ein Problem auftritt und Sie von vorn beginnen müssen.

8. Setzen Sie die Recovery-Konfigurationseinstellungen in `postgresql.conf` (siehe [Abschnitt 19.5.5](19_Serverkonfiguration.md#1955-archive-recovery)) und erstellen Sie eine Datei `recovery.signal` im Cluster-Datenverzeichnis. Möglicherweise möchten Sie auch `pg_hba.conf` vorübergehend ändern, um gewöhnliche Benutzer daran zu hindern, sich zu verbinden, bis Sie sicher sind, dass die Recovery erfolgreich war.

9. Starten Sie den Server. Der Server wechselt in den Recovery-Modus und liest die benötigten archivierten WAL-Dateien. Sollte die Recovery wegen eines externen Fehlers beendet werden, kann der Server einfach neu gestartet werden und setzt die Recovery fort. Nach Abschluss des Recovery-Prozesses entfernt der Server `recovery.signal`, um ein versehentliches späteres erneutes Eintreten in den Recovery-Modus zu verhindern, und beginnt anschließend mit dem normalen Datenbankbetrieb.

10. Prüfen Sie den Inhalt der Datenbank, um sicherzustellen, dass Sie den gewünschten Zustand wiederhergestellt haben. Falls nicht, kehren Sie zu Schritt 1 zurück. Wenn alles in Ordnung ist, erlauben Sie Ihren Benutzern wieder Verbindungen, indem Sie `pg_hba.conf` in den Normalzustand zurückversetzen.

Der entscheidende Teil dieses Verfahrens ist die Einrichtung einer Recovery-Konfiguration, die beschreibt, wie Sie wiederherstellen möchten und wie weit die Recovery laufen soll. Das eine, was Sie unbedingt angeben müssen, ist `restore_command`, der PostgreSQL mitteilt, wie archivierte WAL-Dateisegmente abgerufen werden. Wie `archive_command` ist dies eine Shell-Befehlszeichenkette. Sie kann `%f` enthalten, das durch den Namen der gewünschten WAL-Datei ersetzt wird, und `%p`, das durch den Pfadnamen ersetzt wird, an den die WAL-Datei kopiert werden soll. Der Pfadname ist relativ zum aktuellen Arbeitsverzeichnis, also zum Datenverzeichnis des Clusters. Schreiben Sie `%%`, wenn Sie ein echtes Prozentzeichen in den Befehl einbetten müssen. Der einfachste nützliche Befehl sieht etwa so aus:

```text
restore_command = 'cp /mnt/server/archivedir/%f %p'
```

Damit werden zuvor archivierte WAL-Segmente aus dem Verzeichnis `/mnt/server/archivedir` kopiert. Natürlich können Sie etwas deutlich Komplexeres verwenden, vielleicht sogar ein Shellskript, das den Operator auffordert, ein passendes Band einzuhängen.

Es ist wichtig, dass der Befehl bei einem Fehlschlag einen von null verschiedenen Exit-Status zurückgibt. Der Befehl wird auch für Dateien aufgerufen, die im Archiv nicht vorhanden sind; in diesem Fall muss er einen von null verschiedenen Status zurückgeben. Das ist kein Fehlerzustand. Eine Ausnahme ist, wenn der Befehl durch ein Signal beendet wurde, außer `SIGTERM`, das als Teil eines Datenbankserver-Shutdowns verwendet wird, oder durch einen Shell-Fehler wie einen nicht gefundenen Befehl; dann bricht die Recovery ab und der Server startet nicht.

Nicht alle angeforderten Dateien sind WAL-Segmentdateien; rechnen Sie auch mit Anforderungen nach Dateien mit dem Suffix `.history`. Beachten Sie außerdem, dass der Basisname des `%p`-Pfades von `%f` verschieden ist; erwarten Sie nicht, dass beide austauschbar sind.

WAL-Segmente, die im Archiv nicht gefunden werden, werden in `pg_wal/` gesucht; dadurch können aktuelle, noch nicht archivierte Segmente verwendet werden. Segmente, die im Archiv verfügbar sind, werden jedoch Dateien in `pg_wal/` vorgezogen.

Normalerweise läuft die Recovery durch alle verfügbaren WAL-Segmente und stellt die Datenbank dadurch bis zum aktuellen Zeitpunkt wieder her, oder so nah daran wie mit den verfügbaren WAL-Segmenten möglich. Eine normale Recovery endet daher mit einer Meldung „file not found“, wobei der genaue Text der Fehlermeldung von Ihrer Wahl für `restore_command` abhängt. Sie können zu Beginn der Recovery auch eine Fehlermeldung für eine Datei mit einem Namen wie `00000001.history` sehen. Auch das ist normal und weist in einfachen Recovery-Situationen nicht auf ein Problem hin; siehe [Abschnitt 25.3.6](#2536-zeitleisten).

Wenn Sie zu einem früheren Zeitpunkt wiederherstellen möchten, etwa direkt bevor der Junior-DBA Ihre wichtigste Transaktionstabelle gelöscht hat, geben Sie einfach den erforderlichen Haltepunkt an. Sie können den Haltepunkt, das sogenannte Recovery Target, entweder durch Datum/Uhrzeit, einen benannten Restore Point oder durch den Abschluss einer bestimmten Transaktions-ID angeben. Zum Zeitpunkt dieser Beschreibung sind nur die Optionen Datum/Uhrzeit und benannter Restore Point wirklich gut nutzbar, da es keine Werkzeuge gibt, mit denen sich zuverlässig bestimmen lässt, welche Transaktions-ID verwendet werden soll.

> **Hinweis**
>
> Der Haltepunkt muss nach der Endzeit der Basissicherung liegen, also nach der Endzeit von `pg_backup_stop`. Sie können eine Basissicherung nicht verwenden, um zu einem Zeitpunkt wiederherzustellen, zu dem diese Sicherung noch lief. Um zu einem solchen Zeitpunkt zurückzukehren, müssen Sie auf die vorherige Basissicherung zurückgehen und von dort vorwärts abspielen.

Wenn Recovery beschädigte WAL-Daten findet, hält sie an dieser Stelle an und der Server startet nicht. In einem solchen Fall kann der Recovery-Prozess von Anfang an erneut ausgeführt werden, wobei ein Recovery Target vor dem Punkt der Beschädigung angegeben wird, damit die Recovery normal abgeschlossen werden kann. Wenn Recovery aus einem externen Grund fehlschlägt, etwa wegen eines Systemabsturzes oder weil das WAL-Archiv nicht zugänglich ist, kann sie einfach neu gestartet werden und beginnt nahezu an der Stelle, an der sie fehlgeschlagen ist. Der Neustart der Recovery funktioniert ähnlich wie Checkpointing im normalen Betrieb: Der Server schreibt regelmäßig seinen gesamten Zustand auf Platte und aktualisiert dann die Datei `pg_control`, um anzugeben, dass bereits verarbeitete WAL-Daten nicht erneut durchsucht werden müssen.

### 25.3.6. Zeitleisten

Die Fähigkeit, die Datenbank zu einem früheren Zeitpunkt wiederherzustellen, erzeugt einige Komplexitäten, die an Science-Fiction-Geschichten über Zeitreisen und Paralleluniversen erinnern. Nehmen wir zum Beispiel an, Sie hätten in der ursprünglichen Geschichte der Datenbank am Dienstagabend um 17:15 Uhr eine kritische Tabelle gelöscht, Ihren Fehler aber erst am Mittwochmittag bemerkt. Unerschrocken holen Sie Ihre Sicherung hervor, stellen bis Dienstagabend 17:14 Uhr wieder her und nehmen den Betrieb wieder auf. In dieser Geschichte des Datenbankuniversums haben Sie die Tabelle nie gelöscht. Angenommen aber, Sie stellen später fest, dass das doch keine so gute Idee war, und möchten zu einem Zeitpunkt am Mittwochmorgen in der ursprünglichen Geschichte zurückkehren. Das wird nicht möglich sein, wenn Ihre wieder laufende Datenbank einige WAL-Segmentdateien überschrieben hat, die zu dem Zeitpunkt führten, zu dem Sie nun gern zurückkehren würden. Um das zu vermeiden, müssen Sie die Folge von WAL-Datensätzen, die nach einer Point-in-Time-Recovery erzeugt wurde, von der Folge unterscheiden, die in der ursprünglichen Datenbankgeschichte erzeugt wurde.

Um dieses Problem zu lösen, kennt PostgreSQL das Konzept der Zeitleisten. Immer wenn eine Archive-Recovery abgeschlossen wird, wird eine neue Zeitleiste erzeugt, um die Folge von WAL-Datensätzen zu kennzeichnen, die nach dieser Recovery erzeugt wird. Die Timeline-ID ist Teil der Namen von WAL-Segmentdateien, sodass eine neue Zeitleiste die von früheren Zeitleisten erzeugten WAL-Daten nicht überschreibt. Im WAL-Dateinamen `0000000100001234000055CD` ist zum Beispiel das führende `00000001` die Timeline-ID in hexadezimaler Form. In anderen Kontexten, etwa in Server-Logmeldungen, werden Timeline-IDs üblicherweise dezimal ausgegeben.

Tatsächlich ist es möglich, viele verschiedene Zeitleisten zu archivieren. Das mag wie ein nutzloses Feature erscheinen, ist aber oft lebensrettend. Stellen Sie sich vor, Sie sind nicht ganz sicher, zu welchem Zeitpunkt Sie wiederherstellen sollen, und müssen daher mehrere Point-in-Time-Recoverys durch Ausprobieren durchführen, bis Sie die beste Abzweigung von der alten Geschichte finden. Ohne Zeitleisten würde dieser Prozess schnell ein unüberschaubares Durcheinander erzeugen. Mit Zeitleisten können Sie zu jedem früheren Zustand zurückkehren, einschließlich Zuständen in Zeitleistenästen, die Sie zuvor aufgegeben haben.

Jedes Mal, wenn eine neue Zeitleiste erzeugt wird, erstellt PostgreSQL eine Timeline-History-Datei, die zeigt, von welcher Zeitleiste sie wann abgezweigt ist. Diese History-Dateien sind nötig, damit das System bei der Wiederherstellung aus einem Archiv mit mehreren Zeitleisten die richtigen WAL-Segmentdateien auswählen kann. Daher werden sie wie WAL-Segmentdateien in den WAL-Archivbereich archiviert. Die History-Dateien sind nur kleine Textdateien, daher ist es billig und sinnvoll, sie unbegrenzt aufzubewahren, anders als die großen Segmentdateien. Wenn Sie möchten, können Sie einer History-Datei Kommentare hinzufügen, um eigene Notizen darüber festzuhalten, wie und warum diese bestimmte Zeitleiste erzeugt wurde. Solche Kommentare sind besonders wertvoll, wenn infolge von Experimenten ein Dickicht verschiedener Zeitleisten entstanden ist.

Das Standardverhalten der Recovery ist, zur neuesten im Archiv gefundenen Zeitleiste wiederherzustellen. Wenn Sie zu der Zeitleiste zurückkehren möchten, die zum Zeitpunkt der Basissicherung aktuell war, oder in eine bestimmte untergeordnete Zeitleiste, also in einen Zustand, der selbst nach einem Recovery-Versuch entstanden ist, müssen Sie `current` oder die Ziel-Timeline-ID in `recovery_target_timeline` angeben. Sie können nicht in Zeitleisten wiederherstellen, die früher abgezweigt sind als die Basissicherung.

### 25.3.7. Tipps und Beispiele

Hier folgen einige Tipps zur Konfiguration kontinuierlicher Archivierung.

#### 25.3.7.1. Eigenständige Hot Backups

Es ist möglich, die Sicherungsfunktionen von PostgreSQL zu verwenden, um eigenständige Hot Backups zu erzeugen. Das sind Sicherungen, die nicht für Point-in-Time-Recovery verwendet werden können, aber typischerweise viel schneller zu sichern und wiederherzustellen sind als `pg_dump`-Dumps. Sie sind allerdings auch viel größer als `pg_dump`-Dumps, sodass der Geschwindigkeitsvorteil in manchen Fällen aufgehoben werden kann.

Wie bei Basissicherungen ist der einfachste Weg zu einem eigenständigen Hot Backup das Werkzeug `pg_basebackup`. Wenn Sie beim Aufruf den Parameter `-X` angeben, wird das gesamte für die Nutzung der Sicherung erforderliche Write-Ahead-Log automatisch in die Sicherung aufgenommen, und für die Wiederherstellung sind keine besonderen Aktionen erforderlich.

#### 25.3.7.2. Komprimierte Archiv-Logs

Wenn die Größe des Archivspeichers ein Problem ist, können Sie `gzip` verwenden, um die Archivdateien zu komprimieren:

```text
archive_command = 'gzip < %p > /mnt/server/archivedir/%f.gz'
```

Während der Recovery müssen Sie dann `gunzip` verwenden:

```text
restore_command = 'gunzip < /mnt/server/archivedir/%f.gz > %p'
```

#### 25.3.7.3. `archive_command`-Skripte

Viele Menschen verwenden Skripte, um ihren `archive_command` zu definieren, sodass ihr Eintrag in `postgresql.conf` sehr einfach aussieht:

```text
archive_command = 'local_backup_script.sh "%p" "%f"'
```

Eine separate Skriptdatei empfiehlt sich immer dann, wenn Sie im Archivierungsprozess mehr als einen einzigen Befehl verwenden möchten. Dadurch lässt sich die gesamte Komplexität im Skript verwalten, das in einer verbreiteten Skriptsprache wie `bash` oder `perl` geschrieben werden kann.

Beispiele für Anforderungen, die in einem Skript gelöst werden könnten:

- Kopieren von Daten in einen sicheren externen Datenspeicher
- Bündeln von WAL-Dateien, sodass sie alle drei Stunden statt einzeln übertragen werden
- Anbindung an andere Sicherungs- und Wiederherstellungssoftware
- Anbindung an Überwachungssoftware zur Fehlermeldung

> **Tipp**
>
> Bei Verwendung eines `archive_command`-Skripts ist es sinnvoll, `logging_collector` zu aktivieren. Alle Nachrichten, die das Skript nach `stderr` schreibt, erscheinen dann im Datenbankserver-Log, sodass komplexe Konfigurationen bei Fehlschlägen leichter diagnostiziert werden können.

### 25.3.8. Einschränkungen

Zum Zeitpunkt dieser Beschreibung gibt es mehrere Einschränkungen der Technik der kontinuierlichen Archivierung. Sie werden wahrscheinlich in künftigen Versionen behoben:

- Wenn während einer Basissicherung ein `CREATE DATABASE`-Befehl ausgeführt wird und anschließend die Template-Datenbank, die von `CREATE DATABASE` kopiert wurde, noch während der laufenden Basissicherung geändert wird, ist es möglich, dass die Recovery diese Änderungen auch in die erzeugte Datenbank überträgt. Das ist natürlich unerwünscht. Um dieses Risiko zu vermeiden, sollten Template-Datenbanken während einer Basissicherung am besten nicht geändert werden.

- `CREATE TABLESPACE`-Befehle werden mit dem wörtlichen absoluten Pfad im WAL protokolliert und daher als Tablespace-Erzeugungen mit demselben absoluten Pfad erneut abgespielt. Das kann unerwünscht sein, wenn das WAL auf einer anderen Maschine abgespielt wird. Es kann sogar gefährlich sein, wenn das WAL auf derselben Maschine, aber in ein neues Datenverzeichnis abgespielt wird: Das Replay überschreibt weiterhin den Inhalt des ursprünglichen Tablespace. Um mögliche Fallen dieser Art zu vermeiden, empfiehlt es sich, nach dem Erstellen oder Löschen von Tablespaces eine neue Basissicherung zu erstellen.

Außerdem ist zu beachten, dass das Standard-WAL-Format ziemlich umfangreich ist, weil es viele Snapshots von Plattenseiten enthält. Diese Seitensnapshots dienen der Crash-Recovery, da möglicherweise teilweise geschriebene Plattenseiten repariert werden müssen. Je nach Hardware und Software Ihres Systems kann das Risiko teilweiser Schreibvorgänge klein genug sein, um es zu ignorieren. In diesem Fall können Sie das Gesamtvolumen archivierter WAL-Dateien deutlich reduzieren, indem Sie Seitensnapshots mit dem Parameter `full_page_writes` ausschalten. Lesen Sie vorher die Hinweise und Warnungen in [Kapitel 28](28_Zuverlässigkeit_und_Write_Ahead_Log.md). Das Ausschalten von Seitensnapshots verhindert die Verwendung des WAL für PITR-Operationen nicht. Ein zukünftiger Entwicklungsbereich besteht darin, archivierte WAL-Daten zu komprimieren, indem unnötige Seitenkopien entfernt werden, selbst wenn `full_page_writes` eingeschaltet ist. Bis dahin möchten Administratoren möglicherweise die Anzahl der in WAL enthaltenen Seitensnapshots verringern, indem sie die Checkpoint-Intervallparameter so weit erhöhen, wie es praktikabel ist.
