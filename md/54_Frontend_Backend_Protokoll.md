# 54. Frontend-/Backend-Protokoll

PostgreSQL verwendet für die Kommunikation zwischen Frontends und Backends, also Clients und Servern, ein nachrichtenbasiertes Protokoll. Das Protokoll wird über TCP/IP und auch über Unix-Domain-Sockets unterstützt. Die Portnummer 5432 ist bei der IANA als übliche TCP-Portnummer für Server registriert, die dieses Protokoll unterstützen; in der Praxis kann jedoch jede nicht privilegierte Portnummer verwendet werden.

Dieses Dokument beschreibt Version 3.2 des Protokolls, die in PostgreSQL Version 18 eingeführt wurde. Der Server und die Client-Bibliothek `libpq` sind abwärtskompatibel mit Protokollversion 3.0, die in PostgreSQL 7.4 und später implementiert ist.

Um mehrere Clients effizient zu bedienen, startet der Server für jeden Client einen neuen „Backend“-Prozess. In der aktuellen Implementierung wird unmittelbar nach dem Erkennen einer eingehenden Verbindung ein neuer Kindprozess erzeugt. Für das Protokoll ist dies jedoch transparent. Für die Zwecke des Protokolls sind die Begriffe „Backend“ und „Server“ austauschbar; ebenso sind „Frontend“ und „Client“ austauschbar.

## 54.1. Überblick

Das Protokoll hat getrennte Phasen für den Start und den normalen Betrieb. In der Startphase öffnet das Frontend eine Verbindung zum Server und authentifiziert sich zur Zufriedenheit des Servers. Dies kann je nach verwendeter Authentifizierungsmethode eine einzelne Nachricht oder mehrere Nachrichten umfassen. Wenn alles gut geht, sendet der Server anschließend Statusinformationen an das Frontend und geht schließlich in den normalen Betrieb über. Abgesehen von der anfänglichen Startup-Request-Nachricht wird dieser Teil des Protokolls vom Server gesteuert.

Während des normalen Betriebs sendet das Frontend Abfragen und andere Kommandos an das Backend, und das Backend sendet Abfrageergebnisse und andere Antworten zurück. Es gibt einige Fälle, etwa `NOTIFY`, in denen das Backend unaufgeforderte Nachrichten sendet; der größte Teil dieses Sitzungsabschnitts wird jedoch von Frontend-Anforderungen gesteuert.

Die Beendigung der Sitzung erfolgt normalerweise nach Wahl des Frontends, kann aber in bestimmten Fällen vom Backend erzwungen werden. In jedem Fall rollt das Backend jede offene, unvollständige Transaktion zurück, bevor es beendet wird, wenn es die Verbindung schließt.

Innerhalb des normalen Betriebs können SQL-Kommandos über eines von zwei Unterprotokollen ausgeführt werden. Im „Simple Query“-Protokoll sendet das Frontend einfach eine textuelle Abfragezeichenkette, die vom Backend geparst und unmittelbar ausgeführt wird. Im „Extended Query“-Protokoll wird die Verarbeitung von Abfragen in mehrere Schritte aufgeteilt: Parsen, Binden von Parameterwerten und Ausführen. Dies bietet Flexibilitäts- und Leistungsvorteile, allerdings um den Preis zusätzlicher Komplexität.

Der normale Betrieb hat zusätzliche Unterprotokolle für besondere Operationen wie `COPY`.

### 54.1.1. Überblick über Nachrichten

Die gesamte Kommunikation erfolgt über einen Nachrichtenstrom. Das erste Byte einer Nachricht identifiziert den Nachrichtentyp, und die nächsten vier Bytes geben die Länge des übrigen Teils der Nachricht an. Diese Längenangabe schließt sich selbst ein, aber nicht das Byte für den Nachrichtentyp. Der verbleibende Inhalt der Nachricht wird durch den Nachrichtentyp bestimmt. Aus historischen Gründen hat die allererste vom Client gesendete Nachricht, die Startup-Nachricht, kein anfängliches Nachrichtentyp-Byte.

Um die Synchronisierung mit dem Nachrichtenstrom nicht zu verlieren, lesen sowohl Server als auch Clients typischerweise eine ganze Nachricht in einen Puffer, wobei sie die Byteanzahl verwenden, bevor sie versuchen, ihren Inhalt zu verarbeiten. Das ermöglicht eine einfache Wiederherstellung, wenn beim Verarbeiten des Inhalts ein Fehler erkannt wird. In extremen Situationen, etwa wenn nicht genug Speicher zum Puffern der Nachricht vorhanden ist, kann der Empfänger anhand der Byteanzahl bestimmen, wie viel Eingabe er überspringen muss, bevor er wieder Nachrichten liest.

Umgekehrt müssen sowohl Server als auch Clients darauf achten, niemals eine unvollständige Nachricht zu senden. Dies geschieht üblicherweise, indem die gesamte Nachricht in einem Puffer marshaled wird, bevor mit dem Senden begonnen wird. Wenn während des Sendens oder Empfangens einer Nachricht ein Kommunikationsfehler auftritt, ist die einzig sinnvolle Reaktion, die Verbindung aufzugeben, da kaum Hoffnung besteht, die Synchronisierung der Nachrichtengrenzen wiederherzustellen.

### 54.1.2. Überblick über Extended Query

Im Extended-Query-Protokoll wird die Ausführung von SQL-Kommandos in mehrere Schritte aufgeteilt. Der zwischen den Schritten beibehaltene Zustand wird durch zwei Arten von Objekten dargestellt: vorbereitete Anweisungen und Portale. Eine vorbereitete Anweisung stellt das Ergebnis des Parsens und der semantischen Analyse einer textuellen Abfragezeichenkette dar. Eine vorbereitete Anweisung ist für sich genommen noch nicht ausführungsbereit, weil ihr konkrete Werte für Parameter fehlen können. Ein Portal stellt eine ausführungsbereite oder bereits teilweise ausgeführte Anweisung dar, bei der alle fehlenden Parameterwerte eingesetzt sind. Bei `SELECT`-Anweisungen entspricht ein Portal einem offenen Cursor; wir verwenden jedoch einen anderen Begriff, da Cursor keine Nicht-`SELECT`-Anweisungen behandeln.

Der gesamte Ausführungszyklus besteht aus einem Parse-Schritt, der aus einer textuellen Abfragezeichenkette eine vorbereitete Anweisung erzeugt; einem Bind-Schritt, der aus einer vorbereiteten Anweisung und Werten für alle benötigten Parameter ein Portal erzeugt; und einem Execute-Schritt, der die Abfrage eines Portals ausführt. Bei einer Abfrage, die Zeilen zurückgibt, etwa `SELECT`, `SHOW` usw., kann dem Execute-Schritt mitgeteilt werden, nur eine begrenzte Anzahl von Zeilen abzurufen, sodass mehrere Execute-Schritte nötig sein können, um die Operation abzuschließen.

Das Backend kann mehrere vorbereitete Anweisungen und Portale verwalten. Beachten Sie jedoch, dass diese nur innerhalb einer Sitzung existieren und niemals sitzungsübergreifend geteilt werden. Vorhandene vorbereitete Anweisungen und Portale werden über Namen referenziert, die ihnen bei ihrer Erstellung zugewiesen wurden. Zusätzlich existieren eine „unbenannte“ vorbereitete Anweisung und ein „unbenanntes“ Portal. Obwohl diese sich weitgehend wie benannte Objekte verhalten, sind Operationen auf ihnen für den Fall optimiert, dass eine Abfrage nur einmal ausgeführt und anschließend verworfen wird, während Operationen auf benannten Objekten für die Erwartung mehrfacher Verwendung optimiert sind.

### 54.1.3. Formate und Format-Codes

Daten eines bestimmten Datentyps können in mehreren verschiedenen Formaten übertragen werden. Seit PostgreSQL 7.4 werden nur die Formate „text“ und „binary“ unterstützt, das Protokoll sieht jedoch Erweiterungen für die Zukunft vor. Das gewünschte Format für jeden Wert wird durch einen Format-Code angegeben. Clients können für jeden übertragenen Parameterwert und für jede Spalte eines Abfrageergebnisses einen Format-Code angeben. Text hat den Format-Code null, Binärdaten haben den Format-Code eins, und alle anderen Format-Codes sind für zukünftige Definitionen reserviert.

Die Textdarstellung von Werten besteht aus den Zeichenketten, die von den Ein-/Ausgabekonvertierungsfunktionen des jeweiligen Datentyps erzeugt und akzeptiert werden. In der übertragenen Darstellung gibt es kein abschließendes Nullzeichen; das Frontend muss empfangenen Werten eines hinzufügen, wenn es sie als C-Strings verarbeiten möchte. Das Textformat erlaubt übrigens keine eingebetteten Nullen.

Binärdarstellungen für Ganzzahlen verwenden Netzwerk-Byte-Reihenfolge, also das höchstwertige Byte zuerst. Für andere Datentypen konsultieren Sie die Dokumentation oder den Quellcode, um etwas über die Binärdarstellung zu erfahren. Beachten Sie, dass sich Binärdarstellungen komplexer Datentypen zwischen Serverversionen ändern können; das Textformat ist normalerweise die portablere Wahl.

### 54.1.4. Protokollversionen

Die aktuelle, neueste Version des Protokolls ist Version 3.2. Aus Gründen der Abwärtskompatibilität mit alten Serverversionen und Middleware, die die Versionsaushandlung noch nicht unterstützt, verwendet `libpq` standardmäßig jedoch weiterhin Protokollversion 3.0.

Ein einzelner Server kann mehrere Protokollversionen unterstützen. Die anfängliche Startup-Request-Nachricht teilt dem Server mit, welche Protokollversion der Client zu verwenden versucht. Wenn die vom Client angeforderte Hauptversion vom Server nicht unterstützt wird, wird die Verbindung abgewiesen; dies würde beispielsweise geschehen, wenn der Client Protokollversion 4.0 anfordert, die zum Zeitpunkt dieser Beschreibung nicht existiert. Wenn die vom Client angeforderte Nebenversion vom Server nicht unterstützt wird, etwa wenn der Client Version 3.2 anfordert, der Server aber nur 3.0 unterstützt, kann der Server die Verbindung entweder abweisen oder mit einer `NegotiateProtocolVersion`-Nachricht antworten, die die höchste von ihm unterstützte Nebenprotokollversion enthält. Der Client kann dann wählen, ob er die Verbindung mit der angegebenen Protokollversion fortsetzt oder abbricht.

Die Protokollaushandlung wurde in PostgreSQL Version 9.3.21 eingeführt. Frühere Versionen wiesen die Verbindung ab, wenn der Client eine Nebenversion anforderte, die vom Server nicht unterstützt wurde.

Tabelle 54.1 zeigt die derzeit unterstützten Protokollversionen.

**Tabelle 54.1. Protokollversionen**

| Version | Unterstützt von | Beschreibung |
| --- | --- | --- |
| `3.2` | PostgreSQL 18 und später | Aktuelle neueste Version. Der geheime Schlüssel, der beim Abbrechen von Abfragen verwendet wird, wurde von 4 Byte auf ein Feld variabler Länge erweitert. Die Nachricht `BackendKeyData` wurde entsprechend geändert, und die Nachricht `CancelRequest` wurde neu definiert, sodass sie eine Nutzlast variabler Länge hat. |
| `3.1` | - | Reserviert. Version 3.1 wurde von keiner PostgreSQL-Version verwendet, aber übersprungen, weil alte Versionen der beliebten Anwendung `pgbouncer` einen Fehler in der Protokollaushandlung hatten, durch den sie fälschlich behaupteten, Version 3.1 zu unterstützen. |
| `3.0` | PostgreSQL 7.4 und später | |
| `2.0` | bis PostgreSQL 13 | Details finden Sie in früheren Ausgaben der PostgreSQL-Dokumentation. |

## 54.2. Nachrichtenfluss

Dieser Abschnitt beschreibt den Nachrichtenfluss und die Semantik jedes Nachrichtentyps. Details zur genauen Darstellung jeder Nachricht finden Sie in [Abschnitt 54.7](54_Frontend_Backend_Protokoll.md#547-nachrichtenformate). Abhängig vom Zustand der Verbindung gibt es mehrere unterschiedliche Unterprotokolle: Start, Abfrage, Funktionsaufruf, `COPY` und Beendigung. Außerdem gibt es besondere Vorkehrungen für asynchrone Operationen, einschließlich Benachrichtigungsantworten und Abfrageabbruch, die jederzeit nach der Startphase auftreten können.

### 54.2.1. Start

Um eine Sitzung zu beginnen, öffnet ein Frontend eine Verbindung zum Server und sendet eine Startup-Nachricht. Diese Nachricht enthält die Namen des Benutzers und der Datenbank, zu der der Benutzer eine Verbindung herstellen möchte; außerdem identifiziert sie die konkrete zu verwendende Protokollversion. Optional kann die Startup-Nachricht zusätzliche Einstellungen für Laufzeitparameter enthalten. Der Server verwendet diese Informationen und die Inhalte seiner Konfigurationsdateien, etwa `pg_hba.conf`, um zu bestimmen, ob die Verbindung vorläufig akzeptabel ist und welche zusätzliche Authentifizierung, falls überhaupt, erforderlich ist.

Der Server sendet dann eine passende Authentifizierungsanforderungsnachricht, auf die das Frontend mit einer passenden Authentifizierungsantwortnachricht antworten muss, etwa mit einem Passwort. Für alle Authentifizierungsmethoden außer GSSAPI, SSPI und SASL gibt es höchstens eine Anforderung und eine Antwort. Bei einigen Methoden wird überhaupt keine Antwort vom Frontend benötigt, sodass keine Authentifizierungsanforderung erfolgt. Für GSSAPI, SSPI und SASL können mehrere Paketaustausche nötig sein, um die Authentifizierung abzuschließen.

Der Authentifizierungszyklus endet damit, dass der Server entweder den Verbindungsversuch ablehnt (`ErrorResponse`) oder `AuthenticationOk` sendet.

Die möglichen Nachrichten vom Server in dieser Phase sind:

- `ErrorResponse`
  - Der Verbindungsversuch wurde abgelehnt. Der Server schließt die Verbindung anschließend sofort.

- `AuthenticationOk`
  - Der Authentifizierungsaustausch wurde erfolgreich abgeschlossen.

- `AuthenticationKerberosV5`
  - Das Frontend muss nun mit dem Server an einem Kerberos-V5-Authentifizierungsdialog teilnehmen, der hier nicht beschrieben wird und Teil der Kerberos-Spezifikation ist. Wenn dies erfolgreich ist, antwortet der Server mit `AuthenticationOk`, andernfalls mit `ErrorResponse`. Dies wird nicht mehr unterstützt.

- `AuthenticationCleartextPassword`
  - Das Frontend muss nun eine `PasswordMessage` senden, die das Passwort im Klartext enthält. Wenn dies das richtige Passwort ist, antwortet der Server mit `AuthenticationOk`, andernfalls mit `ErrorResponse`.

- `AuthenticationMD5Password`
  - Das Frontend muss nun eine `PasswordMessage` senden, die das Passwort zusammen mit dem Benutzernamen per MD5 verschlüsselt enthält und anschließend erneut mit dem in der Nachricht `AuthenticationMD5Password` angegebenen zufälligen 4-Byte-Salt verschlüsselt wurde. Wenn dies das richtige Passwort ist, antwortet der Server mit `AuthenticationOk`, andernfalls mit `ErrorResponse`. Die eigentliche `PasswordMessage` kann in SQL als `concat('md5', md5(concat(md5(concat(password, username)), random-salt)))` berechnet werden. Beachten Sie, dass die Funktion `md5()` ihr Ergebnis als Hex-Zeichenkette zurückgibt.

> **Warnung**
>
> Die Unterstützung für MD5-verschlüsselte Passwörter ist veraltet und wird in einer zukünftigen PostgreSQL-Version entfernt. Details zur Migration auf einen anderen Passworttyp finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

- `AuthenticationGSS`
  - Das Frontend muss nun eine GSSAPI-Aushandlung initiieren. Als Antwort darauf sendet das Frontend eine `GSSResponse`-Nachricht mit dem ersten Teil des GSSAPI-Datenstroms. Wenn weitere Nachrichten benötigt werden, antwortet der Server mit `AuthenticationGSSContinue`.

- `AuthenticationSSPI`
  - Das Frontend muss nun eine SSPI-Aushandlung initiieren. Als Antwort darauf sendet das Frontend eine `GSSResponse` mit dem ersten Teil des SSPI-Datenstroms. Wenn weitere Nachrichten benötigt werden, antwortet der Server mit `AuthenticationGSSContinue`.

- `AuthenticationGSSContinue`
  - Diese Nachricht enthält die Antwortdaten aus dem vorherigen Schritt der GSSAPI- oder SSPI-Aushandlung, also aus `AuthenticationGSS`, `AuthenticationSSPI` oder einem vorherigen `AuthenticationGSSContinue`. Wenn die GSSAPI- oder SSPI-Daten in dieser Nachricht anzeigen, dass zum Abschluss der Authentifizierung weitere Daten benötigt werden, muss das Frontend diese Daten als weitere `GSSResponse`-Nachricht senden. Wenn die GSSAPI- oder SSPI-Authentifizierung mit dieser Nachricht abgeschlossen ist, sendet der Server als Nächstes `AuthenticationOk`, um erfolgreiche Authentifizierung anzuzeigen, oder `ErrorResponse`, um einen Fehler anzuzeigen.

- `AuthenticationSASL`
  - Das Frontend muss nun eine SASL-Aushandlung initiieren und dabei einen der in der Nachricht aufgeführten SASL-Mechanismen verwenden. Als Antwort darauf sendet das Frontend eine `SASLInitialResponse` mit dem Namen des ausgewählten Mechanismus und dem ersten Teil des SASL-Datenstroms. Wenn weitere Nachrichten benötigt werden, antwortet der Server mit `AuthenticationSASLContinue`. Details finden Sie in [Abschnitt 54.3](54_Frontend_Backend_Protokoll.md#543-saslauthentifizierung).

- `AuthenticationSASLContinue`
  - Diese Nachricht enthält Challenge-Daten aus dem vorherigen Schritt der SASL-Aushandlung, also aus `AuthenticationSASL` oder einem vorherigen `AuthenticationSASLContinue`. Das Frontend muss mit einer `SASLResponse`-Nachricht antworten.

- `AuthenticationSASLFinal`
  - Die SASL-Authentifizierung wurde mit zusätzlichen mechanismusspezifischen Daten für den Client abgeschlossen. Der Server sendet als Nächstes `AuthenticationOk`, um erfolgreiche Authentifizierung anzuzeigen, oder `ErrorResponse`, um einen Fehler anzuzeigen. Diese Nachricht wird nur gesendet, wenn der SASL-Mechanismus zusätzliche Daten vorsieht, die beim Abschluss vom Server an den Client gesendet werden.

- `NegotiateProtocolVersion`
  - Der Server unterstützt die vom Client angeforderte Nebenprotokollversion nicht, unterstützt aber eine frühere Version des Protokolls; diese Nachricht gibt die höchste unterstützte Nebenversion an. Diese Nachricht wird auch gesendet, wenn der Client nicht unterstützte Protokolloptionen, also solche, die mit `_pq_` beginnen, im Startup-Paket angefordert hat.
  - Nach dieser Nachricht wird die Authentifizierung mit der vom Server angegebenen Version fortgesetzt. Wenn der Client die ältere Version nicht unterstützt, sollte er die Verbindung sofort schließen. Wenn der Server diese Nachricht nicht sendet, unterstützt er die vom Client angeforderte Protokollversion und alle Protokolloptionen.

Wenn das Frontend die vom Server angeforderte Authentifizierungsmethode nicht unterstützt, sollte es die Verbindung sofort schließen.

Nachdem es `AuthenticationOk` empfangen hat, muss das Frontend auf weitere Nachrichten vom Server warten. In dieser Phase wird ein Backend-Prozess gestartet, und das Frontend ist nur ein interessierter Beobachter. Es ist weiterhin möglich, dass der Startversuch fehlschlägt (`ErrorResponse`) oder dass der Server die Unterstützung der angeforderten Nebenprotokollversion ablehnt (`NegotiateProtocolVersion`); im Normalfall sendet das Backend jedoch einige `ParameterStatus`-Nachrichten, `BackendKeyData` und schließlich `ReadyForQuery`.

Während dieser Phase versucht das Backend, alle zusätzlichen Laufzeitparametereinstellungen anzuwenden, die in der Startup-Nachricht angegeben wurden. Bei Erfolg werden diese Werte zu Sitzungs-Standardwerten. Ein Fehler verursacht `ErrorResponse` und beendet die Verbindung.

Die möglichen Nachrichten vom Backend in dieser Phase sind:

- `BackendKeyData`
  - Diese Nachricht stellt geheime Schlüsseldaten bereit, die das Frontend speichern muss, wenn es später Abbruchanforderungen ausgeben können soll. Das Frontend sollte nicht auf diese Nachricht antworten, sondern weiter auf eine `ReadyForQuery`-Nachricht warten.
  - Der PostgreSQL-Server sendet diese Nachricht immer, aber es ist bekannt, dass einige Backend-Implementierungen des Protokolls von Drittanbietern, die keinen Abfrageabbruch unterstützen, dies nicht tun.

- `ParameterStatus`
  - Diese Nachricht informiert das Frontend über die aktuelle, anfängliche Einstellung von Backend-Parametern wie `client_encoding` oder `DateStyle`. Das Frontend kann diese Nachricht ignorieren oder die Einstellungen für spätere Verwendung aufzeichnen; weitere Details finden Sie in Abschnitt 54.2.7. Das Frontend sollte nicht auf diese Nachricht antworten, sondern weiter auf eine `ReadyForQuery`-Nachricht warten.

- `ReadyForQuery`
  - Der Start ist abgeschlossen. Das Frontend kann nun Kommandos ausgeben.

- `ErrorResponse`
  - Der Start ist fehlgeschlagen. Die Verbindung wird nach dem Senden dieser Nachricht geschlossen.

- `NoticeResponse`
  - Eine Warnmeldung wurde ausgegeben. Das Frontend sollte die Nachricht anzeigen, aber weiter auf `ReadyForQuery` oder `ErrorResponse` warten.

Die Nachricht `ReadyForQuery` ist dieselbe, die das Backend nach jedem Kommandozyklus ausgibt. Je nach Codierungsbedarf des Frontends ist es vernünftig, `ReadyForQuery` als Beginn eines Kommandozyklus oder als Ende der Startphase und jedes nachfolgenden Kommandozyklus zu betrachten.

### 54.2.2. Einfache Abfrage

Ein einfacher Abfragezyklus wird dadurch eingeleitet, dass das Frontend eine `Query`-Nachricht an das Backend sendet. Die Nachricht enthält ein SQL-Kommando oder mehrere SQL-Kommandos als Textzeichenkette. Das Backend sendet dann je nach Inhalt der Abfragekommando-Zeichenkette eine oder mehrere Antwortnachrichten und schließlich eine `ReadyForQuery`-Antwortnachricht. `ReadyForQuery` informiert das Frontend, dass es gefahrlos ein neues Kommando senden kann. Es ist nicht wirklich erforderlich, dass das Frontend auf `ReadyForQuery` wartet, bevor es ein weiteres Kommando ausgibt; das Frontend muss dann aber selbst dafür verantwortlich sein herauszufinden, was geschieht, wenn das frühere Kommando fehlschlägt und bereits ausgegebene spätere Kommandos erfolgreich sind.

Die möglichen Antwortnachrichten vom Backend sind:

- `CommandComplete`
  - Ein SQL-Kommando wurde normal abgeschlossen.

- `CopyInResponse`
  - Das Backend ist bereit, Daten vom Frontend in eine Tabelle zu kopieren; siehe Abschnitt 54.2.6.

- `CopyOutResponse`
  - Das Backend ist bereit, Daten aus einer Tabelle zum Frontend zu kopieren; siehe Abschnitt 54.2.6.

- `RowDescription`
  - Zeigt an, dass als Antwort auf eine `SELECT`-, `FETCH`- usw. Abfrage Zeilen zurückgegeben werden. Der Inhalt dieser Nachricht beschreibt das Spaltenlayout der Zeilen. Darauf folgt für jede an das Frontend zurückgegebene Zeile eine `DataRow`-Nachricht.

- `DataRow`
  - Eine der Zeilen aus der Ergebnismenge einer `SELECT`-, `FETCH`- usw. Abfrage.

- `EmptyQueryResponse`
  - Eine leere Abfragezeichenkette wurde erkannt.

- `ErrorResponse`
  - Ein Fehler ist aufgetreten.

- `ReadyForQuery`
  - Die Verarbeitung der Abfragezeichenkette ist abgeschlossen. Dafür wird eine eigene Nachricht gesendet, weil die Abfragezeichenkette mehrere SQL-Kommandos enthalten kann. `CommandComplete` markiert das Ende der Verarbeitung eines SQL-Kommandos, nicht der gesamten Zeichenkette. `ReadyForQuery` wird immer gesendet, unabhängig davon, ob die Verarbeitung erfolgreich oder mit einem Fehler endet.

- `NoticeResponse`
  - Im Zusammenhang mit der Abfrage wurde eine Warnmeldung ausgegeben. Notices kommen zusätzlich zu anderen Antworten; das Backend setzt also die Verarbeitung des Kommandos fort.

Die Antwort auf eine `SELECT`-Abfrage oder andere Abfragen, die Zeilenmengen zurückgeben, etwa `EXPLAIN` oder `SHOW`, besteht normalerweise aus `RowDescription`, null oder mehr `DataRow`-Nachrichten und anschließend `CommandComplete`. `COPY` vom oder zum Frontend löst ein spezielles Protokoll aus, wie in Abschnitt 54.2.6 beschrieben. Alle anderen Abfragetypen erzeugen normalerweise nur eine `CommandComplete`-Nachricht.

Da eine Abfragezeichenkette mehrere durch Semikolons getrennte Abfragen enthalten kann, kann es mehrere solche Antwortsequenzen geben, bevor das Backend die Verarbeitung der Abfragezeichenkette beendet. `ReadyForQuery` wird ausgegeben, wenn die gesamte Zeichenkette verarbeitet wurde und das Backend bereit ist, eine neue Abfragezeichenkette anzunehmen.

Wenn eine vollständig leere Abfragezeichenkette empfangen wird, also ohne Inhalt außer Leerraum, besteht die Antwort aus `EmptyQueryResponse`, gefolgt von `ReadyForQuery`.

Im Fehlerfall wird `ErrorResponse` ausgegeben, gefolgt von `ReadyForQuery`. Die gesamte weitere Verarbeitung der Abfragezeichenkette wird durch `ErrorResponse` abgebrochen, selbst wenn darin weitere Abfragen verblieben sind. Beachten Sie, dass dies mitten in der Sequenz von Nachrichten geschehen kann, die von einer einzelnen Abfrage erzeugt wird.

Im Simple-Query-Modus ist das Format abgerufener Werte immer Text, außer wenn das angegebene Kommando ein `FETCH` aus einem Cursor ist, der mit der Option `BINARY` deklariert wurde. In diesem Fall liegen die abgerufenen Werte im Binärformat vor. Die in der `RowDescription`-Nachricht angegebenen Format-Codes geben an, welches Format verwendet wird.

Ein Frontend muss darauf vorbereitet sein, `ErrorResponse`- und `NoticeResponse`-Nachrichten immer dann zu akzeptieren, wenn es irgendeinen anderen Nachrichtentyp erwartet. Siehe auch Abschnitt 54.2.7 zu Nachrichten, die das Backend aufgrund externer Ereignisse erzeugen kann.

Empfohlen wird, Frontends im Stil einer Zustandsmaschine zu codieren, die jeden Nachrichtentyp zu jeder Zeit akzeptiert, zu der er sinnvoll sein könnte, statt Annahmen über die genaue Reihenfolge von Nachrichten fest zu verdrahten.

#### 54.2.2.1. Mehrere Anweisungen in einer einfachen Abfrage

Wenn eine einfache `Query`-Nachricht mehr als eine SQL-Anweisung enthält, getrennt durch Semikolons, werden diese Anweisungen als eine einzelne Transaktion ausgeführt, sofern keine expliziten Transaktionssteuerungskommandos enthalten sind, die ein anderes Verhalten erzwingen. Wenn die Nachricht beispielsweise Folgendes enthält:

```sql
INSERT INTO mytable VALUES(1);
SELECT 1/0;
INSERT INTO mytable VALUES(2);
```

dann erzwingt der Division-durch-null-Fehler in `SELECT` ein Rollback des ersten `INSERT`. Da die Ausführung der Nachricht außerdem beim ersten Fehler abgebrochen wird, wird der zweite `INSERT` überhaupt nicht versucht.

Wenn die Nachricht stattdessen Folgendes enthält:

```sql
BEGIN;
INSERT INTO mytable VALUES(1);
COMMIT;
INSERT INTO mytable VALUES(2);
SELECT 1/0;
```

dann wird der erste `INSERT` durch das explizite Kommando `COMMIT` committet. Der zweite `INSERT` und das `SELECT` werden weiterhin als eine einzelne Transaktion behandelt, sodass der Division-durch-null-Fehler den zweiten `INSERT` zurückrollt, aber nicht den ersten.

Dieses Verhalten wird implementiert, indem die Anweisungen in einer `Query`-Nachricht mit mehreren Anweisungen in einem impliziten Transaktionsblock ausgeführt werden, sofern es keinen expliziten Transaktionsblock gibt, in dem sie laufen. Der Hauptunterschied zwischen einem impliziten Transaktionsblock und einem regulären Transaktionsblock besteht darin, dass ein impliziter Block am Ende der `Query`-Nachricht automatisch geschlossen wird, entweder durch ein implizites Commit, wenn kein Fehler auftrat, oder durch ein implizites Rollback, wenn ein Fehler auftrat. Dies ähnelt dem impliziten Commit oder Rollback, das bei einer einzeln ausgeführten Anweisung geschieht, wenn sie nicht in einem Transaktionsblock läuft.

Wenn sich die Sitzung bereits infolge eines `BEGIN` in einer früheren Nachricht in einem Transaktionsblock befindet, setzt die `Query`-Nachricht diesen Transaktionsblock einfach fort, unabhängig davon, ob die Nachricht eine oder mehrere Anweisungen enthält. Wenn die `Query`-Nachricht jedoch ein `COMMIT` oder `ROLLBACK` enthält, das den vorhandenen Transaktionsblock schließt, werden alle folgenden Anweisungen in einem impliziten Transaktionsblock ausgeführt. Umgekehrt startet ein `BEGIN`, das in einer `Query`-Nachricht mit mehreren Anweisungen erscheint, einen regulären Transaktionsblock, der nur durch ein explizites `COMMIT` oder `ROLLBACK` beendet wird, unabhängig davon, ob dieses in derselben `Query`-Nachricht oder in einer späteren erscheint. Wenn das `BEGIN` auf einige Anweisungen folgt, die als impliziter Transaktionsblock ausgeführt wurden, werden diese Anweisungen nicht sofort committet; faktisch werden sie nachträglich in den neuen regulären Transaktionsblock einbezogen.

Ein `COMMIT` oder `ROLLBACK`, das in einem impliziten Transaktionsblock erscheint, wird normal ausgeführt und schließt den impliziten Block; es wird jedoch eine Warnung ausgegeben, da ein `COMMIT` oder `ROLLBACK` ohne vorheriges `BEGIN` ein Versehen darstellen könnte. Wenn weitere Anweisungen folgen, wird für sie ein neuer impliziter Transaktionsblock gestartet.

Savepoints sind in einem impliziten Transaktionsblock nicht erlaubt, da sie mit dem Verhalten kollidieren würden, den Block bei jedem Fehler automatisch zu schließen.

Denken Sie daran, dass die Ausführung der `Query`-Nachricht unabhängig von vorhandenen Transaktionssteuerungskommandos beim ersten Fehler stoppt. Bei folgendem Inhalt in einer einzelnen `Query`-Nachricht:

```sql
BEGIN;
SELECT 1/0;
ROLLBACK;
```

bleibt die Sitzung daher innerhalb eines fehlgeschlagenen regulären Transaktionsblocks, weil das `ROLLBACK` nach dem Division-durch-null-Fehler nicht erreicht wird. Ein weiteres `ROLLBACK` ist erforderlich, um die Sitzung wieder in einen verwendbaren Zustand zu bringen.

Ein weiteres bemerkenswertes Verhalten ist, dass die anfängliche lexikalische und syntaktische Analyse für die gesamte Abfragezeichenkette erfolgt, bevor irgendein Teil davon ausgeführt wird. Daher können einfache Fehler, etwa ein falsch geschriebenes Schlüsselwort in späteren Anweisungen, die Ausführung aller Anweisungen verhindern. Das ist für Benutzer normalerweise unsichtbar, da die Anweisungen bei Ausführung als impliziter Transaktionsblock ohnehin alle zurückgerollt würden. Es kann jedoch sichtbar werden, wenn versucht wird, mehrere Transaktionen innerhalb einer `Query` mit mehreren Anweisungen auszuführen. Wenn etwa ein Tippfehler aus dem vorherigen Beispiel Folgendes macht:

```sql
BEGIN;
INSERT INTO mytable VALUES(1);
COMMIT;
INSERT INTO mytable VALUES(2);
SELCT 1/0;
```

dann würde keine der Anweisungen ausgeführt, mit dem sichtbaren Unterschied, dass der erste `INSERT` nicht committet wird. Fehler, die bei der semantischen Analyse oder später erkannt werden, etwa ein falsch geschriebener Tabellen- oder Spaltenname, haben diesen Effekt nicht.

Beachten Sie schließlich, dass alle Anweisungen innerhalb der `Query`-Nachricht denselben Wert von `statement_timestamp()` sehen, da dieser Zeitstempel nur beim Empfang der `Query`-Nachricht aktualisiert wird. Dies führt dazu, dass sie auch alle denselben Wert von `transaction_timestamp()` sehen, außer in Fällen, in denen die Abfragezeichenkette eine zuvor gestartete Transaktion beendet und eine neue beginnt.

### 54.2.3. Erweiterte Abfrage

Das Extended-Query-Protokoll zerlegt das oben beschriebene Simple-Query-Protokoll in mehrere Schritte. Die Ergebnisse vorbereitender Schritte können zur Effizienzsteigerung mehrfach wiederverwendet werden. Außerdem stehen zusätzliche Funktionen zur Verfügung, etwa die Möglichkeit, Datenwerte als getrennte Parameter zu übergeben, statt sie direkt in eine Abfragezeichenkette einzufügen.

Im erweiterten Protokoll sendet das Frontend zunächst eine `Parse`-Nachricht. Diese enthält eine textuelle Abfragezeichenkette, optional einige Informationen über Datentypen von Parameterplatzhaltern und den Namen eines Zielobjekts für eine vorbereitete Anweisung; eine leere Zeichenkette wählt die unbenannte vorbereitete Anweisung aus. Die Antwort ist entweder `ParseComplete` oder `ErrorResponse`. Parametertypen können per OID angegeben werden; wenn sie nicht angegeben sind, versucht der Parser, die Datentypen auf dieselbe Weise abzuleiten wie bei untypisierten literalen Zeichenkettenkonstanten.

> **Hinweis**
>
> Ein Parameterdatentyp kann unangegeben bleiben, indem er auf null gesetzt wird oder indem das Array der Parametertyp-OIDs kürzer gemacht wird als die Anzahl der in der Abfragezeichenkette verwendeten Parametersymbole (`$n`). Ein weiterer Sonderfall ist, dass der Typ eines Parameters als `void` angegeben werden kann, also als OID des Pseudotyps `void`. Dies soll ermöglichen, Parametersymbole für Funktionsparameter zu verwenden, die tatsächlich `OUT`-Parameter sind. Normalerweise gibt es keinen Kontext, in dem ein `void`-Parameter verwendet werden könnte; wenn ein solches Parametersymbol jedoch in der Parameterliste einer Funktion erscheint, wird es effektiv ignoriert. Beispielsweise könnte ein Funktionsaufruf wie `foo($1,$2,$3,$4)` zu einer Funktion mit zwei `IN`- und zwei `OUT`-Argumenten passen, wenn `$3` und `$4` als Typ `void` angegeben sind.

> **Hinweis**
>
> Die in einer `Parse`-Nachricht enthaltene Abfragezeichenkette darf nicht mehr als eine SQL-Anweisung enthalten; andernfalls wird ein Syntaxfehler gemeldet. Diese Einschränkung existiert im Simple-Query-Protokoll nicht, wohl aber im erweiterten Protokoll, weil es das Protokoll unnötig verkomplizieren würde, wenn vorbereitete Anweisungen oder Portale mehrere Kommandos enthalten dürften.

Wenn ein benanntes Prepared-Statement-Objekt erfolgreich erzeugt wurde, bleibt es bis zum Ende der aktuellen Sitzung bestehen, sofern es nicht ausdrücklich zerstört wird. Eine unbenannte vorbereitete Anweisung bleibt nur bis zur nächsten `Parse`-Anweisung bestehen, die die unbenannte Anweisung als Ziel angibt. Beachten Sie, dass eine einfache `Query`-Nachricht die unbenannte Anweisung ebenfalls zerstört. Benannte vorbereitete Anweisungen müssen ausdrücklich geschlossen werden, bevor sie durch eine andere `Parse`-Nachricht neu definiert werden können; für die unbenannte Anweisung ist dies nicht erforderlich. Benannte vorbereitete Anweisungen können auch auf SQL-Kommandoebene mit `PREPARE` und `EXECUTE` erzeugt und angesprochen werden.

Sobald eine vorbereitete Anweisung existiert, kann sie mit einer `Bind`-Nachricht zur Ausführung bereit gemacht werden. Die `Bind`-Nachricht gibt den Namen der vorbereiteten Quellanweisung an, wobei eine leere Zeichenkette die unbenannte vorbereitete Anweisung bezeichnet, den Namen des Zielportals, wobei eine leere Zeichenkette das unbenannte Portal bezeichnet, und die Werte, die für alle in der vorbereiteten Anweisung vorhandenen Parameterplatzhalter verwendet werden sollen. Die übergebene Parametermenge muss zu den von der vorbereiteten Anweisung benötigten Parametern passen. Wenn Sie in der `Parse`-Nachricht `void`-Parameter deklariert haben, übergeben Sie für diese in der `Bind`-Nachricht `NULL`-Werte. `Bind` gibt außerdem das Format an, das für von der Abfrage zurückgegebene Daten verwendet werden soll; das Format kann insgesamt oder spaltenweise angegeben werden. Die Antwort ist entweder `BindComplete` oder `ErrorResponse`.

> **Hinweis**
>
> Die Wahl zwischen Text- und Binärausgabe wird durch die in `Bind` angegebenen Format-Codes bestimmt, unabhängig vom beteiligten SQL-Kommando. Das Attribut `BINARY` in Cursor-Deklarationen ist bei Verwendung des Extended-Query-Protokolls irrelevant.

Die Abfrageplanung erfolgt typischerweise, wenn die `Bind`-Nachricht verarbeitet wird. Wenn die vorbereitete Anweisung keine Parameter hat oder wiederholt ausgeführt wird, kann der Server den erzeugten Plan speichern und bei späteren `Bind`-Nachrichten für dieselbe vorbereitete Anweisung wiederverwenden. Dies tut er jedoch nur, wenn er feststellt, dass ein generischer Plan erzeugt werden kann, der nicht wesentlich ineffizienter ist als ein Plan, der von den konkret übergebenen Parameterwerten abhängt. Soweit es das Protokoll betrifft, geschieht dies transparent.

Wenn ein benanntes Portalobjekt erfolgreich erzeugt wurde, bleibt es bis zum Ende der aktuellen Transaktion bestehen, sofern es nicht ausdrücklich zerstört wird. Ein unbenanntes Portal wird am Ende der Transaktion oder sobald die nächste `Bind`-Anweisung ausgegeben wird, die das unbenannte Portal als Ziel angibt, zerstört. Beachten Sie, dass eine einfache `Query`-Nachricht das unbenannte Portal ebenfalls zerstört. Benannte Portale müssen ausdrücklich geschlossen werden, bevor sie durch eine andere `Bind`-Nachricht neu definiert werden können; für das unbenannte Portal ist dies nicht erforderlich. Benannte Portale können auch auf SQL-Kommandoebene mit `DECLARE CURSOR` und `FETCH` erzeugt und angesprochen werden.

Sobald ein Portal existiert, kann es mit einer `Execute`-Nachricht ausgeführt werden. Die `Execute`-Nachricht gibt den Portalnamen an, wobei eine leere Zeichenkette das unbenannte Portal bezeichnet, sowie eine maximale Anzahl von Ergebniszeilen; null bedeutet „alle Zeilen abrufen“. Die Ergebniszeilenanzahl ist nur für Portale mit Kommandos sinnvoll, die Zeilenmengen zurückgeben; in anderen Fällen wird das Kommando immer vollständig ausgeführt, und die Zeilenanzahl wird ignoriert. Die möglichen Antworten auf `Execute` sind dieselben wie oben für Abfragen beschrieben, die über das Simple-Query-Protokoll ausgegeben werden, mit der Ausnahme, dass `Execute` nicht dazu führt, dass `ReadyForQuery` oder `RowDescription` ausgegeben wird.

Wenn `Execute` beendet wird, bevor die Ausführung eines Portals abgeschlossen ist, weil eine von null verschiedene Ergebniszeilenanzahl erreicht wurde, sendet es eine `PortalSuspended`-Nachricht. Das Auftreten dieser Nachricht teilt dem Frontend mit, dass ein weiteres `Execute` gegen dasselbe Portal ausgegeben werden sollte, um die Operation abzuschließen. Die `CommandComplete`-Nachricht, die den Abschluss des ursprünglichen SQL-Kommandos anzeigt, wird erst gesendet, wenn die Ausführung des Portals abgeschlossen ist. Daher wird eine `Execute`-Phase immer durch genau eine dieser Nachrichten beendet: `CommandComplete`, `EmptyQueryResponse`, wenn das Portal aus einer leeren Abfragezeichenkette erzeugt wurde, `ErrorResponse` oder `PortalSuspended`.

Am Ende jeder Folge von Extended-Query-Nachrichten sollte das Frontend eine `Sync`-Nachricht ausgeben. Diese parameterlose Nachricht veranlasst das Backend, die aktuelle Transaktion zu schließen, wenn es sich nicht innerhalb eines `BEGIN`/`COMMIT`-Transaktionsblocks befindet; „schließen“ bedeutet Commit bei fehlerfreiem Verlauf oder Rollback bei einem Fehler. Anschließend wird eine `ReadyForQuery`-Antwort ausgegeben. Der Zweck von `Sync` besteht darin, einen Resynchronisationspunkt für die Fehlerbehebung bereitzustellen. Wenn beim Verarbeiten irgendeiner Extended-Query-Nachricht ein Fehler erkannt wird, gibt das Backend `ErrorResponse` aus, liest und verwirft dann Nachrichten, bis ein `Sync` erreicht wird, gibt anschließend `ReadyForQuery` aus und kehrt zur normalen Nachrichtenverarbeitung zurück. Beachten Sie jedoch, dass kein Überspringen stattfindet, wenn beim Verarbeiten von `Sync` ein Fehler erkannt wird; dadurch wird sichergestellt, dass für jedes `Sync` genau ein `ReadyForQuery` gesendet wird.

> **Hinweis**
>
> `Sync` schließt keinen mit `BEGIN` geöffneten Transaktionsblock. Diese Situation lässt sich erkennen, da die `ReadyForQuery`-Nachricht Informationen über den Transaktionsstatus enthält.

Zusätzlich zu diesen grundlegenden, erforderlichen Operationen gibt es mehrere optionale Operationen, die mit dem Extended-Query-Protokoll verwendet werden können.

Die `Describe`-Nachricht in der Portalvariante gibt den Namen eines vorhandenen Portals an, oder eine leere Zeichenkette für das unbenannte Portal. Die Antwort ist eine `RowDescription`-Nachricht, die die Zeilen beschreibt, die bei Ausführung des Portals zurückgegeben werden; oder eine `NoData`-Nachricht, wenn das Portal keine Abfrage enthält, die Zeilen zurückgibt; oder `ErrorResponse`, wenn kein solches Portal existiert.

Die `Describe`-Nachricht in der Statement-Variante gibt den Namen einer vorhandenen vorbereiteten Anweisung an, oder eine leere Zeichenkette für die unbenannte vorbereitete Anweisung. Die Antwort ist eine `ParameterDescription`-Nachricht, die die von der Anweisung benötigten Parameter beschreibt, gefolgt von einer `RowDescription`-Nachricht, die die Zeilen beschreibt, die zurückgegeben werden, wenn die Anweisung schließlich ausgeführt wird, oder einer `NoData`-Nachricht, wenn die Anweisung keine Zeilen zurückgeben wird. `ErrorResponse` wird ausgegeben, wenn keine solche vorbereitete Anweisung existiert. Beachten Sie, dass dem Backend die für zurückgegebene Spalten zu verwendenden Formate noch nicht bekannt sind, da `Bind` noch nicht ausgegeben wurde; die Format-Code-Felder in der `RowDescription`-Nachricht sind in diesem Fall Nullen.

> **Tipp**
>
> In den meisten Szenarien sollte das Frontend vor `Execute` die eine oder die andere Variante von `Describe` ausgeben, um sicherzustellen, dass es weiß, wie es die Ergebnisse interpretieren muss, die es zurückerhalten wird.

Die `Close`-Nachricht schließt eine vorhandene vorbereitete Anweisung oder ein vorhandenes Portal und gibt Ressourcen frei. Es ist kein Fehler, `Close` für einen nicht vorhandenen Anweisungs- oder Portalnamen auszugeben. Die Antwort ist normalerweise `CloseComplete`, kann aber `ErrorResponse` sein, wenn beim Freigeben von Ressourcen Schwierigkeiten auftreten. Beachten Sie, dass das Schließen einer vorbereiteten Anweisung implizit alle offenen Portale schließt, die aus dieser Anweisung konstruiert wurden.

Die `Flush`-Nachricht erzeugt keine bestimmte Ausgabe, zwingt das Backend aber, alle in seinen Ausgabepuffern anstehenden Daten auszuliefern. Ein `Flush` muss nach jedem Extended-Query-Kommando außer `Sync` gesendet werden, wenn das Frontend die Ergebnisse dieses Kommandos prüfen möchte, bevor es weitere Kommandos ausgibt. Ohne `Flush` werden vom Backend zurückgegebene Nachrichten in der kleinstmöglichen Anzahl von Paketen zusammengefasst, um den Netzwerk-Overhead zu minimieren.

> **Hinweis**
>
> Die einfache `Query`-Nachricht entspricht ungefähr der Folge `Parse`, `Bind`, Portal-`Describe`, `Execute`, `Close`, `Sync`, wobei die unbenannte vorbereitete Anweisung, das unbenannte Portal und keine Parameter verwendet werden. Ein Unterschied besteht darin, dass sie mehrere SQL-Anweisungen in der Abfragezeichenkette akzeptiert und die Bind/Describe/Execute-Folge automatisch für jede davon nacheinander ausführt. Ein weiterer Unterschied besteht darin, dass sie keine Nachrichten `ParseComplete`, `BindComplete`, `CloseComplete` oder `NoData` zurückgibt.

### 54.2.4. Pipelining

Die Verwendung des Extended-Query-Protokolls erlaubt Pipelining, also das Senden einer Reihe von Abfragen, ohne auf den Abschluss früherer Abfragen zu warten. Dadurch verringert sich die Anzahl der Netzwerk-Roundtrips, die nötig sind, um eine gegebene Folge von Operationen abzuschließen. Der Benutzer muss jedoch sorgfältig überlegen, welches Verhalten erforderlich ist, wenn einer der Schritte fehlschlägt, da spätere Abfragen bereits zum Server unterwegs sind.

Eine Möglichkeit, damit umzugehen, besteht darin, die gesamte Abfragefolge zu einer einzelnen Transaktion zu machen, sie also in `BEGIN ... COMMIT` einzuschließen. Dies hilft jedoch nicht, wenn einige Kommandos unabhängig von anderen committen sollen.

Das Extended-Query-Protokoll bietet eine weitere Möglichkeit, dieses Problem zu verwalten: Zwischen abhängigen Schritten werden keine `Sync`-Nachrichten gesendet. Da das Backend nach einem Fehler Kommandomeldungen überspringt, bis es `Sync` findet, können spätere Kommandos in einer Pipeline automatisch übersprungen werden, wenn ein früheres fehlschlägt, ohne dass der Client dies ausdrücklich mit `BEGIN` und `COMMIT` verwalten muss. Unabhängig committbare Segmente der Pipeline können durch `Sync`-Nachrichten getrennt werden.

Wenn der Client kein explizites `BEGIN` ausgegeben hat, wird ein impliziter Transaktionsblock gestartet, und jedes `Sync` verursacht normalerweise ein implizites `COMMIT`, wenn die vorhergehenden Schritte erfolgreich waren, oder ein implizites `ROLLBACK`, wenn sie fehlgeschlagen sind. Dieser implizite Transaktionsblock wird vom Server erst erkannt, wenn das erste Kommando ohne Sync endet. Es gibt einige DDL-Kommandos, etwa `CREATE DATABASE`, die nicht innerhalb eines Transaktionsblocks ausgeführt werden können. Wenn eines davon in einer Pipeline ausgeführt wird, schlägt es fehl, sofern es nicht das erste Kommando nach einem `Sync` ist. Bei Erfolg erzwingt es außerdem ein sofortiges Commit, um die Datenbankkonsistenz zu wahren. Daher hat ein unmittelbar auf eines dieser Kommandos folgendes `Sync` keine Wirkung außer der Antwort mit `ReadyForQuery`.

Bei Verwendung dieser Methode muss der Abschluss der Pipeline bestimmt werden, indem `ReadyForQuery`-Nachrichten gezählt werden und gewartet wird, bis deren Anzahl die Zahl der gesendeten `Sync`s erreicht. Das Zählen von Kommandoabschlussantworten ist unzuverlässig, da einige Kommandos übersprungen werden können und dann keine Abschlussnachricht erzeugen.

### 54.2.5. Funktionsaufruf

Das Function-Call-Unterprotokoll erlaubt dem Client, den direkten Aufruf jeder Funktion anzufordern, die im Systemkatalog `pg_proc` der Datenbank existiert. Der Client muss Ausführungsrechte für die Funktion haben.

> **Hinweis**
>
> Das Function-Call-Unterprotokoll ist ein Legacy-Feature, das in neuem Code wahrscheinlich besser vermieden wird. Ähnliche Ergebnisse lassen sich erreichen, indem eine vorbereitete Anweisung eingerichtet wird, die `SELECT function($1, ...)` ausführt. Der Function-Call-Zyklus kann dann durch `Bind`/`Execute` ersetzt werden.

Ein Function-Call-Zyklus wird dadurch eingeleitet, dass das Frontend eine `FunctionCall`-Nachricht an das Backend sendet. Das Backend sendet dann je nach Ergebnis des Funktionsaufrufs eine oder mehrere Antwortnachrichten und schließlich eine `ReadyForQuery`-Antwortnachricht. `ReadyForQuery` informiert das Frontend, dass es gefahrlos eine neue Abfrage oder einen neuen Funktionsaufruf senden kann.

Die möglichen Antwortnachrichten vom Backend sind:

- `ErrorResponse`
  - Ein Fehler ist aufgetreten.

- `FunctionCallResponse`
  - Der Funktionsaufruf wurde abgeschlossen und hat das in der Nachricht angegebene Ergebnis zurückgegeben. Beachten Sie, dass das Function-Call-Protokoll nur ein einzelnes skalares Ergebnis behandeln kann, keinen Zeilentyp und keine Ergebnismenge.

- `ReadyForQuery`
  - Die Verarbeitung des Funktionsaufrufs ist abgeschlossen. `ReadyForQuery` wird immer gesendet, unabhängig davon, ob die Verarbeitung erfolgreich oder mit einem Fehler endet.

- `NoticeResponse`
  - Im Zusammenhang mit dem Funktionsaufruf wurde eine Warnmeldung ausgegeben. Notices kommen zusätzlich zu anderen Antworten; das Backend setzt also die Verarbeitung des Kommandos fort.

### 54.2.6. COPY-Operationen

Das Kommando `COPY` ermöglicht schnellen Massendatentransfer zum oder vom Server. Copy-in- und Copy-out-Operationen schalten die Verbindung jeweils in ein eigenes Unterprotokoll um, das bis zum Abschluss der Operation andauert.

Der Copy-in-Modus, also Datentransfer zum Server, wird eingeleitet, wenn das Backend eine SQL-Anweisung `COPY FROM STDIN` ausführt. Das Backend sendet eine `CopyInResponse`-Nachricht an das Frontend. Das Frontend sollte anschließend null oder mehr `CopyData`-Nachrichten senden, die einen Eingabedatenstrom bilden. Die Nachrichtengrenzen müssen nichts mit Zeilengrenzen zu tun haben, obwohl das oft eine vernünftige Wahl ist. Das Frontend kann den Copy-in-Modus beenden, indem es entweder eine `CopyDone`-Nachricht sendet, die erfolgreichen Abschluss erlaubt, oder eine `CopyFail`-Nachricht, die dazu führt, dass die SQL-Anweisung `COPY` mit einem Fehler fehlschlägt. Das Backend kehrt dann zu dem Kommandoverarbeitungsmodus zurück, in dem es sich vor dem Start von `COPY` befand, also entweder zum Simple- oder Extended-Query-Protokoll. Als Nächstes sendet es entweder `CommandComplete`, bei Erfolg, oder `ErrorResponse`, bei Fehler.

Im Fall eines vom Backend erkannten Fehlers während des Copy-in-Modus, einschließlich des Empfangs einer `CopyFail`-Nachricht, gibt das Backend eine `ErrorResponse`-Nachricht aus. Wenn das `COPY`-Kommando über eine Extended-Query-Nachricht ausgegeben wurde, verwirft das Backend nun Frontend-Nachrichten, bis eine `Sync`-Nachricht empfangen wird; dann gibt es `ReadyForQuery` aus und kehrt zur normalen Verarbeitung zurück. Wenn das `COPY`-Kommando in einer einfachen `Query`-Nachricht ausgegeben wurde, wird der Rest dieser Nachricht verworfen und `ReadyForQuery` ausgegeben. In beiden Fällen werden alle nachfolgenden `CopyData`-, `CopyDone`- oder `CopyFail`-Nachrichten, die vom Frontend ausgegeben werden, einfach verworfen.

Das Backend ignoriert während des Copy-in-Modus empfangene `Flush`- und `Sync`-Nachrichten. Der Empfang jedes anderen Nicht-Copy-Nachrichtentyps stellt einen Fehler dar, der den Copy-in-Zustand wie oben beschrieben abbricht. Die Ausnahme für `Flush` und `Sync` dient der Bequemlichkeit von Client-Bibliotheken, die nach einer `Execute`-Nachricht immer `Flush` oder `Sync` senden, ohne zu prüfen, ob das auszuführende Kommando ein `COPY FROM STDIN` ist.

Der Copy-out-Modus, also Datentransfer vom Server, wird eingeleitet, wenn das Backend eine SQL-Anweisung `COPY TO STDOUT` ausführt. Das Backend sendet eine `CopyOutResponse`-Nachricht an das Frontend, gefolgt von null oder mehr `CopyData`-Nachrichten, immer eine pro Zeile, gefolgt von `CopyDone`. Das Backend kehrt dann zu dem Kommandoverarbeitungsmodus zurück, in dem es sich vor dem Start von `COPY` befand, und sendet `CommandComplete`. Das Frontend kann die Übertragung nicht abbrechen, außer durch Schließen der Verbindung oder Ausgabe einer Abbruchanforderung, kann aber unerwünschte `CopyData`- und `CopyDone`-Nachrichten verwerfen.

Im Fall eines vom Backend erkannten Fehlers während des Copy-out-Modus gibt das Backend eine `ErrorResponse`-Nachricht aus und kehrt zur normalen Verarbeitung zurück. Das Frontend sollte den Empfang von `ErrorResponse` als Beendigung des Copy-out-Modus behandeln.

Es ist möglich, dass `NoticeResponse`- und `ParameterStatus`-Nachrichten zwischen `CopyData`-Nachrichten eingestreut werden; Frontends müssen diese Fälle behandeln und sollten auch auf andere asynchrone Nachrichtentypen vorbereitet sein, siehe Abschnitt 54.2.7. Andernfalls kann jeder Nachrichtentyp außer `CopyData` oder `CopyDone` als Beendigung des Copy-out-Modus behandelt werden.

Es gibt einen weiteren Copy-bezogenen Modus namens Copy-both, der schnellen Massendatentransfer zum und vom Server erlaubt. Der Copy-both-Modus wird eingeleitet, wenn ein Backend im Walsender-Modus eine `START_REPLICATION`-Anweisung ausführt. Das Backend sendet eine `CopyBothResponse`-Nachricht an das Frontend. Sowohl Backend als auch Frontend können dann `CopyData`-Nachrichten senden, bis eine der beiden Seiten eine `CopyDone`-Nachricht sendet. Nachdem der Client eine `CopyDone`-Nachricht gesendet hat, wechselt die Verbindung vom Copy-both-Modus in den Copy-out-Modus, und der Client darf keine weiteren `CopyData`-Nachrichten senden. Entsprechend wechselt die Verbindung in den Copy-in-Modus, wenn der Server eine `CopyDone`-Nachricht sendet, und der Server darf keine weiteren `CopyData`-Nachrichten senden. Nachdem beide Seiten eine `CopyDone`-Nachricht gesendet haben, wird der Copy-Modus beendet, und das Backend kehrt zum Kommandoverarbeitungsmodus zurück. Im Fall eines vom Backend erkannten Fehlers während des Copy-both-Modus gibt das Backend eine `ErrorResponse`-Nachricht aus, verwirft Frontend-Nachrichten, bis eine `Sync`-Nachricht empfangen wird, und gibt dann `ReadyForQuery` aus und kehrt zur normalen Verarbeitung zurück. Das Frontend sollte den Empfang von `ErrorResponse` als Beendigung des Copy-Vorgangs in beide Richtungen behandeln; in diesem Fall sollte kein `CopyDone` gesendet werden. Weitere Informationen zum über den Copy-both-Modus übertragenen Unterprotokoll finden Sie in [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll).

Die Nachrichten `CopyInResponse`, `CopyOutResponse` und `CopyBothResponse` enthalten Felder, die das Frontend über die Anzahl der Spalten pro Zeile und die für jede Spalte verwendeten Format-Codes informieren. Nach aktueller Implementierung verwenden alle Spalten in einer gegebenen `COPY`-Operation dasselbe Format, das Nachrichtendesign setzt dies jedoch nicht voraus.

### 54.2.7. Asynchrone Operationen

Es gibt mehrere Fälle, in denen das Backend Nachrichten sendet, die nicht ausdrücklich durch den Kommandostrom des Frontends ausgelöst wurden. Frontends müssen darauf vorbereitet sein, diese Nachrichten jederzeit zu behandeln, auch wenn sie gerade nicht mit einer Abfrage beschäftigt sind. Mindestens sollte auf diese Fälle geprüft werden, bevor mit dem Lesen einer Abfrageantwort begonnen wird.

Es ist möglich, dass `NoticeResponse`-Nachrichten aufgrund externer Aktivitäten erzeugt werden. Wenn der Datenbankadministrator beispielsweise ein „schnelles“ Herunterfahren der Datenbank anordnet, sendet das Backend eine `NoticeResponse`, die dies mitteilt, bevor es die Verbindung schließt. Entsprechend sollten Frontends immer darauf vorbereitet sein, `NoticeResponse`-Nachrichten zu akzeptieren und anzuzeigen, auch wenn die Verbindung nominell im Leerlauf ist.

`ParameterStatus`-Nachrichten werden erzeugt, wann immer sich der aktive Wert eines Parameters ändert, von dem das Backend annimmt, dass das Frontend ihn kennen sollte. Am häufigsten geschieht dies als Antwort auf ein vom Frontend ausgeführtes SQL-Kommando `SET`; dieser Fall ist effektiv synchron. Es ist aber auch möglich, dass Parameterstatusänderungen auftreten, weil der Administrator eine Konfigurationsdatei geändert und anschließend das Signal `SIGHUP` an den Server gesendet hat. Wenn ein `SET`-Kommando zurückgerollt wird, wird ebenfalls eine passende `ParameterStatus`-Nachricht erzeugt, um den aktuell wirksamen Wert zu melden.

Derzeit gibt es eine fest eingebaute Menge von Parametern, für die `ParameterStatus` erzeugt wird:

```text
application_name              scram_iterations
client_encoding               search_path
DateStyle                     server_encoding
default_transaction_read_only server_version
in_hot_standby                session_authorization
integer_datetimes             standard_conforming_strings
IntervalStyle                 TimeZone
is_superuser
```

`default_transaction_read_only` und `in_hot_standby` wurden von Versionen vor 14 nicht gemeldet; `scram_iterations` wurde von Versionen vor 16 nicht gemeldet; `search_path` wurde von Versionen vor 18 nicht gemeldet. Beachten Sie, dass `server_version`, `server_encoding` und `integer_datetimes` Pseudoparameter sind, die sich nach dem Start nicht ändern können. Diese Menge kann sich in Zukunft ändern oder sogar konfigurierbar werden. Daher sollte ein Frontend `ParameterStatus` für Parameter, die es nicht versteht oder die ihm nicht wichtig sind, einfach ignorieren.

Wenn ein Frontend ein `LISTEN`-Kommando ausgibt, sendet das Backend immer dann eine `NotificationResponse`-Nachricht, nicht zu verwechseln mit `NoticeResponse`, wenn ein `NOTIFY`-Kommando für denselben Kanalnamen ausgeführt wird.

> **Hinweis**
>
> Derzeit kann `NotificationResponse` nur außerhalb einer Transaktion gesendet werden und tritt daher nicht mitten in einer Kommando-Antwort-Sequenz auf, obwohl sie direkt vor `ReadyForQuery` auftreten kann. Es ist jedoch unklug, Frontend-Logik zu entwerfen, die davon ausgeht. Gute Praxis ist, `NotificationResponse` an jedem Punkt des Protokolls akzeptieren zu können.

### 54.2.8. Laufende Anforderungen abbrechen

Während der Verarbeitung einer Abfrage kann das Frontend den Abbruch der Abfrage anfordern. Die Abbruchanforderung wird aus Gründen der Implementierungseffizienz nicht direkt über die offene Verbindung zum Backend gesendet: Das Backend soll während der Abfrageverarbeitung nicht ständig auf neue Eingaben vom Frontend prüfen müssen. Abbruchanforderungen sollten relativ selten sein, daher werden sie etwas umständlicher gemacht, um im Normalfall keinen Nachteil zu verursachen.

Um eine Abbruchanforderung auszugeben, öffnet das Frontend eine neue Verbindung zum Server und sendet eine `CancelRequest`-Nachricht statt der `StartupMessage`, die normalerweise über eine neue Verbindung gesendet würde. Der Server verarbeitet diese Anforderung und schließt dann die Verbindung. Aus Sicherheitsgründen wird keine direkte Antwort auf die `CancelRequest`-Nachricht gegeben.

Eine `CancelRequest`-Nachricht wird ignoriert, sofern sie nicht dieselben Schlüsseldaten enthält, also PID und geheimen Schlüssel, die dem Frontend beim Verbindungsstart übergeben wurden. Wenn die Anforderung zu PID und geheimem Schlüssel eines aktuell ausführenden Backends passt, wird die Verarbeitung der aktuellen Abfrage abgebrochen. In der bestehenden Implementierung geschieht dies, indem ein spezielles Signal an den Backend-Prozess gesendet wird, der die Abfrage verarbeitet.

Das Abbruchsignal kann eine Wirkung haben oder auch nicht. Wenn es beispielsweise eintrifft, nachdem das Backend die Verarbeitung der Abfrage abgeschlossen hat, hat es keine Wirkung. Wenn der Abbruch wirksam ist, führt er dazu, dass das aktuelle Kommando vorzeitig mit einer Fehlermeldung beendet wird.

Das Ergebnis all dessen ist, dass das Frontend aus Sicherheits- und Effizienzgründen keine direkte Möglichkeit hat festzustellen, ob eine Abbruchanforderung erfolgreich war. Es muss weiter darauf warten, dass das Backend auf die Abfrage antwortet. Einen Abbruch auszugeben erhöht lediglich die Wahrscheinlichkeit, dass die aktuelle Abfrage bald beendet wird, und erhöht die Wahrscheinlichkeit, dass sie mit einer Fehlermeldung fehlschlägt, statt erfolgreich zu sein.

Da die Abbruchanforderung über eine neue Verbindung zum Server gesendet wird und nicht über die reguläre Frontend/Backend-Kommunikationsverbindung, kann sie von jedem Prozess ausgegeben werden, nicht nur von dem Frontend, dessen Abfrage abgebrochen werden soll. Dies kann beim Bau von Mehrprozessanwendungen zusätzliche Flexibilität bieten. Es führt aber auch ein Sicherheitsrisiko ein, da Unbefugte versuchen könnten, Abfragen abzubrechen. Dieses Sicherheitsrisiko wird dadurch adressiert, dass in Abbruchanforderungen ein dynamisch erzeugter geheimer Schlüssel angegeben werden muss.

### 54.2.9. Beendigung

Das normale, geordnete Beendigungsverfahren besteht darin, dass das Frontend eine `Terminate`-Nachricht sendet und die Verbindung sofort schließt. Beim Empfang dieser Nachricht schließt das Backend die Verbindung und beendet sich.

In seltenen Fällen, etwa bei einem vom Administrator angeordneten Herunterfahren der Datenbank, kann das Backend die Verbindung trennen, ohne dass das Frontend dies angefordert hat. In solchen Fällen versucht das Backend, vor dem Schließen der Verbindung eine Fehler- oder Hinweismeldung zu senden, die den Grund für die Trennung angibt.

Andere Beendigungsszenarien ergeben sich aus verschiedenen Fehlerfällen, etwa einem Core Dump auf der einen oder anderen Seite, Verlust der Kommunikationsverbindung, Verlust der Synchronisierung der Nachrichtengrenzen usw. Wenn entweder Frontend oder Backend ein unerwartetes Schließen der Verbindung bemerkt, sollte es aufräumen und sich beenden. Das Frontend hat die Möglichkeit, durch erneute Kontaktaufnahme mit dem Server ein neues Backend zu starten, wenn es sich nicht selbst beenden möchte. Auch der Empfang eines nicht erkennbaren Nachrichtentyps ist ein guter Grund, die Verbindung zu schließen, da dies wahrscheinlich auf einen Verlust der Synchronisierung der Nachrichtengrenzen hinweist.

Bei normaler wie abnormaler Beendigung wird jede offene Transaktion zurückgerollt, nicht committet. Beachten Sie jedoch, dass das Backend eine Nicht-`SELECT`-Abfrage wahrscheinlich abschließen wird, bevor es die Trennung bemerkt, wenn ein Frontend die Verbindung trennt, während diese Abfrage verarbeitet wird. Wenn die Abfrage außerhalb eines Transaktionsblocks, also einer `BEGIN ... COMMIT`-Sequenz, läuft, können ihre Ergebnisse committet werden, bevor die Trennung erkannt wird.

### 54.2.10. SSL-Sitzungsverschlüsselung

Wenn PostgreSQL mit SSL-Unterstützung gebaut wurde, kann die Frontend/Backend-Kommunikation mit SSL verschlüsselt werden. Dies bietet Kommunikationssicherheit in Umgebungen, in denen Angreifer den Sitzungsverkehr mitschneiden könnten. Weitere Informationen zum Verschlüsseln von PostgreSQL-Sitzungen mit SSL finden Sie in [Abschnitt 18.9](18_Servereinrichtung_und_Betrieb.md#189-sichere-tcpipverbindungen-mit-ssl).

Um eine SSL-verschlüsselte Verbindung einzuleiten, sendet das Frontend zunächst eine `SSLRequest`-Nachricht statt einer `StartupMessage`. Der Server antwortet dann mit einem einzelnen Byte, das `S` oder `N` enthält und damit angibt, ob er bereit oder nicht bereit ist, SSL auszuführen. Das Frontend kann die Verbindung an diesem Punkt schließen, wenn es mit der Antwort unzufrieden ist. Um nach `S` fortzufahren, führen Sie mit dem Server einen SSL-Startup-Handshake aus, der hier nicht beschrieben wird und Teil der SSL-Spezifikation ist. Wenn dies erfolgreich ist, fahren Sie mit dem Senden der üblichen `StartupMessage` fort. In diesem Fall werden die `StartupMessage` und alle nachfolgenden Daten SSL-verschlüsselt. Um nach `N` fortzufahren, senden Sie die übliche `StartupMessage` und fahren ohne Verschlüsselung fort. Alternativ ist es zulässig, nach einer `N`-Antwort eine `GSSENCRequest`-Nachricht auszugeben, um zu versuchen, GSSAPI-Verschlüsselung statt SSL zu verwenden.

Das Frontend sollte auch darauf vorbereitet sein, eine `ErrorMessage`-Antwort des Servers auf `SSLRequest` zu behandeln. Das Frontend sollte diese Fehlermeldung dem Benutzer oder der Anwendung nicht anzeigen, da der Server nicht authentifiziert wurde (CVE-2024-10977). In diesem Fall muss die Verbindung geschlossen werden; das Frontend kann sich aber dafür entscheiden, eine neue Verbindung zu öffnen und ohne SSL-Anforderung fortzufahren.

Wenn SSL-Verschlüsselung durchgeführt werden kann, wird erwartet, dass der Server nur das einzelne Byte `S` sendet und dann darauf wartet, dass das Frontend einen SSL-Handshake einleitet. Wenn an diesem Punkt zusätzliche Bytes zum Lesen verfügbar sind, bedeutet dies wahrscheinlich, dass ein Man-in-the-Middle versucht, einen Buffer-Stuffing-Angriff auszuführen (CVE-2021-23222). Frontends sollten so programmiert sein, dass sie entweder genau ein Byte aus dem Socket lesen, bevor sie den Socket an ihre SSL-Bibliothek übergeben, oder es als Protokollverletzung behandeln, wenn sie feststellen, dass sie zusätzliche Bytes gelesen haben.

Ebenso erwartet der Server, dass der Client die SSL-Aushandlung nicht beginnt, bevor er die Ein-Byte-Antwort des Servers auf die SSL-Anforderung empfangen hat. Wenn der Client die SSL-Aushandlung sofort beginnt, ohne auf den Empfang der Serverantwort zu warten, kann dies die Verbindungslatenz um einen Roundtrip verringern. Dies geschieht jedoch um den Preis, dass der Fall nicht behandelt werden kann, in dem der Server eine negative Antwort auf die SSL-Anforderung sendet. In diesem Fall trennt der Server einfach die Verbindung, statt entweder mit GSSAPI, einer unverschlüsselten Verbindung oder einem Protokollfehler fortzufahren.

Eine anfängliche `SSLRequest` kann auch in einer Verbindung verwendet werden, die geöffnet wird, um eine `CancelRequest`-Nachricht zu senden.

Es gibt eine zweite alternative Möglichkeit, SSL-Verschlüsselung einzuleiten. Der Server erkennt Verbindungen, die sofort mit der SSL-Aushandlung beginnen, ohne vorherige `SSLRequest`-Pakete. Sobald die SSL-Verbindung hergestellt ist, erwartet der Server ein normales Startup-Request-Paket und setzt die Aushandlung über den verschlüsselten Kanal fort. In diesem Fall werden alle anderen Anforderungen für Verschlüsselung abgelehnt. Diese Methode wird für allgemeine Werkzeuge nicht bevorzugt, da sie nicht die beste verfügbare Verbindungsverschlüsselung aushandeln und keine unverschlüsselten Verbindungen behandeln kann. Sie ist jedoch nützlich für Umgebungen, in denen Server und Client gemeinsam kontrolliert werden. In diesem Fall vermeidet sie einen Roundtrip Latenz und erlaubt die Verwendung von Netzwerkwerkzeugen, die auf Standard-SSL-Verbindungen angewiesen sind. Bei Verwendung von SSL-Verbindungen in diesem Stil muss der Client die von RFC 7301 definierte ALPN-Erweiterung verwenden, um vor Protocol-Confusion-Angriffen zu schützen. Das PostgreSQL-Protokoll ist als `postgresql` in der IANA-Registry „TLS ALPN Protocol IDs“ registriert.

Während das Protokoll selbst keine Möglichkeit bietet, dass der Server SSL-Verschlüsselung erzwingt, kann der Administrator den Server so konfigurieren, dass er unverschlüsselte Sitzungen als Nebenprodukt der Authentifizierungsprüfung ablehnt.

<https://www.postgresql.org/support/security/CVE-2024-10977/>
<https://www.postgresql.org/support/security/CVE-2021-23222/>
<https://tools.ietf.org/html/rfc7301>
<https://www.iana.org/assignments/tls-extensiontype-values/tls-extensiontype-values.xhtml#alpn-protocol-ids>

### 54.2.11. GSSAPI-Sitzungsverschlüsselung

Wenn PostgreSQL mit GSSAPI-Unterstützung gebaut wurde, kann die Frontend/Backend-Kommunikation mit GSSAPI verschlüsselt werden. Dies bietet Kommunikationssicherheit in Umgebungen, in denen Angreifer den Sitzungsverkehr mitschneiden könnten. Weitere Informationen zum Verschlüsseln von PostgreSQL-Sitzungen mit GSSAPI finden Sie in [Abschnitt 18.10](18_Servereinrichtung_und_Betrieb.md#1810-sichere-tcpipverbindungen-mit-gssapiverschlüsselung).

Um eine GSSAPI-verschlüsselte Verbindung einzuleiten, sendet das Frontend zunächst eine `GSSENCRequest`-Nachricht statt einer `StartupMessage`. Der Server antwortet dann mit einem einzelnen Byte, das `G` oder `N` enthält und damit angibt, ob er bereit oder nicht bereit ist, GSSAPI-Verschlüsselung auszuführen. Das Frontend kann die Verbindung an diesem Punkt schließen, wenn es mit der Antwort unzufrieden ist. Um nach `G` fortzufahren, führen Sie mit den GSSAPI-C-Bindings, wie in RFC 2744 oder gleichwertig beschrieben, eine GSSAPI-Initialisierung aus, indem Sie `gss_init_sec_context()` in einer Schleife aufrufen und das Ergebnis an den Server senden. Beginnen Sie mit leerer Eingabe und anschließend mit jedem Ergebnis vom Server, bis kein Output mehr zurückgegeben wird. Beim Senden der Ergebnisse von `gss_init_sec_context()` an den Server wird der Nachricht die Länge als Vier-Byte-Integer in Netzwerk-Byte-Reihenfolge vorangestellt. Um nach `N` fortzufahren, senden Sie die übliche `StartupMessage` und fahren ohne Verschlüsselung fort. Alternativ ist es zulässig, nach einer `N`-Antwort eine `SSLRequest`-Nachricht auszugeben, um zu versuchen, SSL-Verschlüsselung statt GSSAPI zu verwenden.

Das Frontend sollte auch darauf vorbereitet sein, eine `ErrorMessage`-Antwort des Servers auf `GSSENCRequest` zu behandeln. Das Frontend sollte diese Fehlermeldung dem Benutzer oder der Anwendung nicht anzeigen, da der Server nicht authentifiziert wurde (CVE-2024-10977). In diesem Fall muss die Verbindung geschlossen werden; das Frontend kann sich aber dafür entscheiden, eine neue Verbindung zu öffnen und ohne Anforderung von GSSAPI-Verschlüsselung fortzufahren.

Wenn GSSAPI-Verschlüsselung durchgeführt werden kann, wird erwartet, dass der Server nur das einzelne Byte `G` sendet und dann darauf wartet, dass das Frontend einen GSSAPI-Handshake einleitet. Wenn an diesem Punkt zusätzliche Bytes zum Lesen verfügbar sind, bedeutet dies wahrscheinlich, dass ein Man-in-the-Middle versucht, einen Buffer-Stuffing-Angriff auszuführen (CVE-2021-23222). Frontends sollten so programmiert sein, dass sie entweder genau ein Byte aus dem Socket lesen, bevor sie den Socket an ihre GSSAPI-Bibliothek übergeben, oder es als Protokollverletzung behandeln, wenn sie feststellen, dass sie zusätzliche Bytes gelesen haben.

Eine anfängliche `GSSENCRequest` kann auch in einer Verbindung verwendet werden, die geöffnet wird, um eine `CancelRequest`-Nachricht zu senden.

Sobald GSSAPI-Verschlüsselung erfolgreich eingerichtet wurde, verwenden Sie `gss_wrap()`, um die übliche `StartupMessage` und alle nachfolgenden Daten zu verschlüsseln, wobei der Länge des Ergebnisses von `gss_wrap()` als Vier-Byte-Integer in Netzwerk-Byte-Reihenfolge die eigentliche verschlüsselte Nutzlast vorangestellt wird. Beachten Sie, dass der Server nur verschlüsselte Pakete vom Client akzeptiert, die kleiner als 16 kB sind; `gss_wrap_size_limit()` sollte vom Client verwendet werden, um die Größe der unverschlüsselten Nachricht zu bestimmen, die in dieses Limit passt, und größere Nachrichten sollten in mehrere `gss_wrap()`-Aufrufe aufgeteilt werden. Typische Segmente bestehen aus 8 kB unverschlüsselten Daten, was zu verschlüsselten Paketen von etwas mehr als 8 kB führt, aber deutlich innerhalb des Maximums von 16 kB bleibt. Es ist zu erwarten, dass der Server keine verschlüsselten Pakete größer als 16 kB an den Client sendet.

Während das Protokoll selbst keine Möglichkeit bietet, dass der Server GSSAPI-Verschlüsselung erzwingt, kann der Administrator den Server so konfigurieren, dass er unverschlüsselte Sitzungen als Nebenprodukt der Authentifizierungsprüfung ablehnt.

<https://datatracker.ietf.org/doc/html/rfc2744>
<https://www.postgresql.org/support/security/CVE-2024-10977/>
<https://www.postgresql.org/support/security/CVE-2021-23222/>

## 54.3. SASL-Authentifizierung

SASL ist ein Framework für Authentifizierung in verbindungsorientierten Protokollen. Derzeit implementiert PostgreSQL drei SASL-Authentifizierungsmechanismen: `SCRAM-SHA-256`, `SCRAM-SHA-256-PLUS` und `OAUTHBEARER`. In Zukunft können weitere hinzukommen. Die folgenden Schritte zeigen allgemein, wie SASL-Authentifizierung durchgeführt wird, während die nächsten Unterabschnitte mehr Details zu bestimmten Mechanismen geben.

**Nachrichtenfluss der SASL-Authentifizierung**

1. Um einen SASL-Authentifizierungsaustausch zu beginnen, sendet der Server eine `AuthenticationSASL`-Nachricht. Sie enthält eine Liste von SASL-Authentifizierungsmechanismen, die der Server akzeptieren kann, in der vom Server bevorzugten Reihenfolge.

2. Der Client wählt einen der unterstützten Mechanismen aus der Liste aus und sendet eine `SASLInitialResponse`-Nachricht an den Server. Die Nachricht enthält den Namen des ausgewählten Mechanismus und, falls der ausgewählte Mechanismus dies verwendet, eine optionale Initial Client Response.

3. Es folgen eine oder mehrere Server-Challenge- und Client-Response-Nachrichten. Jede Server-Challenge wird in einer `AuthenticationSASLContinue`-Nachricht gesendet, gefolgt von einer Antwort des Clients in einer `SASLResponse`-Nachricht. Die Einzelheiten der Nachrichten sind mechanismusspezifisch.

4. Wenn der Authentifizierungsaustausch erfolgreich abgeschlossen ist, sendet der Server schließlich optional eine `AuthenticationSASLFinal`-Nachricht, unmittelbar gefolgt von einer `AuthenticationOk`-Nachricht. `AuthenticationSASLFinal` enthält zusätzliche Daten vom Server zum Client, deren Inhalt vom ausgewählten Authentifizierungsmechanismus abhängt. Wenn der Authentifizierungsmechanismus keine zusätzlichen Daten verwendet, die beim Abschluss gesendet werden, wird die Nachricht `AuthenticationSASLFinal` nicht gesendet.

Bei einem Fehler kann der Server die Authentifizierung in jeder Phase abbrechen und eine `ErrorMessage` senden.

### 54.3.1. SCRAM-SHA-256-Authentifizierung

`SCRAM-SHA-256` und seine Variante mit Channel Binding, `SCRAM-SHA-256-PLUS`, sind passwortbasierte Authentifizierungsmechanismen. Sie werden ausführlich in RFC 7677 und RFC 5802 beschrieben.

Wenn `SCRAM-SHA-256` in PostgreSQL verwendet wird, ignoriert der Server den Benutzernamen, den der Client in der `client-first-message` sendet. Stattdessen wird der Benutzername verwendet, der bereits in der Startup-Nachricht gesendet wurde. PostgreSQL unterstützt mehrere Zeichenkodierungen, während SCRAM vorschreibt, dass für den Benutzernamen UTF-8 verwendet wird; daher kann es unmöglich sein, den PostgreSQL-Benutzernamen in UTF-8 darzustellen.

Die SCRAM-Spezifikation schreibt vor, dass auch das Passwort in UTF-8 vorliegt und mit dem SASLprep-Algorithmus verarbeitet wird. PostgreSQL verlangt jedoch nicht, dass für das Passwort UTF-8 verwendet wird. Wenn das Passwort eines Benutzers gesetzt wird, wird es mit SASLprep verarbeitet, als läge es in UTF-8 vor, unabhängig von der tatsächlich verwendeten Kodierung. Wenn es jedoch keine gültige UTF-8-Bytesequenz ist oder UTF-8-Bytesequenzen enthält, die vom SASLprep-Algorithmus verboten sind, wird das rohe Passwort ohne SASLprep-Verarbeitung verwendet, statt einen Fehler auszulösen. Dadurch kann das Passwort normalisiert werden, wenn es in UTF-8 vorliegt, während weiterhin ein Nicht-UTF-8-Passwort verwendet werden kann und das System nicht wissen muss, in welcher Kodierung das Passwort vorliegt.

Channel Binding wird in PostgreSQL-Builds mit SSL-Unterstützung unterstützt. Der SASL-Mechanismusname für SCRAM mit Channel Binding ist `SCRAM-SHA-256-PLUS`. Der von PostgreSQL verwendete Channel-Binding-Typ ist `tls-server-end-point`.

Bei SCRAM ohne Channel Binding wählt der Server eine Zufallszahl, die an den Client übertragen wird, um sie mit dem vom Benutzer angegebenen Passwort im übertragenen Passwort-Hash zu mischen. Dies verhindert zwar, dass der Passwort-Hash in einer späteren Sitzung erfolgreich erneut übertragen wird, verhindert aber nicht, dass ein falscher Server zwischen dem echten Server und dem Client den Zufallswert des Servers durchreicht und sich erfolgreich authentifiziert.

SCRAM mit Channel Binding verhindert solche Man-in-the-Middle-Angriffe, indem die Signatur des Serverzertifikats in den übertragenen Passwort-Hash gemischt wird. Ein falscher Server kann zwar das Zertifikat des echten Servers erneut übertragen, hat aber keinen Zugriff auf den zu diesem Zertifikat passenden privaten Schlüssel und kann daher nicht beweisen, dass er dessen Besitzer ist, was zum Fehlschlagen der SSL-Verbindung führt.

**Beispiel**

1. Der Server sendet eine `AuthenticationSASL`-Nachricht. Sie enthält eine Liste von SASL-Authentifizierungsmechanismen, die der Server akzeptieren kann. Das sind `SCRAM-SHA-256-PLUS` und `SCRAM-SHA-256`, wenn der Server mit SSL-Unterstützung gebaut wurde, andernfalls nur letzterer.

2. Der Client antwortet mit einer `SASLInitialResponse`-Nachricht, die den gewählten Mechanismus angibt, `SCRAM-SHA-256` oder `SCRAM-SHA-256-PLUS`. Ein Client kann einen der beiden Mechanismen frei wählen, sollte für bessere Sicherheit aber die Channel-Binding-Variante wählen, wenn er sie unterstützt. Im Feld Initial Client Response enthält die Nachricht die SCRAM-`client-first-message`. Die `client-first-message` enthält außerdem den vom Client gewählten Channel-Binding-Typ.

3. Der Server sendet eine `AuthenticationSASLContinue`-Nachricht mit einer SCRAM-`server-first-message` als Inhalt.

4. Der Client sendet eine `SASLResponse`-Nachricht mit einer SCRAM-`client-final-message` als Inhalt.

5. Der Server sendet eine `AuthenticationSASLFinal`-Nachricht mit der SCRAM-`server-final-message`, unmittelbar gefolgt von einer `AuthenticationOk`-Nachricht.

<https://datatracker.ietf.org/doc/html/rfc7677>
<https://datatracker.ietf.org/doc/html/rfc5802>

### 54.3.2. OAUTHBEARER-Authentifizierung

`OAUTHBEARER` ist ein tokenbasierter Mechanismus für föderierte Authentifizierung. Er wird ausführlich in RFC 7628 beschrieben.

Ein typischer Austausch unterscheidet sich je nachdem, ob der Client bereits ein Bearer-Token für den aktuellen Benutzer im Cache hat. Wenn nicht, findet der Austausch über zwei Verbindungen statt: die erste „Discovery“-Verbindung, um OAuth-Metadaten vom Server zu erhalten, und die zweite Verbindung, um das Token zu senden, nachdem der Client es erhalten hat. `libpq` implementiert derzeit keine Caching-Methode als Teil seines eingebauten Ablaufs und verwendet daher den Austausch über zwei Verbindungen.

Dieser Mechanismus wird wie SCRAM vom Client initiiert. Die anfängliche Client-Antwort besteht aus dem von SCRAM verwendeten Standard-„GS2“-Header, gefolgt von einer Liste von `key=value`-Paaren. Der einzige derzeit vom Server unterstützte Schlüssel ist `auth`, der das Bearer-Token enthält. `OAUTHBEARER` spezifiziert zusätzlich drei optionale Komponenten der anfänglichen Client-Antwort, die `authzid` des GS2-Headers sowie die Schlüssel `host` und `port`, die derzeit vom Server ignoriert werden.

`OAUTHBEARER` unterstützt kein Channel Binding, und es gibt keinen Mechanismus `OAUTHBEARER-PLUS`. Dieser Mechanismus verwendet bei erfolgreicher Authentifizierung keine Serverdaten, daher wird die Nachricht `AuthenticationSASLFinal` im Austausch nicht verwendet.

**Beispiel**

1. Während des ersten Austauschs sendet der Server eine `AuthenticationSASL`-Nachricht, in der der Mechanismus `OAUTHBEARER` angeboten wird.

2. Der Client antwortet mit einer `SASLInitialResponse`-Nachricht, die den Mechanismus `OAUTHBEARER` angibt. Unter der Annahme, dass der Client noch kein gültiges Bearer-Token für den aktuellen Benutzer hat, ist das Feld `auth` leer und zeigt damit eine Discovery-Verbindung an.

3. Der Server sendet eine `AuthenticationSASLContinue`-Nachricht, die einen Fehlerstatus zusammen mit einer bekannten URI und Scopes enthält, die der Client verwenden sollte, um einen OAuth-Flow durchzuführen.

4. Der Client sendet eine `SASLResponse`-Nachricht, die die leere Menge enthält, also ein einzelnes Byte `0x01`, um seine Hälfte des Discovery-Austauschs abzuschließen.

5. Der Server sendet eine `ErrorMessage`, um den ersten Austausch fehlschlagen zu lassen.

An diesem Punkt führt der Client einen von vielen möglichen OAuth-Flows aus, um ein Bearer-Token zu erhalten, wobei er neben den vom Server bereitgestellten Metadaten alle Metadaten verwendet, mit denen er konfiguriert wurde. Diese Beschreibung bleibt absichtlich vage; `OAUTHBEARER` spezifiziert oder erzwingt keine bestimmte Methode, um ein Token zu erhalten.

Sobald der Client ein Token hat, verbindet er sich erneut mit dem Server für den endgültigen Austausch:

6. Der Server sendet erneut eine `AuthenticationSASL`-Nachricht, in der der Mechanismus `OAUTHBEARER` angeboten wird.

7. Der Client antwortet mit einer `SASLInitialResponse`-Nachricht, aber diesmal enthält das Feld `auth` in der Nachricht das Bearer-Token, das während des Client-Flows erhalten wurde.

8. Der Server validiert das Token gemäß den Anweisungen des Token-Providers. Wenn der Client zum Verbinden autorisiert ist, sendet er eine `AuthenticationOk`-Nachricht, um den SASL-Austausch zu beenden.

<https://datatracker.ietf.org/doc/html/rfc7628>

## 54.4. Streaming-Replikationsprotokoll

Um Streaming-Replikation einzuleiten, sendet das Frontend den Parameter `replication` in der Startnachricht. Ein boolescher Wert `true` (oder `on`, `yes`, `1`) weist das Backend an, in den Walsender-Modus für physische Replikation zu wechseln. Dort kann statt SQL-Anweisungen eine kleine Menge von Replikationsbefehlen ausgeführt werden, die unten gezeigt wird.

Wird `database` als Wert für den Parameter `replication` übergeben, wechselt das Backend in den Walsender-Modus für logische Replikation und verbindet sich mit der Datenbank, die im Parameter `dbname` angegeben ist. Im Walsender-Modus für logische Replikation können die unten gezeigten Replikationsbefehle sowie normale SQL-Befehle ausgeführt werden.

Sowohl im Walsender-Modus für physische als auch für logische Replikation kann nur das einfache Abfrageprotokoll verwendet werden.

Zum Testen von Replikationsbefehlen kann man über `psql` oder ein anderes Werkzeug, das `libpq` verwendet, mit einer Verbindungszeichenfolge inklusive der Option `replication` eine Replikationsverbindung herstellen, zum Beispiel:

```text
psql "dbname=postgres replication=database" -c "IDENTIFY_SYSTEM;"
```

Oft ist es jedoch nützlicher, `pg_receivewal` (für physische Replikation) oder `pg_recvlogical` (für logische Replikation) zu verwenden.

Replikationsbefehle werden im Serverprotokoll protokolliert, wenn `log_replication_commands` aktiviert ist.

Im Replikationsmodus werden die folgenden Befehle akzeptiert:

`IDENTIFY_SYSTEM`

Fordert den Server auf, sich zu identifizieren. Der Server antwortet mit einer Ergebnismenge aus einer einzelnen Zeile, die vier Felder enthält:

`systemid` (`text`)

Die eindeutige Systemkennung, die den Cluster identifiziert. Damit kann geprüft werden, ob das Basis-Backup, mit dem der Standby initialisiert wurde, aus demselben Cluster stammt.

`timeline` (`int8`)

Die aktuelle Timeline-ID. Ebenfalls nützlich, um zu prüfen, ob der Standby mit dem Primary konsistent ist.

`xlogpos` (`text`)

Die aktuelle WAL-Flush-Position. Sie ist nützlich, um eine bekannte Stelle im Write-Ahead-Log zu erhalten, an der das Streaming beginnen kann.

`dbname` (`text`)

Die verbundene Datenbank oder null.

`SHOW name`

Fordert den Server auf, die aktuelle Einstellung eines Laufzeitparameters zu senden. Dies ähnelt dem SQL-Befehl `SHOW`.

`name`

Der Name eines Laufzeitparameters. Die verfügbaren Parameter sind in [Kapitel 19](19_Serverkonfiguration.md) dokumentiert.

`TIMELINE_HISTORY tli`

Fordert den Server auf, die Timeline-History-Datei für die Timeline `tli` zu senden. Der Server antwortet mit einer Ergebnismenge aus einer einzelnen Zeile, die zwei Felder enthält. Obwohl die Felder als `text` bezeichnet sind, geben sie tatsächlich rohe Bytes ohne Kodierungskonvertierung zurück:

`filename` (`text`)

Dateiname der Timeline-History-Datei, zum Beispiel `00000002.history`.

`content` (`text`)

Inhalt der Timeline-History-Datei.

`CREATE_REPLICATION_SLOT slot_name [ TEMPORARY ] { PHYSICAL | LOGICAL output_plugin } [ ( option [, ...] ) ]`

Erzeugt einen physischen oder logischen Replikations-Slot. Weitere Informationen zu Replikations-Slots finden Sie in Abschnitt 26.2.6.

`slot_name`

Der Name des zu erzeugenden Slots. Er muss ein gültiger Replikations-Slot-Name sein (siehe Abschnitt 26.2.6.1).

`output_plugin`

Der Name des Ausgabe-Plugins, das für die logische Decodierung verwendet wird (siehe [Abschnitt 47.6](47_Logische_Decodierung.md#476-outputplugins-für-logische-decodierung)).

`TEMPORARY`

Gibt an, dass dieser Replikations-Slot temporär ist. Temporäre Slots werden nicht auf Platte gespeichert und bei einem Fehler oder am Ende der Sitzung automatisch gelöscht.

Die folgenden Optionen werden unterstützt:

`TWO_PHASE [ boolean ]`

Wenn `true`, unterstützt dieser logische Replikations-Slot die Decodierung von Two-Phase Commit. Mit dieser Option werden Befehle, die sich auf Two-Phase Commit beziehen, etwa `PREPARE TRANSACTION`, `COMMIT PREPARED` und `ROLLBACK PREPARED`, decodiert und übertragen. Die Transaktion wird zum Zeitpunkt von `PREPARE TRANSACTION` decodiert und übertragen. Der Standardwert ist `false`.

`RESERVE_WAL [ boolean ]`

Wenn `true`, reserviert dieser physische Replikations-Slot WAL sofort. Andernfalls wird WAL erst bei der Verbindung eines Streaming-Replikationsclients reserviert. Der Standardwert ist `false`.

`SNAPSHOT { 'export' | 'use' | 'nothing' }`

Legt fest, was mit dem Snapshot geschehen soll, der während der Initialisierung des logischen Slots erzeugt wird. `export`, der Standardwert, exportiert den Snapshot zur Verwendung in anderen Sitzungen. Diese Option kann nicht innerhalb einer Transaktion verwendet werden. `use` verwendet den Snapshot für die aktuelle Transaktion, die den Befehl ausführt. Diese Option muss in einer Transaktion verwendet werden, und `CREATE_REPLICATION_SLOT` muss der erste Befehl sein, der in dieser Transaktion ausgeführt wird. Schließlich verwendet `nothing` den Snapshot wie üblich nur für die logische Decodierung, tut ansonsten aber nichts mit ihm.

`FAILOVER [ boolean ]`

Wenn `true`, wird der Slot dafür aktiviert, auf die Standbys synchronisiert zu werden, sodass die logische Replikation nach einem Failover wiederaufgenommen werden kann. Der Standardwert ist `false`.

Als Antwort auf diesen Befehl sendet der Server eine einzeilige Ergebnismenge mit den folgenden Feldern:

`slot_name` (`text`)

Der Name des neu erzeugten Replikations-Slots.

`consistent_point` (`text`)

Die WAL-Position, an der der Slot konsistent wurde. Dies ist die früheste Position, ab der das Streaming auf diesem Replikations-Slot beginnen kann.

`snapshot_name` (`text`)

Der Bezeichner des Snapshots, der durch den Befehl exportiert wurde. Der Snapshot ist gültig, bis auf dieser Verbindung ein neuer Befehl ausgeführt oder die Replikationsverbindung geschlossen wird. Null, wenn der erzeugte Slot physisch ist.

`output_plugin` (`text`)

Der Name des Ausgabe-Plugins, das vom neu erzeugten Replikations-Slot verwendet wird. Null, wenn der erzeugte Slot physisch ist.

`CREATE_REPLICATION_SLOT slot_name [ TEMPORARY ] { PHYSICAL [ RESERVE_WAL ] | LOGICAL output_plugin [ EXPORT_SNAPSHOT | NOEXPORT_SNAPSHOT | USE_SNAPSHOT | TWO_PHASE ] }`

Aus Kompatibilitätsgründen mit älteren Versionen wird diese alternative Syntax für den Befehl `CREATE_REPLICATION_SLOT` weiterhin unterstützt.

`ALTER_REPLICATION_SLOT slot_name ( option [, ...] )`

Ändert die Definition eines Replikations-Slots. Weitere Informationen zu Replikations-Slots finden Sie in Abschnitt 26.2.6. Dieser Befehl wird derzeit nur für logische Replikations-Slots unterstützt.

`slot_name`

Der Name des zu ändernden Slots. Er muss ein gültiger Replikations-Slot-Name sein (siehe Abschnitt 26.2.6.1).

Die folgenden Optionen werden unterstützt:

`TWO_PHASE [ boolean ]`

Wenn `true`, unterstützt dieser logische Replikations-Slot die Decodierung von Two-Phase Commit. Mit dieser Option werden Befehle, die sich auf Two-Phase Commit beziehen, etwa `PREPARE TRANSACTION`, `COMMIT PREPARED` und `ROLLBACK PREPARED`, decodiert und übertragen. Die Transaktion wird zum Zeitpunkt von `PREPARE TRANSACTION` decodiert und übertragen.

`FAILOVER [ boolean ]`

Wenn `true`, wird der Slot dafür aktiviert, auf die Standbys synchronisiert zu werden, sodass die logische Replikation nach einem Failover wiederaufgenommen werden kann.

`READ_REPLICATION_SLOT slot_name`

Liest einige Informationen, die mit einem Replikations-Slot verbunden sind. Gibt ein Tupel mit NULL-Werten zurück, wenn der Replikations-Slot nicht existiert. Dieser Befehl wird derzeit nur für physische Replikations-Slots unterstützt.

Als Antwort auf diesen Befehl gibt der Server eine einzeilige Ergebnismenge mit den folgenden Feldern zurück:

`slot_type` (`text`)

Der Typ des Replikations-Slots, entweder `physical` oder `NULL`.

`restart_lsn` (`text`)

Das `restart_lsn` des Replikations-Slots.

`restart_tli` (`int8`)

Die Timeline-ID, die mit `restart_lsn` verbunden ist und der aktuellen Timeline-Historie folgt.

`START_REPLICATION [ SLOT slot_name ] [ PHYSICAL ] XXX/XXX [ TIMELINE tli ]`

Weist den Server an, mit dem Streaming von WAL ab der WAL-Position `XXX/XXX` zu beginnen. Wenn die Option `TIMELINE` angegeben ist, beginnt das Streaming auf der Timeline `tli`; andernfalls wird die aktuelle Timeline des Servers gewählt. Der Server kann mit einem Fehler antworten, zum Beispiel wenn der angeforderte WAL-Abschnitt bereits recycelt wurde. Bei Erfolg antwortet der Server mit einer `CopyBothResponse`-Nachricht und beginnt anschließend, WAL an das Frontend zu streamen.

Wenn über `slot_name` ein Slot-Name angegeben wird, wird dieser im Verlauf der Replikation aktualisiert, damit der Server weiß, welche WAL-Segmente und, falls `hot_standby_feedback` eingeschaltet ist, welche Transaktionen vom Standby noch benötigt werden.

Wenn der Client eine Timeline anfordert, die nicht die neueste ist, aber zur Historie des Servers gehört, streamt der Server das gesamte WAL auf dieser Timeline vom angeforderten Startpunkt bis zu dem Punkt, an dem der Server zu einer anderen Timeline gewechselt ist. Wenn der Client Streaming exakt am Ende einer alten Timeline anfordert, überspringt der Server den COPY-Modus vollständig.

Nachdem der Server alles WAL auf einer Timeline gestreamt hat, die nicht die neueste ist, beendet er das Streaming, indem er den COPY-Modus verlässt. Wenn der Client dies bestätigt, indem er ebenfalls den COPY-Modus verlässt, sendet der Server eine Ergebnismenge mit einer Zeile und zwei Spalten, die die nächste Timeline in der Historie dieses Servers angeben. Die erste Spalte ist die ID der nächsten Timeline (Typ `int8`), und die zweite Spalte ist die WAL-Position, an der der Wechsel stattfand (Typ `text`). Normalerweise ist die Wechselposition das Ende des gestreamten WAL, aber es gibt Randfälle, in denen der Server etwas WAL von der alten Timeline senden kann, das er selbst vor der Promotion noch nicht wiedergegeben hat. Abschließend sendet der Server zwei `CommandComplete`-Nachrichten (eine beendet `CopyData`, die andere `START_REPLICATION` selbst) und ist bereit, einen neuen Befehl anzunehmen.

WAL-Daten werden als Folge von `CopyData`-Nachrichten gesendet; Details finden Sie in [Abschnitt 54.6](54_Frontend_Backend_Protokoll.md#546-nachrichtendatentypen) und [Abschnitt 54.7](54_Frontend_Backend_Protokoll.md#547-nachrichtenformate). Dadurch können andere Informationen eingeflochten werden; insbesondere kann der Server eine `ErrorResponse`-Nachricht senden, wenn nach dem Beginn des Streamings ein Fehler auftritt. Die Nutzlast jeder `CopyData`-Nachricht vom Server zum Client enthält eine Nachricht in einem der folgenden Formate:

`XLogData` (B)

`Byte1('w')`

Identifiziert die Nachricht als WAL-Daten.

`Int64`

Der Startpunkt der WAL-Daten in dieser Nachricht.

`Int64`

Das aktuelle WAL-Ende auf dem Server.

`Int64`

Die Systemuhr des Servers zum Zeitpunkt der Übertragung, in Mikrosekunden seit Mitternacht am 2000-01-01.

`Byten`

Ein Abschnitt des WAL-Datenstroms.

Ein einzelner WAL-Datensatz wird nie auf zwei `XLogData`-Nachrichten aufgeteilt. Wenn ein WAL-Datensatz eine WAL-Seitengrenze überschreitet und daher bereits mit Fortsetzungsdatensätzen aufgeteilt ist, kann er an der Seitengrenze aufgeteilt werden. Anders gesagt: Der erste Haupt-WAL-Datensatz und seine Fortsetzungsdatensätze können in unterschiedlichen `XLogData`-Nachrichten gesendet werden.

`Primary keepalive message` (B)

`Byte1('k')`

Identifiziert die Nachricht als Keepalive des Senders.

`Int64`

Das aktuelle WAL-Ende auf dem Server.

`Int64`

Die Systemuhr des Servers zum Zeitpunkt der Übertragung, in Mikrosekunden seit Mitternacht am 2000-01-01.

`Byte1`

`1` bedeutet, dass der Client so bald wie möglich auf diese Nachricht antworten soll, um eine Trennung wegen Zeitüberschreitung zu vermeiden. Andernfalls `0`.

Der empfangende Prozess kann dem Sender jederzeit Antworten zurücksenden, wobei eines der folgenden Nachrichtenformate verwendet wird (ebenfalls in der Nutzlast einer `CopyData`-Nachricht):

`Standby status update` (F)

`Byte1('r')`

Identifiziert die Nachricht als Statusaktualisierung des Empfängers.

`Int64`

Die Position des letzten WAL-Bytes + 1, das im Standby empfangen und auf Platte geschrieben wurde.

`Int64`

Die Position des letzten WAL-Bytes + 1, das im Standby auf Platte geflusht wurde.

`Int64`

Die Position des letzten WAL-Bytes + 1, das im Standby angewendet wurde.

`Int64`

Die Systemuhr des Clients zum Zeitpunkt der Übertragung, in Mikrosekunden seit Mitternacht am 2000-01-01.

`Byte1`

Wenn `1`, fordert der Client den Server auf, sofort auf diese Nachricht zu antworten. Dies kann verwendet werden, um den Server anzupingen und zu prüfen, ob die Verbindung noch gesund ist.

`Hot standby feedback message` (F)

`Byte1('h')`

Identifiziert die Nachricht als Hot-Standby-Feedback-Nachricht.

`Int64`

Die Systemuhr des Clients zum Zeitpunkt der Übertragung, in Mikrosekunden seit Mitternacht am 2000-01-01.

`Int32`

Das aktuelle globale `xmin` des Standbys, ohne das `catalog_xmin` aus Replikations-Slots. Wenn sowohl dieser Wert als auch das folgende `catalog_xmin` 0 sind, wird dies als Benachrichtigung behandelt, dass über diese Verbindung kein Hot-Standby-Feedback mehr gesendet wird. Spätere Nachrichten mit Werten ungleich null können den Feedback-Mechanismus wieder initiieren.

`Int32`

Die Epoche der globalen `xmin`-XID auf dem Standby.

`Int32`

Das niedrigste `catalog_xmin` aller Replikations-Slots auf dem Standby. Wird auf 0 gesetzt, wenn auf dem Standby kein `catalog_xmin` existiert oder wenn Hot-Standby-Feedback deaktiviert wird.

`Int32`

Die Epoche der `catalog_xmin`-XID auf dem Standby.

`START_REPLICATION SLOT slot_name LOGICAL XXX/XXX [ ( option_name [ option_value ] [, ...] ) ]`

Weist den Server an, mit dem Streaming von WAL für logische Replikation zu beginnen, und zwar entweder ab der WAL-Position `XXX/XXX` oder ab dem `confirmed_flush_lsn` des Slots (siehe [Abschnitt 53.20](53_System_Views.md#5320-pgreplicationslots)), je nachdem, welcher Wert größer ist. Dieses Verhalten erleichtert es Clients, eine Aktualisierung ihres lokalen LSN-Status zu vermeiden, wenn keine Daten zu verarbeiten sind. Wenn allerdings an einer anderen LSN als angefordert begonnen wird, werden bestimmte Arten von Clientfehlern möglicherweise nicht erkannt; der Client kann daher prüfen wollen, ob `confirmed_flush_lsn` seinen Erwartungen entspricht, bevor er `START_REPLICATION` ausgibt.

Der Server kann mit einem Fehler antworten, zum Beispiel wenn der Slot nicht existiert. Bei Erfolg antwortet der Server mit einer `CopyBothResponse`-Nachricht und beginnt anschließend, WAL an das Frontend zu streamen.

Die Nachrichten innerhalb der `CopyBothResponse`-Nachrichten haben dasselbe Format, das für `START_REPLICATION ... PHYSICAL` dokumentiert ist, einschließlich zweier `CommandComplete`-Nachrichten.

Das Ausgabe-Plugin, das dem ausgewählten Slot zugeordnet ist, wird verwendet, um die Ausgabe für das Streaming zu verarbeiten.

`SLOT slot_name`

Der Name des Slots, aus dem Änderungen gestreamt werden. Dieser Parameter ist erforderlich und muss einem existierenden logischen Replikations-Slot entsprechen, der mit `CREATE_REPLICATION_SLOT` im Modus `LOGICAL` erzeugt wurde.

`XXX/XXX`

Die WAL-Position, an der das Streaming beginnen soll.

`option_name`

Der Name einer Option, die an das Ausgabe-Plugin für logische Decodierung des Slots übergeben wird. Optionen, die vom Standard-Plugin (`pgoutput`) akzeptiert werden, finden Sie in [Abschnitt 54.5](54_Frontend_Backend_Protokoll.md#545-logisches-streamingreplikationsprotokoll).

`option_value`

Optionaler Wert in Form einer String-Konstante, der der angegebenen Option zugeordnet ist.

`DROP_REPLICATION_SLOT slot_name [ WAIT ]`

Löscht einen Replikations-Slot und gibt alle reservierten serverseitigen Ressourcen frei.

`slot_name`

Der Name des zu löschenden Slots.

`WAIT`

Diese Option bewirkt, dass der Befehl wartet, falls der Slot aktiv ist, bis er inaktiv wird, statt wie im Standardverhalten einen Fehler auszulösen.

`UPLOAD_MANIFEST`

Lädt ein Backup-Manifest hoch, um ein inkrementelles Backup vorzubereiten.

`BASE_BACKUP [ ( option [, ...] ) ]`

Weist den Server an, mit dem Streaming eines Basis-Backups zu beginnen. Das System wird automatisch in den Backup-Modus versetzt, bevor das Backup beginnt, und nach Abschluss des Backups wieder herausgenommen. Die folgenden Optionen werden akzeptiert:

`LABEL 'label'`

Setzt das Label des Backups. Wenn keines angegeben ist, wird ein Backup-Label `base backup` verwendet. Für das Quoting des Labels gelten dieselben Regeln wie für einen Standard-SQL-String mit aktiviertem `standard_conforming_strings`.

`TARGET 'target'`

Teilt dem Server mit, wohin das Backup gesendet werden soll. Wenn das Ziel `client` ist, was der Standard ist, werden die Backup-Daten an den Client gesendet. Wenn es `server` ist, werden die Backup-Daten auf dem Server unter dem Pfadnamen geschrieben, der durch die Option `TARGET_DETAIL` angegeben wird. Wenn es `blackhole` ist, werden die Backup-Daten nirgendwohin gesendet, sondern einfach verworfen.

Das Ziel `server` erfordert Superuser-Rechte oder die Gewährung der Rolle `pg_write_server_files`.

`TARGET_DETAIL 'detail'`

Stellt zusätzliche Informationen zum Backup-Ziel bereit.

Derzeit kann diese Option nur verwendet werden, wenn das Backup-Ziel `server` ist. Sie gibt das Serververzeichnis an, in das das Backup geschrieben werden soll.

`PROGRESS [ boolean ]`

Wenn auf `true` gesetzt, werden Informationen angefordert, die zur Erzeugung eines Fortschrittsberichts benötigt werden. Dadurch wird im Header jedes Tablespaces eine ungefähre Größe zurückgesendet, mit der berechnet werden kann, wie weit der Stream fortgeschritten ist. Dies wird berechnet, indem alle Dateigrößen einmal aufgezählt werden, bevor die Übertragung überhaupt begonnen hat, und kann daher die Leistung beeinträchtigen. Insbesondere kann es länger dauern, bis die ersten Daten gestreamt werden. Da sich die Datenbankdateien während des Backups ändern können, ist die Größe nur näherungsweise und kann zwischen dem Zeitpunkt der Schätzung und dem Senden der tatsächlichen Dateien sowohl wachsen als auch schrumpfen. Der Standardwert ist `false`.

`CHECKPOINT { 'fast' | 'spread' }`

Setzt den Typ des Checkpoints, der zu Beginn des Basis-Backups ausgeführt werden soll. Der Standardwert ist `spread`.

`WAL [ boolean ]`

Wenn auf `true` gesetzt, werden die erforderlichen WAL-Segmente in das Backup aufgenommen. Dazu gehören alle Dateien zwischen Start und Ende des Backups im Verzeichnis `pg_wal` der Tar-Datei des Basisverzeichnisses. Der Standardwert ist `false`.

`WAIT [ boolean ]`

Wenn auf `true` gesetzt, wartet das Backup, bis das letzte benötigte WAL-Segment archiviert wurde, oder gibt eine Warnung aus, wenn WAL-Archivierung nicht aktiviert ist. Wenn `false`, wartet das Backup weder noch warnt es; der Client ist dann dafür verantwortlich sicherzustellen, dass das benötigte Log verfügbar ist. Der Standardwert ist `true`.

`COMPRESSION 'method'`

Weist den Server an, das Backup mit der angegebenen Methode zu komprimieren. Derzeit werden die Methoden `gzip`, `lz4` und `zstd` unterstützt.

`COMPRESSION_DETAIL detail`

Gibt Details für die gewählte Kompressionsmethode an. Dies sollte nur zusammen mit der Option `COMPRESSION` verwendet werden. Wenn der Wert eine ganze Zahl ist, gibt er die Kompressionsstufe an. Andernfalls sollte es eine durch Kommas getrennte Liste von Elementen sein, jeweils in der Form `keyword` oder `keyword=value`. Derzeit werden die Schlüsselwörter `level`, `long` und `workers` unterstützt.

Das Schlüsselwort `level` setzt die Kompressionsstufe. Für `gzip` sollte die Kompressionsstufe eine ganze Zahl zwischen 1 und 9 sein (Standard `Z_DEFAULT_COMPRESSION` oder -1), für `lz4` eine ganze Zahl zwischen 1 und 12 (Standard 0 für schnellen Kompressionsmodus) und für `zstd` eine ganze Zahl zwischen `ZSTD_minCLevel()` (üblicherweise -131072) und `ZSTD_maxCLevel()` (üblicherweise 22), Standard `ZSTD_CLEVEL_DEFAULT` oder 3.

Das Schlüsselwort `long` aktiviert den Long-Distance-Matching-Modus für ein besseres Kompressionsverhältnis auf Kosten höheren Speicherverbrauchs. Der Long-Distance-Modus wird nur für `zstd` unterstützt.

Das Schlüsselwort `workers` setzt die Anzahl der Threads, die für parallele Kompression verwendet werden sollen. Parallele Kompression wird nur für `zstd` unterstützt.

`MAX_RATE rate`

Begrenzt (drosselt) die maximale Datenmenge, die pro Zeiteinheit vom Server zum Client übertragen wird. Die erwartete Einheit ist Kilobyte pro Sekunde. Wenn diese Option angegeben ist, muss der Wert entweder null sein oder im Bereich von 32 kB bis einschließlich 1 GB liegen. Wenn null übergeben wird oder die Option nicht angegeben ist, wird keine Beschränkung für die Übertragung auferlegt.

`TABLESPACE_MAP [ boolean ]`

Wenn `true`, werden Informationen über symbolische Links im Verzeichnis `pg_tblspc` in eine Datei namens `tablespace_map` aufgenommen. Die Tablespace-Map-Datei enthält jeden Namen eines symbolischen Links, wie er im Verzeichnis `pg_tblspc/` existiert, sowie den vollständigen Pfad dieses symbolischen Links. Der Standardwert ist `false`.

`VERIFY_CHECKSUMS [ boolean ]`

Wenn `true`, werden Checksummen während eines Basis-Backups geprüft, falls sie aktiviert sind. Wenn `false`, wird dies übersprungen. Der Standardwert ist `true`.

`MANIFEST manifest_option`

Wenn diese Option mit dem Wert `yes` oder `force-encode` angegeben wird, wird ein Backup-Manifest erzeugt und zusammen mit dem Backup gesendet. Das Manifest ist eine Liste jeder im Backup vorhandenen Datei, mit Ausnahme etwaiger WAL-Dateien, die enthalten sein können. Außerdem speichert es Größe, Zeitpunkt der letzten Änderung und optional eine Prüfsumme für jede Datei. Ein Wert von `force-encode` erzwingt, dass alle Dateinamen hexadezimal kodiert werden; andernfalls wird diese Kodierung nur für Dateien durchgeführt, deren Namen keine UTF-8-Oktettsequenzen sind. `force-encode` ist in erster Linie für Testzwecke gedacht, um sicherzustellen, dass Clients, die das Backup-Manifest lesen, diesen Fall verarbeiten können. Aus Kompatibilitätsgründen mit früheren Versionen ist der Standard `MANIFEST 'no'`.

`MANIFEST_CHECKSUMS checksum_algorithm`

Gibt den Prüfsummenalgorithmus an, der auf jede im Backup-Manifest enthaltene Datei angewendet werden soll. Derzeit sind die Algorithmen `NONE`, `CRC32C`, `SHA224`, `SHA256`, `SHA384` und `SHA512` verfügbar. Der Standardwert ist `CRC32C`.

`INCREMENTAL`

Fordert ein inkrementelles Backup an. Der Befehl `UPLOAD_MANIFEST` muss ausgeführt werden, bevor ein Basis-Backup mit dieser Option gestartet wird.

Wenn das Backup gestartet wird, sendet der Server zuerst zwei gewöhnliche Ergebnismengen, gefolgt von einem oder mehreren `CopyOutResponse`-Ergebnissen.

Die erste gewöhnliche Ergebnismenge enthält die Startposition des Backups in einer einzelnen Zeile mit zwei Spalten. Die erste Spalte enthält die Startposition im Format `XLogRecPtr`, und die zweite Spalte enthält die zugehörige Timeline-ID.

Die zweite gewöhnliche Ergebnismenge hat eine Zeile für jeden Tablespace. Die Felder in dieser Zeile sind:

`spcoid` (`oid`)

Die OID des Tablespaces oder null, wenn es das Basisverzeichnis ist.

`spclocation` (`text`)

Der vollständige Pfad des Tablespace-Verzeichnisses oder null, wenn es das Basisverzeichnis ist.

`size` (`int8`)

Die ungefähre Größe des Tablespaces in Kilobyte (1024 Bytes), wenn ein Fortschrittsbericht angefordert wurde; andernfalls ist sie null.

Nach der zweiten gewöhnlichen Ergebnismenge wird eine `CopyOutResponse` gesendet. Die Nutzlast jeder `CopyData`-Nachricht enthält eine Nachricht in einem der folgenden Formate:

`new archive` (B)

`Byte1('n')`

Identifiziert die Nachricht als Beginn eines neuen Archivs. Es gibt ein Archiv für das Hauptdatenverzeichnis und eines für jeden zusätzlichen Tablespace; jedes verwendet das Tar-Format (gemäß dem im POSIX-Standard 1003.1-2008 angegebenen »ustar interchange format«).

`String`

Der Dateiname für dieses Archiv.

`String`

Für das Hauptdatenverzeichnis ein leerer String. Für andere Tablespaces der vollständige Pfad zu dem Verzeichnis, aus dem dieses Archiv erzeugt wurde.

`manifest` (B)

`Byte1('m')`

Identifiziert die Nachricht als Beginn des Backup-Manifests.

`archive or manifest data` (B)

`Byte1('d')`

Identifiziert die Nachricht als Archiv- oder Manifestdaten enthaltend.

`Byten`

Datenbytes.

`progress report` (B)

`Byte1('p')`

Identifiziert die Nachricht als Fortschrittsbericht.

`Int64`

Die Anzahl der Bytes aus dem aktuellen Tablespace, deren Verarbeitung abgeschlossen ist.

Nachdem die `CopyOutResponse` oder alle solchen Antworten gesendet wurden, wird eine abschließende gewöhnliche Ergebnismenge gesendet, die die WAL-Endposition des Backups im selben Format wie die Startposition enthält.

Das Tar-Archiv für das Datenverzeichnis und jeden Tablespace enthält alle Dateien in den Verzeichnissen, unabhängig davon, ob es PostgreSQL-Dateien oder andere Dateien sind, die demselben Verzeichnis hinzugefügt wurden. Ausgeschlossen sind nur:

- `postmaster.pid`
- `postmaster.opts`
- `pg_internal.init` (in mehreren Verzeichnissen vorhanden)
- Verschiedene temporäre Dateien und Verzeichnisse, die während des Betriebs des PostgreSQL-Servers erzeugt werden, etwa alle Dateien oder Verzeichnisse, die mit `pgsql_tmp` beginnen, sowie temporäre Relationen.
- Ungeloggte Relationen, mit Ausnahme des Init-Forks, der benötigt wird, um die (leere) ungeloggte Relation bei der Wiederherstellung neu zu erzeugen.
- `pg_wal`, einschließlich Unterverzeichnissen. Wenn das Backup mit eingeschlossenen WAL-Dateien ausgeführt wird, wird eine synthetisierte Version von `pg_wal` aufgenommen, die jedoch nur die Dateien enthält, die für die Funktionsfähigkeit des Backups nötig sind, nicht den übrigen Inhalt.
- `pg_dynshmem`, `pg_notify`, `pg_replslot`, `pg_serial`, `pg_snapshots`, `pg_stat_tmp` und `pg_subtrans` werden als leere Verzeichnisse kopiert (selbst wenn sie symbolische Links sind).
- Dateien, die keine regulären Dateien oder Verzeichnisse sind, etwa symbolische Links (außer für die oben aufgeführten Verzeichnisse) sowie spezielle Geräte- und Betriebssystemdateien, werden übersprungen. Symbolische Links in `pg_tblspc` bleiben erhalten.

Eigentümer, Gruppe und Dateimodus werden gesetzt, wenn das zugrunde liegende Dateisystem auf dem Server dies unterstützt.

In allen oben genannten Befehlen kann bei der Angabe eines Parameters vom Typ `boolean` der Wertteil weggelassen werden; dies entspricht der Angabe von `TRUE`.

## 54.5. Logisches Streaming-Replikationsprotokoll

Dieser Abschnitt beschreibt das logische Replikationsprotokoll, also den Nachrichtenfluss, der durch den Replikationsbefehl `START_REPLICATION SLOT slot_name LOGICAL` gestartet wird.

Das logische Streaming-Replikationsprotokoll baut auf den Primitiven des physischen Streaming-Replikationsprotokolls auf.

Die logische Decodierung von PostgreSQL unterstützt Ausgabe-Plugins. `pgoutput` ist das Standard-Plugin, das für die eingebaute logische Replikation verwendet wird.

### 54.5.1. Parameter der logischen Streaming-Replikation

Mit dem Befehl `START_REPLICATION` akzeptiert `pgoutput` die folgenden Optionen:

`proto_version`

Protokollversion. Derzeit werden die Versionen 1, 2, 3 und 4 unterstützt. Eine gültige Version ist erforderlich.

Version 2 wird nur für Serverversion 14 und höher unterstützt und erlaubt das Streaming großer laufender Transaktionen.

Version 3 wird nur für Serverversion 15 und höher unterstützt und erlaubt das Streaming von Two-Phase Commits.

Version 4 wird nur für Serverversion 16 und höher unterstützt und erlaubt, Streams großer laufender Transaktionen parallel anzuwenden.

`publication_names`

Kommagetrennte Liste der Publikationsnamen, die abonniert werden sollen (Änderungen empfangen). Die einzelnen Publikationsnamen werden als normale Objektnamen behandelt und können bei Bedarf entsprechend gequotet werden. Mindestens ein Publikationsname ist erforderlich.

`binary`

Boolesche Option, um den binären Übertragungsmodus zu verwenden. Der Binärmodus ist schneller als der Textmodus, aber etwas weniger robust.

`messages`

Boolesche Option, um das Senden der Nachrichten zu aktivieren, die von `pg_logical_emit_message` geschrieben werden.

`streaming`

Option, um Streaming laufender Transaktionen zu aktivieren. Gültige Werte sind `off` (der Standard), `on` und `parallel`. Die Einstellung `parallel` aktiviert das Senden zusätzlicher Informationen mit einigen Nachrichten, die zur Parallelisierung verwendet werden. Zum Einschalten ist mindestens Protokollversion 2 erforderlich. Für den Wert `parallel` ist mindestens Protokollversion 4 erforderlich.

`two_phase`

Boolesche Option, um Two-Phase-Transaktionen zu aktivieren. Zum Einschalten ist mindestens Protokollversion 3 erforderlich.

`origin`

Option, um Änderungen nach ihrem Ursprung zu senden. Mögliche Werte sind `none`, um nur Änderungen zu senden, denen kein Ursprung zugeordnet ist, oder `any`, um Änderungen unabhängig von ihrem Ursprung zu senden. Dies kann verwendet werden, um Schleifen (unendliche Replikation derselben Daten) zwischen Replikationsknoten zu vermeiden.

### 54.5.2. Nachrichten des logischen Replikationsprotokolls

Die einzelnen Protokollnachrichten werden in den folgenden Unterabschnitten besprochen. Einzelne Nachrichten sind in [Abschnitt 54.9](54_Frontend_Backend_Protokoll.md#549-nachrichtenformate-der-logischen-replikation) beschrieben.

Alle Protokollnachrichten der obersten Ebene beginnen mit einem Nachrichtentyp-Byte. Obwohl es im Code als Zeichen dargestellt wird, ist es ein vorzeichenbehaftetes Byte ohne zugeordnete Kodierung.

Da das Streaming-Replikationsprotokoll eine Nachrichtenlänge bereitstellt, müssen Protokollnachrichten der obersten Ebene keine Länge in ihren Header einbetten.

### 54.5.3. Nachrichtenfluss des logischen Replikationsprotokolls

Mit Ausnahme des Befehls `START_REPLICATION` und der Nachrichten zum Replay-Fortschritt fließen alle Informationen nur vom Backend zum Frontend.

Das logische Replikationsprotokoll sendet einzelne Transaktionen nacheinander. Das bedeutet, dass alle Nachrichten zwischen einem Paar aus `Begin`- und `Commit`-Nachrichten zur selben Transaktion gehören. Entsprechend gehören alle Nachrichten zwischen einem Paar aus `Begin Prepare`- und `Prepare`-Nachrichten zur selben Transaktion. Außerdem sendet es Änderungen großer laufender Transaktionen zwischen einem Paar aus `Stream Start`- und `Stream Stop`-Nachrichten. Der letzte Stream einer solchen Transaktion enthält eine `Stream Commit`- oder `Stream Abort`-Nachricht.

Jede gesendete Transaktion enthält null oder mehr DML-Nachrichten (`Insert`, `Update`, `Delete`). In einem kaskadierten Aufbau kann sie auch `Origin`-Nachrichten enthalten. Die Origin-Nachricht zeigt an, dass die Transaktion auf einem anderen Replikationsknoten entstanden ist. Da ein Replikationsknoten im Rahmen des logischen Replikationsprotokolls praktisch alles sein kann, ist der Origin-Name der einzige Bezeichner. Es liegt in der Verantwortung des Downstreams, dies nach Bedarf zu behandeln (falls nötig). Die `Origin`-Nachricht wird in der Transaktion immer vor allen DML-Nachrichten gesendet.

Jede DML-Nachricht enthält eine Relations-OID, die die Relation des Publishers identifiziert, auf die eingewirkt wurde. Vor der ersten DML-Nachricht für eine gegebene Relations-OID wird eine `Relation`-Nachricht gesendet, die das Schema dieser Relation beschreibt. Danach wird eine neue `Relation`-Nachricht gesendet, wenn sich die Definition der Relation seit der letzten dafür gesendeten `Relation`-Nachricht geändert hat. Das Protokoll setzt voraus, dass der Client diese Metadaten für so viele Relationen wie nötig behalten kann.

`Relation`-Nachrichten identifizieren Spaltentypen über deren OIDs. Bei einem eingebauten Typ wird angenommen, dass der Client diese Typ-OID lokal nachschlagen kann, sodass keine zusätzlichen Daten benötigt werden. Für eine OID eines nicht eingebauten Typs wird vor der `Relation`-Nachricht eine `Type`-Nachricht gesendet, um den mit dieser OID verbundenen Typnamen bereitzustellen. Ein Client, der die Typen von Relationsspalten gezielt identifizieren muss, sollte daher den Inhalt von `Type`-Nachrichten zwischenspeichern und zuerst diesen Cache konsultieren, um zu sehen, ob die Typ-OID dort definiert ist. Falls nicht, wird die Typ-OID lokal nachgeschlagen.

## 54.6. Nachrichtendatentypen

Dieser Abschnitt beschreibt die Basisdatentypen, die in Nachrichten verwendet werden.

`Intn(i)`

Eine `n`-Bit-Ganzzahl in Netzwerk-Byte-Reihenfolge (höchstwertiges Byte zuerst). Wenn `i` angegeben ist, ist es der exakte Wert, der erscheint; andernfalls ist der Wert variabel. Beispiele: `Int16`, `Int32(42)`.

`Intn[k]`

Ein Array aus `k` `n`-Bit-Ganzzahlen, jede in Netzwerk-Byte-Reihenfolge. Die Array-Länge `k` wird immer durch ein früheres Feld in der Nachricht bestimmt. Beispiel: `Int16[M]`.

`String(s)`

Ein nullterminierter String (String im C-Stil). Für Strings gibt es keine bestimmte Längenbegrenzung. Wenn `s` angegeben ist, ist es der exakte Wert, der erscheint; andernfalls ist der Wert variabel. Beispiele: `String`, `String("user")`.

> **Hinweis:** Es gibt keine vordefinierte Begrenzung für die Länge eines Strings, der vom Backend zurückgegeben werden kann. Eine gute Programmierstrategie für ein Frontend ist, einen erweiterbaren Puffer zu verwenden, sodass alles akzeptiert werden kann, was in den Speicher passt. Wenn das nicht möglich ist, lesen Sie den vollständigen String und verwerfen Sie nachfolgende Zeichen, die nicht in Ihren Puffer fester Größe passen.

`Byten(c)`

Exakt `n` Bytes. Wenn die Feldbreite `n` keine Konstante ist, kann sie immer aus einem früheren Feld in der Nachricht bestimmt werden. Wenn `c` angegeben ist, ist es der exakte Wert. Beispiele: `Byte2`, `Byte1('\n')`.

## 54.7. Nachrichtenformate

Dieser Abschnitt beschreibt das genaue Format jeder Nachricht. Jede Nachricht ist gekennzeichnet, um anzugeben, ob sie von einem Frontend (F), einem Backend (B) oder von beiden (F & B) gesendet werden kann. Beachten Sie: Obwohl jede Nachricht am Anfang eine Byteanzahl enthält, sind die meisten Nachrichten so definiert, dass das Nachrichtenende ohne Bezug auf die Byteanzahl gefunden werden kann. Dies hat historische Gründe, da die ursprüngliche, inzwischen obsolete Protokollversion 2 kein explizites Längenfeld hatte. Außerdem unterstützt dies die Gültigkeitsprüfung.

#### `AuthenticationOk` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32(8)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(0)`: Gibt an, dass die Authentifizierung erfolgreich war.

#### `AuthenticationKerberosV5` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32(8)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(2)`: Gibt an, dass Kerberos-V5-Authentifizierung erforderlich ist.

#### `AuthenticationCleartextPassword` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32(8)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(3)`: Gibt an, dass ein Klartextpasswort erforderlich ist.

#### `AuthenticationMD5Password` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32(12)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(5)`: Gibt an, dass ein MD5-verschlüsseltes Passwort erforderlich ist.

- `Byte4`: Der Salt, der beim Verschlüsseln des Passworts zu verwenden ist.

#### `AuthenticationGSS` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32(8)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(7)`: Gibt an, dass GSSAPI-Authentifizierung erforderlich ist.

#### `AuthenticationGSSContinue` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(8)`: Gibt an, dass diese Nachricht GSSAPI- oder SSPI-Daten enthält.

- `Byten`: GSSAPI- oder SSPI-Authentifizierungsdaten.

#### `AuthenticationSSPI` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32(8)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(9)`: Gibt an, dass SSPI-Authentifizierung erforderlich ist.

#### `AuthenticationSASL` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(10)`: Gibt an, dass SASL-Authentifizierung erforderlich ist.

Der Nachrichtenrumpf ist eine Liste von SASL-Authentifizierungsmechanismen in der bevorzugten Reihenfolge des Servers. Nach dem letzten Namen eines Authentifizierungsmechanismus ist ein Nullbyte als Terminator erforderlich. Für jeden Mechanismus gibt es Folgendes:

- `String`: Name eines SASL-Authentifizierungsmechanismus.

#### `AuthenticationSASLContinue` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(11)`: Gibt an, dass diese Nachricht eine SASL-Challenge enthält.

- `Byten`: SASL-Daten, spezifisch für den verwendeten SASL-Mechanismus.

#### `AuthenticationSASLFinal` (B)

- `Byte1('R')`: Identifiziert die Nachricht als Authentifizierungsanforderung.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(12)`: Gibt an, dass die SASL-Authentifizierung abgeschlossen ist.

- `Byten`: Zusätzliche SASL-Ergebnisdaten, spezifisch für den verwendeten SASL-Mechanismus.

#### `BackendKeyData` (B)

- `Byte1('K')`: Identifiziert die Nachricht als Abbruchschlüsseldaten. Das Frontend muss diese Werte speichern, wenn es später CancelRequest-Nachrichten ausgeben können soll.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32`: Die Prozess-ID dieses Backends.

- `Byten`: Der geheime Schlüssel dieses Backends. Dieses Feld reicht bis zum Ende der Nachricht, das durch das Längenfeld angegeben wird.

Die minimale und maximale Schlüssellänge betragen 4 beziehungsweise 256 Bytes. Der PostgreSQL-Server sendet nur Schlüssel bis 32 Bytes, aber die größere Maximalgröße erlaubt es künftigen Serverversionen sowie Connection Poolern und anderer Middleware, längere Schlüssel zu verwenden. Ein möglicher Anwendungsfall ist, den Schlüssel des Servers um zusätzliche Informationen zu erweitern. Middleware wird daher ebenfalls ermutigt, nicht alle Bytes zu verbrauchen, falls mehrere Middleware-Anwendungen übereinander geschichtet werden und jede den Schlüssel mit zusätzlichen Daten umhüllen kann.

Vor Protokollversion 3.2 war der geheime Schlüssel immer 4 Bytes lang.

#### `Bind` (F)

- `Byte1('B')`: Identifiziert die Nachricht als Bind-Befehl.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Der Name des Zielportals (ein leerer String wählt das unbenannte Portal).

- `String`: Der Name der vorbereiteten Quellanweisung (ein leerer String wählt die unbenannte vorbereitete Anweisung).

- `Int16`: Die Anzahl der folgenden Parameter-Format-Codes (unten mit C bezeichnet). Sie kann null sein, um anzugeben, dass es keine Parameter gibt oder dass alle Parameter das Standardformat (Text) verwenden; oder eins, wobei der angegebene Format-Code auf alle Parameter angewendet wird; oder sie kann der tatsächlichen Anzahl der Parameter entsprechen.

- `Int16[C]`: Die Parameter-Format-Codes. Jeder muss derzeit null (Text) oder eins (Binär) sein.

- `Int16`: Die Anzahl der folgenden Parameterwerte (möglicherweise null). Sie muss der Anzahl der von der Abfrage benötigten Parameter entsprechen.

Als Nächstes erscheint für jeden Parameter das folgende Feldpaar:

- `Int32`: Die Länge des Parameterwerts in Bytes (diese Zählung schließt sich selbst nicht ein). Kann null sein. Als Sonderfall zeigt -1 einen NULL-Parameterwert an. Im NULL-Fall folgen keine Wert-Bytes.

- `Byten`: Der Wert des Parameters in dem Format, das durch den zugehörigen Format-Code angegeben wird. n ist die oben angegebene Länge.

Nach dem letzten Parameter erscheinen die folgenden Felder:

- `Int16`: Die Anzahl der folgenden Format-Codes für Ergebnisspalten (unten mit R bezeichnet). Sie kann null sein, um anzugeben, dass es keine Ergebnisspalten gibt oder dass alle Ergebnisspalten das Standardformat (Text) verwenden sollen; oder eins, wobei der angegebene Format-Code auf alle Ergebnisspalten angewendet wird (falls vorhanden); oder sie kann der tatsächlichen Anzahl der Ergebnisspalten der Abfrage entsprechen.

- `Int16[R]`: Die Format-Codes der Ergebnisspalten. Jeder muss derzeit null (Text) oder eins (Binär) sein.

#### `BindComplete` (B)

- `Byte1('2')`: Identifiziert die Nachricht als Bind-abgeschlossen-Anzeige.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `CancelRequest` (F)

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(80877102)`: Der Abbruchanforderungs-Code. Der Wert ist so gewählt, dass er 1234 in den höchstwertigen 16 Bits und 5678 in den niedrigstwertigen 16 Bits enthält. (Um Verwechslungen zu vermeiden, darf dieser Code nicht mit einer Protokollversionsnummer identisch sein.)

- `Int32`: Die Prozess-ID des Ziel-Backends.

- `Byten`: Der geheime Schlüssel für das Ziel-Backend. Dieses Feld reicht bis zum Ende der Nachricht, das durch das Längenfeld angegeben wird. Die maximale Schlüssellänge beträgt 256 Bytes.

Vor Protokollversion 3.2 war der geheime Schlüssel immer 4 Bytes lang.

#### `Close` (F)

- `Byte1('C')`: Identifiziert die Nachricht als Close-Befehl.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Byte1`: `S`, um eine vorbereitete Anweisung zu schließen; oder `P`, um ein Portal zu schließen.

- `String`: Der Name der zu schließenden vorbereiteten Anweisung oder des zu schließenden Portals (ein leerer String wählt die unbenannte vorbereitete Anweisung oder das unbenannte Portal).

#### `CloseComplete` (B)

- `Byte1('3')`: Identifiziert die Nachricht als Close-abgeschlossen-Anzeige.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `CommandComplete` (B)

- `Byte1('C')`: Identifiziert die Nachricht als Antwort, dass ein Befehl abgeschlossen wurde.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Das Befehls-Tag. Dies ist normalerweise ein einzelnes Wort, das angibt, welcher SQL-Befehl abgeschlossen wurde.

Für einen INSERT-Befehl lautet das Tag `INSERT oid rows`, wobei `rows` die Anzahl der eingefügten Zeilen ist. `oid` war früher die Objekt-ID der eingefügten Zeile, wenn `rows` 1 war und die Zieltabelle OIDs hatte; OID-Systemspalten werden jedoch nicht mehr unterstützt, daher ist `oid` immer 0.

Für einen DELETE-Befehl lautet das Tag `DELETE rows`, wobei `rows` die Anzahl der gelöschten Zeilen ist.

Für einen UPDATE-Befehl lautet das Tag `UPDATE rows`, wobei `rows` die Anzahl der aktualisierten Zeilen ist.

Für einen MERGE-Befehl lautet das Tag `MERGE rows`, wobei `rows` die Anzahl der eingefügten, aktualisierten oder gelöschten Zeilen ist.

Für einen SELECT- oder CREATE TABLE AS-Befehl lautet das Tag `SELECT rows`, wobei `rows` die Anzahl der abgerufenen Zeilen ist.

Für einen MOVE-Befehl lautet das Tag `MOVE rows`, wobei `rows` die Anzahl der Zeilen ist, um die die Cursorposition geändert wurde.

Für einen FETCH-Befehl lautet das Tag `FETCH rows`, wobei `rows` die Anzahl der Zeilen ist, die aus dem Cursor abgerufen wurden.

Für einen COPY-Befehl lautet das Tag `COPY rows`, wobei `rows` die Anzahl der kopierten Zeilen ist. (Hinweis: Die Zeilenzahl erscheint nur in PostgreSQL 8.2 und später.)

#### `CopyData` (F & B)

- `Byte1('d')`: Identifiziert die Nachricht als COPY-Daten.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Byten`: Daten, die Teil eines COPY-Datenstroms sind. Vom Backend gesendete Nachrichten entsprechen immer einzelnen Datenzeilen, vom Frontend gesendete Nachrichten können den Datenstrom jedoch beliebig aufteilen.

#### `CopyDone` (F & B)

- `Byte1('c')`: Identifiziert die Nachricht als COPY-abgeschlossen-Anzeige.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `CopyFail` (F)

- `Byte1('f')`: Identifiziert die Nachricht als COPY-Fehler-Anzeige.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Eine Fehlermeldung, die als Ursache des Fehlschlags gemeldet wird.

#### `CopyInResponse` (B)

- `Byte1('G')`: Identifiziert die Nachricht als Start-Copy-In-Antwort. Das Frontend muss nun Copy-In-Daten senden (wenn es dazu nicht bereit ist, sendet es eine CopyFail-Nachricht).

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int8`: 0 gibt an, dass das gesamte COPY-Format textuell ist (Zeilen getrennt durch Newlines, Spalten getrennt durch Trennzeichen usw.). 1 gibt an, dass das gesamte COPY-Format binär ist (ähnlich dem DataRow-Format). Weitere Informationen finden Sie bei COPY.

- `Int16`: Die Anzahl der Spalten in den zu kopierenden Daten (unten mit N bezeichnet).

- `Int16[N]`: Die Format-Codes, die für jede Spalte verwendet werden sollen. Jeder muss derzeit null (Text) oder eins (Binär) sein. Alle müssen null sein, wenn das gesamte COPY-Format textuell ist.

#### `CopyOutResponse` (B)

- `Byte1('H')`: Identifiziert die Nachricht als Start-Copy-Out-Antwort. Auf diese Nachricht folgen Copy-Out-Daten.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int8`: 0 gibt an, dass das gesamte COPY-Format textuell ist (Zeilen getrennt durch Newlines, Spalten getrennt durch Trennzeichen usw.). 1 gibt an, dass das gesamte COPY-Format binär ist (ähnlich dem DataRow-Format). Weitere Informationen finden Sie bei COPY.

- `Int16`: Die Anzahl der Spalten in den zu kopierenden Daten (unten mit N bezeichnet).

- `Int16[N]`: Die Format-Codes, die für jede Spalte verwendet werden sollen. Jeder muss derzeit null (Text) oder eins (Binär) sein. Alle müssen null sein, wenn das gesamte COPY-Format textuell ist.

#### `CopyBothResponse` (B)

- `Byte1('W')`: Identifiziert die Nachricht als Start-Copy-Both-Antwort. Diese Nachricht wird nur für Streaming-Replikation verwendet.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int8`: 0 gibt an, dass das gesamte COPY-Format textuell ist (Zeilen getrennt durch Newlines, Spalten getrennt durch Trennzeichen usw.). 1 gibt an, dass das gesamte COPY-Format binär ist (ähnlich dem DataRow-Format). Weitere Informationen finden Sie bei COPY.

- `Int16`: Die Anzahl der Spalten in den zu kopierenden Daten (unten mit N bezeichnet).

- `Int16[N]`: Die Format-Codes, die für jede Spalte verwendet werden sollen. Jeder muss derzeit null (Text) oder eins (Binär) sein. Alle müssen null sein, wenn das gesamte COPY-Format textuell ist.

#### `DataRow` (B)

- `Byte1('D')`: Identifiziert die Nachricht als Datenzeile.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int16`: Die Anzahl der folgenden Spaltenwerte (möglicherweise null).

Als Nächstes erscheint für jede Spalte das folgende Feldpaar:

- `Int32`: Die Länge des Spaltenwerts in Bytes (diese Zählung schließt sich selbst nicht ein). Kann null sein. Als Sonderfall zeigt -1 einen NULL-Spaltenwert an. Im NULL-Fall folgen keine Wert-Bytes.

- `Byten`: Der Wert der Spalte in dem Format, das durch den zugehörigen Format-Code angegeben wird. n ist die oben angegebene Länge.

#### `Describe` (F)

- `Byte1('D')`: Identifiziert die Nachricht als Describe-Befehl.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Byte1`: `S`, um eine vorbereitete Anweisung zu beschreiben; oder `P`, um ein Portal zu beschreiben.

- `String`: Der Name der zu beschreibenden vorbereiteten Anweisung oder des zu beschreibenden Portals (ein leerer String wählt die unbenannte vorbereitete Anweisung oder das unbenannte Portal).

#### `EmptyQueryResponse` (B)

- `Byte1('I')`: Identifiziert die Nachricht als Antwort auf einen leeren Abfrage-String. (Dies ersetzt `CommandComplete`.)

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `ErrorResponse` (B)

- `Byte1('E')`: Identifiziert die Nachricht als Fehler.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

Der Nachrichtenrumpf besteht aus einem oder mehreren identifizierten Feldern, gefolgt von einem Nullbyte als Terminator. Felder können in beliebiger Reihenfolge erscheinen. Für jedes Feld gibt es Folgendes:

- `Byte1`: Ein Code, der den Feldtyp identifiziert; wenn er null ist, ist dies der Nachrichtenterminator und es folgt kein String. Die derzeit definierten Feldtypen sind in [Abschnitt 54.8](54_Frontend_Backend_Protokoll.md#548-felder-von-fehler-und-hinweismeldungen) aufgeführt. Da künftig weitere Feldtypen hinzugefügt werden können, sollten Frontends Felder unbekannten Typs stillschweigend ignorieren.

- `String`: Der Feldwert.

#### `Execute` (F)

- `Byte1('E')`: Identifiziert die Nachricht als Execute-Befehl.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Der Name des auszuführenden Portals (ein leerer String wählt das unbenannte Portal).

- `Int32`: Maximale Anzahl zurückzugebender Zeilen, wenn das Portal eine Abfrage enthält, die Zeilen zurückgibt (andernfalls ignoriert). Null bedeutet »keine Begrenzung«.

#### `Flush` (F)

- `Byte1('H')`: Identifiziert die Nachricht als Flush-Befehl.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `FunctionCall` (F)

- `Byte1('F')`: Identifiziert die Nachricht als Funktionsaufruf.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32`: Gibt die Objekt-ID der aufzurufenden Funktion an.

- `Int16`: Die Anzahl der folgenden Argument-Format-Codes (unten mit `C` bezeichnet). Sie kann null sein, um anzugeben, dass es keine Argumente gibt oder dass alle Argumente das Standardformat (Text) verwenden; oder eins, wobei der angegebene Format-Code auf alle Argumente angewendet wird; oder sie kann der tatsächlichen Anzahl der Argumente entsprechen.

- `Int16[C]`: Die Argument-Format-Codes. Jeder muss derzeit null (Text) oder eins (Binär) sein.

- `Int16`: Gibt die Anzahl der an die Funktion übergebenen Argumente an.

Als Nächstes erscheint für jedes Argument das folgende Feldpaar:

- `Int32`: Die Länge des Argumentwerts in Bytes (diese Zählung schließt sich selbst nicht ein). Kann null sein. Als Sonderfall zeigt -1 einen NULL-Argumentwert an. Im NULL-Fall folgen keine Wert-Bytes.

- `Byten`: Der Wert des Arguments in dem Format, das durch den zugehörigen Format-Code angegeben wird. `n` ist die oben angegebene Länge.

Nach dem letzten Argument erscheint das folgende Feld:

- `Int16`: Der Format-Code für das Funktionsergebnis. Muss derzeit null (Text) oder eins (Binär) sein.

#### `FunctionCallResponse` (B)

- `Byte1('V')`: Identifiziert die Nachricht als Ergebnis eines Funktionsaufrufs.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32`: Die Länge des Funktionsergebniswerts in Bytes (diese Zählung schließt sich selbst nicht ein). Kann null sein. Als Sonderfall zeigt -1 ein NULL-Funktionsergebnis an. Im NULL-Fall folgen keine Wert-Bytes.

- `Byten`: Der Wert des Funktionsergebnisses in dem Format, das durch den zugehörigen Format-Code angegeben wird. `n` ist die oben angegebene Länge.

#### `GSSENCRequest` (F)

- `Int32(8)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(80877104)`: Der Anforderungscode für GSSAPI-Verschlüsselung. Der Wert ist so gewählt, dass er 1234 in den höchstwertigen 16 Bits und 5680 in den niedrigstwertigen 16 Bits enthält. (Um Verwechslungen zu vermeiden, darf dieser Code nicht mit einer Protokollversionsnummer identisch sein.)

#### `GSSResponse` (F)

- `Byte1('p')`: Identifiziert die Nachricht als GSSAPI- oder SSPI-Antwort. Beachten Sie, dass dies auch für SASL- und Passwortantwortnachrichten verwendet wird. Der genaue Nachrichtentyp kann aus dem Kontext abgeleitet werden.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Byten`: GSSAPI-/SSPI-spezifische Nachrichtendaten.

#### `NegotiateProtocolVersion` (B)

- `Byte1('v')`: Identifiziert die Nachricht als Nachricht zur Aushandlung der Protokollversion.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32`: Neueste Minor-Protokollversion, die der Server für die vom Client angeforderte Major-Protokollversion unterstützt.

- `Int32`: Anzahl der vom Server nicht erkannten Protokolloptionen.

Dann gibt es für jede vom Server nicht erkannte Protokolloption Folgendes:

- `String`: Der Optionsname.

#### `NoData` (B)

- `Byte1('n')`: Identifiziert die Nachricht als Keine-Daten-Anzeige.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `NoticeResponse` (B)

- `Byte1('N')`: Identifiziert die Nachricht als Hinweis.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

Der Nachrichtenrumpf besteht aus einem oder mehreren identifizierten Feldern, gefolgt von einem Nullbyte als Terminator. Felder können in beliebiger Reihenfolge erscheinen. Für jedes Feld gibt es Folgendes:

- `Byte1`: Ein Code, der den Feldtyp identifiziert; wenn er null ist, ist dies der Nachrichtenterminator und es folgt kein String. Die derzeit definierten Feldtypen sind in [Abschnitt 54.8](54_Frontend_Backend_Protokoll.md#548-felder-von-fehler-und-hinweismeldungen) aufgeführt. Da künftig weitere Feldtypen hinzugefügt werden können, sollten Frontends Felder unbekannten Typs stillschweigend ignorieren.

- `String`: Der Feldwert.

#### `NotificationResponse` (B)

- `Byte1('A')`: Identifiziert die Nachricht als Benachrichtigungsantwort.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32`: Die Prozess-ID des benachrichtigenden Backend-Prozesses.

- `String`: Der Name des Kanals, auf dem NOTIFY ausgelöst wurde.

- `String`: Der vom benachrichtigenden Prozess übergebene Payload-String.

#### `ParameterDescription` (B)

- `Byte1('t')`: Identifiziert die Nachricht als Parameterbeschreibung.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int16`: Die Anzahl der von der Anweisung verwendeten Parameter (kann null sein).

Dann gibt es für jeden Parameter Folgendes:

- `Int32`: Gibt die Objekt-ID des Parameterdatentyps an.

#### `ParameterStatus` (B)

- `Byte1('S')`: Identifiziert die Nachricht als Statusbericht eines Laufzeitparameters.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Der Name des gemeldeten Laufzeitparameters.

- `String`: Der aktuelle Wert des Parameters.

#### `Parse` (F)

- `Byte1('P')`: Identifiziert die Nachricht als Parse-Befehl.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Der Name der vorbereiteten Zielanweisung (ein leerer String wählt die unbenannte vorbereitete Anweisung).

- `String`: Der zu parsende Abfrage-String.

- `Int16`: Die Anzahl der angegebenen Parameterdatentypen (kann null sein). Beachten Sie, dass dies kein Hinweis auf die Anzahl der Parameter ist, die im Abfrage-String erscheinen können, sondern nur auf die Anzahl, für die das Frontend Typen vorab festlegen möchte.

Dann gibt es für jeden Parameter Folgendes:

- `Int32`: Gibt die Objekt-ID des Parameterdatentyps an. Eine Null an dieser Stelle entspricht dem Offenlassen des Typs.

#### `ParseComplete` (B)

- `Byte1('1')`: Identifiziert die Nachricht als Parse-abgeschlossen-Anzeige.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `PasswordMessage` (F)

- `Byte1('p')`: Identifiziert die Nachricht als Passwortantwort. Beachten Sie, dass dies auch für GSSAPI-, SSPI- und SASL-Antwortnachrichten verwendet wird. Der genaue Nachrichtentyp kann aus dem Kontext abgeleitet werden.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Das Passwort (verschlüsselt, falls angefordert).

#### `PortalSuspended` (B)

- `Byte1('s')`: Identifiziert die Nachricht als Portal-angehalten-Anzeige. Beachten Sie, dass dies nur erscheint, wenn die Zeilenzahlbegrenzung einer `Execute`-Nachricht erreicht wurde.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `Query` (F)

- `Byte1('Q')`: Identifiziert die Nachricht als einfache Abfrage.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Der Abfrage-String selbst.

#### `ReadyForQuery` (B)

- `Byte1('Z')`: Identifiziert den Nachrichtentyp. ReadyForQuery wird gesendet, sobald das Backend für einen neuen Abfragezyklus bereit ist.

- `Int32(5)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Byte1`: Aktuelle Transaktionsstatusanzeige des Backends. Mögliche Werte sind `I`, wenn es untätig ist (nicht in einem Transaktionsblock), `T`, wenn es sich in einem Transaktionsblock befindet, oder `E`, wenn es sich in einem fehlgeschlagenen Transaktionsblock befindet (Abfragen werden abgewiesen, bis der Block beendet ist).

#### `RowDescription` (B)

- `Byte1('T')`: Identifiziert die Nachricht als Zeilenbeschreibung.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int16`: Gibt die Anzahl der Felder in einer Zeile an (kann null sein).

Dann gibt es für jedes Feld Folgendes:

- `String`: Der Feldname.

- `Int32`: Wenn das Feld als Spalte einer bestimmten Tabelle identifiziert werden kann, die Objekt-ID der Tabelle; andernfalls null.

- `Int16`: Wenn das Feld als Spalte einer bestimmten Tabelle identifiziert werden kann, die Attributnummer der Spalte; andernfalls null.

- `Int32`: Die Objekt-ID des Datentyps des Feldes.

- `Int16`: Die Datentypgröße (siehe pg_type.typlen). Beachten Sie, dass negative Werte Typen variabler Breite bezeichnen.

- `Int32`: Der Typmodifikator (siehe pg_attribute.atttypmod). Die Bedeutung des Modifikators ist typspezifisch.

- `Int16`: Der für das Feld verwendete Format-Code. Derzeit ist dies null (Text) oder eins (Binär). In einer `RowDescription`, die von der Statement-Variante von `Describe` zurückgegeben wird, ist der Format-Code noch nicht bekannt und ist immer null.

#### `SASLInitialResponse` (F)

- `Byte1('p')`: Identifiziert die Nachricht als initiale SASL-Antwort. Beachten Sie, dass dies auch für GSSAPI-, SSPI- und Passwortantwortnachrichten verwendet wird. Der genaue Nachrichtentyp wird aus dem Kontext abgeleitet.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `String`: Name des SASL-Authentifizierungsmechanismus, den der Client ausgewählt hat.

- `Int32`: Länge der folgenden SASL-mechanismusspezifischen »Initial Client Response« oder -1, wenn es keine Initial Response gibt.

- `Byten`: SASL-mechanismusspezifische »Initial Response«.

#### `SASLResponse` (F)

- `Byte1('p')`: Identifiziert die Nachricht als SASL-Antwort. Beachten Sie, dass dies auch für GSSAPI-, SSPI- und Passwortantwortnachrichten verwendet wird. Der genaue Nachrichtentyp kann aus dem Kontext abgeleitet werden.

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Byten`: SASL-mechanismusspezifische Nachrichtendaten.

#### `SSLRequest` (F)

- `Int32(8)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(80877103)`: Der SSL-Anforderungscode. Der Wert ist so gewählt, dass er 1234 in den höchstwertigen 16 Bits und 5679 in den niedrigstwertigen 16 Bits enthält. (Um Verwechslungen zu vermeiden, darf dieser Code nicht mit einer Protokollversionsnummer identisch sein.)

#### `StartupMessage` (F)

- `Int32`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

- `Int32(196610)`: Die Protokollversionsnummer. Die höchstwertigen 16 Bits sind die Major-Versionsnummer (3 für das hier beschriebene Protokoll). Die niedrigstwertigen 16 Bits sind die Minor-Versionsnummer (2 für das hier beschriebene Protokoll).

Auf die Protokollversionsnummer folgen ein oder mehrere Paare aus Parametername- und Wert-Strings. Nach dem letzten Name/Wert-Paar ist ein Nullbyte als Terminator erforderlich. Parameter können in beliebiger Reihenfolge erscheinen. `user` ist erforderlich, andere sind optional. Jeder Parameter wird wie folgt angegeben:

- `String`: Der Parametername. Derzeit erkannte Namen sind:

- `user`
  - Der Datenbankbenutzername, unter dem verbunden werden soll. Erforderlich; es gibt keinen Standardwert.

- `database`
  - Die Datenbank, zu der verbunden werden soll. Standard ist der Benutzername.

- `options`
  - Kommandozeilenargumente für das Backend. (Dies ist zugunsten des Setzens einzelner Laufzeitparameter veraltet.) Leerzeichen innerhalb dieses Strings gelten als Argumenttrenner, außer sie werden mit einem Backslash (`\`) maskiert; schreiben Sie `\\`, um einen literalen Backslash darzustellen.

- `replication`
  - Wird verwendet, um im Streaming-Replikationsmodus zu verbinden, in dem eine kleine Menge von Replikationsbefehlen statt SQL-Anweisungen ausgegeben werden kann. Der Wert kann `true`, `false` oder `database` sein, und der Standard ist `false`. Details finden Sie in [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll).

Zusätzlich zu den obigen können weitere Parameter aufgeführt werden. Parameternamen, die mit `_pq_.` beginnen, sind für die Verwendung als Protokollerweiterungen reserviert, während andere als Laufzeitparameter behandelt werden, die beim Backend-Start gesetzt werden. Solche Einstellungen werden während des Backend-Starts angewendet (nach dem Parsen etwaiger Kommandozeilenargumente) und wirken als Sitzungsstandardwerte.

- `String`: Der Parameterwert.

#### `Sync` (F)

- `Byte1('S')`: Identifiziert die Nachricht als Sync-Befehl.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

#### `Terminate` (F)

- `Byte1('X')`: Identifiziert die Nachricht als Beendigung.

- `Int32(4)`: Länge des Nachrichteninhalts in Bytes, einschließlich dieses Felds.

## 54.8. Felder von Fehler- und Hinweismeldungen

Dieser Abschnitt beschreibt die Felder, die in `ErrorResponse`- und `NoticeResponse`-Nachrichten erscheinen können. Jeder Feldtyp hat ein Identifikationstoken von einem Byte. Beachten Sie, dass ein bestimmter Feldtyp höchstens einmal pro Nachricht erscheinen sollte.

`S`

Severity: Der Feldinhalt ist `ERROR`, `FATAL` oder `PANIC` (in einer Fehlermeldung) oder `WARNING`, `NOTICE`, `DEBUG`, `INFO` oder `LOG` (in einer Hinweismeldung) oder eine lokalisierte Übersetzung eines dieser Werte. Immer vorhanden.

Severity: Der Feldinhalt ist `ERROR`, `FATAL` oder `PANIC` (in einer Fehlermeldung) oder `WARNING`, `NOTICE`, `DEBUG`, `INFO` oder `LOG` (in einer Hinweismeldung). Dies ist mit dem Feld `S` identisch, außer dass der Inhalt nie lokalisiert wird. Dieses Feld ist nur in Nachrichten vorhanden, die von PostgreSQL 9.6 und später erzeugt werden.

Code: Der SQLSTATE-Code für den Fehler (siehe Anhang A). Nicht lokalisierbar. Immer vorhanden.

Message: Die primäre menschenlesbare Fehlermeldung. Sie sollte genau, aber knapp sein (typischerweise eine Zeile). Immer vorhanden.

Detail: Eine optionale sekundäre Fehlermeldung mit weiteren Details zum Problem. Kann über mehrere Zeilen laufen.

`H`

Hint: Ein optionaler Vorschlag, was gegen das Problem zu tun ist. Dies soll sich von Detail dadurch unterscheiden, dass es einen Rat anbietet (möglicherweise unpassend), statt harte Fakten zu liefern. Kann über mehrere Zeilen laufen.

`P`

Position: Der Feldwert ist eine dezimale ASCII-Ganzzahl, die eine Fehler-Cursorposition als Index in den ursprünglichen Abfrage-String angibt. Das erste Zeichen hat Index 1, und Positionen werden in Zeichen gemessen, nicht in Bytes.

`p`

Interne Position: Dies ist genauso definiert wie das Feld `P`, wird aber verwendet, wenn sich die Cursorposition auf einen intern erzeugten Befehl bezieht statt auf den vom Client übermittelten. Das Feld `q` erscheint immer, wenn dieses Feld erscheint.

`q`

Interne Abfrage: Der Text eines fehlgeschlagenen intern erzeugten Befehls. Das kann zum Beispiel eine SQL-Abfrage sein, die von einer PL/pgSQL-Funktion ausgegeben wurde.

`W`

Where: Eine Angabe des Kontexts, in dem der Fehler aufgetreten ist. Derzeit umfasst dies einen Call-Stack-Traceback aktiver prozeduraler Sprachfunktionen und intern erzeugter Abfragen. Der Trace enthält einen Eintrag pro Zeile, den neuesten zuerst.

`s`

Schemaname: Wenn der Fehler mit einem bestimmten Datenbankobjekt verbunden war, der Name des Schemas, das dieses Objekt enthält, falls vorhanden.

`t`

Tabellenname: Wenn der Fehler mit einer bestimmten Tabelle verbunden war, der Name der Tabelle. Den Namen des Tabellenschemas entnehmen Sie dem Feld für den Schemanamen.

Spaltenname: Wenn der Fehler mit einer bestimmten Tabellenspalte verbunden war, der Name der Spalte. Verwenden Sie die Felder für Schema- und Tabellennamen, um die Tabelle zu identifizieren.

Datentypname: Wenn der Fehler mit einem bestimmten Datentyp verbunden war, der Name des Datentyps. Den Namen des Datentypschemas entnehmen Sie dem Feld für den Schemanamen.

`n`

Constraint-Name: Wenn der Fehler mit einem bestimmten Constraint verbunden war, der Name des Constraints. Die zugehörige Tabelle oder Domain entnehmen Sie den oben aufgeführten Feldern. Für diesen Zweck werden Indizes als Constraints behandelt, selbst wenn sie nicht mit Constraint-Syntax erzeugt wurden.

`F`

File: Der Dateiname der Quellcodeposition, an der der Fehler gemeldet wurde.

Line: Die Zeilennummer der Quellcodeposition, an der der Fehler gemeldet wurde.

`R`

Routine: Der Name der Quellcode-Routine, die den Fehler meldet.

> **Hinweis:** Die Felder für Schemaname, Tabellenname, Spaltenname, Datentypname und Constraint-Name werden nur für eine begrenzte Anzahl von Fehlertypen bereitgestellt; siehe Anhang A. Frontends sollten nicht annehmen, dass das Vorhandensein eines dieser Felder das Vorhandensein eines anderen Feldes garantiert. Fehlerquellen im Kern beachten die oben genannten Beziehungen, aber benutzerdefinierte Funktionen können diese Felder anders verwenden. Ebenso sollten Clients nicht annehmen, dass diese Felder aktuelle Objekte in der gegenwärtigen Datenbank bezeichnen.

Der Client ist dafür verantwortlich, angezeigte Informationen passend zu seinen Bedürfnissen zu formatieren; insbesondere sollte er lange Zeilen bei Bedarf umbrechen. Newline-Zeichen in den Fehlermeldungsfeldern sollten als Absatzumbrüche behandelt werden, nicht als Zeilenumbrüche.

## 54.9. Nachrichtenformate der logischen Replikation

Dieser Abschnitt beschreibt das genaue Format jeder logischen Replikationsnachricht. Diese Nachrichten werden entweder von der SQL-Schnittstelle des Replikations-Slots zurückgegeben oder von einem Walsender gesendet. Im Fall eines Walsenders werden sie in WAL-Nachrichten des Replikationsprotokolls gekapselt, wie in [Abschnitt 54.4](54_Frontend_Backend_Protokoll.md#544-streamingreplikationsprotokoll) beschrieben, und folgen im Allgemeinen demselben Nachrichtenfluss wie die physische Replikation.

`Begin`

`Byte1('B')`: Identifiziert die Nachricht als Begin-Nachricht.

`Int64 (XLogRecPtr)`: Die finale LSN der Transaktion.

`Int64 (TimestampTz)`: Commit-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Int32 (TransactionId)`: XID der Transaktion.

`Message`

`Byte1('M')`: Identifiziert die Nachricht als logische Decoding-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion (nur für gestreamte Transaktionen vorhanden). Dieses Feld ist seit Protokollversion 2 verfügbar.

`Int8`: Flags; entweder 0 für keine Flags oder 1, wenn die logische Decoding-Nachricht transaktional ist.

`Int64 (XLogRecPtr)`: Die LSN der logischen Decoding-Nachricht.

`String`: Das Präfix der logischen Decoding-Nachricht.

`Int32`: Länge des Inhalts.

`Byten`: Der Inhalt der logischen Decoding-Nachricht.

`Commit`

`Byte1('C')`: Identifiziert die Nachricht als Commit-Nachricht.

`Int8(0)`: Flags; derzeit unbenutzt.

`Int64 (XLogRecPtr)`: Die LSN des Commits.

`Int64 (XLogRecPtr)`: Die End-LSN der Transaktion.

`Int64 (TimestampTz)`: Commit-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Origin`

`Byte1('O')`: Identifiziert die Nachricht als Origin-Nachricht.

`Int64 (XLogRecPtr)`: Die LSN des Commits auf dem Ursprungsserver.

`String`: Name des Ursprungs.

Beachten Sie, dass es innerhalb einer einzelnen Transaktion mehrere `Origin`-Nachrichten geben kann.

`Relation`

`Byte1('R')`: Identifiziert die Nachricht als Relation-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion (nur für gestreamte Transaktionen vorhanden). Dieses Feld ist seit Protokollversion 2 verfügbar.

`Int32 (Oid)`: OID der Relation.

`String`: Namespace (leerer String für `pg_catalog`).

`String`: Relationsname.

`Int8`: Replica-Identity-Einstellung für die Relation (entspricht `relreplident` in `pg_class`).

`Int16`: Anzahl der Spalten.

Als Nächstes erscheint für jede in der Publikation enthaltene Spalte der folgende Nachrichtenteil:

`Int8`: Flags für die Spalte. Derzeit entweder 0 für keine Flags oder 1, wodurch die Spalte als Teil des Schlüssels markiert wird.

`String`: Name der Spalte.

`Int32 (Oid)`: OID des Datentyps der Spalte.

`Int32`: Typmodifikator der Spalte (`atttypmod`).

`Type`

`Byte1('Y')`: Identifiziert die Nachricht als Type-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion (nur für gestreamte Transaktionen vorhanden). Dieses Feld ist seit Protokollversion 2 verfügbar.

`Int32 (Oid)`: OID des Datentyps.

`String`: Namespace (leerer String für `pg_catalog`).

`String`: Name des Datentyps.

`Insert`

`Byte1('I')`: Identifiziert die Nachricht als Insert-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion (nur für gestreamte Transaktionen vorhanden). Dieses Feld ist seit Protokollversion 2 verfügbar.

`Int32 (Oid)`: OID der Relation, die der ID in der Relation-Nachricht entspricht.

`Byte1('N')`: Identifiziert die folgende `TupleData`-Nachricht als neues Tupel.

`TupleData`: `TupleData`-Nachrichtenteil, der den Inhalt des neuen Tupels darstellt.

`Update`

`Byte1('U')`: Identifiziert die Nachricht als Update-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion (nur für gestreamte Transaktionen vorhanden). Dieses Feld ist seit Protokollversion 2 verfügbar.

`Int32 (Oid)`: OID der Relation, die der ID in der Relation-Nachricht entspricht.

`Byte1('K')`: Identifiziert die folgende `TupleData`-Unternachricht als Schlüssel. Dieses Feld ist optional und nur vorhanden, wenn das Update Daten in einer der Spalten geändert hat, die Teil des `REPLICA IDENTITY`-Index sind.

`Byte1('O')`: Identifiziert die folgende `TupleData`-Unternachricht als altes Tupel. Dieses Feld ist optional und nur vorhanden, wenn die Tabelle, in der das Update stattfand, `REPLICA IDENTITY` auf `FULL` gesetzt hat.

`TupleData`: `TupleData`-Nachrichtenteil, der den Inhalt des alten Tupels oder Primärschlüssels darstellt. Nur vorhanden, wenn der vorherige Teil `O` oder `K` vorhanden ist.

`Byte1('N')`: Identifiziert die folgende `TupleData`-Nachricht als neues Tupel.

`TupleData`: `TupleData`-Nachrichtenteil, der den Inhalt eines neuen Tupels darstellt.

Die `Update`-Nachricht kann entweder einen `K`-Nachrichtenteil oder einen `O`-Nachrichtenteil oder keinen von beiden enthalten, aber niemals beide.

`Delete`

`Byte1('D')`: Identifiziert die Nachricht als Delete-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion (nur für gestreamte Transaktionen vorhanden). Dieses Feld ist seit Protokollversion 2 verfügbar.

`Int32 (Oid)`: OID der Relation, die der ID in der Relation-Nachricht entspricht.

`Byte1('K')`: Identifiziert die folgende `TupleData`-Unternachricht als Schlüssel. Dieses Feld ist vorhanden, wenn die Tabelle, in der das Delete stattfand, einen Index als `REPLICA IDENTITY` verwendet.

`Byte1('O')`: Identifiziert die folgende `TupleData`-Nachricht als altes Tupel. Dieses Feld ist vorhanden, wenn die Tabelle, in der das Delete stattfand, `REPLICA IDENTITY` auf `FULL` gesetzt hat.

`TupleData`: `TupleData`-Nachrichtenteil, der je nach vorherigem Feld den Inhalt des alten Tupels oder Primärschlüssels darstellt.

Die `Delete`-Nachricht kann entweder einen `K`-Nachrichtenteil oder einen `O`-Nachrichtenteil enthalten, aber niemals beide.

`Truncate`

`Byte1('T')`: Identifiziert die Nachricht als Truncate-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion (nur für gestreamte Transaktionen vorhanden). Dieses Feld ist seit Protokollversion 2 verfügbar.

`Int32`: Anzahl der Relationen.

`Int8`: Optionsbits für `TRUNCATE`: 1 für `CASCADE`, 2 für `RESTART IDENTITY`.

`Int32 (Oid)`: OID der Relation, die der ID in der Relation-Nachricht entspricht. Dieses Feld wird für jede Relation wiederholt.

Die folgenden Nachrichten (`Stream Start`, `Stream Stop`, `Stream Commit` und `Stream Abort`) sind seit Protokollversion 2 verfügbar.

`Stream Start`

`Byte1('S')`: Identifiziert die Nachricht als Stream-Start-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion.

`Int8`: Ein Wert von 1 gibt an, dass dies das erste Stream-Segment für diese XID ist, 0 jedes andere Stream-Segment.

`Stream Stop`

`Byte1('E')`: Identifiziert die Nachricht als Stream-Stop-Nachricht.

`Stream Commit`

`Byte1('c')`: Identifiziert die Nachricht als Stream-Commit-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion.

`Int8(0)`: Flags; derzeit unbenutzt.

`Int64 (XLogRecPtr)`: Die LSN des Commits.

`Int64 (XLogRecPtr)`: Die End-LSN der Transaktion.

`Int64 (TimestampTz)`: Commit-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Stream Abort`

`Byte1('A')`: Identifiziert die Nachricht als Stream-Abort-Nachricht.

`Int32 (TransactionId)`: XID der Transaktion.

`Int32 (TransactionId)`: XID der Subtransaktion (für Top-Level-Transaktionen identisch mit der XID der Transaktion).

`Int64 (XLogRecPtr)`: Die LSN der Abort-Operation, nur vorhanden, wenn `streaming` auf `parallel` gesetzt ist. Dieses Feld ist seit Protokollversion 4 verfügbar.

`Int64 (TimestampTz)`: Abort-Zeitstempel der Transaktion, nur vorhanden, wenn `streaming` auf `parallel` gesetzt ist. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01). Dieses Feld ist seit Protokollversion 4 verfügbar.

Die folgenden Nachrichten (`Begin Prepare`, `Prepare`, `Commit Prepared`, `Rollback Prepared`, `Stream Prepare`) sind seit Protokollversion 3 verfügbar.

`Begin Prepare`

`Byte1('b')`: Identifiziert die Nachricht als Beginn einer Prepared-Transaction-Nachricht.

`Int64 (XLogRecPtr)`: Die LSN des Prepare.

`Int64 (XLogRecPtr)`: Die End-LSN der vorbereiteten Transaktion.

`Int64 (TimestampTz)`: Prepare-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Int32 (TransactionId)`: XID der Transaktion.

`String`: Die benutzerdefinierte GID der vorbereiteten Transaktion.

`Prepare`

`Byte1('P')`: Identifiziert die Nachricht als Prepared-Transaction-Nachricht.

`Int8(0)`: Flags; derzeit unbenutzt.

`Int64 (XLogRecPtr)`: Die LSN des Prepare.

`Int64 (XLogRecPtr)`: Die End-LSN der vorbereiteten Transaktion.

`Int64 (TimestampTz)`: Prepare-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Int32 (TransactionId)`: XID der Transaktion.

`String`: Die benutzerdefinierte GID der vorbereiteten Transaktion.

`Commit Prepared`

`Byte1('K')`: Identifiziert die Nachricht als Commit einer Prepared-Transaction-Nachricht.

`Int8(0)`: Flags; derzeit unbenutzt.

`Int64 (XLogRecPtr)`: Die LSN des Commits der vorbereiteten Transaktion.

`Int64 (XLogRecPtr)`: Die End-LSN des Commits der vorbereiteten Transaktion.

`Int64 (TimestampTz)`: Commit-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Int32 (TransactionId)`: XID der Transaktion.

`String`: Die benutzerdefinierte GID der vorbereiteten Transaktion.

`Rollback Prepared`

`Byte1('r')`: Identifiziert die Nachricht als Rollback einer Prepared-Transaction-Nachricht.

`Int8(0)`: Flags; derzeit unbenutzt.

`Int64 (XLogRecPtr)`: Die End-LSN der vorbereiteten Transaktion.

`Int64 (XLogRecPtr)`: Die End-LSN des Rollbacks der vorbereiteten Transaktion.

`Int64 (TimestampTz)`: Prepare-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Int64 (TimestampTz)`: Rollback-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Int32 (TransactionId)`: XID der Transaktion.

`String`: Die benutzerdefinierte GID der vorbereiteten Transaktion.

`Stream Prepare`

`Byte1('p')`: Identifiziert die Nachricht als Stream-Prepared-Transaction-Nachricht.

`Int8(0)`: Flags; derzeit unbenutzt.

`Int64 (XLogRecPtr)`: Die LSN des Prepare.

`Int64 (XLogRecPtr)`: Die End-LSN der vorbereiteten Transaktion.

`Int64 (TimestampTz)`: Prepare-Zeitstempel der Transaktion. Der Wert ist die Anzahl der Mikrosekunden seit der PostgreSQL-Epoche (2000-01-01).

`Int32 (TransactionId)`: XID der Transaktion.

`String`: Die benutzerdefinierte GID der vorbereiteten Transaktion.

Die folgenden Nachrichtenteile werden von den obigen Nachrichten gemeinsam verwendet.

`TupleData`

`Int16`: Anzahl der Spalten.

Als Nächstes erscheint für jede veröffentlichte Spalte eine der folgenden Unternachrichten:

`Byte1('n')`: Identifiziert die Daten als NULL-Wert.

Oder:

`Byte1('u')`: Identifiziert einen unveränderten TOAST-Wert (der tatsächliche Wert wird nicht gesendet).

Oder:

`Byte1('t')`: Identifiziert die Daten als textformatierten Wert.

Oder:

`Byte1('b')`: Identifiziert die Daten als binär formatierten Wert.

`Int32`: Länge des Spaltenwerts.

`Byten`: Der Wert der Spalte, entweder im Binär- oder im Textformat (wie im vorhergehenden Format-Byte angegeben). `n` ist die oben angegebene Länge.

## 54.10. Zusammenfassung der Änderungen seit Protokoll 2.0

Dieser Abschnitt bietet eine kurze Checkliste der Änderungen für Entwickler, die bestehende Clientbibliotheken auf Protokoll 3.0 aktualisieren wollen.

Das anfängliche Startup-Paket verwendet ein flexibles Listen-von-Strings-Format statt eines festen Formats. Beachten Sie, dass Sitzungs-Standardwerte für Laufzeitparameter nun direkt im Startup-Paket angegeben werden können. Eigentlich war das zuvor über das Feld `options` möglich; wegen dessen begrenzter Breite und fehlender Möglichkeit, Leerraum in den Werten zu quoten, war dies jedoch keine besonders sichere Technik.

Alle Nachrichten haben nun unmittelbar nach dem Nachrichtentyp-Byte eine Längenangabe (außer Startup-Pakete, die kein Typ-Byte haben). Beachten Sie außerdem, dass `PasswordMessage` nun ein Typ-Byte hat.

`ErrorResponse`- und `NoticeResponse`-Nachrichten (`E` und `N`) enthalten nun mehrere Felder, aus denen der Clientcode eine Fehlermeldung mit dem gewünschten Ausführlichkeitsgrad zusammensetzen kann. Beachten Sie, dass einzelne Felder typischerweise nicht mit einem Newline enden, während der einzelne String, der im älteren Protokoll gesendet wurde, dies immer tat.

Die Nachricht `ReadyForQuery` (`Z`) enthält eine Transaktionsstatusanzeige.

Die Unterscheidung zwischen den Nachrichtentypen `BinaryRow` und `DataRow` ist entfallen; die einzelne `DataRow`-Nachricht dient dazu, Daten in allen Formaten zurückzugeben. Beachten Sie, dass sich der Aufbau von `DataRow` geändert hat, um das Parsen zu erleichtern. Außerdem hat sich die Darstellung binärer Werte geändert: Sie ist nicht mehr direkt an die interne Darstellung des Servers gebunden.

Es gibt ein neues Unterprotokoll »Extended Query«, das die Frontend-Nachrichtentypen `Parse`, `Bind`, `Execute`, `Describe`, `Close`, `Flush` und `Sync` sowie die Backend-Nachrichtentypen `ParseComplete`, `BindComplete`, `PortalSuspended`, `ParameterDescription`, `NoData` und `CloseComplete` hinzufügt. Bestehende Clients müssen sich nicht mit diesem Unterprotokoll befassen, aber seine Verwendung kann Leistungs- oder Funktionsverbesserungen ermöglichen.

COPY-Daten werden nun in `CopyData`- und `CopyDone`-Nachrichten gekapselt. Es gibt einen wohldefinierten Weg, sich von Fehlern während `COPY` zu erholen. Die spezielle letzte Zeile `\.` wird nicht mehr benötigt und bei `COPY OUT` nicht gesendet. Sie wird bei `COPY IN` im Textmodus weiterhin als Terminator erkannt, aber nicht im CSV-Modus. Das Verhalten im Textmodus ist veraltet und kann irgendwann entfernt werden. Binäres `COPY` wird unterstützt. Die Nachrichten `CopyInResponse` und `CopyOutResponse` enthalten Felder, die die Anzahl der Spalten und das Format jeder Spalte angeben.

Der Aufbau der Nachrichten `FunctionCall` und `FunctionCallResponse` hat sich geändert. `FunctionCall` kann nun NULL-Argumente an Funktionen übergeben. Außerdem kann es Parameter übergeben und Ergebnisse entweder im Text- oder im Binärformat abrufen. Es gibt keinen Grund mehr, `FunctionCall` als potenzielles Sicherheitsloch zu betrachten, da es keinen direkten Zugriff auf interne Serverdatenrepräsentationen bietet.

Das Backend sendet während des Verbindungsstarts `ParameterStatus`-Nachrichten (`S`) für alle Parameter, die es für die Clientbibliothek interessant findet. Danach wird eine `ParameterStatus`-Nachricht gesendet, wann immer sich der aktive Wert eines dieser Parameter ändert.

Die Nachricht `RowDescription` (`T`) enthält neue Felder für Tabellen-OID und Spaltennummer für jede Spalte der beschriebenen Zeile. Außerdem zeigt sie den Format-Code für jede Spalte.

Die Nachricht `CursorResponse` (`P`) wird vom Backend nicht mehr erzeugt.

Die Nachricht `NotificationResponse` (`A`) hat ein zusätzliches String-Feld, das einen vom Sender des `NOTIFY`-Ereignisses übergebenen Payload-String enthalten kann.

Die Nachricht `EmptyQueryResponse` (`I`) enthielt früher einen leeren String-Parameter; dieser wurde entfernt.
