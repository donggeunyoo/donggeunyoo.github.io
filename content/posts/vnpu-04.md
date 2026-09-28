---
title: "vnpu 개발기 #4 - BAR 0 달기"
date: 2026-09-28T19:12:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

장치에 드라이버가 읽고 쓸 메모리 창을 달았다.
지금까지 장치는 PCI 설정 공간, 그러니까 자기소개만 있었다.

PCI 장치는 설정 공간의 BAR(Base Address Register)로 "이만한 크기의 메모리 창이 필요하다"고 알린다.
부팅할 때 이 창이 물리 주소 공간 어딘가에 배치되고, 그 뒤로 CPU가 그 주소를 읽고 쓰면 RAM이 아니라 장치로 간다.
이게 MMIO이고, 레지스터 접근은 전부 이 창을 거친다.

QEMU에서는 이 창을 MemoryRegion으로 만들고 pci_register_bar로 BAR에 연결한다.
크기는 4 KiB로 잡았다. CPU가 메모리를 페이지 단위로 매핑하니 이보다 작게 잡을 이유가 없었다.
읽고 쓸 때 부를 콜백은 아직 안 넣었다. QEMU는 콜백이 없는 창을 어떤 접근도 받지 않는 영역으로 처리한다.

부팅 로그에서 창이 잡힌 걸 확인했다.
```
pci 0000:00:04.0: BAR 0 [mem 0xfebf1000-0xfebf1fff]
```

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
