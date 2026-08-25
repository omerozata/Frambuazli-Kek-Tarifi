
> [!info] Değiştirilebilir alanlar (Tüm ilgili notlar için)
> 
> - `omero` → kullanıcı adı
> - `pinet` → API servisi için oluşturulacak kısıtlı sistem kullanıcısının adı
> - `~/pinet-api` → proje klasör yolu
> - `10.42.0.1` → Pi'nin hotspot modundaki sabit IP'si
> - `8000` → API'nin çalıştığı port

> [!note] Ön koşul: Bu kurulum, [[1- 📡  Router Kurulum Notları]] tamamlanmış ve Pi hotspot olarak çalışıyor varsayımıyla ilerler.

> [!tip] Devamı var: Bu notta API'nin omurgası kurulur ve sistem izleme modülü eklenir. VPN modülü, servis dosyası ve kullanıcı yetkilendirmesi diğer notlarda kuruluyor — Bütün modüller tamamlanmadan sistemi ayağa kaldırmıyoruz.

---

## 1. Gerekli Paketler ve Klasör İskeleti

```bash
sudo apt update
sudo apt upgrade
sudo apt install -y python3-venv
```

Proje klasörlerini oluştur:

```bash
mkdir -p ~/pinet-api/app/core ~/pinet-api/app/routers ~/pinet-api/app/schemas ~/pinet-api/deployment
cd ~/pinet-api

touch app/__init__.py app/core/__init__.py app/routers/__init__.py app/schemas/__init__.py
```

---

## 2. Sanal Ortam ve Kütüphaneler

```bash
python3 -m venv api_env
source api_env/bin/activate
```

Terminalin başında `(api_env)` görünmeli. Gerekli kütüphaneleri kur:

```bash
pip install fastapi uvicorn python-multipart psutil pydantic-settings
pip freeze > requirements.txt
```

> [!tip] Bonus — Sanal Ortama Girmek İçin Kısayol
> 
> ```bash
> nano ~/.bashrc
> ```
> 
> En alta ekle kaydet çık:
> 
> ```bash
> alias apigir='cd ~/pinet-api && source api_env/bin/activate'
> ```
>
> Ardından:
> 
> ```bash
> source ~/.bashrc
> ```
> 
> Artık `apigir` yazarak doğrudan proje klasörüne ve sanal ortama girebilirsin.

---

## 3. Ayar Yönetimi (.env ve config.py)

Bazı ön tanımlı sistem davranışlarını yönetmek ve `.env` dosyasındaki verileri okumak için `config` dosyasını oluştur. Bu dosya herkese açık şekilde git'e eklenecek:

```bash
nano app/core/config.py
```

```python
"""Uygulama genelinde kullanılan ayarları tek noktadan yönetir."""

from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    # --- Güvenlik ---
    api_key: str

    # --- Uygulama bilgisi ---
    app_name: str = "Pi-Net VPN Gateway API"
    app_version: str = "1.0.0"

    # --- nmcli / VPN davranışı ---
    nmcli_timeout_seconds: int = 15
    upload_max_size_bytes: int = 64 * 1024  # 64 KB

    # --- Loglama ---
    log_level: str = "INFO"

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

Asla git'e eklenmeyecek olan `.env` dosyasını oluştur ve güçlü, rastgele bir API anahtarı üret:

```bash
cd ~/pinet-api
echo "API_KEY=$(openssl rand -hex 32)" > .env
chmod 600 .env
```

Anahtarı görmek için:

```bash
cat .env
```

Bu değeri not al — Yönetim panellerinde her istekte `X-API-Key` header'ına bunu yazacaksın.

> [!warning] .env dosyasını asla paylaşma Git'e commit etme, ekran görüntüsü paylaşma, hiçbir yere yapıştırma. Bu değer sızarsa sisteme dışarıdan tam kontrol demektir.

`.gitignore` dosyasını oluştur:

```bash
nano ~/pinet-api/.gitignore
```

```
.env
api_env/
__pycache__/
*.pyc
```

---

## 4. Loglama Katmanı

```bash
nano app/core/logging_config.py
```

```python
"""Uygulama genelinde tutarlı loglama sağlar."""

import logging
import sys


def configure_logging(level: str = "INFO") -> None:
    logging.basicConfig(
        level=level,
        format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
        stream=sys.stdout,
    )


def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(name)
```

---

## 5. API Anahtarı Doğrulaması

```bash
nano app/core/security.py
```

```python
"""API anahtarı doğrulaması."""

import secrets

from fastapi import Security, HTTPException, status, Depends
from fastapi.security.api_key import APIKeyHeader

from app.core.config import Settings, get_settings

API_KEY_NAME = "X-API-Key"
_api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)


async def get_api_key(
    provided_key: str | None = Security(_api_key_header),
    settings: Settings = Depends(get_settings),
) -> str:
    if provided_key is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API anahtarı eksik. 'X-API-Key' header'ı zorunludur.",
        )

    if not secrets.compare_digest(provided_key, settings.api_key):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Geçersiz API anahtarı.",
        )

    return provided_key
```

---

## 6. Sistem İzleme Modülü (Worker)

```bash
nano app/core/system_worker.py
```

```python
"""Raspberry Pi donanım/sistem metriklerini toplar."""

from dataclasses import dataclass

import psutil

from app.core.logging_config import get_logger

logger = get_logger(__name__)

_THERMAL_ZONE_PATH = "/sys/class/thermal/thermal_zone0/temp"


@dataclass(frozen=True)
class CpuMetrics:
    temperature_c: float
    usage_percent: float
    temperature_available: bool


@dataclass(frozen=True)
class MemoryMetrics:
    total_gb: float
    used_gb: float
    usage_percent: float


@dataclass(frozen=True)
class DiskMetrics:
    total_gb: float
    used_gb: float
    usage_percent: float


@dataclass(frozen=True)
class SystemMetrics:
    cpu: CpuMetrics
    ram: MemoryMetrics
    disk: DiskMetrics


def _read_cpu_temperature() -> tuple[float, bool]:
    try:
        with open(_THERMAL_ZONE_PATH, "r") as f:
            raw_millidegrees = float(f.read())
        return round(raw_millidegrees / 1000.0, 1), True
    except FileNotFoundError:
        logger.warning("Sıcaklık sensörü dosyası bulunamadı: %s", _THERMAL_ZONE_PATH)
    except (ValueError, OSError) as exc:
        logger.error("Sıcaklık okunamadı: %s", exc)
    return 0.0, False


def _bytes_to_gb(value_bytes: int) -> float:
    return round(value_bytes / (1024 ** 3), 2)


def get_system_metrics() -> SystemMetrics:
    """Anlık CPU, RAM ve disk metriklerini toplar. ~0.5 saniye sürer."""
    temperature_c, temperature_available = _read_cpu_temperature()
    cpu_usage_percent = psutil.cpu_percent(interval=0.5)

    ram = psutil.virtual_memory()
    disk = psutil.disk_usage("/")

    return SystemMetrics(
        cpu=CpuMetrics(
            temperature_c=temperature_c,
            usage_percent=cpu_usage_percent,
            temperature_available=temperature_available,
        ),
        ram=MemoryMetrics(
            total_gb=_bytes_to_gb(ram.total),
            used_gb=_bytes_to_gb(ram.used),
            usage_percent=ram.percent,
        ),
        disk=DiskMetrics(
            total_gb=_bytes_to_gb(disk.total),
            used_gb=_bytes_to_gb(disk.used),
            usage_percent=disk.percent,
        ),
    )
```

---

## 7. Veri Modelleri (Schemas)

```bash
nano app/schemas/system.py
```

```python
"""Sistem durumu endpoint'leri için response şemaları."""

from pydantic import BaseModel, Field


class CpuInfo(BaseModel):
    temperature_c: float = Field(..., description="CPU sıcaklığı (santigrat derece)")
    usage_percent: float = Field(..., ge=0, le=100)
    temperature_available: bool = Field(
        ..., description="Sıcaklık sensörü başarıyla okunabildi mi"
    )


class MemoryInfo(BaseModel):
    total_gb: float
    used_gb: float
    usage_percent: float = Field(..., ge=0, le=100)


class DiskInfo(BaseModel):
    total_gb: float
    used_gb: float
    usage_percent: float = Field(..., ge=0, le=100)


class SystemMetricsData(BaseModel):
    cpu: CpuInfo
    ram: MemoryInfo
    disk: DiskInfo


class SystemStatusResponse(BaseModel):
    status: str = "success"
    data: SystemMetricsData
```

---

## 8. System Router

```bash
nano app/routers/system.py
```

```python
"""Sistem durumu (CPU/RAM/disk) endpoint'leri."""

from fastapi import APIRouter, Depends
from starlette.concurrency import run_in_threadpool

from app.core import system_worker
from app.core.security import get_api_key
from app.schemas.system import (
    CpuInfo,
    DiskInfo,
    MemoryInfo,
    SystemMetricsData,
    SystemStatusResponse,
)

router = APIRouter(
    prefix="/api/system",
    tags=["Sistem"],
    dependencies=[Depends(get_api_key)],
)


@router.get("/durum", response_model=SystemStatusResponse)
async def get_system_status() -> SystemStatusResponse:
    # psutil.cpu_percent() ~0.5 saniye süren, senkron (blocking) bir
    # çağrıdır. run_in_threadpool ile ayrı bir thread'e devredilerek
    # bu süre boyunca API'nin diğer isteklere cevap vermeye devam
    # etmesi sağlanır.
    metrics = await run_in_threadpool(system_worker.get_system_metrics)

    return SystemStatusResponse(
        data=SystemMetricsData(
            cpu=CpuInfo(
                temperature_c=metrics.cpu.temperature_c,
                usage_percent=metrics.cpu.usage_percent,
                temperature_available=metrics.cpu.temperature_available,
            ),
            ram=MemoryInfo(
                total_gb=metrics.ram.total_gb,
                used_gb=metrics.ram.used_gb,
                usage_percent=metrics.ram.usage_percent,
            ),
            disk=DiskInfo(
                total_gb=metrics.disk.total_gb,
                used_gb=metrics.disk.used_gb,
                usage_percent=metrics.disk.usage_percent,
            ),
        )
    )
```

---

## 9. Ana Motor (main.py)

```bash
nano ~/pinet-api/main.py
```

```python
"""Pi-Net VPN Gateway API - uygulama giriş noktası."""

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from app.core.config import get_settings
from app.core.logging_config import configure_logging, get_logger
from app.routers import system, vpn

settings = get_settings()
configure_logging(settings.log_level)
logger = get_logger(__name__)

app = FastAPI(
    title=settings.app_name,
    description="Raspberry Pi üzerindeki ağ trafiğini ve VPN tünellerini yönetir.",
    version=settings.app_version,
)

app.include_router(system.router)
app.include_router(vpn.router)


@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    logger.exception("Beklenmeyen hata: %s %s", request.method, request.url.path)
    return JSONResponse(
        status_code=500,
        content={"detail": "Sunucuda beklenmeyen bir hata oluştu."},
    )


@app.get("/", tags=["Health Check"])
def read_root():
    return {
        "service": settings.app_name,
        "version": settings.app_version,
        "status": "running",
    }
```

> [!abstract] VPN router'ı henüz yazılmadı. Yukarıdaki dosya `app.routers` içinden `vpn`'i de import ediyor ama bu modül bir sonraki notta yazılacak. Bu yüzden `main.py` şu an tek başına çalıştırılamaz — devam eden nottaki adımları tamamladıktan sonra test edilecek. Servisi systemd olarak kurmak da üçüncü notun konusu.