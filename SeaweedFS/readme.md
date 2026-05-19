# SeaweedFS
<img src="https://raw.githubusercontent.com/seaweedfs/seaweedfs/master/note/seaweedfs.png" width="300" height="300">

> **3 Node'lu Kurumsal Best Practice (High Availability) SeaweedFS Kümesi Kurulum Rehberi.**

---

## 🧬 SeaweedFS Mimari ve Best Practice Yaklaşımı

Bu kurulum **Hiper-Bütünleşik (Hyper-Converged)** mimariyi temel alır. 3 sunuculu kümelerde Yüksek Erişilebilirlik (HA) sağlamak için **Master servisi 3 sunucuda birden çalıştırılır.**

Raft protokolü gereği, master sunucular aralarında oylama yaparak bir "Lider" seçerler. Sunuculardan biri fiziksel olarak çökse dahi, kalan 2 sunucu yeni lideri seçerek **sıfır kesintiyle** çalışmaya devam eder.

| IP Adresi | Hostname | Rol | Servisler |
|---|---|---|---|
| `192.168.47.21` | node1 | **Master + Volume + Filer** | master, volume, filer (S3) |
| `192.168.47.23` | node2 | **Master + Volume** | master, volume |
| `192.168.47.25` | node3 | **Master + Volume** | master, volume |

---

## 📌 Gereksinimler

- En az **3 adet** sanal makine.
- Veri diskleri **LVM** ile yapılandırılmış ve **Thick Provision** (tam boyut ayrılmış) olmalıdır.
- Node'ların farklı fiziksel sunucularda çalışmasını garanti eden **Anti-Affinity** kuralı tanımlanmalıdır.

---

## 🚀 Kurulum Adımları

### 1. SeaweedFS Kurulumu

SeaweedFS önceden derlenmiş (pre-compiled) tek bir binary dosyadan oluşur. Sunucuya Go kurmanıza gerek yoktur.

> **Not:** Aşağıdaki adımlar **3 NODE'DA DA** çalıştırılmalıdır.

```bash
cd /tmp
wget https://github.com/seaweedfs/seaweedfs/releases/latest/download/linux_amd64.tar.gz
tar -xzf linux_amd64.tar.gz
mv weed /usr/local/bin/
weed version
```

### 2. Sistem Kullanıcısı ve Dizin Yapısı

Güvenlik (Best Practice) gereği servisleri root olarak değil, yetkisiz bir kullanıcı ile çalıştıracağız.

```bash
# 3 NODE'DA DA ÇALIŞTIRIN:
useradd -r -s /bin/false seaweedfs
```

Bu HA mimarisinde dizinleri oluşturup yetkilerini `seaweedfs` kullanıcısına veriyoruz:

```bash
# Node 1 (Master + Filer + Volume) üzerinde:
mkdir -p /data/master /data/volumes /data/filer
chown -R seaweedfs:seaweedfs /data/master /data/volumes /data/filer

# Node 2 ve Node 3 (Master + Volume) üzerinde:
mkdir -p /data/master /data/volumes
chown -R seaweedfs:seaweedfs /data/master /data/volumes
```

> [!IMPORTANT]
> Diskleri `/etc/fstab`'da tanımladıktan sonra `mount -a` ile bağlayın. Servis başlamadan önce ilgili dizinlerin mount edilmiş olması gerekir.

---

## ⚙ Systemd Servisleri (Şablon Kurulum)

Karmaşayı önlemek ve yönetimi kolaylaştırmak için tüm servis konfigürasyonları `services/` klasöründe **ortak birer şablon (template)** olarak ayrılmıştır. Tüm sunucular bu ortak dosyaları kullanır.

```text
services/
├── seaweedfs-master.service
├── seaweedfs-volume.service
└── seaweedfs-filer.service
```

### Servislerin Kurulumu ve IP Düzenleme

Her sunucuda, rolüne uygun servis dosyalarını `/etc/systemd/system/` dizinine kopyalayın ve **içindeki `<SUNUCU_IP_ADRESI>` yazan yeri o sunucunun IP'si ile değiştirin.**

#### 1. Dosyaları Kopyalayın ve Düzenleyin

**Node 1 (`192.168.47.21`) üzerinde:**

```bash
# Servisleri kopyala
cp services/*.service /etc/systemd/system/

# Nano ile açıp <SUNUCU_IP_ADRESI> kısmını 192.168.47.21 yapın
nano /etc/systemd/system/seaweedfs-master.service
nano /etc/systemd/system/seaweedfs-volume.service
nano /etc/systemd/system/seaweedfs-filer.service
```

**Node 2 (`192.168.47.23`) ve Node 3 (`192.168.47.25`) üzerinde:**

```bash
# Sadece Master ve Volume kopyala
cp services/seaweedfs-master.service /etc/systemd/system/
cp services/seaweedfs-volume.service /etc/systemd/system/

# Nano ile açıp <SUNUCU_IP_ADRESI> kısmını ilgili sunucunun IP'si yapın
nano /etc/systemd/system/seaweedfs-master.service
nano /etc/systemd/system/seaweedfs-volume.service
```

#### 2. JWT Güvenlik Ayarları (Opsiyonel ama Önerilir)

Dışarıdan yetkisiz kişilerin dosya yüklemesini veya silmesini engellemek için `security.toml` dosyasını kullanmalıyız. Bu ayar sayesinde okuma (dosya görüntüleme) herkese açıkken, yazma/silme işlemleri JWT şifresine bağlanır.

**TÜM SUNUCULARDA (Node 1, 2 ve 3):**
```bash
mkdir -p /etc/seaweedfs
cp services/security.toml /etc/seaweedfs/
chown -R seaweedfs:seaweedfs /etc/seaweedfs
```
*(Not: `security.toml` içindeki şifreyi kendi sisteminize göre güncelleyebilirsiniz.)*

#### 3. Servisleri Başlatın

Düzenleme işlemi bittikten sonra her sunucuda aşağıdaki komutu çalıştırarak aktif edin:

**Node 1 için:**

```bash
systemctl daemon-reload
systemctl enable --now seaweedfs-master seaweedfs-filer seaweedfs-volume
systemctl status seaweedfs-master seaweedfs-filer seaweedfs-volume --no-pager
```

**Node 2 ve Node 3 için:**

```bash
systemctl daemon-reload
systemctl enable --now seaweedfs-master seaweedfs-volume
systemctl status seaweedfs-master seaweedfs-volume --no-pager
```

---

## 🔄 Replikasyon Yapılandırması

`001` şeması: verinin aynı rack içinde **1 kopyası** tutulur (toplam 2 kopya).

```bash
weed shell
```

Weed shell içinde:

```
volume.configure.replication -replication=001
```

---

## 🌐 Arayüzler ve Erişim Noktaları

| Servis | URL | Açıklama |
|---|---|---|
| **Master UI** | `http://<herhangi-bir-node-ip>:9333` | Küme yönetimi (3 IP'den de girilebilir) |
| **Filer UI** | `http://192.168.47.21:8888` | Dosya tarayıcı, dizin yönetimi |
| **Volume UI** | `http://<node-ip>:8080` | Volume sunucu durumu (her node) |
| **S3 Endpoint** | `http://192.168.47.21:8333` | S3 uyumlu API |

---

## 🛠 Faydalı Komutlar

```bash
# Servis loglarını canlı takip et
journalctl -u seaweedfs-master -f
journalctl -u seaweedfs-volume -f
```

```bash
# Weed shell — küme yönetimi
weed shell
```

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
