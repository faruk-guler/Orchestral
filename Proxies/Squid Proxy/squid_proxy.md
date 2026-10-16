# Squid Proxy Server on Debian 13 (Trixie) Forward Proxy

**Squid**, istemcilerin internete çıkışını düzenleyen yüksek performanslı, açık kaynaklı bir **ileri proxy (Forward Proxy)**, önbellekleme (caching) ve içerik filtreleme sunucusudur. İçerideki cihazların dışarıya (internete) nasıl çıkacağını kontrol eder, IP adresinizi gizler (anonim proxy) ve yerel ağdaki bant genişliğini optimize eder.

Bu rehber, **Debian 13 (Trixie)** üzerinde (Squid **6.13**) paket yöneticisi ile kurulumu, Basic Auth kimlik doğrulamasını, anonimlik/elite ayarlarını, HTTPS filtreleme (CONNECT ve SSL Bump), cache ve güvenlik sertleştirme yapılandırmasını, istemci tarafı proxy ayarlarını ve şeffaf (tproxy) modu kapsamaktadır.

---

## 📘 Adım Adım Manuel Kurulum

### Adım 1: Sistem Güncellemesi ve Paket Kurulumu (Varyant Seçimi)

**Bu adımda seçtiğin paket, dokümanın geri kalanını belirler.** Squid iki derleme (flavour) hâlinde sunulur ve **ikisi yan yana kurulamaz**.

```bash
# 1.1 Sistem güncellemesi
sudo apt update && sudo apt upgrade -y

# 1.2a Yönlendirme + domain filtreleme + cache + kimlik doğrulama (çoğu senaryo):
sudo apt install -y squid apache2-utils

# 1.2b HTTPS trafiğinin İÇERİĞİNİ de filtrelemek istiyorsan (Adım 7.2 — SSL Bump):
sudo apt install -y squid-openssl apache2-utils

# 1.3 Yardımcı araçlar (ayrı paketlerdir; squid ile gelmez)
sudo apt install -y squidclient squid-purge

# 1.4 Servisi başlat ve açılışa ekle
sudo systemctl enable --now squid
sudo systemctl status squid --no-pager
```

| Paket | Derleme | Yetenekleri | Ne zaman |
|---|---|---|---|
| `squid` | **GnuTLS** | Forward/reverse proxy, `CONNECT` tünelleme, domain filtreleme, cache, auth | Adım 5 ve 6 ile yetineceksen |
| `squid-openssl` | **OpenSSL** | Yukarıdakiler + **SSL Bump (HTTPS interception/decrypt)** | Adım 7.2'ye geçeceksen |

> ⚠️ **SSL Bump için `squid` paketi YETMEZ.** Debian'ın `squid` paketi GnuTLS ile derlenmiştir ve SSL-Bump **içermez**; `http_port ... ssl-bump` satırını eklediğinde `squid -k parse` bu seçeneği reddeder. Tam filtreleme istiyorsan baştan `squid-openssl` kur — `squid` paketini kaldırır, config dosyaların ve ACL'lerin olduğu gibi çalışmaya devam eder.

> 🔎 **Doğrulama:** hangi derlemeyi kullandığını ve SSL desteğini görmek için:
>
> ```bash
> squid -v | tr ' ' '\n' | grep -E 'with-openssl|with-gnutls|enable-ssl|crtd'
> ```
> Çıktıda OpenSSL/`--enable-ssl-crtd` yoksa SSL Bump bölümünü atla. Ayrıca `squid -v | grep -iE 'http-violations'` sorgusu tek başına kanıt değildir (bayrak sürüm çıktısında görünmeyebilir); kesin test için bkz. Adım 5.

> 💡 `full-upgrade` yerine `upgrade` kullan: `full-upgrade` bağımlılık değişimlerinde paket **kaldırabilir** ve bu, `squid`/`squid-openssl` geçişi yapılan bir sunucuda beklenmedik sonuç üretir.

---

### Adım 2: Temel Yapılandırma (`squid.conf`)

Squid'in konfigürasyonu `/etc/squid/squid.conf` dosyasından başlar, ancak Debian paketinde bu dosya **kısa bir ana dosyadır**; gerçek ayarların bir kısmı `include` ile başka dosyalardan gelir. Hangi ayarın nerede tanımlı olduğunu görmeden düzenleme yapma:

```bash
# Hangi dosyalar devrede?
grep -n 'include' /etc/squid/squid.conf
ls -l /etc/squid/conf.d/

# Aktif http_port / cache_dir / ACL / http_access satırlarını topluca gör:
grep -RE '^\s*(http_port|cache_dir|acl |http_access )' /etc/squid/squid.conf /etc/squid/conf.d/

# Değişiklik öncesi yedek
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.bak
sudo nano /etc/squid/squid.conf
```

> 💡 **En temiz yöntem:** kendi eklemelerini `/etc/squid/conf.d/99-kurum.conf` gibi **ayrı bir dosyada** tutmak. Böylece paket güncellemesinde `squid.conf` sana dokunulmamış kalır, `dpkg` conffile sorusu üretmez ve değişikliklerin tek dosyada toplanır. Bu rehberdeki eklemeleri de bu dosyaya yazabilirsin.

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

#### 4.2 `squid.conf` dosyasına Auth kurallarını ekleme

`auth_param` blokları ile ACL tanımlamaları `http_access allow/deny` satırlarından **ÖNCE** gelmelidir (Squid yapılandırmayı yukarıdan aşağıya okur; ACL tanımlanmadan kullanılamaz). Dosyayı açın ve mevcut `http_access` kurallarının **ÜSTÜNE** ekleyin:

```text
# Basic Auth Yardımcı Program Tanımı
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5
auth_param basic realm Squid Proxy Sunucusuna Hoş Geldiniz
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

### Adım 5: Anonimlik Ayarları (Proxy İzi Bırakmama)

Squid varsayılan olarak hedef siteye `Via` başlığı ekler ve istemcinin gerçek adresini `X-Forwarded-For` ile iletir. İki seviye var: **hedefli kapatma** (önerilen) ve **agresif beyaz liste** (elite proxy).

#### 5.1 Önerilen Yöntem — Sızıntı Yapan Başlıkları Tek Tek Kapat

```text
# --- ANONİMLİK ---
forwarded_for off
via off

request_header_access X-Forwarded-For deny all
request_header_access X-Real-IP deny all
request_header_access X-Forwarded-Host deny all
request_header_access Via deny all
request_header_access From deny all
```

`forwarded_for off` zaten `X-Forwarded-For` eklenmesini engeller; `request_header_access` satırları **istemciden gelen** aynı addaki başlıkların da iletilmesini keser — yani ikisi gereksiz değil, farklı şeyi kapatır.

#### 5.2 Agresif Beyaz Liste (yalnızca gerçekten gerekiyorsa)

`request_header_access All deny all` yazdığın anda, **tek tek `allow` vermediğin her başlık hedefe gitmez.** Aşağıdaki liste, resmî Squid dokümanındaki `http_anonymizer paranoid` muadili örnek üzerine kuruldu; oturum açmayı ve büyük POST'ları kırmamak için `Cookie`, `Date`, `Pragma` ve gövde taşıma başlıkları elle açıldı:

```text
# Kimlik ve hedef yönlendirme
request_header_access Host allow all
request_header_access Authorization allow all
request_header_access Proxy-Authorization allow all
request_header_access Cookie allow all              # resmî "paranoid" örneğinde YOKTUR → login kırılır

# İçerik müzakeresi ve gövde taşıma — bunlarsız siteler bozulur
request_header_access Accept allow all
request_header_access Accept-Charset allow all
request_header_access Accept-Encoding allow all
request_header_access Accept-Language allow all
request_header_access Cache-Control allow all
request_header_access Content-Type allow all
request_header_access Content-Length allow all
request_header_access Transfer-Encoding allow all
request_header_access Connection allow all
request_header_access TE allow all
request_header_access Expect allow all
request_header_access Range allow all
request_header_access If-Range allow all
request_header_access Date allow all
request_header_access Pragma allow all
request_header_access If-Modified-Since allow all
request_header_access User-Agent allow all          # buraya bırakmak hedefin istemci parmak izini görmesi içindir;
                                                    # resmî örneklerde anonimlik için deny verilir

# Kalan her şeyi reddet — resmî örneklerde bu satır sonda gösterilir
request_header_access All deny all

# Bilerek allow vermediklerimiz (gitmez): Referer, From, X-Forwarded-For, Via
```

> 💡 **Sıralama değil, ad eşleşmesi belirleyici:** Squid her başlık için önce o başlığın **birebir adına** ait kural setine, sonra `Other` setine (yaygın olmayan adlar), en son `All` setine bakar; bir set içinde ACL'ler normal şekilde değerlendirilir. Yani `All deny all` satırını başa koymak `Accept` iznini etkisiz bırakmaz — yine de resmî örneklerin sırasını (spesifik → `All` sonda) korumak okunabilirlik ve taşınabilirlik açısından doğrudur.

> ⚠️ **Bu direktif Squid 8 dokümanlarında listelenmiyor** (resmî sayfadaki "not available in the v8 version" uyarısı). `--enable-http-violations` da aynı akıbete uğrayabilir; Squid 6'dan 7/8'e yükseltirken 5.1 ve 5.2'deki satırların davranışını tekrar doğrula.

> ⚠️ **`Referer` kapatmak** referer kontrolü yapan siteleri, hotlink korumalı CDN bağlantılarını ve bazı ödeme adımlarını kırar. Kurumsal proxy kullanıcı şikâyetlerinin birinci sebebi budur; kapatmadan önce bir gün `access.log` üzerinden hangi sitelerin kırıldığını gözlemleyin.

> ⚠️ **`request_header_access` ve `reply_header_access` kullanabilmek için Squid'in `--enable-http-violations` ile derlenmiş olması gerekir** (resmî Squid dokümanının "Requires" satırı). Debian paketinde etkin olup olmadığını **varsayma**, şu testle doğrula: satırları ekle ve `sudo squid -k parse` çalıştır — `Unknown setting 'request_header_access'` alırsan derleme bu bayrağı içermiyordur; 5.1'deki `forwarded_for off` + `via off` ile yetin (çoğu senaryoda görünür farkı yaratır).

> 🔎 **Basic Auth kullanıyorsan kritik:** `Proxy-Authorization` başlığını bloklarsan kimlik doğrulama **tamamen durur** ve tüm istekler `407` alır. Beyaz liste kullanırken bu satırın `allow` olduğundan emin ol.

#### 5.3 Doğrulama — Hedef Seni Ne Olarak Görüyor?

Ayarı yaptın, sızmadığını kanıtla:

```bash
# Proxy üzerinden düz HTTP çıkışı yap ve hedefin GÖRDÜĞÜ başlıkları oku
curl -sx KULLANICI:SIFRE@SUNUCU_IP:3128 http://httpbin.org/headers
```

Beklenen: dönen JSON'da `X-Forwarded-For`, `Via`, `From` **bulunmamalı**; hedefin gördüğü tek adres proxy sunucusunun IP'sidir. HTTPS'te bu başlıklar zaten şifreli gideceği için testi düz HTTP hedefle yap.

> 🔒 Anonimlik ≠ gizlilik: hedef site artık **proxy sunucunun IP'sini** görür; sunucuyu yöneten kişi ise `access.log`'daki kullanıcı adı ile kimin kullandığını bilir. Üçüncü taraf bir proxy'yi "anonim" sanmak yaygın hatadır; kendi sunucunda kendi log'una kendini yazdırmış olursun.

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

### Adım 7: HTTPS Trafiğini Filtreleme (CONNECT Filtreleme & SSL Bump)

Squid, varsayılan olarak HTTPS trafiğini sadece tünelleyip (`CONNECT`) içeriğine bakamaz. Domain bazlı engellemeyi HTTPS'e taşımak için iki yöntem vardır:

#### 7.1 Hafif Yöntem — CONNECT İsteğinden Domain Filtreleme (Sertifika Gerekmez)

Forward proxy modunda Squid, istemcinin gönderdiği `CONNECT` isteğindeki domain adını doğrudan okuyabilir. Bu yöntem TLS sertifikası gerektirmez ve çoğu senaryoda yeterlidir:

```text
# CONNECT isteğinden domain'i oku (HTTP ve HTTPS için çalışır)
acl blocked_https_domains dstdomain "/etc/squid/blocked_sites.txt"
http_access deny blocked_https_domains
```

> 💡 **Not:** Adım 6'daki `dstdomain` kuralı HTTPS `CONNECT` hedeflerini zaten kapsar. Bu bölümü **ayrıca** eklemenize gerek yoktur; yalnızca Adım 6'yı kullanmadıysanız veya farklı bir ACL adıyla organize etmek istiyorsanız referans amaçlıdır.

#### 7.2 Tam Yöntem — SSL Bump (MITM, İleri Seviye)

> ⚠️ **Bu bölüm yalnız `squid-openssl` paketi ile çalışır.** Adım 1'de `squid` (GnuTLS derlemesi) kurduysan `ssl-bump` seçeneği tanınmaz ve `squid -k parse` hata verir: `apt install -y squid-openssl` ile geçiş yap (eski paket kaldırılır, config'in korunur). `squid -v` çıktısında `--with-openssl` ve `--enable-ssl-crtd` görmelisin.

Trafiğin içeriğini (URL path, sayfa içeriği) görmek için Squid'in araya girip sertifikayı yeniden imzalaması gerekir. Bunun için kendi kök sertifikanızı üretip **her istemciye** güvenilir kök sertifika olarak yüklemeniz gerekir.

```bash
# Kök sertifika (CA) oluşturma
sudo mkdir -p /etc/squid/ssl_cert
cd /etc/squid/ssl_cert

# Özel anahtar ve sertifikayı ayrı dosyalar olarak oluşturun
sudo openssl req -new -newkey rsa:2048 -sha256 -days 3650 -nodes -x509 \
  -keyout squidCA.key -out squidCA.crt \
  -subj "/C=TR/O=SirketAdi/CN=Squid Proxy CA"

# İzin kısıtlaması (anahtar dosyası sadece root okuyabilmeli)
sudo chmod 600 squidCA.key

# DER formatı (istemcilere dağıtmak için)
sudo openssl x509 -in squidCA.crt -outform DER -out squidCA.der

# Sertifika önbelleği veritabanı (sslcrtd)
sudo /usr/lib/squid/security_file_certgen -c -s /var/spool/squid/ssl_db -M 4MB
sudo chown -R proxy:proxy /var/spool/squid/ssl_db
```

`squid.conf` içine:

```text
# Adım 2'deki temel http_port satırını aşağıdaki ile DEĞİŞTİRİN
# (Aynı portu iki kez tanımlamayın! squid.conf backslash ile satır devamı desteklemez, tek satırda yazın.)
http_port 3128 ssl-bump cert=/etc/squid/ssl_cert/squidCA.crt key=/etc/squid/ssl_cert/squidCA.key generate-host-certificates=on dynamic_cert_mem_cache_size=4MB

sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/spool/squid/ssl_db -M 4MB

acl step1 at_step SslBump1
ssl_bump peek step1
# Not: 'blocked_domains' ACL'i Adım 6'da tanımlanmıştır.
# Adım 6'yı atladıysanız önce şunu ekleyin: acl blocked_domains dstdomain "/etc/squid/blocked_sites.txt"
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
> - Kurumsal olmayan (ev/kişisel) kullanımda genelde gereksizdir; çoğu senaryoda Adım 6'daki `dstdomain` yöntemi yeterlidir.

---

### Adım 8: Cache (Önbellekleme) Yapılandırması

Squid'in adındaki "cache" gerçek bir işlevdir, ama **bugünün internetinde hit oranı düşük çıkar**: HTTPS siteleri `CONNECT` tüneli olarak geçtiği için içerik cache'lenmez (SSL Bump açılsa bile cache yalnızca deşifre edilen bağlantılar için geçerlidir). Kurumsal bir ağda `mgr:info` çıktısında hit oranının **%5-15** bandında olması normaldir; "cache kurdum, trafik neden düşmedi" sorusunun cevabı budur.

Önce paketin ne getirdiğini gör — "disk cache kapalı" varsayımı Debian'da doğru olmayabilir, varsayılanda **küçük bir `cache_dir` tanımı vardır**:

```bash
grep -RE '^\s*cache_dir' /etc/squid/squid.conf /etc/squid/conf.d/
```

```text
# RAM cache (sık erişilen küçük objeler için)
# Öneri: cache_mem değeri toplam RAM'in %25'ini geçmemeli.
# Örneğin 1 GB RAM'li bir sunucuda 256 MB, 4 GB RAM için 512-1024 MB uygundur.
cache_mem 256 MB
maximum_object_size_in_memory 512 KB

# Disk cache: ufs tipi, /var/spool/squid dizini, 10000 MB, 16 alt klasör, 256 alt-alt klasör
cache_dir ufs /var/spool/squid 10000 16 256
maximum_object_size 100 MB
```

```text
# --- Kimliği doğrulanmış kullanıcıların özel yanıtları cache'lenmesin ---
acl authenticated proxy_auth REQUIRED
cache deny authenticated

# --- İzlenebilirlik ve temel kimlik ---
visible_hostname proxy.kurum.local
cache_mgr admin@kurum.local
```

Disk cache dizinleri **ilk kez** oluşturulurken (veya `cache_dir` yolu/boyutu değiştiğinde) Squid durdurulup `-z` ile initialize edilir:

```bash
sudo squid -k parse
sudo systemctl stop squid
sudo squid -z
sudo systemctl start squid

# Cache gerçekten çalışıyor mu?
squidclient -p 3128 mgr:storedir
squidclient -p 3128 mgr:info | grep -iE 'cache log|memory|hit rate|store'
```

> 💡 **Not:** `squid -z` sadece cache dizinleri ilk kez oluşturulduğunda veya `cache_dir` boyutu/yolu değiştiğinde çalıştırılır, her reload'da gerekmez. Bazı Squid sürümleri `ufs` dizinlerini açılışta kendisi oluşturur; bu adımı atladıysan hatayı `cache.log` içinde arar ve `mgr:storedir` çıktısında dizinlerin dolu görünüp görünmediğine bakarsın.

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

> 🔒 **Basic Auth kullanıyorsan port 3128 şifresizdir:** `Proxy-Authorization` başlığı düz HTTP ile taşındığı için kimlik bilgisi ağda okunabilir hâle gelir (Base64 şifreleme değildir). Üç gerçekçi çözüm: (1) istemci ile proxy arasını TLS'e al — `https_port` (bkz. Adım 15), (2) proxy portunu tamamen kapatıp **SSH tüneli** üzerinden kullan — `ssh -L 3128:127.0.0.1:3128 kullanici@SUNUCU_IP`, (3) veya sadece Adım 3'teki IP kısıtıyla çıkışı sınırlandırıp kimlik doğrulamayı bırak. Yalnızca firewall kuralı, şifreli bir kanal değildir.

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

# Invoke-WebRequest için doğrudan parametre (Basic Auth ile DOĞRU kullanım)
$cred = Get-Credential   # Squid'deki kullanıcı adı/parolayı gir
Invoke-WebRequest -Uri "https://ifconfig.me" -Proxy "http://SUNUCU_IP_ADRESI:3128" -Credential $cred
```

> ⚠️ **`-ProxyUseDefaultCredentials` ile çalışmaz.** Bu parametre Windows oturum kimliğini (NTLM/Negotiate) proxy'ye gönderir; Squid ise **Basic** bekler, sonuç `407 Proxy Authentication Required` olur. Kimlik bilgisi gereken proxy'de `-Credential` kullan.

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

# Depoları proxy'den muaf tutmak istersen:
Acquire::http::Proxy::deb.debian.org "DIRECT";
Acquire::https::Proxy::security.debian.org "DIRECT";
```

> ⚠️ **`_apt` sandbox tuzağı (en sık yaşanan sorun):** apt indirmelerini varsayılan olarak **`_apt`** kullanıcısı yapar. Şifreyi gizlemek için `95proxy` dosyasını `chmod 600` yaparsan `_apt` dosyayı okuyamaz ve proxy ayarı **sessizce etkisiz** kalır. İki uygulanabilir yol:
>
> ```bash
> # 1) Dosyayı _apt'in okuyabileceği hâle getir
> sudo chgrp _apt /etc/apt/apt.conf.d/95proxy && sudo chmod 640 /etc/apt/apt.conf.d/95proxy
>
> # 2) veya sandbox kullanıcısını değiştir (güvenlik izolasyonunu zayıflatır, yalnız gerekiyorsa)
> echo 'APT::Sandbox::User "root";' | sudo tee /etc/apt/apt.conf.d/95nosandbox
> ```
>
> **En temizi:** şifreyi config dosyasına gömmek yerine **Adım 3'teki IP kısıtını kullan** — apt için proxy'de kimlik doğrulama şart değil.

```bash
# Ayar gerçekten devrede mi ve çalışan kullanıcı kim?
apt-config dump | grep -iE 'proxy|sandbox'
sudo apt update -o Acquire::http::Proxy="http://SUNUCU_IP_ADRESI:3128" -o Acquire::https::Proxy="http://SUNUCU_IP_ADRESI:3128"
```

> 💡 **İpucu:** Debian 13'ün kendi güncellemelerini bu Squid üzerinden geçirmek istiyorsanız, `deb.debian.org` ve `security.debian.org` gibi adreslerin proxy `http_access` kurallarınızda (Adım 3/6) engellenmediğinden emin olun.

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
curl -x "${http_proxy}" https://ifconfig.me
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
# Squid'in yazması gereken dizinler — eksik bırakılan dizin servisi sessizce çökertir.
# /var/lib/squid = state_dir (auth helper ve SSL Bump cert durumu) için şarttır.
ReadWritePaths=/var/spool/squid /var/log/squid /var/lib/squid /run
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

> ⚠️ **Dikkat:** `ProtectSystem=strict` dosya sistemini salt-okunur yapar; `ReadWritePaths` içine Squid'in yazması gereken **tüm** dizinleri (cache, log, `state_dir`, pid) eklemezsen servis başlamaz veya sessizce hata verir. Değişiklikten sonra mutlaka `systemctl status squid` ve log kontrolü yap.

> ⚠️ **AppArmor'u unutma:** Debian'ın squid paketi AppArmor profili ile gelir (`apparmor` önerilen pakettir). Systemd sertlaştırması ile AppArmor birlikte çalıştığında reddetmeler **`ProtectSystem` yüzünden değil AppArmor yüzünden** olur ve hata mesajı bunu söylemez. Teşhis sırası:
>
> ```bash
> sudo aa-status --enabled && sudo aa-status | grep squid
> sudo journalctl -u squid -n 50 | grep -iE 'permission denied|apparmor|read-only'
> sudo dmesg | grep -i apparmor | tail -20
> ```
> Ret'ler görürsen profili kendi kuralınla genişlet (`/etc/apparmor.d/local/usr.sbin.squid`) ya da sertleştirmeyi geri al (`sudo systemctl revert squid`); çalışmayan bir izolasyon, çalışan bir proxy'den değerli değildir.

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

### Adım 14: Güncelleme

```bash
sudo apt update
sudo apt install --only-upgrade squid

# Yapılandırmayı doğrulayın, sonra servisi yeniden yükleyin
sudo squid -k parse && sudo systemctl reload squid

# Paket güncellemesi binary değişikliği içeriyorsa restart gerekebilir
# sudo systemctl restart squid
```

### Adım 15: İstemci ↔ Proxy Trafiğini Şifreleme (`https_port`)

`http_port` üzerinden bağlanan istemcinin kimlik bilgisi ve istediği URL ağda düz görünür (Adım 9'daki uyarı). `https_port` ile istemci **proxy'nin kendisiyle** TLS konuşur.

> ⚠️ Bu bölüm **Adım 7.2'de üretilmiş bir CA çifti olduğunu varsayar** (`/etc/squid/ssl_cert/squidCA.crt` + `.key`). SSL Bump yapmadıysan önce kendi CA'ını üret (aynı `openssl req -x509` komutu, Adım 7.2) — aksi halde aşağıdaki imzalama adımı "unable to load CA certificate" ile başarısız olur.

```bash
# Proxy sunucu sertifikası: kendi CA'n ile imzala, SAN'a hem DNS hem IP ver
cd /etc/squid/ssl_cert
sudo openssl req -new -newkey rsa:2048 -sha256 -nodes \
  -keyout proxy.key -out proxy.csr -subj "/CN=proxy.kurum.local"
sudo openssl x509 -req -in proxy.csr -CA squidCA.crt -CAkey squidCA.key -CAcreateserial \
  -days 825 -sha256 -extfile <(echo "subjectAltName=DNS:proxy.kurum.local,IP:10.0.0.5") -out proxy.crt
sudo chmod 600 proxy.key
```

`squid.conf` içine (tek satır — Squid backslash ile satır devamını desteklemez):

```text
https_port 3129 tls-cert=/etc/squid/ssl_cert/proxy.crt tls-key=/etc/squid/ssl_cert/proxy.key
```

> 💡 **Söz dizimi sürüm notu:** Squid 4.0.15 öncesinde `cert=`/`key=` yazılıyordu; güncel adlar `tls-cert=`/`tls-key=`. Hangisinin tanındığını `sudo squid -k parse` ile doğrula.

```bash
# Doğrulama: proxy'nin kendisiyle TLS konuş
sudo squid -k parse && sudo systemctl restart squid
curl -x https://proxy.kurum.local:3129 --proxy-cacert /etc/squid/ssl_cert/squidCA.crt https://ifconfig.me
```

> ⚠️ **Tarayıcı desteği sınırlıdır:** `curl`, `apt`, `git` ve pek çok kurumsal araç HTTPS proxy sunucusunu destekler; tarayıcıların önemli bir kısmı desteklemez. Bu yüzden **genel kullanıcı çıkışı için pratik çözüm hâlâ ya IP kısıtı ya da SSH tünelidir** (`ssh -L 3128:127.0.0.1:3128 kullanici@SUNUCU_IP`); `https_port`'u araç/otomasyon trafiği için kullan.

---

### Adım 16: Şeffaf (Transparent) Proxy Modu — `intercept` ve `tproxy`

Şeffaf modda istemcinde proxy ayarı **yapılmaz**; trafiği router/Netfilter Squid'e yönlendirir. Squid 6'da iki mod var (eski kurulumlarda gördüğün `transparent` yazımı güncel dokümanda mod olarak geçmez):

| Mod | Ne yapar | Ek gereksinim |
|---|---|---|
| `intercept` | NAT/REDIRECT ile gelen trafiği kabul eder | `iptables -t nat ... REDIRECT` veya `nft dnat` |
| `tproxy` | Linux TPROXY ile dinler ve **istemci IP'siyle** arkaya çıkar | `ip rule` + `ip route local` + `TPROXY` zinciri, `ip_forward` |

> ⚠️ **Her iki modda da o portta kimlik doğrulama DEVRE DIŞI kalır** (resmî Squid dokümanı: "disables authentication on the port"). Yani Adım 4'teki Basic Auth ile şeffaf mod **aynı portta çalışmaz**; ya ikinci bir `http_port` tanımla (3128 explicit + 3129 transparent) ya da şeffaf modda kullanıcıyı IP ile tanı (Adım 3).

**16.1 `intercept` (port 80 için en basit yol)**

```text
http_port 3128 intercept
acl localnet src 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
http_access allow localnet
http_access deny all
```

> ⚠️ Bu blok resmî/varsayılan `squid.conf` kurallarının **yerine geçmez**, tamamlayıcısıdır. Varsayılandaki `SSL_ports`, `Safe_ports`, `manager` ve `localhost` ACL'lerini silersen (Adım 3'teki uyarı) her porta tünelleyebilen, cache-manager'ı herkese açık bir yapıya dönersin. Şeffaf modda istemci tam URL göndermez (`GET /path` + `Host` eksik olabilir); "595 ERR_TRANSFER_FAIL / invalid request" görürsen Squid'in `http_port ... intercept` ile dinlediğini ve `allow direct` kuralının olduğunu doğrula.

```bash
# Yönlendirme (test ağında dene; SSH portunu KESİNLİKLE yönlendirme)
sudo iptables -t nat -A PREROUTING -i br0 -p tcp --dport 80 -j REDIRECT --to-port 3128
```

**16.2 `tproxy` (istemci IP'sini korumak ve 443'ü yakalamak için)**

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo ip rule add fwmark 0x1 table 100 pref 100
sudo ip route add local default dev lo table 100
sudo iptables -t mangle -A PREROUTING -p tcp --dport 80 -j TPROXY \
     --on-ip 0.0.0.0 --on-port 3128 --tproxy-mark 0x1/0x1
```

`squid.conf`: `http_port 3128 tproxy`

> ⚠️ **Derleme ve yetki koşulu:** `tproxy` için Squid'in **netfilter desteğiyle** (`--enable-linux-netfilter`) derlenmiş olması ve sürecin `CAP_NET_ADMIN` yetkisine sahip olması gerekir. Kontrol et:
>
> ```bash
> squid -v | tr ' ' '\n' | grep -i netfilter
> ```
> Çıktı boşsa `intercept`/`tproxy` hiç çalışmaz; paket derlemeni gözden geçir. 443'ü de yakalamak istersen aynı zincire `--dport 443` satırını ekle — ve hatırlatma: şifreli içeriği görmek istiyorsan bu tek başına yetmez, Adım 7.2 (SSL Bump) gerekir.

> ⚠️ **HTTPS'i şeffaf şekilde yakalamak** bu kadarla bitmez: istemci CONNECT göndermediği için hedef adresi kernel'den okumak gerekir ve içeriği görmek **Adım 7.2 (SSL Bump + `squid-openssl`)** şarttır. Kurumsal senaryoda bu işi Squid yerine router'ın (pfSense/FortiGate) yapması ve Squid'i sadece filtreleme/kimlik katmanı olarak kullanması daha yaygındır.

> 💡 Kalıcılık: `iptables` kuralları sunucu yeniden başlatıldığında **silinir**; `iptables-persistent` (`netfilter-persistent save`) veya `nftables` service'i ile kalıcı yap. Testleri yalnızca **SSH üzerinden bağlıyken** yap — yanlış bir mangle/nat kuralı tüm çıkışı kestiğinde masaya fiziksel erişimle oturmak zorunda kalırsın.

---

### Adım 17: Squid'i Reverse Proxy (Accelerator) Olarak Kullanma

Squid sadece çıkış (forward) için değildir; `accel` modu ile öne konur ve arkadaki uygulamayı cache'ler:

```text
http_port 80 accel defaultsite=panel.alanadiniz.com vhost

# Arkadaki gerçek uygulama (origin)
cache_peer 127.0.0.1 parent 8080 0 no-query originserver name=panelAccel

acl our_sites dstdomain panel.alanadiniz.com
http_access allow our_sites
cache_peer_access panelAccel allow our_sites

# Vary başlığını ciddiye al (acceleratorda şart)
cache_vary on

# Varsayılanı kapat
http_access deny all
```

> 💡 Aynı makinede forward ve accelerator portları ayrı tanımlanabilir (ör. `http_port 3128` forward, `http_port 80 accel` reverse). Accelerator'ı **TLS ile** çalıştırmak istersen `http_port` yerine `https_port` kullan: `https_port 443 accel defaultsite=panel.alanadiniz.com vhost tls-cert=/etc/squid/ssl_cert/proxy.crt tls-key=/etc/squid/ssl_cert/proxy.key`. Ancak **dışarıya dönük reverse proxy işi için nginx/HAProxy/NPM daha doğru araçtır**: TLS yönetimi (sertifika yenileme), WebSocket/upgrade başlıkları, header kontrolü ve health check orada çok daha olgun. Squid'in reverse modda sunduğu asıl değer **HTTP cache**'idir (aynı statik içeriği defalarca origin'e sormamak).

---

### Adım 18: Bant Genişliği Sınırlama (Delay Pools)

> ⚠️ **Derleme koşulu:** Delay pools `--enable-delay-pools` ile derlenmiş Squid'de çalışır (resmî dokümanın "Requires" satırı). Debian paketinde etkin olup olmadığını kontrol et:
>
> ```bash
> squid -v | tr ' ' '\n' | grep -i delay-pools
> ```
> Çıktı boşsa bu bölüm çalışmaz; sınırlandırmayı kernel tarafına (nftables/tc) taşı.

```text
acl limited_users src 192.168.10.0/24

delay_pools 1
delay_class 1 2
delay_access 1 allow limited_users
delay_access 1 deny all
# aggregate: 64 KB/s havuz | bireysel: 8 KB/s beklenen / 12 KB/s anlık
delay_parameters 1 64000/64000 8000/12000
delay_histogram 1
```

> 💡 Delay pool'lar kaba bir sınırdır ve sadece HTTP/CONNECT bağlantı hızını etkiler; gerçek QoS/kuyruk yönetimi gerekiyorsa kernel tarafı (tc/nftables) daha doğru araçtır. `squidclient -p 3128 mgr:DelayPools` ile anlık durumu gör.

---

### Adım 19: Yedekleme ve Geri Yükleme

```bash
sudo mkdir -p /backup && sudo chmod 700 /backup
TS=$(date +%F)

# Config + kullanıcı dosyaları + CA/anahtarlar. /var/spool altındaki cache ve
# ssl_db cert cache'i BİLİNÇLİ olarak yedeklenmez; geri yüklemede yeniden üretilir.
sudo tar -czf /backup/squid_$TS.tar.gz \
  --exclude='etc/squid/squid.conf.bak' \
  -C / etc/squid

# Hangi varyantın kurulu olduğunu da belgeye yaz (squid mi squid-openssl mu)
dpkg -l | grep -E '^ii\s+squid' | sudo tee /backup/squid_paketler_$TS.txt
```

```bash
# Geri yükleme
sudo tar -xzf /backup/squid_<TARIH>.tar.gz -C /
sudo squid -k parse && sudo systemctl restart squid

# Cache veritabanı ve SSL Bump cert cache'ini yeniden üret
sudo squid -z
sudo /usr/lib/squid/security_file_certgen -c -s /var/spool/squid/ssl_db -M 4MB
```

> 🔒 `/etc/squid/passwd` ve `ssl_cert/*.key` özel anahtarları arşivdedir; arşivi sunucu dışına çıkaracaksan şifrele (`gpg -c`). `chmod 700` yalnızca yerel izindir, şifreleme değildir.

---

## 📊 Squid Ne Zaman Doğru Araç Değildir?

| İhtiyaç | Daha iyi araç | Neden |
|---|---|---|
| Domain→servis yönlendirme, TLS sonlandırma, çok site yayıncılığı | nginx / NPM / Caddy | Config basit, LE entegrasyonu olgun, header/WebSocket davranışı net |
| Çok kopyalı servise L4/L7 dağıtım, health check, drain | HAProxy | Load balancing birinci sınıf özellik; `mode tcp` + health/agent check |
| Non-HTTP protokolleri yönlendirme (SSH, DB, MQTT) | nginx `stream` / HAProxy | Squid HTTP tabanlıdır; generic TCP proxy değildir |
| Paket seviyesinde şeffaf denetim/VPN tarzı çıkış | router + firewall / ağ katmanı | Squid L7 HTTP vekili; L3/L4 ihlali değil |

Squid'in haklı olduğu yer: **egress kontrolü + kullanıcı bazlı log + cache + içerik filtreleme** gerektiren kurumsal/on-site çıkış katmanı.

---

## 💡 Sık Karşılaşılan Hatalar ve Çözümleri

1. **`http_access` Kural Sıralaması:** `allow` kuralları mutlaka `deny all` satırından **ÖNCE** yazılmalıdır, aksi halde 403 Forbidden alırsın. (`request_header_access` bundan farklı işler: orada belirleyici olan satır sırası değil, başlık adının hangi kural setiyle eşleştiğidir — bkz. 5.2.)
2. **`basic_ncsa_auth` Yolu:** Debian 13'te `/usr/lib/squid/basic_ncsa_auth`. Emin değilsen: `ls /usr/lib/squid/ | grep ncsa` veya `sudo find / -name basic_ncsa_auth 2>/dev/null`.
3. **`Connection Refused`:** Üç şeyi sırayla kontrol et — servis ayakta mı (`systemctl is-active squid`), port dinleniyor mu (`ss -tlnp | grep 3128`), firewall kaynağa izin veriyor mu (`sudo ufw status numbered`). Port dinlenmiyorsa sebep genelde config hatasıdır; `cache.log` son satırlarına bak.
4. **Yapılandırma Doğrulama:** `squid.conf` üzerinde **her** değişiklikten sonra, `reload`/`restart` atmadan önce `sudo squid -k parse` çalıştır. Bu, `deny all` satırının yanlışlıkla silinmesi gibi kritik hataları servis düşmeden yakalar.
5. **Cache dizini ilk kurulum:** `cache_dir` ekleyip dizinleri oluşturmadan servisi başlatırsan cache çalışmaz (bazı sürümler `ufs` dizinlerini kendisi açar). `squid -z` + `squidclient mgr:storedir` ikilisi sorunu kesin çözer.
6. **`squidclient` ayrı pakettir:** Debian'da `squid` paketi `squidclient`'ı **içermez**; `command not found` görürsen `sudo apt install squidclient`. Cache temizleme aracı `squid-purge`, Prometheus metrikleri için `prometheus-squid-exporter` de ayrı pakettir.
7. **PURGE isteği 403 dönüyor:** `squid-purge` kullanabilmek için ACL gerekir (yalnız yönetim hostuna aç):

   ```text
   acl purge method PURGE
   http_access allow purge localhost
   http_access deny purge
   ```

8. **SSL Bump satırı tanınmıyor / `Parse Config File` hatası:** `squid` (GnuTLS) paketi kurulu demektir. `apt install squid-openssl` ile geçiş yap ve `squid -v | grep -i openssl` ile doğrula (Adım 1 ve 7.2).
9. **Şeffaf modda kullanıcı adı sorulmuyor:** `intercept` ve `tproxy` portlarında **kimlik doğrulama devre dışıdır** (Adım 16). Kullanıcı bazlı log istiyorsan şeffaf mod yerine explicit proxy + Basic Auth kullan.
10. **"Cache kurdum, hit oranı %2":** Beklenen bu. Trafiğin çoğu HTTPS `CONNECT` olduğundan cache'lenemez (Adım 8). Hit oranını gerçekten yükseltmek istiyorsan hedef, SSL Bump açmak veya static/accelerator modu (Adım 17) gibi **şifresiz/içerik görülebilir** trafiği çoğaltmaktır.
11. **Siteler bozuk açılıyor / login olmuyor:** 5.2'deki beyaz listede `Cookie`, `Accept*`, `Content-Length` veya `Host` eksikse bu belirtileri görürsün. `Referer` kapatmak da hotlink korumalı içerikleri kırar. Şikâyet gelince beyaz listeyi tek bir `allow` satırı ekleyerek daralt; tüm listeyi bir anda kapatma.
12. **3128'e kaba kuvvet denemeleri:** Basic Auth açık ve port internete görünüyorsa `access.log`'da çok sayıda `TCP_DENIED/407` satırı birikir. Hazır bir fail2ban filteri olmayabilir; ya log satırına özel bir jail/filter yaz, ya (tercih edilen) Adım 3'teki IP kısıtını koruyup portu tünelden/SSH üzerinden kullandır.
13. **Yedek tek dosya değil:** `squid.conf` + `conf.d/` + `passwd` + `ssl_cert/` birlikte alınmalı; `ssl_db` ve cache veritabanı yeniden üretilir (Adım 19).

> 📌 **Sürüm notu:** Bu rehber Debian 13 (trixie) ve **Squid 6.13** esas alınarak hazırlanmıştır. Paket iki derleme sunar (`squid` GnuTLS / `squid-openssl` OpenSSL); kimi seçenekler (`ssl-bump`, `tls-cert=`, `request_header_access`) derlemeye ve sürüme göre değişir. Bir satır `squid -k parse`'ta tanınmıyorsa sebebi sürüm/derlemedir, senin yazımın değil.

*Tebrikler! Debian 13 üzerinde Squid Proxy kurulumunuz ve yapılandırmanız başarıyla tamamlanmıştır.*

---

- Author: faruk-guler
- GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
- Page: [www.farukguler.com](https://www.farukguler.com)
