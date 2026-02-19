# Projet : Agent IA Administrateur de Crédit Bancaire
## Document de Conception — Usage Comité de Direction

---

> **Classification :** Confidentiel — Usage Interne  
> **Version :** 1.0  
> **Date :** Février 2026  
> **Auteur :** Direction Architecture & Innovation IA  
> **Périmètre :** Traitement automatisé de 10 000 demandes de crédit / mois

---

## Table des matières

1. [Résumé Exécutif](#résumé-exécutif)
2. [Architecture du Système](#1-architecture-du-système)
3. [Modèle de Décision](#2-modèle-de-décision)
4. [Objectivité et Réduction des Biais](#3-objectivité-et-réduction-des-biais)
5. [Performance et Scalabilité](#4-performance-et-scalabilité)
6. [Conformité Réglementaire](#5-conformité-réglementaire)
7. [Interface Administrateur](#6-interface-administrateur)
8. [Simulation Chiffrée — 3 Profils Clients](#7-simulation-chiffrée--3-profils-clients)
9. [Feuille de Route & Budget Estimatif](#8-feuille-de-route--budget-estimatif)
10. [Conclusion](#conclusion)

---

## Résumé Exécutif

La banque traite actuellement **10 000 demandes de crédit par mois**, un volume qui génère des délais de traitement élevés, une variabilité humaine dans les décisions et un risque de non-conformité réglementaire croissant.

Ce document présente la conception complète d'un **Agent IA Administrateur de Crédit** — un système hybride multi-agents capable de :

- Traiter chaque dossier en **moins de 90 secondes** en moyenne
- Rendre des décisions **objectivement justifiables** et auditables
- Respecter intégralement le **RGPD, Bâle III et la directive MCD2**
- Réduire le taux de défaut prévisible de **15 à 20 %** par rapport au processus manuel
- Garantir un **droit au recours humain** pour tout demandeur

L'architecture proposée est un système hybride combinant des règles métier, du machine learning supervisé et un module LLM pour la génération d'explications en langage naturel, orchestré via une infrastructure cloud microservices.

---

## 1. Architecture du Système

### 1.1 Type d'Agent : Système Hybride Multi-Agents

Le système repose sur une **architecture multi-agents à trois couches** :

| Couche | Type | Rôle |
|--------|------|-------|
| **Couche 1 — Règles Métier** | Rule-Based Agent | Filtres de conformité réglementaire, seuils planchers |
| **Couche 2 — Scoring ML** | Supervised ML Agent | Évaluation du risque de crédit (XGBoost + réseau neuronal) |
| **Couche 3 — Explication & Audit** | LLM Agent (Claude Sonnet) | Génération de justifications en langage naturel, détection d'anomalies |

Un **Orchestrateur Central** coordonne les agents, gère les flux de données et garantit la traçabilité complète de chaque décision.

---

### 1.2 Description des Modules

#### Module 1 — Collecte et Normalisation des Données

**Fonction :** Agrégation et validation des données entrantes depuis plusieurs sources.

**Sources de données :**
- Formulaire de demande client (revenus, charges, montant souhaité, durée)
- Bureau de crédit (FICO/Score national, historique de remboursement)
- Open Banking API (données bancaires des 12 derniers mois, avec consentement)
- Base interne CRM (historique de relation client, produits détenus)
- Registres publics (statut juridique pour les professionnels, incidents déclarés)

**Traitements :**
- Validation de complétude (champs obligatoires)
- Détection de doublons et de demandes en cours
- Normalisation et encodage des variables
- Chiffrement de bout en bout des données sensibles (AES-256)

---

#### Module 2 — Scoring de Crédit

**Fonction :** Calcul du score de risque individuel sur une échelle de 0 à 1000.

**Modèle principal :** XGBoost (Gradient Boosted Trees) — choisi pour sa performance sur données tabulaires hétérogènes et son explicabilité native.

**Modèle secondaire :** Réseau de neurones dense (3 couches cachées) utilisé en ensemble pour affiner les prédictions sur les profils atypiques.

**Output :** Score de crédit composite + probabilité de défaut à 12, 24 et 36 mois.

---

#### Module 3 — Gestion des Risques

**Fonction :** Transformation du score en décision binaire ou ternaire.

**Zones de décision :**

| Plage de Score | Décision | Traitement |
|----------------|----------|------------|
| 750 — 1000 | ✅ Accord automatique | Génération de l'offre de prêt |
| 500 — 749 | 🔄 Accord conditionnel | Proposition avec conditions ajustées (taux, garanties) |
| 350 — 499 | ⚠️ Révision humaine | Transmis à un analyste pour décision finale |
| 0 — 349 | ❌ Refus automatique | Notification avec motifs et alternatives |

Les dossiers en zone "Révision humaine" représentent environ **8 à 12 %** du volume total, soit 800 à 1 200 dossiers/mois pour les analystes.

---

#### Module 4 — Détection de Fraude

**Fonction :** Identification des demandes frauduleuses ou à risque élevé d'identité usurpée.

**Méthodes :**
- Modèle d'anomalie (Isolation Forest) sur les patterns comportementaux
- Vérification KYC via API tierce (liveness check, comparaison documentaire)
- Analyse de cohérence inter-champs (revenu déclaré vs données Open Banking)
- Score de vélocité (nombre de demandes récentes depuis la même IP ou identité)

**Déclencheur :** Tout dossier avec un score de fraude > 0,7 est mis en quarantaine et transmis à l'équipe sécurité, indépendamment du score de crédit.

---

#### Module 5 — Auditabilité et Traçabilité

**Fonction :** Journalisation immuable de chaque décision pour audit réglementaire.

**Éléments enregistrés :**
- Horodatage de chaque étape du pipeline
- Version du modèle utilisé (Model Registry avec hash SHA-256)
- Valeurs des features d'entrée
- Score brut et score ajusté (post-biais)
- Décision finale + motifs SHAP (top 5 variables)
- Identifiant de l'analyste si révision humaine

**Stockage :** Base de données immuable (append-only) avec retention de 7 ans, conforme au RGPD.

---

### 1.3 Pipeline de Traitement — 10 000 Demandes Mensuelles

```
RÉCEPTION DE LA DEMANDE
        │
        ▼
┌─────────────────────────┐
│  MODULE 1 : COLLECTE    │  ← Validation, normalisation, chiffrement
│  Durée : ~5 secondes    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  MODULE 4 : FRAUDE      │  ← Exécuté en parallèle avec Module 2
│  Durée : ~10 secondes   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  MODULE 2 : SCORING     │  ← XGBoost + Réseau neuronal (ensemble)
│  Durée : ~15 secondes   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  MODULE 3 : DÉCISION    │  ← Règles métier + seuils réglementaires
│  Durée : ~5 secondes    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  MODULE 3b : LLM EXPL.  │  ← Génération de la justification client
│  Durée : ~20 secondes   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  MODULE 5 : AUDIT LOG   │  ← Enregistrement immuable
│  Durée : ~5 secondes    │
└────────────┬────────────┘
             │
             ▼
     NOTIFICATION CLIENT
     (Email / App / Courrier)

     Durée totale moyenne : ~60-90 secondes
```

**Débit cible :**
- 10 000 demandes / mois = ~333 demandes / jour
- Pointe haute : jusqu'à 600 demandes / jour
- Traitement en continu 24h/24, 7j/7

---

## 2. Modèle de Décision

### 2.1 Variables Utilisées

Le modèle s'appuie sur **47 variables** regroupées en 6 catégories :

#### Catégorie A — Capacité de Remboursement (Poids : 35 %)

| Variable | Description | Type |
|----------|-------------|------|
| Revenu net mensuel | Après impôts et charges sociales | Numérique |
| Taux d'endettement actuel | (Charges fixes / Revenus) × 100 | Numérique |
| Taux d'endettement projeté | Avec la nouvelle mensualité | Numérique |
| Reste-à-vivre | Revenu - toutes charges - mensualité projetée | Numérique |
| Stabilité des revenus | Coefficient de variation sur 12 mois (Open Banking) | Numérique |

#### Catégorie B — Historique de Crédit (Poids : 25 %)

| Variable | Description | Type |
|----------|-------------|------|
| Score bureau de crédit | Score national standardisé | Numérique |
| Nombre d'incidents de paiement (36 mois) | Retards, impayés | Entier |
| Ancienneté crédit | Durée depuis premier crédit | Numérique |
| Diversité du portefeuille crédit | Types de crédits actifs | Catégorielle |
| Taux d'utilisation revolving | Utilisation vs limite autorisée | Numérique |

#### Catégorie C — Stabilité Professionnelle (Poids : 20 %)

| Variable | Description | Type |
|----------|-------------|------|
| Statut d'emploi | CDI / CDD / Indépendant / Retraité | Catégorielle |
| Ancienneté dans l'emploi actuel | Mois | Entier |
| Secteur d'activité | Codification NAF/NACE | Catégorielle |
| Ancienneté entreprise (si indépendant) | Années | Numérique |

#### Catégorie D — Caractéristiques du Prêt (Poids : 10 %)

| Variable | Description | Type |
|----------|-------------|------|
| Montant demandé | EUR | Numérique |
| Durée demandée | Mois | Entier |
| Objet du prêt | Immobilier / Auto / Consommation / Professionnel | Catégorielle |
| Ratio LTV | Loan-to-Value (si prêt garanti) | Numérique |

#### Catégorie E — Comportement Bancaire (Poids : 7 %)

| Variable | Description | Type |
|----------|-------------|------|
| Ancienneté client | Mois de relation bancaire | Entier |
| Solde moyen compte courant (12 mois) | EUR | Numérique |
| Fréquence des incidents (découvert) | Nb/an | Entier |
| Épargne détenue | Total épargne bancaire | Numérique |

#### Catégorie F — Contexte Macro (Poids : 3 %)

| Variable | Description | Type |
|----------|-------------|------|
| Région géographique | Code INSEE région | Catégorielle |
| Indice de conjoncture sectorielle | Score externe trimestriel | Numérique |

> **Variables EXCLUES par principe éthique :** Sexe, origine, nationalité, religion, situation familiale, adresse détaillée (pas de redlining géographique).

---

### 2.2 Méthode de Scoring

#### Architecture Ensemble

Le score final est une combinaison pondérée de deux modèles :

**Modèle 1 — XGBoost (Poids : 65 %)**
- Algorithme : Gradient Boosted Decision Trees
- Hyperparamètres optimisés via Bayesian Search (Optuna)
- Entraînement sur 3 ans d'historique (120 000 dossiers labellisés)
- Validation croisée stratifiée (5-fold)
- Métriques de performance cibles : AUC-ROC ≥ 0.82, Gini ≥ 0.64

**Modèle 2 — Réseau Neuronal Dense (Poids : 35 %)**
- Architecture : 3 couches cachées (256 → 128 → 64 neurones), ReLU + Dropout
- Complète XGBoost sur les profils non-standards (travailleurs indépendants, jeunes actifs)
- Normalisation BatchNorm pour stabilité

**Score Final :**
```
Score_Final = 0.65 × Score_XGBoost + 0.35 × Score_NN
Score_Composite = f(Score_Final, Règles_Métier, Score_Fraude)
```

#### Calibration et Gestion du Seuil

Le modèle est calibré via **Platt Scaling** pour que les probabilités de sortie soient bien calibrées. Les seuils de décision sont ajustés trimestriellement en fonction des performances observées et de l'appétit au risque de la banque.

---

### 2.3 Explicabilité des Décisions

#### SHAP (SHapley Additive exPlanations)

Chaque décision est accompagnée d'un rapport SHAP identifiant les **5 variables les plus influentes** sur la décision, avec leur contribution positive ou négative.

**Exemple de sortie SHAP pour un refus :**

```
Décision : REFUS (Score : 342/1000)

Top 5 facteurs explicatifs :
  ⬇ Taux d'endettement projeté : 68% → contribution : -127 pts
  ⬇ Incidents de paiement (3 sur 36 mois) → contribution : -89 pts
  ⬇ CDD (ancienneté 4 mois) → contribution : -54 pts
  ⬆ Score bureau de crédit : 612 → contribution : +43 pts
  ⬆ Ancienneté client (8 ans) → contribution : +28 pts
```

#### LLM Agent — Génération d'Explications Client

Le module LLM transforme les données SHAP en **lettre d'explication compréhensible** pour le client, respectant les obligations réglementaires de transparence (article 22 RGPD).

**Exemple de lettre générée :**
> *"Votre demande de crédit n'a pas pu être approuvée. La principale raison est que votre taux d'endettement projeté (68%) dépasse le seuil prudentiel réglementaire de 35%. Par ailleurs, trois incidents de paiement enregistrés au cours des trois dernières années ont influencé défavorablement notre évaluation. Votre ancienneté de client (8 ans) et votre score de crédit ont cependant été valorisés positivement. Vous pouvez contester cette décision en contactant notre service client ou en demandant l'examen de votre dossier par un conseiller humain."*

#### LIME (Local Interpretable Model-agnostic Explanations)

LIME est utilisé en complément pour les dossiers en zone "Révision Humaine", offrant aux analystes une seconde perspective d'interprétabilité locale.

---

## 3. Objectivité et Réduction des Biais

### 3.1 Méthodes pour Éviter les Discriminations

#### Étape 1 — Exclusion des Proxies Discriminatoires

Lors de la sélection des features, un **test de corrélation systématique** est effectué pour détecter les variables proxy des attributs protégés (exemple : certains codes postaux peuvent être corrélés à l'origine ethnique et constituent du "redlining" digital). Ces variables sont retirées du modèle.

#### Étape 2 — Entraînement avec Contraintes d'Équité

Le modèle intègre une **contrainte de fairness** lors de l'entraînement via la librairie Fairlearn (Microsoft). La fonction de perte est augmentée d'un terme de pénalité si l'écart de taux d'approbation entre groupes protégés dépasse le seuil acceptable.

**Technique utilisée :** Adversarial Debiasing — un réseau adversaire tente de prédire les attributs protégés depuis les représentations internes du modèle principal, qui apprend à les masquer.

#### Étape 3 — Calibration Post-Entraînement

Vérification que les probabilités de défaut sont similaires pour des profils de risque identiques appartenant à différents groupes démographiques.

---

### 3.2 Indicateurs d'Équité (Fairness Metrics)

Les métriques suivantes sont calculées et reportées **mensuellement** :

| Métrique | Description | Seuil d'alerte |
|----------|-------------|----------------|
| **Demographic Parity** | Écart de taux d'approbation entre groupes | > 5 pts de % |
| **Equal Opportunity** | Écart de taux de vrais positifs entre groupes | > 5 pts de % |
| **Calibration Equity** | Cohérence des probabilités de défaut par groupe | Écart > 3 % |
| **Disparate Impact Ratio** | Ratio des taux d'approbation (groupe favorisé / défavorisé) | < 0.80 |
| **Predictive Parity** | Précision du modèle équivalente entre groupes | Écart > 5 % |

Les groupes protégés surveillés incluent : tranche d'âge (jeunes / seniors), genre (si disponible avec consentement), statut d'emploi, région géographique.

---

### 3.3 Audit Interne et Traçabilité

**Audit continu :**
- Monitoring en temps réel des métriques d'équité via dashboard
- Alertes automatiques si un indicateur dépasse le seuil critique

**Audit périodique :**
- Revue trimestrielle par le Comité Risques & Conformité
- Rapport semestriel soumis au régulateur (ACPR en France)
- Audit annuel externe par un cabinet indépendant spécialisé IA

**Traçabilité technique :**
- Chaque décision est stockée avec l'ensemble des entrées, la version exacte du modèle (hash), les scores intermédiaires et les valeurs SHAP
- Journal d'audit immuable (blockchain permissionnée ou base append-only certifiée)
- Durée de conservation : 7 ans conformément aux obligations légales

---

## 4. Performance et Scalabilité

### 4.1 Temps de Traitement

| Étape | Durée Moyenne | Durée Max (P99) |
|-------|---------------|-----------------|
| Collecte & Validation | 5 s | 15 s |
| Détection Fraude | 10 s | 30 s |
| Scoring ML | 15 s | 40 s |
| Décision & Règles | 5 s | 10 s |
| Génération Explication LLM | 20 s | 45 s |
| Audit Log | 3 s | 8 s |
| **Total Pipeline** | **~58 s** | **< 148 s** |

**SLA cible :** 95 % des demandes traitées en moins de 2 minutes.

---

### 4.2 Infrastructure Recommandée

#### Architecture Cloud Microservices

```
                    ┌──────────────────────────────────┐
                    │       API GATEWAY (Kong)          │
                    │  Rate Limiting | Auth | TLS 1.3   │
                    └─────────────┬────────────────────-┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │ Service Collect│  │ Service Fraude │  │ Service Scoring│
     │ (Python/FastAPI│  │ (Python)       │  │ (Python/ML)    │
     │  × 4 replicas) │  │  × 2 replicas) │  │  × 6 replicas) │
     └────────────────┘  └────────────────┘  └────────────────┘
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │Service Décision│  │ Service LLM    │  │ Service Audit  │
     │ (Python)       │  │ (Claude API)   │  │ (Go)           │
     │  × 3 replicas) │  │  × 3 replicas) │  │  × 2 replicas) │
     └────────────────┘  └────────────────┘  └────────────────┘
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  │
                    ┌─────────────┴──────────────┐
                    │     Message Queue (Kafka)   │
                    │  File d'attente asynchrone  │
                    └─────────────┬──────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │ PostgreSQL     │  │ Redis Cache    │  │ Data Warehouse │
     │ (Dossiers)     │  │ (Sessions)     │  │ (Analytics ML) │
     └────────────────┘  └────────────────┘  └────────────────┘
```

**Stack technologique :**

| Composant | Technologie | Justification |
|-----------|-------------|---------------|
| Orchestration | Kubernetes (EKS AWS) | Auto-scaling, haute disponibilité |
| Services ML | Python 3.11 + FastAPI | Performance, écosystème ML |
| Message Queue | Apache Kafka | Découplage, traitement asynchrone |
| Base de données | PostgreSQL + TimescaleDB | Fiabilité, séries temporelles |
| Cache | Redis Cluster | Latence sub-milliseconde |
| LLM | Anthropic Claude API | Qualité des explications |
| Monitoring | Prometheus + Grafana | Observabilité en temps réel |
| CI/CD | GitLab CI + ArgoCD | Déploiement continu sécurisé |
| Sécurité | HashiCorp Vault | Gestion des secrets |

---

### 4.3 Gestion des Pics de Demande

**Stratégie d'auto-scaling horizontal :**
- Kubernetes HPA (Horizontal Pod Autoscaler) configuré sur métriques CPU et longueur de file Kafka
- Scaling de 4 à 20 replicas pour les services ML en moins de 90 secondes
- Circuit breaker pattern (Hystrix) pour dégradation gracieuse

**Capacité nominale vs pic :**

| Scénario | Demandes/heure | Infrastructure |
|----------|----------------|----------------|
| Nominal | 42 dossiers/h | 4 replicas scoring |
| Pic modéré | 120 dossiers/h | 8 replicas scoring |
| Pic exceptionnel | 300 dossiers/h | 16 replicas scoring |
| Maximum théorique | 600 dossiers/h | 20 replicas scoring |

**File d'attente Kafka :** Les demandes excédentaires ne sont pas perdues mais mises en file avec notification au client ("Votre demande est en cours de traitement, résultat sous 2 heures").

---

## 5. Conformité Réglementaire

### 5.1 Respect du RGPD

**Principes appliqués :**

- **Minimisation des données** : Seules les données strictement nécessaires à l'évaluation sont collectées et traitées.
- **Durée de conservation limitée** : Les données brutes sont supprimées après 7 ans (obligation légale bancaire), les données pseudonymisées sont conservées pour l'amélioration des modèles.
- **Consentement explicite** : L'accès aux données Open Banking requiert un consentement explicite du client, révocable à tout moment.
- **Droit d'accès et de rectification** : API dédiée permettant au client de consulter les données utilisées pour sa décision et de signaler des inexactitudes.
- **Droit à l'effacement** : Procédure automatisée de suppression, sous réserve des obligations légales de conservation bancaire.
- **Privacy by Design** : Chiffrement AES-256 au repos, TLS 1.3 en transit, pseudonymisation des identifiants dans les logs ML.

**Data Protection Impact Assessment (DPIA) :** Obligatoire pour ce type de traitement automatisé à large échelle. Réalisé avant mise en production et mis à jour annuellement.

---

### 5.2 Transparence des Décisions Automatisées

Conformément à l'**article 22 du RGPD** et à la **directive sur le crédit à la consommation (MCD2)** :

- Tout client doit être informé qu'une décision automatisée est en cours.
- La logique générale du système doit être expliquée (pas nécessairement le modèle complet).
- Les facteurs déterminants de la décision doivent être communiqués en langage clair.

**Implémentation :**
- Notice d'information automatisée lors du dépôt de la demande.
- Lettre de décision générée par le LLM expliquant les motifs principaux.
- Résumé SHAP des 5 variables clés, traduit en langage non-technique.

---

### 5.3 Droit au Recours Humain

**Mécanisme garanti :**

Tout client peut, **dans un délai de 30 jours** après notification de la décision :

1. **Demander la révision humaine** via formulaire en ligne, agence ou courrier.
2. **Fournir des éléments complémentaires** (justificatifs non pris en compte, situations exceptionnelles).
3. **Obtenir un entretien** avec un conseiller crédit qualifié sous 5 jours ouvrables.
4. **Contester auprès du médiateur bancaire** si la révision humaine est jugée insatisfaisante.

**SLA recours humain :**
- Accusé de réception : < 24h
- Examen du dossier : < 5 jours ouvrables
- Décision finale : < 10 jours ouvrables

**Volume estimé :** 2 à 5 % des décisions font l'objet d'une demande de recours, soit 200 à 500 dossiers/mois.

---

### 5.4 Conformité Bâle III / CRR2

- Le modèle de scoring est soumis à **validation par le modèle de risque interne** (IRB Approach).
- Les paramètres PD (Probabilité de Défaut), LGD (Loss Given Default) et EAD (Exposure at Default) sont calibrés selon les guidelines EBA.
- Backtesting trimestriel des prédictions vs réalisations.
- Rapport au régulateur ACPR sur les modèles utilisés dans le cadre des stress tests.

---

## 6. Interface Administrateur

### 6.1 Dashboard de Supervision

Le dashboard temps réel est accessible aux équipes Risk, Conformité, Crédit et Direction. Il est conçu pour offrir une **vision à 360° du système** en un coup d'œil.

**Architecture du dashboard (React + Grafana) :**

```
┌─────────────────────────────────────────────────────────────────┐
│  AGENT IA CRÉDIT — SUPERVISION TEMPS RÉEL          [Actualiser] │
│  Dernière mise à jour : 19/02/2026 — 14h32                      │
├────────────┬────────────┬────────────┬────────────┬─────────────┤
│ Demandes   │ Taux       │ Score      │ Taux       │ Taux Fraude │
│ en cours   │ Approbation│ Moyen      │ Défaut     │ Détecté     │
│    47      │  63.4 %    │  618/1000  │  2.1 %     │  0.8 %      │
│ ▲ Normal   │ ▲ +1.2%    │ ▼ -12 pts  │ ▲ Cible 2% │ ▼ Faible   │
├────────────┴────────────┴────────────┴────────────┴─────────────┤
│  VOLUME MENSUEL (Fév. 2026)          │  DISTRIBUTION DES SCORES │
│  ████████████████░░ 8,234 / 10,000   │  [Histogramme graphique] │
│  Accord auto : 5,208 (63.2%)         │  0-349 : ████ 18%        │
│  Accord conditionnel : 1,423 (17.3%) │  350-499: ███ 12%        │
│  Révision humaine : 987 (12%)        │  500-749: ████████ 42%   │
│  Refus auto : 616 (7.5%)             │  750-1000:███████ 28%    │
├──────────────────────────────────────┴──────────────────────────┤
│  ALERTES ACTIVES                                                │
│  ⚠️ Taux de révision humaine : 12% (seuil : 10%) — depuis 2j    │
│  ℹ️ Pic de charge prévu demain matin — scaling automatique actif │
└─────────────────────────────────────────────────────────────────┘
```

---

### 6.2 Indicateurs Clés (KPIs)

| KPI | Description | Fréquence | Seuil d'alerte |
|-----|-------------|-----------|----------------|
| Taux d'approbation | % de demandes approuvées (auto + conditionnel) | Temps réel | < 55 % ou > 75 % |
| Score moyen | Score composite moyen de la cohorte mensuelle | Quotidien | Variation > 30 pts |
| Taux de défaut prévisionnel | PD moyen du portefeuille approuvé | Hebdomadaire | > 3 % |
| Taux de défaut réel (J+90) | Défauts avérés sur cohorte N-3 mois | Mensuel | > 2.5 % |
| Taux de fraude détectée | % dossiers signalés fraude | Temps réel | > 2 % |
| Temps de traitement P95 | 95e percentile du temps pipeline | Temps réel | > 3 minutes |
| Taux de recours humain | % clients ayant demandé révision | Hebdomadaire | > 5 % |
| Disparate Impact Ratio | Équité entre groupes démographiques | Mensuel | < 0.80 |
| Disponibilité système | Uptime des microservices | Temps réel | < 99.5 % |

---

### 6.3 Alertes Automatiques

**Système d'alertes à 3 niveaux :**

| Niveau | Condition | Destinataires | Canal |
|--------|-----------|---------------|-------|
| 🟡 **INFO** | Métriques en zone de surveillance | Équipe Tech | Slack |
| 🟠 **WARNING** | Seuil d'alerte dépassé | Risk Manager + Tech Lead | Slack + Email |
| 🔴 **CRITICAL** | SLA non respecté ou anomalie grave | DRO + RSSI + Direction | SMS + Appel |

**Exemples d'alertes :**
- Biais détecté : Disparate Impact Ratio < 0.80 → Revue immédiate du modèle
- Dérive du modèle : AUC-ROC sur données récentes < 0.78 → Retraining déclenché
- Incident technique : Latence pipeline > 5 minutes → Escalade automatique
- Volume inhabituel : > 800 demandes en 1 jour → Scaling préventif déclenché

---

## 7. Simulation Chiffrée — 3 Profils Clients

### Profil 1 — Marie D., 34 ans, Cadre en CDI

**Données du dossier :**

| Variable | Valeur |
|----------|--------|
| Statut emploi | CDI — Cadre Marketing |
| Ancienneté emploi | 7 ans |
| Revenu net mensuel | 4 200 € |
| Charges fixes actuelles | 800 € (loyer) |
| Taux d'endettement actuel | 19 % |
| Score bureau de crédit | 742/850 |
| Incidents de paiement (36 mois) | 0 |
| Montant demandé | 18 000 € (véhicule) |
| Durée demandée | 60 mois |
| Mensualité projetée | 320 € |
| Taux d'endettement projeté | 26.7 % |
| Ancienneté client | 5 ans |

**Traitement IA :**

- Score XGBoost : 791/1000
- Score Réseau Neuronal : 804/1000
- Score Composite Final : **796/1000**
- Score de Fraude : 0.04 (très faible)

**Décision : ✅ ACCORD AUTOMATIQUE**

**Justification SHAP :**
```
Facteurs positifs :
  ⬆ Score bureau crédit élevé (742)    → +142 pts
  ⬆ 0 incident de paiement             → +98 pts
  ⬆ CDI + 7 ans ancienneté             → +87 pts
  ⬆ Taux d'endettement projeté : 26.7% → +73 pts
  ⬆ Revenu stable (σ faible)           → +44 pts
```

**Offre générée automatiquement :**
- Montant accordé : 18 000 €
- Durée : 60 mois
- Taux proposé : 4.2 % (TAEG)
- Mensualité : 333 €
- Assurance incluse selon profil

**Délai de traitement :** 67 secondes — Notification par email et application bancaire.

---

### Profil 2 — Ahmed K., 28 ans, Développeur en Freelance

**Données du dossier :**

| Variable | Valeur |
|----------|--------|
| Statut emploi | Indépendant / Freelance IT |
| Ancienneté activité | 2 ans 3 mois |
| Revenu net mensuel moyen (12 mois) | 5 100 € |
| Variabilité revenus (coefficient) | 0.28 (modérée) |
| Charges fixes actuelles | 1 100 € (loyer) |
| Taux d'endettement actuel | 21.6 % |
| Score bureau de crédit | 619/850 |
| Incidents de paiement (36 mois) | 1 (retard 30j, il y a 18 mois) |
| Montant demandé | 25 000 € (apport immobilier) |
| Durée demandée | 84 mois |
| Mensualité projetée | 358 € |
| Taux d'endettement projeté | 28.6 % |
| Épargne disponible | 12 000 € |

**Traitement IA :**

- Score XGBoost : 578/1000
- Score Réseau Neuronal : 612/1000 (profil atypique — NN plus adapté)
- Score Composite Final : **591/1000**
- Score de Fraude : 0.11 (faible)

**Décision : 🔄 ACCORD CONDITIONNEL**

**Justification SHAP :**
```
Facteurs positifs :
  ⬆ Revenu élevé (5 100€)              → +118 pts
  ⬆ Épargne significative (12 000€)    → +87 pts
  ⬆ Taux endettement projeté : 28.6%   → +62 pts

Facteurs négatifs :
  ⬇ Variabilité revenus freelance       → -94 pts
  ⬇ Ancienneté activité : 27 mois       → -71 pts
  ⬇ 1 incident paiement (18 mois)       → -48 pts
```

**Offre conditionnelle générée :**
- Montant accordé : 20 000 € (80 % du demandé, pour limiter l'exposition)
- Durée : 84 mois
- Taux proposé : 5.8 % (TAEG, prime de risque freelance)
- Mensualité : 286 €
- Condition : Apport personnel de 5 000 € ou garantie d'un tiers
- Alternative : Réduction à 15 000 € sans condition supplémentaire

**Communication client :** Lettre explicative générée par le LLM présentant les options et invitant le client à contacter un conseiller pour finaliser.

**Délai de traitement :** 83 secondes.

---

### Profil 3 — Jacques M., 61 ans, Retraité

**Données du dossier :**

| Variable | Valeur |
|----------|--------|
| Statut emploi | Retraité (pension civile) |
| Revenu net mensuel | 2 650 € (pension + complément) |
| Stabilité revenus | Très élevée (pension indexée) |
| Charges fixes actuelles | 650 € (crédit immo résiduel) |
| Charges restantes | 24 mois de crédit immobilier |
| Taux d'endettement actuel | 24.5 % |
| Score bureau de crédit | 548/850 |
| Incidents de paiement (36 mois) | 2 (retards < 30j, il y a 28 et 33 mois) |
| Montant demandé | 35 000 € (travaux maison) |
| Durée demandée | 120 mois (10 ans) |
| Mensualité projetée | 380 € |
| Taux d'endettement projeté | 38.9 % |
| Âge en fin de prêt | 71 ans |

**Traitement IA :**

- Score XGBoost : 398/1000
- Score Réseau Neuronal : 372/1000
- Score Composite Final : **388/1000**
- Score de Fraude : 0.07 (très faible)

**Décision : ⚠️ TRANSMIS EN RÉVISION HUMAINE**

**Justification SHAP :**
```
Facteurs de blocage :
  ⬇ Taux endettement projeté : 38.9%   → -121 pts (> seuil réglementaire 35%)
  ⬇ 2 incidents de paiement             → -73 pts
  ⬇ Durée prêt / âge (71 ans à terme)  → -68 pts
  ⬇ Score bureau crédit : 548           → -51 pts

Facteurs favorables :
  ⬆ Revenus très stables (pension)      → +89 pts
  ⬆ Crédit immo finissant dans 24 mois → +44 pts
```

**Motifs de transmission en révision humaine :**

Le taux d'endettement projeté de 38.9% dépasse le seuil réglementaire de 35%, mais le crédit immobilier existant se termine dans 24 mois. Un analyste humain doit évaluer : (1) la possibilité d'un montage intégrant la libération de mensualité prévue, (2) l'opportunité d'une assurance emprunteur adaptée à l'âge, (3) un éventuel prêt viager hypothécaire en alternative.

**Message au client :** "Votre dossier requiert un examen complémentaire par l'un de nos conseillers spécialisés. Un rendez-vous vous sera proposé sous 3 jours ouvrables."

**Délai de traitement IA :** 71 secondes. Délai total avec révision humaine : < 5 jours ouvrables.

---

### Récapitulatif de la Simulation

| Profil | Score | Décision | Délai IA | Résultat |
|--------|-------|----------|----------|---------|
| Marie D. — Cadre CDI | 796/1000 | ✅ Accord auto | 67 s | Prêt 18 000 € — 4.2% |
| Ahmed K. — Freelance | 591/1000 | 🔄 Accord conditionnel | 83 s | Prêt 20 000 € — conditions |
| Jacques M. — Retraité | 388/1000 | ⚠️ Révision humaine | 71 s | Analyste sous 5 jours |

---

## 8. Feuille de Route & Budget Estimatif

### 8.1 Phases de Déploiement

| Phase | Durée | Contenu | Jalons |
|-------|-------|---------|--------|
| **Phase 0 — Cadrage** | 1 mois | Audit des données, DPIA, validation conformité | Feu vert ACPR |
| **Phase 1 — MVP** | 3 mois | Infrastructure cloud, module collecte, XGBoost baseline | POC sur 500 dossiers |
| **Phase 2 — Intégration** | 2 mois | Détection fraude, LLM, dashboard, révision humaine | Tests UAT |
| **Phase 3 — Pilote** | 2 mois | Déploiement à 20 % du volume réel, monitoring intensif | Go/No-Go direction |
| **Phase 4 — Généralisation** | 1 mois | Déploiement 100 % du volume | Production complète |
| **Phase 5 — Amélioration continue** | Permanent | Retraining, audit, optimisations | Revues trimestrielles |

**Durée totale jusqu'à production :** 9 mois

---

### 8.2 Budget Estimatif (Année 1)

| Poste | Coût Estimé |
|-------|-------------|
| Infrastructure cloud (AWS/Azure) | 120 000 € / an |
| Licences et APIs (LLM, Bureau crédit, KYC) | 180 000 € / an |
| Développement et intégration (équipe interne + prestataires) | 450 000 € (one-shot) |
| Audit conformité RGPD et ACPR | 60 000 € / an |
| Formation équipes (Risk, Crédit, Tech) | 30 000 € (one-shot) |
| Monitoring et maintenance (DevOps) | 80 000 € / an |
| **Total Année 1** | **~920 000 €** |
| **Coût récurrent (à partir de l'an 2)** | **~440 000 € / an** |

### 8.3 ROI Estimé

- Réduction du temps de traitement : de 5 jours à < 2 minutes → Gain en satisfaction client et conversion
- Réduction du taux de défaut de 15-20 % → Économies de provisions estimées à **600 000 — 1 200 000 € / an** (selon portefeuille)
- Réduction des coûts analystes : automatisation de 88 % du volume → Réallocation de 6 à 8 ETP sur des tâches à valeur ajoutée
- **Retour sur investissement estimé : 18 à 24 mois**

---

## Conclusion

L'Agent IA Administrateur de Crédit présenté dans ce document constitue une solution robuste, équitable, transparente et scalable pour le traitement de 10 000 demandes de crédit mensuelles.

La combinaison d'une architecture multi-agents hybride — règles métier + XGBoost + LLM — permet d'atteindre un équilibre optimal entre performance prédictive, explicabilité des décisions et conformité réglementaire. Le maintien d'un circuit de révision humaine garantit à la fois la protection des clients et la responsabilité juridique de la banque.

Le système est conçu pour évoluer : les modèles se réentraînent automatiquement, les biais sont détectés en continu, et l'infrastructure cloud s'adapte aux variations de charge. La traçabilité complète de chaque décision répond aux exigences les plus strictes des régulateurs européens.

**Prochaines étapes recommandées :**
1. Validation de ce document par le Comité Risques et la Direction Conformité
2. Lancement de la Phase 0 (Cadrage et DPIA) dès approbation du budget
3. Désignation d'un Product Owner IA crédit (profil hybride Risque/Tech)
4. Pré-notification à l'ACPR de l'intention de déployer un système IRB automatisé

---

*Document préparé par la Direction Architecture & Innovation IA — Usage strictement interne — Reproduction interdite sans autorisation*

*Classification : Confidentiel | Version 1.0 | Février 2026*
