## 1. Genel Bakış 

*   **Saldırı Adı:** Burning-Data-with-Malicious-Firewall-Rules-in-Azure-SQL
*   **Kategori:** Misconfiguration Abuse / Network Security Group (NSG) Manipulation / Denial of Service (DoS) / Data Exfiltration
*   **Hedef Cloud Platformları:** Microsoft Azure

## 2. Teknik Özet

Bu saldırı tekniği, bir saldırganın Azure SQL Server üzerindeki firewall (güvenlik duvarı) kurallarını manipüle ederek, veritabanının ağ erişim politikasını değiştirmesini hedefler. Saldırgan, ya meşru kullanıcıların erişimini keserek bir **Denial of Service (DoS)** gerçekleştirmeyi (verinin erişilebilirliğini "yakmak") ya da kendi saldırgan IP adreslerini beyaz listeye (allow-list) ekleyerek veritabanını dış dünyaya açmayı ve **Data Exfiltration (Veri Sızdırma)** gerçekleştirmeyi amaçlar.

**Temel Risk ve Etki:** Veri gizliliğinin (Confidentiality) tamamen kaybı ve veri erişilebilirliğinin (Availability) ortadan kaldırılması.

## 3. Güvenlik Modeli ve Arka Plan

Azure SQL Database, ağ güvenliğini sağlamak için "Firewall Rules" yapısını kullanır. Bu yapı, belirli IP adreslerine veya IP aralıklarına SQL Server'a erişim izni verir.

*   **Güvenlik Modeli:** Azure Resource Manager (ARM) üzerinden yönetilen RBAC (Role-Based Access Control) modeline dayanır.
*   **Zafiyet Noktası:** Eğer bir kimlik (User, Service Principal veya Managed Identity), `Microsoft.Sql/servers/firewallRules/write` iznine sahipse, bu kimlik veritabanının ağ sınırlarını değiştirebilir. Saldırı, network katmanındaki bu yapılandırma yetkisinin kötüye kullanılmasına dayanır.

## 4. Teknik Detaylar ve Çalışma Prensibi

Saldırgan, Azure Resource Manager API'sine yönelik yetkisiz veya kötü niyetli `PUT` veya `DELETE` istekleri göndererek firewall kurallarını değiştirir.

### Adım Adım Teknik İşleyiş:
1.  **Keşif (Reconnaissance):** Saldırgan, elindeki kimlik ile `az sql server firewall-rule list` komutuyla mevcut kuralları tarar.
2.  **Manipülasallyasyon (Exploitation):**
    *   **Scenario A (Exfiltration):** Saldırgan, kendi IP adresini kurallara ekler.
    *   **Scenario B (DoS):** Saldırgan, mevcut meşru IP kurallarını siler veya `0.0.0.0 - 255.255.255.255` gibi geniş bir aralık ekleyerek trafiği manipüle eder.
3.  **Saldırı Mekanizması (API Call):**
    Saldırgan aşağıdaki gibi bir REST API çağrısı yaparak firewall kuralını manipüle edebilir:

```http
PUT https://management.azure.com/subscriptions/{subId}/resourceGroups/{rgName}/providers/Microsoft.Sql/servers/{serverName}/firewallRules/{ruleName}?api-version=2021-11-01
Authorization: Bearer <attacker_token>
Content-Type: application/json

{
  "properties": {
    "startIpAddress": "0.0.0.0",
    "endIpAddress": "255.255.255.255"
  }
}
```

**İç Mekanizma:** Bu `PUT` isteği, Azure SQL üzerindeki mevcut kısıtlayıcı kuralı ezer ve tüm internet trafiğine kapıyı açar.

## 5. Gerekli Yetkiler 

Saldırganın bu işlemi gerçekleştirebilmesi için Azure RBAC üzerinde şu izinlerden en az birine sahip olması gerekir:
*   **SQL Server Contributor:** Veritabanı sunucusu üzerinde tam yetki.
*    Muafiyet gerektirmeyen, daha spesifik bir custom role içerisinde `Microsoft.Sql/servers/firewallRules/write` yetkisi.
*   **Owner / Contributor:** Abonelik veya Resource Group seviyesinde geniş yetkiler.

**Yanlış Konfigürasyon:** Service Principal'lara (SPN) veya otomasyon hesaplarına (CI/CD pipeline) gereğinden fazla (over-privileged) yetki verilmesi.

## 6. Detaylı Saldırı Senaryosu 

1.  **Initial Access:** Bir geliştiricinin bilgisayarına sızan malware, geliştiricinin Azure CLI (az cli) session token'larını çalar.
2.  **Privilege Escalation:** Saldırgan, çaldığı token ile bir Service Principal'ın yetkilerini kullanarak `Microsoft.Sql/servers/firewallRules/write` iznine sahip olduğunu keşfeder.
3.  **Persistence/Manipulation:** Saldırgan, `MaliciousRule` adında bir kural oluşturur ve `startIpAddress: 1.2.3.4` (saldırganın IP'si) olarak ayarlar.
4.  **Exfiltration:** Saldırgan, yeni açılan bu tünel üzerinden `sqlcmd` veya `Azure Data Studio` kullanarak veritabanındaki hassas tabloları (PII, kredi kartı vb.) dışarı çeker.
5.  **Impact (The "Burning" Part):** Aynı anda, mevcut tüm `Allow-Office-IP` kurallarını `DELETE` isteğiyle silerek şirketin veritabanına erişimini keser (DoS).

## 7. Etkilenen Ortamlar

*   **Azure SQL Database (Single Database & Elastic Pool)**
*   **Azure SQL Managed Instance** (Network kısıtlamaları farklı olsa da firewall mantığı benzerdir)
*   **Azure SQL Edge**
*   **Mimari Etki:** Veritabanının bağlı olduğu tüm application tier'lar (web servers, microservices) bağlantı kaybı yaşar.

## 8. Saldırı Karakteristikleri

| Özellik | Değer | Açıklama |
| :--- | :--- | :---term
| **Tespit Edilme Zorluğu** | Orta/Yüksek | API çağrıları "legitimate configuration change" gibi görünebilir. |
| **Gürültü Seviyesi** | Low (Stealthy) | Network trafiği oluşturmaz, sadece Management Plane üzerinde işlem yapar. |
| **Zincirleme Potansiyeli** | Çok Yüksek | Veri sızdırma sonrası Ransomware veya tamamen silme (Drop DB) ile birleştirilebilir. |

## 9. Detection 
### Azure Activity Logs Analizi
Saldırıyı tespit etmek için en kritik kaynak **Azure Activity Logs**'dur.

**SIEM Use-Case (KQL - Azure Sentinel/Log Analytics):**
```kql
AzureActivity
| where OperationNameValue == "Microsoft.Sql/servers/firewallRules/write"
| where ActivityStatus == "Success"
| extend CallerIP = CallerIpAddress, CallerIdentity = Identity
| project TimeGenerated, CallerIdentity, OperationNameValue, CallerIP, Properties_d.ruleName
```

**Anomali Tespiti:**
*   Firewall kurallarında alışılmadık saatlerde yapılan değişiklikler.
*   Yeni ve tanınmayan IP adreslerinden gelen `Write` istekleri.
*   `Delete` operasyonlarının (firewall rule deletion) artışı.

## 10. Mitigation & Prevention 

1.  **Principle of Least Privilege (PoLP):** Uygulama kimliklerine (Managed Identities) asla `Contributor` veya `SQL Server Contributor` rolü vermeyin. Sadece veri okuma/yazma yetkisi verin.
2.  **Azure Policy Kullanımı:** "Sadece belirli IP aralıklarına izin ver" veya "Firewall kuralı değiştirilemez" gibi kısıtlayıcı politikalar (Deny/Audit) uygulayın.
mu
3.  **Infrastructure as Code (IaC) Drift Detection:** Terraform veya Bicep ile tanımlanan altyapıdaki sapmaları (drift) sürekli denetleyin. Manuel değişiklikleri otomatik olarak geri alın.
4.  **Privileged Identity Management (PIM):** Kritik yetkilerin (Contributor vb.) sadece onay mekanizmasıyla ve geçici olarak kullanılmasını sağlayın.

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
