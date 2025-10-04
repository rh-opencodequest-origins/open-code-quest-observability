## Ce dossier est utilisé pour déployer la partie observabilité via GitOps et ACM (avec des policies).

Il met en place les policies ACM pour s'assurer que les manifests necessaire à l'observabilité sont bien présents sur le cluster. Cela nous donne une couche de gouvernance.

```
oc apply - f manifests/apps/root-app.yaml
```
