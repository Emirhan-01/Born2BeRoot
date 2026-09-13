# Born2beRoot – Savunma Notları

Bu doküman, birinin sıfırdan aynı kurulumu yapabilmesi için mantıksal sırayla düzenlenmiştir: disk bölümlendirmeden başlayıp, kullanıcı/grup yönetimi, güvenlik sıkılaştırmaları ve son olarak SSH/firewall/AppArmor ile biter.

> Not: Şifre politikası (pwquality) ve `secure_path` kısımlarında 42'nin Born2beRoot subject'inde standart olarak istenen değerleri kullandım. Kendi sistemindeki gerçek değerlerle (`/etc/security/pwquality.conf`, `sudo -l | grep secure_path`) karşılaştırıp farklıysa güncelle.

---

## 1. Disk bölümlendirme (LVM)

Kurulum sırasında (grafik/otomatik kurulum ekranında) "Manual" veya "Guided - use entire disk and set up LVM" seçilerek şu bölümler oluşturuldu:

- `/` (root)
- `/home`
- `/var`
- `/tmp`
- `/var/log`
- `swap`

Bunlar LVM (Logical Volume Manager) üzerinden **Physical Volume → Volume Group → Logical Volume** olarak yapılandırıldı.

**Savunmada:** LVM, disk alanını fiziksel olarak sabit bölümlere hapsetmek yerine mantıksal birimlere (LV) bölmeyi sağlar; bir LV'yi daha sonra büyütüp küçültebilirsin. Kontrol komutu:

```bash
lsblk
```

Örnek çıktı ve okunuşu:

```
NAME                        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                           8:0    0   20G  0 disk  
├─sda1                        8:1    0  918M  0 part  /boot
├─sda2                        8:2    0    1K  0 part  
└─sda5                        8:5    0 19,1G  0 part  
  └─sda5_crypt              254:0    0 19,1G  0 crypt 
    ├─embostan42--vg-root   254:1    0    7G  0 lvm   /
    ├─embostan42--vg-var    254:2    0    2G  0 lvm   /var
    ├─embostan42--vg-swap_1 254:3    0  936M  0 lvm   [SWAP]
    ├─embostan42--vg-tmp    254:4    0  568M  0 lvm   /tmp
    └─embostan42--vg-home   254:5    0  8,6G  0 lvm   /home
```

### `lsblk` sütun başlıklarının anlamı

- **MAJ:MIN** — Kernel'in cihazı tanımak için kullandığı major:minor numarası (major = cihaz türünü/sürücüsünü, minor = o türdeki kaçıncı cihaz olduğunu belirtir). Savunmada genelde detaya girmen gerekmez, sadece kernel'in cihazı içeride nasıl numaralandırdığını gösterir.
- **RM** — *Removable*: cihaz çıkarılabilir mi? `1` = evet (USB bellek, harici disk gibi), `0` = hayır (sabit disk). Bizim durumda `0`, çünkü VM'e bağlı sanal disk çıkarılabilir bir cihaz değil.
- **SIZE** — Cihazın/bölümün boyutu.
- **RO** — *Read-Only*: cihaz salt-okunur mu? `1` = evet (örneğin bir CD/DVD sürücüsü), `0` = hayır, yazılabilir. `sr0` (CD-ROM) için `1` görülür, disk bölümleri için `0`.
- **TYPE** — Cihazın türü: `disk` (fiziksel/sanal disk), `part` (partition), `crypt` (LUKS ile şifrelenmiş bir bölümün çözülmüş hali), `lvm` (LVM Logical Volume), `rom` (CD/DVD sürücüsü).
- **MOUNTPOINTS** — Cihazın dosya sisteminde nereye bağlı (mount edilmiş) olduğu; boşsa o cihaz/bölüm hiçbir yere mount edilmemiş demektir (örneğin extended partition veya henüz mount edilmemiş bir LV).

- **`sda`** (20G) — VM'e verilen fiziksel disk.
- **`sda1`** (918M, `/boot`) — Ayrı, **şifrelenmemiş** bölüm. Çekirdek/bootloader dosyaları burada; disk şifresi girilmeden önce erişilebilmesi gerektiği için bilinçli olarak şifrelemenin dışında tutulur.
- **`sda2`** (1K) — **Extended partition**; kendi başına veri tutmaz, sadece `sda5`'i barındıran bir "kap"tır.
- **`sda5`** (19,1G) — Gerçek (logical) partition, diskin geri kalan tüm alanı.
- **`sda5_crypt`** — `sda5`'in **LUKS ile şifrelenmiş** halinin, açılışta şifre girilip çözüldükten sonraki hali (device-mapper cihazı).
- **`embostan42-vg` LVM yapısı** — `sda5_crypt` bir **Physical Volume (PV)** olarak LVM'e verilmiş, üzerinde `embostan42-vg` adlı **Volume Group (VG)** kurulmuş, içinde 5 **Logical Volume (LV)** var: `root` (`/`), `var` (`/var`), `swap_1` (`[SWAP]`), `tmp` (`/tmp`), `home` (`/home`). (LVM isimlerindeki çift tire `--`, isimdeki tek tirenin kaçış gösterimidir; grup adı aslında `embostan42-vg`.)

**Akış özeti (savunmada tek cümlede):** şifreli `sda5` → şifre girilir → `sda5_crypt` → PV olarak LVM'e verilir → `embostan42-vg` VG → içinde `root/var/swap/tmp/home` LV'leri. `/boot` ise açılış öncesi erişilebilmesi gerektiği için şifrelemenin dışında.

Diğer kontrol komutları:

```bash
sudo fdisk -l          # fiziksel disk/partition görünümü
sudo pvdisplay         # Physical Volume'ler
sudo vgdisplay         # Volume Group
sudo lvdisplay         # Logical Volume'ler
```

---

## 2. Debian kurulumu ve sudo yüklenmesi

Debian kurulumu bittikten sonra (kurulumda sudo yoktu, çünkü Debian netinstall'da varsayılan gelmiyor):

```bash
su -                       # root'a geç
apt update
apt install sudo -y
```

**Savunmada:** `sudo`, root şifresini paylaşmadan, yetkilendirilmiş kullanıcıların tek tek komutları root olarak çalıştırmasına izin verir.

---

## 3. Kullanıcı ve grup yönetimi

### 3.1 Yeni kullanıcı ekleme

```bash
sudo adduser embostan
```

(`adduser`, `useradd`'a göre daha "interaktif" bir araçtır: ev dizinini otomatik oluşturur, şifreyi sorar, kullanıcı bilgilerini adım adım ister. `useradd` kullanılırsa bu adımların çoğu elle yapılmalıdır: `sudo useradd -m embostan` ardından `sudo passwd embostan`.)

### 3.2 Kullanıcı şifresi değiştirme

#### Kendi kullanıcımız için
```bash
passwd
```

#### Başka kullanıcı için(root için de geçerli "sudo passwd root")
```bash
sudo passwd <kullanıcı_adı>
```

### 3.3 Kullanıcıyı sudo grubuna ekleme

```bash
sudo usermod -aG sudo embostan
```

Kontrol:

```bash
groups embostan
```

**Savunmada:** `-aG` (append + group) kullanılmazsa kullanıcı diğer gruplardan çıkarılıp sadece belirtilen gruba eklenir; `-a` bunu önler, mevcut gruplara **ekleme** yapar.

### 3.4 Yeni grup oluşturma ve kullanıcıyı ekleme (user42)

```bash
sudo groupadd user42
sudo usermod -aG user42 embostan
```

Kontrol:

```bash
getent group user42
groups embostan
```

> Not: 3.3 ve 34'ü tek komutla da yapabilirsin: `sudo usermod -aG sudo,user42 embostan`

### 3.5 Kullanıcı değiştirme

Başka bir kullanıcıya geçmek için:

```bash
su - embostan
```

(`-` işareti, hedef kullanıcının kendi ortamına — home dizini, PATH, environment değişkenleri — geçmeni sağlar; `-` olmadan sadece kimlik değişir ama eski kullanıcının ortamında kalırsın.)

Sudo yetkin varsa root'a geçmeden komut çalıştırmak için:

```bash
sudo -i        # root shell aç
sudo -u embostan <komut>   # tek bir komutu embostan olarak çalıştır
```

### 3.6 Kullanıcı ve grup kaldırma

Kullanıcıyı (ve istersen ev dizinini) silmek:

```bash
sudo deluser embostan          # sadece kullanıcıyı siler
sudo deluser --remove-home embostan   # kullanıcıyı + ev dizinini siler
```

(`userdel` de aynı işi yapar: `sudo userdel -r embostan`, `-r` ev dizinini de siler.)

Grubu silmek:

```bash
sudo groupdel user42
```

**Savunmada:** Bir grup, hâlâ birincil (primary) grup olarak bir kullanıcıya atanmışsa silinemez; önce o kullanıcıların birincil grubunu değiştirmen gerekir.

---

## 4. Sudo'nun kullanabileceği yolların (PATH) kısıtlanması

Subject'te geçen "For security reasons, the paths that can be used by sudo must also be restricted." maddesi, sudoers dosyasında `secure_path` ayarıyla karşılanır.

```bash
sudo visudo
```

(`visudo`, sudoers dosyasını doğrudan `vim /etc/sudoers` ile açmak yerine kullanılır — syntax hatası varsa dosyayı **kaydetmeden önce** uyarır; bu sayede sudo'yu tamamen bozup kilitli kalma riskini önler.)

Dosyaya (veya `/etc/sudoers.d/` altına yeni bir dosya olarak) şu satır eklendi:

```
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

Kontrol:

```bash
sudo -l | grep secure_path
```

**Savunmada:** Bu ayar olmadan sudo, kullanıcının kendi `$PATH` değişkenindeki (mesela `/home/embostan/bin` gibi) dizinlerden komut çalıştırabilir. Bu, kullanıcının kendi yazdığı sahte bir `ls` veya `sudo` betiğini gerçek komutun önüne koyup çalıştırmasına (PATH hijacking) izin verebilir. `secure_path`, sudo çalışırken hangi dizinlerin aranacağını sabit ve güvenilir bir listeyle sınırlayarak bu riski kapatır.

---

## 5. Şifre politikası (pwquality.conf)

Paket kurulumu:

```bash
sudo apt install libpam-pwquality -y
```

Dosya: `/etc/security/pwquality.conf`

```bash
sudo vim /etc/security/pwquality.conf
```

Aşağıdaki satırlar ayarlandı (yorum işaretleri `#` kaldırılarak):

```
minlen = 10
ucredit = -1
lcredit = -1
dcredit = -1
ocredit = -1
maxrepeat = 3
usercheck = 1
difok = 7
gecoscheck = 1
```

Anlamları:

| Parametre | Anlamı |
|---|---|
| `minlen = 10` | Şifre en az 10 karakter olmalı |
| `ucredit = -1` | En az 1 büyük harf zorunlu |
| `lcredit = -1` | En az 1 küçük harf zorunlu |
| `dcredit = -1` | En az 1 rakam zorunlu |
| `ocredit = -1` | En az 1 özel karakter zorunlu |
| `maxrepeat = 3` | Aynı karakter art arda en fazla 3 kez tekrar edebilir |
| `usercheck = 1` | Şifre, kullanıcı adını içeremez |
| `difok = 7` | Eski şifreyle yeni şifre en az 7 karakter farklı olmalı |
| `gecoscheck = 1` | Şifre, kullanıcının GECOS bilgilerini (ad-soyad vb.) içeremez |


#### Belirli sayıdaki geçmiş şifreyi engellemek için("pwquality.conf" bu kontrolü desteklemez)
```bash
sudo vim /etc/pam.d/common-password
```
"password	[success=1 default=ignore]	pam_unix.so obscure use_authtok try_first_pass yescrypt" satırının sonuna
remember=3 ekle böylece önceki 3 şifreyi engellemiş olur


---

## 6. Şifre süresi/aging (login.defs + chage)

Yeni oluşturulacak kullanıcılar için varsayılan değerler, dosya: `/etc/login.defs`

```bash
sudo vim /etc/login.defs
```

```
PASS_MAX_DAYS   30
PASS_MIN_DAYS   2
PASS_WARN_AGE   7
```

Bu değerlerin **mevcut** kullanıcıya da uygulanması için:

```bash
sudo chage -M 30 -m 2 -W 7 embostan
```

Kontrol:

```bash
sudo chage -l embostan
```

**Savunmada:** `login.defs` sadece **yeni oluşturulacak** kullanıcılar için varsayılan değer belirler; **var olan** bir kullanıcıya uygulamak için `chage` komutu kullanılması gerektiğini vurgulamak önemli — jüri bunu sık sorar.

---

## 7. Host'tan VM'e SSH ile bağlanma

VM tarafında SSH sunucusu kuruldu ve etkinleştirildi:

```bash
sudo apt install openssh-server -y
sudo systemctl status ssh      # servis çalışıyor mu kontrolü
```

VM'in IP adresi:

```bash
ip addr show
# veya
hostname -I
```

Host makineden (Mac/Windows/Linux terminalinden) bağlantı:

```bash
ssh embostan@<VM_IP_ADRESI> -p 4242
```

---

## 8. SSH portunun 4242 yapılması

Dosya: `/etc/ssh/sshd_config`

```bash
sudo vim /etc/ssh/sshd_config
```

İçinde şu satır bulunup değiştirildi (veya yorum satırından çıkarılıp değiştirildi):

```
Port 4242
```

Değişikliğin etkili olması için SSH servisi yeniden başlatıldı:

```bash
sudo systemctl restart ssh
```

### Alternatif: dosyayı açmadan, sadece komutla ekleme/silme

**Port satırını eklemek (dosyayı açmadan):**

```bash
echo "Port 4242" | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart ssh
```

`tee -a`, verilen metni dosyanın sonuna (append) ekler; `sudo` ile birlikte kullanılması gerekir çünkü `>>` yönlendirmesi sudo yetkisiyle çalışmaz (redirection shell tarafından, sudo'dan önce yorumlanır).

**Eklenen satırı silmek (dosyayı açmadan):**

```bash
sudo sed -i '/^Port 4242$/d' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

`sed -i` dosyayı yerinde (in-place) düzenler; `/^Port 4242$/d` deseniyle eşleşen satırı siler.

---

## 9. ufw (firewall) kurulumu ve kuralları

```bash
sudo apt install ufw -y
sudo ufw allow 4242
sudo ufw deny 22          # varsayılan 22 portu kapatıldı/reddedildi
sudo ufw enable
sudo ufw status verbose   # kuralları kontrol etmek için
```

Komutla ekleme/silme (dosya açmaya gerek yok, zaten komutla yönetiliyor):

```bash
sudo ufw allow 4242         # kuralı ekle
sudo ufw delete allow 4242  # kuralı sil
```

**Savunmada:** Sadece port değiştirmek yetmez, ufw üzerinde de yeni portun açık, eski portun (22) kapalı olduğunu göstermen istenir.

---

## 10. AppArmor kurulumu

```bash
sudo apt install apparmor apparmor-utils -y
```

Durum kontrolü:

```bash
sudo aa-status
sudo systemctl status apparmor
```

**Savunmada:** AppArmor bir MAC (Mandatory Access Control) aracıdır; her uygulama için profil bazlı erişim sınırları koyar (hangi dosyalara okuma/yazma yapabileceği gibi). `aa-status` çıktısında profillerin "enforce" modunda olduğunu gösterebilirsin.

---

## Genel kontrol / doğrulama komutları (savunma sırasında canlı gösterebileceğin)

```bash
lsblk                        # partition/LVM görünümü
groups embostan               # grup üyelikleri
sudo -l | grep secure_path    # sudo path kısıtlaması
sudo chage -l embostan         # şifre politikası kullanıcıya uygulanmış mı
sudo systemctl status ssh     # SSH servis durumu ve portu
sudo ufw status verbose       # firewall kuralları
sudo aa-status                # AppArmor profilleri
```
