# Wallbox Lademodus State Machine (Home Assistant Blueprint)

Dieser Blueprint bildet eine Zustandsmaschine für eine Wallbox ab. Unterstützte Modi:

- `Deaktiviert`
- `Überschussladen`
- `Manuelles Laden`
- `Manuelles Schnellladen`

Zusätzlich gibt es einen internen Status für den Überschusslademodus:

- `Idle`
- `Laden`
- `Pause`

Der Blueprint entscheidet auf Basis von Lademodus, Wallbox-Zustand, Freigabe-Boolean und berechnetem Ladestrom, welche Aktionen an ein Wallbox-Skript übergeben werden.

---

## Voraussetzungen

Folgende Entitäten solltest du vorbereitet haben:

- `input_select` **Lademodus Wallbox**  
  z. B. Optionen: `Deaktiviert`, `Überschussladen`, `Manuelles Laden`, `Manuelles Schnellladen`
- `input_select` **Zustand Wallbox**  
  z. B. `Angeschlossen`, `Ladevorgang`, `Ladevorgang beendet`, `Frei`
- `input_select` **Status Überschussladen**  
  z. B. `Idle`, `Laden`, `Pause`
- `input_boolean` **Wallbox Freigabe** (z. B. PV-Überschuss vorhanden)
- `sensor` **berechneter möglicher Ladestrom** (A)
- `script` für Wallbox-Aktionen, das folgende Variablen verarbeiten kann:
  - `action`: `Start`, `Stop`, `Pause`, `Resume`, `SetChargingCurrent`
  - `chargingCurrent`: Zahl in Ampere (bei `SetChargingCurrent`)

---

## Installation

1. Die Blueprint-Datei (z. B. `wallbox_lademodus_statemachine.yaml`) nach  
   `config/blueprints/automation/<dein_ordner>/` kopieren.
2. Home Assistant neu laden oder Blueprints neu einlesen.
3. Unter **Einstellungen → Automationen & Szenen → Blueprints** den Blueprint auswählen.
4. Eine neue Automation auf Basis des Blueprints anlegen und die passenden Entitäten zuweisen.

---

## Logik – Überblick

### 1. Lademodus Deaktiviert / unavailable / unknown

- Wenn Wallbox-Zustand `Ladevorgang`:
  - Aktion: `Stop`
- Überschuss-Status wird auf `Idle` gesetzt.

### 2. Lademodus Überschussladen

- Wenn **Freigabe = aus**:
  - `Ladevorgang`: Status `Pause`, Aktion `Pause`
  - `Frei`: Status `Idle`, Aktion `Stop`
  - `Ladevorgang beendet`: keine Aktion
- Wenn **Freigabe = an**:
  - `Idle + Angeschlossen`: Status `Laden`, Aktion `Start`
  - `Pause + Ladevorgang beendet`: Status `Laden`, Aktion `Resume`
  - `Pause + Ladevorgang`: Status `Laden`
  - Immer: Aktion `SetChargingCurrent(ChargingCurrent)`
- Wenn **Freigabe = an und Ladevorgang**:
  - wiederholte Stromnachführung via `SetChargingCurrent(ChargingCurrent)`
- Wenn **nicht Pause und Ladevorgang beendet**:
  - Status `Idle`

### 3. Lademodus Manuelles Laden / Manuelles Schnellladen

- `Angeschlossen`:
  - Aktion `Start`
  - Ladestrom:
    - Manuell: `MINIMUM_CHARGING_CURRENT`
    - Schnellladen: `MAXIMUM_CHARGING_CURRENT`
- `Ladevorgang`:
  - Nur Ladestrom setzen auf MIN/MAX
- `Ladevorgang beendet`:
  - Lademodus wird auf `Deaktiviert` gesetzt
- Am Ende:
  - Überschuss-Status auf `Idle`

---

## Mermaid – UML State Diagram (Lademodus + Überschussstatus)

```mermaid
stateDiagram-v2
    [*] --> Deaktiviert

    state Deaktiviert
    state Ueberschussladen
    state Manuell
    state ManuellSchnell

    Deaktiviert --> Ueberschussladen: Modus = Überschussladen
    Deaktiviert --> Manuell: Modus = Manuelles Laden
    Deaktiviert --> ManuellSchnell: Modus = Manuelles Schnellladen

    Ueberschussladen --> Deaktiviert: Modus = Deaktiviert\nunavailable\nunknown
    Manuell --> Deaktiviert: Ladevorgang beendet
    ManuellSchnell --> Deaktiviert: Ladevorgang beendet

    state Ueberschussladen {
        [*] --> Idle

        Idle --> Laden: Freigabe an und Angeschlossen
        Laden --> Pause: Freigabe aus und Ladevorgang
        Laden --> Idle: Freigabe aus und Frei
        Laden --> Idle: nicht Pause und Ladevorgang beendet
        Pause --> Laden: Freigabe an und Ladevorgang
        Pause --> Laden: Freigabe an und Ladevorgang beendet
    }
