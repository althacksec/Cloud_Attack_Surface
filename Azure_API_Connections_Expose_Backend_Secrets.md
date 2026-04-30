## 1. Genel Bakış

*   **Saldırı Adı:** Azure-API-Connections-Expose-Backend-Secrets
*   **Kategori:** Information Disclosure / Credential Exposure / Misconfiguration Abuse
*   **Hedef Cloud Platformları:** Microsoft Azure (Özellikle Azure Logic Apps, Azure Integration Account ve Azure API Management entegrasyonları)
*   **Kritiklik Seviyesi:** Yüksek (Yanal hareket ve veri sızıntısı potansiyeli nedeniyle)

## 2. Teknik Özet

Bu saldırı tekniği, Azure üzerinde "API Connection" kaynaklarının konfigürasyon metadatalarında (metadata) saklanan hassas kimlik bilgilerinin (API anahtarları, bağlantı dizeleri, şifreler) yetkisiz veya aşırı yetkilendirilmiş kullanıcılar tarafından okunabilmesi durumunu ifade eder. Azure Logic Apps gibi servislerin dış dünyadaki servislerle (SQL, SFTP, Office 36 Key Vault vb.) konuşabilmesi için kullanılan bu bağlantı nesneleri, yanlış yapılandırılmış RBAC (Role-Based Access Control) politikaları nedeniyle saldırganlar tarafından "altın madeni" olarak kullanılabilir.

**Temel Risk:** Saldırganın, doğrudan erişimi olmayan backend sistemlerine (veritabanları, üçüncü taraf SaaS uygulamaları) elde ettiği sızdırılan kimlik bilgileriyle erişim sağlaması.

## 3. Güvenlik Modeli ve Arka Plan

Azure, kaynak yönetimi için **Azure Resource Manager (ARM)** modelini kullanır. Her kaynak (Resource), belirli bir provider (örneğin `Microsoft.Web`) altında tanımlanır. 

**Güvenlik Modeli Açığı:**
Azure API Connection kaynakları, bir "connection" nesnesi olarak ARM üzerinde birer kaynak (resource) olarak temsil edilir. Bu kaynakların `properties` alanı, bağlantı kurulması için gereken parametreleri (`parameters`) içerir. Eğer bir kullanıcıya veya servis hesabına bu kaynak üzerinde `Microsoft.Web/connections/read` veya `Microsoft Yapılandırma/parameters/read` yetkisi verilmişse, bu kullanıcı, bağlantı nesnesinin içindeki plain-text veya base64 encode edilmiş (şifrelenmiş gibi görünen ama kolayca çözülebilen) parametreleri okuyabilir.

## 4. Teknik Detaylar ve Çalışma Prensibi

Saldırı, Azure API'lerine yapılan bir `GET` isteği ile gerçekleşir. 

### İşleyiş Adımları:
1.  **Keşif (Reconnaissance):** Saldırgan, sahip olduğu yetkilerle `Microsoft.Web/connections` tipindeki kaynakları listeler.
2.  **İnceleme (Inspection):** Belirlenen bağlantı kaynağının detayları (JSON body) çekilir.
3.  **Sızdırma (Extraction):** `properties.parameters` altındaki hassas alanlar analiz edilir.

### Örnek JSON Çıktısı :
Aşağıdaki JSON, bir saldırganın `GET` isteği sonucunda elde ettiği ve içinde SQL bağlantı dizesini barındıran bir API Connection örneğidir:

```json
{
  "name": "sql-connection-resource",
  "type": "Microsoft.Web/connections",
  "location": "East US",
  "properties": {
    "connectionString": "Server=tcp:mydb.database.windows.net...;Password=Saldırganın_Bulduğu_Şifre;...",
    "parameters": {
      "apiConnectionKey": {
        "value": "dGhpcy1pcy1hLXNlY3JldC1rZXk=", 
        "type": "string"
    },
      "authType": {
        "value": "Basic",
        "type": "string"
      }
    }
  }
}
```
*Not: `apiConnectionKey` gibi değerler genellikle Base64 formatında olabilir, ancak bu bir şifreleme değildir, sadece encoding'dir.*

## 5. Gerekli Yetkiler 

Saldırganın bu veriyi okuyabilmesi için aşağıdaki minimum yetkilerden birine sahip olması gerekir:

*   **RBAC Rolleri:** `Reader`, `Contributor`, `Owner` veya `Logic App Contributor`.
*   **Özel (Custom) İzinler:** 
    *   `Microsoft.Web/connections/read`
    *   `Microsoft.Web/connections/parameters/read`
*   **Yanlış Konfigürasyon:** "Reader" rolünün, kritik bağlantı bilgilerini içeren bir Resource Group (RG) seviyesinde çok geniş bir kullanıcı grubuna (örn. tüm organizasyon) tanımlanmış olması.

## 6. Detaylı Saldırı Senaryosu

**Senaryo: "The Connected Chain"**

1.  **Initial Access:** Saldırgan, bir geliştiricinin düşük yetkili Azure hesabını (phishing yoluyla) ele geçirir. Bu hesap sadece "Okuyucu" (Reader) yetkisine sahiptir.
2.  **Reconnaissance:** Saldırgan, Azure CLI kullanarak ortamdaki tüm bağlantıları tarar:
    `az resource list --resource-type Microsoft.Web/connections`
3.  **Discovery:** Tarama sırasında, `prod-sql-connection` adlı bir kaynağın kritik bir SQL veritabanına bağlandığını fark eder.
4.  **Exploitation (Credential Extraction):** Saldırgan, bu kaynağın detaylarını çeker:
    `az resource show --ids <resource_id>`
    Çıktıdan SQL `connectionString` içeriğindeki kullanıcı adı ve şifreyi kopyalar.
5.  **Lateral Movement & Exfiltration:** Saldırgan, elde ettiği SQL credentials ile SQL Server'a doğrudan bağlanır ve hassas müşteri verilerini (PII) dışarı sızdırır.

## 7. Etkilenen Ortamlar

*   **Azure Logic Apps (Consumption & Standard):** En büyük risk grubu.
*   **Azure Integration Account:** B2B senaryolarında kullanılan bağlantılar.
*   **Azure API Management (APIM):** Backend bağlantı ayarları üzerinden.
rel
*   **Multi-cloud Etkisi:** Doğrudan bir etkisi yoktur ancak Azure üzerinden elde edilen credentials, AWS veya GCP üzerindeki hibrit bağlantıları (örn. Site-to-Site VPN veya özel bağlantı noktaları) tetikleyebilir.

## 8. Saldırı Karakteristikleri

| Özellik | Değer | Açıklama |
| :--- | :--- | :--- |
| **Tespit Edilme Zorluğu** | Düşük/Orta | Standart `GET` veya `List` işlemleri gibi göründüğü için ayırt edilmesi zordur. |
| **Gürültü Seviyesi** | Stealthy (Sessiz) | Okuma işlemi (Read) loglarda normal bir izleme (auditing) faaliyeti gibi durabilir. |
| **Zincirleme Potansiyeli** | Çok Yüksek | Tek bir bağlantıdan (SQL, Key Vault, Service Bus) alınan veri, tüm altyapıyı çökertebilir. |

## 9. Detection 

**Azure Activity Logs Analizi:**
Saldırganın yaptığı işlemler `AzureActivity` loglarında şu şekilde izlenebilir:
*   **Operation Name:** `List Connections`, `Get Connection`, `List Connection Parameters`.
*   **Anomali Göstergesi:** Olağandışı bir IP adresinden veya normalde hiç erişmemesi gereken bir servis hesabından gelen yoğun `List` operasyonları.

**SIEM Use-Case Önerisi:**
`If (OperationName == "Microsoft.Web/connections/read" AND CallerIP NOT IN [Internal_Subnets] AND UserAgent != [Standard_Admin_Tool]) THEN Alert(Critical)`

## Saldırı Analizi Özet Tablosu 

| Log Kaynağı | İzlenecek Alan | Tehdit Göstergesi |
| :--- | :--- | :--- |
| Azure Activity Log | `operationName` | `Microsoft.Web/connections/read` yoğunluğu |
| Azure Activity Log | `caller` | Yetkisiz veya yeni kullanıcıların erişimi |
| Azure Monitor | `Metrics` | Connection resource'larına yönelik anlık istek artışı |

## 10. Incident Response & Remediation

1.  **Containment :**
    *   Şüpheli kullanıcının/servis hesabının Azure erişimini derhal durdurun.
    *   İlgili Resource Group üzerindeki RBAC izinlerini kısıtlayın.
2.  **Eradication :**
    *   **Credential Rotation:** Sızdırıldığı düşünülen tüm şifreleri, bağlantı dizelerini (connection strings) ve API anahtarlarını derhal değiştirin.
    *   Tüm API Key ve Secret'ları geçersiz kılın.
3.  **Recovery :**
    *   Yeni ve güvenli bağlantı yöntemlerini (Managed Identity gibi) devreye alın.
    *   Eski, zayıf kimlik bilgilerini içeren tüm servisleri güncelleyin.

## 11. Öneriler 

*   **Managed Identities Kullanın:** Mümkün olan her yerde kullanıcı adı/şifre yerine Azure Managed Identity kullanarak credential ihtiyacını ortadan kaldırın.
*   **Principle of Least Privilege (PoLP):** Geliştiricilere veya servislere `Reader` yetkisi verirken, hassas kaynakların bulunduğu aboneliklerde kısıtlama getirin.
*   **Azure Key Vault Entegrasyonu:** Hassas bilgileri doğrudan Connection String içinde tutmak yerine, Key Vault içinden dinamik olarak çekin.
*   **Azure Policy Uygulayın:** "Sadece Managed Identity kullanımına izin ver" gibi politikalarla configuration drift'i engelleyin.

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
