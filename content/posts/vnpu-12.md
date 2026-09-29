---
title: "vnpu 개발기 #12 - 버퍼 객체 정보 읽기"
date: 2026-09-29T13:09:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

핸들을 주면 BO의 크기와 mmap 오프셋을 돌려주는 BO_INFO ioctl을 넣었다.
11편에서는 1바이트로 만든 BO가 정말 4096바이트가 됐는지 볼 방법이 없었는데, 이제 크기를 읽어 확인할 수 있다.
오프셋은 다음에 BO를 mmap할 때 쓴다.

```c
struct drm_vnpu_bo_info {
	__u32 handle;
	__u32 pad;
	__u64 size;
	__u64 mmap_offset;
};
```
pad는 size를 8바이트 경계에 두려고 채운 칸이고, 0이 아니면 거절한다.
ivpu는 BO의 flags와 장치 쪽 주소도 돌려주는데, vnpu에는 정의한 flags도 장치 쪽 주소도 아직 없어서 뺐다.

핸들러는 이 fd의 핸들 표에서 BO를 찾아 크기와 오프셋을 채운다. 못 찾으면 ENOENT다.

크기는 올림이 바뀌는 경계와 가장 큰 값을 봤다.
1과 4096은 4096, 4097은 8192가 나왔다.
가장 큰 0xFFFFF00000도 그대로 나왔다. 4 GiB가 넘는 값도 잘리지 않는다.
오프셋은 0이 아니고, 두 BO의 구간이 겹치지 않았다.

없는 핸들은 세 가지를 넣었다. 유효하지 않은 핸들 값 0, 닫은 핸들, 다른 fd에서 만든 핸들이다. 셋 다 ENOENT다.
pad에 1을 넣으면 EINVAL이다.
```
# PASSED: 19 / 19 tests passed.
# Totals: pass:19 fail:0 xfail:0 xpass:0 skip:0 error:0
```

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
