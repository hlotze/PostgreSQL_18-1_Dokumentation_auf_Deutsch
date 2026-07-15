# 19. Serverkonfiguration

Viele Konfigurationsparameter beeinflussen das Verhalten des Datenbanksystems. Im ersten Abschnitt dieses Kapitels wird beschrieben, wie mit Konfigurationsparametern gearbeitet wird. Die folgenden Abschnitte behandeln jeden Parameter im Detail.

## 19.1. Parameter setzen

### 19.1.1. Parameternamen und Werte

Alle Parameternamen sind unabhängig von Groß- und Kleinschreibung. Jeder Parameter erhält einen Wert aus einem von fünf Typen: Boolean, String, Integer, Gleitkommazahl oder Aufzählungstyp (`enum`). Der Typ bestimmt die Syntax, mit der der Parameter gesetzt wird:

- Boolean: Werte können als `on`, `off`, `true`, `false`, `yes`, `no`, `1`, `0` (jeweils unabhängig von Groß- und Kleinschreibung) oder als eindeutiges Präfix eines dieser Werte geschrieben werden.

- String: Im Allgemeinen wird der Wert in einfache Anführungszeichen eingeschlossen; einfache Anführungszeichen innerhalb des Werts werden verdoppelt. Anführungszeichen können jedoch meist weggelassen werden, wenn der Wert eine einfache Zahl oder ein Identifier ist. (Werte, die einem SQL-Schlüsselwort entsprechen, müssen in manchen Kontexten in Anführungszeichen gesetzt werden.)

- Numerisch (Integer und Gleitkomma): Numerische Parameter können in den üblichen Integer- und Gleitkommaformaten angegeben werden; Bruchwerte werden auf den nächstliegenden Integer gerundet, wenn der Parameter vom Typ Integer ist. Integer-Parameter akzeptieren außerdem hexadezimale Eingaben (beginnend mit `0x`) und oktale Eingaben (beginnend mit `0`), diese Formate dürfen jedoch keinen Bruchteil enthalten. Verwenden Sie keine Tausendertrennzeichen. Anführungszeichen sind nicht erforderlich, außer bei hexadezimaler Eingabe.

- Numerisch mit Einheit: Einige numerische Parameter haben eine implizite Einheit, weil sie Speicher- oder Zeitmengen beschreiben. Die Einheit kann Byte, Kilobyte, Blöcke (typischerweise acht Kilobyte), Millisekunden, Sekunden oder Minuten sein. Ein bloßer numerischer Wert für eine solche Einstellung verwendet die Standardeinheit der Einstellung, die aus `pg_settings.unit` ersichtlich ist. Der Bequemlichkeit halber können Einstellungen mit explizit angegebener Einheit geschrieben werden, zum Beispiel `'120 ms'` für einen Zeitwert; sie werden dann in die tatsächliche Einheit des Parameters umgerechnet. Beachten Sie, dass der Wert als String (mit Anführungszeichen) geschrieben werden muss, um diese Funktion zu nutzen. Der Einheitenname unterscheidet Groß- und Kleinschreibung, und zwischen numerischem Wert und Einheit darf Leerraum stehen.

  - Gültige Speichereinheiten sind `B` (Bytes), `kB` (Kilobytes), `MB` (Megabytes), `GB` (Gigabytes) und `TB` (Terabytes). Der Multiplikator für Speichereinheiten ist 1024, nicht 1000.

  - Gültige Zeiteinheiten sind `us` (Mikrosekunden), `ms` (Millisekunden), `s` (Sekunden), `min` (Minuten), `h` (Stunden) und `d` (Tage).

Wenn ein Bruchwert mit Einheit angegeben wird, wird er auf ein Vielfaches der nächstkleineren Einheit gerundet, sofern es eine solche gibt. Beispielsweise wird `30.1 GB` in `30822 MB` umgerechnet, nicht in `32319628902 B`. Ist der Parameter vom Typ Integer, erfolgt nach einer etwaigen Einheitenumrechnung eine abschließende Rundung auf einen Integer.

- Aufzählungstyp: Parameter mit Aufzählungstyp werden genauso geschrieben wie String-Parameter, sind aber auf eine begrenzte Menge von Werten eingeschränkt. Die für einen solchen Parameter zulässigen Werte können aus `pg_settings.enumvals` entnommen werden. Enum-Parameterwerte sind unabhängig von Groß- und Kleinschreibung.

### 19.1.2. Parameterinteraktion über die Konfigurationsdatei

Die grundlegendste Art, diese Parameter zu setzen, ist das Bearbeiten der Datei `postgresql.conf`, die normalerweise im Datenverzeichnis liegt. Beim Initialisieren des Datenbank-Cluster-Verzeichnisses wird eine Standardkopie installiert. Ein Beispiel dafür, wie diese Datei aussehen könnte:

```text
# This is a comment
log_connections = all
log_destination = 'syslog'
search_path = '"$user", public'
shared_buffers = 128MB
```

Pro Zeile wird ein Parameter angegeben. Das Gleichheitszeichen zwischen Name und Wert ist optional. Leerraum ist bedeutungslos (außer innerhalb eines in Anführungszeichen gesetzten Parameterwerts), und leere Zeilen werden ignoriert. Rauten (`#`) kennzeichnen den Rest der Zeile als Kommentar. Parameterwerte, die keine einfachen Identifier oder Zahlen sind, müssen in einfache Anführungszeichen gesetzt werden. Um ein einfaches Anführungszeichen in einen Parameterwert einzubetten, schreibt man entweder zwei Anführungszeichen (bevorzugt) oder Backslash-Anführungszeichen. Enthält die Datei mehrere Einträge für denselben Parameter, werden alle bis auf den letzten ignoriert.

Auf diese Weise gesetzte Parameter liefern Standardwerte für den Cluster. Aktive Sitzungen sehen diese Werte, sofern sie nicht überschrieben werden. Die folgenden Abschnitte beschreiben, wie der Administrator oder Benutzer diese Standardwerte überschreiben kann.

Die Konfigurationsdatei wird erneut gelesen, wenn der Hauptserverprozess ein `SIGHUP`-Signal erhält; dieses Signal sendet man am einfachsten, indem man `pg_ctl reload` auf der Kommandozeile ausführt oder die SQL-Funktion `pg_reload_conf()` aufruft. Der Hauptserverprozess gibt dieses Signal auch an alle derzeit laufenden Serverprozesse weiter, sodass bestehende Sitzungen ebenfalls die neuen Werte übernehmen (dies geschieht, nachdem sie einen gerade ausgeführten Client-Befehl abgeschlossen haben). Alternativ kann das Signal direkt an einen einzelnen Serverprozess gesendet werden. Manche Parameter können nur beim Serverstart gesetzt werden; Änderungen an ihren Einträgen in der Konfigurationsdatei werden bis zum Neustart des Servers ignoriert. Ungültige Parametereinstellungen in der Konfigurationsdatei werden bei der `SIGHUP`-Verarbeitung ebenfalls ignoriert (aber protokolliert).

Zusätzlich zu `postgresql.conf` enthält ein PostgreSQL-Datenverzeichnis eine Datei `postgresql.auto.conf`, die dasselbe Format wie `postgresql.conf` hat, aber automatisch und nicht manuell bearbeitet werden soll. Diese Datei enthält Einstellungen, die über den Befehl `ALTER SYSTEM` gesetzt wurden. Sie wird immer dann gelesen, wenn `postgresql.conf` gelesen wird, und ihre Einstellungen werden auf dieselbe Weise wirksam. Einstellungen in `postgresql.auto.conf` überschreiben Einstellungen in `postgresql.conf`.

Externe Werkzeuge können `postgresql.auto.conf` ebenfalls ändern. Es wird nicht empfohlen, dies bei laufendem Server zu tun, es sei denn, `allow_alter_system` ist auf `off` gesetzt, da ein gleichzeitiger `ALTER SYSTEM`-Befehl solche Änderungen überschreiben könnte. Solche Werkzeuge können neue Einstellungen einfach am Ende anhängen oder doppelte Einstellungen und/oder Kommentare entfernen (wie es `ALTER SYSTEM` tut).

Die System-View `pg_file_settings` kann hilfreich sein, um Änderungen an den Konfigurationsdateien vorab zu testen oder Probleme zu diagnostizieren, wenn ein `SIGHUP`-Signal nicht die gewünschte Wirkung hatte.

### 19.1.3. Parameterinteraktion über SQL

PostgreSQL stellt drei SQL-Befehle bereit, mit denen Konfigurationsvorgaben festgelegt werden können. Der bereits erwähnte Befehl `ALTER SYSTEM` bietet eine über SQL erreichbare Möglichkeit, globale Vorgaben zu ändern; funktional entspricht dies dem Bearbeiten von `postgresql.conf`. Zusätzlich gibt es zwei Befehle, mit denen Vorgaben datenbank- oder rollenbezogen gesetzt werden können:

- Der Befehl `ALTER DATABASE` erlaubt es, globale Einstellungen pro Datenbank zu überschreiben.

- Der Befehl `ALTER ROLE` erlaubt es, sowohl globale als auch datenbankbezogene Einstellungen mit benutzerspezifischen Werten zu überschreiben.

Mit `ALTER DATABASE` und `ALTER ROLE` gesetzte Werte werden erst beim Start einer neuen Datenbanksitzung angewendet. Sie überschreiben Werte aus den Konfigurationsdateien oder von der Server-Kommandozeile und bilden die Vorgaben für den Rest der Sitzung. Beachten Sie, dass manche Einstellungen nach dem Serverstart nicht geändert werden können und daher weder mit diesen Befehlen noch mit den unten aufgeführten Befehlen gesetzt werden können.

Sobald ein Client mit der Datenbank verbunden ist, stellt PostgreSQL zwei weitere SQL-Befehle (und entsprechende Funktionen) bereit, um mit sitzungslokalen Konfigurationseinstellungen zu arbeiten:

- Der Befehl `SHOW` erlaubt die Prüfung des aktuellen Werts eines beliebigen Parameters. Die entsprechende SQL-Funktion ist `current_setting(setting_name text)` (siehe Abschnitt 9.28.1).

- Der Befehl `SET` erlaubt die Änderung des aktuellen Werts jener Parameter, die lokal für eine Sitzung gesetzt werden können; er hat keine Auswirkungen auf andere Sitzungen. Viele Parameter können auf diese Weise von jedem Benutzer gesetzt werden, einige jedoch nur von Superusern und Benutzern, denen das `SET`-Privileg für diesen Parameter erteilt wurde. Die entsprechende SQL-Funktion ist `set_config(setting_name, new_value, is_local)` (siehe Abschnitt 9.28.1).

Zusätzlich kann die System-View `pg_settings` verwendet werden, um sitzungslokale Werte anzuzeigen und zu ändern:

- Eine Abfrage dieser View ähnelt `SHOW ALL`, liefert aber mehr Details. Sie ist außerdem flexibler, weil Filterbedingungen angegeben oder Joins mit anderen Relationen durchgeführt werden können.

- Ein `UPDATE` auf diese View, insbesondere eine Aktualisierung der Spalte `setting`, entspricht dem Ausführen von `SET`-Befehlen. Zum Beispiel entspricht

```sql
SET configuration_parameter TO DEFAULT;
```

diesem Befehl:

```sql
UPDATE pg_settings SET setting = reset_val WHERE name =
 'configuration_parameter';
```

### 19.1.4. Parameterinteraktion über die Shell

Zusätzlich zum Setzen globaler Vorgaben oder zum Hinterlegen von Überschreibungen auf Datenbank- oder Rollenebene können Einstellungen über Shell-Mechanismen an PostgreSQL übergeben werden. Sowohl der Server als auch die Client-Bibliothek `libpq` akzeptieren Parameterwerte über die Shell.

- Beim Serverstart können Parametereinstellungen mit dem Kommandozeilenparameter `-c name=value` oder der entsprechenden Variante `--name=value` an den Befehl `postgres` übergeben werden. Zum Beispiel:

```text
postgres -c log_connections=all --log-destination='syslog'
```

  Auf diese Weise bereitgestellte Einstellungen überschreiben Einstellungen aus `postgresql.conf` oder `ALTER SYSTEM`, sodass sie ohne Neustart des Servers nicht global geändert werden können.

- Beim Starten einer Client-Sitzung über `libpq` können Parametereinstellungen mit der Umgebungsvariablen `PGOPTIONS` angegeben werden. Auf diese Weise eingerichtete Einstellungen bilden Vorgaben für die Dauer der Sitzung, beeinflussen aber keine anderen Sitzungen. Aus historischen Gründen ähnelt das Format von `PGOPTIONS` dem Format beim Starten des Befehls `postgres`; insbesondere muss vor dem Namen `-c` oder ein vorangestelltes `--` angegeben werden. Zum Beispiel:

```text
env PGOPTIONS="-c geqo=off --statement-timeout=5min" psql
```

  Andere Clients und Bibliotheken können eigene Mechanismen über die Shell oder auf andere Weise bereitstellen, mit denen Benutzer Sitzungseinstellungen ändern können, ohne direkt SQL-Befehle zu verwenden.

### 19.1.5. Inhalte von Konfigurationsdateien verwalten

PostgreSQL bietet mehrere Funktionen, um komplexe `postgresql.conf`-Dateien in Teildateien aufzuteilen. Diese Funktionen sind besonders nützlich, wenn mehrere Server mit verwandten, aber nicht identischen Konfigurationen verwaltet werden.

Zusätzlich zu einzelnen Parametereinstellungen kann die Datei `postgresql.conf` `include`-Direktiven enthalten. Diese geben eine andere Datei an, die gelesen und so verarbeitet wird, als wäre sie an dieser Stelle in die Konfigurationsdatei eingefügt. Mit dieser Funktion kann eine Konfigurationsdatei in physisch getrennte Teile aufgeteilt werden. `include`-Direktiven sehen einfach so aus:

```text
include 'filename'
```

Wenn der Dateiname kein absoluter Pfad ist, wird er relativ zum Verzeichnis der verweisenden Konfigurationsdatei interpretiert. Einbindungen können verschachtelt werden.

Es gibt außerdem eine `include_if_exists`-Direktive, die sich genauso verhält wie die `include`-Direktive, außer wenn die referenzierte Datei nicht existiert oder nicht gelesen werden kann. Ein normales `include` betrachtet dies als Fehlerbedingung, `include_if_exists` protokolliert dagegen lediglich eine Meldung und setzt die Verarbeitung der verweisenden Konfigurationsdatei fort.

Die Datei `postgresql.conf` kann auch `include_dir`-Direktiven enthalten, die ein ganzes Verzeichnis von Konfigurationsdateien einbinden. Sie sehen so aus:

```text
include_dir 'directory'
```

Nicht absolute Verzeichnisnamen werden relativ zum Verzeichnis der verweisenden Konfigurationsdatei interpretiert. Innerhalb des angegebenen Verzeichnisses werden nur Nicht-Verzeichnisdateien eingebunden, deren Namen mit dem Suffix `.conf` enden. Dateinamen, die mit dem Zeichen `.` beginnen, werden ebenfalls ignoriert, um Fehler zu vermeiden, da solche Dateien auf manchen Plattformen verborgen sind. Mehrere Dateien in einem Include-Verzeichnis werden in Dateinamensreihenfolge verarbeitet (gemäß den Regeln der C-Locale, d. h. Zahlen vor Buchstaben und Großbuchstaben vor Kleinbuchstaben).

Include-Dateien oder -Verzeichnisse können verwendet werden, um Teile der Datenbankkonfiguration logisch zu trennen, statt eine einzige große Datei `postgresql.conf` zu haben. Nehmen wir ein Unternehmen mit zwei Datenbankservern, die unterschiedlich viel Speicher besitzen. Es gibt wahrscheinlich Konfigurationselemente, die beide teilen, etwa für das Logging. Speicherbezogene Parameter auf dem Server unterscheiden sich jedoch zwischen den beiden. Außerdem kann es serverspezifische Anpassungen geben. Eine Möglichkeit, diese Situation zu verwalten, besteht darin, die benutzerdefinierten Konfigurationsänderungen für den Standort in drei Dateien aufzuteilen. Am Ende der Datei `postgresql.conf` könnte Folgendes ergänzt werden, um sie einzubinden:

```text
include 'shared.conf'
include 'memory.conf'
include 'server.conf'
```

Alle Systeme hätten dieselbe Datei `shared.conf`. Jeder Server mit einer bestimmten Speichermenge könnte dieselbe Datei `memory.conf` verwenden; man könnte eine für alle Server mit 8 GB RAM haben und eine andere für Server mit 16 GB. Schließlich könnte `server.conf` wirklich serverspezifische Konfigurationsinformationen enthalten.

Eine andere Möglichkeit besteht darin, ein Konfigurationsdateiverzeichnis anzulegen und diese Informationen dort in Dateien abzulegen. Zum Beispiel könnte am Ende von `postgresql.conf` ein Verzeichnis `conf.d` referenziert werden:

```text
include_dir 'conf.d'
```

Dann könnten die Dateien im Verzeichnis `conf.d` so benannt werden:

```text
00shared.conf
01memory.conf
02server.conf
```

Diese Namenskonvention legt eine klare Reihenfolge fest, in der die Dateien geladen werden. Das ist wichtig, weil nur die letzte Einstellung verwendet wird, auf die der Server beim Lesen der Konfigurationsdateien für einen bestimmten Parameter stößt. In diesem Beispiel würde eine Einstellung in `conf.d/02server.conf` einen Wert aus `conf.d/01memory.conf` überschreiben.

Stattdessen könnte man die Dateien mit diesem Ansatz beschreibend benennen:

```text
00shared.conf
01memory-8GB.conf
02server-foo.conf
```

Eine solche Anordnung gibt jeder Konfigurationsdateivariante einen eindeutigen Namen. Das kann Mehrdeutigkeiten vermeiden, wenn die Konfigurationen mehrerer Server an einem gemeinsamen Ort gespeichert werden, etwa in einem Versionskontroll-Repository. (Datenbankkonfigurationsdateien unter Versionskontrolle zu stellen, ist eine weitere empfehlenswerte Praxis.)

## 19.2. Dateispeicherorte

Zusätzlich zur bereits erwähnten Datei `postgresql.conf` verwendet PostgreSQL zwei weitere manuell bearbeitete Konfigurationsdateien, die die Client-Authentifizierung steuern (ihre Verwendung wird in [Kapitel 20](20_Client_Authentifizierung.md) erläutert). Standardmäßig werden alle drei Konfigurationsdateien im Datenverzeichnis des Datenbank-Clusters gespeichert. Die in diesem Abschnitt beschriebenen Parameter erlauben es, die Konfigurationsdateien an anderer Stelle abzulegen. (Das kann die Administration erleichtern. Insbesondere ist es oft einfacher sicherzustellen, dass die Konfigurationsdateien ordnungsgemäß gesichert werden, wenn sie getrennt aufbewahrt werden.)

`data_directory` (`string`)

Gibt das Verzeichnis an, das für die Datenspeicherung verwendet werden soll. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`config_file` (`string`)

Gibt die Hauptkonfigurationsdatei des Servers an (üblicherweise `postgresql.conf` genannt). Dieser Parameter kann nur auf der Kommandozeile von `postgres` gesetzt werden.

`hba_file` (`string`)

Gibt die Konfigurationsdatei für hostbasierte Authentifizierung an (üblicherweise `pg_hba.conf` genannt). Dieser Parameter kann nur beim Serverstart gesetzt werden.

`ident_file` (`string`)

Gibt die Konfigurationsdatei für Benutzernamenzuordnungen an (üblicherweise `pg_ident.conf` genannt). Dieser Parameter kann nur beim Serverstart gesetzt werden. Siehe auch [Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen).

`external_pid_file` (`string`)

Gibt den Namen einer zusätzlichen Prozess-ID-Datei (PID-Datei) an, die der Server zur Verwendung durch Serveradministrationsprogramme erstellen soll. Dieser Parameter kann nur beim Serverstart gesetzt werden.

In einer Standardinstallation ist keiner der obigen Parameter explizit gesetzt. Stattdessen wird das Datenverzeichnis über die Kommandozeilenoption `-D` oder die Umgebungsvariable `PGDATA` angegeben, und alle Konfigurationsdateien befinden sich im Datenverzeichnis.

Wenn die Konfigurationsdateien nicht im Datenverzeichnis aufbewahrt werden sollen, muss die Kommandozeilenoption `postgres -D` oder die Umgebungsvariable `PGDATA` auf das Verzeichnis zeigen, das die Konfigurationsdateien enthält, und der Parameter `data_directory` muss in `postgresql.conf` (oder auf der Kommandozeile) gesetzt werden, um anzugeben, wo sich das Datenverzeichnis tatsächlich befindet. Beachten Sie, dass `data_directory` `-D` und `PGDATA` für den Speicherort des Datenverzeichnisses überschreibt, nicht aber für den Speicherort der Konfigurationsdateien.

Falls gewünscht, können die Namen und Speicherorte der Konfigurationsdateien einzeln mit den Parametern `config_file`, `hba_file` und/oder `ident_file` angegeben werden. `config_file` kann nur auf der Kommandozeile von `postgres` angegeben werden, die anderen können jedoch innerhalb der Hauptkonfigurationsdatei gesetzt werden. Wenn alle drei Parameter plus `data_directory` explizit gesetzt sind, ist es nicht notwendig, `-D` oder `PGDATA` anzugeben.

Beim Setzen eines dieser Parameter wird ein relativer Pfad relativ zu dem Verzeichnis interpretiert, in dem `postgres` gestartet wird.

## 19.3. Verbindungen und Authentifizierung

### 19.3.1. Verbindungseinstellungen

`listen_addresses` (`string`)

Gibt die TCP/IP-Adresse(n) an, auf denen der Server Verbindungen von Client-Anwendungen entgegennimmt. Der Wert hat die Form einer kommagetrennten Liste von Hostnamen und/oder numerischen IP-Adressen. Der besondere Eintrag `*` entspricht allen verfügbaren IP-Schnittstellen. Der Eintrag `0.0.0.0` erlaubt das Lauschen auf allen IPv4-Adressen, und `::` erlaubt das Lauschen auf allen IPv6-Adressen. Ist die Liste leer, lauscht der Server überhaupt auf keiner IP-Schnittstelle; in diesem Fall können nur Unix-Domain-Sockets für Verbindungen verwendet werden. Ist die Liste nicht leer, startet der Server, wenn er auf mindestens einer TCP/IP-Adresse lauschen kann. Für jede TCP/IP-Adresse, die nicht geöffnet werden kann, wird eine Warnung ausgegeben. Der Standardwert ist `localhost`; damit sind nur lokale TCP/IP-Loopback-Verbindungen möglich.

Während die Client-Authentifizierung ([Kapitel 20](20_Client_Authentifizierung.md)) eine fein abgestufte Kontrolle darüber erlaubt, wer auf den Server zugreifen darf, steuert `listen_addresses`, welche Schnittstellen Verbindungsversuche annehmen. Das kann helfen, wiederholte bösartige Verbindungsanfragen auf unsicheren Netzwerkschnittstellen zu verhindern. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`port` (`integer`)

Der TCP-Port, auf dem der Server lauscht; standardmäßig `5432`. Beachten Sie, dass dieselbe Portnummer für alle IP-Adressen verwendet wird, auf denen der Server lauscht. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`max_connections` (`integer`)

Bestimmt die maximale Anzahl gleichzeitiger Verbindungen zum Datenbankserver. Der Standardwert beträgt typischerweise 100 Verbindungen, kann aber niedriger sein, wenn die Kernel-Einstellungen dies nicht unterstützen (wie während `initdb` ermittelt). Dieser Parameter kann nur beim Serverstart gesetzt werden.

PostgreSQL dimensioniert bestimmte Ressourcen direkt anhand des Werts von `max_connections`. Eine Erhöhung dieses Werts führt zu einer höheren Zuweisung solcher Ressourcen, einschließlich Shared Memory.

Beim Betrieb eines Standby-Servers muss dieser Parameter auf denselben oder einen höheren Wert als auf dem Primärserver gesetzt werden. Andernfalls werden auf dem Standby-Server keine Abfragen erlaubt.

`reserved_connections` (`integer`)

Bestimmt die Anzahl der Verbindungs-"Slots", die für Verbindungen von Rollen mit Privilegien der Rolle `pg_use_reserved_connections` reserviert sind. Wenn die Anzahl freier Verbindungs-Slots größer als `superuser_reserved_connections`, aber kleiner oder gleich der Summe aus `superuser_reserved_connections` und `reserved_connections` ist, werden neue Verbindungen nur für Superuser und Rollen mit Privilegien von `pg_use_reserved_connections` akzeptiert. Wenn `superuser_reserved_connections` oder weniger Verbindungs-Slots verfügbar sind, werden neue Verbindungen nur für Superuser akzeptiert.

Der Standardwert ist null Verbindungen. Der Wert muss kleiner sein als `max_connections` minus `superuser_reserved_connections`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`superuser_reserved_connections` (`integer`)

Bestimmt die Anzahl der Verbindungs-"Slots", die für Verbindungen von PostgreSQL-Superusern reserviert sind. Höchstens `max_connections` Verbindungen können jemals gleichzeitig aktiv sein. Wenn die Anzahl aktiver gleichzeitiger Verbindungen mindestens `max_connections` minus `superuser_reserved_connections` beträgt, werden neue Verbindungen nur für Superuser akzeptiert. Die durch diesen Parameter reservierten Verbindungs-Slots sind als letzte Reserve für Notfälle gedacht, nachdem die durch `reserved_connections` reservierten Slots erschöpft sind.

Der Standardwert beträgt drei Verbindungen. Der Wert muss kleiner sein als `max_connections` minus `reserved_connections`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`unix_socket_directories` (`string`)

Gibt das Verzeichnis der Unix-Domain-Sockets an, auf denen der Server Verbindungen von Client-Anwendungen entgegennimmt. Mehrere Sockets können erstellt werden, indem mehrere Verzeichnisse kommagetrennt aufgelistet werden. Leerraum zwischen Einträgen wird ignoriert; schließen Sie einen Verzeichnisnamen in doppelte Anführungszeichen ein, wenn der Name Leerraum oder Kommas enthalten muss. Ein leerer Wert bedeutet, dass auf keinen Unix-Domain-Sockets gelauscht wird; in diesem Fall können nur TCP/IP-Sockets für Verbindungen zum Server verwendet werden.

Ein Wert, der mit `@` beginnt, gibt an, dass ein Unix-Domain-Socket im abstrakten Namensraum erstellt werden soll (derzeit nur unter Linux unterstützt). In diesem Fall bezeichnet der Wert kein "Verzeichnis", sondern ein Präfix, aus dem der tatsächliche Socket-Name auf dieselbe Weise berechnet wird wie im Dateisystem-Namensraum. Obwohl das abstrakte Socket-Namenspräfix frei gewählt werden kann, weil es kein Dateisystemort ist, ist es dennoch Konvention, dateisystemähnliche Werte wie `@/tmp` zu verwenden.

Der Standardwert ist normalerweise `/tmp`, kann aber zur Build-Zeit geändert werden. Unter Windows ist der Standardwert leer, was bedeutet, dass standardmäßig kein Unix-Domain-Socket erstellt wird. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Zusätzlich zur Socket-Datei selbst, die `.s.PGSQL.nnnn` heißt, wobei `nnnn` die Portnummer des Servers ist, wird in jedem der Verzeichnisse aus `unix_socket_directories` eine gewöhnliche Datei namens `.s.PGSQL.nnnn.lock` erstellt. Keine der beiden Dateien sollte jemals manuell entfernt werden. Für Sockets im abstrakten Namensraum wird keine Lock-Datei erstellt.

`unix_socket_group` (`string`)

Setzt die besitzende Gruppe der Unix-Domain-Sockets. (Der besitzende Benutzer der Sockets ist immer der Benutzer, der den Server startet.) In Kombination mit dem Parameter `unix_socket_permissions` kann dies als zusätzlicher Zugriffskontrollmechanismus für Unix-Domain-Verbindungen verwendet werden. Standardmäßig ist dies der leere String, wodurch die Standardgruppe des Serverbenutzers verwendet wird. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Dieser Parameter wird unter Windows nicht unterstützt. Jede Einstellung wird ignoriert. Außerdem haben Sockets im abstrakten Namensraum keinen Dateibesitzer, daher wird diese Einstellung auch in diesem Fall ignoriert.

`unix_socket_permissions` (`integer`)

Setzt die Zugriffsrechte der Unix-Domain-Sockets. Unix-Domain-Sockets verwenden den üblichen Unix-Dateisystem-Rechtesatz. Der Parameterwert wird als numerischer Modus in dem Format erwartet, das von den Systemaufrufen `chmod` und `umask` akzeptiert wird. (Um das übliche oktale Format zu verwenden, muss die Zahl mit einer `0` (Null) beginnen.)

Die Standardrechte sind `0777`, was bedeutet, dass sich jeder verbinden kann. Sinnvolle Alternativen sind `0770` (nur Benutzer und Gruppe, siehe auch `unix_socket_group`) und `0700` (nur Benutzer). (Beachten Sie, dass bei einem Unix-Domain-Socket nur die Schreibberechtigung relevant ist; es hat also keinen Zweck, Lese- oder Ausführungsrechte zu setzen oder zu entziehen.)

Dieser Zugriffskontrollmechanismus ist unabhängig von dem in [Kapitel 20](20_Client_Authentifizierung.md) beschriebenen Mechanismus.

Dieser Parameter kann nur beim Serverstart gesetzt werden.

Dieser Parameter ist auf Systemen irrelevant, die Socket-Rechte vollständig ignorieren, insbesondere Solaris ab Solaris 10. Dort kann ein ähnlicher Effekt erzielt werden, indem `unix_socket_directories` auf ein Verzeichnis zeigt, dessen Suchberechtigung auf die gewünschte Zielgruppe beschränkt ist.

Sockets im abstrakten Namensraum haben keine Dateirechte, daher wird diese Einstellung auch in diesem Fall ignoriert.

`bonjour` (`boolean`)

Aktiviert die Bekanntmachung der Existenz des Servers über Bonjour. Der Standardwert ist `off`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`bonjour_name` (`string`)

Gibt den Bonjour-Dienstnamen an. Der Computername wird verwendet, wenn dieser Parameter auf den leeren String `''` gesetzt ist (das ist der Standardwert). Dieser Parameter wird ignoriert, wenn der Server nicht mit Bonjour-Unterstützung kompiliert wurde. Dieser Parameter kann nur beim Serverstart gesetzt werden.

### 19.3.2. TCP-Einstellungen

`tcp_keepalives_idle` (`integer`)

Gibt die Zeit ohne Netzwerkaktivität an, nach der das Betriebssystem eine TCP-Keepalive-Nachricht an den Client senden soll. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Ein Wert von `0` (der Standardwert) wählt den Standardwert des Betriebssystems. Unter Windows setzt ein Wert von `0` diesen Parameter auf 2 Stunden, da Windows keine Möglichkeit bereitstellt, den Systemstandardwert auszulesen. Dieser Parameter wird nur auf Systemen unterstützt, die `TCP_KEEPIDLE` oder eine entsprechende Socket-Option unterstützen, sowie unter Windows; auf anderen Systemen muss er null sein. In Sitzungen, die über einen Unix-Domain-Socket verbunden sind, wird dieser Parameter ignoriert und immer als null gelesen.

`tcp_keepalives_interval` (`integer`)

Gibt die Zeit an, nach der eine TCP-Keepalive-Nachricht, die vom Client nicht bestätigt wurde, erneut übertragen werden soll. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Ein Wert von `0` (der Standardwert) wählt den Standardwert des Betriebssystems. Unter Windows setzt ein Wert von `0` diesen Parameter auf 1 Sekunde, da Windows keine Möglichkeit bereitstellt, den Systemstandardwert auszulesen. Dieser Parameter wird nur auf Systemen unterstützt, die `TCP_KEEPINTVL` oder eine entsprechende Socket-Option unterstützen, sowie unter Windows; auf anderen Systemen muss er null sein. In Sitzungen, die über einen Unix-Domain-Socket verbunden sind, wird dieser Parameter ignoriert und immer als null gelesen.

`tcp_keepalives_count` (`integer`)

Gibt die Anzahl von TCP-Keepalive-Nachrichten an, die verloren gehen dürfen, bevor die Verbindung des Servers zum Client als tot betrachtet wird. Ein Wert von `0` (der Standardwert) wählt den Standardwert des Betriebssystems. Dieser Parameter wird nur auf Systemen unterstützt, die `TCP_KEEPCNT` oder eine entsprechende Socket-Option unterstützen (Windows gehört nicht dazu); auf anderen Systemen muss er null sein. In Sitzungen, die über einen Unix-Domain-Socket verbunden sind, wird dieser Parameter ignoriert und immer als null gelesen.

`tcp_user_timeout` (`integer`)

Gibt die Zeit an, die übertragene Daten unbestätigt bleiben dürfen, bevor die TCP-Verbindung zwangsweise geschlossen wird. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Ein Wert von `0` (der Standardwert) wählt den Standardwert des Betriebssystems. Dieser Parameter wird nur auf Systemen unterstützt, die `TCP_USER_TIMEOUT` unterstützen (Windows gehört nicht dazu); auf anderen Systemen muss er null sein. In Sitzungen, die über einen Unix-Domain-Socket verbunden sind, wird dieser Parameter ignoriert und immer als null gelesen.

`client_connection_check_interval` (`integer`)

Setzt das Zeitintervall zwischen optionalen Prüfungen, ob der Client während der Ausführung von Abfragen noch verbunden ist. Die Prüfung erfolgt durch Polling des Sockets und erlaubt es, lang laufende Abfragen früher abzubrechen, wenn der Kernel meldet, dass die Verbindung geschlossen ist.

Diese Option hängt von Kernel-Ereignissen ab, die von Linux, macOS, illumos und der BSD-Familie von Betriebssystemen bereitgestellt werden, und ist derzeit auf anderen Systemen nicht verfügbar.

Wird der Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert ist `0`, wodurch Verbindungsprüfungen deaktiviert werden. Ohne Verbindungsprüfungen erkennt der Server den Verlust der Verbindung erst bei der nächsten Interaktion mit dem Socket, wenn er auf Daten wartet, Daten empfängt oder Daten sendet.

Damit der Kernel selbst verlorene TCP-Verbindungen in allen Szenarien, einschließlich Netzwerkausfällen, zuverlässig und innerhalb eines bekannten Zeitrahmens erkennt, kann es außerdem nötig sein, die TCP-Keepalive-Einstellungen des Betriebssystems oder die PostgreSQL-Einstellungen `tcp_keepalives_idle`, `tcp_keepalives_interval` und `tcp_keepalives_count` anzupassen.

### 19.3.3. Authentifizierung

`authentication_timeout` (`integer`)

Maximale Zeit, die für den Abschluss der Client-Authentifizierung zulässig ist. Wenn ein potenzieller Client das Authentifizierungsprotokoll nicht innerhalb dieser Zeit abgeschlossen hat, schließt der Server die Verbindung. Dadurch wird verhindert, dass hängende Clients eine Verbindung unbegrenzt belegen. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Der Standardwert ist eine Minute (`1m`). Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`password_encryption` (`enum`)

Wenn in `CREATE ROLE` oder `ALTER ROLE` ein Passwort angegeben wird, bestimmt dieser Parameter den Algorithmus, mit dem das Passwort verschlüsselt wird. Mögliche Werte sind `scram-sha-256`, wodurch das Passwort mit SCRAM-SHA-256 verschlüsselt wird, und `md5`, wodurch das Passwort als MD5-Hash gespeichert wird. Der Standardwert ist `scram-sha-256`.

Beachten Sie, dass ältere Clients möglicherweise keine Unterstützung für den SCRAM-Authentifizierungsmechanismus haben und daher nicht mit Passwörtern funktionieren, die mit SCRAM-SHA-256 verschlüsselt wurden. Weitere Details finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

> **Warnung**
>
> Die Unterstützung für MD5-verschlüsselte Passwörter ist veraltet und wird in einer zukünftigen PostgreSQL-Version entfernt. Details zur Migration zu einem anderen Passworttyp finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

`scram_iterations` (`integer`)

Die Anzahl der Recheniterationen, die beim Verschlüsseln eines Passworts mit SCRAM-SHA-256 ausgeführt werden. Der Standardwert ist `4096`. Eine höhere Anzahl von Iterationen bietet zusätzlichen Schutz gegen Brute-Force-Angriffe auf gespeicherte Passwörter, verlangsamt aber die Authentifizierung. Eine Änderung des Werts wirkt sich nicht auf vorhandene mit SCRAM-SHA-256 verschlüsselte Passwörter aus, da die Iterationszahl zum Zeitpunkt der Verschlüsselung festgelegt wird. Um einen geänderten Wert zu nutzen, muss ein neues Passwort gesetzt werden.

`md5_password_warnings` (`boolean`)

Steuert, ob eine `WARNING` über die Veraltung von MD5-Passwörtern erzeugt wird, wenn eine `CREATE ROLE`- oder `ALTER ROLE`-Anweisung ein MD5-verschlüsseltes Passwort setzt. Der Standardwert ist `on`.

`krb_server_keyfile` (`string`)

Setzt den Speicherort der Kerberos-Schlüsseldatei des Servers. Der Standardwert ist `FILE:/usr/local/pgsql/etc/krb5.keytab` (wobei der Verzeichnisteil dem entspricht, was zur Build-Zeit als `sysconfdir` angegeben wurde; verwenden Sie `pg_config --sysconfdir`, um diesen Wert zu ermitteln). Wenn dieser Parameter auf einen leeren String gesetzt ist, wird er ignoriert und ein systemabhängiger Standardwert verwendet. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Weitere Informationen finden Sie in [Abschnitt 20.6](20_Client_Authentifizierung.md#206-gssapiauthentifizierung).

`krb_caseins_users` (`boolean`)

Legt fest, ob GSSAPI-Benutzernamen ohne Berücksichtigung der Groß- und Kleinschreibung behandelt werden sollen. Der Standardwert ist `off` (Groß- und Kleinschreibung werden unterschieden). Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`gss_accept_delegation` (`boolean`)

Legt fest, ob GSSAPI-Delegierung vom Client akzeptiert werden soll. Der Standardwert ist `off`, was bedeutet, dass Anmeldedaten vom Client nicht akzeptiert werden. Eine Änderung auf `on` veranlasst den Server, an ihn delegierte Anmeldedaten vom Client zu akzeptieren. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`oauth_validator_libraries` (`string`)

Die Bibliothek(en), die zur Validierung von OAuth-Verbindungstokens verwendet werden. Wenn nur eine Validator-Bibliothek angegeben ist, wird sie standardmäßig für alle OAuth-Verbindungen verwendet; andernfalls müssen alle `oauth`-HBA-Einträge explizit einen Validator aus dieser Liste setzen. Ist der Wert auf einen leeren String gesetzt (der Standard), werden OAuth-Verbindungen abgelehnt. Dieser Parameter kann nur in der Datei `postgresql.conf` gesetzt werden.

Validatormodule müssen separat implementiert oder beschafft werden; PostgreSQL liefert keine Standardimplementierungen mit. Weitere Informationen zur Implementierung von OAuth-Validatoren finden Sie in [Kapitel 50](50_OAuth_Validator_Module.md).

### 19.3.4. SSL

Weitere Informationen zum Einrichten von SSL finden Sie in [Abschnitt 18.9](18_Servereinrichtung_und_Betrieb.md#189-sichere-tcpipverbindungen-mit-ssl). Die Konfigurationsparameter zur Steuerung der Übertragungsverschlüsselung mit TLS-Protokollen heißen aus historischen Gründen `ssl`, obwohl die Unterstützung für das SSL-Protokoll veraltet ist. SSL wird in diesem Kontext austauschbar mit TLS verwendet.

`ssl` (`boolean`)

Aktiviert SSL-Verbindungen. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `off`.

`ssl_ca_file` (`string`)

Gibt den Namen der Datei an, die die SSL-Server-Zertifizierungsstelle (CA) enthält. Relative Pfade beziehen sich auf das Datenverzeichnis. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist leer, was bedeutet, dass keine CA-Datei geladen und keine Client-Zertifikatsprüfung durchgeführt wird.

`ssl_cert_file` (`string`)

Gibt den Namen der Datei an, die das SSL-Serverzertifikat enthält. Relative Pfade beziehen sich auf das Datenverzeichnis. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `server.crt`.

`ssl_crl_file` (`string`)

Gibt den Namen der Datei an, die die Sperrliste für SSL-Clientzertifikate (CRL) enthält. Relative Pfade beziehen sich auf das Datenverzeichnis. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist leer, was bedeutet, dass keine CRL-Datei geladen wird (sofern `ssl_crl_dir` nicht gesetzt ist).

`ssl_crl_dir` (`string`)

Gibt den Namen des Verzeichnisses an, das die Sperrliste für SSL-Clientzertifikate (CRL) enthält. Relative Pfade beziehen sich auf das Datenverzeichnis. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist leer, was bedeutet, dass keine CRLs verwendet werden (sofern `ssl_crl_file` nicht gesetzt ist).

Das Verzeichnis muss mit dem OpenSSL-Befehl `openssl rehash` oder `c_rehash` vorbereitet werden. Details finden Sie in dessen Dokumentation.

Bei Verwendung dieser Einstellung werden CRLs im angegebenen Verzeichnis bei Bedarf zum Verbindungszeitpunkt geladen. Neue CRLs können dem Verzeichnis hinzugefügt werden und werden sofort verwendet. Das unterscheidet sich von `ssl_crl_file`, bei dem die CRL in der Datei beim Serverstart oder beim erneuten Laden der Konfiguration geladen wird. Beide Einstellungen können zusammen verwendet werden.

`ssl_key_file` (`string`)

Gibt den Namen der Datei an, die den privaten SSL-Serverschlüssel enthält. Relative Pfade beziehen sich auf das Datenverzeichnis. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `server.key`.

`ssl_tls13_ciphers` (`string`)

Gibt eine Liste von Cipher Suites an, die für Verbindungen mit TLS-Version 1.3 zulässig sind. Mehrere Cipher Suites können als durch Doppelpunkte getrennte Liste angegeben werden. Bleibt der Wert leer, wird der Standardsatz von Cipher Suites in OpenSSL verwendet.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`ssl_ciphers` (`string`)

Gibt eine Liste von SSL-Ciphers an, die für Verbindungen mit TLS-Version 1.2 und niedriger zulässig sind; für Verbindungen mit TLS-Version 1.3 siehe `ssl_tls13_ciphers`. Die Syntax dieser Einstellung und eine Liste unterstützter Werte finden Sie auf der Manualseite `ciphers` im OpenSSL-Paket. Der Standardwert ist `HIGH:MEDIUM:+3DES:!aNULL`. Der Standard ist normalerweise eine sinnvolle Wahl, sofern keine besonderen Sicherheitsanforderungen bestehen.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Erklärung des Standardwerts:

- `HIGH`: Cipher Suites, die Ciphers aus der Gruppe `HIGH` verwenden (z. B. AES, Camellia, 3DES).

- `MEDIUM`: Cipher Suites, die Ciphers aus der Gruppe `MEDIUM` verwenden (z. B. RC4, SEED).

- `+3DES`: Die OpenSSL-Standardreihenfolge für `HIGH` ist problematisch, weil sie 3DES höher einordnet als AES128. Das ist falsch, weil 3DES weniger Sicherheit als AES128 bietet und außerdem deutlich langsamer ist. `+3DES` ordnet es hinter allen anderen `HIGH`- und `MEDIUM`-Ciphers ein.

- `!aNULL`: Deaktiviert anonyme Cipher Suites, die keine Authentifizierung durchführen. Solche Cipher Suites sind anfällig für MITM-Angriffe und sollten daher nicht verwendet werden.

Die Details der verfügbaren Cipher Suites variieren zwischen OpenSSL-Versionen. Verwenden Sie den Befehl `openssl ciphers -v 'HIGH:MEDIUM:+3DES:!aNULL'`, um die tatsächlichen Details für die aktuell installierte OpenSSL-Version anzuzeigen. Beachten Sie, dass diese Liste zur Laufzeit anhand des Server-Schlüsseltyps gefiltert wird.

`ssl_prefer_server_ciphers` (`boolean`)

Gibt an, ob die SSL-Cipher-Präferenzen des Servers statt derjenigen des Clients verwendet werden sollen. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `on`.

PostgreSQL-Versionen vor 9.4 haben diese Einstellung nicht und verwenden immer die Präferenzen des Clients. Diese Einstellung dient hauptsächlich der Rückwärtskompatibilität mit diesen Versionen. Die Verwendung der Serverpräferenzen ist normalerweise besser, weil es wahrscheinlicher ist, dass der Server passend konfiguriert ist.

`ssl_groups` (`string`)

Gibt den Namen der Kurve an, die beim ECDH-Schlüsselaustausch verwendet werden soll. Sie muss von allen verbindenden Clients unterstützt werden. Mehrere Kurven können als durch Doppelpunkte getrennte Liste angegeben werden. Es muss nicht dieselbe Kurve sein, die vom Elliptic-Curve-Schlüssel des Servers verwendet wird. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `X25519:prime256v1`.

OpenSSL-Namen für die gängigsten Kurven sind: `prime256v1` (NIST P-256), `secp384r1` (NIST P-384), `secp521r1` (NIST P-521). Eine unvollständige Liste verfügbarer Gruppen kann mit dem Befehl `openssl ecparam -list_curves` angezeigt werden. Nicht alle davon sind jedoch mit TLS verwendbar, und viele unterstützte Gruppennamen und Aliasnamen werden ausgelassen.

In PostgreSQL-Versionen vor 18.0 hieß diese Einstellung `ssl_ecdh_curve` und akzeptierte nur einen einzelnen Wert.

`ssl_min_protocol_version` (`enum`)

Setzt die zu verwendende minimale SSL/TLS-Protokollversion. Derzeit gültige Werte sind: `TLSv1`, `TLSv1.1`, `TLSv1.2`, `TLSv1.3`. Ältere Versionen der OpenSSL-Bibliothek unterstützen nicht alle Werte; wenn eine nicht unterstützte Einstellung gewählt wird, wird ein Fehler ausgelöst. Protokollversionen vor TLS 1.0, nämlich SSL-Version 2 und 3, sind immer deaktiviert.

Der Standardwert ist `TLSv1.2`, was zum Zeitpunkt der Erstellung bewährten Branchenpraktiken entspricht.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`ssl_max_protocol_version` (`enum`)

Setzt die zu verwendende maximale SSL/TLS-Protokollversion. Gültige Werte entsprechen denen von `ssl_min_protocol_version`, ergänzt um einen leeren String, der jede Protokollversion erlaubt. Der Standard ist, jede Version zu erlauben. Das Setzen der maximalen Protokollversion ist hauptsächlich zum Testen nützlich oder wenn eine Komponente Probleme mit einem neueren Protokoll hat.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`ssl_dh_params_file` (`string`)

Gibt den Namen der Datei an, die Diffie-Hellman-Parameter enthält, die für die sogenannten ephemeren DH-Familien von SSL-Ciphers verwendet werden. Der Standardwert ist leer; in diesem Fall werden einkompilierte Standard-DH-Parameter verwendet. Benutzerdefinierte DH-Parameter verringern die Angriffsfläche, falls es einem Angreifer gelingt, die bekannten einkompilierten DH-Parameter zu knacken. Sie können eine eigene DH-Parameterdatei mit dem Befehl `openssl dhparam -out dhparams.pem 2048` erstellen.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`ssl_passphrase_command` (`string`)

Setzt einen externen Befehl, der aufgerufen wird, wenn eine Passphrase zum Entschlüsseln einer SSL-Datei, etwa eines privaten Schlüssels, benötigt wird. Standardmäßig ist dieser Parameter leer, was bedeutet, dass der eingebaute Eingabemechanismus verwendet wird.

Der Befehl muss die Passphrase auf die Standardausgabe schreiben und mit Code `0` beenden. Im Parameterwert wird `%p` durch einen Prompt-String ersetzt. (Schreiben Sie `%%` für ein literales `%`.) Beachten Sie, dass der Prompt-String wahrscheinlich Leerraum enthält; achten Sie daher auf ausreichendes Quoting. Ein einzelner Zeilenumbruch am Ende der Ausgabe wird entfernt, falls vorhanden.

Der Befehl muss den Benutzer nicht tatsächlich zur Eingabe einer Passphrase auffordern. Er kann sie aus einer Datei lesen, aus einem Schlüsselbund beziehen oder Ähnliches tun. Es liegt in der Verantwortung des Benutzers, sicherzustellen, dass der gewählte Mechanismus ausreichend sicher ist.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`ssl_passphrase_command_supports_reload` (`boolean`)

Dieser Parameter bestimmt, ob der durch `ssl_passphrase_command` gesetzte Passphrase-Befehl auch während eines erneuten Ladens der Konfiguration aufgerufen wird, wenn eine Schlüsseldatei eine Passphrase benötigt. Wenn dieser Parameter `off` ist (der Standard), wird `ssl_passphrase_command` während eines Reloads ignoriert, und die SSL-Konfiguration wird nicht neu geladen, falls eine Passphrase benötigt wird. Diese Einstellung ist für einen Befehl geeignet, der zum Prompting ein TTY benötigt, das bei laufendem Server möglicherweise nicht verfügbar ist. Diesen Parameter auf `on` zu setzen, kann passend sein, wenn die Passphrase zum Beispiel aus einer Datei bezogen wird.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

## 19.4. Ressourcenverbrauch

### 19.4.1. Speicher

`shared_buffers` (`integer`)

Setzt die Speichermenge, die der Datenbankserver für Shared-Memory-Puffer verwendet. Der Standardwert beträgt typischerweise 128 Megabyte (`128MB`), kann aber niedriger sein, wenn die Kernel-Einstellungen dies nicht unterstützen (wie während `initdb` ermittelt). Diese Einstellung muss mindestens 128 Kilobyte betragen. Für gute Performance sind jedoch normalerweise deutlich höhere Einstellungen als das Minimum nötig. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. (Nicht standardmäßige Werte von `BLCKSZ` ändern den Mindestwert.) Dieser Parameter kann nur beim Serverstart gesetzt werden.

Wenn Sie einen dedizierten Datenbankserver mit 1 GB oder mehr RAM haben, ist ein sinnvoller Startwert für `shared_buffers` 25 % des Arbeitsspeichers im System. Es gibt Workloads, bei denen noch größere Einstellungen für `shared_buffers` wirksam sind; da PostgreSQL jedoch auch auf den Betriebssystem-Cache angewiesen ist, ist es unwahrscheinlich, dass eine Zuweisung von mehr als 40 % des RAM an `shared_buffers` besser funktioniert als eine kleinere Menge. Größere Einstellungen für `shared_buffers` erfordern normalerweise eine entsprechende Erhöhung von `max_wal_size`, um das Schreiben großer Mengen neuer oder geänderter Daten über einen längeren Zeitraum zu verteilen.

Auf Systemen mit weniger als 1 GB RAM ist ein kleinerer Prozentsatz des RAM angemessen, damit ausreichend Platz für das Betriebssystem bleibt.

`huge_pages` (`enum`)

Steuert, ob Huge Pages für den Hauptbereich des Shared Memory angefordert werden. Gültige Werte sind `try` (der Standard), `on` und `off`. Dieser Parameter kann nur beim Serverstart gesetzt werden. Wenn `huge_pages` auf `try` gesetzt ist, versucht der Server, Huge Pages anzufordern, fällt aber auf den Standard zurück, wenn dies fehlschlägt. Bei `on` verhindert ein Fehlschlag beim Anfordern von Huge Pages den Serverstart. Bei `off` werden keine Huge Pages angefordert. Der tatsächliche Zustand von Huge Pages wird durch die Servervariable `huge_pages_status` angezeigt.

Derzeit wird diese Einstellung nur unter Linux und Windows unterstützt. Auf anderen Systemen wird die Einstellung ignoriert, wenn sie auf `try` gesetzt ist. Unter Linux wird sie nur unterstützt, wenn `shared_memory_type` auf `mmap` gesetzt ist (der Standard).

Die Verwendung von Huge Pages führt zu kleineren Page Tables und weniger CPU-Zeit für die Speicherverwaltung, was die Performance erhöht. Weitere Details zur Verwendung von Huge Pages unter Linux finden Sie in [Abschnitt 18.4.5](18_Servereinrichtung_und_Betrieb.md#1845-linux-huge-pages).

Unter Windows sind Huge Pages als Large Pages bekannt. Um sie zu verwenden, müssen Sie dem Windows-Benutzerkonto, das PostgreSQL ausführt, das Benutzerrecht "Lock pages in memory" zuweisen. Sie können das Windows-Gruppenrichtlinienwerkzeug (`gpedit.msc`) verwenden, um das Benutzerrecht "Lock pages in memory" zuzuweisen. Um den Datenbankserver auf der Eingabeaufforderung als eigenständigen Prozess und nicht als Windows-Dienst zu starten, muss die Eingabeaufforderung als Administrator ausgeführt werden, oder die Benutzerkontensteuerung (User Access Control, UAC) muss deaktiviert sein. Wenn UAC aktiviert ist, entzieht die normale Eingabeaufforderung beim Start das Benutzerrecht "Lock pages in memory".

Beachten Sie, dass diese Einstellung nur den Hauptbereich des Shared Memory betrifft. Betriebssysteme wie Linux, FreeBSD und Illumos können Huge Pages (auch als "Super Pages" oder "Large Pages" bekannt) auch automatisch für normale Speicherzuweisungen verwenden, ohne ausdrückliche Anforderung durch PostgreSQL. Unter Linux heißt dies "Transparent Huge Pages" (THP). Diese Funktion hat bei einigen Benutzern auf manchen Linux-Versionen bekanntermaßen zu Performance-Einbußen mit PostgreSQL geführt, daher wird ihre Verwendung derzeit nicht empfohlen (anders als die explizite Verwendung von `huge_pages`).

`huge_page_size` (`integer`)

Steuert die Größe von Huge Pages, wenn diese mit `huge_pages` aktiviert sind. Der Standardwert ist null (`0`). Bei `0` wird die Standardgröße für Huge Pages auf dem System verwendet. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Zu den auf modernen 64-Bit-Serverarchitekturen häufig verfügbaren Page-Größen gehören: `2MB` und `1GB` (Intel und AMD), `16MB` und `16GB` (IBM POWER) sowie `64kB`, `2MB`, `32MB` und `1GB` (ARM). Weitere Informationen zu Verwendung und Unterstützung finden Sie in [Abschnitt 18.4.5](18_Servereinrichtung_und_Betrieb.md#1845-linux-huge-pages).

Nicht standardmäßige Einstellungen werden derzeit nur unter Linux unterstützt.

`temp_buffers` (`integer`)

Setzt die maximale Speichermenge, die innerhalb jeder Datenbanksitzung für temporäre Puffer verwendet wird. Dies sind sitzungslokale Puffer, die nur für den Zugriff auf temporäre Tabellen verwendet werden. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert beträgt acht Megabyte (`8MB`). (Wenn `BLCKSZ` nicht `8kB` beträgt, skaliert der Standardwert proportional dazu.) Diese Einstellung kann innerhalb einzelner Sitzungen geändert werden, jedoch nur vor der ersten Verwendung temporärer Tabellen innerhalb der Sitzung; spätere Versuche, den Wert zu ändern, haben keine Auswirkung auf diese Sitzung.

Eine Sitzung weist temporäre Puffer bei Bedarf bis zu der durch `temp_buffers` vorgegebenen Grenze zu. Die Kosten einer großen Einstellung in Sitzungen, die tatsächlich nicht viele temporäre Puffer benötigen, bestehen nur aus einem Buffer-Deskriptor oder etwa 64 Bytes pro Erhöhung von `temp_buffers`. Wird ein Puffer jedoch tatsächlich verwendet, werden zusätzlich 8192 Bytes dafür verbraucht (oder allgemein `BLCKSZ` Bytes).

`max_prepared_transactions` (`integer`)

Setzt die maximale Anzahl von Transaktionen, die gleichzeitig im Zustand "prepared" sein können (siehe `PREPARE TRANSACTION`). Wird dieser Parameter auf null gesetzt (der Standard), ist die Prepared-Transaction-Funktion deaktiviert. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Wenn Sie nicht planen, Prepared Transactions zu verwenden, sollte dieser Parameter auf null gesetzt werden, um die versehentliche Erstellung vorbereiteter Transaktionen zu verhindern. Wenn Sie Prepared Transactions verwenden, sollten Sie `max_prepared_transactions` wahrscheinlich mindestens so groß wie `max_connections` setzen, damit jede Sitzung eine vorbereitete Transaktion offen haben kann.

Beim Betrieb eines Standby-Servers muss dieser Parameter auf denselben oder einen höheren Wert als auf dem Primärserver gesetzt werden. Andernfalls werden auf dem Standby-Server keine Abfragen erlaubt.

`work_mem` (`integer`)

Setzt die grundlegende maximale Speichermenge, die von einer Abfrageoperation (etwa einer Sortierung oder Hash-Tabelle) verwendet werden darf, bevor in temporäre Dateien auf der Festplatte geschrieben wird. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Der Standardwert beträgt vier Megabyte (`4MB`). Beachten Sie, dass eine komplexe Abfrage mehrere Sortier- und Hash-Operationen gleichzeitig ausführen kann, wobei jede Operation im Allgemeinen so viel Speicher verwenden darf, wie dieser Wert angibt, bevor sie beginnt, Daten in temporäre Dateien zu schreiben. Außerdem können mehrere laufende Sitzungen solche Operationen gleichzeitig ausführen. Daher kann der insgesamt verwendete Speicher ein Vielfaches von `work_mem` betragen; dies muss bei der Wahl des Werts berücksichtigt werden. Sortieroperationen werden für `ORDER BY`, `DISTINCT` und Merge Joins verwendet. Hash-Tabellen werden in Hash Joins, hashbasierter Aggregation, Memoize-Knoten und hashbasierter Verarbeitung von `IN`-Unterabfragen verwendet.

Hashbasierte Operationen reagieren im Allgemeinen empfindlicher auf die Verfügbarkeit von Speicher als entsprechende sortierbasierte Operationen. Die Speichergrenze für eine Hash-Tabelle wird berechnet, indem `work_mem` mit `hash_mem_multiplier` multipliziert wird. Dadurch können hashbasierte Operationen eine Speichermenge verwenden, die über den üblichen Basiswert von `work_mem` hinausgeht.

`hash_mem_multiplier` (`floating point`)

Wird verwendet, um die maximale Speichermenge zu berechnen, die hashbasierte Operationen verwenden dürfen. Die endgültige Grenze wird bestimmt, indem `work_mem` mit `hash_mem_multiplier` multipliziert wird. Der Standardwert ist `2.0`, wodurch hashbasierte Operationen das Doppelte des üblichen Basiswerts von `work_mem` verwenden können.

Erwägen Sie, `hash_mem_multiplier` in Umgebungen zu erhöhen, in denen Query-Operationen regelmäßig auf die Festplatte ausweichen, insbesondere wenn eine einfache Erhöhung von `work_mem` zu Speicherdruck führt (Speicherdruck äußert sich typischerweise in sporadischen Out-of-Memory-Fehlern). Die Standardeinstellung von `2.0` ist bei gemischten Workloads oft wirksam. Höhere Einstellungen im Bereich von `2.0` bis `8.0` oder mehr können in Umgebungen wirksam sein, in denen `work_mem` bereits auf `40MB` oder mehr erhöht wurde.

`maintenance_work_mem` (`integer`)

Gibt die maximale Speichermenge an, die von Wartungsoperationen wie `VACUUM`, `CREATE INDEX` und `ALTER TABLE ADD FOREIGN KEY` verwendet werden darf. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Der Standardwert beträgt 64 Megabyte (`64MB`). Da von einer Datenbanksitzung jeweils nur eine dieser Operationen ausgeführt werden kann und in einer Installation normalerweise nicht viele davon gleichzeitig laufen, kann dieser Wert sicher deutlich größer als `work_mem` gesetzt werden. Größere Einstellungen können die Performance beim Vacuuming und beim Wiederherstellen von Datenbank-Dumps verbessern.

Beachten Sie, dass beim Ausführen von Autovacuum bis zu `autovacuum_max_workers` mal so viel Speicher zugewiesen werden kann; achten Sie daher darauf, den Standardwert nicht zu hoch zu setzen. Es kann sinnvoll sein, dies zu steuern, indem `autovacuum_work_mem` separat gesetzt wird.

`autovacuum_work_mem` (`integer`)

Gibt die maximale Speichermenge an, die von jedem Autovacuum-Worker-Prozess verwendet werden darf. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Der Standardwert ist `-1`, was angibt, dass stattdessen der Wert von `maintenance_work_mem` verwendet werden soll. Die Einstellung hat keine Auswirkung auf das Verhalten von `VACUUM`, wenn es in anderen Kontexten ausgeführt wird. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`vacuum_buffer_usage_limit` (`integer`)

Gibt die Größe der Buffer Access Strategy an, die von den Befehlen `VACUUM` und `ANALYZE` verwendet wird. Eine Einstellung von `0` erlaubt der Operation, eine beliebige Anzahl von `shared_buffers` zu verwenden. Andernfalls liegen gültige Größen zwischen `128 kB` und `16 GB`. Wenn die angegebene Größe 1/8 der Größe von `shared_buffers` überschreiten würde, wird die Größe stillschweigend auf diesen Wert begrenzt. Der Standardwert ist `2MB`. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Dieser Parameter kann jederzeit gesetzt werden. Er kann für `VACUUM` und `ANALYZE` überschrieben werden, wenn die Option `BUFFER_USAGE_LIMIT` übergeben wird. Höhere Einstellungen können `VACUUM` und `ANALYZE` schneller laufen lassen, eine zu große Einstellung kann jedoch dazu führen, dass zu viele andere nützliche Seiten aus den Shared Buffers verdrängt werden.

`logical_decoding_work_mem` (`integer`)

Gibt die maximale Speichermenge an, die von Logical Decoding verwendet werden darf, bevor ein Teil der decodierten Änderungen auf die lokale Festplatte geschrieben wird. Dies begrenzt die Speichermenge, die von logischen Streaming-Replikationsverbindungen verwendet wird. Der Standardwert beträgt 64 Megabyte (`64MB`). Da jede Replikationsverbindung nur einen einzigen Puffer dieser Größe verwendet und eine Installation normalerweise nicht viele solcher Verbindungen gleichzeitig hat (begrenzt durch `max_wal_senders`), kann dieser Wert sicher deutlich höher als `work_mem` gesetzt werden, wodurch weniger decodierte Änderungen auf die Festplatte geschrieben werden.

`commit_timestamp_buffers` (`integer`)

Gibt die Speichermenge an, die zum Zwischenspeichern des Inhalts von `pg_commit_ts` verwendet werden soll (siehe Tabelle 66.1). Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `0`, wodurch `shared_buffers/512` bis zu 1024 Blöcke angefordert werden, aber nicht weniger als 16 Blöcke. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`multixact_member_buffers` (`integer`)

Gibt die Menge an Shared Memory an, die zum Zwischenspeichern des Inhalts von `pg_multixact/members` verwendet werden soll (siehe Tabelle 66.1). Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `32`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`multixact_offset_buffers` (`integer`)

Gibt die Menge an Shared Memory an, die zum Zwischenspeichern des Inhalts von `pg_multixact/offsets` verwendet werden soll (siehe Tabelle 66.1). Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `16`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`notify_buffers` (`integer`)

Gibt die Menge an Shared Memory an, die zum Zwischenspeichern des Inhalts von `pg_notify` verwendet werden soll (siehe Tabelle 66.1). Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `16`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`serializable_buffers` (`integer`)

Gibt die Menge an Shared Memory an, die zum Zwischenspeichern des Inhalts von `pg_serial` verwendet werden soll (siehe Tabelle 66.1). Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `32`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`subtransaction_buffers` (`integer`)

Gibt die Menge an Shared Memory an, die zum Zwischenspeichern des Inhalts von `pg_subtrans` verwendet werden soll (siehe Tabelle 66.1). Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `0`, wodurch `shared_buffers/512` bis zu 1024 Blöcke angefordert werden, aber nicht weniger als 16 Blöcke. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`transaction_buffers` (`integer`)

Gibt die Menge an Shared Memory an, die zum Zwischenspeichern des Inhalts von `pg_xact` verwendet werden soll (siehe Tabelle 66.1). Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `0`, wodurch `shared_buffers/512` bis zu 1024 Blöcke angefordert werden, aber nicht weniger als 16 Blöcke. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`max_stack_depth` (`integer`)

Gibt die maximale sichere Tiefe des Ausführungs-Stacks des Servers an. Die ideale Einstellung für diesen Parameter ist die tatsächliche vom Kernel erzwungene Stackgrößengrenze (wie durch `ulimit -s` oder ein lokales Äquivalent gesetzt), abzüglich einer Sicherheitsreserve von etwa einem Megabyte. Die Sicherheitsreserve ist nötig, weil die Stacktiefe nicht in jeder Routine des Servers geprüft wird, sondern nur in wichtigen potenziell rekursiven Routinen. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Die Standardeinstellung beträgt zwei Megabyte (`2MB`), was konservativ klein ist und wahrscheinlich kein Absturzrisiko birgt. Sie kann jedoch zu klein sein, um die Ausführung komplexer Funktionen zu erlauben. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Wird `max_stack_depth` höher gesetzt als die tatsächliche Kernel-Grenze, kann eine außer Kontrolle geratene rekursive Funktion einen einzelnen Backend-Prozess zum Absturz bringen. Auf Plattformen, auf denen PostgreSQL die Kernel-Grenze bestimmen kann, erlaubt der Server nicht, diese Variable auf einen unsicheren Wert zu setzen. Allerdings stellen nicht alle Plattformen diese Information bereit, daher ist bei der Wahl eines Werts Vorsicht geboten.

`shared_memory_type` (`enum`)

Gibt die Shared-Memory-Implementierung an, die der Server für die Hauptregion des Shared Memory verwenden soll, die PostgreSQLs Shared Buffers und andere gemeinsame Daten enthält. Mögliche Werte sind `mmap` (für anonymes Shared Memory, das mit `mmap` zugewiesen wird), `sysv` (für System-V-Shared-Memory, das über `shmget` zugewiesen wird) und `windows` (für Windows-Shared-Memory). Nicht alle Werte werden auf allen Plattformen unterstützt; die erste unterstützte Option ist der Standard für diese Plattform. Von der Verwendung der Option `sysv`, die auf keiner Plattform der Standard ist, wird im Allgemeinen abgeraten, weil sie typischerweise nicht standardmäßige Kernel-Einstellungen erfordert, um große Zuweisungen zu erlauben (siehe [Abschnitt 18.4.1](18_Servereinrichtung_und_Betrieb.md#1841-shared-memory-und-semaphoren)). Dieser Parameter kann nur beim Serverstart gesetzt werden.

`dynamic_shared_memory_type` (`enum`)

Gibt die Dynamic-Shared-Memory-Implementierung an, die der Server verwenden soll. Mögliche Werte sind `posix` (für POSIX-Shared-Memory, das mit `shm_open` zugewiesen wird), `sysv` (für System-V-Shared-Memory, das über `shmget` zugewiesen wird), `windows` (für Windows-Shared-Memory) und `mmap` (zur Simulation von Shared Memory mit memory-mapped Dateien, die im Datenverzeichnis gespeichert werden). Nicht alle Werte werden auf allen Plattformen unterstützt; die erste unterstützte Option ist normalerweise der Standard für diese Plattform. Von der Verwendung der Option `mmap`, die auf keiner Plattform der Standard ist, wird im Allgemeinen abgeraten, weil das Betriebssystem geänderte Seiten möglicherweise wiederholt auf die Festplatte zurückschreibt und damit die System-I/O-Last erhöht. Sie kann jedoch für Debugging nützlich sein, wenn das Verzeichnis `pg_dynshmem` auf einer RAM-Disk liegt oder wenn andere Shared-Memory-Funktionen nicht verfügbar sind. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`min_dynamic_shared_memory` (`integer`)

Gibt die Speichermenge an, die beim Serverstart zur Verwendung durch parallele Abfragen zugewiesen werden soll. Wenn diese Speicherregion für gleichzeitige Abfragen nicht ausreicht oder erschöpft ist, versuchen neue parallele Abfragen, temporär zusätzliches Shared Memory vom Betriebssystem zuzuweisen, und zwar mit der durch `dynamic_shared_memory_type` konfigurierten Methode; dies kann wegen Verwaltungsaufwand bei der Speicherverwaltung langsamer sein. Speicher, der beim Start mit `min_dynamic_shared_memory` zugewiesen wird, wird auf Betriebssystemen, die dies unterstützen, von der Einstellung `huge_pages` beeinflusst und kann auf Betriebssystemen, auf denen dies automatisch verwaltet wird, eher von größeren Pages profitieren. Der Standardwert ist `0` (keiner). Dieser Parameter kann nur beim Serverstart gesetzt werden.

### 19.4.2. Festplatte

`temp_file_limit` (`integer`)

Gibt die maximale Menge an Festplattenspeicher an, die ein Prozess für temporäre Dateien verwenden darf, etwa temporäre Sortier- und Hash-Dateien oder die Speicherdatei für einen gehaltenen Cursor. Eine Transaktion, die versucht, diese Grenze zu überschreiten, wird abgebrochen. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. `-1` (der Standard) bedeutet keine Begrenzung. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Diese Einstellung begrenzt den gesamten Speicherplatz, der zu einem bestimmten Zeitpunkt von allen temporären Dateien eines bestimmten PostgreSQL-Prozesses verwendet wird. Zu beachten ist, dass Festplattenspeicher, der für explizite temporäre Tabellen verwendet wird, im Gegensatz zu temporären Dateien, die im Hintergrund der Abfrageausführung genutzt werden, nicht auf diese Grenze angerechnet wird.

`file_copy_method` (`enum`)

Gibt die Methode an, die zum Kopieren von Dateien verwendet wird. Mögliche Werte sind `COPY` (Standard) und `CLONE` (wenn Betriebssystemunterstützung verfügbar ist).

Dieser Parameter betrifft:

- `CREATE DATABASE ... STRATEGY=FILE_COPY`

- `ALTER DATABASE ... SET TABLESPACE ...`

`CLONE` verwendet die Systemaufrufe `copy_file_range()` (Linux, FreeBSD) oder `copyfile` (macOS) und gibt dem Kernel dadurch auf manchen Dateisystemen die Möglichkeit, Festplattenblöcke gemeinsam zu nutzen oder Arbeit an tiefere Schichten weiterzugeben.

`max_notify_queue_pages` (`integer`)

Gibt die maximale Anzahl zugewiesener Pages für die `NOTIFY`/`LISTEN`-Queue an. Der Standardwert ist `1048576`. Bei 8-KB-Pages erlaubt dies einen Verbrauch von bis zu 8 GB Festplattenspeicher. Dieser Parameter kann nur beim Serverstart gesetzt werden.

### 19.4.3. Kernel-Ressourcenverbrauch

`max_files_per_process` (`integer`)

Setzt die maximale Anzahl offener Dateien, die jeder Server-Subprozess gleichzeitig öffnen darf; Dateien, die bereits im Postmaster geöffnet sind, werden nicht auf diese Grenze angerechnet. Der Standardwert ist eintausend Dateien.

Wenn der Kernel eine sichere Grenze pro Prozess erzwingt, müssen Sie sich um diese Einstellung nicht kümmern. Auf einigen Plattformen (insbesondere den meisten BSD-Systemen) erlaubt der Kernel einzelnen Prozessen jedoch, deutlich mehr Dateien zu öffnen, als das System tatsächlich unterstützen kann, wenn viele Prozesse gleichzeitig so viele Dateien zu öffnen versuchen. Wenn Fehler wie "Too many open files" auftreten, versuchen Sie, diese Einstellung zu senken. Dieser Parameter kann nur beim Serverstart gesetzt werden.

### 19.4.4. Hintergrundschreiber

Es gibt einen separaten Serverprozess, den sogenannten Background Writer, dessen Aufgabe es ist, "dirty" (neue oder geänderte) Shared Buffers zu schreiben. Wenn die Anzahl sauberer Shared Buffers unzureichend zu sein scheint, schreibt der Background Writer einige Dirty Buffers in das Dateisystem und markiert sie als sauber. Dadurch sinkt die Wahrscheinlichkeit, dass Serverprozesse, die Benutzerabfragen bearbeiten, keine sauberen Puffer finden und Dirty Buffers selbst schreiben müssen. Der Background Writer verursacht jedoch insgesamt eine Nettoerhöhung der I/O-Last, weil eine wiederholt geänderte Page sonst möglicherweise nur einmal pro Checkpoint-Intervall geschrieben würde, während der Background Writer sie im selben Intervall mehrfach schreiben kann, wenn sie wiederholt geändert wird. Die in diesem Unterabschnitt behandelten Parameter können verwendet werden, um dieses Verhalten an lokale Anforderungen anzupassen.

`bgwriter_delay` (`integer`)

Gibt die Verzögerung zwischen Aktivitätsrunden des Background Writers an. In jeder Runde schreibt der Writer eine bestimmte Anzahl Dirty Buffers (steuerbar durch die folgenden Parameter). Danach schläft er für die Dauer von `bgwriter_delay` und wiederholt den Vorgang. Wenn im Buffer Pool keine Dirty Buffers vorhanden sind, geht er allerdings unabhängig von `bgwriter_delay` in einen längeren Schlafzustand. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert beträgt 200 Millisekunden (`200ms`). Beachten Sie, dass auf manchen Systemen die effektive Auflösung von Schlafverzögerungen 10 Millisekunden beträgt; das Setzen von `bgwriter_delay` auf einen Wert, der kein Vielfaches von 10 ist, kann dasselbe Ergebnis haben wie das Setzen auf das nächsthöhere Vielfache von 10. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`bgwriter_lru_maxpages` (`integer`)

In jeder Runde werden höchstens so viele Puffer vom Background Writer geschrieben. Wird dieser Wert auf null gesetzt, wird das Schreiben durch den Background Writer deaktiviert. (Beachten Sie, dass Checkpoints, die von einem separaten, dedizierten Hilfsprozess verwaltet werden, davon nicht betroffen sind.) Der Standardwert beträgt 100 Puffer. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`bgwriter_lru_multiplier` (`floating point`)

Die Anzahl der Dirty Buffers, die in jeder Runde geschrieben werden, basiert auf der Anzahl neuer Puffer, die Serverprozesse in den letzten Runden benötigt haben. Der jüngste durchschnittliche Bedarf wird mit `bgwriter_lru_multiplier` multipliziert, um eine Schätzung der Anzahl von Puffern zu erhalten, die in der nächsten Runde benötigt werden. Dirty Buffers werden geschrieben, bis so viele saubere, wiederverwendbare Puffer verfügbar sind. (Pro Runde werden jedoch höchstens `bgwriter_lru_maxpages` Puffer geschrieben.) Eine Einstellung von `1.0` stellt somit eine "Just-in-time"-Strategie dar, bei der genau die vorhergesagte Anzahl benötigter Puffer geschrieben wird. Größere Werte bieten einen Puffer gegen Bedarfsspitzen, während kleinere Werte absichtlich Schreibvorgänge den Serverprozessen überlassen. Der Standardwert ist `2.0`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`bgwriter_flush_after` (`integer`)

Immer wenn der Background Writer mehr als diese Datenmenge geschrieben hat, wird versucht, das Betriebssystem dazu zu bringen, diese Schreibvorgänge an den darunterliegenden Speicher auszugeben. Dadurch wird die Menge schmutziger Daten im Page Cache des Kernels begrenzt, was die Wahrscheinlichkeit von Stockungen verringert, wenn am Ende eines Checkpoints ein `fsync` ausgeführt wird oder wenn das Betriebssystem Daten im Hintergrund in größeren Batches zurückschreibt. Oft führt dies zu deutlich geringerer Transaktionslatenz, es gibt jedoch auch Fälle, insbesondere bei Workloads, die größer als `shared_buffers`, aber kleiner als der Page Cache des Betriebssystems sind, in denen die Performance schlechter werden kann. Diese Einstellung kann auf manchen Plattformen wirkungslos sein. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der gültige Bereich liegt zwischen `0`, wodurch erzwungenes Writeback deaktiviert wird, und `2MB`. Der Standardwert ist `512kB` unter Linux, andernorts `0`. (Wenn `BLCKSZ` nicht `8kB` beträgt, skalieren Standard- und Maximalwerte proportional dazu.) Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Kleinere Werte von `bgwriter_lru_maxpages` und `bgwriter_lru_multiplier` verringern die zusätzliche I/O-Last, die durch den Background Writer verursacht wird, erhöhen aber die Wahrscheinlichkeit, dass Serverprozesse selbst Schreibvorgänge ausführen müssen, wodurch interaktive Abfragen verzögert werden.

### 19.4.5. I/O

`backend_flush_after` (`integer`)

Immer wenn ein einzelnes Backend mehr als diese Datenmenge geschrieben hat, wird versucht, das Betriebssystem dazu zu bringen, diese Schreibvorgänge an den darunterliegenden Speicher auszugeben. Dadurch wird die Menge schmutziger Daten im Page Cache des Kernels begrenzt, was die Wahrscheinlichkeit von Stockungen verringert, wenn am Ende eines Checkpoints ein `fsync` ausgeführt wird oder wenn das Betriebssystem Daten im Hintergrund in größeren Batches zurückschreibt. Oft führt dies zu deutlich geringerer Transaktionslatenz, es gibt jedoch auch Fälle, insbesondere bei Workloads, die größer als `shared_buffers`, aber kleiner als der Page Cache des Betriebssystems sind, in denen die Performance schlechter werden kann. Diese Einstellung kann auf manchen Plattformen wirkungslos sein. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der gültige Bereich liegt zwischen `0`, wodurch erzwungenes Writeback deaktiviert wird, und `2MB`. Der Standardwert ist `0`, also kein erzwungenes Writeback. (Wenn `BLCKSZ` nicht `8kB` beträgt, skaliert der Maximalwert proportional dazu.)

`effective_io_concurrency` (`integer`)

Setzt die Anzahl gleichzeitiger Storage-I/O-Operationen, von denen PostgreSQL erwartet, dass sie gleichzeitig ausgeführt werden können. Eine Erhöhung dieses Werts erhöht die Anzahl von I/O-Operationen, die eine einzelne PostgreSQL-Sitzung parallel zu starten versucht. Der erlaubte Bereich ist `1` bis `1000` oder `0`, um die Ausgabe asynchroner I/O-Anforderungen zu deaktivieren. Der Standardwert ist `16`.

Höhere Werte wirken sich am stärksten auf Storage mit höherer Latenz aus, bei dem Abfragen andernfalls spürbare I/O-Stockungen erleben, sowie auf Geräte mit hohen IOPS. Unnötig hohe Werte können die I/O-Latenz für alle Abfragen auf dem System erhöhen.

Auf Systemen mit Unterstützung für Prefetch-Hinweise steuert `effective_io_concurrency` außerdem die Prefetch-Distanz.

Dieser Wert kann für Tabellen in einem bestimmten Tablespace überschrieben werden, indem der gleichnamige Tablespace-Parameter gesetzt wird (siehe `ALTER TABLESPACE`).

`maintenance_io_concurrency` (`integer`)

Ähnlich wie `effective_io_concurrency`, wird aber für Wartungsarbeiten verwendet, die im Auftrag vieler Client-Sitzungen ausgeführt werden.

Der Standardwert ist `16`. Dieser Wert kann für Tabellen in einem bestimmten Tablespace überschrieben werden, indem der gleichnamige Tablespace-Parameter gesetzt wird (siehe `ALTER TABLESPACE`).

`io_max_combine_limit` (`integer`)

Steuert die größte I/O-Größe in Operationen, die I/O kombinieren, und begrenzt stillschweigend den vom Benutzer setzbaren Parameter `io_combine_limit`. Dieser Parameter kann nur beim Serverstart gesetzt werden. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Die maximal mögliche Größe hängt vom Betriebssystem und der Blockgröße ab, beträgt aber typischerweise `1MB` unter Unix und `128kB` unter Windows. Der Standardwert ist `128kB`.

`io_combine_limit` (`integer`)

Steuert die größte I/O-Größe in Operationen, die I/O kombinieren. Wenn der Wert höher als der Parameter `io_max_combine_limit` gesetzt wird, wird stattdessen stillschweigend der niedrigere Wert verwendet; möglicherweise müssen also beide erhöht werden, um die I/O-Größe zu steigern. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Die maximal mögliche Größe hängt vom Betriebssystem und der Blockgröße ab, beträgt aber typischerweise `1MB` unter Unix und `128kB` unter Windows. Der Standardwert ist `128kB`.

`io_max_concurrency` (`integer`)

Steuert die maximale Anzahl von I/O-Operationen, die ein Prozess gleichzeitig ausführen kann.

Die Standardeinstellung `-1` wählt eine Anzahl auf Grundlage von `shared_buffers` und der maximalen Anzahl von Prozessen (`max_connections`, `autovacuum_worker_slots`, `max_worker_processes` und `max_wal_senders`), jedoch höchstens `64`.

Dieser Parameter kann nur beim Serverstart gesetzt werden.

`io_method` (`enum`)

Wählt die Methode zur Ausführung asynchroner I/O. Mögliche Werte sind:

- `worker` (asynchrone I/O mit Worker-Prozessen ausführen)

- `io_uring` (asynchrone I/O mit `io_uring` ausführen; erfordert einen Build mit `--with-liburing`/`-Dliburing`)

- `sync` (für asynchrone I/O geeignete Operationen synchron ausführen)

Der Standardwert ist `worker`.

Dieser Parameter kann nur beim Serverstart gesetzt werden.

`io_workers` (`integer`)

Wählt die Anzahl zu verwendender I/O-Worker-Prozesse. Der Standardwert ist `3`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Hat nur eine Wirkung, wenn `io_method` auf `worker` gesetzt ist.

### 19.4.6. Worker-Prozesse

`max_worker_processes` (`integer`)

Setzt die maximale Anzahl von Hintergrundprozessen, die der Cluster unterstützen kann. Dieser Parameter kann nur beim Serverstart gesetzt werden. Der Standardwert ist `8`.

Beim Betrieb eines Standby-Servers muss dieser Parameter auf denselben oder einen höheren Wert als auf dem Primärserver gesetzt werden. Andernfalls werden auf dem Standby-Server keine Abfragen erlaubt.

Wenn Sie diesen Wert ändern, sollten Sie auch `max_parallel_workers`, `max_parallel_maintenance_workers` und `max_parallel_workers_per_gather` anpassen.

`max_parallel_workers_per_gather` (`integer`)

Setzt die maximale Anzahl von Workern, die von einem einzelnen `Gather`- oder `Gather Merge`-Knoten gestartet werden können. Parallele Worker werden aus dem durch `max_worker_processes` eingerichteten Prozesspool entnommen und durch `max_parallel_workers` begrenzt. Beachten Sie, dass die angeforderte Anzahl von Workern zur Laufzeit möglicherweise nicht tatsächlich verfügbar ist. In diesem Fall läuft der Plan mit weniger Workern als erwartet, was ineffizient sein kann. Der Standardwert ist `2`. Wird dieser Wert auf `0` gesetzt, ist parallele Abfrageausführung deaktiviert.

Beachten Sie, dass parallele Abfragen erheblich mehr Ressourcen verbrauchen können als nicht parallele Abfragen, weil jeder Worker-Prozess ein vollständig separater Prozess ist, der ungefähr dieselbe Auswirkung auf das System hat wie eine zusätzliche Benutzersitzung. Dies sollte bei der Wahl eines Werts für diese Einstellung ebenso berücksichtigt werden wie bei der Konfiguration anderer Einstellungen, die die Ressourcennutzung steuern, etwa `work_mem`. Ressourcenbegrenzungen wie `work_mem` werden individuell auf jeden Worker angewendet, was bedeutet, dass die Gesamtnutzung über alle Prozesse hinweg deutlich höher sein kann als normalerweise für einen einzelnen Prozess. Beispielsweise kann eine parallele Abfrage mit 4 Workern bis zu 5-mal so viel CPU-Zeit, Speicher, I/O-Bandbreite usw. verwenden wie eine Abfrage ganz ohne Worker.

Weitere Informationen zu parallelen Abfragen finden Sie in [Kapitel 15](15_Parallele_Abfragen.md).

`max_parallel_maintenance_workers` (`integer`)

Setzt die maximale Anzahl paralleler Worker, die von einem einzelnen Utility-Befehl gestartet werden können. Derzeit sind die parallelen Utility-Befehle, die parallele Worker unterstützen, `CREATE INDEX` beim Aufbau eines B-Tree-, GIN- oder BRIN-Index sowie `VACUUM` ohne Option `FULL`. Parallele Worker werden aus dem durch `max_worker_processes` eingerichteten Prozesspool entnommen und durch `max_parallel_workers` begrenzt. Beachten Sie, dass die angeforderte Anzahl von Workern zur Laufzeit möglicherweise nicht tatsächlich verfügbar ist. In diesem Fall läuft die Utility-Operation mit weniger Workern als erwartet. Der Standardwert ist `2`. Wird dieser Wert auf `0` gesetzt, ist die Verwendung paralleler Worker durch Utility-Befehle deaktiviert.

Beachten Sie, dass parallele Utility-Befehle nicht wesentlich mehr Speicher verbrauchen sollten als entsprechende nicht parallele Operationen. Diese Strategie unterscheidet sich von der parallelen Abfrageausführung, bei der Ressourcenbegrenzungen im Allgemeinen pro Worker-Prozess gelten. Parallele Utility-Befehle behandeln die Ressourcengrenze `maintenance_work_mem` als Grenze, die auf den gesamten Utility-Befehl angewendet wird, unabhängig von der Anzahl paralleler Worker-Prozesse. Parallele Utility-Befehle können jedoch dennoch erheblich mehr CPU-Ressourcen und I/O-Bandbreite verbrauchen.

`max_parallel_workers` (`integer`)

Setzt die maximale Anzahl von Workern, die der Cluster für parallele Operationen unterstützen kann. Der Standardwert ist `8`. Wenn Sie diesen Wert erhöhen oder verringern, sollten Sie auch `max_parallel_maintenance_workers` und `max_parallel_workers_per_gather` anpassen. Beachten Sie außerdem, dass eine Einstellung dieses Werts, die höher als `max_worker_processes` ist, keine Wirkung hat, da parallele Worker aus dem durch diese Einstellung eingerichteten Pool von Worker-Prozessen entnommen werden.

`parallel_leader_participation` (`boolean`)

Erlaubt dem Leader-Prozess, den Query-Plan unter `Gather`- und `Gather Merge`-Knoten auszuführen, statt auf Worker-Prozesse zu warten. Der Standardwert ist `on`. Wird dieser Wert auf `off` gesetzt, verringert sich die Wahrscheinlichkeit, dass Worker blockiert werden, weil der Leader Tupel nicht schnell genug liest; allerdings muss der Leader-Prozess dann warten, bis Worker-Prozesse gestartet sind, bevor die ersten Tupel erzeugt werden können. In welchem Maß der Leader die Performance verbessern oder beeinträchtigen kann, hängt vom Plantyp, der Anzahl der Worker und der Abfragedauer ab.

## 19.5. Write-Ahead Log

Weitere Informationen zur Abstimmung dieser Einstellungen finden Sie in [Abschnitt 28.5](28_Zuverlässigkeit_und_Write_Ahead_Log.md#285-walkonfiguration).

### 19.5.1. Einstellungen

`wal_level` (`enum`)

`wal_level` bestimmt, wie viele Informationen in das WAL geschrieben werden. Der Standardwert ist `replica`; dieser schreibt genug Daten, um WAL-Archivierung und Replikation zu unterstützen, einschließlich der Ausführung schreibgeschützter Abfragen auf einem Standby-Server. `minimal` entfernt alle Protokollierung außer den Informationen, die zur Wiederherstellung nach einem Absturz oder sofortigen Shutdown erforderlich sind. Schließlich fügt `logical` Informationen hinzu, die zur Unterstützung von Logical Decoding nötig sind. Jede Stufe enthält die Informationen, die auf allen niedrigeren Stufen protokolliert werden. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Die Stufe `minimal` erzeugt das geringste WAL-Volumen. Sie protokolliert keine Zeileninformationen für permanente Relationen in Transaktionen, die diese erzeugen oder neu schreiben. Das kann Operationen deutlich beschleunigen (siehe [Abschnitt 14.4.7](14_Hinweise_zur_Performance.md#1447-walarchivierung-und-streamingreplikation-deaktivieren)). Zu den Operationen, die diese Optimierung auslösen, gehören:

```sql
ALTER ... SET TABLESPACE
CLUSTER
CREATE TABLE
REFRESH MATERIALIZED VIEW (without CONCURRENTLY)
REINDEX
TRUNCATE
```

`minimal`-WAL enthält jedoch nicht genügend Informationen für Point-in-Time-Recovery, daher muss `replica` oder höher verwendet werden, um kontinuierliche Archivierung (`archive_mode`) und binäre Streaming-Replikation zu aktivieren. Tatsächlich startet der Server in diesem Modus nicht einmal, wenn `max_wal_senders` ungleich null ist. Beachten Sie, dass eine Änderung von `wal_level` auf `minimal` frühere Basis-Backups für Point-in-Time-Recovery und Standby-Server unbrauchbar macht.

Auf der Stufe `logical` werden dieselben Informationen protokolliert wie bei `replica`, plus Informationen, die zum Extrahieren logischer Änderungsmengen aus dem WAL nötig sind. Die Verwendung der Stufe `logical` erhöht das WAL-Volumen, insbesondere wenn viele Tabellen für `REPLICA IDENTITY FULL` konfiguriert sind und viele `UPDATE`- und `DELETE`-Anweisungen ausgeführt werden.

In Versionen vor 9.6 erlaubte dieser Parameter außerdem die Werte `archive` und `hot_standby`. Diese werden weiterhin akzeptiert, aber auf `replica` abgebildet.

`fsync` (`boolean`)

Wenn dieser Parameter `on` ist, versucht der PostgreSQL-Server sicherzustellen, dass Aktualisierungen physisch auf die Festplatte geschrieben werden, indem er `fsync()`-Systemaufrufe oder verschiedene äquivalente Methoden ausführt (siehe `wal_sync_method`). Dadurch wird sichergestellt, dass der Datenbank-Cluster nach einem Betriebssystem- oder Hardwareabsturz in einen konsistenten Zustand wiederhergestellt werden kann.

Das Ausschalten von `fsync` bringt häufig Performancevorteile, kann bei Stromausfall oder Systemabsturz aber zu nicht wiederherstellbarer Datenbeschädigung führen. Daher ist es nur ratsam, `fsync` auszuschalten, wenn Sie Ihre gesamte Datenbank leicht aus externen Daten neu erstellen können.

Beispiele für sichere Situationen zum Ausschalten von `fsync` sind das anfängliche Laden eines neuen Datenbank-Clusters aus einer Backup-Datei, die Verwendung eines Datenbank-Clusters zur Verarbeitung eines Datenbatches, nach dessen Abschluss die Datenbank verworfen und neu erstellt wird, oder ein schreibgeschützter Datenbankklon, der häufig neu erstellt und nicht für Failover verwendet wird. Hochwertige Hardware allein ist keine ausreichende Begründung dafür, `fsync` auszuschalten.

Für zuverlässige Wiederherstellung beim Wechsel von `fsync` `off` zu `on` ist es notwendig, alle geänderten Puffer im Kernel auf dauerhaften Speicher zu zwingen. Dies kann bei heruntergefahrenem Cluster oder bei eingeschaltetem `fsync` geschehen, indem `initdb --sync-only` ausgeführt wird, `sync` ausgeführt wird, das Dateisystem ausgehängt wird oder der Server neu gestartet wird.

In vielen Situationen kann das Ausschalten von `synchronous_commit` für unkritische Transaktionen einen großen Teil des potenziellen Performancevorteils des Ausschaltens von `fsync` liefern, ohne die damit verbundenen Risiken einer Datenbeschädigung.

`fsync` kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Wenn Sie diesen Parameter ausschalten, sollten Sie außerdem erwägen, `full_page_writes` auszuschalten.

`synchronous_commit` (`enum`)

Gibt an, wie viel WAL-Verarbeitung abgeschlossen sein muss, bevor der Datenbankserver dem Client eine "success"-Meldung zurückgibt. Gültige Werte sind `remote_apply`, `on` (der Standard), `remote_write`, `local` und `off`.

Wenn `synchronous_standby_names` leer ist, sind nur die Einstellungen `on` und `off` sinnvoll; `remote_apply`, `remote_write` und `local` bieten alle dieselbe lokale Synchronisationsstufe wie `on`. Das lokale Verhalten aller Modi außer `off` besteht darin, auf das lokale Flushen des WAL auf die Festplatte zu warten. Im Modus `off` wird nicht gewartet, sodass eine Verzögerung zwischen der Erfolgsmeldung an den Client und dem späteren garantierten Schutz der Transaktion gegen einen Serverabsturz bestehen kann. (Die maximale Verzögerung beträgt das Dreifache von `wal_writer_delay`.) Anders als bei `fsync` erzeugt das Setzen dieses Parameters auf `off` kein Risiko einer Datenbankinkonsistenz: Ein Betriebssystem- oder Datenbankabsturz kann dazu führen, dass einige kürzlich angeblich committed Transaktionen verloren gehen, aber der Datenbankzustand ist derselbe, als wären diese Transaktionen sauber abgebrochen worden. Daher kann das Ausschalten von `synchronous_commit` eine nützliche Alternative sein, wenn Performance wichtiger ist als genaue Gewissheit über die Dauerhaftigkeit einer Transaktion. Weitere Diskussion finden Sie in [Abschnitt 28.4](28_Zuverlässigkeit_und_Write_Ahead_Log.md#284-asynchroner-commit).

Wenn `synchronous_standby_names` nicht leer ist, steuert `synchronous_commit` außerdem, ob Transaktions-Commits darauf warten, dass ihre WAL-Records auf den Standby-Servern verarbeitet werden.

Bei `remote_apply` warten Commits, bis Antworten der aktuellen synchronen Standbys anzeigen, dass sie den Commit-Record der Transaktion empfangen und angewendet haben, sodass er für Abfragen auf den Standbys sichtbar geworden ist, und ihn außerdem auf dauerhaften Speicher auf den Standbys geschrieben haben. Dies verursacht deutlich größere Commit-Verzögerungen als frühere Einstellungen, da auf WAL-Replay gewartet wird. Bei `on` warten Commits, bis Antworten der aktuellen synchronen Standbys anzeigen, dass sie den Commit-Record der Transaktion empfangen und auf dauerhaften Speicher geflusht haben. Dies stellt sicher, dass die Transaktion nicht verloren geht, sofern nicht sowohl der Primärserver als auch alle synchronen Standbys eine Beschädigung ihres Datenbankspeichers erleiden. Bei `remote_write` warten Commits, bis Antworten der aktuellen synchronen Standbys anzeigen, dass sie den Commit-Record der Transaktion empfangen und in ihre Dateisysteme geschrieben haben. Diese Einstellung stellt Datenerhalt sicher, wenn eine Standby-Instanz von PostgreSQL abstürzt, aber nicht, wenn der Standby einen Absturz auf Betriebssystemebene erleidet, weil die Daten nicht notwendigerweise dauerhaften Speicher auf dem Standby erreicht haben. Die Einstellung `local` bewirkt, dass Commits auf lokales Flushen auf die Festplatte warten, aber nicht auf Replikation. Dies ist normalerweise unerwünscht, wenn synchrone Replikation verwendet wird, wird aber der Vollständigkeit halber bereitgestellt.

Dieser Parameter kann jederzeit geändert werden; das Verhalten einer einzelnen Transaktion wird durch die Einstellung bestimmt, die beim Commit wirksam ist. Daher ist es möglich und nützlich, manche Transaktionen synchron und andere asynchron committen zu lassen. Um beispielsweise eine einzelne mehrzeilige Transaktion asynchron committen zu lassen, wenn der Standard das Gegenteil ist, führen Sie innerhalb der Transaktion `SET LOCAL synchronous_commit TO OFF` aus.

Tabelle 19.1 fasst die Fähigkeiten der `synchronous_commit`-Einstellungen zusammen.

Tabelle 19.1. `synchronous_commit`-Modi

| `synchronous_commit`-Einstellung | Lokaler dauerhafter Commit | Standby-Commit dauerhaft nach PG-Absturz | Standby-Commit dauerhaft nach OS-Absturz | Standby-Abfragekonsistenz |
| --- | --- | --- | --- | --- |
| `remote_apply` | x | x | x | x |
| `on` | x | x | x |  |
| `remote_write` | x | x |  |  |
| `local` | x |  |  |  |
| `off` |  |  |  |  |

`wal_sync_method` (`enum`)

Methode, mit der WAL-Aktualisierungen auf die Festplatte erzwungen werden. Wenn `fsync` ausgeschaltet ist, ist diese Einstellung irrelevant, da WAL-Dateiaktualisierungen dann überhaupt nicht erzwungen werden. Mögliche Werte sind:

- `open_datasync` (WAL-Dateien mit der `open()`-Option `O_DSYNC` schreiben)

- `fdatasync` (`fdatasync()` bei jedem Commit aufrufen)

- `fsync` (`fsync()` bei jedem Commit aufrufen)

- `fsync_writethrough` (`fsync()` bei jedem Commit aufrufen und Write-through eines etwaigen Festplatten-Schreibcaches erzwingen)

- `open_sync` (WAL-Dateien mit der `open()`-Option `O_SYNC` schreiben)

Nicht alle diese Auswahlmöglichkeiten sind auf allen Plattformen verfügbar. Der Standard ist die erste Methode in der obigen Liste, die von der Plattform unterstützt wird, außer dass `fdatasync` unter Linux und FreeBSD der Standard ist. Der Standard ist nicht notwendigerweise ideal; es kann nötig sein, diese Einstellung oder andere Aspekte der Systemkonfiguration zu ändern, um eine absturzsichere Konfiguration zu schaffen oder optimale Performance zu erreichen. Diese Aspekte werden in [Abschnitt 28.1](28_Zuverlässigkeit_und_Write_Ahead_Log.md#281-zuverlässigkeit) behandelt. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`full_page_writes` (`boolean`)

Wenn dieser Parameter `on` ist, schreibt der PostgreSQL-Server beim ersten Ändern jeder Festplatten-Page nach einem Checkpoint den gesamten Inhalt dieser Page in das WAL. Dies ist nötig, weil ein Page-Schreibvorgang, der während eines Betriebssystemabsturzes läuft, möglicherweise nur teilweise abgeschlossen wird, was zu einer Page auf der Festplatte führen kann, die eine Mischung aus alten und neuen Daten enthält. Die normalerweise im WAL gespeicherten Änderungsdaten auf Zeilenebene reichen nicht aus, um eine solche Page bei der Wiederherstellung nach dem Absturz vollständig wiederherzustellen. Das Speichern des vollständigen Page-Images garantiert, dass die Page korrekt wiederhergestellt werden kann, allerdings um den Preis, dass mehr Daten in das WAL geschrieben werden müssen. (Da WAL-Replay immer bei einem Checkpoint beginnt, reicht es aus, dies bei der ersten Änderung jeder Page nach einem Checkpoint zu tun. Eine Möglichkeit, die Kosten von Full-Page-Writes zu reduzieren, besteht daher darin, die Checkpoint-Intervallparameter zu erhöhen.)

Das Ausschalten dieses Parameters beschleunigt den Normalbetrieb, kann aber nach einem Systemausfall entweder zu nicht wiederherstellbarer Datenbeschädigung oder zu stiller Datenbeschädigung führen. Die Risiken ähneln denen beim Ausschalten von `fsync`, sind aber geringer; er sollte nur unter denselben Umständen ausgeschaltet werden, die für diesen Parameter empfohlen werden.

Das Ausschalten dieses Parameters beeinflusst die Verwendung der WAL-Archivierung für Point-in-Time-Recovery (PITR) nicht (siehe [Abschnitt 25.3](25_Sicherung_und_Wiederherstellung.md#253-kontinuierliche-archivierung-und-pointintimerecovery-pitr)).

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `on`.

`wal_log_hints` (`boolean`)

Wenn dieser Parameter `on` ist, schreibt der PostgreSQL-Server beim ersten Ändern jeder Festplatten-Page nach einem Checkpoint den gesamten Inhalt dieser Page in das WAL, selbst bei nicht kritischen Änderungen sogenannter Hint Bits.

Wenn Data Checksums aktiviert sind, werden Hint-Bit-Aktualisierungen immer im WAL protokolliert, und diese Einstellung wird ignoriert. Mit dieser Einstellung können Sie testen, wie viel zusätzliche WAL-Protokollierung auftreten würde, wenn Ihre Datenbank Data Checksums aktiviert hätte.

Dieser Parameter kann nur beim Serverstart gesetzt werden. Der Standardwert ist `off`.

`wal_compression` (`enum`)

Dieser Parameter aktiviert die Komprimierung des WAL mit der angegebenen Komprimierungsmethode. Wenn er aktiviert ist, komprimiert der PostgreSQL-Server Full-Page-Images, die in das WAL geschrieben werden (z. B. wenn `full_page_writes` eingeschaltet ist, während eines Basis-Backups usw.). Ein komprimiertes Page-Image wird während des WAL-Replay dekomprimiert. Die unterstützten Methoden sind `pglz`, `lz4` (wenn PostgreSQL mit `--with-lz4` kompiliert wurde) und `zstd` (wenn PostgreSQL mit `--with-zstd` kompiliert wurde). Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Das Aktivieren der Komprimierung kann das WAL-Volumen verringern, ohne das Risiko nicht wiederherstellbarer Datenbeschädigung zu erhöhen, kostet aber zusätzliche CPU-Zeit für die Komprimierung während der WAL-Protokollierung und für die Dekomprimierung während des WAL-Replay.

`wal_init_zero` (`boolean`)

Wenn diese Option auf `on` gesetzt ist (der Standard), werden neue WAL-Dateien mit Nullen gefüllt. Auf manchen Dateisystemen stellt dies sicher, dass Speicherplatz zugewiesen ist, bevor WAL-Records geschrieben werden müssen. Copy-On-Write-Dateisysteme (COW) profitieren jedoch möglicherweise nicht von dieser Technik; daher gibt es die Option, diese unnötige Arbeit zu überspringen. Bei `off` wird beim Erstellen der Datei nur das letzte Byte geschrieben, damit sie die erwartete Größe hat.

`wal_recycle` (`boolean`)

Wenn diese Option auf `on` gesetzt ist (der Standard), werden WAL-Dateien durch Umbenennen wiederverwendet, sodass keine neuen erstellt werden müssen. Auf COW-Dateisystemen kann es schneller sein, neue Dateien zu erstellen; daher gibt es die Option, dieses Verhalten zu deaktivieren.

`wal_buffers` (`integer`)

Die Menge an Shared Memory, die für WAL-Daten verwendet wird, die noch nicht auf die Festplatte geschrieben wurden. Die Standardeinstellung `-1` wählt eine Größe von 1/32 (etwa 3 %) von `shared_buffers`, aber nicht weniger als `64kB` und nicht mehr als die Größe eines WAL-Segments, typischerweise `16MB`. Dieser Wert kann manuell gesetzt werden, wenn die automatische Wahl zu groß oder zu klein ist, aber jeder positive Wert unter `32kB` wird als `32kB` behandelt. Wird dieser Wert ohne Einheit angegeben, wird er als WAL-Blöcke interpretiert, also als `XLOG_BLCKSZ` Bytes, typischerweise `8kB`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Der Inhalt der WAL-Puffer wird bei jedem Transaktions-Commit auf die Festplatte geschrieben, daher bieten extrem große Werte wahrscheinlich keinen erheblichen Vorteil. Wenn dieser Wert jedoch auf mindestens einige Megabyte gesetzt wird, kann dies die Schreibperformance auf einem stark ausgelasteten Server verbessern, auf dem viele Clients gleichzeitig committen. Die durch die Standardeinstellung `-1` ausgewählte automatische Abstimmung sollte in den meisten Fällen vernünftige Ergebnisse liefern.

`wal_writer_delay` (`integer`)

Gibt zeitbezogen an, wie häufig der WAL Writer WAL flusht. Nach dem Flushen von WAL schläft der Writer für die durch `wal_writer_delay` angegebene Zeit, sofern er nicht früher durch eine asynchron committende Transaktion geweckt wird. Wenn der letzte Flush weniger als `wal_writer_delay` zurückliegt und seitdem weniger als `wal_writer_flush_after` an WAL erzeugt wurde, wird WAL nur in das Betriebssystem geschrieben, nicht aber auf die Festplatte geflusht. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert beträgt 200 Millisekunden (`200ms`). Beachten Sie, dass auf manchen Systemen die effektive Auflösung von Schlafverzögerungen 10 Millisekunden beträgt; das Setzen von `wal_writer_delay` auf einen Wert, der kein Vielfaches von 10 ist, kann dasselbe Ergebnis haben wie das Setzen auf das nächsthöhere Vielfache von 10. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`wal_writer_flush_after` (`integer`)

Gibt volumenbezogen an, wie häufig der WAL Writer WAL flusht. Wenn der letzte Flush weniger als `wal_writer_delay` zurückliegt und seitdem weniger als `wal_writer_flush_after` an WAL erzeugt wurde, wird WAL nur in das Betriebssystem geschrieben, nicht aber auf die Festplatte geflusht. Wenn `wal_writer_flush_after` auf `0` gesetzt ist, werden WAL-Daten immer sofort geflusht. Wird dieser Wert ohne Einheit angegeben, wird er als WAL-Blöcke interpretiert, also als `XLOG_BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist `1MB`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`wal_skip_threshold` (`integer`)

Wenn `wal_level` `minimal` ist und eine Transaktion nach dem Erzeugen oder Neuschreiben einer permanenten Relation committet, bestimmt diese Einstellung, wie die neuen Daten dauerhaft gemacht werden. Sind die Daten kleiner als diese Einstellung, werden sie in das WAL-Log geschrieben; andernfalls wird ein `fsync` der betroffenen Dateien verwendet. Abhängig von den Eigenschaften Ihres Speichers kann das Erhöhen oder Senken dieses Werts helfen, wenn solche Commits gleichzeitige Transaktionen verlangsamen. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Der Standardwert beträgt zwei Megabyte (`2MB`).

`commit_delay` (`integer`)

Das Setzen von `commit_delay` fügt eine Zeitverzögerung hinzu, bevor ein WAL-Flush gestartet wird. Dies kann den Durchsatz von Group Commit verbessern, indem eine größere Anzahl von Transaktionen über einen einzigen WAL-Flush committen kann, sofern die Systemlast hoch genug ist, dass zusätzliche Transaktionen innerhalb des angegebenen Intervalls commitbereit werden. Es erhöht jedoch auch die Latenz um bis zu `commit_delay` pro WAL-Flush. Da die Verzögerung verschwendet ist, wenn keine anderen Transaktionen commitbereit werden, wird sie nur ausgeführt, wenn mindestens `commit_siblings` andere Transaktionen aktiv sind, wenn ein Flush gestartet werden soll. Außerdem werden keine Verzögerungen ausgeführt, wenn `fsync` deaktiviert ist. Wird dieser Wert ohne Einheit angegeben, wird er als Mikrosekunden interpretiert. Der Standardwert von `commit_delay` ist null (keine Verzögerung). Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

In PostgreSQL-Versionen vor 9.3 verhielt sich `commit_delay` anders und war deutlich weniger wirksam: Es betraf nur Commits, nicht alle WAL-Flushes, und wartete die gesamte konfigurierte Verzögerung ab, selbst wenn der WAL-Flush früher abgeschlossen war. Seit PostgreSQL 9.3 wartet der erste Prozess, der zum Flushen bereit wird, für das konfigurierte Intervall, während nachfolgende Prozesse nur warten, bis der Leader die Flush-Operation abgeschlossen hat.

`commit_siblings` (`integer`)

Mindestanzahl gleichzeitig offener Transaktionen, die erforderlich ist, bevor die `commit_delay`-Verzögerung ausgeführt wird. Ein größerer Wert macht es wahrscheinlicher, dass während des Verzögerungsintervalls mindestens eine weitere Transaktion commitbereit wird. Der Standardwert ist fünf Transaktionen.

### 19.5.2. Checkpoints

`checkpoint_timeout` (`integer`)

Maximale Zeit zwischen automatischen WAL-Checkpoints. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Der gültige Bereich liegt zwischen 30 Sekunden und einem Tag. Der Standardwert beträgt fünf Minuten (`5min`). Eine Erhöhung dieses Parameters kann die für Crash Recovery benötigte Zeit verlängern. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`checkpoint_completion_target` (`floating point`)

Gibt das Ziel für die Checkpoint-Fertigstellung als Anteil der Gesamtzeit zwischen Checkpoints an. Der Standardwert ist `0.9`; dadurch wird der Checkpoint über fast das gesamte verfügbare Intervall verteilt, was eine ziemlich gleichmäßige I/O-Last ergibt und zugleich etwas Zeit für Checkpoint-Fertigstellungsaufwand lässt. Eine Verringerung dieses Parameters wird nicht empfohlen, weil der Checkpoint dadurch schneller abgeschlossen wird. Dies führt zu einer höheren I/O-Rate während des Checkpoints, gefolgt von einer Phase mit weniger I/O zwischen Checkpoint-Fertigstellung und dem nächsten geplanten Checkpoint. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`checkpoint_flush_after` (`integer`)

Immer wenn während eines Checkpoints mehr als diese Datenmenge geschrieben wurde, wird versucht, das Betriebssystem dazu zu bringen, diese Schreibvorgänge an den darunterliegenden Speicher auszugeben. Dadurch wird die Menge schmutziger Daten im Page Cache des Kernels begrenzt, was die Wahrscheinlichkeit von Stockungen verringert, wenn am Ende des Checkpoints ein `fsync` ausgeführt wird oder wenn das Betriebssystem Daten im Hintergrund in größeren Batches zurückschreibt. Oft führt dies zu deutlich geringerer Transaktionslatenz, es gibt jedoch auch Fälle, insbesondere bei Workloads, die größer als `shared_buffers`, aber kleiner als der Page Cache des Betriebssystems sind, in denen die Performance schlechter werden kann. Diese Einstellung kann auf manchen Plattformen wirkungslos sein. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der gültige Bereich liegt zwischen `0`, wodurch erzwungenes Writeback deaktiviert wird, und `2MB`. Der Standardwert ist `256kB` unter Linux, andernorts `0`. (Wenn `BLCKSZ` nicht `8kB` beträgt, skalieren Standard- und Maximalwerte proportional dazu.) Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`checkpoint_warning` (`integer`)

Schreibt eine Meldung in das Serverlog, wenn durch das Füllen von WAL-Segmentdateien verursachte Checkpoints dichter beieinander liegen als diese Zeitspanne (was darauf hindeutet, dass `max_wal_size` erhöht werden sollte). Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Der Standardwert beträgt 30 Sekunden (`30s`). Null deaktiviert die Warnung. Es werden keine Warnungen erzeugt, wenn `checkpoint_timeout` kleiner als `checkpoint_warning` ist. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`max_wal_size` (`integer`)

Maximale Größe, bis zu der das WAL während automatischer Checkpoints wachsen darf. Dies ist eine weiche Grenze; die WAL-Größe kann `max_wal_size` unter besonderen Umständen überschreiten, etwa bei hoher Last, einem fehlschlagenden `archive_command` oder `archive_library` oder einer hohen Einstellung von `wal_keep_size`. Wird dieser Wert ohne Einheit angegeben, wird er als Megabyte interpretiert. Der Standardwert ist `1 GB`. Eine Erhöhung dieses Parameters kann die für Crash Recovery benötigte Zeit verlängern. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`min_wal_size` (`integer`)

Solange die WAL-Festplattennutzung unter dieser Einstellung bleibt, werden alte WAL-Dateien bei einem Checkpoint immer für zukünftige Verwendung recycelt, statt entfernt zu werden. Dies kann genutzt werden, um sicherzustellen, dass genügend WAL-Speicherplatz reserviert ist, um Spitzen in der WAL-Nutzung zu bewältigen, zum Beispiel beim Ausführen großer Batch-Jobs. Wird dieser Wert ohne Einheit angegeben, wird er als Megabyte interpretiert. Der Standardwert ist `80 MB`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

### 19.5.3. Archivierung

`archive_mode` (`enum`)

Wenn `archive_mode` aktiviert ist, werden abgeschlossene WAL-Segmente durch Setzen von `archive_command` oder `archive_library` an den Archivspeicher gesendet. Neben `off` zum Deaktivieren gibt es zwei Modi: `on` und `always`. Im Normalbetrieb gibt es keinen Unterschied zwischen den beiden Modi, aber bei `always` ist der WAL-Archiver auch während Archive Recovery oder im Standby-Modus aktiviert. Im Modus `always` werden alle aus dem Archiv wiederhergestellten oder per Streaming-Replikation gestreamten Dateien (erneut) archiviert. Details finden Sie in Abschnitt 26.2.9.

`archive_mode` ist eine separate Einstellung von `archive_command` und `archive_library`, damit `archive_command` und `archive_library` geändert werden können, ohne den Archivierungsmodus zu verlassen. Dieser Parameter kann nur beim Serverstart gesetzt werden. `archive_mode` kann nicht aktiviert werden, wenn `wal_level` auf `minimal` gesetzt ist.

`archive_command` (`string`)

Der lokale Shell-Befehl, der ausgeführt wird, um ein abgeschlossenes WAL-Dateisegment zu archivieren. Jedes `%p` im String wird durch den Pfadnamen der zu archivierenden Datei ersetzt, und jedes `%f` nur durch den Dateinamen. (Der Pfadname ist relativ zum Arbeitsverzeichnis des Servers, also zum Datenverzeichnis des Clusters.) Verwenden Sie `%%`, um ein tatsächliches `%`-Zeichen in den Befehl einzubetten. Es ist wichtig, dass der Befehl nur bei Erfolg einen Exit-Status von null zurückgibt. Weitere Informationen finden Sie in Abschnitt 25.3.1.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Er wird nur verwendet, wenn `archive_mode` beim Serverstart aktiviert war und `archive_library` auf einen leeren String gesetzt ist. Wenn sowohl `archive_command` als auch `archive_library` gesetzt sind, wird ein Fehler ausgelöst. Wenn `archive_command` ein leerer String ist (der Standard), während `archive_mode` aktiviert ist (und `archive_library` auf einen leeren String gesetzt ist), wird die WAL-Archivierung vorübergehend deaktiviert, aber der Server sammelt weiterhin WAL-Segmentdateien in der Erwartung, dass bald ein Befehl bereitgestellt wird. Das Setzen von `archive_command` auf einen Befehl, der nichts tut außer `true` zurückzugeben, z. B. `/bin/true` (`REM` unter Windows), deaktiviert die Archivierung effektiv, unterbricht aber auch die Kette von WAL-Dateien, die für Archive Recovery benötigt wird, und sollte daher nur unter ungewöhnlichen Umständen verwendet werden.

`archive_library` (`string`)

Die Bibliothek, die zum Archivieren abgeschlossener WAL-Dateisegmente verwendet wird. Ist sie auf einen leeren String gesetzt (der Standard), ist Archivierung über die Shell aktiviert und `archive_command` wird verwendet. Wenn sowohl `archive_command` als auch `archive_library` gesetzt sind, wird ein Fehler ausgelöst. Andernfalls wird die angegebene Shared Library für die Archivierung verwendet. Der WAL-Archiver-Prozess wird vom Postmaster neu gestartet, wenn sich dieser Parameter ändert. Weitere Informationen finden Sie in Abschnitt 25.3.1 und [Kapitel 49](49_Archivmodule.md).

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`archive_timeout` (`integer`)

`archive_command` oder `archive_library` wird nur für abgeschlossene WAL-Segmente aufgerufen. Wenn Ihr Server also wenig WAL-Traffic erzeugt (oder Ruhephasen hat, in denen dies der Fall ist), kann es eine lange Verzögerung zwischen dem Abschluss einer Transaktion und ihrer sicheren Aufzeichnung im Archivspeicher geben. Um zu begrenzen, wie alt nicht archivierte Daten werden können, können Sie `archive_timeout` setzen, um den Server regelmäßig zum Wechsel auf eine neue WAL-Segmentdatei zu zwingen. Wenn dieser Parameter größer als null ist, wechselt der Server immer dann zu einer neuen Segmentdatei, wenn diese Zeitspanne seit dem letzten Segmentdateiwechsel verstrichen ist und es irgendeine Datenbankaktivität gab, einschließlich eines einzelnen Checkpoints (Checkpoints werden übersprungen, wenn es keine Datenbankaktivität gibt). Beachten Sie, dass archivierte Dateien, die wegen eines erzwungenen Wechsels früh geschlossen werden, weiterhin dieselbe Länge haben wie vollständig gefüllte Dateien. Daher ist es unklug, ein sehr kurzes `archive_timeout` zu verwenden; es bläht Ihren Archivspeicher auf. `archive_timeout`-Einstellungen von etwa einer Minute sind normalerweise vernünftig. Sie sollten Streaming-Replikation statt Archivierung erwägen, wenn Daten schneller vom Primärserver wegkopiert werden sollen. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

### 19.5.4. Recovery

Dieser Abschnitt beschreibt die Einstellungen, die allgemein für Recovery gelten und Crash Recovery, Streaming-Replikation sowie archivbasierte Replikation beeinflussen.

`recovery_prefetch` (`enum`)

Legt fest, ob während der Recovery versucht werden soll, Blöcke vorab zu laden, die im WAL referenziert werden, sich aber noch nicht im Buffer Pool befinden. Gültige Werte sind `off`, `on` und `try` (der Standard). Die Einstellung `try` aktiviert Prefetching nur, wenn das Betriebssystem Unterstützung für Read-Ahead-Hinweise bereitstellt.

Das Vorabladen von Blöcken, die bald benötigt werden, kann bei manchen Workloads die I/O-Wartezeiten während der Recovery verringern. Siehe auch die Einstellungen `wal_decode_buffer_size` und `maintenance_io_concurrency`, welche die Prefetching-Aktivität begrenzen.

`wal_decode_buffer_size` (`integer`)

Eine Grenze dafür, wie weit der Server im WAL vorausschauen kann, um Blöcke zum Vorabladen zu finden. Wird dieser Wert ohne Einheit angegeben, wird er als Bytes interpretiert. Der Standardwert ist `512kB`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

### 19.5.5. Archive Recovery

Dieser Abschnitt beschreibt die Einstellungen, die nur für die Dauer der Recovery gelten. Sie müssen für jede spätere Recovery, die Sie ausführen möchten, zurückgesetzt werden.

"Recovery" umfasst die Verwendung des Servers als Standby oder die Ausführung einer zielgerichteten Recovery. Typischerweise wird der Standby-Modus verwendet, um Hochverfügbarkeit und/oder Leseskalierbarkeit bereitzustellen, während eine zielgerichtete Recovery verwendet wird, um Datenverlust zu beheben.

Um den Server im Standby-Modus zu starten, erstellen Sie im Datenverzeichnis eine Datei namens `standby.signal`. Der Server tritt in die Recovery ein und beendet die Recovery nicht, wenn das Ende des archivierten WAL erreicht ist, sondern versucht weiter, die Recovery fortzusetzen, indem er sich gemäß der Einstellung `primary_conninfo` mit dem sendenden Server verbindet und/oder neue WAL-Segmente mit `restore_command` holt. Für diesen Modus sind die Parameter aus diesem Abschnitt und [Abschnitt 19.6.3](#1963-standbyserver) relevant. Parameter aus [Abschnitt 19.5.6](#1956-recoveryziel) werden ebenfalls angewendet, sind in diesem Modus aber normalerweise nicht nützlich.

Um den Server im Modus für zielgerichtete Recovery zu starten, erstellen Sie im Datenverzeichnis eine Datei namens `recovery.signal`. Wenn sowohl `standby.signal` als auch `recovery.signal` angelegt sind, hat der Standby-Modus Vorrang. Der Modus für zielgerichtete Recovery endet, wenn das archivierte WAL vollständig wiedergegeben wurde oder wenn `recovery_target` erreicht ist. In diesem Modus werden die Parameter aus diesem Abschnitt und aus [Abschnitt 19.5.6](#1956-recoveryziel) verwendet.

`restore_command` (`string`)

Der lokale Shell-Befehl, der ausgeführt wird, um ein archiviertes Segment der WAL-Dateiserie abzurufen. Dieser Parameter ist für Archive Recovery erforderlich, für Streaming-Replikation aber optional. Jedes `%f` im String wird durch den Namen der aus dem Archiv abzurufenden Datei ersetzt, und jedes `%p` durch den Zielpfadnamen der Kopie auf dem Server. (Der Pfadname ist relativ zum aktuellen Arbeitsverzeichnis, d. h. zum Datenverzeichnis des Clusters.) Jedes `%r` wird durch den Namen der Datei ersetzt, die den letzten gültigen Restartpoint enthält. Das ist die früheste Datei, die aufbewahrt werden muss, damit eine Wiederherstellung neu gestartet werden kann; diese Information kann also genutzt werden, um das Archiv auf das Minimum zu kürzen, das für einen Neustart der aktuellen Wiederherstellung benötigt wird. `%r` wird typischerweise nur von Warm-Standby-Konfigurationen verwendet (siehe [Abschnitt 26.2](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#262-logshippingstandbyserver)). Schreiben Sie `%%`, um ein tatsächliches `%`-Zeichen einzubetten.

Es ist wichtig, dass der Befehl nur bei Erfolg einen Exit-Status von null zurückgibt. Der Befehl wird nach Dateinamen gefragt werden, die im Archiv nicht vorhanden sind; in diesem Fall muss er einen Wert ungleich null zurückgeben. Beispiele:

```text
restore_command = 'cp /mnt/server/archivedir/%f "%p"'
restore_command = 'copy "C:\\server\\archivedir\\%f" "%p"' # Windows
```

Eine Ausnahme gilt, wenn der Befehl durch ein Signal beendet wurde (außer `SIGTERM`, das als Teil eines Datenbankserver-Shutdowns verwendet wird) oder durch einen Shell-Fehler (etwa "command not found"); dann bricht die Recovery ab und der Server startet nicht.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`archive_cleanup_command` (`string`)

Dieser optionale Parameter gibt einen Shell-Befehl an, der bei jedem Restartpoint ausgeführt wird. Der Zweck von `archive_cleanup_command` besteht darin, einen Mechanismus zum Bereinigen alter archivierter WAL-Dateien bereitzustellen, die vom Standby-Server nicht mehr benötigt werden. Jedes `%r` wird durch den Namen der Datei ersetzt, die den letzten gültigen Restartpoint enthält. Das ist die früheste Datei, die aufbewahrt werden muss, damit eine Wiederherstellung neu gestartet werden kann; daher können alle Dateien vor `%r` sicher entfernt werden. Diese Information kann genutzt werden, um das Archiv auf das Minimum zu kürzen, das für einen Neustart aus der aktuellen Wiederherstellung benötigt wird. Das Modul `pg_archivecleanup` wird häufig in `archive_cleanup_command` für Single-Standby-Konfigurationen verwendet, zum Beispiel:

```text
archive_cleanup_command = 'pg_archivecleanup /mnt/server/archivedir %r'
```

Beachten Sie jedoch: Wenn mehrere Standby-Server aus demselben Archivverzeichnis wiederherstellen, müssen Sie sicherstellen, dass WAL-Dateien erst gelöscht werden, wenn sie von keinem der Server mehr benötigt werden. `archive_cleanup_command` wird typischerweise in einer Warm-Standby-Konfiguration verwendet (siehe [Abschnitt 26.2](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#262-logshippingstandbyserver)). Schreiben Sie `%%`, um ein tatsächliches `%`-Zeichen in den Befehl einzubetten.

Wenn der Befehl einen Exit-Status ungleich null zurückgibt, wird eine Warnmeldung ins Log geschrieben. Eine Ausnahme gilt, wenn der Befehl durch ein Signal oder durch einen Shell-Fehler (etwa "command not found") beendet wurde; dann wird ein fataler Fehler ausgelöst.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`recovery_end_command` (`string`)

Dieser Parameter gibt einen Shell-Befehl an, der genau einmal am Ende der Recovery ausgeführt wird. Dieser Parameter ist optional. Der Zweck von `recovery_end_command` besteht darin, einen Mechanismus zur Bereinigung nach Replikation oder Recovery bereitzustellen. Jedes `%r` wird wie bei `archive_cleanup_command` durch den Namen der Datei ersetzt, die den letzten gültigen Restartpoint enthält.

Wenn der Befehl einen Exit-Status ungleich null zurückgibt, wird eine Warnmeldung ins Log geschrieben, und die Datenbank fährt dennoch mit dem Start fort. Eine Ausnahme gilt, wenn der Befehl durch ein Signal oder durch einen Shell-Fehler (etwa "command not found") beendet wurde; dann fährt die Datenbank nicht mit dem Start fort.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

### 19.5.6. Recovery-Ziel

Standardmäßig stellt Recovery bis zum Ende des WAL-Logs wieder her. Die folgenden Parameter können verwendet werden, um einen früheren Haltepunkt festzulegen. Höchstens einer von `recovery_target`, `recovery_target_lsn`, `recovery_target_name`, `recovery_target_time` oder `recovery_target_xid` darf verwendet werden; wenn mehr als einer davon in der Konfigurationsdatei angegeben ist, wird ein Fehler ausgelöst. Diese Parameter können nur beim Serverstart gesetzt werden.

`recovery_target = 'immediate'`

Dieser Parameter gibt an, dass die Recovery enden soll, sobald ein konsistenter Zustand erreicht ist, also so früh wie möglich. Beim Wiederherstellen aus einem Online-Backup bedeutet das den Punkt, an dem die Backup-Erstellung endete.

Technisch ist dies ein String-Parameter, aber `'immediate'` ist derzeit der einzige erlaubte Wert.

`recovery_target_name` (`string`)

Dieser Parameter gibt den benannten Wiederherstellungspunkt an (erstellt mit `pg_create_restore_point()`), bis zu dem die Recovery fortschreiten soll.

`recovery_target_time` (`timestamp`)

Dieser Parameter gibt den Zeitstempel an, bis zu dem die Recovery fortschreiten soll. Der genaue Haltepunkt wird außerdem durch `recovery_target_inclusive` beeinflusst.

Der Wert dieses Parameters ist ein Zeitstempel im selben Format, das vom Datentyp `timestamp with time zone` akzeptiert wird, mit der Ausnahme, dass Sie keine Zeitzonenabkürzung verwenden können (es sei denn, die Variable `timezone_abbreviations` wurde zuvor in der Konfigurationsdatei gesetzt). Bevorzugt wird die Verwendung eines numerischen Offsets von UTC; alternativ können Sie einen vollständigen Zeitzonennamen schreiben, z. B. `Europe/Helsinki`, nicht `EEST`.

`recovery_target_xid` (`string`)

Dieser Parameter gibt die Transaktions-ID an, bis zu der die Recovery fortschreiten soll. Beachten Sie, dass Transaktions-IDs zwar bei Transaktionsstart sequenziell vergeben werden, Transaktionen aber in einer anderen numerischen Reihenfolge abgeschlossen werden können. Wiederhergestellt werden die Transaktionen, die vor der angegebenen Transaktion (und optional einschließlich dieser) committed wurden. Der genaue Haltepunkt wird außerdem durch `recovery_target_inclusive` beeinflusst.

`recovery_target_lsn` (`pg_lsn`)

Dieser Parameter gibt die LSN der Write-Ahead-Log-Position an, bis zu der die Recovery fortschreiten soll. Der genaue Haltepunkt wird außerdem durch `recovery_target_inclusive` beeinflusst. Dieser Parameter wird mit dem Systemdatentyp `pg_lsn` geparst.

Die folgenden Optionen präzisieren das Recovery-Ziel weiter und beeinflussen, was geschieht, wenn das Ziel erreicht wird:

`recovery_target_inclusive` (`boolean`)

Gibt an, ob unmittelbar nach dem angegebenen Recovery-Ziel (`on`) oder unmittelbar davor (`off`) angehalten werden soll. Gilt, wenn `recovery_target_lsn`, `recovery_target_time` oder `recovery_target_xid` angegeben ist. Diese Einstellung steuert, ob Transaktionen mit exakt der Ziel-WAL-Position (LSN), der Commit-Zeit bzw. der Transaktions-ID in die Recovery einbezogen werden. Der Standardwert ist `on`.

`recovery_target_timeline` (`string`)

Gibt an, in welche bestimmte Timeline wiederhergestellt werden soll. Der Wert kann eine numerische Timeline-ID oder ein besonderer Wert sein. Der Wert `current` stellt entlang derselben Timeline wieder her, die aktuell war, als das Basis-Backup erstellt wurde. Der Wert `latest` stellt bis zur neuesten im Archiv gefundenen Timeline wieder her; das ist bei einem Standby-Server nützlich. `latest` ist der Standard.

Um eine Timeline-ID hexadezimal anzugeben (zum Beispiel, wenn sie aus einem WAL-Dateinamen oder einer History-Datei extrahiert wurde), stellen Sie `0x` voran. Wenn der WAL-Dateiname beispielsweise `00000011000000A10000004F` lautet, ist die Timeline-ID `0x11` (oder dezimal 17).

Sie müssen diesen Parameter normalerweise nur in komplexen Situationen erneuter Recovery setzen, in denen Sie zu einem Zustand zurückkehren müssen, der selbst nach einer Point-in-Time-Recovery erreicht wurde. Siehe [Abschnitt 25.3.6](25_Sicherung_und_Wiederherstellung.md#2536-zeitleisten) zur Diskussion.

`recovery_target_action` (`enum`)

Gibt an, welche Aktion der Server ausführen soll, sobald das Recovery-Ziel erreicht ist. Der Standardwert ist `pause`, was bedeutet, dass die Recovery angehalten wird. `promote` bedeutet, dass der Recovery-Prozess abgeschlossen wird und der Server beginnt, Verbindungen anzunehmen. Schließlich stoppt `shutdown` den Server nach Erreichen des Recovery-Ziels.

Die beabsichtigte Verwendung der Einstellung `pause` besteht darin, Abfragen gegen die Datenbank ausführen zu können, um zu prüfen, ob dieses Recovery-Ziel der gewünschteste Wiederherstellungspunkt ist. Der angehaltene Zustand kann mit `pg_wal_replay_resume()` fortgesetzt werden (siehe Tabelle 9.99), wodurch die Recovery endet. Wenn dieses Recovery-Ziel nicht der gewünschte Haltepunkt ist, fahren Sie den Server herunter, ändern Sie die Recovery-Zieleinstellungen auf ein späteres Ziel und starten Sie neu, um die Recovery fortzusetzen.

Die Einstellung `shutdown` ist nützlich, um die Instanz exakt am gewünschten Replay-Punkt bereitzuhalten. Die Instanz kann weiterhin weitere WAL-Records wiedergeben (und muss beim nächsten Start tatsächlich WAL-Records seit dem letzten Checkpoint wiedergeben).

Beachten Sie: Da `recovery.signal` nicht entfernt wird, wenn `recovery_target_action` auf `shutdown` gesetzt ist, endet jeder spätere Start mit sofortigem Shutdown, sofern die Konfiguration nicht geändert oder die Datei `recovery.signal` manuell entfernt wird.

Diese Einstellung hat keine Wirkung, wenn kein Recovery-Ziel gesetzt ist. Wenn `hot_standby` nicht aktiviert ist, wirkt die Einstellung `pause` genauso wie `shutdown`. Wenn das Recovery-Ziel erreicht wird, während eine Promotion läuft, wirkt die Einstellung `pause` genauso wie `promote`.

In jedem Fall fährt der Server mit einem fatalen Fehler herunter, wenn ein Recovery-Ziel konfiguriert ist, die Archive Recovery aber endet, bevor das Ziel erreicht wird.

### 19.5.7. WAL-Zusammenfassung

Diese Einstellungen steuern die WAL-Zusammenfassung, eine Funktion, die aktiviert sein muss, um ein inkrementelles Backup auszuführen.

`summarize_wal` (`boolean`)

Aktiviert den WAL-Summarizer-Prozess. Beachten Sie, dass WAL-Zusammenfassung sowohl auf einem Primärserver als auch auf einem Standby aktiviert werden kann. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `off`.

Der Server kann nicht mit `summarize_wal=on` gestartet werden, wenn `wal_level` auf `minimal` gesetzt ist. Wenn `summarize_wal=on` nach dem Serverstart konfiguriert wird, während `wal_level=minimal` ist, läuft der Summarizer zwar, weigert sich aber, Summary-Dateien für WAL zu erzeugen, das mit `wal_level=minimal` generiert wurde.

`wal_summary_keep_time` (`integer`)

Konfiguriert die Zeitspanne, nach der der WAL-Summarizer alte WAL-Summaries automatisch entfernt. Der Datei-Zeitstempel wird verwendet, um zu bestimmen, welche Dateien alt genug zum Entfernen sind. Typischerweise sollten Sie diesen Wert deutlich höher setzen als die Zeit, die zwischen einem Backup und einem späteren inkrementellen Backup vergehen kann, das davon abhängt. WAL-Summaries müssen für den gesamten Bereich der WAL-Records zwischen dem vorherigen Backup und dem neu erstellten Backup verfügbar sein; andernfalls schlägt das inkrementelle Backup fehl. Wenn dieser Parameter auf null gesetzt ist, werden WAL-Summaries nicht automatisch gelöscht, aber Sie können Dateien sicher manuell entfernen, von denen Sie wissen, dass sie für zukünftige inkrementelle Backups nicht benötigt werden. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Wird dieser Wert ohne Einheit angegeben, wird er als Minuten interpretiert. Der Standardwert ist 10 Tage. Wenn `summarize_wal = off` ist, werden vorhandene WAL-Summaries unabhängig vom Wert dieses Parameters nicht entfernt, weil der WAL-Summarizer nicht läuft.

## 19.6. Replikation

Diese Einstellungen steuern das Verhalten der eingebauten Streaming-Replikation (siehe Abschnitt 26.2.5) und der eingebauten logischen Replikation (siehe [Kapitel 29](29_Logische_Replikation.md)).

Bei Streaming-Replikation ist ein Server entweder Primärserver oder Standby-Server. Primärserver können Daten senden, während Standbys immer Empfänger replizierter Daten sind. Wenn kaskadierende Replikation verwendet wird (siehe Abschnitt 26.2.7), können Standby-Server ebenfalls Sender und Empfänger sein. Die Parameter gelten hauptsächlich für sendende und Standby-Server, auch wenn einige Parameter nur auf dem Primärserver Bedeutung haben. Einstellungen dürfen bei Bedarf ohne Probleme im Cluster variieren.

Bei logischer Replikation replizieren Publisher (Server, die `CREATE PUBLICATION` ausführen) Daten an Subscriber (Server, die `CREATE SUBSCRIPTION` ausführen). Server können auch gleichzeitig Publisher und Subscriber sein. Beachten Sie, dass die folgenden Abschnitte Publisher als "Sender" bezeichnen. Weitere Details zu Konfigurationseinstellungen der logischen Replikation finden Sie in [Abschnitt 29.12](29_Logische_Replikation.md#2912-konfigurationseinstellungen).

### 19.6.1. Sendende Server

Diese Parameter können auf jedem Server gesetzt werden, der Replikationsdaten an einen oder mehrere Standby-Server senden soll. Der Primärserver ist immer ein sendender Server, daher müssen diese Parameter immer auf dem Primärserver gesetzt werden. Rolle und Bedeutung dieser Parameter ändern sich nicht, nachdem ein Standby zum Primärserver geworden ist.

`max_wal_senders` (`integer`)

Gibt die maximale Anzahl gleichzeitiger Verbindungen von Standby-Servern oder Streaming-Base-Backup-Clients an (d. h. die maximale Anzahl gleichzeitig laufender WAL-Sender-Prozesse). Der Standardwert ist 10. Der Wert 0 bedeutet, dass Replikation deaktiviert ist. Ein abrupter Verbindungsabbruch eines Streaming-Clients kann einen verwaisten Verbindungsslot zurücklassen, bis ein Timeout erreicht ist; daher sollte dieser Parameter etwas höher gesetzt werden als die maximale Anzahl erwarteter Clients, damit getrennte Clients sich sofort wieder verbinden können. Dieser Parameter kann nur beim Serverstart gesetzt werden. Außerdem muss `wal_level` auf `replica` oder höher gesetzt sein, um Verbindungen von Standby-Servern zu erlauben.

Beim Betrieb eines Standby-Servers müssen Sie diesen Parameter auf denselben oder einen höheren Wert als auf dem Primärserver setzen. Andernfalls sind Abfragen auf dem Standby-Server nicht erlaubt.

`max_replication_slots` (`integer`)

Gibt die maximale Anzahl von Replikationsslots an (siehe Abschnitt 26.2.6), die der Server unterstützen kann. Der Standardwert ist 10. Dieser Parameter kann nur beim Serverstart gesetzt werden. Wird er auf einen kleineren Wert gesetzt als die Anzahl der derzeit existierenden Replikationsslots, verhindert dies den Serverstart. Außerdem muss `wal_level` auf `replica` oder höher gesetzt sein, damit Replikationsslots verwendet werden können.

`wal_keep_size` (`integer`)

Gibt die Mindestgröße alter WAL-Dateien an, die im Verzeichnis `pg_wal` aufbewahrt werden, falls ein Standby-Server sie für Streaming-Replikation abrufen muss. Wenn ein mit dem sendenden Server verbundener Standby-Server um mehr als `wal_keep_size` Megabyte zurückfällt, kann der sendende Server ein WAL-Segment entfernen, das der Standby noch benötigt; in diesem Fall wird die Replikationsverbindung beendet. Nachgelagerte Verbindungen schlagen infolgedessen schließlich ebenfalls fehl. (Der Standby-Server kann sich jedoch durch Abrufen des Segments aus dem Archiv erholen, sofern WAL-Archivierung verwendet wird.)

Dies setzt nur die Mindestgröße der in `pg_wal` vorgehaltenen Segmente; das System muss möglicherweise mehr Segmente für WAL-Archivierung oder zur Wiederherstellung aus einem Checkpoint vorhalten. Wenn `wal_keep_size` null ist (der Standard), hält das System keine zusätzlichen Segmente für Standby-Zwecke vor; die Anzahl alter WAL-Segmente, die Standby-Servern zur Verfügung stehen, ist dann eine Funktion der Position des vorherigen Checkpoints und des Status der WAL-Archivierung. Wird dieser Wert ohne Einheit angegeben, wird er als Megabyte interpretiert. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`max_slot_wal_keep_size` (`integer`)

Gibt die maximale Größe von WAL-Dateien an, die Replikationsslots zum Checkpoint-Zeitpunkt im Verzeichnis `pg_wal` zurückhalten dürfen. Wenn `max_slot_wal_keep_size` `-1` ist (der Standard), dürfen Replikationsslots eine unbegrenzte Menge an WAL-Dateien zurückhalten. Andernfalls kann der Standby, der den Slot verwendet, möglicherweise die Replikation nicht mehr fortsetzen, wenn `restart_lsn` eines Replikationsslots um mehr als die angegebene Größe hinter die aktuelle LSN zurückfällt und erforderliche WAL-Dateien entfernt wurden. Sie können die WAL-Verfügbarkeit von Replikationsslots in `pg_replication_slots` sehen. Wird dieser Wert ohne Einheit angegeben, wird er als Megabyte interpretiert. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`idle_replication_slot_timeout` (`integer`)

Invalidiert Replikationsslots, die länger als diese Dauer inaktiv geblieben sind (also nicht von einer Replikationsverbindung verwendet wurden). Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Der Wert null (der Standard) deaktiviert den Invalidierungsmechanismus für Leerlauf-Timeouts. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Slot-Invalidierung aufgrund eines Leerlauf-Timeouts erfolgt während eines Checkpoints. Da Checkpoints in `checkpoint_timeout`-Intervallen auftreten, kann es eine Verzögerung zwischen dem Überschreiten von `idle_replication_slot_timeout` und der tatsächlichen Slot-Invalidierung beim nächsten Checkpoint geben. Um solche Verzögerungen zu vermeiden, können Benutzer einen Checkpoint erzwingen, damit inaktive Slots umgehend invalidiert werden. Die Dauer der Slot-Inaktivität wird anhand des Werts `pg_replication_slots.inactive_since` des Slots berechnet.

Beachten Sie, dass der Invalidierungsmechanismus für Leerlauf-Timeouts nicht auf Slots anwendbar ist, die kein WAL reservieren, und auch nicht auf Slots auf dem Standby-Server, die vom Primärserver synchronisiert werden (d. h. Standby-Slots mit dem Wert `true` in `pg_replication_slots.synced`). Synchronisierte Slots gelten immer als inaktiv, weil sie keine logische Dekodierung durchführen, um Änderungen zu erzeugen.

`wal_sender_timeout` (`integer`)

Beendet Replikationsverbindungen, die länger als diese Zeitspanne inaktiv sind. Dies ist für den sendenden Server nützlich, um einen Standby-Absturz oder einen Netzwerkausfall zu erkennen. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert beträgt 60 Sekunden. Der Wert null deaktiviert den Timeout-Mechanismus.

Bei einem Cluster, der über mehrere geografische Standorte verteilt ist, bieten unterschiedliche Werte pro Standort mehr Flexibilität bei der Cluster-Verwaltung. Ein kleinerer Wert ist nützlich, um Ausfälle bei einem Standby mit Netzwerkverbindung niedriger Latenz schneller zu erkennen; ein größerer Wert hilft, den Zustand eines Standbys an einem entfernten Standort mit hoher Netzwerklatenz besser einzuschätzen.

`track_commit_timestamp` (`boolean`)

Zeichnet die Commit-Zeit von Transaktionen auf. Dieser Parameter kann nur beim Serverstart gesetzt werden. Der Standardwert ist `off`.

`synchronized_standby_slots` (`string`)

Eine durch Kommas getrennte Liste von Slot-Namen von Streaming-Replikations-Standby-Servern, auf die logische WAL-Sender-Prozesse warten. Logische WAL-Sender-Prozesse senden dekodierte Änderungen erst dann an Plugins, nachdem die angegebenen Replikationsslots den Empfang des WAL bestätigt haben. Dies garantiert, dass Failover-Slots der logischen Replikation Änderungen erst dann verbrauchen, wenn diese Änderungen von den entsprechenden physischen Standbys empfangen und geflusht wurden. Wenn eine logische Replikationsverbindung nach der Promotion eines Standbys auf einen physischen Standby wechseln soll, sollte der physische Replikationsslot für diesen Standby hier aufgeführt werden. Beachten Sie, dass logische Replikation nicht fortschreitet, wenn die in `synchronized_standby_slots` angegebenen Slots nicht existieren oder invalidiert wurden. Außerdem blockieren die Replikationsverwaltungsfunktionen `pg_replication_slot_advance`, `pg_logical_slot_get_changes` und `pg_logical_slot_peek_changes`, wenn sie mit logischen Failover-Slots verwendet werden, bis alle in `synchronized_standby_slots` angegebenen physischen Slots den WAL-Empfang bestätigt haben.

Die Standbys, die den physischen Replikationsslots in `synchronized_standby_slots` entsprechen, müssen `sync_replication_slots = true` konfigurieren, damit sie Änderungen logischer Failover-Slots vom Primärserver empfangen können.

### 19.6.2. Primärserver

Diese Parameter können auf dem Primärserver gesetzt werden, der Replikationsdaten an einen oder mehrere Standby-Server senden soll. Beachten Sie, dass zusätzlich zu diesen Parametern `wal_level` auf dem Primärserver passend gesetzt sein muss; optional kann außerdem WAL-Archivierung aktiviert werden (siehe [Abschnitt 19.5.3](#1953-archivierung)). Die Werte dieser Parameter auf Standby-Servern sind irrelevant, auch wenn Sie sie dort vorsorglich setzen möchten, falls ein Standby später zum Primärserver wird.

`synchronous_standby_names` (`string`)

Gibt eine Liste von Standby-Servern an, die synchrone Replikation unterstützen können, wie in Abschnitt 26.2.8 beschrieben. Es gibt einen oder mehrere aktive synchrone Standbys; Transaktionen, die auf Commit warten, dürfen fortfahren, nachdem diese Standby-Server den Empfang ihrer Daten bestätigt haben. Die synchronen Standbys sind diejenigen, deren Namen in dieser Liste erscheinen und die aktuell verbunden sind sowie Daten in Echtzeit streamen (angezeigt durch den Zustand `streaming` in der View `pg_stat_replication`). Die Angabe von mehr als einem synchronen Standby kann sehr hohe Verfügbarkeit und Schutz vor Datenverlust ermöglichen.

Der Name eines Standby-Servers für diesen Zweck ist die Einstellung `application_name` des Standbys, wie sie in den Verbindungsinformationen des Standbys gesetzt ist. Bei einem physischen Replikations-Standby sollte dies in der Einstellung `primary_conninfo` gesetzt werden; der Standard ist die Einstellung `cluster_name`, falls gesetzt, andernfalls `walreceiver`. Für logische Replikation kann dies in den Verbindungsinformationen der Subscription gesetzt werden; standardmäßig ist es der Name der Subscription. Für andere Verbraucher von Replikationsstreams konsultieren Sie deren Dokumentation.

Dieser Parameter gibt eine Liste von Standby-Servern mit einer der folgenden Syntaxformen an:

```text
[FIRST] num_sync ( standby_name [, ...] )
ANY num_sync ( standby_name [, ...] )
standby_name [, ...]
```

Dabei ist `num_sync` die Anzahl synchroner Standbys, auf deren Antworten Transaktionen warten müssen, und `standby_name` ist der Name eines Standby-Servers. `num_sync` muss ein ganzzahliger Wert größer als null sein. `FIRST` und `ANY` geben die Methode an, mit der synchrone Standbys aus den aufgeführten Servern ausgewählt werden.

Das Schlüsselwort `FIRST` gibt zusammen mit `num_sync` prioritätsbasierte synchrone Replikation an und bewirkt, dass Transaktions-Commits warten, bis ihre WAL-Records auf `num_sync` synchrone Standbys repliziert wurden, die nach Priorität ausgewählt werden. Zum Beispiel bewirkt die Einstellung `FIRST 3 (s1, s2, s3, s4)`, dass jeder Commit auf Antworten von drei Standbys höherer Priorität wartet, die aus den Standby-Servern `s1`, `s2`, `s3` und `s4` ausgewählt werden. Standbys, deren Namen früher in der Liste erscheinen, erhalten höhere Priorität und werden als synchron betrachtet. Andere Standby-Server, die später in dieser Liste erscheinen, stellen potenzielle synchrone Standbys dar. Wenn einer der aktuellen synchronen Standbys aus irgendeinem Grund die Verbindung trennt, wird er sofort durch den Standby mit der nächsthöheren Priorität ersetzt. Das Schlüsselwort `FIRST` ist optional.

Das Schlüsselwort `ANY` gibt zusammen mit `num_sync` quorum-basierte synchrone Replikation an und bewirkt, dass Transaktions-Commits warten, bis ihre WAL-Records auf mindestens `num_sync` der aufgeführten Standbys repliziert wurden. Zum Beispiel bewirkt die Einstellung `ANY 3 (s1, s2, s3, s4)`, dass jeder Commit fortfahren darf, sobald mindestens drei beliebige Standbys aus `s1`, `s2`, `s3` und `s4` antworten.

`FIRST` und `ANY` unterscheiden nicht zwischen Groß- und Kleinschreibung. Wenn diese Schlüsselwörter als Name eines Standby-Servers verwendet werden, muss der jeweilige `standby_name` doppelt gequotet werden.

Die dritte Syntax wurde vor PostgreSQL Version 9.6 verwendet und wird weiterhin unterstützt. Sie entspricht der ersten Syntax mit `FIRST` und `num_sync` gleich 1. Zum Beispiel haben `FIRST 1 (s1, s2)` und `s1, s2` dieselbe Bedeutung: Entweder `s1` oder `s2` wird als synchroner Standby ausgewählt.

Der besondere Eintrag `*` passt auf jeden Standby-Namen.

Es gibt keinen Mechanismus, der Eindeutigkeit von Standby-Namen erzwingt. Bei Duplikaten wird einer der passenden Standbys als höher priorisiert betrachtet, welcher genau das ist, ist jedoch unbestimmt.

Hinweis: Jeder `standby_name` sollte die Form eines gültigen SQL-Bezeichners haben, sofern er nicht `*` ist. Sie können bei Bedarf doppelte Anführungszeichen verwenden. Beachten Sie aber, dass `standby_name` unabhängig von doppelter Quotierung ohne Beachtung der Groß-/Kleinschreibung mit Standby-Application-Namen verglichen wird.

Wenn hier keine Namen synchroner Standbys angegeben sind, ist synchrone Replikation nicht aktiviert, und Transaktions-Commits warten nicht auf Replikation. Dies ist die Standardkonfiguration. Selbst wenn synchrone Replikation aktiviert ist, können einzelne Transaktionen so konfiguriert werden, dass sie nicht auf Replikation warten, indem der Parameter `synchronous_commit` auf `local` oder `off` gesetzt wird.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

### 19.6.3. Standby-Server

Diese Einstellungen steuern das Verhalten eines Standby-Servers, der Replikationsdaten empfangen soll. Ihre Werte auf dem Primärserver sind irrelevant.

`primary_conninfo` (`string`)

Gibt einen Verbindungsstring an, mit dem sich der Standby-Server mit einem sendenden Server verbindet. Dieser String hat das in Abschnitt 32.1.1 beschriebene Format. Wenn in diesem String eine Option nicht angegeben ist, wird die entsprechende Umgebungsvariable geprüft (siehe [Abschnitt 32.15](32_libpq_C_Bibliothek.md#3215-umgebungsvariablen)). Ist auch die Umgebungsvariable nicht gesetzt, werden Standardwerte verwendet.

Der Verbindungsstring sollte den Hostnamen (oder die Adresse) des sendenden Servers angeben sowie die Portnummer, wenn sie nicht dem Standard des Standby-Servers entspricht. Geben Sie außerdem einen Benutzernamen an, der einer ausreichend privilegierten Rolle auf dem sendenden Server entspricht (siehe Abschnitt 26.2.5.1). Ein Passwort muss ebenfalls bereitgestellt werden, wenn der Sender Passwortauthentifizierung verlangt. Es kann im String `primary_conninfo` oder in einer separaten Datei `~/.pgpass` auf dem Standby-Server bereitgestellt werden (verwenden Sie `replication` als Datenbanknamen).

Für die Synchronisierung von Replikationsslots (siehe Abschnitt 47.2.3) ist es außerdem notwendig, einen gültigen `dbname` im String `primary_conninfo` anzugeben. Dieser wird nur für die Slot-Synchronisierung verwendet und beim Streaming ignoriert.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Wenn dieser Parameter geändert wird, während der WAL-Receiver-Prozess läuft, wird diesem Prozess signalisiert, sich zu beenden; anschließend wird erwartet, dass er mit der neuen Einstellung neu startet (außer wenn `primary_conninfo` ein leerer String ist). Diese Einstellung hat keine Wirkung, wenn sich der Server nicht im Standby-Modus befindet.

`primary_slot_name` (`string`)

Gibt optional einen vorhandenen Replikationsslot an, der beim Verbinden mit dem sendenden Server über Streaming-Replikation verwendet wird, um das Entfernen von Ressourcen auf dem vorgelagerten Knoten zu steuern (siehe Abschnitt 26.2.6). Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Wenn dieser Parameter geändert wird, während der WAL-Receiver-Prozess läuft, wird diesem Prozess signalisiert, sich zu beenden; anschließend wird erwartet, dass er mit der neuen Einstellung neu startet. Diese Einstellung hat keine Wirkung, wenn `primary_conninfo` nicht gesetzt ist oder sich der Server nicht im Standby-Modus befindet.

`hot_standby` (`boolean`)

Gibt an, ob Sie während der Recovery Verbindungen herstellen und Abfragen ausführen können, wie in Abschnitt 26.4 beschrieben. Der Standardwert ist `on`. Dieser Parameter kann nur beim Serverstart gesetzt werden. Er wirkt nur während Archive Recovery oder im Standby-Modus.

`max_standby_archive_delay` (`integer`)

Wenn Hot Standby aktiv ist, bestimmt dieser Parameter, wie lange der Standby-Server warten soll, bevor er Standby-Abfragen abbricht, die mit WAL-Einträgen kollidieren, die gerade angewendet werden sollen, wie in Abschnitt 26.4.2 beschrieben. `max_standby_archive_delay` gilt, wenn WAL-Daten aus dem WAL-Archiv gelesen werden (und daher nicht aktuell sind). Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert ist 30 Sekunden. Der Wert `-1` erlaubt dem Standby, unbegrenzt auf den Abschluss kollidierender Abfragen zu warten. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Beachten Sie, dass `max_standby_archive_delay` nicht dasselbe ist wie die maximale Laufzeit einer Abfrage vor dem Abbruch; vielmehr ist es die maximal erlaubte Gesamtzeit, um die Daten eines einzelnen WAL-Segments anzuwenden. Wenn also eine Abfrage früher im WAL-Segment eine erhebliche Verzögerung verursacht hat, haben spätere kollidierende Abfragen deutlich weniger Schonzeit.

`max_standby_streaming_delay` (`integer`)

Wenn Hot Standby aktiv ist, bestimmt dieser Parameter, wie lange der Standby-Server warten soll, bevor er Standby-Abfragen abbricht, die mit WAL-Einträgen kollidieren, die gerade angewendet werden sollen, wie in Abschnitt 26.4.2 beschrieben. `max_standby_streaming_delay` gilt, wenn WAL-Daten per Streaming-Replikation empfangen werden. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert ist 30 Sekunden. Der Wert `-1` erlaubt dem Standby, unbegrenzt auf den Abschluss kollidierender Abfragen zu warten. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Beachten Sie, dass `max_standby_streaming_delay` nicht dasselbe ist wie die maximale Laufzeit einer Abfrage vor dem Abbruch; vielmehr ist es die maximal erlaubte Gesamtzeit, um WAL-Daten anzuwenden, nachdem sie vom Primärserver empfangen wurden. Wenn also eine Abfrage eine erhebliche Verzögerung verursacht hat, haben spätere kollidierende Abfragen deutlich weniger Schonzeit, bis der Standby-Server wieder aufgeholt hat.

`wal_receiver_create_temp_slot` (`boolean`)

Gibt an, ob der WAL-Receiver-Prozess auf der entfernten Instanz einen temporären Replikationsslot erstellen soll, wenn kein permanenter zu verwendender Replikationsslot konfiguriert wurde (über `primary_slot_name`). Der Standardwert ist `off`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Wenn dieser Parameter geändert wird, während der WAL-Receiver-Prozess läuft, wird diesem Prozess signalisiert, sich zu beenden; anschließend wird erwartet, dass er mit der neuen Einstellung neu startet.

`wal_receiver_status_interval` (`integer`)

Gibt die Mindesthäufigkeit an, mit der der WAL-Receiver-Prozess auf dem Standby Informationen über den Replikationsfortschritt an den Primärserver oder den vorgelagerten Standby sendet, wo sie über die View `pg_stat_replication` sichtbar sind. Der Standby meldet die letzte Write-Ahead-Log-Position, die er geschrieben hat, die letzte Position, die er auf die Festplatte geflusht hat, und die letzte Position, die er angewendet hat. Der Wert dieses Parameters ist die maximale Zeitspanne zwischen Berichten. Aktualisierungen werden immer gesendet, wenn sich Schreib- oder Flush-Positionen ändern, oder so häufig, wie durch diesen Parameter angegeben, wenn er auf einen Wert ungleich null gesetzt ist. Es gibt zusätzliche Fälle, in denen Aktualisierungen unter Ignorieren dieses Parameters gesendet werden; zum Beispiel, wenn die Verarbeitung des vorhandenen WAL abgeschlossen ist oder wenn `synchronous_commit` auf `remote_apply` gesetzt ist. Daher kann die Apply-Position leicht hinter der tatsächlichen Position zurückliegen. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Der Standardwert ist 10 Sekunden. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`hot_standby_feedback` (`boolean`)

Gibt an, ob ein Hot Standby Feedback über aktuell auf dem Standby ausgeführte Abfragen an den Primärserver oder vorgelagerten Standby sendet. Dieser Parameter kann verwendet werden, um Abfrageabbrüche durch Cleanup-Records zu vermeiden, kann aber bei manchen Workloads zu Datenbank-Bloat auf dem Primärserver führen. Feedback-Nachrichten werden nicht häufiger als einmal pro `wal_receiver_status_interval` gesendet. Der Standardwert ist `off`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Wenn kaskadierende Replikation verwendet wird, wird das Feedback nach oben weitergegeben, bis es schließlich den Primärserver erreicht. Standbys verwenden empfangenes Feedback nicht anderweitig, außer es nach oben weiterzureichen.

Beachten Sie, dass die Feedback-Nachricht möglicherweise nicht im erforderlichen Intervall gesendet wird, wenn die Uhr auf dem Standby vor- oder zurückgestellt wird. In Extremfällen kann dies über längere Zeiträume zu einem erhöhten Risiko führen, dass tote Zeilen auf dem Primärserver nicht entfernt werden, da der Feedback-Mechanismus auf Zeitstempeln basiert.

`wal_receiver_timeout` (`integer`)

Beendet Replikationsverbindungen, die länger als diese Zeitspanne inaktiv sind. Dies ist für den empfangenden Standby-Server nützlich, um einen Absturz des Primärknotens oder einen Netzwerkausfall zu erkennen. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert beträgt 60 Sekunden. Der Wert null deaktiviert den Timeout-Mechanismus. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`wal_retrieve_retry_interval` (`integer`)

Gibt an, wie lange der Standby-Server warten soll, wenn WAL-Daten aus keiner Quelle verfügbar sind (Streaming-Replikation, lokales `pg_wal` oder WAL-Archiv), bevor er erneut versucht, WAL-Daten abzurufen. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert beträgt 5 Sekunden. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Dieser Parameter ist in Konfigurationen nützlich, in denen ein Knoten in Recovery steuern muss, wie lange er auf neue verfügbare WAL-Daten wartet. Bei Archive Recovery lässt sich die Recovery beispielsweise durch Verringern dieses Werts reaktionsschneller beim Erkennen einer neuen WAL-Datei machen. Auf einem System mit geringer WAL-Aktivität verringert ein höherer Wert die Anzahl der nötigen Zugriffe auf WAL-Archive; das ist zum Beispiel in Cloud-Umgebungen nützlich, in denen die Anzahl der Infrastrukturzugriffe berücksichtigt wird.

Bei logischer Replikation begrenzt dieser Parameter außerdem, wie häufig ein fehlgeschlagener Replication Apply Worker oder Table Synchronization Worker neu gestartet wird.

`recovery_min_apply_delay` (`integer`)

Standardmäßig stellt ein Standby-Server WAL-Records vom sendenden Server so schnell wie möglich wieder her. Es kann nützlich sein, eine zeitverzögerte Kopie der Daten zu haben, um Möglichkeiten zur Korrektur von Datenverlustfehlern zu bieten. Mit diesem Parameter können Sie die Recovery um eine angegebene Zeitspanne verzögern. Wenn Sie diesen Parameter beispielsweise auf `5min` setzen, gibt der Standby jeden Transaktions-Commit erst dann wieder, wenn die Systemzeit auf dem Standby mindestens fünf Minuten nach der vom Primärserver gemeldeten Commit-Zeit liegt. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert ist null, wodurch keine Verzögerung hinzugefügt wird.

Es ist möglich, dass die Replikationsverzögerung zwischen Servern den Wert dieses Parameters überschreitet; in diesem Fall wird keine zusätzliche Verzögerung hinzugefügt. Beachten Sie, dass die Verzögerung zwischen dem WAL-Zeitstempel, wie er auf dem Primärserver geschrieben wurde, und der aktuellen Zeit auf dem Standby berechnet wird. Übertragungsverzögerungen durch Netzwerklatenz oder kaskadierende Replikationskonfigurationen können die tatsächliche Wartezeit erheblich verringern. Wenn die Systemuhren auf Primärserver und Standby nicht synchronisiert sind, kann dies dazu führen, dass die Recovery Records früher als erwartet anwendet; das ist jedoch kein großes Problem, weil sinnvolle Einstellungen dieses Parameters viel größer sind als typische Zeitabweichungen zwischen Servern.

Die Verzögerung tritt nur bei WAL-Records für Transaktions-Commits auf. Andere Records werden so schnell wie möglich wiedergegeben; das ist kein Problem, weil MVCC-Sichtbarkeitsregeln sicherstellen, dass ihre Effekte erst sichtbar sind, wenn der entsprechende Commit-Record angewendet wurde.

Die Verzögerung tritt ein, sobald die Datenbank in Recovery einen konsistenten Zustand erreicht hat, und dauert an, bis der Standby promoted oder getriggert wird. Danach beendet der Standby die Recovery ohne weiteres Warten.

WAL-Records müssen auf dem Standby aufbewahrt werden, bis sie angewendet werden können. Daher führen längere Verzögerungen zu einer größeren Ansammlung von WAL-Dateien und erhöhen den Speicherplatzbedarf für das Verzeichnis `pg_wal` des Standbys.

Dieser Parameter ist für den Einsatz mit Streaming-Replikationsbereitstellungen gedacht; wenn er angegeben ist, wird er jedoch in allen Fällen außer Crash Recovery beachtet. `hot_standby_feedback` wird durch die Verwendung dieser Funktion verzögert, was zu Bloat auf dem Primärserver führen kann; verwenden Sie beides zusammen mit Vorsicht.

Warnung: Synchrone Replikation wird von dieser Einstellung beeinflusst, wenn `synchronous_commit` auf `remote_apply` gesetzt ist; jeder `COMMIT` muss warten, bis er angewendet wurde.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`sync_replication_slots` (`boolean`)

Aktiviert, dass ein physischer Standby logische Failover-Slots vom Primärserver synchronisiert, sodass logische Subscriber nach einem Failover die Replikation vom neuen Primärserver fortsetzen können.

Standardmäßig ist dies deaktiviert. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

### 19.6.4. Subscriber

Diese Einstellungen steuern das Verhalten eines Subscribers der logischen Replikation. Ihre Werte auf dem Publisher sind irrelevant. Weitere Details finden Sie in [Abschnitt 29.12](29_Logische_Replikation.md#2912-konfigurationseinstellungen).

`max_active_replication_origins` (`integer`)

Gibt an, wie viele Replication Origins (siehe [Kapitel 48](48_Fortschritt_der_Replikation_verfolgen.md)) gleichzeitig verfolgt werden können, wodurch effektiv begrenzt wird, wie viele logische Replikations-Subscriptions auf dem Server erstellt werden können. Wird der Wert kleiner gesetzt als die aktuelle Anzahl verfolgter Replication Origins (sichtbar in `pg_replication_origin_status`), verhindert dies den Serverstart. Der Standardwert ist 10. Dieser Parameter kann nur beim Serverstart gesetzt werden. `max_active_replication_origins` muss mindestens auf die Anzahl der Subscriptions gesetzt werden, die dem Subscriber hinzugefügt werden, plus eine Reserve für Tabellensynchronisierung.

`max_logical_replication_workers` (`integer`)

Gibt die maximale Anzahl logischer Replikations-Worker an. Dies umfasst Leader Apply Worker, Parallel Apply Worker und Table Synchronization Worker.

Worker der logischen Replikation werden aus dem durch `max_worker_processes` definierten Pool entnommen.

Der Standardwert ist 4. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`max_sync_workers_per_subscription` (`integer`)

Maximale Anzahl von Synchronisierungs-Workern pro Subscription. Dieser Parameter steuert den Parallelitätsgrad der anfänglichen Datenkopie während der Subscription-Initialisierung oder wenn neue Tabellen hinzugefügt werden.

Derzeit kann es nur einen Synchronisierungs-Worker pro Tabelle geben.

Die Synchronisierungs-Worker werden aus dem durch `max_logical_replication_workers` definierten Pool entnommen.

Der Standardwert ist 2. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`max_parallel_apply_workers_per_subscription` (`integer`)

Maximale Anzahl paralleler Apply Worker pro Subscription. Dieser Parameter steuert den Parallelitätsgrad für das Streaming laufender Transaktionen mit dem Subscription-Parameter `streaming = parallel`.

Die parallelen Apply Worker werden aus dem durch `max_logical_replication_workers` definierten Pool entnommen.

Der Standardwert ist 2. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

## 19.7. Abfrageplanung

### 19.7.1. Planner-Methodenkonfiguration

Diese Konfigurationsparameter bieten eine grobe Möglichkeit, die vom Query Optimizer gewählten Abfragepläne zu beeinflussen. Wenn der vom Optimizer für eine bestimmte Abfrage gewählte Standardplan nicht optimal ist, besteht eine temporäre Lösung darin, einen dieser Konfigurationsparameter zu verwenden, um den Optimizer zur Wahl eines anderen Plans zu zwingen. Bessere Möglichkeiten zur Verbesserung der Qualität der vom Optimizer gewählten Pläne sind das Anpassen der Planner-Kostenkonstanten (siehe [Abschnitt 19.7.2](#1972-plannerkostenkonstanten)), das manuelle Ausführen von `ANALYZE`, das Erhöhen des Werts des Konfigurationsparameters `default_statistics_target` sowie das Erhöhen der für bestimmte Spalten gesammelten Statistikmenge mit `ALTER TABLE SET STATISTICS`.

`enable_async_append` (`boolean`)

Aktiviert oder deaktiviert die Verwendung async-fähiger Append-Plantypen durch den Query Planner. Der Standardwert ist `on`.

`enable_bitmapscan` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Bitmap-Scan-Plantypen durch den Query Planner. Der Standardwert ist `on`.

`enable_distinct_reordering` (`boolean`)

Aktiviert oder deaktiviert die Fähigkeit des Query Planners, `DISTINCT`-Schlüssel so umzuordnen, dass sie zu den Pathkeys des Eingabepfads passen. Der Standardwert ist `on`.

`enable_gathermerge` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Gather-Merge-Plantypen durch den Query Planner. Der Standardwert ist `on`.

`enable_group_by_reordering` (`boolean`)

Steuert, ob der Query Planner einen Plan erzeugt, der `GROUP BY`-Schlüssel in der Reihenfolge der Schlüssel eines Kindknotens des Plans bereitstellt, etwa eines Index-Scans. Ist dies deaktiviert, erzeugt der Query Planner einen Plan mit `GROUP BY`-Schlüsseln, die nur passend zur `ORDER BY`-Klausel sortiert sind, falls eine vorhanden ist. Ist es aktiviert, versucht der Planner, einen effizienteren Plan zu erzeugen. Der Standardwert ist `on`.

`enable_hashagg` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Hash-Aggregations-Plantypen durch den Query Planner. Der Standardwert ist `on`.

`enable_hashjoin` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Hash-Join-Plantypen durch den Query Planner. Der Standardwert ist `on`.

`enable_incremental_sort` (`boolean`)

Aktiviert oder deaktiviert die Verwendung inkrementeller Sortierschritte durch den Query Planner. Der Standardwert ist `on`.

`enable_indexscan` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Index-Scan- und Index-Only-Scan-Plantypen durch den Query Planner. Der Standardwert ist `on`. Siehe auch `enable_indexonlyscan`.

`enable_indexonlyscan` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Index-Only-Scan-Plantypen durch den Query Planner (siehe [Abschnitt 11.9](11_Indizes.md#119-indexonlyscans-und-coveringindizes)). Der Standardwert ist `on`. Die Einstellung `enable_indexscan` muss ebenfalls aktiviert sein, damit der Query Planner Index-Only-Scans berücksichtigt.

`enable_material` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Materialisierung durch den Query Planner. Es ist unmöglich, Materialisierung vollständig zu unterdrücken, aber das Ausschalten dieser Variable hält den Planner davon ab, `Materialize`-Knoten einzufügen, außer in Fällen, in denen sie aus Korrektheitsgründen erforderlich sind. Der Standardwert ist `on`.

`enable_memoize` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Memoize-Plänen durch den Query Planner, um Ergebnisse parameterisierter Scans innerhalb von Nested-Loop-Joins zwischenzuspeichern. Dieser Plantyp erlaubt es, Scans der zugrunde liegenden Pläne zu überspringen, wenn die Ergebnisse für die aktuellen Parameter bereits im Cache liegen. Seltener nachgeschlagene Ergebnisse können aus dem Cache verdrängt werden, wenn mehr Platz für neue Einträge benötigt wird. Der Standardwert ist `on`.

`enable_mergejoin` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Merge-Join-Plantypen durch den Query Planner. Der Standardwert ist `on`.

`enable_nestloop` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Nested-Loop-Join-Plänen durch den Query Planner. Es ist unmöglich, Nested-Loop-Joins vollständig zu unterdrücken, aber das Ausschalten dieser Variable hält den Planner davon ab, einen solchen Join zu verwenden, wenn andere Methoden verfügbar sind. Der Standardwert ist `on`.

`enable_parallel_append` (`boolean`)

Aktiviert oder deaktiviert die Verwendung parallel-fähiger Append-Plantypen durch den Query Planner. Der Standardwert ist `on`.

`enable_parallel_hash` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von Hash-Join-Plantypen mit Parallel Hash durch den Query Planner. Dies hat keine Wirkung, wenn Hash-Join-Pläne nicht ebenfalls aktiviert sind. Der Standardwert ist `on`.

`enable_partition_pruning` (`boolean`)

Aktiviert oder deaktiviert die Fähigkeit des Query Planners, Partitionen einer partitionierten Tabelle aus Abfrageplänen zu entfernen. Dies steuert außerdem die Fähigkeit des Planners, Abfragepläne zu erzeugen, die dem Query Executor erlauben, Partitionen während der Abfrageausführung zu entfernen bzw. zu ignorieren. Der Standardwert ist `on`. Details siehe Abschnitt 5.12.4.

`enable_partitionwise_join` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von partitionwise join durch den Query Planner, wodurch ein Join zwischen partitionierten Tabellen durch Joinen der passenden Partitionen ausgeführt werden kann. Partitionwise Join gilt derzeit nur, wenn die Join-Bedingungen alle Partitionsschlüssel enthalten, die denselben Datentyp haben und eins-zu-eins passende Mengen von Kindpartitionen besitzen müssen. Wenn diese Einstellung aktiviert ist, kann die Anzahl der Knoten im endgültigen Plan, deren Speicherverbrauch durch `work_mem` begrenzt ist, linear mit der Anzahl der gescannten Partitionen steigen. Dies kann während der Abfrageausführung zu einem starken Anstieg des gesamten Speicherverbrauchs führen. Auch die Abfrageplanung wird hinsichtlich Speicher und CPU deutlich teurer. Der Standardwert ist `off`.

`enable_partitionwise_aggregate` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von partitionwise grouping oder aggregation durch den Query Planner, wodurch Gruppierung oder Aggregation auf partitionierten Tabellen getrennt pro Partition ausgeführt werden kann. Wenn die `GROUP BY`-Klausel die Partitionsschlüssel nicht enthält, kann pro Partition nur partielle Aggregation ausgeführt werden, und die Finalisierung muss später erfolgen. Wenn diese Einstellung aktiviert ist, kann die Anzahl der Knoten im endgültigen Plan, deren Speicherverbrauch durch `work_mem` begrenzt ist, linear mit der Anzahl der gescannten Partitionen steigen. Dies kann während der Abfrageausführung zu einem starken Anstieg des gesamten Speicherverbrauchs führen. Auch die Abfrageplanung wird hinsichtlich Speicher und CPU deutlich teurer. Der Standardwert ist `off`.

`enable_presorted_aggregate` (`boolean`)

Steuert, ob der Query Planner einen Plan erzeugt, der Zeilen bereitstellt, die in der für die `ORDER BY`- oder `DISTINCT`-Aggregatfunktionen der Abfrage erforderlichen Reihenfolge vorsortiert sind. Ist dies deaktiviert, erzeugt der Query Planner einen Plan, bei dem der Executor immer eine Sortierung ausführen muss, bevor die Aggregation jeder Aggregatfunktion mit `ORDER BY`- oder `DISTINCT`-Klausel erfolgt. Ist es aktiviert, versucht der Planner, einen effizienteren Plan zu erzeugen, der Eingaben für die Aggregatfunktionen in der für ihre Aggregation erforderlichen Reihenfolge vorsortiert bereitstellt. Der Standardwert ist `on`.

`enable_self_join_elimination` (`boolean`)

Aktiviert oder deaktiviert die Optimierung des Query Planners, die den Abfragebaum analysiert und Self-Joins durch semantisch äquivalente einzelne Scans ersetzt. Berücksichtigt werden nur gewöhnliche Tabellen. Der Standardwert ist `on`.

`enable_seqscan` (`boolean`)

Aktiviert oder deaktiviert die Verwendung sequenzieller Scan-Plantypen durch den Query Planner. Es ist unmöglich, sequenzielle Scans vollständig zu unterdrücken, aber das Ausschalten dieser Variable hält den Planner davon ab, einen solchen Scan zu verwenden, wenn andere Methoden verfügbar sind. Der Standardwert ist `on`.

`enable_sort` (`boolean`)

Aktiviert oder deaktiviert die Verwendung expliziter Sortierschritte durch den Query Planner. Es ist unmöglich, explizite Sortierungen vollständig zu unterdrücken, aber das Ausschalten dieser Variable hält den Planner davon ab, eine solche Sortierung zu verwenden, wenn andere Methoden verfügbar sind. Der Standardwert ist `on`.

`enable_tidscan` (`boolean`)

Aktiviert oder deaktiviert die Verwendung von TID-Scan-Plantypen durch den Query Planner. Der Standardwert ist `on`.

### 19.7.2. Planner-Kostenkonstanten

Die in diesem Abschnitt beschriebenen Kostenvariablen werden auf einer willkürlichen Skala gemessen. Nur ihre relativen Werte sind relevant; daher führt das Skalieren aller Werte nach oben oder unten um denselben Faktor zu keiner Änderung der Planner-Entscheidungen. Standardmäßig basieren diese Kostenvariablen auf den Kosten sequenzieller Page-Abrufe; das heißt, `seq_page_cost` wird konventionell auf `1.0` gesetzt, und die anderen Kostenvariablen werden darauf bezogen. Sie können jedoch eine andere Skala verwenden, wenn Sie möchten, etwa tatsächliche Ausführungszeiten in Millisekunden auf einer bestimmten Maschine.

Hinweis: Leider gibt es keine wohldefinierte Methode zur Bestimmung idealer Werte für die Kostenvariablen. Sie sollten am besten als Durchschnittswerte über die gesamte Mischung von Abfragen behandelt werden, die eine bestimmte Installation erhält. Das bedeutet, dass Änderungen auf Basis nur weniger Experimente sehr riskant sind.

`seq_page_cost` (`floating point`)

Setzt die Schätzung des Planners für die Kosten eines Disk-Page-Abrufs, der Teil einer Reihe sequenzieller Abrufe ist. Der Standardwert ist `1.0`. Dieser Wert kann für Tabellen und Indizes in einem bestimmten Tablespace überschrieben werden, indem der gleichnamige Tablespace-Parameter gesetzt wird (siehe `ALTER TABLESPACE`).

`random_page_cost` (`floating point`)

Setzt die Schätzung des Planners für die Kosten einer nicht sequenziell abgerufenen Disk Page. Der Standardwert ist `4.0`. Dieser Wert kann für Tabellen und Indizes in einem bestimmten Tablespace überschrieben werden, indem der gleichnamige Tablespace-Parameter gesetzt wird (siehe `ALTER TABLESPACE`).

Das Verringern dieses Werts relativ zu `seq_page_cost` bewirkt, dass das System Index-Scans bevorzugt; das Erhöhen lässt Index-Scans relativ teurer erscheinen. Sie können beide Werte gemeinsam erhöhen oder senken, um die Bedeutung von Disk-I/O-Kosten relativ zu CPU-Kosten zu ändern, die durch die folgenden Parameter beschrieben werden.

Zufälliger Zugriff auf dauerhaften Speicher ist normalerweise viel teurer als viermal sequenzieller Zugriff. Es wird jedoch ein niedrigerer Standard verwendet (`4.0`), weil angenommen wird, dass die Mehrheit zufälliger Speicherzugriffe, etwa indexierte Lesezugriffe, im Cache liegt. Außerdem verringert die Latenz netzwerkgebundenen Speichers tendenziell den relativen Overhead zufälliger Zugriffe.

Wenn Sie glauben, dass Caching seltener ist, als der Standardwert widerspiegelt, und die Netzwerklatenz minimal ist, können Sie `random_page_cost` erhöhen, um die tatsächlichen Kosten zufälliger Speicherlesevorgänge besser abzubilden. Speicher mit höheren zufälligen Lesekosten relativ zu sequenziellen, etwa magnetische Festplatten, lässt sich möglicherweise ebenfalls besser mit einem höheren Wert für `random_page_cost` modellieren. Entsprechend kann es angemessen sein, `random_page_cost` zu senken, wenn Ihre Daten wahrscheinlich vollständig im Cache liegen, etwa wenn die Datenbank kleiner ist als der gesamte Serverspeicher, oder wenn die Netzwerklatenz hoch ist.

Tipp: Obwohl das System zulässt, `random_page_cost` kleiner als `seq_page_cost` zu setzen, ist das physikalisch nicht sinnvoll. Die Werte gleichzusetzen ergibt jedoch Sinn, wenn die Datenbank vollständig im RAM gecacht ist, da dann keine Strafe für das Berühren von Pages außerhalb der Reihenfolge anfällt. Außerdem sollten Sie in einer stark gecachten Datenbank beide Werte relativ zu den CPU-Parametern senken, weil die Kosten für das Abrufen einer bereits im RAM befindlichen Page viel kleiner sind als normalerweise.

`cpu_tuple_cost` (`floating point`)

Setzt die Schätzung des Planners für die Kosten der Verarbeitung jeder Zeile während einer Abfrage. Der Standardwert ist `0.01`.

`cpu_index_tuple_cost` (`floating point`)

Setzt die Schätzung des Planners für die Kosten der Verarbeitung jedes Indexeintrags während eines Index-Scans. Der Standardwert ist `0.005`.

`cpu_operator_cost` (`floating point`)

Setzt die Schätzung des Planners für die Kosten der Verarbeitung jedes Operators oder jeder Funktion, die während einer Abfrage ausgeführt wird. Der Standardwert ist `0.0025`.

`parallel_setup_cost` (`floating point`)

Setzt die Schätzung des Planners für die Kosten des Startens paralleler Worker-Prozesse. Der Standardwert ist `1000`.

`parallel_tuple_cost` (`floating point`)

Setzt die Schätzung des Planners für die Kosten der Übertragung eines Tupels von einem parallelen Worker-Prozess zu einem anderen Prozess. Der Standardwert ist `0.1`.

`min_parallel_table_scan_size` (`integer`)

Setzt die Mindestmenge an Tabellendaten, die gescannt werden muss, damit ein paralleler Scan in Betracht gezogen wird. Bei einem parallelen sequenziellen Scan entspricht die Menge der gescannten Tabellendaten immer der Größe der Tabelle; wenn jedoch Indizes verwendet werden, ist die Menge der gescannten Tabellendaten normalerweise geringer. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist 8 Megabyte (`8MB`).

`min_parallel_index_scan_size` (`integer`)

Setzt die Mindestmenge an Indexdaten, die gescannt werden muss, damit ein paralleler Scan in Betracht gezogen wird. Beachten Sie, dass ein paralleler Index-Scan typischerweise nicht den gesamten Index berührt; relevant ist die Anzahl der Pages, von denen der Planner annimmt, dass sie durch den Scan tatsächlich berührt werden. Dieser Parameter wird außerdem verwendet, um zu entscheiden, ob ein bestimmter Index an einem parallelen Vacuum teilnehmen kann. Siehe `VACUUM`. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist 512 Kilobyte (`512kB`).

`effective_cache_size` (`integer`)

Setzt die Annahme des Planners über die effektive Größe des Disk-Caches, die einer einzelnen Abfrage zur Verfügung steht. Dies fließt in Kostenschätzungen für die Verwendung eines Index ein; ein höherer Wert macht Index-Scans wahrscheinlicher, ein niedrigerer Wert macht sequenzielle Scans wahrscheinlicher. Beim Setzen dieses Parameters sollten Sie sowohl die Shared Buffers von PostgreSQL als auch den Teil des Kernel-Disk-Caches berücksichtigen, der für PostgreSQL-Datendateien verwendet wird, auch wenn manche Daten an beiden Orten vorhanden sein können. Berücksichtigen Sie außerdem die erwartete Anzahl gleichzeitiger Abfragen auf verschiedenen Tabellen, da diese den verfügbaren Platz teilen müssen. Dieser Parameter hat keine Wirkung auf die Größe des von PostgreSQL zugewiesenen Shared Memory und reserviert auch keinen Kernel-Disk-Cache; er wird nur zu Schätzungszwecken verwendet. Das System nimmt außerdem nicht an, dass Daten zwischen Abfragen im Disk-Cache verbleiben. Wird dieser Wert ohne Einheit angegeben, wird er als Blöcke interpretiert, also als `BLCKSZ` Bytes, typischerweise `8kB`. Der Standardwert ist 4 Gigabyte (`4GB`). (Wenn `BLCKSZ` nicht `8kB` beträgt, skaliert der Standardwert proportional dazu.)

`jit_above_cost` (`floating point`)

Setzt die Abfragekosten, oberhalb derer JIT-Kompilierung aktiviert wird, sofern sie eingeschaltet ist (siehe [Kapitel 30](30_Just_in_Time_Kompilierung_JIT.md)). Das Ausführen von JIT verursacht Planungszeit, kann aber die Abfrageausführung beschleunigen. Das Setzen auf `-1` deaktiviert JIT-Kompilierung. Der Standardwert ist `100000`.

`jit_inline_above_cost` (`floating point`)

Setzt die Abfragekosten, oberhalb derer JIT-Kompilierung versucht, Funktionen und Operatoren zu inline-en. Inlining erhöht die Planungszeit, kann aber die Ausführungsgeschwindigkeit verbessern. Es ist nicht sinnvoll, diesen Wert kleiner als `jit_above_cost` zu setzen. Das Setzen auf `-1` deaktiviert Inlining. Der Standardwert ist `500000`.

`jit_optimize_above_cost` (`floating point`)

Setzt die Abfragekosten, oberhalb derer JIT-Kompilierung teure Optimierungen anwendet. Eine solche Optimierung erhöht die Planungszeit, kann aber die Ausführungsgeschwindigkeit verbessern. Es ist nicht sinnvoll, diesen Wert kleiner als `jit_above_cost` zu setzen, und es ist wahrscheinlich nicht vorteilhaft, ihn größer als `jit_inline_above_cost` zu setzen. Das Setzen auf `-1` deaktiviert teure Optimierungen. Der Standardwert ist `500000`.

### 19.7.3. Genetischer Query Optimizer

Der genetische Query Optimizer (GEQO) ist ein Algorithmus, der Abfrageplanung mit heuristischer Suche durchführt. Dies verringert die Planungszeit für komplexe Abfragen (solche, die viele Relationen joinen), allerdings um den Preis, dass gelegentlich Pläne entstehen, die schlechter sind als die vom normalen erschöpfenden Suchalgorithmus gefundenen Pläne. Weitere Informationen finden Sie in [Kapitel 61](61_Genetischer_Query_Optimizer.md).

`geqo` (`boolean`)

Aktiviert oder deaktiviert genetische Query-Optimierung. Dies ist standardmäßig eingeschaltet. In der Produktion ist es normalerweise am besten, dies nicht auszuschalten; die Variable `geqo_threshold` bietet feinere Kontrolle über GEQO.

`geqo_threshold` (`integer`)

Verwendet genetische Query-Optimierung zum Planen von Abfragen, die mindestens so viele `FROM`-Einträge betreffen. (Beachten Sie, dass ein `FULL OUTER JOIN`-Konstrukt nur als ein `FROM`-Eintrag zählt.) Der Standardwert ist 12. Für einfachere Abfragen ist es normalerweise am besten, den regulären, erschöpfend suchenden Planner zu verwenden; bei Abfragen mit vielen Tabellen dauert die erschöpfende Suche jedoch zu lange, häufig länger als die Strafe für die Ausführung eines suboptimalen Plans. Daher ist ein Schwellenwert für die Größe der Abfrage eine bequeme Möglichkeit, die Verwendung von GEQO zu steuern.

`geqo_effort` (`integer`)

Steuert den Kompromiss zwischen Planungszeit und Qualität des Abfrageplans in GEQO. Diese Variable muss eine ganze Zahl im Bereich von 1 bis 10 sein. Der Standardwert ist fünf. Größere Werte erhöhen die für die Abfrageplanung aufgewendete Zeit, erhöhen aber auch die Wahrscheinlichkeit, dass ein effizienter Abfrageplan gewählt wird.

`geqo_effort` tut eigentlich nicht direkt etwas; es wird nur verwendet, um die Standardwerte für die anderen Variablen zu berechnen, die das GEQO-Verhalten beeinflussen (unten beschrieben). Wenn Sie möchten, können Sie stattdessen die anderen Parameter von Hand setzen.

`geqo_pool_size` (`integer`)

Steuert die von GEQO verwendete Pool-Größe, also die Anzahl der Individuen in der genetischen Population. Sie muss mindestens zwei betragen, und nützliche Werte liegen typischerweise zwischen 100 und 1000. Wenn sie auf null gesetzt ist (die Standardeinstellung), wird ein geeigneter Wert basierend auf `geqo_effort` und der Anzahl der Tabellen in der Abfrage gewählt.

`geqo_generations` (`integer`)

Steuert die von GEQO verwendete Anzahl von Generationen, also die Anzahl der Iterationen des Algorithmus. Sie muss mindestens eins betragen, und nützliche Werte liegen im selben Bereich wie die Pool-Größe. Wenn sie auf null gesetzt ist (die Standardeinstellung), wird ein geeigneter Wert basierend auf `geqo_pool_size` gewählt.

`geqo_selection_bias` (`floating point`)

Steuert den von GEQO verwendeten Selektionsbias. Der Selektionsbias ist der Selektionsdruck innerhalb der Population. Werte können von `1.50` bis `2.00` reichen; Letzteres ist der Standard.

`geqo_seed` (`floating point`)

Steuert den Anfangswert des Zufallszahlengenerators, den GEQO verwendet, um zufällige Pfade durch den Suchraum der Join-Reihenfolge auszuwählen. Der Wert kann von null (dem Standard) bis eins reichen. Das Variieren des Werts ändert die Menge der untersuchten Join-Pfade und kann dazu führen, dass ein besserer oder schlechterer bester Pfad gefunden wird.

### 19.7.4. Weitere Planner-Optionen

`default_statistics_target` (`integer`)

Setzt das Standard-Statistikziel für Tabellenspalten ohne spaltenspezifisches Ziel, das über `ALTER TABLE SET STATISTICS` gesetzt wurde. Größere Werte erhöhen die für `ANALYZE` benötigte Zeit, können aber die Qualität der Planner-Schätzungen verbessern. Der Standardwert ist 100. Weitere Informationen zur Verwendung von Statistiken durch den PostgreSQL Query Planner finden Sie in [Abschnitt 14.2](14_Hinweise_zur_Performance.md#142-vom-planner-verwendete-statistiken).

`constraint_exclusion` (`enum`)

Steuert die Verwendung von Tabellen-Constraints durch den Query Planner zur Optimierung von Abfragen. Die erlaubten Werte von `constraint_exclusion` sind `on` (Constraints für alle Tabellen prüfen), `off` (Constraints niemals prüfen) und `partition` (Constraints nur für Vererbungs-Kindtabellen und `UNION ALL`-Unterabfragen prüfen). `partition` ist die Standardeinstellung. Sie wird häufig mit traditionellen Vererbungsbäumen verwendet, um die Performance zu verbessern.

Wenn dieser Parameter es für eine bestimmte Tabelle erlaubt, vergleicht der Planner Abfragebedingungen mit den `CHECK`-Constraints der Tabelle und lässt das Scannen von Tabellen aus, bei denen die Bedingungen den Constraints widersprechen. Beispiel:

```sql
CREATE TABLE parent(key integer, ...);
CREATE TABLE child1000(check (key between 1000 and 1999))
 INHERITS(parent);
CREATE TABLE child2000(check (key between 2000 and 2999))
 INHERITS(parent);
...
SELECT * FROM parent WHERE key = 2400;
```

Wenn Constraint Exclusion aktiviert ist, scannt dieses `SELECT` `child1000` überhaupt nicht, was die Performance verbessert.

Derzeit ist Constraint Exclusion standardmäßig nur für Fälle aktiviert, die häufig verwendet werden, um Tabellenpartitionierung über Vererbungsbäume zu implementieren. Das Einschalten für alle Tabellen verursacht zusätzlichen Planungsaufwand, der bei einfachen Abfragen deutlich spürbar ist, und bringt bei einfachen Abfragen meist keinen Nutzen. Wenn Sie keine Tabellen haben, die mit traditioneller Vererbung partitioniert sind, möchten Sie es möglicherweise vollständig ausschalten. (Beachten Sie, dass die entsprechende Funktion für partitionierte Tabellen durch einen separaten Parameter, `enable_partition_pruning`, gesteuert wird.)

Weitere Informationen zur Verwendung von Constraint Exclusion zur Implementierung von Partitionierung finden Sie in Abschnitt 5.12.5.

`cursor_tuple_fraction` (`floating point`)

Setzt die Schätzung des Planners für den Anteil der Zeilen eines Cursors, die abgerufen werden. Der Standardwert ist `0.1`. Kleinere Werte dieser Einstellung beeinflussen den Planner zugunsten von "Fast Start"-Plänen für Cursor, die die ersten wenigen Zeilen schnell abrufen, während das Abrufen aller Zeilen möglicherweise lange dauert. Größere Werte legen mehr Gewicht auf die geschätzte Gesamtzeit. Bei der maximalen Einstellung `1.0` werden Cursor genau wie reguläre Abfragen geplant, wobei nur die geschätzte Gesamtzeit berücksichtigt wird und nicht, wie schnell die ersten Zeilen geliefert werden könnten.

`from_collapse_limit` (`integer`)

Der Planner führt Unterabfragen in übergeordnete Abfragen zusammen, wenn die resultierende `FROM`-Liste höchstens so viele Einträge hätte. Kleinere Werte verringern die Planungszeit, können aber schlechtere Abfragepläne erzeugen. Der Standardwert ist acht. Weitere Informationen finden Sie in [Abschnitt 14.3](14_Hinweise_zur_Performance.md#143-den-planner-mit-expliziten-joinklauseln-steuern).

Das Setzen dieses Werts auf `geqo_threshold` oder höher kann die Verwendung des GEQO-Planners auslösen, was zu nicht optimalen Plänen führen kann. Siehe [Abschnitt 19.7.3](#1973-genetischer-query-optimizer).

`jit` (`boolean`)

Bestimmt, ob JIT-Kompilierung von PostgreSQL verwendet werden darf, sofern verfügbar (siehe [Kapitel 30](30_Just_in_Time_Kompilierung_JIT.md)). Der Standardwert ist `on`.

`join_collapse_limit` (`integer`)

Der Planner schreibt explizite `JOIN`-Konstrukte (außer `FULL JOIN`s) in Listen von `FROM`-Einträgen um, sofern dadurch eine Liste mit höchstens so vielen Einträgen entsteht. Kleinere Werte verringern die Planungszeit, können aber schlechtere Abfragepläne erzeugen.

Standardmäßig ist diese Variable auf denselben Wert wie `from_collapse_limit` gesetzt, was für die meisten Verwendungen passend ist. Das Setzen auf 1 verhindert jede Umordnung expliziter `JOIN`s. Dadurch ist die in der Abfrage angegebene explizite Join-Reihenfolge die tatsächliche Reihenfolge, in der die Relationen gejoint werden. Da der Query Planner nicht immer die optimale Join-Reihenfolge wählt, können fortgeschrittene Benutzer diese Variable vorübergehend auf 1 setzen und dann die gewünschte Join-Reihenfolge explizit angeben. Weitere Informationen finden Sie in [Abschnitt 14.3](14_Hinweise_zur_Performance.md#143-den-planner-mit-expliziten-joinklauseln-steuern).

Das Setzen dieses Werts auf `geqo_threshold` oder höher kann die Verwendung des GEQO-Planners auslösen, was zu nicht optimalen Plänen führen kann. Siehe [Abschnitt 19.7.3](#1973-genetischer-query-optimizer).

`plan_cache_mode` (`enum`)

Prepared Statements (entweder explizit vorbereitet oder implizit erzeugt, zum Beispiel durch PL/pgSQL) können mit custom oder generic plans ausgeführt werden. Custom Plans werden für jede Ausführung mit ihrem spezifischen Satz von Parameterwerten neu erstellt, während Generic Plans nicht von den Parameterwerten abhängen und über Ausführungen hinweg wiederverwendet werden können. Die Verwendung eines Generic Plan spart daher Planungszeit; wenn der ideale Plan jedoch stark von den Parameterwerten abhängt, kann ein Generic Plan ineffizient sein. Die Wahl zwischen diesen Optionen wird normalerweise automatisch getroffen, kann aber mit `plan_cache_mode` überschrieben werden. Die erlaubten Werte sind `auto` (der Standard), `force_custom_plan` und `force_generic_plan`. Diese Einstellung wird berücksichtigt, wenn ein gecachter Plan ausgeführt werden soll, nicht wenn er vorbereitet wird. Weitere Informationen finden Sie unter `PREPARE`.

`recursive_worktable_factor` (`floating point`)

Setzt die Schätzung des Planners für die durchschnittliche Größe der Working Table einer rekursiven Abfrage als Vielfaches der geschätzten Größe des anfänglichen nichtrekursiven Terms der Abfrage. Dies hilft dem Planner, die passendste Methode zum Joinen der Working Table mit den anderen Tabellen der Abfrage zu wählen. Der Standardwert ist `10.0`. Ein kleinerer Wert wie `1.0` kann hilfreich sein, wenn die Rekursion von einem Schritt zum nächsten einen niedrigen "Fan-out" hat, etwa bei Shortest-Path-Abfragen. Graphanalyse-Abfragen können von höheren Werten als dem Standard profitieren.

## 19.8. Fehlerberichte und Logging

### 19.8.1. Wohin geloggt wird

`log_destination` (`string`)

PostgreSQL unterstützt mehrere Methoden zum Loggen von Servermeldungen, darunter `stderr`, `csvlog`, `jsonlog` und `syslog`. Unter Windows wird außerdem `eventlog` unterstützt. Setzen Sie diesen Parameter auf eine durch Kommas getrennte Liste der gewünschten Log-Ziele. Standardmäßig wird nur nach `stderr` geloggt. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Wenn `csvlog` in `log_destination` enthalten ist, werden Log-Einträge im CSV-Format ("comma-separated value") ausgegeben, was für das Laden von Logs in Programme praktisch ist. Details siehe Abschnitt 19.8.4. `logging_collector` muss aktiviert sein, um Log-Ausgaben im CSV-Format zu erzeugen.

Wenn `jsonlog` in `log_destination` enthalten ist, werden Log-Einträge im JSON-Format ausgegeben, was für das Laden von Logs in Programme praktisch ist. Details siehe Abschnitt 19.8.5. `logging_collector` muss aktiviert sein, um Log-Ausgaben im JSON-Format zu erzeugen.

Wenn `stderr`, `csvlog` oder `jsonlog` enthalten ist, wird die Datei `current_logfiles` erstellt, um den Speicherort der aktuell vom Logging Collector verwendeten Log-Datei(en) und das zugehörige Log-Ziel festzuhalten. Dies bietet eine bequeme Möglichkeit, die aktuell von der Instanz verwendeten Logs zu finden. Hier ist ein Beispiel für den Inhalt dieser Datei:

```text
stderr log/postgresql.log
csvlog log/postgresql.csv
jsonlog log/postgresql.json
```

`current_logfiles` wird neu erstellt, wenn infolge von Rotation eine neue Log-Datei erzeugt wird und wenn `log_destination` neu geladen wird. Die Datei wird entfernt, wenn keines von `stderr`, `csvlog` oder `jsonlog` in `log_destination` enthalten ist und wenn der Logging Collector deaktiviert ist.

> Hinweis: Auf den meisten Unix-Systemen müssen Sie die Konfiguration des `syslog`-Daemons Ihres Systems ändern, um die Option `syslog` für `log_destination` nutzen zu können. PostgreSQL kann in die `syslog`-Facilities `LOCAL0` bis `LOCAL7` loggen (siehe `syslog_facility`), aber die Standard-`syslog`-Konfiguration auf den meisten Plattformen verwirft alle solchen Meldungen. Sie müssen der Konfigurationsdatei des `syslog`-Daemons beispielsweise etwas wie Folgendes hinzufügen:
>
> ```text
> local0.*            /var/log/postgresql
> ```
>
> Unter Windows sollten Sie bei Verwendung der Option `eventlog` für `log_destination` eine Ereignisquelle und ihre Bibliothek beim Betriebssystem registrieren, damit die Windows-Ereignisanzeige Event-Log-Meldungen sauber anzeigen kann. Details siehe [Abschnitt 18.12](18_Servereinrichtung_und_Betrieb.md#1812-event-log-unter-windows-registrieren).

`logging_collector` (`boolean`)

Dieser Parameter aktiviert den Logging Collector, einen Hintergrundprozess, der an `stderr` gesendete Log-Meldungen erfasst und in Log-Dateien umleitet. Dieser Ansatz ist häufig nützlicher als Logging nach `syslog`, da manche Arten von Meldungen möglicherweise nicht in der `syslog`-Ausgabe erscheinen. (Ein häufiges Beispiel sind Fehlermeldungen des Dynamic Linkers; ein anderes sind Fehlermeldungen, die von Skripten wie `archive_command` erzeugt werden.) Dieser Parameter kann nur beim Serverstart gesetzt werden.

> Hinweis: Es ist möglich, nach `stderr` zu loggen, ohne den Logging Collector zu verwenden; die Log-Meldungen gehen dann einfach dorthin, wohin `stderr` des Servers geleitet wird. Diese Methode eignet sich jedoch nur für geringe Log-Volumen, da sie keine bequeme Möglichkeit zur Rotation von Log-Dateien bietet. Außerdem kann es auf manchen Plattformen ohne Logging Collector zu verlorener oder verstümmelter Log-Ausgabe kommen, weil mehrere Prozesse, die gleichzeitig in dieselbe Log-Datei schreiben, einander überschreiben können.

> Hinweis: Der Logging Collector ist so entworfen, dass er niemals Meldungen verliert. Das bedeutet, dass Serverprozesse bei extrem hoher Last blockieren könnten, während sie versuchen, zusätzliche Log-Meldungen zu senden, wenn der Collector zurückgefallen ist. Im Gegensatz dazu verwirft `syslog` bevorzugt Meldungen, wenn es sie nicht schreiben kann; dadurch loggt es in solchen Fällen möglicherweise einige Meldungen nicht, blockiert aber nicht den Rest des Systems.

`log_directory` (`string`)

Wenn `logging_collector` aktiviert ist, bestimmt dieser Parameter das Verzeichnis, in dem Log-Dateien erstellt werden. Es kann als absoluter Pfad oder relativ zum Datenverzeichnis des Clusters angegeben werden. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `log`.

`log_filename` (`string`)

Wenn `logging_collector` aktiviert ist, setzt dieser Parameter die Dateinamen der erstellten Log-Dateien. Der Wert wird als `strftime`-Muster behandelt, sodass `%-`-Escapes verwendet werden können, um zeitabhängige Dateinamen anzugeben. (Beachten Sie, dass bei zeitzonenabhängigen `%-`-Escapes die Berechnung in der durch `log_timezone` angegebenen Zone erfolgt.) Die unterstützten `%-`-Escapes ähneln denen, die in der `strftime(1)`-Spezifikation der Open Group aufgeführt sind. Beachten Sie, dass das systemeigene `strftime` nicht direkt verwendet wird; plattformspezifische, nicht standardisierte Erweiterungen funktionieren daher nicht. Der Standardwert ist `postgresql-%Y-%m-%d_%H%M%S.log`.

Wenn Sie einen Dateinamen ohne Escapes angeben, sollten Sie ein Log-Rotationswerkzeug einplanen, um zu vermeiden, dass irgendwann die gesamte Festplatte gefüllt wird. In Releases vor 8.4 hängte PostgreSQL, wenn keine `%`-Escapes vorhanden waren, die Epoch-Zeit der Erzeugung der neuen Log-Datei an; das ist nicht mehr der Fall.

Wenn Ausgabe im CSV-Format in `log_destination` aktiviert ist, wird `.csv` an den zeitgestempelten Log-Dateinamen angehängt, um den Dateinamen für die CSV-Ausgabe zu erzeugen. (Wenn `log_filename` auf `.log` endet, wird der Suffix stattdessen ersetzt.)

Wenn Ausgabe im JSON-Format in `log_destination` aktiviert ist, wird `.json` an den zeitgestempelten Log-Dateinamen angehängt, um den Dateinamen für die JSON-Ausgabe zu erzeugen. (Wenn `log_filename` auf `.log` endet, wird der Suffix stattdessen ersetzt.)

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`log_file_mode` (`integer`)

Auf Unix-Systemen setzt dieser Parameter die Berechtigungen für Log-Dateien, wenn `logging_collector` aktiviert ist. (Unter Microsoft Windows wird dieser Parameter ignoriert.) Als Parameterwert wird ein numerischer Modus erwartet, der in dem von den Systemaufrufen `chmod` und `umask` akzeptierten Format angegeben ist. (Um das übliche oktale Format zu verwenden, muss die Zahl mit einer `0` beginnen.)

Die Standardberechtigungen sind `0600`, was bedeutet, dass nur der Servereigentümer die Log-Dateien lesen oder schreiben kann. Die andere häufig nützliche Einstellung ist `0640`, wodurch Mitglieder der Gruppe des Eigentümers die Dateien lesen können. Beachten Sie jedoch, dass Sie zur Verwendung einer solchen Einstellung `log_directory` ändern müssen, um die Dateien außerhalb des Datenverzeichnisses des Clusters zu speichern. In jedem Fall ist es unklug, Log-Dateien weltweit lesbar zu machen, da sie sensible Daten enthalten könnten.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`log_rotation_age` (`integer`)

Wenn `logging_collector` aktiviert ist, bestimmt dieser Parameter die maximale Zeitspanne, für die eine einzelne Log-Datei verwendet wird; danach wird eine neue Log-Datei erstellt. Wird dieser Wert ohne Einheit angegeben, wird er als Minuten interpretiert. Der Standardwert ist 24 Stunden. Setzen Sie ihn auf null, um die zeitbasierte Erstellung neuer Log-Dateien zu deaktivieren. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`log_rotation_size` (`integer`)

Wenn `logging_collector` aktiviert ist, bestimmt dieser Parameter die maximale Größe einer einzelnen Log-Datei. Nachdem diese Datenmenge in eine Log-Datei ausgegeben wurde, wird eine neue Log-Datei erstellt. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Der Standardwert ist 10 Megabyte. Setzen Sie ihn auf null, um die größenbasierte Erstellung neuer Log-Dateien zu deaktivieren. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`log_truncate_on_rotation` (`boolean`)

Wenn `logging_collector` aktiviert ist, bewirkt dieser Parameter, dass PostgreSQL eine vorhandene Log-Datei mit demselben Namen abschneidet bzw. überschreibt, statt an sie anzuhängen. Das Abschneiden erfolgt jedoch nur, wenn aufgrund zeitbasierter Rotation eine neue Datei geöffnet wird, nicht beim Serverstart oder bei größenbasierter Rotation. Wenn der Parameter `off` ist, wird in allen Fällen an vorhandene Dateien angehängt. Beispielsweise würde die Verwendung dieser Einstellung zusammen mit einem `log_filename` wie `postgresql-%H.log` dazu führen, dass vierundzwanzig stündliche Log-Dateien erzeugt und dann zyklisch überschrieben werden. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Siehe auch: <https://pubs.opengroup.org/onlinepubs/009695399/functions/strftime.html>

Beispiel: Um sieben Tage Logs aufzubewahren, mit einer Log-Datei pro Tag namens `server_log.Mon`, `server_log.Tue` usw., und das Log der letzten Woche automatisch mit dem Log dieser Woche zu überschreiben, setzen Sie `log_filename` auf `server_log.%a`, `log_truncate_on_rotation` auf `on` und `log_rotation_age` auf `1440`.

Beispiel: Um 24 Stunden Logs aufzubewahren, mit einer Log-Datei pro Stunde, aber außerdem früher zu rotieren, wenn die Größe der Log-Datei `1GB` überschreitet, setzen Sie `log_filename` auf `server_log.%H%M`, `log_truncate_on_rotation` auf `on`, `log_rotation_age` auf `60` und `log_rotation_size` auf `1000000`. Das Einbeziehen von `%M` in `log_filename` erlaubt größenbedingten Rotationen, die eventuell auftreten, einen anderen Dateinamen als den anfänglichen Dateinamen der Stunde zu wählen.

`syslog_facility` (`enum`)

Wenn Logging nach `syslog` aktiviert ist, bestimmt dieser Parameter die zu verwendende `syslog`-"Facility". Sie können zwischen `LOCAL0`, `LOCAL1`, `LOCAL2`, `LOCAL3`, `LOCAL4`, `LOCAL5`, `LOCAL6` und `LOCAL7` wählen; der Standardwert ist `LOCAL0`. Siehe auch die Dokumentation des `syslog`-Daemons Ihres Systems. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`syslog_ident` (`string`)

Wenn Logging nach `syslog` aktiviert ist, bestimmt dieser Parameter den Programmnamen, der zur Identifizierung von PostgreSQL-Meldungen in `syslog`-Logs verwendet wird. Der Standardwert ist `postgres`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`syslog_sequence_numbers` (`boolean`)

Wenn nach `syslog` geloggt wird und dieser Parameter `on` ist (der Standard), wird jeder Meldung eine steigende Sequenznummer vorangestellt (z. B. `[2]`). Dies umgeht die Unterdrückung "`--- last message repeated N times ---`", die viele `syslog`-Implementierungen standardmäßig durchführen. In moderneren `syslog`-Implementierungen kann die Unterdrückung wiederholter Meldungen konfiguriert werden (zum Beispiel `$RepeatedMsgReduction` in `rsyslog`), sodass dies möglicherweise nicht nötig ist. Sie können diesen Parameter außerdem ausschalten, wenn Sie wiederholte Meldungen tatsächlich unterdrücken möchten.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`syslog_split_messages` (`boolean`)

Wenn Logging nach `syslog` aktiviert ist, bestimmt dieser Parameter, wie Meldungen an `syslog` übergeben werden. Wenn er `on` ist (der Standard), werden Meldungen nach Zeilen aufgeteilt, und lange Zeilen werden so geteilt, dass sie in 1024 Bytes passen, was eine typische Größenbeschränkung traditioneller `syslog`-Implementierungen ist. Wenn er `off` ist, werden PostgreSQL-Serverlog-Meldungen unverändert an den `syslog`-Dienst übergeben, und es ist Aufgabe des `syslog`-Dienstes, mit den potenziell umfangreichen Meldungen umzugehen.

Wenn `syslog` letztlich in eine Textdatei loggt, ist die Wirkung in beiden Fällen dieselbe, und es ist am besten, die Einstellung auf `on` zu belassen, da die meisten `syslog`-Implementierungen große Meldungen entweder nicht verarbeiten können oder speziell dafür konfiguriert werden müssten. Wenn `syslog` jedoch letztlich in ein anderes Medium schreibt, kann es notwendig oder nützlicher sein, Meldungen logisch zusammenzuhalten.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`event_source` (`string`)

Wenn Logging in das Event Log aktiviert ist, bestimmt dieser Parameter den Programmnamen, der zur Identifizierung von PostgreSQL-Meldungen im Log verwendet wird. Der Standardwert ist `PostgreSQL`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

### 19.8.2. Wann geloggt wird

`log_min_messages` (`enum`)

Steuert, welche Meldungsstufen in das Serverlog geschrieben werden. Gültige Werte sind `DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1`, `INFO`, `NOTICE`, `WARNING`, `ERROR`, `LOG`, `FATAL` und `PANIC`. Jede Stufe umfasst alle nachfolgenden Stufen. Je später die Stufe, desto weniger Meldungen werden an das Log gesendet. Der Standardwert ist `WARNING`. Beachten Sie, dass `LOG` hier einen anderen Rang hat als in `client_min_messages`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_min_error_statement` (`enum`)

Steuert, welche SQL-Anweisungen, die eine Fehlerbedingung verursachen, im Serverlog aufgezeichnet werden. Die aktuelle SQL-Anweisung wird in den Log-Eintrag für jede Meldung der angegebenen Schwere oder höher aufgenommen. Gültige Werte sind `DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1`, `INFO`, `NOTICE`, `WARNING`, `ERROR`, `LOG`, `FATAL` und `PANIC`. Der Standardwert ist `ERROR`, was bedeutet, dass Anweisungen geloggt werden, die Fehler, Log-Meldungen, fatale Fehler oder Panics verursachen. Um das Logging fehlgeschlagener Anweisungen effektiv auszuschalten, setzen Sie diesen Parameter auf `PANIC`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_min_duration_statement` (`integer`)

Bewirkt, dass die Dauer jeder abgeschlossenen Anweisung geloggt wird, wenn die Anweisung mindestens die angegebene Zeitspanne lief. Wenn Sie den Parameter beispielsweise auf `250ms` setzen, werden alle SQL-Anweisungen geloggt, die `250ms` oder länger laufen. Das Aktivieren dieses Parameters kann hilfreich sein, um nicht optimierte Abfragen in Ihren Anwendungen aufzuspüren. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Das Setzen auf null gibt alle Anweisungsdauern aus. `-1` (der Standard) deaktiviert das Logging von Anweisungsdauern. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Dies überschreibt `log_min_duration_sample`, was bedeutet, dass Abfragen mit einer Dauer oberhalb dieser Einstellung keinem Sampling unterliegen und immer geloggt werden.

Für Clients, die das Extended Query Protocol verwenden, werden die Dauern der Schritte `Parse`, `Bind` und `Execute` unabhängig voneinander geloggt.

> Hinweis: Wenn diese Option zusammen mit `log_statement` verwendet wird, wird der Text von Anweisungen, die wegen `log_statement` geloggt werden, in der Dauer-Logmeldung nicht wiederholt. Wenn Sie nicht `syslog` verwenden, wird empfohlen, die PID oder Sitzungs-ID mit `log_line_prefix` zu loggen, damit Sie die Anweisungsmeldung über Prozess-ID oder Sitzungs-ID mit der späteren Dauermeldung verknüpfen können.

`log_min_duration_sample` (`integer`)

Erlaubt Sampling der Dauer abgeschlossener Anweisungen, die mindestens die angegebene Zeitspanne liefen. Dies erzeugt dieselbe Art von Log-Einträgen wie `log_min_duration_statement`, aber nur für eine Teilmenge der ausgeführten Anweisungen; die Sampling-Rate wird durch `log_statement_sample_rate` gesteuert. Wenn Sie den Parameter beispielsweise auf `100ms` setzen, werden alle SQL-Anweisungen, die `100ms` oder länger laufen, für Sampling in Betracht gezogen. Das Aktivieren dieses Parameters kann hilfreich sein, wenn der Traffic zu hoch ist, um alle Abfragen zu loggen. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Das Setzen auf null sampelt alle Anweisungsdauern. `-1` (der Standard) deaktiviert das Sampling von Anweisungsdauern. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Diese Einstellung hat niedrigere Priorität als `log_min_duration_statement`, was bedeutet, dass Anweisungen mit Dauern oberhalb von `log_min_duration_statement` keinem Sampling unterliegen und immer geloggt werden.

Die weiteren Hinweise zu `log_min_duration_statement` gelten auch für diese Einstellung.

`log_statement_sample_rate` (`floating point`)

Bestimmt den Anteil der Anweisungen mit einer Dauer oberhalb von `log_min_duration_sample`, die geloggt werden. Sampling ist stochastisch; zum Beispiel bedeutet `0.5`, dass statistisch eine Chance von eins zu zwei besteht, dass eine bestimmte Anweisung geloggt wird. Der Standardwert ist `1.0`, was bedeutet, dass alle gesampelten Anweisungen geloggt werden. Das Setzen auf null deaktiviert das gesampelte Logging von Anweisungsdauern, genauso wie das Setzen von `log_min_duration_sample` auf `-1`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_transaction_sample_rate` (`floating point`)

Setzt den Anteil der Transaktionen, deren Anweisungen zusätzlich zu aus anderen Gründen geloggten Anweisungen vollständig geloggt werden. Dies gilt für jede neue Transaktion unabhängig von der Dauer ihrer Anweisungen. Sampling ist stochastisch; zum Beispiel bedeutet `0.1`, dass statistisch eine Chance von eins zu zehn besteht, dass eine bestimmte Transaktion geloggt wird. `log_transaction_sample_rate` kann hilfreich sein, um eine Stichprobe von Transaktionen aufzubauen. Der Standardwert ist 0, was bedeutet, dass keine Anweisungen aus zusätzlichen Transaktionen geloggt werden. Das Setzen auf 1 loggt alle Anweisungen aller Transaktionen. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

> Hinweis: Wie alle Optionen zum Logging von Anweisungen kann diese Option erheblichen Overhead verursachen.

`log_startup_progress_interval` (`integer`)

Setzt die Zeitspanne, nach der der Startup-Prozess eine Meldung über eine lang laufende Operation loggt, die noch in Gang ist, sowie das Intervall zwischen weiteren Fortschrittsmeldungen für diese Operation. Der Standardwert ist 10 Sekunden. Eine Einstellung von `0` deaktiviert die Funktion. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Diese Einstellung wird separat auf jede Operation angewendet. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Wenn beispielsweise die Synchronisierung des Datenverzeichnisses 25 Sekunden dauert und danach das Zurücksetzen ungeloggter Relationen 8 Sekunden dauert, und wenn diese Einstellung den Standardwert von 10 Sekunden hat, werden Meldungen für die Synchronisierung des Datenverzeichnisses geloggt, nachdem sie 10 Sekunden und erneut nachdem sie 20 Sekunden in Gang war; für das Zurücksetzen ungeloggter Relationen wird jedoch nichts geloggt.

Tabelle 19.2 erklärt die von PostgreSQL verwendeten Meldungsschweregrade. Wenn die Logging-Ausgabe an `syslog` oder das Windows-`eventlog` gesendet wird, werden die Schweregrade wie in der Tabelle gezeigt übersetzt.

Tabelle 19.2. Meldungsschweregrade

| Schweregrad | Verwendung | `syslog` | `eventlog` |
| --- | --- | --- | --- |
| `DEBUG1` .. `DEBUG5` | Liefert zunehmend detaillierte Informationen zur Verwendung durch Entwickler. | `DEBUG` | `INFORMATION` |
| `INFO` | Liefert implizit vom Benutzer angeforderte Informationen, z. B. Ausgabe von `VACUUM VERBOSE`. | `INFO` | `INFORMATION` |
| `NOTICE` | Liefert Informationen, die für Benutzer hilfreich sein können, z. B. Hinweise auf das Kürzen langer Bezeichner. | `NOTICE` | `INFORMATION` |
| `WARNING` | Liefert Warnungen vor wahrscheinlichen Problemen, z. B. `COMMIT` außerhalb eines Transaktionsblocks. | `NOTICE` | `WARNING` |
| `ERROR` | Meldet einen Fehler, der den aktuellen Befehl abgebrochen hat. | `WARNING` | `ERROR` |
| `LOG` | Meldet Informationen, die für Administratoren interessant sind, z. B. Checkpoint-Aktivität. | `INFO` | `INFORMATION` |
| `FATAL` | Meldet einen Fehler, der die aktuelle Sitzung abgebrochen hat. | `ERR` | `ERROR` |
| `PANIC` | Meldet einen Fehler, der alle Datenbanksitzungen abgebrochen hat. | `CRIT` | `ERROR` |

### 19.8.3. Was geloggt wird

> Hinweis: Was Sie loggen, kann Auswirkungen auf die Sicherheit haben; siehe [Abschnitt 24.3](24_Regelmäßige_Datenbankwartung.md#243-logdateiwartung).

`application_name` (`string`)

`application_name` kann jede Zeichenkette mit weniger als `NAMEDATALEN` Zeichen sein (64 Zeichen in einem Standard-Build). Sie wird typischerweise von einer Anwendung beim Verbinden mit dem Server gesetzt. Der Name wird in der View `pg_stat_activity` angezeigt und in CSV-Log-Einträge aufgenommen. Er kann außerdem über den Parameter `log_line_prefix` in normale Log-Einträge aufgenommen werden. Im Wert von `application_name` dürfen nur druckbare ASCII-Zeichen verwendet werden. Andere Zeichen werden durch hexadezimale C-Escapes ersetzt.

`debug_print_parse` (`boolean`)
`debug_print_rewritten` (`boolean`)
`debug_print_plan` (`boolean`)

Diese Parameter aktivieren verschiedene Debugging-Ausgaben. Wenn sie gesetzt sind, geben sie für jede ausgeführte Abfrage den resultierenden Parse-Baum, die Ausgabe des Query Rewriters oder den Ausführungsplan aus. Diese Meldungen werden mit der Meldungsstufe `LOG` ausgegeben, sodass sie standardmäßig im Serverlog erscheinen, aber nicht an den Client gesendet werden. Sie können das ändern, indem Sie `client_min_messages` und/oder `log_min_messages` anpassen. Diese Parameter sind standardmäßig ausgeschaltet.

`debug_pretty_print` (`boolean`)

Wenn gesetzt, rückt `debug_pretty_print` die von `debug_print_parse`, `debug_print_rewritten` oder `debug_print_plan` erzeugten Meldungen ein. Dies führt zu besser lesbarer, aber deutlich längerer Ausgabe als das "kompakte" Format, das verwendet wird, wenn der Parameter ausgeschaltet ist. Er ist standardmäßig eingeschaltet.

`log_autovacuum_min_duration` (`integer`)

Bewirkt, dass jede von Autovacuum ausgeführte Aktion geloggt wird, wenn sie mindestens die angegebene Zeitspanne lief. Das Setzen auf null loggt alle Autovacuum-Aktionen. `-1` deaktiviert das Logging von Autovacuum-Aktionen. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Wenn Sie ihn beispielsweise auf `250ms` setzen, werden alle automatischen Vacuums und Analyzes geloggt, die `250ms` oder länger laufen. Zusätzlich wird, wenn dieser Parameter auf einen anderen Wert als `-1` gesetzt ist, eine Meldung geloggt, wenn eine Autovacuum-Aktion wegen einer kollidierenden Sperre oder einer gleichzeitig gelöschten Relation übersprungen wird. Der Standardwert ist `10min`. Das Aktivieren dieses Parameters kann hilfreich sein, um Autovacuum-Aktivität nachzuverfolgen. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`log_checkpoints` (`boolean`)

Bewirkt, dass Checkpoints und Restartpoints im Serverlog geloggt werden. Einige Statistiken sind in den Log-Meldungen enthalten, darunter die Anzahl geschriebener Buffer und die dafür aufgewendete Schreibzeit. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `on`.

`log_connections` (`string`)

Bewirkt, dass Aspekte jeder Verbindung zum Server geloggt werden. Der Standardwert ist der leere String `''`, der sämtliches Verbindungslogging deaktiviert. Die folgenden Optionen können einzeln oder in einer durch Kommas getrennten Liste angegeben werden:

Tabelle 19.3. Optionen für Verbindungslogging

| Name | Beschreibung |
| --- | --- |
| `receipt` | Loggt den Empfang einer Verbindung. |
| `authentication` | Loggt die ursprüngliche Identität, die von einer Authentifizierungsmethode verwendet wurde, um einen Benutzer zu identifizieren. In den meisten Fällen entspricht die Identitätszeichenkette dem PostgreSQL-Benutzernamen, aber manche Authentifizierungsmethoden von Drittanbietern können die ursprüngliche Benutzerkennung ändern, bevor der Server sie speichert. Fehlgeschlagene Authentifizierung wird unabhängig vom Wert dieser Einstellung immer geloggt. |
| `authorization` | Loggt den erfolgreichen Abschluss der Autorisierung. Zu diesem Zeitpunkt ist die Verbindung hergestellt, aber das Backend ist noch nicht vollständig eingerichtet. Die Log-Meldung enthält den autorisierten Benutzernamen sowie, falls zutreffend, den Datenbanknamen und den Anwendungsnamen. |
| `setup_durations` | Loggt die Zeit, die zum Herstellen der Verbindung und Einrichten des Backends benötigt wurde, bis die Verbindung bereit ist, ihre erste Abfrage auszuführen. Die Log-Meldung enthält drei Dauern: die gesamte Einrichtungsdauer (vom Akzeptieren der eingehenden Verbindung durch den Postmaster bis zur Query-Bereitschaft der Verbindung), die Zeit zum Forken des neuen Backends und die Zeit zur Authentifizierung des Benutzers. |
| `all` | Ein Komfortalias, der der Angabe aller Optionen entspricht. Wenn `all` in einer Liste mit anderen Optionen angegeben ist, werden alle Verbindungsaspekte geloggt. |

Das Logging von Verbindungsabbrüchen wird separat durch `log_disconnections` gesteuert.

Aus Gründen der Rückwärtskompatibilität werden `on`, `off`, `true`, `false`, `yes`, `no`, `1` und `0` weiterhin unterstützt. Die positiven Werte entsprechen der Angabe der Optionen `receipt`, `authentication` und `authorization`.

Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diesen Parameter beim Sitzungsstart ändern; innerhalb einer Sitzung kann er überhaupt nicht geändert werden.

> Hinweis: Manche Clientprogramme, etwa `psql`, versuchen beim Ermitteln, ob ein Passwort erforderlich ist, zweimal eine Verbindung herzustellen; doppelte Meldungen "connection received" weisen daher nicht notwendigerweise auf ein Problem hin.

`log_disconnections` (`boolean`)

Bewirkt, dass Sitzungsbeendigungen geloggt werden. Die Log-Ausgabe liefert ähnliche Informationen wie `log_connections`, zusätzlich zur Dauer der Sitzung. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diesen Parameter beim Sitzungsstart ändern; innerhalb einer Sitzung kann er überhaupt nicht geändert werden. Der Standardwert ist `off`.

`log_duration` (`boolean`)

Bewirkt, dass die Dauer jeder abgeschlossenen Anweisung geloggt wird. Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Für Clients, die das Extended Query Protocol verwenden, werden die Dauern der Schritte `Parse`, `Bind` und `Execute` unabhängig voneinander geloggt.

> Hinweis: Der Unterschied zwischen dem Aktivieren von `log_duration` und dem Setzen von `log_min_duration_statement` auf null besteht darin, dass das Überschreiten von `log_min_duration_statement` erzwingt, dass der Text der Abfrage geloggt wird, diese Option jedoch nicht. Wenn also `log_duration` eingeschaltet ist und `log_min_duration_statement` einen positiven Wert hat, werden alle Dauern geloggt, aber der Abfragetext wird nur für Anweisungen aufgenommen, die den Schwellenwert überschreiten. Dieses Verhalten kann nützlich sein, um Statistiken in Installationen mit hoher Last zu sammeln.

`log_error_verbosity` (`enum`)

Steuert, wie viele Details für jede geloggte Meldung in das Serverlog geschrieben werden. Gültige Werte sind `TERSE`, `DEFAULT` und `VERBOSE`, wobei jeder Wert mehr Felder zu den angezeigten Meldungen hinzufügt. `TERSE` schließt das Logging der Fehlerinformationen `DETAIL`, `HINT`, `QUERY` und `CONTEXT` aus. `VERBOSE`-Ausgabe enthält den SQLSTATE-Fehlercode (siehe auch Anhang A) sowie Dateiname, Funktionsname und Zeilennummer im Quellcode, die den Fehler erzeugt haben. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_hostname` (`boolean`)

Standardmäßig zeigen Verbindungs-Logmeldungen nur die IP-Adresse des verbindenden Hosts. Das Einschalten dieses Parameters bewirkt, dass zusätzlich der Hostname geloggt wird. Beachten Sie, dass dies je nach Ihrer Konfiguration der Namensauflösung eine nicht unerhebliche Performance-Strafe verursachen kann. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`log_line_prefix` (`string`)

Dies ist ein String im `printf`-Stil, der am Anfang jeder Log-Zeile ausgegeben wird. `%`-Zeichen leiten Escape-Sequenzen ein, die wie unten beschrieben durch Statusinformationen ersetzt werden. Nicht erkannte Escapes werden ignoriert. Andere Zeichen werden direkt in die Log-Zeile kopiert. Manche Escapes werden nur von Sitzungsprozessen erkannt und von Hintergrundprozessen, etwa dem Hauptserverprozess, als leer behandelt. Statusinformationen können links oder rechts ausgerichtet werden, indem nach dem `%` und vor der Option ein numerisches Literal angegeben wird. Ein negativer Wert bewirkt, dass die Statusinformation rechts mit Leerzeichen aufgefüllt wird, um eine Mindestbreite zu erreichen, während ein positiver Wert links auffüllt. Auffüllung kann die menschliche Lesbarkeit von Log-Dateien verbessern.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `'%m [%p] '`, womit ein Zeitstempel und die Prozess-ID geloggt werden.

| Escape | Wirkung | Nur Sitzung |
| --- | --- | --- |
| `%a` | Anwendungsname | ja |
| `%u` | Benutzername | ja |
| `%d` | Datenbankname | ja |
| `%r` | Entfernter Hostname oder IP-Adresse und entfernter Port | ja |
| `%h` | Entfernter Hostname oder IP-Adresse | ja |
| `%L` | Lokale Adresse (die IP-Adresse auf dem Server, mit der sich der Client verbunden hat) | ja |
| `%b` | Backend-Typ | nein |
| `%p` | Prozess-ID | nein |
| `%P` | Prozess-ID des Parallel-Group-Leaders, wenn dieser Prozess ein Parallel-Query-Worker ist | nein |
| `%t` | Zeitstempel ohne Millisekunden | nein |
| `%m` | Zeitstempel mit Millisekunden | nein |
| `%n` | Zeitstempel mit Millisekunden als Unix-Epoch | nein |
| `%i` | Command Tag: Typ des aktuellen Befehls der Sitzung | ja |
| `%e` | SQLSTATE-Fehlercode | nein |
| `%c` | Sitzungs-ID; siehe unten | nein |
| `%l` | Nummer der Log-Zeile für jede Sitzung oder jeden Prozess, beginnend bei 1 | nein |
| `%s` | Zeitstempel des Prozessstarts | nein |
| `%v` | Virtuelle Transaktions-ID (`procNumber/localXID`); siehe Abschnitt 67.1 | nein |
| `%x` | Transaktions-ID (0, wenn keine zugewiesen ist); siehe [Abschnitt 67.1](67_Transaktionsverarbeitung.md#671-transaktionen-und-identifikatoren) | nein |
| `%q` | Erzeugt keine Ausgabe, weist aber Nicht-Sitzungsprozesse an, an dieser Stelle im String aufzuhören; wird von Sitzungsprozessen ignoriert | nein |
| `%Q` | Query-Identifier der aktuellen Abfrage. Query-Identifier werden standardmäßig nicht berechnet, daher ist dieses Feld null, sofern der Parameter `compute_query_id` nicht aktiviert ist oder ein Drittanbieter-Modul konfiguriert ist, das Query-Identifier berechnet. | ja |
| `%%` | Literales `%` | nein |

Der Backend-Typ entspricht der Spalte `backend_type` in der View `pg_stat_activity`, aber im Log können zusätzliche Typen erscheinen, die in dieser View nicht angezeigt werden.

Das Escape `%c` gibt einen quasi eindeutigen Sitzungsbezeichner aus, der aus zwei 4-Byte-Hexadezimalzahlen ohne führende Nullen besteht, getrennt durch einen Punkt. Die Zahlen sind die Startzeit des Prozesses und die Prozess-ID; daher kann `%c` auch als platzsparende Möglichkeit dienen, diese Werte auszugeben. Um den Sitzungsbezeichner aus `pg_stat_activity` zu erzeugen, verwenden Sie beispielsweise diese Abfrage:

```sql
SELECT to_hex(trunc(EXTRACT(EPOCH FROM backend_start))::integer)
       || '.' ||
       to_hex(pid)
FROM pg_stat_activity;
```

> Tipp: Wenn Sie einen nichtleeren Wert für `log_line_prefix` setzen, sollte sein letztes Zeichen normalerweise ein Leerzeichen sein, um eine visuelle Trennung vom Rest der Log-Zeile zu schaffen. Auch ein Satzzeichen kann verwendet werden.

> Tipp: `Syslog` erzeugt eigene Zeitstempel- und Prozess-ID-Informationen, daher möchten Sie diese Escapes wahrscheinlich nicht aufnehmen, wenn Sie nach `syslog` loggen.

> Tipp: Das Escape `%q` ist nützlich, wenn Informationen aufgenommen werden, die nur im Sitzungs- bzw. Backend-Kontext verfügbar sind, etwa Benutzer- oder Datenbankname. Beispiel:
>
> ```text
> log_line_prefix = '%m [%p] %q%u@%d/%a '
> ```

> Hinweis: Das Escape `%Q` meldet für Zeilen, die von `log_statement` ausgegeben werden, immer einen Null-Identifier, weil `log_statement` Ausgabe erzeugt, bevor ein Identifier berechnet werden kann. Das gilt auch für ungültige Anweisungen, für die kein Identifier berechnet werden kann.

`log_lock_waits` (`boolean`)

Steuert, ob eine Log-Meldung erzeugt wird, wenn eine Sitzung länger als `deadlock_timeout` darauf wartet, eine Sperre zu erwerben. Dies ist nützlich, um festzustellen, ob Sperrwartezeiten schlechte Performance verursachen. Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_lock_failures` (`boolean`)

Steuert, ob eine detaillierte Log-Meldung erzeugt wird, wenn der Erwerb einer Sperre fehlschlägt. Dies ist nützlich, um die Ursachen von Sperrfehlern zu analysieren. Derzeit werden nur Sperrfehler aufgrund von `SELECT NOWAIT` unterstützt. Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_recovery_conflict_waits` (`boolean`)

Steuert, ob eine Log-Meldung erzeugt wird, wenn der Startup-Prozess länger als `deadlock_timeout` auf Recovery-Konflikte wartet. Dies ist nützlich, um festzustellen, ob Recovery-Konflikte verhindern, dass die Recovery WAL anwendet.

Der Standardwert ist `off`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`log_parameter_max_length` (`integer`)

Wenn größer als null, wird jeder Bind-Parameterwert, der mit einer nicht fehlerbezogenen Meldung zum Statement-Logging geloggt wird, auf so viele Bytes gekürzt. Null deaktiviert das Logging von Bind-Parametern für nicht fehlerbezogene Statement-Logs. `-1` (der Standard) erlaubt, Bind-Parameter vollständig zu loggen. Wird dieser Wert ohne Einheit angegeben, wird er als Bytes interpretiert. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Diese Einstellung betrifft nur Log-Meldungen, die infolge von `log_statement`, `log_duration` und verwandten Einstellungen ausgegeben werden. Werte ungleich null für diese Einstellung verursachen etwas Overhead, insbesondere wenn Parameter in binärer Form gesendet werden, da dann eine Konvertierung nach Text erforderlich ist.

`log_parameter_max_length_on_error` (`integer`)

Wenn größer als null, wird jeder in Fehlermeldungen gemeldete Bind-Parameterwert auf so viele Bytes gekürzt. Null (der Standard) deaktiviert das Einbeziehen von Bind-Parametern in Fehlermeldungen. `-1` erlaubt, Bind-Parameter vollständig auszugeben. Wird dieser Wert ohne Einheit angegeben, wird er als Bytes interpretiert.

Werte ungleich null für diese Einstellung verursachen Overhead, da PostgreSQL zu Beginn jeder Anweisung textuelle Darstellungen von Parameterwerten im Speicher ablegen muss, unabhängig davon, ob später tatsächlich ein Fehler auftritt. Der Overhead ist größer, wenn Bind-Parameter in binärer Form gesendet werden, als wenn sie als Text gesendet werden, da im ersten Fall Datenkonvertierung nötig ist, während im zweiten nur die Zeichenkette kopiert werden muss.

`log_statement` (`enum`)

Steuert, welche SQL-Anweisungen geloggt werden. Gültige Werte sind `none` (aus), `ddl`, `mod` und `all` (alle Anweisungen). `ddl` loggt alle Datendefinitionsanweisungen, etwa `CREATE`-, `ALTER`- und `DROP`-Anweisungen. `mod` loggt alle `ddl`-Anweisungen plus datenverändernde Anweisungen wie `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE` und `COPY FROM`. `PREPARE`-, `EXECUTE`- und `EXPLAIN ANALYZE`-Anweisungen werden ebenfalls geloggt, wenn ihr enthaltener Befehl von einem passenden Typ ist. Für Clients, die das Extended Query Protocol verwenden, erfolgt das Logging, wenn eine `Execute`-Meldung empfangen wird, und die Werte der `Bind`-Parameter werden einbezogen (wobei enthaltene einfache Anführungszeichen verdoppelt werden).

Der Standardwert ist `none`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

> Hinweis: Anweisungen, die einfache Syntaxfehler enthalten, werden selbst mit der Einstellung `log_statement = all` nicht geloggt, weil die Log-Meldung erst nach grundlegendem Parsing ausgegeben wird, mit dem der Anweisungstyp bestimmt wird. Beim Extended Query Protocol loggt diese Einstellung ebenfalls keine Anweisungen, die vor der `Execute`-Phase fehlschlagen, also während Parse-Analyse oder Planung. Setzen Sie `log_min_error_statement` auf `ERROR` oder niedriger, um solche Anweisungen zu loggen.
>
> Geloggte Anweisungen können sensible Daten offenlegen und sogar Klartextpasswörter enthalten.

`log_replication_commands` (`boolean`)

Bewirkt, dass jeder Replikationsbefehl und jeder Erwerb bzw. jede Freigabe eines Replikationsslots durch einen Walsender-Prozess im Serverlog geloggt wird. Weitere Informationen über Replikationsbefehle finden Sie in [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll). Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_temp_files` (`integer`)

Steuert das Logging von Namen und Größen temporärer Dateien. Temporäre Dateien können für Sortierungen, Hashes und temporäre Abfrageergebnisse erstellt werden. Wenn durch diese Einstellung aktiviert, wird beim Löschen jeder temporären Datei ein Log-Eintrag mit der Dateigröße in Bytes ausgegeben. Ein Wert von null loggt alle Informationen zu temporären Dateien, während positive Werte nur Dateien loggen, deren Größe größer oder gleich der angegebenen Datenmenge ist. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Die Standardeinstellung ist `-1`, wodurch dieses Logging deaktiviert wird. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`log_timezone` (`string`)

Setzt die Zeitzone, die für in das Serverlog geschriebene Zeitstempel verwendet wird. Anders als `TimeZone` ist dieser Wert clusterweit, sodass alle Sitzungen Zeitstempel konsistent melden. Der eingebaute Standardwert ist `GMT`, aber dieser wird typischerweise in `postgresql.conf` überschrieben; `initdb` installiert dort eine Einstellung, die der Systemumgebung entspricht. Weitere Informationen finden Sie in Abschnitt 8.5.3. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

### 19.8.4. Log-Ausgabe im CSV-Format verwenden

Das Einbeziehen von `csvlog` in die Liste `log_destination` bietet eine bequeme Möglichkeit, Log-Dateien in eine Datenbanktabelle zu importieren. Diese Option gibt Log-Zeilen im CSV-Format aus, mit diesen Spalten: Zeitstempel mit Millisekunden, Benutzername, Datenbankname, Prozess-ID, Client-Host:Port-Nummer, Sitzungs-ID, Zeilennummer pro Sitzung, Command Tag, Sitzungsstartzeit, virtuelle Transaktions-ID, reguläre Transaktions-ID, Fehlerschwere, SQLSTATE-Code, Fehlermeldung, Fehlermeldungsdetail, Hinweis, interne Abfrage, die zum Fehler führte (falls vorhanden), Zeichenzählung der Fehlerposition darin, Fehlerkontext, Benutzerabfrage, die zum Fehler führte (falls vorhanden und durch `log_min_error_statement` aktiviert), Zeichenzählung der Fehlerposition darin, Ort des Fehlers im PostgreSQL-Quellcode (wenn `log_error_verbosity` auf `verbose` gesetzt ist), Anwendungsname, Backend-Typ, Prozess-ID des Parallel-Group-Leaders und Query-ID. Hier ist eine Beispiel-Tabellendefinition zum Speichern von Log-Ausgaben im CSV-Format:

```sql
CREATE TABLE postgres_log
(
   log_time timestamp(3) with time zone,
   user_name text,
   database_name text,
   process_id integer,
   connection_from text,
   session_id text,
   session_line_num bigint,
   command_tag text,
   session_start_time timestamp with time zone,
   virtual_transaction_id text,
   transaction_id bigint,
   error_severity text,
   sql_state_code text,
   message text,
   detail text,
   hint text,
   internal_query text,
   internal_query_pos integer,
   context text,
   query text,
   query_pos integer,
   location text,
   application_name text,
   backend_type text,
   leader_pid integer,
   query_id bigint,
   PRIMARY KEY (session_id, session_line_num)
);
```

Um eine Log-Datei in diese Tabelle zu importieren, verwenden Sie den Befehl `COPY FROM`:

```sql
COPY postgres_log FROM '/full/path/to/logfile.csv' WITH csv;
```

Es ist außerdem möglich, auf die Datei als Foreign Table zuzugreifen, indem das mitgelieferte Modul `file_fdw` verwendet wird.

Es gibt einige Dinge, die Sie tun sollten, um das Importieren von CSV-Log-Dateien zu vereinfachen:

1. Setzen Sie `log_filename` und `log_rotation_age`, um ein konsistentes, vorhersagbares Benennungsschema für Ihre Log-Dateien bereitzustellen. Dadurch können Sie vorhersagen, wie der Dateiname lauten wird, und erkennen, wann eine einzelne Log-Datei vollständig und damit importbereit ist.
2. Setzen Sie `log_rotation_size` auf `0`, um größenbasierte Log-Rotation zu deaktivieren, da sie den Dateinamen schwer vorhersagbar macht.
3. Setzen Sie `log_truncate_on_rotation` auf `on`, damit alte Log-Daten nicht mit neuen in derselben Datei vermischt werden.
4. Die obige Tabellendefinition enthält eine Primary-Key-Angabe. Das ist nützlich, um sich gegen versehentliches mehrfaches Importieren derselben Informationen zu schützen. Der Befehl `COPY` committet alle Daten, die er importiert, auf einmal; daher führt jeder Fehler dazu, dass der gesamte Import fehlschlägt. Wenn Sie eine teilweise Log-Datei importieren und später dieselbe Datei erneut importieren, wenn sie vollständig ist, lässt die Primary-Key-Verletzung den Import fehlschlagen. Warten Sie, bis das Log vollständig und geschlossen ist, bevor Sie importieren. Dieses Verfahren schützt außerdem davor, versehentlich eine unvollständig geschriebene Teilzeile zu importieren, was ebenfalls dazu führen würde, dass `COPY` fehlschlägt.

### 19.8.5. Log-Ausgabe im JSON-Format verwenden

Das Einbeziehen von `jsonlog` in die Liste `log_destination` bietet eine bequeme Möglichkeit, Log-Dateien in viele verschiedene Programme zu importieren. Diese Option gibt Log-Zeilen im JSON-Format aus.

String-Felder mit Nullwerten werden von der Ausgabe ausgeschlossen. In Zukunft können zusätzliche Felder hinzukommen. Benutzeranwendungen, die `jsonlog`-Ausgabe verarbeiten, sollten unbekannte Felder ignorieren.

Jede Log-Zeile wird als JSON-Objekt mit den in Tabelle 19.4 gezeigten Schlüsseln und zugehörigen Werten serialisiert.

Tabelle 19.4. Schlüssel und Werte von JSON-Log-Einträgen

| Schlüsselname | Typ | Beschreibung |
| --- | --- | --- |
| `timestamp` | `string` | Zeitstempel mit Millisekunden |
| `user` | `string` | Benutzername |
| `dbname` | `string` | Datenbankname |
| `pid` | `number` | Prozess-ID |
| `remote_host` | `string` | Client-Host |
| `remote_port` | `number` | Client-Port |
| `session_id` | `string` | Sitzungs-ID |
| `line_num` | `number` | Zeilennummer pro Sitzung |
| `ps` | `string` | Aktuelle `ps`-Anzeige |
| `session_start` | `string` | Sitzungsstartzeit |
| `vxid` | `string` | Virtuelle Transaktions-ID |
| `txid` | `string` | Reguläre Transaktions-ID |
| `error_severity` | `string` | Fehlerschwere |
| `state_code` | `string` | SQLSTATE-Code |
| `message` | `string` | Fehlermeldung |
| `detail` | `string` | Fehlermeldungsdetail |
| `hint` | `string` | Fehlermeldungshinweis |
| `internal_query` | `string` | Interne Abfrage, die zum Fehler führte |
| `internal_position` | `number` | Cursor-Index in der internen Abfrage |
| `context` | `string` | Fehlerkontext |
| `statement` | `string` | Vom Client gelieferter Abfragestring |
| `cursor_position` | `number` | Cursor-Index im Abfragestring |
| `func_name` | `string` | Funktionsname des Fehlerorts |
| `file_name` | `string` | Dateiname des Fehlerorts |
| `file_line_num` | `number` | Zeilennummer des Fehlerorts |
| `application_name` | `string` | Name der Client-Anwendung |
| `backend_type` | `string` | Backend-Typ |
| `leader_pid` | `number` | Prozess-ID des Leaders für aktive parallele Worker |
| `query_id` | `number` | Query-ID |

### 19.8.6. Prozesstitel

Diese Einstellungen steuern, wie Prozesstitel von Serverprozessen geändert werden. Prozesstitel werden typischerweise mit Programmen wie `ps` oder unter Windows mit Process Explorer betrachtet. Details siehe [Abschnitt 27.1](27_Datenbankaktivität_überwachen.md#271-standardunixwerkzeuge).

`cluster_name` (`string`)

Setzt einen Namen, der diesen Datenbank-Cluster bzw. diese Instanz für verschiedene Zwecke identifiziert. Der Clustername erscheint im Prozesstitel aller Serverprozesse in diesem Cluster. Außerdem ist er der standardmäßige Anwendungsname für eine Standby-Verbindung (siehe `synchronous_standby_names`).

Der Name kann jede Zeichenkette mit weniger als `NAMEDATALEN` Zeichen sein (64 Zeichen in einem Standard-Build). Im Wert von `cluster_name` dürfen nur druckbare ASCII-Zeichen verwendet werden. Andere Zeichen werden durch hexadezimale C-Escapes ersetzt. Es wird kein Name angezeigt, wenn dieser Parameter auf den leeren String `''` gesetzt ist (was der Standard ist). Dieser Parameter kann nur beim Serverstart gesetzt werden.

`update_process_title` (`boolean`)

Aktiviert die Aktualisierung des Prozesstitels jedes Mal, wenn ein neuer SQL-Befehl vom Server empfangen wird. Diese Einstellung ist auf den meisten Plattformen standardmäßig `on`, unter Windows jedoch wegen des dort größeren Overheads für die Aktualisierung des Prozesstitels standardmäßig `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

## 19.9. Laufzeitstatistiken

### 19.9.1. Kumulative Abfrage- und Indexstatistiken

Diese Parameter steuern das serverweite System kumulativer Statistiken. Wenn es aktiviert ist, können die gesammelten Daten über die System-Views der Familien `pg_stat` und `pg_statio` abgerufen werden. Weitere Informationen finden Sie in [Kapitel 27](27_Datenbankaktivität_überwachen.md).

`track_activities` (`boolean`)

Aktiviert das Sammeln von Informationen über den aktuell ausgeführten Befehl jeder Sitzung, zusammen mit seinem Identifier und dem Zeitpunkt, zu dem die Ausführung dieses Befehls begann. Dieser Parameter ist standardmäßig eingeschaltet. Beachten Sie, dass diese Informationen selbst bei Aktivierung nur für Superuser, Rollen mit Privilegien der Rolle `pg_read_all_stats` und den Benutzer sichtbar sind, dem die gemeldeten Sitzungen gehören (einschließlich Sitzungen, die zu einer Rolle gehören, für die er Privilegien hat); daher sollte dies kein Sicherheitsrisiko darstellen. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`track_activity_query_size` (`integer`)

Gibt die Speichermenge an, die reserviert wird, um den Text des aktuell ausgeführten Befehls für jede aktive Sitzung im Feld `pg_stat_activity.query` zu speichern. Wird dieser Wert ohne Einheit angegeben, wird er als Bytes interpretiert. Der Standardwert ist 1024 Bytes. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`track_counts` (`boolean`)

Aktiviert das Sammeln von Statistiken über Datenbankaktivität. Dieser Parameter ist standardmäßig eingeschaltet, da der Autovacuum-Daemon die gesammelten Informationen benötigt. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`track_cost_delay_timing` (`boolean`)

Aktiviert die Zeitmessung der kostenbasierten Vacuum-Verzögerung (siehe [Abschnitt 19.10.2](#19102-kostenbasierte-vacuumverzögerung)). Dieser Parameter ist standardmäßig ausgeschaltet, da er wiederholt die aktuelle Zeit beim Betriebssystem abfragt, was auf manchen Plattformen erheblichen Overhead verursachen kann. Sie können das Werkzeug `pg_test_timing` verwenden, um den Overhead der Zeitmessung auf Ihrem System zu messen. Timing-Informationen zur kostenbasierten Vacuum-Verzögerung werden in `pg_stat_progress_vacuum`, `pg_stat_progress_analyze`, in der Ausgabe von `VACUUM` und `ANALYZE` bei Verwendung der Option `VERBOSE` sowie durch Autovacuum für Auto-Vacuums und Auto-Analyzes angezeigt, wenn `log_autovacuum_min_duration` gesetzt ist. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`track_io_timing` (`boolean`)

Aktiviert die Zeitmessung von Datenbank-I/O-Wartezeiten. Dieser Parameter ist standardmäßig ausgeschaltet, da er wiederholt die aktuelle Zeit beim Betriebssystem abfragt, was auf manchen Plattformen erheblichen Overhead verursachen kann. Sie können das Werkzeug `pg_test_timing` verwenden, um den Overhead der Zeitmessung auf Ihrem System zu messen. I/O-Timing-Informationen werden in `pg_stat_database`, in `pg_stat_io` (wenn `object` nicht `wal` ist), in der Ausgabe der Funktion `pg_stat_get_backend_io()` (wenn `object` nicht `wal` ist), in der Ausgabe von `EXPLAIN` bei Verwendung der Option `BUFFERS`, in der Ausgabe von `VACUUM` bei Verwendung der Option `VERBOSE`, durch Autovacuum für Auto-Vacuums und Auto-Analyzes bei gesetztem `log_autovacuum_min_duration` sowie durch `pg_stat_statements` angezeigt. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`track_wal_io_timing` (`boolean`)

Aktiviert die Zeitmessung von WAL-I/O-Wartezeiten. Dieser Parameter ist standardmäßig ausgeschaltet, da er wiederholt die aktuelle Zeit beim Betriebssystem abfragt, was auf manchen Plattformen erheblichen Overhead verursachen kann. Sie können das Werkzeug `pg_test_timing` verwenden, um den Overhead der Zeitmessung auf Ihrem System zu messen. I/O-Timing-Informationen werden in `pg_stat_io` für das Objekt `wal` und in der Ausgabe der Funktion `pg_stat_get_backend_io()` für das Objekt `wal` angezeigt. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`track_functions` (`enum`)

Aktiviert das Verfolgen der Anzahl von Funktionsaufrufen und der verwendeten Zeit. Geben Sie `pl` an, um nur Funktionen prozeduraler Sprachen zu verfolgen, oder `all`, um zusätzlich SQL- und C-Sprachfunktionen zu verfolgen. Der Standardwert ist `none`, wodurch das Verfolgen von Funktionsstatistiken deaktiviert wird. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

> Hinweis: SQL-Sprachfunktionen, die einfach genug sind, um in die aufrufende Abfrage "inline" eingefügt zu werden, werden unabhängig von dieser Einstellung nicht verfolgt.

`stats_fetch_consistency` (`enum`)

Bestimmt das Verhalten, wenn innerhalb einer Transaktion mehrfach auf kumulative Statistiken zugegriffen wird. Bei `none` ruft jeder Zugriff die Zähler erneut aus Shared Memory ab. Bei `cache` cached der erste Zugriff auf Statistiken für ein Objekt diese Statistiken bis zum Ende der Transaktion, sofern `pg_stat_clear_snapshot()` nicht aufgerufen wird. Bei `snapshot` cached der erste Statistikzugriff alle in der aktuellen Datenbank zugänglichen Statistiken bis zum Ende der Transaktion, sofern `pg_stat_clear_snapshot()` nicht aufgerufen wird. Das Ändern dieses Parameters in einer Transaktion verwirft den Statistik-Snapshot. Der Standardwert ist `cache`.

> Hinweis: `none` eignet sich am besten für Monitoring-Systeme. Wenn Werte nur einmal abgerufen werden, ist dies am effizientesten. `cache` stellt sicher, dass wiederholte Zugriffe dieselben Werte liefern, was für Abfragen mit beispielsweise Self-Joins wichtig ist. `snapshot` kann nützlich sein, wenn Statistiken interaktiv untersucht werden, hat aber höheren Overhead, insbesondere wenn viele Datenbankobjekte existieren.

### 19.9.2. Statistik-Monitoring

`compute_query_id` (`enum`)

Aktiviert die serverinterne Berechnung eines Query-Identifiers. Query-Identifier können in der View `pg_stat_activity`, mit `EXPLAIN` angezeigt oder im Log ausgegeben werden, wenn dies über den Parameter `log_line_prefix` konfiguriert ist. Die Erweiterung `pg_stat_statements` erfordert ebenfalls, dass ein Query-Identifier berechnet wird. Beachten Sie, dass alternativ ein externes Modul verwendet werden kann, wenn die serverinterne Berechnungsmethode des Query-Identifiers nicht akzeptabel ist. In diesem Fall muss die serverinterne Berechnung immer deaktiviert sein. Gültige Werte sind `off` (immer deaktiviert), `on` (immer aktiviert), `auto`, wodurch Module wie `pg_stat_statements` sie automatisch aktivieren können, und `regress`, das dieselbe Wirkung wie `auto` hat, außer dass der Query-Identifier in der Ausgabe von `EXPLAIN` nicht angezeigt wird, um automatisierte Regressionstests zu erleichtern. Der Standardwert ist `auto`.

> Hinweis: Um sicherzustellen, dass nur ein Query-Identifier berechnet und angezeigt wird, sollten Erweiterungen, die Query-Identifier berechnen, einen Fehler auslösen, wenn bereits ein Query-Identifier berechnet wurde.

`log_statement_stats` (`boolean`)
`log_parser_stats` (`boolean`)
`log_planner_stats` (`boolean`)
`log_executor_stats` (`boolean`)

Gibt für jede Abfrage Performance-Statistiken des jeweiligen Moduls in das Serverlog aus. Dies ist ein grobes Profiling-Instrument, ähnlich der Unix-Betriebssystemfunktion `getrusage()`. `log_statement_stats` meldet Gesamtstatistiken für die Anweisung, während die anderen Optionen Statistiken pro Modul melden. `log_statement_stats` kann nicht zusammen mit einer der modulbezogenen Optionen aktiviert werden. Alle diese Optionen sind standardmäßig deaktiviert. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellungen ändern.

## 19.10. Vacuuming

Diese Parameter steuern das Vacuum-Verhalten. Weitere Informationen zu Zweck und Aufgaben von Vacuum finden Sie in [Abschnitt 24.1](24_Regelmäßige_Datenbankwartung.md#241-routinemäßiges-vacuuming).

### 19.10.1. Automatisches Vacuuming

Diese Einstellungen steuern das Verhalten der Autovacuum-Funktion. Weitere Informationen finden Sie in Abschnitt 24.1.6. Beachten Sie, dass viele dieser Einstellungen pro Tabelle überschrieben werden können; siehe Storage Parameters.

`autovacuum` (`boolean`)

Steuert, ob der Server den Autovacuum-Launcher-Daemon ausführen soll. Dies ist standardmäßig eingeschaltet; allerdings muss auch `track_counts` aktiviert sein, damit Autovacuum funktioniert. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; Autovacuuming kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter deaktiviert werden.

Beachten Sie, dass das System auch dann Autovacuum-Prozesse startet, wenn dieser Parameter deaktiviert ist, falls dies notwendig ist, um Transaction-ID-Wraparound zu verhindern. Weitere Informationen finden Sie in Abschnitt 24.1.5.

`autovacuum_worker_slots` (`integer`)

Gibt die Anzahl der Backend-Slots an, die für Autovacuum-Worker-Prozesse reserviert werden. Der Standardwert beträgt typischerweise 16 Slots, kann aber kleiner sein, wenn Ihre Kernel-Einstellungen dies nicht unterstützen (wie während `initdb` ermittelt). Dieser Parameter kann nur beim Serverstart gesetzt werden.

Wenn Sie diesen Wert ändern, sollten Sie auch `autovacuum_max_workers` anpassen.

`autovacuum_max_workers` (`integer`)

Gibt die maximale Anzahl von Autovacuum-Prozessen an (außer dem Autovacuum Launcher), die gleichzeitig laufen dürfen. Der Standardwert ist 3. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

Beachten Sie, dass eine Einstellung dieses Werts, die höher als `autovacuum_worker_slots` ist, keine Wirkung hat, da Autovacuum-Worker aus dem durch diese Einstellung eingerichteten Slot-Pool entnommen werden.

`autovacuum_naptime` (`integer`)

Gibt die Mindestverzögerung zwischen Autovacuum-Läufen auf einer bestimmten Datenbank an. In jeder Runde untersucht der Daemon die Datenbank und führt bei Bedarf `VACUUM`- und `ANALYZE`-Befehle für Tabellen in dieser Datenbank aus. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Der Standardwert ist eine Minute (`1min`). Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`autovacuum_vacuum_threshold` (`integer`)

Gibt die Mindestanzahl aktualisierter oder gelöschter Tupel an, die benötigt wird, um ein `VACUUM` in einer Tabelle auszulösen. Der Standardwert ist 50 Tupel. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`autovacuum_vacuum_insert_threshold` (`integer`)

Gibt die Anzahl eingefügter Tupel an, die benötigt wird, um ein `VACUUM` in einer Tabelle auszulösen. Der Standardwert ist 1000 Tupel. Wenn `-1` angegeben wird, löst Autovacuum für keine Tabelle eine `VACUUM`-Operation auf Basis der Anzahl der Einfügungen aus. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`autovacuum_analyze_threshold` (`integer`)

Gibt die Mindestanzahl eingefügter, aktualisierter oder gelöschter Tupel an, die benötigt wird, um ein `ANALYZE` in einer Tabelle auszulösen. Der Standardwert ist 50 Tupel. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`autovacuum_vacuum_scale_factor` (`floating point`)

Gibt einen Anteil der Tabellengröße an, der zu `autovacuum_vacuum_threshold` addiert wird, wenn entschieden wird, ob ein `VACUUM` ausgelöst werden soll. Der Standardwert ist `0.2` (20 % der Tabellengröße). Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`autovacuum_vacuum_insert_scale_factor` (`floating point`)

Gibt einen Anteil der nicht eingefrorenen Pages in der Tabelle an, der zu `autovacuum_vacuum_insert_threshold` addiert wird, wenn entschieden wird, ob ein `VACUUM` ausgelöst werden soll. Der Standardwert ist `0.2` (20 % der nicht eingefrorenen Pages in der Tabelle). Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`autovacuum_analyze_scale_factor` (`floating point`)

Gibt einen Anteil der Tabellengröße an, der zu `autovacuum_analyze_threshold` addiert wird, wenn entschieden wird, ob ein `ANALYZE` ausgelöst werden soll. Der Standardwert ist `0.1` (10 % der Tabellengröße). Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`autovacuum_vacuum_max_threshold` (`integer`)

Gibt die maximale Anzahl aktualisierter oder gelöschter Tupel an, die benötigt wird, um ein `VACUUM` in einer Tabelle auszulösen, also eine Grenze für den mit `autovacuum_vacuum_threshold` und `autovacuum_vacuum_scale_factor` berechneten Wert. Der Standardwert ist 100.000.000 Tupel. Wenn `-1` angegeben wird, erzwingt Autovacuum keine maximale Anzahl aktualisierter oder gelöschter Tupel, die eine `VACUUM`-Operation auslöst. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Speicherparameter überschrieben werden.

`autovacuum_freeze_max_age` (`integer`)

Gibt das maximale Alter (in Transaktionen) an, das das Feld `pg_class.relfrozenxid` einer Tabelle erreichen darf, bevor eine `VACUUM`-Operation erzwungen wird, um Transaction-ID-Wraparound innerhalb der Tabelle zu verhindern. Beachten Sie, dass das System Autovacuum-Prozesse startet, um Wraparound zu verhindern, selbst wenn Autovacuum ansonsten deaktiviert ist.

Vacuum ermöglicht außerdem das Entfernen alter Dateien aus dem Unterverzeichnis `pg_xact`, weshalb der Standard relativ niedrige 200 Millionen Transaktionen beträgt. Dieser Parameter kann nur beim Serverstart gesetzt werden, aber die Einstellung kann für einzelne Tabellen durch Ändern der Tabellenspeicherparameter verringert werden. Weitere Informationen finden Sie in Abschnitt 24.1.5.

`autovacuum_multixact_freeze_max_age` (`integer`)

Gibt das maximale Alter (in Multixacts) an, das das Feld `pg_class.relminmxid` einer Tabelle erreichen darf, bevor eine `VACUUM`-Operation erzwungen wird, um Multixact-ID-Wraparound innerhalb der Tabelle zu verhindern. Beachten Sie, dass das System Autovacuum-Prozesse startet, um Wraparound zu verhindern, selbst wenn Autovacuum ansonsten deaktiviert ist.

Das Vacuuming von Multixacts ermöglicht außerdem das Entfernen alter Dateien aus den Unterverzeichnissen `pg_multixact/members` und `pg_multixact/offsets`, weshalb der Standard relativ niedrige 400 Millionen Multixacts beträgt. Dieser Parameter kann nur beim Serverstart gesetzt werden, aber die Einstellung kann für einzelne Tabellen durch Ändern der Tabellenspeicherparameter verringert werden. Weitere Informationen finden Sie in Abschnitt 24.1.5.1.

`autovacuum_vacuum_cost_delay` (`floating point`)

Gibt den Cost-Delay-Wert an, der in automatischen `VACUUM`-Operationen verwendet wird. Wenn `-1` angegeben wird, wird der normale Wert von `vacuum_cost_delay` verwendet. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert ist 2 Millisekunden. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

`autovacuum_vacuum_cost_limit` (`integer`)

Gibt den Cost-Limit-Wert an, der in automatischen `VACUUM`-Operationen verwendet wird. Wenn `-1` angegeben wird (der Standard), wird der normale Wert von `vacuum_cost_limit` verwendet. Beachten Sie, dass der Wert proportional auf die laufenden Autovacuum-Worker verteilt wird, wenn mehr als einer vorhanden ist, sodass die Summe der Limits jedes Workers den Wert dieser Variable nicht überschreitet. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

### 19.10.2. Kostenbasierte Vacuum-Verzögerung

Während der Ausführung von `VACUUM`- und `ANALYZE`-Befehlen führt das System einen internen Zähler, der die geschätzten Kosten der verschiedenen ausgeführten I/O-Operationen verfolgt. Wenn die akkumulierten Kosten ein Limit erreichen (angegeben durch `vacuum_cost_limit`), schläft der die Operation ausführende Prozess für eine kurze Zeitspanne, wie durch `vacuum_cost_delay` angegeben. Dann setzt er den Zähler zurück und setzt die Ausführung fort.

Ziel dieser Funktion ist es, Administratoren zu ermöglichen, die I/O-Auswirkungen dieser Befehle auf gleichzeitige Datenbankaktivität zu verringern. In vielen Situationen ist es nicht wichtig, dass Wartungsbefehle wie `VACUUM` und `ANALYZE` schnell fertig werden; meist ist es jedoch sehr wichtig, dass diese Befehle die Fähigkeit des Systems, andere Datenbankoperationen auszuführen, nicht erheblich beeinträchtigen. Die kostenbasierte Vacuum-Verzögerung bietet Administratoren eine Möglichkeit, dies zu erreichen.

Diese Funktion ist für manuell ausgegebene `VACUUM`-Befehle standardmäßig deaktiviert. Um sie zu aktivieren, setzen Sie die Variable `vacuum_cost_delay` auf einen Wert ungleich null.

`vacuum_cost_delay` (`floating point`)

Die Zeitspanne, die der Prozess schläft, wenn das Kostenlimit überschritten wurde. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert ist 0, wodurch die kostenbasierte Vacuum-Verzögerung deaktiviert wird. Positive Werte aktivieren kostenbasiertes Vacuuming.

Bei Verwendung von kostenbasiertem Vacuuming sind passende Werte für `vacuum_cost_delay` normalerweise recht klein, vielleicht kleiner als 1 Millisekunde. Obwohl `vacuum_cost_delay` auf Werte mit Bruchteilen von Millisekunden gesetzt werden kann, werden solche Verzögerungen auf älteren Plattformen möglicherweise nicht genau gemessen. Auf solchen Plattformen erfordert das Erhöhen des gedrosselten Ressourcenverbrauchs von `VACUUM` über das hinaus, was Sie bei `1ms` erhalten, das Ändern der anderen Vacuum-Kostenparameter. Dennoch sollten Sie `vacuum_cost_delay` so klein halten, wie Ihre Plattform zuverlässig messen kann; große Verzögerungen sind nicht hilfreich.

`vacuum_cost_page_hit` (`integer`)

Die geschätzten Kosten für das Vacuuming eines Buffers, der im Shared-Buffer-Cache gefunden wurde. Dies repräsentiert die Kosten zum Sperren des Buffer Pools, Nachschlagen in der Shared Hash Table und Scannen des Page-Inhalts. Der Standardwert ist 1.

`vacuum_cost_page_miss` (`integer`)

Die geschätzten Kosten für das Vacuuming eines Buffers, der von der Festplatte gelesen werden muss. Dies repräsentiert den Aufwand zum Sperren des Buffer Pools, Nachschlagen in der Shared Hash Table, Einlesen des gewünschten Blocks von der Festplatte und Scannen seines Inhalts. Der Standardwert ist 2.

`vacuum_cost_page_dirty` (`integer`)

Die geschätzten Kosten, die berechnet werden, wenn Vacuum einen Block verändert, der zuvor clean war. Dies repräsentiert das zusätzliche I/O, das erforderlich ist, um den dirty Block wieder auf die Festplatte zu flushen. Der Standardwert ist 20.

`vacuum_cost_limit` (`integer`)

Dies sind die akkumulierten Kosten, die dazu führen, dass der Vacuum-Prozess für `vacuum_cost_delay` schläft. Der Standardwert ist 200.

> Hinweis: Es gibt bestimmte Operationen, die kritische Sperren halten und daher so schnell wie möglich abgeschlossen werden sollten. Während solcher Operationen treten keine kostenbasierten Vacuum-Verzögerungen auf. Daher ist es möglich, dass sich Kosten weit über das angegebene Limit hinaus ansammeln. Um in solchen Fällen nutzlos lange Verzögerungen zu vermeiden, wird die tatsächliche Verzögerung als `vacuum_cost_delay * accumulated_balance / vacuum_cost_limit` berechnet, mit einem Maximum von `vacuum_cost_delay * 4`.

### 19.10.3. Standardverhalten

`vacuum_truncate` (`boolean`)

Aktiviert oder deaktiviert, dass Vacuum versucht, leere Pages am Ende der Tabelle abzuschneiden. Der Standardwert ist `true`. Wenn `true`, führen `VACUUM` und Autovacuum die Kürzung durch, und der Festplattenspeicher der abgeschnittenen Pages wird an das Betriebssystem zurückgegeben. Beachten Sie, dass die Kürzung eine `ACCESS EXCLUSIVE`-Sperre auf der Tabelle erfordert. Der Parameter `TRUNCATE` von `VACUUM`, falls angegeben, überschreibt den Wert dieses Parameters. Die Einstellung kann außerdem für einzelne Tabellen durch Ändern der Tabellenspeicherparameter überschrieben werden.

### 19.10.4. Einfrieren

Um Korrektheit auch nach dem Wraparound von Transaktions-IDs zu erhalten, markiert PostgreSQL ausreichend alte Zeilen als eingefroren. Diese Zeilen sind für alle sichtbar; andere Transaktionen müssen ihre einfügende XID nicht untersuchen, um Sichtbarkeit zu bestimmen. `VACUUM` ist dafür verantwortlich, Zeilen als eingefroren zu markieren. Die folgenden Einstellungen steuern das Freezing-Verhalten von `VACUUM` und sollten anhand der XID-Verbrauchsrate des Systems und der Datenzugriffsmuster der dominierenden Workloads abgestimmt werden. Weitere Informationen zu Transaction-ID-Wraparound und zum Abstimmen dieser Parameter finden Sie in Abschnitt 24.1.5.

`vacuum_freeze_table_age` (`integer`)

`VACUUM` führt einen aggressiven Scan aus, wenn das Feld `pg_class.relfrozenxid` der Tabelle das durch diese Einstellung angegebene Alter erreicht hat. Ein aggressiver Scan unterscheidet sich von einem regulären `VACUUM` dadurch, dass er jede Page besucht, die nicht eingefrorene XIDs oder MXIDs enthalten könnte, nicht nur solche, die tote Tupel enthalten könnten. Der Standardwert ist 150 Millionen Transaktionen. Obwohl Benutzer diesen Wert irgendwo zwischen null und zwei Milliarden setzen können, begrenzt `VACUUM` den effektiven Wert stillschweigend auf 95 % von `autovacuum_freeze_max_age`, damit ein periodisches manuelles `VACUUM` eine Chance hat zu laufen, bevor ein Anti-Wraparound-Autovacuum für die Tabelle gestartet wird. Weitere Informationen finden Sie in Abschnitt 24.1.5.

`vacuum_freeze_min_age` (`integer`)

Gibt das Cutoff-Alter (in Transaktionen) an, das `VACUUM` verwenden soll, um zu entscheiden, ob das Einfrieren von Pages mit einer älteren XID ausgelöst wird. Der Standardwert ist 50 Millionen Transaktionen. Obwohl Benutzer diesen Wert irgendwo zwischen null und einer Milliarde setzen können, begrenzt `VACUUM` den effektiven Wert stillschweigend auf die Hälfte von `autovacuum_freeze_max_age`, damit die Zeit zwischen erzwungenen Autovacuums nicht unangemessen kurz wird. Weitere Informationen finden Sie in Abschnitt 24.1.5.

`vacuum_failsafe_age` (`integer`)

Gibt das maximale Alter (in Transaktionen) an, das das Feld `pg_class.relfrozenxid` einer Tabelle erreichen darf, bevor `VACUUM` außergewöhnliche Maßnahmen ergreift, um einen systemweiten Transaction-ID-Wraparound-Fehler zu vermeiden. Dies ist `VACUUM`s Strategie der letzten Instanz. Der Failsafe löst typischerweise aus, wenn ein Autovacuum zur Verhinderung von Transaction-ID-Wraparound bereits einige Zeit läuft, obwohl er bei jedem `VACUUM` auslösen kann.

Wenn der Failsafe ausgelöst wird, wird jede wirksame kostenbasierte Verzögerung nicht mehr angewendet, weitere nicht wesentliche Wartungsaufgaben (etwa Index-Vacuuming) werden übersprungen, und eine verwendete Buffer Access Strategy wird deaktiviert, sodass `VACUUM` alle Shared Buffers frei nutzen kann.

Der Standardwert ist 1,6 Milliarden Transaktionen. Obwohl Benutzer diesen Wert irgendwo zwischen null und 2,1 Milliarden setzen können, passt `VACUUM` den effektiven Wert stillschweigend auf mindestens 105 % von `autovacuum_freeze_max_age` an.

`vacuum_multixact_freeze_table_age` (`integer`)

`VACUUM` führt einen aggressiven Scan aus, wenn das Feld `pg_class.relminmxid` der Tabelle das durch diese Einstellung angegebene Alter erreicht hat. Ein aggressiver Scan unterscheidet sich von einem regulären `VACUUM` dadurch, dass er jede Page besucht, die nicht eingefrorene XIDs oder MXIDs enthalten könnte, nicht nur solche, die tote Tupel enthalten könnten. Der Standardwert ist 150 Millionen Multixacts. Obwohl Benutzer diesen Wert irgendwo zwischen null und zwei Milliarden setzen können, begrenzt `VACUUM` den effektiven Wert stillschweigend auf 95 % von `autovacuum_multixact_freeze_max_age`, damit ein periodisches manuelles `VACUUM` eine Chance hat zu laufen, bevor ein Anti-Wraparound für die Tabelle gestartet wird. Weitere Informationen finden Sie in Abschnitt 24.1.5.1.

`vacuum_multixact_freeze_min_age` (`integer`)

Gibt das Cutoff-Alter (in Multixacts) an, das `VACUUM` verwenden soll, um zu entscheiden, ob das Einfrieren von Pages mit einer älteren Multixact-ID ausgelöst wird. Der Standardwert ist 5 Millionen Multixacts. Obwohl Benutzer diesen Wert irgendwo zwischen null und einer Milliarde setzen können, begrenzt `VACUUM` den effektiven Wert stillschweigend auf die Hälfte von `autovacuum_multixact_freeze_max_age`, damit die Zeit zwischen erzwungenen Autovacuums nicht unangemessen kurz wird. Weitere Informationen finden Sie in Abschnitt 24.1.5.1.

`vacuum_multixact_failsafe_age` (`integer`)

Gibt das maximale Alter (in Multixacts) an, das das Feld `pg_class.relminmxid` einer Tabelle erreichen darf, bevor `VACUUM` außergewöhnliche Maßnahmen ergreift, um einen systemweiten Multixact-ID-Wraparound-Fehler zu vermeiden. Dies ist `VACUUM`s Strategie der letzten Instanz. Der Failsafe löst typischerweise aus, wenn ein Autovacuum zur Verhinderung von Transaction-ID-Wraparound bereits einige Zeit läuft, obwohl er bei jedem `VACUUM` auslösen kann.

Wenn der Failsafe ausgelöst wird, wird jede wirksame kostenbasierte Verzögerung nicht mehr angewendet, und weitere nicht wesentliche Wartungsaufgaben (etwa Index-Vacuuming) werden übersprungen.

Der Standardwert ist 1,6 Milliarden Multixacts. Obwohl Benutzer diesen Wert irgendwo zwischen null und 2,1 Milliarden setzen können, passt `VACUUM` den effektiven Wert stillschweigend auf mindestens 105 % von `autovacuum_multixact_freeze_max_age` an.

`vacuum_max_eager_freeze_failure_rate` (`floating point`)

Gibt die maximale Anzahl von Pages (als Anteil an allen Pages in der Relation) an, die `VACUUM` scannen darf, ohne sie in der Visibility Map auf all-frozen setzen zu können, bevor eager scanning deaktiviert wird. Ein Wert von 0 deaktiviert eager scanning vollständig. Der Standardwert ist `0.03` (3 %).

Beachten Sie, dass bei aktiviertem eager scanning nur Freeze-Fehlschläge auf das Limit angerechnet werden, nicht erfolgreiches Einfrieren. Erfolgreiche Page-Freezes werden intern auf 20 % der all-visible, aber nicht all-frozen Pages in der Relation begrenzt. Das Begrenzen erfolgreicher Page-Freezes hilft, den Overhead über mehrere normale Vacuums zu amortisieren, und begrenzt den potenziellen Nachteil verschwendeter eager freezes von Pages, die vor dem nächsten aggressiven Vacuum erneut geändert werden.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden; die Einstellung kann jedoch für einzelne Tabellen durch Ändern des entsprechenden Tabellenspeicherparameters überschrieben werden. Weitere Informationen zum Abstimmen des Freezing-Verhaltens von Vacuum finden Sie in Abschnitt 24.1.5.

## 19.11. Standardwerte für Client-Verbindungen

### 19.11.1. Anweisungsverhalten

`client_min_messages` (`enum`)

Steuert, welche Meldungsstufen an den Client gesendet werden. Gültige Werte sind `DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1`, `LOG`, `NOTICE`, `WARNING` und `ERROR`. Jede Stufe umfasst alle nachfolgenden Stufen. Je später die Stufe, desto weniger Meldungen werden gesendet. Der Standardwert ist `NOTICE`. Beachten Sie, dass `LOG` hier einen anderen Rang hat als in `log_min_messages`.

Meldungen der Stufe `INFO` werden immer an den Client gesendet.

`search_path` (`string`)

Diese Variable gibt die Reihenfolge an, in der Schemata durchsucht werden, wenn ein Objekt (Tabelle, Datentyp, Funktion usw.) mit einem einfachen Namen ohne angegebenes Schema referenziert wird. Wenn es Objekte gleichen Namens in verschiedenen Schemata gibt, wird das zuerst im Suchpfad gefundene verwendet. Ein Objekt, das in keinem der Schemata im Suchpfad liegt, kann nur durch Angabe seines enthaltenden Schemas mit einem qualifizierten Namen referenziert werden.

Der Wert für `search_path` muss eine durch Kommas getrennte Liste von Schemanamen sein. Jeder Name, der kein vorhandenes Schema ist oder ein Schema bezeichnet, für das der Benutzer keine `USAGE`-Berechtigung hat, wird stillschweigend ignoriert.

Wenn einer der Listeneinträge der besondere Name `$user` ist, wird das Schema mit dem von `CURRENT_USER` zurückgegebenen Namen eingesetzt, sofern ein solches Schema existiert und der Benutzer dafür `USAGE`-Berechtigung hat. Andernfalls wird `$user` ignoriert.

Das Systemkatalogschema `pg_catalog` wird immer durchsucht, unabhängig davon, ob es im Pfad erwähnt wird oder nicht. Wenn es im Pfad erwähnt wird, wird es in der angegebenen Reihenfolge durchsucht. Wenn `pg_catalog` nicht im Pfad ist, wird es vor den übrigen Pfadeinträgen durchsucht.

Ebenso wird das temporäre Tabellenschema der aktuellen Sitzung, `pg_temp_nnn`, immer durchsucht, wenn es existiert. Es kann mit dem Alias `pg_temp` explizit im Pfad aufgeführt werden. Wenn es nicht im Pfad aufgeführt ist, wird es zuerst durchsucht, sogar vor `pg_catalog`. Das temporäre Schema wird jedoch nur nach Relationsnamen (Tabelle, View, Sequenz usw.) und Datentypnamen durchsucht. Für Funktions- oder Operatornamen wird es niemals durchsucht.

Wenn Objekte ohne Angabe eines bestimmten Zielschema erstellt werden, werden sie im ersten gültigen Schema abgelegt, das in `search_path` genannt ist. Wenn der Suchpfad leer ist, wird ein Fehler gemeldet.

Der Standardwert für diesen Parameter ist `"$user", public`. Diese Einstellung unterstützt die gemeinsame Nutzung einer Datenbank, bei der keine Benutzer private Schemata haben und alle `public` gemeinsam verwenden, private Schemata pro Benutzer und Kombinationen davon. Andere Effekte können erreicht werden, indem die Standard-Suchpfadeinstellung global oder pro Benutzer geändert wird.

Weitere Informationen zur Schemabehandlung finden Sie in [Abschnitt 5.10](05_Datendefinition.md#510-schemata). Insbesondere ist die Standardkonfiguration nur geeignet, wenn die Datenbank einen einzelnen Benutzer oder wenige einander vertrauende Benutzer hat.

Der aktuelle effektive Wert des Suchpfads kann mit der SQL-Funktion `current_schemas` untersucht werden (siehe [Abschnitt 9.27](09_Funktionen_und_Operatoren.md#927-systeminformationsfunktionen-und-operatoren)). Das ist nicht ganz dasselbe wie das Untersuchen des Werts von `search_path`, da `current_schemas` zeigt, wie die in `search_path` auftretenden Einträge aufgelöst wurden.

`row_security` (`boolean`)

Diese Variable steuert, ob statt der Anwendung einer Row-Security-Policy ein Fehler ausgelöst wird. Bei `on` werden Policies normal angewendet. Bei `off` schlagen Abfragen fehl, bei denen andernfalls mindestens eine Policy angewendet würde. Der Standardwert ist `on`. Ändern Sie dies auf `off`, wenn eingeschränkte Zeilensichtbarkeit falsche Ergebnisse verursachen könnte; `pg_dump` nimmt diese Änderung beispielsweise standardmäßig vor. Diese Variable hat keine Wirkung auf Rollen, die jede Row-Security-Policy umgehen, nämlich Superuser und Rollen mit dem Attribut `BYPASSRLS`.

Weitere Informationen zu Row-Security-Policies finden Sie unter `CREATE POLICY`.

`default_table_access_method` (`string`)

Dieser Parameter gibt die Standard-Table-Access-Methode an, die beim Erstellen von Tabellen oder materialisierten Views verwendet wird, wenn der `CREATE`-Befehl keine Access-Methode explizit angibt, oder wenn `SELECT ... INTO` verwendet wird, das keine Angabe einer Table-Access-Methode erlaubt. Der Standard ist `heap`.

`default_tablespace` (`string`)

Diese Variable gibt den Standard-Tablespace an, in dem Objekte (Tabellen und Indizes) erstellt werden, wenn ein `CREATE`-Befehl keinen Tablespace explizit angibt.

Der Wert ist entweder der Name eines Tablespaces oder ein leerer String, der die Verwendung des Standard-Tablespaces der aktuellen Datenbank angibt. Wenn der Wert nicht dem Namen eines vorhandenen Tablespaces entspricht, verwendet PostgreSQL automatisch den Standard-Tablespace der aktuellen Datenbank. Wenn ein nicht standardmäßiger Tablespace angegeben ist, muss der Benutzer dafür das Privileg `CREATE` haben, andernfalls schlagen Erstellungsversuche fehl.

Diese Variable wird nicht für temporäre Tabellen verwendet; für diese wird stattdessen `temp_tablespaces` herangezogen.

Diese Variable wird auch beim Erstellen von Datenbanken nicht verwendet. Standardmäßig erbt eine neue Datenbank ihre Tablespace-Einstellung von der Template-Datenbank, von der sie kopiert wird.

Wenn dieser Parameter beim Erstellen einer partitionierten Tabelle auf einen anderen Wert als den leeren String gesetzt ist, wird der Tablespace der partitionierten Tabelle auf diesen Wert gesetzt; dieser wird als Standard-Tablespace für künftig erstellte Partitionen verwendet, selbst wenn `default_tablespace` sich seitdem geändert hat.

Weitere Informationen zu Tablespaces finden Sie in [Abschnitt 22.6](22_Datenbanken_verwalten.md#226-tablespaces).

`default_toast_compression` (`enum`)

Diese Variable setzt die Standard-TOAST-Kompressionsmethode für Werte komprimierbarer Spalten. Dies kann für einzelne Spalten überschrieben werden, indem die Spaltenoption `COMPRESSION` in `CREATE TABLE` oder `ALTER TABLE` gesetzt wird. Die unterstützten Kompressionsmethoden sind `pglz` und, wenn PostgreSQL mit `--with-lz4` kompiliert wurde, `lz4`. Der Standard ist `pglz`.

`temp_tablespaces` (`string`)

Diese Variable gibt Tablespaces an, in denen temporäre Objekte (temporäre Tabellen und Indizes auf temporären Tabellen) erstellt werden, wenn ein `CREATE`-Befehl keinen Tablespace explizit angibt. Auch temporäre Dateien für Zwecke wie das Sortieren großer Datenmengen werden in diesen Tablespaces erstellt.

Der Wert ist eine Liste von Tablespace-Namen. Wenn mehr als ein Name in der Liste steht, wählt PostgreSQL jedes Mal, wenn ein temporäres Objekt erstellt wird, ein zufälliges Mitglied der Liste; innerhalb einer Transaktion werden nacheinander erstellte temporäre Objekte jedoch nacheinander in die Tablespaces aus der Liste gelegt. Wenn das ausgewählte Listenelement ein leerer String ist, verwendet PostgreSQL stattdessen automatisch den Standard-Tablespace der aktuellen Datenbank.

Wenn `temp_tablespaces` interaktiv gesetzt wird, ist die Angabe eines nicht vorhandenen Tablespaces ein Fehler, ebenso die Angabe eines Tablespaces, für den der Benutzer kein `CREATE`-Privileg hat. Bei Verwendung eines zuvor gesetzten Werts werden nicht vorhandene Tablespaces jedoch ignoriert, ebenso Tablespaces, für die dem Benutzer das `CREATE`-Privileg fehlt. Diese Regel gilt insbesondere bei Verwendung eines in `postgresql.conf` gesetzten Werts.

Der Standardwert ist ein leerer String, was dazu führt, dass alle temporären Objekte im Standard-Tablespace der aktuellen Datenbank erstellt werden.

Siehe auch `default_tablespace`.

`check_function_bodies` (`boolean`)

Dieser Parameter ist normalerweise `on`. Bei `off` deaktiviert er die Validierung des Routine-Body-Strings während `CREATE FUNCTION` und `CREATE PROCEDURE`. Das Deaktivieren der Validierung vermeidet Seiteneffekte des Validierungsprozesses, insbesondere falsche Positivmeldungen durch Probleme wie Vorwärtsreferenzen. Setzen Sie diesen Parameter auf `off`, bevor Sie Funktionen im Namen anderer Benutzer laden; `pg_dump` tut dies automatisch.

`default_transaction_isolation` (`enum`)

Jede SQL-Transaktion hat einen Isolationsgrad, der entweder `read uncommitted`, `read committed`, `repeatable read` oder `serializable` sein kann. Dieser Parameter steuert den Standard-Isolationsgrad jeder neuen Transaktion. Der Standard ist `read committed`.

Weitere Informationen finden Sie in [Kapitel 13](13_Nebenläufigkeitskontrolle.md) und unter `SET TRANSACTION`.

`default_transaction_read_only` (`boolean`)

Eine schreibgeschützte SQL-Transaktion kann keine nicht temporären Tabellen ändern. Dieser Parameter steuert den standardmäßigen Read-only-Status jeder neuen Transaktion. Der Standard ist `off` (read/write).

Weitere Informationen finden Sie unter `SET TRANSACTION`.

`default_transaction_deferrable` (`boolean`)

Beim Ausführen mit dem Isolationsgrad `serializable` kann eine deferrable schreibgeschützte SQL-Transaktion verzögert werden, bevor sie fortfahren darf. Sobald sie jedoch mit der Ausführung beginnt, verursacht sie keinen der Overheads, die zur Sicherstellung der Serialisierbarkeit erforderlich sind; daher hat der Serialisierungscode keinen Grund, sie wegen gleichzeitiger Aktualisierungen abzubrechen. Das macht diese Option für lang laufende schreibgeschützte Transaktionen geeignet.

Dieser Parameter steuert den standardmäßigen Deferrable-Status jeder neuen Transaktion. Er hat derzeit keine Wirkung auf Read-write-Transaktionen oder solche, die mit Isolationsgraden unterhalb von `serializable` arbeiten. Der Standard ist `off`.

Weitere Informationen finden Sie unter `SET TRANSACTION`.

`transaction_isolation` (`enum`)

Dieser Parameter spiegelt den Isolationsgrad der aktuellen Transaktion wider. Zu Beginn jeder Transaktion wird er auf den aktuellen Wert von `default_transaction_isolation` gesetzt. Jeder spätere Versuch, ihn zu ändern, entspricht einem `SET TRANSACTION`-Befehl.

`transaction_read_only` (`boolean`)

Dieser Parameter spiegelt den Read-only-Status der aktuellen Transaktion wider. Zu Beginn jeder Transaktion wird er auf den aktuellen Wert von `default_transaction_read_only` gesetzt. Jeder spätere Versuch, ihn zu ändern, entspricht einem `SET TRANSACTION`-Befehl.

`transaction_deferrable` (`boolean`)

Dieser Parameter spiegelt den Deferrability-Status der aktuellen Transaktion wider. Zu Beginn jeder Transaktion wird er auf den aktuellen Wert von `default_transaction_deferrable` gesetzt. Jeder spätere Versuch, ihn zu ändern, entspricht einem `SET TRANSACTION`-Befehl.

`session_replication_role` (`enum`)

Steuert das Auslösen replikationsbezogener Trigger und Rules für die aktuelle Sitzung. Mögliche Werte sind `origin` (der Standard), `replica` und `local`. Das Setzen dieses Parameters führt dazu, dass zuvor gecachte Abfragepläne verworfen werden. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Die beabsichtigte Verwendung dieser Einstellung besteht darin, dass logische Replikationssysteme sie auf `replica` setzen, wenn sie replizierte Änderungen anwenden. Die Wirkung besteht darin, dass Trigger und Rules, die nicht von ihrer Standardkonfiguration abweichen, auf der Replica nicht ausgelöst werden. Weitere Informationen finden Sie in den `ALTER TABLE`-Klauseln `ENABLE TRIGGER` und `ENABLE RULE`.

PostgreSQL behandelt die Einstellungen `origin` und `local` intern gleich. Replikationssysteme von Drittanbietern können diese beiden Werte für interne Zwecke verwenden, zum Beispiel `local`, um eine Sitzung zu kennzeichnen, deren Änderungen nicht repliziert werden sollen.

Da Foreign Keys als Trigger implementiert sind, deaktiviert das Setzen dieses Parameters auf `replica` auch alle Foreign-Key-Prüfungen, was bei unsachgemäßer Verwendung Daten in einen inkonsistenten Zustand bringen kann.

`statement_timeout` (`integer`)

Bricht jede Anweisung ab, die länger als die angegebene Zeitspanne benötigt. Wenn `log_min_error_statement` auf `ERROR` oder niedriger gesetzt ist, wird die Anweisung, bei der das Timeout auftrat, ebenfalls geloggt. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Ein Wert von null (der Standard) deaktiviert das Timeout.

Das Timeout wird ab dem Zeitpunkt gemessen, zu dem ein Befehl beim Server eintrifft, bis zu seinem Abschluss durch den Server. Wenn mehrere SQL-Anweisungen in einer einzelnen Simple-Query-Meldung erscheinen, wird das Timeout auf jede Anweisung separat angewendet. (PostgreSQL-Versionen vor 13 behandelten das Timeout normalerweise so, als gelte es für den gesamten Query-String.) Beim Extended Query Protocol beginnt das Timeout zu laufen, wenn eine abfragebezogene Meldung (`Parse`, `Bind`, `Execute`, `Describe`) eintrifft, und es wird durch den Abschluss einer `Execute`- oder `Sync`-Meldung aufgehoben.

Das Setzen von `statement_timeout` in `postgresql.conf` wird nicht empfohlen, weil es alle Sitzungen betreffen würde.

`transaction_timeout` (`integer`)

Beendet jede Sitzung, die länger als die angegebene Zeitspanne in einer Transaktion verbringt. Das Limit gilt sowohl für explizite Transaktionen (mit `BEGIN` gestartet) als auch für eine implizit gestartete Transaktion, die einer einzelnen Anweisung entspricht. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Ein Wert von null (der Standard) deaktiviert das Timeout.

Wenn `transaction_timeout` kürzer oder gleich `idle_in_transaction_session_timeout` oder `statement_timeout` ist, wird das längere Timeout ignoriert.

Das Setzen von `transaction_timeout` in `postgresql.conf` wird nicht empfohlen, weil es alle Sitzungen betreffen würde.

> Hinweis: Prepared Transactions unterliegen diesem Timeout nicht.

`lock_timeout` (`integer`)

Bricht jede Anweisung ab, die länger als die angegebene Zeitspanne wartet, während sie versucht, eine Sperre auf einer Tabelle, einem Index, einer Zeile oder einem anderen Datenbankobjekt zu erwerben. Das Zeitlimit gilt separat für jeden Sperrerwerbsversuch. Das Limit gilt sowohl für explizite Sperranforderungen wie `LOCK TABLE` oder `SELECT FOR UPDATE` ohne `NOWAIT` als auch für implizit erworbene Sperren. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Ein Wert von null (der Standard) deaktiviert das Timeout.

Anders als `statement_timeout` kann dieses Timeout nur beim Warten auf Sperren auftreten. Beachten Sie: Wenn `statement_timeout` ungleich null ist, ist es ziemlich sinnlos, `lock_timeout` auf denselben oder einen größeren Wert zu setzen, da das Statement-Timeout immer zuerst auslösen würde. Wenn `log_min_error_statement` auf `ERROR` oder niedriger gesetzt ist, wird die Anweisung, bei der das Timeout auftrat, geloggt.

Das Setzen von `lock_timeout` in `postgresql.conf` wird nicht empfohlen, weil es alle Sitzungen betreffen würde.

`idle_in_transaction_session_timeout` (`integer`)

Beendet jede Sitzung, die innerhalb einer offenen Transaktion länger als die angegebene Zeitspanne idle war, also auf eine Client-Abfrage wartete. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Ein Wert von null (der Standard) deaktiviert das Timeout.

Diese Option kann verwendet werden, um sicherzustellen, dass idle Sitzungen Sperren nicht unangemessen lange halten. Selbst wenn keine wesentlichen Sperren gehalten werden, verhindert eine offene Transaktion, dass Vacuum kürzlich tote Tupel entfernt, die möglicherweise nur für diese Transaktion sichtbar sind; daher kann langes Idle-Bleiben zu Table Bloat beitragen. Details siehe [Abschnitt 24.1](24_Regelmäßige_Datenbankwartung.md#241-routinemäßiges-vacuuming).

`idle_session_timeout` (`integer`)

Beendet jede Sitzung, die länger als die angegebene Zeitspanne idle war, also auf eine Client-Abfrage wartete, aber nicht innerhalb einer offenen Transaktion. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Ein Wert von null (der Standard) deaktiviert das Timeout.

Anders als bei einer offenen Transaktion verursacht eine idle Sitzung ohne Transaktion keine hohen Kosten für den Server; daher ist es weniger nötig, dieses Timeout zu aktivieren als `idle_in_transaction_session_timeout`.

Seien Sie vorsichtig, wenn Sie dieses Timeout für Verbindungen erzwingen, die über Connection-Pooling-Software oder andere Middleware hergestellt werden, da eine solche Schicht möglicherweise schlecht auf unerwartetes Schließen von Verbindungen reagiert. Es kann hilfreich sein, dieses Timeout nur für interaktive Sitzungen zu aktivieren, vielleicht indem es nur auf bestimmte Benutzer angewendet wird.

`bytea_output` (`enum`)

Setzt das Ausgabeformat für Werte des Typs `bytea`. Gültige Werte sind `hex` (der Standard) und `escape` (das traditionelle PostgreSQL-Format). Weitere Informationen finden Sie in [Abschnitt 8.4](08_Datentypen.md#84-binärdatentypen). Der Typ `bytea` akzeptiert bei der Eingabe immer beide Formate, unabhängig von dieser Einstellung.

`xmlbinary` (`enum`)

Setzt, wie Binärwerte in XML kodiert werden. Dies gilt zum Beispiel, wenn `bytea`-Werte durch die Funktionen `xmlelement` oder `xmlforest` in XML konvertiert werden. Mögliche Werte sind `base64` und `hex`, die beide im XML-Schema-Standard definiert sind. Der Standard ist `base64`. Weitere Informationen zu XML-bezogenen Funktionen finden Sie in [Abschnitt 9.15](09_Funktionen_und_Operatoren.md#915-xmlfunktionen).

Die tatsächliche Wahl ist hier größtenteils Geschmackssache und nur durch mögliche Einschränkungen in Client-Anwendungen begrenzt. Beide Methoden unterstützen alle möglichen Werte, obwohl die Hex-Kodierung etwas größer ist als die Base64-Kodierung.

`xmloption` (`enum`)

Setzt, ob beim Konvertieren zwischen XML- und Zeichenkettenwerten implizit `DOCUMENT` oder `CONTENT` gilt. Eine Beschreibung finden Sie in [Abschnitt 8.13](08_Datentypen.md#813-xmltyp). Gültige Werte sind `DOCUMENT` und `CONTENT`. Der Standard ist `CONTENT`.

Gemäß SQL-Standard lautet der Befehl zum Setzen dieser Option:

```sql
SET XML OPTION { DOCUMENT | CONTENT };
```

Diese Syntax ist auch in PostgreSQL verfügbar.

`gin_pending_list_limit` (`integer`)

Setzt die maximale Größe der Pending List eines GIN-Index, die verwendet wird, wenn `fastupdate` aktiviert ist. Wenn die Liste über diese maximale Größe hinauswächst, wird sie bereinigt, indem die enthaltenen Einträge in einem Batch in die Haupt-GIN-Datenstruktur des Index verschoben werden. Wird dieser Wert ohne Einheit angegeben, wird er als Kilobyte interpretiert. Der Standardwert ist vier Megabyte (`4MB`). Diese Einstellung kann für einzelne GIN-Indizes durch Ändern der Indexspeicherparameter überschrieben werden. Weitere Informationen finden Sie in Abschnitt 65.4.4.1 und Abschnitt 65.4.5.

`createrole_self_grant` (`string`)

Wenn ein Benutzer, der `CREATEROLE`, aber nicht `SUPERUSER` hat, eine Rolle erstellt und dieser Parameter auf einen nichtleeren Wert gesetzt ist, wird die neu erstellte Rolle dem erstellenden Benutzer mit den angegebenen Optionen gewährt. Der Wert muss `set`, `inherit` oder eine durch Kommas getrennte Liste dieser Werte sein. Der Standardwert ist ein leerer String, der die Funktion deaktiviert.

Zweck dieser Option ist es, einem `CREATEROLE`-Benutzer, der kein Superuser ist, zu erlauben, automatisch zu erben oder automatisch die Fähigkeit zu erhalten, `SET ROLE` auf erstellte Benutzer anzuwenden. Da einem `CREATEROLE`-Benutzer für erstellte Rollen immer implizit `ADMIN OPTION` gewährt wird, könnte dieser Benutzer immer eine `GRANT`-Anweisung ausführen, die dieselbe Wirkung wie diese Einstellung erzielt. Aus Gründen der Benutzbarkeit kann es jedoch praktisch sein, wenn das Grant automatisch geschieht. Ein Superuser erbt automatisch die Privilegien jeder Rolle und kann immer `SET ROLE` auf jede Rolle anwenden; diese Einstellung kann verwendet werden, um ein ähnliches Verhalten für `CREATEROLE`-Benutzer für von ihnen erstellte Benutzer zu erzeugen.

`event_triggers` (`boolean`)

Erlaubt das vorübergehende Deaktivieren der Ausführung von Event Triggers, um fehlerhafte Event Triggers zu untersuchen und zu reparieren. Alle Event Triggers werden deaktiviert, wenn der Wert auf `false` gesetzt wird. Das Setzen auf `true` erlaubt, dass alle Event Triggers ausgelöst werden; dies ist der Standardwert. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`restrict_nonsystem_relation_kind` (`string`)

Setzt Relationstypen, für die der Zugriff auf Nicht-System-Relationen verboten ist. Der Wert hat die Form einer durch Kommas getrennten Liste von Relationstypen. Derzeit werden die Relationstypen `view` und `foreign-table` unterstützt.

### 19.11.2. Locale und Formatierung

`DateStyle` (`string`)

Setzt das Anzeigeformat für Datums- und Zeitwerte sowie die Regeln zur Interpretation mehrdeutiger Datumseingaben. Aus historischen Gründen enthält diese Variable zwei unabhängige Komponenten: die Ausgabeformatspezifikation (`ISO`, `Postgres`, `SQL` oder `German`) und die Ein-/Ausgabespezifikation für die Reihenfolge von Jahr, Monat und Tag (`DMY`, `MDY` oder `YMD`). Diese können getrennt oder gemeinsam gesetzt werden. Die Schlüsselwörter `Euro` und `European` sind Synonyme für `DMY`; die Schlüsselwörter `US`, `NonEuro` und `NonEuropean` sind Synonyme für `MDY`. Weitere Informationen finden Sie in [Abschnitt 8.5](08_Datentypen.md#85-datumszeittypen). Der eingebaute Standardwert ist `ISO, MDY`, aber `initdb` initialisiert die Konfigurationsdatei mit einer Einstellung, die dem Verhalten der gewählten Locale `lc_time` entspricht.

`IntervalStyle` (`enum`)

Setzt das Anzeigeformat für Intervallwerte. Der Wert `sql_standard` erzeugt eine Ausgabe, die SQL-Standard-Intervallliteralen entspricht. Der Wert `postgres` (der Standardwert) erzeugt eine Ausgabe, die PostgreSQL-Versionen vor 8.4 entspricht, wenn der Parameter `DateStyle` auf `ISO` gesetzt war. Der Wert `postgres_verbose` erzeugt eine Ausgabe, die PostgreSQL-Versionen vor 8.4 entspricht, wenn `DateStyle` auf eine Nicht-ISO-Ausgabe gesetzt war. Der Wert `iso_8601` erzeugt eine Ausgabe, die dem in Abschnitt 4.4.3.2 von ISO 8601 definierten Zeitintervall-„Format mit Designatoren“ entspricht.

Der Parameter `IntervalStyle` beeinflusst außerdem die Interpretation mehrdeutiger Intervalleingaben. Weitere Informationen finden Sie in Abschnitt 8.5.4.

`TimeZone` (`string`)

Setzt die Zeitzone zum Anzeigen und Interpretieren von Zeitstempeln. Der eingebaute Standardwert ist `GMT`, wird aber typischerweise in `postgresql.conf` überschrieben; `initdb` legt dort eine Einstellung an, die seiner Systemumgebung entspricht. Weitere Informationen finden Sie in Abschnitt 8.5.3.

`timezone_abbreviations` (`string`)

Setzt die Sammlung zusätzlicher Zeitzonenabkürzungen, die der Server bei Datums-/Zeiteingaben akzeptiert (zusätzlich zu den durch die aktuelle Einstellung `TimeZone` definierten Abkürzungen). Der Standardwert ist `'Default'`, eine Sammlung, die in den meisten Teilen der Welt funktioniert; außerdem gibt es `'Australia'` und `'India'`, und für eine bestimmte Installation können weitere Sammlungen definiert werden. Weitere Informationen finden Sie in Abschnitt B.4.

`extra_float_digits` (`integer`)

Dieser Parameter passt die Anzahl der Stellen an, die für die textuelle Ausgabe von Gleitkommawerten verwendet werden, einschließlich `float4`, `float8` und geometrischer Datentypen.

Wenn der Wert `1` (der Standardwert) oder höher ist, werden Float-Werte im kürzesten präzisen Format ausgegeben; siehe Abschnitt 8.1.3. Die tatsächlich erzeugte Anzahl von Stellen hängt nur vom ausgegebenen Wert ab, nicht vom Wert dieses Parameters. Für `float8`-Werte sind höchstens 17 Stellen erforderlich, für `float4`-Werte 9. Dieses Format ist sowohl schnell als auch präzise und bewahrt den ursprünglichen binären Float-Wert exakt, wenn er korrekt wieder eingelesen wird. Aus Gründen historischer Kompatibilität sind Werte bis `3` erlaubt.

Wenn der Wert null oder negativ ist, wird die Ausgabe auf eine bestimmte Dezimalpräzision gerundet. Die verwendete Präzision ist die Standardanzahl von Stellen für den Typ (`FLT_DIG` beziehungsweise `DBL_DIG`), reduziert entsprechend dem Wert dieses Parameters. Beispielsweise führt die Angabe von `-1` dazu, dass `float4`-Werte auf 5 signifikante Stellen und `float8`-Werte auf 14 Stellen gerundet ausgegeben werden. Dieses Format ist langsamer und bewahrt nicht alle Bits des binären Float-Werts, kann aber besser menschenlesbar sein.

> Hinweis: Die Bedeutung dieses Parameters und sein Standardwert haben sich in PostgreSQL 12 geändert; siehe Abschnitt 8.1.3 für eine weitere Erörterung.

`client_encoding` (`string`)

Setzt die clientseitige Kodierung (den Zeichensatz). Der Standard ist die Verwendung der Datenbankkodierung. Die vom PostgreSQL-Server unterstützten Zeichensätze werden in Abschnitt 23.3.1 beschrieben.

`lc_messages` (`string`)

Setzt die Sprache, in der Meldungen angezeigt werden. Zulässige Werte sind systemabhängig; weitere Informationen finden Sie in [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung). Wenn diese Variable auf den leeren String gesetzt ist (der Standardwert), wird der Wert systemabhängig aus der Ausführungsumgebung des Servers geerbt.

Auf manchen Systemen existiert diese Locale-Kategorie nicht. Das Setzen dieser Variable funktioniert trotzdem, hat aber keine Wirkung. Außerdem besteht die Möglichkeit, dass keine übersetzten Meldungen für die gewünschte Sprache vorhanden sind. In diesem Fall sehen Sie weiterhin englische Meldungen.

Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`lc_monetary` (`string`)

Setzt die Locale, die für die Formatierung von Geldbeträgen verwendet wird, zum Beispiel mit der Funktionsfamilie `to_char`. Zulässige Werte sind systemabhängig; weitere Informationen finden Sie in [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung). Wenn diese Variable auf den leeren String gesetzt ist (der Standardwert), wird der Wert systemabhängig aus der Ausführungsumgebung des Servers geerbt.

`lc_numeric` (`string`)

Setzt die Locale, die für die Formatierung von Zahlen verwendet wird, zum Beispiel mit der Funktionsfamilie `to_char`. Zulässige Werte sind systemabhängig; weitere Informationen finden Sie in [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung). Wenn diese Variable auf den leeren String gesetzt ist (der Standardwert), wird der Wert systemabhängig aus der Ausführungsumgebung des Servers geerbt.

`lc_time` (`string`)

Setzt die Locale, die für die Formatierung von Datums- und Zeitwerten verwendet wird, zum Beispiel mit der Funktionsfamilie `to_char`. Zulässige Werte sind systemabhängig; weitere Informationen finden Sie in [Abschnitt 23.1](23_Lokalisierung.md#231-localeunterstützung). Wenn diese Variable auf den leeren String gesetzt ist (der Standardwert), wird der Wert systemabhängig aus der Ausführungsumgebung des Servers geerbt.

`icu_validation_level` (`enum`)

Steuert, welche Meldungsstufe verwendet wird, wenn Probleme bei der ICU-Locale-Validierung auftreten. Gültige Werte sind `DISABLED`, `DEBUG5`, `DEBUG4`, `DEBUG3`, `DEBUG2`, `DEBUG1`, `INFO`, `NOTICE`, `WARNING`, `ERROR` und `LOG`.

Wenn der Wert auf `DISABLED` gesetzt ist, werden Validierungsprobleme gar nicht gemeldet. Andernfalls werden Probleme mit der angegebenen Meldungsstufe gemeldet. Der Standardwert ist `WARNING`.

`default_text_search_config` (`string`)

Wählt die Textsuche-Konfiguration aus, die von den Varianten der Textsuche-Funktionen verwendet wird, die kein explizites Argument zur Angabe der Konfiguration haben. Weitere Informationen finden Sie in [Kapitel 12](12_Volltextsuche.md). Der eingebaute Standardwert ist `pg_catalog.simple`, aber `initdb` initialisiert die Konfigurationsdatei mit einer Einstellung, die der gewählten Locale `lc_ctype` entspricht, wenn eine zu dieser Locale passende Konfiguration identifiziert werden kann.

### 19.11.3. Vorladen gemeinsam genutzter Bibliotheken

Mehrere Einstellungen stehen zum Vorladen gemeinsam genutzter Bibliotheken in den Server zur Verfügung, um zusätzliche Funktionalität zu laden oder Performancevorteile zu erzielen. Beispielsweise würde die Einstellung `'$libdir/mylib'` dazu führen, dass `mylib.so` (oder auf manchen Plattformen `mylib.sl`) aus dem standardmäßigen Bibliotheksverzeichnis der Installation vorgeladen wird. Die Einstellungen unterscheiden sich darin, wann sie wirksam werden und welche Privilegien erforderlich sind, um sie zu ändern.

PostgreSQL-Bibliotheken für prozedurale Sprachen können auf diese Weise vorgeladen werden, typischerweise mit der Syntax `'$libdir/plXXX'`, wobei `XXX` für `pgsql`, `perl`, `tcl` oder `python` steht.

Nur gemeinsam genutzte Bibliotheken, die ausdrücklich für die Verwendung mit PostgreSQL bestimmt sind, können auf diese Weise geladen werden. Jede von PostgreSQL unterstützte Bibliothek besitzt einen „Magic Block“, der geprüft wird, um Kompatibilität zu garantieren. Aus diesem Grund können Nicht-PostgreSQL-Bibliotheken nicht auf diese Weise geladen werden. Möglicherweise können Sie dafür Betriebssystemmechanismen wie `LD_PRELOAD` verwenden.

Im Allgemeinen sollten Sie in der Dokumentation des jeweiligen Moduls nachsehen, welche Art des Ladens für dieses Modul empfohlen wird.

`local_preload_libraries` (`string`)

Diese Variable gibt eine oder mehrere gemeinsam genutzte Bibliotheken an, die beim Verbindungsstart vorgeladen werden sollen. Sie enthält eine durch Kommas getrennte Liste von Bibliotheksnamen, wobei jeder Name wie beim Befehl `LOAD` interpretiert wird. Whitespace zwischen Einträgen wird ignoriert; setzen Sie einen Bibliotheksnamen in doppelte Anführungszeichen, wenn der Name Whitespace oder Kommas enthalten muss. Der Parameterwert wird nur beim Start der Verbindung wirksam. Spätere Änderungen haben keine Wirkung. Wenn eine angegebene Bibliothek nicht gefunden wird, schlägt der Verbindungsversuch fehl.

Diese Option kann von jedem Benutzer gesetzt werden. Deshalb sind die ladbaren Bibliotheken auf diejenigen beschränkt, die im Unterverzeichnis `plugins` des standardmäßigen Bibliotheksverzeichnisses der Installation erscheinen. (Es liegt in der Verantwortung des Datenbankadministrators, sicherzustellen, dass dort nur „sichere“ Bibliotheken installiert sind.) Einträge in `local_preload_libraries` können dieses Verzeichnis explizit angeben, zum Beispiel `$libdir/plugins/mylib`, oder nur den Bibliotheksnamen angeben; `mylib` hätte dieselbe Wirkung wie `$libdir/plugins/mylib`.

Ziel dieser Funktion ist es, nicht privilegierten Benutzern zu erlauben, Debugging- oder Performance-Messbibliotheken in bestimmte Sitzungen zu laden, ohne dass ein expliziter `LOAD`-Befehl erforderlich ist. Dafür wäre es typisch, diesen Parameter mit der Umgebungsvariable `PGOPTIONS` auf dem Client oder mit `ALTER ROLE SET` zu setzen.

Wenn ein Modul jedoch nicht ausdrücklich dafür entworfen ist, auf diese Weise von Nicht-Superusern verwendet zu werden, ist dies normalerweise nicht die richtige Einstellung. Sehen Sie stattdessen `session_preload_libraries` an.

`session_preload_libraries` (`string`)

Diese Variable gibt eine oder mehrere gemeinsam genutzte Bibliotheken an, die beim Verbindungsstart vorgeladen werden sollen. Sie enthält eine durch Kommas getrennte Liste von Bibliotheksnamen, wobei jeder Name wie beim Befehl `LOAD` interpretiert wird. Whitespace zwischen Einträgen wird ignoriert; setzen Sie einen Bibliotheksnamen in doppelte Anführungszeichen, wenn der Name Whitespace oder Kommas enthalten muss. Der Parameterwert wird nur beim Start der Verbindung wirksam. Spätere Änderungen haben keine Wirkung. Wenn eine angegebene Bibliothek nicht gefunden wird, schlägt der Verbindungsversuch fehl. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Ziel dieser Funktion ist es, Debugging- oder Performance-Messbibliotheken in bestimmte Sitzungen zu laden, ohne dass ein expliziter `LOAD`-Befehl angegeben wird. Beispielsweise könnte `auto_explain` für alle Sitzungen unter einem bestimmten Benutzernamen aktiviert werden, indem dieser Parameter mit `ALTER ROLE SET` gesetzt wird. Außerdem kann dieser Parameter ohne Serverneustart geändert werden (Änderungen werden aber erst wirksam, wenn eine neue Sitzung gestartet wird), sodass neue Module auf diese Weise leichter hinzugefügt werden können, selbst wenn sie für alle Sitzungen gelten sollen.

Anders als bei `shared_preload_libraries` gibt es keinen großen Performancevorteil, eine Bibliothek beim Sitzungsstart statt bei ihrer ersten Verwendung zu laden. Bei Verwendung von Connection Pooling gibt es allerdings einen gewissen Vorteil.

`shared_preload_libraries` (`string`)

Diese Variable gibt eine oder mehrere gemeinsam genutzte Bibliotheken an, die beim Serverstart vorgeladen werden sollen. Sie enthält eine durch Kommas getrennte Liste von Bibliotheksnamen, wobei jeder Name wie beim Befehl `LOAD` interpretiert wird. Whitespace zwischen Einträgen wird ignoriert; setzen Sie einen Bibliotheksnamen in doppelte Anführungszeichen, wenn der Name Whitespace oder Kommas enthalten muss. Dieser Parameter kann nur beim Serverstart gesetzt werden. Wenn eine angegebene Bibliothek nicht gefunden wird, startet der Server nicht.

Einige Bibliotheken müssen bestimmte Operationen ausführen, die nur beim Start des Postmasters stattfinden können, etwa Shared Memory reservieren, Lightweight Locks reservieren oder Background Worker starten. Solche Bibliotheken müssen beim Serverstart über diesen Parameter geladen werden. Details finden Sie in der Dokumentation der jeweiligen Bibliothek.

Auch andere Bibliotheken können vorgeladen werden. Durch das Vorladen einer gemeinsam genutzten Bibliothek entfällt die Startzeit der Bibliothek bei ihrer ersten Verwendung. Allerdings kann sich die Startzeit jedes neuen Serverprozesses leicht erhöhen, selbst wenn dieser Prozess die Bibliothek nie verwendet. Daher wird dieser Parameter nur für Bibliotheken empfohlen, die in den meisten Sitzungen verwendet werden. Außerdem erfordert das Ändern dieses Parameters einen Serverneustart; daher ist dies zum Beispiel nicht die richtige Einstellung für kurzfristige Debugging-Aufgaben. Verwenden Sie dafür stattdessen `session_preload_libraries`.

> Hinweis: Auf Windows-Hosts reduziert das Vorladen einer Bibliothek beim Serverstart nicht die Zeit, die zum Start jedes neuen Serverprozesses erforderlich ist; jeder Serverprozess lädt alle vorgeladenen Bibliotheken erneut. `shared_preload_libraries` ist auf Windows-Hosts jedoch weiterhin nützlich für Bibliotheken, die Operationen zur Postmaster-Startzeit ausführen müssen.

`jit_provider` (`string`)

Diese Variable ist der Name der zu verwendenden JIT-Provider-Bibliothek (siehe Abschnitt 30.4.2). Der Standardwert ist `llvmjit`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Wenn er auf eine nicht vorhandene Bibliothek gesetzt wird, ist JIT nicht verfügbar, aber es wird kein Fehler ausgelöst. Dadurch kann JIT-Unterstützung getrennt vom Hauptpaket von PostgreSQL installiert werden.

### 19.11.4. Sonstige Standardwerte

`dynamic_library_path` (`string`)

Wenn ein dynamisch ladbares Modul geöffnet werden muss und der im Befehl `CREATE FUNCTION` oder `LOAD` angegebene Dateiname keine Verzeichniskomponente enthält (das heißt, der Name enthält keinen Schrägstrich), durchsucht das System diesen Pfad nach der benötigten Datei.

Der Wert von `dynamic_library_path` muss eine Liste absoluter Verzeichnispfade sein, getrennt durch Doppelpunkte (oder Semikolons unter Windows). Wenn ein Listenelement mit dem besonderen String `$libdir` beginnt, wird `$libdir` durch das einkompilierte Bibliotheksverzeichnis des PostgreSQL-Pakets ersetzt; dort sind die Module installiert, die von der Standarddistribution von PostgreSQL bereitgestellt werden. (Verwenden Sie `pg_config --pkglibdir`, um den Namen dieses Verzeichnisses herauszufinden.) Zum Beispiel:

```text
dynamic_library_path = '/usr/local/lib/postgresql:/home/my_project/lib:$libdir'
```

Oder in einer Windows-Umgebung:

```text
dynamic_library_path = 'C:\tools\postgresql;H:\my_project\lib;$libdir'
```

Der Standardwert für diesen Parameter ist `'$libdir'`. Wenn der Wert auf einen leeren String gesetzt wird, wird die automatische Pfadsuche abgeschaltet.

Dieser Parameter kann zur Laufzeit von Superusern und Benutzern mit dem entsprechenden `SET`-Privileg geändert werden, aber eine so vorgenommene Einstellung bleibt nur bis zum Ende der Client-Verbindung bestehen; daher sollte diese Methode Entwicklungszwecken vorbehalten bleiben. Die empfohlene Art, diesen Parameter zu setzen, ist die Konfigurationsdatei `postgresql.conf`.

`extension_control_path` (`string`)

Ein Pfad, in dem nach Erweiterungen gesucht wird, genauer nach Extension-Control-Dateien (`name.control`). Die verbleibenden Erweiterungsskripte und sekundären Control-Dateien werden dann aus demselben Verzeichnis geladen, in dem die primäre Control-Datei gefunden wurde. Details finden Sie in Abschnitt 36.17.1.

Der Wert von `extension_control_path` muss eine Liste absoluter Verzeichnispfade sein, getrennt durch Doppelpunkte (oder Semikolons unter Windows). Wenn ein Listenelement mit dem besonderen String `$system` beginnt, wird `$system` durch das einkompilierte Extension-Verzeichnis von PostgreSQL ersetzt; dort sind die Extensions installiert, die von der Standarddistribution von PostgreSQL bereitgestellt werden. (Verwenden Sie `pg_config --sharedir`, um den Namen dieses Verzeichnisses herauszufinden.) Zum Beispiel:

```text
extension_control_path = '/usr/local/share/postgresql:/home/my_project/share:$system'
```

Oder in einer Windows-Umgebung:

```text
extension_control_path = 'C:\tools\postgresql;H:\my_project\share;$system'
```

Beachten Sie, dass von den angegebenen Pfadelementen erwartet wird, dass sie ein Unterverzeichnis `extension` haben, das die Dateien `.control` und `.sql` enthält; das Suffix `extension` wird automatisch an jedes Pfadelement angehängt.

Der Standardwert für diesen Parameter ist `'$system'`. Wenn der Wert auf einen leeren String gesetzt wird, wird ebenfalls der Standardwert `'$system'` angenommen.

Wenn Extensions mit gleichem Namen in mehreren Verzeichnissen des konfigurierten Pfads vorhanden sind, wird nur die zuerst im Pfad gefundene Instanz verwendet.

Dieser Parameter kann zur Laufzeit von Superusern und Benutzern mit dem entsprechenden `SET`-Privileg geändert werden, aber eine so vorgenommene Einstellung bleibt nur bis zum Ende der Client-Verbindung bestehen; daher sollte diese Methode Entwicklungszwecken vorbehalten bleiben. Die empfohlene Art, diesen Parameter zu setzen, ist die Konfigurationsdatei `postgresql.conf`.

Beachten Sie: Wenn Sie diesen Parameter setzen, um Extensions aus nicht standardmäßigen Orten laden zu können, müssen Sie höchstwahrscheinlich auch `dynamic_library_path` auf einen entsprechenden Ort setzen, zum Beispiel:

```text
extension_control_path = '/usr/local/share/postgresql:$system'
dynamic_library_path = '/usr/local/lib/postgresql:$libdir'
```

`gin_fuzzy_search_limit` (`integer`)

Weiche Obergrenze für die Größe der von GIN-Index-Scans zurückgegebenen Menge. Weitere Informationen finden Sie in Abschnitt 65.4.5.

## 19.12. Sperrverwaltung

`deadlock_timeout` (`integer`)

Dies ist die Zeitspanne, die beim Warten auf eine Sperre verstreicht, bevor geprüft wird, ob eine Deadlock-Situation vorliegt. Die Prüfung auf Deadlocks ist relativ teuer, daher führt der Server sie nicht jedes Mal aus, wenn er auf eine Sperre wartet. Optimistisch wird angenommen, dass Deadlocks in Produktionsanwendungen nicht häufig sind, und daher wird zunächst eine Weile auf die Sperre gewartet, bevor auf einen Deadlock geprüft wird. Das Erhöhen dieses Werts verringert die Zeit, die durch unnötige Deadlock-Prüfungen verschwendet wird, verzögert aber die Meldung echter Deadlock-Fehler. Wird dieser Wert ohne Einheit angegeben, wird er als Millisekunden interpretiert. Der Standardwert ist eine Sekunde (`1s`), was in der Praxis wahrscheinlich ungefähr der kleinste sinnvolle Wert ist. Auf einem stark ausgelasteten Server möchten Sie ihn möglicherweise erhöhen. Idealerweise sollte die Einstellung Ihre typische Transaktionsdauer überschreiten, um die Wahrscheinlichkeit zu erhöhen, dass eine Sperre freigegeben wird, bevor der Wartende entscheidet, auf einen Deadlock zu prüfen. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Wenn `log_lock_waits` gesetzt ist, bestimmt dieser Parameter außerdem die Zeitspanne, die gewartet wird, bevor eine Logmeldung über das Warten auf eine Sperre ausgegeben wird. Wenn Sie Sperrverzögerungen untersuchen möchten, kann es sinnvoll sein, `deadlock_timeout` kürzer als normal zu setzen.

`max_locks_per_transaction` (`integer`)

Die gemeinsame Sperrtabelle hat Platz für `max_locks_per_transaction` Objekte (zum Beispiel Tabellen) pro Serverprozess oder Prepared Transaction; daher können zu einem Zeitpunkt nicht mehr als so viele verschiedene Objekte gesperrt werden. Dieser Parameter begrenzt die durchschnittliche Anzahl von Objektsperren, die von jeder Transaktion verwendet werden; einzelne Transaktionen können mehr Objekte sperren, solange die Sperren aller Transaktionen in die Sperrtabelle passen. Dies ist nicht die Anzahl der sperrbaren Zeilen; dieser Wert ist unbegrenzt. Der Standardwert `64` hat sich historisch als ausreichend erwiesen, aber Sie müssen diesen Wert möglicherweise erhöhen, wenn Sie Abfragen haben, die in einer einzelnen Transaktion viele verschiedene Tabellen berühren, zum Beispiel eine Abfrage einer Elterntabelle mit vielen Kindern. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Beim Betrieb eines Standby-Servers müssen Sie diesen Parameter auf denselben oder einen höheren Wert setzen wie auf dem Primärserver. Andernfalls werden Abfragen auf dem Standby-Server nicht zugelassen.

`max_pred_locks_per_transaction` (`integer`)

Die gemeinsame Predicate-Lock-Tabelle hat Platz für `max_pred_locks_per_transaction` Objekte (zum Beispiel Tabellen) pro Serverprozess oder Prepared Transaction; daher können zu einem Zeitpunkt nicht mehr als so viele verschiedene Objekte gesperrt werden. Dieser Parameter begrenzt die durchschnittliche Anzahl von Objektsperren, die von jeder Transaktion verwendet werden; einzelne Transaktionen können mehr Objekte sperren, solange die Sperren aller Transaktionen in die Sperrtabelle passen. Dies ist nicht die Anzahl der sperrbaren Zeilen; dieser Wert ist unbegrenzt. Der Standardwert `64` hat sich historisch als ausreichend erwiesen, aber Sie müssen diesen Wert möglicherweise erhöhen, wenn Sie Clients haben, die in einer einzelnen serialisierbaren Transaktion viele verschiedene Tabellen berühren. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`max_pred_locks_per_relation` (`integer`)

Dieser Parameter steuert, wie viele Pages oder Tupel einer einzelnen Relation mit Predicate Locks gesperrt werden können, bevor die Sperre so hochgestuft wird, dass sie die gesamte Relation abdeckt. Werte größer oder gleich null bedeuten ein absolutes Limit, während negative Werte `max_pred_locks_per_transaction` geteilt durch den Absolutwert dieser Einstellung bedeuten. Der Standardwert ist `-2`, wodurch das Verhalten früherer PostgreSQL-Versionen beibehalten wird. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`max_pred_locks_per_page` (`integer`)

Dieser Parameter steuert, wie viele Zeilen auf einer einzelnen Page mit Predicate Locks gesperrt werden können, bevor die Sperre so hochgestuft wird, dass sie die gesamte Page abdeckt. Der Standardwert ist `2`. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

## 19.13. Versions- und Plattformkompatibilität

### 19.13.1. Frühere PostgreSQL-Versionen

`array_nulls` (`boolean`)

Dieser Parameter steuert, ob der Array-Eingabeparser ein nicht in Anführungszeichen gesetztes `NULL` als Angabe eines Null-Array-Elements erkennt. Standardmäßig ist dies aktiviert, sodass Array-Werte mit Nullwerten eingegeben werden können. PostgreSQL-Versionen vor 8.2 unterstützten jedoch keine Nullwerte in Arrays und behandelten `NULL` daher als Angabe eines normalen Array-Elements mit dem Stringwert „NULL“. Für die Abwärtskompatibilität mit Anwendungen, die das alte Verhalten benötigen, kann diese Variable ausgeschaltet werden.

Beachten Sie, dass es möglich ist, Array-Werte mit Nullwerten zu erzeugen, selbst wenn diese Variable ausgeschaltet ist.

`backslash_quote` (`enum`)

Dieser Parameter steuert, ob ein Anführungszeichen in einem Stringliteral durch `\'` dargestellt werden kann. Die bevorzugte, SQL-standardkonforme Art, ein Anführungszeichen darzustellen, besteht darin, es zu verdoppeln (`''`), aber PostgreSQL hat historisch auch `\'` akzeptiert. Die Verwendung von `\'` erzeugt jedoch Sicherheitsrisiken, weil es in einigen Client-Zeichensatzkodierungen Multibyte-Zeichen gibt, deren letztes Byte numerisch ASCII `\` entspricht. Wenn clientseitiger Code Escaping falsch ausführt, ist ein SQL-Injection-Angriff möglich. Dieses Risiko kann verhindert werden, indem der Server Abfragen zurückweist, in denen ein Anführungszeichen offenbar durch einen Backslash escaped wird. Die zulässigen Werte von `backslash_quote` sind `on` (`\'` immer erlauben), `off` (immer zurückweisen) und `safe_encoding` (nur erlauben, wenn die Client-Kodierung ASCII `\` innerhalb eines Multibyte-Zeichens nicht erlaubt). `safe_encoding` ist die Standardeinstellung.

Beachten Sie, dass in einem standardkonformen Stringliteral `\` ohnehin einfach `\` bedeutet. Dieser Parameter betrifft nur die Behandlung nicht standardkonformer Literale, einschließlich Escape-String-Syntax (`E'...'`).

`escape_string_warning` (`boolean`)

Wenn aktiviert, wird eine Warnung ausgegeben, wenn ein Backslash (`\`) in einem gewöhnlichen Stringliteral (`'...'`-Syntax) erscheint und `standard_conforming_strings` ausgeschaltet ist. Der Standardwert ist `on`.

Anwendungen, die Backslash als Escape-Zeichen verwenden möchten, sollten auf Escape-String-Syntax (`E'...'`) umgestellt werden, weil gewöhnliche Strings Backslash standardmäßig gemäß SQL-Standard als normales Zeichen behandeln. Diese Variable kann aktiviert werden, um Code zu finden, der geändert werden muss.

`lo_compat_privileges` (`boolean`)

In PostgreSQL-Versionen vor 9.0 hatten Large Objects keine Zugriffsprivilegien und waren daher für alle Benutzer immer lesbar und schreibbar. Das Setzen dieser Variable auf `on` deaktiviert die neuen Privilegprüfungen zur Kompatibilität mit früheren Versionen. Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

Das Setzen dieser Variable deaktiviert nicht alle Sicherheitsprüfungen im Zusammenhang mit Large Objects, sondern nur diejenigen, deren Standardverhalten sich in PostgreSQL 9.0 geändert hat.

`quote_all_identifiers` (`boolean`)

Wenn die Datenbank SQL erzeugt, erzwingt dieser Parameter, dass alle Bezeichner quotiert werden, selbst wenn sie (derzeit) keine Schlüsselwörter sind. Dies betrifft sowohl die Ausgabe von `EXPLAIN` als auch die Ergebnisse von Funktionen wie `pg_get_viewdef`. Siehe auch die Option `--quote-all-identifiers` von `pg_dump` und `pg_dumpall`.

`standard_conforming_strings` (`boolean`)

Dieser Parameter steuert, ob gewöhnliche Stringliterale (`'...'`) Backslashes wörtlich behandeln, wie es der SQL-Standard vorgibt. Seit PostgreSQL 9.1 ist der Standardwert `on` (frühere Versionen hatten standardmäßig `off`). Anwendungen können diesen Parameter prüfen, um festzustellen, wie Stringliterale verarbeitet werden. Das Vorhandensein dieses Parameters kann außerdem als Hinweis darauf verstanden werden, dass die Escape-String-Syntax (`E'...'`) unterstützt wird. Escape-String-Syntax (Abschnitt 4.1.2.2) sollte verwendet werden, wenn eine Anwendung möchte, dass Backslashes als Escape-Zeichen behandelt werden.

`synchronize_seqscans` (`boolean`)

Dieser Parameter erlaubt es sequenziellen Scans großer Tabellen, sich miteinander zu synchronisieren, sodass gleichzeitige Scans ungefähr zur selben Zeit denselben Block lesen und sich dadurch die I/O-Last teilen. Wenn dies aktiviert ist, kann ein Scan in der Mitte der Tabelle beginnen und dann am Ende „umbiegen“, um alle Zeilen abzudecken, sodass er sich mit bereits laufenden Scans synchronisiert. Dies kann zu unvorhersehbaren Änderungen in der Reihenfolge der von Abfragen ohne `ORDER BY`-Klausel zurückgegebenen Zeilen führen. Das Setzen dieses Parameters auf `off` stellt das Verhalten vor 8.3 sicher, bei dem ein sequenzieller Scan immer am Anfang der Tabelle beginnt. Der Standardwert ist `on`.

### 19.13.2. Plattform- und Client-Kompatibilität

`transform_null_equals` (`boolean`)

Wenn aktiviert, werden Ausdrücke der Form `expr = NULL` (oder `NULL = expr`) als `expr IS NULL` behandelt, das heißt, sie geben wahr zurück, wenn `expr` zum Nullwert ausgewertet wird, andernfalls falsch. Das korrekte, SQL-spezifikationskonforme Verhalten von `expr = NULL` besteht darin, immer null (unbekannt) zurückzugeben. Daher ist dieser Parameter standardmäßig ausgeschaltet.

Gefilterte Formulare in Microsoft Access erzeugen jedoch Abfragen, die offenbar `expr = NULL` verwenden, um auf Nullwerte zu testen. Wenn Sie diese Schnittstelle für den Zugriff auf die Datenbank verwenden, möchten Sie diese Option daher möglicherweise aktivieren. Da Ausdrücke der Form `expr = NULL` bei SQL-standardgemäßer Interpretation immer den Nullwert zurückgeben, sind sie nicht sehr nützlich und kommen in normalen Anwendungen selten vor, sodass diese Option in der Praxis wenig Schaden anrichtet. Neue Benutzer sind jedoch häufig durch die Semantik von Ausdrücken mit Nullwerten verwirrt, weshalb diese Option standardmäßig ausgeschaltet ist.

Beachten Sie, dass diese Option nur die exakte Form `= NULL` betrifft, nicht andere Vergleichsoperatoren oder andere Ausdrücke, die rechnerisch einem Ausdruck mit dem Gleichheitsoperator entsprechen (etwa `IN`). Daher ist diese Option keine allgemeine Korrektur für schlechte Programmierung.

Verwandte Informationen finden Sie in [Abschnitt 9.2](09_Funktionen_und_Operatoren.md#92-vergleichsfunktionen-und-operatoren).

`allow_alter_system` (`boolean`)

Wenn `allow_alter_system` auf `off` gesetzt ist, wird ein Fehler zurückgegeben, wenn der Befehl `ALTER SYSTEM` ausgeführt wird. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden. Der Standardwert ist `on`.

Beachten Sie, dass diese Einstellung nicht als Sicherheitsfunktion betrachtet werden darf. Sie deaktiviert nur den Befehl `ALTER SYSTEM`. Sie verhindert nicht, dass ein Superuser die Konfiguration mit anderen SQL-Befehlen ändert. Ein Superuser hat viele Möglichkeiten, Shell-Befehle auf Betriebssystemebene auszuführen, und kann daher `postgresql.auto.conf` unabhängig vom Wert dieser Einstellung ändern.

Das Ausschalten dieser Einstellung ist für Umgebungen gedacht, in denen die Konfiguration von PostgreSQL durch ein externes Werkzeug verwaltet wird. In solchen Umgebungen könnte ein gutmeinender Superuser versehentlich `ALTER SYSTEM` verwenden, um die Konfiguration zu ändern, statt das externe Werkzeug zu benutzen. Das kann zu unbeabsichtigtem Verhalten führen, zum Beispiel dazu, dass das externe Werkzeug die Änderung zu einem späteren Zeitpunkt überschreibt, wenn es die Konfiguration aktualisiert. Das Setzen dieses Parameters auf `off` kann helfen, solche Fehler zu vermeiden.

Dieser Parameter steuert nur die Verwendung von `ALTER SYSTEM`. Die in `postgresql.auto.conf` gespeicherten Einstellungen werden wirksam, selbst wenn `allow_alter_system` auf `off` gesetzt ist.

## 19.14. Fehlerbehandlung

`exit_on_error` (`boolean`)

Wenn aktiviert, beendet jeder Fehler die aktuelle Sitzung. Standardmäßig ist dies ausgeschaltet, sodass nur `FATAL`-Fehler die Sitzung beenden.

`restart_after_crash` (`boolean`)

Wenn dieser Parameter auf `on` gesetzt ist, was dem Standard entspricht, initialisiert sich PostgreSQL nach einem Backend-Absturz automatisch neu. Diesen Wert auf `on` zu belassen, ist normalerweise die beste Methode, um die Verfügbarkeit der Datenbank zu maximieren. Unter bestimmten Umständen, etwa wenn PostgreSQL von Clusterware gestartet wird, kann es jedoch sinnvoll sein, den Neustart zu deaktivieren, damit die Clusterware die Kontrolle übernehmen und die von ihr für angemessen gehaltenen Aktionen ausführen kann.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`data_sync_retry` (`boolean`)

Wenn dieser Parameter auf `off` gesetzt ist, was dem Standard entspricht, löst PostgreSQL einen Fehler der Stufe `PANIC` aus, wenn geänderte Datendateien nicht in das Dateisystem geschrieben werden können. Dadurch stürzt der Datenbankserver ab. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Auf manchen Betriebssystemen ist der Status von Daten im Page Cache des Kernels nach einem Write-back-Fehler unbekannt. In einigen Fällen könnten sie vollständig vergessen worden sein, sodass ein erneuter Versuch unsicher wäre; der zweite Versuch kann als erfolgreich gemeldet werden, obwohl die Daten tatsächlich verloren gegangen sind. Unter diesen Umständen besteht die einzige Möglichkeit, Datenverlust zu vermeiden, darin, nach jedem gemeldeten Fehler aus dem WAL wiederherzustellen, vorzugsweise nachdem die Ursache des Fehlers untersucht und defekte Hardware ersetzt wurde.

Wenn dieser Parameter auf `on` gesetzt ist, meldet PostgreSQL stattdessen einen Fehler, läuft aber weiter, sodass der Datenflush-Vorgang in einem späteren Checkpoint erneut versucht werden kann. Setzen Sie ihn nur auf `on`, nachdem Sie untersucht haben, wie das Betriebssystem gepufferte Daten bei Write-back-Fehlern behandelt.

`recovery_init_sync_method` (`enum`)

Wenn dieser Parameter auf `fsync` gesetzt ist, was dem Standard entspricht, öffnet und synchronisiert PostgreSQL rekursiv alle Dateien im Datenverzeichnis, bevor die Crash Recovery beginnt. Die Dateisuche folgt symbolischen Links für das WAL-Verzeichnis und jeden konfigurierten Tablespace (aber keinen anderen symbolischen Links). Dies soll sicherstellen, dass alle WAL- und Datendateien dauerhaft auf der Platte gespeichert sind, bevor Änderungen erneut abgespielt werden. Dies gilt immer dann, wenn ein Datenbankcluster gestartet wird, der nicht sauber heruntergefahren wurde, einschließlich Kopien, die mit `pg_basebackup` erstellt wurden.

Unter Linux kann stattdessen `syncfs` verwendet werden, um das Betriebssystem aufzufordern, die Dateisysteme zu synchronisieren, die das Datenverzeichnis, die WAL-Dateien und jeden Tablespace enthalten (aber keine anderen Dateisysteme, die möglicherweise über symbolische Links erreichbar sind). Dies kann viel schneller sein als die Einstellung `fsync`, weil nicht jede Datei einzeln geöffnet werden muss. Andererseits kann es langsamer sein, wenn ein Dateisystem mit anderen Anwendungen geteilt wird, die viele Dateien ändern, da auch diese Dateien auf die Platte geschrieben werden. Außerdem werden auf Linux-Versionen vor 5.8 I/O-Fehler, die beim Schreiben von Daten auf die Platte auftreten, möglicherweise nicht an PostgreSQL gemeldet; relevante Fehlermeldungen erscheinen dann eventuell nur in Kernel-Logs.

Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

## 19.15. Voreingestellte Optionen

Die folgenden „Parameter“ sind schreibgeschützt. Deshalb wurden sie aus der Beispiel-Datei `postgresql.conf` ausgeschlossen. Diese Optionen berichten verschiedene Aspekte des PostgreSQL-Verhaltens, die für bestimmte Anwendungen interessant sein können, insbesondere für administrative Frontends. Die meisten von ihnen werden festgelegt, wenn PostgreSQL kompiliert oder installiert wird.

`block_size` (`integer`)

Meldet die Größe eines Plattenblocks. Sie wird beim Bau des Servers durch den Wert von `BLCKSZ` bestimmt. Der Standardwert ist `8192` Byte. Die Bedeutung einiger Konfigurationsvariablen (etwa `shared_buffers`) wird durch `block_size` beeinflusst. Informationen finden Sie in [Abschnitt 19.4](19_Serverkonfiguration.md#194-ressourcenverbrauch).

`data_checksums` (`boolean`)

Meldet, ob Datenprüfsummen für diesen Cluster aktiviert sind. Weitere Informationen finden Sie bei `-k`.

`data_directory_mode` (`integer`)

Auf Unix-Systemen meldet dieser Parameter die Berechtigungen, die das Datenverzeichnis (definiert durch `data_directory`) beim Serverstart hatte. (Unter Microsoft Windows zeigt dieser Parameter immer `0700` an.) Weitere Informationen finden Sie bei der Option `initdb -g`.

`debug_assertions` (`boolean`)

Meldet, ob PostgreSQL mit aktivierten Assertions gebaut wurde. Das ist der Fall, wenn beim Bau von PostgreSQL das Makro `USE_ASSERT_CHECKING` definiert ist (zum Beispiel durch die `configure`-Option `--enable-cassert`). Standardmäßig wird PostgreSQL ohne Assertions gebaut.

`huge_pages_status` (`enum`)

Meldet den Zustand von Huge Pages in der aktuellen Instanz: `on`, `off` oder `unknown` (wenn mit `postgres -C` angezeigt). Dieser Parameter ist nützlich, um festzustellen, ob die Allokation von Huge Pages unter `huge_pages=try` erfolgreich war. Weitere Informationen finden Sie bei `huge_pages`.

`integer_datetimes` (`boolean`)

Meldet, ob PostgreSQL mit Unterstützung für 64-Bit-Integer-Datums- und Zeitwerte gebaut wurde. Seit PostgreSQL 10 ist dies immer `on`.

`in_hot_standby` (`boolean`)

Meldet, ob sich der Server derzeit im Hot-Standby-Modus befindet. Wenn dies aktiviert ist, werden alle Transaktionen erzwungen schreibgeschützt. Innerhalb einer Sitzung kann sich dies nur ändern, wenn der Server zum Primärserver befördert wird. Weitere Informationen finden Sie in [Abschnitt 26.4](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md#264-hot-standby).

`max_function_args` (`integer`)

Meldet die maximale Anzahl von Funktionsargumenten. Sie wird beim Bau des Servers durch den Wert von `FUNC_MAX_ARGS` bestimmt. Der Standardwert ist `100` Argumente.

`max_identifier_length` (`integer`)

Meldet die maximale Bezeichnerlänge. Sie wird als eins weniger als der Wert von `NAMEDATALEN` beim Bau des Servers bestimmt. Der Standardwert von `NAMEDATALEN` ist `64`; daher beträgt der Standardwert von `max_identifier_length` `63` Byte, was bei Verwendung von Multibyte-Kodierungen weniger als 63 Zeichen sein kann.

`max_index_keys` (`integer`)

Meldet die maximale Anzahl von Indexschlüsseln. Sie wird beim Bau des Servers durch den Wert von `INDEX_MAX_KEYS` bestimmt. Der Standardwert ist `32` Schlüssel.

`num_os_semaphores` (`integer`)

Meldet die Anzahl der Semaphore, die der Server auf Grundlage der konfigurierten Anzahl erlaubter Verbindungen (`max_connections`), erlaubter Autovacuum-Worker-Prozesse (`autovacuum_max_workers`), erlaubter WAL-Sender-Prozesse (`max_wal_senders`), erlaubter Hintergrundprozesse (`max_worker_processes`) usw. benötigt.

`segment_size` (`integer`)

Meldet die Anzahl von Blöcken (Pages), die in einem Dateisegment gespeichert werden können. Sie wird beim Bau des Servers durch den Wert von `RELSEG_SIZE` bestimmt. Die maximale Größe einer Segmentdatei in Byte entspricht `segment_size` multipliziert mit `block_size`; standardmäßig ist dies `1GB`.

`server_encoding` (`string`)

Meldet die Datenbankkodierung (den Zeichensatz). Sie wird beim Erstellen der Datenbank bestimmt. Normalerweise müssen Clients sich nur um den Wert von `client_encoding` kümmern.

`server_version` (`string`)

Meldet die Versionsnummer des Servers. Sie wird beim Bau des Servers durch den Wert von `PG_VERSION` bestimmt.

`server_version_num` (`integer`)

Meldet die Versionsnummer des Servers als Integer. Sie wird beim Bau des Servers durch den Wert von `PG_VERSION_NUM` bestimmt.

`shared_memory_size` (`integer`)

Meldet die Größe des Hauptbereichs des Shared Memory, auf das nächste Megabyte aufgerundet.

`shared_memory_size_in_huge_pages` (`integer`)

Meldet die Anzahl von Huge Pages, die für den Hauptbereich des Shared Memory auf Grundlage der angegebenen `huge_page_size` benötigt werden. Wenn Huge Pages nicht unterstützt werden, ist dies `-1`.

Diese Einstellung wird nur unter Linux unterstützt. Auf anderen Plattformen ist sie immer auf `-1` gesetzt. Weitere Details zur Verwendung von Huge Pages unter Linux finden Sie in Abschnitt 18.4.5.

`ssl_library` (`string`)

Meldet den Namen der SSL-Bibliothek, mit der dieser PostgreSQL-Server gebaut wurde (selbst wenn SSL in dieser Instanz derzeit nicht konfiguriert oder verwendet wird), zum Beispiel `OpenSSL`, oder einen leeren String, wenn keine vorhanden ist.

`wal_block_size` (`integer`)

Meldet die Größe eines WAL-Plattenblocks. Sie wird beim Bau des Servers durch den Wert von `XLOG_BLCKSZ` bestimmt. Der Standardwert ist `8192` Byte.

`wal_segment_size` (`integer`)

Meldet die Größe von Write-Ahead-Log-Segmenten. Der Standardwert ist `16MB`. Weitere Informationen finden Sie in [Abschnitt 28.5](28_Zuverlässigkeit_und_Write_Ahead_Log.md#285-walkonfiguration).

## 19.16. Angepasste Optionen

Diese Funktion wurde entworfen, um Parametern, die PostgreSQL normalerweise nicht kennt, das Hinzufügen durch Add-on-Module (etwa prozedurale Sprachen) zu erlauben. Dadurch können Erweiterungsmodule auf die üblichen Arten konfiguriert werden.

Angepasste Optionen haben zweiteilige Namen: einen Extension-Namen, dann einen Punkt, dann den eigentlichen Parameternamen, ähnlich wie qualifizierte Namen in SQL. Ein Beispiel ist `plpgsql.variable_conflict`.

Da angepasste Optionen möglicherweise in Prozessen gesetzt werden müssen, die das betreffende Extension-Modul noch nicht geladen haben, akzeptiert PostgreSQL eine Einstellung für jeden zweiteiligen Parameternamen. Solche Variablen werden als Platzhalter behandelt und haben keine Funktion, bis das Modul geladen wird, das sie definiert. Wenn ein Extension-Modul geladen wird, fügt es seine Variablendefinitionen hinzu und konvertiert alle Platzhalterwerte entsprechend diesen Definitionen. Wenn es nicht erkannte Platzhalter gibt, die mit seinem Extension-Namen beginnen, werden Warnungen ausgegeben und diese Platzhalter entfernt.

## 19.17. Entwickleroptionen

Die folgenden Parameter sind für Entwicklertests gedacht und sollten niemals in einer Produktionsdatenbank verwendet werden. Einige von ihnen können jedoch bei der Wiederherstellung schwer beschädigter Datenbanken helfen. Daher wurden sie aus der Beispiel-Datei `postgresql.conf` ausgeschlossen. Beachten Sie, dass viele dieser Parameter besondere Kompilierungsflags im Quellcode benötigen, um überhaupt zu funktionieren.

`allow_in_place_tablespaces` (`boolean`)

Erlaubt, Tablespaces als Verzeichnisse innerhalb von `pg_tblspc` zu erstellen, wenn dem Befehl `CREATE TABLESPACE` ein leerer Location-String übergeben wird. Dies soll das Testen von Replikationsszenarien ermöglichen, in denen Primär- und Standby-Server auf derselben Maschine laufen. Solche Verzeichnisse werden Backup-Werkzeuge wahrscheinlich verwirren, die an dieser Stelle nur symbolische Links erwarten. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`allow_system_table_mods` (`boolean`)

Erlaubt das Ändern der Struktur von Systemtabellen sowie bestimmte andere riskante Aktionen an Systemtabellen. Dies ist sonst selbst Superusern nicht erlaubt. Unüberlegte Verwendung dieser Einstellung kann zu unwiederbringlichem Datenverlust führen oder das Datenbanksystem schwer beschädigen. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`backtrace_functions` (`string`)

Dieser Parameter enthält eine durch Kommas getrennte Liste von C-Funktionsnamen. Wenn ein Fehler ausgelöst wird und der Name der internen C-Funktion, in der der Fehler auftritt, mit einem Wert in der Liste übereinstimmt, wird zusammen mit der Fehlermeldung ein Backtrace in das Serverlog geschrieben. Dies kann verwendet werden, um bestimmte Bereiche des Quellcodes zu debuggen.

Backtrace-Unterstützung ist nicht auf allen Plattformen verfügbar, und die Qualität der Backtraces hängt von den Kompilierungsoptionen ab.

Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`debug_copy_parse_plan_trees` (`boolean`)

Das Aktivieren dieses Parameters erzwingt, dass alle Parse- und Plan-Bäume durch `copyObject()` laufen, um Fehler und Auslassungen in `copyObject()` leichter zu finden. Der Standardwert ist `off`.

Dieser Parameter ist nur verfügbar, wenn `DEBUG_NODE_TESTS_ENABLED` zur Kompilierungszeit definiert wurde (was automatisch geschieht, wenn die `configure`-Option `--enable-cassert` verwendet wird).

`debug_discard_caches` (`integer`)

Bei einem Wert von `1` wird jeder Systemkatalog-Cache-Eintrag bei der ersten möglichen Gelegenheit invalidiert, unabhängig davon, ob tatsächlich etwas passiert ist, das ihn ungültig machen würde. Das Caching von Systemkatalogen ist dadurch faktisch deaktiviert, sodass der Server extrem langsam läuft. Höhere Werte führen die Cache-Invalidierung rekursiv aus, was noch langsamer ist und nur zum Testen der Caching-Logik selbst nützlich ist. Der Standardwert `0` wählt das normale Katalog-Caching-Verhalten.

Dieser Parameter kann sehr hilfreich sein, wenn schwer reproduzierbare Fehler im Zusammenhang mit gleichzeitigen Katalogänderungen ausgelöst werden sollen; ansonsten wird er selten benötigt. Details finden Sie in den Quellcodedateien `inval.c` und `pg_config_manual.h`.

Dieser Parameter wird unterstützt, wenn `DISCARD_CACHES_ENABLED` zur Kompilierungszeit definiert wurde (was automatisch geschieht, wenn die `configure`-Option `--enable-cassert` verwendet wird). In Produktions-Builds ist sein Wert immer `0`, und Versuche, ihn auf einen anderen Wert zu setzen, lösen einen Fehler aus.

`debug_io_direct` (`string`)

Fordert den Kernel auf, Caching-Effekte für Relationsdaten und WAL-Dateien mit `O_DIRECT` (die meisten Unix-ähnlichen Systeme), `F_NOCACHE` (macOS) oder `FILE_FLAG_NO_BUFFERING` (Windows) zu minimieren.

Kann auf einen leeren String (den Standardwert) gesetzt werden, um die Verwendung von direktem I/O zu deaktivieren, oder auf eine durch Kommas getrennte Liste von Operationen, die direkten I/O verwenden sollen. Die gültigen Optionen sind `data` für Hauptdatendateien, `wal` für WAL-Dateien und `wal_init` für WAL-Dateien bei ihrer ersten Allokation. Dieser Parameter kann nur beim Serverstart gesetzt werden.

Einige Betriebssysteme und Dateisysteme unterstützen direkten I/O nicht, daher können vom Standard abweichende Einstellungen beim Start zurückgewiesen werden oder Fehler verursachen.

Derzeit reduziert diese Funktion die Performance und ist nur für Entwicklertests gedacht.

`debug_parallel_query` (`enum`)

Erlaubt die Verwendung paralleler Abfragen zu Testzwecken selbst in Fällen, in denen kein Performancevorteil erwartet wird. Die zulässigen Werte von `debug_parallel_query` sind `off` (Parallelmodus nur verwenden, wenn eine Performanceverbesserung erwartet wird), `on` (parallele Abfrage für alle Abfragen erzwingen, für die dies als sicher gilt) und `regress` (wie `on`, aber mit zusätzlichen Verhaltensänderungen wie unten erläutert).

Genauer gesagt fügt das Setzen dieses Werts auf `on` einen `Gather`-Knoten an die Spitze jedes Abfrageplans hinzu, für den dies sicher erscheint, sodass die Abfrage innerhalb eines Parallel Workers läuft. Selbst wenn kein Parallel Worker verfügbar ist oder verwendet werden kann, werden Operationen wie das Starten einer Subtransaktion, die in einem parallelen Abfragekontext verboten wären, verboten, sofern der Planner nicht glaubt, dass dies dazu führen wird, dass die Abfrage fehlschlägt. Wenn bei gesetzter Option Fehler oder unerwartete Ergebnisse auftreten, müssen einige von der Abfrage verwendete Funktionen möglicherweise als `PARALLEL UNSAFE` (oder möglicherweise `PARALLEL RESTRICTED`) markiert werden.

Das Setzen dieses Werts auf `regress` hat alle Wirkungen von `on` plus einige zusätzliche Effekte, die automatisierte Regressionstests erleichtern sollen. Normalerweise enthalten Meldungen von einem Parallel Worker eine Kontextzeile, die darauf hinweist; die Einstellung `regress` unterdrückt diese Zeile, sodass die Ausgabe dieselbe ist wie bei nicht paralleler Ausführung. Außerdem werden die durch diese Einstellung zu Plänen hinzugefügten `Gather`-Knoten in der `EXPLAIN`-Ausgabe verborgen, damit die Ausgabe dem entspricht, was bei ausgeschalteter Einstellung erzeugt würde.

`debug_raw_expression_coverage_test` (`boolean`)

Das Aktivieren dieses Parameters erzwingt, dass alle rohen Parse-Bäume für DML-Anweisungen von `raw_expression_tree_walker()` gescannt werden, um Fehler und Auslassungen in dieser Funktion leichter zu finden. Der Standardwert ist `off`.

Dieser Parameter ist nur verfügbar, wenn `DEBUG_NODE_TESTS_ENABLED` zur Kompilierungszeit definiert wurde (was automatisch geschieht, wenn die `configure`-Option `--enable-cassert` verwendet wird).

`debug_write_read_parse_plan_trees` (`boolean`)

Das Aktivieren dieses Parameters erzwingt, dass alle Parse- und Plan-Bäume durch `outfuncs.c`/`readfuncs.c` laufen, um Fehler und Auslassungen in diesen Modulen leichter zu finden. Der Standardwert ist `off`.

Dieser Parameter ist nur verfügbar, wenn `DEBUG_NODE_TESTS_ENABLED` zur Kompilierungszeit definiert wurde (was automatisch geschieht, wenn die `configure`-Option `--enable-cassert` verwendet wird).

`ignore_system_indexes` (`boolean`)

Ignoriert Systemindizes beim Lesen von Systemtabellen (aktualisiert die Indizes aber weiterhin beim Ändern der Tabellen). Dies ist nützlich bei der Wiederherstellung von beschädigten Systemindizes. Dieser Parameter kann nach Sitzungsstart nicht geändert werden.

`post_auth_delay` (`integer`)

Die Zeitspanne, um die verzögert wird, wenn ein neuer Serverprozess gestartet wurde, nachdem er die Authentifizierungsprozedur durchgeführt hat. Dies soll Entwicklern die Gelegenheit geben, mit einem Debugger an den Serverprozess anzudocken. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Ein Wert von null (der Standardwert) deaktiviert die Verzögerung. Dieser Parameter kann nach Sitzungsstart nicht geändert werden.

`pre_auth_delay` (`integer`)

Die Zeitspanne, um die unmittelbar nach dem Forken eines neuen Serverprozesses verzögert wird, bevor dieser die Authentifizierungsprozedur durchführt. Dies soll Entwicklern die Gelegenheit geben, mit einem Debugger an den Serverprozess anzudocken, um Fehlverhalten bei der Authentifizierung aufzuspüren. Wird dieser Wert ohne Einheit angegeben, wird er als Sekunden interpretiert. Ein Wert von null (der Standardwert) deaktiviert die Verzögerung. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`trace_notify` (`boolean`)

Erzeugt eine große Menge Debugging-Ausgabe für die Befehle `LISTEN` und `NOTIFY`. `client_min_messages` beziehungsweise `log_min_messages` müssen `DEBUG1` oder niedriger sein, damit diese Ausgabe an den Client beziehungsweise in die Serverlogs gesendet wird.

`trace_sort` (`boolean`)

Wenn aktiviert, werden Informationen über die Ressourcennutzung während Sortieroperationen ausgegeben.

`trace_locks` (`boolean`)

Wenn aktiviert, werden Informationen über die Sperrnutzung ausgegeben. Die ausgegebenen Informationen umfassen die Art der Sperroperation, den Sperrtyp und die eindeutige Kennung des Objekts, das gesperrt oder entsperrt wird. Ebenfalls enthalten sind Bitmasken für die Sperrtypen, die für dieses Objekt bereits gewährt wurden, sowie für die Sperrtypen, auf die für dieses Objekt gewartet wird. Für jeden Sperrtyp wird außerdem die Anzahl gewährter und wartender Sperren sowie die Gesamtsummen ausgegeben. Ein Beispiel für die Ausgabe in der Logdatei:

```text
LOG:  LockAcquire: new: lock(0xb7acd844) id(24688,24696,0,0,0,1)
      grantMask(0) req(0,0,0,0,0,0,0)=0 grant(0,0,0,0,0,0,0)=0
      wait(0) type(AccessShareLock)
LOG: GrantLock: lock(0xb7acd844) id(24688,24696,0,0,0,1)
      grantMask(2) req(1,0,0,0,0,0,0)=1 grant(1,0,0,0,0,0,0)=1
      wait(0) type(AccessShareLock)
LOG: UnGrantLock: updated: lock(0xb7acd844)
 id(24688,24696,0,0,0,1)
      grantMask(0) req(0,0,0,0,0,0,0)=0 grant(0,0,0,0,0,0,0)=0
      wait(0) type(AccessShareLock)
LOG: CleanUpLock: deleting: lock(0xb7acd844)
 id(24688,24696,0,0,0,1)
      grantMask(0) req(0,0,0,0,0,0,0)=0 grant(0,0,0,0,0,0,0)=0
      wait(0) type(INVALID)
```

Details der ausgegebenen Struktur finden Sie in `src/include/storage/lock.h`.

Dieser Parameter ist nur verfügbar, wenn das Makro `LOCK_DEBUG` definiert war, als PostgreSQL kompiliert wurde.

`trace_lwlocks` (`boolean`)

Wenn aktiviert, werden Informationen über die Nutzung von Lightweight Locks ausgegeben. Lightweight Locks dienen in erster Linie dazu, wechselseitigen Ausschluss beim Zugriff auf Shared-Memory-Datenstrukturen bereitzustellen.

Dieser Parameter ist nur verfügbar, wenn das Makro `LOCK_DEBUG` definiert war, als PostgreSQL kompiliert wurde.

`trace_userlocks` (`boolean`)

Wenn aktiviert, werden Informationen über die Nutzung von User Locks ausgegeben. Die Ausgabe entspricht der von `trace_locks`, nur für Advisory Locks.

Dieser Parameter ist nur verfügbar, wenn das Makro `LOCK_DEBUG` definiert war, als PostgreSQL kompiliert wurde.

`trace_lock_oidmin` (`integer`)

Wenn gesetzt, werden Sperren für Tabellen unterhalb dieser OID nicht verfolgt (verwendet, um Ausgaben für Systemtabellen zu vermeiden).

Dieser Parameter ist nur verfügbar, wenn das Makro `LOCK_DEBUG` definiert war, als PostgreSQL kompiliert wurde.

`trace_lock_table` (`integer`)

Verfolgt bedingungslos Sperren auf dieser Tabelle (OID).

Dieser Parameter ist nur verfügbar, wenn das Makro `LOCK_DEBUG` definiert war, als PostgreSQL kompiliert wurde.

`debug_deadlocks` (`boolean`)

Wenn gesetzt, werden Informationen über alle aktuellen Sperren ausgegeben, wenn ein Deadlock-Timeout auftritt.

Dieser Parameter ist nur verfügbar, wenn das Makro `LOCK_DEBUG` definiert war, als PostgreSQL kompiliert wurde.

`log_btree_build_stats` (`boolean`)

Wenn gesetzt, werden Statistiken zur Nutzung von Systemressourcen (Speicher und CPU) bei verschiedenen B-Tree-Operationen geloggt.

Dieser Parameter ist nur verfügbar, wenn das Makro `BTREE_BUILD_STATS` definiert war, als PostgreSQL kompiliert wurde.

`wal_consistency_checking` (`string`)

Dieser Parameter ist dafür gedacht, Fehler in den WAL-Redo-Routinen zu prüfen. Wenn er aktiviert ist, werden Full-Page-Images aller Puffer, die im Zusammenhang mit dem WAL-Record geändert wurden, zum Record hinzugefügt. Wenn der Record später erneut abgespielt wird, wendet das System zuerst jeden Record an und prüft dann, ob die durch den Record geänderten Puffer den gespeicherten Images entsprechen. In bestimmten Fällen (etwa bei Hint Bits) sind kleine Abweichungen zulässig und werden ignoriert. Jede unerwartete Abweichung führt zu einem fatalen Fehler, der die Recovery beendet.

Der Standardwert dieser Einstellung ist der leere String, der die Funktion deaktiviert. Sie kann auf `all` gesetzt werden, um alle Records zu prüfen, oder auf eine durch Kommas getrennte Liste von Resource Managern, um nur Records zu prüfen, die von diesen Resource Managern stammen. Derzeit werden die Resource Manager `heap`, `heap2`, `btree`, `hash`, `gin`, `gist`, `sequence`, `spgist`, `brin` und `generic` unterstützt. Extensions können weitere Resource Manager definieren. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`wal_debug` (`boolean`)

Wenn aktiviert, wird WAL-bezogene Debugging-Ausgabe erzeugt. Dieser Parameter ist nur verfügbar, wenn das Makro `WAL_DEBUG` definiert war, als PostgreSQL kompiliert wurde.

`ignore_checksum_failure` (`boolean`)

Hat nur Wirkung, wenn `-k` aktiviert ist.

Das Erkennen eines Prüfsummenfehlers während eines Lesevorgangs führt normalerweise dazu, dass PostgreSQL einen Fehler meldet und die aktuelle Transaktion abbricht. Das Setzen von `ignore_checksum_failure` auf `on` bewirkt, dass das System den Fehler ignoriert (aber weiterhin eine Warnung meldet) und die Verarbeitung fortsetzt. Dieses Verhalten kann Abstürze verursachen, Beschädigungen weitertragen oder verbergen oder andere schwerwiegende Probleme hervorrufen. Es kann jedoch erlauben, den Fehler zu umgehen und unbeschädigte Tupel abzurufen, die möglicherweise noch in der Tabelle vorhanden sind, wenn der Blockheader noch plausibel ist. Wenn der Header beschädigt ist, wird ein Fehler gemeldet, selbst wenn diese Option aktiviert ist. Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`zero_damaged_pages` (`boolean`)

Das Erkennen eines beschädigten Page-Headers führt normalerweise dazu, dass PostgreSQL einen Fehler meldet und die aktuelle Transaktion abbricht. Das Setzen von `zero_damaged_pages` auf `on` bewirkt, dass das System stattdessen eine Warnung meldet, die beschädigte Page im Speicher auf null setzt und die Verarbeitung fortsetzt. Dieses Verhalten zerstört Daten, nämlich alle Zeilen auf der beschädigten Page. Es erlaubt jedoch, den Fehler zu umgehen und Zeilen aus allen unbeschädigten Pages abzurufen, die möglicherweise in der Tabelle vorhanden sind. Es ist nützlich zur Datenwiederherstellung, wenn eine Beschädigung durch einen Hardware- oder Softwarefehler aufgetreten ist. Im Allgemeinen sollten Sie dies erst aktivieren, wenn Sie die Hoffnung aufgegeben haben, Daten aus den beschädigten Pages einer Tabelle wiederherzustellen. Genullte Pages werden nicht erzwungen auf die Platte geschrieben; daher wird empfohlen, die Tabelle oder den Index neu zu erstellen, bevor Sie diesen Parameter wieder ausschalten. Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`ignore_invalid_pages` (`boolean`)

Wenn dieser Parameter auf `off` gesetzt ist (der Standardwert), führt das Erkennen von WAL-Records mit Verweisen auf ungültige Pages während der Recovery dazu, dass PostgreSQL einen Fehler der Stufe `PANIC` auslöst und die Recovery abbricht. Das Setzen von `ignore_invalid_pages` auf `on` bewirkt, dass das System ungültige Page-Verweise in WAL-Records ignoriert (aber weiterhin eine Warnung meldet) und die Recovery fortsetzt. Dieses Verhalten kann Abstürze, Datenverlust, das Weitertragen oder Verbergen von Beschädigung oder andere schwerwiegende Probleme verursachen. Es kann jedoch erlauben, den `PANIC`-Fehler zu umgehen, die Recovery abzuschließen und den Server starten zu lassen. Der Parameter kann nur beim Serverstart gesetzt werden. Er hat nur während der Recovery oder im Standby-Modus Wirkung.

`jit_debugging_support` (`boolean`)

Wenn LLVM die erforderliche Funktionalität besitzt, werden generierte Funktionen bei GDB registriert. Dies erleichtert das Debugging. Der Standardwert ist `off`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`jit_dump_bitcode` (`boolean`)

Schreibt die erzeugte LLVM-IR in das Dateisystem, innerhalb von `data_directory`. Dies ist nur nützlich für Arbeiten am Inneren der JIT-Implementierung. Der Standardwert ist `off`. Nur Superuser und Benutzer mit dem entsprechenden `SET`-Privileg können diese Einstellung ändern.

`jit_expressions` (`boolean`)

Bestimmt, ob Ausdrücke JIT-kompiliert werden, wenn JIT-Kompilierung aktiviert ist (siehe Abschnitt 30.2). Der Standardwert ist `on`.

`jit_profiling_support` (`boolean`)

Wenn LLVM die erforderliche Funktionalität besitzt, werden die Daten ausgegeben, die es `perf` erlauben, von JIT erzeugte Funktionen zu profilieren. Dies schreibt Dateien nach `~/.debug/jit/`; der Benutzer ist dafür verantwortlich, bei Bedarf aufzuräumen. Der Standardwert ist `off`. Dieser Parameter kann nur beim Serverstart gesetzt werden.

`jit_tuple_deforming` (`boolean`)

Bestimmt, ob Tuple Deforming JIT-kompiliert wird, wenn JIT-Kompilierung aktiviert ist (siehe [Abschnitt 30.2](30_Just_in_Time_Kompilierung_JIT.md#302-wann-jit-verwenden)). Der Standardwert ist `on`.

`remove_temp_files_after_crash` (`boolean`)

Wenn dieser Parameter auf `on` gesetzt ist, was dem Standard entspricht, entfernt PostgreSQL nach einem Backend-Absturz automatisch temporäre Dateien. Wenn er deaktiviert ist, bleiben die Dateien erhalten und können zum Beispiel für Debugging verwendet werden. Wiederholte Abstürze können allerdings zur Ansammlung nutzloser Dateien führen. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`send_abort_for_crash` (`boolean`)

Standardmäßig stoppt der Postmaster nach einem Backend-Absturz verbleibende Kindprozesse, indem er ihnen `SIGQUIT`-Signale sendet, was ihnen erlaubt, mehr oder weniger geordnet zu beenden. Wenn diese Option auf `on` gesetzt ist, wird stattdessen `SIGABRT` gesendet. Das führt normalerweise zur Erzeugung einer Core-Dump-Datei für jeden solchen Kindprozess. Dies kann praktisch sein, um die Zustände anderer Prozesse nach einem Absturz zu untersuchen. Es kann bei wiederholten Abstürzen aber auch viel Plattenplatz verbrauchen; aktivieren Sie dies daher nicht auf Systemen, die Sie nicht sorgfältig überwachen. Beachten Sie, dass keine Unterstützung existiert, um die Core-Datei(en) automatisch aufzuräumen. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`send_abort_for_kill` (`boolean`)

Standardmäßig wartet der Postmaster nach dem Versuch, einen Kindprozess mit `SIGQUIT` zu stoppen, fünf Sekunden und sendet dann `SIGKILL`, um die sofortige Beendigung zu erzwingen. Wenn diese Option auf `on` gesetzt ist, wird statt `SIGKILL` `SIGABRT` gesendet. Das führt normalerweise zur Erzeugung einer Core-Dump-Datei für jeden solchen Kindprozess. Dies kann praktisch sein, um die Zustände „hängender“ Kindprozesse zu untersuchen. Es kann bei wiederholten Abstürzen aber auch viel Plattenplatz verbrauchen; aktivieren Sie dies daher nicht auf Systemen, die Sie nicht sorgfältig überwachen. Beachten Sie, dass keine Unterstützung existiert, um die Core-Datei(en) automatisch aufzuräumen. Dieser Parameter kann nur in der Datei `postgresql.conf` oder auf der Server-Kommandozeile gesetzt werden.

`debug_logical_replication_streaming` (`enum`)

Die zulässigen Werte sind `buffered` und `immediate`. Der Standardwert ist `buffered`. Dieser Parameter ist dafür gedacht, logisches Decoding und Replikation großer Transaktionen zu testen. Die Wirkung von `debug_logical_replication_streaming` unterscheidet sich für Publisher und Subscriber:

Auf Publisher-Seite erlaubt `debug_logical_replication_streaming` das sofortige Streamen oder Serialisieren von Änderungen beim logischen Decoding. Bei `immediate` wird jede Änderung gestreamt, wenn die Streaming-Option von `CREATE SUBSCRIPTION` aktiviert ist; andernfalls wird jede Änderung serialisiert. Bei `buffered` streamt oder serialisiert das Decoding Änderungen, wenn `logical_decoding_work_mem` erreicht ist.

Auf Subscriber-Seite kann `debug_logical_replication_streaming`, wenn die Streaming-Option auf `parallel` gesetzt ist, verwendet werden, um den Leader-Apply-Worker anzuweisen, Änderungen an die Shared-Memory-Queue zu senden oder alle Änderungen in die Datei zu serialisieren. Bei `buffered` sendet der Leader Änderungen über eine Shared-Memory-Queue an parallele Apply-Worker. Bei `immediate` serialisiert der Leader alle Änderungen in Dateien und benachrichtigt die parallelen Apply-Worker, sie am Ende der Transaktion zu lesen und anzuwenden.

## 19.18. Kurzoptionen

Aus Bequemlichkeit stehen für einige Parameter auch einbuchstabige Kommandozeilenoptionsschalter zur Verfügung. Sie sind in Tabelle 19.5 beschrieben. Einige dieser Optionen existieren aus historischen Gründen, und ihre Verfügbarkeit als Einbuchstabenoption bedeutet nicht unbedingt eine Empfehlung, die Option intensiv zu verwenden.

Tabelle 19.5. Schlüssel für Kurzoptionen

| Kurzoption | Entsprechung |
| --- | --- |
| `-B x` | `shared_buffers = x` |
| `-d x` | `log_min_messages = DEBUGx` |
| `-e` | `datestyle = euro` |
| `-fb`, `-fh`, `-fi`, `-fm`, `-fn`, `-fo`, `-fs`, `-ft` | `enable_bitmapscan = off`, `enable_hashjoin = off`, `enable_indexscan = off`, `enable_mergejoin = off`, `enable_nestloop = off`, `enable_indexonlyscan = off`, `enable_seqscan = off`, `enable_tidscan = off` |
| `-F` | `fsync = off` |
| `-h x` | `listen_addresses = x` |
| `-i` | `listen_addresses = '*'` |
| `-k x` | `unix_socket_directories = x` |
| `-l` | `ssl = on` |
| `-N x` | `max_connections = x` |
| `-O` | `allow_system_table_mods = on` |
| `-p x` | `port = x` |
| `-P` | `ignore_system_indexes = on` |
| `-s` | `log_statement_stats = on` |
| `-S x` | `work_mem = x` |
| `-tpa`, `-tpl`, `-te` | `log_parser_stats = on`, `log_planner_stats = on`, `log_executor_stats = on` |
| `-W x` | `post_auth_delay = x` |
