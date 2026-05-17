# GlusterFS: Mimari ve Temel Kavramlar

## 1. GlusterFS Nedir?

<img src=".\img\glusterfs.jpg" alt="alt text" width="360" height="250">

GlusterFS, petabaytlarca veriyi işleyebilen, ölçeklenebilir, açık kaynak kodlu ve dağıtık bir ağ dosya sistemidir (Distributed File System). Verileri tek bir sunucuda tutmak yerine, ağ üzerindeki birden fazla sunucuya (node/peer) dağıtarak veya kopyalayarak yüksek erişilebilirlik (High Availability), hata toleransı ve yüksek performans sağlar.

Genellikle bulut bilişim, medya yayıncılığı (streaming), büyük veri analizi (Big Data) ve sanallaştırma altyapılarında tercih edilir.

## 2. Temel Kavramlar ve Terminoloji

GlusterFS mimarisini anlamak için aşağıdaki terimlerin bilinmesi önemlidir:

- **Node (Peer):** GlusterFS'in kurulu olduğu ve depolama havuzuna (storage pool) dahil olan fiziksel veya sanal sunuculardır.
- **Trusted Storage Pool:** Birlikte çalışan ve birbirine güvenen GlusterFS sunucularının (nodeların) oluşturduğu havuzdur.
- **Brick:** GlusterFS'teki en temel depolama birimidir. Bir sunucu üzerindeki belirli bir dizini ifade eder (Örn: `Sunucu1:/veri/brick1`).
- **Volume:** Brick'lerin bir araya getirilmesiyle oluşturulan ve istemcilere (client) sunulan mantıksal depolama alanıdır. İstemciler bu Volume'u kendi sistemlerine bağlarlar (mount ederler).
- **Subvolume:** Bir Volume oluşturulurken mantıksal olarak gruplanan brick topluluğudur.
- **Translator:** GlusterFS'in kalbidir. Kullanıcı isteklerini alıp veriye dönüştüren, replikasyon, dağıtım, şifreleme veya önbellekleme gibi işlemleri gerçekleştiren modüllerdir.
- **FUSE (Filesystem in Userspace):** Çekirdek (kernel) kodunu değiştirmeden kullanıcı alanında dosya sistemleri oluşturmaya olanak tanıyan bir yazılım arayüzüdür. GlusterFS istemcileri genellikle FUSE kullanır.

## 3. GlusterFS Mimarisinin Avantajları

- **Metadata Sunucusu Yoktur (No-Metadata Server Architecture):** Geleneksel dosya sistemlerinin aksine (HDFS gibi), verinin nerede olduğunu tutan merkezi bir metadata sunucusu yoktur. Bunun yerine algoritmik bir yaklaşım (Elastic Hash Algorithm) kullanarak verinin hangi brick'te olduğunu hesaplar. Bu, darboğazları (bottleneck) ve tek nokta hatalarını (Single Point of Failure) ortadan kaldırır.
- **Otomatik Onarım (Self-Heal):** Çöken veya ağdan düşen bir sunucu (node) tekrar çevrimiçi olduğunda, eksik veya değişmiş verileri diğer düğümlerden otomatik olarak çekerek kendini onarır.
- **Kolay Ölçeklenebilirlik:** İhtiyaç anında sistemi durdurmadan yeni sunucular (nodelar) veya brickler eklenebilir.
- **Donanımdan Bağımsızlık:** Özel donanımlar gerektirmez. Standart (commodity) sunucular ve disklerle çalışabilir.

## 4. Volume Tipleri (Mimariler)

GlusterFS, ihtiyaca göre (performans, güvenlik veya kapasite) farklı Volume tipleri sunar:

### 4.1. Distributed Volume (Dağıtık Volume)

- **Nasıl Çalışır:** Dosyalar, havuzdaki brick'ler arasına rastgele dağıtılır (Dosya1 BrickA'ya, Dosya2 BrickB'ye gider).
- **Kullanım Amacı:** Maksimum depolama kapasitesi ve okuma/yazma hızı.
- **Dezavantaj:** Veri kopyalanmaz. Bir brick çökerse, o brick'teki veriler kaybolur.

### 4.2. Replicated Volume (Kopyalanmış Volume)

- **Nasıl Çalışır:** Aynı dosyanın kopyaları birden fazla brick'e yazılır (RAID 1 mantığı).
- **Kullanım Amacı:** Yüksek erişilebilirlik (High Availability) ve veri güvenliği.
- **Dezavantaj:** Kullanılabilir depolama alanı azalır (Örn: 2 adet 1TB diskiniz varsa, kopyalama yüzünden kullanılabilir alan 1TB olur).

### 4.3. Distributed Replicated Volume (Dağıtık ve Kopyalanmış)

- **Nasıl Çalışır:** Hem dağıtma hem de kopyalama işlemini aynı anda yapar (RAID 10 mantığı). Dosyalar önce gruplara dağıtılır, ardından grup içindeki disklerde kopyalanır.
- **Kullanım Amacı:** Hem yüksek performans hem de yüksek veri güvenliğinin gerektiği üretim (production) ortamları için en çok tercih edilen tiptir.

### 4.4. Striped Volume (Bölünmüş Volume) - *Eski Sürümlerde*

- **Nasıl Çalışır:** Çok büyük bir dosya, küçük parçalara (stripes) bölünerek farklı brick'lere yazılır (RAID 0 mantığı).
- **Not:** Yeni GlusterFS sürümlerinde yerini büyük ölçüde Sharding özelliğine bırakmıştır.

### 4.5. Dispersed Volume (Dağılmış Volume)

- **Nasıl Çalışır:** Erasure Coding (Hata Düzeltme Kodlaması) algoritmasını kullanır. Veriyi parçalara böler ve bir miktar "kurtarma" (parity) verisi ekleyerek disklere dağıtır.
- **Kullanım Amacı:** Replicated Volume kadar disk alanı israf etmeden, veri güvenliği sağlamak (RAID 5 / RAID 6 mantığı).

### 4.6. Arbiter Volume (Hakem Volume)

- **Nasıl Çalışır:** Replicated volume mantığına dayanır ancak veri kopyası tutmayan 3. bir "Arbiter" (hakem) sunucusu barındırır. Arbiter sadece dosya adlarını ve yapısını (metadata) tutar.
- **Kullanım Amacı:** 2 düğümlü (node) yapılarda ağ kopması durumunda "Split-Brain" (veri ayrışması) riskini, 3. tam donanımlı bir veri sunucusu maliyetine girmeden oylama ile engellemektir.

---

## 5. GlusterFS Kullanım Senaryoları (Usecases)

1. **Konteyner ve Kubernetes Depolaması:** Pod'lar için kalıcı depolama (Persistent Storage) sağlamak. Heketi gibi araçlarla entegre çalışabilir.
2. **Yedekleme ve Arşivleme:** Ucuz disklerle yüksek kapasiteli dağıtık depolama alanları oluşturmak.
3. **Web Sunucusu Kümeleri (Web Server Clusters):** Yüzlerce web sunucusunun aynı `/var/www/html` dizinine aynı anda okuma/yazma yapabilmesi.
4. **Medya Akışı (Media Streaming):** Büyük boyutlu video ve ses dosyalarının yüksek okuma hızlarıyla kullanıcılara sunulması.

## 6. Dezavantajları ve Sınırları

- **Küçük Dosya Performansı:** Milyonlarca çok küçük boyuttaki dosyanın (kb seviyesinde) okunması ve yazılması (metadata overhead nedeniyle) geleneksel sistemlere göre daha yavaş olabilir.
- **Ağ Bağımlılığı:** Nodelar arasındaki veri senkronizasyonu ağ üzerinden yapıldığı için, düşük gecikmeli (low latency) ve yüksek bant genişliğine sahip (örn: 10 Gbps) bir ağ altyapısı gerektirir.
- **Split-Brain Senaryosu:** Nodelar arasındaki ağ bağlantısı koparsa ve her iki node da kendi üzerindeki veriyi değiştirmeye devam ederse, ağ geri geldiğinde hangi verinin doğru olduğuna karar verilemeyen "Split-Brain" durumu yaşanabilir (Önlemek için genellikle 3. bir karar verici 'arbiter' node kullanılır).
