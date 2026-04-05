# Entra ID Actor Token Validation Bug — Cross-Tenant Global Admin (CVE-2025-55241)

> **Keşif Tarihi:** 14 Temmuz 2025  
> **Düzeltme Tarihi:** 17 Temmuz 2025  
> **CVE:** CVE-2025-55241  
> **CVSS Skoru:** 10.0 (CRITICAL — Maksimum)  
> **Platform:** Microsoft Entra ID (Azure Active Directory)  
> **Saldırı Tipi:** Cross-Tenant Identity Impersonation → Global Admin Takeover  
> **Keşfeden:** Dirk-jan Mollema  
> **Durum:** Yamalandı — Müşteri eylemi gerekmez

---

## Özet

CVE-2025-55241, Microsoft Entra ID (eski adıyla Azure Active Directory) tarihinde keşfedilmiş **en kritik kimlik güvenliği açıklarından biridir**. Bu güvenlik açığı, bir saldırganın kendi kontrol ettiği lab tenant'ında aldığı bir token'ı kullanarak **dünyadaki herhangi bir Entra ID tenant'ında herhangi bir kullanıcı olarak** — Global Administrator dahil — kimliğine bürünmesine olanak tanımaktaydı.

Araştırmacı Dirk-jan Mollema'nın ifadesiyle: **"Eğer bir Entra ID yöneticisiyseniz, bu tenant'ınıza tam erişim anlamına geliyor."**

---

## Güvenlik Açığının İki Bileşeni

### Bileşen 1: Actor Token'ları

**Actor token'lar**, Microsoft'un kendi backend servislerinin servis-servis (S2S) iletişiminde kullandığı, belgelenmemiş dahili JWT token türüdür. Bu token'lar:

- Access Control Service (ACS) tarafından yayınlanır
- Microsoft'un iç servislerinin (Exchange, SharePoint vb.) adına kullanıcı işlemleri yapmasını sağlar
- **Belgelenmez** — dış geliştiricilerin kullanması için değil
- **Log oluşturmaz** — yayınlandıklarında izler bırakmaz

### Bileşen 2: Azure AD Graph API Tenant Doğrulama Hatası

Legacy Azure AD Graph API (`graph.windows.net`), Actor token'larını işlerken **token'ın hangi tenant'dan geldiğini yeterince doğrulamıyordu**.

```
Normal Doğrulama:
Token alındı → Tenant ID doğrulandı → Tenant A token'ı → Tenant A'da işlem

Savunmasız Doğrulama:
Token alındı → Tenant doğrulaması YETERSİZ
→ Tenant A'dan alınan token, Tenant B'de kullanıldı ✓ (KABUL EDİLDİ)
```

---

## Saldırı Mekanizması

### Teknik Akış

```
Saldırganın Sahip Oldukları:
└── Kendi kontrol ettiği herhangi bir Entra ID tenant
    └── (Ücretsiz Azure deneme hesabı bile yeterli)

Aşama 1: Saldırgan kendi tenant'ında Actor token talep eder
         → ACS token yayınlar (bu token herhangi bir kullanıcıyı taklit edebilir)

Aşama 2: Token, hedef tenant'daki Azure AD Graph API'ye gönderilir
         → API, tenant origin'i yeterince doğrulamaz
         → Token geçerli kabul edilir

Aşama 3: Saldırgan hedef tenant'da Global Admin olarak kimliğine bürünür
         → Tenant ayarlarını okur ve değiştirir
         → Yeni hesaplar oluşturur
         → Rol atamalarını değiştirir
         → Conditional Access politikalarını devre dışı bırakır
```

---

## Saldırının Kritik Özellikleri

### Sıfır İz (Zero Telemetry)

Bu güvenlik açığı, geleneksel tespit mekanizmalarını tamamen by-pass eder:

| Kontrol | Durum |
|---------|-------|
| Actor token yayınlama log'u | ❌ Yok |
| Azure AD Graph API okuma log'u | ❌ Yok (erken preview'da) |
| MFA / Multi-Factor Authentication | ❌ By-pass |
| Conditional Access Politikaları | ❌ By-pass |
| Sign-in log'ları | ❌ Anormal aktivite görünmez |

Saldırı; Exchange Online, SharePoint gibi servisler meşru admin aktivitesi gibi görünür.

### Hedef Gereklilikleri

```
Saldırgan → HİÇBİR ŞEYE İHTİYACI YOK:
✗ Hedef organizasyona önceden erişim
✗ Hedef organizasyonda hesap
✗ Kimlik bilgisi
✗ Sosyal mühendislik

Tek ihtiyaç: Kendi lab tenant'ı (ücretsiz Azure hesabı)
```

---

## Etki Analizi

### Tam Tenant Ele Geçirme

Başarılı bir saldırı şunları sağlar:

```
Microsoft 365 Erişimi:
├── Exchange Online — tüm e-postalar (okuma + gönderme)
├── SharePoint / OneDrive — tüm belgeler
├── Teams — tüm mesajlar ve dosyalar
└── Microsoft 365 Admin Center — tam kontrol

Azure Erişimi:
├── Tüm Azure abonelik kaynakları
├── Key Vault secret'ları
└── Azure AD yapılandırması

Yönetimsel Kontrol:
├── Yeni Global Admin hesapları oluşturma
├── Conditional Access politikaları devre dışı bırakma
├── Audit log ayarlarını değiştirme
├── Federated identity sağlayıcısı ekleme (kalıcı arka kapı)
└── Tüm tenant kullanıcılarının şifrelerini sıfırlama
```

---

## Zaman Çizelgesi

```
14 Temmuz 2025   → Dirk-jan Mollema, Black Hat/DEF CON hazırlığı sırasında keşfeder
14 Temmuz 2025   → MSRC'ye bildirilir
15 Temmuz 2025   → Ek etki detayları bildirilir
15 Temmuz 2025   → MSRC, testi durdurmasını talep eder
17 Temmuz 2025   → Microsoft global production'a düzeltme deploy eder
17 Temmuz 2025   → MSRC ile çözüm doğrulanır
6 Ağustos 2025   → Ek mitigasyon: Service Principal kimlik bilgileriyle
                    Actor token talebi engellenir
4 Eylül 2025     → CVE-2025-55241 resmi olarak atanır
17 Eylül 2025    → Dirk-jan, teknik blog yazısını yayınlar
31 Ağustos 2025  → Azure AD Graph API (graph.windows.net) tamamen kullanımdan kaldırılır
```

---

## Tespit — Geriye Dönük Analiz

Saldırı neredeyse iz bırakmaz. Ancak yazma operasyonları bazı audit log'ları üretir:

### KQL Detection Query

```kusto
// Azure AD Audit Logs'da şüpheli Actor token aktivitesini ara
AuditLogs
| where TimeGenerated between (datetime(2025-07-01) .. datetime(2025-09-01))
| where InitiatedBy has_any ("Exchange Online", "SharePoint", "Microsoft.Dynamics")
| where TargetResources has "Global Administrator"
| where ActivityDisplayName has_any ("Add member to role", "Reset password",
                                      "Update policy", "Create user")
| project TimeGenerated, ActivityDisplayName, 
          InitiatedBy, TargetResources, Result
```

**Şüpheli Gösterge:** İşlemi başlatan kullanıcının UPN'i var ama uygulama adı "Exchange Online" veya "SharePoint" gibi Microsoft iç servislerine ait. Bu, Actor token taklidinin izini olabilir.

---

## Düzeltme Sonrası Durum

### Microsoft'un Aldığı Önlemler

1. **Cross-tenant Actor token kabulü engellendi** — Azure AD Graph API artık başka tenant'lardan gelen Actor token'larını kabul etmiyor.
2. **Service Principal kimlik bilgileriyle Actor token talebi engellendi** — Yalnızca Microsoft iç servisleri bu token'ları talep edebiliyor.
3. **Azure AD Graph API kullanımdan kaldırıldı** — 31 Ağustos 2025'te tamamen kaldırıldı.

### Müşteri Eylemleri

Microsoft, bu güvenlik açığı için müşteri eylemi gerektirmediğini açıkladı. Ancak en iyi pratikler olarak:

```bash
# 1. Azure AD Graph bağımlılıklarını Microsoft Graph'e taşıyın:
# graph.windows.net → graph.microsoft.com

# 2. Legacy Graph API çağrılarını audit edin:
az monitor activity-log list \
  --query "[?contains(resourceId, 'graph.windows.net')]"

# 3. Global Admin rolünü gözden geçirin:
az ad user list --query "[?accountEnabled==true]" | \
  az ad role assignment list --role "Global Administrator"
```

---

## Genel Dersler: Bulut Kimliği Güvenliği

CVE-2025-55241, kimlik güvenliğinin neden kritik olduğunu açıkça ortaya koyar:

### 1. Legacy API'lerin Sistem Riski

Kullanımdan kaldırılan API'ler aktif olmaya devam ettiği sürece, bunların güvenlik açıkları tüm sistemi etkiler.

### 2. İç Token'ların Dış Tehdidi

Yalnızca iç kullanım için tasarlanmış token mekanizmaları, dışarıya sızabilir.

### 3. Log'suz Erişim = Görünmez Saldırı

Yeterli telemetri olmadan, en kapsamlı SIEM sistemi bile kör kalır.

### 4. Tenant İzolasyonu Kritik Varsayımdır

Çok kiracılı kimlik sistemlerinde tenant izolasyonunun bozulması, sadece bir müşteriyi değil, tüm ekosistemi tehdit eder.

---

## Kaynaklar

- [Dirk-jan Mollema Teknik Blog](https://dirkjanm.io/obtaining-global-admin-in-every-entra-id-tenant-with-actor-tokens/)
- [Microsoft MSRC — CVE-2025-55241](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2025-55241)
- [The Hacker News](https://thehackernews.com/2025/09/microsoft-patches-critical-entra-id.html)
- [Mitiga Analizi](https://www.mitiga.io/blog/breaking-down-the-microsoft-entra-id-actor-token-vulnerability-the-perfect-crime-in-the-cloud)
- [CloudVulnDB](https://www.cloudvulndb.org/global-admin-entra-id-actor-tokens)

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
