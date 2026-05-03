# 자원 추상화

## 한 문장 정의

서로 다른 하드웨어와 커널 내부 상태를 process, file, socket, page 같은 일관된 kernel object로 감싸는 일.

## 왜 필요한가

하드웨어는 제각각 다르다. CPU, RAM, SSD, NIC, GPU, USB 장치는 동작 방식도 다르고 실패 방식도 다르다. user program이 이런 자원을 직접 만지면 프로그램마다 하드웨어 세부를 알아야 하고, 한 프로그램의 실수가 시스템 전체를 망가뜨릴 수 있다.

커널은 이 복잡한 자원을 직접 노출하지 않는다. 대신 자원을 kernel object로 감싸고, user space에는 syscall, fd, address, socket 같은 제한된 손잡이만 준다.

자원 추상화를 이해한다는 것은 “이 기능 뒤에 실제로 어떤 자원이 있고, 커널은 그것을 어떤 object로 바꿔 보여주는가?”를 묻는 것이다.

## 쉬운 설명

자원 추상화는 번역이다.

```text
실제 자원
    -> kernel object
    -> 제한된 operation
    -> user-visible handle
```

예를 들어 디스크 위 파일은 단순한 byte 배열이 아니다. 커널은 path lookup으로 `dentry`와 `inode`를 찾고, open instance인 `struct file`을 만든 뒤, user space에는 작은 정수 fd를 돌려준다. 프로그램은 fd만 보지만 커널 안에는 여러 object와 규칙이 붙어 있다.

다른 자원도 같은 방식으로 바뀐다.

- CPU 시간 -> `task_struct`, scheduler state
- 물리 메모리 -> `struct page`, VMA, slab object
- 파일시스템 -> `inode`, `dentry`, `struct file`
- 네트워크 통신 -> socket, `struct sock`, `sk_buff`
- 장치 -> device object, `file_operations`, ioctl command

## 작동 흐름

1. 커널이 실제 자원의 종류와 제약을 파악한다.
2. 그 자원을 표현할 kernel object를 정의한다.
3. object에 허용되는 operation을 정한다.
4. user space에는 직접 포인터가 아니라 fd, address, syscall 같은 handle을 제공한다.
5. 요청이 들어오면 handle을 kernel object로 변환한다.
6. object의 상태와 operation 규칙에 따라 실제 자원을 조작한다.
7. 하드웨어 세부 대신 성공값, 오류값, event 같은 안정된 의미를 user space에 돌려준다.

## 대표 예시

`open("/tmp/a", O_RDONLY)`는 path 문자열을 열린 file object로 바꾸는 추상화다.

```text
"/tmp/a"
    -> path lookup
    -> dentry / inode
    -> struct file
    -> fd
```

프로그램은 fd만 받지만, 커널은 그 뒤에서 filesystem object, open flag, file position, credential, file operation을 함께 관리한다.

이 예시를 읽을 때는 다음을 확인한다.

- user가 보는 값은 무엇이고, 커널 내부 object는 무엇인가?
- object가 실제 하드웨어 또는 subsystem을 어떻게 감싸는가?
- 이 추상화가 새면 user space가 어떤 내부 상태에 의존하게 되는가?

## 핵심 용어

- `resource`: CPU, memory, disk, network device처럼 공유되고 보호되어야 하는 대상.
- `kernel object`: 커널이 resource 상태를 표현하기 위해 만든 구조체.
- `handle`: user space가 kernel object를 직접 포인터가 아니라 fd, address, id 같은 값으로 가리키는 방식.
- `operation`: object에 허용된 동작. file이면 read/write/ioctl/mmap 같은 함수가 된다.
- `abstraction leak`: 감춰야 할 내부 구현이나 상태가 user space에 새는 상황.

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): handle이 syscall을 통해 kernel object로 바뀐다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fd는 자원 추상화의 대표적인 user-visible handle이다.
- [3. Memory Management](../3-memory-management/README.md): 물리 메모리는 page, VMA, slab object로 추상화된다.
- [10. Device Drivers](../10-device-drivers/README.md): 하드웨어는 driver object와 file operation으로 감싸진다.

## 헷갈리기 쉬운 부분

- fd, pid, address 같은 handle을 실제 object 자체로 보는 것
- kernel object를 단순 data structure로만 보고 그 뒤의 하드웨어/자원 의미를 놓치는 것
- 추상화가 많을수록 안전하다고 생각하는 것

## 보안/취약점 관점

자원 추상화가 깨지면 user space가 커널 내부 object를 잘못 선택하거나, 다른 type의 object를 같은 것처럼 다루거나, 감춰야 할 kernel address와 상태를 보게 된다.

코드를 읽을 때는 다음 질문을 붙인다.

1. user space가 받은 handle은 어떤 kernel object로 변환되는가?
2. 그 object type은 모든 경로에서 동일하게 보장되는가?
3. user가 object 내부 상태를 직접 추측하거나 조작할 수 있는가?
4. object가 실제 hardware resource와 연결되는 지점은 어디인가?
5. 오류 경로에서도 같은 추상화 경계가 유지되는가?

## 다음에 읽을 문서

- [접근 통제](access-control.md)
- [상태 관리](state-management.md)
- [6. VFS / FD Model](../6-vfs-fd-model/README.md)
- [3. Memory Management](../3-memory-management/README.md)

## 기억할 문장

자원 추상화는 위험하고 제각각인 하드웨어를 커널이 통제 가능한 object와 operation으로 바꾸는 일이다.
