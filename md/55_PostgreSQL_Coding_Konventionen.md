# 55. PostgreSQL-Coding-Konventionen

## 55.1. Formatierung

Die Quellcodeformatierung verwendet Tabulatorabstände von 4 Spalten, wobei Tabulatoren erhalten bleiben, also nicht in Leerzeichen umgewandelt werden. Jede logische Einrückungsebene entspricht einem zusätzlichen Tabstopp.

Layoutregeln, etwa die Positionierung von geschweiften Klammern, folgen BSD-Konventionen. Insbesondere stehen geschweifte Klammern für kontrollierte Blöcke von `if`, `while`, `switch` usw. auf eigenen Zeilen.

Begrenzen Sie Zeilenlängen so, dass der Code in einem Fenster mit 80 Spalten lesbar ist. Das bedeutet nicht, dass Sie 80 Spalten niemals überschreiten dürfen. Zum Beispiel ist es wahrscheinlich kein Gewinn an Lesbarkeit, eine lange Fehlermeldungszeichenkette an beliebigen Stellen umzubrechen, nur um den Code innerhalb von 80 Spalten zu halten.

Um einen konsistenten Coding Style zu bewahren, verwenden Sie keine C++-Kommentare (`//`-Kommentare). `pgindent` ersetzt sie durch `/* ... */`.

Der bevorzugte Stil für mehrzeilige Kommentarblöcke ist:

```c
/*
 * comment text begins here
 * and continues here
 */
```

Beachten Sie, dass Kommentarblöcke, die in Spalte 1 beginnen, von `pgindent` unverändert erhalten bleiben; eingerückte Kommentarblöcke werden jedoch wie normaler Text neu umbrochen. Wenn Sie die Zeilenumbrüche in einem eingerückten Block erhalten wollen, fügen Sie Bindestriche wie hier hinzu:

```c
/*----------
 * comment text begins here
 * and continues here
 *----------
 */
```

Eingereichte Patches müssen diesen Formatierungsregeln zwar nicht zwingend folgen, es ist aber sinnvoll, dies zu tun. Ihr Code wird vor der nächsten Veröffentlichung durch `pgindent` laufen, daher bringt es nichts, ihn unter anderen Formatierungskonventionen schön aussehen zu lassen. Eine gute Faustregel für Patches lautet: Lassen Sie neuen Code so aussehen wie den vorhandenen Code in seiner Umgebung.

Das Verzeichnis `src/tools/editors` enthält Beispiel-Einstellungsdateien, die mit den Editoren Emacs, xemacs oder vim verwendet werden können, um sicherzustellen, dass sie Code nach diesen Konventionen formatieren.

Wenn Sie `pgindent` lokal ausführen möchten, damit Ihr Code zum Projektstil passt, lesen Sie das Verzeichnis `src/tools/pgindent`.

Die Textanzeigewerkzeuge `more` und `less` können so aufgerufen werden:

```text
more -x4
less -x4
```

Dadurch zeigen sie Tabulatoren passend an.

## 55.2. Fehler im Server melden

Fehler-, Warn- und Logmeldungen, die im Servercode erzeugt werden, sollten mit `ereport` oder seinem älteren Verwandten `elog` erstellt werden. Die Verwendung dieser Funktion ist komplex genug, um einige Erläuterungen zu benötigen.

Für jede Meldung gibt es zwei erforderliche Elemente: einen Schweregrad (von `DEBUG` bis `PANIC`, definiert in `src/include/utils/elog.h`) und einen primären Meldungstext. Zusätzlich gibt es optionale Elemente; das häufigste ist ein Fehlerkennungscode, der den SQLSTATE-Konventionen der SQL-Spezifikation folgt. `ereport` selbst ist nur ein Shell-Makro, das vor allem der syntaktischen Bequemlichkeit dient, sodass die Meldungserzeugung im C-Quellcode wie ein einzelner Funktionsaufruf aussieht. Der einzige Parameter, den `ereport` direkt akzeptiert, ist der Schweregrad. Der primäre Meldungstext und alle optionalen Meldungselemente werden erzeugt, indem innerhalb des `ereport`-Aufrufs Hilfsfunktionen wie `errmsg` aufgerufen werden.

Ein typischer Aufruf von `ereport` könnte so aussehen:

```c
ereport(ERROR,
        errcode(ERRCODE_DIVISION_BY_ZERO),
        errmsg("division by zero"));
```

Dies gibt den Fehlerschweregrad `ERROR` an, also einen gewöhnlichen Fehler. Der Aufruf `errcode` gibt den SQLSTATE-Fehlercode mit einem Makro an, das in `src/include/utils/errcodes.h` definiert ist. Der Aufruf `errmsg` stellt den primären Meldungstext bereit.

Häufig sieht man auch noch diesen älteren Stil mit einem zusätzlichen Klammerpaar um die Hilfsfunktionsaufrufe:

```c
ereport(ERROR,
        (errcode(ERRCODE_DIVISION_BY_ZERO),
         errmsg("division by zero")));
```

Die zusätzlichen Klammern waren vor PostgreSQL Version 12 erforderlich, sind nun aber optional.

Hier ist ein komplexeres Beispiel:

```c
ereport(ERROR,
        errcode(ERRCODE_AMBIGUOUS_FUNCTION),
        errmsg("function %s is not unique",
               func_signature_string(funcname, nargs,
                                     NIL, actual_arg_types)),
        errhint("Unable to choose a best candidate function. "
                "You might need to add explicit typecasts."));
```

Dies veranschaulicht die Verwendung von Format-Codes, um Laufzeitwerte in einen Meldungstext einzubetten. Außerdem wird eine optionale »hint«-Meldung bereitgestellt. Die Hilfsfunktionsaufrufe können in beliebiger Reihenfolge geschrieben werden, aber üblicherweise stehen `errcode` und `errmsg` zuerst.

Wenn der Schweregrad `ERROR` oder höher ist, bricht `ereport` die Ausführung der aktuellen Abfrage ab und kehrt nicht zum Aufrufer zurück. Wenn der Schweregrad niedriger als `ERROR` ist, kehrt `ereport` normal zurück.

Die verfügbaren Hilfsroutinen für `ereport` sind:

- `errcode(sqlerrcode)`
  - Gibt den SQLSTATE-Fehlerkennungscode für die Bedingung an. Wenn diese Routine nicht aufgerufen wird, ist die Fehlerkennung standardmäßig `ERRCODE_INTERNAL_ERROR`, wenn der Fehlerschweregrad `ERROR` oder höher ist, `ERRCODE_WARNING`, wenn der Fehlergrad `WARNING` ist, andernfalls (für `NOTICE` und darunter) `ERRCODE_SUCCESSFUL_COMPLETION`. Diese Standardwerte sind oft praktisch, aber prüfen Sie immer, ob sie angemessen sind, bevor Sie den Aufruf `errcode()` weglassen.

- `errmsg(const char *msg, ...)`
  - Gibt den primären Fehlermeldungstext und möglicherweise darin einzufügende Laufzeitwerte an. Einfügungen werden durch Format-Codes im Stil von `sprintf` angegeben. Zusätzlich zu den Standard-Format-Codes, die `sprintf` akzeptiert, kann der Format-Code `%m` verwendet werden, um die von `strerror` für den aktuellen Wert von `errno` zurückgegebene Fehlermeldung einzufügen. `%m` benötigt keinen entsprechenden Eintrag in der Parameterliste für `errmsg`. Beachten Sie, dass der Meldungsstring vor der Verarbeitung der Format-Codes durch `gettext` zur möglichen Lokalisierung läuft.

- `errmsg_internal(const char *msg, ...)`
  - Entspricht `errmsg`, außer dass der Meldungsstring nicht übersetzt und nicht in das Internationalisierungs-Meldungswörterbuch aufgenommen wird. Dies sollte für Fälle verwendet werden, die »nicht passieren können« und für die sich Übersetzungsaufwand wahrscheinlich nicht lohnt.

- `errmsg_plural(const char *fmt_singular, const char *fmt_plural, unsigned long n, ...)`
  - Entspricht `errmsg`, unterstützt aber verschiedene Pluralformen der Meldung. `fmt_singular` ist das englische Singularformat, `fmt_plural` das englische Pluralformat, `n` der ganzzahlige Wert, der bestimmt, welche Pluralform benötigt wird, und die übrigen Argumente werden gemäß dem ausgewählten Formatstring formatiert. Weitere Informationen finden Sie in Abschnitt 56.2.2.

- `errdetail(const char *msg, ...)`
  - Stellt eine optionale Detailmeldung bereit. Sie wird verwendet, wenn zusätzliche Informationen vorhanden sind, die für die primäre Meldung ungeeignet erscheinen. Der Meldungsstring wird genauso verarbeitet wie bei `errmsg`.

- `errdetail_internal(const char *msg, ...)`
  - Entspricht `errdetail`, außer dass der Meldungsstring nicht übersetzt und nicht in das Internationalisierungs-Meldungswörterbuch aufgenommen wird. Dies sollte für Detailmeldungen verwendet werden, für die sich Übersetzungsaufwand nicht lohnt, etwa weil sie zu technisch sind, um den meisten Benutzern zu helfen.

- `errdetail_plural(const char *fmt_singular, const char *fmt_plural, unsigned long n, ...)`
  - Entspricht `errdetail`, unterstützt aber verschiedene Pluralformen der Meldung. Weitere Informationen finden Sie in Abschnitt 56.2.2.

- `errdetail_log(const char *msg, ...)`
  - Entspricht `errdetail`, nur dass dieser String ausschließlich ins Serverlog geht, niemals zum Client. Wenn sowohl `errdetail` (oder eines seiner oben genannten Äquivalente) als auch `errdetail_log` verwendet werden, geht ein String zum Client und der andere ins Log. Dies ist nützlich für Fehlerdetails, die zu sicherheitssensibel oder zu umfangreich sind, um in den an den Client gesendeten Bericht aufgenommen zu werden.

- `errdetail_log_plural(const char *fmt_singular, const char *fmt_plural, unsigned long n, ...)`
  - Entspricht `errdetail_log`, unterstützt aber verschiedene Pluralformen der Meldung. Weitere Informationen finden Sie in Abschnitt 56.2.2.

- `errhint(const char *msg, ...)`
  - Stellt eine optionale Hint-Meldung bereit. Sie wird verwendet, wenn Vorschläge angeboten werden, wie das Problem behoben werden kann, im Unterschied zu sachlichen Details darüber, was schiefgelaufen ist. Der Meldungsstring wird genauso verarbeitet wie bei `errmsg`.

- `errhint_plural(const char *fmt_singular, const char *fmt_plural, unsigned long n, ...)`
  - Entspricht `errhint`, unterstützt aber verschiedene Pluralformen der Meldung. Weitere Informationen finden Sie in Abschnitt 56.2.2.

- `errcontext(const char *msg, ...)`
  - Wird normalerweise nicht direkt an einer `ereport`-Meldungsstelle aufgerufen, sondern in `error_context_stack`-Callback-Funktionen verwendet, um Informationen über den Kontext bereitzustellen, in dem ein Fehler auftrat, etwa die aktuelle Position in einer PL-Funktion. Der Meldungsstring wird genauso verarbeitet wie bei `errmsg`. Anders als die anderen Hilfsfunktionen kann diese Funktion mehr als einmal pro `ereport`-Aufruf aufgerufen werden; die so bereitgestellten aufeinanderfolgenden Strings werden mit trennenden Newlines verkettet.

- `errposition(int cursorpos)`
  - Gibt die Textposition eines Fehlers innerhalb eines Abfragestrings an. Derzeit ist dies nur für Fehler nützlich, die in den lexikalischen und syntaktischen Analysephasen der Abfrageverarbeitung erkannt werden.

- `errtable(Relation rel)`
  - Gibt eine Relation an, deren Name und Schemaname als Hilfsfelder in den Fehlerbericht aufgenommen werden sollen.

- `errtablecol(Relation rel, int attnum)`
  - Gibt eine Spalte an, deren Name, Tabellenname und Schemaname als Hilfsfelder in den Fehlerbericht aufgenommen werden sollen.

- `errtableconstraint(Relation rel, const char *conname)`
  - Gibt einen Tabellen-Constraint an, dessen Name, Tabellenname und Schemaname als Hilfsfelder in den Fehlerbericht aufgenommen werden sollen. Indizes sollten für diesen Zweck als Constraints betrachtet werden, unabhängig davon, ob sie einen zugehörigen Eintrag in `pg_constraint` haben. Achten Sie darauf, als `rel` die zugrunde liegende Heap-Relation zu übergeben, nicht den Index selbst.

- `errdatatype(Oid datatypeOid)`
  - Gibt einen Datentyp an, dessen Name und Schemaname als Hilfsfelder in den Fehlerbericht aufgenommen werden sollen.

- `errdomainconstraint(Oid datatypeOid, const char *conname)`
  - Gibt einen Domain-Constraint an, dessen Name, Domainname und Schemaname als Hilfsfelder in den Fehlerbericht aufgenommen werden sollen.

- `errcode_for_file_access()`
  - Eine Komfortfunktion, die für einen Fehler in einem dateizugriffsbezogenen Systemaufruf eine passende SQLSTATE-Fehlerkennung auswählt. Sie verwendet das gespeicherte `errno`, um zu bestimmen, welcher Fehlercode erzeugt werden soll. Normalerweise sollte dies zusammen mit `%m` im primären Fehlermeldungstext verwendet werden.

- `errcode_for_socket_access()`
  - Eine Komfortfunktion, die für einen Fehler in einem socketbezogenen Systemaufruf eine passende SQLSTATE-Fehlerkennung auswählt.

- `errhidestmt(bool hide_stmt)`
  - Kann aufgerufen werden, um die Unterdrückung des `STATEMENT:`-Teils einer Meldung im Postmaster-Log anzugeben. Im Allgemeinen ist dies angemessen, wenn der Meldungstext die aktuelle Anweisung bereits enthält.

- `errhidecontext(bool hide_ctx)`
  - Kann aufgerufen werden, um die Unterdrückung des `CONTEXT:`-Teils einer Meldung im Postmaster-Log anzugeben. Dies sollte nur für ausführliche Debugging-Meldungen verwendet werden, bei denen die wiederholte Aufnahme des Kontexts das Log zu stark aufblähen würde.

> **Hinweis:** Höchstens eine der Funktionen `errtable`, `errtablecol`, `errtableconstraint`, `errdatatype` oder `errdomainconstraint` sollte in einem `ereport`-Aufruf verwendet werden. Diese Funktionen existieren, damit Anwendungen den Namen eines Datenbankobjekts extrahieren können, das mit der Fehlerbedingung verbunden ist, ohne den potenziell lokalisierten Fehlermeldungstext untersuchen zu müssen. Diese Funktionen sollten in Fehlerberichten verwendet werden, bei denen Anwendungen wahrscheinlich automatische Fehlerbehandlung wünschen. Seit PostgreSQL 9.3 existiert vollständige Abdeckung nur für Fehler in SQLSTATE-Klasse 23 (Integritäts-Constraint-Verletzung), dies wird aber voraussichtlich künftig erweitert.

Es gibt eine ältere Funktion `elog`, die noch immer häufig verwendet wird. Ein `elog`-Aufruf:

```c
elog(level, "format string", ...);
```

ist exakt äquivalent zu:

```c
ereport(level, errmsg_internal("format string", ...));
```

Beachten Sie, dass der SQLSTATE-Fehlercode immer auf den Standardwert gesetzt wird und der Meldungsstring nicht übersetzt wird. Daher sollte `elog` nur für interne Fehler und Low-Level-Debug-Logging verwendet werden. Jede Meldung, die für gewöhnliche Benutzer wahrscheinlich von Interesse ist, sollte über `ereport` laufen. Dennoch gibt es im System genug interne Prüfungen auf »kann nicht passieren«-Fehler, dass `elog` weiterhin breit verwendet wird; wegen seiner notationellen Einfachheit wird es für diese Meldungen bevorzugt.

Hinweise zum Schreiben guter Fehlermeldungen finden Sie in [Abschnitt 55.3](55_PostgreSQL_Coding_Konventionen.md#553-stilleitfaden-für-fehlermeldungen).

## 55.3. Stilleitfaden für Fehlermeldungen

Dieser Stilleitfaden wird in der Hoffnung angeboten, in allen von PostgreSQL erzeugten Meldungen einen konsistenten, benutzerfreundlichen Stil zu bewahren.

### Was wohin gehört

Die primäre Meldung sollte kurz und sachlich sein und Verweise auf Implementierungsdetails wie bestimmte Funktionsnamen vermeiden. »Kurz« bedeutet: Sie sollte unter normalen Bedingungen in eine Zeile passen. Verwenden Sie bei Bedarf eine Detailmeldung, um die primäre Meldung kurz zu halten, oder wenn Sie Implementierungsdetails wie den konkreten fehlgeschlagenen Systemaufruf erwähnen müssen. Sowohl primäre als auch Detailmeldungen sollten sachlich sein. Verwenden Sie eine Hint-Meldung für Vorschläge, wie das Problem behoben werden kann, besonders wenn der Vorschlag nicht immer anwendbar sein könnte.

Schreiben Sie zum Beispiel statt:

```text
IpcMemoryCreate: shmget(key=%d, size=%u, 0%o) failed: %m
(plus a long addendum that is basically a hint)
```

dies:

```text
Primary:          could not create shared memory segment: %m
Detail:           Failed syscall was shmget(key=%d, size=%u, 0%o).
Hint:             The addendum, written as a complete sentence.
```

Begründung: Wenn die primäre Meldung kurz bleibt, bleibt sie auf den Punkt, und Clients können den Bildschirmplatz unter der Annahme gestalten, dass eine Zeile für Fehlermeldungen ausreicht. Detail- und Hint-Meldungen können in einen ausführlichen Modus oder vielleicht in ein Pop-up-Fenster mit Fehlerdetails ausgelagert werden. Außerdem werden Details und Hinweise normalerweise aus Platzgründen im Serverlog unterdrückt. Verweise auf Implementierungsdetails sollten möglichst vermieden werden, da Benutzer diese Details nicht kennen müssen.

### Formatierung

Legen Sie in den Meldungstexten keine bestimmten Annahmen über die Formatierung fest. Erwarten Sie, dass Clients und das Serverlog Zeilen passend zu ihren eigenen Bedürfnissen umbrechen. In langen Meldungen können Newline-Zeichen (`\n`) verwendet werden, um vorgeschlagene Absatzumbrüche anzugeben. Beenden Sie eine Meldung nicht mit einem Newline. Verwenden Sie keine Tabulatoren oder andere Formatierungszeichen. In Fehlerkontextanzeigen werden Newlines automatisch hinzugefügt, um Kontextebenen wie Funktionsaufrufe zu trennen.

Begründung: Meldungen werden nicht notwendigerweise auf terminalartigen Anzeigen dargestellt. In GUI-Anzeigen oder Browsern werden solche Formatierungsanweisungen bestenfalls ignoriert.

### Anführungszeichen

Englischer Text sollte doppelte Anführungszeichen verwenden, wenn Quoting angemessen ist. Text in anderen Sprachen sollte konsistent eine Art von Anführungszeichen verwenden, die den Veröffentlichungskonventionen und der Computerausgabe anderer Programme in dieser Sprache entspricht.

Begründung: Die Wahl doppelter statt einfacher Anführungszeichen ist etwas willkürlich, entspricht aber tendenziell der bevorzugten Verwendung. Einige haben vorgeschlagen, die Art der Anführungszeichen je nach Objekttyp gemäß SQL-Konventionen zu wählen, also Strings in einfachen Anführungszeichen und Bezeichner in doppelten. Das ist jedoch eine sprachinterne technische Frage, mit der viele Benutzer nicht einmal vertraut sind, skaliert nicht auf andere Arten zitierter Begriffe, lässt sich nicht auf andere Sprachen übertragen und ist außerdem ziemlich zwecklos.

### Verwendung von Anführungszeichen

Verwenden Sie immer Anführungszeichen, um Dateinamen, benutzerangegebene Bezeichner, Konfigurationsvariablennamen und andere Variablen abzugrenzen, die Wörter enthalten könnten. Verwenden Sie sie nicht, um Variablen zu markieren, die keine Wörter enthalten, zum Beispiel Operatornamen.

Im Backend gibt es Funktionen, die ihre eigene Ausgabe bei Bedarf doppelt quoten, etwa `format_type_be()`. Setzen Sie um die Ausgabe solcher Funktionen keine zusätzlichen Anführungszeichen.

Begründung: Objekte können Namen haben, die in einer Meldung mehrdeutig werden. Seien Sie konsistent dabei, kenntlich zu machen, wo ein eingesetzter Name beginnt und endet. Überladen Sie Meldungen aber nicht mit unnötigen oder doppelten Anführungszeichen.

### Grammatik und Zeichensetzung

Für primäre Fehlermeldungen und für Detail-/Hint-Meldungen gelten unterschiedliche Regeln:

Primäre Fehlermeldungen: Schreiben Sie den ersten Buchstaben nicht groß. Beenden Sie eine Meldung nicht mit einem Punkt. Denken Sie nicht einmal daran, eine Meldung mit einem Ausrufezeichen zu beenden.

Detail- und Hint-Meldungen: Verwenden Sie vollständige Sätze und beenden Sie jeden mit einem Punkt. Schreiben Sie das erste Wort von Sätzen groß. Setzen Sie zwei Leerzeichen nach dem Punkt, wenn ein weiterer Satz folgt (für englischen Text; in anderen Sprachen kann dies unpassend sein).

Fehlerkontext-Strings: Schreiben Sie den ersten Buchstaben nicht groß und beenden Sie den String nicht mit einem Punkt. Kontext-Strings sollten normalerweise keine vollständigen Sätze sein.

Begründung: Das Vermeiden von Zeichensetzung erleichtert es Clientanwendungen, die Meldung in verschiedene grammatische Kontexte einzubetten. Häufig sind primäre Meldungen ohnehin keine grammatisch vollständigen Sätze. Wenn sie lang genug sind, um mehr als einen Satz zu bilden, sollten sie in primären und Detailteil aufgeteilt werden. Detail- und Hint-Meldungen sind dagegen länger und müssen möglicherweise mehrere Sätze enthalten. Aus Konsistenzgründen sollten sie dem Stil vollständiger Sätze folgen, auch wenn es nur einen Satz gibt.

### Groß- und Kleinschreibung

Verwenden Sie Kleinschreibung für den Meldungstext, einschließlich des ersten Buchstabens einer primären Fehlermeldung. Verwenden Sie Großschreibung für SQL-Befehle und Schlüsselwörter, wenn sie in der Meldung erscheinen.

Begründung: So ist es leichter, alles konsistent aussehen zu lassen, da manche Meldungen vollständige Sätze sind und manche nicht.

### Passiv vermeiden

Verwenden Sie Aktiv. Verwenden Sie vollständige Sätze, wenn es ein handelndes Subjekt gibt (»A could not do B«). Verwenden Sie Telegrammstil ohne Subjekt, wenn das Subjekt das Programm selbst wäre; verwenden Sie nicht »I« für das Programm.

Begründung: Das Programm ist kein Mensch. Tun Sie nicht so, als wäre es einer.

### Gegenwart und Vergangenheit

Verwenden Sie Vergangenheitsform, wenn ein Versuch, etwas zu tun, fehlgeschlagen ist, beim nächsten Mal aber vielleicht gelingen könnte, etwa nachdem ein Problem behoben wurde. Verwenden Sie Gegenwartsform, wenn der Fehler sicher dauerhaft ist.

Es gibt einen nichttrivialen semantischen Unterschied zwischen Sätzen der Form:

```text
could not open file "%s": %m
```

und:

```text
cannot open file "%s"
```

Die erste Form bedeutet, dass der Versuch, die Datei zu öffnen, fehlgeschlagen ist. Die Meldung sollte einen Grund angeben, etwa »disk full« oder »file doesn't exist«. Die Vergangenheitsform ist angemessen, weil die Platte beim nächsten Mal vielleicht nicht mehr voll ist oder die betreffende Datei existiert.

Die zweite Form zeigt an, dass die Funktionalität, die benannte Datei zu öffnen, im Programm überhaupt nicht existiert oder konzeptionell unmöglich ist. Die Gegenwartsform ist angemessen, weil die Bedingung unbegrenzt fortbestehen wird.

Begründung: Zugegeben, der durchschnittliche Benutzer wird allein aus der Zeitform der Meldung keine großen Schlüsse ziehen können; da die Sprache uns aber Grammatik bereitstellt, sollten wir sie korrekt verwenden.

### Art des Objekts

Wenn Sie den Namen eines Objekts anführen, geben Sie an, um welche Art von Objekt es sich handelt.

Begründung: Andernfalls weiß niemand, worauf sich `foo.bar.baz` bezieht.

### Klammern

Eckige Klammern sollten nur verwendet werden, um in Befehlssynopsen optionale Argumente zu kennzeichnen oder um einen Array-Index anzugeben.

Begründung: Alles andere entspricht keinem weithin bekannten üblichen Gebrauch und verwirrt Menschen.

### Fehlermeldungen zusammensetzen

Wenn eine Meldung Text enthält, der an anderer Stelle erzeugt wird, betten Sie ihn in diesem Stil ein:

```text
could not open file %s: %m
```

Begründung: Es wäre schwierig, alle möglichen Fehlercodes zu berücksichtigen, um sie in einen einzigen glatten Satz einzufügen; daher ist irgendeine Art von Zeichensetzung nötig. Auch wurde vorgeschlagen, den eingebetteten Text in Klammern zu setzen, aber das ist unnatürlich, wenn der eingebettete Text wahrscheinlich der wichtigste Teil der Meldung ist, wie es oft der Fall ist.

### Fehlergründe

Meldungen sollten immer angeben, warum ein Fehler aufgetreten ist. Zum Beispiel:

```text
BAD:    could not open file %s
BETTER: could not open file %s (I/O failure)
```

Wenn kein Grund bekannt ist, sollten Sie besser den Code reparieren.

### Funktionsnamen

Nehmen Sie den Namen der meldenden Routine nicht in den Fehlertext auf. Es gibt andere Mechanismen, dies bei Bedarf herauszufinden, und für die meisten Benutzer ist es keine hilfreiche Information. Wenn der Fehlertext ohne den Funktionsnamen nicht ausreichend sinnvoll ist, formulieren Sie ihn um.

```text
BAD:    pg_strtoint32: error in "z": cannot parse "z"
BETTER: invalid input syntax for type integer: "z"
```

Vermeiden Sie auch die Nennung aufgerufener Funktionsnamen; sagen Sie stattdessen, was der Code zu tun versuchte:

```text
BAD:    open() failed: %m
BETTER: could not open file %s: %m
```

Wenn es wirklich nötig erscheint, erwähnen Sie den Systemaufruf in der Detailmeldung. In einigen Fällen kann es angemessen sein, die tatsächlich an den Systemaufruf übergebenen Werte in der Detailmeldung anzugeben.

Begründung: Benutzer wissen nicht, was all diese Funktionen tun.

### Schwierige Wörter, die zu vermeiden sind

- Unable
  - »Unable« ist fast Passiv. Verwenden Sie besser je nach Situation »cannot« oder »could not«.

- Bad
  - Fehlermeldungen wie »bad result« sind schwer sinnvoll zu interpretieren. Besser ist zu schreiben, warum das Ergebnis »bad« ist, zum Beispiel »invalid format«.

- Illegal
  - »Illegal« steht für einen Gesetzesverstoß; alles andere ist »invalid«. Noch besser ist, zu sagen, warum es ungültig ist.

- Unknown
  - Vermeiden Sie nach Möglichkeit »unknown«. Betrachten Sie »error: unknown response«: Wenn Sie nicht wissen, was die Antwort ist, woher wissen Sie dann, dass sie fehlerhaft ist? »Unrecognized« ist oft die bessere Wahl. Achten Sie außerdem darauf, den beanstandeten Wert aufzunehmen.

```text
BAD:    unknown node type
BETTER: unrecognized node type: 42
```

- Find vs. Exists
  - Wenn das Programm einen nichttrivialen Algorithmus verwendet, um eine Ressource zu lokalisieren, etwa eine Pfadsuche, und dieser Algorithmus fehlschlägt, ist es fair zu sagen, dass das Programm die Ressource nicht »finden« konnte. Wenn dagegen der erwartete Ort der Ressource bekannt ist, das Programm dort aber nicht darauf zugreifen kann, sagen Sie, dass die Ressource nicht »existiert«. In diesem Fall klingt »find« schwach und verwirrt die Frage.

- May vs. Can vs. Might
  - »May« deutet auf Erlaubnis hin (zum Beispiel »You may borrow my rake.«) und hat in Dokumentation oder Fehlermeldungen wenig Nutzen. »Can« deutet auf Fähigkeit hin (zum Beispiel »I can lift that log.«), und »might« deutet auf Möglichkeit hin (zum Beispiel »It might rain today.«). Das richtige Wort klärt die Bedeutung und unterstützt die Übersetzung.

- Contractions
  - Vermeiden Sie Zusammenziehungen wie »can't«; verwenden Sie stattdessen »cannot«.

- Non-negative
  - Vermeiden Sie »non-negative«, weil unklar ist, ob null akzeptiert wird. Besser ist »greater than zero« oder »greater than or equal to zero«.

### Korrekte Schreibweise

Schreiben Sie Wörter vollständig aus. Vermeiden Sie zum Beispiel:

- `spec`
- `stats`
- `parens`
- `auth`
- `xact`

Begründung: Das verbessert die Konsistenz.

### Lokalisierung

Denken Sie daran, dass Fehlermeldungstexte in andere Sprachen übersetzt werden müssen. Befolgen Sie die Richtlinien in Abschnitt 56.2.2, um Übersetzern das Leben nicht schwer zu machen.

## 55.4. Verschiedene Coding-Konventionen

### C-Standard

Code in PostgreSQL sollte sich nur auf Sprachmerkmale stützen, die im C99-Standard verfügbar sind. Das bedeutet, dass ein konformer C99-Compiler `postgres` kompilieren können muss, zumindest abgesehen von einigen plattformabhängigen Teilen.

Einige Merkmale, die im C99-Standard enthalten sind, dürfen derzeit im PostgreSQL-Core-Code nicht verwendet werden. Dazu gehören aktuell Arrays variabler Länge, vermischte Deklarationen und Code, `//`-Kommentare sowie universelle Zeichennamen. Gründe dafür sind unter anderem Portabilität und historische Praxis.

Merkmale aus späteren Revisionen des C-Standards oder compilerspezifische Merkmale können verwendet werden, wenn ein Fallback bereitgestellt wird.

Zum Beispiel werden `_Static_assert()` und `__builtin_constant_p` derzeit verwendet, obwohl sie aus neueren Revisionen des C-Standards beziehungsweise eine GCC-Erweiterung sind. Wenn sie nicht verfügbar sind, fallen wir jeweils auf einen C99-kompatiblen Ersatz zurück, der dieselben Prüfungen ausführt, aber ziemlich kryptische Meldungen ausgibt, und verwenden `__builtin_constant_p` nicht.

### Funktionsartige Makros und Inline-Funktionen

Sowohl Makros mit Argumenten als auch `static inline`-Funktionen dürfen verwendet werden. Letztere sind vorzuziehen, wenn bei einer Makroform Gefahren durch Mehrfachauswertung bestehen, wie etwa bei:

```c
#define Max(x, y)                    ((x) > (y) ? (x) : (y))
```

oder wenn das Makro sehr lang wäre. In anderen Fällen ist es nur möglich oder zumindest einfacher, Makros zu verwenden, etwa weil Ausdrücke verschiedener Typen an das Makro übergeben werden müssen.

Wenn die Definition einer Inline-Funktion auf Symbole verweist, also Variablen oder Funktionen, die nur als Teil des Backends verfügbar sind, darf die Funktion nicht sichtbar sein, wenn sie aus Frontend-Code eingebunden wird.

```c
#ifndef FRONTEND
static inline MemoryContext
MemoryContextSwitchTo(MemoryContext context)
{
    MemoryContext old = CurrentMemoryContext;

    CurrentMemoryContext = context;
    return old;
}
#endif        /* FRONTEND */
```

In diesem Beispiel wird auf `CurrentMemoryContext` verwiesen, das nur im Backend verfügbar ist; die Funktion wird daher mit `#ifndef FRONTEND` verborgen. Diese Regel existiert, weil einige Compiler Verweise auf Symbole ausgeben, die in Inline-Funktionen enthalten sind, selbst wenn die Funktion nicht verwendet wird.

### Signal-Handler schreiben

Damit Code in einem Signal-Handler ausgeführt werden kann, muss er sehr sorgfältig geschrieben sein. Das Grundproblem ist, dass ein Signal-Handler Code jederzeit unterbrechen kann, sofern Signale nicht blockiert sind. Wenn Code im Signal-Handler denselben Zustand verwendet wie Code außerhalb, kann Chaos entstehen. Betrachten Sie zum Beispiel, was passiert, wenn ein Signal-Handler versucht, eine Sperre zu erwerben, die im unterbrochenen Code bereits gehalten wird.

Ohne besondere Vorkehrungen darf Code in Signal-Handlern nur async-signal-sichere Funktionen aufrufen (wie in POSIX definiert) und auf Variablen des Typs `volatile sig_atomic_t` zugreifen. Einige Funktionen in `postgres` gelten ebenfalls als signalsicher, wichtig ist insbesondere `SetLatch()`.

In den meisten Fällen sollten Signal-Handler nicht mehr tun, als zu vermerken, dass ein Signal eingetroffen ist, und Code außerhalb des Handlers mit einer Latch zu wecken. Ein Beispiel für einen solchen Handler ist:

```c
static void
handle_sighup(SIGNAL_ARGS)
{
    got_SIGHUP = true;
    SetLatch(MyLatch);
}
```

### Funktionszeiger aufrufen

Aus Gründen der Klarheit wird bevorzugt, einen Funktionszeiger beim Aufruf der Funktion, auf die er zeigt, ausdrücklich zu dereferenzieren, wenn der Zeiger eine einfache Variable ist, zum Beispiel:

```c
(*emit_log_hook) (edata);
```

Dies gilt, obwohl auch `emit_log_hook(edata)` funktionieren würde. Wenn der Funktionszeiger Teil einer Struktur ist, kann und sollte die zusätzliche Zeichensetzung normalerweise weggelassen werden, zum Beispiel:

```c
paramInfo->paramFetch(paramInfo, paramId);
```
