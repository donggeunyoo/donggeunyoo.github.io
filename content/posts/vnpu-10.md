---
title: "vnpu 개발기 #10 - 장치 리셋"
date: 2026-09-28T23:14:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

장치가 리셋될 때 IRQ_STATUS를 0으로 되돌리게 했다.
9편에서 만든 이 레지스터는 게스트를 재부팅해도 값이 그대로 남아 있었다.

make run은 QEMU를 -no-reboot로 띄우는데, 이러면 게스트가 재부팅할 때 QEMU가 그냥 꺼진다.
그래서 이 옵션만 빼고 QEMU를 직접 띄웠다.
드라이버를 올리면 핸들러가 비트를 지우니까, 드라이버 없이 RAISE에 0x5를 쓰고 재부팅했다.
a는 5편과 같이 BAR 0의 시작 주소다.
```
~ # devmem $((a+8)) 32 0x5
~ # devmem $((a+4)) 32
0x00000005
~ # reboot -f
[   14.562993] reboot: Restarting system
[   14.563494] reboot: machine restart
```
다시 부팅한 뒤 STATUS를 읽었다.
```
~ # devmem $((a+4)) 32
0x00000005
```
재부팅 전에 세운 5가 그대로 남아 있다.

게스트가 재부팅하면 QEMU는 기계 전체를 리셋하고, 버스 트리를 따라 모든 장치의 리셋 함수를 부른다.
PCI 설정 공간의 COMMAND와 MSI 설정은 PCI 공통 코드가 지워 주지만, irq_status는 vnpu만 가진 값이라 vnpu가 지워야 한다.
그런데 vnpu에는 리셋 함수가 없었다.

QEMU의 리셋은 enter, hold, exit 세 단계로 나뉘고, 모든 장치의 enter가 끝나야 hold로 넘어간다.
enter에서는 자기 상태만 되돌리고, 인터럽트를 올리거나 게스트 메모리에 쓰는 것처럼 바깥에 영향을 주는 일은 hold부터 할 수 있다.
irq_status를 0으로 만드는 건 vnpu 안의 값만 바꾸는 일이라 enter 단계에 넣었다.

고친 뒤 같은 순서로 다시 해 봤다.
재부팅 전 STATUS는 똑같이 5였고, 재부팅 뒤에는 0이 나왔다.
```
~ # devmem $((a+4)) 32
0x00000000
```

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
