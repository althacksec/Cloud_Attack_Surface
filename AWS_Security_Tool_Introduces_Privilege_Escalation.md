## 1. Genel Bakış

*   **Saldırı Adı:** AWS-Security-Tool-Introduces-Privilege-Escalation-Risk (Güvenlik Araçları Aracılığıyla Yetki Yükseltme)
*   **Kategori:** Privilege Escalation (Yetki Yükseltlikme), IAM Abuse, Misconfiguration Abuse
*   **Hedef Cloud Platformları:** AWS (Birincil), Azure, GCP (Konsept benzerlik gösterir)
*   **Kritiklik Seviyesi:** **CRITICAL**

## 2. Teknik Özet

Bu saldırı tekniği, bir zafiyetten ziyade bir **güvenlik mimarisi tasarım hatasına** dayanır. Kurulu olan üçüncü taraf güvenlik tarayıcıları (vulnerability scanners), maliyet/performans veya "çalışma kolaylığı" amacıyla AWS üzerinde aşırı yetkilendirilmiş (over-privileged) IAM rolleri ile çalıştırılır. 

Saldırgan, bu güvenlik aracının çalıştığı compute instance (EC2, Lambda, vb.) üzerinde bir sızma gerçekleştirdiğinde, aracın sahip olduğu geniş kapsamlı IAM yetkilerini devralarak AWS ortamında tam yetki (AdministratorAccess) elde edebilir. Saldırı, "güvenlik aracının" meşru hareketleri gibi göründüğü için tespit edilmesi son derece zordur.

## 3. Güvenlik Modeli ve Arka Plan

AWS IAM (Identity and Access Management) modeli, **"Explicit Deny"** ve **"Least Privilege"** prensipleri üzerine kuruludur. Ancak, güvenlik araçları (örneğin; bir CSPM veya Nessus gibi bir tarayıcı), bulut ortamındaki tüm kaynakları okuyabilmek ve zafiyetleri raporlayabilmek için geniş `ReadOnlyAccess` veya daha tehlikeli olan `FullAccess` yetkilerine ihtiyaç duyarlar.

**Risk Mekanizması:**
Saldırı, IAM'in **"Trust Relationship"** ve **"Permission Boundary"** eksikliklerini istismar eder. Eğer güvenlik aracı `iam:PutRolePolicy` veya `iam:CreatePolicyVersion` gibi "yazma" yetkilerine sahipse, bu araç artık bir "proxy" saldırı vektörüne dönüşür.

## 4. Teknik Detaylar ve Çalışma Prensibi

Saldırı, güvenlik aracının IAM rolündeki kritik izinlerin suistimal edilmesiyle gerçekleşir.

### Adım Adım Mekanizma:
1.  **Initial Access:** Saldırgan, güvenlik aracının çalıştığı EC2 instance üzerinde bir zafiyet (örn. SSRF, RCE) bulur.
2.  **Credential Exfiltration:** Saldırgan, EC2 Instance Metadata Service (IMDSv2) üzerinden aracın IAM Role geçici kimlik bilgilerini (Access Key, Secret Key, Session Token) çalar.
3.  **Privilege Escalation via Policy Manipulation:** Saldırgan, aracın sahip olduğu `iam:CreatePolicyVersion` iznini kullanarak, mevcut bir politikayı (policy) kendi lehine günceller.

### Örnek Tehlikeli JSON Policy:
Aşağıdaki policy, bir güvenlik aracına verilmiş "yanlış" bir yetki örneğidir:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:Get*",
                "iam:List*",
                "iam:CreatePolicyVersion",
                "iam:SetDefaultPolicyVersion",
                "iam:PutRolePolicy"
            ],
            "Resource": "*"
        }
    ]
}
```
*Buradaki `iam:CreatePolicyVersion` izni, saldırganın mevcut bir politikaya `AdministratorAccess` eklemesine olanak tanır.*

## 5. Gerekli Yetkiler 

Saldırganın başarılı olması için aşağıdaki minimum yetki setine sahip bir "identity" (kimlik) ele geçirmesi gerekir:

*   **Minimum İzinler:**
    *   `iam:CreatePolicyVersion` (Politika versiyonlarını manipüle etmek için)
    *   `iam:SetDefaultPolicyVersion` (Yeni versiyonu aktif etmek için)
    *   `iam:PutRolePolicy` (Inline policy eklemek için)
    *   `iam:PassRole` (Yeni oluşturulan yetkili rolleri servislere atamak için)
*   **Yanlış Konfigürasyonlar:**
    *   IMDSv1 kullanımı (SSRF saldırılarını kolaylaştırır).
    *   Güvenlik araçlarının `AdministratorAccess` ile çalıştırılması.
    *   Permission Boundary (Yetki Sınırı) eksikliği.

## 6. Detaylı Saldırı Senaryosu 

**Senaryo: "The Scanner's Shadow"**

1.  **Initial Access (Giriş):** Bir saldırgan, şirketin AWS ortamında çalışan bir "Vulnerability Scanner" EC2 instance'ındaki bir web arayüzü zafiyetini (RCE) kullanarak içeri sızar.
2.  **Reconnaissance (Keşif):** Saldırgan `aws sts get-caller-identity` komutuyla mevcut rolün `Scanner-Role` olduğunu ve geniş yetkilere sahip olduğunu anlar.
3.  **Privilege Escalation (Yetki Yüksekletme):**
    *   Saldırgan, `Scanner-Role` üzerinden mevcut bir politikayı (`Scanner-Policy`) hedef alır.
    *   `iam:CreatePolicyVersion` kullanarak, bu politikaya `s3:GetObject` ve `iam:*` yetkilerini içeren yeni bir versiyon ekler.
    *   `iam:SetDefaultPolicyVersion` ile bu versiyonu "default" yapar.
4.  **Persistence (Kalıcılık):** Saldırgan, `iam:CreateAccessKey` kullanarak kendine yeni bir kullanıcı oluşturur ve bu kullanıcıya `AdministratorAccess` atar.
5.  **Exfiltration (Veri Sızdırma):** Artık tam yetkili olan saldırgan, S3 bucket'larındaki hassas verileri (PII, secrets) dışarı sızdırır.

## 7. Etkilenen Ortamlar

*   **Güvenlik Araçları:** CSPM (Cloud Security Posture Management), Vulnerability Scanners, Cloud Custodian, Automated Remediation Scripts.
*   **CI/CD Pipeline:** Jenkins, GitLab Runner, GitHub Actions (AWS Runner) - Bu araçlar genellikle yüksek yetkiyle çalışır.
*   **Infrastructure as Code (IaC) Servisleri:** Terraform Cloud, Pulumi işleyicileri.

## 8. Saldırı Karakteristikleri

| Özellik | Analiz |
| :--- | :--- |
| **Tespit Zorluğu** | **Çok Yüksek.** İşlemler meşru bir servis kimliği tarafından yapılıyor. |
| **Gürültü Seviyesi** | **Düşük (Stealthy).** API çağrıları mevcut iş akışına (workflow) çok benzer. |
term | **Zincirleme Potansiyeli** | **Çok Yüksek.** `iam:PassRole` ile başka servislere sıçrayabilir. |

## 9. Detection

### CloudTrail Analizi
Saldırıyı tespit etmek için aşağıdaki CloudTrail event pattern'larına odaklanılmalıdır:
*   `CreatePolicyVersion` veya `SetDefaultPolicyVersion` eventleri, beklenmedik bir zaman diliminde veya beklenmedik bir entity (EC2 Instance Profile) tarafından yapılıyorsa.
*   `PutRolePolicy` çağrılarının, güvenlik ekiplerinin onaylı CI/CD süreçleri dışında gerçekleşmesi.

### GuardDuty ve Algoritmalar
*   **Anomalous IAM Activity:** Bir IAM role'un normalde yapmadığı `iam:Create*` veya `iam:Update*` işlemlerini gerçekleştirmesi.
*   **Unusual API Call Patterns:** Bir EC2 instance'dan gelen ani API çağrısı artışı.

## 10. Mitigation

1.  **Least Privilege (En Az Yetki):** Güvenlik araçlarına sadece "Read-Only" yetki verilmelidir. Eğer yazma yetkisi gerekiyorsa, kapsam (scope) çok dar tutulmalıdır.
2.  **Permission Boundaries:** Kritik rollere "Permission Boundary" uygulanarak, bu rollerin yetkilerini aşması (Privilege Escalation) engellenmelidir.
3.  **Service Control Policies (SCP):** AWS Organizations seviyesinde, belirli kritik IAM operasyonlarının (örneğin `iam:DeleteRole`) hiçbir şekilde yapılamayacağı kısıtlanmalıdır.
4.  **Infrastructure as Code (IaC) Only:** IAM değişiklikleri sadece onaylı bir CI/CD pipeline üzerinden yapılmalı, manuel değişiklikler (drift) anında alarm üretmelidir.

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
