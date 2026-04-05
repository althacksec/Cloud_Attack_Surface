# AWS Security Tool Introduces Privilege Escalation Risk

> **Keşif Tarihi:** 11 Aralık 2024  
> **Açıklama Tarihi:** Şubat 2026  
> **Platform:** Amazon Web Services — Account Assessment for AWS Organizations  
> **Saldırı Tipi:** Cross-Account Privilege Escalation (Flawed Deployment Instructions)  
> **Keşfeden:** Token Security  
> **Durum:** Vendor (AWS) düzeltmesi tamamlandı — Dokümantasyon güncellendi

---

## Özet

AWS'nin kendi geliştirdiği güvenlik aracı olan **Account Assessment for AWS Organizations**, kuruluşların AWS organizasyonu içindeki çapraz hesap erişimlerini denetlemek amacıyla tasarlanmıştır. Ancak bu aracın dağıtım talimatları, paradoks biçimde **ciddi ayrıcalık yükseltme (privilege escalation) riskleri** içermekteydi.

AWS, müşterileri aracı yönetim hesabı (management account) yerine herhangi bir üye hesabına dağıtmaya yönlendirmiş; bu da güvensiz trust path'leri oluşturarak saldırganların düşük güvenceli geliştirme ortamlarından yüksek hassasiyetli üretim ve yönetim hesaplarına pivot yapmasına zemin hazırlamıştır.

---

## Araç Mimarisi

Account Assessment for AWS Organizations, hub-spoke modeliyle çalışır:

```
Hub Hesap (Ana Tarayıcı)
    ├── Hub Role — tüm spoke hesaplarını tarama yetkisi
    │   └── ScanSpokeResource izni
    │
Spoke Hesaplar (Taranan Hesaplar)
    ├── Spoke Role — hub role'e güven (trust)
    │   └── AccountAssessment-Spoke-ExecutionRole
    ├── Production Hesap → Spoke Role
    ├── Management Hesap → Spoke Role
    └── Development Hesap → Spoke Role
```

**Güven Zinciri:** Hub Role, tüm Spoke Role'leri üstlenebilir (AssumeRole). Bu durum, Hub'ın dağıtıldığı hesabın güvenlik düzeyi, tüm taranan hesapların efektif güvenlik düzeyini belirler.

---

## Güvenlik Açığının Kökeni

### AWS'nin Problemli Dağıtım Talimatı

```
Resmi AWS dokümantasyonu (önceki hali):
"Hub stack — Deploy to any member account in your AWS Organization
 EXCEPT the Organizations management account."
```

Bu talimat şunu ima ediyordu: *Hub'ı yönetim hesabı dışında herhangi bir yere koyabilirsiniz.*

### Neden Bu Tehlikeliydi?

```
Senaryo: Müşteri hub'ı development hesabına kurdu
                ↓
Development Hesap (Hub Role var)
    → Hub Role, production'daki Spoke Role'ü üstlenebilir
    → Hub Role, management hesabındaki Spoke Role'ü üstlenebilir
                ↓
Saldırgan development hesabını ele geçirirse:
    → Hub Role'ün kimlik bilgilerini çalabilir
    → Production ve management hesaplarına tam erişim sağlar
```

**Sonuç:** Development hesabına yapılan bir saldırı, tüm organizasyonu ele geçirmeye dönüşebilir.

---

## Teknik Saldırı Akışı

```
1. Saldırgan, savunmasız bir web uygulaması, IAM misconfiguration
   veya başka bir yöntemle development hesabını ele geçirir.

2. Development hesabında ScanSpokeResource veya
   AccountAssessment-Spoke-ExecutionRole içeren IAM rolleri arar:
   
   aws iam list-roles --query \
     "Roles[?contains(RoleName, 'ScanSpokeResource') || \
     contains(RoleName, 'AccountAssessment-Spoke-ExecutionRole')]"

3. Hub Role'ünü tespit eder ve kimlik bilgilerini çalar.

4. Hub Role kimlik bilgileriyle üretim hesabındaki spoke role'ü üstlenir:
   
   aws sts assume-role \
     --role-arn arn:aws:iam::<prod-account>:role/AccountAssessment-Spoke-ExecutionRole \
     --role-session-name attacker-session

5. Production hesabına tam erişim sağlanır.
   Aynı yöntemle management hesabına da erişilebilir.
```

---

## Risk Matrisi

| Risk Faktörü | Değerlendirme |
|-------------|---------------|
| Saldırı başlangıç noktası | Düşük güvenceli development hesabı |
| Pivot hedef | Production + Management hesapları |
| Gerekli ön erişim | Development hesabında herhangi bir role |
| Tespit kolaylığı | Düşük (meşru AWS API çağrıları) |
| Organizasyonel etki | Tam organizasyon ele geçirme potansiyeli |

---

## Etkilenen Organizasyonlar

Aşağıdaki koşulların hepsini karşılayan organizasyonlar etkilenmiştir:

1. AWS'nin Account Assessment for AWS Organizations aracını kullandı
2. Hub stack'i management account yerine başka bir hesaba kurdu
3. Spoke role'leri production ve/veya management hesaplarına dağıttı

---

## Düzeltme ve Güvenlik Önerileri

### Acil Tespit

```bash
# Organizasyonunuzda savunmasız roller var mı kontrol edin:
aws iam list-roles --query \
  "Roles[?contains(RoleName, 'ScanSpokeResource') || \
   contains(RoleName, 'AccountAssessment-Spoke-ExecutionRole')]" \
  --output table

# Rolün hub'a güven yapılandırmasını kontrol edin:
aws iam get-role --role-name AccountAssessment-Spoke-ExecutionRole \
  | python3 -c "import sys,json; \
    role=json.load(sys.stdin); \
    print(json.dumps(role['Role']['AssumeRolePolicyDocument'], indent=2))"
```

### Düzeltme Seçenekleri

**Seçenek 1: Aracı Kaldırın**
```bash
# CloudFormation stack'lerini silin:
# Hub, Spoke ve Org-Management bileşenlerinin stack'lerini kaldırın
aws cloudformation delete-stack --stack-name AccountAssessment-Hub
aws cloudformation delete-stack --stack-name AccountAssessment-Spoke
```

**Seçenek 2: Güvenli Yeniden Dağıtım**
```
Hub role'ünü YÖNETİM HESABINA veya eşdeğer güvenlik düzeyindeki
bir hesaba yeniden dağıtın.

Kural: Hub role, taradığı en hassas hesap kadar güvenli bir
hesapta bulunmalıdır.
```

### Trust Policy Denetimi

```bash
# Tüm rolleri, development hesabına güvenen production/management
# rollerini arayarak tarayın:
aws iam list-roles --output json | python3 -c "
import json, sys
roles = json.load(sys.stdin)
for role in roles['Roles']:
    doc = role.get('AssumeRolePolicyDocument', {})
    for stmt in doc.get('Statement', []):
        principal = stmt.get('Principal', {})
        if 'AWS' in principal:
            print(f'{role[\"RoleName\"]}: {principal[\"AWS\"]}')
"
```

---

## AWS'nin Yanıtı ve Düzeltme

Token Security, 11 Aralık 2024'te bulguyu AWS Güvenlik Ekibi'ne bildirdi. AWS:

1. Problemi ciddiye aldı ve hızlı biçimde yanıt verdi
2. Dokümantasyonu açık bir uyarıyla güncelledi: Hub role'ünün, taranacak tüm hesaplar kadar güvenli bir hesaba dağıtılması gerektiğini belirtti
3. Müşterilere güvenlik açıklaması yayınladı

---

## Genel Dersler: Güvenlik Araçlarının Güvenliği

Bu olay, kritik bir paradoksu vurgular: **Güvenliği artırmak için tasarlanan araçlar, hatalı dağıtım talimatlarıyla yeni saldırı yüzeyleri oluşturabilir.**

Organizasyonlar için çıkarımlar:

1. **Vendor talimatlarını güvenlik perspektifiyle okuyun** — Resmi dokümantasyon her zaman güvenlik en iyi pratiklerini yansıtmaz.
2. **IAM trust policy'leri denetleyin** — Çapraz hesap erişimini düzenli olarak gözden geçirin.
3. **En az ayrıcalık ilkesi** — Hiçbir güvenlik aracına, denetlediği hesaplardan daha düşük güvenlik düzeyinde bir hesaptan erişim verilmemelidir.
4. **Güven zincirini haritalayın** — Tüm cross-account trust ilişkilerini belgelendirin ve görselleştirin.

---

## Kaynaklar

- [Token Security Blog — Keşif Süreci](https://www.token.security/blog/aws-built-a-security-tool-it-introduced-a-security-risk)
- [CloudVulnDB](https://www.cloudvulndb.org/aws-security-tool-risk)
- [AWS Account Assessment for AWS Organizations](https://aws.amazon.com/solutions/implementations/account-assessment-for-aws-organizations/)

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
