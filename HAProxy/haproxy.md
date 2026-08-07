# HAProxy on Debian 13 (Trixie) Load Balancer & Reverse Proxy

**HAProxy** (High Availability Proxy), yüksek performanslı, açık kaynaklı bir **yük dengeleyici (load balancer)** ve **ters proxy (Reverse Proxy)** çözümüdür. Gelen trafiği birden fazla backend sunucuya dağıtarak yüksek erişilebilirlik (high availability), ölçeklenebilirlik ve hata toleransı sağlar. Özellikle web sunucuları, API gateway'leri ve veritabanı kümeleri önünde kurumsal ortamlarda yaygın olarak kullanılır.

Bu rehber, **Debian 13 (Trixie)** üzerinde HAProxy'nin paket yöneticisi ile kurulumunu, HTTP/HTTPS yük dengeleme yapılandırmasını, SSL sonlandırmasını (SSL termination), sağlık kontrollerini (health checks) ve güvenlik pratiklerini kapsamaktadır.

---

## 📘 Adım Adım Manuel Kurulum

### Adım 1: Sistem Güncellemeleri ve Paket Kurulumu

Sistem paketlerinizi güncelleyin ve HAProxy'yi yükleyin:

```bash
# 1.1 Sistem güncellemesi
sudo apt update && sudo apt full-upgrade -y

# 1.2 HAProxy'yi yükleyin
sudo apt install -y haproxy nano

# 1.3 Sürümü doğrulayın
haproxy -v

# 1.4 Servisi başlatın ve otomatik açılışa ekleyin
sudo systemctl enable --now haproxy
sudo systemctl status haproxy
```

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

    # TLS/SSL Güvenlik Sertleştirmeleri
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

#---------------------------------------------------------------------
# Varsayılan Ayarlar
#---------------------------------------------------------------------
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http
```

---

### Adım 3: HTTP Yük Dengeleme (Load Balancing)

Aşağıdaki örnek senaryo, gelen HTTP trafiğini `192.168.1.10`, `192.168.1.11` ve `192.168.1.12` adreslerindeki üç web sunucusuna eşit olarak dağıtır:

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
    option httpchk GET /health
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

# Aktif bağlantıları kesmeden yeniden yükleme
sudo systemctl reload haproxy
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
sudo cat domain.crt intermediate.crt domain.key | sudo tee /etc/haproxy/certs/kurumsal.pem

# 2. Dosya izinlerini kısıtlayın (güvenlik için sadece root okuyabilsin)
sudo chmod 600 /etc/haproxy/certs/kurumsal.pem

# 3. (İsteğe Bağlı) Yerel CA Kök Sertifikasını Debian Sistemine Tanıtma
# HAProxy veya backend servislerinizin yerel CA'yı tanıması için:
sudo cp local-ca.crt /usr/local/share/ca-certificates/local-ca.crt
sudo update-ca-certificates
```

#### 4.2 HTTPS Frontend Yapılandırması

```text
frontend https_frontend
    bind *:443 ssl crt /etc/haproxy/certs/kurumsal.pem
    # Güvenlik başlıkları ekle
    http-response set-header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    default_backend web_servers

frontend http_frontend
    bind *:80
    # HTTP isteklerini HTTPS'e kalıcı olarak yönlendir
    http-request redirect scheme https code 301
```

---

### Adım 5: Sağlık Kontrolleri (Health Checks)

HAProxy, backend sunucuları düzenli aralıklarla kontrol eder. Erişilemeyen sunucuyu otomatik olarak devre dışı bırakır:

```text
backend web_servers
    balance roundrobin
    option httpchk GET /health "HTTP/1.1\r\nHost: localhost"

    # inter: kontrol aralığı | rise: sağlıklı sayılmak için başarı sayısı | fall: devre dışı için başarısızlık sayısı
    server web1 192.168.1.10:80 check inter 3s rise 2 fall 3
    server web2 192.168.1.11:80 check inter 3s rise 2 fall 3
    server web3 192.168.1.12:80 check inter 3s rise 2 fall 3 backup
```

> 💡 **`backup` parametresi:** `web3` yalnızca `web1` ve `web2` devre dışı kaldığında devreye girer. Yedek sunucu (failover) senaryoları için kullanın.

---

### Adım 6: İstatistik Sayfası (Stats Dashboard)

HAProxy, yerleşik bir izleme arayüzü sunar. `haproxy.cfg` dosyasına ekleyin:

```text
listen stats
    bind *:8404
    stats enable
    stats uri /haproxy-stats
    stats realm HAProxy\ Statistics
    stats auth admin:GUCLU_SIFRE_BURAYA
    stats refresh 10s
    stats show-node
    stats show-legends
```

Servisi yeniden yükledikten sonra `http://SUNUCU_IP_ADRESI:8404/haproxy-stats` adresinden erişebilirsiniz.

> ⚠️ **Stats sayfasını (8404) dış dünyaya açmayın.** Güvenlik duvarınızda bu porta sadece yönetim ağınızdan erişime izin verin.

---

### Adım 7: Güvenlik Duvarı Yapılandırması

Kullandığınız güvenlik duvarı çözümünde (UFW, iptables, FortiGate, pfSense vb.) aşağıdaki portlara izin verin:

| Port | Protokol | Açıklama |
|---|---|---|
| **22** | TCP | SSH Uzak Erişim |
| **80** | TCP | HTTP Trafiği |
| **443** | TCP | HTTPS Trafiği |

> ⚠️ **Port 8404 (Stats Sayfası):** Dış dünyaya açmayın. Sadece yönetim ağınızdan erişime izin verin.

---

### Adım 8: Log Takibi ve Bakım

```bash
# Canlı HAProxy loglarını izleme (systemd-journald — Debian 13 Varsayılanı)
sudo journalctl -u haproxy -f

# (Alternatif) rsyslog kurulu ise log dosyasından izleme
sudo tail -f /var/log/haproxy.log

# Servis durumu
sudo systemctl status haproxy

# Yapılandırma değişikliği sonrası — aktif bağlantıları kesmeden yeniden yükleme
sudo systemctl reload haproxy
```

---

### Adım 9: Güncelleme

```bash
sudo apt update
sudo apt install --only-upgrade haproxy
sudo systemctl reload haproxy
```

---

## 💡 Dikkat Edilmesi Gereken İpuçları & Düzeltmeler

1. **Yapılandırma Doğrulama:** Her değişiklikten önce `sudo haproxy -c -f /etc/haproxy/haproxy.cfg` komutu ile sözdizimi kontrolü yapın. Hatalı bir config servisi durdurabilir.
2. **`reload` vs `restart`:** `reload`, aktif bağlantıları kesmeden yeni yapılandırmayı yükler. Production ortamında her zaman `restart` yerine `reload` tercih edin.
3. **`502 Bad Gateway` Hatası:** Backend sunucunun çalıştığını ve HAProxy'nin ona ağ üzerinden erişebildiğini doğrulayın. `curl -v http://BACKEND_IP:80` ile bağlantı testi yapabilirsiniz.
4. **Oturum Sürekliliği (Session Persistence):** Kullanıcı oturumlarının hep aynı sunucuya gitmesi gerekiyorsa (örn. oturum bilgisi sunucuda tutuluyorsa), backend bloğuna `cookie SERVERID insert indirect nocache` satırını ekleyin.
5. **`/health` Endpoint:** Sağlık kontrolünün çalışması için backend uygulamalarınızda `GET /health` isteğine `200 OK` dönen bir endpoint bulunmalıdır.
6. **Kurumsal SSL Sertifikası Yenileme:** Kurumunuzun iç CA'i veya SSL sağlayıcısı sertifikayı yenilediğinde, yeni `.crt` ve `.key` dosyalarını aynı sırayla tekrar birleştirip (`cat domain.crt intermediate.crt domain.key > /etc/haproxy/certs/kurumsal.pem`) ardından `sudo systemctl reload haproxy` çalıştırmanız yeterlidir. Servis kesintisi yaşanmaz.

*Debian 13 üzerinde HAProxy kurulumunuz ve yapılandırmanız başarıyla tamamlanmıştır.*

---
Author: faruk-guler
GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
Page: [www.farukguler.com](https://www.farukguler.com)
