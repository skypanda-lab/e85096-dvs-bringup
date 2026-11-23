# E850-96 OV5647 실습 이미지 빌드 기록

Windows의 WSL2 Ubuntu에서 E850-96용 Linux 5.10 커널과 Debian 12 설치 이미지를 만든 기록이다. OV5647은 CN4에 연결하는 일반 이미지 센서다. 검증 범위는 이미지 생성까지다.

## 빌드 환경

- 호스트는 Windows의 Ubuntu 26.04.1 LTS (WSL2, x86_64)다.
- 기반 커널은 [Linaro E850-96](https://gitlab.com/Linaro/96boards/e850-96/kernel)의 android-exynos-5.10-linaro 브랜치, 커밋 3b6b9ca5103485746684be1921bc01c0c04c8420이다.
- 컴파일러는 Android Clang 14.0.7 (clang-r450784e)이다. Debian 이미지는 Docker Desktop에서 ARM64 Ubuntu 22.04 컨테이너로 만든다.

## 커널과 Device Tree 빌드

WSL Ubuntu에서 실행한다. 이 문서가 있는 포크 브랜치를 홈의 e85096-dvs-bringup 폴더에 복제한 상태를 기준으로 한다.

~~~bash
sudo apt update
sudo apt install -y git wget build-essential bc bison flex libssl-dev libelf-dev \
  gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu device-tree-compiler \
  dwarves cpio rsync libncurses-dev python-is-python3

git clone --depth 1 --branch android-exynos-5.10-linaro \
  https://gitlab.com/Linaro/96boards/e850-96/kernel.git ~/e850-ov5647-kernel
cd ~/e850-ov5647-kernel
git rev-parse HEAD

mkdir -p ~/e850-ov5647-toolchain
wget -O ~/e850-ov5647-toolchain/clang.tar.gz \
  https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/8b0689df9225c41530631f9bd6d7703815f08ae7/clang-r450784e.tar.gz
tar -xzf ~/e850-ov5647-toolchain/clang.tar.gz -C ~/e850-ov5647-toolchain
export PATH="$HOME/e850-ov5647-toolchain/bin:$PATH"
clang --version
~~~

커널 커밋이 위와 일치하는지 확인한 뒤, 센서 드라이버와 CSI 캡처 설정, HDMI 변경을 적용한다. 추가 패치는 SoC Device Tree가 디스플레이 노드를 포함하게 한다. 이 노드가 빠진 상태에서는 HDMI 오버레이를 합칠 때 FDT_ERR_NOTFOUND가 발생했다.

~~~bash
lab="$HOME/e85096-dvs-bringup"
out="$HOME/e850-ov5647-out"
sh "$lab/modules/ov5647-pablo/apply.sh" "$PWD"
sh "$lab/modules/ov5647-pablo/bypass.sh" "$PWD"
sh "$lab/modules/e850-hdmi/apply.sh" "$PWD"
git apply "$lab/tools/linux510/enable-e850-display-dt.patch"

mkdir -p "$out"
scripts/kconfig/merge_config.sh -m -O "$out" \
  arch/arm64/configs/gki_defconfig \
  arch/arm64/configs/s5e3830-base_defconfig \
  arch/arm64/configs/erd3830_t_defconfig \
  arch/arm64/configs/erd3830_t_mali.cfg
scripts/config --file "$out/.config" \
  -e DEVTMPFS -e DEVTMPFS_MOUNT -d KVM -d KVM_S2MPU \
  -e VT -e VT_CONSOLE -e FRAMEBUFFER_CONSOLE \
  -e FRAMEBUFFER_CONSOLE_DETECT_PRIMARY

build=(make O="$out" ARCH=arm64 LLVM=1 LLVM_IAS=1
  CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_COMPAT=arm-linux-gnueabi-
  TARGET_SOC=s5e3830 KCFLAGS=-D__ANDROID_COMMON_KERNEL__)
"${build[@]}" olddefconfig
"${build[@]}" -j8 Image modules dtbs
~~~

커널 링크 단계에서 python 명령을 찾지 못하는 오류가 발생했다. WSL에서 python-is-python3를 설치한 뒤 같은 빌드 명령을 다시 실행해 해결했다. ARM64 커널 Image와 fimc-is.ko, is-cis-ov5647.ko가 생성됐고, 합친 Device Tree에 ov5647@36이 포함된 것을 확인했다.

## 설치 이미지 생성

Docker Desktop에서 Ubuntu 배포판의 WSL 통합을 켠 다음, WSL Ubuntu에서 실행한다. 첫 명령의 출력은 aarch64여야 한다.

~~~bash
docker run --rm --platform linux/arm64 ubuntu:22.04 uname -m
lab="$HOME/e85096-dvs-bringup"
out="$HOME/e850-ov5647-out"
images="$HOME/e850-ov5647-images"
mkdir -p "$images"

docker run --rm --platform linux/arm64 \
  -v "$out:/o:ro" -v "$lab/tools/linux510:/t:ro" -v "$images:/out" \
  ubuntu:22.04 sh /t/build-bootimg.sh root=PARTLABEL=userdata

(
  set -o pipefail
  docker run --rm --privileged --platform linux/arm64 \
    -v "$out:/o:ro" -v "$lab/tools/linux510:/t:ro" -v "$images:/out" \
    ubuntu:22.04 sh -c \
    'sed "s@ >/dev/null 2>&1@@g" /t/build-rootfs.sh > /tmp/build-rootfs.sh &&
     sh -x /tmp/build-rootfs.sh' 2>&1 | tee "$images/rootfs-build.log"
)
ls -lh "$images/"{boot.img,rootfs.simg}
~~~

첫 실행에서는 Debian 이미지가 생성되지 않았다. 패키지 설치 출력을 표시하고 상세 로그를 남겨 다시 실행한 뒤 Debian 설치와 두 카메라 모듈의 포함을 확인했다. 첫 실행의 종료 원인은 확인되지 않았다. 아래 해시는 당시 만든 파일을 식별한다. 다시 빌드하면 Debian 패키지와 이미지 메타데이터가 달라질 수 있다.

| 파일 | 바이트 | SHA-256 |
| --- | ---: | --- |
| boot.img | 52,860,928 | 75bd2570f864085191dc85288834c22a0bb636434da414d890612e8033853c96 |
| rootfs.simg | 398,557,704 | 0d50afcd109d279f32fd8246db5af90a3530fef513d5390055772a1be2146daf |

## 보드 검증 상태

HDMI로 Android 부팅을 확인했다. USB 디버깅을 켜고 보드의 J1 Micro-USB를 PC에 연결했으나 Windows 장치 목록에서 보드를 확인하지 못했다. Android 백업, 이미지 기록, Debian 부팅과 OV5647 촬영은 아직 진행 전이다.

현재 만든 부팅 이미지에는 Android 복구용 원본 boot 파티션 백업이 들어 있지 않다. 보드에 이미지를 기록하기 전에 Android 백업과 복구 경로를 확인한다. Debian rootfs.simg를 userdata에 기록하면 기존 Android 사용자 데이터가 지워진다. [기존 OV5647 구동 및 복구 기록](E85096-LINUX510-BOOT.md)
