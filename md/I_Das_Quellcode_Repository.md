# Anhang I. Das Quellcode-Repository

Der PostgreSQL-Quellcode wird mit dem Versionskontrollsystem Git gespeichert und verwaltet. Ein öffentlicher Spiegel des Haupt-Repositorys ist verfügbar; er wird innerhalb einer Minute nach jeder Änderung am Haupt-Repository aktualisiert.

Das Wiki enthält weitere Hinweise zur Arbeit mit Git:

<https://wiki.postgresql.org/wiki/Working_with_Git>

### I.1. Den Quellcode mit Git beziehen

Mit Git erstellen Sie eine Kopie des gesamten Code-Repositorys auf Ihrem lokalen Rechner. Dadurch haben Sie offline Zugriff auf die gesamte Historie und alle Branches. Dies ist die schnellste und flexibelste Methode, um Patches zu entwickeln oder zu testen.

1. Sie benötigen eine installierte Version von Git. Git erhalten Sie unter <https://git-scm.com>. Viele Systeme haben bereits standardmäßig eine aktuelle Git-Version installiert oder stellen sie über ihr Paketverwaltungssystem bereit.

2. Um mit dem Git-Repository zu beginnen, klonen Sie den offiziellen Spiegel:

   ```sh
   git clone https://git.postgresql.org/git/postgresql.git
   ```

   Dadurch wird das vollständige Repository auf Ihren lokalen Rechner kopiert. Je nach Internetverbindung kann das eine Weile dauern. Die Dateien werden in einem neuen Unterverzeichnis `postgresql` Ihres aktuellen Verzeichnisses abgelegt.

3. Wenn Sie später die neuesten Änderungen holen möchten, wechseln Sie in das Repository und führen Sie aus:

   ```sh
   git fetch
   ```

Git kann weit mehr, als nur den Quellcode abzurufen. Weitere Informationen finden Sie in den Git-Manpages oder auf der Website unter <https://git-scm.com>.
