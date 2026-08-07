# Nginx Proxy Manager (NPM) on Debian 13 (Trixie) Docker

**Nginx Proxy Manager (NPM)**, web sitelerinizi, konteynerlerinizi ve servislerinizi güvenli bir şekilde dış dünyaya açmanızı sağlayan, kullanımı kolay ve güçlü bir grafik arayüzlü (GUI) **ters proxy (reverse proxy)** yönetim aracıdır.

Bu rehber, **Debian 13 (Trixie)** üzerinde Docker ve Docker Compose ile NPM'in baştan sona eksiksiz kurulumu, SSL sertifikası yapılandırması ve güvenlik pratiklerini kapsamaktadır.

---

## 📘 Adım Adım Docker Kurulumu

### Adım 1: Sistem Güncellemesi ve Hazırlık

Sistem paketlerinizi güncelleyin ve temel araçları yükleyin:

```bash
# 1.1 Sistem Güncellemesi
sudo apt update && sudo apt full-upgrade -y

# 1.2 Temel Araçlar
sudo apt install -y curl nano
```

---

### Adım 2: Docker ve Docker Compose Kontrolü

> 💡 **Not:** Sunucunuzda Docker ve Docker Compose zaten kurulu olduğu için bu adımda sadece sürüm doğrulaması yapılmaktadır.

```bash
docker --version
docker compose version
sudo systemctl status docker
```

---

### Adım 3: Çalışma Dizini ve Docker Compose Dosyası

NPM dosyalarını düzenli tutmak için `/opt` dizininde bir klasör oluşturun:

```bash
sudo mkdir -p /opt/nginx-proxy-manager
cd /opt/nginx-proxy-manager
```

#### Seçenek A: MariaDB Veritabanı ile (Üretim Ortamı İçin Önerilen)

`docker-compose.yml` dosyasını oluşturun:

```bash
nano docker-compose.yml
```

Aşağıdaki konfigürasyonu yapıştırın:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'     # HTTP Trafiği
      - '81:81'     # Yönetim Arayüzü (Admin Panel)
      - '443:443'   # HTTPS Trafiği
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm_guvenli_sifre_buraya"
      DB_MYSQL_NAME: "npm"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db
    networks:
      - npm-network

  db:
    image: 'mariadb:10.11'
    container_name: npm-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "root_guvenli_sifre_buraya"
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm_guvenli_sifre_buraya'
    volumes:
      - ./mysql:/var/lib/mysql
    networks:
      - npm-network

networks:
  npm-network:
    driver: bridge
```

> ⚠️ `DB_MYSQL_PASSWORD` ve `MYSQL_PASSWORD` değerlerini kendi güçlü şifrelerinizle değiştirin. `app` servisindeki `DB_MYSQL_PASSWORD` ile `db` servisindeki `MYSQL_PASSWORD` değerleri **birbiriyle eşleşmek zorundadır.** `MYSQL_ROOT_PASSWORD` ise veritabanı yönetimi için ayrı bir root şifresidir.

#### Seçenek B: SQLite ile (Hafif Sistemler İçin)

RAM kaynağınız kısıtlıysa (512MB - 1GB RAM'li küçük bir VPS), dahili SQLite veritabanını kullanabilirsiniz:

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
```

---

### Adım 4: Konteynerlerin Başlatılması

```bash
# 4.1 Konteynerleri arka planda başlatın
docker compose up -d

# 4.2 Durumu kontrol edin
docker compose ps

# 4.3 Canlı logları takip edin (isteğe bağlı)
docker compose logs -f
```

Her şey yolundaysa `nginx-proxy-manager` ve `npm-db` konteynerlerinin durumu **Up (Running)** olarak görünecektir.

---

### Adım 5: İlk Giriş ve Güvenlik Ayarları

1. Tarayıcınızdan `http://SUNUCU_IP_ADRESINIZ:81` adresine gidin.
2. **Varsayılan Giriş Bilgileri:**
   - **Email:** `admin@example.com`
   - **Password:** `changeme`
3. İlk giriş sonrası sizden **Ad / Soyad** ve **E-posta** bilgilerinizi güncellemeniz istenecektir.
4. Varsayılan `changeme` şifresini **güçlü yeni bir şifre** ile değiştirin.

---

### Adım 6: Yeni Bir Proxy Host (Domain) Ekleme

Örnek Senaryo: Sunucunuzda port `8080` üzerinde çalışan bir web uygulamanız var (ör. Portainer veya Nextcloud) ve bunu `panel.alanadiniz.com` üzerinden yayınlamak istiyorsunuz.

1. Sol menüden **Hosts** > **Proxy Hosts** sekmesine gelin.
2. **Add Proxy Host** butonuna tıklayın.
3. **Details** sekmesinde:
   - **Domain Names:** `panel.alanadiniz.com` (Yazıp Enter'a basın)
   - **Scheme:** `http` (veya `https`)
   - **Forward Hostname / IP:** Hedef servisinizin IP veya Konteyner adı
   - **Forward Port:** `8080`
   - **Block Common Exploits:** ✅ Açık yapın

> ⚠️ **KRİTİK NOT (Forward IP / Hostname Mantığı):**  
> NPM Docker konteyneri içinde çalıştığından `127.0.0.1` yazmak NPM'in kendi içine yönlenmesine sebep olur.
>
> - **Sunucudaki (Host) Servisler İçin:** Sunucunuzun yerel IP adresini (ör. `192.168.1.50` veya Docker köprü IP'si `172.17.0.1`) kullanın.
> - **Diğer Docker Konteynerleri İçin:** Ortak bir Docker ağı kurup doğrudan **Konteyner Adını** (ör. `portainer`, `wordpress`) yazın.

---

### Adım 7: Ücretsiz Let's Encrypt SSL Sertifikası Alma

Proxy Host ekleme penceresinde **SSL** sekmesine geçin:

1. **SSL Certificate** menüsünden **Request a new SSL Certificate** seçin.
2. **Force SSL:** ✅ Açık yapın (HTTP isteklerini otomatik HTTPS'e yönlendirir).
3. **HTTP/2 Support:** ✅ Açık yapın.
4. **Email Address for Let's Encrypt:** Geçerli e-posta adresinizi girin.
5. **I Agree to the Let's Encrypt Terms of Service:** ✅ Onaylayın.
6. **Save** butonuna basın.

NPM, arka planda Let's Encrypt ile haberleşerek SSL sertifikasını saniyeler içinde alacak ve otomatik yenilemeyi ayarlayacaktır.

---

### Adım 8: Erişim Kısıtlama (Access Lists)

Hassas yönetim panellerinize parola koruması koymak için:

1. Menüden **Access Lists** sekmesine gelin.
2. **Add Access List** butonuna tıklayın.
3. **Details** kısmında bir isim verin (Örn: *Admin Koruması*).
4. **Authorization** sekmesinde Kullanıcı Adı ve Parola belirleyin.
5. **Access** sekmesinde dilerseniz sadece belirli IP adreslerine izin verin (`satisfy any` veya `satisfy all`).
6. Oluşturduğunuz listeyi herhangi bir **Proxy Host** ayarlarındaki **Access List** seçeneğinden bağlayabilirsiniz.

---

### Adım 9: Güvenlik Sertleştirmeleri

#### 9.1 Güvenlik Duvarı Yapılandırması

Kullandığınız güvenlik duvarı çözümünde (UFW, iptables, FortiGate, pfSense vb.) aşağıdaki portlara izin verin:

| Port | Protokol | Açıklama |
|---|---|---|
| **22** | TCP | SSH Uzak Erişim |
| **80** | TCP | HTTP Trafiği & SSL Doğrulama |
| **443** | TCP | HTTPS Trafiği |

> ⚠️ **Port 81 (Yönetim Paneli):**  
> Port 81'i dış dünyaya açmayın. Yönetim paneline erişmek için:
>
> 1. `docker-compose.yml` içinde port ayarını `- '127.0.0.1:81:81'` yapıp SSH Tüneli ile bağlanın.
> 2. Veya NPM üzerinden kendi port 81'ine yönlenen bir Proxy Host oluşturup, **Access Lists** ile IP kısıtlaması uygulayın.

#### 9.2 Docker Ortak Ağ (Custom Docker Network) Kullanımı

Diğer Docker konteynerlerinizin portlarını dışarı açmadan (`ports:` yazmadan) güvenli haberleşme için ortak bir Docker ağı oluşturun:

```bash
docker network create web-network
```

NPM `docker-compose.yml` dosyanıza bu ağı ekleyin ve `app` servisine bağlayın:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    # ... mevcut ayarlar ...
    networks:
      - npm-network
      - web-network    # Bu satırı ekleyin

networks:
  npm-network:
    driver: bridge
  web-network:
    external: true
```

Artık diğer servislerinizi `web-network` ağına bağlayarak NPM içinden doğrudan Konteyner Adı (ör. `Forward Hostname: wordpress_app`, `Forward Port: 80`) ile yönlendirebilirsiniz.

---

### Adım 10: Yedekleme ve Güncelleme

#### 10.1 Sürüm Güncelleme

```bash
cd /opt/nginx-proxy-manager
docker compose pull
docker compose up -d
docker image prune -f
```

#### 10.2 Manuel Yedekleme

```bash
# Konteynerleri durdurun (Veri tutarlılığı için)
cd /opt/nginx-proxy-manager
docker compose down

# Klasörü arşivleyin (data/, letsencrypt/ ve mysql/ dahil tüm veriler)
tar -czvf npm_yedek_$(date +%F).tar.gz /opt/nginx-proxy-manager

# Konteynerleri tekrar başlatın
docker compose up -d
```

---

## 💡 Dikkat Edilmesi Gereken İpuçları & Düzeltmeler

1. **`127.0.0.1` Tuzağı:** NPM Docker içinde çalıştığından Forward IP alanına `127.0.0.1` yazmak NPM'in kendi konteynerine yönlenmesine sebep olur. Sunucu IP'sini veya konteyner adını kullanın.
2. **SSL `Internal Error` Hatası:** Port 80 ve 443'ün dış dünyadan erişilebilir olduğundan ve DNS A kaydının doğru IP'ye yönlendiğinden emin olun.
3. **`Address already in use` Hatası:** `sudo ss -tlpn | grep -E ':80|:443'` ile portu hangi servisin işgal ettiğini tespit edip durdurun.
4. **WebSocket Kopmaları (502/504):** Proxy Host ayarlarında **Websockets Support** seçeneğinin açık olduğunu doğrulayın.

*Tebrikler! Debian 13 üzerinde Nginx Proxy Manager kurulumunuz başarıyla tamamlanmıştır.*

---
Author: faruk-guler
GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
Page: [www.farukguler.com](https://www.farukguler.com)
