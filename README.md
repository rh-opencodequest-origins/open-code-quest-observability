# Repository pour le workshop Open Code Quest - la partie observabilité

## Contexte du Projet

Ce projet met en place une **stratégie GitOps complète** pour gérer l’infrastructure d'observabilité sur plusieurs clusters via **Red Hat Advanced Cluster Management (ACM)** et **Argo CD**.

## Ce que fait ce repo

Ce repo organise l’ensemble des politiques et applications de manière **hiérarchique et modulaire** :

- **Root App Argo CD (`apps/root-app.yaml`)** : orchestrateur central de toutes les sous-applications.
- **Sous-applications Argo CD** :
  - `gitops-policy-app.yaml` → déploie Argo CD avec ConfigMap pour plugins alpha
  - `grafana-policy-app.yaml` → déploie Grafana Operator
  - `mco-policy-app.yaml` → déploie MCO avec MinIO persistant
- **Dossiers `policies/*`** : contiennent les manifests ACM et les Kustomization pour chaque policy.

---

## Structure du Repo

```
gitops-repo/
├── apps/                     # Applications Argo CD (App-of-Apps)
│   ├── root-app.yaml          # Application racine qui orchestre toutes les sous-applications
│   ├── gitops-policy-app.yaml
│   ├── grafana-policy-app.yaml
│   └── mco-policy-app.yaml
├── policies/                 # Policies ACM
│   ├── gitops-policy/         # Installation de GitOps avec ConfigMap alpha
│   ├── grafana-policy/        # Installation du Grafana Operator
│   └── mco-policy/            # Installation MCO + MinIO persistent
```

---

## 1. Déploiement d'Argo CD

1. Vérifier que l’opérateur GitOps est installé via ACM (`openshift-gitops` namespace).
2. Obtenir l’URL de l’UI Argo CD :

```bash
oc get route -n openshift-gitops
```

3. Récupérer le mot de passe admin :

```bash
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

---

## 2. App-of-Apps

- `apps/root-app.yaml` est l’application racine Argo CD.
- Elle référence toutes les sous-applications : GitOps, Grafana, MCO.
- Chaque sous-application déploie ses propres manifests depuis `policies/<policy-name>`.

```bash
# Créer l'application root dans le cluster via kubectl
oc apply -f apps/root-app.yaml -n openshift-gitops

# Synchroniser l'application root pour déployer toutes les sous-applications
argocd app sync root-app
```

---

---

## 3. Multi-Cluster

- Chaque application peut être ciblée vers un ou plusieurs clusters via `placementDefaults` et `clusterSelectors`.
- Vous pouvez dupliquer les sous-applications pour différents clusters en changeant le label `local-cluster: true` ou un autre label cluster spécifique.

---

## 4. Commandes utiles

- Vérifier les applications Argo CD :
```bash
oc get applications -n openshift-gitops
```

- Synchroniser manuellement une application :
```bash
argocd app sync <app-name>
```

- Voir l’état d’une application :
```bash
argocd app get <app-name>
```

## 5. Fonctionnement des Policies dans ACM

Les **policies ACM** permettent de définir des règles déclaratives qui doivent être respectées sur les clusters gérés. Elles peuvent :

- Vérifier la présence ou la configuration de ressources (ex: opérateurs, namespaces, ConfigMaps).
- Déclencher des actions correctives automatiquement via le paramètre `remediationAction` (`enforce` pour appliquer la configuration, `inform` pour alerter sans changer).
- Être associées à des **PlacementRules** pour cibler un ou plusieurs clusters.
- Être regroupées et orchestrées via des générateurs de policies (`PolicyGenerator`) pour simplifier la création et la gestion de multiples policies similaires.

Grâce aux policies, ACM assure que l’état réel du cluster reste aligné avec l’état souhaité défini dans Git, garantissant **conformité, sécurité et automatisation**.
