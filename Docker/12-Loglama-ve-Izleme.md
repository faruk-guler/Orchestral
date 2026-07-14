# Bölüm 12: İleri Seviye Loglama ve İzleme (Monitoring)

Yüzlerce konteynerden oluşan devasa bir sisteminiz varsa, bir şeyler ters gittiğinde "Hangi konteyner çöktü? Neden çöktü? CPU mu yetmedi?" sorularının cevabını bulmak samanlıkta iğne aramaya benzer.

Docker, uygulamanızın `stdout` ve `stderr` (ekrana bastığı çıktılar) verilerini yakalar ve bunları **Log Sürücüleri (Log Drivers)** aracılığıyla işler. Ayrıca **Metrikler** ile anlık kaynak tüketimini raporlar.

## 1. Log Sürücüleri (Log Drivers) Tam Listesi

Varsayılan olarak Docker, logları host makinenin diskinde JSON formatında (`json-file`) saklar. Ancak disk dolarsa sunucu çöker (Bkz: Bölüm 2, log rotasyonu). Profesyonel sistemlerde loglar diskte tutulmaz, merkezi bir sunucuya (Elasticsearch, Splunk, AWS) gönderilir.

Bir konteyneri başlatırken `--log-driver` parametresiyle sürücüyü seçebilirsiniz:

- **`json-file` (Varsayılan):** Logları diskte JSON dosyası olarak tutar. `docker logs` komutuyla okunabilen tek sürücüdür.
- **`syslog`:** Logları doğrudan host makinenin syslog servisine yazar. Geleneksel Linux yöneticilerinin favorisidir.
- **`journald`:** Logları Linux'un modern `systemd-journald` servisine gönderir. `journalctl` komutuyla incelenir.
- **`fluentd`:** Çok popüler bir log toplayıcıdır. Konteynerin loglarını doğrudan başka bir sunucuda çalışan Fluentd veya Logstash (ELK Stack) servisine TCP/UDP ile fırlatır. Disk hiç dolmaz!
- **`awslogs`:** Logları doğrudan Amazon CloudWatch'a yazar.
- **`splunk`:** Kurumsal log analitik platformu Splunk'a HTTP Event Collector üzerinden log gönderir.

### Fluentd Örneği:
```bash
docker run -d \
  --log-driver=fluentd \
  --log-opt fluentd-address=192.168.1.100:24224 \
  --log-opt tag="front-end-app" \
  nginx
```

## 2. Docker Metrikleri: `docker stats` ve Formatlama

Konteynerlerin o anki CPU, RAM ve Ağ tüketimini canlı görmek için `docker stats` kullanırız.

**Gelişmiş Formatlama (Sadece İstediğiniz Verileri Çekmek İçin):**
Eğer kendi monitoring script'inizi yazıyorsanız, ekranı temiz bir tablo olarak almak isteyebilirsiniz:
```bash
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```
*Bu komut, canlı akışı durdurur (`--no-stream`) ve sadece Konteyner Adı, CPU % ve RAM Kullanımı sütunlarını verir.*

## 3. "Hero" Seviyesi: Prometheus ve Grafana Stack

`docker stats` sadece o anı gösterir, geçmişi (Dün gece saat 03:00'te CPU neden fırladı?) göstermez. Üretim (Production) ortamlarında geçmişe dönük izleme için endüstri standardı **cAdvisor + Prometheus + Grafana** üçlüsüdür.

### İzleme (Monitoring) Mimarisi:
1. **cAdvisor (Container Advisor):** Google tarafından geliştirilmiştir. Docker Daemon'a bağlanır ve her saniye tüm konteynerlerin CPU, RAM ve Disk I/O metriklerini okur.
2. **Prometheus:** cAdvisor'dan bu metrikleri çeker ve kendi özel zaman serisi (Time-Series) veritabanında yıllarca saklar.
3. **Grafana:** Prometheus'taki verileri okur ve devasa, harika görünümlü görsel grafiklere (Dashboard) dönüştürür.

### Tek Tıkla Kurulum (`docker-compose.monitoring.yml`):
```yaml
# Güncel Compose versiyonlarında 'version' satırına gerek yoktur.

services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
```
*(Bu dosyayı ayağa kaldırdığınızda `localhost:3000` adresinden Grafana'ya girip, uygulamanızın milisaniyelik tüm performans grafiklerini dev ekranlarda izleyebilirsiniz).*

---
*Kodumuzu yazdık, güvene aldık, yayınladık ve monitör ettik. Peki ya takıma yeni bir yazılımcı katıldığında onun bilgisayarına da günlerce kurulum yapacak mıyız? Hayır. Sıradaki bölüm: DevContainers (Bölüm 13).*
