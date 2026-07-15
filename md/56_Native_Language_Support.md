# 56. Unterstützung natürlicher Sprachen

## 56.1. Für Übersetzer

PostgreSQL-Programme (Server und Client) können ihre Meldungen in Ihrer bevorzugten Sprache ausgeben, wenn die Meldungen übersetzt wurden. Das Erstellen und Pflegen übersetzter Meldungssätze braucht die Hilfe von Menschen, die ihre eigene Sprache gut beherrschen und zum PostgreSQL-Projekt beitragen möchten. Dafür müssen Sie überhaupt kein Programmierer sein. Dieser Abschnitt erklärt, wie Sie helfen können.

### 56.1.1. Voraussetzungen

Wir beurteilen hier nicht Ihre Sprachkenntnisse; in diesem Abschnitt geht es um Softwarewerkzeuge. Theoretisch benötigen Sie nur einen Texteditor. Das gilt aber nur für den unwahrscheinlichen Fall, dass Sie Ihre übersetzten Meldungen nicht ausprobieren möchten. Wenn Sie Ihren Quellbaum konfigurieren, verwenden Sie unbedingt die Option `--enable-nls`. Dadurch wird auch auf die Bibliothek `libintl` und das Programm `msgfmt` geprüft, die alle Endbenutzer ohnehin benötigen. Um Ihre Arbeit auszuprobieren, folgen Sie den zutreffenden Teilen der Installationsanweisungen.

Wenn Sie eine neue Übersetzungsarbeit beginnen oder eine Zusammenführung eines Meldungskatalogs durchführen möchten (später beschrieben), benötigen Sie die Programme `xgettext` beziehungsweise `msgmerge` in einer GNU-kompatiblen Implementierung. Später soll es möglichst so eingerichtet werden, dass Sie bei Verwendung einer paketierten Quellverteilung `xgettext` nicht benötigen. Wenn Sie aus Git arbeiten, benötigen Sie es weiterhin. Derzeit wird GNU Gettext 0.10.36 oder später empfohlen.

Ihre lokale Gettext-Implementierung sollte mit eigener Dokumentation geliefert werden. Ein Teil davon wird wahrscheinlich im Folgenden wiederholt, aber für zusätzliche Details sollten Sie dort nachsehen.

### 56.1.2. Konzepte

Die Paare aus ursprünglichen (englischen) Meldungen und ihren (möglicherweise) übersetzten Entsprechungen werden in Meldungskatalogen aufbewahrt, je einer für jedes Programm (obwohl verwandte Programme einen Meldungskatalog teilen können) und für jede Zielsprache. Für Meldungskataloge gibt es zwei Dateiformate: Das erste ist die PO-Datei (für Portable Object), eine reine Textdatei mit spezieller Syntax, die Übersetzer bearbeiten. Das zweite ist die MO-Datei (für Machine Object), eine Binärdatei, die aus der jeweiligen PO-Datei erzeugt und verwendet wird, während das internationalisierte Programm läuft. Übersetzer arbeiten nicht mit MO-Dateien; tatsächlich tut das kaum jemand.

Die Dateiendung eines Meldungskatalogs ist wenig überraschend entweder `.po` oder `.mo`. Der Basisname ist je nach Situation entweder der Name des zugehörigen Programms oder die Sprache, für die die Datei gedacht ist. Das ist etwas verwirrend. Beispiele sind `psql.po` (PO-Datei für `psql`) oder `fr.mo` (MO-Datei auf Französisch).

Das Dateiformat der PO-Dateien wird hier veranschaulicht:

```text
# comment

msgid "original string"
msgstr "translated string"

msgid "more original"
msgstr "another translated"
"string can be broken up like this"

...
```

Die `msgid`-Zeilen werden aus dem Programmquellcode extrahiert. Sie müssen das nicht, aber es ist der häufigste Weg. Die `msgstr`-Zeilen sind anfangs leer und werden vom Übersetzer mit sinnvollen Strings gefüllt. Die Strings können Escape-Zeichen im C-Stil enthalten und wie gezeigt über mehrere Zeilen fortgesetzt werden. Die nächste Zeile muss am Zeilenanfang beginnen.

Das Zeichen `#` leitet einen Kommentar ein. Wenn unmittelbar nach dem Zeichen `#` Leerraum folgt, ist dies ein vom Übersetzer gepflegter Kommentar. Es kann auch automatische Kommentare geben, bei denen unmittelbar nach dem `#` ein Nicht-Leerraumzeichen folgt. Diese werden von den verschiedenen Werkzeugen gepflegt, die mit den PO-Dateien arbeiten, und sollen den Übersetzer unterstützen.

```text
#. automatic comment
#: filename.c:1023
#, flags, flags
```

Kommentare der Form `#.` werden aus der Quelldatei extrahiert, in der die Meldung verwendet wird. Möglicherweise hat der Programmierer Informationen für den Übersetzer eingefügt, etwa zur erwarteten Ausrichtung. Die Kommentare `#:` geben die genauen Stellen an, an denen die Meldung im Quellcode verwendet wird. Der Übersetzer muss den Programmquellcode nicht ansehen, kann dies aber tun, wenn Zweifel an der richtigen Übersetzung bestehen. Die Kommentare `#,` enthalten Flags, die die Meldung auf irgendeine Weise beschreiben. Derzeit gibt es zwei Flags: `fuzzy` wird gesetzt, wenn die Meldung wegen Änderungen im Programmquellcode möglicherweise veraltet ist. Der Übersetzer kann dies dann prüfen und das `fuzzy`-Flag gegebenenfalls entfernen. Beachten Sie, dass Fuzzy-Meldungen dem Endbenutzer nicht verfügbar gemacht werden. Das andere Flag ist `c-format`; es zeigt an, dass die Meldung eine Formatvorlage im `printf`-Stil ist. Das bedeutet, dass die Übersetzung ebenfalls ein Formatstring mit derselben Anzahl und Art von Platzhaltern sein sollte. Es gibt Werkzeuge, die dies anhand des Flags `c-format` prüfen können.

### 56.1.3. Meldungskataloge erstellen und pflegen

Wie erstellt man also einen »leeren« Meldungskatalog? Wechseln Sie zuerst in das Verzeichnis, das das Programm enthält, dessen Meldungen Sie übersetzen möchten. Wenn es dort eine Datei `nls.mk` gibt, wurde dieses Programm für Übersetzung vorbereitet.

Wenn bereits einige `.po`-Dateien vorhanden sind, hat jemand schon Übersetzungsarbeit geleistet. Die Dateien heißen `language.po`, wobei `language` der zweibuchstabige Sprachcode nach ISO 639-1 (in Kleinbuchstaben) ist, zum Beispiel `fr.po` für Französisch. Wenn es wirklich mehr als eine Übersetzungsarbeit pro Sprache geben muss, können die Dateien auch `language_region.po` heißen, wobei `region` der zweibuchstabige Ländercode nach ISO 3166-1 (in Großbuchstaben) ist, zum Beispiel `pt_BR.po` für Portugiesisch in Brasilien. Wenn Sie die gewünschte Sprache finden, können Sie einfach an dieser Datei weiterarbeiten.

Wenn Sie eine neue Übersetzungsarbeit beginnen müssen, führen Sie zuerst den Befehl aus:

```text
make init-po
```

Dies erzeugt eine Datei `progname.pot`. `.pot` unterscheidet sie von PO-Dateien, die »in Produktion« sind; das `T` steht für »template«. Kopieren Sie diese Datei nach `language.po` und bearbeiten Sie sie. Damit bekannt wird, dass die neue Sprache verfügbar ist, bearbeiten Sie außerdem die Datei `po/LINGUAS` und fügen den Sprachcode (oder Sprach- und Ländercode) neben den bereits aufgeführten Sprachen hinzu, zum Beispiel:

```text
de fr
```

Natürlich können dort auch andere Sprachen stehen.

Wenn sich das zugrunde liegende Programm oder die Bibliothek ändert, können Programmierer Meldungen ändern oder hinzufügen. In diesem Fall müssen Sie nicht von vorne beginnen. Führen Sie stattdessen den Befehl aus:

```text
make update-po
```

Dadurch wird eine neue leere Meldungskatalogdatei erzeugt (die POT-Datei, mit der Sie begonnen haben) und mit den vorhandenen PO-Dateien zusammengeführt. Wenn der Merge-Algorithmus bei einer bestimmten Meldung unsicher ist, markiert er sie wie oben erklärt als `fuzzy`. Die neue PO-Datei wird mit der Endung `.po.new` gespeichert.

<https://www.loc.gov/standards/iso639-2/php/English_list.php>

<https://www.iso.org/iso-3166-country-codes.html>

### 56.1.4. Die PO-Dateien bearbeiten

Die PO-Dateien können mit einem normalen Texteditor bearbeitet werden. Es gibt außerdem mehrere spezialisierte Editoren für PO-Dateien, die den Prozess mit übersetzungsspezifischen Funktionen unterstützen können. Es gibt wenig überraschend einen PO-Modus für Emacs, der recht nützlich sein kann.

Der Übersetzer sollte nur den Bereich zwischen den Anführungszeichen nach der Direktive `msgstr` ändern, Kommentare hinzufügen und das Flag `fuzzy` verändern.

Die PO-Dateien müssen nicht vollständig ausgefüllt sein. Die Software fällt automatisch auf den Originalstring zurück, wenn keine Übersetzung oder eine leere Übersetzung verfügbar ist. Es ist kein Problem, unvollständige Übersetzungen zur Aufnahme in den Quellbaum einzureichen; dadurch können andere Ihre Arbeit aufgreifen. Sie werden jedoch ermutigt, nach einem Merge vorrangig Fuzzy-Einträge zu entfernen. Denken Sie daran, dass Fuzzy-Einträge nicht installiert werden; sie dienen nur als Referenz dafür, was die richtige Übersetzung sein könnte.

Beim Bearbeiten der Übersetzungen sollten Sie Folgendes beachten:

- Stellen Sie sicher, dass die Übersetzung ebenfalls mit einem Newline endet, wenn das Original mit einem Newline endet. Entsprechendes gilt für Tabulatoren usw.

- Wenn das Original ein `printf`-Formatstring ist, muss auch die Übersetzung einer sein. Die Übersetzung muss außerdem dieselben Formatangaben in derselben Reihenfolge haben. Manchmal machen die natürlichen Regeln der Sprache dies unmöglich oder zumindest umständlich. In diesem Fall können Sie die Formatangaben so verändern:

```text
msgstr "Die Datei %2$s hat %1$u Zeichen."
```

Dann verwendet der erste Platzhalter tatsächlich das zweite Argument aus der Liste. `digits$` muss unmittelbar auf `%` folgen, vor allen anderen Formatmanipulatoren. Dieses Merkmal existiert tatsächlich in der `printf`-Funktionsfamilie. Vielleicht haben Sie bisher nichts davon gehört, weil es außerhalb der Meldungsinternationalisierung wenig Verwendung dafür gibt.

- Wenn der Originalstring einen sprachlichen Fehler enthält, melden Sie ihn oder beheben Sie ihn selbst im Programmquellcode und übersetzen Sie normal. Der korrigierte String kann eingemischt werden, wenn die Programmquellen aktualisiert wurden. Wenn der Originalstring einen sachlichen Fehler enthält, melden Sie ihn oder beheben Sie ihn selbst und übersetzen Sie ihn nicht. Stattdessen können Sie den String in der PO-Datei mit einem Kommentar markieren.

  - Bewahren Sie Stil und Ton des Originalstrings. Insbesondere sollten Meldungen, die keine Sätze sind (`cannot open file %s`), wahrscheinlich nicht mit einem Großbuchstaben beginnen, wenn Ihre Sprache Groß-/Kleinschreibung unterscheidet, und nicht mit einem Punkt enden, wenn Ihre Sprache Satzzeichen verwendet. Es kann helfen, [Abschnitt 55.3](55_PostgreSQL_Coding_Konventionen.md#553-stilleitfaden-für-fehlermeldungen) zu lesen.

- Wenn Sie nicht wissen, was eine Meldung bedeutet, oder wenn sie mehrdeutig ist, fragen Sie auf der Entwickler-Mailingliste. Es besteht die Chance, dass auch englischsprachige Endbenutzer sie nicht verstehen oder mehrdeutig finden; daher ist es am besten, die Meldung zu verbessern.

## 56.2. Für Programmierer

### 56.2.1. Mechanik

Dieser Abschnitt beschreibt, wie Native Language Support in einem Programm oder einer Bibliothek implementiert wird, die Teil der PostgreSQL-Distribution ist. Derzeit gilt er nur für C-Programme.

Native-Language-Support zu einem Programm hinzufügen:

1. Fügen Sie diesen Code in die Startsequenz des Programms ein:

```c
#ifdef ENABLE_NLS
#include <locale.h>
#endif

...

#ifdef ENABLE_NLS
setlocale(LC_ALL, "");
bindtextdomain("progname", LOCALEDIR);
textdomain("progname");
#endif
```

`progname` kann tatsächlich frei gewählt werden.

2. Überall dort, wo eine Meldung gefunden wird, die für eine Übersetzung infrage kommt, muss ein Aufruf von `gettext()` eingefügt werden. Zum Beispiel:

```c
fprintf(stderr, "panic level %d\n", lvl);
```

würde geändert zu:

```c
fprintf(stderr, gettext("panic level %d\n"), lvl);
```

`gettext` ist als No-op definiert, wenn NLS-Unterstützung nicht konfiguriert ist.

Dies fügt tendenziell viel Unordnung hinzu. Eine verbreitete Abkürzung ist:

```c
#define _(x) gettext(x)
```

Eine andere Lösung ist möglich, wenn das Programm einen Großteil seiner Kommunikation über eine oder wenige Funktionen abwickelt, etwa `ereport()` im Backend. Dann lassen Sie diese Funktion intern auf allen Eingabestrings `gettext` aufrufen.

3. Fügen Sie im Verzeichnis mit den Programmquellen eine Datei `nls.mk` hinzu. Diese Datei wird als Makefile gelesen. Die folgenden Variablenzuweisungen müssen hier vorgenommen werden:

- `CATALOG_NAME`
  - Der Programmname, wie er im Aufruf `textdomain()` angegeben wurde.

- `GETTEXT_FILES`
  - Liste der Dateien, die übersetzbare Strings enthalten, also solche, die mit `gettext` oder einer alternativen Lösung markiert sind. Schließlich wird dies fast alle Quelldateien des Programms umfassen. Wenn diese Liste zu lang wird, können Sie als erste »Datei« ein `+` verwenden und als zweites Wort eine Datei angeben, die pro Zeile einen Dateinamen enthält.

- `GETTEXT_TRIGGERS`
  - Die Werkzeuge, die Meldungskataloge für die Arbeit der Übersetzer erzeugen, müssen wissen, welche Funktionsaufrufe übersetzbare Strings enthalten. Standardmäßig sind nur `gettext()`-Aufrufe bekannt. Wenn Sie `_` oder andere Bezeichner verwendet haben, müssen Sie sie hier aufführen. Wenn der übersetzbare String nicht das erste Argument ist, muss der Eintrag die Form `func:2` haben (für das zweite Argument). Wenn Sie eine Funktion haben, die pluralisierte Meldungen unterstützt, sollte der Eintrag wie `func:1,2` aussehen (zur Identifikation der Singular- und Plural-Meldungsargumente).

4. Fügen Sie eine Datei `po/LINGUAS` hinzu, die die Liste der bereitgestellten Übersetzungen enthält, anfangs leer.

Das Buildsystem kümmert sich automatisch um das Erstellen und Installieren der Meldungskataloge.

### 56.2.2. Richtlinien zum Schreiben von Meldungen

Hier sind einige Richtlinien für das Schreiben von Meldungen, die leicht übersetzbar sind.

- Konstruieren Sie keine Sätze zur Laufzeit, etwa:

```c
printf("Files were %s.\n", flag ? "copied" : "removed");
```

Die Wortstellung innerhalb des Satzes kann in anderen Sprachen anders sein. Selbst wenn Sie daran denken, jedes Fragment mit `gettext()` aufzurufen, lassen sich die Fragmente möglicherweise nicht gut getrennt übersetzen. Es ist besser, ein wenig Code zu duplizieren, sodass jede zu übersetzende Meldung ein zusammenhängendes Ganzes ist. Nur Zahlen, Dateinamen und ähnliche Laufzeitvariablen sollten zur Laufzeit in einen Meldungstext eingefügt werden.

- Aus ähnlichen Gründen funktioniert dies nicht:

```c
printf("copied %d file%s", n, n!=1 ? "s" : "");
```

denn es setzt voraus, wie der Plural gebildet wird. Wenn Sie dachten, Sie könnten es so lösen:

```c
if (n==1)
     printf("copied 1 file");
else
     printf("copied %d files", n):
```

dann werden Sie enttäuscht sein. Einige Sprachen haben mehr als zwei Formen mit teils eigenartigen Regeln. Oft ist es am besten, die Meldung so zu gestalten, dass die Frage ganz vermieden wird, zum Beispiel so:

```c
printf("number of copied files: %d", n);
```

Wenn Sie wirklich eine korrekt pluralisierte Meldung konstruieren möchten, gibt es Unterstützung dafür, aber sie ist etwas umständlich. Beim Erzeugen einer primären oder Detailfehlermeldung in `ereport()` können Sie etwa Folgendes schreiben:

```c
errmsg_plural("copied %d file",
              "copied %d files",
              n,
              n)
```

Das erste Argument ist der passende Formatstring für die englische Singularform, das zweite der passende Formatstring für die englische Pluralform, und das dritte ist der ganzzahlige Steuerwert, der bestimmt, welche Pluralform zu verwenden ist. Nachfolgende Argumente werden wie üblich gemäß dem Formatstring formatiert. Normalerweise ist der Pluralisierungs-Steuerwert auch einer der zu formatierenden Werte, daher muss er zweimal geschrieben werden. Im Englischen ist nur wichtig, ob `n` 1 oder nicht 1 ist; in anderen Sprachen kann es viele verschiedene Pluralformen geben. Der Übersetzer sieht die beiden englischen Formen als Gruppe und hat die Möglichkeit, mehrere Ersatzstrings bereitzustellen, wobei der passende anhand des Laufzeitwerts von `n` ausgewählt wird.

Wenn Sie eine Meldung pluralisieren müssen, die nicht direkt in einen `errmsg`- oder `errdetail`-Bericht geht, müssen Sie die zugrunde liegende Funktion `ngettext` verwenden. Siehe die Gettext-Dokumentation.

- Wenn Sie dem Übersetzer etwas mitteilen möchten, etwa wie eine Meldung mit anderer Ausgabe ausgerichtet werden soll, stellen Sie dem Auftreten des Strings einen Kommentar voran, der mit `translator` beginnt, zum Beispiel:

```c
/* translator: This message is not what it seems to be. */
```

Diese Kommentare werden in die Meldungskatalogdateien kopiert, sodass Übersetzer sie sehen können.
