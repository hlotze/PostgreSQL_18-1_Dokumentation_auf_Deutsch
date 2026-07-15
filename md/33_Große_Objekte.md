# 33. Große Objekte

PostgreSQL besitzt eine Large-Object-Funktionalität, die streamartigen Zugriff auf Benutzerdaten ermöglicht, die in einer speziellen Large-Object-Struktur gespeichert werden. Streaming-Zugriff ist nützlich, wenn Datenwerte zu groß sind, um bequem als Ganzes verarbeitet zu werden.

Dieses Kapitel beschreibt die Implementierung sowie die Programmier- und SQL-Schnittstellen für Large-Object-Daten in PostgreSQL. Die Beispiele in diesem Kapitel verwenden die C-Bibliothek `libpq`, aber die meisten nativen PostgreSQL-Programmierschnittstellen unterstützen entsprechende Funktionalität. Andere Schnittstellen können die Large-Object-Schnittstelle intern verwenden, um generische Unterstützung für große Werte bereitzustellen; das wird hier nicht beschrieben.

## 33.1. Einführung

Alle Large Objects werden in einer einzelnen Systemtabelle namens `pg_largeobject` gespeichert. Zusätzlich hat jedes Large Object einen Eintrag in der Systemtabelle `pg_largeobject_metadata`. Large Objects können über eine Lese-/Schreib-API erzeugt, geändert und gelöscht werden, die den üblichen Dateioperationen ähnelt.

PostgreSQL unterstützt außerdem ein Speichersystem namens „TOAST“, das Werte, die größer als eine einzelne Datenbankseite sind, automatisch in einen sekundären Speicherbereich pro Tabelle auslagert. Dadurch ist die Large-Object-Funktionalität teilweise überholt. Ein verbleibender Vorteil von Large Objects ist, dass sie Werte bis zu 4 TB Größe erlauben, während TOAST-Felder höchstens 1 GB groß sein können. Außerdem können Teile eines Large Objects effizient gelesen und aktualisiert werden, während die meisten Operationen auf einem TOAST-Feld den gesamten Wert als Einheit lesen oder schreiben.

## 33.2. Implementierungsmerkmale

Die Large-Object-Implementierung zerlegt große Objekte in „Chunks“ und speichert diese Chunks als Zeilen in der Datenbank. Ein B-Tree-Index sorgt bei wahlfreien Lese- und Schreibzugriffen für schnelle Suche nach der richtigen Chunk-Nummer.

Die gespeicherten Chunks eines Large Objects müssen nicht zusammenhängend sein. Wenn eine Anwendung beispielsweise ein neues Large Object öffnet, zu Offset `1000000` springt und dort einige Bytes schreibt, werden dadurch nicht automatisch 1000000 Bytes Speicher belegt. Es werden nur Chunks für den tatsächlich geschriebenen Datenbereich angelegt. Ein Lesevorgang liefert für nicht zugewiesene Positionen vor dem letzten vorhandenen Chunk jedoch Nullbytes zurück. Das entspricht dem üblichen Verhalten dünn belegter Dateien in Unix-Dateisystemen.

Seit PostgreSQL 9.0 haben Large Objects einen Eigentümer und Zugriffsberechtigungen, die mit `GRANT` und `REVOKE` verwaltet werden können. Zum Lesen eines Large Objects ist das Privileg `SELECT` erforderlich, zum Schreiben oder Abschneiden `UPDATE`. Nur der Eigentümer des Large Objects oder ein Datenbank-Superuser kann ein Large Object löschen, kommentieren oder seinen Eigentümer ändern. Für Kompatibilität mit älteren Versionen siehe den Laufzeitparameter `lo_compat_privileges`.

## 33.3. Client-Schnittstellen

Dieser Abschnitt beschreibt die Funktionen, die die PostgreSQL-Clientbibliothek `libpq` für den Zugriff auf Large Objects bereitstellt. Die PostgreSQL-Large-Object-Schnittstelle ist der Unix-Dateisystemschnittstelle nachempfunden, mit Entsprechungen zu `open`, `read`, `write`, `lseek` usw.

Alle Manipulationen von Large Objects mit diesen Funktionen müssen innerhalb eines SQL-Transaktionsblocks stattfinden, da Large-Object-Dateideskriptoren nur für die Dauer einer Transaktion gültig sind. Schreiboperationen, einschließlich `lo_open` mit dem Modus `INV_WRITE`, sind in einer schreibgeschützten Transaktion nicht erlaubt.

Tritt beim Ausführen einer dieser Funktionen ein Fehler auf, gibt die Funktion einen sonst unmöglichen Wert zurück, typischerweise `0` oder `-1`. Eine Fehlermeldung wird im Verbindungsobjekt gespeichert und kann mit `PQerrorMessage` abgerufen werden.

Client-Anwendungen, die diese Funktionen verwenden, sollten die Headerdatei `libpq/libpq-fs.h` einbinden und gegen die Bibliothek `libpq` linken. Während sich eine `libpq`-Verbindung im Pipeline-Modus befindet, können diese Funktionen nicht verwendet werden.

### 33.3.1. Ein Large Object erzeugen

Die Funktion

```c
Oid lo_create(PGconn *conn, Oid lobjId);
```

erzeugt ein neues Large Object. Die zu verwendende OID kann mit `lobjId` angegeben werden; ist diese OID bereits für ein Large Object vergeben, schlägt der Aufruf fehl. Ist `lobjId` `InvalidOid` (`0`), weist `lo_create` eine freie OID zu. Der Rückgabewert ist die dem neuen Large Object zugewiesene OID oder `InvalidOid` (`0`) bei einem Fehler.

Beispiel:

```c
inv_oid = lo_create(conn, desired_oid);
```

Die ältere Funktion

```c
Oid lo_creat(PGconn *conn, int mode);
```

erzeugt ebenfalls ein neues Large Object und weist immer eine freie OID zu. Der Rückgabewert ist die zugewiesene OID oder `InvalidOid` (`0`) bei einem Fehler.

In PostgreSQL 8.1 und neuer wird `mode` ignoriert, sodass `lo_creat` genau `lo_create` mit `0` als zweitem Argument entspricht. Es gibt kaum einen Grund, `lo_creat` zu verwenden, außer wenn Sie mit Servern vor 8.1 arbeiten müssen. Bei solchen alten Servern muss `mode` auf `INV_READ`, `INV_WRITE` oder `INV_READ | INV_WRITE` gesetzt werden. Diese symbolischen Konstanten sind in `libpq/libpq-fs.h` definiert.

Beispiel:

```c
inv_oid = lo_creat(conn, INV_READ | INV_WRITE);
```

### 33.3.2. Ein Large Object importieren

Um eine Betriebssystemdatei als Large Object zu importieren, rufen Sie auf:

```c
Oid lo_import(PGconn *conn, const char *filename);
```

`filename` gibt den Betriebssystemnamen der Datei an, die als Large Object importiert werden soll. Der Rückgabewert ist die dem neuen Large Object zugewiesene OID oder `InvalidOid` (`0`) bei einem Fehler. Beachten Sie, dass die Datei von der Client-Schnittstellenbibliothek gelesen wird, nicht vom Server. Sie muss daher im Dateisystem des Clients vorhanden und für die Client-Anwendung lesbar sein.

Die Funktion

```c
Oid lo_import_with_oid(PGconn *conn, const char *filename, Oid lobjId);
```

importiert ebenfalls ein neues Large Object. Die zuzuweisende OID kann mit `lobjId` angegeben werden; ist diese OID bereits für ein Large Object vergeben, schlägt der Aufruf fehl. Ist `lobjId` `InvalidOid` (`0`), weist `lo_import_with_oid` eine freie OID zu. Der Rückgabewert ist die zugewiesene OID oder `InvalidOid` (`0`) bei einem Fehler.

`lo_import_with_oid` gibt es seit PostgreSQL 8.4 und verwendet intern `lo_create`, das seit 8.1 vorhanden ist. Gegen Server der Version 8.0 oder älter schlägt diese Funktion fehl und gibt `InvalidOid` zurück.

### 33.3.3. Ein Large Object exportieren

Um ein Large Object in eine Betriebssystemdatei zu exportieren, rufen Sie auf:

```c
int lo_export(PGconn *conn, Oid lobjId, const char *filename);
```

`lobjId` gibt die OID des zu exportierenden Large Objects an, `filename` den Betriebssystemnamen der Datei. Die Datei wird von der Client-Schnittstellenbibliothek geschrieben, nicht vom Server. Bei Erfolg wird `1` zurückgegeben, bei einem Fehler `-1`.

### 33.3.4. Ein vorhandenes Large Object öffnen

Um ein vorhandenes Large Object zum Lesen oder Schreiben zu öffnen, rufen Sie auf:

```c
int lo_open(PGconn *conn, Oid lobjId, int mode);
```

`lobjId` gibt die OID des zu öffnenden Large Objects an. Die Mode-Bits steuern, ob das Objekt zum Lesen (`INV_READ`), Schreiben (`INV_WRITE`) oder beidem geöffnet wird. `lo_open` gibt einen nichtnegativen Large-Object-Deskriptor zurück, der später mit `lo_read`, `lo_write`, `lo_lseek`, `lo_lseek64`, `lo_tell`, `lo_tell64`, `lo_truncate`, `lo_truncate64` und `lo_close` verwendet wird. Der Deskriptor ist nur für die Dauer der aktuellen Transaktion gültig. Bei einem Fehler wird `-1` zurückgegeben.

Der Server unterscheidet derzeit nicht zwischen `INV_WRITE` und `INV_READ | INV_WRITE`: In beiden Fällen darf vom Deskriptor gelesen werden. Zwischen diesen Modi und reinem `INV_READ` gibt es jedoch einen wichtigen Unterschied. Mit `INV_READ` kann nicht geschrieben werden, und gelesene Daten spiegeln den Inhalt des Large Objects zum Zeitpunkt des Transaktions-Snapshots wider, der beim Ausführen von `lo_open` aktiv war. Lesevorgänge von einem mit `INV_WRITE` geöffneten Deskriptor spiegeln auch Schreibvorgänge anderer bereits festgeschriebener Transaktionen sowie Schreibvorgänge der aktuellen Transaktion wider. Das ähnelt dem Unterschied zwischen `REPEATABLE READ` und `READ COMMITTED` bei normalen SQL-`SELECT`-Befehlen.

`lo_open` schlägt fehl, wenn für das Large Object kein `SELECT`-Privileg verfügbar ist oder wenn `INV_WRITE` angegeben wurde und kein `UPDATE`-Privileg verfügbar ist. Diese Privilegprüfungen können mit dem Laufzeitparameter `lo_compat_privileges` deaktiviert werden.

Beispiel:

```c
inv_fd = lo_open(conn, inv_oid, INV_READ | INV_WRITE);
```

### 33.3.5. Daten in ein Large Object schreiben

Die Funktion

```c
int lo_write(PGconn *conn, int fd, const char *buf, size_t len);
```

schreibt `len` Bytes aus `buf` in den Large-Object-Deskriptor `fd`. `fd` muss von einem vorherigen `lo_open` zurückgegeben worden sein. Zurückgegeben wird die Anzahl der tatsächlich geschriebenen Bytes; in der aktuellen Implementierung entspricht sie bei Erfolg immer `len`. Bei einem Fehler wird `-1` zurückgegeben.

Obwohl `len` als `size_t` deklariert ist, weist diese Funktion Längenwerte größer als `INT_MAX` zurück. Praktisch ist es ohnehin am besten, Daten in Chunks von höchstens einigen Megabytes zu übertragen.

### 33.3.6. Daten aus einem Large Object lesen

Die Funktion

```c
int lo_read(PGconn *conn, int fd, char *buf, size_t len);
```

liest bis zu `len` Bytes aus dem Large-Object-Deskriptor `fd` in `buf`. `fd` muss von einem vorherigen `lo_open` zurückgegeben worden sein. Zurückgegeben wird die Anzahl der tatsächlich gelesenen Bytes; sie ist kleiner als `len`, wenn zuerst das Ende des Large Objects erreicht wird. Bei einem Fehler wird `-1` zurückgegeben.

Auch hier weist die Funktion trotz `size_t`-Deklaration Längenwerte größer als `INT_MAX` zurück. In der Praxis sollten Daten in Chunks von höchstens einigen Megabytes übertragen werden.

### 33.3.7. In einem Large Object positionieren

Um die aktuelle Lese- oder Schreibposition eines Large-Object-Deskriptors zu ändern, rufen Sie auf:

```c
int lo_lseek(PGconn *conn, int fd, int offset, int whence);
```

Diese Funktion verschiebt den Positionszeiger des durch `fd` identifizierten Large-Object-Deskriptors an die durch `offset` angegebene neue Position. Gültige Werte für `whence` sind `SEEK_SET` (vom Objektanfang), `SEEK_CUR` (von der aktuellen Position) und `SEEK_END` (vom Objektende). Der Rückgabewert ist die neue Position oder `-1` bei einem Fehler.

Bei Large Objects, die größer als 2 GB werden können, verwenden Sie stattdessen:

```c
int64_t lo_lseek64(PGconn *conn, int fd, int64_t offset, int whence);
```

Diese Funktion verhält sich wie `lo_lseek`, kann aber Offsets und Ergebnisse größer als 2 GB verarbeiten. `lo_lseek` schlägt fehl, wenn die neue Position größer als 2 GB wäre. `lo_lseek64` gibt es seit PostgreSQL 9.3; gegen ältere Server schlägt die Funktion fehl und gibt `-1` zurück.

### 33.3.8. Die aktuelle Position eines Large Objects ermitteln

Um die aktuelle Lese- oder Schreibposition eines Large-Object-Deskriptors zu ermitteln, rufen Sie auf:

```c
int lo_tell(PGconn *conn, int fd);
```

Bei einem Fehler wird `-1` zurückgegeben. Bei Large Objects, die größer als 2 GB werden können, verwenden Sie stattdessen:

```c
int64_t lo_tell64(PGconn *conn, int fd);
```

Diese Funktion verhält sich wie `lo_tell`, kann aber Ergebnisse größer als 2 GB liefern. `lo_tell` schlägt fehl, wenn die aktuelle Lese-/Schreibposition größer als 2 GB ist. `lo_tell64` gibt es seit PostgreSQL 9.3; gegen ältere Server schlägt die Funktion fehl und gibt `-1` zurück.

### 33.3.9. Ein Large Object abschneiden

Um ein Large Object auf eine bestimmte Länge abzuschneiden, rufen Sie auf:

```c
int lo_truncate(PGconn *conn, int fd, size_t len);
```

Diese Funktion schneidet den Large-Object-Deskriptor `fd` auf die Länge `len` ab. `fd` muss von einem vorherigen `lo_open` zurückgegeben worden sein. Ist `len` größer als die aktuelle Länge des Large Objects, wird das Objekt mit Nullbytes (`'\0'`) auf die angegebene Länge erweitert. Bei Erfolg gibt `lo_truncate` null zurück, bei einem Fehler `-1`.

Die mit `fd` verbundene Lese-/Schreibposition wird nicht geändert. Obwohl `len` als `size_t` deklariert ist, weist `lo_truncate` Längenwerte größer als `INT_MAX` zurück.

Bei Large Objects, die größer als 2 GB werden können, verwenden Sie stattdessen:

```c
int lo_truncate64(PGconn *conn, int fd, int64_t len);
```

Diese Funktion verhält sich wie `lo_truncate`, kann aber einen `len`-Wert größer als 2 GB verarbeiten. `lo_truncate` gibt es seit PostgreSQL 8.3; gegen ältere Server schlägt die Funktion fehl und gibt `-1` zurück. `lo_truncate64` gibt es seit PostgreSQL 9.3; gegen ältere Server schlägt die Funktion ebenfalls fehl und gibt `-1` zurück.

### 33.3.10. Einen Large-Object-Deskriptor schließen

Ein Large-Object-Deskriptor kann mit folgender Funktion geschlossen werden:

```c
int lo_close(PGconn *conn, int fd);
```

`fd` ist ein von `lo_open` zurückgegebener Large-Object-Deskriptor. Bei Erfolg gibt `lo_close` null zurück, bei einem Fehler `-1`. Large-Object-Deskriptoren, die am Ende einer Transaktion noch offen sind, werden automatisch geschlossen.

### 33.3.11. Ein Large Object entfernen

Um ein Large Object aus der Datenbank zu entfernen, rufen Sie auf:

```c
int lo_unlink(PGconn *conn, Oid lobjId);
```

`lobjId` gibt die OID des zu entfernenden Large Objects an. Bei Erfolg wird `1` zurückgegeben, bei einem Fehler `-1`.

## 33.4. Serverseitige Funktionen

Serverseitige Funktionen, die auf die Manipulation von Large Objects aus SQL zugeschnitten sind, sind in Tabelle 33.1 aufgeführt.

**Tabelle 33.1. SQL-orientierte Large-Object-Funktionen**

| Funktion | Beschreibung | Beispiel |
| --- | --- | --- |
| `lo_from_bytea(loid oid, data bytea) → oid` | Erzeugt ein Large Object und speichert `data` darin. Ist `loid` null, wählt das System eine freie OID, andernfalls wird diese OID verwendet; ein Fehler tritt auf, wenn bereits ein Large Object mit dieser OID existiert. Bei Erfolg wird die OID des Large Objects zurückgegeben. | `lo_from_bytea(0, '\xffffff00') → 24528` |
| `lo_put(loid oid, offset bigint, data bytea) → void` | Schreibt `data` ab dem angegebenen Offset in das Large Object; das Objekt wird bei Bedarf vergrößert. | `lo_put(24528, 1, '\xaa') →` |
| `lo_get(loid oid [, offset bigint, length integer]) → bytea` | Extrahiert den Inhalt des Large Objects oder einen Teilstring davon. | `lo_get(24528, 0, 3) → \xffaaff` |

Es gibt weitere serverseitige Funktionen, die den zuvor beschriebenen clientseitigen Funktionen entsprechen. Tatsächlich sind die clientseitigen Funktionen größtenteils nur Schnittstellen zu den entsprechenden serverseitigen Funktionen. Besonders bequem direkt über SQL aufzurufen sind `lo_creat`, `lo_create`, `lo_unlink`, `lo_import` und `lo_export`. Beispiele:

```sql
CREATE TABLE image (
    name             text,
    raster           oid
);

SELECT lo_creat(-1);

SELECT lo_create(43213);

SELECT lo_unlink(173454);

INSERT INTO image (name, raster)
    VALUES ('beautiful image', lo_import('/etc/motd'));

INSERT INTO image (name, raster)
    VALUES ('beautiful image', lo_import('/etc/motd', 68583));

SELECT lo_export(image.raster, '/tmp/motd') FROM image
    WHERE name = 'beautiful image';
```

Die serverseitigen Funktionen `lo_import` und `lo_export` verhalten sich deutlich anders als ihre clientseitigen Gegenstücke. Diese beiden Funktionen lesen und schreiben Dateien im Dateisystem des Servers und verwenden dabei die Berechtigungen des Datenbankbesitzers. Deshalb ist ihre Verwendung standardmäßig auf Superuser beschränkt. Die clientseitigen Import- und Exportfunktionen dagegen lesen und schreiben Dateien im Dateisystem des Clients mit den Berechtigungen des Clientprogramms. Die clientseitigen Funktionen benötigen außer dem Privileg zum Lesen oder Schreiben des betreffenden Large Objects keine weiteren Datenbankprivilegien.

> **Achtung**
>
> Es ist möglich, Nicht-Superusern die Verwendung der serverseitigen Funktionen `lo_import` und `lo_export` per `GRANT` zu erlauben, aber die Sicherheitsfolgen müssen sorgfältig bedacht werden. Ein böswilliger Benutzer mit solchen Privilegien könnte daraus leicht Superuser-Rechte ableiten, etwa durch Umschreiben von Serverkonfigurationsdateien, oder das Dateisystem des Servers angreifen, ohne formale Datenbank-Superuser-Rechte zu besitzen. Zugriff auf Rollen mit solchen Privilegien muss daher ebenso sorgfältig geschützt werden wie Zugriff auf Superuser-Rollen. Wenn `lo_import` oder `lo_export` serverseitig für Routineaufgaben benötigt werden, ist eine Rolle mit genau diesen Privilegien dennoch sicherer als eine Rolle mit vollständigen Superuser-Rechten, weil das Risiko unbeabsichtigter Schäden sinkt.

Die Funktionalität von `lo_read` und `lo_write` ist ebenfalls über serverseitige Aufrufe verfügbar. Die Namen der serverseitigen Funktionen unterscheiden sich jedoch von den clientseitigen Schnittstellen dadurch, dass sie keine Unterstriche enthalten: Sie müssen als `loread` und `lowrite` aufgerufen werden.

## 33.5. Beispielprogramm

Beispiel 33.1 zeigt ein Beispielprogramm, das die Large-Object-Schnittstelle in `libpq` verwendet. Teile des Programms sind auskommentiert, bleiben aber zum Nutzen des Lesers im Quelltext. Dieses Programm befindet sich auch als `src/test/examples/testlo.c` in der Quellcodedistribution.

**Beispiel 33.1. Large Objects mit `libpq`**

```c
/*-----------------------------------------------------------------
 *
 * testlo.c
 *     Test der Verwendung von Large Objects mit libpq.
 *
 * Portions Copyright (c) 1996-2025, PostgreSQL Global Development Group
 * Portions Copyright (c) 1994, Regents of the University of California
 *
 * IDENTIFICATION
 *     src/test/examples/testlo.c
 *
 *-----------------------------------------------------------------
 */
#include <stdio.h>
#include <stdlib.h>

#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

#include "libpq-fe.h"
#include "libpq/libpq-fs.h"

#define BUFSIZE 1024

/*
 * importFile -
 *    Datei "filename" als Large Object in die Datenbank importieren.
 */
static Oid
importFile(PGconn *conn, char *filename)
{
    Oid         lobjId;
    int         lobj_fd;
    char        buf[BUFSIZE];
    int         nbytes,
                tmp;
    int         fd;

    /* Zu lesende Datei öffnen. */
    fd = open(filename, O_RDONLY, 0666);
    if (fd < 0)
        fprintf(stderr, "cannot open unix file \"%s\"\n", filename);

    /* Large Object erzeugen. */
    lobjId = lo_creat(conn, INV_READ | INV_WRITE);
    if (lobjId == 0)
        fprintf(stderr, "cannot create large object");

    lobj_fd = lo_open(conn, lobjId, INV_WRITE);

    /* Aus der Unix-Datei lesen und in das Large Object schreiben. */
    while ((nbytes = read(fd, buf, BUFSIZE)) > 0)
    {
        tmp = lo_write(conn, lobj_fd, buf, nbytes);
        if (tmp < nbytes)
            fprintf(stderr, "error while reading \"%s\"", filename);
    }

    close(fd);
    lo_close(conn, lobj_fd);

    return lobjId;
}

static void
pickout(PGconn *conn, Oid lobjId, int start, int len)
{
    int         lobj_fd;
    char       *buf;
    int         nbytes;
    int         nread;

    lobj_fd = lo_open(conn, lobjId, INV_READ);
    if (lobj_fd < 0)
        fprintf(stderr, "cannot open large object %u", lobjId);

    lo_lseek(conn, lobj_fd, start, SEEK_SET);
    buf = malloc(len + 1);

    nread = 0;
    while (len - nread > 0)
    {
        nbytes = lo_read(conn, lobj_fd, buf, len - nread);
        buf[nbytes] = '\0';
        fprintf(stderr, ">>> %s", buf);
        nread += nbytes;
        if (nbytes <= 0)
            break;              /* keine weiteren Daten? */
    }
    free(buf);
    fprintf(stderr, "\n");
    lo_close(conn, lobj_fd);
}

static void
overwrite(PGconn *conn, Oid lobjId, int start, int len)
{
    int         lobj_fd;
    char       *buf;
    int         nbytes;
    int         nwritten;
    int         i;

    lobj_fd = lo_open(conn, lobjId, INV_WRITE);
    if (lobj_fd < 0)
        fprintf(stderr, "cannot open large object %u", lobjId);

    lo_lseek(conn, lobj_fd, start, SEEK_SET);
    buf = malloc(len + 1);

    for (i = 0; i < len; i++)
        buf[i] = 'X';
    buf[i] = '\0';

    nwritten = 0;
    while (len - nwritten > 0)
    {
        nbytes = lo_write(conn, lobj_fd, buf + nwritten, len - nwritten);
        nwritten += nbytes;
        if (nbytes <= 0)
        {
            fprintf(stderr, "\nWRITE FAILED!\n");
            break;
        }
    }
    free(buf);
    fprintf(stderr, "\n");

    lo_close(conn, lobj_fd);
}

/*
 * exportFile -
 *    Large Object "lobjId" in Datei "filename" exportieren.
 */
static void
exportFile(PGconn *conn, Oid lobjId, char *filename)
{
    int         lobj_fd;
    char        buf[BUFSIZE];
    int         nbytes,
                tmp;
    int         fd;

    /* Large Object öffnen. */
    lobj_fd = lo_open(conn, lobjId, INV_READ);
    if (lobj_fd < 0)
        fprintf(stderr, "cannot open large object %u", lobjId);

    /* Zu schreibende Datei öffnen. */
    fd = open(filename, O_CREAT | O_WRONLY | O_TRUNC, 0666);
    if (fd < 0)
        fprintf(stderr, "cannot open unix file \"%s\"", filename);

    /* Aus dem Large Object lesen und in die Unix-Datei schreiben. */
    while ((nbytes = lo_read(conn, lobj_fd, buf, BUFSIZE)) > 0)
    {
        tmp = write(fd, buf, nbytes);
        if (tmp < nbytes)
            fprintf(stderr, "error while writing \"%s\"", filename);
    }

    lo_close(conn, lobj_fd);
    close(fd);
}

static void
exit_nicely(PGconn *conn)
{
    PQfinish(conn);
    exit(1);
}

int
main(int argc, char **argv)
{
    char       *in_filename,
               *out_filename;
    char       *database;
    Oid         lobjOid;
    PGconn     *conn;
    PGresult   *res;

    if (argc != 4)
    {
        fprintf(stderr, "Usage: %s database_name in_filename out_filename\n",
                argv[0]);
        exit(1);
    }

    database = argv[1];
    in_filename = argv[2];
    out_filename = argv[3];

    /* Verbindung aufbauen. */
    conn = PQsetdb(NULL, NULL, NULL, NULL, database);

    /* Prüfen, ob die Verbindung erfolgreich hergestellt wurde. */
    if (PQstatus(conn) != CONNECTION_OK)
    {
        fprintf(stderr, "%s", PQerrorMessage(conn));
        exit_nicely(conn);
    }

    /* Sicheren Suchpfad setzen, damit keine fremden Objekte bevorzugt werden. */
    res = PQexec(conn,
                 "SELECT pg_catalog.set_config('search_path', '', false)");
    if (PQresultStatus(res) != PGRES_TUPLES_OK)
    {
        fprintf(stderr, "SET failed: %s", PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }
    PQclear(res);

    res = PQexec(conn, "begin");
    PQclear(res);
    printf("importing file \"%s\" ...\n", in_filename);
/*  lobjOid = importFile(conn, in_filename); */
    lobjOid = lo_import(conn, in_filename);
    if (lobjOid == 0)
        fprintf(stderr, "%s\n", PQerrorMessage(conn));
    else
    {
        printf("\tas large object %u.\n", lobjOid);

        printf("picking out bytes 1000-2000 of the large object\n");
        pickout(conn, lobjOid, 1000, 1000);

        printf("overwriting bytes 1000-2000 of the large object with X's\n");
        overwrite(conn, lobjOid, 1000, 1000);

        printf("exporting large object to file \"%s\" ...\n", out_filename);
/*      exportFile(conn, lobjOid, out_filename); */
        if (lo_export(conn, lobjOid, out_filename) < 0)
            fprintf(stderr, "%s\n", PQerrorMessage(conn));
    }

    res = PQexec(conn, "end");
    PQclear(res);
    PQfinish(conn);
    return 0;
}
```
