# 36. SQL erweitern

In den folgenden Abschnitten wird beschrieben, wie Sie die SQL-Abfragesprache von PostgreSQL erweitern können, indem Sie Folgendes hinzufügen:

- Funktionen (ab [Abschnitt 36.3](#363-benutzerdefinierte-funktionen))
- Aggregate (ab [Abschnitt 36.12](#3612-benutzerdefinierte-aggregate))
- Datentypen (ab [Abschnitt 36.13](#3613-benutzerdefinierte-typen))
- Operatoren (ab [Abschnitt 36.14](#3614-benutzerdefinierte-operatoren))
- Operatorklassen für Indizes (ab [Abschnitt 36.16](#3616-erweiterungen-an-indizes-anbinden))
- Pakete zusammengehöriger Objekte (ab [Abschnitt 36.17](#3617-zusammengehörige-objekte-in-einer-erweiterung-paketieren))

## 36.1. Wie Erweiterbarkeit funktioniert

PostgreSQL ist erweiterbar, weil sein Betrieb kataloggesteuert ist. Wenn Sie mit herkömmlichen relationalen Datenbanksystemen vertraut sind, wissen Sie, dass sie Informationen über Datenbanken, Tabellen, Spalten usw. in sogenannten Systemkatalogen speichern. Manche Systeme nennen dies das Data Dictionary. Die Kataloge erscheinen dem Benutzer wie gewöhnliche Tabellen, aber das DBMS speichert darin seine interne Buchführung. Ein wesentlicher Unterschied zwischen PostgreSQL und herkömmlichen relationalen Datenbanksystemen besteht darin, dass PostgreSQL viel mehr Informationen in seinen Katalogen speichert: nicht nur Informationen über Tabellen und Spalten, sondern auch über Datentypen, Funktionen, Zugriffsmethoden und so weiter. Diese Tabellen können vom Benutzer geändert werden, und da PostgreSQL seinen Betrieb auf diese Tabellen stützt, bedeutet das, dass PostgreSQL von Benutzern erweitert werden kann. Herkömmliche Datenbanksysteme können dagegen nur erweitert werden, indem fest einprogrammierte Prozeduren im Quellcode geändert oder speziell vom DBMS-Hersteller geschriebene Module geladen werden.

Der PostgreSQL-Server kann außerdem durch dynamisches Laden benutzergeschriebenen Code in sich aufnehmen. Das heißt, der Benutzer kann eine Objektcodedatei, zum Beispiel eine Shared Library, angeben, die einen neuen Typ oder eine neue Funktion implementiert, und PostgreSQL lädt sie bei Bedarf. In SQL geschriebener Code lässt sich noch einfacher zum Server hinzufügen. Diese Fähigkeit, den Betrieb »im laufenden Betrieb« zu ändern, macht PostgreSQL besonders geeignet für das schnelle Prototyping neuer Anwendungen und Speicherstrukturen.

## 36.2. Das PostgreSQL-Typsystem

PostgreSQL-Datentypen lassen sich in Basistypen, Containertypen, Domains und Pseudotypen einteilen.

### 36.2.1. Basistypen

Basistypen sind Typen wie `integer`, die unterhalb der Ebene der SQL-Sprache implementiert sind, typischerweise in einer Low-Level-Sprache wie C. Sie entsprechen im Allgemeinen dem, was häufig als abstrakte Datentypen bezeichnet wird. PostgreSQL kann mit solchen Typen nur über Funktionen arbeiten, die vom Benutzer bereitgestellt werden, und versteht das Verhalten solcher Typen nur so weit, wie der Benutzer es beschreibt. Die eingebauten Basistypen werden in [Kapitel 8](08_Datentypen.md) beschrieben.

Aufzählungstypen (`enum`) können als Unterkategorie der Basistypen betrachtet werden. Der Hauptunterschied besteht darin, dass sie allein mit SQL-Befehlen erstellt werden können, ohne Low-Level-Programmierung. Weitere Informationen finden Sie in [Abschnitt 8.7](08_Datentypen.md#87-aufzählungstypen).

### 36.2.2. Containertypen

PostgreSQL besitzt drei Arten von »Container«-Typen, also Typen, die mehrere Werte anderer Typen enthalten. Dies sind Arrays, zusammengesetzte Typen und Bereichstypen.

Arrays können mehrere Werte aufnehmen, die alle denselben Typ haben. Für jeden Basistyp, zusammengesetzten Typ, Bereichstyp und Domain-Typ wird automatisch ein Array-Typ erstellt. Es gibt jedoch keine Arrays von Arrays. Für das Typsystem sind mehrdimensionale Arrays dasselbe wie eindimensionale Arrays. Weitere Informationen finden Sie in [Abschnitt 8.15](08_Datentypen.md#815-arrays).

Zusammengesetzte Typen oder Row-Typen werden erstellt, wann immer der Benutzer eine Tabelle erstellt. Es ist außerdem möglich, mit `CREATE TYPE` einen »eigenständigen« zusammengesetzten Typ ohne zugehörige Tabelle zu definieren. Ein zusammengesetzter Typ ist einfach eine Liste von Typen mit zugehörigen Feldnamen. Ein Wert eines zusammengesetzten Typs ist eine Zeile oder ein Datensatz von Feldwerten. Weitere Informationen finden Sie in [Abschnitt 8.16](08_Datentypen.md#816-zusammengesetzte-typen).

Ein Bereichstyp kann zwei Werte desselben Typs aufnehmen, nämlich die untere und obere Grenze des Bereichs. Bereichstypen werden vom Benutzer erstellt, auch wenn einige eingebaute Bereichstypen existieren. Weitere Informationen finden Sie in [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen).

### 36.2.3. Domains

Eine Domain basiert auf einem bestimmten zugrunde liegenden Typ und ist für viele Zwecke mit diesem zugrunde liegenden Typ austauschbar. Eine Domain kann jedoch Constraints haben, die ihre gültigen Werte auf eine Teilmenge dessen beschränken, was der zugrunde liegende Typ erlauben würde. Domains werden mit dem SQL-Befehl `CREATE DOMAIN` erstellt. Weitere Informationen finden Sie in [Abschnitt 8.18](08_Datentypen.md#818-domänentypen).

### 36.2.4. Pseudotypen

Es gibt einige »Pseudotypen« für besondere Zwecke. Pseudotypen können nicht als Spalten von Tabellen oder als Komponenten von Containertypen erscheinen, aber sie können verwendet werden, um die Argument- und Ergebnistypen von Funktionen zu deklarieren. Dies stellt innerhalb des Typsystems einen Mechanismus bereit, um besondere Klassen von Funktionen zu kennzeichnen. Tabelle 8.27 listet die vorhandenen Pseudotypen auf.

### 36.2.5. Polymorphe Typen

Einige besonders interessante Pseudotypen sind die polymorphen Typen, die zum Deklarieren polymorpher Funktionen verwendet werden. Diese mächtige Funktion erlaubt es, dass eine einzige Funktionsdefinition mit vielen verschiedenen Datentypen arbeitet, wobei die konkreten Datentypen durch die Datentypen bestimmt werden, die in einem bestimmten Aufruf tatsächlich übergeben werden. Die polymorphen Typen sind in Tabelle 36.1 gezeigt. Einige Beispiele für ihre Verwendung erscheinen in [Abschnitt 36.5.11](#36511-polymorphe-sqlfunktionen).

**Tabelle 36.1. Polymorphe Typen**

| Name | Familie | Beschreibung |
|---|---|---|
| `anyelement` | Einfach | Gibt an, dass eine Funktion einen beliebigen Datentyp akzeptiert |
| `anyarray` | Einfach | Gibt an, dass eine Funktion einen beliebigen Array-Datentyp akzeptiert |
| `anynonarray` | Einfach | Gibt an, dass eine Funktion einen beliebigen Nicht-Array-Datentyp akzeptiert |
| `anyenum` | Einfach | Gibt an, dass eine Funktion einen beliebigen Enum-Datentyp akzeptiert; siehe [Abschnitt 8.7](08_Datentypen.md#87-aufzählungstypen) |
| `anyrange` | Einfach | Gibt an, dass eine Funktion einen beliebigen Bereichsdatentyp akzeptiert; siehe [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen) |
| `anymultirange` | Einfach | Gibt an, dass eine Funktion einen beliebigen Multirange-Datentyp akzeptiert; siehe [Abschnitt 8.17](08_Datentypen.md#817-bereichstypen) |
| `anycompatible` | Gemeinsam | Gibt an, dass eine Funktion einen beliebigen Datentyp akzeptiert, wobei mehrere Argumente automatisch auf einen gemeinsamen Datentyp hochgestuft werden |
| `anycompatiblearray` | Gemeinsam | Gibt an, dass eine Funktion einen beliebigen Array-Datentyp akzeptiert, wobei mehrere Argumente automatisch auf einen gemeinsamen Datentyp hochgestuft werden |
| `anycompatiblenonarray` | Gemeinsam | Gibt an, dass eine Funktion einen beliebigen Nicht-Array-Datentyp akzeptiert, wobei mehrere Argumente automatisch auf einen gemeinsamen Datentyp hochgestuft werden |
| `anycompatiblerange` | Gemeinsam | Gibt an, dass eine Funktion einen beliebigen Bereichsdatentyp akzeptiert, wobei mehrere Argumente automatisch auf einen gemeinsamen Datentyp hochgestuft werden |
| `anycompatiblemultirange` | Gemeinsam | Gibt an, dass eine Funktion einen beliebigen Multirange-Datentyp akzeptiert, wobei mehrere Argumente automatisch auf einen gemeinsamen Datentyp hochgestuft werden |

Polymorphe Argumente und Ergebnisse sind miteinander verknüpft und werden zu konkreten Datentypen aufgelöst, wenn eine Abfrage, die eine polymorphe Funktion aufruft, geparst wird. Wenn es mehr als ein polymorphes Argument gibt, müssen die tatsächlichen Datentypen der Eingabewerte wie unten beschrieben zusammenpassen. Wenn der Ergebnistyp der Funktion polymorph ist oder die Funktion Ausgabeparameter polymorpher Typen hat, werden die Typen dieser Ergebnisse aus den tatsächlichen Typen der polymorphen Eingaben abgeleitet, wie unten beschrieben.

Für die »einfache« Familie polymorpher Typen funktionieren die Abgleichs- und Ableitungsregeln so:

Jede Position, ob Argument oder Rückgabewert, die als `anyelement` deklariert ist, darf einen beliebigen konkreten tatsächlichen Datentyp haben; in einem bestimmten Aufruf müssen aber alle diese Positionen denselben tatsächlichen Typ haben. Jede als `anyarray` deklarierte Position kann einen beliebigen Array-Datentyp haben, aber auch hier müssen alle denselben Typ haben. Entsprechend müssen Positionen, die als `anyrange` deklariert sind, alle denselben Bereichstyp haben. Dasselbe gilt für `anymultirange`.

Wenn außerdem Positionen als `anyarray` und andere als `anyelement` deklariert sind, muss der tatsächliche Array-Typ in den `anyarray`-Positionen ein Array sein, dessen Elemente denselben Typ haben wie die `anyelement`-Positionen. `anynonarray` wird genau wie `anyelement` behandelt, fügt aber die zusätzliche Einschränkung hinzu, dass der tatsächliche Typ kein Array-Typ sein darf. `anyenum` wird ebenfalls genau wie `anyelement` behandelt, fügt aber die zusätzliche Einschränkung hinzu, dass der tatsächliche Typ ein Enum-Typ sein muss.

Ähnlich gilt: Wenn Positionen als `anyrange` und andere als `anyelement` oder `anyarray` deklariert sind, muss der tatsächliche Bereichstyp in den `anyrange`-Positionen ein Bereich sein, dessen Subtyp derselbe Typ ist wie in den `anyelement`-Positionen und derselbe wie der Elementtyp der `anyarray`-Positionen. Wenn Positionen als `anymultirange` deklariert sind, muss ihr tatsächlicher Multirange-Typ Bereiche enthalten, die zu als `anyrange` deklarierten Parametern passen, sowie Basiselemente, die zu als `anyelement` und `anyarray` deklarierten Parametern passen.

Wenn also mehr als eine Argumentposition mit einem polymorphen Typ deklariert ist, sind im Ergebnis nur bestimmte Kombinationen tatsächlicher Argumenttypen erlaubt. Eine Funktion, die als `equal(anyelement, anyelement)` deklariert ist, nimmt zum Beispiel zwei beliebige Eingabewerte an, solange beide denselben Datentyp haben.

Wenn der Rückgabewert einer Funktion als polymorpher Typ deklariert ist, muss es mindestens eine Argumentposition geben, die ebenfalls polymorph ist, und die für die polymorphen Argumente bereitgestellten tatsächlichen Datentypen bestimmen den tatsächlichen Ergebnistyp dieses Aufrufs. Wenn es beispielsweise noch keinen Array-Subskriptmechanismus gäbe, könnte man eine Funktion definieren, die Subskription als `subscript(anyarray, integer) returns anyelement` implementiert. Diese Deklaration beschränkt das tatsächliche erste Argument auf einen Array-Typ und erlaubt dem Parser, den korrekten Ergebnistyp aus dem tatsächlichen Typ des ersten Arguments abzuleiten. Ein weiteres Beispiel ist, dass eine als `f(anyarray) returns anyenum` deklarierte Funktion nur Arrays von Enum-Typen akzeptiert.

In den meisten Fällen kann der Parser den tatsächlichen Datentyp für einen polymorphen Ergebnistyp aus Argumenten ableiten, die zu einem anderen polymorphen Typ derselben Familie gehören; beispielsweise kann `anyarray` aus `anyelement` abgeleitet werden oder umgekehrt. Eine Ausnahme ist, dass ein polymorphes Ergebnis vom Typ `anyrange` ein Argument vom Typ `anyrange` benötigt; es kann nicht aus `anyarray`- oder `anyelement`-Argumenten abgeleitet werden. Das liegt daran, dass es mehrere Bereichstypen mit demselben Subtyp geben könnte.

Beachten Sie, dass `anynonarray` und `anyenum` keine separaten Typvariablen darstellen; sie sind derselbe Typ wie `anyelement`, nur mit einer zusätzlichen Einschränkung. Eine Funktionsdeklaration als `f(anyelement, anyenum)` ist zum Beispiel äquivalent zu einer Deklaration als `f(anyenum, anyenum)`: Beide tatsächlichen Argumente müssen derselbe Enum-Typ sein.

Für die »gemeinsame« Familie polymorpher Typen funktionieren die Abgleichs- und Ableitungsregeln ungefähr wie bei der »einfachen« Familie, mit einem wichtigen Unterschied: Die tatsächlichen Typen der Argumente müssen nicht identisch sein, solange sie implizit in einen einzigen gemeinsamen Typ gecastet werden können. Der gemeinsame Typ wird nach denselben Regeln ausgewählt wie bei `UNION` und verwandten Konstrukten; siehe [Abschnitt 10.5](10_Typumwandlung.md#105-union-case-und-verwandte-konstrukte). Bei der Auswahl des gemeinsamen Typs werden die tatsächlichen Typen von `anycompatible`- und `anycompatiblenonarray`-Eingaben, die Array-Elementtypen von `anycompatiblearray`-Eingaben, die Bereichssubtypen von `anycompatiblerange`-Eingaben und die Multirange-Subtypen von `anycompatiblemultirange`-Eingaben berücksichtigt. Wenn `anycompatiblenonarray` vorhanden ist, muss der gemeinsame Typ ein Nicht-Array-Typ sein. Sobald ein gemeinsamer Typ ermittelt wurde, werden Argumente an `anycompatible`- und `anycompatiblenonarray`-Positionen automatisch auf diesen Typ gecastet, und Argumente an `anycompatiblearray`-Positionen werden automatisch auf den Array-Typ für diesen Typ gecastet.

Da es keine Möglichkeit gibt, einen Bereichstyp nur anhand seines Subtyps auszuwählen, erfordert die Verwendung von `anycompatiblerange` und/oder `anycompatiblemultirange`, dass alle mit diesem Typ deklarierten Argumente denselben tatsächlichen Bereichs- und/oder Multirange-Typ haben und dass der Subtyp dieses Typs mit dem ausgewählten gemeinsamen Typ übereinstimmt, sodass kein Cast der Bereichswerte erforderlich ist. Wie bei `anyrange` und `anymultirange` erfordert die Verwendung von `anycompatiblerange` und `anymultirange` als Funktionsergebnistyp, dass es ein Argument vom Typ `anycompatiblerange` oder `anycompatiblemultirange` gibt.

Beachten Sie, dass es keinen Typ `anycompatibleenum` gibt. Ein solcher Typ wäre nicht besonders nützlich, da es normalerweise keine impliziten Casts zu Enum-Typen gibt; folglich gäbe es keine Möglichkeit, einen gemeinsamen Typ für unterschiedliche Enum-Eingaben aufzulösen.

Die »einfache« und die »gemeinsame« polymorphe Familie stellen zwei unabhängige Mengen von Typvariablen dar. Betrachten Sie zum Beispiel:

```sql
CREATE FUNCTION myfunc(a anyelement, b anyelement,
                       c anycompatible, d anycompatible)
RETURNS anycompatible AS ...
```

In einem tatsächlichen Aufruf dieser Funktion müssen die ersten beiden Eingaben exakt denselben Typ haben. Die letzten beiden Eingaben müssen auf einen gemeinsamen Typ hochgestuft werden können, aber dieser Typ muss nichts mit dem Typ der ersten beiden Eingaben zu tun haben. Das Ergebnis hat den gemeinsamen Typ der letzten beiden Eingaben.

Eine variadische Funktion, also eine Funktion mit variabler Anzahl von Argumenten wie in [Abschnitt 36.5.6](#3656-sqlfunktionen-mit-variabler-argumentanzahl), kann polymorph sein: Dazu wird ihr letzter Parameter als `VARIADIC anyarray` oder `VARIADIC anycompatiblearray` deklariert. Für den Argumentabgleich und die Bestimmung des tatsächlichen Ergebnistyps verhält sich eine solche Funktion genauso, als hätten Sie die passende Anzahl von `anynonarray`- oder `anycompatiblenonarray`-Parametern geschrieben.

## 36.3. Benutzerdefinierte Funktionen

PostgreSQL stellt vier Arten von Funktionen bereit:

- Query-Language-Funktionen, also in SQL geschriebene Funktionen ([Abschnitt 36.5](#365-querylanguagesqlfunktionen))
- Funktionen in prozeduralen Sprachen, zum Beispiel in PL/pgSQL oder PL/Tcl geschriebene Funktionen ([Abschnitt 36.8](#368-funktionen-in-prozeduralen-sprachen))
- interne Funktionen ([Abschnitt 36.9](#369-interne-funktionen))
- C-Sprachfunktionen ([Abschnitt 36.10](#3610-csprachfunktionen))

Jede Art von Funktion kann Basistypen, zusammengesetzte Typen oder Kombinationen davon als Argumente (Parameter) annehmen. Außerdem kann jede Art von Funktion einen Basistyp oder einen zusammengesetzten Typ zurückgeben. Funktionen können auch so definiert werden, dass sie Mengen von Basis- oder zusammengesetzten Werten zurückgeben.

Viele Arten von Funktionen können bestimmte Pseudotypen, etwa polymorphe Typen, annehmen oder zurückgeben; die verfügbaren Möglichkeiten unterscheiden sich jedoch. Einzelheiten finden Sie in der Beschreibung der jeweiligen Funktionsart.

SQL-Funktionen sind am einfachsten zu definieren, deshalb beginnen wir mit ihnen. Die meisten Konzepte, die für SQL-Funktionen vorgestellt werden, übertragen sich auf die anderen Funktionstypen.

Im gesamten Kapitel kann es hilfreich sein, die Referenzseite des Befehls `CREATE FUNCTION` anzusehen, um die Beispiele besser zu verstehen. Einige Beispiele aus diesem Kapitel finden Sie in `funcs.sql` und `funcs.c` im Verzeichnis `src/tutorial` der PostgreSQL-Quelldistribution.

## 36.4. Benutzerdefinierte Prozeduren

Eine Prozedur ist ein Datenbankobjekt, das einer Funktion ähnelt. Die wichtigsten Unterschiede sind:

- Prozeduren werden mit dem Befehl `CREATE PROCEDURE` definiert, nicht mit `CREATE FUNCTION`.
- Prozeduren geben keinen Funktionswert zurück; daher hat `CREATE PROCEDURE` keine `RETURNS`-Klausel. Prozeduren können stattdessen über Ausgabeparameter Daten an ihre Aufrufer zurückgeben.
- Während eine Funktion als Teil einer Abfrage oder eines DML-Befehls aufgerufen wird, wird eine Prozedur eigenständig mit dem Befehl `CALL` aufgerufen.
- Eine Prozedur kann während ihrer Ausführung Transaktionen committen oder zurückrollen und danach automatisch eine neue Transaktion beginnen, solange der aufrufende `CALL`-Befehl nicht Teil eines expliziten Transaktionsblocks ist. Eine Funktion kann das nicht.
- Bestimmte Funktionsattribute, etwa Strictness, gelten nicht für Prozeduren. Diese Attribute steuern, wie die Funktion in einer Abfrage verwendet wird; das ist für Prozeduren nicht relevant.

Die Erklärungen in den folgenden Abschnitten zur Definition benutzerdefinierter Funktionen gelten, abgesehen von den oben genannten Punkten, auch für Prozeduren.

Funktionen und Prozeduren werden zusammen auch als Routinen bezeichnet. Es gibt Befehle wie `ALTER ROUTINE` und `DROP ROUTINE`, die auf Funktionen und Prozeduren angewendet werden können, ohne wissen zu müssen, um welche Art es sich handelt. Beachten Sie jedoch, dass es keinen Befehl `CREATE ROUTINE` gibt.

## 36.5. Query-Language(SQL)-Funktionen

SQL-Funktionen führen eine beliebige Liste von SQL-Anweisungen aus und geben das Ergebnis der letzten Abfrage in der Liste zurück. Im einfachen Fall, der keine Menge zurückgibt, wird die erste Zeile des Ergebnisses der letzten Abfrage zurückgegeben. Beachten Sie, dass »die erste Zeile« eines mehrzeiligen Ergebnisses ohne `ORDER BY` nicht wohldefiniert ist. Wenn die letzte Abfrage gar keine Zeilen zurückgibt, wird der Nullwert zurückgegeben.

Alternativ kann eine SQL-Funktion so deklariert werden, dass sie eine Menge, also mehrere Zeilen, zurückgibt, indem der Rückgabetyp der Funktion als `SETOF sometype` angegeben oder gleichwertig als `RETURNS TABLE(columns)` deklariert wird. In diesem Fall werden alle Zeilen des Ergebnisses der letzten Abfrage zurückgegeben. Weitere Einzelheiten folgen weiter unten.

Der Rumpf einer SQL-Funktion muss eine Liste von SQL-Anweisungen sein, die durch Semikolons getrennt sind. Ein Semikolon nach der letzten Anweisung ist optional. Sofern die Funktion nicht so deklariert ist, dass sie `void` zurückgibt, muss die letzte Anweisung ein `SELECT` sein oder ein `INSERT`, `UPDATE`, `DELETE` oder `MERGE` mit einer `RETURNING`-Klausel.

Jede Sammlung von Befehlen der SQL-Sprache kann zusammengepackt und als Funktion definiert werden. Neben `SELECT`-Abfragen können die Befehle Datenänderungsabfragen (`INSERT`, `UPDATE`, `DELETE` und `MERGE`) sowie andere SQL-Befehle enthalten. Transaktionssteuerungsbefehle wie `COMMIT` und `SAVEPOINT` sowie einige Utility-Befehle wie `VACUUM` können Sie in SQL-Funktionen jedoch nicht verwenden. Der letzte Befehl muss allerdings ein `SELECT` sein oder eine `RETURNING`-Klausel besitzen, die das zurückgibt, was als Rückgabetyp der Funktion angegeben ist. Wenn Sie dagegen eine SQL-Funktion definieren möchten, die Aktionen ausführt, aber keinen nützlichen Rückgabewert hat, können Sie sie als `void` zurückgebend definieren. Diese Funktion entfernt zum Beispiel Zeilen mit negativen Gehältern aus der Tabelle `emp`:

```sql
CREATE FUNCTION clean_emp() RETURNS void AS '
    DELETE FROM emp
        WHERE salary < 0;
' LANGUAGE SQL;

SELECT clean_emp();
```

```text
 clean_emp
-----------

(1 row)
```

Sie können dies auch als Prozedur schreiben und damit die Frage des Rückgabetyps vermeiden. Zum Beispiel:

```sql
CREATE PROCEDURE clean_emp() AS '
    DELETE FROM emp
        WHERE salary < 0;
' LANGUAGE SQL;

CALL clean_emp();
```

In einfachen Fällen wie diesem ist der Unterschied zwischen einer Funktion, die `void` zurückgibt, und einer Prozedur vor allem stilistisch. Prozeduren bieten jedoch zusätzliche Funktionalität wie Transaktionssteuerung, die in Funktionen nicht verfügbar ist. Außerdem entsprechen Prozeduren dem SQL-Standard, während die Rückgabe von `void` eine PostgreSQL-Erweiterung ist.

Die Syntax des Befehls `CREATE FUNCTION` verlangt, dass der Funktionsrumpf als Zeichenkettenkonstante geschrieben wird. Für diese Zeichenkettenkonstante ist es meist am bequemsten, Dollar-Quoting zu verwenden; siehe [Abschnitt 4.1.2.4](04_SQL_Syntax.md#4124-dollarquotedzeichenkettenkonstanten). Wenn Sie stattdessen die normale Syntax mit einfach gequoteten Zeichenkettenkonstanten verwenden, müssen Sie einfache Anführungszeichen (`'`) und Backslashes (`\`) im Funktionsrumpf verdoppeln, sofern Escape-Zeichenkettensyntax angenommen wird; siehe [Abschnitt 4.1.2.1](04_SQL_Syntax.md#4121-zeichenkettenkonstanten).

### 36.5.1. Argumente für SQL-Funktionen

Argumente einer SQL-Funktion können im Funktionsrumpf entweder über Namen oder über Nummern referenziert werden. Beispiele für beide Methoden folgen weiter unten.

Um einen Namen zu verwenden, deklarieren Sie das Funktionsargument mit einem Namen und schreiben diesen Namen dann im Funktionsrumpf. Wenn der Argumentname mit einem Spaltennamen im aktuellen SQL-Befehl innerhalb der Funktion übereinstimmt, hat der Spaltenname Vorrang. Um dies zu übersteuern, qualifizieren Sie den Argumentnamen mit dem Namen der Funktion selbst, also `function_name.argument_name`. Wenn dies wiederum mit einem qualifizierten Spaltennamen kollidieren würde, gewinnt erneut der Spaltenname. Sie können die Mehrdeutigkeit vermeiden, indem Sie innerhalb des SQL-Befehls einen anderen Alias für die Tabelle wählen.

Beim älteren numerischen Ansatz werden Argumente mit der Syntax `$n` referenziert: `$1` verweist auf das erste Eingabeargument, `$2` auf das zweite und so weiter. Das funktioniert unabhängig davon, ob das jeweilige Argument mit einem Namen deklariert wurde.

Wenn ein Argument einen zusammengesetzten Typ hat, kann die Punktnotation, zum Beispiel `argname.fieldname` oder `$1.fieldname`, verwendet werden, um auf Attribute des Arguments zuzugreifen. Auch hier müssen Sie den Namen des Arguments gegebenenfalls mit dem Funktionsnamen qualifizieren, damit die Form mit Argumentnamen eindeutig wird.

Argumente von SQL-Funktionen können nur als Datenwerte verwendet werden, nicht als Bezeichner. Daher ist zum Beispiel Folgendes sinnvoll:

```sql
INSERT INTO mytable VALUES ($1);
```

aber dies funktioniert nicht:

```sql
INSERT INTO $1 VALUES (42);
```

> Hinweis: Die Möglichkeit, SQL-Funktionsargumente über Namen zu referenzieren, wurde in PostgreSQL 9.2 hinzugefügt. Funktionen, die auf älteren Servern verwendet werden sollen, müssen die `$n`-Notation verwenden.

### 36.5.2. SQL-Funktionen auf Basistypen

Die einfachste mögliche SQL-Funktion hat keine Argumente und gibt einfach einen Basistyp zurück, etwa `integer`:

```sql
CREATE FUNCTION one() RETURNS integer AS $$
    SELECT 1 AS result;
$$ LANGUAGE SQL;

-- Alternative Syntax für das Zeichenkettenliteral:
CREATE FUNCTION one() RETURNS integer AS '
    SELECT 1 AS result;
' LANGUAGE SQL;

SELECT one();
```

```text
 one
-----
```

Beachten Sie, dass wir im Funktionsrumpf einen Spaltenalias für das Ergebnis der Funktion definiert haben, nämlich `result`, dieser Spaltenalias aber außerhalb der Funktion nicht sichtbar ist. Daher wird das Ergebnis mit `one` statt mit `result` beschriftet.

Fast genauso einfach ist es, SQL-Funktionen zu definieren, die Basistypen als Argumente annehmen:

```sql
CREATE FUNCTION add_em(x integer, y integer) RETURNS integer AS $$
    SELECT x + y;
$$ LANGUAGE SQL;

SELECT add_em(1, 2) AS answer;
```

```text
 answer
--------
```

Alternativ könnten wir auf Namen für die Argumente verzichten und Nummern verwenden:

```sql
CREATE FUNCTION add_em(integer, integer) RETURNS integer AS $$
    SELECT $1 + $2;
$$ LANGUAGE SQL;

SELECT add_em(1, 2) AS answer;
```

```text
 answer
--------
```

Hier ist eine nützlichere Funktion, die verwendet werden könnte, um ein Bankkonto zu belasten:

```sql
CREATE FUNCTION tf1 (accountno integer, debit numeric) RETURNS
 numeric AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tf1.accountno;
    SELECT 1;
$$ LANGUAGE SQL;
```

Ein Benutzer könnte diese Funktion wie folgt ausführen, um Konto 17 mit 100,00 Dollar zu belasten:

```sql
SELECT tf1(17, 100.0);
```

In diesem Beispiel haben wir für das erste Argument den Namen `accountno` gewählt, aber das ist zugleich der Name einer Spalte in der Tabelle `bank`. Innerhalb des `UPDATE`-Befehls verweist `accountno` auf die Spalte `bank.accountno`, daher muss `tf1.accountno` verwendet werden, um auf das Argument zu verweisen. Natürlich könnten wir dies vermeiden, indem wir einen anderen Namen für das Argument verwenden.

In der Praxis möchte man wahrscheinlich ein nützlicheres Ergebnis als die Konstante `1` aus der Funktion erhalten, daher ist eine wahrscheinlichere Definition:

```sql
CREATE FUNCTION tf1 (accountno integer, debit numeric) RETURNS
 numeric AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tf1.accountno;
    SELECT balance FROM bank WHERE accountno = tf1.accountno;
$$ LANGUAGE SQL;
```

Dies passt den Kontostand an und gibt den neuen Kontostand zurück. Dasselbe könnte mit `RETURNING` in einem einzigen Befehl erledigt werden:

```sql
CREATE FUNCTION tf1 (accountno integer, debit numeric) RETURNS
 numeric AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tf1.accountno
    RETURNING balance;
$$ LANGUAGE SQL;
```

Wenn das abschließende `SELECT` oder die abschließende `RETURNING`-Klausel in einer SQL-Funktion nicht exakt den deklarierten Ergebnistyp der Funktion zurückgibt, castet PostgreSQL den Wert automatisch auf den erforderlichen Typ, sofern dies mit einem impliziten Cast oder einem Zuweisungscast möglich ist. Andernfalls müssen Sie einen expliziten Cast schreiben. Angenommen zum Beispiel, die frühere Funktion `add_em` soll stattdessen den Typ `float8` zurückgeben. Es genügt, Folgendes zu schreiben:

```sql
CREATE FUNCTION add_em(integer, integer) RETURNS float8 AS $$
    SELECT $1 + $2;
$$ LANGUAGE SQL;
```

weil die Ganzzahlsumme implizit nach `float8` gecastet werden kann. Weitere Informationen zu Casts finden Sie in [Kapitel 10](10_Typumwandlung.md) oder bei `CREATE CAST`.

### 36.5.3. SQL-Funktionen auf zusammengesetzten Typen

Beim Schreiben von Funktionen mit Argumenten zusammengesetzter Typen müssen wir nicht nur angeben, welches Argument wir wollen, sondern auch das gewünschte Attribut (Feld) dieses Arguments. Angenommen zum Beispiel, `emp` ist eine Tabelle mit Mitarbeiterdaten und damit zugleich der Name des zusammengesetzten Typs jeder Tabellenzeile. Hier ist eine Funktion `double_salary`, die berechnet, wie hoch das Gehalt einer Person wäre, wenn es verdoppelt würde:

```sql
CREATE TABLE emp (
    name        text,
    salary      numeric,
    age         integer,
    cubicle     point
);

INSERT INTO emp VALUES ('Bill', 4200, 45, '(2,1)');

CREATE FUNCTION double_salary(emp) RETURNS numeric AS $$
    SELECT $1.salary * 2 AS salary;
$$ LANGUAGE SQL;

SELECT name, double_salary(emp.*) AS dream
    FROM emp
    WHERE emp.cubicle ~= point '(2,1)';
```

```text
 name | dream
------+-------
 Bill | 8400
```

Beachten Sie die Verwendung der Syntax `$1.salary`, um ein Feld des Argument-Zeilenwerts auszuwählen. Beachten Sie außerdem, dass der aufrufende `SELECT`-Befehl `table_name.*` verwendet, um die gesamte aktuelle Zeile einer Tabelle als zusammengesetzten Wert auszuwählen. Alternativ kann die Tabellenzeile auch nur mit dem Tabellennamen referenziert werden:

```sql
SELECT name, double_salary(emp) AS dream
    FROM emp
    WHERE emp.cubicle ~= point '(2,1)';
```

Diese Verwendung ist jedoch veraltet, weil sie leicht verwirrt. Einzelheiten zu diesen beiden Schreibweisen für den zusammengesetzten Wert einer Tabellenzeile finden Sie in [Abschnitt 8.16.5](08_Datentypen.md#8165-zusammengesetzte-typen-in-abfragen-verwenden).

Manchmal ist es praktisch, einen zusammengesetzten Argumentwert direkt zu konstruieren. Das geht mit dem `ROW`-Konstrukt. Wir könnten zum Beispiel die Daten anpassen, die an die Funktion übergeben werden:

```sql
SELECT name, double_salary(ROW(name, salary*1.1, age, cubicle)) AS
 dream
    FROM emp;
```

Es ist auch möglich, eine Funktion zu bauen, die einen zusammengesetzten Typ zurückgibt. Dies ist ein Beispiel für eine Funktion, die eine einzelne `emp`-Zeile zurückgibt:

```sql
CREATE FUNCTION new_emp() RETURNS emp AS $$
    SELECT text 'None' AS name,
        1000.0 AS salary,
        25 AS age,
        point '(2,2)' AS cubicle;
$$ LANGUAGE SQL;
```

In diesem Beispiel haben wir jedes Attribut mit einem konstanten Wert angegeben, aber anstelle dieser Konstanten hätte auch jede beliebige Berechnung stehen können.

Beachten Sie zwei wichtige Punkte bei der Definition der Funktion:

- Die Reihenfolge der Auswahlliste in der Abfrage muss exakt der Reihenfolge entsprechen, in der die Spalten im zusammengesetzten Typ erscheinen. Die Benennung der Spalten, wie oben gezeigt, ist für das System unerheblich.
- Wir müssen sicherstellen, dass der Typ jedes Ausdrucks in den Typ der entsprechenden Spalte des zusammengesetzten Typs gecastet werden kann. Andernfalls erhalten wir Fehler wie diesen:

```text
ERROR: return type mismatch in function declared to return emp
DETAIL: Final statement returns text instead of point at column
 4.
```

Wie im Fall der Basistypen fügt das System keine expliziten Casts automatisch ein, sondern nur implizite Casts oder Zuweisungscasts.

Eine andere Möglichkeit, dieselbe Funktion zu definieren, ist:

```sql
CREATE FUNCTION new_emp() RETURNS emp AS $$
    SELECT ROW('None', 1000.0, 25, '(2,2)')::emp;
$$ LANGUAGE SQL;
```

Hier haben wir ein `SELECT` geschrieben, das nur eine einzelne Spalte des richtigen zusammengesetzten Typs zurückgibt. In dieser Situation ist das nicht wirklich besser, aber in manchen Fällen ist es eine praktische Alternative, zum Beispiel wenn wir das Ergebnis berechnen müssen, indem wir eine andere Funktion aufrufen, die den gewünschten zusammengesetzten Wert zurückgibt. Ein weiteres Beispiel: Wenn wir eine Funktion schreiben wollen, die eine Domain über einem zusammengesetzten Typ statt eines einfachen zusammengesetzten Typs zurückgibt, ist es immer erforderlich, sie so zu schreiben, dass sie eine einzelne Spalte zurückgibt, da es keine Möglichkeit gibt, eine Koerzierung des gesamten Zeilenergebnisses auszulösen.

Wir könnten diese Funktion direkt aufrufen, indem wir sie entweder in einem Wertausdruck verwenden:

```sql
SELECT new_emp();
```

```text
         new_emp
--------------------------
 (None,1000.0,25,"(2,2)")
```

oder indem wir sie als Tabellenfunktion aufrufen:

```sql
SELECT * FROM new_emp();
```

```text
 name | salary | age | cubicle
------+--------+-----+---------
 None | 1000.0 | 25 | (2,2)
```

Die zweite Variante wird in [Abschnitt 36.5.8](#3658-sqlfunktionen-als-tabellenquellen) ausführlicher beschrieben.

Wenn Sie eine Funktion verwenden, die einen zusammengesetzten Typ zurückgibt, möchten Sie vielleicht nur ein Feld (Attribut) aus ihrem Ergebnis haben. Das geht mit einer Syntax wie dieser:

```sql
SELECT (new_emp()).name;
```

```text
 name
------
 None
```

Die zusätzlichen Klammern sind nötig, damit der Parser nicht durcheinanderkommt. Wenn Sie es ohne sie versuchen, erhalten Sie etwa Folgendes:

```sql
SELECT new_emp().name;
```

```text
ERROR: syntax error at or near "."
LINE 1: SELECT new_emp().name;
                        ^
```

Eine weitere Möglichkeit besteht darin, funktionale Notation zu verwenden, um ein Attribut zu extrahieren:

```sql
SELECT name(new_emp());
```

```text
 name
------
 None
```

Wie in [Abschnitt 8.16.5](08_Datentypen.md#8165-zusammengesetzte-typen-in-abfragen-verwenden) erklärt, sind Feldnotation und funktionale Notation gleichwertig.

Eine weitere Verwendungsmöglichkeit für eine Funktion, die einen zusammengesetzten Typ zurückgibt, besteht darin, das Ergebnis an eine andere Funktion zu übergeben, die den passenden Row-Typ als Eingabe akzeptiert:

```sql
CREATE FUNCTION getname(emp) RETURNS text AS $$
    SELECT $1.name;
$$ LANGUAGE SQL;

SELECT getname(new_emp());
```

```text
 getname
---------
 None
(1 row)
```

### 36.5.4. SQL-Funktionen mit Ausgabeparametern

Eine alternative Möglichkeit, die Ergebnisse einer Funktion zu beschreiben, besteht darin, sie mit Ausgabeparametern zu definieren, wie in diesem Beispiel:

```sql
CREATE FUNCTION add_em (IN x int, IN y int, OUT sum int)
AS 'SELECT x + y'
LANGUAGE SQL;

SELECT add_em(3,7);
```

```text
 add_em
--------

(1 row)
```

Dies unterscheidet sich im Wesentlichen nicht von der in [Abschnitt 36.5.2](#3652-sqlfunktionen-auf-basistypen) gezeigten Version von `add_em`. Der eigentliche Wert von Ausgabeparametern liegt darin, dass sie eine bequeme Möglichkeit bieten, Funktionen zu definieren, die mehrere Spalten zurückgeben. Zum Beispiel:

```sql
CREATE FUNCTION sum_n_product (x int, y int, OUT sum int, OUT
 product int)
AS 'SELECT x + y, x * y'
LANGUAGE SQL;

SELECT * FROM sum_n_product(11,42);
```

```text
 sum | product
-----+---------
  53 |     462
(1 row)
```

Im Wesentlichen haben wir hier einen anonymen zusammengesetzten Typ für das Ergebnis der Funktion erstellt. Das obige Beispiel hat dasselbe Endergebnis wie:

```sql
CREATE TYPE sum_prod AS (sum int, product int);

CREATE FUNCTION sum_n_product (int, int) RETURNS sum_prod
AS 'SELECT $1 + $2, $1 * $2'
LANGUAGE SQL;
```

Es ist jedoch oft praktisch, sich nicht um die separate Definition des zusammengesetzten Typs kümmern zu müssen. Beachten Sie, dass die Namen, die den Ausgabeparametern zugewiesen werden, nicht nur Dekoration sind, sondern die Spaltennamen des anonymen zusammengesetzten Typs bestimmen. Wenn Sie für einen Ausgabeparameter keinen Namen angeben, wählt das System selbst einen Namen.

Beachten Sie, dass Ausgabeparameter nicht in die aufrufende Argumentliste aufgenommen werden, wenn eine solche Funktion aus SQL heraus aufgerufen wird. Das liegt daran, dass PostgreSQL nur die Eingabeparameter als aufrufende Signatur der Funktion betrachtet. Das bedeutet auch, dass nur die Eingabeparameter wichtig sind, wenn die Funktion etwa zum Löschen referenziert wird. Die obige Funktion könnten wir mit einer der beiden folgenden Formen löschen:

```sql
DROP FUNCTION sum_n_product (x int, y int, OUT sum int, OUT product
 int);
DROP FUNCTION sum_n_product (int, int);
```

Parameter können als `IN` (die Vorgabe), `OUT`, `INOUT` oder `VARIADIC` markiert werden. Ein `INOUT`-Parameter dient sowohl als Eingabeparameter, also als Teil der aufrufenden Argumentliste, als auch als Ausgabeparameter, also als Teil des Ergebnis-Record-Typs. `VARIADIC`-Parameter sind Eingabeparameter, werden aber wie unten beschrieben besonders behandelt.

### 36.5.5. SQL-Prozeduren mit Ausgabeparametern

Ausgabeparameter werden auch in Prozeduren unterstützt, funktionieren dort aber etwas anders als bei Funktionen. In `CALL`-Befehlen müssen Ausgabeparameter in der Argumentliste enthalten sein. Die Routine zur Belastung eines Bankkontos von oben könnte zum Beispiel so geschrieben werden:

```sql
CREATE PROCEDURE tp1 (accountno integer, debit numeric, OUT
 new_balance numeric) AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tp1.accountno
    RETURNING balance;
$$ LANGUAGE SQL;
```

Um diese Prozedur aufzurufen, muss ein Argument enthalten sein, das zum `OUT`-Parameter passt. Üblicherweise schreibt man `NULL`:

```sql
CALL tp1(17, 100.0, NULL);
```

Wenn Sie etwas anderes schreiben, muss es ein Ausdruck sein, der implizit in den deklarierten Typ des Parameters koerziert werden kann, genau wie bei Eingabeparametern. Beachten Sie jedoch, dass ein solcher Ausdruck nicht ausgewertet wird.

Wenn Sie eine Prozedur aus PL/pgSQL heraus aufrufen, müssen Sie statt `NULL` eine Variable schreiben, die die Ausgabe der Prozedur aufnimmt. Einzelheiten finden Sie in [Abschnitt 41.6.3](41_PL_pgSQL_SQL_Prozedursprache.md#4163-eine-prozedur-aufrufen).

### 36.5.6. SQL-Funktionen mit variabler Argumentanzahl

SQL-Funktionen können so deklariert werden, dass sie eine variable Anzahl von Argumenten akzeptieren, solange alle »optionalen« Argumente denselben Datentyp haben. Die optionalen Argumente werden der Funktion als Array übergeben. Die Funktion wird deklariert, indem der letzte Parameter als `VARIADIC` markiert wird; dieser Parameter muss als Array-Typ deklariert sein. Zum Beispiel:

```sql
CREATE FUNCTION mleast(VARIADIC arr numeric[]) RETURNS numeric AS $$
    SELECT min($1[i]) FROM generate_subscripts($1, 1) g(i);
$$ LANGUAGE SQL;

SELECT mleast(10, -1, 5, 4.4);
```

```text
 mleast
--------
     -1
(1 row)
```

Effektiv werden alle tatsächlichen Argumente ab der `VARIADIC`-Position in einem eindimensionalen Array gesammelt, als hätten Sie Folgendes geschrieben:

```sql
SELECT mleast(ARRAY[10, -1, 5, 4.4]);                           -- funktioniert nicht
```

Das können Sie allerdings nicht wirklich schreiben, oder zumindest passt es nicht zu dieser Funktionsdefinition. Ein als `VARIADIC` markierter Parameter passt zu einem oder mehreren Vorkommen seines Elementtyps, nicht zu seinem eigenen Typ.

Manchmal ist es nützlich, ein bereits konstruiertes Array an eine variadische Funktion übergeben zu können; das ist besonders praktisch, wenn eine variadische Funktion ihren Array-Parameter an eine andere weitergeben möchte. Außerdem ist dies die einzige sichere Möglichkeit, eine variadische Funktion aufzurufen, die in einem Schema gefunden wurde, in dem nicht vertrauenswürdige Benutzer Objekte erstellen dürfen; siehe [Abschnitt 10.3](10_Typumwandlung.md#103-funktionen). Dazu geben Sie im Aufruf `VARIADIC` an:

```sql
SELECT mleast(VARIADIC ARRAY[10, -1, 5, 4.4]);
```

Dies verhindert die Expansion des variadischen Parameters der Funktion in seinen Elementtyp, sodass der Array-Argumentwert normal passen kann. `VARIADIC` kann nur an das letzte tatsächliche Argument eines Funktionsaufrufs angehängt werden.

`VARIADIC` im Aufruf anzugeben ist außerdem die einzige Möglichkeit, ein leeres Array an eine variadische Funktion zu übergeben, zum Beispiel:

```sql
SELECT mleast(VARIADIC ARRAY[]::numeric[]);
```

Ein einfaches `SELECT mleast()` funktioniert nicht, weil ein variadischer Parameter zu mindestens einem tatsächlichen Argument passen muss. Sie könnten eine zweite Funktion namens `mleast` ohne Parameter definieren, wenn Sie solche Aufrufe erlauben möchten.

Die aus einem variadischen Parameter erzeugten Array-Elementparameter werden so behandelt, als hätten sie keine eigenen Namen. Das bedeutet, dass es nicht möglich ist, eine variadische Funktion mit benannten Argumenten aufzurufen (siehe [Abschnitt 4.3](04_SQL_Syntax.md#43-funktionsaufrufe)), außer wenn Sie `VARIADIC` angeben. Zum Beispiel funktioniert dies:

```sql
SELECT mleast(VARIADIC arr => ARRAY[10, -1, 5, 4.4]);
```

aber dies nicht:

```sql
SELECT mleast(arr => 10);
SELECT mleast(arr => ARRAY[10, -1, 5, 4.4]);
```

### 36.5.7. SQL-Funktionen mit Default-Werten für Argumente

Funktionen können mit Default-Werten für einige oder alle Eingabeargumente deklariert werden. Die Default-Werte werden eingefügt, wenn die Funktion mit zu wenigen tatsächlichen Argumenten aufgerufen wird. Da Argumente nur am Ende der tatsächlichen Argumentliste weggelassen werden können, müssen alle Parameter nach einem Parameter mit Default-Wert ebenfalls Default-Werte haben. Obwohl die Verwendung benannter Argumentnotation erlauben könnte, diese Einschränkung zu lockern, wird sie dennoch durchgesetzt, damit positionale Argumentnotation sinnvoll funktioniert. Unabhängig davon, ob Sie diese Möglichkeit verwenden, erfordert sie Vorsicht beim Aufruf von Funktionen in Datenbanken, in denen manche Benutzer anderen Benutzern misstrauen; siehe [Abschnitt 10.3](10_Typumwandlung.md#103-funktionen).

Zum Beispiel:

```sql
CREATE FUNCTION foo(a int, b int DEFAULT 2, c int DEFAULT 3)
RETURNS int
LANGUAGE SQL
AS $$
    SELECT $1 + $2 + $3;
$$;

SELECT foo(10, 20, 30);
```

```text
 foo
-----
(1 row)
```

```sql
SELECT foo(10, 20);
```

```text
 foo
-----
(1 row)
```

```sql
SELECT foo(10);
```

```text
 foo
-----
(1 row)
```

```sql
SELECT foo(); -- schlägt fehl, da es keinen Default für das erste Argument gibt
```

```text
ERROR: function foo() does not exist
```

Das Gleichheitszeichen (`=`) kann auch anstelle des Schlüsselworts `DEFAULT` verwendet werden.

### 36.5.8. SQL-Funktionen als Tabellenquellen

Alle SQL-Funktionen können in der `FROM`-Klausel einer Abfrage verwendet werden, aber besonders nützlich ist dies bei Funktionen, die zusammengesetzte Typen zurückgeben. Wenn die Funktion so definiert ist, dass sie einen Basistyp zurückgibt, erzeugt die Tabellenfunktion eine einspaltige Tabelle. Wenn die Funktion so definiert ist, dass sie einen zusammengesetzten Typ zurückgibt, erzeugt die Tabellenfunktion eine Spalte für jedes Attribut des zusammengesetzten Typs.

Hier ist ein Beispiel:

```sql
CREATE TABLE foo (fooid int, foosubid int, fooname text);
INSERT INTO foo VALUES (1, 1, 'Joe');
INSERT INTO foo VALUES (1, 2, 'Ed');
INSERT INTO foo VALUES (2, 1, 'Mary');

CREATE FUNCTION getfoo(int) RETURNS foo AS $$
    SELECT * FROM foo WHERE fooid = $1;
$$ LANGUAGE SQL;

SELECT *, upper(fooname) FROM getfoo(1) AS t1;
```

```text
 fooid | foosubid | fooname | upper
-------+----------+---------+-------
     1 |        1 | Joe     | JOE
(1 row)
```

Wie das Beispiel zeigt, können wir mit den Spalten des Funktionsergebnisses genauso arbeiten, als wären es Spalten einer regulären Tabelle.

Beachten Sie, dass wir nur eine Zeile aus der Funktion erhalten haben. Das liegt daran, dass wir `SETOF` nicht verwendet haben. Dies wird im nächsten Abschnitt beschrieben.

### 36.5.9. SQL-Funktionen, die Mengen zurückgeben

Wenn eine SQL-Funktion so deklariert ist, dass sie `SETOF sometype` zurückgibt, wird die letzte Abfrage der Funktion vollständig ausgeführt, und jede von ihr ausgegebene Zeile wird als Element der Ergebnismenge zurückgegeben.

Diese Funktionalität wird normalerweise verwendet, wenn die Funktion in der `FROM`-Klausel aufgerufen wird. In diesem Fall wird jede von der Funktion zurückgegebene Zeile zu einer Zeile der Tabelle, die die Abfrage sieht. Angenommen zum Beispiel, die Tabelle `foo` hat denselben Inhalt wie oben, und wir schreiben:

```sql
CREATE FUNCTION getfoo(int) RETURNS SETOF foo AS $$
    SELECT * FROM foo WHERE fooid = $1;
$$ LANGUAGE SQL;

SELECT * FROM getfoo(1) AS t1;
```

Dann erhalten wir:

```text
 fooid | foosubid | fooname
-------+----------+---------
     1 |        1 | Joe
     1 |        2 | Ed
(2 rows)
```

Es ist auch möglich, mehrere Zeilen mit Spalten zurückzugeben, die durch Ausgabeparameter definiert sind, zum Beispiel:

```sql
CREATE TABLE tab (y int, z int);
INSERT INTO tab VALUES (1, 2), (3, 4), (5, 6), (7, 8);

CREATE FUNCTION sum_n_product_with_tab (x int, OUT sum int, OUT
 product int)
RETURNS SETOF record
AS $$
    SELECT $1 + tab.y, $1 * tab.y FROM tab;
$$ LANGUAGE SQL;

SELECT * FROM sum_n_product_with_tab(10);
```

```text
 sum | product
-----+---------
  11 |      10
  13 |      30
  15 |      50
  17 |      70
(4 rows)
```

Der entscheidende Punkt ist hier, dass Sie `RETURNS SETOF record` schreiben müssen, um anzugeben, dass die Funktion mehrere Zeilen statt nur einer zurückgibt. Wenn es nur einen Ausgabeparameter gibt, schreiben Sie den Typ dieses Parameters statt `record`.

Es ist häufig nützlich, das Ergebnis einer Abfrage zu konstruieren, indem eine set-returning function mehrfach aufgerufen wird, wobei die Parameter jedes Aufrufs aus aufeinanderfolgenden Zeilen einer Tabelle oder Unterabfrage stammen. Die bevorzugte Methode dafür ist das Schlüsselwort `LATERAL`, das in [Abschnitt 7.2.1.5](07_Abfragen.md#7215-lateralunterabfragen) beschrieben wird. Hier ist ein Beispiel, das eine set-returning function verwendet, um Elemente einer Baumstruktur aufzuzählen:

```sql
SELECT * FROM nodes;
```

```text
   name    | parent
-----------+--------
 Top       |
 Child1    | Top
 Child2    | Top
 Child3    | Top
 SubChild1 | Child1
 SubChild2 | Child1
(6 rows)
```

```sql
CREATE FUNCTION listchildren(text) RETURNS SETOF text AS $$
    SELECT name FROM nodes WHERE parent = $1
$$ LANGUAGE SQL STABLE;

SELECT * FROM listchildren('Top');
```

```text
 listchildren
--------------
 Child1
 Child2
 Child3
(3 rows)
```

```sql
SELECT name, child FROM nodes, LATERAL listchildren(name) AS child;
```

```text
  name  |   child
--------+-----------
 Top    | Child1
 Top    | Child2
 Top    | Child3
 Child1 | SubChild1
 Child1 | SubChild2
(5 rows)
```

Dieses Beispiel tut nichts, was wir nicht auch mit einem einfachen Join hätten tun können, aber bei komplexeren Berechnungen kann die Möglichkeit, einen Teil der Arbeit in eine Funktion auszulagern, sehr praktisch sein.

Funktionen, die Mengen zurückgeben, können auch in der Auswahlliste einer Abfrage aufgerufen werden. Für jede Zeile, die die Abfrage selbst erzeugt, wird die set-returning function aufgerufen, und für jedes Element der Ergebnismenge der Funktion wird eine Ausgabezeile erzeugt. Das vorherige Beispiel könnte auch mit Abfragen wie diesen geschrieben werden:

```sql
SELECT listchildren('Top');
```

```text
 listchildren
--------------
 Child1
 Child2
 Child3
(3 rows)
```

```sql
SELECT name, listchildren(name) FROM nodes;
```

```text
  name  | listchildren
--------+--------------
 Top    | Child1
 Top    | Child2
 Top    | Child3
 Child1 | SubChild1
 Child1 | SubChild2
(5 rows)
```

Beachten Sie im letzten `SELECT`, dass keine Ausgabezeile für `Child2`, `Child3` usw. erscheint. Das geschieht, weil `listchildren` für diese Argumente eine leere Menge zurückgibt; daher werden keine Ergebniszeilen erzeugt. Dies ist dasselbe Verhalten wie bei einem Inner Join auf das Funktionsergebnis unter Verwendung der `LATERAL`-Syntax.

Das Verhalten von PostgreSQL für eine set-returning function in der Auswahlliste einer Abfrage ist fast exakt dasselbe, als wäre die set-returning function stattdessen als `LATERAL`-Element in der `FROM`-Klausel geschrieben worden. Zum Beispiel:

```sql
SELECT x, generate_series(1,5) AS g FROM tab;
```

ist fast gleichwertig mit:

```sql
SELECT x, g FROM tab, LATERAL generate_series(1,5) AS g;
```

Es wäre exakt dasselbe, außer dass der Planner in diesem speziellen Beispiel entscheiden könnte, `g` auf die Außenseite des Nested-Loop-Joins zu legen, weil `g` keine tatsächliche laterale Abhängigkeit von `tab` hat. Das würde zu einer anderen Reihenfolge der Ausgabezeilen führen. Set-returning functions in der Auswahlliste werden immer so ausgewertet, als stünden sie auf der Innenseite eines Nested-Loop-Joins mit dem Rest der `FROM`-Klausel, sodass die Funktion(en) vollständig ausgeführt werden, bevor die nächste Zeile aus der `FROM`-Klausel betrachtet wird.

Wenn es in der Auswahlliste der Abfrage mehr als eine set-returning function gibt, ähnelt das Verhalten dem, was Sie erhalten, wenn Sie die Funktionen in ein einzelnes `LATERAL ROWS FROM( ... )`-Element der `FROM`-Klausel setzen.

Für jede Zeile der zugrunde liegenden Abfrage gibt es eine Ausgabezeile mit dem ersten Ergebnis jeder Funktion, dann eine Ausgabezeile mit dem zweiten Ergebnis usw. Wenn einige der set-returning functions weniger Ausgaben erzeugen als andere, werden Nullwerte für die fehlenden Daten eingesetzt, sodass die Gesamtzahl der für eine zugrunde liegende Zeile ausgegebenen Zeilen der set-returning function entspricht, die die meisten Ausgaben erzeugt hat. Die set-returning functions laufen also im Gleichschritt, bis sie alle erschöpft sind; danach fährt die Ausführung mit der nächsten zugrunde liegenden Zeile fort.

Set-returning functions können in einer Auswahlliste verschachtelt werden, obwohl dies in Elementen der `FROM`-Klausel nicht erlaubt ist. In solchen Fällen wird jede Verschachtelungsebene separat behandelt, als wäre sie ein eigenes `LATERAL ROWS FROM( ... )`-Element. Zum Beispiel:

```sql
SELECT srf1(srf2(x), srf3(y)), srf4(srf5(z)) FROM tab;
```

Die set-returning functions `srf2`, `srf3` und `srf5` würden für jede Zeile von `tab` im Gleichschritt ausgeführt; danach würden `srf1` und `srf4` im Gleichschritt auf jede von den unteren Funktionen erzeugte Zeile angewendet.

Set-returning functions können nicht innerhalb von Konstrukten zur bedingten Auswertung wie `CASE` oder `COALESCE` verwendet werden. Betrachten Sie zum Beispiel:

```sql
SELECT x, CASE WHEN x > 0 THEN generate_series(1, 5) ELSE 0 END
 FROM tab;
```

Es könnte so aussehen, als sollte dies fünf Wiederholungen der Eingabezeilen mit `x > 0` und eine einzelne Wiederholung der übrigen Zeilen erzeugen. Tatsächlich würde `generate_series(1, 5)` aber in einem impliziten `LATERAL`-`FROM`-Element ausgeführt, bevor der `CASE`-Ausdruck überhaupt ausgewertet wird, und dadurch fünf Wiederholungen jeder Eingabezeile erzeugen. Um Verwirrung zu vermeiden, erzeugen solche Fälle stattdessen einen Parse-Fehler.

> Hinweis: Wenn der letzte Befehl einer Funktion `INSERT`, `UPDATE`, `DELETE` oder `MERGE` mit `RETURNING` ist, wird dieser Befehl immer vollständig ausgeführt, selbst wenn die Funktion nicht mit `SETOF` deklariert ist oder die aufrufende Abfrage nicht alle Ergebniszeilen abruft. Alle zusätzlichen Zeilen, die von der `RETURNING`-Klausel erzeugt werden, werden stillschweigend verworfen, aber die angeordneten Tabellenänderungen finden dennoch statt und sind vollständig abgeschlossen, bevor die Funktion zurückkehrt.

> Hinweis: Vor PostgreSQL 10 verhielt es sich nicht besonders sinnvoll, mehr als eine set-returning function in dieselbe Auswahlliste zu setzen, sofern sie nicht immer dieselbe Anzahl von Zeilen erzeugten. Andernfalls erhielt man eine Anzahl von Ausgabezeilen, die dem kleinsten gemeinsamen Vielfachen der von den set-returning functions erzeugten Zeilenzahlen entsprach. Außerdem funktionierten verschachtelte set-returning functions nicht wie oben beschrieben; stattdessen konnte eine set-returning function höchstens ein set-returning Argument haben, und jede Verschachtelung von set-returning functions wurde unabhängig ausgeführt. Auch bedingte Ausführung, also set-returning functions innerhalb von `CASE` usw., war früher erlaubt, was die Sache noch weiter verkomplizierte. Wenn Sie Abfragen schreiben, die in älteren PostgreSQL-Versionen funktionieren müssen, wird die Verwendung der `LATERAL`-Syntax empfohlen, weil sie über verschiedene Versionen hinweg konsistente Ergebnisse liefert. Wenn Sie eine Abfrage haben, die sich auf die bedingte Ausführung einer set-returning function verlässt, können Sie sie möglicherweise reparieren, indem Sie den bedingten Test in eine eigene set-returning function verschieben. Zum Beispiel:
>
> ```sql
> SELECT x, CASE WHEN y > 0 THEN generate_series(1, z) ELSE 5
>  END FROM tab;
> ```
>
> könnte werden zu:
>
> ```sql
> CREATE FUNCTION case_generate_series(cond bool, start int, fin
>  int, els int)
>   RETURNS SETOF int AS $$
> BEGIN
>   IF cond THEN
>     RETURN QUERY SELECT generate_series(start, fin);
>   ELSE
>     RETURN QUERY SELECT els;
>   END IF;
> END$$ LANGUAGE plpgsql;
>
> SELECT x, case_generate_series(y > 0, 1, z, 5) FROM tab;
> ```
>
> Diese Formulierung funktioniert in allen PostgreSQL-Versionen gleich.

### 36.5.10. SQL-Funktionen, die `TABLE` zurückgeben

Eine weitere Möglichkeit, eine Funktion als mengenrückgebend zu deklarieren, ist die Syntax `RETURNS TABLE(columns)`. Dies ist gleichwertig damit, einen oder mehrere `OUT`-Parameter zu verwenden und die Funktion zusätzlich als `SETOF record` zu markieren oder, falls passend, als `SETOF` des Typs eines einzelnen Ausgabeparameters. Diese Schreibweise ist in neueren Versionen des SQL-Standards spezifiziert und kann daher portabler sein als die Verwendung von `SETOF`.

Das vorherige Summen-und-Produkt-Beispiel könnte zum Beispiel auch so geschrieben werden:

```sql
CREATE FUNCTION sum_n_product_with_tab (x int)
RETURNS TABLE(sum int, product int) AS $$
    SELECT $1 + tab.y, $1 * tab.y FROM tab;
$$ LANGUAGE SQL;
```

Es ist nicht erlaubt, explizite `OUT`- oder `INOUT`-Parameter zusammen mit der `RETURNS TABLE`-Notation zu verwenden; Sie müssen alle Ausgabespalten in die `TABLE`-Liste setzen.

### 36.5.11. Polymorphe SQL-Funktionen

SQL-Funktionen können so deklariert werden, dass sie die in [Abschnitt 36.2.5](#3625-polymorphe-typen) beschriebenen polymorphen Typen annehmen und zurückgeben. Hier ist eine polymorphe Funktion `make_array`, die aus zwei Elementen beliebiger Datentypen ein Array aufbaut:

```sql
CREATE FUNCTION make_array(anyelement, anyelement) RETURNS anyarray
 AS $$
    SELECT ARRAY[$1, $2];
$$ LANGUAGE SQL;

SELECT make_array(1, 2) AS intarray, make_array('a'::text, 'b') AS
 textarray;
```

```text
 intarray | textarray
----------+-----------
 {1,2}    | {a,b}
(1 row)
```

Beachten Sie die Verwendung des Typecasts `'a'::text`, um anzugeben, dass das Argument vom Typ `text` ist. Dies ist erforderlich, wenn das Argument nur ein Zeichenkettenliteral ist, da es sonst als Typ `unknown` behandelt würde und ein Array von `unknown` kein gültiger Typ ist. Ohne den Typecast erhalten Sie Fehler wie diesen:

```text
ERROR: could not determine polymorphic type because input has type
 unknown
```

Mit der oben deklarierten Funktion `make_array` müssen Sie zwei Argumente bereitstellen, die exakt denselben Datentyp haben; das System versucht nicht, Typunterschiede aufzulösen. Daher funktioniert zum Beispiel Folgendes nicht:

```sql
SELECT make_array(1, 2.5) AS numericarray;
```

```text
ERROR: function make_array(integer, numeric) does not exist
```

Ein alternativer Ansatz ist die Verwendung der »gemeinsamen« Familie polymorpher Typen, die dem System erlaubt, einen passenden gemeinsamen Typ zu ermitteln:

```sql
CREATE FUNCTION make_array2(anycompatible, anycompatible)
RETURNS anycompatiblearray AS $$
    SELECT ARRAY[$1, $2];
$$ LANGUAGE SQL;

SELECT make_array2(1, 2.5) AS numericarray;
```

```text
 numericarray
--------------
 {1,2.5}
(1 row)
```

Da die Regeln zur Auflösung eines gemeinsamen Typs standardmäßig den Typ `text` wählen, wenn alle Eingaben vom Typ `unknown` sind, funktioniert auch dies:

```sql
SELECT make_array2('a', 'b') AS textarray;
```

```text
 textarray
-----------
 {a,b}
(1 row)
```

Polymorphe Argumente mit festem Rückgabetyp sind erlaubt, der umgekehrte Fall jedoch nicht. Zum Beispiel:

```sql
CREATE FUNCTION is_greater(anyelement, anyelement) RETURNS boolean
 AS $$
    SELECT $1 > $2;
$$ LANGUAGE SQL;

SELECT is_greater(1, 2);
```

```text
 is_greater
------------
 f
(1 row)
```

```sql
CREATE FUNCTION invalid_func() RETURNS anyelement AS $$
    SELECT 1;
$$ LANGUAGE SQL;
```

```text
ERROR: cannot determine result data type
DETAIL: A result of type anyelement requires at least one input of
 type anyelement, anyarray, anynonarray, anyenum, or anyrange.
```

Polymorphie kann mit Funktionen verwendet werden, die Ausgabeparameter haben. Zum Beispiel:

```sql
CREATE FUNCTION dup (f1 anyelement, OUT f2 anyelement, OUT f3
 anyarray)
AS 'select $1, array[$1,$1]' LANGUAGE SQL;

SELECT * FROM dup(22);
```

```text
 f2 |   f3
----+---------
 22 | {22,22}
(1 row)
```

Polymorphie kann auch mit variadischen Funktionen verwendet werden. Zum Beispiel:

```sql
CREATE FUNCTION anyleast (VARIADIC anyarray) RETURNS anyelement AS
 $$
    SELECT min($1[i]) FROM generate_subscripts($1, 1) g(i);
$$ LANGUAGE SQL;

SELECT anyleast(10, -1, 5, 4);
```

```text
 anyleast
----------
       -1
(1 row)
```

```sql
SELECT anyleast('abc'::text, 'def');
```

```text
 anyleast
----------
 abc
(1 row)
```

```sql
CREATE FUNCTION concat_values(text, VARIADIC anyarray) RETURNS text
 AS $$
    SELECT array_to_string($2, $1);
$$ LANGUAGE SQL;

SELECT concat_values('|', 1, 4, 2);
```

```text
 concat_values
---------------
 1|4|2
(1 row)
```

### 36.5.12. SQL-Funktionen mit Collations

Wenn eine SQL-Funktion einen oder mehrere Parameter von collatable Datentypen hat, wird für jeden Funktionsaufruf abhängig von den Collations, die den tatsächlichen Argumenten zugewiesen sind, eine Collation bestimmt, wie in [Abschnitt 23.2](23_Lokalisierung.md#232-collationunterstützung) beschrieben. Wenn eine Collation erfolgreich bestimmt wird, also keine Konflikte zwischen impliziten Collations der Argumente bestehen, werden alle collatable Parameter so behandelt, als hätten sie implizit diese Collation. Dies beeinflusst das Verhalten collation-sensitiver Operationen innerhalb der Funktion. Wenn zum Beispiel die oben beschriebene Funktion `anyleast` verwendet wird, hängt das Ergebnis von:

```sql
SELECT anyleast('abc'::text, 'ABC');
```

von der Default-Collation der Datenbank ab. In der Locale `C` ist das Ergebnis `ABC`, in vielen anderen Locales dagegen `abc`. Die zu verwendende Collation kann erzwungen werden, indem einem der Argumente eine `COLLATE`-Klausel hinzugefügt wird, zum Beispiel:

```sql
SELECT anyleast('abc'::text, 'ABC' COLLATE "C");
```

Wenn eine Funktion unabhängig von den Argumenten, mit denen sie aufgerufen wird, mit einer bestimmten Collation arbeiten soll, fügen Sie alternativ in der Funktionsdefinition nach Bedarf `COLLATE`-Klauseln ein. Diese Version von `anyleast` würde beim Vergleichen von Zeichenketten immer die Locale `en_US` verwenden:

```sql
CREATE FUNCTION anyleast (VARIADIC anyarray) RETURNS anyelement AS
 $$
    SELECT min($1[i] COLLATE "en_US") FROM generate_subscripts($1,
 1) g(i);
$$ LANGUAGE SQL;
```

Beachten Sie aber, dass dies einen Fehler auslöst, wenn es auf einen nicht collatable Datentyp angewendet wird.

Wenn unter den tatsächlichen Argumenten keine gemeinsame Collation bestimmt werden kann, behandelt eine SQL-Funktion ihre Parameter so, als hätten sie die Default-Collation ihrer Datentypen. Das ist gewöhnlich die Default-Collation der Datenbank, kann bei Parametern von Domain-Typen aber anders sein.

Das Verhalten von collatable Parametern kann als eingeschränkte Form von Polymorphie betrachtet werden, die nur auf textuelle Datentypen anwendbar ist.

## 36.6. Funktionsüberladung

Es kann mehr als eine Funktion mit demselben SQL-Namen definiert werden, solange sie unterschiedliche Argumente annehmen. Mit anderen Worten: Funktionsnamen können überladen werden. Unabhängig davon, ob Sie diese Möglichkeit verwenden, erfordert sie Sicherheitsvorkehrungen beim Aufruf von Funktionen in Datenbanken, in denen manche Benutzer anderen Benutzern misstrauen; siehe [Abschnitt 10.3](10_Typumwandlung.md#103-funktionen). Wenn eine Abfrage ausgeführt wird, bestimmt der Server anhand der Datentypen und der Anzahl der bereitgestellten Argumente, welche Funktion aufgerufen werden soll. Überladung kann auch verwendet werden, um Funktionen mit variabler Argumentanzahl bis zu einer endlichen Maximalzahl zu simulieren.

Beim Erstellen einer Familie überladener Funktionen sollte man darauf achten, keine Mehrdeutigkeiten zu erzeugen. Gegeben seien zum Beispiel die Funktionen:

```sql
CREATE FUNCTION test(int, real) RETURNS ...
CREATE FUNCTION test(smallint, double precision) RETURNS ...
```

Es ist nicht sofort klar, welche Funktion bei einer einfachen Eingabe wie `test(1, 1.5)` aufgerufen würde. Die derzeit implementierten Auflösungsregeln werden in [Kapitel 10](10_Typumwandlung.md) beschrieben, aber es ist unklug, ein System zu entwerfen, das sich auf subtile Weise auf dieses Verhalten verlässt.

Eine Funktion, die ein einzelnes Argument eines zusammengesetzten Typs annimmt, sollte im Allgemeinen nicht denselben Namen haben wie irgendein Attribut (Feld) dieses Typs. Denken Sie daran, dass `attribute(table)` als gleichwertig zu `table.attribute` betrachtet wird. Falls eine Mehrdeutigkeit zwischen einer Funktion auf einem zusammengesetzten Typ und einem Attribut des zusammengesetzten Typs besteht, wird immer das Attribut verwendet. Diese Wahl lässt sich übersteuern, indem der Funktionsname mit dem Schema qualifiziert wird, also `schema.func(table)`, aber es ist besser, das Problem durch Vermeiden konfliktierender Namen gar nicht erst entstehen zu lassen.

Ein weiterer möglicher Konflikt besteht zwischen variadischen und nicht variadischen Funktionen. Es ist zum Beispiel möglich, sowohl `foo(numeric)` als auch `foo(VARIADIC numeric[])` zu erstellen. In diesem Fall ist unklar, welche Funktion zu einem Aufruf mit einem einzelnen numerischen Argument wie `foo(10.1)` passen soll. Die Regel lautet: Es wird die Funktion verwendet, die früher im Suchpfad erscheint; wenn beide Funktionen im selben Schema liegen, wird die nicht variadische bevorzugt.

Beim Überladen von C-Sprachfunktionen gibt es eine zusätzliche Einschränkung: Der C-Name jeder Funktion in der Familie überladener Funktionen muss sich von den C-Namen aller anderen Funktionen unterscheiden, unabhängig davon, ob diese intern oder dynamisch geladen sind. Wenn diese Regel verletzt wird, ist das Verhalten nicht portabel. Sie erhalten möglicherweise einen Laufzeit-Linkerfehler, oder eine der Funktionen wird aufgerufen, gewöhnlich die interne. Die alternative Form der `AS`-Klausel des SQL-Befehls `CREATE FUNCTION` entkoppelt den SQL-Funktionsnamen vom Funktionsnamen im C-Quellcode. Zum Beispiel:

```sql
CREATE FUNCTION test(int) RETURNS int
    AS 'filename', 'test_1arg'
    LANGUAGE C;
CREATE FUNCTION test(int, int) RETURNS int
    AS 'filename', 'test_2arg'
    LANGUAGE C;
```

Die Namen der C-Funktionen spiegeln hier eine von vielen möglichen Konventionen wider.

## 36.7. Volatilitätskategorien von Funktionen

Jede Funktion hat eine Volatilitätsklassifikation mit den Möglichkeiten `VOLATILE`, `STABLE` oder `IMMUTABLE`. `VOLATILE` ist die Vorgabe, wenn der Befehl `CREATE FUNCTION` keine Kategorie angibt. Die Volatilitätskategorie ist ein Versprechen an den Optimizer über das Verhalten der Funktion:

- Eine `VOLATILE`-Funktion kann alles tun, einschließlich Änderungen an der Datenbank. Sie kann bei aufeinanderfolgenden Aufrufen mit denselben Argumenten unterschiedliche Ergebnisse zurückgeben. Der Optimizer trifft keine Annahmen über das Verhalten solcher Funktionen. Eine Abfrage, die eine volatile Funktion verwendet, wertet die Funktion in jeder Zeile neu aus, in der ihr Wert benötigt wird.
- Eine `STABLE`-Funktion kann die Datenbank nicht verändern und garantiert, bei denselben Argumenten für alle Zeilen innerhalb einer einzelnen Anweisung dieselben Ergebnisse zurückzugeben. Diese Kategorie erlaubt dem Optimizer, mehrere Aufrufe der Funktion zu einem einzigen Aufruf zu optimieren. Insbesondere ist es sicher, einen Ausdruck, der eine solche Funktion enthält, in einer Index-Scan-Bedingung zu verwenden. Da ein Index-Scan den Vergleichswert nur einmal auswertet und nicht einmal pro Zeile, ist es nicht zulässig, eine `VOLATILE`-Funktion in einer Index-Scan-Bedingung zu verwenden.
- Eine `IMMUTABLE`-Funktion kann die Datenbank nicht verändern und garantiert, bei denselben Argumenten für immer dieselben Ergebnisse zurückzugeben. Diese Kategorie erlaubt dem Optimizer, die Funktion vorab auszuwerten, wenn eine Abfrage sie mit konstanten Argumenten aufruft. Eine Abfrage wie `SELECT ... WHERE x = 2 + 2` kann beispielsweise sofort zu `SELECT ... WHERE x = 4` vereinfacht werden, weil die dem Ganzzahl-Additionsoperator zugrunde liegende Funktion als `IMMUTABLE` markiert ist.

Für die besten Optimierungsergebnisse sollten Sie Ihre Funktionen mit der strengsten Volatilitätskategorie kennzeichnen, die für sie gültig ist.

Jede Funktion mit Seiteneffekten muss als `VOLATILE` markiert werden, damit Aufrufe nicht wegoptimiert werden können. Selbst eine Funktion ohne Seiteneffekte muss als `VOLATILE` markiert werden, wenn sich ihr Wert innerhalb einer einzelnen Abfrage ändern kann; Beispiele sind `random()`, `currval()` und `timeofday()`.

Ein weiteres wichtiges Beispiel ist, dass die Funktionen der `current_timestamp`-Familie als `STABLE` gelten, da sich ihre Werte innerhalb einer Transaktion nicht ändern.

Bei einfachen interaktiven Abfragen, die geplant und sofort ausgeführt werden, gibt es relativ wenig Unterschied zwischen den Kategorien `STABLE` und `IMMUTABLE`: Es macht nicht viel aus, ob eine Funktion einmal während der Planung oder einmal beim Start der Abfrageausführung ausgeführt wird. Einen großen Unterschied gibt es jedoch, wenn der Plan gespeichert und später wiederverwendet wird. Eine Funktion als `IMMUTABLE` zu kennzeichnen, obwohl sie es nicht wirklich ist, kann dazu führen, dass sie während der Planung vorzeitig zu einer Konstante gefaltet wird, sodass bei späteren Verwendungen des Plans ein veralteter Wert wiederverwendet wird. Das ist eine Gefahr bei vorbereiteten Anweisungen oder bei Funktionssprachen, die Pläne zwischenspeichern, etwa PL/pgSQL.

Für Funktionen, die in SQL oder in einer der standardmäßigen prozeduralen Sprachen geschrieben sind, gibt es eine zweite wichtige Eigenschaft, die durch die Volatilitätskategorie bestimmt wird: die Sichtbarkeit von Datenänderungen, die vom SQL-Befehl vorgenommen wurden, der die Funktion aufruft. Eine `VOLATILE`-Funktion sieht solche Änderungen, eine `STABLE`- oder `IMMUTABLE`-Funktion nicht. Dieses Verhalten wird durch das Snapshot-Verhalten von MVCC implementiert (siehe [Kapitel 13](13_Nebenläufigkeitskontrolle.md)): `STABLE`- und `IMMUTABLE`-Funktionen verwenden einen Snapshot vom Beginn der aufrufenden Abfrage, während `VOLATILE`-Funktionen zu Beginn jeder Abfrage, die sie ausführen, einen frischen Snapshot erhalten.

> Hinweis: In C geschriebene Funktionen können Snapshots verwalten, wie sie wollen, aber es ist normalerweise eine gute Idee, C-Funktionen ebenfalls auf diese Weise arbeiten zu lassen.

Wegen dieses Snapshot-Verhaltens kann eine Funktion, die nur `SELECT`-Befehle enthält, sicher als `STABLE` markiert werden, selbst wenn sie aus Tabellen liest, die möglicherweise durch gleichzeitige Abfragen geändert werden. PostgreSQL führt alle Befehle einer `STABLE`-Funktion mit dem Snapshot aus, der für die aufrufende Abfrage eingerichtet wurde, und sieht daher während dieser Abfrage eine feste Sicht der Datenbank.

Dasselbe Snapshot-Verhalten wird für `SELECT`-Befehle innerhalb von `IMMUTABLE`-Funktionen verwendet. Im Allgemeinen ist es unklug, innerhalb einer `IMMUTABLE`-Funktion überhaupt aus Datenbanktabellen zu lesen, da die Unveränderlichkeit gebrochen wird, wenn sich der Tabelleninhalt jemals ändert. PostgreSQL erzwingt jedoch nicht, dass Sie dies unterlassen.

Ein häufiger Fehler ist es, eine Funktion als `IMMUTABLE` zu kennzeichnen, wenn ihre Ergebnisse von einem Konfigurationsparameter abhängen. Eine Funktion, die Zeitstempel verarbeitet, kann zum Beispiel durchaus Ergebnisse haben, die von der Einstellung `TimeZone` abhängen. Aus Sicherheitsgründen sollten solche Funktionen stattdessen als `STABLE` markiert werden.

> Hinweis: PostgreSQL verlangt, dass `STABLE`- und `IMMUTABLE`-Funktionen außer `SELECT` keine SQL-Befehle enthalten, um Datenänderungen zu verhindern. Dies ist kein vollkommen wasserdichter Test, da solche Funktionen weiterhin `VOLATILE`-Funktionen aufrufen könnten, die die Datenbank ändern. Wenn Sie das tun, werden Sie feststellen, dass die `STABLE`- oder `IMMUTABLE`-Funktion die von der aufgerufenen Funktion vorgenommenen Datenbankänderungen nicht bemerkt, weil sie vor ihrem Snapshot verborgen sind.

## 36.8. Funktionen in prozeduralen Sprachen

PostgreSQL erlaubt, benutzerdefinierte Funktionen außer in SQL und C auch in anderen Sprachen zu schreiben. Diese anderen Sprachen werden allgemein prozedurale Sprachen (PLs) genannt. Prozedurale Sprachen sind nicht in den PostgreSQL-Server eingebaut; sie werden durch ladbare Module bereitgestellt. Weitere Informationen finden Sie in [Kapitel 40](40_Prozedurale_Sprachen.md) und den folgenden Kapiteln.

## 36.9. Interne Funktionen

Interne Funktionen sind in C geschriebene Funktionen, die statisch in den PostgreSQL-Server gelinkt wurden. Der »Rumpf« der Funktionsdefinition gibt den C-Sprachnamen der Funktion an, der nicht derselbe sein muss wie der Name, der für die SQL-Verwendung deklariert wird. Aus Gründen der Rückwärtskompatibilität wird ein leerer Rumpf so akzeptiert, als bedeute er, dass der C-Sprachfunktionsname dem SQL-Namen entspricht.

Normalerweise werden alle im Server vorhandenen internen Funktionen während der Initialisierung des Datenbankclusters deklariert (siehe [Abschnitt 18.2](18_Servereinrichtung_und_Betrieb.md#182-einen-datenbankcluster-erstellen)), aber ein Benutzer könnte `CREATE FUNCTION` verwenden, um zusätzliche Aliasnamen für eine interne Funktion zu erstellen. Interne Funktionen werden in `CREATE FUNCTION` mit dem Sprachnamen `internal` deklariert. Um zum Beispiel einen Alias für die Funktion `sqrt` zu erstellen:

```sql
CREATE FUNCTION square_root(double precision) RETURNS double
 precision
    AS 'dsqrt'
    LANGUAGE internal
    STRICT;
```

Die meisten internen Funktionen erwarten, als »strict« deklariert zu werden.

> Hinweis: Nicht alle »vordefinierten« Funktionen sind in diesem Sinne »intern«. Einige vordefinierte Funktionen sind in SQL geschrieben.

## 36.10. C-Sprachfunktionen

Benutzerdefinierte Funktionen können in C geschrieben werden oder in einer Sprache, die mit C kompatibel gemacht werden kann, etwa C++. Solche Funktionen werden zu dynamisch ladbaren Objekten kompiliert, auch Shared Libraries genannt, und vom Server bei Bedarf geladen. Das dynamische Laden ist das Merkmal, das »C-Sprachfunktionen« von »internen« Funktionen unterscheidet; die eigentlichen Programmierkonventionen sind für beide im Wesentlichen dieselben. Daher ist die standardmäßige Bibliothek interner Funktionen eine ergiebige Quelle von Programmierbeispielen für benutzerdefinierte C-Funktionen.

Derzeit wird für C-Funktionen nur eine Aufrufkonvention verwendet (»Version 1«). Die Unterstützung dieser Aufrufkonvention wird angezeigt, indem für die Funktion ein Makroaufruf `PG_FUNCTION_INFO_V1()` geschrieben wird, wie unten gezeigt.

### 36.10.1. Dynamisches Laden

Wenn eine benutzerdefinierte Funktion aus einer bestimmten ladbaren Objektdatei zum ersten Mal in einer Sitzung aufgerufen wird, lädt der dynamische Loader diese Objektdatei in den Speicher, damit die Funktion aufgerufen werden kann. `CREATE FUNCTION` für eine benutzerdefinierte C-Funktion muss daher zwei Informationen für die Funktion angeben: den Namen der ladbaren Objektdatei und den C-Namen (Link-Symbol) der konkreten Funktion, die innerhalb dieser Objektdatei aufgerufen werden soll. Wenn der C-Name nicht explizit angegeben wird, wird angenommen, dass er mit dem SQL-Funktionsnamen übereinstimmt.

Der folgende Algorithmus wird verwendet, um die Shared-Object-Datei anhand des im Befehl `CREATE FUNCTION` angegebenen Namens zu finden:

1. Wenn der Name ein absoluter Pfad ist, wird die angegebene Datei geladen.
2. Wenn der Name mit der Zeichenkette `$libdir` beginnt, wird dieser Teil durch den Namen des PostgreSQL-Paketbibliotheksverzeichnisses ersetzt, der zur Build-Zeit bestimmt wird.
3. Wenn der Name keinen Verzeichnisanteil enthält, wird die Datei im Pfad gesucht, der durch die Konfigurationsvariable `dynamic_library_path` angegeben ist.
4. Andernfalls, also wenn die Datei im Pfad nicht gefunden wurde oder einen nicht absoluten Verzeichnisanteil enthält, versucht der dynamische Loader, den Namen wie angegeben zu verwenden, was höchstwahrscheinlich fehlschlägt. Es ist unzuverlässig, sich auf das aktuelle Arbeitsverzeichnis zu verlassen.

Wenn diese Sequenz nicht funktioniert, wird die plattformspezifische Dateiendung für Shared Libraries, häufig `.so`, an den angegebenen Namen angehängt und die Sequenz erneut versucht. Wenn auch das fehlschlägt, schlägt das Laden fehl.

Es wird empfohlen, Shared Libraries entweder relativ zu `$libdir` oder über den dynamischen Bibliothekspfad zu lokalisieren. Das vereinfacht Versions-Upgrades, wenn die neue Installation an einem anderen Ort liegt. Das tatsächliche Verzeichnis, für das `$libdir` steht, kann mit dem Befehl `pg_config --pkglibdir` ermittelt werden.

Die Benutzer-ID, unter der der PostgreSQL-Server läuft, muss den Pfad zu der Datei durchlaufen können, die Sie laden möchten. Es ist ein häufiger Fehler, die Datei oder ein übergeordnetes Verzeichnis für den Benutzer `postgres` nicht lesbar und/oder nicht ausführbar zu machen.

In jedem Fall wird der im Befehl `CREATE FUNCTION` angegebene Dateiname wörtlich in den Systemkatalogen gespeichert. Wenn die Datei später erneut geladen werden muss, wird daher dasselbe Verfahren angewendet.

> Hinweis: PostgreSQL kompiliert eine C-Funktion nicht automatisch. Die Objektdatei muss kompiliert sein, bevor sie in einem `CREATE FUNCTION`-Befehl referenziert wird. Weitere Informationen finden Sie in [Abschnitt 36.10.5](#36105-dynamisch-geladene-funktionen-kompilieren-und-linken).

Um sicherzustellen, dass eine dynamisch geladene Objektdatei nicht in einen inkompatiblen Server geladen wird, prüft PostgreSQL, ob die Datei einen »Magic Block« mit passenden Inhalten enthält. Dadurch kann der Server offensichtliche Inkompatibilitäten erkennen, etwa Code, der für eine andere Hauptversion von PostgreSQL kompiliert wurde. Um einen Magic Block einzufügen, schreiben Sie Folgendes in genau eine der Modulquelldateien, nachdem Sie den Header `fmgr.h` eingebunden haben:

```c
PG_MODULE_MAGIC;
```

oder

```c
PG_MODULE_MAGIC_EXT(parameters);
```

Die Variante `PG_MODULE_MAGIC_EXT` erlaubt, zusätzliche Informationen zum Modul anzugeben; derzeit können ein Name und/oder eine Versionszeichenkette hinzugefügt werden. In Zukunft könnten weitere Felder erlaubt werden. Schreiben Sie zum Beispiel:

```c
PG_MODULE_MAGIC_EXT(
    .name = "my_module_name",
    .version = "1.2.3"
);
```

Anschließend können Name und Version über die Funktion `pg_get_loaded_modules()` abgefragt werden. Die Bedeutung der Versionszeichenkette wird von PostgreSQL nicht eingeschränkt, aber die Verwendung semantischer Versionierungsregeln wird empfohlen.

Nachdem eine dynamisch geladene Objektdatei zum ersten Mal verwendet wurde, bleibt sie im Speicher. Spätere Aufrufe derselben Sitzung an die Funktion(en) in dieser Datei verursachen nur den geringen Overhead einer Symboltabellensuche. Wenn Sie ein erneutes Laden einer Objektdatei erzwingen müssen, zum Beispiel nach dem Neukompilieren, beginnen Sie eine neue Sitzung.

Optional kann eine dynamisch geladene Datei eine Initialisierungsfunktion enthalten. Wenn die Datei eine Funktion namens `_PG_init` enthält, wird diese Funktion unmittelbar nach dem Laden der Datei aufgerufen. Die Funktion erhält keine Parameter und sollte `void` zurückgeben. Derzeit gibt es keine Möglichkeit, eine dynamisch geladene Datei wieder zu entladen.

### 36.10.2. Basistypen in C-Sprachfunktionen

Um C-Sprachfunktionen schreiben zu können, müssen Sie wissen, wie PostgreSQL Basistypen intern darstellt und wie sie an Funktionen übergeben und von ihnen zurückgegeben werden können. Intern betrachtet PostgreSQL einen Basistyp als »Speicherklumpen«. Die benutzerdefinierten Funktionen, die Sie für einen Typ definieren, bestimmen wiederum, wie PostgreSQL mit ihm arbeiten kann. Das heißt: PostgreSQL speichert und liest die Daten nur von der Platte und verwendet Ihre benutzerdefinierten Funktionen, um die Daten einzugeben, zu verarbeiten und auszugeben.

Basistypen können eines von drei internen Formaten haben:

- Übergabe per Wert, feste Länge
- Übergabe per Referenz, feste Länge
- Übergabe per Referenz, variable Länge

By-value-Typen können nur 1, 2 oder 4 Byte lang sein, außerdem 8 Byte, wenn `sizeof(Datum)` auf Ihrer Maschine 8 ist. Sie sollten Ihre Typen sorgfältig so definieren, dass sie auf allen Architekturen dieselbe Größe in Byte haben. Der Typ `long` ist zum Beispiel gefährlich, weil er auf manchen Maschinen 4 Byte und auf anderen 8 Byte groß ist, während der Typ `int` auf den meisten Unix-Maschinen 4 Byte groß ist. Eine vernünftige Implementierung des Typs `int4` auf Unix-Maschinen könnte sein:

```c
/* 4-byte integer, passed by value */
typedef int int4;
```

Der tatsächliche PostgreSQL-C-Code nennt diesen Typ `int32`, weil es in C Konvention ist, dass `intXX` XX Bits bedeutet. Beachten Sie daher auch, dass der C-Typ `int8` 1 Byte groß ist. Der SQL-Typ `int8` heißt in C `int64`. Siehe auch Tabelle 36.2.

Andererseits können Typen fester Länge beliebiger Größe per Referenz übergeben werden. Hier ist zum Beispiel eine Beispielimplementierung eines PostgreSQL-Typs:

```c
/* 16-byte structure, passed by reference */
typedef struct
{
    double x, y;
} Point;
```

Beim Übergeben solcher Typen an PostgreSQL-Funktionen und aus ihnen heraus können nur Zeiger verwendet werden. Um einen Wert eines solchen Typs zurückzugeben, reservieren Sie mit `palloc` die passende Speichermenge, füllen den reservierten Speicher und geben einen Zeiger darauf zurück. Wenn Sie außerdem nur denselben Wert zurückgeben möchten wie eines Ihrer Eingabeargumente desselben Datentyps, können Sie das zusätzliche `palloc` überspringen und einfach den Zeiger auf den Eingabewert zurückgeben.

Schließlich müssen auch alle Typen variabler Länge per Referenz übergeben werden. Alle Typen variabler Länge müssen mit einem undurchsichtigen Längenfeld von exakt 4 Byte beginnen, das von `SET_VARSIZE` gesetzt wird; setzen Sie dieses Feld niemals direkt! Alle Daten, die in diesem Typ gespeichert werden sollen, müssen im Speicher unmittelbar nach diesem Längenfeld liegen. Das Längenfeld enthält die Gesamtlänge der Struktur, schließt also die Größe des Längenfelds selbst ein.

Ein weiterer wichtiger Punkt ist, keine uninitialisierten Bits innerhalb von Datentypwerten zurückzulassen. Achten Sie zum Beispiel darauf, eventuell vorhandene Alignment-Padding-Bytes in Strukturen auf null zu setzen. Andernfalls könnten logisch gleichwertige Konstanten Ihres Datentyps vom Planner als ungleich angesehen werden, was zu ineffizienten, wenn auch nicht falschen, Plänen führt.

> Warnung: Ändern Sie niemals den Inhalt eines per Referenz übergebenen Eingabewerts. Wenn Sie das tun, werden Sie wahrscheinlich Daten auf der Platte beschädigen, da der Zeiger, den Sie erhalten, direkt in einen Plattenpuffer zeigen könnte. Die einzige Ausnahme von dieser Regel wird in [Abschnitt 36.12](#3612-benutzerdefinierte-aggregate) erklärt.

Als Beispiel können wir den Typ `text` wie folgt definieren:

```c
typedef struct {
    int32 length;
    char data[FLEXIBLE_ARRAY_MEMBER];
} text;
```

Die Notation `[FLEXIBLE_ARRAY_MEMBER]` bedeutet, dass die tatsächliche Länge des Datenteils durch diese Deklaration nicht angegeben wird.

Beim Bearbeiten von Typen variabler Länge müssen wir darauf achten, die richtige Speichermenge zu reservieren und das Längenfeld korrekt zu setzen. Wenn wir zum Beispiel 40 Byte in einer `text`-Struktur speichern wollten, könnten wir ein Codefragment wie dieses verwenden:

```c
#include "postgres.h"
...
char buffer[40]; /* our source data */
...
text *destination = (text *) palloc(VARHDRSZ + 40);
SET_VARSIZE(destination, VARHDRSZ + 40);
memcpy(destination->data, buffer, 40);
...
```

`VARHDRSZ` ist dasselbe wie `sizeof(int32)`, aber es gilt als guter Stil, das Makro `VARHDRSZ` zu verwenden, um auf die Größe des Overheads für einen Typ variabler Länge zu verweisen. Außerdem muss das Längenfeld mit dem Makro `SET_VARSIZE` gesetzt werden, nicht durch einfache Zuweisung.

Tabelle 36.2 zeigt die C-Typen, die vielen eingebauten SQL-Datentypen von PostgreSQL entsprechen. Die Spalte »Definiert in« gibt die Header-Datei an, die eingebunden werden muss, um die Typdefinition zu erhalten. Die eigentliche Definition könnte in einer anderen Datei liegen, die von der aufgelisteten Datei eingebunden wird. Es wird empfohlen, dass Benutzer bei der definierten Schnittstelle bleiben. Beachten Sie, dass Sie in jeder Quelldatei von Servercode immer zuerst `postgres.h` einbinden sollten, weil diese Datei eine Reihe von Dingen deklariert, die Sie ohnehin benötigen, und weil das Einbinden anderer Header zuerst Portabilitätsprobleme verursachen kann.

**Tabelle 36.2. Entsprechende C-Typen für eingebaute SQL-Typen**

| SQL-Typ | C-Typ | Definiert in |
|---|---|---|
| `boolean` | `bool` | `postgres.h` (eventuell Compiler-eingebaut) |
| `box` | `BOX*` | `utils/geo_decls.h` |
| `bytea` | `bytea*` | `postgres.h` |
| `"char"` | `char` | Compiler-eingebaut |
| `character` | `BpChar*` | `postgres.h` |
| `cid` | `CommandId` | `postgres.h` |
| `date` | `DateADT` | `utils/date.h` |
| `float4` (`real`) | `float4` | `postgres.h` |
| `float8` (`double precision`) | `float8` | `postgres.h` |
| `int2` (`smallint`) | `int16` | `postgres.h` |
| `int4` (`integer`) | `int32` | `postgres.h` |
| `int8` (`bigint`) | `int64` | `postgres.h` |
| `interval` | `Interval*` | `datatype/timestamp.h` |
| `lseg` | `LSEG*` | `utils/geo_decls.h` |
| `name` | `Name` | `postgres.h` |
| `numeric` | `Numeric` | `utils/numeric.h` |
| `oid` | `Oid` | `postgres.h` |
| `oidvector` | `oidvector*` | `postgres.h` |
| `path` | `PATH*` | `utils/geo_decls.h` |
| `point` | `POINT*` | `utils/geo_decls.h` |
| `regproc` | `RegProcedure` | `postgres.h` |
| `text` | `text*` | `postgres.h` |
| `tid` | `ItemPointer` | `storage/itemptr.h` |
| `time` | `TimeADT` | `utils/date.h` |
| `time with time zone` | `TimeTzADT` | `utils/date.h` |
| `timestamp` | `Timestamp` | `datatype/timestamp.h` |
| `timestamp with time zone` | `TimestampTz` | `datatype/timestamp.h` |
| `varchar` | `VarChar*` | `postgres.h` |
| `xid` | `TransactionId` | `postgres.h` |

Nachdem wir alle möglichen Strukturen für Basistypen durchgegangen sind, können wir einige Beispiele echter Funktionen zeigen.

### 36.10.3. Aufrufkonventionen der Version 1

Die Aufrufkonvention der Version 1 verwendet Makros, um den größten Teil der Komplexität beim Übergeben von Argumenten und Ergebnissen zu verbergen. Die C-Deklaration einer Version-1-Funktion lautet immer:

```c
Datum funcname(PG_FUNCTION_ARGS)
```

Zusätzlich muss der Makroaufruf:

```c
PG_FUNCTION_INFO_V1(funcname);
```

in derselben Quelldatei erscheinen. Konventionell steht er direkt vor der Funktion selbst. Für Funktionen der Sprache `internal` ist dieser Makroaufruf nicht nötig, da PostgreSQL annimmt, dass alle internen Funktionen die Version-1-Konvention verwenden. Für dynamisch geladene Funktionen ist er jedoch erforderlich.

In einer Version-1-Funktion wird jedes tatsächliche Argument mit einem `PG_GETARG_xxx()`-Makro geholt, das dem Datentyp des Arguments entspricht. In nicht strikten Funktionen muss vorher mit `PG_ARGISNULL()` geprüft werden, ob das Argument null ist; siehe unten. Das Ergebnis wird mit einem `PG_RETURN_xxx()`-Makro für den Rückgabetyp zurückgegeben. `PG_GETARG_xxx()` erhält als Argument die Nummer des zu holenden Funktionsarguments, wobei die Zählung bei 0 beginnt. `PG_RETURN_xxx()` erhält als Argument den tatsächlichen zurückzugebenden Wert.

Um eine andere Version-1-Funktion aufzurufen, können Sie `DirectFunctionCalln(func, arg1, ..., argn)` verwenden. Das ist besonders nützlich, wenn Sie Funktionen aus der standardmäßigen internen Bibliothek über eine Schnittstelle aufrufen möchten, die ihrer SQL-Signatur ähnelt.

Diese Komfortfunktionen und ähnliche Hilfen finden sich in `fmgr.h`. Die Familie `DirectFunctionCalln` erwartet einen C-Funktionsnamen als erstes Argument. Außerdem gibt es `OidFunctionCalln`, das die OID der Zielfunktion annimmt, sowie einige weitere Varianten. Alle erwarten, dass die Funktionsargumente als `Datum` geliefert werden, und geben ebenfalls `Datum` zurück. Beachten Sie, dass bei Verwendung dieser Komfortfunktionen weder Argumente noch Ergebnis `NULL` sein dürfen.

Um zum Beispiel die Funktion `starts_with(text, text)` aus C aufzurufen, können Sie im Katalog nachsehen und feststellen, dass ihre C-Implementierung die Funktion `Datum text_starts_with(PG_FUNCTION_ARGS)` ist. Typischerweise würden Sie `DirectFunctionCall2(text_starts_with, ...)` verwenden, um eine solche Funktion aufzurufen. `starts_with(text, text)` benötigt jedoch Collation-Informationen und würde bei diesem Aufruf mit »could not determine which collation to use for string comparison« fehlschlagen. Stattdessen müssen Sie `DirectFunctionCall2Coll(text_starts_with, ...)` verwenden und die gewünschte Collation bereitstellen, die typischerweise einfach aus `PG_GET_COLLATION()` weitergereicht wird, wie im folgenden Beispiel gezeigt.

`fmgr.h` stellt außerdem Makros bereit, die Umwandlungen zwischen C-Typen und `Datum` erleichtern. Um zum Beispiel `Datum` in `text*` umzuwandeln, können Sie `DatumGetTextPP(X)` verwenden. Einige Typen haben Makros mit Namen wie `TypeGetDatum(X)` für die umgekehrte Umwandlung, `text*` jedoch nicht; dafür genügt das generische Makro `PointerGetDatum(X)`. Wenn Ihre Erweiterung zusätzliche Typen definiert, ist es normalerweise praktisch, ähnliche Makros auch für Ihre Typen zu definieren.

Hier sind einige Beispiele mit der Version-1-Aufrufkonvention:

```c
#include "postgres.h"
#include <string.h>
#include "fmgr.h"
#include "utils/geo_decls.h"
#include "varatt.h"

PG_MODULE_MAGIC;

/* by value */

PG_FUNCTION_INFO_V1(add_one);

Datum
add_one(PG_FUNCTION_ARGS)
{
    int32   arg = PG_GETARG_INT32(0);

    PG_RETURN_INT32(arg + 1);
}

/* by reference, fixed length */

PG_FUNCTION_INFO_V1(add_one_float8);

Datum
add_one_float8(PG_FUNCTION_ARGS)
{
    /* The macros for FLOAT8 hide its pass-by-reference nature. */
    float8   arg = PG_GETARG_FLOAT8(0);

    PG_RETURN_FLOAT8(arg + 1.0);
}

PG_FUNCTION_INFO_V1(makepoint);

Datum
makepoint(PG_FUNCTION_ARGS)
{
    /* Here, the pass-by-reference nature of Point is not hidden. */
    Point     *pointx = PG_GETARG_POINT_P(0);
    Point     *pointy = PG_GETARG_POINT_P(1);
    Point     *new_point = (Point *) palloc(sizeof(Point));

    new_point->x = pointx->x;
    new_point->y = pointy->y;

    PG_RETURN_POINT_P(new_point);
}

/* by reference, variable length */

PG_FUNCTION_INFO_V1(copytext);

Datum
copytext(PG_FUNCTION_ARGS)
{
    text     *t = PG_GETARG_TEXT_PP(0);

    /*
     * VARSIZE_ANY_EXHDR is the size of the struct in bytes, minus
     * the VARHDRSZ or VARHDRSZ_SHORT of its header. Construct the
     * copy with a full-length header.
     */
    text     *new_t = (text *) palloc(VARSIZE_ANY_EXHDR(t) + VARHDRSZ);
    SET_VARSIZE(new_t, VARSIZE_ANY_EXHDR(t) + VARHDRSZ);

    /*
     * VARDATA is a pointer to the data region of the new struct.
     * The source could be a short datum, so retrieve its data through
     * VARDATA_ANY.
     */
    memcpy(VARDATA(new_t),          /* destination */
           VARDATA_ANY(t),          /* source */
           VARSIZE_ANY_EXHDR(t));   /* how many bytes */
    PG_RETURN_TEXT_P(new_t);
}

PG_FUNCTION_INFO_V1(concat_text);

Datum
concat_text(PG_FUNCTION_ARGS)
{
    text *arg1 = PG_GETARG_TEXT_PP(0);
    text *arg2 = PG_GETARG_TEXT_PP(1);
    int32 arg1_size = VARSIZE_ANY_EXHDR(arg1);
    int32 arg2_size = VARSIZE_ANY_EXHDR(arg2);
    int32 new_text_size = arg1_size + arg2_size + VARHDRSZ;
    text *new_text = (text *) palloc(new_text_size);

    SET_VARSIZE(new_text, new_text_size);
    memcpy(VARDATA(new_text), VARDATA_ANY(arg1), arg1_size);
    memcpy(VARDATA(new_text) + arg1_size, VARDATA_ANY(arg2),
           arg2_size);
    PG_RETURN_TEXT_P(new_text);
}

/* A wrapper around starts_with(text, text) */

PG_FUNCTION_INFO_V1(t_starts_with);

Datum
t_starts_with(PG_FUNCTION_ARGS)
{
    text             *t1 = PG_GETARG_TEXT_PP(0);
    text             *t2 = PG_GETARG_TEXT_PP(1);
    Oid               collid = PG_GET_COLLATION();
    bool              result;

    result = DatumGetBool(DirectFunctionCall2Coll(text_starts_with,
                                                  collid,
                                                  PointerGetDatum(t1),
                                                  PointerGetDatum(t2)));
    PG_RETURN_BOOL(result);
}
```

Angenommen, der obige Code wurde in der Datei `funcs.c` vorbereitet und zu einem Shared Object kompiliert, dann könnten wir die Funktionen mit Befehlen wie diesen in PostgreSQL definieren:

```sql
CREATE FUNCTION add_one(integer) RETURNS integer
    AS 'DIRECTORY/funcs', 'add_one'
    LANGUAGE C STRICT;

-- note overloading of SQL function name "add_one"
CREATE FUNCTION add_one(double precision) RETURNS double precision
    AS 'DIRECTORY/funcs', 'add_one_float8'
    LANGUAGE C STRICT;

CREATE FUNCTION makepoint(point, point) RETURNS point
    AS 'DIRECTORY/funcs', 'makepoint'
    LANGUAGE C STRICT;

CREATE FUNCTION copytext(text) RETURNS text
    AS 'DIRECTORY/funcs', 'copytext'
    LANGUAGE C STRICT;

CREATE FUNCTION concat_text(text, text) RETURNS text
    AS 'DIRECTORY/funcs', 'concat_text'
    LANGUAGE C STRICT;

CREATE FUNCTION t_starts_with(text, text) RETURNS boolean
    AS 'DIRECTORY/funcs', 't_starts_with'
    LANGUAGE C STRICT;
```

Hier steht `DIRECTORY` für das Verzeichnis der Shared-Library-Datei, zum Beispiel das PostgreSQL-Tutorialverzeichnis, das den Code für die in diesem Abschnitt verwendeten Beispiele enthält. Besserer Stil wäre es, in der `AS`-Klausel nur `'funcs'` zu verwenden, nachdem `DIRECTORY` zum Suchpfad hinzugefügt wurde. In jedem Fall können wir die systemspezifische Endung für eine Shared Library, üblicherweise `.so`, weglassen.

Beachten Sie, dass wir die Funktionen als »strict« angegeben haben. Das bedeutet, dass das System automatisch ein Nullergebnis annimmt, wenn irgendein Eingabewert null ist. Dadurch vermeiden wir, im Funktionscode auf Nulleingaben prüfen zu müssen. Ohne dies müssten wir mit `PG_ARGISNULL()` explizit auf Nullwerte prüfen.

Das Makro `PG_ARGISNULL(n)` erlaubt einer Funktion zu testen, ob eine Eingabe null ist. Das ist natürlich nur in Funktionen nötig, die nicht als »strict« deklariert sind. Wie bei den `PG_GETARG_xxx()`-Makros werden die Eingabeargumente ab null gezählt. Beachten Sie, dass `PG_GETARG_xxx()` erst ausgeführt werden sollte, nachdem geprüft wurde, dass das Argument nicht null ist. Um ein Nullergebnis zurückzugeben, führen Sie `PG_RETURN_NULL()` aus; das funktioniert sowohl in strikten als auch in nicht strikten Funktionen.

Auf den ersten Blick könnten die Version-1-Programmierkonventionen im Vergleich zu einfachen C-Aufrufkonventionen wie unnötige Verschleierung wirken. Sie ermöglichen jedoch den Umgang mit nullable Argumenten und Rückgabewerten sowie mit »toasted« Werten, also komprimierten oder ausgelagerten Werten.

Weitere Optionen der Version-1-Schnittstelle sind zwei Varianten der `PG_GETARG_xxx()`-Makros. Die erste, `PG_GETARG_xxx_COPY()`, garantiert, eine Kopie des angegebenen Arguments zurückzugeben, in die sicher geschrieben werden kann. Die normalen Makros geben manchmal einen Zeiger auf einen Wert zurück, der physisch in einer Tabelle gespeichert ist und nicht beschrieben werden darf. Die Verwendung der `PG_GETARG_xxx_COPY()`-Makros garantiert ein schreibbares Ergebnis. Die zweite Variante besteht aus den `PG_GETARG_xxx_SLICE()`-Makros, die drei Argumente annehmen. Das erste ist die Nummer des Funktionsarguments wie oben. Das zweite und dritte sind Offset und Länge des zurückzugebenden Segments. Offsets werden ab null gezählt, und eine negative Länge verlangt, dass der Rest des Werts zurückgegeben wird. Diese Makros ermöglichen effizienteren Zugriff auf Teile großer Werte, wenn diese den Speichertyp »external« haben. Der Speichertyp einer Spalte kann mit `ALTER TABLE tablename ALTER COLUMN colname SET STORAGE storagetype` angegeben werden. `storagetype` ist einer von `plain`, `external`, `extended` oder `main`.

Schließlich ermöglichen die Version-1-Funktionsaufrufkonventionen, Mengenergebnisse zurückzugeben ([Abschnitt 36.10.9](#36109-mengen-zurückgeben)) sowie Triggerfunktionen ([Kapitel 37](37_Trigger.md)) und Call-Handler für prozedurale Sprachen ([Kapitel 57](57_Einen_Handler_für_prozedurale_Sprachen_schreiben.md)) zu implementieren. Weitere Einzelheiten finden Sie in `src/backend/utils/fmgr/README` in der Quelldistribution.

### 36.10.4. Code schreiben

Bevor wir uns fortgeschritteneren Themen zuwenden, sollten wir einige Programmierregeln für PostgreSQL-C-Sprachfunktionen besprechen. Es mag zwar möglich sein, in anderen Sprachen als C geschriebene Funktionen in PostgreSQL zu laden, doch ist dies normalerweise schwierig, wenn es überhaupt möglich ist, weil andere Sprachen wie C++, FORTRAN oder Pascal oft nicht dieselbe Aufrufkonvention wie C verwenden. Das heißt, andere Sprachen übergeben Argument- und Rückgabewerte zwischen Funktionen nicht auf dieselbe Weise. Aus diesem Grund nehmen wir an, dass Ihre C-Sprachfunktionen tatsächlich in C geschrieben sind.

Die Grundregeln zum Schreiben und Bauen von C-Funktionen lauten:

- Verwenden Sie `pg_config --includedir-server`, um herauszufinden, wo die PostgreSQL-Server-Headerdateien auf Ihrem System installiert sind, beziehungsweise auf dem System, auf dem Ihre Benutzer arbeiten werden.
- Das Kompilieren und Linken Ihres Codes, sodass er dynamisch in PostgreSQL geladen werden kann, erfordert immer spezielle Flags. Eine detaillierte Erklärung für Ihr jeweiliges Betriebssystem finden Sie in [Abschnitt 36.10.5](#36105-dynamisch-geladene-funktionen-kompilieren-und-linken).
- Denken Sie daran, einen »Magic Block« für Ihre Shared Library zu definieren, wie in [Abschnitt 36.10.1](#36101-dynamisches-laden) beschrieben.
- Verwenden Sie beim Reservieren von Speicher die PostgreSQL-Funktionen `palloc` und `pfree` statt der entsprechenden C-Bibliotheksfunktionen `malloc` und `free`. Der mit `palloc` reservierte Speicher wird am Ende jeder Transaktion automatisch freigegeben, wodurch Speicherlecks vermieden werden.
- Setzen Sie die Bytes Ihrer Strukturen immer mit `memset` auf null, oder reservieren Sie sie von vornherein mit `palloc0`. Selbst wenn Sie jedem Feld Ihrer Struktur einen Wert zuweisen, kann es Alignment-Padding geben, also Lücken in der Struktur, die Datenmüll enthalten. Ohne dies ist es schwierig, Hash-Indizes oder Hash-Joins zu unterstützen, da Sie nur die signifikanten Bits Ihrer Datenstruktur zur Berechnung eines Hashes heranziehen müssen. Der Planner verlässt sich manchmal auch darauf, Konstanten über bitweise Gleichheit zu vergleichen; daher können unerwünschte Planungsergebnisse entstehen, wenn logisch gleichwertige Werte nicht bitweise gleich sind.
- Die meisten internen PostgreSQL-Typen sind in `postgres.h` deklariert, während die Function-Manager-Schnittstellen (`PG_FUNCTION_ARGS` usw.) in `fmgr.h` liegen. Sie müssen daher mindestens diese beiden Dateien einbinden. Aus Portabilitätsgründen ist es am besten, `postgres.h` zuerst einzubinden, vor allen anderen System- oder Benutzer-Headerdateien. Das Einbinden von `postgres.h` bindet außerdem `elog.h` und `palloc.h` für Sie ein.
- Symbolnamen, die innerhalb von Objektdateien definiert sind, dürfen weder miteinander noch mit Symbolen kollidieren, die im ausführbaren PostgreSQL-Serverprogramm definiert sind. Sie müssen Ihre Funktionen oder Variablen umbenennen, wenn Sie entsprechende Fehlermeldungen erhalten.

### 36.10.5. Dynamisch geladene Funktionen kompilieren und linken

Bevor Sie Ihre in C geschriebenen PostgreSQL-Erweiterungsfunktionen verwenden können, müssen sie auf besondere Weise kompiliert und gelinkt werden, sodass eine Datei entsteht, die der Server dynamisch laden kann. Genauer gesagt muss eine Shared Library erstellt werden.

Für Informationen, die über diesen Abschnitt hinausgehen, sollten Sie die Dokumentation Ihres Betriebssystems lesen, insbesondere die Handbuchseiten für den C-Compiler `cc` und den Link-Editor `ld`. Außerdem enthält der PostgreSQL-Quellcode im Verzeichnis `contrib` mehrere funktionierende Beispiele. Wenn Sie sich auf diese Beispiele stützen, machen Sie Ihre Module allerdings von der Verfügbarkeit des PostgreSQL-Quellcodes abhängig.

Das Erstellen von Shared Libraries entspricht im Allgemeinen dem Linken ausführbarer Programme: Zuerst werden die Quelldateien zu Objektdateien kompiliert, danach werden die Objektdateien zusammen gelinkt. Die Objektdateien müssen als position-independent code (PIC) erzeugt werden. Konzeptionell bedeutet das, dass sie beim Laden durch das ausführbare Programm an eine beliebige Stelle im Speicher gelegt werden können. Objektdateien für ausführbare Programme werden normalerweise nicht so kompiliert. Der Befehl zum Linken einer Shared Library enthält besondere Flags, die ihn vom Linken eines ausführbaren Programms unterscheiden, zumindest in der Theorie; auf manchen Systemen ist die Praxis deutlich unschöner.

In den folgenden Beispielen nehmen wir an, dass Ihr Quellcode in der Datei `foo.c` steht und dass wir eine Shared Library `foo.so` erzeugen. Die Zwischen-Objektdatei heißt, sofern nicht anders angegeben, `foo.o`. Eine Shared Library kann mehr als eine Objektdatei enthalten; hier verwenden wir jedoch nur eine.

FreeBSD

Das Compiler-Flag zum Erzeugen von PIC ist `-fPIC`. Zum Erzeugen von Shared Libraries lautet das Compiler-Flag `-shared`.

```sh
cc -fPIC -c foo.c
cc -shared -o foo.so foo.o
```

Dies gilt ab FreeBSD 13.0; ältere Versionen verwendeten den Compiler `gcc`.

Linux

Das Compiler-Flag zum Erzeugen von PIC ist `-fPIC`. Das Compiler-Flag zum Erzeugen einer Shared Library ist `-shared`. Ein vollständiges Beispiel sieht so aus:

```sh
cc -fPIC -c foo.c
cc -shared -o foo.so foo.o
```

macOS

Hier ist ein Beispiel. Es setzt voraus, dass die Entwicklerwerkzeuge installiert sind.

```sh
cc -c foo.c
cc -bundle -flat_namespace -undefined suppress -o foo.so foo.o
```

NetBSD

Das Compiler-Flag zum Erzeugen von PIC ist `-fPIC`. Auf ELF-Systemen wird der Compiler mit dem Flag `-shared` zum Linken von Shared Libraries verwendet. Auf älteren Nicht-ELF-Systemen wird `ld -Bshareable` verwendet.

```sh
gcc -fPIC -c foo.c
gcc -shared -o foo.so foo.o
```

OpenBSD

Das Compiler-Flag zum Erzeugen von PIC ist `-fPIC`. Zum Linken von Shared Libraries wird `ld -Bshareable` verwendet.

```sh
gcc -fPIC -c foo.c
ld -Bshareable -o foo.so foo.o
```

Solaris

Das Compiler-Flag zum Erzeugen von PIC ist `-KPIC` beim Sun-Compiler und `-fPIC` bei GCC. Zum Linken von Shared Libraries lautet die Compiler-Option bei beiden Compilern `-G`; mit GCC kann alternativ `-shared` verwendet werden.

```sh
cc -KPIC -c foo.c
cc -G -o foo.so foo.o
```

oder

```sh
gcc -fPIC -c foo.c
gcc -G -o foo.so foo.o
```

> Tipp: Wenn Ihnen das zu kompliziert ist, sollten Sie GNU Libtool (<https://www.gnu.org/software/libtool/>) in Betracht ziehen. Es verbirgt die Plattformunterschiede hinter einer einheitlichen Schnittstelle.

Die resultierende Shared-Library-Datei kann anschließend in PostgreSQL geladen werden. Wenn Sie den Dateinamen im Befehl `CREATE FUNCTION` angeben, müssen Sie den Namen der Shared-Library-Datei angeben, nicht den der Zwischen-Objektdatei. Beachten Sie, dass die Standardsuffixe des Systems für Shared Libraries, üblicherweise `.so` oder `.sl`, im Befehl `CREATE FUNCTION` weggelassen werden können und aus Gründen der Portierbarkeit normalerweise weggelassen werden sollten.

Siehe erneut [Abschnitt 36.10.1](#36101-dynamisches-laden), wo beschrieben ist, an welcher Stelle der Server Shared-Library-Dateien erwartet.

### 36.10.6. Hinweise zur Stabilität von Server-API und -ABI

Dieser Abschnitt enthält Hinweise für Autoren von Erweiterungen und anderen Server-Plugins zur API- und ABI-Stabilität im PostgreSQL-Server.

#### 36.10.6.1. Allgemeines

Der PostgreSQL-Server enthält mehrere klar abgegrenzte APIs für Server-Plugins, etwa den Function Manager (`fmgr`, in diesem Kapitel beschrieben), SPI ([Kapitel 45](45_Server_Programming_Interface.md)) sowie verschiedene Hooks, die speziell für Erweiterungen vorgesehen sind. Diese Schnittstellen werden sorgfältig im Hinblick auf langfristige Stabilität und Kompatibilität gepflegt. Allerdings bildet die Gesamtheit der globalen Funktionen und Variablen im Server faktisch die öffentlich nutzbare API, und der größte Teil davon wurde nicht mit Erweiterbarkeit und langfristiger Stabilität als Ziel entworfen.

Deshalb ist es zwar zulässig, diese Schnittstellen zu nutzen; je weiter man sich jedoch vom gut ausgetretenen Pfad entfernt, desto wahrscheinlicher wird es, dass irgendwann API- oder ABI-Kompatibilitätsprobleme auftreten. Autoren von Erweiterungen werden ermutigt, Rückmeldung zu ihren Anforderungen zu geben, damit im Laufe der Zeit, wenn neue Nutzungsmuster entstehen, bestimmte Schnittstellen als stabiler betrachtet oder neue, besser entworfene Schnittstellen hinzugefügt werden können.

#### 36.10.6.2. API-Kompatibilität

Die API, also das Application Programming Interface, ist die zur Kompilierzeit verwendete Schnittstelle.

##### 36.10.6.2.1. Hauptversionen

Zwischen PostgreSQL-Hauptversionen wird keine API-Kompatibilität zugesichert. Erweiterungscode kann daher Quellcodeänderungen benötigen, um mit mehreren Hauptversionen zu funktionieren. Diese lassen sich in der Regel mit Präprozessorbedingungen wie `#if PG_VERSION_NUM >= 160000` handhaben. Anspruchsvollere Erweiterungen, die Schnittstellen außerhalb der klar abgegrenzten APIs verwenden, benötigen meist einige solcher Änderungen für jede Hauptversion des Servers.

##### 36.10.6.2.2. Nebenversionen

PostgreSQL bemüht sich, Brüche der Server-API in Nebenversionen zu vermeiden. Im Allgemeinen sollte Erweiterungscode, der mit einer Nebenversion kompiliert und funktioniert, auch mit jeder anderen Nebenversion derselben Hauptversion kompilieren und funktionieren, unabhängig davon, ob diese älter oder neuer ist.

Wenn eine Änderung erforderlich ist, wird sie sorgfältig verwaltet und berücksichtigt die Anforderungen von Erweiterungen. Solche Änderungen werden in den Release Notes ([Anhang E](E_Versionshinweise.md)) kommuniziert.

#### 36.10.6.3. ABI-Kompatibilität

Die ABI, also das Application Binary Interface, ist die zur Laufzeit verwendete Schnittstelle.

##### 36.10.6.3.1. Hauptversionen

Server unterschiedlicher Hauptversionen haben absichtlich inkompatible ABIs. Erweiterungen, die Server-APIs verwenden, müssen daher für jede Hauptversion neu kompiliert werden. Die Einbindung von `PG_MODULE_MAGIC` (siehe [Abschnitt 36.10.1](#36101-dynamisches-laden)) stellt sicher, dass Code, der für eine Hauptversion kompiliert wurde, von anderen Hauptversionen zurückgewiesen wird.

##### 36.10.6.3.2. Nebenversionen

PostgreSQL bemüht sich, Brüche der Server-ABI in Nebenversionen zu vermeiden. Im Allgemeinen sollte eine Erweiterung, die gegen eine beliebige Nebenversion kompiliert wurde, mit jeder anderen Nebenversion derselben Hauptversion funktionieren, unabhängig davon, ob diese älter oder neuer ist.

Wenn eine Änderung erforderlich ist, wählt PostgreSQL die am wenigsten invasive Änderung, etwa indem ein neues Feld in Padding-Bereiche eingefügt oder an das Ende einer Struktur angehängt wird. Solche Änderungen sollten Erweiterungen nicht beeinträchtigen, außer sie verwenden sehr ungewöhnliche Codemuster.

In seltenen Fällen können allerdings selbst solche nicht invasiven Änderungen unpraktikabel oder unmöglich sein. In einem solchen Fall wird die Änderung sorgfältig verwaltet und berücksichtigt die Anforderungen von Erweiterungen. Solche Änderungen werden ebenfalls in den Release Notes ([Anhang E](E_Versionshinweise.md)) dokumentiert.

Beachten Sie jedoch, dass viele Teile des Servers nicht als öffentlich nutzbare APIs entworfen oder gepflegt werden; in den meisten Fällen ist auch die tatsächliche Grenze nicht klar definiert. Wenn dringende Anforderungen entstehen, werden Änderungen in diesen Bereichen naturgemäß mit weniger Rücksicht auf Erweiterungscode vorgenommen als Änderungen an gut definierten und weit verbreiteten Schnittstellen.

Außerdem ist dies mangels automatischer Erkennung solcher Änderungen keine Garantie, auch wenn solche brechenden Änderungen historisch äußerst selten waren.

### 36.10.7. Argumente mit zusammengesetztem Typ

Zusammengesetzte Typen haben kein festes Layout wie C-Strukturen. Instanzen eines zusammengesetzten Typs können Nullfelder enthalten. Außerdem können zusammengesetzte Typen, die Teil einer Vererbungshierarchie sind, andere Felder haben als andere Mitglieder derselben Vererbungshierarchie. Deshalb stellt PostgreSQL eine Funktionsschnittstelle bereit, mit der Felder zusammengesetzter Typen aus C heraus angesprochen werden können.

Angenommen, wir möchten eine Funktion schreiben, um folgende Abfrage zu beantworten:

```sql
SELECT name, c_overpaid(emp, 1500) AS overpaid
    FROM emp
    WHERE name = 'Bill' OR name = 'Sam';
```

Mit den Aufrufkonventionen der Version 1 können wir `c_overpaid` so definieren:

```c
#include "postgres.h"
#include "executor/executor.h"                    /* for GetAttributeByName() */

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(c_overpaid);

Datum
c_overpaid(PG_FUNCTION_ARGS)
{
    HeapTupleHeader t = PG_GETARG_HEAPTUPLEHEADER(0);
    int32           limit = PG_GETARG_INT32(1);
    bool            isnull;
    Datum           salary;

    salary = GetAttributeByName(t, "salary", &isnull);
    if (isnull)
        PG_RETURN_BOOL(false);
    /* Alternatively, we might prefer to do PG_RETURN_NULL() for null salary. */

    PG_RETURN_BOOL(DatumGetInt32(salary) > limit);
}
```

`GetAttributeByName` ist die PostgreSQL-Systemfunktion, die Attribute aus der angegebenen Zeile zurückgibt. Sie hat drei Argumente: das an die Funktion übergebene Argument vom Typ `HeapTupleHeader`, den Namen des gewünschten Attributs und einen Rückgabeparameter, der angibt, ob das Attribut null ist. `GetAttributeByName` gibt einen `Datum`-Wert zurück, den Sie mit der passenden Funktion `DatumGetXXX()` in den richtigen Datentyp umwandeln können. Beachten Sie, dass der Rückgabewert bedeutungslos ist, wenn das Null-Flag gesetzt ist; prüfen Sie das Null-Flag immer, bevor Sie versuchen, mit dem Ergebnis etwas zu tun.

Außerdem gibt es `GetAttributeByNum`, das das Zielattribut nach Spaltennummer statt nach Name auswählt.

Der folgende Befehl deklariert die Funktion `c_overpaid` in SQL:

```sql
CREATE FUNCTION c_overpaid(emp, integer) RETURNS boolean
    AS 'DIRECTORY/funcs', 'c_overpaid'
    LANGUAGE C STRICT;
```

Beachten Sie, dass wir `STRICT` verwendet haben, sodass wir nicht prüfen mussten, ob die Eingabeargumente `NULL` sind.

### 36.10.8. Zeilen zurückgeben (zusammengesetzte Typen)

Um aus einer C-Sprachfunktion eine Zeile oder einen Wert eines zusammengesetzten Typs zurückzugeben, können Sie eine besondere API verwenden, die Makros und Funktionen bereitstellt, um den größten Teil der Komplexität beim Aufbau zusammengesetzter Datentypen zu verbergen. Um diese API zu verwenden, muss die Quelldatei Folgendes einbinden:

```c
#include "funcapi.h"
```

Es gibt zwei Wege, einen zusammengesetzten Datenwert, im Folgenden ein „Tupel“, aufzubauen: aus einem Array von `Datum`-Werten oder aus einem Array von C-Strings, die an die Eingabekonvertierungsfunktionen der Spaltendatentypen des Tupels übergeben werden können. In beiden Fällen müssen Sie zuerst einen `TupleDesc`-Deskriptor für die Tupelstruktur beschaffen oder erzeugen. Wenn Sie mit `Datum`-Werten arbeiten, übergeben Sie den `TupleDesc` an `BlessTupleDesc` und rufen danach für jede Zeile `heap_form_tuple` auf. Wenn Sie mit C-Strings arbeiten, übergeben Sie den `TupleDesc` an `TupleDescGetAttInMetadata` und rufen danach für jede Zeile `BuildTupleFromCStrings` auf. Bei einer Funktion, die eine Menge von Tupeln zurückgibt, können alle Einrichtungsschritte beim ersten Aufruf der Funktion einmalig ausgeführt werden.

Zum Einrichten des benötigten `TupleDesc` stehen mehrere Hilfsfunktionen zur Verfügung. Der empfohlene Weg für die meisten Funktionen, die zusammengesetzte Werte zurückgeben, ist der Aufruf von:

```c
TypeFuncClass get_call_result_type(FunctionCallInfo fcinfo,
                                   Oid *resultTypeId,
                                   TupleDesc *resultTupleDesc)
```

Dabei wird dieselbe `fcinfo`-Struktur übergeben, die auch an die aufgerufene Funktion selbst übergeben wurde. Das setzt natürlich voraus, dass Sie die Aufrufkonventionen der Version 1 verwenden. `resultTypeId` kann als `NULL` oder als Adresse einer lokalen Variablen angegeben werden, die die OID des Ergebnistyps der Funktion aufnehmen soll. `resultTupleDesc` sollte die Adresse einer lokalen `TupleDesc`-Variablen sein. Prüfen Sie, dass das Ergebnis `TYPEFUNC_COMPOSITE` ist; in diesem Fall wurde `resultTupleDesc` mit dem benötigten `TupleDesc` gefüllt. Falls nicht, können Sie einen Fehler der Art „function returning record called in context that cannot accept type record“ melden.

> Tipp: `get_call_result_type` kann den tatsächlichen Typ eines polymorphen Funktionsergebnisses auflösen; daher ist es nicht nur für Funktionen nützlich, die zusammengesetzte Werte zurückgeben, sondern auch für Funktionen, die skalare polymorphe Ergebnisse zurückgeben. Die Ausgabe `resultTypeId` ist vor allem für Funktionen nützlich, die polymorphe Skalare zurückgeben.

> Hinweis: `get_call_result_type` hat mit `get_expr_result_type` eine Schwesterfunktion, die den erwarteten Ausgabetyp eines Funktionsaufrufs auflösen kann, der durch einen Ausdrucksbaum dargestellt wird. Das kann verwendet werden, wenn man den Ergebnistyp von außerhalb der Funktion selbst bestimmen möchte. Außerdem gibt es `get_func_result_type`, das verwendet werden kann, wenn nur die OID der Funktion verfügbar ist. Diese Funktionen können jedoch nicht mit Funktionen umgehen, die als Rückgabetyp `record` deklariert sind, und `get_func_result_type` kann keine polymorphen Typen auflösen; daher sollten Sie bevorzugt `get_call_result_type` verwenden.

Ältere, inzwischen missbilligte Funktionen zum Beschaffen von `TupleDesc`-Werten sind:

```c
TupleDesc RelationNameGetTupleDesc(const char *relname)
```

um einen `TupleDesc` für den Zeilentyp einer benannten Relation zu erhalten, und:

```c
TupleDesc TypeGetTupleDesc(Oid typeoid, List *colaliases)
```

um einen `TupleDesc` anhand einer Typ-OID zu erhalten. Dies kann verwendet werden, um einen `TupleDesc` für einen Basis- oder zusammengesetzten Typ zu erhalten. Für eine Funktion, die `record` zurückgibt, funktioniert es jedoch nicht, und polymorphe Typen kann es nicht auflösen.

Sobald Sie einen `TupleDesc` haben, rufen Sie Folgendes auf:

```c
TupleDesc BlessTupleDesc(TupleDesc tupdesc)
```

wenn Sie mit `Datum`-Werten arbeiten wollen, oder:

```c
AttInMetadata *TupleDescGetAttInMetadata(TupleDesc tupdesc)
```

wenn Sie mit C-Strings arbeiten wollen. Wenn Sie eine mengenrückgebende Funktion schreiben, können Sie die Ergebnisse dieser Funktionen in der Struktur `FuncCallContext` speichern, und zwar im Feld `tuple_desc` beziehungsweise `attinmeta`.

Wenn Sie mit `Datum`-Werten arbeiten, verwenden Sie:

```c
HeapTuple heap_form_tuple(TupleDesc tupdesc, Datum *values, bool *isnull)
```

um aus Benutzerdaten in `Datum`-Form ein `HeapTuple` aufzubauen.

Wenn Sie mit C-Strings arbeiten, verwenden Sie:

```c
HeapTuple BuildTupleFromCStrings(AttInMetadata *attinmeta, char **values)
```

um aus Benutzerdaten in C-String-Form ein `HeapTuple` aufzubauen. `values` ist ein Array von C-Strings, eines für jedes Attribut der zurückgegebenen Zeile. Jeder C-String sollte die Form haben, die die Eingabefunktion des jeweiligen Attributdatentyps erwartet. Um für eines der Attribute einen Nullwert zurückzugeben, muss der entsprechende Zeiger im Array `values` auf `NULL` gesetzt werden. Diese Funktion muss für jede zurückgegebene Zeile erneut aufgerufen werden.

Sobald Sie ein Tupel aufgebaut haben, das Ihre Funktion zurückgeben soll, muss es in ein `Datum` umgewandelt werden. Verwenden Sie:

```c
HeapTupleGetDatum(HeapTuple tuple)
```

um ein `HeapTuple` in ein gültiges `Datum` umzuwandeln. Dieses `Datum` kann direkt zurückgegeben werden, wenn Sie nur eine einzelne Zeile zurückgeben wollen, oder es kann als aktueller Rückgabewert in einer mengenrückgebenden Funktion verwendet werden.

Ein Beispiel erscheint im nächsten Abschnitt.

### 36.10.9. Mengen zurückgeben

C-Sprachfunktionen haben zwei Möglichkeiten, Mengen, also mehrere Zeilen, zurückzugeben. Bei der einen Methode, dem sogenannten ValuePerCall-Modus, wird eine mengenrückgebende Funktion wiederholt aufgerufen, wobei jedes Mal dieselben Argumente übergeben werden. Sie gibt bei jedem Aufruf eine neue Zeile zurück, bis sie keine weiteren Zeilen mehr hat und dies durch die Rückgabe von `NULL` signalisiert. Die mengenrückgebende Funktion (set-returning function, SRF) muss daher über Aufrufe hinweg genug Zustand speichern, um zu wissen, was sie gerade tut, und bei jedem Aufruf das richtige nächste Element zurückzugeben. Bei der anderen Methode, dem Materialize-Modus, füllt eine SRF ein `tuplestore`-Objekt mit ihrem gesamten Ergebnis und gibt es zurück; dann erfolgt für das gesamte Ergebnis nur ein Aufruf, und zwischen den Aufrufen muss kein Zustand gehalten werden.

Bei Verwendung des ValuePerCall-Modus ist wichtig, daran zu denken, dass nicht garantiert ist, dass die Abfrage bis zum Ende ausgeführt wird. Aufgrund von Optionen wie `LIMIT` kann der Executor also aufhören, die mengenrückgebende Funktion aufzurufen, bevor alle Zeilen abgerufen wurden. Das bedeutet, dass es nicht sicher ist, Aufräumarbeiten im letzten Aufruf auszuführen, weil dieser Aufruf möglicherweise nie stattfindet. Für Funktionen, die Zugriff auf externe Ressourcen benötigen, etwa Dateideskriptoren, wird empfohlen, den Materialize-Modus zu verwenden.

Der Rest dieses Abschnitts dokumentiert eine Reihe von Hilfsmakros, die häufig, aber nicht zwingend, für SRFs im ValuePerCall-Modus verwendet werden. Weitere Einzelheiten zum Materialize-Modus finden Sie in `src/backend/utils/fmgr/README`. Außerdem enthalten die `contrib`-Module in der PostgreSQL-Quelldistribution viele Beispiele für SRFs, die sowohl den ValuePerCall- als auch den Materialize-Modus verwenden.

Um die hier beschriebenen ValuePerCall-Hilfsmakros zu verwenden, binden Sie `funcapi.h` ein. Diese Makros arbeiten mit einer Struktur `FuncCallContext`, die den Zustand enthält, der über Aufrufe hinweg gespeichert werden muss. Innerhalb der aufgerufenen SRF wird `fcinfo->flinfo->fn_extra` verwendet, um über Aufrufe hinweg einen Zeiger auf `FuncCallContext` zu halten. Die Makros füllen dieses Feld bei der ersten Verwendung automatisch und erwarten, denselben Zeiger bei späteren Verwendungen dort wiederzufinden.

```c
typedef struct FuncCallContext
{
    /*
     * Number of times we've been called before
     *
     * call_cntr is initialized to 0 for you by SRF_FIRSTCALL_INIT(), and
     * incremented for you every time SRF_RETURN_NEXT() is called.
     */
    uint64 call_cntr;

    /*
     * OPTIONAL maximum number of calls
     *
     * max_calls is here for convenience only and setting it is optional.
     * If not set, you must provide alternative means to know when the
     * function is done.
     */
    uint64 max_calls;

    /*
     * OPTIONAL pointer to miscellaneous user-provided context information
     *
     * user_fctx is for use as a pointer to your own data to retain
     * arbitrary context information between calls of your function.
     */
    void *user_fctx;

    /*
     * OPTIONAL pointer to struct containing attribute type input metadata
     *
     * attinmeta is for use when returning tuples (i.e., composite data types)
     * and is not used when returning base data types. It is only needed
     * if you intend to use BuildTupleFromCStrings() to create the return
     * tuple.
     */
    AttInMetadata *attinmeta;

    /*
     * memory context used for structures that must live for multiple calls
     *
     * multi_call_memory_ctx is set by SRF_FIRSTCALL_INIT() for you, and used
     * by SRF_RETURN_DONE() for cleanup. It is the most appropriate memory
     * context for any memory that is to be reused across multiple calls
     * of the SRF.
     */
    MemoryContext multi_call_memory_ctx;

    /*
     * OPTIONAL pointer to struct containing tuple description
     *
     * tuple_desc is for use when returning tuples (i.e., composite data types)
     * and is only needed if you are going to build the tuples with
     * heap_form_tuple() rather than with BuildTupleFromCStrings(). Note that
     * the TupleDesc pointer stored here should usually have been run through
     * BlessTupleDesc() first.
     */
    TupleDesc tuple_desc;
} FuncCallContext;
```

Die Makros, die eine SRF mit dieser Infrastruktur verwendet, sind:

```c
SRF_IS_FIRSTCALL()
```

Verwenden Sie dies, um festzustellen, ob Ihre Funktion zum ersten oder zu einem späteren Mal aufgerufen wird. Beim ersten Aufruf, und nur dann, rufen Sie Folgendes auf:

```c
SRF_FIRSTCALL_INIT()
```

um den `FuncCallContext` zu initialisieren. Bei jedem Funktionsaufruf, einschließlich des ersten, rufen Sie Folgendes auf:

```c
SRF_PERCALL_SETUP()
```

um die Verwendung des `FuncCallContext` vorzubereiten.

Wenn Ihre Funktion im aktuellen Aufruf Daten zurückzugeben hat, verwenden Sie:

```c
SRF_RETURN_NEXT(funcctx, result)
```

um sie an den Aufrufer zurückzugeben. `result` muss vom Typ `Datum` sein, entweder ein einzelner Wert oder ein Tupel, das wie oben beschrieben vorbereitet wurde. Wenn Ihre Funktion schließlich keine weiteren Daten mehr zurückzugeben hat, verwenden Sie:

```c
SRF_RETURN_DONE(funcctx)
```

um aufzuräumen und die SRF zu beenden.

Der Speicher-Kontext, der aktuell ist, wenn die SRF aufgerufen wird, ist ein transienter Kontext, der zwischen Aufrufen geleert wird. Das bedeutet, dass Sie nicht für alles, was Sie mit `palloc` reserviert haben, `pfree` aufrufen müssen; es verschwindet ohnehin. Wenn Sie jedoch Datenstrukturen reservieren wollen, die über Aufrufe hinweg bestehen bleiben, müssen Sie sie an anderer Stelle unterbringen. Der von `multi_call_memory_ctx` referenzierte Speicher-Kontext ist ein geeigneter Ort für Daten, die bis zum Ende der Ausführung der SRF überleben müssen. In den meisten Fällen bedeutet das, dass Sie während der Einrichtung beim ersten Aufruf in `multi_call_memory_ctx` wechseln sollten. Verwenden Sie `funcctx->user_fctx`, um einen Zeiger auf solche aufrufübergreifenden Datenstrukturen zu halten. Daten, die Sie in `multi_call_memory_ctx` reservieren, verschwinden automatisch, wenn die Abfrage endet; daher ist es auch nicht nötig, diese Daten manuell freizugeben.

> Warnung: Während die tatsächlichen Argumente der Funktion zwischen Aufrufen unverändert bleiben, werden detoastete Kopien bei jedem Zyklus freigegeben, wenn Sie die Argumentwerte im transienten Kontext detoasten, was normalerweise transparent durch das Makro `PG_GETARG_xxx` geschieht. Wenn Sie in Ihrem `user_fctx` Referenzen auf solche Werte aufbewahren, müssen Sie sie daher entweder nach dem Detoasten in den `multi_call_memory_ctx` kopieren oder sicherstellen, dass Sie die Werte nur in diesem Kontext detoasten.

Ein vollständiges Pseudocode-Beispiel sieht so aus:

```c
Datum
my_set_returning_function(PG_FUNCTION_ARGS)
{
    FuncCallContext *funcctx;
    Datum            result;
    further declarations as needed

    if (SRF_IS_FIRSTCALL())
    {
        MemoryContext oldcontext;

        funcctx = SRF_FIRSTCALL_INIT();
        oldcontext = MemoryContextSwitchTo(funcctx->multi_call_memory_ctx);
        /* One-time setup code appears here: */
        user code
        if returning composite
            build TupleDesc, and perhaps AttInMetadata
        endif returning composite
        user code
        MemoryContextSwitchTo(oldcontext);
    }

    /* Each-time setup code appears here: */
    user code
    funcctx = SRF_PERCALL_SETUP();
    user code

    /* this is just one way we might test whether we are done: */
    if (funcctx->call_cntr < funcctx->max_calls)
    {
        /* Here we want to return another item: */
        user code
        obtain result Datum
        SRF_RETURN_NEXT(funcctx, result);
    }
    else
    {
        /* Here we are done returning items, so just report that fact. */
        /* (Resist the temptation to put cleanup code here.) */
        SRF_RETURN_DONE(funcctx);
    }
}
```

Ein vollständiges Beispiel für eine einfache SRF, die einen zusammengesetzten Typ zurückgibt, sieht so aus:

```c
PG_FUNCTION_INFO_V1(retcomposite);

Datum
retcomposite(PG_FUNCTION_ARGS)
{
    FuncCallContext *funcctx;
    int              call_cntr;
    int              max_calls;
    TupleDesc        tupdesc;
    AttInMetadata   *attinmeta;

    /* stuff done only on the first call of the function */
    if (SRF_IS_FIRSTCALL())
    {
        MemoryContext oldcontext;

        /* create a function context for cross-call persistence */
        funcctx = SRF_FIRSTCALL_INIT();

        /* switch to memory context appropriate for multiple function calls */
        oldcontext = MemoryContextSwitchTo(funcctx->multi_call_memory_ctx);

        /* total number of tuples to be returned */
        funcctx->max_calls = PG_GETARG_INT32(0);

        /* Build a tuple descriptor for our result type */
        if (get_call_result_type(fcinfo, NULL, &tupdesc) != TYPEFUNC_COMPOSITE)
            ereport(ERROR,
                    (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                     errmsg("function returning record called in context "
                            "that cannot accept type record")));

        /*
         * generate attribute metadata needed later to produce tuples from raw
         * C strings
         */
        attinmeta = TupleDescGetAttInMetadata(tupdesc);
        funcctx->attinmeta = attinmeta;

        MemoryContextSwitchTo(oldcontext);
    }

    /* stuff done on every call of the function */
    funcctx = SRF_PERCALL_SETUP();

    call_cntr = funcctx->call_cntr;
    max_calls = funcctx->max_calls;
    attinmeta = funcctx->attinmeta;

    if (call_cntr < max_calls)      /* do when there is more left to send */
    {
        char      **values;
        HeapTuple   tuple;
        Datum       result;

        /*
         * Prepare a values array for building the returned tuple.
         * This should be an array of C strings which will
         * be processed later by the type input functions.
         */
        values = (char **) palloc(3 * sizeof(char *));
        values[0] = (char *) palloc(16 * sizeof(char));
        values[1] = (char *) palloc(16 * sizeof(char));
        values[2] = (char *) palloc(16 * sizeof(char));

        snprintf(values[0], 16, "%d", 1 * PG_GETARG_INT32(1));
        snprintf(values[1], 16, "%d", 2 * PG_GETARG_INT32(1));
        snprintf(values[2], 16, "%d", 3 * PG_GETARG_INT32(1));

        /* build a tuple */
        tuple = BuildTupleFromCStrings(attinmeta, values);

        /* make the tuple into a datum */
        result = HeapTupleGetDatum(tuple);

        /* clean up (this is not really necessary) */
        pfree(values[0]);
        pfree(values[1]);
        pfree(values[2]);
        pfree(values);

        SRF_RETURN_NEXT(funcctx, result);
    }
    else    /* do when there is no more left */
    {
        SRF_RETURN_DONE(funcctx);
    }
}
```

Eine Möglichkeit, diese Funktion in SQL zu deklarieren, ist:

```sql
CREATE TYPE __retcomposite AS (f1 integer, f2 integer, f3 integer);

CREATE OR REPLACE FUNCTION retcomposite(integer, integer)
    RETURNS SETOF __retcomposite
    AS 'filename', 'retcomposite'
    LANGUAGE C IMMUTABLE STRICT;
```

Eine andere Möglichkeit ist die Verwendung von `OUT`-Parametern:

```sql
CREATE OR REPLACE FUNCTION retcomposite(IN integer, IN integer,
    OUT f1 integer, OUT f2 integer, OUT f3 integer)
    RETURNS SETOF record
    AS 'filename', 'retcomposite'
    LANGUAGE C IMMUTABLE STRICT;
```

Beachten Sie, dass der Ausgabetyp der Funktion bei dieser Methode formal ein anonymer Record-Typ ist.

### 36.10.10. Polymorphe Argumente und Rückgabetypen

C-Sprachfunktionen können so deklariert werden, dass sie die in [Abschnitt 36.2.5](#3625-polymorphe-typen) beschriebenen polymorphen Typen annehmen und zurückgeben. Wenn die Argumente oder Rückgabetypen einer Funktion als polymorphe Typen definiert sind, kann der Autor der Funktion nicht im Voraus wissen, mit welchem Datentyp sie aufgerufen wird oder welchen Datentyp sie zurückgeben muss. In `fmgr.h` werden zwei Routinen bereitgestellt, mit denen eine C-Funktion der Version 1 die tatsächlichen Datentypen ihrer Argumente und den erwarteten Rückgabetyp herausfinden kann. Die Routinen heißen `get_fn_expr_rettype(FmgrInfo *flinfo)` und `get_fn_expr_argtype(FmgrInfo *flinfo, int argnum)`. Sie geben die OID des Ergebnis- oder Argumenttyps zurück oder `InvalidOid`, wenn die Information nicht verfügbar ist. Auf die Struktur `flinfo` wird normalerweise als `fcinfo->flinfo` zugegriffen. Der Parameter `argnum` ist nullbasiert. `get_call_result_type` kann ebenfalls als Alternative zu `get_fn_expr_rettype` verwendet werden. Außerdem gibt es `get_fn_expr_variadic`, mit dem festgestellt werden kann, ob variadische Argumente zu einem Array zusammengeführt wurden. Das ist vor allem für Funktionen mit `VARIADIC "any"` nützlich, da eine solche Zusammenführung bei variadischen Funktionen, die normale Array-Typen annehmen, immer bereits erfolgt ist.

Nehmen wir zum Beispiel an, wir möchten eine Funktion schreiben, die ein einzelnes Element beliebigen Typs annimmt und ein eindimensionales Array dieses Typs zurückgibt:

```c
PG_FUNCTION_INFO_V1(make_array);

Datum
make_array(PG_FUNCTION_ARGS)
{
    ArrayType *result;
    Oid        element_type = get_fn_expr_argtype(fcinfo->flinfo, 0);
    Datum      element;
    bool       isnull;
    int16      typlen;
    bool       typbyval;
    char       typalign;
    int        ndims;
    int        dims[MAXDIM];
    int        lbs[MAXDIM];

    if (!OidIsValid(element_type))
        elog(ERROR, "could not determine data type of input");

    /* get the provided element, being careful in case it's NULL */
    isnull = PG_ARGISNULL(0);
    if (isnull)
        element = (Datum) 0;
    else
        element = PG_GETARG_DATUM(0);

    /* we have one dimension */
    ndims = 1;
    /* and one element */
    dims[0] = 1;
    /* and lower bound is 1 */
    lbs[0] = 1;

    /* get required info about the element type */
    get_typlenbyvalalign(element_type, &typlen, &typbyval, &typalign);

    /* now build the array */
    result = construct_md_array(&element, &isnull, ndims, dims, lbs,
                                element_type, typlen, typbyval, typalign);

    PG_RETURN_ARRAYTYPE_P(result);
}
```

Der folgende Befehl deklariert die Funktion `make_array` in SQL:

```sql
CREATE FUNCTION make_array(anyelement) RETURNS anyarray
    AS 'DIRECTORY/funcs', 'make_array'
    LANGUAGE C IMMUTABLE;
```

Es gibt eine Variante von Polymorphie, die nur C-Sprachfunktionen zur Verfügung steht: Sie können so deklariert werden, dass sie Parameter des Typs `"any"` annehmen. Beachten Sie, dass dieser Typname in doppelte Anführungszeichen gesetzt werden muss, da er auch ein reserviertes SQL-Wort ist. Das funktioniert ähnlich wie `anyelement`, außer dass verschiedene `"any"`-Argumente nicht auf denselben Typ eingeschränkt werden und auch nicht helfen, den Ergebnistyp der Funktion zu bestimmen. Eine C-Sprachfunktion kann außerdem ihren letzten Parameter als `VARIADIC "any"` deklarieren. Das passt auf ein oder mehrere tatsächliche Argumente beliebigen Typs, die nicht notwendigerweise denselben Typ haben. Diese Argumente werden nicht wie bei normalen variadischen Funktionen in einem Array gesammelt; sie werden einfach getrennt an die Funktion übergeben. Das Makro `PG_NARGS()` und die oben beschriebenen Methoden müssen verwendet werden, um bei dieser Funktionalität die Anzahl der tatsächlichen Argumente und ihre Typen zu bestimmen. Außerdem möchten Benutzer einer solchen Funktion möglicherweise das Schlüsselwort `VARIADIC` im Funktionsaufruf verwenden, in der Erwartung, dass die Funktion die Array-Elemente als separate Argumente behandelt. Die Funktion selbst muss dieses Verhalten, falls gewünscht, implementieren, nachdem sie mit `get_fn_expr_variadic` erkannt hat, dass das tatsächliche Argument mit `VARIADIC` markiert wurde.

### 36.10.11. Gemeinsam genutzter Speicher

#### 36.10.11.1. Shared Memory beim Start anfordern

Add-ins können beim Serverstart Shared Memory reservieren. Dazu muss die Shared Library des Add-ins vorgeladen werden, indem sie in `shared_preload_libraries` angegeben wird. Die Shared Library sollte außerdem in ihrer Funktion `_PG_init` einen `shmem_request_hook` registrieren. Dieser `shmem_request_hook` kann Shared Memory durch folgenden Aufruf reservieren:

```c
void RequestAddinShmemSpace(Size size)
```

Jedes Backend sollte einen Zeiger auf den reservierten Shared Memory durch folgenden Aufruf erhalten:

```c
void *ShmemInitStruct(const char *name, Size size, bool *foundPtr)
```

Wenn diese Funktion `foundPtr` auf `false` setzt, sollte der Aufrufer fortfahren und den Inhalt des reservierten Shared Memory initialisieren. Wenn `foundPtr` auf `true` gesetzt wird, wurde der Shared Memory bereits von einem anderen Backend initialisiert, und der Aufrufer muss nicht weiter initialisieren.

Um Race Conditions zu vermeiden, sollte jedes Backend beim Initialisieren seiner Shared-Memory-Zuweisung das LWLock `AddinShmemInitLock` verwenden, wie hier gezeigt:

```c
static mystruct *ptr = NULL;
bool             found;

LWLockAcquire(AddinShmemInitLock, LW_EXCLUSIVE);
ptr = ShmemInitStruct("my struct name", size, &found);
if (!found)
{
    ... initialize contents of shared memory ...
    ptr->locks = GetNamedLWLockTranche("my tranche name");
}
LWLockRelease(AddinShmemInitLock);
```

`shmem_startup_hook` bietet einen praktischen Ort für den Initialisierungscode, es ist aber nicht zwingend erforderlich, den gesamten entsprechenden Code in diesem Hook unterzubringen. Unter Windows, und überall sonst, wo `EXEC_BACKEND` definiert ist, führt jedes Backend den registrierten `shmem_startup_hook` kurz nach dem Anhängen an den Shared Memory aus. Add-ins sollten daher innerhalb dieses Hooks weiterhin `AddinShmemInitLock` erwerben, wie im obigen Beispiel gezeigt. Auf anderen Plattformen führt nur der Postmaster-Prozess den `shmem_startup_hook` aus, und jedes Backend erbt die Zeiger auf Shared Memory automatisch.

Ein Beispiel für einen `shmem_request_hook` und einen `shmem_startup_hook` findet sich in `contrib/pg_stat_statements/pg_stat_statements.c` im PostgreSQL-Quellbaum.

#### 36.10.11.2. Shared Memory nach dem Start anfordern

Es gibt eine weitere, flexiblere Methode, Shared Memory zu reservieren, die nach dem Serverstart und außerhalb eines `shmem_request_hook` verwendet werden kann. Dazu sollte jedes Backend, das den Shared Memory verwenden wird, durch folgenden Aufruf einen Zeiger darauf erhalten:

```c
void *GetNamedDSMSegment(const char *name, size_t size,
                         void (*init_callback) (void *ptr),
                         bool *found)
```

Wenn ein dynamisches Shared-Memory-Segment mit dem angegebenen Namen noch nicht existiert, reserviert diese Funktion es und initialisiert es mit der bereitgestellten Callback-Funktion `init_callback`. Wenn das Segment bereits von einem anderen Backend reserviert und initialisiert wurde, hängt diese Funktion das vorhandene dynamische Shared-Memory-Segment einfach an das aktuelle Backend an.

Anders als bei Shared Memory, der beim Serverstart reserviert wird, ist es nicht nötig, `AddinShmemInitLock` zu erwerben oder auf andere Weise Race Conditions zu vermeiden, wenn Shared Memory mit `GetNamedDSMSegment` reserviert wird. Diese Funktion stellt sicher, dass nur ein Backend das Segment reserviert und initialisiert und dass alle anderen Backends einen Zeiger auf das vollständig reservierte und initialisierte Segment erhalten.

Ein vollständiges Nutzungsbeispiel für `GetNamedDSMSegment` findet sich in `src/test/modules/test_dsm_registry/test_dsm_registry.c` im PostgreSQL-Quellbaum.

### 36.10.12. LWLocks

#### 36.10.12.1. LWLocks beim Start anfordern

Add-ins können beim Serverstart LWLocks reservieren. Wie bei beim Serverstart reserviertem Shared Memory muss die Shared Library des Add-ins vorgeladen werden, indem sie in `shared_preload_libraries` angegeben wird, und die Shared Library sollte in ihrer Funktion `_PG_init` einen `shmem_request_hook` registrieren. Dieser `shmem_request_hook` kann LWLocks durch folgenden Aufruf reservieren:

```c
void RequestNamedLWLockTranche(const char *tranche_name, int num_lwlocks)
```

Dies stellt sicher, dass unter dem Namen `tranche_name` ein Array von `num_lwlocks` LWLocks verfügbar ist. Ein Zeiger auf dieses Array kann durch folgenden Aufruf erhalten werden:

```c
LWLockPadded *GetNamedLWLockTranche(const char *tranche_name)
```

#### 36.10.12.2. LWLocks nach dem Start anfordern

Es gibt eine weitere, flexiblere Methode, LWLocks zu erhalten, die nach dem Serverstart und außerhalb eines `shmem_request_hook` verwendet werden kann. Reservieren Sie dazu zunächst eine `tranche_id` durch folgenden Aufruf:

```c
int LWLockNewTrancheId(void)
```

Initialisieren Sie anschließend jedes LWLock und übergeben Sie die neue `tranche_id` als Argument:

```c
void LWLockInitialize(LWLock *lock, int tranche_id)
```

Ähnlich wie bei Shared Memory sollte jedes Backend sicherstellen, dass nur ein Prozess eine neue `tranche_id` reserviert und jedes neue LWLock initialisiert. Eine Möglichkeit dafür ist, diese Funktionen nur in Ihrem Shared-Memory-Initialisierungscode aufzurufen, während `AddinShmemInitLock` exklusiv gehalten wird. Bei Verwendung von `GetNamedDSMSegment` reicht es aus, diese Funktionen in der Callback-Funktion `init_callback` aufzurufen, um Race Conditions zu vermeiden.

Schließlich sollte jedes Backend, das die `tranche_id` verwendet, sie durch folgenden Aufruf mit einem `tranche_name` verknüpfen:

```c
void LWLockRegisterTranche(int tranche_id, const char *tranche_name)
```

Ein vollständiges Nutzungsbeispiel für `LWLockNewTrancheId`, `LWLockInitialize` und `LWLockRegisterTranche` findet sich in `contrib/pg_prewarm/autoprewarm.c` im PostgreSQL-Quellbaum.

### 36.10.13. Benutzerdefinierte Wait Events

Add-ins können unter dem Wait-Event-Typ `Extension` benutzerdefinierte Wait Events definieren, indem sie Folgendes aufrufen:

```c
uint32 WaitEventExtensionNew(const char *wait_event_name)
```

Das Wait Event wird mit einer benutzerseitig sichtbaren Zeichenkette verknüpft. Ein Beispiel findet sich in `src/test/modules/worker_spi` im PostgreSQL-Quellbaum.

Benutzerdefinierte Wait Events können in `pg_stat_activity` angezeigt werden:

```sql
SELECT wait_event_type, wait_event FROM pg_stat_activity
    WHERE backend_type ~ 'worker_spi';
```

```text
 wait_event_type | wait_event
-----------------+---------------
 Extension       | WorkerSpiMain
(1 row)
```

### 36.10.14. Injection Points

Ein Injection Point mit einem bestimmten Namen wird mit folgendem Makro deklariert:

```c
INJECTION_POINT(name, arg);
```

Einige Injection Points sind bereits an strategischen Stellen im Servercode deklariert. Nachdem ein neuer Injection Point hinzugefügt wurde, muss der Code kompiliert werden, damit dieser Injection Point im Binärprogramm verfügbar ist. In C geschriebene Add-ins können Injection Points in ihrem eigenen Code mit demselben Makro deklarieren. Die Namen von Injection Points sollten Kleinbuchstaben verwenden, wobei Begriffe durch Bindestriche getrennt werden. `arg` ist ein optionaler Argumentwert, der dem Callback zur Laufzeit übergeben wird.

Das Ausführen eines Injection Points kann die Reservierung einer kleinen Menge Speicher erfordern, was fehlschlagen kann. Wenn Sie einen Injection Point in einem kritischen Abschnitt benötigen, in dem dynamische Speicherreservierungen nicht erlaubt sind, können Sie mit den folgenden Makros einen zweistufigen Ansatz verwenden:

```c
INJECTION_POINT_LOAD(name);
INJECTION_POINT_CACHED(name, arg);
```

Rufen Sie vor dem Betreten des kritischen Abschnitts `INJECTION_POINT_LOAD` auf. Es prüft den Zustand im Shared Memory und lädt den Callback in backend-privaten Speicher, falls er aktiv ist. Innerhalb des kritischen Abschnitts verwenden Sie `INJECTION_POINT_CACHED`, um den Callback auszuführen.

Add-ins können Callbacks an einen bereits deklarierten Injection Point anhängen, indem sie Folgendes aufrufen:

```c
extern void InjectionPointAttach(const char *name,
                                 const char *library,
                                 const char *function,
                                 const void *private_data,
                                 int private_data_size);
```

`name` ist der Name des Injection Points, der bei Erreichen während der Ausführung die aus `library` geladene Funktion ausführt. `private_data` ist ein privater Datenbereich der Größe `private_data_size`, der dem Callback bei der Ausführung als Argument übergeben wird.

Hier ist ein Beispiel für einen Callback für `InjectionPointCallback`:

```c
static void
custom_injection_callback(const char *name,
                          const void *private_data,
                          void *arg)
{
    uint32 wait_event_info = WaitEventInjectionPointNew(name);

    pgstat_report_wait_start(wait_event_info);
    elog(NOTICE, "%s: executed custom callback", name);
    pgstat_report_wait_end();
}
```

Dieser Callback schreibt eine Meldung mit dem Schweregrad `NOTICE` in das Server-Fehlerprotokoll, Callbacks können aber auch komplexere Logik implementieren.

Eine alternative Möglichkeit, die beim Erreichen eines Injection Points auszuführende Aktion zu definieren, besteht darin, den Testcode neben den normalen Quellcode zu setzen. Das kann nützlich sein, wenn die Aktion beispielsweise von lokalen Variablen abhängt, die für geladene Module nicht zugänglich sind. Das Makro `IS_INJECTION_POINT_ATTACHED` kann dann verwendet werden, um zu prüfen, ob ein Injection Point angehängt ist, zum Beispiel:

```c
#ifdef USE_INJECTION_POINTS
if (IS_INJECTION_POINT_ATTACHED("before-foobar"))
{
    /* change a local variable if injection point is attached */
    local_var = 123;

    /* also execute the callback */
    INJECTION_POINT_CACHED("before-foobar", NULL);
}
#endif
```

Beachten Sie, dass der an den Injection Point angehängte Callback nicht durch das Makro `IS_INJECTION_POINT_ATTACHED` ausgeführt wird. Wenn Sie den Callback ausführen möchten, müssen Sie zusätzlich `INJECTION_POINT_CACHED` wie im obigen Beispiel aufrufen.

Optional ist es möglich, einen Injection Point durch folgenden Aufruf wieder zu lösen:

```c
extern bool InjectionPointDetach(const char *name);
```

Bei Erfolg wird `true` zurückgegeben, andernfalls `false`.

Ein an einen Injection Point angehängter Callback ist in allen Backends verfügbar, einschließlich der Backends, die nach dem Aufruf von `InjectionPointAttach` gestartet werden. Er bleibt angehängt, solange der Server läuft oder bis der Injection Point mit `InjectionPointDetach` gelöst wird.

Ein Beispiel findet sich in `src/test/modules/injection_points` im PostgreSQL-Quellbaum.

Das Aktivieren von Injection Points erfordert `--enable-injection-points` mit `configure` oder `-Dinjection_points=true` mit Meson.

### 36.10.15. Benutzerdefinierte kumulative Statistiken

In C geschriebene Add-ins können benutzerdefinierte Typen kumulativer Statistiken verwenden, die im Cumulative Statistics System registriert sind.

Definieren Sie zunächst eine `PgStat_KindInfo`, die alle Informationen zu dem registrierten benutzerdefinierten Typ enthält. Zum Beispiel:

```c
static const PgStat_KindInfo custom_stats = {
    .name = "custom_stats",
    .fixed_amount = false,
    .shared_size = sizeof(PgStatShared_Custom),
    .shared_data_off = offsetof(PgStatShared_Custom, stats),
    .shared_data_len = sizeof(((PgStatShared_Custom *) 0)->stats),
    .pending_size = sizeof(PgStat_StatCustomEntry),
}
```

Danach muss jedes Backend, das diesen benutzerdefinierten Typ verwenden muss, ihn mit `pgstat_register_kind` und einer eindeutigen ID registrieren, die zum Speichern der Einträge für diesen Statistiktyp verwendet wird:

```c
extern PgStat_Kind pgstat_register_kind(PgStat_Kind kind,
                                        const PgStat_KindInfo *kind_info);
```

Während der Entwicklung einer neuen Erweiterung verwenden Sie `PGSTAT_KIND_EXPERIMENTAL` für `kind`. Wenn Sie bereit sind, die Erweiterung für Benutzer freizugeben, reservieren Sie eine Kind-ID auf der Seite Custom Cumulative Statistics (<https://wiki.postgresql.org/wiki/CustomCumulativeStats>).

Die Details der API für `PgStat_KindInfo` finden sich in `src/include/utils/pgstat_internal.h`.

Der registrierte Statistiktyp wird mit einem Namen und einer eindeutigen ID verknüpft, die serverweit im Shared Memory geteilt werden. Jedes Backend, das einen benutzerdefinierten Statistiktyp verwendet, pflegt einen lokalen Cache, der die Informationen jeder benutzerdefinierten `PgStat_KindInfo` speichert.

Platzieren Sie das Erweiterungsmodul, das den benutzerdefinierten kumulativen Statistiktyp implementiert, in `shared_preload_libraries`, damit es früh während des PostgreSQL-Starts geladen wird.

Ein Beispiel, das beschreibt, wie benutzerdefinierte Statistiken registriert und verwendet werden, findet sich in `src/test/modules/injection_points`.

### 36.10.16. C++ für Erweiterbarkeit verwenden

Obwohl das PostgreSQL-Backend in C geschrieben ist, ist es möglich, Erweiterungen in C++ zu schreiben, wenn die folgenden Richtlinien eingehalten werden:

- Alle Funktionen, auf die das Backend zugreift, müssen dem Backend eine C-Schnittstelle präsentieren; diese C-Funktionen können dann C++-Funktionen aufrufen. Beispielsweise ist `extern C`-Linkage für Funktionen erforderlich, auf die das Backend zugreift. Das ist auch für alle Funktionen nötig, die als Zeiger zwischen Backend und C++-Code übergeben werden.
- Geben Sie Speicher mit der passenden Freigabemethode frei. Beispielsweise wird der meiste Backend-Speicher mit `palloc()` reserviert, verwenden Sie daher `pfree()`, um ihn freizugeben. Die Verwendung von C++ `delete` schlägt in solchen Fällen fehl.
- Verhindern Sie, dass Exceptions in den C-Code weitergereicht werden. Verwenden Sie dazu einen Catch-all-Block auf oberster Ebene aller `extern C`-Funktionen. Das ist auch dann nötig, wenn der C++-Code nicht ausdrücklich Exceptions wirft, weil Ereignisse wie Speichermangel trotzdem Exceptions auslösen können. Alle Exceptions müssen abgefangen und passende Fehler an die C-Schnittstelle zurückgegeben werden. Wenn möglich, kompilieren Sie C++ mit `-fno-exceptions`, um Exceptions vollständig zu beseitigen; in solchen Fällen müssen Sie Fehler in Ihrem C++-Code prüfen, zum Beispiel auf von `new()` zurückgegebenes `NULL`.
- Wenn Sie Backend-Funktionen aus C++-Code aufrufen, stellen Sie sicher, dass der C++-Call-Stack nur plain old data structures (POD) enthält. Das ist nötig, weil Backend-Fehler ein entferntes `longjmp()` erzeugen, das einen C++-Call-Stack mit Nicht-POD-Objekten nicht korrekt abwickelt.

Zusammengefasst ist es am besten, C++-Code hinter einer Wand aus `extern C`-Funktionen zu platzieren, die mit dem Backend kommunizieren, und das Durchsickern von Exceptions, Speicherverwaltungsdetails und Call-Stack-Effekten zu vermeiden.

## 36.11. Informationen zur Funktionsoptimierung

Standardmäßig ist eine Funktion nur eine „Black Box“, über deren Verhalten das Datenbanksystem sehr wenig weiß. Das bedeutet jedoch, dass Abfragen, die diese Funktion verwenden, möglicherweise deutlich weniger effizient ausgeführt werden, als sie könnten. Es ist möglich, zusätzliches Wissen bereitzustellen, das dem Planner hilft, Funktionsaufrufe zu optimieren.

Einige grundlegende Fakten können durch deklarative Annotationen bereitgestellt werden, die im Befehl `CREATE FUNCTION` angegeben werden. Am wichtigsten ist die Volatilitätskategorie der Funktion (`IMMUTABLE`, `STABLE` oder `VOLATILE`); beim Definieren einer Funktion sollte man immer sorgfältig darauf achten, diese korrekt anzugeben. Auch die Eigenschaft zur Parallel-Sicherheit (`PARALLEL UNSAFE`, `PARALLEL RESTRICTED` oder `PARALLEL SAFE`) muss angegeben werden, wenn Sie hoffen, die Funktion in parallelisierten Abfragen zu verwenden. Es kann außerdem nützlich sein, die geschätzten Ausführungskosten der Funktion und/oder die Anzahl der Zeilen anzugeben, die eine mengenrückgebende Funktion voraussichtlich zurückgibt. Die deklarative Angabe dieser beiden Fakten erlaubt jedoch nur einen konstanten Wert, was häufig unzureichend ist.

Es ist auch möglich, eine Planner-Support-Funktion an eine aus SQL aufrufbare Funktion, die sogenannte Zielfunktion, anzuhängen und dadurch Wissen über die Zielfunktion bereitzustellen, das zu komplex ist, um deklarativ dargestellt zu werden. Planner-Support-Funktionen müssen in C geschrieben werden, auch wenn ihre Zielfunktionen das nicht sein müssen; daher ist dies eine fortgeschrittene Funktionalität, die vergleichsweise wenige Anwender nutzen werden.

Eine Planner-Support-Funktion muss die SQL-Signatur haben:

```sql
supportfn(internal) returns internal
```

Sie wird an ihre Zielfunktion angehängt, indem beim Erstellen der Zielfunktion die Klausel `SUPPORT` angegeben wird.

Die Details der API für Planner-Support-Funktionen finden sich in der Datei `src/include/nodes/supportnodes.h` im PostgreSQL-Quellcode. Hier geben wir nur einen Überblick darüber, was Planner-Support-Funktionen tun können. Die Menge möglicher Anfragen an eine Support-Funktion ist erweiterbar, daher könnten in zukünftigen Versionen weitere Dinge möglich sein.

Einige Funktionsaufrufe können während der Planung auf Grundlage funktionsspezifischer Eigenschaften vereinfacht werden. Beispielsweise könnte `int4mul(n, 1)` zu schlicht `n` vereinfacht werden. Diese Art von Transformation kann von einer Planner-Support-Funktion durchgeführt werden, indem sie den Anfragetyp `SupportRequestSimplify` implementiert. Die Support-Funktion wird für jede Instanz ihrer Zielfunktion aufgerufen, die im Parse Tree der Abfrage gefunden wird. Wenn sie feststellt, dass der betreffende Aufruf in eine andere Form vereinfacht werden kann, kann sie einen Parse Tree bauen und zurückgeben, der diesen Ausdruck darstellt. Das funktioniert automatisch auch für Operatoren, die auf der Funktion beruhen; im gerade genannten Beispiel würde auch `n * 1` zu `n` vereinfacht. Beachten Sie jedoch, dass dies nur ein Beispiel ist; diese konkrete Optimierung wird von Standard-PostgreSQL tatsächlich nicht durchgeführt. Es wird nicht garantiert, dass PostgreSQL die Zielfunktion niemals in Fällen aufruft, die die Support-Funktion vereinfachen könnte. Stellen Sie strenge Äquivalenz zwischen dem vereinfachten Ausdruck und einer tatsächlichen Ausführung der Zielfunktion sicher.

Für Zielfunktionen, die `boolean` zurückgeben, ist es oft nützlich, den Anteil der Zeilen zu schätzen, der durch eine `WHERE`-Klausel mit dieser Funktion ausgewählt wird. Das kann durch eine Support-Funktion geschehen, die den Anfragetyp `SupportRequestSelectivity` implementiert.

Wenn die Laufzeit der Zielfunktion stark von ihren Eingaben abhängt, kann es nützlich sein, eine nicht konstante Kostenschätzung für sie bereitzustellen. Das kann durch eine Support-Funktion geschehen, die den Anfragetyp `SupportRequestCost` implementiert.

Für Zielfunktionen, die Mengen zurückgeben, ist es oft nützlich, eine nicht konstante Schätzung für die Anzahl der zurückgegebenen Zeilen bereitzustellen. Das kann durch eine Support-Funktion geschehen, die den Anfragetyp `SupportRequestRows` implementiert.

Für Zielfunktionen, die `boolean` zurückgeben, kann es möglich sein, einen in `WHERE` erscheinenden Funktionsaufruf in eine indexierbare Operatorklausel oder mehrere solche Klauseln umzuwandeln. Die umgewandelten Klauseln können exakt äquivalent zur Bedingung der Funktion sein, oder sie können etwas schwächer sein, das heißt, sie können einige Werte akzeptieren, die die Funktionsbedingung nicht akzeptiert. Im letzteren Fall wird die Indexbedingung als lossy bezeichnet; sie kann weiterhin zum Scannen eines Index verwendet werden, aber der Funktionsaufruf muss für jede vom Index zurückgegebene Zeile ausgeführt werden, um festzustellen, ob sie die `WHERE`-Bedingung tatsächlich erfüllt. Um solche Bedingungen zu erzeugen, muss die Support-Funktion den Anfragetyp `SupportRequestIndexCondition` implementieren.

## 36.12. Benutzerdefinierte Aggregate

Aggregatfunktionen in PostgreSQL werden über Zustandswerte und Zustandsübergangsfunktionen definiert. Das heißt, ein Aggregat arbeitet mit einem Zustandswert, der aktualisiert wird, während jede aufeinanderfolgende Eingabezeile verarbeitet wird. Um eine neue Aggregatfunktion zu definieren, wählt man einen Datentyp für den Zustandswert, einen Anfangswert für den Zustand und eine Zustandsübergangsfunktion aus. Die Zustandsübergangsfunktion nimmt den vorherigen Zustandswert und den oder die Eingabewerte des Aggregats für die aktuelle Zeile entgegen und gibt einen neuen Zustandswert zurück. Außerdem kann eine finale Funktion angegeben werden, falls das gewünschte Ergebnis des Aggregats von den Daten abweicht, die im laufenden Zustandswert gehalten werden müssen. Die finale Funktion nimmt den abschließenden Zustandswert entgegen und gibt das gewünschte Aggregatergebnis zurück. Im Prinzip sind Übergangs- und finale Funktionen einfach gewöhnliche Funktionen, die auch außerhalb des Aggregatkontexts verwendet werden könnten. In der Praxis ist es aus Performance-Gründen oft hilfreich, spezialisierte Übergangsfunktionen zu erstellen, die nur funktionieren, wenn sie als Teil eines Aggregats aufgerufen werden.

Zusätzlich zu den Argument- und Ergebnisdatentypen, die ein Benutzer des Aggregats sieht, gibt es daher einen internen Datentyp für den Zustandswert, der sich sowohl von den Argumenttypen als auch vom Ergebnistyp unterscheiden kann.

Wenn wir ein Aggregat definieren, das keine finale Funktion verwendet, haben wir ein Aggregat, das eine laufende Funktion der Spaltenwerte aus jeder Zeile berechnet. `sum` ist ein Beispiel für diese Art von Aggregat. `sum` beginnt bei null und addiert immer den Wert der aktuellen Zeile zu seiner laufenden Summe. Wenn wir zum Beispiel ein `sum`-Aggregat erstellen möchten, das mit einem Datentyp für komplexe Zahlen arbeitet, benötigen wir nur die Additionsfunktion für diesen Datentyp. Die Aggregatdefinition wäre:

```sql
CREATE AGGREGATE sum (complex)
(
    sfunc = complex_add,
    stype = complex,
    initcond = '(0,0)'
);
```

Diese könnten wir so verwenden:

```sql
SELECT sum(a) FROM test_complex;
```

```text
   sum
-----------
 (34,53.9)
```

Beachten Sie, dass wir uns hier auf Funktionsüberladung verlassen: Es gibt mehr als ein Aggregat namens `sum`, aber PostgreSQL kann herausfinden, welche Art von `sum` auf eine Spalte vom Typ `complex` anzuwenden ist.

Die obige Definition von `sum` gibt null, also den Anfangszustandswert, zurück, wenn es keine nichtnulligen Eingabewerte gibt. Vielleicht möchten wir in diesem Fall stattdessen null zurückgeben; der SQL-Standard erwartet, dass sich `sum` so verhält. Das können wir einfach erreichen, indem wir die Phrase `initcond` weglassen, sodass der Anfangszustandswert null ist. Normalerweise würde das bedeuten, dass `sfunc` auf eine nullige Zustandswert-Eingabe prüfen müsste. Für `sum` und einige andere einfache Aggregate wie `max` und `min` reicht es jedoch aus, den ersten nichtnulligen Eingabewert in die Zustandsvariable einzufügen und dann ab dem zweiten nichtnulligen Eingabewert die Übergangsfunktion anzuwenden. PostgreSQL erledigt das automatisch, wenn der Anfangszustandswert null ist und die Übergangsfunktion als „strict“ markiert ist, also nicht für Null-Eingaben aufgerufen werden soll.

Ein weiteres Standardverhalten einer „strikten“ Übergangsfunktion ist, dass der vorherige Zustandswert unverändert beibehalten wird, wenn ein Null-Eingabewert auftritt. Nullwerte werden also ignoriert. Wenn Sie für Null-Eingaben ein anderes Verhalten benötigen, deklarieren Sie Ihre Übergangsfunktion nicht als strict; schreiben Sie sie stattdessen so, dass sie auf Null-Eingaben prüft und das jeweils Nötige tut.

`avg` (average) ist ein komplexeres Beispiel für ein Aggregat. Es benötigt zwei Bestandteile des laufenden Zustands: die Summe der Eingaben und die Anzahl der Eingaben. Das Endergebnis wird durch Division dieser Größen ermittelt. Average wird typischerweise mit einem Array als Zustandswert implementiert. Die eingebaute Implementierung von `avg(float8)` sieht zum Beispiel so aus:

```sql
CREATE AGGREGATE avg (float8)
(
    sfunc = float8_accum,
    stype = float8[],
    finalfunc = float8_avg,
    initcond = '{0,0,0}'
);
```

> Hinweis: `float8_accum` benötigt ein dreielementiges Array, nicht nur zwei Elemente, weil es neben Summe und Anzahl der Eingaben auch die Summe der Quadrate akkumuliert. Dadurch kann es außer für `avg` auch für einige andere Aggregate verwendet werden.

Aggregatfunktionsaufrufe in SQL erlauben die Optionen `DISTINCT` und `ORDER BY`, die steuern, welche Zeilen in welcher Reihenfolge an die Übergangsfunktion des Aggregats übergeben werden. Diese Optionen werden hinter den Kulissen implementiert und betreffen die Support-Funktionen des Aggregats nicht.

Weitere Einzelheiten finden Sie beim Befehl `CREATE AGGREGATE`.

### 36.12.1. Moving-Aggregate-Modus

Aggregatfunktionen können optional den Moving-Aggregate-Modus unterstützen, der eine deutlich schnellere Ausführung von Aggregatfunktionen innerhalb von Windows mit beweglichen Frame-Startpunkten ermöglicht. Siehe [Abschnitt 3.5](03_Fortgeschrittene_Funktionen.md#35-windowfunktionen) und [Abschnitt 4.2.8](04_SQL_Syntax.md#428-aufrufe-von-windowfunktionen) für Informationen zur Verwendung von Aggregatfunktionen als Window-Funktionen. Die Grundidee ist, dass das Aggregat zusätzlich zu einer normalen „vorwärts gerichteten“ Übergangsfunktion eine inverse Übergangsfunktion bereitstellt, mit der Zeilen aus dem laufenden Zustandswert des Aggregats entfernt werden können, wenn sie den Window-Frame verlassen. Ein `sum`-Aggregat, das Addition als vorwärts gerichtete Übergangsfunktion verwendet, würde zum Beispiel Subtraktion als inverse Übergangsfunktion verwenden. Ohne inverse Übergangsfunktion muss der Window-Funktionsmechanismus das Aggregat jedes Mal von Grund auf neu berechnen, wenn sich der Frame-Startpunkt bewegt; daraus ergibt sich eine Laufzeit proportional zur Anzahl der Eingabezeilen multipliziert mit der durchschnittlichen Frame-Länge. Mit einer inversen Übergangsfunktion ist die Laufzeit nur proportional zur Anzahl der Eingabezeilen.

Der inversen Übergangsfunktion werden der aktuelle Zustandswert und der oder die Aggregat-Eingabewerte für die früheste Zeile übergeben, die im aktuellen Zustand enthalten ist. Sie muss rekonstruieren, wie der Zustandswert ausgesehen hätte, wenn die angegebene Eingabezeile nie aggregiert worden wäre, sondern nur die darauf folgenden Zeilen. Das erfordert manchmal, dass die vorwärts gerichtete Übergangsfunktion mehr Zustand hält, als für den einfachen Aggregationsmodus nötig ist. Deshalb verwendet der Moving-Aggregate-Modus eine vollständig separate Implementierung gegenüber dem einfachen Modus: Er hat seinen eigenen Zustandsdatentyp, seine eigene vorwärts gerichtete Übergangsfunktion und, falls nötig, seine eigene finale Funktion. Diese können dieselben sein wie Datentyp und Funktionen des einfachen Modus, wenn kein zusätzlicher Zustand benötigt wird.

Als Beispiel könnten wir das oben angegebene `sum`-Aggregat so erweitern, dass es den Moving-Aggregate-Modus unterstützt:

```sql
CREATE AGGREGATE sum (complex)
(
    sfunc = complex_add,
    stype = complex,
    initcond = '(0,0)',
    msfunc = complex_add,
    minvfunc = complex_sub,
    mstype = complex,
    minitcond = '(0,0)'
);
```

Die Parameter, deren Namen mit `m` beginnen, definieren die Moving-Aggregate-Implementierung. Mit Ausnahme der inversen Übergangsfunktion `minvfunc` entsprechen sie den Parametern des einfachen Aggregats ohne `m`.

Die vorwärts gerichtete Übergangsfunktion für den Moving-Aggregate-Modus darf nicht null als neuen Zustandswert zurückgeben. Wenn die inverse Übergangsfunktion null zurückgibt, wird dies als Hinweis verstanden, dass die inverse Funktion die Zustandsberechnung für diese bestimmte Eingabe nicht rückgängig machen kann; daher wird die Aggregatberechnung für die aktuelle Frame-Startposition von Grund auf neu ausgeführt. Diese Konvention erlaubt es, den Moving-Aggregate-Modus in Situationen zu verwenden, in denen es einige seltene Fälle gibt, die sich praktisch nicht aus dem laufenden Zustandswert herausrechnen lassen. Die inverse Übergangsfunktion kann in diesen Fällen „ausweichen“ und dennoch einen Vorteil bringen, solange sie für die meisten Fälle funktioniert. Ein Aggregat, das mit Gleitkommazahlen arbeitet, könnte sich zum Beispiel dafür entscheiden auszuweichen, wenn eine NaN-Eingabe (not a number) aus dem laufenden Zustandswert entfernt werden muss.

Beim Schreiben von Support-Funktionen für Moving Aggregates ist es wichtig sicherzustellen, dass die inverse Übergangsfunktion den korrekten Zustandswert exakt rekonstruieren kann. Andernfalls können benutzerseitig sichtbare Unterschiede in den Ergebnissen auftreten, je nachdem, ob der Moving-Aggregate-Modus verwendet wird. Ein Beispiel für ein Aggregat, bei dem das Hinzufügen einer inversen Übergangsfunktion zunächst einfach erscheint, diese Anforderung aber nicht erfüllt werden kann, ist `sum` über `float4`- oder `float8`-Eingaben. Eine naive Deklaration von `sum(float8)` könnte lauten:

```sql
CREATE AGGREGATE unsafe_sum (float8)
(
    stype = float8,
    sfunc = float8pl,
    mstype = float8,
    msfunc = float8pl,
    minvfunc = float8mi
);
```

Dieses Aggregat kann jedoch drastisch andere Ergebnisse liefern, als es ohne inverse Übergangsfunktion liefern würde. Betrachten Sie zum Beispiel:

```sql
SELECT
  unsafe_sum(x) OVER (ORDER BY n ROWS BETWEEN CURRENT ROW AND 1 FOLLOWING)
FROM (VALUES (1, 1.0e20::float8),
             (2, 1.0::float8)) AS v (n,x);
```

Diese Abfrage gibt als zweites Ergebnis `0` zurück, statt des erwarteten Ergebnisses `1`. Die Ursache ist die begrenzte Genauigkeit von Gleitkommawerten: Das Addieren von `1` zu `1e20` ergibt wieder `1e20`, und das Subtrahieren von `1e20` davon ergibt daher `0`, nicht `1`. Beachten Sie, dass dies eine allgemeine Einschränkung der Gleitkommaarithmetik ist, keine Einschränkung von PostgreSQL.

### 36.12.2. Polymorphe und variadische Aggregate

Aggregatfunktionen können polymorphe Zustandsübergangsfunktionen oder finale Funktionen verwenden, sodass dieselben Funktionen zur Implementierung mehrerer Aggregate genutzt werden können. Eine Erklärung polymorpher Funktionen finden Sie in [Abschnitt 36.2.5](#3625-polymorphe-typen). Noch einen Schritt weitergehend kann die Aggregatfunktion selbst mit polymorphen Eingabetypen und einem polymorphen Zustandstyp angegeben werden, sodass eine einzige Aggregatdefinition für mehrere Eingabedatentypen dienen kann. Hier ist ein Beispiel für ein polymorphes Aggregat:

```sql
CREATE AGGREGATE array_accum (anycompatible)
(
    sfunc = array_append,
    stype = anycompatiblearray,
    initcond = '{}'
);
```

Hier ist der tatsächliche Zustandstyp für jeden konkreten Aggregataufruf der Array-Typ, dessen Elemente den tatsächlichen Eingabetyp haben. Das Verhalten des Aggregats besteht darin, alle Eingaben zu einem Array dieses Typs zu verketten. Beachten Sie, dass das eingebaute Aggregat `array_agg` ähnliche Funktionalität bereitstellt, mit besserer Performance, als diese Definition hätte.

Hier ist die Ausgabe mit zwei verschiedenen tatsächlichen Datentypen als Argumenten:

```sql
SELECT attrelid::regclass, array_accum(attname)
    FROM pg_attribute
    WHERE attnum > 0 AND attrelid = 'pg_tablespace'::regclass
    GROUP BY attrelid;
```

```text
   attrelid    |              array_accum
---------------+---------------------------------------
 pg_tablespace | {spcname,spcowner,spcacl,spcoptions}
(1 row)
```

```sql
SELECT attrelid::regclass, array_accum(atttypid::regtype)
    FROM pg_attribute
    WHERE attnum > 0 AND attrelid = 'pg_tablespace'::regclass
    GROUP BY attrelid;
```

```text
   attrelid    |        array_accum
---------------+---------------------------
 pg_tablespace | {name,oid,aclitem[],text[]}
(1 row)
```

Normalerweise hat eine Aggregatfunktion mit polymorphem Ergebnistyp einen polymorphen Zustandstyp, wie im obigen Beispiel. Das ist notwendig, weil die finale Funktion sonst nicht sinnvoll deklariert werden kann: Sie müsste einen polymorphen Ergebnistyp haben, aber keinen polymorphen Argumenttyp, was `CREATE FUNCTION` mit der Begründung zurückweist, dass der Ergebnistyp nicht aus einem Aufruf abgeleitet werden kann. Manchmal ist es jedoch unpraktisch, einen polymorphen Zustandstyp zu verwenden. Der häufigste Fall ist, dass die Aggregat-Support-Funktionen in C geschrieben werden sollen und der Zustandstyp als `internal` deklariert werden sollte, weil es dafür kein SQL-seitiges Äquivalent gibt. Für diesen Fall ist es möglich, die finale Funktion so zu deklarieren, dass sie zusätzliche „Dummy“-Argumente annimmt, die den Eingabeargumenten des Aggregats entsprechen. Solche Dummy-Argumente werden immer als Nullwerte übergeben, da beim Aufruf der finalen Funktion kein konkreter Wert verfügbar ist. Ihr einziger Zweck ist, den Ergebnistyp einer polymorphen finalen Funktion mit den Eingabetypen des Aggregats verbinden zu können. Zum Beispiel ist die Definition des eingebauten Aggregats `array_agg` äquivalent zu:

```sql
CREATE FUNCTION array_agg_transfn(internal, anynonarray)
  RETURNS internal ...;

CREATE FUNCTION array_agg_finalfn(internal, anynonarray)
  RETURNS anyarray ...;

CREATE AGGREGATE array_agg (anynonarray)
(
    sfunc = array_agg_transfn,
    stype = internal,
    finalfunc = array_agg_finalfn,
    finalfunc_extra
);
```

Hier gibt die Option `finalfunc_extra` an, dass die finale Funktion zusätzlich zum Zustandswert weitere Dummy-Argumente erhält, die den Eingabeargumenten des Aggregats entsprechen. Das zusätzliche Argument `anynonarray` erlaubt eine gültige Deklaration von `array_agg_finalfn`.

Eine Aggregatfunktion kann so eingerichtet werden, dass sie eine veränderliche Anzahl von Argumenten akzeptiert, indem ihr letztes Argument als `VARIADIC`-Array deklariert wird, ganz ähnlich wie bei regulären Funktionen; siehe [Abschnitt 36.5.6](#3656-sqlfunktionen-mit-variabler-argumentanzahl). Die Übergangsfunktionen des Aggregats müssen denselben Array-Typ als letztes Argument haben. Die Übergangsfunktionen würden typischerweise ebenfalls als `VARIADIC` markiert, das ist aber nicht zwingend erforderlich.

> Hinweis: Variadische Aggregate können in Verbindung mit der Option `ORDER BY` leicht falsch verwendet werden (siehe [Abschnitt 4.2.7](04_SQL_Syntax.md#427-aggregatausdrücke)), da der Parser bei einer solchen Kombination nicht erkennen kann, ob die falsche Anzahl tatsächlicher Argumente angegeben wurde. Denken Sie daran, dass alles rechts von `ORDER BY` ein Sortierschlüssel ist, kein Argument des Aggregats. Zum Beispiel sieht der Parser in
>
> ```sql
> SELECT myaggregate(a ORDER BY a, b, c) FROM ...
> ```
>
> ein einzelnes Aggregatfunktionsargument und drei Sortierschlüssel. Der Benutzer könnte jedoch Folgendes gemeint haben:
>
> ```sql
> SELECT myaggregate(a, b, c ORDER BY a) FROM ...
> ```
>
> Wenn `myaggregate` variadisch ist, könnten beide Aufrufe vollkommen gültig sein.
>
> Aus demselben Grund ist es ratsam, zweimal nachzudenken, bevor man Aggregatfunktionen mit denselben Namen und unterschiedlicher Anzahl regulärer Argumente erstellt.

### 36.12.3. Ordered-Set-Aggregate

Die Aggregate, die wir bisher beschrieben haben, sind „normale“ Aggregate. PostgreSQL unterstützt außerdem Ordered-Set-Aggregate, die sich in zwei wichtigen Punkten von normalen Aggregaten unterscheiden. Erstens kann ein Ordered-Set-Aggregat zusätzlich zu gewöhnlichen aggregierten Argumenten, die einmal pro Eingabezeile ausgewertet werden, „direkte“ Argumente haben, die nur einmal pro Aggregationsoperation ausgewertet werden. Zweitens gibt die Syntax für die gewöhnlichen aggregierten Argumente explizit eine Sortierreihenfolge für sie an. Ein Ordered-Set-Aggregat wird üblicherweise verwendet, um eine Berechnung zu implementieren, die von einer bestimmten Zeilenreihenfolge abhängt, etwa Rang oder Perzentil, sodass die Sortierreihenfolge ein erforderlicher Bestandteil jedes Aufrufs ist. Zum Beispiel ist die eingebaute Definition von `percentile_disc` äquivalent zu:

```sql
CREATE FUNCTION ordered_set_transition(internal, anyelement)
  RETURNS internal ...;
CREATE FUNCTION percentile_disc_final(internal, float8, anyelement)
  RETURNS anyelement ...;

CREATE AGGREGATE percentile_disc (float8 ORDER BY anyelement)
(
    sfunc = ordered_set_transition,
    stype = internal,
    finalfunc = percentile_disc_final,
    finalfunc_extra
);
```

Dieses Aggregat nimmt ein direktes Argument vom Typ `float8`, den Perzentilanteil, und eine aggregierte Eingabe, die von jedem sortierbaren Datentyp sein kann. Es könnte so verwendet werden, um ein medianes Haushaltseinkommen zu ermitteln:

```sql
SELECT percentile_disc(0.5) WITHIN GROUP (ORDER BY income) FROM households;
```

```text
 percentile_disc
-----------------
```

Hier ist `0.5` ein direktes Argument; es ergäbe keinen Sinn, wenn der Perzentilanteil ein Wert wäre, der über die Zeilen hinweg variiert.

Anders als bei normalen Aggregaten erfolgt die Sortierung der Eingabezeilen für ein Ordered-Set-Aggregat nicht hinter den Kulissen, sondern liegt in der Verantwortung der Support-Funktionen des Aggregats. Der typische Implementierungsansatz besteht darin, im Zustandswert des Aggregats eine Referenz auf ein `tuplesort`-Objekt zu halten, die eingehenden Zeilen in dieses Objekt einzuspeisen und dann in der finalen Funktion die Sortierung abzuschließen und die Daten auszulesen. Dieses Design erlaubt der finalen Funktion, besondere Operationen durchzuführen, etwa zusätzliche „hypothetische“ Zeilen in die zu sortierenden Daten einzufügen. Während normale Aggregate oft mit in PL/pgSQL oder einer anderen PL-Sprache geschriebenen Support-Funktionen implementiert werden können, müssen Ordered-Set-Aggregate im Allgemeinen in C geschrieben werden, da ihre Zustandswerte nicht als SQL-Datentyp definierbar sind. Beachten Sie im obigen Beispiel, dass der Zustandswert als Typ `internal` deklariert ist; das ist typisch. Außerdem ist es, weil die finale Funktion die Sortierung durchführt, nicht möglich, später weitere Eingabezeilen hinzuzufügen, indem die Übergangsfunktion erneut ausgeführt wird. Das bedeutet, dass die finale Funktion nicht `READ_ONLY` ist; sie muss in `CREATE AGGREGATE` als `READ_WRITE` deklariert werden, oder als `SHAREABLE`, wenn zusätzliche Aufrufe der finalen Funktion den bereits sortierten Zustand nutzen können.

Die Zustandsübergangsfunktion für ein Ordered-Set-Aggregat erhält den aktuellen Zustandswert plus die aggregierten Eingabewerte für jede Zeile und gibt den aktualisierten Zustandswert zurück. Das ist dieselbe Definition wie bei normalen Aggregaten, aber beachten Sie, dass die direkten Argumente, falls vorhanden, nicht bereitgestellt werden. Die finale Funktion erhält den letzten Zustandswert, die Werte der direkten Argumente, falls vorhanden, und, wenn `finalfunc_extra` angegeben ist, Nullwerte, die den aggregierten Eingaben entsprechen. Wie bei normalen Aggregaten ist `finalfunc_extra` nur dann wirklich nützlich, wenn das Aggregat polymorph ist; dann werden die zusätzlichen Dummy-Argumente benötigt, um den Ergebnistyp der finalen Funktion mit den Eingabetypen des Aggregats zu verbinden.

Derzeit können Ordered-Set-Aggregate nicht als Window-Funktionen verwendet werden, und deshalb müssen sie den Moving-Aggregate-Modus nicht unterstützen.

### 36.12.4. Partielle Aggregation

Optional kann eine Aggregatfunktion partielle Aggregation unterstützen. Die Idee der partiellen Aggregation besteht darin, die Zustandsübergangsfunktion des Aggregats unabhängig über verschiedene Teilmengen der Eingabedaten laufen zu lassen und anschließend die aus diesen Teilmengen resultierenden Zustandswerte zu kombinieren, um denselben Zustandswert zu erzeugen, der sich ergeben hätte, wenn alle Eingaben in einem einzigen Vorgang gescannt worden wären. Dieser Modus kann für parallele Aggregation verwendet werden, indem verschiedene Worker-Prozesse verschiedene Teile einer Tabelle scannen. Jeder Worker erzeugt einen partiellen Zustandswert, und am Ende werden diese Zustandswerte kombiniert, um einen finalen Zustandswert zu erzeugen. In Zukunft könnte dieser Modus auch für Zwecke wie das Kombinieren von Aggregationen über lokale und entfernte Tabellen verwendet werden; das ist jedoch noch nicht implementiert.

Um partielle Aggregation zu unterstützen, muss die Aggregatdefinition eine Kombinationsfunktion bereitstellen, die zwei Werte des Zustandstyps des Aggregats annimmt, die die Ergebnisse der Aggregation über zwei Teilmengen der Eingabezeilen darstellen, und einen neuen Wert des Zustandstyps erzeugt, der darstellt, wie der Zustand nach der Aggregation über die Kombination dieser Zeilenmengen gewesen wäre. Es ist nicht festgelegt, wie die relative Reihenfolge der Eingabezeilen aus den beiden Mengen gewesen wäre. Das bedeutet, dass es gewöhnlich unmöglich ist, eine nützliche Kombinationsfunktion für Aggregate zu definieren, die empfindlich auf die Reihenfolge der Eingabezeilen reagieren.

Als einfache Beispiele können `MAX`- und `MIN`-Aggregate dazu gebracht werden, partielle Aggregation zu unterstützen, indem als Kombinationsfunktion dieselbe Vergleichsfunktion „größerer von zwei Werten“ beziehungsweise „kleinerer von zwei Werten“ angegeben wird, die auch als Übergangsfunktion verwendet wird. `SUM`-Aggregate benötigen lediglich eine Additionsfunktion als Kombinationsfunktion. Auch das ist dieselbe wie ihre Übergangsfunktion, sofern der Zustandswert nicht breiter ist als der Eingabedatentyp.

Die Kombinationsfunktion wird ähnlich behandelt wie eine Übergangsfunktion, die zufällig als zweites Argument einen Wert des Zustandstyps annimmt, nicht des zugrunde liegenden Eingabetyps. Insbesondere sind die Regeln für den Umgang mit Nullwerten und strikten Funktionen ähnlich. Wenn die Aggregatdefinition ein nicht-nulliges `initcond` angibt, bedenken Sie außerdem, dass dieses nicht nur als Anfangszustand für jeden partiellen Aggregationslauf verwendet wird, sondern auch als Anfangszustand für die Kombinationsfunktion, die aufgerufen wird, um jedes partielle Ergebnis in diesen Zustand zu kombinieren.

Wenn der Zustandstyp des Aggregats als `internal` deklariert ist, liegt es in der Verantwortung der Kombinationsfunktion, dass ihr Ergebnis im richtigen Speicher-Kontext für Aggregat-Zustandswerte reserviert wird. Das bedeutet insbesondere, dass es bei einer ersten Eingabe von `NULL` ungültig ist, einfach die zweite Eingabe zurückzugeben, da dieser Wert im falschen Kontext liegt und keine ausreichende Lebensdauer hat.

Wenn der Zustandstyp des Aggregats als `internal` deklariert ist, ist es gewöhnlich außerdem sinnvoll, dass die Aggregatdefinition eine Serialisierungsfunktion und eine Deserialisierungsfunktion bereitstellt, mit denen ein solcher Zustandswert von einem Prozess in einen anderen kopiert werden kann. Ohne diese Funktionen kann parallele Aggregation nicht durchgeführt werden, und zukünftige Anwendungen wie lokale/entfernte Aggregation werden wahrscheinlich ebenfalls nicht funktionieren.

Eine Serialisierungsfunktion muss ein einzelnes Argument vom Typ `internal` annehmen und ein Ergebnis vom Typ `bytea` zurückgeben, das den Zustandswert als flachen Byte-Blob verpackt darstellt. Umgekehrt macht eine Deserialisierungsfunktion diese Umwandlung rückgängig. Sie muss zwei Argumente der Typen `bytea` und `internal` annehmen und ein Ergebnis vom Typ `internal` zurückgeben. Das zweite Argument wird nicht verwendet und ist immer null, wird aber aus Gründen der Typsicherheit benötigt. Das Ergebnis der Deserialisierungsfunktion sollte einfach im aktuellen Speicher-Kontext reserviert werden, da es im Gegensatz zum Ergebnis der Kombinationsfunktion nicht langlebig ist.

Erwähnenswert ist außerdem, dass ein Aggregat selbst als `PARALLEL SAFE` markiert sein muss, damit es parallel ausgeführt werden kann. Die Parallel-Safety-Markierungen seiner Support-Funktionen werden nicht herangezogen.

### 36.12.5. Support-Funktionen für Aggregate

Eine in C geschriebene Funktion kann erkennen, dass sie als Aggregat-Support-Funktion aufgerufen wird, indem sie `AggCheckCallContext` aufruft, zum Beispiel:

```c
if (AggCheckCallContext(fcinfo, NULL))
```

Ein Grund für diese Prüfung ist, dass die erste Eingabe, wenn die Prüfung wahr ist, ein temporärer Zustandswert sein muss und daher sicher direkt verändert werden kann, statt eine neue Kopie zu reservieren. Siehe `int8inc()` als Beispiel. Während Aggregatübergangsfunktionen den Übergangswert immer direkt verändern dürfen, wird bei finalen Aggregatfunktionen im Allgemeinen davon abgeraten; wenn sie es dennoch tun, muss dieses Verhalten beim Erstellen des Aggregats deklariert werden. Weitere Einzelheiten finden Sie bei `CREATE AGGREGATE`.

Das zweite Argument von `AggCheckCallContext` kann verwendet werden, um den Speicher-Kontext abzurufen, in dem Aggregat-Zustandswerte gehalten werden. Das ist nützlich für Übergangsfunktionen, die „expanded“ Objects (siehe [Abschnitt 36.13.1](#36131-toastüberlegungen)) als Zustandswerte verwenden möchten. Beim ersten Aufruf sollte die Übergangsfunktion ein Expanded Object zurückgeben, dessen Speicher-Kontext ein Kind des Aggregat-Zustandskontexts ist, und bei späteren Aufrufen weiterhin dasselbe Expanded Object zurückgeben. Siehe `array_append()` als Beispiel. `array_append()` ist nicht die Übergangsfunktion eines eingebauten Aggregats, ist aber so geschrieben, dass es effizient arbeitet, wenn es als Übergangsfunktion eines benutzerdefinierten Aggregats verwendet wird.

Eine weitere Support-Routine, die in C geschriebenen Aggregatfunktionen zur Verfügung steht, ist `AggGetAggref`, die den `Aggref`-Parse-Node zurückgibt, der den Aggregataufruf definiert. Das ist hauptsächlich für Ordered-Set-Aggregate nützlich, die die Unterstruktur des `Aggref`-Nodes untersuchen können, um herauszufinden, welche Sortierreihenfolge sie implementieren sollen. Beispiele finden sich in `orderedsetaggs.c` im PostgreSQL-Quellcode.

## 36.13. Benutzerdefinierte Typen

Wie in [Abschnitt 36.2](#362-das-postgresqltypsystem) beschrieben, kann PostgreSQL erweitert werden, um neue Datentypen zu unterstützen. Dieser Abschnitt beschreibt, wie neue Basistypen definiert werden, also Datentypen unterhalb der Ebene der SQL-Sprache. Das Erstellen eines neuen Basistyps erfordert die Implementierung von Funktionen, die auf dem Typ in einer Low-Level-Sprache arbeiten, gewöhnlich C.

Die Beispiele in diesem Abschnitt finden sich in `complex.sql` und `complex.c` im Verzeichnis `src/tutorial` der Quelldistribution. Hinweise zum Ausführen der Beispiele finden Sie in der Datei `README` in diesem Verzeichnis.

Ein benutzerdefinierter Typ muss immer Eingabe- und Ausgabefunktionen haben. Diese Funktionen bestimmen, wie der Typ in Zeichenketten erscheint, sowohl bei der Eingabe durch den Benutzer als auch bei der Ausgabe an den Benutzer, und wie der Typ im Speicher organisiert ist. Die Eingabefunktion nimmt eine nullterminierte Zeichenkette als Argument und gibt die interne Darstellung des Typs im Speicher zurück. Die Ausgabefunktion nimmt die interne Darstellung des Typs als Argument und gibt eine nullterminierte Zeichenkette zurück. Wenn wir mit dem Typ mehr tun wollen, als ihn nur zu speichern, müssen wir zusätzliche Funktionen bereitstellen, um die gewünschten Operationen für den Typ zu implementieren.

Angenommen, wir möchten einen Typ `complex` definieren, der komplexe Zahlen darstellt. Eine natürliche Möglichkeit, eine komplexe Zahl im Speicher darzustellen, wäre die folgende C-Struktur:

```c
typedef struct Complex {
    double      x;
    double      y;
} Complex;
```

Wir müssen daraus einen per Referenz übergebenen Typ machen, da er zu groß ist, um in einen einzelnen `Datum`-Wert zu passen.

Als externe Zeichenkettendarstellung des Typs wählen wir eine Zeichenkette der Form `(x,y)`.

Die Eingabe- und Ausgabefunktionen sind normalerweise nicht schwer zu schreiben, insbesondere die Ausgabefunktion. Denken Sie beim Definieren der externen Zeichenkettendarstellung des Typs aber daran, dass Sie letztlich einen vollständigen und robusten Parser für diese Darstellung als Eingabefunktion schreiben müssen. Zum Beispiel:

```c
PG_FUNCTION_INFO_V1(complex_in);

Datum
complex_in(PG_FUNCTION_ARGS)
{
    char       *str = PG_GETARG_CSTRING(0);
    double      x,
                y;
    Complex    *result;

    if (sscanf(str, " ( %lf , %lf )", &x, &y) != 2)
        ereport(ERROR,
                (errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
                 errmsg("invalid input syntax for type %s: \"%s\"",
                        "complex", str)));

    result = (Complex *) palloc(sizeof(Complex));
    result->x = x;
    result->y = y;
    PG_RETURN_POINTER(result);
}
```

Die Ausgabefunktion kann einfach so aussehen:

```c
PG_FUNCTION_INFO_V1(complex_out);

Datum
complex_out(PG_FUNCTION_ARGS)
{
    Complex    *complex = (Complex *) PG_GETARG_POINTER(0);
    char       *result;

    result = psprintf("(%g,%g)", complex->x, complex->y);
    PG_RETURN_CSTRING(result);
}
```

Sie sollten sorgfältig darauf achten, dass Eingabe- und Ausgabefunktion zueinander invers sind. Andernfalls bekommen Sie erhebliche Probleme, wenn Sie Ihre Daten in eine Datei ausgeben und anschließend wieder einlesen müssen. Das ist ein besonders häufiges Problem, wenn Gleitkommazahlen beteiligt sind.

Optional kann ein benutzerdefinierter Typ binäre Eingabe- und Ausgaberoutinen bereitstellen. Binäre Ein-/Ausgabe ist normalerweise schneller, aber weniger portabel als textuelle Ein-/Ausgabe. Wie bei textueller Ein-/Ausgabe liegt es bei Ihnen, exakt festzulegen, wie die externe binäre Darstellung aussieht. Die meisten eingebauten Datentypen versuchen, eine maschinenunabhängige binäre Darstellung bereitzustellen. Für `complex` stützen wir uns auf die binären Ein-/Ausgabe-Konverter für den Typ `float8`:

```c
PG_FUNCTION_INFO_V1(complex_recv);

Datum
complex_recv(PG_FUNCTION_ARGS)
{
    StringInfo buf = (StringInfo) PG_GETARG_POINTER(0);
    Complex    *result;

    result = (Complex *) palloc(sizeof(Complex));
    result->x = pq_getmsgfloat8(buf);
    result->y = pq_getmsgfloat8(buf);
    PG_RETURN_POINTER(result);
}

PG_FUNCTION_INFO_V1(complex_send);

Datum
complex_send(PG_FUNCTION_ARGS)
{
    Complex    *complex = (Complex *) PG_GETARG_POINTER(0);
    StringInfoData buf;

    pq_begintypsend(&buf);
    pq_sendfloat8(&buf, complex->x);
    pq_sendfloat8(&buf, complex->y);
    PG_RETURN_BYTEA_P(pq_endtypsend(&buf));
}
```

Sobald wir die Ein-/Ausgabefunktionen geschrieben und in eine Shared Library kompiliert haben, können wir den Typ `complex` in SQL definieren. Zuerst deklarieren wir ihn als Shell-Typ:

```sql
CREATE TYPE complex;
```

Dies dient als Platzhalter, der es uns erlaubt, den Typ zu referenzieren, während wir seine Ein-/Ausgabefunktionen definieren. Nun können wir die Ein-/Ausgabefunktionen definieren:

```sql
CREATE FUNCTION complex_in(cstring)
    RETURNS complex
    AS 'filename'
    LANGUAGE C IMMUTABLE STRICT;

CREATE FUNCTION complex_out(complex)
    RETURNS cstring
    AS 'filename'
    LANGUAGE C IMMUTABLE STRICT;

CREATE FUNCTION complex_recv(internal)
   RETURNS complex
   AS 'filename'
   LANGUAGE C IMMUTABLE STRICT;

CREATE FUNCTION complex_send(complex)
   RETURNS bytea
   AS 'filename'
   LANGUAGE C IMMUTABLE STRICT;
```

Schließlich können wir die vollständige Definition des Datentyps bereitstellen:

```sql
CREATE TYPE complex (
   internallength = 16,
   input = complex_in,
   output = complex_out,
   receive = complex_recv,
   send = complex_send,
   alignment = double
);
```

Wenn Sie einen neuen Basistyp definieren, stellt PostgreSQL automatisch Unterstützung für Arrays dieses Typs bereit. Der Array-Typ hat typischerweise denselben Namen wie der Basistyp, jedoch mit vorangestelltem Unterstrich (`_`).

Sobald der Datentyp existiert, können wir zusätzliche Funktionen deklarieren, um nützliche Operationen auf dem Datentyp bereitzustellen. Auf diesen Funktionen können dann Operatoren definiert werden, und falls nötig können Operatorklassen erstellt werden, um die Indizierung des Datentyps zu unterstützen. Diese zusätzlichen Schichten werden in den folgenden Abschnitten besprochen.

Wenn die interne Darstellung des Datentyps variable Länge hat, muss die interne Darstellung dem Standardlayout für Daten variabler Länge folgen: Die ersten vier Bytes müssen ein Feld `char[4]` sein, auf das niemals direkt zugegriffen wird und das üblicherweise `vl_len_` heißt. Sie müssen das Makro `SET_VARSIZE()` verwenden, um die Gesamtgröße des Datums, einschließlich des Längenfelds selbst, in diesem Feld zu speichern, und `VARSIZE()`, um sie abzurufen. Diese Makros existieren, weil das Längenfeld je nach Plattform kodiert sein kann.

Weitere Einzelheiten finden Sie in der Beschreibung des Befehls `CREATE TYPE`.

### 36.13.1. TOAST-Überlegungen

Wenn die Werte Ihres Datentyps in interner Form in ihrer Größe variieren, ist es normalerweise wünschenswert, den Datentyp TOAST-fähig zu machen (siehe [Abschnitt 66.2](66_Physische_Speicherung_der_Datenbank.md#662-toast)). Das sollten Sie auch dann tun, wenn die Werte immer zu klein sind, um komprimiert oder extern gespeichert zu werden, weil TOAST durch die Verringerung des Header-Overheads auch bei kleinen Daten Platz sparen kann.

Um TOAST-Speicherung zu unterstützen, müssen die C-Funktionen, die auf dem Datentyp arbeiten, stets darauf achten, ihnen übergebene toasted Werte mit `PG_DETOAST_DATUM` zu entpacken. Dieses Detail wird üblicherweise verborgen, indem typspezifische `GETARG_DATATYPE_P`-Makros definiert werden. Geben Sie anschließend beim Ausführen des Befehls `CREATE TYPE` die interne Länge als variabel an und wählen Sie eine passende Speicheroption außer `plain`.

Wenn Daten-Alignment unwichtig ist, entweder nur für eine bestimmte Funktion oder weil der Datentyp ohnehin Byte-Alignment angibt, kann ein Teil des Overheads von `PG_DETOAST_DATUM` vermieden werden. Sie können stattdessen `PG_DETOAST_DATUM_PACKED` verwenden, üblicherweise verborgen durch ein `GETARG_DATATYPE_PP`-Makro, und mit den Makros `VARSIZE_ANY_EXHDR` und `VARDATA_ANY` auf ein möglicherweise gepacktes Datum zugreifen. Auch hier sind die von diesen Makros zurückgegebenen Daten nicht ausgerichtet, selbst wenn die Datentypdefinition ein Alignment angibt. Wenn das Alignment wichtig ist, müssen Sie die reguläre Schnittstelle `PG_DETOAST_DATUM` verwenden.

> Hinweis: Älterer Code deklariert `vl_len_` häufig als Feld `int32` statt als `char[4]`. Das ist in Ordnung, solange die Strukturdefinition andere Felder hat, die mindestens `int32`-Alignment besitzen. Es ist jedoch gefährlich, eine solche Strukturdefinition bei der Arbeit mit einem möglicherweise nicht ausgerichteten Datum zu verwenden; der Compiler kann dies als Erlaubnis verstehen anzunehmen, dass das Datum tatsächlich ausgerichtet ist, was auf Architekturen, die striktes Alignment verlangen, zu Core Dumps führen kann.

Eine weitere Funktionalität, die durch TOAST-Unterstützung ermöglicht wird, ist die Möglichkeit, eine expandierte In-Memory-Datendarstellung zu haben, die bequemer zu bearbeiten ist als das auf der Platte gespeicherte Format. Das reguläre oder „flache“ `varlena`-Speicherformat ist letztlich nur ein Byte-Blob; es kann zum Beispiel keine Zeiger enthalten, da es an andere Speicherstellen kopiert werden kann. Bei komplexen Datentypen kann die Arbeit mit dem flachen Format recht teuer sein, daher stellt PostgreSQL eine Möglichkeit bereit, das flache Format in eine für Berechnungen besser geeignete Darstellung zu „expandieren“ und dieses Format dann im Speicher zwischen Funktionen des Datentyps weiterzureichen.

Um expandierte Speicherung zu verwenden, muss ein Datentyp ein expandiertes Format definieren, das den in `src/include/utils/expandeddatum.h` angegebenen Regeln folgt, und Funktionen bereitstellen, um einen flachen `varlena`-Wert in das expandierte Format zu „expandieren“ und das expandierte Format wieder in die reguläre `varlena`-Darstellung zu „flatten“. Stellen Sie dann sicher, dass alle C-Funktionen für den Datentyp beide Darstellungen akzeptieren können, möglicherweise indem sie eine Darstellung unmittelbar nach Erhalt in die andere umwandeln. Das erfordert nicht, alle vorhandenen Funktionen für den Datentyp auf einmal anzupassen, weil das Standardmakro `PG_DETOAST_DATUM` so definiert ist, dass es expandierte Eingaben in das reguläre flache Format umwandelt. Daher funktionieren vorhandene Funktionen, die mit dem flachen `varlena`-Format arbeiten, weiterhin mit expandierten Eingaben, wenn auch etwas ineffizienter; sie müssen erst umgestellt werden, wenn bessere Performance wichtig ist.

C-Funktionen, die mit einer expandierten Darstellung arbeiten können, fallen typischerweise in zwei Kategorien: solche, die nur das expandierte Format verarbeiten können, und solche, die sowohl expandierte als auch flache `varlena`-Eingaben verarbeiten können. Erstere sind leichter zu schreiben, können aber insgesamt weniger effizient sein, weil die Umwandlung einer flachen Eingabe in die expandierte Form für die Verwendung durch eine einzelne Funktion mehr kosten kann, als durch das Arbeiten mit dem expandierten Format eingespart wird. Wenn nur das expandierte Format verarbeitet werden muss, kann die Umwandlung flacher Eingaben in die expandierte Form in einem Makro zum Abrufen von Argumenten verborgen werden, sodass die Funktion nicht komplexer erscheint als eine, die mit traditioneller `varlena`-Eingabe arbeitet. Um beide Eingabetypen zu verarbeiten, schreiben Sie eine Funktion zum Abrufen von Argumenten, die externe, kurz-Header- und komprimierte `varlena`-Eingaben detoastet, expandierte Eingaben jedoch nicht. Eine solche Funktion kann so definiert werden, dass sie einen Zeiger auf eine Union aus dem flachen `varlena`-Format und dem expandierten Format zurückgibt. Aufrufer können mit dem Makro `VARATT_IS_EXPANDED_HEADER()` feststellen, welches Format sie erhalten haben.

Die TOAST-Infrastruktur erlaubt nicht nur, reguläre `varlena`-Werte von expandierten Werten zu unterscheiden, sondern unterscheidet außerdem „read-write“- und „read-only“-Zeiger auf expandierte Werte. C-Funktionen, die einen expandierten Wert nur untersuchen müssen oder ihn nur auf sichere und semantisch nicht sichtbare Weise ändern, müssen sich nicht darum kümmern, welchen Zeigertyp sie erhalten. C-Funktionen, die eine geänderte Version eines Eingabewerts erzeugen, dürfen einen expandierten Eingabewert in-place ändern, wenn sie einen read-write-Zeiger erhalten, dürfen die Eingabe aber nicht ändern, wenn sie einen read-only-Zeiger erhalten; in diesem Fall müssen sie den Wert zuerst kopieren und einen neuen Wert erzeugen, der geändert werden kann. Eine C-Funktion, die einen neuen expandierten Wert konstruiert hat, sollte immer einen read-write-Zeiger darauf zurückgeben. Außerdem sollte eine C-Funktion, die einen read-write-expandierten Wert in-place ändert, darauf achten, den Wert in einem konsistenten Zustand zu hinterlassen, falls sie teilweise fehlschlägt.

Beispiele für die Arbeit mit expandierten Werten finden Sie in der Standard-Array-Infrastruktur, insbesondere in `src/backend/utils/adt/array_expanded.c`.

## 36.14. Benutzerdefinierte Operatoren

Jeder Operator ist „syntactic sugar“ für einen Aufruf einer zugrunde liegenden Funktion, die die eigentliche Arbeit erledigt. Sie müssen daher zuerst die zugrunde liegende Funktion erstellen, bevor Sie den Operator erstellen können. Ein Operator ist jedoch nicht nur syntactic sugar, weil er zusätzliche Informationen mit sich trägt, die dem Query Planner helfen, Abfragen zu optimieren, die den Operator verwenden. Der nächste Abschnitt erklärt diese zusätzlichen Informationen.

PostgreSQL unterstützt Präfix- und Infixoperatoren. Operatoren können überladen werden; derselbe Operatorname kann also für verschiedene Operatoren verwendet werden, die unterschiedliche Anzahlen und Typen von Operanden haben. Wenn eine Abfrage ausgeführt wird, bestimmt das System anhand der Anzahl und Typen der bereitgestellten Operanden, welcher Operator aufzurufen ist.

Hier ist ein Beispiel für das Erstellen eines Operators zum Addieren zweier komplexer Zahlen. Wir nehmen an, dass wir die Definition des Typs `complex` bereits erstellt haben (siehe [Abschnitt 36.13](#3613-benutzerdefinierte-typen)). Zuerst benötigen wir eine Funktion, die die Arbeit erledigt, dann können wir den Operator definieren:

```sql
CREATE FUNCTION complex_add(complex, complex)
    RETURNS complex
    AS 'filename', 'complex_add'
    LANGUAGE C IMMUTABLE STRICT;

CREATE OPERATOR + (
    leftarg = complex,
    rightarg = complex,
    function = complex_add,
    commutator = +
);
```

Nun könnten wir eine Abfrage wie diese ausführen:

```sql
SELECT (a + b) AS c FROM test_complex;
```

```text
-----------------
 (5.2,6.05)
 (133.42,144.95)
```

Wir haben hier gezeigt, wie ein binärer Operator erstellt wird. Um einen Präfixoperator zu erstellen, lassen Sie einfach `leftarg` weg. Die Klausel `function` und die Argumentklauseln sind die einzigen Pflichtangaben in `CREATE OPERATOR`. Die im Beispiel gezeigte Klausel `commutator` ist ein optionaler Hinweis für den Query Optimizer. Weitere Einzelheiten zu Kommutator und anderen Optimizer-Hinweisen erscheinen im nächsten Abschnitt.

## 36.15. Informationen zur Operatoroptimierung

Eine PostgreSQL-Operatordefinition kann mehrere optionale Klauseln enthalten, die dem System nützliche Informationen darüber geben, wie sich der Operator verhält. Diese Klauseln sollten immer dann angegeben werden, wenn es angemessen ist, weil sie bei der Ausführung von Abfragen, die den Operator verwenden, erhebliche Beschleunigungen ermöglichen können. Wenn Sie sie angeben, müssen Sie aber sicher sein, dass sie korrekt sind. Falsche Verwendung einer Optimierungsklausel kann zu langsamen Abfragen, subtil falscher Ausgabe oder anderen unschönen Folgen führen. Sie können eine Optimierungsklausel jederzeit weglassen, wenn Sie sich nicht sicher sind; die einzige Folge ist, dass Abfragen möglicherweise langsamer laufen als nötig.

In zukünftigen Versionen von PostgreSQL könnten weitere Optimierungsklauseln hinzukommen. Die hier beschriebenen sind alle, die Release 18.1 versteht.

Es ist außerdem möglich, an die Funktion, die einem Operator zugrunde liegt, eine Planner-Support-Funktion anzuhängen. Das bietet eine weitere Möglichkeit, dem System Informationen über das Verhalten des Operators mitzuteilen. Weitere Informationen finden Sie in [Abschnitt 36.11](#3611-informationen-zur-funktionsoptimierung).

### 36.15.1. COMMUTATOR

Die Klausel `COMMUTATOR`, falls angegeben, benennt einen Operator, der der Kommutator des zu definierenden Operators ist. Wir sagen, Operator A ist der Kommutator von Operator B, wenn `(x A y)` für alle möglichen Eingabewerte `x` und `y` gleich `(y B x)` ist. Beachten Sie, dass B dann auch der Kommutator von A ist. Zum Beispiel sind die Operatoren `<` und `>` für einen bestimmten Datentyp normalerweise gegenseitige Kommutatoren, und der Operator `+` ist üblicherweise mit sich selbst kommutativ. Der Operator `-` ist dagegen normalerweise mit nichts kommutativ.

Der linke Operandentyp eines kommutierbaren Operators ist derselbe wie der rechte Operandentyp seines Kommutators und umgekehrt. Daher ist der Name des Kommutatoroperators alles, was PostgreSQL zum Nachschlagen des Kommutators erhalten muss, und das ist alles, was in der Klausel `COMMUTATOR` angegeben werden muss.

Es ist entscheidend, Kommutatorinformationen für Operatoren bereitzustellen, die in Indizes und Join-Klauseln verwendet werden, weil der Query Optimizer dadurch eine solche Klausel in die Formen „umdrehen“ kann, die für verschiedene Plantypen benötigt werden. Betrachten Sie zum Beispiel eine Abfrage mit einer `WHERE`-Klausel wie `tab1.x = tab2.y`, wobei `tab1.x` und `tab2.y` einen benutzerdefinierten Typ haben und angenommen `tab2.y` ist indiziert. Der Optimizer kann keinen Index Scan erzeugen, wenn er nicht bestimmen kann, wie die Klausel zu `tab2.y = tab1.x` umgedreht werden kann, weil die Index-Scan-Mechanik erwartet, die indizierte Spalte links vom übergebenen Operator zu sehen. PostgreSQL nimmt nicht einfach an, dass diese Transformation gültig ist; der Ersteller des Operators `=` muss angeben, dass sie gültig ist, indem er den Operator mit Kommutatorinformationen markiert.

### 36.15.2. NEGATOR

Die Klausel `NEGATOR`, falls angegeben, benennt einen Operator, der der Negator des zu definierenden Operators ist. Wir sagen, Operator A ist der Negator von Operator B, wenn beide boolesche Ergebnisse zurückgeben und `(x A y)` für alle möglichen Eingaben `x` und `y` gleich `NOT (x B y)` ist. Beachten Sie, dass B dann auch der Negator von A ist. Zum Beispiel bilden `<` und `>=` für die meisten Datentypen ein Negatorpaar. Ein Operator kann niemals gültig sein eigener Negator sein.

Anders als bei Kommutatoren könnte ein Paar unärer Operatoren gültig als gegenseitige Negatoren markiert werden; das würde bedeuten, dass `(A x)` für alle `x` gleich `NOT (B x)` ist.

Der Negator eines Operators muss dieselben linken und/oder rechten Operandentypen haben wie der zu definierende Operator. Daher muss, wie bei `COMMUTATOR`, in der Klausel `NEGATOR` nur der Operatorname angegeben werden.

Einen Negator bereitzustellen ist für den Query Optimizer sehr hilfreich, da Ausdrücke wie `NOT (x = y)` dadurch zu `x <> y` vereinfacht werden können. Das kommt häufiger vor, als man vielleicht denkt, weil `NOT`-Operationen als Folge anderer Umformungen eingefügt werden können.

### 36.15.3. RESTRICT

Die Klausel `RESTRICT`, falls angegeben, benennt eine Funktion zur Schätzung der Restriktionsselektivität für den Operator. Beachten Sie, dass dies ein Funktionsname ist, kein Operatorname. `RESTRICT`-Klauseln sind nur für binäre Operatoren sinnvoll, die `boolean` zurückgeben. Die Idee hinter einem Restriktionsselektivitätsschätzer ist, zu schätzen, welcher Anteil der Zeilen in einer Tabelle eine `WHERE`-Klauselbedingung der Form:

```text
column OP constant
```

für den aktuellen Operator und einen bestimmten Konstantenwert erfüllt. Das hilft dem Optimizer, indem es ihm eine Vorstellung davon gibt, wie viele Zeilen durch `WHERE`-Klauseln dieser Form eliminiert werden. Was passiert, wenn die Konstante links steht? Dafür ist unter anderem `COMMUTATOR` da.

Das Schreiben neuer Funktionen zur Schätzung der Restriktionsselektivität geht weit über den Rahmen dieses Kapitels hinaus. Glücklicherweise können Sie für viele eigene Operatoren normalerweise einfach einen der Standardschätzer des Systems verwenden. Dies sind die standardmäßigen Restriktionsschätzer:

| Schätzer | Für |
| --- | --- |
| `eqsel` | `=` |
| `neqsel` | `<>` |
| `scalarltsel` | `<` |
| `scalarlesel` | `<=` |
| `scalargtsel` | `>` |
| `scalargesel` | `>=` |

Häufig können Sie für Operatoren mit sehr hoher oder sehr niedriger Selektivität entweder `eqsel` oder `neqsel` verwenden, selbst wenn sie nicht wirklich Gleichheit oder Ungleichheit darstellen. Zum Beispiel verwenden die geometrischen Operatoren für ungefähre Gleichheit `eqsel` unter der Annahme, dass sie normalerweise nur einen kleinen Anteil der Einträge in einer Tabelle treffen.

Sie können `scalarltsel`, `scalarlesel`, `scalargtsel` und `scalargesel` für Vergleiche auf Datentypen verwenden, die auf sinnvolle Weise in numerische Skalare für Bereichsvergleiche umgewandelt werden können. Wenn möglich, fügen Sie den Datentyp zu denen hinzu, die von der Funktion `convert_to_scalar()` in `src/backend/utils/adt/selfuncs.c` verstanden werden. Irgendwann sollte diese Funktion durch datentypspezifische Funktionen ersetzt werden, die über eine Spalte des Systemkatalogs `pg_type` identifiziert werden; das ist aber noch nicht geschehen. Wenn Sie dies nicht tun, funktioniert es trotzdem, aber die Schätzungen des Optimizers werden nicht so gut sein, wie sie sein könnten.

Eine weitere nützliche eingebaute Funktion zur Selektivitätsschätzung ist `matchingsel`, die für fast jeden binären Operator funktioniert, wenn Standard-MCV- und/oder Histogrammstatistiken für die Eingabedatentypen gesammelt werden. Ihre Standardschätzung ist auf das Doppelte der in `eqsel` verwendeten Standardschätzung gesetzt, wodurch sie besonders für Vergleichsoperatoren geeignet ist, die etwas weniger streng sind als Gleichheit. Alternativ könnten Sie die zugrunde liegende Funktion `generic_restriction_selectivity` mit einer anderen Standardschätzung aufrufen.

Es gibt zusätzliche Selektivitätsschätzfunktionen für geometrische Operatoren in `src/backend/utils/adt/geo_selfuncs.c`: `areasel`, `positionsel` und `contsel`. Zum Zeitpunkt dieser Beschreibung sind dies nur Stubs, aber Sie könnten sie trotzdem verwenden oder, noch besser, verbessern.

### 36.15.4. JOIN

Die Klausel `JOIN`, falls angegeben, benennt eine Funktion zur Schätzung der Join-Selektivität für den Operator. Beachten Sie, dass dies ein Funktionsname ist, kein Operatorname. `JOIN`-Klauseln sind nur für binäre Operatoren sinnvoll, die `boolean` zurückgeben. Die Idee hinter einem Join-Selektivitätsschätzer ist, zu schätzen, welcher Anteil der Zeilen in einem Tabellenpaar eine `WHERE`-Klauselbedingung der Form:

```text
table1.column1 OP table2.column2
```

für den aktuellen Operator erfüllt. Wie bei der Klausel `RESTRICT` hilft dies dem Optimizer erheblich, weil er dadurch abschätzen kann, welche von mehreren möglichen Join-Reihenfolgen wahrscheinlich den geringsten Aufwand verursacht.

Wie zuvor versucht dieses Kapitel nicht zu erklären, wie man eine Funktion zur Schätzung der Join-Selektivität schreibt, sondern schlägt nur vor, einen der Standardschätzer zu verwenden, sofern einer passt:

| Schätzer | Für |
| --- | --- |
| `eqjoinsel` | `=` |
| `neqjoinsel` | `<>` |
| `scalarltjoinsel` | `<` |
| `scalarlejoinsel` | `<=` |
| `scalargtjoinsel` | `>` |
| `scalargejoinsel` | `>=` |
| `matchingjoinsel` | generische Matching-Operatoren |
| `areajoinsel` | 2D-flächenbasierte Vergleiche |
| `positionjoinsel` | 2D-positionsbasierte Vergleiche |
| `contjoinsel` | 2D-Containment-basierte Vergleiche |

### 36.15.5. HASHES

Die Klausel `HASHES`, falls vorhanden, teilt dem System mit, dass die Hash-Join-Methode für einen Join auf Basis dieses Operators verwendet werden darf. `HASHES` ist nur für einen binären Operator sinnvoll, der `boolean` zurückgibt, und in der Praxis muss der Operator Gleichheit für einen Datentyp oder ein Paar von Datentypen darstellen.

Die dem Hash Join zugrunde liegende Annahme ist, dass der Join-Operator nur für Paare linker und rechter Werte wahr zurückgeben kann, die auf denselben Hashcode hashen. Wenn zwei Werte in unterschiedliche Hash Buckets gelangen, vergleicht der Join sie überhaupt nicht und nimmt implizit an, dass das Ergebnis des Join-Operators false sein muss. Daher ist es nie sinnvoll, `HASHES` für Operatoren anzugeben, die keine Form von Gleichheit darstellen. In den meisten Fällen ist es nur praktisch, Hashing für Operatoren zu unterstützen, die auf beiden Seiten denselben Datentyp annehmen. Manchmal ist es jedoch möglich, kompatible Hashfunktionen für zwei oder mehr Datentypen zu entwerfen, also Funktionen, die dieselben Hashcodes für „gleiche“ Werte erzeugen, obwohl die Werte unterschiedliche Darstellungen haben. Beim Hashing von Ganzzahlen unterschiedlicher Breite lässt sich diese Eigenschaft zum Beispiel recht einfach herstellen.

Um als `HASHES` markiert zu werden, muss der Join-Operator in einer Hash-Index-Operatorfamilie erscheinen. Das wird beim Erstellen des Operators nicht erzwungen, da die referenzierende Operatorfamilie zu diesem Zeitpunkt natürlich noch nicht existieren könnte. Versuche, den Operator in Hash Joins zu verwenden, schlagen zur Laufzeit jedoch fehl, wenn keine solche Operatorfamilie existiert. Das System benötigt die Operatorfamilie, um die datentypspezifischen Hashfunktionen für die Eingabedatentypen des Operators zu finden. Natürlich müssen Sie auch passende Hashfunktionen erstellen, bevor Sie die Operatorfamilie erstellen können.

Beim Vorbereiten einer Hashfunktion ist Vorsicht geboten, weil sie auf maschinenabhängige Weise fehlschlagen kann. Wenn Ihr Datentyp zum Beispiel eine Struktur ist, in der uninteressante Padding-Bits vorhanden sein können, können Sie nicht einfach die gesamte Struktur an `hash_any` übergeben, es sei denn, Sie schreiben Ihre anderen Operatoren und Funktionen so, dass die unbenutzten Bits immer null sind, was die empfohlene Strategie ist. Ein weiteres Beispiel ist, dass auf Maschinen, die dem IEEE-Gleitkommastandard entsprechen, negatives und positives Null unterschiedliche Werte, also unterschiedliche Bitmuster, sind, aber als gleich vergleichend definiert sind. Wenn ein `float`-Wert negative Null enthalten könnte, sind zusätzliche Schritte nötig, um sicherzustellen, dass er denselben Hashwert erzeugt wie positive Null.

Ein hash-joinbarer Operator muss einen Kommutator haben, der in derselben Operatorfamilie erscheint: sich selbst, wenn die beiden Operandendatentypen gleich sind, oder einen verwandten Gleichheitsoperator, wenn sie verschieden sind. Ist das nicht der Fall, können Planner-Fehler auftreten, wenn der Operator verwendet wird. Außerdem ist es für eine Hash-Operatorfamilie, die mehrere Datentypen unterstützt, eine gute Idee, wenn auch nicht zwingend erforderlich, Gleichheitsoperatoren für jede Kombination der Datentypen bereitzustellen; das ermöglicht bessere Optimierung.

> Hinweis: Die Funktion, die einem hash-joinbaren Operator zugrunde liegt, muss als immutable oder stable markiert sein. Wenn sie volatile ist, versucht das System niemals, den Operator für einen Hash Join zu verwenden.

> Hinweis: Wenn ein hash-joinbarer Operator eine zugrunde liegende Funktion hat, die als strict markiert ist, muss die Funktion außerdem vollständig sein: Sie sollte also für zwei beliebige nichtnullige Eingaben `true` oder `false` zurückgeben, niemals null. Wird diese Regel nicht eingehalten, kann die Hash-Optimierung von `IN`-Operationen falsche Ergebnisse erzeugen. Konkret könnte `IN` false zurückgeben, wo das korrekte Ergebnis nach dem Standard null wäre, oder einen Fehler ausgeben, dass es nicht auf ein Nullergebnis vorbereitet war.

### 36.15.6. MERGES

Die Klausel `MERGES`, falls vorhanden, teilt dem System mit, dass die Merge-Join-Methode für einen Join auf Basis dieses Operators verwendet werden darf. `MERGES` ist nur für einen binären Operator sinnvoll, der `boolean` zurückgibt, und in der Praxis muss der Operator Gleichheit für einen Datentyp oder ein Paar von Datentypen darstellen.

Merge Join beruht auf der Idee, die linke und die rechte Tabelle in eine Reihenfolge zu sortieren und sie dann parallel zu scannen. Daher müssen beide Datentypen vollständig sortierbar sein, und der Join-Operator muss nur für Wertepaare erfolgreich sein können, die an derselben Stelle in der Sortierreihenfolge liegen. In der Praxis bedeutet das, dass sich der Join-Operator wie Gleichheit verhalten muss. Es ist jedoch möglich, zwei verschiedene Datentypen per Merge Join zu verbinden, solange sie logisch kompatibel sind. Zum Beispiel ist der Gleichheitsoperator zwischen `smallint` und `integer` merge-joinbar. Wir benötigen nur Sortieroperatoren, die beide Datentypen in eine logisch kompatible Reihenfolge bringen.

Um als `MERGES` markiert zu werden, muss der Join-Operator als Gleichheitsmitglied einer B-Tree-Index-Operatorfamilie erscheinen. Das wird beim Erstellen des Operators nicht erzwungen, da die referenzierende Operatorfamilie zu diesem Zeitpunkt natürlich noch nicht existieren könnte. Der Operator wird jedoch tatsächlich nicht für Merge Joins verwendet, sofern keine passende Operatorfamilie gefunden werden kann. Das Flag `MERGES` wirkt daher als Hinweis an den Planner, dass es sich lohnt, nach einer passenden Operatorfamilie zu suchen.

Ein merge-joinbarer Operator muss einen Kommutator haben, der in derselben Operatorfamilie erscheint: sich selbst, wenn die beiden Operandendatentypen gleich sind, oder einen verwandten Gleichheitsoperator, wenn sie verschieden sind. Ist das nicht der Fall, können Planner-Fehler auftreten, wenn der Operator verwendet wird. Außerdem ist es für eine B-Tree-Operatorfamilie, die mehrere Datentypen unterstützt, eine gute Idee, wenn auch nicht zwingend erforderlich, Gleichheitsoperatoren für jede Kombination der Datentypen bereitzustellen; das ermöglicht bessere Optimierung.

> Hinweis: Die Funktion, die einem merge-joinbaren Operator zugrunde liegt, muss als immutable oder stable markiert sein. Wenn sie volatile ist, versucht das System niemals, den Operator für einen Merge Join zu verwenden.

## 36.16. Erweiterungen an Indizes anbinden

Mit den bisher beschriebenen Verfahren können Sie neue Typen, neue Funktionen und neue Operatoren definieren. Wir können jedoch noch keinen Index auf einer Spalte eines neuen Datentyps definieren. Dazu müssen wir eine Operatorklasse für den neuen Datentyp definieren. Später in diesem Abschnitt veranschaulichen wir dieses Konzept an einem Beispiel: einer neuen Operatorklasse für die B-Tree-Indexmethode, die komplexe Zahlen speichert und nach aufsteigendem Absolutwert sortiert.

Operatorklassen können zu Operatorfamilien gruppiert werden, um die Beziehungen zwischen semantisch kompatiblen Klassen zu zeigen. Wenn nur ein einzelner Datentyp beteiligt ist, reicht eine Operatorklasse aus; daher konzentrieren wir uns zuerst auf diesen Fall und kehren danach zu Operatorfamilien zurück.

### 36.16.1. Indexmethoden und Operatorklassen

Operatorklassen sind mit einer Indexzugriffsmethode wie B-Tree oder GIN verknüpft. Benutzerdefinierte Indexzugriffsmethoden können mit `CREATE ACCESS METHOD` definiert werden. Einzelheiten finden Sie in [Kapitel 63](63_Schnittstellendefinition_für_Index_Access_Methods.md).

Die Routinen einer Indexmethode wissen nicht direkt etwas über die Datentypen, auf denen die Indexmethode arbeitet. Stattdessen identifiziert eine Operatorklasse die Menge von Operationen, die die Indexmethode verwenden muss, um mit einem bestimmten Datentyp zu arbeiten. Operatorklassen heißen so, weil sie unter anderem die Menge der `WHERE`-Klausel-Operatoren angeben, die mit einem Index verwendet werden können, also in eine Index-Scan-Qualifikation umgewandelt werden können. Eine Operatorklasse kann außerdem Support-Funktionen angeben, die von den internen Operationen der Indexmethode benötigt werden, aber keinem `WHERE`-Klausel-Operator direkt entsprechen, der mit dem Index verwendet werden kann.

Es ist möglich, mehrere Operatorklassen für denselben Datentyp und dieselbe Indexmethode zu definieren. Dadurch können mehrere Mengen von Indizierungssemantik für einen einzelnen Datentyp definiert werden. Ein B-Tree-Index benötigt zum Beispiel für jeden Datentyp, mit dem er arbeitet, eine definierte Sortierreihenfolge. Für einen Datentyp für komplexe Zahlen könnte es nützlich sein, eine B-Tree-Operatorklasse zu haben, die die Daten nach komplexem Absolutwert sortiert, eine andere, die nach Realteil sortiert, und so weiter. Typischerweise wird eine der Operatorklassen als am allgemeinsten nützlich angesehen und als Standard-Operatorklasse für diesen Datentyp und diese Indexmethode markiert.

Derselbe Operatorklassenname kann für mehrere verschiedene Indexmethoden verwendet werden; zum Beispiel haben sowohl B-Tree- als auch Hash-Indexmethoden Operatorklassen namens `int4_ops`. Jede solche Klasse ist jedoch eine eigenständige Entität und muss separat definiert werden.

### 36.16.2. Strategien von Indexmethoden

Die mit einer Operatorklasse verknüpften Operatoren werden durch „Strategienummern“ identifiziert, die dazu dienen, die Semantik jedes Operators im Kontext seiner Operatorklasse zu bestimmen. B-Trees erzwingen zum Beispiel eine strikte Ordnung der Schlüssel von kleiner nach größer, daher sind Operatoren wie „kleiner als“ und „größer oder gleich“ im Hinblick auf einen B-Tree interessant. Da PostgreSQL Benutzern erlaubt, Operatoren zu definieren, kann PostgreSQL nicht einfach auf den Namen eines Operators, etwa `<` oder `>=`, schauen und daraus ableiten, welche Art von Vergleich er darstellt. Stattdessen definiert die Indexmethode eine Menge von „Strategien“, die man als verallgemeinerte Operatoren verstehen kann. Jede Operatorklasse gibt an, welcher tatsächliche Operator für einen bestimmten Datentyp und eine bestimmte Interpretation der Indexsemantik welcher Strategie entspricht.

Die B-Tree-Indexmethode definiert fünf Strategien, wie in Tabelle 36.3 gezeigt.

**Tabelle 36.3. B-Tree-Strategien**

| Operation | Strategienummer |
| --- | ---: |
| kleiner als | 1 |
| kleiner oder gleich | 2 |
| gleich | 3 |
| größer oder gleich | 4 |
| größer als | 5 |

Hash-Indizes unterstützen nur Gleichheitsvergleiche und verwenden daher nur eine Strategie, wie in Tabelle 36.4 gezeigt.

**Tabelle 36.4. Hash-Strategien**

| Operation | Strategienummer |
| --- | ---: |
| gleich | 1 |

GiST-Indizes sind flexibler: Sie haben überhaupt keine feste Menge von Strategien. Stattdessen interpretiert die „consistency“-Support-Routine jeder einzelnen GiST-Operatorklasse die Strategienummern nach Belieben. Als Beispiel indizieren mehrere der eingebauten GiST-Indexoperatorklassen zweidimensionale geometrische Objekte und stellen die in Tabelle 36.5 gezeigten „R-Tree“-Strategien bereit. Vier davon sind echte zweidimensionale Tests (überlappt, gleich, enthält, ist enthalten in); vier betrachten nur die X-Richtung; und die übrigen vier stellen dieselben Tests in Y-Richtung bereit.

**Tabelle 36.5. Zweidimensionale „R-Tree“-Strategien für GiST**

| Operation | Strategienummer |
| --- | ---: |
| strikt links von | 1 |
| erstreckt sich nicht rechts von | 2 |
| überlappt | 3 |
| erstreckt sich nicht links von | 4 |
| strikt rechts von | 5 |
| gleich | 6 |
| enthält | 7 |
| ist enthalten in | 8 |
| erstreckt sich nicht oberhalb | 9 |
| strikt unterhalb | 10 |
| strikt oberhalb | 11 |
| erstreckt sich nicht unterhalb | 12 |

SP-GiST-Indizes ähneln GiST-Indizes in ihrer Flexibilität: Sie haben keine feste Menge von Strategien. Stattdessen interpretieren die Support-Routinen jeder Operatorklasse die Strategienummern entsprechend der Definition der Operatorklasse. Als Beispiel sind die von den eingebauten Operatorklassen für Punkte verwendeten Strategienummern in Tabelle 36.6 gezeigt.

**Tabelle 36.6. SP-GiST-Punktstrategien**

| Operation | Strategienummer |
| --- | ---: |
| strikt links von | 1 |
| strikt rechts von | 5 |
| gleich | 6 |
| ist enthalten in | 8 |
| strikt unterhalb | 10 |
| strikt oberhalb | 11 |

GIN-Indizes ähneln GiST- und SP-GiST-Indizes darin, dass auch sie keine feste Menge von Strategien haben. Stattdessen interpretieren die Support-Routinen jeder Operatorklasse die Strategienummern entsprechend der Definition der Operatorklasse. Als Beispiel sind die von der eingebauten Operatorklasse für Arrays verwendeten Strategienummern in Tabelle 36.7 gezeigt.

**Tabelle 36.7. GIN-Array-Strategien**

| Operation | Strategienummer |
| --- | ---: |
| überlappt | 1 |
| enthält | 2 |
| ist enthalten in | 3 |
| gleich | 4 |

BRIN-Indizes ähneln GiST-, SP-GiST- und GIN-Indizes darin, dass auch sie keine feste Menge von Strategien haben. Stattdessen interpretieren die Support-Routinen jeder Operatorklasse die Strategienummern entsprechend der Definition der Operatorklasse. Als Beispiel sind die von den eingebauten Minmax-Operatorklassen verwendeten Strategienummern in Tabelle 36.8 gezeigt.

**Tabelle 36.8. BRIN-Minmax-Strategien**

| Operation | Strategienummer |
| --- | ---: |
| kleiner als | 1 |
| kleiner oder gleich | 2 |
| gleich | 3 |
| größer oder gleich | 4 |
| größer als | 5 |

Beachten Sie, dass alle oben aufgeführten Operatoren boolesche Werte zurückgeben. In der Praxis müssen alle als Suchoperatoren einer Indexmethode definierten Operatoren den Typ `boolean` zurückgeben, da sie auf oberster Ebene einer `WHERE`-Klausel erscheinen müssen, um mit einem Index verwendet zu werden. Einige Indexzugriffsmethoden unterstützen außerdem Ordnungsoperatoren, die typischerweise keine booleschen Werte zurückgeben; diese Funktionalität wird in [Abschnitt 36.16.7](#36167-ordnungsoperatoren) besprochen.

### 36.16.3. Support-Routinen von Indexmethoden

Strategien liefern dem System normalerweise nicht genug Informationen, um herauszufinden, wie ein Index verwendet werden soll. In der Praxis benötigen Indexmethoden zusätzliche Support-Routinen, um zu funktionieren. Die B-Tree-Indexmethode muss zum Beispiel zwei Schlüssel vergleichen und bestimmen können, ob einer größer als, gleich oder kleiner als der andere ist. Ebenso muss die Hash-Indexmethode Hashcodes für Schlüsselwerte berechnen können. Diese Operationen entsprechen nicht den Operatoren, die in Qualifikationen von SQL-Befehlen verwendet werden; es sind administrative Routinen, die intern von den Indexmethoden genutzt werden.

Wie bei Strategien identifiziert die Operatorklasse, welche konkreten Funktionen für einen bestimmten Datentyp und eine bestimmte semantische Interpretation jede dieser Rollen übernehmen sollen. Die Indexmethode definiert die Menge von Funktionen, die sie benötigt, und die Operatorklasse identifiziert die richtigen zu verwendenden Funktionen, indem sie sie den von der Indexmethode angegebenen „Support-Funktionsnummern“ zuordnet.

Zusätzlich erlauben einige Operatorklassen Benutzern, Parameter anzugeben, die ihr Verhalten steuern. Jede eingebaute Indexzugriffsmethode hat eine optionale Options-Support-Funktion, die eine Menge von operatorklassenspezifischen Parametern definiert.

B-Trees benötigen eine Vergleichs-Support-Funktion und erlauben, nach Wahl des Autors der Operatorklasse, fünf zusätzliche Support-Funktionen, wie in Tabelle 36.9 gezeigt. Die Anforderungen an diese Support-Funktionen werden in [Abschnitt 65.1.3](65_Eingebaute_Index_Access_Methods.md#6513-btreesupportfunktionen) näher erläutert.

**Tabelle 36.9. B-Tree-Support-Funktionen**

| Funktion | Support-Nummer |
| --- | ---: |
| Zwei Schlüssel vergleichen und eine Ganzzahl kleiner als null, null oder größer als null zurückgeben, die angibt, ob der erste Schlüssel kleiner als, gleich oder größer als der zweite ist | 1 |
| Die Adressen von aus C aufrufbaren Sort-Support-Funktionen zurückgeben (optional) | 2 |
| Einen Testwert mit einem Basiswert plus/minus einem Offset vergleichen und je nach Vergleichsergebnis true oder false zurückgeben (optional) | 3 |
| Bestimmen, ob es für Indizes, die die Operatorklasse verwenden, sicher ist, die B-Tree-Deduplication-Optimierung anzuwenden (optional) | 4 |
| Optionen definieren, die spezifisch für diese Operatorklasse sind (optional) | 5 |
| Die Adressen von aus C aufrufbaren Skip-Support-Funktionen zurückgeben (optional) | 6 |

Hash-Indizes benötigen eine Support-Funktion und erlauben, nach Wahl des Autors der Operatorklasse, zwei zusätzliche, wie in Tabelle 36.10 gezeigt.

**Tabelle 36.10. Hash-Support-Funktionen**

| Funktion | Support-Nummer |
| --- | ---: |
| Den 32-Bit-Hashwert für einen Schlüssel berechnen | 1 |
| Den 64-Bit-Hashwert für einen Schlüssel bei gegebenem 64-Bit-Salt berechnen; wenn der Salt 0 ist, müssen die unteren 32 Bit des Ergebnisses dem Wert entsprechen, der von Funktion 1 berechnet worden wäre (optional) | 2 |
| Optionen definieren, die spezifisch für diese Operatorklasse sind (optional) | 3 |

GiST-Indizes haben zwölf Support-Funktionen, von denen sieben optional sind, wie in Tabelle 36.11 gezeigt. Weitere Informationen finden Sie in [Abschnitt 65.2](65_Eingebaute_Index_Access_Methods.md#652-gistindizes).

**Tabelle 36.11. GiST-Support-Funktionen**

| Funktion | Beschreibung | Support-Nummer |
| --- | --- | ---: |
| `consistent` | bestimmen, ob der Schlüssel die Query-Qualifikation erfüllt | 1 |
| `union` | die Vereinigung einer Menge von Schlüsseln berechnen | 2 |
| `compress` | eine komprimierte Darstellung eines zu indizierenden Schlüssels oder Werts berechnen (optional) | 3 |
| `decompress` | eine dekomprimierte Darstellung eines komprimierten Schlüssels berechnen (optional) | 4 |
| `penalty` | die Strafkosten für das Einfügen eines neuen Schlüssels in einen Teilbaum mit gegebenem Teilbaumschlüssel berechnen | 5 |
| `picksplit` | bestimmen, welche Einträge einer Seite auf die neue Seite verschoben werden sollen, und die Union-Schlüssel für die entstehenden Seiten berechnen | 6 |
| `same` | zwei Schlüssel vergleichen und true zurückgeben, wenn sie gleich sind | 7 |
| `distance` | die Distanz vom Schlüssel zum Query-Wert bestimmen (optional) | 8 |
| `fetch` | die Originaldarstellung eines komprimierten Schlüssels für Index-Only Scans berechnen (optional) | 9 |
| `options` | Optionen definieren, die spezifisch für diese Operatorklasse sind (optional) | 10 |
| `sortsupport` | einen Sortierkomparator bereitstellen, der bei schnellen Index-Builds verwendet wird (optional) | 11 |
| `translate_cmptype` | Vergleichstypen in von der Operatorklasse verwendete Strategienummern übersetzen (optional) | 12 |

SP-GiST-Indizes haben sechs Support-Funktionen, von denen eine optional ist, wie in Tabelle 36.12 gezeigt. Weitere Informationen finden Sie in [Abschnitt 65.3](65_Eingebaute_Index_Access_Methods.md#653-spgistindizes).

**Tabelle 36.12. SP-GiST-Support-Funktionen**

| Funktion | Beschreibung | Support-Nummer |
| --- | --- | ---: |
| `config` | grundlegende Informationen über die Operatorklasse bereitstellen | 1 |
| `choose` | bestimmen, wie ein neuer Wert in ein inneres Tupel eingefügt wird | 2 |
| `picksplit` | bestimmen, wie eine Menge von Werten partitioniert wird | 3 |
| `inner_consistent` | bestimmen, welche Unterpartitionen für eine Abfrage durchsucht werden müssen | 4 |
| `leaf_consistent` | bestimmen, ob der Schlüssel die Query-Qualifikation erfüllt | 5 |
| `options` | Optionen definieren, die spezifisch für diese Operatorklasse sind (optional) | 6 |

GIN-Indizes haben sieben Support-Funktionen, von denen vier optional sind, wie in Tabelle 36.13 gezeigt. Weitere Informationen finden Sie in [Abschnitt 65.4](65_Eingebaute_Index_Access_Methods.md#654-ginindizes).

**Tabelle 36.13. GIN-Support-Funktionen**

| Funktion | Beschreibung | Support-Nummer |
| --- | --- | ---: |
| `compare` | zwei Schlüssel vergleichen und eine Ganzzahl kleiner als null, null oder größer als null zurückgeben, die angibt, ob der erste Schlüssel kleiner als, gleich oder größer als der zweite ist | 1 |
| `extractValue` | Schlüssel aus einem zu indizierenden Wert extrahieren | 2 |
| `extractQuery` | Schlüssel aus einer Query-Bedingung extrahieren | 3 |
| `consistent` | bestimmen, ob ein Wert zur Query-Bedingung passt (boolesche Variante; optional, wenn Support-Funktion 6 vorhanden ist) | 4 |
| `comparePartial` | einen partiellen Schlüssel aus der Query mit einem Schlüssel aus dem Index vergleichen und eine Ganzzahl kleiner als null, null oder größer als null zurückgeben, die angibt, ob GIN diesen Indexeintrag ignorieren, ihn als Treffer behandeln oder den Index Scan beenden soll (optional) | 5 |
| `triConsistent` | bestimmen, ob ein Wert zur Query-Bedingung passt (ternäre Variante; optional, wenn Support-Funktion 4 vorhanden ist) | 6 |
| `options` | Optionen definieren, die spezifisch für diese Operatorklasse sind (optional) | 7 |

BRIN-Indizes haben fünf grundlegende Support-Funktionen, von denen eine optional ist, wie in Tabelle 36.14 gezeigt. Einige Versionen der Grundfunktionen erfordern zusätzliche Support-Funktionen. Weitere Informationen finden Sie in [Abschnitt 65.5.3](65_Eingebaute_Index_Access_Methods.md#6553-erweiterbarkeit).

**Tabelle 36.14. BRIN-Support-Funktionen**

| Funktion | Beschreibung | Support-Nummer |
| --- | --- | ---: |
| `opcInfo` | interne Informationen zurückgeben, die die Zusammenfassungsdaten der indizierten Spalten beschreiben | 1 |
| `add_value` | einen neuen Wert zu einem vorhandenen Summary-Index-Tupel hinzufügen | 2 |
| `consistent` | bestimmen, ob ein Wert zur Query-Bedingung passt | 3 |
| `union` | die Vereinigung zweier Summary-Tupel berechnen | 4 |
| `options` | Optionen definieren, die spezifisch für diese Operatorklasse sind (optional) | 5 |

Anders als Suchoperatoren geben Support-Funktionen den Datentyp zurück, den die jeweilige Indexmethode erwartet; im Fall der Vergleichsfunktion für B-Trees zum Beispiel eine vorzeichenbehaftete Ganzzahl. Anzahl und Typen der Argumente jeder Support-Funktion hängen ebenfalls von der Indexmethode ab. Bei B-Tree und Hash nehmen die Vergleichs- und Hash-Support-Funktionen dieselben Eingabedatentypen an wie die in der Operatorklasse enthaltenen Operatoren, aber das gilt nicht für die meisten GiST-, SP-GiST-, GIN- und BRIN-Support-Funktionen.

### 36.16.4. Ein Beispiel

Nachdem wir die Ideen gesehen haben, folgt hier das versprochene Beispiel zum Erstellen einer neuen Operatorklasse. Eine lauffähige Kopie dieses Beispiels finden Sie in `src/tutorial/complex.c` und `src/tutorial/complex.sql` in der Quelldistribution. Die Operatorklasse kapselt Operatoren, die komplexe Zahlen nach ihrem Absolutwert sortieren, daher wählen wir den Namen `complex_abs_ops`. Zuerst benötigen wir eine Menge von Operatoren. Das Verfahren zum Definieren von Operatoren wurde in [Abschnitt 36.14](#3614-benutzerdefinierte-operatoren) besprochen. Für eine Operatorklasse auf B-Trees benötigen wir folgende Operatoren:

- Absolutwert-kleiner-als (Strategie 1)
- Absolutwert-kleiner-oder-gleich (Strategie 2)
- Absolutwert-gleich (Strategie 3)
- Absolutwert-größer-oder-gleich (Strategie 4)
- Absolutwert-größer-als (Strategie 5)

Die am wenigsten fehleranfällige Methode, eine zusammengehörige Menge von Vergleichsoperatoren zu definieren, besteht darin, zuerst die B-Tree-Vergleichs-Support-Funktion zu schreiben und danach die anderen Funktionen als einzeilige Wrapper um die Support-Funktion zu implementieren. Das verringert die Wahrscheinlichkeit, bei Randfällen inkonsistente Ergebnisse zu erhalten. Nach diesem Ansatz schreiben wir zunächst:

```c
#define Mag(c)          ((c)->x*(c)->x + (c)->y*(c)->y)

static int
complex_abs_cmp_internal(Complex *a, Complex *b)
{
    double      amag = Mag(a),
                bmag = Mag(b);

    if (amag < bmag)
        return -1;
    if (amag > bmag)
        return 1;
    return 0;
}
```

Die Kleiner-als-Funktion sieht dann so aus:

```c
PG_FUNCTION_INFO_V1(complex_abs_lt);

Datum
complex_abs_lt(PG_FUNCTION_ARGS)
{
    Complex    *a = (Complex *) PG_GETARG_POINTER(0);
    Complex    *b = (Complex *) PG_GETARG_POINTER(1);

    PG_RETURN_BOOL(complex_abs_cmp_internal(a, b) < 0);
}
```

Die anderen vier Funktionen unterscheiden sich nur darin, wie sie das Ergebnis der internen Funktion mit null vergleichen.

Als Nächstes deklarieren wir die Funktionen und die darauf beruhenden Operatoren in SQL:

```sql
CREATE FUNCTION complex_abs_lt(complex, complex) RETURNS bool
    AS 'filename', 'complex_abs_lt'
    LANGUAGE C IMMUTABLE STRICT;

CREATE OPERATOR < (
   leftarg = complex, rightarg = complex, procedure = complex_abs_lt,
   commutator = > , negator = >= ,
   restrict = scalarltsel, join = scalarltjoinsel
);
```

Es ist wichtig, die richtigen Kommutator- und Negatoroperatoren sowie passende Funktionen für Restriktions- und Join-Selektivität anzugeben; andernfalls kann der Optimizer den Index nicht wirksam nutzen.

Hier geschehen noch weitere Dinge, die erwähnenswert sind:

- Es kann nur einen Operator geben, der zum Beispiel `=` heißt und für beide Operanden den Typ `complex` annimmt. In diesem Fall haben wir keinen anderen Operator `=` für `complex`, aber wenn wir einen praktischen Datentyp bauen würden, wollten wir wahrscheinlich, dass `=` die gewöhnliche Gleichheitsoperation für komplexe Zahlen ist und nicht die Gleichheit der Absolutwerte. In diesem Fall müssten wir für `complex_abs_eq` einen anderen Operatornamen verwenden.
- Obwohl PostgreSQL mit Funktionen umgehen kann, die denselben SQL-Namen haben, solange sie unterschiedliche Argumentdatentypen haben, kann C nur mit einer globalen Funktion eines bestimmten Namens umgehen. Deshalb sollten wir die C-Funktion nicht einfach `abs_eq` nennen. Üblicherweise ist es gute Praxis, den Datentypnamen in den C-Funktionsnamen aufzunehmen, um Konflikte mit Funktionen für andere Datentypen zu vermeiden.
- Wir hätten den SQL-Namen der Funktion `abs_eq` nennen können und uns darauf verlassen, dass PostgreSQL sie anhand der Argumentdatentypen von anderen SQL-Funktionen gleichen Namens unterscheidet. Um das Beispiel einfach zu halten, geben wir der Funktion auf C-Ebene und SQL-Ebene denselben Namen.

Der nächste Schritt ist die Registrierung der von B-Trees benötigten Support-Routine. Der Beispiel-C-Code, der dies implementiert, steht in derselben Datei wie die Operatorfunktionen. So deklarieren wir die Funktion:

```sql
CREATE FUNCTION complex_abs_cmp(complex, complex)
    RETURNS integer
    AS 'filename'
    LANGUAGE C IMMUTABLE STRICT;
```

Nachdem wir nun die erforderlichen Operatoren und die Support-Routine haben, können wir schließlich die Operatorklasse erstellen:

```sql
CREATE OPERATOR CLASS complex_abs_ops
    DEFAULT FOR TYPE complex USING btree AS
        OPERATOR        1       < ,
        OPERATOR        2       <= ,
        OPERATOR        3       = ,
        OPERATOR        4       >= ,
        OPERATOR        5       > ,
        FUNCTION        1       complex_abs_cmp(complex, complex);
```

Damit sind wir fertig. Es sollte nun möglich sein, B-Tree-Indizes auf `complex`-Spalten zu erstellen und zu verwenden.

Wir hätten die Operatoreinträge ausführlicher schreiben können, etwa so:

```sql
OPERATOR        1       < (complex, complex) ,
```

Das ist aber nicht nötig, wenn die Operatoren denselben Datentyp annehmen, für den wir die Operatorklasse definieren.

Das obige Beispiel nimmt an, dass Sie diese neue Operatorklasse zur Standard-B-Tree-Operatorklasse für den Datentyp `complex` machen möchten. Wenn Sie das nicht wollen, lassen Sie einfach das Wort `DEFAULT` weg.

### 36.16.5. Operatorklassen und Operatorfamilien

Bisher haben wir stillschweigend angenommen, dass eine Operatorklasse nur mit einem Datentyp zu tun hat. Zwar kann es in einer bestimmten Indexspalte immer nur einen Datentyp geben, aber es ist oft nützlich, Operationen zu indizieren, die eine indizierte Spalte mit einem Wert eines anderen Datentyps vergleichen. Wenn außerdem ein typübergreifender Operator im Zusammenhang mit einer Operatorklasse nützlich ist, besitzt der andere Datentyp häufig eine eigene verwandte Operatorklasse. Es hilft, die Verbindungen zwischen verwandten Klassen ausdrücklich zu machen, weil dies den Planer beim Optimieren von SQL-Abfragen unterstützen kann. Das gilt besonders für B-Tree-Operatorklassen, da der Planer viel Wissen darüber besitzt, wie mit ihnen zu arbeiten ist.

Für diese Anforderungen verwendet PostgreSQL das Konzept einer Operatorfamilie. Eine Operatorfamilie enthält eine oder mehrere Operatorklassen und kann außerdem indizierbare Operatoren sowie zugehörige Support-Funktionen enthalten, die zur Familie als Ganzes gehören, aber zu keiner einzelnen Klasse innerhalb der Familie. Solche Operatoren und Funktionen nennen wir innerhalb der Familie „lose“, im Gegensatz zu solchen, die an eine bestimmte Klasse gebunden sind. Typischerweise enthält jede Operatorklasse Operatoren für einen einzelnen Datentyp, während typübergreifende Operatoren lose in der Familie liegen.

Alle Operatoren und Funktionen in einer Operatorfamilie müssen kompatible Semantik besitzen; die Kompatibilitätsanforderungen werden dabei von der Indexmethode vorgegeben. Man könnte sich daher fragen, warum man überhaupt bestimmte Teilmengen der Familie als Operatorklassen herausgreift. Tatsächlich sind die Klassengrenzen für viele Zwecke irrelevant, und die Familie ist die eigentlich interessante Gruppierung. Der Grund für Operatorklassen ist, dass sie angeben, wie viel von der Familie benötigt wird, um einen bestimmten Index zu unterstützen. Wenn ein Index eine Operatorklasse verwendet, kann diese Operatorklasse nicht gelöscht werden, ohne auch den Index zu löschen. Andere Teile der Operatorfamilie, nämlich andere Operatorklassen und lose Operatoren, könnten hingegen gelöscht werden. Eine Operatorklasse sollte daher die minimale Menge an Operatoren und Funktionen enthalten, die vernünftigerweise für einen Index auf einem bestimmten Datentyp benötigt werden. Verwandte, aber nicht unbedingt erforderliche Operatoren können dann als lose Mitglieder zur Operatorfamilie hinzugefügt werden.

Als Beispiel besitzt PostgreSQL die eingebaute B-Tree-Operatorfamilie `integer_ops`. Sie enthält die Operatorklassen `int8_ops`, `int4_ops` und `int2_ops` für Indizes auf Spalten der Typen `bigint` (`int8`), `integer` (`int4`) beziehungsweise `smallint` (`int2`). Die Familie enthält außerdem typübergreifende Vergleichsoperatoren, mit denen beliebige zwei dieser Typen verglichen werden können, sodass ein Index auf einem dieser Typen mit einem Vergleichswert eines anderen Typs durchsucht werden kann. Die Familie ließe sich mit folgenden Definitionen nachbilden:

```sql
CREATE OPERATOR FAMILY integer_ops USING btree;

CREATE OPERATOR CLASS int8_ops
DEFAULT FOR TYPE int8 USING btree FAMILY integer_ops AS
  -- Standardvergleiche fuer int8
  OPERATOR 1 < ,
  OPERATOR 2 <= ,
  OPERATOR 3 = ,
  OPERATOR 4 >= ,
  OPERATOR 5 > ,
  FUNCTION 1 btint8cmp(int8, int8) ,
  FUNCTION 2 btint8sortsupport(internal) ,
  FUNCTION 3 in_range(int8, int8, int8, boolean, boolean) ,
  FUNCTION 4 btequalimage(oid) ,
  FUNCTION 6 btint8skipsupport(internal) ;

CREATE OPERATOR CLASS int4_ops
DEFAULT FOR TYPE int4 USING btree FAMILY integer_ops AS
  -- Standardvergleiche fuer int4
  OPERATOR 1 < ,
  OPERATOR 2 <= ,
  OPERATOR 3 = ,
  OPERATOR 4 >= ,
  OPERATOR 5 > ,
  FUNCTION 1 btint4cmp(int4, int4) ,
  FUNCTION 2 btint4sortsupport(internal) ,
  FUNCTION 3 in_range(int4, int4, int4, boolean, boolean) ,
  FUNCTION 4 btequalimage(oid) ,
  FUNCTION 6 btint4skipsupport(internal) ;

CREATE OPERATOR CLASS int2_ops
DEFAULT FOR TYPE int2 USING btree FAMILY integer_ops AS
  -- Standardvergleiche fuer int2
  OPERATOR 1 < ,
  OPERATOR 2 <= ,
  OPERATOR 3 = ,
  OPERATOR 4 >= ,
  OPERATOR 5 > ,
  FUNCTION 1 btint2cmp(int2, int2) ,
  FUNCTION 2 btint2sortsupport(internal) ,
  FUNCTION 3 in_range(int2, int2, int2, boolean, boolean) ,
  FUNCTION 4 btequalimage(oid) ,
  FUNCTION 6 btint2skipsupport(internal) ;

ALTER OPERATOR FAMILY integer_ops USING btree ADD
  -- typuebergreifende Vergleiche int8 gegen int2
  OPERATOR 1 < (int8, int2) ,
  OPERATOR 2 <= (int8, int2) ,
  OPERATOR 3 = (int8, int2) ,
  OPERATOR 4 >= (int8, int2) ,
  OPERATOR 5 > (int8, int2) ,
  FUNCTION 1 btint82cmp(int8, int2) ,

  -- typuebergreifende Vergleiche int8 gegen int4
  OPERATOR 1 < (int8, int4) ,
  OPERATOR 2 <= (int8, int4) ,
  OPERATOR 3 = (int8, int4) ,
  OPERATOR 4 >= (int8, int4) ,
  OPERATOR 5 > (int8, int4) ,
  FUNCTION 1 btint84cmp(int8, int4) ,

  -- typuebergreifende Vergleiche int4 gegen int2
  OPERATOR 1 < (int4, int2) ,
  OPERATOR 2 <= (int4, int2) ,
  OPERATOR 3 = (int4, int2) ,
  OPERATOR 4 >= (int4, int2) ,
  OPERATOR 5 > (int4, int2) ,
  FUNCTION 1 btint42cmp(int4, int2) ,

  -- typuebergreifende Vergleiche int4 gegen int8
  OPERATOR 1 < (int4, int8) ,
  OPERATOR 2 <= (int4, int8) ,
  OPERATOR 3 = (int4, int8) ,
  OPERATOR 4 >= (int4, int8) ,
  OPERATOR 5 > (int4, int8) ,
  FUNCTION 1 btint48cmp(int4, int8) ,

  -- typuebergreifende Vergleiche int2 gegen int8
  OPERATOR 1 < (int2, int8) ,
  OPERATOR 2 <= (int2, int8) ,
  OPERATOR 3 = (int2, int8) ,
  OPERATOR 4 >= (int2, int8) ,
  OPERATOR 5 > (int2, int8) ,
  FUNCTION 1 btint28cmp(int2, int8) ,

  -- typuebergreifende Vergleiche int2 gegen int4
  OPERATOR 1 < (int2, int4) ,
  OPERATOR 2 <= (int2, int4) ,
  OPERATOR 3 = (int2, int4) ,
  OPERATOR 4 >= (int2, int4) ,
  OPERATOR 5 > (int2, int4) ,
  FUNCTION 1 btint24cmp(int2, int4) ,

  -- typuebergreifende in_range-Funktionen
  FUNCTION 3 in_range(int4, int4, int8, boolean, boolean) ,
  FUNCTION 3 in_range(int4, int4, int2, boolean, boolean) ,
  FUNCTION 3 in_range(int2, int2, int8, boolean, boolean) ,
  FUNCTION 3 in_range(int2, int2, int4, boolean, boolean) ;
```

Beachten Sie, dass diese Definition die Operatorstrategie- und Support-Funktionsnummern „überlädt“: Jede Nummer kommt innerhalb der Familie mehrfach vor. Das ist zulässig, solange jede Instanz einer bestimmten Nummer unterschiedliche Eingabedatentypen besitzt. Die Instanzen, bei denen beide Eingabetypen dem Eingabetyp einer Operatorklasse entsprechen, sind die primären Operatoren und Support-Funktionen für diese Operatorklasse. In den meisten Fällen sollten sie als Teil der Operatorklasse deklariert werden, nicht als lose Mitglieder der Familie.

In einer B-Tree-Operatorfamilie müssen alle Operatoren der Familie kompatibel sortieren, wie in [Abschnitt 65.1.2](65_Eingebaute_Index_Access_Methods.md#6512-verhalten-von-btreeoperatorklassen) ausführlich beschrieben. Für jeden Operator in der Familie muss es eine Support-Funktion mit denselben beiden Eingabedatentypen wie beim Operator geben. Es wird empfohlen, eine Familie vollständig zu machen, das heißt für jede Kombination von Datentypen alle Operatoren aufzunehmen. Jede Operatorklasse sollte nur die nicht typübergreifenden Operatoren und die Support-Funktion für ihren Datentyp enthalten.

Um eine Hash-Operatorfamilie mit mehreren Datentypen aufzubauen, müssen für jeden von der Familie unterstützten Datentyp kompatible Hash-Support-Funktionen erstellt werden. Kompatibilität bedeutet hier, dass die Funktionen garantiert denselben Hashcode für zwei beliebige Werte zurückgeben, die von den Gleichheitsoperatoren der Familie als gleich betrachtet werden, selbst wenn die Werte unterschiedliche Typen haben. Das ist normalerweise schwer zu erreichen, wenn die Typen unterschiedliche physische Darstellungen besitzen, ist aber in manchen Fällen möglich. Außerdem darf eine Umwandlung eines Wertes von einem in der Operatorfamilie vertretenen Datentyp in einen anderen dort vertretenen Datentyp über eine implizite oder binäre Koerzionsumwandlung den berechneten Hashwert nicht verändern. Beachten Sie, dass es nur eine Support-Funktion pro Datentyp gibt, nicht eine pro Gleichheitsoperator. Es wird empfohlen, eine Familie vollständig zu machen, das heißt für jede Kombination von Datentypen einen Gleichheitsoperator bereitzustellen. Jede Operatorklasse sollte nur den nicht typübergreifenden Gleichheitsoperator und die Support-Funktion für ihren Datentyp enthalten.

GiST-, SP-GiST- und GIN-Indizes kennen kein ausdrückliches Konzept für typübergreifende Operationen. Die Menge der unterstützten Operatoren ist einfach das, womit die primären Support-Funktionen einer gegebenen Operatorklasse umgehen können.

Bei BRIN hängen die Anforderungen vom Framework ab, das die Operatorklassen bereitstellt. Für Operatorklassen auf Basis von `minmax` ist das erforderliche Verhalten dasselbe wie bei B-Tree-Operatorfamilien: Alle Operatoren der Familie müssen kompatibel sortieren, und Typumwandlungen dürfen die zugehörige Sortierreihenfolge nicht verändern.

> **Hinweis**
>
> Vor PostgreSQL 8.3 gab es kein Konzept von Operatorfamilien. Deshalb mussten typübergreifende Operatoren, die mit einem Index verwendet werden sollten, direkt an die Operatorklasse des Index gebunden werden. Dieser Ansatz funktioniert zwar weiterhin, ist aber veraltet, weil er die Abhängigkeiten eines Index zu breit macht und weil der Planer typübergreifende Vergleiche effektiver behandeln kann, wenn beide Datentypen Operatoren in derselben Operatorfamilie besitzen.

### 36.16.6. Systemabhängigkeiten von Operatorklassen

PostgreSQL verwendet Operatorklassen nicht nur, um abzuleiten, ob Operatoren mit Indizes verwendet werden können, sondern auch, um weitere Eigenschaften von Operatoren zu erschließen. Deshalb kann es sinnvoll sein, Operatorklassen zu erstellen, selbst wenn Sie gar nicht beabsichtigen, Spalten Ihres Datentyps zu indizieren.

Insbesondere gibt es SQL-Funktionen wie `ORDER BY` und `DISTINCT`, die Werte vergleichen und sortieren müssen. Um diese Funktionen für einen benutzerdefinierten Datentyp umzusetzen, sucht PostgreSQL nach der Standard-B-Tree-Operatorklasse für den Datentyp. Das „Gleich“-Mitglied dieser Operatorklasse definiert den systemweiten Begriff von Gleichheit für Werte in `GROUP BY` und `DISTINCT`; die von der Operatorklasse vorgegebene Sortierreihenfolge definiert die Standardreihenfolge für `ORDER BY`.

Wenn es für einen Datentyp keine Standard-B-Tree-Operatorklasse gibt, sucht das System nach einer Standard-Hash-Operatorklasse. Da diese Art von Operatorklasse aber nur Gleichheit bereitstellt, kann sie Gruppierung unterstützen, nicht jedoch Sortierung.

Wenn es für einen Datentyp keine Standard-Operatorklasse gibt, erhalten Sie Fehlermeldungen wie „could not identify an ordering operator“, wenn Sie diese SQL-Funktionen mit dem Datentyp verwenden.

> **Hinweis**
>
> In PostgreSQL-Versionen vor 7.4 verwendeten Sortier- und Gruppierungsoperationen implizit Operatoren mit den Namen `=`, `<` und `>`. Das neue Verhalten, sich auf Standard-Operatorklassen zu stützen, vermeidet Annahmen über das Verhalten von Operatoren mit bestimmten Namen.

Das Sortieren mit einer nicht standardmäßigen B-Tree-Operatorklasse ist möglich, indem der Kleiner-als-Operator der Klasse in einer `USING`-Option angegeben wird, zum Beispiel:

```sql
SELECT * FROM mytable ORDER BY somecol USING ~<~;
```

Alternativ wählt die Angabe des Größer-als-Operators der Klasse in `USING` eine absteigende Sortierung.

Auch der Vergleich von Arrays eines benutzerdefinierten Typs stützt sich auf die Semantik, die durch die Standard-B-Tree-Operatorklasse des Typs definiert wird. Wenn es keine Standard-B-Tree-Operatorklasse gibt, aber eine Standard-Hash-Operatorklasse vorhanden ist, wird Array-Gleichheit unterstützt, jedoch keine Ordnungsvergleiche.

Eine weitere SQL-Funktion, die noch mehr datentypspezifisches Wissen benötigt, ist die Framing-Option `RANGE` mit `offset PRECEDING/FOLLOWING` für Fensterfunktionen (siehe [Abschnitt 4.2.8](04_SQL_Syntax.md#428-aufrufe-von-windowfunktionen)). Für eine Abfrage wie

```sql
SELECT sum(x) OVER (ORDER BY x RANGE BETWEEN 5 PRECEDING AND 10 FOLLOWING)
  FROM mytable;
```

reicht es nicht aus zu wissen, wie nach `x` sortiert wird. Die Datenbank muss auch verstehen, wie sie vom Wert `x` der aktuellen Zeile „5 subtrahiert“ oder „10 addiert“, um die Grenzen des aktuellen Fensterrahmens zu bestimmen. Der Vergleich der resultierenden Grenzen mit den Werten `x` anderer Zeilen ist mit den Vergleichsoperatoren möglich, die von der B-Tree-Operatorklasse bereitgestellt werden, welche die `ORDER BY`-Reihenfolge definiert. Additions- und Subtraktionsoperatoren sind jedoch nicht Teil der Operatorklasse. Welche sollten also verwendet werden? Diese Wahl fest einzubauen wäre unerwünscht, weil unterschiedliche Sortierreihenfolgen, also unterschiedliche B-Tree-Operatorklassen, unterschiedliches Verhalten benötigen könnten. Daher kann eine B-Tree-Operatorklasse eine `in_range`-Support-Funktion angeben, die das für ihre Sortierreihenfolge sinnvolle Additions- und Subtraktionsverhalten kapselt. Sie kann sogar mehr als eine `in_range`-Support-Funktion bereitstellen, falls mehr als ein Datentyp als Offset in `RANGE`-Klauseln sinnvoll ist. Wenn die zur `ORDER BY`-Klausel des Fensters gehörende B-Tree-Operatorklasse keine passende `in_range`-Support-Funktion besitzt, wird die Option `RANGE offset PRECEDING/FOLLOWING` nicht unterstützt.

Ein weiterer wichtiger Punkt ist, dass ein Gleichheitsoperator, der in einer Hash-Operatorfamilie vorkommt, ein Kandidat für Hash-Joins, Hash-Aggregation und verwandte Optimierungen ist. Die Hash-Operatorfamilie ist hier wesentlich, weil sie die zu verwendenden Hash-Funktionen identifiziert.

### 36.16.7. Ordnungsoperatoren

Einige Indexzugriffsmethoden, derzeit nur GiST und SP-GiST, unterstützen das Konzept von Ordnungsoperatoren. Bisher haben wir Suchoperatoren besprochen. Ein Suchoperator ist ein Operator, bei dem der Index durchsucht werden kann, um alle Zeilen zu finden, die `WHERE indexed_column operator constant` erfüllen. Beachten Sie, dass dabei nichts über die Reihenfolge versprochen wird, in der die passenden Zeilen zurückgegeben werden.

Ein Ordnungsoperator schränkt die Menge der zurückgebbaren Zeilen dagegen nicht ein, sondern bestimmt ihre Reihenfolge. Ein Ordnungsoperator ist ein Operator, bei dem der Index so gescannt werden kann, dass Zeilen in der durch `ORDER BY indexed_column operator constant` dargestellten Reihenfolge zurückgegeben werden. Diese Art der Definition von Ordnungsoperatoren unterstützt Nächster-Nachbar-Suchen, wenn der Operator eine Distanz misst. Zum Beispiel findet eine Abfrage wie

```sql
SELECT * FROM places ORDER BY location <-> point '(101,456)' LIMIT 10;
```

die zehn Orte, die einem gegebenen Zielpunkt am nächsten liegen. Ein GiST-Index auf der Spalte `location` kann dies effizient tun, weil `<->` ein Ordnungsoperator ist.

Während Suchoperatoren boolesche Ergebnisse zurückgeben müssen, liefern Ordnungsoperatoren normalerweise einen anderen Typ, etwa `float` oder `numeric` für Distanzen. Dieser Typ ist normalerweise nicht derselbe wie der indizierte Datentyp. Um keine festen Annahmen über das Verhalten unterschiedlicher Datentypen einzubauen, muss die Definition eines Ordnungsoperators eine B-Tree-Operatorfamilie benennen, die die Sortierreihenfolge des Ergebnisdatentyps angibt. Wie im vorherigen Abschnitt gesagt, definieren B-Tree-Operatorfamilien PostgreSQLs Begriff von Ordnung, daher ist dies eine natürliche Darstellung. Da der Operator `<->` für `point` den Typ `float8` zurückgibt, könnte er in einem Befehl zum Erstellen einer Operatorklasse so angegeben werden:

```sql
OPERATOR 15 <-> (point, point) FOR ORDER BY float_ops
```

Dabei ist `float_ops` die eingebaute Operatorfamilie, die Operationen auf `float8` enthält. Diese Deklaration besagt, dass der Index Zeilen in aufsteigender Reihenfolge der Werte des Operators `<->` zurückgeben kann.

### 36.16.8. Besondere Merkmale von Operatorklassen

Es gibt zwei besondere Merkmale von Operatorklassen, die wir noch nicht besprochen haben, vor allem, weil sie bei den am häufigsten verwendeten Indexmethoden nicht nützlich sind.

Normalerweise bedeutet die Deklaration eines Operators als Mitglied einer Operatorklasse oder Operatorfamilie, dass die Indexmethode exakt die Menge der Zeilen liefern kann, die eine `WHERE`-Bedingung mit diesem Operator erfüllen. Zum Beispiel kann

```sql
SELECT * FROM table WHERE integer_column < 4;
```

exakt durch einen B-Tree-Index auf der Integer-Spalte erfüllt werden. Es gibt aber Fälle, in denen ein Index als ungenauer Wegweiser zu den passenden Zeilen nützlich ist. Wenn ein GiST-Index für geometrische Objekte zum Beispiel nur Begrenzungsrechtecke speichert, kann er eine `WHERE`-Bedingung, die Überlappung zwischen nicht rechteckigen Objekten wie Polygonen prüft, nicht exakt erfüllen. Dennoch könnten wir den Index verwenden, um Objekte zu finden, deren Begrenzungsrechteck das Begrenzungsrechteck des Zielobjekts überlappt, und anschließend den exakten Überlappungstest nur für die vom Index gefundenen Objekte durchführen. Wenn dieses Szenario zutrifft, nennt man den Index für den Operator „verlustbehaftet“.

Verlustbehaftete Indexsuchen werden umgesetzt, indem die Indexmethode ein Recheck-Flag zurückgibt, wenn eine Zeile die Abfragebedingung tatsächlich erfüllen könnte oder auch nicht. Das Kernsystem prüft dann die ursprüngliche Abfragebedingung an der abgerufenen Zeile, um zu entscheiden, ob sie als gültiger Treffer zurückgegeben werden soll. Dieser Ansatz funktioniert, wenn der Index garantiert alle benötigten Zeilen zurückgibt, eventuell plus einige zusätzliche Zeilen, die durch Ausführen des ursprünglichen Operatoraufrufs ausgeschieden werden können. Die Indexmethoden, die verlustbehaftete Suchen unterstützen, derzeit GiST, SP-GiST und GIN, erlauben den Support-Funktionen einzelner Operatorklassen, das Recheck-Flag zu setzen. Damit ist dies im Wesentlichen ein Merkmal der Operatorklasse.

Betrachten wir erneut die Situation, in der wir im Index nur das Begrenzungsrechteck eines komplexen Objekts wie eines Polygons speichern. In diesem Fall hat es wenig Wert, das gesamte Polygon im Indexeintrag zu speichern; wir können ebenso gut nur ein einfacheres Objekt vom Typ `box` speichern. Diese Situation wird durch die Option `STORAGE` in `CREATE OPERATOR CLASS` ausgedrückt. Man würde etwa schreiben:

```sql
CREATE OPERATOR CLASS polygon_ops
    DEFAULT FOR TYPE polygon USING gist AS
        ...
        STORAGE box;
```

Derzeit unterstützen nur die Indexmethoden GiST, SP-GiST, GIN und BRIN einen `STORAGE`-Typ, der sich vom Spaltendatentyp unterscheidet. Die GiST-Support-Routinen `compress` und `decompress` müssen mit der Datentypumwandlung umgehen, wenn `STORAGE` verwendet wird. SP-GiST benötigt ebenfalls eine `compress`-Support-Funktion, um in den Speichertyp umzuwandeln, wenn dieser abweicht. Wenn eine SP-GiST-Operatorklasse außerdem das Abrufen von Daten unterstützt, muss die Rückumwandlung von der Funktion `consistent` behandelt werden. Bei GIN identifiziert der `STORAGE`-Typ den Typ der „Schlüssel“-Werte, der normalerweise vom Typ der indizierten Spalte abweicht. Eine Operatorklasse für Integer-Array-Spalten könnte zum Beispiel Schlüssel haben, die einfach Integer-Werte sind. Die GIN-Support-Routinen `extractValue` und `extractQuery` sind dafür verantwortlich, Schlüssel aus indizierten Werten zu extrahieren. BRIN ähnelt GIN: Der `STORAGE`-Typ identifiziert den Typ der gespeicherten Zusammenfassungswerte, und die Support-Prozeduren der Operatorklassen sind dafür verantwortlich, die Zusammenfassungswerte korrekt zu interpretieren.

## 36.17. Zusammengehörige Objekte in einer Erweiterung paketieren

Eine nützliche Erweiterung für PostgreSQL umfasst typischerweise mehrere SQL-Objekte. Ein neuer Datentyp benötigt zum Beispiel neue Funktionen, neue Operatoren und wahrscheinlich neue Indexoperatorklassen. Es ist hilfreich, all diese Objekte in einem einzelnen Paket zu sammeln, um die Datenbankverwaltung zu vereinfachen. PostgreSQL nennt ein solches Paket eine Erweiterung. Um eine Erweiterung zu definieren, benötigen Sie mindestens eine Skriptdatei, die die SQL-Befehle zum Erzeugen der Objekte der Erweiterung enthält, sowie eine Control-Datei, die einige grundlegende Eigenschaften der Erweiterung selbst angibt. Wenn die Erweiterung C-Code enthält, gibt es typischerweise außerdem eine Shared-Library-Datei, in die der C-Code übersetzt wurde. Sobald diese Dateien vorhanden sind, lädt ein einfacher Befehl `CREATE EXTENSION` die Objekte in Ihre Datenbank.

Der Hauptvorteil einer Erweiterung gegenüber dem bloßen Ausführen eines SQL-Skripts, das eine Reihe „loser“ Objekte in die Datenbank lädt, besteht darin, dass PostgreSQL dann versteht, dass die Objekte der Erweiterung zusammengehören. Sie können alle Objekte mit einem einzigen Befehl `DROP EXTENSION` löschen; ein separates Deinstallationsskript muss nicht gepflegt werden. Noch nützlicher ist, dass `pg_dump` weiß, dass es die einzelnen Mitgliedsobjekte der Erweiterung nicht ausgeben soll. Stattdessen nimmt es nur einen Befehl `CREATE EXTENSION` in Dumps auf. Das vereinfacht die Migration auf eine neue Version der Erweiterung erheblich, die möglicherweise mehr oder andere Objekte enthält als die alte Version. Beachten Sie jedoch, dass die Control-, Skript- und sonstigen Dateien der Erweiterung verfügbar sein müssen, wenn ein solcher Dump in eine neue Datenbank geladen wird.

PostgreSQL erlaubt nicht, ein einzelnes Objekt zu löschen, das in einer Erweiterung enthalten ist, außer indem die gesamte Erweiterung gelöscht wird. Außerdem können Sie zwar die Definition eines Mitgliedsobjekts einer Erweiterung ändern, etwa mit `CREATE OR REPLACE FUNCTION` für eine Funktion, sollten aber bedenken, dass die geänderte Definition nicht von `pg_dump` ausgegeben wird. Eine solche Änderung ist normalerweise nur sinnvoll, wenn Sie gleichzeitig dieselbe Änderung in der Skriptdatei der Erweiterung vornehmen. Für Tabellen mit Konfigurationsdaten gibt es allerdings besondere Vorkehrungen; siehe [Abschnitt 36.17.3](#36173-erweiterungskonfigurationstabellen). In Produktionsumgebungen ist es im Allgemeinen besser, ein Update-Skript für die Erweiterung zu erstellen, um Änderungen an Mitgliedsobjekten der Erweiterung vorzunehmen.

Das Erweiterungsskript kann mit `GRANT`- und `REVOKE`-Anweisungen Rechte auf Objekten setzen, die zur Erweiterung gehören. Die endgültige Rechtemenge jedes Objekts, sofern Rechte gesetzt wurden, wird im Systemkatalog `pg_init_privs` gespeichert. Wenn `pg_dump` verwendet wird, enthält der Dump den Befehl `CREATE EXTENSION`, gefolgt von den `GRANT`- und `REVOKE`-Anweisungen, die nötig sind, um die Rechte auf den Objekten auf den Stand zum Zeitpunkt des Dumps zu setzen.

PostgreSQL unterstützt derzeit nicht, dass Erweiterungsskripte Anweisungen `CREATE POLICY` oder `SECURITY LABEL` ausgeben. Diese werden erwartungsgemäß nach dem Erstellen der Erweiterung gesetzt. Alle RLS-Richtlinien und Sicherheitslabels auf Erweiterungsobjekten werden in von `pg_dump` erzeugten Dumps aufgenommen.

Der Erweiterungsmechanismus bietet außerdem Vorkehrungen zum Paketieren von Änderungsskripten, die die Definitionen der in einer Erweiterung enthaltenen SQL-Objekte anpassen. Wenn Version 1.1 einer Erweiterung gegenüber Version 1.0 zum Beispiel eine Funktion hinzufügt und den Rumpf einer anderen Funktion ändert, kann der Autor der Erweiterung ein Update-Skript bereitstellen, das genau diese beiden Änderungen vornimmt. Der Befehl `ALTER EXTENSION UPDATE` kann dann verwendet werden, um diese Änderungen anzuwenden und nachzuverfolgen, welche Version der Erweiterung tatsächlich in einer bestimmten Datenbank installiert ist.

Welche Arten von SQL-Objekten Mitglieder einer Erweiterung sein können, ist in der Beschreibung von `ALTER EXTENSION` aufgeführt. Insbesondere können datenbankclusterweite Objekte wie Datenbanken, Rollen und Tablespaces keine Erweiterungsmitglieder sein, da eine Erweiterung nur innerhalb einer Datenbank bekannt ist. Ein Erweiterungsskript darf solche Objekte zwar erzeugen, sie werden dann aber nicht als Teil der Erweiterung verfolgt. Beachten Sie außerdem, dass eine Tabelle zwar Mitglied einer Erweiterung sein kann, ihre untergeordneten Objekte wie Indizes jedoch nicht direkt als Mitglieder der Erweiterung gelten. Ein weiterer wichtiger Punkt ist, dass Schemata zu Erweiterungen gehören können, aber nicht umgekehrt: Eine Erweiterung als solche hat einen unqualifizierten Namen und existiert nicht „innerhalb“ irgendeines Schemas. Die Mitgliedsobjekte der Erweiterung gehören jedoch, soweit es für ihre Objekttypen passend ist, zu Schemata. Es kann angemessen sein oder auch nicht, dass eine Erweiterung die Schemata besitzt, in denen sich ihre Mitgliedsobjekte befinden.

Wenn das Skript einer Erweiterung temporäre Objekte erzeugt, zum Beispiel temporäre Tabellen, werden diese Objekte für den Rest der aktuellen Sitzung als Erweiterungsmitglieder behandelt, aber wie jedes temporäre Objekt am Sitzungsende automatisch gelöscht. Dies ist eine Ausnahme von der Regel, dass Erweiterungsmitgliedsobjekte nicht gelöscht werden können, ohne die gesamte Erweiterung zu löschen.

### 36.17.1. Erweiterungsdateien

Der Befehl `CREATE EXTENSION` stützt sich für jede Erweiterung auf eine Control-Datei. Sie muss denselben Namen wie die Erweiterung mit dem Suffix `.control` haben und im Verzeichnis `SHAREDIR/extension` der Installation liegen. Außerdem muss es mindestens eine SQL-Skriptdatei geben, die dem Namensmuster `extension--version.sql` folgt, zum Beispiel `foo--1.0.sql` für Version 1.0 der Erweiterung `foo`. Standardmäßig liegen auch die Skriptdateien im Verzeichnis `SHAREDIR/extension`; die Control-Datei kann aber ein anderes Verzeichnis für die Skriptdateien angeben.

Zusätzliche Speicherorte für Extension-Control-Dateien können mit dem Parameter `extension_control_path` konfiguriert werden.

Das Dateiformat einer Extension-Control-Datei ist dasselbe wie bei der Datei `postgresql.conf`: eine Liste von Zuweisungen `parameter_name = value`, eine pro Zeile. Leere Zeilen und Kommentare, die mit `#` beginnen, sind erlaubt. Achten Sie darauf, jeden Wert zu quoten, der nicht ein einzelnes Wort oder eine einzelne Zahl ist.

Eine Control-Datei kann die folgenden Parameter setzen:

- `directory` (`string`): Das Verzeichnis, das die SQL-Skriptdateien der Erweiterung enthält. Wenn kein absoluter Pfad angegeben ist, wird der Name relativ zu dem Verzeichnis interpretiert, in dem die Control-Datei gefunden wurde. Standardmäßig werden die Skriptdateien im selben Verzeichnis gesucht, in dem die Control-Datei gefunden wurde.

- `default_version` (`string`): Die Standardversion der Erweiterung, also die Version, die installiert wird, wenn in `CREATE EXTENSION` keine Version angegeben ist. Dies kann zwar weggelassen werden, führt aber dazu, dass `CREATE EXTENSION` fehlschlägt, wenn keine Option `VERSION` erscheint; im Allgemeinen möchte man das also nicht.

- `comment` (`string`): Ein Kommentar, also eine beliebige Zeichenkette, zur Erweiterung. Der Kommentar wird beim anfänglichen Erstellen einer Erweiterung angewendet, aber nicht bei Erweiterungsupdates, da dies von Benutzern hinzugefügte Kommentare überschreiben könnte. Alternativ kann der Kommentar der Erweiterung gesetzt werden, indem in der Skriptdatei ein `COMMENT`-Befehl geschrieben wird.

- `encoding` (`string`): Die Zeichensatzkodierung, die von den Skriptdateien verwendet wird. Dies sollte angegeben werden, wenn die Skriptdateien Nicht-ASCII-Zeichen enthalten. Andernfalls wird angenommen, dass die Dateien in der Datenbankkodierung vorliegen.

- `module_pathname` (`string`): Der Wert dieses Parameters wird für jedes Vorkommen von `MODULE_PATHNAME` in den Skriptdateien eingesetzt. Wenn er nicht gesetzt ist, erfolgt keine Ersetzung. Typischerweise wird er nur auf `shared_library_name` gesetzt, und anschließend wird `MODULE_PATHNAME` in `CREATE FUNCTION`-Befehlen für Funktionen in C-Sprache verwendet, damit die Skriptdateien den Namen der Shared Library nicht fest einbauen müssen.

- `requires` (`string`): Eine Liste der Namen von Erweiterungen, von denen diese Erweiterung abhängt, zum Beispiel `requires = 'foo, bar'`. Diese Erweiterungen müssen installiert sein, bevor diese Erweiterung installiert werden kann.

- `no_relocate` (`string`): Eine Liste der Namen von Erweiterungen, von denen diese Erweiterung abhängt und denen untersagt werden soll, ihre Schemata mit `ALTER EXTENSION ... SET SCHEMA` zu ändern. Das ist nötig, wenn das Skript dieser Erweiterung den Namen des Schemas einer benötigten Erweiterung mit der Syntax `@extschema:name@` auf eine Weise referenziert, die Umbenennungen nicht nachverfolgen kann.

- `superuser` (`boolean`): Wenn dieser Parameter `true` ist, was der Standard ist, können nur Superuser die Erweiterung erstellen oder auf eine neue Version aktualisieren; siehe aber auch `trusted` weiter unten. Wenn er auf `false` gesetzt ist, sind nur die Rechte erforderlich, die zum Ausführen der Befehle im Installations- oder Update-Skript nötig sind. Dies sollte normalerweise auf `true` gesetzt werden, wenn irgendeiner der Skriptbefehle Superuser-Rechte benötigt. Solche Befehle würden ohnehin fehlschlagen, aber es ist benutzerfreundlicher, den Fehler sofort auszugeben.

- `trusted` (`boolean`): Wenn dieser Parameter auf `true` gesetzt ist, was nicht der Standard ist, dürfen einige Nicht-Superuser eine Erweiterung installieren, deren `superuser` auf `true` gesetzt ist. Konkret wird die Installation jedem erlaubt, der das Recht `CREATE` auf der aktuellen Datenbank besitzt. Wenn der Benutzer, der `CREATE EXTENSION` ausführt, kein Superuser ist, aufgrund dieses Parameters aber installieren darf, wird das Installations- oder Update-Skript als Bootstrap-Superuser ausgeführt, nicht als aufrufender Benutzer. Dieser Parameter ist irrelevant, wenn `superuser` auf `false` gesetzt ist. Im Allgemeinen sollte er nicht für Erweiterungen auf `true` gesetzt werden, die Zugriff auf sonst nur Superusern vorbehaltene Fähigkeiten erlauben könnten, etwa Dateisystemzugriff. Außerdem erfordert es erheblichen Zusatzaufwand, eine Erweiterung als vertrauenswürdig zu markieren, weil die Installations- und Update-Skripte sicher geschrieben werden müssen; siehe [Abschnitt 36.17.6](#36176-sicherheitsüberlegungen-für-erweiterungen).

- `relocatable` (`boolean`): Eine Erweiterung ist verschiebbar, wenn es möglich ist, ihre enthaltenen Objekte nach der anfänglichen Erstellung der Erweiterung in ein anderes Schema zu verschieben. Der Standard ist `false`, das heißt, die Erweiterung ist nicht verschiebbar. Weitere Informationen finden Sie in [Abschnitt 36.17.2](#36172-verschiebbarkeit-von-erweiterungen).

- `schema` (`string`): Dieser Parameter kann nur für nicht verschiebbare Erweiterungen gesetzt werden. Er erzwingt, dass die Erweiterung genau in das benannte Schema geladen wird und in kein anderes. Der Parameter `schema` wird nur beim anfänglichen Erstellen einer Erweiterung herangezogen, nicht bei Erweiterungsupdates. Weitere Informationen finden Sie in [Abschnitt 36.17.2](#36172-verschiebbarkeit-von-erweiterungen).

Zusätzlich zur primären Control-Datei `extension.control` kann eine Erweiterung sekundäre Control-Dateien mit Namen der Form `extension--version.control` haben. Falls sie bereitgestellt werden, müssen sie im Skriptdateiverzeichnis liegen. Sekundäre Control-Dateien folgen demselben Format wie die primäre Control-Datei. Alle Parameter, die in einer sekundären Control-Datei gesetzt werden, überschreiben beim Installieren oder Aktualisieren auf diese Version der Erweiterung die primäre Control-Datei. Die Parameter `directory` und `default_version` können jedoch nicht in einer sekundären Control-Datei gesetzt werden.

Die SQL-Skriptdateien einer Erweiterung können beliebige SQL-Befehle enthalten, mit Ausnahme von Transaktionssteuerungsbefehlen (`BEGIN`, `COMMIT` usw.) und Befehlen, die nicht innerhalb eines Transaktionsblocks ausgeführt werden können, etwa `VACUUM`. Der Grund ist, dass die Skriptdateien implizit innerhalb eines Transaktionsblocks ausgeführt werden.

Die SQL-Skriptdateien einer Erweiterung können außerdem Zeilen enthalten, die mit `\echo` beginnen; diese werden vom Erweiterungsmechanismus ignoriert, also wie Kommentare behandelt. Diese Vorkehrung wird häufig verwendet, um einen Fehler auszulösen, wenn die Skriptdatei an `psql` übergeben wird, statt über `CREATE EXTENSION` geladen zu werden; siehe das Beispielskript in [Abschnitt 36.17.7](#36177-erweiterungsbeispiel). Ohne dies könnten Benutzer versehentlich den Inhalt der Erweiterung als „lose“ Objekte statt als Erweiterung laden, ein Zustand, von dem man sich nur etwas mühsam erholt.

Wenn das Erweiterungsskript die Zeichenkette `@extowner@` enthält, wird diese Zeichenkette durch den passend gequoteten Namen des Benutzers ersetzt, der `CREATE EXTENSION` oder `ALTER EXTENSION` aufruft. Typischerweise wird dieses Merkmal von Erweiterungen verwendet, die als `trusted` markiert sind, um ausgewählten Objekten den aufrufenden Benutzer statt des Bootstrap-Superusers als Eigentümer zuzuweisen. Dabei sollte man jedoch vorsichtig sein. Würde man zum Beispiel einer Funktion in C-Sprache einen Nicht-Superuser als Eigentümer zuweisen, entstünde für diesen Benutzer ein Weg zur Rechteausweitung.

Während die Skriptdateien alle Zeichen enthalten können, die von der angegebenen Kodierung erlaubt sind, sollten Control-Dateien nur reines ASCII enthalten, weil PostgreSQL keine Möglichkeit hat zu erkennen, in welcher Kodierung eine Control-Datei vorliegt. In der Praxis ist dies nur ein Problem, wenn Sie Nicht-ASCII-Zeichen im Kommentar der Erweiterung verwenden möchten. Die empfohlene Vorgehensweise in diesem Fall ist, den Control-Datei-Parameter `comment` nicht zu verwenden, sondern den Kommentar mit `COMMENT ON EXTENSION` innerhalb einer Skriptdatei zu setzen.

### 36.17.2. Verschiebbarkeit von Erweiterungen

Benutzer möchten die in einer Erweiterung enthaltenen Objekte oft in ein anderes Schema laden, als es der Autor der Erweiterung vorgesehen hatte. Es gibt drei unterstützte Stufen der Verschiebbarkeit:

- Eine vollständig verschiebbare Erweiterung kann jederzeit in ein anderes Schema verschoben werden, sogar nachdem sie in eine Datenbank geladen wurde. Dies geschieht mit dem Befehl `ALTER EXTENSION SET SCHEMA`, der alle Mitgliedsobjekte automatisch in das neue Schema umbenennt. Normalerweise ist dies nur möglich, wenn die Erweiterung keine internen Annahmen darüber enthält, in welchem Schema sich ihre Objekte befinden. Außerdem müssen sich die Objekte der Erweiterung anfangs alle in einem einzigen Schema befinden; Objekte, die zu keinem Schema gehören, etwa prozedurale Sprachen, werden dabei ignoriert. Kennzeichnen Sie eine vollständig verschiebbare Erweiterung, indem Sie in ihrer Control-Datei `relocatable = true` setzen.

- Eine Erweiterung kann während der Installation verschiebbar sein, danach aber nicht mehr. Das ist typischerweise der Fall, wenn die Skriptdatei der Erweiterung das Zielschema ausdrücklich referenzieren muss, zum Beispiel beim Setzen von `search_path`-Eigenschaften für SQL-Funktionen. Setzen Sie für eine solche Erweiterung in der Control-Datei `relocatable = false`, und verwenden Sie `@extschema@`, um in der Skriptdatei auf das Zielschema zu verweisen. Alle Vorkommen dieser Zeichenkette werden vor dem Ausführen des Skripts durch den tatsächlichen Namen des Zielschemas ersetzt, falls nötig doppelt gequotet. Der Benutzer kann das Zielschema mit der Option `SCHEMA` von `CREATE EXTENSION` setzen.

- Wenn die Erweiterung überhaupt keine Verschiebung unterstützt, setzen Sie in ihrer Control-Datei `relocatable = false` und zusätzlich `schema` auf den Namen des vorgesehenen Zielschemas. Dadurch wird die Verwendung der Option `SCHEMA` von `CREATE EXTENSION` verhindert, es sei denn, sie gibt dasselbe Schema an, das in der Control-Datei benannt ist. Diese Wahl ist typischerweise nötig, wenn die Erweiterung interne Annahmen über ihren Schemanamen enthält, die nicht durch Verwendungen von `@extschema@` ersetzt werden können. Der Ersetzungsmechanismus `@extschema@` ist auch in diesem Fall verfügbar, allerdings nur begrenzt nützlich, weil der Schemaname von der Control-Datei bestimmt wird.

In allen Fällen wird die Skriptdatei anfänglich mit einem `search_path` ausgeführt, der auf das Zielschema zeigt. Das heißt, `CREATE EXTENSION` führt sinngemäß Folgendes aus:

```sql
SET LOCAL search_path TO @extschema@, pg_temp;
```

Dadurch gelangen die von der Skriptdatei erzeugten Objekte in das Zielschema. Die Skriptdatei kann `search_path` ändern, wenn sie möchte, aber das ist im Allgemeinen unerwünscht. Nach Abschluss von `CREATE EXTENSION` wird `search_path` auf seinen vorherigen Wert zurückgesetzt.

Das Zielschema wird durch den Parameter `schema` in der Control-Datei bestimmt, falls dieser angegeben ist, andernfalls durch die Option `SCHEMA` von `CREATE EXTENSION`, falls diese angegeben ist, andernfalls durch das aktuelle Standardschema für Objekterzeugung, also das erste Schema im `search_path` des Aufrufers. Wenn der Control-Datei-Parameter `schema` verwendet wird, wird das Zielschema erzeugt, falls es noch nicht existiert. In den beiden anderen Fällen muss es bereits existieren.

Wenn in der Control-Datei unter `requires` vorausgesetzte Erweiterungen aufgeführt sind, werden deren Zielschemata zur anfänglichen Einstellung von `search_path` hinzugefügt, hinter dem Zielschema der neuen Erweiterung. Dadurch sind ihre Objekte für die Skriptdatei der neuen Erweiterung sichtbar.

Aus Sicherheitsgründen wird `pg_temp` in allen Fällen automatisch an das Ende von `search_path` angehängt.

Obwohl eine nicht verschiebbare Erweiterung Objekte enthalten kann, die über mehrere Schemata verteilt sind, ist es normalerweise wünschenswert, alle für externe Verwendung gedachten Objekte in ein einziges Schema zu legen, das als Zielschema der Erweiterung gilt. Eine solche Anordnung funktioniert bequem mit der Standardeinstellung von `search_path` beim Erstellen abhängiger Erweiterungen.

Wenn eine Erweiterung Objekte referenziert, die zu einer anderen Erweiterung gehören, wird empfohlen, diese Referenzen schemaqualifiziert zu schreiben. Schreiben Sie dazu `@extschema:name@` in die Skriptdatei der Erweiterung, wobei `name` der Name der anderen Erweiterung ist, die in der `requires`-Liste dieser Erweiterung aufgeführt sein muss. Diese Zeichenkette wird durch den Namen des Zielschemas dieser Erweiterung ersetzt, falls nötig doppelt gequotet. Obwohl diese Notation vermeidet, dass feste Annahmen über Schemanamen in die Skriptdatei der Erweiterung eingebaut werden müssen, kann ihre Verwendung den Schemanamen der anderen Erweiterung in den installierten Objekten dieser Erweiterung einbetten. Typischerweise geschieht das, wenn `@extschema:name@` innerhalb eines Zeichenkettenliterals verwendet wird, etwa in einem Funktionsrumpf oder einer `search_path`-Einstellung. In anderen Fällen wird die Objektreferenz beim Parsen auf eine OID reduziert und benötigt keine späteren Nachschlagevorgänge. Wenn der Schemaname der anderen Erweiterung auf diese Weise eingebettet ist, sollten Sie verhindern, dass die andere Erweiterung nach der Installation Ihrer Erweiterung verschoben wird, indem Sie den Namen der anderen Erweiterung zur Liste `no_relocate` Ihrer Erweiterung hinzufügen.

### 36.17.3. Erweiterungskonfigurationstabellen

Einige Erweiterungen enthalten Konfigurationstabellen, die Daten enthalten, welche der Benutzer nach der Installation der Erweiterung hinzufügen oder ändern kann. Normalerweise werden weder die Definition noch der Inhalt einer Tabelle von `pg_dump` ausgegeben, wenn die Tabelle Teil einer Erweiterung ist. Für eine Konfigurationstabelle ist dieses Verhalten jedoch unerwünscht: Alle vom Benutzer vorgenommenen Datenänderungen müssen in Dumps enthalten sein, sonst verhält sich die Erweiterung nach Dump und Wiederherstellung anders.

Um dieses Problem zu lösen, kann die Skriptdatei einer Erweiterung eine von ihr erzeugte Tabelle oder Sequenz als Konfigurationsrelation markieren. Dadurch nimmt `pg_dump` den Inhalt der Tabelle oder Sequenz in Dumps auf, nicht aber ihre Definition. Rufen Sie dazu nach dem Erstellen der Tabelle oder Sequenz die Funktion `pg_extension_config_dump(regclass, text)` auf, zum Beispiel:

```sql
CREATE TABLE my_config (key text, value text);
CREATE SEQUENCE my_config_seq;

SELECT pg_catalog.pg_extension_config_dump('my_config', '');
SELECT pg_catalog.pg_extension_config_dump('my_config_seq', '');
```

Auf diese Weise können beliebig viele Tabellen oder Sequenzen markiert werden. Auch Sequenzen, die mit `serial`- oder `bigserial`-Spalten verbunden sind, können markiert werden.

Wenn das zweite Argument von `pg_extension_config_dump` eine leere Zeichenkette ist, wird der gesamte Inhalt der Tabelle von `pg_dump` ausgegeben. Das ist normalerweise nur korrekt, wenn die Tabelle so, wie sie vom Erweiterungsskript erzeugt wurde, anfänglich leer ist. Wenn die Tabelle eine Mischung aus Anfangsdaten und benutzerbereitgestellten Daten enthält, liefert das zweite Argument von `pg_extension_config_dump` eine `WHERE`-Bedingung, die die auszugebenden Daten auswählt. Zum Beispiel könnte man schreiben:

```sql
CREATE TABLE my_config (key text, value text, standard_entry boolean);

SELECT pg_catalog.pg_extension_config_dump('my_config', 'WHERE NOT standard_entry');
```

und dann sicherstellen, dass `standard_entry` nur in den vom Skript der Erweiterung erzeugten Zeilen `true` ist.

Für Sequenzen hat das zweite Argument von `pg_extension_config_dump` keine Wirkung.

Kompliziertere Situationen, etwa anfänglich bereitgestellte Zeilen, die von Benutzern verändert werden könnten, können durch Trigger auf der Konfigurationstabelle behandelt werden, die sicherstellen, dass geänderte Zeilen korrekt markiert werden.

Sie können die zu einer Konfigurationstabelle gehörende Filterbedingung ändern, indem Sie `pg_extension_config_dump` erneut aufrufen. Dies ist typischerweise in einem Erweiterungs-Update-Skript nützlich. Die einzige Möglichkeit, eine Tabelle nicht mehr als Konfigurationstabelle zu markieren, besteht darin, sie mit `ALTER EXTENSION ... DROP TABLE` von der Erweiterung zu lösen.

Beachten Sie, dass Fremdschlüsselbeziehungen zwischen diesen Tabellen die Reihenfolge bestimmen, in der die Tabellen von `pg_dump` ausgegeben werden. Konkret versucht `pg_dump`, die referenzierte Tabelle vor der referenzierenden Tabelle auszugeben. Da die Fremdschlüsselbeziehungen zum Zeitpunkt von `CREATE EXTENSION` eingerichtet werden, also bevor Daten in die Tabellen geladen werden, werden zirkuläre Abhängigkeiten nicht unterstützt. Wenn zirkuläre Abhängigkeiten bestehen, werden die Daten zwar trotzdem ausgegeben, der Dump kann aber nicht direkt wiederhergestellt werden, und ein Benutzereingriff ist erforderlich.

Sequenzen, die mit `serial`- oder `bigserial`-Spalten verbunden sind, müssen direkt markiert werden, damit ihr Zustand ausgegeben wird. Die Markierung ihrer übergeordneten Relation reicht dafür nicht aus.

### 36.17.4. Erweiterungsupdates

Ein Vorteil des Erweiterungsmechanismus besteht darin, dass er bequeme Wege bereitstellt, Updates an den SQL-Befehlen zu verwalten, die die Objekte einer Erweiterung definieren. Dazu wird jeder veröffentlichten Version des Installationsskripts der Erweiterung ein Versionsname oder eine Versionsnummer zugeordnet. Wenn Sie außerdem möchten, dass Benutzer ihre Datenbanken dynamisch von einer Version auf die nächste aktualisieren können, sollten Sie Update-Skripte bereitstellen, die die notwendigen Änderungen von einer Version zur nächsten vornehmen. Update-Skripte haben Namen nach dem Muster `extension--old_version--target_version.sql`; zum Beispiel enthält `foo--1.0--1.1.sql` die Befehle, um Version 1.0 der Erweiterung `foo` in Version 1.1 zu ändern.

Wenn ein geeignetes Update-Skript vorhanden ist, aktualisiert der Befehl `ALTER EXTENSION UPDATE` eine installierte Erweiterung auf die angegebene neue Version. Das Update-Skript wird in derselben Umgebung ausgeführt, die `CREATE EXTENSION` für Installationsskripte bereitstellt. Insbesondere wird `search_path` auf dieselbe Weise eingerichtet, und alle neuen Objekte, die das Skript erzeugt, werden automatisch zur Erweiterung hinzugefügt. Wenn das Skript Erweiterungsmitgliedsobjekte löscht, werden sie außerdem automatisch von der Erweiterung gelöst.

Wenn eine Erweiterung sekundäre Control-Dateien besitzt, werden für ein Update-Skript die Control-Parameter verwendet, die zur Zielversion, also zur neuen Version, des Skripts gehören.

`ALTER EXTENSION` kann Folgen von Update-Skriptdateien ausführen, um ein angefordertes Update zu erreichen. Wenn zum Beispiel nur `foo--1.0--1.1.sql` und `foo--1.1--2.0.sql` verfügbar sind, wendet `ALTER EXTENSION` sie nacheinander an, wenn ein Update auf Version 2.0 angefordert wird, während aktuell 1.0 installiert ist.

PostgreSQL nimmt nichts über die Eigenschaften von Versionsnamen an. Es weiß zum Beispiel nicht, ob 1.1 auf 1.0 folgt. Es gleicht lediglich die verfügbaren Versionsnamen ab und folgt dem Pfad, bei dem die wenigsten Update-Skripte angewendet werden müssen. Ein Versionsname kann tatsächlich jede Zeichenkette sein, die weder `--` noch ein führendes oder abschließendes `-` enthält.

Manchmal ist es nützlich, „Downgrade“-Skripte bereitzustellen, zum Beispiel `foo--1.1--1.0.sql`, um die mit Version 1.1 verbundenen Änderungen rückgängig machen zu können. Wenn Sie das tun, achten Sie auf die Möglichkeit, dass ein Downgrade-Skript unerwartet angewendet wird, weil es zu einem kürzeren Pfad führt. Der riskante Fall liegt vor, wenn es sowohl ein „Fast-Path“-Update-Skript gibt, das mehrere Versionen überspringt, als auch ein Downgrade-Skript zum Startpunkt dieses Fast-Path. Es könnte weniger Schritte erfordern, erst das Downgrade und dann den Fast-Path anzuwenden, als Version für Version vorwärtszugehen. Wenn das Downgrade-Skript unersetzliche Objekte löscht, führt das zu unerwünschten Ergebnissen.

Um auf unerwartete Update-Pfade zu prüfen, verwenden Sie diesen Befehl:

```sql
SELECT * FROM pg_extension_update_paths('extension_name');
```

Dies zeigt jedes Paar unterschiedlicher bekannter Versionsnamen für die angegebene Erweiterung zusammen mit der Update-Pfadfolge, die genommen würde, um von der Quellversion zur Zielversion zu gelangen, oder `NULL`, wenn kein Update-Pfad verfügbar ist. Der Pfad wird in Textform mit `--` als Trennzeichen angezeigt. Sie können `regexp_split_to_array(path, '--')` verwenden, wenn Sie ein Array-Format bevorzugen.

### 36.17.5. Erweiterungen mit Update-Skripten installieren

Eine Erweiterung, die es schon eine Weile gibt, wird wahrscheinlich in mehreren Versionen existieren, für die der Autor Update-Skripte schreiben muss. Wenn Sie zum Beispiel eine Erweiterung `foo` in den Versionen 1.0, 1.1 und 1.2 veröffentlicht haben, sollte es die Update-Skripte `foo--1.0--1.1.sql` und `foo--1.1--1.2.sql` geben. Vor PostgreSQL 10 musste man außerdem neue Skriptdateien `foo--1.1.sql` und `foo--1.2.sql` erstellen, die die neueren Erweiterungsversionen direkt aufbauen. Andernfalls konnten die neueren Versionen nicht direkt installiert werden, sondern nur durch Installation von 1.0 und anschließendes Aktualisieren. Das war mühsam und redundant, ist nun aber nicht mehr nötig, weil `CREATE EXTENSION` Update-Ketten automatisch folgen kann. Wenn zum Beispiel nur die Skriptdateien `foo--1.0.sql`, `foo--1.0--1.1.sql` und `foo--1.1--1.2.sql` verfügbar sind, wird eine Anforderung, Version 1.2 zu installieren, erfüllt, indem diese drei Skripte nacheinander ausgeführt werden. Die Verarbeitung ist dieselbe, als hätten Sie zuerst 1.0 installiert und dann auf 1.2 aktualisiert. Wie bei `ALTER EXTENSION UPDATE` wird der kürzeste Pfad bevorzugt, wenn mehrere Pfade verfügbar sind. Eine solche Anordnung der Skriptdateien einer Erweiterung kann den Wartungsaufwand für kleine Updates verringern.

Wenn Sie sekundäre, versionsspezifische Control-Dateien mit einer Erweiterung verwenden, die in diesem Stil gepflegt wird, beachten Sie, dass jede Version eine Control-Datei benötigt, selbst wenn sie kein eigenständiges Installationsskript besitzt. Diese Control-Datei bestimmt, wie das implizite Update auf diese Version ausgeführt wird. Wenn zum Beispiel `foo--1.0.control` `requires = 'bar'` angibt, die anderen Control-Dateien von `foo` aber nicht, wird die Abhängigkeit der Erweiterung von `bar` beim Aktualisieren von 1.0 auf eine andere Version entfernt.

### 36.17.6. Sicherheitsüberlegungen für Erweiterungen

Weit verbreitete Erweiterungen sollten wenig über die Datenbank annehmen, in der sie installiert sind. Daher ist es angemessen, von einer Erweiterung bereitgestellte Funktionen in einem sicheren Stil zu schreiben, der nicht durch Angriffe über den Suchpfad kompromittiert werden kann.

Eine Erweiterung, deren Eigenschaft `superuser` auf `true` gesetzt ist, muss auch Sicherheitsrisiken für die Aktionen berücksichtigen, die innerhalb ihrer Installations- und Update-Skripte ausgeführt werden. Für einen böswilligen Benutzer ist es nicht besonders schwierig, Trojaner-Objekte zu erzeugen, die die spätere Ausführung eines unvorsichtig geschriebenen Erweiterungsskripts kompromittieren und diesem Benutzer erlauben, Superuser-Rechte zu erlangen.

Wenn eine Erweiterung als `trusted` markiert ist, kann ihr Installationsschema vom installierenden Benutzer gewählt werden, der absichtlich ein unsicheres Schema verwenden könnte, in der Hoffnung, Superuser-Rechte zu erlangen. Eine vertrauenswürdige Erweiterung ist daher aus Sicherheitssicht besonders exponiert, und alle ihre Skriptbefehle müssen sorgfältig geprüft werden, um sicherzustellen, dass keine Kompromittierung möglich ist.

Hinweise zum sicheren Schreiben von Funktionen finden Sie unten in [Abschnitt 36.17.6.1](#361761-sicherheitsüberlegungen-für-erweiterungsfunktionen), Hinweise zum sicheren Schreiben von Installationsskripten in [Abschnitt 36.17.6.2](#361762-sicherheitsüberlegungen-für-erweiterungsskripte).

#### 36.17.6.1. Sicherheitsüberlegungen für Erweiterungsfunktionen

Von Erweiterungen bereitgestellte Funktionen in SQL-Sprache und PL-Sprachen sind bei ihrer Ausführung anfällig für Angriffe über den Suchpfad, da diese Funktionen zur Ausführungszeit geparst werden, nicht zur Erstellungszeit.

Die Referenzseite zu `CREATE FUNCTION` enthält Hinweise dazu, wie `SECURITY DEFINER`-Funktionen sicher geschrieben werden. Es ist gute Praxis, diese Techniken auf jede von einer Erweiterung bereitgestellte Funktion anzuwenden, da die Funktion möglicherweise von einem Benutzer mit hohen Rechten aufgerufen wird.

Wenn Sie `search_path` nicht so setzen können, dass er nur sichere Schemata enthält, nehmen Sie an, dass jeder unqualifizierte Name zu einem Objekt aufgelöst werden könnte, das ein böswilliger Benutzer definiert hat. Hüten Sie sich vor Konstrukten, die implizit von `search_path` abhängen. Zum Beispiel wählen `IN` und `CASE expression WHEN` immer einen Operator über den Suchpfad aus. Verwenden Sie stattdessen `OPERATOR(schema.=) ANY` und `CASE WHEN expression`.

Eine Allzweckerweiterung sollte normalerweise nicht annehmen, dass sie in ein sicheres Schema installiert wurde. Das bedeutet, dass selbst schemaqualifizierte Referenzen auf ihre eigenen Objekte nicht völlig risikofrei sind. Wenn die Erweiterung zum Beispiel eine Funktion `myschema.myfunc(bigint)` definiert hat, könnte ein Aufruf wie `myschema.myfunc(42)` von einer feindlichen Funktion `myschema.myfunc(integer)` abgefangen werden. Achten Sie darauf, dass die Datentypen von Funktions- und Operatorparametern exakt den deklarierten Argumenttypen entsprechen; verwenden Sie bei Bedarf explizite Typumwandlungen.

#### 36.17.6.2. Sicherheitsüberlegungen für Erweiterungsskripte

Ein Installations- oder Update-Skript einer Erweiterung sollte so geschrieben sein, dass es gegen Angriffe über den Suchpfad geschützt ist, die während der Ausführung des Skripts auftreten können. Wenn eine Objektreferenz im Skript auf ein anderes Objekt aufgelöst werden kann, als der Skriptautor beabsichtigt hat, kann eine Kompromittierung sofort auftreten oder später, wenn das falsch definierte Erweiterungsobjekt verwendet wird.

DDL-Befehle wie `CREATE FUNCTION` und `CREATE OPERATOR CLASS` sind im Allgemeinen sicher, aber Vorsicht ist bei jedem Befehl geboten, der einen allgemeinen Ausdruck als Bestandteil hat. Zum Beispiel muss `CREATE VIEW` geprüft werden, ebenso ein `DEFAULT`-Ausdruck in `CREATE FUNCTION`.

Manchmal muss ein Erweiterungsskript allgemeines SQL ausführen, etwa um Kataloganpassungen vorzunehmen, die per DDL nicht möglich sind. Achten Sie darauf, solche Befehle mit einem sicheren `search_path` auszuführen; vertrauen Sie nicht darauf, dass der von `CREATE/ALTER EXTENSION` bereitgestellte Pfad sicher ist. Die beste Praxis besteht darin, `search_path` vorübergehend auf `pg_catalog, pg_temp` zu setzen und Referenzen auf das Installationsschema der Erweiterung dort ausdrücklich einzufügen, wo sie benötigt werden. Diese Praxis kann auch beim Erstellen von Views hilfreich sein. Beispiele finden sich in den `contrib`-Modulen der PostgreSQL-Quellcodedistribution.

Sichere Referenzen zwischen Erweiterungen erfordern typischerweise eine Schemaqualifizierung der Namen der Objekte der anderen Erweiterung mit der Syntax `@extschema:name@`, zusätzlich zum sorgfältigen Abgleich der Argumenttypen für Funktionen und Operatoren.

### 36.17.7. Erweiterungsbeispiel

Hier ist ein vollständiges Beispiel für eine reine SQL-Erweiterung: ein zweielementiger zusammengesetzter Typ, der Werte beliebigen Typs in seinen Feldern speichern kann, die „k“ und „v“ heißen. Nicht-Text-Werte werden zum Speichern automatisch nach `text` koerziert.

Die Skriptdatei `pair--1.0.sql` sieht so aus:

```sql
-- Fehler ausgeben, wenn das Skript in psql eingelesen wird,
-- statt via CREATE EXTENSION geladen zu werden.
\echo Use "CREATE EXTENSION pair" to load this file. \quit

CREATE TYPE pair AS ( k text, v text );

CREATE FUNCTION pair(text, text)
RETURNS pair LANGUAGE SQL
AS 'SELECT ROW($1, $2)::@extschema@.pair;';

CREATE OPERATOR ~> (LEFTARG = text, RIGHTARG = text, FUNCTION = pair);

-- "SET search_path" ist leicht korrekt zu verwenden, aber
-- qualifizierte Namen sind performanter.
CREATE FUNCTION lower(pair)
RETURNS pair LANGUAGE SQL
AS 'SELECT ROW(lower($1.k), lower($1.v))::@extschema@.pair;'
SET search_path = pg_temp;

CREATE FUNCTION pair_concat(pair, pair)
RETURNS pair LANGUAGE SQL
AS 'SELECT ROW($1.k OPERATOR(pg_catalog.||) $2.k,
               $1.v OPERATOR(pg_catalog.||) $2.v)::@extschema@.pair;';
```

Die Control-Datei `pair.control` sieht so aus:

```text
# pair extension
comment = 'A key/value pair data type'
default_version = '1.0'
# cannot be relocatable because of use of @extschema@
relocatable = false
```

Sie brauchen kaum ein Makefile, um diese beiden Dateien in das richtige Verzeichnis zu installieren, könnten aber eines mit folgendem Inhalt verwenden:

```makefile
EXTENSION = pair
DATA = pair--1.0.sql

PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

Dieses Makefile stützt sich auf PGXS, das in [Abschnitt 36.18](#3618-infrastruktur-zum-bauen-von-erweiterungen) beschrieben wird. Der Befehl `make install` installiert die Control- und Skriptdateien in das richtige Verzeichnis, wie es von `pg_config` gemeldet wird.

Sobald die Dateien installiert sind, verwenden Sie den Befehl `CREATE EXTENSION`, um die Objekte in eine bestimmte Datenbank zu laden.

## 36.18. Infrastruktur zum Bauen von Erweiterungen

Wenn Sie darüber nachdenken, Ihre PostgreSQL-Erweiterungsmodule zu verteilen, kann es recht schwierig sein, ein portables Build-System dafür einzurichten. Deshalb stellt die PostgreSQL-Installation eine Build-Infrastruktur für Erweiterungen bereit, genannt PGXS, damit einfache Erweiterungsmodule einfach gegen einen bereits installierten Server gebaut werden können. PGXS ist hauptsächlich für Erweiterungen gedacht, die C-Code enthalten, kann aber auch für reine SQL-Erweiterungen verwendet werden. Beachten Sie, dass PGXS nicht als universelles Build-System-Framework gedacht ist, mit dem beliebige Software gebaut werden kann, die an PostgreSQL angebunden ist. Es automatisiert lediglich gängige Build-Regeln für einfache Server-Erweiterungsmodule. Für kompliziertere Pakete müssen Sie möglicherweise ein eigenes Build-System schreiben.

Um die PGXS-Infrastruktur für Ihre Erweiterung zu verwenden, müssen Sie ein einfaches Makefile schreiben. Im Makefile müssen Sie einige Variablen setzen und das globale PGXS-Makefile einbinden. Hier ist ein Beispiel, das ein Erweiterungsmodul namens `isbn_issn` baut, bestehend aus einer Shared Library mit etwas C-Code, einer Extension-Control-Datei, einem SQL-Skript, einer Include-Datei, die nur benötigt wird, wenn andere Module ohne den Umweg über SQL auf die Erweiterungsfunktionen zugreifen müssen, und einer Dokumentationstextdatei:

```makefile
MODULES = isbn_issn

EXTENSION = isbn_issn
DATA = isbn_issn--1.0.sql
DOCS = README.isbn_issn
HEADERS_isbn_issn = isbn_issn.h

PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

Die letzten drei Zeilen sollten immer gleich sein. Weiter oben in der Datei weisen Sie Variablen zu oder fügen eigene Make-Regeln hinzu.

Setzen Sie eine dieser drei Variablen, um anzugeben, was gebaut wird:

- `MODULES`: Liste von Shared-Library-Objekten, die aus Quelldateien mit demselben Stamm gebaut werden; nehmen Sie keine Library-Suffixe in diese Liste auf.

- `MODULE_big`: Eine Shared Library, die aus mehreren Quelldateien gebaut wird; listen Sie Objektdateien in `OBJS` auf.

- `PROGRAM`: Ein ausführbares Programm, das gebaut werden soll; listen Sie Objektdateien in `OBJS` auf.

Die folgenden Variablen können ebenfalls gesetzt werden:

- `EXTENSION`: Name oder Namen der Erweiterungen. Für jeden Namen müssen Sie eine Datei `extension.control` bereitstellen, die nach `prefix/share/extension` installiert wird.

- `MODULEDIR`: Unterverzeichnis von `prefix/share`, in das `DATA`- und `DOCS`-Dateien installiert werden sollen. Wenn nicht gesetzt, ist der Standard `extension`, falls `EXTENSION` gesetzt ist, sonst `contrib`.

- `DATA`: Beliebige Dateien, die nach `prefix/share/$MODULEDIR` installiert werden sollen.

- `DATA_built`: Beliebige Dateien, die nach `prefix/share/$MODULEDIR` installiert werden sollen und zuerst gebaut werden müssen.

- `DATA_TSEARCH`: Beliebige Dateien, die unter `prefix/share/tsearch_data` installiert werden sollen.

- `DOCS`: Beliebige Dateien, die unter `prefix/doc/$MODULEDIR` installiert werden sollen.

- `HEADERS`, `HEADERS_built`: Dateien, die unter `prefix/include/server/$MODULEDIR/$MODULE_big` installiert werden, bei `HEADERS_built` nachdem sie optional gebaut wurden. Anders als `DATA_built` werden Dateien in `HEADERS_built` vom Ziel `clean` nicht entfernt. Wenn Sie möchten, dass sie entfernt werden, fügen Sie sie zusätzlich zu `EXTRA_CLEAN` hinzu oder ergänzen eigene Regeln dafür.

- `HEADERS_$MODULE`, `HEADERS_built_$MODULE`: Dateien, die unter `prefix/include/server/$MODULEDIR/$MODULE` installiert werden, nach dem Bauen, wenn angegeben. Dabei muss `$MODULE` ein Modulname sein, der in `MODULES` oder `MODULE_big` verwendet wird. Anders als `DATA_built` werden Dateien in `HEADERS_built_$MODULE` vom Ziel `clean` nicht entfernt. Wenn Sie möchten, dass sie entfernt werden, fügen Sie sie zusätzlich zu `EXTRA_CLEAN` hinzu oder ergänzen eigene Regeln dafür.

Es ist zulässig, beide Variablen für dasselbe Modul oder in beliebiger Kombination zu verwenden, außer wenn Sie in der Liste `MODULES` zwei Modulnamen haben, die sich nur durch das Vorhandensein eines Präfixes `built_` unterscheiden; das würde Mehrdeutigkeit verursachen. In diesem hoffentlich unwahrscheinlichen Fall sollten Sie nur die Variablen `HEADERS_built_$MODULE` verwenden.

- `SCRIPTS`: Skriptdateien, keine Binärdateien, die nach `prefix/bin` installiert werden sollen.

- `SCRIPTS_built`: Skriptdateien, keine Binärdateien, die nach `prefix/bin` installiert werden sollen und zuerst gebaut werden müssen.

- `REGRESS`: Liste von Regressionstestfällen ohne Suffix; siehe unten.

- `REGRESS_OPTS`: Zusätzliche Optionen, die an `pg_regress` übergeben werden.

- `ISOLATION`: Liste von Isolationstestfällen; weitere Details siehe unten.

- `ISOLATION_OPTS`: Zusätzliche Optionen, die an `pg_isolation_regress` übergeben werden.

- `TAP_TESTS`: Schalter, der festlegt, ob TAP-Tests ausgeführt werden müssen; siehe unten.

- `NO_INSTALL`: Definiert kein Ziel `install`; nützlich für Testmodule, deren Build-Produkte nicht installiert werden müssen.

- `NO_INSTALLCHECK`: Definiert kein Ziel `installcheck`; nützlich zum Beispiel, wenn Tests besondere Konfiguration benötigen oder `pg_regress` nicht verwenden.

- `EXTRA_CLEAN`: Zusätzliche Dateien, die bei `make clean` entfernt werden.

- `PG_CPPFLAGS`: Wird `CPPFLAGS` vorangestellt.

- `PG_CFLAGS`: Wird an `CFLAGS` angehängt.

- `PG_CXXFLAGS`: Wird an `CXXFLAGS` angehängt.

- `PG_LDFLAGS`: Wird `LDFLAGS` vorangestellt.

- `PG_LIBS`: Wird der Linkzeile von `PROGRAM` hinzugefügt.

- `SHLIB_LINK`: Wird der Linkzeile von `MODULE_big` hinzugefügt.

- `PG_CONFIG`: Pfad zum Programm `pg_config` der PostgreSQL-Installation, gegen die gebaut werden soll; typischerweise einfach `pg_config`, um das erste im `PATH` gefundene Programm zu verwenden.

Legen Sie dieses Makefile als `Makefile` in das Verzeichnis, das Ihre Erweiterung enthält. Dann können Sie `make` zum Kompilieren und anschließend `make install` zum Installieren Ihres Moduls ausführen. Standardmäßig wird die Erweiterung für die PostgreSQL-Installation kompiliert und installiert, die dem ersten im `PATH` gefundenen Programm `pg_config` entspricht. Sie können eine andere Installation verwenden, indem Sie `PG_CONFIG` auf deren `pg_config`-Programm zeigen lassen, entweder innerhalb des Makefiles oder auf der `make`-Befehlszeile.

Sie können ein separates Verzeichnispräfix auswählen, in das die Dateien Ihrer Erweiterung installiert werden, indem Sie die Make-Variable `prefix` beim Ausführen von `make install` setzen:

```sh
make install prefix=/usr/local/postgresql
```

Dies installiert die Extension-Control- und SQL-Dateien nach `/usr/local/postgresql/share` und die Shared Modules nach `/usr/local/postgresql/lib`. Wenn das Präfix nicht die Zeichenketten `postgres` oder `pgsql` enthält, etwa:

```sh
make install prefix=/usr/local/extras
```

dann wird `postgresql` an die Verzeichnisnamen angehängt. Die Control- und SQL-Dateien werden also nach `/usr/local/extras/share/postgresql/extension` installiert und die Shared Modules nach `/usr/local/extras/lib/postgresql`. In beiden Fällen müssen Sie `extension_control_path` und `dynamic_library_path` setzen, damit der PostgreSQL-Server die Dateien finden kann:

```text
extension_control_path = '/usr/local/extras/share/postgresql:$system'
dynamic_library_path = '/usr/local/extras/lib/postgresql:$libdir'
```

Sie können `make` auch in einem Verzeichnis außerhalb des Quellbaums Ihrer Erweiterung ausführen, wenn Sie das Build-Verzeichnis getrennt halten möchten. Dieses Verfahren wird auch VPATH-Build genannt. So geht es:

```sh
mkdir build_dir
cd build_dir
make -f /path/to/extension/source/tree/Makefile
make -f /path/to/extension/source/tree/Makefile install
```

Alternativ können Sie ein Verzeichnis für einen VPATH-Build ähnlich einrichten, wie es für den Kerncode gemacht wird. Eine Möglichkeit dafür ist das Kernskript `config/prep_buildtree`. Nachdem dies erledigt ist, können Sie bauen, indem Sie die Make-Variable `VPATH` so setzen:

```sh
make VPATH=/path/to/extension/source/tree
make VPATH=/path/to/extension/source/tree install
```

Dieses Verfahren kann mit einer größeren Vielfalt von Verzeichnislayouts funktionieren.

Die in der Variablen `REGRESS` aufgeführten Skripte werden für Regressionstests Ihres Moduls verwendet. Sie können mit `make installcheck` aufgerufen werden, nachdem `make install` ausgeführt wurde. Dafür muss ein PostgreSQL-Server laufen. Die in `REGRESS` aufgeführten Skriptdateien müssen in einem Unterverzeichnis namens `sql/` im Verzeichnis Ihrer Erweiterung liegen. Diese Dateien müssen die Erweiterung `.sql` haben, die nicht in die Liste `REGRESS` im Makefile aufgenommen werden darf. Für jeden Test sollte es außerdem eine Datei mit der erwarteten Ausgabe in einem Unterverzeichnis namens `expected/` geben, mit demselben Stamm und der Erweiterung `.out`. `make installcheck` führt jedes Testskript mit `psql` aus und vergleicht die resultierende Ausgabe mit der passenden Erwartungsdatei. Alle Unterschiede werden im Format `diff -c` in die Datei `regression.diffs` geschrieben. Beachten Sie, dass der Versuch, einen Test auszuführen, dessen Erwartungsdatei fehlt, als „trouble“ gemeldet wird; stellen Sie also sicher, dass alle Erwartungsdateien vorhanden sind.

Die in der Variablen `ISOLATION` aufgeführten Skripte werden für Tests verwendet, die das Verhalten gleichzeitiger Sitzungen mit Ihrem Modul belasten. Sie können mit `make installcheck` aufgerufen werden, nachdem `make install` ausgeführt wurde. Dafür muss ein PostgreSQL-Server laufen. Die in `ISOLATION` aufgeführten Skriptdateien müssen in einem Unterverzeichnis namens `specs/` im Verzeichnis Ihrer Erweiterung liegen. Diese Dateien müssen die Erweiterung `.spec` haben, die nicht in die Liste `ISOLATION` im Makefile aufgenommen werden darf. Für jeden Test sollte es außerdem eine Datei mit der erwarteten Ausgabe in einem Unterverzeichnis namens `expected/` geben, mit demselben Stamm und der Erweiterung `.out`. `make installcheck` führt jedes Testskript aus und vergleicht die resultierende Ausgabe mit der passenden Erwartungsdatei. Alle Unterschiede werden im Format `diff -c` in die Datei `output_iso/regression.diffs` geschrieben. Beachten Sie, dass der Versuch, einen Test auszuführen, dessen Erwartungsdatei fehlt, als „trouble“ gemeldet wird; stellen Sie also sicher, dass alle Erwartungsdateien vorhanden sind.

`TAP_TESTS` aktiviert die Verwendung von TAP-Tests. Daten aus jedem Lauf liegen in einem Unterverzeichnis namens `tmp_check/`. Weitere Details finden Sie auch in [Abschnitt 31.4](31_Regressionstests.md#314-taptests).

> **Tipp**
>
> Die einfachste Möglichkeit, die Erwartungsdateien zu erzeugen, besteht darin, leere Dateien anzulegen und dann einen Testlauf auszuführen, der natürlich Unterschiede melden wird. Prüfen Sie die tatsächlichen Ergebnisdateien im Verzeichnis `results/` für Tests in `REGRESS` oder im Verzeichnis `output_iso/results/` für Tests in `ISOLATION`, und kopieren Sie sie anschließend nach `expected/`, wenn sie dem entsprechen, was Sie vom Test erwarten.
