# Bölüm 14: İmaj Zafiyet Taraması, SBOM ve Güvenlik Politikaları

Bölüm 9'da konteynerleri çalışırken (Runtime) nasıl koruyacağımızı (Rootless, Seccomp vs.) öğrendik. Ancak en büyük tehlike genellikle kodun veya kullandığımız imajın içine sızmış olan gizli güvenlik açıklarıdır (Vulnerabilities).

Örneğin `FROM node:14` yazdınız. Node 14'ün içinde barındırdığı eski bir OpenSSL kütüphanesinde kritik bir açık olabilir ve hackerlar bu açığı kullanarak sisteminize sızabilir. Bunu engellemek için **İmaj Tarama (Image Scanning)** araçları kullanırız.

## 1. Tarama Araçları: Trivy ve Snyk

Sektörde Docker imajlarını taramak için en çok kullanılan iki güçlü araç vardır: **Trivy** (Açık kaynaklı ve çok hızlı) ve **Snyk** (Kurumsal ve geliştirici odaklı).

### Trivy Kullanımı
Trivy, imajın içindeki işletim sistemi paketlerini (apt, apk, yum) ve uygulama bağımlılıklarını (npm, pip, pom) tarar ve bilinen CVE (Ortak Güvenlik Açıkları) veritabanıyla karşılaştırır.

```bash
# Sadece Kritik (CRITICAL) ve Yüksek (HIGH) seviyeli açıkları göster
trivy image --severity HIGH,CRITICAL benim-imajim:latest
```

### Docker'ın İçindeki Gömülü Tarayıcı
Docker Desktop kullanıyorsanız veya `docker scan` eklentisi yüklüyse, Docker arka planda Snyk motorunu kullanarak imajınızı tarayabilir:
```bash
docker scan benim-imajim:latest
```

## 2. SBOM (Software Bill of Materials) Nedir?

Son yıllarda siber güvenliğin en büyük konusu olan **SBOM (Yazılım Malzeme Listesi)**, imajınızın içinde tam olarak hangi kütüphanelerin, hangi versiyonlarla bulunduğunun çıkarıldığı resmi bir "içindekiler" belgesidir.

Tıpkı marketten aldığınız bir kekin arkasındaki içindekiler (şeker, un, aroma) listesi gibi, uygulamanızın da içindekiler listesi olmalıdır ki ileride "Log4j" gibi global bir kriz patladığında, uygulamanızda o kütüphanenin olup olmadığını saniyeler içinde bulabilesiniz.

```bash
# Trivy ile imajın SBOM belgesini (SPDX veya CycloneDX formatında) üretme
trivy image --format spdx-json --output sbom.json benim-imajim:latest
```
Bu `sbom.json` dosyasını CI/CD süreçlerinizde arşive kaydederek, kurumunuzun yasal uyumluluk (Compliance) şartlarını yerine getirmiş olursunuz.

## 3. İleri Seviye: Policy (Kural) Yazımı ve Zafiyetleri Yoksayma

Bazen tarama araçları bir zafiyet bulur ama siz o paketi projenizde hiç kullanmıyorsunuzdur (False Positive) veya yamanın çıkmasına daha aylar vardır. CI/CD boru hattınızın bu yüzden durmasını (fail olmasını) istemezsiniz. 

Bu durumda **Ignore (Yoksayma)** kuralları veya **Güvenlik Politikaları (Security Policies)** yazarız.

### Trivy İçin `.trivyignore` Dosyası
Projenizin ana dizinine `.trivyignore` adında bir dosya açıp, yoksaymak istediğiniz CVE (hata) kodlarını yazabilirsiniz:
```text
# Bu zafiyet sadece Windows'u etkiliyor, biz Alpine Linux kullanıyoruz, yoksay.
CVE-2021-34527

# Bu paketi sildiğimiz için hata asılsız
CVE-2022-12345
```
Trivy tarama yaparken bu dosyadaki ID'leri görünce görmezden gelir ve CI/CD pipeline'ınız yeşil yanarak (başarılı) devam eder.

### OPA (Open Policy Agent) ve Rego ile Kural Yazımı
Daha büyük şirketlerde kurallar kod olarak yazılır (Policy as Code). Trivy, **Rego** diliyle yazılmış OPA kurallarını destekler.

Örneğin: *"İmajın içinde root kullanıcısına izin veriliyorsa veya imaj `ubuntu:18.04` tabanlıysa build işlemini DURDUR"* gibi gelişmiş mantıksal kurallar yazabilir ve tarama aracına entegre edebilirsiniz.

---
*Güvenlik açıklarını kodumuz yayınlanmadan önce yakalamayı da öğrendik. Şimdi kendi imajlarımızı Docker Hub (genel internet) yerine şirketimizin kendi kapalı ağındaki özel bir depoda (Registry) nasıl tutacağımızı öğreneceğiz (Bölüm 15).*
