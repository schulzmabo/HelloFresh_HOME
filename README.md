# Küchenwoche — Rike & Papa 🍽

Mobile Web-App für den wöchentlichen Mahlzeitenplan.  
Gleiche Struktur wie der Papa-Kalender — statisches HTML + JSON auf GitHub Pages.

## Setup GitHub Pages

1. Neues Repo anlegen: z.B. `kuechen-schulz`
2. Beide Dateien hochladen: `index.html` + `data.json`
3. Settings → Pages → Branch: `main` → `/ (root)` → Save
4. URL: `https://schulzmabo.github.io/kuechen-schulz/`

## data.json aktualisieren

Die `data.json` wird durch den **ernaehrung Skill** in Claude erzeugt — immer wenn:
- Neuer Wochenplan generiert wurde
- Neue Prospekte eingelesen wurden
- Neue Rezepte hinzugekommen sind

Workflow:
1. `ernaehrung.db` bei Claude hochladen
2. Skill generiert neue `data.json`
3. `data.json` im GitHub Repo ersetzen (Upload oder git push)
4. Fertig — die App zeigt sofort die neuen Daten

## Views (URL-Parameter)

| URL | Inhalt |
|-----|--------|
| `?view=wochenplan` | Wochenplan mit Rezept-Links |
| `?view=einkaufen`  | Einkaufsliste mit App-Preisen |
| `?view=angebote`   | Alle Markt-Angebote |
| `?view=rezepte`    | Rezeptbuch mit Filter |

## Regeln

- Kein Schwein, keine Pilze
- Käse: Rike mit 🧀, Papa ohne — immer getrennt
- Budget: 8,50 €/Tag für 2 Personen
- 📱 = App-Preis (EDEKA App / Penny App aktivieren!)
