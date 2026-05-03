# capabilities

## 한 문장 정의

capabilities는 전통적인 root 권한을 여러 `CAP_*` bit로 나누어, 특정 종류의 privileged operation만 허용하게 하는 권한 모델이다.

## 왜 중요한가

uid 0 하나로 모든 권한을 설명하면 너무 거칠다. 네트워크 설정, mount, process 조작, BPF 사용, raw socket 생성은 서로 다른 권한 요구를 가질 수 있다.

capability check는 “root인가?”보다 “이 operation에 필요한 capability가 있는가?”를 묻는다.

```text
operation
    -> 필요한 CAP_* 결정
    -> subject cred의 capability set 확인
    -> user namespace 기준 해석
    -> 허용 또는 거부
```

## capability bit

대표적으로 자주 만나는 capability는 다음과 같다.

- `CAP_NET_ADMIN`: network configuration, routing, netfilter 관련 관리
- `CAP_SYS_ADMIN`: 매우 넓은 범위의 system administration 권한
- `CAP_SYS_PTRACE`: 다른 process 관찰과 ptrace 관련 권한
- `CAP_DAC_OVERRIDE`: filesystem DAC permission 우회
- `CAP_SYS_MODULE`: kernel module load/unload
- `CAP_BPF`: BPF 관련 privileged operation 일부
- `CAP_PERFMON`: perf 관련 privileged operation 일부

`CAP_SYS_ADMIN`은 범위가 넓기 때문에 코드 리뷰에서 특히 조심해서 봐야 한다.

## capability set

credential에는 capability가 하나의 bitset만 있는 것이 아니다.

- permitted: task가 가질 수 있는 capability의 상한
- effective: 실제 권한 check에 쓰이는 active capability
- inheritable: exec 이후 상속 가능성에 연결
- bounding: process 계열이 얻을 수 있는 capability 상한
- ambient: 특정 조건에서 exec 이후 유지될 수 있는 capability

권한 check는 보통 effective set을 보지만, exec와 file capability를 읽을 때는 set 간 전환을 봐야 한다.

## capable과 ns_capable

capability는 user namespace 기준으로 해석된다.

- `capable(CAP_*)`: 보통 current의 namespace 관계를 기준으로 check한다.
- `ns_capable(user_ns, CAP_*)`: 특정 user namespace 기준으로 capability를 check한다.

namespace가 얽힌 object에서는 `capable()`을 써도 되는지, object가 속한 namespace 기준 `ns_capable()`이 필요한지 확인해야 한다.

## file capability와 exec

실행 파일은 file capability를 가질 수 있다. `execve()` 과정에서 file capability, setuid bit, current capability set, bounding set이 결합되어 새 credential이 만들어진다.

이 경계에서는 다음을 확인해야 한다.

- exec 이후 effective capability가 어떻게 바뀌는가?
- bounding set 때문에 capability 획득이 막히는가?
- `no_new_privs`가 privilege gain을 막는가?
- LSM이 exec transition을 추가로 제한하는가?

## over-privilege

capability check가 있다고 해서 항상 좋은 권한 분리가 되는 것은 아니다. 너무 넓은 capability를 요구하면 사실상 root와 비슷한 권한을 주게 된다.

확인할 점은 다음과 같다.

- operation에 필요한 최소 capability인가?
- `CAP_SYS_ADMIN` 대신 더 좁은 capability가 있는가?
- namespace 내부 root에게 허용해도 되는 operation인가?
- unprivileged user namespace에서 도달 가능한 경로인가?

## 코드에서 확인할 것

1. 이 operation에 필요한 capability가 정확히 무엇인가?
2. check가 상태 변경 전에 실행되는가?
3. `capable()`과 `ns_capable()` 중 어떤 기준이 맞는가?
4. target object가 속한 user namespace를 기준으로 check하는가?
5. `CAP_SYS_ADMIN`을 과도하게 쓰고 있지 않은가?
6. exec 이후 capability transition을 고려해야 하는가?

## 보안 관점

capability 관련 버그는 권한 경계가 잘못 열리는 형태로 나타난다.

- capability check 누락
- 잘못된 `CAP_*` 사용
- host namespace 기준 check가 필요한데 container user namespace 기준으로 허용
- object namespace 기준을 무시하고 current 기준으로만 check
- `CAP_SYS_ADMIN` 과다 사용으로 공격면 확대
- file capability와 exec transition을 잘못 해석

## 다른 문서와의 연결

- [cred](cred.md): capability set이 들어 있는 credential object
- [user namespace](user-namespace.md): capability 해석 기준
- [seccomp](seccomp.md): capability가 있어도 syscall 자체가 필터링될 수 있음
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): namespace별 권한 분리
- [11. eBPF](../11-ebpf/README.md): BPF 관련 capability와 verifier entry point

## 기억할 문장

capabilities를 읽을 때 핵심은 “이 privileged operation에 필요한 `CAP_*`가 어느 user namespace 기준으로 확인되는가?”다.
