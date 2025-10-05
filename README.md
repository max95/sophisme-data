# sophisme-data

**Générateur de “spurious data” (sophismes) à partir de l’actualité + données publiques**

> ⚠️ Projet satirique et pédagogique : ce dépôt montre *comment* on peut fabriquer une conclusion **fausse** à partir de **données vraies** en combinant mal des indicateurs (mauvais dénominateur, fenêtre temporelle biaisée, unités hétérogènes, etc.). Chaque sortie comporte des **drapeaux rouges** et une **lecture correcte** pour l’éducation aux médias.

---

## Pourquoi

* Transformer un titre d’actu (RSS) + 2 jeux de données publics (INSEE, SSMSI, Banque de France, Eurostat, etc.) en **“histoire” volontairement biaisée**.
* **Expliquer** en quoi le raisonnement est fallacieux → fournir la **version correcte**.
* Alimenter des contenus courts (X/Threads, Shorts/TikTok, "Radio Londres" codée) **avec garde‑fous**.

---

## Fonctions clés

* **3 stratégies prêtes** :

  * *wrong_denominator* : absolu vs per capita / périmètres non homogènes.
  * *time_window_bias* : cherry‑picking d’une fenêtre temporelle “qui arrange”.
  * *mixed_units* : euros courants vs constants, journalier vs annuel, etc.
* **Sortie structurée** (`JSON`) : titre sophisme (FAUX), bullets, drapeaux rouges, correction, sources.
* **Branchable** à un dashboard (Gradio) et à un pipeline RSS (Franceinfo, etc.).

---

## Aperçu (JSON de sortie)

```json
{
  "topic": "Tempête Amy vs mortalité routière",
  "strategy": "Mauvais dénominateur (absolu vs per capita / périmètres hétérogènes)",
  "sophism": {
    "title": "[SOPHISME] Tempête Amy vs mortalité routière : « la situation A dépasse B »",
    "bullets": ["…"],
    "tag": "FAUX / SATIRE — démonstration de biais"
  },
  "red_flags": ["Dénominateur non comparable", "…"],
  "correction": {
    "title": "Lecture correcte : comparer des métriques homogènes",
    "bullets": ["…"]
  },
  "sources": ["https://…/datasetA", "https://…/datasetB"]
}
```

---

## Architecture

```
┌───────────────────────┐       ┌────────────────────────┐
│ Flux actu (RSS)       │       │ Jeux de données publics│
│  titres, dates, liens │       │  INSEE / SSMSI / BdF…  │
└──────────┬────────────┘       └─────────────┬──────────┘
           │                                   │
           ▼                                   ▼
     Sélection d’un                   Sélection de 2+ datasets
     sujet/“topic”                    et des colonnes clés
           │                                   │
           └──────────────┬────────────────────┘
                          ▼
                 **sophism_generator.py**
             (stratégie de biais + paramètres)
                          │
                          ▼
                 Sortie : JSON + texte
             (sophisme FAUX + flags + fix)
```

---

## Pré‑requis

* Python **3.11+**
* macOS / Linux / WSL

### Dépendances

```
pandas>=2.0
numpy>=1.25
```

*(Gradio facultatif pour l’UI, pytest pour les tests.)*

---

## Installation

```bash
# 1) Cloner
git clone https://github.com/votre-org/sophisme-data.git
cd sophisme-data

# 2) Environnement
python -m venv .venv && source .venv/bin/activate

# 3) Dépendances
pip install -r requirements.txt
```

`requirements.txt` minimal :

```
pandas>=2.0
numpy>=1.25
```

---

## Données d’exemple

`data/tempete_amy.csv`

```csv
date,value
2025-10-03,2
2025-10-04,4
2025-10-05,3
```

`data/deces_route.csv`

```csv
date,value
2025-09-25,9
2025-09-26,8
2025-09-27,7
2025-09-28,6
2025-09-29,8
2025-09-30,7
2025-10-01,5
2025-10-02,6
2025-10-03,7
2025-10-04,5
```

---

## Utilisation (CLI)

> Le script central est `sophism_generator.py`.

### A. Mauvais dénominateur (fabrique volontairement un faux message “A > B”)

```bash
python sophism_generator.py \
  --topic "Tempête Amy vs mortalité routière" \
  --A.path data/tempete_amy.csv --A.name "Victimes liées à la tempête (zone locale)" --A.value value --A.date date --A.unit "décès" --A.source "https://data.gouv.fr/..." \
  --B.path data/deces_route.csv --B.name "Décès routiers France" --B.value value --B.date date --B.unit "décès/jour" --B.source "https://www.onisr.securite-routiere.gouv.fr/..." \
  --strategy wrong_denominator
```

### B. Fenêtre temporelle biaisée (cherry‑pick 7 jours)

```bash
python sophism_generator.py \
  --topic "Sécurité : ça explose (ou pas)" \
  --A.path data/deces_route.csv --A.name "Décès routiers" --A.value value --A.date date \
  --B.path data/deces_route.csv --B.name "Décès routiers (référence)" --B.value value --B.date date \
  --strategy time_window_bias --params '{"window_days":7}'
```

### C. Unités mélangées (nominal vs réel)

```bash
python sophism_generator.py \
  --topic "Pouvoir d'achat vs prix du billet" \
  --A.path data/prix_billet.csv --A.name "Billet promo (jour J)" --A.value value --A.date date --A.unit "€ courants" \
  --B.path data/revenu_median.csv --B.name "Revenu médian annuel" --B.value value --B.date date --B.unit "€ constants (2015=100)" \
  --strategy mixed_units
```

---

## Utilisation (Python)

```python
from sophism_generator import DatasetSpec, run_sophism

payload = run_sophism(
  topic="Tempête Amy vs mortalité routière",
  dsA=DatasetSpec(name="Victimes tempête", path="data/tempete_amy.csv", value_col="value", date_col="date", unit="décès"),
  dsB=DatasetSpec(name="Décès route FR", path="data/deces_route.csv", value_col="value", date_col="date", unit="décès/jour"),
  strategy_id="wrong_denominator",
)
print(payload["sophism"]["title"])  # [SOPHISME] …
```

---

## Connecteurs (optionnels)

* **INSEE** : séries locales (population, revenus, emploi).
* **SSMSI** : faits constatés, vols, VIF, etc.
* **Banque de France (SDMX)** : inflation, taux, crédit.
* **Eurostat** : comparaisons UE (PPA, chômage, prix).

> Implémentez des loaders dédiés (HTTP→CSV/Parquet) ou stockez en SQLite puis pointez `--A.path` / `--B.path` vers la table/SQL.

---

## Intégration UI (Gradio)

```python
import gradio as gr
from sophism_generator import DatasetSpec, run_sophism

def gen(topic, a_path, a_name, a_date, a_val, a_unit, b_path, b_name, b_date, b_val, b_unit, strategy, params):
    dsA = DatasetSpec(a_name, a_path, value_col=a_val, date_col=a_date, unit=a_unit)
    dsB = DatasetSpec(b_name, b_path, value_col=b_val, date_col=b_date, unit=b_unit)
    out = run_sophism(topic, dsA, dsB, strategy, params or {})
    txt = f"""# {out['sophism']['title']}

**Drapeaux rouges**\n- " + "\n- ".join(out['red_flags']) + "\n\n**Lecture correcte**\n- " + "\n- ".join(out['correction']['bullets']) + "\n\n**Sources**\n- " + "\n- ".join(out['sources'])
    return txt

with gr.Blocks() as demo:
    # … champs …
    pass

demo.launch()
```

---

## Structure du dépôt

```
sophisme-data/
├─ data/                     # jeux d'exemple
├─ src/                      # (si vous factorisez en package)
├─ scripts/                  # CLI utilitaires
├─ sophism_generator.py      # script principal (fourni)
├─ dashboard_gradio.py       # (optionnel) UI
├─ tests/                    # tests unitaires
├─ requirements.txt
├─ README.md
└─ LICENSE
```

---

## Roadmap

* [ ] Connecteurs natifs INSEE / Eurostat / BdF (SDMX).
* [ ] +3 stratégies (Simpson, base rate neglect, cherry-pick géo).
* [ ] Export auto drafts (X/Threads, Shorts, Radio Londres).
* [ ] Paquet Python (`pip install sophisme-data`).

---

## Contribuer

Issues & PRs bienvenues. Merci d’ajouter **tests** + **exemples minimaux**.

---

## Licence

À votre choix (recommandé : **MIT**). Voir `LICENSE`.

---

## Avertissement éthique

Ce projet illustre des **biais** et **manipulations**. Les sorties sont taguées **FAUX / SATIRE** et incluent la **lecture correcte**. N’utilisez pas ce dépôt pour tromper ou désinformer.
