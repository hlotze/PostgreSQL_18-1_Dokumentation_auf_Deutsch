# 62. Schnittstellendefinition für Table Access Methods

Dieses Kapitel erklärt die Schnittstelle zwischen dem PostgreSQL-Kernsystem und Table Access Methods, die den Speicher für Tabellen verwalten. Das Kernsystem weiß über diese Zugriffsmethoden nur, was hier festgelegt ist. Daher ist es möglich, durch Zusatzcode vollständig neue Typen von Zugriffsmethoden zu entwickeln.

Jede Table Access Method wird durch eine Zeile im Systemkatalog `pg_am` beschrieben. Der `pg_am`-Eintrag gibt einen Namen und eine Handlerfunktion für die Table Access Method an. Diese Einträge können mit den SQL-Befehlen `CREATE ACCESS METHOD` und `DROP ACCESS METHOD` erzeugt und gelöscht werden.

Eine Handlerfunktion für eine Table Access Method muss so deklariert sein, dass sie ein einzelnes Argument vom Typ `internal` akzeptiert und den Pseudotyp `table_am_handler` zurückgibt. Das Argument ist ein Dummy-Wert, der lediglich verhindert, dass Handlerfunktionen direkt aus SQL-Befehlen aufgerufen werden.

So könnte eine SQL-Skriptdatei einer Erweiterung einen Handler für eine Table Access Method erzeugen:

```sql
CREATE OR REPLACE FUNCTION my_tableam_handler(internal)
  RETURNS table_am_handler AS 'my_extension', 'my_tableam_handler'
  LANGUAGE C STRICT;

CREATE ACCESS METHOD myam TYPE TABLE HANDLER my_tableam_handler;
```

Das Ergebnis der Funktion muss ein Zeiger auf eine Struktur vom Typ `TableAmRoutine` sein. Diese enthält alles, was der Kerncode wissen muss, um die Table Access Method zu verwenden. Der Rückgabewert muss Server-Lebensdauer haben; typischerweise erreicht man das, indem er als `static const`-Variable im globalen Gültigkeitsbereich definiert wird.

So könnte eine Quelldatei mit dem Handler der Table Access Method aussehen:

```c
#include "postgres.h"

#include "access/tableam.h"
#include "fmgr.h"

PG_MODULE_MAGIC;

static const TableAmRoutine my_tableam_methods = {
    .type = T_TableAmRoutine,

    /*
     * Methods of TableAmRoutine omitted from example, add them
     * here.
     */
};

PG_FUNCTION_INFO_V1(my_tableam_handler);

Datum
my_tableam_handler(PG_FUNCTION_ARGS)
{
    PG_RETURN_POINTER(&my_tableam_methods);
}
```

Die Struktur `TableAmRoutine`, auch API-Struktur der Zugriffsmethode genannt, definiert das Verhalten der Zugriffsmethode mithilfe von Callbacks. Diese Callbacks sind Zeiger auf einfache C-Funktionen und auf SQL-Ebene weder sichtbar noch aufrufbar. Alle Callbacks und ihr Verhalten sind in der Struktur `TableAmRoutine` definiert, wobei Kommentare innerhalb der Struktur die Anforderungen an die Callbacks beschreiben. Für die meisten Callbacks gibt es Wrapperfunktionen, die aus der Sicht eines Benutzers der Table Access Method dokumentiert sind, nicht aus der Sicht des Implementierers. Einzelheiten finden Sie in der Datei `src/include/access/tableam.h`.

Um eine Zugriffsmethode zu implementieren, muss ein Implementierer typischerweise einen AM-spezifischen Typ von Tuple-Table-Slot implementieren; siehe `src/include/executor/tuptable.h`. Dieser erlaubt Code außerhalb der Zugriffsmethode, Referenzen auf Tupel der AM zu halten und auf die Spalten des Tupels zuzugreifen.

Derzeit ist die tatsächliche Speicherung von Daten durch eine AM recht wenig eingeschränkt. Beispielsweise ist es möglich, aber nicht erforderlich, den Shared Buffer Cache von PostgreSQL zu verwenden. Falls er verwendet wird, ist es wahrscheinlich sinnvoll, das in [Abschnitt 66.6](66_Physische_Speicherung_der_Datenbank.md#666-layout-von-datenbankseiten) beschriebene Standardseitenlayout von PostgreSQL zu verwenden.

Eine recht große Einschränkung der API für Table Access Methods besteht derzeit darin, dass, wenn die AM Modifikationen und/oder Indizes unterstützen möchte, jedes Tupel einen Tupelbezeichner (TID) besitzen muss, der aus einer Blocknummer und einer Itemnummer besteht; siehe auch [Abschnitt 66.6](66_Physische_Speicherung_der_Datenbank.md#666-layout-von-datenbankseiten). Es ist nicht strikt erforderlich, dass die Teilbestandteile von TIDs dieselbe Bedeutung haben wie etwa bei `heap`. Wenn jedoch Bitmap-Scan-Unterstützung gewünscht ist, was optional ist, muss die Blocknummer Lokalität bereitstellen.

Für Crash-Sicherheit kann eine AM das WAL von PostgreSQL oder eine eigene Implementierung verwenden. Wird WAL gewählt, können entweder generische WAL-Records verwendet oder ein Custom WAL Resource Manager implementiert werden.

Um Transaktionsunterstützung so zu implementieren, dass innerhalb einer einzelnen Transaktion auf verschiedene Table Access Methods zugegriffen werden kann, ist wahrscheinlich eine enge Integration mit der Maschinerie in `src/backend/access/transam/xlog.c` erforderlich.

Entwickler einer neuen Table Access Method können für Details der Implementierung die bestehende Heap-Implementierung in `src/backend/access/heap/heapam_handler.c` heranziehen.

```text
https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/include/access/tableam.h;hb=HEAD
https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/include/executor/tuptable.h;hb=HEAD
```
