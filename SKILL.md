---
name: HelloFresh_Home
description: "Wochen-Mahlzeitenplaner fuer Rike (13, Pubertaet) und Papa (47) mit Prospekt-Angebots-Integration, Resteverwertung, Budget 8,50 EUR/Tag und Küchen-Vielfalt-Pflichtregeln (1x Italienisch, 1x Deutsch, 1x Vegetarisch, 1x Burger/HotDog pro Woche). Trigger: Essensplan, Wochenplan Essen, Mahlzeitenplan, was kochen wir, Rezeptvorschlag, Einkaufsliste, Angebote einlesen, Prospekt scannen, Reste verbrauchen, Budget Essen, TK-Planung, neues Rezept suchen, Chefkoch Rezept. Output: Wochenplan-PDF, Einkaufsliste nach Markt, aktualisierte ernaehrung.db."
---

# Ernaehrungs-Skill v2.2

## Zweck

Woechentlicher Mahlzeitenplaner fuer **Rike (13 J., Pubertaet) + Papa Matthias (47)**. Kombiniert:
1. Persistente `ernaehrung.db` (Rezepte, Angebots-Historie, Wochenplan-Verlauf)
2. Woechentlich eingelesene Supermarkt-Prospekte (Foto-Scan oder Text-Liste)
3. Budget-Steuerung (8,50 EUR/Tag)
4. Wochenrhythmus-Pflichtregeln (Kuechen-Vielfalt)
5. Wiederholungs-Schutz (keine gleichen Rezepte 2 Wochen hintereinander)
6. Neue Rezepte durch Online-Recherche (Chefkoch, Reddit, Communities)
7. Resteverwertungs-Logik
8. Langzeit-Preis-Analyse (Angebots-Daten bleiben dauerhaft in DB)

---

## MAHLZEITEN-STRUKTUR

| Zeit | Mahlzeit | Inhalt |
|------|----------|--------|
| 07:00 | Schulbrot Rike | rikes-schul-snacks Skill — KEIN Doppel-Output hier |
| **14:30** | **Kleines Mittag** | leicht, ~500-600 kcal, max 20 Min Zubereitung |
| **18:00-19:00** | **Grosses Abend** | ~650-750 kcal, Protein-Fokus, vor 20:30 fertig |
| Bett | 21:00 | Kein Essen nach 20:30 |

### Abendmahlzeit-Constraints:
- Kein schweres Fleisch/Frittiertes als Regel-Abendessen (Wochenende Ausnahme)
- Protein: Haehnchen, Pute, Fisch, Magerquark, Rind
- TK-Gemuese bevorzugt (praktisch, naehrstoffreich, kein Wegwerfen)

---

## WOCHENRHYTHMUS-PFLICHTREGELN (NEU v2.2)

Pro Woche MUSS genau einmal vorkommen (als Abend- oder Mittagsmahlzeit):

| Slot | Pflicht | Erlaubte Varianten |
|------|---------|--------------------|
| **Slot I** | **Italienisch** | Pasta (Bolognese, Carbonara-Art, Pomodoro, Pesto, Arrabiata), Pizza (Wochenende), Risotto |
| **Slot II** | **Deutsch klassisch** | Schnitzel + Kartoffeln, Frikadellen + Kartoffelpueree, Gulasch, Eintopf rheinisch, Braten (Sonntag) |
| **Slot III** | **Vegetarisch** | Normales Alltagsessen — Linsen-Curry, Bohnen-Chili, Gemuese-Pasta, Erbsensuppe, Quark-Bowl. KEIN "Hippie-Food", KEINE Zutaten die man nicht im Supermarkt bekommt |
| **Slot IV** | **Burger oder Hot Dog** | Haehnchen-Burger, Rinderburger, Hot Dog mit Bratwurst, Cheese Burger. Ofen-Pommes oder Karotten-Pommes als Beilage |

### Pflichtregeln-Umsetzung:
- Slots koennen auf beliebige Wochentage verteilt werden
- Slot II (Deutsch) bevorzugt Freitag oder Wochenende (Familienessen-Charakter)
- Slot IV (Burger/HotDog) passt gut auf Samstag oder Mittwoch als Highlight
- Slot III (Vegetarisch) bevorzugt Montag oder Dienstag (pragmatischer Wochenstart)
- Wenn ein Angebot einen Slot beguenstigt → diesen Slot entsprechend belegen

### Pflichtregeln-Check vor PDF-Export:
```python
def check_pflichtregel_slots(wochenplan):
    """wochenplan = Liste der Rezepte der Woche mit ihren Tags"""
    required = {
        'italienisch': False,
        'deutsch-klassisch': False,
        'vegetarisch': False,
        'burger-hotdog': False
    }
    for rezept in wochenplan:
        tags = rezept.get('tags', '')
        if any(t in tags for t in ['pasta','pizza','risotto','italienisch']):
            required['italienisch'] = True
        if any(t in tags for t in ['deutsch','schnitzel','gulasch','frikadelle','eintopf','braten','kartoffel']):
            required['deutsch-klassisch'] = True
        if rezept.get('subkategorie') == 'vegetarisch' or 'vegetarisch' in tags:
            required['vegetarisch'] = True
        if any(t in tags for t in ['burger','hot-dog','hotdog']):
            required['burger-hotdog'] = True
    fehlend = [k for k, v in required.items() if not v]
    return fehlend  # Leere Liste = alle Slots belegt
```

---

## WIEDERHOLUNGS-SCHUTZ (NEU v2.2)

### Regel:
Ein Rezept darf in der aktuell geplanten Woche NICHT vorkommen, wenn es in der **unmittelbar vorangegangenen Woche** bereits verwendet wurde.

### Umsetzung:
```python
def get_vorwoche_rezepte(db_path, aktuelle_kw, variante='aktuell'):
    """Gibt Set von rezept_ids der Vorwoche zurueck."""
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute("""
        SELECT DISTINCT rezept_id FROM wochenplan
        WHERE woche = ? AND variante = ?
    """, (aktuelle_kw - 1, variante))
    ids = {row[0] for row in cur.fetchall()}
    conn.close()
    return ids

def filter_rezepte_wiederholung(rezepte_liste, vorwoche_ids):
    """Entfernt Rezepte die letzte Woche bereits verwendet wurden."""
    return [r for r in rezepte_liste if r['id'] not in vorwoche_ids]
```

### Wichtig:
- Wenn ein Rezept in Vorwoche UND einzige Option fuer einen Pflicht-Slot → ausnahmsweise erlaubt, aber als "(Wiederholung)" im PDF markieren
- Wochenplan-Verlauf bleibt dauerhaft in der DB (nicht loeschen!) → Basis fuer Jahres-Analyse

---

## NEUE REZEPTE: ONLINE-RECHERCHE (NEU v2.2)

### Wann neue Rezepte suchen:
- User sagt explizit "such mir ein Rezept fuer ..." oder "neues Rezept fuer Slot [X]"
- Ein Pflicht-Slot (Italienisch/Deutsch/Vegetarisch/Burger) hat kein passendes Rezept in der DB
- Angebots-Produkt in DB hat kein passendes Rezept (z.B. neues Gemuese im Angebot)

### Recherche-Quellen (in dieser Prioritaet):
1. **Chefkoch.de** — Suche mit `web_search`: `site:chefkoch.de [Zutat] [Stil] einfach`
2. **Reddit communities**: `r/kochen`, `r/einfacheRezepte`, `r/budgetkochen`
3. **Lebensmittelmarkt-Blogs**: Aldi-Rezepte, Lidl-Rezepte (Marken kochen mit eigenen Angeboten)
4. **Lecker.de**, **kochbar.de** als Fallback

### Recherche-Kriterien (Rezept muss bestehen):
- Zubereitung max 30 Min (Mittag) oder 40 Min (Abend)
- Zutaten in deutschen Supermärkten verfuegbar (kein Exoten-Zeug)
- Kein Tofu, kein Thunfisch, keine Susskartoffel
- Budget: Zutaten fuer 2P max 3,50 EUR
- Rike (13 J.) muss es moeglicherweise essen wollen — kein ultra-scharfes, nichts Exotisches ohne Anpassung
- Gemuese gerne als TK-Variante nutzbar

### Neues Rezept in DB speichern:
```python
import json

def add_neues_rezept(db_path, rezept_dict):
    """
    rezept_dict Pflichtfelder:
      name, kategorie (mittag/abend), subkategorie, zutaten_json (JSON-String),
      schritte_json (JSON-String), kcal, protein_g, kosten_2p_eur,
      zubereitungszeit_min, tags
    Optional: hinweis, tipp, brotsorte, kosten_1p_eur
    """
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute("""
        INSERT INTO rezepte
        (name, kategorie, subkategorie, zutaten_json, schritte_json,
         kcal, protein_g, kosten_2p_eur, kosten_1p_eur,
         zubereitungszeit_min, hinweis, tipp, ausschluss_frei, tags)
        VALUES (?,?,?,?,?,?,?,?,?,?,?,?,1,?)
    """, (
        rezept_dict['name'], rezept_dict['kategorie'],
        rezept_dict['subkategorie'],
        json.dumps(rezept_dict['zutaten'], ensure_ascii=False),
        json.dumps(rezept_dict['schritte'], ensure_ascii=False),
        rezept_dict.get('kcal'), rezept_dict.get('protein_g'),
        rezept_dict.get('kosten_2p_eur'), rezept_dict.get('kosten_1p_eur'),
        rezept_dict.get('zubereitungszeit_min'),
        rezept_dict.get('hinweis'), rezept_dict.get('tipp'),
        rezept_dict['tags']
    ))
    new_id = cur.lastrowid
    conn.commit()
    conn.close()
    return new_id
```

### Neue Rezept-Tags fuer Pflicht-Slots:
```
Slot I Italienisch:  tags muss enthalten: 'italienisch' ODER ('pasta' ODER 'pizza' ODER 'risotto')
Slot II Deutsch:     tags muss enthalten: 'deutsch' ODER 'schnitzel' ODER 'frikadelle' ODER 'gulasch' ODER 'eintopf'
Slot III Vegetarisch: subkategorie='vegetarisch' ODER tags enthaelt 'vegetarisch'
Slot IV Burger/HotDog: tags muss enthalten: 'burger' ODER 'hot-dog'
```

---

## ANGEBOTS-GUELTIGKEITS-REGEL (KRITISCH)

**Ein Angebot darf NUR fuer Planung verwendet werden, wenn `gueltig_bis >= Planungsdatum`.**

Angebote bleiben jedoch DAUERHAFT in der DB (fuer Jahres-Preis-Analyse, siehe unten).

### Planungsdatum bestimmen (Schritt 0):
- User sagt "Wochenplan fuer KW 23" oder "Plan ab 02. Juni" → Planungsdatum = 2026-06-02
- NIEMALS heute als Planungsdatum wenn User zukuenftige Woche meint

### Gueltiger Angebots-Query:
```python
def get_gueltige_angebote(db_path, planungsdatum_iso):
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute("""
        SELECT produkt, preis_eur, einheit, kg_preis, supermarkt, kategorie,
               gueltig_von, gueltig_bis
        FROM angebote
        WHERE gueltig_bis >= ? AND relevant_fuer = 'ok'
        ORDER BY kategorie, kg_preis
    """, (planungsdatum_iso,))
    result = cur.fetchall()
    conn.close()
    return result
```

### Session-Start Pflicht-Ausgabe:
```
Planungsdatum:     02.06.2026 (KW 23)
Angebote gueltig:  X aktuell / Y abgelaufen in DB
Vorwoche-Rezepte:  [Liste der Rezepte aus KW 22 — werden gesperrt]
Pflicht-Slots:     Italienisch / Deutsch / Vegetarisch / Burger-HotDog
```

### Wenn keine gueltigen Angebote:
Skill informiert, bietet Plan mit Standardpreisen ODER wartet auf Prospekt-Einlesen.

---

## ANGEBOTS-PREIS-ARCHIV & JAHRES-ANALYSE (NEU v2.2)

### Grundprinzip:
Alle eingelesenen Angebote bleiben DAUERHAFT in der `angebote`-Tabelle.
Sie werden NICHT geloescht wenn sie ablaufen — nur beim erneuten Einlesen des GLEICHEN Marktes fuer den GLEICHEN Zeitraum (Idempotenz-Delete).

### Jahres-Analyse-Query (Ende des Jahres oder auf Anfrage):
```python
def preis_verlauf_analyse(db_path, produkt_schluessel):
    """
    Zeigt Preisverlauf eines Produkts ueber alle gespeicherten Zeitraeume.
    produkt_schluessel: Suchbegriff, z.B. "hähnchen" oder "hackfleisch"
    """
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute("""
        SELECT supermarkt, produkt, preis_eur, einheit, kg_preis,
               gueltig_von, gueltig_bis
        FROM angebote
        WHERE LOWER(produkt) LIKE ?
        ORDER BY gueltig_von
    """, (f'%{produkt_schluessel.lower()}%',))
    rows = cur.fetchall()
    conn.close()
    return rows
```

### Was man damit Ende des Jahres sehen kann:
- Rhythmus der Angebote (z.B. "Haehnchen bei Aldi jede 3. Woche im Angebot")
- Preisentwicklung (steigen Preise? Saisonale Schwankungen?)
- Welcher Markt ist konsistent guenstig fuer welche Kategorie
- Welche Monate haben die besten Fleisch-Angebote

### Trigger fuer Analyse:
"Wie haben sich die Preise entwickelt", "Preis-Analyse Hackfleisch", "Wann war Haehnchen am guenstigsten", "Jahresrueckblick Einkauf"

---

## WORKFLOW A: Prospekt einlesen (wöchentlich)

### Input-Formen:
1. Foto/Scan des Prospekts → Vision/OCR liest Produkte + Preise
2. Text-Liste → User tippt oder spricht, Skill strukturiert
3. Mischform (mehrere Maerkte in einer Session)

### Einlese-Schema:
```
supermarkt    TEXT  (z.B. "ALDI Süd", "Lidl", "Penny", "Edeka", "Hit")
gueltig_von   TEXT  (YYYY-MM-DD)
gueltig_bis   TEXT  (YYYY-MM-DD)
kategorie     TEXT  (fleisch | fisch | gemüse-tk | milch | staerke | konserve | sonstige)
produkt       TEXT  (Produktname wie im Prospekt)
detail        TEXT  (z.B. "TK 500g", "aus Deutschland")
preis_eur     REAL  (Normalpreis ohne App)
preis_app_eur REAL  (App-Preis wenn im Prospekt genannt, sonst NULL)
preis_alt     REAL  (Streichpreis / UVP, sonst NULL)
rabatt_pct    INT   (Rabatt %, sonst NULL)
einheit       TEXT  (z.B. "500g", "1kg", "Stk")
kg_preis      REAL  (Preis/kg — immer aus bestem Preis berechnen: App-Preis wenn vorhanden)
relevant_fuer TEXT  ("ok" | "schwein_kein" | "skip")
notiz         TEXT  (optional)
```

### App-Preise — welche Supermarkt-Apps es gibt:
- **EDEKA App** — App-Preise direkt im Prospekt, oft 10–20% günstiger
- **Penny App** — regelmäßige App-Exklusivpreise auf Fleisch, Milch, Grundnahrung
- **HIT App** — vereinzelte App-Preise
- **Lidl Plus App** — Coupons (kein fixer App-Preis im Prospekt, aber Rabattcoupons)
- **ALDI Nord/Süd** — keine App-Preise
- **Netto App** — App-Rabatte wenn Netto-Prospekte eingelesen werden

Beim Einlesen: Immer auf "mit App X,XX€" oder "App-Preis" im Prospekt achten → in `preis_app_eur` speichern.

### App-Preis Logik bei Budget und Kosten:
```python
def bester_preis(row):
    """Gibt (preis, ist_app_preis) zurück — immer günstigsten nehmen."""
    app = row['preis_app_eur']
    normal = row['preis_eur']
    if app is not None and (normal is None or float(app) < float(normal)):
        return float(app), True
    return float(normal) if normal else 0.0, False
```

Budget-Berechnung verwendet **immer den besten Preis**.
Einkaufsliste zeigt App-Preis mit **📱** Kennzeichnung.

### Einkaufsliste App-Format:
```
📱 = App-Preis aktiv (EDEKA App / Penny App etc. vor dem Einkauf aktivieren!)

Fleisch & Fisch
  □  Patros Grillkäse    180g   Penny    1,69€ 📱   ________
  □  Hähnchenbrust       800g   HIT      6,99€       ________
  □  Kerrygold Butter    250g   EDEKA    1,11€ 📱   ________
```

### Relevanz:
- `ok` → fuer Rezeptplanung nutzbar
- `schwein_kein` → Schwein (behalten fuer Analyse, nicht fuer Rezepte)
- `skip` → Alkohol, Non-Food, irrelevant

### Speicher-Code (mit preis_app_eur):
```python
def save_angebote(db_path, angebote_list):
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    if angebote_list:
        supermarkt = angebote_list[0]['supermarkt']
        gueltig_von = angebote_list[0]['gueltig_von']
        cur.execute(
            "DELETE FROM angebote WHERE supermarkt=? AND gueltig_von=?",
            (supermarkt, gueltig_von)
        )
    for a in angebote_list:
        cur.execute("""
            INSERT INTO angebote
            (supermarkt, gueltig_von, gueltig_bis, kategorie, produkt,
             detail, preis_eur, preis_app_eur, preis_alt, rabatt_pct,
             einheit, kg_preis, relevant_fuer, notiz)
            VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?)
        """, (a.get('supermarkt'), a.get('gueltig_von'), a.get('gueltig_bis'),
              a.get('kategorie'), a.get('produkt'), a.get('detail'),
              a.get('preis_eur'), a.get('preis_app_eur'),
              a.get('preis_alt'), a.get('rabatt_pct'),
              a.get('einheit'), a.get('kg_preis'),
              a.get('relevant_fuer','ok'), a.get('notiz')))
    conn.commit()
    conn.close()
```

---

## WORKFLOW B: Wochenplan generieren

### Schritt-fuer-Schritt:

**1. Session-Status ausgeben** (Planungsdatum, gueltige Angebote, Vorwoche-Rezepte)

**2. Pflicht-Slots besetzen** (Italienisch, Deutsch, Vegetarisch, Burger/HotDog)
   - Angebote pruefen: welche Zutat ist diese Woche guenstig?
   - Passende Rezepte aus DB holen (gefiltert: ausschluss_frei=1, nicht in Vorwoche)
   - Fehlt Rezept fuer Slot → Online-Recherche (Workflow: Neue Rezepte)

**3. MENGEN-AGGREGATION vor Restplanung (PFLICHT ab v2.5)**

BEVOR weitere Tage besetzt werden: Alle Proteinmengen der bisher gewaehlten Rezepte
aufaddieren und gegen verfuegbare Handelseinheiten pruefen.

```python
def aggregiere_protein(wochenplan_rezepte, ziel_protein):
    """
    wochenplan_rezepte: Liste der bereits gewaehlten Rezepte mit Zutaten
    ziel_protein: z.B. 'rinderhack', 'haehnchen', 'fisch'
    Gibt zurueck: Gesamtgramm benötigt, empfohlene Einkaufsmenge
    """
    KEYWORDS = {
        'rinderhack':  ['rinderhack','hackfleisch rind','hack vom weide'],
        'haehnchen':   ['hähnchenbrustfilet','haehnchen','hähnchenbrust'],
        'pute':        ['putenbrust','putenhack','putengeschnetzel'],
        'fisch':       ['seelachs','pangasius','fischfilet','lachs'],
    }
    total_gr = 0
    for r in wochenplan_rezepte:
        for z in r.get('zutaten', []):
            name = (z[1] if isinstance(z,list) else z).lower()
            menge = z[0] if isinstance(z,list) else ''
            if any(kw in name for kw in KEYWORDS.get(ziel_protein,[])):
                gr = parse_gramm(menge)
                if gr: total_gr += gr
    return total_gr
```

**HANDELSEINHEITEN-MAPPING (immer beachten):**
```
Rinderhack:       500g oder 1000g Packs → kaufe Vielfaches von 500g
Haehnchenbrust:   200g-Portion (HIT) oder 400g/800g Pack (Penny/ALDI)
Fisch TK:         380g (Iglo Schlemmerfilet) oder 450g (Fischstaebchen)
Fisch frisch:     100g-Einheiten (Theke) — flexibel aber teurer
Rahm-Spinat TK:   800g Pack → gut einfrierbar
```

**WENN benoetigte Menge > groesste Handelseinheit:**
Rezepte ANPASSEN statt zweite Packung kaufen, wenn Rest > 200g und nicht einfrierbar.
Beispiel: Brauche 1.300g Hack, kaufe 2x500g=1.000g → passe ein Rezept von 300g auf 250g an
ODER waehle ein Rezept mit weniger Hack.

**4. Resteverwertungs-Ketten (mengengenau planen):**

Kette MUSS aufgehen — d.h. alle Rezepte in der Kette verbrauchen gemeinsam
exakt eine Handelseinheit (oder Rest ist TK-faehig):

```
HACK-KETTE (1kg Pack):
  → Rezept A: 400g  (z.B. Bolognese)
  → Rezept B: 350g  (z.B. Frikadellen)
  → Rezept C: 250g  (z.B. Kotleti)
  = 1.000g exakt ✓

HAEHNCHEN-KETTE (800g Penny Pack):
  → Rezept A: 375g  (z.B. Ofen-Haehnchen)
  → Rezept B: 350g  (z.B. Burger 2 Filets)
  = 725g + 75g Rest → einfrieren ✓

FISCH TK (380g Iglo):
  → 1 Rezept: 375g  (Fischboulette)
  = 5g Rest → vernachlaessigbar ✓
```

**ANTI-PATTERN (vermeiden):**
```
FALSCH: Hack-Kette die 1.300g braucht aber nur 1kg-Pack existiert
FALSCH: Haehnchen-Rezept waehlen wenn passendes Angebot Schenkel statt Brust ist
FALSCH: Gleiches Produkt aus zwei verschiedenen Maerkten einkaufen
```

**5. Angebots-Matching-Regel (KRITISCH ab v2.5):**

Pro Protein-Zutat NUR EIN Angebot auswaehlen — das mit dem besten kg-Preis.
Matchinglogik:
- "Hähnchenbrustfilet" → matcht auf Angebote mit "hähnchen" UND "brust" bevorzugt
- "Rinderhack" → matcht auf "rinderhack" NICHT auf "hackfleisch gemischt" (Schwein!)
- Schenkel/Keule/Oberschenkel ≠ Brustfilet — explizit pruefen

```python
def finde_bestes_angebot(zutat_name, angebote):
    """Gibt einziges bestes Angebot zurueck (niedrigster kg-Preis)."""
    keywords = zutat_name.lower().split()
    keywords = [k for k in keywords if len(k)>3 and k not in
                ['oder','und','mit','aus','von','aufgetaut','kein','keine']]
    treffer = []
    for ang in angebote:
        p = ang['produkt'].lower()
        score = sum(1 for kw in keywords if kw in p)
        if score > 0:
            treffer.append((score, float(ang.get('kg_preis') or 999), ang))
    if not treffer:
        return None
    # Sortiere: erst Score (hoeher=besser), dann kg-Preis (niedriger=besser)
    treffer.sort(key=lambda x: (-x[0], x[1]))
    return treffer[0][2]
```

**6. Budget-Check pro Tag:**
```python
def check_budget(mittag_kosten, abend_kosten, budget=8.50):
    gesamt = (mittag_kosten or 0) + (abend_kosten or 0) + 1.50
    return {'ok': gesamt <= budget, 'gesamt': gesamt, 'rest': budget - gesamt}
```

**7. Wochenplan in DB speichern** (Verlauf fuer Wiederholungs-Schutz + Analyse)

---

## WORKFLOW C: Einkaufsliste

### Prinzip (v2.5 — aggregiert):
- Alle Zutaten ALLER Wochenrezepte zuerst summieren (Gramm aufaddieren)
- Pro Zutat NUR EIN Angebot auswaehlen (bester kg-Preis)
- Handelseinheit berechnen: wie viele Packungen, was bleibt uebrig
- Nach Supermarkt sortiert
- Grundvorrat wird als vorhanden angenommen

### Grundvorrat (nicht auf Liste):
Olivenoel, Salz, Pfeffer, Knoblauch, Zwiebeln, Tomatenmark, Curry, Paprika gemahlen,
Kurkuma, Oregano, Spaghetti/Fusilli/Penne (Reserve), Basmati-Reis, Kokosmilch Dose,
Gehackte Tomaten, Honig, Senf, Worcestersauce, Mehl, Bruehepulver, Sojasosse

### Listen-Format (v2.5):
```
[MARKT] (App-Preis verfuegbar? → App aktivieren!)
  □  Produkt              2× 500g   benoetigt ~1.300g    10,49€
                          Rest ~200g → einfrieren 🧊
                          fuer: Di Kotleti · Mi Bolo · Do Vorkoch · Fr Frikadellen
  □  ...

♻️ Reste diese Woche:
  Rinderhack ~200g → 🧊 einfrieren
  Haehnchen ~75g  → 🧊 einfrieren

📊 Budget-Check:
  Einkauf gesamt: XX,XX€ · Budget: 42,50€ · Differenz: +/-X,XX€
```

---

## WORKFLOW C.2: Rike Nachtisch & Süßes (PFLICHT bei jeder neuen Woche)

### Regel:
Bei jeder neuen Wochenplanung werden 3 Nachtisch-Kombis fuer Rike vorgeschlagen.
Budget: 5-8 EUR extra (nicht im Mahlzeiten-Budget enthalten).
NUR fuer Rike — Papa bekommt keinen Nachtisch eingeplant.

### Vorgehensweise:
1. Aktuelle Prospekte durchsuchen nach: Eis, Schokolade, Pudding, Gummibärchen,
   Waffeln, Crêpes, Riegel, Joghurt-Desserts, Kekse
2. 3 Kombis zusammenstellen mit unterschiedlichem Charakter:
   - KOMBI A: günstigste Option (~5€), z.B. Pudding + Schoko + 1 Packung Süßes
   - KOMBI B: Eis-Fokus (~6€), z.B. 2 verschiedene Eissorten = 7-8 Eis/Woche
   - KOMBI C: Abwechslungs-Mix (~7-8€), z.B. Eis + Pudding + Gummibärchen + Waffeln + Riegel
3. User waehlt eine Kombi
4. Gewaehlte Kombi wird in `data.json` als `extras_einkauf` gespeichert
5. Erscheint in der Einkaufsliste als eigener Block "🍬 Rike Nachtisch"
6. Taucht NICHT im Wochenplan/Tagesmenü auf

### data.json Format:
```json
"extras_einkauf": [
  {"markt": "Lidl",  "produkt": "Kuchenzauber Waffeln XXL 335g", "preis": 1.79, "fuer": "Rike Nachtisch"},
  {"markt": "Netto", "produkt": "Nimm2 Lachgummi 325g",          "preis": 1.00, "fuer": "Rike Nachtisch"}
]
```

### Typische Produkte die Rike mag (aus bisherigen Wochen):
- Kinder Maxi King, Kinder Bueno Eis
- Paula Pudding, Grand Dessert
- Nimm2 Lachgummi, Haribo
- Milka Schokolade
- Mövenpick Eis, Langnese Eis
- Kuchenzauber Crêpes/Waffeln
- Monte Mega

---

## WORKFLOW D: PDF ausgeben

### PDF-Style: pdf-style Skill lesen vor Generierung.

### Seiten-Struktur:
1. Titelseite "Wochenplan KW XX — Rike + Papa" + Budget-Badge + Pflicht-Slot-Badges
2. Uebersicht-Tabelle (7 Tage x 2 Mahlzeiten, Kochdauer, Kosten, Slot-Icons)
3. Angebots-Highlights diese Woche
4. Rezeptseiten pro Tag (Mittag + Abend, Zutaten + Schritte)
5. Resteverwertungs-Plan
6. Einkaufsliste (nach Markt sortiert, Preisspalte)

### Farb-Code:
- Gelb  (#FFF8E1 / #FFC107) = Kleines Mittag 14:30
- Blau  (#E1F5FE / #0288D1) = Grosses Abend 18:00
- Gruen (#E8F5E9 / #388E3C) = Angebots-Highlight / Guenstig
- Orange (#FFEBE0 / #E65100) = Wochenende / Sonder / Burger-HotDog

---

## AUSSCHLUESSE (unveraenderlich)

NIEMALS — kommt in keinem Rezept vor, wird nicht vorgeschlagen, nicht substituiert:
- Tofu
- Thunfisch (Quecksilber)
- Susskartoffel
- Omelett als Mahlzeit
- Reiner Reis ohne Kokosmilch
- **Pilze** — weder frisch, noch TK, noch aus der Dose, in keiner Verarbeitungsform
- **Schweinehackfleisch** — ABSOLUTES VERBOT. Kein gemischtes Hack, kein reines Schweinehack.
  Hack immer aus Rind, Pute oder Haehnchen.

SCHWEIN-REGELUNG (differenziert):
- **Schweinefleisch als Braten/Schnitzel/Steak** → kein bevorzugter Kauf, nicht vorschlagen
- **Schweinewurst (Bratwurst, Wiener, Thüringer)** → Notfall-Option wenn KEINE Geflügelwurst
  verfuegbar ist. IMMER mit Hinweis: "1. Wahl Geflügel, notfalls Schweinewurst ok"
- **Schweinespeck/Schinken/Salami** → nicht vorschlagen, kein Bedarf in den Rezepten
- **Schweinehackfleisch** → NIEMALS, keine Ausnahme

Bei Burgern/Hot Dogs:
- 1. Wahl: Haehnchenbratwurst / Gefluegelwurst / Rinder-Patty
- Notfall: Schweinewurst/Bratwurst vom Schwein mit Hinweis
- NIEMALS: Schweinehack als Patty

Bevorzugt:
- TK-Gemuese (Brokkoli, Spinat, Erbsen, Bohnen, Mais, Karotten)
- Haehnchen, Pute, Rind, Magerquark, Lachs, Fischstaebchen, Haehnchen-Bratwurst
- Hülsenfrüchte (Linsen, Bohnen, Erbsen) als guenstige Protein-Basis
- Vollkorn-Pasta, Kokos-Reis, Kartoffeln

---

## KAESE-REGEL (Rike mag Kaese / Papa mag ohne)

**Grundregel:** Jedes Rezept mit Kaese MUSS ohne zusaetzlichen Aufwand auch ohne Kaese funktionieren.

### Kochpraxis:
- Kaese wird IMMER separat gerieben/aufgelegt — nie ins Gericht eingeruehrt oder ueberbacken wenn das den Papa ausschliesst
- Ausnahme: Rezept wo Kaese integral ist (z.B. Caprese-Salat) → dann als reines Rike-Gericht kennzeichnen und Papa bekommt Variante ohne

### Im PDF:
- Rezepte mit Kaese-Option bekommen den Hinweis: **"Kaese separat — Papa ohne"**
- Zutaten-Liste: Kaese mit Sternchen `*` markieren + Fussnote "* nur fuer Rike"

### Beim Kochen:
- Sauce/Basis fuer beide gleich kochen
- Rike's Portion → Kaese drauf oder drunter
- Papa's Portion → ohne
- Gilt fuer: Pasta (Parmesan/Reibekaese), Burger (Scheiben-Kaese), Ofengerichte (Gratinierung), Taco/Wrap

### Tags in der DB:
- `käse-trennbar` → bereits vorhanden, bedeutet genau das
- Neue Rezepte: wenn Kaese vorkommt → immer `käse-trennbar` als Tag setzen

### Was NICHT geht (Rezepte meiden oder anpassen):
- Kaese-Fondue (Kaese ist das Gericht selbst)
- Lasagne mit Bechamel-Kaese-Schicht (Kaese integral — nur wenn Papa-Variante machbar)
- Kaesespätzle (Kaese integral) → nur als Rike-Solo-Gericht, Papa bekommt Spätzle mit Sauce

---

## ALTERS-SPEZIFISCHES: RIKE 13 J. PUBERTAET

- Eisen hoch: Rind, Hülsenfrüchte, Spinat + Vitamin C
- Kalzium hoch: Kaese, Milch, Joghurt
- Protein: mind. 45-50g/Tag
- Energie: ~2.000-2.200 kcal/Tag
- Rike mag: Kaese, Burger, Pizza, Suesses — als Motivation einsetzen
- Kein Verbot, aber ausgewogene Vielfalt durch Pflicht-Slots automatisch sichergestellt

---

## DB-DATEI MANAGEMENT

1. User laedt `ernaehrung.db` hoch → `cp /mnt/user-data/uploads/ernaehrung.db /home/claude/ernaehrung.db`
2. Skill liest/schreibt in `/home/claude/ernaehrung.db`
3. Finale Version → `cp /home/claude/ernaehrung.db /mnt/user-data/outputs/ernaehrung.db`
4. User laedt aktualisierte DB herunter

**Angebote und Wochenplaene NIEMALS loeschen** (Langzeit-Analyse-Basis).

---

## WEB-APP DEPLOYMENT & CACHE (nach Papa-Kalender-Vorbild)

### Repo: `rezepte-schulz` auf GitHub Pages

Dateien im Repo:
```
index.html      — die App (Views: Wochenplan, Einkaufen, Angebote, Rezepte)
data.json       — alle Daten (Rezepte, Angebote, Wochenplan, Extras)
sw.js           — Service Worker mit VERSION-Konstante
```

### Service Worker Versionierung (wie Papa-Kalender):
`sw.js` hat eine `VERSION`-Konstante, z.B. `'kuechen-v9'`. Bei JEDER neuen
Veroeffentlichung von index.html diese Zahl hochzaehlen, damit Clients
beim naechsten Oeffnen die neue Version laden statt die alte aus dem Cache.

### Cache-Zwang (iPhone + Desktop):
Die index.html erzwingt Neuladen durch drei Mechanismen:
1. **Meta-Tags**: `Cache-Control: no-cache, no-store, must-revalidate`
2. **SW sofort-Update**: `reg.update()` bei jedem App-Start
3. **Fokus-Update**: `visibilitychange`-Event — wenn App aus Hintergrund kommt,
   wird SW-Update geprueft und bei Bedarf Seite neu geladen

### data.json Cache-Bust:
```javascript
fetch('data.json?v=' + Date.now(), { cache: 'no-store' })
```
Timestamp-Parameter + no-store = Browser kann sie NIE aus Cache nehmen.

### Wochenplan-Anzeige:
Zeigt immer die **neueste Woche** (hoechste `woche`-Nummer in `wochenplan`).
Alte Wochen bleiben in data.json fuer Verlauf, werden aber nicht angezeigt.

### Tagesreihenfolge:
Mittagessen wird immer VOR Abendessen angezeigt (sort: mittag < abend).

### Einkaufsliste:
Generiert aus Zutaten der neuesten Woche + `extras_einkauf` (Rike Nachtisch).
Aggregiert Mengen, zeigt Packungsanzahl, Reste, Budget-Check.

### Deployment-Checkliste bei neuer Woche:
1. `data.json` im Repo ersetzen (neue Woche, neue Angebote)
2. `index.html` nur ersetzen wenn Code-Aenderungen
3. `sw.js` VERSION hochzaehlen wenn index.html geaendert
4. Nach 1-2 Min auf GitHub Pages live

---

## QUALITAETS-CHECKLISTE VOR EXPORT

- [ ] Planungsdatum und Angebots-Gueltigkeit geprueft?
- [ ] Alle 4 Pflicht-Slots belegt (Italienisch / Deutsch / Vegetarisch / Burger-HotDog)?
- [ ] Keine Rezepte aus der Vorwoche wiederholt?
- [ ] Budget eingehalten?
- [ ] TK-Gemuese bevorzugt wo moeglich?
- [ ] Resteverwertungs-Ketten mengengenau geplant?
- [ ] Mengen-Aggregation geprueft (Protein-Ketten passen auf Handelseinheiten)?
- [ ] Neue Rezepte korrekt in DB gespeichert (mit Tags fuer Pflicht-Slot)?
- [ ] Keine Ausschluss-Produkte (Pilze, Schweinehack, Tofu, Thunfisch)?
- [ ] Rike-Naehrstoffe abgedeckt (Eisen, Kalzium, Protein)?
- [ ] Einkaufsliste nach Markt sortiert?
- [ ] **3 Nachtisch-Kombis fuer Rike vorgeschlagen?**
- [ ] **extras_einkauf in data.json eingetragen?**
- [ ] Mittagessen vor Abendessen in Tagesreihenfolge?
- [ ] sw.js VERSION hochgezaehlt wenn index.html geaendert?
- [ ] Kein "Trading" im PDF?

---

## VERSION

- **v2.6 — 2026-05-29** — Rike Nachtisch-Workflow (3 Kombis bei jeder neuen Woche, extras_einkauf in data.json, pinker Block in Einkaufsliste). Web-App Cache-Zwang dokumentiert (Meta-Tags, SW sofort-Update, visibilitychange). Tagesreihenfolge: Mittag vor Abend. Schweinewurst als Notfall-Option bei Würsten. sw.js v9.
- **v2.5 — 2026-05-27** — Mengen-Aggregation, Handelseinheiten-Mapping, mengengenaue Resteverwertungs-Ketten, verbessertes Angebots-Matching. Web-App v4 mit aggregierter Einkaufsliste, Packungsanzahl, Reste-Block, Budget-Check.
- **v2.4 — 2026-05-26** — App-Preise: preis_app_eur in DB, Budget nutzt besten Preis, Einkaufsliste zeigt 📱.
- **v2.3 — 2026-05-26** — Pilze und Schweinefleisch ausgeschlossen. Kaese-Regel: Papa ohne / Rike mit.
- **v2.2 — 2026-05-26** — Pflicht-Slots, Wiederholungs-Schutz, Online-Recherche, Preis-Archiv.
- **v2.1 — 2026-05-26** — Angebots-Gueltigkeits-Regel strikt erzwungen.
- **v2.0 — 2026-05-26** — Neufassung: ernaehrung.db-Integration, Prospekt-Workflow, Budget, Resteverwertung.
- **v1.x** — Vorgaenger: rezeptbuch Skill (nur PDF, keine DB).
