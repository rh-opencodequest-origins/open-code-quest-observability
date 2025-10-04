## Ce dossier est utilisé pour déployer la partie observabilité via GitOps et ACM.

Il utilise le plugin Kustomize PolicyGenerator d'ACM.

https://github.com/open-cluster-management-io/policy-generator-plugin?tab=readme-ov-file#installing-the-policy-generator


Un manifest est disponible pour déployer et étendre les capacités d'ArgoCD avec le plugin PolicyGenerator.

```
oc apply - f manifests/argocd-instance.yaml

oc apply - f manifests/apps/root-app.yaml
```

Le plugin Policy Generator transforme une CRD `PolicyGenerator` en 3 CRDs `Placement`, `PlacementBinding` et `Policy`