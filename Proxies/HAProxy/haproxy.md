# HAProxy on Debian 13 (Trixie) Load Balancer & Reverse Proxy

**HAProxy** (High Availability Proxy), yüksek performanslı, açık kaynaklı bir **yük dengeleyici (load balancer)** ve **ters proxy (Reverse Proxy)** çözümüdür. Gelen trafiği birden fazla backend sunucuya dağıtarak yüksek erişilebilirlik (high availability), ölçeklenebilirlik ve hata toleransı sağlar. Özellikle web sunucuları, API gateway'leri ve veritabanı kümeleri önünde kurumsal ortamlarda yaygın olarak kullanılır.

Bu rehber, **Debian 13 (Trixie)** üzerinde HAProxy'nin paket yöneticisi ile kurulumunu, HTTP/HTTPS yük dengeleme yapılandırmasını, SSL sonlandırmasını (SSL termination), sağlık kontrollerini (health checks), L4/TLS passthrough ve çok domainli yönlendirmeyi, oran sınırlamayı ve güvenlik pratiklerini kapsamaktadır.

> 📌 **Sürüm:** Debian 13 deposunda **haproxy 3.0.x (LTS)** bulunur (`haproxy -v` ile teyit et). Bu rehber 3.0 sözdizimine göredir; 2.4+ sürümlerinde yinelenen bölüm adları **hata** olarak değerlendirilir — aşağıdaki adımlardaki bloklar ayrı dosyalar değil, **tek `haproxy.cfg` içinde birleştirilecek parçalardır**.

---

## 📘 Adım Adım Manuel Kurulum

### Adım 1: Sistem Güncellemeleri ve Paket Kurulumu

Sistem paketlerinizi güncelleyin ve HAProxy'yi yükleyin:

```bash
# 1.1 Sistem güncellemesi
sudo apt update && sudo apt upgrade -y

# 1.2 HAProxy'yi yükleyin
sudo apt install -y haproxy nano

# 1.3 Sürümü ve derleme özelliklerini doğrulayın (-vv: OpenSSL sürümü, kullanılabilir seçenekler)
haproxy -vv

# 1.4 Servisi başlatın ve otomatik açılışa ekleyin
sudo systemctl enable --now haproxy
sudo systemctl status haproxy --no-pager
```

> 💡 `haproxy -vv` çıktısında **`Built with OpenSSL`** satırını gör; SSL termination ve `crt` seçeneği buna bağlıdır. Ayrıca `mode tcp` ve yerleşik **Prometheus exporter** (2.4+) bu derlemede hazır gelir.

> ⚠️ `full-upgrade` yerine `upgrade`: `full-upgrade` bağımlılık değişimlerinde paket kaldırabilir; DNS/load balancer gibi kritik bir rolde sürpriz yaşamamak için çıkan listeyi `-s` ile elle incelemeden yükseltme yapma.

---

### Adım 2: Temel Yapılandırma (`haproxy.cfg`)

HAProxy'nin tüm konfigürasyonu `/etc/haproxy/haproxy.cfg` dosyasında tutulur. Değişiklik yapmadan önce yedek alın:

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo nano /etc/haproxy/haproxy.cfg
```

Aşağıdaki temel yapıyı referans alın. Dosyanın mevcut içeriğini tamamen bu yapıyla değiştirin:

```text
#---------------------------------------------------------------------
# Global Ayarlar
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    maxconn 4096                # Eşzamanlı bağlantı tavanı (bkz. aşağıdaki LimitNOFILE notu)
    hard-stop-after 30m         # reload sonrası eski işlemlere en fazla 30 dk tanınır
    tune.bufsize 32768          # büyük cookie/JWT'li istekler için (bkz. İpuçları #7)

    # TLS/SSL Güvenlik Sertleştirmeleri (TLS 1.2 + TLS 1.3)
    ssl-default-bind-ciphers ECDHE+AESGCM:ECDHE+AES256:ECDHE+AES128:!aNULL:!MD5:!DSS
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options prefer-client-ciphers no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets

#---------------------------------------------------------------------
# Varsayılan Ayarlar
#---------------------------------------------------------------------
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  forwardfor
    option  http-server-close
    retries 3
    timeout connect          5s
    timeout client           1m
    timeout server           1m
    timeout tunnel           5m    # WebSocket/CONNECT için şart — aşağıdaki nota bak
    timeout http-request     10s
    timeout http-keep-alive  10s
    timeout check            10s
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http
```

> ⚠️ **`timeout tunnel` olmadan WebSocket çalışmaz:** Bu seçenek tanımlanmadığında HAProxy, client/server timeout'larından uzun olanını kullanır — yani sessiz bir 60 sn limiti vardır ve canlı terminal/graf/bildirim bağlantıları ~1 dakikada düşer. Uzun ömürlü bağlantılı servisler için `5m` başlangıç için yeterli değilse `1h` yap.

> 💡 **`maxconn` ve dosya sınırı:** `maxconn` değerini açtığında işçi süreçlerin **file-descriptor** limiti de yeterli olmalı (her bağlantı birden fazla fd tüketir). Debian'ın systemd unit'inde `LimitNOFILE` kontrol et ve gerekirse override ile yükselt: `systemctl show -p LimitNOFILE haproxy` → `systemctl edit haproxy` içine `[Service]\nLimitNOFILE=65536`.

---

### Adım 3: HTTP Yük Dengeleme (Load Balancing)

Aşağıdaki örnek senaryo, gelen HTTP trafiğini `192.168.1.10`, `192.168.1.11` ve `192.168.1.12` adreslerindeki üç web sunucusuna eşit olarak dağıtır.

> ⚠️ **Bölüm adları config genelinde benzersiz olmalıdır ve bu adımdaki bloklar sonraki adımlarla AYNI adı taşır.** Kopyala-yapıştır yaparken "ikisini de ekle" mantığı `already declared` hatası verir. Doğru okuma biçimi:
>
> - Adım 4'teki `frontend http_frontend` bloğu, **Adım 3'tekinin yerine geçer** (üzerine yazılır; `bind *:80` iki kez kalmaz).
> - Adım 5'teki `backend web_servers` bloğu, **Adım 3'tekinin yerine geçer** (health check parametreleriyle birlikte gelir).
> - Seçenek A/B'den yalnız biri seçilir; `backend web_servers` adı korunur.
>
> Adım 10 ve 11 ise **yeni** adlar kullanır (`fe_main`, `be_panel` vb.) ve Adım 3-5 bloklarıyla çakışmaz. Hangi bloğun hangisinin yerine geçtiğini takip etmek istemiyorsan sondaki **Nihai Örnek** tek parça hâlindedir; ondan şaşma. (HAProxy 2.4+ yinelenen bölüm adlarında config'i reddeder.)

#### Seçenek A: `roundrobin` (Sıralı Dağıtım) — Varsayılan

Her istek sırayla bir sonraki sunucuya yönlendirilir. Sunucular eşit güçteyse bu yöntem idealdir:

```text
#---------------------------------------------------------------------
# Frontend: Gelen İstekleri Dinle
#---------------------------------------------------------------------
frontend http_frontend
    bind *:80
    default_backend web_servers

#---------------------------------------------------------------------
# Backend: Yük Dağıtılacak Sunucular
#---------------------------------------------------------------------
backend web_servers
    balance roundrobin
    option httpchk GET /health        # eski/kısa yazım; modern biçim Adım 5'teki `option httpchk` + `http-check send` ikilisi
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
    server web3 192.168.1.12:80 check
```

#### Seçenek B: `leastconn` (En Az Bağlantı) — Uzun Süreli Oturumlar İçin

En az aktif bağlantısı olan sunucuya yönlendirir. Veritabanı proxy'leri veya WebSocket bağlantıları için uygundur:

```text
backend web_servers
    balance leastconn
    option httpchk GET /health
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
    server web3 192.168.1.12:80 check
```

Yapılandırmayı doğrulayın ve servisi yeniden yükleyin:

```bash
# Sözdizimi kontrolü (servisi durdurmadan)
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Beklenen çıktı:

```text
[NOTICE]   (999) : haproxy version is 3.0.11
[WARNING]  : config : missing timeouts for proxy 'http_frontend'   ← uyarı olabilir ama hata değil
Configuration file is valid
```

`Configuration file is valid` satırını gördüysen servise dokunma zamanı geldi. Hatalıysa çıktı `Unknown keyword` / `'x' already declared` / `refers to an undefined backend` gibi bir satırla biter ve **servis çalışmaya devam eder** — bu komut reload yapmaz.

```bash
# Aktif bağlantıları kesmeden yeniden yükleme
sudo systemctl reload haproxy

# Doğrulama: servis hâlâ ayakta ve portlar dinleniyor mu
systemctl is-active haproxy
ss -lntp | grep -E ':(80|443) '
```

---

### Adım 4: HTTPS ve SSL Sonlandırma (SSL Termination)

HAProxy, gelen HTTPS trafiğini kendi üzerinde çözümleyip (decrypt) backend sunuculara düz HTTP olarak iletebilir. Bu sayede backend sunucularınızı SSL yükünden kurtarırsınız.

#### 4.1 SSL Sertifikasını Hazırlama (Kurumsal / Yerel CA)

HAProxy, sunucu sertifikasını, varsa ara (intermediate) CA sertifikasını ve özel anahtarı (private key) **tek bir birleşik `.pem` dosyasında** bekler.

Kurumunuzun sağladığı sertifika dosyalarını (`domain.crt`, `intermediate.crt`, `domain.key`) birleştirin:

```bash
# Sertifika dizini oluşturun
sudo mkdir -p /etc/haproxy/certs

# 1. Sertifika, Ara CA ve Özel Anahtarı sırasıyla birleştirin
cat domain.crt intermediate.crt domain.key | sudo tee /etc/haproxy/certs/kurumsal.pem > /dev/null

# 2. Dosya izinlerini kısıtlayın (güvenlik için sadece root okuyabilsin)
sudo chmod 600 /etc/haproxy/certs/kurumsal.pem

# 3. (İsteğe Bağlı) Yerel CA Kök Sertifikasını Debian Sistemine Tanıtma
# HAProxy veya backend servislerinizin yerel CA'yı tanıması için:
sudo cp local-ca.crt /usr/local/share/ca-certificates/local-ca.crt
sudo update-ca-certificates
```

> ⚠️ **Private key şifreli olmamalı.** `domain.key` passphrase ile korunuyorsa HAProxy açılışta parolayı sorar ve systemd altında bu istek karşılanamaz → servis başarısız olur. Anahtarı şifresiz hâle getir (`openssl rsa -in encrypted.key -out plain.key`) veya parolasız üret.

> 💡 **Çok domain (SNI):** `crt` seçeneğine dosya yerine **dizin** verirsen HAProxy her `.pem`'i SNI'a göre kendisi seçer:
>
> ```text
> bind *:443 ssl crt /etc/haproxy/certs/ alpn h2,http/1.1 strict-sni
> ```
>
> Dosya adlarını `<domain>.pem` biçiminde tut; `strict-sni` ile tanımsız SNI'lı el sıkışmalar reddedilir (bu satırı koymadan önce tüm domainlerinin DNS'inin doğru olduğundan emin ol).

**Birleştirmeden önce doğrula** — HAProxy hatalı/eksik zinciri çoğu zaman kabul eder ve hatayı ilk gören istemci olur:

```bash
# 1) Anahtar ↔ sertifika eşleşmesi: iki md5 AYNI çıkmalı
openssl x509 -noout -modulus -in domain.crt | openssl md5
openssl rsa  -noout -modulus -in domain.key | openssl md5

# 2) Kapsanan adlar ve tarihler (CN tek başına yetmez, SAN listesi şart)
openssl x509 -noout -subject -issuer -dates -ext subjectAltName -in domain.crt

# 3) Zincir bütünlüğü: ara CA'yı -untrusted ile göster
openssl verify -CAfile kok-ca.crt -untrusted intermediate.crt domain.crt
# Beklenen: domain.crt: OK          ("unable to get issuer certificate" → ara CA eksik/yanlış)

# 4) Birleşik dosyanın iç yapısı
grep -c 'BEGIN CERTIFICATE' /etc/haproxy/certs/kurumsal.pem   # Beklenen: 2 (sunucu + ara CA)
grep -c 'PRIVATE KEY'      /etc/haproxy/certs/kurumsal.pem   # Beklenen: 1
```

> ⚠️ **Satır sonu tuzağı:** Dosyalardan birinin sonunda boş satır yoksa `-----END CERTIFICATE-----` ile `-----BEGIN PRIVATE KEY-----` aynı satıra düşebilir ve PEM bozulur. Belirtisi: `haproxy -c` çıktısında `cannot load certificate ... PEM routines`. Güvenli birleştirme: `printf '%s\n' "$(cat domain.crt)" "$(cat intermediate.crt)" "$(cat domain.key)" | sudo tee /etc/haproxy/certs/kurumsal.pem > /dev/null` ya da her dosyanın sonuna boş satır ekle.

```bash
# Kurum PKCS#12 (.pfx) verdiyse PEM'e çevir — HAProxy PEM bekler
openssl pkcs12 -in Sertifika.pfx -out domain.crt      -nokeys  -clcerts -passin pass:SIFRE
openssl pkcs12 -in Sertifika.pfx -out domain.key      -nocerts -nodes   -passin pass:SIFRE
openssl pkcs12 -in Sertifika.pfx -out intermediate.crt -nokeys -cacerts -passin pass:SIFRE
```

> 🔒 İç CA ile üretilmiş sertifika **kurum dışındaki tarayıcılarda uyarı verir**; sessiz çalışması için kök CA'nın GPO/MDM ile cihazlara dağıtılması gerekir. Vatandaşa/müşteriye açık servislerde public CA kullan. Sertifikanın bitiş tarihini `openssl x509 -enddate -noout -in /etc/haproxy/certs/kurumsal.pem` ile izle; **custom sertifika otomatik yenilenmez**, Adım 4.3'teki deploy hook yalnız Let's Encrypt kayıtları için çalışır.

#### 4.2 HTTPS Frontend Yapılandırması

```text
frontend https_frontend
    # alpn h2 olmadan HTTP/2 konuşulmaz — `ssl crt` tek başına yetmez
    bind *:443 ssl crt /etc/haproxy/certs/kurumsal.pem alpn h2,http/1.1

    # Security header'ları: 'always' olmadan HAProxy'nin ürettiği 4xx/5xx yanıtlarına eklenmez
    http-response set-header X-Content-Type-Options nosniff always
    http-response set-header X-Frame-Options SAMEORIGIN always
    # HSTS: preload'u bilinçli karar vermeden ekleme (bkz. aşağıdaki uyarı)
    http-response set-header Strict-Transport-Security "max-age=15768000; includeSubDomains" always

    # Backend uygulamalara gerçek protokol bilgisini ilet (Django, Laravel, WordPress için şart)
    http-request set-header X-Forwarded-Proto https

    default_backend web_servers

frontend http_frontend
    bind *:80
    # HTTP isteklerini HTTPS'e kalıcı olarak yönlendir
    http-request redirect scheme https code 301
```

> ⚠️ **HSTS ve `preload`:** `preload` doğrudur ama **geri alması çok zor** bir taahhüttür: alan adının ve **tüm** alt alan adlarının HTTPS'e hazır olması, hstspreload.org başvurusu ve listeden çıkarma sürecinin aylar sürmesi gerekir. Bu yüzden yukarıda `preload` bilinçli olarak yok. İlk kurulumda `max-age` değerini kısa tut (ör. 600), HTTPS akışı birkaç gün stabil çalıştıktan sonra yükselt.

#### 4.3 Let's Encrypt ile Otomatik Sertifika

HAProxy 2.6+ yerleşik bir ACME istemcisi taşır; ancak sözdizimi sürümler arasında değiştiği için bunu **Debian 13'teki 3.0 sürümünün resmî dokümanından doğrulayarak** kullan. Doğrulamak istemediğin durumda üç pratik yol var (hepsi aynı deploy hook ile biter):

> ⚠️ **Önce bu gerçeği bil:** HAProxy **statik dosya sunmaz.** Yani `certbot --webroot` tek başına işe yaramaz — challenge dosyasını birinin HTTP ile servis etmesi gerekir. Ya küçük bir statik backend kur (Yöntem C) ya da 80 portuna hiç ihtiyaç duymayan bir yönteme geç (Yöntem A).

**Yöntem A — DNS-01 (kesintisiz, wildcard desteği):**

```bash
sudo apt install -y certbot
# Sağlayıcına göre eklenti: python3-certbot-dns-cloudflare / -route53 / -desec vb.
sudo certbot certonly --dns-cloudflare \
     --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
     -d panel.alanadiniz.com -d '*.alanadiniz.com' \
     --deploy-hook "/etc/haproxy/deploy-hook.sh panel.alanadiniz.com"
```

**Yöntem B — Standalone + pre/post hook (kısa süreli kesinti):**

```bash
sudo certbot certonly --standalone \
     --pre-hook  "systemctl stop haproxy" \
     --post-hook "systemctl start haproxy" \
     -d panel.alanadiniz.com \
     --deploy-hook "/etc/haproxy/deploy-hook.sh panel.alanadiniz.com"
```

**Yöntem C — Webroot + HAProxy içinde statik backend:**

```text
# frontend http_frontend içinde, redirect'ten ÖNCE:
acl acme path_beg /.well-known/acme-challenge/
use_backend acme_static if acme

backend acme_static
    server local 127.0.0.1:8888
```

```bash
# 8888'de yalnızca localhost'tan dinleyen küçük bir statik sunucu (ör. nginx veya busybox httpd)
sudo mkdir -p /var/www/certbot
sudo certbot certonly --webroot -w /var/www/certbot -d panel.alanadiniz.com \
     --deploy-hook "/etc/haproxy/deploy-hook.sh panel.alanadiniz.com"
```

Hangi yöntemi seçersen seç, HAProxy tek birleşik `.pem` istediği için yenilemede dosyayı yeniden üreten bir deploy hook gerekir:

```bash
sudo tee /etc/haproxy/deploy-hook.sh > /dev/null <<'EOF'
#!/bin/bash
# Kullanım: certbot --deploy-hook "/etc/haproxy/deploy-hook.sh <domain>"
set -euo pipefail
DOM="${1:?domain parametresi gerekli}"
cat "/etc/letsencrypt/live/${DOM}/fullchain.pem" \
    "/etc/letsencrypt/live/${DOM}/privkey.pem" \
    > "/etc/haproxy/certs/${DOM}.pem"
chmod 600 "/etc/haproxy/certs/${DOM}.pem"
systemctl reload haproxy
EOF
sudo chmod 755 /etc/haproxy/deploy-hook.sh

# Hook dahil doğrulama
sudo certbot renew --dry-run
```

> ⚠️ **ACME challenge'ı redirect'e kurban etme (bu bölümün en sık yapılan hatası).** `http-request` kuralları backend seçiminden **önce** çalışır; yani `use_backend acme_static` satırını redirect'ten önce yazman **yetmez** — redirect yine de çalışır ve HTTP-01 dosyası `301` döner. Çözüm redirect'i muaf tutmaktır:
>
> ```text
> acl acme path_beg /.well-known/acme-challenge/
> http-request redirect scheme https code 301 unless acme
> use_backend acme_static if acme
> ```

> 💡 `certbot renew --dry-run`, `--deploy-hook`'u **çalıştırmaz** — hook'un gerçekten çalıştığını bir kez elle doğrula. `--force-renewal` limit tüketir (Let's Encrypt: registered domain başına 7 günde 50 sertifika); alışkanlık yapma.

---

### Adım 5: Sağlık Kontrolleri (Health Checks)

HAProxy, backend sunucuları düzenli aralıklarla kontrol eder. Erişilemeyen sunucuyu otomatik olarak devre dışı bırakır:

```text
backend web_servers
    balance roundrobin
    option httpchk
    http-check send meth GET uri /health ver HTTP/1.1 hdr Host localhost

    # Beklenti net olmalı: HAProxy varsayılan olarak 2xx VE 3xx'i "sağlıklı" sayar.
    # 302 dönen bir /health endpoint'i, uygulama bozuk olsa bile UP görünür.
    http-check expect status 200

    # inter: kontrol aralığı | rise: sağlıklı sayılmak için başarı sayısı | fall: devre dışı için başarısızlık sayısı
    # maxconn: backend'i koruyan eşzamanlı bağlantı tavanı
    default-server inter 3s rise 2 fall 3 maxconn 200
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
    server web3 192.168.1.12:80 check backup
```

> 💡 **`backup` parametresi:** `web3` yalnızca `web1` ve `web2` devre dışı kaldığında devreye girer. Yedek sunucu (failover) senaryoları için kullanın.

> ⚠️ **`default-server` satırı** tekrarlanan sunucu parametrelerini toplar; bunu kullanmak yerine her satıra `inter/rise/fall` yazmak da geçerlidir. `check` bayrağını **her satırda** bırakmayı unutma — `check` olmadan HAProxy o sunucuya körlemesine trafik gönderir.

> 💡 **Tüm sunucular düştüyse:** client'a `503` döner ve logda `no available server` görürsün. Durumu görmek ve bakım moduna almak için:
>
> ```bash
> echo "show stat" | sudo socat stdio /run/haproxy/admin.sock | awk -F, '/web_servers/ {print $1, $2, $18, $20}'
> # Beklenen: her satırda backend, sunucu adı, status (UP/DOWN/MAINT) ve aktif bağlantı sayısı
>
> echo "set server web_servers/web1 state maint" | sudo socat stdio /run/haproxy/admin.sock
> echo "set server web_servers/web1 state ready"  | sudo socat stdio /run/haproxy/admin.sock
> ```

> 💡 **TCP backend'lerde health check farklıdır:** `mode tcp` bloğunda `option httpchk` kullanılmaz. Yalnızca "port açık mı" testi için `check` yeterlidir (HAProxy TCP connect yapar). İçerik testi istersen `option tcp-check` + `tcp-check connect/send/expect` satırlarını kullan (bkz. Adım 9).

---

### Adım 6: İzleme — Stats Ekranı, Stats Socket, Prometheus ve Sertifika Yönetimi

#### 6.1 Stats Ekranı

HAProxy yerleşik bir izleme arayüzü sunar. `haproxy.cfg` dosyasına ekleyin (Prometheus satırı da **bu tek blokta** olmalı — ikinci bir `listen stats` tanımlama):

```text
listen stats
    bind 127.0.0.1:8404   # Sadece localhost — dışarıya açmayın!
    stats enable
    stats uri /haproxy-stats
    stats realm HAProxy\ Statistics
    stats auth admin:GUCLU_SIFRE_BURAYA
    stats refresh 10s
    stats show-node
    stats show-legends
    # Yerleşik Prometheus exporter (HAProxy 2.4+): /metrics yolu
    http-request use-service prometheus-exporter if { path /metrics }
    # Yönetim aksiyonlarını yalnız yönetim ağına aç
    stats admin if { src 127.0.0.1/32 }
```

Servisi yeniden yükledikten sonra **SSH tüneli ile** eriş:

```bash
ssh -L 8404:127.0.0.1:8404 kullanici@SUNUCU_IP_ADRESI
# Yerel tarayıcıda: http://localhost:8404/haproxy-stats
```

> ⚠️ Stats sayfasını ve `/metrics` ucunu **dış dünyaya açma**. Config'te düz metin duran `stats auth` parolası herkesin erişebildiği bir ağda ciddi risktir: parolayı uzun ve sunucuya özgü üret, `haproxy.cfg` izinlerini `640 root:haproxy` yap, ve erişimi `127.0.0.1` + SSH tüneliyle sınırlı tut. Sürümüne göre hash'lenmiş parola seçenekleri de vardır; kullanmadan önce `haproxy -vv` ve resmî dokümanla doğrula.

> 💡 `stats admin if { src 127.0.0.1/32 }` olmadan ekrandaki "disable/enable server" butonları çalışmaz; vermemek güvenli varsayılandır. Elle sunucu çıkarıp koyma ihtiyacı doğarsa Adım 6.2'deki socket komutları daha denetilebilir.

> 💡 **Parolayı config içinde düz metin tutmamak:** HAProxy 2.0+ config içinde ortam değişkeni genişletmesini destekler. systemd tarafına `EnvironmentFile` ile ver, config'de `${...}` kullan:
>
> ```bash
> printf 'STATS_USER=admin\nSTATS_PASS=uzun-ve-ozel-parola\n' | sudo tee /etc/haproxy/stats.env
> sudo chmod 600 /etc/haproxy/stats.env
> sudo systemctl edit haproxy      # içine: [Service]\nEnvironmentFile=/etc/haproxy/stats.env
> ```
>
> ```text
>     stats auth ${STATS_USER}:${STATS_PASS}
> ```
>
> Yüklemeden önce `sudo haproxy -c -f /etc/haproxy/haproxy.cfg` çalıştır — ancak bu komut değişkenleri **senin shell ortamından** okur; servis ortamıyla aynı olduğunu doğrulamak için `systemctl show -p EnvironmentFiles haproxy` kullan. Hash'lenmiş parola desteği de vardır; sürümünde çalışıp çalışmadığını resmî yönetim dokümanından teyit et.

#### 6.2 Stats Socket ile Canlı Durum

```bash
# Paket gerekli olabilir: sudo apt install -y socat
echo "show stat"  | sudo socat stdio /run/haproxy/admin.sock | head
echo "show info"  | sudo socat stdio /run/haproxy/admin.sock
echo "show errors" | sudo socat stdio /run/haproxy/admin.sock
echo "show sess" | sudo socat stdio /run/haproxy/admin.sock | head -50
# Detaylı dump için "show sess all" kullan — çok kalabalık çıktı üretir, yalnız debug amaçlıdır

# Bir sunucuyu devre dışı / aktif etme
echo "disable server web_servers/web2" | sudo socat stdio /run/haproxy/admin.sock
echo "enable server web_servers/web2"  | sudo socat stdio /run/haproxy/admin.sock
```

> 💡 `show stat` alanlarından en kritik ikisi: `status` (UP/DOWN/MAINT) ve `dreq`/`dresp` (reddedilen istek/yanıt sayısı). "Site açılıyor ama yavaş" şikâyetinde `qcur` (kuyruk) ve `scur` (aktif bağlantı) değerlerine bak.

#### 6.3 Prometheus ile İzleme

`http-request use-service prometheus-exporter if { path /metrics }` satırı, `haproxy_` öneki ile klasik metrikleri (`haproxy_server_up`, `haproxy_frontend_http_requests_total`, `haproxy_backend_active_servers` vb.) verir. Scrape'i yönetim ağından yap; `node_exporter` + `haproxy_exporter` ikilisi yerine yerleşik exporter tercih et (tek süreç, ek servis yok).

> 💡 Alert kuralı olarak `haproxy_server_up == 0` ve `haproxy_backend_active_servers == 0` eşikleri, "site açık ama kimse bilmiyor" durumunu sen kullanıcı şikâyeti almadan yakalar.

#### 6.4 Sertifika değişimini reload'suz uygulamak (isteğe bağlı)

`crt` seçeneğine **dizin** vermişsen (`/etc/haproxy/certs/`) HAProxy 2.2+ sertifikayı çalışır durumda değiştirebilir; reload'e gerek kalmaz ve mevcut bağlantılar kopmaz:

```bash
# Master socket yolunu doğrula (Debian unit'i genelde -S /run/haproxy-master.sock kullanır)
systemctl cat haproxy | grep -- '-S '

# Yeni sertifikayı hazırla (aynı dizine yaz), sonra master sokete commit ettir:
D=$(base64 -w0 /etc/haproxy/certs/panel.alanadiniz.com.pem)
echo "set ssl cert /etc/haproxy/certs/panel.alanadiniz.com.pem payload $D" | sudo socat stdio /run/haproxy-master.sock
echo "commit ssl cert /etc/haproxy/certs/panel.alanadiniz.com.pem"       | sudo socat stdio /run/haproxy-master.sock
echo "persist ssl cert /etc/haproxy/certs/panel.alanadiniz.com.pem"      | sudo socat stdio /run/haproxy-master.sock
```

> ⚠️ Bu komutlar **master worker modunda ve master soketinde** çalışır; normal `admin.sock` reddeder. Adım 4.3'teki deploy hook'un yaptığı `systemctl reload haproxy` her sürümde güvenli yoldur — bu bölümü onu değiştirmek için değil, reload sıklığını azaltmak için kullan. `commit` yerine önce `abort ssl cert` ile vazgeçebilir, `show ssl cert <dosya>` ile durumu görebilirsin. Ayrıca `persist` yazmadıysan reload sonrası diskteki eski dosya geri yüklenir.

---

### Adım 7: Güvenlik Duvarı Yapılandırması

Kullandığınız güvenlik duvarı çözümünde (UFW, iptables, FortiGate, pfSense vb.) aşağıdaki portlara izin verin:

| Port | Protokol | Açıklama |
|---|---|---|
| **22** | TCP | SSH Uzak Erişim |
| **80** | TCP | HTTP Trafiği (ve ACME HTTP-01 doğrulaması) |
| **443** | TCP | HTTPS Trafiği |
| **8404** | — | **AÇMAYIN.** Stats/metrik ekranı; `127.0.0.1`'e bağlı, SSH tüneliyle erişilir |

> ⚠️ **Port 8404 (Stats Sayfası):** Dış dünyaya açmayın. Adım 6.1'de `bind 127.0.0.1:8404` olarak bırakıldı; uzaktan erişim için SSH tüneli kullan:
>
> ```bash
> ssh -L 8404:127.0.0.1:8404 kullanici@SUNUCU_IP_ADRESI
> ```

> 💡 HAProxy host üzerinde (`systemd`) çalıştığından UFW/iptables kuralları **normal şekilde uygulanır**. Aynı makinede Docker konteyneri de çalışıyorsa dikkat: Docker, `ports:` ile yayınlanan konteyner portlarını FORWARD zincirinden geçirdiği için UFW `INPUT` kurallarını baypas eder — HAProxy'nin dinlediği portlarla çakışan bir konteyner yayını olmamasına özen göster.

---

### Adım 8: Loglar ve Tanılama

```bash
# Canlı HAProxy logları — Debian 13'te journald, /dev/log üzerinden akar
sudo journalctl -u haproxy -f

# Servis durumu ve son hata satırları
sudo systemctl status haproxy --no-pager

# Yapılandırma değişikliği sonrası — aktif bağlantıları kesmeden yeniden yükleme
sudo systemctl reload haproxy
```

#### 8.1 `/var/log/haproxy.log` İstiyorsan (rsyslog)

`journalctl` çoğu zaman yeterlidir. Merkezî log ya da satır bazlı araçsal analiz istiyorsan rsyslog kuralı eklemelisin — `haproxy.cfg` içindeki `log /dev/log` satırı tek başına dosyaya yazmaz:

```bash
sudo apt install -y rsyslog

sudo tee /etc/rsyslog.d/99-haproxy.conf > /dev/null <<'EOF'
# HAProxy'nin local0/local1 facility çıktısını ayrı dosyaya yaz
module(load="imuxsock")
$umask 0000
local0.*  /var/log/haproxy.log
&stop
local1.*  /var/log/haproxy.log
EOF

sudo systemctl restart rsyslog
sudo tail -f /var/log/haproxy.log
```

> 💡 Facility adları config ile **eşleşmek** zorunda (`log /dev/log local0`). Logları uzak syslog sunucusuna göndermek için: `log 10.0.0.20:514 local0`.

#### 8.2 Log Satırını Okumak

`option httplog` ile tipik bir satır:

```text
Sep 12 10:30:11 proxy haproxy[1234]: 203.0.113.7:52114 [12/Sep/2026:10:30:11.204] https_frontend web_servers/web2 0/0/1/42/43 200 1810 - - ---- 3/3/0/0/0 0/0 "GET /api/health HTTP/2.0"
```

En çok işe yarayan alanlar: `frontend/backend/server` üçlüsü (istek hangi kurala ve hangi sunucuya düştü), `Tq/Tw/Tc/Tr/Tt` zamanları (**`Tc` yüksek** = backend'e bağlantı kuramıyor, **`Tr` yüksek** = uygulama yavaş), durum kodu ve son alandaki istek satırı.

> 💡 `mode tcp` bloklarında `option httplog` yerine **`option tcplog`** kullan; aksi halde satır eksik veya yanlış yorumlanır.

---

### Adım 9: L4 Modu (`mode tcp`), TLS Passthrough ve SNI Yönlendirme

> 💡 **L7 mi L4 mü?** HAProxy'yi nginx'ten ayıran asıl güç burada: `mode tcp` katman 4'te çalışır, HTTP'yi çözmez. Bu yüzden HTTP dışı protokolleri (SSH, veritabanı, MQTT) dengeleyebilir, şifreyi **bozmadan** SNI'a göre yönlendirebilir ve `timeout tunnel` derdi olmadan uzun ömürlü bağlantıları taşır.

#### 9.1 TCP Yük Dengeleme (MQTT / Veritabanı / SSH)

```text
frontend tcp_in
    bind *:8883
    mode tcp
    option tcplog
    timeout client 30m          # mode tcp'de client/server timeout'ları kritik
    timeout server 30m
    default_backend mqtt_cluster

backend mqtt_cluster
    mode tcp
    balance roundrobin
    option tcp-check
    tcp-check connect
    server mq1 10.0.0.21:8883 check inter 5s fall 3
    server mq2 10.0.0.22:8883 check inter 5s fall 3
```

#### 9.2 Gerçek İstemci IP'sini Koruma (PROXY protocol)

`mode tcp` içinde `option forwardfor` **işlemez** (o bir HTTP başlığıdır). Arka uygulamanın gerçek istemci IP'sini görmesi için PROXY protocol kullan:

```text
    server mq1 10.0.0.21:8883 send-proxy check        # HAProxy gönderir
```

Arka uçta dinleyen taraf bunu kabul etmeli: HAProxy frontend'i için `bind :8883 ssl crt ... accept-proxy`, nginx için `listen 8883 proxy_protocol` + `real_ip` ayarı. Bir LB zincirinde (`Cloudflare → HAProxy → app`) her halkanın bunu ya taşıması ya da hiç göndermemesi gerekir; yarım bırakılırsa bağlantılar `PROXY ...` metniyle açılır ve bozuk yanıt alırsın.

#### 9.3 TLS Passthrough ve SNI ile Yönlendirme (sertifika backend'de kalır)

```text
frontend tls_in
    bind *:443
    mode tcp
    option tcplog

    # ClientHello'yu beklemek için inceleme gecesi — trafiği bu kadar geciktirir
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }

    use_backend bk_api if { req.ssl_sni -i api.alanadiniz.com }
    use_backend bk_grpc if { req.ssl_sni -i grpc.alanadiniz.com }
    default_backend bk_web

backend bk_api
    mode tcp
    server api1 10.0.0.31:443 check
backend bk_grpc
    mode tcp
    server grpc1 10.0.0.41:443 check
backend bk_web
    mode tcp
    server web1 192.168.1.10:443 check
```

> ⚠️ **Passthrough'un bedeli:** HAProxy şifreyi çözmediği için L7 ACL'leri (`path`, `hdr(host)`, header ekleme, health check'in HTTP olanı) **çalışmaz**; sertifika backend'lerde durur ve yenilemesini de backend'ler yapar. Merkezi TLS + L7 kontrol istiyorsan Adım 4'teki sonlandırma modeline dön.
>
> **Fetch adı uyarısı:** Yukarıdaki bloklarda güncel adlar (`req.ssl_hello_type`, `req.ssl_sni`) kullanıldı. Eski dokümanlarda göreceğin `req_ssl_hello_type` / `req_ssl_sni` yazımları alias'tır; HAProxy 3.0'da kaldırılmış olabilirler — `haproxy -c` doğrulaması bu farkı yakalar.

> 💡 **Karışık mod mümkün:** `mode tcp` + `bind ... ssl crt /etc/haproxy/certs/` birlikte kullanılabilir; TLS'i burada sonlandırıp arkaya **ham TCP** taşırsın (TLS'li Redis gibi). Bu durumda `option tcplog` kullanmaya devam et.

---

### Adım 10: Çok Domainli Reverse Proxy (Host Switching)

Aynı HAProxy üzerinde birden çok alan adını farklı servislere yönlendirmek, yük dengeleme kadar yaygın kullanılan moddur:

```text
frontend fe_main
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/ alpn h2,http/1.1

    # HTTP → HTTPS (ACME yolu hariç, bkz. Adım 4.3)
    http-request redirect scheme https code 301 unless { ssl_fc } { path_beg /.well-known/acme-challenge/ }

    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-header X-Forwarded-Proto http  unless { ssl_fc }

    # Host başlığına göre backend
    use_backend be_panel if { hdr(host) -i panel.alanadiniz.com }
    use_backend be_cloud if { hdr(host) -i cloud.alanadiniz.com }
    use_backend be_api   if { hdr(host) -i api.alanadiniz.com  }

    # Path bazlı yönlendirme (host'tan bağımsız) — UYARI: bu satır metrik ucunu public 80/443'te
    # açar. Yönetim ağına kısıtla (bkz. Adım 6.3) ya da kaldır.
    use_backend be_metrics if { path_beg /metrics }

    default_backend be_web

backend be_panel
    server p1 127.0.0.1:8080 check
backend be_cloud
    server c1 127.0.0.1:9000 check
backend be_api
    server a1 127.0.0.1:3000 check
backend be_metrics
    server m1 127.0.0.1:9101 check
backend be_web
    server w1 192.168.1.10:80 check
    server w2 192.168.1.11:80 check
```

#### 10.1 Domain Sayısı Artınca: Map Dosyası

Her domain için `use_backend` satırı yazmak uzarsa, eşlemeyi dışarı taşı:

```bash
sudo nano /etc/haproxy/backends.map
```

```text
panel.alanadiniz.com   be_panel
cloud.alanadiniz.com   be_cloud
api.alanadiniz.com     be_api
```

```text
frontend fe_main
    # ... bind satırları ...
    use_backend %[req.hdr(host),lower,map(/etc/haproxy/backends.map,be_web)]
```

> 💡 Map dosyası reload anında okunur; domain ekledikten sonra `haproxy -c` + `systemctl reload haproxy` şart. Dosya `root` sahibi ve en az `644` olmalı. Sertifika/map/auth dosyaları **açılışta, chroot'a geçilmeden önce** okunduğu için `/var/lib/haproxy` chroot ağacının içinde görünmeleri gerekmez. (Erişim hatası alırsan ilk şüpheli dosya izinleri olsun; `ls -l /etc/haproxy/` ile kontrol et.)

> ⚠️ **Map dosyasında geçen HER backend config'te tanımlı olmak zorundadır.** `be_api` satırı var ama `backend be_api` bloğu yoksa `haproxy -c` `refers to an undefined backend 'be_api'` hatası verir. Map'i genişletirken backend'i de ekle.

#### 10.2 DNS Tabanlı Backend Çözümü (`resolvers`)

55 servisli bir ortamda backend'ler sık sık IP/servis değiştiriyorsa (konteyner, bulut, VM taşınması), statik `server` satırları yerine DNS'ten canlı çözümü:

```text
resolvers dns_filo
    nameserver dns1 10.0.0.53:53
    resolve_retries 3
    timeout resolve 1s
    timeout retry   1s
    accepted_statuscodes 200
    hold valid 10s
    hold nx 30s

backend be_panel
    server tpl @local            # çözümlenen ilk adres
    default-server resolvers dns_filo
    server node1 panel.ic.local:8080 check resolvers dns_filo
```

> ⚠️ `resolvers` yalnız **isimle** tanımlanmış sunucularda işe yarar; konteyner/orkestrasyon ortamında `init-addr last,libc,none` ve `hold stale` davranışını test ağında izle. Ayrıca HAProxy'nin çözümleyiciye gerçekten UDP 53 gönderebildiğini `ss -unp | grep haproxy` ile doğrula — chroot/where networking kısıtı varsa sessizce eski IP'ye yapışıp kalır.

---

### Adım 11: Oran Sınırlama ve Kaba Kuvvet Koruması

```text
# Aşağıdaki satırları Adım 10'daki `fe_main` frontend'inin İÇİNE ekle;
# yeni bir `frontend fe_main` veya `bind *:80` bloğu DAHA açma (bölüm adı ve bind çakışır).

    # İstemci başına son 10 saniyedeki istek sayısı
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 50 }

    # Login gibi hassas uçlarda daha sert kural
    acl login path_beg /login
    http-request deny deny_status 429 if login { sc_http_req_rate(0) gt 10 }
```

> 💡 Bağlantı sayısı ile sınırlamak istersen tabloyu `store http_req_rate(10s),conn_cur,conn_rate(3s)` ile genişlet ve `{ sc_conn_rate(0) gt 25 }` gibi ikinci bir koşul kullan. Aynı `frontend` içinde yalnız **bir** `stick-table` tanımlanabildiğini, ek veri türlerini aynı satırın `store` listesine eklemen gerektiğini unutma.

> ⚠️ **CDN/LB arkasında `src` doğru IP değildir.** `src`, paket kaynağıdır (yani Cloudflare/ön load balancer adresi); tüm dünya tek IP'den geliyormuş gibi görünür ve limit herkese vurur. İki gerçek çözüm:
>
> 1. **PROXY protocol:** önündeki düğüm `send-proxy` ile gönderiyorsa burada `bind ... accept-proxy` ile dinle; `src` gerçekten istemci adresi olur (bkz. Adım 9.2).
> 2. **Başlık bazlı izleme:** `option forwardfor` bunu **düzeltmez** (o seçenek backend'e giden istekte XFF *ekler*); istemciyi başlıktan izlemek için `http-request track-sc0 hdr_ip(X-Forwarded-For,-1)` kullan — **yalnızca bu başlığı koyan kaynağa güveniyorsan** (proxy zinciri sabitse), aksi halde herkes sahte başlıkla limitten kaçar.

> 💡 `deny_status 429` HAProxy 2.2+ sürümlerinde geçerlidir; eski sürümlerde varsayılan `403` döner — `haproxy -c` hata vermezse sürümünü kontrol et. Aşım loglarda `HTTP 429` + `state` bilgisiyle görünür; yanlışlıkla meşru kullanıcıları engellemediğini `stats` ekranından izleyin.

---

### Adım 12: Güncelleme, Yedekleme ve Geri Yükleme

#### 12.1 Yedekleme

```bash
sudo mkdir -p /backup && sudo chmod 700 /backup
sudo tar -czf /backup/haproxy_$(date +%F).tar.gz -C / etc/haproxy /etc/letsencrypt
dpkg -l | grep -E '^ii\s+haproxy' | sudo tee /backup/haproxy_paket_$(date +%F).txt
```

> 🔒 `/etc/haproxy/certs/*.pem` dosyaları **özel anahtar** içerir. Arşivi sunucu dışına çıkaracaksan şifrele (`gpg -c /backup/haproxy_<tarih>.tar.gz`); `chmod 700` yalnızca yerel izindir, şifreleme değildir.

```bash
# Geri yükleme
sudo tar -xzf /backup/haproxy_<TARIH>.tar.gz -C /
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # önce doğrula
sudo systemctl reload haproxy
```

#### 12.2 Sürüm Güncelleme

```bash
sudo apt update
sudo apt install --only-upgrade -y haproxy

# Config'i yeni binary ile doğrula, sonra kesintisiz yükle
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy

# Major sıçramada (ör. 2.6 -> 3.0) yeni binary için restart gerekebilir
# sudo systemctl restart haproxy
```

> ⚠️ **Debian güncellemesi sırasında `haproxy.cfg` için "keep the local version currently installed" sorulursa mevcut sürümü seç.** Aksi halde kişisel ayarların `.dpkg-old` / `.dpkg-dist` dosyasına taşınır ve HAProxy varsayılan config ile ayağa kalkar. Kalıntıları `diff` ile karşılaştırıp kendin birleştir.
>
> **Rollback:** Major sürüm sonrası config'te kaldırılmış bir direktif kullanıyorsan `haproxy -c` seni güncellemeden önce durdurur; sürüm düşürme `apt install haproxy=<eski-sürüm>` ile ancak o sürüm hâlâ apt önbelleğinde/archiv'de varsa mümkündür — bu yüzden **önce yedek (12.1), sonra güncelleme**.

---

## 🧩 Nihai Örnek: Tek `haproxy.cfg`

Adımlardaki parçaların **bir arada çalışan** hâli. Bölüm adları benzersizdir; kopyalayıp `/etc/haproxy/haproxy.cfg` yerine koyabilir, sonra kendi backend adreslerini düzenleyebilirsin:

```text
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    hard-stop-after 30m
    ssl-default-bind-ciphers ECDHE+AESGCM:ECDHE+AES256:ECDHE+AES128:!aNULL:!MD5:!DSS
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options prefer-client-ciphers no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  forwardfor
    option  http-server-close
    retries 3
    timeout connect          5s
    timeout client           1m
    timeout server           1m
    timeout tunnel           1h
    timeout http-request     10s
    timeout http-keep-alive  10s
    timeout check            10s
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# ---- HTTP: ACME'yi karşıla, gerisini HTTPS'e çevir ----
frontend fe_http
    bind *:80
    acl acme path_beg /.well-known/acme-challenge/
    # http-request kuralları backend seçiminden ÖNCE çalışır: redirect'i ACME için
    # muaf tutmazsan use_backend hiç devreye giremez ve HTTP-01 301'e takılır.
    http-request redirect scheme https code 301 unless acme
    use_backend be_acme if acme

# Yöntem C kullanmıyorsan bu backend'i sil ve 4.3'teki Yöntem A/B'ye geç
backend be_acme
    server static 127.0.0.1:8888

# ---- HTTPS: SNI ile çok domain, dizinden sertifika seçimi ----
frontend fe_https
    bind *:443 ssl crt /etc/haproxy/certs/ alpn h2,http/1.1
    http-request set-header X-Forwarded-Proto https
    http-response set-header X-Content-Type-Options nosniff always
    http-response set-header Strict-Transport-Security "max-age=15768000; includeSubDomains" always

    use_backend %[req.hdr(host),lower,map(/etc/haproxy/backends.map,be_web)]

backend be_web
    balance roundrobin
    option httpchk
    http-check send meth GET uri /health ver HTTP/1.1 hdr Host localhost
    http-check expect status 200
    default-server inter 3s rise 2 fall 3 maxconn 200
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
    server web3 192.168.1.12:80 check backup

backend be_panel
    server p1 127.0.0.1:8080 check

# backends.map içinde geçen her backend burada tanımlı olmalı:
backend be_cloud
    server c1 127.0.0.1:9000 check
backend be_api
    server a1 127.0.0.1:3000 check

# ---- L4 örneği (MQTT): gerekli değilse bu iki bloğu sil ----
frontend fe_mqtt
    bind *:8883
    mode tcp
    option tcplog
    timeout client 30m
    timeout server 30m
    default_backend be_mqtt

backend be_mqtt
    mode tcp
    balance roundrobin
    option tcp-check
    tcp-check connect
    server mq1 10.0.0.21:8883 check inter 5s fall 3
    server mq2 10.0.0.22:8883 check inter 5s fall 3

# ---- İzleme ----
listen stats
    bind 127.0.0.1:8404
    stats enable
    stats uri /haproxy-stats
    stats realm HAProxy\ Statistics
    stats auth admin:GUCLU_SIFRE_BURAYA
    stats refresh 10s
    stats show-node
    stats show-legends
    http-request use-service prometheus-exporter if { path /metrics }
```

```bash
# Bu config'i koymadan önce: certbot Yöntem C için 8888'de statik sunucu gerekir,
# backends.map dosyası olmalı ve /etc/haproxy/certs/ içinde birleşik .pem'ler bulunmalı.
sudo haproxy -c -f /etc/haproxy/haproxy.cfg && sudo systemctl reload haproxy
```

> ⚠️ **Bu örnek olduğu gibi alınmadan önce üç şeyi tamamla:** (1) `/etc/haproxy/backends.map` dosyası — ve içinde geçen **her backend'in config'te tanımlı olması** (`refers to an undefined backend` hatası almamak için), (2) `/etc/haproxy/certs/` içinde en az bir `<domain>.pem`, (3) `be_acme` bloğu — Yöntem A (DNS-01) kullanıyorsan `be_acme` bloğunu, `acl acme` satırını ve `use_backend be_acme` satırını **sil**; aksi halde ACME olmayan her istek 8888'e gitmeye çalışır ve 503 üretir.

---

## 💡 Sık Karşılaşılan Hatalar ve Çözümleri

1. **Yapılandırma Doğrulama:** Her değişiklikten önce `sudo haproxy -c -f /etc/haproxy/haproxy.cfg`. Bu komut servise dokunmaz; hatalı config'i **yüklemeden** yakalarsın. `reload` sonrası servisin ayakta kaldığını `systemctl is-active haproxy` ile teyit et.
2. **`reload` vs `restart`:** `reload` yeni süreç olarak ayağa kalkar ve eski süreç aktif bağlantılar bitene kadar (ya da `hard-stop-after` dolana kadar) çalışmaya devam eder; production'da tercih edilen budur. Major sürüm sıçramasında yeni binary'nin tamamen devreye girmesi için `restart` gerekebilir.
3. **502 / 503 / 504 ayrımı:** **502** = backend'e ulaştı ama yanıt bozuk ya da hiç gelmedi (uygulama hatası, yanlış port). **503** = kullanılabilir sunucu yok (hepsi DOWN, maintenance, ya da Adım 11 limiti) — logda `no available server`. **504** = backend geç yanıtlıyor (`timeout server` / `timeout tunnel` dolu). Üçünün teşhisi farklı; log satırındaki `backend/server` alanına bak.
4. **Oturum Sürekliliği (Stickiness) eksik yazılmıştı:** `cookie SRV insert indirect nocache` satırı **tek başına yetmez**; her sunucu satırında `cookie <ad>` parametresi de olmalı:

   ```text
   backend web_servers
       cookie SRV insert indirect nocache
       server web1 192.168.1.10:80 cookie w1 check
       server web2 192.168.1.11:80 cookie w2 check
   ```

   Çerez istemiyorsan IP bazlı stick kullan: `stick-table type ip size 100k expire 30m` + `stick on src`. Eski `appsession` seçeneği kaldırılmıştır — yeni kurulumlarda kullanma.
5. **`/health` Endpoint:** Sağlık kontrolünün çalışması için backend uygulamalarında `GET /health` isteğine `200 OK` dönen bir endpoint bulunmalı. 302 dönen bir sağlık ucu (login'e yönlendirme) `http-check expect status 200` ile **DOWN** görünür — bu genellikle istenen davranıştır.
6. **Kurumsal SSL Sertifikası Yenileme:** Yeni `.crt`/`.key` dosyalarını aynı sırayla birleştirip (`cat domain.crt intermediate.crt domain.key > /etc/haproxy/certs/<domain>.pem`) `systemctl reload haproxy` çalıştır; kesinti olmaz. LE kullanıyorsan bu işi Adım 4.3'teki deploy hook otomatik yapar.
7. **Büyük cookie/JWT ile `400 Bad Request`:** `tune.bufsize` varsayılanının üzerine çıkan başlık boyutunda HAProxy isteği reddeder. Global'e `tune.bufsize 32768` ekleyip `haproxy -c` ile doğrula (değeri iki katına çıkararak test et; gereğinden büyük bufsize bellek maliyeti yaratır).
8. **`['backend'|'frontend'] 'x' already declared`:** Bölüm adları config genelinde benzersiz olmalı. Adımlardaki örnekleri kopyalarken aynı ada ikinci bir blok bırakma; yukarıdaki **Nihai Örnek** tek parça hâlinde doğrudur.
9. **WebSocket ~60 sn'de kopuyor:** `timeout tunnel` tanımsızsa client/server timeout'larının uzunu kullanılır. `timeout tunnel 1h` (defaults veya ilgili frontend) ile çözülür; L7'de ayrıca `timeout http-request` kısa kalıyorsa yavaş gövdeli POST'lar kesilir.
10. **`socat: command not found`:** Stats socket komutları için `sudo apt install -y socat`. Socket'e yazma izni için `level admin` + `mode 660` gerekir; `sudo` kullanmıyorsan kullanıcıyı `haproxy` grubuna ekle.
11. **Gerçek client IP kaybolduysa:** L7 için `option forwardfor` (config'te var) + uygulamanın `X-Forwarded-For`'u güvenilir proxy'den okuması; L4 için `send-proxy`/`accept-proxy` (Adım 9.2). İkisini karıştırmak tipik hatadır.
12. **Bir sonraki adım:** İki HAProxy düğümü arasında VIP failover için **keepalived** (VRRP) kullanılır; tek HAProxy'de DNS ağırlıklı round-robin yeterli değildir. Health check'i dışarıdan izlemek için Prometheus + Alertmanager (Adım 6.3) ve `haproxy_server_up < 1` uyarısı ekleyin.

> 📌 **Sürüm notu:** Bu rehber Debian 13 (trixie) ve **haproxy 3.0.x (LTS)** esas alınarak hazırlanmıştır. 2.x sürümlerinde bazı seçenek adları/eski yazımlar farklı olabilir; bir satır `haproxy -c`'de tanınmıyorsa sebebi sürüm farkıdır — resmî sürüm notlarından kontrol et.

*Debian 13 üzerinde HAProxy kurulumunuz ve yapılandırmanız başarıyla tamamlanmıştır.*

---

- Author: faruk-guler
- GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
- Page: [www.farukguler.com](https://www.farukguler.com)
