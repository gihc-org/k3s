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
