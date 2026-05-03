# cgroup ns

## 한 문장 정의

cgroup namespace는 process가 보는 cgroup path와 hierarchy root를 가상화해, container 내부에서 host cgroup 배치를 그대로 드러내지 않게 하는 namespace다.

## 왜 중요한가

cgroup은 CPU, memory, I/O 같은 resource control과 process grouping을 담당한다. cgroup namespace는 resource limit 자체를 새로 만드는 장치가 아니라, cgroup hierarchy가 process에게 어떻게 보이는지를 바꾼다.

```text
task
    -> cgroup membership
    -> cgroup namespace root
    -> /proc/self/cgroup 표시
    -> cgroupfs path view
```

즉 cgroup namespace는 resource controller보다 view isolation에 가깝다.

## cgroup membership과 namespace view

task는 실제 cgroup membership을 가진다. cgroup namespace는 그 membership을 어떤 path로 보여줄지 바꾼다.

host에서는 긴 path가 보일 수 있다.

```text
/sys/fs/cgroup/user.slice/.../container-123
```

container 내부에서는 namespace root 기준으로 짧게 보일 수 있다.

```text
/
```

같은 membership이라도 표시되는 path가 다를 수 있다.

## cgroup namespace가 하지 않는 것

cgroup namespace 자체는 다음을 직접 제공하지 않는다.

- CPU limit 설정
- memory limit 설정
- I/O bandwidth 제한
- device access 정책
- process migration 권한 부여

이런 기능은 cgroup controller, cgroup filesystem permission, capability, runtime 설정과 연결된다. cgroup namespace는 이 기능들이 보이는 경로를 다르게 만든다.

## procfs와 cgroupfs

cgroup namespace는 `/proc/self/cgroup` 출력과 cgroupfs path view에 영향을 준다.

확인할 점은 다음이다.

- container 내부에서 host cgroup path가 노출되는가?
- cgroupfs mount가 namespace root와 일치하는가?
- process가 자기 namespace 밖 cgroup path를 볼 수 있는가?
- cgroup v1/v2 차이 때문에 path 의미가 달라지는가?

host path 노출은 직접 권한 상승은 아니더라도 host 구조 정보를 드러낼 수 있다.

## 권한과 delegation

cgroup을 조작하려면 namespace view뿐 아니라 filesystem permission, controller delegation, capability가 필요하다.

container runtime이 cgroup subtree를 delegate하면 내부 process가 제한된 범위에서 cgroup을 만들거나 조작할 수 있다. 이때 namespace view와 실제 delegated subtree가 일치해야 한다.

잘못된 delegation은 host cgroup hierarchy 조작으로 이어질 수 있다.

## 코드에서 확인할 것

1. 이 task의 실제 cgroup membership은 무엇인가?
2. cgroup namespace root 기준으로 어떤 path가 보이는가?
3. `/proc/self/cgroup`이 host path를 노출하지 않는가?
4. cgroupfs mount와 namespace view가 일치하는가?
5. controller 조작 권한이 delegated subtree 안으로 제한되는가?
6. cgroup namespace cleanup 뒤 cgroup reference가 남지 않는가?

## 보안 관점

cgroup namespace 문제는 resource control 자체보다 view와 delegation 혼동에서 자주 나온다.

- host cgroup path가 container 내부에 노출된다.
- delegated subtree 밖 cgroup을 조작할 수 있다.
- cgroupfs permission이 namespace view와 맞지 않는다.
- cgroup v1/v2 차이를 잘못 처리한다.
- cgroup namespace cleanup 뒤 task나 cgroup reference가 남는다.

## 다른 문서와의 연결

- [2. Process / Task](../2-process-task/README.md): task와 cgroup membership
- [mount ns](mount-ns.md): cgroupfs mount view
- [7. Permission Model](../7-permission-model/README.md): cgroupfs permission과 capability
- [4. Object Lifetime](../4-object-lifetime/README.md): cgroup object reference
- [13. Debugging / Testing](../13-debugging-testing/README.md): container 환경에서 관찰되는 cgroup path

## 기억할 문장

cgroup namespace를 읽을 때 핵심은 “resource limit이 아니라 cgroup hierarchy path가 누구 기준으로 보이는가?”다.
