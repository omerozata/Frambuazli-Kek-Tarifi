
---

### Tam Proje Ağacı

```
pinet-api/
├── .env                  (git'e girmez)
├── .env.example
├── .gitignore
├── requirements.txt
├── main.py
├── api_env/               (git'e girmez)
├── app/
│   ├── __init__.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py
│   │   ├── logging_config.py
│   │   ├── security.py
│   │   ├── system_worker.py
│   │   └── nmcli_worker.py
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── system.py
│   │   └── vpn.py
│   └── schemas/
│       ├── __init__.py
│       ├── system.py
│       └── vpn.py
└── deployment/
    ├── pinet-api.service
    └── pinet-sudoers
```


---

##   API Referansı

Bu bölüm, Pi-Net API'nin sunduğu tüm uç noktaların (endpoint) resmi referansıdır. Herhangi bir istemci buradaki tanımlara göre istek gönderebilir.

### Temel Bilgiler

|||
|---|---|
|**Taban adres**|`http://10.42.0.1:8000`|
|**Veri formatı**|JSON (dosya yükleme hariç)|
|**Karakter kodlaması**|UTF-8|
|**Kimlik doğrulama**|`X-API-Key` header (`GET /` hariç tüm uç noktalarda zorunlu)|

### Kimlik Doğrulama

`GET /` dışındaki her istek, header'da geçerli bir API anahtarı taşımalıdır:

```
X-API-Key: <anahtar>
```

Anahtar eksikse `401`, yanlışsa `403` döner (bkz. [Hata Cevapları](https://claude.ai/chat/fa19508c-a0bc-4a75-9f76-72fa2802e215#hata-cevaplar%C4%B1)).

### İnteraktif Dokümantasyon

API çalışırken, tüm uç noktaları tarayıcı üzerinden deneyebileceğin bir arayüz otomatik olarak sunulur:

```
http://10.42.0.1:8000/docs
```

---

## Uç Noktalar

### `GET /`

Servisin ayakta olup olmadığını kontrol eder. Kimlik doğrulama gerektirmez.

**İstek**

```
GET /
```

**Cevap — `200 OK`**

```json
{
  "service": "Pi-Net VPN Gateway API",
  "version": "1.0.0",
  "status": "running"
}
```

|Alan|Tip|Açıklama|
|---|---|---|
|`service`|string|Servisin adı|
|`version`|string|Servis sürüm numarası|
|`status`|string|Her zaman `"running"`|

---

### `GET /api/system/durum`

Anlık CPU, RAM ve disk metriklerini döner.

**İstek**

```
GET /api/system/durum
X-API-Key: <anahtar>
```

**Cevap — `200 OK`**

```json
{
  "status": "success",
  "data": {
    "cpu": {
      "temperature_c": 47.8,
      "usage_percent": 12.5,
      "temperature_available": true
    },
    "ram": {
      "total_gb": 3.7,
      "used_gb": 1.2,
      "usage_percent": 32.4
    },
    "disk": {
      "total_gb": 29.5,
      "used_gb": 8.1,
      "usage_percent": 27.5
    }
  }
}
```

|Alan|Tip|Açıklama|
|---|---|---|
|`data.cpu.temperature_c`|number|CPU sıcaklığı (°C)|
|`data.cpu.usage_percent`|number|CPU kullanım yüzdesi (0–100)|
|`data.cpu.temperature_available`|boolean|Sıcaklık sensörü okunabildi mi. `false` ise `temperature_c` anlamsızdır (sensör okunamadı)|
|`data.ram.total_gb`|number|Toplam RAM (GB)|
|`data.ram.used_gb`|number|Kullanılan RAM (GB)|
|`data.ram.usage_percent`|number|RAM kullanım yüzdesi (0–100)|
|`data.disk.total_gb`|number|Toplam disk alanı (GB)|
|`data.disk.used_gb`|number|Kullanılan disk alanı (GB)|
|`data.disk.usage_percent`|number|Disk kullanım yüzdesi (0–100)|

---

### `GET /api/vpn/list`

Sistemde kayıtlı tüm WireGuard VPN profillerini listeler.

**İstek**

```
GET /api/vpn/list
X-API-Key: <anahtar>
```

**Cevap — `200 OK`**

```json
{
  "profiles": ["proton-nl", "proton-uk", "wg-home"]
}
```

|Alan|Tip|Açıklama|
|---|---|---|
|`profiles`|string[]|Kayıtlı VPN profillerinin adları. Hiç profil yoksa boş dizi döner|

---

### `GET /api/vpn/durum`

Şu an aktif bir VPN tüneli olup olmadığını, varsa hangisi olduğunu döner.

**İstek**

```
GET /api/vpn/durum
X-API-Key: <anahtar>
```

**Cevap — `200 OK` (tünel açık)**

```json
{
  "online": true,
  "active_connection": "proton-nl"
}
```

**Cevap — `200 OK` (tünel kapalı)**

```json
{
  "online": false,
  "active_connection": null
}
```

|Alan|Tip|Açıklama|
|---|---|---|
|`online`|boolean|Aktif bir VPN tüneli var mı|
|`active_connection`|string \| null|Aktif profilin adı; hiçbiri aktif değilse `null`|

---

### `POST /api/vpn/baglan`

Belirtilen profile bağlanır. Bağlanmadan önce, o an açık olan başka bir tünel varsa otomatik olarak kapatılır.

**İstek**

```
POST /api/vpn/baglan
X-API-Key: <anahtar>
Content-Type: application/json
```

```json
{
  "profile_name": "proton-nl"
}
```

|Alan|Tip|Zorunlu|Açıklama|
|---|---|---|---|
|`profile_name`|string|Evet|Bağlanılacak profilin adı (1–64 karakter)|

**Cevap — `200 OK`**

```json
{
  "success": true,
  "message": "'proton-nl' tüneli aktif edildi."
}
```

**Olası hatalar:** `422` (geçersiz `profile_name`), `502` (nmcli bağlantıyı kuramadı — bkz. `detail`)

---

### `POST /api/vpn/kapat`

Açık olan VPN bağlantısını keser. Gövde gerektirmez. Zaten açık bir tünel yoksa da başarıyla döner.

**İstek**

```
POST /api/vpn/kapat
X-API-Key: <anahtar>
```

**Cevap — `200 OK`**

```json
{
  "success": true,
  "message": "'proton-nl' tüneli kapatıldı."
}
```

Açık tünel yoksa:

```json
{
  "success": true,
  "message": "Zaten açık bir VPN yok."
}
```

---

### `POST /api/vpn/sil`

Belirtilen profili sistemden kalıcı olarak siler. Profil o an aktifse önce bağlantı kesilir, sonra silinir.

**İstek**

```
POST /api/vpn/sil
X-API-Key: <anahtar>
Content-Type: application/json
```

```json
{
  "profile_name": "proton-nl"
}
```

|Alan|Tip|Zorunlu|Açıklama|
|---|---|---|---|
|`profile_name`|string|Evet|Silinecek profilin adı (1–64 karakter)|

**Cevap — `200 OK`**

```json
{
  "success": true,
  "message": "'proton-nl' profili sistemden silindi."
}
```

**Olası hatalar:** `422` (geçersiz `profile_name`), `502` (nmcli silemedi — bkz. `detail`)

---

### `POST /api/vpn/ekle`

Bir WireGuard `.conf` dosyasını sisteme yükler ve profil olarak kaydeder. Diğer uç noktaların aksine JSON değil, `multipart/form-data` formatında gövde bekler. Yüklenen profil güvenlik amacıyla **pasif** kaydedilir; otomatik olarak bağlanmaz, ayrıca `/api/vpn/baglan` ile etkinleştirilmesi gerekir.

**İstek**

```
POST /api/vpn/ekle
X-API-Key: <anahtar>
Content-Type: multipart/form-data
```

|Alan|Tip|Zorunlu|Açıklama|
|---|---|---|---|
|`display_name`|form alanı (metin)|Evet|Profilin görünecek adı (1–64 karakter)|
|`file`|dosya|Evet|`.conf` uzantılı WireGuard yapılandırma dosyası (azami 64 KB)|

**Cevap — `200 OK`**

```json
{
  "success": true,
  "message": "'proton-de' sisteme eklendi (pasif durumda)."
}
```

**Olası hatalar:** `400` (geçersiz `display_name` veya boş dosya), `413` (dosya boyut sınırını aşıyor), `502` (nmcli içe aktaramadı — bkz. `detail`)

---

## Hata Cevapları

Tüm uç noktalar aynı hata formatını paylaşır:

```json
{
  "detail": "hata açıklaması"
}
```

|Durum kodu|Anlamı|Ne zaman döner|
|---|---|---|
|`400`|Geçersiz istek|Gönderilen alan kurala uymuyor (örn. boş dosya)|
|`401`|Kimlik eksik|`X-API-Key` header'ı hiç gönderilmemiş|
|`403`|Kimlik geçersiz|Gönderilen anahtar yanlış|
|`413`|İçerik çok büyük|Yüklenen dosya boyut sınırını aşıyor|
|`422`|Doğrulama hatası|Gövdedeki bir alan eksik veya kurala uymuyor (örn. `profile_name` boş ya da 64 karakterden uzun)|
|`502`|Sistem hatası|Sunucu üzerindeki `nmcli` komutu başarısız oldu; sebep `detail` alanında belirtilir|
|`500`|Beklenmeyen hata|Sunucu tarafında öngörülmeyen bir durum oluştu|