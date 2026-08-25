
> [!NOTE] 
>  ##### Değiştirilebilir alanlar: Bu notta geçen aşağıdaki değerler **senin ortamına özel** — her yerde arayıp güncel değerle değiştir:
> 
> - `omero` → kullanıcı adı
> - `10.77.191.37` → cihazın ilk IP adresi (ağ değişebilir)
> - `10.42.0.1` → hotspot modundayken Pi'nin sabit IP'si
> - `frambuazlikek` / `12345678` → SSID ve şifre
> - `wlan0`, `wlan1`, `usb0`, `eth0` → arayüz adları (cihaza göre değişebilir)
> -  `EthernetPaylasim`, `TelefonKablo` → nmcli bağlantı profili isimleri
> -  `raspi` → ssh kısayol ismi

## 1 - Pi Imager ile Kurulum

> [!TIP]
> ##### Kurulum Ayarları
> - Cihaz: Rasperry Pi 4
> - İşletim Sistemi: Other → Pi OS Lite 64 bit
> - Depolama: micro sd kart
> - Ana Makine Adı: frambuazlikek
> - Yerelleştirme: TR
> - Kullanıcı Adı: omero (ve parola)
> - Wifi Ağı SSID: telefonun hotspotu ve parolası (bilgisayar da aynı ağa bağlı olacak)
> - SSH Etkinleştir (parola doğrulaması ile)
> - Pi Connect: gerek yok
>

Linux'ta doğrulama adımı hata verirse **doğrulamayı atla**.

Yazma işlemi bitince terminalde şunu çalıştır ve bir süre bekle (kartın güvenle çıkarılması için):

```bash
sync
```

---

## 2 - İlk SSH Bağlantısı

cihazın adı ve telefona bağlı ip adresi

```bash
ssh omero@10.77.191.37
```

> [!tip] Bağlanırken `yes` de ve ardından şifreni gir.

> [!warning] Host key hatası: Aynı cihaza yeniden kurulum yaptıysan (aynı IP, yeni SSH anahtarı) bağlantı **host key** hatası verebilir. Bu durumda önce parmak izini sıfırla:
> 
> ```bash
> ssh-keygen -R 10.77.191.37
> ```

✅ **İçerideyiz.**  

---

## 3 - Pi'yi Router'a (Hotspot) Dönüştürme

```bash
sudo nmcli device wifi hotspot ifname wlan0 ssid frambuazlikek password "12345678" band bg
```

- Cihaz adını (`ssid`) ve şifreyi (`password`) istediğin gibi değiştirebilirsin.
- PIN uyarısı gelirse **"parola gir"** seçeneğini seçip belirlediğin şifreyi gir.

### Hotspot'a Bağlanınca Tekrar Sızmak İçin

```bash
ssh omero@10.42.0.1
```

> [!note] Sabit IP Hotspot modundayken Pi'nin IP'si her zaman `10.42.0.1` olur.

> [!warning] Yine host key hatası alırsan
> 
> ```bash
> ssh-keygen -R 10.42.0.1
> ```

---

## 3 Buçuk - Şifresiz Bağlanmak İçin SSH Key

> [!note] Bu ayar kendi bilgisayarının terminalinde yapılacak. Terminalde `exit` diyerek ssh bağlantısından çıkabilirsin.

> [!danger] Bu bilgisayarında sadece tek seferlik oluşturulması gereken bir anahtar. Geçmişte başka bir sebeple oluşturduysan override yapma. Diğer adımla devam et.
> 
> ```bash
> ssh-keygen -t ed25519
> ```

Anahtarı Raspberry Pi'ye tanıt

```bash
ssh-copy-id omero@10.42.0.1
```

Girişi kolaylaştırmak için kısayol tanımlayacağız. Dosyayı aç.

```bash
nano ~/.ssh/config
```

Bunları yapıştır. Kaydet ve çık `Ctrl+O`, `Enter`, `Ctrl+X`.

```bash
Host raspi 
HostName 10.42.0.1 
User omero
```


> [!success] Artık `ssh raspi` yazarak içeri girebilirsin 

---

## 3 Buçuk Buçuk - Şifre ile girişi tamamen kapat (daha sert güvenlik için)

SSH konfigürasyon dosyasını aç:

```bash
sudo nano /etc/ssh/sshd_config
```

Dosya içinde **`PasswordAuthentication`** satırını bul. Eğer başında **`#`** işareti varsa kaldır. Ayarı şu şekilde güncelle (eğer bu ayar yoksa en alta ekle) 

```toml
PasswordAuthentication no
```

Dosyayı kaydedip çık (`Ctrl+O`, `Enter`, `Ctrl+X`) ve SSH servisini yeniden başlat:

```bash
sudo systemctl restart ssh
```

---

## 4 - Hotspot Profilini Kalıcı ve Öncelikli Yapma

Bağlantı kopsa bile profilin her zaman otomatik açılmasını sağla:

```bash
sudo nmcli connection modify Hotspot connection.autoconnect yes
```

Pi'nin bu profili kendi kendine kapatmasına izin verme:

```bash
sudo nmcli connection modify Hotspot connection.wait-device-timeout 0
```

WPS'yi devre dışı bırak:

```bash
sudo nmcli connection modify Hotspot 802-11-wireless-security.wps-method 1
```

WiFi güç tasarrufunu kapat:

```bash
sudo nmcli connection modify Hotspot 802-11-wireless.powersave 2
```

İnternete bu profilden çıkış yapılmasını engellemek için yüksek metrik ver

```bash
sudo nmcli connection modify Hotspot ipv4.route-metric 600
```

---

## 5 - IP Forwarding

Dosyayı aç:

```bash
sudo nano /etc/sysctl.conf
```

Şu satırları ekle (varsa başlarındaki `#` işaretini kaldır, yoksa en alta yaz):

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.disable_ipv6 = 1 
net.ipv6.conf.default.disable_ipv6 = 1
```

Kaydet: `Ctrl+X` → `Y` → `Enter`

Ayarı aktif et:

```bash
sudo sysctl -p
```


---

## 6 - Gerekli Paketler

Kurulum sırasında gelen mavi ekranlara **yes** de:

```bash
sudo apt update && sudo apt install iptables iptables-persistent -y
```

---

## 7 - TTL Manipülasyonu

```bash
sudo iptables -t mangle -A POSTROUTING -j TTL --ttl-set 65
```

Kuralı hafızaya yaz — her açılışta otomatik devreye girsin:

```bash
sudo netfilter-persistent save
```

---

## 8 - Ethernet Paylaşımı

```bash
sudo nmcli connection add type ethernet ifname eth0 con-name EthernetPaylasim ipv4.method shared
```

Otomatik bağlansın:

```bash
sudo nmcli connection modify EthernetPaylasim connection.autoconnect yes
```

Buna da yüksek metrik veriyoruz ki sistem internete çıkış portu olarak algılamasın

```bash
sudo nmcli connection modify EthernetPaylasim ipv4.route-metric 100
```

Profili bir kereliğine elle başlat:

```bash
sudo nmcli connection up EthernetPaylasim
```

---

## 9 - USB Kablo (Telefon) Profili

`usb0` portuna özel, telefonun MAC adresini umursamayan kalıcı bir profil oluştur — hangi telefon takılırsa takılsın sorunsuz çalışsın:

```bash
sudo nmcli connection add type ethernet con-name "TelefonKablo" ifname usb0
```

Saniyeler içinde otomatik bağlansın, onay beklemesin:

```bash
sudo nmcli connection modify "TelefonKablo" connection.autoconnect yes
```

İnternet önceliğini (metric) en tepeye çıkar — **düşük rakam = yüksek öncelik**:

```bash
sudo nmcli connection modify "TelefonKablo" ipv4.route-metric 10
```

Profili bir kereliğine elle başlat:

```bash
sudo nmcli connection up TelefonKablo
```

> [!warning] Ayar değiştirince Hotspot'u baştan başlatman gerekir, bağlantı kısa süreliğine gidip gelir:
> 
> ```bash
> sudo nmcli connection up Hotspot
> ```

---

## 10 - Genel Bakım Komutları

**Sistemi yeniden başlat:**

```bash
sudo reboot
```

**TTL kuralını ve üzerinden geçen paket sayısını canlı gör:**

```bash
sudo iptables -t mangle -L POSTROUTING -v -n
```