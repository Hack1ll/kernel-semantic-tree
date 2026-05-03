# cred

## 한 문장 정의

`struct cred`는 task가 커널 안에서 어떤 uid, gid, group, capability, security label을 가진 주체인지 표현하는 credential object다.

## 왜 중요한가

권한 검사는 “누가 요청했는가”에서 시작한다. 리눅스에서 그 주체 정보는 대부분 `task_struct`가 가리키는 `cred`에서 나온다.

```text
task_struct
    -> cred
    -> uid / gid
    -> groups
    -> capabilities
    -> user namespace
    -> LSM security blob
```

`cred`를 이해해야 `current->cred`, `file->f_cred`, `override_creds()` 같은 권한 기준 차이를 읽을 수 있다.

## cred가 담는 신원

`cred`에는 여러 종류의 uid/gid가 들어간다.

- real uid/gid: process의 실제 소유자 의미
- effective uid/gid: 일반 permission check에서 주로 쓰이는 권한 의미
- saved uid/gid: setuid 전환과 복구에 연결되는 값
- fs uid/gid: filesystem permission check에 쓰이는 값
- supplementary groups: group permission 판단에 쓰이는 group 목록

권한 코드를 읽을 때는 “uid가 몇인가?”보다 “어떤 uid field를 보는가?”가 더 중요하다.

## current->cred와 real_cred

task에는 credential pointer가 둘 이상 등장할 수 있다.

- `current->cred`: 현재 권한 판단에 쓰이는 credential
- `current->real_cred`: task의 실제 credential 의미를 보존하는 pointer
- `file->f_cred`: file open 시점의 credential

대부분의 일반 권한 check는 `current_cred()` 경로를 통해 현재 credential을 본다. 하지만 file operation, worker, override credential 경로에서는 기준이 달라질 수 있다.

## cred 변경

credential은 제자리에서 아무렇게나 수정하는 object가 아니다. 보통 새 credential을 준비하고, 필요한 field를 바꾼 뒤 commit하는 형태로 처리한다.

```text
prepare_creds()
    -> 새 cred 준비
    -> uid/gid/capability 변경
    -> commit_creds()
    -> old cred는 reference/RCU 규칙으로 정리
```

실패하면 `abort_creds()` 같은 정리 경로가 필요하다. credential 변경은 권한 경계이므로 중간 상태가 노출되지 않아야 한다.

## override_creds

커널 내부 작업이 일시적으로 다른 credential로 실행되어야 할 때 `override_creds()`와 `revert_creds()`가 쓰일 수 있다.

대표적인 위험은 error path에서 복구를 빠뜨리는 것이다.

```text
old = override_creds(new)
    -> privileged operation
    -> 모든 return path에서 revert_creds(old)
```

임시 credential은 worker나 filesystem helper에서 특히 조심해서 읽어야 한다.

## cred lifetime

credential은 여러 task나 file이 참조할 수 있다. `file->f_cred`는 open 시점 credential을 붙잡고 있을 수 있고, RCU로 읽히는 credential pointer도 있다.

따라서 credential pointer를 저장하거나 RCU section 밖에서 쓰려면 reference와 lifetime 규칙을 확인해야 한다.

## 코드에서 확인할 것

1. 권한 check가 `current->cred`, `real_cred`, `file->f_cred` 중 무엇을 기준으로 하는가?
2. real/effective/fs uid 중 어떤 field가 쓰이는가?
3. credential 변경이 prepare/commit/abort 규칙을 따르는가?
4. `override_creds()` 이후 모든 path에서 `revert_creds()`가 실행되는가?
5. worker나 async path에서 원래 요청자의 credential이 보존되는가?
6. credential pointer를 RCU 보호 범위 밖에서 쓰지 않는가?

## 보안 관점

`cred` 관련 버그는 권한 상승이나 권한 우회로 이어질 수 있다.

- current credential과 opener credential을 혼동한다.
- effective uid 대신 real uid를 잘못 사용한다.
- credential 변경 중 error path가 old/new cred reference를 틀리게 처리한다.
- `override_creds()` 복구를 빠뜨린다.
- worker가 원래 요청자가 아닌 worker 자신의 credential로 판단한다.
- RCU로 읽은 cred pointer를 lifetime 보장 없이 계속 사용한다.

## 다른 문서와의 연결

- [capabilities](capabilities.md): `cred` 안의 capability set
- [user namespace](user-namespace.md): uid/gid와 capability 해석 기준
- [LSM](lsm.md): credential에 붙는 security label과 hook
- [2. Process / Task](../2-process-task/README.md): `task_struct -> cred`
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): `file->f_cred`

## 기억할 문장

`cred`를 읽을 때 핵심은 “이 권한 판단이 정확히 어떤 credential object의 어떤 field를 기준으로 하는가?”다.
