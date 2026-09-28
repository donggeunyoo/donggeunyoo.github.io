---
title: "vnpu 개발기 #11 - 버퍼 객체 만들기"
date: 2026-09-29T00:20:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

사용자 프로그램이 장치에 넘길 메모리를 만들 수 있게 했다.
이 메모리 덩어리를 버퍼 객체(BO, buffer object)라고 한다. ivpu에서도 작업을 제출할 때 필요한 BO들을 핸들 배열로 넘긴다.

BO는 DRM의 메모리 관리자인 GEM(Graphics Execution Manager)이 관리한다.
GEM은 BO 안에 뭐가 들었는지는 모르고, 사용자 공간에는 BO를 핸들이라는 번호로 보여 준다.
핸들 표는 장치 파일을 open할 때마다 따로 생긴다.

BO 뒤의 메모리는 shmem 헬퍼로 만든다.
shmem은 커널 안의 공유 메모리 파일 시스템이고, 사용자 공간에는 tmpfs로 보인다. 헬퍼는 BO마다 그 크기의 shmem 파일을 하나 만들어 둔다.
ivpu의 BO도 이 헬퍼 위에 있다.

uapi 구조체는 크기와 flags를 넣고 핸들을 받는다.
```c
struct drm_vnpu_bo_create {
	__u64 size;
	__u32 handle;
	__u32 flags;
};
```
ivpu 구조체에 있는 장치 쪽 주소는 아직 쓸 데가 없어서 뺐다.
flags는 아직 정의한 비트가 없어서 0이 아니면 거절한다.

드라이버는 DRIVER_GEM을 켜야 했다. 이게 켜져 있어야 open 때 fd마다 핸들 표가 생긴다.
핸들러는 크기를 4 KiB 배수로 올린 값이 0이면 거절하고, shmem BO를 만들어 핸들을 붙인다.
올린 값이 0이 되는 건 0을 넣었을 때와, 2^64 근처 값이 올림하다 넘칠 때다.

핸들이 진짜 BO를 가리키는지는 DRM 공통 ioctl인 GEM_CLOSE로 봤다.
만든 핸들을 닫으면 성공하고, 같은 번호를 한 번 더 닫으면 EINVAL이 온다.

크기는 경계마다 양쪽 값을 넣었다.
0은 EINVAL이고 1은 성공한다.
0xFFFFF00000(1 TiB - 1 MiB)까지는 성공하고, 1바이트만 더 커도 ENOSPC다.
2^64-4095는 올림하다 넘쳐 0이 되니 EINVAL이다.

가장 큰 값은 BO를 만들 때 같이 잡는 mmap 오프셋(나중에 mmap할 때 쓸 가짜 파일 위치) 공간의 크기다.
메모리 512 MiB짜리 VM에서 이렇게 큰 BO가 만들어지는 건, 만들 때 페이지를 잡지 않기 때문이다.

0과 2^64-4095를 막는 게 정말 크기 검사인지 보려고, 검사를 지우고 돌려 봤다.
```
#  RUN           vnpu.bo_create_size_min ...
# tools/vnpu-test.c:95:bo_create_size_min:Expected EINVAL (22) == errno (28)
# bo_create_size_min: Test failed
#          FAIL  vnpu.bo_create_size_min
not ok 7 vnpu.bo_create_size_min
#  RUN           vnpu.bo_create_size_offset_space ...
#            OK  vnpu.bo_create_size_offset_space
ok 8 vnpu.bo_create_size_offset_space
#  RUN           vnpu.bo_create_size_wrap ...
# tools/vnpu-test.c:117:bo_create_size_wrap:Expected EINVAL (22) == errno (28)
# bo_create_size_wrap: Test failed
#          FAIL  vnpu.bo_create_size_wrap
not ok 9 vnpu.bo_create_size_wrap
```
검사가 없으면 두 값 다 ENOSPC(28)로 실패한다. 크기 0짜리 BO를 만들다 오프셋 구간을 못 잡아서다.
잘못된 크기에는 EINVAL이 맞으니 검사는 되돌렸다.

flags에 1을 넣으면 EINVAL이다.
핸들은 fd마다 따로다. 첫 fd의 핸들을 다른 fd에서 닫으면 EINVAL이고, 첫 fd에서는 그대로 닫힌다.
```
# PASSED: 12 / 12 tests passed.
# Totals: pass:12 fail:0 xfail:0 xpass:0 skip:0 error:0
```
KASAN을 켠 커널인데 로그에 경고는 없었다.

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
