## 1. Genel Bakış

*   **Saldırı Adı:** ImageRunner: Privilege Escalation in GCP Cloud Run
*   **Kategori:** Privilege Escalation, IAM Abuse, Identity Exhaustion, Side-channel Token Extraction
*   **Hedef Cloud Platformları:** Google Cloud Platform (GCP)
*   **Etkilenen Servisler:** Google Cloud Run, Google Kubernetes Engine (GKE) - *Servis Hesabı paylaşımı nedeniyle*

## 2. Teknik Özet

**ImageRunner**, bir container runtime veya uygulama seviyesindeki zafiyetin (RCE veya SSRF), Google Cloud Run üzerindeki çalışan bir container'dan, container'a atanan **Service Account (SA)** kimlik bilgilerinin ele geçirilmesiyle sonuçlandığı bir saldırı zinciridir. Saldırının temelinde, GCP Metadata Server'ın (169.2matadata.google.internal) kullanımı ve bu servis hesabına atanan aşırı yetkili (over-privileged) IAM rollerinin kötüye kullanılması yatar. 

**Temel Risk:** Bir saldırganın, sınırlı bir container ortamından çıkıp, tüm GCP projesinde geniş kapsamlı yetkilere (örneğin; `Editor`, `Storage Admin` veya `Compute Admin`) ulaşması ve veri sızıntısı veya altyapı sabotajı gerçekleştirmesi.

## 3. Güvenlik Modeli ve Arka Plan

GCP Cloud Run, "Serverless" bir model sunar. Kullanıcılar altyapıdan (node, OS, runtime) sorumlu değildir; ancak **Identity (Kimlik)** yönetiminden tamamen sorumludur.

*   **IAM ve Service Accounts:** Her Cloud Run servisi, bir "Identity" (Service Account) ile çalışır. Bu SA, servis içindeki kodun GCP API'leri ile konuşmasını sağlayan anahtardır.
*   **Metadata Server:** GCP'deki tüm compute kaynakları, kendilerine ait kimlik bilgilerini (Access Token) almak için `http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token` adresine erişebilir.
*   **Zafiyetin Temeli:** Güvenlik modeli, container'ın içindeki kodun "güvenilir" olduğunu varsayar. Eğer kodun içinde bir açık varsa (SSRF gibi), saldırgan bu iç mekanizmayı kullanarak "Identity"yi dışarı sızdırabilir.

## 4. Teknik Detaylar ve Çalışma Prensibi

Saldırı, tipik olarak bir **Server-Side Request Forgery (SSRF)** veya **Remote Code Execution (RCE)** ile başlar.

### Adım Adط Adım Mekanizma:

1.  **Initial Access:** Saldırgan, Cloud Run üzerinde çalışan web uygulamasına bir payload gönderir (Örn: `http://example.com/proxy?url=...`).
2.  **Metadata Interrogation:** Saldırgan, uygulamanın içinden GCP Metadata Server'a istek atar.
    *   **Kritik Header:** `Metadata-Flavor: Google` başlığı olmadan istek reddedilir. Saldırgan bu başlığı manipüle edebilmelidir.
3.  **Token Extraction:** Aşağıdaki API çağrısı ile OAuth2 Access Token ele geçirilir:
    ```bash
    curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
    ```
4.  **Response Analysis:**
    ```json
    {
      "access_token": "ya29.a0AfH6SM...",
      "expires_in": 3599,
      "token_type": "Bearer"
    }
    ```
5.  **Privilege Escalation:** Ele geçirilen token, saldırganın kendi makinesinde `gcloud` veya `curl` ile kullanılarak projedeki diğer kaynaklara (GCS Buckets, Secret Manager, Cloud SQL) erişim sağlanır.

## 5. Gerekli Yetkiler 

Saldırganın nihai hedefe ulaşması için şu minimum koşullar gereklidir:

*   **Uygulama Seviyesinde:** SSRF veya RCE zafiyeti barındıran bir web endpoint.
*   **IAM Seviyesinde:** Cloud Run servis hesabının (Service Account) sahip olduğu rollerin, saldırganın istediği aksiyonu yapmaya yetecek genişlikte olması (Örn: `roles/editor`, `roles/storage.admin`).
    *   *Önemli:* Varsayılan olarak kullanılan `Compute Engine default service account` genellikle çok geniş yetkilere sahiptir ve en büyük riski oluşturur.

## 6. Detaylı Saldırı Senaryosu

| Aşama | Eylem | Teknik Detay |
| :--- | :--- | :--- |
| **1. Initial Access** | Web Zafiyeti İstismarı | Saldırgan, Cloud Run üzerindeki bir PDF oluşturma servisinde SSRF açığı bulur. |
| **2. Reconnaissance** | Metadata Taraması | `curl` ile metadata üzerinden projenin Service Account listesi ve mevcut roller çekilir. |
| **3. Privilege Escalation** | Token Çalma | Metadata Server üzerinden `access_token` çekilir. |
| **4. Persistence** | Yeni Kimlik Oluşturma | Saldırgan, `iam.serviceAccounts.createKey` yetkisine sahipse, kendine yeni bir Service Account Key oluşturur. |
| **5. Exfiltration** | Veri Sızıntısı | `gsutil cp gs://hassas-veriler/musteri_listesi.csv .` komutu ile veriler dışarı taşınır. |

## 7. Etkilenen Ortamlar

*   **Cloud Run:** Doğrudan etkilenen birincil servis.
*   **Cloud Functions:** Benzer metadata yapısını kullandığı için risk altında.
*   **GKE (Google Kubernetes Engine):** Pod içindeki Service Account yetkileri üzerinden.
cap
*   **Multi-cloud Etkisi:** Düşük (Bu spesifik teknik GCP-native bir metadata istismarıdır).

## 8. Saldırı Karakteristikleri

*   **Tespit Edilme Zorluğu:** **Yüksek.** İstekler, uygulamanın kendi trafiği gibi göründüğü için standart WAF kuralları tarafından yakalanması zordur.
*   **Gürültü Seviyesi:** **Düşük (Stealthy).** Tek bir HTTP isteği (SSRF) tüm altyapıyı tehlikeye atabilir.
*   **Zincirleme Kullanım:** Çok yüksek. Token çalındıktan sonra saldırı, tamamen standart Cloud API çağrılarına dönüşür.

## 9. Detection 

### Cloud Audit Logs Analizi
GCP Cloud Audit Logs üzerinde şu anomaliye dikkat edilmelidir:
*   **Anormal IP Kaynakları:** Cloud Run servislerinin IP aralıklarından gelen, ancak alışılmadık `iam.serviceAccounts.getAccessToken` veya `secretmanager.versions.access` çağrıları.
*   **Metadata Header Tespiti:** Eğer bir WAF veya API Gateway kullanılıyorsa, `Metadata-Flavor: Google` içeren dış dünyadan gelen (External) isteklerin bloklanması ve loglanması.

### SIEM Use-Case Örneği:
`Event: AccessToken Generation` + `Source: Cloud Run Service Account` + `Caller: External IP` $\rightarrow$ **CRITICAL ALERT**

## 10. Incident Response & Remediation

1.  **Containment (Karantina):** İlgili Cloud Run servisini derhal durdurun veya trafiği kesin.
2.  **Credential Revocation:** Ele geçirilen Service Account için tüm aktif session'ları sonlandırın ve varsa bağlı olan Service Account Key'lerini silin.
3.  **Investigation:** Cloud Audit Logs üzerinden token'ın hangi saat aralığında kullanıldığını ve hangi kaynaklara (Bucket, DB vb.) eriştiğini belirleyin.
4.  **Remediation (Düzeltme):** Uygulama seviyesindeki SSRF/RCE açığını kapatın.

## 11. Önleme Stratejileri

*   **Principle of Least Privilege (PoLP):** Servis hesaplarına sadece ihtiyaç duydukları minimum izinleri verin (Örn: Sadece `storage.objectViewer`).
*   **Workload Identity:** Mümkünse statik anahtarlar yerine Workload Identity kullanarak kimlik yönetimini güçlendirin.
*   **Network Perimeters:** Cloud Armor veya benzeri WAF çözümleriyle outbound (dışa giden) istekleri ve anormal patternleri izleyin.
*   **VPC Service Controls:** Hassas verilerin bulunduğu servisleri, yetkisiz erişime karşı VPC sınırları içine alın.

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
