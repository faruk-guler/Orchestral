# Bölüm 10: Gerçek Dünya Projeleri ve Mimariler

Teorik bilgileri öğrendik, ağları bağladık ve güvenliği sağladık. Şimdi "Zero to Hero" vizyonunun "Hero" aşamasına geldik: Üretim ortamında (Production) çalışan, kurumsal seviyede projeleri nasıl mimarilendiririz?

Bu bölümde 2 farklı tam teşekküllü mimariyi inceleyeceğiz: Multi-Stage Frontend ve Ters Vekil (Reverse Proxy) kullanan Backend.

## Proje 1: Multi-Stage Build ile React (veya Vue/Angular) Uygulaması

Modern Frontend projelerinde kaynak kodunuz binlerce dosya ve megabaytlarca `node_modules` içerir. Ancak canlı sunucuda bunların hiçbirine ihtiyacınız yoktur; tek ihtiyacınız olan derlenmiş (build edilmiş) saf HTML, CSS ve JS dosyalarıdır.

Bu yüzden "Multi-Stage Build" kullanırız. 

### `Dockerfile.prod`
```dockerfile
# --- BİRİNCİ AŞAMA: BUILDER (Derleyici) ---
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
# Sadece derleme (build) için gerekli paketleri kur
RUN npm ci

COPY . .
# Kaynak koddan optimize edilmiş static dosyaları üretir (genelde /app/build veya /app/dist klasörü)
RUN npm run build 

# --- İKİNCİ AŞAMA: RUNNER (Sunucu) ---
# Üretime (Production) gidecek olan asıl ince imaj
FROM nginx:alpine

# Güvenlik: Kök dizini (rootfs) okuma yetkilerini ayarla vs.
WORKDIR /usr/share/nginx/html

# Mevcut default Nginx sayfasını sil
RUN rm -rf ./*

# BİRİNCİ aşamadan sadece derlenmiş klasörü alıp buraya kopyala (Kalan her şeyi çöpe at!)
COPY --from=builder /app/build .

# İsteğe bağlı: Nginx için özel konfigürasyonunuz varsa kopyalayın
# COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
*Sonuç:* Belki de 1GB sürecek olan bir Node.js imajı yerine, sadece 20MB'lık ve içinde sadece saf statik dosyaları barındıran inanılmaz hızlı ve güvenli bir imaj elde edersiniz.

---

## Proje 2: Reverse Proxy (Ters Vekil) ve Mikroservis Mimarisi

Büyük projelerde tek bir sunucuda hem React Frontend, hem Node.js API, hem Python API çalışabilir. Dışarıdan gelen `app.com/api` isteklerinin Node.js'e, `app.com/ai` isteklerinin Python'a, kalan isteklerin ise React'a gitmesi gerekir. 

Bunu bir **Ters Vekil (Reverse Proxy)** ile çözeriz. Sektörde bunun için genellikle **Nginx** veya **Traefik** kullanılır.

### Gelişmiş `docker-compose.prod.yml` Mimarisi

```yaml
# Güncel Compose versiyonlarında 'version' satırına gerek yoktur.

services:
  # Ters Vekil (Kapıdaki Güvenlik ve Yönlendirici)
  reverse-proxy:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443" # SSL/TLS Sertifikaları proxy üzerinde tutulur
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl-certs:/etc/ssl/certs:ro
    networks:
      - public_net
    depends_on:
      frontend:
        condition: service_started
      backend-api:
        condition: service_started

  # Kullanıcı Arayüzü (Dışarıya port açmaz, sadece proxy ile konuşur)
  frontend:
    image: benim-react-app:prod
    networks:
      - public_net
    # Port yok! Dışarıdan doğrudan erişilemez, proxy üzerinden gelinir.

  # Arka Plan Servisi (Node.js)
  backend-api:
    image: benim-nodejs-api:prod
    environment:
      - DB_HOST=veritabani
      - NODE_ENV=production
    networks:
      - public_net
      - private_db_net # Sadece veritabanıyla konuşacağı güvenli ağ

  # Veritabanı (İnternete tamamen kapalı, şifreli)
  veritabani:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_pass
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - private_db_net
    secrets:
      - db_pass

volumes:
  db-data:

networks:
  public_net:
    driver: bridge
  private_db_net: # İnternet erişimi olmayan iç ağ
    internal: true

secrets:
  db_pass:
    file: ./secrets/db_password.txt
```

### Bu Mimarinin "Hero" Özellikleri:
1. **İç Ağ Güvenliği (Internal Network):** `private_db_net` ağı `internal: true` olarak işaretlenmiştir. Bu ağdaki veritabanı, internetten gelen hiçbir paketi alamaz ve internete veri gönderemez. Dünyanın en güvenli veritabanıdır. Sadece `backend-api` ona erişebilir.
2. **Reverse Proxy İzolasyonu:** Frontend ve Backend konteynerleri dışarıya port açmaz (`ports: - "3000:3000"` gibi bir satır YOKTUR). Bütün trafik `reverse-proxy` (Nginx) üzerinden geçer. Nginx bu trafiği süzer, SSL sertifikasını çözer ve güvenli paketleri ilgili konteynere iletir.
3. **Secrets Kullanımı:** Veritabanı şifresi compose dosyasına metin olarak (plaintext) yazılmamış, dışarıdaki bir dosyadan Docker Secrets ile içeri enjekte edilmiştir.

---
*Gördüğünüz gibi Docker'ı temel seviyede bilmek sadece uygulamayı çalıştırmaktır. "Hero" seviyesinde bilmek ise o uygulamanın nefes aldığı ağı, veritabanını ve proxy'i kusursuz bir mimariyle örebilmektir.*

*Sıradaki bölümde, bu harika üretim (production) kodumuzu biz uyurken bile sunucuya otomatik taşıyacak olan CI/CD süreçlerini (Bölüm 11) işleyeceğiz.*
