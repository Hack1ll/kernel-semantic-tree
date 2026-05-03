# procfs / sysfs

## 한 문장 정의

kernel state와 device model을 file read/write처럼 보이게 노출하는 pseudo filesystem interface.

## 핵심 질문

디스크에 존재하지 않는 kernel state가 왜 파일처럼 읽히고 쓰이는가?

## 왜 필요한가

모든 kernel state를 전용 syscall로 만들면 interface가 너무 많아진다. procfs와 sysfs는 kernel 내부 상태를 file model로 노출해 user space가 `cat`, `echo`, library, monitoring tool로 조회하거나 설정하게 한다.

하지만 이 파일들은 일반 파일이 아니다. read/write는 disk I/O가 아니라 kernel callback 실행이다.

```text
path lookup
    -> pseudo inode / attribute
    -> read or write callback
    -> kernel object state
```

## procfs와 sysfs의 차이

procfs는 process와 kernel runtime state를 보여주는 성격이 강하다.

- `/proc/<pid>/...`
- `/proc/self/status`
- `/proc/sys/...`
- `/proc/net/...`

sysfs는 device model과 kobject hierarchy를 노출하는 성격이 강하다.

- `/sys/devices/...`
- `/sys/class/...`
- `/sys/bus/...`
- `/sys/module/...`

둘 다 file처럼 보이지만, object lifetime과 callback 규칙이 다르다.

## 작동 흐름

1. user program이 `/proc` 또는 `/sys` path를 open한다.
2. VFS가 pseudo filesystem의 inode와 operation을 찾는다.
3. read는 kernel object 상태를 text 또는 binary output으로 만든다.
4. write는 user buffer를 kernel parser로 넘긴다.
5. parser가 문자열, 숫자, flag, 범위를 검증한다.
6. permission과 namespace 기준을 확인한다.
7. kernel object state를 조회하거나 변경한다.
8. object removal과 open file lifetime을 정리한다.

## 대표 예시

`cat /proc/self/status`는 현재 process 상태를 읽는다. 디스크 파일을 읽는 것이 아니라 kernel이 현재 task 정보를 text로 만들어 돌려준다.

```text
/proc/self/status
    -> current task 해석
    -> procfs callback
    -> task state formatting
    -> user buffer로 copy
```

`echo 1 > /sys/.../enable` 같은 write는 kernel attribute parser를 거쳐 device나 subsystem state를 바꿀 수 있다.

이 예시를 읽을 때는 다음을 확인한다.

- read output은 어느 kernel object에서 만들어지는가?
- write input은 text parser에서 어떤 범위 검증을 거치는가?
- open된 file이 object removal 이후에도 접근 가능한가?

## 핵심 용어

- `procfs`: process와 kernel runtime state를 노출하는 pseudo filesystem.
- `sysfs`: device, driver, kobject hierarchy를 노출하는 pseudo filesystem.
- `seq_file`: 긴 procfs output을 안전하게 순차 출력하기 위한 helper.
- `kobject`: sysfs hierarchy에 연결되는 kernel object model.
- `attribute`: sysfs에서 하나의 값이나 설정을 나타내는 file.
- `sysctl`: `/proc/sys`를 통해 kernel parameter를 읽고 쓰는 interface.

## 다른 큰가지와의 연결

- [6. VFS / FD Model](../6-vfs-fd-model/README.md): procfs/sysfs도 VFS path와 file operation을 사용한다.
- [2. Process / Task](../2-process-task/README.md): procfs는 task state를 많이 노출한다.
- [10. Device Drivers](../10-device-drivers/README.md): sysfs는 device와 driver model을 노출한다.
- [7. Permission Model](../7-permission-model/README.md): write 가능한 entry는 권한과 LSM 정책을 확인해야 한다.

## 헷갈리기 쉬운 부분

- procfs/sysfs file을 실제 disk file로 보는 것
- read path는 안전하고 write path만 위험하다고 생각하는 것
- text parser의 overflow, range, newline 처리를 가볍게 보는 것
- object가 제거된 뒤에도 open file callback이 실행될 수 있다는 점을 놓치는 것

## 보안/취약점 관점

procfs/sysfs bug는 text parser, object lifetime, permission check에서 자주 나온다. file처럼 보이기 때문에 평범해 보이지만, 실제로는 kernel callback entry다.

코드를 읽을 때는 다음 질문을 붙인다.

1. 이 entry는 어떤 kernel object를 노출하는가?
2. read output에 kernel pointer나 민감한 state가 새지 않는가?
3. write parser가 length, range, format을 모두 검증하는가?
4. object removal과 open file lifetime이 충돌하지 않는가?
5. namespace별로 보여야 할 정보와 host-global 정보가 섞이지 않는가?

## 다음에 읽을 문서

- [6. VFS / FD Model](../6-vfs-fd-model/README.md)
- [2. Process / Task](../2-process-task/README.md)
- [10. Device Drivers](../10-device-drivers/README.md)

## 기억할 문장

procfs와 sysfs의 파일은 disk data가 아니라 kernel object를 향한 callback이다.
