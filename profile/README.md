# VPS Dokumentacija - Hostinger Setup

## 📋 Pregled Sistema

**VPS Provider:** Hostinger VPS KVM2  
**OS:** Ubuntu Server  
**Hostname:** srv744936  
**Datum dokumentacije:** 24. Novembar 2025

---

## 👥 Korisnici

- `root` - Administratorski nalog
- `ubuntu` - Sistemski nalog
- `uros` - Glavni razvojni nalog (Docker grupa)

---

## 🏗️ Arhitektura Sistema

### Docker Stack

Koristi se **Traefik v3.1** kao reverse proxy sa automatskim SSL sertifikatima (Let's Encrypt).

```
Internet
   ↓
[Traefik Reverse Proxy]
   ├── Port 80 (HTTP) → automatski redirect na HTTPS
   ├── Port 443 (HTTPS)
   │
   ├─→ suninnova.com (Nginx container)
   ├─→ ushrt.sunninova.com (Nginx container)
   └─→ utrainer.sunninova.com (Nginx container)
```

---

## 📁 Struktura Direktorijuma

```
/home/uros/
├── projects/
│   ├── proxy/                    # Traefik reverse proxy
│   │   ├── docker-compose.yml
│   │   └── (Traefik automatski generiše SSL)
│   │
│   ├── suninnova/               # Glavni sajt
│   │   ├── docker-compose.yml
│   │   └── public/              # HTML fajlovi
│   │
│   ├── ushrt/                   # Subdomen za testiranje
│   │   ├── docker-compose.yml
│   │   └── public/              # HTML fajlovi
│   │
│   └── utrainer/                # Subdomen za testiranje
│       ├── docker-compose.yml
│       └── public/              # HTML fajlovi
│
└── acme-backup-*.json           # Let's Encrypt SSL backup
```

---

## 🐳 Docker Konfiguracija

### Instalirana Verzija

- **Docker:** 28.4.0 (build d8eb465)
- **Docker Compose:** v2.39.2

### Docker Network

Svi kontejneri koriste zajedničku mrežu `web` za komunikaciju sa Traefik-om.

---

## 🔧 Traefik Reverse Proxy Setup

### Lokacija

`/home/uros/projects/proxy/docker-compose.yml`

### Konfiguracija

```yaml
version: "3.9"
services:
  traefik:
    image: traefik:v3.1
    container_name: traefik
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.httpchallenge=true
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.le.acme.email=ops@suninnova.com
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --api.dashboard=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - web
    restart: unless-stopped
networks:
  web:
    name: web
volumes:
  traefik_letsencrypt:
```

### Ključne Karakteristike

- ✅ Automatsko Let's Encrypt SSL obnovljavanje
- ✅ HTTP → HTTPS redirect
- ✅ Docker service discovery
- ✅ Email za notifikacije: ops@suninnova.com
- ✅ Backup SSL sertifikata u Docker volume

---

## 🌐 Web Aplikacije

### 1. SunInnova (Glavni Sajt)

**Domen:** suninnova.com  
**Lokacija:** `/home/uros/projects/suninnova/`  
**Status:** Testni HTML (priprema za CI/CD)

### 2. Ushrt

**Domen:** ushrt.sunninova.com  
**Lokacija:** `/home/uros/projects/ushrt/`  
**Status:** Testni HTML (priprema za CI/CD)

### 3. UTrainer

**Domen:** utrainer.sunninova.com  
**Lokacija:** `/home/uros/projects/utrainer/`  
**Status:** Testni HTML (priprema za CI/CD)

### Zajednička Docker Compose Struktura

Za svaku aplikaciju (`suninnova`, `ushrt`, `utrainer`):

```yaml
version: "3.9"
services:
  web:
    image: nginx:stable-alpine
    container_name: [ime-aplikacije]-web-1
    volumes:
      - ./public:/usr/share/nginx/html:ro
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.[ime].rule=Host(`[domen]`)"
      - "traefik.http.routers.[ime].entrypoints=websecure"
      - "traefik.http.routers.[ime].tls.certresolver=le"
    restart: unless-stopped

networks:
  web:
    external: true
```

---

## 🔐 SSL/TLS Sertifikati

### Let's Encrypt

- **Provider:** Let's Encrypt (via Traefik)
- **Challenge Type:** HTTP-01
- **Auto-renewal:** Omogućeno
- **Storage:** Docker volume `traefik_letsencrypt`
- **Backup fajlovi:** `/home/uros/acme-backup-*.json`

### Domenski Sertifikati

- suninnova.com
- ushrt.sunninova.com
- utrainer.sunninova.com

---

## ⚙️ Sistemski Servisi

### Ključni Servisi (systemctl)

```
containerd.service     - Container runtime
docker.service         - Docker daemon
ssh.service            - OpenBSD SSH server
cron.service           - Cron daemon
dbus.service           - D-Bus System Message Bus
monarx-agent.service   - Security Scanner (Hostinger)
systemd-networkd       - Network Configuration
systemd-resolved       - DNS Resolution
```

---

## 🚀 Kako Pokrenuti Sistem od Nule

### 1. Osnovno Postavljanje Servera

```bash
# Update sistema
sudo apt update && sudo apt upgrade -y

# Instalacija osnovnih alata
sudo apt install -y curl git vim ufw

# Kreiranje korisnika
sudo adduser uros
sudo usermod -aG sudo uros
```

### 2. Instalacija Docker-a

```bash
# Dodavanje Docker repositorija
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Dodavanje korisnika u docker grupu
sudo usermod -aG docker uros

# Provera instalacije
docker --version
docker compose version
```

### 3. Konfiguracija Firewall-a

```bash
# Omogući UFW
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 4. Podešavanje Strukture Projekata

```bash
# Kao korisnik 'uros'
mkdir -p ~/projects/{proxy,suninnova,ushrt,utrainer}

# Za svaki projekat
cd ~/projects/suninnova
mkdir public
```

### 5. Deploy Traefik Reverse Proxy

```bash
cd ~/projects/proxy

# Napravi docker-compose.yml (koristiti config iz ovog dokumenta)
nano docker-compose.yml

# Pokreni Traefik
docker compose up -d

# Proveri status
docker compose ps
docker compose logs -f
```

### 6. Deploy Web Aplikacija

Za svaku aplikaciju (`suninnova`, `ushrt`, `utrainer`):

```bash
cd ~/projects/[ime-aplikacije]

# Napravi docker-compose.yml
nano docker-compose.yml

# Dodaj testni HTML
echo "<h1>Test - [Ime Aplikacije]</h1>" > public/index.html

# Pokreni kontejner
docker compose up -d

# Proveri
docker compose ps
docker compose logs -f
```

### 7. Provera DNS Podešavanja

Pre deploy-a, osiguraj da DNS A zapisi pokazuju na VPS IP:

```
suninnova.com           → [VPS-IP]
ushrt.sunninova.com     → [VPS-IP]
utrainer.sunninova.com  → [VPS-IP]
```

### 8. Testiranje

```bash
# Proveri sve kontejnere
docker ps

# Proveri SSL
curl -I https://suninnova.com
curl -I https://ushrt.sunninova.com
curl -I https://utrainer.sunninova.com

# Proveri Traefik sertifikate
docker exec traefik cat /letsencrypt/acme.json | grep -o '"domain":"[^"]*"'
```

---

## 🔄 Planirana CI/CD Integracija

### Trenutno Stanje

- ✅ Traefik reverse proxy sa SSL
- ✅ Docker Compose setup za svaku aplikaciju
- ✅ Testni HTML sajt u svakom `public/` folderu

### Sledeći Koraci

1. **Git Repository Setup**

   - Povezati svaki projekat sa Git repozitorijumom
   - Podesiti `.gitignore` fajlove

2. **GitHub Actions / GitLab CI**

   - Automatsko build-ovanje Docker image-a
   - Push na Docker registry (Docker Hub ili privatni)
   - Deploy na VPS preko SSH

3. **Docker Image Strategy**
   - Trenutno: Nginx servira statički `public/` folder
   - Buduće: Custom Docker image sa aplikacijom

### Primer CI/CD Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to VPS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_HOST }}
          username: uros
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/projects/suninnova
            git pull
            docker compose down
            docker compose up -d
```

---

## 📦 Backup Strategija

### SSL Sertifikati

- **Automatski backup:** `/home/uros/acme-backup-*.json`
- **Frekvencija:** Multiple snapshots (9. Sep 2025)
- **Lokacija:** Docker volume `traefik_letsencrypt`

### Preporuka za Backup

```bash
# Backup skripta (dodati u cron)
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M-%S)
docker run --rm -v traefik_letsencrypt:/letsencrypt \
  -v /home/uros/backups:/backup \
  alpine tar czf /backup/letsencrypt-$DATE.tar.gz -C / letsencrypt

# Backup svih docker-compose fajlova
tar czf /home/uros/backups/docker-configs-$DATE.tar.gz \
  /home/uros/projects/*/docker-compose.yml
```

---

## 🛠️ Korisne Komande

### Docker Management

```bash
# Prikaz svih kontejnera
docker ps -a

# Prikaz logova
docker compose logs -f [servis-ime]

# Restart servisa
docker compose restart

# Zaustavi sve
docker compose down

# Pokreni sve
docker compose up -d

# Očisti nekorišćene resurse
docker system prune -a
```

### Traefik Management

```bash
# Provera Traefik logova
docker logs traefik -f

# Provera SSL sertifikata
docker exec traefik cat /letsencrypt/acme.json

# Restart Traefik
cd ~/projects/proxy
docker compose restart
```

### Debugging

```bash
# Provera da li portovi rade
sudo netstat -tlnp | grep -E '80|443'

# Provera Docker mreže
docker network ls
docker network inspect web

# Provera DNS rezolucije
nslookup suninnova.com
dig suninnova.com

# Test HTTP zahteva
curl -v https://suninnova.com
```

---

## ⚠️ Važne Napomene

1. **Docker Socket Security**

   - Traefik ima pristup `/var/run/docker.sock`
   - Ovo omogućava auto-discovery kontejnera
   - **Rizik:** Root pristup Docker API-ju

2. **SSL Email**

   - Let's Encrypt šalje notifikacije na: ops@suninnova.com
   - Proveravaj email za upozorenja o isticanju sertifikata

3. **Restart Policy**

   - Svi kontejneri imaju `restart: unless-stopped`
   - Automatski se pokreću nakon reboot-a

4. **Port Exposure**
   - **80:** HTTP (Traefik automatski redirect-uje na 443)
   - **443:** HTTPS
   - **Svi ostali portovi:** Zatvoreni (samo interna Docker mreža)

---

## 📚 Dodatni Resursi

### Dokumentacija

- [Traefik Docs](https://doc.traefik.io/traefik/)
- [Docker Compose Docs](https://docs.docker.com/compose/)
- [Let's Encrypt Docs](https://letsencrypt.org/docs/)

### Hostinger Specifično

- Monarx Agent - Automatski security scanning (Hostinger feature)
- `/etc/motd` - Message of the Day sa Hostinger info

---

## 📞 Troubleshooting Kontakt

Ako imate problema:

1. Proverite Docker logove: `docker compose logs -f`
2. Proverite Traefik dashboard (ako je omogućen)
3. Proverite DNS propagaciju: `https://dnschecker.org`
4. Kontaktirajte Hostinger podršku za VPS probleme

---

**Kraj dokumentacije**  
_Ažurirano: 24. Novembar 2025_
