---
title: "vnpu 개발기 #7 - accel 장치로 등록하기"
date: 2026-09-28T21:40:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

드라이버를 accel 장치로 등록해서 /dev/accel/accel0을 만들었다.
사용자 공간 프로그램이 NPU에 말을 걸 통로다. 앞으로 버퍼 할당이나 job 제출은 이 파일의 ioctl로 들어오게 된다.

NPU 같은 계산 가속기는 리눅스 accel 서브시스템에 붙는다.
뼈대는 GPU용 DRM 코어를 그대로 쓰고, 장치 파일만 /dev/dri가 아니라 /dev/accel 아래 생긴다.
커널 문서(Documentation/accel/introduction.rst)에 따르면, 그래픽 쪽 사용자 공간 소프트웨어가 가속기를 GPU로 착각하지 않게 하려는 것이다.

드라이버 쪽에서 할 일은 두 가지다.
drm_driver에 DRIVER_COMPUTE_ACCEL 플래그를 켜고, 파일 연산은 DEFINE_DRM_ACCEL_FOPS로 만든다.
probe에서는 ID 확인이 끝난 뒤 devm_drm_dev_alloc으로 DRM 장치를 할당하고, drm_dev_register로 등록한다. 여기서 장치 파일이 생긴다.

처음 빌드는 실패했다.
```
include/drm/drm_accel.h:26:27: error: ‘drm_ioctl’ undeclared here (not in a function); did you mean ‘drm_poll’?
include/drm/drm_accel.h:27:27: error: ‘drm_compat_ioctl’ undeclared here (not in a function)
include/drm/drm_accel.h:31:27: error: ‘drm_gem_mmap’ undeclared here (not in a function); did you mean ‘drm_get_cap’?
```
DEFINE_DRM_ACCEL_FOPS가 펼쳐 넣는 함수들의 선언이 drm_accel.h 안에 없어서다.
drm_ioctl.h와 drm_gem.h를 직접 include해서 해결했다. ivpu도 이 둘을 같이 include한다.

등록은 devm이 아니라서, 드라이버가 떨어질 때 풀어 줄 remove가 필요하다.
remove에서는 drm_dev_unplug를 불렀다. 장치를 뽑혔음으로 표시하고, 진행 중인 접근이 끝나기를 기다린 뒤 등록을 푼다.

결과는 이렇다.
```
[   13.838326] [drm] Initialized vnpu 1.0.0 for 0000:00:04.0 on minor 0
~ # ls -l /dev/accel/
crw-------    1 0        0         261,   0 Sep 28 12:27 accel0
~ # exec 3< /dev/accel/accel0 && echo OPENED
OPENED
```
261은 accel 장치에 따로 배정된 major 번호다.

파일을 연 채로 장치가 떨어지는 경우도 돌려 봤다.
```
~ # exec 3< /dev/accel/accel0 && echo OPENED
OPENED
~ # echo 0000:00:04.0 > /sys/bus/pci/drivers/vnpu/unbind
~ # ls /dev/accel/
ls: /dev/accel/: No such file or directory
~ # exec 3<&-
~ # echo CLOSED
CLOSED
```
연 상태로 장치를 떨어뜨리고 나서, 3번으로 열어 둔 파일을 닫았다.
이미 사라진 장치의 파일을 닫는 건데도 패닉 없이 CLOSED까지 정상으로 찍혔다.
파일을 열 때 DRM 장치의 참조를 하나 잡고 닫을 때 놓기 때문에, 장치가 떨어져도 마지막 파일이 닫힐 때까지는 메모리가 살아 있다.

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
