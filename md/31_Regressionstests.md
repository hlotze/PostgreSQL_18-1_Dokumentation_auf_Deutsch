# 31. Regressionstests

Die Regressionstests sind eine umfassende Testsammlung für die SQL-Implementierung in PostgreSQL. Sie testen Standard-SQL-Operationen ebenso wie die erweiterten Fähigkeiten von PostgreSQL.

## 31.1. Tests ausführen

Die Regressionstests können gegen einen bereits installierten und laufenden Server oder mit einer temporären Installation innerhalb des Build-Baums ausgeführt werden. Außerdem gibt es einen parallelen und einen sequenziellen Modus zur Ausführung der Tests. Die sequenzielle Methode führt jedes Testskript allein aus, während die parallele Methode mehrere Serverprozesse startet, um Gruppen von Tests parallel auszuführen. Paralleles Testen erhöht das Vertrauen, dass Interprozesskommunikation und Sperren korrekt funktionieren. Einige Tests können auch im parallelen Modus sequenziell laufen, falls dies für den Test erforderlich ist.

### 31.1.1. Tests gegen eine temporäre Installation ausführen

Um die parallelen Regressionstests nach dem Bauen, aber vor der Installation auszuführen, geben Sie im obersten Verzeichnis ein:

```text
make check
```

Sie können auch nach `src/test/regress` wechseln und den Befehl dort ausführen. Tests, die parallel ausgeführt werden, sind mit `+` markiert, Tests, die sequenziell laufen, mit `-`. Am Ende sollten Sie etwa Folgendes sehen:

```text
# All 213 tests passed.
```

Andernfalls sehen Sie einen Hinweis darauf, welche Tests fehlgeschlagen sind. Lesen Sie [Abschnitt 31.2](31_Regressionstests.md#312-testauswertung), bevor Sie annehmen, dass ein Fehlschlag ein ernsthaftes Problem darstellt.

Da diese Testmethode einen temporären Server ausführt, funktioniert sie nicht, wenn Sie den Build als Benutzer `root` erstellt haben, weil der Server nicht als `root` startet. Empfohlen wird, den Build nicht als `root` zu erstellen oder die Tests erst nach Abschluss der Installation auszuführen.

Wenn Sie PostgreSQL so konfiguriert haben, dass es an einem Ort installiert wird, an dem bereits eine ältere PostgreSQL-Installation existiert, und Sie vor der Installation der neuen Version `make check` ausführen, können die Tests fehlschlagen, weil die neuen Programme versuchen, die bereits installierten Shared Libraries zu verwenden. Typische Symptome sind Beschwerden über undefinierte Symbole. Wenn Sie die Tests ausführen möchten, bevor Sie die alte Installation überschreiben, müssen Sie mit `configure --disable-rpath` bauen. Für die endgültige Installation wird diese Option jedoch nicht empfohlen.

Der parallele Regressionstest startet ziemlich viele Prozesse unter Ihrer Benutzerkennung. Derzeit beträgt die maximale Parallelität zwanzig parallele Testskripte, also vierzig Prozesse: Für jedes Testskript gibt es einen Serverprozess und einen `psql`-Prozess. Wenn Ihr System also eine benutzerbezogene Grenze für die Anzahl der Prozesse erzwingt, stellen Sie sicher, dass diese Grenze mindestens ungefähr fünfzig beträgt; andernfalls können im parallelen Test scheinbar zufällige Fehler auftreten. Wenn Sie diese Grenze nicht erhöhen können, können Sie den Parallelitätsgrad durch Setzen des Parameters `MAX_CONNECTIONS` verringern. Zum Beispiel:

```text
make MAX_CONNECTIONS=10 check
```

Dies führt höchstens zehn Tests gleichzeitig aus.

### 31.1.2. Tests gegen eine vorhandene Installation ausführen

Um die Tests nach der Installation auszuführen (siehe [Kapitel 17](17_Installation_aus_dem_Quellcode.md)), initialisieren Sie ein Datenverzeichnis und starten den Server wie in [Kapitel 18](18_Servereinrichtung_und_Betrieb.md) beschrieben. Geben Sie dann ein:

```text
make installcheck
```

Oder für einen parallelen Test:

```text
make installcheck-parallel
```

Die Tests erwarten, den Server auf dem lokalen Host und dem Standard-Port zu erreichen, sofern nicht durch die Umgebungsvariablen `PGHOST` und `PGPORT` etwas anderes angegeben wurde. Die Tests werden in einer Datenbank namens `regression` ausgeführt; eine vorhandene Datenbank dieses Namens wird gelöscht.

Die Tests erzeugen außerdem vorübergehend einige clusterweite Objekte, etwa Rollen, Tablespaces und Subscriptions. Diese Objekte haben Namen, die mit `regress_` beginnen. Seien Sie vorsichtig, wenn Sie den Modus `installcheck` mit einer Installation verwenden, die tatsächliche globale Objekte mit solchen Namen enthält.

### 31.1.3. Zusätzliche Testsuiten

Die Befehle `make check` und `make installcheck` führen nur die Kern-Regressionstests aus, die eingebaute Funktionen des PostgreSQL-Servers testen. Die Quelldistribution enthält viele zusätzliche Testsuiten, von denen die meisten mit Zusatzfunktionen wie optionalen prozeduralen Sprachen zu tun haben.

Um alle Testsuiten auszuführen, die auf die zum Bauen ausgewählten Module anwendbar sind, einschließlich der Kerntests, geben Sie im obersten Verzeichnis des Build-Baums einen dieser Befehle ein:

```text
make check-world
make installcheck-world
```

Diese Befehle führen die Tests mit temporären Servern beziehungsweise mit einem bereits installierten Server aus, so wie zuvor für `make check` und `make installcheck` beschrieben. Die übrigen Überlegungen sind dieselben wie für die jeweilige Methode. Beachten Sie, dass `make check-world` für jedes getestete Modul eine eigene Instanz, also ein temporäres Datenverzeichnis, erstellt und daher mehr Zeit und Plattenplatz benötigt als `make installcheck-world`.

Auf einem modernen Rechner mit mehreren CPU-Kernen und ohne enge Betriebssystemgrenzen können Sie die Ausführung durch Parallelität deutlich beschleunigen. Das Rezept, das viele PostgreSQL-Entwickler tatsächlich verwenden, um alle Tests auszuführen, sieht etwa so aus:

```text
make check-world -j8 >/dev/null
```

Dabei liegt die `-j`-Grenze nahe bei oder etwas über der Anzahl verfügbarer Kerne. Das Verwerfen von `stdout` beseitigt Meldungen, die uninteressant sind, wenn Sie nur den Erfolg überprüfen möchten. Im Fehlerfall reichen die `stderr`-Meldungen normalerweise aus, um zu erkennen, wo genauer nachgesehen werden muss.

Alternativ können Sie einzelne Testsuiten ausführen, indem Sie `make check` oder `make installcheck` im passenden Unterverzeichnis des Build-Baums eingeben. Beachten Sie, dass `make installcheck` voraussetzt, dass die relevanten Module installiert wurden, nicht nur der Kernserver.

Zu den zusätzlichen Tests, die auf diese Weise aufgerufen werden können, gehören:

- Regressionstests für optionale prozedurale Sprachen. Diese befinden sich unter `src/pl`.
- Regressionstests für `contrib`-Module, die sich unter `contrib` befinden. Nicht alle `contrib`-Module haben Tests.
- Regressionstests für die Schnittstellenbibliotheken, die sich in `src/interfaces/libpq/test` und `src/interfaces/ecpg/test` befinden.
- Tests für vom Kern unterstützte Authentifizierungsmethoden, die sich in `src/test/authentication` befinden. Weitere authentifizierungsbezogene Tests sind unten beschrieben.
- Tests, die das Verhalten gleichzeitiger Sitzungen belasten, unter `src/test/isolation`.
- Tests für Crash-Recovery und physische Replikation unter `src/test/recovery`.
- Tests für logische Replikation unter `src/test/subscription`.
- Tests von Clientprogrammen unter `src/bin`.

Bei Verwendung des Modus `installcheck` erzeugen und löschen diese Tests Testdatenbanken, deren Namen `regression` enthalten, zum Beispiel `pl_regression` oder `contrib_regression`. Seien Sie vorsichtig, wenn Sie `installcheck` mit einer Installation verwenden, die Nicht-Test-Datenbanken mit solchen Namen enthält.

Einige dieser Hilfstestsuiten verwenden die TAP-Infrastruktur, die in [Abschnitt 31.4](31_Regressionstests.md#314-taptests) beschrieben ist. TAP-basierte Tests werden nur ausgeführt, wenn PostgreSQL mit der Option `--enable-tap-tests` konfiguriert wurde. Dies wird für die Entwicklung empfohlen, kann aber weggelassen werden, wenn keine geeignete Perl-Installation vorhanden ist.

Einige Testsuiten werden standardmäßig nicht ausgeführt, entweder weil ihre Ausführung auf einem Mehrbenutzersystem nicht sicher ist, weil sie spezielle Software erfordern oder weil sie ressourcenintensiv sind. Sie können entscheiden, welche Testsuiten zusätzlich ausgeführt werden, indem Sie die `make`- oder Umgebungsvariable `PG_TEST_EXTRA` auf eine durch Leerzeichen getrennte Liste setzen, zum Beispiel:

```text
make check-world PG_TEST_EXTRA='kerberos ldap ssl load_balance libpq_encryption'
```

Die folgenden Werte werden derzeit unterstützt:

- `kerberos`: Führt die Testsuite unter `src/test/kerberos` aus. Dafür ist eine MIT-Kerberos-Installation erforderlich, und es werden TCP/IP-Listen-Sockets geöffnet.
- `ldap`: Führt die Testsuite unter `src/test/ldap` aus. Dafür ist eine OpenLDAP-Installation erforderlich, und es werden TCP/IP-Listen-Sockets geöffnet.
- `libpq_encryption`: Führt den Test `src/interfaces/libpq/t/005_negotiate_encryption.pl` aus. Dieser öffnet TCP/IP-Listen-Sockets. Wenn `PG_TEST_EXTRA` außerdem `kerberos` enthält, werden zusätzliche Tests aktiviert, die eine MIT-Kerberos-Installation erfordern.
- `load_balance`: Führt den Test `src/interfaces/libpq/t/004_load_balance_dns.pl` aus. Dafür muss die System-Hosts-Datei bearbeitet werden, und es werden TCP/IP-Listen-Sockets geöffnet.
- `oauth`: Führt die Testsuite unter `src/test/modules/oauth_validator` aus. Dies öffnet TCP/IP-Listen-Sockets für einen Testserver, der HTTPS ausführt.
- `regress_dump_restore`: Führt eine zusätzliche Testsuite in `src/bin/pg_upgrade/t/002_pg_upgrade.pl` aus, die die Datenbank `regression` durch `pg_dump`/`pg_restore` schleust. Sie ist standardmäßig nicht aktiviert, weil sie ressourcenintensiv ist.
- `sepgsql`: Führt die Testsuite unter `contrib/sepgsql` aus. Dafür ist eine SELinux-Umgebung erforderlich, die auf eine bestimmte Weise eingerichtet ist; siehe Abschnitt F.40.3.
- `ssl`: Führt die Testsuite unter `src/test/ssl` aus. Dies öffnet TCP/IP-Listen-Sockets.
- `wal_consistency_checking`: Verwendet `wal_consistency_checking=all`, während bestimmte Tests unter `src/test/recovery` laufen. Dies ist standardmäßig nicht aktiviert, weil es ressourcenintensiv ist.
- `xid_wraparound`: Führt die Testsuite unter `src/test/modules/xid_wraparound` aus. Sie ist standardmäßig nicht aktiviert, weil sie ressourcenintensiv ist.

Tests für Funktionen, die von der aktuellen Build-Konfiguration nicht unterstützt werden, werden nicht ausgeführt, selbst wenn sie in `PG_TEST_EXTRA` genannt werden.

Außerdem gibt es Tests in `src/test/modules`, die von `make check-world`, aber nicht von `make installcheck-world` ausgeführt werden. Das liegt daran, dass sie Nicht-Produktions-Erweiterungen installieren oder andere Nebeneffekte haben, die für eine Produktionsinstallation als unerwünscht gelten. Sie können in einem dieser Unterverzeichnisse `make install` und `make installcheck` verwenden, wenn Sie möchten; mit einem Nicht-Test-Server wird dies jedoch nicht empfohlen.

### 31.1.4. Locale und Encoding

Standardmäßig verwenden Tests mit einer temporären Installation die in der aktuellen Umgebung definierte Locale und das entsprechende Datenbank-Encoding, wie es von `initdb` bestimmt wird. Es kann nützlich sein, verschiedene Locales zu testen, indem die entsprechenden Umgebungsvariablen gesetzt werden, zum Beispiel:

```text
make check LANG=C
make check LC_COLLATE=en_US.utf8 LC_CTYPE=fr_CA.utf8
```

Aus Implementierungsgründen funktioniert das Setzen von `LC_ALL` für diesen Zweck nicht; alle anderen locale-bezogenen Umgebungsvariablen funktionieren.

Beim Testen gegen eine vorhandene Installation wird die Locale durch den vorhandenen Datenbankcluster bestimmt und kann für den Testlauf nicht separat gesetzt werden.

Sie können das Datenbank-Encoding auch ausdrücklich durch Setzen der Variablen `ENCODING` wählen, zum Beispiel:

```text
make check LANG=C ENCODING=EUC_JP
```

Das Setzen des Datenbank-Encodings auf diese Weise ist typischerweise nur sinnvoll, wenn die Locale `C` ist; andernfalls wird das Encoding automatisch aus der Locale gewählt, und die Angabe eines Encodings, das nicht zur Locale passt, führt zu einem Fehler.

Das Datenbank-Encoding kann für Tests gegen eine temporäre oder eine vorhandene Installation gesetzt werden, wobei es im letzteren Fall mit der Locale der Installation kompatibel sein muss.

### 31.1.5. Angepasste Servereinstellungen

Es gibt mehrere Möglichkeiten, bei der Ausführung einer Testsuite angepasste Servereinstellungen zu verwenden. Das kann nützlich sein, um zusätzliche Protokollierung zu aktivieren, Ressourcengrenzen anzupassen oder zusätzliche Laufzeitprüfungen wie `debug_discard_caches` zu aktivieren. Beachten Sie aber, dass nicht alle Tests mit beliebigen Einstellungen sauber bestehen müssen.

Zusätzliche Optionen können mit der Umgebungsvariablen `PG_TEST_INITDB_EXTRA_OPTS` an die verschiedenen `initdb`-Befehle übergeben werden, die während der Testeinrichtung intern ausgeführt werden. Um zum Beispiel einen Test mit aktivierten Prüfsummen, angepasster WAL-Segmentgröße und angepasster `work_mem`-Einstellung auszuführen, verwenden Sie:

```text
make check PG_TEST_INITDB_EXTRA_OPTS='-k --wal-segsize=4 -c work_mem=50MB'
```

Für die Kern-Regressionstestsuite und andere von `pg_regress` gesteuerte Tests können angepasste Laufzeit-Servereinstellungen auch in der Umgebungsvariablen `PGOPTIONS` gesetzt werden, sofern die Einstellungen dies erlauben, zum Beispiel:

```text
make check PGOPTIONS="-c debug_parallel_query=regress -c work_mem=50MB"
```

Dies nutzt Funktionalität von `libpq`; Details finden Sie bei `options`.

Beim Ausführen gegen eine temporäre Installation können angepasste Einstellungen auch durch Bereitstellen einer vorbereiteten `postgresql.conf` gesetzt werden:

```text
echo 'log_checkpoints = on' > test_postgresql.conf
echo 'work_mem = 50MB' >> test_postgresql.conf
make check EXTRA_REGRESS_OPTS="--temp-config=test_postgresql.conf"
```

### 31.1.6. Zusätzliche Tests

Die Kern-Regressionstestsuite enthält einige Testdateien, die standardmäßig nicht ausgeführt werden, weil sie plattformabhängig sein können oder sehr lange laufen. Sie können diese oder andere zusätzliche Testdateien ausführen, indem Sie die Variable `EXTRA_TESTS` setzen. Um zum Beispiel den Test `numeric_big` auszuführen:

```text
make check EXTRA_TESTS=numeric_big
```

## 31.2. Testauswertung

Einige korrekt installierte und voll funktionsfähige PostgreSQL-Installationen können einige dieser Regressionstests aufgrund plattformspezifischer Artefakte, etwa unterschiedlicher Gleitkommadarstellung oder abweichender Formulierungen von Meldungen, scheinbar nicht bestehen. Die Tests werden derzeit mit einem einfachen `diff`-Vergleich gegen Ausgaben bewertet, die auf einem Referenzsystem erzeugt wurden; daher reagieren die Ergebnisse empfindlich auf kleine Systemunterschiede. Wenn ein Test als fehlgeschlagen gemeldet wird, untersuchen Sie immer die Unterschiede zwischen erwarteten und tatsächlichen Ergebnissen; möglicherweise sind die Unterschiede nicht bedeutsam. Dennoch bemühen wir uns, genaue Referenzdateien für alle unterstützten Plattformen zu pflegen, sodass erwartet werden kann, dass alle Tests bestehen.

Die tatsächlichen Ausgaben der Regressionstests befinden sich in Dateien im Verzeichnis `src/test/regress/results`. Das Testskript verwendet `diff`, um jede Ausgabedatei mit den Referenzausgaben im Verzeichnis `src/test/regress/expected` zu vergleichen. Unterschiede werden zur Untersuchung in `src/test/regress/regression.diffs` gespeichert. Wenn Sie eine andere Testsuite als die Kerntests ausführen, erscheinen diese Dateien natürlich im jeweiligen Unterverzeichnis, nicht in `src/test/regress`.

Wenn Ihnen die standardmäßig verwendeten `diff`-Optionen nicht gefallen, setzen Sie die Umgebungsvariable `PG_REGRESS_DIFF_OPTS`, zum Beispiel `PG_REGRESS_DIFF_OPTS='-c'`. Sie können `diff` auch selbst ausführen, wenn Ihnen das lieber ist.

Wenn eine bestimmte Plattform aus irgendeinem Grund für einen Test einen Fehlschlag erzeugt, Sie nach Prüfung der Ausgabe aber überzeugt sind, dass das Ergebnis gültig ist, können Sie eine neue Vergleichsdatei hinzufügen, um diese Fehlermeldung in künftigen Testläufen zu unterdrücken. Details finden Sie in [Abschnitt 31.3](31_Regressionstests.md#313-variantenvergleichsdateien).

### 31.2.1. Unterschiede in Fehlermeldungen

Einige Regressionstests enthalten absichtlich ungültige Eingabewerte. Fehlermeldungen können entweder aus dem PostgreSQL-Code oder aus Systemroutinen der Hostplattform stammen. Im letzteren Fall können sich die Meldungen zwischen Plattformen unterscheiden, sollten aber ähnliche Informationen enthalten. Solche Meldungsunterschiede führen zu einem fehlgeschlagenen Regressionstest, der durch Prüfung bestätigt werden kann.

### 31.2.2. Locale-Unterschiede

Wenn Sie die Tests gegen einen Server ausführen, der mit einer Sortier-Locale ungleich `C` initialisiert wurde, kann es Unterschiede aufgrund der Sortierreihenfolge und daraus folgende Fehler geben. Die Regressionstestsuite ist darauf vorbereitet, indem sie alternative Ergebnisdateien bereitstellt, die zusammen eine große Zahl von Locales abdecken.

Um die Tests bei Verwendung der temporären Installationsmethode in einer anderen Locale auszuführen, übergeben Sie die passenden locale-bezogenen Umgebungsvariablen auf der `make`-Befehlszeile, zum Beispiel:

```text
make check LANG=de_DE.utf8
```

Der Regressionstesttreiber entfernt `LC_ALL`, daher funktioniert es nicht, die Locale mit dieser Variablen zu wählen. Um keine Locale zu verwenden, entfernen Sie entweder alle locale-bezogenen Umgebungsvariablen oder setzen sie auf `C`, oder verwenden Sie den folgenden speziellen Aufruf:

```text
make check NO_LOCALE=1
```

Beim Ausführen der Tests gegen eine vorhandene Installation wird die Locale-Einrichtung durch die vorhandene Installation bestimmt. Um sie zu ändern, initialisieren Sie den Datenbankcluster mit einer anderen Locale, indem Sie die entsprechenden Optionen an `initdb` übergeben.

Im Allgemeinen ist es ratsam zu versuchen, die Regressionstests mit der Locale-Einrichtung auszuführen, die für den Produktionseinsatz gewünscht ist, da dadurch die locale- und encoding-bezogenen Codebereiche getestet werden, die tatsächlich in Produktion verwendet werden. Je nach Betriebssystemumgebung können Fehlschläge auftreten; dann wissen Sie aber zumindest, welches locale-spezifische Verhalten bei echten Anwendungen zu erwarten ist.

### 31.2.3. Datums- und Uhrzeitunterschiede

Die meisten Datums- und Uhrzeitergebnisse hängen von der Zeitzonenumgebung ab. Die Referenzdateien werden für die Zeitzone `America/Los_Angeles` erzeugt, und es kommt zu scheinbaren Fehlschlägen, wenn die Tests nicht mit dieser Zeitzoneneinstellung ausgeführt werden. Der Regressionstesttreiber setzt die Umgebungsvariable `PGTZ` auf `America/Los_Angeles`, was normalerweise korrekte Ergebnisse sicherstellt.

### 31.2.4. Gleitkommaunterschiede

Einige Tests berechnen 64-Bit-Gleitkommazahlen (`double precision`) aus Tabellenspalten. Unterschiede bei Ergebnissen, die mathematische Funktionen von `double precision`-Spalten betreffen, wurden beobachtet. Die Tests `float8` und `geometry` sind besonders anfällig für kleine Unterschiede zwischen Plattformen oder sogar bei unterschiedlichen Compiler-Optimierungseinstellungen. Eine menschliche Sichtprüfung ist nötig, um die tatsächliche Bedeutung dieser Unterschiede zu beurteilen, die normalerweise zehn Stellen rechts vom Dezimalpunkt liegen.

Einige Systeme zeigen minus null als `-0`, andere nur als `0`.

Einige Systeme melden Fehler aus `pow()` und `exp()` anders als es der aktuelle PostgreSQL-Code erwartet.

### 31.2.5. Unterschiede in der Zeilenreihenfolge

Sie können Unterschiede sehen, bei denen dieselben Zeilen in einer anderen Reihenfolge ausgegeben werden als in der erwarteten Datei. In den meisten Fällen ist dies streng genommen kein Fehler. Die meisten Regressionstestskripte sind nicht so pedantisch, für jedes einzelne `SELECT` ein `ORDER BY` zu verwenden; daher sind ihre Ergebniszeilenreihenfolgen nach SQL-Spezifikation nicht wohldefiniert. In der Praxis erhalten wir normalerweise auf allen Plattformen dieselbe Ergebnisreihenfolge, weil dieselben Abfragen auf denselben Daten mit derselben Software ausgeführt werden; deshalb ist das Fehlen von `ORDER BY` meist kein Problem. Einige Abfragen zeigen jedoch plattformübergreifende Reihenfolgeunterschiede. Beim Testen gegen einen bereits installierten Server können Reihenfolgeunterschiede außerdem durch Nicht-`C`-Locale-Einstellungen oder nicht standardmäßige Parametereinstellungen verursacht werden, etwa angepasste Werte von `work_mem` oder Planner-Kostenparametern.

Wenn Sie also einen Reihenfolgeunterschied sehen, ist das kein Grund zur Sorge, es sei denn, die Abfrage hat ein `ORDER BY`, gegen das Ihr Ergebnis verstößt. Bitte melden Sie ihn dennoch, damit in dieser bestimmten Abfrage ein `ORDER BY` ergänzt werden kann und der falsche Fehlschlag in zukünftigen Versionen verschwindet.

Vielleicht fragen Sie sich, warum nicht alle Regressionstestabfragen ausdrücklich sortiert werden, um dieses Problem endgültig zu beseitigen. Der Grund ist, dass die Regressionstests dadurch weniger nützlich würden, nicht nützlicher, weil sie dann bevorzugt Query-Plantypen testen würden, die geordnete Ergebnisse erzeugen, und solche ausschließen würden, die das nicht tun.

### 31.2.6. Unzureichende Stack-Tiefe

Wenn der Test `errors` beim Befehl `select infinite_recurse()` zu einem Serverabsturz führt, bedeutet dies, dass die Grenze der Plattform für die Prozess-Stackgröße kleiner ist, als der Parameter `max_stack_depth` angibt. Dies kann behoben werden, indem der Server mit einer höheren Stackgrößengrenze ausgeführt wird; 4 MB werden beim Standardwert von `max_stack_depth` empfohlen. Wenn Sie das nicht tun können, besteht eine Alternative darin, den Wert von `max_stack_depth` zu verringern.

Auf Plattformen mit Unterstützung für `getrlimit()` sollte der Server automatisch einen sicheren Wert für `max_stack_depth` wählen. Wenn Sie diese Einstellung nicht manuell überschrieben haben, ist ein solcher Fehlschlag daher ein meldepflichtiger Fehler.

### 31.2.7. Der Test `random`

Das Testskript `random` ist dafür gedacht, zufällige Ergebnisse zu erzeugen. In sehr seltenen Fällen führt dies dazu, dass dieser Regressionstest fehlschlägt. Die Eingabe:

```text
diff results/random.out expected/random.out
```

sollte nur eine oder wenige Zeilen Unterschiede erzeugen. Sie müssen sich keine Sorgen machen, sofern der Test `random` nicht wiederholt fehlschlägt.

### 31.2.8. Konfigurationsparameter

Beim Ausführen der Tests gegen eine vorhandene Installation können einige nicht standardmäßige Parametereinstellungen dazu führen, dass die Tests fehlschlagen. Beispielsweise kann das Ändern von Parametern wie `enable_seqscan` oder `enable_indexscan` Planänderungen verursachen, die die Ergebnisse von Tests beeinflussen, die `EXPLAIN` verwenden.

## 31.3. Variantenvergleichsdateien

Da einige Tests von Natur aus umgebungsabhängige Ergebnisse erzeugen, gibt es Möglichkeiten, alternative erwartete Ergebnisdateien anzugeben. Jeder Regressionstest kann mehrere Vergleichsdateien haben, die mögliche Ergebnisse auf unterschiedlichen Plattformen zeigen. Es gibt zwei unabhängige Mechanismen, um zu bestimmen, welche Vergleichsdatei für einen Test verwendet wird.

Der erste Mechanismus erlaubt, Vergleichsdateien für bestimmte Plattformen auszuwählen. Es gibt eine Zuordnungsdatei `src/test/regress/resultmap`, die definiert, welche Vergleichsdatei für jede Plattform zu verwenden ist. Um falsche Testfehlschläge für eine bestimmte Plattform zu beseitigen, wählen oder erstellen Sie zunächst eine Varianten-Ergebnisdatei und fügen dann eine Zeile zur Datei `resultmap` hinzu.

Jede Zeile in der Zuordnungsdatei hat die Form:

```text
testname:output:platformpattern=comparisonfilename
```

Der Testname ist einfach der Name des jeweiligen Regressionstestmoduls. Der Ausgabewert gibt an, welche Ausgabedatei zu prüfen ist. Für die Standard-Regressionstests ist dies immer `out`. Der Wert entspricht der Dateiendung der Ausgabedatei. Das Plattformmuster ist ein Muster im Stil des Unix-Werkzeugs `expr`, also ein regulärer Ausdruck mit implizitem `^`-Anker am Anfang. Es wird mit dem Plattformnamen verglichen, den `config.guess` ausgibt. Der Vergleichsdateiname ist der Basisname der ersetzenden Ergebnisvergleichsdatei.

Zum Beispiel fehlt einigen Systemen eine funktionierende Funktion `strtof`; unser Workaround dafür verursacht Rundungsfehler im Regressionstest `float4`. Daher stellen wir eine Variantenvergleichsdatei `float4-misrounded-input.out` bereit, die die auf solchen Systemen zu erwartenden Ergebnisse enthält. Um die falsche Fehlschlagmeldung auf Cygwin-Plattformen zu unterdrücken, enthält `resultmap`:

```text
float4:out:.*-.*-cygwin.*=float4-misrounded-input.out
```

Dies wird auf jedem Rechner ausgelöst, bei dem die Ausgabe von `config.guess` zu `.*-.*-cygwin.*` passt. Andere Zeilen in `resultmap` wählen die Variantenvergleichsdatei für weitere Plattformen aus, wo dies passend ist.

Der zweite Auswahlmechanismus für Variantenvergleichsdateien ist deutlich automatischer: Er verwendet einfach die beste Übereinstimmung unter mehreren bereitgestellten Vergleichsdateien. Das Regressionstest-Treiberskript berücksichtigt sowohl die Standardvergleichsdatei eines Tests, `testname.out`, als auch Varianten mit Namen `testname_digit.out`, wobei `digit` eine einzelne Ziffer von 0 bis 9 ist. Wenn eine solche Datei exakt passt, gilt der Test als bestanden; andernfalls wird diejenige verwendet, die den kürzesten Diff erzeugt, um den Fehlerbericht zu erstellen. Wenn `resultmap` einen Eintrag für den jeweiligen Test enthält, ist der Basis-`testname` der dort angegebene Ersatzname.

Für den Test `char` enthält die Vergleichsdatei `char.out` beispielsweise Ergebnisse, die in den Locales `C` und `POSIX` erwartet werden, während die Datei `char_1.out` Ergebnisse enthält, die so sortiert sind, wie sie in vielen anderen Locales erscheinen.

Der Best-Match-Mechanismus wurde entwickelt, um locale-abhängige Ergebnisse zu behandeln, kann aber in jeder Situation verwendet werden, in der die Testergebnisse nicht leicht allein aus dem Plattformnamen vorhergesagt werden können. Eine Einschränkung dieses Mechanismus ist, dass der Testtreiber nicht erkennen kann, welche Variante für die aktuelle Umgebung tatsächlich korrekt ist; er wählt nur die Variante, die am besten zu funktionieren scheint. Daher ist es am sichersten, diesen Mechanismus nur für Variantenergebnisse zu verwenden, die Sie in allen Kontexten als gleichermaßen gültig betrachten.

## 31.4. TAP-Tests

Verschiedene Tests, insbesondere die Clientprogrammtests unter `src/bin`, verwenden die Perl-TAP-Werkzeuge und werden mit dem Perl-Testprogramm `prove` ausgeführt. Sie können Befehlszeilenoptionen an `prove` übergeben, indem Sie die `make`-Variable `PROVE_FLAGS` setzen, zum Beispiel:

```text
make -C src/bin check PROVE_FLAGS='--timer'
```

Weitere Informationen finden Sie in der Manualpage von `prove`.

Die `make`-Variable `PROVE_TESTS` kann verwendet werden, um eine durch Leerzeichen getrennte Liste von Pfaden relativ zum Makefile zu definieren, das `prove` aufruft, um statt der Standardmenge `t/*.pl` die angegebene Teilmenge von Tests auszuführen. Zum Beispiel:

```text
make check PROVE_TESTS='t/001_test1.pl t/003_test3.pl'
```

Die TAP-Tests benötigen das Perl-Modul `IPC::Run`. Dieses Modul ist von CPAN (<https://metacpan.org/dist/IPC-Run>) oder als Betriebssystempaket verfügbar. Außerdem muss PostgreSQL mit der Option `--enable-tap-tests` konfiguriert sein.

Allgemein gesagt testen die TAP-Tests die Programme in einem zuvor installierten Installationsbaum, wenn Sie `make installcheck` eingeben, oder bauen einen neuen lokalen Installationsbaum aus den aktuellen Quellen, wenn Sie `make check` eingeben. In beiden Fällen initialisieren sie eine lokale Instanz, also ein Datenverzeichnis, und führen vorübergehend einen Server darin aus. Einige dieser Tests führen mehr als einen Server aus. Daher können diese Tests recht ressourcenintensiv sein.

Wichtig ist, dass die TAP-Tests auch dann Testserver starten, wenn Sie `make installcheck` eingeben. Das unterscheidet sie von der traditionellen Nicht-TAP-Testinfrastruktur, die in diesem Fall erwartet, einen bereits laufenden Testserver zu verwenden. Einige PostgreSQL-Unterverzeichnisse enthalten sowohl Tests im traditionellen Stil als auch TAP-Tests; `make installcheck` erzeugt dort eine Mischung aus Ergebnissen von temporären Servern und dem bereits laufenden Testserver.

### 31.4.1. Umgebungsvariablen

Datenverzeichnisse werden nach der Testdatei benannt und bleiben erhalten, wenn ein Test fehlschlägt. Wenn die Umgebungsvariable `PG_TEST_NOCLEAN` gesetzt ist, bleiben Datenverzeichnisse unabhängig vom Testergebnis erhalten. Um zum Beispiel das Datenverzeichnis beim Ausführen der `pg_dump`-Tests unabhängig von den Testergebnissen zu behalten:

```text
PG_TEST_NOCLEAN=1 make -C src/bin/pg_dump check
```

Diese Umgebungsvariable verhindert außerdem, dass die temporären Verzeichnisse des Tests entfernt werden.

Viele Operationen in den Testsuiten verwenden ein Timeout von 180 Sekunden, was auf langsamen Hosts zu lastbedingten Timeouts führen kann. Wenn Sie die Umgebungsvariable `PG_TEST_TIMEOUT_DEFAULT` auf eine höhere Zahl setzen, wird der Standard entsprechend geändert, um dies zu vermeiden.

## 31.5. Testabdeckung untersuchen

Der PostgreSQL-Quellcode kann mit Instrumentierung für Coverage-Tests kompiliert werden, sodass untersucht werden kann, welche Teile des Codes durch die Regressionstests oder eine andere Testsuite abgedeckt werden, die mit dem Code ausgeführt wird. Dies wird derzeit beim Kompilieren mit GCC unterstützt und erfordert die Pakete `gcov` und `lcov`.

### 31.5.1. Coverage mit Autoconf und Make

Ein typischer Ablauf sieht so aus:

```text
./configure --enable-coverage ... WEITERE OPTIONEN ...
make
make check # oder eine andere Testsuite
make coverage-html
```

Öffnen Sie anschließend `coverage/index.html` in Ihrem HTML-Browser.

Wenn Sie `lcov` nicht haben oder Textausgabe einem HTML-Bericht vorziehen, können Sie stattdessen

```text
make coverage
```

ausführen. Dies erzeugt `.gcov`-Ausgabedateien für jede Quelldatei, die für den Test relevant ist. `make coverage` und `make coverage-html` überschreiben gegenseitig ihre Dateien, daher kann das Mischen verwirrend sein.

Sie können mehrere verschiedene Tests ausführen, bevor Sie den Coverage-Bericht erstellen; die Ausführungszähler werden aufsummiert. Wenn Sie die Ausführungszähler zwischen Testläufen zurücksetzen möchten, führen Sie aus:

```text
make coverage-clean
```

Sie können `make coverage-html` oder `make coverage` in einem Unterverzeichnis ausführen, wenn Sie einen Coverage-Bericht nur für einen Teil des Codebaums möchten.

Verwenden Sie `make distclean`, um anschließend aufzuräumen.

### 31.5.2. Coverage mit Meson

Ein typischer Ablauf sieht so aus:

```text
meson setup -Db_coverage=true ... WEITERE OPTIONEN ... builddir/
meson compile -C builddir/
meson test -C builddir/
cd builddir/
ninja coverage-html
```

Öffnen Sie anschließend `./meson-logs/coveragereport/index.html` in Ihrem HTML-Browser.

Sie können mehrere verschiedene Tests ausführen, bevor Sie den Coverage-Bericht erstellen; die Ausführungszähler werden aufsummiert.
