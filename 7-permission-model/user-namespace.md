# user namespace

## 한 문장 정의

user namespace는 uid/gid 값과 capability의 의미를 namespace별로 분리해, 같은 uid 0이라도 어느 권한 범위에 속하는지 다르게 해석하게 하는 구조다.

## 왜 중요한가

container 안의 root는 host root와 같은 권한을 가져서는 안 된다. user namespace는 이 차이를 만든다.

```text
task cred
    -> uid/gid
    -> user_namespace
    -> uid/gid mapping
    -> namespace 기준 capability
```

권한 코드를 읽을 때 uid 값만 보면 부족하다. 그 uid가 어느 user namespace에서 어떤 host id로 mapping되는지 봐야 한다.

## uid/gid mapping

user namespace는 내부 id와 외부 id의 mapping을 가진다.

예를 들어 namespace 내부 uid 0이 host uid 100000에 mapping될 수 있다. 이 경우 namespace 내부에서는 root처럼 보이지만 host 전체 권한을 의미하지 않는다.

커널 코드에서는 raw integer uid보다 `kuid_t`, `kgid_t` 같은 kernel id 타입을 통해 namespace 변환을 다루는 경우가 많다.

## parent와 child namespace

user namespace는 계층 구조를 가진다. child namespace 안에서 capability가 있어도 parent namespace나 host namespace object에 대한 권한이 자동으로 생기지는 않는다.

질문은 항상 target object가 어느 user namespace에 속하는지다.

```text
current user_ns
    -> child namespace 안에서는 CAP_* 있음
target object user_ns
    -> host 또는 parent namespace object일 수 있음
```

이 둘이 다르면 단순 current capability check가 잘못될 수 있다.

## ns_capable

namespace-aware 권한 check에서는 `ns_capable(target_user_ns, CAP_*)` 같은 형태가 중요하다.

핵심은 current task가 target user namespace에 대해 해당 capability를 가지는지 확인하는 것이다.

예를 들어 network namespace, mount namespace, pid namespace object가 어떤 user namespace에 의해 소유되는지에 따라 권한 기준이 달라질 수 있다.

## user namespace와 다른 namespace

user namespace는 다른 namespace 권한의 기준점이 된다.

- mount namespace 생성과 mount operation 권한
- network namespace 설정 권한
- pid namespace 내부 process 조작 권한
- cgroup namespace와 resource control 권한
- filesystem id mapping과 permission 해석

“namespace로 격리됐다”는 말은 보이는 범위와 권한 기준을 함께 봐야 한다.

## unprivileged user namespace

일부 환경에서는 unprivileged user가 새 user namespace를 만들 수 있다. 그러면 user namespace 내부 root로 도달 가능한 kernel code path가 늘어난다.

취약점 연구에서는 다음을 특히 봐야 한다.

- user namespace root에게 허용되는 syscall인가?
- host kernel object를 잘못 노출하는가?
- namespace 내부 capability만으로 너무 강한 operation을 허용하는가?
- cleanup 경로에서 cross-namespace reference가 남는가?

## 코드에서 확인할 것

1. current task는 어느 user namespace에 속하는가?
2. target object는 어느 user namespace 기준으로 소유되는가?
3. uid/gid 비교 전에 namespace 변환이 필요한가?
4. `capable()` 대신 `ns_capable()`이 필요한가?
5. unprivileged user namespace root가 이 경로에 도달할 수 있는가?
6. parent/child namespace 관계를 잘못 단순화하지 않았는가?

## 보안 관점

user namespace 버그는 container escape나 권한 우회로 이어질 수 있다.

- namespace 내부 root를 host root로 착각한다.
- target object가 속한 user namespace가 아닌 current 기준으로만 check한다.
- uid/gid mapping 없이 raw integer를 비교한다.
- unprivileged user namespace에서 privileged path가 열려 있다.
- cross-namespace object reference가 cleanup 후에도 남는다.
- filesystem permission이 idmapped mount와 user namespace를 잘못 결합한다.

## 다른 문서와의 연결

- [capabilities](capabilities.md): capability가 해석되는 기준
- [cred](cred.md): task credential과 user namespace pointer
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): namespace 전체 구조
- [mount namespace](../6-vfs-fd-model/mount-namespace.md): mount operation과 user namespace 권한
- [9. Networking](../9-networking/README.md): network namespace와 `CAP_NET_ADMIN`

## 기억할 문장

user namespace를 읽을 때 핵심은 “이 uid와 capability가 어느 namespace 기준으로 의미를 가지는가?”다.
