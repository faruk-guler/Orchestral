# Squid Proxy Server on Debian 13 (Trixie) Forward Proxy

**Squid**, istemcilerin internete çıkışını düzenleyen yüksek performanslı, açık kaynaklı bir **ileri proxy (Forward Proxy)**, önbellekleme (caching) ve içerik filtreleme sunucusudur. İçerideki cihazların dışarıya (internete) nasıl çıkacağını kontrol eder, IP adresinizi gizler (anonim proxy) ve yerel ağdaki bant genişliğini optimize eder.

Bu rehber, **Debian 13 (Trixie)** üzerinde Squid Proxy'nin doğrudan paket yöneticisi ile kurulumunu, Basic Auth kimlik doğrulamasını, Elite Proxy (yüksek anonimlik), HTTPS filtreleme, cache ve güvenlik sertleştirme yapılandırmasını kapsamaktadır.

---

## 📘 Adım Adım Manuel Kurulum

### Adım 1: Sistem Güncellemeleri ve Paket Kurulumları

Sistem paketlerinizi güncelleyin, Squid ve bağımlılıklarını yükleyin:

```bash
# 1.1 Sistem güncellemesi
sudo apt update && sudo apt full-upgrade -y

# 1.2 Squid ve ek doğrulama araçlarını (apache2-utils) yükleyin
sudo apt install -y squid apache2-utils

# 1.3 Servisleri başlatın ve otomatik açılışa ekleyin
sudo systemctl enable --now squid
sudo systemctl status squid
```

---

### Adım 2: Temel Yapılandırma (`squid.conf`)

Squid'in tüm konfigürasyonu `/etc/squid/squid.conf` dosyasında tutulur. Değişiklik yapmadan önce yedek alın:

```bash
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.bak
sudo nano /etc/squid/squid.conf
```

#### Port Değiştirme (İsteğe Bağlı)

Varsayılan port `3128`'dir. Güvenlik amacıyla farklı bir port kullanmak isterseniz `squid.conf` içindeki şu satırı bulun ve port numarasını değiştirin:

```text
# Varsayılan:
http_port 3128

# Örneğin 8080 yapmak için:
http_port 8080
```

> 💡 **Not:** Portu değiştirirseniz bu rehberin ilerleyen adımlarındaki (Güvenlik Duvarı, Test, İstemci Yapılandırması) tüm `3128` referanslarını da kendi port numaranızla güncellemeyi unutmayın.

---

### Adım 3: IP Bazlı Erişim İzni Verme (ACL)

Varsayılan ayarlar dışarıdan gelen tüm bağlantıları reddeder (`http_access deny all`). Belirli IP adreslerine izin vermek için ACL kuralı ekleyin:

```text
# 1. Kendi Ev/Ofis IP Adresinizi Tanımlayın
acl allowed_clients src 185.190.10.45 93.180.2.11

# 2. Tanımlanan IP'lere Erişim İzni Verin
http_access allow allowed_clients

# 3. Diğer Tüm İstekleri Engelleyin (Zaten dosya sonunda vardır)
http_access deny all
```

> ⚠️ **Dikkat:** Debian'ın varsayılan `squid.conf` dosyasında zaten `acl localnet`, `acl SSL_ports`, `acl Safe_ports` gibi hazır kurallar bulunur. Bu ACL'i eklerken bu varsayılan satırları **silmeyin** — özellikle `SSL_ports`/`CONNECT` kısıtlaması kaldırılırsa proxy'niz her porta tünelleme yapabilen güvensiz bir yapıya dönüşür.

Ayarları uygulamadan önce mutlaka sözdizimini kontrol edin, sonra uygulayın:

```bash
sudo squid -k parse && sudo systemctl reload squid
```

---

### Adım 4: Kullanıcı Adı ve Parola ile Kimlik Doğrulama (Basic Auth)

Sabit IP adresiniz yoksa veya proxy'nize kullanıcı adı ve şifre ile giriş yapılmasını istiyorsanız:

#### 4.1 Parola Dosyası Oluşturma

```bash
# İlk kullanıcı için -c parametresi yeni dosya oluşturur
sudo htpasswd -c /etc/squid/passwd ahmet_kullanicisi

# İkinci kullanıcı eklerken -c KULLANMAYIN (dosyanın üzerine yazar)
sudo htpasswd /etc/squid/passwd mehmet_kullanicisi

# Dosya izinlerini ayarlayın
sudo chown proxy:proxy /etc/squid/passwd
sudo chmod 640 /etc/squid/passwd
```

#### 4.2 `squid.conf` Dosyasına Auth Kurallarını Ekleme

Tüm `http_access allow ...` satırlarını ekledikten sonra, dosyanın en altındaki `http_access deny all` kuralından **ÖNCE** şunları ekleyin:

```text
# Basic Auth Yardımcı Program Tanımı
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5
auth_param basic realm Squid Proxy Sunucusuna Hos Geldiniz
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive on

# ACL Tanımı
acl authenticated_users proxy_auth REQUIRED

# Erişim İzni
http_access allow authenticated_users
```

*(Önemli: `http_access deny all` kuralının dosyanın en sonunda kaldığından emin olun, çünkü Squid kuralları yukarıdan aşağıya doğru okur.)*

Değişiklikleri uygulamadan önce doğrulayın:

```bash
sudo squid -k parse && sudo systemctl restart squid
```

---

### Adım 5: Yüksek Anonimlik Ayarları (Elite Proxy)

Varsayılan olarak Squid, hedef web sitelerine orijinal IP adresinizi ve proxy kullandığınızı gösteren HTTP başlıkları gönderir (`X-Forwarded-For`, `Via`). Tam anonimlik için `squid.conf` dosyasının en altına ekleyin:

```text
# --- YÜKSEK ANONİMLİK (ELITE PROXY) AYARLARI ---
forwarded_for off
via off

# Gerçek IP ve İstemci Bilgisi Sızdıran HTTP Header'larını Engelle
request_header_access X-Forwarded-For deny all
request_header_access Via deny all
request_header_access From deny all
request_header_access Referer deny all
request_header_access User-Agent allow all
request_header_access Authorization allow all
request_header_access Proxy-Authorization allow all
request_header_access Cache-Control allow all
request_header_access Content-Type allow all
request_header_access All deny all
```

> 💡 **Not:** `forwarded_for off` zaten `X-Forwarded-For` başlığının eklenmesini engeller; `request_header_access` satırı buna ek bir güvenlik katmanı olarak eklenmiştir (redundant ama zararsız).
>
> ⚠️ **Uyarı:** `request_header_access` gibi HTTP standardını "ihlal eden" (violation) directive'ler Squid'in `--enable-http-violations` derleme bayrağı ile aktif olur. Debian'ın resmi paketi bunu varsayılan olarak destekler, ancak yine de değişiklikten sonra parse kontrolü şart:

```bash
sudo squid -k parse && sudo systemctl restart squid
```

> 🔓 **Güvenlik Notu:** İnternete açık, kimlik doğrulamasız (sadece IP whitelist ile) çalışan bir "Elite Proxy" yanlış yapılandırılırsa açık/anonim bir relay'e dönüşüp kötüye kullanılabilir (spam, saldırı gizleme vb.). Mutlaka Adım 4 (Basic Auth) veya sıkı IP kısıtlaması ile birlikte kullanın.

---

### Adım 6: Web Sitesi Engelleme & İçerik Filtreleme

```bash
# 6.1 Engellenecek siteler dosyası oluşturun
sudo nano /etc/squid/blocked_sites.txt
```

Yasaklamak istediğiniz alan adlarını yazın *(başına `.` koyarak alt alan adlarını da yakalayın)*:

```text
.facebook.com
.instagram.com
.tiktok.com
.betting-site.com
```

`squid.conf` dosyasına kuralı ekleyin (Erişim izinlerinden **ÖNCE**):

```text
acl blocked_domains dstdomain "/etc/squid/blocked_sites.txt"
http_access deny blocked_domains
```

Yapılandırmayı doğrulayıp yeniden yükleyin:

```bash
sudo squid -k parse && sudo systemctl reload squid
```

> ⚠️ **Önemli Kısıtlama:** Bu kural yalnızca **düz HTTP** trafiğinde ve HTTPS `CONNECT` isteğinin hedef domain bilgisinde çalışır. Günümüzde trafiğin büyük çoğunluğu HTTPS olduğu için, sitelerin içeriğini (path, sayfa içi filtreleme) görebilmek için **Adım 7 (SSL Bump)** gereklidir. `dstdomain` tabanlı bu kural HTTPS sitelerde sadece CONNECT hedefine bakar, bu da çoğu durumda yeterlidir ama tam filtreleme değildir.

---

### Adım 7: HTTPS Trafiğini Filtreleme (SSL Bump)

Squid, varsayılan olarak HTTPS trafiğini sadece tünelleyip (`CONNECT`) içeriğine bakamaz. Domain bazlı engellemeyi HTTPS'e taşımak için iki yöntem vardır:

#### 7.1 Hafif Yöntem — SNI Bazlı Filtreleme (Önerilen, MITM Yok)

Sertifikayı hiç açmadan, istemcinin gönderdiği SNI (Server Name Indication) bilgisinden domain okuyup engelleme yapılır. İstemcilere sertifika dağıtmaya gerek yoktur:

```text
acl blocked_sni ssl::server_name "/etc/squid/blocked_sites.txt"

http_port 3128 ssl-bump
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump splice all

http_access deny blocked_sni
```

#### 7.2 Tam Yöntem — SSL Bump (MITM, İleri Seviye)

Trafiğin içeriğini (URL path, sayfa içeriği) görmek için Squid'in araya girip sertifikayı yeniden imzalaması gerekir. Bunun için kendi kök sertifikanızı üretip **her istemciye** güvenilir kök sertifika olarak yüklemeniz gerekir.

```bash
# Kök sertifika (CA) oluşturma
sudo mkdir -p /etc/squid/ssl_cert
cd /etc/squid/ssl_cert
sudo openssl req -new -newkey rsa:2048 -sha256 -days 3650 -nodes -x509 \
  -keyout squidCA.pem -out squidCA.pem \
  -subj "/C=TR/O=SirketAdi/CN=Squid Proxy CA"
sudo openssl x509 -in squidCA.pem -outform DER -out squidCA.der

# Sertifika önbellek veritabanı (sslcrtd)
sudo /usr/lib/squid/security_file_certgen -c -s /var/spool/squid/ssl_db -M 4MB
sudo chown -R proxy:proxy /var/spool/squid/ssl_db
```

`squid.conf` içine:

```text
http_port 3128 ssl-bump cert=/etc/squid/ssl_cert/squidCA.pem generate-host-certificates=on
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/spool/squid/ssl_db -M 4MB

acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump blocked_domains
ssl_bump splice all
```

```bash
sudo squid -k parse && sudo systemctl restart squid
```

`squidCA.der` dosyasını istemcilere kök sertifika olarak yükleyin:

- **Windows:** `certutil -addstore -f "ROOT" squidCA.der`
- **Debian/Linux:** `.crt` uzantısıyla `/usr/local/share/ca-certificates/` içine kopyalayıp `sudo update-ca-certificates`
- **Mobil (iOS/Android):** Profil olarak yükleyip "güvenilir kök sertifika" olarak onaylatmak gerekir.

> 🔴 **Kritik Uyarı — Hukuki/Politika Boyutu:** Tam SSL Bump, kullanıcıların şifreli trafiğini gerçek zamanlı olarak deşifre eder (MITM). Bu:
>
> - Sertifika pinning kullanan uygulamaları (bankacılık, bazı mobil uygulamalar) kırar.
> - KVKK/GDPR kapsamında çalışan/kullanıcı bilgilendirmesi ve genelde yazılı onay gerektirir — sadece teknik bir adım değil, bir politika kararıdır.
> - Kurumsal olmayan (ev/kişisel) kullanımda genelde gereksizdir; çoğu senaryoda 7.1'deki SNI yöntemi yeterlidir.

---

### Adım 8: Cache (Önbellekleme) Yapılandırması

Squid'in temel işlevlerinden biri caching'dir, ancak varsayılan kurulumda disk cache kapalıdır. `squid.conf` dosyasının uygun bir yerine ekleyin:

```text
# RAM cache (sık erişilen küçük objeler için)
cache_mem 256 MB
maximum_object_size_in_memory 512 KB

# Disk cache: ufs tipi, /var/spool/squid dizini, 10000 MB, 16 alt klasör, 256 alt-alt klasör
cache_dir ufs /var/spool/squid 10000 16 256
maximum_object_size 100 MB
```

Disk cache dizinlerini **ilk kez** oluşturmak için Squid durdurulup `-z` parametresiyle initialize edilmelidir — bu adım atlanırsa Squid başlamaz veya cache çalışmaz:

```bash
sudo squid -k parse
sudo systemctl stop squid
sudo squid -z
sudo systemctl start squid
```

> 💡 **Not:** `squid -z` sadece cache dizinleri ilk kez oluşturulduğunda veya `cache_dir` boyutu/yolu değiştiğinde çalıştırılır, her reload'da gerekmez.

---

### Adım 9: Güvenlik Duvarı Yapılandırması

Kullandığınız güvenlik duvarı çözümünde (UFW, iptables, FortiGate, pfSense vb.) aşağıdaki portlara izin verin:

| Port | Protokol | Açıklama |
|---|---|---|
| **22** | TCP | SSH Uzak Erişim |
| **3128** | TCP | Squid Proxy (varsayılan) |

Örnek UFW komutları (sadece belirli kaynak IP'ye izin vererek):

```bash
sudo ufw allow from 185.190.10.45 to any port 3128 proto tcp
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

> 💡 **İpucu:** Proxy portunu (3128) herkese açmak yerine, sadece izin vermek istediğiniz kaynak IP adreslerine kısıtlamanız önerilir.

---

### Adım 10: İstemci Tarafında Proxy Test Etme

#### 10.1 Terminal / cURL ile Test

```bash
# Kullanıcı adı ve parolasız (IP izinli ise)
curl -x http://SUNUCU_IP_ADRESI:3128 https://ifconfig.me

# Kullanıcı adı ve parolalı (Basic Auth ile)
curl -x http://ahmet_kullanicisi:SIFRENIZ@SUNUCU_IP_ADRESI:3128 https://ifconfig.me
```

*(Dönen sonuçta sunucunuzun IP adresi görünüyorsa proxy başarıyla çalışıyor demektir.)*

#### 10.2 Tarayıcıda Kullanma

- **Firefox:** `Ayarlar` > `Ağ Ayarları (Network Settings)` > `Manuel Proxy Yapılandırması`.
  - **HTTP Proxy:** `SUNUCU_IP_ADRESİ` | **Port:** `3128`
  - "Tüm protokoller için bu proxy sunucusunu kullan" seçeneğini işaretleyin.
- **FoxyProxy / SwitchyOmega:** Chrome veya Firefox eklentileri ile tek tıkla proxy geçişi yapabilirsiniz.

#### 10.3 `squidclient` ile Cache ve Performans Testi

`curl` sadece bağlantıyı test eder; Squid'in kendi cache manager arayüzü hit ratio, aktif bağlantı ve depolama durumunu gösterir:

```bash
# Genel sunucu bilgisi (uptime, sürüm, istek sayısı)
squidclient -p 3128 mgr:info

# Disk cache durumu / hit ratio
squidclient -p 3128 mgr:storedir

# O anki aktif istekler
squidclient -p 3128 mgr:active_requests
```

> 💡 **Not:** Cache manager sayfaları (`mgr:*`) Debian'ın varsayılan `squid.conf`'unda **sadece localhost'tan** erişime açıktır. Bu komutları proxy sunucusunun kendisi üzerinde çalıştırın. Uzaktan erişim gerekiyorsa `cachemgr_passwd` ile parola tanımlayıp `manager` ACL'ini genişletmeniz gerekir.

---

### Adım 11: İstemci İşletim Sistemi Bazında Proxy Yapılandırması

Tarayıcı dışında, işletim sisteminin veya belirli araçların (apt, git, curl vb.) trafiğini de proxy üzerinden geçirmek isteyebilirsiniz. Aşağıda **Windows** ve **Debian 13 (Trixie)** istemciler için sistem geneli ve araç bazlı yapılandırmalar yer almaktadır.

#### 11.1 Windows İstemci Yapılandırması

**A) Sistem Geneli Proxy (GUI)**

`Ayarlar` > `Ağ ve İnternet` > `Proxy` > `Manuel proxy kurulumu` bölümünden:

- **Proxy sunucusu kullan:** Açık
- **Adres:** `SUNUCU_IP_ADRESİ` | **Port:** `3128`
- Basic Auth kullanıyorsanız, tarayıcı ilk bağlantıda kullanıcı adı/parola soracaktır (Windows GUI'de doğrudan kimlik bilgisi alanı yoktur).

**B) PowerShell / CMD ile Sistem Geneli Proxy (`netsh`)**

WinHTTP katmanını kullanan uygulamalar (Windows Update, bazı servisler) için:

```powershell
# Proxy ayarla (Yönetici olarak PowerShell/CMD)
netsh winhttp set proxy proxy-server="SUNUCU_IP_ADRESI:3128" bypass-list="localhost;127.0.0.1;<local>"

# Mevcut ayarı görüntüle
netsh winhttp show proxy

# Proxy ayarını kaldırma (rollback)
netsh winhttp reset proxy
```

> 💡 **İpucu:** `netsh winhttp` ayarları tarayıcı ayarlarından **bağımsızdır** ve genelde sistem servisleri tarafından kullanılır. Kullanıcı oturumu bazlı proxy için yukarıdaki GUI ayarı yeterlidir.

**C) PowerShell Oturumu / Script Bazlı Proxy**

```powershell
# Ortam değişkeni ile (mevcut oturum için)
$env:HTTP_PROXY  = "http://SUNUCU_IP_ADRESI:3128"
$env:HTTPS_PROXY = "http://SUNUCU_IP_ADRESI:3128"

# Basic Auth ile
$env:HTTP_PROXY  = "http://ahmet_kullanicisi:SIFRENIZ@SUNUCU_IP_ADRESI:3128"

# Invoke-WebRequest için doğrudan parametre
Invoke-WebRequest -Uri "https://ifconfig.me" -Proxy "http://SUNUCU_IP_ADRESI:3128" -ProxyUseDefaultCredentials
```

**D) Kalıcı Kullanıcı Değişkeni (Sistem Genelinde)**

```powershell
[System.Environment]::SetEnvironmentVariable("HTTP_PROXY", "http://SUNUCU_IP_ADRESI:3128", "User")
[System.Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://SUNUCU_IP_ADRESI:3128", "User")
```

*(Değişikliğin etkili olması için oturumu yeniden başlatmanız gerekir.)*

---

#### 11.2 Debian 13 (Trixie) İstemci Yapılandırması

**A) Ortam Değişkenleri (Sistem/Kullanıcı Geneli)**

Tüm kabuk (shell) tabanlı araçlar (`curl`, `wget`, `apt` bazı durumlarda, `git` vb.) için:

```bash
# Sadece mevcut oturum için (geçici)
export http_proxy="http://SUNUCU_IP_ADRESI:3128"
export https_proxy="http://SUNUCU_IP_ADRESI:3128"
export no_proxy="localhost,127.0.0.1,::1"

# Basic Auth ile
export http_proxy="http://ahmet_kullanicisi:SIFRENIZ@SUNUCU_IP_ADRESI:3128"
export https_proxy="$http_proxy"
```

Kalıcı hale getirmek için kullanıcı bazlı `~/.bashrc` / `~/.profile`'a, **sistem geneli** için `/etc/environment` dosyasına ekleyin:

```bash
sudo nano /etc/environment
```

```text
http_proxy="http://SUNUCU_IP_ADRESI:3128"
https_proxy="http://SUNUCU_IP_ADRESI:3128"
no_proxy="localhost,127.0.0.1,::1"
```

*(`/etc/environment` değişiklikleri için yeni bir oturum açmanız/`source` etmeniz gerekir; bazı servisler bu dosyayı otomatik okumaz.)*

**B) APT için Proxy (`apt`)**

Ortam değişkeni `apt` tarafından her zaman otomatik okunmaz; en garantili yöntem ayrı bir apt config dosyası:

```bash
sudo nano /etc/apt/apt.conf.d/95proxy
```

```text
Acquire::http::Proxy "http://SUNUCU_IP_ADRESI:3128";
Acquire::https::Proxy "http://SUNUCU_IP_ADRESI:3128";
```

Basic Auth kullanıyorsanız:

```text
Acquire::http::Proxy "http://ahmet_kullanicisi:SIFRENIZ@SUNUCU_IP_ADRESI:3128";
```

> 💡 **İpucu:** Debian 13'ün kendi güncellemelerini bu Squid üzerinden geçirmek istiyorsanız, `archive.debian.org` ve `security.debian.org` gibi adreslerin proxy `http_access` kurallarınızda (Adım 3/6) engellenmediğinden emin olun.

**C) Git için Proxy**

```bash
# Global (tüm repolar için)
git config --global http.proxy "http://SUNUCU_IP_ADRESI:3128"
git config --global https.proxy "http://SUNUCU_IP_ADRESI:3128"

# Proxy'yi kaldırmak için
git config --global --unset http.proxy
git config --global --unset https.proxy
```

**D) systemd Servisleri İçin Proxy (örn. Docker daemon)**

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```

```ini
[Service]
Environment="HTTP_PROXY=http://SUNUCU_IP_ADRESI:3128"
Environment="HTTPS_PROXY=http://SUNUCU_IP_ADRESI:3128"
Environment="NO_PROXY=localhost,127.0.0.1"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

**E) Doğrulama**

```bash
# apt proxy'sinin etkin olup olmadığını kontrol edin
apt-config dump | grep -i proxy

# genel bağlantı testi
curl -x $http_proxy https://ifconfig.me
```

---

### Adım 12: Squid Servisini Systemd ile Sertleştirme (Hardening)

Squid varsayılan olarak `proxy` sistem kullanıcısıyla çalışır, ancak systemd override ile ek izolasyon katmanları eklemek olası bir güvenlik açığında (ör. uzaktan kod çalıştırma) saldırganın sistemde hareket alanını daraltır. Ana unit dosyasını **doğrudan düzenlemeyin**, override kullanın:

```bash
sudo systemctl edit squid
```

Açılan editöre şunu ekleyin:

```ini
[Service]
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=strict
ReadWritePaths=/var/spool/squid /var/log/squid /var/run
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart squid
sudo systemctl status squid
```

Uyguladıktan sonra sertleştirme seviyesini analiz edin ve servisin hâlâ normal çalıştığını (log yazabildiğini, cache'e erişebildiğini) doğrulayın:

```bash
systemd-analyze security squid
sudo tail -20 /var/log/squid/cache.log
```

> ⚠️ **Dikkat:** `ProtectSystem=strict` dosya sistemini salt-okunur yapar; `ReadWritePaths` içine Squid'in yazması gereken **tüm** dizinleri (cache, log, pid) eklemezseniz servis başlamaz veya sessizce hata verir. Değişiklikten sonra mutlaka `systemctl status squid` ve log kontrolü yapın.

---

### Adım 13: Log Takibi ve Bakım

```bash
# Canlı erişim loglarını izleme
sudo tail -f /var/log/squid/access.log

# Hata loglarını izleme
sudo tail -f /var/log/squid/cache.log
```

Log çıktısı örneği:

```text
1722950400.123   150 185.190.10.45 TCP_TUNNEL/200 5412 CONNECT google.com:443 ahmet_kullanicisi HIER_DIRECT/142.250.185.206 -
```

#### 13.1 Log Rotasyonu Doğrulama

Debian'ın `squid` paketi kurulumla birlikte `/etc/logrotate.d/squid` dosyasını **zaten getirir**, ayrıca oluşturmanıza gerek yok — sadece varlığını doğrulayıp ihtiyacınıza göre ayarlayın:

```bash
# Mevcut yapılandırmayı görüntüle
cat /etc/logrotate.d/squid

# Gerçek rotasyon yapmadan test edin (dry-run)
sudo logrotate -d /etc/logrotate.d/squid
```

Trafiğiniz yoğunsa `rotate` sayısını artırıp `daily` yapabilirsiniz; diski hızla dolduran bir ortamsa `compress` satırının aktif olduğundan emin olun.

---

## 💡 Dikkat Edilmesi Gereken İpuçları & Düzeltmeler

1. **`http_access` Kural Sıralaması:** `allow` kuralları mutlaka `deny all` satırından **ÖNCE** yazılmalıdır, aksi halde 403 Forbidden hatası alırsınız.
2. **`basic_ncsa_auth` Yolu:** Dağıtımınıza göre farklılık gösterebilir. `sudo find / -name basic_ncsa_auth 2>/dev/null` ile doğru yolu bulun.
3. **`Connection Refused` Hatası:** `sudo systemctl status squid` ile servisin çalıştığını ve sunucunuzun güvenlik duvarında (örn. UFW, iptables) 3128 portunun açık olduğunu doğrulayın.
4. **Yapılandırma Doğrulama:** `squid.conf` üzerinde **her** değişiklikten sonra, `reload`/`restart` atmadan önce mutlaka `sudo squid -k parse` ile sözdizimi kontrolü yapın. Bu, `deny all` kuralının yanlışlıkla silinmesi gibi kritik hataları servis yeniden başlamadan önce yakalamanızı sağlar.
5. **Cache dizini ilk kurulum hatası:** `cache_dir` tanımlayıp `squid -z` çalıştırmadan servisi başlatırsanız Squid hata verir veya cache'i kullanmaz — Adım 8'i atlamayın.

*Tebrikler! Debian 13 üzerinde Squid Proxy kurulumunuz başarıyla tamamlanmıştır.*

---
Author: faruk-guler
GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
Page: [www.farukguler.com](https://www.farukguler.com)
