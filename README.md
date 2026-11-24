# Mendelu seminář správa zranitelností

Prezentace a teoretická část separátně v souboru Prezentace_sprava_zranitelnosti.pdf.


# Praktické cvičení: Skenování zranitelností a hardening v Greenbone (OpenVAS)

Tento repozitář obsahuje prostředí pro nácvik skenování zranitelností, analýzy reportů a základního zabezpečení (hardeningu) linuxového serveru pomocí nástroje **Greenbone Community Edition** v Dockeru.

---

## Požadavky

- Nainstalovaný **Docker** a **Docker Compose** (v2)
- Alespoň **8 GB RAM** (doporučeno 16 GB) a cca **15 GB volného místa** na disku

---

## Struktura projektu

```text
.
├── greenbone-lab/
│   ├── docker-compose.yml          # Oficiální Greenbone stack (staženo skriptem/curl)
│   └── docker-compose.override.yml # Napojení skeneru do izolované sítě lab-net
└── target/
    ├── Dockerfile                  # Zranitelný Ubuntu kontejner (SSH + Apache)
    └── docker-compose.yml          # Definice cíle s pevnou IP adresou 172.28.0.10
```

---

## 1. Příprava a spuštění prostředí

### Krok 1: Vytvoření izolované Docker sítě
Vytvořte sdílenou síť `lab-net` s pevným rozsahem adres:

```bash
docker network create --subnet=172.28.0.0/24 lab-net
```

### Krok 2: Nasazení Greenbone
Přejděte do složky `greenbone-lab`, stáhněte oficiální `docker-compose.yml` a spusťte stack:

```bash
cd greenbone-lab
curl -f -L https://greenbone.github.io/docs/latest/_static/docker-compose.yml -o docker-compose.yml
docker compose up -d
```

> **Poznámka:** Při prvním spuštění se stahují a zpracovávají databáze zranitelností (feedy). Tento proces může trvat **20–40 minut**. Stav synchronizace můžete sledovat příkazem:
> ```bash
> docker compose logs -f data-objects
> ```
> Webové rozhraní je dostupné na adrese **http://localhost:9392** (výchozí přihlášení: `admin` / `admin`).

### Krok 3: Spuštění cílového Linuxu (`target-linux`)
V novém terminálu přejděte do složky `target` a spusťte cílový kontejner:

```bash
cd target
docker compose up -d --build
```

### Ověření funkčnosti
Zkontrolujte, zda skener vidí cílový kontejner na IP adrese `172.28.0.10`:

```bash
docker run --rm --network lab-net alpine ping -c 2 172.28.0.10
```

---

## 2. Průběh cvičení

### Úkol 1: Výchozí (baseline) sken
1. Přihlaste se do webového rozhraní Greenbone (`http://localhost:9392`).
   - **Username:** `admin`
   - **Password:** `admin`
2. V menu **Configuration -> Credentials** vytvořte nové SSH přihlašovací údaje:
   - **Name:** `Target Linux Root`
   - **Username:** `root`
   - **Password:** `student123`
3. V menu **Configuration -> Targets** vytvořte nový cíl:
   - **Name:** `Target Linux`
   - **Hosts:** `172.28.0.10`
   - **SSH Credential:** vyberte vytvořený `Target Linux Root` (port `22`)
4. V menu **Scans -> Tasks** vytvořte novou úlohu nad tímto cílem (Scan Config: `Full and fast`) a spusťte ji.
5. Po dokončení skenu (`Done`) otevřete vygenerovaný report a prostudujte nalezené zranitelnosti (zejména nastavení SSH, běžící služby a zastaralé balíčky).

---

### Úkol 2: Hardening cílového systému
Připojte se do běžícího cílového kontejneru:

```bash
docker exec -it target-linux bash
```

Proveďte následující kroky zabezpečení:

1. **Aktualizace systému a balíčků:**
   ```bash
   apt update && apt upgrade -y
   ```

2. **Zastavení nepotřebné služby (Apache HTTP server):**
   ```bash
   service apache2 stop
   ```

3. **Zabezpečení konfigurace SSH (`/etc/ssh/sshd_config`):**
   - Zakažte přihlášení uživatele `root` (`PermitRootLogin no`)
   - Odstraňte nebo zakomentujte řádky se slabými šiframi (`Ciphers` a `MACs`), aby se použila bezpečná výchozí nastavení OpenSSH
   - Změňte heslo uživatele `root` příkazem `passwd`
   - Restartujte SSH službu:
     ```bash
     service ssh restart
     ```

---

### Úkol 3: Kontrolní sken a porovnání výsledků
1. V Greenbone v sekci **Scans -> Tasks** spusťte stejnou skenovací úlohu znovu.
2. Po dokončení otevřete nový report a porovnejte jej s výchozím (baseline) skenem.
3. Ověřte, které zranitelnosti zmizely a zdůvodněte proč (včetně vlivu vypnutí `PermitRootLogin` na možnosti autentizovaného skenu).

---

## Obnovení výchozího stavu (Reset cíle)

Pokud chcete cílový Linux vrátit do původního zranitelného stavu:

```bash
cd target
docker compose down
docker compose up -d --build
```


## Úklid prostředí po cvičení

### 1. Zastavení a odstranění kontejnerů (a případně i dat)
Ve složce s cílovým Linuxem (`target`):
```bash
docker compose down
```

Ve složce s Greenbone (`greenbone-lab`) – pozor, přepínač `-v` smaže i volumes (včetně stažených feedů a dat skeneru):
```bash
docker compose down -v
```

### 2. Odstranění sítě `lab-net`
```bash
docker network rm lab-net
```

### 3. Volitelný úklid nepoužitých obrazů a cache
```bash
docker image prune -f
```