# 12. io_uring

## 핵심 질문

비동기 I/O request는 제출된 뒤 완료되기 전까지 무엇을 붙잡고 있어야 하는가?

## 큰가지의 의미

io_uring은 syscall 횟수를 줄이고 비동기 I/O를 빠르게 처리하기 위한 subsystem이다. 하지만 성능을 위해 queue, registered buffer, fixed file, worker, cancellation을 연결하면서 lifetime과 concurrency가 복잡해진다.

```text
SQE
    -> io_kiocb request
    -> file / buffer / cred reference
    -> issue or worker
    -> completion or cancellation
    -> CQE
```

## 하위 문서의 역할

- [SQ / CQ](sq-cq.md): user와 kernel이 공유하는 submission/completion queue
- [request lifetime](request-lifetime.md): `io_kiocb`가 생성, 실행, 취소, 완료, 해제되는 규칙
- [registered buffers](registered-buffers.md): user memory를 미리 pin하고 request에서 재사용하는 기능
- [fixed files](fixed-files.md): `struct file` reference를 ring에 미리 등록하는 기능
- [workers / cancellation](workers-cancellation.md): blocking 작업을 worker로 넘기고 진행 중 request를 취소하는 경로

## 이 장에서 특히 구분할 것

submit syscall이 끝났다고 request가 끝난 것은 아니다. request는 worker, timeout, linked request, completion path에서 계속 살아 있을 수 있다.

```text
submit path
worker path
cancel path
completion path
teardown path
```

이 경로들이 같은 request를 만진다.

## 대표 흐름

```text
read SQE
    -> io_kiocb 생성
    -> file ref 잡기
    -> buffer 확인
    -> worker로 이동 가능
    -> CQE 기록
    -> request put
```

```text
timeout
    -> target request lookup
    -> cancel 시도
    -> completion과 경쟁
```

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): SQE는 user-provided request description이다.
- [4. Object Lifetime](../4-object-lifetime/README.md): request, file, buffer reference가 핵심이다.
- [5. Concurrency](../5-concurrency/README.md): cancel과 completion이 같은 request를 두고 경쟁한다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fixed file은 fd lookup 대신 `struct file` reference를 사용한다.

## 보안 관점

io_uring 취약점은 lifetime과 double completion에서 자주 나온다.

- request UAF
- cancel/complete race
- fixed file reference leak
- registered buffer unregister와 I/O race
- worker credential 혼동
- CQE 중복 기록

## 읽고 나서 확인할 것

1. request가 잡는 file, buffer, cred reference는 무엇인가?
2. request는 어느 경로에서 완료될 수 있는가?
3. cancel과 completion이 동시에 일어나면 누가 이기는가?
4. buffer와 fixed file은 unregister 중에도 안전한가?
5. CQE는 정확히 한 번만 기록되는가?
