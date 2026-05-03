# syscall

## 한 문장 정의

user mode code가 kernel mode service를 요청하기 위해 CPU privilege boundary를 넘는 공식 entry point.

## 핵심 질문

일반 함수 호출처럼 보이는 `read()`, `open()`, `mmap()`은 어떻게 kernel mode 실행으로 바뀌는가?

## 왜 필요한가

user program은 CPU, page table, block device, network stack 같은 권한 있는 자원을 직접 조작할 수 없다. 하지만 파일을 열고, 메모리를 매핑하고, process를 만들고, socket을 사용하려면 커널 기능이 필요하다.

syscall은 이 요청을 정해진 ABI로 커널에 전달하는 가장 기본적인 문이다. 여기서 중요한 것은 함수 이름이 아니라 privilege transition이다.

```text
user mode
    -> syscall instruction
    -> kernel mode
    -> syscall dispatch
    -> kernel object operation
    -> return to user mode
```

## syscall을 볼 때의 중심 구조

syscall은 세 층으로 나누어 읽는다.

- ABI layer: syscall number, register argument, return value 규칙
- dispatch layer: syscall table이 number를 handler로 연결하는 지점
- implementation layer: handler가 fd, task, memory, permission을 실제로 다루는 지점

`read()`라는 이름만 보면 단순 I/O 함수지만, 커널에서는 fd lookup, `struct file`, file operation, user buffer copy가 모두 등장한다.

## 작동 흐름

1. user program이 libc wrapper를 호출하거나 직접 syscall instruction을 실행한다.
2. syscall number와 argument가 architecture ABI에 맞게 register에 놓인다.
3. CPU가 user mode에서 kernel mode로 전환한다.
4. kernel entry code가 현재 task의 syscall context를 준비한다.
5. syscall table이 number에 맞는 handler를 선택한다.
6. handler가 argument를 해석하고 필요한 kernel object를 찾는다.
7. 권한, 범위, 상태를 확인한 뒤 작업을 수행한다.
8. return value 또는 errno 의미를 user space로 돌려준다.

## 대표 예시

`read(fd, buf, size)`는 단순히 buffer에 데이터를 넣는 함수가 아니다.

```text
read(fd, buf, size)
    -> syscall number: read
    -> fd table lookup
    -> struct file
    -> file read operation
    -> copy_to_user(buf)
    -> return bytes or errno
```

이 예시를 읽을 때는 다음을 확인한다.

- syscall argument 중 user-controlled 값은 무엇인가?
- fd, pointer, size가 각각 어떤 kernel object나 memory boundary로 이어지는가?
- handler가 실패할 때 object state를 바꾸기 전에 멈추는가?

## 핵심 용어

- `syscall number`: 어떤 kernel service를 요청하는지 나타내는 번호.
- `syscall ABI`: syscall number와 argument를 register에 배치하고 return value를 해석하는 architecture별 약속.
- `syscall table`: syscall number를 실제 kernel handler로 연결하는 table.
- `syscall handler`: `ksys_*`, `__do_sys_*` 계열처럼 요청을 실제로 처리하는 kernel function.
- `errno`: 실패 이유를 user space가 해석할 수 있게 표현하는 오류 값.
- `restart`: signal 등으로 중단된 syscall을 다시 수행할 수 있게 하는 kernel mechanism.

## 다른 큰가지와의 연결

- [2. Process / Task](../2-process-task/README.md): syscall은 현재 task의 context에서 실행된다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): 많은 syscall은 fd를 `struct file`로 바꾼다.
- [7. Permission Model](../7-permission-model/README.md): privileged syscall은 credential과 capability를 확인한다.
- [3. Memory Management](../3-memory-management/README.md): pointer argument는 user memory boundary와 연결된다.

## 헷갈리기 쉬운 부분

- libc 함수와 kernel syscall handler를 같은 것으로 보는 것
- syscall argument가 모두 안전한 kernel value라고 생각하는 것
- syscall entry만 보고 실제 subsystem callback까지 따라가지 않는 것
- return value와 errno 처리를 대충 보는 것

## 보안/취약점 관점

syscall은 가장 넓은 attack surface다. 위험은 syscall entry 자체보다 handler가 argument를 kernel object와 memory operation으로 바꾸는 과정에서 많이 생긴다.

코드를 읽을 때는 다음 질문을 붙인다.

1. 이 syscall은 unprivileged user가 호출할 수 있는가?
2. argument가 fd, pointer, size, flag 중 무엇으로 해석되는가?
3. argument 검증과 object lookup의 순서는 안전한가?
4. permission check가 상태 변경 전에 수행되는가?
5. return path와 error path에서 reference와 memory가 정리되는가?

## 다음에 읽을 문서

- [copy_from_user / copy_to_user](copy-from-user-copy-to-user.md)
- [ioctl](ioctl.md)
- [6. VFS / FD Model](../6-vfs-fd-model/README.md)

## 기억할 문장

syscall은 user function call이 아니라 CPU 권한 경계를 넘어 kernel object operation으로 들어가는 공식 문이다.
