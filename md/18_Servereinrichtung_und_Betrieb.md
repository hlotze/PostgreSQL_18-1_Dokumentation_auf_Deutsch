# 18. Servereinrichtung und Betrieb

Dieses Kapitel beschreibt, wie der Datenbankserver eingerichtet und betrieben wird und wie er mit dem Betriebssystem zusammenwirkt.

Die Anleitungen in diesem Kapitel setzen voraus, dass Sie mit einem unveränderten PostgreSQL ohne zusätzliche Infrastruktur arbeiten, zum Beispiel mit einer Version, die Sie nach den Anweisungen in den vorhergehenden Kapiteln aus dem Quellcode gebaut haben. Wenn Sie mit einer vorgepackten oder von einem Anbieter bereitgestellten Version von PostgreSQL arbeiten, hat der Paketbetreuer wahrscheinlich besondere Vorkehrungen getroffen, um den Datenbankserver gemäß den Konventionen Ihres Systems zu installieren und zu starten. Einzelheiten finden Sie in der Dokumentation des jeweiligen Pakets.

## 18.1. Das PostgreSQL-Benutzerkonto

Wie bei jedem Server-Daemon, der von außen erreichbar ist, empfiehlt es sich, PostgreSQL unter einem eigenen Benutzerkonto auszuführen. Dieses Benutzerkonto sollte nur die Daten besitzen, die vom Server verwaltet werden, und nicht mit anderen Daemons geteilt werden. Die Verwendung des Benutzers `nobody` ist beispielsweise keine gute Idee. Insbesondere sollte dieses Benutzerkonto nicht Eigentümer der PostgreSQL-Programmdateien sein, damit ein kompromittierter Serverprozess diese Programme nicht verändern kann.

Vorgepackte Versionen von PostgreSQL legen während der Paketinstallation normalerweise automatisch ein geeignetes Benutzerkonto an.

Um auf einem Unix-System ein Benutzerkonto hinzuzufügen, suchen Sie nach einem Befehl wie `useradd` oder `adduser`. Häufig wird der Benutzername `postgres` verwendet; er wird auch in diesem Buch vorausgesetzt. Sie können aber einen anderen Namen verwenden.

## 18.2. Einen Datenbankcluster erstellen

Bevor Sie irgendetwas tun können, müssen Sie einen Speicherbereich für Datenbanken auf der Platte initialisieren. Wir nennen das einen Datenbankcluster. Der SQL-Standard verwendet dafür den Begriff Katalogcluster. Ein Datenbankcluster ist eine Sammlung von Datenbanken, die von einer einzelnen Instanz eines laufenden Datenbankservers verwaltet wird. Nach der Initialisierung enthält ein Datenbankcluster eine Datenbank namens `postgres`, die als Standarddatenbank für Hilfsprogramme, Benutzer und Anwendungen Dritter gedacht ist. Der Datenbankserver selbst verlangt nicht, dass die Datenbank `postgres` existiert, aber viele externe Hilfsprogramme setzen sie voraus. Außerdem werden bei der Initialisierung in jedem Cluster zwei weitere Datenbanken namens `template1` und `template0` angelegt. Wie ihre Namen andeuten, werden sie als Vorlagen für später erzeugte Datenbanken verwendet; sie sollten nicht für echte Arbeit benutzt werden. Siehe [Kapitel 22](22_Datenbanken_verwalten.md) für Informationen zum Erstellen neuer Datenbanken innerhalb eines Clusters.

Aus Sicht des Dateisystems ist ein Datenbankcluster ein einzelnes Verzeichnis, unterhalb dessen alle Daten gespeichert werden. Wir nennen es Datenverzeichnis oder Datenbereich. Wo Sie Ihre Daten speichern, bleibt vollständig Ihnen überlassen. Es gibt keinen Standardort, auch wenn Orte wie `/usr/local/pgsql/data` oder `/var/lib/pgsql/data` verbreitet sind. Das Datenverzeichnis muss vor der Verwendung mit dem Programm `initdb` initialisiert werden, das zusammen mit PostgreSQL installiert wird.

Wenn Sie eine vorgepackte Version von PostgreSQL verwenden, gibt es möglicherweise eine bestimmte Konvention, wo das Datenverzeichnis liegen soll, und eventuell auch ein Skript zum Erstellen des Datenverzeichnisses. In diesem Fall sollten Sie dieses Skript verwenden, statt `initdb` direkt aufzurufen. Einzelheiten finden Sie in der Paketdokumentation.

Um einen Datenbankcluster manuell zu initialisieren, führen Sie `initdb` aus und geben mit der Option `-D` den gewünschten Dateisystemort des Datenbankclusters an, zum Beispiel:

```text
$ initdb -D /usr/local/pgsql/data
```

Beachten Sie, dass Sie diesen Befehl ausführen müssen, während Sie mit dem im vorherigen Abschnitt beschriebenen PostgreSQL-Benutzerkonto angemeldet sind.

> **Tipp:** Als Alternative zur Option `-D` können Sie die Umgebungsvariable `PGDATA` setzen.

Alternativ können Sie `initdb` über das Programm `pg_ctl` ausführen:

```text
$ pg_ctl -D /usr/local/pgsql/data initdb
```

Das kann intuitiver sein, wenn Sie `pg_ctl` auch zum Starten und Stoppen des Servers verwenden (siehe [Abschnitt 18.3](#183-den-datenbankserver-starten)), sodass `pg_ctl` der einzige Befehl ist, mit dem Sie die Datenbankserverinstanz verwalten.

`initdb` versucht, das angegebene Verzeichnis zu erstellen, falls es noch nicht existiert. Das schlägt natürlich fehl, wenn `initdb` im übergeordneten Verzeichnis keine Schreibrechte hat. Im Allgemeinen ist es empfehlenswert, dass der PostgreSQL-Benutzer nicht nur Eigentümer des Datenverzeichnisses ist, sondern auch des übergeordneten Verzeichnisses, damit dieses Problem nicht auftritt. Wenn auch das gewünschte übergeordnete Verzeichnis noch nicht existiert, müssen Sie es zuerst erstellen, gegebenenfalls mit Root-Rechten, falls das darüberliegende Verzeichnis nicht beschreibbar ist. Der Ablauf könnte also so aussehen:

```text
root# mkdir /usr/local/pgsql
root# chown postgres /usr/local/pgsql
root# su postgres
postgres$ initdb -D /usr/local/pgsql/data
```

`initdb` weigert sich zu laufen, wenn das Datenverzeichnis existiert und bereits Dateien enthält. Damit wird verhindert, dass eine vorhandene Installation versehentlich überschrieben wird.

Da das Datenverzeichnis alle in der Datenbank gespeicherten Daten enthält, muss es unbedingt vor unbefugtem Zugriff geschützt werden. `initdb` entzieht daher allen außer dem PostgreSQL-Benutzer und optional dessen Gruppe die Zugriffsrechte. Wenn Gruppenzugriff aktiviert ist, ist er nur lesend. Dadurch kann ein nicht privilegierter Benutzer in derselben Gruppe wie der Cluster-Eigentümer eine Sicherung der Clusterdaten erstellen oder andere Operationen ausführen, die nur Lesezugriff benötigen.

Beachten Sie, dass das Aktivieren oder Deaktivieren des Gruppenzugriffs bei einem bestehenden Cluster erfordert, dass der Cluster heruntergefahren wird und der passende Modus für alle Verzeichnisse und Dateien gesetzt wird, bevor PostgreSQL neu gestartet wird. Andernfalls können im Datenverzeichnis gemischte Modi entstehen. Für Cluster, die nur dem Eigentümer Zugriff erlauben, sind die passenden Modi `0700` für Verzeichnisse und `0600` für Dateien. Für Cluster, die zusätzlich Leserechte für die Gruppe erlauben, sind die passenden Modi `0750` für Verzeichnisse und `0640` für Dateien.

Obwohl die Verzeichnisinhalte geschützt sind, erlaubt die voreingestellte Client-Authentifizierung jedem lokalen Benutzer, sich mit der Datenbank zu verbinden und sogar zum Datenbank-Superuser zu werden. Wenn Sie anderen lokalen Benutzern nicht vertrauen, empfehlen wir, eine der Optionen `-W`, `--pwprompt` oder `--pwfile` von `initdb` zu verwenden, um dem Datenbank-Superuser ein Passwort zuzuweisen. Geben Sie außerdem `-A scram-sha-256` an, damit nicht der Standardmodus `trust` verwendet wird. Alternativ können Sie die erzeugte Datei `pg_hba.conf` nach dem Ausführen von `initdb`, aber vor dem ersten Start des Servers ändern. Andere sinnvolle Ansätze sind etwa Peer-Authentifizierung oder Dateisystemrechte zur Beschränkung von Verbindungen. Siehe [Kapitel 20](20_Client_Authentifizierung.md) für weitere Informationen.

`initdb` initialisiert außerdem die Standard-Locale für den Datenbankcluster. Normalerweise übernimmt es die Locale-Einstellungen aus der Umgebung und wendet sie auf die initialisierte Datenbank an. Es ist möglich, eine andere Locale für die Datenbank anzugeben; weitere Informationen dazu finden Sie in [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung). Die Standardsortierreihenfolge innerhalb des jeweiligen Datenbankclusters wird von `initdb` festgelegt. Zwar können Sie neue Datenbanken mit anderer Sortierreihenfolge erstellen, aber die Sortierreihenfolge der von `initdb` erzeugten Vorlagendatenbanken lässt sich nicht ändern, ohne sie zu löschen und neu zu erstellen. Außerdem hat die Verwendung anderer Locales als `C` oder `POSIX` Auswirkungen auf die Performance. Deshalb ist es wichtig, diese Entscheidung beim ersten Mal richtig zu treffen.

`initdb` legt auch die Standard-Zeichensatzkodierung für den Datenbankcluster fest. Normalerweise sollte sie zur Locale-Einstellung passen. Einzelheiten finden Sie in [Abschnitt 23.3](23_Lokalisierung.md#233-zeichensatzunterstützung).

Nicht-`C`- und Nicht-`POSIX`-Locales stützen sich für die Zeichensortierung auf die Kollationsbibliothek des Betriebssystems. Das steuert die Reihenfolge von Schlüsseln, die in Indizes gespeichert werden. Deshalb kann ein Cluster nicht auf eine inkompatible Version der Kollationsbibliothek wechseln, weder durch Wiederherstellung eines Snapshots, binäre Streaming-Replikation, ein anderes Betriebssystem noch durch ein Betriebssystem-Upgrade.

### 18.2.1. Verwendung sekundärer Dateisysteme

Viele Installationen legen ihre Datenbankcluster auf Dateisystemen oder Volumes an, die nicht das Root-Volume der Maschine sind. Wenn Sie das tun, ist es nicht ratsam, das oberste Verzeichnis des sekundären Volumes, also den Einhängepunkt, direkt als Datenverzeichnis zu verwenden. Bewährte Praxis ist es, im Einhängepunkt ein Verzeichnis anzulegen, das dem PostgreSQL-Benutzer gehört, und darin dann das Datenverzeichnis zu erstellen. Das vermeidet Rechteprobleme, insbesondere bei Operationen wie `pg_upgrade`, und stellt außerdem sicher, dass Fehler sauber auftreten, falls das sekundäre Volume offline genommen wird.

### 18.2.2. Dateisysteme

Im Allgemeinen kann jedes Dateisystem mit POSIX-Semantik für PostgreSQL verwendet werden. Benutzer bevorzugen aus verschiedenen Gründen unterschiedliche Dateisysteme, etwa wegen Herstellerunterstützung, Performance oder Vertrautheit. Die Erfahrung zeigt, dass man bei ansonsten gleichen Bedingungen keine großen Performance- oder Verhaltensänderungen allein dadurch erwarten sollte, dass man das Dateisystem wechselt oder kleinere Dateisystemkonfigurationen ändert.

#### 18.2.2.1. NFS

Es ist möglich, ein NFS-Dateisystem zum Speichern des PostgreSQL-Datenverzeichnisses zu verwenden. PostgreSQL tut für NFS-Dateisysteme nichts Besonderes; es nimmt also an, dass NFS sich genau wie lokal angeschlossene Laufwerke verhält. PostgreSQL verwendet keine Funktionen, von denen bekannt ist, dass sie sich auf NFS nicht standardkonform verhalten, zum Beispiel Dateisperren.

Die einzige feste Voraussetzung für NFS mit PostgreSQL ist, dass das Dateisystem mit der Option `hard` eingehängt wird. Mit `hard` können Prozesse bei Netzwerkproblemen unbegrenzt hängen bleiben; diese Konfiguration erfordert daher eine sorgfältige Überwachung. Die Option `soft` unterbricht Systemaufrufe bei Netzwerkproblemen, aber PostgreSQL wiederholt solche unterbrochenen Systemaufrufe nicht. Jede solche Unterbrechung führt daher zu einem gemeldeten I/O-Fehler.

Die Mount-Option `sync` muss nicht verwendet werden. Das Verhalten von `async` ist ausreichend, weil PostgreSQL zu passenden Zeitpunkten `fsync`-Aufrufe ausgibt, um Schreib-Caches zu leeren. Das entspricht der Arbeitsweise auf lokalen Dateisystemen. Auf dem NFS-Server wird jedoch dringend empfohlen, auf Systemen, auf denen sie existiert, die Export-Option `sync` zu verwenden, vor allem unter Linux. Andernfalls ist nicht garantiert, dass ein `fsync` oder Äquivalent auf dem NFS-Client tatsächlich den dauerhaften Speicher auf dem Server erreicht. Das könnte zu Beschädigungen führen, ähnlich wie ein Betrieb mit dem Parameter `fsync` auf `off`. Die Voreinstellungen dieser Mount- und Export-Optionen unterscheiden sich je nach Anbieter und Version. Prüfen und gegebenenfalls explizites Setzen ist daher in jedem Fall empfehlenswert, um Mehrdeutigkeiten zu vermeiden.

In manchen Fällen kann ein externes Speicherprodukt entweder über NFS oder über ein niedrigeres Protokoll wie iSCSI angesprochen werden. Im letzteren Fall erscheint der Speicher als Blockgerät, auf dem jedes verfügbare Dateisystem angelegt werden kann. Dieser Ansatz kann dem DBA einige Besonderheiten von NFS ersparen; die Komplexität der Verwaltung entfernten Speichers verlagert sich dann allerdings auf andere Ebenen.

## 18.3. Den Datenbankserver starten

Bevor jemand auf die Datenbank zugreifen kann, müssen Sie den Datenbankserver starten. Das Datenbankserverprogramm heißt `postgres`.

Wenn Sie eine vorgepackte Version von PostgreSQL verwenden, enthält sie mit hoher Wahrscheinlichkeit Vorkehrungen, um den Server nach den Konventionen Ihres Betriebssystems als Hintergrundprozess zu betreiben. Die Infrastruktur des Pakets zum Starten des Servers zu verwenden, ist viel weniger Arbeit, als das selbst einzurichten. Einzelheiten finden Sie in der Paketdokumentation.

Die einfachste manuelle Methode zum Starten des Servers ist, `postgres` direkt aufzurufen und mit der Option `-D` den Ort des Datenverzeichnisses anzugeben, zum Beispiel:

```text
$ postgres -D /usr/local/pgsql/data
```

Dadurch läuft der Server im Vordergrund. Das muss geschehen, während Sie mit dem PostgreSQL-Benutzerkonto angemeldet sind. Ohne `-D` versucht der Server, das Datenverzeichnis aus der Umgebungsvariable `PGDATA` zu verwenden. Wenn auch diese Variable nicht gesetzt ist, schlägt der Start fehl.

Normalerweise ist es besser, `postgres` im Hintergrund zu starten. Verwenden Sie dafür die übliche Unix-Shell-Syntax:

```text
$ postgres -D /usr/local/pgsql/data >logfile 2>&1 &
```

Es ist wichtig, die Ausgaben des Servers auf `stdout` und `stderr` irgendwo zu speichern, wie oben gezeigt. Das hilft bei Prüfzwecken und bei der Diagnose von Problemen. Siehe [Abschnitt 24.3](24_Regelmäßige_Datenbankwartung.md#243-logdateiwartung) für eine ausführlichere Diskussion der Logdateiverwaltung.

Das Programm `postgres` akzeptiert außerdem eine Reihe weiterer Kommandozeilenoptionen. Weitere Informationen finden Sie auf der Referenzseite zu `postgres` und im folgenden [Kapitel 19](19_Serverkonfiguration.md).

Diese Shell-Syntax wird schnell mühsam. Daher gibt es das Wrapper-Programm `pg_ctl`, das einige Aufgaben vereinfacht. Zum Beispiel:

```text
pg_ctl start -l logfile
```

startet den Server im Hintergrund und schreibt die Ausgabe in die angegebene Logdatei. Die Option `-D` hat hier dieselbe Bedeutung wie bei `postgres`. `pg_ctl` kann den Server auch stoppen.

Normalerweise wollen Sie den Datenbankserver beim Booten des Rechners starten. Autostart-Skripte sind betriebssystemspezifisch. Einige Beispielskripte werden mit PostgreSQL im Verzeichnis `contrib/start-scripts` ausgeliefert. Für ihre Installation benötigen Sie Root-Rechte.

Verschiedene Systeme haben unterschiedliche Konventionen zum Starten von Daemons beim Booten. Viele Systeme haben eine Datei `/etc/rc.local` oder `/etc/rc.d/rc.local`; andere verwenden Verzeichnisse wie `init.d` oder `rc.d`. Was immer Sie tun: Der Server muss unter dem PostgreSQL-Benutzerkonto laufen, nicht als `root` oder ein anderer Benutzer. Daher sollten Sie Ihre Befehle wahrscheinlich mit `su postgres -c '...'` formulieren. Zum Beispiel:

```text
su postgres -c 'pg_ctl start -D /usr/local/pgsql/data -l serverlog'
```

Hier sind einige weitere betriebssystemspezifische Hinweise. Verwenden Sie jeweils das richtige Installationsverzeichnis und den richtigen Benutzernamen, wenn hier generische Werte gezeigt werden.

- Für FreeBSD sehen Sie sich die Datei `contrib/start-scripts/freebsd` in der PostgreSQL-Quelldistribution an.

- Unter OpenBSD fügen Sie der Datei `/etc/rc.local` die folgenden Zeilen hinzu:

  ```text
  if [ -x /usr/local/pgsql/bin/pg_ctl -a -x /usr/local/pgsql/bin/postgres ]; then
      su -l postgres -c '/usr/local/pgsql/bin/pg_ctl start -s -l /var/postgresql/log -D /usr/local/pgsql/data'
      echo -n ' postgresql'
  fi
  ```

- Unter Linux-Systemen fügen Sie entweder

  ```text
  /usr/local/pgsql/bin/pg_ctl start -l logfile -D /usr/local/pgsql/data
  ```

  zu `/etc/rc.d/rc.local` oder `/etc/rc.local` hinzu, oder Sie sehen sich die Datei `contrib/start-scripts/linux` in der PostgreSQL-Quelldistribution an.

  Bei Verwendung von `systemd` können Sie die folgende Service-Unit-Datei verwenden, zum Beispiel unter `/etc/systemd/system/postgresql.service`:

  ```text
  [Unit]
  Description=PostgreSQL database server
  Documentation=man:postgres(1)
  After=network-online.target
  Wants=network-online.target

  [Service]
  Type=notify
  User=postgres
  ExecStart=/usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data
  ExecReload=/bin/kill -HUP $MAINPID
  KillMode=mixed
  KillSignal=SIGINT
  TimeoutSec=infinity

  [Install]
  WantedBy=multi-user.target
  ```

  Die Verwendung von `Type=notify` setzt voraus, dass die Server-Binärdatei mit `configure --with-systemd` gebaut wurde.

  Prüfen Sie die Timeout-Einstellung sorgfältig. `systemd` hat zum Zeitpunkt dieser Dokumentation standardmäßig ein Timeout von 90 Sekunden und beendet einen Prozess, der innerhalb dieser Zeit keine Bereitschaft meldet. Ein PostgreSQL-Server, der beim Start eine Crash-Recovery durchführen muss, kann aber deutlich länger brauchen, bis er bereit ist. Der vorgeschlagene Wert `infinity` deaktiviert diese Timeout-Logik.

- Unter NetBSD verwenden Sie je nach Vorliebe entweder die FreeBSD- oder die Linux-Startskripte.

- Unter Solaris erstellen Sie eine Datei namens `/etc/init.d/postgresql`, die die folgende Zeile enthält:

  ```text
  su - postgres -c "/usr/local/pgsql/bin/pg_ctl start -l logfile -D /usr/local/pgsql/data"
  ```

  Erstellen Sie anschließend einen symbolischen Link darauf in `/etc/rc3.d` als `S99postgresql`.

Während der Server läuft, wird seine PID in der Datei `postmaster.pid` im Datenverzeichnis gespeichert. Das verhindert, dass mehrere Serverinstanzen im selben Datenverzeichnis laufen, und kann auch zum Herunterfahren des Servers verwendet werden.

### 18.3.1. Serverstartfehler

Es gibt mehrere häufige Gründe, warum der Server nicht startet. Prüfen Sie die Logdatei des Servers oder starten Sie ihn von Hand ohne Umleitung von Standardausgabe und Standardfehler und sehen Sie sich die Fehlermeldungen an. Im Folgenden werden einige der häufigsten Fehlermeldungen genauer erläutert.

```text
LOG: could not bind IPv4 address "127.0.0.1": Address already in use
HINT: Is another postmaster already running on port 5432? If not, wait a few seconds and retry.
FATAL: could not create any TCP/IP sockets
```

Das bedeutet meistens genau das, was es sagt: Sie haben versucht, einen weiteren Server auf demselben Port zu starten, auf dem bereits einer läuft. Wenn die Kernel-Fehlermeldung jedoch nicht `Address already in use` oder eine Variante davon ist, kann ein anderes Problem vorliegen. Der Versuch, einen Server auf einem reservierten Port zu starten, könnte zum Beispiel Folgendes erzeugen:

```text
$ postgres -p 666
LOG: could not bind IPv4 address "127.0.0.1": Permission denied
HINT: Is another postmaster already running on port 666? If not, wait a few seconds and retry.
FATAL: could not create any TCP/IP sockets
```

Eine Meldung wie:

```text
FATAL: could not create shared memory segment: Invalid argument
DETAIL: Failed system call was shmget(key=5440001, size=4011376640, 03600).
```

bedeutet wahrscheinlich, dass die Kernel-Grenze für die Größe von Shared Memory kleiner ist als der Arbeitsbereich, den PostgreSQL erstellen will (in diesem Beispiel `4011376640` Byte). Das ist nur wahrscheinlich, wenn Sie `shared_memory_type` auf `sysv` gesetzt haben. In diesem Fall können Sie versuchen, den Server mit weniger Puffern als üblich (`shared_buffers`) zu starten oder Ihren Kernel so zu konfigurieren, dass größere Shared-Memory-Bereiche erlaubt sind. Sie können diese Meldung auch sehen, wenn Sie mehrere Server auf derselben Maschine starten und deren insgesamt angeforderter Speicher die Kernel-Grenze überschreitet.

Ein Fehler wie:

```text
FATAL: could not create semaphores: No space left on device
DETAIL: Failed system call was semget(5440126, 17, 03600).
```

bedeutet nicht, dass der Plattenspeicher erschöpft ist. Er bedeutet, dass die Kernel-Grenze für die Anzahl der System-V-Semaphoren kleiner ist als die Anzahl, die PostgreSQL erstellen will. Wie oben können Sie das Problem vorübergehend umgehen, indem Sie den Server mit weniger erlaubten Verbindungen (`max_connections`) starten; dauerhaft sollten Sie jedoch die Kernel-Grenze erhöhen.

Einzelheiten zur Konfiguration von System-V-IPC-Funktionen finden Sie in [Abschnitt 18.4.1](#1841-shared-memory-und-semaphoren).

### 18.3.2. Client-Verbindungsprobleme

Obwohl mögliche Fehlerbedingungen auf der Client-Seite sehr vielfältig und anwendungsabhängig sind, können einige direkt damit zusammenhängen, wie der Server gestartet wurde. Andere als die unten gezeigten Bedingungen sollten in der jeweiligen Client-Anwendung dokumentiert sein.

```text
psql: error: connection to server at "server.joe.com" (123.123.123.123), port 5432 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
```

Das ist der allgemeine Fehler „Ich konnte keinen Server finden, mit dem ich sprechen kann“. Er sieht so aus, wenn TCP/IP-Kommunikation versucht wird. Ein häufiger Fehler ist, zu vergessen, `listen_addresses` so zu konfigurieren, dass der Server entfernte TCP-Verbindungen akzeptiert.

Alternativ kann beim Versuch einer Unix-Domain-Socket-Kommunikation zu einem lokalen Server Folgendes erscheinen:

```text
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: No such file or directory
        Is the server running locally and accepting connections on that socket?
```

Wenn der Server tatsächlich läuft, prüfen Sie, ob die Vorstellung des Clients vom Socket-Pfad (hier `/tmp`) zur Einstellung `unix_socket_directories` des Servers passt.

Eine Verbindungsfehlermeldung zeigt immer die Serveradresse oder den Socket-Pfadnamen an. Das ist nützlich, um zu prüfen, ob der Client wirklich am richtigen Ort zu verbinden versucht. Wenn dort tatsächlich kein Server lauscht, lautet die Kernel-Fehlermeldung typischerweise `Connection refused` oder `No such file or directory`, wie oben gezeigt. Wichtig ist: `Connection refused` bedeutet in diesem Zusammenhang nicht, dass der Server Ihre Verbindungsanfrage erhalten und abgelehnt hat. Dieser Fall erzeugt eine andere Meldung, wie in [Abschnitt 20.16](20_Client_Authentifizierung.md#2016-authentifizierungsprobleme) gezeigt. Andere Fehlermeldungen wie `Connection timed out` können auf grundlegendere Probleme hinweisen, etwa fehlende Netzwerkverbindung oder eine Firewall, die die Verbindung blockiert.

## 18.4. Kernel-Ressourcen verwalten

PostgreSQL kann gelegentlich verschiedene Ressourcenlimits des Betriebssystems ausschöpfen, besonders wenn mehrere Kopien des Servers auf demselben System laufen oder bei sehr großen Installationen. Dieser Abschnitt erklärt die von PostgreSQL verwendeten Kernel-Ressourcen und die Schritte, mit denen Sie Probleme im Zusammenhang mit Kernel-Ressourcenverbrauch beheben können.

### 18.4.1. Shared Memory und Semaphoren

PostgreSQL benötigt vom Betriebssystem Funktionen zur Interprozesskommunikation (IPC), insbesondere Shared Memory und Semaphoren. Unix-abgeleitete Systeme stellen typischerweise „System V“-IPC, „POSIX“-IPC oder beides bereit. Windows hat eine eigene Implementierung dieser Funktionen und wird hier nicht behandelt.

Standardmäßig reserviert PostgreSQL eine sehr kleine Menge System-V-Shared-Memory sowie eine deutlich größere Menge anonymes `mmap`-Shared-Memory. Alternativ kann ein einzelner großer System-V-Shared-Memory-Bereich verwendet werden (siehe `shared_memory_type`). Zusätzlich wird beim Serverstart eine beträchtliche Zahl von Semaphoren erzeugt, die entweder im System-V- oder POSIX-Stil vorliegen können. Derzeit werden POSIX-Semaphoren auf Linux- und FreeBSD-Systemen verwendet, während andere Plattformen System-V-Semaphoren verwenden.

System-V-IPC-Funktionen werden typischerweise durch systemweite Zuweisungsgrenzen eingeschränkt. Wenn PostgreSQL eine dieser Grenzen überschreitet, verweigert der Server den Start und sollte eine hilfreiche Fehlermeldung ausgeben, die das Problem und mögliche Abhilfe beschreibt. Siehe auch [Abschnitt 18.3.1](#1831-serverstartfehler). Die relevanten Kernel-Parameter haben über verschiedene Systeme hinweg konsistente Namen; Tabelle 18.1 gibt einen Überblick. Die Methoden, um sie zu setzen, unterscheiden sich jedoch. Für einige Plattformen folgen unten Hinweise.

**Tabelle 18.1. System-V-IPC-Parameter**

| Name | Beschreibung | Benötigte Werte zum Betrieb einer PostgreSQL-Instanz |
|---|---|---|
| `SHMMAX` | Maximale Größe eines Shared-Memory-Segments (Byte) | mindestens 1 kB, der Standard ist aber meist viel höher |
| `SHMMIN` | Minimale Größe eines Shared-Memory-Segments (Byte) | 1 |
| `SHMALL` | Insgesamt verfügbarer Shared Memory (Byte oder Seiten) | wie `SHMMAX`, wenn in Byte gemessen; oder `ceil(SHMMAX/PAGE_SIZE)`, wenn in Seiten gemessen; plus Platz für andere Anwendungen |
| `SHMSEG` | Maximale Anzahl von Shared-Memory-Segmenten pro Prozess | nur 1 Segment wird benötigt, der Standard ist aber meist viel höher |
| `SHMMNI` | Maximale Anzahl von Shared-Memory-Segmenten systemweit | wie `SHMSEG` plus Platz für andere Anwendungen |
| `SEMMNI` | Maximale Anzahl von Semaphor-Identifikatoren, also Sets | mindestens `ceil(num_os_semaphores / 16)` plus Platz für andere Anwendungen |
| `SEMMNS` | Maximale Anzahl von Semaphoren systemweit | `ceil(num_os_semaphores / 16) * 17` plus Platz für andere Anwendungen |
| `SEMMSL` | Maximale Anzahl von Semaphoren pro Set | mindestens 17 |
| `SEMMAP` | Anzahl der Einträge in der Semaphor-Map | siehe Text |
| `SEMVMX` | Maximaler Semaphorwert | mindestens 1000; Standard ist oft 32767, nur bei Bedarf ändern |

PostgreSQL benötigt für jede Serverkopie einige Byte System-V-Shared-Memory, typischerweise 48 Byte auf 64-Bit-Plattformen. Auf den meisten modernen Betriebssystemen lässt sich diese Menge problemlos reservieren. Wenn Sie jedoch viele Serverkopien betreiben oder den Server explizit so konfigurieren, dass große Mengen System-V-Shared-Memory verwendet werden (siehe `shared_memory_type` und `dynamic_shared_memory_type`), kann es nötig sein, `SHMALL` zu erhöhen, also die systemweite Gesamtmenge an System-V-Shared-Memory. Beachten Sie, dass `SHMALL` auf vielen Systemen in Seiten und nicht in Byte gemessen wird.

Weniger wahrscheinlich problematisch ist die Mindestgröße für Shared-Memory-Segmente (`SHMMIN`), die für PostgreSQL höchstens etwa 32 Byte betragen sollte; üblicherweise ist sie einfach 1. Die maximale Anzahl von Segmenten systemweit (`SHMMNI`) oder pro Prozess (`SHMSEG`) verursacht wahrscheinlich kein Problem, es sei denn, Ihr System hat sie auf null gesetzt.

Bei System-V-Semaphoren verwendet PostgreSQL je einen Semaphor pro erlaubter Verbindung (`max_connections`), erlaubtem Autovacuum-Worker-Prozess (`autovacuum_worker_slots`), erlaubtem WAL-Sender-Prozess (`max_wal_senders`), erlaubtem Hintergrundprozess (`max_worker_processes`) usw., jeweils in Sets zu 16. Der zur Laufzeit berechnete Parameter `num_os_semaphores` meldet die erforderliche Anzahl von Semaphoren. Dieser Parameter kann vor dem Start des Servers mit einem `postgres`-Befehl wie diesem angezeigt werden:

```text
$ postgres -D $PGDATA -C num_os_semaphores
```

Jedes Set von 16 Semaphoren enthält außerdem einen 17. Semaphor mit einer „magischen Zahl“, um Kollisionen mit von anderen Anwendungen verwendeten Semaphor-Sets zu erkennen. Die maximale Anzahl von Semaphoren im System wird durch `SEMMNS` festgelegt und muss daher mindestens so hoch sein wie `num_os_semaphores` plus ein zusätzlicher Semaphor für jedes benötigte Set von 16 Semaphoren (siehe die Formel in Tabelle 18.1). Der Parameter `SEMMNI` bestimmt die Grenze für die Anzahl der Semaphor-Sets, die gleichzeitig im System existieren können. Dieser Parameter muss daher mindestens `ceil(num_os_semaphores / 16)` betragen. Das Senken der Zahl erlaubter Verbindungen ist eine vorübergehende Umgehung für Fehler, die meist irreführend als `No space left on device` von der Funktion `semget` formuliert werden.

In einigen Fällen kann es auch nötig sein, `SEMMAP` mindestens in der Größenordnung von `SEMMNS` zu erhöhen. Wenn das System diesen Parameter hat (viele haben ihn nicht), definiert er die Größe der Semaphor-Ressourcen-Map, in der jeder zusammenhängende Block verfügbarer Semaphoren einen Eintrag benötigt. Wenn ein Semaphor-Set freigegeben wird, wird es entweder einem bestehenden benachbarten Eintrag hinzugefügt oder unter einem neuen Map-Eintrag registriert. Ist die Map voll, gehen die freigegebenen Semaphoren bis zum Neustart verloren. Fragmentierung des Semaphor-Raums kann im Laufe der Zeit dazu führen, dass weniger Semaphoren verfügbar sind, als eigentlich vorhanden sein sollten.

Verschiedene andere Einstellungen im Zusammenhang mit „semaphore undo“, etwa `SEMMNU` und `SEMUME`, wirken sich nicht auf PostgreSQL aus.

Bei POSIX-Semaphoren ist die benötigte Zahl dieselbe wie bei System V: ein Semaphor pro erlaubter Verbindung (`max_connections`), erlaubtem Autovacuum-Worker-Prozess (`autovacuum_worker_slots`), erlaubtem WAL-Sender-Prozess (`max_wal_senders`), erlaubtem Hintergrundprozess (`max_worker_processes`) usw. Auf den Plattformen, auf denen diese Option bevorzugt wird, gibt es kein spezifisches Kernel-Limit für die Anzahl der POSIX-Semaphoren.

**FreeBSD**

Die Standardwerte für Shared Memory reichen normalerweise aus, es sei denn, Sie haben `shared_memory_type` auf `sysv` gesetzt. System-V-Semaphoren werden auf dieser Plattform nicht verwendet.

Die Standard-IPC-Einstellungen können über die Schnittstellen `sysctl` oder `loader` geändert werden. Die folgenden Parameter können mit `sysctl` gesetzt werden:

```text
# sysctl kern.ipc.shmall=32768
# sysctl kern.ipc.shmmax=134217728
```

Um diese Einstellungen über Neustarts hinweg beizubehalten, ändern Sie `/etc/sysctl.conf`.

Wenn Sie `shared_memory_type` auf `sysv` gesetzt haben, möchten Sie eventuell außerdem Ihren Kernel so konfigurieren, dass System-V-Shared-Memory im RAM gesperrt und nicht in den Swap ausgelagert wird. Das kann mit der `sysctl`-Einstellung `kern.ipc.shm_use_phys` erreicht werden.

Wenn PostgreSQL in einer FreeBSD-Jail läuft, sollten Sie deren Parameter `sysvshm` auf `new` setzen, sodass sie einen eigenen System-V-Shared-Memory-Namensraum hat. Vor FreeBSD 11.0 war es nötig, den gemeinsamen Zugriff aus Jails auf den IPC-Namensraum des Hosts zu aktivieren und Maßnahmen gegen Kollisionen zu treffen.

**NetBSD**

Die Standardwerte für Shared Memory reichen normalerweise aus, es sei denn, Sie haben `shared_memory_type` auf `sysv` gesetzt. Sie müssen jedoch `kern.ipc.semmni` und `kern.ipc.semmns` erhöhen, da die NetBSD-Standardwerte dafür unbrauchbar klein sind.

IPC-Parameter können mit `sysctl` angepasst werden, zum Beispiel:

```text
# sysctl -w kern.ipc.semmni=100
```

Um diese Einstellungen über Neustarts hinweg beizubehalten, ändern Sie `/etc/sysctl.conf`.

Wenn Sie `shared_memory_type` auf `sysv` gesetzt haben, möchten Sie eventuell außerdem Ihren Kernel so konfigurieren, dass System-V-Shared-Memory im RAM gesperrt und nicht in den Swap ausgelagert wird. Das kann mit der `sysctl`-Einstellung `kern.ipc.shm_use_phys` erreicht werden.

**OpenBSD**

Die Standardwerte für Shared Memory reichen normalerweise aus, es sei denn, Sie haben `shared_memory_type` auf `sysv` gesetzt. Sie müssen jedoch `kern.seminfo.semmni` und `kern.seminfo.semmns` erhöhen, da die OpenBSD-Standardwerte dafür unbrauchbar klein sind.

IPC-Parameter können mit `sysctl` angepasst werden, zum Beispiel:

```text
# sysctl kern.seminfo.semmni=100
```

Um diese Einstellungen über Neustarts hinweg beizubehalten, ändern Sie `/etc/sysctl.conf`.

**Linux**

Die Standardwerte für Shared Memory reichen normalerweise aus, es sei denn, Sie haben `shared_memory_type` auf `sysv` gesetzt, und selbst dann nur auf älteren Kernelversionen, die mit niedrigen Standardwerten ausgeliefert wurden. System-V-Semaphoren werden auf dieser Plattform nicht verwendet.

Die Einstellungen für die Shared-Memory-Größe können über die `sysctl`-Schnittstelle geändert werden. Um zum Beispiel 16 GB zu erlauben:

```text
$ sysctl -w kernel.shmmax=17179869184
$ sysctl -w kernel.shmall=4194304
```

Um diese Einstellungen über Neustarts hinweg beizubehalten, siehe `/etc/sysctl.conf`.

**macOS**

Die Standardwerte für Shared Memory und Semaphoren reichen normalerweise aus, es sei denn, Sie haben `shared_memory_type` auf `sysv` gesetzt.

Die empfohlene Methode zum Konfigurieren von Shared Memory unter macOS besteht darin, eine Datei namens `/etc/sysctl.conf` mit Variablenzuweisungen wie diesen zu erstellen:

```text
kern.sysv.shmmax=4194304
kern.sysv.shmmin=1
kern.sysv.shmmni=32
kern.sysv.shmseg=8
kern.sysv.shmall=1024
```

Beachten Sie, dass in einigen macOS-Versionen alle fünf Shared-Memory-Parameter in `/etc/sysctl.conf` gesetzt sein müssen; andernfalls werden die Werte ignoriert.

`SHMMAX` kann nur auf ein Vielfaches von 4096 gesetzt werden.

`SHMALL` wird auf dieser Plattform in 4-kB-Seiten gemessen.

Es ist möglich, alle Parameter außer `SHMMNI` im laufenden Betrieb mit `sysctl` zu ändern. Trotzdem ist es am besten, die gewünschten Werte über `/etc/sysctl.conf` einzurichten, damit sie Neustarts überdauern.

**Solaris und illumos**

Die Standardwerte für Shared Memory und Semaphoren reichen für die meisten PostgreSQL-Anwendungen normalerweise aus. Solaris setzt `SHMMAX` standardmäßig auf ein Viertel des System-RAM. Um diese Einstellung weiter anzupassen, verwenden Sie eine Projekteinstellung, die dem Benutzer `postgres` zugeordnet ist. Führen Sie zum Beispiel Folgendes als `root` aus:

```text
projadd -c "PostgreSQL DB User" -K "project.max-shm-memory=(privileged,8GB,deny)" -U postgres -G postgres user.postgres
```

Dieser Befehl fügt das Projekt `user.postgres` hinzu und setzt das Shared-Memory-Maximum für den Benutzer `postgres` auf 8 GB. Er wird wirksam, wenn sich dieser Benutzer das nächste Mal anmeldet oder wenn Sie PostgreSQL neu starten, nicht beim Neuladen. Obiges Beispiel setzt voraus, dass PostgreSQL vom Benutzer `postgres` in der Gruppe `postgres` ausgeführt wird. Ein Neustart des Servers ist nicht erforderlich.

Weitere empfohlene Kernel-Einstellungen für Datenbankserver mit vielen Verbindungen sind:

```text
project.max-shm-ids=(priv,32768,deny)
project.max-sem-ids=(priv,4096,deny)
project.max-msg-ids=(priv,4096,deny)
```

Wenn Sie PostgreSQL in einer Zone ausführen, müssen Sie möglicherweise außerdem die Ressourcennutzungslimits der Zone erhöhen. Weitere Informationen zu Projekten und `prctl` finden Sie in „Chapter 2: Projects and Tasks“ im System Administrator's Guide.

### 18.4.2. systemd RemoveIPC

Wenn `systemd` verwendet wird, muss darauf geachtet werden, dass IPC-Ressourcen einschließlich Shared Memory nicht vorzeitig vom Betriebssystem entfernt werden. Das ist besonders relevant, wenn PostgreSQL aus dem Quellcode installiert wird. Benutzer von Distributionspaketen von PostgreSQL sind seltener betroffen, da der Benutzer `postgres` dann normalerweise als Systembenutzer angelegt wird.

Die Einstellung `RemoveIPC` in `logind.conf` steuert, ob IPC-Objekte entfernt werden, wenn sich ein Benutzer vollständig abmeldet. Systembenutzer sind ausgenommen. In unverändertem `systemd` ist diese Einstellung standardmäßig `on`, aber einige Betriebssystemdistributionen setzen sie standardmäßig auf `off`.

Ein typischer beobachteter Effekt bei aktivierter Einstellung ist, dass Shared-Memory-Objekte, die für parallele Abfrageausführung verwendet werden, scheinbar zufällig entfernt werden. Das führt beim Öffnen und Entfernen zu Fehlern und Warnungen wie:

```text
WARNING: could not remove shared memory segment "/PostgreSQL.1450751626": No such file or directory
```

Verschiedene Arten von IPC-Objekten, etwa Shared Memory gegenüber Semaphoren oder System V gegenüber POSIX, werden von `systemd` leicht unterschiedlich behandelt. Man kann daher beobachten, dass manche IPC-Ressourcen nicht auf dieselbe Weise entfernt werden wie andere. Es ist aber nicht ratsam, sich auf solche feinen Unterschiede zu verlassen.

Eine „Abmeldung eines Benutzers“ kann im Rahmen eines Wartungsjobs passieren oder manuell, wenn sich ein Administrator als Benutzer `postgres` oder ähnlich anmeldet. Das lässt sich daher allgemein schwer verhindern.

Was ein „Systembenutzer“ ist, wird zur Kompilierzeit von `systemd` aus der Einstellung `SYS_UID_MAX` in `/etc/login.defs` bestimmt.

Paketierungs- und Bereitstellungsskripte sollten darauf achten, den Benutzer `postgres` mit `useradd -r`, `adduser --system` oder einem Äquivalent als Systembenutzer anzulegen.

Alternativ wird empfohlen, wenn das Benutzerkonto falsch angelegt wurde oder nicht geändert werden kann,

```text
RemoveIPC=no
```

in `/etc/systemd/logind.conf` oder einer anderen passenden Konfigurationsdatei zu setzen.

> **Vorsicht:** Mindestens eines dieser beiden Dinge muss sichergestellt sein, sonst wird der PostgreSQL-Server sehr unzuverlässig.

### 18.4.3. Ressourcenlimits

Unix-artige Betriebssysteme erzwingen verschiedene Arten von Ressourcenlimits, die den Betrieb Ihres PostgreSQL-Servers beeinträchtigen können. Besonders wichtig sind Grenzen für die Anzahl der Prozesse pro Benutzer, die Anzahl geöffneter Dateien pro Prozess und die einem Prozess verfügbare Speichermenge. Jede dieser Grenzen hat ein „hartes“ und ein „weiches“ Limit. Das weiche Limit zählt tatsächlich, kann aber vom Benutzer bis zum harten Limit geändert werden. Das harte Limit kann nur vom Benutzer `root` geändert werden. Der Systemaufruf `setrlimit` ist für diese Parameter zuständig. Die Shell-Builtins `ulimit` (Bourne-Shells) oder `limit` (`csh`) dienen dazu, Ressourcenlimits von der Kommandozeile aus zu steuern. Auf BSD-abgeleiteten Systemen steuert die Datei `/etc/login.conf` verschiedene beim Login gesetzte Ressourcenlimits. Einzelheiten finden Sie in der Dokumentation des Betriebssystems. Die relevanten Parameter sind `maxproc`, `openfiles` und `datasize`. Zum Beispiel:

```text
default:\
...
        :datasize-cur=256M:\
        :maxproc-cur=256:\
        :openfiles-cur=256:\
...
```

`-cur` ist das weiche Limit. Hängen Sie `-max` an, um das harte Limit zu setzen.

Kernel können außerdem systemweite Grenzen für bestimmte Ressourcen haben.

- Unter Linux bestimmt der Kernel-Parameter `fs.file-max` die maximale Anzahl geöffneter Dateien, die der Kernel unterstützt. Er kann mit `sysctl -w fs.file-max=N` geändert werden. Um die Einstellung über Neustarts hinweg beizubehalten, fügen Sie eine Zuweisung in `/etc/sysctl.conf` ein. Das maximale Limit für Dateien pro Prozess wird beim Kompilieren des Kernels festgelegt; weitere Informationen finden Sie in `/usr/src/linux/Documentation/proc.txt`.

Der PostgreSQL-Server verwendet einen Prozess pro Verbindung, daher sollten Sie mindestens so viele Prozesse vorsehen wie erlaubte Verbindungen, zusätzlich zu dem, was Sie für den Rest des Systems benötigen. Das ist normalerweise kein Problem, kann aber knapp werden, wenn Sie mehrere Server auf einer Maschine betreiben.

Das werkseitige Standardlimit für offene Dateien ist oft auf „sozial verträgliche“ Werte gesetzt, die vielen Benutzern erlauben, auf einer Maschine zu koexistieren, ohne einen unangemessenen Teil der Systemressourcen zu verbrauchen. Wenn Sie viele Server auf einer Maschine betreiben, ist das vielleicht erwünscht; auf dedizierten Servern möchten Sie dieses Limit möglicherweise erhöhen.

Umgekehrt erlauben einige Systeme einzelnen Prozessen, sehr viele Dateien zu öffnen. Wenn mehr als ein paar Prozesse das tun, kann das systemweite Limit leicht überschritten werden. Wenn Sie feststellen, dass das passiert, und das systemweite Limit nicht ändern möchten, können Sie den PostgreSQL-Konfigurationsparameter `max_files_per_process` setzen, um den Verbrauch offener Dateien zu begrenzen.

Ein weiteres Kernel-Limit, das bei sehr vielen Client-Verbindungen relevant sein kann, ist die maximale Länge der Warteschlange für Socket-Verbindungen. Wenn mehr Verbindungsanfragen innerhalb sehr kurzer Zeit eintreffen, als dort hinein passen, können einige abgewiesen werden, bevor der PostgreSQL-Server sie bearbeiten kann. Diese Clients erhalten dann wenig hilfreiche Verbindungsfehler wie `Resource temporarily unavailable` oder `Connection refused`. Das Standardlimit für die Warteschlangenlänge beträgt auf vielen Plattformen 128. Um es zu erhöhen, passen Sie den passenden Kernel-Parameter per `sysctl` an und starten Sie dann den PostgreSQL-Server neu. Der Parameter heißt unter Linux `net.core.somaxconn`, unter neueren FreeBSD-Versionen `kern.ipc.soacceptqueue` und unter macOS sowie anderen BSD-Varianten `kern.ipc.somaxconn`.

### 18.4.4. Linux Memory Overcommit

Das standardmäßige Verhalten des virtuellen Speichers unter Linux ist für PostgreSQL nicht optimal. Wegen der Art, wie der Kernel Memory Overcommit implementiert, kann der Kernel den PostgreSQL-Postmaster, also den überwachenden Serverprozess, beenden, wenn der Speicherbedarf von PostgreSQL oder eines anderen Prozesses dazu führt, dass dem System virtueller Speicher ausgeht.

Wenn das passiert, sehen Sie eine Kernel-Meldung wie diese. Wo Sie eine solche Meldung finden, hängt von Ihrer Systemdokumentation und Konfiguration ab:

```text
Out of Memory: Killed process 12345 (postgres).
```

Das zeigt an, dass der Prozess `postgres` wegen Speicherdruck beendet wurde. Bestehende Datenbankverbindungen funktionieren zwar weiter normal, aber neue Verbindungen werden nicht angenommen. Zur Wiederherstellung muss PostgreSQL neu gestartet werden.

Eine Möglichkeit, dieses Problem zu vermeiden, besteht darin, PostgreSQL auf einer Maschine zu betreiben, bei der Sie sicher sein können, dass andere Prozesse den Speicher nicht erschöpfen. Wenn der Speicher knapp ist, kann das Erhöhen des Swap-Speichers des Betriebssystems helfen, das Problem zu vermeiden, weil der Out-of-Memory-Killer (OOM-Killer) nur aufgerufen wird, wenn sowohl physischer Speicher als auch Swap-Speicher erschöpft sind.

Wenn PostgreSQL selbst die Ursache für den Speichermangel ist, können Sie das Problem durch Ändern Ihrer Konfiguration vermeiden. In einigen Fällen hilft es, speicherbezogene Konfigurationsparameter zu senken, insbesondere `shared_buffers`, `work_mem` und `hash_mem_multiplier`. In anderen Fällen kann das Problem dadurch entstehen, dass zu viele Verbindungen zum Datenbankserver selbst erlaubt sind. Häufig ist es besser, `max_connections` zu reduzieren und stattdessen externe Connection-Pooling-Software zu verwenden.

Es ist möglich, das Verhalten des Kernels so zu ändern, dass er Speicher nicht „überbucht“. Diese Einstellung verhindert zwar nicht vollständig, dass der OOM-Killer aufgerufen wird, senkt aber die Wahrscheinlichkeit deutlich und führt daher zu robusterem Systemverhalten. Das geschieht, indem per `sysctl` der strikte Overcommit-Modus gewählt wird:

```text
sysctl -w vm.overcommit_memory=2
```

oder indem ein entsprechender Eintrag in `/etc/sysctl.conf` gesetzt wird. Eventuell möchten Sie auch die verwandte Einstellung `vm.overcommit_ratio` ändern. Einzelheiten finden Sie in der Kernel-Dokumentation unter `https://www.kernel.org/doc/Documentation/vm/overcommit-accounting`.

Ein anderer Ansatz, der mit oder ohne Änderung von `vm.overcommit_memory` verwendet werden kann, besteht darin, den prozessspezifischen OOM-Score-Adjustment-Wert für den Postmaster-Prozess auf `-1000` zu setzen und damit zu garantieren, dass er nicht vom OOM-Killer ausgewählt wird. Am einfachsten geschieht das durch Ausführen von:

```text
echo -1000 > /proc/self/oom_score_adj
```

im PostgreSQL-Startskript unmittelbar vor dem Aufruf von `postgres`. Beachten Sie, dass diese Aktion als `root` ausgeführt werden muss, sonst hat sie keine Wirkung. Ein Startskript im Besitz von `root` ist daher der einfachste Ort dafür. Wenn Sie das tun, sollten Sie außerdem vor dem Aufruf von `postgres` diese Umgebungsvariablen im Startskript setzen:

```text
export PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj
export PG_OOM_ADJUST_VALUE=0
```

Diese Einstellungen bewirken, dass Postmaster-Kindprozesse mit dem normalen OOM-Score-Adjustment von null laufen, sodass der OOM-Killer sie bei Bedarf weiterhin auswählen kann. Sie können einen anderen Wert für `PG_OOM_ADJUST_VALUE` verwenden, wenn die Kindprozesse mit einem anderen OOM-Score-Adjustment laufen sollen. `PG_OOM_ADJUST_VALUE` kann auch weggelassen werden; dann ist der Standard null. Wenn Sie `PG_OOM_ADJUST_FILE` nicht setzen, laufen die Kindprozesse mit demselben OOM-Score-Adjustment wie der Postmaster. Das ist unklug, weil der Sinn der Maßnahme gerade darin besteht, dem Postmaster eine bevorzugte Einstellung zu geben.

Weitere Hintergrundinformationen finden Sie unter `https://lwn.net/Articles/104179/`.

### 18.4.5. Linux Huge Pages

Die Verwendung von Huge Pages verringert den Overhead bei großen zusammenhängenden Speicherbereichen, wie PostgreSQL sie insbesondere bei großen Werten von `shared_buffers` verwendet. Um diese Funktion in PostgreSQL zu verwenden, benötigen Sie einen Kernel mit `CONFIG_HUGETLBFS=y` und `CONFIG_HUGETLB_PAGE=y`. Außerdem müssen Sie das Betriebssystem so konfigurieren, dass genügend Huge Pages der gewünschten Größe bereitgestellt werden. Der zur Laufzeit berechnete Parameter `shared_memory_size_in_huge_pages` meldet die erforderliche Anzahl von Huge Pages. Dieser Parameter kann vor dem Start des Servers mit einem `postgres`-Befehl wie diesem angezeigt werden:

```text
$ postgres -D $PGDATA -C shared_memory_size_in_huge_pages
$ grep ^Hugepagesize /proc/meminfo
Hugepagesize:       2048 kB
$ ls /sys/kernel/mm/hugepages
hugepages-1048576kB hugepages-2048kB
```

In diesem Beispiel beträgt der Standard 2 MB, aber Sie können mit `huge_page_size` auch ausdrücklich 2 MB oder 1 GB anfordern, um die von `shared_memory_size_in_huge_pages` berechnete Seitenzahl anzupassen. Obwohl in diesem Beispiel mindestens 3170 Huge Pages benötigt werden, wäre ein höherer Wert passend, wenn andere Programme auf der Maschine ebenfalls Huge Pages benötigen. Wir können das so setzen:

```text
# sysctl -w vm.nr_hugepages=3170
```

Vergessen Sie nicht, diese Einstellung in `/etc/sysctl.conf` einzutragen, damit sie nach Neustarts erneut angewendet wird. Für nicht standardmäßige Huge-Page-Größen können wir stattdessen Folgendes verwenden:

```text
# echo 3170 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

Es ist auch möglich, diese Einstellungen beim Booten mit Kernel-Parametern wie `hugepagesz=2M hugepages=3170` bereitzustellen.

Manchmal kann der Kernel wegen Fragmentierung die gewünschte Anzahl von Huge Pages nicht sofort reservieren. Dann kann es nötig sein, den Befehl zu wiederholen oder neu zu starten. Unmittelbar nach einem Neustart sollte der größte Teil des Arbeitsspeichers der Maschine verfügbar sein, um ihn in Huge Pages umzuwandeln. Um die Huge-Page-Zuweisungssituation für eine bestimmte Größe zu prüfen, verwenden Sie:

```text
$ cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

Es kann außerdem nötig sein, dem Betriebssystembenutzer des Datenbankservers die Berechtigung zur Verwendung von Huge Pages zu geben, indem `vm.hugetlb_shm_group` per `sysctl` gesetzt wird, und/oder die Berechtigung zum Sperren von Speicher mit `ulimit -l` zu erteilen.

Das Standardverhalten von PostgreSQL für Huge Pages ist, sie nach Möglichkeit mit der Standard-Huge-Page-Größe des Systems zu verwenden und bei Fehlern auf normale Seiten zurückzufallen. Um die Verwendung von Huge Pages zu erzwingen, können Sie `huge_pages` in `postgresql.conf` auf `on` setzen. Beachten Sie, dass PostgreSQL mit dieser Einstellung nicht startet, wenn nicht genügend Huge Pages verfügbar sind.

Eine ausführliche Beschreibung der Linux-Huge-Pages-Funktion finden Sie unter `https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt`.

## 18.5. Den Server herunterfahren

Es gibt mehrere Möglichkeiten, den Datenbankserver herunterzufahren. Im Kern laufen sie alle darauf hinaus, ein Signal an den überwachenden `postgres`-Prozess zu senden.

Wenn Sie eine vorgepackte Version von PostgreSQL verwenden und deren Vorkehrungen zum Starten des Servers genutzt haben, sollten Sie auch deren Vorkehrungen zum Stoppen des Servers verwenden. Einzelheiten finden Sie in der Paketdokumentation.

Wenn Sie den Server direkt verwalten, können Sie die Art des Herunterfahrens steuern, indem Sie unterschiedliche Signale an den `postgres`-Prozess senden:

- `SIGTERM`
  - Dies ist der Smart-Shutdown-Modus. Nach Empfang von `SIGTERM` lässt der Server keine neuen Verbindungen mehr zu, erlaubt bestehenden Sitzungen aber, ihre Arbeit normal zu beenden. Er fährt erst herunter, nachdem alle Sitzungen beendet sind. Wenn sich der Server in Recovery befindet, wenn ein Smart Shutdown angefordert wird, werden Recovery und Streaming-Replikation erst gestoppt, nachdem alle regulären Sitzungen beendet sind.

- `SIGINT`
  - Dies ist der Fast-Shutdown-Modus. Der Server lässt keine neuen Verbindungen mehr zu und sendet allen bestehenden Serverprozessen `SIGTERM`, wodurch diese ihre aktuellen Transaktionen abbrechen und zügig beenden. Danach wartet er auf das Ende aller Serverprozesse und fährt schließlich herunter.

- `SIGQUIT`
  - Dies ist der Immediate-Shutdown-Modus. Der Server sendet `SIGQUIT` an alle Kindprozesse und wartet darauf, dass sie sich beenden. Wenn einige davon nicht innerhalb von fünf Sekunden beendet sind, wird ihnen `SIGKILL` gesendet. Der überwachende Serverprozess beendet sich, sobald alle Kindprozesse beendet sind, ohne die normale Datenbank-Shutdown-Verarbeitung auszuführen. Das führt beim nächsten Start zu Recovery durch Wiedereinspielen des WAL-Logs. Dieser Modus wird nur für Notfälle empfohlen.

Das Programm `pg_ctl` bietet eine bequeme Schnittstelle zum Senden dieser Signale, um den Server herunterzufahren. Alternativ können Sie auf Nicht-Windows-Systemen das Signal direkt mit `kill` senden. Die PID des `postgres`-Prozesses finden Sie mit dem Programm `ps` oder in der Datei `postmaster.pid` im Datenverzeichnis. Ein Fast Shutdown sieht zum Beispiel so aus:

```text
$ kill -INT `head -1 /usr/local/pgsql/data/postmaster.pid`
```

> **Wichtig:** Es ist am besten, `SIGKILL` nicht zum Herunterfahren des Servers zu verwenden. Dadurch kann der Server Shared Memory und Semaphoren nicht freigeben. Außerdem beendet `SIGKILL` den `postgres`-Prozess, ohne dass dieser das Signal an seine Unterprozesse weitergeben kann. Daher kann es nötig werden, die einzelnen Unterprozesse ebenfalls von Hand zu beenden.

Um eine einzelne Sitzung zu beenden, während andere Sitzungen weiterlaufen, verwenden Sie `pg_terminate_backend()` (siehe Tabelle 9.96) oder senden Sie ein `SIGTERM`-Signal an den der Sitzung zugeordneten Kindprozess.

## 18.6. Einen PostgreSQL-Cluster aktualisieren

Dieser Abschnitt beschreibt, wie Sie Ihre Datenbankdaten von einer PostgreSQL-Version auf eine neuere Version aktualisieren.

Aktuelle PostgreSQL-Versionsnummern bestehen aus einer Haupt- und einer Nebenversionsnummer. In der Versionsnummer 10.1 ist zum Beispiel die 10 die Hauptversionsnummer und die 1 die Nebenversionsnummer; das wäre also die erste Nebenveröffentlichung der Hauptversion 10. Bei Veröffentlichungen vor PostgreSQL 10.0 bestehen Versionsnummern aus drei Zahlen, zum Beispiel 9.5.3. In diesen Fällen besteht die Hauptversion aus den ersten beiden Zahlengruppen der Versionsnummer, also etwa 9.5, und die Nebenversion ist die dritte Zahl, etwa 3; das wäre also die dritte Nebenveröffentlichung der Hauptversion 9.5.

Nebenveröffentlichungen ändern niemals das interne Speicherformat und sind immer mit früheren und späteren Nebenveröffentlichungen derselben Hauptversionsnummer kompatibel. Version 10.1 ist zum Beispiel mit Version 10.0 und Version 10.6 kompatibel. Ebenso ist 9.5.3 mit 9.5.0, 9.5.1 und 9.5.6 kompatibel. Um zwischen kompatiblen Versionen zu aktualisieren, ersetzen Sie einfach die ausführbaren Dateien, während der Server gestoppt ist, und starten den Server neu. Das Datenverzeichnis bleibt unverändert; Nebenversions-Upgrades sind so einfach.

Bei PostgreSQL-Hauptversionen kann sich das interne Datenspeicherformat ändern, was Upgrades komplizierter macht. Die traditionelle Methode zum Übertragen von Daten auf eine neue Hauptversion besteht darin, die Datenbank zu dumpen und wiederherzustellen, was langsam sein kann. Eine schnellere Methode ist `pg_upgrade`. Außerdem stehen Replikationsmethoden zur Verfügung, wie unten beschrieben. Wenn Sie eine vorgepackte Version von PostgreSQL verwenden, stellt sie möglicherweise Skripte zur Unterstützung von Hauptversions-Upgrades bereit. Einzelheiten finden Sie in der Paketdokumentation.

Neue Hauptversionen führen typischerweise auch einige für Benutzer sichtbare Inkompatibilitäten ein, sodass Änderungen an Anwendungsprogrammen nötig sein können. Alle sichtbaren Änderungen sind in den Versionshinweisen aufgeführt ([Anhang E](E_Versionshinweise.md)); achten Sie besonders auf den Abschnitt „Migration“. Obwohl Sie von einer Hauptversion auf eine andere aktualisieren können, ohne die dazwischenliegenden Versionen zu installieren, sollten Sie die Versionshinweise aller dazwischenliegenden Hauptversionen lesen.

Vorsichtige Benutzer werden ihre Client-Anwendungen auf der neuen Version testen wollen, bevor sie vollständig umschalten. Daher ist es oft sinnvoll, alte und neue Version parallel zu installieren. Beachten Sie beim Testen eines PostgreSQL-Hauptversions-Upgrades die folgenden Kategorien möglicher Änderungen:

- `Administration`
  - Die Funktionen, die Administratoren zum Überwachen und Steuern des Servers zur Verfügung stehen, ändern und verbessern sich oft in jeder Hauptversion.

- `SQL`
  - Dazu gehören typischerweise neue Fähigkeiten von SQL-Befehlen und keine Verhaltensänderungen, sofern diese nicht ausdrücklich in den Versionshinweisen erwähnt werden.

- `Bibliotheks-API`
  - Bibliotheken wie `libpq` fügen typischerweise nur neue Funktionalität hinzu, sofern in den Versionshinweisen nichts anderes erwähnt wird.

- `Systemkataloge`
  - Änderungen an Systemkatalogen betreffen normalerweise nur Datenbankverwaltungswerkzeuge.

- `Server-API in C`
  - Dies betrifft Änderungen an der Backend-Funktions-API, die in der Programmiersprache C geschrieben ist. Solche Änderungen wirken sich auf Code aus, der tief im Server Backend-Funktionen referenziert.

### 18.6.1. Daten mit pg_dumpall aktualisieren

Eine Upgrade-Methode besteht darin, Daten aus einer PostgreSQL-Hauptversion zu dumpen und in einer anderen wiederherzustellen. Dazu müssen Sie ein logisches Sicherungswerkzeug wie `pg_dumpall` verwenden; Sicherungsmethoden auf Dateisystemebene funktionieren nicht. Es gibt Prüfungen, die verhindern, dass Sie ein Datenverzeichnis mit einer inkompatiblen PostgreSQL-Version verwenden; großer Schaden entsteht also nicht, wenn Sie versuchen, die falsche Serverversion auf einem Datenverzeichnis zu starten.

Es wird empfohlen, die Programme `pg_dump` und `pg_dumpall` aus der neueren PostgreSQL-Version zu verwenden, um von Verbesserungen in diesen Programmen zu profitieren. Aktuelle Versionen der Dump-Programme können Daten von jeder Serverversion zurück bis 9.2 lesen.

Diese Anweisungen setzen voraus, dass Ihre bestehende Installation unter `/usr/local/pgsql` liegt und der Datenbereich in `/usr/local/pgsql/data`. Ersetzen Sie die Pfade passend für Ihr System.

1. Wenn Sie eine Sicherung erstellen, stellen Sie sicher, dass Ihre Datenbank nicht aktualisiert wird. Das beeinflusst nicht die Integrität der Sicherung, aber geänderte Daten wären natürlich nicht enthalten. Ändern Sie bei Bedarf die Berechtigungen in der Datei `/usr/local/pgsql/data/pg_hba.conf` oder einer entsprechenden Datei, um allen außer Ihnen den Zugriff zu verwehren. Siehe [Kapitel 20](20_Client_Authentifizierung.md) für weitere Informationen zur Zugriffskontrolle.

   Um Ihre Datenbankinstallation zu sichern, geben Sie ein:

   ```text
   pg_dumpall > outputfile
   ```

   Für die Sicherung können Sie den `pg_dumpall`-Befehl der Version verwenden, die Sie gerade ausführen; siehe Abschnitt 25.1.2 für weitere Einzelheiten. Für beste Ergebnisse sollten Sie jedoch versuchen, den `pg_dumpall`-Befehl aus PostgreSQL 18.1 zu verwenden, da diese Version Fehlerkorrekturen und Verbesserungen gegenüber älteren Versionen enthält. Dieser Rat kann eigenwillig wirken, weil Sie die neue Version noch nicht installiert haben, ist aber sinnvoll, wenn Sie die neue Version parallel zur alten installieren wollen. In diesem Fall können Sie die Installation normal abschließen und die Daten später übertragen. Das verringert außerdem die Ausfallzeit.

2. Fahren Sie den alten Server herunter:

   ```text
   pg_ctl stop
   ```

   Auf Systemen, die PostgreSQL beim Booten starten, gibt es wahrscheinlich eine Startdatei, die dasselbe erledigt. Auf einem Red-Hat-Linux-System könnte zum Beispiel Folgendes funktionieren:

   ```text
   /etc/rc.d/init.d/postgresql stop
   ```

   Siehe [Kapitel 18](18_Servereinrichtung_und_Betrieb.md) für Einzelheiten zum Starten und Stoppen des Servers.

3. Wenn Sie aus einer Sicherung wiederherstellen, benennen Sie das alte Installationsverzeichnis um oder löschen Sie es, falls es nicht versionsspezifisch ist. Es ist eine gute Idee, das Verzeichnis umzubenennen statt es zu löschen, falls Probleme auftreten und Sie zurückkehren müssen. Denken Sie daran, dass das Verzeichnis erheblichen Plattenplatz verbrauchen kann. Zum Umbenennen verwenden Sie einen Befehl wie:

   ```text
   mv /usr/local/pgsql /usr/local/pgsql.old
   ```

   Stellen Sie sicher, dass Sie das Verzeichnis als Einheit verschieben, damit relative Pfade unverändert bleiben.

4. Installieren Sie die neue Version von PostgreSQL wie in [Kapitel 17](17_Installation_aus_dem_Quellcode.md) beschrieben.

5. Erstellen Sie bei Bedarf einen neuen Datenbankcluster. Denken Sie daran, dass Sie diese Befehle ausführen müssen, während Sie mit dem speziellen Datenbankbenutzerkonto angemeldet sind, das Sie beim Upgrade bereits haben.

   ```text
   /usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
   ```

6. Stellen Sie Ihre vorherige `pg_hba.conf` und alle Änderungen an `postgresql.conf` wieder her.

7. Starten Sie den Datenbankserver, erneut mit dem speziellen Datenbankbenutzerkonto:

   ```text
   /usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data
   ```

8. Stellen Sie schließlich Ihre Daten aus der Sicherung mit dem neuen `psql` wieder her:

   ```text
   /usr/local/pgsql/bin/psql -d postgres -f outputfile
   ```

Die geringste Ausfallzeit erreichen Sie, indem Sie den neuen Server in einem anderen Verzeichnis installieren und alten und neuen Server parallel auf verschiedenen Ports laufen lassen. Dann können Sie zum Übertragen Ihrer Daten etwa Folgendes verwenden:

```text
pg_dumpall -p 5432 | psql -d postgres -p 5433
```

### 18.6.2. Daten mit pg_upgrade aktualisieren

Das Modul `pg_upgrade` erlaubt es, eine Installation direkt von einer PostgreSQL-Hauptversion auf eine andere zu migrieren. Upgrades können in Minuten durchgeführt werden, insbesondere im Modus `--link`. Es erfordert ähnliche Schritte wie `pg_dumpall`, etwa Starten und Stoppen des Servers sowie Ausführen von `initdb`. Die Dokumentation zu `pg_upgrade` beschreibt die nötigen Schritte.

### 18.6.3. Daten per Replikation aktualisieren

Es ist auch möglich, mit logischen Replikationsmethoden einen Standby-Server mit der aktualisierten PostgreSQL-Version zu erstellen. Das ist möglich, weil logische Replikation die Replikation zwischen unterschiedlichen PostgreSQL-Hauptversionen unterstützt. Der Standby kann auf demselben Rechner oder auf einem anderen Rechner laufen. Sobald er mit dem Primärserver synchronisiert ist, der die ältere PostgreSQL-Version ausführt, können Sie die Rollen wechseln, den Standby zum Primärserver machen und die ältere Datenbankinstanz herunterfahren. Ein solcher Wechsel führt bei einem Upgrade nur zu einigen Sekunden Ausfallzeit.

Diese Upgrade-Methode kann mit den eingebauten logischen Replikationsfunktionen ausgeführt werden, aber auch mit externen logischen Replikationssystemen wie `pglogical`, Slony, Londiste und Bucardo.

## 18.7. Server-Spoofing verhindern

Während der Server läuft, kann ein böswilliger Benutzer nicht an die Stelle des normalen Datenbankservers treten. Wenn der Server jedoch gestoppt ist, kann ein lokaler Benutzer den normalen Server vortäuschen, indem er einen eigenen Server startet. Der vorgetäuschte Server könnte Passwörter und Abfragen lesen, die Clients senden, aber keine Daten zurückgeben, weil das Verzeichnis `PGDATA` durch Verzeichnisrechte weiterhin geschützt wäre. Spoofing ist möglich, weil jeder Benutzer einen Datenbankserver starten kann; ein Client kann einen ungültigen Server nur erkennen, wenn er speziell dafür konfiguriert ist.

Eine Möglichkeit, Spoofing lokaler Verbindungen zu verhindern, besteht darin, ein Unix-Domain-Socket-Verzeichnis (`unix_socket_directories`) zu verwenden, in dem nur ein vertrauenswürdiger lokaler Benutzer Schreibrechte hat. Dadurch wird verhindert, dass ein böswilliger Benutzer in diesem Verzeichnis eine eigene Socket-Datei erstellt. Wenn Sie befürchten, dass einige Anwendungen weiterhin `/tmp` für die Socket-Datei referenzieren und dadurch für Spoofing anfällig sind, erstellen Sie beim Start des Betriebssystems einen symbolischen Link `/tmp/.s.PGSQL.5432`, der auf die verlegte Socket-Datei zeigt. Möglicherweise müssen Sie auch Ihr Skript zum Aufräumen von `/tmp` ändern, damit der symbolische Link nicht entfernt wird.

Eine weitere Option für lokale Verbindungen ist, dass Clients mit `requirepeer` den erforderlichen Eigentümer des Serverprozesses angeben, der mit dem Socket verbunden ist.

Um Spoofing bei TCP-Verbindungen zu verhindern, verwenden Sie entweder SSL-Zertifikate und stellen sicher, dass Clients das Zertifikat des Servers prüfen, oder verwenden Sie GSSAPI-Verschlüsselung, oder beides, wenn es sich um getrennte Verbindungen handelt.

Um Spoofing mit SSL zu verhindern, muss der Server so konfiguriert sein, dass er nur `hostssl`-Verbindungen akzeptiert ([Abschnitt 20.1](20_Client_Authentifizierung.md#201-die-datei-pghbaconf)), und SSL-Schlüssel- und Zertifikatsdateien besitzen ([Abschnitt 18.9](#189-sichere-tcpipverbindungen-mit-ssl)). Der TCP-Client muss mit `sslmode=verify-ca` oder `verify-full` verbinden und die passende Root-Zertifikatsdatei installiert haben (Abschnitt 32.19.1). Alternativ kann der System-CA-Pool, wie von der SSL-Implementierung definiert, mit `sslrootcert=system` verwendet werden. In diesem Fall wird aus Sicherheitsgründen `sslmode=verify-full` erzwungen, da es im Allgemeinen trivial ist, Zertifikate zu erhalten, die von einer öffentlichen CA signiert sind.

Um Server-Spoofing bei Passwortauthentifizierung mit `scram-sha-256` über ein Netzwerk zu verhindern, sollten Sie sicherstellen, dass Sie mit SSL und mit einer der im vorherigen Absatz beschriebenen Anti-Spoofing-Methoden zum Server verbinden. Zusätzlich kann die SCRAM-Implementierung in `libpq` nicht den gesamten Authentifizierungsaustausch schützen, aber der Verbindungsparameter `channel_binding=require` bietet eine Abschwächung gegen Server-Spoofing. Ein Angreifer, der einen betrügerischen Server nutzt, um einen SCRAM-Austausch abzufangen, kann durch Offline-Analyse möglicherweise das gehashte Passwort des Clients ermitteln.

Um Spoofing mit GSSAPI zu verhindern, muss der Server so konfiguriert sein, dass er nur `hostgssenc`-Verbindungen akzeptiert ([Abschnitt 20.1](20_Client_Authentifizierung.md#201-die-datei-pghbaconf)) und dafür `gss`-Authentifizierung verwendet. Der TCP-Client muss mit `gssencmode=require` verbinden.

## 18.8. Verschlüsselungsoptionen

PostgreSQL bietet Verschlüsselung auf mehreren Ebenen und ist flexibel darin, Daten vor Offenlegung durch Diebstahl des Datenbankservers, unredliche Administratoren und unsichere Netzwerke zu schützen. Verschlüsselung kann auch erforderlich sein, um sensible Daten wie medizinische Aufzeichnungen oder Finanztransaktionen abzusichern.

**Passwortverschlüsselung**

Datenbankbenutzerpasswörter werden als Hashes gespeichert, bestimmt durch die Einstellung `password_encryption`, sodass der Administrator das dem Benutzer zugewiesene tatsächliche Passwort nicht ermitteln kann. Wenn für die Client-Authentifizierung SCRAM- oder MD5-Verschlüsselung verwendet wird, liegt das unverschlüsselte Passwort nicht einmal vorübergehend auf dem Server vor, weil der Client es vor dem Versand über das Netzwerk verschlüsselt. SCRAM wird bevorzugt, weil es ein Internetstandard ist und sicherer als das PostgreSQL-spezifische MD5-Authentifizierungsprotokoll.

> **Warnung:** Die Unterstützung für MD5-verschlüsselte Passwörter ist veraltet und wird in einer zukünftigen PostgreSQL-Version entfernt. Siehe [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung) für Einzelheiten zur Migration auf einen anderen Passworttyp.

**Verschlüsselung bestimmter Spalten**

Das Modul `pgcrypto` erlaubt es, bestimmte Felder verschlüsselt zu speichern. Das ist nützlich, wenn nur ein Teil der Daten sensibel ist. Der Client liefert den Entschlüsselungsschlüssel, und die Daten werden auf dem Server entschlüsselt und anschließend an den Client gesendet.

Die entschlüsselten Daten und der Entschlüsselungsschlüssel sind für kurze Zeit auf dem Server vorhanden, während entschlüsselt und zwischen Client und Server kommuniziert wird. Dadurch entsteht ein kurzer Moment, in dem Daten und Schlüssel von jemandem abgefangen werden können, der vollständigen Zugriff auf den Datenbankserver hat, zum Beispiel dem Systemadministrator.

**Datenpartition-Verschlüsselung**

Speicherverschlüsselung kann auf Dateisystemebene oder Blockebene erfolgen. Linux-Dateisystemverschlüsselungsoptionen umfassen eCryptfs und EncFS, während FreeBSD PEFS verwendet. Optionen für Block- oder vollständige Festplattenverschlüsselung sind unter Linux `dm-crypt` + LUKS und unter FreeBSD die GEOM-Module `geli` und `gbde`. Viele andere Betriebssysteme unterstützen diese Funktion ebenfalls, darunter Windows.

Dieser Mechanismus verhindert, dass unverschlüsselte Daten von Laufwerken gelesen werden, wenn die Laufwerke oder der ganze Rechner gestohlen werden. Er schützt nicht vor Angriffen, während das Dateisystem eingehängt ist, denn im eingehängten Zustand stellt das Betriebssystem eine unverschlüsselte Sicht auf die Daten bereit. Zum Einhängen des Dateisystems müssen Sie den Verschlüsselungsschlüssel jedoch irgendwie an das Betriebssystem übergeben, und manchmal wird der Schlüssel irgendwo auf dem Host gespeichert, der die Platte einhängt.

**Daten über ein Netzwerk verschlüsseln**

SSL-Verbindungen verschlüsseln alle über das Netzwerk gesendeten Daten: das Passwort, die Abfragen und die zurückgegebenen Daten. Die Datei `pg_hba.conf` erlaubt Administratoren festzulegen, welche Hosts unverschlüsselte Verbindungen (`host`) verwenden dürfen und welche SSL-verschlüsselte Verbindungen (`hostssl`) erfordern. Außerdem können Clients angeben, dass sie nur über SSL zu Servern verbinden.

GSSAPI-verschlüsselte Verbindungen verschlüsseln alle über das Netzwerk gesendeten Daten, einschließlich Abfragen und zurückgegebener Daten. Es wird kein Passwort über das Netzwerk gesendet. Die Datei `pg_hba.conf` erlaubt Administratoren festzulegen, welche Hosts unverschlüsselte Verbindungen (`host`) verwenden dürfen und welche GSSAPI-verschlüsselte Verbindungen (`hostgssenc`) erfordern. Außerdem können Clients angeben, dass sie nur über GSSAPI-verschlüsselte Verbindungen zu Servern verbinden (`gssencmode=require`).

Auch Stunnel oder SSH können verwendet werden, um Übertragungen zu verschlüsseln.

**SSL-Host-Authentifizierung**

Es ist möglich, dass Client und Server einander SSL-Zertifikate vorlegen. Das erfordert auf beiden Seiten zusätzliche Konfiguration, bietet aber eine stärkere Identitätsprüfung als die bloße Verwendung von Passwörtern. Es verhindert, dass ein Rechner lange genug vorgibt, der Server zu sein, um das vom Client gesendete Passwort zu lesen. Es hilft außerdem gegen „Man-in-the-Middle“-Angriffe, bei denen ein Rechner zwischen Client und Server vorgibt, der Server zu sein, und alle Daten zwischen Client und Server liest und weiterleitet.

**Client-seitige Verschlüsselung**

Wenn dem Systemadministrator der Servermaschine nicht vertraut werden kann, muss der Client die Daten verschlüsseln. So erscheinen unverschlüsselte Daten niemals auf dem Datenbankserver. Daten werden auf dem Client verschlüsselt, bevor sie an den Server gesendet werden, und Datenbankergebnisse müssen auf dem Client entschlüsselt werden, bevor sie verwendet werden.

## 18.9. Sichere TCP/IP-Verbindungen mit SSL

PostgreSQL unterstützt nativ SSL-Verbindungen, um die Client/Server-Kommunikation für erhöhte Sicherheit zu verschlüsseln. Dazu muss OpenSSL sowohl auf Client- als auch auf Serversystemen installiert sein, und PostgreSQL muss zur Build-Zeit mit dieser Unterstützung gebaut worden sein (siehe [Kapitel 17](17_Installation_aus_dem_Quellcode.md)).

Die Begriffe SSL und TLS werden häufig austauschbar für eine sichere verschlüsselte Verbindung mit einem TLS-Protokoll verwendet. SSL-Protokolle sind die Vorläufer der TLS-Protokolle, und der Begriff SSL wird weiterhin für verschlüsselte Verbindungen verwendet, obwohl SSL-Protokolle nicht mehr unterstützt werden. In PostgreSQL wird SSL austauschbar mit TLS verwendet.

### 18.9.1. Grundeinrichtung

Wenn SSL-Unterstützung einkompiliert ist, kann der PostgreSQL-Server mit Unterstützung für verschlüsselte Verbindungen über TLS-Protokolle gestartet werden, indem der Parameter `ssl` in `postgresql.conf` auf `on` gesetzt wird. Der Server lauscht auf demselben TCP-Port sowohl auf normale als auch auf SSL-Verbindungen und verhandelt mit jedem verbindenden Client, ob SSL verwendet wird. Standardmäßig liegt diese Entscheidung beim Client; siehe [Abschnitt 20.1](20_Client_Authentifizierung.md#201-die-datei-pghbaconf), wie Sie den Server so einrichten, dass SSL für einige oder alle Verbindungen erforderlich ist.

Zum Starten im SSL-Modus müssen Dateien mit dem Serverzertifikat und dem privaten Schlüssel existieren. Standardmäßig werden diese Dateien als `server.crt` beziehungsweise `server.key` im Datenverzeichnis des Servers erwartet. Andere Namen und Orte können über die Konfigurationsparameter `ssl_cert_file` und `ssl_key_file` angegeben werden.

Auf Unix-Systemen müssen die Rechte an `server.key` jeden Zugriff für Welt oder Gruppe ausschließen. Das erreichen Sie mit:

```text
chmod 0600 server.key
```

Alternativ kann die Datei `root` gehören und Gruppen-Lesezugriff haben, also Rechte `0640`. Diese Einrichtung ist für Installationen gedacht, bei denen Zertifikats- und Schlüsseldateien vom Betriebssystem verwaltet werden. Der Benutzer, unter dem der PostgreSQL-Server läuft, sollte dann Mitglied der Gruppe sein, die Zugriff auf diese Zertifikats- und Schlüsseldateien hat.

Wenn das Datenverzeichnis Gruppen-Lesezugriff erlaubt, müssen Zertifikatsdateien eventuell außerhalb des Datenverzeichnisses liegen, um die oben genannten Sicherheitsanforderungen einzuhalten. Gruppenzugriff wird im Allgemeinen aktiviert, damit ein nicht privilegierter Benutzer die Datenbank sichern kann; in diesem Fall kann die Sicherungssoftware die Zertifikatsdateien nicht lesen und wird wahrscheinlich einen Fehler melden.

Wenn der private Schlüssel mit einer Passphrase geschützt ist, fragt der Server nach der Passphrase und startet erst, nachdem sie eingegeben wurde. Die Verwendung einer Passphrase verhindert standardmäßig, dass die SSL-Konfiguration des Servers ohne Serverneustart geändert werden kann; siehe aber `ssl_passphrase_command_supports_reload`. Außerdem können private Schlüssel mit Passphrase unter Windows überhaupt nicht verwendet werden.

Das erste Zertifikat in `server.crt` muss das Serverzertifikat sein, weil es zum privaten Schlüssel des Servers passen muss. Die Zertifikate „intermediärer“ Zertifizierungsstellen können ebenfalls an die Datei angehängt werden. Dadurch müssen Zwischenzertifikate nicht auf Clients gespeichert werden, vorausgesetzt Root- und Zwischenzertifikate wurden mit `v3_ca`-Erweiterungen erstellt. Dadurch wird die Basiseinschränkung `CA` des Zertifikats auf `true` gesetzt. Dies erleichtert das Ablaufenlassen von Zwischenzertifikaten.

Es ist nicht nötig, das Root-Zertifikat zu `server.crt` hinzuzufügen. Stattdessen müssen Clients das Root-Zertifikat der Zertifikatskette des Servers besitzen.

### 18.9.2. OpenSSL-Konfiguration

PostgreSQL liest die systemweite OpenSSL-Konfigurationsdatei. Standardmäßig heißt diese Datei `openssl.cnf` und liegt in dem Verzeichnis, das `openssl version -d` meldet. Dieser Standard kann überschrieben werden, indem die Umgebungsvariable `OPENSSL_CONF` auf den Namen der gewünschten Konfigurationsdatei gesetzt wird.

OpenSSL unterstützt eine große Bandbreite von Chiffren und Authentifizierungsalgorithmen unterschiedlicher Stärke. Obwohl eine Liste von Chiffren in der OpenSSL-Konfigurationsdatei angegeben werden kann, können Sie Chiffren speziell für die Verwendung durch den Datenbankserver festlegen, indem Sie `ssl_ciphers` in `postgresql.conf` ändern.

> **Hinweis:** Es ist möglich, Authentifizierung ohne Verschlüsselungs-Overhead zu haben, indem Chiffren wie `NULL-SHA` oder `NULL-MD5` verwendet werden. Ein Man-in-the-Middle könnte dann jedoch die Kommunikation zwischen Client und Server lesen und weiterleiten. Außerdem ist der Verschlüsselungs-Overhead im Vergleich zum Authentifizierungs-Overhead gering. Aus diesen Gründen werden NULL-Chiffren nicht empfohlen.

### 18.9.3. Client-Zertifikate verwenden

Um zu verlangen, dass der Client ein vertrauenswürdiges Zertifikat liefert, legen Sie die Zertifikate der Root-Zertifizierungsstellen (CAs), denen Sie vertrauen, in einer Datei im Datenverzeichnis ab, setzen den Parameter `ssl_ca_file` in `postgresql.conf` auf den neuen Dateinamen und fügen die Authentifizierungsoption `clientcert=verify-ca` oder `clientcert=verify-full` zu den passenden `hostssl`-Zeilen in `pg_hba.conf` hinzu. Beim Aufbau der SSL-Verbindung wird dann ein Zertifikat vom Client angefordert. Siehe [Abschnitt 32.19](32_libpq_C_Bibliothek.md#3219-sslunterstützung) für eine Beschreibung, wie Zertifikate auf dem Client eingerichtet werden.

Bei einem `hostssl`-Eintrag mit `clientcert=verify-ca` prüft der Server, ob das Client-Zertifikat von einer der vertrauenswürdigen Zertifizierungsstellen signiert wurde. Wenn `clientcert=verify-full` angegeben ist, prüft der Server nicht nur die Zertifikatskette, sondern auch, ob der Benutzername oder dessen Mapping dem `cn` (Common Name) des vorgelegten Zertifikats entspricht. Beachten Sie, dass die Validierung der Zertifikatskette immer sichergestellt ist, wenn die Authentifizierungsmethode `cert` verwendet wird (siehe [Abschnitt 20.12](20_Client_Authentifizierung.md#2012-zertifikatsauthentifizierung)).

Zwischenzertifikate, die zu bestehenden Root-Zertifikaten führen, können ebenfalls in der Datei `ssl_ca_file` erscheinen, wenn Sie vermeiden möchten, sie auf Clients zu speichern, vorausgesetzt Root- und Zwischenzertifikate wurden mit `v3_ca`-Erweiterungen erstellt. Einträge in Certificate Revocation Lists (CRL) werden ebenfalls geprüft, wenn der Parameter `ssl_crl_file` oder `ssl_crl_dir` gesetzt ist.

Die Authentifizierungsoption `clientcert` ist für alle Authentifizierungsmethoden verfügbar, aber nur in `pg_hba.conf`-Zeilen, die als `hostssl` angegeben sind. Wenn `clientcert` nicht angegeben ist, prüft der Server das Client-Zertifikat nur dann gegen seine CA-Datei, wenn ein Client-Zertifikat vorgelegt wird und die CA konfiguriert ist.

Es gibt zwei Ansätze, um zu erzwingen, dass Benutzer beim Login ein Zertifikat vorlegen.

Der erste Ansatz verwendet die Authentifizierungsmethode `cert` für `hostssl`-Einträge in `pg_hba.conf`, sodass das Zertifikat selbst zur Authentifizierung verwendet wird und zugleich SSL-Verbindungssicherheit bietet. Siehe [Abschnitt 20.12](20_Client_Authentifizierung.md#2012-zertifikatsauthentifizierung) für Einzelheiten. Es ist nicht nötig, bei Verwendung der Authentifizierungsmethode `cert` explizit `clientcert`-Optionen anzugeben. In diesem Fall wird der im Zertifikat angegebene `cn` (Common Name) gegen den Benutzernamen oder ein passendes Mapping geprüft.

Der zweite Ansatz kombiniert eine beliebige Authentifizierungsmethode für `hostssl`-Einträge mit der Überprüfung von Client-Zertifikaten, indem die Authentifizierungsoption `clientcert` auf `verify-ca` oder `verify-full` gesetzt wird. Die erste Option erzwingt nur, dass das Zertifikat gültig ist, während die zweite zusätzlich sicherstellt, dass der `cn` (Common Name) im Zertifikat dem Benutzernamen oder einem passenden Mapping entspricht.

### 18.9.4. Verwendung von SSL-Serverdateien

Tabelle 18.2 fasst die Dateien zusammen, die für die SSL-Einrichtung auf dem Server relevant sind. Die gezeigten Dateinamen sind Standardnamen; lokal konfigurierte Namen können abweichen.

**Tabelle 18.2. Verwendung von SSL-Serverdateien**

| Datei | Inhalt | Wirkung |
|---|---|---|
| `ssl_cert_file` (`$PGDATA/server.crt`) | Serverzertifikat | wird an den Client gesendet, um die Identität des Servers anzugeben |
| `ssl_key_file` (`$PGDATA/server.key`) | privater Serverschlüssel | beweist, dass das Serverzertifikat vom Eigentümer gesendet wurde; sagt nicht aus, ob der Zertifikatseigentümer vertrauenswürdig ist |
| `ssl_ca_file` | vertrauenswürdige Zertifizierungsstellen | prüft, ob das Client-Zertifikat von einer vertrauenswürdigen Zertifizierungsstelle signiert ist |
| `ssl_crl_file` | von Zertifizierungsstellen widerrufene Zertifikate | das Client-Zertifikat darf nicht auf dieser Liste stehen |

Der Server liest diese Dateien beim Serverstart und immer dann, wenn die Serverkonfiguration neu geladen wird. Auf Windows-Systemen werden sie außerdem erneut gelesen, wenn für eine neue Client-Verbindung ein neuer Backend-Prozess gestartet wird.

Wenn beim Serverstart ein Fehler in diesen Dateien erkannt wird, verweigert der Server den Start. Wird ein Fehler dagegen beim Neuladen der Konfiguration erkannt, werden die Dateien ignoriert und die alte SSL-Konfiguration wird weiterverwendet. Auf Windows-Systemen kann ein Backend keine SSL-Verbindung herstellen, wenn beim Backend-Start ein Fehler in diesen Dateien erkannt wird. In all diesen Fällen wird der Fehlerzustand im Serverlog gemeldet.

### 18.9.5. Zertifikate erstellen

Um ein einfaches selbstsigniertes Zertifikat für den Server zu erstellen, das 365 Tage gültig ist, verwenden Sie den folgenden OpenSSL-Befehl und ersetzen `dbhost.yourdomain.com` durch den Hostnamen des Servers:

```text
openssl req -new -x509 -days 365 -nodes -text -out server.crt \
  -keyout server.key -subj "/CN=dbhost.yourdomain.com"
```

Danach:

```text
chmod og-rwx server.key
```

denn der Server weist die Datei zurück, wenn ihre Rechte großzügiger sind. Weitere Einzelheiten zum Erstellen Ihres privaten Serverschlüssels und Zertifikats finden Sie in der OpenSSL-Dokumentation.

Während ein selbstsigniertes Zertifikat zum Testen verwendet werden kann, sollte in Produktion ein von einer Zertifizierungsstelle (CA), üblicherweise einer unternehmensweiten Root-CA, signiertes Zertifikat verwendet werden.

Um ein Serverzertifikat zu erstellen, dessen Identität von Clients validiert werden kann, erstellen Sie zuerst eine Certificate Signing Request (CSR) und eine Datei mit öffentlichem und privatem Schlüssel:

```text
openssl req -new -nodes -text -out root.csr \
  -keyout root.key -subj "/CN=root.yourdomain.com"
chmod og-rwx root.key
```

Signieren Sie dann die Anforderung mit dem Schlüssel, um eine Root-Zertifizierungsstelle zu erstellen. Dieses Beispiel verwendet den Standardort der OpenSSL-Konfigurationsdatei unter Linux:

```text
openssl x509 -req -in root.csr -text -days 3650 \
  -extfile /etc/ssl/openssl.cnf -extensions v3_ca \
  -signkey root.key -out root.crt
```

Erstellen Sie schließlich ein Serverzertifikat, das von der neuen Root-Zertifizierungsstelle signiert wird:

```text
openssl req -new -nodes -text -out server.csr \
  -keyout server.key -subj "/CN=dbhost.yourdomain.com"
chmod og-rwx server.key

openssl x509 -req -in server.csr -text -days 365 \
  -CA root.crt -CAkey root.key -CAcreateserial \
  -out server.crt
```

`server.crt` und `server.key` sollten auf dem Server gespeichert werden, und `root.crt` sollte auf dem Client gespeichert werden, damit der Client prüfen kann, dass das Leaf-Zertifikat des Servers von seinem vertrauenswürdigen Root-Zertifikat signiert wurde. `root.key` sollte offline gespeichert werden, um zukünftige Zertifikate zu erstellen.

Es ist auch möglich, eine Vertrauenskette zu erstellen, die Zwischenzertifikate umfasst:

```text
# root
openssl req -new -nodes -text -out root.csr \
  -keyout root.key -subj "/CN=root.yourdomain.com"
chmod og-rwx root.key
openssl x509 -req -in root.csr -text -days 3650 \
  -extfile /etc/ssl/openssl.cnf -extensions v3_ca \
  -signkey root.key -out root.crt

# intermediate
openssl req -new -nodes -text -out intermediate.csr \
  -keyout intermediate.key -subj "/CN=intermediate.yourdomain.com"
chmod og-rwx intermediate.key
openssl x509 -req -in intermediate.csr -text -days 1825 \
  -extfile /etc/ssl/openssl.cnf -extensions v3_ca \
  -CA root.crt -CAkey root.key -CAcreateserial \
  -out intermediate.crt

# leaf
openssl req -new -nodes -text -out server.csr \
  -keyout server.key -subj "/CN=dbhost.yourdomain.com"
chmod og-rwx server.key
openssl x509 -req -in server.csr -text -days 365 \
  -CA intermediate.crt -CAkey intermediate.key -CAcreateserial \
  -out server.crt
```

`server.crt` und `intermediate.crt` sollten zu einem Zertifikatsdatei-Bundle zusammengefügt und auf dem Server gespeichert werden. `server.key` sollte ebenfalls auf dem Server gespeichert werden. `root.crt` sollte auf dem Client gespeichert werden, damit der Client prüfen kann, dass das Leaf-Zertifikat des Servers von einer Zertifikatskette signiert wurde, die mit seinem vertrauenswürdigen Root-Zertifikat verbunden ist. `root.key` und `intermediate.key` sollten offline gespeichert werden, um zukünftige Zertifikate zu erstellen.

## 18.10. Sichere TCP/IP-Verbindungen mit GSSAPI-Verschlüsselung

PostgreSQL unterstützt außerdem nativ GSSAPI, um die Client/Server-Kommunikation für erhöhte Sicherheit zu verschlüsseln. Die Unterstützung setzt voraus, dass eine GSSAPI-Implementierung wie MIT Kerberos sowohl auf Client- als auch auf Serversystemen installiert ist und dass PostgreSQL zur Build-Zeit mit dieser Unterstützung gebaut wurde (siehe [Kapitel 17](17_Installation_aus_dem_Quellcode.md)).

### 18.10.1. Grundeinrichtung

Der PostgreSQL-Server lauscht auf demselben TCP-Port sowohl auf normale als auch auf GSSAPI-verschlüsselte Verbindungen und verhandelt mit jedem verbindenden Client, ob GSSAPI für Verschlüsselung und Authentifizierung verwendet werden soll. Standardmäßig liegt diese Entscheidung beim Client, was bedeutet, dass sie von einem Angreifer herabgestuft werden kann. Siehe [Abschnitt 20.1](20_Client_Authentifizierung.md#201-die-datei-pghbaconf), wie Sie den Server so einrichten, dass GSSAPI für einige oder alle Verbindungen erforderlich ist.

Wenn GSSAPI für Verschlüsselung verwendet wird, ist es üblich, GSSAPI auch für Authentifizierung zu verwenden, da der zugrunde liegende Mechanismus in jedem Fall sowohl Client- als auch Serveridentitäten gemäß der GSSAPI-Implementierung bestimmt. Das ist aber nicht erforderlich; eine andere PostgreSQL-Authentifizierungsmethode kann zur zusätzlichen Prüfung gewählt werden.

Abgesehen von der Konfiguration des Verhandlungsverhaltens erfordert GSSAPI-Verschlüsselung keine Einrichtung über das hinaus, was für GSSAPI-Authentifizierung nötig ist. Weitere Informationen zur Konfiguration finden Sie in [Abschnitt 20.6](20_Client_Authentifizierung.md#206-gssapiauthentifizierung).

## 18.11. Sichere TCP/IP-Verbindungen mit SSH-Tunneln

Es ist möglich, SSH zu verwenden, um die Netzwerkverbindung zwischen Clients und einem PostgreSQL-Server zu verschlüsseln. Richtig eingerichtet bietet das eine ausreichend sichere Netzwerkverbindung, auch für Clients, die kein SSL unterstützen.

Stellen Sie zuerst sicher, dass auf derselben Maschine wie der PostgreSQL-Server ein SSH-Server korrekt läuft und dass Sie sich mit `ssh` als irgendein Benutzer anmelden können. Dann können Sie einen sicheren Tunnel zum entfernten Server aufbauen. Ein sicherer Tunnel lauscht auf einem lokalen Port und leitet den gesamten Verkehr an einen Port auf der entfernten Maschine weiter.

Verkehr, der an den entfernten Port gesendet wird, kann an dessen `localhost`-Adresse oder, falls gewünscht, an einer anderen Bind-Adresse ankommen. Er erscheint nicht so, als käme er von Ihrer lokalen Maschine. Dieser Befehl erstellt einen sicheren Tunnel von der Client-Maschine zur entfernten Maschine `foo.com`:

```text
ssh -L 63333:localhost:5432 joe@foo.com
```

Die erste Zahl im Argument `-L`, `63333`, ist die lokale Portnummer des Tunnels; sie kann jeder unbenutzte Port sein. IANA reserviert die Ports 49152 bis 65535 für private Nutzung. Der Name oder die IP-Adresse danach ist die entfernte Bind-Adresse, zu der Sie verbinden, also `localhost`, was der Standard ist. Die zweite Zahl, `5432`, ist das entfernte Ende des Tunnels, also zum Beispiel die Portnummer, die Ihr Datenbankserver verwendet. Um über diesen Tunnel zum Datenbankserver zu verbinden, verbinden Sie auf der lokalen Maschine mit Port `63333`:

```text
psql -h localhost -p 63333 postgres
```

Für den Datenbankserver sieht es dann so aus, als ob Sie als Benutzer `joe` auf Host `foo.com` zur Bind-Adresse `localhost` verbinden. Er verwendet das Authentifizierungsverfahren, das für Verbindungen dieses Benutzers zu dieser Bind-Adresse konfiguriert wurde. Beachten Sie, dass der Server die Verbindung nicht als SSL-verschlüsselt ansieht, da sie zwischen SSH-Server und PostgreSQL-Server tatsächlich nicht verschlüsselt ist. Das sollte kein zusätzliches Sicherheitsrisiko darstellen, weil beide auf derselben Maschine laufen.

Damit der Tunnelaufbau gelingt, müssen Sie sich per `ssh` als `joe@foo.com` verbinden dürfen, genau wie bei einer SSH-Terminalsitzung.

Sie hätten Portweiterleitung auch so einrichten können:

```text
ssh -L 63333:foo.com:5432 joe@foo.com
```

Dann sieht der Datenbankserver die Verbindung jedoch so, als käme sie auf seiner Bind-Adresse `foo.com` an, die durch die Standardeinstellung `listen_addresses = 'localhost'` nicht geöffnet ist. Das ist normalerweise nicht das, was Sie wollen.

Wenn Sie über einen Login-Host zum Datenbankserver „springen“ müssen, könnte eine mögliche Einrichtung so aussehen:

```text
ssh -L 63333:db.foo.com:5432 joe@shell.foo.com
```

Beachten Sie, dass die Verbindung von `shell.foo.com` zu `db.foo.com` auf diese Weise nicht durch den SSH-Tunnel verschlüsselt wird. SSH bietet recht viele Konfigurationsmöglichkeiten, wenn das Netzwerk auf verschiedene Weise eingeschränkt ist. Einzelheiten finden Sie in der SSH-Dokumentation.

> **Tipp:** Es gibt mehrere andere Anwendungen, die sichere Tunnel mit einem konzeptionell ähnlichen Verfahren bereitstellen können.

## 18.12. Event Log unter Windows registrieren

Um eine Windows-Event-Log-Bibliothek beim Betriebssystem zu registrieren, geben Sie diesen Befehl ein:

```text
regsvr32 pgsql_library_directory/pgevent.dll
```

Dadurch werden Registry-Einträge erstellt, die vom Event Viewer verwendet werden, unter der Standard-Ereignisquelle `PostgreSQL`.

Um einen anderen Namen für die Ereignisquelle anzugeben (siehe `event_source`), verwenden Sie die Optionen `/n` und `/i`:

```text
regsvr32 /n /i:event_source_name pgsql_library_directory/pgevent.dll
```

Um die Event-Log-Bibliothek beim Betriebssystem abzumelden, geben Sie diesen Befehl ein:

```text
regsvr32 /u [/i:event_source_name] pgsql_library_directory/pgevent.dll
```

> **Hinweis:** Um Event Logging im Datenbankserver zu aktivieren, ändern Sie `log_destination` in `postgresql.conf` so, dass `eventlog` enthalten ist.
