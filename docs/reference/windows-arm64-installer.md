# Windows ARM64 설치 파일 만들기

이 문서는 Windows 10/11 ARM64 PC에서 Orca NSIS 설치 파일(`orca-windows-setup.exe`)을 만들고, SafeNet USB 토큰으로 서명한 뒤 GitHub Release에 올리는 절차다. 공식 저장소의 Windows 설치 파일은 x64만 있다. 여기서 만드는 파일은 소스 트리의 패키지 버전을 그대로 쓴다.

설치 프로그램 껍데기는 x64용과 같이 32비트 NSIS 로더다. 설치되면 `Orca.exe`는 ARM64다.

## 준비

- Windows ARM64, Node.js 24 ARM64, Git, pnpm 12 (`corepack prepare pnpm@12.0.0 --activate`)
- Visual Studio 2022 (네이티브 모듈 컴파일)
- 7-Zip
- Windows SDK의 `signtool.exe`
- SafeNet Authentication Client. 토큰만 꽂혀 있으면 코드 서명 인증서가 보이지 않는다.
- 현재 사용자 인증서 저장소에 EV 코드 서명 인증서가 보여야 한다. 이 PC에서는 주체가 `Nanum Space Co,. Ltd`이고, 지문은 `3CE49DE1124F325082FA90BDE4944756D1626251`이다. 토큰이 바뀌면 `certutil -silent -scinfo`와 `Get-ChildItem Cert:\CurrentUser\My`로 다시 확인한다.

저장소를 받는다.

```powershell
git clone --depth 1 https://github.com/stablyai/orca.git
cd orca
pnpm install
```

데스크톱 `pnpm install`은 `mobile/`을 설치하지 않는다. 설치 파일은 모바일 웹 번들을 요구하므로 따로 설치한다.

```powershell
pnpm --dir mobile install
```

`sherpa-onnx`의 Windows 패키지는 x64만 있다. ARM64 호스트의 `pnpm install`은 이 패키지를 빼므로, 패키징 전에 받아 둔다.

```powershell
npm pack sherpa-onnx-win-x64@1.12.37
New-Item -ItemType Directory -Force node_modules\sherpa-onnx-win-x64 | Out-Null
tar -xf .\sherpa-onnx-win-x64-1.12.37.tgz -C node_modules\sherpa-onnx-win-x64 --strip-components=1
```

`agent-browser`도 Windows ARM64 바이너리가 없다. 설치 파일에는 `agent-browser-win32-x64.exe`가 들어간다.

## 앱 빌드

`pnpm run build:release`는 `verify:computer-native`에서 `python3`를 호출한다. Windows의 `python3.exe`가 Microsoft Store 바로 가기이면 그 단계가 실패한다. 스토어 바로 가기보다 앞에 실제 인터프리터를 둔다.

```powershell
$shim = "$env:TEMP\orca-python3-shim"
New-Item -ItemType Directory -Force -Path $shim | Out-Null
Copy-Item C:\Python313-arm64\python.exe "$shim\python3.exe"
$env:Path = "$shim;" + (($env:Path -split ';' | Where-Object { $_ -notmatch 'WindowsApps' }) -join ';')
```

그다음 설치 파일에 넣을 산출물을 만든다. `build:win` 전체를 쓰지 않아도 된다. 아래 순서로 데스크톱 산출물과 모바일 웹 번들이 있으면 된다.

```powershell
pnpm run build:relay
pnpm run build:native
pnpm run build:cli
pnpm run build:electron-vite
pnpm run build:web-from-renderer
pnpm run build:mobile-web
```

`build:native`가 만드는 `native\windows-cli-launcher\.build\orca.exe`는 .NET Framework 컴파일 결과라 ARM64 네이티브가 아니다. 패키징 스크립트가 그 파일을 재사용한다.

## ARM64에서 7-Zip 필터를 끄기

electron-builder는 이 PC에서 ARM64용 7-Zip으로 `app-arm64.7z`를 만든다. 그 압축은 `ARM64 LZMA2 BCJ2` 필터를 쓴다. NSIS가 푸는 쪽은 32비트 `nsis7z`라서 이 필터를 풀지 못한다. 데이터 파일만 설치되고 `Orca.exe`와 DLL은 빠진다. 바로 가기는 생기고, 실행하면 "Orca.exe 파일을 찾는 중" 창이 뜬다.

`nsis.useZip = true`로 바꾸면 파일 이름만 `app-arm64.zip`이 된다. 내용은 여전히 7z다. NSIS의 ZIP 해제기는 "Error opening ZIP file"로 멈춘다. ZIP 옵션은 쓰지 않는다.

7z를 유지하고 실행 파일 필터만 끈다. electron-builder 26.15.3은 `ELECTRON_BUILDER_7Z_FILTER`를 받지만 허용 목록에 `OFF`가 없다. 패키징에 쓰이는 파일에 `OFF`를 추가한다.

`node_modules/.pnpm/app-builder-lib@26.15.3_dmg_30c09b2f8bed2efe829941a1bdee0a00/node_modules/app-builder-lib/out/targets/archive.js`

```javascript
const ALLOWED_7Z_FILTERS = new Set([
  "BCJ", "BCJ2", "ARM", "ARMT", "IA64", "PPC", "SPARC", "DELTA", "OFF"
]);
```

이 수정은 `node_modules` 안이라 `pnpm install`을 다시 하면 사라진다.

## 설치 파일 생성

```powershell
$env:ELECTRON_BUILDER_7Z_FILTER = "off"
pnpm exec electron-builder --config config/electron-builder.config.cjs --win --arm64
```

산출물은 `dist\orca-windows-setup.exe`와 `dist\win-arm64-unpacked\`다.

만든 7z가 필터 없이 압축됐는지 확인한다. `Method`가 `LZMA2`여야 하고 `ARM64`가 있으면 안 된다.

```powershell
7z x .\dist\orca-windows-setup.exe -o$env:TEMP\orca-nsis-check -y
7z l -slt "$env:TEMP\orca-nsis-check\`$PLUGINSDIR\app-arm64.7z" | Select-String "^Method"
```

자동 설치로 `Orca.exe`가 실제로 복사되는지도 확인한다.

```powershell
.\dist\orca-windows-setup.exe /S
Get-Item "$env:LOCALAPPDATA\Programs\orca\Orca.exe"
```

## 서명

`config/scripts/windows-uninstaller-signing.cjs`의 서명 훅은 `signtool`을 대신한다. SignPath용이라, USB 인증서가 있어도 electron-builder 로그의 "signing with signtool.exe"는 파일을 서명하지 않는다.

순서는 두 단계다.

1. 패키징된 `dist\win-arm64-unpacked\Orca.exe`를 서명한다.
2. 서명된 폴더로 NSIS만 다시 만들고, 그 설치 파일을 서명한다. 먼저 설치 파일을 만들면 그 안에 서명 전 `Orca.exe`가 들어간다.

```powershell
$sig = "C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0\arm64\signtool.exe"
$thumb = "3CE49DE1124F325082FA90BDE4944756D1626251"
$tr = "http://timestamp.globalsign.com/tsa/r6advanced1"

& $sig sign /sha1 $thumb /fd SHA256 /td SHA256 /tr $tr .\dist\win-arm64-unpacked\Orca.exe

$env:ELECTRON_BUILDER_7Z_FILTER = "off"
pnpm exec electron-builder --config config/electron-builder.config.cjs --prepackaged dist\win-arm64-unpacked --win --arm64

& $sig sign /sha1 $thumb /fd SHA256 /td SHA256 /tr $tr .\dist\orca-windows-setup.exe
& $sig verify /pa .\dist\orca-windows-setup.exe
```

토큰 PIN 창이 뜨면 입력한다. 지문은 인증서가 바뀔 때마다 다시 읽는다.

제거 프로그램은 같은 훅 때문에 설치 파일에 서명 없이 들어갈 수 있다.

## GitHub Release

포크에 올릴 때는 자산 이름을 ARM64임이 보이게 한다. 공식 x64 파일명 `orca-windows-setup.exe`와 겹치지 않게 `orca-windows-arm64-setup.exe`를 쓴다.

```powershell
Copy-Item .\dist\orca-windows-setup.exe $env:TEMP\orca-windows-arm64-setup.exe
gh release create v1.4.214-windows-arm64 `
  --repo kimmingul/orca `
  --target <빌드한 커밋 SHA> `
  --title "v1.4.214 Windows ARM64" `
  --notes-file release-notes.md `
  $env:TEMP\orca-windows-arm64-setup.exe
```

같은 태그의 파일만 바꿀 때는 `gh release upload <tag> <file> --repo kimmingul/orca --clobber`를 쓴다.

## 확인 목록

- 설치 후 `C:\Users\kimmi\AppData\Local\Programs\orca\Orca.exe`가 있고 크기가 `dist\win-arm64-unpacked\Orca.exe`와 같다.
- 같은 폴더에 `ffmpeg.dll`, `d3dcompiler_47.dll`, `libEGL.dll`, `vulkan-1.dll`이 있다.
- `signtool verify /pa`가 설치 파일과 `Orca.exe` 둘 다 성공한다.
- 바탕화면 바로 가기의 대상이 그 `Orca.exe`다.
