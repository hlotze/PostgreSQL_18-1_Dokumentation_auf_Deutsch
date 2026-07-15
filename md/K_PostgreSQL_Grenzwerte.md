# Anhang K. PostgreSQL-Grenzwerte

Tabelle K.1 beschreibt verschiedene feste Obergrenzen von PostgreSQL. Praktische Grenzen, etwa Performance-Einschränkungen oder verfügbarer Plattenplatz, können jedoch erreicht werden, bevor absolute harte Grenzen relevant werden.

**Tabelle K.1. PostgreSQL-Grenzwerte**

| Element | Obergrenze | Kommentar |
| --- | --- | --- |
| Datenbankgröße | unbegrenzt |  |
| Anzahl Datenbanken | 4.294.950.911 |  |
| Relationen pro Datenbank | 1.431.650.303 |  |
| Relationsgröße | 32 TB | Bei der Standard-`BLCKSZ` von 8192 Byte. |
| Zeilen pro Tabelle | begrenzt durch die Anzahl der Tupel, die auf 4.294.967.295 Seiten passen |  |
| Spalten pro Tabelle | 1.600 | Zusätzlich begrenzt dadurch, ob die Tupelgröße auf eine einzelne Seite passt; siehe Hinweis unten. |
| Spalten in einer Ergebnismenge | 1.664 |  |
| Feldgröße | 1 GB |  |
| Indizes pro Tabelle | unbegrenzt | Begrenzt durch die maximale Zahl von Relationen pro Datenbank. |
| Spalten pro Index | 32 | Kann durch Neukompilieren von PostgreSQL erhöht werden. |
| Partitionierungsschlüssel | 32 | Kann durch Neukompilieren von PostgreSQL erhöht werden. |
| Bezeichnerlänge | 63 Byte | Kann durch Neukompilieren von PostgreSQL erhöht werden. |
| Funktionsargumente | 100 | Kann durch Neukompilieren von PostgreSQL erhöht werden. |
| Abfrageparameter | 65.535 |  |

Die maximale Spaltenzahl einer Tabelle wird weiter dadurch reduziert, dass das zu speichernde Tupel auf eine einzelne 8192-Byte-Heap-Seite passen muss. Ohne Tupel-Header würde ein Tupel aus 1.600 `int`-Spalten beispielsweise 6400 Byte belegen und könnte auf einer Heap-Seite gespeichert werden. Ein Tupel aus 1.600 `bigint`-Spalten würde dagegen 12800 Byte belegen und daher nicht auf eine Heap-Seite passen. Felder variabler Länge wie `text`, `varchar` und `char` können ihre Werte in der TOAST-Tabelle der Tabelle außerhalb der Hauptzeile speichern, wenn die Werte groß genug sind. Im Tupel des Tabellen-Heaps muss dann nur ein 18-Byte-Zeiger verbleiben. Für kürzere Felder variabler Länge wird ein Feld-Header von 4 Byte oder 1 Byte verwendet, und der Wert wird direkt im Heap-Tupel gespeichert.

Auch Spalten, die aus der Tabelle entfernt wurden, zählen weiterhin zur maximalen Spaltenzahl. Zwar werden die Werte gelöschter Spalten in neu erzeugten Tupeln intern als `NULL` in der Null-Bitmap des Tupels markiert, doch auch diese Null-Bitmap benötigt Speicherplatz.

Jede Tabelle kann theoretisch höchstens `2^32` außerhalb der Hauptzeile gespeicherte Werte enthalten; eine ausführliche Beschreibung dieser Speicherform finden Sie in [Abschnitt 66.2](66_Physische_Speicherung_der_Datenbank.md#662-toast). Diese Grenze ergibt sich daraus, dass jeder solche Wert über eine 32-Bit-OID identifiziert wird. Die praktische Grenze liegt deutlich darunter, weil das Finden einer noch freien OID teurer werden kann, je voller der OID-Raum wird. Das verlangsamt wiederum `INSERT`- und `UPDATE`-Anweisungen. Typischerweise ist dies nur bei Tabellen mit vielen Terabyte Daten ein Problem; Partitionierung kann ein möglicher Ausweg sein.
