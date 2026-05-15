# GlusterFS: Kurulum ve Kullanım Rehberi

Üretim (Production) ortamları için en güvenilir yapı olan **3 Sunuculu (Arbiter)** GlusterFS kurulumunun temel adımları:

**Senaryo:** `server1` (Veri), `server2` (Veri) ve `server3` (Hakem/Arbiter) adında üç sunucumuz var. Arbiter sunucusu verilerin içeriğini (kopyasını) tutmaz, sadece dosya adlarını ve yapısını (metadata) tutarak ağ kopmalarında "Split-Brain" (veri ayrışması) riskini engeller. Bu yüzden küçük bir disk ile kurulabilir.

## Adım 1: Ön Hazırlık (Zaman Senkronizasyonu ve Hostname Ayarları)

GlusterFS'in sağlıklı çalışması ve dosyaların onarılma (Self-heal) sürecinde hangi verinin daha güncel olduğuna doğru karar verebilmesi için sunucu saatlerinin birbiriyle tam uyumlu olması **kritiktir.**

### 1.1. Zaman Senkronizasyonu (NTP)

Her üç sunucuda da zamanın güncel olduğundan emin olun:

```bash
sudo apt update && sudo apt install chrony -y
sudo systemctl enable --now chrony
# Zaman senkronizasyon durumunu kontrol et
chronyc tracking
```

### 1.2. Hostname ve /etc/hosts Ayarları

GlusterFS nodelarının birbirleriyle doğrudan IP yerine isim (hostname) üzerinden haberleşmesi çok daha güvenilir bir yaklaşımdır. Her **üç** sunucuda da aşağıdaki gibi isim çözümlemesini ayarlamalısınız:

```bash
# Hostname belirleme (örneğin server1 için)
sudo hostnamectl set-hostname server1

# /etc/hosts dosyasını düzenleyerek tüm nodeları ekleyin
# sudo nano /etc/hosts
192.168.1.25   server1
192.168.1.26   server2
192.168.1.27   server3 # Hakem/Arbiter
```

Değişikliklerin etkili olması için sunucuları `sudo reboot` komutuyla yeniden başlatabilirsiniz.

## Adım 2: Paket Kurulumu (Her Üç Sunucuda)

Debian 13 resmi depolarında GlusterFS paketleri kararlı ve güncel olarak bulunmaktadır. Doğrudan standart depolardan güvenle kurulum yapabiliriz.

```bash
# Debian 13 için GlusterFS ve XFS araçlarının kurulumu
sudo apt update
sudo apt install glusterfs-server xfsprogs -y
sudo systemctl enable --now glusterd
```

Yardımcı ve sürüm kontrol komutları:

```bash
sudo gluster --version
sudo gluster --help
sudo gluster volume list
```

## Adım 3: Güvenli Havuz (Trusted Pool) Oluşturma

Sadece `server1` üzerinden çalıştırılır:

```bash
# Diğer sunucuları havuza dahil et
sudo gluster peer probe server2
sudo gluster peer probe server3

# Durumu kontrol et (2 adet bağlı peer görünmelidir)
sudo gluster peer status
```

## Adım 4: Disk Hazırlığı ve Brick Dizini Oluşturma (Her Üç Sunucuda)

GlusterFS için en iyi pratik (best practice), ana işletim sisteminden bağımsız, **XFS** olarak biçimlendirilmiş ayrı bir disk bölümü kullanmaktır. (Arbiter olan `server3` çok fazla depolama alanına ihtiyaç duymaz, çok daha küçük bir disk ile XFS formatlanabilir).

> **Not:** Eğer test ortamındaysanız ve ayrı diskiniz yoksa, doğrudan `sudo mkdir -p /data/glusterfs/mybrick/brick` ile kök dizinde klasör açarak devam edebilirsiniz. Ancak production ortamları için aşağıdaki yöntem şarttır.

```bash
# Örnek: Sisteme eklenen /dev/sdb diskinin XFS ile formatlanması
sudo mkfs.xfs -i size=512 /dev/sdb

# Diskin bağlanacağı (mount) ana dizinin oluşturulması
sudo mkdir -p /data/glusterfs/mybrick

# Diski mount etme (ayarları anında uygulamak için parametrelerle)
sudo mount -o inode64,noatime,nodiratime /dev/sdb /data/glusterfs/mybrick

# fstab'a ekleme (kalıcı olması için)
echo '/dev/sdb /data/glusterfs/mybrick xfs inode64,noatime,nodiratime 0 0' | sudo tee -a /etc/fstab

# Brick klasörünün oluşturulması
sudo mkdir -p /data/glusterfs/mybrick/brick
```

## Adım 5: Volume Oluşturma ve Başlatma

Sadece `server1` üzerinden çalıştırılır:

```bash
# Replica 3 (2 Veri + 1 Arbiter) olan bir volume oluştur
# Sıralama önemlidir: İlk ikisi veri tutar, 3. yazılan arbiter olur.
sudo gluster volume create myvolume replica 3 arbiter 1 server1:/data/glusterfs/mybrick/brick server2:/data/glusterfs/mybrick/brick server3:/data/glusterfs/mybrick/brick force

# Volume'u başlat
sudo gluster volume start myvolume

# Güvenlik: Sadece belirtilen IP bloğunun erişimine izin ver (Best Practice)
sudo gluster volume set myvolume auth.allow 192.168.1.*

# Performans: Ağ kopmalarının daha hızlı algılanması için ping süresini düşür
sudo gluster volume set myvolume network.ping-timeout 5

# Çoğunluk (Quorum) kontrolünü aktif et (Veri güvenliği için kritik)
sudo gluster volume set myvolume cluster.quorum-type auto

# Volume durumunu gör
sudo gluster volume info
```

> **Not:** Arbiter yapısı sayesinde, sunuculardan biri veya ağ bağlantısı çökse bile sistem hangi sunucudaki verinin doğru olduğunu `server3`'ün oyuyla belirler ve Split-Brain senaryosu asla yaşanmaz.

## Adım 6: İstemciye (Client) Bağlama (Mount Etme)

Client makinesinde GlusterFS volume'unu sisteme bağlayabilmek için öncelikle Debian istemci paketinin kurulu olması gerekir:

```bash
# Debian 13 İstemci (Client) paketi kurulumu:
sudo apt update && sudo apt install glusterfs-client -y
```

Ardından volume'u mount edebilirsiniz. `backup-volfile-servers` parametresi, ilk bağlantı anında `server1` kapalı veya ulaşılamaz durumdaysa, istemcinin diğer sunucular üzerinden ağa dahil olmasını sağlar:

```bash
# Mount edilecek klasörü oluştur
sudo mkdir -p /mnt/glusterdata

# GlusterFS volume'unu geçici olarak mount et
sudo mount -t glusterfs -o backup-volfile-servers=server2:server3 server1:/myvolume /mnt/glusterdata

# Yazma izinlerini ayarla (İsteğe bağlı - Mevcut kullanıcıya yetki verir)
sudo chown -R $USER:$USER /mnt/glusterdata
```

### 6.1. Kalıcı Mount (Fstab) İşlemi

Sunucu yeniden başladığında volume'un otomatik bağlanması için `/etc/fstab` dosyasına eklemeliyiz. Ağ bağlantısı ayağa kalktıktan sonra mount işleminin yapıldığından emin olmak için `_netdev` parametresi mutlaka kullanılmalıdır:

```bash
echo 'server1:/myvolume /mnt/glusterdata glusterfs defaults,_netdev,backup-volfile-servers=server2:server3 0 0' | sudo tee -a /etc/fstab
```

> **Not:** İstemci mount işleminden sonra arka planda tüm GlusterFS havuzunun farkındadır. Sunuculardan herhangi biri sonradan çökse bile sistem kesintisiz okuma/yazma hizmeti almaya devam eder.

## Adım 7: Bakım ve İzleme (Sık Kullanılan Komutlar)

Sistemi canlıya aldıktan sonra durumunu kontrol etmek ve yönetmek için şu komutlar hayati önem taşır:

```bash
# Volume sağlık durumunu ve brick bağlantılarını kontrol et
sudo gluster volume status myvolume

# Kendi kendini onarma (Self-heal) durumunu kontrol et
sudo gluster volume heal myvolume info

# Bağlı olan sunucuların (peer) listesini ve durumunu gör
sudo gluster peer status

# Performans ölçümünü başlat/durdur ve sonuçları gör
sudo gluster volume profile myvolume start
sudo gluster volume profile myvolume info
sudo gluster volume profile myvolume stop
```

---

> **[ÖNEMLİ GÜVENLİK NOTU - FIREWALL]:**
> GlusterFS nodelarının haberleşebilmesi için Güvenlik Duvarı (Firewall) üzerinden şu portlara izin verilmelidir:
>
> - **24007** (Gluster Daemon)
> - **24008** (Management)
> - **49152 ve üzeri** (Her oluşturulan brick için sırayla 49152, 49153... şeklinde atanan TCP portları)
