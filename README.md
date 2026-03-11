# esp-32-firmware

## OTA release rehberi

Bu repo OTA dagitimi icin iki dosya tutar:

- `ota/esp32/prod/firmware.bin`
- `ota/esp32/prod/metadata.json`

Cihaz once `metadata.json` dosyasini ceker, sonra `url` alanindaki `firmware.bin` dosyasini indirir.

## Kritik kural

`metadata.json` icindeki `sha256` alani, `Get-FileHash` ile alinan dosya hash'i degildir.  
Bu alan firmware image'in **Validation Hash** degeridir.

## Adim adim release

### 1) Firmware'i build et

`D:\esp_idf\wifi_demo` klasorunde:

```powershell
idf.py build
```

### 2) Yeni bin dosyasini OTA repo'ya kopyala

```powershell
Copy-Item D:\esp_idf\wifi_demo\build\wifi_demo.bin D:\esp-32-firmware\ota\esp32\prod\firmware.bin -Force
```

### 3) Validation Hash hesapla

PowerShell'de asagidaki komut, gerekli yollari otomatik bulup `image_info` calistirir:

```powershell
$idfPath = $env:IDF_PATH
if ([string]::IsNullOrWhiteSpace($idfPath)) {
  $idfPyScript = Get-ChildItem "$env:USERPROFILE\esp" -Recurse -Filter idf.py -ErrorAction SilentlyContinue | Select-Object -First 1
  if ($idfPyScript) { $idfPath = Split-Path (Split-Path $idfPyScript.FullName -Parent) -Parent }
}
$idfPy = Get-ChildItem "$env:USERPROFILE\.espressif\python_env" -Directory -ErrorAction SilentlyContinue |
  Sort-Object LastWriteTime -Descending |
  ForEach-Object { Join-Path $_.FullName "Scripts\python.exe" } |
  Where-Object { Test-Path $_ } |
  Select-Object -First 1
$esptool = Join-Path $idfPath "components\esptool_py\esptool\esptool.py"
& $idfPy $esptool image_info .\firmware.bin
```

Bu satirdaki degeri al:

```text
Validation Hash: <BURADAKI_DEGER>
```

Tek satirda sadece hash almak istersen:

```powershell
( & $idfPy $esptool image_info .\firmware.bin | Select-String "Validation Hash").ToString().Split(":")[1].Split("(")[0].Trim().ToLower()
```

### 4) metadata.json guncelle

`ota/esp32/prod/metadata.json`:

```json
{
  "version": "1.0.X",
  "url": "https://uzunberkay.github.io/esp-32-firmware/ota/esp32/prod/firmware.bin",
  "sha256": "<Validation Hash buraya>"
}
```

Notlar:

- `version` degeri, firmware icindeki app version ile ayni olmali.
- URL HTTPS olmali.
- `sha256` hex degeri buyuk/kucuk harf fark etmez, ama tek format kullanmak icin kucuk harf onerilir.

### 5) GitHub'a yayinla

```powershell
git add ota/esp32/prod/firmware.bin ota/esp32/prod/metadata.json
git commit -m "Release OTA 1.0.X"
git push
```

### 6) Canli linkleri kontrol et

Tarayicida ac:

- `https://uzunberkay.github.io/esp-32-firmware/ota/esp32/prod/metadata.json`
- `https://uzunberkay.github.io/esp-32-firmware/ota/esp32/prod/firmware.bin`

Ikisi de `200 OK` donmeli.
