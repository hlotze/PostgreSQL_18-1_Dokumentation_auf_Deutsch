# Anhang N. Farbunterstützung

Die meisten Programme im PostgreSQL-Paket können farbige Konsolenausgabe erzeugen. Dieser Anhang beschreibt, wie diese Ausgabe konfiguriert wird.

### N.1. Wann Farbe verwendet wird

Um farbige Ausgabe zu verwenden, setzen Sie die Umgebungsvariable `PG_COLOR` wie folgt:

1. Wenn der Wert `always` ist, wird Farbe verwendet.

2. Wenn der Wert `auto` ist und der Standardfehlerstrom mit einem Terminalgerät verbunden ist, wird Farbe verwendet.

3. Andernfalls wird keine Farbe verwendet.

### N.2. Die Farben konfigurieren

Die tatsächlich zu verwendenden Farben werden über die Umgebungsvariable `PG_COLORS` konfiguriert. Der Wert ist eine durch Doppelpunkte getrennte Liste von `key=value`-Paaren. Die Schlüssel geben an, wofür die Farbe verwendet wird. Die Werte sind SGR-Spezifikationen (Select Graphic Rendition), die vom Terminal interpretiert werden.

Die folgenden Schlüssel werden derzeit verwendet:

- `error`
  - Wird verwendet, um den Text `error` in Fehlermeldungen hervorzuheben.

- `warning`
  - Wird verwendet, um den Text `warning` in Warnmeldungen hervorzuheben.

- `note`
  - Wird verwendet, um die Texte `detail` und `hint` in solchen Meldungen hervorzuheben.

- `locus`
  - Wird verwendet, um Ortsinformationen hervorzuheben, zum Beispiel Programmname und Dateiname in Meldungen.

Der Standardwert ist:

```text
error=01;31:warning=01;35:note=01;36:locus=01
```

Dabei steht `01;31` für fett rot, `01;35` für fett magenta, `01;36` für fett cyan und `01` für fett in der Standardfarbe.

> **Tipp:** Dieses Farbspezifikationsformat wird auch von anderen Softwarepaketen verwendet, etwa GCC, GNU coreutils und GNU grep.
