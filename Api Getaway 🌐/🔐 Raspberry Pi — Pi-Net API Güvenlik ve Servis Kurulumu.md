

> [!info] Değiştirilebilir alanlar
> 
> - `omero` → kullanıcı adı
> - `pinet` → API servisi için oluşturulacak kısıtlı sistem kullanıcısının adı
> - `/home/omero/pinet-api` → proje klasörünün tam yolu (systemd göreli yol kabul etmez)

> [!note] Ön koşul: Bu adımların uygulanabilmesi için [[🧩 Raspberry Pi — Pi-Net API  Omurga ve Sistem İzleme]] ile [[🌍 Raspberry Pi — Pi-Net API  VPN Modülü]] notları tamamlanmış olmalı.

> [!tip] Bu notta ne yapılıyor: API'yi çalıştıracak, sadece `nmcli` komutunu çalıştırmaya yetkili, kısıtlı bir sistem kullanıcısı oluşturuluyor; ardından servis bu kullanıcıyla systemd üzerinden kalıcı hale getiriliyor. API hiçbir aşamada `root` olarak çalışmıyor.

---

## 1. Kısıtlı Sistem Kullanıcısı Oluştur

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin pinet
```

- `--system`: normal bir kullanıcı değil, servis amaçlı bir kullanıcı olarak işaretler.
- `--no-create-home`: ev klasörü oluşturmaz.
- `--shell /usr/sbin/nologin`: bu kullanıcıyla terminale giriş yapılamaz, sadece systemd bu kimlikle servisi çalıştırabilir.

Proje klasörünün sahipliğini yeni kullanıcıya ver:

```bash
sudo chown -R pinet:pinet /home/omero/pinet-api
sudo chmod 600 /home/omero/pinet-api/.env
```


>[!attention] Eğer permission hatası alınırsa ek bir izin vermek gerekebilir. Bu izin `omero` klasörünü, içinden geçilebilir yapar. Klasörün içindeki dosyalar listelenemez veya başka hiçbir yetkiyi etkilemez. Sadece `pinet` kullanıcısının bu yoldan geçip kendi klasörüne ulaşabilmesini sağlar:
>```bash
sudo chmod o+x /home/omero
> ```




---

## 2. Scoped Sudo Kuralı Tanımla

`nmcli connection up/down/delete/modify/import` komutları ağ arayüzlerini değiştirdiği için root yetkisi gerektirir. `pinet` kullanıcısına genel bir root yetkisi vermek yerine, **sadece `nmcli` binary'sini** çalıştırabileceği dar kapsamlı bir kural tanımlanır.

Önce `nmcli`'nin gerçek yolunu doğrula:

```bash
which nmcli
```

Genelde `/usr/bin/nmcli` çıkar. Farklıysa aşağıdaki kuralda yolu buna göre güncelle.

Sudoers dosyasını `visudo` ile oluştur — doğrudan bir metin editörüyle değil, çünkü sözdizimi hatası tüm sudo sistemini kilitleyebilir:

```bash
sudo visudo -f /etc/sudoers.d/pinet-api
```

İçine tek satır yaz:

```
pinet ALL=(root) NOPASSWD: /usr/bin/nmcli
```

Kaydet, çık. İzinleri doğrula:

```bash
sudo chmod 440 /etc/sudoers.d/pinet-api
sudo visudo -c
```

> [!danger] Hata görürsen `visudo -c` çıktısında hata görürsen dosyayı sil ve tekrar dene:
> 
> ```bash
> sudo rm /etc/sudoers.d/pinet-api
> ```

Yetkinin doğru çalıştığını test et — komut **şifre sormadan** çalışmalı:

```bash
sudo -u pinet sudo -n nmcli connection show
```

"a password is required" hatası alırsan sudoers dosyasını tekrar kontrol et.

---

## 3. Systemd Servis Dosyası

```bash
sudo nano /etc/systemd/system/pinet-api.service
```

```ini
[Unit]
Description=Pi-Net VPN Gateway API Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pinet
Group=pinet
WorkingDirectory=/home/omero/pinet-api

EnvironmentFile=/home/omero/pinet-api/.env
Environment="PATH=/home/omero/pinet-api/api_env/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

ExecStart=/home/omero/pinet-api/api_env/bin/uvicorn main:app --host 0.0.0.0 --port 8000

Restart=always
RestartSec=3

; --- Ek sertleştirmeler ---
NoNewPrivileges=false
; false olmalı: sudo çağrısı "yeni yetki kazanma" sayılır (pinet -> root).
; true yapılırsa nmcli komutları çalışmaz.

ProtectSystem=strict
ReadWritePaths=/tmp
ProtectHome=read-only
PrivateTmp=true
; PrivateTmp=true: servise, sistemin geri kalanından izole kendine
; özel bir /tmp verir - .conf yükleme dosyaları böylece diğer
; kullanıcı ve servislerden tamamen gizli kalır.

[Install]
WantedBy=multi-user.target
```

> [!note] `ProtectSystem` / `ProtectHome` ne işe yarar `ProtectSystem=strict`, `/`, `/usr`, `/boot` gibi sistem dizinlerini salt-okunur yapar; sadece `ReadWritePaths` ile belirtilen yerlere (`/tmp`) yazılabilir. `ProtectHome=read-only`, `/home` altındaki dosyalara (proje klasörü dahil) yazma izni vermez — servis kendi kodunu değiştiremez. Bu iki satır, API'de bir açık bulunsa dahi sistemin geri kalanının korunmasını sağlar.

---

## 4. Servisi Başlat

```bash
sudo systemctl daemon-reload
sudo systemctl enable pinet-api
sudo systemctl start pinet-api
sudo systemctl status pinet-api
```

> [!tip] Durum ekranından çıkmak için `q` tuşuna bas.

Canlı logları izlemek için:

```bash
journalctl -u pinet-api -f
```

---

## 5. Uçtan Uca Doğrulama

```bash
# Health check (key gerektirmez)
curl http://localhost:8000/

# Key olmadan - 401 beklenir
curl -i http://localhost:8000/api/system/durum

# Key ile (.env dosyandaki gerçek değeri kullan)
curl -H "X-API-Key: <buraya-.env-deki-deger>" http://localhost:8000/api/system/durum
curl -H "X-API-Key: <buraya-.env-deki-deger>" http://localhost:8000/api/vpn/list
```

Tarayıcıdan interaktif dokümantasyona da erişebilirsin:

```
http://10.42.0.1:8000/docs
```

Sağ üstteki **Authorize** butonuna tıklayıp `.env` dosyandaki `API_KEY` değerini gir — böylece tüm korumalı endpoint'leri buradan deneyebilirsin.

> [!tip] Bir şeyler ters giderse `Permission denied` gibi bir hata görürsen adım 1'deki `chown` komutunu, ya da adım 2'deki sudoers kuralını tekrar kontrol et. Loglardaki (`journalctl -u pinet-api -f`) hata mesajı genelde sorunun tam olarak nerede olduğunu gösterir.

---

