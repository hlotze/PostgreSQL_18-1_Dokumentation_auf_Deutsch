# Vorwort

Dieses Buch ist die offizielle Dokumentation von PostgreSQL. Es wurde von den PostgreSQL-Entwicklern und anderen Freiwilligen parallel zur Entwicklung der PostgreSQL-Software geschrieben. Es beschreibt die gesamte Funktionalität, die von der aktuellen Version von PostgreSQL offiziell unterstützt wird.

Um die große Menge an Informationen über PostgreSQL handhabbar zu machen, ist dieses Buch in mehrere Teile gegliedert. Jeder Teil richtet sich an eine andere Benutzergruppe oder an Benutzer in unterschiedlichen Phasen ihrer PostgreSQL-Erfahrung:

- Teil I ist eine informelle Einführung für neue Benutzer.
- Teil II dokumentiert die SQL-Abfragesprache, einschließlich Datentypen, Funktionen und Performance-Tuning auf Benutzerebene. Jeder PostgreSQL-Benutzer sollte diesen Teil lesen.
- Teil III beschreibt Installation und Administration des Servers. Jeder, der einen PostgreSQL-Server betreibt, ob für private Zwecke oder für andere, sollte diesen Teil lesen.
- Teil IV beschreibt die Programmierschnittstellen für PostgreSQL-Clientprogramme.
- Teil V enthält Informationen für fortgeschrittene Benutzer über die Erweiterungsmöglichkeiten des Servers. Zu den Themen gehören benutzerdefinierte Datentypen und Funktionen.
- Teil VI enthält Referenzinformationen zu SQL-Befehlen sowie Client- und Serverprogrammen. Dieser Teil unterstützt die anderen Teile mit strukturierten, nach Befehl oder Programm sortierten Informationen.
- Teil VII enthält verschiedene Informationen, die für PostgreSQL-Entwickler nützlich sein können.

## 1. Was ist PostgreSQL?

PostgreSQL ist ein objektrelationales Datenbankmanagementsystem (ORDBMS), das auf POSTGRES Version 4.21 basiert, das am Computer Science Department der University of California at Berkeley entwickelt wurde. POSTGRES führte viele Konzepte ein, die in einigen kommerziellen Datenbanksystemen erst viel später verfügbar wurden.

PostgreSQL ist ein Open-Source-Nachfolger dieses ursprünglichen Berkeley-Codes. Es unterstützt einen großen Teil des SQL-Standards und bietet viele moderne Funktionen:

- komplexe Abfragen
- Fremdschlüssel
- Trigger
- aktualisierbare Sichten
- transaktionale Integrität
- Multiversion Concurrency Control

Außerdem kann PostgreSQL vom Benutzer auf viele Arten erweitert werden, zum Beispiel durch Hinzufügen neuer:

- Datentypen
- Funktionen
- Operatoren
- Aggregatfunktionen
- Indexmethoden
- prozeduraler Sprachen

Aufgrund der liberalen Lizenz darf PostgreSQL von jedem kostenlos für jeden Zweck verwendet, verändert und weitergegeben werden, sei es privat, kommerziell oder akademisch.

## 2. Eine kurze Geschichte von PostgreSQL

Das objektrelationale Datenbankmanagementsystem, das heute als PostgreSQL bekannt ist, geht auf das POSTGRES-Paket der University of California at Berkeley zurück. Nach Jahrzehnten der Entwicklung ist PostgreSQL heute eines der fortschrittlichsten verfügbaren Open-Source-Datenbanksysteme.

Eine andere Darstellung dieser Geschichte findet sich in Dr. Joe Hellersteins Aufsatz "Looking Back at Postgres" [hell18].

### 2.1. Das Berkeley-POSTGRES-Projekt

Das POSTGRES-Projekt unter der Leitung von Professor Michael Stonebraker wurde von der Defense Advanced Research Projects Agency (DARPA), dem Army Research Office (ARO), der National Science Foundation (NSF) und ESL, Inc. gefördert. Die Implementierung von POSTGRES begann 1986. Die ursprünglichen Konzepte des Systems wurden in [ston86] vorgestellt, und die Definition des ursprünglichen Datenmodells erschien in [rowe87]. Das Design des damaligen Regelsystems wurde in [ston87a] beschrieben. Begründung und Architektur des Storage Managers wurden in [ston87b] ausführlich dargestellt.

POSTGRES durchlief seitdem mehrere Hauptversionen. Das erste "Demoware"-System wurde 1987 betriebsbereit und auf der ACM-SIGMOD-Konferenz 1988 gezeigt. Version 1, beschrieben in [ston90a], wurde im Juni 1989 an einige externe Benutzer freigegeben. Als Reaktion auf eine Kritik am ersten Regelsystem ([ston89]) wurde das Regelsystem neu entworfen ([ston90b]), und Version 2 erschien im Juni 1990 mit dem neuen Regelsystem. Version 3 erschien 1991 und ergänzte Unterstützung für mehrere Storage Manager, einen verbesserten Query Executor und ein neu geschriebenes Regelsystem. Die späteren Versionen bis Postgres95 konzentrierten sich größtenteils auf Portabilität und Zuverlässigkeit.

POSTGRES wurde zur Umsetzung vieler unterschiedlicher Forschungs- und Produktionsanwendungen verwendet. Dazu gehörten ein System zur Analyse von Finanzdaten, ein Paket zur Überwachung der Leistung von Strahltriebwerken, eine Datenbank zur Verfolgung von Asteroiden, eine medizinische Informationsdatenbank und mehrere geografische Informationssysteme. POSTGRES wurde außerdem an mehreren Universitäten als Lehrmittel eingesetzt. Schließlich griff Illustra Information Technologies, später in Informix aufgegangen und heute im Besitz von IBM, den Code auf und kommerzialisierte ihn. Ende 1992 wurde POSTGRES zum primären Datenmanager des wissenschaftlichen Rechenprojekts Sequoia 2000, das in [ston92] beschrieben ist.

Die externe Benutzergemeinschaft verdoppelte sich 1993 nahezu. Es wurde immer offensichtlicher, dass Pflege des Prototypcodes und Support große Mengen an Zeit beanspruchten, die eigentlich der Datenbankforschung hätten gewidmet werden sollen. Um diese Supportlast zu reduzieren, endete das Berkeley-POSTGRES-Projekt offiziell mit Version 4.2.

### 2.2. Postgres95

1994 ergänzten Andrew Yu und Jolly Chen einen SQL-Sprachinterpreter zu POSTGRES. Unter einem neuen Namen wurde Postgres95 anschließend im Web veröffentlicht, um als Open-Source-Nachfolger des ursprünglichen Berkeley-POSTGRES-Codes seinen eigenen Weg zu finden.

Der Postgres95-Code war vollständig in ANSI C geschrieben und um 25 Prozent verkleinert. Viele interne Änderungen verbesserten Performance und Wartbarkeit. Postgres95 Release 1.0.x lief im Wisconsin Benchmark etwa 30 bis 50 Prozent schneller als POSTGRES Version 4.2. Abgesehen von Fehlerkorrekturen waren dies die wichtigsten Verbesserungen:

- Die Abfragesprache PostQUEL wurde durch SQL ersetzt, das im Server implementiert wurde. Die Schnittstellenbibliothek `libpq` wurde nach PostQUEL benannt. Unterabfragen wurden erst ab PostgreSQL unterstützt, konnten in Postgres95 aber mit benutzerdefinierten SQL-Funktionen nachgebildet werden. Aggregatfunktionen wurden neu implementiert. Außerdem wurde Unterstützung für die `GROUP BY`-Klausel hinzugefügt.
- Ein neues Programm (`psql`) für interaktive SQL-Abfragen wurde bereitgestellt und verwendete GNU Readline. Es ersetzte weitgehend das alte Monitorprogramm.
- Eine neue Frontend-Bibliothek, `libpgtcl`, unterstützte Tcl-basierte Clients. Eine Beispiel-Shell, `pgtclsh`, stellte neue Tcl-Befehle zur Schnittstelle zwischen Tcl-Programmen und dem Postgres95-Server bereit.
- Die Large-Object-Schnittstelle wurde überarbeitet. Inversion Large Objects waren der einzige Mechanismus zum Speichern großer Objekte. Das Inversion File System wurde entfernt.
- Das instanzweite Regelsystem wurde entfernt. Regeln standen weiterhin als Rewrite-Regeln zur Verfügung.
- Mit dem Quellcode wurde ein kurzes Tutorial ausgeliefert, das reguläre SQL-Funktionen sowie Funktionen von Postgres95 einführte.
- Für den Build wurde GNU make statt BSD make verwendet. Außerdem konnte Postgres95 mit einem ungepatchten GCC kompiliert werden; die Datenausrichtung von `double` wurde korrigiert.

### 2.3. PostgreSQL

1996 wurde deutlich, dass der Name "Postgres95" der Zeit nicht standhalten würde. Es wurde ein neuer Name gewählt: PostgreSQL, um die Beziehung zwischen dem ursprünglichen POSTGRES und den neueren Versionen mit SQL-Fähigkeit widerzuspiegeln. Gleichzeitig wurde die Versionsnummerierung auf 6.0 gesetzt und damit wieder in die Reihenfolge gebracht, die ursprünglich mit dem Berkeley-POSTGRES-Projekt begonnen hatte.

Postgres gilt weiterhin als offizieller Projektname, sowohl aus Tradition als auch weil viele Menschen Postgres leichter aussprechen können als PostgreSQL.

Während der Entwicklung von Postgres95 lag der Schwerpunkt darauf, bestehende Probleme im Servercode zu identifizieren und zu verstehen. Mit PostgreSQL verlagerte sich der Schwerpunkt auf die Erweiterung von Funktionen und Fähigkeiten, auch wenn die Arbeit in allen Bereichen fortgesetzt wird.

Details zu den Änderungen in den einzelnen PostgreSQL-Versionen seitdem finden sich unter <https://www.postgresql.org/docs/release/>.

## 3. Konventionen

In der Synopsis eines Befehls werden die folgenden Konventionen verwendet: Klammern (`[` und `]`) kennzeichnen optionale Teile. Geschweifte Klammern (`{` und `}`) und senkrechte Striche (`|`) zeigen an, dass eine Alternative gewählt werden muss. Punkte (`...`) bedeuten, dass das vorangehende Element wiederholt werden kann. Alle anderen Symbole, einschließlich runder Klammern, sind wörtlich zu verstehen.

Wo es die Klarheit verbessert, werden SQL-Befehlen der Prompt `=>` und Shell-Befehlen der Prompt `$` vorangestellt. Normalerweise werden Prompts jedoch nicht gezeigt.

Ein Administrator ist im Allgemeinen eine Person, die für Installation und Betrieb des Servers zuständig ist. Ein Benutzer kann jeder sein, der irgendeinen Teil des PostgreSQL-Systems verwendet oder verwenden möchte. Diese Begriffe sollten nicht zu eng ausgelegt werden; dieses Buch trifft keine festen Annahmen über Verfahren der Systemadministration.

## 4. Weitere Informationen

Neben der Dokumentation, also diesem Buch, gibt es weitere Ressourcen zu PostgreSQL:

**Wiki**

Das PostgreSQL-Wiki enthält die FAQ-Liste (Frequently Asked Questions) des Projekts, die TODO-Liste und ausführliche Informationen zu vielen weiteren Themen.

**Website**

Die PostgreSQL-Website enthält Details zur neuesten Version und weitere Informationen, die Ihre Arbeit oder Ihr Experimentieren mit PostgreSQL produktiver machen.

**Mailinglisten**

Die Mailinglisten sind ein guter Ort, um Fragen beantwortet zu bekommen, Erfahrungen mit anderen Benutzern auszutauschen und die Entwickler zu kontaktieren. Details finden Sie auf der PostgreSQL-Website.

**Sie selbst**

PostgreSQL ist ein Open-Source-Projekt. Als solches ist es für laufende Unterstützung auf die Benutzergemeinschaft angewiesen. Wenn Sie beginnen, PostgreSQL zu verwenden, werden Sie sich auf andere stützen, sei es über die Dokumentation oder über die Mailinglisten. Erwägen Sie, Ihr Wissen wieder beizutragen. Lesen Sie die Mailinglisten und beantworten Sie Fragen. Wenn Sie etwas lernen, das nicht in der Dokumentation steht, schreiben Sie es auf und tragen Sie es bei. Wenn Sie dem Code Funktionen hinzufügen, tragen Sie sie bei.

## 5. Richtlinien zur Fehlermeldung

Wenn Sie einen Fehler in PostgreSQL finden, möchten wir davon erfahren. Ihre Fehlermeldungen spielen eine wichtige Rolle dabei, PostgreSQL zuverlässiger zu machen, denn selbst größte Sorgfalt kann nicht garantieren, dass jeder Teil von PostgreSQL auf jeder Plattform unter allen Umständen funktioniert.

Die folgenden Hinweise sollen Ihnen helfen, Fehlermeldungen so zu formulieren, dass sie effektiv bearbeitet werden können. Niemand ist verpflichtet, ihnen zu folgen, aber es ist meist zum Vorteil aller Beteiligten.

Wir können nicht versprechen, jeden Fehler sofort zu beheben. Wenn der Fehler offensichtlich, kritisch oder für viele Benutzer relevant ist, stehen die Chancen gut, dass sich jemand darum kümmert. Es kann auch passieren, dass wir Sie bitten, auf eine neuere Version zu aktualisieren, um zu prüfen, ob der Fehler dort ebenfalls auftritt. Oder wir entscheiden, dass der Fehler nicht behoben werden kann, bevor eine größere geplante Überarbeitung abgeschlossen ist. Vielleicht ist er auch einfach zu schwierig, während wichtigere Dinge anstehen. Wenn Sie sofort Hilfe benötigen, sollten Sie einen kommerziellen Supportvertrag in Betracht ziehen.

### 5.1. Fehler erkennen

Ein Fehler liegt vor, wenn PostgreSQL etwas nicht so tut, wie es laut Dokumentation tun sollte, oder wenn die Dokumentation unklar oder falsch ist. Dazu gehören unter anderem die folgenden Fälle:

- Ein Programm beendet sich mit einem fatalen Signal oder einer Betriebssystemfehlermeldung, die auf ein Problem im Programm hinweist. Eine Meldung wie "disk full" ist dagegen gewöhnlich kein PostgreSQL-Fehler, weil Sie dieses Problem selbst beheben müssen.
- Ein Programm erzeugt für eine bestimmte Eingabe eine falsche Ausgabe.
- Ein Programm verweigert gültige Eingaben, wie sie in der Dokumentation definiert sind.
- Ein Programm akzeptiert ungültige Eingaben ohne Hinweis oder Fehlermeldung. Bedenken Sie dabei, dass Ihre Vorstellung von ungültiger Eingabe aus unserer Sicht eine Erweiterung oder Kompatibilität mit traditionellem Verhalten sein kann.
- PostgreSQL lässt sich auf unterstützten Plattformen nicht gemäß Anleitung kompilieren, bauen oder installieren.

Hier bezeichnet "Programm" jede ausführbare Datei, nicht nur den Backend-Prozess.

Langsamkeit oder hoher Ressourcenverbrauch sind nicht unbedingt Fehler. Lesen Sie die Dokumentation oder fragen Sie auf einer der Mailinglisten nach Hilfe beim Tuning Ihrer Anwendungen. Auch eine Abweichung vom SQL-Standard ist nicht notwendigerweise ein Fehler, sofern die Konformität für die konkrete Funktion nicht ausdrücklich zugesichert wird.

Bevor Sie fortfahren, prüfen Sie die TODO-Liste und die FAQ, um zu sehen, ob Ihr Fehler bereits bekannt ist. Wenn Sie die Informationen in der TODO-Liste nicht deuten können, melden Sie Ihr Problem. Das Mindeste, was wir tun können, ist, die TODO-Liste klarer zu machen.

### 5.2. Was gemeldet werden sollte

Spekulieren Sie nicht darüber, was Ihrer Meinung nach schiefgelaufen ist, was es "zu tun schien" oder welcher Teil des Programms schuld ist. Wenn Sie mit der Implementierung nicht vertraut sind, würden Sie wahrscheinlich falsch raten und uns nicht helfen. Auch wenn Sie vertraut sind, sind sachkundige Erklärungen eine gute Ergänzung, aber kein Ersatz für Fakten. Wenn wir den Fehler beheben sollen, müssen wir ihn zunächst selbst sehen können. Die reinen Fakten zu melden ist relativ einfach, häufig fehlen aber wichtige Details, weil jemand dachte, sie seien unwichtig oder der Bericht werde ohnehin verstanden.

Jede Fehlermeldung sollte Folgendes enthalten:

- Die genaue Abfolge der Schritte ab Programmstart, die nötig ist, um das Problem zu reproduzieren. Sie sollte in sich geschlossen sein; es reicht nicht, eine einzelne `SELECT`-Anweisung zu senden, wenn vorherige `CREATE TABLE`- und `INSERT`-Anweisungen für das Ergebnis relevant sind.
- Die erhaltene Ausgabe. Schreiben Sie nicht nur, dass etwas "nicht funktioniert" oder "abgestürzt" sei. Wenn es eine Fehlermeldung gibt, zeigen Sie sie, auch wenn Sie sie nicht verstehen.
- Die erwartete Ausgabe. Diese Angabe ist sehr wichtig, denn sonst müssten wir die genaue Semantik Ihrer Befehle erraten.
- Alle Kommandozeilenoptionen und sonstigen Startoptionen, einschließlich relevanter Umgebungsvariablen oder Konfigurationsdateien, die Sie gegenüber der Vorgabe geändert haben.
- Alles, was Sie anders gemacht haben als in der Installationsanleitung beschrieben.
- Die PostgreSQL-Version. Mit `SELECT version();` können Sie die Version des verbundenen Servers ermitteln. Die meisten ausführbaren Programme unterstützen außerdem eine Option `--version`, mindestens `postgres --version` und `psql --version` sollten funktionieren.
- Plattforminformationen, darunter Kernelname und -version, C-Bibliothek, Prozessor, Speicherinformationen und Ähnliches. In den meisten Fällen reichen Hersteller und Version aus; bei Installationsproblemen sind außerdem Informationen zur Toolchain auf Ihrer Maschine erforderlich.

Scheuen Sie sich nicht, wenn Ihre Fehlermeldung recht lang wird. Es ist besser, beim ersten Mal alles zu melden, als dass wir Ihnen die Fakten mühsam entlocken müssen. Wenn Ihre Eingabedateien sehr groß sind, ist es allerdings fair, zuerst zu fragen, ob jemand sie ansehen möchte. Weitere Hinweise zum Melden von Fehlern finden sich unter <https://www.chiark.greenend.org.uk/~sgtatham/bugs.html>.

Verbringen Sie nicht Ihre ganze Zeit damit herauszufinden, welche Änderungen an der Eingabe das Problem verschwinden lassen. Das hilft wahrscheinlich nicht bei der Lösung. Falls sich herausstellt, dass der Fehler nicht sofort behoben werden kann, bleibt immer noch Zeit, einen Workaround zu finden und weiterzugeben. Verschwenden Sie außerdem keine Zeit damit zu raten, warum der Fehler existiert. Das finden wir früh genug heraus.

Vermeiden Sie beim Schreiben einer Fehlermeldung verwirrende Terminologie. Das Softwarepaket insgesamt heißt "PostgreSQL", manchmal kurz "Postgres". Wenn Sie speziell über den Backend-Prozess sprechen, sagen Sie das auch und schreiben Sie nicht einfach "PostgreSQL stürzt ab". Der Absturz eines einzelnen Backend-Prozesses ist etwas anderes als der Absturz des übergeordneten `postgres`-Prozesses. Auch Clientprogramme wie das interaktive Frontend `psql` sind vollständig vom Backend getrennt. Bitte seien Sie möglichst genau, ob das Problem auf Client- oder Serverseite liegt.

### 5.3. Wo Fehler gemeldet werden sollten

Fehlermeldungen können an die Mailingliste `pgsql-bugs@lists.postgresql.org` gesendet werden. Verwenden Sie bitte einen aussagekräftigen Betreff, vielleicht mit Teilen der Fehlermeldung.

Eine andere Möglichkeit ist das Fehlermeldeformular auf der Projektwebsite unter <https://www.postgresql.org/account/submitbug/>. Eine dort eingetragene Fehlermeldung wird an die Mailingliste `pgsql-bugs@lists.postgresql.org` gesendet.

Wenn Ihre Fehlermeldung sicherheitsrelevante Auswirkungen hat und nicht sofort in öffentlichen Archiven sichtbar werden soll, senden Sie sie nicht an `pgsql-bugs`. Sicherheitsprobleme können vertraulich an `security@postgresql.org` gemeldet werden.
