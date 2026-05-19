# SeaweedFS (High Availability) Kurulum Rehberi

<img src="https://raw.githubusercontent.com/seaweedfs/seaweedfs/master/note/seaweedfs.png" width="300" height="300">

---

## 🧬 SeaweedFS Mimari ve Best Practice Yaklaşımı

Bu kurulum **Hiper-Bütünleşik (Hyper-Converged)** mimariyi temel alır. 3 sunuculu kümelerde Yüksek Erişilebilirlik (HA) sağlamak için **Master servisi 3 sunucuda birden çalıştırılır.**

Raft protokolü gereği, master sunucular aralarında oylama yaparak bir "Lider" seçerler. Sunuculardan biri fiziksel olarak çökse dahi, kalan 2 sunucu yeni lideri seçerek **sıfır kesintiyle** çalışmaya devam eder.

| IP Adresi | Hostname | Rol | Servisler |
|---|---|---|---|
| `192.168.47.21` | node1 | **Master + Volume + Filer** | master, volume, filer (S3) |
| `192.168.47.23` | node2 | **Master + Volume** | master, volume |
| `192.168.47.25` | node3 | **Master + Volume** | master, volume |

> [!WARNING]
> **Filer (S3) Tek Nokta Hatası (SPOF) ve Yedekleme Uyarısı:**
> Bu kurulumda Filer ve S3 servisleri yerel `leveldb2` veritabanı motoru kullanılarak sadece **Node 1** üzerinde çalışmaktadır.
> Node 1 fiziksel olarak çökerse, Master ve Volume katmanları (veri bütünlüğü) korunsa dahi, **Filer ve S3 API'leriniz çevrimdışı kalır.**
>
> **Filer Katmanını Yüksek Erişilebilir (HA) Yapmak İçin:**
> Filer servisini de Yüksek Erişilebilir hale getirmek ve SPOF'u ortadan kaldırmak için:
>
> 1. Filer servisini tüm düğümlerde (Node 1, 2 ve 3) çalıştırın.
> 2. Yerel `leveldb2` yerine tüm Filer düğümlerinin bağlanacağı ortak/dağıtık bir veritabanı (örn. **PostgreSQL**, **MySQL** veya **Redis**) yapılandırın. Bunun için `/etc/seaweedfs/filer.toml` dosyasını oluşturup ilgili veritabanı bağlantı bilgilerini tanımlamanız gerekir.
>
> **Kritik Tavsiye (LevelDB2 kullanılıyorsa Yedekleme):**
> Eğer LevelDB2 kullanmaya devam edecekseniz, tüm dosya yolları ve klasör yapısı Node 1 üzerindeki `/data/filer` dizininde saklanır. Veri kaybı yaşamamak için bu dizini düzenli aralıklarla (örneğin günlük yedekleme scriptleri ile) başka bir sunucuya yedeklemeniz önemle tavsiye edilir.

---

## 📌 Gereksinimler

- En az **3 adet** Linux sanal makine.
- Veri diskleri **LVM** ile yapılandırılmış ve **Thick Provision** (tam boyut ayrılmış) olmalıdır.
- Node'ların farklı fiziksel sunucularda çalışmasını garanti eden **Anti-Affinity** kuralı tanımlanmalıdır.
- **Disk Kapasite Planlaması:** Volume sunucularında tanımlanan `-max` parametresi (şablonda 100), her bir hacmin maksimum boyutu 32GB olacağından, kullanılabilir disk alanına göre ayarlanmalıdır. Örneğin, 1 TB disk alanı için `-max=30` (`30 * 32GB = 960GB`) olarak yapılandırılmalıdır.

---

## 🚀 Kurulum Adımları

### 1. Binary Kurulumu (Tüm Düğümlerde)

```bash
cd /tmp
wget https://github.com/seaweedfs/seaweedfs/releases/latest/download/linux_amd64.tar.gz
tar -xzf linux_amd64.tar.gz
sudo mv weed /usr/local/bin/
weed version
```

### 2. Kullanıcı ve Dizin Yapılandırması

```bash
# Sistem kullanıcısı (Tüm Düğümlerde):
sudo useradd -r -s /bin/false seaweedfs

# Node 1 Dizin Yapısı (Master + Filer + Volume):
sudo mkdir -p /data/master /data/volumes /data/filer
sudo chown -R seaweedfs:seaweedfs /data/master /data/volumes /data/filer

# Node 2 ve Node 3 Dizin Yapısı (Master + Volume):
sudo mkdir -p /data/master /data/volumes
sudo chown -R seaweedfs:seaweedfs /data/master /data/volumes
```

> [!IMPORTANT]
> Servisleri başlatmadan önce disklerin `/etc/fstab` üzerinden başarıyla mount edildiğinden emin olun.

---

## ⚙ Sistem Servis ve Yapılandırma Şablonları

Yönetimi kolaylaştırmak için tüm servis ve yapılandırma dosyaları **ortak şablonlar (template)** halinde hazırlanmıştır:

```text
proje-kök-dizini/
├── security/                 # Güvenlik ve Kimlik Doğrulama Şablonları (Klasör)
│   ├── security.toml         # JWT ve gRPC mTLS Ayarları
│   └── s3.json               # S3 API Yetkileri ve Kullanıcı Bilgileri
└── services/                 # Sistem Servis Şablonları (Klasör)
    ├── seaweedfs-master.service  # Master Servisi
    ├── seaweedfs-volume.service  # Volume Servisi
    └── seaweedfs-filer.service   # Filer Servisi
```

### Servislerin Kurulumu ve IP Yapılandırması (Manuel Yöntem)

Servis şablon dosyaları (`services/seaweedfs-*.service`) doğrudan sunuculardaki hedeflerine yerleştirildikten sonra, dosya içerisindeki `-ip=<SUNUCU_IP_ADRESI>` parametresi el ile o sunucunun kendi IP adresiyle değiştirilir.

#### 1. Servis Dosyalarını Kopyalayın ve IP Adreslerini Düzenleyin

Şablon dosyalarını sunuculardaki hedeflere kopyalayın:

| Servis | Sunucu | Hedef Dosya Yolu | Şablon Kaynağı |
|---|---|---|---|
| **Master** | Node 1, Node 2, Node 3 | `/etc/systemd/system/seaweedfs-master.service` | `services/seaweedfs-master.service` |
| **Volume** | Node 1, Node 2, Node 3 | `/etc/systemd/system/seaweedfs-volume.service` | `services/seaweedfs-volume.service` |
| **Filer (S3)** | Sadece Node 1 | `/etc/systemd/system/seaweedfs-filer.service` | `services/seaweedfs-filer.service` |

Kopyalama işleminin ardından her sunucudaki servis dosyalarını açarak `-ip=<SUNUCU_IP_ADRESI>` alanını güncelleyin:

- **Node 1 (`192.168.47.21`) üzerinde:** Master, Volume ve Filer servislerinde `-ip=192.168.47.21` olarak ayarlayın.
- **Node 2 (`192.168.47.23`) üzerinde:** Master ve Volume servislerinde `-ip=192.168.47.23` olarak ayarlayın.
- **Node 3 (`192.168.47.25`) üzerinde:** Master ve Volume servislerinde `-ip=192.168.47.25` olarak ayarlayın.

#### 2. Servisleri Başlatın

**Node 1 üzerinde:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now seaweedfs-master seaweedfs-volume seaweedfs-filer
sudo systemctl status seaweedfs-master seaweedfs-volume seaweedfs-filer --no-pager
```

**Node 2 ve Node 3 üzerinde:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now seaweedfs-master seaweedfs-volume
sudo systemctl status seaweedfs-master seaweedfs-volume --no-pager
```

---

## 🛡️ Güvenlik ve Kimlik Doğrulama (Hardening)

### 1. JWT ve S3 Kimlik Doğrulama Yapılandırması

- Tüm düğümlerde `/etc/seaweedfs` dizinini oluşturun.
- `security/security.toml` (JWT/mTLS) şablonunu **Tüm Düğümlerde** `/etc/seaweedfs/` dizini altına kopyalayın.
- `security/s3.json` (S3 IAM) şablonunu **Sadece Node 1** üzerinde `/etc/seaweedfs/` dizini altına kopyalayın.
- Dizin sahipliğini `seaweedfs` kullanıcısına atayıp yetkileri kısıtlayın:

**Node 1 üzerinde:**

```bash
sudo mkdir -p /etc/seaweedfs
# Dosyaları yerleştirdikten sonra:
sudo chown -R seaweedfs:seaweedfs /etc/seaweedfs
sudo chmod 600 /etc/seaweedfs/security.toml /etc/seaweedfs/s3.json
```

**Node 2 ve Node 3 üzerinde:**

```bash
sudo mkdir -p /etc/seaweedfs
# Dosyaları yerleştirdikten sonra:
sudo chown -R seaweedfs:seaweedfs /etc/seaweedfs
sudo chmod 600 /etc/seaweedfs/security.toml
```

### 2. Sunucular Arası gRPC Şifreleme (mTLS)

Sertifikalarınızı `/etc/seaweedfs/tls/` dizinine yerleştirin ve özel anahtar (private key) izinlerini kısıtlayın:

```bash
# Tüm Düğümlerde:
sudo mkdir -p /etc/seaweedfs/tls
# Sertifikaları yerleştirdikten sonra izinleri düzenleyin:
sudo chown -R seaweedfs:seaweedfs /etc/seaweedfs/tls
sudo chmod 600 /etc/seaweedfs/tls/*.key
```

> [!WARNING]
> **Kritik SAN (Subject Alternative Name) Sertifika Gereksinimi:**
> gRPC TLS protokolü, hostname ve IP doğrulamasını sıkı şekilde uygular. Eğer 3 sunucuya da birebir aynı (örneğin sadece `node1` IP'si için üretilmiş) sertifikayı kopyalarsanız, sunucular birbirine gRPC ile bağlanırken **TLS handshake hatası** alırsınız.
>
> **Çözüm Seçenekleri:**
>
> 1. **Ortak Sertifika Kullanımı:** Tek bir sertifika üretip **SAN (Subject Alternative Name)** alanına tüm sunucu IP adreslerini (`192.168.47.21`, `192.168.47.23`, `192.168.47.25`) ekleyin ve bu ortak sertifikayı tüm düğümlere kopyalayın.
> 2. **Düğüme Özel Sertifika Kullanımı:** Her node için kendi IP'sine özel ayrı sertifika çifti üretin. Bu durumda, `security.toml` içerisindeki dosya yollarının (`master.crt`, `volume.crt` vb.) aynı kalması için her sunucudaki `/etc/seaweedfs/tls/` altına o sunucuya ait sertifikayı aynı isimlerle (örneğin yine `master.crt` ve `master.key` olarak) kopyalayın.

### 3. Servislerin Yeniden Başlatılması (Güvenlik Ayarlarının Uygulanması)

Hem JWT/S3 yapılandırma dosyaları hem de TLS sertifikaları tüm sunucularda yerlerine kopyalandıktan sonra, bu güvenlik politikalarının devreye girmesi için tüm sunucularda SeaweedFS servislerini yeniden başlatın:

```bash
# Tüm sunucularda (Node 1, 2 ve 3) çalıştırın:
sudo systemctl restart seaweedfs-master seaweedfs-volume

# Sadece Node 1 üzerinde ek olarak Filer servisini de yeniden başlatın:
sudo systemctl restart seaweedfs-filer
```

Servislerin durumunu kontrol etmek için:

```bash
sudo systemctl status seaweedfs-* --no-pager
```

---

## 🔄 Replikasyon Yapılandırması

Yeni oluşturulacak tüm hacimler (volume) için varsayılan replikasyon seviyesi, `seaweedfs-master.service` servisindeki `-defaultReplication=001` parametresi ile otomatik olarak **001** (aynı rack üzerinde farklı sunucularda 1 ek kopya, toplam 2 kopya) olarak yapılandırılmıştır.

Mevcut veya sonradan değiştirilen hacimlerin replikasyon durumlarını yönetmek için `weed shell` kullanılır:

```bash
# mTLS ve master listesini belirterek shell'i başlatın
sudo weed shell -master=192.168.47.21:9333,192.168.47.23:9333,192.168.47.25:9333
```

Weed shell içerisinde kullanışlı replikasyon komutları:

```text
# Belirli bir hacmin (örn. Hacim ID: 7) replikasyon politikasını değiştirme
volume.configure.replication -volumeId=7 -replication=001

# Replikasyon politikası değişen veya eksik kopyası olan hacimleri otomatik eşleme
volume.fix.replication

# Kümedeki hacim dağılımını dengeli hale getirme
volume.balance -force
```

---

## 🌐 Arayüzler ve Erişim Noktaları

| Servis | URL | Açıklama |
|---|---|---|
| **Master UI** | `http://<herhangi-bir-node-ip>:9333` | Küme yönetimi (3 IP'den de girilebilir) |
| **Filer UI** | `http://192.168.47.21:8888` | Dosya tarayıcı, dizin yönetimi |
| **Volume UI** | `http://<node-ip>:8080` | Volume sunucu durumu (her node) |
| **S3 Endpoint** | `http://192.168.47.21:8333` | S3 uyumlu API |

> [!TIP]
> **Enterprise Reverse Proxy (Nginx / HAProxy) Tavsiyesi:**
> SeaweedFS'in HTTP/HTTPS endpoints'leri (Filer, S3 ve UI portları) doğrudan dış dünyaya açılmamalıdır. Üretim ortamında, bu portların önüne **Nginx**, **HAProxy** veya **Envoy** gibi bir reverse proxy yerleştirilmesi şiddetle tavsiye edilir.
> Bu proxy katmanı şu amaçlara hizmet eder:
>
> 1. **SSL/TLS Sonlandırma:** HTTP isteklerini HTTPS'e yönlendirerek istemci-sunucu arasındaki veri trafiğini şifrelemek.
> 2. **Yük Dengeleme (Load Balancing):** Filer servislerini birden fazla sunucuya kurduğunuzda, gelen S3/Filer yükünü düğümlere dengeli dağıtmak.
> 3. **IP Kısıtlama ve Güvenlik:** Sadece yetkili IP adreslerinin Master UI ve Volume UI portlarına erişmesini sağlamak.

### 🔒 Güvenlik Duvarı (Firewall) ve Port İzinleri

SeaweedFS bileşenleri arasındaki gRPC iletişimi için **gRPC portu = HTTP portu + 10000** kuralı geçerlidir. Kümenin sağlıklı çalışabilmesi ve mTLS bağlantısının kurulabilmesi için aşağıdaki portların sunucular arasında açık olması **şarttır**:

| Servis | HTTP Portu | gRPC Portu | Kullanım Amacı | Erişim Yönü |
|---|---|---|---|---|
| **Master** | `9333 / TCP` | `19333 / TCP` | Raft lider seçimi, eşleme ve kontrol | Tüm Node'lar kendi aralarında |
| **Volume** | `8080 / TCP` | `18080 / TCP` | Dosya okuma/yazma, Heartbeat bildirimleri | Volume <-> Master / Filer <-> Volume |
| **Filer** | `8888 / TCP` | `18888 / TCP` | POSIX/Filer REST işlemleri, Metadata senkronizasyonu | Filer <-> Master / Filer <-> Volume |
| **S3 API** | `8333 / TCP` | - | S3 uyumlu API istekleri | İstemciler -> Node1 |
| **Metrics** | `9324-9326 / TCP`| - | Prometheus metrik takibi (İsteğe bağlı) | Metrik Toplayıcı -> Tüm Node'lar |

> [!IMPORTANT]
> Güvenlik duvarınızda (firewalld, ufw veya cloud security groups) gRPC portlarının (`19333`, `18080`, `18888`) engellenmediğinden emin olun. Aksi halde mTLS el sıkışması başarısız olacak ve küme birleşemeyecektir.

---

## 🛠 Faydalı Komutlar

```bash
# Servis loglarını canlı takip et
journalctl -u seaweedfs-master -f
journalctl -u seaweedfs-volume -f
```

```bash
# Weed shell — küme yönetimi
# (mTLS etkinken güvenlik dosyalarını okuyabilmesi için sudo veya seaweedfs kullanıcısıyla çalıştırılmalıdır)
sudo weed shell
```

> **Not (mTLS ve Yetki Uyarısı):** gRPC mTLS şifrelemesi aktif edildiğinde, `weed shell` istemcisi master düğümlerle güvenli el sıkışma yapabilmek için `/etc/seaweedfs/security.toml` ve client sertifikalarına ihtiyaç duyar. Bu dosyaların izinleri `600` (sadece seaweedfs kullanıcısına açık) olduğundan, yetki hatası (Permission Denied) almamak için komutu `sudo weed shell` ya da `sudo -u seaweedfs weed shell` şeklinde çalıştırmanız gerekmektedir.

Weed shell içinde:

```text
cluster.ps              # Tüm master ve volume nodelarının durumunu gösterir
volume.list             # volume listesi
```

```bash
# Dosya yükle — Filer API üzerinden test
curl -F "file=@test.txt" http://192.168.47.21:8888/uploads/

# Dosya yükle — S3 API üzerinden test
aws s3 cp test.txt s3://bucket-name/ --endpoint-url http://192.168.47.21:8333
```

> **Not:** Eğer 2. adımdaki `security.toml` (JWT) güvenlik ayarını aktif ettiyseniz, yukarıdaki basit yükleme komutları `401 Unauthorized` hatası verecektir. Test edebilmek için backend üzerinden JWT Token üretip header ile göndermeniz gerekir.
