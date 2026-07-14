# Bölüm 6: İleri Seviye Ağ Yönetimi (Networking)

Bir konteyner izole bir kutu gibidir, ancak modern uygulamalar (Örn: Web sunucusu ve Veritabanı) iletişim kurmak zorundadır. Docker, konteynerlerin birbirleriyle veya dış dünyayla güvenli bir şekilde iletişim kurmasını sağlamak için devasa bir **Ağ (Network)** altyapısı sunar.

Docker kurulduğunda varsayılan olarak arka planda bazı ağ sürücüleri (Network Drivers) oluşturur. Gelin tüm bu ağ mimarilerini en basitinden en karmaşığına doğru inceleyelim.

## 1. Ağ Sürücüleri (Network Drivers)

### A. Bridge Network (Köprü Ağı - Varsayılan)
Aksi belirtilmedikçe her konteyner `bridge` (docker0) adlı varsayılan ağa katılır. Bu ağ, host makinenizin içinde sanal bir router (yönlendirici) oluşturur.
- **Kullanım Yeri:** Aynı bilgisayarda çalışan, birbiriyle izole bir şekilde haberleşmesi gereken konteynerler.
- **Özel Bridge Ağları:** Kendi bridge ağınızı (User-defined bridge) oluşturmak, varsayılan `docker0` ağına göre çok daha üstündür. Çünkü kendi oluşturduğunuz ağlarda **Otomatik DNS Çözümlemesi** (Automatic DNS Resolution) vardır.

```bash
# Kendi ağımızı (bridge) oluşturuyoruz
docker network create benim-agim

# Konteynerleri bu ağa dahil ediyoruz
docker run -d --name veritabani --network benim-agim -e MYSQL_ROOT_PASSWORD=gizlisifre mysql
docker run -d --name web-sunucu --network benim-agim nginx
```
*DNS Mucizesi:* Artık `web-sunucu` konteynerinin içine girip `ping veritabani` yazarsanız, Docker arka planda bu ismi otomatik olarak veritabanının IP adresine çevirecektir! (Varsayılan `docker0` ağında bu özellik yoktur, o yüzden hep kendi ağınızı oluşturun).

### B. Host Network
Eğer bir konteyneri `--network host` parametresiyle başlatırsanız, o konteynerin ağ izolasyonu tamamen kaldırılır.
Konteyner, doğrudan host makinenizin (bilgisayarınızın) IP adresini ve portlarını kullanır.
- **Kullanım Yeri:** Aşırı yüksek ağ performansı gerektiren durumlar (çünkü aradaki sanal köprü atlanmış olur). Sadece Linux'ta çalışır.
- **Örnek:** `docker run -d --network host nginx` derseniz, host makinenin 80 portu direkt Nginx tarafından işgal edilir (Port yönlendirmeye `-p 80:80` gerek kalmaz).

### C. None Network
Konteynerin dış dünya ile veya diğer konteynerlerle olan tüm bağını koparır. Sadece loopback (`lo`, 127.0.0.1) arayüzü kalır.
- **Kullanım Yeri:** Sadece dosya işleyen, çok yüksek güvenlikli, kapalı devre batch-işlem (veri işleme) konteynerleri.

---

## 2. İleri Seviye (Advanced) Ağ Sürücüleri

Büyük veri merkezleri ve kurumsal ağlar için Docker'ın sunduğu daha derin sürücüler vardır:

### D. Overlay Network
Docker Swarm (Bölüm 8) kullanırken devreye girer. **Farklı fiziksel sunuculardaki** konteynerlerin, sanki aynı bilgisayardaymış gibi şifreli (encrypted) ve güvenli bir şekilde birbirleriyle haberleşmesini sağlayan "Bulut" ağ türüdür.

### E. Macvlan ve IPvlan Networks
Bazı durumlarda şirketinizin ağ yöneticisi "Konteynerlerinizin sanal (172.x.x.x) IP'ler almasını istemiyorum, doğrudan şirketin fiziksel ağına (192.168.1.x) fiziksel bir cihazmış gibi bağlansınlar ve MAC adresi alsınlar" diyebilir.
İşte `macvlan` bu işe yarar. Her konteynere ayrı bir MAC adresi atar ve doğrudan fiziksel router'ınıza bağlar.

```bash
# Örnek Macvlan ağ oluşturma (Fiziksel ağa eth0 üzerinden köprüleme)
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  sirket-macvlan
```

*(Not: `ipvlan` ise macvlan'a çok benzer ancak tüm konteynerler tek bir MAC adresini (host'un MAC adresi) paylaşır. Güvenlik politikaları MAC adresi kopyalamayı engelleyen Switch'lerde `ipvlan` hayat kurtarır).*

---

## 3. Ağ Komutları ve Yönetimi

Docker ağlarını yönetmek için `docker network` komut ailesini kullanırız:

```bash
# Tüm ağları listele
docker network ls

# Bir ağın içindeki cihazları (IP'leri ve bağlı konteynerleri) gör
docker network inspect benim-agim

# Çalışan bir konteyneri sonradan bir ağa bağla
docker network connect benim-agim web-sunucu

# Çalışan bir konteyneri ağdan kopar
docker network disconnect benim-agim web-sunucu

# Kullanılmayan (boşta kalan) tüm ağları sil
docker network prune
```

### DNS ve Network Alias (Ağ Takma İsimleri)
Bir konteyneri bir ağa bağlarken ona birden fazla "Ağ Alias" (takma ad) verebilirsiniz. Bu sayede mikroservis mimarilerinde Load Balancing (Yük Dengeleme) yapılabilir.
Örneğin 3 farklı Nginx konteyneri başlatıp üçüne de `--network-alias frontend` verirseniz, ağ içindeki başka bir bilgisayar `frontend` ismine istek attığında, Docker dahili DNS sunucusu (Embedded DNS) bu 3 konteyner arasında Round-Robin (sırayla) yük dağıtımı yapar!

---
*Artık her bir konteyneri detaylıca oluşturmayı, verilerini bağlamayı ve aynı ağa sokmayı biliyoruz. Ancak gerçek bir projede 5 farklı konteyneri tek tek bu komutlarla ayağa kaldırmak çok yorucudur. İşte burada devreye sihirli değneğimiz giriyor: **Docker Compose**.*
