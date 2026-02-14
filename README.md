# pixel8

Pixel 8 (코드명: `shiba`)

Pixel 8은 Google Tensor G3 (`zuma`) SoC

**Bazel (Kleaf)** 도입

### 1. 사전 준비 (Host Environment)

Ubuntu 22.04 LTS

* **필수 패키지 설치:**
```bash
sudo apt update && sudo apt install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc fontconfig

```


* **Repo 도구 설치:**
```bash
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=~/bin:$PATH

```



### 2. 부트로더 언락 (Bootloader Unlock)

커널이나 커스텀 이미지를 올리기 위해 필수입니다. **데이터가 초기화되므로 백업 필수입니다.**

1. **개발자 옵션 활성화:** 설정 > 휴대전화 정보 > 빌드 번호 7번 터치.
2. **OEM 잠금 해제:** 설정 > 시스템 > 개발자 옵션 > **OEM 잠금 해제** 허용.
3. **부트로더 진입:**
```bash
adb reboot bootloader

```


4. **언락 실행:**
```bash
fastboot flashing unlock

```


(볼륨 키로 이동 후 전원 키로 확인)

### 3. Pixel 8 커널 소스 및 빌드 (Kleaf/Bazel)

Pixel 8부터는 커널 빌드 시스템이 Bazel 기반의 **Kleaf**로 완전히 전환되었습니다.

#### A. 소스 코드 다운로드 (Repo init & sync)

Pixel 8의 커널은 `android-gs-shiba-5.15-android14` (또는 최신 Android 버전에 맞춰 `android15-6.1` 등) 브랜치를 사용합니다.

* **작업 디렉토리 생성:**
```bash
mkdir pixel8-kernel && cd pixel8-kernel

```

사용자님의 빌드번호 **UD1A.230803**은 Pixel 8의 초기 출시 버전 (Day 1) 펌웨어입니다.

이 버전에 해당하는 커널 소스 브랜치는 뒤에 -d1 (Day 1) 접미사가 붙어 있습니다. 아까 실패했던 명령어에 -d1을 붙이면 해결됩니다.

🚀 해결된 명령어 (초기 버전용)
Bash
repo init -u https://android.googlesource.com/kernel/manifest -b android-gs-shusky-5.15-android14-d1
(성공 후 repo sync -c -j$(nproc) 잊지 마세요!)

💡 연구자(kernel-lab)를 위한 조언
현재 사용 중인 UD1A 버전은 출시 초기 상태라 버그가 많고, 최신 보안 패치가 적용되지 않은 상태입니다.

N-day 취약점 연구 목적이라면:
그대로 진행하셔도 좋습니다. 초기 버전의 취약점을 분석하기에 딱 좋은 환경입니다.

최신 익스플로잇/기능 연구 목적이라면:
폰을 업데이트하고 커널 소스도 QPR3 (최신) 브랜치로 맞추는 것이 정신 건강에 좋습니다. 초기 버전은 빌드 시스템(Bazel) 설정이 지금과 미묘하게 달라서, 최신 가이드와 안 맞을 때가 있습니다.

최신 버전 명령어 (참고용):

Bash
repo init -u https://android.googlesource.com/kernel/manifest -b android-gs-shusky-5.15-android14-qpr3
일단은 -d1 붙인 명령어로 진행


* **Repo 초기화 (Android 14 기준 예시):**
```bash
repo init -u https://android.googlesource.com/kernel/manifest -b android-gs-shiba-5.15-android14

```


*(팁: 최신 펌웨어를 사용 중이라면 설정 > 휴대전화 정보 > Android 버전 > 커널 버전에서 정확한 커널 버전을 확인 후 브랜치를 선택하세요.)*
* **동기화:**
```bash
repo sync -c -j$(nproc)

```

1시간 걸렸네.. ㅜㅜ



#### B. 커널 빌드 (Bazel 사용)

이전의 `make` 명령어가 아닌 `tools/bazel`을 사용합니다.

* **빌드 명령어:**
```bash
tools/bazel build //private/google-modules/soc/gs:shiba_dist

```

명령어 찾기가 어려웠네..
```bash
tools/bazel build \
  --//private/google-modules/soc/gs:gs_kernel_build=//private/devices/google/shusky:zuma_shusky \
  //private/devices/google/shusky:zuma_shusky_dist
```
  

* `shiba`: Pixel 8
* `husky`: Pixel 8 Pro


* **결과물 위치:** `out/shiba/dist/` (Image, init_boot.img, vendor_boot.img 등이 생성됨)

### 4. 커널 익스플로잇 연구를 위한 설정 (중요)

사용자님의 `kernel-lab` 프로젝트와 연계하여, 디버깅 기능을 활성화하려면 Bazel 빌드 설정을 수정하거나 플래그를 사용해야 합니다.

* **KASAN (Kernel Address Sanitizer) 활성화:**
메모리 오염 취약점을 찾기 위해 필수적입니다.
```bash
tools/bazel build --config=kasan //private/google-modules/soc/gs:shiba_dist

```


* **GKI (Generic Kernel Image) 참고:**
Pixel 8은 GKI를 따르므로, 커널 코어(`vmlinux`)와 모듈(`*.ko`)이 분리되어 관리됩니다. 모듈 로딩 문제를 피하기 위해 **vendor_boot** 파티션도 함께 업데이트하는 것이 안전합니다.

### 5. 플래싱 (Flashing)

빌드된 이미지를 기기에 적용합니다.

1. **부트로더 진입:** `adb reboot bootloader`
2. **이미지 플래싱:**
```bash
fastboot flash boot out/shiba/dist/boot.img
fastboot flash vendor_boot out/shiba/dist/vendor_boot.img
fastboot flash dtbo out/shiba/dist/dtbo.img

```


3. **재부팅:** `fastboot reboot`

