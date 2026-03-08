# 📊 Formation Data Dashboard — Segmentation & Pilotage d'un Centre de Formation

**Auteure :** Marie-Angélique Pied  
**Outils :** BigQuery (SQL) · Looker Studio  
**Dataset :** Kaggle Marketing Strategy — recontextualisé centre de formation  
**Dashboard en ligne :** [Voir le rapport Looker Studio](https://lookerstudio.google.com/reporting/379bf50e-c11b-4685-b97f-32b764895acb)

---

## 🎯 Objectif du projet

Simuler l'analyse de données d'un centre de formation pour répondre à 3 questions business concrètes :

- Quels profils de stagiaires génèrent le plus de valeur ?
- Quels canaux d'inscription sont réellement efficaces ?
- Quels domaines de formation sont rentables et lesquels doivent être repensés ?

---

## 🗂️ Structure du dashboard (4 pages)

**Page 1 — Tableau de bord Performance & Pilotage**  
Vue globale : segmentation Or/Argent/Bronze, canaux d'inscription par segment, évolution mensuelle.

**Page 2 — Partie Finances**  
CA par segment et par domaine de formation, budget moyen par profil de stagiaire.

**Page 3 — Top Stagiaires**  
Classement des 10 meilleurs profils par budget formation et volume de formations suivies.

**Page 4 — Insights & Recommandations**  
3 recommandations actionnables issues de l'analyse.

---

## 🔧 Étapes SQL — BigQuery

### 1. Suppression des colonnes inutiles
```sql
ALTER TABLE `mon_projet.formation_data_project.marketing_data`
DROP COLUMN Education, Marital_Status, Kidhome, Teenhome, Recency, 
acceptedcmp3, acceptedcmp4, acceptedcmp5, acceptedcmp1, acceptedcmp2, 
complain, z_costcontact, Z_revenue, Response;
-- Pour l'exécution : ajouter DROP COLUMN devant chaque nom de colonne
```

### 2. Renommage des colonnes (recontextualisation formation)
```sql
ALTER TABLE `mon_projet.formation_data_project.marketing_data`
    RENAME COLUMN ID TO stagiaire_id,           -- plus parlant pour un centre
    RENAME COLUMN Year_Birth TO annee_naissance, -- utile pour les stats âge
    RENAME COLUMN Income TO budget_formation,    -- potentiel CPF/OPCO/financement
    RENAME COLUMN Dt_Customer TO date_entree,    -- date de la première formation
    RENAME COLUMN MntWines TO ca_langues,        -- simulation de thématique
    RENAME COLUMN MntFruits TO ca_informatique,
    RENAME COLUMN MntMeatProducts TO ca_management,
    RENAME COLUMN MntFishProducts TO ca_art_culture,
    RENAME COLUMN MntSweetProducts TO ca_bien_etre,
    RENAME COLUMN NumWebPurchases TO inscription_web,          -- MonCompteFormation.fr
    RENAME COLUMN NumCatalogPurchases TO inscription_partenaires, -- OPCO/France Travail
    RENAME COLUMN NumStorePurchases TO inscription_centre;     -- inscriptions physiques
```

### 3. Création de la colonne volume_formation_finale
```sql
-- Contournement du blocage UPDATE sans abonnement BigQuery
CREATE OR REPLACE TABLE `mon_projet.formation_data_project.marketing_data_final` AS
SELECT 
    *,
    (inscription_web + inscription_partenaires + inscription_centre) 
    AS volume_formation_finale
FROM `mon_projet.formation_data_project.marketing_data`;
```

### 4. Vue segmentation fidélité (RFM adapté)
```sql
CREATE OR REPLACE VIEW `mon_projet.formation_data_project.v_segmentation_stagiaires` AS
SELECT 
    *,
    CASE 
        WHEN volume_formation_finale >= 15 THEN "Stagiaire Or (Fidèle)"
        WHEN volume_formation_finale >= 5  THEN "Stagiaire Argent (Régulier)"
        ELSE "Stagiaire Bronze (Occasionnel)"
    END AS segment_fidelite
FROM `mon_projet.formation_data_project.marketing_data_final`;
```

### 5. Vue canaux d'inscription (stagiaires uniques par canal)
```sql
SELECT 
    segment_fidelite, 
    COUNTIF(inscription_centre > 0)      AS nb_stagiaires_centre,
    COUNTIF(inscription_partenaires > 0) AS nb_stagiaires_partenaires,
    COUNTIF(inscription_web > 0)         AS nb_stagiaires_web  
FROM `mon_projet.formation_data_project.v_segmentation_stagiaires`
GROUP BY segment_fidelite;
-- COUNTIF > 0 pour compter les stagiaires uniques par canal
-- et non la somme des inscriptions (évite les doublons)
```

### 6. Vue Top 10 stagiaires
```sql
SELECT
    stagiaire_id,
    segment_fidelite,
    volume_formation_finale,
    budget_formation,
    date_entree,
    CASE
        WHEN ca_langues = GREATEST(ca_langues, ca_informatique, ca_management, 
             ca_art_culture, ca_bien_etre) THEN "Langues"
        WHEN ca_informatique = GREATEST(ca_langues, ca_informatique, ca_management, 
             ca_art_culture, ca_bien_etre) THEN "Informatique"
        WHEN ca_management = GREATEST(ca_langues, ca_informatique, ca_management, 
             ca_art_culture, ca_bien_etre) THEN "Management"
        WHEN ca_art_culture = GREATEST(ca_langues, ca_informatique, ca_management, 
             ca_art_culture, ca_bien_etre) THEN "Art & Culture"
        ELSE "Bien-être"
    END AS domaine_favori
FROM `mon_projet.formation_data_project.v_segmentation_stagiaires`
WHERE budget_formation < 200000
ORDER BY budget_formation DESC
LIMIT 11;
-- LIMIT 11 : le 1er résultat (666 666€) est irréaliste (données simulées)
-- on prend 11 lignes et on filtre le outlier dans Looker Studio
```

---

## 💡 Insights & Recommandations

**Insight 1 — Acquisition**  
Les stagiaires Or et Argent représentent plus de 80% des effectifs et sont acquis quasi exclusivement via le centre et le web.  
→ *Concentrer le budget acquisition sur ces 2 canaux. Le canal partenaires est peu stratégique pour le grand public.*

**Insight 2 — Offre**  
Les formations Langues et Management concentrent l'essentiel du CA. Art & Culture, Bien-être et Informatique génèrent peu de revenus à coût fixe élevé.  
→ *Créer une offre hybride Langues/Management à forte valeur. Basculer les petits domaines en format digital autonome pour supprimer les coûts formateur.*

**Insight 3 — Fidélisation**  
Les stagiaires Or reviennent régulièrement (20 à 30 formations) mais avec un budget unitaire faible. Les Bronze dépensent plus en une fois puis disparaissent.  
→ *Créer un programme de fidélisation pour convertir les Bronze en Argent, et un parcours premium pour maximiser le panier moyen des Or.*

---

## ⚠️ Note
Données simulées à des fins de démonstration — Dataset Kaggle recontextualisé.  
Les valeurs monétaires ne reflètent pas la réalité du marché de la formation professionnelle.
