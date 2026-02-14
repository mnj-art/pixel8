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
이것도 에러나서

```bash
tools/bazel build \
  --//private/google-modules/soc/gs:gs_kernel_build=//private/devices/google/shusky:zuma_shusky \
  --//private/google-modules/soc/gs:gs_soc_module=//private/devices/google/shusky:zuma_soc.shusky_internal \
  //private/devices/google/shusky:zuma_shusky_dist
```
이거로 해결이 되기를 기대.

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


Windows(호스트)에서 VMware의 Ubuntu(게스트)로 Pixel 8을 완벽하게 넘겨주기 위한 설정입니다.

특히 **`adb` (켜진 상태)**와 **`fastboot` (부트로더 상태)**는 USB 장치 ID가 서로 다릅니다. 따라서 폰이 재부팅될 때마다 VMware가 연결을 놓치지 않게 하는 것이 핵심입니다.

---
vmware의 우분투로 usb 을 넘겨줄거예요

### 1단계: VMware USB 컨트롤러 설정 (가상머신 꺼진 상태)

먼저 가상머신이 USB 3.1을 제대로 지원하도록 설정해야 합니다.

1. VMware에서 Ubuntu 가상머신을 **Power Off** 합니다.
2. **Edit virtual machine settings**를 클릭합니다.
3. **USB Controller** 항목을 선택합니다.
4. **USB compatibility**를 **USB 3.1**로 변경합니다.
5. **"Show all USB input devices"** 체크박스를 켭니다.
6. **OK**를 누르고 부팅합니다.

### 2단계: Ubuntu 내부 설정 (udev rules)

이 설정을 안 하면 `no permissions` 에러가 뜨거나 `sudo`를 매번 붙여야 합니다.

터미널을 열고 아래 명령어를 **한 번에 복사해서 붙여넣으세요.** (Google USB 장치 권한 허용 설정)

```bash
# 1. 안드로이드 udev 규칙 생성
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", MODE="0666", GROUP="plugdev"' | sudo tee /etc/udev/rules.d/51-android.rules

# 2. 권한 재로딩
sudo udevadm control --reload-rules
sudo service udev restart

# 3. 현재 사용자를 plugdev 그룹에 추가 (필요시)
sudo usermod -aG plugdev $USER

```

### 3단계: USB 연결 및 확인 (가장 중요)

이제 폰을 PC에 연결합니다. 이때 VMware 화면 하단이나 상단 메뉴에서 장치를 연결해야 합니다.

1. **연결 방법:**
* **메뉴바:** `Player` > `Manage` > `Removable Devices` > `Google Pixel 8` > **Connect (Disconnect from Host)**
* 또는 VMware 창 오른쪽 하단의 **USB 아이콘(아이팟 모양)** 우클릭 > **Connect**.


2. **연결 확인 (ADB):**
* 폰에서 'USB 디버깅'을 켭니다.
* Ubuntu 터미널에서:
```bash
adb devices

```


* 폰 화면에 "이 컴퓨터를 허용하시겠습니까?" 팝업이 뜨면 **허용**을 누릅니다.
* `device`라고 뜨면 성공!



### 4단계: Fastboot 모드에서의 연결 (함정 주의 ⚠️)

커널 연구를 하려면 `adb reboot bootloader`를 자주 씁니다. 이때 폰이 재부팅되면서 **연결이 끊어집니다.**

1. `adb reboot bootloader` 명령을 치면 폰이 꺼집니다.
2. 그 순간 Windows가 "띠링~" 하고 새 장치(부트로더 모드의 Pixel)를 인식해버릴 수 있습니다.
3. **해결법:**
* 폰이 재부팅되는 순간, **VMware 창을 마우스로 클릭해서 활성화(Focus)** 상태로 두세요.
* VMware가 "New USB Device Detected"라고 물어보면 **"Connect to a virtual machine"**을 선택하고 **"Remember my choice"**를 체크하세요.


4. **확인:**
```bash
fastboot devices

```


여기서 시리얼 넘버가 나오면 성공입니다.

**팁:** 만약 `fastboot` 상태에서 자꾸 Windows가 폰을 뺏어간다면, 폰을 부트로더 모드로 둔 상태에서 **3단계의 메뉴**를 통해 다시 한번 수동으로 연결해 주면, 그 뒤로는 VMware가 기억합니다.

일단 언락을 해야함 : 디벨러퍼 메뉴에서 oem unlock 을 키고

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


```
~ ❯ fastboot flashing unlock                                                                                            05:26:49 PM
                                                   OKAY [  0.047s]
Finished. Total time: 0.047s
~ ❯ cd pixel8-kernel                                                                                                    05:27:46 PM
~/pixel8-kernel ❯ fastboot flash boot out/dist/boot.img                                                                 05:28:49 PM
Sending 'boot_b' (52224 KB)                        OKAY [  2.292s]
Writing 'boot_b'                                   OKAY [  0.105s]
Finished. Total time: 2.463s
~/pixel8-kernel ❯ fastboot flash vendor_kernel_boot out/dist/vendor_kernel_boot.img                                     05:29:06 PM
Sending 'vendor_kernel_boot_b' (8530 KB)           OKAY [  0.387s]
Writing 'vendor_kernel_boot_b'                     OKAY [  0.054s]
Finished. Total time: 0.461s
~/pixel8-kernel ❯ fastboot flash dtbo out/dist/dtbo.img                                                                 05:29:29 PM
Sending 'dtbo_b' (2046 KB)                         OKAY [  0.094s]
Writing 'dtbo_b'                                   OKAY [  0.047s]
Finished. Total time: 0.146s
~/pixel8-kernel ❯ fastboot flash vendor_dlkm out/dist/vendor_dlkm.img                                                   05:29:40 PM
Sending 'vendor_dlkm' (58932 KB)                   OKAY [  2.675s]
Writing 'vendor_dlkm'                              FAILED (remote: 'partition (vendor_dlkm) not found')
fastboot: error: Command failed
~/pixel8-kernel ❯ fastboot reboot fastboot                                                                              05:30:03 PM
Rebooting into fastboot                            OKAY [  0.001s]
< waiting for any device >
Finished. Total time: 35.804s
~/pixel8-kernel ❯ fastboot flash vendor_dlkm out/dist/vendor_dlkm.img                                               36s 05:32:12 PM
Resizing 'vendor_dlkm_b'                           OKAY [  0.020s]
Sending 'vendor_dlkm_b' (58932 KB)                 OKAY [  2.810s]
Writing 'vendor_dlkm_b'                            OKAY [  0.244s]
Finished. Total time: 3.117s
~/pixel8-kernel ❯ fastboot reboot                                                                                    3s 05:32:26 PM
Rebooting                                          OKAY [  0.002s]
Finished. Total time: 0.052s
~/pixel8-kernel ❯
```

vendor_dlkm 이 super 이미지에 있기 때문에, fastbootd 로 진입해서 flashing 해야함

~/Downloads ❯ unzip shiba-ud1a.230803.041-factory-3ed11735.zip                                                      12s 05:48:26 PM
Archive:  shiba-ud1a.230803.041-factory-3ed11735.zip
   creating: shiba-ud1a.230803.041/
  inflating: shiba-ud1a.230803.041/flash-base.sh  
 extracting: shiba-ud1a.230803.041/image-shiba-ud1a.230803.041.zip  
  inflating: shiba-ud1a.230803.041/flash-all.sh  
  inflating: shiba-ud1a.230803.041/radio-shiba-g5300i-230829-230906-b-10765749.img  
  inflating: shiba-ud1a.230803.041/bootloader-shiba-ripcurrent-14.0-10807316.img  
  inflating: shiba-ud1a.230803.041/flash-all.bat  
~/Downloads ❯ ls                                                                                                    15s 05:48:45 PM
shiba-ud1a.230803.041  shiba-ud1a.230803.041-factory-3ed11735.zip
~/Downloads ❯ cd shiba-ud1a.230803.041                                                                                  05:48:47 PM
~/Downloads/shiba-ud1a.230803.041 ❯ ls                                                                                  05:48:49 PM
bootloader-shiba-ripcurrent-14.0-10807316.img  flash-all.sh   image-shiba-ud1a.230803.041.zip
flash-all.bat                                  flash-base.sh  radio-shiba-g5300i-230829-230906-b-10765749.img
~/Downloads/shiba-ud1a.230803.041 ❯ unzip image-shiba-ud1a.230803.041.zip                                               05:48:50 PM
Archive:  image-shiba-ud1a.230803.041.zip
  inflating: android-info.txt        
  inflating: boot.img                
  inflating: init_boot.img           
  inflating: vendor_boot.img         
  inflating: vendor_kernel_boot.img  
  inflating: system.img              
  inflating: vendor.img              
  inflating: product.img             
  inflating: system_ext.img          
  inflating: vendor_dlkm.img         
  inflating: system_dlkm.img         
  inflating: system_other.img        
  inflating: dtbo.img                
  inflating: pvmfw.img               
  inflating: vbmeta_system.img       
  inflating: vbmeta_vendor.img       
  inflating: vbmeta.img              
  inflating: super_empty.img         
~/Downloads/shiba-ud1a.230803.041 ❯ ls                                                                              36s 05:49:39 PM
android-info.txt                               init_boot.img                                    system_other.img
boot.img                                       product.img                                      vbmeta.img
bootloader-shiba-ripcurrent-14.0-10807316.img  pvmfw.img                                        vbmeta_system.img
dtbo.img                                       radio-shiba-g5300i-230829-230906-b-10765749.img  vbmeta_vendor.img
flash-all.bat                                  super_empty.img                                  vendor_boot.img
flash-all.sh                                   system_dlkm.img                                  vendor_dlkm.img
flash-base.sh                                  system_ext.img                                   vendor.img
image-shiba-ud1a.230803.041.zip                system.img                                       vendor_kernel_boot.img
~/Downloads/shiba-ud1a.230803.041 ❯ fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img            05:49:44 PM
Sending 'vbmeta' (12 KB)                           OKAY [  0.003s]
Writing 'vbmeta'                                   FAILED (remote: 'No such file or directory')
fastboot: error: Command failed
~/Downloads/shiba-ud1a.230803.041 ❯ fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img            05:49:54 PM
Sending 'vbmeta_b' (12 KB)                         OKAY [  0.002s]
Writing 'vbmeta_b'                                 OKAY [  0.010s]
Finished. Total time: 0.016s
~/Downloads/shiba-ud1a.230803.041 ❯             


vmbeta 를 바꾸니 부팅이 된다..
