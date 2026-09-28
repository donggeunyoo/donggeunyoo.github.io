---
title: "vnpu 개발기 #5 - 첫 레지스터"
date: 2026-09-28T21:01:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

BAR 0 창에 첫 레지스터를 넣었다.
오프셋 0x00을 읽으면 0x564e5055가 나오는 ID 레지스터다.

드라이버가 장치를 잡고 가장 먼저 할 일은 이 창 너머에 정말 vnpu가 있는지 확인하는 것이다.
창 연결이 어딘가 잘못돼 있으면 이 값이 안 나오니까, 이 값이 나오면 BAR 배치부터 매핑까지 제대로 됐다는 뜻이 된다.
값은 16진수로 56 4e 50 55, ASCII로 "VNPU"다.

게스트가 창을 읽으면 KVM이 그 접근을 QEMU로 넘기고, QEMU는 그 창의 MemoryRegionOps에 등록된 read 콜백을 부른다.
콜백은 창 시작 기준 오프셋을 받아서, 0x00이면 ID를 돌려준다.
레지스터는 전부 32비트라서 4바이트 접근만 받게 했다.

읽기만 필요한 레지스터지만 write 콜백도 넣어야 했다.
write 콜백 없이 게스트가 BAR 0 시작 주소(a)에 한 번 쓰게 했더니, QEMU가 바로 죽었다.
```
~ # devmem $a 32 0x1
Thread 4 "CPU 1/KVM" received signal SIGSEGV, Segmentation fault.
rip            0x0                 0x0
#0  0x0000000000000000 in ??? ()
#1  0x0000555555da7f81 in memory_region_write_with_attrs_accessor (...) at ../system/memory.c:513
```
rip이 0이다. write 콜백이 비어 있으면 QEMU는 write_with_attrs 경로로 넘어가는데, 그것도 비어 있어서 NULL 함수 포인터를 부른 것이다.
그래서 쓰기는 로그만 남기고 무시하는 콜백을 넣었다.

드라이버는 아직 없어서, 게스트의 busybox devmem으로 물리 주소를 직접 읽고 써 봤다.
a에는 sysfs에서 읽은 BAR 0의 시작 물리 주소(0xfebf1000)를 넣었다.
```
~ # a=$(head -1 /sys/bus/pci/devices/0000:00:04.0/resource | cut -d" " -f1)
~ # devmem $a 32
0x564E5055
~ # devmem $((a+4)) 32
0x00000000
~ # devmem $a 32 0x1
~ # devmem $a 32
0x564E5055
```
ID는 그대로 나오고, 없는 레지스터는 0, 써도 QEMU는 멀쩡했다.
0x04는 창 안이라 우리 read 콜백이 받아서 0을 돌려준 것이다.

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
