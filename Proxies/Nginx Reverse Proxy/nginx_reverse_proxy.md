# Nginx Reverse Proxy on Debian 13 (Trixie)

**Reverse proxy (ters proxy)**, dış dünyadan gelen istekleri tek bir kapıdan (80/443) alıp, sunduğu sertifikayı doğruladıktan sonra içerideki doğru servise yönlendiren katmandır. Dışarıdan bakıldığında sadece `nginx` görülür; arkadaki Portainer, Nextcloud, API servisleri görünmez.

Bu rehber, **Debian 13 (Trixie)** üzerinde Debian deposundan kurulan `nginx` ile elle reverse proxy yapılandırmasını; Let's Encrypt SSL veya **kendi sertifikanı kullanma**, WebSocket/upload ayarları, güvenlik sertleştirmeleri, log/sorun giderme, yedekleme-geri yükleme, **L4 (TCP/UDP) `stream` yönlendirme** ve **`proxy_cache` ile reverse cache** adımlarını kapsamaktadır. Grafik arayüzli bir araçla (Nginx Proxy Manager) değil, **doğrudan konfigürasyon dosyalarıyla** çalışılır.

> 💡 **Bu rehber kimin için?** 1-3 servis yayınlayıp nginx'i öğrenmek/kontrol etmek isteyenler için idealdir. Onlarca domain, sık değişen yönlendirme ve çok kişilik yönetim gerekiyorsa Nginx Proxy Manager daha uygundur; en sondaki karşılaştırma tablosuna bakın.

---

## 📘 Adım Adım Kurulum

### Adım 1: Sistem Hazırlığı ve Port Kontrolü

Paketleri güncelleyin ve saatin doğru çalıştığından emin olun (sertifika tarihi doğrulaması sistem saatine bağlıdır):

```bash
# 1.1 Sistem güncellemesi
sudo apt update && sudo apt upgrade -y

# 1.2 Saat/NTP kontrolü (sertifika işlemleri için şart)
timedatectl | grep -E 'synchronized|Time zone'

# 1.3 80 ve 443 portları boş mu? (Boşsa çıktı hiç gelmez)
sudo ss -tlpn | grep -E ':80 |:443 '
```

> ⚠️ 80 veya 443 doluysa mevcut servisi tespit edip durdurun (`apache2`, `nginx`, başka bir konteyner, `caddy`). İki reverse proxy aynı anda aynı portu paylaşamaz.

---

### Adım 2: Nginx Kurulumu ve Varsayılan Sitenin Kaldırılması

```bash
# 2.1 Nginx'i Debian deposundan kurun
sudo apt install -y nginx
nginx -v

# 2.2 Servisi başlatın ve açılışta gelmesini sağlayın
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager

# 2.3 Yerelde ilk testi yapın (Debian varsayılan karşılama sayfası dönmezse servis çalışıyordur)
curl -I http://127.0.0.1
```

Debian 13 deposundaki `nginx` **1.26.x** serisindedir. Bu sürüm numarası ileride önemli olacak (HTTP/2 söz dizimi, Adım 6).

Kendi alan adlarımızı kullanacağımız için açılıştaki varsayılan site devre dışı bırakılır:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

> 💡 Bu symlink'i silmek konfigürasyonu silmez; `/etc/nginx/sites-available/default` durur. Geri almak için `sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/` yeterlidir.

#### 2.4 Güvenlik Duvarı

```bash
sudo apt install -y ufw
sudo ufw allow OpenSSH        # ÖNCE SSH, SONRA enable — aksi halde bağlanamazsınız
sudo ufw enable
sudo ufw allow 'Nginx Full'   # 80 + 443
sudo ufw status verbose
```

| Port | Protokol | Açıklama |
|---|---|---|
| **22** | TCP | SSH Uzak Erişim |
| **80** | TCP | HTTP Trafiği & ACME (SSL) Doğrulama |
| **443** | TCP | HTTPS Trafiği |

> 🔒 Nginx host üzerinde (`systemd`) çalıştığından UFW kuralları **normal şekilde uygulanır**. Docker yayınlanmış portların UFW'ü bayması problemi burada yoktur.

---

### Adım 3: Konfigürasyon Yapısını Tanıyın

```
/etc/nginx/
├── nginx.conf                    # Ana dosya; http{} bloğu include'ları çağırır
├── conf.d/*.conf                 # Genel ayarlar (rate limit zone vb.) — ÖNCE yüklenir
├── sites-available/*.conf        # Her site için asıl config dosyaları (siz yazırsınız)
├── sites-enabled/                # Etkin siteler (sites-available'a symlink)
└── snippets/*                    # fastcgi-php.conf, ssl-*.conf gibi hazır parçalar
```

Debian'ın `nginx.conf` dosyası `http {}` bloğu içinde sırasıyla şunları yükler:

```nginx
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

> ⚠️ **Yükleme sırası:** `conf.d` her zaman `sites-enabled`'dan **önce** okunur. Aynı `server_name` iki yerde tanımlıysa ilk eşleşen kazanır ve `nginx -t` çıktısında `conflicting server name` uyarısı görürsünüz.

Pratik kural:

- **Her domain/servis için** `sites-available/<domain>.conf` yazın, `sites-enabled/` içine symlink koyun.
- **Tüm sunucuya etki eden** (`http {}` seviyesindeki) ayarları `conf.d/` içine yazın (log format, rate limit zone, `server_tokens off`).

Yüklü tüm config'i tek seferde görmek için en faydalı komut:

```bash
sudo nginx -T | less
```

---

### Adım 4: İlk Reverse Proxy (Şimdilik HTTP ile Test)

Senaryo: Sunucunuzda `127.0.0.1:8080` portunda çalışan bir panel var ve bunu `panel.alanadiniz.com` üzerinden yayınlayacaksınız.

```bash
sudo nano /etc/nginx/sites-available/panel.alanadiniz.com.conf
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name panel.alanadiniz.com;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
    }
}
```

```bash
# Etkinleştirin
sudo ln -s /etc/nginx/sites-available/panel.alanadiniz.com.conf /etc/nginx/sites-enabled/

# Söz dizimini doğrulayın (reload'dan ÖNCE şart)
sudo nginx -t

# Sorunsuzsa yapılandırmayı yeniden yükleyin (kesinti olmaz)
sudo systemctl reload nginx

# Sunucu üzerinden test: Host header'ı elle vererek
curl -I -H 'Host: panel.alanadiniz.com' http://127.0.0.1
```

> ⚠️ **DNS uyarısı:** Tarayıcıdan erişmeden önce `panel.alanadiniz.com` A kaydının sunucunuzun **public IP**'sine yönlenmiş olması gerekir. Ev internetinde ISP'niz 80/443'ü kapattıysa veya CGNAT arkasındaysanız HTTP-01 doğrulaması çalışmaz; Adım 5'te DNS challenge'a bakın.

#### 4.1 `proxy_pass` Hedefi Nasıl Seçilir?

Düz nginx host'ta çalıştığından `127.0.0.1` **gerçekten host'un kendi** adresidir — konteyner tabanlı araçlardaki "kendi içine dönme" tuzağı burada yoktur.

| Hedef servis | Yazılacak adres |
|---|---|
| Host'ta port dinleyen bir uygulama | `http://127.0.0.1:<port>` |
| LAN'daki başka bir makine | `http://192.168.1.50:8080` |
| Docker konteyneri (port yayınlamış) | `http://127.0.0.1:<yayınlanan-port>` |
| Birden çok arka uç / load balancing | `upstream` bloğu (Adım 4.2) |

#### 4.2 İsteğe Bağlı: `upstream` ile Havuz

```nginx
upstream panel_backend {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:8080 max_fails=3 fail_timeout=30s backup;
    keepalive 32;
}

server {
    # ...
    location / {
        proxy_pass http://panel_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

---

### Adım 5: Ücretsiz SSL Sertifikası (certbot)

```bash
sudo apt install -y certbot python3-certbot-nginx
```

#### 5.1 Otomatik Yöntem (config'i certbot düzenler)

```bash
sudo certbot --nginx -d panel.alanadiniz.com
# Birden fazla alias: -d panel.alanadiniz.com -d www.panel.alanadiniz.com
```

`--nginx` seçeneği sertifikanın yanına `ssl_certificate` satırlarını ve HTTP→HTTPS yönlendirmesini **kendi dosyanıza yazar**. Bu pratik ama dosyanızın sizin yazdığınız hâlini değiştirir.

#### 5.2 Kontrollü Yöntem (önerilen: sadece sertifika al, config'i siz yönetin)

```bash
sudo certbot certonly --nginx -d panel.alanadiniz.com
```

Sertifika dosyaları:

```
/etc/letsencrypt/live/panel.alanadiniz.com/fullchain.pem   # nginx'te bu kullanılır
/etc/letsencrypt/live/panel.alanadiniz.com/privkey.pem     # private key — asla dışa vermeyin
/etc/letsencrypt/archive/...                                # gerçek dosyalar; live/ sadece symlink
```

> ⚠️ `cert.pem` değil **`fullchain.pem`** kullanın. `cert.pem` tek başına zinciri tamamlamaz ve bazı istemciler/telefon tarayıcıları sertifikayı güvenilmez sayar.

#### 5.3 Yenileme (Otomatik mi Çalışıyor?)

Debian paketi systemd timer ile gelir:

```bash
systemctl list-timers | grep -i certbot
sudo certbot renew --dry-run          # Tüm sertifikaları zorlamadan test eder
```

`--dry-run` hatasız geçmiyse gerçek yenileme de çalışmaz; `journalctl -u certbot.renew.service` ya da `/var/log/letsencrypt/letsencrypt.log` içinde hatayı arayın.

#### 5.4 Port 80 Açılmıyorsa / Wildcard Sertifika (DNS-01)

```bash
# Manuel DNS kaydı ekleyerek (wildcard dahil) — yenilemede de sorulur
sudo certbot certonly --manual --preferred-challenges dns \
     -d alanadiniz.com -d '*.alanadiniz.com'
```

Otomatik yenileme istiyorsanız DNS sağlayıcınızın certbot eklentisini kurun (ör. `python3-certbot-dns-cloudflare`) ve `--dns-cloudflare-credentials` ile bir kimlik dosyası verin.

> ⚠️ **Let's Encrypt limitleri:** Registered domain başına 7 günde en fazla 50 sertifika (bu limit o domain için **tüm dünyadan** gelen talepleri sayar) ve aynı isim seti için haftada 5 "duplicate" sertifika. Yanlış config ile ısrarla denemek limiti tüketir; yeni bir alan adı eklerken önce `--dry-run` kullanın. Staging API (`acme-staging-v02`) limiti tüketmemek için geliştirme amaçlıdır, tarayıcı onu güvenilmez sayar.

---

### Adım 5b: Kendi Sertifikanı Kullanma (certbot dışı / kurumsal CA)

Kurumdan aldığın veya kendi iç CA'nda ürettiğin bir sertifikayı kullanacaksan certbot'a hiç ihtiyaç yok — nginx zaten `ssl_certificate` iki dosyayla çalışır. Fark şu: **bitiş tarihini ve zincir doğruluğunu sen takip edersin**, otomatik yenileme yoktur.

#### 5b.1 Dosyaları yerleştir

```bash
sudo mkdir -p /etc/nginx/ssl/panel.alanadiniz.com
cd /etc/nginx/ssl/panel.alanadiniz.com
# domain.crt (sunucu), intermediate.crt (ara CA), domain.key (anahtar) buraya kopyalanır

# Zinciri sırayla birleştir: sunucu sertifikası ÖNCE, ara CA sonra. KÖK CA'yı ekleme!
cat domain.crt intermediate.crt > fullchain.pem

sudo chmod 600 domain.key fullchain.pem
sudo chown root:root domain.key fullchain.pem
```

> 💡 nginx worker'ları `www-data` olarak çalışır ama sertifika ve anahtar **master süreç (root) tarafından, yetkiler düşürülmeden önce** okunur. Bu yüzden `root:root 600` doğrudur; `www-data`'ya okutmana gerek yok (gerekse `ssl_password_file` gibi ayrıntılara hiç girmemiş olursun).

> ⚠️ **Anahtar şifreli olmamalı.** Passphrase'li key verirsen `nginx -t` şu tipte hata verir: `cannot load certificate key ... PEM_read_bio_PrivateKey() failed`. Çözüm: `openssl rsa -in encrypted.key -out domain.key`.

`.pfx/.p12` olarak verildiyse önce PEM'e çevir:

```bash
openssl pkcs12 -in Sertifika.pfx -out domain.crt      -nokeys  -clcerts -passin pass:SIFRE
openssl pkcs12 -in Sertifika.pfx -out domain.key      -nocerts -nodes   -passin pass:SIFRE
openssl pkcs12 -in Sertifika.pfx -out intermediate.crt -nokeys -cacerts -passin pass:SIFRE
```

#### 5b.2 Server bloğunu kendi sertifikanla kur

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name panel.alanadiniz.com;

    ssl_certificate     /etc/nginx/ssl/panel.alanadiniz.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/panel.alanadiniz.com/domain.key;

    # ssl_trusted_certificate YALNIZ OCSP stapling kullanıp responder imzasını doğrulayacaksan gerekir.
    # İstemci sertifikası doğrulamak ayrı bir iştir: onun için ssl_client_certificate kullanılır.
    # ssl_trusted_certificate /etc/nginx/ssl/panel.alanadiniz.com/intermediate.crt;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache   shared:SSL:10m;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> ⚠️ `ssl_certificate` ve `ssl_certificate_key` **`server` seviyesinde** olmalı; `location` içine yazılmaz (yazarsan `"ssl_certificate" directive is not allowed here` alırsın).

#### 5b.3 Doğrulama (beklenen çıktılarla)

```bash
# 1) Anahtar ↔ sertifika eşleşmesi: iki md5 AYNI çıkmalı
openssl x509 -noout -modulus -in domain.crt | openssl md5
openssl rsa  -noout -modulus -in domain.key | openssl md5

# 2) Kapsanan adlar ve tarihler (CN yetmez, SAN listesi şart)
openssl x509 -noout -subject -issuer -dates -ext subjectAltName -in domain.crt

# 3) Zincir bütünlüğü
openssl verify -CAfile kok-ca.crt -untrusted intermediate.crt domain.crt
# Beklenen: domain.crt: OK          ("unable to get issuer certificate" → ara CA yanlış/eksik)

# 4) nginx söz dizimi + TLS el sıkışması
sudo nginx -t
# Beklenen: nginx: configuration file /etc/nginx/nginx.conf test is successful
echo | openssl s_client -connect 127.0.0.1:443 -servername panel.alanadiniz.com -CAfile kok-ca.crt 2>/dev/null | grep 'Verify return code'
# Beklenen: Verify return code: 0 (ok)
```

> 🔒 `-CAfile` **verilmezse** iç CA sertifikası `unable to get local issuer certificate` döner — bu **normal ve beklenen** davranıştır: senin kök CA'na güvenen cihaz listesi, senin GPO/MDM ile dağıttığın cihazlardır. İnternete açık serviste bu davranış kullanıcıya uyarı olarak yansır; dış uçlar için public CA kullan.

#### 5b.4 Sık karşılaşılan hatalar

| Belirti | Sebep | Çözüm |
|---|---|---|
| `cannot load certificate ... PEM_read_bio_PrivateKey() failed` | Anahtar şifreli ya da yanlış anahtar | `openssl rsa` ile şifresizleştir; modulus eşleşmesine bak |
| `"ssl_certificate" directive is not allowed here` | Direktif `location` içine yazılmış | `server` bloğuna taşı |
| `curl -k` ile açılıyor ama tarayıcı uyarıyor | `cert.pem` tek başına konmuş, ara CA yok | `fullchain.pem` kullan |
| Bazı Android/Java istemcileri kırık, masaüstü normal | Zincir sırası yanlış veya ara CA eksik | sunucu sertifikası → ara CA sırasıyla `cat`'la |
| `nginx -t` geçiyor ama 443 dışarıdan kapalı | `listen 443 ssl` yok ya da firewall | `ss -lntp \| grep 443`, `ufw status` |
| Yenileme sonrası eski sertifika görünüyor | `reload` edilmemiş (nginx sertifikayı her reload'da diskten okur) | `sudo systemctl reload nginx` |

#### 5b.5 Bitiş tarihi takibi

```bash
# Tek satırda son kullanma tarihi
openssl x509 -enddate -noout -in /etc/nginx/ssl/panel.alanadiniz.com/fullchain.pem
# Beklenen: notAfter=Aug  1 10:00:00 2027 GMT

# 30 günden az kaldıysa uyaran basit kontrol
for f in /etc/nginx/ssl/*/*.pem; do
  openssl x509 -checkend $((30*86400)) -noout -in "$f" >/dev/null || echo "YAKINDA BİTİYOR: $f"
done
```

> ⚠️ certbot'un yenileme timer'ı **bu sertifikaları yenilemez** (kayıtlı değildir); `certbot certificates` listesinde de görünmezler. Takvime ya da yukarıdaki `openssl x509 -checkend` döngüsünü bir cron/görevine bağlamaya ihtiyacın var.

---

### Adım 6: HTTPS, HTTP/2 ve Zorunlu Yönlendirme (Nihai Config)

`certonly` kullandıysanız dosyanızın son hâli şöyle olmalı. Kendi sertifikanı kullanıyorsan (Adım 5b) tek fark bu iki satırın yoludur: `ssl_certificate` ve `ssl_certificate_key`.

```bash
sudo nano /etc/nginx/sites-available/panel.alanadiniz.com.conf
```

```nginx
# HTTP: sadece ACME doğrulaması + HTTPS'e yönlendirme
server {
    listen 80;
    listen [::]:80;
    server_name panel.alanadiniz.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name panel.alanadiniz.com;

    http2 on;

    ssl_certificate     /etc/letsencrypt/live/panel.alanadiniz.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/panel.alanadiniz.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
# Sağlama: yönlendirme ve sertifika
curl -I http://panel.alanadiniz.com          # 301 Location: https://... beklenir
curl -vI https://panel.alanadiniz.com 2>&1 | grep -E 'SSL certificate|subject|expire'
```

> 💡 **`http2 on;` hakkında:** Bu söz dizimi nginx **1.25.1+** içindir ve Debian 13'ün 1.26.x sürümü bunu destekler. Eski sürümlerde (ör. Debian 12 / nginx 1.22) `http2 on;` tanınmaz; onun yerine `listen 443 ssl http2;` yazılır. Debian 13'te eski söz dizimini kullanırsanız `nginx -t` şu uyarıyı verir: `the "listen ... http2" directive is deprecated`.

> ⚠️ **HSTS'yi hemen eklemeyin.** `add_header Strict-Transport-Security "max-age=31536000"` bir kez yayıldığında tarayıcılar alan adını **6 ay boyunca** HTTPS'e kilitler; sertifika/CONFIG hatası yaparsanız siteyi kendi tarayıcınızdan da açamazsınız. HTTPS akışı birkaç gün stabil çalıştıktan sonra, önce `max-age=600` gibi kısa bir değerle deneyerek ekleyin.

---

### Adım 7: WebSocket, Upload Boyutu ve Path Bazlı Yönlendirme

#### 7.1 WebSocket (Portainer, Graphing, Chat, CI live-log)

WebSocket, HTTP upgrade ile başlar; upgrade header'ları gönderilmezse bağlantı ya hemen kapanır ya da birkaç saniye sonra kopar.

```bash
sudo nano /etc/nginx/conf.d/websocket.conf
```

```nginx
# http{} seviyesinde; upgrade isteyen istekte "upgrade", istemeyende "close"
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

`location /` bloğuna eklenecek satırlar:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
proxy_read_timeout  300s;   # sessiz kalan bağlantıyı nginx'in kapatma süresi
proxy_send_timeout  300s;
```

> 💡 `proxy_read_timeout` **veri gelmeyen** bağlantılarda çalışır; uygulamanızın ping/heartbeat aralığından **büyük** olmalı. Nginx varsayılanı 60s olduğundan "her ~1 dakikada bir kopuyor" şikâyetinin asıl çözümü bu satırdır, `Websockets Support` benzeri bir seçenek değil.

#### 7.2 413 "Request Entity Too Large" (Dosya Yükleme)

```nginx
client_max_body_size 512m;   # http, server veya location seviyesinde yazılabilir
```

Nextcloud, Owncloud, wikiler, large import'lar için şart. Yalnız nginx'i yükseltmek yetmez; **arkadaki uygulamanın kendi upload limiti** de artırılmalıdır (PHP'de `upload_max_filesize` / `post_max_size`).

#### 7.3 Path Bazlı Yönlendirme ve `proxy_pass` Sondaki `/` Tuzağı

```nginx
# /api/ öneki silinir: /api/users → backend'e /users gider
location /api/ {
    proxy_pass http://127.0.0.1:3000/;     # sondaki slash kritik
}

# öneki korur: /api/users → backend'e /api/users gider
location /api/ {
    proxy_pass http://127.0.0.1:3000;      # slash yok
}
```

> ⚠️ Bu tek karakter farkı, "çalışıyor ama tüm istekler 404 dönüyor" hatalarının en sık sebebidir. Öneki silinen durumda uygulama `X-Forwarded-Prefix` bekleyebilir: `proxy_set_header X-Forwarded-Prefix /api;`

#### 7.4 SSE / Stream Yanıtlar (Bildirim, Canlı Log, LLM Token Akışı)

Tamponlama yanıtları biriktirdiği için "gecikmeli/parça parça" görünür:

```nginx
location /events/ {
    proxy_pass http://127.0.0.1:8080;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 1h;
    add_header X-Accel-Buffering no;
}
```

---

### Adım 8: Güvenlik Sertleştirmeleri

#### 8.1 `conf.d/hardening.conf` (Tüm Siteler İçin)

```bash
sudo nano /etc/nginx/conf.d/hardening.conf
```

```nginx
server_tokens off;                 # Sunucu başlığında nginx versiyonu görünmez
keepalive_timeout 65;
client_body_timeout 15s;
client_header_timeout 15s;
send_timeout 15s;
types_hash_max_size 4096;

# Kaba kuvvet / bot trafiği için havuz (kullanımı 8.4)
limit_req_zone  $binary_remote_addr zone=genel:10m  rate=10r/s;
limit_conn_zone $binary_remote_addr zone=baglanti:10m;
```

#### 8.2 Gizli Dosyaları Kapatın — ama ACME'yi Açık Bırakın

```nginx
location ~ /\. {
    deny all;
    log_not_found off;
}
```

> ⚠️ **Kritik ayrıntı:** `/\.` deseni `.well-known` klasörünü de kapatır ve Let's Encrypt HTTP-01 doğrulaması **404** ile çöker. `.well-known/acme-challenge` için daha önce (kısa önek eşleşmesi `^~` ile) bir istisna tanımlayın:
>
> ```nginx
> location ^~ /.well-known/acme-challenge/ {
>     auth_basic off;
>     allow all;
>     root /var/www/html;
> }
> ```
>
> Aynı istisna, bir sonraki adımdaki Basic Auth için de gereklidir.

#### 8.3 Basic Auth ve IP Kısıtı (Yönetim Panelleri İçin)

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin      # -c YALNIZ ilk seferde (dosyayı ezer!)
sudo chmod 640 /etc/nginx/.htpasswd && sudo chown root:www-data /etc/nginx/.htpasswd
```

```nginx
location / {
    auth_basic          "Yonetim Paneli";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # İsteğe bağlı: yalnız belirli ağlardan aç
    allow 192.168.1.0/24;
    allow 203.0.113.7;
    deny  all;

    proxy_pass http://127.0.0.1:8080;
}
```

> 🔒 Basic Auth kimlik bilgisi Base64 ile taşınır, **şifrelenmez**. Bu yüzden sadece 443 üzerinden çalışan `server` bloğunda kullanın; 80'de `return 301` ile HTTPS'e zorlayın.

#### 8.4 Rate Limit Uygulaması

```nginx
location / {
    limit_req  zone=genel burst=20 nodelay;
    limit_conn baglanti 20;
}
```

Login endpoint'i gibi hassas yerlerde daha düşük `rate` değeri veren ayrı zone tanımlayın. Aşım anında nginx varsayılan olarak **503** döner (`limit_req_status 429;` ile 429'a çevirebilirsiniz).

#### 8.5 CDN / Cloudflare Arkasındaysa Gerçek İstemci IP'si

Modulün derlemeye dahil olduğunu doğrulayın:

```bash
nginx -V 2>&1 | grep -o -- '--with-http_realip_module'
```

```nginx
# Sadece CDN'in verdiği proxy_set_header değerlerine güvenin
set_real_ip_from 173.245.48.0/20;   # örnek; güncel IP listesi sağlayıcının dokümanından alınır
real_ip_header   X-Forwarded-For;    # Cloudflare için: CF-Connecting-IP
real_ip_recursive on;
```

Aksine, bunu yapmadan `X-Forwarded-For`'a güvenmek **IP sahteciliğine** yol açar; fail2ban ve allow/deny kuralları anlamsız hâle gelir.

#### 8.6 fail2ban (Panel Kaba Kuvvet ve Tarama Filtreleri)

```bash
sudo apt install -y fail2ban
sudo nano /etc/fail2ban/jail.local
```

```ini
[nginx-http-auth]
enabled = true
port    = http,https

[nginx-bad-request]
enabled = true
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status nginx-http-auth
```

---

### Adım 9: Loglar, Doğrulama ve Sorun Giderme

#### 9.1 Loglar

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log      # 502/504 sebepleri buraya yazılır
```

`conf.d` içine tanımlayacağınız zengin bir format, sorun çözmeyi kat kat hızlandırır:

```nginx
log_format ayrinti '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent rt=$request_time '
                   'up=$upstream_addr ust=$upstream_status urt=$upstream_response_time';

access_log /var/log/nginx/access.log ayrinti;
```

`up=127.0.0.1:8080 ust=502` gibi bir çıktı görürseniz hata **arka uçtadır**; `ust=-` ise nginx bağlantı bile kuramamıştır (yanlış port / servis kapalı). Log rotasyonu Debian paketiyle gelir (`/etc/logrotate.d/nginx`), ayrıca bir şey yapmanıza gerek yoktur.

#### 9.2 Sık Karşılaşılan Hatalar

| Belirti | Olası Sebep | Çözüm |
|---|---|---|
| **502 Bad Gateway** | Arka uç kapalı, yanlış port, `proxy_pass` hedefi çözümleyemiyor | `error.log`'a bakın; `curl -I http://127.0.0.1:<port>` ile arka ucu doğrudan test edin |
| **504 Gateway Time-out** | Arka uç geç cevap veriyor | `proxy_read_timeout` / `proxy_send_timeout` artırın, uygulamanın yavaş kaldığı yeri bulun |
| **413** | `client_max_body_size` küçük | Adım 7.2; uygulamanın kendi limitini de yükseltin |
| **Sonsuz yönlendirme döngüsü** | Arka uç `X-Forwarded-Proto` görmediği için kendini HTTP sanıyor | `proxy_set_header X-Forwarded-Proto $scheme;` ekleyin; uygulamanın "trust proxy" ayarını açın |
| **WebSocket kopması** | Upgrade header'ları yok / timeout kısa | Adım 7.1 |
| **`conflicting server name`** uyarısı | Aynı `server_name` iki dosyada tanımlı | `nginx -T` ile hangi dosya olduğunu bulun, birini silin |
| **`cannot load certificate ... No such file`** | Yanlış/eksik certbot satırı veya eksik symlink | `sudo certbot certificates`; `live/<domain>/fullchain.pem` yolunu kontrol edin |
| **`Address already in use`** | 80/443 başka serviste | `sudo ss -tlpn \| grep -E ':80\|:443'` |
| certbot **404** hatası | `/\.` kuralı `.well-known`'u da engelliyor | 8.2'deki `^~` istisnasını ekleyin |
| `nginx -t` OK ama değişiklik etkisiz | Servis reload edilmemiş / config symlink'i eksik | `sudo systemctl reload nginx`; `ls -l /etc/nginx/sites-enabled/` |

#### 9.3 Dışarıdan Tanılama

```bash
# Sertifikayı sunucu adıyla doğrula (tarayıcı önbelleğine girmeden)
curl -kvI --resolve panel.alanadiniz.com:443:127.0.0.1 https://panel.alanadiniz.com
echo | openssl s_client -connect panel.alanadiniz.com:443 -servername panel.alanadiniz.com 2>/dev/null | openssl x509 -noout -dates -issuer
```

---

### Adım 10: Güncelleme, Yedekleme ve Geri Yükleme

#### 10.1 Nginx Güncellemesi

```bash
sudo apt update
sudo apt install --only-upgrade -y nginx
sudo nginx -t && sudo systemctl reload nginx
```

> ⚠️ Debian güncellemesi sırasında `nginx.conf` için **"keep the local version currently installed"** sorulursa **mevcut sürümü seçin**; aksi hâlde kişisel ayarlarınız `.dpkg-conf` dosyasına taşınır. Elinizde kalıntı olanlar `/etc/nginx/*.dpkg-old` ve `*.dpkg-dist` adıyla durur; `diff` ile karşılaştırıp kendiniz birleştirin.

#### 10.2 Yedekleme

```bash
# /backup gibi, yedeklenen ağacın DIŞINDA bir hedef kullanın
sudo mkdir -p /backup && sudo chmod 700 /backup

sudo tar -czPf /backup/nginx_$(hostname)_$(date +%F).tar.gz \
    /etc/nginx /etc/letsencrypt /etc/fail2ban/jail.local
sudo dpkg -l | grep -E 'nginx|certbot' | awk '{print $2, $3}' \
    | sudo tee /backup/nginx_paketleri_$(date +%F).txt
```

> 💡 `/etc/letsencrypt` **klasörünün tamamı** yedeklenmelidir: `live/` klasörü gerçek dosya değil symlink'tir ve asıl anahtarlar `archive/` altında durur; yenileme için gereken **hesap anahtarı** da `/etc/letsencrypt/accounts` içindedir. `tar -P` symlink'leri çözümlenmiş kopyaya dönüştürmez, yapı olduğu gibi korunur. Private key içeren arşivi şifreleyin ve sunucu dışında saklayın (`chmod 700` yalnız yerel izin düzeltmesidir, şifreleme değildir).

#### 10.3 Geri Yükleme (Yeni Sunucuya Taşıma)

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
sudo rm -f /etc/nginx/sites-enabled/default
sudo tar -xzPf /backup/nginx_<hostname>_<tarih>.tar.gz -C /

sudo nginx -t                    # ÖNCE doğrula
sudo systemctl reload nginx
sudo certbot certificates
sudo certbot renew --dry-run     # Yenileme yolu hâlâ çalışıyor mu?
```

> ⚠️ Taşıdığınız alan adının DNS'i yeni sunucuya yönlenmeden Let's Encrypt HTTP-01 doğrulaması **eski sunucuya** gider. Sertifika kopyalanmış olsa bile yenileme başarısız olur; önce DNS A kaydını güncelleyip `dig +short panel.alanadiniz.com` ile teyit edin.

---

### Adım 11: L4 (TCP/UDP) Yönlendirme — `stream` Bloğu

`http {}` bloğu HTTP konuşan her şeyi anlar; SSH (22), veritabanı (5432/3306), MQTT (8883), NTP/DNS (UDP) gibi protokolleri **anlayamaz** — bunlar için L4 katmanında çalışan `stream` bloğu gerekir. `stream` ayrıca TLS'i **sonlandırmadan** (passthrough) SNI alan adına göre dağıtım yapmanı sağlar.

#### 11.1 Modül denetimi (önce bunu yap)

Debian paketleri `stream` modülünü **dinamik modül** olarak derleyebilir; bu durumda modülü elle yüklemen gerekir.

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep -E 'with-stream'
```

| Çıktı | Anlamı | Ne yapacaksın |
|---|---|---|
| `--with-stream` | Dahili (static) | Doğrudan 11.2'ye geç |
| `--with-stream=dynamic` | Dinamik modül | Ana context'e `load_module modules/ngx_stream_module.so;` ekle (Debian bu satırı `/etc/nginx/modules-enabled/*.conf` üzerinden otomatik getiriyor mu kontrol et) |
| Hiçbir şey | Varyantın modülü içermiyor | `sudo apt install nginx-full` ile varyant değiştirme kararı ver (config'in korunur, servis restart olur) |

SNI okuma ve PROXY protocol için ayrı modüller gerekir:

```bash
nginx -V 2>&1 | grep -oE 'stream_ssl_preread_module|stream_realip_module|http_realip_module' | sort -u
```

#### 11.2 Yerleşim kuralı: `stream` ANA context'te olmalı

> ⚠️ **En sık yapılan hata:** Debian'ın `nginx.conf` dosyasındaki `include /etc/nginx/conf.d/*.conf;` satırı `http { ... }` **içindedir**. `stream {}` bloğunu `conf.d/` altına koyarsan nginx `"stream" directive is not allowed here` veya `unknown directive "stream"` hatası verir. Aynı şekilde `sites-available/` dosyaları da http context'ine dahil edilir; `stream` oraya da giremez.

`stream` için ana context'te bir include satırı kendin ekle:

```bash
sudo mkdir -p /etc/nginx/stream.d
sudo nano /etc/nginx/nginx.conf
```

`nginx.conf` içinde hedef yapı (http bloğunun **dışında**, `events {}` ile `http {}` arasında):

```nginx
user www-data;
worker_processes auto;

include /etc/nginx/stream.d/*.conf;   # ← ana context; http bloğunun DIŞINDA olmalı

events { worker_connections 1024; }

http {
    # ...
    include /etc/nginx/conf.d/*.conf;   # ← bu satır http İÇİNDE; stream dosyalarını buraya verme
    include /etc/nginx/sites-enabled/*;
}
```

#### 11.3 TCP örneği (SSH / veritabanı / MQTT)

```nginx
# /etc/nginx/stream.d/tcp.conf
upstream ssh_filo {
    server 10.0.0.11:22 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:22 max_fails=3 fail_timeout=30s;
}

server {
    listen 2222;
    listen [::]:2222;

    proxy_connect_timeout 3s;
    proxy_timeout 15m;        # boşta kalan bağlantının kapanma süresi
    proxy_pass ssh_filo;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
ss -lntp | grep 2222                       # Beklenen: nginx by "ssh_filo" dinliyor
nc -vz 127.0.0.1 2222                      # Beklenen: connection succeeded
ssh -p 2222 kullanici@SUNUCU_IP            # Beklenen: arka uç makinenin parmakizi sorulur
```

> ⚠️ `stream` tarafında **aktif health check yoktur**: nginx yalnızca denediği bağlantı başarısız olursa sunucuyu `fail_timeout` boyunca işaretler. "Servis ayakta ama uygulaması bozuk" durumunu yakalamazsın; buna ihtiyacın varsa HAProxy (`mode tcp` + `option tcp-check`) daha doğru araçtır.

> 💡 `proxy_timeout` oturum uzunluğundan büyük olmalı; SSH ara vermeden 15 dakikadan uzun oturumlar kapanır, MQTT gibi heartbeat'li protokollerde heartbeat aralığını geçecek değeri ver.

#### 11.4 UDP örneği (DNS / NTP)

```nginx
# /etc/nginx/stream.d/udp.conf
server {
    listen 5353 udp reuseport;
    proxy_timeout 20s;
    proxy_pass 10.0.0.53:53;
}
```

> ⚠️ `reuseport`, çok çekirdekli sistemde her worker'ın kendi socket'ini açmasını sağlar; yüksek UDP hacminde paket düşmesini azaltır. DNS yönlendirirken UDP yanıtının **aynı worker'a dönmesi** kritik olduğundan `reuseport`'u kaldırma.

#### 11.5 TLS sonlandırmadan SNI ile dağıtım (passthrough)

```nginx
# /etc/nginx/stream.d/sni.conf
map $ssl_preread_server_name $l4_backend {
    hostnames;                       # .ornek.com biçimli maskeler izin verilsin
    node1.ankara.local    node1;
    rpc.ankara.local      rpc_pool;
    default               default_pool;
}

upstream node1     { server 10.0.0.21:8443; }
upstream rpc_pool  { server 10.0.0.31:993;  server 10.0.0.32:993; }
upstream default_pool { server 10.0.0.9:443; }

server {
    listen 443;
    proxy_connect_timeout 3s;
    proxy_timeout 10m;
    proxy_pass $l4_backend;
    ssl_preread on;                  # ClientHello'daki SNI'ı okur, şifreyi çözmez
}
```

> ⚠️ **Passthrough'un sınırları:** nginx içeriği görmediği için `location`, header ekleme, `proxy_cache`, WAF mantığı gibi L7 işleri bu hattaki trafiğe uygulayamazsın. Sertifika backend'lerde kalır ve yenilemesini backend'ler yapar.

> ⚠️ **ECH (Encrypted Client Hello) SNI'ı gizlerse** `ssl_preread` boş kalır ve tüm bağlantılar `default` havuzuna düşer. Tarayıcı tarafında ECH aktifse bu modeli kullanamazsın; TLS'i sonlandirip L7'den yönlendir.

> ⚠️ `proxy_pass` içinde **değişken kullandığında** nginx bunu çalışma zamanında çözümlemek ister; havuz adları `upstream` olarak tanımlıysa sorun yoktur, ancak hostname verdiysen `stream` bloğunda `resolver 10.0.0.53 valid=30s;` tanımlamak zorunludur.

#### 11.6 Gerçek istemci IP'si (PROXY protocol)

```nginx
# Öndeki yük dengeleyici/CDN send-proxy ile gönderiyorsa:
server {
    listen 8443 proxy_protocol;
    proxy_protocol_timeout 3s;     # kötüye kullanımda soket beklemesin
    proxy_pass backend_havuzu;
}

# Backend'e PROXY protocol ile devretmek istiyorsan:
server {
    listen 443;
    proxy_pass backend_havuzu;
    proxy_protocol on;
}
```

```bash
ss -tnp | grep 8443                       # Beklenen: $remote_addr artık gerçek istemci IP'si
nginx -T 2>/dev/null | grep -c proxy_protocol   # Kaç yerde tanımlı?
```

> ⚠️ `listen ... proxy_protocol` beklerken istemci düz HTTP/TLS gönderirse bağlantı **çöp veriyle açılır ve hata verir**. Zincirdeki her halkanın ya hepsi bunu göndermesi ya da hiçbirmemesi gerekir; yarıda bırakılan yapı tipik olarak "yer yer 400 Bad Request" olarak görünür.

> 💡 L7 tarafında aynı işi `http_realip_module` ile yaparsın (Adım 8.5). `stream` tarafında istemci IP'yi logda görmek istiyorsan `stream_realip_module` gerekir (11.1'de kontrol etmiştik).

#### 11.7 Log ve doğrulama

```nginx
# stream{} içinde:
log_format l4 '$remote_addr [$time_local] $protocol "$upstream_addr" '
              'sent=$upstream_bytes_sent recv=$upstream_bytes_received sess=$session_time';
access_log /var/log/nginx/stream.log l4;
```

`$session_time` saniye cinsindendir; `error`/`timeout` durumlarını yakalamak için `upstream_addr` boşsa nginx bağlantı kuramamıştır.

> ⚠️ **Güvenlik duvarı:** L4 yönlendirdiğin her port (2222, 8443, 5353/udp) tabii olarak dışarı açılır. Adım 2.4'teki port tablosunu genişletirken **sadece açman gereken portu** aç; öntanımlı olarak yönetim portlarını kaynak IP ile kısıtla.

#### 11.8 Bu bölümde sık hatalar

| Belirti | Sebep | Çözüm |
|---|---|---|
| `"stream" directive is not allowed here` | `stream {}` http bloğunun içine konmuş | Ana context'e taşı (11.2) |
| `unknown directive "stream"` | Modül derlemede yok veya load_module eksik | 11.1 denetimi; dinamik ise `load_module` ekle |
| `unknown directive "ssl_preread"` | `stream_ssl_preread_module` yok | Modülsüz kurulumda SNI routing yapılamaz |
| Her bağlantı `default` havuzuna düşüyor | SNI okunamadı (ECH, eski TLS, client SNI göndermiyor) | `log_format` içine `$ssl_preread_server_name` ekleyip gözle |
| Bağlantılar düzensiz kapanıyor | `proxy_timeout` kısa | Oturum süresine göre yükselt |
| Logda client IP `127.0.0.1` | Arada proxy var, PROXY protocol yok | 11.6 veya realip |

---

### Adım 12: Reverse Cache — `proxy_cache`

`proxy_cache`, nginx'i arka ucun **önüne koyulan bir HTTP cache'i** yapar: aynı içeriği her istekte origin'e sormaz. Statik/varlığı ağır servislerde (JS/CSS sürümlü dosyalar, rapor çıktıları, herkese açık JSON) origin yükünü ciddi biçimde düşürür.

#### 12.1 Ne zaman kur, ne zaman kurma

- **Kur:** herkese açık, cache başlığı doğru (`Cache-Control: public/max-age`) veya hiç başlık göndermeyen statik/varlığı ağır uçlar; çok ziyaret edilen sayfalar.
- **Kurma:** oturum bazlı paneller, API'ler (`Authorization` ile gelen yanıtlar), Nextcloud WebDAV (`PROPFIND`), rapor/CSV indirme gibi tek seferlik büyük dosyalar. `Set-Cookie` içeren yanıtlar zaten cache'lenmez (12.4) ama riski peşinen almak daha doğru.

#### 12.2 Cache zone tanımla (http context) ve dizini hazırla

```bash
sudo mkdir -p /var/cache/nginx/panel
sudo chown -R www-data:www-data /var/cache/nginx
```

```nginx
# /etc/nginx/conf.d/cache.conf  (http context içinde; proxy_cache_path YALNIZ http'te tanımlanır)
proxy_cache_path /var/cache/nginx/panel
                 levels=1:2
                 keys_zone=panel_cache:20m
                 max_size=2g
                 inactive=60m
                 use_temp_path=off;
```

| Parametre | Anlamı | Not |
|---|---|---|
| `levels=1:2` | Diskte klasör derinliği | Tek dizinde yüz binlerce dosyaya izin verme |
| `keys_zone=ad:boyut` | Bellekteki index boyutu | Kaba ölçü: ~1 MB ≈ 8.000 anahtar |
| `max_size` | Disk tavanı | Aşınca `inactive`e göre eskiyi süpürür |
| `inactive` | Ne kadar süre hiç erişilmezse silinir | `Cache-Control`'dan bağımsız temizlik zamanlayıcısı |
| `use_temp_path=off` | Doğrudan cache'e yaz | Kopyalama maliyetini kaldırır (nginx 1.7.11+) |

#### 12.3 Location içinde kullanımı (tam örnek)

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    # Adım 4'teki proxy_set_header dörtlüsünü buraya kopyala:
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_cache panel_cache;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_valid 200 301 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_methods GET HEAD;             # varsayılan zaten GET HEAD
    proxy_cache_convert_head on;              # HEAD'i GET yanıtıyla besle (1.7.7+)

    # Yıldırım (stampede) koruması: MISS sırasında tek istek origin'e gitsin
    proxy_cache_lock on;
    proxy_cache_lock_timeout 5s;
    proxy_cache_background_update on;

    # Süresi dolmuş içeriği, origin düşerken/yanıt yenilenirken servis et
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_revalidate on;                # If-Modified-Since ile tazeleme

    add_header X-Cache-Status $upstream_cache_status always;
}
```

#### 12.4 nginx neyi cache'lemez (resmî davranış)

- Yanıtta **`Set-Cookie`** varsa → **cache'lenmez**.
- `Cache-Control` içinde **`no-store`**, **`no-cache`** veya **`private`** varsa → cache'lenmez; **`max-age`/`Expires`** değerleri TTL olarak kullanılır.
- `Vary: *` olan yanıt cache'lenmez; başka bir `Vary` değeri varsa ilgili istek başlığı anahtara eklenerek cache'lenir.
- Origin `X-Accel-Expires: <sn>` gönderirse cache süresini **origin** belirler (`0` = cache dışı). Bu başlık, proxy'siz cache kontrolünün en temiz yoludur.

> ⚠️ **Tehlikeli ayar:** `proxy_ignore_headers Set-Cookie Cache-Control Expires;` yazarsan yukarıdaki korumaların hepsi kapanır — **oturum bazlı, kişiye özel sayfalar herkese servis edilebilir** (kimlik/PII sızıntısı). Bu seçenek yalnızca origin'in cache başlıklarını yanlış gönderdiği, içeriğin gerçekten herkese açık olduğu uçlarda ve **tek bir location'a** uygulanır.

#### 12.5 Oturum/cookie bazlı bypass

```nginx
# http context (conf.d içine):
map $http_cookie $skip_cache {
    default                       "";
    "~(wordpress_logged_in|sessionid|csrf|JSESSIONID)"  "1";
}

# location içinde:
proxy_cache_bypass $http_authorization $http_pragma $skip_cache;
proxy_no_cache     $http_authorization $skip_cache;
```

> 💡 `proxy_cache_bypass` cache'i **okumaz** (origin'e gider, yazabilir); `proxy_no_cache` **yazmaz**. Oturumlu kullanıcıyı korumak için ikisini birlikte kullan.

#### 12.6 Hit/miss doğrulama (beklenen çıktıyla)

```bash
for i in 1 2; do curl -sI https://panel.alanadiniz.com | grep -i x-cache-status; done
```

```text
X-Cache-Status: MISS      # ilk istek: origin'e gitti
X-Cache-Status: HIT       # ikinci istek: cache'ten servis edildi
```

Diğer durumlar: `BYPASS` (kural gereği atlandı), `EXPIRED` (süresi dolmuş, yenilendi), `STALE` (origin düşmüş, eski servis edildi), `UPDATING` (bayat servis edilirken arka planda tazelendi), `REVALIDATED` (304 ile tazelendi).

```bash
sudo du -sh /var/cache/nginx/panel
sudo grep -E 'cache|worker process' /var/log/nginx/error.log | tail
```

#### 12.7 Cache'i temizleme (purge)

> ⚠️ **Doğrulanması gereken nokta:** `proxy_cache_purge` direktifi bazı nginx derlemelerinde bulunur, bazılarında üçüncü parti `ngx_cache_purge` modülü gerekir. Debian paketinde tanınıp tanınmadığını **servisi bozmadan** böyle test et:
>
> ```bash
> echo 'proxy_cache_purge 1;' | sudo tee /etc/nginx/conf.d/99-purge-test.conf
> sudo nginx -t; sudo rm /etc/nginx/conf.d/99-purge-test.conf
> ```
> `nginx -t` `"proxy_cache_purge" directive is unknown` veriyorsa çekirdekte yok; ya `--add-module=third_party/ngx_cache_purge` ile derle ya da aşağıdaki modülsüz yöntemi kullan.

**Modülsüz, güvenli yöntem — başlık ile zorla tazeleme:**

```nginx
location / {
    # X-Force-Refresh gelen istekte cache'i atla ve yeniden doldur
    proxy_cache_bypass $http_x_force_refresh;
    proxy_no_cache     $http_x_force_refresh;
}
```

```bash
curl -sI -H 'X-Force-Refresh: 1' https://panel.alanadiniz.com | grep -i x-cache-status
# Beklenen: MISS
```

> 🔒 Bu başlığı **rastgele kişilere açma**: `map` ya da `geo` ile yalnızca yönetim ağına bağla (Adım 8.3'teki allow/deny deseni gibi); aksi halde herkes origin'i bombalatan bir tıklama hakkına sahip olur.

#### 12.8 Disk, bellek ve arıza davranışı

- Disk dolarsa nginx cache'e **yazamaz**; hata loglanır ve istekler cache'siz servis edilmeye devam eder — yani cache arızası siteyi düşürmez ama origin'i yorar. Bunu `error.log`daki `alert: failed to create temporary path` / `no free space` satırlarından takip et.
- `max_size` aşılınca temizlik **manager** süreci tarafından yapılır; çok büyük `inactive` değeri disk kullanımını şişirir.
- `min_free` ile diskin boş kalması gereken miktarını belirtebilirsin (uygulama diski ile aynı bölümdeyse faydalıdır).
- Cache dizinine `www-data` yazabiliyor olmalı; `/var/cache` ayrı bir bölümse mount seçeneği `noexec,nosuid` olsun.

> 💡 `add_header X-Cache-Status ...` satırını `location` içine koyduysan, o location'da üst seviyedeki **diğer `add_header` satırları uygulanmaz** (bkz. sondaki İpucu #6). Bu iki satırı aynı yere yazmak en az süprizli davranıştır.

---

## 📊 Düz Nginx mi, Nginx Proxy Manager mi?

| Ölçüt | Düz nginx (bu rehber) | Nginx Proxy Manager |
|---|---|---|
| Kurulum | `apt install nginx` + dosya yazma | Docker Compose + web paneli |
| Yeni domain ekleme | Config dosyası + `reload` | 3 tıklama |
| TLS bilgisi | Gerekli (bu rehber karşılıyor) | Gerekli değil |
| Kaynak tüketimi | ~10-30 MB RAM | ~200-400 MB RAM (Node + veritabanı + nginx) |
| Kontrol / hata ayıklama | Tam; tek gerçek kaynak dosyalar | Template arkasında; `nginx -T` benzeri görüş yok |
| Saldırı yüzeyi | Sadece nginx | nginx + yönetim arayüzü (:81) + veritabanı |
| Çok kullanıcı / rol / audit log / MFA | Yok | Var |
| TCP/UDP (non-HTTP) yönlendirme | `stream {}` bloğu (Adım 11) | "Streams" ekranı |
| Veri yükseltme riski | Config'i siz yönetirsiniz | Otomatik veritabanı migrate'si, geri dönüşü zor |

**Özet kural:** Az sayıda sabit servis, minimum bağımlılık ve tam kontrol istiyorsanız düz nginx; sık değişen çok sayıda domain, ekip kullanımı ve arayüzden yönetim istiyorsanız Nginx Proxy Manager. İkisi **aynı makinede 80/443 için yarışamaz** — birini dış kapı olarak seçin.

---

## 💡 Dikkat Edilmesi Gereken İpuçları

1. **Her değişiklikten önce `sudo nginx -t`.** `reload` edilmiş bozuk config, tüm siteleri düşürür; `nginx -t` hatayı çalışır servisi etkilemeden gösterir.
2. **`systemctl restart` değil `systemctl reload`.** Reload mevcut bağlantıları kesmez; restart tüm worker'ları düşürür.
3. **`default_server` yalnızca bir kez tanımlanmalı.** Yanlış sertifikalı bir sitenin "yedek" olarak öne çıkmasını engeller.
4. **`127.0.0.1` burada güvenli.** Nginx host'ta çalıştığı için konteyner tabanlı araçlardaki kendi içine dönme tuzağı yoktur.
5. **`/\.` kuralı ile `.well-known` çakışması** en çok certbot yenilemesini bozan tuzaktır; istisnayı mutlaka `^~` ile önceleyin.
6. **`add_header` mirası:** Bir `location` içinde tek bir `add_header` bile yazarsanız, üst seviyedeki `add_header` satırlarının hiçbiri o location'a uygulanmaz.
7. **DNS + port teyidi:** Sertifika hatalarının büyük kısmı config değil, A kaydının yanlış olması veya 80/443'ün ISP/docker tarafından kapalı olmasıdır.
8. **Yönlendirme döngüsü = genelde `X-Forwarded-Proto`.** Hem nginx tarafında header'ı gönderin hem uygulamanın proxy trust ayarını açın.
9. **L4 ve cache artık bu rehberin kapsamında:** TCP/UDP yönlendirme Adım 11'de (`stream` bloğu), reverse cache Adım 12'de (`proxy_cache`) anlatılıyor. `stream` modülü Debian paket varyantına göre dinamik olabilir; başladıktan önce modül denetimi Adım 11.1'de.

> 📌 **Sürüm notu:** Bu rehber Debian 13 (trixie) ve **nginx 1.26.x** esas alınarak hazırlanmıştır. `http2 on;` gibi söz dizimleri nginx 1.25.1+ içindir; daha eski sürümlerde (ör. Debian 12 / nginx 1.22) `listen 443 ssl http2;` yazımı kullanılır. Bir direktif `nginx -t`'de tanınmıyorsa sebebi sürüm farkıdır.

*Tebrikler! Debian 13 üzerinde nginx ile reverse proxy kurulumunuz ve sertleştirmeniz tamamlanmıştır.*

---

- Author: faruk-guler
- GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
- Page: [www.farukguler.com](https://www.farukguler.com)
