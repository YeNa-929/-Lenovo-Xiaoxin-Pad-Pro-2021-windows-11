# Lenovo Xiaoxin Pad Pro 2021에서 Renegade UEFI 사용하기

대상 모델은 **TB-J716F**(Snapdragon 870, SM8250-AC)이며 빌드 타깃은 `j716f`다. Renegade Project가 2022년에 이 모델을 추가했다. 원래 포팅 PR에 기기 사진이 있으며, 이 가이드는 업스트림 커밋 `e1952621f419f8db60ed28271264e1b5184c571d`을 기준으로 한다. [기기 지원 PR](https://github.com/edk2-porting/edk2-msm/pull/161) · [기기별 소스](https://github.com/edk2-porting/edk2-msm/tree/e1952621f419f8db60ed28271264e1b5184c571d/Platform/Lenovo/sm8250)

## 빌드된 이미지 받기

[시험용 UEFI 릴리스](https://github.com/YeNa-929/-Lenovo-Xiaoxin-Pad-Pro-2021-windows-11/releases/tag/j716f-uefi-preview-1)에서 `boot-j716f.img`와 `SHA256SUMS`를 같은 폴더에 내려받는다. Linux에서는 `sha256sum -c SHA256SUMS`, Windows PowerShell에서는 `Get-FileHash .\boot-j716f.img -Algorithm SHA256`로 확인할 수 있다.

Ubuntu 24.04에서 빌드한 이미지는 6,725,632바이트다. Android boot header v1을 읽고 압축을 해제해 BootShim, 7 MiB UEFI 펌웨어, TB-J716F 전용 DTB를 원본과 바이트 단위로 대조했다. 이 빌드의 실기기 부팅과 Windows 11 구동은 미검증이다.

## 어디까지 가능한가

| 항목 | 확인 범위 |
| --- | --- |
| UEFI | TB-J716F 전용 메모리 맵, 2560×1600 화면 설정, ACPI DSDT, 부팅용 DTB가 있다. |
| Windows PE | 업스트림에는 SM8250 계열의 최소 WinPE 부팅 보고가 있다. 이 조합으로 TB-J716F에서 WinPE를 직접 부팅한 결과는 확인하지 못했다. |
| Windows 11 ARM64 | 이 기기용 UFS 저장장치, GPU, 터치, 무선랜, 오디오, 전원 관리까지 검증된 드라이버 세트를 찾지 못했다. 현재 첫 시험의 목표는 UEFI 메뉴 진입이다. |

[SM8250 WinPE에 대한 업스트림 설명](https://github.com/edk2-porting/edk2-msm/issues/300#issuecomment-1605325322)

## Ubuntu 24.04 x86-64에서 이미지 만들기

빈 작업 디렉터리에서 다음 명령을 실행한다. 첫 빌드는 EDK II와 여러 서브모듈을 내려받으므로 인터넷 연결이 필요하다.

```bash
sudo apt update
sudo apt install -y build-essential git ca-certificates uuid-dev acpica-tools nasm gettext gcc-aarch64-linux-gnu clang llvm lld device-tree-compiler python3 python-is-python3 python3-pil python3-git abootimg cpio libssl-dev
git clone https://github.com/YeNa-929/-Lenovo-Xiaoxin-Pad-Pro-2021-windows-11.git tb-j716f
cd tb-j716f
git submodule update --init edk2-msm
cd edk2-msm
./build.sh --device j716f
sha256sum boot-j716f.img
```

기기 설정은 `configs/devices/j716f.conf`에 있다. 빌드는 펌웨어와 BootShim을 압축하고 기기 전용 DTB를 붙인 다음 Android boot image로 포장한다. PC에서 시험할 파일은 **boot-j716f.img**다. TB-J706F나 TB-9707F용 이미지는 다른 보드용이다.

## 내장 저장장치에 설치하기 전에 한 번 부팅하기

1. Android 설정의 모델 번호가 TB-J716F인지 확인하고 개인 데이터를 백업한다. 아래 절차는 부트로더가 이미 잠금 해제된 기기 기준이다. 잠금 해제는 데이터를 초기화할 수 있다.
2. PC에 [Android SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools)를 설치한다. USB 디버깅을 켠 태블릿을 연결하고 PC 연결을 승인한다. 터미널에서 다음을 실행한다.

   ```sh
   adb devices
   adb shell getprop ro.product.model
   adb reboot bootloader
   fastboot devices
   ```

3. 연결된 기기가 한 대이고 모델을 확인했다면 이미지가 있는 디렉터리에서 아래 명령을 실행한다. 이 명령은 이미지를 RAM으로 전송해 이번 한 번 부팅한다.

   ```sh
   fastboot boot boot-j716f.img
   ```

4. UEFI 메뉴가 표시되는지 확인하고 화면을 기록한다. 처음에는 파티션 편집이나 OS 설치를 진행하지 않는다. 재시작 메뉴 또는 전원 버튼을 길게 눌러 기기를 재시작해 Android 복귀까지 확인한다. fastboot에서 오류가 나면 오류 원문을 기록하고 fastboot에 남아 있을 때는 `fastboot reboot`로 복귀한다. 임시 부팅을 거부하는 기기에서 영구 플래시로 우회하면 기존 부팅 환경을 잃을 수 있다.

## Windows를 목표로 할 때

다음 검증 단계는 ARM64 WinPE의 부트 매니저 실행, 화면과 입력 확인, UFS 디스크의 읽기 전용 인식 확인 순이다. 각 단계에서 ZUI 버전, RAM 용량, UEFI 이미지 SHA-256, 관찰한 화면이나 오류를 함께 기록한다. 같은 Snapdragon을 사용하는 다른 제품의 드라이버만으로 TB-J716F의 주변기기 지원을 판단할 수 없다. 이 단계들이 확인될 때까지 Windows용 ESP나 시스템 파티션을 만들 이유가 없다.
