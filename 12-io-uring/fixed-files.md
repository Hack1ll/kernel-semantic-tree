# fixed files

## 한 문장 정의

`fixed files`는 request마다 fd table lookup을 하지 않도록, io_uring context에 미리 등록해 둔 `struct file` reference 집합이다.

## 왜 중요한가

일반 I/O request는 SQE에 들어 있는 fd를 기준으로 현재 task의 fd table에서 `struct file`을 찾는다. fixed file을 사용하면 ring context의 file table에서 slot index로 file을 가져온다.

이 문서의 줄기는 이것이다.

```text
fixed file은 fd 번호가 아니라, 미리 잡아 둔 struct file reference를 request에서 재사용하는 기능이다.
```

따라서 fixed file을 읽을 때는 fd 값보다 file reference와 slot lifetime을 봐야 한다.

## 등록 흐름

대표 등록 경로는 다음과 같다.

```text
io_uring_register(IORING_REGISTER_FILES)
    -> user fd 배열 복사
    -> 각 fd를 struct file로 변환
    -> file reference 획득
    -> ring context의 file table slot에 저장
```

이후 request는 SQE flag로 fixed file 사용을 표시하고, fd field를 slot index처럼 사용할 수 있다.

```text
SQE uses fixed file
    -> slot index 확인
    -> registered file lookup
    -> request가 file reference 사용
    -> completion 후 request cleanup
```

## fd와 fixed file의 차이

fd와 fixed file slot은 같은 개념이 아니다.

구분은 다음과 같다.

- fd: process의 fd table 안에서 `struct file`을 찾는 번호
- fixed file slot: io_uring context 안에서 등록된 `struct file`을 찾는 index
- fd close: process fd table에서 file reference를 내려놓음
- fixed file unregister: ring file table에서 file reference를 내려놓음

user가 원래 fd를 close해도 fixed file slot이 file reference를 잡고 있으면 file은 계속 살아 있을 수 있다.

## file reference

fixed file의 핵심은 `struct file` lifetime이다.

확인할 규칙은 다음이다.

- register 시점에 file reference를 증가시키는가?
- request가 slot에서 file을 가져올 때 필요한 reference가 보장되는가?
- unregister 또는 update가 in-flight request의 file을 해제하지 않는가?
- ring exit 중 모든 file reference가 내려가는가?
- sparse slot이나 empty slot 접근이 안전하게 실패하는가?

fixed file은 [6. VFS / FD Model](../6-vfs-fd-model/README.md)의 `struct file` 문서와 직접 연결된다.

## update와 sparse file table

io_uring은 등록된 file table을 갱신할 수 있다. 일부 slot은 비어 있을 수 있고, 나중에 채워질 수 있다.

위험한 흐름은 다음이다.

```text
request A가 slot 5의 file 사용 중
    -> user가 slot 5를 다른 file로 update
    -> request A는 기존 file로 완료되어야 함
    -> 새 request는 새 file을 보아야 함
```

slot 갱신은 table entry의 교체일 뿐, 이미 file을 잡은 request의 의미를 바꾸면 안 된다.

## credentials와 권한

fixed file은 등록 시점의 fd를 통해 file을 가져온다. 하지만 request 실행은 나중에, 다른 worker context에서 일어날 수 있다.

구분해야 할 질문은 다음이다.

- file open 시점 권한은 이미 `struct file`에 반영되어 있는가?
- request opcode가 추가 권한 검사를 요구하는가?
- worker가 어떤 credentials로 실행되는가?
- fixed file로 등록된 file을 다른 task가 공유하는 ring에서 사용할 수 있는가?
- file의 `f_cred`와 current cred 중 어느 것을 기준으로 삼는가?

모든 작업이 open 시점 권한만으로 충분한 것은 아니다. opcode별 permission check를 확인해야 한다.

## file type과 operation

fixed file slot에는 regular file, socket, pipe, eventfd 등 다양한 file object가 들어갈 수 있다. request opcode마다 허용되는 file type이 다르다.

확인할 항목은 다음이다.

- read/write가 가능한 file인가?
- poll 가능한 file인가?
- socket 전용 opcode에 regular file이 들어오면 거부되는가?
- file operation이 async path에서 안전한가?
- blocking 가능성이 있으면 worker로 이동하는가?

fixed file은 lookup 비용을 줄여줄 뿐, file type 검증을 생략하게 해주는 기능이 아니다.

## 코드에서 확인할 것

fixed file 코드를 읽을 때는 다음을 본다.

- register path에서 fd 배열을 복사하는 위치
- fd를 `struct file`로 변환하고 reference를 잡는 지점
- file table slot의 상태 표현
- request가 fixed file flag를 해석하는 위치
- update/unregister와 in-flight request 동기화
- ring exit에서 file table을 정리하는 경로
- opcode별 file type 및 permission check

## 보안 관점

fixed file 관련 취약점은 주로 slot lifetime과 권한 의미에서 나온다.

- slot index bounds check 누락
- unregister/update 중 in-flight request의 file UAF
- file reference leak
- empty slot을 valid file처럼 사용하는 문제
- fd와 slot index 의미 혼동
- worker credentials와 file permission 기준 혼동
- file type check 누락으로 잘못된 file operation 호출

검토할 질문은 다음과 같다.

1. SQE의 fd field가 fixed file mode에서 slot index로만 해석되는가?
2. slot이 교체되어도 이미 진행 중인 request의 file reference는 안전한가?
3. register 실패 중간 경로에서 이미 잡은 file reference가 모두 정리되는가?
4. opcode가 요구하는 file type과 permission을 별도로 확인하는가?
5. ring teardown 중 file table과 pending request의 순서가 맞는가?

## 다른 문서와의 연결

- [request lifetime](request-lifetime.md): request가 file reference를 언제 얻고 언제 내려놓는지 이어진다.
- [workers / cancellation](workers-cancellation.md): worker가 fixed file request를 실행할 때 cred와 file lifetime이 중요하다.
- [SQ / CQ](sq-cq.md): SQE flag와 fd field가 fixed file 사용 여부를 전달한다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fixed file의 본체는 `struct file`이다.

## 기억할 문장

fixed file은 fd lookup을 줄이는 기능이지만, 보안 관점의 핵심은 slot index가 가리키는 `struct file` reference가 언제까지 살아 있는가다.
