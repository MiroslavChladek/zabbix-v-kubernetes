# Zabbix v Kubernetes – návod na instalaci za 5 minut

Průvodce instalací základního Zabbix stacku do K3s pomocí Helmu, jejímž cílem je seznámení se s novým přístupem a vyzkoušení prvních kroků v tomto prostředí.

VM: Rocky Linux 9.6, Hezner Cloud, CX33

Video: 

**Instalace je rozdělena do následujících kroků:**

- Konfigurace operačního systému: Instalace nezbytných závislostí.
- Nasazení Kubernetes a vytvoření clusteru.
- Instalace Helm a Zabbix repozitáře.
- Deployment - nasazení Zabbix stacku.
- Kontrola stavu.
- Zprovoznění přístupu - přesmětrování portu.

# Příprava operačního systému

Před samotnou inicializací clusteru je nutné zajistit přítomnost standardních nástrojů pro práci s archivy a GIT. Pro účely testovacího prostředí také deaktivujeme firewall, v produkčním nasazení je však nezbytné otevřít jen nezbytné porty.

### Instalace potřebnych nástrojů
```
dnf install git tar -y
```

### Deaktivace firewalld (doporučeno pouze pro lab/testovací účely)
Pro Hezner se nepoužije, VM ve výchozím stavu nemá firewall nainstalovaný.
```
systemctl disable firewalld.service --now
```


# Instalace K3s a Helm

Jako cílovou platformu jsem zvolil distribuci K3s. Ke správě aplikací využijeme Helm, který nám umožní definovat infrastrukturu i konfiguraci pomocí chartů namísto manuální tvorby YAML manifestů.

### Instalace distribuce K3s pomocí oficiálního skriptu
```
curl -sfL https://get.k3s.io | sh -
```
### Kontrola stavu
```
kubectl get all -n zabbix-namespace
```

### Nastavení konfiguračního souboru pro nástroj kubectl
```
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

### Instalace Helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```


# Konfigurace repozitářů Helm

Pro instalaci využijeme komunitní repozitář zabbix-community, který obsahuje odladěné Helm charty pro nasazení všech komponent systému Zabbix.

### Přidání a aktualizace repozitáře
```
helm repo add zabbix-community https://zabbix-community.github.io/helm-zabbix
helm repo update
```


# Deployment Zabbix stacku

Samotné nasazení probíhá do izolovaného jmenného prostoru (namespace), což usnadňuje správu oprávnění a oddělení od ostatních služeb běžících v clusteru.

### Instalace systému Zabbix do dedikovaného jmenného prostoru
```
helm install zabbix zabbix-community/zabbix --create-namespace -n zabbix-namespace
```


# Monitoring stavu a verifikace

### Sledování stavu podů v reálném čase
```
watch kubectl get pods -n zabbix-namespace
```

### Výpis všech prostředků ve jmenném prostoru
```
kubectl get all -n zabbix-namespace
```


# Nastavení přístupu k frontendu

Pro přístup k webovému rozhraní bez nutnosti konfigurovat Ingress využijeme port-forwarding, který přesměruje provoz z lokálního rozhraní VM přímo na příslušnou službu uvnitř clusteru.

### Přesměrování lokálního portu 8080 na službu webového rozhraní
```
kubectl port-forward service/zabbix-zabbix-web 8080:80 -n zabbix-namespace --address 0.0.0.0
```

**Konfigurace přístupu:**

- **URL:** http://<adresa-serveru>:8080 (http://zbx-aio-k3s-lab-2EE0426A.nip.io:8080)
- **Výchozí uživatel:** Admin
- **Výchozí heslo:** zabbix

# První kroky a orinetace

Tato sekce obsahuje základní příkazy pro rychlou orientaci v rámci jmenného prostoru `zabbix-namespace`. Tyto kroky vám pomohou identifikovat stav nasazených aplikací a odhalit případné chyby.

### Přehled o stavu prostředí
Nejdříve zjistíme, co v clusteru běží a v jaké je to kondici.

* `kubectl get pods -n zabbix-namespace`
  * **Výpis podů:** Zobrazí seznam všech kontejnerů a jejich stav (např. zda běží, nebo selhaly).
* `kubectl get all -n zabbix-namespace`
  * **Kompletní inventura:** Rychlý přehled všech objektů (pody, služby, deploymenty) na jednom místě.
* `kubectl get pods --output=wide -n zabbix-namespace`
  * **Rozšířený výpis:** Zobrazí navíc IP adresy podů a názvy uzlů (node), na kterých běží.

### Detailní diagnostika
Pokud narazíte na problém, použijte tyto příkazy pro hlubší analýzu.

* `kubectl describe pods/POD_NAME -n zabbix-namespace`
  * **Technické detaily:** Zobrazí specifikaci podu a historii událostí (vhodné pro hledání příčin, proč pod nenastartoval).
* `kubectl logs -f pods/POD_NAME -c CONTAINER_NAME -n zabbix-namespace`
  * **Sledování logů:** Výpis výstupu z aplikace v reálném čase (přepínač `-f` udržuje spojení).
* `kubectl get svc -n zabbix-namespace`
  * **Seznam služeb:** Přehled portů a IP adres, na kterých jsou aplikace dostupné.

### Interaktivní přístup a síť
Pro přímou kontrolu vnitřního nastavení.

* `kubectl exec -it pods/POD_NAME -c CONTAINER_NAME -n zabbix-namespace -- sh`
  * **Vstup do kontejneru:** Otevře terminál přímo uvnitř běžícího kontejneru.


> **Poznámka:** Nahraďte `POD_NAME` a `CONTAINER_NAME` skutečnými názvy, které získáte pomocí příkazu `get pods`.


# Odinstalace

Pro kompletní odstranění všech instalovaných komponent a uvolnění prostředků clusteru stačí odinstalovat příslušný Helm release.

### Odstranění Zabbix stacku a jmenného prostoru (namespace)
```
helm uninstall zabbix -n zabbix-namespace
kubectl delete namespace zabbix-namespace
```

# 💡 Dotazy a návrhy
Narazili jste na problém, nebo máte nápad, jak tuto sérii vylepšit? 

**Vaše podněty uvítám!**

Kromě technických dotazů rád uvítám i **vaše tipy na témata týkající se provozu Zabbixu v prostředí Kubernetes**. 
Napište mi také pokud vás zajímá konkrétní oblast (např. instalace modulů, konfigurace, HA a pdobně). 

**📧 [Kontaktní formulář](https://docs.google.com/forms/d/e/1FAIpQLSeezO3fqbadGtfJZY4bn8MVRbaEWU1PMXzj-xzlJxyFKqmWrw/viewform?usp=dialog)**.

# Související odkazy
- https://www.zabbix.com/container_images
- https://kubernetes.io/docs/home/
- https://k3s.io/
- https://helm.sh/
- https://github.com/zabbix-community/helm-zabbix
- https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/
- https://kubernetes.io/docs/reference/kubectl/quick-reference/

