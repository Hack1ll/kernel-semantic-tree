# 1. User/Kernel Boundary

## 핵심 질문

신뢰할 수 없는 user input은 어떤 문을 통해 권한 있는 kernel state로 바뀌는가?

## 큰가지의 의미

User/Kernel Boundary는 커널의 가장 중요한 신뢰 경계다. user program은 커널 object를 직접 만지지 않는다. syscall, ioctl, netlink, procfs/sysfs 같은 입구를 통해 요청하고, 커널은 그 값을 복사하고 검증한 뒤 내부 object를 조작한다.

이 장의 핵심은 “어떤 API인가”보다 “언제 user 값이 kernel 값이 되는가”다.

```text
user value
    -> entry point
    -> copy / parse
    -> validation
    -> object lookup
    -> permission check
    -> state change
```

## 하위 문서의 역할

- [syscall](syscall.md): user mode에서 kernel mode로 들어오는 기본 입구
- [ioctl](ioctl.md): fd 뒤의 subsystem에 command와 구조체를 전달하는 확장 입구
- [netlink](netlink.md): 복잡한 설정과 object 생성을 message로 표현하는 control path
- [procfs / sysfs](procfs-sysfs.md): 커널 상태를 file read/write처럼 노출하는 pseudo filesystem
- [copy_from_user / copy_to_user](copy-from-user-copy-to-user.md): user memory와 kernel memory 사이의 복사 규칙

## 이 장에서 특히 구분할 것

`syscall`은 입구다.  
`copy_from_user()`는 user memory를 kernel buffer로 가져오는 경계다.  
`ioctl`과 `netlink`는 user가 복잡한 구조를 커널 object 변경으로 표현하는 방식이다.  
`procfs/sysfs`는 파일처럼 보이지만 실제로는 kernel callback이다.

## 대표 흐름

```text
read(fd, buf, size)
    -> syscall entry
    -> fd lookup
    -> struct file
    -> file operation
    -> copy_to_user(buf)
```

```text
nft add rule
    -> netlink message
    -> attribute parser
    -> capability check
    -> nftables object transaction
```

## 다른 큰가지와의 연결

- [2. Process / Task](../2-process-task/README.md): 요청 주체는 보통 `current` task다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fd는 user-visible handle이고 `struct file`로 변환된다.
- [7. Permission Model](../7-permission-model/README.md): boundary를 통과한 요청은 권한 검사를 거친다.
- [3. Memory Management](../3-memory-management/README.md): user pointer와 kernel pointer는 같은 의미가 아니다.

## 보안 관점

이 장에서 취약점은 주로 user input이 충분히 검증되지 않은 채 kernel object를 바꿀 때 생긴다.

- user pointer 직접 역참조
- size, offset, flag 검증 누락
- copy 후 user memory를 다시 믿는 TOCTOU
- ioctl command별 구조체 크기 혼동
- netlink attribute length 또는 nesting 검증 오류

## 읽고 나서 확인할 것

1. 이 요청은 어떤 boundary entry로 들어오는가?
2. user-provided 값은 언제 kernel buffer로 복사되는가?
3. object lookup과 permission check는 어떤 순서로 일어나는가?
4. 실패한 copy, parse, validation은 상태 변경 전에 멈추는가?
