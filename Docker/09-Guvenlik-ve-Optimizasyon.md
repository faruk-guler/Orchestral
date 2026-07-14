# Bölüm 9: İleri Seviye Güvenlik (Hardening) ve Optimizasyon

Docker, yapısı gereği izolasyon sağlar ancak varsayılan ayarlarıyla **%100 güvenli değildir**. Eğer bir hacker konteynerinize sızmayı başarırsa ve içeride `root` yetkisine sahipse, özel teknikler kullanarak (Container Breakout) doğrudan ana sunucunuzu ele geçirebilir.

Bu bölümde konteynerleri adeta bir kaleye dönüştürecek ileri düzey güvenlik "Hardening" (sıkılaştırma) yöntemlerini inceleyeceğiz.

## 1. Altın Kural: Asla Root Olmayın (Non-Root User)

Varsayılan olarak Docker konteynerleri içerideki süreçleri `root` kullanıcısı (UID: 0) olarak çalıştırır. Uygulamanızın çalışması için gerçekten root yetkisine ihtiyacı yoksa, bunu mutlaka engellemelisiniz.

### Dockerfile İçinde Kullanıcı Değişimi
```dockerfile
FROM node:18-alpine

# Linux üzerinde 'appuser' adında kısıtlı bir kullanıcı oluştur
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

# Dosya sahipliğini yeni kullanıcıya ver
RUN chown -R appuser:appgroup /app

# Bundan sonraki tüm komutları root olmayan kullanıcıyla çalıştır
USER appuser

CMD ["npm", "start"]
```

## 2. İleri Seviye: Rootless Docker Daemon

Konteynerin içi root olmasa bile, host makinenizde çalışan **Docker Daemon'un kendisi** root yetkisiyle (sudo) çalışır.
"Rootless Mode", Docker motorunun doğrudan normal bir Linux kullanıcısı olarak kurulmasıdır. Kurulumu zahmetlidir ancak en üst düzey güvenlik sağlar. Eğer sistemde bir güvenlik açığı olursa, hacker sadece o normal kullanıcının yetkilerine sahip olabilir. Sistemin kök dizinini asla ele geçiremez.

*(Kurulum detayı için resmi dokümantasyona bakabilirsiniz: `dockerd-rootless-setuptool.sh` aracı kullanılır).*

## 3. User Namespace Remapping (Kullanıcı Ad Alanı Yeniden Eşleme)

Rootless kullanmak zor geliyorsa, ikinci en iyi seçenek User Remapping'dir.
Bu yöntemde konteyner içindeki `root` (UID 0), host makinede örneğin `dokku` kullanıcısına (UID 100000) eşitlenir.
Yani konteyner içinde kodunuz kendini root sanır, istediği dosyayı yükler ama dışarıya bir sızıntı olduğunda Linux çekirdeği onu kısıtlı bir alt-kullanıcı olarak görür.

Aktivasyon için `daemon.json` (Bölüm 2) dosyasına şu satır eklenir:
```json
{
  "userns-remap": "default"
}
```

## 4. Yetkileri (Capabilities) Kırpmak

Linux'ta root demek, 40'a yakın farklı süper gücün (Capability) toplamı demektir (Saati değiştirmek, ağ ayarlarını oynamak, disk mount etmek vb.).
Docker varsayılan olarak bu 40 gücün sadece 14'üne izin verir. Ancak uygulamanız için sadece web servisi ayağa kaldırmak yetiyorsa, geri kalan 14 yetkiyi de silebilirsiniz (Drop).

```bash
docker run --rm \
  --cap-drop ALL \
  --cap-add NET_RAW \
  alpine ping -c 4 8.8.8.8
```
*(Bu Alpine konteynerinden tüm yetkileri aldık, sadece ağ paketleri gönderebilmesi (ping atabilmesi) için NET_RAW yetkisini geri verdik. Oldukça kısıtlı ve güvenli bir konteyner elde ettik!)*

## 5. Linux Güvenlik Profilleri: AppArmor ve Seccomp

Docker, Linux çekirdeğinin güvenlik modülleriyle mükemmel entegredir:

- **Seccomp (Secure Computing):** Konteynerin işletim sistemine gönderebileceği sistem çağrılarını (syscalls) kısıtlar. Linux'ta yaklaşık 330 syscall vardır, Docker varsayılan Seccomp profiliyle bunların 44 tanesini engeller (örn: reboot komutu). Özel bir JSON dosyasıyla uygulamanıza has bir Seccomp profili yazabilirsiniz:
  ```bash
  docker run --security-opt seccomp=/path/to/profile.json benim-imajim
  ```

- **AppArmor & SELinux:** Belirli dosya yollarına (örn: `/etc/shadow`) erişimi çekirdek seviyesinde yasaklar. Docker'ın varsayılan `docker-default` AppArmor profili çok etkilidir. 

## 6. Dosya Sistemi Optimizasyonu (Read-Only)

Eğer uygulamanız sadece okuma işlemi yapıyor ve diskteki dosyalarda kalıcı bir değişiklik yapmıyorsa, konteyneri salt-okunur (read-only) modda çalıştırın:

```bash
docker run --read-only nginx
```
Böylece hacker içeri sızsa bile sunucuya zararlı bir dosya indiremez veya var olan bir betiği (script) değiştiremez!

---
*Güvenlik duvarlarımızı aşılamaz hale getirdik. Sıradaki bölümde, teoriyi bırakıp tamamen pratiğe geçeceğiz: Gerçek dünyada bir Node.js ve React mimarisinin nasıl ayağa kaldırıldığını (Bölüm 10) inceleyeceğiz.*
