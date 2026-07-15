# 4. SQL-Syntax

Dieses Kapitel beschreibt die Syntax von SQL. Es bildet die Grundlage für das Verständnis der folgenden Kapitel, die im Detail zeigen, wie SQL-Befehle angewendet werden, um Daten zu definieren und zu ändern.

Auch Benutzern, die bereits mit SQL vertraut sind, empfehlen wir, dieses Kapitel sorgfältig zu lesen, da es mehrere Regeln und Konzepte enthält, die in SQL-Datenbanken uneinheitlich implementiert sind oder speziell für PostgreSQL gelten.

## 4.1. Lexikalische Struktur

SQL-Eingabe besteht aus einer Folge von Befehlen. Ein Befehl setzt sich aus einer Folge von Tokens zusammen und wird durch ein Semikolon (`;`) beendet. Auch das Ende des Eingabestroms beendet einen Befehl. Welche Tokens gültig sind, hängt von der Syntax des jeweiligen Befehls ab.

Ein Token kann ein Schlüsselwort, ein Bezeichner, ein quoted identifier, ein Literal (oder eine Konstante) oder ein Sonderzeichen sein. Tokens werden normalerweise durch Leerraum getrennt (Leerzeichen, Tabulator, Zeilenumbruch), müssen es aber nicht, wenn keine Mehrdeutigkeit entsteht. Das ist im Allgemeinen nur der Fall, wenn ein Sonderzeichen an einen anderen Tokentyp grenzt.

Das Folgende ist beispielsweise syntaktisch gültige SQL-Eingabe:

```sql
SELECT * FROM MY_TABLE;
UPDATE MY_TABLE SET A = 5;
INSERT INTO MY_TABLE VALUES (3, 'hi there');
```

Dies ist eine Folge von drei Befehlen, einer pro Zeile. Das ist allerdings nicht erforderlich; mehr als ein Befehl kann in einer Zeile stehen, und Befehle können sinnvoll über mehrere Zeilen verteilt werden.

Zusätzlich können in SQL-Eingaben Kommentare vorkommen. Sie sind keine Tokens, sondern entsprechen effektiv Leerraum.

Die SQL-Syntax ist nicht sehr konsistent darin, welche Tokens Befehle kennzeichnen und welche Operanden oder Parameter sind. Die ersten Tokens sind im Allgemeinen der Befehlsname; im obigen Beispiel würden wir daher gewöhnlich von einem `SELECT`-, einem `UPDATE`- und einem `INSERT`-Befehl sprechen. Der Befehl `UPDATE` verlangt aber beispielsweise immer, dass an einer bestimmten Position ein `SET`-Token erscheint, und diese Variante von `INSERT` benötigt außerdem `VALUES`, um vollständig zu sein. Die genauen Syntaxregeln für jeden Befehl werden in Teil VI beschrieben.

### 4.1.1. Bezeichner und Schlüsselwörter

Tokens wie `SELECT`, `UPDATE` oder `VALUES` im obigen Beispiel sind Schlüsselwörter, also Wörter mit fester Bedeutung in der SQL-Sprache. Die Tokens `MY_TABLE` und `A` sind Beispiele für Bezeichner. Sie bezeichnen je nach verwendetem Befehl Namen von Tabellen, Spalten oder anderen Datenbankobjekten. Daher werden sie manchmal einfach "Namen" genannt. Schlüsselwörter und Bezeichner haben dieselbe lexikalische Struktur; ohne Kenntnis der Sprache lässt sich also nicht feststellen, ob ein Token ein Bezeichner oder ein Schlüsselwort ist. Eine vollständige Liste der Schlüsselwörter findet sich in Anhang C.

SQL-Bezeichner und Schlüsselwörter müssen mit einem Buchstaben (`a-z`, aber auch Buchstaben mit diakritischen Zeichen und nichtlateinische Buchstaben) oder einem Unterstrich (`_`) beginnen. Nachfolgende Zeichen in einem Bezeichner oder Schlüsselwort können Buchstaben, Unterstriche, Ziffern (`0-9`) oder Dollarzeichen (`$`) sein. Beachten Sie, dass Dollarzeichen nach dem Wortlaut des SQL-Standards in Bezeichnern nicht erlaubt sind; ihre Verwendung kann Anwendungen also weniger portabel machen. Der SQL-Standard wird kein Schlüsselwort definieren, das Ziffern enthält oder mit einem Unterstrich beginnt oder endet; Bezeichner dieser Form sind daher vor möglichen Konflikten mit zukünftigen Erweiterungen des Standards sicher.

Das System verwendet höchstens `NAMEDATALEN - 1` Bytes eines Bezeichners; längere Namen können in Befehlen geschrieben werden, werden aber abgeschnitten. Standardmäßig ist `NAMEDATALEN` 64, sodass die maximale Bezeichnerlänge 63 Byte beträgt. Wenn diese Grenze problematisch ist, kann sie durch Ändern der Konstante `NAMEDATALEN` in `src/include/pg_config_manual.h` erhöht werden.

Schlüsselwörter und nicht gequotete Bezeichner unterscheiden nicht zwischen Groß- und Kleinschreibung. Daher kann

```sql
UPDATE MY_TABLE SET A = 5;
```

gleichwertig geschrieben werden als:

```sql
uPDaTE my_TabLE SeT a = 5;
```

Eine häufig verwendete Konvention ist, Schlüsselwörter in Großbuchstaben und Namen in Kleinbuchstaben zu schreiben, zum Beispiel:

```sql
UPDATE my_table SET a = 5;
```

Es gibt eine zweite Art von Bezeichner: den delimited identifier oder quoted identifier. Er wird gebildet, indem eine beliebige Zeichenfolge in doppelte Anführungszeichen (`"`) eingeschlossen wird. Ein solcher Bezeichner ist immer ein Bezeichner und nie ein Schlüsselwort. Daher könnte `"select"` verwendet werden, um auf eine Spalte oder Tabelle namens `select` zu verweisen, während ein nicht gequotetes `select` als Schlüsselwort interpretiert würde und daher dort einen Parse-Fehler auslöst, wo ein Tabellen- oder Spaltenname erwartet wird. Das Beispiel kann mit gequoteten Bezeichnern so geschrieben werden:

```sql
UPDATE "my_table" SET "a" = 5;
```

Gequotete Bezeichner können jedes Zeichen außer dem Zeichen mit Code null enthalten. Um ein doppeltes Anführungszeichen einzuschließen, schreiben Sie zwei doppelte Anführungszeichen. Dadurch lassen sich Tabellen- oder Spaltennamen konstruieren, die sonst nicht möglich wären, etwa Namen mit Leerzeichen oder Ampersands. Die Längenbeschränkung gilt weiterhin.

Das Quoten eines Bezeichners macht ihn außerdem groß-/kleinschreibungssensitiv, während nicht gequotete Namen immer in Kleinbuchstaben gefaltet werden. Beispielsweise gelten die Bezeichner `FOO`, `foo` und `"foo"` in PostgreSQL als gleich, während `"Foo"` und `"FOO"` von diesen drei und voneinander verschieden sind. Das Falten nicht gequoteter Namen in Kleinbuchstaben ist in PostgreSQL nicht kompatibel mit dem SQL-Standard, der verlangt, dass nicht gequotete Namen in Großbuchstaben gefaltet werden. Nach dem Standard sollte also `foo` äquivalent zu `"FOO"` sein, nicht zu `"foo"`. Wenn Sie portable Anwendungen schreiben möchten, sollten Sie einen bestimmten Namen entweder immer quoten oder nie quoten.

Eine Variante gequoteter Bezeichner erlaubt, escapte Unicode-Zeichen einzuschließen, die durch ihre Codepunkte identifiziert werden. Diese Variante beginnt mit `U&` (großes oder kleines `U`, gefolgt von einem Ampersand) unmittelbar vor dem öffnenden doppelten Anführungszeichen, ohne Leerzeichen dazwischen, zum Beispiel `U&"foo"`. Beachten Sie, dass dies eine Mehrdeutigkeit mit dem Operator `&` erzeugt; verwenden Sie Leerzeichen um den Operator, um dieses Problem zu vermeiden. Innerhalb der Anführungszeichen können Unicode-Zeichen in escapeter Form angegeben werden, indem ein Backslash gefolgt von der vierstelligen hexadezimalen Codepunktnummer geschrieben wird, oder alternativ ein Backslash gefolgt von einem Pluszeichen und einer sechsstelligen hexadezimalen Codepunktnummer. Der Bezeichner `"data"` könnte zum Beispiel so geschrieben werden:

```sql
U&"d\0061t\+000061"
```

Das folgende weniger triviale Beispiel schreibt das russische Wort »slon« (Elefant) in kyrillischen Buchstaben:

```sql
U&"\0441\043B\043E\043D"
```

Wenn ein anderes Escape-Zeichen als der Backslash gewünscht ist, kann es mit der `UESCAPE`-Klausel nach der Zeichenkette angegeben werden, zum Beispiel:

```sql
U&"d!0061t!+000061" UESCAPE '!'
```

Das Escape-Zeichen kann ein beliebiges einzelnes Zeichen sein, außer einer hexadezimalen Ziffer, dem Pluszeichen, einem einfachen Anführungszeichen, einem doppelten Anführungszeichen oder einem Leerraumzeichen. Beachten Sie, dass das Escape-Zeichen nach `UESCAPE` in einfachen Anführungszeichen geschrieben wird, nicht in doppelten.

Um das Escape-Zeichen wörtlich in den Bezeichner aufzunehmen, schreiben Sie es zweimal.

Sowohl die vierstellige als auch die sechsstellige Escape-Form kann verwendet werden, um UTF-16-Surrogatpaare anzugeben, aus denen Zeichen mit Codepunkten größer als `U+FFFF` zusammengesetzt werden, obwohl die Verfügbarkeit der sechsstelligen Form dies technisch unnötig macht. Surrogatpaare werden nicht direkt gespeichert, sondern zu einem einzelnen Codepunkt zusammengeführt.

Wenn die Serverkodierung nicht UTF-8 ist, wird der durch eine dieser Escape-Sequenzen identifizierte Unicode-Codepunkt in die tatsächliche Serverkodierung umgewandelt; wenn das nicht möglich ist, wird ein Fehler gemeldet.

### 4.1.2. Konstanten

In PostgreSQL gibt es drei Arten implizit typisierter Konstanten: Zeichenketten, Bit-Strings und Zahlen. Konstanten können auch mit expliziten Typen angegeben werden, was eine genauere Darstellung und eine effizientere Verarbeitung durch das System ermöglichen kann. Diese Alternativen werden in den folgenden Unterabschnitten besprochen.

#### 4.1.2.1. Zeichenkettenkonstanten

Eine Zeichenkettenkonstante in SQL ist eine beliebige Folge von Zeichen, die durch einfache Anführungszeichen (`'`) begrenzt wird, zum Beispiel `'This is a string'`. Um ein einfaches Anführungszeichen innerhalb einer Zeichenkettenkonstante aufzunehmen, schreiben Sie zwei aufeinanderfolgende einfache Anführungszeichen, zum Beispiel `'Dianne''s horse'`. Beachten Sie, dass dies nicht dasselbe ist wie ein doppeltes Anführungszeichen (`"`).

Zwei Zeichenkettenkonstanten, die nur durch Leerraum mit mindestens einem Zeilenumbruch getrennt sind, werden verkettet und effektiv so behandelt, als wäre die Zeichenkette als eine Konstante geschrieben worden. Zum Beispiel:

```sql
SELECT 'foo'
'bar';
```

ist äquivalent zu:

```sql
SELECT 'foobar';
```

aber:

```sql
SELECT 'foo'                'bar';
```

ist keine gültige Syntax. Dieses etwas eigenartige Verhalten ist von SQL festgelegt; PostgreSQL folgt hier dem Standard.

#### 4.1.2.2. Zeichenkettenkonstanten mit C-artigen Escapes

PostgreSQL akzeptiert außerdem »Escape«-Zeichenkettenkonstanten, die eine Erweiterung des SQL-Standards sind. Eine Escape-Zeichenkettenkonstante wird angegeben, indem der Buchstabe `E` (groß oder klein) unmittelbar vor das öffnende einfache Anführungszeichen geschrieben wird, zum Beispiel `E'foo'`. Wenn eine Escape-Zeichenkettenkonstante über mehrere Zeilen fortgesetzt wird, schreiben Sie `E` nur vor das erste öffnende Anführungszeichen. Innerhalb einer Escape-Zeichenkette beginnt ein Backslash-Zeichen (`\`) eine C-artige Backslash-Escape-Sequenz, bei der die Kombination aus Backslash und nachfolgenden Zeichen einen speziellen Bytewert darstellt, wie in Tabelle 4.1 gezeigt.

**Tabelle 4.1. Backslash-Escape-Sequenzen**

| Backslash-Escape-Sequenz | Interpretation |
|---|---|
| `\b` | Rückschritt |
| `\f` | Seitenvorschub |
| `\n` | Zeilenumbruch |
| `\r` | Wagenrücklauf |
| `\t` | Tabulator |
| `\o`, `\oo`, `\ooo` (`o = 0-7`) | Oktaler Bytewert |
| `\xh`, `\xhh` (`h = 0-9`, `A-F`) | Hexadezimaler Bytewert |
| `\uxxxx`, `\Uxxxxxxxx` (`x = 0-9`, `A-F`) | 16- oder 32-Bit-hexadezimales Unicode-Zeichen |

Jedes andere Zeichen nach einem Backslash wird wörtlich genommen. Um also ein Backslash-Zeichen einzuschließen, schreiben Sie zwei Backslashes (`\\`). Außerdem kann ein einfaches Anführungszeichen in einer Escape-Zeichenkette durch `\'` eingeschlossen werden, zusätzlich zur normalen Schreibweise `''`.

Es liegt in Ihrer Verantwortung, dass die von Ihnen erzeugten Bytefolgen, insbesondere bei Verwendung oktaler oder hexadezimaler Escapes, gültige Zeichen in der Zeichensatzkodierung des Servers ergeben. Eine nützliche Alternative ist die Verwendung von Unicode-Escapes oder der alternativen Unicode-Escape-Syntax, die in [Abschnitt 4.1.2.3](#4123-zeichenkettenkonstanten-mit-unicodeescapes) erklärt wird; dann prüft der Server, ob die Zeichenumwandlung möglich ist.

> Achtung: Wenn der Konfigurationsparameter `standard_conforming_strings` ausgeschaltet ist, erkennt PostgreSQL Backslash-Escapes sowohl in regulären als auch in Escape-Zeichenkettenkonstanten. Seit PostgreSQL 9.1 ist die Vorgabe jedoch eingeschaltet, was bedeutet, dass Backslash-Escapes nur in Escape-Zeichenkettenkonstanten erkannt werden. Dieses Verhalten entspricht stärker dem Standard, kann aber Anwendungen beschädigen, die sich auf das historische Verhalten verlassen, bei dem Backslash-Escapes immer erkannt wurden. Als Workaround können Sie diesen Parameter ausschalten; besser ist es jedoch, die Verwendung von Backslash-Escapes zu vermeiden. Wenn Sie einen Backslash-Escape benötigen, um ein Sonderzeichen darzustellen, schreiben Sie die Zeichenkettenkonstante mit einem `E`.
>
> Zusätzlich zu `standard_conforming_strings` steuern die Konfigurationsparameter `escape_string_warning` und `backslash_quote` die Behandlung von Backslashes in Zeichenkettenkonstanten.

Das Zeichen mit dem Code null kann nicht in einer Zeichenkettenkonstante vorkommen.

#### 4.1.2.3. Zeichenkettenkonstanten mit Unicode-Escapes

PostgreSQL unterstützt außerdem eine weitere Escape-Syntax für Zeichenketten, mit der beliebige Unicode-Zeichen über ihren Codepunkt angegeben werden können. Eine Unicode-Escape-Zeichenkettenkonstante beginnt mit `U&` (großes oder kleines `U`, gefolgt von einem Ampersand) unmittelbar vor dem öffnenden Anführungszeichen, ohne Leerzeichen dazwischen, zum Beispiel `U&'foo'`. Beachten Sie, dass dies eine Mehrdeutigkeit mit dem Operator `&` erzeugt; verwenden Sie Leerzeichen um den Operator, um dieses Problem zu vermeiden. Innerhalb der Anführungszeichen können Unicode-Zeichen in escapeter Form angegeben werden, indem ein Backslash gefolgt von der vierstelligen hexadezimalen Codepunktnummer geschrieben wird, oder alternativ ein Backslash gefolgt von einem Pluszeichen und einer sechsstelligen hexadezimalen Codepunktnummer. Die Zeichenkette `'data'` könnte zum Beispiel so geschrieben werden:

```sql
U&'d\0061t\+000061'
```

Das folgende weniger triviale Beispiel schreibt das russische Wort »slon« (Elefant) in kyrillischen Buchstaben:


```sql
U&'\0441\043B\043E\043D'
```

Wenn ein anderes Escape-Zeichen als der Backslash gewünscht ist, kann es mit der `UESCAPE`-Klausel nach der Zeichenkette angegeben werden, zum Beispiel:

```sql
U&'d!0061t!+000061' UESCAPE '!'
```

Das Escape-Zeichen kann ein beliebiges einzelnes Zeichen sein, außer einer hexadezimalen Ziffer, dem Pluszeichen, einem einfachen Anführungszeichen, einem doppelten Anführungszeichen oder einem Leerraumzeichen.

Um das Escape-Zeichen wörtlich in die Zeichenkette aufzunehmen, schreiben Sie es zweimal.

Sowohl die vierstellige als auch die sechsstellige Escape-Form kann verwendet werden, um UTF-16-Surrogatpaare anzugeben, aus denen Zeichen mit Codepunkten größer als `U+FFFF` zusammengesetzt werden, obwohl die Verfügbarkeit der sechsstelligen Form dies technisch unnötig macht. Surrogatpaare werden nicht direkt gespeichert, sondern zu einem einzelnen Codepunkt zusammengeführt.

Wenn die Serverkodierung nicht UTF-8 ist, wird der durch eine dieser Escape-Sequenzen identifizierte Unicode-Codepunkt in die tatsächliche Serverkodierung umgewandelt; wenn das nicht möglich ist, wird ein Fehler gemeldet.

Außerdem funktioniert die Unicode-Escape-Syntax für Zeichenkettenkonstanten nur, wenn der Konfigurationsparameter `standard_conforming_strings` eingeschaltet ist. Andernfalls könnte diese Syntax Clients, die SQL-Anweisungen parsen, so verwirren, dass SQL-Injection und ähnliche Sicherheitsprobleme möglich würden. Wenn der Parameter ausgeschaltet ist, wird diese Syntax mit einer Fehlermeldung zurückgewiesen.

#### 4.1.2.4. Dollar-Quoted-Zeichenkettenkonstanten

Während die Standardsyntax zur Angabe von Zeichenkettenkonstanten normalerweise bequem ist, kann sie schwer verständlich werden, wenn die gewünschte Zeichenkette viele einfache Anführungszeichen enthält, da jedes davon verdoppelt werden muss. Um in solchen Situationen besser lesbare Abfragen zu ermöglichen, stellt PostgreSQL eine weitere Schreibweise namens Dollar-Quoting bereit. Eine dollar-gequotete Zeichenkettenkonstante besteht aus einem Dollarzeichen (`$`), einem optionalen »Tag« aus null oder mehr Zeichen, einem weiteren Dollarzeichen, einer beliebigen Folge von Zeichen als Zeichenketteninhalt, einem Dollarzeichen, demselben Tag, mit dem dieses Dollar-Quote begonnen hat, und einem abschließenden Dollarzeichen. Hier sind zum Beispiel zwei verschiedene Möglichkeiten, die Zeichenkette `Dianne's horse` mit Dollar-Quoting anzugeben:

```sql
$$Dianne's horse$$
$SomeTag$Dianne's horse$SomeTag$
```

Beachten Sie, dass innerhalb der dollar-gequoteten Zeichenkette einfache Anführungszeichen verwendet werden können, ohne escapet werden zu müssen. Tatsächlich werden innerhalb einer dollar-gequoteten Zeichenkette niemals Zeichen escapet: Der Zeichenketteninhalt wird immer wörtlich geschrieben. Backslashes sind nicht besonders, und Dollarzeichen ebenfalls nicht, außer sie sind Teil einer Zeichenfolge, die zum öffnenden Tag passt.

Es ist möglich, dollar-gequotete Zeichenkettenkonstanten zu verschachteln, indem auf jeder Verschachtelungsebene unterschiedliche Tags gewählt werden. Dies wird am häufigsten beim Schreiben von Funktionsdefinitionen verwendet. Zum Beispiel:

```sql
$function$
BEGIN
     RETURN ($1 ~ $q$[\t\r\n\v\\]$q$);
END;
$function$
```

Hier stellt die Sequenz `$q$[\t\r\n\v\\]$q$` eine dollar-gequotete Literalzeichenkette `[\t\r\n\v\\]` dar, die erkannt wird, wenn der Funktionskörper von PostgreSQL ausgeführt wird. Da die Sequenz aber nicht zum äußeren Dollar-Quoting-Begrenzer `$function$` passt, ist sie aus Sicht der äußeren Zeichenkette nur weiterer Inhalt der Konstante.


Der Tag einer dollar-gequoteten Zeichenkette, falls vorhanden, folgt denselben Regeln wie ein nicht gequoteter Bezeichner, außer dass er kein Dollarzeichen enthalten darf. Tags sind groß-/kleinschreibungssensitiv, daher ist `$tag$String content$tag$` korrekt, `$TAG$String content$tag$` jedoch nicht.

Eine dollar-gequotete Zeichenkette, die auf ein Schlüsselwort oder einen Bezeichner folgt, muss durch Leerraum davon getrennt werden; andernfalls würde der Dollar-Quoting-Begrenzer als Teil des vorangehenden Bezeichners gelesen.

Dollar-Quoting ist nicht Teil des SQL-Standards, aber oft eine bequemere Möglichkeit, komplizierte Zeichenkettenliterale zu schreiben als die standardkonforme Syntax mit einfachen Anführungszeichen. Es ist besonders nützlich, wenn Zeichenkettenkonstanten innerhalb anderer Konstanten dargestellt werden müssen, wie es in prozeduralen Funktionsdefinitionen häufig nötig ist. Mit der Syntax einfacher Anführungszeichen müsste jeder Backslash im obigen Beispiel als vier Backslashes geschrieben werden; diese würden beim Parsen der ursprünglichen Zeichenkettenkonstante auf zwei Backslashes reduziert und dann beim erneuten Parsen der inneren Zeichenkettenkonstante während der Funktionsausführung auf einen.

#### 4.1.2.5. Bit-String-Konstanten

Bit-String-Konstanten sehen wie reguläre Zeichenkettenkonstanten aus, besitzen aber unmittelbar vor dem öffnenden Anführungszeichen ein `B` (Groß- oder Kleinbuchstabe), ohne dazwischenliegenden Leerraum, zum Beispiel `B'1001'`. Innerhalb von Bit-String-Konstanten sind nur die Zeichen `0` und `1` erlaubt.

Alternativ können Bit-String-Konstanten in hexadezimaler Schreibweise angegeben werden, mit einem führenden `X` (Groß- oder Kleinbuchstabe), zum Beispiel `X'1FF'`. Diese Schreibweise entspricht einer Bit-String-Konstante mit vier Binärziffern für jede hexadezimale Ziffer.

Beide Formen von Bit-String-Konstanten können über Zeilen hinweg fortgesetzt werden, genauso wie reguläre Zeichenkettenkonstanten. Dollar-Quoting kann in einer Bit-String-Konstante nicht verwendet werden.

#### 4.1.2.6. Numerische Konstanten

Numerische Konstanten werden in diesen allgemeinen Formen akzeptiert:

```text
digits
digits.[digits][e[+-]digits]
[digits].digits[e[+-]digits]
digitse[+-]digits
```

Dabei steht `digits` für eine oder mehrere Dezimalziffern (`0` bis `9`). Wenn ein Dezimalpunkt verwendet wird, muss mindestens eine Ziffer davor oder danach stehen. Wenn ein Exponentenmarker (`e`) vorhanden ist, muss mindestens eine Ziffer darauf folgen. Innerhalb der Konstante dürfen keine Leerzeichen oder anderen Zeichen stehen, mit Ausnahme von Unterstrichen, die wie unten beschrieben zur visuellen Gruppierung verwendet werden können. Beachten Sie, dass ein führendes Plus- oder Minuszeichen nicht tatsächlich als Teil der Konstante betrachtet wird; es ist ein Operator, der auf die Konstante angewendet wird.

Dies sind Beispiele gültiger numerischer Konstanten:

```text
3.5
4.
.001
5e2
1.925e-3
```

Zusätzlich werden nichtdezimale Ganzzahlkonstanten in diesen Formen akzeptiert:

```text
0xhexdigits
0ooctdigits
0bbindigits
```

Dabei steht `hexdigits` für eine oder mehrere hexadezimale Ziffern (`0-9`, `A-F`), `octdigits` für eine oder mehrere oktale Ziffern (`0-7`) und `bindigits` für eine oder mehrere binäre Ziffern (`0` oder `1`). Hexadezimale Ziffern und Radix-Präfixe können groß- oder kleingeschrieben werden. Beachten Sie, dass nur Ganzzahlen nichtdezimale Formen haben können, nicht Zahlen mit Nachkommanteil.

Dies sind Beispiele gültiger nichtdezimaler Ganzzahlkonstanten:

```text
0b100101
0B10011001
0o273
0O755
0x42f
0XFFFF
```

Zur visuellen Gruppierung können Unterstriche zwischen Ziffern eingefügt werden. Sie haben keinen weiteren Einfluss auf den Wert der Konstante. Zum Beispiel:

```text
1_500_000_000
0b10001000_00000000
0o_1_755
0xFFFF_FFFF
1.618_034
```

Unterstriche sind am Anfang oder Ende einer numerischen Konstante oder einer Zifferngruppe nicht erlaubt, also unmittelbar vor oder nach dem Dezimalpunkt oder Exponentenmarker. Mehr als ein Unterstrich hintereinander ist ebenfalls nicht erlaubt.

Eine numerische Konstante, die weder Dezimalpunkt noch Exponent enthält, wird zunächst als Typ `integer` angenommen, wenn ihr Wert in `integer` (32 Bit) passt; andernfalls als Typ `bigint`, wenn ihr Wert in `bigint` (64 Bit) passt; andernfalls als Typ `numeric`. Konstanten, die Dezimalpunkte und/oder Exponenten enthalten, werden immer zunächst als Typ `numeric` angenommen.

Der zunächst zugewiesene Datentyp einer numerischen Konstante ist nur ein Ausgangspunkt für die Algorithmen zur Typauflösung. In den meisten Fällen wird die Konstante je nach Kontext automatisch in den passendsten Typ umgewandelt. Wenn nötig, können Sie durch einen Cast erzwingen, dass ein numerischer Wert als bestimmter Datentyp interpretiert wird. Zum Beispiel können Sie einen numerischen Wert als Typ `real` (`float4`) behandeln lassen, indem Sie schreiben:

```sql
REAL '1.23'         -- Zeichenkettenstil
1.23::REAL          -- historischer PostgreSQL-Stil
```

Dies sind eigentlich nur Spezialfälle der allgemeinen Cast-Notationen, die als Nächstes besprochen werden.

#### 4.1.2.7. Konstanten anderer Typen

Eine Konstante eines beliebigen Typs kann mit einer der folgenden Schreibweisen eingegeben werden:

```sql
type 'string'
'string'::type
CAST ( 'string' AS type )
```

Der Text der Zeichenkettenkonstante wird an die Eingabekonvertierungsroutine für den Typ `type` übergeben. Das Ergebnis ist eine Konstante des angegebenen Typs. Der explizite Typecast kann weggelassen werden, wenn keine Mehrdeutigkeit darüber besteht, welchen Typ die Konstante haben muss, etwa wenn sie direkt einer Tabellenspalte zugewiesen wird; in diesem Fall wird sie automatisch umgewandelt.

Die Zeichenkettenkonstante kann entweder in regulärer SQL-Notation oder mit Dollar-Quoting geschrieben werden.

Es ist auch möglich, eine Typumwandlung mit funktionsähnlicher Syntax anzugeben:

```sql
typename ( 'string' )
```

Nicht alle Typnamen können jedoch auf diese Weise verwendet werden; Details stehen in [Abschnitt 4.2.9](#429-typecasts).

Die Syntaxen `::`, `CAST()` und Funktionsaufruf können auch verwendet werden, um Laufzeit-Typumwandlungen beliebiger Ausdrücke anzugeben, wie in [Abschnitt 4.2.9](#429-typecasts) beschrieben. Um syntaktische Mehrdeutigkeit zu vermeiden, kann die Syntax `type 'string'` nur verwendet werden, um den Typ einer einfachen Literal-Konstante anzugeben. Eine weitere Einschränkung der Syntax `type 'string'` ist, dass sie nicht für Array-Typen funktioniert; verwenden Sie `::` oder `CAST()`, um den Typ einer Array-Konstante anzugeben.

Die Syntax `CAST()` entspricht SQL. Die Syntax `type 'string'` ist eine Verallgemeinerung des Standards: SQL spezifiziert diese Syntax nur für einige Datentypen, PostgreSQL erlaubt sie jedoch für alle Typen. Die Syntax mit `::` ist historischer PostgreSQL-Gebrauch, ebenso wie die Funktionsaufrufsyntax.

### 4.1.3. Operatoren

Ein Operatorname ist eine Folge von bis zu `NAMEDATALEN - 1` Zeichen (standardmäßig 63) aus der folgenden Liste:

```text
+-*/<>=~!@#%^&|`?
```

Es gibt jedoch einige Einschränkungen für Operatornamen:

- `--` und `/*` dürfen nirgendwo in einem Operatornamen vorkommen, da sie als Beginn eines Kommentars interpretiert würden.
- Ein mehrzeichniger Operatorname darf nicht mit `+` oder `-` enden, es sei denn, der Name enthält außerdem mindestens eines dieser Zeichen:

```text
~!@#%^&|`?
```

Zum Beispiel ist `@-` ein erlaubter Operatorname, `*-` dagegen nicht. Diese Einschränkung erlaubt PostgreSQL, SQL-konforme Abfragen zu parsen, ohne Leerzeichen zwischen Tokens zu verlangen.

Beim Arbeiten mit nicht standardkonformen SQL-Operatornamen müssen Sie benachbarte Operatoren gewöhnlich durch Leerzeichen trennen, um Mehrdeutigkeit zu vermeiden. Wenn Sie beispielsweise einen Präfixoperator namens `@` definiert haben, können Sie nicht `X*@Y` schreiben; Sie müssen `X* @Y` schreiben, damit PostgreSQL dies als zwei Operatornamen und nicht als einen liest.

### 4.1.4. Sonderzeichen

Einige nicht alphanumerische Zeichen haben eine besondere Bedeutung, die sich davon unterscheidet, Operator zu sein. Details zur Verwendung finden Sie dort, wo das jeweilige Syntaxelement beschrieben wird. Dieser Abschnitt soll nur auf ihre Existenz hinweisen und ihre Zwecke zusammenfassen.

- Ein Dollarzeichen (`$`) gefolgt von Ziffern wird verwendet, um im Rumpf einer Funktionsdefinition oder vorbereiteten Anweisung einen positionsbezogenen Parameter darzustellen. In anderen Kontexten kann das Dollarzeichen Teil eines Bezeichners oder einer Dollar-Quoted-Zeichenkettenkonstante sein.
- Klammern (`()`) haben ihre übliche Bedeutung, um Ausdrücke zu gruppieren und Präzedenz zu erzwingen. In einigen Fällen sind Klammern Teil der festen Syntax eines bestimmten SQL-Befehls und erforderlich.
- Eckige Klammern (`[]`) werden verwendet, um Elemente eines Arrays auszuwählen. Weitere Informationen zu Arrays finden Sie in [Abschnitt 8.15](08_Datentypen.md#815-arrays).
- Kommas (`,`) werden in einigen syntaktischen Konstrukten verwendet, um Elemente einer Liste zu trennen.
- Das Semikolon (`;`) beendet einen SQL-Befehl. Es darf innerhalb eines Befehls nirgendwo vorkommen, außer innerhalb einer Zeichenkettenkonstante oder eines gequoteten Bezeichners.
- Der Doppelpunkt (`:`) wird verwendet, um "Slices" aus Arrays auszuwählen. Siehe [Abschnitt 8.15](08_Datentypen.md#815-arrays). In bestimmten SQL-Dialekten, etwa Embedded SQL, wird der Doppelpunkt verwendet, um Variablennamen einzuleiten.
- Der Stern (`*`) wird in einigen Kontexten verwendet, um alle Felder einer Tabellenzeile oder eines zusammengesetzten Werts zu bezeichnen. Als Argument einer Aggregatfunktion hat er außerdem eine besondere Bedeutung, nämlich dass das Aggregat keinen ausdrücklichen Parameter benötigt.
- Der Punkt (`.`) wird in numerischen Konstanten verwendet sowie zur Trennung von Schema-, Tabellen- und Spaltennamen.

### 4.1.5. Kommentare

Ein Kommentar ist eine Zeichenfolge, die mit zwei Bindestrichen beginnt und bis zum Ende der Zeile reicht, zum Beispiel:

```sql
-- Dies ist ein Standard-SQL-Kommentar
```

Alternativ können C-artige Blockkommentare verwendet werden:

```sql
/* mehrzeiliger Kommentar
 * mit Verschachtelung: /* verschachtelter Blockkommentar */
 */
```

Dabei beginnt der Kommentar mit `/*` und reicht bis zum passenden Auftreten von `*/`. Diese Blockkommentare sind verschachtelbar, wie im SQL-Standard festgelegt, anders als in C. Dadurch kann man größere Codeblöcke auskommentieren, die bereits vorhandene Blockkommentare enthalten könnten.

Ein Kommentar wird vor der weiteren Syntaxanalyse aus dem Eingabestrom entfernt und effektiv durch Leerraum ersetzt.

### 4.1.6. Operatorpräzedenz

Tabelle 4.2 zeigt Präzedenz und Assoziativität der Operatoren in PostgreSQL. Die meisten Operatoren haben dieselbe Präzedenz und sind linksassoziativ. Präzedenz und Assoziativität der Operatoren sind fest im Parser verdrahtet. Fügen Sie Klammern hinzu, wenn ein Ausdruck mit mehreren Operatoren anders geparst werden soll, als es die Präzedenzregeln nahelegen.

**Tabelle 4.2. Operatorpräzedenz (von höchster zu niedrigster)**

| Operator/Element | Assoziativität | Beschreibung |
|---|---|---|
| `.` | links | Trenner für Tabellen-/Spaltennamen |
| `::` | links | PostgreSQL-artiger Typecast |
| `[]` | links | Auswahl eines Array-Elements |
| `+`, `-` | rechts | Unäres Plus, unäres Minus |
| `COLLATE` | links | Collation-Auswahl |
| `AT` | links | `AT TIME ZONE`, `AT LOCAL` |
| `^` | links | Potenzierung |
| `*`, `/`, `%` | links | Multiplikation, Division, Modulo |
| `+`, `-` | links | Addition, Subtraktion |
| jeder andere Operator | links | Alle anderen nativen und benutzerdefinierten Operatoren |
| `BETWEEN`, `IN`, `LIKE`, `ILIKE`, `SIMILAR` |  | Bereichseinschluss, Mengenzugehörigkeit, Zeichenkettenvergleich |
| `<`, `>`, `=`, `<=`, `>=`, `<>` |  | Vergleichsoperatoren |
| `IS`, `ISNULL`, `NOTNULL` |  | `IS TRUE`, `IS FALSE`, `IS NULL`, `IS DISTINCT FROM` usw. |
| `NOT` | rechts | Logische Negation |
| `AND` | links | Logische Konjunktion |
| `OR` | links | Logische Disjunktion |

Beachten Sie, dass die Operatorpräzedenzregeln auch für benutzerdefinierte Operatoren gelten, die denselben Namen haben wie die oben genannten eingebauten Operatoren. Wenn Sie beispielsweise einen `+`-Operator für einen eigenen Datentyp definieren, hat er dieselbe Präzedenz wie der eingebaute `+`-Operator, ganz gleich, was Ihr Operator tut.

Wenn ein schemaqualifizierter Operatorname in der `OPERATOR`-Syntax verwendet wird, wie zum Beispiel in:

```sql
SELECT 3 OPERATOR(pg_catalog.+) 4;
```

dann wird das `OPERATOR`-Konstrukt mit der in Tabelle 4.2 für "jeder andere Operator" gezeigten Standardpräzedenz behandelt. Das gilt unabhängig davon, welcher konkrete Operator innerhalb von `OPERATOR()` erscheint.

> Hinweis: PostgreSQL-Versionen vor 9.5 verwendeten leicht andere Regeln für die Operatorpräzedenz. Insbesondere wurden `<=`, `>=` und `<>` früher als generische Operatoren behandelt; `IS`-Tests hatten eine höhere Priorität; und `NOT BETWEEN` sowie verwandte Konstrukte verhielten sich uneinheitlich, indem sie in einigen Fällen mit der Präzedenz von `NOT` statt `BETWEEN` interpretiert wurden. Diese Regeln wurden geändert, um dem SQL-Standard besser zu entsprechen und Verwirrung durch uneinheitliche Behandlung logisch gleichwertiger Konstrukte zu verringern. In den meisten Fällen führt dies zu keiner Verhaltensänderung oder vielleicht zu Fehlern wie "no such operator", die durch Hinzufügen von Klammern behoben werden können. Es gibt jedoch Grenzfälle, in denen eine Abfrage ihr Verhalten ändern kann, ohne dass ein Parse-Fehler gemeldet wird.

## 4.2. Wertausdrücke

Wertausdrücke werden in vielen Kontexten verwendet, etwa in der Zielliste des `SELECT`-Befehls, als neue Spaltenwerte in `INSERT` oder `UPDATE` oder in Suchbedingungen verschiedener Befehle. Das Ergebnis eines Wertausdrucks wird manchmal Skalar genannt, um es vom Ergebnis eines Tabellenausdrucks zu unterscheiden, das eine Tabelle ist. Wertausdrücke werden daher auch skalare Ausdrücke oder einfach Ausdrücke genannt. Die Ausdruckssyntax erlaubt die Berechnung von Werten aus primitiven Bestandteilen mithilfe arithmetischer, logischer, mengenbezogener und anderer Operationen.

Ein Wertausdruck ist eines der folgenden Elemente:

- Ein Konstanten- oder Literalwert

- Ein Spaltenverweis

- Ein positionsbezogener Parameterverweis im Rumpf einer Funktionsdefinition oder vorbereiteten Anweisung

- Ein indizierter Ausdruck

- Ein Feldauswahlausdruck

- Ein Operatoraufruf

- Ein Funktionsaufruf

- Ein Aggregatausdruck

- Ein Window-Funktionsaufruf

- Ein Typecast

- Ein Collation-Ausdruck

- Eine skalare Unterabfrage

- Ein Array-Konstruktor

- Ein Row-Konstruktor

- Ein weiterer Wertausdruck in Klammern, um Unterausdrücke zu gruppieren und Präzedenz zu überschreiben

Zusätzlich zu dieser Liste gibt es eine Reihe von Konstrukten, die als Ausdruck klassifiziert werden können, aber keinen allgemeinen Syntaxregeln folgen. Sie haben im Allgemeinen die Semantik einer Funktion oder eines Operators und werden an der jeweils passenden Stelle in [Kapitel 9](09_Funktionen_und_Operatoren.md) erklärt. Ein Beispiel ist die `IS NULL`-Klausel.

Konstanten wurden bereits in [Abschnitt 4.1.2](#412-konstanten) besprochen. Die folgenden Abschnitte behandeln die verbleibenden Optionen.

### 4.2.1. Spaltenverweise

Eine Spalte kann in folgender Form referenziert werden:

```sql
correlation.columnname
```

`correlation` ist der Name einer Tabelle (gegebenenfalls mit Schemaqualifikation) oder ein Alias für eine Tabelle, der über eine `FROM`-Klausel definiert wurde. Der Korrelationsname und der trennende Punkt können weggelassen werden, wenn der Spaltenname unter allen in der aktuellen Abfrage verwendeten Tabellen eindeutig ist. Siehe auch [Kapitel 7](07_Abfragen.md).

### 4.2.2. Positionsbezogene Parameter

Ein positionsbezogener Parameterverweis bezeichnet einen Wert, der einer SQL-Anweisung von außen bereitgestellt wird. Parameter werden in SQL-Funktionsdefinitionen und in vorbereiteten Abfragen verwendet. Einige Client-Bibliotheken unterstützen außerdem, Datenwerte getrennt vom SQL-Befehlsstring anzugeben; dann verweisen Parameter auf diese ausgelagerten Datenwerte. Die Form eines Parameterverweises ist:

```sql
$number
```

Betrachten Sie zum Beispiel die Definition einer Funktion `dept`:

```sql
CREATE FUNCTION dept(text) RETURNS dept
    AS $$ SELECT * FROM dept WHERE name = $1 $$
    LANGUAGE SQL;
```

Hier verweist `$1` bei jedem Funktionsaufruf auf den Wert des ersten Funktionsarguments.

### 4.2.3. Indizes

Wenn ein Ausdruck einen Wert eines Array-Typs liefert, kann ein bestimmtes Element des Arrays so extrahiert werden:

```sql
expression[subscript]
```

Mehrere benachbarte Elemente, also ein "Array-Slice", können so extrahiert werden:

```sql
expression[lower_subscript:upper_subscript]
```

Die eckigen Klammern `[` und `]` sind dabei wörtlich gemeint. Jeder Index ist selbst ein Ausdruck, der auf den nächsten ganzzahligen Wert gerundet wird.

Im Allgemeinen muss der Array-Ausdruck in Klammern gesetzt werden. Die Klammern können aber weggelassen werden, wenn der zu indizierende Ausdruck nur ein Spaltenverweis oder ein positionsbezogener Parameter ist. Bei mehrdimensionalen Arrays können außerdem mehrere Indizes aneinandergehängt werden. Zum Beispiel:

```sql
mytable.arraycolumn[4]
mytable.two_d_column[17][34]
$1[10:42]
(arrayfunction(a,b))[42]
```

Die Klammern im letzten Beispiel sind erforderlich. Mehr zu Arrays finden Sie in [Abschnitt 8.15](08_Datentypen.md#815-arrays).

### 4.2.4. Feldauswahl

Wenn ein Ausdruck einen Wert eines zusammengesetzten Typs (Row-Typs) liefert, kann ein bestimmtes Feld der Zeile so extrahiert werden:

```sql
expression.fieldname
```

Im Allgemeinen muss der Zeilenausdruck in Klammern gesetzt werden. Die Klammern können aber weggelassen werden, wenn der Ausdruck, aus dem ausgewählt wird, nur ein Tabellenverweis oder ein positionsbezogener Parameter ist. Zum Beispiel:

```sql
mytable.mycolumn
$1.somecolumn
(rowfunction(a,b)).col3
```

Ein qualifizierter Spaltenverweis ist also eigentlich nur ein Spezialfall der Feldauswahlsyntax. Ein wichtiger Sonderfall ist das Extrahieren eines Felds aus einer Tabellenspalte, die einen zusammengesetzten Typ hat:

```sql
(compositecol).somefield
(mytable.compositecol).somefield
```

Die Klammern sind hier erforderlich, um zu zeigen, dass `compositecol` ein Spaltenname und kein Tabellenname ist, beziehungsweise dass `mytable` im zweiten Fall ein Tabellenname und kein Schemaname ist.

Alle Felder eines zusammengesetzten Werts können mit `.*` angefordert werden:

```sql
(compositecol).*
```

Diese Schreibweise verhält sich je nach Kontext unterschiedlich; Details stehen in [Abschnitt 8.16.5](08_Datentypen.md#8165-zusammengesetzte-typen-in-abfragen-verwenden).

### 4.2.5. Operatoraufrufe

Für einen Operatoraufruf gibt es zwei mögliche Syntaxformen:

```sql
expression operator expression
operator expression
```

Die erste Form ist ein binärer Infixoperator, die zweite ein unärer Präfixoperator. Das Operator-Token folgt den Syntaxregeln aus [Abschnitt 4.1.3](#413-operatoren), ist eines der Schlüsselwörter `AND`, `OR` und `NOT` oder ist ein qualifizierter Operatorname in dieser Form:

```sql
OPERATOR(schema.operatorname)
```

Welche konkreten Operatoren existieren und ob sie unär oder binär sind, hängt davon ab, welche Operatoren vom System oder vom Benutzer definiert wurden. [Kapitel 9](09_Funktionen_und_Operatoren.md) beschreibt die eingebauten Operatoren.

### 4.2.6. Funktionsaufrufe

Die Syntax für einen Funktionsaufruf besteht aus dem Namen der Funktion (gegebenenfalls mit Schemaqualifikation), gefolgt von der in Klammern eingeschlossenen Argumentliste:

```sql
function_name ([expression [, expression ... ]] )
```

Das folgende Beispiel berechnet die Quadratwurzel aus 2:

```sql
sqrt(2)
```

Die Liste der eingebauten Funktionen steht in [Kapitel 9](09_Funktionen_und_Operatoren.md). Weitere Funktionen können vom Benutzer hinzugefügt werden.

Wenn Abfragen in einer Datenbank ausgeführt werden, in der einige Benutzer anderen Benutzern nicht vertrauen, beachten Sie beim Schreiben von Funktionsaufrufen die Sicherheitsvorkehrungen aus [Abschnitt 10.3](10_Typumwandlung.md#103-funktionen).

Argumente können optional mit Namen versehen werden. Details stehen in [Abschnitt 4.3](#43-funktionsaufrufe).

> Hinweis: Eine Funktion, die ein einzelnes Argument eines zusammengesetzten Typs entgegennimmt, kann optional mit Feldauswahlsyntax aufgerufen werden; umgekehrt kann Feldauswahl im funktionalen Stil geschrieben werden. Die Schreibweisen `col(table)` und `table.col` sind also austauschbar. Dieses Verhalten gehört nicht zum SQL-Standard, wird in PostgreSQL aber bereitgestellt, weil Funktionen dadurch "berechnete Felder" nachbilden können. Weitere Informationen finden Sie in [Abschnitt 8.16.5](08_Datentypen.md#8165-zusammengesetzte-typen-in-abfragen-verwenden).

### 4.2.7. Aggregatausdrücke

Ein Aggregatausdruck stellt die Anwendung einer Aggregatfunktion auf die von einer Abfrage ausgewählten Zeilen dar. Eine Aggregatfunktion reduziert mehrere Eingabewerte auf einen einzelnen Ausgabewert, etwa die Summe oder den Durchschnitt der Eingaben. Die Syntax eines Aggregatausdrucks ist eine der folgenden:

```sql
aggregate_name (expression [ , ... ] [ order_by_clause ] ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name (ALL expression [ , ... ] [ order_by_clause ] ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name (DISTINCT expression [ , ... ] [ order_by_clause ] ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name ( * ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name ( [ expression [ , ... ] ] ) WITHIN GROUP ( order_by_clause ) [ FILTER ( WHERE filter_clause ) ]
```

Dabei ist `aggregate_name` ein zuvor definiertes Aggregat (gegebenenfalls mit Schemaqualifikation), und `expression` ist ein beliebiger Wertausdruck, der selbst keinen Aggregatausdruck und keinen Window-Funktionsaufruf enthält. Die optionalen Klauseln `order_by_clause` und `filter_clause` werden unten beschrieben.

Die erste Form ruft das Aggregat einmal für jede Eingabezeile auf. Die zweite Form ist identisch, da `ALL` der Standard ist. Die dritte Form ruft das Aggregat einmal für jeden eindeutigen Wert des Ausdrucks auf, beziehungsweise für jede eindeutige Wertekombination bei mehreren Ausdrücken. Die vierte Form ruft das Aggregat einmal für jede Eingabezeile auf; da kein bestimmter Eingabewert angegeben wird, ist sie im Allgemeinen nur für die Aggregatfunktion `count(*)` nützlich. Die letzte Form wird für Ordered-Set-Aggregatfunktionen verwendet, die weiter unten beschrieben werden.

Die meisten Aggregatfunktionen ignorieren Nullwerte, sodass Zeilen verworfen werden, in denen einer oder mehrere Ausdrücke `null` ergeben. Wenn nichts anderes angegeben ist, kann dies für alle eingebauten Aggregate angenommen werden.

Zum Beispiel liefert `count(*)` die Gesamtzahl der Eingabezeilen; `count(f1)` liefert die Anzahl der Eingabezeilen, in denen `f1` nicht null ist, da `count` Nullwerte ignoriert; und `count(distinct f1)` liefert die Anzahl der eindeutigen Nicht-Null-Werte von `f1`.

Normalerweise werden die Eingabezeilen in nicht festgelegter Reihenfolge an die Aggregatfunktion übergeben. In vielen Fällen spielt das keine Rolle; `min` erzeugt beispielsweise unabhängig von der Reihenfolge dasselbe Ergebnis. Einige Aggregatfunktionen, etwa `array_agg` und `string_agg`, erzeugen jedoch Ergebnisse, die von der Reihenfolge der Eingabezeilen abhängen. Bei solchen Aggregaten kann die optionale Klausel `order_by_clause` die gewünschte Reihenfolge angeben. Sie hat dieselbe Syntax wie eine `ORDER BY`-Klausel auf Abfrageebene, wie in [Abschnitt 7.5](07_Abfragen.md#75-zeilen-sortieren-order-by) beschrieben, außer dass ihre Ausdrücke immer einfache Ausdrücke sind und keine Ausgabespaltennamen oder -nummern sein können. Zum Beispiel:

```sql
WITH vals (v) AS ( VALUES (1),(3),(4),(3),(2) )
SELECT array_agg(v ORDER BY v DESC) FROM vals;
  array_agg
-------------
 {4,3,3,2,1}
```

Da `jsonb` nur den letzten passenden Schlüssel behält, kann die Reihenfolge der Schlüssel bedeutsam sein:

```sql
WITH vals (k, v) AS ( VALUES ('key0','1'), ('key1','3'),
 ('key1','2') )
SELECT jsonb_object_agg(k, v ORDER BY v) FROM vals;
      jsonb_object_agg
----------------------------
 {"key0": "1", "key1": "3"}
```

Bei Aggregatfunktionen mit mehreren Argumenten steht die `ORDER BY`-Klausel nach allen Aggregatargumenten. Schreiben Sie also:

```sql
SELECT string_agg(a, ',' ORDER BY a) FROM table;
```

nicht:

```sql
SELECT string_agg(a ORDER BY a, ',') FROM table;                             -- falsch
```

Letzteres ist syntaktisch gültig, beschreibt aber den Aufruf einer einargumentigen Aggregatfunktion mit zwei `ORDER BY`-Schlüsseln; der zweite ist dabei wenig nützlich, weil er eine Konstante ist.

Wenn `DISTINCT` zusammen mit einer `order_by_clause` angegeben wird, dürfen `ORDER BY`-Ausdrücke nur Spalten aus der `DISTINCT`-Liste referenzieren. Zum Beispiel:

```sql
WITH vals (v) AS ( VALUES (1),(3),(4),(3),(2) )
SELECT array_agg(DISTINCT v ORDER BY v DESC) FROM vals;
 array_agg
-----------
 {4,3,2,1}
```

Die bisher beschriebene Platzierung von `ORDER BY` innerhalb der regulären Argumentliste des Aggregats wird für allgemeine und statistische Aggregate verwendet, bei denen die Sortierung optional ist. Daneben gibt es Ordered-Set-Aggregate, für die eine `order_by_clause` erforderlich ist, meist weil ihre Berechnung nur in Bezug auf eine bestimmte Eingabereihenfolge sinnvoll ist. Typische Beispiele sind Rang- und Perzentilberechnungen. Bei einem Ordered-Set-Aggregat steht die `order_by_clause` in `WITHIN GROUP (...)`, wie in der letzten Syntaxalternative oben gezeigt.

Die Ausdrücke in der `order_by_clause` werden einmal pro Eingabezeile ausgewertet, wie reguläre Aggregatargumente sortiert und der Aggregatfunktion als Eingabeargumente übergeben. Dies unterscheidet sich von einer `order_by_clause` außerhalb von `WITHIN GROUP`, die nicht als Argument der Aggregatfunktion behandelt wird. Die Ausdrücke vor `WITHIN GROUP`, sofern vorhanden, heißen direkte Argumente, um sie von den aggregierten Argumenten in der `order_by_clause` zu unterscheiden. Anders als reguläre Aggregatargumente werden direkte Argumente nur einmal pro Aggregataufruf ausgewertet, nicht einmal pro Eingabezeile. Sie dürfen daher Variablen nur enthalten, wenn diese Variablen durch `GROUP BY` gruppiert sind. Direkte Argumente werden typischerweise für Dinge wie Perzentilanteile verwendet, die nur als einzelner Wert pro Aggregationsberechnung sinnvoll sind. Die Liste direkter Argumente kann leer sein; schreiben Sie dann `()` und nicht `*`. PostgreSQL akzeptiert zwar beide Schreibweisen, aber nur die erste entspricht dem SQL-Standard.

Ein Beispiel für einen Ordered-Set-Aggregataufruf ist:

```sql
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY income) FROM
 households;
 percentile_cont
-----------------
```

Dies ermittelt das 50. Perzentil, also den Median, der Spalte `income` aus der Tabelle `households`. Hier ist `0.5` ein direktes Argument; es wäre nicht sinnvoll, wenn der Perzentilanteil zeilenweise variieren würde.

Wenn `FILTER` angegeben ist, werden nur die Eingabezeilen an die Aggregatfunktion übergeben, für die `filter_clause` wahr ergibt; andere Zeilen werden verworfen. Zum Beispiel:

```sql
SELECT
    count(*) AS unfiltered,
    count(*) FILTER (WHERE i < 5) AS filtered
FROM generate_series(1,10) AS s(i);
 unfiltered | filtered
------------+----------
         10 |        4
(1 row)
```

Die vordefinierten Aggregatfunktionen werden in [Abschnitt 9.21](09_Funktionen_und_Operatoren.md#921-aggregatfunktionen) beschrieben. Weitere Aggregatfunktionen können vom Benutzer hinzugefügt werden.

Ein Aggregatausdruck darf nur in der Ergebnisliste oder in der `HAVING`-Klausel eines `SELECT`-Befehls erscheinen. In anderen Klauseln, etwa `WHERE`, ist er verboten, weil diese Klauseln logisch ausgewertet werden, bevor die Aggregate gebildet werden.

Wenn ein Aggregatausdruck in einer Unterabfrage erscheint (siehe [Abschnitt 4.2.11](#4211-skalare-unterabfragen) und [Abschnitt 9.24](09_Funktionen_und_Operatoren.md#924-unterabfrageausdrücke)), wird das Aggregat normalerweise über die Zeilen der Unterabfrage ausgewertet. Eine Ausnahme gilt, wenn die Argumente des Aggregats und gegebenenfalls `filter_clause` nur Variablen einer äußeren Ebene enthalten: Dann gehört das Aggregat zur nächsten solchen äußeren Ebene und wird über die Zeilen dieser Abfrage ausgewertet. Der gesamte Aggregatausdruck ist dann eine äußere Referenz für die Unterabfrage, in der er erscheint, und wirkt bei jeder einzelnen Auswertung der Unterabfrage wie eine Konstante. Die Beschränkung auf Ergebnisliste oder `HAVING`-Klausel gilt in Bezug auf die Abfrageebene, zu der das Aggregat gehört.

### 4.2.8. Aufrufe von Window-Funktionen

Ein Aufruf einer Window-Funktion stellt die Anwendung einer aggregatähnlichen Funktion auf einen Teil der von einer Abfrage ausgewählten Zeilen dar. Anders als bei normalen Aggregataufrufen werden die ausgewählten Zeilen dabei nicht zu einer einzigen Ausgabezeile gruppiert; jede Zeile bleibt in der Abfrageausgabe separat. Die Window-Funktion hat jedoch Zugriff auf alle Zeilen, die gemäß der Gruppierungsspezifikation (`PARTITION BY`-Liste) des Aufrufs zur Gruppe der aktuellen Zeile gehören würden. Die Syntax eines solchen Aufrufs ist eine der folgenden:

```sql
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
```

Dabei hat `window_definition` folgende Syntax:

```sql
[ existing_window_name ]
[ PARTITION BY expression [, ...] ]
[ ORDER BY expression [ ASC | DESC | USING operator ] [ NULLS { FIRST | LAST } ] [, ...] ]
[ frame_clause ]
```

Die optionale `frame_clause` kann eine der folgenden Formen haben:

```sql
{ RANGE | ROWS | GROUPS } frame_start [ frame_exclusion ]
{ RANGE | ROWS | GROUPS } BETWEEN frame_start AND frame_end [ frame_exclusion ]
```

`frame_start` und `frame_end` können jeweils eines der folgenden Elemente sein:

```sql
UNBOUNDED PRECEDING
offset PRECEDING
CURRENT ROW
offset FOLLOWING
UNBOUNDED FOLLOWING
```

`frame_exclusion` kann eines der folgenden Elemente sein:

```sql
EXCLUDE CURRENT ROW
EXCLUDE GROUP
EXCLUDE TIES
EXCLUDE NO OTHERS
```

Hier steht `expression` für einen beliebigen Wertausdruck, der selbst keine Aufrufe von Window-Funktionen enthält.

`window_name` ist ein Verweis auf eine benannte Window-Spezifikation, die in der `WINDOW`-Klausel der Abfrage definiert wurde. Alternativ kann eine vollständige `window_definition` in Klammern angegeben werden, mit derselben Syntax wie bei der Definition eines benannten Windows in der `WINDOW`-Klausel; Details stehen auf der Referenzseite zu `SELECT`. Wichtig ist, dass `OVER wname` nicht exakt gleichbedeutend mit `OVER (wname ...)` ist: Letzteres bedeutet, dass die Window-Definition kopiert und verändert wird, und wird zurückgewiesen, wenn die referenzierte Window-Spezifikation eine Frame-Klausel enthält.

Die `PARTITION BY`-Klausel gruppiert die Zeilen der Abfrage in Partitionen, die von der Window-Funktion getrennt verarbeitet werden. `PARTITION BY` funktioniert ähnlich wie eine `GROUP BY`-Klausel auf Abfrageebene, mit dem Unterschied, dass ihre Ausdrücke immer einfache Ausdrücke sind und keine Ausgabespaltennamen oder -nummern sein können. Ohne `PARTITION BY` werden alle von der Abfrage erzeugten Zeilen als eine einzige Partition behandelt. Die `ORDER BY`-Klausel bestimmt die Reihenfolge, in der die Zeilen einer Partition von der Window-Funktion verarbeitet werden. Sie funktioniert ähnlich wie eine `ORDER BY`-Klausel auf Abfrageebene, kann aber ebenfalls keine Ausgabespaltennamen oder -nummern verwenden. Ohne `ORDER BY` werden Zeilen in nicht festgelegter Reihenfolge verarbeitet.

Die `frame_clause` legt für Window-Funktionen, die auf dem Frame und nicht auf der ganzen Partition arbeiten, die Menge der Zeilen fest, die den Window-Frame bilden. Dieser Frame ist eine Teilmenge der aktuellen Partition und kann je nach aktueller Zeile variieren. Der Frame kann im Modus `RANGE`, `ROWS` oder `GROUPS` angegeben werden; in jedem Fall reicht er von `frame_start` bis `frame_end`. Wird `frame_end` weggelassen, ist das Ende standardmäßig `CURRENT ROW`.

`UNBOUNDED PRECEDING` als `frame_start` bedeutet, dass der Frame mit der ersten Zeile der Partition beginnt. Entsprechend bedeutet `UNBOUNDED FOLLOWING` als `frame_end`, dass der Frame mit der letzten Zeile der Partition endet.

Im Modus `RANGE` oder `GROUPS` bedeutet `CURRENT ROW` als `frame_start`, dass der Frame mit der ersten Peer-Zeile der aktuellen Zeile beginnt, also mit einer Zeile, die durch die `ORDER BY`-Klausel des Windows als gleichwertig zur aktuellen Zeile sortiert wird. `CURRENT ROW` als `frame_end` bedeutet, dass der Frame mit der letzten Peer-Zeile der aktuellen Zeile endet. Im Modus `ROWS` bedeutet `CURRENT ROW` einfach die aktuelle Zeile.

Bei den Frame-Optionen `offset PRECEDING` und `offset FOLLOWING` muss `offset` ein Ausdruck sein, der keine Variablen, Aggregatfunktionen oder Window-Funktionen enthält. Die Bedeutung des Offsets hängt vom Frame-Modus ab:

- Im Modus `ROWS` muss der Offset eine nicht-nullige, nichtnegative Ganzzahl ergeben; die Option bedeutet, dass der Frame die angegebene Anzahl von Zeilen vor oder nach der aktuellen Zeile beginnt oder endet.
- Im Modus `GROUPS` muss der Offset ebenfalls eine nicht-nullige, nichtnegative Ganzzahl ergeben; die Option bedeutet, dass der Frame die angegebene Anzahl von Peer-Gruppen vor oder nach der Peer-Gruppe der aktuellen Zeile beginnt oder endet. Eine Peer-Gruppe ist dabei eine Menge von Zeilen, die in der `ORDER BY`-Sortierung gleichwertig sind. Für den Modus `GROUPS` muss in der Window-Definition eine `ORDER BY`-Klausel vorhanden sein.
- Im Modus `RANGE` verlangen diese Optionen, dass die `ORDER BY`-Klausel genau eine Spalte angibt. Der Offset bestimmt die maximale Differenz zwischen dem Wert dieser Spalte in der aktuellen Zeile und ihrem Wert in vorangehenden oder folgenden Zeilen des Frames. Der Datentyp des Offset-Ausdrucks hängt vom Datentyp der Sortierspalte ab. Bei numerischen Sortierspalten ist es typischerweise derselbe Typ wie der der Sortierspalte; bei Datums-/Zeit-Sortierspalten ist es ein Intervall. Wenn die Sortierspalte beispielsweise vom Typ `date` oder `timestamp` ist, könnte man `RANGE BETWEEN '1 day' PRECEDING AND '10 days' FOLLOWING` schreiben. Der Offset muss weiterhin nicht-null und nichtnegativ sein, auch wenn die Bedeutung von "nichtnegativ" vom Datentyp abhängt.

In jedem Fall ist der Abstand zum Ende des Frames durch den Abstand zum Ende der Partition begrenzt, sodass der Frame bei Zeilen nahe an den Partitionsenden weniger Zeilen enthalten kann als anderswo.

In den Modi `ROWS` und `GROUPS` sind `0 PRECEDING` und `0 FOLLOWING` gleichbedeutend mit `CURRENT ROW`. Im Modus `RANGE` gilt dies normalerweise ebenfalls, mit einer zum Datentyp passenden Bedeutung von "Null".

Die Option `frame_exclusion` erlaubt es, Zeilen um die aktuelle Zeile herum aus dem Frame auszuschließen, selbst wenn sie gemäß Frame-Start und Frame-Ende enthalten wären. `EXCLUDE CURRENT ROW` schließt die aktuelle Zeile aus dem Frame aus. `EXCLUDE GROUP` schließt die aktuelle Zeile und ihre Sortier-Peers aus. `EXCLUDE TIES` schließt alle Peers der aktuellen Zeile aus, aber nicht die aktuelle Zeile selbst. `EXCLUDE NO OTHERS` gibt ausdrücklich das Standardverhalten an, also keine aktuelle Zeile und keine Peers auszuschließen.

Die Standard-Frame-Option ist `RANGE UNBOUNDED PRECEDING`; das entspricht `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Mit `ORDER BY` umfasst der Frame alle Zeilen vom Partitionsanfang bis zur letzten `ORDER BY`-Peer-Zeile der aktuellen Zeile. Ohne `ORDER BY` umfasst der Window-Frame alle Zeilen der Partition, weil alle Zeilen zu Peers der aktuellen Zeile werden.

Einschränkungen: `frame_start` darf nicht `UNBOUNDED FOLLOWING` sein, `frame_end` darf nicht `UNBOUNDED PRECEDING` sein, und die Wahl von `frame_end` darf in der oben gezeigten Liste der Optionen nicht früher erscheinen als die Wahl von `frame_start`. Zum Beispiel ist `RANGE BETWEEN CURRENT ROW AND offset PRECEDING` nicht erlaubt. `ROWS BETWEEN 7 PRECEDING AND 8 PRECEDING` ist dagegen erlaubt, auch wenn es nie Zeilen auswählen würde.

Wenn `FILTER` angegeben ist, werden nur die Eingabezeilen an die Window-Funktion übergeben, für die `filter_clause` wahr ergibt; andere Zeilen werden verworfen. Nur Window-Funktionen, die Aggregate sind, akzeptieren eine `FILTER`-Klausel.

Die eingebauten Window-Funktionen werden in [Tabelle 9.67](09_Funktionen_und_Operatoren.md#922-fensterfunktionen) beschrieben. Weitere Window-Funktionen können vom Benutzer hinzugefügt werden. Außerdem kann jedes eingebaute oder benutzerdefinierte allgemeine oder statistische Aggregat als Window-Funktion verwendet werden. Ordered-Set- und Hypothetical-Set-Aggregate können derzeit nicht als Window-Funktionen verwendet werden.

Die Syntaxformen mit `*` werden verwendet, um parameterlose Aggregatfunktionen als Window-Funktionen aufzurufen, zum Beispiel `count(*) OVER (PARTITION BY x ORDER BY y)`. Der Stern (`*`) wird für window-spezifische Funktionen üblicherweise nicht verwendet. Window-spezifische Funktionen erlauben weder `DISTINCT` noch `ORDER BY` innerhalb der Funktionsargumentliste.

Aufrufe von Window-Funktionen sind nur in der `SELECT`-Liste und in der `ORDER BY`-Klausel der Abfrage erlaubt.

Weitere Informationen zu Window-Funktionen finden Sie in [Abschnitt 3.5](03_Fortgeschrittene_Funktionen.md#35-windowfunktionen), [Abschnitt 9.22](09_Funktionen_und_Operatoren.md#922-fensterfunktionen) und [Abschnitt 7.2.5](07_Abfragen.md#725-verarbeitung-von-windowfunktionen).

### 4.2.9. Typecasts

Ein Typecast gibt eine Umwandlung von einem Datentyp in einen anderen an. PostgreSQL akzeptiert zwei gleichwertige Syntaxformen für Typecasts:

```sql
CAST ( expression AS type )
expression::type
```

Die `CAST`-Syntax entspricht SQL; die Syntax mit `::` ist historischer PostgreSQL-Gebrauch.

Wenn ein Cast auf einen Wertausdruck bekannten Typs angewendet wird, beschreibt er eine Laufzeit-Typumwandlung. Der Cast ist nur erfolgreich, wenn eine passende Typumwandlungsoperation definiert wurde. Das unterscheidet sich subtil von der Verwendung von Casts mit Konstanten, wie in [Abschnitt 4.1.2.7](#4127-konstanten-anderer-typen) gezeigt. Ein Cast auf ein unverziertes Zeichenkettenliteral stellt die anfängliche Zuweisung eines Typs zu einem Literalwert dar und ist daher für jeden Typ erfolgreich, sofern der Inhalt des Zeichenkettenliterals eine zulässige Eingabesyntax für den Datentyp ist.

Ein expliziter Typecast kann normalerweise weggelassen werden, wenn keine Mehrdeutigkeit darüber besteht, welchen Typ ein Wertausdruck erzeugen muss, etwa wenn er einer Tabellenspalte zugewiesen wird; das System wendet in solchen Fällen automatisch einen Typecast an. Automatisches Casting erfolgt jedoch nur für Casts, die in den Systemkatalogen als "implizit anwendbar" markiert sind. Andere Casts müssen mit expliziter Cast-Syntax aufgerufen werden. Diese Einschränkung soll verhindern, dass überraschende Umwandlungen stillschweigend angewendet werden.

Es ist auch möglich, einen Typecast mit funktionsähnlicher Syntax anzugeben:

```sql
typename ( expression )
```

Das funktioniert jedoch nur für Typen, deren Namen auch als Funktionsnamen gültig sind. Zum Beispiel kann `double precision` auf diese Weise nicht verwendet werden, wohl aber das äquivalente `float8`. Außerdem können die Namen `interval`, `time` und `timestamp` in dieser Form nur verwendet werden, wenn sie in doppelte Anführungszeichen gesetzt werden, weil es syntaktische Konflikte gibt. Die funktionsähnliche Cast-Syntax führt daher zu Inkonsistenzen und sollte vermutlich vermieden werden.

> Hinweis: Die funktionsähnliche Syntax ist tatsächlich nur ein Funktionsaufruf. Wenn eine der beiden Standard-Cast-Syntaxen für eine Laufzeitumwandlung verwendet wird, ruft PostgreSQL intern eine registrierte Funktion auf, die die Umwandlung ausführt. Nach Konvention haben diese Umwandlungsfunktionen denselben Namen wie ihr Ausgabetyp; die "funktionsähnliche Syntax" ist also nichts anderes als ein direkter Aufruf der zugrunde liegenden Umwandlungsfunktion. Darauf sollte sich eine portable Anwendung offensichtlich nicht verlassen. Weitere Details finden Sie unter `CREATE CAST`.

### 4.2.10. Collation-Ausdrücke

Die `COLLATE`-Klausel überschreibt die Sortierfolge eines Ausdrucks. Sie wird an den Ausdruck angehängt, für den sie gelten soll:

```sql
expr COLLATE collation
```

Dabei ist `collation` ein gegebenenfalls schemaqualifizierter Bezeichner. Die `COLLATE`-Klausel bindet stärker als Operatoren; bei Bedarf können Klammern verwendet werden.

Wenn keine Sortierfolge explizit angegeben wird, leitet das Datenbanksystem entweder eine Sortierfolge aus den am Ausdruck beteiligten Spalten ab oder verwendet die Standardsortierfolge der Datenbank, wenn keine Spalte am Ausdruck beteiligt ist.

Die beiden häufigen Verwendungen der `COLLATE`-Klausel sind das Überschreiben der Sortierreihenfolge in einer `ORDER BY`-Klausel, zum Beispiel:

```sql
SELECT a, b, c FROM tbl WHERE ... ORDER BY a COLLATE "C";
```

und das Überschreiben der Sortierfolge eines Funktions- oder Operatoraufrufs, dessen Ergebnis locale-abhängig ist, zum Beispiel:

```sql
SELECT * FROM tbl WHERE a > 'foo' COLLATE "C";
```

Im zweiten Fall ist die `COLLATE`-Klausel an ein Eingabeargument des Operators angehängt, den wir beeinflussen wollen. Es spielt keine Rolle, an welches Argument des Operators oder Funktionsaufrufs die `COLLATE`-Klausel angehängt wird, weil die vom Operator oder der Funktion angewendete Sortierfolge aus allen Argumenten abgeleitet wird und eine explizite `COLLATE`-Klausel die Sortierfolgen aller anderen Argumente überschreibt. Mehrere nicht übereinstimmende `COLLATE`-Klauseln an verschiedenen Argumenten sind jedoch ein Fehler. Weitere Details finden Sie in [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung). Daher liefert dies dasselbe Ergebnis wie das vorige Beispiel:

```sql
SELECT * FROM tbl WHERE a COLLATE "C" > 'foo';
```

Dies ist dagegen ein Fehler:

```sql
SELECT * FROM tbl WHERE (a > 'foo') COLLATE "C";
```

Denn hier würde versucht, eine Sortierfolge auf das Ergebnis des Operators `>` anzuwenden, das den nicht sortierbaren Datentyp `boolean` hat.

### 4.2.11. Skalare Unterabfragen

Eine skalare Unterabfrage ist eine gewöhnliche `SELECT`-Abfrage in Klammern, die genau eine Zeile mit genau einer Spalte zurückgibt. Informationen zum Schreiben von Abfragen finden Sie in [Kapitel 7](07_Abfragen.md). Die `SELECT`-Abfrage wird ausgeführt, und der einzelne zurückgegebene Wert wird im umgebenden Wertausdruck verwendet. Es ist ein Fehler, eine Abfrage als skalare Unterabfrage zu verwenden, die mehr als eine Zeile oder mehr als eine Spalte zurückgibt. Wenn die Unterabfrage bei einer bestimmten Ausführung keine Zeilen zurückgibt, ist das jedoch kein Fehler; das skalare Ergebnis wird dann als null angenommen. Die Unterabfrage kann Variablen aus der umgebenden Abfrage referenzieren, die während jeder einzelnen Auswertung der Unterabfrage wie Konstanten wirken. Siehe auch [Abschnitt 9.24](09_Funktionen_und_Operatoren.md#924-unterabfrageausdrücke) für andere Ausdrücke mit Unterabfragen.

Das folgende Beispiel findet die größte Stadtbevölkerung in jedem Bundesstaat:

```sql
SELECT name, (SELECT max(pop) FROM cities WHERE cities.state =
 states.name)
    FROM states;
```

### 4.2.12. Array-Konstruktoren

Ein Array-Konstruktor ist ein Ausdruck, der aus Werten für seine Elemente einen Array-Wert erzeugt. Ein einfacher Array-Konstruktor besteht aus dem Schlüsselwort `ARRAY`, einer öffnenden eckigen Klammer `[`, einer durch Kommas getrennten Liste von Ausdrücken für die Array-Elementwerte und einer schließenden eckigen Klammer `]`. Zum Beispiel:

```sql
SELECT ARRAY[1,2,3+4];
  array
---------
 {1,2,7}
(1 row)
```

Standardmäßig ist der Array-Elementtyp der gemeinsame Typ der Elementausdrücke, bestimmt nach denselben Regeln wie bei `UNION`- oder `CASE`-Konstrukten (siehe [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte)). Dies kann überschrieben werden, indem der Array-Konstruktor explizit in den gewünschten Typ gecastet wird:

```sql
SELECT ARRAY[1,2,22.7]::integer[];
  array
----------
 {1,2,23}
(1 row)
```

Das hat denselben Effekt, als würde jeder Ausdruck einzeln in den Array-Elementtyp gecastet. Mehr zu Casts finden Sie in [Abschnitt 4.2.9](#429-typecasts).

Mehrdimensionale Array-Werte können durch Verschachteln von Array-Konstruktoren erzeugt werden. In den inneren Konstruktoren kann das Schlüsselwort `ARRAY` weggelassen werden. Zum Beispiel erzeugen diese beiden Abfragen dasselbe Ergebnis:

```sql
SELECT ARRAY[ARRAY[1,2], ARRAY[3,4]];
     array
---------------
 {{1,2},{3,4}}
(1 row)

SELECT ARRAY[[1,2],[3,4]];
     array
---------------
 {{1,2},{3,4}}
(1 row)
```

Da mehrdimensionale Arrays rechteckig sein müssen, müssen innere Konstruktoren auf derselben Ebene Unterarrays mit identischen Dimensionen erzeugen. Jeder Cast, der auf den äußeren `ARRAY`-Konstruktor angewendet wird, wird automatisch auf alle inneren Konstruktoren übertragen.

Elemente eines mehrdimensionalen Array-Konstruktors können alles sein, was ein Array der passenden Art ergibt, nicht nur ein Unter-`ARRAY`-Konstrukt. Zum Beispiel:

```sql
CREATE TABLE arr(f1 int[], f2 int[]);

INSERT INTO arr VALUES (ARRAY[[1,2],[3,4]], ARRAY[[5,6],[7,8]]);

SELECT ARRAY[f1, f2, '{{9,10},{11,12}}'::int[]] FROM arr;
                     array
------------------------------------------------
 {{{1,2},{3,4}},{{5,6},{7,8}},{{9,10},{11,12}}}
(1 row)
```

Ein leeres Array kann erzeugt werden. Da ein Array ohne Typ jedoch nicht möglich ist, muss das leere Array explizit in den gewünschten Typ gecastet werden:

```sql
SELECT ARRAY[]::integer[];
 array
-------
 {}
(1 row)
```

Es ist auch möglich, ein Array aus den Ergebnissen einer Unterabfrage zu konstruieren. In dieser Form wird der Array-Konstruktor mit dem Schlüsselwort `ARRAY` gefolgt von einer in runde Klammern gesetzten Unterabfrage geschrieben, nicht mit eckigen Klammern. Zum Beispiel:

```sql
SELECT ARRAY(SELECT oid FROM pg_proc WHERE proname LIKE 'bytea%');
                              array
------------------------------------------------------------------
 {2011,1954,1948,1952,1951,1244,1950,2005,1949,1953,2006,31,2412}
(1 row)

SELECT ARRAY(SELECT ARRAY[i, i*2] FROM generate_series(1,5) AS
 a(i));
              array
----------------------------------
 {{1,2},{2,4},{3,6},{4,8},{5,10}}
(1 row)
```

Die Unterabfrage muss eine einzelne Spalte zurückgeben. Wenn die Ausgabespalte der Unterabfrage kein Array-Typ ist, hat das resultierende eindimensionale Array ein Element für jede Zeile im Unterabfrageergebnis; der Elementtyp entspricht dem Typ der Ausgabespalte. Wenn die Ausgabespalte der Unterabfrage ein Array-Typ ist, ist das Ergebnis ein Array desselben Typs, aber mit einer Dimension mehr. In diesem Fall müssen alle Unterabfragezeilen Arrays identischer Dimensionalität liefern, sonst wäre das Ergebnis nicht rechteckig.

Die Indizes eines mit `ARRAY` erzeugten Array-Werts beginnen immer bei eins. Weitere Informationen zu Arrays finden Sie in [Abschnitt 8.15](08_Datentypen.md#815-arrays).

### 4.2.13. Row-Konstruktoren

Ein Row-Konstruktor ist ein Ausdruck, der aus Werten für seine Felder einen Zeilenwert erzeugt, auch zusammengesetzter Wert genannt. Ein Row-Konstruktor besteht aus dem Schlüsselwort `ROW`, einer öffnenden runden Klammer, null oder mehr durch Kommas getrennten Ausdrücken für die Feldwerte der Zeile und einer schließenden runden Klammer. Zum Beispiel:

```sql
SELECT ROW(1,2.5,'this is a test');
```

Das Schlüsselwort `ROW` ist optional, wenn die Liste mehr als einen Ausdruck enthält.

Ein Row-Konstruktor kann die Syntax `rowvalue.*` enthalten. Sie wird zu einer Liste der Elemente des Row-Werts erweitert, genauso wie die `.*`-Syntax auf oberster Ebene einer `SELECT`-Liste erweitert wird (siehe [Abschnitt 8.16.5](08_Datentypen.md#8165-zusammengesetzte-typen-in-abfragen-verwenden)). Wenn die Tabelle `t` zum Beispiel die Spalten `f1` und `f2` hat, sind diese beiden Abfragen gleichbedeutend:

```sql
SELECT ROW(t.*, 42) FROM t;
SELECT ROW(t.f1, t.f2, 42) FROM t;
```

> Hinweis: Vor PostgreSQL 8.2 wurde die `.*`-Syntax in Row-Konstruktoren nicht erweitert, sodass `ROW(t.*, 42)` eine zweifeldrige Zeile erzeugte, deren erstes Feld ein weiterer Row-Wert war. Das neue Verhalten ist meistens nützlicher. Wenn Sie das alte Verhalten verschachtelter Row-Werte benötigen, schreiben Sie den inneren Row-Wert ohne `.*`, zum Beispiel `ROW(t, 42)`.

Standardmäßig hat der durch einen `ROW`-Ausdruck erzeugte Wert einen anonymen `record`-Typ. Bei Bedarf kann er in einen benannten zusammengesetzten Typ gecastet werden, entweder in den Row-Typ einer Tabelle oder in einen mit `CREATE TYPE AS` erzeugten zusammengesetzten Typ. Ein expliziter Cast kann nötig sein, um Mehrdeutigkeit zu vermeiden. Zum Beispiel:

```sql
CREATE TABLE mytable(f1 int, f2 float, f3 text);

CREATE FUNCTION getf1(mytable) RETURNS int AS 'SELECT $1.f1'
 LANGUAGE SQL;

-- Kein Cast nötig, da nur ein getf1() existiert
SELECT getf1(ROW(1,2.5,'this is a test'));
 getf1
-------
(1 row)

CREATE TYPE myrowtype AS (f1 int, f2 text, f3 numeric);

CREATE FUNCTION getf1(myrowtype) RETURNS int AS 'SELECT $1.f1'
 LANGUAGE SQL;

-- Jetzt brauchen wir einen Cast, um anzugeben, welche Funktion aufgerufen werden soll:
SELECT getf1(ROW(1,2.5,'this is a test'));
ERROR: function getf1(record) is not unique

SELECT getf1(ROW(1,2.5,'this is a test')::mytable);
 getf1
-------
(1 row)

SELECT getf1(CAST(ROW(11,'this is a test',2.5) AS myrowtype));
 getf1
-------
(1 row)
```

Row-Konstruktoren können verwendet werden, um zusammengesetzte Werte zu erzeugen, die in einer Tabellenspalte zusammengesetzten Typs gespeichert oder an eine Funktion übergeben werden, die einen zusammengesetzten Parameter akzeptiert. Außerdem können Zeilen mit den Standardvergleichsoperatoren getestet werden, wie in [Abschnitt 9.2](09_Funktionen_und_Operatoren.md#92-vergleichsfunktionen-und-operatoren) beschrieben; eine Zeile kann mit einer anderen verglichen werden, wie in [Abschnitt 9.25](09_Funktionen_und_Operatoren.md#925-zeilen-und-arrayvergleiche) beschrieben; und Row-Konstruktoren können in Verbindung mit Unterabfragen verwendet werden, wie in [Abschnitt 9.24](09_Funktionen_und_Operatoren.md#924-unterabfrageausdrücke) erläutert.

### 4.2.14. Regeln für die Ausdrucksauswertung

Die Auswertungsreihenfolge von Teilausdrücken ist nicht definiert. Insbesondere werden die Eingaben eines Operators oder einer Funktion nicht notwendigerweise von links nach rechts oder in irgendeiner anderen festen Reihenfolge ausgewertet.

Wenn das Ergebnis eines Ausdrucks durch Auswertung nur einiger Teile bestimmt werden kann, werden andere Teilausdrücke möglicherweise überhaupt nicht ausgewertet. Wenn man zum Beispiel schreibt:

```sql
SELECT true OR somefunc();
```

dann würde `somefunc()` wahrscheinlich gar nicht aufgerufen. Dasselbe wäre bei folgender Schreibweise der Fall:

```sql
SELECT somefunc() OR true;
```

Beachten Sie, dass dies nicht dasselbe ist wie das von links nach rechts laufende "Short-Circuiting" boolescher Operatoren, das es in manchen Programmiersprachen gibt.

Daher ist es unklug, Funktionen mit Nebeneffekten als Teil komplexer Ausdrücke zu verwenden. Besonders gefährlich ist es, sich in `WHERE`- und `HAVING`-Klauseln auf Nebeneffekte oder eine bestimmte Auswertungsreihenfolge zu verlassen, da diese Klauseln beim Erstellen eines Ausführungsplans umfassend umgearbeitet werden. Boolesche Ausdrücke, also Kombinationen aus `AND`, `OR` und `NOT`, können in jeder durch die Gesetze der booleschen Algebra erlaubten Weise reorganisiert werden.

Wenn es unbedingt nötig ist, die Auswertungsreihenfolge zu erzwingen, kann ein `CASE`-Konstrukt verwendet werden (siehe [Abschnitt 9.18](09_Funktionen_und_Operatoren.md#918-bedingte-ausdrücke)). Zum Beispiel ist dies ein unzuverlässiger Versuch, eine Division durch null in einer `WHERE`-Klausel zu vermeiden:

```sql
SELECT ... WHERE x > 0 AND y/x > 1.5;
```

Dies ist dagegen sicher:

```sql
SELECT ... WHERE CASE WHEN x > 0 THEN y/x > 1.5 ELSE false END;
```

Ein so verwendetes `CASE`-Konstrukt verhindert Optimierungsversuche und sollte daher nur eingesetzt werden, wenn es notwendig ist. In diesem konkreten Beispiel wäre es besser, das Problem durch `y > 1.5*x` zu umgehen.

`CASE` ist für solche Fragen allerdings kein Allheilmittel. Eine Einschränkung der oben gezeigten Technik ist, dass sie die frühe Auswertung konstanter Teilausdrücke nicht verhindert. Wie in [Abschnitt 36.7](36_SQL_erweitern.md#367-volatilitätskategorien-von-funktionen) beschrieben, können Funktionen und Operatoren, die als `IMMUTABLE` markiert sind, bereits beim Planen der Abfrage statt erst bei der Ausführung ausgewertet werden. Daher führt zum Beispiel

```sql
SELECT CASE WHEN x > 0 THEN x ELSE 1/0 END FROM tab;
```

wahrscheinlich zu einem Fehler wegen Division durch null, weil der Planner versucht, den konstanten Teilausdruck zu vereinfachen, selbst wenn jede Zeile in der Tabelle `x > 0` erfüllt und der `ELSE`-Zweig zur Laufzeit nie erreicht würde.

Dieses konkrete Beispiel mag künstlich wirken, aber verwandte Fälle ohne offensichtlich konstante Ausdrücke können in Abfragen auftreten, die innerhalb von Funktionen ausgeführt werden: Werte von Funktionsargumenten und lokalen Variablen können für Planungszwecke als Konstanten in Abfragen eingesetzt werden. Innerhalb von PL/pgSQL-Funktionen ist es zum Beispiel deutlich sicherer, eine riskante Berechnung mit einer `IF-THEN-ELSE`-Anweisung zu schützen, als sie nur in einen `CASE`-Ausdruck zu verschachteln.

Eine weitere Einschränkung derselben Art ist, dass ein `CASE` die Auswertung eines darin enthaltenen Aggregatausdrucks nicht verhindern kann, weil Aggregatausdrücke berechnet werden, bevor andere Ausdrücke in einer `SELECT`-Liste oder `HAVING`-Klausel betrachtet werden. Die folgende Abfrage kann zum Beispiel trotz scheinbaren Schutzes eine Division durch null auslösen:

```sql
SELECT CASE WHEN min(employees) > 0
            THEN avg(expenses / employees)
       END
    FROM departments;
```

Die Aggregate `min()` und `avg()` werden gleichzeitig über alle Eingabezeilen berechnet. Wenn also irgendeine Zeile `employees` gleich null hat, tritt der Fehler wegen Division durch null auf, bevor das Ergebnis von `min()` geprüft werden kann. Verwenden Sie stattdessen eine `WHERE`- oder `FILTER`-Klausel, um problematische Eingabezeilen von vornherein von der Aggregatfunktion fernzuhalten.

## 4.3. Funktionsaufrufe

PostgreSQL erlaubt, Funktionen mit benannten Parametern entweder mit positionsbezogener oder mit benannter Notation aufzurufen. Benannte Notation ist besonders nützlich für Funktionen mit vielen Parametern, weil sie die Zuordnung zwischen Parametern und tatsächlichen Argumenten expliziter und verlässlicher macht. In positionsbezogener Notation wird ein Funktionsaufruf mit den Argumentwerten in derselben Reihenfolge geschrieben, in der sie in der Funktionsdeklaration definiert sind. In benannter Notation werden die Argumente anhand ihres Namens den Funktionsparametern zugeordnet und können in beliebiger Reihenfolge geschrieben werden. Beachten Sie bei jeder Notation auch die Wirkung der Funktionsargumenttypen, die in [Abschnitt 10.3](10_Typumwandlung.md#103-funktionen) dokumentiert ist.

In beiden Notationen müssen Parameter, für die in der Funktionsdeklaration Standardwerte angegeben sind, beim Aufruf nicht geschrieben werden. Besonders nützlich ist das bei benannter Notation, weil beliebige Parameterkombinationen weggelassen werden können; bei positionsbezogener Notation können Parameter dagegen nur von rechts nach links weggelassen werden.

PostgreSQL unterstützt außerdem gemischte Notation, die positionsbezogene und benannte Notation kombiniert. In diesem Fall werden positionsbezogene Parameter zuerst geschrieben, benannte Parameter folgen danach.

Die folgenden Beispiele zeigen die Verwendung aller drei Notationen anhand dieser Funktionsdefinition:

```sql
CREATE FUNCTION concat_lower_or_upper(a text, b text, uppercase
 boolean DEFAULT false)
RETURNS text
 AS
 $$
  SELECT CASE
         WHEN $3 THEN UPPER($1 || ' ' || $2)
         ELSE LOWER($1 || ' ' || $2)
         END;
 $$
 LANGUAGE SQL IMMUTABLE STRICT;
```

Die Funktion `concat_lower_or_upper` hat zwei verpflichtende Parameter, `a` und `b`. Zusätzlich gibt es den optionalen Parameter `uppercase`, der standardmäßig `false` ist. Die Eingaben `a` und `b` werden verkettet und abhängig vom Parameter `uppercase` in Groß- oder Kleinschreibung umgewandelt. Die übrigen Details dieser Funktionsdefinition sind hier nicht wichtig; weitere Informationen finden Sie in [Kapitel 36](36_SQL_erweitern.md).

### 4.3.1. Positionsbezogene Notation verwenden

Positionsbezogene Notation ist der traditionelle Mechanismus, um Argumente an Funktionen in PostgreSQL zu übergeben. Ein Beispiel:

```sql
SELECT concat_lower_or_upper('Hello', 'World', true);
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)
```

Alle Argumente werden der Reihe nach angegeben. Das Ergebnis ist in Großbuchstaben, weil `uppercase` als `true` angegeben wurde. Ein weiteres Beispiel:

```sql
SELECT concat_lower_or_upper('Hello', 'World');
 concat_lower_or_upper
-----------------------
 hello world
(1 row)
```

Hier wird der Parameter `uppercase` weggelassen, sodass er seinen Standardwert `false` erhält und die Ausgabe in Kleinbuchstaben erfolgt. In positionsbezogener Notation können Argumente von rechts nach links weggelassen werden, solange sie Standardwerte haben.

### 4.3.2. Benannte Notation verwenden

In benannter Notation wird der Name jedes Arguments angegeben; `=>` trennt ihn vom Argumentausdruck. Zum Beispiel:

```sql
SELECT concat_lower_or_upper(a => 'Hello', b => 'World');
 concat_lower_or_upper
-----------------------
 hello world
(1 row)
```

Auch hier wurde das Argument `uppercase` weggelassen und daher implizit auf `false` gesetzt. Ein Vorteil benannter Notation ist, dass die Argumente in beliebiger Reihenfolge angegeben werden können:

```sql
SELECT concat_lower_or_upper(a => 'Hello', b => 'World', uppercase
 => true);
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)

SELECT concat_lower_or_upper(a => 'Hello', uppercase => true, b =>
 'World');
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)
```

Eine ältere Syntax auf Basis von `:=` wird aus Gründen der Rückwärtskompatibilität unterstützt:

```sql
SELECT concat_lower_or_upper(a := 'Hello', uppercase := true, b :=
 'World');
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)
```

### 4.3.3. Gemischte Notation verwenden

Die gemischte Notation kombiniert positionsbezogene und benannte Notation. Wie bereits erwähnt, dürfen benannte Argumente jedoch nicht vor positionsbezogenen Argumenten stehen. Zum Beispiel:

```sql
SELECT concat_lower_or_upper('Hello', 'World', uppercase => true);
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)
```

In dieser Abfrage werden die Argumente `a` und `b` positionsbezogen angegeben, während `uppercase` per Name angegeben wird. In diesem Beispiel bringt das außer Dokumentation wenig. Bei komplexeren Funktionen mit zahlreichen Parametern, die Standardwerte haben, kann benannte oder gemischte Notation jedoch viel Schreibarbeit sparen und Fehlerwahrscheinlichkeit reduzieren.

> Hinweis: Benannte und gemischte Aufrufnotationen können derzeit nicht beim Aufruf einer Aggregatfunktion verwendet werden. Sie funktionieren jedoch, wenn eine Aggregatfunktion als Window-Funktion verwendet wird.
