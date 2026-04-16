---
layout: simple
title: Build and emulate custom android kernel
---

## **Introdution**

[저번 포스트에서](/study/linux_kernel/android/android_kernel_debugging) 안드로이 커널을 수정하여 android emulator로 작동 시켰었다. 

이런 방법은 주로 AOSP 에서 배포하는 goldfish 커널을 이용한 방법이다. 

하지만 `5.4` 버전 이후 goldfish 배포는 중단되었고, 이후 배포되는 커널은 빌드해도 android emulator 로 에뮬레이팅이 되지 않는다. 

그 이유는 `5.4` 버전 이전에는 ramdisk를 사용하지만 그 이후에는 initramfs 를 이용한 early mount 를 사용하는 것으로 추정된다. 

해당 버전 이후로는 `cuttlefish`라는 새로운 에뮬레이팅 프레임워크로 에뮬레이션이 가능하고, 커널 커스텀 역시 일부 버전이 가능하다. 

인터넷에서 android emulation 을 하는 방법은 죄다 goldfish 커널로 에뮬레이팅 하는 방법만 있기에 해당 포스트에 cuttlefish 로 커스텀 빌드를 하는 방법을 소개한다. 


## **Install cuttlefish**

먼저 호스트에 cuttlefish 를 설치해야 한다. culttlefish를 사용허려면 호스트가 KVM 을 지원해야한다. 

나는 KVM을 지원하는 Ubuntu에서 cuttlefish를 설치했다. 

```bash
sudo apt install -y git devscripts config-package-dev debhelper-compat golang curl
git clone https://github.com/google/android-cuttlefish
cd android-cuttlefish
for dir in base frontend; do
  cd $dir
  debuild -i -us -uc -b -d
  cd ..
done
sudo dpkg -i ./cuttlefish-base_*_*64.deb || sudo apt-get install -f
sudo dpkg -i ./cuttlefish-user_*_*64.deb || sudo apt-get install -f
sudo usermod -aG kvm,cvdnetwork,render $USER
sudo reboot
```

다음과 같은 명령어로 호스트에 cuttlefish를 설치 할 수 있다. 

## **Build Kernel**

[AOSP 에서는 에뮬레이터에서 돌아가는 안드로이드 커널을 종종 배포한다.](https://source.android.com/docs/setup/build/building-kernels?hl=ko)

해당 페이지에서 안드로이드 에뮬레이터를 지원하는 적절한 커널을 다운받는다. 

나는 `common-android12-5.10`를 다운받았다. 

```bash
mkdir common-android12-5.10
cd common-android12-5.10
repo init -u https://android.googlesource.com/kernel/manifest -b common-android12-5.10
repo sync
```

그 후 내가 원하는 부분만 소스코드를 적절하게 고친 다음 KASAN을 적용해서 커널을 빌드했다. 

KASAN 관련 빌드 설정은 `common-modules/virtual-device` 디렉토리에서 구할 수 있다. 

```bash
BUILD_CONFIG=./common-modules/virtual-device/build.config.virtual_device_kasan.x86_64 ./build/build.sh
```

`5.13`버전 이후로는 `build.sh`가 아니라 `bazel`을 이용해 빌드하도록 패키지를 배포한다. 

```bash
tools/bazel build //common-modules/virtual-device:virtual_device_kasan_x86_64_dist
```

빌드를 하게 되면 out 디렉토리에 `dist` 가 생성되고 거기에는 다음과 같은 두 파일이 존재한다. 

+ **bzImage**: 커널 이미지
+ **initramfs.img**: 파일시스템 이미지

## **Download Emulator**

빌드를 하게 되면 `out` 디렉토리에 `dist` 패키지가 생성된다. 해당 `dist` 패키지를 이용해 `cuttlefish`로 에뮬레이팅을 할 수 있다. 

먼저 `cuttlefish`로 에뮬레이팅 프레임워크를 설치했다면, 주기적으로 개발되어 배포되는 에뮬레이터를 받아야 한다. 그 링크는 다음과 같다. 

+ <https://ci.android.com/builds/branches/>

나는 여기서 `aosp_android13-gsi` 브랜치에 있는 에뮬레이터를 다운받았다. 

거기에는 여러 아키텍처를 지원하는 에뮬레이터들이 있는데, 나는 x86_64 에서 테스트를 할 것이므로 `aosp_cf_x86_64_phone` 을 받았다. 

그 후 `Artifacts` 탭에 들어가 `cvd-host_package.tar.gz` 라는 이름의 호스트 패키지 아티팩트를 다운받았다. 

그 후 해당 패키지를 압축 해제 한 후에, 빌드한 커널과, 파일시스템 이미지를 인자로 주어 실행하면 된다. 

```bash
mkdir emulator
cp emulator
mv [host package artifact].tar.gz .
tar -xvf [host package artifact].tar.gz]

HOME=$PWD ./bin/cvd start -kernel_path=[path to bzImage] -initramfs_path=[path to initramfs.img]
```

그럼 안드로이드 에뮬레이팅이 실행되는 것을 확인할 수 있다. 

## **Access to GUI**

`cuttlefish`는 실행될때 기본적으로 `--start_webrtc` 옵션을 활성화 한다. 이 옵션은 웹을 통해 에뮬레이터의 GUI를 볼 수 있게 하는 옵션이다. 

해당 옵션을 주면 `https://localhost:8443` 으로 접속하여 GUI를 볼 수 있다. 

해당 GUI를 통해 디바이스를 ON 하면 에뮬이 실행되는 것을 확인할 수 있다.

![](/assets/img/study/build_and_emulate_custom_kernel/gui-emulate.png)

## **Acess to ADB**

다운받은 에뮬레이터는 여러가지 툴들을 지원한다. 그 툴들은 `bin` 디렉토리에서 확인 할 수 있다. 

```bash
$ ls ./bin
acloud         avbtool            cvd_host_bugreport           cvd_status           fsck.f2fs           log_tee    metrics          ms-tpm-20-ref   restart_cvd         tombstone_receiver  wmediumd_control
adb            bt_connector       cvd_internal_host_bugreport  cvd_test_gce_driver  gnss_grpc_proxy     lpmake     mkbootfs         mtools          root-canal          toybox              wmediumd_gen_config
adb_connector  config_server      cvd_internal_start           defrag.f2fs          health              lpunpack   mkbootimg        newfs_msdos     run_cvd             unpack_bootimg      x86_64-linux-gnu
allocd         console_forwarder  cvd_internal_status          dump.f2fs            kernel_log_monitor  lz4        mkenvimage_slim  operator_proxy  secure_env          webRTC
allocd_client  crosvm             cvd_internal_stop            extract-ikconfig     launch_cvd          make_f2fs  mmd              powerwash_cvd   socket_vsock_proxy  webrtc_operator
assemble_cvd   cvd                cvd_send_sms   
```

이 중에는 adb 라는 툴이 존재한다. 해당 adb를 통해 에뮬레이팅 된 커널에 쉘로 접근할 수 있다. 

```bash
$ adb shell
vsoc_x86_64:/ $ uname -a
Linux localhost 5.10.198-android12-9-00014-g99512f12160f-dirty #1 SMP PREEMPT Fri Dec 22 22:47:40 UTC 2023 x86_64 Toybox
```

## **Conclusion**

원래는 옛날 버전의 안드로이그 커널만 몇번 에뮬 해봤다가 상대적으로 요즘 커널을 에뮬레이팅 하려는데 계속 안돼서 고난을 겪었다.

이제는 그 문제가 해결됐으니 여러 원데이 공부를 할 수 있을것이라 생각된다. 

## **References**

+ <https://source.android.com/docs/setup/create/cuttlefish-use?hl=ko>
+ <https://source.android.com/docs/setup/build/building-kernels?hl=ko>
+ <https://source.android.com/docs/setup/create/cuttlefish-kernel-dev?hl=ko>