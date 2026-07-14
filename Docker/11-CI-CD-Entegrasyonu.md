# Bölüm 11: CI/CD Süreçlerine İleri Seviye Entegrasyon

Kodumuzu kendi bilgisayarımızda `docker build` komutuyla imaja dönüştürmeyi öğrendik. Ancak profesyonel iş dünyasında bu işlem **asla manuel** yapılmaz. 

Siz kodunuzu GitHub'a veya GitLab'a gönderdiğiniz (push) anda; sunuculardaki sistemlerin bu yeni kodu alıp, Docker imajını otomatik olarak derlemesi, güvenlik testlerinden geçirmesi ve yayınlaması beklenir. İşte bu sürece **CI/CD (Continuous Integration / Continuous Deployment)** adı verilir.

## 1. CI/CD'nin Docker İçin Aşamaları (Pipelines)

Standart bir Docker CI/CD boru hattı (pipeline) şu aşamalardan (stages) oluşur:
1. **Lint & Test:** Dockerfile'ın sözdizimi kontrol edilir (Örn: `hadolint` aracı ile) ve kodun Unit testleri çalıştırılır.
2. **Build:** `docker build` komutuyla imaj derlenir.
3. **Security Scan:** İmaj içindeki zafiyetler (Trivy/Snyk) taranır. (Bölüm 14'te detaylandırılacaktır).
4. **Push:** İmaj, Docker Hub veya Private Registry'e yüklenir (`docker push`).
5. **Deploy:** Canlı sunucuya SSH ile bağlanılıp `docker compose up -d` komutu veya Kubernetes'e `kubectl apply` tetiklenir.

---

## 2. GitHub Actions ile Otomatik Docker Build ve Push

GitHub, `.github/workflows/` dizini altına koyduğunuz YAML dosyalarını okuyarak sunucularında ücretsiz CI/CD çalıştırır.

### Profesyonel Bir `.github/workflows/docker-ci.yml` Dosyası

```yaml
name: Production Docker CI/CD

# Sadece main branch'ine push edildiğinde ve 'v1.x' tagleri atıldığında çalışır
on:
  push:
    branches: [ "main" ]
    tags: [ 'v*.*.*' ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    
    # 1. Kaynak Kodu Çek
    - name: Checkout Code
      uses: actions/checkout@v3

    # 2. QEMU Kurulumu (Multi-Arch Build İçin - Örn: Hem Intel hem Apple M1 için build alma)
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v2

    # 3. Docker Buildx Kurulumu (Gelişmiş Build Motoru)
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    # 4. Registry'e Giriş Yap (Şifreler GitHub Secrets'tan gelir)
    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    # 5. İmaj Metadatasını Ayarla (Otomatik etiketleme)
    - name: Extract metadata (tags, labels) for Docker
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: kullaniciadim/benim-appim

    # 6. İnşa Et ve Gönder (Multi-Arch)
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        platforms: linux/amd64,linux/arm64
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=registry,ref=kullaniciadim/benim-appim:buildcache
        cache-to: type=registry,ref=kullaniciadim/benim-appim:buildcache,mode=max
```

### Bu Pipeline'ın "Hero" Özellikleri:
- **Multi-Architecture Build:** `platforms: linux/amd64,linux/arm64` satırı sayesinde bu imaj hem standart sunucularda (Intel/AMD) hem de Apple Silicon / AWS Graviton (ARM) işlemcilerde kusursuz çalışır!
- **Build Cache (Önbellek):** `cache-from` ve `cache-to` satırları, bir sonraki build işleminin baştan başlamasını engeller ve süreyi 10 dakikadan 30 saniyeye düşürür. Cache, doğrudan Docker Registry üzerinde tutulur.

---

## 3. GitLab CI ile Entegrasyon

GitLab kullanıyorsanız, deponuzun ana dizininde `.gitlab-ci.yml` oluşturursunuz. GitLab CI'ın mimarisi, işlemlerin doğrudan bir Docker konteyneri içinde (Docker-in-Docker - DinD) çalıştırılmasına dayanır.

### Örnek `.gitlab-ci.yml`

```yaml
stages:
  - build

variables:
  # DinD (Docker in Docker) servisini kullanmak için gerekli ayarlar
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""

docker-build:
  image: docker:24.0.5
  stage: build
  services:
    - docker:24.0.5-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

## 4. Jenkins ile Entegrasyon (Declarative Pipeline)

Eğer şirkette geleneksel Jenkins kullanılıyorsa, projenizin içine `Jenkinsfile` adında bir dosya konulur.

```groovy
pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred')
    }
    stages {
        stage('Build Image') {
            steps {
                script {
                    app = docker.build("kullaniciadim/benim-appim:${env.BUILD_ID}")
                }
            }
        }
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-cred') {
                        app.push()
                        app.push("latest")
                    }
                }
            }
        }
    }
}
```

---
*İmajlarımız artık insan eli değmeden en güvenli ve en hızlı şekilde üretilip buluta çıkıyor. Peki bu kodlar canlı sunucuda çalışmaya başladığında, "Sistem şu an ne durumda? Hata veriyor mu?" sorularını nasıl yanıtlarız? Sıradaki bölümde "Loglama ve İzleme" (Monitoring) mimarilerine dalıyoruz.*
