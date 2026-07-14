# Bölüm 15: Özel Kayıt Defteri (Private Registry) ve İleri Seviye Sorun Giderme

Şirketinizin yazdığı özel kodları barındıran Docker imajlarını herkesin görebileceği Docker Hub'a (genel ağa) yüklemek istemezsiniz. Kendi sunucunuzda, sadece sizin şifreyle girebileceğiniz tamamen özel bir Docker Deposu (Private Registry) kurmak zorunluluktur.

Ayrıca bu bölümde, sistem gerçekten çöktüğünde `docker logs`'un bile yetmediği durumlarda uygulayacağımız Linux Kernel tabanlı sorun giderme (Troubleshooting) tekniklerini göreceğiz.

## 1. Kendi Private Registry'nizi Kurmak

Docker'ın orijinal `registry` isimli resmi bir imajı vardır. Bunu kendi sunucumuzda ayağa kaldırarak anında kendi Docker Hub'ımızı yaratabiliriz. Ancak güvenlik (Authentication) olmadan olmaz!

### A. htpasswd ile Kullanıcı/Şifre Oluşturma
Önce Apache'nin `htpasswd` aracını kullanarak şifreli bir dosya oluşturuyoruz:
```bash
# Gerekli paketi kur
sudo apt-get install apache2-utils

# "admin" adında bir kullanıcı ve şifre oluştur, auth klasörüne kaydet
mkdir auth
htpasswd -Bc auth/htpasswd admin
```

### B. Registry'i Compose ile Ayağa Kaldırma (`docker-compose.yml`)
```yaml
# Güncel Compose versiyonlarında 'version' satırına gerek yoktur.

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      # Şifreli giriş (Basic Auth) ayarları
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      # Webhook ayarı (İmaj yüklendiğinde başka bir sunucuya haber ver)
      REGISTRY_NOTIFICATIONS_ENDPOINTS: |
        - name: slack-bildirim
          url: https://hooks.slack.com/services/XXXX/YYYY
          timeout: 500ms
          threshold: 5
          action: push
    volumes:
      - ./auth:/auth
      - ./registry-data:/var/lib/registry
```

### C. Kullanımı
Önce kendi sunucunuza login olursunuz:
```bash
docker login myregistry.sirketim.com:5000
# Kullanıcı adı: admin, Şifre: *****
```

Sonra imajı etiketler ve kendi sunucunuza (Push) gönderirsiniz:
```bash
docker tag benim-appim:latest myregistry.sirketim.com:5000/benim-appim:latest
docker push myregistry.sirketim.com:5000/benim-appim:latest
```

## 2. Registry Bakımı: Garbage Collection (Çöp Toplama)

Özel Registry'nize her gün onlarca imaj yüklerseniz diskiniz hızla dolar. Bir imajı silseniz bile (`docker rmi`), diskteki fiziksel katmanlar anında silinmez. Registry üzerinde periyodik olarak **Garbage Collection (GC)** çalıştırmalısınız.

```bash
# Registry konteynerinin içine girip çöp toplama işlemini tetikle
docker exec -it registry-konteyner-id bin/registry garbage-collect /etc/docker/registry/config.yml
```
*(Bu komutu Linux sunucunuzda her gece saat 03:00'te çalışacak bir `cron` job olarak ayarlamanız best-practice'dir).*

---

## 3. Alternatif: Üçüncü Parti ve GitHub Container Registry (GHCR)

Sunucu kurmakla uğraşmak istemiyorsanız GitHub'ın sunduğu `ghcr.io` (GitHub Container Registry) harika bir ücretsiz (veya limitli) alternatiftir.

### GHCR (GitHub) Kullanımı
GitHub üzerinden bir **Personal Access Token (PAT)** (write/read packages yetkili) oluşturduktan sonra sisteme giriş yapabilirsiniz:
```bash
# Token'ı kullanarak terminalden giriş yapma
echo "TOKEN" | docker login ghcr.io -u github_kullanici_adiniz --password-stdin

# İmajı GHCR formatında etiketleme
docker tag benim-appim:latest ghcr.io/github_kullanici_adiniz/benim-appim:latest

# İmajı GitHub'a gönderme
docker push ghcr.io/github_kullanici_adiniz/benim-appim:latest
```

### Kendi Alan Adınızla (Domain) Depo Kullanımı
Kendi alan adınız üzerinden (örn: `farukguler.com/docker-images`) imaj çekerken iki farklı durum vardır:
- **HTTPS (Güvenli):** İsteğinizi doğrudan `docker pull farukguler.com/docker-images/imaj:etiket` şeklinde atabilirsiniz. Docker `https://` protokolünü kendisi otomatik ekler.
- **HTTP (Güvensiz):** Eğer sitenizde SSL yoksa Docker varsayılan olarak bu işlemi reddeder. Bunu aşmak için sunucudaki `/etc/docker/daemon.json` dosyasına `"insecure-registries": ["farukguler.com"]` ayarını girmeli ve Docker servisini yeniden başlatmalısınız.

---

## 4. İleri Seviye Sorun Giderme (Troubleshooting)

Konteyneriniz sürekli kapanıyor (CrashLoopBackOff) ve `docker logs konteyner-id` yazdığınızda ekranda hiçbir hata görmüyorsanız ne yapacaksınız? Sorun muhtemelen uygulamanızda değil, **işletim sisteminin kendisinde veya Docker Daemon'dadır.**

### A. OOM (Out Of Memory) Çökmelerini Bulmak
Konteyneriniz aşırı RAM tükettiği için Linux çekirdeği (OOM Killer) onu zorla öldürmüş olabilir. Konteyner bu sırada log yazmaya vakit bile bulamaz. Bunu görmek için Linux Kernel loglarına (dmesg) bakmalısınız:
```bash
sudo dmesg -T | grep -i oom
```
Çıktıda `Out of memory: Killed process 4598 (java) total-vm:1500000kB` görüyorsanız, konteynerinize RAM limiti dar gelmiş demektir.

### B. Docker Daemon (Motoru) Logları
Konteynerin değil, **Docker'ın kendi loglarına** bakmak için `journalctl` kullanırız:
```bash
# Docker motorunun son hatalarını göster
sudo journalctl -u docker.service --no-pager -n 50
```
Örneğin diskiniz `overlay2` sürücüsünde inode (dosya sayısı) limitine ulaştıysa veya ağ köprüsü (`docker0`) çöktüyse bu hatayı sadece burada görebilirsiniz.

### C. Ağ Zafiyetleri (Network Troubleshooting)
İki konteyner birbiriyle konuşamıyorsa, DNS mi bozuk yoksa port mu kapalı anlamak için özel bir "Ağ test konteyneri" (Netshoot) kullanırız. Bu imajın içinde `ping`, `curl`, `telnet`, `nslookup` gibi tüm hacker/sistem araçları bulunur.
```bash
docker run -it --rm --network benim-agim nicolaka/netshoot

# İçeri girdikten sonra diğer konteynerin portunu test edin
telnet veritabani-konteyneri 3306
```

---
*Geliştirme ortamları, CI/CD, güvenlik, hata ayıklama... Docker evreninde bilmeniz gereken her şeyi öğrendiniz. Artık sadece Docker komutlarını ezberleyen biri değil, arka planda neyin nasıl çalıştığını bilen, üretim ortamlarına (Production) yön verebilen gerçek bir Docker Hero'sunuz. Yeni nesil yazılım mimarilerinde başarılar dileriz!*

