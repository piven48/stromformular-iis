# Formular-Scanner 1400 – IIS Deployment Anleitung

## Übersicht

Diese Anleitung beschreibt die Installation der Webanwendung auf einem Windows-Server mit IIS.  
Die Anwendung läuft als Node.js-Prozess hinter IIS (via **iisnode**).

---

## Voraussetzungen (einmalig vom IT-Administrator)

### 1. Node.js installieren
- Download: **https://nodejs.org** → LTS-Version (`.msi`)
- Installation mit Standardeinstellungen durchführen
- Prüfen: Eingabeaufforderung öffnen → `node --version` → sollte `v22.x.x` zeigen

### 2. iisnode-Modul installieren
iisnode ermöglicht das Ausführen von Node.js-Anwendungen direkt in IIS.

- Download: **https://github.com/azure/iisnode/releases**
  - Datei: `iisnode-full-v0.2.26-x64.msi` (oder aktuellste Version)
- Installation durchführen
- IIS Manager neu starten

**Prüfen:** IIS Manager → Server → Module → „iisnode" muss in der Liste erscheinen

### 3. URL Rewrite Modul installieren
- Download: **https://www.iis.net/downloads/microsoft/url-rewrite**
- Installation durchführen, IIS neu starten

---

## Installation der Anwendung

### Schritt 1: Anwendungsordner kopieren

Den kompletten Ordner auf den Server kopieren:
```
C:\inetpub\wwwroot\formular-scanner\
```

### Schritt 2: Abhängigkeiten installieren

Eingabeaufforderung **als Administrator** öffnen und ausführen:

```cmd
cd C:\inetpub\wwwroot\formular-scanner
npm install
```

### Schritt 3: Chromium-Browser installieren (WICHTIG – genau so ausführen!)

> **Hintergrund:** IIS führt die Anwendung als App-Pool-Identity aus (nicht als der aktuell angemeldete
> Windows-Benutzer). Chromium muss daher in einem **geteilten Ordner** installiert werden,
> auf den der App-Pool zugreifen kann. Der folgende Befehl erledigt das automatisch.

Eingabeaufforderung **als Administrator** — Befehle **nacheinander** ausführen:

```cmd
mkdir C:\ms-playwright
set PLAYWRIGHT_BROWSERS_PATH=C:\ms-playwright
cd C:\inetpub\wwwroot\formular-scanner
npx playwright install chromium
```

Danach Lese-Rechte für den App-Pool setzen:
```cmd
icacls "C:\ms-playwright" /grant "IIS_IUSRS":(OI)(CI)RX
```

> Falls Antivirus den Download blockiert: `C:\ms-playwright` als Ausnahme hinzufügen,
> dann `npx playwright install chromium` erneut ausführen.

### Schritt 4: Systemumgebungsvariable setzen

Damit der App-Pool Chromium immer findet, die Variable **dauerhaft** als Systemvariable setzen:

1. Windows-Taste → „Umgebungsvariablen" → „Systemumgebungsvariablen bearbeiten"
2. Unter **Systemvariablen** → „Neu"
3. Name: `PLAYWRIGHT_BROWSERS_PATH` | Wert: `C:\ms-playwright`
4. Mit OK bestätigen
5. Server neu starten (oder IIS und App-Pool neu starten)

### Schritt 5: IIS-Anwendung einrichten

1. **IIS Manager** öffnen (`inetmgr`)
2. Unter **Sites** → gewünschte Site → rechte Maustaste → **„Anwendung hinzufügen"**
3. Einstellungen:
   - **Alias:** `formular-scanner`
   - **Anwendungspool:** Neuen Pool anlegen (Name z.B. `FormularScannerPool`)
   - **Physischer Pfad:** `C:\inetpub\wwwroot\formular-scanner`
4. **OK** klicken

### Schritt 6: Anwendungspool konfigurieren

1. IIS Manager → **Anwendungspools** → `FormularScannerPool` → **Erweiterte Einstellungen**
2. Folgende Werte setzen:

| Einstellung | Wert | Grund |
|---|---|---|
| .NET CLR-Version | Kein verwalteter Code | Node.js-App, kein .NET |
| Pipelinemodus | Integriert | Standard |
| Identität | `LocalSystem` oder Dienstkonto | Braucht Schreibrechte auf Temp |
| Leerlauftimeout | `0` | Scans dauern bis 10 Minuten |
| Reguläres Zeitlimit | `0` | Keine automatische Unterbrechung |
| Maximale Arbeitsprozesse | `1` | Wichtig für Node.js |

### Schritt 7: Berechtigungen setzen

```cmd
icacls "C:\inetpub\wwwroot\formular-scanner" /grant "IIS AppPool\FormularScannerPool":(OI)(CI)F
icacls "C:\Windows\Temp" /grant "IIS AppPool\FormularScannerPool":(OI)(CI)F
```

---

## Testen

1. Browser öffnen → `http://SERVERNAME/formular-scanner`
2. Weboberfläche sollte erscheinen
3. Excel-Datei hochladen und Scan starten
4. Fortschrittsbalken sollte sich **live** aktualisieren (nicht erst am Ende)

---

## Optionaler Passwortschutz

Wenn die Seite nur mit Passwort zugänglich sein soll:

1. Windows-Taste → Systemumgebungsvariablen → Neue Systemvariable
2. Name: `SITE_PASSWORD` | Wert: `IhrPasswort`
3. App-Pool neu starten

---

## Troubleshooting

### „500.1000 – iisnode failed to initialize"
→ iisnode-Modul nicht installiert  
→ Node.js nicht im PATH → Node.js neu installieren, Server neu starten

### „404 / Seite nicht gefunden"
→ URL Rewrite Modul nicht installiert  
→ `web.config` fehlt im Anwendungsordner

### „Browser konnte nicht gestartet werden" im Scan-Log
→ Chromium nicht unter `C:\ms-playwright` installiert (Schritt 3 wiederholen)  
→ Systemumgebungsvariable `PLAYWRIGHT_BROWSERS_PATH` nicht gesetzt (Schritt 4)  
→ Antivirus blockiert Chromium — `C:\ms-playwright` als Ausnahme eintragen  
→ App-Pool-Identity hat keine Leserechte auf `C:\ms-playwright` (icacls-Befehl aus Schritt 3 wiederholen)

### Fortschrittsbalken bewegt sich nicht
→ App-Pool neu starten  
→ Prüfen ob `web.config` vollständig vorhanden ist

### Scan bricht nach ca. 2 Minuten ab
→ App-Pool Leerlauftimeout und Zeitlimit prüfen (Schritt 6 — beide auf `0` setzen)

### Fehler-Log prüfen
```
C:\inetpub\wwwroot\formular-scanner\iisnode-logs\
```

---

## Hinweise

- Die Anwendung ruft nur `formulare-bfinv.de` auf — keine anderen externen Verbindungen
- Kein API-Key oder Cloud-Dienst erforderlich
- Ergebnisse werden temporär in `C:\Windows\Temp\formular-jobs\` gespeichert
- Lokaler Betrieb (`.bat`-Datei) bleibt weiterhin möglich und ist unverändert
