# Squid Proxy Server on Debian 13 (Trixie) Forward Proxy

**Squid**, istemcilerin internete çıkışını düzenleyen yüksek performanslı, açık kaynaklı bir **ileri proxy (forward proxy)**, önbellekleme (caching) ve içerik filtreleme sunucusudur. İçerideki cihazların dışarıya (internete) nasıl çıkacağını kontrol eder, IP adresinizi gizler (anonim proxy) ve yerel ağdaki bant genişliğini optimize eder.

Bu rehber, **Debian 13 (Trixie)** üzerinde Squid Proxy'nin doğrudan paket yöneticisi ile kurulumunu, Basic Auth kimlik doğrulamasını, Elite Proxy (yüksek anonimlik) yapılandırmasını ve güvenlik kurallarını kapsamaktadır.

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

Ayarları uygulayın:

```bash
sudo systemctl reload squid
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

Değişiklikleri uygulayın:

```bash
sudo systemctl restart squid
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

Servisi yeniden başlatın:

```bash
sudo systemctl restart squid
```

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

Yapılandırmayı yeniden yükleyin:

```bash
sudo systemctl reload squid
```

---

### Adım 7: Güvenlik Duvarı Yapılandırması

Kullandığınız güvenlik duvarı çözümünde (UFW, iptables, FortiGate, pfSense vb.) aşağıdaki portlara izin verin:

| Port | Protokol | Açıklama |
|---|---|---|
| **22** | TCP | SSH Uzak Erişim |
| **3128** | TCP | Squid Proxy (varsayılan) |

> 💡 **İpucu:** Proxy portunu (3128) herkese açmak yerine, sadece izin vermek istediğiniz kaynak IP adreslerine kısıtlamanız önerilir.

---

### Adım 8: İstemci Tarafında Proxy Test Etme

#### 8.1 Terminal / cURL ile Test

```bash
# Kullanıcı adı ve parolasız (IP izinli ise)
curl -x http://SUNUCU_IP_ADRESI:3128 https://ifconfig.me

# Kullanıcı adı ve parolalı (Basic Auth ile)
curl -x http://ahmet_kullanicisi:SIFRENIZ@SUNUCU_IP_ADRESI:3128 https://ifconfig.me
```

*(Dönen sonuçta sunucunuzun IP adresi görünüyorsa proxy başarıyla çalışıyor demektir.)*

#### 8.2 Tarayıcıda Kullanma

- **Firefox:** `Ayarlar` > `Ağ Ayarları (Network Settings)` > `Manuel Proxy Yapılandırması`.
  - **HTTP Proxy:** `SUNUCU_IP_ADRESİ` | **Port:** `3128`
  - "Tüm protokoller için bu proxy sunucusunu kullan" seçeneğini işaretleyin.
- **FoxyProxy / SwitchyOmega:** Chrome veya Firefox eklentileri ile tek tıkla proxy geçişi yapabilirsiniz.

---

### Adım 9: İstemci İşletim Sistemi Bazında Proxy Yapılandırması

Tarayıcı dışında, işletim sisteminin veya belirli araçların (apt, git, curl vb.) trafiğini de proxy üzerinden geçirmek isteyebilirsiniz. Aşağıda **Windows** ve **Debian 13 (Trixie)** istemciler için sistem geneli ve araç bazlı yapılandırmalar yer almaktadır.

#### 10.1 Windows İstemci Yapılandırması

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

#### 10.2 Debian 13 (Trixie) İstemci Yapılandırması

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

### Adım 10: Log Takibi ve Bakım

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

---

## 💡 Dikkat Edilmesi Gereken İpuçları & Düzeltmeler

1. **`http_access` Kural Sıralaması:** `allow` kuralları mutlaka `deny all` satırından **ÖNCE** yazılmalıdır, aksi halde 403 Forbidden hatası alırsınız.
2. **`basic_ncsa_auth` Yolu:** Dağıtımınıza göre farklılık gösterebilir. `sudo find / -name basic_ncsa_auth 2>/dev/null` ile doğru yolu bulun.
3. **`Connection Refused` Hatası:** `sudo systemctl status squid` ile servisin çalıştığını ve sunucunuzun güvenlik duvarında (örn. UFW, iptables) 3128 portunun açık olduğunu doğrulayın.
4. **Yapılandırma Doğrulama:** Değişiklik yaptıktan sonra `sudo squid -k parse` komutu ile sözdizimi kontrolü yapabilirsiniz.

*Tebrikler! Debian 13 üzerinde Squid Proxy kurulumunuz başarıyla tamamlanmıştır.*

---
Author: faruk-guler
GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
Page: [www.farukguler.com](https://www.farukguler.com)
