# Technische Dokumentation: KI-gestütztes Artikeldaten-Mapping

## Überblick

Dieser Workflow automatisiert die Extraktion und das Mapping von Artikelstammdaten aus Möbelhersteller-Dokumenten (Excel oder PDF) in ein standardisiertes ERP-Zielformat. Der Workflow ist generisch aufgebaut — er verarbeitet Dokumente von beliebigen Möbelherstellern ohne herstellerspezifische Konfiguration.

---

## Architektur

### Infrastruktur

Das System läuft vollständig in einer selbstverwalteten Docker-Umgebung mit zwei Diensten:

- **n8n** — Workflow-Automatisierungsengine, zuständig für Dateieingang, Orchestrierung und Antwort
- **Ollama** — lokaler LLM-Inferenzserver, hostet das DeepSeek-R1-Modell

Beide Dienste laufen im selben Docker-Netzwerk (`moebelhaus` Bridge-Netzwerk) und kommunizieren über den Service-Hostnamen (z. B. `http://ollama:11434`).

Es werden keine externen API-Aufrufe während der Dokumentenverarbeitung durchgeführt. Alle Daten verbleiben innerhalb der Unternehmensinfrastruktur — die Lösung ist vollständig DSGVO-konform.

### Workflow-Struktur

```
Webhook (HTTP POST) ─────────────────────────────────────────────┐
                                                                   │
                    ┌──────────────────────────────────────────────┤
                    │                                              │
Read Headers ──► Extract from JSON ──► Parse Headers              │
                                            │                      │
                                            ▼                      ▼
                                      Merge Headers ◄─── Normalize Data
                                            │
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                         Pass 1         Pass 2         Pass 3
                      (Shared)       (Varianten)    (Restfelder)
                              │             │             │
                              └──────┬──────┘             │
                                     ▼                     │
                              Merge Known ◄────────────────┘
                              Specialized
                                     │
                                     ▼
                              Merge Remaining
                                     │
                                     ▼
                           XLSX Content Builder
                                     │
                                     ▼
                           Return Ziel-Tabelle
```

---

## KI-Modellauswahl

### Modell: DeepSeek R1 7B (via Ollama)

**Begründung der Modellwahl:**

**1. Open Source & Self-Hostable**
DeepSeek R1 ist unter einer permissiven Open-Source-Lizenz veröffentlicht und kann lokal über Ollama mit derselben Docker-basierten Infrastruktur betrieben werden. Es sind keine externen API-Schlüssel oder Cloud-Abhängigkeiten erforderlich.

**2. Formatagnostische Informationsextraktion**
Die Aufgabe erfordert die Verarbeitung von Dokumenten beliebiger Möbelhersteller mit unbekannten Layouts und Terminologien. Die starken Reasoning-Fähigkeiten von DeepSeek R1 ermöglichen die Identifikation und Extraktion von Produktinformationen aus unterschiedlichsten Dokumentstrukturen — ohne Nachtraining oder Fine-Tuning.

**3. DSGVO-Konformität**
Durch den lokalen Betrieb von DeepSeek R1 via Ollama werden keine Artikelstammdaten, Lieferantendokumente oder geschäftssensiblen Informationen an externe Dienste übermittelt. Dies ist besonders relevant für europäische Kunden, die strengen Datenschutzanforderungen unterliegen. Die Architektur ist vollständig innerhalb der eigenen Unternehmensinfrastruktur gekapselt.

**4. Kontrollierte und planbare Kosten**
Im Gegensatz zu API-basierten Modellen (GPT-4o, Claude Sonnet) entstehen bei lokaler Inferenz keine tokenbasierten Kosten. Die Betriebskosten beschränken sich auf die Serverinfrastruktur, was eine budgetplanbare Verarbeitung großer Artikelstammdatenmengen ermöglicht.

**5. OpenAI-kompatible API**
Ollama stellt DeepSeek R1 über eine OpenAI-kompatible REST-API bereit (`/v1/chat/completions`). Der Workflow kann dadurch auf jeden OpenAI-kompatiblen Anbieter (GPT-4o, Mistral, zukünftige DeepSeek-Versionen) migriert werden, indem lediglich die Endpunkt-URL und der Modellname geändert werden — ohne Anpassungen am Workflow selbst.

**6. Kein Vendor Lock-in**
Modell-Upgrades (z. B. neuere DeepSeek-Version) erfordern lediglich einen `ollama pull`-Befehl. Der Workflow selbst bleibt unverändert.

**7. Hardware-Verfügbarkeit**
DeepSeek R1 7B wurde als Standardmodell gewählt, das eine Balance zwischen Extraktionsqualität und Hardware-Zugänglichkeit bietet. Es läuft auf rein CPU-basierten Servern, wie sie in europäischen KMU-Umgebungen verbreitet sind. Größere Modelle (14B, 32B) bieten bessere Extraktionsqualität, setzen jedoch GPU-Hardware voraus, um produktionstauglich zu sein.

**Bekannte Einschränkungen:**
- Das 7B-Modell läuft in der aktuellen Konfiguration auf CPU, was zu Antwortzeiten von 1–3 Minuten pro Dokument führt. GPU-Beschleunigung wird für den Produktionsbetrieb empfohlen (siehe Produktionsempfehlungen).
- Als destilliertes Reasoning-Modell benötigt DeepSeek R1 7B bei komplexer Feldunterscheidung explizite Prompt-Anweisungen. Dies wird durch die Drei-Pass-Extraktionsarchitektur adressiert.

---

## Extraktionsarchitektur: Drei-Pass-Ansatz

Um die Extraktionsqualität zu maximieren und gleichzeitig Prompt-Komplexität und Inferenzzeit zu steuern, unterteilt der Workflow die KI-Extraktion in drei parallele Aufrufe:

### Pass 1 — Gemeinsame Produktfelder
Extrahiert Informationen, die für alle Varianten eines Produkts gelten:
- `ArtBez`, `Hersteller`, Abmessungen (`Breite_mm`, `Hoehe_mm`, `Tiefe_mm`)
- `Material`, `Artikelfakten`, `Artikelbeschreibung`
- Verpackungsabmessungen und Gesamtgewicht (`Packstueck_Gewicht_kg`)
- `Gewicht_kg` (Nettogewicht, nur wenn außerhalb der Verpackungsdaten explizit angegeben)

**Begründung:** Gemeinsame Felder profitieren vom vollständigen Dokumentkontext ohne die Ablenkung durch variantenspezifische Daten.

### Pass 2 — Variantenspezifische Felder
Extrahiert Felder, die je Farb-/Größenvariante unterschiedlich sind:
- `EAN`, `Farbe`, `Hersteller_Artikelnummer`

Gibt ein JSON-Array mit einem Objekt pro Variante zurück. Jede Variante wird zu einer eigenen Zeile im Zielformat.

**Begründung:** Die Isolation der Variantenextraktion verhindert, dass das Modell gemeinsame und variantenspezifische Daten vermischt — ein häufiges Fehlermuster bei kombinierter Abfrage.

### Pass 3 — Best-Effort-Restfelder
Extrahiert eine kuratierte Untermenge möbelrelevanter Zielformat-Felder, die in manchen Dokumenten vorhanden sein können:
- Montageinformationen (`Selbstaufbau möglich`, `Montagehinweis für Endkunden`)
- Kastendetails (`Anzahl Schubkästen`, `Anzahl Türen`, `Schubkastenführung`)
- Materialspezifika (`Holz`, `Holzwerkstoff`, `Glas`)
- Produktkategorisierung (`Warengruppe`, `Stil`, `Gestellfarbe`)

Gibt nur Felder mit tatsächlichen Werten zurück — leere Felder werden vollständig weggelassen.

**Begründung:** Dieser Pass ermöglicht die Erfassung zusätzlicher strukturierter Daten aus Dokumenten, die diese bereitstellen (z. B. detaillierte XLSX-Exporte aus Hersteller-ERP-Systemen), ohne die Performance bei einfachen PDF-Prospekten zu beeinträchtigen.

---

## Dateitypenerkennung

Der Workflow erkennt den Eingabedateityp anhand der binären Dateiendungseigenschaft:

```
fileExtension = 'pdf'  → PDF-Extraktionspfad
fileExtension = 'xlsx' → Excel-Extraktionspfad
fileExtension = andere → HTTP 415 Unsupported Media Type
```

**XLS (Legacy-Excel) wird nicht unterstützt.** Dateien sollten vor der Übermittlung in XLSX konvertiert werden. Dies ist in der Fehlermeldung dokumentiert.

---

## Zielformat-Spaltenmapping

Das Zielformat enthält 338 Spalten in mehreren logischen Abschnitten. Der Workflow mappt extrahierte Felder über zwei Strategien auf korrekte Spaltenpositionen:

**Eindeutige Spalten** — Mapping per Spaltenname-Suche (`indexOf`). Bei Zielformat-Updates passen sich diese Mappings automatisch an, solange der Spaltenname unverändert bleibt.

**Doppelte Spaltennamen** — Das Zielformat enthält 8 Spaltennamen, die in verschiedenen Abschnitten mehrfach vorkommen (z. B. `Breite` erscheint 3-mal: Verpackungsabschnitt, Artikelabschnitt, Leuchtmittelabschnitt). Diese werden über hartcodierte Positionsindizes gemappt, die im Workflow-Code dokumentiert sind.

| Doppelte Spalte | Verwendete Positionen |
|---|---|
| EAN | 0 (primär), 96 (Artikelabschnitt) |
| Breite | 59 (Verpackung cm), 94 (Artikel mm), 240 (Leuchtmittel mm) |
| Höhe | 110 (Artikel mm), 241 (Leuchtmittel mm) |
| Tiefe | 62 (Verpackung cm), 130 (Artikel mm), 242 (Leuchtmittel mm) |
| Farbe | 101 (primär), 181 (Farbabschnitt) |
| Material | 118 (primär), 187 (Materialabschnitt) |
| Durchmesser | 95 (Artikel), 243 (Leuchtmittel) |
| justierbar | 186 (Füße), 211 (Höhenverstellung) |

---

## Bekannte Einschränkungen & Designentscheidungen

### Felder abhängig vom Dokumentinhalt

Felder werden nur befüllt, wenn das Quelldokument die entsprechenden Daten explizit enthält. Leere Felder bedeuten fehlende Eingabedaten, keine Workflow-Einschränkung.

**Herstellerland** — Nur extrahierbar, wenn im Dokument als Text angegeben. PDFs, die das Herstellerland als Bild darstellen (z. B. „MADE IN GERMANY"-Logo), können per Textextraktion nicht verarbeitet werden. XLSX-Eingaben mit einer dedizierten Herstellerland-Spalte befüllen dieses Feld korrekt.

**Gewicht (Nettogewicht)** — Nettogewicht wird nur extrahiert, wenn es außerhalb des Verpackungskontexts explizit angegeben ist. PDF-Prospekte listen typischerweise nur Verpackungsgewichte auf. XLSX-Eingaben mit einer dedizierten Nettogewicht-Spalte befüllen dieses Feld korrekt.

**Variantenbehandlung** — Das Zielformat verwendet `Farbe` als primären Variantendifferenziator. Nicht-Farbvarianten (z. B. Größenvarianten) werden in das `Farbe`-Feld gemappt, da das Zielformat kein generisches Variantentypfeld vorsieht.

Dieses Verhalten entspricht der Aufgabenstellung: *„Felder die nicht gefüllt werden können bleiben leer."*

### Generische Dokumentstrukturverarbeitung

Das „generisch"-Kriterium bezieht sich auf die Adaptionsfähigkeit an unterschiedliche Dokumentstrukturen — der Workflow verarbeitet Dokumente beliebiger Möbelhersteller unabhängig von Layout, Sprache oder Feldreihenfolge durch KI-basierte semantische Extraktion statt Mustererkennung. Der XLSX Content Builder verwendet eine schlüsselwortbasierte Fallback-Feldauflösung, um unterschiedliche Spaltennamen verschiedener Hersteller-Excel-Exporte zu verarbeiten — ohne Standardisierung der Spaltennamen seitens des Lieferanten.

### Pass-3-Nicht-Determinismus
Pass-3-Ergebnisse können zwischen Ausführungen variieren, bedingt durch die probabilistische Natur der LLM-Inferenz. In einer Ausführung extrahierte Felder erscheinen möglicherweise nicht in einer anderen. Dies ist erwartetes Verhalten für Best-Effort-Extraktion und beeinträchtigt nicht die deterministischen Ergebnisse von Pass 1 und Pass 2.

### Doppelte Spaltennamen im Zielformat
Das Zielformat enthält 8 Spaltennamen, die in verschiedenen Produktkategorie-Abschnitten mehrfach vorkommen. Aufgrund der Sandbox-Einschränkungen des n8n Code-Knotens kann die `xlsx`-JavaScript-Bibliothek nicht direkt geladen werden, was positionsbasiertes Array-Schreiben verhindert. Daten werden daher nur in das **erste Vorkommen** jedes doppelten Spaltennamens geschrieben. Sekundäre Vorkommen bleiben leer.

Betroffene Spalten: `EAN`, `Breite`, `Höhe`, `Tiefe`, `Farbe`, `Material`, `Durchmesser`, `justierbar`.

Dies ist eine Plattformeinschränkung, keine Workflow-Design-Limitation. In einer Produktionsumgebung mit uneingeschränktem Node.js-Zugriff kann der Workflow durch den `xlsx`-Array-of-Arrays-Ansatz (`aoa_to_sheet`) auf alle Spaltenpositionen erweitert werden.

### Pass-3-Feldfehlerkennung
Pass 3 (Best-Effort-Extraktion) verwendet einen kompakten, schnellen Prompt für möbelspezifische Felder. Bei weniger verbreiteten Feldern mit mehrdeutigem Dokumentkontext (z. B. `Holz`, `Gestellmaterial`) kann das Modell Werte falsch zuordnen — beispielsweise einen Farbnamen in ein Materialfeld. Diese Fehlzuordnungen sind auf Pass 3 beschränkt und beeinflussen die Kernextraktion von Pass 1 und Pass 2 nicht. Felder mit offensichtlich falschen Werten sollten im Rahmen normaler Datenqualitätsprozesse im ERP-System geprüft und korrigiert werden.

### Mehrvarianten-Dokumente
Dokumente mit mehreren Produktvarianten (erkannt anhand mehrerer EAN-/Farbkombinationen) erzeugen eine Ausgabezeile pro Variante. Gemeinsame Produktdaten (Abmessungen, Merkmale, Hersteller) werden in alle Variantenzeilen dupliziert — konsistent mit der Standard-ERP-Dateneingabepraxis.

### Doppelter Inhalt in PDFs
Zweiseitige PDF-Prospekte, bei denen beide Seiten identische Produktinformationen enthalten, werden vor der KI-Verarbeitung durch einen Content-Hash-Algorithmus dedupliziert. Dies verhindert redundanten Token-Verbrauch und Extraktionsfehler.

---

## Produktionsempfehlungen

### GPU-Beschleunigung
Die aktuelle Konfiguration führt DeepSeek R1 7B auf CPU aus, was zu Inferenzzeiten von 1–3 Minuten pro Dokument führt. Für den Produktionsbetrieb wird GPU-Beschleunigung empfohlen:

- **Minimum:** NVIDIA GPU mit 6 GB+ VRAM (z. B. RTX 3060)
- **Empfohlen:** NVIDIA GPU mit 12 GB+ VRAM für komfortablen 7B-Modell-Betrieb
- Aktivierung über `deploy.resources.reservations.devices` in docker-compose.yml

Für europäische Kunden ohne geeignete On-Premises-Hardware bieten **Hetzner Cloud GPU-Instanzen** (CCX-Serie) DSGVO-konforme Rechenkapazität in deutschen Rechenzentren ab ca. 1–2 €/Stunde.

### Modell-Upgrades
DeepSeek R1 7B ist das Standardmodell, gewählt für zuverlässigen Betrieb auf CPU-basierter Infrastruktur. Kunden mit GPU-Servern können auf DeepSeek R1 14B upgraden — insbesondere für komplexe Feldunterscheidung und strukturierte XLSX-Eingaben — ohne Workflow-Änderungen:

```bash
docker exec ollama ollama pull deepseek-r1:14b
```

| Modell | VRAM-Bedarf | CPU-Fallback | Empfohlen für |
|---|---|---|---|
| deepseek-r1:7b | ~4,5 GB | ✅ Geeignet | CPU-only oder Low-VRAM-Server |
| deepseek-r1:14b | ~8,5 GB | ⚠️ Langsam | GPU-Server mit 12 GB+ VRAM |
| deepseek-r1:32b | ~19 GB | ❌ | Dedizierte GPU-Infrastruktur |

### Gescannte PDF-Unterstützung
Für Kunden, die gescannte PDF-Dokumente (Nicht-Text-PDFs) übermitteln, kann ein OCR-Vorverarbeitungsschritt zwischen dem PDF-Extraktionsknoten und dem Normalize-Data-Knoten mit Tesseract OCR ergänzt werden. Tesseract ist Open Source und selbst hostbar — die DSGVO-Konformität bleibt gewahrt.

---

## Kunden-Konfiguration

Mehrere Parameter, die derzeit im Workflow-Code eingebettet sind, eignen sich für die Auslagerung in Konfigurationsdateien, um kundenspezifische Anpassungen ohne Workflow-Änderungen zu ermöglichen:

| Parameter | Aktueller Speicherort | Auslagerungspfad |
|---|---|---|
| Zielformat-Duplikatspaltenpositionen | XLSX Content Builder Code-Knoten | `zielformat-config.json` |
| Pass-3-Feldauswahl | Extract Remaining Headers Code-Knoten | `zielformat-config.json` |
| KI-Modellname | Build Request Code-Knoten | `docker-compose.yml` Umgebungsvariable |
| Ollama-Endpunkt-URL | DeepSeek HTTP Request-Knoten | `docker-compose.yml` Umgebungsvariable |

**Pass-3-Feldauswahl** ist besonders für die Kundenkonfiguration relevant. Die aktuelle kuratierte Liste zielt auf Büromöbel und Kastenmöbel ab. Kunden mit Schwerpunkt auf Polstermöbeln, Leuchten oder Textilien profitieren von einer angepassten Pass-3-Feldliste.

**Zielformat-Updates** — Bei Änderungen der Zielformat-Spaltenstruktur muss `extract_headers.py` erneut ausgeführt werden, um `zielformat-headers.json` zu aktualisieren. Duplikatspaltenpositionen müssen ebenfalls manuell geprüft und bei Bedarf angepasst werden.

---

## Sicherheitshinweise

- Für den Webhook-Endpunkt ist in der aktuellen Konfiguration keine Authentifizierung eingerichtet. Für den Produktionsbetrieb sollte die integrierte n8n-Authentifizierung aktiviert oder der Webhook hinter einem API-Gateway platziert werden.
- Die n8n-Administrationsoberfläche ist durch Basic Authentication geschützt (konfiguriert in docker-compose.yml).
- Alle KI-Inferenzen werden lokal durchgeführt — kein Dokumentinhalt verlässt den Server.
