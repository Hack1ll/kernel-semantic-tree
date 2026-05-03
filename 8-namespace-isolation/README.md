# 8. Namespace / Isolation

## 핵심 질문

같은 kernel을 공유하면서 어떻게 서로 다른 system처럼 보이게 하는가?

## 큰가지의 의미

namespace는 kernel을 여러 개로 복제하지 않는다. kernel은 하나지만, task가 보는 object의 범위와 권한 기준을 나눈다.

container 보안의 핵심은 이 문장이다.

```text
view는 분리된다.
kernel은 공유된다.
```

따라서 namespace는 강한 격리 장치이지만, kernel bug 자체를 없애지는 않는다.

## 하위 문서의 역할

- [user ns](user-ns.md): uid/gid와 capability 기준을 분리한다.
- [mount ns](mount-ns.md): filesystem mount tree view를 분리한다.
- [net ns](net-ns.md): socket, device, route, firewall state를 분리한다.
- [pid ns](pid-ns.md): process 번호 공간과 보이는 process tree를 분리한다.
- [cgroup ns](cgroup-ns.md): cgroup hierarchy view를 분리한다.

## 이 장에서 특히 구분할 것

각 namespace는 분리하는 대상이 다르다.

```text
user ns  -> 권한 기준
mount ns -> path / filesystem view
net ns   -> network stack view
pid ns   -> process id view
cgroup ns -> resource control path view
```

하나의 object가 어느 namespace에 속하는지, 또는 global object인지 확인하는 습관이 중요하다.

## 대표 흐름

```text
container process
    -> user namespace에서 root
    -> host init_user_ns에서는 일반 uid
```

```text
socket()
    -> current task의 network namespace
    -> namespace 안의 route table / netfilter table 사용
```

## 다른 큰가지와의 연결

- [7. Permission Model](../7-permission-model/README.md): namespace는 capability 해석 기준을 바꾼다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): mount namespace는 path lookup 결과를 바꾼다.
- [9. Networking](../9-networking/README.md): network namespace는 socket과 packet policy를 분리한다.
- [2. Process / Task](../2-process-task/README.md): task가 namespace 묶음을 들고 있다.

## 보안 관점

namespace bug는 container isolation bug로 이어질 수 있다.

- container 내부 root를 host root처럼 처리
- namespace별 object와 global object 혼동
- cleanup 중 cross-namespace reference 잔류
- procfs/sysfs mount로 host 정보 노출
- net namespace cleanup과 packet path race

## 읽고 나서 확인할 것

1. 이 object는 어느 namespace에 속하는가?
2. 권한 check는 object의 namespace 기준으로 수행되는가?
3. 같은 문자열 path나 pid가 다른 namespace에서 다른 의미를 갖는가?
4. namespace cleanup 후에도 global reference가 남는가?
5. container에서 도달 가능한 kernel path인가?
