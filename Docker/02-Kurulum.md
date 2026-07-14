# Bölüm 2: İleri Seviye Kurulum ve Daemon Konfigürasyonu

Docker'ı bilgisayarınıza veya sunucunuza kurmak genellikle "İleri - İleri - Kur" tarzı basit bir işlemdir. Ancak "Zero to Hero" seviyesinde bir mühendis olarak, işletim sistemlerinin arka planında Docker'ın donanımla nasıl konuştuğunu ve ince ayarlarını (Daemon Configuration) bilmeliyiz.

## 1. Windows Kurulumu ve WSL 2 Mimarisi

Windows, yapısı gereği Linux tabanlı konteynerleri doğrudan çalıştıramaz. Eskiden bu sorun **Hyper-V** üzerinde tam bir sanal makine kurularak çözülüyordu. Ancak bu yöntem hantal ve yavaştı. 

Günümüzde Docker, Windows üzerinde **WSL 2 (Windows Subsystem for Linux 2)** mimarisini kullanır.

### Kurulum Adımları
1. [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)'u indirin.
2. Kurulum ekranında **"Use WSL 2 instead of Hyper-V"** seçeneğinin İŞARETLİ olduğundan emin olun (Önerilen).
3. Bilgisayarı yeniden başlatın.

### WSL 2 Detayları
WSL 2, Windows'un içine gömülü hafif bir gerçek Linux çekirdeğidir (Lightweight Utility VM). Docker Desktop kurulduğunda arka planda iki tane özel WSL dağıtımı (distro) oluşturur:
- `docker-desktop`: Docker motorunun konfigürasyonlarını tutar.
- `docker-desktop-data`: İndirdiğiniz imajların ve volumelerin bulunduğu devasa disktir (`ext4.vhdx` formatında).

*İleri Seviye Not:* Eğer `C:\` sürücünüz dolarsa, Docker Desktop ayarlarındaki "Virtual Disk Location" kısmından arayüzü kullanarak veya WSL komutlarıyla (`wsl --export` ve `wsl --import`) diski D:\ veya başka bir sürücüye kolayca taşıyabilirsiniz.

## 2. macOS Kurulumu ve Alternatifler

macOS de (Unix tabanlı olmasına rağmen) Linux çekirdek özelliklerine (cgroups, namespaces) doğrudan sahip değildir. Bu nedenle Docker Desktop for Mac arka planda hafif bir Linux VM çalıştırır (Eskiden HyperKit, şimdi Apple'ın yerel Virtualization Framework'ünü kullanır).

### Kurulum
1. [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)'i indirin (İşlemcinizin Intel mi yoksa Apple Silicon - M1/M2 mi olduğuna dikkat edin!).
2. Kurun ve çalıştırın.

### Mac için Hafif ve Ücretsiz Alternatif: Colima
Docker Desktop şirketler için belirli bir ölçeğin üzerinde ücretlidir ve bazen çok RAM tüketir. Mac üzerinde harika bir alternatif **Colima**'dır.

```bash
# Terminalden Colima kurulumu (Homebrew ile)
brew install colima docker docker-compose

# Arka planda Docker Daemon'u başlat (2 CPU, 4GB RAM ile)
colima start --cpu 2 --memory 4
```
Colima, komut satırı üzerinden Docker'ı sanki native (doğal) olarak çalışıyormuş gibi kullanmanızı sağlar ve çok daha hafiftir.

## 3. Linux (Ubuntu/Debian) Kurulumu - Native Ortam

Docker'ın asıl vatanı Linux'tur. Linux üzerinde kurduğunuz Docker, araya hiçbir sanal makine girmeden doğrudan işlemciniz ve RAM'iniz ile konuşur. En yüksek performans buradadır.

Kurulumu Ubuntu/Debian sistemlerde şu standart script ile saniyeler içinde yapabilirsiniz:

```bash
# Otomatik kurulum scriptini indir ve çalıştır
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### Kurulum Sonrası Kritik Güvenlik Ayarı: Root Olmadan Kullanım
Linux'ta `docker` komutları varsayılan olarak sadece `root` (veya `sudo`) yetkisiyle çalışır. Her seferinde `sudo docker...` yazmamak ve güvenliği sağlamak için kendi kullanıcınızı `docker` grubuna eklemelisiniz:

```bash
# Kendi kullanıcınızı docker grubuna ekleyin
sudo usermod -aG docker $USER

# Değişikliklerin aktif olması için oturumu kapatıp açın (veya şu komutu girin)
newgrp docker
```

---

## 4. İleri Seviye: Docker Daemon Konfigürasyonu (`daemon.json`)

Docker Sunucusunun (Daemon) beyni `/etc/docker/daemon.json` dosyasıdır (Windows/Mac için Docker Desktop ayarlarındaki "Docker Engine" JSON sekmesi).

Varsayılan olarak bu dosya boştur. Ancak performansı, güvenliği veya ağ ayarlarını özelleştirmek için buraya çok kritik JSON anahtarları eklenebilir.

### Kapsamlı Bir `daemon.json` Örneği ve Tüm Önemli Parametreler:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "default-address-pools": [
    {
      "base": "172.80.0.0/16",
      "size": 24
    }
  ],
  "insecure-registries": ["myregistry.local:5000"],
  "registry-mirrors": ["https://mirror.gcr.io"],
  "bip": "192.168.1.5/24",
  "mtu": 1440,
  "storage-driver": "overlay2",
  "max-concurrent-downloads": 5,
  "max-concurrent-uploads": 5,
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

### Parametrelerin Anlamları:
- **`log-driver` ve `log-opts`**: Docker konteyner logları diskte sonsuza kadar büyüyüp diski doldurmasın diye log rotasyonu (Rotation) yapar. Yukarıdaki örnekte: "Her bir log dosyası maksimum 50MB olsun ve en fazla 3 tane eski dosyayı tut, sonrasını sil" diyoruz. (Hayat kurtarır!)
- **`default-address-pools`**: Yeni Docker network'leri oluşturduğunuzda Docker rastgele IP blokları dağıtır (örn: 172.17.x.x). Eğer bu IP blokları şirket ağınızdaki VPN veya veritabanı IP'leri ile çakışıyorsa, Docker'ın kullanacağı havuzu buradan değiştirmelisiniz.
- **`insecure-registries`**: HTTPS sertifikası (SSL) olmayan özel depo adreslerinize bağlanmanızı sağlar (Bölüm 15'te değineceğiz).
- **`registry-mirrors`**: Docker Hub'a bağlanırken hız kazanmak (veya limitlere takılmamak) için Google'ın ayna (mirror) sunucularını kullanmanızı sağlar.
- **`bip` (Bridge IP):** Ana `docker0` bridge ağının sabit IP bloğunu belirler.
- **`mtu`**: Ağ paketlerinin maksimum boyutunu ayarlar. Kötü ağ yapılandırmalarında paket düşmelerini engeller.
- **`storage-driver`**: İmajların diske yazılma biçimidir. Genelde Linux için `overlay2`, Windows/Mac için `vfs` veya özel driver'lar seçilir.
- **`max-concurrent-downloads`**: `docker pull` yaparken aynı anda indirilecek maksimum katman (layer) sayısıdır. Bant genişliğini sınırlandırmak için kullanılır.
- **`metrics-addr`**: Prometheus gibi monitoring araçlarının Docker'ın iç metriklerini (CPU, RAM vb.) okuması için dışarıya bir port açar.
- **`experimental`**: Docker'ın henüz test aşamasındaki (yeni çıkan) komut setlerini kullanıma açar.

*Bu JSON dosyasını her güncellediğinizde, değişikliğin algılanması için Docker servisini yeniden başlatmanız (Linux'ta `sudo systemctl restart docker`) gerekir.*

---
*Artık Docker motoruna tamamen hakimiz. Sıradaki bölümde, bir konteyneri komut satırından çalıştırırken kullanabileceğimiz TÜM argümanları (flag) inceleyeceğiz.*
