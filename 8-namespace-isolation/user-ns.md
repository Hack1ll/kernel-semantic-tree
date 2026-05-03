# user ns

## 한 문장 정의

user namespace는 uid/gid mapping과 capability 해석 범위를 분리해, namespace 내부 root와 host root를 다른 권한 주체로 만드는 격리 기준이다.

## 왜 중요한가

container isolation에서 user namespace는 다른 namespace 권한의 기준점이 된다. mount namespace, network namespace, pid namespace를 조작할 때도 “어느 user namespace 기준 capability인가?”가 함께 따라온다.

```text
task cred
    -> user namespace
    -> uid/gid mapping
    -> namespace 내부 capability
    -> 다른 namespace object의 권한 기준
```

이 문서는 user namespace를 permission 세부보다 isolation 경계 관점에서 본다. capability set 자체는 [7. Permission Model](../7-permission-model/README.md)에서 더 자세히 다룬다.

## 내부 root와 host root

user namespace 내부 uid 0은 그 namespace 안에서 root처럼 보일 수 있다. 하지만 host의 `init_user_ns`에서 uid 0이라는 뜻은 아니다.

예를 들어 내부 uid 0이 host uid 100000에 mapping될 수 있다. 이 경우 내부에서 privileged operation처럼 보이는 작업도 host 전체 권한으로 이어지면 안 된다.

중요한 질문은 다음이다.

- current task가 어느 user namespace에 속하는가?
- target object는 어느 user namespace가 소유하는가?
- uid/gid 비교가 mapping 이후 값으로 수행되는가?
- capability check가 target namespace 기준인가?

## namespace ownership

다른 namespace object는 대개 어떤 user namespace에 의해 소유된다. network namespace의 설정 권한, mount namespace 조작 권한, pid namespace 내부 process 조작 권한은 이 소유 관계와 연결된다.

```text
network namespace
    -> owner user namespace
    -> CAP_NET_ADMIN check 기준

mount namespace
    -> owner user namespace
    -> mount operation capability 기준
```

따라서 namespace object를 볼 때는 “이 namespace는 누가 만들었는가?”보다 “이 namespace의 권한 기준 user namespace는 무엇인가?”를 봐야 한다.

## unprivileged user namespace

환경에 따라 일반 user가 새 user namespace를 만들 수 있다. 그러면 내부 root로 실행 가능한 kernel path가 늘어난다.

취약점 연구에서는 이 점이 중요하다.

- host 권한 없이도 user namespace 내부 root가 될 수 있다.
- namespace-aware capability check를 통과하는 path가 생긴다.
- container에서 도달 가능한 kernel attack surface가 넓어진다.
- cleanup 중 user namespace와 다른 namespace object reference가 얽힐 수 있다.

## global object와의 경계

user namespace가 있다고 해서 모든 kernel object가 복제되는 것은 아니다. 어떤 object는 namespace별로 분리되지만, 어떤 state는 global하거나 host namespace 기준이다.

위험한 지점은 다음이다.

- namespace 내부 capability로 global object를 바꾸는 path
- host object를 namespace object처럼 취급하는 path
- namespace cleanup 뒤 global table에 남은 reference
- uid/gid mapping 없이 raw integer id를 비교하는 path

## 코드에서 확인할 것

1. current task와 target object의 user namespace가 각각 무엇인가?
2. capability check가 target object의 namespace 기준으로 수행되는가?
3. uid/gid 값이 namespace mapping을 거친 값인가?
4. unprivileged user namespace 내부 root가 이 path에 도달할 수 있는가?
5. global object를 namespace-local object처럼 다루지 않는가?
6. namespace cleanup 뒤 다른 namespace나 global table에 reference가 남지 않는가?

## 보안 관점

user namespace isolation이 깨지면 container boundary가 무너질 수 있다.

- 내부 root를 host root처럼 처리한다.
- `capable()`만 사용해 target namespace 기준을 놓친다.
- uid/gid mapping 없이 raw id를 비교한다.
- namespace 내부 권한으로 host-global resource를 수정한다.
- user namespace cleanup 뒤 다른 namespace object가 stale reference를 가진다.

## 다른 문서와의 연결

- [7. Permission Model / user namespace](../7-permission-model/user-namespace.md): uid/gid와 capability 해석 세부
- [mount ns](mount-ns.md): mount operation 권한 기준
- [net ns](net-ns.md): network namespace owner와 `CAP_NET_ADMIN`
- [pid ns](pid-ns.md): pid namespace 내부 process 권한
- [2. Process / Task](../2-process-task/README.md): task가 namespace 묶음을 들고 있는 방식

## 기억할 문장

user namespace를 isolation 관점에서 읽을 때 핵심은 “이 namespace 내부 권한이 host-global 권한으로 새지 않는가?”다.
