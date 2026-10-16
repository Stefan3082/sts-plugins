---
name: browser-download
description: "Lädt Dateien aus dem integrierten Browser von Claude Desktop herunter, ohne dass der Nutzer jeden Windows-Speichern-Dialog bestätigen muss. Verwenden, wenn der Nutzer im geöffneten Browser eine oder mehrere Dateien herunterladen will (z. B. Handelsregister-Ausdrucke, PDFs aus Portalen)."
---

# Browser-Download (Speichern-Dialog umgehen)

Lade eine oder mehrere Dateien aus dem geöffneten Browser-Pane herunter, ohne dass
der Nutzer jeden Windows-Speichern-Dialog einzeln bestätigen muss.

## Kontext

Der integrierte Browser (Claude Desktop) öffnet bei jedem Download einen
Windows-Speichern-Dialog **außerhalb** des Browser-Panes. Solange dieser Dialog
offen ist, liegt die Datei als `<GUID>.tmp` im Downloads-Ordner des Nutzers.
Dieses Verfahren kopiert die .tmp-Datei direkt heraus und schließt den Dialog
danach automatisch per PowerShell.

**Wichtig**: Der Dialog hat in Claude Desktop als Titel die **URL der Seite**
(nicht "Speichern unter") und die Window-Klasse `#32770`. `#32770` ist die
allgemeine Windows-Klasse für ALLE Dialogfelder aller Programme — deshalb den
Dialog immer zusätzlich über Prozess (Claude) und Titel (Host der Seite) eingrenzen.

## Argumente (aus $ARGUMENTS)

Der Nutzer übergibt Freitext, z. B.:
- `klick auf AD` — einzelner Download, Element wird per Beschreibung identifiziert
- `alle AD-Links auf dieser Seite` — mehrere Downloads nacheinander
- `css:#myBtn` — Element per CSS-Selektor
- `ref:ref_42` — Element per Accessibility-Ref aus read_page

Optional am Ende:
- `-> Ordnerpfad` — Zielordner (Default: aktueller Projektordner / Working Directory)
- `als Dateiname.pdf` — fester Dateiname (Default: aus Content-Disposition oder Zeitstempel)

## Ablauf für JEDEN einzelnen Download

### 1. Vorher: bestehende .tmp-Dateien merken

```bash
ls "$USERPROFILE/Downloads/"*.tmp 2>/dev/null | sort > /tmp/tmp_before.txt
```

### 2. Download auslösen

Identifiziere das Download-Element im Browser-Pane (per `find`, `read_page`
oder `javascript_tool`) und klicke es. Verwende bevorzugt `javascript_tool`
mit `element.click()`, damit kein visuelles Zielen nötig ist.

Falls das Element ein PrimeFaces-Formular-Submit ist (erkennbar an
`PrimeFaces.monitorDownload` im onclick), klicke per `.click()` auf das
`<a>`- oder `<button>`-Element.

**WICHTIG**: Keine fetch()/XMLHttpRequest-Versuche für den Download — der
Claude-Desktop-Browser fängt Antworten mit Content-Disposition:attachment ab,
bevor JavaScript sie lesen kann. Solche Versuche zerstören außerdem die
PrimeFaces-Session (ViewState wird ungültig).

### 3. Auf .tmp-Datei warten

Warte 2–5 Sekunden, dann suche nach einer NEUEN .tmp-Datei:

```bash
ls "$USERPROFILE/Downloads/"*.tmp 2>/dev/null | sort > /tmp/tmp_after.txt
comm -13 /tmp/tmp_before.txt /tmp/tmp_after.txt
```

Falls keine neue Datei erscheint, bis zu 5× mit je 3 s Pause wiederholen
(manche Server brauchen bis zu 15 s). Danach abbrechen und dem Nutzer melden.

### 3b. Warten, bis die Datei vollständig ist

Der Browser lädt weiter, während der Dialog offen ist. Erst kopieren, wenn die
Größe bei zwei Messungen im Abstand von 1 s gleich und größer als 0 ist
(höchstens 30 s warten, sonst abbrechen und melden):

```bash
prev=-1; for i in $(seq 1 30); do s=$(stat -c %s "<tmp-Pfad>"); [ "$s" -gt 0 ] && [ "$s" = "$prev" ] && break; prev=$s; sleep 1; done
```

### 4. Dateiname bestimmen

Bevorzugte Reihenfolge:
1. Vom Nutzer angegebener Name (`als ...`)
2. Aus dem Content-Disposition-Header (per `read_network_requests` die letzte
   POST/GET-Anfrage prüfen, die `attachment;filename=` enthält) — nur ein
   Versuch, die Header sind dort nicht immer sichtbar
3. Fallback: `download_<YYYYMMDD_HHmmss>.<endung>`; die Endung aus den ersten
   Bytes ableiten: `%PDF-` → `.pdf`, `PK` → `.zip` (bzw. `.xlsx`/`.docx`, wenn die
   Seite das nahelegt), `<?xml` → `.xml`, sonst `.bin` und den Nutzer fragen

### 5. Kopieren (nie überschreiben)

Vorher prüfen, ob die Zieldatei schon existiert. Falls ja: Nutzer fragen oder
einen neuen Namen mit Zusatz `_2`, `_3` vergeben. Dann ohne Überschreiben kopieren:

```bash
[ -e "<Zielordner>/<Dateiname>" ] && echo "EXISTIERT" || cp -n "<tmp-Pfad>" "<Zielordner>/<Dateiname>"
```

NIEMALS die .tmp-Datei verschieben (mv) — nur kopieren (cp). Die .tmp
verschwindet von selbst, wenn der Dialog geschlossen wird.

### 6. Verifizieren

- Größe der Kopie = Größe der .tmp-Datei (`stat -c %s`)
- Magic Bytes (`head -c 5 | cat -v`): PDF → `%PDF-`
- Bei PDF zusätzlich: `%%EOF` in den letzten 1024 Bytes (`tail -c 1024 | grep -a -c '%%EOF'`),
  sonst ist die Datei vermutlich abgeschnitten → warnen

Melde dem Nutzer Dateiname, Größe und Pfad.

### 7. Speichern-Dialog automatisch schließen

ERST schließen, NACHDEM Kopie und Prüfung in Schritt 5 und 6 erfolgreich waren.

Nur einen Dialog schließen, der (a) die Klasse `#32770` hat, (b) zu einem
Prozess namens `claude` gehört und (c) den Host der Download-Seite im Titel
trägt (z. B. `handelsregister.de`). Ohne `-Close` listet das Skript nur die
Treffer auf (zum Prüfen).

```powershell
param([Parameter(Mandatory=$true)][string]$TitlePart, [switch]$Close)
Add-Type @'
using System;
using System.Runtime.InteropServices;
using System.Text;
using System.Collections.Generic;
public class DialogFinder {
    public delegate bool EnumWindowsProc(IntPtr hWnd, IntPtr lParam);
    [DllImport("user32.dll")] public static extern bool EnumWindows(EnumWindowsProc f, IntPtr l);
    [DllImport("user32.dll", CharSet=CharSet.Unicode)] public static extern int GetWindowText(IntPtr h, StringBuilder s, int n);
    [DllImport("user32.dll")] public static extern bool IsWindowVisible(IntPtr h);
    [DllImport("user32.dll", CharSet=CharSet.Unicode)] public static extern int GetClassName(IntPtr h, StringBuilder s, int n);
    [DllImport("user32.dll")] public static extern uint GetWindowThreadProcessId(IntPtr h, out uint pid);
    [DllImport("user32.dll")] public static extern bool PostMessage(IntPtr h, uint m, IntPtr w, IntPtr l);
    public const uint WM_CLOSE = 0x0010;
    public static List<object[]> FindDialogs() {
        var list = new List<object[]>();
        EnumWindows((h, l) => {
            if (!IsWindowVisible(h)) return true;
            var cls = new StringBuilder(256); GetClassName(h, cls, 256);
            if (cls.ToString() != "#32770") return true;
            var t = new StringBuilder(1024); GetWindowText(h, t, 1024);
            uint pid; GetWindowThreadProcessId(h, out pid);
            list.Add(new object[] { h, t.ToString(), pid });
            return true;
        }, IntPtr.Zero);
        return list;
    }
}
'@
$hits = @()
foreach ($d in [DialogFinder]::FindDialogs()) {
    $proc = Get-Process -Id ([int]$d[2]) -ErrorAction SilentlyContinue
    if ($proc -and $proc.ProcessName -ieq 'claude' -and $d[1] -like "*$TitlePart*") { $hits += ,$d }
}
Write-Host ("Treffer: " + $hits.Count)
foreach ($d in $hits) { Write-Host ("  " + $d[1]) }
if ($Close) {
    if ($hits.Count -eq 1) {
        [void][DialogFinder]::PostMessage($hits[0][0], [DialogFinder]::WM_CLOSE, [IntPtr]::Zero, [IntPtr]::Zero)
        Write-Host "Dialog geschlossen."
    } else {
        Write-Host "Nicht geschlossen: genau ein Treffer erwartet."
    }
}
```

Das Skript als `.ps1` (nur ASCII-Zeichen) in einem Temp-Ordner speichern und mit
`powershell -NoProfile -ExecutionPolicy Bypass -File <pfad> -TitlePart <host> -Close`
aufrufen. Bei 0 oder mehreren Treffern nichts schließen, sondern den Nutzer bitten,
im Dialog auf „Abbrechen“ zu klicken.

## Batch-Modus (mehrere Downloads)

Bei mehreren Downloads (z. B. „alle AD-Links"):
- Iteriere über die Elemente in einer Schleife
- Zwischen den Downloads jeweils die Schritte 1–7 durchlaufen
- Nach dem Dialog-Dismiss (Schritt 7) 1–2 Sekunden warten, damit der Browser
  bereit für den nächsten Download ist
- Fasse am Ende zusammen: wie viele Dateien heruntergeladen, Gesamtgröße,
  etwaige Fehler

## Fehlerbehandlung

- Keine .tmp-Datei nach 20 s → Meldung: „Kein Download erkannt. Möglicherweise
  erfordert die Seite eine Anmeldung oder das Element löst keinen Download aus."
- Mehrere neue .tmp-Dateien → die größte nehmen und warnen
- Dateigröße 0 → warnen, nicht kopieren
- PDF ohne `%%EOF` → warnen (Datei vermutlich unvollständig)
- Zielordner existiert nicht → nachfragen, ob er erstellt werden soll
- Dialog nicht gefunden oder mehrere Treffer → Nutzer bitten, manuell auf Abbrechen zu klicken

## Sicherheit

- Niemals Dateien aus dem Downloads-Ordner löschen oder verschieben
- Niemals bestehende Dateien im Zielordner überschreiben ohne Rückfrage
- Downloads nur von der Seite auslösen, die der Nutzer im Browser geöffnet hat
- Heruntergeladene Dateien nicht öffnen oder ausführen — nur Größe und erste/letzte
  Bytes prüfen; Inhalte erst lesen, wenn der Nutzer das möchte
- Nur Dialoge von Claude schließen (Prozess und Titel prüfen), nie andere Programme
