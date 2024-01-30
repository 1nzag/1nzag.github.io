---
layout: simple
title: Build Custom kernel on Pixel 6
---

## **Intro**

실기기에서 안드로이드 익스플로잇을 테스트하기 위해선 커스텀 커널을 올리는 과정이 필요하다. 필요하면 KASAN 및 kgdb를 올릴 수 있으면 더 좋다. 

해당 포스트는 Pixel 6 에 커스텀 커널을 빌드하여 올리는 과정을 소개한다. 

## **Build Kerenl**

먼저 픽셀 커널과 관련된 커널 소스를 다운받아야 한다. 

나는 해당 과정을 통해 커널 소스를 받았다. 

```bash
repo init --depth=1 -u https://android.googlesource.com/kernel/manifest -b gs-android-gs-raviole-mainline
```

`gs-android-gs-raviole-mainline` 브랜치를 가져온 이유는 pixel 6 기기를 지원하는 커널 의 브랜치가 해당 브랜치이기 때문이다. 

각 기기당 지원하는 브랜치를 보려면 [해당 링크를 보면 된다. ](https://source.android.com/docs/setup/build/building-pixel-kernels?hl=ko)

커널을 빌드해주기 전에 vendor_boot 를 픽셀 기기에 맞춰줘야 한다. 이는 공급업체에 맞는 램디스크 부트로더를 맞춰주기 위함이다. pixel6 에 맞는 vendor_boot 는 공장 펌웨어에서 추출 해 낼 수 있는데, 공장 펌웨어는 [해당 링크에서 구할 수 있다.](https://developers.google.com/android/images?hl=ko) 

나는 pixel 6 pro에 맞는 Raven for pixel 6 Pro version 12.0.0 (SD1A.210817.015.A4, Oct 2021) 펌웨어를 다운받았다. 

해당 펌웨어를 압축해제하고, image 가 들어있는 압축파일을 압축 해제 하면 `vendor_boot.img` 파일을 확인 할 수 있다. 여기서 공급업체 램디스크 이미지를 추출하려면, 받은 커널 소스 디렉토리에 있는 `tools/mkbooting/unpack_booting.py` 스크립트를 사용하면 된다. 

```bash
tools/mkbooting/unpack_booting.py --boot_img [vendor_boot.img path] --out vendor_boot_out
```

그럼 지정한 `vendor_boot_out` 이라는 이름의 디렉토리가 생기고 아래에는 다음과 같은 파일들이 생성된다. 

```bash
├── bootconfig
├── dtb
├── vendor_ramdisk
├── vendor_ramdisk00
├── vendor_ramdisk01
└── vendor-ramdisk-by-name
    ├── ramdisk_ -> ../vendor_ramdisk00
    └── ramdisk_dlkm -> ../vendor_ramdisk01

1 directory, 7 files
```

여기에 `ramdisk_` 파일을 가져오면 되는데, 해당 파일은 `vendor_ramdisk00` 의 링크이므로 해당 파일을 이용하면 된다. 

이제 이 공급업체 램디스크 부트로더를 커널 소스에서 받은 prebuilt ramdisk image에 덮어씌워준다. 

```bash
cp vendor_boot_out/vendor_ramdisk00 prebuilds/boot-artifacts/ramdisk/vendor_ramdisk-oriole.img
```

그 뒤에 커널을 다음과 같이 빌드해준다. 

```bash
./build_slider.sh
```

그러면 `out` 디렉토리에 다음과 같은 파일들이 생성된다. 

+ boot.img
+ dtbo.img
+ vendor_boot.img
+ vendor_dlkm.img

## **Flash Image**

빌드를 통해 생성된 이미지를 pixel 기기에 플래싱 해주자. 플래싱은 Android ndk 에 fastboot 툴을 이용해주면 된다. 

먼저 pixel 6 에서 fastboot 모드에 진입해준다. 방법은 전원을 끈 상태에서 전원버튼 + 볼륨 다운버튼을 길게 눌러주면 된다. 

그리고 pixel 기기를 컴퓨터에 usb로 연결해준 다음, fastboot로 연결한다. 

```bash
cutebear@cutebear:~/Android/Sdk/platform-tools$ fastboot devices
23011F********R	 fastboot
```

먼저 oem lock 과 flash lock을 풀어주고, oem verification 을 해제해준다. 

```bash
fastboot flashing unlock
fastboot oem unlock
fastboot oem disable-verification
```

다음엔 차례대로 빌드한 이미지를 플래싱 해주면 된다. 

```bash
fastboot flash boot boot.img
fastboot flash dtbo dtbo.img
fastboot flash vendor_boot vendor_boot.img
fastboot reboot fastboot
fastboot flash vendor_dlkm vendro_dlkm.img
fastboot reboot
```

그럼 커스텀 빌드된 커널을 실행시킬 수 있게 된다. 

## **Conclusion**

아직 kasan과 kgdb 는 올려보지 못했다. 성공하면 다음 포스트에 이어서 쓰도록 하겠다. 

## **References**

+ <https://source.android.com/docs/setup/build/building-pixel-kernels?hl=ko>
+ <https://developers.google.com/android/images?hl=ko>
+ <https://xdaforums.com/t/guide-compile-kernel-raviole-from-sources-5-10-mainline.4596285/>