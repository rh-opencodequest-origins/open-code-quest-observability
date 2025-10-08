# Repository pour le workshop Open Code Quest - la partie observabilité

## Contexte du Projet

Ce projet met en place une **stratégie GitOps complète** pour gérer l’infrastructure d'observabilité sur plusieurs clusters via **Red Hat Advanced Cluster Management (ACM)** et **Argo CD**.
Il s’appuie sur le plugin **PolicyGenerator** pour générer automatiquement les `Policy`, `Placement` et `PlacementBinding` à partir de définitions déclaratives (`PolicyGenerator.yaml`).  

## Organisation de ce repo

Ce repo organise l’ensemble des politiques et applications de manière **hiérarchique et modulaire** :

- **argocd-instance.yaml** : instance ArgoCD préconfiguré avec le plugin PolicyGenerator.
- **Root App Argo CD (`apps/root-app.yaml`)** : orchestrateur central de toutes les sous-applications.
- **Sous-applications Argo CD** :
  - `grafana-policy-app.yaml` → déploie Grafana Operator, une instance Grafana et les dashboards d'observabilité
  - `mco-policy-app.yaml` → déploie MCO avec MinIO persistant
  - `uw-policy-app.yaml` → Active le user workload monitoring sur les serveurs managés
- **Dossiers `policies/*`** : contiennent les manifests ACM et les Kustomization pour chaque policy.

---

## Structure du Repo

```
manifests/
├── argocd-instance.yaml       # Instance ArgoCD
├── apps/                      # Applications Argo CD (App-of-Apps)
│   ├── root-app.yaml          # Application racine qui orchestre toutes les sous-applications
│   ├── uw-policy-app.yaml
│   ├── grafana-policy-app.yaml
│   └── mco-policy-app.yaml
├── policies/                  # Policies ACM
│   ├── uw-policy/             # Mise en place de UW Monitoring
│   ├── grafana-policy/        # Installation de Grafana 
│   └── mco-policy/            # Installation MCO + MinIO persistent
```

---

## 1. Déploiement d'Argo CD

`PolicyGenerator` est un **plugin Kustomize alpha**, ce qui signifie qu’il n’est pas activé par défaut dans Argo CD.

Dans OpenShift GitOps, la bonne pratique consiste à activer les plugins alpha directement via la ressource `ArgoCD` (gérée par l’opérateur).  
Cela se fait en ajoutant l’option `--enable-alpha-plugins` dans la section `repo.buildOptions` et en copiant le plugin `PolicyGenerator` dans le repertoire approprié :

```
oc create -f manifests/argocd-instance.yaml
```

## 2. App-of-Apps

- `apps/root-app.yaml` est l’application racine Argo CD.
- Elle référence toutes les sous-applications : GitOps, Grafana, MCO.
- Chaque sous-application déploie ses propres manifests depuis `policies/<policy-name>`.

```bash
# Créer l'application root dans le cluster
oc create -f manifests/apps/root-app.yaml

# Synchroniser l'application root pour déployer toutes les sous-applications
argocd app sync root-app
```

## 3. Vérifications

1 - Il faut s'assurer que les applications ArgoCD sont bien synchrnonisés.
2 - Depuis un cluster managé, consulter la page `Observe` > `Metrics`. Vérifier que la metrique `catalog_processed_entities_count_total` est disponible.
3 - Depuis le dashboard Grafana dans ACM, vérifier que ces métriques sont présentes.

## 4. Commandes utiles

- récuperer le mdp Argo CD :
```bash
oc extract secret/argocd-observability-cluster  -n observability --to=-

```

- Vérifier les applications Argo CD :
```bash
oc get applications -n observability
```

- Synchroniser manuellement une application :
```bash
argocd app sync <app-name>
```

- Voir l’état d’une application :
```bash
argocd app get <app-name>
```

---

## Fonctionnement des Policies dans ACM

Les **policies ACM** permettent de définir des règles déclaratives qui doivent être respectées sur les clusters gérés. Elles peuvent :

- Vérifier la présence ou la configuration de ressources (ex: opérateurs, namespaces, ConfigMaps).
- Déclencher des actions correctives automatiquement via le paramètre `remediationAction` (`enforce` pour appliquer la configuration, `inform` pour alerter sans changer).
- Être associées à des **Placement** pour cibler un ou plusieurs clusters.
- Être regroupées et orchestrées via des générateurs de policies (`PolicyGenerator`) pour simplifier la création et la gestion de multiples policies similaires.

Grâce aux policies, ACM assure que l’état réel du cluster reste aligné avec l’état souhaité défini dans Git, garantissant **conformité, sécurité et automatisation**.
