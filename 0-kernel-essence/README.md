# 0. Kernel의 본질

## 핵심 질문

커널은 왜 존재하고, user program 대신 무엇을 관리하는가?

## 줄기

커널은 여러 프로그램이 공유하는 하드웨어 자원을 안전하게 나눠 쓰게 만드는 권한 있는 상태 관리자다.

이 장은 커널 전체의 줄기를 다룬다. 세부 subsystem을 보기 전에 커널이 반복해서 수행하는 세 가지 일을 먼저 잡는다.

```text
자원을 object로 감싼다.
요청 주체가 그 object에 접근해도 되는지 판단한다.
object 상태가 깨지지 않게 변경하고 정리한다.
```

## 세 축

- [자원 추상화](resource-abstraction.md): 실제 하드웨어와 내부 상태를 process, file, socket, page 같은 object로 바꾸는 일
- [접근 통제](access-control.md): 누가 어떤 object에 어떤 action을 해도 되는지 판단하는 일
- [상태 관리](state-management.md): object가 생성, 변경, 공유, 해제되는 동안 규칙을 유지하는 일

## 이 장을 읽는 관점

커널 기능을 볼 때마다 다음 문장으로 되돌아온다.

```text
user request
    -> kernel object
    -> permission decision
    -> state transition
    -> lifetime / cleanup
```

`read()`, `mmap()`, `socket()`, `bpf()`, `io_uring_enter()`는 모두 다른 기능처럼 보이지만, 위 구조를 공유한다.

## 하위 문서의 역할

`자원 추상화`는 “무엇을 감싸서 보여주는가”를 설명한다.  
`접근 통제`는 “누가 그것을 사용할 수 있는가”를 설명한다.  
`상태 관리`는 “사용 후에도 object가 일관된가”를 설명한다.

셋 중 하나만 빠져도 커널은 안전한 관리자 역할을 하지 못한다.

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): user request가 커널로 들어오는 입구
- [3. Memory Management](../3-memory-management/README.md): memory resource를 page, VMA, slab object로 감싸는 방식
- [4. Object Lifetime](../4-object-lifetime/README.md): 상태 관리 중 object가 언제 죽어도 되는지에 대한 세부 규칙
- [7. Permission Model](../7-permission-model/README.md): 접근 통제의 실제 구현 계층

## 보안 관점

커널 취약점은 대개 이 세 축 중 하나가 깨진 결과다.

- 추상화 실패: 잘못된 object type 접근, kernel pointer leak, type confusion
- 접근 통제 실패: capability check 누락, namespace 기준 오류, LSM hook 누락
- 상태 관리 실패: UAF, double free, stale state, incomplete rollback

## 읽고 나서 확인할 것

1. 커널을 “권한 있는 상태 관리자”라고 설명할 수 있는가?
2. 자원 추상화, 접근 통제, 상태 관리가 서로 어떻게 다른지 구분할 수 있는가?
3. 임의의 syscall 하나를 골라 object, permission, state transition으로 나눠 볼 수 있는가?
