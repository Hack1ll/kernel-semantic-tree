# 접근 통제

## 한 문장 정의

요청한 주체가 특정 kernel object에 특정 작업을 해도 되는지 판단하는 일.

## 왜 필요한가

커널은 user program 대신 권한 있는 일을 수행한다. 파일 삭제, 네트워크 설정 변경, mount, BPF program load, device ioctl 같은 작업은 커널이 실행하면 시스템 전체에 영향을 준다.

그래서 커널은 요청을 받으면 바로 실행하지 않는다. 먼저 세 가지를 분리한다.

```text
누가 요청했는가?      -> subject
무엇을 대상으로 하는가? -> object
무슨 작업을 하려는가?  -> action
```

접근 통제는 이 세 가지를 기준으로 요청을 허용하거나 거부하는 규칙이다.

## 쉬운 설명

자원 추상화가 “무엇을 object로 보여줄 것인가”를 정한다면, 접근 통제는 “그 object에 누가 무엇을 할 수 있는가”를 정한다.

권한 판단은 uid 하나로 끝나지 않는다. 리눅스에서는 여러 층이 함께 작동한다.

- `cred`: 현재 task의 uid, gid, capability 상태
- `file->f_cred`: file을 open한 시점의 credential
- `capability`: root 권한을 나눈 세부 권한
- `user namespace`: capability를 해석하는 기준
- `LSM`: SELinux/AppArmor 같은 추가 정책
- `seccomp`: syscall 자체를 제한하는 필터

중요한 점은 권한 검사가 상태 변경 전에 끝나야 한다는 것이다. 거부해야 할 요청이 object 상태를 먼저 바꿔버리면 이미 늦다.

## 작동 흐름

1. user space 또는 kernel 내부 경로가 어떤 action을 요청한다.
2. 커널이 요청 주체를 찾는다. 보통 `current->cred` 또는 `file->f_cred`가 기준이 된다.
3. 대상 object를 찾는다. file, socket, mount, namespace, BPF map 등이 될 수 있다.
4. action의 종류를 정한다. read, write, configure, attach, mount, load처럼 의미를 분리한다.
5. uid/gid, capability, namespace 기준을 확인한다.
6. 필요한 경우 LSM hook이나 seccomp filter가 추가 판단을 한다.
7. 허용되면 상태 변경으로 넘어가고, 거부되면 object를 바꾸기 전에 오류를 반환한다.

## 대표 예시

`nft add rule`은 단순히 rule object를 만드는 명령이 아니다. 커널은 먼저 이 요청이 어느 network namespace에서 들어왔는지 보고, 그 namespace 기준으로 `CAP_NET_ADMIN`이 있는지 확인해야 한다.

```text
netlink message
    -> network namespace 확인
    -> capability 확인
    -> nftables object 생성
    -> transaction commit
```

이 예시를 읽을 때는 다음을 확인한다.

- 권한 기준이 `init_user_ns`인지, 현재 user namespace인지, network namespace인지?
- 권한 검사가 object 생성 전에 수행되는가?
- worker thread나 delayed path에서 원래 요청자의 credential이 유지되는가?

## 핵심 용어

- `subject`: 요청을 보낸 주체. 보통 task나 file open 시점의 credential이다.
- `object`: 보호해야 하는 대상. file, socket, namespace, BPF map, device 등이 된다.
- `action`: object에 수행하려는 작업. read와 configure는 다른 권한을 요구할 수 있다.
- `cred`: uid, gid, capability 등 권한 상태를 담는 객체.
- `capability`: root 권한을 여러 세부 권한으로 나눈 bit.
- `namespace`: 권한과 object visibility의 기준을 나누는 격리 장치.
- `LSM hook`: 추가 보안 정책이 커널 decision point에 개입하는 지점.

## 다른 큰가지와의 연결

- [7. Permission Model](../7-permission-model/README.md): 접근 통제의 세부 모델을 다룬다.
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): 권한 기준이 namespace에 따라 달라진다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): open 시점 권한과 사용 시점 권한이 분리될 수 있다.
- [11. eBPF](../11-ebpf/README.md): BPF program load와 helper 사용은 강한 권한 경계다.

## 헷갈리기 쉬운 부분

- root인지 아닌지만 보면 된다고 생각하는 것
- `current->cred`와 `file->f_cred`를 구분하지 않는 것
- 권한 검사를 했다는 사실만 보고 어떤 namespace 기준인지 확인하지 않는 것
- object 상태를 바꾼 뒤 권한 검사를 하는 것

## 보안/취약점 관점

접근 통제가 깨지면 권한이 없는 사용자가 privileged operation을 실행한다. 영향은 단순 오류가 아니라 privilege escalation, sandbox escape, container escape로 이어질 수 있다.

코드를 읽을 때는 다음 질문을 붙인다.

1. subject, object, action이 명확히 분리되어 있는가?
2. 권한 검사는 상태 변경 전에 수행되는가?
3. capability check의 namespace 기준이 맞는가?
4. LSM hook이 모든 경로에서 호출되는가?
5. async worker나 callback에서도 원래 요청자의 권한 기준이 유지되는가?

## 다음에 읽을 문서

- [7. Permission Model](../7-permission-model/README.md)
- [8. Namespace / Isolation](../8-namespace-isolation/README.md)
- [자원 추상화](resource-abstraction.md)
- [상태 관리](state-management.md)

## 기억할 문장

접근 통제는 커널이 “대신 할 수 있는 일”과 “대신 해도 되는 일”을 구분하게 만드는 규칙이다.
