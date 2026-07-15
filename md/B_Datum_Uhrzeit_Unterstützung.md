# Anhang B. Datum-/Uhrzeit-Unterstützung

PostgreSQL verwendet für alle Datum-/Uhrzeit-Eingaben einen internen heuristischen Parser. Datums- und Uhrzeitwerte werden als Zeichenketten eingegeben und in einzelne Felder zerlegt; dabei wird vorläufig bestimmt, welche Art von Information in jedem Feld stehen kann. Jedes Feld wird interpretiert und entweder einem numerischen Wert zugewiesen, ignoriert oder zurückgewiesen. Der Parser enthält interne Lookup-Tabellen für alle textuellen Felder, einschließlich Monatsnamen, Wochentagen und Zeitzonen.

Dieser Anhang enthält Informationen über den Inhalt dieser Lookup-Tabellen und beschreibt die Schritte, mit denen der Parser Datums- und Uhrzeitangaben decodiert.

### B.1. Interpretation von Datum-/Uhrzeit-Eingaben

Datum-/Uhrzeit-Eingabezeichenketten werden nach folgendem Verfahren decodiert:

1. Die Eingabezeichenkette wird in Tokens zerlegt, und jedes Token wird als Zeichenkette, Uhrzeit, Zeitzone oder Zahl kategorisiert.
   - Wenn das numerische Token einen Doppelpunkt (`:`) enthält, ist es eine Uhrzeitzeichenkette. Alle nachfolgenden Ziffern und Doppelpunkte werden einbezogen.
   - Wenn das numerische Token einen Bindestrich (`-`), einen Schrägstrich (`/`) oder zwei oder mehr Punkte (`.`) enthält, ist es eine Datumszeichenkette, die einen textuellen Monat enthalten kann. Wenn bereits ein Datumstoken gesehen wurde, wird es stattdessen als Zeitzonenname interpretiert, zum Beispiel `America/New_York`.
   - Wenn das Token nur numerisch ist, ist es entweder ein einzelnes Feld oder ein nach ISO 8601 zusammengefügtes Datum, zum Beispiel `19990113` für den 13. Januar 1999, oder eine zusammengefügte Uhrzeit, zum Beispiel `141516` für 14:15:16.
   - Wenn das Token mit Plus (`+`) oder Minus (`-`) beginnt, ist es entweder eine numerische Zeitzone oder ein Sonderfeld.

2. Wenn das Token eine alphabetische Zeichenkette ist, wird es mit möglichen Zeichenketten abgeglichen.
   - Zuerst wird geprüft, ob das Token einer bekannten Zeitzonenabkürzung entspricht. Diese Abkürzungen werden durch die in [Abschnitt B.4](B_Datum_Uhrzeit_Unterstützung.md#b4-datumuhrzeitkonfigurationsdateien) beschriebenen Konfigurationseinstellungen bestimmt.
   - Wenn nichts gefunden wird, wird in einer internen Tabelle danach gesucht, ob das Token eine besondere Zeichenkette, zum Beispiel `today`, ein Wochentag, zum Beispiel `Thursday`, ein Monat, zum Beispiel `January`, oder ein Füllwort, zum Beispiel `at` oder `on`, ist.
   - Wenn weiterhin nichts gefunden wird, wird ein Fehler ausgelöst.

3. Wenn das Token eine Zahl oder ein Zahlenfeld ist:
   - Wenn es acht oder sechs Ziffern hat und noch keine anderen Datumsfelder gelesen wurden, wird es als „zusammengefügtes Datum“ interpretiert, zum Beispiel `19990118` oder `990118`. Die Interpretation ist `YYYYMMDD` oder `YYMMDD`.
   - Wenn das Token drei Ziffern hat und bereits ein Jahr gelesen wurde, wird es als Tag des Jahres interpretiert.
   - Wenn es vier oder sechs Ziffern hat und bereits ein Jahr gelesen wurde, wird es als Uhrzeit interpretiert (`HHMM` oder `HHMMSS`).
   - Wenn es drei oder mehr Ziffern hat und noch keine Datumsfelder gefunden wurden, wird es als Jahr interpretiert. Das erzwingt für die übrigen Datumsfelder die Reihenfolge `yy-mm-dd`.
   - Andernfalls wird angenommen, dass die Reihenfolge der Datumsfelder der Einstellung `DateStyle` folgt: `mm-dd-yy`, `dd-mm-yy` oder `yy-mm-dd`. Es wird ein Fehler ausgelöst, wenn ein Monats- oder Tagesfeld außerhalb des zulässigen Bereichs liegt.

4. Wenn `BC` angegeben wurde, wird das Jahr negiert und für die interne Speicherung eins addiert. Im gregorianischen Kalender gibt es kein Jahr null; numerisch wird daher 1 v. Chr. zu Jahr null.

5. Wenn `BC` nicht angegeben wurde und das Jahresfeld zwei Ziffern lang ist, wird das Jahr auf vier Ziffern angepasst. Wenn das Feld kleiner als 70 ist, wird 2000 addiert, andernfalls 1900.

> **Tipp:** Gregorianische Jahre n. Chr. 1 bis 99 können durch vier Ziffern mit führenden Nullen eingegeben werden, zum Beispiel `0099` für 99 n. Chr.

### B.2. Behandlung ungültiger oder mehrdeutiger Zeitstempel

Normalerweise wird ein Fehler ausgelöst, wenn eine Datum-/Uhrzeitzeichenkette syntaktisch gültig ist, aber Feldwerte außerhalb des zulässigen Bereichs enthält. Eine Eingabe, die den 31. Februar angibt, wird zum Beispiel zurückgewiesen.

Während eines Sommerzeitübergangs kann es passieren, dass eine scheinbar gültige Zeitstempelzeichenkette einen nicht existierenden oder mehrdeutigen Zeitstempel darstellt. Solche Fälle werden nicht zurückgewiesen; die Mehrdeutigkeit wird aufgelöst, indem bestimmt wird, welcher UTC-Offset anzuwenden ist. Angenommen, der Parameter `TimeZone` ist auf `America/New_York` gesetzt:

```sql
SELECT '2018-03-11 02:30'::timestamptz;
```

```text
      timestamptz
------------------------
 2018-03-11 03:30:00-04
(1 row)
```

Da dieser Tag in dieser Zeitzone ein Spring-forward-Übergang war, gab es keinen zivilen Zeitpunkt 2:30 Uhr; die Uhren sprangen von 2:00 Uhr EST auf 3:00 Uhr EDT. PostgreSQL interpretiert die angegebene Zeit so, als wäre sie Standardzeit (UTC-5), was dann als 3:30 Uhr EDT (UTC-4) dargestellt wird.

Umgekehrt verhält es sich bei einem Fall-back-Übergang:

```sql
SELECT '2018-11-04 01:30'::timestamptz;
```

```text
      timestamptz
------------------------
 2018-11-04 01:30:00-05
(1 row)
```

An diesem Datum gab es zwei mögliche Interpretationen von 1:30 Uhr: einmal 1:30 Uhr EDT und eine Stunde später, nachdem die Uhren von 2:00 Uhr EDT auf 1:00 Uhr EST zurückgesprungen waren, 1:30 Uhr EST. Wieder interpretiert PostgreSQL die angegebene Zeit so, als wäre sie Standardzeit (UTC-5). Die andere Interpretation kann erzwungen werden, indem Sommerzeit angegeben wird:

```sql
SELECT '2018-11-04 01:30 EDT'::timestamptz;
```

```text
      timestamptz
------------------------
 2018-11-04 01:30:00-04
(1 row)
```

Die genaue Regel für solche Fälle lautet: Ein ungültiger Zeitstempel, der in einen Vorwärtssprung einer Sommerzeitumstellung zu fallen scheint, erhält den UTC-Offset, der in der Zeitzone unmittelbar vor dem Übergang galt. Ein mehrdeutiger Zeitstempel, der auf beiden Seiten eines Rückwärtssprungs liegen könnte, erhält den UTC-Offset, der unmittelbar nach dem Übergang galt. In den meisten Zeitzonen entspricht das der Aussage: Im Zweifel wird die Standardzeit-Interpretation bevorzugt.

In allen Fällen kann der mit einem Zeitstempel verbundene UTC-Offset explizit angegeben werden, entweder durch einen numerischen UTC-Offset oder durch eine Zeitzonenabkürzung, die einem festen UTC-Offset entspricht. Die gerade beschriebene Regel gilt nur dann, wenn ein UTC-Offset für eine Zeitzone abgeleitet werden muss, in der sich der Offset ändert.

### B.3. Datum-/Uhrzeit-Schlüsselwörter

Tabelle B.1 zeigt die Tokens, die als Monatsnamen erkannt werden.

**Tabelle B.1. Monatsnamen**

| Monat | Abkürzungen |
| --- | --- |
| `January` | `Jan` |
| `February` | `Feb` |
| `March` | `Mar` |
| `April` | `Apr` |
| `May` | |
| `June` | `Jun` |
| `July` | `Jul` |
| `August` | `Aug` |
| `September` | `Sep`, `Sept` |
| `October` | `Oct` |
| `November` | `Nov` |
| `December` | `Dec` |

Tabelle B.2 zeigt die Tokens, die als Wochentagsnamen erkannt werden.

**Tabelle B.2. Wochentagsnamen**

| Tag | Abkürzungen |
| --- | --- |
| `Sunday` | `Sun` |
| `Monday` | `Mon` |
| `Tuesday` | `Tue`, `Tues` |
| `Wednesday` | `Wed`, `Weds` |
| `Thursday` | `Thu`, `Thur`, `Thurs` |
| `Friday` | `Fri` |
| `Saturday` | `Sat` |

Tabelle B.3 zeigt die Tokens, die verschiedene Modifikatorzwecke erfüllen.

**Tabelle B.3. Datum-/Uhrzeit-Feldmodifikatoren**

| Bezeichner | Beschreibung |
| --- | --- |
| `AM` | Zeit liegt vor 12:00 |
| `AT` | Wird ignoriert |
| `JULIAN`, `JD`, `J` | Das nächste Feld ist ein julianisches Datum |
| `ON` | Wird ignoriert |
| `PM` | Zeit liegt bei oder nach 12:00 |
| `T` | Das nächste Feld ist eine Uhrzeit |

### B.4. Datum-/Uhrzeit-Konfigurationsdateien

Da Zeitzonenabkürzungen nicht gut standardisiert sind, stellt PostgreSQL eine Möglichkeit bereit, die Menge der Abkürzungen anzupassen, die in Datum-/Uhrzeit-Eingaben akzeptiert werden. Es gibt zwei Quellen für diese Abkürzungen:

1. Der Laufzeitparameter `TimeZone` ist normalerweise auf den Namen eines Eintrags in der IANA-Zeitzonendatenbank gesetzt. Wenn diese Zone verbreitet verwendete Zonenabkürzungen hat, erscheinen sie in den IANA-Daten, und PostgreSQL erkennt diese Abkürzungen bevorzugt mit den Bedeutungen aus den IANA-Daten. Wenn `timezone` zum Beispiel auf `America/New_York` gesetzt ist, wird `EST` als UTC-5 und `EDT` als UTC-4 verstanden. Diese IANA-Abkürzungen werden auch in der Datum-/Uhrzeit-Ausgabe verwendet, wenn `DateStyle` auf einen Stil gesetzt ist, der nichtnumerische Zonenabkürzungen bevorzugt.

2. Wenn eine Abkürzung in der aktuellen IANA-Zeitzone nicht gefunden wird, wird sie in der durch den Laufzeitparameter `timezone_abbreviations` angegebenen Liste gesucht. Die Liste `timezone_abbreviations` ist hauptsächlich nützlich, damit Datum-/Uhrzeit-Eingaben Abkürzungen für andere Zeitzonen als die aktuelle erkennen können. Diese Abkürzungen werden nicht in der Datum-/Uhrzeit-Ausgabe verwendet.

Der Parameter `timezone_abbreviations` kann zwar von jedem Datenbankbenutzer geändert werden, seine möglichen Werte stehen jedoch unter Kontrolle des Datenbankadministrators: Es handelt sich tatsächlich um Namen von Konfigurationsdateien im Verzeichnis `.../share/timezonesets/` des Installationsverzeichnisses. Durch Hinzufügen oder Ändern von Dateien in diesem Verzeichnis kann der Administrator lokale Regeln für Zeitzonenabkürzungen festlegen.

`timezone_abbreviations` kann auf jeden Dateinamen gesetzt werden, der in `.../share/timezonesets/` gefunden wird, wenn der Dateiname ausschließlich alphabetisch ist. Das Verbot nichtalphabetischer Zeichen in `timezone_abbreviations` verhindert das Lesen von Dateien außerhalb des vorgesehenen Verzeichnisses sowie das Lesen von Editor-Backup-Dateien und anderen fremden Dateien.

Eine Datei mit Zeitzonenabkürzungen kann Leerzeilen und Kommentare enthalten, die mit `#` beginnen. Nicht-Kommentarzeilen müssen eines dieser Formate haben:

```text
zone_abbreviation offset
zone_abbreviation offset D
zone_abbreviation time_zone_name
@INCLUDE file_name
@OVERRIDE
```

`zone_abbreviation` ist einfach die zu definierende Abkürzung. `offset` ist eine Ganzzahl, die den entsprechenden Offset in Sekunden von UTC angibt; positiv bedeutet östlich von Greenwich, negativ westlich. Zum Beispiel steht `-18000` für fünf Stunden westlich von Greenwich, also nordamerikanische Ostküsten-Standardzeit. `D` zeigt an, dass der Zonenname lokale Sommerzeit statt Standardzeit repräsentiert.

Alternativ kann ein `time_zone_name` angegeben werden, der einen Zonennamen aus der IANA-Zeitzonendatenbank referenziert. Die Definition der Zone wird herangezogen, um zu prüfen, ob die Abkürzung in dieser Zone verwendet wird oder wurde; falls ja, wird die passende Bedeutung verwendet, also die Bedeutung, die zum zu bestimmenden Zeitstempel aktuell verwendet wurde, oder die unmittelbar davor verwendete Bedeutung, wenn sie zu diesem Zeitpunkt nicht aktuell war, oder die älteste Bedeutung, wenn sie erst nach diesem Zeitpunkt verwendet wurde. Dieses Verhalten ist wesentlich für Abkürzungen, deren Bedeutung sich historisch geändert hat. Es ist auch erlaubt, eine Abkürzung über einen Zonennamen zu definieren, in dem diese Abkürzung nicht vorkommt; dann ist die Verwendung der Abkürzung einfach äquivalent zum Ausschreiben des Zonennamens.

> **Tipp:** Beim Definieren einer Abkürzung, deren Offset von UTC sich nie geändert hat, ist ein einfacher ganzzahliger Offset vorzuziehen, weil solche Abkürzungen deutlich billiger zu verarbeiten sind als solche, die das Nachschlagen einer Zeitzonendefinition erfordern.

Die Syntax `@INCLUDE` erlaubt das Einbinden einer anderen Datei im Verzeichnis `.../share/timezonesets/`. Einbindungen können bis zu einer begrenzten Tiefe verschachtelt werden.

Die Syntax `@OVERRIDE` zeigt an, dass nachfolgende Einträge in der Datei frühere Einträge überschreiben können, typischerweise Einträge aus eingebundenen Dateien. Ohne dies gelten widersprüchliche Definitionen derselben Zeitzonenabkürzung als Fehler.

In einer unveränderten Installation enthält die Datei `Default` alle konfliktfreien Zeitzonenabkürzungen für den größten Teil der Welt. Zusätzliche Dateien `Australia` und `India` werden für diese Regionen bereitgestellt; diese Dateien binden zuerst die Datei `Default` ein und fügen dann Abkürzungen nach Bedarf hinzu oder ändern sie.

Zu Referenzzwecken enthält eine Standardinstallation außerdem Dateien wie `Africa.txt`, `America.txt` usw., die Informationen über jede Zeitzonenabkürzung enthalten, die laut IANA-Zeitzonendatenbank verwendet wird. Die in diesen Dateien gefundenen Zonennamendefinitionen können bei Bedarf in eine eigene Konfigurationsdatei kopiert werden. Beachten Sie, dass diese Dateien wegen des Punkts in ihren Namen nicht direkt als Einstellungen für `timezone_abbreviations` referenziert werden können.

> **Hinweis:** Wenn beim Lesen des Zeitzonenabkürzungssatzes ein Fehler auftritt, wird kein neuer Wert angewendet und der alte Satz bleibt erhalten. Tritt der Fehler beim Starten der Datenbank auf, schlägt der Start fehl.

> **Achtung:** In der Konfigurationsdatei definierte Zeitzonenabkürzungen überschreiben Nicht-Zeitzonen-Bedeutungen, die in PostgreSQL eingebaut sind. Zum Beispiel definiert die Konfigurationsdatei `Australia` `SAT` für South Australian Standard Time. Wenn diese Datei aktiv ist, wird `SAT` nicht als Abkürzung für Saturday erkannt.

> **Achtung:** Wenn Sie Dateien in `.../share/timezonesets/` ändern, liegt es an Ihnen, Sicherungskopien anzulegen; ein normaler Datenbank-Dump enthält dieses Verzeichnis nicht.

### B.5. POSIX-Zeitzonenspezifikationen

PostgreSQL kann Zeitzonenspezifikationen akzeptieren, die nach den POSIX-Regeln für die Umgebungsvariable `TZ` geschrieben sind. POSIX-Zeitzonenspezifikationen sind ungeeignet, um die Komplexität realer Zeitzonengeschichte abzubilden, aber es gibt gelegentlich Gründe, sie zu verwenden.

Eine POSIX-Zeitzonenspezifikation hat die Form:

```text
STD offset [ DST [ dstoffset ] [ , rule ] ]
```

Zur besseren Lesbarkeit werden hier Leerzeichen zwischen den Feldern gezeigt, in der Praxis sollten jedoch keine Leerzeichen verwendet werden. Die Felder sind:

- `STD`
  - Die Zonenabkürzung, die für Standardzeit verwendet wird.

- `offset`
  - Der Standardzeit-Offset der Zone von UTC.

- `DST`
  - Die Zonenabkürzung, die für Sommerzeit verwendet wird. Wenn dieses Feld und die folgenden Felder weggelassen werden, verwendet die Zone einen festen UTC-Offset ohne Sommerzeitregel.

- `dstoffset`
  - Der Sommerzeit-Offset von UTC. Dieses Feld wird typischerweise weggelassen, weil der Standardwert eine Stunde weniger als der Standardzeit-Offset ist, was normalerweise richtig ist.

- `rule`
  - Definiert die Regel dafür, wann Sommerzeit gilt, wie unten beschrieben.

In dieser Syntax kann eine Zonenabkürzung eine Zeichenkette aus Buchstaben sein, etwa `EST`, oder eine beliebige Zeichenkette in spitzen Klammern, etwa `<UTC-05>`. Beachten Sie, dass die hier angegebenen Zonenabkürzungen nur für die Ausgabe verwendet werden, und auch dann nur in einigen Zeitstempel-Ausgabeformaten. Die in Zeitstempel-Eingaben erkannten Zonenabkürzungen werden wie in [Abschnitt B.4](B_Datum_Uhrzeit_Unterstützung.md#b4-datumuhrzeitkonfigurationsdateien) erklärt bestimmt.

Die Offset-Felder geben die Differenz zu UTC in Stunden und optional Minuten und Sekunden an. Sie haben das Format `hh[:mm[:ss]]`, optional mit einem führenden Vorzeichen (`+` oder `-`). Das positive Vorzeichen wird für Zonen westlich von Greenwich verwendet. Beachten Sie, dass dies die entgegengesetzte Vorzeichenkonvention zur ISO-8601-Konvention ist, die PostgreSQL an anderer Stelle verwendet. `hh` kann eine oder zwei Ziffern haben; `mm` und `ss` müssen, falls verwendet, zwei Ziffern haben.

Die Sommerzeit-Übergangsregel hat das Format:

```text
dstdate [ / dsttime ] , stddate [ / stdtime ]
```

Auch hier sollten in der Praxis keine Leerzeichen enthalten sein. Die Felder `dstdate` und `dsttime` definieren, wann die Sommerzeit beginnt, während `stddate` und `stdtime` definieren, wann die Standardzeit beginnt. In manchen Fällen, insbesondere in Zonen südlich des Äquators, kann Ersteres später im Jahr liegen als Letzteres. Die Datumsfelder haben eines dieser Formate:

- `n`
  - Eine einfache Ganzzahl bezeichnet einen Tag des Jahres, gezählt von null bis 364 oder in Schaltjahren bis 365.

- `Jn`
  - In dieser Form zählt `n` von 1 bis 365, und der 29. Februar wird nicht mitgezählt, selbst wenn er vorhanden ist. Ein Übergang am 29. Februar könnte auf diese Weise also nicht angegeben werden. Tage nach Februar haben jedoch dieselben Nummern, egal ob es ein Schaltjahr ist oder nicht; deshalb ist diese Form für Übergänge an festen Daten gewöhnlich nützlicher als die einfache Ganzzahlform.

- `Mm.n.d`
  - Diese Form gibt einen Übergang an, der immer im selben Monat und am selben Wochentag stattfindet. `m` bezeichnet den Monat von 1 bis 12. `n` gibt das n-te Vorkommen des durch `d` bezeichneten Wochentags an. `n` ist eine Zahl zwischen 1 und 4 oder 5 für das letzte Vorkommen dieses Wochentags im Monat, das das vierte oder fünfte sein kann. `d` ist eine Zahl zwischen 0 und 6, wobei 0 Sonntag bedeutet. Zum Beispiel bedeutet `M3.2.0` „der zweite Sonntag im März“.

> **Hinweis:** Das `M`-Format reicht aus, um viele verbreitete Sommerzeitregelungen zu beschreiben. Beachten Sie jedoch, dass keine dieser Varianten Änderungen an Sommerzeitgesetzen abbilden kann. In der Praxis sind daher die historischen Daten für benannte Zeitzonen in der IANA-Zeitzonendatenbank nötig, um vergangene Zeitstempel korrekt zu interpretieren.

Die Zeitfelder in einer Übergangsregel haben dasselbe Format wie die zuvor beschriebenen Offset-Felder, dürfen jedoch keine Vorzeichen enthalten. Sie definieren die aktuelle lokale Zeit, zu der der Wechsel zur anderen Zeit erfolgt. Wenn sie weggelassen werden, ist der Standardwert `02:00:00`.

Wenn eine Sommerzeitabkürzung angegeben ist, das Feld für die Übergangsregel aber weggelassen wird, wird als Fallback die Regel `M3.2.0,M11.1.0` verwendet. Sie entspricht der US-Praxis im Jahr 2020: Vorwärtssprung am zweiten Sonntag im März, Rückwärtssprung am ersten Sonntag im November, beide Übergänge um 2 Uhr der jeweils geltenden Zeit. Diese Regel liefert keine korrekten US-Übergangsdaten für Jahre vor 2007.

Als Beispiel beschreibt `CET-1CEST,M3.5.0,M10.5.0/3` die aktuelle, Stand 2020, Zeitregelung in Paris. Diese Spezifikation sagt, dass die Standardzeit die Abkürzung `CET` hat und eine Stunde vor UTC liegt; Sommerzeit hat die Abkürzung `CEST` und liegt implizit zwei Stunden vor UTC; Sommerzeit beginnt am letzten Sonntag im März um 2 Uhr CET und endet am letzten Sonntag im Oktober um 3 Uhr CEST.

Die vier Zeitzonennamen `EST5EDT`, `CST6CDT`, `MST7MDT` und `PST8PDT` sehen wie POSIX-Zonenspezifikationen aus. Tatsächlich werden sie jedoch als benannte Zeitzonen behandelt, weil es aus historischen Gründen Dateien mit diesen Namen in der IANA-Zeitzonendatenbank gibt. Die praktische Konsequenz ist, dass diese Zonennamen gültige historische US-Sommerzeitübergänge liefern, auch wenn eine einfache POSIX-Spezifikation das nicht täte.

Man sollte vorsichtig sein: Eine POSIX-artige Zeitzonenspezifikation lässt sich leicht falsch schreiben, weil nicht geprüft wird, ob die Zonenabkürzung oder Abkürzungen plausibel sind. Zum Beispiel funktioniert `SET TIMEZONE TO FOOBAR0` und lässt das System effektiv eine ziemlich eigenartige Abkürzung für UTC verwenden.

### B.6. Geschichte der Einheiten

Der SQL-Standard sagt, dass innerhalb der Definition eines „datetime literal“ die „datetime values“ durch die natürlichen Regeln für Daten und Zeiten gemäß dem gregorianischen Kalender beschränkt sind. PostgreSQL folgt dem SQL-Standard, indem es Daten ausschließlich im gregorianischen Kalender zählt, auch für Jahre, bevor dieser Kalender verwendet wurde. Diese Regel ist als proleptischer gregorianischer Kalender bekannt.

Der julianische Kalender wurde 45 v. Chr. von Julius Caesar eingeführt. In der westlichen Welt war er bis 1582 allgemein in Gebrauch, als Länder begannen, zum gregorianischen Kalender zu wechseln. Im julianischen Kalender wird das tropische Jahr als 365 1/4 Tage, also 365,25 Tage, angenähert. Das ergibt einen Fehler von ungefähr einem Tag in 128 Jahren.

Der anwachsende Kalenderfehler veranlasste Papst Gregor XIII., den Kalender gemäß den Anweisungen des Konzils von Trient zu reformieren. Im gregorianischen Kalender wird das tropische Jahr als `365 + 97 / 400` Tage, also 365,2425 Tage, angenähert. Dadurch dauert es ungefähr 3300 Jahre, bis sich das tropische Jahr gegenüber dem gregorianischen Kalender um einen Tag verschiebt.

Die Näherung `365 + 97 / 400` wird erreicht, indem es in jeweils 400 Jahren 97 Schaltjahre gibt, nach folgenden Regeln:

- Jedes durch 4 teilbare Jahr ist ein Schaltjahr.
- Jedes durch 100 teilbare Jahr ist jedoch kein Schaltjahr.
- Jedes durch 400 teilbare Jahr ist jedoch trotzdem ein Schaltjahr.

Die Jahre 1700, 1800, 1900, 2100 und 2200 sind also keine Schaltjahre. Dagegen sind 1600, 2000 und 2400 Schaltjahre. Im älteren julianischen Kalender sind dagegen alle durch 4 teilbaren Jahre Schaltjahre.

Die päpstliche Bulle vom Februar 1582 ordnete an, dass im Oktober 1582 zehn Tage ausgelassen werden sollten, sodass auf den 4. Oktober unmittelbar der 15. Oktober folgen sollte. Das wurde in Italien, Polen, Portugal und Spanien umgesetzt. Andere katholische Länder folgten kurz darauf, protestantische Länder waren zögerlicher, und die griechisch-orthodoxen Länder wechselten erst zu Beginn des 20. Jahrhunderts. Großbritannien und seine Herrschaftsgebiete, einschließlich der heutigen USA, vollzogen die Reform 1752. Daher folgte auf den 2. September 1752 der 14. September 1752. Deshalb erzeugen Unix-Systeme mit dem Programm `cal` Folgendes:

```text
$ cal 9 1752
   September 1752
 S  M Tu  W Th  F  S
       1  2 14 15 16
17 18 19 20 21 22 23
24 25 26 27 28 29 30
```

Dieser Kalender gilt natürlich nur für Großbritannien und seine Herrschaftsgebiete, nicht für andere Orte. Da es schwierig und verwirrend wäre, die tatsächlich an verschiedenen Orten zu verschiedenen Zeiten verwendeten Kalender zu verfolgen, versucht PostgreSQL das nicht, sondern folgt für alle Daten den Regeln des gregorianischen Kalenders, auch wenn diese Methode historisch nicht genau ist.

In verschiedenen Teilen der Welt wurden unterschiedliche Kalender entwickelt, viele davon älter als das gregorianische System. Die Anfänge des chinesischen Kalenders lassen sich zum Beispiel bis ins 14. Jahrhundert v. Chr. zurückverfolgen. Der Legende nach erfand Kaiser Huangdi diesen Kalender im Jahr 2637 v. Chr. Die Volksrepublik China verwendet den gregorianischen Kalender für zivile Zwecke. Der chinesische Kalender wird zur Bestimmung von Festen verwendet.

### B.7. Julianische Daten

Das System der Julianischen Daten ist eine Methode zur Nummerierung von Tagen. Es ist nicht mit dem julianischen Kalender verwandt, auch wenn es verwirrenderweise ähnlich heißt. Das System der Julianischen Daten wurde vom französischen Gelehrten Joseph Justus Scaliger (1540–1609) erfunden und ist vermutlich nach Scaligers Vater, dem italienischen Gelehrten Julius Caesar Scaliger (1484–1558), benannt.

Im System der Julianischen Daten hat jeder Tag eine fortlaufende Nummer, beginnend mit JD 0, das manchmal als Julianisches Datum bezeichnet wird. JD 0 entspricht dem 1. Januar 4713 v. Chr. im julianischen Kalender oder dem 24. November 4714 v. Chr. im gregorianischen Kalender. Die Zählung Julianischer Daten wird am häufigsten von Astronomen verwendet, um ihre nächtlichen Beobachtungen zu bezeichnen; ein Datum läuft daher von Mittag UTC bis zum nächsten Mittag UTC, nicht von Mitternacht bis Mitternacht. JD 0 bezeichnet die 24 Stunden von Mittag UTC am 24. November 4714 v. Chr. bis Mittag UTC am 25. November 4714 v. Chr.

Obwohl PostgreSQL die Schreibweise Julianischer Daten für Eingabe und Ausgabe von Daten unterstützt und Julianische Daten auch für einige interne Datum-/Uhrzeitberechnungen verwendet, übernimmt es nicht die Feinheit, Daten von Mittag bis Mittag laufen zu lassen. PostgreSQL behandelt ein Julianisches Datum wie ein normales Datum als von lokaler Mitternacht bis lokaler Mitternacht laufend.

Diese Definition bietet jedoch eine Möglichkeit, die astronomische Definition zu erhalten, wenn sie benötigt wird: Führen Sie die Arithmetik in der Zeitzone `UTC+12` aus. Zum Beispiel:

```sql
SELECT extract(julian from '2021-06-23 7:00:00-04'::timestamptz
       at time zone 'UTC+12');
```

```text
           extract
------------------------------
 2459388.95833333333333333333
(1 row)
```

```sql
SELECT extract(julian from '2021-06-23 8:00:00-04'::timestamptz
       at time zone 'UTC+12');
```

```text
               extract
--------------------------------------
 2459389.0000000000000000000000000000
(1 row)
```

```sql
SELECT extract(julian from date '2021-06-23');
```

```text
 extract
---------
(1 row)
```
