# refcount

## 한 문장 정의

refcount는 object를 현재 몇 경로가 사용 중인지 숫자로 추적하고, 마지막 사용자가 떠날 때 release 함수를 실행하게 하는 수명 관리 방식이다.

## 왜 중요한가

커널 object는 여러 fd, list entry, request, socket, worker가 동시에 가리킬 수 있다. pointer를 가진 코드가 많아질수록 “누가 마지막인가”를 단순히 코드 흐름만으로 판단하기 어렵다.

refcount는 이 문제를 명시적인 get/put 규칙으로 다룬다.

```text
object 생성
    -> refcount = 1
    -> 새 사용자 get
    -> 사용 완료 put
    -> 마지막 put
    -> release
    -> memory free 또는 subsystem 정리
```

## get과 put

refcount의 기본 규칙은 단순하다.

- object를 오래 쓰려면 get으로 reference를 얻는다.
- reference를 얻은 경로는 반드시 put으로 반납한다.
- 마지막 put이 release path를 호출한다.
- refcount가 0이 된 object는 다시 살리면 안 된다.

“오래 쓴다”는 말은 함수 호출 하나를 넘어서 저장하거나, lock을 풀고도 사용하거나, callback에 넘기거나, 다른 thread가 접근할 수 있게 만든다는 뜻이다.

## borrowed reference와 owned reference

코드를 읽을 때는 reference의 성격을 구분해야 한다.

```text
borrowed reference
    -> 잠깐 빌려 본 pointer
    -> 별도 put 책임 없음
    -> 보호 조건이 사라지면 사용 불가

owned reference
    -> get으로 얻은 reference
    -> 반드시 put 책임이 생김
    -> lock 밖이나 나중 시점에서도 사용 가능
```

많은 버그는 borrowed reference를 owned reference처럼 저장하거나, owned reference를 put하지 않는 데서 나온다.

## 마지막 put과 release

마지막 put은 단순히 숫자를 줄이는 동작이 아니다. object의 실제 정리 함수가 실행되는 지점이다.

release path에서는 보통 다음 일이 일어난다.

- 내부 buffer를 해제한다.
- list나 table에 남은 연결이 없는지 확인한다.
- file, cred, page 같은 다른 object reference를 put한다.
- RCU callback으로 실제 free를 미룰 수 있다.
- slab allocator에 memory를 반환한다.

따라서 마지막 put이 어느 context에서 실행될 수 있는지 봐야 한다. sleep 가능한 release인지, lock을 잡은 상태에서 호출되는지에 따라 안전성이 달라진다.

## refcount_t, kref, atomic_t

커널에는 여러 reference 관리 형태가 있다.

- `refcount_t`: overflow와 underflow 방어를 의식한 reference counter
- `kref`: reference count와 release callback을 묶은 helper
- `atomic_t`: 단순 atomic counter로 쓰이기도 하지만 lifetime 용도에는 더 조심해야 함
- subsystem 전용 get/put: `get_file()`, `fput()`, `get_task_struct()`, `put_task_struct()` 같은 API

API 이름이 다르더라도 질문은 같다. “이 get은 누가 put하는가?”

## 0에서 증가시키는 문제

이미 refcount가 0인 object는 release가 시작되었거나 끝난 상태다. 이 object에 다시 get을 걸면 해제 중인 object를 되살리는 문제가 생길 수 있다.

그래서 lookup 경로에서는 `refcount_inc_not_zero()` 같은 형태가 필요할 수 있다. pointer를 찾았다는 사실과 reference 획득에 성공했다는 사실을 구분해야 한다.

## 코드에서 확인할 것

1. 이 pointer는 borrowed reference인가, owned reference인가?
2. get을 호출한 모든 경로에 대응하는 put이 있는가?
3. error path와 early return에서도 put이 실행되는가?
4. refcount가 0인 object를 다시 증가시킬 가능성이 있는가?
5. 마지막 put의 release 함수가 어떤 lock/context에서 실행되는가?
6. release 함수가 다른 object reference를 올바른 순서로 내려놓는가?

## 보안 관점

refcount 버그는 object lifetime을 직접 깨뜨린다.

- missing get: 다른 경로가 object를 free한 뒤 stale pointer를 사용한다.
- missing put: object와 내부 resource가 leak된다.
- double put: refcount가 너무 빨리 0이 되어 UAF가 생긴다.
- refcount underflow: 0 아래로 내려가 release가 반복되거나 상태가 깨진다.
- refcount overflow: 매우 큰 값으로 wrap되어 lifetime 보장이 무너질 수 있다.
- resurrected object: 이미 0이 된 object를 lookup 경로가 다시 사용한다.

## 다른 문서와의 연결

- [ownership](ownership.md): reference를 얻은 경로가 해제 책임도 갖는지 구분한다.
- [cleanup path](cleanup-path.md): 실패 경로에서 get/put 짝을 맞춘다.
- [RCU](rcu.md): pointer lookup과 free 지연이 refcount와 함께 쓰일 수 있다.
- [3. Memory Management](../3-memory-management/README.md): 마지막 put 이후 memory는 재사용될 수 있다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): `struct file` reference와 fd close

## 기억할 문장

refcount를 읽을 때 가장 중요한 질문은 “누가 get했고, 정확히 어느 경로에서 put하는가?”다.
