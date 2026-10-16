---
name: "datev-buchungsimport"
description: "Erstellt Buchungen in DATEV über Klardaten-API oder als ASCII-Datei. Fragt Buchungsquelle, Sonderfelder und Geschäftsjahr ab. Erstellt automatisch ASCII-Datei wenn Gesellschafternummer benötigt wird."
---

# DATEV Buchungsimport

Dieser Skill erstellt Buchungen in DATEV Rechnungswesen — entweder über die Klardaten-API (DATEV Connect) oder als ASCII-Datei für den manuellen Import.

## Schritt 0: Klardaten-Konnektor ermitteln (vor allem anderen)

In der Kanzlei gibt es mehrere Klardaten-Konnektoren mit unterschiedlichen Rechten (z.B. `KlardatenAlleRechte`, `connect-tutoren`, `connect-ma`). Die Werkzeugnamen in diesem Skill sind deshalb ohne Präfix geschrieben (z.B. `datev_accounting_list`). Der vollständige Name lautet `mcp__<Konnektorname>__<Werkzeug>`; Sonderzeichen im Konnektornamen können dabei umgeschrieben sein.

1. Per Werkzeugsuche (ToolSearch) nach `datev_accounting_list` suchen und alle Treffer notieren.
2. **Genau ein Konnektor gefunden:** diesen verwenden.
3. **Mehrere gefunden:** den Nutzer fragen, welcher verwendet werden soll (die Rechte unterscheiden sich).
4. **Keiner gefunden:** Nutzer informieren, dass kein Klardaten-Konnektor verbunden ist, und den ASCII-Export anbieten.

Alle Werkzeuge eines Laufs immer über denselben Konnektor aufrufen.

## Ablauf

### Schritt 1: Buchungsquelle erfragen

Frage den Nutzer:

**Woher kommen die Buchungsdaten?**
1. **DMS** — Belege aus dem DATEV Dokumentenmanagement
2. **Unternehmen Online** — Belege aus DATEV Unternehmen Online (Bank/Kasse/Belege)
3. **Ordner** — Dateien aus einem lokalen Ordner (Pfad erfragen)
4. **Manuell** — Nutzer gibt Buchungsdaten direkt ein

Bei **DMS**:
- Klardaten-Verbindung prüfen (Tool `datev_describe` mit topic='overview')
- Falls Verbindung fehlschlägt: Nutzer informieren, dass Klardaten-MCP nicht verbunden ist
- **Mandantennummer erfragen** (z.B. 10000)
- Client-ID über `datev_accounting_list` mit `resourceUri: datev://accounting/clients` und Filter `number eq <mandantennummer>` auflösen
- Fiscal Year über `datev_accounting_list` mit `resourceUri: datev://accounting/fiscal_years` und clientId holen

Bei **Unternehmen Online**:
- Klardaten-Verbindung prüfen
- **Mandantennummer erfragen**
- **Export-Datei mit Beleglinks anfordern:**
  > "Bitte stelle eine Export-Datei aus Unternehmen Online bereit, die die Beleg-IDs enthält (Spalte 'Beleg-ID' oder ähnlich). Du kannst diese in Unternehmen Online unter Belege → Export erstellen."
- Die Beleg-IDs (GUIDs) aus der Export-Datei werden als BEDI-Links verwendet

### Schritt 1b: Belegverknüpfung zu Unternehmen Online (PFLICHT-Abfrage)

**Frage explizit:**

> **Sollen die Buchungen mit Belegen aus DATEV Unternehmen Online verknüpft werden?**
> - **Ja** — Buchungen erhalten einen BEDI-Beleglink (Beleg-ID aus Unternehmen Online)
> - **Nein** — Buchungen ohne Belegverknüpfung

**Wenn JA:**
1. Fordere eine Export-Datei mit den Beleg-IDs an (falls nicht bereits im Ordner vorhanden)
2. Verwende `documentSystem: "BEDI"` (NICHT "DDMS"!)
3. Das Format für den Beleglink ist die reine UUID ohne Präfix: `documentLink: "00000000-0000-0000-0000-000000000000"`

**WICHTIG — BEDI vs. DDMS:**
| Belegquelle | documentSystem | Verwendung |
|-------------|----------------|------------|
| DATEV Unternehmen Online | `BEDI` | Belege aus Bank/Kasse/Belege-Modul |
| DATEV DMS | `DDMS` | Belege aus dem Dokumentenmanagement |

### Schritt 2: Feldprüfung

Frage den Nutzer:

**Benötigst du ein Buchungsfeld, das in der folgenden Liste NICHT enthalten ist?**

Unterstützte Felder (API + ASCII):
- Umsatz / Betrag
- Konto (Soll)
- Gegenkonto (Haben)
- BU-Schlüssel
- Belegdatum
- Belegfeld 1
- Belegfeld 2
- Buchungstext
- Beleglink (DMS- oder Unternehmen-Online-Verknüpfung)
- Kostenstelle 1 (kost1CostCenterId)
- Kostenstelle 2 (kost2CostCenterId)

Falls JA → Nutzer das benötigte Feld beschreiben lassen.

### Schritt 3: API vs. ASCII entscheiden

**Felder, die NUR per ASCII-Import übertragen werden können (nicht über API):**
- Beteiligtennummer / Gesellschafternummer
- Gesellschaftername

**Wenn eines dieser Felder benötigt wird:**
1. Informiere den Nutzer: "Das Feld [Feldname] wird von der DATEV-API nicht unterstützt. Ich erstelle stattdessen eine ASCII-Datei für den manuellen Import in DATEV."
2. Erstelle eine ASCII-Datei im DATEV-Format (siehe ASCII-Format unten)

**Wenn alle Felder über die API gehen:**
1. Erstelle die Buchungssequenz über `datev_accounting_create_sequence`
2. Informiere den Nutzer, dass die Buchungen als "nicht festgeschrieben" in DATEV liegen

### Schritt 4: Buchungsdaten vorbereiten

#### 4.1 Geschäftsjahr erfragen

Frage den Nutzer:

**In welches Geschäftsjahr sollen die Buchungen?**
- Zeige die verfügbaren Geschäftsjahre aus `datev_accounting_list` (resourceUri: datev://accounting/fiscal_years)
- Nutzer wählt das Ziel-Geschäftsjahr (z.B. 2026)

#### 4.2 Historische Buchungen analysieren (PFLICHT — IMMER!)

**WICHTIG: Vor dem Buchen IMMER die Vergangenheit prüfen — nicht den Nutzer fragen, sondern automatisch tun!**

##### Schritt A: Bestehende Buchungen abrufen

Für jeden Kreditor/Lieferanten im Buchungslauf:

1. **Bestehende Buchungen abrufen** (letzte 12 Monate):
   ```
   datev_accounting_list
     resourceUri: datev://accounting/account_postings
     clientId: <GUID>
     fiscalYearId: <aktuelles + Vorjahr>
     filter: contraAccountNumber eq <Kreditorenkonto>
   ```

2. **Aus der Vergangenheit ableiten:**
   - Welches **Sachkonto** wurde verwendet?
   - Welches **Gegenkonto** (Kreditor, Bank 1200, Privateinlage 1890)?
   - Wurde ein **BU-Schlüssel** verwendet oder **OHNE** gebucht?
   - Bei ähnlichen Beträgen/Positionen: Wurden diese **aktiviert** (04xx) oder als **Aufwand** gebucht?

3. **Kontierung übernehmen** — nicht raten, sondern die etablierte Buchungslogik fortführen.

##### Schritt B: Kein Vorsteuerabzug erkennen

**WICHTIG: Wenn historische Buchungen OHNE BU-Schlüssel erfolgten, bedeutet das oft: Kein Vorsteuerabzug!**

Typische Fälle ohne Vorsteuerabzug:
- **Holdinggesellschaften** ohne operative Tätigkeit
- **Vermögensverwaltende Gesellschaften**
- **Steuerfreie Unternehmer** (z.B. Ärzte, Kleinunternehmer)

**Vorgehensweise:**
1. Prüfe ob vergangene Eingangsrechnungen mit BU-Schlüssel (9, 8, 94 etc.) gebucht wurden
2. **Vorher ausschließen: Automatikkonto.** Auf DATEV-Automatikkonten (AV/AM) rechnet DATEV die Vorsteuer auch OHNE BU-Schlüssel. Fehlende BU-Schlüssel beweisen dann keinen fehlenden Vorsteuerabzug. Prüfe, ob zu den historischen Buchungen Vorsteuer ausgewiesen wurde oder das Konto eine Steuerautomatik hat. Im Zweifel den Nutzer fragen.
3. Falls NEIN (alle Buchungen ohne accountingTransactionKey, kein Automatikkonto, keine Vorsteuer): Fortfahren OHNE BU-Schlüssel
4. Buche Bruttobeträge direkt auf das Aufwandskonto

**Beispiel (Gesellschaft ohne Vorsteuerabzug):**
```json
{
  "accountNumber": 4950,
  "contraAccountNumber": 1200,
  "amount": 5950.00,
  "debitCreditIdentifier": "S",
  "date": "2026-01-30T00:00:00+01:00",
  "postingDescription": "Muster JA-Pruefung 2024"
  // KEIN accountingTransactionKey!
}
```

##### Schritt C: Kreditorenstammdaten auslesen

**Falls kein historischer Match gefunden wird:**

```
datev_accounting_list
  resourceUri: datev://accounting/creditors
  clientId: <GUID>
  fiscalYearId: <YYYYMMDD>
```

Daraus:
- **Prüfen ob Kreditor bereits existiert** — nach Name/Caption suchen
- **Sammelkreditoren identifizieren** (typisches Muster: X-Diverse nach Anfangsbuchstabe)
  - A-Diverse, B-Diverse, ... T-Diverse, etc.
  - Nur verwenden wenn KEIN spezifischer Kreditor existiert!
- **Für jeden Lieferanten auf der Rechnung:**
  1. Suche exakten Match im Kreditorenstamm (z.B. "Muster Telekom" → 70100)
  2. Falls gefunden: Dieses Konto verwenden!
  3. Falls nicht gefunden: Sammelkreditor nach Anfangsbuchstabe (z.B. "Muster Lieferdienst" → M-Diverse)

##### Schritt D: Wenn kein Kreditor gefunden — Sachkonten in der Historie suchen

**WICHTIG: Nicht alle Gegenkonten sind Kreditoren! Bei Reisekosten, Auslagen, Privateinlagen etc. werden oft Sachkonten verwendet.**

1. **Historische Buchungen nach Buchungstext durchsuchen**
2. **Typische Sachkonten-Bereiche als Gegenkonto:**
   
   | Bereich | Verwendung |
   |---------|------------|
   | 99xxx | Personenkonten (Mitarbeiter-Verrechnungskonten) |
   | 1890 | Privateinlage Gesellschafter |
   | 1800 | Privatentnahme |
   | 1360-1369 | Geldtransit |
   | 1200 | Bank (Direktzahlung) |
   | 1610 | Verbindlichkeiten a.L.u.L. |

**Erst wenn weder Kreditor noch historisches Sachkonto gefunden wird:** Nutzer fragen oder Sammelkreditor vorschlagen.

**Privatkonten 1800–1899 (Erfahrung aus der Kanzlei):**
- Privatkonten sind wie Bank- und Personenkonten die steuerneutrale Seite. Den BU-Schlüssel immer so setzen, dass das **Aufwands-/Ertragskonto das Gegenkonto** ist. Sonst berechnet DATEV die Steuer am Privatkonto.
- Bei Mandanten mit Beteiligten (z.B. Personengesellschaften) brauchen Buchungen auf 1800/1810/1890 die **Beteiligtennummer** → ASCII-Import (siehe Schritt 3). Die Nummer beim Nutzer erfragen, nicht raten.

#### 4.3 Abweichendes Leistungsdatum (WICHTIG!)

**Die API unterstützt separate Felder für Rechnungsdatum und Leistungsdatum!**

| API-Feld | Bedeutung |
|----------|-----------|
| `date` | **Rechnungsdatum / Belegdatum** |
| `deliveryDate` | **Leistungsdatum / Lieferdatum** |

Wenn das Leistungsdatum vom Rechnungsdatum abweicht:

**Vorgehensweise:**
1. `date` auf das **Rechnungsdatum** setzen
2. `deliveryDate` auf das **Leistungsdatum** setzen
3. Den Buchungszeitraum (`dateFrom`/`dateTo`) auf den Monat des **Leistungsdatums** setzen
4. Die Rechnungsnummer in `documentField1` und den Beleglink normal erfassen

**Beispiel:**
- Rechnungsdatum: 15.03.2026
- Leistungsdatum: 28.02.2026
- → `date: "2026-03-15T00:00:00+01:00"` (Rechnungsdatum)
- → `deliveryDate: "2026-02-28T00:00:00+01:00"` (Leistungsdatum)
- → `dateFrom: "2026-02-01"`, `dateTo: "2026-02-28"` (Monat des Leistungsdatums!)

**Wichtige Einschränkung (API strenger als DATEV-Oberfläche):**
- Die API validiert, dass `date` innerhalb des Geschäftsjahres liegt
- DATEV selbst erlaubt ein Belegdatum außerhalb des WJ, solange das Leistungsdatum innerhalb liegt
- **Workaround bei Rechnungsdatum außerhalb WJ:**
  1. Über API mit `date` = Leistungsdatum buchen
  2. In DATEV manuell das Belegdatum auf das echte Rechnungsdatum korrigieren (22.04. statt 31.03.)
  3. DATEV akzeptiert die Korrektur, da das Leistungsdatum im WJ liegt

**Ergebnis:**
- Die Buchung landet in der Periode des Leistungsdatums
- Nach manueller Korrektur zeigt DATEV beide Daten korrekt an

#### 4.4 Buchungen erstellen

##### API-Import (Standard)

**WICHTIG: Die API akzeptiert nur Buchungen innerhalb EINES Kalendermonats!**

> **Fehlermeldung:** "dateFrom and dateTo must be in the same calendar month"

**Konsequenz:** Für jeden Monat muss ein separater Buchungsstapel erstellt werden.

Verwende `datev_accounting_create_sequence` mit:
- `clientId`: GUID des Mandanten
- `fiscalYearId`: Geschäftsjahr im Format YYYYMMDD (z.B. "20260101" oder "20250401" bei abweichendem WJ)
- `accountLength`: Kontenlänge (meist 4)
- `dateFrom` / `dateTo`: Buchungszeitraum **innerhalb eines Kalendermonats**
- `description`: Beschreibung der Sequenz (max 30 Zeichen)
- `initials`: Kürzel (max 2 Zeichen)
- `accountingRecords`: Array der Buchungssätze

**Unterstützte Felder pro Buchungssatz:**
- `accountNumber`: Soll-Konto (Pflicht)
- `contraAccountNumber`: Haben-Konto (Pflicht)
- `amount`: Betrag (Pflicht, immer positiv)
- `debitCreditIdentifier`: "S" für Soll, "H" für Haben (Pflicht)
- `date`: **Rechnungsdatum/Belegdatum** im ISO-Format (Pflicht)
- `deliveryDate`: **Leistungsdatum** im ISO-Format (optional, bei abweichendem Leistungszeitraum)
- `postingDescription`: Buchungstext (max 60 Zeichen)
- `documentField1`: Belegnummer 1 (max 36 Zeichen)
- `documentField2`: Belegnummer 2 (max 12 Zeichen)
- `accountingTransactionKey`: BU-Schlüssel (z.B. 9 für VSt 19%, 40 für ohne USt) — **weglassen wenn kein Vorsteuerabzug!**
- `casesRelatedToGoodsAndServices`: Pflicht bei BU 91–99 (siehe Fehler #4)
- `documentLink`: **Nur die UUID** ohne Anführungszeichen
- `documentSystem`: **Pflicht bei documentLink** — `BEDI` für Unternehmen Online, `DDMS` für DMS
- `kost1CostCenterId`: Kostenstelle 1
- `kost2CostCenterId`: Kostenstelle 2

##### ASCII-Import (bei Sonderfeldern)

Erstelle eine CSV-Datei mit Semikolon als Trennzeichen.

**Kodierung: immer cp1252 (Windows-ANSI), nicht UTF-8!** Der DATEV-ASCII-Import mit der Formatvorlage „Kanzlei“ liest die Datei als ANSI; in UTF-8 werden Umlaute zu „Ã¤“. In Python: `open(pfad, "w", encoding="cp1252", newline="")`. Beträge ohne Tausenderpunkt (z.B. `1234,56`).

**Header-Zeile:**
```
Umsatz (mit Soll/Haben-Kz);Konto;Gegenkonto (ohne BU-Schlüssel);BU-Schlüssel;Belegdatum;Belegfeld 1;Belegfeld 2;Buchungstext;Beteiligtennummer;Gesellschafter;Beleglink;Festschreibung;Identifikationsnummer;Zeichnernummer;Kost 1 - Kostenstelle;Kost 2 - Kostenstelle
```

**Datenzeilen:**
- Umsatz: Betrag mit Komma als Dezimaltrennzeichen, negativ für Haben
- Konto: Soll-Konto
- Gegenkonto: Haben-Konto
- BU-Schlüssel: z.B. 9 (VSt 19%), 94 (Reverse Charge), leer (ohne)
- Belegdatum: DD.MM.YYYY
- Belegfeld 1: Belegnummer
- Belegfeld 2: optional
- Buchungstext: max 60 Zeichen
- Beteiligtennummer: Gesellschafternummer (z.B. 1, 2, 3...)
- Gesellschafter: Name des Gesellschafters
- Beleglink: 
  - DMS: `DDMS "GUID"` 
  - Unternehmen Online: `BEDI "GUID"`
- Festschreibung: 0
- Identifikationsnummer: optional (z.B. USt-IdNr.)
- Zeichnernummer: optional
- Kost 1 - Kostenstelle: Kostenstelle 1
- Kost 2 - Kostenstelle: Kostenstelle 2

**Beispiel mit BEDI-Link (Unternehmen Online):**
```
5950,00;4952;1200;;30.01.2026;2026/001;;Muster JA-Pruefung 2024;;;"BEDI ""00000000-0000-0000-0000-000000000000""";0;;;;
```

**Dateiname:** `Buchungsstapel_ASCII_MMYYYY_[Beschreibung].csv`

**Speicherort:** Im Projektordner oder vom Nutzer gewählten Ordner

### Schritt 5: Abschluss

**Bei API-Import:**
- Informiere über die Anzahl der erstellten Buchungssätze
- Hinweis: "Die Buchungen liegen als 'nicht festgeschrieben' in DATEV und müssen dort finalisiert werden."

**WICHTIGER HINWEIS am Ende:**

> **Beachte:** Die DATEV-API akzeptiert nur Buchungsstapel innerhalb eines Kalendermonats. Für Q1 2026 wurden daher 3 separate Stapel erstellt (Januar, Februar, März). Bei Jahresbuchhaltungen oder quartalsweiser Erfassung muss für jeden Monat ein eigener Stapel hochgeladen werden.

**Bei ASCII-Import:**
- Gib den Dateipfad an
- Anleitung: "Importiere die Datei in DATEV über: Buchen → Stapelverarbeitung → ASCII-Import und wähle dabei die Formatvorlage 'Kanzlei'"

## Belegverknüpfung: BEDI vs. DDMS

### DATEV Unternehmen Online (BEDI)

Belege aus dem Modul "Belege" in Unternehmen Online:

1. **Export-Datei aus Unternehmen Online anfordern:**
   - Die Datei enthält eine Spalte mit der Beleg-ID (GUID)
   - Typischer Spaltenname: "Beleg-ID" oder ähnlich

2. **Bei API-Import:**
   - `documentLink`: Nur die UUID (z.B. `00000000-0000-0000-0000-000000000000`)
   - `documentSystem`: `BEDI`

3. **Bei ASCII-Import:**
   - Beleglink-Spalte: `BEDI "GUID"` mit Präfix und escaped Anführungszeichen
   - Beispiel: `"BEDI ""00000000-0000-0000-0000-000000000000"""`

### DATEV DMS (DDMS)

Belege aus dem Dokumentenmanagement:

1. **Dokument-GUID ermitteln:**
   ```
   datev_dms_list mit resourceUri: datev://dms/documents und filter: number eq <belegnummer>
   ```

2. **Bei API-Import:**
   - `documentLink`: Nur die UUID
   - `documentSystem`: `DDMS`

3. **Bei ASCII-Import:**
   - Beleglink-Spalte: `DDMS "GUID"` mit Präfix und escaped Anführungszeichen

**ACHTUNG:** API und ASCII verwenden unterschiedliche Formate für den Beleglink!

| Import-Art | Format |
|------------|--------|
| API | `documentLink: "UUID"` + `documentSystem: "BEDI"` oder `"DDMS"` |
| ASCII | Spalte Beleglink: `"BEDI ""UUID"""` oder `"DDMS ""UUID"""` |

## Belegauslese: Manuell oder automatisch?

**Vor der Buchungserstellung zwingend abfragen:**

**Wie sollen die Belegdaten erfasst werden?**

1. **Manuelle Eingabe** — Der Nutzer liest die Daten aus dem Beleg selbst aus und gibt sie ein. Claude erstellt Buchungsvorschläge nur basierend auf:
   - DMS-Metadaten (Belegnummer, Datum, Beschreibung)
   - Vergangenen Buchungen (Konten, BU-Schlüssel, Buchungstexte als Vorlage)

2. **Automatische Auslese (OCR)** — Claude versucht, Beträge, Rechnungsnummern und weitere Daten direkt aus dem Beleg auszulesen.

### Bei automatischer Auslese (Option 2):

#### Schritt 1: Beleg lokal zwischenspeichern

DMS-Belege müssen zuerst heruntergeladen werden:

1. Document-File-ID ermitteln über `datev_dms_list`:
   - resourceUri: `datev://dms/structure_items`
   - id: Dokument-GUID
   - Ergebnis enthält `document_file_id` für jede Datei im Beleg

2. Download-Link abrufen über `datev_dms_get`:
   - resourceUri: `datev://dms/document_files`
   - documentFileId: die ermittelte ID
   - Ergebnis: temporärer Single-Use Download-Link

3. Datei herunterladen und lokal zwischenspeichern (z.B. im Temp-Ordner)

#### Schritt 1b: ALLE Anhänge eines Belegs prüfen (PFLICHT)

**WICHTIG: Ein DMS-Beleg kann mehrere Dateien enthalten — IMMER alle prüfen!**

1. **Alle document_files auflisten:**
   - Bei `datev://dms/structure_items` werden alle Dateien eines Belegs mit ihrer Extension gelistet
   - Typisch: PDF, MSG, XML, JPG, TIFF

2. **MSG-Dateien auf eingebettete Anhänge prüfen:**
   - MSG-Dateien (Outlook-E-Mails) können PDF-Rechnungen als Anhang enthalten
   - Im DMS erscheint das nur als "MSG(2)" o.ä. — die PDFs sind nicht direkt sichtbar!
   
   **Vorgehensweise bei MSG-Dateien:**
   ```python
   import extract_msg
   
   msg = extract_msg.Message("beleg.msg")
   for attachment in msg.attachments:
       if attachment.longFilename.lower().endswith('.pdf'):
           # PDF-Anhang gefunden — separat speichern und auslesen!
           attachment.save(customPath="temp/")
   ```

3. **Jede gefundene PDF/Rechnung separat verarbeiten:**
   - Ein Beleg kann MEHRERE Rechnungen enthalten (z.B. Sammelrechnung als MSG mit 2 PDF-Anhängen)
   - Anzahl gefundener Rechnungen mit Belegbeschreibung/Erwartung abgleichen
   - Bei Diskrepanz: Nutzer informieren ("Im Beleg X habe ich 2 Rechnungen gefunden statt 1")

**Typische Fälle:**
| DMS zeigt | Tatsächlicher Inhalt |
|-----------|---------------------|
| MSG(1) | E-Mail ohne Anhang |
| MSG(2) | E-Mail + 1 PDF-Anhang (Rechnung!) |
| MSG(3) | E-Mail + 2 PDF-Anhänge (2 Rechnungen!) |
| PDF, MSG(2) | Separates PDF + E-Mail mit weiterem PDF |

#### Schritt 2: Beleginhalt auslesen

**E-Rechnung (ZUGFeRD / XRechnung) — BEVORZUGT:**

E-Rechnungen enthalten strukturierte XML-Daten — diese sind zuverlässiger als OCR!

- **ZUGFeRD-PDF:** PDF mit eingebettetem XML (factur-x.xml oder zugferd-invoice.xml)
- **XRechnung:** Reines XML-Format

**Vorgehensweise bei E-Rechnungen:**
```
1. Prüfen ob E-Rechnung vorliegt:
   - Bei PDF: Eingebettete Dateien prüfen (Attachment-Namen wie "factur-x.xml", "zugferd-invoice.xml", "xrechnung.xml")
   - Dateiendung .xml → direkt XRechnung

2. XML extrahieren/lesen:
   - Bei ZUGFeRD-PDF: XML-Attachment aus PDF extrahieren (z.B. mit Python pdfplumber oder PyMuPDF)
   - Bei XRechnung: XML direkt lesen

3. Strukturierte Daten aus XML parsen:
   - Rechnungsnummer: //rsm:ExchangedDocument/ram:ID
   - Rechnungsdatum: //rsm:ExchangedDocument/ram:IssueDateTime
   - Nettobetrag: //ram:SpecifiedTradeSettlementHeaderMonetarySummation/ram:TaxBasisTotalAmount
   - USt-Betrag: //ram:SpecifiedTradeSettlementHeaderMonetarySummation/ram:TaxTotalAmount
   - Bruttobetrag: //ram:SpecifiedTradeSettlementHeaderMonetarySummation/ram:GrandTotalAmount
   - Lieferant: //ram:SellerTradeParty/ram:Name
   - USt-IdNr: //ram:SellerTradeParty/ram:SpecifiedTaxRegistration/ram:ID

4. Buchungsvorschlag aus XML-Daten erstellen
```

**Vorteil:** 100% zuverlässige Datenübernahme, keine OCR-Fehler möglich.

**PDF mit Textebene (keine E-Rechnung):**
- Verwende das `Read`-Tool direkt auf die PDF-Datei
- Extrahiere Beträge, Rechnungsnummern, Datum, Lieferant etc.

**PDF ohne Textebene (Scan) oder Bilddatei (JPG, PNG, TIFF):**
- Claude ist multimodal und kann Bilder direkt analysieren
- Verwende das `Read`-Tool auf die Bilddatei — Claude "sieht" den Inhalt visuell
- Alternativ bei komplexen mehrseitigen Scans: PDF-Seiten einzeln als Bilder extrahieren und analysieren

**Vorgehensweise bei Bildanalyse:**
```
1. Read-Tool auf die Belegdatei (PDF/Bild) anwenden
2. Claude analysiert das Bild visuell und extrahiert:
   - Rechnungsbetrag (brutto/netto/USt)
   - Rechnungsnummer
   - Rechnungsdatum
   - Lieferant/Kreditor
   - Ggf. Positionen
3. Ergebnisse dem Nutzer zur Bestätigung vorlegen
4. Nach Bestätigung: Buchungsvorschlag erstellen
```

#### Schritt 2b: Gemischte Steuersätze erkennen und aufteilen (PFLICHT)

**WICHTIG: Rechnungen können mehrere USt-Sätze enthalten — IMMER die USt-Aufstellung prüfen!**

Typische Fälle mit gemischten Steuersätzen:
- **Lebensmittellieferanten** (z.B. Lieferdienste, Supermärkte, Großhandel): Lebensmittel 7%, Verpackung/Tüten 19%
- **Gastronomiebedarf**: Speisen 7%, Getränke/Zubehör 19%
- **Bürobedarf mit Lebensmitteln**: Kaffee/Snacks 7%, Büromaterial 19%
- **Gutschriften/Credits**: Oft 0% (steuerfreie Verrechnung)

**Vorgehensweise:**

1. **USt-Aufstellung im Beleg suchen** — meist am Ende der Rechnung als Tabelle:
   ```
   USt %    Netto      USt      Brutto
   19       1,00 €     0,19 €   1,19 €
   7        80,00 €    5,60 €   85,60 €
   0        -20,00 €   0,00 €   -20,00 €
   ```

2. **Bei mehreren Steuersätzen: Buchung splitten!**
   - Jeder Steuersatz wird eine separate Buchungszeile
   - BU 9 für 19%-Anteil
   - BU 8 für 7%-Anteil
   - Ohne BU für 0%-Anteil (Gutschriften, steuerfreie Positionen)

3. **Beispiel Lieferdienst (66,79 € brutto):**
   ```
   Zeile 1: 1,19 €   / Konto 4650 / BU 9  (Tüten 19%)
   Zeile 2: 85,60 €  / Konto 4650 / BU 8  (Kaffee 7%)
   Zeile 3: -20,00 € / Konto 4650 / ohne BU (Credits)
   ```

**FEHLER vermeiden:** Eine einzelne Buchung mit dem Gesamtbetrag und einem pauschalen BU-Schlüssel führt zu falscher Vorsteuer!

#### Schritt 3: Nutzerbestätigung

**Wichtig:** Bei automatischer Auslese immer die erkannten Werte dem Nutzer zur Prüfung vorlegen, bevor Buchungen erstellt werden:
- "Ich habe folgende Daten aus dem Beleg erkannt: [Betrag], [Datum], [Lieferant]. Stimmt das?"
- Erst nach Bestätigung fortfahren

### Hinweis zur Genauigkeit

- OCR-Ergebnisse können fehlerhaft sein, insbesondere bei:
  - Handschriftlichen Belegen
  - Schlechter Scanqualität
  - Ungewöhnlichen Layouts
- Bei Unsicherheit: Nutzer auf manuelle Prüfung hinweisen

## Fehlerbehandlung

- **Klardaten nicht verbunden:** Nutzer informieren, ASCII-Export anbieten
- **Mandant nicht gefunden:** Mandantennummer prüfen lassen
- **Unbekanntes Feld benötigt:** Prüfen ob in API-Feldliste, sonst ASCII-Export, wenn ASCII Export nicht dem Format aus der Kanzleivorlage entspricht (z.B. Zusatzfelder), gib dem Nutzer zusätzlich einen Hinweis, dass er sich selbst eine Formatvorlage in DATEV erstellen muss.
- **API-Fehler:** Fehlermeldung anzeigen, ASCII-Export als Fallback anbieten
- **Beleg nicht lesbar:** Bei OCR-Fehlern auf manuelle Eingabe (Option 1) zurückfallen

## Häufige API-Fehler und Lösungen

### 1. "dateFrom and dateTo must be in the same calendar month"

**Ursache:** Der Buchungszeitraum erstreckt sich über mehrere Kalendermonate.

**Lösung:** 
- **Für jeden Kalendermonat einen separaten Buchungsstapel erstellen!**
- Bei Q1 2026 also 3 Stapel: Januar, Februar, März
- Bei Jahresbuchhaltungen: 12 Stapel (je Monat einer)

**Beispiel Q1 2026:**
```
Stapel 1: dateFrom 2026-01-01, dateTo 2026-01-31 (alle Januar-Buchungen)
Stapel 2: dateFrom 2026-02-01, dateTo 2026-02-28 (alle Februar-Buchungen)
Stapel 3: dateFrom 2026-03-01, dateTo 2026-03-31 (alle März-Buchungen)
```

### 2. "Umsatzsteuerschlüssel ist mit den Zusatzfunktionen der beiden Konten nicht verträglich"

**Ursache:** Das Sachkonto hat eine eigene Steuerautomatik, die mit dem BU-Schlüssel kollidiert.

**Lösung:** 
- **VOR dem Buchen:** Bestehende Buchungen auf diesem Konto/Kreditor analysieren (`datev://accounting/account_postings`)
- Prüfen, welches Sachkonto und welcher BU-Schlüssel in der Vergangenheit verwendet wurde
- Manche Mandanten buchen Nettobetrag + Vorsteuer **separat** (z.B. Aufwand auf 5400, VSt auf 1406), statt über BU-Schlüssel
- Im Zweifel: Buchung ohne `accountingTransactionKey` erstellen — DATEV verwendet dann die Kontenautomatik

### 3. "Buchungsschlüssel 'Aufhebung der Automatik' ist nicht erlaubt"

**Ursache:** BU-Schlüssel 40 (ohne Vorsteuer / Aufhebung der Automatik) ist für diese Kontenkombination nicht zulässig.

**Lösung:** Buchung **ohne** `accountingTransactionKey` erstellen. Typische Fälle:
- **Behördengebühren** (keine USt auf der Rechnung)
- **Versicherungsprämien** (Versicherungsteuer, keine Vorsteuer)
- **Ausländische Rechnungen ohne deutsche USt**
- **Gesellschaften ohne Vorsteuerabzug** (Holdings, vermögensverwaltende GmbHs)

### 4. "Lieferungs- und Leistungsart ist falsch oder wurde nicht erfasst"

**Ursache:** BU-Schlüssel 91–99 (§ 13b UStG / Reverse Charge / ig. Erwerb) erfordern das Zusatzfeld `casesRelatedToGoodsAndServices`.

**WICHTIG: Bei BU 91–99 IMMER das Zusatzfeld `casesRelatedToGoodsAndServices` mitgeben!**

**Betroffene BU-Schlüssel:**
- 91 (Steuerpflichtige innergemeinschaftliche Lieferung)
- 93 (Innergemeinschaftlicher Erwerb)
- 94 (Reverse Charge § 13b Eingangsleistung)
- 95 (Reverse Charge § 13b Bauleistungen)
- 96–99 (Weitere § 13b-Sachverhalte)

**Werte für `casesRelatedToGoodsAndServices`:**

| Wert | § 13b-Fall |
|------|------------|
| 10 | Werklieferungen/sonstige Leistungen eines im Ausland ansässigen Unternehmers |
| 11 | Lieferungen sicherungsübereigneter Gegenstände |
| 12 | Grundstücksübertragungen |
| 13 | Bauleistungen |
| 18 | Telekommunikationsleistungen |
| 19 | Gas-/Elektrizitätslieferungen |

**Beispiel (Drittlandsleistung USA):**
```json
{
  "accountNumber": 6815,
  "contraAccountNumber": 70100,
  "amount": 350.00,
  "debitCreditIdentifier": "S",
  "date": "2026-09-15T00:00:00+02:00",
  "postingDescription": "Softwarelizenz USA",
  "accountingTransactionKey": 94,
  "casesRelatedToGoodsAndServices": 10
}
```

### 5. "Die Buchungsstapel konnten nicht erzeugt werden"

**Ursache:** Unspezifischer Fehler, oft durch ungültige Zeichen oder Feldformate.

**Lösung:** 
- **Keine Leerzeichen** in `documentField1` (Belegnummer)
- **Keine Umlaute** in `postingDescription` — ä→ae, ö→oe, ü→ue, ß→ss
- Buchungen einzeln testen, um die fehlerhafte zu identifizieren

## Best Practices

### Vor der ersten Buchung: Vergangenheit analysieren

Bevor Buchungsvorschläge erstellt werden, **immer** die bestehenden Buchungen analysieren:

```
datev_accounting_list
  resourceUri: datev://accounting/account_postings
  clientId: <GUID>
  fiscalYearId: <YYYYMMDD>
```

Daraus ableiten:
- Welches **Sachkonto** wurde verwendet?
- Wurde ein **BU-Schlüssel** verwendet oder OHNE gebucht (kein Vorsteuerabzug)?
- Gibt es **Buchungstextmuster**?

### Neue Kreditoren

**VOR dem Anlegen eines neuen Kreditors: Sammelkreditoren prüfen!**

Viele Mandanten haben Sammelkreditoren für einmalige oder seltene Lieferanten:
- **T-Diverse**, **Diverse Kreditoren**, **Sonstige** o.ä.
- Typisch im Bereich 70000–70010 oder am Ende des Kreditorenbereichs

**Vorgehensweise:**

1. **Bestehende Kreditoren durchsuchen** nach Mustern wie:
   - "Diverse", "Sonstige", "Sammel", "Einmal"
   - Kreditoren mit auffällig vielen unterschiedlichen Buchungstexten

2. **Nutzer fragen:**
   > "Für [Lieferantenname] existiert noch kein Kreditor. Ich habe folgende Sammelkreditoren gefunden: [Liste]. Soll ich:
   > - A) Einen neuen Kreditor anlegen (nächste freie Nr. wäre 70xxx)
   > - B) Auf Sammelkreditor [Name] buchen?"

3. **Nur nach Bestätigung** einen neuen Kreditor anlegen.

### Monatliche Stapel bei Jahres-/Quartalsbuchhaltungen

**WICHTIGER HINWEIS für Nutzer:**

Die DATEV-API erfordert, dass `dateFrom` und `dateTo` im selben Kalendermonat liegen. Das bedeutet:

- **Q1 2026** → 3 Stapel (Januar, Februar, März)
- **Jahresbuchhaltung 2026** → 12 Stapel (je Monat einer)
- **Abweichendes Wirtschaftsjahr** (z.B. 01.04.–31.03.) → ebenfalls monatliche Stapel

**Am Ende jeder Buchungserstellung darauf hinweisen!**

## Buchungsmuster nach Belegart

| Belegart | Typisches Konto | BU-Schlüssel | Zusatzfeld | Hinweis |
|----------|-----------------|--------------|------------|---------|
| Rechnung mit 19% USt | Aufwandskonto | 9 | — | Standard |
| Rechnung mit 7% USt | Aufwandskonto | 8 | — | Ermäßigter Satz |
| Behördengebühr (keine USt) | 6300 o.ä. | — (keinen) | — | Kontenautomatik |
| Versicherungsprämie | 6400 | — (keinen) | — | VersStG, keine VSt |
| Rechnung bei Gesellschaft ohne Vorsteuerabzug | Aufwandskonto | — (keinen) | — | Bruttobetrag buchen |
| Drittlandsleistung (USA, UK, CH) | Aufwandskonto | 94 | `casesRelatedToGoodsAndServices: 10` | Reverse Charge § 13b |
| EU-Leistung (Reverse Charge) | Aufwandskonto | 94 | `casesRelatedToGoodsAndServices: 10` | Reverse Charge § 13b |
| Innergemeinschaftlicher Erwerb | Aufwandskonto | 93 | `casesRelatedToGoodsAndServices: 10` | ig. Erwerb |
| Bauleistung (§ 13b) | Aufwandskonto | 95 | `casesRelatedToGoodsAndServices: 13` | Bauabzugssteuer |
| Strom/Energie (PV-Anlagen) | Prüfen! | Prüfen! | — | Oft Netto + VSt separat |
