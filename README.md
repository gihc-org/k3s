# k3s på Raspberry Pi

## Fejl ved installation

Efter installation af k3s fejlede servicen med:

```
Job for k3s.service failed because the control process exited with error code.
```

## Fejlfinding

`journalctl -xeu k3s.service` viste den præcise fejlbesked:

```
level=fatal msg="Error: failed to find memory cgroup (v2)"
```

En gennemgang af `/proc/cgroups` bekræftede at `memory`-controlleren slet ikke var listet — den var aldrig aktiveret af kernen.

Dette er et kendt problem på Raspberry Pi OS, hvor memory cgroup ikke er aktiveret som standard.

## Løsning

Tilføj `cgroup_memory=1 cgroup_enable=memory` til kernel-parametrene i `/boot/firmware/cmdline.txt`:

```bash
sudo sed -i 's/$/ cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
```

Genstart herefter maskinen:

```bash
sudo reboot
```

Efter genstart starter k3s automatisk. Verificér med:

```bash
systemctl status k3s.service
sudo k3s kubectl get nodes
```

## Bemærk

Hvis `/boot/firmware/cmdline.txt` overskrives af en fremtidig OS-opdatering, skal parametrene tilføjes igen.

---

## Kubernetes — grundlæggende begreber

Manifesterne nedenfor findes i nummererede undermapper der afspejler tilstanden på det tidspunkt emnet blev introduceret:

```
01-deployment/   → Deployment og Service (NodePort)
02-configmap/    → tilføjer ConfigMap, Secret og volumes
03-namespaces/   → tilføjer namespace: webapps
04-ingress/      → skifter til ClusterIP og tilføjer Ingress
```

### YAML-filer (manifests)

Kubernetes styres ved at beskrive den ønskede tilstand i YAML-filer, som kaldes **manifests**. I stedet for at fortælle Kubernetes *hvad det skal gøre* (imperativt), fortæller du det *hvad du vil have* (deklarativt). Kubernetes sørger selv for at virkeligheden matcher beskrivelsen.

Kører du den samme kommando igen uden at have ændret filen, sker der ingenting — Kubernetes registrerer at tilstanden allerede matcher.

Én fil kan indeholde flere ressourcer adskilt af `---`.

**Anvend et manifest:**
```bash
sudo k3s kubectl apply -f filnavn.yaml
```

**Slet en ressource:**
```bash
sudo k3s kubectl delete -f filnavn.yaml
```

### Pod

Den mindste enhed i Kubernetes. En Pod indeholder én eller flere containers der deler netværk og storage. Du opretter sjældent Pods direkte — det gøres typisk via en Deployment.

**Inspicér kørende Pods:**
```bash
sudo k3s kubectl get pods
sudo k3s kubectl describe pod <pod-navn>
sudo k3s kubectl logs <pod-navn>
```

### Deployment

En Deployment beskriver hvordan en applikation skal køre: hvilket container-image der bruges, hvor mange kopier (replicas) der skal køre, og hvordan opdateringer håndteres. Hvis en Pod crasher, opretter Kubernetes automatisk en ny for at opretholde det ønskede antal replicas.

Manifest: [`01-deployment/nginx.yaml`](01-deployment/nginx.yaml)

**Anvend:**
```bash
sudo k3s kubectl apply -f 01-deployment/nginx.yaml
```

**Inspicér:**
```bash
sudo k3s kubectl get deployments
sudo k3s kubectl describe deployment nginx
```

**Verificér:** Pod'en skal have status `Running`:
```bash
sudo k3s kubectl get pods
```

### Service

En Service eksponerer en Deployment for netværkstrafik. Pods får tilfældige IP-adresser der skifter, når de genskabes — en Service giver et stabilt netværkspunkt foran dem. Der findes flere typer:

| Type | Beskrivelse |
|------|-------------|
| `ClusterIP` | Kun tilgængelig inde i clusteret (standard) |
| `NodePort` | Eksponerer en fast port på selve noden, tilgængelig udefra |
| `LoadBalancer` | Opretter en ekstern load balancer (kræver cloud-udbyder eller MetalLB) |

**Inspicér:**
```bash
sudo k3s kubectl get services
sudo k3s kubectl describe service nginx
```

**Verificér:** Med NodePort på port 30080:
```bash
curl http://localhost:30080
```

### Labels og selectors

Labels er nøgle/værdi-par der sættes på ressourcer, f.eks. `app: nginx`. En Service bruger en selector til at finde de Pods den skal sende trafik til — alle Pods med et matchende label modtager trafik. Det er denne mekanisme der kobler en Service og en Deployment sammen.

**Inspicér labels på Pods:**
```bash
sudo k3s kubectl get pods --show-labels
```

### Container-images og registries

Kubernetes henter container-images fra et **container registry**. Når du skriver `image: nginx:alpine`, slår Kubernetes op i **Docker Hub** (hub.docker.com) som er standard-registryet. Formatet er:

```
[registry/][bruger/]image[:tag]

nginx:alpine               → Docker Hub, officielt nginx-image, alpine-variant
ubuntu:24.04               → Docker Hub, officielt Ubuntu-image
ghcr.io/bruger/app:v1.0   → GitHub Container Registry
myregistry.com/app:latest  → privat registry
```

Udelades tag (f.eks. `nginx` uden `:alpine`), bruges `:latest` automatisk. I produktion bør man altid angive et specifikt tag for at undgå uventede opdateringer.

### ConfigMap

En ConfigMap gemmer konfigurationsdata som nøgle/værdi-par — f.eks. en konfigurationsfil, en HTML-fil eller en app-indstilling. Data er ikke krypteret og må ikke indeholde følsomme oplysninger. ConfigMaps kan injiceres i en container på to måder:

- **Som en fil** — monteres på en sti inde i containeren via et volume
- **Som en miljøvariabel** — værdien bliver tilgængelig som en `$VARIABEL` inde i containeren

Manifest: [`02-configmap/configmap.yaml`](02-configmap/configmap.yaml)

**Anvend:**
```bash
sudo k3s kubectl apply -f 02-configmap/configmap.yaml
```

**Inspicér:**
```bash
sudo k3s kubectl get configmaps
sudo k3s kubectl describe configmap nginx-config
```

### Secret

En Secret fungerer som en ConfigMap, men er beregnet til følsomme data som passwords, API-nøgler og certifikater. Kubernetes base64-koder indholdet automatisk, men det er ikke kryptering — Secrets bør beskyttes med adgangskontrol (RBAC) i produktion. Secrets injiceres i containers på samme måde som ConfigMaps: som filer eller miljøvariabler.

Manifest: [`02-configmap/secret.yaml`](02-configmap/secret.yaml)

**Anvend:**
```bash
sudo k3s kubectl apply -f 02-configmap/secret.yaml
```

**Inspicér:**
```bash
sudo k3s kubectl get secrets
```

**Verificér at Secret er tilgængelig som miljøvariabel inde i containeren:**
```bash
sudo k3s kubectl exec <pod-navn> -- env | grep DB_PASSWORD
```

### Volumes og volumeMounts

Et **volume** er en datakilde der kan monteres ind i en container — f.eks. en ConfigMap, en Secret eller et stykke disk-storage. Et **volumeMount** beskriver hvor i containerens filsystem volumet skal monteres.

De kobles sammen ved hjælp af et navn:

```yaml
volumeMounts:
  - name: html                        # Refererer til volumet nedenfor
    mountPath: /usr/share/nginx/html  # Stien inde i containeren

volumes:
  - name: html                        # Samme navn som i volumeMount
    configMap:
      name: nginx-config              # Indholdet hentes fra denne ConfigMap
```

Når en ConfigMap eller Secret monteres som et volume, bliver hver nøgle til en fil — nøglenavnet bliver filnavnet og værdien bliver filindholdet.

Manifest: [`02-configmap/nginx.yaml`](02-configmap/nginx.yaml)

**Verificér at filen er monteret korrekt inde i containeren:**
```bash
sudo k3s kubectl exec <pod-navn> -- cat /usr/share/nginx/html/index.html
```

### Namespaces

Et namespace er en logisk opdeling af et cluster. Ressourcer i forskellige namespaces er isolerede fra hinanden — to Deployments i hvert sit namespace kan have samme navn uden at konflikte.

Kubernetes opretter fire namespaces som standard:

| Namespace | Formål |
|-----------|--------|
| `default` | Bruges hvis intet namespace angives |
| `kube-system` | Kubernetes' egne systemkomponenter |
| `kube-public` | Offentligt tilgængeligt data, sjældent brugt |
| `kube-node-lease` | Bruges internt til at registrere om noder er tilgængelige |

Namespaces bruges typisk til at gruppere ressourcer der hører logisk sammen — f.eks. alle web-applikationer i ét namespace og monitoring-værktøjer i et andet. Det gør det nemmere at få overblik og styre adgang pr. gruppe.

Et namespace angives i `metadata` på en ressource:

```yaml
metadata:
  name: nginx
  namespace: webapps
```

En ressource kan kun se ConfigMaps og Secrets i sit eget namespace. Namespace skal eksistere før ressourcer oprettes i det — anvend derfor `namespace.yaml` før de øvrige manifests.

Manifests: [`03-namespaces/`](03-namespaces/)

**Anvend:**
```bash
sudo k3s kubectl apply -f 03-namespaces/namespace.yaml
sudo k3s kubectl apply -f 03-namespaces/configmap.yaml
sudo k3s kubectl apply -f 03-namespaces/secret.yaml
sudo k3s kubectl apply -f 03-namespaces/nginx.yaml
```

**Inspicér:**
```bash
sudo k3s kubectl get namespaces
sudo k3s kubectl get all -n webapps
```

**Namespaces vs. separate clusters:** Namespaces giver logisk adskillelse, men ikke fuld isolation. Til adskillelse af `development` og `production` bruger man typisk separate clusters — en fejl i ét miljø kan da ikke påvirke det andet.

### Ingress

En Ingress er regler der styrer hvordan ekstern HTTP/HTTPS-trafik routes til Services inde i clusteret. I stedet for at hver Service eksponerer sin egen port udadtil, går al trafik ind gennem ét fælles indgangspunkt.

Ingress kræver en **Ingress-controller** — en komponent der læser Ingress-reglerne og rent faktisk håndterer trafikken. k3s leveres med **Traefik** som Ingress-controller, der lytter på port 80 og 443.

Ingress-controlleren er ikke nok alene — den ved ikke hvor trafikken skal hen. Et Ingress-manifest definerer reglerne:

```yaml
rules:
  - http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: nginx
              port:
                number: 80
```

Med Ingress bruges `ClusterIP` som Service-type i stedet for `NodePort`.

Manifests: [`04-ingress/`](04-ingress/)

**Anvend:**
```bash
sudo k3s kubectl apply -f 04-ingress/namespace.yaml
sudo k3s kubectl apply -f 04-ingress/configmap.yaml
sudo k3s kubectl apply -f 04-ingress/secret.yaml
sudo k3s kubectl apply -f 04-ingress/nginx.yaml
sudo k3s kubectl apply -f 04-ingress/ingress.yaml
```

**Inspicér:**
```bash
sudo k3s kubectl get ingress -n webapps
sudo k3s kubectl describe ingress nginx -n webapps
```

**Verificér:**
```bash
curl http://localhost
```

#### Hvorfor Ingress/ClusterIP er bedre end NodePort

Med NodePort skal hver Service have sin egen unikke port i intervallet 30000–32767. Med to applikationer er det til at overskue, men forestil dig ti applikationer:

```
nginx:       http://<ip>:30080
api:         http://<ip>:30081
dashboard:   http://<ip>:30082
auth:        http://<ip>:30083
...
```

Det giver tre problemer:
- **Portkonflikter** — du skal holde styr på hvilke porte der er i brug og manuelt undgå overlap
- **Brugervenlighed** — brugere og systemer skal kende det specifikke portnummer for hver applikation
- **Ingen HTTPS** — NodePort eksponerer en rå TCP-port; TLS skal konfigureres individuelt i hver applikation

Med Ingress ser det i stedet sådan ud:

```
nginx:       http://<ip>/
api:         http://<ip>/api
dashboard:   http://<ip>/dashboard
auth:        http://<ip>/auth
```

Al trafik går ind på port 80/443. Ingress-controlleren læser URL-stien og router trafikken til den rigtige Service internt. Nye applikationer tilføjes ved at tilføje en regel i Ingress-manifestet — ingen porte at holde styr på, ingen konflikter.

### RBAC

RBAC (Role-Based Access Control) styrer hvem der må gøre hvad i clusteret. Uden eksplicit tildelte rettigheder kan en Pod ikke tilgå nogen Kubernetes-ressourcer. Princippet er **least privilege** — en Pod får kun præcis de rettigheder den har brug for.

RBAC bygger på fire ressourcer:

| Ressource | Beskrivelse |
|-----------|-------------|
| `ServiceAccount` | En identitet for processer der kører i Pods |
| `Role` | Definerer tilladte handlinger på ressourcer inden for ét namespace |
| `ClusterRole` | Som Role, men gælder på tværs af alle namespaces |
| `RoleBinding` | Knytter en Role til en ServiceAccount i ét namespace |
| `ClusterRoleBinding` | Knytter en ClusterRole til en ServiceAccount på tværs af namespaces |

En `Role` definerer rettigheder via `rules` med tre felter:
- `apiGroups` — hvilken API-gruppe ressourcen tilhører (`""` er core-gruppen med Pods, Services, ConfigMaps osv.)
- `resources` — hvilke ressourcetyper reglen gælder for
- `verbs` — tilladte handlinger: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`

En ServiceAccount tilknyttes en Pod via `serviceAccountName` i Deployment'ens spec:

```yaml
spec:
  serviceAccountName: nginx-sa
```

Manifest: [`05-rbac/`](05-rbac/)

**Anvend:**
```bash
sudo k3s kubectl apply -f 05-rbac/namespace.yaml
sudo k3s kubectl apply -f 05-rbac/configmap.yaml
sudo k3s kubectl apply -f 05-rbac/secret.yaml
sudo k3s kubectl apply -f 05-rbac/rbac.yaml
sudo k3s kubectl apply -f 05-rbac/nginx.yaml
sudo k3s kubectl apply -f 05-rbac/ingress.yaml
```

**Inspicér:**
```bash
sudo k3s kubectl get serviceaccounts -n webapps
sudo k3s kubectl get roles -n webapps
sudo k3s kubectl get rolebindings -n webapps
```

**Verificér at Pod'en bruger den korrekte ServiceAccount:**
```bash
sudo k3s kubectl get pod <pod-navn> -n webapps -o jsonpath='{.spec.serviceAccountName}'
```

### PersistentVolumes og PersistentVolumeClaims

Containerens filsystem er **ephemeral** — alt data der skrives inde i en container går tabt når Pod'en genstarter. PersistentVolumes løser dette ved at koble ekstern lagerplads til en container.

To ressourcer arbejder sammen:

| Ressource | Beskrivelse |
|-----------|-------------|
| `PersistentVolume (PV)` | Det faktiske stykke lagerplads — lokal disk, NFS, cloud-storage osv. Cluster-niveau ressource |
| `PersistentVolumeClaim (PVC)` | En anmodning om lagerplads fra en Pod. Kubernetes finder og binder en passende PV |

k3s leveres med StorageClass'en `local-path` der automatisk opretter PVs på nodens lokale disk (**dynamisk provisionering**). Det betyder vi kun behøver at oprette en PVC — k3s klarer PV'en selv.

En PVC monteres i en Deployment som et volume — på samme måde som en ConfigMap, men med `persistentVolumeClaim` i stedet for `configMap`:

```yaml
volumes:
  - name: logs
    persistentVolumeClaim:
      claimName: nginx-logs     # Refererer til PVC'en
```

**Access modes** styrer hvordan lagerplads deles:

| Mode | Beskrivelse |
|------|-------------|
| `ReadWriteOnce` | Kun én node må skrive ad gangen (typisk til lokal disk) |
| `ReadOnlyMany` | Mange noder må læse samtidigt |
| `ReadWriteMany` | Mange noder må skrive samtidigt (kræver netværksbaseret storage, f.eks. NFS) |

Manifest: [`06-persistentvolumes/`](06-persistentvolumes/)

**Anvend:**
```bash
sudo k3s kubectl apply -f 06-persistentvolumes/pvc.yaml
sudo k3s kubectl apply -f 06-persistentvolumes/nginx.yaml
```

**Inspicér:**
```bash
sudo k3s kubectl get pvc -n webapps
sudo k3s kubectl get pv
```

**Verificér** at PVC har status `Bound` og at logfiler ligger persistent:
```bash
sudo k3s kubectl exec -n webapps <pod-navn> -- ls /var/log/nginx
# → access.log  error.log
```

---

## Rettelse af UTF-8 locale (æøå i terminalen)

### Problem

Kommandoer som `less` viste danske tegn som rå bytes, f.eks. `p<C3><A5>` i stedet for `på`. Fejlen skyldtes at `da_DK.UTF-8` var sat i flere `LC_*`-variabler, men localen var ikke genereret på systemet.

### Fejlfinding

```bash
locale
# → "Cannot set LC_ALL to default locale: No such file or directory"

locale -a | grep da_DK
# → (ingen output)

grep da_DK /etc/locale.gen
# → # da_DK ISO-8859-1
# → # da_DK.UTF-8 UTF-8   ← kommenteret ud
```

### Løsning

Uncomment `da_DK.UTF-8` i `/etc/locale.gen` og generér localen:

```bash
sudo sed -i 's/# da_DK.UTF-8 UTF-8/da_DK.UTF-8 UTF-8/' /etc/locale.gen
sudo locale-gen
```

Output skal vise:
```
Generating locales (this might take a while)...
  da_DK.UTF-8... done
  en_GB.UTF-8... done
Generation complete.
```

Verificér med `locale -a | grep da_DK` — den skal returnere `da_DK.utf8`.

---

## SSH-agent til Claude Code

Claude Code kører kommandoer i et ikke-interaktivt miljø uden en terminal. SSH kan derfor ikke prompte om en passphrase, og `git push` fejler med "Permission denied" selvom nøgle og config er korrekt sat op.

### Løsning

Start en SSH-agent og load nøglen inden Claude Code startes:

```bash
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_ed25519.github
```

Indtast passphrasen én gang. Start derefter Claude Code fra den samme shell — den arver `SSH_AUTH_SOCK` og kan pushe til GitHub uden interaktiv input.

### Hvad `eval $(ssh-agent -s)` gør

`ssh-agent -s` starter en agent som en baggrundsproces og printer:
```
SSH_AUTH_SOCK=/tmp/ssh-XXXXX/agent.1234; export SSH_AUTH_SOCK;
SSH_AGENT_PID=1234; export SSH_AGENT_PID;
```

`eval` udfører dette output som shell-kommandoer i den aktuelle session, så `SSH_AUTH_SOCK` og `SSH_AGENT_PID` sættes som miljøvariabler. Uden `eval` ville agenten køre i baggrunden, men din shell — og Claude Code — ville ikke vide hvor den er.
