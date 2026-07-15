# Anhang J. Dokumentation

PostgreSQL stellt vier primäre Dokumentationsformate bereit:

- Klartext für Informationen vor der Installation
- HTML zum Online-Lesen und als Referenz
- PDF zum Drucken
- Manpages für schnelle Nachschlagezwecke

Zusätzlich finden sich im PostgreSQL-Quellbaum zahlreiche Klartextdateien namens `README`, die verschiedene Implementierungsfragen dokumentieren.

HTML-Dokumentation und Manpages sind Teil einer Standarddistribution und werden standardmäßig installiert. Die Dokumentation im PDF-Format steht separat zum Herunterladen bereit.

### J.1. DocBook

Die Dokumentationsquellen sind in DocBook geschrieben, einer in XML definierten Auszeichnungssprache. Im Folgenden werden die Begriffe DocBook und XML beide verwendet; technisch sind sie jedoch nicht austauschbar.

DocBook erlaubt es Autorinnen und Autoren, Struktur und Inhalt eines technischen Dokuments festzulegen, ohne sich um Präsentationsdetails kümmern zu müssen. Ein Dokumentstil definiert, wie dieser Inhalt in eine der endgültigen Formen gerendert wird. DocBook wird von der OASIS-Gruppe gepflegt:

<https://www.oasis-open.org>

Die offizielle DocBook-Site enthält gute Einführungs- und Referenzdokumentation:

<https://www.oasis-open.org/docbook/>

Auch das FreeBSD Documentation Project verwendet DocBook und bietet nützliche Informationen, darunter Stilrichtlinien, die erwägenswert sein können:

<https://www.freebsd.org/docproj/>

### J.2. Werkzeugsätze

Die folgenden Werkzeuge werden zum Verarbeiten der Dokumentation verwendet. Einige davon können, wie angegeben, optional sein.

- DocBook DTD
  - Dies ist die Definition von DocBook selbst. Derzeit wird Version 4.5 verwendet; frühere oder spätere Versionen können nicht verwendet werden. Benötigt wird die XML-Variante der DocBook-DTD, nicht die SGML-Variante.
  - <https://www.oasis-open.org/docbook/>

- DocBook XSL Stylesheets
  - Diese enthalten die Verarbeitungsanweisungen zur Umwandlung der DocBook-Quellen in andere Formate, etwa HTML.
  - Die derzeit mindestens erforderliche Version ist 1.77.0. Für beste Ergebnisse wird jedoch empfohlen, die neueste verfügbare Version zu verwenden.
  - <https://github.com/docbook/wiki/wiki/DocBookXslStylesheets>

- Libxml2 für `xmllint`
  - Diese Bibliothek und das darin enthaltene Werkzeug `xmllint` werden zur Verarbeitung von XML verwendet. Viele Entwicklerinnen und Entwickler haben Libxml2 bereits installiert, da es auch beim Bau des PostgreSQL-Codes verwendet wird. Beachten Sie jedoch, dass `xmllint` möglicherweise aus einem separaten Unterpaket installiert werden muss.
  - <http://xmlsoft.org/>

- Libxslt für `xsltproc`
  - `xsltproc` ist ein XSLT-Prozessor, also ein Programm zur Umwandlung von XML in andere Formate mithilfe von XSLT-Stylesheets.
  - <http://xmlsoft.org/XSLT/>

- FOP
  - Dies ist ein Programm, das unter anderem XML in PDF umwandeln kann. Es wird nur benötigt, wenn Sie die Dokumentation im PDF-Format bauen möchten.
  - <https://xmlgraphics.apache.org/fop/>

Für verschiedene Installationsmethoden der Werkzeuge, die zur Verarbeitung der Dokumentation benötigt werden, gibt es dokumentierte Erfahrungen. Diese werden im Folgenden beschrieben. Es kann weitere Paketdistributionen für diese Werkzeuge geben. Melden Sie den Paketstatus bitte an die Dokumentations-Mailingliste, damit diese Informationen hier ergänzt werden können.

#### J.2.1. Installation auf Fedora, RHEL und abgeleiteten Distributionen

Installieren Sie die erforderlichen Pakete mit:

```sh
yum install docbook-dtds docbook-style-xsl libxslt fop
```

#### J.2.2. Installation auf FreeBSD

Installieren Sie die erforderlichen Pakete mit `pkg`:

```sh
pkg install docbook-xml docbook-xsl libxslt fop
```

Beim Bau der Dokumentation aus dem Verzeichnis `doc` müssen Sie `gmake` verwenden, da das bereitgestellte Makefile nicht für FreeBSDs `make` geeignet ist.

#### J.2.3. Debian-Pakete

Für Debian GNU/Linux gibt es einen vollständigen Satz von Paketen für die Dokumentationswerkzeuge. Installieren Sie sie einfach mit:

```sh
apt-get install docbook-xml docbook-xsl libxml2-utils xsltproc fop
```

#### J.2.4. macOS

Wenn Sie MacPorts verwenden, richten Sie die benötigte Umgebung so ein:

```sh
sudo port install docbook-xml docbook-xsl-nons libxslt fop
```

Wenn Sie Homebrew verwenden, nutzen Sie:

```sh
brew install docbook docbook-xsl libxslt fop
```

Die von Homebrew bereitgestellten Programme erfordern, dass die folgende Umgebungsvariable gesetzt ist. Auf Intel-basierten Rechnern verwenden Sie:

```sh
export XML_CATALOG_FILES=/usr/local/etc/xml/catalog
```

Auf Apple-Silicon-Rechnern verwenden Sie:

```sh
export XML_CATALOG_FILES=/opt/homebrew/etc/xml/catalog
```

Ohne diese Einstellung gibt `xsltproc` Fehler wie diese aus:

```text
I/O error : Attempt to load network entity http://www.oasis-open.org/docbook/xml/4.5/docbookx.dtd
postgres.sgml:21: warning: failed to load external entity "http://www.oasis-open.org/docbook/xml/4.5/docbookx.dtd"
...
```

Es ist möglich, die von Apple bereitgestellten Versionen von `xmllint` und `xsltproc` anstelle der Versionen aus MacPorts oder Homebrew zu verwenden. Sie müssen dann jedoch weiterhin die DocBook-DTD und die Stylesheets installieren und eine Katalogdatei einrichten, die darauf verweist.

#### J.2.5. Erkennung durch `configure`

Bevor Sie die Dokumentation bauen können, müssen Sie das Skript `configure` ausführen, wie Sie es auch beim Bau der PostgreSQL-Programme selbst tun würden. Prüfen Sie die Ausgabe gegen Ende des Laufs; sie sollte etwa so aussehen:

```text
checking for xmllint... xmllint
checking for xsltproc... xsltproc
checking for fop... fop
checking for dbtoepub... dbtoepub
```

Wenn `xmllint` oder `xsltproc` nicht gefunden wird, können Sie keine Dokumentation bauen. `fop` wird nur benötigt, um die Dokumentation im PDF-Format zu bauen. `dbtoepub` wird nur benötigt, um die Dokumentation im EPUB-Format zu bauen.

Bei Bedarf können Sie `configure` mitteilen, wo diese Programme zu finden sind, zum Beispiel:

```sh
./configure ... XMLLINT=/opt/local/bin/xmllint ...
```

Wenn Sie PostgreSQL lieber mit Meson bauen, führen Sie stattdessen `meson setup` aus, wie in [Abschnitt 17.4](17_Installation_aus_dem_Quellcode.md#174-bauen-und-installieren-mit-meson) beschrieben, und lesen Sie anschließend [Abschnitt J.4](#j4-die-dokumentation-mit-meson-bauen).

### J.3. Die Dokumentation mit Make bauen

Wenn alles eingerichtet ist, wechseln Sie in das Verzeichnis `doc/src/sgml` und führen einen der in den folgenden Unterabschnitten beschriebenen Befehle aus, um die Dokumentation zu bauen. Denken Sie daran, GNU `make` zu verwenden.

#### J.3.1. HTML

Um die HTML-Version der Dokumentation zu bauen:

```sh
doc/src/sgml$ make html
```

Dies ist auch das Standardziel. Die Ausgabe erscheint im Unterverzeichnis `html`.

Um HTML-Dokumentation mit dem auf postgresql.org verwendeten Stylesheet statt mit dem einfachen Standardstil zu erzeugen, verwenden Sie:

```sh
doc/src/sgml$ make STYLE=website html
```

Wenn die Option `STYLE=website` verwendet wird, enthalten die erzeugten HTML-Dateien Verweise auf auf postgresql.org gehostete Stylesheets und benötigen Netzwerkzugriff zum Anzeigen.

<https://www.postgresql.org/docs/current/>

#### J.3.2. Manpages

Die DocBook-XSL-Stylesheets werden verwendet, um DocBook-`refentry`-Seiten in eine für Manpages geeignete *roff-Ausgabe umzuwandeln. Um die Manpages zu erzeugen, verwenden Sie:

```sh
doc/src/sgml$ make man
```

#### J.3.3. PDF

Um mit FOP eine PDF-Ausgabe der Dokumentation zu erzeugen, verwenden Sie je nach gewünschtem Papierformat einen der folgenden Befehle:

- Für A4:

  ```sh
  doc/src/sgml$ make postgres-A4.pdf
  ```

- Für US Letter:

  ```sh
  doc/src/sgml$ make postgres-US.pdf
  ```

Da die PostgreSQL-Dokumentation ziemlich groß ist, benötigt FOP eine erhebliche Menge Arbeitsspeicher. Auf manchen Systemen schlägt der Bau deshalb mit einer speicherbezogenen Fehlermeldung fehl. Dies lässt sich normalerweise beheben, indem Java-Heap-Einstellungen in der Konfigurationsdatei `~/.foprc` gesetzt werden, zum Beispiel:

```sh
# FOP binary distribution
FOP_OPTS='-Xmx1500m'
# Debian
JAVA_ARGS='-Xmx1500m'
# Red Hat
ADDITIONAL_FLAGS='-Xmx1500m'
```

Es gibt eine Mindestmenge an benötigtem Speicher; in gewissem Umfang scheint zusätzlicher Speicher den Bau etwas zu beschleunigen. Auf Systemen mit sehr wenig Speicher, weniger als 1 GB, ist der Bau entweder wegen Swap-Aktivität sehr langsam oder funktioniert gar nicht.

In der Standardkonfiguration gibt FOP für jede Seite eine `INFO`-Meldung aus. Die Protokollstufe kann über `~/.foprc` geändert werden:

```sh
LOGCHOICE=-Dorg.apache.commons.logging.Log=org.apache.commons.logging.impl.SimpleLog
LOGLEVEL=-Dorg.apache.commons.logging.simplelog.defaultlog=WARN
```

Andere XSL-FO-Prozessoren können ebenfalls manuell verwendet werden, der automatisierte Bauprozess unterstützt jedoch nur FOP.

#### J.3.4. Syntaxprüfung

Der Bau der Dokumentation kann sehr lange dauern. Es gibt jedoch eine Methode, nur die korrekte Syntax der Dokumentationsdateien zu prüfen; dies dauert nur wenige Sekunden:

```sh
doc/src/sgml$ make check
```

### J.4. Die Dokumentation mit Meson bauen

Um die Dokumentation mit Meson zu bauen, wechseln Sie vor dem Ausführen eines der folgenden Befehle in das Build-Verzeichnis oder ergänzen Sie `-C build`.

Um nur die HTML-Version der Dokumentation zu bauen:

```sh
build$ ninja html
```

Eine Liste weiterer Dokumentationsziele finden Sie in [Abschnitt 17.4.4.3](17_Installation_aus_dem_Quellcode.md#17443-dokumentationstargets). Die Ausgabe erscheint im Unterverzeichnis `build/doc/src/sgml`.

### J.5. Dokumentation schreiben

Die Dokumentationsquellen lassen sich am bequemsten mit einem Editor bearbeiten, der einen Modus für XML-Dokumente besitzt, und noch besser mit einem Editor, der XML-Schema-Sprachen kennt und deshalb speziell DocBook-Syntax versteht.

Aus historischen Gründen tragen die Quelldateien der Dokumentation die Endung `.sgml`, obwohl sie heute XML-Dateien sind. Möglicherweise müssen Sie daher Ihre Editor-Konfiguration anpassen, damit der richtige Modus verwendet wird.

#### J.5.1. Emacs

Der mit Emacs ausgelieferte nXML Mode ist der verbreitetste Modus zum Bearbeiten von XML-Dokumenten mit Emacs. Er erlaubt es, Tags einzufügen und die Konsistenz des Markups zu prüfen, und unterstützt DocBook von Haus aus. Ausführliche Dokumentation finden Sie im nXML-Handbuch:

<https://www.gnu.org/software/emacs/manual/html_mono/nxml-mode.html>

`src/tools/editors/emacs.samples` enthält empfohlene Einstellungen für diesen Modus.

### J.6. Styleguide

#### J.6.1. Referenzseiten

Referenzseiten sollten einem Standardaufbau folgen. Dadurch finden Benutzerinnen und Benutzer die gewünschten Informationen schneller, und Autorinnen und Autoren werden dazu angehalten, alle relevanten Aspekte eines Befehls zu dokumentieren. Konsistenz ist nicht nur zwischen PostgreSQL-Referenzseiten erwünscht, sondern auch gegenüber Referenzseiten des Betriebssystems und anderer Pakete. Daraus ergeben sich die folgenden Richtlinien. Sie stimmen größtenteils mit ähnlichen Richtlinien verschiedener Betriebssysteme überein.

Referenzseiten, die ausführbare Befehle beschreiben, sollten die folgenden Abschnitte in dieser Reihenfolge enthalten. Nicht zutreffende Abschnitte können weggelassen werden. Zusätzliche Abschnitte auf oberster Ebene sollten nur in besonderen Fällen verwendet werden; oft gehört solche Information in den Abschnitt „Verwendung“.

- Name
  - Dieser Abschnitt wird automatisch erzeugt. Er enthält den Befehlsnamen und eine halbsatzartige Zusammenfassung der Funktion.

- Synopsis
  - Dieser Abschnitt enthält das Syntaxdiagramm des Befehls. Die Synopsis sollte normalerweise nicht jede Befehlszeilenoption aufführen; das geschieht weiter unten. Stattdessen sollten die Hauptbestandteile der Befehlszeile aufgeführt werden, etwa wo Eingabe- und Ausgabedateien stehen.

- Beschreibung
  - Mehrere Absätze, die erklären, was der Befehl tut.

- Optionen
  - Eine Liste, die jede Befehlszeilenoption beschreibt. Wenn es viele Optionen gibt, können Unterabschnitte verwendet werden.

- Exit-Status
  - Wenn das Programm `0` für Erfolg und einen Wert ungleich null für Fehler verwendet, muss dies nicht dokumentiert werden. Wenn die verschiedenen Nichtnull-Exitcodes eine Bedeutung haben, werden sie hier aufgelistet.

- Verwendung
  - Beschreibt eine Untersprache oder Laufzeitschnittstelle des Programms. Wenn das Programm nicht interaktiv ist, kann dieser Abschnitt meist weggelassen werden. Andernfalls ist er ein Sammelabschnitt für Laufzeitfunktionen. Verwenden Sie Unterabschnitte, wenn das sinnvoll ist.

- Umgebung
  - Listet alle Umgebungsvariablen auf, die das Programm verwenden kann. Versuchen Sie vollständig zu sein; selbst scheinbar triviale Variablen wie `SHELL` können für Benutzerinnen und Benutzer interessant sein.

- Dateien
  - Listet Dateien auf, auf die das Programm implizit zugreifen kann. Eingabe- und Ausgabedateien, die auf der Befehlszeile angegeben werden, werden hier nicht aufgeführt; Konfigurationsdateien und Ähnliches dagegen schon.

- Diagnose
  - Erklärt ungewöhnliche Ausgaben, die das Programm erzeugen kann. Listen Sie nicht jede mögliche Fehlermeldung auf. Das ist viel Arbeit und in der Praxis wenig nützlich. Wenn Fehlermeldungen jedoch ein Standardformat haben, das Benutzerinnen und Benutzer parsen können, gehört die Erklärung hierher.

- Hinweise
  - Alles, was sonst nirgends passt, insbesondere Fehler, Implementierungsschwächen, Sicherheitsaspekte und Kompatibilitätsfragen.

- Beispiele
  - Beispiele.

- Geschichte
  - Wenn es wichtige Meilensteine in der Geschichte des Programms gab, können sie hier aufgeführt werden. Normalerweise kann dieser Abschnitt weggelassen werden.

- Autor
  - Autor; wird nur im Bereich `contrib` verwendet.

- Siehe auch
  - Querverweise in folgender Reihenfolge: andere PostgreSQL-Befehlsreferenzseiten, PostgreSQL-SQL-Befehlsreferenzseiten, Verweise auf PostgreSQL-Handbücher, andere Referenzseiten, etwa Betriebssystem- oder Paketdokumentation, und sonstige Dokumentation. Elemente innerhalb derselben Gruppe werden alphabetisch sortiert.

Referenzseiten, die SQL-Befehle beschreiben, sollten die folgenden Abschnitte enthalten: Name, Synopsis, Beschreibung, Parameter, Ausgaben, Hinweise, Beispiele, Kompatibilität, Geschichte, Siehe auch.

Der Abschnitt „Parameter“ entspricht dem Abschnitt „Optionen“, lässt aber mehr Freiheit dabei, welche Klauseln des Befehls aufgeführt werden. Der Abschnitt „Ausgaben“ wird nur benötigt, wenn der Befehl etwas anderes als ein Standard-Command-Completion-Tag zurückgibt. Der Abschnitt „Kompatibilität“ sollte erklären, in welchem Umfang der Befehl dem SQL-Standard oder den SQL-Standards entspricht oder mit welchem anderen Datenbanksystem er kompatibel ist. Der Abschnitt „Siehe auch“ von SQL-Befehlen sollte SQL-Befehle vor Querverweisen auf Programme aufführen.
