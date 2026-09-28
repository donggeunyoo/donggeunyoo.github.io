---
title: "vnpu 개발기 #9 - 인터럽트"
date: 2026-09-28T22:54:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

장치가 드라이버에게 인터럽트를 보낼 수 있게 하는 작업을 시작했다.
앞으로 IPC나 job 완료 알림은 전부 인터럽트로 온다.

인터럽트는 MSI로 보낸다.
MSI는 인터럽트 전용 선 대신, 장치가 정해진 주소에 정해진 값을 메모리 쓰기하는 방식이다. QEMU에서도 MSI를 보내는 코드는 결국 메모리 쓰기 한 번이다.
어떤 주소에 어떤 값을 쓸지는 운영체제가 정해서, 장치의 설정 공간에 있는 MSI capability에 적어 준다.

그래서 첫 단계는 설정 공간에 MSI capability를 넣는 것이다.
QEMU에서는 realize에서 msi_init 한 번이면 되고, 장치가 없어질 때는 msi_uninit으로 뺀다.
벡터는 하나, 64비트 주소로 보낼 수 있게 했다.

게스트에서 설정 공간을 직접 읽어 확인했다.
capability는 연결 리스트다. 오프셋 0x34(10진수 52)가 첫 항목의 위치를 가리키고, 각 항목의 첫 바이트가 종류, 둘째 바이트가 다음 항목의 위치다. MSI의 종류는 0x05다.
```
~ # c=/sys/bus/pci/devices/0000:00:04.0/config
~ # p=$(od -An -tu1 -j52 -N1 $c)
~ # echo CAP_PTR $p
CAP_PTR 64
~ # od -An -tx1 -j$((p)) -N2 $c
 05 00
```
0x40(64) 자리에 MSI(05)가 있고, 다음 위치가 00이라 capability는 이것 하나다.

다음은 장치가 실제로 인터럽트를 올리게 하는 것이다. 레지스터 두 개를 새로 뒀다.
0x04 IRQ_STATUS는 걸려 있는 인터럽트 비트다. 처리한 비트에 1을 쓰면 그 비트만 지워진다.
0x08 IRQ_RAISE에 비트를 쓰면 장치가 그 비트를 STATUS에 세우고 MSI를 보낸다.

MSI는 무슨 일이 일어났는지는 알려 주지 않는다. 드라이버가 인터럽트를 받고 STATUS를 읽어야 안다.
1을 쓴 비트만 지우게 한 건, 드라이버가 여러 비트 중 처리한 것만 골라 지울 수 있게 하려는 것이다.
IRQ_RAISE는 아직 인터럽트를 만들 진짜 일이 없어서, 드라이버가 인터럽트 경로를 시험할 수 있게 둔 레지스터다.

드라이버가 MSI를 켜기 전에는 신호는 안 가고 비트만 남는다. 그래서 먼저 devmem으로 비트가 서고 지워지는지만 봤다.
a는 5편과 같이 BAR 0의 시작 주소다.
```
~ # devmem $((a+4)) 32
0x00000000
~ # devmem $((a+8)) 32 0x5
~ # devmem $((a+4)) 32
0x00000005
~ # devmem $((a+4)) 32 0x1
~ # devmem $((a+4)) 32
0x00000004
~ # devmem $((a+4)) 32 0x4
~ # devmem $((a+4)) 32
0x00000000
```
RAISE에 0x5(비트 0, 2)를 쓰니 STATUS가 5가 됐고, 비트 0과 2를 하나씩 지우니 4, 0이 됐다.

마지막으로 드라이버가 MSI를 켜고 인터럽트를 받게 했다. 드라이버가 할 일은 세 가지다.
pci_alloc_irq_vectors로 MSI 벡터를 하나 받는다. 이때 리눅스가 장치의 MSI capability에 보낼 주소와 값을 적고 MSI를 켠다.
devm_request_irq로 그 인터럽트에 핸들러를 건다. 핸들러는 STATUS를 읽고, 읽은 비트를 그대로 써서 지운다.
pci_set_master로 버스 마스터를 켠다. MSI는 장치가 메모리에 쓰는 동작인데, QEMU는 PCI COMMAND 레지스터의 버스 마스터 비트가 꺼져 있으면 장치의 메모리 쓰기를 막는다.

드라이버를 올리고 RAISE에 1을 써서 장치가 인터럽트를 보내게 했다.
```
~ # grep vnpu /proc/interrupts
  24:          0          0          0          0  PCI-MSI-0000:00:04.0    0-edge      vnpu
~ # devmem $((a+8)) 32 0x1
~ # grep vnpu /proc/interrupts
  24:          1          0          0          0  PCI-MSI-0000:00:04.0    0-edge      vnpu
~ # devmem $((a+4)) 32
0x00000000
```
/proc/interrupts는 인터럽트마다 CPU별로 받은 횟수를 보여 준다. RAISE를 쓰기 전 0이던 횟수가 쓴 뒤 1이 됐다.
마지막 STATUS가 0인 건 핸들러가 비트를 지웠기 때문이다.

핸들러와 벡터는 둘 다 드라이버가 떨어질 때 자동으로 풀린다. 떼었다 다시 붙여도 인터럽트가 다시 오는지 봤다.
```
~ # echo 0000:00:04.0 > /sys/bus/pci/drivers/vnpu/unbind
~ # grep -c vnpu /proc/interrupts
0
~ # echo 0000:00:04.0 > /sys/bus/pci/drivers/vnpu/bind
[   14.161948] [drm] Initialized vnpu 1.0.0 for 0000:00:04.0 on minor 0
[   14.162905] vnpu 0000:00:04.0: ID 0x564e5055
~ # devmem $((a+8)) 32 0x1
~ # grep vnpu /proc/interrupts
  24:          1          0          0          0  PCI-MSI-0000:00:04.0    0-edge      vnpu
```
unbind 뒤엔 vnpu 인터럽트 줄이 없어졌고, 다시 bind하고 RAISE를 쓰니 새로 등록된 핸들러가 다시 한 번 받았다.

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
