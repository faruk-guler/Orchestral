# Bölüm 5: İleri Seviye Veri Yönetimi (Volumes, Bind Mounts ve tmpfs)

Konteynerler doğaları gereği "geçicidir" (ephemeral). Bir konteyneri silip yenisini yarattığınızda (örneğin imajın yeni bir versiyonunu yayınladığınızda), konteynerin içinde yazılmış olan tüm veriler (loglar, veritabanı kayıtları, yüklenen dosyalar) sonsuza dek kaybolur.

Bir veritabanı (MySQL, PostgreSQL) çalıştırdığınızı düşünün; verilerinizin konteyner silindiğinde kaybolmasını asla istemezsiniz. Bu noktada devreye **Kalıcı Veri (Persistent Data)** yöntemleri girer. Docker'da veriyi kalıcı hale getirmenin üç temel yöntemi vardır:

## 1. Docker Volumes (Önerilen Yöntem)

Volumes, doğrudan Docker tarafından yönetilen ve ana makinenin (host) diskinde özel bir alanda (`/var/lib/docker/volumes/` altında) saklanan veri alanlarıdır. Docker dışındaki süreçlerin (veya kullanıcıların) bu verilere müdahale etmesi zordur, bu yüzden en güvenli ve önerilen yöntemdir.

### Volume Komutları:
```bash
# Yeni bir volume oluşturmak
docker volume create benim-veritabani-volumem

# Mevcut volumeleri listelemek
docker volume ls

# Bir volume hakkında detaylı bilgi almak (nerede saklandığını görmek vb.)
docker volume inspect benim-veritabani-volumem

# Kullanılmayan (hiçbir konteynere bağlı olmayan) volumeleri silmek
docker volume prune
```

### Volume'ü Konteynere Bağlamak
```bash
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=gizlisifre \
  -v benim-veritabani-volumem:/var/lib/mysql \
  mysql:8.0
```
Bu komutla, konteynerin içindeki `/var/lib/mysql` dizininde yapılan her değişiklik, güvenli bir şekilde `benim-veritabani-volumem` isimli volume'e yazılır. Konteyner silinse bile veri güvendedir.

---

## 2. Bind Mounts (Klasör Bağlama)

Bind mount, host makinenizdeki spesifik bir klasörün veya dosyanın, konteynerin içine "bağlanmasıdır". Volume'den farkı şudur: Dosyalar `/var/lib/docker/` gibi izole bir alanda değil, **doğrudan sizin seçtiğiniz bir klasörde** (örn: `C:\Users\Projelerim\Kodlar` veya `/home/user/app`) durur.

Özellikle yazılım geliştirirken, bilgisayarınızda yazdığınız kodun anında konteynerin içine yansıması için (canlı kodlama / hot-reload) mükemmeldir.

### Bind Mount Örneği:
```bash
docker run -d \
  --name web-app \
  -v $(pwd)/src:/usr/share/nginx/html \
  nginx:latest
```
*Not:* `$(pwd)` komutu, bulunduğunuz dizini (print working directory) ifade eder. Windows PowerShell'de `${PWD}` kullanılır.

### Read-Only (Salt Okunur) Bind Mount
Güvenlik açısından, konteynerin makinenizdeki dosyalara **sadece okuma** yetkisiyle erişmesini istiyorsanız `:ro` parametresini ekleyebilirsiniz:
```bash
-v $(pwd)/ayarlar.json:/app/ayarlar.json:ro
```
Bu sayede konteyner hacklense dahi, hacker host makinenizdeki o dosyayı değiştiremez.

---

## 3. tmpfs Mounts (RAM Üzerinde Geçici Veri)

Çok az bilinen ama performans için kritik olan üçüncü yöntemdir. Eğer verinizin diske **hiç yazılmasını istemiyorsanız** (örneğin çok hassas şifreleme anahtarları (keys) tutuyorsanız veya inanılmaz hızlı I/O performansı gerektiren geçici cache veriniz varsa), veriyi sadece RAM üzerinde (tmpfs) tutabilirsiniz. Konteyner durduğunda veri anında yok olur.

```bash
docker run -d \
  --name hizli-cache \
  --tmpfs /app/cache:rw,size=256m \
  benim-imajim
```
Bu komut, konteynerin `/app/cache` dizinini doğrudan makinenin RAM'ine bağlar (256MB limit ile).

---

## İleri Seviye Veri Stratejileri

### A. Volume Backup (Yedekleme) ve Restore (Geri Yükleme)
Bir volume'ü doğrudan host üzerinden yedeklemek zordur. Bunun yerine "geçici (rm)" bir konteyner kullanarak veriyi tar/zip arşivine dönüştürmek en profesyonel yöntemdir:

**Yedek Alma (Backup):**
```bash
docker run --rm \
  -v benim-veritabani-volumem:/veri-kaynagi \
  -v $(pwd):/yedek-hedefi \
  ubuntu tar cvf /yedek-hedefi/yedek_2026.tar /veri-kaynagi
```
Bu komut geçici bir Ubuntu konteyneri açar, volume'ü ve bulunduğunuz klasörü içeri bağlar, verileri tar ile sıkıştırıp bilgisayarınıza `yedek_2026.tar` olarak bırakır ve kendini imha eder.

### B. İleri Seviye Volume Sürücüleri (Volume Drivers)
Docker Volume'leri sadece yerel diskinizde durmak zorunda değildir. **Volume Plugins (Eklentiler)** sayesinde verilerinizi bambaşka sunucularda tutabilirsiniz:
- **SSHFS/NFS Sürücüleri:** Konteyner verilerini uzak bir sunucuya (NFS veya SSH üzerinden) yazar.
- **Cloud Sürücüleri:** Amazon EBS, Google Cloud Persistent Disk veya Azure Files üzerinden devasa bulut depolama alanlarını doğrudan bir Docker Volume gibi kullanabilirsiniz.

**NFS Volume Örneği:**
```bash
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.50,rw \
  --opt device=:/path/to/nfs/share \
  nfs-volumem
```
Bu sayede Swarm veya Kubernetes kullanmadan bile birden fazla sunucudaki konteynerlerin aynı paylaşımlı klasöre veri yazmasını sağlayabilirsiniz.

---
*Verilerimizi kalıcı ve güvenli hale getirdik. Sıradaki bölümde, Konteynerlerin birbirleriyle ve dış dünyayla nasıl konuştuğunu "Ağ Yönetimi (Network)" konusunda derinlemesine inceleyeceğiz.*
