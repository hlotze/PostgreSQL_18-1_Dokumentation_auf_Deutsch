# 37. Trigger

Dieses Kapitel enthält allgemeine Informationen zum Schreiben von Triggerfunktionen. Triggerfunktionen können in den meisten verfügbaren prozeduralen Sprachen geschrieben werden, darunter PL/pgSQL ([Kapitel 41](41_PL_pgSQL_SQL_Prozedursprache.md)), PL/Tcl ([Kapitel 42](42_PL_Tcl_Tcl_Prozedursprache.md)), PL/Perl ([Kapitel 43](43_PL_Perl_Perl_Prozedursprache.md)) und PL/Python ([Kapitel 44](44_PL_Python_Python_Prozedursprache.md)). Nachdem Sie dieses Kapitel gelesen haben, sollten Sie das Kapitel zu Ihrer bevorzugten prozeduralen Sprache heranziehen, um die sprachspezifischen Details zum Schreiben eines Triggers in dieser Sprache zu erfahren.

Es ist auch möglich, eine Triggerfunktion in C zu schreiben, obwohl die meisten Menschen es einfacher finden, eine der prozeduralen Sprachen zu verwenden. Derzeit ist es nicht möglich, eine Triggerfunktion in der einfachen SQL-Funktionssprache zu schreiben.

## 37.1. Überblick über das Triggerverhalten

Ein Trigger ist eine Festlegung, dass die Datenbank automatisch eine bestimmte Funktion ausführen soll, sobald eine bestimmte Art von Operation ausgeführt wird. Trigger können an Tabellen, partitioniert oder nicht, Views und Fremdtabellen angehängt werden.

Auf Tabellen und Fremdtabellen können Trigger so definiert werden, dass sie entweder vor oder nach einer `INSERT`-, `UPDATE`- oder `DELETE`-Operation ausgeführt werden, entweder einmal pro geänderter Zeile oder einmal pro SQL-Anweisung. `UPDATE`-Trigger können außerdem so eingestellt werden, dass sie nur feuern, wenn bestimmte Spalten in der `SET`-Klausel der `UPDATE`-Anweisung erwähnt werden. Trigger können auch bei `TRUNCATE`-Anweisungen feuern. Wenn ein Triggerereignis eintritt, wird die Funktion des Triggers zum passenden Zeitpunkt aufgerufen, um das Ereignis zu behandeln.

Auf Views können Trigger so definiert werden, dass sie statt `INSERT`-, `UPDATE`- oder `DELETE`-Operationen ausgeführt werden. Solche `INSTEAD OF`-Trigger feuern einmal für jede Zeile, die in der View geändert werden muss. Es liegt in der Verantwortung der Triggerfunktion, die notwendigen Änderungen an den zugrunde liegenden Basistabellen der View vorzunehmen und, wo passend, die geänderte Zeile so zurückzugeben, wie sie in der View erscheinen wird. Trigger auf Views können auch so definiert werden, dass sie einmal pro SQL-Anweisung vor oder nach `INSERT`-, `UPDATE`- oder `DELETE`-Operationen ausgeführt werden. Solche Trigger feuern jedoch nur, wenn es auf der View auch einen `INSTEAD OF`-Trigger gibt. Andernfalls muss jede Anweisung, die auf die View zielt, in eine Anweisung umgeschrieben werden, die ihre zugrunde liegenden Basistabellen betrifft; dann feuern die Trigger, die an den Basistabellen hängen.

Die Triggerfunktion muss definiert werden, bevor der Trigger selbst erstellt werden kann. Die Triggerfunktion muss als Funktion deklariert werden, die keine Argumente annimmt und den Typ `trigger` zurückgibt. Die Triggerfunktion erhält ihre Eingabe über eine speziell übergebene Struktur `TriggerData`, nicht in Form gewöhnlicher Funktionsargumente.

Sobald eine geeignete Triggerfunktion erstellt wurde, wird der Trigger mit `CREATE TRIGGER` eingerichtet. Dieselbe Triggerfunktion kann für mehrere Trigger verwendet werden.

PostgreSQL bietet sowohl Trigger pro Zeile als auch Trigger pro Anweisung. Bei einem Trigger pro Zeile wird die Triggerfunktion einmal für jede Zeile aufgerufen, die von der Anweisung betroffen ist, welche den Trigger ausgelöst hat. Ein Trigger pro Anweisung wird dagegen nur einmal aufgerufen, wenn eine passende Anweisung ausgeführt wird, unabhängig davon, wie viele Zeilen diese Anweisung betrifft. Insbesondere führt auch eine Anweisung, die null Zeilen betrifft, zur Ausführung aller zutreffenden Trigger pro Anweisung. Diese beiden Triggerarten werden auch Zeilen-Trigger und Anweisungs-Trigger genannt. Trigger auf `TRUNCATE` können nur auf Anweisungsebene definiert werden, nicht pro Zeile.

Trigger werden außerdem danach klassifiziert, ob sie vor, nach oder statt der Operation feuern. Sie werden entsprechend als `BEFORE`-Trigger, `AFTER`-Trigger und `INSTEAD OF`-Trigger bezeichnet. `BEFORE`-Trigger auf Anweisungsebene feuern naturgemäß, bevor die Anweisung irgendetwas tut, während `AFTER`-Trigger auf Anweisungsebene ganz am Ende der Anweisung feuern. Diese Triggerarten können auf Tabellen, Views oder Fremdtabellen definiert werden. `BEFORE`-Trigger auf Zeilenebene feuern unmittelbar bevor eine bestimmte Zeile bearbeitet wird, während `AFTER`-Trigger auf Zeilenebene am Ende der Anweisung feuern, aber vor allen `AFTER`-Triggern auf Anweisungsebene. Diese Triggerarten können nur auf Tabellen und Fremdtabellen definiert werden, nicht auf Views. `INSTEAD OF`-Trigger können nur auf Views und nur auf Zeilenebene definiert werden; sie feuern unmittelbar, sobald jede Zeile in der View als zu bearbeitend identifiziert wird.

Die Ausführung eines `AFTER`-Triggers kann bis zum Ende der Transaktion statt nur bis zum Ende der Anweisung aufgeschoben werden, wenn er als Constraint-Trigger definiert wurde. In allen Fällen wird ein Trigger als Teil derselben Transaktion ausgeführt wie die Anweisung, die ihn ausgelöst hat. Wenn also entweder die Anweisung oder der Trigger einen Fehler verursacht, werden die Wirkungen beider zurückgerollt. Außerdem läuft der Trigger immer unter der Rolle, die das Triggerereignis eingereiht hat, es sei denn, die Triggerfunktion ist als `SECURITY DEFINER` markiert; in diesem Fall läuft sie als Eigentümer der Funktion.

Wenn ein `INSERT` eine Klausel `ON CONFLICT DO UPDATE` enthält, können für ausgelöste Zeilen zunächst zeilenbezogene `BEFORE INSERT`- und dann `BEFORE UPDATE`-Trigger ausgeführt werden. Solche Wechselwirkungen können komplex sein, wenn die Trigger nicht idempotent sind, weil Änderungen durch `BEFORE INSERT`-Trigger von `BEFORE UPDATE`-Triggern gesehen werden, einschließlich Änderungen an `EXCLUDED`-Spalten.

Beachten Sie, dass `UPDATE`-Trigger auf Anweisungsebene ausgeführt werden, wenn `ON CONFLICT DO UPDATE` angegeben ist, unabhängig davon, ob tatsächlich Zeilen vom `UPDATE` betroffen waren, und unabhängig davon, ob der alternative `UPDATE`-Pfad überhaupt genommen wurde. Ein `INSERT` mit einer Klausel `ON CONFLICT DO UPDATE` führt zuerst `BEFORE INSERT`-Trigger auf Anweisungsebene aus, dann `BEFORE UPDATE`-Trigger auf Anweisungsebene, gefolgt von `AFTER UPDATE`-Triggern auf Anweisungsebene und schließlich `AFTER INSERT`-Triggern auf Anweisungsebene.

Eine Anweisung, die auf eine Elterntabelle in einer Vererbungs- oder Partitionierungshierarchie zielt, bewirkt nicht, dass die Trigger auf Anweisungsebene betroffener Kindtabellen feuern; es feuern nur die Trigger auf Anweisungsebene der Elterntabelle. Zeilenbezogene Trigger aller betroffenen Kindtabellen feuern jedoch.

Wenn ein `UPDATE` auf einer partitionierten Tabelle dazu führt, dass eine Zeile in eine andere Partition verschoben wird, wird dies als `DELETE` aus der ursprünglichen Partition gefolgt von einem `INSERT` in die neue Partition ausgeführt. In diesem Fall feuern alle zeilenbezogenen `BEFORE UPDATE`-Trigger und alle zeilenbezogenen `BEFORE DELETE`-Trigger auf der ursprünglichen Partition. Danach feuern alle zeilenbezogenen `BEFORE INSERT`-Trigger auf der Zielpartition. Die Möglichkeit überraschender Ergebnisse sollte bedacht werden, wenn all diese Trigger die verschobene Zeile beeinflussen. Was `AFTER ROW`-Trigger betrifft, werden `AFTER DELETE`- und `AFTER INSERT`-Trigger angewendet; `AFTER UPDATE`-Trigger werden jedoch nicht angewendet, weil das `UPDATE` in ein `DELETE` und ein `INSERT` umgewandelt wurde. Was Trigger auf Anweisungsebene betrifft, feuern keine `DELETE`- oder `INSERT`-Trigger, selbst wenn Zeilenverschiebung auftritt; es feuern nur die `UPDATE`-Trigger, die auf der in der `UPDATE`-Anweisung verwendeten Zieltabelle definiert sind.

Für `MERGE` werden keine separaten Trigger definiert. Stattdessen feuern `UPDATE`-, `DELETE`- und `INSERT`-Trigger auf Anweisungs- oder Zeilenebene, abhängig davon, welche Aktionen für Trigger auf Anweisungsebene in der `MERGE`-Abfrage angegeben sind und welche Aktionen für Trigger auf Zeilenebene tatsächlich ausgeführt werden.

Während ein `MERGE`-Befehl läuft, feuern `BEFORE`- und `AFTER`-Trigger auf Anweisungsebene für Ereignisse, die in den Aktionen des `MERGE`-Befehls angegeben sind, unabhängig davon, ob die Aktion letztlich ausgeführt wird. Das entspricht einer `UPDATE`-Anweisung, die keine Zeilen aktualisiert, aber dennoch Trigger auf Anweisungsebene feuert. Die Trigger auf Zeilenebene feuern nur, wenn eine Zeile tatsächlich aktualisiert, eingefügt oder gelöscht wird. Es ist also völlig zulässig, dass für bestimmte Aktionsarten Trigger auf Anweisungsebene feuern, aber keine Trigger auf Zeilenebene derselben Aktionsart.

Triggerfunktionen, die von Triggern pro Anweisung aufgerufen werden, sollten immer `NULL` zurückgeben. Triggerfunktionen, die von Triggern pro Zeile aufgerufen werden, können dem aufrufenden Executor eine Tabellenzeile zurückgeben, also einen Wert vom Typ `HeapTuple`, wenn sie möchten. Ein zeilenbezogener Trigger, der vor einer Operation feuert, hat folgende Möglichkeiten:

- Er kann `NULL` zurückgeben, um die Operation für die aktuelle Zeile zu überspringen. Das weist den Executor an, die zeilenbezogene Operation, die den Trigger aufgerufen hat, nicht auszuführen, also das Einfügen, Ändern oder Löschen einer bestimmten Tabellenzeile.

- Nur bei zeilenbezogenen `INSERT`- und `UPDATE`-Triggern wird die zurückgegebene Zeile zur Zeile, die eingefügt wird oder die zu aktualisierende Zeile ersetzt. Dadurch kann die Triggerfunktion die einzufügende oder zu aktualisierende Zeile ändern.

Ein zeilenbezogener `BEFORE`-Trigger, der keines dieser beiden Verhalten auslösen soll, muss darauf achten, als Ergebnis dieselbe Zeile zurückzugeben, die ihm übergeben wurde, also die `NEW`-Zeile für `INSERT`- und `UPDATE`-Trigger und die `OLD`-Zeile für `DELETE`-Trigger.

Ein zeilenbezogener `INSTEAD OF`-Trigger sollte entweder `NULL` zurückgeben, um anzuzeigen, dass er keine Daten in den zugrunde liegenden Basistabellen der View geändert hat, oder er sollte die View-Zeile zurückgeben, die ihm übergeben wurde, also die `NEW`-Zeile für `INSERT`- und `UPDATE`-Operationen oder die `OLD`-Zeile für `DELETE`-Operationen. Ein von `NULL` verschiedener Rückgabewert signalisiert, dass der Trigger die notwendigen Datenänderungen in der View vorgenommen hat. Dadurch wird die Anzahl der vom Befehl betroffenen Zeilen erhöht. Nur bei `INSERT`- und `UPDATE`-Operationen darf der Trigger die `NEW`-Zeile ändern, bevor er sie zurückgibt. Das ändert die von `INSERT RETURNING` oder `UPDATE RETURNING` zurückgegebenen Daten und ist nützlich, wenn die View nicht exakt dieselben Daten zeigt, die bereitgestellt wurden.

Der Rückgabewert wird bei zeilenbezogenen Triggern ignoriert, die nach einer Operation feuern; sie können daher `NULL` zurückgeben.

Für generierte Spalten gelten einige besondere Überlegungen. Gespeicherte generierte Spalten werden nach `BEFORE`-Triggern und vor `AFTER`-Triggern berechnet. Daher kann der generierte Wert in `AFTER`-Triggern inspiziert werden. In `BEFORE`-Triggern enthält die `OLD`-Zeile wie erwartet den alten generierten Wert, aber die `NEW`-Zeile enthält den neuen generierten Wert noch nicht und sollte nicht darauf zugreifen. In der C-Sprachschnittstelle ist der Inhalt der Spalte zu diesem Zeitpunkt undefiniert; eine höhere Programmiersprache sollte den Zugriff auf eine gespeicherte generierte Spalte in der `NEW`-Zeile eines `BEFORE`-Triggers verhindern. Änderungen am Wert einer generierten Spalte in einem `BEFORE`-Trigger werden ignoriert und überschrieben. Virtuelle generierte Spalten werden nie berechnet, wenn Trigger feuern. In der C-Sprachschnittstelle ist ihr Inhalt in einer Triggerfunktion undefiniert. Höhere Programmiersprachen sollten den Zugriff auf virtuelle generierte Spalten in Triggern verhindern.

Wenn für dasselbe Ereignis auf derselben Relation mehr als ein Trigger definiert ist, feuern die Trigger in alphabetischer Reihenfolge nach Triggername. Bei `BEFORE`- und `INSTEAD OF`-Triggern wird die möglicherweise geänderte Zeile, die jeder Trigger zurückgibt, zur Eingabe für den nächsten Trigger. Wenn irgendein `BEFORE`- oder `INSTEAD OF`-Trigger `NULL` zurückgibt, wird die Operation für diese Zeile abgebrochen, und nachfolgende Trigger feuern für diese Zeile nicht.

Eine Triggerdefinition kann auch eine boolesche `WHEN`-Bedingung angeben, die geprüft wird, um zu entscheiden, ob der Trigger feuern soll. In Triggern auf Zeilenebene kann die `WHEN`-Bedingung die alten und/oder neuen Werte von Spalten der Zeile untersuchen. Trigger auf Anweisungsebene können ebenfalls `WHEN`-Bedingungen haben, obwohl diese Funktion für sie nicht so nützlich ist. In einem `BEFORE`-Trigger wird die `WHEN`-Bedingung unmittelbar bevor die Funktion ausgeführt wird oder ausgeführt würde ausgewertet; die Verwendung von `WHEN` unterscheidet sich daher nicht wesentlich davon, dieselbe Bedingung am Anfang der Triggerfunktion zu prüfen. In einem `AFTER`-Trigger wird die `WHEN`-Bedingung jedoch direkt nach der Zeilenaktualisierung ausgewertet und bestimmt, ob ein Ereignis eingereiht wird, um den Trigger am Ende der Anweisung feuern zu lassen. Wenn die `WHEN`-Bedingung eines `AFTER`-Triggers also nicht wahr zurückgibt, muss weder ein Ereignis eingereiht noch die Zeile am Ende der Anweisung erneut geholt werden. Das kann bei Anweisungen, die viele Zeilen ändern, zu erheblichen Beschleunigungen führen, wenn der Trigger nur für wenige Zeilen feuern muss. `INSTEAD OF`-Trigger unterstützen keine `WHEN`-Bedingungen.

Typischerweise werden zeilenbezogene `BEFORE`-Trigger verwendet, um die Daten zu prüfen oder zu ändern, die eingefügt oder aktualisiert werden sollen. Ein `BEFORE`-Trigger könnte zum Beispiel die aktuelle Zeit in eine Zeitstempelspalte eintragen oder prüfen, dass zwei Elemente der Zeile konsistent sind. Zeilenbezogene `AFTER`-Trigger werden am sinnvollsten verwendet, um Aktualisierungen an andere Tabellen weiterzugeben oder Konsistenzprüfungen gegen andere Tabellen vorzunehmen. Der Grund für diese Arbeitsteilung ist, dass ein `AFTER`-Trigger sicher sein kann, den endgültigen Wert der Zeile zu sehen, während ein `BEFORE`-Trigger das nicht kann; es könnten nach ihm noch andere `BEFORE`-Trigger feuern. Wenn Sie keinen besonderen Grund haben, einen Trigger als `BEFORE` oder `AFTER` zu definieren, ist der `BEFORE`-Fall effizienter, da die Informationen über die Operation nicht bis zum Ende der Anweisung gespeichert werden müssen.

Wenn eine Triggerfunktion SQL-Befehle ausführt, können diese Befehle erneut Trigger auslösen. Das wird als kaskadierende Trigger bezeichnet. Es gibt keine direkte Begrenzung der Anzahl von Kaskadenebenen. Kaskaden können zu einem rekursiven Aufruf desselben Triggers führen; zum Beispiel könnte ein `INSERT`-Trigger einen Befehl ausführen, der eine zusätzliche Zeile in dieselbe Tabelle einfügt, wodurch der `INSERT`-Trigger erneut feuert. Es liegt in der Verantwortung des Triggerprogrammierers, in solchen Szenarien unendliche Rekursion zu vermeiden.

Wenn eine Fremdschlüsselbeschränkung referenzielle Aktionen angibt, also kaskadierende Aktualisierungen oder Löschungen, werden diese Aktionen über gewöhnliche SQL-Befehle `UPDATE` oder `DELETE` auf der referenzierenden Tabelle ausgeführt. Insbesondere feuern alle Trigger, die auf der referenzierenden Tabelle existieren, für diese Änderungen. Wenn ein solcher Trigger die Wirkung eines dieser Befehle verändert oder blockiert, könnte das Endergebnis sein, dass die referenzielle Integrität verletzt wird. Es liegt in der Verantwortung des Triggerprogrammierers, dies zu vermeiden.

Wenn ein Trigger definiert wird, können Argumente für ihn angegeben werden. Der Zweck von Argumenten in der Triggerdefinition besteht darin, verschiedenen Triggern mit ähnlichen Anforderungen zu erlauben, dieselbe Funktion aufzurufen. Als Beispiel könnte es eine verallgemeinerte Triggerfunktion geben, die zwei Spaltennamen als Argumente annimmt und den aktuellen Benutzer in die eine und den aktuellen Zeitstempel in die andere einträgt. Richtig geschrieben wäre diese Triggerfunktion unabhängig von der konkreten Tabelle, auf der sie feuert. Dieselbe Funktion könnte also für `INSERT`-Ereignisse auf jeder Tabelle mit passenden Spalten verwendet werden, etwa um die Erzeugung von Datensätzen in einer Transaktionstabelle automatisch nachzuverfolgen. Sie könnte auch zur Nachverfolgung von letzten Aktualisierungen verwendet werden, wenn sie als `UPDATE`-Trigger definiert ist.

Jede Programmiersprache, die Trigger unterstützt, hat ihre eigene Methode, um der Triggerfunktion die Trigger-Eingabedaten verfügbar zu machen. Diese Eingabedaten enthalten die Art des Triggerereignisses, zum Beispiel `INSERT` oder `UPDATE`, sowie alle Argumente, die in `CREATE TRIGGER` aufgeführt wurden. Bei einem Trigger auf Zeilenebene enthalten die Eingabedaten außerdem die `NEW`-Zeile für `INSERT`- und `UPDATE`-Trigger und/oder die `OLD`-Zeile für `UPDATE`- und `DELETE`-Trigger.

Standardmäßig haben Trigger auf Anweisungsebene keine Möglichkeit, die einzelnen von der Anweisung geänderten Zeilen zu untersuchen. Ein `AFTER STATEMENT`-Trigger kann jedoch verlangen, dass Übergangstabellen erstellt werden, um die Mengen der betroffenen Zeilen für den Trigger verfügbar zu machen. Auch `AFTER ROW`-Trigger können Übergangstabellen anfordern, sodass sie sowohl die gesamten Änderungen in der Tabelle als auch die Änderung an der einzelnen Zeile sehen können, für die sie gerade feuern. Die Methode zum Untersuchen der Übergangstabellen hängt wiederum von der verwendeten Programmiersprache ab; der typische Ansatz besteht aber darin, die Übergangstabellen wie schreibgeschützte temporäre Tabellen wirken zu lassen, auf die SQL-Befehle innerhalb der Triggerfunktion zugreifen können.

## 37.2. Sichtbarkeit von Datenänderungen

Wenn Sie in Ihrer Triggerfunktion SQL-Befehle ausführen und diese Befehle auf die Tabelle zugreifen, für die der Trigger gilt, müssen Sie die Regeln zur Datensichtbarkeit kennen. Diese Regeln bestimmen, ob die SQL-Befehle die Datenänderung sehen, für die der Trigger feuert. Kurz gesagt:

- Trigger auf Anweisungsebene folgen einfachen Sichtbarkeitsregeln: Keine der von einer Anweisung vorgenommenen Änderungen ist für `BEFORE`-Trigger auf Anweisungsebene sichtbar, während alle Änderungen für `AFTER`-Trigger auf Anweisungsebene sichtbar sind.

- Die Datenänderung, also Einfügen, Aktualisieren oder Löschen, die den Trigger zum Feuern bringt, ist für SQL-Befehle, die in einem zeilenbezogenen `BEFORE`-Trigger ausgeführt werden, natürlich nicht sichtbar, weil sie noch nicht stattgefunden hat.

- SQL-Befehle, die in einem zeilenbezogenen `BEFORE`-Trigger ausgeführt werden, sehen jedoch die Wirkungen von Datenänderungen für Zeilen, die zuvor im selben äußeren Befehl verarbeitet wurden. Das erfordert Vorsicht, da die Reihenfolge dieser Änderungsereignisse im Allgemeinen nicht vorhersehbar ist; ein SQL-Befehl, der mehrere Zeilen betrifft, kann die Zeilen in beliebiger Reihenfolge besuchen.

- Ebenso sieht ein zeilenbezogener `INSTEAD OF`-Trigger die Wirkungen von Datenänderungen, die durch vorherige Ausführungen von `INSTEAD OF`-Triggern im selben äußeren Befehl vorgenommen wurden.

- Wenn ein zeilenbezogener `AFTER`-Trigger feuert, sind alle vom äußeren Befehl vorgenommenen Datenänderungen bereits abgeschlossen und für die aufgerufene Triggerfunktion sichtbar.

Wenn Ihre Triggerfunktion in einer der Standard-Prozedursprachen geschrieben ist, gelten die obigen Aussagen nur, wenn die Funktion als `VOLATILE` deklariert ist. Funktionen, die als `STABLE` oder `IMMUTABLE` deklariert sind, sehen in keinem Fall Änderungen, die vom aufrufenden Befehl vorgenommen wurden.

Weitere Informationen zu den Regeln der Datensichtbarkeit finden Sie in [Abschnitt 45.5](45_Server_Programming_Interface.md#455-sichtbarkeit-von-datenänderungen). Das Beispiel in [Abschnitt 37.4](#374-ein-vollständiges-triggerbeispiel) demonstriert diese Regeln.

## 37.3. Triggerfunktionen in C schreiben

Dieser Abschnitt beschreibt die Low-Level-Details der Schnittstelle zu einer Triggerfunktion. Diese Informationen werden nur benötigt, wenn Triggerfunktionen in C geschrieben werden. Wenn Sie eine höhere Sprache verwenden, werden diese Details für Sie behandelt. In den meisten Fällen sollten Sie eine prozedurale Sprache in Betracht ziehen, bevor Sie Ihre Trigger in C schreiben. Die Dokumentation jeder prozeduralen Sprache erklärt, wie ein Trigger in dieser Sprache geschrieben wird.

Triggerfunktionen müssen die Funktionsmanager-Schnittstelle „Version 1“ verwenden.

Wenn eine Funktion vom Trigger-Manager aufgerufen wird, werden ihr keine normalen Argumente übergeben, sondern ein „context“-Zeiger, der auf eine Struktur `TriggerData` zeigt. C-Funktionen können prüfen, ob sie vom Trigger-Manager aufgerufen wurden, indem sie das folgende Makro ausführen:

```c
CALLED_AS_TRIGGER(fcinfo)
```

Dies expandiert zu:

```c
((fcinfo)->context != NULL && IsA((fcinfo)->context, TriggerData))
```

Wenn dies wahr zurückgibt, ist es sicher, `fcinfo->context` in den Typ `TriggerData *` umzuwandeln und die Struktur `TriggerData`, auf die gezeigt wird, zu verwenden. Die Funktion darf weder die Struktur `TriggerData` noch irgendeine der Daten verändern, auf die sie zeigt.

`struct TriggerData` ist in `commands/trigger.h` definiert:

```c
typedef struct TriggerData
{
    NodeTag          type;
    TriggerEvent     tg_event;
    Relation         tg_relation;
    HeapTuple        tg_trigtuple;
    HeapTuple        tg_newtuple;
    Trigger         *tg_trigger;
    TupleTableSlot  *tg_trigslot;
    TupleTableSlot  *tg_newslot;
    Tuplestorestate *tg_oldtable;
    Tuplestorestate *tg_newtable;
    const Bitmapset *tg_updatedcols;
} TriggerData;
```

Die Mitglieder sind wie folgt definiert:

- `type`
  - Immer `T_TriggerData`.

- `tg_event`
  - Beschreibt das Ereignis, für das die Funktion aufgerufen wird. Sie können die folgenden Makros verwenden, um `tg_event` zu untersuchen:

    - `TRIGGER_FIRED_BEFORE(tg_event)`
      - Gibt wahr zurück, wenn der Trigger vor der Operation gefeuert hat.

    - `TRIGGER_FIRED_AFTER(tg_event)`
      - Gibt wahr zurück, wenn der Trigger nach der Operation gefeuert hat.

    - `TRIGGER_FIRED_INSTEAD(tg_event)`
      - Gibt wahr zurück, wenn der Trigger statt der Operation gefeuert hat.

    - `TRIGGER_FIRED_FOR_ROW(tg_event)`
      - Gibt wahr zurück, wenn der Trigger für ein Ereignis auf Zeilenebene gefeuert hat.

    - `TRIGGER_FIRED_FOR_STATEMENT(tg_event)`
      - Gibt wahr zurück, wenn der Trigger für ein Ereignis auf Anweisungsebene gefeuert hat.

    - `TRIGGER_FIRED_BY_INSERT(tg_event)`
      - Gibt wahr zurück, wenn der Trigger durch einen `INSERT`-Befehl ausgelöst wurde.

    - `TRIGGER_FIRED_BY_UPDATE(tg_event)`
      - Gibt wahr zurück, wenn der Trigger durch einen `UPDATE`-Befehl ausgelöst wurde.

    - `TRIGGER_FIRED_BY_DELETE(tg_event)`
      - Gibt wahr zurück, wenn der Trigger durch einen `DELETE`-Befehl ausgelöst wurde.

    - `TRIGGER_FIRED_BY_TRUNCATE(tg_event)`
      - Gibt wahr zurück, wenn der Trigger durch einen `TRUNCATE`-Befehl ausgelöst wurde.

- `tg_relation`
  - Ein Zeiger auf eine Struktur, die die Relation beschreibt, für die der Trigger gefeuert hat. Details zu dieser Struktur finden Sie in `utils/rel.h`. Die interessantesten Dinge sind `tg_relation->rd_att`, der Deskriptor der Relationstupel, und `tg_relation->rd_rel->relname`, der Relationsname. Der Typ ist nicht `char *`, sondern `NameData`; verwenden Sie `SPI_getrelname(tg_relation)`, um ein `char *` zu erhalten, wenn Sie eine Kopie des Namens benötigen.

- `tg_trigtuple`
  - Ein Zeiger auf die Zeile, für die der Trigger gefeuert hat. Dies ist die Zeile, die eingefügt, aktualisiert oder gelöscht wird. Wenn dieser Trigger für ein `INSERT` oder `DELETE` gefeuert hat, ist dies der Wert, den Sie aus der Funktion zurückgeben sollten, wenn Sie die Zeile nicht durch eine andere ersetzen, im Fall von `INSERT`, oder die Operation überspringen möchten. Bei Triggern auf Fremdtabellen sind die Werte von Systemspalten hierin nicht spezifiziert.

- `tg_newtuple`
  - Ein Zeiger auf die neue Version der Zeile, wenn der Trigger für ein `UPDATE` gefeuert hat, und `NULL`, wenn es sich um ein `INSERT` oder `DELETE` handelt. Dies ist der Wert, den Sie aus der Funktion zurückgeben müssen, wenn das Ereignis ein `UPDATE` ist und Sie diese Zeile nicht durch eine andere ersetzen oder die Operation überspringen möchten. Bei Triggern auf Fremdtabellen sind die Werte von Systemspalten hierin nicht spezifiziert.

- `tg_trigger`
  - Ein Zeiger auf eine Struktur vom Typ `Trigger`, definiert in `utils/reltrigger.h`:

    ```c
    typedef struct Trigger
    {
        Oid     tgoid;
        char   *tgname;
        Oid     tgfoid;
        int16   tgtype;
        char    tgenabled;
        bool    tgisinternal;
        bool    tgisclone;
        Oid     tgconstrrelid;
        Oid     tgconstrindid;
        Oid     tgconstraint;
        bool    tgdeferrable;
        bool    tginitdeferred;
        int16   tgnargs;
        int16   tgnattr;
        int16  *tgattr;
        char  **tgargs;
        char   *tgqual;
        char   *tgoldtable;
        char   *tgnewtable;
    } Trigger;
    ```

    Dabei ist `tgname` der Name des Triggers, `tgnargs` die Anzahl der Argumente in `tgargs`, und `tgargs` ist ein Array von Zeigern auf die Argumente, die in der Anweisung `CREATE TRIGGER` angegeben wurden. Die anderen Mitglieder dienen nur der internen Verwendung.

- `tg_trigslot`
  - Der Slot, der `tg_trigtuple` enthält, oder ein `NULL`-Zeiger, wenn es kein solches Tupel gibt.

- `tg_newslot`
  - Der Slot, der `tg_newtuple` enthält, oder ein `NULL`-Zeiger, wenn es kein solches Tupel gibt.

- `tg_oldtable`
  - Ein Zeiger auf eine Struktur vom Typ `Tuplestorestate`, die null oder mehr Zeilen im durch `tg_relation` angegebenen Format enthält, oder ein `NULL`-Zeiger, wenn es keine Übergangsrelation `OLD TABLE` gibt.

- `tg_newtable`
  - Ein Zeiger auf eine Struktur vom Typ `Tuplestorestate`, die null oder mehr Zeilen im durch `tg_relation` angegebenen Format enthält, oder ein `NULL`-Zeiger, wenn es keine Übergangsrelation `NEW TABLE` gibt.

- `tg_updatedcols`
  - Bei `UPDATE`-Triggern eine Bitmap-Menge, die die Spalten angibt, die durch den auslösenden Befehl aktualisiert wurden. Generische Triggerfunktionen können dies zur Optimierung verwenden, indem sie Spalten, die nicht geändert wurden, nicht behandeln müssen.

    Um beispielsweise zu bestimmen, ob eine Spalte mit der Attributnummer `attnum` (1-basiert) Mitglied dieser Bitmap-Menge ist, rufen Sie Folgendes auf:

    ```c
    bms_is_member(attnum - FirstLowInvalidHeapAttributeNumber,
                  trigdata->tg_updatedcols)
    ```

    Bei anderen Triggern als `UPDATE`-Triggern ist dies `NULL`.

Damit über SPI ausgegebene Abfragen Übergangstabellen referenzieren können, siehe `SPI_register_trigger_data`.

Eine Triggerfunktion muss entweder einen `HeapTuple`-Zeiger oder einen `NULL`-Zeiger zurückgeben, nicht einen SQL-Nullwert; setzen Sie also nicht `isNull` auf wahr. Achten Sie darauf, je nach Situation entweder `tg_trigtuple` oder `tg_newtuple` zurückzugeben, wenn Sie die Zeile, auf die die Operation angewendet wird, nicht ändern möchten.

## 37.4. Ein vollständiges Triggerbeispiel

Hier ist ein sehr einfaches Beispiel für eine Triggerfunktion, die in C geschrieben ist. Beispiele für Trigger, die in prozeduralen Sprachen geschrieben sind, finden Sie in der Dokumentation der jeweiligen prozeduralen Sprachen.

Die Funktion `trigf` meldet die Anzahl der Zeilen in der Tabelle `ttest` und überspringt die eigentliche Operation, wenn der Befehl versucht, einen Nullwert in die Spalte `x` einzufügen. Der Trigger verhält sich also wie eine Nicht-Null-Beschränkung, bricht aber die Transaktion nicht ab.

Zuerst die Tabellendefinition:

```sql
CREATE TABLE ttest (
    x integer
);
```

Dies ist der Quellcode der Triggerfunktion:

```c
#include "postgres.h"
#include "fmgr.h"
#include "executor/spi.h"     /* this is what you need to work with SPI */
#include "commands/trigger.h" /* ... triggers ... */
#include "utils/rel.h"        /* ... and relations */

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(trigf);

Datum
trigf(PG_FUNCTION_ARGS)
{
    TriggerData *trigdata = (TriggerData *) fcinfo->context;
    TupleDesc   tupdesc;
    HeapTuple   rettuple;
    char       *when;
    bool        checknull = false;
    bool        isnull;
    int         ret, i;

    /* make sure it's called as a trigger at all */
    if (!CALLED_AS_TRIGGER(fcinfo))
        elog(ERROR, "trigf: not called by trigger manager");

    /* tuple to return to executor */
    if (TRIGGER_FIRED_BY_UPDATE(trigdata->tg_event))
        rettuple = trigdata->tg_newtuple;
    else
        rettuple = trigdata->tg_trigtuple;

    /* check for null values */
    if (!TRIGGER_FIRED_BY_DELETE(trigdata->tg_event)
        && TRIGGER_FIRED_BEFORE(trigdata->tg_event))
        checknull = true;

    if (TRIGGER_FIRED_BEFORE(trigdata->tg_event))
        when = "before";
    else
        when = "after ";

    tupdesc = trigdata->tg_relation->rd_att;

    /* connect to SPI manager */
    SPI_connect();

    /* get number of rows in table */
    ret = SPI_exec("SELECT count(*) FROM ttest", 0);

    if (ret < 0)
        elog(ERROR, "trigf (fired %s): SPI_exec returned %d", when, ret);

    /* count(*) returns int8, so be careful to convert */
    i = DatumGetInt64(SPI_getbinval(SPI_tuptable->vals[0],
                                    SPI_tuptable->tupdesc,
                                    1,
                                    &isnull));

    elog(INFO, "trigf (fired %s): there are %d rows in ttest", when, i);

    SPI_finish();

    if (checknull)
    {
        SPI_getbinval(rettuple, tupdesc, 1, &isnull);
        if (isnull)
            rettuple = NULL;
    }

    return PointerGetDatum(rettuple);
}
```

Nachdem Sie den Quellcode kompiliert haben (siehe [Abschnitt 36.10.5](36_SQL_erweitern.md#36105-dynamisch-geladene-funktionen-kompilieren-und-linken)), deklarieren Sie die Funktion und die Trigger:

```sql
CREATE FUNCTION trigf() RETURNS trigger
    AS 'filename'
    LANGUAGE C;

CREATE TRIGGER tbefore BEFORE INSERT OR UPDATE OR DELETE ON ttest
    FOR EACH ROW EXECUTE FUNCTION trigf();

CREATE TRIGGER tafter AFTER INSERT OR UPDATE OR DELETE ON ttest
    FOR EACH ROW EXECUTE FUNCTION trigf();
```

Nun können Sie die Arbeitsweise des Triggers testen:

```text
=> INSERT INTO ttest VALUES (NULL);
INFO: trigf (fired before): there are 0 rows in ttest
INSERT 0 0

-- Insertion skipped and AFTER trigger is not fired

=> SELECT * FROM ttest;
---
(0 rows)

=> INSERT INTO ttest VALUES (1);
INFO:   trigf (fired before): there are 0 rows in ttest
INFO:   trigf (fired after ): there are 1 rows in ttest
                                        ^^^^^^^^
                              remember what we said about
                              visibility.
INSERT 167793 1

vac=> SELECT * FROM ttest;
---
(1 row)

=> INSERT INTO ttest SELECT x * 2 FROM ttest;
INFO: trigf (fired before): there are 1 rows in ttest
INFO: trigf (fired after ): there are 2 rows in ttest
                                       ^^^^^^
                             remember what we said about
                             visibility.
INSERT 167794 1

=> SELECT * FROM ttest;
---
(2 rows)

=> UPDATE ttest SET x = NULL WHERE x = 2;
INFO: trigf (fired before): there are 2 rows in ttest
UPDATE 0

=> UPDATE ttest SET x = 4 WHERE x = 2;
INFO: trigf (fired before): there are 2 rows in ttest
INFO: trigf (fired after ): there are 2 rows in ttest
UPDATE 1

vac=> SELECT * FROM ttest;
---
(2 rows)

=> DELETE FROM ttest;
INFO: trigf (fired before): there are 2 rows in ttest
INFO: trigf (fired before): there are 1 rows in ttest
INFO: trigf (fired after ): there are 0 rows in ttest
INFO: trigf (fired after ): there are 0 rows in ttest
                                      ^^^^^^
                            remember what we said about
                            visibility.
DELETE 2

=> SELECT * FROM ttest;
---
(0 rows)
```

Weitere komplexere Beispiele finden Sie in `src/test/regress/regress.c` und in `spi`.
