---
title: "vnpu 개발기 #2 - 개발 환경 세팅"
date: 2026-09-28T18:22:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

NPU 디바이스를 어디에 만들지부터 정해야 했다.
앞으로 만들 NPU 드라이버가 잡을 NPU는 QEMU에 없으므로, 드라이버를 돌리려면 디바이스 모델부터 직접 만들어야 한다.

방법은 두 가지였다.
하나는 QEMU 소스를 받아서 hw/misc 아래에 PCI 디바이스로 직접 넣는 것이고, 다른 하나는 libvfio-user로 디바이스를 별도 프로세스로 만들고 QEMU는 vfio-user로 붙이는 것이다.

QEMU 포크로 정했다.
QEMU에 디바이스가 들어가는 원래 방식 그대로라서다.
대신 QEMU 전체를 소스에서 빌드해야 한다.
기준은 QEMU 최신 안정판인 11.1.1로 잡았다.

VM에서 돌릴 커널도 최신 릴리스인 7.2로 새로 빌드했다.
이전 프로젝트에서 쓰던 커널은 accel 서브시스템(CONFIG_DRM_ACCEL)이 꺼져 있어서 그대로 쓸 수 없었다.

이 둘로 VM을 띄우고, 로그만 찍는 빈 드라이버 모듈을 올렸다 내려 봤다.
```
[   14.027509] vnpu: loaded
~ # rmmod vnpu
[   14.029630] vnpu: unloaded
```
아직 디바이스는 없지만, make run 한 번으로 드라이버 빌드부터 VM 부팅까지 돈다.

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
