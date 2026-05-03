# Linux Kernel Semantic Tree Wiki

이 위키는 커널의 구조를 단계적으로 이해하기 위한 교과서형 Semantic Tree다. 목표는 세부 함수나 구조체를 먼저 외우는 것이 아니라, 커널의 줄기와 큰가지를 먼저 이해하게 만드는 것이다.

## 먼저 붙잡을 문장

> 리눅스 커널은 user space의 요청을 받아 하드웨어와 커널 객체를 대신 조작하면서, 자원 추상화, 접근 통제, 상태 관리를 수행하는 권한 있는 관리자다.

이 문장만 제대로 이해해도 커널의 큰 구조가 보인다. 커널은 프로그램 대신 위험한 일을 한다. 프로그램은 커널에게 요청하고, 커널은 그 요청이 안전하고 허용되는지 확인한 뒤 내부 객체의 상태를 바꾼다.

## 왜 Semantic Tree로 읽는가

커널 문서를 열면 `task_struct`, `kmalloc()`, `spinlock`, `ioctl`, `sk_buff`, `BPF verifier` 같은 낯선 단어가 한꺼번에 나온다. 이것들은 모두 잎사귀다. 잎사귀부터 외우면 금방 흩어진다.

이 위키는 다음 순서로 읽는다.

```text
줄기   -> 커널은 왜 존재하는가?
큰가지 -> 커널은 어떤 자원을 어떤 규칙으로 관리하는가?
잎사귀 -> 그 규칙이 실제 코드와 API에서 어떻게 구현되는가?
```

예를 들어 `copy_from_user()`는 그냥 함수 이름이 아니다. user memory에서 kernel memory로 값이 들어오는 경계다. 이 경계는 User/Kernel Boundary 큰가지에 붙고, 다시 "커널은 위험한 자원을 대신 관리한다"는 줄기에 붙는다.

## 줄기: 커널이 하는 세 가지 일

1. 자원을 추상화한다.
   CPU, 메모리, 디스크, 네트워크 카드, 장치는 모두 다르게 생겼다. 커널은 이것들을 process, virtual memory, file, socket, device 같은 형태로 감싼다.

2. 접근을 통제한다.
   아무 프로그램이나 디스크를 지우거나 네트워크 설정을 바꾸면 안 된다. 커널은 uid, credential, capability, namespace, LSM 정책을 이용해 요청을 허용할지 판단한다.

3. 상태를 안전하게 유지한다.
   커널 객체는 생성되고, 참조되고, 공유되고, 해제된다. 여러 CPU와 비동기 callback이 동시에 같은 객체를 만질 수 있으므로 lifetime과 concurrency 규칙이 필요하다.

## 큰가지 지도

- [0. Kernel의 본질](0-kernel-essence/README.md): 커널은 왜 존재하고, 어떤 일을 대신 해주는가?
- [1. User/Kernel Boundary](1-user-kernel-boundary/README.md): user space 요청은 어떤 문을 통해 kernel space로 들어오는가?
- [2. Process / Task](2-process-task/README.md): 커널은 실행 중인 프로그램과 요청 주체를 어떻게 표현하는가?
- [3. Memory Management](3-memory-management/README.md): 커널은 메모리를 어떻게 나누고 보호하고 재사용하는가?
- [4. Object Lifetime](4-object-lifetime/README.md): 커널 객체는 언제 태어나고, 누가 들고 있고, 언제 죽는가?
- [5. Concurrency](5-concurrency/README.md): 여러 실행 흐름이 같은 커널 상태를 동시에 만지면 어떻게 안전하게 만드는가?
- [6. VFS / FD Model](6-vfs-fd-model/README.md): 리눅스는 왜 많은 것을 파일처럼 다루는가?
- [7. Permission Model](7-permission-model/README.md): 커널은 누가 무엇을 할 수 있는지 어떻게 판단하는가?
- [8. Namespace / Isolation](8-namespace-isolation/README.md): 같은 커널을 공유하면서 어떻게 서로 다른 세계처럼 보이게 하는가?
- [9. Networking](9-networking/README.md): packet과 socket과 network policy는 커널 안에서 어떻게 만나는가?
- [10. Device Drivers](10-device-drivers/README.md): 커널은 제각각 다른 하드웨어를 어떻게 안전한 인터페이스로 노출하는가?
- [11. eBPF](11-ebpf/README.md): user가 준 작은 프로그램을 커널 안에서 어떻게 안전하게 실행하는가?
- [12. io_uring](12-io-uring/README.md): 비동기 I/O 요청은 제출 후 완료될 때까지 무엇을 붙잡고 있어야 하는가?
- [13. Debugging / Testing](13-debugging-testing/README.md): 내가 이해한 커널 흐름이 실제 실행에서도 맞는지 어떻게 확인하는가?

## 큰가지 사이의 의존 관계

```text
Kernel의 본질
    -> User/Kernel Boundary
        -> Process / Task
            -> VFS / FD Model
            -> Memory Management
            -> Permission Model

Memory Management
    -> Object Lifetime
        -> Concurrency

Permission Model
    -> Namespace / Isolation

VFS + Memory + Lifetime + Concurrency + Permission
    -> Networking
    -> Device Drivers
    -> eBPF
    -> io_uring

모든 개념
    -> Debugging / Testing
```

전문 영역은 갑자기 따로 생긴 것이 아니다. nftables는 netlink, object lifetime, packet path concurrency, namespace permission이 합쳐진 영역이다. io_uring은 fd, memory buffer, request lifetime, worker concurrency가 합쳐진 영역이다.

## 반복해서 사용할 대표 예시

- `open/read/write`: syscall, fd, VFS, permission, user/kernel boundary를 이해하는 기본 예시다.
- `mmap`: memory management, VFS, driver boundary를 연결하는 예시다.
- `socket`: networking, fd model, namespace를 연결하는 예시다.
- `ioctl`: user input validation과 device driver를 연결하는 예시다.
- `fork/exec`: task, memory, credential의 관계를 보여주는 예시다.
- container: namespace, permission, isolation을 이해하는 예시다.

## 추천 학습 순서

1. [0. Kernel의 본질](0-kernel-essence/README.md)에서 커널이 왜 필요한지 먼저 잡는다.
2. [1. User/Kernel Boundary](1-user-kernel-boundary/README.md)에서 user program이 커널에 들어오는 문을 이해한다.
3. [2. Process / Task](2-process-task/README.md)에서 요청 주체가 어떻게 표현되는지 본다.
4. [6. VFS / FD Model](6-vfs-fd-model/README.md)에서 fd가 왜 커널 객체 handle인지 본다.
5. [3. Memory Management](3-memory-management/README.md)에서 주소 공간, page, slab object를 이해한다.
6. [4. Object Lifetime](4-object-lifetime/README.md)와 [5. Concurrency](5-concurrency/README.md)에서 커널 상태가 언제 깨지는지 본다.
7. [7. Permission Model](7-permission-model/README.md)과 [8. Namespace / Isolation](8-namespace-isolation/README.md)에서 권한과 격리를 이해한다.
8. [9. Networking](9-networking/README.md), [10. Device Drivers](10-device-drivers/README.md), [11. eBPF](11-ebpf/README.md), [12. io_uring](12-io-uring/README.md)을 앞의 큰가지 조합으로 읽는다.
9. [13. Debugging / Testing](13-debugging-testing/README.md)에서 실제 커널로 확인한다.

## 용어집

반복해서 나오는 용어는 [glossary.md](glossary.md)에 따로 정리했다. 용어를 한 번에 완벽히 외우려 하지 말고, 문서를 읽다가 막히는 단어가 나오면 용어집으로 돌아가면 된다.

## 모든 문서에 적용할 7가지 질문

1. 이 개념은 어떤 자원을 추상화하는가?
2. user space에서 이 개념에 어떻게 도달하는가?
3. 여기서 만들어지거나 바뀌는 kernel object는 무엇인가?
4. 그 object의 owner와 lifetime은 무엇인가?
5. 어떤 lock, refcount, RCU 규칙으로 보호되는가?
6. 어떤 permission 또는 namespace 경계를 통과하는가?
7. 실패 경로, 취소 경로, 비동기 경로에서도 같은 규칙이 유지되는가?

## 읽을 때의 기준

모든 세부를 한 번에 이해할 필요는 없다. 먼저 각 문서의 `한 문장 정의`와 `왜 필요한가` 또는 `왜 중요한가`를 읽고, 그 개념이 어느 큰가지에 붙는지 잡는다. 그다음 `코드에서 확인할 것`, `보안 관점`, `기억할 문장`을 읽으면 실제 커널 코드와 취약점 분석으로 연결하기 쉬워진다.
