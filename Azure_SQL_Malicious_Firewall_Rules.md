# Burning Data with Malicious Firewall Rules in Azure SQL

> **Araştırma Başlangıcı:** 2024  
> **Kısmi Yama:** 30 Ağustos 2024  
> **Tam Yama:** 9 Nisan 2025  
> **Platform:** Microsoft Azure — Azure SQL Server  
> **Saldırı Tipi:** Post-Compromise Data Destruction + Denial of Service  
> **Keşfeden:** Varonis Threat Labs  
> **Durum:** Tamamen yamalandı — Müşteri eylemi gerekmez

---

## Özet

Varonis Threat Labs, Azure SQL Server'ın güvenlik duvarı (firewall) kural yönetiminde kritik bir tasarım güvenlik açığı keşfetti. Yüksek yetkili erişime sahip bir saldırgan, TSQL komutlarını kullanarak Azure SQL Server güvenlik duvarı kurallarına **özel hazırlanmış kötü niyetli isimler** verebilir ve bu şekilde bir yöneticinin (admin) güvenlik duvarı arayüzüyle etkileşime girmesi durumunda tetiklenecek **yıkıcı bir silah (booby trap)** oluşturabilir.

Bu teknik, ilk erişim sağlandıktan sonra verilen hasarı katlamak için kullanılabilir ve yöneticilerin farkında olmadan kendi sistemlerini tahrip etmesine neden olabilir.

---

## Azure SQL Güvenlik Duvarı Mimarisi

Azure SQL Server'da iki katmanlı güvenlik duvarı kuralı mevcuttur:

```
Azure SQL Server Firewall
├── Sunucu Düzeyi Kurallar (Server-Level Rules)
│   → Azure Management API ve Portal üzerinden yönetilebilir
│   → sp_set_firewall_rule ile TSQL ile de yönetilebilir
│
└── Veritabanı Düzeyi Kurallar (Database-Level Rules)
    → YALNIZCA TSQL ile yönetilebilir (sp_set_database_firewall_rule)
    → Portal ve API üzerinden YÖNETEMEZSINIZ
```

**Kritik Gözlem:** Tüm kurallar Portal UI'da gösterilir, ancak yalnızca veritabanı kural silme işlemleri başarısız olabilir.

---

## Teknik Saldırı Mekanizması

### Güvenlik Açığının Özü

Azure SQL güvenlik duvarı kural adlarına özel karakterler, komutlar veya konfigürasyonlar enjekte edilebilmektedir. Bu "kötü niyetli kurallar" şöyle çalışır:

```sql
-- Kötü niyetli güvenlik duvarı kuralı oluşturma örneği
-- (Saldırgan yüksek yetkili SQL erişimiyle):
EXEC sp_set_firewall_rule 
  @name = N'<özel_hazırlanmış_kötü_niyetli_isim>',
  @start_ip_address = '0.0.0.0',
  @end_ip_address = '0.0.0.0';
-- Bu kural, UI'de zararsız görünür ancak saldırgan tarafından silme
-- işlemi tetiklendiğinde Azure backend'de yıkıcı API çağrıları oluşturur
```

### Saldırı Zinciri

```
1. Saldırgan Azure SQL Server'a yüksek yetkili erişim sağlar
   (web uygulaması açığı, kimlik bilgisi çalma, vb.)

2. TSQL aracılığıyla kötü niyetli güvenlik duvarı kuralları oluşturur
   → Kural isimleri gizlenmiş komutlar içerir
   → UI'de kurallar normal görünür (belki biraz farklı bir isimle)

3. Saldırgan erişimi sonlandırır → passif saldırı hazır

4. Meşru bir yönetici Portal veya API aracılığıyla güvenlik duvarını gözden geçirir
   → Tuhaf görünen kuralı seçer ve "Sil" düğmesine tıklar

5. [TRIGGER] Silme işlemi tetiklendiğinde Azure backend:
   → Kötü niyetli kural ismindeki komutları işler
   → Azure kaynaklarını (SQL veritabanları dahil) SİLER

6. Yönetici farkında olmadan kendi veritabanlarını silmiş olur
```

### Neden Bu Kadar Tehlikeli?

Saldırı şu avantajları taşır:

1. **Stealth (Gizlilik):** Yeni belirgin bir kural yerine mevcut kurallara benzer isimler kullanılır; yöneticinin şüphe duyması zorlaşır.
2. **Toplu Silme:** Tek bir işlemle onlarca Azure kaynağı silinebilir.
3. **Sosyal Mühendislik Boyutu:** Teknik değil; güvenen bir yöneticinin alışkanlık ve reflekslerini kullanır.
4. **Geciktirilmiş Etki:** Saldırgan çoktan erişimini kesmiş olabilirken hasar daha sonra tetiklenir.

---

## Etkilenen Bileşenler

```
Doğrudan Etkilenen:
├── Azure SQL Databases (veritabanı silme)
├── Azure SQL Server yapılandırmaları

Zincirleme Etkileri:
├── Azure SQL'e bağlı uygulama hizmetleri
├── Yedekleme politikaları (silme sonrası recovery penceresi)
└── Bağlı Power BI, Logic Apps ve diğer entegrasyonlar
```

---

## XSS Girişimi ve API Davranışı

Varonis araştırmacıları, güvenlik duvarı kuralı ismine XSS payload'ı enjekte ederek Portal web sayfasına kötü niyetli script yerleştirmeyi denedi:

```
Deney: Kural ismine XSS payload eklendi
Sonuç: XSS çalışmadı; yasaklı karakterler gri renkte gösterildi
       → Portal "bu kurallar yalnızca TSQL ile silinebilir" dedi

Ancak: Grileştirilen kuralın "Sil" ikonu hâlâ tıklanabilirdi!
       → Silme isteği GÖNDERİLİYORDU (API çağrısı yapılıyordu)
       → Bazı durumlarda kaynak silme gerçekleşiyordu
```

Bu bulgu, API katmanında yeterli doğrulama (validation) yapılmadığını ortaya koydu.

---

## Olası Saldırı Senaryosu (Teorik)

```
Aşama 1: İlk Erişim
SQL injection, kimlik bilgisi çalma veya ele geçirilmiş bir 
web uygulaması aracılığıyla Azure SQL Server'a yüksek yetkili erişim.

Aşama 2: Silah Hazırlama
Kötü niyetli güvenlik duvarı kuralları oluşturulur.
Saldırgan izlerini temizler veya erişimini sonlandırır.

Aşama 3: Tetikleme
IT personeli rutin bir bakım veya güvenlik denetimi sırasında
portal üzerinden güvenlik duvarı kurallarını inceler.
"Şüpheli" kural silinir → Booby trap tetiklenir.

Aşama 4: Etki
Büyük ölçekli kaynak ve veri kaybı gerçekleşir.
Olay, IT personelinin kendi hatasıymış gibi görünür.
Forensic analiz zorlaşır.
```

---

## Düzeltme Bilgileri

### Microsoft'un Yama Yaklaşımı

- **30 Ağustos 2024:** Kısmi yama yayınlandı
- **9 Nisan 2025:** Tam yama tamamlandı

Güvenlik açığı artık mevcut değildir. **Mevcut müşterilerin herhangi bir işlem yapması gerekmemektedir.**

### Proaktif Önlemler

Her ne kadar açık kapatılmış olsa da aşağıdaki güvenlik hijyeni önerilir:

```sql
-- Mevcut güvenlik duvarı kurallarını inceleyin:
SELECT name, start_ip_address, end_ip_address
FROM sys.firewall_rules
ORDER BY name;

-- Olağandışı karakterler içeren kural isimlerini kontrol edin:
SELECT name FROM sys.firewall_rules
WHERE name LIKE '%[;<>$()]%';
```

```bash
# Azure CLI ile sunucu düzeyi kuralları listeleyin:
az sql server firewall-rule list \
  --resource-group <rg-name> \
  --server <server-name> \
  --output table
```

---

## Güvenlik Dersleri

### Saldırı Kategorisi: "Trusted Insider Action Abuse"

Bu saldırı tipi, meşru yönetici eylemlerini silah haline getirir. Geleneksel güvenlik kontrollerini atlatır çünkü:

- Eylemi gerçekleştiren kişi meşru bir kullanıcıdır
- İmzalanan kimlik doğrulama token'ları geçerlidir
- Audit log'lar, yöneticinin kendi isteğiyle silme yaptığını gösterir

### Tasarım Güvenliği İlkeleri

Bu zafiyet şu tasarım ilkelerinin ihlaline işaret eder:

1. **Giriş Doğrulama (Input Validation):** Kural isimleri API katmanında sıkı biçimde doğrulanmalıydı.
2. **Defense in Depth:** Silme işlemleri, oluşturma işlemlerinden farklı güvenlik katmanlarına tabi tutulmalıdır.
3. **Separation of Privileges:** Yüksek riskli işlemler (kaynak silme) için ek onay mekanizmaları uygulanmalıdır.

---

## Azure SQL Güvenliği Genel Öneriler

```sql
-- Minimum ayrıcalık ilkesiyle kullanıcı oluşturun:
CREATE USER [app_user] WITH PASSWORD = '<güçlü_parola>';
GRANT SELECT, INSERT, UPDATE ON dbo.TableName TO [app_user];
-- CONTROL veya OWNER vermeyin

-- Güvenlik duvarı kurallarını yönetmek için ayrı bir admin hesabı kullanın
-- Uygulama hesaplarına güvenlik duvarı yönetim izni vermeyin
```

---

## Kaynaklar

- [Varonis Teknik Araştırması](https://www.varonis.com/blog/malicious-firewall-rules-in-azure-sql)
- [CloudVulnDB](https://www.cloudvulndb.org/burning-data-azure-sql-firewall)

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
