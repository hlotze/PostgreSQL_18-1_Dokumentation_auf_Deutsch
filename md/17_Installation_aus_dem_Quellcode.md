# 17. Installation aus dem Quellcode

Dieses Kapitel beschreibt die Installation von PostgreSQL aus der Quellcode-Distribution. Wenn Sie eine vorgefertigte Distribution installieren, etwa ein RPM- oder Debian-Paket, überspringen Sie dieses Kapitel und lesen stattdessen [Kapitel 16](16_Installation_aus_Binärpaketen.md).

## 17.1. Anforderungen

Im Allgemeinen sollte PostgreSQL auf einer modernen Unix-kompatiblen Plattform laufen können. Die Plattformen, die zum Zeitpunkt der Veröffentlichung ausdrücklich getestet wurden, werden weiter unten in [Abschnitt 17.6](#176-unterstützte-plattformen) beschrieben.

Die folgenden Softwarepakete werden benötigt, um PostgreSQL zu bauen:

- GNU make in Version 3.81 oder neuer ist erforderlich; andere make-Programme oder ältere Versionen von GNU make funktionieren nicht. GNU make ist manchmal unter dem Namen `gmake` installiert. Zum Testen geben Sie ein:

```text
make --version
```

- Alternativ kann PostgreSQL mit Meson gebaut werden. Dies ist die einzige Möglichkeit, PostgreSQL unter Windows mit Visual Studio zu bauen. Auf anderen Plattformen ist die Verwendung von Meson derzeit experimentell. Wenn Sie Meson verwenden, benötigen Sie GNU make nicht, aber die übrigen Anforderungen unten gelten weiterhin. Die mindestens erforderliche Meson-Version ist 0.54. Weitere Informationen finden Sie unter <https://mesonbuild.com/>.

- Sie benötigen einen ISO/ANSI-C-Compiler, der mindestens C99-konform ist. Aktuelle Versionen von GCC werden empfohlen, aber PostgreSQL lässt sich mit vielen Compilern verschiedener Hersteller bauen.

- `tar` wird benötigt, um die Quellcode-Distribution zu entpacken, zusätzlich zu entweder `gzip` oder `bzip2`.

- Flex und Bison sind erforderlich. Andere `lex`- und `yacc`-Programme können nicht verwendet werden. Bison muss mindestens Version 2.3 haben.

- Perl 5.14 oder neuer wird während des Build-Prozesses und zum Ausführen einiger Testsuiten benötigt. Diese Anforderung ist unabhängig von den Anforderungen zum Bauen von PL/Perl; siehe unten.

- Die GNU-Readline-Bibliothek wird standardmäßig verwendet. Sie erlaubt `psql`, dem PostgreSQL-Kommandozeilen-SQL-Interpreter, eingegebene Befehle zu merken und frühere Befehle mit den Pfeiltasten wieder aufzurufen und zu bearbeiten. Das ist sehr hilfreich und wird dringend empfohlen. Wenn Sie sie nicht verwenden möchten, müssen Sie bei `configure` die Option `--without-readline` angeben. Als Alternative kann häufig die BSD-lizenzierte Bibliothek `libedit` verwendet werden, die ursprünglich auf NetBSD entwickelt wurde. `libedit` ist mit GNU Readline kompatibel und wird verwendet, wenn `libreadline` nicht gefunden wird oder wenn `--with-libedit-preferred` als Option für `configure` verwendet wird. Wenn Sie eine paketbasierte Linux-Distribution verwenden, beachten Sie, dass Sie sowohl das Paket `readline` als auch `readline-devel` benötigen, falls diese in Ihrer Distribution getrennt sind.

- Die Kompressionsbibliothek `zlib` wird standardmäßig verwendet. Wenn Sie sie nicht verwenden möchten, müssen Sie bei `configure` die Option `--without-zlib` angeben. Diese Option deaktiviert Unterstützung für komprimierte Archive in `pg_dump` und `pg_restore`.

- Die ICU-Bibliothek wird standardmäßig verwendet. Wenn Sie sie nicht verwenden möchten, müssen Sie bei `configure` die Option `--without-icu` angeben. Diese Option deaktiviert Unterstützung für ICU-Collation-Funktionen; siehe [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung).

  ICU-Unterstützung erfordert, dass das Paket ICU4C installiert ist. Die derzeit mindestens erforderliche ICU4C-Version ist 4.2.

  Standardmäßig wird `pkg-config` verwendet, um die benötigten Compileroptionen zu finden. Dies wird für ICU4C ab Version 4.6 unterstützt. Für ältere Versionen oder wenn `pkg-config` nicht verfügbar ist, können die Variablen `ICU_CFLAGS` und `ICU_LIBS` für `configure` angegeben werden, zum Beispiel:

```text
./configure ... ICU_CFLAGS='-I/some/where/include' ICU_LIBS='-L/some/where/lib -licui18n -licuuc -licudata'
```

  Wenn ICU4C im Standardsuchpfad des Compilers liegt, müssen Sie dennoch nichtleere Zeichenketten angeben, um die Verwendung von `pkg-config` zu vermeiden, zum Beispiel `ICU_CFLAGS=' '`.

Die folgenden Pakete sind optional. Sie werden in der Standardkonfiguration nicht benötigt, sind aber erforderlich, wenn bestimmte Build-Optionen aktiviert werden:

- Zum Bauen der serverseitigen Programmiersprache PL/Perl benötigen Sie eine vollständige Perl-Installation einschließlich der Bibliothek `libperl` und der Header-Dateien. Die mindestens erforderliche Version ist Perl 5.14. Da PL/Perl eine Shared Library wird, muss auch `libperl` auf den meisten Plattformen eine Shared Library sein. In aktuellen Perl-Versionen scheint dies der Standard zu sein, in älteren Versionen war es das nicht; in jedem Fall hängt es davon ab, wie Perl auf Ihrer Site installiert wurde. `configure` schlägt fehl, wenn PL/Perl ausgewählt ist, aber keine gemeinsam genutzte `libperl` gefunden wird. In diesem Fall müssen Sie Perl manuell neu bauen und installieren, damit PL/Perl gebaut werden kann. Fordern Sie während der Konfiguration von Perl eine Shared Library an.

  Wenn Sie PL/Perl mehr als nur gelegentlich verwenden möchten, sollten Sie sicherstellen, dass die Perl-Installation mit aktivierter Option `usemultiplicity` gebaut wurde. `perl -V` zeigt, ob dies der Fall ist.

- Zum Bauen der serverseitigen Programmiersprache PL/Python benötigen Sie eine Python-Installation mit Header-Dateien und dem Modul `sysconfig`. Die mindestens unterstützte Version ist Python 3.6.8.

  Da PL/Python eine Shared Library wird, muss auch `libpython` auf den meisten Plattformen eine Shared Library sein. Das ist bei einer aus dem Quellcode gebauten Standard-Python-Installation nicht der Fall, aber in vielen Betriebssystemdistributionen ist eine Shared Library verfügbar. `configure` schlägt fehl, wenn PL/Python ausgewählt ist, aber keine gemeinsam genutzte `libpython` gefunden wird. Das kann bedeuten, dass Sie zusätzliche Pakete installieren oder Ihre Python-Installation ganz oder teilweise neu bauen müssen, um diese Shared Library bereitzustellen. Wenn Sie Python aus dem Quellcode bauen, führen Sie Pythons `configure` mit der Option `--enable-shared` aus.

- Zum Bauen der prozeduralen Sprache PL/Tcl benötigen Sie eine Tcl-Installation. Die mindestens erforderliche Version ist Tcl 8.4.

- Um Native Language Support (NLS) zu aktivieren, also die Fähigkeit, Programmmeldungen in einer anderen Sprache als Englisch anzuzeigen, benötigen Sie eine Implementierung der Gettext-API. Einige Betriebssysteme haben diese eingebaut, zum Beispiel Linux, NetBSD und Solaris. Für andere Systeme können Sie ein Zusatzpaket von <https://www.gnu.org/software/gettext/> herunterladen. Wenn Sie die Gettext-Implementierung der GNU-C-Bibliothek verwenden, benötigen Sie zusätzlich das GNU-Gettext-Paket für einige Hilfsprogramme. Für andere Implementierungen benötigen Sie es nicht.

- Sie benötigen OpenSSL, wenn Sie verschlüsselte Client-Verbindungen unterstützen möchten. OpenSSL ist außerdem auf Plattformen erforderlich, die kein `/dev/urandom` haben, mit Ausnahme von Windows. Die mindestens erforderliche Version ist 1.1.1.

  Zusätzlich wird LibreSSL über die OpenSSL-Kompatibilitätsschicht unterstützt. Die mindestens erforderliche Version ist 3.4, aus OpenBSD Version 7.0.

- Sie benötigen MIT Kerberos für GSSAPI, OpenLDAP und/oder PAM, wenn Sie Authentifizierung über diese Dienste unterstützen möchten.

- Sie benötigen Curl, um ein optionales Modul zu bauen, das den OAuth Device Authorization Flow für Client-Anwendungen implementiert.

- Sie benötigen LZ4, wenn Sie Datenkompression mit dieser Methode unterstützen möchten; siehe `default_toast_compression` und `wal_compression`.

- Sie benötigen Zstandard, wenn Sie Datenkompression mit dieser Methode unterstützen möchten; siehe `wal_compression`. Die mindestens erforderliche Version ist 1.4.0.

- Zum Bauen der PostgreSQL-Dokumentation gibt es eigene Anforderungen; siehe Abschnitt J.2.

Wenn Sie ein GNU-Paket benötigen, finden Sie es auf Ihrem lokalen GNU-Mirror, siehe <https://www.gnu.org/prep/ftp>, oder unter <ftp://ftp.gnu.org/gnu/>.

## 17.2. Den Quellcode beziehen

Der PostgreSQL-Quellcode veröffentlichter Versionen kann aus dem Downloadbereich der Website bezogen werden: <https://www.postgresql.org/ftp/source/>. Laden Sie die gewünschte Datei `postgresql-version.tar.gz` oder `postgresql-version.tar.bz2` herunter und entpacken Sie sie:

```text
tar xf postgresql-version.tar.bz2
```

Dadurch entsteht unterhalb des aktuellen Verzeichnisses ein Verzeichnis `postgresql-version` mit den PostgreSQL-Quellen. Wechseln Sie für den Rest der Installationsprozedur in dieses Verzeichnis.

Alternativ können Sie das Versionskontrollsystem Git verwenden; weitere Informationen finden Sie in Abschnitt I.1.

## 17.3. Bauen und Installieren mit Autoconf und Make

### 17.3.1. Kurzfassung

```text
./configure
make
su
make install
adduser postgres
mkdir -p /usr/local/pgsql/data
chown postgres /usr/local/pgsql/data
su - postgres
/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start
/usr/local/pgsql/bin/createdb test
/usr/local/pgsql/bin/psql test
```

Die ausführliche Fassung ist der Rest dieses Abschnitts.

### 17.3.2. Installationsprozedur

1. **Konfiguration**

   Der erste Schritt der Installation besteht darin, den Quellbaum für Ihr System zu konfigurieren und die gewünschten Optionen auszuwählen. Dies geschieht durch Ausführen des Skripts `configure`. Für eine Standardinstallation geben Sie einfach ein:

```text
./configure
```

   Dieses Skript führt eine Reihe von Tests aus, um Werte für verschiedene systemabhängige Variablen zu bestimmen und Besonderheiten Ihres Betriebssystems zu erkennen. Anschließend erzeugt es im Build-Baum mehrere Dateien, in denen die Ergebnisse festgehalten werden.

   Sie können `configure` auch in einem Verzeichnis außerhalb des Quellbaums ausführen und dort bauen, wenn Sie das Build-Verzeichnis von den ursprünglichen Quelldateien getrennt halten möchten. Dieses Verfahren wird als VPATH-Build bezeichnet:

```text
mkdir build_dir
cd build_dir
/path/to/source/tree/configure [options go here]
make
```

   Die Standardkonfiguration baut den Server und die Hilfsprogramme sowie alle Client-Anwendungen und Schnittstellen, die nur einen C-Compiler benötigen. Standardmäßig werden alle Dateien unter `/usr/local/pgsql` installiert.

   Sie können den Build- und Installationsprozess anpassen, indem Sie `configure` eine oder mehrere Kommandozeilenoptionen übergeben. Typischerweise passen Sie den Installationsort oder die Menge der zu bauenden optionalen Funktionen an. `configure` besitzt viele Optionen, die in [Abschnitt 17.3.3](#1733-configureoptionen) beschrieben werden.

   Außerdem reagiert `configure` auf bestimmte Umgebungsvariablen, wie in [Abschnitt 17.3.4](#1734-configureumgebungsvariablen) beschrieben. Sie bieten weitere Möglichkeiten, die Konfiguration anzupassen.

2. **Bauen**

   Um den Build zu starten, geben Sie eines der folgenden Kommandos ein:

```text
make
make all
```

   Denken Sie daran, GNU make zu verwenden. Der Build dauert je nach Hardware einige Minuten.

   Wenn Sie alles bauen möchten, was gebaut werden kann, einschließlich Dokumentation in HTML- und Manpage-Form sowie zusätzlicher Module aus `contrib`, geben Sie stattdessen ein:

```text
make world
```

   Wenn Sie alles bauen möchten, was gebaut werden kann, einschließlich der zusätzlichen Module aus `contrib`, aber ohne Dokumentation, geben Sie stattdessen ein:

```text
make world-bin
```

   Wenn Sie den Build nicht manuell, sondern aus einem anderen Makefile heraus aufrufen möchten, müssen Sie `MAKELEVEL` zurücksetzen oder auf null setzen, zum Beispiel so:

```text
build-postgresql:
        $(MAKE) -C postgresql MAKELEVEL=0 all
```

   Andernfalls kann es zu merkwürdigen Fehlermeldungen kommen, typischerweise über fehlende Header-Dateien.

3. **Regressionstests**

   Wenn Sie den neu gebauten Server vor der Installation testen möchten, können Sie an dieser Stelle die Regressionstests ausführen. Die Regressionstests sind eine Testsuite, die prüft, ob PostgreSQL auf Ihrer Maschine so läuft, wie es die Entwickler erwarten. Geben Sie ein:

```text
make check
```

   Dies funktioniert nicht als `root`; führen Sie es als unprivilegierter Benutzer aus. Ausführliche Informationen zur Interpretation der Testergebnisse finden Sie in [Kapitel 31](31_Regressionstests.md). Sie können diesen Test jederzeit später mit demselben Befehl wiederholen.

4. **Dateien installieren**

> **Hinweis:** Wenn Sie ein bestehendes System aktualisieren, lesen Sie unbedingt [Abschnitt 18.6](18_Servereinrichtung_und_Betrieb.md#186-einen-postgresqlcluster-aktualisieren), der Anweisungen zum Aktualisieren eines Clusters enthält.

   Um PostgreSQL zu installieren, geben Sie ein:

```text
make install
```

   Dadurch werden Dateien in die Verzeichnisse installiert, die in Schritt 1 angegeben wurden. Stellen Sie sicher, dass Sie Schreibrechte für diesen Bereich haben. Normalerweise müssen Sie diesen Schritt als `root` ausführen. Alternativ können Sie die Zielverzeichnisse vorher erstellen und passende Berechtigungen vergeben.

   Um die Dokumentation in HTML- und Manpage-Form zu installieren, geben Sie ein:

```text
make install-docs
```

   Wenn Sie oben `world` gebaut haben, geben Sie stattdessen ein:

```text
make install-world
```

   Das installiert auch die Dokumentation.

   Wenn Sie oben `world` ohne Dokumentation gebaut haben, geben Sie stattdessen ein:

```text
make install-world-bin
```

   Sie können `make install-strip` statt `make install` verwenden, um ausführbare Dateien und Bibliotheken während der Installation zu strippen. Das spart Platz. Wenn Sie mit Debug-Unterstützung gebaut haben, entfernt Strippen diese Unterstützung praktisch; es sollte daher nur erfolgen, wenn Debugging nicht mehr benötigt wird. `install-strip` versucht, sinnvoll Platz zu sparen, weiß aber nicht perfekt, wie jedes unnötige Byte aus einer ausführbaren Datei entfernt werden kann. Wenn Sie möglichst viel Plattenplatz sparen möchten, müssen Sie manuell nacharbeiten.

   Die Standardinstallation stellt alle Header-Dateien bereit, die für die Entwicklung von Client-Anwendungen und für serverseitige Programmentwicklung benötigt werden, etwa eigene Funktionen oder Datentypen in C.

   **Nur-Client-Installation:** Wenn Sie nur die Client-Anwendungen und Schnittstellenbibliotheken installieren möchten, können Sie diese Befehle verwenden:

```text
make -C src/bin install
make -C src/include install
make -C src/interfaces install
make -C doc install
```

   `src/bin` enthält einige Binärdateien, die nur für den Servergebrauch gedacht sind, aber sie sind klein.

   **Deinstallation:** Um die Installation rückgängig zu machen, verwenden Sie `make uninstall`. Dadurch werden jedoch keine erzeugten Verzeichnisse entfernt.

   **Bereinigung:** Nach der Installation können Sie Plattenplatz freigeben, indem Sie die gebauten Dateien mit `make clean` aus dem Quellbaum entfernen. Die von `configure` erzeugten Dateien bleiben dabei erhalten, sodass Sie später erneut mit `make` bauen können. Um den Quellbaum in den Zustand der Auslieferung zurückzusetzen, verwenden Sie `make distclean`. Wenn Sie innerhalb desselben Quellbaums für mehrere Plattformen bauen möchten, müssen Sie dies tun und für jede Plattform neu konfigurieren. Alternativ verwenden Sie für jede Plattform einen eigenen Build-Baum, sodass der Quellbaum unverändert bleibt.

   Wenn Sie einen Build durchführen und danach feststellen, dass Ihre `configure`-Optionen falsch waren, oder wenn sich etwas ändert, das `configure` untersucht, zum Beispiel durch Software-Upgrades, ist es sinnvoll, vor dem erneuten Konfigurieren und Bauen `make distclean` auszuführen. Andernfalls werden Änderungen an der Konfiguration möglicherweise nicht überall wirksam, wo sie benötigt werden.

### 17.3.3. configure-Optionen

Die Kommandozeilenoptionen von `configure` werden unten erläutert. Diese Liste ist nicht vollständig; mit `./configure --help` erhalten Sie eine vollständige Liste. Die hier nicht behandelten Optionen sind für fortgeschrittene Anwendungsfälle wie Cross-Compilation gedacht und in der Standarddokumentation von Autoconf beschrieben.

#### 17.3.3.1. Installationsorte

Diese Optionen steuern, wo `make install` die Dateien ablegt. Die Option `--prefix` genügt in den meisten Fällen. Wenn Sie besondere Anforderungen haben, können Sie die Installationsunterverzeichnisse mit den weiteren Optionen dieses Abschnitts anpassen. Beachten Sie jedoch, dass Änderungen an den relativen Positionen der verschiedenen Unterverzeichnisse die Installation nicht verschiebbar machen können; Sie können sie nach der Installation dann nicht mehr einfach verschieben. Die Orte für Manpages und Dokumentation sind von dieser Einschränkung nicht betroffen. Für verschiebbare Installationen möchten Sie möglicherweise die später beschriebene Option `--disable-rpath` verwenden.

- `--prefix=PREFIX`

  Installiert alle Dateien unterhalb des Verzeichnisses `PREFIX` statt unter `/usr/local/pgsql`. Die tatsächlichen Dateien werden in verschiedene Unterverzeichnisse installiert; direkt in das Verzeichnis `PREFIX` werden nie Dateien installiert.

- `--exec-prefix=EXEC-PREFIX`

  Architekturabhängige Dateien können unter einem anderen Präfix `EXEC-PREFIX` installiert werden als dem mit `PREFIX` gesetzten. Das kann nützlich sein, um architekturunabhängige Dateien zwischen Hosts zu teilen. Wenn Sie dies weglassen, wird `EXEC-PREFIX` gleich `PREFIX` gesetzt, und architekturabhängige wie architekturunabhängige Dateien werden unter demselben Baum installiert.

- `--bindir=DIRECTORY`

  Gibt das Verzeichnis für ausführbare Programme an. Standard ist `EXEC-PREFIX/bin`, also normalerweise `/usr/local/pgsql/bin`.

- `--sysconfdir=DIRECTORY`

  Setzt das Verzeichnis für verschiedene Konfigurationsdateien, standardmäßig `PREFIX/etc`.

- `--libdir=DIRECTORY`

  Setzt den Installationsort für Bibliotheken und dynamisch ladbare Module. Standard ist `EXEC-PREFIX/lib`.

- `--includedir=DIRECTORY`

  Setzt das Verzeichnis für C- und C++-Header-Dateien. Standard ist `PREFIX/include`.

- `--datarootdir=DIRECTORY`

  Setzt das Wurzelverzeichnis für verschiedene Arten schreibgeschützter Datendateien. Dies setzt nur den Standard für einige der folgenden Optionen. Standard ist `PREFIX/share`.

- `--datadir=DIRECTORY`

  Setzt das Verzeichnis für schreibgeschützte Datendateien, die von den installierten Programmen verwendet werden. Standard ist `DATAROOTDIR`. Dies hat nichts damit zu tun, wo Ihre Datenbankdateien abgelegt werden.

- `--localedir=DIRECTORY`

  Setzt das Verzeichnis für Locale-Daten, insbesondere Katalogdateien für Meldungsübersetzungen. Standard ist `DATAROOTDIR/locale`.

- `--mandir=DIRECTORY`

  Die mit PostgreSQL gelieferten Manpages werden unterhalb dieses Verzeichnisses in ihren jeweiligen `manx`-Unterverzeichnissen installiert. Standard ist `DATAROOTDIR/man`.

- `--docdir=DIRECTORY`

  Setzt das Wurzelverzeichnis für Dokumentationsdateien außer Manpages. Dies setzt nur den Standard für die folgenden Optionen. Der Standardwert ist `DATAROOTDIR/doc/postgresql`.

- `--htmldir=DIRECTORY`

  Die HTML-Dokumentation von PostgreSQL wird unterhalb dieses Verzeichnisses installiert. Standard ist `DATAROOTDIR`.

> **Hinweis:** PostgreSQL wurde so vorbereitet, dass es in gemeinsam genutzte Installationsorte wie `/usr/local/include` installiert werden kann, ohne den Namensraum des übrigen Systems zu stören. Zunächst wird die Zeichenkette `/postgresql` automatisch an `datadir`, `sysconfdir` und `docdir` angehängt, sofern der vollständig expandierte Verzeichnisname nicht bereits `postgres` oder `pgsql` enthält. Wenn Sie zum Beispiel `/usr/local` als Präfix wählen, wird die Dokumentation unter `/usr/local/doc/postgresql` installiert; bei `/opt/postgres` als Präfix landet sie unter `/opt/postgres/doc`. Die öffentlichen C-Header-Dateien der Client-Schnittstellen werden in `includedir` installiert und sind namensraumsauber. Interne Header-Dateien und Server-Header-Dateien werden in privaten Verzeichnissen unter `includedir` installiert. Informationen zum Zugriff auf Header-Dateien finden Sie in der Dokumentation der jeweiligen Schnittstelle. Schließlich wird, falls passend, auch unter `libdir` ein privates Unterverzeichnis für dynamisch ladbare Module erstellt.

#### 17.3.3.2. PostgreSQL-Funktionen

Die in diesem Abschnitt beschriebenen Optionen aktivieren den Bau verschiedener PostgreSQL-Funktionen, die standardmäßig nicht gebaut werden. Die meisten sind nur deshalb nicht standardmäßig aktiv, weil sie zusätzliche Software erfordern, wie in [Abschnitt 17.1](#171-anforderungen) beschrieben.

- `--enable-nls[=LANGUAGES]`

  Aktiviert Native Language Support (NLS), also die Fähigkeit, Programmmeldungen in einer anderen Sprache als Englisch anzuzeigen. `LANGUAGES` ist eine optionale, durch Leerzeichen getrennte Liste der Sprachcodes, die unterstützt werden sollen, zum Beispiel `--enable-nls='de fr'`. Die Schnittmenge Ihrer Liste mit den tatsächlich bereitgestellten Übersetzungen wird automatisch berechnet. Wenn Sie keine Liste angeben, werden alle verfügbaren Übersetzungen installiert.

  Für diese Option benötigen Sie eine Implementierung der Gettext-API.

- `--with-perl`

  Baut die serverseitige Sprache PL/Perl.

- `--with-python`

  Baut die serverseitige Sprache PL/Python.

- `--with-tcl`

  Baut die serverseitige Sprache PL/Tcl.

- `--with-tclconfig=DIRECTORY`

  Tcl installiert die Datei `tclConfig.sh`, die Konfigurationsinformationen zum Bauen von Modulen enthält, die mit Tcl zusammenarbeiten. Diese Datei wird normalerweise automatisch an einer bekannten Stelle gefunden. Wenn Sie eine andere Tcl-Version verwenden möchten, können Sie das Verzeichnis angeben, in dem nach `tclConfig.sh` gesucht werden soll.

- `--with-llvm`

  Baut mit Unterstützung für LLVM-basierte JIT-Kompilierung; siehe [Kapitel 30](30_Just_in_Time_Kompilierung_JIT.md). Dafür muss die LLVM-Bibliothek installiert sein. Die derzeit mindestens erforderliche LLVM-Version ist 14.

  `llvm-config` wird verwendet, um die benötigten Compileroptionen zu finden. Es wird in Ihrem `PATH` gesucht. Wenn das nicht das gewünschte Programm ergibt, verwenden Sie `LLVM_CONFIG`, um den Pfad zum richtigen `llvm-config` anzugeben:

```text
./configure ... --with-llvm LLVM_CONFIG='/path/to/llvm/bin/llvm-config'
```

  LLVM-Unterstützung erfordert einen kompatiblen `clang`-Compiler, falls nötig mit der Umgebungsvariable `CLANG` angegeben, und einen funktionierenden C++-Compiler, falls nötig mit der Umgebungsvariable `CXX` angegeben.

- `--with-lz4`

  Baut mit Unterstützung für LZ4-Kompression.

- `--with-zstd`

  Baut mit Unterstützung für Zstandard-Kompression.

- `--with-ssl=LIBRARY`

  Baut mit Unterstützung für SSL-Verbindungen, also verschlüsselte Verbindungen. Die einzige unterstützte `LIBRARY` ist `openssl`, die sowohl für OpenSSL als auch LibreSSL verwendet wird. Dafür muss das OpenSSL-Paket installiert sein. `configure` prüft vor dem Fortfahren, ob die erforderlichen Header-Dateien und Bibliotheken vorhanden sind und Ihre OpenSSL-Installation ausreicht.

- `--with-openssl`

  Veraltetes Äquivalent zu `--with-ssl=openssl`.

- `--with-gssapi`

  Baut mit Unterstützung für GSSAPI-Authentifizierung. Dafür muss MIT Kerberos installiert sein. Auf vielen Systemen liegt das GSSAPI-System, ein Teil der MIT-Kerberos-Installation, nicht an einem standardmäßig durchsuchten Ort wie `/usr/include` oder `/usr/lib`; verwenden Sie in diesem Fall zusätzlich die Optionen `--with-includes` und `--with-libraries`. `configure` prüft, ob die erforderlichen Header-Dateien und Bibliotheken vorhanden sind.

- `--with-ldap`

  Baut mit LDAP-Unterstützung für Authentifizierung und Nachschlagen von Verbindungsparametern; weitere Informationen finden Sie in [Abschnitt 32.18](32_libpq_C_Bibliothek.md#3218-ldapsuche-nach-verbindungsparametern) und [Abschnitt 20.10](20_Client_Authentifizierung.md#2010-ldapauthentifizierung). Unter Unix muss dafür OpenLDAP installiert sein. Unter Windows wird die standardmäßige WinLDAP-Bibliothek verwendet.

- `--with-pam`

  Baut mit PAM-Unterstützung, also Pluggable Authentication Modules.

- `--with-bsd-auth`

  Baut mit Unterstützung für BSD Authentication. Das BSD-Authentication-Framework ist derzeit nur auf OpenBSD verfügbar.

- `--with-systemd`

  Baut mit Unterstützung für systemd-Service-Benachrichtigungen. Das verbessert die Integration, wenn der Server unter systemd gestartet wird, hat ansonsten aber keine Auswirkungen; siehe [Abschnitt 18.3](18_Servereinrichtung_und_Betrieb.md#183-den-datenbankserver-starten). `libsystemd` und die zugehörigen Header-Dateien müssen installiert sein.

- `--with-bonjour`

  Baut mit Unterstützung für automatische Dienstsuche über Bonjour. Dafür muss Ihr Betriebssystem Bonjour unterstützen. Auf macOS wird dies empfohlen.

- `--with-uuid=LIBRARY`

  Baut das Modul `uuid-ossp`, das Funktionen zum Erzeugen von UUIDs bereitstellt, mit der angegebenen UUID-Bibliothek. `LIBRARY` muss eine der folgenden sein:

  - `bsd`, um die UUID-Funktionen aus FreeBSD und einigen anderen BSD-abgeleiteten Systemen zu verwenden
  - `e2fs`, um die vom Projekt e2fsprogs erzeugte UUID-Bibliothek zu verwenden; sie ist auf den meisten Linux-Systemen und in macOS vorhanden und kann auch für andere Plattformen bezogen werden
  - `ossp`, um die OSSP-UUID-Bibliothek zu verwenden: <http://www.ossp.org/pkg/lib/uuid/>

- `--with-ossp-uuid`

  Veraltetes Äquivalent zu `--with-uuid=ossp`.

- `--with-libcurl`

  Baut mit libcurl-Unterstützung für OAuth-2.0-Client-Flows. Für diese Funktion ist libcurl Version 7.61.0 oder neuer erforderlich.

- `--with-libnuma`

  Baut mit `libnuma`-Unterstützung für grundlegende NUMA-Unterstützung. Dies wird nur auf Plattformen unterstützt, für die die Bibliothek `libnuma` implementiert ist.

- `--with-liburing`

  Baut mit `liburing` und aktiviert damit `io_uring`-Unterstützung für asynchrones I/O. PostgreSQL fragt `pkg-config` ab, um die benötigten Compiler- und Linkeroptionen zu ermitteln. Für eine `liburing`-Installation an einem ungewöhnlichen Ort können Sie `pkg-config`-bezogene Umgebungsvariablen setzen.

- `--with-libxml`

  Baut mit `libxml2` und aktiviert SQL/XML-Unterstützung. Dafür ist libxml2 Version 2.6.23 oder neuer erforderlich. PostgreSQL fragt `pkg-config` ab, falls es installiert ist und `libxml2` kennt; andernfalls wird, falls gefunden, das von `libxml2` installierte Programm `xml2-config` verwendet. Die Verwendung von `pkg-config` wird bevorzugt, weil sie besser mit Multi-Architektur-Installationen umgehen kann.

  Für eine `libxml2`-Installation an einem ungewöhnlichen Ort können Sie `pkg-config`-bezogene Umgebungsvariablen setzen, `XML2_CONFIG` auf das passende `xml2-config` zeigen lassen oder `XML2_CFLAGS` und `XML2_LIBS` setzen. Wenn `pkg-config` installiert ist und dessen Annahme über den Ort von `libxml2` überschrieben werden soll, müssen Sie entweder `XML2_CONFIG` setzen oder sowohl `XML2_CFLAGS` als auch `XML2_LIBS` auf nichtleere Zeichenketten setzen.

- `--with-libxslt`

  Baut mit `libxslt` und ermöglicht dem Modul `xml2`, XSL-Transformationen von XML auszuführen. `--with-libxml` muss ebenfalls angegeben werden.

- `--with-selinux`

  Baut mit SELinux-Unterstützung und aktiviert die Erweiterung `sepgsql`.

#### 17.3.3.3. Anti-Features

Die in diesem Abschnitt beschriebenen Optionen erlauben das Deaktivieren bestimmter PostgreSQL-Funktionen, die standardmäßig gebaut werden, aber abgeschaltet werden müssen, wenn die erforderliche Software oder Systemfunktion nicht verfügbar ist. Die Verwendung dieser Optionen wird nur empfohlen, wenn sie wirklich notwendig ist.

- `--without-icu`

  Baut ohne Unterstützung für die ICU-Bibliothek und deaktiviert damit ICU-Collation-Funktionen; siehe [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung).

- `--without-readline`

  Verhindert die Verwendung der Readline-Bibliothek und auch von `libedit`. Diese Option deaktiviert Kommandozeilenbearbeitung und Verlauf in `psql`.

- `--with-libedit-preferred`

  Bevorzugt die BSD-lizenzierte Bibliothek `libedit` gegenüber der GPL-lizenzierten Readline-Bibliothek. Diese Option ist nur relevant, wenn beide Bibliotheken installiert sind; standardmäßig wird dann Readline verwendet.

- `--without-zlib`

  Verhindert die Verwendung der Zlib-Bibliothek. Dies deaktiviert Unterstützung für komprimierte Archive in `pg_dump` und `pg_restore`.

#### 17.3.3.4. Details zum Build-Prozess

- `--with-includes=DIRECTORIES`

  `DIRECTORIES` ist eine durch Doppelpunkte getrennte Liste von Verzeichnissen, die zur Suchliste des Compilers für Header-Dateien hinzugefügt werden. Wenn optionale Pakete wie GNU Readline an einem nicht standardmäßigen Ort installiert sind, müssen Sie diese Option und wahrscheinlich auch `--with-libraries` verwenden.

  Beispiel: `--with-includes=/opt/gnu/include:/usr/sup/include`.

- `--with-libraries=DIRECTORIES`

  `DIRECTORIES` ist eine durch Doppelpunkte getrennte Liste von Verzeichnissen, in denen nach Bibliotheken gesucht wird. Sie werden diese Option und die zugehörige Option `--with-includes` wahrscheinlich benötigen, wenn Pakete an nicht standardmäßigen Orten installiert sind.

  Beispiel: `--with-libraries=/opt/gnu/lib:/usr/sup/lib`.

- `--with-system-tzdata=DIRECTORY`

  PostgreSQL enthält eine eigene Zeitzonendatenbank, die für Datums- und Zeitoperationen benötigt wird. Diese Zeitzonendatenbank ist mit der IANA-Zeitzonendatenbank kompatibel, die viele Betriebssysteme wie FreeBSD, Linux und Solaris bereitstellen. Es wäre daher redundant, sie erneut zu installieren. Wenn diese Option verwendet wird, wird die vom System bereitgestellte Zeitzonendatenbank in `DIRECTORY` statt der in der PostgreSQL-Quellcode-Distribution enthaltenen verwendet. `DIRECTORY` muss als absoluter Pfad angegeben werden. Auf einigen Betriebssystemen ist `/usr/share/zoneinfo` ein wahrscheinlicher Ort.

  Die Installationsroutine erkennt keine nicht passenden oder fehlerhaften Zeitzonendaten. Wenn Sie diese Option verwenden, sollten Sie die Regressionstests ausführen, um zu prüfen, ob die angegebenen Zeitzonendaten korrekt mit PostgreSQL funktionieren.

  Diese Option richtet sich vor allem an Distributoren von Binärpaketen, die ihr Zielbetriebssystem gut kennen. Der Hauptvorteil ist, dass das PostgreSQL-Paket nicht aktualisiert werden muss, wenn sich eine der vielen lokalen Sommerzeitregeln ändert. Ein weiterer Vorteil ist, dass PostgreSQL einfacher cross-kompiliert werden kann, wenn die Zeitzonendatenbankdateien während der Installation nicht gebaut werden müssen.

- `--with-extra-version=STRING`

  Hängt `STRING` an die PostgreSQL-Versionsnummer an. Damit können Sie zum Beispiel Binärdateien aus unveröffentlichten Git-Snapshots oder mit eigenen Patches mit einer zusätzlichen Versionszeichenkette markieren, etwa mit einem `git describe`-Bezeichner oder einer Paket-Release-Nummer einer Distribution.

- `--disable-rpath`

  Markiert die ausführbaren Dateien von PostgreSQL nicht so, dass sie im Bibliotheksverzeichnis der Installation nach Shared Libraries suchen; siehe `--libdir`. Auf den meisten Plattformen verwendet diese Markierung einen absoluten Pfad zum Bibliotheksverzeichnis, was nicht hilfreich ist, wenn die Installation später verschoben wird. Sie müssen dann jedoch auf andere Weise dafür sorgen, dass die ausführbaren Dateien die Shared Libraries finden. Typischerweise muss dafür der dynamische Linker des Betriebssystems so konfiguriert werden, dass er das Bibliotheksverzeichnis durchsucht; siehe [Abschnitt 17.5.1](#1751-shared-libraries).

#### 17.3.3.5. Verschiedenes

Gerade bei Test-Builds ist es recht üblich, die Standardportnummer mit `--with-pgport` anzupassen. Die anderen Optionen in diesem Abschnitt werden nur für fortgeschrittene Benutzer empfohlen.

- `--with-pgport=NUMBER`

  Setzt `NUMBER` als Standardportnummer für Server und Clients. Standard ist 5432. Der Port kann später immer geändert werden. Wenn Sie ihn hier angeben, ist jedoch für Server und Clients derselbe Standard einkompiliert, was sehr praktisch sein kann. Normalerweise ist der einzige gute Grund für einen vom Standard abweichenden Wert, dass Sie mehrere PostgreSQL-Server auf derselben Maschine betreiben möchten.

- `--with-krb-srvnam=NAME`

  Der Standardname des von GSSAPI verwendeten Kerberos-Service-Principals. Standard ist `postgres`. Meist gibt es keinen Grund, dies zu ändern, außer Sie bauen für eine Windows-Umgebung; dort muss der Name in Großbuchstaben `POSTGRES` lauten.

- `--with-segsize=SEGSIZE`

  Setzt die Segmentgröße in Gigabyte. Große Tabellen werden in mehrere Betriebssystemdateien aufgeteilt, deren Größe jeweils der Segmentgröße entspricht. Das vermeidet Probleme mit Dateigrößenbeschränkungen auf vielen Plattformen. Die Standardsegmentgröße von 1 Gigabyte ist auf allen unterstützten Plattformen sicher. Wenn Ihr Betriebssystem „largefile“-Unterstützung hat, was heute meistens der Fall ist, können Sie eine größere Segmentgröße verwenden. Das kann helfen, bei sehr großen Tabellen die Zahl der verbrauchten Dateideskriptoren zu verringern. Wählen Sie aber keinen Wert, der größer ist als von Ihrer Plattform und den vorgesehenen Dateisystemen unterstützt. Andere Werkzeuge wie `tar` können ebenfalls Grenzen für die nutzbare Dateigröße setzen. Es wird empfohlen, wenn auch nicht zwingend verlangt, einen Wert zu wählen, der eine Potenz von 2 ist. Eine Änderung dieses Werts bricht die Datenbankkompatibilität auf Plattenebene; Sie können `pg_upgrade` dann nicht verwenden, um auf einen Build mit anderer Segmentgröße zu aktualisieren.

- `--with-blocksize=BLOCKSIZE`

  Setzt die Blockgröße in Kilobyte. Dies ist die Einheit für Speicherung und I/O innerhalb von Tabellen. Der Standardwert von 8 Kilobyte passt für die meisten Situationen, aber andere Werte können in Sonderfällen nützlich sein. Der Wert muss eine Potenz von 2 zwischen 1 und 32 Kilobyte sein. Eine Änderung dieses Werts bricht die Datenbankkompatibilität auf Plattenebene; Sie können `pg_upgrade` dann nicht verwenden, um auf einen Build mit anderer Blockgröße zu aktualisieren.

- `--with-wal-blocksize=BLOCKSIZE`

  Setzt die WAL-Blockgröße in Kilobyte. Dies ist die Einheit für Speicherung und I/O im WAL-Log. Der Standardwert von 8 Kilobyte passt für die meisten Situationen, aber andere Werte können in Sonderfällen nützlich sein. Der Wert muss eine Potenz von 2 zwischen 1 und 64 Kilobyte sein. Eine Änderung dieses Werts bricht die Datenbankkompatibilität auf Plattenebene; Sie können `pg_upgrade` dann nicht verwenden, um auf einen Build mit anderer WAL-Blockgröße zu aktualisieren.

#### 17.3.3.6. Entwickleroptionen

Die meisten Optionen in diesem Abschnitt sind nur für die Entwicklung oder das Debugging von PostgreSQL interessant. Sie werden für Produktions-Builds nicht empfohlen, außer `--enable-debug`, das nützlich sein kann, um im unglücklichen Fall eines Fehlers detaillierte Fehlerberichte zu ermöglichen. Auf Plattformen mit DTrace-Unterstützung kann auch `--enable-dtrace` für Produktion sinnvoll sein.

Wenn eine Installation gebaut wird, mit der Code innerhalb des Servers entwickelt werden soll, wird empfohlen, mindestens `--enable-debug` und `--enable-cassert` zu verwenden.

- `--enable-debug`

  Kompiliert alle Programme und Bibliotheken mit Debug-Symbolen. Dadurch können Sie die Programme in einem Debugger ausführen, um Probleme zu analysieren. Die installierten ausführbaren Dateien werden erheblich größer, und bei Nicht-GCC-Compilern wird häufig auch Compileroptimierung deaktiviert, was Verlangsamungen verursacht. Die verfügbaren Symbole sind jedoch äußerst hilfreich, wenn Probleme auftreten. Derzeit wird diese Option für Produktionsinstallationen nur empfohlen, wenn Sie GCC verwenden. Für Entwicklungsarbeit oder eine Beta-Version sollten Sie sie immer aktivieren.

- `--enable-cassert`

  Aktiviert Assertion-Prüfungen im Server, die viele „kann nicht passieren“-Bedingungen testen. Das ist für Codeentwicklung sehr wertvoll, kann den Server aber deutlich verlangsamen. Außerdem erhöht das Aktivieren dieser Tests nicht zwangsläufig die Stabilität des Servers. Assertion-Prüfungen sind nicht nach Schweregrad kategorisiert; daher kann auch ein relativ harmloser Fehler zu Serverneustarts führen, wenn er eine Assertion auslöst. Diese Option wird für Produktion nicht empfohlen, sollte aber bei Entwicklungsarbeit oder beim Betrieb einer Beta-Version aktiv sein.

- `--enable-tap-tests`

  Aktiviert Tests mit den Perl-TAP-Werkzeugen. Dafür werden eine Perl-Installation und das Perl-Modul `IPC::Run` benötigt. Weitere Informationen finden Sie in [Abschnitt 31.4](31_Regressionstests.md#314-taptests).

- `--enable-depend`

  Aktiviert automatische Abhängigkeitsverfolgung. Mit dieser Option werden die Makefiles so eingerichtet, dass alle betroffenen Objektdateien neu gebaut werden, wenn eine Header-Datei geändert wird. Das ist für Entwicklungsarbeit nützlich, aber nur unnötiger Overhead, wenn Sie lediglich einmal kompilieren und installieren möchten. Derzeit funktioniert diese Option nur mit GCC.

- `--enable-coverage`

  Bei Verwendung von GCC werden alle Programme und Bibliotheken mit Instrumentierung für Code-Coverage-Tests kompiliert. Beim Ausführen erzeugen sie im Build-Verzeichnis Dateien mit Code-Coverage-Metriken. Weitere Informationen finden Sie in [Abschnitt 31.5](31_Regressionstests.md#315-testabdeckung-untersuchen). Diese Option ist nur für GCC und Entwicklungsarbeit gedacht.

- `--enable-profiling`

  Bei Verwendung von GCC werden alle Programme und Bibliotheken so kompiliert, dass sie profiliert werden können. Beim Beenden eines Backends wird ein Unterverzeichnis erzeugt, das die Datei `gmon.out` mit Profildaten enthält. Diese Option ist nur für GCC und Entwicklungsarbeit gedacht.

- `--enable-dtrace`

  Kompiliert PostgreSQL mit Unterstützung für das dynamische Tracing-Werkzeug DTrace. Weitere Informationen finden Sie in [Abschnitt 27.5](27_Datenbankaktivität_überwachen.md#275-dynamisches-tracing).

  Um auf das Programm `dtrace` zu verweisen, kann die Umgebungsvariable `DTRACE` gesetzt werden. Das ist häufig notwendig, weil `dtrace` typischerweise unter `/usr/sbin` installiert ist, das möglicherweise nicht in Ihrem `PATH` liegt.

  Zusätzliche Kommandozeilenoptionen für `dtrace` können in der Umgebungsvariable `DTRACEFLAGS` angegeben werden. Unter Solaris müssen Sie für DTrace-Unterstützung in einer 64-Bit-Binärdatei `DTRACEFLAGS="-64"` angeben. Beispiel mit GCC:

```text
./configure CC='gcc -m64' --enable-dtrace DTRACEFLAGS='-64' ...
```

  Mit Suns Compiler:

```text
./configure CC='/opt/SUNWspro/bin/cc -xtarget=native64' --enable-dtrace DTRACEFLAGS='-64' ...
```

- `--enable-injection-points`

  Kompiliert PostgreSQL mit Unterstützung für Injection Points im Server. Injection Points erlauben es, benutzerdefinierten Code innerhalb des Servers in vordefinierten Codepfaden auszuführen. Das hilft beim Testen und bei der Untersuchung von Nebenläufigkeitsszenarien unter kontrollierten Bedingungen. Diese Option ist standardmäßig deaktiviert. Weitere Details finden Sie in Abschnitt 36.10.14. Diese Option ist nur für Entwicklertests gedacht.

- `--with-segsize-blocks=SEGSIZE_BLOCKS`

  Gibt die Größe von Relationssegmenten in Blöcken an. Wenn sowohl `--with-segsize` als auch diese Option angegeben sind, gewinnt diese Option. Sie ist nur für Entwickler gedacht, um segmentbezogenen Code zu testen.

### 17.3.4. configure-Umgebungsvariablen

Zusätzlich zu den oben beschriebenen normalen Kommandozeilenoptionen reagiert `configure` auf eine Reihe von Umgebungsvariablen. Sie können Umgebungsvariablen auf der `configure`-Kommandozeile angeben:

```text
./configure CC=/opt/bin/gcc CFLAGS='-O2 -pipe'
```

In dieser Verwendung unterscheidet sich eine Umgebungsvariable kaum von einer Kommandozeilenoption. Sie können solche Variablen auch vorher setzen:

```text
export CC=/opt/bin/gcc
export CFLAGS='-O2 -pipe'
./configure
```

Das kann praktisch sein, weil viele Konfigurationsskripte ähnlich auf diese Variablen reagieren.

Die am häufigsten verwendeten Variablen sind `CC` und `CFLAGS`. Wenn Sie einen anderen C-Compiler bevorzugen als den von `configure` gewählten, setzen Sie `CC` auf das gewünschte Programm. Standardmäßig wählt `configure` `gcc`, falls verfügbar, andernfalls den Plattformstandard, meist `cc`. Entsprechend können Sie mit `CFLAGS` bei Bedarf die Standard-Compilerflags überschreiben.

Wichtige Variablen, die auf diese Weise gesetzt werden können:

| Variable | Bedeutung |
| --- | --- |
| `BISON` | Bison-Programm |
| `CC` | C-Compiler |
| `CFLAGS` | Optionen für den C-Compiler |
| `CLANG` | Pfad zum `clang`-Programm, das bei `--with-llvm` zum Verarbeiten von Quellcode für Inlining verwendet wird |
| `CPP` | C-Präprozessor |
| `CPPFLAGS` | Optionen für den C-Präprozessor |
| `CXX` | C++-Compiler |
| `CXXFLAGS` | Optionen für den C++-Compiler |
| `DTRACE` | Ort des Programms `dtrace` |
| `DTRACEFLAGS` | Optionen für das Programm `dtrace` |
| `FLEX` | Flex-Programm |
| `LDFLAGS` | Optionen zum Linken ausführbarer Dateien oder Shared Libraries |
| `LDFLAGS_EX` | zusätzliche Optionen nur zum Linken ausführbarer Dateien |
| `LDFLAGS_SL` | zusätzliche Optionen nur zum Linken von Shared Libraries |
| `LLVM_CONFIG` | `llvm-config`-Programm zum Auffinden der LLVM-Installation |
| `MSGFMT` | `msgfmt`-Programm für Native Language Support |
| `PERL` | Perl-Interpreter; er wird verwendet, um die Abhängigkeiten zum Bauen von PL/Perl zu bestimmen. Standard ist `perl`. |
| `PYTHON` | Python-Interpreter; er wird verwendet, um die Abhängigkeiten zum Bauen von PL/Python zu bestimmen. Falls nicht gesetzt, wird in dieser Reihenfolge gesucht: `python3`, `python`. |
| `TCLSH` | Tcl-Interpreter; er wird verwendet, um die Abhängigkeiten zum Bauen von PL/Tcl zu bestimmen. Falls nicht gesetzt, wird in dieser Reihenfolge gesucht: `tclsh`, `tcl`, `tclsh8.6`, `tclsh86`, `tclsh8.5`, `tclsh85`, `tclsh8.4`, `tclsh84`. |
| `XML2_CONFIG` | `xml2-config`-Programm zum Auffinden der libxml2-Installation |

Manchmal ist es nützlich, Compilerflags nachträglich zu den von `configure` gewählten Flags hinzuzufügen. Ein wichtiges Beispiel ist GCCs Option `-Werror`, die nicht in den an `configure` übergebenen `CFLAGS` enthalten sein kann, weil sie viele eingebaute Tests von `configure` brechen würde. Um solche Flags hinzuzufügen, setzen Sie beim Ausführen von `make` die Umgebungsvariable `COPT`. Der Inhalt von `COPT` wird zu den von `configure` eingerichteten Optionen `CFLAGS`, `CXXFLAGS` und `LDFLAGS` hinzugefügt. Beispiele:

```text
make COPT='-Werror'
```

oder:

```text
export COPT='-Werror'
make
```

> **Hinweis:** Bei GCC ist es am besten, mit mindestens Optimierungsstufe `-O1` zu bauen, weil keine Optimierung (`-O0`) einige wichtige Compilerwarnungen deaktiviert, etwa die Verwendung nicht initialisierter Variablen. Nicht-null Optimierungsstufen können das Debugging allerdings erschweren, weil das schrittweise Durchlaufen kompilierten Codes meist nicht eins zu eins den Quellcodezeilen entspricht. Wenn Sie beim Debuggen optimierten Codes durcheinanderkommen, kompilieren Sie die betreffenden Dateien mit `-O0` neu. Ein einfacher Weg ist, `make` eine Option zu übergeben: `make PROFILE=-O0 file.o`.
>
> Die Umgebungsvariablen `COPT` und `PROFILE` werden von den PostgreSQL-Makefiles tatsächlich identisch behandelt. Welche Sie verwenden, ist Geschmackssache. Unter Entwicklern ist es üblich, `PROFILE` für einmalige Flag-Anpassungen zu verwenden, während `COPT` dauerhaft gesetzt bleiben kann.

## 17.4. Bauen und Installieren mit Meson

### 17.4.1. Kurzfassung

```text
meson setup build --prefix=/usr/local/pgsql
cd build
ninja
su
ninja install
adduser postgres
mkdir -p /usr/local/pgsql/data
chown postgres /usr/local/pgsql/data
su - postgres
/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start
/usr/local/pgsql/bin/createdb test
/usr/local/pgsql/bin/psql test
```

Die ausführliche Fassung ist der Rest dieses Abschnitts.

### 17.4.2. Installationsprozedur

1. **Konfiguration**

   Der erste Schritt besteht darin, den Build-Baum für Ihr System zu konfigurieren und die gewünschten Optionen auszuwählen. Zum Erstellen und Konfigurieren des Build-Verzeichnisses beginnen Sie mit dem Befehl `meson setup`:

```text
meson setup build
```

   Der Setup-Befehl akzeptiert ein `builddir`- und ein `srcdir`-Argument. Wenn kein `srcdir` angegeben ist, leitet Meson es aus dem aktuellen Verzeichnis und der Position von `meson.build` ab. `builddir` ist erforderlich.

   `meson setup` lädt die Build-Konfigurationsdatei und richtet das Build-Verzeichnis ein. Zusätzlich können Sie Meson mehrere Build-Optionen übergeben. Einige häufig verwendete Optionen werden in den folgenden Abschnitten erwähnt:

```text
# configure with a different installation prefix
meson setup build --prefix=/home/user/pg-install

# configure to generate a debug build
meson setup build --buildtype=debug

# configure to build with OpenSSL support
meson setup build -Dssl=openssl
```

   Das Einrichten des Build-Verzeichnisses ist ein einmaliger Schritt. Zum Neukonfigurieren vor einem neuen Build können Sie einfach `meson configure` verwenden:

```text
meson configure -Dcassert=true
```

   Häufig verwendete Kommandozeilenoptionen von `meson configure` werden in [Abschnitt 17.4.3](#1743-meson-setupoptionen) erläutert.

2. **Bauen**

   Standardmäßig verwendet Meson das Build-Werkzeug Ninja. Weitere Informationen finden Sie unter <https://ninja-build.org/>. Zum Bauen von PostgreSQL aus dem Quellcode mit Meson verwenden Sie im Build-Verzeichnis einfach:

```text
ninja
```

   Ninja erkennt automatisch die Zahl der CPUs Ihres Rechners und parallelisiert entsprechend. Die Zahl paralleler Prozesse können Sie mit dem Kommandozeilenargument `-j` überschreiben.

   Nach dem ersten Konfigurationsschritt ist `ninja` normalerweise der einzige Befehl, den Sie zum Kompilieren eingeben müssen. Egal wie Sie den Quellbaum verändern, solange Sie ihn nicht an einen ganz anderen Ort verschieben, erkennt Meson die Änderungen und regeneriert sich entsprechend. Das ist besonders praktisch, wenn Sie mehrere Build-Verzeichnisse haben. Oft wird eines für Entwicklung verwendet, etwa ein Debug-Build, und andere nur gelegentlich, etwa für statische Analyse. Jede Konfiguration lässt sich bauen, indem Sie in das passende Verzeichnis wechseln und Ninja ausführen.

   Wenn Sie mit einem anderen Backend als Ninja bauen möchten, können Sie mit der Option `--backend` das gewünschte Backend auswählen und dann mit `meson compile` bauen. Weitere Informationen zu Backends und weiteren Ninja-Argumenten finden Sie in der Meson-Dokumentation: <https://mesonbuild.com/Running-Meson.html#building-from-the-source>.

3. **Regressionstests**

   Wenn Sie den neu gebauten Server vor der Installation testen möchten, können Sie jetzt die Regressionstests ausführen:

```text
meson test
```

   Dies funktioniert nicht als `root`; führen Sie es als unprivilegierter Benutzer aus. Ausführliche Informationen zur Interpretation der Testergebnisse finden Sie in [Kapitel 31](31_Regressionstests.md). Sie können den Test später jederzeit mit demselben Befehl wiederholen.

   Um Tests von `pg_regress` und `pg_isolation_regress` gegen eine laufende `postgres`-Instanz auszuführen, geben Sie `--setup running` als Argument für `meson test` an.

4. **Dateien installieren**

> **Hinweis:** Wenn Sie ein bestehendes System aktualisieren, lesen Sie unbedingt [Abschnitt 18.6](18_Servereinrichtung_und_Betrieb.md#186-einen-postgresqlcluster-aktualisieren), der Anweisungen zum Aktualisieren eines Clusters enthält.

   Sobald PostgreSQL gebaut ist, können Sie es installieren, indem Sie einfach `ninja install` ausführen:

```text
ninja install
```

   Dadurch werden Dateien in die Verzeichnisse installiert, die in Schritt 1 angegeben wurden. Stellen Sie sicher, dass Sie Schreibrechte für diesen Bereich haben. Möglicherweise müssen Sie diesen Schritt als `root` ausführen. Alternativ können Sie die Zielverzeichnisse vorher erstellen und passende Berechtigungen vergeben. Die Standardinstallation stellt alle Header-Dateien bereit, die für die Entwicklung von Client-Anwendungen und serverseitige Programmentwicklung benötigt werden, zum Beispiel eigene Funktionen oder Datentypen in C.

   `ninja install` sollte in den meisten Fällen funktionieren. Wenn Sie mehr Optionen verwenden möchten, etwa `--quiet`, um zusätzliche Ausgaben zu unterdrücken, können Sie stattdessen `meson install` verwenden. Weitere Informationen zu `meson install` und seinen Optionen finden Sie in der Meson-Dokumentation: <https://mesonbuild.com/Commands.html#install>.

   **Deinstallation:** Um die Installation rückgängig zu machen, können Sie `ninja uninstall` verwenden.

   **Bereinigung:** Nach der Installation können Sie Plattenplatz freigeben, indem Sie mit `ninja clean` die gebauten Dateien aus dem Quellbaum entfernen.

### 17.4.3. meson setup-Optionen

Die Kommandozeilenoptionen von `meson setup` werden unten erläutert. Diese Liste ist nicht vollständig; eine vollständige Liste erhalten Sie mit `meson configure --help`. Die hier nicht behandelten Optionen sind für fortgeschrittene Anwendungsfälle gedacht und in der Meson-Standarddokumentation beschrieben. Diese Argumente können auch mit `meson setup` verwendet werden. Siehe <https://mesonbuild.com/Commands.html#configure>.

#### 17.4.3.1. Installationsorte

Diese Optionen steuern, wo `ninja install` oder `meson install` die Dateien ablegt. Die Option `--prefix`, siehe Beispiel in [Abschnitt 17.4.1](#1741-kurzfassung), genügt in den meisten Fällen. Wenn Sie besondere Anforderungen haben, können Sie die Installationsunterverzeichnisse mit den weiteren Optionen dieses Abschnitts anpassen. Beachten Sie jedoch, dass Änderungen an den relativen Positionen der verschiedenen Unterverzeichnisse die Installation nicht verschiebbar machen können. Die Orte für Manpages und Dokumentation sind von dieser Einschränkung nicht betroffen. Für verschiebbare Installationen möchten Sie möglicherweise die später beschriebene Option `-Drpath=false` verwenden.

- `--prefix=PREFIX`

  Installiert alle Dateien unterhalb von `PREFIX` statt unter `/usr/local/pgsql` auf Unix-basierten Systemen oder unter `aktueller Laufwerksbuchstabe:/usr/local/pgsql` auf Windows. Die tatsächlichen Dateien werden in verschiedene Unterverzeichnisse installiert; direkt in `PREFIX` werden nie Dateien installiert.

- `--bindir=DIRECTORY`

  Gibt das Verzeichnis für ausführbare Programme an. Standard ist `PREFIX/bin`.

- `--sysconfdir=DIRECTORY`

  Setzt das Verzeichnis für verschiedene Konfigurationsdateien, standardmäßig `PREFIX/etc`.

- `--libdir=DIRECTORY`

  Setzt den Installationsort für Bibliotheken und dynamisch ladbare Module. Standard ist `PREFIX/lib`.

- `--includedir=DIRECTORY`

  Setzt das Verzeichnis für C- und C++-Header-Dateien. Standard ist `PREFIX/include`.

- `--datadir=DIRECTORY`

  Setzt das Verzeichnis für schreibgeschützte Datendateien, die von installierten Programmen verwendet werden. Standard ist `PREFIX/share`. Dies hat nichts damit zu tun, wo Ihre Datenbankdateien abgelegt werden.

- `--localedir=DIRECTORY`

  Setzt das Verzeichnis für Locale-Daten, insbesondere Katalogdateien für Meldungsübersetzungen. Standard ist `DATADIR/locale`.

- `--mandir=DIRECTORY`

  Die mit PostgreSQL gelieferten Manpages werden unterhalb dieses Verzeichnisses in ihren jeweiligen `manx`-Unterverzeichnissen installiert. Standard ist `DATADIR/man`.

> **Hinweis:** Wie bei Autoconf wurde darauf geachtet, PostgreSQL in gemeinsam genutzte Installationsorte installieren zu können, ohne den Namensraum des Systems zu stören. Die Zeichenkette `/postgresql` wird automatisch an `datadir`, `sysconfdir` und `docdir` angehängt, sofern der vollständig expandierte Verzeichnisname nicht bereits `postgres` oder `pgsql` enthält. Öffentliche Header-Dateien der Client-Schnittstellen werden in `includedir` installiert und sind namensraumsauber; interne und Server-Header-Dateien landen in privaten Verzeichnissen. Auch unter `libdir` wird bei Bedarf ein privates Unterverzeichnis für dynamisch ladbare Module erstellt.

#### 17.4.3.2. PostgreSQL-Funktionen

Die Optionen in diesem Abschnitt aktivieren den Bau verschiedener optionaler PostgreSQL-Funktionen. Die meisten erfordern zusätzliche Software, wie in [Abschnitt 17.1](#171-anforderungen) beschrieben, und werden automatisch aktiviert, wenn die benötigte Software gefunden wird. Dieses Verhalten können Sie ändern, indem Sie Funktionen manuell auf `enabled` setzen, um sie zu verlangen, oder auf `disabled`, um sie nicht zu bauen.

PostgreSQL-spezifische Optionen müssen mit `-D` vorangestellt werden.

- `-Dnls={ auto | enabled | disabled }`

  Aktiviert oder deaktiviert Native Language Support (NLS), also die Fähigkeit, Programmmeldungen in einer anderen Sprache als Englisch anzuzeigen. Standard ist `auto`; NLS wird automatisch aktiviert, wenn eine Implementierung der Gettext-API gefunden wird.

- `-Dplperl={ auto | enabled | disabled }`

  Baut die serverseitige Sprache PL/Perl. Standard ist `auto`.

- `-Dplpython={ auto | enabled | disabled }`

  Baut die serverseitige Sprache PL/Python. Standard ist `auto`.

- `-Dpltcl={ auto | enabled | disabled }`

  Baut die serverseitige Sprache PL/Tcl. Standard ist `auto`.

- `-Dtcl_version=TCL_VERSION`

  Gibt die Tcl-Version an, die beim Bauen von PL/Tcl verwendet werden soll.

- `-Dicu={ auto | enabled | disabled }`

  Baut mit Unterstützung für die ICU-Bibliothek und aktiviert ICU-Collation-Funktionen; siehe [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung). Standard ist `auto`; dafür muss ICU4C installiert sein. Die derzeit mindestens erforderliche ICU4C-Version ist 4.2.

- `-Dllvm={ auto | enabled | disabled }`

  Baut mit Unterstützung für LLVM-basierte JIT-Kompilierung; siehe [Kapitel 30](30_Just_in_Time_Kompilierung_JIT.md). Dafür muss die LLVM-Bibliothek installiert sein. Die derzeit mindestens erforderliche LLVM-Version ist 14. Standardmäßig ist dies deaktiviert.

  `llvm-config` und anschließend `llvm-config-$version` für alle unterstützten Versionen werden in Ihrem `PATH` gesucht. Wenn das nicht das gewünschte Programm ergibt, verwenden Sie `LLVM_CONFIG`, um den Pfad zum richtigen `llvm-config` anzugeben.

- `-Dlz4={ auto | enabled | disabled }`

  Baut mit LZ4-Kompressionsunterstützung. Standard ist `auto`.

- `-Dzstd={ auto | enabled | disabled }`

  Baut mit Zstandard-Kompressionsunterstützung. Standard ist `auto`.

- `-Dssl={ auto | LIBRARY }`

  Baut mit Unterstützung für SSL-Verbindungen. Die einzige unterstützte `LIBRARY` ist `openssl`. Dafür muss OpenSSL installiert sein. Vor dem Fortfahren werden Header-Dateien und Bibliotheken geprüft. Standard ist `auto`.

- `-Dgssapi={ auto | enabled | disabled }`

  Baut mit Unterstützung für GSSAPI-Authentifizierung. Dafür muss MIT Kerberos installiert sein. Auf vielen Systemen liegt GSSAPI nicht an einem Standardsuchort. In solchen Fällen fragt PostgreSQL `pkg-config` ab, um die benötigten Compiler- und Linkeroptionen zu finden. Standard ist `auto`.

- `-Dldap={ auto | enabled | disabled }`

  Baut mit LDAP-Unterstützung für Authentifizierung und Nachschlagen von Verbindungsparametern; siehe [Abschnitt 32.18](32_libpq_C_Bibliothek.md#3218-ldapsuche-nach-verbindungsparametern) und [Abschnitt 20.10](20_Client_Authentifizierung.md#2010-ldapauthentifizierung). Unter Unix ist OpenLDAP erforderlich; unter Windows wird die standardmäßige WinLDAP-Bibliothek verwendet. Standard ist `auto`.

- `-Dpam={ auto | enabled | disabled }`

  Baut mit PAM-Unterstützung. Standard ist `auto`.

- `-Dbsd_auth={ auto | enabled | disabled }`

  Baut mit BSD-Authentication-Unterstützung. Dieses Framework ist derzeit nur auf OpenBSD verfügbar. Standard ist `auto`.

- `-Dsystemd={ auto | enabled | disabled }`

  Baut mit Unterstützung für systemd-Service-Benachrichtigungen. Das verbessert die Integration, wenn der Server unter systemd gestartet wird, hat ansonsten aber keine Auswirkungen; siehe [Abschnitt 18.3](18_Servereinrichtung_und_Betrieb.md#183-den-datenbankserver-starten). Standard ist `auto`. `libsystemd` und die zugehörigen Header-Dateien müssen installiert sein.

- `-Dbonjour={ auto | enabled | disabled }`

  Baut mit Unterstützung für automatische Dienstsuche über Bonjour. Standard ist `auto`; Ihr Betriebssystem muss Bonjour unterstützen. Auf macOS wird dies empfohlen.

- `-Duuid=LIBRARY`

  Baut das Modul `uuid-ossp`, das Funktionen zum Erzeugen von UUIDs bereitstellt, mit der angegebenen UUID-Bibliothek. `LIBRARY` muss eine der folgenden sein:

  - `none`, um das UUID-Modul nicht zu bauen; dies ist der Standard
  - `bsd`, um UUID-Funktionen aus FreeBSD und einigen anderen BSD-abgeleiteten Systemen zu verwenden
  - `e2fs`, um die vom Projekt e2fsprogs erzeugte UUID-Bibliothek zu verwenden
  - `ossp`, um die OSSP-UUID-Bibliothek zu verwenden: <http://www.ossp.org/pkg/lib/uuid/>

- `-Dlibcurl={ auto | enabled | disabled }`

  Baut mit libcurl-Unterstützung für OAuth-2.0-Client-Flows. Dafür ist libcurl Version 7.61.0 oder neuer erforderlich. Standard ist `auto`.

- `-Dliburing={ auto | enabled | disabled }`

  Baut mit `liburing` und aktiviert damit `io_uring`-Unterstützung für asynchrones I/O. Standard ist `auto`. Für Installationen an ungewöhnlichen Orten können `pkg-config`-bezogene Umgebungsvariablen gesetzt werden.

- `-Dlibnuma={ auto | enabled | disabled }`

  Baut mit `libnuma`-Unterstützung für grundlegende NUMA-Unterstützung. Dies wird nur auf Plattformen unterstützt, auf denen `libnuma` implementiert ist. Standard ist `auto`.

- `-Dlibxml={ auto | enabled | disabled }`

  Baut mit `libxml2` und aktiviert SQL/XML-Unterstützung. Standard ist `auto`. Für diese Funktion ist libxml2 Version 2.6.23 oder neuer erforderlich.

- `-Dlibxslt={ auto | enabled | disabled }`

  Baut mit `libxslt` und ermöglicht dem Modul `xml2`, XSL-Transformationen von XML auszuführen. `-Dlibxml` muss ebenfalls angegeben werden. Standard ist `auto`.

- `-Dselinux={ auto | enabled | disabled }`

  Baut mit SELinux-Unterstützung und aktiviert die Erweiterung `sepgsql`. Standard ist `auto`.

#### 17.4.3.3. Anti-Features

- `-Dreadline={ auto | enabled | disabled }`

  Erlaubt die Verwendung der Readline-Bibliothek und auch von `libedit`. Standard ist `auto`; damit werden Kommandozeilenbearbeitung und Verlauf in `psql` aktiviert, was dringend empfohlen wird.

- `-Dlibedit_preferred={ true | false }`

  Wenn dies auf `true` gesetzt ist, wird die BSD-lizenzierte Bibliothek `libedit` gegenüber der GPL-lizenzierten Readline-Bibliothek bevorzugt. Diese Option ist nur relevant, wenn beide Bibliotheken installiert sind. Standard ist `false`, also Readline.

- `-Dzlib={ auto | enabled | disabled }`

  Aktiviert die Verwendung der Zlib-Bibliothek. Standard ist `auto`; damit wird Unterstützung für komprimierte Archive in `pg_dump`, `pg_restore` und `pg_basebackup` aktiviert. Dies wird empfohlen.

#### 17.4.3.4. Details zum Build-Prozess

- `--auto-features={ auto | enabled | disabled }`

  Mit dieser Option können Sie den Wert aller `auto`-Features überschreiben, also aller Funktionen, die automatisch aktiviert werden, wenn die benötigte Software gefunden wird. Das ist nützlich, wenn Sie alle optionalen Funktionen auf einmal deaktivieren oder aktivieren möchten, ohne jede einzeln zu setzen. Standard ist `auto`.

- `--backend=BACKEND`

  Das standardmäßige Meson-Backend ist `ninja` und reicht für die meisten Fälle aus. Wenn Sie jedoch vollständig in Visual Studio integrieren möchten, können Sie `BACKEND` auf `vs` setzen.

- `-Dc_args=OPTIONS`

  Übergibt zusätzliche Optionen an den C-Compiler.

- `-Dc_link_args=OPTIONS`

  Übergibt zusätzliche Optionen an den C-Linker.

- `-Dextra_include_dirs=DIRECTORIES`

  `DIRECTORIES` ist eine durch Kommata getrennte Liste von Verzeichnissen, die zur Suchliste des Compilers für Header-Dateien hinzugefügt werden. Wenn optionale Pakete wie GNU Readline an einem nicht standardmäßigen Ort installiert sind, müssen Sie diese Option und wahrscheinlich auch `-Dextra_lib_dirs` verwenden. Beispiel: `-Dextra_include_dirs=/opt/gnu/include,/usr/sup/include`.

- `-Dextra_lib_dirs=DIRECTORIES`

  `DIRECTORIES` ist eine durch Kommata getrennte Liste von Verzeichnissen, in denen nach Bibliotheken gesucht wird. Beispiel: `-Dextra_lib_dirs=/opt/gnu/lib,/usr/sup/lib`.

- `-Dsystem_tzdata=DIRECTORY`

  Verwendet die vom System bereitgestellte Zeitzonendatenbank in `DIRECTORY` statt der in der PostgreSQL-Quellcode-Distribution enthaltenen. Die Beschreibung entspricht der Option `--with-system-tzdata` im Autoconf-Build. `DIRECTORY` muss ein absoluter Pfad sein, zum Beispiel `/usr/share/zoneinfo`.

- `-Dextra_version=STRING`

  Hängt `STRING` an die PostgreSQL-Versionsnummer an, zum Beispiel für unveröffentlichte Git-Snapshots, eigene Patches oder Paket-Release-Nummern.

- `-Drpath={ true | false }`

  Standard ist `true`. Bei `false` werden PostgreSQLs ausführbare Dateien nicht so markiert, dass sie im Bibliotheksverzeichnis der Installation nach Shared Libraries suchen; siehe `--libdir`. Dann müssen Sie auf andere Weise dafür sorgen, dass die ausführbaren Dateien die Shared Libraries finden, typischerweise durch Konfiguration des dynamischen Linkers; siehe [Abschnitt 17.5.1](#1751-shared-libraries).

- `-DBINARY_NAME=PATH`

  Wenn ein zum Bauen benötigtes Programm an einem nicht standardmäßigen Pfad liegt, können Sie es manuell für `meson configure` angeben. Die vollständige Liste unterstützter Programme erhalten Sie mit `meson configure`. Beispiel:

```text
meson configure -DBISON=PATH_TO_BISON
```

#### 17.4.3.5. Dokumentation

Die zum Bauen der Dokumentation benötigten Werkzeuge finden Sie in Abschnitt J.2.

- `-Ddocs={ auto | enabled | disabled }`

  Aktiviert das Bauen der Dokumentation im HTML- und Manpage-Format. Standard ist `auto`.

- `-Ddocs_pdf={ auto | enabled | disabled }`

  Aktiviert das Bauen der Dokumentation im PDF-Format. Standard ist `auto`.

- `-Ddocs_html_style={ simple | website }`

  Steuert, welches CSS-Stylesheet verwendet wird. Standard ist `simple`. Bei `website` verweist die HTML-Dokumentation auf das Stylesheet für postgresql.org: <https://www.postgresql.org/docs/current/>.

#### 17.4.3.6. Verschiedenes

- `-Dpgport=NUMBER`

  Setzt `NUMBER` als Standardportnummer für Server und Clients. Standard ist 5432. Der Port kann später immer geändert werden. Wenn Sie ihn hier angeben, ist jedoch für Server und Clients derselbe Standard einkompiliert. Ein vom Standard abweichender Wert ist meist nur sinnvoll, wenn mehrere PostgreSQL-Server auf derselben Maschine laufen sollen.

- `-Dkrb_srvnam=NAME`

  Der Standardname des von GSSAPI verwendeten Kerberos-Service-Principals. Standard ist `postgres`. Meist gibt es keinen Grund, dies zu ändern, außer beim Bauen für Windows, wo der Name in Großbuchstaben `POSTGRES` lauten muss.

- `-Dsegsize=SEGSIZE`

  Setzt die Segmentgröße in Gigabyte. Große Tabellen werden in mehrere Betriebssystemdateien aufgeteilt, deren Größe jeweils der Segmentgröße entspricht. Die Standardsegmentgröße von 1 Gigabyte ist auf allen unterstützten Plattformen sicher. Größere Werte können bei sehr großen Tabellen die Zahl der verbrauchten Dateideskriptoren verringern. Wählen Sie keinen Wert, der größer ist als von Plattform und Dateisystemen unterstützt. Es wird empfohlen, eine Potenz von 2 zu verwenden.

- `-Dblocksize=BLOCKSIZE`

  Setzt die Blockgröße in Kilobyte. Dies ist die Einheit für Speicherung und I/O innerhalb von Tabellen. Der Standardwert von 8 Kilobyte passt für die meisten Situationen. Der Wert muss eine Potenz von 2 zwischen 1 und 32 Kilobyte sein.

- `-Dwal_blocksize=BLOCKSIZE`

  Setzt die WAL-Blockgröße in Kilobyte. Dies ist die Einheit für Speicherung und I/O im WAL-Log. Der Standardwert von 8 Kilobyte passt für die meisten Situationen. Der Wert muss eine Potenz von 2 zwischen 1 und 64 Kilobyte sein.

#### 17.4.3.7. Entwickleroptionen

Die meisten Optionen in diesem Abschnitt sind nur für die Entwicklung oder das Debugging von PostgreSQL interessant. Sie werden für Produktions-Builds nicht empfohlen, außer `--debug`, das detaillierte Fehlerberichte erleichtern kann. Auf Plattformen mit DTrace-Unterstützung kann auch `-Ddtrace` für Produktion sinnvoll sein.

Für eine Installation, die zur Entwicklung von Servercode verwendet wird, werden mindestens `--buildtype=debug` und `-Dcassert` empfohlen.

- `--buildtype=BUILDTYPE`

  Gibt den Build-Typ an; Standard ist `debugoptimized`. Wenn Sie genauere Kontrolle über Debug-Symbole und Optimierungsstufen wünschen, verwenden Sie `--debug` und `--optimization`. Übliche Build-Typen sind `plain`, `debug`, `debugoptimized` und `release`.

- `--debug`

  Kompiliert alle Programme und Bibliotheken mit Debug-Symbolen. Dies vergrößert installierte ausführbare Dateien deutlich und deaktiviert auf Nicht-GCC-Compilern häufig die Optimierung, ist aber zur Problemanalyse sehr hilfreich.

- `--optimization=LEVEL`

  Gibt die Optimierungsstufe an. `LEVEL` kann einen der Werte `{0,g,1,2,3,s}` haben.

- `--werror`

  Veranlasst den Compiler, Warnungen als Fehler zu behandeln. Das kann bei Codeentwicklung nützlich sein.

- `-Dcassert={ true | false }`

  Aktiviert Assertion-Prüfungen im Server. Das ist für Codeentwicklung sehr wertvoll, kann den Server aber deutlich verlangsamen und wird für Produktion nicht empfohlen.

- `-Dtap_tests={ auto | enabled | disabled }`

  Aktiviert Tests mit den Perl-TAP-Werkzeugen. Standard ist `auto`; dafür werden eine Perl-Installation und das Perl-Modul `IPC::Run` benötigt. Siehe [Abschnitt 31.4](31_Regressionstests.md#314-taptests).

- `-DPG_TEST_EXTRA=TEST_SUITES`

  Aktiviert zusätzliche Testsuiten, die standardmäßig nicht ausgeführt werden, weil sie auf Mehrbenutzersystemen unsicher sind, besondere Software benötigen oder ressourcenintensiv sind. Das Argument ist eine durch Leerraum getrennte Liste der zu aktivierenden Tests. Siehe Abschnitt 31.1.3. Wenn die Umgebungsvariable `PG_TEST_EXTRA` beim Ausführen der Tests gesetzt ist, überschreibt sie diese Setup-Option.

- `-Db_coverage={ true | false }`

  Bei Verwendung von GCC werden alle Programme und Bibliotheken mit Code-Coverage-Instrumentierung kompiliert. Weitere Informationen finden Sie in [Abschnitt 31.5](31_Regressionstests.md#315-testabdeckung-untersuchen).

- `-Ddtrace={ auto | enabled | disabled }`

  Aktiviert DTrace-Unterstützung. Siehe [Abschnitt 27.5](27_Datenbankaktivität_überwachen.md#275-dynamisches-tracing). Um auf das Programm `dtrace` zu verweisen, kann die Option `DTRACE` gesetzt werden.

- `-Dinjection_points={ true | false }`

  Kompiliert PostgreSQL mit Unterstützung für Injection Points im Server. Diese erlauben benutzerdefinierten Code in vordefinierten Codepfaden und helfen beim Testen sowie bei kontrollierten Untersuchungen von Nebenläufigkeit. Standardmäßig deaktiviert; nur für Entwicklertests gedacht. Siehe Abschnitt 36.10.14.

- `-Dsegsize_blocks=SEGSIZE_BLOCKS`

  Gibt die Größe von Relationssegmenten in Blöcken an. Wenn sowohl `-Dsegsize` als auch diese Option angegeben sind, gewinnt diese Option. Sie dient nur Entwicklern zum Testen segmentbezogenen Codes.

### 17.4.4. Meson-Build-Targets

Einzelne Build-Targets können mit `ninja target` gebaut werden. Wenn kein Target angegeben wird, wird alles außer der Dokumentation gebaut. Einzelne Build-Produkte können mit `path/filename` als Target gebaut werden.

#### 17.4.4.1. Code-Targets

| Target | Bedeutung |
| --- | --- |
| `all` | baut alles außer Dokumentation |
| `backend` | baut Backend und zugehörige Module |
| `bin` | baut Frontend-Binärdateien |
| `contrib` | baut Contrib-Module |
| `pl` | baut prozedurale Sprachen |

#### 17.4.4.2. Entwickler-Targets

| Target | Bedeutung |
| --- | --- |
| `reformat-dat-files` | schreibt Katalogdatendateien ins Standardformat um |
| `expand-dat-files` | erweitert alle Datendateien um Standardwerte |
| `update-unicode` | aktualisiert Unicode-Daten auf eine neue Version |

#### 17.4.4.3. Dokumentations-Targets

| Target | Bedeutung |
| --- | --- |
| `html` | baut Dokumentation im mehrseitigen HTML-Format |
| `man` | baut Dokumentation im Manpage-Format |
| `docs` | baut Dokumentation im mehrseitigen HTML- und Manpage-Format |
| `doc/src/sgml/postgres-A4.pdf` | baut Dokumentation im PDF-Format mit A4-Seiten |
| `doc/src/sgml/postgres-US.pdf` | baut Dokumentation im PDF-Format mit US-Letter-Seiten |
| `doc/src/sgml/postgres.html` | baut Dokumentation im einseitigen HTML-Format |
| `alldocs` | baut Dokumentation in allen unterstützten Formaten |

#### 17.4.4.4. Installations-Targets

| Target | Bedeutung |
| --- | --- |
| `install` | installiert PostgreSQL ohne Dokumentation |
| `install-docs` | installiert Dokumentation im mehrseitigen HTML- und Manpage-Format |
| `install-html` | installiert Dokumentation im mehrseitigen HTML-Format |
| `install-man` | installiert Dokumentation im Manpage-Format |
| `install-quiet` | wie `install`, aber installierte Dateien werden nicht angezeigt |
| `install-world` | installiert PostgreSQL einschließlich mehrseitiger HTML- und Manpage-Dokumentation |
| `uninstall` | entfernt installierte Dateien |

#### 17.4.4.5. Weitere Targets

| Target | Bedeutung |
| --- | --- |
| `clean` | entfernt alle Build-Produkte |
| `test` | führt alle aktivierten Tests aus, einschließlich `contrib` |
| `world` | baut alles, einschließlich Dokumentation |
| `help` | listet wichtige Targets auf |

## 17.5. Einrichtung nach der Installation

### 17.5.1. Shared Libraries

Auf einigen Systemen mit Shared Libraries müssen Sie dem System mitteilen, wie es die neu installierten Shared Libraries findet. Zu den Systemen, auf denen dies nicht nötig ist, gehören FreeBSD, Linux, NetBSD, OpenBSD und Solaris.

Die Methode zum Setzen des Suchpfads für Shared Libraries unterscheidet sich je nach Plattform. Die am weitesten verbreitete Methode ist das Setzen der Umgebungsvariable `LD_LIBRARY_PATH`. In Bourne-Shells wie `sh`, `ksh`, `bash` oder `zsh`:

```text
LD_LIBRARY_PATH=/usr/local/pgsql/lib
export LD_LIBRARY_PATH
```

oder in `csh` oder `tcsh`:

```text
setenv LD_LIBRARY_PATH /usr/local/pgsql/lib
```

Ersetzen Sie `/usr/local/pgsql/lib` durch den Wert, den Sie in Schritt 1 für `--libdir` gesetzt haben. Sie sollten diese Befehle in eine Shell-Startdatei wie `/etc/profile` oder `~/.bash_profile` eintragen. Hinweise zu Fallstricken dieser Methode finden Sie unter <http://xahlee.info/UnixResource_dir/_/ldpath.html>.

Auf manchen Systemen ist es besser, die Umgebungsvariable `LD_RUN_PATH` vor dem Bauen zu setzen.

Unter Cygwin fügen Sie das Bibliotheksverzeichnis zu `PATH` hinzu oder verschieben die `.dll`-Dateien in das `bin`-Verzeichnis.

Wenn Sie unsicher sind, lesen Sie die Manpages Ihres Systems, etwa `ld.so` oder `rld`. Wenn später eine Meldung wie die folgende erscheint, war dieser Schritt notwendig:

```text
psql: error in loading shared libraries
libpq.so.2.1: cannot open shared object file: No such file or directory
```

Kümmern Sie sich dann einfach darum.

Wenn Sie auf Linux arbeiten und Root-Zugriff haben, können Sie nach der Installation ausführen:

```text
/sbin/ldconfig /usr/local/pgsql/lib
```

oder das entsprechende Verzeichnis angeben, damit der Laufzeit-Linker die Shared Libraries schneller findet. Weitere Informationen finden Sie in der Manpage von `ldconfig`. Auf FreeBSD, NetBSD und OpenBSD lautet der Befehl stattdessen:

```text
/sbin/ldconfig -m /usr/local/pgsql/lib
```

Für andere Systeme ist kein gleichwertiger Befehl bekannt.

### 17.5.2. Umgebungsvariablen

Wenn Sie nach `/usr/local/pgsql` oder an einen anderen Ort installiert haben, der standardmäßig nicht nach Programmen durchsucht wird, sollten Sie `/usr/local/pgsql/bin` oder den mit `--bindir` gesetzten Ort zu Ihrem `PATH` hinzufügen. Streng genommen ist das nicht notwendig, macht die Verwendung von PostgreSQL aber deutlich bequemer.

Fügen Sie dazu Folgendes in Ihre Shell-Startdatei ein, zum Beispiel `~/.bash_profile` oder `/etc/profile`, wenn es für alle Benutzer gelten soll:

```text
PATH=/usr/local/pgsql/bin:$PATH
export PATH
```

Wenn Sie `csh` oder `tcsh` verwenden, nutzen Sie:

```text
set path = ( /usr/local/pgsql/bin $path )
```

Damit Ihr System die Manpage-Dokumentation findet, fügen Sie Zeilen wie die folgenden in eine Shell-Startdatei ein, sofern Sie nicht an einen standardmäßig durchsuchten Ort installiert haben:

```text
MANPATH=/usr/local/pgsql/share/man:$MANPATH
export MANPATH
```

Die Umgebungsvariablen `PGHOST` und `PGPORT` teilen Client-Anwendungen Host und Port des Datenbankservers mit und überschreiben die einkompilierten Standardwerte. Wenn Sie Client-Anwendungen entfernt ausführen, ist es praktisch, wenn jeder Datenbankbenutzer `PGHOST` setzt. Erforderlich ist dies aber nicht; die Einstellungen können den meisten Client-Programmen auch über Kommandozeilenoptionen übergeben werden.

## 17.6. Unterstützte Plattformen

Eine Plattform, also eine Kombination aus CPU-Architektur und Betriebssystem, gilt in der PostgreSQL-Entwicklergemeinschaft als unterstützt, wenn der Code Vorkehrungen enthält, auf dieser Plattform zu funktionieren, und kürzlich verifiziert wurde, dass er dort baut und seine Regressionstests besteht. Derzeit wird die meiste Plattformkompatibilität automatisch durch Testmaschinen in der PostgreSQL Build Farm geprüft: <https://buildfarm.postgresql.org/>. Wenn Sie PostgreSQL auf einer Plattform verwenden möchten, die in der Build Farm nicht vertreten ist, auf der der Code aber funktioniert oder zum Funktionieren gebracht werden kann, wird dringend empfohlen, eine Build-Farm-Maschine einzurichten, damit dauerhafte Kompatibilität gewährleistet werden kann.

Im Allgemeinen kann erwartet werden, dass PostgreSQL auf folgenden CPU-Architekturen funktioniert: x86, PowerPC, S/390, SPARC, ARM, MIPS und RISC-V, einschließlich Big-Endian-, Little-Endian-, 32-Bit- und 64-Bit-Varianten, soweit anwendbar.

PostgreSQL kann voraussichtlich auf aktuellen Versionen dieser Betriebssysteme funktionieren: Linux, Windows, FreeBSD, OpenBSD, NetBSD, DragonFlyBSD, macOS, Solaris und illumos. Andere Unix-ähnliche Systeme können ebenfalls funktionieren, werden aber derzeit nicht getestet. In den meisten Fällen funktionieren alle CPU-Architekturen, die ein bestimmtes Betriebssystem unterstützt. Prüfen Sie [Abschnitt 17.7](#177-plattformspezifische-hinweise), ob es betriebssystemspezifische Hinweise gibt, besonders bei älteren Systemen.

Wenn Sie Installationsprobleme auf einer Plattform haben, die nach aktuellen Build-Farm-Ergebnissen unterstützt wird, melden Sie diese bitte an <pgsql-bugs@lists.postgresql.org>. Wenn Sie PostgreSQL auf eine neue Plattform portieren möchten, ist <pgsql-hackers@lists.postgresql.org> der richtige Ort für die Diskussion.

Historische Versionen von PostgreSQL oder POSTGRES liefen außerdem auf CPU-Architekturen wie Alpha, Itanium, M32R, M68K, M88K, NS32K, PA-RISC, SuperH und VAX sowie auf Betriebssystemen wie 4.3BSD, AIX, BEOS, BSD/OS, DG/UX, Dynix, HP-UX, IRIX, NeXTSTEP, QNX, SCO, SINIX, Sprite, SunOS, Tru64 UNIX und ULTRIX.

## 17.7. Plattformspezifische Hinweise

Dieser Abschnitt dokumentiert zusätzliche plattformspezifische Aspekte der Installation und Einrichtung von PostgreSQL. Lesen Sie außerdem die Installationsanweisungen und insbesondere [Abschnitt 17.1](#171-anforderungen). Zur Interpretation von Regressionstestergebnissen lesen Sie auch [Kapitel 31](31_Regressionstests.md).

Für Plattformen, die hier nicht behandelt werden, sind keine plattformspezifischen Installationsprobleme bekannt.

### 17.7.1. Cygwin

PostgreSQL kann mit Cygwin gebaut werden, einer Linux-ähnlichen Umgebung für Windows. Diese Methode ist jedoch dem nativen Windows-Build unterlegen, und der Betrieb eines Servers unter Cygwin wird nicht mehr empfohlen.

Beim Bauen aus dem Quellcode gehen Sie nach der Unix-artigen Installationsprozedur vor, also `./configure`, `make` und so weiter, beachten aber folgende Cygwin-spezifische Unterschiede:

- Setzen Sie Ihren Pfad so, dass das Cygwin-`bin`-Verzeichnis vor den Windows-Hilfsprogrammen steht. Das hilft, Kompilationsprobleme zu vermeiden.
- Der Befehl `adduser` wird nicht unterstützt; verwenden Sie die passende Benutzerverwaltungsanwendung unter Windows. Andernfalls überspringen Sie diesen Schritt.
- Der Befehl `su` wird nicht unterstützt; verwenden Sie `ssh`, um `su` unter Windows zu simulieren. Andernfalls überspringen Sie diesen Schritt.
- OpenSSL wird nicht unterstützt.
- Starten Sie `cygserver` für Shared-Memory-Unterstützung. Geben Sie dazu `/usr/sbin/cygserver &` ein. Dieses Programm muss laufen, wann immer Sie den PostgreSQL-Server starten oder einen Datenbankcluster mit `initdb` initialisieren. Die Standardkonfiguration von `cygserver` muss möglicherweise geändert werden, zum Beispiel durch Erhöhen von `SEMMNS`, damit PostgreSQL nicht wegen fehlender Systemressourcen fehlschlägt.
- Der Build kann auf manchen Systemen fehlschlagen, wenn eine andere Locale als `C` verwendet wird. Setzen Sie zur Behebung vor dem Build `export LANG=C.utf8` und stellen Sie nach der Installation von PostgreSQL die vorherige Einstellung wieder her.
- Die parallelen Regressionstests, also `make check`, können aufgrund einer überlaufenden `listen()`-Backlog-Warteschlange scheinbare Testfehler erzeugen, was zu „connection refused“-Fehlern oder Hängern führt. Sie können die Zahl der Verbindungen mit der make-Variable `MAX_CONNECTIONS` begrenzen:

```text
make MAX_CONNECTIONS=5 check
```

  Auf manchen Systemen sind bis zu etwa 10 gleichzeitige Verbindungen möglich.

Es ist möglich, `cygserver` und den PostgreSQL-Server als Windows-NT-Dienste zu installieren. Informationen dazu finden Sie im README-Dokument, das im PostgreSQL-Binärpaket für Cygwin enthalten ist. Es wird im Verzeichnis `/usr/share/doc/Cygwin` installiert.

### 17.7.2. macOS

Um PostgreSQL unter macOS aus dem Quellcode zu bauen, müssen Sie Apples Kommandozeilen-Entwicklerwerkzeuge installieren:

```text
xcode-select --install
```

Dabei erscheint ein GUI-Dialogfenster zur Bestätigung. Ob Sie zusätzlich Xcode installieren möchten, bleibt Ihnen überlassen.

Auf neueren macOS-Versionen muss der Pfad zum „sysroot“ in die Include-Schalter eingebettet werden, mit denen einige System-Header-Dateien gefunden werden. Dadurch variieren die Ausgaben des `configure`-Skripts je nach SDK-Version, die während `configure` verwendet wurde. In einfachen Szenarien sollte das kein Problem sein. Wenn Sie jedoch zum Beispiel eine Erweiterung auf einer anderen Maschine bauen möchten als der, auf der der Servercode gebaut wurde, müssen Sie möglicherweise die Verwendung eines anderen Sysroot-Pfads erzwingen. Setzen Sie dazu `PG_SYSROOT`, zum Beispiel:

```text
make PG_SYSROOT=/desired/path all
```

Den passenden Pfad auf Ihrer Maschine finden Sie mit:

```text
xcrun --show-sdk-path
```

Das Bauen einer Erweiterung mit einer anderen Sysroot-Version als der, mit der der Kernserver gebaut wurde, wird nicht wirklich empfohlen. Im schlimmsten Fall kann dies zu schwer zu debuggenden ABI-Inkonsistenzen führen.

Sie können auch beim Konfigurieren einen nicht standardmäßigen Sysroot-Pfad auswählen, indem Sie `PG_SYSROOT` an `configure` übergeben:

```text
./configure ... PG_SYSROOT=/desired/path
```

Das ist hauptsächlich nützlich, wenn für eine andere macOS-Version cross-kompiliert werden soll. Es gibt keine Garantie, dass die resultierenden ausführbaren Dateien auf dem aktuellen Host laufen.

Um die `-isysroot`-Optionen vollständig zu unterdrücken, verwenden Sie:

```text
./configure ... PG_SYSROOT=none
```

Jeder nicht existierende Pfad funktioniert ebenfalls. Das kann nützlich sein, wenn Sie mit einem Nicht-Apple-Compiler bauen möchten. Beachten Sie aber, dass dieser Fall von den PostgreSQL-Entwicklern nicht getestet oder unterstützt wird.

Die macOS-Funktion System Integrity Protection (SIP) bricht `make check`, weil sie verhindert, dass die benötigte Einstellung von `DYLD_LIBRARY_PATH` an die getesteten ausführbaren Dateien weitergegeben wird. Sie können dies umgehen, indem Sie `make install` vor `make check` ausführen. Die meisten PostgreSQL-Entwickler schalten SIP allerdings einfach ab.

### 17.7.3. MinGW

PostgreSQL für Windows kann mit MinGW gebaut werden, einer Unix-ähnlichen Build-Umgebung für Windows. Es wird empfohlen, dafür MSYS2 zu verwenden, siehe <https://www.msys2.org/>, und auch alle benötigten Pakete zu installieren.

#### 17.7.3.1. Crash Dumps sammeln

Wenn PostgreSQL unter Windows abstürzt, kann es Minidumps erzeugen, die ähnlich wie Core Dumps unter Unix zur Ursachenanalyse verwendet werden können. Diese Dumps können mit den Windows Debugger Tools oder mit Visual Studio gelesen werden. Um die Erzeugung von Dumps unter Windows zu aktivieren, erstellen Sie im Datenverzeichnis des Clusters ein Unterverzeichnis namens `crashdumps`. Die Dumps werden dann mit einem eindeutigen Namen, basierend auf der Kennung des abstürzenden Prozesses und der aktuellen Absturzzeit, in dieses Verzeichnis geschrieben.

### 17.7.4. Solaris

PostgreSQL wird auf Solaris gut unterstützt. Je aktueller Ihr Betriebssystem ist, desto weniger Probleme werden auftreten.

#### 17.7.4.1. Erforderliche Werkzeuge

Sie können entweder mit GCC oder mit Suns Compiler-Suite bauen. Für bessere Codeoptimierung wird auf der SPARC-Architektur Suns Compiler dringend empfohlen. Wenn Sie Suns Compiler verwenden, achten Sie darauf, nicht `/usr/ucb/cc` auszuwählen; verwenden Sie `/opt/SUNWspro/bin/cc`.

Sun Studio kann von <https://www.oracle.com/technetwork/server-storage/solarisstudio/downloads/> heruntergeladen werden. Viele GNU-Werkzeuge sind in Solaris 10 integriert oder auf der Solaris Companion CD vorhanden. Wenn Sie Pakete für ältere Solaris-Versionen benötigen, finden Sie diese Werkzeuge unter <http://www.sunfreeware.com>. Wenn Sie Quellen bevorzugen, sehen Sie unter <https://www.gnu.org/prep/ftp> nach.

#### 17.7.4.2. configure beschwert sich über ein fehlgeschlagenes Testprogramm

Wenn `configure` sich über ein fehlgeschlagenes Testprogramm beschwert, kann der Laufzeit-Linker wahrscheinlich eine Bibliothek nicht finden, zum Beispiel `libz`, `libreadline` oder eine andere nicht standardmäßige Bibliothek wie `libssl`. Um auf den richtigen Ort zu verweisen, setzen Sie die Umgebungsvariable `LDFLAGS` auf der `configure`-Kommandozeile:

```text
configure ... LDFLAGS="-R /usr/sfw/lib:/opt/sfw/lib:/usr/local/lib"
```

Weitere Informationen finden Sie in der Manpage von `ld`.

#### 17.7.4.3. Für optimale Performance kompilieren

Auf der SPARC-Architektur wird Sun Studio zum Kompilieren dringend empfohlen. Verwenden Sie die Optimierungsoption `-xO5`, um deutlich schnellere Binärdateien zu erzeugen. Verwenden Sie keine Flags, die das Verhalten von Gleitkommaoperationen und `errno`-Verarbeitung ändern, zum Beispiel `-fast`.

Wenn Sie keinen Grund haben, auf SPARC 64-Bit-Binärdateien zu verwenden, bevorzugen Sie die 32-Bit-Version. 64-Bit-Operationen sind langsamer, und 64-Bit-Binärdateien sind langsamer als die 32-Bit-Varianten. Andererseits ist 32-Bit-Code auf der AMD64-CPU-Familie nicht nativ, sodass 32-Bit-Code auf dieser CPU-Familie deutlich langsamer ist.

#### 17.7.4.4. DTrace zum Tracing von PostgreSQL verwenden

Ja, DTrace kann verwendet werden. Weitere Informationen finden Sie in [Abschnitt 27.5](27_Datenbankaktivität_überwachen.md#275-dynamisches-tracing).

Wenn das Linken der ausführbaren Datei `postgres` mit einer Fehlermeldung wie dieser abbricht:

```text
Undefined                       first referenced
 symbol                             in file
AbortTransaction                    utils/probes.o
CommitTransaction                   utils/probes.o
ld: fatal: Symbol referencing errors. No output written to postgres
collect2: ld returned 1 exit status
make: *** [postgres] Error 1
```

dann ist Ihre DTrace-Installation zu alt, um Probes in statischen Funktionen zu verarbeiten. Sie benötigen Solaris 10u4 oder neuer, um DTrace zu verwenden.

### 17.7.5. Visual Studio

Den meisten Benutzern wird empfohlen, die Binärdistribution für Windows herunterzuladen, die als grafisches Installationspaket von der PostgreSQL-Website verfügbar ist: <https://www.postgresql.org/download/>. Das Bauen aus dem Quellcode ist nur für Personen gedacht, die PostgreSQL selbst oder Erweiterungen entwickeln.

PostgreSQL für Windows kann mit Visual Studio über Meson gebaut werden, wie in [Abschnitt 17.4](#174-bauen-und-installieren-mit-meson) beschrieben. Der native Windows-Port erfordert eine 32- oder 64-Bit-Version von Windows 10 oder neuer.

Native Builds von `psql` unterstützen keine Kommandozeilenbearbeitung. Der Cygwin-Build unterstützt Kommandozeilenbearbeitung und sollte daher verwendet werden, wenn `psql` unter Windows interaktiv genutzt werden soll.

PostgreSQL kann mit der Visual-C++-Compiler-Suite von Microsoft gebaut werden. Diese Compiler können aus Visual Studio, Visual Studio Express oder bestimmten Versionen des Microsoft Windows SDK stammen. Wenn Sie noch keine Visual-Studio-Umgebung eingerichtet haben, ist es am einfachsten, die Compiler aus Visual Studio 2022 oder dem Windows SDK 10 zu verwenden; beide sind kostenlose Downloads von Microsoft.

Sowohl 32-Bit- als auch 64-Bit-Builds sind mit der Microsoft-Compiler-Suite möglich. 32-Bit-Builds von PostgreSQL sind mit Visual Studio 2015 bis Visual Studio 2022 sowie eigenständigen Windows-SDK-Versionen 10 und neuer möglich. 64-Bit-Builds werden mit Microsoft Windows SDK Version 10 und neuer oder Visual Studio 2015 und neuer unterstützt.

Wenn Ihre Build-Umgebung keine unterstützte Version des Microsoft Windows SDK enthält, wird empfohlen, auf die neueste Version zu aktualisieren, derzeit Version 10, verfügbar unter <https://www.microsoft.com/download>.

Sie müssen immer den Teil Windows Headers and Libraries des SDK einbeziehen. Wenn Sie ein Windows SDK einschließlich Visual-C++-Compilern installieren, benötigen Sie Visual Studio zum Bauen nicht. Beachten Sie, dass das Windows SDK seit Version 8.0a keine vollständige Kommandozeilen-Build-Umgebung mehr enthält.

#### 17.7.5.1. Anforderungen

Die folgenden zusätzlichen Produkte werden zum Bauen von PostgreSQL unter Windows benötigt.

- **Strawberry Perl**

  Strawberry Perl wird benötigt, um die Skripte zur Build-Erzeugung auszuführen. MinGW- oder Cygwin-Perl funktionieren nicht. Es muss außerdem in `PATH` vorhanden sein. Binärdateien können von <https://strawberryperl.com> heruntergeladen werden.

- **Bison und Flex**

  Binärdateien für Bison und Flex können von <https://github.com/lexxmark/winflexbison> heruntergeladen werden.

Die folgenden zusätzlichen Produkte werden nicht benötigt, um zu beginnen, sind aber erforderlich, um das vollständige Paket zu bauen.

- **Magicsplat Tcl**

  Erforderlich zum Bauen von PL/Tcl. Binärdateien können von <https://www.magicsplat.com/tcl-installer/index.html> heruntergeladen werden.

- **Diff**

  `diff` wird zum Ausführen der Regressionstests benötigt und kann von <http://gnuwin32.sourceforge.net> heruntergeladen werden.

- **Gettext**

  Gettext wird zum Bauen mit NLS-Unterstützung benötigt und kann von <http://gnuwin32.sourceforge.net> heruntergeladen werden. Beachten Sie, dass Binärdateien, Abhängigkeiten und Entwicklerdateien benötigt werden.

- **MIT Kerberos**

  Erforderlich für GSSAPI-Authentifizierungsunterstützung. MIT Kerberos kann von <https://web.mit.edu/Kerberos/dist/index.html> heruntergeladen werden.

- **libxml2 und libxslt**

  Erforderlich für XML-Unterstützung. Binärdateien können von <https://zlatkovic.com/pub/libxml> oder Quellen von <http://xmlsoft.org> heruntergeladen werden. Beachten Sie, dass `libxml2` `iconv` benötigt, das am selben Downloadort verfügbar ist.

- **LZ4**

  Erforderlich für LZ4-Kompression. Binärdateien und Quellen können von <https://github.com/lz4/lz4/releases> heruntergeladen werden.

- **Zstandard**

  Erforderlich für Zstandard-Kompression. Binärdateien und Quellen können von <https://github.com/facebook/zstd/releases> heruntergeladen werden.

- **OpenSSL**

  Erforderlich für SSL-Unterstützung. Binärdateien können von <https://slproweb.com/products/Win32OpenSSL.html> oder Quellen von <https://www.openssl.org> heruntergeladen werden.

- **ossp-uuid**

  Erforderlich für UUID-OSSP-Unterstützung, nur `contrib`. Quellen können von <http://www.ossp.org/pkg/lib/uuid/> heruntergeladen werden.

- **Python**

  Erforderlich zum Bauen von PL/Python. Binärdateien können von <https://www.python.org> heruntergeladen werden.

- **zlib**

  Erforderlich für Kompressionsunterstützung in `pg_dump` und `pg_restore`. Binärdateien können von <https://www.zlib.net> heruntergeladen werden.

#### 17.7.5.2. Besondere Überlegungen für 64-Bit-Windows

PostgreSQL wird unter 64-Bit-Windows nur für die x64-Architektur gebaut.

Das Mischen von 32-Bit- und 64-Bit-Versionen im selben Build-Baum wird nicht unterstützt. Das Build-System erkennt automatisch, ob es in einer 32- oder 64-Bit-Umgebung läuft, und baut PostgreSQL entsprechend. Deshalb ist es wichtig, vor dem Bauen die richtige Eingabeaufforderung zu starten.

Wenn Sie eine serverseitige Drittanbieterbibliothek wie Python oder OpenSSL verwenden möchten, muss diese Bibliothek ebenfalls 64-Bit sein. Das Laden einer 32-Bit-Bibliothek in einen 64-Bit-Server wird nicht unterstützt. Mehrere von PostgreSQL unterstützte Drittanbieterbibliotheken sind möglicherweise nur in 32-Bit-Versionen verfügbar und können dann nicht mit 64-Bit-PostgreSQL verwendet werden.

#### 17.7.5.3. Crash Dumps sammeln

Wenn PostgreSQL unter Windows abstürzt, kann es Minidumps erzeugen, die ähnlich wie Core Dumps unter Unix zur Ursachenanalyse verwendet werden können. Diese Dumps können mit den Windows Debugger Tools oder mit Visual Studio gelesen werden. Um die Erzeugung von Dumps unter Windows zu aktivieren, erstellen Sie im Datenverzeichnis des Clusters ein Unterverzeichnis namens `crashdumps`. Die Dumps werden dann mit einem eindeutigen Namen, basierend auf der Kennung des abstürzenden Prozesses und der aktuellen Absturzzeit, in dieses Verzeichnis geschrieben.
