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
- Elle référence toutes les sous-applications : Grafana, MCO et le user workload monitoring (UW).
- Chaque sous-application déploie ses propres manifests depuis `policies/<policy-name>`.

```bash
# Créer l'application root dans le cluster
oc create -f manifests/apps/root-app.yaml

# Synchroniser l'application root pour déployer toutes les sous-applications
argocd app sync root-app
```

## 3. Ajout de clusters au projet d'observabilité

Pour qu'un cluster géré bénéficie de l'observabilité déployée par ce projet, il suffit de l'ajouter au ManagedClusterSet nommé `observability`. Cette opération s'effectue directement depuis l'interface web de Red Hat Advanced Cluster Management :

  1. Accédez à la console ACM via All Clusters
  2. Dans le menu de navigation, allez dans Infrastructure > Clusters
  3. Sélectionnez le cluster que vous souhaitez intégrer
  4. Dans l'onglet Cluster sets, cliquez sur Edit cluster sets
  5. Cochez le clusterset observability
  6. Validez en cliquant sur Save

  Une fois le cluster ajouté au clusterset observability, les policies ACM configurées dans ce projet (User Workload Monitoring, Grafana, MCO) seront automatiquement évaluées et appliquées sur ce cluster. Le ManagedClusterSetBinding présent dans le namespace observability permet aux Placement de cibler les clusters membres de ce clusterset. Vous pourrez
  ensuite visualiser les métriques de ce cluster dans les dashboards Grafana déployés par le projet.

## 4. Vérifications

1 - Il faut s'assurer que les applications ArgoCD sont bien synchrnonisés.
2 - Depuis un cluster managé, consulter la page `Observe` > `Metrics`. Vérifier que la metrique `catalog_processed_entities_count_total` est disponible.
3 - Depuis le dashboard Grafana dans ACM, vérifier que ces métriques sont présentes.

## 5. Modification des dashboards Grafana

Pour personnaliser les dashboards d'observabilité, il est nécessaire d'utiliser une instance Grafana de développement. Cette approche permet de tester et affiner les dashboards avant de les déployer en production via les policies ACM.

### Configuration de l'instance Grafana de développement

1. **Déployer une instance Grafana dev** :
   ```bash
   oc apply -f - <<EOF
   apiVersion: grafana.integreatly.org/v1beta1
   kind: Grafana
   metadata:
     name: grafana-dev
     namespace: open-cluster-management-observability
   spec:
     config:
       auth:
         disable_login_form: false
       auth.anonymous:
         enabled: true
     deployment:
       spec:
         template:
           spec:
             containers:
             - name: grafana
               env:
               - name: GF_SECURITY_ADMIN_USER
                 value: admin
               - name: GF_SECURITY_ADMIN_PASSWORD
                 value: admin
   EOF
   ```

2. **Configurer la datasource Thanos** pour accéder aux métriques MCO :
   ```bash
   oc apply -f - <<EOF
   apiVersion: grafana.integreatly.org/v1beta1
   kind: GrafanaDatasource
   metadata:
     name: observability-datasource
     namespace: open-cluster-management-observability
   spec:
     instanceSelector:
       matchLabels:
         dashboards: "grafana-dev"
     datasource:
       name: Observability-Thanos
       type: prometheus
       access: proxy
       url: https://observability-thanos-query-frontend.open-cluster-management-observability.svc:9090
       isDefault: true
       jsonData:
         httpHeaderName1: Authorization
         tlsSkipVerify: true
       secureJsonData:
         httpHeaderValue1: Bearer \$(oc sa get-token grafana-dev-sa -n open-cluster-management-observability)
   EOF
   ```

3. **Accéder à l'interface Grafana dev** :
   ```bash
   oc get route grafana-dev-route -n open-cluster-management-observability
   ```

4. **Créer et tester vos dashboards** dans l'interface Grafana, puis exporter le JSON du dashboard.

Pour plus de détails, consultez la [documentation officielle Red Hat ACM](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.11/html-single/observability/index#setting-up-the-grafana-developer-instance).

## 6. Commandes utiles

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
