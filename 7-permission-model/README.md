# 7. Permission Model

## 핵심 질문

커널은 누가 무엇을 할 수 있는지 어떤 기준으로 판단하는가?

## 큰가지의 의미

리눅스 권한은 uid 하나로 결정되지 않는다. 요청 주체의 credential, capability, user namespace, file credential, LSM, seccomp가 함께 작동한다.

이 장은 접근 통제의 실제 구성요소를 분해한다.

```text
subject
    -> cred
    -> capability
    -> namespace 기준
    -> LSM policy
    -> allow / deny
```

## 하위 문서의 역할

- [cred](cred.md): task의 uid, gid, capability 같은 권한 상태
- [capabilities](capabilities.md): root 권한을 여러 세부 권한으로 나눈 모델
- [user namespace](user-namespace.md): uid/gid와 capability 해석 기준을 분리하는 namespace
- [LSM](lsm.md): SELinux/AppArmor 같은 추가 보안 정책 hook
- [seccomp](seccomp.md): process가 사용할 syscall 자체를 제한하는 sandboxing 메커니즘

## 이 장에서 특히 구분할 것

`current->cred`와 `file->f_cred`는 다르다.  
`capable()`과 `ns_capable()`도 다르다.  
host root와 user namespace 내부 root도 다르다.

권한 코드를 읽을 때는 항상 세 가지를 분리한다.

```text
subject: 누가 요청했는가?
object: 무엇을 대상으로 하는가?
action: 어떤 작업을 하려는가?
```

## 대표 흐름

```text
mount request
    -> current credential
    -> namespace 기준 capability 확인
    -> LSM hook
    -> mount tree 변경
```

```text
file operation
    -> open 시점 f_cred
    -> current cred
    -> operation별 permission 판단
```

## 다른 큰가지와의 연결

- [0. Kernel의 본질](../0-kernel-essence/README.md): 접근 통제는 커널의 세 축 중 하나다.
- [2. Process / Task](../2-process-task/README.md): task가 credential을 들고 있다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): file은 open 시점 credential을 보관할 수 있다.
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): namespace는 권한 기준을 바꾼다.

## 보안 관점

권한 모델 취약점은 기능 bug가 아니라 보안 경계 붕괴로 이어진다.

- capability check 누락
- wrong namespace 기준
- `current`와 opener credential 혼동
- override_creds 후 revert 누락
- LSM hook 누락
- seccomp로 막았다고 생각했지만 다른 syscall 조합으로 우회

## 읽고 나서 확인할 것

1. 이 operation에 필요한 권한은 무엇인가?
2. 권한 기준은 current cred인가 file cred인가?
3. capability는 어느 user namespace 기준인가?
4. LSM hook이 상태 변경 전에 호출되는가?
5. worker나 async path에서 권한 기준이 유지되는가?
