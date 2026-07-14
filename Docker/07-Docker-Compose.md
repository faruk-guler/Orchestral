# Bölüm 7: Docker Compose ve İleri Seviye Konfigürasyonlar

Günümüzde hiçbir uygulama tek başına (sadece web sunucusu olarak) çalışmaz. Bir React frontend'i, bir Node.js backend'i, bir PostgreSQL veritabanı ve Redis cache'i aynı anda çalıştırmanız gerekebilir. 

Tüm bunları tek tek `docker run ...` komutlarıyla ayağa kaldırmak, ağlara bağlamak ve volume'lerini ayarlamak bir kabusa dönüşür. **Docker Compose**, bu karmaşayı tek bir YAML dosyasıyla çözmenizi sağlayan resmi Docker aracıdır.

## 1. Temel Compose Mimarisi

Projenizin ana dizininde `docker-compose.yml` (veya `compose.yaml`) adında bir dosya oluşturursunuz.

```yaml
# Güncel Compose versiyonlarında 'version' satırına gerek yoktur.

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: gizlisifre
```
Bu dosyanın bulunduğu klasörde terminale sadece `docker compose up -d` yazmanız, yukarıdaki iki sistemi aynı ağda kurup bağlamaya yeterlidir. 
*(Kapatmak için: `docker compose down`)*

---

## 2. İleri Seviye Compose Direktifleri (Kapsamlı Liste)

"Zero to Hero" seviyesinde bir `docker-compose.yml` dosyası yazarken kullanabileceğiniz tüm güçlü parametreler şunlardır:

### A. Bağımlılıklar (Depends On)
Bir servisin diğerinden önce başlamasını garanti eder. 
```yaml
services:
  backend:
    image: benim-api:latest
    depends_on:
      db:
        condition: service_healthy # Sadece db servisi BAŞLADIĞINDA değil, SAĞLIKLI olduğunda backend'i başlat.
      redis:
        condition: service_started
```

### B. Sağlık Kontrolü (Healthcheck)
Servisin gerçekten çalışıp çalışmadığını (sadece ayakta olup olmadığını değil, HTTP 200 dönüp dönmediğini) test eder.
```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s       # Her 10 saniyede bir kontrol et
      timeout: 5s         # 5 saniye cevap gelmezse fail say
      retries: 5          # 5 kez üst üste fail verirse konteyneri sağlıksız ilan et
      start_period: 30s   # İlk 30 saniye boyunca hataları görmezden gel (Veritabanının açılma süresi)
```

### C. Profiller (Profiles)
Bazı servisleri sadece geliştirme (dev) veya sadece test ortamında çalıştırmak isteyebilirsiniz.
```yaml
services:
  frontend:
    image: react-app
    profiles: ["frontend", "dev"]
    
  debug-tool:
    image: phpmyadmin
    profiles: ["debug"]
```
*(Kullanımı: Sadece debug aracını başlatmak için: `docker compose --profile debug up`)*

### D. Restart Politikaları
Konteyner çökerse ne olacağı:
```yaml
    restart: always           # Her durumda yeniden başlat
    # restart: on-failure     # Sadece hata koduyla kapanırsa başlat
    # restart: unless-stopped # Manuel durdurana kadar başlat
```

### E. Gizli Veriler ve Yapılandırmalar (Secrets & Configs)
Şifreleri environment variable (çevresel değişken) olarak vermek yerine, Docker Secrets üzerinden RAM'e enjekte etmek en güvenli yoldur.
```yaml
services:
  app:
    image: my-app
    secrets:
      - db_password
    configs:
      - my_config

secrets:
  db_password:
    file: ./secrets/db_password.txt # Bu dosyanın içindeki metni konteynere şifreli iletir

configs:
  my_config:
    file: ./nginx.conf
```

### F. Kaynak Sınırlandırmaları (Deploy > Resources)
Konteynerin kullanabileceği CPU ve RAM'i kısıtlamak için `deploy` anahtarı kullanılır (Özellikle Swarm mode ve v3 formatında).
```yaml
services:
  agir-uygulama:
    image: java-app
    deploy:
      resources:
        limits:
          cpus: '0.50'     # Maksimum yarım çekirdek
          memory: 512M     # Maksimum 512MB RAM
        reservations:
          cpus: '0.25'     # Garanti edilen çeyrek çekirdek
          memory: 256M     # Garanti edilen 256MB RAM
```

### G. Build Parametreleri
Sadece hazır imajları (image: nginx) kullanmak zorunda değilsiniz. Compose, sizin yerinize Dockerfile'ı build edebilir.
```yaml
services:
  webapp:
    build:
      context: ./src             # Dockerfile'ın bulunduğu dizin
      dockerfile: Dockerfile.dev # Eğer ismi farklıysa
      args:                      # Build ARG değişkenleri
        - NODE_VERSION=18
      target: builder            # Multi-stage build'de sadece belli bir aşamayı build et
```

---

## 3. Komutlar ve Püf Noktaları

- **Logları Takip Etmek:**
  ```bash
  docker compose logs -f --tail 100 webapp
  ```
- **İmajları Yeniden Build Etmek:** Kodunuzu değiştirdiniz ama `up` yaptığınızda eski imaj çalışıyor. Yenisini inşa etmeye zorlamak için:
  ```bash
  docker compose up -d --build
  ```
- **Farklı Dosya İsimleri:**
  Eğer dosya adınız `docker-compose.yml` değilse (örneğin `docker-compose.prod.yml`):
  ```bash
  docker compose -f docker-compose.prod.yml up -d
  ```

---
*Compose sayesinde yüzlerce konteyneri tek dosyada yönettik. Ancak bu konteynerler hep **tek bir makinede (sizin laptop'ınızda veya tek bir sunucuda)** çalışıyor. Peki ya yükler artarsa ve 10 farklı sunucudan oluşan bir kümeye (cluster) ihtiyaç duyarsak? Sıradaki bölüm: Docker Swarm.*
