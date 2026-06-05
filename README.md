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

### YAML-filer (manifests)

Kubernetes styres ved at beskrive den ønskede tilstand i YAML-filer, som kaldes **manifests**. I stedet for at fortælle Kubernetes *hvad det skal gøre* (imperativt), fortæller du det *hvad du vil have* (deklarativt). Kubernetes sørger selv for at virkeligheden matcher beskrivelsen.

En YAML-fil sendes til Kubernetes med:
```bash
sudo k3s kubectl apply -f filnavn.yaml
```

Kører du den samme kommando igen uden at have ændret filen, sker der ingenting — Kubernetes registrerer at tilstanden allerede matcher.

Én fil kan indeholde flere ressourcer adskilt af `---`.

### Pod

Den mindste enhed i Kubernetes. En Pod indeholder én eller flere containers der deler netværk og storage. Du opretter sjældent Pods direkte — det gøres typisk via en Deployment.

### Deployment

En Deployment beskriver hvordan en applikation skal køre: hvilket container-image der bruges, hvor mange kopier (replicas) der skal køre, og hvordan opdateringer håndteres. Hvis en Pod crasher, opretter Kubernetes automatisk en ny for at opretholde det ønskede antal replicas.

### Service

En Service eksponerer en Deployment for netværkstrafik. Pods får tilfældige IP-adresser der skifter, når de genskabes — en Service giver et stabilt netværkspunkt foran dem. Der findes flere typer:

| Type | Beskrivelse |
|------|-------------|
| `ClusterIP` | Kun tilgængelig inde i clusteret (standard) |
| `NodePort` | Eksponerer en fast port på selve noden, tilgængelig udefra |
| `LoadBalancer` | Opretter en ekstern load balancer (kræver cloud-udbyder eller MetalLB) |

### Labels og selectors

Labels er nøgle/værdi-par der sættes på ressourcer, f.eks. `app: nginx`. En Service bruger en selector til at finde de Pods den skal sende trafik til — alle Pods med et matchende label modtager trafik. Det er denne mekanisme der kobler en Service og en Deployment sammen.

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

### Secret

En Secret fungerer som en ConfigMap, men er beregnet til følsomme data som passwords, API-nøgler og certifikater. Kubernetes base64-koder indholdet automatisk, men det er ikke kryptering — Secrets bør beskyttes med adgangskontrol (RBAC) i produktion. Secrets injiceres i containers på samme måde som ConfigMaps: som filer eller miljøvariabler.

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
