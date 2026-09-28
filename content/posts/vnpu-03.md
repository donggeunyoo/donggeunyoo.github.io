---
title: "vnpu 개발기 #3 - 빈 NPU 디바이스를 꽂고 드라이버로 잡기"
date: 2026-09-28T18:55:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

드라이버보다 디바이스를 먼저 만들었다.
드라이버가 probe할 대상이 없으면, 드라이버를 짜도 돌려 볼 방법이 없어서다.

QEMU에서 PCI 디바이스를 만드는 건 PCI 설정 공간을 채우는 일부터 시작한다.
리눅스는 부팅할 때 PCI 버스를 훑으면서 장치마다 벤더 ID, 디바이스 ID, 클래스 코드를 읽고, 그 값에 맞는 드라이버를 찾아 붙인다.

벤더 ID는 QEMU가 자체 가상 장치에 쓰는 0x1234를 썼다.
디바이스 ID는 0x4e50으로 직접 골랐다. 0x1234 안에서 비어 있는 번호이고, ASCII로 읽으면 "NP"다.
클래스 코드는 처리 가속기를 뜻하는 0x1200이다.
리눅스에는 이 값의 이름(PCI_CLASS_ACCELERATOR_PROCESSING)이 있는데 QEMU에는 없어서, 같은 이름으로 하나 추가했다.

처음엔 hw/pci/pci.h만 include해서 빌드가 깨졌다.
PCI 장치를 만들 때 쓰는 PCIDeviceClass와 TYPE_PCI_DEVICE는 hw/pci/pci_device.h에 있었다.
참고한 edu는 hw/pci/msi.h를 통해 이 헤더를 간접적으로 받고 있어서, edu 코드만 봐서는 알 수 없었다.

-device vnpu로 VM을 띄우니 게스트에서 이렇게 보였다.
```
0x1234 0x4e50 0x120000
```
아직 레지스터도 메모리도 없는 빈 장치고, 잡는 드라이버도 없다.

그다음 드라이버가 이 장치를 잡게 했다.
리눅스 PCI 드라이버는 맡을 벤더/디바이스 ID를 테이블로 들고 PCI 코어에 등록하고, PCI 코어는 찾아 둔 장치가 테이블과 맞으면 드라이버의 probe를 불러 준다.
테이블에 1234:4e50 하나를 적고, probe에서는 로그만 찍었다.
```
[   13.706517] vnpu 0000:00:04.0: probed
~ # ls /sys/bus/pci/drivers/vnpu/
0000:00:04.0  module        remove_id     unbind
```
insmod하자마자 드라이버가 00:04.0에 있는 장치에 붙었다.

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
