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
12. RBAC

Næste emner på listen (ikke dokumenteret endnu):
- PersistentVolumes og PersistentVolumeClaims
- ResourceQuotas og LimitRanges

## Manifests
Manifests organiseres i to lag:

- **Roden** (`nginx.yaml`, `ingress.yaml` osv.) — de aktuelle, kørende manifests
- **Nummererede undermapper** — snapshot af manifesternes tilstand da emnet blev introduceret:
  - `01-deployment/` — Deployment og Service (NodePort)
  - `02-configmap/` — tilføjer ConfigMap, Secret og volumes
  - `03-namespaces/` — tilføjer namespace: webapps
  - `04-ingress/` — skifter til ClusterIP og tilføjer Ingress
  - `05-rbac/` — tilføjer ServiceAccount, Role og RoleBinding

Når et nyt emne introducerer ændringer til eksisterende manifests, oprettes en ny nummereret mappe med alle relevante filer i deres tilstand på det tidspunkt.

Hvert afsnit i README indeholder:
1. Forklaring af konceptet
2. Link til relevant manifest-mappe
3. kubectl-kommandoer: **Anvend**, **Inspicér**, **Verificér**

Nye manifests dokumenteres linje for linje med kommentarer direkte i YAML-filen.

## Git
- Commit og push efter hvert afsluttet emne
- Commit-beskeder på dansk
