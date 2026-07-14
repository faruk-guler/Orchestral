# Bölüm 3: Temel ve İleri Seviye Konteyner Komutları

Docker'da her şey komut satırından (CLI) başlar. Bir `docker run` komutu yazdığınızda aslında Docker Daemon'a uzun bir istek (request) gönderirsiniz. Bu bölümde, günlük hayatta en çok kullanacağınız komutları ve bu komutların yanına eklenebilen **tüm kritik parametreleri (flag/argümanlar)** detaylıca inceleyeceğiz.

## 1. Konteyner Başlatma: `docker run`

`docker run`, Docker evreninin en çok kullanılan komutudur. Arka planda aslında iki işlemi birleştirir: `docker create` (konteyneri oluşturur) ve `docker start` (konteyneri başlatır).

### En Kapsamlı `docker run` Örneği:
```bash
docker run -d \
  --name veritabani \
  -p 8080:80 \
  -v benim-volumem:/data \
  -e MYSQL_ROOT_PASSWORD=gizlisifre \
  --network benim-agim \
  --restart always \
  --memory="512m" \
  --cpus="1.5" \
  --user 1000:1000 \
  --read-only \
  --rm \
  nginx:latest
```

### Parametrelerin (Flag) Eksiksiz Listesi:
- `-d` veya `--detach`: Konteyneri arka planda (sessizce) çalıştırır. Terminalinizi kilitlenmekten kurtarır.
- `-it`: Interactive ve TTY (Terminal) anlamına gelir. Konteynerin içine komut satırından bağlanmanızı (shell almanızı) sağlar. Genellikle `-it` şeklinde birleşik yazılır.
- `--name <isim>`: Konteynerinize rastgele bir isim (örn: *jovial_einstein*) verilmesi yerine sizin belirlediğiniz ismi atar. Konteynerleri yönetmek için şarttır.
- `-p <host_port>:<container_port>` veya `--publish`: Port yönlendirmesi yapar. Makinenizin 8080 portuna gelen istekleri, konteynerin içindeki 80 portuna iletir.
- `-P` (Büyük P) veya `--publish-all`: Konteynerin `EXPOSE` ile açtığı tüm portları, host makinedeki rastgele boş portlara bağlar.
- `-v <host_path>:<container_path>` veya `--volume`: Makinenizdeki bir klasörü veya Docker Volume'unu, konteynerin içindeki bir klasöre bağlar (Kalıcı veri için).
- `-e <ANAHTAR>=<deger>` veya `--env`: Konteynere çevresel değişken (environment variable) gönderir. Şifreler, API anahtarları gibi ayarlar bu yolla iletilir.
- `--env-file <dosya.env>`: Tüm ortam değişkenlerini tek tek `-e` ile yazmak yerine `.env` uzantılı bir dosyadan topluca okutmanızı sağlar.
- `--network <ag_adi>`: Konteynerin hangi Docker ağına bağlanacağını belirler. (Aynı ağdaki konteynerler isimleriyle birbirleriyle haberleşebilir).
- `--restart <kural>`: Konteyner çökerse veya bilgisayar yeniden başlarsa ne olacağını belirler. (Seçenekler: `no`, `on-failure`, `always`, `unless-stopped`).
- `-m` veya `--memory`: Konteynerin kullanabileceği maksimum RAM miktarını sınırlar. (Örn: `512m`, `1g`).
- `--cpus`: Konteynerin kullanabileceği maksimum CPU çekirdek sayısını sınırlar. (Örn: `0.5` yarım çekirdek, `2` iki çekirdek).
- `--user <uid>:<gid>`: Konteynerin içindeki uygulamanın `root` yerine hangi Linux kullanıcısı ile çalışacağını belirler. (Güvenlik için çok önemlidir).
- `--read-only`: Konteynerin kök dosya sistemini (rootfs) salt-okunur (read-only) yapar. İçine hiçbir şey yazılamaz, virüs bulaşması zorlaşır.
- `--rm`: Konteyner durdurulduğu anda kendisini (ve oluşturduğu geçici dosyaları) otomatik olarak siler. Test amaçlı çalıştırılan geçici (disposable) konteynerler için kullanılır.

---

## 2. Çalışan Konteynerleri İzleme: `docker ps`

Host makinenizde o an nelerin çalıştığını görmek için kullanılır.

```bash
docker ps
```

### Parametreler:
- `-a` veya `--all`: Sadece çalışanları değil, durdurulmuş (çökmüş veya işi bitmiş) tüm konteynerleri gösterir.
- `-q` veya `--quiet`: Ekrana sadece konteyner ID'lerini basar. (Özellikle toplu silme komutlarında çok işe yarar: `docker rm -f $(docker ps -aq)`).
- `-s` veya `--size`: Ekrana fazladan bir sütun ekleyerek konteynerlerin diskte ne kadar boyut kapladığını gösterir.
- `--format`: Çıktıyı JSON veya belirli bir Go şablonuna göre formatlamanızı sağlar.

---

## 3. Çalışan Konteynerin İçine Girmek: `docker exec`

Halihazırda çalışan bir konteynerin içine "dışarıdan" yeni bir süreç başlatarak girmeyi (örneğin terminal açmayı) sağlar. `docker run` ile karıştırılmamalıdır; `run` yeni konteyner yaratır, `exec` ise çalışana müdahale eder.

```bash
# Çalışan "web-sunucu" adlı konteynerin içinde sh (shell) başlat ve bana bağla (-it)
docker exec -it web-sunucu /bin/sh
```

### Parametreler:
- `-it`: Etkileşimli bir terminal açar.
- `-u` veya `--user`: İçeriye root olarak değil, farklı bir kullanıcı olarak (örn: `-u postgres`) bağlanmanızı sağlar.
- `-e`: Sadece o anki exec süreci için geçerli olacak yeni ortam değişkenleri tanımlar.
- `-w` veya `--workdir`: Konteynerin içine girdiğinizde doğrudan belirli bir klasörde (örn: `/var/www/html`) başlamanızı sağlar.

---

## 4. Temizlik ve Yönetim Komutları

Bir süre sonra Docker makineniz kullanılmayan imajlar ve durmuş konteynerlerle çöplüğe dönebilir.

### Silme Komutları:
- `docker rm <konteyner_id>`: Durdurulmuş bir konteyneri siler.
- `docker rm -f <konteyner_id>`: Çalışan bir konteyneri **zorla (force)** durdurup siler.
- `docker rmi <imaj_id>`: İndirilmiş bir imajı siler.
- `docker image prune`: İsmi veya etiketi olmayan (dangling) eski imajları temizler.

### Nihai Temizlik Silahı: `docker system prune`
Mevcut disk alanınızı geri kazanmak için en çok kullanacağınız komuttur:
```bash
docker system prune -a --volumes
```
*(Bu komut kullanılmayan tüm imajları, durdurulmuş konteynerleri, kullanılmayan ağları ve boştaki volume'leri tek kalemde tamamen siler. Production ortamlarında dikkatli olunmalıdır!)*

---
*Konteynerleri başlatmayı, içine girmeyi ve detaylı parametrelerini öğrendik. Peki ya bizim kendi uygulamalarımız? Sıradaki bölümde, kendi uygulamalarımızı paketleyip kendi imajlarımızı oluşturmamızı sağlayan "Dockerfile" konusunu detaylıca inceleyeceğiz.*
