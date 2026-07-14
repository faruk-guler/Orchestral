# Bölüm 13: DevContainers ve Bulut Geliştirme Ortamları

Yazılımcıların en büyük zaman kayıplarından biri, takıma yeni katılan birinin bilgisayarına projeyi kurmasıdır (Node.js kur, doğru Python versiyonunu ayarla, veritabanını indir vs.).

**DevContainers (Development Containers)**, projenizin "Geliştirme Ortamını" (IDE, eklentiler, derleyiciler ve bağımlılıklar dahil) bir Docker konteyneri içine hapseden teknolojidir. Geliştirici sadece VS Code'u açar ve her şey (tam istediğiniz versiyonlarla) konteynerin içinde saniyeler içinde hazır olur.

## 1. DevContainer Mimarisi

DevContainer kullanmak için projenizin ana dizininde `.devcontainer` adında bir klasör ve içine `devcontainer.json` adında bir dosya oluşturmanız yeterlidir.

### Gelişmiş Bir `devcontainer.json` Örneği

```json
{
  "name": "Node.js & TypeScript Kurumsal Ortam",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:18",
  
  // 1. Özellikler (Features): İmaja hazır paketler ekler (Sıfırdan Dockerfile yazmaya gerek kalmaz)
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/aws-cli:1": {}
  },

  // 2. Çevresel Değişkenler
  "containerEnv": {
    "NODE_ENV": "development",
    "MY_CUSTOM_VAR": "deger"
  },

  // 3. Port Yönlendirmeleri
  "forwardPorts": [3000, 5432],
  "portsAttributes": {
    "3000": {
      "label": "React Frontend",
      "onAutoForward": "notify" // Port açıldığında VS Code'da bildirim göster
    }
  },

  // 4. VS Code Ayarları ve Eklentileri
  "customizations": {
    "vscode": {
      "settings": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "terminal.integrated.defaultProfile.linux": "bash"
      },
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "eamodio.gitlens",
        "ms-azuretools.vscode-docker"
      ]
    }
  },

  // 5. Yaşam Döngüsü (Lifecycle Scripts)
  "initializeCommand": "echo 'Host makinede (kendi bilgisayarınızda) konteyner başlamadan önce çalışır'",
  "onCreateCommand": "npm install -g pnpm", // Konteyner ilk yaratıldığında çalışır
  "updateContentCommand": "pnpm install",    // Kodunuz değişip konteyner yeniden başlatıldığında çalışır
  "postCreateCommand": "echo 'Konteyner tamamen hazır!'",

  // 6. Güvenlik ve Kullanıcı
  "remoteUser": "node"
}
```

## 2. Özelliklerin İncelenmesi (Deep Dive)

### A. Özellikler (Features)
Eskiden konteyner içine AWS CLI, GitHub CLI veya Terraform kurmak için devasa Dockerfile'lar yazardık. Şimdi `features` anahtarı ile Microsoft'un ve topluluğun hazırladığı binlerce hazır paketi sadece ismini yazarak projemize çekebiliyoruz.

### B. VS Code Özelleştirmeleri (Customizations)
Takımdaki bir kişi ESlint kullanırken diğeri kullanmıyorsa kodlar çakışır. `customizations > extensions` alanı sayesinde, bu projeyi açan herkesin VS Code'una **otomatik olarak** ESlint, Prettier ve GitLens gibi eklentiler yüklenir. İşe yeni başlayan biri için kusursuz bir deneyimdir!

### C. Yaşam Döngüsü Betikleri (Lifecycle Scripts)
Geliştirme ortamı açılmadan önce bazı veritabanı migration'ları çalıştırmak veya paket yüklemek (npm install) isteyebilirsiniz. `postCreateCommand` veya `updateContentCommand` tam olarak bu işe yarar.

## 3. DevContainers ile Docker Compose Kullanımı

Uygulamanız sadece Node.js'ten oluşmuyor; bir de PostgreSQL ve Redis'e ihtiyaç duyuyorsa, `devcontainer.json` dosyasına doğrudan `docker-compose.yml` dosyanızı bağlayabilirsiniz:

```json
{
  "name": "Full Stack Proje",
  "dockerComposeFile": "../docker-compose.dev.yml",
  "service": "app", // Compose dosyasındaki hangi servisin içine VS Code bağlanacak?
  "workspaceFolder": "/workspace"
}
```
Bu konfigürasyonla VS Code açıldığında, arka planda otomatik olarak `docker compose up` çalışır, veritabanları ayağa kalkar ve siz uygulamanızın ("app") içine girip kod yazmaya başlarsınız.

---
*DevContainers ile "Benim makinemde çalışıyordu" devrini tamamen kapattık; artık herkesin makinesinde aynı ortam çalışıyor. Peki ya yazdığımız bu kodlar ve hazırladığımız imajlar içinde bilmediğimiz güvenlik açıkları (Zafiyetler) varsa? Sıradaki bölüm: İleri Seviye Güvenlik Taraması (Snyk ve Trivy).*
