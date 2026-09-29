---
title: "vnpu 개발기 #8 - 첫 ioctl, GET_PARAM"
date: 2026-09-28T22:11:00+09:00
tags: ["vnpu", "qemu", "accel", "npu"]
---

사용자 공간 프로그램이 ioctl로 장치 정보를 물어볼 수 있게 했다.
첫 항목은 PCI 디바이스 ID 하나다. ivpu도 첫 ioctl이 GET_PARAM이고, 첫 항목이 디바이스 ID다.

ioctl 번호와 주고받는 구조체는 드라이버와 사용자 공간이 똑같이 알아야 해서, 둘이 같이 include하는 uapi 헤더에 뒀다.
```c
struct drm_vnpu_param {
	__u32 param;
	__u32 pad;
	__u64 value;
};
```
value가 64비트라 구조체 크기를 64비트 배수로 맞추려고 pad를 명시했고, pad가 0이 아니면 드라이버가 거절한다.
커널 문서(Documentation/process/botching-up-ioctls.rst)에 있는 규칙이다.

드라이버 쪽은 ioctl 표에 함수 하나를 등록하는 게 전부다.
DRM의 drm_ioctl이 번호로 표를 찾고, 구조체를 커널로 복사해 드라이버 함수를 부른 뒤, 결과를 다시 사용자 공간으로 복사해 준다.

테스트는 커널 셀프테스트 하네스(kselftest_harness.h)로 짰다.
처음 테스트 빌드는 이 에러로 깨졌다.
```
kselftest_harness.h:640:52: error: comparison of integer expressions of different signedness: ‘int’ and ‘__u64’ {aka ‘long long unsigned int’} [-Werror=sign-compare]
```
EXPECT_EQ(0x4e50, args.value)에서 0x4e50은 부호 있는 int이고 value는 부호 없는 __u64라서다. 0x4e50ULL로 타입을 맞췄다.

테스트는 다섯 개다. 디바이스 ID, 모르는 param, 0이 아닌 pad, NULL 인자, 그리고 장치가 떨어진 뒤의 ioctl이다.
마지막 테스트는 파일을 연 채로 unbind하고, 그 파일로 ioctl을 불러 ENODEV가 오는지 본다.
drm_ioctl이 맨 앞에서 장치가 뽑혔는지 검사해서, 드라이버 함수까지 가지 않는다.

그런데 테스트를 끝내고 보니 장치 파일 이름이 바뀌어 있었다.
```
~ # ls /dev/accel/
accel1
```
마지막 테스트는 끝날 때 다시 bind해서 원래대로 돌려놓는데, accel0이 아니라 accel1로 생겼다. 이러면 테스트를 한 번 더 돌릴 때 accel0을 못 연다.
원인은 열어 둔 파일이었다. 파일이 옛 장치의 참조를 잡고 있는 동안은 옛 장치 객체가 해제되지 않고, 장치 번호 0도 반납되지 않는다. 그래서 새 장치가 1번을 받았다.
bind 전에 파일을 먼저 닫게 고치니, 두 번 연달아 돌려도 둘 다 통과하고 장치도 accel0으로 돌아왔다.
```
# PASSED: 5 / 5 tests passed.
# PASSED: 5 / 5 tests passed.
accel0
```

Repo: [github.com/donggeunyoo/vnpu](https://github.com/donggeunyoo/vnpu)
