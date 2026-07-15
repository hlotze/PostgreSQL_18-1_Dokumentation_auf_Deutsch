# PostgreSQL 18.1 Dokumentation auf Deutsch

Dieses Repository enthält eine ins Deutsche übertragene Fassung der PostgreSQL-18.1-Dokumentation im Markdown-Format und umfasst die über 3.000 Seiten des Originals.

Sie entstand mit Unterstützung von ChatGPT, aus dem Wunsch heraus, eine solche Übersetzung im Markdown-Format zu haben, und als persönlicher Proof of Concept, ob KI eine brauchbare Übersetzung in Sprache und Formatierung liefern kann. Aus meiner Sicht ist das hier ziemlich gelungen.

Die Dokumentation ist kapitelweise in einzelne Markdown-Dateien aufgeteilt. Der Einstiegspunkt ist das Inhaltsverzeichnis:

- [Inhaltsverzeichnis](md/00_Inhaltsverzeichnis.md)
- [Vorwort](md/00_Vorwort.md)
- [Teil I. Tutorial](md/Teil_I_Tutorial.md)
- [Teil VIII. Anhänge](md/Teil_VIII_Anhänge.md)

## Status

Die Übersetzung und Markdown-Aufbereitung wurde nach bestem Wissen und Gewissen erstellt. Sie ist keine offizielle Übersetzung des PostgreSQL-Projekts.

Für die Richtigkeit, Vollständigkeit und Aktualität der Angaben wird keine Gewähr übernommen.

Die ursprüngliche Dokumentation stammt aus der PostgreSQL-18.1-Dokumentation. Codebeispiele, SQL-Befehle, Funktionsnamen, Dateipfade und technische Bezeichner wurden soweit möglich unverändert beibehalten.

## Struktur

- `md/`
  - deutsche Markdown-Fassung der PostgreSQL-Dokumentation
  - ein Kapitel oder Anhang pro Datei
  - `00_Inhaltsverzeichnis.md` als zentraler Einstieg
- `README.md`
  - diese Projektübersicht

Lokale Arbeitsdateien, Rohquellen, PDF-Extrakte und Hilfsskripte sind über `.gitignore` ausgeschlossen und gehören nicht zum Veröffentlichungsumfang.

## Hinweis zur Nutzung

Diese Fassung eignet sich zum Lesen in GitHub, Obsidian, Markdown-Editoren oder statischen Dokumentationswerkzeugen. Interne Links sind als relative Markdown-Links angelegt.

Wenn fachliche Details kritisch sind, sollte die englische Originaldokumentation des PostgreSQL-Projekts zum Abgleich herangezogen werden:

<https://www.postgresql.org/docs/>

## Rechtlicher Hinweis

PostgreSQL Database Management System, also known as Postgres, formerly known as Postgres95.

Portions Copyright (c) 1996-2025, PostgreSQL Global Development Group.

Portions Copyright (c) 1994, The Regents of the University of California.

Permission to use, copy, modify, and distribute this software and its documentation for any purpose, without fee, and without a written agreement is hereby granted, provided that the above copyright notice and this paragraph and the following two paragraphs appear in all copies.

In no event shall the University of California be liable to any party for direct, indirect, special, incidental, or consequential damages, including lost profits, arising out of the use of this software and its documentation, even if the University of California has been advised of the possibility of such damage.

The University of California specifically disclaims any warranties, including, but not limited to, the implied warranties of merchantability and fitness for a particular purpose. The software provided hereunder is on an "as-is" basis, and the University of California has no obligations to provide maintenance, support, updates, enhancements, or modifications.
