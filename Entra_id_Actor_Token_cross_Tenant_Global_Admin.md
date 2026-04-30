## 1. Genel Bakış

| Parametre | Detay |
| :--- | :--- |
| **Saldırı Adı** | Entra-ID-Actor-Token-Validation-Bug-Cross-Entity-Global-Admin |
| **Kategori** | Privilege Escalation (Yetki Yükseltme), Broken Access Control, Identity Provider (IdP) Logic Flaw |
| **Hedef Platform** | Microsoft Azure (Microsoft Entra ID) |
| **Etki Seviyesi** | **Kritik (Critical)** - Tam Tenant Ele Geçirme |
| **Saldırı Vektörü** | Token Delegation / On-Behalf-Of (OBO) Flow Manipulation |

---

## 2. Teknik Özet

Bu rapor, Microsoft Entra ID (eski adıyla Azure AD) ekosisteminde, "Actor" (aktör) claim'inin (imla/iddia) doğrulanması sırasında meydana gelen mantıksal bir hatayı incelemektedir. Zafiyet, bir saldırganın kendi yönettiği bir tenant (Tenant A) üzerinden aldığı geçerli bir token'ı, "actor" (aracı) kimliğiyle manipüle ederek, hedef bir tenant (Tenant B) üzerindeki kaynaklara, sanki hedef tenant'ın bir yetkilisiymnya gibi erişebilmesine olanak tanır. Temel risk, **Cross-Tenant Identity Impersonation** (Tenantlar arası kimlik taklidi) yoluyla Global Admin yetkilerine ulaşılmasıdır.

---

## 3. Güvenlik Modeli ve Arka Plan

Microsoft Entra ID, **OAuth 2.0** ve **OpenID Connect (OIDC)** protokollerini kullanır. Modern bulut mimarilerinde, bir servis (Service Principal), bir kullanıcının adına işlem yapmak için **On-Behalf-Of (OBO)** akışını kullanır.

*   **Güven Sınırı (Trust Boundary):** Normal şartlarda, bir token'ın `tid` (Tenant ID) ve `iss` (Issuer) değerleri, token'ın ait olduğu organizasyonu tanımlar.
*   **Zafiyetin Doğası:** Güvenlik modeli, token'ın imzasının (signature) geçerli olmasını kontrol eder; ancak "actor" (aracı) claim'i kullanıldığında, sistemin "aracı" ile "asıl kullanıcı" arasındaki tenant bağını (tenant binding) çapraz kontrole tabi tutmadığı varsayılmaktadır. Bu durum, imza geçerli olsa bile, güven sınırının aşılmasına neden olur.

---

## 4. Teknik Detaylar ve Çalışma Prensibi

Saldırı, token içerisindeki `act` (actor) claim'inin manipülasyonuna dayanır.

### Token Yapısı ve Manipülasyon
Bir OBO akışında token içinde şu tür bir yapı bulunur:
```json
{
  "aud": "https://graph.microsoft.com",
  "iss": "https://sts.windows.net/tenant-A-id/",
  "sub": "user-from-tenant-A",
  "tid": "tenant-A-id",
  "act": {
    "claims": [
      {
        "typ": "name",
        "val": "malicious-actor-service-principal"
      }
    ]
  }
}
```

### Çalışma Mantığı 
1.  **Signature Bypass (Mantıksal):** Saldırgan, Tenant A'dan geçerli bir imza almış bir token üretir.
2.  **Claim Injection:** `act` claim'i kullanılarak, token'ın bir "aracı" üzerinden geldiği simüle edilir.
3.  **Validation Failure:** Hedef kaynak (API veya Resource), token'ın imzasını Tenant A'nın public key'i ile doğrular (imza doğrudur). Ancak, sistem `iss` (issuer) değerinin hedef tenant (Tenant B) ile eşleşmediğini veya `act` içindeki aracı kimliğinin hedef tenant'a ait bir servis olmadığı gerçeğini denetlemeyi atlar.

---

## 5. Gerekli Yetkiler 

*   **Attacker Requirements:** 
    *   Herhangi bir aktif Microsoft Entra ID Tenant'ına erişim.
    *   Bir App Registration (App ID) oluşturma ve token talep etme yetkisi.
*   **Target Requirements:**
    *   Hedef tenant'ta "On-Behalf-Of" akışını destekleyen veya bu akışın kullanıldığı bir servis/API (örn: Custom API, Logic Apps, Function Apps).
    *   Yanlış konfigüre edilmiş, tenant ayrımı yapmayan (tenant-agnostic) bir validation middleware.

---

## 6. Detaylı Saldırı Senaryosu

**Senaryo: Cross-Tenant Privilege Escalation**

1.  **Initial Access:** Saldırgan, kendi kontrolündeki `attacker-tenant` üzerinde bir uygulama kaydı oluşturur.
2.  **Token Generation:** Saldırgan, `attacker-tenant` üzerinden bir access token alır. Bu token, imza açısından tamamen geçerlidir.
3.  **Token Manipulation (The Attack):** Saldırgan, token'ı `act` claim'i ekleyerek veya mevcut olanı manipüle ederek, hedef `victim-tenant`'daki bir servis prensibini (Service Principal) "aracı" olarak gösterir.
4.  **Exploitation:** Saldırgan, bu manipüle edilmiş token ile `victim-tenant`'ın Microsoft Graph API uç noktalarına veya özel API'larına istek atar.
5.  **Privilege Escalation:** Eğer hedef API, gelen token'daki `roles` veya `groups` claim'lerini (token içindeki sahte verilerden) sorguluyorsa, saldırgan "Global Administrator" yetkileriyle işlem yapabilir.
6._**Impact:** Saldırgan, hedef tenant üzerinde kullanıcı oluşturabilir, gizli verileri sızdırabilir veya tüm altyapıyı şifreleyebilir.

---

## 7. Etki Analizi 

| Etki Alanı | Derece | Açıklama |
| :--- | :--- | :--- |
| **Gizlilik (Confidentiality)** | **Kritik** | Tüm kurumsal veriler (E-postalar, Azure Key Vault, DB) sızdırılabilir. |
| **Bütünlük (Integrity)** | **Kritik** | Kayıtlar değiştirilebilir, kullanıcı yetkileri manipüle edilebilir. |
| **Erişilebilirlik (Availability)** | **Yüksek** | Kaynaklar silinebilir veya Ransomware saldırısına açık hale getirilebilir. |

---

## 8. Tespit ve Önleme (Detection & Mitigation)

### Tespit 
*   **SIEM/Log Analytics:** Azure AD (Entra ID) Sign-in loglarında, alışılmadık `Issuer` (iss) değerine sahip ancak başarılı olan giriş denemelerini izleyin.
*   **Anomali Tespiti:** Farklı tenant'lardan gelen ve aynı kaynak IP üzerinden gelen çok sayıda "Success" logunu takip edin.
*   **Audit Logs:** `Service Principal` yetkilerinin veya `Role Assignment` işlemlerinin beklenmedik zamanlarda yapıldığını denetleyin.

### Önleme 
*   **Strict Issuer Validation:** Uygulamalarınızda ve API Gateway'lerinizde mutlaka `iss` (Issuer) değerini sadece beklenen tenant ID'si ile kısıtlayın (White-listing).
*   **Cross-Tenant Policy:** Azure AD üzerinden "Cross-tenant access settings" kullanarak sadece güvenilir tenant'lara izin verin.
*   **Claims Validation:** Token içindeki claim'leri sadece doğrulamayın, bu claim'lerin kaynağını (Issuer) ve doğruluğunu tenant bazlı kontrol edin.
*   **Zero Trust:** "Asla güvenme, her zaman doğrula" prensibiyle, her isteğin hangi tenant'a ait olduğunu katı bir şekilde denetleyin.

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
