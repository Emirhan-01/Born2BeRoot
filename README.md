*This project has been created as part of the 42 curriculum by embostan.*


---

## Description / Açıklama

**EN:** Born2beRoot is a system administration project. The goal is to set up a virtual machine (VirtualBox/UTM) running Debian, configured according to strict security rules: encrypted LVM partitioning, SSH on a non-default port with root login disabled, a firewall restricting access to a single port, a strong password policy, a hardened sudo configuration, and a monitoring script that periodically broadcasts system information.

**TR:** Born2beRoot, bir sistem yönetimi (system administration) projesidir. Amaç, VirtualBox/UTM üzerinde Debian çalıştıran bir sanal makineyi, sıkı güvenlik kurallarına göre yapılandırmaktır: şifreli LVM partitionlama, root girişi kapalı ve varsayılan olmayan bir portta çalışan SSH, tek bir portu açık bırakan firewall, güçlü bir şifre politikası, sıkılaştırılmış sudo yapılandırması ve sistem bilgilerini periyodik olarak terminallere yayınlayan bir monitoring script'i.

---

## Instructions / Kurulum ve Çalıştırma

**EN:**
1. Install VirtualBox (or UTM) on the host machine.
2. Create a new VM and install the latest stable version of Debian, manually partitioning the disk with LVM and encryption (at least 2 encrypted partitions).
3. After installation, connect via SSH: `ssh <login>@<VM_IP> -p 4242`.
4. Follow the configuration steps described in the "Project description" section below (sudo, password policy, firewall, AppArmor, monitoring script).
5. To verify the machine matches the submission, compare the disk signature: `sha1sum <disk>.vdi` (or `.qcow2` for UTM) against the one in `signature.txt`.

**TR:**
1. Host makinede VirtualBox (veya UTM) kur.
2. Yeni bir VM oluştur ve Debian'ın en güncel stable sürümünü, diski LVM ile ve şifreli olarak manuel partitionlayarak kur (en az 2 şifreli partition).
3. Kurulumdan sonra SSH ile bağlan: `ssh <login>@<VM_IP> -p 4242`.
4. Aşağıdaki "Project description" bölümünde anlatılan yapılandırma adımlarını uygula (sudo, şifre politikası, firewall, AppArmor, monitoring script).
5. Makinenin teslim edilenle aynı olduğunu doğrulamak için disk imzasını karşılaştır: `sha1sum <disk>.vdi` (UTM için `.qcow2`) çıktısını `signature.txt` ile kıyasla.

---

## Resources / Kaynaklar

**EN:**
- Official Debian documentation: https://www.debian.org/doc/
- `man` pages for `sudo`, `visudo`, `ufw`, `crontab`, `chage`, `pam_pwquality`
- AppArmor documentation: https://wiki.debian.org/AppArmor
- **AI usage disclosure:** Claude (Anthropic) was used as a learning aid — to get explanations of concepts already attempted manually first (e.g. how LVM/LUKS encryption layers relate to each other, how `pam_pwquality` vs `pam_unix remember` differ, why `@reboot` cron jobs may run before any terminal session exists), to debug specific configuration issues after manual troubleshooting (e.g. a `passwd` prompt not looping back correctly, a `grep` returning no output because a setting was written to a different file than expected), and to help organize/format this README and defense notes. AI was not used to generate the security configuration itself; each setting was written and tested manually first.

**TR:**
- Resmi Debian dokümantasyonu: https://www.debian.org/doc/
- `sudo`, `visudo`, `ufw`, `crontab`, `chage`, `pam_pwquality` için `man` sayfaları
- AppArmor dokümantasyonu: https://wiki.debian.org/AppArmor
- **AI kullanım açıklaması:** Claude (Anthropic), bir öğrenme yardımcısı olarak kullanıldı — önce manuel olarak denenmiş konuların açıklanması için (ör. LVM/LUKS şifreleme katmanlarının birbiriyle ilişkisi, `pam_pwquality` ile `pam_unix remember` arasındaki fark, `@reboot` cron job'larının neden bazen hiçbir terminal oturumu yokken çalışabildiği), manuel deneme sonrası tıkanılan spesifik yapılandırma sorunlarını debug etmek için (ör. `passwd` isteminin doğru döngüye girmemesi, bir ayarın beklenenden farklı bir dosyaya yazıldığı için `grep`'in boş dönmesi), ve bu README ile savunma notlarının düzenlenmesi/biçimlendirilmesi için kullanıldı. Güvenlik yapılandırmasının kendisi AI ile üretilmedi; her ayar önce manuel olarak yazılıp test edildi.

---

## Project description / Proje Açıklaması

### Operating system choice / İşletim sistemi seçimi

**EN:** Debian was chosen over Rocky Linux.
- **Debian advantages:** simpler setup for beginners, uses `apt`/`ufw`/AppArmor which are lighter-weight and more beginner-friendly, huge community documentation, very stable for long-running servers.
- **Debian disadvantages:** slower release cycle for newer software versions, less enterprise/commercial support compared to RHEL-based distros.
- **Rocky advantages:** closer to enterprise/production environments (RHEL-compatible), SELinux offers more granular, label-based access control.
- **Rocky disadvantages:** steeper learning curve, SELinux configuration is significantly more complex to set up correctly, `firewalld` has a different (zone-based) mental model than `ufw`.

**TR:** Rocky Linux yerine Debian seçildi.
- **Debian avantajları:** yeni başlayanlar için daha basit kurulum, daha hafif ve öğrenmesi kolay `apt`/`ufw`/AppArmor araçları, çok geniş topluluk dokümantasyonu, uzun süre çalışan sunucular için oldukça kararlı.
- **Debian dezavantajları:** yeni yazılım sürümlerine geçiş daha yavaş, RHEL tabanlı dağıtımlara kıyasla kurumsal/ticari destek daha az.
- **Rocky avantajları:** kurumsal/production ortamlara daha yakın (RHEL uyumlu), SELinux daha ayrıntılı, etiket (label) tabanlı erişim kontrolü sunar.
- **Rocky dezavantajları:** öğrenme eğrisi daha dik, SELinux'u doğru yapılandırmak belirgin şekilde daha karmaşık, `firewalld`'ın (zone tabanlı) mantığı `ufw`'den farklı ve daha soyut.

### AppArmor vs SELinux

**EN:** Both are Mandatory Access Control (MAC) systems that restrict what a process can do beyond standard Unix permissions. AppArmor (used on Debian) is path-based: it restricts programs according to filesystem paths defined in per-application profiles, and is generally easier to write and read. SELinux (used on Rocky) is label-based: every file, process, and resource gets a security label, and policy is defined in terms of these labels, which is more powerful and fine-grained but considerably harder to configure and debug.

**TR:** İkisi de standart Unix izinlerinin ötesinde, bir sürecin neler yapabileceğini kısıtlayan MAC (Mandatory Access Control) sistemleridir. AppArmor (Debian'da kullanılır) yol (path) tabanlıdır: programları, her uygulama için tanımlanmış profillerdeki dosya sistemi yollarına göre kısıtlar, genelde yazması ve okuması daha kolaydır. SELinux (Rocky'de kullanılır) etiket (label) tabanlıdır: her dosya, süreç ve kaynağa bir güvenlik etiketi atanır ve politika bu etiketler üzerinden tanımlanır; bu daha güçlü ve ince taneli (fine-grained) bir kontrol sağlar ama yapılandırması ve debug edilmesi belirgin şekilde daha zordur.

### UFW vs firewalld

**EN:** UFW (Uncomplicated Firewall, used on Debian) is a simple front-end for `iptables`/`nftables`, managed with straightforward rule commands (`ufw allow 4242`). firewalld (used on Rocky) is zone-based: interfaces/sources are assigned to zones (e.g. `public`, `internal`), and rules are defined per zone, which is more flexible for multi-network servers but adds conceptual overhead for a single-purpose VM like this project's.

**TR:** UFW (Uncomplicated Firewall, Debian'da kullanılır), `iptables`/`nftables` için basit bir ön yüzdür; kurallar doğrudan komutlarla yönetilir (`ufw allow 4242`). firewalld (Rocky'de kullanılır) zone (bölge) tabanlıdır: arayüzler/kaynaklar zone'lara atanır (ör. `public`, `internal`) ve kurallar zone bazında tanımlanır; bu, çoklu ağa sahip sunucular için daha esnektir ama bu projedeki gibi tek amaçlı bir VM için ekstra kavramsal karmaşıklık getirir.

### VirtualBox vs UTM

**EN:** VirtualBox is a cross-platform (Windows/Linux/Intel Mac) type-2 hypervisor with broad hardware virtualization support and a large ecosystem of guest additions. UTM is a QEMU-based hypervisor primarily used on Apple Silicon (M1/M2/M3) Macs, since VirtualBox does not support ARM-based Macs; it uses `.qcow2` disk images instead of VirtualBox's `.vdi`. Functionally both provide the virtualization needed for this project — the choice mainly depends on the host machine's CPU architecture.

**TR:** VirtualBox, geniş donanım sanallaştırma desteğine ve büyük bir guest additions ekosistemine sahip, çapraz platform (Windows/Linux/Intel Mac) çalışan bir type-2 hipervizördür. UTM ise QEMU tabanlı bir hipervizördür ve öncelikle Apple Silicon (M1/M2/M3) Mac'lerde kullanılır, çünkü VirtualBox ARM tabanlı Mac'leri desteklemez; VirtualBox'ın `.vdi`'si yerine `.qcow2` disk imajlarını kullanır. İşlevsel olarak ikisi de proje için gereken sanallaştırmayı sağlar — seçim büyük ölçüde host makinenin CPU mimarisine bağlıdır.

### Design choices / Tasarım kararları

**EN:**
- **Partitioning:** at least 2 encrypted LVM partitions were created (see the `lsblk` layout in the defense notes) — separate `/`, `/var`, `/tmp`, `/home` and `swap` logical volumes, with `/boot` kept unencrypted since it must be readable before the disk is decrypted.
- **User management:** in addition to `root`, a user matching the login is created and added to both the `sudo` and `user42` groups.
- **Security policies:** a strong password policy (`pwquality.conf`, `login.defs`/`chage`) and a hardened sudo configuration (`passwd_tries=3`, custom bad-password message, logging to `/var/log/sudo/`, TTY mode, restricted `secure_path`) were applied.
- **Services installed:** `openssh-server` (port 4242, root login disabled), `ufw` (only port 4242 open), `apparmor`, `cron` (for the monitoring script).

**TR:**
- **Partitionlama:** en az 2 şifreli LVM partition oluşturuldu (savunma notlarındaki `lsblk` çıktısına bakılabilir) — ayrı `/`, `/var`, `/tmp`, `/home` ve `swap` logical volume'ları, `/boot` ise disk şifresi çözülmeden önce okunabilmesi gerektiği için şifrelemenin dışında bırakıldı.
- **Kullanıcı yönetimi:** `root`'a ek olarak, login ile aynı isimde bir kullanıcı oluşturuldu ve hem `sudo` hem `user42` gruplarına eklendi.
- **Güvenlik politikaları:** güçlü bir şifre politikası (`pwquality.conf`, `login.defs`/`chage`) ve sıkılaştırılmış bir sudo yapılandırması (`passwd_tries=3`, özel hatalı şifre mesajı, `/var/log/sudo/` altında loglama, TTY modu, kısıtlanmış `secure_path`) uygulandı.
- **Kurulu servisler:** `openssh-server` (port 4242, root girişi kapalı), `ufw` (sadece port 4242 açık), `apparmor`, `cron` (monitoring script için).
