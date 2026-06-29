# Rapport de Pre-processing et Feature Engineering
## World Happiness Report — Étapes 1 & 2
**Membre A : Lead Data Preparation & Architecture**

---

## 1. Présentation du jeu de données

| Caractéristique | Valeur |
|---|---|
| Fichier source | `World-happiness-report-updated_2024.csv` |
| Dimensions brutes | 2 363 lignes × 11 colonnes |
| Pays couverts | 165 pays |
| Période temporelle | 2005 – 2023 (19 années) |
| Structure | Données de panel (pays × année) |
| Variable cible | `Life Ladder` (score de bonheur, échelle de 1 à 8) |

Le dataset est issu du Gallup World Poll et constitue la base officielle du **World Happiness Report**. Il capture 9 indicateurs socio-économiques et subjectifs par pays et par année. Sa structure de **panel non-cylindrique** (tous les pays n'ont pas de données pour chaque année) est une caractéristique importante à prendre en compte lors du pré-traitement.

---

## 2. Profiling du dataset brut

### 2.1 Qualité des données

**Doublons** : Aucun doublon détecté sur la clé (pays, année). Le dataset est structurellement propre.

**Valeurs manquantes** : La variable cible `Life Ladder` ne présente aucun manquant (0%). Les autres variables présentent des taux faibles mais non négligeables :

| Variable | Manquants | Taux |
|---|---|---|
| Perceptions of corruption | 125 | 5.29% |
| Generosity | 81 | 3.43% |
| Healthy life expectancy at birth | 63 | 2.67% |
| Freedom to make life choices | 36 | 1.52% |
| Log GDP per capita | 28 | 1.18% |
| Positive affect | 24 | 1.02% |
| Negative affect | 16 | 0.68% |
| Social support | 13 | 0.55% |

Le taux global de manquants est de **1.8%**, ce qui est faible mais non nul.

**Outliers notables** :
- `Healthy life expectancy at birth` (Haïti, 2006) : valeur = **6.72 ans**, incohérente avec la réalité (l'espérance de vie en bonne santé en Haïti était d'environ 52 ans à cette période). Les valeurs 2008 (17.36) et 2010 (28.0) confirment une anomalie de saisie progressive dans les données sources. Ces valeurs ont été conservées et traitées par interpolation.
- `Generosity` peut prendre des valeurs négatives (résidus après contrôle du PIB), ce qui est normal selon la méthodologie WHR.

### 2.2 Statistiques clés

| Variable | Min | Max | Moyenne | Médiane | Écart-type |
|---|---|---|---|---|---|
| Life Ladder | 1.281 | 8.019 | 5.484 | 5.449 | 1.126 |
| Log GDP per capita | 5.527 | 11.676 | 9.400 | 9.503 | 1.152 |
| Social support | 0.228 | 0.987 | 0.809 | 0.834 | 0.121 |
| Healthy life expectancy | 6.72 | 74.6 | 63.4 | 65.1 | 6.84 |
| Freedom | 0.228 | 0.985 | 0.750 | 0.771 | 0.139 |
| Generosity | -0.340 | 0.700 | 0.000 | -0.022 | 0.161 |
| Perceptions of corruption | 0.035 | 0.983 | 0.744 | 0.798 | 0.185 |
| Positive affect | 0.179 | 0.884 | 0.652 | 0.663 | 0.106 |
| Negative affect | 0.083 | 0.705 | 0.273 | 0.262 | 0.087 |

L'hétérogénéité des échelles est flagrante : `healthy_life_expectancy` couvre [6.7, 74.6] ans tandis que les indicateurs socio-économiques sont bornés dans [0, 1] et le PIB log est sur [5.5, 11.7]. Ce constat justifie directement la normalisation.

---

## 3. Gestion des valeurs manquantes

### Choix de méthode : interpolation temporelle intra-pays

**Pourquoi ne pas supprimer les lignes manquantes ?**
Le taux global de 1.8% est faible. Supprimer les lignes concernées réduirait la couverture géographique et temporelle du dataset, ce qui serait préjudiciable pour les analyses de tendances et les visualisations.

**Pourquoi ne pas imputer par la moyenne/médiane globale directement ?**
Les données sont structurées en **panel temporel** : les valeurs d'un pays suivent une trajectoire cohérente d'une année à l'autre. Imputer avec la moyenne mondiale ignorait cette structure et introduirait un biais (ex : imputer la moyenne mondiale pour le PIB afghan autour de 2008 serait très éloigné de la réalité).

**Stratégie choisie — 3 passes successives :**

1. **Interpolation linéaire intra-pays** (Passe 1) : pour chaque pays, on interpole linéairement entre les valeurs connues. C'est la méthode la plus précise pour les séries temporelles car elle préserve les tendances chronologiques du pays.

2. **Forward fill / Backward fill intra-pays** (Passe 2) : pour les valeurs manquantes en début ou fin de série temporelle d'un pays (non couvertes par l'interpolation), on propage la valeur connue la plus proche.

3. **Médiane globale par colonne** (Passe 3) : pour les très rares cas où un pays n'a aucune observation dans une colonne donnée. La **médiane** est préférée à la moyenne car certaines variables présentent des distributions asymétriques ou des outliers (ex : `Perceptions of corruption` a une distribution étalée à gauche).

**Résultat** : 0 valeur manquante résiduelle sur les variables numériques, sans aucune suppression de ligne.

---

## 4. Normalisation et Standardisation

### Pourquoi normaliser ?

L'hétérogénéité des échelles crée deux problèmes concrets :
- **Visuellement** : dans un radar chart ou une heatmap comparative, `healthy_life_expectancy` (0–74) écrase tous les autres indicateurs (0–1 ou 0–11).
- **Statistiquement** : certains algorithmes de machine learning (k-means, ACP) sont sensibles aux échelles et favoriseraient mécaniquement les variables à forte variance absolue.

### Deux jeux de colonnes normalisées créés

**A. Normalisation Min-Max `[0, 1]`** — suffix `_minmax`

$$x_{norm} = \frac{x - x_{min}}{x_{max} - x_{min}}$$

- **Usage** : visualisations comparatives (radar charts de pays, heatmaps), indicateurs agrégés dans les dashboards Power BI / Tableau.
- **Avantage** : interprétation directe — 0 = minimum mondial, 1 = maximum mondial.
- **Limite** : sensible aux outliers extrêmes (un seul pays au minimum ou maximum tire toute l'échelle).

**B. Standardisation Z-score `(µ=0, σ=1)`** — suffix `_zscore`

$$z = \frac{x - \mu}{\sigma}$$

- **Usage** : analyses statistiques (ACP, clustering, corrélation partielle), input pour des modèles de machine learning.
- **Avantage** : interprétation en termes d'écarts-types par rapport à la moyenne mondiale. Ex : z = +2 signifie "2 écarts-types au-dessus de la moyenne mondiale pour cet indicateur".
- **Robustesse** : meilleure gestion des distributions asymétriques.

**Note** : la variable cible `life_ladder` est **délibérément exclue** de la normalisation dans les colonnes principales pour préserver son interprétabilité (l'échelle de Cantril de 1 à 10 est compréhensible par tous les utilisateurs du dashboard).

---

## 5. Feature Engineering

Quatre nouvelles variables ont été créées pour enrichir le dataset et faciliter les analyses :

### 5.1 `happiness_category` — Catégorie ordinale de bonheur

Découpe le `life_ladder` en 5 groupes ordinals :

| Catégorie | Seuil Life Ladder | Interprétation |
|---|---|---|
| Très malheureux | < 3.5 | Pays en crise grave |
| Malheureux | 3.5 – 5.0 | Pays défavorisés |
| Modérément heureux | 5.0 – 6.0 | Pays en développement |
| Heureux | 6.0 – 7.0 | Pays développés |
| Très heureux | ≥ 7.0 | Pays nordiques, leaders |

**Utilité** : filtres dans les dashboards, encodage ordinal pour la modélisation, segmentation rapide pour la visualisation.

### 5.2 `life_ladder_yoy` — Variation annuelle (Year-over-Year)

Différence du Life Ladder d'une année à l'autre pour chaque pays. Calcul : `diff()` intra-groupe après tri par (pays, année).

**Utilité** : identifier les pays en progression ou régression rapide, détecter les effets de chocs (pandémie COVID-19 en 2020, crises politiques, guerres). NaN pour la première année observée de chaque pays — valeur intentionnelle.

### 5.3 `net_affect` — Solde émotionnel

$$\text{net\_affect} = \text{positive\_affect} - \text{negative\_affect}$$

Indicateur synthétique du bien-être émotionnel subjectif. Une valeur positive indique un équilibre favorable (plus d'émotions positives que négatives dans les expériences quotidiennes).

**Utilité** : complément au Life Ladder (qui mesure l'évaluation cognitive de la vie), `net_affect` capture la **dimension émotionnelle** du bonheur. Les deux dimensions sont distinctes et complémentaires.

### 5.4 `economic_wellbeing_score` — Score composite pondéré

$$\text{score} = 0.35 \times z(\text{gdp}) + 0.35 \times z(\text{social}) + 0.30 \times z(\text{santé})$$

Synthèse des trois principaux déterminants économico-sanitaires du bonheur, identifiés par la littérature WHR comme les plus corrélés au Life Ladder :

| Composante | Poids | Corrélation avec Life Ladder |
|---|---|---|
| Log GDP per capita | 35% | +0.78 |
| Social support | 35% | +0.72 |
| Healthy life expectancy | 30% | +0.72 |

Les versions Z-score sont utilisées pour que chaque composante contribue de manière équilibrée au score final, indépendamment de son échelle d'origine.

**Corrélation du score composite avec Life Ladder** : r ≈ 0.84, ce qui confirme sa pertinence comme indicateur synthétique.

---

## 6. Dataset nettoyé : structure finale

Le fichier `World-happiness-report-cleaned.csv` contient **2 363 lignes × 31 colonnes** :

| Groupe | Colonnes | Description |
|---|---|---|
| Identifiants | `country`, `year` | Clé du panel |
| Variables originales | `life_ladder` + 8 indicateurs | Données brutes nettoyées, renommées en snake_case |
| Normalisation Min-Max | 8 colonnes `*_minmax` | Pour les visualisations |
| Standardisation Z-score | 8 colonnes `*_zscore` | Pour les analyses statistiques |
| Features créées | `happiness_category`, `life_ladder_yoy`, `net_affect`, `economic_wellbeing_score` | Variables synthétiques |

---

## 7. Limitations et points d'attention

1. **Couverture temporelle non uniforme** : certains pays ont moins d'observations (ex : Suriname = 1 observation). Les analyses de tendances seront moins fiables pour ces pays.

2. **Anomalie Haïti** : les valeurs de `healthy_life_expectancy` en 2006-2010 sont probablement erronées dans les données sources. Elles ont été interpolées mais restent à valider avec les données WHO.

3. **Poids arbitraires** dans `economic_wellbeing_score` : les poids (35%, 35%, 30%) sont inspirés des analyses de sensibilité du WHR mais ne sont pas officiels. Pour une analyse rigoureuse, une ACP permettrait de déterminer des poids optimaux.

4. **Biais d'imputation** : pour les pays avec de nombreuses valeurs manquantes dans une colonne, l'imputation par médiane globale (Passe 3) introduit un biais vers la médiane mondiale. Ce cas est marginal mais mérite attention pour les analyses par sous-groupe.
