# baculovirus-genome-gathering

Dieser Workflow identifiziert vollständige Baculovirus-Nucleotide-Records in NCBI, lädt die von NCBI bereitgestellten Originaldateien herunter und erzeugt daraus eine reproduzierbare Metadatentabelle.

## Datenquelle und Suchstrategie

Die einzige externe Datenquelle ist NCBI. Ausgangspunkt ist die Nucleotide-Datenbank (`nuccore`) mit der unveränderten Suchabfrage:

`Baculoviridae[Organism] AND complete genome[Title]`

Die Abfrage wird mit ESearch und NCBI History/WebEnv ausgeführt. Abfrage, Trefferzahl, Zeitstempel und technische Schnittstellen werden in `data/metadata/ncbi_query_metadata.json` gespeichert. Records werden nicht aufgrund eigener biologischer Kriterien entfernt.

Für jede eindeutige Accession werden, soweit NCBI verfügbar, FASTA und GenBank Flatfile sowie natives NCBI-GFF3 über EFetch gespeichert (`db=nuccore`, `rettype` = `fasta` / `gb` / `gff3`, `retmode=text`; das GFF3 wird von NCBI annotwriter ausgeliefert). Das GFF3 wird nicht selbst aus GenBank erzeugt. Die Beziehung zwischen GenBank- und RefSeq-Record wird ausschließlich über ELink (`nuccore_nuccore_gbrs`) übernommen.

Heruntergeladene Dateien werden nicht allein wegen ihrer Existenz als gültig betrachtet, sondern gegen Accession, Version und die von NCBI gemeldete Sequenzlänge geprüft; ungültige oder unvollständige Dateien werden verworfen und erneut geladen.

## Inkrementelle Wiederholungsläufe

Der Workflow ist inkrementell. Jeder Lauf führt die vollständige NCBI-Suche aus und vergleicht das Ergebnis mit der vorhandenen `data/metadata/baculovirus_genome_metadata.tsv`. Vollständig verarbeitet werden nur:

- neue Accessions, die noch nicht in der Tabelle stehen,
- Accessions, deren `accession_version` sich bei NCBI geändert hat,
- Records, deren Pflichtfelder oder lokale Dateien nachweislich fehlen.

Alle übrigen Zeilen werden unverändert aus der bestehenden Tabelle übernommen. Für sie erfolgt keine ESummary-, Taxonomy-, ELink- oder EFetch-Abfrage und keine erneute Inhaltsprüfung der lokalen Dateien. Die Bestandsaufnahme der aktuellen `accession.version`-Werte läuft über EFetch mit `rettype="acc"` (rund 7 kB für 630 Records) statt über einen vollständigen ESummary-Abruf.

Zu Beginn jedes Laufs wird ausgegeben, wie viele Records gefunden, unverändert, neu, versionsgeändert, unvollständig oder nicht mehr im Suchergebnis enthalten sind. Ein Lauf ohne NCBI-Änderungen dauert wenige Sekunden und schreibt die Metadatentabelle nicht neu, weil nur bei tatsächlicher inhaltlicher Änderung geschrieben wird.

Records, die nicht mehr im aktuellen Suchergebnis erscheinen, werden nicht gelöscht. Sie bleiben in der Tabelle, werden gezählt und in `data/logs/records_not_in_current_search.tsv` aufgeführt.

Steuerung im Chunk "Laufmodus":

- `test_accessions` – erzwingt die gezielte Neuverarbeitung genau dieser Accessions; alle übrigen bleiben unverändert. Alternativ über die Umgebungsvariable `BACULOVIRUS_TEST_ACCESSIONS`.
- `recheck_unavailable_gff3` – fragt Records erneut ab, für die NCBI bisher kein GFF3 bereitgestellt hat.
- `force_reprocess_all` – ignoriert den Bestand und baut alles neu auf.

## Versionskontrolle

Versioniert werden `baculovirus-genome-gathering.qmd`, `README.md`, `.gitignore` und der Inhalt von `data/metadata/`. Die Metadatentabelle wird bewusst mitversioniert, damit nachvollziehbar bleibt, welche NCBI-Records die Grundlage eines bestimmten Projektstands waren.

Nicht versioniert werden die reproduzierbaren NCBI-Rohdownloads (`genomes/fasta/`, `genomes/genbank/`, `genomes/gff3/`), Lauf-Artefakte (`data/cache/`, `data/logs/`), die Quarto-Renderausgabe sowie `.Renviron`. Der API-Key darf in keiner von Git erfassten Datei stehen.

## Einrichtung

Benötigt werden R, Quarto sowie `rentrez`, `jsonlite` und `XML`. Der API-Key wird ausschließlich lokal über `.Renviron` bereitgestellt:

```text
ENTREZ_KEY=YOUR_NCBI_API_KEY
```

`.Renviron` ist von Git ausgeschlossen. Der tatsächliche Key darf nicht in QMD, README, Logs oder Ergebnisdateien eingetragen werden.

## Workflow starten

Im Projektverzeichnis `baculovirus-genome-gathering.qmd` öffnen und rendern, oder mit Quarto rendern. Der Workflow ist inkrementell: bereits vorhandene und valide Dateien werden übersprungen; fehlgeschlagene Arbeitsschritte werden in `data/logs/failed_downloads.tsv` protokolliert.

## Erzeugte Dateien

`data/metadata/baculovirus_genome_metadata.tsv` enthält die aus NCBI-Records, NCBI-Taxonomy und Sequenzen reproduzierbar erzeugten Metadaten. `data/metadata/ncbi_query_metadata.json` dokumentiert die Provenienz. Originaldateien liegen unter `genomes/fasta/`, `genomes/genbank/` und `genomes/gff3/`.

Die Referenzdatei `genome_metadata.tsv` wird ausschließlich zur Feldplanung verwendet; ihre Werte werden nicht übernommen. Die aktuelle ICTV-Species-Zuordnung, historische Namensharmonisierung, phylogenetische Analysen, Core-Gene-Analysen und biologische Korrekturen sind nicht Bestandteil dieses Workflows.