# Theater HA Testsetup - Windows

## ⚠️ Der Syntaxfehler ist gefunden!

In deinem `scripts.yaml` war in `theater_streit_lauf_v2`:
```yaml
repeat:
  while: '{{ true }}'   # ❌ falsch
```
Richtig:
```yaml
repeat:
  while:
    - condition: template
      value_template: 'true'   # ✅
```
Ist in der mitgelieferten `scripts.yaml` schon gefixt.

---

## Neu in diesem Paket: MQTT statt Template Lights

Template Lights sind in neueren HA-Versionen deprecated und zicken. Wir nutzen stattdessen **echte MQTT Lights** mit einem eigenen Broker (zweiter Container). Das ist der Standard-Weg in HA und funktioniert stabil.

---

## Schritt 1: Ordner & Files

Ordner `C:\theater-ha` anlegen. Files reinkopieren:
- `ANLEITUNG.md`
- `docker-compose.yml`
- `configuration.yaml`
- `scripts.yaml`
- `visualizer.html`
- Ordner `mosquitto\config\mosquitto.conf` (wichtig: Unterordner!)

Die Struktur muss so aussehen:
```
C:\theater-ha\
├── ANLEITUNG.md
├── docker-compose.yml
├── configuration.yaml
├── scripts.yaml
├── visualizer.html
└── mosquitto\
    └── config\
        └── mosquitto.conf
```

---

## Schritt 2: Alte Installation wegräumen (falls du vorher schon probiert hast)

In Powershell:
```powershell
cd C:\theater-ha
docker compose down -v
```
Das löscht alle alten Container und Daten - sauberer Start.

Löscht auch den alten `config`-Ordner im theater-ha falls vorhanden:
```powershell
Remove-Item -Recurse -Force .\config -ErrorAction SilentlyContinue
```

---

## Schritt 3: Container starten

```powershell
cd C:\theater-ha
docker compose up -d
```

Jetzt laufen **zwei** Container:
- `theater-ha` - Home Assistant auf Port 8123
- `theater-mqtt` - MQTT Broker auf Port 1883

Erster Start dauert 2-5 Minuten (HA lädt Pakete).

Check:
```powershell
docker compose ps
```
Beide müssen "running" sein.

---

## Schritt 4: HA UI öffnen & User anlegen

Browser: **http://localhost:8123**

- Account anlegen (irgendwas, lokal egal)
- Beim Standort/Zeitzone durchklicken
- Bei "Integrationsvorschläge" auf **Finish** klicken

---

## Schritt 5: MQTT-Integration hinzufügen

Das ist der Schritt der neu dazukommt:

1. **Settings → Devices & Services → + Add Integration**
2. Suchen nach: **MQTT**
3. Klicken, Formular öffnet sich:
   - Broker: `mosquitto` (der Container-Name!)
   - Port: `1883`
   - Username: (leer lassen)
   - Password: (leer lassen)
4. **Submit**

HA sollte "Connection successful" zeigen. Die 12 MQTT-Lampen werden beim nächsten Reload angezeigt.

---

## Schritt 6: Check - sind die Lampen jetzt da?

**Developer Tools → States** (Sidebar links, nach unten scrollen)

Du solltest sehen:
- `light.l1` bis `light.l12` mit State "off"
- `input_number.theater_master_brightness` etc.
- `script.theater_neutral_v2` etc.

Wenn ja: **läuft.**

Falls Developer Tools nicht sichtbar: Oben rechts auf deinen Namen → runter scrollen → "Advanced Mode" einschalten.

---

## Schritt 7: Access Token

Profil (oben rechts auf Namen klicken) → **Security** → runter zu **Long-Lived Access Tokens** → **Create Token**. Namen z.B. "powerpoint". Token **SOFORT kopieren** (einmaliger Anzeige).

---

## Schritt 8: Erster Test - Lampe im UI

Developer Tools → **Services**:
- Service: `light.turn_on`
- Target Entity: `light.l1`
- YAML-Modus einschalten, dann:
```yaml
service: light.turn_on
target:
  entity_id: light.l1
data:
  rgb_color: [255, 0, 0]
  brightness_pct: 100
```
- **Call Service**

Zurück zu Developer Tools → States → `light.l1`: Sollte State "on" haben, `rgb_color: [255, 0, 0]`, `brightness: 255`.

Wenn das klappt: **Test jetzt ein Script!**

Developer Tools → Services → Service: `script.theater_blau_v2` → Call Service.

States prüfen: Die Lampen haben jetzt rgb_color `[0, 150, 255]`.

---

## Schritt 9: Visualizer starten

`visualizer.html` mit Doppelklick öffnen (öffnet sich im Browser).

- HA URL: `http://localhost:8123` (default passt)
- Access Token: den aus Schritt 7 einsetzen
- **Verbinden**

Die 12 Lampen-Punkte werden live synchron mit HA. Am besten auf zweitem Monitor platzieren.

### Zwei Modi - wichtig!

Oben im Visualizer kannst du umschalten zwischen:

**Ideal**: zeigt exakt was HA sendet, mit smoothen transitions. So würden *perfekte* Lampen aussehen.

**Tuya LedAdvance**: simuliert das echte Verhalten deiner Theater-Lampen:
- ~250ms Command-Latenz (wie WiFi-Tuya in echt braucht)
- `transition:` wird **ignoriert** — die Lampe springt immer zum Endwert
- Commands die weniger als 180ms auseinander kommen, werden verworfen
- Color↔White-Mode-Wechsel blitzt weiß auf
- Brightness unter 1% funktioniert nicht

**Schalte auf Tuya-Mode wenn du testen willst wie es im Theater wirklich aussehen wird.** Deine Rot-Flicker und Streit-Phasen sehen da wahrscheinlich ganz anders aus als im Ideal-Mode!

---

## Schritt 10: VBA auf lokales HA umstellen

In deiner PPT (VBA Editor mit Alt+F11):

In `Homeassitent.bas` und `Modul4Folien.bas`:
```vba
Private Const HA_BASE As String = "http://localhost:8123"
Private Const HA_TOKEN As String = "<dein Token aus Schritt 7>"
```

Speichern. PPT in den Präsentationsmodus, Szenen durchklicken, Visualizer beobachten.

---

## Wichtige Kommandos

```powershell
# Beide Container stoppen
docker compose down

# Neustarten (nach config-Änderung)
docker compose restart homeassistant

# Logs live anschauen
docker compose logs -f homeassistant

# Alles frisch (ACHTUNG: löscht auch HA-User!)
docker compose down -v
```

Nach Änderung an `scripts.yaml` reicht oft:
- HA UI → Developer Tools → "YAML Configuration Reloading" → **Scripts**

Nach Änderung an `configuration.yaml`:
- `docker compose restart homeassistant`

---

## Wenn was nicht klappt

**Container startet nicht:**
```powershell
docker compose logs -f homeassistant
docker compose logs -f mosquitto
```
Die Fehlermeldung zeigt's.

**MQTT-Integration findet den Broker nicht:**
Im Feld "Broker" muss `mosquitto` stehen (nicht `localhost`!), weil innerhalb des Container-Netzwerks gesprochen wird.

**Lampen tauchen nicht auf nach Broker-Setup:**
HA neu starten: `docker compose restart homeassistant`. Dann prüfen ob in Settings → Devices & Services unter MQTT die 12 Entities auftauchen.

**Scripts haben Fehler:**
Settings → System → **Logs**. Dort werden Script-Fehler zeilengenau angezeigt. Schick mir den Text.

**Visualizer zeigt nichts:**
F12 öffnen → Console-Tab. Wenn rote Errors: Token falsch? URL falsch?
