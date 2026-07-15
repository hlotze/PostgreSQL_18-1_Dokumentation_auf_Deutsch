# 10. Typumwandlung

SQL-Anweisungen können absichtlich oder unabsichtlich erfordern, dass verschiedene Datentypen im selben Ausdruck gemischt werden. PostgreSQL verfügt über umfangreiche Möglichkeiten, Ausdrücke mit gemischten Typen auszuwerten.

In vielen Fällen muss ein Benutzer die Einzelheiten des Typumwandlungsmechanismus nicht kennen. Implizite Umwandlungen durch PostgreSQL können jedoch die Ergebnisse einer Abfrage beeinflussen. Wenn nötig, lassen sich diese Ergebnisse durch explizite Typumwandlung steuern.

Dieses Kapitel stellt die Mechanismen und Konventionen der Typumwandlung in PostgreSQL vor. Weitere Informationen zu bestimmten Datentypen sowie zulässigen Funktionen und Operatoren finden Sie in den entsprechenden Abschnitten von [Kapitel 8](08_Datentypen.md) und [Kapitel 9](09_Funktionen_und_Operatoren.md).

## 10.1. Überblick

SQL ist eine stark typisierte Sprache. Das heißt, jedes Datenelement hat einen zugeordneten Datentyp, der sein Verhalten und seine zulässige Verwendung bestimmt. PostgreSQL besitzt ein erweiterbares Typsystem, das allgemeiner und flexibler ist als das vieler anderer SQL-Implementierungen. Deshalb wird das meiste Typumwandlungsverhalten in PostgreSQL durch allgemeine Regeln gesteuert und nicht durch Ad-hoc-Heuristiken. Dadurch lassen sich Ausdrücke mit gemischten Typen auch bei benutzerdefinierten Typen verwenden.

Der Scanner/Parser von PostgreSQL teilt lexikalische Elemente in fünf grundlegende Kategorien ein: Ganzzahlen, nicht ganzzahlige Zahlen, Zeichenketten, Bezeichner und Schlüsselwörter. Konstanten der meisten nicht numerischen Typen werden zunächst als Zeichenketten klassifiziert. Die SQL-Sprachdefinition erlaubt es, Typnamen zusammen mit Zeichenketten anzugeben, und PostgreSQL kann diesen Mechanismus nutzen, um den Parser auf den richtigen Weg zu bringen. Zum Beispiel hat die Abfrage

```sql
SELECT text 'Origin' AS "label", point '(0,0)' AS "value";
```

die Ausgabe

```text
 label | value
-------+-------
 Origin | (0,0)
(1 row)
```

Sie enthält zwei Literal-Konstanten der Typen `text` und `point`. Wenn für ein Zeichenkettenliteral kein Typ angegeben wird, bekommt es zunächst den Platzhaltertyp `unknown`, der in späteren Phasen wie unten beschrieben aufgelöst wird.

Es gibt vier grundlegende SQL-Konstrukte, die im PostgreSQL-Parser jeweils eigene Regeln für die Typumwandlung benötigen:

**Funktionsaufrufe**

Ein großer Teil des PostgreSQL-Typsystems ist um eine reichhaltige Menge von Funktionen herum aufgebaut. Funktionen können ein oder mehrere Argumente haben. Da PostgreSQL das Überladen von Funktionen erlaubt, identifiziert der Funktionsname allein die aufzurufende Funktion nicht eindeutig; der Parser muss die passende Funktion anhand der Datentypen der übergebenen Argumente auswählen.

**Operatoren**

PostgreSQL erlaubt Ausdrücke mit Präfixoperatoren mit einem Argument sowie Infixoperatoren mit zwei Argumenten. Wie Funktionen können auch Operatoren überladen sein, sodass dasselbe Problem der Auswahl des richtigen Operators entsteht.

**Wertspeicherung**

SQL-Anweisungen `INSERT` und `UPDATE` legen die Ergebnisse von Ausdrücken in einer Tabelle ab. Die Ausdrücke in der Anweisung müssen den Typen der Zielspalten zugeordnet und gegebenenfalls in diese Typen umgewandelt werden.

**`UNION`, `CASE` und verwandte Konstrukte**

Da alle Abfrageergebnisse einer mit `UNION` verbundenen `SELECT`-Anweisung in einem einzigen Satz von Spalten erscheinen müssen, müssen die Typen der Ergebnisse jeder `SELECT`-Klausel einander angepasst und in einen einheitlichen Satz umgewandelt werden. Ebenso müssen die Ergebnisausdrücke eines `CASE`-Konstrukts in einen gemeinsamen Typ umgewandelt werden, damit der `CASE`-Ausdruck insgesamt einen bekannten Ausgabetyp hat. Einige andere Konstrukte, etwa `ARRAY[]` sowie die Funktionen `GREATEST` und `LEAST`, benötigen auf ähnliche Weise die Bestimmung eines gemeinsamen Typs für mehrere Teilausdrücke.

Die Systemkataloge speichern Informationen darüber, welche Umwandlungen oder Casts zwischen welchen Datentypen existieren und wie diese Umwandlungen ausgeführt werden. Benutzer können mit dem Befehl `CREATE CAST` weitere Casts hinzufügen. Dies geschieht normalerweise im Zusammenhang mit der Definition neuer Datentypen. Die Menge der Casts zwischen eingebauten Typen wurde sorgfältig gestaltet und sollte möglichst nicht verändert werden.

Eine zusätzliche Heuristik des Parsers ermöglicht eine bessere Bestimmung des passenden Cast-Verhaltens innerhalb von Typgruppen, die implizite Casts besitzen. Datentypen sind in mehrere grundlegende Typkategorien eingeteilt, unter anderem `boolean`, numerisch, Zeichenkette, Bit-String, Datum/Uhrzeit, Zeitspanne, geometrisch, Netzwerk und benutzerdefiniert. Eine Liste finden Sie in [Tabelle 52.65](52_Systemkataloge.md#5264-pgtype); beachten Sie jedoch, dass auch eigene Typkategorien erstellt werden können. Innerhalb jeder Kategorie kann es einen oder mehrere bevorzugte Typen geben, die bei mehreren möglichen Typen bevorzugt werden. Durch sorgfältige Auswahl bevorzugter Typen und verfügbarer impliziter Casts lässt sich sicherstellen, dass mehrdeutige Ausdrücke, also Ausdrücke mit mehreren möglichen Parserlösungen, auf nützliche Weise aufgelöst werden können.

Alle Regeln für Typumwandlung folgen mehreren Grundsätzen:

- Implizite Umwandlungen sollten niemals überraschende oder unvorhersehbare Ergebnisse haben.

- Wenn eine Abfrage keine implizite Typumwandlung benötigt, sollte dadurch kein zusätzlicher Aufwand im Parser oder Executor entstehen. Ist eine Abfrage also wohlgeformt und stimmen die Typen bereits überein, soll sie ohne zusätzliche Parserzeit und ohne unnötige implizite Umwandlungsaufrufe in der Abfrage ausgeführt werden.

- Wenn eine Abfrage normalerweise eine implizite Umwandlung für eine Funktion benötigt und der Benutzer anschließend eine neue Funktion mit den passenden Argumenttypen definiert, sollte der Parser diese neue Funktion verwenden und nicht mehr implizit umwandeln, um die alte Funktion zu benutzen.

## 10.2. Operatoren

Der konkrete Operator, auf den sich ein Operatorausdruck bezieht, wird mit dem folgenden Verfahren bestimmt. Beachten Sie, dass dieses Verfahren indirekt von der Präzedenz der beteiligten Operatoren beeinflusst wird, weil diese festlegt, welche Teilausdrücke als Eingaben welcher Operatoren gelten. Weitere Informationen finden Sie in [Abschnitt 4.1.6](04_SQL_Syntax.md#416-operatorpräzedenz).

**Typauflösung für Operatoren**

1. Wählen Sie die zu betrachtenden Operatoren aus dem Systemkatalog `pg_operator` aus. Wenn ein nicht schemaqualifizierter Operatorname verwendet wurde, was der übliche Fall ist, werden diejenigen Operatoren betrachtet, die den passenden Namen und die passende Argumentanzahl haben und im aktuellen Suchpfad sichtbar sind (siehe [Abschnitt 5.10.3](05_Datendefinition.md#5103-der-schemasuchpfad)). Wenn ein qualifizierter Operatorname angegeben wurde, werden nur Operatoren im angegebenen Schema betrachtet.

2. Optional: Wenn der Suchpfad mehrere Operatoren mit identischen Argumenttypen findet, wird nur derjenige berücksichtigt, der im Pfad zuerst erscheint. Operatoren mit unterschiedlichen Argumenttypen werden unabhängig von ihrer Position im Suchpfad gleich behandelt.

3. Prüfen Sie, ob es einen Operator gibt, der genau die Eingabeargumenttypen akzeptiert. Wenn ein solcher Operator existiert, wird er verwendet; in der betrachteten Operatormenge kann es nur eine exakte Übereinstimmung geben. Fehlt eine exakte Übereinstimmung, entsteht bei einem Aufruf über einen qualifizierten Namen ein Sicherheitsrisiko, falls der Operator in einem Schema gefunden wird, in dem nicht vertrauenswürdige Benutzer Objekte erstellen dürfen. In solchen Situationen sollten Argumente gecastet werden, um eine exakte Übereinstimmung zu erzwingen.

4. Optional: Wenn ein Argument eines binären Operatoraufrufs den Typ `unknown` hat, nehmen Sie für diese Prüfung an, dass es denselben Typ wie das andere Argument hat. Aufrufe mit zwei `unknown`-Eingaben oder ein Präfixoperator mit einer `unknown`-Eingabe finden in diesem Schritt niemals eine Übereinstimmung.

5. Optional: Wenn ein Argument eines binären Operatoraufrufs den Typ `unknown` und das andere einen Domain-Typ hat, prüfen Sie anschließend, ob es einen Operator gibt, der auf beiden Seiten genau den Basistyp der Domain akzeptiert. Wenn ja, verwenden Sie ihn.

6. Suchen Sie nach der besten Übereinstimmung.

7. Verwerfen Sie Kandidatenoperatoren, deren Eingabetypen nicht übereinstimmen und nicht durch implizite Umwandlung passend gemacht werden können. `unknown`-Literale gelten hierfür als in alles umwandelbar. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

8. Wenn ein Eingabeargument ein Domain-Typ ist, behandeln Sie es in allen folgenden Schritten als den Basistyp der Domain. Dadurch verhalten sich Domains bei der Auflösung mehrdeutiger Operatoren wie ihre Basistypen.

9. Durchlaufen Sie alle Kandidaten und behalten Sie diejenigen mit den meisten exakten Übereinstimmungen der Eingabetypen. Wenn keiner exakte Übereinstimmungen hat, behalten Sie alle Kandidaten. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

10. Durchlaufen Sie alle Kandidaten und behalten Sie diejenigen, die an den meisten Positionen, an denen Typumwandlung erforderlich wäre, bevorzugte Typen der jeweiligen Typkategorie akzeptieren. Wenn keiner bevorzugte Typen akzeptiert, behalten Sie alle Kandidaten. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

11. Wenn Eingabeargumente vom Typ `unknown` vorhanden sind, prüfen Sie die Typkategorien, die von den verbleibenden Kandidaten an diesen Argumentpositionen akzeptiert werden. Wählen Sie an jeder Position die Kategorie Zeichenkette, wenn irgendein Kandidat diese Kategorie akzeptiert; diese Bevorzugung von Zeichenketten ist sinnvoll, weil ein Literal unbekannten Typs wie eine Zeichenkette aussieht. Andernfalls wählen Sie die Kategorie, wenn alle verbleibenden Kandidaten dieselbe Typkategorie akzeptieren; sonst schlägt die Auflösung fehl, weil die richtige Wahl ohne weitere Hinweise nicht bestimmt werden kann. Verwerfen Sie nun Kandidaten, die die gewählte Typkategorie nicht akzeptieren. Wenn außerdem ein Kandidat einen bevorzugten Typ in dieser Kategorie akzeptiert, verwerfen Sie Kandidaten, die für dieses Argument nicht bevorzugte Typen akzeptieren. Wenn nach diesen Prüfungen keine Kandidaten übrig bleiben, behalten Sie alle Kandidaten. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

12. Wenn es sowohl Argumente vom Typ `unknown` als auch Argumente bekannten Typs gibt und alle bekannten Argumente denselben Typ haben, nehmen Sie an, dass auch die `unknown`-Argumente diesen Typ haben, und prüfen Sie, welche Kandidaten diesen Typ an den `unknown`-Argumentpositionen akzeptieren können. Wenn genau ein Kandidat diese Prüfung besteht, wird er verwendet. Andernfalls schlägt die Auflösung fehl.

> **Hinweis** Das Sicherheitsrisiko entsteht nicht bei einem nicht schemaqualifizierten Namen, weil ein Suchpfad, der Schemas enthält, in denen nicht vertrauenswürdige Benutzer Objekte erstellen dürfen, ohnehin kein sicheres Schema-Verwendungsmuster ist.

Es folgen einige Beispiele.

**Beispiel 10.1. Typauflösung für den Quadratwurzeloperator**

Im Standardkatalog ist nur ein Quadratwurzeloperator (Präfix `|/`) definiert, und er nimmt ein Argument des Typs `double precision` entgegen. Der Scanner weist dem Argument in diesem Abfrageausdruck zunächst den Typ `integer` zu:

```sql
SELECT |/ 40 AS "square root of 40";
```

```text
 square root of 40
-------------------
 6.324555320336759
(1 row)
```

Der Parser führt also eine Typumwandlung für den Operanden durch, und die Abfrage ist gleichbedeutend mit:

```sql
SELECT |/ CAST(40 AS double precision) AS "square root of 40";
```

**Beispiel 10.2. Typauflösung für den Zeichenkettenverkettungsoperator**

Eine zeichenkettenartige Syntax wird sowohl für Zeichenkettentypen als auch für komplexe Erweiterungstypen verwendet. Zeichenketten ohne angegebenen Typ werden mit wahrscheinlichen Operatorkandidaten abgeglichen.

Ein Beispiel mit einem nicht spezifizierten Argument:

```sql
SELECT text 'abc' || 'def' AS "text and unknown";
```

```text
 text and unknown
------------------
 abcdef
(1 row)
```

In diesem Fall prüft der Parser, ob es einen Operator gibt, der für beide Argumente `text` annimmt. Da es ihn gibt, nimmt er an, dass das zweite Argument als Typ `text` interpretiert werden soll.

Hier ist eine Verkettung von zwei Werten ohne spezifizierte Typen:

```sql
SELECT 'abc' || 'def' AS "unspecified";
```

```text
 unspecified
-------------
 abcdef
(1 row)
```

In diesem Fall gibt es keinen anfänglichen Hinweis, welcher Typ verwendet werden soll, weil in der Abfrage keine Typen angegeben sind. Der Parser sucht daher alle Kandidatenoperatoren und findet Kandidaten, die Eingaben sowohl aus der Kategorie Zeichenkette als auch aus der Kategorie Bit-String akzeptieren. Da die Kategorie Zeichenkette bevorzugt wird, wenn sie verfügbar ist, wird diese Kategorie gewählt. Anschließend wird der bevorzugte Zeichenkettentyp `text` als konkreter Typ verwendet, um die Literale unbekannten Typs aufzulösen.

**Beispiel 10.3. Typauflösung für Absolutwert- und Negationsoperatoren**

Der PostgreSQL-Operatorkatalog enthält mehrere Einträge für den Präfixoperator `@`, die alle Absolutwertoperationen für verschiedene numerische Datentypen implementieren. Einer dieser Einträge ist für den Typ `float8`, den bevorzugten Typ in der numerischen Kategorie. Deshalb verwendet PostgreSQL diesen Eintrag bei einer unbekannten Eingabe:

```sql
SELECT @ '-4.5' AS "abs";
```

```text
 abs
-----
 4.5
(1 row)
```

Hier hat das System das Literal unbekannten Typs implizit als Typ `float8` aufgelöst, bevor es den gewählten Operator angewendet hat. Wir können überprüfen, dass `float8` und kein anderer Typ verwendet wurde:

```sql
SELECT @ '-4.5e500' AS "abs";
```

```text
ERROR:      "-4.5e500" is out of range for type double precision
```

Der Präfixoperator `~` für bitweise Negation ist dagegen nur für Ganzzahltypen definiert, nicht für `float8`. Wenn wir also einen ähnlichen Fall mit `~` versuchen, erhalten wir:

```sql
SELECT ~ '20' AS "negation";
```

```text
ERROR: operator is not unique: ~ "unknown"
HINT: Could not choose a best candidate operator. You might need to add explicit type casts.
```

Das passiert, weil das System nicht entscheiden kann, welcher der mehreren möglichen `~`-Operatoren bevorzugt werden soll. Wir können mit einem expliziten Cast helfen:

```sql
SELECT ~ CAST('20' AS int8) AS "negation";
```

```text
 negation
----------
      -21
(1 row)
```

**Beispiel 10.4. Typauflösung für den Array-Inklusionsoperator**

Hier ist ein weiteres Beispiel für die Auflösung eines Operators mit einer bekannten und einer unbekannten Eingabe:

```sql
SELECT array[1,2] <@ '{1,2,3}' as "is subset";
```

```text
 is subset
-----------
 t
(1 row)
```

Der PostgreSQL-Operatorkatalog enthält mehrere Einträge für den Infixoperator `<@`, aber die einzigen zwei, die möglicherweise ein Integer-Array auf der linken Seite akzeptieren könnten, sind Array-Inklusion (`anyarray <@ anyarray`) und Range-Inklusion (`anyelement <@ anyrange`). Da keiner dieser polymorphen Pseudotypen (siehe [Abschnitt 8.21](08_Datentypen.md#821-pseudotypen)) als bevorzugt gilt, kann der Parser die Mehrdeutigkeit nicht auf dieser Grundlage auflösen. Schritt 12 weist ihn jedoch an, anzunehmen, dass das Literal unbekannten Typs denselben Typ hat wie die andere Eingabe, also Integer-Array. Nun kann nur einer der beiden Operatoren passen, sodass Array-Inklusion gewählt wird. Wäre Range-Inklusion gewählt worden, hätten wir einen Fehler erhalten, weil die Zeichenkette nicht das richtige Format für ein Range-Literal hat.

**Beispiel 10.5. Benutzerdefinierter Operator für einen Domain-Typ**

Benutzer versuchen manchmal, Operatoren nur für einen Domain-Typ zu deklarieren. Das ist möglich, aber längst nicht so nützlich, wie es scheint, weil die Regeln der Operatorauflösung darauf ausgelegt sind, Operatoren für den Basistyp der Domain auszuwählen. Betrachten Sie zum Beispiel:

```sql
CREATE DOMAIN mytext AS text CHECK(...);
CREATE FUNCTION mytext_eq_text (mytext, text) RETURNS boolean
 AS ...;

CREATE OPERATOR = (procedure=mytext_eq_text, leftarg=mytext,
 rightarg=text);
CREATE TABLE mytable (val mytext);

SELECT * FROM mytable WHERE val = 'foo';
```

Diese Abfrage verwendet den benutzerdefinierten Operator nicht. Der Parser prüft zuerst, ob es einen Operator `mytext = mytext` gibt (Schritt 4); den gibt es nicht. Dann betrachtet er den Basistyp der Domain, `text`, und prüft, ob es einen Operator `text = text` gibt (Schritt 5); den gibt es. Daher löst er das Literal unbekannten Typs als `text` auf und verwendet den Operator `text = text`. Die einzige Möglichkeit, den benutzerdefinierten Operator zu verwenden, besteht darin, das Literal ausdrücklich zu casten:

```sql
SELECT * FROM mytable WHERE val = text 'foo';
```

Dann wird der Operator `mytext = text` sofort nach der Regel für exakte Übereinstimmung gefunden. Wenn die Regeln für die beste Übereinstimmung erreicht werden, benachteiligen sie Operatoren auf Domain-Typen bewusst. Andernfalls würde ein solcher Operator zu vielen Fehlern wegen mehrdeutiger Operatoren führen, weil die Cast-Regeln eine Domain immer als in ihren Basistyp oder aus ihrem Basistyp castbar betrachten. Der Domain-Operator würde daher in denselben Fällen als verwendbar gelten wie ein gleichnamiger Operator auf dem Basistyp.

## 10.3. Funktionen

Die konkrete Funktion, auf die sich ein Funktionsaufruf bezieht, wird mit dem folgenden Verfahren bestimmt.

**Typauflösung für Funktionen**

1. Wählen Sie die zu betrachtenden Funktionen aus dem Systemkatalog `pg_proc` aus. Wenn ein nicht schemaqualifizierter Funktionsname verwendet wurde, werden diejenigen Funktionen betrachtet, die den passenden Namen und die passende Argumentanzahl haben und im aktuellen Suchpfad sichtbar sind (siehe [Abschnitt 5.10.3](05_Datendefinition.md#5103-der-schemasuchpfad)). Wenn ein qualifizierter Funktionsname angegeben wurde, werden nur Funktionen im angegebenen Schema betrachtet.

2. Optional: Wenn der Suchpfad mehrere Funktionen mit identischen Argumenttypen findet, wird nur diejenige berücksichtigt, die im Pfad zuerst erscheint. Funktionen mit unterschiedlichen Argumenttypen werden unabhängig von ihrer Position im Suchpfad gleich behandelt.

3. Optional: Wenn eine Funktion mit einem `VARIADIC`-Arrayparameter deklariert ist und der Aufruf das Schlüsselwort `VARIADIC` nicht verwendet, wird die Funktion so behandelt, als wäre der Arrayparameter durch eine oder mehrere Vorkommen seines Elementtyps ersetzt, je nachdem, was für den Aufruf nötig ist. Nach dieser Erweiterung kann die Funktion effektive Argumenttypen haben, die mit denen einer nicht variadischen Funktion identisch sind. In diesem Fall wird die im Suchpfad früher erscheinende Funktion verwendet; wenn beide Funktionen im selben Schema liegen, wird die nicht variadische Funktion bevorzugt.

4. Dies erzeugt ein Sicherheitsrisiko, wenn über einen qualifizierten Namen eine variadische Funktion in einem Schema aufgerufen wird, in dem nicht vertrauenswürdige Benutzer Objekte erstellen dürfen. Ein böswilliger Benutzer kann die Kontrolle übernehmen und beliebige SQL-Funktionen ausführen, als hätten Sie sie ausgeführt. Verwenden Sie stattdessen einen Aufruf mit dem Schlüsselwort `VARIADIC`; dieser umgeht das Risiko. Aufrufe, die `VARIADIC`-Parameter vom Typ `"any"` befüllen, haben oft keine gleichwertige Formulierung mit dem Schlüsselwort `VARIADIC`. Um solche Aufrufe sicher auszuführen, darf das Schema der Funktion nur vertrauenswürdigen Benutzern das Erstellen von Objekten erlauben.

5. Optional: Funktionen mit Standardwerten für Parameter gelten als passend für jeden Aufruf, der null oder mehr der Parameterpositionen mit Standardwerten auslässt. Wenn mehr als eine solche Funktion zu einem Aufruf passt, wird diejenige verwendet, die im Suchpfad zuerst erscheint. Wenn es zwei oder mehr solche Funktionen im selben Schema mit identischen Parametertypen in den nicht mit Standardwerten belegten Positionen gibt, was möglich ist, wenn sie verschiedene Mengen von Standardparametern haben, kann das System nicht bestimmen, welche bevorzugt werden soll. Dann entsteht ein Fehler wegen eines mehrdeutigen Funktionsaufrufs, sofern keine bessere Übereinstimmung für den Aufruf gefunden werden kann.

6. Dies erzeugt ein Verfügbarkeitsrisiko, wenn über einen qualifizierten Namen eine Funktion in einem Schema aufgerufen wird, in dem nicht vertrauenswürdige Benutzer Objekte erstellen dürfen. Ein böswilliger Benutzer kann eine Funktion mit dem Namen einer vorhandenen Funktion erstellen, deren Parameter nachbilden und neue Parameter mit Standardwerten anhängen. Dadurch werden neue Aufrufe der ursprünglichen Funktion blockiert. Um dieses Risiko zu vermeiden, legen Sie Funktionen in Schemas ab, in denen nur vertrauenswürdige Benutzer Objekte erstellen dürfen.

7. Prüfen Sie, ob es eine Funktion gibt, die genau die Eingabeargumenttypen akzeptiert. Wenn eine solche Funktion existiert, wird sie verwendet; in der betrachteten Funktionsmenge kann es nur eine exakte Übereinstimmung geben. Fehlt eine exakte Übereinstimmung, entsteht bei einem Aufruf über einen qualifizierten Namen ein Sicherheitsrisiko, falls die Funktion in einem Schema gefunden wird, in dem nicht vertrauenswürdige Benutzer Objekte erstellen dürfen. In solchen Situationen sollten Argumente gecastet werden, um eine exakte Übereinstimmung zu erzwingen. Fälle mit `unknown` finden in diesem Schritt niemals eine Übereinstimmung.

8. Wenn keine exakte Übereinstimmung gefunden wurde, prüfen Sie, ob der Funktionsaufruf wie eine spezielle Typumwandlungsanforderung aussieht. Dies ist der Fall, wenn der Funktionsaufruf genau ein Argument hat und der Funktionsname dem internen Namen eines Datentyps entspricht. Außerdem muss das Funktionsargument entweder ein Literal unbekannten Typs sein, ein Typ, der binär in den genannten Datentyp erzwingbar ist, oder ein Typ, der durch Anwendung der I/O-Funktionen dieses Typs in den genannten Datentyp umgewandelt werden könnte, das heißt, die Umwandlung geht zu oder von einem der Standard-Zeichenkettentypen. Wenn diese Bedingungen erfüllt sind, wird der Funktionsaufruf als eine Form einer `CAST`-Spezifikation behandelt.

9. Suchen Sie nach der besten Übereinstimmung.

10. Verwerfen Sie Kandidatenfunktionen, deren Eingabetypen nicht übereinstimmen und nicht durch implizite Umwandlung passend gemacht werden können. `unknown`-Literale gelten hierfür als in alles umwandelbar. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

11. Wenn ein Eingabeargument ein Domain-Typ ist, behandeln Sie es in allen folgenden Schritten als den Basistyp der Domain. Dadurch verhalten sich Domains bei der Auflösung mehrdeutiger Funktionen wie ihre Basistypen.

12. Durchlaufen Sie alle Kandidaten und behalten Sie diejenigen mit den meisten exakten Übereinstimmungen der Eingabetypen. Wenn keiner exakte Übereinstimmungen hat, behalten Sie alle Kandidaten. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

13. Durchlaufen Sie alle Kandidaten und behalten Sie diejenigen, die an den meisten Positionen, an denen Typumwandlung erforderlich wäre, bevorzugte Typen der jeweiligen Typkategorie akzeptieren. Wenn keiner bevorzugte Typen akzeptiert, behalten Sie alle Kandidaten. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

14. Wenn Eingabeargumente vom Typ `unknown` vorhanden sind, prüfen Sie die Typkategorien, die von den verbleibenden Kandidaten an diesen Argumentpositionen akzeptiert werden. Wählen Sie an jeder Position die Kategorie Zeichenkette, wenn irgendein Kandidat diese Kategorie akzeptiert; diese Bevorzugung von Zeichenketten ist sinnvoll, weil ein Literal unbekannten Typs wie eine Zeichenkette aussieht. Andernfalls wählen Sie die Kategorie, wenn alle verbleibenden Kandidaten dieselbe Typkategorie akzeptieren; sonst schlägt die Auflösung fehl, weil die richtige Wahl ohne weitere Hinweise nicht bestimmt werden kann. Verwerfen Sie nun Kandidaten, die die gewählte Typkategorie nicht akzeptieren. Wenn außerdem ein Kandidat einen bevorzugten Typ in dieser Kategorie akzeptiert, verwerfen Sie Kandidaten, die für dieses Argument nicht bevorzugte Typen akzeptieren. Wenn nach diesen Prüfungen keine Kandidaten übrig bleiben, behalten Sie alle Kandidaten. Wenn nur ein Kandidat übrig bleibt, wird er verwendet; andernfalls geht es mit dem nächsten Schritt weiter.

15. Wenn es sowohl Argumente vom Typ `unknown` als auch Argumente bekannten Typs gibt und alle bekannten Argumente denselben Typ haben, nehmen Sie an, dass auch die `unknown`-Argumente diesen Typ haben, und prüfen Sie, welche Kandidaten diesen Typ an den `unknown`-Argumentpositionen akzeptieren können. Wenn genau ein Kandidat diese Prüfung besteht, wird er verwendet. Andernfalls schlägt die Auflösung fehl.

> **Hinweis** Das Sicherheitsrisiko entsteht nicht bei einem nicht schemaqualifizierten Namen, weil ein Suchpfad, der Schemas enthält, in denen nicht vertrauenswürdige Benutzer Objekte erstellen dürfen, ohnehin kein sicheres Schema-Verwendungsmuster ist. Der Sonderfall für funktionsartige Cast-Spezifikationen unterstützt Cast-Schreibweisen in Fällen, in denen es keine echte Cast-Funktion gibt. Wenn es eine Cast-Funktion gibt, wird sie konventionell nach ihrem Ausgabetyp benannt, sodass kein Sonderfall nötig ist. Weitere Hinweise finden Sie bei `CREATE CAST`.

Beachten Sie, dass die Regeln für die "beste Übereinstimmung" bei der Typauflösung von Operatoren und Funktionen identisch sind. Es folgen einige Beispiele.

**Beispiel 10.6. Typauflösung für Argumente der Rundungsfunktion**

Es gibt nur eine Funktion `round`, die zwei Argumente annimmt; sie erwartet als erstes Argument den Typ `numeric` und als zweites Argument den Typ `integer`. Daher wandelt die folgende Abfrage das erste Argument vom Typ `integer` automatisch in `numeric` um:

```sql
SELECT round(4, 4);
```

```text
 round
--------
 4.0000
(1 row)
```

Diese Abfrage wird vom Parser tatsächlich umgeformt zu:

```sql
SELECT round(CAST (4 AS numeric), 4);
```

Da numerische Konstanten mit Dezimalpunkt zunächst den Typ `numeric` erhalten, benötigt die folgende Abfrage keine Typumwandlung und kann daher geringfügig effizienter sein:

```sql
SELECT round(4.0, 4);
```

**Beispiel 10.7. Auflösung variadischer Funktionen**

```sql
CREATE FUNCTION public.variadic_example(VARIADIC numeric[]) RETURNS int
  LANGUAGE sql AS 'SELECT 1';
CREATE FUNCTION
```

Diese Funktion akzeptiert das Schlüsselwort `VARIADIC`, verlangt es aber nicht. Sie toleriert sowohl `integer`- als auch `numeric`-Argumente:

```sql
SELECT public.variadic_example(0),
       public.variadic_example(0.0),
       public.variadic_example(VARIADIC array[0.0]);
```

```text
 variadic_example | variadic_example | variadic_example
------------------+------------------+------------------
                 1 |                1 |               1
(1 row)
```

Der erste und der zweite Aufruf bevorzugen jedoch spezifischere Funktionen, falls solche verfügbar sind:

```sql
CREATE FUNCTION public.variadic_example(numeric) RETURNS int
  LANGUAGE sql AS 'SELECT 2';
CREATE FUNCTION

CREATE FUNCTION public.variadic_example(int) RETURNS int
  LANGUAGE sql AS 'SELECT 3';
CREATE FUNCTION

SELECT public.variadic_example(0),
       public.variadic_example(0.0),
       public.variadic_example(VARIADIC array[0.0]);
```

```text
 variadic_example | variadic_example | variadic_example
------------------+------------------+------------------
                 3 |                2 |               1
(1 row)
```

Bei Standardkonfiguration und wenn nur die erste Funktion existiert, sind der erste und der zweite Aufruf unsicher. Jeder Benutzer könnte sie abfangen, indem er die zweite oder dritte Funktion erstellt. Der dritte Aufruf ist sicher, weil er den Argumenttyp exakt abgleicht und das Schlüsselwort `VARIADIC` verwendet.

**Beispiel 10.8. Typauflösung für die Funktion `substring`**

Es gibt mehrere Funktionen `substr`, darunter eine mit den Typen `text` und `integer`. Wenn sie mit einer Zeichenkettenkonstante ohne angegebenen Typ aufgerufen wird, wählt das System die Kandidatenfunktion, die ein Argument der bevorzugten Kategorie Zeichenkette akzeptiert, nämlich den Typ `text`.

```sql
SELECT substr('1234', 3);
```

```text
 substr
--------
     34
(1 row)
```

Wenn die Zeichenkette als Typ `varchar` deklariert ist, wie es der Fall sein kann, wenn sie aus einer Tabelle stammt, versucht der Parser, sie in `text` umzuwandeln:

```sql
SELECT substr(varchar '1234', 3);
```

```text
 substr
--------
     34
(1 row)
```

Dies wird vom Parser effektiv zu Folgendem umgeformt:

```sql
SELECT substr(CAST (varchar '1234' AS text), 3);
```

> **Hinweis** Der Parser erfährt aus dem Katalog `pg_cast`, dass `text` und `varchar` binärkompatibel sind. Das bedeutet, dass ein Wert des einen Typs an eine Funktion übergeben werden kann, die den anderen Typ akzeptiert, ohne dass eine physische Umwandlung ausgeführt wird. Daher wird in diesem Fall tatsächlich kein Typumwandlungsaufruf eingefügt.

Wenn die Funktion dagegen mit einem Argument des Typs `integer` aufgerufen wird, versucht der Parser, dieses in `text` umzuwandeln:

```sql
SELECT substr(1234, 3);
```

```text
ERROR: function substr(integer, integer) does not exist
HINT: No function matches the given name and argument types. You might need to add explicit type casts.
```

Das funktioniert nicht, weil `integer` keinen impliziten Cast nach `text` besitzt. Ein expliziter Cast funktioniert jedoch:

```sql
SELECT substr(CAST (1234 AS text), 3);
```

```text
 substr
--------
     34
(1 row)
```

## 10.4. Wertspeicherung

Werte, die in eine Tabelle eingefügt werden sollen, werden nach den folgenden Schritten in den Datentyp der Zielspalte umgewandelt.

**Typumwandlung bei der Wertspeicherung**

1. Prüfen Sie auf eine exakte Übereinstimmung mit dem Zieltyp.

2. Andernfalls versuchen Sie, den Ausdruck in den Zieltyp umzuwandeln. Das ist möglich, wenn im Katalog `pg_cast` ein Zuweisungs-Cast zwischen den beiden Typen registriert ist (siehe `CREATE CAST`). Wenn der Ausdruck stattdessen ein Literal unbekannten Typs ist, wird der Inhalt der Literalzeichenkette an die Eingabe-Umwandlungsroutine des Zieltyps übergeben.

3. Prüfen Sie, ob es einen Größen-Cast für den Zieltyp gibt. Ein Größen-Cast ist ein Cast von diesem Typ in denselben Typ. Wenn ein solcher Cast im Katalog `pg_cast` gefunden wird, wird er auf den Ausdruck angewendet, bevor dieser in der Zielspalte gespeichert wird. Die Implementierungsfunktion eines solchen Casts nimmt immer einen zusätzlichen Parameter vom Typ `integer` entgegen, der den Wert `atttypmod` der Zielspalte erhält, typischerweise ihre deklarierte Länge, obwohl die Interpretation von `atttypmod` je nach Datentyp variiert. Außerdem kann sie einen dritten Parameter vom Typ `boolean` annehmen, der angibt, ob der Cast explizit oder implizit ist. Die Cast-Funktion ist dafür verantwortlich, längenabhängige Semantik wie Größenprüfung oder Abschneiden anzuwenden.

**Beispiel 10.9. Typumwandlung bei der Speicherung in `character`**

Für eine Zielspalte, die als `character(20)` deklariert ist, zeigt die folgende Anweisung, dass der gespeicherte Wert korrekt auf die Größe gebracht wird:

```sql
CREATE TABLE vv (v character(20));
INSERT INTO vv SELECT 'abc' || 'def';
SELECT v, octet_length(v) FROM vv;
```

```text
          v           | octet_length
----------------------+--------------
 abcdef               |           20
(1 row)
```

Tatsächlich wurden hier die beiden unbekannten Literale standardmäßig als `text` aufgelöst, wodurch der Operator `||` als Textverkettung aufgelöst werden konnte. Dann wird das `text`-Ergebnis des Operators in `bpchar` umgewandelt, "blank-padded char", den internen Namen des Datentyps `character`, um zum Typ der Zielspalte zu passen. Da die Umwandlung von `text` nach `bpchar` binär erzwingbar ist, fügt diese Umwandlung keinen echten Funktionsaufruf ein. Schließlich wird die Größenfunktion `bpchar(bpchar, integer, boolean)` im Systemkatalog gefunden und auf das Ergebnis des Operators und die gespeicherte Spaltenlänge angewendet. Diese typspezifische Funktion führt die erforderliche Längenprüfung und das Auffüllen mit Leerzeichen aus.

## 10.5. `UNION`, `CASE` und verwandte Konstrukte

SQL-Konstrukte mit `UNION` müssen möglicherweise unterschiedliche Typen zu einer einzigen Ergebnismenge zusammenführen. Der Auflösungsalgorithmus wird für jede Ausgabespalte einer Union-Abfrage separat angewendet. Die Konstrukte `INTERSECT` und `EXCEPT` lösen unterschiedliche Typen auf dieselbe Weise wie `UNION` auf. Einige andere Konstrukte, darunter `CASE`, `ARRAY`, `VALUES` sowie die Funktionen `GREATEST` und `LEAST`, verwenden denselben Algorithmus, um ihre Teilausdrücke aneinander anzupassen und einen Ergebnisdatentyp auszuwählen.

**Typauflösung für `UNION`, `CASE` und verwandte Konstrukte**

1. Wenn alle Eingaben denselben Typ haben und dieser nicht `unknown` ist, wird dieser Typ gewählt.

2. Wenn eine Eingabe ein Domain-Typ ist, behandeln Sie sie in allen folgenden Schritten als den Basistyp der Domain.

3. Wenn alle Eingaben den Typ `unknown` haben, wird der Typ `text` gewählt, der bevorzugte Typ der Kategorie Zeichenkette. Andernfalls werden `unknown`-Eingaben für die verbleibenden Regeln ignoriert.

4. Wenn die nicht unbekannten Eingaben nicht alle derselben Typkategorie angehören, schlägt die Auflösung fehl.

5. Wählen Sie den ersten nicht unbekannten Eingabetyp als Kandidatentyp und betrachten Sie dann jeden anderen nicht unbekannten Eingabetyp von links nach rechts. Wenn der Kandidatentyp implizit in den anderen Typ umgewandelt werden kann, aber nicht umgekehrt, wählen Sie den anderen Typ als neuen Kandidatentyp. Fahren Sie dann mit den verbleibenden Eingaben fort. Wenn in irgendeinem Schritt dieses Prozesses ein bevorzugter Typ gewählt wird, werden keine weiteren Eingaben betrachtet.

6. Wandeln Sie alle Eingaben in den endgültigen Kandidatentyp um. Schlägt fehl, wenn es keine implizite Umwandlung von einem gegebenen Eingabetyp in den Kandidatentyp gibt.

> **Hinweis** Ähnlich wie die Behandlung von Domain-Eingaben für Operatoren und Funktionen ermöglicht dieses Verhalten, einen Domain-Typ durch ein `UNION`- oder ähnliches Konstrukt hindurch zu bewahren, solange der Benutzer sorgfältig sicherstellt, dass alle Eingaben implizit oder explizit genau diesen Typ haben. Andernfalls wird der Basistyp der Domain verwendet. Aus historischen Gründen behandelt `CASE` seine `ELSE`-Klausel, falls vorhanden, als die "erste" Eingabe, wobei die `THEN`-Klauseln danach betrachtet werden. In allen anderen Fällen bedeutet "von links nach rechts" die Reihenfolge, in der die Ausdrücke im Abfragetext erscheinen.

Es folgen einige Beispiele.

**Beispiel 10.10. Typauflösung mit unterspezifizierten Typen in einer Union**

```sql
SELECT text 'a' AS "text" UNION SELECT 'b';
```

```text
 text
------
 a
 b
(2 rows)
```

Hier wird das Literal `'b'` unbekannten Typs als Typ `text` aufgelöst.

**Beispiel 10.11. Typauflösung in einer einfachen Union**

```sql
SELECT 1.2 AS "numeric" UNION SELECT 1;
```

```text
 numeric
---------
       1
     1.2
(2 rows)
```

Das Literal `1.2` hat den Typ `numeric`, und der ganzzahlige Wert `1` kann implizit nach `numeric` gecastet werden, sodass dieser Typ verwendet wird.

**Beispiel 10.12. Typauflösung in einer vertauschten Union**

```sql
SELECT 1 AS "real" UNION SELECT CAST('2.2' AS REAL);
```

```text
 real
------
    1
  2.2
(2 rows)
```

Hier kann der Typ `real` nicht implizit nach `integer` gecastet werden, aber `integer` kann implizit nach `real` gecastet werden. Deshalb wird der Ergebnistyp der Union als `real` aufgelöst.

**Beispiel 10.13. Typauflösung in einer verschachtelten Union**

```sql
SELECT NULL UNION SELECT NULL UNION SELECT 1;
```

```text
ERROR:      UNION types text and integer cannot be matched
```

Dieser Fehler entsteht, weil PostgreSQL mehrere `UNION`-Operationen als verschachtelte paarweise Operationen behandelt; diese Eingabe ist also gleichbedeutend mit:

```sql
(SELECT NULL UNION SELECT NULL) UNION SELECT 1;
```

Die innere `UNION` wird nach den oben angegebenen Regeln so aufgelöst, dass sie den Typ `text` ausgibt. Die äußere `UNION` hat dann Eingaben der Typen `text` und `integer`, was zum beobachteten Fehler führt. Das Problem lässt sich beheben, indem sichergestellt wird, dass die am weitesten links stehende `UNION` mindestens eine Eingabe des gewünschten Ergebnistyps hat.

Auch `INTERSECT`- und `EXCEPT`-Operationen werden paarweise aufgelöst. Die anderen in diesem Abschnitt beschriebenen Konstrukte betrachten dagegen alle ihre Eingaben in einem einzigen Auflösungsschritt.

## 10.6. Ausgabespalten von `SELECT`

Die in den vorangegangenen Abschnitten beschriebenen Regeln führen dazu, dass allen Ausdrücken in einer SQL-Abfrage nicht unbekannte Datentypen zugewiesen werden. Eine Ausnahme bilden Literale ohne angegebenen Typ, die als einfache Ausgabespalten eines `SELECT`-Befehls erscheinen. Zum Beispiel gibt es in

```sql
SELECT 'Hello World';
```

nichts, das erkennen lässt, als welcher Typ das Zeichenkettenliteral betrachtet werden soll. In dieser Situation fällt PostgreSQL darauf zurück, den Typ des Literals als `text` aufzulösen.

Wenn das `SELECT` ein Arm eines `UNION`-, `INTERSECT`- oder `EXCEPT`-Konstrukts ist oder innerhalb von `INSERT ... SELECT` erscheint, wird diese Regel nicht angewendet, weil die in den vorherigen Abschnitten angegebenen Regeln Vorrang haben. Der Typ eines Literals ohne angegebenen Typ kann im ersten Fall vom anderen `UNION`-Arm oder im zweiten Fall von der Zielspalte übernommen werden.

`RETURNING`-Listen werden für diesen Zweck genauso behandelt wie Ausgabelisten von `SELECT`.

> **Hinweis** Vor PostgreSQL 10 gab es diese Regel nicht, und Literale ohne angegebenen Typ in einer `SELECT`-Ausgabeliste blieben vom Typ `unknown`. Das hatte verschiedene unerwünschte Folgen und wurde daher geändert.
