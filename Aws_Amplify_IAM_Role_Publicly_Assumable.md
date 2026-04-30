## 1. Genel Bakış 

*   **Saldırı Adı:** AWS-Amplify-IAM-Role-Publicly-Assumable-Exposure
*   **Kategori:** IAM Misconfiguration, Privilege Escalation, Identity Provider (IdP) Abuse
*   **Hedef Cloud Platformları:** AWS (Amazon Web Services)
*   **Etki Alanı:** Web ve Mobil Uygulamalar (AWS Amplify & Amazon Cognito entegrasyonu)

## 2. Teknik Özet

Bu zafiyet, AWS Amplify kullanarak geliştirilen uygulamalarda, istemci tarafında (client-side) çalışan kodların erişmesi gereken IAM rollerinin, **Trust Policy (Güven İlişkisi Politikası)** yapılandırmasındaki bir hata nedeniyle, herhangi bir kimlik doğrulaması yapılmaksızın (unauthenticated) internet üzerindeki herhangi bir kullanıcı tarafından "assume" edilebilir (üstlenilebilir) olması durumudur. Saldırgan, bu role sahip olduğunda, bu rolün sahip olduğu tüm AWS izinlerini (S3, DynamoDB, Lambda vb.) kullanarak veri sızdırabilir veya bulut altyapısında yanal hareket (lateral movement) gerçekleştirebilir.

## 3. Güvenlik Modeli ve Arka Plan

AWS Amplify ekosisteminde, frontend uygulamalarının AWS kaynaklarına (örneğin bir S3 bucket'ına dosya yüklemek) doğrudan erişebilmesi için **Amazon Cognito Identity Pools** kullanılır. 

Güvenlik modeli şu üç bileşene dayanır:
1.  **Cognito Identity Pool:** Kullanıcılara geçici AWS kimlik bilgileri (Access Key, Secret Key, Session Token) sağlar.
2.  **IAM Role:** Bu kimlik bilgilerinin arkasında çalışan, yetkileri tanımlı roldür.
3.  **Trust Policy:** Hangi "Principal" (asıl unsur)ın bu rolü üstlenebileceğini belirleyen mekanizmadır.

**Zafiyetin Kaynağı:** Amplify'ın otomatize kurulum süreçlerinde, "Unauthenticated Access" (Kimliksiz Erişim) özelliği aktif edildiğinde, oluşturulan IAM rolünün Trust Policy'si, `cognito-identity.amazonaws.com` servisine güvenecek şekilde yapılandırılır ancak `Condition` (Koşul) bloğu eksik bırakılarak, Cognito Identity Pool ID'si bilinen herkesin bu rolü almasına izin verilir.

## 4. Teknik Detaylar ve Çalışma Prensibi

Saldırının teknik işleyişi, bir Identity Provider (IdP) federasyonunun yanlış yapılandırılmasına dayanır.

### Mekanizma:
Normal şartlarda, bir IAM rolü sadece belirli bir Cognito Identity Pool'dan gelen isteklere izin vermelidir. Ancak zafiyet durumunda, Trust Policy şu şekildedde görünür:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "cognito-identity.amazonaws.com:aud": "us-east-1:xxxx-xxxx-xxxx-xxxx"
        }
      }
    }
  ]
}
```

**Zafiyet Anı:** Eğer `Condition` bloğundaki `aud` (Audience) kontrolü kaldırılırsa veya `StringLike` gibi daha gevşek bir operatörle yanlış yapılandırılırsa, saldırgan kendi Cognito Identity Pool'unu kullanarak bu rolü üstlenebilir.

### Exploit Mantığı:
1.  **Discovery:** Saldırgan, tarayıcıdaki JavaScript dosyalarını (amplifyclient.js vb.) inceleyerek `IdentityPoolId` bilgisini bulur.
2.  **Token Generation:** Saldırgan, AWS STS (Security Token Service) API'sine `AssumeRoleWithWebIdentity` çağrısı yaparak, bulunan `IdentityPoolId` ile geçici kimlik bilgileri talep eder.
3.  **Impersonation:** Elde edilen `AccessKeyId`, `SecretAccessKey` ve `SessionToken` ile saldırgan artık o IAM rolünün yetkilerine sahip bir "user" gibi davranır.

## 5. Gerekli Yetkiler 

*   **Saldırgan İçin:** 
    *   Hedef uygulamanın public JavaScript bundle'larına erişim.
    *   `sts:AssumeRoleWithWebIdentity` API çağrısını yapabilmek için standart bir internet bağlantısı.
*   **Yanlış Konfigürasyonlar:**
    *   Cognito Identity Pool üzerinde "Enable Unauthenticated Access" seçeneğinin açık olması.
    *   IAM Role Trust Policy'sinde `Condition` kontrolünün eksikliği veya hatalı olması.
    *   IAM Rolüne atanan Permission Policy'nin (izin politikası) aşırı geniş (overly permissive) olması (örn: `s3:*`).

## 6. Detaylı Saldırı Senaryosu

**Senaryo: Veri Sızıntısı**

1.  **Initial Access (Keşif):** Bir Red Teamer, bir e-ticaret sitesinin frontend kodlarını analiz ederken `aws-exports.js` dosyasını bulur. Dosyada şu bilgi yer alır: `identityPoolId: 'us-east-1:abc123de-456f-789g-0123-456789abcdef'`.
2.  **Privilege Escalation (Kimlik Üstlenme):** Saldırgan, AWS CLI kullanarak Cognito üzerinden kimlik bilgilerini alır:
    ```bash
    # Cognito üzerinden kimlik bilgilerini alma (Unauthenticated)
    aws cognito-identity get-id --identity-pool-id us-string-id-here --region us-east-1
    
    # Alınan IdentityID ile geçici credential üretme
    aws cognito-identity get-credentials-for-identity --identity-id <identity-id-from-previous-step> --region us-east-1
    ```
3.  **Persistence & Action (Eylem):** Elde edilen `AccessKey`, `SecretKey` ve `SessionToken` ortam değişkenlerine set edilir.
4.  **Exfiltration (Veri Sızdırma):** Saldırgan, rolün yetkilerini kullanarak S3 bucket'larını listeler ve hassas verileri çeker:
    ```bash
    aws s3 ls s3://company-customer-data-bucket/
    aws s3 cp s3://company-customer-data-bucket/customers.csv .
    ```

## 7. Etkilenen Ortamlar

*   **AWS Amplify Hosting:** Frontend uygulamalarının barındırıldığı ortamlar.
*   **Amazon Cognito Identity Pools:** Kimlik yönetimi sağlanan katman.
*   **Serverless Mimariler:** Lambda, API Gateway ve DynamoDB kullanan, istemci tarafı doğrudan buluta bağlı (client-to-cloud) mimariler.

## 8. Saldırı Karakteristikleri

| Özellik | Açıklama |
| :--- | :--- |
| **Tespit Edilme Zorluğu** | **Yüksek.** Saldırgan tamamen geçerli AWS API çağrıları yapmaktadır. |
| **Gürültü Seviyesi** | **Düşük (Stealthy).** Trafik, uygulamanın normal işleyişine çok benzer. |
rolün yetkilerine bağlı olarak S3, DynamoDB, SNS gibi servislerle zincirlenebilir. |
| **Saldırı Türü** | **Misconfiguration Abuse.** |

## 9. Detection 

### CloudTrail Analizi
Saldırı, CloudTrail loglarında `AssumeRoleWithWebIdentity` olayları ile izlenebilir.
*   **Anomali Göstergesi:** Belirli bir `IdentityPoolId` üzerinden gelen, ancak uygulamanın normal kullanıcı trafiğiyle örtüşmeyen, alışılmadık `userAgent` veya farklı coğrafi konumlardan gelen `AssumeRoleWithWebIdentity` çağrıları.
*   **Sorgu Örneği (CloudWatch Logs/Athena):**
    ```sql
    SELECT eventname, sourceipaddress, useragent, requestparameters
    FROM cloudtrail_logs
    WHERE eventname = 'AssumeRoleWithWebIdentity'
    AND useragent LIKE '%Unknown%';
    ```

### GuardDuty ve IDS
AWS GuardDuty, olağandışı API çağrılarını (ör- `Discovery:IAM/AnomalousBehavior`) tespit edebilir.

## 10. Mitigation

1.  **Strict Trust Policy:** IAM Role Trust Policy içerisinde `Condition` bloğunu kullanarak mutlaka `StringEquals` ile `cognito-identity.amazonaws.com:aud` (veya ilgili Identity Pool ID) kontrolü yapılmalıdır.
2.  **Principle of Least Privilege:** Unutulmamalıdır ki, istemci tarafında (client-side) kullanılan rollere asla `s3:ListAllMyBuckets` veya `dynamodb:Scan` gibi geniş kapsamlı izinler verilmemelidir.
3.  **Infrastructure as Code (IaC) Scanning:** Terraform veya CloudFormation şablonları, CI/CD süreçlerinde `checkov` veya `tfsec` gibi araçlarla taranarak zayıf Trust Policy yapıları tespit edilmelidir.
4.  **Monitoring:** Cognito Identity Pool kullanım istatistikleri ve CloudTrail logları üzerinde sürekli alarm mekanizmaları kurulmalıdır.

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
