# esp-32-firmware

## OTA Release Rehberi

Bu repo, ESP32 cihazlari icin OTA dagitimi amaciyla iki temel dosya barindirir:

- `ota/esp32/prod/firmware.bin`
- `ota/esp32/prod/metadata.json`

Cihaz OTA surecinde once `metadata.json` dosyasini indirir, ardindan bu dosya icindeki `url` alanindan `firmware.bin` dosyasini ceker.

---

## Dosya Yapisi

```text
ota/
└── esp32/
    └── prod/
        ├── firmware.bin
        └── metadata.json
```

---

## Kritik Kural

`metadata.json` icindeki `sha256` alani, `Get-FileHash` ile hesaplanan dosya hash'i degildir.

Bu alana yazilmasi gereken deger, firmware image'in:

- **Validation Hash**

degeridir.

Yanlis hash kullanilirsa cihaz yeni firmware'i indirsa bile dogrulama basarisiz olabilir.

---

## Release Adimlari

### 1) Firmware'i build et

`D:\esp_idf\wifi_demo` klasorunde:

```powershell
idf.py build
```

Build sonunda ornek output dosyasi:

```text
D:\esp_idf\wifi_demo\build\wifi_demo.bin
```

---

### 2) Yeni firmware dosyasini OTA repo'ya kopyala

```powershell
Copy-Item D:\esp_idf\wifi_demo\build\wifi_demo.bin D:\esp-32-firmware\ota\esp32\prod\firmware.bin -Force
```

---

### 3) Validation Hash hesapla

PowerShell'de asagidaki komut, gerekli Python ve `esptool.py` yolunu bulup `image_info` calistirir:

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

Komut ciktilarinda su satiri bul:

```text
Validation Hash: <BURADAKI_DEGER>
```

`metadata.json` icindeki `sha256` alanina yazman gereken deger budur.

#### Sadece hash degerini tek satirda almak icin

```powershell
( & $idfPy $esptool image_info .\firmware.bin | Select-String "Validation Hash").ToString().Split(":")[1].Split("(")[0].Trim().ToLower()
```

---

### 4) metadata.json dosyasini guncelle

`ota/esp32/prod/metadata.json` dosyasini su sekilde duzenle:

```json
{
  "version": "1.0.X",
  "url": "https://uzunberkay.github.io/esp-32-firmware/ota/esp32/prod/firmware.bin",
  "sha256": "<Validation Hash buraya>"
}
```

#### Dikkat edilmesi gerekenler

- `version` degeri, firmware icindeki app version ile ayni olmali.
- `url` alani dogrudan indirilebilir ve `HTTPS` olmali.
- `sha256` alani Validation Hash olmali.
- Hex karakterlerde buyuk/kucuk harf teknik olarak fark etmese de, tutarlilik icin kucuk harf kullanilmasi onerilir.

---

### 5) GitHub'a push et

```powershell
git add ota/esp32/prod/firmware.bin ota/esp32/prod/metadata.json
git commit -m "Release OTA 1.0.X"
git push
```

---

### 6) Canli linkleri kontrol et

Push sonrasi tarayicida su iki adresi kontrol et:

- `https://uzunberkay.github.io/esp-32-firmware/ota/esp32/prod/metadata.json`
- `https://uzunberkay.github.io/esp-32-firmware/ota/esp32/prod/firmware.bin`

Ikisi de erisilebilir olmali.

Beklenen durum:

- `metadata.json` acilmali
- `firmware.bin` indirilebilmeli
- her iki URL de `404` vermemeli

---

## Hizli Kontrol Listesi

Release oncesi su maddeleri kontrol et:

- Firmware basarili build edildi mi?
- `firmware.bin` dogru klasore kopyalandi mi?
- Validation Hash dogru alindi mi?
- `metadata.json` icindeki `version` guncellendi mi?
- `metadata.json` icindeki `url` dogru mu?
- `metadata.json` icindeki `sha256` Validation Hash ile ayni mi?
- Dosyalar commit edilip pushlandi mi?
- GitHub Pages linkleri aciliyor mu?

---

## Ornek metadata.json

```json
{
  "version": "1.0.3",
  "url": "https://uzunberkay.github.io/esp-32-firmware/ota/esp32/prod/firmware.bin",
  "sha256": "ornekvalidationhashburaya"
}
```

---

## Ozet

OTA akisi su sekildedir:

1. Cihaz `metadata.json` dosyasini indirir
2. `version` kontrol edilir
3. Yeni surum varsa `url` alanindaki firmware indirilir
4. Firmware Validation Hash ile dogrulanir
5. Dogrulama basariliysa OTA uygulanir



### Test

$idfPath = $env:IDF_PATH

if ([string]::IsNullOrWhiteSpace($idfPath)) {
  $idfSearchRoots = @(
    "$env:USERPROFILE\esp",
    "C:\esp"
  )

  foreach ($root in $idfSearchRoots) {
    if (Test-Path $root) {
      $idfPyScript = Get-ChildItem $root -Recurse -Filter idf.py -ErrorAction SilentlyContinue |
        Where-Object { $_.FullName -like "*\tools\idf.py" } |
        Select-Object -First 1

      if ($idfPyScript) {
        $idfPath = Split-Path (Split-Path $idfPyScript.FullName -Parent) -Parent
        break
      }
    }
  }
}

$idfPySearchRoots = @(
  "$env:USERPROFILE\.espressif\python_env",
  "C:\Espressif\tools\python"
)

$idfPy = $null
foreach ($root in $idfPySearchRoots) {
  if (Test-Path $root) {
    $idfPy = Get-ChildItem $root -Recurse -Filter python.exe -ErrorAction SilentlyContinue |
      Where-Object { $_.FullName -like "*\Scripts\python.exe" } |
      Sort-Object LastWriteTime -Descending |
      Select-Object -ExpandProperty FullName -First 1

    if ($idfPy) { break }
  }
}

if ([string]::IsNullOrWhiteSpace($idfPath)) {
  throw "ESP-IDF bulunamadi. IDF_PATH ayarli degil ve idf.py tespit edilemedi."
}

if ([string]::IsNullOrWhiteSpace($idfPy)) {
  throw "ESP-IDF Python ortami bulunamadi."
}

& $idfPy -m esptool image-info .\firmware.bin
