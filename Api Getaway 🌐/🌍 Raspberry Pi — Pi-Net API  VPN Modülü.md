
> [!info] Değiştirilebilir alanlar (Tüm ilgili notlar için)
> 
> - `omero` → kullanıcı adı
> - `pinet` → API servisi için oluşturulacak kısıtlı sistem kullanıcısının adı
> - `~/pinet-api` → proje klasör yolu
> - `10.42.0.1` → Pi'nin hotspot modundaki sabit IP'si
> - `8000` → API'nin çalıştığı port

> [!note] Ön koşul: Bu not, [[🧩 Raspberry Pi — Pi-Net API  Omurga ve Sistem İzleme]] notundaki adımların tamamlanmış olduğu varsayımıyla ilerler.

> [!tip] Devamı var: Bu notta VPN kontrol modülü yazılır. Servisi systemd olarak kurup çalışır hale getirmek için yapılacaklar üçüncü nottadır 

---

## 1. Veri Modelleri (Schemas)

```bash
nano ~/pinet-api/app/schemas/vpn.py
```

```python
"""VPN/nmcli endpoint'leri için request/response şemaları."""

from pydantic import BaseModel, Field


class VpnProfileListResponse(BaseModel):
    profiles: list[str] = Field(..., description="Sistemdeki VPN profillerinin adları")


class VpnStatusResponse(BaseModel):
    online: bool
    active_connection: str | None = Field(
        None, description="Aktif bağlantının adı; hiçbiri aktif değilse null"
    )


class VpnActionResponse(BaseModel):
    success: bool
    message: str


class VpnConnectRequest(BaseModel):
    profile_name: str = Field(..., min_length=1, max_length=64)


class VpnDeleteRequest(BaseModel):
    profile_name: str = Field(..., min_length=1, max_length=64)
```

---

## 2. İş Katmanı (nmcli_worker.py)

Bu dosya, `nmcli` komutlarını çalıştıran tek yerdir. Bağlantı adları burada bir whitelist'ten geçirilerek doğrulanır, komutlar zaman aşımı korumasıyla ve engellemeden (async) çalıştırılır.

```bash
nano ~/pinet-api/app/core/nmcli_worker.py
```

```python
"""NetworkManager (nmcli) ile etkileşim katmanı."""

import asyncio
import re
import shutil
from dataclasses import dataclass

from app.core.logging_config import get_logger

logger = get_logger(__name__)

_NMCLI_PATH = shutil.which("nmcli") or "/usr/bin/nmcli"

# Bağlantı adlarında sadece bu karakter setine izin verilir.
_CONNECTION_NAME_PATTERN = re.compile(r"^[A-Za-z0-9._-]{1,64}$")


class InvalidConnectionNameError(ValueError):
    """Bağlantı adı whitelist'i geçemediğinde fırlatılır."""


def validate_connection_name(name: str) -> str:
    """Bağlantı adının güvenli karakter setiyle sınırlı olduğunu doğrular.

    nmcli'ye argüman olarak geçecek her kullanıcı girdisi bu
    fonksiyondan geçmeden kullanılamaz.
    """
    if not _CONNECTION_NAME_PATTERN.match(name):
        raise InvalidConnectionNameError(
            f"Geçersiz bağlantı adı: {name!r}. Sadece harf, rakam, "
            "nokta, tire ve alt çizgi kullanılabilir (maks. 64 karakter)."
        )
    # "-" ile başlayan bir ad (örn. "--help"), nmcli tarafından bir
    # komut satırı seçeneği gibi yorumlanabileceği için ayrıca reddedilir.
    if name.startswith("-"):
        raise InvalidConnectionNameError(
            f"Geçersiz bağlantı adı: {name!r}. Ad '-' ile başlayamaz."
        )
    return name


@dataclass(frozen=True)
class NmcliResult:
    success: bool
    stdout: str
    stderr: str
    returncode: int


async def _run_command(args: list[str], timeout: int) -> NmcliResult:
    """Verilen komutu asenkron olarak, zaman aşımı korumasıyla çalıştırır."""
    logger.info("Komut çalıştırılıyor: %s", " ".join(args))
    try:
        proc = await asyncio.create_subprocess_exec(
            *args,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        stdout_bytes, stderr_bytes = await asyncio.wait_for(
            proc.communicate(), timeout=timeout
        )
    except asyncio.TimeoutError:
        logger.error("Komut zaman aşımına uğradı (%ss): %s", timeout, args)
        try:
            proc.kill()
        except ProcessLookupError:
            pass
        return NmcliResult(
            success=False,
            stdout="",
            stderr=f"Komut {timeout} saniye içinde tamamlanmadı.",
            returncode=-1,
        )

    stdout = stdout_bytes.decode("utf-8", errors="replace").strip()
    stderr = stderr_bytes.decode("utf-8", errors="replace").strip()

    if proc.returncode != 0:
        logger.warning(
            "Komut başarısız (rc=%s): %s | stderr: %s",
            proc.returncode, args, stderr,
        )

    return NmcliResult(
        success=proc.returncode == 0,
        stdout=stdout,
        stderr=stderr,
        returncode=proc.returncode if proc.returncode is not None else -1,
    )


async def get_nmcli_connections(timeout: int) -> NmcliResult:
    """Tüm bağlantı profillerini getirir. Salt-okunur, sudo gerekmez."""
    return await _run_command(
        [_NMCLI_PATH, "-t", "-g", "NAME,TYPE", "connection", "show"],
        timeout=timeout,
    )


async def get_active_connections(timeout: int) -> NmcliResult:
    """Aktif bağlantıları getirir. Salt-okunur, sudo gerekmez."""
    return await _run_command(
        [_NMCLI_PATH, "-t", "-g", "NAME,TYPE,STATE", "connection", "show", "--active"],
        timeout=timeout,
    )


async def connection_up(name: str, timeout: int) -> NmcliResult:
    validate_connection_name(name)
    return await _run_command(
        ["sudo", "-n", _NMCLI_PATH, "connection", "up", name], timeout=timeout
    )


async def connection_down(name: str, timeout: int) -> NmcliResult:
    validate_connection_name(name)
    return await _run_command(
        ["sudo", "-n", _NMCLI_PATH, "connection", "down", name], timeout=timeout
    )


async def connection_delete(name: str, timeout: int) -> NmcliResult:
    validate_connection_name(name)
    return await _run_command(
        ["sudo", "-n", _NMCLI_PATH, "connection", "delete", name], timeout=timeout
    )


async def connection_modify(name: str, prop: str, value: str, timeout: int) -> NmcliResult:
    validate_connection_name(name)
    return await _run_command(
        ["sudo", "-n", _NMCLI_PATH, "connection", "modify", name, prop, value],
        timeout=timeout,
    )


async def connection_import_wireguard(file_path: str, timeout: int) -> NmcliResult:
    """file_path kullanıcı girdisi değildir; routers/vpn.py tarafından
    üretilen rastgele (uuid4) bir geçici dosya yoludur."""
    return await _run_command(
        ["sudo", "-n", _NMCLI_PATH, "connection", "import", "type", "wireguard", "file", file_path],
        timeout=timeout,
    )
```

> [!info] `sudo -n` ne işe yarar? `-n` (non-interactive) bayrağı, sudo'nun bir şifre sormaya çalışıp süresiz beklemesini engeller; yetki doğru tanımlanmadıysa komut hemen hata verir. Bu yetkinin nasıl güvenli şekilde (root vermeden, sadece `nmcli`'ye özel) tanımlanacağı bir sonraki notta.

---

## 3. Resepsiyon Katmanı (vpn.py)

Profil adları URL yerine JSON body üzerinden taşınır; aynı anda gelen birden fazla bağlan/kapat isteğinin çakışmasını önlemek için bir kilit (`asyncio.Lock`) kullanılır.

```bash
nano ~/pinet-api/app/routers/vpn.py
```

```python
"""VPN (WireGuard/nmcli) kontrol endpoint'leri."""

import asyncio
import uuid
from functools import lru_cache
from pathlib import Path

from fastapi import APIRouter, Depends, File, Form, HTTPException, UploadFile, status

from app.core import nmcli_worker
from app.core.config import Settings, get_settings
from app.core.logging_config import get_logger
from app.core.security import get_api_key
from app.schemas.vpn import (
    VpnActionResponse,
    VpnConnectRequest,
    VpnDeleteRequest,
    VpnProfileListResponse,
    VpnStatusResponse,
)

logger = get_logger(__name__)

router = APIRouter(
    prefix="/api/vpn",
    tags=["VPN Kontrol"],
    dependencies=[Depends(get_api_key)],
)


@lru_cache(maxsize=1)
def _get_lock() -> asyncio.Lock:
    return asyncio.Lock()


def _is_wireguard(name: str, conn_type: str) -> bool:
    return "wireguard" in conn_type.lower() or name.startswith("wg")


@router.get("/list", response_model=VpnProfileListResponse)
async def list_vpn_profiles(settings: Settings = Depends(get_settings)):
    """Sistemdeki tüm WireGuard/VPN profillerini listeler."""
    result = await nmcli_worker.get_nmcli_connections(settings.nmcli_timeout_seconds)
    if not result.success:
        logger.error("Profil listesi alınamadı: %s", result.stderr)
        raise HTTPException(status_code=502, detail="nmcli profil listesi alınamadı.")

    profiles = []
    for line in result.stdout.splitlines():
        if not line or ":" not in line:
            continue
        name, conn_type = line.split(":", 1)
        if _is_wireguard(name, conn_type):
            profiles.append(name)

    return VpnProfileListResponse(profiles=profiles)


@router.get("/durum", response_model=VpnStatusResponse)
async def get_vpn_status(settings: Settings = Depends(get_settings)):
    """Şu an aktif bir VPN tüneli var mı, varsa hangisi onu söyler."""
    result = await nmcli_worker.get_active_connections(settings.nmcli_timeout_seconds)
    if not result.success:
        logger.error("Aktif bağlantı durumu alınamadı: %s", result.stderr)
        raise HTTPException(status_code=502, detail="nmcli durum sorgusu başarısız.")

    for line in result.stdout.splitlines():
        if not line or line.count(":") < 2:
            continue
        name, conn_type, state = line.split(":", 2)
        if _is_wireguard(name, conn_type) and "activated" in state:
            return VpnStatusResponse(online=True, active_connection=name)

    return VpnStatusResponse(online=False, active_connection=None)


@router.post("/kapat", response_model=VpnActionResponse)
async def stop_vpn(settings: Settings = Depends(get_settings)):
    """Açık olan VPN bağlantısını keser (Acil Fren)."""
    async with _get_lock():
        return await _stop_vpn_unlocked(settings)


async def _stop_vpn_unlocked(settings: Settings) -> VpnActionResponse:
    """Kilit çağıranda alınmış varsayılarak çalışır (bkz. /kapat ve /baglan)."""
    current = await get_vpn_status(settings)
    if not current.online:
        return VpnActionResponse(success=True, message="Zaten açık bir VPN yok.")

    active_name = current.active_connection
    result = await nmcli_worker.connection_down(active_name, settings.nmcli_timeout_seconds)
    if not result.success:
        raise HTTPException(status_code=502, detail=f"Kapatma hatası: {result.stderr}")
    return VpnActionResponse(success=True, message=f"'{active_name}' tüneli kapatıldı.")


@router.post("/baglan", response_model=VpnActionResponse)
async def start_vpn(body: VpnConnectRequest, settings: Settings = Depends(get_settings)):
    """Belirtilen VPN profiline bağlanır (önce diğerlerini kapatarak)."""
    async with _get_lock():
        await _stop_vpn_unlocked(settings)

        result = await nmcli_worker.connection_up(
            body.profile_name, settings.nmcli_timeout_seconds
        )
        if not result.success:
            raise HTTPException(status_code=502, detail=f"Bağlantı hatası: {result.stderr}")
        return VpnActionResponse(
            success=True, message=f"'{body.profile_name}' tüneli aktif edildi."
        )


@router.post("/ekle", response_model=VpnActionResponse)
async def add_vpn(
    display_name: str = Form(..., min_length=1, max_length=64),
    file: UploadFile = File(...),
    settings: Settings = Depends(get_settings),
):
    """WireGuard .conf dosyasını yükler ve otomatik açılmamasını sağlar."""
    safe_display_name = "".join(
        c for c in display_name if c.isalnum() or c in "._-"
    ).strip()
    if not safe_display_name:
        raise HTTPException(status_code=400, detail="Geçerli bir görünür ad giriniz.")

    content = await file.read(settings.upload_max_size_bytes + 1)
    if len(content) > settings.upload_max_size_bytes:
        raise HTTPException(status_code=413, detail="Dosya çok büyük.")
    if not content:
        raise HTTPException(status_code=400, detail="Boş dosya.")

    # Geçici dosya adı rastgele (uuid4) üretilir; kullanıcı girdisinden
    # değil - bu, dosya adı üzerinden herhangi bir path traversal
    # ihtimalini tamamen ortadan kaldırır.
    temp_path = Path(f"/tmp/wg{uuid.uuid4().hex[:8]}.conf")

    try:
        temp_path.write_bytes(content)

        result = await nmcli_worker.connection_import_wireguard(
            str(temp_path), settings.nmcli_timeout_seconds
        )
        if not result.success:
            raise HTTPException(status_code=502, detail=f"Import hatası: {result.stderr}")

        try:
            imported_name = result.stdout.split("Connection '")[1].split("'")[0]
        except IndexError:
            imported_name = safe_display_name

        await nmcli_worker.connection_modify(
            imported_name, "connection.autoconnect", "no",
            settings.nmcli_timeout_seconds,
        )
        await nmcli_worker.connection_modify(
            imported_name, "connection.id", safe_display_name,
            settings.nmcli_timeout_seconds,
        )
        await nmcli_worker.connection_down(safe_display_name, settings.nmcli_timeout_seconds)

        return VpnActionResponse(
            success=True,
            message=f"'{safe_display_name}' sisteme eklendi (pasif durumda).",
        )
    finally:
        # Hata olsa da olmasa da geçici dosya diskte kalmamalı.
        temp_path.unlink(missing_ok=True)


@router.post("/sil", response_model=VpnActionResponse)
async def delete_vpn(body: VpnDeleteRequest, settings: Settings = Depends(get_settings)):
    """Belirtilen VPN profilini sistemden siler."""
    async with _get_lock():
        current = await get_vpn_status(settings)
        if current.online and current.active_connection == body.profile_name:
            await _stop_vpn_unlocked(settings)

        result = await nmcli_worker.connection_delete(
            body.profile_name, settings.nmcli_timeout_seconds
        )
        if not result.success:
            raise HTTPException(status_code=502, detail=f"Silme hatası: {result.stderr}")
        return VpnActionResponse(
            success=True, message=f"'{body.profile_name}' profili sistemden silindi."
        )
```


> [!tip] VPN modülü artık `main.py` içinde import edildiği tam haliyle çalışabilir durumda. Ancak servisi başlatmadan önce, `nmcli`'nin root gerektiren komutlarını (yukarıdaki `sudo -n` çağrıları) çalıştırabilmesi için doğru kullanıcı ve yetki yapılandırmasının kurulması gerekiyor. Bu adım ve systemd servisinin kurulumu sıradaki notta.

