# Instruktioner til Claude Code

## Formål
Dette repository bruges til at lære Kubernetes ved at bygge en k3s-installation op på en Raspberry Pi. Nye emner introduceres praktisk med YAML-manifests og dokumenteres efterfølgende i README.md.

## README.md — struktur og rækkefølge

Nye Kubernetes-emner tilføjes altid som underafsnit (`###`) under `## Kubernetes — grundlæggende begreber` i den rækkefølge de behandles.

Nuværende rækkefølge:
1. YAML-filer (manifests)
2. Pod
3. Deployment
4. Service
5. Labels og selectors
6. Container-images og registries
7. ConfigMap
8. Secret
9. Volumes og volumeMounts
10. Namespaces
11. Ingress

Næste emner på listen (ikke dokumenteret endnu):
- RBAC
- PersistentVolumes og PersistentVolumeClaims
- ResourceQuotas og LimitRanges

## Manifests
Alle YAML-manifests ligger i roden af repositoriet. Nye manifests dokumenteres linje for linje med kommentarer direkte i YAML-filen.

## Git
- Commit og push efter hvert afsluttet emne
- Commit-beskeder på dansk
