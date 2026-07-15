# 40. Prozedurale Sprachen

PostgreSQL erlaubt, benutzerdefinierte Funktionen außer in SQL und C auch in anderen Sprachen zu schreiben. Diese anderen Sprachen werden allgemein prozedurale Sprachen (PLs) genannt. Bei einer Funktion, die in einer prozeduralen Sprache geschrieben ist, besitzt der Datenbankserver kein eingebautes Wissen darüber, wie der Quelltext der Funktion zu interpretieren ist. Stattdessen wird diese Aufgabe an einen speziellen Handler übergeben, der die Details der Sprache kennt. Der Handler kann entweder die gesamte Arbeit des Parsens, der Syntaxanalyse, der Ausführung usw. selbst erledigen, oder er kann als „Klebstoff“ zwischen PostgreSQL und einer vorhandenen Implementierung einer Programmiersprache dienen. Der Handler selbst ist eine C-Funktion, die in ein Shared Object kompiliert und bei Bedarf geladen wird, genau wie jede andere C-Funktion.

Derzeit sind in der Standarddistribution von PostgreSQL vier prozedurale Sprachen verfügbar: PL/pgSQL ([Kapitel 41](41_PL_pgSQL_SQL_Prozedursprache.md)), PL/Tcl ([Kapitel 42](42_PL_Tcl_Tcl_Prozedursprache.md)), PL/Perl ([Kapitel 43](43_PL_Perl_Perl_Prozedursprache.md)) und PL/Python ([Kapitel 44](44_PL_Python_Python_Prozedursprache.md)). Weitere prozedurale Sprachen sind verfügbar, aber nicht in der Kerndistribution enthalten. [Anhang H](H_Externe_Projekte.md) enthält Informationen dazu, wie man sie findet. Außerdem können Benutzer weitere Sprachen definieren; die Grundlagen der Entwicklung einer neuen prozeduralen Sprache werden in [Kapitel 57](57_Einen_Handler_für_prozedurale_Sprachen_schreiben.md) behandelt.

## 40.1. Prozedurale Sprachen installieren

Eine prozedurale Sprache muss in jeder Datenbank „installiert“ werden, in der sie verwendet werden soll. Prozedurale Sprachen, die in der Datenbank `template1` installiert sind, stehen jedoch automatisch in allen danach erstellten Datenbanken zur Verfügung, weil ihre Einträge aus `template1` durch `CREATE DATABASE` kopiert werden. Der Datenbankadministrator kann also entscheiden, welche Sprachen in welchen Datenbanken verfügbar sind, und bestimmte Sprachen bei Bedarf standardmäßig bereitstellen.

Für die mit der Standarddistribution gelieferten Sprachen muss lediglich `CREATE EXTENSION language_name` ausgeführt werden, um die Sprache in der aktuellen Datenbank zu installieren. Das unten beschriebene manuelle Verfahren wird nur für die Installation von Sprachen empfohlen, die nicht als Erweiterungen paketiert wurden.

### Manuelle Installation prozeduraler Sprachen

Eine prozedurale Sprache wird in fünf Schritten in einer Datenbank installiert. Diese Schritte müssen von einem Datenbank-Superuser ausgeführt werden. In den meisten Fällen sollten die erforderlichen SQL-Befehle als Installationsskript einer „Extension“ paketiert sein, sodass `CREATE EXTENSION` verwendet werden kann, um sie auszuführen.

1. Das Shared Object für den Sprach-Handler muss kompiliert und in ein geeignetes Bibliotheksverzeichnis installiert werden. Das funktioniert genauso wie das Bauen und Installieren von Modulen mit normalen benutzerdefinierten C-Funktionen; siehe [Abschnitt 36.10.5](36_SQL_erweitern.md#36105-dynamisch-geladene-funktionen-kompilieren-und-linken). Häufig hängt der Sprach-Handler von einer externen Bibliothek ab, die die eigentliche Programmiersprachen-Engine bereitstellt. In diesem Fall muss auch diese Bibliothek installiert werden.

2. Der Handler muss mit folgendem Befehl deklariert werden:

   ```sql
   CREATE FUNCTION handler_function_name()
       RETURNS language_handler
       AS 'path-to-shared-object'
       LANGUAGE C;
   ```

   Der spezielle Rückgabetyp `language_handler` teilt dem Datenbanksystem mit, dass diese Funktion keinen der definierten SQL-Datentypen zurückgibt und nicht direkt in SQL-Anweisungen verwendbar ist.

3. Optional kann der Sprach-Handler eine „Inline“-Handler-Funktion bereitstellen, die anonyme Codeblöcke (`DO`-Befehle) ausführt, die in dieser Sprache geschrieben sind. Wenn die Sprache eine Inline-Handler-Funktion bereitstellt, deklarieren Sie sie mit einem Befehl wie:

   ```sql
   CREATE FUNCTION inline_function_name(internal)
       RETURNS void
       AS 'path-to-shared-object'
       LANGUAGE C;
   ```

4. Optional kann der Sprach-Handler eine „Validator“-Funktion bereitstellen, die eine Funktionsdefinition auf Korrektheit prüft, ohne sie tatsächlich auszuführen. Die Validator-Funktion wird von `CREATE FUNCTION` aufgerufen, sofern sie existiert. Wenn die Sprache eine Validator-Funktion bereitstellt, deklarieren Sie sie mit einem Befehl wie:

   ```sql
   CREATE FUNCTION validator_function_name(oid)
       RETURNS void
       AS 'path-to-shared-object'
       LANGUAGE C STRICT;
   ```

5. Schließlich muss die PL mit folgendem Befehl deklariert werden:

   ```sql
   CREATE [TRUSTED] LANGUAGE language_name
       HANDLER handler_function_name
       [INLINE inline_function_name]
       [VALIDATOR validator_function_name] ;
   ```

   Das optionale Schlüsselwort `TRUSTED` gibt an, dass die Sprache keinen Zugriff auf Daten gewährt, auf die der Benutzer sonst keinen Zugriff hätte. Vertrauenswürdige Sprachen sind für normale Datenbankbenutzer gedacht, also Benutzer ohne Superuser-Privileg, und erlauben ihnen, sicher Funktionen und Prozeduren zu erstellen. Da PL-Funktionen innerhalb des Datenbankservers ausgeführt werden, sollte das Flag `TRUSTED` nur für Sprachen vergeben werden, die keinen Zugriff auf Interna des Datenbankservers oder auf das Dateisystem erlauben. Die Sprachen PL/pgSQL, PL/Tcl und PL/Perl gelten als vertrauenswürdig; die Sprachen PL/TclU, PL/PerlU und PL/PythonU sind dafür ausgelegt, uneingeschränkte Funktionalität bereitzustellen, und sollten nicht als vertrauenswürdig markiert werden.

Beispiel 40.1 zeigt, wie das manuelle Installationsverfahren mit der Sprache PL/Perl funktionieren würde.

### Beispiel 40.1. Manuelle Installation von PL/Perl

Der folgende Befehl teilt dem Datenbankserver mit, wo er das Shared Object für die Call-Handler-Funktion der Sprache PL/Perl findet:

```sql
CREATE FUNCTION plperl_call_handler() RETURNS language_handler AS
    '$libdir/plperl' LANGUAGE C;
```

PL/Perl besitzt eine Inline-Handler-Funktion und eine Validator-Funktion, daher deklarieren wir auch diese:

```sql
CREATE FUNCTION plperl_inline_handler(internal) RETURNS void AS
    '$libdir/plperl' LANGUAGE C STRICT;

CREATE FUNCTION plperl_validator(oid) RETURNS void AS
    '$libdir/plperl' LANGUAGE C STRICT;
```

Der Befehl:

```sql
CREATE TRUSTED LANGUAGE plperl
    HANDLER plperl_call_handler
    INLINE plperl_inline_handler
    VALIDATOR plperl_validator;
```

legt dann fest, dass die zuvor deklarierten Funktionen für Funktionen und Prozeduren aufgerufen werden sollen, deren Sprachattribut `plperl` ist.

In einer Standardinstallation von PostgreSQL wird der Handler für die Sprache PL/pgSQL gebaut und im „library“-Verzeichnis installiert. Außerdem ist die Sprache PL/pgSQL selbst in allen Datenbanken installiert. Wenn Tcl-Unterstützung konfiguriert ist, werden die Handler für PL/Tcl und PL/TclU gebaut und im Bibliotheksverzeichnis installiert, die Sprache selbst wird jedoch standardmäßig in keiner Datenbank installiert. Ebenso werden die Handler für PL/Perl und PL/PerlU gebaut und installiert, wenn Perl-Unterstützung konfiguriert ist, und der Handler für PL/PythonU wird installiert, wenn Python-Unterstützung konfiguriert ist; diese Sprachen werden aber nicht standardmäßig installiert.
