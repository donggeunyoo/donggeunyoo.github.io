---
title: "vnpu 개발기 #13 - 버퍼 객체 mmap과 크기 상한"
date: 2026-09-29T14:41:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

BO를 사용자 공간에 mmap해서 CPU로 직접 읽고 쓸 수 있게 했다.
드라이버 코드는 한 줄도 없다. accel 장치 파일의 mmap은 이미 DRM 공통 함수(drm_gem_mmap)로 이어져 있고, BO는 shmem 헬퍼의 mmap을 쓴다.
mmap에는 12편의 mmap 오프셋을 넘긴다. 커널은 이 오프셋으로 BO를 찾고, 이 fd에 권한이 있는지 본 뒤 매핑한다.

새 BO는 0으로 차 있었고, 쓴 값은 munmap했다가 다시 매핑해도 그대로였다.
BO보다 긴 매핑, BO 중간부터의 매핑, MAP_PRIVATE는 EINVAL이다. 오프셋 0과 닫은 BO의 오프셋도 찾을 BO가 없어서 EINVAL이다.
다른 fd의 오프셋은 EACCES다. 오프셋 공간은 장치에 하나라 BO는 찾지만, 그 fd에는 권한이 없다.

11편에서 BO를 만들 때는 페이지를 잡지 않는다고 했는데, 그 페이지를 잡는 게 mmap이다.
mmap하는 순간 페이지 목록표를 만들고, BO의 페이지를 전부 메모리에 올려 고정한다. 목록표는 페이지마다 8바이트라 BO 크기의 1/512다.
그래서 11편의 가장 큰 BO(0xFFFFF00000바이트)를 작은 프로그램으로 mmap해 봤다.
```
~ # bigmmap
size 0xfffff00000 offset 0x100000000
[   14.547686] bigmmap: vmalloc error: size 2147481600, exceeds total pages, mode:0xcc0(GFP_KERNEL), nodemask=(null),cpuset=/,mems_allowed=0
[   14.549670] CPU: 0 UID: 0 PID: 81 Comm: bigmmap Tainted: G           O        7.2.0 #1 PREEMPT(lazy) 
[   14.549676] Tainted: [O]=OOT_MODULE
[   14.549677] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.17.0-0-gb52ca86e094d-prebuilt.qemu.org 04/01/2014
[   14.549679] Call Trace:
```
호출 스택과 메모리 상태가 90줄 가까이 이어진 뒤에 mmap이 실패했다.
```
mmap failed: Cannot allocate memory
```
목록표가 약 2 GiB라 VM의 전체 메모리(330 MB)보다 컸고, vmalloc이 경고를 찍고 거절했다.
호출 스택을 뿜게 두는 대신, BO를 만들 때 상한을 검사해서 넘으면 에러 코드를 돌려주게 했다.

ivpu는 BO 핸들을 만들 때 장치 주소를 정해진 범위에서 잡아 주고, 범위 크기는 하드웨어 세대마다 고정이다. 37XX 다음 세대는 256 GiB다.
범위보다 큰 BO는 받을 주소가 없어서 만들 때 막힌다.
vnpu도 장치 주소 공간 크기를 고정하고, 그 크기를 BO 크기의 상한으로 두기로 했다.

후보는 128 GiB와 64 GiB였고, 둘 다 mmap해 봤다.
```
~ # bigmmap 0x2000000000
size 0x2000000000 offset 0x100000000
mmap failed: Cannot allocate memory
```
```
~ # bigmmap 0x1000000000
size 0x1000000000 offset 0x100000000
mmap failed: Cannot allocate memory
```
둘 다 경고 없이 ENOMEM으로 끝났다.
차이는 여유다. 128 GiB의 목록표 256 MiB는 그때 빈 메모리(286 MB)와 거의 같고, 64 GiB의 128 MiB는 그 절반이다. 그래서 64 GiB로 정했다.

mmap이 성공하려면 결국 BO의 페이지 전부가 그 순간 빈 메모리에 들어가야 한다.
상한은 그걸 보장하지 않고, 목록표가 메모리에 여유 있게 들어가는 쪽으로 상황을 이끌 뿐이다.

상한을 넘는 BO는 EINVAL로 거절한다. ivpu는 주소를 못 받으면 ENOSPC가 나오는데, 주소 공간 전체보다 큰 BO는 언제 요청해도 들어갈 수 없어서 잘못된 인자로 봤다.
64 GiB보다 1바이트 큰 BO는 EINVAL, 64 GiB BO는 만들어진다.
```
# PASSED: 26 / 26 tests passed.
# Totals: pass:26 fail:0 xfail:0 xpass:0 skip:0 error:0
```

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
