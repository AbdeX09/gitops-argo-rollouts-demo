# gitops-argo-rollouts-demo
# GitOps & Progressive Delivery avec Argo Rollouts

> Déploiements **canary** automatisés et **rollback automatique** sur Kubernetes, pilotés par GitOps (Argo CD + Argo Rollouts).
> Projet de recherche — Écosystème Kubernetes & GitOps.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-kind-326CE5?logo=kubernetes&logoColor=white)](https://kind.sigs.k8s.io)
[![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io)
[![Argo Rollouts](https://img.shields.io/badge/Argo%20Rollouts-Canary-02C39A?logo=argo&logoColor=white)](https://argo-rollouts.readthedocs.io)
[![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io)

---

## 🎯 Objectif

Démontrer une chaîne de **progressive delivery** complète : un commit Git déclenche un déploiement **canary** (montée progressive du trafic 20 % → 50 % → 100 %), validé par une **analyse automatique**. Si la nouvelle version est défaillante, le système effectue un **rollback automatique** vers la version stable — sans aucune intervention humaine.

## 🏗️ Architecture

```
Développeur ──git push──▶ GitHub (repo GitOps)
                              │
                              ▼
        ┌─────────────────────────────────────────────┐
        │   Cluster Kubernetes (kind sur Codespaces)   │
        │                                              │
        │   Argo CD ──sync auto──▶ Argo Rollouts       │
        │   (GitOps)               (canary + rollback) │
        │      ▲                          │            │
        │   Prometheus ◀──────────  Application démo   │
        │   (observabilité)         (stable + canary)  │
        └─────────────────────────────────────────────┘
```

![Architecture](img/architecture.png)

**Git est la source de vérité unique.** Argo CD synchronise le cluster sur l'état décrit dans le dépôt ; Argo Rollouts gère le déploiement progressif et sécurisé de chaque version.

## 📦 Stack technique

| Composant | Rôle |
|-----------|------|
| **kind** | Cluster Kubernetes local (1 nœud) |
| **Argo CD** | Synchronisation GitOps (auto-sync, self-heal) |
| **Argo Rollouts** | Déploiement canary / rollback automatique |
| **Prometheus** | Collecte de métriques (observabilité) |
| **argoproj/rollouts-demo** | Application de démo (versions colorées) |

L'ensemble tourne sur **GitHub Codespaces** (4 cœurs / 16 Go RAM), pour un environnement reproductible et accessible partout.

## 📂 Structure du dépôt

```
.
├── apps/
│   └── demo-app/
│       ├── rollout.yaml            # Rollout (stratégie canary)
│       ├── service.yaml            # services stable + canary
│       └── analysis-template.yaml  # analyse automatique
├── prometheus-values.yaml          # configuration Helm de Prometheus
└── README.md
```

## 🚀 Mise en route

```bash
# 1. Créer le cluster
kind create cluster --name gitops-demo

# 2. Installer Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Installer Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 4. Installer Prometheus (Helm)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace

# 5. Déclarer l'application GitOps (Argo CD pointe vers ce dépôt)
kubectl apply -f apps/demo-app/
```

---

## 📸 Démonstration en images

### 1. Le cluster et l'application sont sains

L'application démarre sur un état stable : nœud `Ready`, Rollout `Healthy`, version `blue`, 5 pods `Running`.

![État initial sain](img/03-rollout-initial.png)

### 2. GitOps : un commit modifie l'infrastructure

Une modification du manifeste (ici `replicas: 3`) commitée sur GitHub est automatiquement appliquée au cluster par Argo CD — sans commande `kubectl`.

![Commit GitHub](img/02-commit-replicas.png)

Le dashboard Argo CD confirme l'état `Healthy` / `Synced` et visualise toute la chaîne de ressources :

![Dashboard Argo CD](img/01-argocd-dashboard.png)

### 3. Déploiement canary : pause à 20 %

Lors d'un changement de version (`blue` → `yellow`), Argo Rollouts dirige 20 % du trafic vers le canary, puis se met en pause. Les deux versions coexistent.

![Canary en pause à 20%](img/04-canary-pause.png)

Vue graphique du dashboard Argo Rollouts :

![Dashboard Rollouts](img/06-dashboard-rollouts.png)

### 4. Promotion : bascule complète

Après validation, la nouvelle version monte à 100 % et l'ancienne est éteinte (`ScaledDown`).

![Promotion](img/05-promotion.png)

### 5. Analyse automatique (health-check)

À chaque canary, un Job de health-check teste la version. Sur une version saine, les `AnalysisRun` se terminent en `Successful` et la promotion est autorisée. Sur une version défaillante, l'analyse échoue → **rollback automatique**.

![Analyse Successful](img/09-rollout-healthy-analysis.png)

---

## 🔑 Concepts clés démontrés

- **GitOps** : Git comme source de vérité, synchronisation pull-based, auto-réparation (`selfHeal`).
- **Progressive delivery** : montée progressive du trafic (canary), limitation du « blast radius ».
- **Analyse automatique** : validation d'une version sur des critères mesurables avant promotion.
- **Rollback automatique** : retour à la version stable sans intervention humaine en cas d'échec.

## 📚 Références

- [Documentation Argo Rollouts](https://argo-rollouts.readthedocs.io)
- [Documentation Argo CD](https://argo-cd.readthedocs.io)
- [CNCF](https://www.cncf.io)

---

*Projet réalisé par AbdeX09 dans le cadre d'un travail de recherche sur l'écosystème Kubernetes & GitOps.*
