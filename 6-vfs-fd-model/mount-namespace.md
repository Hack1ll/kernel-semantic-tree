# mount namespace

## 한 문장 정의

mount namespace는 task가 보는 filesystem mount tree를 분리해, 같은 path 문자열도 namespace마다 다른 mount와 inode로 해석될 수 있게 하는 격리 장치다.

## 왜 중요한가

container 안의 `/`와 host의 `/`는 같은 문자열이어도 같은 mount tree가 아닐 수 있다. path lookup은 항상 task가 속한 mount namespace의 view를 기준으로 진행된다.

```text
task_struct
    -> nsproxy
    -> mnt_namespace
    -> mount tree
    -> path lookup 결과
```

mount namespace는 path 기반 접근의 보이는 범위를 나눈다. 하지만 이미 열린 fd의 의미까지 단순히 path 문자열 기준으로 바뀌는 것은 아니다.

## path view 분리

mount namespace가 다르면 다음이 달라질 수 있다.

- `/`가 가리키는 root mount
- `/proc`, `/sys`, `/dev` 같은 pseudo filesystem의 mount 상태
- bind mount로 노출되는 host directory
- mount propagation 규칙
- path lookup 중 mount point crossing 결과

따라서 permission check나 path validation이 어느 namespace 기준인지 명확해야 한다.

## fd와 mount namespace

fd는 이미 열린 `struct file`에 대한 handle이다. path 이름이 namespace 안에서 보이지 않더라도, 이미 열린 fd가 있으면 그 file object를 계속 사용할 수 있다.

```text
namespace A에서 file open
    -> fd 획득
    -> 다른 path view로 이동
    -> fd는 여전히 struct file reference
```

이 차이 때문에 “path로는 안 보인다”와 “fd로도 접근할 수 없다”는 같은 말이 아니다.

## bind mount

bind mount는 기존 directory나 file subtree를 mount tree의 다른 위치에 노출할 수 있다.

보안 관점에서는 다음을 확인해야 한다.

- host path가 container 안으로 bind mount 되었는가?
- read-only로 보이는 mount라도 하위 mount나 fd가 writable path를 제공하지 않는가?
- 같은 inode가 다른 path로 접근 가능한가?
- mount namespace 바깥 object가 의도치 않게 보이는가?

bind mount는 path 문자열 기반 차단을 우회하는 경로가 될 수 있다.

## mount propagation

mount propagation은 mount event가 다른 namespace나 peer mount로 전파되는 방식을 정한다. shared, private, slave 같은 속성이 여기에 연결된다.

namespace를 분리했다고 해서 mount 변화가 항상 완전히 고립되는 것은 아니다. propagation 설정에 따라 mount 또는 unmount event가 다른 view에 영향을 줄 수 있다.

## user namespace와 capability

mount namespace 조작 권한은 user namespace와도 연결된다. container 내부 root가 host root와 같은 권한을 갖는 것은 아니다.

코드를 읽을 때는 다음을 구분해야 한다.

- 어느 user namespace 기준으로 capability를 검사하는가?
- mount namespace 생성이나 mount operation이 허용되는가?
- filesystem type별 추가 제한이 있는가?
- LSM hook이 path와 mount 기준을 모두 보는가?

## 코드에서 확인할 것

1. path lookup을 수행하는 task의 mount namespace는 무엇인가?
2. path validation이 host view 기준인가, caller namespace 기준인가?
3. 이미 열린 fd가 namespace 경계를 넘어 전달될 수 있는가?
4. bind mount로 같은 inode가 다른 path에 노출되는가?
5. mount propagation이 다른 namespace에 영향을 줄 수 있는가?
6. mount operation의 capability check가 올바른 user namespace 기준인가?

## 보안 관점

mount namespace 문제는 path view와 권한 기준을 혼동할 때 생긴다.

- host path 기준 검사를 container path에 그대로 적용한다.
- bind mount로 민감한 directory가 노출된다.
- fd 전달로 path namespace 제한을 우회한다.
- mount propagation 때문에 격리된다고 생각한 변화가 전파된다.
- user namespace root 권한을 host root 권한으로 착각한다.
- LSM이나 capability check가 namespace 기준을 놓친다.

## 다른 문서와의 연결

- [path lookup](path-lookup.md): mount namespace가 path 해석 기준을 바꾸는 과정
- [fd table](fd-table.md): 이미 열린 fd는 path view와 다른 lifetime을 가진다.
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): namespace 격리 전체 구조
- [7. Permission Model](../7-permission-model/README.md): user namespace와 capability
- [10. Device Drivers](../10-device-drivers/README.md): device node와 `/dev` mount 노출

## 기억할 문장

mount namespace를 읽을 때 핵심은 “이 path가 어느 mount tree view에서 해석되고, 이미 열린 fd와 어떻게 다르게 동작하는가?”다.
