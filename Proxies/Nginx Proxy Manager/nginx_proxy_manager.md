# Nginx Proxy Manager (NPM) on Debian 13 (Trixie) Docker

**Nginx Proxy Manager (NPM)**, web sitelerinizi, konteynerlerinizi ve servislerinizi güvenli bir şekilde dış dünyaya açmanızı sağlayan, kullanımı kolay ve güçlü bir grafik arayüzlü (GUI) **ters proxy (reverse proxy)** yönetim aracıdır.

Bu rehber, **Debian 13 (Trixie)** üzerinde Docker ve Docker Compose ile NPM kurulumunu; SSL sertifika yapılandırmasını, erişim kontrollerini, güvenlik sertleştirmelerini ve yedekleme/geri yükleme işlemlerini adım adım kapsamaktadır. Docker motorunun kurulu olduğu varsayılır (bkz. Adım 2). NPM'in grafik arayüzü yerine `/etc/nginx` dosyalarıyla elle reverse proxy yapılandırması ayrı bir konudur.

---

## 📘 Adım Adım Kurulum ve Yapılandırma

### Adım 1: Sistem Güncellemesi ve Hazırlık

Sistem paketlerinizi güncelleyin ve temel araçları yükleyin:

```bash
# 1.1 Sistem Güncellemesi
sudo apt update && sudo apt upgrade -y

# 1.2 Temel Araçlar
sudo apt install -y curl nano

# 1.3 Docker servisi etkin mi?
sudo systemctl enable --now docker
```

> 💡 `full-upgrade` yerine `upgrade` tercih edilir: `full-upgrade`, bağımlılık değişikliklerinde paket **kaldırabilir** ve Docker host'unda bu `docker-ce`/`containerd`/`nfs-common` gibi kritik paketlere dokunabilir. `full-upgrade` çalıştırmanız gerekiyorsa çıkan listeyi `-s` (simulate) ile önce inceleyin.

---

### Adım 2: Docker ve Docker Compose Kontrolü

> 💡 **Not:** Sunucunuzda Docker ve Docker Compose zaten kurulu olduğu için bu adımda sadece sürüm doğrulaması yapılmaktadır.
>
> Bu rehberdeki `docker compose` (boşluklu) komutları **Compose v2** içindir; eski `docker-compose` (tireli) ikiliyle karıştırmayın. Adım 6'da kullanılan `host-gateway` özelliği **Docker Engine 20.10+** gerektirir.

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
    image: 'jc21/nginx-proxy-manager:2.16.0'  # latest değil; sabit sürüm (bkz. Adım 10.1)
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'     # HTTP Trafiği
      - '127.0.0.1:81:81'  # Yönetim Arayüzü (Sadece localhost — dışarıya açmayın!)
      - '443:443'   # HTTPS Trafiği
    environment:
      TZ: "Europe/Istanbul"          # Log ve sertifika yenileme zamanları için
      # DISABLE_IPV6: 'true'         # Hostta IPv6 yoksa bu satırın yorumunu kaldırın
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm_guvenli_sifre_buraya"
      DB_MYSQL_NAME: "npm"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'
    depends_on:
      db:
        condition: service_healthy
    extra_hosts:
      - "host.docker.internal:host-gateway"   # Hosttaki servislere yönlendirme (bkz. Adım 6)
    networks:
      - npm-network

  db:
    image: 'mariadb:10.11'   # yeni kurulumda 11.4 (LTS) de tercih edilebilir — bkz. aşağıdaki not
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
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      start_period: 30s
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  npm-network:
    driver: bridge
```

> 💡 **MariaDB sürüm seçimi:** `mariadb:10.11` hâlâ desteklenen bir LTS serisidir ve sorunsuz çalışır. Daha yeni bir kurulum için `mariadb:11.4` (LTS) tercih edebilirsiniz; Debian 13'ün kendi deposu da 11.8 serisini içerir. **Önemli fark:** MariaDB Docker Official Image'ında `mysqldump` symlink'i **11.0'dan itibaren kaldırılmıştır**, yalnızca `mariadb-dump` bulunur. Adım 10.2'deki yedekleme komutunu sürümünüze göre seçin.
>
> 💡 **`DISABLE_IPV6`:** Host'unuzda IPv6 yoksa NPM içindeki nginx `[::]:80` / `[::]:443` bind'lerinde hata verebilir; bu durumda ilgili satırın yorumunu kaldırın. `TZ` ise yenileme loglerini doğru saat diliminde görmenizi sağlar.

> ⚠️ `DB_MYSQL_PASSWORD` ve `MYSQL_PASSWORD` değerlerini kendi güçlü şifrelerinizle değiştirin. `app` servisindeki `DB_MYSQL_PASSWORD` ile `db` servisindeki `MYSQL_PASSWORD` değerleri **birbiriyle eşleşmek zorundadır.** `MYSQL_ROOT_PASSWORD` ise veritabanı yönetimi için ayrı bir root şifresidir.

> 🔒 **Güvenlik — `.env` Dosyası Kullanımı:** Şifrelerinizi `docker-compose.yml` içine düz metin olarak yazmak yerine, aynı dizinde bir `.env` dosyası oluşturun ve `docker-compose.yml`'de `${DEGISKEN_ADI}` sözdizimini kullanın:
>
> ```bash
> # /opt/nginx-proxy-manager/.env
> MYSQL_ROOT_PASSWORD=guclu_root_sifreniz
> MYSQL_PASSWORD=guclu_npm_sifreniz
> ```
>
> Ardından `docker-compose.yml` içinde:
>
> ```yaml
> MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
> MYSQL_PASSWORD: "${MYSQL_PASSWORD}"
> DB_MYSQL_PASSWORD: "${MYSQL_PASSWORD}"
> ```
>
> `.env` dosyasını mutlaka `.gitignore`'a ekleyin; aksi halde şifreler versiyon kontrolüne sızar.

#### Seçenek B: SQLite ile (Hafif Sistemler İçin)

RAM kaynağınız kısıtlıysa (512MB - 1GB RAM'li küçük bir VPS), dahili SQLite veritabanını kullanabilirsiniz.

> ⚠️ **SQLite Kısıtlamaları:**
>
> - **Yüksek eşzamanlı** erişimde write-lock sorunları yaşanabilir; yoğun trafikli ortamlar için MariaDB tercih edin.
> - **Veritabanı seçimini kurulumun başında yapın.** SQLite → MariaDB geçişi resmî bir araçla sunulmaz; topluluk yöntemi veritabanını döküp (`mysqldump`) `mysql2sqlite` benzeri bir dönüştürücüyle SQLite dosyasına aktarmaktır — yani mümkün ama zahmetli ve risklidir.
> - **Kalıcı dosya yolu:** `./data/database.sqlite` (konteyner içinde `/data/database.sqlite`; `DB_SQLITE_FILE` ile değiştirilebilir).
> - **Yedekleme:** Yalnızca `database.sqlite` dosyası **yetmez**. İlk açılışta üretilen **JWT imzalama anahtarları da `./data` klasöründe** tutulur; panelin tamamını yedekleyin (Adım 10.2).
> - **Ortada kalan `DB_MYSQL_*` değişkenleri:** Resmî dokümanın uyarısına göre `DB_MYSQL_*`, `DB_SQLITE_*` seçeneklerini **ezer**. SQLite'a dönmek için bu değişkenleri `docker-compose.yml`/`.env` içinden tamamen kaldırın; aksi halde NPM sessizce MariaDB modunda kalır ve arayüz boş/hatalı görünebilir.

> 💡 MariaDB/MySQL dışında **PostgreSQL** de desteklenir (`DB_POSTGRES_HOST`, `DB_POSTGRES_PORT`, `DB_POSTGRES_USER`, `DB_POSTGRES_PASSWORD`, `DB_POSTGRES_NAME`; özel schema desteklenmez, `public` kullanılır). Bu rehber örneklerinde MariaDB tercih edilmiştir.

SQLite yapılandırması:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:2.16.0'  # latest değil; sabit sürüm (bkz. Adım 10.1)
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '127.0.0.1:81:81'  # Yönetim Arayüzü (Sadece localhost)
      - '443:443'
    environment:
      TZ: "Europe/Istanbul"
      # DISABLE_IPV6: 'true'                  # Hostta IPv6 yoksa kaldırın
      # DB_SQLITE_FILE: "/data/database.sqlite"  # varsayılan yolu değiştirmek için
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    extra_hosts:
      - "host.docker.internal:host-gateway"   # Hosttaki servislere yönlendirme (bkz. Adım 6)
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'
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

> 💡 **İlk açılış uzun sürebilir:** Konteyner ilk çalıştığında JWT anahtarlarını üretir, veritabanını şemayla doldurur ve varsayılan yönetici hesabını oluşturur. Bu işlem donanıma bağlı olarak **birkaç dakika** sürebilir; bu sırada panelin henüz cevap vermemesi normaldir.

> ⚠️ **`npm-db is unhealthy` / app restart döngüsü:** Genellikle MariaDB'nin ilk initialization'ı healthcheck'in `start_period` süresinden uzun sürdüğünde ortaya çıkar (bu yüzden yukarıda 30s verilmiştir). Sırayla bakın:
>
> ```bash
> docker compose ps              # db servisi unhealthy mi?
> docker compose logs --tail=50 db
> docker compose logs --tail=50 app
> ```

---

### Adım 5: İlk Giriş ve Güvenlik Ayarları

Port 81 sadece localhost'a bağlı olduğundan, yönetim paneline erişmek için:

- **Sunucu üzerindeyseniz:** Tarayıcıdan `http://localhost:81` adresine gidin.
- **Uzaktan bağlanıyorsanız:** Önce SSH tüneli açın, ardından yerel tarayıcınızdan `http://localhost:8181` ile bağlanın:

  ```bash
  ssh -L 8181:127.0.0.1:81 kullanici@SUNUCU_IP_ADRESI
  ```

1. **Varsayılan Giriş Bilgileri:**
   - **Email:** `admin@example.com`
   - **Password:** `changeme`
2. İlk giriş sonrası sizden **Ad / Soyad** ve **E-posta** bilgilerinizi güncellemeniz istenecektir.
3. Varsayılan `changeme` şifresini **güçlü yeni bir şifre** ile değiştirin. Bu adımı atlamayın: bilgiler internette herkes tarafından bilinen sabit değerlerdir.
4. **MFA (2FA) açın.** Kullanıcı profili ekranındaki *Two Factor Authentication* bölümünden kimlik doğrulayıcı uygulamanızla (Aegis, Authy, Google Authenticator) kurulum kodunu tarayın. Panele ağ üzerinden erişiliyorsa en etkili ikinci önlem budur.
5. Yönlendirme yapacak ikinci bir kullanıcı/rol tanımlayın (Settings > Users) ve yönetim hesabını yalnızca bakım için kullanın; kimin neyi değiştirdiğini **Audit Log**'dan izleyin.

> ⚠️ Panelde yapılan her değişiklik nginx yapılandırmasını yeniden üretir ve yükletir. Yanlış kaydedilmiş bir hedef adres, tüm HTTPS sitelerinizi etkisiz bırakabilir — bu yüzden Adım 10.2'deki yedek almadan değişiklik yapmayın.

---

### Adım 6: Yeni Bir Proxy Host (Domain) Ekleme

Örnek Senaryo: Sunucunuzda port `8080` üzerinde çalışan bir web uygulamanız var (ör. Portainer veya Nextcloud) ve bunu `panel.alanadiniz.com` üzerinden yayınlamak istiyorsunuz.

#### 6.0 Ön Koşul: Ortak Docker Ağı

Başka konteynerleri **konteyner adıyla** yönlendirecekseniz, o konteynerlerin NPM ile **aynı ağda** olması gerekir; aksi halde `Forward Hostname: portainer` yazsanız bile nginx hedefi çözümleyemez ve **502** alırsınız. Ağ kurulumu bu rehberde Adım 9.2'de ayrıntılı anlatılıyor; konteyner adlarıyla çalışacaksanız **önce** o adımı uygulayın:

```bash
docker network create web-network
```

1. Sol menüden **Hosts** > **Proxy Hosts** sekmesine gelin.
2. **Add Proxy Host** butonuna tıklayın.
3. **Details** sekmesinde:
   - **Domain Names:** `panel.alanadiniz.com` (Yazıp Enter'a basın)
   - **Scheme:** `http` (veya `https`)
   - **Forward Hostname / IP:** Hedef servisinizin IP veya Konteyner adı
   - **Forward Port:** `8080`
   - **Block Common Exploits:** ✅ Açık yapın
   - **Websockets Support:** ✅ Açık yapın (canlı terminal, grafik, bildirim gibi upgrade isteyen servisler için)

> 💡 Uzun süre sessiz kalan WebSocket bağlantıları nginx'in varsayılan timeout'unda (60 sn) kopar. Proxy Host'un **Advanced** sekmesine `proxy_read_timeout 300s;` ekleyerek bunu çözebilirsiniz.

> ⚠️ **KRİTİK NOT (Forward IP / Hostname Mantığı):**  
> NPM Docker konteyneri içinde çalıştığından `127.0.0.1` yazmak NPM'in kendi içine yönlenmesine sebep olur.
>
> - **Sunucudaki (Host) Servisler İçin:** Adım 3'teki yığına eklenen `extra_hosts: "host.docker.internal:host-gateway"` satırı sayesinde hedef alanına `host.docker.internal` yazmanız yeterlidir (Docker Engine 20.10+). Bu ad her zaman host'un gateway adresine çözülür ve köprü IP değişimlerinden etkilenmez.
> - **Neden `172.17.0.1` önerilmiyor:** `172.17.0.1` varsayılan `docker0` köprüsünün adresidir; bu yığın ise kendi `npm-network` ağını kullanır ve o ağın gateway'i farklıdır. Sabit IP yazmak, ağ/köprü değiştiğinde sessizce bozulur. Mecbur kalırsanız `docker network inspect npm-network | grep Gateway` ile doğru gateway'i görün.
> - **Yalnızca loopback'i dinleyen servisler:** Hostta `127.0.0.1:9000` gibi sadece loopback'e bağlanmış bir uygulama konteynerden **erişilemez**. Dinleme adresini `0.0.0.0`/LAN adresine çevirin veya o servisi `web-network` ağına alın.
> - **Diğer Docker Konteynerleri İçin:** Ortak bir Docker ağı kurup (Adım 6.0) doğrudan **Konteyner Adını** (ör. `portainer`, `wordpress`) yazın.

---

### Adım 7: Ücretsiz Let's Encrypt SSL Sertifikası Alma

Proxy Host ekleme penceresinde **SSL** sekmesine geçin:

> 💡 **Kendi sertifikan mı var?** Bu adımı atlayıp doğrudan **Adım 7b**'ye geç; orada Custom sertifika yükleme, zincir hazırlama ve re-encrypt ayarı anlatılıyor.

1. **SSL Certificate** menüsünden **Request a new SSL Certificate** seçin.
2. **Force SSL:** ✅ Açık yapın (HTTP isteklerini otomatik HTTPS'e yönlendirir).
3. **HTTP/2 Support:** ✅ Açık yapın.
4. **Email Address for Let's Encrypt:** Geçerli e-posta adresinizi girin.
5. **I Agree to the Let's Encrypt Terms of Service:** ✅ Onaylayın.
6. **Save** butonuna basın.

NPM, arka planda certbot ile Let's Encrypt'ten sertifika alır ve yenilemeyi kendisi yönetir. Doğru DNS kaydı ve açık 80/443 portlarıyla bu işlem tipik olarak 10-30 saniye sürer.

#### 7.1 Wildcard veya Port 80 Açık Değilse: DNS Challenge

SSL sekmesinde **Use a DNS Challenge** seçeneğiyle, DNS sağlayıcınızın kimlik dosyasını arayüze vererek wildcard (`*.alanadiniz.com`) sertifika alabilirsiniz. Bu yöntem 80/443'ün dışarıya kapalı olduğu ortamlarda (CGNAT, ISP engeli) de çalışır.

> ⚠️ Aynı NPM örneğinde **birden fazla DNS sağlayıcı** kullanmak, certbot eklentileri arasındaki Python bağımlılık çakışmaları nedeniyle önerilmez. Eklenti çalışmazsa önce konteyner loglarındaki certbot/pip hatalarına bakın.

#### 7.2 Let's Encrypt Limitleri ve Limitsiz Test Yöntemleri

Registered domain başına uygulanan resmî limitler (o domain için **dünya genelindeki** tüm talepler sayılır):

| Limit | Değer |
|---|---|
| Yeni sertifika / registered domain | 7 günde 50 |
| Aynı isim seti için duplicate sertifika | 7 günde 5 |
| Başarısız doğrulama (failed order) | saatte 100 |

> ⚠️ **NPM arayüzünde "Staging" anahtarı yoktur** (özel ACME CA / staging desteği projede hâlâ açık bir enhancement talebidir). Yanlış config ile production limitini tüketmemek için uygulanan iki güvenli yol:
>
> 1. **Gözden çıkarılabilir bir alt alan adı** ile test edin (ör. `test.alanadiniz.com`); limit tükense bile asıl servisleriniz etkilenmez.
> 2. **Certbot CLI ile staging testi** (NPM 80/443'ü bırakırken) — DNS ve port doğrulamanızın çalıştığını limitsiz ortamda görürsünüz:
>
>    ```bash
>    sudo apt install -y certbot
>    docker compose stop app
>    sudo certbot certonly --standalone --test-cert -d test.alanadiniz.com
>    docker compose up -d app
>    ```
>
>    Bu yöntemle alınan staging sertifikası NPM'in veritabanına **girmez**; amaç yalnızca doğrulamanın geçtiğini teyit etmektir.
>
> İleri düzey ve **resmî olmayan** bir yol daha var: volume-mounted `./letsencrypt/cli.ini` dosyasına `server = https://acme-staging-v02.api.letsencrypt.org/directory` satırını eklemek. Staging sertifikaları tarayıcıda güvenilmez görünür; bu yöntemi kendi NPM sürümünüzde **doğrulamadan üretimde kullanmayın**.

> 💡 **Yenilemeyi izleme:** `docker compose logs -f | grep -iE 'renew|certificate'` ile denemeleri takip edin; son kullanma tarihlerini **SSL/TLS Certificates** ekranından kontrol edin. Yenileme başarısız olduğunda NPM mevcut sertifikayı kullanmaya devam eder — yani hemen çökmez, sona yakın fark edersiniz; bu yüzden takibi alışkanlık haline getirin.

---

### Adım 7b: Kendi Sertifikanı Yükleme (Custom SSL + Termination)

Let's Encrypt kullanmak istemiyor veya **kurum içi CA / KamuSM / satın alınmış sertifikan** varsa, SSL sonlandırma yine NPM üzerinde yapılır; tek fark sertifikayı sen getirirsin. NPM'in yaptığı iş aynıdır: 443'te TLS'i sonlandırıp backend'e yönlendirmek.

#### 7b.1 Hangi durumda hangi sertifika

| Erişen kim | Sertifika kaynağı | Sonuç |
|---|---|---|
| İnternetteki herkes (vatandaş/müşteri portalı) | **Public CA** (Let's Encrypt veya ücretli) | Uyarısız açılır |
| Yalnız kurum içi yönetilen cihazlar | **Kendi iç CA'n** (kök sertifikası GPO/MDM ile cihazlara dağıtılmış) | Uyarısız açılır; dağıtım yoksa her cihazda uyarı |
| Karma | Dış uçlar public cert, iç servisler iç cert + VPN/ZTNA | En doğru kurumsal model |

> ⚠️ İç CA ile üretilmiş sertifika **dış kullanıcının tarayıcısında "güvenli değil" uyarısı verir** — bu yapılandırma hatası değil, güven zinciri gerçeğidir. Kurum dışına açacağın servislerde public CA kullan.

#### 7b.2 Dosyaları hazırla

```bash
# 1) Anahtar şifresiz olmalı — passphrase soruyorsa şifreli demektir
openssl rsa -in server_encrypted.key -check -noout
openssl rsa -in server_encrypted.key -out server.key        # şifresiz hâli

# 2) Zinciri sırayla birleştir: sunucu sertifikası ÖNCE, ara CA sonra (KÖK CA'yı ekleme)
cat server.crt intermediate.crt > fullchain.pem

# 3) Anahtar-sertifika eşleşmesi (iki md5 değeri AYNI çıkmalı)
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa -noout -modulus -in server.key  | openssl md5

# 4) Sertifika içeriği: CN, SAN listesi, veriliş/bitiş, veren otorite
openssl x509 -noout -subject -issuer -dates -ext subjectAltName -in server.crt
```

`.pfx/.p12` (PKCS#12) dosyan varsa NPM **PEM** beklediği için önce çevir:

```bash
openssl pkcs12 -in Sertifika.pfx -out server.crt  -nokeys -clcerts -passin pass:SIFRE
openssl pkcs12 -in Sertifika.pfx -out server.key  -nocerts -nodes  -passin pass:SIFRE
openssl pkcs12 -in Sertifika.pfx -out chain.crt   -nokeys -cacerts -passin pass:SIFRE
```

> 💡 `-nodes` bayrağı anahtarı şifresiz yazar; `server.key` dosyasının izinlerini hemen `chmod 600` ile kısıtla.

#### 7b.3 Panele yükle ve proxy host'a bağla

1. Sol menüden **SSL/TLS Certificates** (bazı sürümlerde *Certificates*) sekmesine gel.
2. **Add SSL Certificate** butonuna tıkla ve tür olarak **Custom** seç.
3. Alanları doldur (etiketler sürüme göre değişebilir; üç alan görürsün):
   - **Key:** `server.key` içeriği (`-----BEGIN PRIVATE KEY-----` satırı dahil)
   - **Certificate:** `fullchain.pem` içeriği (sunucu sertifikası + ara CA)
   - **Chain / Intermediate CA:** ara zinciri ayrı dosya olarak da verebilirsin; `Certificate` alanına birleştirilmiş zinciri yapıştırdıysan bu alanı boş bırak
4. **Save** → geri dönüp proxy host'un **SSL** sekmesinde **SSL Certificate = Use a existing SSL Certificate** seç ve bu kaydı göster; **Force SSL** ve **HTTP/2 Support** açık.

> ⚠️ **Wildcard kapsamı tek seviyelidir:** `*.alanadiniz.com` → `panel.` ve `mail.` kapsar; `a.b.alanadiniz.com` **kapsamaz** (onun için `*.b.alanadiniz.com` gerekir). 55 alt alan adını tek sertifikayla açacaksan hepsi tek seviyede olmalı.

#### 7b.4 Doğrulama (beklenen çıktılarla)

```bash
# Sunucu üzerinden: nginx doğru sertifikayı sunuyor mu?
echo | openssl s_client -connect 127.0.0.1:443 -servername panel.alanadiniz.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
# Beklenen: subject CN=panel.alanadiniz.com (veya *.alanadiniz.com), issuer=kendi CA'n, notAfter gelecek tarihi gösterir

# Zincir bütünlüğü: kök CA'yı verince doğrulama BAŞARILI olmalı
echo | openssl s_client -connect 127.0.0.1:443 -servername panel.alanadiniz.com -CAfile kok-ca.crt 2>/dev/null | grep 'Verify return code'
# Beklenen: Verify return code: 0 (ok)

# Kök CA verilmeden test (dış kullanıcının gördüğü durum)
echo | openssl s_client -connect 127.0.0.1:443 -servername panel.alanadiniz.com 2>/dev/null | grep -E 'verify error|Verify return code'
# Beklenen: unable to get local issuer certificate / code: 20 — iç CA sertifikası için BU NORMALDİR

docker compose logs --tail=50 app | grep -iE 'ssl|certificate'
```

#### 7b.5 Backend'e şifreli devam: termination mi, re-encrypt mi

| Proxy host **Scheme** değeri | İç ağda ne olur | Ne zaman |
|---|---|---|
| `http` | **Termination**: TLS yalnız istemci↔NPM arasında; NPM→backend düz HTTP | Varsayılan ve en basit; backend'de sertifika gerekmez |
| `https` | **Re-encrypt (bridging)**: iç ağ da şifreli | KVKK/veya ağ izolasyonu gereksinimi varsa |

> ⚠️ `Scheme: https` seçtiğinde backend'in sertifikası NPM tarafından **doğrulanamazsa bağlantı kurulamaz ve 502 alırsın**. NPM arayüzünde "backend sertifikasını doğrula/kabul et" alanı yoktur; iş **Advanced** sekmesinde çözülür:
>
> ```nginx
> # Yöntem 1 — iç CA kökünü tanıtarak doğrulamayı AÇIK tut (önerilen)
> proxy_ssl_verify on;
> proxy_ssl_trusted_certificate /data/internal-ca.pem;
> proxy_ssl_name panel-backend.alanadiniz.local;
>
> # Yöntem 2 — doğrulamayı kapat (iç ağda kabul edilebilir, kayıt altına al)
> proxy_ssl_verify off;
> ```
>
> `internal-ca.pem` dosyasını yığında `./data/internal-ca.pem` olarak tut; konteyner içindeki karşılığı `/data/internal-ca.pem` olur (`./data:/data` mount'u sayesinde). Yüklemeden sonra erişimi doğrula: `docker compose exec app ls -l /data/internal-ca.pem`.

#### 7b.6 Yenileme ve değiştirme

> 🔑 **Kritik fark:** Custom sertifika **otomatik yenilenmez**. `docker compose logs | grep renew` çıktısı yalnız Let's Encrypt kayıtları için anlamlıdır. Bitiş tarihini (`7b.2`deki `notAfter`) takvim/alert ile izle; NPM ekranındaki **Expires On** sütunu tek başına izleme planı değildir.

Değiştirme sırası (kesintisiz):

1. Yeni sertifikayı **ayrı bir kayıt olarak** ekle (ör. `kurum-2027`) — eski kaydın üzerine yazma.
2. Proxy host'un SSL sekmesinde kaydı yeni sertifikayla değiştir, **Save** (NPM nginx config'ini üretir ve reload eder; kesinti olmaz).
3. Tüm hostların geçtiğini doğruladıktan sonra eski kaydı sil.

> ⚠️ Sırayı tersine çevirme: önce eski kaydı silersen, o kaydı kullanan host'ların TLS'i düşer. NPM kullanılan bir sertifikayı silmeye çoğu sürümde izin vermeyebilir; bu durumda host'ları tek tek taşıdıktan sonra sil.

> 🔒 Private key'ler `./data` altında saklanır; Adım 10.2'deki yedekleme bu klasörü de kapsıyor. Arşivi sunucu dışına çıkarırken **şifrele** — sertifika değil, anahtar kritiktir.

#### 7b.7 Sık karşılaşılan hatalar

| Belirti | Sebep | Çözüm |
|---|---|---|
| "certificate and private key do not match" | Yanlış anahtar/sertifika çifti | 7b.2'deki modulus md5 karşılaştırması |
| Yükleme sonrası site açılmıyor, konteyner restart döngüsü | Anahtar **passphrase**'li | `openssl rsa -in k -out k_plain` ile şifresizleştir |
| Bazı cihaz/tarayıcılarda "AUTHORITY_INVALID", bazılarında yok | **Ara CA eksik veya sıra yanlış** | `cat server.crt intermediate.crt` zincirini yeniden yükle |
| Kayıt oluştu ama proxy host'ta listelenmiyor | Custom yerine LE kaydı seçili / host'un domain'i SAN listesinde yok | Sertifikanın `subjectAltName` listesinde o alan adının olup olmadığına bak |
| Backend 502 veriyor | `Scheme: https` + doğrulanmayan backend cert | 7b.5'teki iki yöntemden biri |
| `*.alanadiniz.com` varken alt küme kırılıyor | Wildcard tek seviye kapsar | İki seviyeli ad için ayrı/wildcard sertifika |
| HSTS sonrası iç CA site açılmıyor | HSTS + güvenilmeyen CA kombinasyonu tarayıcıda geçişsiz hâle getirir | 9.3'teki gibi kısa `max-age` ile geri al; dış uçlarda public cert kullan |

---

### Adım 8: Erişim Kısıtlaması (Access Lists)

Hassas yönetim panellerinize parola ve/veya IP koruması koymak için:

1. Menüden **Access Lists** sekmesine gelin.
2. **Add Access List** butonuna tıklayın.
3. **Details** kısmında bir isim verin (Örn: *Admin Koruması*).
4. Kullanıcı adı/parola tanımlayın; isterseniz IP izin listesine ağlarınızı yazın (ör. `192.168.1.0/24`).
5. Oluşturduğunuz listeyi herhangi bir **Proxy Host** ayarlarındaki **Access List** seçeneğinden bağlayın.

> ⚠️ Parola **ve** IP kısıtını aynı listede kullandığınızda, nginx tarafında `satisfy any` / `satisfy all` davranışı devreye girer. Bu seçeneğin arayüzdeki adı ve varsayılan değeri sürüme göre değişebildiği için, kurulumunuzda **hangi koşulun erişime izin verdiğini mutlaka test edin** (parolasız ama IP'nizden olan bir istekle).

> 🔒 Basic Auth kimlik bilgisi yalnızca Base64 ile kodlanır, **şifrelenmez**. Access List'i sadece HTTPS'i çalışan (Force HTTPS açık) hostlarda kullanın; HTTP üzerinden parolalar ağda düz okunabilir hâle gelir.

> 💡 Access List, uygulamanın kendi login ekranının **önüne** geçen bir katmandır. API/otomasyon kullanan araçlar bu katmanda `401` alabilir; böyle bir entegrasyon varsa o host için parola yerine yalnızca IP kısıtı kullanın.

---

### Adım 9: Güvenlik Sertleştirmeleri

#### 9.1 Güvenlik Duvarı Yapılandırması

Kullandığınız güvenlik duvarı çözümünde (UFW, iptables, FortiGate, pfSense vb.) aşağıdaki portlara izin verin:

| Port | Protokol | Açıklama |
|---|---|---|
| **22** | TCP | SSH Uzak Erişim |
| **80** | TCP | HTTP Trafiği & SSL Doğrulama |
| **443** | TCP | HTTPS Trafiği |
| **81** | — | **AÇMAYIN.** Yönetim paneli; yalnız SSH tüneli üzerinden erişilir |

> ⚠️ **Docker ve güvenlik duvarı ilişkisi (en sık atlanan nokta):** Docker, `ports:` ile yayınlanan konteyner portları için kendi iptables/nftables kurallarını (DOCKER ve DOCKER-USER zincirleri) **FORWARD hattına** yazarken UFW'un `INPUT` kurallarını baypas eder. Yani `ufw deny 81` vermek, `- '81:81'` şeklinde yayınlanmış bir portu dışarıya kapatmaz. Bu rehberdeki `127.0.0.1:81:81` bağlama biçimi bu yüzden tercih edilmiştir; başka konteynerlerin portlarını da dışarı açmadan `expose` + ortak ağ ile yönlendirin. Doğrudan kural gerekiyorsa `DOCKER-USER` zincirini kullanın.

> 💡 **Port 81 (Yönetim Paneli):**  
> Adım 3'teki yapılandırmada port 81 zaten `127.0.0.1:81:81` olarak sadece localhost'a bağlı tutulmuştur. Uzaktan erişim için SSH tüneli kullanın:
>
> ```bash
> ssh -L 8181:127.0.0.1:81 kullanici@SUNUCU_IP_ADRESI
> ```
>
> Ardından yerel tarayıcınızdan `http://localhost:8181` ile bağlanın.

#### 9.2 Docker Ortak Ağ (Custom Docker Network) Kullanımı

Diğer Docker konteynerlerinizin portlarını dışarı açmadan (`ports:` yazmadan) güvenli haberleşme için ortak bir Docker ağı oluşturun:

```bash
docker network create web-network
```

NPM `docker-compose.yml` dosyanıza bu ağı ekleyin ve `app` servisine bağlayın:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:2.16.0'  # latest değil; sabit sürüm (bkz. Adım 10.1)
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

> ⚠️ Port 81'i `127.0.0.1:81:81` olarak bağladığınızda panele **konteynerlerden de** erişemezsiniz: bu bağlama yalnızca host'un loopback adresindedir, `172.x` köprüsü üzerinden gelen istekleri reddeder. Yani paneli kendi ürettiğiniz bir proxy host üzerinden yayınlamak isterseniz önce `ports:` satırını geçici olarak `81:81` yapmanız gerekir — ve bunu yapıyorsanız Adım 5'teki MFA + Access List katmanlarını mutlaka açın.

#### 9.3 Proxy Host Düzeyinde Sertleştirme (Advanced Sekmesi)

NPM arayüzünün açık vermediği ama en çok ihtiyaç duyulan üç ayar, Proxy Host'un **Advanced** sekmesine yazılır:

```nginx
# 1) Büyük dosya yükleme/413 hataları (Nextcloud, import, yedek upload)
client_max_body_size 512m;

# 2) HSTS — Force HTTPS çalıştıktan ve birkaç gün gözlemledikten sonra ekleyin
add_header Strict-Transport-Security "max-age=31536000" always;

# 3) CDN/Cloudflare arkasındaysanız gerçek istemci IP'si
set_real_ip_from 173.245.48.0/20;   # örnek; güncel listeyi sağlayıcının dokümanından alın
real_ip_header   CF-Connecting-IP;  # proxy kullanmıyorsanız X-Forwarded-For
real_ip_recursive on;
```

> ⚠️ **HSTS konusunda dikkat:** `max-age` bir kez tarayıcıya yayıldığında alan adı o süre boyunca HTTPS'e kilitlenir; sertifika/yapılandırma hatasında siteyi kendi tarayıcınızdan da açamazsınız. Önce kısa bir değerle (`max-age=600`) deneyin.

> 💡 `client_max_body_size` sadece NPM tarafında çözümez; arkadaki uygulamanın kendi upload limiti (ör. PHP `upload_max_filesize` / `post_max_size`) de yükseltilmelidir.

#### 9.4 Bu Rehberde Anlatılmayan NPM Özellikleri

Kurulum temelleri dışında şu özellikler de kullanılabilir durumdadır: **Streams** (TCP/UDP port yönlendirme — SSH, veritabanı, oyun sunucusu), **Redirection Hosts** (`www` → apex gibi yönlendirmeler), **Custom Hosts / 404 Hosts**, **SSL/TLS Certificates** ekranından kendi sertifikanı yükleme (bkz. Adım 7b) ve **Audit Log**. Ayrıca panel `Settings` bölümü üzerinden ikinci yönetici hesabı ve rol/permission tanımları yapılabilir.

---

### Adım 10: Yedekleme ve Güncelleme

#### 10.1 Sürüm Güncelleme

NPM, açılışta veritabanı şemasını **otomatik** olarak yeni sürüme taşır ve bu işlem tek yönlüdür. Bu yüzden güncelleme öncesi yedek almak ve sürümleri **kademeli** yükseltmek gerekir.

```bash
cd /opt/nginx-proxy-manager

# 1. Yedek alın (Adım 10.2) — şema migrate'si geri alınabilir değildir
# 2. Hedef sürümün release notes'unu okuyun (sürüme özel adımlar olabiliyor)
# 3. Image etiketini yeni sabit sürümle değiştirin (latest değil!)
nano docker-compose.yml      # ör. 2.16.0 -> sonraki sürüm

docker compose pull
docker compose up -d         # konteyneri yeni imajla yeniden oluşturur
docker compose ps
docker compose logs --tail=50 app
docker image prune -f
```

> ⚠️ **`latest` etiketiyle yükseltme yapmayın:** `latest` ile `docker compose pull`, hangi sürümden hangi sürüme atladığınızı göstermez; bir release'te dönen bir hata olursa (geçmişte yaşandı) hangi sürüme döneceğinizi bilemezsiniz. Sabit etiket hem rollback'i hem de "bu rehber hangi sürümde test edildi" sorusunu netleştirir.
>
> **Rollback:** Etiketi eski sabit değere döndürüp `docker compose up -d` çalıştırın. Ancak veritabanı şeması çoktan migrate edilmiş olabilir; en güvenli dönüş Adım 10.3'teki geri yüklemedir. Küçük adımlarla yükseltin.
>
> Docker Hub anonim indirme kotasına takmamak için üretim sunucusunda `docker login` kullanın veya imajı kendi registry'nize kopyalayın.

#### 10.2 Manuel Yedekleme

**Önce döküm aracının adını doğrulayın.** MariaDB Docker Official Image'ında komut adı sürüme göre değişir: 11.0 ve sonrasında yalnızca `mariadb-dump` bulunur (eski `mysqldump` symlink'i kaldırıldı), 10.11 gibi eski imajlarda `mysqldump` her zaman vardır:

```bash
docker compose exec db sh -c 'command -v mariadb-dump || command -v mysqldump'
```

```bash
cd /opt/nginx-proxy-manager
sudo mkdir -p /backup && sudo chmod 700 /backup
set -a; source ./.env; set +a          # şifreleri .env'den oku, elle yazmayın
DUMP=mariadb-dump                       # üstteki kontrol sonuçsuzsa: mysqldump
TS=$(date +%F)

# 1. Veritabanı dökümü (konteyner çalışırken). -T: betik içinde TTY atanmaz.
docker compose exec -T -e MYSQL_PWD="$MYSQL_ROOT_PASSWORD" db $DUMP -u root npm \
  | gzip > /backup/npm_db_$TS.sql.gz

# 2. Dosya sistemi: ./data (JWT anahtarları + loglar) ve ./letsencrypt (sertifikalar)
docker compose down
sudo tar -czf /backup/npm_files_$TS.tar.gz -C /opt nginx-proxy-manager

# 3. Konteynerleri tekrar başlatın
docker compose up -d
```

> ⚠️ **Yedeği, yedeklediğiniz dizinin İÇİNE yazmayın.** Arşiv kaynak ağacın içinde oluşturulursa `tar` kendisini fark edip `file is the archive; not dumped` uyarısı verir ve yarım/yanıltıcı bir yedek üretir. `/backup` gibi harici bir hedef kullanın (yukarıdaki `-C /opt` de aynı amaca hizmet eder: mutlak yol yerine bağıl saklama).

> 💡 **MariaDB binary dosyalarını** (`mysql/` dizini) çalışır durumdaki konteynerde `tar` ile yedeklemek veri tutarsızlığına yol açabilir; bu akış önce SQL dökümü alır, sonra `down` ile durdurup tam yedek çeker.

> 💡 **SQLite kullanıyorsanız** döküm adımını atlayın, ancak **yalnızca `data/database.sqlite` dosyasını kopyalamayın:** ilk açılışta üretilen JWT imzalama anahtarları da `./data` içinde durur. Adım 2'deki arşiv yeterlidir.

> ⚠️ **Döküm dosyası uyumluluğu:** MariaDB 10.6.18 / 10.11.8 / 11.x sonrası dökümlerin başına `/*!999999\- enable the sandbox mode */` satırı eklenir. Eski MySQL/MariaDB istemcileri bunu `ERROR at line 1: Unknown command '\-'` ile reddeder. Dökümü başka bir sunucuya aktaracaksanız ya güncel bir MariaDB istemcisi kullanın ya da o satırı dosyadan çıkarın.

> 🔒 Arşiv `.env` ve şifreleri de içerir; `chmod 700` yalnızca yerel izin önlemidir, **şifreleme değildir**. Sunucu dışına çıkaracaksanız `gpg -c /backup/npm_files_$TS.tar.gz` ile şifreleyin.

#### 10.3 Geri Yükleme

```bash
cd /opt/nginx-proxy-manager
docker compose down

# 1. Dosyaları geri yükleyin (docker-compose.yml, .env, data/, letsencrypt/)
sudo tar -xzf /backup/npm_files_<TARIH>.tar.gz -C /opt

# 2. Önce veritabanını ayağa kaldırın, sonra dökümü içe aktarın
docker compose up -d db
sleep 15
gunzip -c /backup/npm_db_<TARIH>.sql.gz | \
  docker compose exec -T -e MYSQL_PWD="$MYSQL_ROOT_PASSWORD" db mysql -u root npm
# 11.x imajlarda istemci adı `mysql` yerine `mariadb` olabilir:
# docker compose exec db sh -c 'command -v mysql || command -v mariadb'

# 3. Paneli başlatın ve doğrulayın
docker compose up -d
docker compose ps
docker compose logs --tail=50 app
```

> ⚠️ **Sıra önemli:** SQL dökümünü dosyalar yerine oturduktan sonra yükleyin. Resmî dokümanın belirttiği gibi JWT anahtarları `data` klasöründe üretilir; yalnızca dökümü geri yüklerseniz imzalama anahtarları ile veritabanındaki oturum/2FA kayıtları uyumsuz kalabilir ve tüm kullanıcılar yeniden giriş yapmak zorunda olur.

> 💡 **Sertifika durumu:** `letsencrypt/` klasörünü kopyalasanız bile yenilemenin çalışması için alan adı kaydının bu sunucuyu göstermesi gerekir. Yeni sunucuya taşıyorsanız DNS A kaydını da taşıyın, ardından `docker compose logs -f | grep -i renew` ile ilk yenilemeyi gözlemleyin.

---

## 💡 Sık Karşılaşılan Hatalar ve Çözümleri

1. **`127.0.0.1` Tuzağı:** NPM Docker içinde çalıştığından Forward IP alanına `127.0.0.1` yazmak NPM'in kendi konteynerine yönlenmesine sebep olur. Host servisleri için `host.docker.internal`, konteynerler için ortak ağ üzerinde konteyner adını kullanın (Adım 6).
2. **SSL `Internal Error` Hatası:** Port 80/443'ün dış dünyadan erişilebilir olduğundan ve DNS A kaydının doğru IP'ye yöneldiğinden emin olun. Hata devam ederse 3-4 denemeden sonra durun (saatte 100 başarısız order limitine takılmamak için) ve `docker compose logs app` çıktısındaki certbot satırlarını okuyun.
3. **`Address already in use` Hatası:** `sudo ss -tlpn | grep -E ':80|:443'` ile işgalciyi bulun. Hostta kurulu `nginx`/`apache2`/`caddy` veya başka bir konteynerin `ports:` yayını sık görülen sebeplerdir.
4. **502 Bad Gateway:** Hedef yanlış veya ulaşılamaz. İki adımı sırayla ayır: (1) isim çözülüyor mu — `docker compose exec app getent hosts host.docker.internal` (boş çıktıysa `extra_hosts` satırı eksil/yanlış), (2) çözülen hedefe erişilebiliyor mu — `docker compose exec app sh -c 'wget -qO- http://host.docker.internal:8080 >/dev/null && echo ERISIM_VAR'` (imaj busybox `wget` taşır; `curl` her sürümde olmayabilir). Konteyner adı yazdıysan aynı ağda olup olmadıklarını `docker network inspect web-network` ile doğrula.
5. **504 Gateway Time-out:** Uygulama geç cevap veriyor. Proxy Host **Advanced** sekmesine `proxy_read_timeout 300s;` ve `proxy_send_timeout 300s;` ekleyin.
6. **WebSocket bağlantısı ~60 saniyede kopuyor:** **Websockets Support** açık olmalı; ayrıca sessiz kalan bağlantılar timeout'a girdiği için `proxy_read_timeout` yükseltilmelidir. (Bu iki madde 502/504 hatalarından farklıdır: 502 hedefin ulaşılamazlığını, 504 yavaşlığı gösterir.)
7. **413 Request Entity Too Large:** Adım 9.3'teki `client_max_body_size`; arkadaki uygulamanın kendi upload limitini de yükseltin.
8. **Panelde değişiklik sonrası siteler açılmıyor:** Üretilen nginx config'i geçersiz kılınmıştır. `docker compose logs app | grep -iE 'nginx.*(emerg|alert|failed)'` ile hatayı bulun, **Audit Log**'dan son değişikliği görüp geri alın.
9. **`pull` sonrası arayüz boş/hatalı:** `DB_MYSQL_*` ile SQLite yapılandırmasının çakışması (Seçenek B uyarısı) ya da migrate edilmiş şemanın eski imajla okunamaması. Sabit etiketi eski sürüme döndürün ve gerekiyorsa 10.3'teki yedekten yükleyin.
10. **Yedek tek başına yetmez:** SQL dökümü + `./data` + `./letsencrypt` birlikte alınmalıdır (Adım 10.2).

> 📌 **Sürüm notu:** Bu rehber NPM `2.16.0`, Docker Compose v2 ve Debian 13 (trixie) esas alınarak hazırlanmıştır. Arayüz etiketleri sürümler arasında değişebildiği için, bir alanı farklı adla görürseniz aynı işi yapan seçeneği arayüzde doğrulayın.

*Tebrikler! Debian 13 üzerinde Nginx Proxy Manager kurulumunuz başarıyla tamamlanmıştır.*

---

- Author: faruk-guler
- GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
- Page: [www.farukguler.com](https://www.farukguler.com)
