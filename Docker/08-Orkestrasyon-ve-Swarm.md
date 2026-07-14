# Bölüm 8: Docker Swarm ve Konteyner Orkestrasyonu

Bölüm 7'ye kadar öğrendiğimiz her şey (Docker Run, Compose, Volume, Network) **tek bir host (bilgisayar veya sunucu)** üzerinde çalışmak içindir. 

Ancak uygulamanız günde milyonlarca ziyaretçi almaya başladığında, tek bir sunucunun CPU'su veya RAM'i yetersiz kalacaktır. Uygulamanızı 10 farklı sunucuya (Node) dağıtmanız (yatay ölçekleme - horizontal scaling) gerekir. İşte birden fazla sunucuyu tek bir devasa bilgisayarmış gibi yönetmeye **Orkestrasyon (Orchestration)** denir.

Docker'ın kendi içinde gömülü gelen (ekstra kurulum gerektirmeyen) resmi orkestrasyon aracı **Docker Swarm**'dır.

## 1. Swarm Mimarisi ve Temel Kavramlar

Swarm'da sunucular (Node'lar) ikiye ayrılır:
1. **Manager Node (Yönetici):** Kümeyi (Cluster) yönetir, görevleri dağıtır. Arka planda **Raft Consensus** algoritmasını kullanarak birden fazla manager olsa bile kararların oylamayla, tutarlı alınmasını sağlar. (Üretim ortamında tek sayı olmalıdır: 3, 5 veya 7 manager).
2. **Worker Node (İşçi):** Manager'dan gelen emirleri uygular ve konteynerleri (Swarm dilinde: Task/Görev) çalıştırır.

### Küme (Cluster) Kurulumu
1. **Manager sunucusunda:** Swarm modunu aktif edin.
   ```bash
   docker swarm init --advertise-addr <MANAGER_IP>
   ```
   *Bu komut size uzun bir `docker swarm join --token ...` komutu verecektir.*

2. **Worker sunucularında:** O uzun komutu çalıştırarak kümeye katılın.

## 2. Servisler (Services) ve Dağıtım Türleri

Swarm modunda artık `docker run` kullanmayız. Onun yerine "Servis" (Service) oluştururuz. Servis, bir imajın kaç kopyasının (replica) çalışacağının soyutlanmış halidir.

### A. Replicated (Kopyalanmış) Servisler
Belirli bir sayıda kopya istersiniz ve Swarm bunları müsait olan node'lara dağıtır.

```bash
docker service create \
  --name web-sunucu \
  --replicas 5 \
  -p 80:80 \
  nginx
```
*(Bu komut, 5 farklı Nginx konteynerini sunucularınız arasına dağıtır).*

### B. Global Servisler
Her bir fiziksel sunucuda (Node) **kesinlikle 1 adet** çalışması gereken servislerdir (Örneğin Anti-virüs yazılımı, log toplayıcı ajanlar).

```bash
docker service create \
  --name log-toplayici \
  --mode global \
  fluentd
```

## 3. İleri Seviye Swarm Yetenekleri

### A. Ingress Routing Mesh (Yönlendirme Ağı)
Swarm'ın en efsanevi özelliğidir. Yukarıdaki Nginx örneğinde `-p 80:80` yazdık. Swarm'daki **tüm** sunucuların (içinde Nginx çalışsın veya çalışmasın) 80 portu açılır. 

Ziyaretçi kümeye **herhangi bir sunucunun** 80 portundan bağlandığında, Routing Mesh bu isteği alır ve arka planda Nginx'in **gerçekten çalıştığı** sunucuya otomatik yönlendirir. Dahili bir Load Balancer (Yük Dengeleyici) görevi görür.

### B. Overlay Network (Şifreli Bulut Ağı)
Farklı fiziksel sunuculardaki konteynerler nasıl iletişim kurar? Swarm, VXLAN teknolojisini kullanarak tüm sunucuları kapsayan devasa bir **Overlay Network** kurar.
Eğer finans veya sağlık projesi yapıyorsanız, bu ağ üzerinden geçen verileri şifreleyebilirsiniz:
```bash
docker network create \
  --opt encrypted \
  --driver overlay \
  benim-guvenli-agim
```
Böylece sunucular arasındaki trafik (ISP seviyesinde bile) AES algoritmasıyla donanımsal olarak şifrelenir.

### C. Sıfır Kesintiyle Güncelleme (Rolling Update)
Uygulamanızın V1 versiyonundan V2 versiyonuna geçerken sitenizin çökmesini istemezsiniz. Swarm, konteynerleri tek tek kapatıp yenisini açarak (Rolling Update) kesintisiz geçiş sağlar.

```bash
docker service update \
  --image benim-app:v2 \
  --update-parallelism 2 \
  --update-delay 10s \
  web-sunucu
```
*(Aynı anda en fazla 2 konteyneri güncelle, her güncelleme arası 10 saniye bekle).*

### D. Placement Constraints (Yerleştirme Kısıtlamaları)
"Veritabanı servisim sadece SSD diski olan sunucularda çalışsın" demek isterseniz Node'lara etiket (label) ekleyebilir ve kısıtlama koyabilirsiniz:
```bash
# Node'a etiket ekle
docker node update --label-add disk=ssd worker-1

# Servisi oluştururken kısıtlama koy
docker service create \
  --name veri-deposu \
  --constraint 'node.labels.disk == ssd' \
  redis
```

---
*Birden fazla sunucuyu devasa bir süper bilgisayara dönüştüren Swarm'ı öğrendik. Ancak sistem büyüdükçe güvenlik açıkları da büyür. Sıradaki bölümde, Docker ortamımızı dışarıdan gelebilecek hacker saldırılarına karşı korumanın "Hardening" yollarını (Bölüm 9) öğreneceğiz.*
