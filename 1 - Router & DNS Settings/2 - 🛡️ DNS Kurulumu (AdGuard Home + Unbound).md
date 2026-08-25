
> [!NOTE] 
> #### Ön Koşul:
> Bu kurulum,  [[1- 📡  Router Kurulum Notları]]  tamamlanmış ve Pi hotspot olarak çalışıyor varsayımıyla ilerler.

---
## 0 - CasaOS Kurulumu
Şimdi kullanmayacağız ama ileride işimize yarayacak. Önce bunu kurmakta fayda var:
```bash
curl -fsSL https://get.casaos.io | sudo bash
```


## 1 - Unbound Kurulumu

Paket listesini güncelle ve Unbound'u kur:

```bash
sudo apt update && sudo apt install unbound -y
```

AdGuard Home'u kur: (bu adım sadece indirir ve servisi hazırlar, portu henüz işgal etmez)

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

---

## 2 - Unbound Konfigürasyonu

AdGuard'a özel, optimize edilmiş konfigürasyon dosyasını tek seferde oluştur:

```bash
sudo bash -c 'cat <<EOF > /etc/unbound/unbound.conf.d/adguard.conf
server:
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # Güvenlik ve Gizlilik Ayarları
    access-control: 127.0.0.0/8 allow
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no

    # Performans ve Önbellek (Cache) Optimizasyonları
    edns-buffer-size: 1232
    prefetch: yes
    prefetch-key: yes
    num-threads: 1
    msg-cache-slabs: 1
    rrset-cache-slabs: 1
    infra-cache-slabs: 1
    key-cache-slabs: 1

    # Pi Donanımı İçin Bellek Ayarları
    so-rcvbuf: 4m
    so-sndbuf: 4m
    msg-cache-size: 4m
    rrset-cache-size: 8m
EOF'
```

Kuralın devreye girmesi için Unbound servisini yeniden başlat ve kalıcı yap:

```bash
sudo systemctl enable unbound
sudo systemctl restart unbound
```

---

## 3 - Koltuk Savaşları (Port 53 Çakışması)

> [!WARNING] 
> #### Neden gerekli
> Pi üzerinde birden fazla servis (NetworkManager, dnsmasq, Unbound, AdGuard) aynı anda **port 53**'ü kullanmak isteyebilir. Bu bölüm, port 53'ü tamamen AdGuard Home'a bırakıp diğerlerini devre dışı bırakır. İnatla koltuğu işgal edenleri indirip internetin cumhurbaşkanlığı koltuğuna gerçekten hizmet edecek adamı koyuyoruz...

### 3.1 - NetworkManager'ın DNS Yönetimini Kapat

```bash
sudo bash -c 'cat <<EOF > /etc/NetworkManager/conf.d/no-dns.conf
[main]
dns=none
EOF'
```

### 3.2 - Port 53'ü Boşalt, Bağlı Cihazlara Pi'yi DNS Olarak Göster

```bash
sudo mkdir -p /etc/NetworkManager/dnsmasq-shared.d
sudo bash -c 'cat <<EOF > /etc/NetworkManager/dnsmasq-shared.d/port0.conf
port=0
dhcp-option=option:dns-server,10.42.0.1
EOF'
```

### 3.3 - Pi'nin Kendi DNS Rotasını Localhost'a Sabitle

```bash
sudo rm -f /etc/resolv.conf
sudo bash -c 'cat <<EOF > /etc/resolv.conf
nameserver 127.0.0.1
EOF'
```

---

## 4 - Yeniden Başlat

Ayarların devreye girmesi için sistemi yeniden başlat:

```bash
sudo reboot
```

---

## 5. AdGuard Kurulumu ve Ayarları

Sistem açıldıktan sonra tarayıcıdan AdGuard Home kurulum arayüzüne gir:

```
http://10.42.0.1:3000
```

### 5.1 Sihirbaz Ayarları

|Ayar|Değer|Not|
|---|---|---|
|Admin Web Interface|`8080`|Port `80` **değil** — orada CasaOS çalışıyor|
|DNS server port|`53`|Kesinlikle bu — port 53 için verilen tüm savaş bu ayar için|

> [!WARNING] 
> Sihirbazda başka ayar değiştirme Yukarıdaki iki port dışında hiçbir şeye dokunma. Kullanıcı adı ve şifreni gir, kurulumu tamamla.

---
> [!IMPORTANT] 
> Artık Adguard paneline http://10.42.0.1:8080 adresinden ulaşabilirsin.

---

### 5.2 DNS (Upstream) — Unbound Bağlantısı

```
127.0.0.1:5335
```

Bu, AdGuard Home'un tüm sorguları yerel Unbound sunucusuna yönlendirmesini sağlar.

### 5.3 Fallback DNS Servers

Unbound yanıt vermezse sorguyu bunlara atar.

```
1.1.1.1
8.8.8.8
```

### 5.4 DNS Cache Configuration

| Ayar                 | Değer      | Not                                                             |
| -------------------- | ---------- | --------------------------------------------------------------- |
| Cache size           | `10485760` | Hafızada veri tutmak için açtığımız alan. 10 mb                 |
| Override minimum TTL | `3600`     | Sürekli istek atan şapşalları durdurmak için. Min 1 saat kuralı |
| Optimistic caching   | `true`     | Gecikmeyi azaltacak bir ayar                                    |

### 5.5 Blocklist

- `oisd big`
- `tr list`


---

## 6. Kontrol Mekanizmaları

Port kontrolü — hangi servisin hangi portu dinlediğini doğrula:

```bash
sudo ss -tulpn | grep :53
sudo ss -tulpn | grep :5335
```