# Git Ultimate Master Rehberi

<img src="img/git-logo.png" width="300" height="300">

Git, sadece bir versiyon kontrol sistemi değil; modern yazılım geliştirme süreçlerinin kalbi, hız ve güvenliğin simgesidir. Bu rehberde, Git'in yüzeysel komutlarından ziyade kaputun altındaki mimariyi ve mühendislik felsefesini inceleyeceğiz.

## 🧠 Git'in Özü: Dağıtık ve İçerik-Adresli Yapı

Geleneksel SVN gibi merkezi sistemlerin aksine Git, **Dağıtık (Distributed)** bir yapıdadır. Her geliştirici, projenin sadece bir kopyasını değil, tüm tarihçesini ve veritabanını içeren tam bir yerel kopyasını (Clone) taşır.

### 1. Snapshot (Anlık Görüntü) Mantığı

Git, diğer sistemler gibi dosyalar arasındaki "farkları" (diff) saklamaz. Her commit, projenin o anki halinin tam bir **Snapshot** (anlık görüntü) bilgisini tutar. Eğer bir dosya değişmemişse, Git yeni bir kopya oluşturmak yerine önceki commit'teki dosyaya bir bağlantı (pointer) atar. Bu, devasa projelerde inanılmaz bir hız sağlar.

### 2. Veri Bütünlüğü (SHA-1)

Git'teki her şey, içeriğine göre isimlendirilir ve takip edilir. Her dosya, dizin ve commit; içeriğinden üretilen benzersiz bir **SHA-1 Hash** (40 karakterli HEX kodu) ile saklanır. Bu durum, veri bozulmasını veya gizlice değiştirilmesini imkansız kılar. Git, bir dosyanın içeriği değiştiğinde bunu anında anlar.

---

## 🏗️ Git'in Üç Ana Katmanı

Git ile çalışırken kodunuz şu üç durumdan birindedir:

1. **Working Directory (Çalışma Dizini):** Dosyaların disk üzerindeki fiziksel halleridir. Burada düzenleme yaparsınız.
2. **Staging Area (Index - Hazırlık Alanı):** Bir sonraki commit'e nelerin dahil edileceğini belirlediğiniz ara katmandır. Git'in en güçlü özelliklerinden biridir; her değişikliği değil, sadece "hazır" olanları paketlemenizi sağlar.
3. **Local Repository (.git dizini):** Onaylanmış (committed) değişikliklerin kalıcı olarak saklandığı veritabanıdır.

---

## 🔱 Dallanma (Branching) Mimarisi

Git'i rakiplerinden ayıran en büyük fark "Branching" hızıdır. Git'te bir branch açmak, devasa dosyaları kopyalamak değil; sadece belirli bir commit SHA'sına işaret eden **41 byte'lık küçük bir metin dosyası** oluşturmaktır.

- **Merging:** Dallarımızı birleştirirken Git, "Three-way Merge" algoritmasını kullanarak ortak ataları bulur ve akıllıca birleştirir.
- **Rebase:** Tarihçeyi daha lineer ve temiz tutmak için commit'leri başka bir noktanın üzerine "yeniden inşa etme" sanatıdır.

---

## 🛠️ Neden Git Master Olmalısınız?

Modern dünyada Git bilmek sadece `push` ve `pull` yapmak değildir. Gerçek bir Git Master;

- **CI/CD** süreçlerini otomatize edebilir,
- **Merge Conflict** (çakışma) savaşlarını saniyeler içinde çözebilir,
- `git reflog` ile silindiği sanılan kodları kurtarabilir,
- Dev projelerde (Monorepo) performansı optimize edebilir.

---

## 🛠️ Hızlı Erişim

- [Git Cheat Sheet (Hızlı Referans)](Git_CheatSheet.md)

> [!TIP]
> Bu dizindeki Modülleri (01-20) tek tek incelediğinizde; mimariden otomatiğe, güvenlikten performansa kadar Git'in hiçbir karanlık noktası kalmayacaktır.
