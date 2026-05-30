## Kontext
Dieses Projekt entstand im Rahmen eines Take-Home-Assessments. Der n8n-Workflow 
zentralisiert Kundenkontakte aus drei Kanälen (Webhook, E-Mail/IMAP und Audio) 
und verarbeitet sie mit einem vollständig lokal betriebenen KI-Stack: Pyannote 
übernimmt die Sprecher-Diarisierung und Transkription, Mistral 7B via Ollama 
analysiert die Stimmung und erkennt kritische Fälle in Echtzeit. Alle Dienste 
laufen selbst gehostet via Docker Compose – ohne externe API-Aufrufe.

# Möbelhaus Artikeldaten-Mapper

Ein n8n-Workflow, der Artikelstammdaten aus Möbelhersteller-Dokumenten (PDF-Prospekte oder Excel-Dateien) automatisch extrahiert und in ein standardisiertes ERP-Zielformat mappt — unter Verwendung eines lokal gehosteten KI-Modells.

---

## Voraussetzungen

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/macOS) oder Docker Engine (Linux)
- [Python 3.x](https://www.python.org/downloads/) mit `openpyxl` (`pip install openpyxl`)
- [Postman](https://www.postman.com/downloads/) (optional, zum Testen)
- NVIDIA GPU mit CUDA-Unterstützung (optional, für Produktionsbetrieb empfohlen)

---

## Projektstruktur

```
moebelhaus-mapper/
├── docker-compose.yml               # Infrastrukturdefinition
├── extract_headers.py               # Einmaliges Setup-Skript
├── Ziel-Tabelle.xlsx                # Zielformat-Vorlage
├── Beispielartikel.pdf              # PDF-Testdatei
├── Beispielartikel_XLSX.xlsx        # Excel-Testdatei
├── files/
│   └── zielformat-headers.json     # Generiert von extract_headers.py
├── workflow/
│   └── moebelhaus-mapper.json      # n8n Workflow-Export
└── postman/
    └── moebelhaus-mapper.json      # Postman-Collection
```

---

## Einrichtung

### 1. Projekt entpacken

Alle Dateien in ein Verzeichnis legen, z. B. `Y:/Sandbox/moebelhaus-mapper/`.

### 2. Zielformat-Header generieren

Einmalig vor dem ersten Start ausführen:

```bash
python3 extract_headers.py
```

Dieses Skript liest `Ziel-Tabelle.xlsx` und erzeugt `files/zielformat-headers.json`, das der Workflow verwendet, um die Ausgabe-Excel-Datei mit allen 338 Spalten in der korrekten Reihenfolge zu erstellen.

> **Hinweis:** Bei Änderungen an `Ziel-Tabelle.xlsx` muss dieses Skript erneut ausgeführt werden.

### 3. Dienste starten

```bash
docker compose up -d
```

Folgende Dienste werden gestartet:
- **n8n** unter `http://localhost:5678`
- **Ollama** unter `http://localhost:11434`

Der erste Start kann einige Minuten dauern, da Docker die Images herunterlädt.

### 4. DeepSeek-Modell laden

```bash
docker exec ollama ollama pull deepseek-r1:7b
```

Lädt ca. 4,7 GB herunter. Einmalig ausführen — das Modell wird im `ollama_data` Docker-Volume gespeichert.

### 5. Workflow importieren

1. n8n unter `http://localhost:5678` öffnen
2. Mit `admin` / `admin123` anmelden (für Produktion in docker-compose.yml ändern)
3. **Workflows** → **Aus Datei importieren**
4. `workflow/moebelhaus-mapper.json` auswählen
5. Workflow öffnen und **Aktivieren** klicken

---

## GPU-Beschleunigung (optional, empfohlen)

Standardmäßig läuft Ollama auf CPU. Für deutlich schnellere Inferenz (~30 Sekunden statt ~3 Minuten pro Dokument) GPU-Unterstützung aktivieren:

### NVIDIA

Voraussetzung: [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) installiert. Der `deploy`-Abschnitt in `docker-compose.yml` ist bereits für GPU-Durchleitung konfiguriert.

Nach Neustart prüfen:

```bash
docker logs ollama | grep "inference compute"
```

Erwartete Ausgabe: `inference compute id=gpu library=cuda`

### Auf größeres Modell upgraden

Für bessere Extraktionsqualität auf GPU-Servern:

```bash
docker exec ollama ollama pull deepseek-r1:14b
```

Anschließend den Modellnamen in den drei **Build Request** Code-Knoten im Workflow aktualisieren.

---

## Testen

### Mit Postman

1. `postman/moebelhaus-mapper.json` in Postman importieren
2. Datei im `data`-Formularfeld auswählen (PDF oder XLSX)
3. Anfrage senden — die Antwort ist eine herunterladbare `.xlsx`-Datei

### Mit PowerShell 7+

```powershell
# PDF-Test
Invoke-WebRequest -Uri "http://localhost:5678/webhook-test/moebelhaus-mapper" `
  -Method POST `
  -Form @{ data = Get-Item ".\Beispielartikel.pdf" } `
  -OutFile ".\output.xlsx"

# Excel-Test
Invoke-WebRequest -Uri "http://localhost:5678/webhook-test/moebelhaus-mapper" `
  -Method POST `
  -Form @{ data = Get-Item ".\Beispielartikel_XLSX.xlsx" } `
  -OutFile ".\output.xlsx"
```

> **Hinweis:** Bei aktiviertem Workflow die Produktions-Webhook-URL (`/webhook/`) statt `/webhook-test/` verwenden.

### Mit curl (Linux/macOS/Git Bash)

```bash
# PDF-Test
curl -X POST http://localhost:5678/webhook-test/moebelhaus-mapper \
  -F "data=@Beispielartikel.pdf" \
  -o output.xlsx

# Excel-Test
curl -X POST http://localhost:5678/webhook-test/moebelhaus-mapper \
  -F "data=@Beispielartikel_XLSX.xlsx" \
  -o output.xlsx
```

---

## Erwartete Ausgabe

Der Workflow gibt eine befüllte `zielformat-output.xlsx` zurück mit:

- Allen 338 Spalten aus dem Zielformat in korrekter Reihenfolge
- Einer Zeile pro Produktvariante (z. B. zwei Zeilen für ein Produkt mit Sonoma-Eiche- und Naturbuche-Variante)
- Befüllten Feldern, wo Daten im Quelldokument verfügbar sind
- Leeren Feldern, wo keine Daten vorhanden sind (gemäß Aufgabenstellung: *„Felder die nicht gefüllt werden können bleiben leer"*)

### Aus PDF-Prospekten extrahierte Felder

| Feld | Hinweis |
|---|---|
| EAN | Pro Variante |
| ArtBez | Produktbezeichnung |
| Hersteller | Vollständiger Firmenname |
| Hersteller-Artikelnummer | Pro Variante |
| Breite / Höhe / Tiefe | Aus WxHxT-mm-Zeile |
| Farbe | Pro Variante |
| Material | Aus Merkmalsliste |
| Artikelfakten | Alle Aufzählungspunkte zusammengefasst |
| Packstücke Gesamt-Gewicht | Summe aller Verpackungsgewichte |
| Verpackungsabmessungen | Aus Verpackungsdaten-Abschnitt |

### Zusätzliche Felder aus Excel-Eingaben

| Feld | Hinweis |
|---|---|
| Gewicht | Nettogewicht (wenn Spalte vorhanden) |
| Herstellerland | Wenn Spalte vorhanden |
| Selbstaufbau möglich | Wenn Spalte vorhanden |
| Warengruppe | Wenn Spalte vorhanden |
| + weitere Pass-3-Felder | Wenn Spalten vorhanden |

---

## Unterstützte Dateiformate

| Format | Unterstützung |
|---|---|
| `.pdf` | ✅ Textbasierte PDFs |
| `.xlsx` | ✅ Excel 2007+ |
| `.xls` | ❌ Bitte vorher in XLSX konvertieren |
| Gescannte PDFs | ⚠️ OCR-Vorverarbeitung erforderlich (siehe Technische Dokumentation) |

Nicht unterstützte Dateitypen geben HTTP 415 mit einer beschreibenden Fehlermeldung zurück.

---

## Dienste beenden

```bash
docker compose down
```

Daten werden in Docker-Volumes (`n8n_data`, `ollama_data`) gespeichert und überleben Container-Neustarts.

Alle Daten entfernen:

```bash
docker compose down -v
```

---

## Fehlerbehebung

**Timeout-Fehler bei der KI-Extraktion**
- Der CPU-Betrieb ist langsam (~3 Min./Dokument). GPU-Beschleunigung aktivieren (siehe oben) oder das Timeout im DeepSeek HTTP-Request-Knoten erhöhen (Einstellungen → Timeout).

**„Zugriff auf Datei nicht erlaubt"-Fehler**
- Sicherstellen, dass `./files` als `./files:/home/node/.n8n-files` in `docker-compose.yml` gemountet ist und n8n neu starten.

**„Kein Modul namens openpyxl"**
- `pip install openpyxl` ausführen, bevor `extract_headers.py` gestartet wird.

**Ollama nicht von n8n erreichbar**
- Prüfen, ob beide Container im selben Docker-Netzwerk sind: `docker inspect n8n --format '{{json .NetworkSettings.Networks}}'`
- Der Netzwerkname sollte `moebelhaus` sein.

**Ausgabe enthält falsche Daten in einigen Feldern**
- Das KI-Modell kann bei ungewöhnlichen Dokumentlayouts Felder falsch identifizieren. Die Ausgabe des Parse-Shared/Variant-Response-Knotens im n8n-Ausführungsprotokoll zeigt die rohe Extraktion. Prompt-Anpassungen können in den Build-Request-Code-Knoten vorgenommen werden.
