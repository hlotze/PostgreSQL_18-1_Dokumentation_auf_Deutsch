# 1. Erste Schritte

## 1.1. Installation

Bevor Sie PostgreSQL verwenden können, müssen Sie es natürlich installieren. Möglicherweise ist PostgreSQL an Ihrem Standort bereits installiert, entweder weil es in Ihrer Betriebssystemdistribution enthalten ist oder weil der Systemadministrator es bereits eingerichtet hat. In diesem Fall sollten Sie der Dokumentation Ihres Betriebssystems oder Ihrem Systemadministrator entnehmen, wie Sie auf PostgreSQL zugreifen.

Wenn Sie nicht sicher sind, ob PostgreSQL bereits verfügbar ist oder ob Sie es für Ihre Experimente verwenden können, können Sie es selbst installieren. Das ist nicht schwer und kann eine gute Übung sein. PostgreSQL kann von jedem unprivilegierten Benutzer installiert werden; Superuser-Zugriff (`root`) ist nicht erforderlich.

Wenn Sie PostgreSQL selbst installieren, lesen Sie die Installationsanleitung in [Kapitel 17](17_Installation_aus_dem_Quellcode.md) und kehren Sie zu dieser Einführung zurück, sobald die Installation abgeschlossen ist. Achten Sie besonders auf den Abschnitt, in dem die passenden Umgebungsvariablen eingerichtet werden.

Wenn Ihr Standortadministrator nicht die Standardeinstellungen verwendet hat, müssen Sie eventuell noch etwas nacharbeiten. Wenn der Datenbankserver beispielsweise auf einem entfernten Rechner läuft, müssen Sie die Umgebungsvariable `PGHOST` auf den Namen dieses Rechners setzen. Eventuell muss auch `PGPORT` gesetzt werden. Kurz gesagt: Wenn Sie versuchen, ein Anwendungsprogramm zu starten, und es meldet, dass es keine Verbindung zur Datenbank herstellen kann, sollten Sie Ihren Standortadministrator fragen oder, wenn Sie selbst verantwortlich sind, in der Dokumentation prüfen, ob Ihre Umgebung korrekt eingerichtet ist. Wenn der vorangehende Absatz unklar war, lesen Sie den nächsten Abschnitt.

## 1.2. Architektonische Grundlagen

Bevor wir weitermachen, sollten Sie die grundlegende Systemarchitektur von PostgreSQL verstehen. Wenn Sie wissen, wie die Teile von PostgreSQL zusammenwirken, wird dieses Kapitel etwas verständlicher.

In der Datenbanksprache verwendet PostgreSQL ein Client/Server-Modell. Eine PostgreSQL-Sitzung besteht aus den folgenden zusammenarbeitenden Prozessen, also Programmen:

- Ein Serverprozess verwaltet die Datenbankdateien, nimmt Verbindungen von Client-Anwendungen zur Datenbank entgegen und führt Datenbankaktionen im Auftrag der Clients aus. Das Datenbankserverprogramm heißt `postgres`.
- Die Client-Anwendung (Frontend) des Benutzers möchte Datenbankoperationen ausführen. Client-Anwendungen können sehr unterschiedlicher Art sein: ein textorientiertes Werkzeug, eine grafische Anwendung, ein Webserver, der auf die Datenbank zugreift, um Webseiten anzuzeigen, oder ein spezialisiertes Werkzeug zur Datenbankwartung. Einige Client-Anwendungen werden mit der PostgreSQL-Distribution mitgeliefert; die meisten werden von Benutzern entwickelt.

Wie bei Client/Server-Anwendungen üblich, können Client und Server auf verschiedenen Hosts laufen. In diesem Fall kommunizieren sie über eine TCP/IP-Netzwerkverbindung. Das sollten Sie im Hinterkopf behalten, denn Dateien, auf die auf dem Client-Rechner zugegriffen werden kann, sind auf dem Datenbankserver möglicherweise nicht verfügbar oder nur unter einem anderen Dateinamen.

Der PostgreSQL-Server kann mehrere gleichzeitige Verbindungen von Clients verarbeiten. Dazu startet er für jede Verbindung einen neuen Prozess ("fork"). Von diesem Zeitpunkt an kommunizieren der Client und der neue Serverprozess ohne Eingriff des ursprünglichen `postgres`-Prozesses. Der überwachende Serverprozess läuft also ständig und wartet auf Client-Verbindungen, während Clients und die zugehörigen Serverprozesse kommen und gehen. Für den Benutzer ist all dies natürlich unsichtbar; es wird hier nur der Vollständigkeit halber erwähnt.

## 1.3. Eine Datenbank erstellen

Der erste Test, ob Sie auf den Datenbankserver zugreifen können, besteht darin, eine Datenbank zu erstellen. Ein laufender PostgreSQL-Server kann viele Datenbanken verwalten. Typischerweise wird für jedes Projekt oder für jeden Benutzer eine eigene Datenbank verwendet.

Möglicherweise hat Ihr Standortadministrator bereits eine Datenbank für Sie angelegt. In diesem Fall können Sie diesen Schritt auslassen und mit dem nächsten Abschnitt fortfahren.

Um von der Kommandozeile aus eine neue Datenbank zu erstellen, in diesem Beispiel mit dem Namen `mydb`, verwenden Sie den folgenden Befehl:

```console
$ createdb mydb
```

Wenn keine Ausgabe erscheint, war dieser Schritt erfolgreich, und Sie können den Rest dieses Abschnitts überspringen.

Wenn Sie eine Meldung wie diese sehen:

```console
createdb: command not found
```

dann wurde PostgreSQL nicht korrekt installiert. Entweder wurde es gar nicht installiert, oder der Suchpfad Ihrer Shell enthält das Programm nicht. Versuchen Sie stattdessen, den Befehl mit einem absoluten Pfad aufzurufen:

```console
$ /usr/local/pgsql/bin/createdb mydb
```

Der Pfad kann an Ihrem Standort anders lauten. Wenden Sie sich an Ihren Standortadministrator oder prüfen Sie die Installationsanleitung, um die Situation zu korrigieren.

Eine andere Antwort könnte so aussehen:

```console
createdb: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: No such file or directory
        Is the server running locally and accepting connections on that socket?
```

Das bedeutet, dass der Server nicht gestartet wurde oder nicht dort lauscht, wo `createdb` ihn zu erreichen erwartet. Prüfen Sie erneut die Installationsanleitung oder fragen Sie den Administrator.

Eine weitere mögliche Antwort ist:

```console
createdb: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL: role "joe" does not exist
```

wobei Ihr eigener Anmeldename genannt wird. Das passiert, wenn der Administrator noch kein PostgreSQL-Benutzerkonto für Sie erstellt hat. PostgreSQL-Benutzerkonten sind von Betriebssystem-Benutzerkonten getrennt. Wenn Sie der Administrator sind, finden Sie Hilfe zum Anlegen von Konten in [Kapitel 21](21_Datenbankrollen.md). Um das erste Benutzerkonto anzulegen, müssen Sie zum Betriebssystembenutzer wechseln, unter dem PostgreSQL installiert wurde, normalerweise `postgres`. Es kann auch sein, dass Ihnen ein PostgreSQL-Benutzername zugewiesen wurde, der sich von Ihrem Betriebssystembenutzernamen unterscheidet; in diesem Fall müssen Sie den Schalter `-U` verwenden oder die Umgebungsvariable `PGUSER` setzen, um Ihren PostgreSQL-Benutzernamen anzugeben.

Wenn Sie ein Benutzerkonto haben, dieses aber nicht die erforderlichen Rechte zum Erstellen einer Datenbank besitzt, sehen Sie Folgendes:

```console
createdb: error: database creation failed: ERROR: permission denied to create database
```

Nicht jeder Benutzer ist berechtigt, neue Datenbanken zu erstellen. Wenn PostgreSQL Ihnen das Erstellen von Datenbanken verweigert, muss der Standortadministrator Ihnen die entsprechende Berechtigung erteilen. Wenden Sie sich in diesem Fall an Ihren Standortadministrator. Wenn Sie PostgreSQL selbst installiert haben, sollten Sie sich für dieses Tutorial mit dem Benutzerkonto anmelden, unter dem Sie den Server gestartet haben.

Sie können auch Datenbanken mit anderen Namen erstellen. PostgreSQL erlaubt an einem Standort beliebig viele Datenbanken. Datenbanknamen müssen mit einem alphabetischen Zeichen beginnen und sind auf 63 Byte Länge begrenzt. Eine praktische Wahl ist eine Datenbank mit demselben Namen wie Ihr aktueller Benutzername. Viele Werkzeuge nehmen diesen Datenbanknamen als Standard an, wodurch Sie etwas Tipparbeit sparen. Um eine solche Datenbank zu erstellen, geben Sie einfach ein:

```console
$ createdb
```

Wenn Sie Ihre Datenbank nicht mehr verwenden möchten, können Sie sie entfernen. Wenn Sie beispielsweise Eigentümer (Ersteller) der Datenbank `mydb` sind, können Sie sie mit folgendem Befehl löschen:

```console
$ dropdb mydb
```

Bei diesem Befehl wird der Datenbankname nicht standardmäßig aus dem Benutzerkontonamen abgeleitet; Sie müssen ihn immer angeben. Diese Aktion entfernt physisch alle Dateien, die mit der Datenbank verbunden sind, und kann nicht rückgängig gemacht werden. Sie sollte daher nur nach sorgfältiger Überlegung ausgeführt werden.

Mehr über `createdb` und `dropdb` finden Sie in der jeweiligen Dokumentation zu `createdb` und `dropdb`.

## 1.4. Auf eine Datenbank zugreifen

Nachdem Sie eine Datenbank erstellt haben, können Sie auf sie zugreifen, indem Sie:

- das interaktive PostgreSQL-Terminalprogramm `psql` ausführen, mit dem Sie SQL-Befehle interaktiv eingeben, bearbeiten und ausführen können,
- ein vorhandenes grafisches Frontend-Werkzeug wie `pgAdmin` oder eine Office-Suite mit ODBC- oder JDBC-Unterstützung verwenden, um eine Datenbank zu erstellen und zu bearbeiten. Diese Möglichkeiten werden in diesem Tutorial nicht behandelt,
- eine eigene Anwendung schreiben und dafür eine der mehreren verfügbaren Sprachbindungen verwenden. Diese Möglichkeiten werden ausführlicher in Teil IV behandelt.

Sie werden wahrscheinlich zunächst `psql` starten wollen, um die Beispiele in diesem Tutorial auszuprobieren. Für die Datenbank `mydb` kann es mit folgendem Befehl gestartet werden:

```console
$ psql mydb
```

Wenn Sie den Datenbanknamen nicht angeben, wird standardmäßig Ihr Benutzerkontoname verwendet. Dieses Schema haben Sie im vorigen Abschnitt bereits mit `createdb` kennengelernt.

In `psql` werden Sie mit folgender Meldung begrüßt:

```console
psql (18.1)
Type "help" for help.

mydb=>
```

Die letzte Zeile könnte auch so aussehen:

```console
mydb=#
```

Das würde bedeuten, dass Sie Datenbank-Superuser sind, was sehr wahrscheinlich ist, wenn Sie die PostgreSQL-Instanz selbst installiert haben. Superuser zu sein bedeutet, dass Sie keinen Zugriffskontrollen unterliegen. Für dieses Tutorial ist das nicht wichtig.

Wenn beim Starten von `psql` Probleme auftreten, gehen Sie zum vorherigen Abschnitt zurück. Die Diagnosemeldungen von `createdb` und `psql` ähneln sich; wenn Ersteres funktioniert hat, sollte Letzteres ebenfalls funktionieren.

Die letzte von `psql` ausgegebene Zeile ist der Prompt. Er zeigt an, dass `psql` auf Ihre Eingaben wartet und dass Sie SQL-Abfragen in einen von `psql` verwalteten Arbeitsbereich eingeben können. Probieren Sie diese Befehle aus:

```console
mydb=> SELECT version();
                                         version
-------------------------------------------------------------------
-----------------------
 PostgreSQL 18.1 on x86_64-pc-linux-gnu, compiled by gcc (Debian
 4.9.2-10) 4.9.2, 64-bit
(1 row)

mydb=> SELECT current_date;
    date
------------
 2016-01-07
(1 row)

mydb=> SELECT 2 + 2;
 ?column?
----------
(1 row)
```

Das Programm `psql` besitzt eine Reihe interner Befehle, die keine SQL-Befehle sind. Sie beginnen mit dem Backslash-Zeichen `\`. Hilfe zur Syntax verschiedener PostgreSQL-SQL-Befehle erhalten Sie beispielsweise mit:

```console
mydb=> \h
```

Um `psql` zu verlassen, geben Sie ein:

```console
mydb=> \q
```

Daraufhin beendet sich `psql` und kehrt zu Ihrer Kommando-Shell zurück. Weitere interne Befehle erhalten Sie mit `\?` am `psql`-Prompt. Die vollständigen Möglichkeiten von `psql` sind in der Dokumentation zu `psql` beschrieben. In diesem Tutorial werden wir diese Funktionen nicht ausdrücklich verwenden, Sie können sie aber selbst nutzen, wenn sie hilfreich sind.
