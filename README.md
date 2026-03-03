# PV Zero Surplus (Solar-Überschuss Steuerung)

Dieser Blueprint steuert den Solar-Überschuss auf drei verschiedene Arten. Er reagiert auf Änderungen des Netzbezugs-Sensors und passt entweder das Wechselrichter-Limit an, schaltet Geräte ein/aus oder regelt die Leistung von Verbrauchern stufenlos.

---

## Installation

Den Blueprint einfach über den Button in dein Home Assistant intrieren. Die Datei wird automatisch unter Blueprints gespeichert.

<a href="https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fjayjojayson%2Fblueprint_pv-zero-surplus%2Fmain%2Fpv-zero-surplus.yaml">
  <img width="250" alt="blueprint" src="https://github.com/user-attachments/assets/fa01530a-1d52-4b2b-b637-1269bd0cd747">
</a>  

> Trigger:
> Blueprint wird bei jeder Änderung des Netzbezug-Sensors ausgeführt (außer bei "unavailable" oder "unknown").


---

## Betriebsmodi

### ⚡ Modus 1: WR-Limit (Nulleinspeisung)

Passt das Leistungslimit des Wechselrichters dynamisch an, um eine Nulleinspeisung zu erreichen.

**Berechnung:**
```
Ziel-Limit = Aktuelles Limit + Netzbezug - Puffer
```

- **Puffer:** Gewünschter Restbezug (z.B. 10W), um Messschwankungen auszugleichen
- **Hysterese:** Änderungen werden erst gesendet, wenn der Unterschied größer als der Hysterese-Wert ist (schont die Funkverbindung)

**Beispiel:**
- Netzbezug: 200W (Bezug vom Netz)
- Puffer: 10W
- Berechnetes Limit: 190W höher als aktuell → WR wird reduziert

---

### 🔌 Modus 2: Gerät Ein/Aus

Schaltet Geräte basierend auf dem verfügbaren Überschuss ein oder aus.

**Logik für jedes Gerät:**

```
WENN Solarleistung > 0 UND Überschuss >= Einschaltschwelle UND Gerät ist AUS
    UND (Gerät hat Priorität 1 ODER alle Geräte mit höherer Priorität sind bereits eingeschaltet)
    → Gerät EINschalten

WENN (Überschuss < Ausschaltschwelle ODER Solarleistung = 0) UND Gerät ist AN
    → Gerät AUSschalten
```

**Beispiel mit Prioritäten:**
- Gerät A: Priorität 1, Einschaltschwelle 300W
- Gerät B: Priorität 2, Einschaltschwelle 500W
- Überschuss: 600W

→ Gerät A wird zuerst eingeschaltet (höhere Priorität)
→ Gerät B wird eingeschaltet, wenn noch Überschuss verfügbar ist

**Hinweis:** Die Ausschaltschwelle sollte immer niedriger als die Einschaltschwelle sein (Hysterese), um ständiges Ein/Aus-Fladern zu vermeiden.

---

### 📊 Modus 3: Leistungssteuerung

Regelt die Leistung von Verbrauchern stufenlos (z.B. Heizstab, Wallbox).

**Berechnung für jedes Gerät:**

```
Verfügbarer Überschuss = Überschuss - Puffer

WENN Solarleistung = 0 ODER Verfügbarer Überschuss < Minimale Geräteleistung
    → Ziel-Leistung = 0
SONST WENN (Gerät hat Priorität 1 ODER alle Geräte mit höherer Priorität sind auf Maximalleistung)
    → Ziel-Leistung = min(Verfügbarer Überschuss, Maximale Geräteleistung)
SONST
    → Ziel-Leistung = 0
```

**Beispiel mit Prioritäten:**
- Gerät A: Priorität 1, Min 100W, Max 1000W
- Gerät B: Priorität 2, Min 200W, Max 1500W
- Überschuss: 2000W, Puffer: 10W → Verfügbar: 1990W

1. Gerät A (Priorität 1) → 1000W (maximal)
2. Verbleibend: 990W
3. Gerät B (Priorität 2) → 990W (begrenzt durch verfügbaren Überschuss)

---

## Unterstützte Gerätetypen

### Ein/Aus-Geräte
- Switch
- Input Boolean
- Light
- Fan
- Alle Entitäten mit `turn_on` / `turn_off` Support

### Leistungssteuerungs-Geräte
- Number (z.B. OpenDTU Limit)
- Input Number
- Alle Entitäten mit `set_value` Support

---

## Sensoren

| Sensor | Beschreibung |
|--------|--------------|
| **Stromzähler Sensor** | Netzbezug/Einspeisung in Watt. Positiv = Bezug, Negativ = Einspeisung |
| **Wechselrichter Leistung** | Aktuelle Solarproduktion in Watt |

---

## Wichtige Hinweise

1. **WR-Limit:** Unbedingt die NON-PERSISTENT Limit-Entity verwenden! Bei häufigem Schreiben wird sonst der EEPROM/Flash des Wechselrichters zerstört.

2. **Puffer:** Ein Puffer von 5-20W verhindert ungewollte Einspeisung durch Messschwankungen.

3. **Mehrere Geräte:** Bis zu 3 Ein/Aus-Geräte und 3 Leistungssteuerungs-Geräte können konfiguriert werden. Leere Slots werden automatisch ignoriert.

4. **Prioritäten:** 
   - Priorität 1 = höchste Priorität (wird zuerst eingeschaltet/geregelt)
   - Geräte mit niedrigerer Priorität (2, 3) werden erst eingeschaltet/geregelt, wenn alle Geräte mit höherer Priorität bereits eingeschaltet sind oder ihre Maximalleistung erreicht haben

5. **Hysterese:** Sowohl beim WR-Limit als auch bei Ein/Aus-Geräten verhindert die Hysterese ständiges Flattern bei kleinen Änderungen.

