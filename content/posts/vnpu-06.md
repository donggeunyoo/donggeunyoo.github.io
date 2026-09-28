---
title: "vnpu 개발기 #6 - 드라이버에서 ID 읽기"
date: 2026-09-28T21:14:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

드라이버가 probe에서 ID 레지스터를 읽게 했다.
ID가 0x564e5055가 아니면 이 장치를 잡지 않는다.

레지스터를 읽기 전에 두 가지가 필요하다.
pcim_enable_device로 장치를 켜고, pcim_iomap_region으로 BAR 0을 커널 주소에 매핑해야 드라이버가 readl로 읽을 수 있다.

pcim_iomap_region은 매핑과 함께 BAR 0 범위를 "vnpu" 이름으로 확보한다.
둘 다 이름의 m이 managed라는 뜻이라, 드라이버가 떨어질 때 커널이 알아서 되돌린다.
unbind했다가 다시 bind해서 정말 풀리는지 봤다.
```
[   13.877076] vnpu 0000:00:04.0: ID 0x564e5055
~ # grep vnpu /proc/iomem
    febf1000-febf1fff : vnpu
~ # echo 0000:00:04.0 > /sys/bus/pci/drivers/vnpu/unbind
~ # grep -c vnpu /proc/iomem
0
~ # echo 0000:00:04.0 > /sys/bus/pci/drivers/vnpu/bind
[   13.893207] vnpu 0000:00:04.0: ID 0x564e5055
```
풀어 주는 코드를 따로 안 썼는데, unbind하자 범위가 풀렸고 다시 bind해도 문제없이 잡혔다.

ID가 다를 때 정말 거절하는지는 QEMU의 ID 값을 잠깐 0x12345678로 바꿔서 확인했다.
```
[   14.084977] vnpu 0000:00:04.0: error -ENODEV: unexpected ID 0x12345678
~ # ls /sys/bus/pci/drivers/vnpu/
bind       module     new_id     remove_id  uevent     unbind
~ # grep -c vnpu /proc/iomem
0
```
드라이버는 장치를 잡지 않았고, ID를 읽기 전에 확보했던 BAR 범위도 풀려 있었다.

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
