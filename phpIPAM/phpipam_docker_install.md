# phpIPAM Docker & Docker Compose Container (Debian 13)

**phpIPAM**, web tabanlı, açık kaynaklı ve modüler bir IP Adres Yönetim (IPAM - IP Address Management) yazılımıdır.

Bu rehber, **Debian 13 (Trixie)** üzerinde **Docker**, **Docker Compose**, **MariaDB 11** ve **Nginx Proxy Manager** kullanarak phpIPAM'in konteyner mimarisi ile kurulmasını adım adım anlatmaktadır.

---

## Mimari Yapı

```
             [ Internet / Kullanıcılar ]
                         │
                         ▼
        [ Nginx Proxy Manager / Reverse Proxy ]
               (HTTPS - Let's Encrypt)
                         │
        ┌────────────────┴────────────────┐
        ▼                                 ▼
[ phpipam-web Container ]       [ phpipam-cron Container ]
  (Web Arayüzü & API)             (Ping & Discovery Taramaları)
        │                                 │
        └────────────────┬────────────────┘
                         ▼
             [ MariaDB 11 Container ]
                         │
                         ▼
            [ Kalıcı Veri (Volumes) ]
```

---

## Adım 1: Debian 13 Üzerine Docker ve Docker Compose Kurulumu

Sistem paketlerini güncelleyin ve resmi Docker deposunu ekleyin:

```bash
# Sistem güncellemeleri ve gerekli araçlar
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release

# Docker resmi GPG anahtarını ekleyin
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker apt deposunu tanımlayın
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker Engine ve Docker Compose Eklentisini kurun
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Docker servisini otomatik başlatmaya ayarlayın ve başlatın
sudo systemctl enable --now docker
```

---

## Adım 2: Özel config.php Dosyasını Hazırlama

phpIPAM'in resmi Docker imajları yapılandırmayı environment değişkenlerinden çekse de, PHP 8.4 uyarılarını atlamak (`$allow_untested_php_versions=true;`) gibi özel durumlar için özel bir `config.php` kullanmak en güvenli (resmi) yöntemdir.

phpIPAM için çalışma dizini oluşturun ve özel config dosyasını hazırlayın:

```bash
sudo mkdir -p /opt/phpipam && cd /opt/phpipam
sudo nano config.php
```

Aşağıdaki içeriği yapıştırın:

```php
<?php
$db['host'] = getenv("IPAM_DATABASE_HOST") ?: "phpipam-db";
$db['user'] = getenv("IPAM_DATABASE_USER") ?: "phpipam";
$db['pass'] = getenv("IPAM_DATABASE_PASS") ?: "GuvenliDbSifreniz123!";
$db['name'] = getenv("IPAM_DATABASE_NAME") ?: "phpipam";
$db['port'] = 3306;
$db['webhost'] = "%";

// Debian 13 PHP 8.4 uyumluluk bypass'ı (phpIPAM henüz PHP 8.4'ü resmi desteklemiyor)
$allow_untested_php_versions=true;

// Ek Güvenlik Katmanı: Kurulum modülünü devre dışı bırak
$disable_installer = true;
?>
```

---

## Adım 3: Docker Compose Yapılandırması (`docker-compose.yml`)

Aynı klasörde (`/opt/phpipam`) Compose dosyasını oluşturun:

```bash
sudo nano docker-compose.yml
```

Aşağıdaki `docker-compose.yml` içeriğini yapıştırın:

```yaml
services:
  # 1. MariaDB Veritabanı Servisi
  phpipam-db:
    image: mariadb:11
    container_name: phpipam-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: GuvenliRootSifreniz123!
      MYSQL_DATABASE: phpipam
      MYSQL_USER: phpipam
      MYSQL_PASSWORD: GuvenliDbSifreniz123!
    volumes:
      - phpipam-db-data:/var/lib/mysql

  # 2. phpIPAM Web Arayüzü (www)
  phpipam-web:
    image: phpipam/phpipam-www:latest
    container_name: phpipam-web
    restart: unless-stopped
    # ports:
    #   - "8080:80" # Güvenlik için dışarı kapalı, Nginx/Caddy proxy üzerinden erişilecek
    environment:
      IPAM_DATABASE_HOST: phpipam-db
      IPAM_DATABASE_USER: phpipam
      IPAM_DATABASE_PASS: GuvenliDbSifreniz123!
      IPAM_DATABASE_NAME: phpipam
      IPAM_DATABASE_WEBHOST: "%"
      TZ: Europe/Istanbul
    volumes:
      - ./config.php:/phpipam/config.php
      - phpipam-logo:/phpipam/css/images/logo
      - phpipam-ca:/usr/local/share/ca-certificates:ro
    cap_add:
      - NET_ADMIN
      - NET_RAW
    networks:
      - default
      - proxy-net
    depends_on:
      - phpipam-db

  # 3. phpIPAM Otomatik Taramalar (Cron & Discovery Agent)
  phpipam-cron:
    image: phpipam/phpipam-cron:latest
    container_name: phpipam-cron
    restart: unless-stopped
    environment:
      IPAM_DATABASE_HOST: phpipam-db
      IPAM_DATABASE_USER: phpipam
      IPAM_DATABASE_PASS: GuvenliDbSifreniz123!
      IPAM_DATABASE_NAME: phpipam
      SCAN_INTERVAL: 1h
      TZ: Europe/Istanbul
    volumes:
      - ./config.php:/phpipam/config.php
      - phpipam-ca:/usr/local/share/ca-certificates:ro
    cap_add:
      - NET_ADMIN
      - NET_RAW
    networks:
      - default
      - proxy-net
    depends_on:
      - phpipam-db

volumes:
  phpipam-db-data:
  phpipam-logo:
  phpipam-ca:

networks:
  proxy-net:
    external: true
```

---

## Adım 4: Konteynerleri Başlatma ve İlk Oturum

1. Ortak proxy ağını oluşturun ve konteynerleri başlatın:

```bash
sudo docker network create proxy-net
cd /opt/phpipam
sudo docker compose up -d
```

1. Konteyner durumlarını doğrulayın:

```bash
sudo docker compose ps
```

1. Web Arayüzüne Bağlanın:
   - Tarayıcınızdan Nginx Proxy Manager veya Caddy üzerinde ayarladığınız domain adresi (Örn: `http://ipam.domaininiz.com/`) ile erişin.
   - Veritabanı otomatik olarak yüklenecektir.
   - **Varsayılan Kullanıcı Adı:** `admin`
   - **Varsayılan Şifre:** `ipamadmin`
   - Giriş yaptıktan sonra şifrenizi değiştirin.
2. **Panel İçi Yol Ayarları (Kritik):** Taramaların çalışabilmesi için **Administration > phpIPAM settings** menüsünden **Ping path** (`/usr/bin/ping`) ve **FPing path** (`/usr/bin/fping`) alanlarının dolu ve doğru olduğundan emin olun.

---

## Adım 5: Reverse Proxy & Otomatik SSL (Nginx Proxy Manager)

HTTPS (SSL) yayınlama için aynı sunucuda Nginx Proxy Manager çalıştırabilirsiniz.

```bash
sudo mkdir -p /opt/nginx-proxy-manager && cd /opt/nginx-proxy-manager
sudo nano docker-compose.yml
```

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

```bash
sudo docker compose up -d
```

- Tarayıcıdan `http://SUNUCU_IP:81` portuna girin (Varsayılan: `admin@example.com` / `changeme`).
- **Proxy Hosts -> Add Proxy Host** adımlarını izleyerek `ipam.domaininiz.com` adresini **`http://phpipam-web:80`** hedefine yönlendirin. (Aynı proxy-net ağında oldukları için doğrudan konteyner adını kullanabilirsiniz).
- **SSL -> Request a new SSL Certificate (Let's Encrypt)** seçeneğini işaretleyip kaydedin.

---

## Container Yönetimi, Güncelleme ve Yedekleme

- **Logları İzlemek İçin:**  

  ```bash
  sudo docker compose logs -f phpipam-web
  ```

- **Sistemi Güncellemek İçin:**  

  ```bash
  cd /opt/phpipam
  sudo docker compose pull
  sudo docker compose up -d
  ```

- **Veritabanı Yedeği (Dump) Almak İçin:**  

  ```bash
  sudo docker exec phpipam-db mariadb-dump -u root -pGuvenliRootSifreniz123! phpipam > /var/backups/phpipam_docker_$(date +%F).sql
  ```

- **Otomatik Günlük Veritabanı Yedeği (Cron):**  
  Host sunucunun `sudo crontab -e` dosyasına ekleyebilirsiniz:

  ```cron
  0 2 * * * docker exec phpipam-db mariadb-dump -u root -pGuvenliRootSifreniz123! phpipam > /var/backups/phpipam_docker_$(date +\%Y\%m\%d).sql
  0 3 * * * find /var/backups/ -name "phpipam_docker_*.sql" -mtime +10 -exec rm {} \;
  ```

- **Yedekten Geri Yüklemek (Restore) İçin:**  

  ```bash
  sudo docker exec -i phpipam-db mariadb -u root -pGuvenliRootSifreniz123! phpipam < /var/backups/phpipam_docker_2026-08-04.sql
  ```

---

## 🛠️ Docker Ortamında Sık Karşılaşılan Sorunlar ve Çözümleri

| Sorun | Neden Olur? | Çözüm |
| :--- | :--- | :--- |
| **502 Bad Gateway** | Nginx Proxy Manager konteynere erişemiyor veya phpipam-web henüz başlamadı. | `sudo docker compose ps` ile kontrol edin. `sudo docker compose logs phpipam-web` ile logları inceleyin. |
| **Database connection error** | Veritabanı konteyneri henüz hazır değil veya şifreler eşleşmiyor. | `docker-compose.yml` içindeki `IPAM_DATABASE_PASS` ve `MYSQL_PASSWORD` alanlarının aynı olduğunu doğrulayın. |
| **Port 80/443 already in use** | Sunucuda önceden kurulu olan Apache veya varsayılan Nginx çalışıyor. | `sudo systemctl stop apache2` veya `sudo systemctl stop nginx` çalıştırarak portları boşaltın. |
| **Beyaz Ekran / 500 Internal Error** | PHP tarafında izin eksikliği veya veritabanı şeması eksik yüklenmiş. | `sudo docker compose restart phpipam-web` yapın. |
| **PHP Version not officially supported** | Kullanılan PHP sürümü phpipam'in destek listesinden daha yeni (örn: PHP 8.4). | Özel `config.php` dosyasının oluşturulduğundan ve volume olarak mount edildiğinden emin olun. |

---

## 💡 Alternatif: Caddy Server ile Otomatik HTTPS (Proxy Manager Yerine)

Nginx Proxy Manager yerine Caddy Server kullanmak isterseniz `docker-compose.yml` dosyanıza Caddy ekleyerek sıfır konfigürasyonla Let's Encrypt SSL sertifikasına sahip olabilirsiniz:

```yaml
  caddy:
    image: caddy:latest
    container_name: phpipam-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    networks:
      - proxy-net
    command: caddy reverse-proxy --from ipam.domaininiz.com --to phpipam-web:80
```

---
Author: faruk-guler
GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
Page: [www.farukguler.com](https://www.farukguler.com)
