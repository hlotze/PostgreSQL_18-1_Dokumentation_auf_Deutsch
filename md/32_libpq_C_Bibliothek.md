# 32. libpq - C-Bibliothek

`libpq` ist die C-Programmierschnittstelle für PostgreSQL-Anwendungen. `libpq` ist eine Sammlung von Bibliotheksfunktionen, mit denen Clientprogramme Abfragen an den PostgreSQL-Backend-Server senden und die Ergebnisse dieser Abfragen empfangen können.

`libpq` ist außerdem die zugrunde liegende Engine für mehrere andere PostgreSQL-Anwendungsschnittstellen, darunter solche für C++, Perl, Python, Tcl und ECPG. Daher sind einige Aspekte des `libpq`-Verhaltens auch wichtig, wenn Sie eines dieser Pakete verwenden. Insbesondere beschreiben [Abschnitt 32.15](32_libpq_C_Bibliothek.md#3215-umgebungsvariablen), [Abschnitt 32.16](32_libpq_C_Bibliothek.md#3216-die-passwortdatei) und [Abschnitt 32.19](32_libpq_C_Bibliothek.md#3219-sslunterstützung) Verhalten, das für Benutzer jeder Anwendung sichtbar ist, die `libpq` verwendet.

Am Ende dieses Kapitels sind einige kurze Programme enthalten ([Abschnitt 32.23](32_libpq_C_Bibliothek.md#3223-beispielprogramme)), die zeigen, wie Programme geschrieben werden, die `libpq` verwenden. Außerdem gibt es in der Quellcodedistribution mehrere vollständige Beispiele für `libpq`-Anwendungen im Verzeichnis `src/test/examples`.

Clientprogramme, die `libpq` verwenden, müssen die Header-Datei `libpq-fe.h` einbinden und gegen die Bibliothek `libpq` linken.

## 32.1. Funktionen zur Steuerung von Datenbankverbindungen

Die folgenden Funktionen behandeln den Aufbau einer Verbindung zu einem PostgreSQL-Backend-Server. Ein Anwendungsprogramm kann gleichzeitig mehrere Backend-Verbindungen offen haben. Ein Grund dafür ist etwa der Zugriff auf mehr als eine Datenbank. Jede Verbindung wird durch ein `PGconn`-Objekt repräsentiert, das von der Funktion `PQconnectdb`, `PQconnectdbParams` oder `PQsetdbLogin` geliefert wird. Beachten Sie, dass diese Funktionen immer einen Nicht-Null-Objektzeiger zurückgeben, außer möglicherweise dann, wenn selbst für die Zuweisung des `PGconn`-Objekts zu wenig Speicher vorhanden ist. Die Funktion `PQstatus` sollte aufgerufen werden, um vor dem Senden von Abfragen über das Verbindungsobjekt zu prüfen, ob der Rückgabewert eine erfolgreiche Verbindung darstellt.

> **Warnung**
>
> Wenn nicht vertrauenswürdige Benutzer Zugriff auf eine Datenbank haben, die kein sicheres Schema-Nutzungsmuster übernommen hat, beginnen Sie jede Sitzung damit, öffentlich beschreibbare Schemas aus `search_path` zu entfernen. Dazu kann man das Parameterschlüsselwort `options` auf den Wert `-csearch_path=` setzen. Alternativ kann man nach dem Verbindungsaufbau `PQexec(conn, "SELECT pg_catalog.set_config('search_path', '', false)")` ausführen. Diese Überlegung ist nicht `libpq`-spezifisch; sie gilt für jede Schnittstelle, über die beliebige SQL-Befehle ausgeführt werden.

> **Warnung**
>
> Unter Unix kann das Forken eines Prozesses mit offenen `libpq`-Verbindungen zu unvorhersehbaren Ergebnissen führen, weil Eltern- und Kindprozess dieselben Sockets und Betriebssystemressourcen gemeinsam nutzen. Deshalb wird diese Verwendung nicht empfohlen. Sicher ist hingegen ein `exec` aus dem Kindprozess heraus, um ein neues ausführbares Programm zu laden.

**`PQconnectdbParams`**

Stellt eine neue Verbindung zum Datenbankserver her.

```c
PGconn *PQconnectdbParams(const char * const *keywords,
                          const char * const *values,
                          int expand_dbname);
```

Diese Funktion öffnet eine neue Datenbankverbindung unter Verwendung der Parameter aus zwei `NULL`-terminierten Arrays. Das erste, `keywords`, ist als Array von Zeichenketten definiert, von denen jede ein Schlüsselwort ist. Das zweite, `values`, liefert den Wert zu jedem Schlüsselwort. Anders als bei `PQsetdbLogin` unten kann der Parametersatz erweitert werden, ohne die Funktionssignatur zu ändern. Daher wird diese Funktion, oder ihre nichtblockierenden Gegenstücke `PQconnectStartParams` und `PQconnectPoll`, für neue Anwendungsprogrammierung bevorzugt.

Die derzeit erkannten Parameterschlüsselwörter sind in Abschnitt 32.1.2 aufgeführt.

Die übergebenen Arrays können leer sein, um ausschließlich Standardparameter zu verwenden, oder eine oder mehrere Parametereinstellungen enthalten. Sie müssen gleich lang sein. Die Verarbeitung endet beim ersten `NULL`-Eintrag im Array `keywords`. Wenn außerdem der zu einem Nicht-`NULL`-Eintrag in `keywords` gehörende Eintrag in `values` `NULL` oder eine leere Zeichenkette ist, wird dieser Eintrag ignoriert und die Verarbeitung mit dem nächsten Array-Paar fortgesetzt.

Wenn `expand_dbname` ungleich null ist, wird der Wert des ersten Schlüsselworts `dbname` darauf geprüft, ob er ein Verbindungsstring ist. Falls ja, wird er in die einzelnen, aus der Zeichenkette extrahierten Verbindungsparameter „expandiert“. Der Wert gilt als Verbindungsstring und nicht nur als Datenbankname, wenn er ein Gleichheitszeichen (`=`) enthält oder mit einem URI-Schema-Bezeichner beginnt. Weitere Einzelheiten zu Verbindungsstring-Formaten finden Sie in Abschnitt 32.1.1. Nur das erste Auftreten von `dbname` wird auf diese Weise behandelt; jeder nachfolgende Parameter `dbname` wird als einfacher Datenbankname verarbeitet.

Im Allgemeinen werden die Parameterarrays von Anfang bis Ende verarbeitet. Wenn ein Schlüsselwort wiederholt wird, wird der letzte Wert verwendet, der nicht `NULL` und nicht leer ist. Diese Regel gilt insbesondere dann, wenn ein in einem Verbindungsstring gefundenes Schlüsselwort mit einem Schlüsselwort im Array `keywords` kollidiert. Der Programmierer kann also festlegen, ob Array-Einträge Werte aus einem Verbindungsstring überschreiben können oder von ihnen überschrieben werden. Array-Einträge, die vor einem expandierten `dbname`-Eintrag erscheinen, können durch Felder des Verbindungsstrings überschrieben werden; diese Felder wiederum werden durch Array-Einträge überschrieben, die nach `dbname` erscheinen, wiederum nur, wenn diese Einträge nichtleere Werte liefern.

Nachdem alle Array-Einträge und ein eventuell expandierter Verbindungsstring verarbeitet wurden, werden alle noch nicht gesetzten Verbindungsparameter mit Standardwerten gefüllt. Wenn die entsprechende Umgebungsvariable eines nicht gesetzten Parameters gesetzt ist (siehe [Abschnitt 32.15](32_libpq_C_Bibliothek.md#3215-umgebungsvariablen)), wird ihr Wert verwendet. Ist auch die Umgebungsvariable nicht gesetzt, wird der eingebaute Standardwert des Parameters verwendet.

**`PQconnectdb`**

Stellt eine neue Verbindung zum Datenbankserver her.

```c
PGconn *PQconnectdb(const char *conninfo);
```

Diese Funktion öffnet eine neue Datenbankverbindung unter Verwendung der Parameter aus der Zeichenkette `conninfo`.

Die übergebene Zeichenkette kann leer sein, um alle Standardparameter zu verwenden, oder sie kann eine oder mehrere durch Leerraum getrennte Parametereinstellungen enthalten, oder sie kann einen URI enthalten. Einzelheiten finden Sie in Abschnitt 32.1.1.

**`PQsetdbLogin`**

Stellt eine neue Verbindung zum Datenbankserver her.

```c
PGconn *PQsetdbLogin(const char *pghost,
                     const char *pgport,
                     const char *pgoptions,
                     const char *pgtty,
                     const char *dbName,
                     const char *login,
                     const char *pwd);
```

Dies ist der Vorgänger von `PQconnectdb` mit einem festen Parametersatz. Er bietet dieselbe Funktionalität, außer dass fehlende Parameter immer Standardwerte annehmen. Schreiben Sie `NULL` oder eine leere Zeichenkette für jeden der festen Parameter, der auf seinen Standardwert gesetzt werden soll.

Wenn `dbName` ein `=`-Zeichen enthält oder ein gültiges Verbindungs-URI-Präfix hat, wird er genau so als `conninfo`-String behandelt, als wäre er an `PQconnectdb` übergeben worden; die verbleibenden Parameter werden dann wie für `PQconnectdbParams` beschrieben angewendet.

`pgtty` wird nicht mehr verwendet, und jeder übergebene Wert wird ignoriert.

**`PQsetdb`**

Stellt eine neue Verbindung zum Datenbankserver her.

```c
PGconn *PQsetdb(char *pghost,
                char *pgport,
                char *pgoptions,
                char *pgtty,
                char *dbName);
```

Dies ist ein Makro, das `PQsetdbLogin` mit Nullzeigern für die Parameter `login` und `pwd` aufruft. Es wird aus Gründen der Abwärtskompatibilität mit sehr alten Programmen bereitgestellt.

**`PQconnectStartParams`**, **`PQconnectStart`**, **`PQconnectPoll`**

Stellen nichtblockierend eine Verbindung zum Datenbankserver her.

```c
PGconn *PQconnectStartParams(const char * const *keywords,
                             const char * const *values,
                             int expand_dbname);

PGconn *PQconnectStart(const char *conninfo);

PostgresPollingStatusType PQconnectPoll(PGconn *conn);
```

Diese drei Funktionen werden verwendet, um eine Verbindung zu einem Datenbankserver so zu öffnen, dass der Ausführungsthread Ihrer Anwendung dabei nicht durch entfernte Ein-/Ausgabe blockiert wird. Sinn dieses Ansatzes ist, dass Wartezeiten auf den Abschluss von Ein-/Ausgabe in der Hauptschleife der Anwendung stattfinden können, statt tief innerhalb von `PQconnectdbParams` oder `PQconnectdb`. So kann die Anwendung diesen Vorgang parallel zu anderen Aktivitäten verwalten.

Bei `PQconnectStartParams` wird die Datenbankverbindung mit den Parametern aus den Arrays `keywords` und `values` aufgebaut und durch `expand_dbname` gesteuert, wie oben für `PQconnectdbParams` beschrieben.

Bei `PQconnectStart` wird die Datenbankverbindung mit den Parametern aus der Zeichenkette `conninfo` aufgebaut, wie oben für `PQconnectdb` beschrieben.

Weder `PQconnectStartParams` noch `PQconnectStart` noch `PQconnectPoll` blockieren, solange mehrere Einschränkungen eingehalten werden:

- Der Parameter `hostaddr` muss geeignet verwendet werden, um DNS-Abfragen zu verhindern. Einzelheiten finden Sie in der Dokumentation dieses Parameters in Abschnitt 32.1.2.
- Wenn Sie `PQtrace` aufrufen, stellen Sie sicher, dass das Stream-Objekt, in das protokolliert wird, nicht blockiert.
- Sie müssen sicherstellen, dass sich der Socket im passenden Zustand befindet, bevor `PQconnectPoll` aufgerufen wird, wie unten beschrieben.

Um eine nichtblockierende Verbindungsanforderung zu beginnen, rufen Sie `PQconnectStart` oder `PQconnectStartParams` auf. Wenn das Ergebnis null ist, konnte `libpq` keine neue `PGconn`-Struktur zuweisen. Andernfalls wird ein gültiger `PGconn`-Zeiger zurückgegeben, der allerdings noch keine gültige Verbindung zur Datenbank repräsentiert. Rufen Sie anschließend `PQstatus(conn)` auf. Wenn das Ergebnis `CONNECTION_BAD` ist, ist der Verbindungsversuch bereits fehlgeschlagen, typischerweise wegen ungültiger Verbindungsparameter.

Wenn `PQconnectStart` oder `PQconnectStartParams` erfolgreich ist, besteht der nächste Schritt darin, `libpq` abzufragen, damit die Verbindungssequenz fortgesetzt werden kann. Verwenden Sie `PQsocket(conn)`, um den Deskriptor des Sockets zu erhalten, der der Datenbankverbindung zugrunde liegt. Achtung: Gehen Sie nicht davon aus, dass der Socket über mehrere `PQconnectPoll`-Aufrufe hinweg derselbe bleibt. Die Schleife funktioniert folgendermaßen: Wenn `PQconnectPoll(conn)` zuletzt `PGRES_POLLING_READING` zurückgegeben hat, warten Sie, bis der Socket lesebereit ist, wie durch `select()`, `poll()` oder eine ähnliche Systemfunktion angezeigt. Beachten Sie, dass `PQsocketPoll` Boilerplate reduzieren kann, indem es den Aufbau von `select(2)` oder `poll(2)` abstrahiert, sofern es auf Ihrem System verfügbar ist. Rufen Sie dann erneut `PQconnectPoll(conn)` auf. Umgekehrt gilt: Wenn `PQconnectPoll(conn)` zuletzt `PGRES_POLLING_WRITING` zurückgegeben hat, warten Sie, bis der Socket schreibbereit ist, und rufen dann wieder `PQconnectPoll(conn)` auf. In der ersten Iteration, also wenn Sie `PQconnectPoll` noch nicht aufgerufen haben, verhalten Sie sich so, als hätte der letzte Aufruf `PGRES_POLLING_WRITING` zurückgegeben. Führen Sie diese Schleife fort, bis `PQconnectPoll(conn)` `PGRES_POLLING_FAILED` zurückgibt, was ein Scheitern des Verbindungsaufbaus anzeigt, oder `PGRES_POLLING_OK`, was eine erfolgreich hergestellte Verbindung anzeigt.

Während des Verbindungsaufbaus kann der Status der Verbindung jederzeit durch Aufruf von `PQstatus` geprüft werden. Wenn dieser Aufruf `CONNECTION_BAD` zurückgibt, ist der Verbindungsaufbau fehlgeschlagen; wenn er `CONNECTION_OK` zurückgibt, ist die Verbindung bereit. Beide Zustände sind ebenso über den oben beschriebenen Rückgabewert von `PQconnectPoll` erkennbar. Während eines asynchronen Verbindungsaufbaus können außerdem weitere Zustände auftreten, und nur dann. Sie geben die aktuelle Phase des Verbindungsaufbaus an und können zum Beispiel nützlich sein, um dem Benutzer Rückmeldungen zu geben. Diese Zustände sind:

- `CONNECTION_STARTED`: Warten darauf, dass die Verbindung hergestellt wird.
- `CONNECTION_MADE`: Verbindung in Ordnung; Warten auf das Senden.
- `CONNECTION_AWAITING_RESPONSE`: Warten auf eine Antwort vom Server.
- `CONNECTION_AUTH_OK`: Authentifizierung empfangen; Warten darauf, dass der Backend-Start abgeschlossen wird.
- `CONNECTION_SSL_STARTUP`: SSL-Verschlüsselung wird ausgehandelt.
- `CONNECTION_GSS_STARTUP`: GSS-Verschlüsselung wird ausgehandelt.
- `CONNECTION_CHECK_WRITABLE`: Es wird geprüft, ob die Verbindung Schreibtransaktionen verarbeiten kann.
- `CONNECTION_CHECK_STANDBY`: Es wird geprüft, ob die Verbindung zu einem Server im Standby-Modus besteht.
- `CONNECTION_CONSUME`: Verbleibende Antwortmeldungen auf der Verbindung werden konsumiert.

Beachten Sie, dass diese Konstanten zwar erhalten bleiben, um Kompatibilität zu wahren, eine Anwendung sich aber niemals darauf verlassen sollte, dass sie in einer bestimmten Reihenfolge auftreten, überhaupt auftreten oder dass der Status immer einer dieser dokumentierten Werte ist. Eine Anwendung könnte etwa so vorgehen:

```c
switch(PQstatus(conn))
{
        case CONNECTION_STARTED:
            feedback = "Connecting...";
            break;

        case CONNECTION_MADE:
            feedback = "Connected to server...";
            break;
.
.
.
        default:
            feedback = "Connecting...";
}
```

Der Verbindungsparameter `connect_timeout` wird bei Verwendung von `PQconnectPoll` ignoriert; die Anwendung ist selbst dafür verantwortlich zu entscheiden, ob übermäßig viel Zeit verstrichen ist. Ansonsten ist `PQconnectStart` gefolgt von einer `PQconnectPoll`-Schleife äquivalent zu `PQconnectdb`.

Beachten Sie: Wenn `PQconnectStart` oder `PQconnectStartParams` einen Nicht-Null-Zeiger zurückgibt, müssen Sie `PQfinish` aufrufen, sobald Sie damit fertig sind, um die Struktur und alle zugehörigen Speicherblöcke freizugeben. Das muss auch dann geschehen, wenn der Verbindungsversuch fehlschlägt oder abgebrochen wird.

**`PQsocketPoll`**

Fragt den zugrunde liegenden Socket-Deskriptor einer Verbindung ab, der mit `PQsocket` ermittelt wurde. Die Hauptverwendung dieser Funktion besteht darin, die in der Dokumentation von `PQconnectStartParams` beschriebene Verbindungssequenz zu durchlaufen.

```c
typedef int64_t pg_usec_time_t;

int PQsocketPoll(int sock, int forRead, int forWrite,
                 pg_usec_time_t end_time);
```

Diese Funktion führt ein Polling eines Dateideskriptors aus, optional mit Timeout. Wenn `forRead` ungleich null ist, beendet sich die Funktion, sobald der Socket lesebereit ist. Wenn `forWrite` ungleich null ist, beendet sich die Funktion, sobald der Socket schreibbereit ist.

Das Timeout wird durch `end_time` angegeben. Das ist der Zeitpunkt, an dem das Warten beendet werden soll, ausgedrückt als Anzahl von Mikrosekunden seit der Unix-Epoche, also `time_t` mal 1 Million. Das Timeout ist unendlich, wenn `end_time` `-1` ist. Das Timeout ist sofort, also nichtblockierend, wenn `end_time` `0` ist, oder tatsächlich jeder Zeitpunkt vor jetzt. Timeout-Werte lassen sich bequem berechnen, indem die gewünschte Anzahl Mikrosekunden zum Ergebnis von `PQgetCurrentTimeUSec` addiert wird. Beachten Sie, dass die zugrunde liegenden Systemaufrufe eine geringere Präzision als eine Mikrosekunde haben können, sodass die tatsächliche Verzögerung ungenau sein kann.

Die Funktion gibt einen Wert größer als `0` zurück, wenn die angegebene Bedingung erfüllt ist, `0`, wenn ein Timeout aufgetreten ist, oder `-1`, wenn ein Fehler aufgetreten ist. Der Fehler kann durch Prüfung des Werts von `errno(3)` ermittelt werden. Falls sowohl `forRead` als auch `forWrite` null sind, gibt die Funktion sofort eine Timeout-Anzeige zurück.

`PQsocketPoll` ist je nach Plattform entweder mit `poll(2)` oder mit `select(2)` implementiert. Weitere Informationen finden Sie bei `POLLIN` und `POLLOUT` aus `poll(2)` beziehungsweise bei `readfds` und `writefds` aus `select(2)`.

**`PQconndefaults`**

Gibt die Standard-Verbindungsoptionen zurück.

```c
PQconninfoOption *PQconndefaults(void);

typedef struct
{
    char   *keyword;                /* The keyword of the option */
    char   *envvar;                 /* Fallback environment variable name */
    char   *compiled;               /* Fallback compiled in default value */
    char   *val;                    /* Option's current value, or NULL */
    char   *label;                  /* Label for field in connect dialog */
    char   *dispchar;               /* Indicates how to display this field
                                       in a connect dialog. Values are:
                                       ""        Display entered value as is
                                       "*"       Password field - hide value
                                       "D"       Debug option - don't show by
                                                 default */
    int     dispsize;               /* Field size in characters for dialog */
} PQconninfoOption;
```

Gibt ein Array von Verbindungsoptionen zurück. Dies kann verwendet werden, um alle möglichen `PQconnectdb`-Optionen und ihre aktuellen Standardwerte zu bestimmen. Der Rückgabewert zeigt auf ein Array von `PQconninfoOption`-Strukturen, das mit einem Eintrag endet, dessen `keyword`-Zeiger null ist. Der Nullzeiger wird zurückgegeben, wenn kein Speicher zugewiesen werden konnte. Beachten Sie, dass die aktuellen Standardwerte, also die `val`-Felder, von Umgebungsvariablen und anderem Kontext abhängen. Eine fehlende oder ungültige Service-Datei wird stillschweigend ignoriert. Aufrufer müssen die Daten der Verbindungsoptionen als schreibgeschützt behandeln.

Nach der Verarbeitung des Optionsarrays geben Sie es frei, indem Sie es an `PQconninfoFree` übergeben. Geschieht dies nicht, wird bei jedem Aufruf von `PQconndefaults` eine kleine Menge Speicher verloren.

**`PQconninfo`**

Gibt die von einer aktiven Verbindung verwendeten Verbindungsoptionen zurück.

```c
PQconninfoOption *PQconninfo(PGconn *conn);
```

Gibt ein Array von Verbindungsoptionen zurück. Dies kann verwendet werden, um alle möglichen `PQconnectdb`-Optionen und die Werte zu bestimmen, die für den Verbindungsaufbau zum Server verwendet wurden. Der Rückgabewert zeigt auf ein Array von `PQconninfoOption`-Strukturen, das mit einem Eintrag endet, dessen `keyword`-Zeiger null ist. Alle obigen Hinweise zu `PQconndefaults` gelten auch für das Ergebnis von `PQconninfo`.

**`PQconninfoParse`**

Gibt aus dem bereitgestellten Verbindungsstring geparste Verbindungsoptionen zurück.

```c
PQconninfoOption *PQconninfoParse(const char *conninfo, char **errmsg);
```

Parst einen Verbindungsstring und gibt die daraus resultierenden Optionen als Array zurück; bei einem Problem mit dem Verbindungsstring gibt die Funktion `NULL` zurück. Diese Funktion kann verwendet werden, um die `PQconnectdb`-Optionen aus dem bereitgestellten Verbindungsstring zu extrahieren. Der Rückgabewert zeigt auf ein Array von `PQconninfoOption`-Strukturen, das mit einem Eintrag endet, dessen `keyword`-Zeiger null ist.

Alle zulässigen Optionen sind im Ergebnisarray vorhanden, aber die `PQconninfoOption` für jede Option, die nicht im Verbindungsstring vorkommt, hat `val` auf `NULL` gesetzt; Standardwerte werden nicht eingefügt.

Wenn `errmsg` nicht `NULL` ist, wird `*errmsg` bei Erfolg auf `NULL` gesetzt, andernfalls auf eine mit `malloc` erzeugte Fehlerzeichenkette, die das Problem erklärt. Es ist auch möglich, dass `*errmsg` auf `NULL` gesetzt wird und die Funktion `NULL` zurückgibt; dies zeigt eine Speichermangelbedingung an.

Nach der Verarbeitung des Optionsarrays geben Sie es frei, indem Sie es an `PQconninfoFree` übergeben. Geschieht dies nicht, wird bei jedem Aufruf von `PQconninfoParse` Speicher verloren. Umgekehrt gilt: Wenn ein Fehler auftritt und `errmsg` nicht `NULL` ist, geben Sie die Fehlerzeichenkette mit `PQfreemem` frei.

**`PQfinish`**

Schließt die Verbindung zum Server. Außerdem wird der vom `PGconn`-Objekt verwendete Speicher freigegeben.

```c
void PQfinish(PGconn *conn);
```

Beachten Sie: Selbst wenn der Verbindungsversuch zum Server fehlschlägt, wie durch `PQstatus` angezeigt, sollte die Anwendung `PQfinish` aufrufen, um den vom `PGconn`-Objekt verwendeten Speicher freizugeben. Der `PGconn`-Zeiger darf nach dem Aufruf von `PQfinish` nicht mehr verwendet werden.

**`PQreset`**

Setzt den Kommunikationskanal zum Server zurück.

```c
void PQreset(PGconn *conn);
```

Diese Funktion schließt die Verbindung zum Server und versucht, mit denselben zuvor verwendeten Parametern eine neue Verbindung aufzubauen. Das kann für die Fehlerbehebung nützlich sein, wenn eine funktionierende Verbindung verloren geht.

**`PQresetStart`**, **`PQresetPoll`**

Setzen den Kommunikationskanal zum Server nichtblockierend zurück.

```c
int PQresetStart(PGconn *conn);

PostgresPollingStatusType PQresetPoll(PGconn *conn);
```

Diese Funktionen schließen die Verbindung zum Server und versuchen, mit denselben zuvor verwendeten Parametern eine neue Verbindung aufzubauen. Das kann für die Fehlerbehebung nützlich sein, wenn eine funktionierende Verbindung verloren geht. Sie unterscheiden sich von `PQreset` oben dadurch, dass sie nichtblockierend arbeiten. Diese Funktionen unterliegen denselben Einschränkungen wie `PQconnectStartParams`, `PQconnectStart` und `PQconnectPoll`.

Um ein Zurücksetzen der Verbindung einzuleiten, rufen Sie `PQresetStart` auf. Gibt die Funktion `0` zurück, ist das Zurücksetzen fehlgeschlagen. Gibt sie `1` zurück, pollen Sie das Zurücksetzen mit `PQresetPoll` auf genau dieselbe Weise, wie Sie die Verbindung mit `PQconnectPoll` erstellen würden.

**`PQpingParams`**

`PQpingParams` meldet den Status des Servers. Die Funktion akzeptiert Verbindungsparameter, die mit denen von `PQconnectdbParams` identisch sind, wie oben beschrieben. Es ist nicht erforderlich, korrekte Werte für Benutzername, Passwort oder Datenbankname anzugeben, um den Serverstatus zu erhalten; wenn jedoch falsche Werte angegeben werden, protokolliert der Server einen fehlgeschlagenen Verbindungsversuch.

```c
PGPing PQpingParams(const char * const *keywords,
                    const char * const *values,
                    int expand_dbname);
```

Die Funktion gibt einen der folgenden Werte zurück:

- `PQPING_OK`: Der Server läuft und scheint Verbindungen anzunehmen.
- `PQPING_REJECT`: Der Server läuft, befindet sich aber in einem Zustand, der Verbindungen nicht zulässt, etwa Start, Herunterfahren oder Crash-Recovery.
- `PQPING_NO_RESPONSE`: Der Server konnte nicht kontaktiert werden. Dies kann bedeuten, dass der Server nicht läuft, dass mit den angegebenen Verbindungsparametern etwas nicht stimmt, zum Beispiel eine falsche Portnummer, oder dass ein Problem mit der Netzwerkverbindung besteht, zum Beispiel eine Firewall, die die Verbindungsanforderung blockiert.
- `PQPING_NO_ATTEMPT`: Es wurde kein Versuch unternommen, den Server zu kontaktieren, weil die bereitgestellten Parameter offensichtlich falsch waren oder ein clientseitiges Problem vorlag, zum Beispiel Speichermangel.

**`PQping`**

`PQping` meldet den Status des Servers. Die Funktion akzeptiert Verbindungsparameter, die mit denen von `PQconnectdb` identisch sind, wie oben beschrieben. Es ist nicht erforderlich, korrekte Werte für Benutzername, Passwort oder Datenbankname anzugeben, um den Serverstatus zu erhalten; wenn jedoch falsche Werte angegeben werden, protokolliert der Server einen fehlgeschlagenen Verbindungsversuch.

```c
PGPing PQping(const char *conninfo);
```

Die Rückgabewerte sind dieselben wie bei `PQpingParams`.

**`PQsetSSLKeyPassHook_OpenSSL`**

`PQsetSSLKeyPassHook_OpenSSL` erlaubt einer Anwendung, die Standardbehandlung verschlüsselter Schlüsseldateien für Clientzertifikate durch `libpq` zu überschreiben, die sonst `sslpassword` oder interaktive Abfrage verwendet.

```c
void PQsetSSLKeyPassHook_OpenSSL(PQsslKeyPassHook_OpenSSL_type hook);
```

Die Anwendung übergibt einen Zeiger auf eine Callback-Funktion mit folgender Signatur:

```c
int callback_fn(char *buf, int size, PGconn *conn);
```

`libpq` ruft diese Funktion dann anstelle seines Standardhandlers `PQdefaultSSLKeyPassHook_OpenSSL` auf. Der Callback sollte das Passwort für den Schlüssel bestimmen und es in den Ergebnispuffer `buf` der Größe `size` kopieren. Die Zeichenkette in `buf` muss nullterminiert sein. Der Callback muss die Länge des in `buf` gespeicherten Passworts ohne Nullterminator zurückgeben. Bei einem Fehler sollte der Callback `buf[0] = '\0'` setzen und `0` zurückgeben. Ein Beispiel finden Sie bei `PQdefaultSSLKeyPassHook_OpenSSL` im Quellcode von `libpq`.

Wenn der Benutzer einen ausdrücklichen Schlüsselort angegeben hat, steht dessen Pfad in `conn->sslkey`, wenn der Callback aufgerufen wird. Dieser Wert ist leer, wenn der Standardschlüsselpfad verwendet wird. Bei Schlüsseln, die Engine-Spezifizierer sind, hängt es von den Engine-Implementierungen ab, ob sie den OpenSSL-Passwort-Callback verwenden oder eine eigene Behandlung definieren.

Der Anwendungs-Callback kann nicht behandelte Fälle an `PQdefaultSSLKeyPassHook_OpenSSL` delegieren, diesen zuerst aufrufen und bei Rückgabe von `0` etwas anderes versuchen oder ihn vollständig überschreiben.

Der Callback darf den normalen Kontrollfluss nicht mit Exceptions, `longjmp(...)` usw. verlassen. Er muss normal zurückkehren.

**`PQgetSSLKeyPassHook_OpenSSL`**

`PQgetSSLKeyPassHook_OpenSSL` gibt den aktuellen Passwort-Hook für Clientzertifikatschlüssel zurück, oder `NULL`, wenn keiner gesetzt wurde.

```c
PQsslKeyPassHook_OpenSSL_type PQgetSSLKeyPassHook_OpenSSL(void);
```

### 32.1.1. Verbindungsstrings

Mehrere `libpq`-Funktionen parsen eine vom Benutzer angegebene Zeichenkette, um Verbindungsparameter zu erhalten. Für solche Zeichenketten werden zwei Formate akzeptiert: einfache Schlüsselwort/Wert-Zeichenketten und URIs. URIs folgen im Allgemeinen [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986), außer dass Verbindungsstrings mit mehreren Hosts erlaubt sind, wie weiter unten beschrieben.

#### 32.1.1.1. Schlüsselwort/Wert-Verbindungsstrings

Im Schlüsselwort/Wert-Format hat jede Parametereinstellung die Form `keyword = value`, wobei Leerraum zwischen den Einstellungen steht. Leerzeichen um das Gleichheitszeichen einer Einstellung sind optional. Um einen leeren Wert oder einen Wert mit Leerzeichen zu schreiben, schließen Sie ihn in einfache Anführungszeichen ein, zum Beispiel `keyword = 'a value'`. Einfache Anführungszeichen und Backslashes innerhalb eines Werts müssen mit einem Backslash maskiert werden, also `\'` und `\\`.

Beispiel:

```text
host=localhost port=5432 dbname=mydb connect_timeout=10
```

Die erkannten Parameterschlüsselwörter sind in Abschnitt 32.1.2 aufgeführt.

#### 32.1.1.2. Verbindungs-URIs

Die allgemeine Form eines Verbindungs-URI ist:

```text
postgresql://[userspec@][hostspec][/dbname][?paramspec]
```

Dabei ist `userspec`:

```text
user[:password]
```

und `hostspec` ist:

```text
[host][:port][,...]
```

und `paramspec` ist:

```text
name=value[&...]
```

Der URI-Schema-Bezeichner kann entweder `postgresql://` oder `postgres://` sein. Alle verbleibenden URI-Teile sind optional. Die folgenden Beispiele zeigen gültige URI-Syntax:

```text
postgresql://
postgresql://localhost
postgresql://localhost:5433
postgresql://localhost/mydb
postgresql://user@localhost
postgresql://user:secret@localhost
postgresql://other@localhost/otherdb?connect_timeout=10&application_name=myapp
postgresql://host1:123,host2:456/somedb?target_session_attrs=any&application_name=myapp
```

Werte, die normalerweise im hierarchischen Teil des URI erscheinen würden, können alternativ als benannte Parameter angegeben werden. Zum Beispiel:

```text
postgresql:///mydb?host=localhost&port=5433
```

Alle benannten Parameter müssen den in Abschnitt 32.1.2 aufgeführten Schlüsselwörtern entsprechen, mit einer Ausnahme: Aus Gründen der Kompatibilität mit JDBC-Verbindungs-URIs werden Vorkommen von `ssl=true` in `sslmode=require` übersetzt.

Der Verbindungs-URI muss mit Prozentkodierung kodiert werden, wenn er in einem seiner Teile Symbole mit besonderer Bedeutung enthält. Siehe dazu [RFC 3986, Abschnitt 2.1](https://datatracker.ietf.org/doc/html/rfc3986#section-2.1). Hier ist ein Beispiel, in dem das Gleichheitszeichen (`=`) durch `%3D` und das Leerzeichen durch `%20` ersetzt wird:

```text
postgresql://user@localhost:5433/mydb?options=-c%20synchronous_commit%3Doff
```

Der Host-Teil kann entweder ein Hostname oder eine IP-Adresse sein. Um eine IPv6-Adresse anzugeben, schließen Sie sie in eckige Klammern ein:

```text
postgresql://[2001:db8::1234]/database
```

Der Host-Teil wird so interpretiert, wie es für den Parameter `host` beschrieben ist. Insbesondere wird eine Unix-Domain-Socket-Verbindung gewählt, wenn der Host-Teil entweder leer ist oder wie ein absoluter Pfadname aussieht; andernfalls wird eine TCP/IP-Verbindung eingeleitet. Beachten Sie jedoch, dass der Schrägstrich im hierarchischen Teil des URI ein reserviertes Zeichen ist. Um also ein nicht standardmäßiges Unix-Domain-Socket-Verzeichnis anzugeben, lassen Sie entweder den Host-Teil des URI weg und geben `host` als benannten Parameter an, oder kodieren Sie den Pfad im Host-Teil des URI mit Prozentkodierung:

```text
postgresql:///dbname?host=/var/lib/postgresql
postgresql://%2Fvar%2Flib%2Fpostgresql/dbname
```

Es ist möglich, in einem einzelnen URI mehrere Host-Komponenten anzugeben, jeweils mit optionaler Port-Komponente. Ein URI der Form

```text
postgresql://host1:port1,host2:port2,host3:port3/
```

ist äquivalent zu einem Verbindungsstring der Form

```text
host=host1,host2,host3 port=port1,port2,port3
```

Wie weiter unten beschrieben, wird jeder Host der Reihe nach versucht, bis erfolgreich eine Verbindung hergestellt wurde.

#### 32.1.1.3. Mehrere Hosts angeben

Es ist möglich, mehrere Hosts für den Verbindungsaufbau anzugeben, sodass sie in der angegebenen Reihenfolge versucht werden. Im Schlüsselwort/Wert-Format akzeptieren die Optionen `host`, `hostaddr` und `port` kommagetrennte Wertelisten. In jeder angegebenen Option muss dieselbe Anzahl von Elementen stehen, sodass zum Beispiel der erste Eintrag in `hostaddr` dem ersten Hostnamen entspricht, der zweite Eintrag in `hostaddr` dem zweiten Hostnamen usw. Als Ausnahme gilt: Wenn nur ein `port` angegeben ist, gilt er für alle Hosts.

Im Verbindungs-URI-Format können Sie mehrere durch Kommas getrennte `host:port`-Paare in der Host-Komponente des URI aufführen.

In beiden Formaten kann ein einzelner Hostname in mehrere Netzwerkadressen übersetzt werden. Ein häufiges Beispiel dafür ist ein Host, der sowohl eine IPv4- als auch eine IPv6-Adresse hat.

Wenn mehrere Hosts angegeben sind oder ein einzelner Hostname in mehrere Adressen übersetzt wird, werden alle Hosts und Adressen der Reihe nach versucht, bis einer erfolgreich ist. Wenn keiner der Hosts erreicht werden kann, schlägt die Verbindung fehl. Wenn eine Verbindung erfolgreich hergestellt wird, die Authentifizierung aber fehlschlägt, werden die verbleibenden Hosts in der Liste nicht versucht.

Wenn eine Passwortdatei verwendet wird, können Sie verschiedene Passwörter für verschiedene Hosts haben. Alle anderen Verbindungsoptionen sind für jeden Host in der Liste gleich; es ist nicht möglich, zum Beispiel verschiedene Benutzernamen für verschiedene Hosts anzugeben.

### 32.1.2. Parameterschlüsselwörter

Die derzeit erkannten Parameterschlüsselwörter sind:

**`host`**

Name des Hosts, zu dem eine Verbindung hergestellt werden soll. Wenn ein Hostname wie ein absoluter Pfadname aussieht, bezeichnet er Unix-Domain-Kommunikation statt TCP/IP-Kommunikation; der Wert ist dann der Name des Verzeichnisses, in dem die Socket-Datei gespeichert ist. Unter Unix beginnt ein absoluter Pfadname mit einem Schrägstrich. Unter Windows werden auch Pfade erkannt, die mit Laufwerksbuchstaben beginnen. Wenn der Hostname mit `@` beginnt, wird er als Unix-Domain-Socket im abstrakten Namensraum verstanden, was derzeit unter Linux und Windows unterstützt wird. Wenn `host` nicht angegeben oder leer ist, verbindet sich `libpq` standardmäßig mit einem Unix-Domain-Socket in `/tmp` oder mit dem Socket-Verzeichnis, das beim Bauen von PostgreSQL angegeben wurde. Unter Windows ist der Standard eine Verbindung zu `localhost`.

Eine kommagetrennte Liste von Hostnamen wird ebenfalls akzeptiert. In diesem Fall wird jeder Hostname in der Liste der Reihe nach versucht; ein leerer Eintrag in der Liste wählt das oben erklärte Standardverhalten. Einzelheiten finden Sie in Abschnitt 32.1.1.3.

**`hostaddr`**

Numerische IP-Adresse des Hosts, zu dem eine Verbindung hergestellt werden soll. Diese sollte im Standardformat für IPv4-Adressen angegeben werden, zum Beispiel `172.28.40.9`. Wenn Ihr Rechner IPv6 unterstützt, können Sie auch solche Adressen verwenden. TCP/IP-Kommunikation wird immer verwendet, wenn für diesen Parameter eine nichtleere Zeichenkette angegeben ist. Wenn dieser Parameter nicht angegeben ist, wird der Wert von `host` aufgelöst, um die entsprechende IP-Adresse zu finden; wenn `host` bereits eine IP-Adresse angibt, wird dieser Wert direkt verwendet.

Die Verwendung von `hostaddr` erlaubt es der Anwendung, eine Hostnamenauflösung zu vermeiden, was in Anwendungen mit Zeitvorgaben wichtig sein kann. Für die Authentifizierungsmethoden GSSAPI oder SSPI sowie für die SSL-Zertifikatsprüfung mit `verify-full` ist jedoch ein Hostname erforderlich. Es gelten die folgenden Regeln:

- Wenn `host` ohne `hostaddr` angegeben ist, findet eine Hostnamenauflösung statt. Bei Verwendung von `PQconnectPoll` findet die Auflösung statt, wenn `PQconnectPoll` diesen Hostnamen erstmals betrachtet, und sie kann dazu führen, dass `PQconnectPoll` für beträchtliche Zeit blockiert.
- Wenn `hostaddr` ohne `host` angegeben ist, gibt der Wert von `hostaddr` die Netzwerkadresse des Servers an. Der Verbindungsversuch schlägt fehl, wenn die Authentifizierungsmethode einen Hostnamen benötigt.
- Wenn sowohl `host` als auch `hostaddr` angegeben sind, gibt der Wert von `hostaddr` die Netzwerkadresse des Servers an. Der Wert von `host` wird ignoriert, sofern die Authentifizierungsmethode ihn nicht benötigt; in diesem Fall wird er als Hostname verwendet.

Beachten Sie, dass die Authentifizierung wahrscheinlich fehlschlägt, wenn `host` nicht der Name des Servers unter der Netzwerkadresse `hostaddr` ist. Außerdem wird `host` zur Identifikation der Verbindung in einer Passwortdatei verwendet, wenn sowohl `host` als auch `hostaddr` angegeben sind (siehe [Abschnitt 32.16](32_libpq_C_Bibliothek.md#3216-die-passwortdatei)).

Eine kommagetrennte Liste von `hostaddr`-Werten wird ebenfalls akzeptiert. In diesem Fall wird jeder Host in der Liste der Reihe nach versucht. Ein leerer Eintrag in der Liste bewirkt, dass der entsprechende Hostname verwendet wird, oder der Standardhostname, wenn auch dieser leer ist. Einzelheiten finden Sie in Abschnitt 32.1.1.3.

Ohne Hostnamen und ohne Hostadresse verbindet sich `libpq` über einen lokalen Unix-Domain-Socket; unter Windows versucht es, sich mit `localhost` zu verbinden.

**`port`**

Portnummer, zu der auf dem Serverhost verbunden werden soll, oder Dateinamenerweiterung der Socket-Datei für Unix-Domain-Verbindungen. Wenn in den Parametern `host` oder `hostaddr` mehrere Hosts angegeben wurden, kann dieser Parameter eine kommagetrennte Liste von Ports derselben Länge wie die Hostliste angeben, oder eine einzelne Portnummer, die für alle Hosts verwendet wird. Eine leere Zeichenkette oder ein leerer Eintrag in einer kommagetrennten Liste gibt die Standardportnummer an, die beim Bauen von PostgreSQL festgelegt wurde.

**`dbname`**

Der Datenbankname. Standardmäßig entspricht er dem Benutzernamen. In bestimmten Kontexten wird der Wert auf erweiterte Formate geprüft; Einzelheiten dazu finden Sie in Abschnitt 32.1.1.

**`user`**

PostgreSQL-Benutzername, unter dem verbunden werden soll. Standardmäßig entspricht er dem Betriebssystemnamen des Benutzers, der die Anwendung ausführt.

**`password`**

Passwort, das verwendet wird, wenn der Server Passwortauthentifizierung verlangt.

**`passfile`**

Gibt den Namen der Datei an, in der Passwörter gespeichert werden (siehe [Abschnitt 32.16](32_libpq_C_Bibliothek.md#3216-die-passwortdatei)). Standard ist `~/.pgpass` beziehungsweise `%APPDATA%\postgresql\pgpass.conf` unter Microsoft Windows. Es wird kein Fehler gemeldet, wenn diese Datei nicht existiert.

**`require_auth`**

Gibt die Authentifizierungsmethode an, die der Client vom Server verlangt. Wenn der Server nicht die erforderliche Methode zur Authentifizierung des Clients verwendet oder wenn der Authentifizierungs-Handshake vom Server nicht vollständig abgeschlossen wird, schlägt die Verbindung fehl. Es kann auch eine kommagetrennte Liste von Methoden angegeben werden; von diesen muss der Server genau eine verwenden, damit die Verbindung erfolgreich ist. Standardmäßig wird jede Authentifizierungsmethode akzeptiert, und der Server darf die Authentifizierung vollständig überspringen.

Methoden können durch Hinzufügen eines Präfixes `!` negiert werden. In diesem Fall darf der Server die aufgeführte Methode nicht versuchen; jede andere Methode wird akzeptiert, und der Server darf den Client auch gar nicht authentifizieren. Wenn eine kommagetrennte Liste angegeben ist, darf der Server keine der aufgeführten negierten Methoden versuchen. Negierte und nicht negierte Formen dürfen nicht in derselben Einstellung kombiniert werden.

Als letzter Sonderfall verlangt die Methode `none`, dass der Server keine Authentifizierungs-Challenge verwendet. Sie kann ebenfalls negiert werden, um irgendeine Form von Authentifizierung zu verlangen.

Die folgenden Methoden können angegeben werden:

- `password`: Der Server muss Klartext-Passwortauthentifizierung anfordern.
- `md5`: Der Server muss MD5-gehashte Passwortauthentifizierung anfordern.
- `gss`: Der Server muss entweder einen Kerberos-Handshake über GSSAPI anfordern oder einen GSS-verschlüsselten Kanal aufbauen; siehe auch `gssencmode`.
- `sspi`: Der Server muss Windows-SSPI-Authentifizierung anfordern.
- `scram-sha-256`: Der Server muss einen SCRAM-SHA-256-Authentifizierungsaustausch mit dem Client erfolgreich abschließen.
- `oauth`: Der Server muss vom Client ein OAuth-Bearer-Token anfordern.
- `none`: Der Server darf den Client nicht zu einem Authentifizierungsaustausch auffordern. Dies verbietet weder Clientzertifikat-Authentifizierung über TLS noch GSS-Authentifizierung über deren verschlüsselten Transport.

> **Warnung**
>
> Die Unterstützung für MD5-verschlüsselte Passwörter ist veraltet und wird in einer zukünftigen PostgreSQL-Version entfernt. Details zur Migration auf einen anderen Passworttyp finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

**`channel_binding`**

Diese Option steuert die Verwendung von Channel Binding durch den Client. Die Einstellung `require` bedeutet, dass die Verbindung Channel Binding verwenden muss; `prefer` bedeutet, dass der Client Channel Binding wählt, wenn es verfügbar ist; `disable` verhindert die Verwendung von Channel Binding. Der Standard ist `prefer`, wenn PostgreSQL mit SSL-Unterstützung kompiliert wurde; andernfalls ist der Standard `disable`.

Channel Binding ist eine Methode, mit der sich der Server gegenüber dem Client authentifiziert. Sie wird nur über SSL-Verbindungen zu PostgreSQL-11-oder-neueren Servern unterstützt, die die SCRAM-Authentifizierungsmethode verwenden.

**`connect_timeout`**

Maximale Wartezeit beim Verbindungsaufbau in Sekunden, geschrieben als Dezimalzahl, zum Beispiel `10`. Null, ein negativer Wert oder keine Angabe bedeutet unbegrenztes Warten. Dieses Timeout gilt separat für jeden Hostnamen oder jede IP-Adresse. Wenn Sie zum Beispiel zwei Hosts angeben und `connect_timeout` `5` ist, läuft jeder Host nach 5 Sekunden ohne erfolgreiche Verbindung in ein Timeout; die gesamte Wartezeit auf eine Verbindung kann also bis zu 10 Sekunden betragen.

**`client_encoding`**

Setzt den Konfigurationsparameter `client_encoding` für diese Verbindung. Zusätzlich zu den Werten, die von der entsprechenden Serveroption akzeptiert werden, können Sie `auto` verwenden, um das richtige Encoding aus der aktuellen Locale des Clients zu bestimmen, unter Unix-Systemen aus der Umgebungsvariablen `LC_CTYPE`.

**`options`**

Gibt Befehlszeilenoptionen an, die beim Verbindungsstart an den Server gesendet werden. Wenn Sie diesen Wert zum Beispiel auf `-c geqo=off` oder `--geqo=off` setzen, wird der Sitzungswert des Parameters `geqo` auf `off` gesetzt. Leerzeichen innerhalb dieser Zeichenkette gelten als Trennzeichen zwischen Befehlszeilenargumenten, sofern sie nicht mit einem Backslash (`\`) maskiert sind; schreiben Sie `\\`, um einen literalen Backslash darzustellen. Eine ausführliche Diskussion der verfügbaren Optionen finden Sie in [Kapitel 19](19_Serverkonfiguration.md).

**`application_name`**

Gibt einen Wert für den Konfigurationsparameter `application_name` an.

**`fallback_application_name`**

Gibt einen Fallback-Wert für den Konfigurationsparameter `application_name` an. Dieser Wert wird verwendet, wenn kein Wert für `application_name` über einen Verbindungsparameter oder die Umgebungsvariable `PGAPPNAME` angegeben wurde. Die Angabe eines Fallback-Namens ist in generischen Hilfsprogrammen nützlich, die einen Standard-Anwendungsnamen setzen, aber dem Benutzer erlauben möchten, ihn zu überschreiben.

**`keepalives`**

Steuert, ob clientseitige TCP-Keepalives verwendet werden. Der Standardwert ist `1`, also ein; Sie können ihn auf `0` setzen, also aus, wenn Keepalives nicht gewünscht sind. Dieser Parameter wird für Verbindungen über einen Unix-Domain-Socket ignoriert.

**`keepalives_idle`**

Steuert die Anzahl von Sekunden ohne Aktivität, nach denen TCP eine Keepalive-Nachricht an den Server senden soll. Ein Wert von null verwendet den Systemstandard. Dieser Parameter wird bei Verbindungen über einen Unix-Domain-Socket ignoriert oder wenn Keepalives deaktiviert sind. Er wird nur auf Systemen unterstützt, auf denen `TCP_KEEPIDLE` oder eine äquivalente Socket-Option verfügbar ist, sowie unter Windows; auf anderen Systemen hat er keine Wirkung.

**`keepalives_interval`**

Steuert die Anzahl von Sekunden, nach denen eine vom Server nicht bestätigte TCP-Keepalive-Nachricht erneut übertragen werden soll. Ein Wert von null verwendet den Systemstandard. Dieser Parameter wird bei Verbindungen über einen Unix-Domain-Socket ignoriert oder wenn Keepalives deaktiviert sind. Er wird nur auf Systemen unterstützt, auf denen `TCP_KEEPINTVL` oder eine äquivalente Socket-Option verfügbar ist, sowie unter Windows; auf anderen Systemen hat er keine Wirkung.

**`keepalives_count`**

Steuert die Anzahl von TCP-Keepalives, die verloren gehen dürfen, bevor die Clientverbindung zum Server als tot betrachtet wird. Ein Wert von null verwendet den Systemstandard. Dieser Parameter wird bei Verbindungen über einen Unix-Domain-Socket ignoriert oder wenn Keepalives deaktiviert sind. Er wird nur auf Systemen unterstützt, auf denen `TCP_KEEPCNT` oder eine äquivalente Socket-Option verfügbar ist; auf anderen Systemen hat er keine Wirkung.

**`tcp_user_timeout`**

Steuert die Anzahl von Millisekunden, die übertragene Daten unbestätigt bleiben dürfen, bevor eine Verbindung zwangsweise geschlossen wird. Ein Wert von null verwendet den Systemstandard. Dieser Parameter wird bei Verbindungen über einen Unix-Domain-Socket ignoriert. Er wird nur auf Systemen unterstützt, auf denen `TCP_USER_TIMEOUT` verfügbar ist; auf anderen Systemen hat er keine Wirkung.

**`replication`**

Diese Option legt fest, ob die Verbindung statt des normalen Protokolls das Replikationsprotokoll verwenden soll. PostgreSQL-Replikationsverbindungen sowie Werkzeuge wie `pg_basebackup` verwenden dies intern, es kann aber auch von Drittanwendungen verwendet werden. Eine Beschreibung des Replikationsprotokolls finden Sie in [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll).

Die folgenden Werte werden unterstützt; Groß-/Kleinschreibung spielt keine Rolle:

- `true`, `on`, `yes`, `1`: Die Verbindung geht in den physischen Replikationsmodus.
- `database`: Die Verbindung geht in den logischen Replikationsmodus und verbindet sich mit der im Parameter `dbname` angegebenen Datenbank.
- `false`, `off`, `no`, `0`: Die Verbindung ist eine reguläre Verbindung; dies ist das Standardverhalten.

Im physischen oder logischen Replikationsmodus kann nur das einfache Abfrageprotokoll verwendet werden.

**`gssencmode`**

Diese Option legt fest, ob und mit welcher Priorität eine sichere GSS-TCP/IP-Verbindung mit dem Server ausgehandelt wird. Es gibt drei Modi:

- `disable`: Nur eine nicht GSSAPI-verschlüsselte Verbindung versuchen.
- `prefer` (Standard): Wenn GSSAPI-Anmeldedaten vorhanden sind, also in einem Credentials-Cache, zuerst eine GSSAPI-verschlüsselte Verbindung versuchen; wenn dies fehlschlägt oder keine Anmeldedaten vorhanden sind, eine nicht GSSAPI-verschlüsselte Verbindung versuchen. Dies ist der Standard, wenn PostgreSQL mit GSSAPI-Unterstützung kompiliert wurde.
- `require`: Nur eine GSSAPI-verschlüsselte Verbindung versuchen.

`gssencmode` wird für Unix-Domain-Socket-Kommunikation ignoriert. Wenn PostgreSQL ohne GSSAPI-Unterstützung kompiliert wurde, verursacht die Option `require` einen Fehler, während `prefer` akzeptiert wird, `libpq` aber tatsächlich keine GSSAPI-verschlüsselte Verbindung versucht.

**`sslmode`**

Diese Option legt fest, ob und mit welcher Priorität eine sichere SSL-TCP/IP-Verbindung mit dem Server ausgehandelt wird. Es gibt sechs Modi:

- `disable`: Nur eine Nicht-SSL-Verbindung versuchen.
- `allow`: Zuerst eine Nicht-SSL-Verbindung versuchen; wenn diese fehlschlägt, eine SSL-Verbindung versuchen.
- `prefer` (Standard): Zuerst eine SSL-Verbindung versuchen; wenn diese fehlschlägt, eine Nicht-SSL-Verbindung versuchen.
- `require`: Nur eine SSL-Verbindung versuchen. Wenn eine Root-CA-Datei vorhanden ist, das Zertifikat so prüfen, als wäre `verify-ca` angegeben.
- `verify-ca`: Nur eine SSL-Verbindung versuchen und prüfen, dass das Serverzertifikat von einer vertrauenswürdigen Zertifizierungsstelle (CA) ausgestellt wurde.
- `verify-full`: Nur eine SSL-Verbindung versuchen, prüfen, dass das Serverzertifikat von einer vertrauenswürdigen CA ausgestellt wurde, und prüfen, dass der angeforderte Serverhostname mit dem Namen im Zertifikat übereinstimmt.

Eine ausführliche Beschreibung der Funktionsweise dieser Optionen finden Sie in [Abschnitt 32.19](32_libpq_C_Bibliothek.md#3219-sslunterstützung).

`sslmode` wird für Unix-Domain-Socket-Kommunikation ignoriert. Wenn PostgreSQL ohne SSL-Unterstützung kompiliert wurde, verursachen die Optionen `require`, `verify-ca` oder `verify-full` einen Fehler, während `allow` und `prefer` akzeptiert werden, `libpq` aber tatsächlich keine SSL-Verbindung versucht.

Beachten Sie: Wenn GSSAPI-Verschlüsselung möglich ist, wird sie unabhängig vom Wert von `sslmode` bevorzugt gegenüber SSL-Verschlüsselung verwendet. Um in einer Umgebung mit funktionierender GSSAPI-Infrastruktur, etwa einem Kerberos-Server, die Verwendung von SSL-Verschlüsselung zu erzwingen, setzen Sie zusätzlich `gssencmode` auf `disable`.

**`requiressl`**

Diese Option ist zugunsten der Einstellung `sslmode` veraltet.

Wenn sie auf `1` gesetzt ist, ist eine SSL-Verbindung zum Server erforderlich; dies entspricht `sslmode=require`. `libpq` verweigert dann die Verbindung, wenn der Server keine SSL-Verbindung akzeptiert. Wenn sie auf `0` gesetzt ist, dem Standard, handelt `libpq` den Verbindungstyp mit dem Server aus; dies entspricht `sslmode=prefer`. Diese Option ist nur verfügbar, wenn PostgreSQL mit SSL-Unterstützung kompiliert wurde.

**`sslnegotiation`**

Diese Option steuert, wie SSL-Verschlüsselung mit dem Server ausgehandelt wird, sofern SSL verwendet wird. Im Standardmodus `postgres` fragt der Client den Server zuerst, ob SSL unterstützt wird. Im Modus `direct` startet der Client den standardmäßigen SSL-Handshake direkt nach dem Aufbau der TCP/IP-Verbindung. Die traditionelle PostgreSQL-Protokollaushandlung ist bei unterschiedlichen Serverkonfigurationen am flexibelsten. Wenn bekannt ist, dass der Server direkte SSL-Verbindungen unterstützt, benötigt die letztere Variante einen Roundtrip weniger, reduziert damit die Verbindungslatenz und erlaubt außerdem die Verwendung protokollagnostischer SSL-Netzwerkwerkzeuge. Die direkte SSL-Option wurde in PostgreSQL Version 17 eingeführt.

- `postgres`: PostgreSQL-Protokollaushandlung durchführen. Dies ist der Standard, wenn die Option nicht angegeben wird.
- `direct`: Den SSL-Handshake direkt nach dem Aufbau der TCP/IP-Verbindung starten. Dies ist nur mit `sslmode=require` oder höher erlaubt, weil schwächere Einstellungen zu einem unbeabsichtigten Fallback auf Klartextauthentifizierung führen könnten, wenn der Server den direkten SSL-Handshake nicht unterstützt.

**`sslcompression`**

Wenn auf `1` gesetzt, werden über SSL-Verbindungen gesendete Daten komprimiert. Wenn auf `0` gesetzt, wird Kompression deaktiviert. Der Standard ist `0`. Dieser Parameter wird ignoriert, wenn keine SSL-Verbindung hergestellt wird.

SSL-Kompression gilt heute als unsicher, und ihre Verwendung wird nicht mehr empfohlen. OpenSSL 1.1.0 hat Kompression standardmäßig deaktiviert, und viele Betriebssystemdistributionen haben sie auch in früheren Versionen deaktiviert. Das Setzen dieses Parameters auf `on` hat daher keine Wirkung, wenn der Server Kompression nicht akzeptiert. PostgreSQL 14 hat Kompression im Backend vollständig deaktiviert.

Wenn Sicherheit nicht im Vordergrund steht, kann Kompression den Durchsatz verbessern, wenn das Netzwerk der Engpass ist. Das Deaktivieren der Kompression kann Antwortzeit und Durchsatz verbessern, wenn die CPU-Leistung der begrenzende Faktor ist.

**`sslcert`**

Dieser Parameter gibt den Dateinamen des SSL-Clientzertifikats an und ersetzt den Standard `~/.postgresql/postgresql.crt`. Der Parameter wird ignoriert, wenn keine SSL-Verbindung hergestellt wird.

**`sslkey`**

Dieser Parameter gibt den Ort des geheimen Schlüssels an, der für das Clientzertifikat verwendet wird. Er kann entweder einen Dateinamen angeben, der statt des Standards `~/.postgresql/postgresql.key` verwendet wird, oder einen von einer externen „Engine“ erhaltenen Schlüssel; Engines sind ladbare OpenSSL-Module. Eine externe Engine-Spezifikation sollte aus einem durch Doppelpunkt getrennten Engine-Namen und einem enginespezifischen Schlüsselbezeichner bestehen. Der Parameter wird ignoriert, wenn keine SSL-Verbindung hergestellt wird.

**`sslkeylogfile`**

Dieser Parameter gibt den Ort an, an dem `libpq` die in diesem SSL-Kontext verwendeten Schlüssel protokolliert. Dies ist nützlich zum Debuggen von PostgreSQL-Protokollinteraktionen oder Clientverbindungen mit Netzwerk-Inspektionswerkzeugen wie Wireshark. Der Parameter wird ignoriert, wenn keine SSL-Verbindung hergestellt wird oder wenn LibreSSL verwendet wird; LibreSSL unterstützt kein Schlüssel-Logging. Schlüssel werden im NSS-Format protokolliert.

> **Warnung**
>
> Schlüssel-Logging legt potenziell vertrauliche Informationen in der Keylog-Datei offen. Keylog-Dateien sollten mit derselben Sorgfalt behandelt werden wie `sslkey`-Dateien.

**`sslpassword`**

Dieser Parameter gibt das Passwort für den in `sslkey` angegebenen geheimen Schlüssel an. Damit können private Schlüssel für Clientzertifikate verschlüsselt auf der Platte gespeichert werden, auch wenn interaktive Passphrase-Eingabe nicht praktikabel ist.

Die Angabe dieses Parameters mit einem nichtleeren Wert unterdrückt die Eingabeaufforderung `Enter PEM pass phrase:`, die OpenSSL standardmäßig ausgibt, wenn `libpq` ein verschlüsselter Clientzertifikatsschlüssel bereitgestellt wird.

Wenn der Schlüssel nicht verschlüsselt ist, wird dieser Parameter ignoriert. Der Parameter hat keine Wirkung auf Schlüssel, die durch OpenSSL-Engines angegeben werden, sofern die Engine nicht den OpenSSL-Passwort-Callback-Mechanismus für Abfragen verwendet.

Es gibt keine entsprechende Umgebungsvariable für diese Option und keine Möglichkeit, sie in `.pgpass` nachzuschlagen. Sie kann in einer Verbindungsdefinition in einer Service-Datei verwendet werden. Benutzer mit anspruchsvolleren Anforderungen sollten OpenSSL-Engines und Werkzeuge wie PKCS#11 oder USB-Krypto-Offload-Geräte in Betracht ziehen.

**`sslcertmode`**

Diese Option legt fest, ob ein Clientzertifikat an den Server gesendet werden darf und ob der Server eines anfordern muss. Es gibt drei Modi:

- `disable`: Ein Clientzertifikat wird nie gesendet, auch wenn eines verfügbar ist, entweder am Standardort oder über `sslcert` bereitgestellt.
- `allow` (Standard): Ein Zertifikat darf gesendet werden, wenn der Server eines anfordert und der Client eines senden kann.
- `require`: Der Server muss ein Zertifikat anfordern. Die Verbindung schlägt fehl, wenn der Client kein Zertifikat sendet und der Server den Client dennoch erfolgreich authentifiziert.

> **Hinweis**
>
> `sslcertmode=require` fügt keine zusätzliche Sicherheit hinzu, da nicht garantiert ist, dass der Server das Zertifikat korrekt validiert; PostgreSQL-Server fordern TLS-Zertifikate von Clients im Allgemeinen unabhängig davon an, ob sie diese validieren oder nicht. Die Option kann bei der Fehlersuche in komplexeren TLS-Konfigurationen nützlich sein.

**`sslrootcert`**

Dieser Parameter gibt den Namen einer Datei an, die SSL-Zertifikate von Zertifizierungsstellen (CAs) enthält. Wenn die Datei existiert, wird geprüft, ob das Zertifikat des Servers von einer dieser Stellen signiert wurde. Der Standard ist `~/.postgresql/root.crt`.

Stattdessen kann der besondere Wert `system` angegeben werden. In diesem Fall werden die vertrauenswürdigen CA-Roots aus der SSL-Implementierung geladen. Die genauen Orte dieser Root-Zertifikate unterscheiden sich je nach SSL-Implementierung und Plattform. Insbesondere bei OpenSSL können die Orte zusätzlich durch die Umgebungsvariablen `SSL_CERT_DIR` und `SSL_CERT_FILE` verändert werden.

> **Hinweis**
>
> Bei Verwendung von `sslrootcert=system` wird der Standardwert von `sslmode` auf `verify-full` geändert, und jede schwächere Einstellung führt zu einem Fehler. In den meisten Fällen ist es trivial, für einen kontrollierten Hostnamen ein vom System vertrautes Zertifikat zu erhalten, wodurch `verify-ca` und alle schwächeren Modi nutzlos werden.
>
> Der magische Wert `system` hat Vorrang vor einer lokalen Zertifikatsdatei mit demselben Namen. Wenn Sie sich aus irgendeinem Grund in dieser Situation befinden, verwenden Sie einen alternativen Pfad wie `sslrootcert=./system`.

**`sslcrl`**

Dieser Parameter gibt den Dateinamen der Zertifikatssperrliste (CRL) für SSL-Serverzertifikate an. In dieser Datei aufgeführte Zertifikate werden, falls die Datei existiert, beim Versuch, das Serverzertifikat zu authentifizieren, abgelehnt. Wenn weder `sslcrl` noch `sslcrldir` gesetzt ist, wird diese Einstellung als `~/.postgresql/root.crl` verstanden.

**`sslcrldir`**

Dieser Parameter gibt den Verzeichnisnamen der Zertifikatssperrliste (CRL) für SSL-Serverzertifikate an. Zertifikate, die in den Dateien dieses Verzeichnisses aufgeführt sind, werden, falls das Verzeichnis existiert, beim Versuch, das Serverzertifikat zu authentifizieren, abgelehnt.

Das Verzeichnis muss mit dem OpenSSL-Befehl `openssl rehash` oder `c_rehash` vorbereitet werden. Einzelheiten finden Sie in dessen Dokumentation.

`sslcrl` und `sslcrldir` können zusammen angegeben werden.

**`sslsni`**

Wenn auf `1` gesetzt, dem Standard, setzt `libpq` bei SSL-fähigen Verbindungen die TLS-Erweiterung „Server Name Indication“ (SNI). Durch Setzen dieses Parameters auf `0` wird dies ausgeschaltet.

SNI kann von SSL-bewussten Proxys verwendet werden, um Verbindungen zu routen, ohne den SSL-Datenstrom entschlüsseln zu müssen. Beachten Sie, dass dies, sofern der Proxy den PostgreSQL-Protokoll-Handshake nicht kennt, das Setzen von `sslnegotiation` auf `direct` erfordern würde. SNI lässt den Zielhostnamen jedoch im Netzwerkverkehr im Klartext erscheinen und kann daher in manchen Fällen unerwünscht sein.

**`requirepeer`**

Dieser Parameter gibt den Betriebssystem-Benutzernamen des Servers an, zum Beispiel `requirepeer=postgres`. Beim Aufbau einer Unix-Domain-Socket-Verbindung prüft der Client am Anfang der Verbindung, ob der Serverprozess unter dem angegebenen Benutzernamen läuft, falls dieser Parameter gesetzt ist; andernfalls wird die Verbindung mit einem Fehler abgebrochen. Dieser Parameter kann verwendet werden, um Serverauthentifizierung bereitzustellen, ähnlich wie sie bei TCP/IP-Verbindungen mit SSL-Zertifikaten verfügbar ist. Beachten Sie: Wenn sich der Unix-Domain-Socket in `/tmp` oder an einem anderen öffentlich beschreibbaren Ort befindet, könnte jeder Benutzer dort einen lauschenden Server starten. Verwenden Sie diesen Parameter, um sicherzustellen, dass Sie mit einem Server verbunden sind, der von einem vertrauenswürdigen Benutzer ausgeführt wird. Diese Option wird nur auf Plattformen unterstützt, für die die Peer-Authentifizierungsmethode implementiert ist; siehe [Abschnitt 20.9](20_Client_Authentifizierung.md#209-peerauthentifizierung).

**`ssl_min_protocol_version`**

Dieser Parameter gibt die minimale SSL/TLS-Protokollversion an, die für die Verbindung erlaubt ist. Gültige Werte sind `TLSv1`, `TLSv1.1`, `TLSv1.2` und `TLSv1.3`. Die unterstützten Protokolle hängen von der verwendeten OpenSSL-Version ab; ältere Versionen unterstützen die modernsten Protokollversionen nicht. Wenn nichts angegeben ist, ist der Standard `TLSv1.2`, was zum Zeitpunkt dieser Dokumentation branchenüblichen Best Practices entspricht.

**`ssl_max_protocol_version`**

Dieser Parameter gibt die maximale SSL/TLS-Protokollversion an, die für die Verbindung erlaubt ist. Gültige Werte sind `TLSv1`, `TLSv1.1`, `TLSv1.2` und `TLSv1.3`. Die unterstützten Protokolle hängen von der verwendeten OpenSSL-Version ab; ältere Versionen unterstützen die modernsten Protokollversionen nicht. Wenn dieser Parameter nicht gesetzt ist, wird er ignoriert, und die Verbindung verwendet die vom Backend definierte Obergrenze, sofern eine gesetzt ist. Das Setzen der maximalen Protokollversion ist hauptsächlich für Tests nützlich oder wenn eine Komponente Probleme mit einem neueren Protokoll hat.

**`min_protocol_version`**

Gibt die minimale Protokollversion an, die für die Verbindung erlaubt ist. Standardmäßig ist jede Version des PostgreSQL-Protokolls erlaubt, die von `libpq` unterstützt wird; derzeit bedeutet das `3.0`. Wenn der Server mindestens diese Protokollversion nicht unterstützt, wird die Verbindung geschlossen.

Die derzeit unterstützten Werte sind `3.0`, `3.2` und `latest`. Der Wert `latest` entspricht der neuesten Protokollversion, die von der verwendeten `libpq`-Version unterstützt wird; derzeit ist das `3.2`.

**`max_protocol_version`**

Gibt die Protokollversion an, die vom Server angefordert werden soll. Standardmäßig wird Version `3.0` des PostgreSQL-Protokolls verwendet, sofern der Verbindungsstring keine Funktion angibt, die auf einer höheren Protokollversion beruht; in diesem Fall wird die neueste von `libpq` unterstützte Version verwendet. Wenn der Server die vom Client angeforderte Protokollversion nicht unterstützt, wird die Verbindung automatisch auf eine niedrigere Nebenprotokollversion herabgestuft, die der Server unterstützt. Nach Abschluss des Verbindungsversuchs können Sie mit `PQfullProtocolVersion` herausfinden, welche genaue Protokollversion ausgehandelt wurde.

Die derzeit unterstützten Werte sind `3.0`, `3.2` und `latest`. Der Wert `latest` entspricht der neuesten Protokollversion, die von der verwendeten `libpq`-Version unterstützt wird; derzeit ist das `3.2`.

**`krbsrvname`**

Kerberos-Servicename, der bei der Authentifizierung mit GSSAPI verwendet wird. Er muss mit dem in der Serverkonfiguration angegebenen Servicenamen übereinstimmen, damit Kerberos-Authentifizierung erfolgreich ist; siehe auch Abschnitt 20.6. Der Standardwert ist normalerweise `postgres`, kann aber beim Bauen von PostgreSQL über die Option `--with-krb-srvnam` von `configure` geändert werden. In den meisten Umgebungen muss dieser Parameter nie geändert werden. Einige Kerberos-Implementierungen können einen anderen Servicenamen erfordern, etwa Microsoft Active Directory, das den Servicenamen in Großbuchstaben verlangt (`POSTGRES`).

**`gsslib`**

GSS-Bibliothek, die für GSSAPI-Authentifizierung verwendet werden soll. Derzeit wird dies außer bei Windows-Builds ignoriert, die sowohl GSSAPI- als auch SSPI-Unterstützung enthalten. In diesem Fall setzen Sie den Wert auf `gssapi`, damit `libpq` für die Authentifizierung die GSSAPI-Bibliothek statt der standardmäßigen SSPI verwendet.

**`gssdelegation`**

Leitet GSS-Anmeldedaten an den Server weiter, also delegiert sie. Der Standard ist `0`, was bedeutet, dass Anmeldedaten nicht an den Server weitergeleitet werden. Setzen Sie den Wert auf `1`, damit Anmeldedaten weitergeleitet werden, wenn dies möglich ist.

**`scram_client_key`**

Der Base64-kodierte SCRAM-Clientschlüssel. Er kann von Foreign-Data-Wrappern oder ähnlicher Middleware verwendet werden, um durchgereichte SCRAM-Authentifizierung zu ermöglichen. Eine solche Implementierung finden Sie in Abschnitt F.38.1.10. Er ist nicht dafür gedacht, direkt von Benutzern oder Clientanwendungen angegeben zu werden.

**`scram_server_key`**

Der Base64-kodierte SCRAM-Serverschlüssel. Er kann von Foreign-Data-Wrappern oder ähnlicher Middleware verwendet werden, um durchgereichte SCRAM-Authentifizierung zu ermöglichen. Eine solche Implementierung finden Sie in Abschnitt F.38.1.10. Er ist nicht dafür gedacht, direkt von Benutzern oder Clientanwendungen angegeben zu werden.

**`service`**

Servicename, der für zusätzliche Parameter verwendet wird. Er gibt einen Servicenamen in `pg_service.conf` an, der zusätzliche Verbindungsparameter enthält. Dadurch können Anwendungen nur einen Servicenamen angeben, sodass Verbindungsparameter zentral verwaltet werden können. Siehe [Abschnitt 32.17](32_libpq_C_Bibliothek.md#3217-die-verbindungsdienstdatei).

**`target_session_attrs`**

Diese Option legt fest, ob die Sitzung bestimmte Eigenschaften haben muss, um akzeptabel zu sein. Sie wird typischerweise zusammen mit mehreren Hostnamen verwendet, um aus mehreren Hosts die erste akzeptable Alternative auszuwählen. Es gibt sechs Modi:

- `any` (Standard): Jede erfolgreiche Verbindung ist akzeptabel.
- `read-write`: Die Sitzung muss standardmäßig Lese-/Schreibtransaktionen akzeptieren, das heißt, der Server darf nicht im Hot-Standby-Modus sein, und der Parameter `default_transaction_read_only` muss `off` sein.
- `read-only`: Die Sitzung darf standardmäßig keine Lese-/Schreibtransaktionen akzeptieren, also das Gegenteil.
- `primary`: Der Server darf nicht im Hot-Standby-Modus sein.
- `standby`: Der Server muss im Hot-Standby-Modus sein.
- `prefer-standby`: Zuerst versuchen, einen Standby-Server zu finden; wenn keiner der aufgeführten Hosts ein Standby-Server ist, in beliebigem Modus erneut versuchen.

**`load_balance_hosts`**

Steuert die Reihenfolge, in der der Client versucht, sich mit den verfügbaren Hosts und Adressen zu verbinden. Sobald ein Verbindungsversuch erfolgreich ist, werden keine weiteren Hosts und Adressen versucht. Dieser Parameter wird typischerweise zusammen mit mehreren Hostnamen oder einem DNS-Eintrag verwendet, der mehrere IPs zurückgibt. Er kann zusammen mit `target_session_attrs` verwendet werden, um zum Beispiel nur über Standby-Server zu verteilen. Nach erfolgreichem Verbindungsaufbau werden alle nachfolgenden Abfragen auf der zurückgegebenen Verbindung an denselben Server gesendet. Derzeit gibt es zwei Modi:

- `disable` (Standard): Es wird keine Lastverteilung über Hosts durchgeführt. Hosts werden in der Reihenfolge versucht, in der sie angegeben wurden, und Adressen in der Reihenfolge, in der sie von DNS oder einer Hosts-Datei geliefert werden.
- `random`: Hosts und Adressen werden in zufälliger Reihenfolge versucht. Dieser Wert ist hauptsächlich nützlich, wenn mehrere Verbindungen gleichzeitig geöffnet werden, möglicherweise von verschiedenen Rechnern. So können Verbindungen über mehrere PostgreSQL-Server verteilt werden.

Zufällige Lastverteilung wird aufgrund ihres zufälligen Charakters fast nie zu einer vollständig gleichmäßigen Verteilung führen, kommt ihr statistisch aber recht nahe. Wichtig ist, dass dieser Algorithmus zwei Ebenen zufälliger Auswahl verwendet: Zuerst werden die Hosts in zufälliger Reihenfolge aufgelöst. Danach werden, bevor der nächste Host aufgelöst wird, alle aufgelösten Adressen des aktuellen Hosts in zufälliger Reihenfolge versucht. Dieses Verhalten kann in bestimmten Fällen stark verzerren, wie viele Verbindungen jeder Knoten erhält, etwa wenn manche Hosts auf mehr Adressen auflösen als andere. Eine solche Verzerrung kann aber auch absichtlich genutzt werden, zum Beispiel um einem größeren Server mehr Verbindungen zuzuweisen, indem sein Hostname mehrfach in der Host-Zeichenkette angegeben wird.

Bei Verwendung dieses Werts wird empfohlen, auch einen vernünftigen Wert für `connect_timeout` zu konfigurieren. Dann wird ein neuer Knoten versucht, wenn einer der für die Lastverteilung verwendeten Knoten nicht antwortet.

**`oauth_issuer`**

Die HTTPS-URL eines vertrauenswürdigen Ausstellers, der kontaktiert werden soll, wenn der Server für die Verbindung ein OAuth-Token anfordert. Dieser Parameter ist für alle OAuth-Verbindungen erforderlich; er sollte exakt mit der Einstellung `issuer` in der HBA-Konfiguration des Servers übereinstimmen.

Als Teil des normalen Authentifizierungs-Handshakes fragt `libpq` den Server nach einem Discovery-Dokument: einer URL, die eine Menge von OAuth-Konfigurationsparametern bereitstellt. Der Server muss eine URL bereitstellen, die direkt aus den Komponenten von `oauth_issuer` konstruiert ist, und dieser Wert muss exakt mit dem Ausstellerbezeichner übereinstimmen, der im Discovery-Dokument selbst deklariert ist; andernfalls schlägt die Verbindung fehl. Dies ist erforderlich, um eine Klasse von „Mix-up-Angriffen“ auf OAuth-Clients zu verhindern. Hintergrundinformationen finden Sie in der OAuth-Mailingliste: <https://mailarchive.ietf.org/arch/msg/oauth/JIVxFBGsJBVtm7ljwJhPUm3Fr-w/>.

Sie können `oauth_issuer` auch ausdrücklich auf den für OAuth-Discovery verwendeten `/.well-known/`-URI setzen. In diesem Fall schlägt die Verbindung fehl, wenn der Server eine andere URL anfordert; ein angepasster OAuth-Flow kann den Standard-Handshake jedoch möglicherweise beschleunigen, indem zuvor zwischengespeicherte Tokens verwendet werden. In diesem Fall wird empfohlen, auch `oauth_scope` zu setzen, da der Client keine Gelegenheit hat, den Server nach einer korrekten Scope-Einstellung zu fragen, und die Standard-Scopes für ein Token möglicherweise nicht ausreichen, um eine Verbindung herzustellen. `libpq` unterstützt derzeit die folgenden Well-known-Endpunkte:

- `/.well-known/openid-configuration`
- `/.well-known/oauth-authorization-server`

> **Warnung**
>
> Aussteller sind während des OAuth-Verbindungs-Handshakes hoch privilegiert. Als Faustregel gilt: Wenn Sie dem Betreiber einer URL nicht zutrauen würden, Zugriff auf Ihre Server zu verwalten oder Sie direkt zu imitieren, sollte diese URL nicht als `oauth_issuer` vertraut werden.

**`oauth_client_id`**

Ein OAuth-2.0-Clientbezeichner, wie er vom Autorisierungsserver ausgegeben wurde. Wenn der PostgreSQL-Server für die Verbindung ein OAuth-Token anfordert und kein angepasster OAuth-Hook installiert ist, um eines bereitzustellen, muss dieser Parameter gesetzt sein; andernfalls schlägt die Verbindung fehl.

**`oauth_client_secret`**

Das Clientpasswort, falls eines verwendet wird, beim Kontaktieren des OAuth-Autorisierungsservers. Ob dieser Parameter erforderlich ist oder nicht, bestimmt der OAuth-Anbieter; „öffentliche“ Clients verwenden im Allgemeinen kein Secret, während „vertrauliche“ Clients dies im Allgemeinen tun.

**`oauth_scope`**

Der Scope der Zugriffsanforderung, die an den Autorisierungsserver gesendet wird, angegeben als möglicherweise leere, durch Leerzeichen getrennte Liste von OAuth-Scope-Bezeichnern. Dieser Parameter ist optional und für fortgeschrittene Verwendung gedacht.

Normalerweise erhält der Client passende Scope-Einstellungen vom PostgreSQL-Server. Wenn dieser Parameter verwendet wird, wird die vom Server angeforderte Scope-Liste ignoriert. Dies kann verhindern, dass ein weniger vertrauenswürdiger Server vom Endbenutzer unangemessene Zugriffs-Scopes anfordert. Wenn die Scope-Einstellung des Clients jedoch nicht die vom Server benötigten Scopes enthält, wird der Server das ausgestellte Token wahrscheinlich ablehnen, und die Verbindung schlägt fehl.

Die Bedeutung einer leeren Scope-Liste ist anbieterabhängig. Ein OAuth-Autorisierungsserver kann entscheiden, ein Token mit einem beliebigen „Default Scope“ auszustellen, oder die Tokenanforderung vollständig ablehnen.

## 32.2. Funktionen zum Verbindungsstatus

Mit diesen Funktionen kann der Status eines vorhandenen Datenbankverbindungsobjekts abgefragt werden.

> **Tipp**
>
> `libpq`-Anwendungsprogrammierer sollten darauf achten, die `PGconn`-Abstraktion beizubehalten. Verwenden Sie die unten beschriebenen Zugriffsfunktionen, um an die Inhalte von `PGconn` zu gelangen. Der direkte Zugriff auf interne `PGconn`-Felder über `libpq-int.h` wird nicht empfohlen, weil diese sich in Zukunft ändern können.

Die folgenden Funktionen geben Parameterwerte zurück, die beim Verbindungsaufbau festgelegt wurden. Diese Werte sind für die Lebensdauer der Verbindung fest. Wenn ein Multi-Host-Verbindungsstring verwendet wird, können sich die Werte von `PQhost`, `PQport` und `PQpass` ändern, wenn mit demselben `PGconn`-Objekt eine neue Verbindung aufgebaut wird. Andere Werte sind für die Lebensdauer des `PGconn`-Objekts fest.

**`PQdb`**

Gibt den Datenbanknamen der Verbindung zurück.

```c
char *PQdb(const PGconn *conn);
```

**`PQuser`**

Gibt den Benutzernamen der Verbindung zurück.

```c
char *PQuser(const PGconn *conn);
```

**`PQpass`**

Gibt das Passwort der Verbindung zurück.

```c
char *PQpass(const PGconn *conn);
```

`PQpass` gibt entweder das in den Verbindungsparametern angegebene Passwort zurück oder, wenn dort keines angegeben war und das Passwort aus der Passwortdatei stammt, dieses Passwort. Im letzteren Fall kann man sich bei mehreren in den Verbindungsparametern angegebenen Hosts erst nach dem Verbindungsaufbau auf das Ergebnis von `PQpass` verlassen. Der Status der Verbindung kann mit der Funktion `PQstatus` geprüft werden.

**`PQhost`**

Gibt den Serverhostnamen der aktiven Verbindung zurück. Dies kann ein Hostname, eine IP-Adresse oder ein Verzeichnispfad sein, wenn die Verbindung über einen Unix-Socket läuft. Der Pfadfall ist daran zu erkennen, dass es immer ein absoluter Pfad ist, der mit `/` beginnt.

```c
char *PQhost(const PGconn *conn);
```

Wenn in den Verbindungsparametern sowohl `host` als auch `hostaddr` angegeben wurden, gibt `PQhost` die Hostinformation zurück. Wenn nur `hostaddr` angegeben wurde, wird diese zurückgegeben. Wenn mehrere Hosts in den Verbindungsparametern angegeben wurden, gibt `PQhost` den Host zurück, zu dem tatsächlich verbunden wurde.

`PQhost` gibt `NULL` zurück, wenn das Argument `conn` `NULL` ist. Andernfalls gibt es eine leere Zeichenkette zurück, wenn beim Erzeugen der Hostinformation ein Fehler auftritt, etwa weil die Verbindung noch nicht vollständig aufgebaut wurde oder ein Fehler vorlag.

Wenn mehrere Hosts in den Verbindungsparametern angegeben wurden, kann man sich erst nach dem Verbindungsaufbau auf das Ergebnis von `PQhost` verlassen. Der Status der Verbindung kann mit der Funktion `PQstatus` geprüft werden.

**`PQhostaddr`**

Gibt die Server-IP-Adresse der aktiven Verbindung zurück. Dies kann die Adresse sein, zu der ein Hostname aufgelöst wurde, oder eine über den Parameter `hostaddr` bereitgestellte IP-Adresse.

```c
char *PQhostaddr(const PGconn *conn);
```

`PQhostaddr` gibt `NULL` zurück, wenn das Argument `conn` `NULL` ist. Andernfalls gibt es eine leere Zeichenkette zurück, wenn beim Erzeugen der Hostinformation ein Fehler auftritt, etwa weil die Verbindung noch nicht vollständig aufgebaut wurde oder ein Fehler vorlag.

**`PQport`**

Gibt den Port der aktiven Verbindung zurück.

```c
char *PQport(const PGconn *conn);
```

Wenn mehrere Ports in den Verbindungsparametern angegeben wurden, gibt `PQport` den Port zurück, zu dem tatsächlich verbunden wurde.

`PQport` gibt `NULL` zurück, wenn das Argument `conn` `NULL` ist. Andernfalls gibt es eine leere Zeichenkette zurück, wenn beim Erzeugen der Portinformation ein Fehler auftritt, etwa weil die Verbindung noch nicht vollständig aufgebaut wurde oder ein Fehler vorlag.

Wenn mehrere Ports in den Verbindungsparametern angegeben wurden, kann man sich erst nach dem Verbindungsaufbau auf das Ergebnis von `PQport` verlassen. Der Status der Verbindung kann mit der Funktion `PQstatus` geprüft werden.

**`PQtty`**

Diese Funktion tut nichts mehr, bleibt aber aus Gründen der Abwärtskompatibilität erhalten. Die Funktion gibt immer eine leere Zeichenkette zurück, oder `NULL`, wenn das Argument `conn` `NULL` ist.

```c
char *PQtty(const PGconn *conn);
```

**`PQoptions`**

Gibt die Befehlszeilenoptionen zurück, die in der Verbindungsanforderung übergeben wurden.

```c
char *PQoptions(const PGconn *conn);
```

Die folgenden Funktionen geben Statusdaten zurück, die sich ändern können, während Operationen auf dem `PGconn`-Objekt ausgeführt werden.

**`PQstatus`**

Gibt den Status der Verbindung zurück.

```c
ConnStatusType PQstatus(const PGconn *conn);
```

Der Status kann verschiedene Werte haben. Außerhalb eines asynchronen Verbindungsaufbaus sieht man jedoch nur zwei davon: `CONNECTION_OK` und `CONNECTION_BAD`. Eine gute Verbindung zur Datenbank hat den Status `CONNECTION_OK`. Ein fehlgeschlagener Verbindungsversuch wird durch den Status `CONNECTION_BAD` signalisiert. Normalerweise bleibt ein OK-Status bis `PQfinish` erhalten, aber ein Kommunikationsfehler kann dazu führen, dass der Status vorzeitig zu `CONNECTION_BAD` wechselt. In diesem Fall könnte die Anwendung versuchen, sich durch Aufruf von `PQreset` zu erholen.

Weitere Statuscodes, die zurückgegeben werden können, finden Sie bei den Einträgen zu `PQconnectStartParams`, `PQconnectStart` und `PQconnectPoll`.

**`PQtransactionStatus`**

Gibt den aktuellen Transaktionsstatus des Servers zurück.

```c
PGTransactionStatusType PQtransactionStatus(const PGconn *conn);
```

Der Status kann `PQTRANS_IDLE` sein (derzeit untätig), `PQTRANS_ACTIVE` (ein Befehl läuft), `PQTRANS_INTRANS` (untätig, in einem gültigen Transaktionsblock) oder `PQTRANS_INERROR` (untätig, in einem fehlgeschlagenen Transaktionsblock). `PQTRANS_UNKNOWN` wird gemeldet, wenn die Verbindung schlecht ist. `PQTRANS_ACTIVE` wird nur gemeldet, wenn eine Abfrage an den Server gesendet wurde und noch nicht abgeschlossen ist.

**`PQparameterStatus`**

Schlägt eine aktuelle Parametereinstellung des Servers nach.

```c
const char *PQparameterStatus(const PGconn *conn, const char *paramName);
```

Bestimmte Parameterwerte werden vom Server automatisch beim Verbindungsstart oder immer dann gemeldet, wenn sich ihre Werte ändern. Mit `PQparameterStatus` können diese Einstellungen abgefragt werden. Die Funktion gibt den aktuellen Wert eines Parameters zurück, wenn er bekannt ist, oder `NULL`, wenn der Parameter nicht bekannt ist.

Zu den in der aktuellen Version gemeldeten Parametern gehören:

```text
application_name                                  scram_iterations
client_encoding                                   search_path
DateStyle                                         server_encoding
default_transaction_read_only                     server_version
in_hot_standby                                    session_authorization
integer_datetimes                                 standard_conforming_strings
IntervalStyle                                     TimeZone
is_superuser
```

`default_transaction_read_only` und `in_hot_standby` wurden von Versionen vor 14 nicht gemeldet; `scram_iterations` wurde von Versionen vor 16 nicht gemeldet; `search_path` wurde von Versionen vor 18 nicht gemeldet. Beachten Sie, dass `server_version`, `server_encoding` und `integer_datetimes` nach dem Start nicht geändert werden können.

Wenn kein Wert für `standard_conforming_strings` gemeldet wird, können Anwendungen annehmen, dass er `off` ist, das heißt, Backslashes werden in Zeichenkettenliteralen als Escape-Zeichen behandelt. Außerdem kann das Vorhandensein dieses Parameters als Hinweis gewertet werden, dass die Escape-String-Syntax (`E'...'`) akzeptiert wird.

Obwohl der zurückgegebene Zeiger als `const` deklariert ist, zeigt er tatsächlich auf veränderlichen Speicher, der mit der `PGconn`-Struktur verbunden ist. Es ist unklug anzunehmen, dass der Zeiger über Abfragen hinweg gültig bleibt.

**`PQfullProtocolVersion`**

Fragt das verwendete Frontend/Backend-Protokoll ab.

```c
int PQfullProtocolVersion(const PGconn *conn);
```

Anwendungen können diese Funktion verwenden, um zu bestimmen, ob bestimmte Funktionen unterstützt werden. Das Ergebnis wird gebildet, indem die Hauptversionsnummer des Servers mit 10000 multipliziert und die Nebenversionsnummer addiert wird. Zum Beispiel würde Version 3.2 als `30002` zurückgegeben, Version 4.0 als `40000`. Null wird zurückgegeben, wenn die Verbindung schlecht ist. Das Protokoll 3.0 wird von PostgreSQL-Serverversionen ab 7.4 unterstützt.

Die Protokollversion ändert sich nicht mehr, nachdem der Verbindungsstart abgeschlossen ist, könnte sich theoretisch aber während eines Verbindungsresets ändern.

**`PQprotocolVersion`**

Fragt die Hauptversion des Frontend/Backend-Protokolls ab.

```c
int PQprotocolVersion(const PGconn *conn);
```

Im Unterschied zu `PQfullProtocolVersion` gibt diese Funktion nur die verwendete Hauptprotokollversion zurück, wird aber von einer größeren Bandbreite von `libpq`-Versionen bis zurück zu Version 7.4 unterstützt. Derzeit sind die möglichen Werte `3` (Protokoll 3.0) oder null (schlechte Verbindung). Vor Release 14.0 konnte `libpq` zusätzlich `2` (Protokoll 2.0) zurückgeben.

**`PQserverVersion`**

Gibt eine Ganzzahl zurück, die die Serverversion repräsentiert.

```c
int PQserverVersion(const PGconn *conn);
```

Anwendungen können diese Funktion verwenden, um die Version des Datenbankservers zu bestimmen, mit dem sie verbunden sind. Das Ergebnis wird gebildet, indem die Hauptversionsnummer des Servers mit 10000 multipliziert und die Nebenversionsnummer addiert wird. Zum Beispiel wird Version 10.1 als `100001` zurückgegeben und Version 11.0 als `110000`. Null wird zurückgegeben, wenn die Verbindung schlecht ist.

Vor Hauptversion 10 verwendete PostgreSQL dreiteilige Versionsnummern, bei denen die ersten beiden Teile zusammen die Hauptversion repräsentierten. Für diese Versionen verwendet `PQserverVersion` zwei Ziffern für jeden Teil; beispielsweise wird Version 9.1.5 als `90105` zurückgegeben und Version 9.2.0 als `90200`.

Um die Funktionskompatibilität zu bestimmen, sollten Anwendungen daher das Ergebnis von `PQserverVersion` durch 100 und nicht durch 10000 teilen, um eine logische Hauptversionsnummer zu erhalten. In allen Release-Serien unterscheiden sich nur die letzten beiden Ziffern zwischen Minor-Releases, also Bugfix-Releases.

**`PQerrorMessage`**

Gibt die Fehlermeldung zurück, die zuletzt durch eine Operation auf der Verbindung erzeugt wurde.

```c
char *PQerrorMessage(const PGconn *conn);
```

Fast alle `libpq`-Funktionen setzen bei einem Fehlschlag eine Meldung für `PQerrorMessage`. Beachten Sie, dass nach `libpq`-Konvention ein nichtleeres Ergebnis von `PQerrorMessage` aus mehreren Zeilen bestehen kann und einen abschließenden Zeilenumbruch enthält. Der Aufrufer sollte das Ergebnis nicht direkt freigeben. Es wird freigegeben, wenn das zugehörige `PGconn`-Handle an `PQfinish` übergeben wird. Von der Ergebniszeichenkette sollte nicht erwartet werden, dass sie über Operationen auf der `PGconn`-Struktur hinweg gleich bleibt.

**`PQsocket`**

Ermittelt die Dateideskriptornummer des Verbindungssockets zum Server. Ein gültiger Deskriptor ist größer oder gleich 0; ein Ergebnis von `-1` zeigt an, dass derzeit keine Serververbindung geöffnet ist. Dies ändert sich im normalen Betrieb nicht, kann sich aber während des Verbindungsaufbaus oder beim Zurücksetzen der Verbindung ändern.

```c
int PQsocket(const PGconn *conn);
```

**`PQbackendPID`**

Gibt die Prozess-ID (PID) des Backend-Prozesses zurück, der diese Verbindung bearbeitet.

```c
int PQbackendPID(const PGconn *conn);
```

Die Backend-PID ist für Debugging-Zwecke und zum Vergleich mit `NOTIFY`-Meldungen nützlich, die die PID des benachrichtigenden Backend-Prozesses enthalten. Beachten Sie, dass die PID zu einem Prozess gehört, der auf dem Datenbankserverhost läuft, nicht auf dem lokalen Host.

**`PQconnectionNeedsPassword`**

Gibt wahr (`1`) zurück, wenn die Authentifizierungsmethode der Verbindung ein Passwort verlangte, aber keines verfügbar war. Andernfalls wird falsch (`0`) zurückgegeben.

```c
int PQconnectionNeedsPassword(const PGconn *conn);
```

Diese Funktion kann nach einem fehlgeschlagenen Verbindungsversuch verwendet werden, um zu entscheiden, ob der Benutzer nach einem Passwort gefragt werden soll.

**`PQconnectionUsedPassword`**

Gibt wahr (`1`) zurück, wenn die Authentifizierungsmethode der Verbindung ein Passwort verwendet hat. Andernfalls wird falsch (`0`) zurückgegeben.

```c
int PQconnectionUsedPassword(const PGconn *conn);
```

Diese Funktion kann nach einem fehlgeschlagenen oder erfolgreichen Verbindungsversuch verwendet werden, um zu erkennen, ob der Server ein Passwort verlangt hat.

**`PQconnectionUsedGSSAPI`**

Gibt wahr (`1`) zurück, wenn die Authentifizierungsmethode der Verbindung GSSAPI verwendet hat. Andernfalls wird falsch (`0`) zurückgegeben.

```c
int PQconnectionUsedGSSAPI(const PGconn *conn);
```

Diese Funktion kann verwendet werden, um zu erkennen, ob die Verbindung mit GSSAPI authentifiziert wurde.

Die folgenden Funktionen geben SSL-bezogene Informationen zurück. Diese Informationen ändern sich normalerweise nach dem Herstellen einer Verbindung nicht.

**`PQsslInUse`**

Gibt wahr (`1`) zurück, wenn die Verbindung SSL verwendet, andernfalls falsch (`0`).

```c
int PQsslInUse(const PGconn *conn);
```

**`PQsslAttribute`**

Gibt SSL-bezogene Informationen über die Verbindung zurück.

```c
const char *PQsslAttribute(const PGconn *conn, const char *attribute_name);
```

Die Liste der verfügbaren Attribute variiert je nach verwendeter SSL-Bibliothek und Verbindungstyp. Die Funktion gibt `NULL` zurück, wenn die Verbindung kein SSL verwendet oder wenn der angegebene Attributname für die verwendete Bibliothek nicht definiert ist.

Die folgenden Attribute sind häufig verfügbar:

- `library`: Name der verwendeten SSL-Implementierung. Derzeit ist nur `OpenSSL` implementiert.
- `protocol`: Verwendete SSL/TLS-Version. Häufige Werte sind `TLSv1`, `TLSv1.1` und `TLSv1.2`, aber eine Implementierung kann andere Zeichenketten zurückgeben, wenn ein anderes Protokoll verwendet wird.
- `key_bits`: Anzahl der Schlüsselbits, die vom Verschlüsselungsalgorithmus verwendet werden.
- `cipher`: Kurzname der verwendeten Cipher-Suite, zum Beispiel `DHE-RSA-DES-CBC3-SHA`. Die Namen sind spezifisch für jede SSL-Implementierung.
- `compression`: Gibt `on` zurück, wenn SSL-Kompression verwendet wird, andernfalls `off`.
- `alpn`: Anwendungsprotokoll, das von der TLS-Erweiterung Application-Layer Protocol Negotiation (ALPN) ausgewählt wurde. Das einzige von `libpq` unterstützte Protokoll ist `postgresql`; daher ist dies hauptsächlich nützlich, um zu prüfen, ob der Server ALPN unterstützt hat. Eine leere Zeichenkette bedeutet, dass ALPN nicht verwendet wurde.

Als Sonderfall kann das Attribut `library` ohne Verbindung abgefragt werden, indem `NULL` als Argument `conn` übergeben wird. Das Ergebnis ist der Name der Standard-SSL-Bibliothek oder `NULL`, wenn `libpq` ohne SSL-Unterstützung kompiliert wurde. Vor PostgreSQL Version 15 führte die Übergabe von `NULL` als Argument `conn` immer zu `NULL`. Clientprogramme, die zwischen der neueren und der älteren Implementierung dieses Falls unterscheiden müssen, können das Feature-Makro `LIBPQ_HAS_SSL_LIBRARY_DETECTION` prüfen.

**`PQsslAttributeNames`**

Gibt ein Array von SSL-Attributnamen zurück, die in `PQsslAttribute()` verwendet werden können. Das Array wird durch einen `NULL`-Zeiger beendet.

```c
const char * const * PQsslAttributeNames(const PGconn *conn);
```

Wenn `conn` `NULL` ist, werden die für die Standard-SSL-Bibliothek verfügbaren Attribute zurückgegeben, oder eine leere Liste, wenn `libpq` ohne SSL-Unterstützung kompiliert wurde. Wenn `conn` nicht `NULL` ist, werden die Attribute zurückgegeben, die für die in der Verbindung verwendete SSL-Bibliothek verfügbar sind, oder eine leere Liste, wenn die Verbindung nicht verschlüsselt ist.

**`PQsslStruct`**

Gibt einen Zeiger auf ein SSL-implementierungsspezifisches Objekt zurück, das die Verbindung beschreibt. Die Funktion gibt `NULL` zurück, wenn die Verbindung nicht verschlüsselt ist oder wenn der angeforderte Objekttyp von der SSL-Implementierung der Verbindung nicht verfügbar ist.

```c
void *PQsslStruct(const PGconn *conn, const char *struct_name);
```

Die verfügbaren Strukturen hängen von der verwendeten SSL-Implementierung ab. Für OpenSSL gibt es eine Struktur unter dem Namen `OpenSSL`; sie gibt einen Zeiger auf die `SSL`-Struktur von OpenSSL zurück. Um diese Funktion zu verwenden, könnte Code in der folgenden Art eingesetzt werden:

```c
#include <libpq-fe.h>
#include <openssl/ssl.h>

...

SSL *ssl;

dbconn = PQconnectdb(...);
...

ssl = PQsslStruct(dbconn, "OpenSSL");
if (ssl)
{
    /* use OpenSSL functions to access ssl */
}
```

Diese Struktur kann verwendet werden, um Verschlüsselungsstufen zu überprüfen, Serverzertifikate zu prüfen und mehr. Informationen zu dieser Struktur finden Sie in der OpenSSL-Dokumentation.

**`PQgetssl`**

Gibt die in der Verbindung verwendete SSL-Struktur zurück, oder `NULL`, wenn SSL nicht verwendet wird.

```c
void *PQgetssl(const PGconn *conn);
```

Diese Funktion ist äquivalent zu `PQsslStruct(conn, "OpenSSL")`. Sie sollte in neuen Anwendungen nicht verwendet werden, weil die zurückgegebene Struktur OpenSSL-spezifisch ist und nicht verfügbar sein wird, wenn eine andere SSL-Implementierung verwendet wird. Um zu prüfen, ob eine Verbindung SSL verwendet, rufen Sie stattdessen `PQsslInUse` auf; für weitere Details zur Verbindung verwenden Sie `PQsslAttribute`.

## 32.3. Funktionen zur Befehlsausführung

Nachdem eine Verbindung zu einem Datenbankserver erfolgreich hergestellt wurde, werden die hier beschriebenen Funktionen verwendet, um SQL-Abfragen und Befehle auszuführen.

### 32.3.1. Hauptfunktionen

**`PQexec`**

Sendet einen Befehl an den Server und wartet auf das Ergebnis.

```c
PGresult *PQexec(PGconn *conn, const char *command);
```

Die Funktion gibt einen Zeiger auf `PGresult` oder möglicherweise einen Nullzeiger zurück. Ein Nicht-Null-Zeiger wird normalerweise zurückgegeben, außer bei Speichermangel oder schwerwiegenden Fehlern, etwa wenn der Befehl nicht an den Server gesendet werden kann. `PQresultStatus` sollte aufgerufen werden, um den Rückgabewert auf Fehler zu prüfen, einschließlich eines Nullzeigers, bei dem `PQresultStatus` `PGRES_FATAL_ERROR` zurückgibt. Mit `PQerrorMessage` erhält man weitere Informationen zu solchen Fehlern.

Die Befehlszeichenkette kann mehrere SQL-Befehle enthalten, die durch Semikolons getrennt sind. Mehrere Abfragen, die mit einem einzigen `PQexec`-Aufruf gesendet werden, werden in einer einzigen Transaktion verarbeitet, sofern keine expliziten `BEGIN`-/`COMMIT`-Befehle in der Abfragezeichenkette enthalten sind, die sie in mehrere Transaktionen aufteilen. Weitere Einzelheiten dazu, wie der Server Mehrfachabfrage-Zeichenketten behandelt, finden sich in Abschnitt 54.2.2.1. Beachten Sie jedoch, dass die zurückgegebene `PGresult`-Struktur nur das Ergebnis des letzten aus der Zeichenkette ausgeführten Befehls beschreibt.

Wenn einer der Befehle fehlschlägt, endet die Verarbeitung der Zeichenkette an dieser Stelle, und das zurückgegebene `PGresult` beschreibt den Fehlerzustand.

**`PQexecParams`**

Sendet einen Befehl an den Server und wartet auf das Ergebnis, wobei Parameter getrennt vom SQL-Befehlstext übergeben werden können.

```c
PGresult *PQexecParams(PGconn *conn,
                       const char *command,
                       int nParams,
                       const Oid *paramTypes,
                       const char * const *paramValues,
                       const int *paramLengths,
                       const int *paramFormats,
                       int resultFormat);
```

`PQexecParams` ähnelt `PQexec`, bietet aber zusätzliche Möglichkeiten: Parameterwerte können getrennt von der eigentlichen Befehlszeichenkette angegeben werden, und Abfrageergebnisse können entweder im Text- oder im Binärformat angefordert werden.

Die Funktionsargumente sind:

**`conn`**

Das Verbindungsobjekt, über das der Befehl gesendet wird.

**`command`**

Die auszuführende SQL-Befehlszeichenkette. Wenn Parameter verwendet werden, werden sie in der Befehlszeichenkette als `$1`, `$2` usw. angesprochen.

**`nParams`**

Die Anzahl der übergebenen Parameter; sie ist die Länge der Arrays `paramTypes[]`, `paramValues[]`, `paramLengths[]` und `paramFormats[]`. Die Array-Zeiger können `NULL` sein, wenn `nParams` null ist.

**`paramTypes[]`**

Gibt über OIDs die Datentypen an, die den Parametersymbolen zugewiesen werden sollen. Wenn `paramTypes` `NULL` ist oder ein bestimmtes Element im Array null ist, leitet der Server den Datentyp des Parametersymbols genauso ab wie bei einer typfreien Literalzeichenkette.

**`paramValues[]`**

Gibt die tatsächlichen Werte der Parameter an. Ein Nullzeiger in diesem Array bedeutet, dass der entsprechende Parameter null ist; andernfalls zeigt der Zeiger auf eine nullterminierte Textzeichenkette im Textformat oder auf Binärdaten im vom Server erwarteten Format.

**`paramLengths[]`**

Gibt die tatsächliche Datenlänge von Parametern im Binärformat an. Für Nullparameter und Parameter im Textformat wird dies ignoriert. Der Array-Zeiger kann null sein, wenn es keine binären Parameter gibt.

**`paramFormats[]`**

Gibt an, ob Parameter Text sind (null im Array-Eintrag des entsprechenden Parameters) oder binär (eins im Array-Eintrag des entsprechenden Parameters). Wenn der Array-Zeiger null ist, werden alle Parameter als Textzeichenketten angenommen.

Im Binärformat übergebene Werte erfordern Kenntnis der internen Darstellung, die das Backend erwartet. Beispielsweise müssen Ganzzahlen in Netzwerk-Byte-Reihenfolge übergeben werden. Für numerische Werte muss das Speicherformat des Servers bekannt sein, wie es in `src/backend/utils/adt/numeric.c::numeric_send()` und `src/backend/utils/adt/numeric.c::numeric_recv()` implementiert ist.

**`resultFormat`**

Geben Sie null an, um Ergebnisse im Textformat zu erhalten, oder eins, um Ergebnisse im Binärformat zu erhalten. Es gibt derzeit keine Möglichkeit, verschiedene Ergebnisspalten in verschiedenen Formaten zu erhalten, obwohl das zugrunde liegende Protokoll dies grundsätzlich zulässt.

Der Hauptvorteil von `PQexecParams` gegenüber `PQexec` besteht darin, dass Parameterwerte von der Befehlszeichenkette getrennt werden können. Dadurch entfällt das mühsame und fehleranfällige Quotieren und Escapen.

Anders als `PQexec` erlaubt `PQexecParams` höchstens einen SQL-Befehl in der angegebenen Zeichenkette. Es dürfen Semikolons darin vorkommen, aber nicht mehr als ein nichtleerer Befehl. Dies ist eine Einschränkung des zugrunde liegenden Protokolls, kann aber auch als zusätzliche Abwehr gegen SQL-Injection-Angriffe nützlich sein.

> **Tipp**
>
> Parametertypen über OIDs anzugeben ist umständlich, besonders wenn man keine bestimmten OID-Werte fest im Programm verdrahten möchte. Man kann dies jedoch auch dann vermeiden, wenn der Server den Parametertyp nicht selbst bestimmen kann oder einen anderen Typ wählen würde als gewünscht. Hängen Sie im SQL-Befehlstext eine explizite Typumwandlung an das Parametersymbol an, um zu zeigen, welchen Datentyp Sie senden werden. Zum Beispiel:
>
> ```sql
> SELECT * FROM mytable WHERE x = $1::bigint;
> ```
>
> Dadurch wird Parameter `$1` gezwungen, als `bigint` behandelt zu werden, während er standardmäßig denselben Typ wie `x` erhalten würde. Die Parametertypentscheidung auf diese Weise oder durch Angabe einer numerischen Typ-OID zu erzwingen, wird dringend empfohlen, wenn Parameterwerte im Binärformat gesendet werden, weil das Binärformat weniger Redundanz als das Textformat besitzt und der Server einen Typkonflikt daher weniger wahrscheinlich für Sie erkennt.

**`PQprepare`**

Sendet eine Anforderung zum Erstellen einer vorbereiteten Anweisung mit den angegebenen Parametern und wartet auf den Abschluss.

```c
PGresult *PQprepare(PGconn *conn,
                    const char *stmtName,
                    const char *query,
                    int nParams,
                    const Oid *paramTypes);
```

`PQprepare` erzeugt eine vorbereitete Anweisung zur späteren Ausführung mit `PQexecPrepared`. Dadurch können Befehle wiederholt ausgeführt werden, ohne jedes Mal neu geparst und geplant zu werden; Einzelheiten finden sich bei `PREPARE`.

Die Funktion erzeugt aus der Abfragezeichenkette eine vorbereitete Anweisung mit dem Namen `stmtName`; die Zeichenkette muss einen einzelnen SQL-Befehl enthalten. `stmtName` kann `""` sein, um eine unbenannte Anweisung zu erzeugen. In diesem Fall wird eine bereits vorhandene unbenannte Anweisung automatisch ersetzt; andernfalls ist es ein Fehler, wenn der Anweisungsname in der aktuellen Sitzung bereits definiert ist. Wenn Parameter verwendet werden, werden sie in der Abfrage als `$1`, `$2` usw. angesprochen. `nParams` ist die Anzahl der Parameter, deren Typen im Array `paramTypes[]` vorgegeben sind. Der Array-Zeiger kann `NULL` sein, wenn `nParams` null ist. `paramTypes[]` gibt über OIDs die Datentypen an, die den Parametersymbolen zugewiesen werden sollen. Wenn `paramTypes` `NULL` ist oder ein bestimmtes Element im Array null ist, weist der Server dem Parametersymbol auf dieselbe Weise einen Datentyp zu wie einer typfreien Literalzeichenkette. Außerdem kann die Abfrage Parametersymbole mit Nummern verwenden, die größer als `nParams` sind; auch für diese Symbole werden Datentypen abgeleitet. Mit `PQdescribePrepared` kann man feststellen, welche Datentypen abgeleitet wurden.

Wie bei `PQexec` ist das Ergebnis normalerweise ein `PGresult`-Objekt, dessen Inhalt Erfolg oder Fehlschlag auf Serverseite anzeigt. Ein Nullergebnis zeigt Speichermangel oder die Unfähigkeit an, den Befehl überhaupt zu senden. Mit `PQerrorMessage` erhält man weitere Informationen zu solchen Fehlern.

Vorbereitete Anweisungen zur Verwendung mit `PQexecPrepared` können auch durch Ausführen von SQL-`PREPARE`-Anweisungen erzeugt werden.

**`PQexecPrepared`**

Sendet eine Anforderung zum Ausführen einer vorbereiteten Anweisung mit angegebenen Parametern und wartet auf das Ergebnis.

```c
PGresult *PQexecPrepared(PGconn *conn,
                         const char *stmtName,
                         int nParams,
                         const char * const *paramValues,
                         const int *paramLengths,
                         const int *paramFormats,
                         int resultFormat);
```

`PQexecPrepared` ähnelt `PQexecParams`, aber der auszuführende Befehl wird durch Benennung einer zuvor vorbereiteten Anweisung angegeben, nicht durch eine Abfragezeichenkette. Damit können Befehle, die wiederholt verwendet werden, nur einmal geparst und geplant werden, statt bei jeder Ausführung erneut. Die Anweisung muss zuvor in der aktuellen Sitzung vorbereitet worden sein.

Die Parameter sind identisch mit denen von `PQexecParams`, außer dass der Name einer vorbereiteten Anweisung statt einer Abfragezeichenkette angegeben wird und der Parameter `paramTypes[]` fehlt, weil die Parametertypen der vorbereiteten Anweisung bei ihrer Erstellung bestimmt wurden.

**`PQdescribePrepared`**

Sendet eine Anforderung, Informationen über die angegebene vorbereitete Anweisung zu erhalten, und wartet auf den Abschluss.

```c
PGresult *PQdescribePrepared(PGconn *conn, const char *stmtName);
```

`PQdescribePrepared` erlaubt einer Anwendung, Informationen über eine zuvor vorbereitete Anweisung abzufragen.

`stmtName` kann `""` oder `NULL` sein, um die unbenannte Anweisung zu referenzieren; andernfalls muss es der Name einer bestehenden vorbereiteten Anweisung sein. Bei Erfolg wird ein `PGresult` mit dem Status `PGRES_COMMAND_OK` zurückgegeben. Die Funktionen `PQnparams` und `PQparamtype` können auf dieses `PGresult` angewendet werden, um Informationen über die Parameter der vorbereiteten Anweisung zu erhalten; die Funktionen `PQnfields`, `PQfname`, `PQftype` usw. liefern Informationen über die Ergebnisspalten der Anweisung, falls vorhanden.

**`PQdescribePortal`**

Sendet eine Anforderung, Informationen über das angegebene Portal zu erhalten, und wartet auf den Abschluss.

```c
PGresult *PQdescribePortal(PGconn *conn, const char *portalName);
```

`PQdescribePortal` erlaubt einer Anwendung, Informationen über ein zuvor erzeugtes Portal abzufragen. `libpq` bietet keinen direkten Zugriff auf Portale, aber mit dieser Funktion können die Eigenschaften eines Cursors untersucht werden, der mit einem SQL-Befehl `DECLARE CURSOR` erstellt wurde.

`portalName` kann `""` oder `NULL` sein, um das unbenannte Portal zu referenzieren; andernfalls muss es der Name eines bestehenden Portals sein. Bei Erfolg wird ein `PGresult` mit dem Status `PGRES_COMMAND_OK` zurückgegeben. Die Funktionen `PQnfields`, `PQfname`, `PQftype` usw. können auf das `PGresult` angewendet werden, um Informationen über die Ergebnisspalten des Portals zu erhalten, falls vorhanden.

**`PQclosePrepared`**

Sendet eine Anforderung zum Schließen der angegebenen vorbereiteten Anweisung und wartet auf den Abschluss.

```c
PGresult *PQclosePrepared(PGconn *conn, const char *stmtName);
```

`PQclosePrepared` erlaubt einer Anwendung, eine zuvor vorbereitete Anweisung zu schließen. Das Schließen einer Anweisung gibt alle zugehörigen Ressourcen auf dem Server frei und erlaubt, ihren Namen wiederzuverwenden.

`stmtName` kann `""` oder `NULL` sein, um die unbenannte Anweisung zu referenzieren. Wenn keine Anweisung mit diesem Namen existiert, ist das in Ordnung; die Operation bewirkt dann nichts. Bei Erfolg wird ein `PGresult` mit dem Status `PGRES_COMMAND_OK` zurückgegeben.

**`PQclosePortal`**

Sendet eine Anforderung zum Schließen des angegebenen Portals und wartet auf den Abschluss.

```c
PGresult *PQclosePortal(PGconn *conn, const char *portalName);
```

`PQclosePortal` erlaubt einer Anwendung, ein zuvor erzeugtes Portal zu schließen. Das Schließen eines Portals gibt alle zugehörigen Ressourcen auf dem Server frei und erlaubt, seinen Namen wiederzuverwenden. `libpq` bietet keinen direkten Zugriff auf Portale, aber mit dieser Funktion können Sie einen Cursor schließen, der mit einem SQL-Befehl `DECLARE CURSOR` erstellt wurde.

`portalName` kann `""` oder `NULL` sein, um das unbenannte Portal zu referenzieren. Wenn kein Portal mit diesem Namen existiert, ist das in Ordnung; die Operation bewirkt dann nichts. Bei Erfolg wird ein `PGresult` mit dem Status `PGRES_COMMAND_OK` zurückgegeben.

Die Struktur `PGresult` kapselt das vom Server zurückgegebene Ergebnis. `libpq`-Anwendungsprogrammierer sollten darauf achten, die `PGresult`-Abstraktion beizubehalten. Verwenden Sie die unten beschriebenen Zugriffsfunktionen, um an die Inhalte von `PGresult` zu gelangen. Der direkte Zugriff auf Felder der `PGresult`-Struktur sollte vermieden werden, weil sie sich in Zukunft ändern können.

**`PQresultStatus`**

Gibt den Ergebnisstatus des Befehls zurück.

```c
ExecStatusType PQresultStatus(const PGresult *res);
```

`PQresultStatus` kann einen der folgenden Werte zurückgeben:

**`PGRES_EMPTY_QUERY`**

Die an den Server gesendete Zeichenkette war leer.

**`PGRES_COMMAND_OK`**

Ein Befehl, der keine Daten zurückgibt, wurde erfolgreich abgeschlossen.

**`PGRES_TUPLES_OK`**

Ein Befehl, der Daten zurückgibt, etwa `SELECT` oder `SHOW`, wurde erfolgreich abgeschlossen.

**`PGRES_COPY_OUT`**

Eine `COPY OUT`-Datenübertragung vom Server wurde gestartet.

**`PGRES_COPY_IN`**

Eine `COPY IN`-Datenübertragung zum Server wurde gestartet.

**`PGRES_BAD_RESPONSE`**

Die Antwort des Servers wurde nicht verstanden.

**`PGRES_NONFATAL_ERROR`**

Ein nicht schwerwiegender Fehler, eine Notice oder eine Warnung ist aufgetreten.

**`PGRES_FATAL_ERROR`**

Ein schwerwiegender Fehler ist aufgetreten.

**`PGRES_COPY_BOTH`**

Eine `COPY IN`-/`COPY OUT`-Datenübertragung zum und vom Server wurde gestartet. Diese Funktion wird derzeit nur für Streaming-Replikation verwendet, daher sollte dieser Status in gewöhnlichen Anwendungen nicht auftreten.

**`PGRES_SINGLE_TUPLE`**

Das `PGresult` enthält ein einzelnes Ergebnistupel aus dem aktuellen Befehl. Dieser Status tritt nur auf, wenn für die Abfrage der Einzelzeilenmodus gewählt wurde (siehe [Abschnitt 32.6](32_libpq_C_Bibliothek.md#326-abfrageergebnisse-in-chunks-abrufen)).

**`PGRES_TUPLES_CHUNK`**

Das `PGresult` enthält mehrere Ergebnistupel aus dem aktuellen Befehl. Dieser Status tritt nur auf, wenn für die Abfrage der Chunk-Modus gewählt wurde (siehe [Abschnitt 32.6](32_libpq_C_Bibliothek.md#326-abfrageergebnisse-in-chunks-abrufen)). Die Anzahl der Tupel überschreitet nicht die an `PQsetChunkedRowsMode` übergebene Grenze.

**`PGRES_PIPELINE_SYNC`**

Das `PGresult` stellt einen Synchronisationspunkt im Pipeline-Modus dar, der entweder durch `PQpipelineSync` oder `PQsendPipelineSync` angefordert wurde. Dieser Status tritt nur auf, wenn der Pipeline-Modus gewählt wurde.

**`PGRES_PIPELINE_ABORTED`**

Das `PGresult` stellt eine Pipeline dar, die vom Server einen Fehler erhalten hat. `PQgetResult` muss wiederholt aufgerufen werden; jedes Mal wird dieser Statuscode zurückgegeben, bis das Ende der aktuellen Pipeline erreicht ist. Danach wird `PGRES_PIPELINE_SYNC` zurückgegeben, und die normale Verarbeitung kann fortgesetzt werden.

Wenn der Ergebnisstatus `PGRES_TUPLES_OK`, `PGRES_SINGLE_TUPLE` oder `PGRES_TUPLES_CHUNK` ist, können die unten beschriebenen Funktionen verwendet werden, um die von der Abfrage zurückgegebenen Zeilen abzurufen. Beachten Sie, dass ein `SELECT`-Befehl, der zufällig null Zeilen abruft, trotzdem `PGRES_TUPLES_OK` liefert. `PGRES_COMMAND_OK` ist für Befehle gedacht, die niemals Zeilen zurückgeben können, etwa `INSERT` oder `UPDATE` ohne `RETURNING`-Klausel. Eine Antwort `PGRES_EMPTY_QUERY` könnte auf einen Fehler in der Clientsoftware hinweisen.

Ein Ergebnis mit Status `PGRES_NONFATAL_ERROR` wird niemals direkt von `PQexec` oder anderen Abfrageausführungsfunktionen zurückgegeben. Ergebnisse dieser Art werden stattdessen an den Notice-Prozessor weitergereicht (siehe [Abschnitt 32.13](32_libpq_C_Bibliothek.md#3213-noticeverarbeitung)).

**`PQresStatus`**

Wandelt den von `PQresultStatus` zurückgegebenen Aufzählungstyp in eine Zeichenkettenkonstante um, die den Statuscode beschreibt. Der Aufrufer sollte das Ergebnis nicht freigeben.

```c
char *PQresStatus(ExecStatusType status);
```

**`PQresultErrorMessage`**

Gibt die mit dem Befehl verbundene Fehlermeldung zurück, oder eine leere Zeichenkette, wenn kein Fehler vorlag.

```c
char *PQresultErrorMessage(const PGresult *res);
```

Wenn ein Fehler vorlag, enthält die zurückgegebene Zeichenkette einen abschließenden Zeilenumbruch. Der Aufrufer sollte das Ergebnis nicht direkt freigeben. Es wird freigegeben, wenn das zugehörige `PGresult`-Handle an `PQclear` übergeben wird.

Unmittelbar nach einem Aufruf von `PQexec` oder `PQgetResult` gibt `PQerrorMessage` auf der Verbindung dieselbe Zeichenkette zurück wie `PQresultErrorMessage` auf dem Ergebnis. Ein `PGresult` behält seine Fehlermeldung jedoch bis zu seiner Zerstörung, während sich die Fehlermeldung der Verbindung bei nachfolgenden Operationen ändert. Verwenden Sie `PQresultErrorMessage`, wenn Sie den Status eines bestimmten `PGresult` kennen möchten; verwenden Sie `PQerrorMessage`, wenn Sie den Status der letzten Operation auf der Verbindung benötigen.

**`PQresultVerboseErrorMessage`**

Gibt eine neu formatierte Version der Fehlermeldung zurück, die mit einem `PGresult`-Objekt verbunden ist.

```c
char *PQresultVerboseErrorMessage(const PGresult *res,
                                  PGVerbosity verbosity,
                                  PGContextVisibility show_context);
```

In manchen Situationen möchte ein Client eine ausführlichere Version eines zuvor gemeldeten Fehlers erhalten. `PQresultVerboseErrorMessage` erfüllt diesen Zweck, indem die Meldung berechnet wird, die `PQresultErrorMessage` erzeugt hätte, wenn die angegebenen Ausführlichkeitseinstellungen für die Verbindung gegolten hätten, als das angegebene `PGresult` erzeugt wurde. Wenn das `PGresult` kein Fehlerergebnis ist, wird stattdessen `PGresult is not an error result` gemeldet. Die zurückgegebene Zeichenkette enthält einen abschließenden Zeilenumbruch.

Anders als bei den meisten anderen Funktionen zum Extrahieren von Daten aus einem `PGresult` ist das Ergebnis dieser Funktion eine frisch allozierte Zeichenkette. Der Aufrufer muss sie mit `PQfreemem()` freigeben, wenn sie nicht mehr benötigt wird.

Bei unzureichendem Speicher kann `NULL` zurückgegeben werden.

**`PQresultErrorField`**

Gibt ein einzelnes Feld eines Fehlerberichts zurück.

```c
char *PQresultErrorField(const PGresult *res, int fieldcode);
```

`fieldcode` ist ein Fehlerfeldbezeichner; siehe die unten aufgeführten Symbole. `NULL` wird zurückgegeben, wenn das `PGresult` kein Fehler- oder Warnungsergebnis ist oder das angegebene Feld nicht enthält. Feldwerte enthalten normalerweise keinen abschließenden Zeilenumbruch. Der Aufrufer sollte das Ergebnis nicht direkt freigeben. Es wird freigegeben, wenn das zugehörige `PGresult`-Handle an `PQclear` übergeben wird.

Die folgenden Feldcodes sind verfügbar:

**`PG_DIAG_SEVERITY`**

Die Schwere; der Feldinhalt ist `ERROR`, `FATAL` oder `PANIC` in einer Fehlermeldung, oder `WARNING`, `NOTICE`, `DEBUG`, `INFO` oder `LOG` in einer Notice-Meldung, oder eine lokalisierte Übersetzung eines dieser Werte. Immer vorhanden.

**`PG_DIAG_SEVERITY_NONLOCALIZED`**

Die Schwere; der Feldinhalt ist `ERROR`, `FATAL` oder `PANIC` in einer Fehlermeldung, oder `WARNING`, `NOTICE`, `DEBUG`, `INFO` oder `LOG` in einer Notice-Meldung. Dies ist identisch mit dem Feld `PG_DIAG_SEVERITY`, außer dass der Inhalt niemals lokalisiert ist. Es ist nur in Berichten vorhanden, die von PostgreSQL-Versionen 9.6 und neuer erzeugt wurden.

**`PG_DIAG_SQLSTATE`**

Der SQLSTATE-Code des Fehlers. Der SQLSTATE-Code identifiziert die Art des aufgetretenen Fehlers; Frontend-Anwendungen können ihn verwenden, um als Reaktion auf einen bestimmten Datenbankfehler gezielte Operationen auszuführen, etwa Fehlerbehandlung. Eine Liste der möglichen SQLSTATE-Codes steht in Anhang A. Dieses Feld ist nicht lokalisierbar und immer vorhanden.

**`PG_DIAG_MESSAGE_PRIMARY`**

Die primäre menschenlesbare Fehlermeldung, typischerweise eine Zeile. Immer vorhanden.

**`PG_DIAG_MESSAGE_DETAIL`**

Detail: eine optionale sekundäre Fehlermeldung mit mehr Einzelheiten zum Problem. Sie kann mehrere Zeilen umfassen.

**`PG_DIAG_MESSAGE_HINT`**

Hinweis: ein optionaler Vorschlag, was gegen das Problem getan werden kann. Er soll sich von Detail dadurch unterscheiden, dass er Rat gibt, der potenziell unpassend sein kann, statt harte Fakten zu liefern. Er kann mehrere Zeilen umfassen.

**`PG_DIAG_STATEMENT_POSITION`**

Eine Zeichenkette mit einer Dezimalzahl, die eine Fehlercursorposition als Index in der ursprünglichen Anweisungszeichenkette angibt. Das erste Zeichen hat Index 1, und Positionen werden in Zeichen gemessen, nicht in Bytes.

**`PG_DIAG_INTERNAL_POSITION`**

Dies ist genauso definiert wie das Feld `PG_DIAG_STATEMENT_POSITION`, wird aber verwendet, wenn sich die Cursorposition auf einen intern erzeugten Befehl bezieht und nicht auf den vom Client übermittelten. Das Feld `PG_DIAG_INTERNAL_QUERY` erscheint immer, wenn dieses Feld erscheint.

**`PG_DIAG_INTERNAL_QUERY`**

Der Text eines fehlgeschlagenen intern erzeugten Befehls. Dies könnte zum Beispiel eine SQL-Abfrage sein, die von einer PL/pgSQL-Funktion ausgegeben wurde.

**`PG_DIAG_CONTEXT`**

Ein Hinweis auf den Kontext, in dem der Fehler auftrat. Derzeit umfasst dies eine Stack-Trace-Ausgabe aktiver Funktionen prozeduraler Sprachen und intern erzeugter Abfragen. Der Trace enthält einen Eintrag pro Zeile, den jüngsten zuerst.

**`PG_DIAG_SCHEMA_NAME`**

Wenn der Fehler mit einem bestimmten Datenbankobjekt verbunden war, der Name des Schemas, das dieses Objekt enthält, falls vorhanden.

**`PG_DIAG_TABLE_NAME`**

Wenn der Fehler mit einer bestimmten Tabelle verbunden war, der Name der Tabelle. Der Name des Schemas der Tabelle steht im Schema-Namensfeld.

**`PG_DIAG_COLUMN_NAME`**

Wenn der Fehler mit einer bestimmten Tabellenspalte verbunden war, der Name der Spalte. Das Schema- und Tabellen-Namensfeld identifizieren die Tabelle.

**`PG_DIAG_DATATYPE_NAME`**

Wenn der Fehler mit einem bestimmten Datentyp verbunden war, der Name des Datentyps. Der Name des Schemas des Datentyps steht im Schema-Namensfeld.

**`PG_DIAG_CONSTRAINT_NAME`**

Wenn der Fehler mit einer bestimmten Constraint verbunden war, der Name der Constraint. Die oben aufgeführten Felder verweisen auf die zugehörige Tabelle oder Domäne. Zu diesem Zweck werden Indizes als Constraints behandelt, auch wenn sie nicht mit Constraint-Syntax erzeugt wurden.

**`PG_DIAG_SOURCE_FILE`**

Der Dateiname der Quellcodeposition, an der der Fehler gemeldet wurde.

**`PG_DIAG_SOURCE_LINE`**

Die Zeilennummer der Quellcodeposition, an der der Fehler gemeldet wurde.

**`PG_DIAG_SOURCE_FUNCTION`**

Der Name der Quellcodefunktion, die den Fehler gemeldet hat.

> **Hinweis**
>
> Die Felder für Schemaname, Tabellenname, Spaltenname, Datentypname und Constraint-Name werden nur für eine begrenzte Anzahl von Fehlertypen geliefert; siehe Anhang A. Gehen Sie nicht davon aus, dass die Anwesenheit eines dieser Felder die Anwesenheit eines anderen Feldes garantiert. Fehlerquellen im Kernsystem beachten die oben genannten Beziehungen, aber benutzerdefinierte Funktionen können diese Felder auf andere Weise verwenden. Gehen Sie ebenso wenig davon aus, dass diese Felder zeitgenössische Objekte in der aktuellen Datenbank bezeichnen.

Der Client ist dafür verantwortlich, angezeigte Informationen nach Bedarf zu formatieren; insbesondere sollte er lange Zeilen bei Bedarf umbrechen. Zeilenumbruchzeichen in Fehlermeldungsfeldern sollten als Absatzumbrüche behandelt werden, nicht als einfache Zeilenumbrüche.

Von `libpq` intern erzeugte Fehler haben eine Schwere und eine primäre Meldung, aber normalerweise keine weiteren Felder.

Beachten Sie, dass Fehlerfelder nur aus `PGresult`-Objekten verfügbar sind, nicht aus `PGconn`-Objekten; es gibt keine Funktion `PQerrorField`.

**`PQclear`**

Gibt den Speicher frei, der mit einem `PGresult` verbunden ist. Jedes Befehlsergebnis sollte mit `PQclear` freigegeben werden, sobald es nicht mehr benötigt wird.

```c
void PQclear(PGresult *res);
```

Wenn das Argument ein `NULL`-Zeiger ist, wird keine Operation ausgeführt.

Ein `PGresult`-Objekt kann so lange behalten werden, wie es benötigt wird. Es verschwindet nicht, wenn Sie einen neuen Befehl ausgeben, und auch nicht, wenn Sie die Verbindung schließen. Um es loszuwerden, müssen Sie `PQclear` aufrufen. Wird dies versäumt, entstehen Speicherlecks in der Anwendung.

### 32.3.2. Informationen aus Abfrageergebnissen abrufen

Diese Funktionen werden verwendet, um Informationen aus einem `PGresult`-Objekt zu extrahieren, das ein erfolgreiches Abfrageergebnis darstellt, also einen Status `PGRES_TUPLES_OK`, `PGRES_SINGLE_TUPLE` oder `PGRES_TUPLES_CHUNK` hat. Sie können auch verwendet werden, um Informationen aus einer erfolgreichen Describe-Operation zu extrahieren: Ein Describe-Ergebnis enthält dieselben Spalteninformationen, die eine tatsächliche Ausführung der Abfrage liefern würde, hat aber null Zeilen. Für Objekte mit anderen Statuswerten verhalten sich diese Funktionen so, als hätte das Ergebnis null Zeilen und null Spalten.

**`PQntuples`**

Gibt die Anzahl der Zeilen (Tupel) im Abfrageergebnis zurück. `PGresult`-Objekte sind auf höchstens `INT_MAX` Zeilen begrenzt, daher genügt ein `int`-Ergebnis.

```c
int PQntuples(const PGresult *res);
```

**`PQnfields`**

Gibt die Anzahl der Spalten (Felder) in jeder Zeile des Abfrageergebnisses zurück.

```c
int PQnfields(const PGresult *res);
```

**`PQfname`**

Gibt den Spaltennamen zurück, der mit der angegebenen Spaltennummer verbunden ist. Spaltennummern beginnen bei 0. Der Aufrufer sollte das Ergebnis nicht direkt freigeben. Es wird freigegeben, wenn das zugehörige `PGresult`-Handle an `PQclear` übergeben wird.

```c
char *PQfname(const PGresult *res, int column_number);
```

`NULL` wird zurückgegeben, wenn die Spaltennummer außerhalb des gültigen Bereichs liegt.

**`PQfnumber`**

Gibt die Spaltennummer zurück, die mit dem angegebenen Spaltennamen verbunden ist.

```c
int PQfnumber(const PGresult *res, const char *column_name);
```

`-1` wird zurückgegeben, wenn der angegebene Name zu keiner Spalte passt.

Der angegebene Name wird wie ein Bezeichner in einem SQL-Befehl behandelt, das heißt, er wird in Kleinbuchstaben umgewandelt, sofern er nicht in doppelte Anführungszeichen gesetzt ist. Bei einem Abfrageergebnis aus dem SQL-Befehl:

```sql
SELECT 1 AS FOO, 2 AS "BAR";
```

ergäben sich zum Beispiel diese Resultate:

```text
PQfname(res, 0)                              foo
PQfname(res, 1)                              BAR
PQfnumber(res, "FOO")                        0
PQfnumber(res, "foo")                        0
PQfnumber(res, "BAR")                        -1
PQfnumber(res, "\"BAR\"")                    1
```

**`PQftable`**

Gibt die OID der Tabelle zurück, aus der die angegebene Spalte geholt wurde. Spaltennummern beginnen bei 0.

```c
Oid PQftable(const PGresult *res, int column_number);
```

`InvalidOid` wird zurückgegeben, wenn die Spaltennummer außerhalb des gültigen Bereichs liegt oder die angegebene Spalte kein einfacher Verweis auf eine Tabellenspalte ist. Die Systemtabelle `pg_class` kann abgefragt werden, um genau zu bestimmen, auf welche Tabelle verwiesen wird.

Der Typ `Oid` und die Konstante `InvalidOid` sind definiert, wenn die `libpq`-Headerdatei eingebunden wird. Beide sind irgendein Ganzzahltyp.

**`PQftablecol`**

Gibt die Spaltennummer innerhalb ihrer Tabelle für die Spalte zurück, aus der die angegebene Abfrageergebnisspalte besteht. Spaltennummern im Abfrageergebnis beginnen bei 0, Tabellenspalten haben jedoch von null verschiedene Nummern.

```c
int PQftablecol(const PGresult *res, int column_number);
```

Null wird zurückgegeben, wenn die Spaltennummer außerhalb des gültigen Bereichs liegt oder die angegebene Spalte kein einfacher Verweis auf eine Tabellenspalte ist.

**`PQfformat`**

Gibt den Formatcode zurück, der das Format der angegebenen Spalte angibt. Spaltennummern beginnen bei 0.

```c
int PQfformat(const PGresult *res, int column_number);
```

Formatcode null bedeutet textuelle Datendarstellung, Formatcode eins bedeutet Binärdarstellung. Andere Codes sind für zukünftige Definitionen reserviert.

**`PQftype`**

Gibt den Datentyp zurück, der mit der angegebenen Spaltennummer verbunden ist. Die zurückgegebene Ganzzahl ist die interne OID-Nummer des Typs. Spaltennummern beginnen bei 0.

```c
Oid PQftype(const PGresult *res, int column_number);
```

Die Systemtabelle `pg_type` kann abgefragt werden, um Namen und Eigenschaften der verschiedenen Datentypen zu erhalten. Die OIDs der eingebauten Datentypen sind in der Datei `catalog/pg_type_d.h` im Include-Verzeichnis der PostgreSQL-Installation definiert.

**`PQfmod`**

Gibt den Typmodifikator der Spalte zurück, die mit der angegebenen Spaltennummer verbunden ist. Spaltennummern beginnen bei 0.

```c
int PQfmod(const PGresult *res, int column_number);
```

Die Interpretation von Modifikatorwerten ist typspezifisch; typischerweise geben sie Genauigkeit oder Größenbeschränkungen an. Der Wert `-1` bedeutet, dass keine Informationen verfügbar sind. Die meisten Datentypen verwenden keine Modifikatoren; in diesem Fall ist der Wert immer `-1`.

**`PQfsize`**

Gibt die Größe der mit der angegebenen Spaltennummer verbundenen Spalte in Bytes zurück. Spaltennummern beginnen bei 0.

```c
int PQfsize(const PGresult *res, int column_number);
```

`PQfsize` gibt den Platz zurück, der für diese Spalte in einer Datenbankzeile reserviert ist, also die Größe der internen Serverdarstellung des Datentyps. Für Clients ist dies daher nicht besonders nützlich. Ein negativer Wert zeigt an, dass der Datentyp eine variable Länge hat.

**`PQbinaryTuples`**

Gibt `1` zurück, wenn das `PGresult` Binärdaten enthält, und `0`, wenn es Textdaten enthält.

```c
int PQbinaryTuples(const PGresult *res);
```

Diese Funktion ist veraltet, außer bei Verwendung im Zusammenhang mit `COPY`, weil ein einzelnes `PGresult` Textdaten in einigen Spalten und Binärdaten in anderen enthalten kann. `PQfformat` wird bevorzugt. `PQbinaryTuples` gibt nur dann `1` zurück, wenn alle Spalten des Ergebnisses binär sind (Format 1).

**`PQgetvalue`**

Gibt einen einzelnen Feldwert einer Zeile eines `PGresult` zurück. Zeilen- und Spaltennummern beginnen bei 0. Der Aufrufer sollte das Ergebnis nicht direkt freigeben. Es wird freigegeben, wenn das zugehörige `PGresult`-Handle an `PQclear` übergeben wird.

```c
char *PQgetvalue(const PGresult *res,
                 int row_number,
                 int column_number);
```

Für Daten im Textformat ist der von `PQgetvalue` zurückgegebene Wert eine nullterminierte Zeichenkettendarstellung des Feldwerts. Für Daten im Binärformat liegt der Wert in der binären Darstellung vor, die durch die `typsend`- und `typreceive`-Funktionen des Datentyps bestimmt wird. Auch in diesem Fall folgt dem Wert tatsächlich ein Nullbyte, das aber gewöhnlich nicht nützlich ist, weil der Wert eingebettete Nullen enthalten kann.

Eine leere Zeichenkette wird zurückgegeben, wenn der Feldwert null ist. Verwenden Sie `PQgetisnull`, um Nullwerte von leeren Zeichenkettenwerten zu unterscheiden.

Der von `PQgetvalue` zurückgegebene Zeiger zeigt auf Speicher, der Teil der `PGresult`-Struktur ist. Die Daten, auf die er zeigt, sollten nicht verändert werden, und sie müssen ausdrücklich in anderen Speicher kopiert werden, wenn sie über die Lebensdauer der `PGresult`-Struktur hinaus verwendet werden sollen.

**`PQgetisnull`**

Prüft ein Feld auf einen Nullwert. Zeilen- und Spaltennummern beginnen bei 0.

```c
int PQgetisnull(const PGresult *res,
                int row_number,
                int column_number);
```

Diese Funktion gibt `1` zurück, wenn das Feld null ist, und `0`, wenn es einen Nicht-Null-Wert enthält. Beachten Sie, dass `PQgetvalue` für ein Nullfeld eine leere Zeichenkette zurückgibt, keinen Nullzeiger.

**`PQgetlength`**

Gibt die tatsächliche Länge eines Feldwerts in Bytes zurück. Zeilen- und Spaltennummern beginnen bei 0.

```c
int PQgetlength(const PGresult *res,
                int row_number,
                int column_number);
```

Dies ist die tatsächliche Datenlänge für den jeweiligen Datenwert, also die Größe des Objekts, auf das `PQgetvalue` zeigt. Für Textdaten ist dies dasselbe wie `strlen()`. Für das Binärformat ist dies eine wesentliche Information. Man sollte sich nicht auf `PQfsize` verlassen, um die tatsächliche Datenlänge zu erhalten.

**`PQnparams`**

Gibt die Anzahl der Parameter einer vorbereiteten Anweisung zurück.

```c
int PQnparams(const PGresult *res);
```

Diese Funktion ist nur nützlich, wenn das Ergebnis von `PQdescribePrepared` untersucht wird. Für andere Ergebnistypen gibt sie null zurück.

**`PQparamtype`**

Gibt den Datentyp des angegebenen Anweisungsparameters zurück. Parameternummern beginnen bei 0.

```c
Oid PQparamtype(const PGresult *res, int param_number);
```

Diese Funktion ist nur nützlich, wenn das Ergebnis von `PQdescribePrepared` untersucht wird. Für andere Ergebnistypen gibt sie null zurück.

**`PQprint`**

Gibt alle Zeilen und optional die Spaltennamen auf den angegebenen Ausgabestrom aus.

```c
void PQprint(FILE *fout, const PGresult *res, const PQprintOpt *po);

typedef struct
{
    pqbool header;       /* print output field headings and row count */
    pqbool align;        /* fill align the fields */
    pqbool standard;     /* old brain dead format */
    pqbool html3;        /* output HTML tables */
    pqbool expanded;     /* expand tables */
    pqbool pager;        /* use pager for output if needed */
    char   *fieldSep;    /* field separator */
    char   *tableOpt;    /* attributes for HTML table element */
    char   *caption;     /* HTML table caption */
    char   **fieldName;  /* null-terminated array of replacement field names */
} PQprintOpt;
```

Diese Funktion wurde früher von `psql` verwendet, um Abfrageergebnisse auszugeben, ist dort aber nicht mehr im Einsatz. Beachten Sie, dass sie annimmt, dass alle Daten im Textformat vorliegen.

### 32.3.3. Weitere Ergebnisinformationen abrufen

Diese Funktionen werden verwendet, um weitere Informationen aus `PGresult`-Objekten zu extrahieren.

**`PQcmdStatus`**

Gibt das Befehlsstatustag des SQL-Befehls zurück, der das `PGresult` erzeugt hat.

```c
char *PQcmdStatus(PGresult *res);
```

Häufig ist dies nur der Name des Befehls, es kann aber zusätzliche Daten enthalten, etwa die Anzahl der verarbeiteten Zeilen. Der Aufrufer sollte das Ergebnis nicht direkt freigeben. Es wird freigegeben, wenn das zugehörige `PGresult`-Handle an `PQclear` übergeben wird.

**`PQcmdTuples`**

Gibt die Anzahl der Zeilen zurück, die vom SQL-Befehl betroffen waren.

```c
char *PQcmdTuples(PGresult *res);
```

Diese Funktion gibt eine Zeichenkette zurück, die die Anzahl der Zeilen enthält, die von der SQL-Anweisung betroffen waren, welche das `PGresult` erzeugt hat. Sie kann nur nach der Ausführung einer `SELECT`-, `CREATE TABLE AS`-, `INSERT`-, `UPDATE`-, `DELETE`-, `MERGE`-, `MOVE`-, `FETCH`- oder `COPY`-Anweisung verwendet werden, oder nach einem `EXECUTE` einer vorbereiteten Abfrage, die eine `INSERT`-, `UPDATE`-, `DELETE`- oder `MERGE`-Anweisung enthält. Wenn der Befehl, der das `PGresult` erzeugt hat, etwas anderes war, gibt `PQcmdTuples` eine leere Zeichenkette zurück. Der Aufrufer sollte den Rückgabewert nicht direkt freigeben. Er wird freigegeben, wenn das zugehörige `PGresult`-Handle an `PQclear` übergeben wird.

**`PQoidValue`**

Gibt die OID der eingefügten Zeile zurück, wenn der SQL-Befehl ein `INSERT` war, das genau eine Zeile in eine Tabelle mit OIDs eingefügt hat, oder ein `EXECUTE` einer vorbereiteten Abfrage mit einer geeigneten `INSERT`-Anweisung. Andernfalls gibt diese Funktion `InvalidOid` zurück. Sie gibt auch dann `InvalidOid` zurück, wenn die vom `INSERT` betroffene Tabelle keine OIDs enthält.

```c
Oid PQoidValue(const PGresult *res);
```

**`PQoidStatus`**

Diese Funktion ist zugunsten von `PQoidValue` veraltet und nicht thread-sicher. Sie gibt eine Zeichenkette mit der OID der eingefügten Zeile zurück, während `PQoidValue` den OID-Wert zurückgibt.

```c
char *PQoidStatus(const PGresult *res);
```

### 32.3.4. Zeichenketten für SQL-Befehle escapen

**`PQescapeLiteral`**

```c
char *PQescapeLiteral(PGconn *conn, const char *str, size_t length);
```

`PQescapeLiteral` escapet eine Zeichenkette zur Verwendung innerhalb eines SQL-Befehls. Dies ist nützlich, wenn Datenwerte als literale Konstanten in SQL-Befehle eingefügt werden. Bestimmte Zeichen, etwa Anführungszeichen und Backslashes, müssen escaped werden, damit sie vom SQL-Parser nicht speziell interpretiert werden. `PQescapeLiteral` führt diese Operation aus.

`PQescapeLiteral` gibt eine escaped Version des Parameters `str` in mit `malloc()` alloziertem Speicher zurück. Dieser Speicher sollte mit `PQfreemem()` freigegeben werden, wenn das Ergebnis nicht mehr benötigt wird. Ein abschließendes Nullbyte ist nicht erforderlich und sollte in `length` nicht mitgezählt werden. Wenn vor der Verarbeitung von `length` Bytes ein abschließendes Nullbyte gefunden wird, hält `PQescapeLiteral` dort an; das Verhalten ähnelt damit `strncpy`. In der Rückgabezeichenkette sind alle Sonderzeichen ersetzt, sodass sie vom PostgreSQL-Parser für Zeichenkettenliterale korrekt verarbeitet werden können. Ein abschließendes Nullbyte wird ebenfalls hinzugefügt. Die einfachen Anführungszeichen, die PostgreSQL-Zeichenkettenliterale umschließen müssen, sind in der Ergebniszeichenkette enthalten.

Bei einem Fehler gibt `PQescapeLiteral` `NULL` zurück, und eine passende Meldung wird im Objekt `conn` gespeichert.

> **Tipp**
>
> Korrektes Escaping ist besonders wichtig, wenn Zeichenketten verarbeitet werden, die aus einer nicht vertrauenswürdigen Quelle stammen. Andernfalls besteht ein Sicherheitsrisiko: Die Anwendung ist anfällig für SQL-Injection-Angriffe, bei denen unerwünschte SQL-Befehle an die Datenbank übergeben werden.

Beachten Sie, dass Escaping weder nötig noch korrekt ist, wenn ein Datenwert als separater Parameter an `PQexecParams` oder verwandte Routinen übergeben wird.

**`PQescapeIdentifier`**

```c
char *PQescapeIdentifier(PGconn *conn, const char *str, size_t length);
```

`PQescapeIdentifier` escapet eine Zeichenkette zur Verwendung als SQL-Bezeichner, etwa als Tabellen-, Spalten- oder Funktionsname. Dies ist nützlich, wenn ein vom Benutzer gelieferter Bezeichner Sonderzeichen enthalten könnte, die der SQL-Parser sonst nicht als Teil des Bezeichners interpretieren würde, oder wenn der Bezeichner Großbuchstaben enthält, deren Schreibweise erhalten bleiben soll.

`PQescapeIdentifier` gibt eine als SQL-Bezeichner escapete Version des Parameters `str` in mit `malloc()` alloziertem Speicher zurück. Dieser Speicher muss mit `PQfreemem()` freigegeben werden, wenn das Ergebnis nicht mehr benötigt wird. Ein abschließendes Nullbyte ist nicht erforderlich und sollte in `length` nicht mitgezählt werden. Wenn vor der Verarbeitung von `length` Bytes ein abschließendes Nullbyte gefunden wird, hält `PQescapeIdentifier` dort an; das Verhalten ähnelt damit `strncpy`. In der Rückgabezeichenkette sind alle Sonderzeichen ersetzt, sodass sie als SQL-Bezeichner korrekt verarbeitet wird. Ein abschließendes Nullbyte wird ebenfalls hinzugefügt. Die Rückgabezeichenkette wird außerdem in doppelte Anführungszeichen gesetzt.

Bei einem Fehler gibt `PQescapeIdentifier` `NULL` zurück, und eine passende Meldung wird im Objekt `conn` gespeichert.

> **Tipp**
>
> Wie bei Zeichenkettenliteralen müssen SQL-Bezeichner escaped werden, wenn sie aus einer nicht vertrauenswürdigen Quelle stammen, um SQL-Injection-Angriffe zu verhindern.

**`PQescapeStringConn`**

```c
size_t PQescapeStringConn(PGconn *conn,
                          char *to,
                          const char *from,
                          size_t length,
                          int *error);
```

`PQescapeStringConn` escapet Zeichenkettenliterale, ähnlich wie `PQescapeLiteral`. Anders als bei `PQescapeLiteral` ist der Aufrufer dafür verantwortlich, einen ausreichend großen Puffer bereitzustellen. Außerdem erzeugt `PQescapeStringConn` nicht die einfachen Anführungszeichen, die PostgreSQL-Zeichenkettenliterale umschließen müssen; sie sollten im SQL-Befehl bereitgestellt werden, in den das Ergebnis eingefügt wird. Der Parameter `from` zeigt auf das erste Zeichen der zu escapenden Zeichenkette, und `length` gibt die Anzahl der Bytes in dieser Zeichenkette an. Ein abschließendes Nullbyte ist nicht erforderlich und sollte in `length` nicht mitgezählt werden. Wenn vor der Verarbeitung von `length` Bytes ein abschließendes Nullbyte gefunden wird, hält `PQescapeStringConn` dort an; das Verhalten ähnelt damit `strncpy`. `to` muss auf einen Puffer zeigen, der mindestens ein Byte mehr als das Doppelte von `length` aufnehmen kann; andernfalls ist das Verhalten undefiniert. Ebenso undefiniert ist das Verhalten, wenn sich die Zeichenketten `to` und `from` überlappen.

Wenn der Parameter `error` nicht `NULL` ist, wird `*error` bei Erfolg auf null und bei einem Fehler auf einen von null verschiedenen Wert gesetzt. Derzeit betreffen die einzigen möglichen Fehlerbedingungen ungültige Multibyte-Kodierung in der Quellzeichenkette. Die Ausgabezeichenkette wird auch bei einem Fehler erzeugt, aber man kann erwarten, dass der Server sie als fehlerhaft ablehnt. Bei einem Fehler wird eine passende Meldung im Objekt `conn` gespeichert, unabhängig davon, ob `error` `NULL` ist.

`PQescapeStringConn` gibt die Anzahl der in `to` geschriebenen Bytes zurück, ohne das abschließende Nullbyte.

**`PQescapeString`**

`PQescapeString` ist eine ältere, veraltete Version von `PQescapeStringConn`.

```c
size_t PQescapeString(char *to, const char *from, size_t length);
```

Der einzige Unterschied zu `PQescapeStringConn` besteht darin, dass `PQescapeString` keine Parameter `PGconn` oder `error` entgegennimmt. Deshalb kann die Funktion ihr Verhalten nicht an Verbindungseigenschaften wie die Zeichenkodierung anpassen und kann daher falsche Ergebnisse liefern. Außerdem kann sie Fehlerbedingungen nicht melden.

`PQescapeString` kann sicher in Clientprogrammen verwendet werden, die jeweils nur mit einer PostgreSQL-Verbindung arbeiten; in diesem Fall kann die Funktion das Nötige hinter den Kulissen herausfinden. In anderen Kontexten ist sie ein Sicherheitsrisiko und sollte zugunsten von `PQescapeStringConn` vermieden werden.

**`PQescapeByteaConn`**

Escapet Binärdaten zur Verwendung innerhalb eines SQL-Befehls mit dem Typ `bytea`. Wie `PQescapeStringConn` wird dies nur verwendet, wenn Daten direkt in eine SQL-Befehlszeichenkette eingefügt werden.

```c
unsigned char *PQescapeByteaConn(PGconn *conn,
                                 const unsigned char *from,
                                 size_t from_length,
                                 size_t *to_length);
```

Bestimmte Bytewerte müssen escaped werden, wenn sie als Teil eines `bytea`-Literals in einer SQL-Anweisung verwendet werden. `PQescapeByteaConn` escapet Bytes entweder mit Hex-Kodierung oder Backslash-Escaping. Weitere Informationen stehen in [Abschnitt 8.4](08_Datentypen.md#84-binärdatentypen).

Der Parameter `from` zeigt auf das erste Byte der zu escapenden Zeichenkette, und `from_length` gibt die Anzahl der Bytes in dieser Binärzeichenkette an. Ein abschließendes Nullbyte ist weder nötig noch wird es gezählt. Der Parameter `to_length` zeigt auf eine Variable, die die Länge der resultierenden escaped Zeichenkette aufnehmen wird. Diese Länge enthält das abschließende Nullbyte des Ergebnisses.

`PQescapeByteaConn` gibt eine escaped Version der binären Zeichenkette `from` in mit `malloc()` alloziertem Speicher zurück. Dieser Speicher sollte mit `PQfreemem()` freigegeben werden, wenn das Ergebnis nicht mehr benötigt wird. In der Rückgabezeichenkette sind alle Sonderzeichen ersetzt, sodass sie vom PostgreSQL-Parser für Zeichenkettenliterale und von der Eingabefunktion für `bytea` korrekt verarbeitet werden können. Ein abschließendes Nullbyte wird ebenfalls hinzugefügt. Die einfachen Anführungszeichen, die PostgreSQL-Zeichenkettenliterale umschließen müssen, sind nicht Teil der Ergebniszeichenkette.

Bei einem Fehler wird ein Nullzeiger zurückgegeben, und eine passende Fehlermeldung wird im Objekt `conn` gespeichert. Derzeit ist der einzige mögliche Fehler unzureichender Speicher für die Ergebniszeichenkette.

**`PQescapeBytea`**

`PQescapeBytea` ist eine ältere, veraltete Version von `PQescapeByteaConn`.

```c
unsigned char *PQescapeBytea(const unsigned char *from,
                             size_t from_length,
                             size_t *to_length);
```

Der einzige Unterschied zu `PQescapeByteaConn` besteht darin, dass `PQescapeBytea` keinen Parameter `PGconn` entgegennimmt. Deshalb kann `PQescapeBytea` nur in Clientprogrammen sicher verwendet werden, die jeweils nur eine PostgreSQL-Verbindung verwenden; in diesem Fall kann die Funktion das Nötige hinter den Kulissen herausfinden. In Programmen mit mehreren Datenbankverbindungen kann sie falsche Ergebnisse liefern; verwenden Sie in solchen Fällen `PQescapeByteaConn`.

**`PQunescapeBytea`**

Wandelt eine Zeichenkettendarstellung von Binärdaten in Binärdaten um, also die Umkehrung von `PQescapeBytea`. Dies wird benötigt, wenn `bytea`-Daten im Textformat abgerufen werden, nicht aber beim Abruf im Binärformat.

```c
unsigned char *PQunescapeBytea(const unsigned char *from, size_t *to_length);
```

Der Parameter `from` zeigt auf eine Zeichenkette, wie sie `PQgetvalue` bei Anwendung auf eine `bytea`-Spalte zurückgeben könnte. `PQunescapeBytea` wandelt diese Zeichenkettendarstellung in ihre Binärdarstellung um. Die Funktion gibt einen Zeiger auf einen mit `malloc()` allozierten Puffer zurück, oder `NULL` bei einem Fehler, und legt die Größe des Puffers in `to_length` ab. Das Ergebnis muss mit `PQfreemem` freigegeben werden, wenn es nicht mehr benötigt wird.

Diese Umwandlung ist nicht exakt die Umkehrung von `PQescapeBytea`, weil die Zeichenkette, die von `PQgetvalue` empfangen wird, nicht als „escaped“ erwartet wird. Insbesondere bedeutet dies, dass keine Überlegungen zum Quotieren von Zeichenketten nötig sind und daher auch kein `PGconn`-Parameter.

## 32.4. Asynchrone Befehlsverarbeitung

Die Funktion `PQexec` ist ausreichend, um Befehle in normalen synchronen Anwendungen zu senden. Sie hat jedoch einige Einschränkungen, die für manche Benutzer wichtig sein können:

- `PQexec` wartet, bis der Befehl abgeschlossen ist. Die Anwendung hat möglicherweise andere Aufgaben zu erledigen, etwa eine Benutzerschnittstelle zu pflegen, und möchte dann nicht blockierend auf die Antwort warten.
- Da die Ausführung der Clientanwendung angehalten wird, während sie auf das Ergebnis wartet, ist es für die Anwendung schwierig zu entscheiden, dass sie versuchen möchte, den laufenden Befehl abzubrechen. Dies ist zwar aus einem Signal-Handler möglich, aber sonst nicht.
- `PQexec` kann nur eine `PGresult`-Struktur zurückgeben. Wenn die übergebene Befehlszeichenkette mehrere SQL-Befehle enthält, verwirft `PQexec` alle `PGresult`-Objekte außer dem letzten.
- `PQexec` sammelt immer das gesamte Ergebnis des Befehls und puffert es in einem einzigen `PGresult`. Dies vereinfacht zwar die Fehlerbehandlung der Anwendung, kann aber bei Ergebnissen mit vielen Zeilen unpraktisch sein.

Anwendungen, denen diese Einschränkungen nicht passen, können stattdessen die zugrunde liegenden Funktionen verwenden, auf denen `PQexec` aufbaut: `PQsendQuery` und `PQgetResult`. Außerdem gibt es `PQsendQueryParams`, `PQsendPrepare`, `PQsendQueryPrepared`, `PQsendDescribePrepared`, `PQsendDescribePortal`, `PQsendClosePrepared` und `PQsendClosePortal`, die zusammen mit `PQgetResult` verwendet werden können, um die Funktionalität von `PQexecParams`, `PQprepare`, `PQexecPrepared`, `PQdescribePrepared`, `PQdescribePortal`, `PQclosePrepared` beziehungsweise `PQclosePortal` nachzubilden.

**`PQsendQuery`**

Sendet einen Befehl an den Server, ohne auf das Ergebnis oder die Ergebnisse zu warten. `1` wird zurückgegeben, wenn der Befehl erfolgreich abgesetzt wurde, andernfalls `0`; in diesem Fall liefert `PQerrorMessage` weitere Informationen zum Fehlschlag.

```c
int PQsendQuery(PGconn *conn, const char *command);
```

Nach einem erfolgreichen Aufruf von `PQsendQuery` muss `PQgetResult` ein- oder mehrmals aufgerufen werden, um die Ergebnisse zu erhalten. `PQsendQuery` kann auf derselben Verbindung erst wieder aufgerufen werden, wenn `PQgetResult` einen Nullzeiger zurückgegeben hat, der anzeigt, dass der Befehl abgeschlossen ist.

Im Pipeline-Modus ist diese Funktion nicht erlaubt.

**`PQsendQueryParams`**

Sendet einen Befehl und separate Parameter an den Server, ohne auf das Ergebnis oder die Ergebnisse zu warten.

```c
int PQsendQueryParams(PGconn *conn,
                      const char *command,
                      int nParams,
                      const Oid *paramTypes,
                      const char * const *paramValues,
                      const int *paramLengths,
                      const int *paramFormats,
                      int resultFormat);
```

Dies entspricht `PQsendQuery`, außer dass Abfrageparameter getrennt von der Abfragezeichenkette angegeben werden können. Die Parameter der Funktion werden genauso behandelt wie bei `PQexecParams`. Wie `PQexecParams` erlaubt sie nur einen Befehl in der Abfragezeichenkette.

**`PQsendPrepare`**

Sendet eine Anforderung zum Erstellen einer vorbereiteten Anweisung mit den angegebenen Parametern, ohne auf den Abschluss zu warten.

```c
int PQsendPrepare(PGconn *conn,
                  const char *stmtName,
                  const char *query,
                  int nParams,
                  const Oid *paramTypes);
```

Dies ist eine asynchrone Version von `PQprepare`: Sie gibt `1` zurück, wenn sie die Anforderung absetzen konnte, andernfalls `0`. Nach einem erfolgreichen Aufruf muss `PQgetResult` aufgerufen werden, um festzustellen, ob der Server die vorbereitete Anweisung erfolgreich erstellt hat. Die Funktionsparameter werden genauso behandelt wie bei `PQprepare`.

**`PQsendQueryPrepared`**

Sendet eine Anforderung zum Ausführen einer vorbereiteten Anweisung mit angegebenen Parametern, ohne auf das Ergebnis oder die Ergebnisse zu warten.

```c
int PQsendQueryPrepared(PGconn *conn,
                        const char *stmtName,
                        int nParams,
                        const char * const *paramValues,
                        const int *paramLengths,
                        const int *paramFormats,
                        int resultFormat);
```

Dies ähnelt `PQsendQueryParams`, aber der auszuführende Befehl wird durch Benennung einer zuvor vorbereiteten Anweisung angegeben, nicht durch eine Abfragezeichenkette. Die Funktionsparameter werden genauso behandelt wie bei `PQexecPrepared`.

**`PQsendDescribePrepared`**

Sendet eine Anforderung, Informationen über die angegebene vorbereitete Anweisung zu erhalten, ohne auf den Abschluss zu warten.

```c
int PQsendDescribePrepared(PGconn *conn, const char *stmtName);
```

Dies ist eine asynchrone Version von `PQdescribePrepared`: Sie gibt `1` zurück, wenn sie die Anforderung absetzen konnte, andernfalls `0`. Nach einem erfolgreichen Aufruf muss `PQgetResult` aufgerufen werden, um die Ergebnisse zu erhalten. Die Funktionsparameter werden genauso behandelt wie bei `PQdescribePrepared`.

**`PQsendDescribePortal`**

Sendet eine Anforderung, Informationen über das angegebene Portal zu erhalten, ohne auf den Abschluss zu warten.

```c
int PQsendDescribePortal(PGconn *conn, const char *portalName);
```

Dies ist eine asynchrone Version von `PQdescribePortal`: Sie gibt `1` zurück, wenn sie die Anforderung absetzen konnte, andernfalls `0`. Nach einem erfolgreichen Aufruf muss `PQgetResult` aufgerufen werden, um die Ergebnisse zu erhalten. Die Funktionsparameter werden genauso behandelt wie bei `PQdescribePortal`.

**`PQsendClosePrepared`**

Sendet eine Anforderung zum Schließen der angegebenen vorbereiteten Anweisung, ohne auf den Abschluss zu warten.

```c
int PQsendClosePrepared(PGconn *conn, const char *stmtName);
```

Dies ist eine asynchrone Version von `PQclosePrepared`: Sie gibt `1` zurück, wenn sie die Anforderung absetzen konnte, andernfalls `0`. Nach einem erfolgreichen Aufruf muss `PQgetResult` aufgerufen werden, um die Ergebnisse zu erhalten. Die Funktionsparameter werden genauso behandelt wie bei `PQclosePrepared`.

**`PQsendClosePortal`**

Sendet eine Anforderung zum Schließen des angegebenen Portals, ohne auf den Abschluss zu warten.

```c
int PQsendClosePortal(PGconn *conn, const char *portalName);
```

Dies ist eine asynchrone Version von `PQclosePortal`: Sie gibt `1` zurück, wenn sie die Anforderung absetzen konnte, andernfalls `0`. Nach einem erfolgreichen Aufruf muss `PQgetResult` aufgerufen werden, um die Ergebnisse zu erhalten. Die Funktionsparameter werden genauso behandelt wie bei `PQclosePortal`.

**`PQgetResult`**

Wartet auf das nächste Ergebnis eines vorherigen Aufrufs von `PQsendQuery`, `PQsendQueryParams`, `PQsendPrepare`, `PQsendQueryPrepared`, `PQsendDescribePrepared`, `PQsendDescribePortal`, `PQsendClosePrepared`, `PQsendClosePortal`, `PQsendPipelineSync` oder `PQpipelineSync` und gibt es zurück. Ein Nullzeiger wird zurückgegeben, wenn der Befehl abgeschlossen ist und es keine weiteren Ergebnisse gibt.

```c
PGresult *PQgetResult(PGconn *conn);
```

`PQgetResult` muss wiederholt aufgerufen werden, bis es einen Nullzeiger zurückgibt, der anzeigt, dass der Befehl abgeschlossen ist. Wenn es aufgerufen wird, obwohl kein Befehl aktiv ist, gibt `PQgetResult` sofort einen Nullzeiger zurück. Jedes Nicht-Null-Ergebnis von `PQgetResult` sollte mit denselben zuvor beschriebenen `PGresult`-Zugriffsfunktionen verarbeitet werden. Vergessen Sie nicht, jedes Ergebnisobjekt anschließend mit `PQclear` freizugeben. Beachten Sie, dass `PQgetResult` nur dann blockiert, wenn ein Befehl aktiv ist und die benötigten Antwortdaten noch nicht mit `PQconsumeInput` gelesen wurden.

Im Pipeline-Modus gibt `PQgetResult` normalerweise zurück, sofern kein Fehler auftritt. Für jede nach der fehlerauslösenden Abfrage gesendete weitere Abfrage bis zum nächsten Synchronisationspunkt, diesen ausgenommen, wird ein spezielles Ergebnis des Typs `PGRES_PIPELINE_ABORTED` zurückgegeben, danach ein Nullzeiger. Wenn der Pipeline-Synchronisationspunkt erreicht wird, wird ein Ergebnis des Typs `PGRES_PIPELINE_SYNC` zurückgegeben. Das Ergebnis der nächsten Abfrage nach dem Synchronisationspunkt folgt unmittelbar, das heißt, nach dem Synchronisationspunkt wird kein Nullzeiger zurückgegeben.

> **Hinweis**
>
> Auch wenn `PQresultStatus` einen schwerwiegenden Fehler anzeigt, sollte `PQgetResult` aufgerufen werden, bis es einen Nullzeiger zurückgibt, damit `libpq` die Fehlerinformationen vollständig verarbeiten kann.

Die Verwendung von `PQsendQuery` und `PQgetResult` löst eines der Probleme von `PQexec`: Wenn eine Befehlszeichenkette mehrere SQL-Befehle enthält, können die Ergebnisse dieser Befehle einzeln abgerufen werden. Nebenbei ermöglicht dies eine einfache Form überlappender Verarbeitung: Der Client kann die Ergebnisse eines Befehls bearbeiten, während der Server noch an späteren Abfragen derselben Befehlszeichenkette arbeitet.

Eine weitere häufig gewünschte Fähigkeit, die mit `PQsendQuery` und `PQgetResult` erreicht werden kann, ist das Abrufen großer Abfrageergebnisse mit einer begrenzten Anzahl von Zeilen auf einmal. Dies wird in [Abschnitt 32.6](32_libpq_C_Bibliothek.md#326-abfrageergebnisse-in-chunks-abrufen) besprochen.

Für sich allein führt ein Aufruf von `PQgetResult` weiterhin dazu, dass der Client blockiert, bis der Server den nächsten SQL-Befehl abgeschlossen hat. Dies kann durch korrekte Verwendung zweier weiterer Funktionen vermieden werden:

**`PQconsumeInput`**

Wenn Eingaben vom Server verfügbar sind, werden sie verbraucht.

```c
int PQconsumeInput(PGconn *conn);
```

`PQconsumeInput` gibt normalerweise `1` zurück, was „kein Fehler“ bedeutet, aber `0`, wenn irgendein Problem auftrat; in diesem Fall kann `PQerrorMessage` abgefragt werden. Beachten Sie, dass das Ergebnis nicht aussagt, ob tatsächlich Eingabedaten gesammelt wurden. Nach einem Aufruf von `PQconsumeInput` kann die Anwendung `PQisBusy` und/oder `PQnotifies` prüfen, um zu sehen, ob sich deren Zustand geändert hat.

`PQconsumeInput` kann auch dann aufgerufen werden, wenn die Anwendung noch nicht bereit ist, ein Ergebnis oder eine Benachrichtigung zu verarbeiten. Die Funktion liest verfügbare Daten und speichert sie in einem Puffer, wodurch eine `select()`-Anzeige für Lesebereitschaft verschwindet. Die Anwendung kann also `PQconsumeInput` verwenden, um die `select()`-Bedingung sofort zu löschen, und die Ergebnisse anschließend in Ruhe untersuchen.

**`PQisBusy`**

Gibt `1` zurück, wenn ein Befehl beschäftigt ist, das heißt, wenn `PQgetResult` beim Warten auf Eingaben blockieren würde. Ein Rückgabewert `0` bedeutet, dass `PQgetResult` mit der Gewissheit aufgerufen werden kann, nicht zu blockieren.

```c
int PQisBusy(PGconn *conn);
```

`PQisBusy` versucht nicht selbst, Daten vom Server zu lesen; daher muss zuerst `PQconsumeInput` aufgerufen werden, sonst endet der Busy-Zustand nie.

Eine typische Anwendung, die diese Funktionen nutzt, hat eine Hauptschleife, die `select()` oder `poll()` verwendet, um auf alle Bedingungen zu warten, auf die sie reagieren muss. Eine dieser Bedingungen ist verfügbare Eingabe vom Server; in Begriffen von `select()` bedeutet dies lesbare Daten auf dem Dateideskriptor, den `PQsocket` liefert. Wenn die Hauptschleife Eingabebereitschaft erkennt, sollte sie `PQconsumeInput` aufrufen, um die Eingabe zu lesen. Anschließend kann sie `PQisBusy` aufrufen, gefolgt von `PQgetResult`, wenn `PQisBusy` falsch (`0`) zurückgibt. Außerdem kann sie `PQnotifies` aufrufen, um `NOTIFY`-Meldungen zu erkennen (siehe [Abschnitt 32.9](32_libpq_C_Bibliothek.md#329-asynchrone-benachrichtigung)).

Ein Client, der `PQsendQuery`/`PQgetResult` verwendet, kann außerdem versuchen, einen Befehl abzubrechen, der noch vom Server verarbeitet wird; siehe [Abschnitt 32.7](32_libpq_C_Bibliothek.md#327-laufende-abfragen-abbrechen). Unabhängig vom Rückgabewert von `PQcancelBlocking` muss die Anwendung aber mit der normalen Ergebnislesesequenz über `PQgetResult` fortfahren. Ein erfolgreicher Abbruch führt lediglich dazu, dass der Befehl früher beendet wird, als er es sonst getan hätte.

Mit den oben beschriebenen Funktionen ist es möglich, Blockieren beim Warten auf Eingaben vom Datenbankserver zu vermeiden. Es ist jedoch weiterhin möglich, dass die Anwendung blockiert, während sie darauf wartet, Ausgaben an den Server zu senden. Dies ist relativ selten, kann aber passieren, wenn sehr lange SQL-Befehle oder Datenwerte gesendet werden. Es ist allerdings deutlich wahrscheinlicher, wenn die Anwendung Daten über `COPY IN` sendet. Um diese Möglichkeit zu verhindern und vollständig nichtblockierenden Datenbankbetrieb zu erreichen, können die folgenden zusätzlichen Funktionen verwendet werden.

**`PQsetnonblocking`**

Setzt den Nichtblockierungsstatus der Verbindung.

```c
int PQsetnonblocking(PGconn *conn, int arg);
```

Setzt den Zustand der Verbindung auf nichtblockierend, wenn `arg` `1` ist, oder auf blockierend, wenn `arg` `0` ist. Gibt bei Erfolg `0` zurück, bei Fehler `-1`.

Im nichtblockierenden Zustand blockieren erfolgreiche Aufrufe von `PQsendQuery`, `PQputline`, `PQputnbytes`, `PQputCopyData` und `PQendcopy` nicht; ihre Änderungen werden im lokalen Ausgabepuffer gespeichert, bis sie geleert werden. Nicht erfolgreiche Aufrufe geben einen Fehler zurück und müssen erneut versucht werden.

Beachten Sie, dass `PQexec` den nichtblockierenden Modus nicht beachtet; wenn es aufgerufen wird, verhält es sich trotzdem blockierend.

**`PQisnonblocking`**

Gibt den Blockierungsstatus der Datenbankverbindung zurück.

```c
int PQisnonblocking(const PGconn *conn);
```

Gibt `1` zurück, wenn die Verbindung auf nichtblockierenden Modus gesetzt ist, und `0`, wenn sie blockierend ist.

**`PQflush`**

Versucht, alle in der Warteschlange stehenden Ausgabedaten zum Server zu leeren. Gibt `0` zurück, wenn dies erfolgreich war oder die Sendewarteschlange leer ist, `-1`, wenn es aus irgendeinem Grund fehlschlug, oder `1`, wenn noch nicht alle Daten in der Sendewarteschlange gesendet werden konnten. Dieser letzte Fall kann nur auftreten, wenn die Verbindung nichtblockierend ist.

```c
int PQflush(PGconn *conn);
```

Nach dem Senden eines Befehls oder von Daten auf einer nichtblockierenden Verbindung rufen Sie `PQflush` auf. Wenn es `1` zurückgibt, warten Sie, bis der Socket lese- oder schreibbereit wird. Wenn er schreibbereit wird, rufen Sie erneut `PQflush` auf. Wenn er lesebereit wird, rufen Sie `PQconsumeInput` und dann erneut `PQflush` auf. Wiederholen Sie dies, bis `PQflush` `0` zurückgibt. Es ist notwendig, auf Lesebereitschaft zu prüfen und die Eingabe mit `PQconsumeInput` abzuleeren, weil der Server blockieren kann, während er versucht, uns Daten zu senden, etwa `NOTICE`-Meldungen, und unsere Daten nicht liest, bis wir seine gelesen haben. Sobald `PQflush` `0` zurückgibt, warten Sie, bis der Socket lesebereit ist, und lesen dann die Antwort wie oben beschrieben.

## 32.5. Pipeline-Modus

Der Pipeline-Modus von `libpq` erlaubt Anwendungen, eine Abfrage zu senden, ohne zuvor das Ergebnis der vorher gesendeten Abfrage lesen zu müssen. Durch Nutzung des Pipeline-Modus wartet ein Client weniger auf den Server, weil mehrere Abfragen und Ergebnisse in einer einzigen Netzwerktransaktion gesendet und empfangen werden können.

Obwohl der Pipeline-Modus einen deutlichen Leistungsgewinn bringen kann, ist das Schreiben von Clients, die ihn verwenden, komplexer: Die Anwendung muss eine Warteschlange ausstehenden Abfragen verwalten und herausfinden, welches Ergebnis zu welcher Abfrage in der Warteschlange gehört.

Der Pipeline-Modus verbraucht im Allgemeinen auch mehr Speicher auf Client und Server, obwohl eine sorgfältige und aggressive Verwaltung der Sende-/Empfangswarteschlange dies abmildern kann. Dies gilt unabhängig davon, ob die Verbindung im blockierenden oder nichtblockierenden Modus arbeitet.

Die Pipeline-API von `libpq` wurde zwar in PostgreSQL 14 eingeführt, ist aber eine clientseitige Funktion, die keine besondere Serverunterstützung erfordert und mit jedem Server funktioniert, der das erweiterte Abfrageprotokoll v3 unterstützt. Weitere Informationen stehen in Abschnitt 54.2.4.

### 32.5.1. Pipeline-Modus verwenden

Um Pipelines auszugeben, muss die Anwendung die Verbindung in den Pipeline-Modus umschalten; dies geschieht mit `PQenterPipelineMode`. Mit `PQpipelineStatus` kann geprüft werden, ob der Pipeline-Modus aktiv ist. Im Pipeline-Modus sind nur asynchrone Operationen erlaubt, die das erweiterte Abfrageprotokoll verwenden; Befehlszeichenketten mit mehreren SQL-Befehlen sind nicht erlaubt, ebenso wenig `COPY`. Die Verwendung synchroner Befehlsausführungsfunktionen wie `PQfn`, `PQexec`, `PQexecParams`, `PQprepare`, `PQexecPrepared`, `PQdescribePrepared`, `PQdescribePortal`, `PQclosePrepared` oder `PQclosePortal` ist ein Fehlerzustand. Auch `PQsendQuery` ist nicht erlaubt, weil es das einfache Abfrageprotokoll verwendet. Nachdem alle abgesetzten Befehle verarbeitet und das abschließende Pipeline-Ergebnis konsumiert wurden, kann die Anwendung mit `PQexitPipelineMode` in den Nicht-Pipeline-Modus zurückkehren.

> **Hinweis**
>
> Es ist am besten, den Pipeline-Modus mit `libpq` im nichtblockierenden Modus zu verwenden. Bei Verwendung im blockierenden Modus kann ein Client/Server-Deadlock auftreten. Der Client blockiert dann beim Versuch, Abfragen an den Server zu senden, während der Server blockiert, weil er Ergebnisse bereits verarbeiteter Abfragen an den Client senden möchte. Dies passiert nur, wenn der Client genug Abfragen sendet, um sowohl seinen Ausgabepuffer als auch den Empfangspuffer des Servers zu füllen, bevor er zur Verarbeitung von Eingaben vom Server wechselt; der genaue Zeitpunkt ist aber schwer vorherzusagen.

#### 32.5.1.1. Abfragen ausgeben

Nach dem Eintritt in den Pipeline-Modus setzt die Anwendung Anforderungen mit `PQsendQueryParams` oder der zugehörigen Prepared-Query-Funktion `PQsendQueryPrepared` ab. Diese Anforderungen werden clientseitig in eine Warteschlange gestellt, bis sie zum Server geleert werden; dies geschieht, wenn mit `PQpipelineSync` ein Synchronisationspunkt in der Pipeline eingerichtet wird oder wenn `PQflush` aufgerufen wird. Die Funktionen `PQsendPrepare`, `PQsendDescribePrepared`, `PQsendDescribePortal`, `PQsendClosePrepared` und `PQsendClosePortal` funktionieren ebenfalls im Pipeline-Modus. Die Ergebnisverarbeitung wird unten beschrieben.

Der Server führt Anweisungen aus und gibt Ergebnisse in der Reihenfolge zurück, in der der Client sie sendet. Er beginnt sofort mit der Ausführung der Befehle in der Pipeline und wartet nicht auf das Ende der Pipeline. Beachten Sie, dass Ergebnisse serverseitig gepuffert werden; der Server leert diesen Puffer, wenn ein Synchronisationspunkt mit `PQpipelineSync` oder `PQsendPipelineSync` eingerichtet wird oder wenn `PQsendFlushRequest` aufgerufen wird. Wenn eine Anweisung auf einen Fehler trifft, bricht der Server die aktuelle Transaktion ab und führt keine nachfolgenden Befehle in der Warteschlange bis zum nächsten Synchronisationspunkt aus; für jeden solchen Befehl wird ein Ergebnis `PGRES_PIPELINE_ABORTED` erzeugt. Dies gilt auch dann, wenn die Befehle in der Pipeline die Transaktion zurückrollen würden. Die Abfrageverarbeitung wird nach dem Synchronisationspunkt fortgesetzt.

Es ist in Ordnung, wenn eine Operation von den Ergebnissen einer vorherigen abhängt; beispielsweise kann eine Abfrage eine Tabelle definieren, die die nächste Abfrage in derselben Pipeline verwendet. Ebenso kann eine Anwendung eine benannte vorbereitete Anweisung erstellen und sie mit späteren Anweisungen in derselben Pipeline ausführen.

#### 32.5.1.2. Ergebnisse verarbeiten

Um das Ergebnis einer Abfrage in einer Pipeline zu verarbeiten, ruft die Anwendung wiederholt `PQgetResult` auf und verarbeitet jedes Ergebnis, bis `PQgetResult` null zurückgibt. Das Ergebnis der nächsten Abfrage in der Pipeline kann dann erneut mit `PQgetResult` abgerufen werden, und der Zyklus wiederholt sich. Die Anwendung verarbeitet einzelne Anweisungsergebnisse wie gewohnt. Wenn die Ergebnisse aller Abfragen in der Pipeline zurückgegeben wurden, gibt `PQgetResult` ein Ergebnis mit dem Statuswert `PGRES_PIPELINE_SYNC` zurück.

Der Client kann die Ergebnisverarbeitung aufschieben, bis die vollständige Pipeline gesendet wurde, oder sie mit dem Senden weiterer Abfragen in der Pipeline verzahnen; siehe Abschnitt 32.5.1.4.

`PQgetResult` verhält sich wie bei normaler asynchroner Verarbeitung, außer dass es die neuen `PGresult`-Typen `PGRES_PIPELINE_SYNC` und `PGRES_PIPELINE_ABORTED` enthalten kann. `PGRES_PIPELINE_SYNC` wird genau einmal für jedes `PQpipelineSync` oder `PQsendPipelineSync` an der entsprechenden Stelle in der Pipeline gemeldet. `PGRES_PIPELINE_ABORTED` wird anstelle eines normalen Abfrageergebnisses für den ersten Fehler und alle nachfolgenden Ergebnisse bis zum nächsten `PGRES_PIPELINE_SYNC` ausgegeben; siehe Abschnitt 32.5.1.3.

`PQisBusy`, `PQconsumeInput` usw. funktionieren bei der Verarbeitung von Pipeline-Ergebnissen wie gewohnt. Insbesondere gibt ein Aufruf von `PQisBusy` mitten in einer Pipeline `0` zurück, wenn die Ergebnisse für alle bisher ausgegebenen Abfragen konsumiert wurden.

`libpq` liefert der Anwendung keine Informationen darüber, welche Abfrage gerade verarbeitet wird, außer dass `PQgetResult` null zurückgibt, um anzuzeigen, dass nun die Ergebnisse der nächsten Abfrage folgen. Die Anwendung muss selbst verfolgen, in welcher Reihenfolge sie Abfragen gesendet hat, um sie den entsprechenden Ergebnissen zuzuordnen. Anwendungen verwenden dafür typischerweise eine Zustandsmaschine oder eine FIFO-Warteschlange.

#### 32.5.1.3. Fehlerbehandlung

Aus Sicht des Clients wird die Pipeline als abgebrochen markiert, nachdem `PQresultStatus` `PGRES_FATAL_ERROR` zurückgegeben hat. `PQresultStatus` meldet für jede verbleibende eingereihte Operation in einer abgebrochenen Pipeline ein Ergebnis `PGRES_PIPELINE_ABORTED`. Das Ergebnis für `PQpipelineSync` oder `PQsendPipelineSync` wird als `PGRES_PIPELINE_SYNC` gemeldet, um das Ende der abgebrochenen Pipeline und die Wiederaufnahme normaler Ergebnisverarbeitung zu signalisieren.

Der Client muss während der Fehlerbehebung Ergebnisse mit `PQgetResult` verarbeiten.

Wenn die Pipeline eine implizite Transaktion verwendet hat, werden bereits ausgeführte Operationen zurückgerollt, und Operationen, die nach der fehlgeschlagenen Operation eingereiht waren, werden vollständig übersprungen. Dasselbe Verhalten gilt, wenn die Pipeline eine einzelne explizite Transaktion beginnt und festschreibt, also wenn die erste Anweisung `BEGIN` und die letzte `COMMIT` ist; allerdings bleibt die Sitzung am Ende der Pipeline in einem abgebrochenen Transaktionszustand. Wenn eine Pipeline mehrere explizite Transaktionen enthält, bleiben alle Transaktionen, die vor dem Fehler festgeschrieben wurden, festgeschrieben; die gerade laufende Transaktion wird abgebrochen, und alle nachfolgenden Operationen werden vollständig übersprungen, einschließlich nachfolgender Transaktionen. Wenn ein Pipeline-Synchronisationspunkt bei einem expliziten Transaktionsblock im abgebrochenen Zustand auftritt, wird die nächste Pipeline sofort abgebrochen, sofern nicht der nächste Befehl die Transaktion mit `ROLLBACK` wieder in den normalen Modus versetzt.

> **Hinweis**
>
> Der Client darf nicht annehmen, dass Arbeit festgeschrieben ist, sobald er ein `COMMIT` sendet, sondern erst dann, wenn das zugehörige Ergebnis empfangen wurde und bestätigt, dass der Commit abgeschlossen ist. Weil Fehler asynchron eintreffen, muss die Anwendung in der Lage sein, vom letzten empfangenen festgeschriebenen Änderungsstand neu zu starten und danach ausgeführte Arbeit erneut zu senden, falls etwas schiefgeht.

#### 32.5.1.4. Ergebnisverarbeitung und Abfrageausgabe verzahnen

Um Deadlocks bei großen Pipelines zu vermeiden, sollte der Client um eine nichtblockierende Ereignisschleife herum aufgebaut sein, die Betriebssystemfunktionen wie `select`, `poll`, `WaitForMultipleObjectEx` usw. verwendet.

Die Clientanwendung sollte im Allgemeinen eine Warteschlange für noch auszugebende Arbeit und eine Warteschlange für bereits ausgegebene Arbeit führen, deren Ergebnisse noch nicht verarbeitet wurden. Wenn der Socket schreibbereit ist, sollte sie weitere Arbeit ausgeben. Wenn der Socket lesebereit ist, sollte sie Ergebnisse lesen und verarbeiten und sie dem nächsten Eintrag in der entsprechenden Ergebniswarteschlange zuordnen. Je nach verfügbarem Speicher sollten Ergebnisse vom Socket häufig gelesen werden: Es ist nicht nötig, bis zum Ende der Pipeline zu warten, um die Ergebnisse zu lesen. Pipelines sollten auf logische Arbeitseinheiten begrenzt werden, üblicherweise, aber nicht zwingend, eine Transaktion pro Pipeline. Es ist nicht nötig, den Pipeline-Modus zwischen Pipelines zu verlassen und erneut zu betreten oder auf den Abschluss einer Pipeline zu warten, bevor die nächste gesendet wird.

Ein Beispiel, das `select()` und eine einfache Zustandsmaschine verwendet, um gesendete und empfangene Arbeit zu verfolgen, befindet sich in der PostgreSQL-Quelldistribution unter `src/test/modules/libpq_pipeline/libpq_pipeline.c`.

### 32.5.2. Funktionen zum Pipeline-Modus

**`PQpipelineStatus`**

Gibt den aktuellen Pipeline-Modus-Status der `libpq`-Verbindung zurück.

```c
PGpipelineStatus PQpipelineStatus(const PGconn *conn);
```

`PQpipelineStatus` kann einen der folgenden Werte zurückgeben:

**`PQ_PIPELINE_ON`**

Die `libpq`-Verbindung befindet sich im Pipeline-Modus.

**`PQ_PIPELINE_OFF`**

Die `libpq`-Verbindung befindet sich nicht im Pipeline-Modus.

**`PQ_PIPELINE_ABORTED`**

Die `libpq`-Verbindung befindet sich im Pipeline-Modus, und beim Verarbeiten der aktuellen Pipeline ist ein Fehler aufgetreten. Das Abbruch-Flag wird gelöscht, wenn `PQgetResult` ein Ergebnis des Typs `PGRES_PIPELINE_SYNC` zurückgibt.

**`PQenterPipelineMode`**

Versetzt eine Verbindung in den Pipeline-Modus, wenn sie derzeit untätig oder bereits im Pipeline-Modus ist.

```c
int PQenterPipelineMode(PGconn *conn);
```

Gibt bei Erfolg `1` zurück. Gibt `0` zurück und hat keine Wirkung, wenn die Verbindung derzeit nicht untätig ist, also beispielsweise ein Ergebnis bereitsteht oder sie auf weitere Eingaben vom Server wartet. Diese Funktion sendet tatsächlich nichts an den Server; sie ändert nur den `libpq`-Verbindungszustand.

**`PQexitPipelineMode`**

Veranlasst eine Verbindung, den Pipeline-Modus zu verlassen, wenn sie sich derzeit im Pipeline-Modus mit leerer Warteschlange und ohne ausstehende Ergebnisse befindet.

```c
int PQexitPipelineMode(PGconn *conn);
```

Gibt bei Erfolg `1` zurück. Gibt ebenfalls `1` zurück und unternimmt nichts, wenn sich die Verbindung nicht im Pipeline-Modus befindet. Wenn die aktuelle Anweisung noch nicht fertig verarbeitet ist oder `PQgetResult` nicht aufgerufen wurde, um Ergebnisse aus allen zuvor gesendeten Abfragen einzusammeln, wird `0` zurückgegeben; in diesem Fall liefert `PQerrorMessage` weitere Informationen zum Fehlschlag.

**`PQpipelineSync`**

Markiert einen Synchronisationspunkt in einer Pipeline, indem eine Sync-Nachricht gesendet und der Sendepuffer geleert wird. Dies dient als Begrenzung einer impliziten Transaktion und als Fehlerbehebungspunkt; siehe Abschnitt 32.5.1.3.

```c
int PQpipelineSync(PGconn *conn);
```

Gibt bei Erfolg `1` zurück. Gibt `0` zurück, wenn die Verbindung nicht im Pipeline-Modus ist oder das Senden einer Sync-Nachricht fehlgeschlagen ist.

**`PQsendPipelineSync`**

Markiert einen Synchronisationspunkt in einer Pipeline, indem eine Sync-Nachricht gesendet wird, ohne den Sendepuffer zu leeren. Dies dient als Begrenzung einer impliziten Transaktion und als Fehlerbehebungspunkt; siehe Abschnitt 32.5.1.3.

```c
int PQsendPipelineSync(PGconn *conn);
```

Gibt bei Erfolg `1` zurück. Gibt `0` zurück, wenn die Verbindung nicht im Pipeline-Modus ist oder das Senden einer Sync-Nachricht fehlgeschlagen ist. Beachten Sie, dass die Nachricht selbst nicht automatisch zum Server geleert wird; verwenden Sie bei Bedarf `PQflush`.

**`PQsendFlushRequest`**

Sendet eine Anforderung an den Server, seinen Ausgabepuffer zu leeren.

```c
int PQsendFlushRequest(PGconn *conn);
```

Gibt bei Erfolg `1` zurück. Gibt bei jedem Fehlschlag `0` zurück.

Der Server leert seinen Ausgabepuffer automatisch infolge eines Aufrufs von `PQpipelineSync` oder bei jeder Anforderung, wenn er sich nicht im Pipeline-Modus befindet. Diese Funktion ist nützlich, um den Server im Pipeline-Modus zu veranlassen, seinen Ausgabepuffer zu leeren, ohne einen Synchronisationspunkt einzurichten. Beachten Sie, dass die Anforderung selbst nicht automatisch zum Server geleert wird; verwenden Sie bei Bedarf `PQflush`.

### 32.5.3. Wann der Pipeline-Modus verwendet werden sollte

Ähnlich wie beim asynchronen Abfragemodus gibt es keinen nennenswerten Leistungs-Overhead bei der Verwendung des Pipeline-Modus. Er erhöht die Komplexität der Clientanwendung, und zusätzliche Vorsicht ist nötig, um Client/Server-Deadlocks zu verhindern. Dafür kann der Pipeline-Modus beträchtliche Leistungsverbesserungen bringen, im Austausch für erhöhten Speicherverbrauch, weil Zustand länger vorgehalten wird.

Der Pipeline-Modus ist am nützlichsten, wenn der Server entfernt ist, also die Netzwerklatenz („Ping-Zeit“) hoch ist, und wenn viele kleine Operationen schnell nacheinander ausgeführt werden. Die Verwendung gepipelter Befehle bringt gewöhnlich weniger Vorteile, wenn jede Abfrage ein Vielfaches der Client/Server-Roundtrip-Zeit für ihre Ausführung benötigt. Eine Operation mit 100 Anweisungen, die auf einem Server mit 300 ms Roundtrip-Zeit läuft, würde ohne Pipelining allein durch Netzwerklatenz 30 Sekunden benötigen; mit Pipelining verbringt sie möglicherweise nur 0,3 Sekunden mit Warten auf Ergebnisse vom Server.

Verwenden Sie Pipelining, wenn Ihre Anwendung viele kleine `INSERT`-, `UPDATE`- und `DELETE`-Operationen ausführt, die sich nicht leicht in Mengenoperationen oder in eine `COPY`-Operation umwandeln lassen.

Der Pipeline-Modus ist nicht nützlich, wenn der Client Informationen aus einer Operation benötigt, um die nächste Operation zu erzeugen. In solchen Fällen müsste der Client einen Synchronisationspunkt einführen und eine vollständige Client/Server-Roundtrip-Zeit warten, um die benötigten Ergebnisse zu erhalten. Oft ist es jedoch möglich, das Clientdesign so anzupassen, dass die benötigten Informationen serverseitig ausgetauscht werden. Read-Modify-Write-Zyklen sind besonders gute Kandidaten; zum Beispiel:

```sql
BEGIN;
SELECT x FROM mytable WHERE id = 42 FOR UPDATE;
-- result: x=2
-- client adds 1 to x:
UPDATE mytable SET x = 3 WHERE id = 42;
COMMIT;
```

könnte deutlich effizienter so erledigt werden:

```sql
UPDATE mytable SET x = x + 1 WHERE id = 42;
```

Pipelining ist weniger nützlich und komplexer, wenn eine einzelne Pipeline mehrere Transaktionen enthält; siehe Abschnitt 32.5.1.3.

## 32.6. Abfrageergebnisse in Chunks abrufen

Normalerweise sammelt `libpq` das gesamte Ergebnis eines SQL-Befehls und gibt es der Anwendung als ein einzelnes `PGresult` zurück. Für Befehle, die eine große Anzahl von Zeilen zurückgeben, kann das unpraktisch sein. Für solche Fälle können Anwendungen `PQsendQuery` und `PQgetResult` im Einzelzeilenmodus oder im Chunk-Modus verwenden. In diesen Modi werden Ergebniszeilen an die Anwendung zurückgegeben, sobald sie vom Server empfangen werden: im Einzelzeilenmodus einzeln, im Chunk-Modus in Gruppen.

Um einen dieser Modi zu aktivieren, rufen Sie unmittelbar nach einem erfolgreichen Aufruf von `PQsendQuery` oder einer verwandten Funktion `PQsetSingleRowMode` oder `PQsetChunkedRowsMode` auf. Diese Modusauswahl gilt nur für die aktuell ausgeführte Abfrage. Rufen Sie anschließend wiederholt `PQgetResult` auf, bis es null zurückgibt, wie in [Abschnitt 32.4](32_libpq_C_Bibliothek.md#324-asynchrone-befehlsverarbeitung) beschrieben. Wenn die Abfrage Zeilen zurückgibt, werden sie als ein oder mehrere `PGresult`-Objekte zurückgegeben, die wie normale Abfrageergebnisse aussehen, aber statt `PGRES_TUPLES_OK` den Statuscode `PGRES_SINGLE_TUPLE` für den Einzelzeilenmodus oder `PGRES_TUPLES_CHUNK` für den Chunk-Modus haben. Jedes `PGRES_SINGLE_TUPLE`-Objekt enthält genau eine Ergebniszeile, während ein `PGRES_TUPLES_CHUNK`-Objekt mindestens eine Zeile, aber nicht mehr als die angegebene Anzahl von Zeilen pro Chunk enthält. Nach der letzten Zeile, oder sofort, wenn die Abfrage null Zeilen zurückgibt, wird ein Nullzeilenobjekt mit Status `PGRES_TUPLES_OK` zurückgegeben; dies ist das Signal, dass keine weiteren Zeilen eintreffen werden. Beachten Sie aber, dass `PQgetResult` trotzdem weiter aufgerufen werden muss, bis es null zurückgibt. Alle diese `PGresult`-Objekte enthalten dieselben Zeilenbeschreibungsdaten, also Spaltennamen, Typen usw., die ein gewöhnliches `PGresult`-Objekt für die Abfrage hätte. Jedes Objekt sollte wie üblich mit `PQclear` freigegeben werden.

Bei Verwendung des Pipeline-Modus muss der Einzelzeilen- oder Chunk-Modus für jede Abfrage in der Pipeline aktiviert werden, bevor Ergebnisse für diese Abfrage mit `PQgetResult` abgerufen werden. Weitere Informationen stehen in [Abschnitt 32.5](32_libpq_C_Bibliothek.md#325-pipelinemodus).

**`PQsetSingleRowMode`**

Wählt den Einzelzeilenmodus für die aktuell ausgeführte Abfrage.

```c
int PQsetSingleRowMode(PGconn *conn);
```

Diese Funktion kann nur unmittelbar nach `PQsendQuery` oder einer verwandten Funktion aufgerufen werden, bevor irgendeine andere Operation auf der Verbindung ausgeführt wird, etwa `PQconsumeInput` oder `PQgetResult`. Wenn sie zum richtigen Zeitpunkt aufgerufen wird, aktiviert die Funktion den Einzelzeilenmodus für die aktuelle Abfrage und gibt `1` zurück. Andernfalls bleibt der Modus unverändert und die Funktion gibt `0` zurück. In jedem Fall kehrt der Modus nach Abschluss der aktuellen Abfrage zum Normalzustand zurück.

**`PQsetChunkedRowsMode`**

Wählt den Chunk-Modus für die aktuell ausgeführte Abfrage.

```c
int PQsetChunkedRowsMode(PGconn *conn, int chunkSize);
```

Diese Funktion ähnelt `PQsetSingleRowMode`, gibt aber an, dass bis zu `chunkSize` Zeilen pro `PGresult` abgerufen werden sollen, nicht notwendigerweise nur eine Zeile. Diese Funktion kann nur unmittelbar nach `PQsendQuery` oder einer verwandten Funktion aufgerufen werden, bevor irgendeine andere Operation auf der Verbindung ausgeführt wird, etwa `PQconsumeInput` oder `PQgetResult`. Wenn sie zum richtigen Zeitpunkt aufgerufen wird, aktiviert die Funktion den Chunk-Modus für die aktuelle Abfrage und gibt `1` zurück. Andernfalls bleibt der Modus unverändert und die Funktion gibt `0` zurück. In jedem Fall kehrt der Modus nach Abschluss der aktuellen Abfrage zum Normalzustand zurück.

> **Achtung**
>
> Während der Verarbeitung einer Abfrage kann der Server einige Zeilen zurückgeben und dann auf einen Fehler treffen, wodurch die Abfrage abgebrochen wird. Normalerweise verwirft `libpq` solche Zeilen und meldet nur den Fehler. Im Einzelzeilen- oder Chunk-Modus können jedoch bereits einige Zeilen an die Anwendung zurückgegeben worden sein. Die Anwendung sieht daher einige `PGRES_SINGLE_TUPLE`- oder `PGRES_TUPLES_CHUNK`-`PGresult`-Objekte, gefolgt von einem `PGRES_FATAL_ERROR`-Objekt. Für korrektes Transaktionsverhalten muss die Anwendung so entworfen sein, dass sie alles verwirft oder rückgängig macht, was mit den zuvor verarbeiteten Zeilen getan wurde, falls die Abfrage letztlich fehlschlägt.

## 32.7. Laufende Abfragen abbrechen

### 32.7.1. Funktionen zum Senden von Abbruchanforderungen

**`PQcancelCreate`**

Bereitet eine Verbindung vor, über die eine Abbruchanforderung gesendet werden kann.

```c
PGcancelConn *PQcancelCreate(PGconn *conn);
```

`PQcancelCreate` erzeugt ein `PGcancelConn`-Objekt, beginnt aber nicht sofort damit, über diese Verbindung eine Abbruchanforderung zu senden. Eine Abbruchanforderung kann über diese Verbindung blockierend mit `PQcancelBlocking` und nichtblockierend mit `PQcancelStart` gesendet werden. Der Rückgabewert kann an `PQcancelStatus` übergeben werden, um zu prüfen, ob das `PGcancelConn`-Objekt erfolgreich erzeugt wurde. Das `PGcancelConn`-Objekt ist eine opake Struktur, auf die die Anwendung nicht direkt zugreifen soll. Mit diesem `PGcancelConn`-Objekt kann die auf der ursprünglichen Verbindung laufende Abfrage thread-sicher abgebrochen werden.

Viele Verbindungsparameter des ursprünglichen Clients werden beim Einrichten der Verbindung für die Abbruchanforderung wiederverwendet. Wichtig ist: Wenn die ursprüngliche Verbindung Verschlüsselung und/oder Prüfung des Zielhosts verlangt, etwa über `sslmode` oder `gssencmode`, dann wird die Verbindung für die Abbruchanforderung mit denselben Anforderungen hergestellt. Verbindungsoptionen, die nur während der Authentifizierung oder nach der Authentifizierung des Clients verwendet werden, werden jedoch ignoriert, weil Abbruchanforderungen keine Authentifizierung benötigen und die Verbindung unmittelbar nach dem Absenden der Abbruchanforderung geschlossen wird.

Beachten Sie: Wenn `PQcancelCreate` einen Nicht-Null-Zeiger zurückgibt, müssen Sie `PQcancelFinish` aufrufen, sobald Sie damit fertig sind, um die Struktur und alle zugehörigen Speicherblöcke freizugeben. Dies muss auch dann geschehen, wenn die Abbruchanforderung fehlgeschlagen ist oder aufgegeben wurde.

**`PQcancelBlocking`**

Fordert den Server blockierend auf, die Verarbeitung des aktuellen Befehls aufzugeben.

```c
int PQcancelBlocking(PGcancelConn *cancelConn);
```

Die Anforderung wird über das angegebene `PGcancelConn` gesendet, das mit `PQcancelCreate` erzeugt worden sein muss. Der Rückgabewert von `PQcancelBlocking` ist `1`, wenn die Abbruchanforderung erfolgreich abgesetzt wurde, andernfalls `0`. Wenn sie nicht erfolgreich war, kann die Fehlermeldung mit `PQcancelErrorMessage` abgerufen werden.

Das erfolgreiche Absenden der Abbruchanforderung garantiert jedoch nicht, dass sie tatsächlich eine Wirkung hat. Wenn der Abbruch wirksam ist, endet der abgebrochene Befehl vorzeitig und gibt ein Fehlerergebnis zurück. Wenn der Abbruch fehlschlägt, etwa weil der Server die Verarbeitung des Befehls bereits beendet hatte, gibt es überhaupt kein sichtbares Ergebnis.

**`PQcancelStart`**  
**`PQcancelPoll`**

Fordern den Server nichtblockierend auf, die Verarbeitung des aktuellen Befehls aufzugeben.

```c
int PQcancelStart(PGcancelConn *cancelConn);

PostgresPollingStatusType PQcancelPoll(PGcancelConn *cancelConn);
```

Die Anforderung wird über das angegebene `PGcancelConn` gesendet, das mit `PQcancelCreate` erzeugt worden sein muss. Der Rückgabewert von `PQcancelStart` ist `1`, wenn die Abbruchanforderung gestartet werden konnte, andernfalls `0`. Wenn sie nicht erfolgreich war, kann die Fehlermeldung mit `PQcancelErrorMessage` abgerufen werden.

Wenn `PQcancelStart` erfolgreich ist, besteht die nächste Phase darin, `libpq` zu pollen, damit es mit der Verbindungssequenz für den Abbruch fortfahren kann. Verwenden Sie `PQcancelSocket`, um den Deskriptor des Sockets zu erhalten, der der Datenbankverbindung zugrunde liegt. Achtung: Gehen Sie nicht davon aus, dass der Socket über Aufrufe von `PQcancelPoll` hinweg derselbe bleibt. Die Schleife funktioniert so: Wenn `PQcancelPoll(cancelConn)` zuletzt `PGRES_POLLING_READING` zurückgegeben hat, warten Sie, bis der Socket lesebereit ist, wie durch `select()`, `poll()` oder eine ähnliche Systemfunktion angezeigt. Rufen Sie dann erneut `PQcancelPoll(cancelConn)` auf. Wenn `PQcancelPoll(cancelConn)` zuletzt `PGRES_POLLING_WRITING` zurückgegeben hat, warten Sie entsprechend, bis der Socket schreibbereit ist, und rufen dann wieder `PQcancelPoll(cancelConn)` auf. In der ersten Iteration, also bevor Sie `PQcancelPoll(cancelConn)` aufgerufen haben, verhalten Sie sich so, als hätte es zuletzt `PGRES_POLLING_WRITING` zurückgegeben. Setzen Sie diese Schleife fort, bis `PQcancelPoll(cancelConn)` `PGRES_POLLING_FAILED` zurückgibt, was ein Fehlschlagen der Verbindungsprozedur anzeigt, oder `PGRES_POLLING_OK`, was anzeigt, dass die Abbruchanforderung erfolgreich abgesetzt wurde.

Das erfolgreiche Absenden der Abbruchanforderung garantiert jedoch nicht, dass sie tatsächlich eine Wirkung hat. Wenn der Abbruch wirksam ist, endet der abgebrochene Befehl vorzeitig und gibt ein Fehlerergebnis zurück. Wenn der Abbruch fehlschlägt, etwa weil der Server die Verarbeitung des Befehls bereits beendet hatte, gibt es überhaupt kein sichtbares Ergebnis.

Zu jedem Zeitpunkt während der Verbindung kann der Status der Verbindung durch Aufruf von `PQcancelStatus` geprüft werden. Wenn dieser Aufruf `CONNECTION_BAD` zurückgibt, ist die Abbruchprozedur fehlgeschlagen; wenn er `CONNECTION_OK` zurückgibt, wurde die Abbruchanforderung erfolgreich abgesetzt. Beide Zustände sind ebenso über den oben beschriebenen Rückgabewert von `PQcancelPoll` erkennbar. Andere Zustände können ebenfalls während, und nur während, einer asynchronen Verbindungsprozedur auftreten. Sie zeigen die aktuelle Phase der Verbindungsprozedur an und können zum Beispiel nützlich sein, um dem Benutzer Rückmeldung zu geben. Diese Statuswerte sind:

**`CONNECTION_ALLOCATED`**

Wartet auf einen Aufruf von `PQcancelStart` oder `PQcancelBlocking`, um den Socket tatsächlich zu öffnen. Dies ist der Verbindungszustand direkt nach dem Aufruf von `PQcancelCreate` oder `PQcancelReset`. Zu diesem Zeitpunkt wurde noch keine Verbindung zum Server gestartet. Um die Abbruchanforderung tatsächlich zu senden, verwenden Sie `PQcancelStart` oder `PQcancelBlocking`.

**`CONNECTION_STARTED`**

Wartet darauf, dass die Verbindung hergestellt wird.

**`CONNECTION_MADE`**

Verbindung OK; wartet auf das Senden.

**`CONNECTION_AWAITING_RESPONSE`**

Wartet auf eine Antwort vom Server.

**`CONNECTION_SSL_STARTUP`**

Verhandelt SSL-Verschlüsselung.

**`CONNECTION_GSS_STARTUP`**

Verhandelt GSS-Verschlüsselung.

Beachten Sie: Obwohl diese Konstanten aus Kompatibilitätsgründen erhalten bleiben, sollte eine Anwendung niemals davon ausgehen, dass sie in einer bestimmten Reihenfolge oder überhaupt auftreten oder dass der Status immer einer dieser dokumentierten Werte ist. Eine Anwendung könnte beispielsweise so vorgehen:

```c
switch(PQcancelStatus(conn))
{
    case CONNECTION_STARTED:
        feedback = "Connecting...";
        break;

    case CONNECTION_MADE:
        feedback = "Connected to server...";
        break;

    ...

    default:
        feedback = "Connecting...";
}
```

Der Verbindungsparameter `connect_timeout` wird bei Verwendung von `PQcancelPoll` ignoriert; die Anwendung ist selbst dafür verantwortlich zu entscheiden, ob übermäßig viel Zeit vergangen ist. Ansonsten ist `PQcancelStart` gefolgt von einer `PQcancelPoll`-Schleife äquivalent zu `PQcancelBlocking`.

**`PQcancelStatus`**

Gibt den Status der Abbruchverbindung zurück.

```c
ConnStatusType PQcancelStatus(const PGcancelConn *cancelConn);
```

Der Status kann einer von mehreren Werten sein. Außerhalb einer asynchronen Abbruchprozedur sieht man jedoch nur drei davon: `CONNECTION_ALLOCATED`, `CONNECTION_OK` und `CONNECTION_BAD`. Der Anfangszustand eines `PGcancelConn`, das erfolgreich mit `PQcancelCreate` erzeugt wurde, ist `CONNECTION_ALLOCATED`. Eine erfolgreich abgesetzte Abbruchanforderung hat den Status `CONNECTION_OK`. Ein fehlgeschlagener Abbruchversuch wird durch den Status `CONNECTION_BAD` signalisiert. Ein OK-Status bleibt bestehen, bis `PQcancelFinish` oder `PQcancelReset` aufgerufen wird.

Weitere Statuscodes, die zurückgegeben werden können, sind beim Eintrag zu `PQcancelStart` beschrieben.

Das erfolgreiche Absenden der Abbruchanforderung garantiert jedoch nicht, dass sie tatsächlich eine Wirkung hat. Wenn der Abbruch wirksam ist, endet der abgebrochene Befehl vorzeitig und gibt ein Fehlerergebnis zurück. Wenn der Abbruch fehlschlägt, etwa weil der Server die Verarbeitung des Befehls bereits beendet hatte, gibt es überhaupt kein sichtbares Ergebnis.

**`PQcancelSocket`**

Ermittelt die Dateideskriptornummer des Sockets der Abbruchverbindung zum Server.

```c
int PQcancelSocket(const PGcancelConn *cancelConn);
```

Ein gültiger Deskriptor ist größer oder gleich `0`; ein Ergebnis von `-1` zeigt an, dass derzeit keine Serververbindung geöffnet ist. Dies kann sich durch Aufruf jeder Funktion in diesem Abschnitt auf dem `PGcancelConn` ändern, mit Ausnahme von `PQcancelErrorMessage` und `PQcancelSocket` selbst.

**`PQcancelErrorMessage`**

Gibt die Fehlermeldung zurück, die zuletzt durch eine Operation auf der Abbruchverbindung erzeugt wurde.

```c
char *PQcancelErrorMessage(const PGcancelConn *cancelconn);
```

Fast alle `libpq`-Funktionen, die ein `PGcancelConn` entgegennehmen, setzen bei einem Fehlschlag eine Meldung für `PQcancelErrorMessage`. Beachten Sie, dass nach `libpq`-Konvention ein nichtleeres Ergebnis von `PQcancelErrorMessage` aus mehreren Zeilen bestehen kann und einen abschließenden Zeilenumbruch enthält. Der Aufrufer sollte das Ergebnis nicht direkt freigeben. Es wird freigegeben, wenn das zugehörige `PGcancelConn`-Handle an `PQcancelFinish` übergeben wird. Von der Ergebniszeichenkette sollte nicht erwartet werden, dass sie über Operationen auf der `PGcancelConn`-Struktur hinweg gleich bleibt.

**`PQcancelFinish`**

Schließt die Abbruchverbindung, falls sie das Senden der Abbruchanforderung noch nicht abgeschlossen hatte. Außerdem gibt sie den vom `PGcancelConn`-Objekt verwendeten Speicher frei.

```c
void PQcancelFinish(PGcancelConn *cancelConn);
```

Beachten Sie, dass die Anwendung auch dann `PQcancelFinish` aufrufen sollte, um den vom `PGcancelConn`-Objekt verwendeten Speicher freizugeben, wenn der Abbruchversuch fehlgeschlagen ist, wie durch `PQcancelStatus` angezeigt. Der Zeiger `PGcancelConn` darf nach dem Aufruf von `PQcancelFinish` nicht mehr verwendet werden.

**`PQcancelReset`**

Setzt das `PGcancelConn` zurück, sodass es für eine neue Abbruchverbindung wiederverwendet werden kann.

```c
void PQcancelReset(PGcancelConn *cancelConn);
```

Wenn das `PGcancelConn` derzeit verwendet wird, um eine Abbruchanforderung zu senden, wird diese Verbindung geschlossen. Danach wird das `PGcancelConn`-Objekt so vorbereitet, dass es zum Senden einer neuen Abbruchanforderung verwendet werden kann.

Damit kann ein einziges `PGcancelConn` für ein `PGconn` erzeugt und während der Lebensdauer des ursprünglichen `PGconn` mehrfach wiederverwendet werden.

### 32.7.2. Veraltete Funktionen zum Senden von Abbruchanforderungen

Diese Funktionen stellen ältere Methoden zum Senden von Abbruchanforderungen dar. Obwohl sie weiterhin funktionieren, sind sie veraltet, weil sie Abbruchanforderungen nicht verschlüsselt senden, selbst wenn die ursprüngliche Verbindung über `sslmode` oder `gssencmode` Verschlüsselung verlangt hat. Daher wird von der Verwendung dieser älteren Methoden in neuem Code dringend abgeraten; bestehender Code sollte nach Möglichkeit auf die neuen Funktionen umgestellt werden.

**`PQgetCancel`**

Erzeugt eine Datenstruktur, die die Informationen enthält, die zum Abbrechen eines Befehls mit `PQcancel` benötigt werden.

```c
PGcancel *PQgetCancel(PGconn *conn);
```

`PQgetCancel` erzeugt ein `PGcancel`-Objekt aus einem angegebenen `PGconn`-Verbindungsobjekt. Es gibt `NULL` zurück, wenn das angegebene `conn` `NULL` oder eine ungültige Verbindung ist. Das `PGcancel`-Objekt ist eine opake Struktur, auf die die Anwendung nicht direkt zugreifen soll; sie kann nur an `PQcancel` oder `PQfreeCancel` übergeben werden.

**`PQfreeCancel`**

Gibt eine durch `PQgetCancel` erzeugte Datenstruktur frei.

```c
void PQfreeCancel(PGcancel *cancel);
```

`PQfreeCancel` gibt ein zuvor durch `PQgetCancel` erzeugtes Datenobjekt frei.

**`PQcancel`**

`PQcancel` ist eine veraltete und unsichere Variante von `PQcancelBlocking`, kann aber sicher aus einem Signal-Handler heraus verwendet werden.

```c
int PQcancel(PGcancel *cancel, char *errbuf, int errbufsize);
```

`PQcancel` existiert nur aus Gründen der Abwärtskompatibilität. Stattdessen sollte `PQcancelBlocking` verwendet werden. Der einzige Vorteil von `PQcancel` besteht darin, dass es aus einem Signal-Handler heraus sicher aufgerufen werden kann, wenn `errbuf` eine lokale Variable im Signal-Handler ist. Dieser Vorteil gilt jedoch im Allgemeinen nicht als groß genug, um die Sicherheitsprobleme dieser Funktion aufzuwiegen.

Das `PGcancel`-Objekt ist aus Sicht von `PQcancel` schreibgeschützt, daher kann die Funktion auch aus einem Thread aufgerufen werden, der von dem Thread getrennt ist, der das `PGconn`-Objekt manipuliert.

Der Rückgabewert von `PQcancel` ist `1`, wenn die Abbruchanforderung erfolgreich abgesetzt wurde, andernfalls `0`. Bei Fehlschlag wird `errbuf` mit einer erklärenden Fehlermeldung gefüllt. `errbuf` muss ein `char`-Array der Größe `errbufsize` sein; die empfohlene Größe beträgt 256 Bytes.

**`PQrequestCancel`**

`PQrequestCancel` ist eine veraltete und unsichere Variante von `PQcancelBlocking`.

```c
int PQrequestCancel(PGconn *conn);
```

`PQrequestCancel` existiert nur aus Gründen der Abwärtskompatibilität. Stattdessen sollte `PQcancelBlocking` verwendet werden. Es gibt keinen Vorteil, `PQrequestCancel` statt `PQcancelBlocking` zu verwenden.

Die Funktion fordert den Server auf, die Verarbeitung des aktuellen Befehls aufzugeben. Sie arbeitet direkt auf dem `PGconn`-Objekt und speichert bei einem Fehlschlag die Fehlermeldung im `PGconn`-Objekt, von wo sie mit `PQerrorMessage` abgerufen werden kann. Obwohl die Funktionalität dieselbe ist, ist dieser Ansatz in Programmen mit mehreren Threads oder in Signal-Handlern nicht sicher, weil das Überschreiben der Fehlermeldung des `PGconn` die gerade auf der Verbindung laufende Operation stören kann.

## 32.8. Die Fast-Path-Schnittstelle

PostgreSQL stellt eine Fast-Path-Schnittstelle bereit, um einfache Funktionsaufrufe an den Server zu senden.

> **Tipp**
>
> Diese Schnittstelle ist etwas veraltet, da sich eine ähnliche Leistung und größere Funktionalität erreichen lässt, indem eine vorbereitete Anweisung eingerichtet wird, die den Funktionsaufruf definiert. Die Ausführung dieser Anweisung mit binärer Übertragung von Parametern und Ergebnissen ersetzt dann einen Fast-Path-Funktionsaufruf.

Die Funktion `PQfn` fordert die Ausführung einer Serverfunktion über die Fast-Path-Schnittstelle an:

```c
PGresult *PQfn(PGconn *conn,
               int fnid,
               int *result_buf,
               int *result_len,
               int result_is_int,
               const PQArgBlock *args,
               int nargs);

typedef struct
{
    int len;
    int isint;
    union
    {
        int *ptr;
        int integer;
    } u;
} PQArgBlock;
```

Das Argument `fnid` ist die OID der auszuführenden Funktion. `args` und `nargs` definieren die Parameter, die an die Funktion übergeben werden; sie müssen zur deklarierten Argumentliste der Funktion passen. Wenn das Feld `isint` einer Parameterstruktur wahr ist, wird der Wert `u.integer` als Ganzzahl der angegebenen Länge an den Server gesendet; dies muss 2 oder 4 Bytes sein, und die passende Byte-Umsortierung wird durchgeführt. Wenn `isint` falsch ist, wird die angegebene Anzahl von Bytes bei `*u.ptr` unverarbeitet gesendet; die Daten müssen in dem Format vorliegen, das der Server für die binäre Übertragung des Argumentdatentyps der Funktion erwartet. Die Deklaration von `u.ptr` als Typ `int *` ist historisch; es wäre besser, sie als `void *` zu betrachten.

`result_buf` zeigt auf den Puffer, in dem der Rückgabewert der Funktion abgelegt wird. Der Aufrufer muss ausreichend Speicherplatz für den Rückgabewert bereitgestellt haben; dies wird nicht geprüft. Die tatsächliche Ergebnislänge in Bytes wird in der Ganzzahl zurückgegeben, auf die `result_len` zeigt. Wenn ein 2- oder 4-Byte-Ganzzahlergebnis erwartet wird, setzen Sie `result_is_int` auf `1`, andernfalls auf `0`. Wenn `result_is_int` auf `1` gesetzt ist, führt `libpq` bei Bedarf Byte-Umsortierung durch, sodass der Wert als korrektes `int` für die Clientmaschine geliefert wird; beachten Sie, dass eine 4-Byte-Ganzzahl bei beiden erlaubten Ergebnisgrößen in `*result_buf` geliefert wird. Wenn `result_is_int` `0` ist, wird die vom Server gesendete Bytezeichenkette im Binärformat unverändert zurückgegeben. In diesem Fall ist es besser, `result_buf` als Typ `void *` zu betrachten.

`PQfn` gibt immer einen gültigen `PGresult`-Zeiger zurück, mit Status `PGRES_COMMAND_OK` bei Erfolg oder `PGRES_FATAL_ERROR`, wenn ein Problem aufgetreten ist. Der Ergebnisstatus sollte geprüft werden, bevor das Ergebnis verwendet wird. Der Aufrufer ist dafür verantwortlich, das `PGresult` mit `PQclear` freizugeben, wenn es nicht mehr benötigt wird.

Um der Funktion ein `NULL`-Argument zu übergeben, setzen Sie das Feld `len` der entsprechenden Parameterstruktur auf `-1`; die Felder `isint` und `u` sind dann irrelevant.

Wenn die Funktion `NULL` zurückgibt, wird `*result_len` auf `-1` gesetzt, und `*result_buf` wird nicht verändert.

Beachten Sie, dass es mit dieser Schnittstelle nicht möglich ist, mengenwertige Ergebnisse zu verarbeiten. Außerdem muss die Funktion eine einfache Funktion sein, keine Aggregatfunktion, Fensterfunktion oder Prozedur.

## 32.9. Asynchrone Benachrichtigung

PostgreSQL bietet asynchrone Benachrichtigungen über die Befehle `LISTEN` und `NOTIFY`. Eine Client-Sitzung registriert ihr Interesse an einem bestimmten Benachrichtigungskanal mit dem Befehl `LISTEN` und kann mit `UNLISTEN` wieder aufhören zu lauschen. Alle Sitzungen, die auf einem bestimmten Kanal lauschen, werden asynchron benachrichtigt, wenn eine beliebige Sitzung einen `NOTIFY`-Befehl mit diesem Kanalnamen ausführt. Eine „Payload“-Zeichenkette kann übergeben werden, um zusätzliche Daten an die Listener zu kommunizieren.

`libpq`-Anwendungen senden `LISTEN`-, `UNLISTEN`- und `NOTIFY`-Befehle als gewöhnliche SQL-Befehle. Das Eintreffen von `NOTIFY`-Meldungen kann anschließend durch Aufruf von `PQnotifies` erkannt werden.

Die Funktion `PQnotifies` gibt die nächste Benachrichtigung aus einer Liste unverarbeiteter Benachrichtigungsmeldungen zurück, die vom Server empfangen wurden. Sie gibt einen Nullzeiger zurück, wenn keine Benachrichtigungen ausstehen. Sobald eine Benachrichtigung von `PQnotifies` zurückgegeben wurde, gilt sie als verarbeitet und wird aus der Benachrichtigungsliste entfernt.

```c
PGnotify *PQnotifies(PGconn *conn);

typedef struct pgNotify
{
    char *relname;  /* notification channel name */
    int be_pid;     /* process ID of notifying server process */
    char *extra;    /* notification payload string */
} PGnotify;
```

Nach der Verarbeitung eines von `PQnotifies` zurückgegebenen `PGnotify`-Objekts muss es mit `PQfreemem` freigegeben werden. Es genügt, den `PGnotify`-Zeiger freizugeben; die Felder `relname` und `extra` stellen keine separaten Allokationen dar. Die Namen dieser Felder sind historisch; insbesondere müssen Kanalnamen nichts mit Relationsnamen zu tun haben.

Beispiel 32.2 zeigt ein Beispielprogramm, das die Verwendung asynchroner Benachrichtigungen illustriert.

`PQnotifies` liest selbst keine Daten vom Server; es gibt nur Meldungen zurück, die zuvor von einer anderen `libpq`-Funktion aufgenommen wurden. In sehr alten Versionen von `libpq` bestand die einzige Möglichkeit, den rechtzeitigen Empfang von `NOTIFY`-Meldungen sicherzustellen, darin, ständig Befehle, auch leere, zu senden und anschließend nach jedem `PQexec` `PQnotifies` zu prüfen. Das funktioniert zwar weiterhin, ist aber als Verschwendung von Rechenleistung veraltet.

Eine bessere Methode, nach `NOTIFY`-Meldungen zu suchen, wenn keine nützlichen Befehle auszuführen sind, besteht darin, `PQconsumeInput` aufzurufen und danach `PQnotifies` zu prüfen. Mit `select()` kann auf Daten vom Server gewartet werden, sodass keine CPU-Zeit verbraucht wird, solange nichts zu tun ist. Verwenden Sie `PQsocket`, um die Dateideskriptornummer zu erhalten, die mit `select()` verwendet werden soll. Beachten Sie, dass dies funktioniert, unabhängig davon, ob Befehle mit `PQsendQuery`/`PQgetResult` gesendet werden oder einfach `PQexec` verwendet wird. Denken Sie jedoch daran, nach jedem `PQgetResult` oder `PQexec` `PQnotifies` zu prüfen, um zu sehen, ob während der Befehlsverarbeitung Benachrichtigungen eingetroffen sind.

## 32.10. Funktionen zum `COPY`-Befehl

Der PostgreSQL-Befehl `COPY` hat Optionen, um aus der von `libpq` verwendeten Netzwerkverbindung zu lesen oder in sie zu schreiben. Die in diesem Abschnitt beschriebenen Funktionen erlauben Anwendungen, diese Fähigkeit zu nutzen, indem sie kopierte Daten liefern oder konsumieren.

Der allgemeine Ablauf ist, dass die Anwendung zuerst den SQL-Befehl `COPY` über `PQexec` oder eine der entsprechenden Funktionen ausgibt. Die Antwort darauf, sofern kein Fehler im Befehl vorliegt, ist ein `PGresult`-Objekt mit dem Statuscode `PGRES_COPY_OUT` oder `PGRES_COPY_IN`, abhängig von der angegebenen Kopierrichtung. Die Anwendung sollte dann die Funktionen dieses Abschnitts verwenden, um Datenzeilen zu empfangen oder zu übertragen. Wenn die Datenübertragung abgeschlossen ist, wird ein weiteres `PGresult`-Objekt zurückgegeben, das Erfolg oder Fehlschlag der Übertragung anzeigt. Sein Status ist bei Erfolg `PGRES_COMMAND_OK` oder `PGRES_FATAL_ERROR`, wenn ein Problem aufgetreten ist. Ab diesem Zeitpunkt können weitere SQL-Befehle über `PQexec` ausgegeben werden. Es ist nicht möglich, während einer laufenden `COPY`-Operation andere SQL-Befehle über dieselbe Verbindung auszuführen.

Wenn ein `COPY`-Befehl über `PQexec` in einer Zeichenkette ausgegeben wird, die weitere Befehle enthalten könnte, muss die Anwendung nach Abschluss der `COPY`-Sequenz weiterhin Ergebnisse mit `PQgetResult` abrufen. Erst wenn `PQgetResult` `NULL` zurückgibt, ist sicher, dass die Befehlszeichenkette von `PQexec` abgeschlossen ist und gefahrlos weitere Befehle ausgegeben werden können.

Die Funktionen dieses Abschnitts sollten nur ausgeführt werden, nachdem ein Ergebnisstatus `PGRES_COPY_OUT` oder `PGRES_COPY_IN` von `PQexec` oder `PQgetResult` erhalten wurde.

Ein `PGresult`-Objekt mit einem dieser Statuswerte enthält einige zusätzliche Daten über die beginnende `COPY`-Operation. Diese zusätzlichen Daten sind über Funktionen verfügbar, die auch im Zusammenhang mit Abfrageergebnissen verwendet werden:

**`PQnfields`**

Gibt die Anzahl der zu kopierenden Spalten (Felder) zurück.

**`PQbinaryTuples`**

`0` zeigt an, dass das gesamte Kopierformat textuell ist, also Zeilen durch Zeilenumbrüche, Spalten durch Trennzeichen usw. getrennt werden. `1` zeigt an, dass das gesamte Kopierformat binär ist. Weitere Informationen stehen bei `COPY`.

**`PQfformat`**

Gibt den Formatcode (`0` für Text, `1` für binär) zurück, der jeder Spalte der Kopieroperation zugeordnet ist. Die Formatcodes pro Spalte sind immer null, wenn das gesamte Kopierformat textuell ist; das Binärformat kann jedoch sowohl Text- als auch Binärspalten unterstützen. Allerdings erscheinen in der aktuellen Implementierung von `COPY` in einer binären Kopie nur Binärspalten, sodass die Formate pro Spalte derzeit immer dem Gesamtformat entsprechen.

### 32.10.1. Funktionen zum Senden von `COPY`-Daten

Diese Funktionen werden verwendet, um während `COPY FROM STDIN` Daten zu senden. Sie schlagen fehl, wenn sie aufgerufen werden, während die Verbindung nicht im Zustand `COPY_IN` ist.

**`PQputCopyData`**

Sendet während des Zustands `COPY_IN` Daten an den Server.

```c
int PQputCopyData(PGconn *conn,
                  const char *buffer,
                  int nbytes);
```

Überträgt die `COPY`-Daten im angegebenen Puffer der Länge `nbytes` an den Server. Das Ergebnis ist `1`, wenn die Daten eingereiht wurden, null, wenn sie wegen voller Puffer nicht eingereiht wurden; dies geschieht nur im nichtblockierenden Modus; oder `-1`, wenn ein Fehler aufgetreten ist. Verwenden Sie `PQerrorMessage`, um Details abzurufen, wenn der Rückgabewert `-1` ist. Wenn der Wert null ist, warten Sie auf Schreibbereitschaft und versuchen es erneut.

Die Anwendung kann den `COPY`-Datenstrom in Pufferladungen beliebiger praktischer Größe aufteilen. Grenzen zwischen Pufferladungen haben beim Senden keine semantische Bedeutung. Der Inhalt des Datenstroms muss dem vom `COPY`-Befehl erwarteten Datenformat entsprechen; Einzelheiten stehen bei `COPY`.

**`PQputCopyEnd`**

Sendet während des Zustands `COPY_IN` die Ende-der-Daten-Anzeige an den Server.

```c
int PQputCopyEnd(PGconn *conn, const char *errormsg);
```

Beendet die `COPY_IN`-Operation erfolgreich, wenn `errormsg` `NULL` ist. Wenn `errormsg` nicht `NULL` ist, wird `COPY` zum Fehlschlagen gezwungen, wobei die Zeichenkette, auf die `errormsg` zeigt, als Fehlermeldung verwendet wird. Man sollte jedoch nicht annehmen, dass genau diese Fehlermeldung vom Server zurückkommt, da der Server `COPY` möglicherweise bereits aus eigenen Gründen hat fehlschlagen lassen.

Das Ergebnis ist `1`, wenn die Abschlussmeldung gesendet wurde; im nichtblockierenden Modus kann dies auch nur bedeuten, dass die Abschlussmeldung erfolgreich eingereiht wurde. Um im nichtblockierenden Modus sicherzugehen, dass die Daten gesendet wurden, sollte man danach auf Schreibbereitschaft warten und `PQflush` aufrufen, wiederholt, bis es null zurückgibt. Null zeigt an, dass die Funktion die Abschlussmeldung wegen voller Puffer nicht einreihen konnte; dies geschieht nur im nichtblockierenden Modus. Warten Sie in diesem Fall auf Schreibbereitschaft und versuchen Sie den Aufruf von `PQputCopyEnd` erneut. Wenn ein harter Fehler auftritt, wird `-1` zurückgegeben; Details können mit `PQerrorMessage` abgerufen werden.

Nach einem erfolgreichen Aufruf von `PQputCopyEnd` rufen Sie `PQgetResult` auf, um den endgültigen Ergebnisstatus des `COPY`-Befehls zu erhalten. Auf dieses Ergebnis kann wie üblich gewartet werden. Danach kehrt die Anwendung zum normalen Betrieb zurück.

### 32.10.2. Funktionen zum Empfangen von `COPY`-Daten

Diese Funktionen werden verwendet, um während `COPY TO STDOUT` Daten zu empfangen. Sie schlagen fehl, wenn sie aufgerufen werden, während die Verbindung nicht im Zustand `COPY_OUT` ist.

**`PQgetCopyData`**

Empfängt während des Zustands `COPY_OUT` Daten vom Server.

```c
int PQgetCopyData(PGconn *conn,
                  char **buffer,
                  int async);
```

Versucht, während eines `COPY` eine weitere Datenzeile vom Server zu erhalten. Daten werden immer zeilenweise zurückgegeben; wenn nur eine Teilzeile verfügbar ist, wird sie nicht zurückgegeben. Die erfolgreiche Rückgabe einer Datenzeile umfasst die Allokation eines Speicherblocks, der die Daten enthält. Der Parameter `buffer` darf nicht `NULL` sein. `*buffer` wird so gesetzt, dass es auf den allozierten Speicher zeigt, oder auf `NULL`, wenn kein Puffer zurückgegeben wird. Ein Nicht-Null-Ergebnispuffer sollte mit `PQfreemem` freigegeben werden, wenn er nicht mehr benötigt wird.

Wenn eine Zeile erfolgreich zurückgegeben wird, ist der Rückgabewert die Anzahl der Datenbytes in der Zeile; dieser Wert ist immer größer als null. Die zurückgegebene Zeichenkette ist immer nullterminiert, obwohl dies vermutlich nur für textuelles `COPY` nützlich ist. Ein Ergebnis von null zeigt an, dass `COPY` noch läuft, aber noch keine Zeile verfügbar ist; dies ist nur möglich, wenn `async` wahr ist. Ein Ergebnis von `-1` zeigt an, dass `COPY` abgeschlossen ist. Ein Ergebnis von `-2` zeigt an, dass ein Fehler aufgetreten ist; den Grund liefert `PQerrorMessage`.

Wenn `async` wahr, also nicht null, ist, blockiert `PQgetCopyData` nicht beim Warten auf Eingaben; es gibt null zurück, wenn `COPY` noch läuft, aber keine vollständige Zeile verfügbar ist. In diesem Fall warten Sie auf Lesebereitschaft und rufen dann `PQconsumeInput` auf, bevor Sie `PQgetCopyData` erneut aufrufen. Wenn `async` falsch, also null, ist, blockiert `PQgetCopyData`, bis Daten verfügbar sind oder die Operation abgeschlossen ist.

Nachdem `PQgetCopyData` `-1` zurückgegeben hat, rufen Sie `PQgetResult` auf, um den endgültigen Ergebnisstatus des `COPY`-Befehls zu erhalten. Auf dieses Ergebnis kann wie üblich gewartet werden. Danach kehrt die Anwendung zum normalen Betrieb zurück.

### 32.10.3. Veraltete Funktionen für `COPY`

Diese Funktionen stellen ältere Methoden zur Behandlung von `COPY` dar. Obwohl sie weiterhin funktionieren, sind sie wegen schlechter Fehlerbehandlung, unbequemer Methoden zum Erkennen des Datenendes und fehlender Unterstützung für binäre oder nichtblockierende Übertragungen veraltet.

**`PQgetline`**

Liest eine vom Server übertragene, mit Zeilenumbruch abgeschlossene Zeichenzeile in eine Pufferzeichenkette der Größe `length`.

```c
int PQgetline(PGconn *conn,
              char *buffer,
              int length);
```

Diese Funktion kopiert bis zu `length - 1` Zeichen in den Puffer und wandelt den abschließenden Zeilenumbruch in ein Nullbyte um. `PQgetline` gibt `EOF` am Ende der Eingabe zurück, `0`, wenn die gesamte Zeile gelesen wurde, und `1`, wenn der Puffer voll ist, aber der abschließende Zeilenumbruch noch nicht gelesen wurde.

Beachten Sie, dass die Anwendung prüfen muss, ob eine neue Zeile aus den zwei Zeichen `\.` besteht; dies zeigt an, dass der Server das Senden der Ergebnisse des `COPY`-Befehls abgeschlossen hat. Wenn die Anwendung Zeilen empfangen könnte, die länger als `length - 1` Zeichen sind, ist Sorgfalt nötig, um die `\.`-Zeile korrekt zu erkennen und sie beispielsweise nicht mit dem Ende einer langen Datenzeile zu verwechseln.

**`PQgetlineAsync`**

Liest eine vom Server übertragene Zeile von `COPY`-Daten nichtblockierend in einen Puffer.

```c
int PQgetlineAsync(PGconn *conn,
                   char *buffer,
                   int bufsize);
```

Diese Funktion ähnelt `PQgetline`, kann aber von Anwendungen verwendet werden, die `COPY`-Daten asynchron, also ohne Blockieren, lesen müssen. Nachdem die Anwendung den `COPY`-Befehl ausgegeben und eine Antwort `PGRES_COPY_OUT` erhalten hat, sollte sie `PQconsumeInput` und `PQgetlineAsync` aufrufen, bis das Ende-der-Daten-Signal erkannt wird.

Anders als `PQgetline` übernimmt diese Funktion die Verantwortung für das Erkennen des Datenendes.

Bei jedem Aufruf gibt `PQgetlineAsync` Daten zurück, wenn eine vollständige Datenzeile im Eingabepuffer von `libpq` verfügbar ist. Andernfalls werden keine Daten zurückgegeben, bis der Rest der Zeile eintrifft. Die Funktion gibt `-1` zurück, wenn die Ende-von-COPY-Daten-Markierung erkannt wurde, `0`, wenn keine Daten verfügbar sind, oder eine positive Zahl, die die Anzahl der zurückgegebenen Datenbytes angibt. Wenn `-1` zurückgegeben wird, muss der Aufrufer als Nächstes `PQendcopy` aufrufen und dann zur normalen Verarbeitung zurückkehren.

Die zurückgegebenen Daten reichen nicht über eine Datenzeilengrenze hinaus. Wenn möglich, wird eine ganze Zeile auf einmal zurückgegeben. Wenn der vom Aufrufer angebotene Puffer jedoch zu klein ist, um eine vom Server gesendete Zeile aufzunehmen, wird eine teilweise Datenzeile zurückgegeben. Bei Textdaten kann dies erkannt werden, indem geprüft wird, ob das letzte zurückgegebene Byte `\n` ist. Bei binärem `COPY` muss das `COPY`-Datenformat tatsächlich geparst werden, um die entsprechende Feststellung zu treffen. Die zurückgegebene Zeichenkette ist nicht nullterminiert. Wenn Sie eine abschließende Null hinzufügen möchten, übergeben Sie unbedingt eine `bufsize`, die um eins kleiner ist als der tatsächlich verfügbare Platz.

**`PQputline`**

Sendet eine nullterminierte Zeichenkette an den Server. Gibt `0` bei Erfolg und `EOF` zurück, wenn die Zeichenkette nicht gesendet werden konnte.

```c
int PQputline(PGconn *conn, const char *string);
```

Der durch eine Folge von `PQputline`-Aufrufen gesendete `COPY`-Datenstrom hat dasselbe Format wie der von `PQgetlineAsync` zurückgegebene, außer dass Anwendungen nicht verpflichtet sind, genau eine Datenzeile pro `PQputline`-Aufruf zu senden; es ist in Ordnung, eine Teilzeile oder mehrere Zeilen pro Aufruf zu senden.

> **Hinweis**
>
> Vor PostgreSQL-Protokoll 3.0 musste die Anwendung ausdrücklich die zwei Zeichen `\.` als letzte Zeile senden, um dem Server anzuzeigen, dass sie das Senden von `COPY`-Daten abgeschlossen hatte. Dies funktioniert zwar weiterhin, ist aber veraltet; die besondere Bedeutung von `\.` kann in einem zukünftigen Release entfernt werden. Im CSV-Modus verhält es sich bereits falsch. Es genügt, `PQendcopy` aufzurufen, nachdem die eigentlichen Daten gesendet wurden.

**`PQputnbytes`**

Sendet eine nicht nullterminierte Zeichenkette an den Server. Gibt `0` bei Erfolg und `EOF` zurück, wenn die Zeichenkette nicht gesendet werden konnte.

```c
int PQputnbytes(PGconn *conn,
                const char *buffer,
                int nbytes);
```

Dies ist genau wie `PQputline`, außer dass der Datenpuffer nicht nullterminiert sein muss, weil die Anzahl der zu sendenden Bytes direkt angegeben wird. Verwenden Sie diese Prozedur beim Senden von Binärdaten.

**`PQendcopy`**

Synchronisiert mit dem Server.

```c
int PQendcopy(PGconn *conn);
```

Diese Funktion wartet, bis der Server das Kopieren abgeschlossen hat. Sie sollte entweder ausgegeben werden, wenn die letzte Zeichenkette mit `PQputline` an den Server gesendet wurde, oder wenn die letzte Zeichenkette mit `PQgetline` vom Server empfangen wurde. Sie muss ausgegeben werden, sonst gerät der Server gegenüber dem Client „out of sync“. Nach der Rückkehr aus dieser Funktion ist der Server bereit, den nächsten SQL-Befehl zu empfangen. Der Rückgabewert ist `0` bei erfolgreichem Abschluss, andernfalls von null verschieden. Verwenden Sie `PQerrorMessage`, um Details abzurufen, wenn der Rückgabewert von null verschieden ist.

Bei Verwendung von `PQgetResult` sollte die Anwendung auf ein Ergebnis `PGRES_COPY_OUT` reagieren, indem sie wiederholt `PQgetline` ausführt, gefolgt von `PQendcopy`, nachdem die Terminatorzeile gesehen wurde. Danach sollte sie zur `PQgetResult`-Schleife zurückkehren, bis `PQgetResult` einen Nullzeiger zurückgibt. Entsprechend wird ein Ergebnis `PGRES_COPY_IN` durch eine Folge von `PQputline`-Aufrufen verarbeitet, gefolgt von `PQendcopy`, und danach durch Rückkehr zur `PQgetResult`-Schleife. Diese Anordnung stellt sicher, dass ein `COPY`-Befehl, der in eine Reihe von SQL-Befehlen eingebettet ist, korrekt ausgeführt wird.

Ältere Anwendungen senden ein `COPY` wahrscheinlich über `PQexec` und nehmen an, dass die Transaktion nach `PQendcopy` abgeschlossen ist. Das funktioniert nur dann korrekt, wenn `COPY` der einzige SQL-Befehl in der Befehlszeichenkette ist.

## 32.11. Steuerfunktionen

Diese Funktionen steuern verschiedene Details des Verhaltens von `libpq`.

**`PQclientEncoding`**

Gibt die Client-Kodierung zurück.

```c
int PQclientEncoding(const PGconn *conn);
```

Die Funktion gibt die Kodierungs-ID zurück, nicht eine symbolische Zeichenkette wie `EUC_JP`. Bei Misserfolg gibt sie `-1` zurück. Um eine Kodierungs-ID in einen Kodierungsnamen umzuwandeln, können Sie Folgendes verwenden:

```c
char *pg_encoding_to_char(int encoding_id);
```

**`PQsetClientEncoding`**

Setzt die Client-Kodierung.

```c
int PQsetClientEncoding(PGconn *conn, const char *encoding);
```

`conn` ist die Verbindung zum Server, und `encoding` ist die gewünschte Kodierung. Wenn die Funktion die Kodierung erfolgreich setzt, gibt sie `0` zurück, andernfalls `-1`. Die aktuelle Kodierung dieser Verbindung kann mit `PQclientEncoding` ermittelt werden.

**`PQsetErrorVerbosity`**

Bestimmt die Ausführlichkeit von Meldungen, die von `PQerrorMessage` und `PQresultErrorMessage` zurückgegeben werden.

```c
typedef enum
{
    PQERRORS_TERSE,
    PQERRORS_DEFAULT,
    PQERRORS_VERBOSE,
    PQERRORS_SQLSTATE
} PGVerbosity;

PGVerbosity PQsetErrorVerbosity(PGconn *conn,
                                PGVerbosity verbosity);
```

`PQsetErrorVerbosity` setzt den Ausführlichkeitsmodus und gibt die vorherige Einstellung der Verbindung zurück. Im Modus `TERSE` enthalten die Meldungen nur Schweregrad, Haupttext und Position und passen normalerweise in eine Zeile. `DEFAULT` ergänzt Details, Hinweise oder Kontextfelder. `VERBOSE` enthält alle verfügbaren Felder. `SQLSTATE` enthält nur den Fehlerschweregrad und den SQLSTATE-Fehlercode, falls vorhanden; andernfalls ähnelt die Ausgabe dem Modus `TERSE`.

Eine Änderung dieser Einstellung wirkt sich nicht auf Meldungen aus, die bereits in vorhandenen `PGresult`-Objekten gespeichert sind, sondern nur auf anschließend erzeugte Objekte. Siehe auch `PQresultVerboseErrorMessage`, wenn ein früherer Fehler mit einer anderen Ausführlichkeit ausgegeben werden soll.

**`PQsetErrorContextVisibility`**

Bestimmt die Behandlung von `CONTEXT`-Feldern in Meldungen, die von `PQerrorMessage` und `PQresultErrorMessage` zurückgegeben werden.

```c
typedef enum
{
    PQSHOW_CONTEXT_NEVER,
    PQSHOW_CONTEXT_ERRORS,
    PQSHOW_CONTEXT_ALWAYS
} PGContextVisibility;

PGContextVisibility PQsetErrorContextVisibility(PGconn *conn,
                                                PGContextVisibility show_context);
```

`PQsetErrorContextVisibility` setzt den Anzeigemodus für Kontextinformationen und gibt die vorherige Einstellung der Verbindung zurück. `NEVER` unterdrückt `CONTEXT` immer, `ALWAYS` zeigt es an, wenn es verfügbar ist. Im Standardmodus `ERRORS` werden Kontextfelder nur in Fehlermeldungen angezeigt, nicht in Notices und Warnungen. Wenn die Ausführlichkeit auf `TERSE` oder `SQLSTATE` steht, werden Kontextfelder unabhängig von dieser Einstellung ausgelassen.

**`PQtrace`**

Aktiviert das Protokollieren der Client/Server-Kommunikation in einen Debugging-Dateistrom.

```c
void PQtrace(PGconn *conn, FILE *stream);
```

Jede Zeile besteht aus einem optionalen Zeitstempel, einem Richtungskennzeichen (`F` für Nachrichten vom Client zum Server oder `B` für Nachrichten vom Server zum Client), der Nachrichtenlänge, dem Nachrichtentyp und dem Nachrichteninhalt. Weitere typspezifische Details finden Sie in [Abschnitt 54.7](54_Frontend_Backend_Protokoll.md#547-nachrichtenformate).

> **Hinweis**
>
> Unter Windows kann dieser Funktionsaufruf die Anwendung zum Absturz bringen, wenn die `libpq`-Bibliothek und die Anwendung mit unterschiedlichen Flags kompiliert wurden, weil sich dann die interne Darstellung von `FILE`-Zeigern unterscheiden kann.

**`PQsetTraceFlags`**

Steuert das Trace-Verhalten der Client/Server-Kommunikation.

```c
void PQsetTraceFlags(PGconn *conn, int flags);
```

`flags` enthält Bit-Flags für die Betriebsart des Tracings. `PQTRACE_SUPPRESS_TIMESTAMPS` unterdrückt Zeitstempel. `PQTRACE_REGRESS_MODE` schwärzt einige Felder, etwa Objekt-OIDs, damit die Ausgabe in Testframeworks besser verwendbar ist. Diese Funktion muss nach `PQtrace` aufgerufen werden.

**`PQuntrace`**

Deaktiviert das mit `PQtrace` gestartete Tracing.

```c
void PQuntrace(PGconn *conn);
```

## 32.12. Verschiedene Funktionen

Wie üblich gibt es einige Funktionen, die in keine andere Kategorie passen.

**`PQfreemem`**

Gibt von `libpq` allokierten Speicher frei.

```c
void PQfreemem(void *ptr);
```

Diese Funktion gibt Speicher frei, der von `libpq` allokiert wurde, insbesondere durch `PQescapeByteaConn`, `PQescapeBytea`, `PQunescapeBytea` und `PQnotifies`. Unter Microsoft Windows ist es besonders wichtig, diese Funktion statt `free()` zu verwenden. Auf anderen Plattformen entspricht sie der Standardbibliotheksfunktion `free()`.

**`PQconninfoFree`**

Gibt die Datenstrukturen frei, die von `PQconndefaults` oder `PQconninfoParse` allokiert wurden.

```c
void PQconninfoFree(PQconninfoOption *connOptions);
```

Ist das Argument ein Nullzeiger, wird nichts getan. Ein einfaches `PQfreemem` reicht hier nicht aus, da das Array Verweise auf weitere Zeichenketten enthält.

**`PQencryptPasswordConn`**

Bereitet die verschlüsselte Form eines PostgreSQL-Passworts vor.

```c
char *PQencryptPasswordConn(PGconn *conn,
                            const char *passwd,
                            const char *user,
                            const char *algorithm);
```

Diese Funktion ist für Client-Anwendungen gedacht, die Befehle wie `ALTER USER joe PASSWORD 'pwd'` senden möchten. Es ist gute Praxis, das Klartextpasswort nicht direkt in einem solchen Befehl zu senden, weil es in Befehlsprotokollen, Aktivitätsanzeigen und Ähnlichem sichtbar werden könnte. Stattdessen wandelt diese Funktion das Passwort vor dem Senden in die verschlüsselte Form um.

`passwd` und `user` sind das Klartextpasswort und der SQL-Name des Benutzers. `algorithm` gibt den Verschlüsselungsalgorithmus an. Unterstützt werden derzeit `md5` und `scram-sha-256`; `on` und `off` werden aus Kompatibilitätsgründen ebenfalls als Aliase für `md5` akzeptiert. Unterstützung für `scram-sha-256` wurde in PostgreSQL 10 eingeführt und funktioniert mit älteren Serverversionen nicht korrekt.

Wenn `algorithm` `NULL` ist, fragt diese Funktion den aktuellen Wert von `password_encryption` vom Server ab. Das kann blockieren und schlägt fehl, wenn die aktuelle Transaktion abgebrochen ist oder die Verbindung gerade eine andere Abfrage ausführt. Wer den Standardalgorithmus des Servers verwenden, aber Blockieren vermeiden möchte, sollte `password_encryption` selbst abfragen und diesen Wert übergeben.

Der Rückgabewert ist eine mit `malloc` allokierte Zeichenkette. Geben Sie das Ergebnis mit `PQfreemem` frei. Bei einem Fehler wird `NULL` zurückgegeben und im Verbindungsobjekt eine passende Meldung gespeichert.

**`PQchangePassword`**

Ändert ein PostgreSQL-Passwort.

```c
PGresult *PQchangePassword(PGconn *conn,
                           const char *user,
                           const char *passwd);
```

Diese Funktion verwendet `PQencryptPasswordConn`, um den Befehl `ALTER USER ... PASSWORD '...'` aufzubauen und auszuführen. Sie existiert aus demselben Grund wie `PQencryptPasswordConn`, ist aber bequemer, weil sie den Befehl nicht nur erstellt, sondern auch ausführt. Als Algorithmus wird `NULL` übergeben, sodass die Verschlüsselung gemäß der Servereinstellung `password_encryption` erfolgt.

Zurückgegeben wird ein `PGresult`-Zeiger für das Ergebnis des `ALTER USER`-Befehls oder ein Nullzeiger, wenn die Routine vor dem Senden eines Befehls scheitert. Verwenden Sie `PQresultStatus`, um den Rückgabewert auf Fehler zu prüfen, und `PQerrorMessage` für weitere Informationen.

**`PQencryptPassword`**

Bereitet die mit `md5` verschlüsselte Form eines PostgreSQL-Passworts vor.

```c
char *PQencryptPassword(const char *passwd, const char *user);
```

`PQencryptPassword` ist eine ältere, veraltete Version von `PQencryptPasswordConn`. Sie benötigt kein Verbindungsobjekt und verwendet immer `md5` als Verschlüsselungsalgorithmus.

**`PQmakeEmptyPGresult`**

Erzeugt ein leeres `PGresult`-Objekt mit dem angegebenen Status.

```c
PGresult *PQmakeEmptyPGresult(PGconn *conn, ExecStatusType status);
```

Dies ist eine interne `libpq`-Funktion zum Allokieren und Initialisieren eines leeren `PGresult`-Objekts. Sie gibt `NULL` zurück, wenn kein Speicher allokiert werden konnte. Sie ist exportiert, weil einige Anwendungen selbst Ergebnisobjekte erzeugen möchten, insbesondere Objekte mit Fehlerstatus. Ist `conn` nicht null und zeigt `status` einen Fehler an, wird die aktuelle Fehlermeldung der Verbindung in das `PGresult` kopiert. Registrierte Event-Prozeduren der Verbindung werden ebenfalls kopiert; sie erhalten dabei jedoch keine `PGEVT_RESULTCREATE`-Aufrufe, siehe `PQfireResultCreateEvents`. Wie bei jedem von `libpq` zurückgegebenen `PGresult` muss schließlich `PQclear` aufgerufen werden.

**`PQfireResultCreateEvents`**

Löst für jede im `PGresult`-Objekt registrierte Event-Prozedur ein `PGEVT_RESULTCREATE`-Ereignis aus; siehe [Abschnitt 32.14](32_libpq_C_Bibliothek.md#3214-eventsystem). Gibt bei Erfolg einen von null verschiedenen Wert zurück, oder null, wenn eine Event-Prozedur fehlschlägt.

```c
int PQfireResultCreateEvents(PGconn *conn, PGresult *res);
```

Das Argument `conn` wird an Event-Prozeduren durchgereicht, aber nicht direkt verwendet. Es kann `NULL` sein, wenn die Event-Prozeduren es nicht benötigen. Event-Prozeduren, die für dieses Objekt bereits ein `PGEVT_RESULTCREATE`- oder `PGEVT_RESULTCOPY`-Ereignis erhalten haben, werden nicht erneut aufgerufen.

**`PQcopyResult`**

Erzeugt eine Kopie eines `PGresult`-Objekts.

```c
PGresult *PQcopyResult(const PGresult *src, int flags);
```

Die Kopie ist in keiner Weise mit dem Ursprungsergebnis verknüpft und muss mit `PQclear` freigegeben werden, wenn sie nicht mehr benötigt wird. Bei einem Fehler wird `NULL` zurückgegeben. Die Funktion erzeugt keine exakte Kopie: Das Ergebnis erhält immer den Status `PGRES_TUPLES_OK`, und Fehlermeldungen werden nicht kopiert. Die Kommando-Statuszeichenkette wird jedoch kopiert.

Das Argument `flags` ist ein bitweises Oder mehrerer Flags. `PG_COPYRES_ATTRS` kopiert die Attribute des Ursprungsergebnisses, `PG_COPYRES_TUPLES` die Tupel und implizit auch die Attribute, `PG_COPYRES_NOTICEHOOKS` die Notice-Hooks und `PG_COPYRES_EVENTS` die Events. Instanzdaten des Ursprungs werden dabei nicht automatisch kopiert; Event-Prozeduren erhalten `PGEVT_RESULTCOPY`-Ereignisse.

**`PQsetResultAttrs`**

Setzt die Attribute eines `PGresult`-Objekts.

```c
int PQsetResultAttrs(PGresult *res,
                     int numAttributes,
                     PGresAttDesc *attDescs);
```

Die angegebenen `attDescs` werden in das Ergebnis kopiert. Ist der Zeiger `NULL` oder ist `numAttributes` kleiner als eins, wird die Anforderung ignoriert und gilt als erfolgreich. Enthält `res` bereits Attribute, schlägt die Funktion fehl. Bei Erfolg wird ein von null verschiedener Wert zurückgegeben, andernfalls null.

**`PQsetvalue`**

Setzt den Wert eines Tupelfelds in einem `PGresult`-Objekt.

```c
int PQsetvalue(PGresult *res,
               int tup_num,
               int field_num,
               char *value,
               int len);
```

Die Funktion vergrößert das interne Tupelarray bei Bedarf automatisch. `tup_num` muss jedoch kleiner oder gleich `PQntuples` sein; das Array kann also nur jeweils um ein Tupel wachsen. Jedes Feld eines vorhandenen Tupels kann in beliebiger Reihenfolge geändert werden. Existiert an `field_num` bereits ein Wert, wird er überschrieben. Ist `len` `-1` oder `value` `NULL`, wird ein SQL-Nullwert gesetzt. Der Wert wird in den privaten Speicher des Ergebnisses kopiert. Bei Erfolg wird ein von null verschiedener Wert zurückgegeben, andernfalls null.

**`PQresultAlloc`**

Allokiert zusätzlichen Speicher für ein `PGresult`-Objekt.

```c
void *PQresultAlloc(PGresult *res, size_t nBytes);
```

Mit dieser Funktion allokierter Speicher wird freigegeben, wenn `res` mit `PQclear` freigegeben wird. Bei einem Fehler wird `NULL` zurückgegeben. Das Ergebnis ist wie bei `malloc` ausreichend für jeden Datentyp ausgerichtet.

**`PQresultMemorySize`**

Ermittelt die Anzahl der für ein `PGresult`-Objekt allokierten Bytes.

```c
size_t PQresultMemorySize(const PGresult *res);
```

Der Wert ist die Summe aller mit dem `PGresult` verbundenen `malloc`-Anforderungen, also des Speichers, der durch `PQclear` freigegeben wird. Diese Information kann für die Verwaltung des Speicherverbrauchs hilfreich sein.

**`PQlibVersion`**

Gibt die verwendete Version von `libpq` zurück.

```c
int PQlibVersion(void);
```

Das Ergebnis kann zur Laufzeit verwendet werden, um festzustellen, ob bestimmte Funktionen in der geladenen `libpq`-Version verfügbar sind. Der Wert entsteht, indem die Hauptversionsnummer mit `10000` multipliziert und die Nebenversionsnummer addiert wird. Version 10.1 wird daher als `100001` zurückgegeben, Version 11.0 als `110000`.

Vor Hauptversion 10 verwendete PostgreSQL dreiteilige Versionsnummern, bei denen die ersten beiden Teile zusammen die Hauptversion bildeten. Für diese Versionen verwendet `PQlibVersion` je zwei Ziffern pro Teil, zum Beispiel `90105` für 9.1.5 und `90200` für 9.2.0. Für Feature-Kompatibilität sollten Anwendungen das Ergebnis daher durch `100`, nicht durch `10000`, teilen, um eine logische Hauptversion zu erhalten.

> **Hinweis**
>
> Diese Funktion erschien in PostgreSQL 9.1 und kann daher nicht verwendet werden, um benötigte Funktionalität in noch älteren Versionen zu erkennen, da ihr Aufruf bereits eine Link-Abhängigkeit von Version 9.1 oder neuer erzeugt.

**`PQgetCurrentTimeUSec`**

Ermittelt die aktuelle Zeit als Anzahl der Mikrosekunden seit der Unix-Epoche.

```c
pg_usec_time_t PQgetCurrentTimeUSec(void);
```

Dies ist vor allem nützlich, um Timeout-Werte für `PQsocketPoll` zu berechnen.

## 32.13. Notice-Verarbeitung

Vom Server erzeugte Notice- und Warnmeldungen werden nicht von den Funktionen zur Abfrageausführung zurückgegeben, weil sie keinen Fehlschlag der Abfrage bedeuten. Stattdessen werden sie an eine Notice-Behandlungsfunktion übergeben; die Ausführung läuft danach normal weiter. Die Standardfunktion schreibt die Meldung auf `stderr`, aber eine Anwendung kann dieses Verhalten durch eine eigene Behandlungsfunktion ersetzen.

Aus historischen Gründen gibt es zwei Ebenen: den Notice-Receiver und den Notice-Processor. Standardmäßig formatiert der Notice-Receiver die Notice und übergibt eine Zeichenkette an den Notice-Processor. Eine Anwendung, die einen eigenen Notice-Receiver bereitstellt, ignoriert die Processor-Ebene typischerweise und erledigt die gesamte Arbeit im Receiver.

```c
typedef void (*PQnoticeReceiver) (void *arg, const PGresult *res);

PQnoticeReceiver
PQsetNoticeReceiver(PGconn *conn,
                    PQnoticeReceiver proc,
                    void *arg);

typedef void (*PQnoticeProcessor) (void *arg, const char *message);

PQnoticeProcessor
PQsetNoticeProcessor(PGconn *conn,
                     PQnoticeProcessor proc,
                     void *arg);
```

`PQsetNoticeReceiver` setzt oder ermittelt den aktuellen Notice-Receiver einer Verbindung; `PQsetNoticeProcessor` tut dasselbe für den Notice-Processor. Beide Funktionen geben den vorherigen Funktionszeiger zurück und setzen den neuen Wert. Wird ein Nullzeiger übergeben, wird nichts geändert, aber der aktuelle Zeiger zurückgegeben.

Wenn eine Notice oder Warnung vom Server empfangen oder intern von `libpq` erzeugt wird, ruft `libpq` die Notice-Receiver-Funktion auf. Die Meldung wird als `PGRES_NONFATAL_ERROR`-`PGresult` übergeben, sodass der Receiver einzelne Felder mit `PQresultErrorField` extrahieren oder eine fertig formatierte Meldung mit `PQresultErrorMessage` bzw. `PQresultVerboseErrorMessage` abrufen kann.

Der Standard-Notice-Receiver extrahiert die Meldung mit `PQresultErrorMessage` und übergibt sie an den Notice-Processor. Der Standard-Processor entspricht im Wesentlichen:

```c
static void
defaultNoticeProcessor(void *arg, const char *message)
{
    fprintf(stderr, "%s", message);
}
```

Sobald Sie einen Notice-Receiver oder -Processor gesetzt haben, müssen Sie damit rechnen, dass er aufgerufen werden kann, solange das `PGconn`-Objekt oder daraus erzeugte `PGresult`-Objekte existieren. Beim Erzeugen eines `PGresult` werden die aktuellen Notice-Behandlungszeiger der Verbindung in das Ergebnis kopiert.

## 32.14. Event-System

Das Event-System von `libpq` informiert registrierte Event-Handler über interessante Ereignisse, etwa das Erzeugen oder Zerstören von `PGconn`- und `PGresult`-Objekten. Ein wichtiger Anwendungsfall ist, dass Anwendungen eigene Daten mit einer Verbindung oder einem Ergebnis verknüpfen und diese Daten zum passenden Zeitpunkt freigeben können.

Jeder registrierte Event-Handler ist mit zwei Datenzeigern verbunden, die `libpq` nur als undurchsichtige `void *`-Zeiger kennt. Der Pass-through-Zeiger wird beim Registrieren des Handlers an einem `PGconn` bereitgestellt und bleibt für die Lebensdauer der Verbindung und aller daraus erzeugten Ergebnisse unverändert. Zusätzlich gibt es einen Instanzdatenzeiger, der in jedem `PGconn` und `PGresult` zunächst `NULL` ist. Er kann mit `PQinstanceData`, `PQsetInstanceData`, `PQresultInstanceData` und `PQresultSetInstanceData` verwaltet werden. `libpq` kennt die Bedeutung dieser Zeiger nicht und gibt sie nie selbst frei; das ist Aufgabe des Event-Handlers.

### 32.14.1. Event-Typen

Das Enum `PGEventId` benennt die vom Event-System behandelten Ereignistypen. Alle Werte beginnen mit `PGEVT`. Zu jedem Event-Typ gibt es eine passende Informationsstruktur, deren Zeiger dem Event-Handler übergeben wird.

**`PGEVT_REGISTER`**

Dieses Ereignis tritt auf, wenn `PQregisterEventProc` aufgerufen wird. Es ist der passende Zeitpunkt, um benötigte `instanceData` zu initialisieren. Pro Event-Handler und Verbindung wird dieses Ereignis nur einmal ausgelöst. Gibt die Event-Prozedur null zurück, wird die Registrierung abgebrochen.

```c
typedef struct
{
    PGconn *conn;
} PGEventRegister;
```

**`PGEVT_CONNRESET`**

Dieses Ereignis wird nach einem erfolgreichen `PQreset` oder `PQresetPoll` ausgelöst. Es kann verwendet werden, um zugehörige Instanzdaten zurückzusetzen, neu zu laden oder neu abzufragen.

```c
typedef struct
{
    PGconn *conn;
} PGEventConnReset;
```

**`PGEVT_CONNDESTROY`**

Dieses Ereignis wird als Reaktion auf `PQfinish` ausgelöst. Die Event-Prozedur muss ihre Event-Daten selbst bereinigen, weil `libpq` diesen Speicher nicht verwalten kann.

```c
typedef struct
{
    PGconn *conn;
} PGEventConnDestroy;
```

**`PGEVT_RESULTCREATE`**

Dieses Ereignis wird von jeder Abfrageausführungsfunktion ausgelöst, die ein Ergebnis erzeugt, einschließlich `PQgetResult`. Es wird erst ausgelöst, nachdem das Ergebnis erfolgreich erzeugt wurde.

```c
typedef struct
{
    PGconn *conn;
    PGresult *result;
} PGEventResultCreate;
```

**`PGEVT_RESULTCOPY`**

Dieses Ereignis wird als Reaktion auf `PQcopyResult` ausgelöst, nachdem die Kopie vollständig erstellt wurde. Es kann verwendet werden, um Instanzdaten tief zu kopieren, da `PQcopyResult` dies nicht automatisch tut.

```c
typedef struct
{
    const PGresult *src;
    PGresult *dest;
} PGEventResultCopy;
```

**`PGEVT_RESULTDESTROY`**

Dieses Ereignis wird als Reaktion auf `PQclear` ausgelöst. Die Event-Prozedur muss zugehörige Event-Daten bereinigen; ein Fehler der Event-Prozedur darf die Speicherbereinigung nicht abbrechen.

```c
typedef struct
{
    PGresult *result;
} PGEventResultDestroy;
```

### 32.14.2. Event-Callback-Prozedur

`PGEventProc` ist ein Typedef für einen Zeiger auf eine Event-Prozedur, also die Benutzer-Callback-Funktion, die Events von `libpq` empfängt. Die Signatur lautet:

```c
int eventproc(PGEventId evtId, void *evtInfo, void *passThrough);
```

`evtId` gibt an, welches `PGEVT`-Ereignis eingetreten ist. `evtInfo` muss in die passende Struktur gecastet werden, um weitere Informationen über das Ereignis zu erhalten. `passThrough` ist der Zeiger, der beim Registrieren mit `PQregisterEventProc` bereitgestellt wurde. Die Funktion soll bei Erfolg einen von null verschiedenen Wert und bei Fehler null zurückgeben.

Eine bestimmte Event-Prozedur kann in einem `PGconn` nur einmal registriert werden, weil die Adresse der Prozedur als Schlüssel verwendet wird, um zugehörige Instanzdaten zu finden.

> **Achtung**
>
> Unter Windows können Funktionen zwei unterschiedliche Adressen haben: eine außerhalb und eine innerhalb einer DLL sichtbare Adresse. Für `libpq`-Event-Prozeduren sollte immer dieselbe Adresse verwendet werden. Am einfachsten ist es, Event-Prozeduren als `static` zu deklarieren. Muss ihre Adresse außerhalb der Quelldatei verfügbar sein, sollte eine separate Funktion diese Adresse zurückgeben.


### 32.14.3. Event-Unterstützungsfunktionen

**`PQregisterEventProc`**

Registriert eine Event-Callback-Prozedur bei `libpq`.

```c
int PQregisterEventProc(PGconn *conn,
                        PGEventProc proc,
                        const char *name,
                        void *passThrough);
```

Eine Event-Prozedur muss auf jedem `PGconn`, für das sie Events empfangen soll, einmal registriert werden. Abgesehen vom verfügbaren Speicher gibt es keine feste Grenze für die Anzahl registrierter Event-Prozeduren. `name` wird in Fehlermeldungen verwendet und darf weder `NULL` noch leer sein. `passThrough` wird bei jedem Event an `proc` übergeben und darf `NULL` sein.

**`PQsetInstanceData`**

Setzt die Instanzdaten der Verbindung `conn` für die Prozedur `proc` auf `data`.

```c
int PQsetInstanceData(PGconn *conn, PGEventProc proc, void *data);
```

**`PQinstanceData`**

Gibt die mit `proc` verbundenen Instanzdaten der Verbindung zurück oder `NULL`, wenn keine vorhanden sind.

```c
void *PQinstanceData(const PGconn *conn, PGEventProc proc);
```

**`PQresultSetInstanceData`**

Setzt die Instanzdaten eines Ergebnisses für `proc` auf `data`.

```c
int PQresultSetInstanceData(PGresult *res,
                            PGEventProc proc,
                            void *data);
```

Speicher, auf den `data` zeigt, wird von `PQresultMemorySize` nur berücksichtigt, wenn er mit `PQresultAlloc` allokiert wurde. Das ist empfehlenswert, weil solcher Speicher beim Zerstören des Ergebnisses automatisch freigegeben wird.

**`PQresultInstanceData`**

Gibt die mit `proc` verbundenen Instanzdaten eines Ergebnisses zurück oder `NULL`, wenn keine vorhanden sind.

```c
void *PQresultInstanceData(const PGresult *res, PGEventProc proc);
```


### 32.14.4. Event-Beispiel

Das folgende Grundgerüst zeigt, wie private Daten verwaltet werden können, die mit `libpq`-Verbindungen und Ergebnissen verknüpft sind.

     /* required header for libpq events (note: includes libpq-fe.h) */
     #include <libpq-events.h>

     /* The instanceData */
     typedef struct
     {
         int n;
         char *str;
     } mydata;

     /* PGEventProc */
     static int myEventProc(PGEventId evtId, void *evtInfo, void
      *passThrough);

     int
     main(void)
     {
         mydata *data;
         PGresult *res;
         PGconn *conn =
             PQconnectdb("dbname=postgres options=-csearch_path=");

          if (PQstatus(conn) != CONNECTION_OK)
          {
              /* PQerrorMessage's result includes a trailing newline */
              fprintf(stderr, "%s", PQerrorMessage(conn));
              PQfinish(conn);
              return 1;
          }

         /* called once on any connection that should receive events.
           * Sends a PGEVT_REGISTER to myEventProc.
           */
         if (!PQregisterEventProc(conn, myEventProc, "mydata_proc",
      NULL))
         {
              fprintf(stderr, "Cannot register PGEventProc\n");
              PQfinish(conn);
              return 1;
         }

          /* conn instanceData is available */
          data = PQinstanceData(conn, myEventProc);

          /* Sends a PGEVT_RESULTCREATE to myEventProc */
          res = PQexec(conn, "SELECT 1 + 1");

          /* result instanceData is available */
          data = PQresultInstanceData(res, myEventProc);

         /* If PG_COPYRES_EVENTS is used, sends a PGEVT_RESULTCOPY to
      myEventProc */


    res_copy = PQcopyResult(res, PG_COPYRES_TUPLES |
 PG_COPYRES_EVENTS);

    /* result instanceData is available if PG_COPYRES_EVENTS was
     * used during the PQcopyResult call.
     */
    data = PQresultInstanceData(res_copy, myEventProc);

    /* Both clears send a PGEVT_RESULTDESTROY to myEventProc */
    PQclear(res);
    PQclear(res_copy);

    /* Sends a PGEVT_CONNDESTROY to myEventProc */
    PQfinish(conn);

    return 0;
}

static int
myEventProc(PGEventId evtId, void *evtInfo, void *passThrough)
{
    switch (evtId)
    {
        case PGEVT_REGISTER:
        {
            PGEventRegister *e = (PGEventRegister *)evtInfo;
            mydata *data = get_mydata(e->conn);

            /* associate app specific data with connection */
            PQsetInstanceData(e->conn, myEventProc, data);
            break;
        }

        case PGEVT_CONNRESET:
        {
            PGEventConnReset *e = (PGEventConnReset *)evtInfo;
            mydata *data = PQinstanceData(e->conn, myEventProc);

            if (data)
              memset(data, 0, sizeof(mydata));
            break;
        }

        case PGEVT_CONNDESTROY:
        {
            PGEventConnDestroy *e = (PGEventConnDestroy *)evtInfo;
            mydata *data = PQinstanceData(e->conn, myEventProc);

            /* free instance data because the conn is being
 destroyed */
            if (data)
              free_mydata(data);
            break;
        }

        case PGEVT_RESULTCREATE:
        {


                     PGEventResultCreate *e = (PGEventResultCreate
     *)evtInfo;
                mydata *conn_data = PQinstanceData(e->conn,
     myEventProc);
                mydata *res_data = dup_mydata(conn_data);

                /* associate app specific data with result (copy it
     from conn) */
                PQresultSetInstanceData(e->result, myEventProc,
     res_data);
                break;
            }

            case PGEVT_RESULTCOPY:
            {
                PGEventResultCopy *e = (PGEventResultCopy *)evtInfo;
                mydata *src_data = PQresultInstanceData(e->src,
     myEventProc);
                mydata *dest_data = dup_mydata(src_data);

                 /* associate app specific data with result (copy it
     from a result) */
                 PQresultSetInstanceData(e->dest, myEventProc,
     dest_data);
                 break;
            }

            case PGEVT_RESULTDESTROY:
            {
                PGEventResultDestroy *e = (PGEventResultDestroy
     *)evtInfo;
                mydata *data = PQresultInstanceData(e->result,
     myEventProc);

                /* free instance data because the result is being
     destroyed */
                if (data)
                  free_mydata(data);
                break;
            }

               /* unknown event ID, just return true. */
               default:
                   break;
         }

         return true; /* event processing succeeded */
    }

## 32.15. Umgebungsvariablen

Die folgenden Umgebungsvariablen können Standardwerte für Verbindungsparameter festlegen. Diese Werte werden von `PQconnectdb`, `PQsetdbLogin` und `PQsetdb` verwendet, wenn der aufrufende Code keinen entsprechenden Wert direkt angibt. Das ist besonders für einfache Client-Anwendungen nützlich, die Verbindungsdaten nicht fest einprogrammieren sollen.

| Umgebungsvariable | Entspricht dem Verbindungsparameter |
| --- | --- |
| `PGHOST` | `host` |
| `PGSSLNEGOTIATION` | `sslnegotiation` |
| `PGHOSTADDR` | `hostaddr`; kann statt oder zusätzlich zu `PGHOST` gesetzt werden, um DNS-Lookups zu vermeiden |
| `PGPORT` | `port` |
| `PGDATABASE` | `dbname` |
| `PGUSER` | `user` |
| `PGPASSWORD` | `password`; aus Sicherheitsgründen nicht empfohlen, da manche Betriebssysteme Prozessumgebungen für andere Benutzer sichtbar machen |
| `PGPASSFILE` | `passfile` |
| `PGREQUIREAUTH` | `require_auth` |
| `PGCHANNELBINDING` | `channel_binding` |
| `PGSERVICE` | `service` |
| `PGSERVICEFILE` | benutzerspezifische Dienstdatei; siehe [Abschnitt 32.17](32_libpq_C_Bibliothek.md#3217-die-verbindungsdienstdatei) |
| `PGOPTIONS` | `options` |
| `PGAPPNAME` | `application_name` |
| `PGSSLMODE` | `sslmode` |
| `PGREQUIRESSL` | `requiressl`; veraltet zugunsten von `PGSSLMODE` |
| `PGSSLCOMPRESSION` | `sslcompression` |
| `PGSSLCERT` | `sslcert` |
| `PGSSLKEY` | `sslkey` |
| `PGSSLCERTMODE` | `sslcertmode` |
| `PGSSLROOTCERT` | `sslrootcert` |
| `PGSSLCRL` | `sslcrl` |
| `PGSSLCRLDIR` | `sslcrldir` |
| `PGSSLSNI` | `sslsni` |
| `PGREQUIREPEER` | `requirepeer` |
| `PGSSLMINPROTOCOLVERSION` | `ssl_min_protocol_version` |
| `PGSSLMAXPROTOCOLVERSION` | `ssl_max_protocol_version` |
| `PGGSSENCMODE` | `gssencmode` |
| `PGKRBSRVNAME` | `krbsrvname` |
| `PGGSSLIB` | `gsslib` |
| `PGGSSDELEGATION` | `gssdelegation` |
| `PGCONNECT_TIMEOUT` | `connect_timeout` |
| `PGCLIENTENCODING` | `client_encoding` |
| `PGTARGETSESSIONATTRS` | `target_session_attrs` |
| `PGLOADBALANCEHOSTS` | `load_balance_hosts` |
| `PGMINPROTOCOLVERSION` | `min_protocol_version` |
| `PGMAXPROTOCOLVERSION` | `max_protocol_version` |

`PGDATESTYLE`, `PGTZ` und `PGGEQO` können Standardverhalten für jede PostgreSQL-Sitzung setzen; sie entsprechen sinngemäß `SET datestyle TO ...`, `SET timezone TO ...` und `SET geqo TO ...`. Gültige Werte sind beim SQL-Befehl `SET` beschrieben.

`PGSYSCONFDIR` legt das Verzeichnis fest, das `pg_service.conf` und künftig möglicherweise weitere systemweite Konfigurationsdateien enthält. `PGLOCALEDIR` legt das Verzeichnis mit Locale-Dateien für die Meldungslokalisierung fest.

## 32.16. Die Passwortdatei

Die Datei `.pgpass` im Home-Verzeichnis eines Benutzers kann Passwörter enthalten, die verwendet werden, wenn eine Verbindung ein Passwort benötigt und kein Passwort auf anderem Weg angegeben wurde. Auf Unix-Systemen wird das Home-Verzeichnis über `HOME` oder, falls diese Variable nicht gesetzt ist, über das Home-Verzeichnis des effektiven Benutzers bestimmt. Unter Microsoft Windows heißt die Datei `%APPDATA%\postgresql\pgpass.conf`. Alternativ kann die Passwortdatei mit dem Verbindungsparameter `passfile` oder der Umgebungsvariable `PGPASSFILE` angegeben werden.

Die Datei enthält Zeilen im folgenden Format:

```text
hostname:port:database:username:password
```

Jedes der ersten vier Felder kann ein Literalwert oder `*` sein; `*` passt auf alles. Das Passwort aus der ersten Zeile, die zu den aktuellen Verbindungsparametern passt, wird verwendet. Spezifischere Einträge sollten daher vor allgemeineren Wildcard-Einträgen stehen. Müssen `:` oder `\` in einem Feld vorkommen, werden sie mit `\` maskiert.

Auf Unix-Systemen darf die Passwortdatei weder für Gruppe noch für andere Benutzer zugänglich sein, zum Beispiel:

```sh
chmod 0600 ~/.pgpass
```

Sind die Berechtigungen weniger streng, ignoriert `libpq` die Datei. Unter Microsoft Windows wird angenommen, dass das Profilverzeichnis ausreichend geschützt ist; dort erfolgt keine entsprechende Berechtigungsprüfung.

## 32.17. Die Verbindungsdienstdatei

Die Verbindungsdienstdatei ordnet einem Servicenamen eine Menge von `libpq`-Verbindungsparametern zu. Der Servicename kann dann mit dem Schlüsselwort `service` in einem Verbindungsstring oder über die Umgebungsvariable `PGSERVICE` angegeben werden. Dadurch lassen sich Verbindungsparameter ändern, ohne die Anwendung neu zu kompilieren.

Servicenamen können in einer benutzerspezifischen oder in einer systemweiten Dienstdatei definiert werden. Existiert derselbe Servicename in beiden Dateien, hat die Benutzerdatei Vorrang. Standardmäßig heißt die Benutzerdatei `~/.pg_service.conf`; unter Microsoft Windows heißt sie `%APPDATA%\postgresql\.pg_service.conf`. Ein anderer Dateiname kann mit `PGSERVICEFILE` angegeben werden. Die systemweite Datei heißt `pg_service.conf` und wird standardmäßig im `etc`-Verzeichnis der PostgreSQL-Installation gesucht; `pg_config --sysconfdir` zeigt dieses Verzeichnis an. Ein anderes Verzeichnis kann mit `PGSYSCONFDIR` angegeben werden.

Die Dienstdatei verwendet ein INI-Format. Der Abschnittsname ist der Servicename, die Einträge sind Verbindungsparameter:

```ini
# Kommentar
[mydb]
host=somehost
port=5433
user=admin
```

Eine Beispieldatei wird unter `share/pg_service.conf.sample` installiert. Parameter aus einer Dienstdatei werden mit Parametern aus anderen Quellen kombiniert. Ein Eintrag aus der Dienstdatei überschreibt die entsprechende Umgebungsvariable, kann aber wiederum durch einen direkt im Verbindungsstring angegebenen Wert überschrieben werden.

## 32.18. LDAP-Suche nach Verbindungsparametern

Wenn `libpq` mit LDAP-Unterstützung kompiliert wurde, können Verbindungsoptionen wie `host` oder `dbname` über LDAP von einem zentralen Server abgefragt werden. Ändern sich die Verbindungsparameter einer Datenbank, müssen dann nicht alle Client-Rechner einzeln aktualisiert werden.

Die LDAP-Suche verwendet die Dienstdatei `pg_service.conf`. Beginnt eine Zeile innerhalb eines Dienstabschnitts mit `ldap://`, wird sie als LDAP-URL erkannt und eine LDAP-Abfrage ausgeführt. Das Ergebnis muss eine Liste von `keyword = value`-Paaren sein, die als Verbindungsoptionen verwendet werden. Die URL hat die Form:

```text
ldap://[hostname[:port]]/search_base?attribute?search_scope?filter
```

`hostname` ist standardmäßig `localhost`, `port` standardmäßig `389`. Nach einer erfolgreichen LDAP-Suche endet die Verarbeitung von `pg_service.conf`. Kann der LDAP-Server nicht erreicht werden, wird die Verarbeitung fortgesetzt, sodass weitere LDAP-URLs, klassische `keyword = value`-Paare oder Standardoptionen als Fallback dienen können.

Beispiel für einen Dienstabschnitt, der reguläre Angaben und LDAP kombiniert:

```ini
[customerdb]
dbname=customer
user=appuser
ldap://ldap.acme.com/cn=dbserver,cn=hosts?pgconnectinfo?base?(objectclass=*)
```

## 32.19. SSL-Unterstützung

PostgreSQL unterstützt SSL-Verbindungen nativ, um die Client/Server-Kommunikation mit TLS-Protokollen zu verschlüsseln. Details zur serverseitigen SSL-Funktionalität stehen in [Abschnitt 18.9](18_Servereinrichtung_und_Betrieb.md#189-sichere-tcpipverbindungen-mit-ssl).

`libpq` liest die systemweite OpenSSL-Konfigurationsdatei. Standardmäßig heißt diese Datei `openssl.cnf` und befindet sich in dem Verzeichnis, das `openssl version -d` meldet. Dieser Standard kann mit der Umgebungsvariable `OPENSSL_CONF` überschrieben werden.

### 32.19.1. Clientseitige Prüfung von Serverzertifikaten

Standardmäßig prüft PostgreSQL das Serverzertifikat nicht. Dadurch könnte die Serveridentität vorgetäuscht werden, etwa durch manipulierte DNS-Einträge oder Übernahme einer IP-Adresse. Um das zu verhindern, muss der Client die Identität des Servers über eine Vertrauenskette prüfen können.

Bei `sslmode=verify-ca` prüft `libpq`, ob das Serverzertifikat über die Zertifikatskette zu einem auf dem Client gespeicherten Root-Zertifikat führt. Bei `sslmode=verify-full` prüft `libpq` zusätzlich, ob der Hostname des Servers zum Namen im Serverzertifikat passt. In sicherheitssensiblen Umgebungen wird `verify-full` empfohlen.

Im Modus `verify-full` wird der Hostname gegen die Subject-Alternative-Name-Attribute des Zertifikats geprüft, oder gegen den Common Name, wenn kein passender `dNSName`-SAN vorhanden ist. Wildcards mit führendem `*` passen nur innerhalb eines Namenssegments und nicht über Punkte hinweg. Wird eine IP-Adresse verwendet, wird diese ohne DNS-Lookup gegen passende SANs oder, falls nötig, gegen den Common Name geprüft.

Root-Zertifikate für die Serverzertifikatsprüfung müssen in `~/.postgresql/root.crt` liegen; unter Microsoft Windows heißt die Datei `%APPDATA%\postgresql\root.crt`. Sperrlisten werden geprüft, wenn `~/.postgresql/root.crl` beziehungsweise `%APPDATA%\postgresql\root.crl` existiert. Die Speicherorte können mit `sslrootcert`, `sslcrl`, `sslcrldir` beziehungsweise den Umgebungsvariablen `PGSSLROOTCERT`, `PGSSLCRL` und `PGSSLCRLDIR` geändert werden.

> **Hinweis**
>
> Aus Kompatibilitätsgründen verhält sich `sslmode=require` wie `verify-ca`, wenn eine Root-CA-Datei vorhanden ist. Anwendungen, die Zertifikatsprüfung benötigen, sollten sich nicht darauf verlassen, sondern ausdrücklich `verify-ca` oder `verify-full` verwenden.

### 32.19.2. Client-Zertifikate

Wenn der Server die Identität des Clients durch ein Client-Zertifikat prüft, sendet `libpq` die Zertifikate aus `~/.postgresql/postgresql.crt`; der passende private Schlüssel muss in `~/.postgresql/postgresql.key` liegen. Unter Microsoft Windows heißen diese Dateien `%APPDATA%\postgresql\postgresql.crt` und `%APPDATA%\postgresql\postgresql.key`. Die Orte können mit `sslcert`, `sslkey`, `PGSSLCERT` und `PGSSLKEY` überschrieben werden.

Auf Unix-Systemen darf die private Schlüsseldatei für Gruppe und andere nicht zugänglich sein, zum Beispiel mit `chmod 0600 ~/.postgresql/postgresql.key`. Alternativ kann die Datei `root` gehören und gruppenlesbar sein (`0640`), wenn Zertifikate und Schlüssel vom Betriebssystem verwaltet werden. Das erste Zertifikat in `postgresql.crt` muss das Client-Zertifikat sein, weil es zum privaten Schlüssel passen muss. Zwischenzertifikate können angehängt werden.

Zertifikat und Schlüssel können im PEM- oder ASN.1-DER-Format vorliegen. Ist der Schlüssel mit einer Passphrase verschlüsselt, kann diese über `sslpassword` bereitgestellt werden; andernfalls fragt OpenSSL interaktiv danach, falls ein TTY verfügbar ist.

### 32.19.3. Schutzwirkung der verschiedenen Modi

Die Werte von `sslmode` bieten unterschiedliche Schutzstufen. SSL kann gegen Abhören, Man-in-the-Middle-Angriffe und Client-Impersonation schützen. Damit eine Verbindung zuverlässig SSL-gesichert ist, muss SSL vor dem Verbindungsaufbau auf Client und Server passend konfiguriert sein. In `libpq` wird eine sichere Serverprüfung durch `sslmode=verify-full` oder `sslmode=verify-ca` plus ein vertrauenswürdiges Root-Zertifikat erreicht.

| `sslmode` | Schutz vor Abhören | MITM-Schutz | Bedeutung |
| --- | --- | --- | --- |
| `disable` | Nein | Nein | Keine SSL-Nutzung. |
| `allow` | Vielleicht | Nein | SSL wird akzeptiert, wenn der Server es verlangt. |
| `prefer` | Vielleicht | Nein | SSL wird bevorzugt, falls der Server es unterstützt. |
| `require` | Ja | Nein | Verschlüsselung wird verlangt, aber der Server wird nicht geprüft. |
| `verify-ca` | Ja | abhängig von der CA-Politik | Das Zertifikat muss von einer vertrauenswürdigen CA stammen. |
| `verify-full` | Ja | Ja | Zertifikat und Servername müssen passen. |

Der Unterschied zwischen `verify-ca` und `verify-full` hängt von der CA-Politik ab. Bei öffentlichen CAs sollte immer `verify-full` verwendet werden, weil sonst auch ein Zertifikat für einen fremden, aber CA-signierten Server akzeptiert werden könnte. Der Standardwert von `sslmode` ist `prefer`; das ist aus Sicherheitssicht keine starke Einstellung und existiert vor allem aus Gründen der Abwärtskompatibilität.

### 32.19.4. Verwendung der SSL-Clientdateien

| Datei | Inhalt | Wirkung |
| --- | --- | --- |
| `~/.postgresql/postgresql.crt` | Client-Zertifikat | wird an den Server gesendet |
| `~/.postgresql/postgresql.key` | privater Client-Schlüssel | belegt den Besitz des gesendeten Client-Zertifikats |
| `~/.postgresql/root.crt` | vertrauenswürdige Zertifizierungsstellen | prüft, ob das Serverzertifikat von einer vertrauenswürdigen CA signiert wurde |
| `~/.postgresql/root.crl` | gesperrte Zertifikate | das Serverzertifikat darf nicht auf dieser Liste stehen |

### 32.19.5. Initialisierung der SSL-Bibliothek

Anwendungen, die mit älteren PostgreSQL-Versionen und OpenSSL 1.0.2 oder älter kompatibel sein müssen, mussten die SSL-Bibliothek vor der Verwendung initialisieren. Anwendungen, die `libssl` oder `libcrypto` selbst initialisieren, konnten `PQinitOpenSSL` verwenden, um `libpq` darüber zu informieren.

```c
void PQinitOpenSSL(int do_ssl, int do_crypto);
void PQinitSSL(int do_ssl);
```

Beide Funktionen sind veraltet, nur noch aus Kompatibilitätsgründen vorhanden und tun nichts. Seit PostgreSQL 18 sind sie nicht mehr erforderlich.

## 32.20. OAuth-Unterstützung

`libpq` implementiert Unterstützung für den OAuth-v2-Device-Authorization-Client-Flow als optionales Modul. Ist diese Unterstützung aktiviert und das Modul installiert, verwendet `libpq` standardmäßig den eingebauten Flow, wenn der Server während der Authentifizierung ein Bearer-Token anfordert. Das funktioniert auch auf Systemen ohne nutzbaren Webbrowser, etwa bei einem Client über SSH.

Der eingebaute Flow gibt standardmäßig eine URL und einen Benutzercode aus:

```text
$ psql 'dbname=postgres oauth_issuer=https://example.com oauth_client_id=...'
Visit https://example.com/device and enter the code: ABCD-EFGH
```

Der Benutzer meldet sich anschließend beim OAuth-Provider an und bestätigt, dass `libpq` und der Server in seinem Namen handeln dürfen. URL und angezeigte Berechtigungen sollten vor dem Fortfahren sorgfältig geprüft werden.

Für einen OAuth-Client-Flow muss der Verbindungsstring mindestens `oauth_issuer` und `oauth_client_id` enthalten. Der eingebaute Flow benötigt außerdem einen Device-Authorization-Endpunkt des OAuth-Autorisierungsservers.

> **Hinweis**
>
> Der eingebaute Device-Authorization-Flow wird derzeit unter Windows nicht unterstützt. Eigene Client-Flows können weiterhin implementiert werden.

### 32.20.1. Authdata-Hooks

Das Verhalten des OAuth-Flows kann von einem Client über die folgende Hook-API geändert oder ersetzt werden.

```c
void PQsetAuthDataHook(PQauthDataHook_type hook);
PQauthDataHook_type PQgetAuthDataHook(void);
```

`PQsetAuthDataHook` setzt den `PGauthDataHook` und überschreibt damit einen oder mehrere Aspekte des OAuth-Client-Flows. Ist `hook` `NULL`, wird der Standard-Handler wieder installiert. Andernfalls übergibt die Anwendung einen Callback mit folgender Signatur:

```c
int hook_fn(PGauthData type, PGconn *conn, void *data);
```

`libpq` ruft diesen Callback auf, wenn eine Aktion der Anwendung erforderlich ist. `type` beschreibt die Anforderung, `conn` ist das zu authentifizierende Verbindungshandle, und `data` verweist auf anforderungsspezifische Metadaten. Hooks können verkettet werden, um kooperatives oder fallbackartiges Verhalten zu ermöglichen. Erfolg wird durch einen Rückgabewert größer als null angezeigt; ein negativer Wert signalisiert einen Fehler und bricht den Verbindungsversuch ab.

#### 32.20.1.1. Hook-Typen

Die folgenden `PGauthData`-Typen und zugehörigen Datenstrukturen sind definiert.

**`PQAUTHDATA_PROMPT_OAUTH_DEVICE`**

Ersetzt die Standard-Benutzeraufforderung während des eingebauten Device-Authorization-Flows. `data` zeigt auf eine Instanz von `PGpromptOAuthDevice`:

```c
typedef struct _PGpromptOAuthDevice
{
    const char *verification_uri;
    const char *user_code;
    const char *verification_uri_complete;
    int         expires_in;
} PGpromptOAuthDevice;
```

Der Standard-Prompt schreibt `verification_uri` und `user_code` auf die Standardfehlerausgabe. Ersatzimplementierungen können diese Informationen beliebig darstellen, etwa in einer grafischen Oberfläche. `verification_uri_complete` kann zusätzlich für nichttextuelle Bestätigungsmethoden wie QR-Codes verwendet werden.

**`PQAUTHDATA_OAUTH_BEARER_TOKEN`**

Erlaubt eine eigene Flow-Implementierung und ersetzt den eingebauten Flow, falls dieser installiert ist. Der Hook sollte entweder direkt ein Bearer-Token für die aktuelle Benutzer/Issuer/Scope-Kombination zurückgeben oder einen asynchronen Callback einrichten, der ein Token beschafft.

```c
typedef struct PGoauthBearerRequest
{
    const char *openid_configuration;
    const char *scope;
    PostgresPollingStatusType (*async) (PGconn *conn,
                                        struct PGoauthBearerRequest *request,
                                        SOCKTYPE *altsock);
    void        (*cleanup) (PGconn *conn,
                            struct PGoauthBearerRequest *request);
    char       *token;
    void       *user;
} PGoauthBearerRequest;
```

`openid_configuration` enthält die URL eines OAuth-Discovery-Dokuments, `scope` die angeforderten OAuth-Scopes. Das endgültige Ergebnis ist `token`, das auf ein gültiges Bearer-Token für die Verbindung zeigen muss. Kann der Hook nicht sofort ein Token liefern, sollte er den `async`-Callback setzen und nichtblockierende Kommunikation verwenden. Blockierende Operationen innerhalb des Hooks stören nichtblockierende Verbindungs-APIs wie `PQconnectPoll`.

### 32.20.2. Debugging- und Entwicklereinstellungen

Ein gefährlicher Debugging-Modus kann mit `PGOAUTHDEBUG=UNSAFE` aktiviert werden. Er ist nur für lokale Entwicklung und Tests gedacht. Er erlaubt unverschlüsseltes HTTP beim Austausch mit dem OAuth-Provider, kann die vertrauenswürdige CA-Liste über `PGOAUTHCAFILE` ersetzen, schreibt HTTP-Verkehr mit kritischen Geheimnissen auf die Standardfehlerausgabe und erlaubt Retry-Intervalle von null Sekunden.

> **Warnung**
>
> Geben Sie die Ausgabe des OAuth-Flows nicht an Dritte weiter. Sie enthält Geheimnisse, die zum Angriff auf Clients und Server verwendet werden können.

## 32.21. Verhalten in Programmen mit Threads

Seit Version 17 ist `libpq` immer reentrant und thread-sicher. Es gilt jedoch die Einschränkung, dass nicht zwei Threads gleichzeitig dasselbe `PGconn`-Objekt manipulieren dürfen. Insbesondere können nicht mehrere Threads gleichzeitig Befehle über dieselbe Verbindung senden. Verwenden Sie für parallele Befehle mehrere Verbindungen.

`PGresult`-Objekte sind nach ihrer Erzeugung normalerweise schreibgeschützt und können daher frei zwischen Threads weitergegeben werden. Wenn Sie jedoch die in [Abschnitt 32.12](32_libpq_C_Bibliothek.md#3212-verschiedene-funktionen) oder [Abschnitt 32.14](32_libpq_C_Bibliothek.md#3214-eventsystem) beschriebenen Funktionen verwenden, die `PGresult` verändern, müssen Sie parallele Operationen auf demselben Objekt selbst verhindern.

```c
int PQisthreadsafe(void);
```

`PQisthreadsafe` gibt `1` zurück, wenn `libpq` thread-sicher ist, andernfalls `0`. Ab Version 17 wird immer `1` zurückgegeben.

Die veralteten Funktionen `PQrequestCancel` und `PQoidStatus` sind nicht thread-sicher und sollten in Programmen mit Threads nicht verwendet werden. Verwenden Sie stattdessen `PQcancelBlocking` beziehungsweise `PQoidValue`. Wenn Ihre Anwendung zusätzlich zu `libpq` Kerberos oder Curl verwendet, können je nach Bibliotheksversion kooperative Sperren über `PQregisterThreadLock` erforderlich sein.

## 32.22. `libpq`-Programme erstellen

Um ein Programm mit `libpq` zu erstellen, müssen Sie den Header einbinden, den Compiler auf die PostgreSQL-Header verweisen und beim Linken die `libpq`-Bibliothek angeben.

```c
#include <libpq-fe.h>
```

Fehlt dieser Header, meldet der Compiler typischerweise unbekannte Bezeichner wie `PGconn`, `PGresult`, `CONNECTION_BAD` oder `PGRES_TUPLES_OK`.

Der Include-Pfad wird mit `-I` angegeben:

```sh
cc -c -I/usr/local/pgsql/include testprog.c
```

In Makefiles gehört die Option normalerweise in `CPPFLAGS`:

```make
CPPFLAGS += -I/usr/local/pgsql/include
```

Für portable Builds sollte der Pfad nicht fest codiert werden. Verwenden Sie stattdessen `pg_config` oder `pkg-config`:

```sh
pg_config --includedir
pkg-config --cflags libpq
```

Beim Linken geben Sie `-lpq` an und bei Bedarf mit `-L` das Bibliotheksverzeichnis. Für maximale Portabilität steht `-L` vor `-lpq`:

```sh
cc -o testprog testprog1.o testprog2.o -L/usr/local/pgsql/lib -lpq
```

Das Bibliotheksverzeichnis erhalten Sie ebenfalls über `pg_config` oder `pkg-config`:

```sh
pg_config --libdir
pkg-config --libs libpq
```

Fehlermeldungen wie `undefined reference to 'PQstatus'` bedeuten meist, dass `-lpq` fehlt. `cannot find -lpq` weist darauf hin, dass die `-L`-Option fehlt oder auf das falsche Verzeichnis zeigt.

## 32.23. Beispielprogramme

Diese und weitere Beispiele befinden sich im Verzeichnis `src/test/examples` der Quellcodedistribution.

**Beispiel 32.1. `libpq`-Beispielprogramm 1**

```c
/*
 * src/test/examples/testlibpq.c
 *
 * testlibpq.c
 *      Test der C-Version von libpq, der PostgreSQL-Frontend-Bibliothek.
 */

#include <stdio.h>
#include <stdlib.h>
#include "libpq-fe.h"

static void
exit_nicely(PGconn *conn)
{
    PQfinish(conn);
    exit(1);
}

int
main(int argc, char **argv)
{
    const char *conninfo;
    PGconn     *conn;
    PGresult   *res;
    int         nFields;
    int         i,
                j;

    /*
     * Wenn ein Befehlszeilenparameter angegeben wurde, verwende ihn als
     * conninfo-String. Andernfalls wird dbname=postgres gesetzt und alle
     * weiteren Verbindungsparameter kommen aus Umgebungsvariablen oder
     * Standardwerten.
     */
    if (argc > 1)
        conninfo = argv[1];
    else
        conninfo = "dbname = postgres";

    /* Verbindung zur Datenbank herstellen. */
    conn = PQconnectdb(conninfo);

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

    /* PGresult immer freigeben, sobald es nicht mehr benötigt wird. */
    PQclear(res);

    /*
     * Dieses Beispiel verwendet einen Cursor. Dafür muss es innerhalb eines
     * Transaktionsblocks laufen. Ein einzelnes PQexec() mit
     * "select * from pg_database" wäre zu schlicht für ein gutes Beispiel.
     */

    /* Transaktionsblock starten. */
    res = PQexec(conn, "BEGIN");
    if (PQresultStatus(res) != PGRES_COMMAND_OK)
    {
        fprintf(stderr, "BEGIN command failed: %s", PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }
    PQclear(res);

    /* Zeilen aus pg_database, dem Systemkatalog der Datenbanken, abrufen. */
    res = PQexec(conn,
                 "DECLARE myportal CURSOR FOR select * from pg_database");
    if (PQresultStatus(res) != PGRES_COMMAND_OK)
    {
        fprintf(stderr, "DECLARE CURSOR failed: %s",
                PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }
    PQclear(res);

    res = PQexec(conn, "FETCH ALL in myportal");
    if (PQresultStatus(res) != PGRES_TUPLES_OK)
    {
        fprintf(stderr, "FETCH ALL failed: %s", PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }

    /* Zuerst die Attributnamen ausgeben. */
    nFields = PQnfields(res);
    for (i = 0; i < nFields; i++)
        printf("%-15s", PQfname(res, i));
    printf("\n\n");

    /* Danach die Zeilen ausgeben. */
    for (i = 0; i < PQntuples(res); i++)
    {
        for (j = 0; j < nFields; j++)
            printf("%-15s", PQgetvalue(res, i, j));
        printf("\n");
    }

    PQclear(res);

    /* Portal schließen; Fehler werden hier nicht geprüft. */
    res = PQexec(conn, "CLOSE myportal");
    PQclear(res);

    /* Transaktion beenden. */
    res = PQexec(conn, "END");
    PQclear(res);

    /* Verbindung schließen und aufräumen. */
    PQfinish(conn);

    return 0;
}
```

**Beispiel 32.2. `libpq`-Beispielprogramm 2**

```c
/*
 * src/test/examples/testlibpq2.c
 *
 * testlibpq2.c
 *      Test der Schnittstelle für asynchrone Benachrichtigungen.
 *
 * Dieses Programm starten und dann in einem zweiten Fenster in psql ausführen:
 *   NOTIFY TBL2;
 * Nach vier Wiederholungen beendet sich das Programm.
 *
 * Alternativ kann eine Datenbank mit den folgenden Befehlen vorbereitet werden
 * (auch in src/test/examples/testlibpq2.sql enthalten):
 *
 *   CREATE SCHEMA TESTLIBPQ2;
 *   SET search_path = TESTLIBPQ2;
 *   CREATE TABLE TBL1 (i int4);
 *   CREATE TABLE TBL2 (i int4);
 *   CREATE RULE r1 AS ON INSERT TO TBL1 DO
 *     (INSERT INTO TBL2 VALUES (new.i); NOTIFY TBL2);
 *
 * Danach dieses Programm starten und in psql viermal ausführen:
 *
 *   INSERT INTO TESTLIBPQ2.TBL1 VALUES (10);
 */

#ifdef WIN32
#include <windows.h>
#endif

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>

#include "libpq-fe.h"

static void
exit_nicely(PGconn *conn)
{
    PQfinish(conn);
    exit(1);
}

int
main(int argc, char **argv)
{
    const char *conninfo;
    PGconn     *conn;
    PGresult   *res;
    PGnotify   *notify;
    int         nnotifies;

    /*
     * Wenn ein Befehlszeilenparameter angegeben wurde, verwende ihn als
     * conninfo-String. Andernfalls wird dbname=postgres gesetzt und alle
     * weiteren Verbindungsparameter kommen aus Umgebungsvariablen oder
     * Standardwerten.
     */
    if (argc > 1)
        conninfo = argv[1];
    else
        conninfo = "dbname = postgres";

    /* Verbindung zur Datenbank herstellen. */
    conn = PQconnectdb(conninfo);

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

    /* PGresult immer freigeben, sobald es nicht mehr benötigt wird. */
    PQclear(res);

    /* LISTEN aktivieren, damit NOTIFY-Meldungen der Regel empfangen werden. */
    res = PQexec(conn, "LISTEN TBL2");
    if (PQresultStatus(res) != PGRES_COMMAND_OK)
    {
        fprintf(stderr, "LISTEN command failed: %s",
                PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }
    PQclear(res);

    /* Nach vier Benachrichtigungen beenden. */
    nnotifies = 0;
    while (nnotifies < 4)
    {
        /*
         * Warten, bis auf der Verbindung etwas passiert. Dieses Beispiel
         * verwendet select(2); poll() oder ähnliche Mechanismen wären auch
         * möglich.
         */
        int         sock;
        fd_set      input_mask;

        sock = PQsocket(conn);

        if (sock < 0)
            break;                  /* sollte nicht passieren */

        FD_ZERO(&input_mask);
        FD_SET(sock, &input_mask);

        if (select(sock + 1, &input_mask, NULL, NULL, NULL) < 0)
        {
            fprintf(stderr, "select() failed: %s\n", strerror(errno));
            exit_nicely(conn);
        }

        /* Jetzt die Eingabe prüfen. */
        PQconsumeInput(conn);
        while ((notify = PQnotifies(conn)) != NULL)
        {
            fprintf(stderr,
                    "ASYNC NOTIFY of '%s' received from backend PID %d\n",
                    notify->relname, notify->be_pid);
            PQfreemem(notify);
            nnotifies++;
            PQconsumeInput(conn);
        }
    }

    fprintf(stderr, "Done.\n");

    /* Verbindung schließen und aufräumen. */
    PQfinish(conn);

    return 0;
}
```

**Beispiel 32.3. `libpq`-Beispielprogramm 3**

```c
/*
 * src/test/examples/testlibpq3.c
 *
 * testlibpq3.c
 *      Test von Out-of-line-Parametern und binärer Ein-/Ausgabe.
 *
 * Vor dem Ausführen eine Datenbank mit folgenden Befehlen vorbereiten
 * (auch in src/test/examples/testlibpq3.sql enthalten):
 *
 * CREATE SCHEMA testlibpq3;
 * SET search_path = testlibpq3;
 * SET standard_conforming_strings = ON;
 * CREATE TABLE test1 (i int4, t text, b bytea);
 * INSERT INTO test1 values (1, 'joe''s place', '\000\001\002\003\004');
 * INSERT INTO test1 values (2, 'ho there', '\004\003\002\001\000');
 *
 * Die erwartete Ausgabe ist:
 *
 * tuple 0: got
 * i = (4 bytes) 1
 * t = (11 bytes) 'joe's place'
 * b = (5 bytes) \000\001\002\003\004
 *
 * tuple 0: got
 * i = (4 bytes) 2
 * t = (8 bytes) 'ho there'
 * b = (5 bytes) \004\003\002\001\000
 */

#ifdef WIN32
#include <windows.h>
#endif

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <sys/types.h>
#include "libpq-fe.h"

/* Für ntohl/htonl. */
#include <netinet/in.h>
#include <arpa/inet.h>

static void
exit_nicely(PGconn *conn)
{
    PQfinish(conn);
    exit(1);
}

/*
 * Diese Funktion gibt ein Abfrageergebnis aus, das als binärer Fetch aus der
 * oben beschriebenen Tabelle stammt. Sie ist ausgelagert, weil main() sie
 * zweimal verwendet.
 */
static void
show_binary_results(PGresult *res)
{
    int         i,
                j;
    int         i_fnum,
                t_fnum,
                b_fnum;

    /* PQfnumber vermeidet Annahmen über die Reihenfolge der Felder. */
    i_fnum = PQfnumber(res, "i");
    t_fnum = PQfnumber(res, "t");
    b_fnum = PQfnumber(res, "b");

    for (i = 0; i < PQntuples(res); i++)
    {
        char       *iptr;
        char       *tptr;
        char       *bptr;
        int         blen;
        int         ival;

        /* Feldwerte abrufen; mögliche NULL-Werte werden hier ignoriert. */
        iptr = PQgetvalue(res, i, i_fnum);
        tptr = PQgetvalue(res, i, t_fnum);
        bptr = PQgetvalue(res, i, b_fnum);

        /*
         * Die binäre Darstellung von INT4 liegt in Netzwerk-Byte-Reihenfolge
         * vor und wird daher in die lokale Byte-Reihenfolge umgewandelt.
         */
        ival = ntohl(*((uint32_t *) iptr));

        /*
         * Die binäre Darstellung von TEXT ist Text. Da libpq ein Nullbyte
         * anhängt, kann der Wert hier als C-Zeichenkette verwendet werden.
         *
         * BYTEA ist eine Bytefolge, die eingebettete Nullbytes enthalten kann;
         * deshalb muss die Feldlänge beachtet werden.
         */
        blen = PQgetlength(res, i, b_fnum);

        printf("tuple %d: got\n", i);
        printf(" i = (%d bytes) %d\n",
               PQgetlength(res, i, i_fnum), ival);
        printf(" t = (%d bytes) '%s'\n",
               PQgetlength(res, i, t_fnum), tptr);
        printf(" b = (%d bytes) ", blen);
        for (j = 0; j < blen; j++)
            printf("\\%03o", bptr[j]);
        printf("\n\n");
    }
}

int
main(int argc, char **argv)
{
    const char *conninfo;
    PGconn     *conn;
    PGresult   *res;
    const char *paramValues[1];
    int         paramLengths[1];
    int         paramFormats[1];
    uint32_t    binaryIntVal;

    /*
     * Wenn ein Befehlszeilenparameter angegeben wurde, verwende ihn als
     * conninfo-String. Andernfalls wird dbname=postgres gesetzt und alle
     * weiteren Verbindungsparameter kommen aus Umgebungsvariablen oder
     * Standardwerten.
     */
    if (argc > 1)
        conninfo = argv[1];
    else
        conninfo = "dbname = postgres";

    /* Verbindung zur Datenbank herstellen. */
    conn = PQconnectdb(conninfo);

    /* Prüfen, ob die Verbindung erfolgreich hergestellt wurde. */
    if (PQstatus(conn) != CONNECTION_OK)
    {
        fprintf(stderr, "%s", PQerrorMessage(conn));
        exit_nicely(conn);
    }

    /* Sicheren Suchpfad setzen, damit keine fremden Objekte bevorzugt werden. */
    res = PQexec(conn, "SET search_path = testlibpq3");
    if (PQresultStatus(res) != PGRES_COMMAND_OK)
    {
        fprintf(stderr, "SET failed: %s", PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }
    PQclear(res);

    /*
     * Dieses Programm zeigt die Verwendung von PQexecParams() mit
     * Out-of-line-Parametern sowie die binäre Übertragung von Daten.
     *
     * Das erste Beispiel sendet Parameter als Text, empfängt die Ergebnisse
     * aber im Binärformat. Durch Out-of-line-Parameter entfällt mühsames
     * Quoting und Escaping, obwohl die Daten Text sind.
     */

    /* Out-of-line-Parameterwert. */
    paramValues[0] = "joe's place";

    res = PQexecParams(conn,
                       "SELECT * FROM test1 WHERE t = $1",
                       1,       /* ein Parameter */
                       NULL,    /* Backend leitet den Parametertyp ab */
                       paramValues,
                       NULL,    /* bei Textparametern keine Längen nötig */
                       NULL,    /* standardmäßig alles Textparameter */
                       1);      /* Ergebnisse im Binärformat anfordern */

    if (PQresultStatus(res) != PGRES_TUPLES_OK)
    {
        fprintf(stderr, "SELECT failed: %s", PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }

    show_binary_results(res);

    PQclear(res);

    /*
     * Im zweiten Beispiel senden wir einen Integer-Parameter in Binärform
     * und empfangen die Ergebnisse ebenfalls binär.
     *
     * Obwohl PQexecParams den Parametertyp vom Backend ableiten lässt,
     * erzwingen wir die Entscheidung durch einen Cast im Abfragetext. Das ist
     * beim Senden binärer Parameter eine sinnvolle Sicherheitsmaßnahme.
     */

    /* Integerwert "2" in Netzwerk-Byte-Reihenfolge umwandeln. */
    binaryIntVal = htonl((uint32_t) 2);

    /* Parameterarrays für PQexecParams vorbereiten. */
    paramValues[0] = (char *) &binaryIntVal;
    paramLengths[0] = sizeof(binaryIntVal);
    paramFormats[0] = 1;        /* binary */

    res = PQexecParams(conn,
                       "SELECT * FROM test1 WHERE i = $1::int4",
                       1,       /* ein Parameter */
                       NULL,    /* Backend leitet den Parametertyp ab */
                       paramValues,
                       paramLengths,
                       paramFormats,
                       1);      /* Ergebnisse im Binärformat anfordern */

    if (PQresultStatus(res) != PGRES_TUPLES_OK)
    {
        fprintf(stderr, "SELECT failed: %s", PQerrorMessage(conn));
        PQclear(res);
        exit_nicely(conn);
    }

    show_binary_results(res);

    PQclear(res);

    /* Verbindung schließen und aufräumen. */
    PQfinish(conn);

    return 0;
}
```
