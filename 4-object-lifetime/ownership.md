# ownership

## 한 문장 정의

ownership은 object를 해제하거나 다음 관리 주체로 넘길 책임이 어느 코드에 있는지 정하는 규칙이다.

## 왜 중요한가

refcount가 “몇 명이 쓰는가”를 숫자로 세는 방식이라면, ownership은 “누가 책임지는가”를 코드 계약으로 정한다. 커널 함수는 object pointer를 받거나 돌려줄 때 이 책임을 함께 이동시킬 수 있다.

ownership을 놓치면 같은 object를 두 경로가 free하거나, 아무도 free하지 않거나, 등록된 object를 caller가 임의로 없애는 문제가 생긴다.

```text
allocate
    -> caller owns object
    -> initialize
    -> register 성공
    -> subsystem owns object
    -> unregister
    -> release
```

## owner의 의미

owner는 단순히 pointer를 가진 코드가 아니다. owner는 object 정리에 대한 책임을 가진 코드다.

owner가 할 수 있는 일은 보통 다음과 같다.

- object를 free한다.
- 다른 subsystem에 소유권을 넘긴다.
- 실패 경로에서 이미 얻은 resource를 되돌린다.
- object를 list, table, idr, xarray, fd table 같은 곳에 등록한다.
- unregister 이후 release가 실행되도록 만든다.

pointer가 여러 곳에 있어도 owner는 하나이거나, 명확한 규칙으로 나뉘어야 한다.

## ownership transfer

함수 호출은 ownership을 바꿀 수 있다. 문서를 읽거나 코드를 볼 때 함수가 어떤 계약을 갖는지 확인해야 한다.

대표 형태는 다음과 같다.

```text
caller keeps ownership
    -> callee는 잠깐 사용만 함
    -> caller가 계속 free 책임을 가짐

callee takes ownership
    -> 성공하면 callee가 object를 관리함
    -> caller는 성공 이후 free하면 안 됨

conditional transfer
    -> 성공하면 callee 소유
    -> 실패하면 caller가 정리
```

conditional transfer가 가장 위험하다. return value에 따라 cleanup 책임이 바뀌기 때문이다.

## registration과 ownership

object를 global table, list, fd table, net namespace, device model에 등록하면 ownership 규칙이 바뀌는 경우가 많다.

예를 들어 object를 table에 넣은 뒤에는 lookup path가 그 object를 찾을 수 있다. 이 상태에서 caller가 바로 free하면 table에는 죽은 pointer가 남는다.

보통 안전한 흐름은 다음과 같다.

```text
object allocation
    -> initialization 완료
    -> table/list 등록
    -> external lookup 가능
    -> unregister로 lookup 차단
    -> 남은 reference 대기
    -> free
```

등록 전, 등록 후, 등록 실패 후의 owner가 각각 누구인지 구분해야 한다.

## borrowed pointer와 ownership

borrowed pointer는 ownership이 아니다. 함수가 pointer를 인자로 받았다고 해서 그 pointer를 free할 수 있는 것은 아니다.

자주 보는 실수는 다음과 같다.

- 인자로 받은 borrowed pointer를 error path에서 free한다.
- 등록 함수가 ownership을 가져갔는데 caller도 free한다.
- helper가 내부 reference를 가져갔다고 생각했지만 실제로는 가져가지 않았다.
- object field에 pointer를 저장하면서 owner 관계를 문서화하지 않는다.

## 코드에서 확인할 것

1. object를 할당한 직후 owner는 누구인가?
2. 함수 성공 시 ownership이 이동하는가?
3. 함수 실패 시 ownership이 caller에게 남는가, callee가 정리하는가?
4. object가 table, list, queue에 등록된 뒤 caller가 free하지 않는가?
5. unregister 이후 남은 reference나 callback을 기다리는가?
6. 함수 이름, 주석, error label이 ownership 계약과 일치하는가?

## 보안 관점

ownership이 흐려지면 cleanup이 깨진다.

- double free: caller와 callee가 모두 같은 object를 free한다.
- leak: 성공 또는 실패 경로에서 owner가 사라진다.
- UAF: 등록된 object를 owner가 먼저 free한다.
- dangling list entry: list에서 제거하지 않고 object memory를 반환한다.
- stale callback: callback owner를 정리하지 않고 object를 해제한다.

## 다른 문서와의 연결

- [refcount](refcount.md): owner와 reference holder를 구분한다.
- [cleanup path](cleanup-path.md): ownership이 바뀌는 실패 경로를 정리한다.
- [async callback lifetime](async-callback-lifetime.md): callback을 등록한 뒤 owner 책임이 어떻게 바뀌는지 본다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fd table, file object, private data ownership
- [10. Device Drivers](../10-device-drivers/README.md): device registration과 remove path

## 기억할 문장

ownership을 읽을 때 핵심은 “이 object를 free할 책임이 지금 누구에게 있는가?”다.
