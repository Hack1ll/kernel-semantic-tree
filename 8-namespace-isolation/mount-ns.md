# mount ns

## 한 문장 정의

mount namespace는 task가 보는 filesystem mount tree를 분리해, 같은 path 문자열도 namespace마다 다른 filesystem view로 해석되게 한다.

## 왜 중요한가

container가 자기 root filesystem, `/proc`, `/sys`, `/dev`, bind mount를 따로 보려면 mount namespace가 필요하다. 이 격리는 path view를 바꾸지만, 이미 열린 fd나 host kernel 자체를 분리하지는 않는다.

```text
task
    -> mount namespace
    -> namespace별 mount tree
    -> path lookup 결과
```

path lookup의 세부 과정은 [6. VFS / FD Model](../6-vfs-fd-model/README.md)에서 다루고, 이 문서는 container isolation 관점의 mount view를 본다.

## mount tree view

mount namespace가 다르면 다음 view가 달라질 수 있다.

- `/`가 가리키는 root filesystem
- `/proc`, `/sys`, `/dev`가 어떤 filesystem으로 mount되는지
- bind mount로 어떤 host directory가 보이는지
- read-only mount와 writable mount의 배치
- mount point crossing 결과

같은 path 문자열도 namespace마다 다른 dentry, inode, superblock으로 이어질 수 있다.

## bind mount와 노출 범위

bind mount는 host의 특정 directory나 file subtree를 namespace 안의 다른 위치에 보이게 할 수 있다.

보안 관점에서는 다음을 확인한다.

- container 안에 host sensitive path가 bind mount 되었는가?
- read-only bind mount라고 믿지만 다른 writable path가 같은 inode로 이어지지 않는가?
- `/proc`이나 `/sys`가 host 정보를 과도하게 노출하지 않는가?
- device node가 `/dev`를 통해 노출되는가?

mount namespace는 filesystem view를 구성하는 도구이지, 잘못 mount한 path를 자동으로 안전하게 만들지는 않는다.

## propagation

mount propagation은 mount event가 다른 namespace나 peer mount로 전파되는 방식을 정한다.

분리된 mount namespace라도 propagation 설정에 따라 mount/unmount event가 예상보다 넓게 영향을 줄 수 있다.

확인할 지점은 다음이다.

- shared mount인가, private mount인가?
- container 내부 mount event가 host 쪽으로 전파될 수 있는가?
- host mount 변화가 container view에 반영될 수 있는가?
- cleanup 시 mount reference가 남는가?

## fd와 mount namespace

mount namespace는 path lookup view를 바꾼다. 하지만 fd는 이미 열린 `struct file` reference다.

```text
host에서 file open
    -> fd 전달
    -> container mount namespace 안에서 path로는 안 보임
    -> fd로는 계속 접근 가능할 수 있음
```

따라서 path view 제한과 fd 전달 정책을 함께 봐야 한다.

## 코드에서 확인할 것

1. path lookup이 어느 mount namespace 기준으로 수행되는가?
2. bind mount로 host object가 container 안에 노출되는가?
3. `/proc`, `/sys`, `/dev` mount가 isolation 목적에 맞게 제한되는가?
4. mount propagation 설정이 예상한 방향으로만 작동하는가?
5. 이미 열린 fd가 mount namespace 경계를 넘어 전달될 수 있는가?
6. mount namespace cleanup 뒤 mount, dentry, file reference가 남지 않는가?

## 보안 관점

mount namespace 문제는 path view 격리와 실제 접근 가능성을 혼동할 때 생긴다.

- 민감한 host path를 bind mount로 노출한다.
- read-only path만 보고 같은 object의 writable path를 놓친다.
- fd 전달로 path view 제한을 우회한다.
- `/proc`이나 `/sys` mount로 host 상태가 노출된다.
- propagation 설정 때문에 mount 변화가 격리 밖으로 전파된다.

## 다른 문서와의 연결

- [6. VFS / FD Model / mount namespace](../6-vfs-fd-model/mount-namespace.md): path lookup과 mount namespace 세부
- [user ns](user-ns.md): mount operation capability 기준
- [cgroup ns](cgroup-ns.md): cgroupfs path view와 container view
- [7. Permission Model](../7-permission-model/README.md): mount 권한과 capability
- [10. Device Drivers](../10-device-drivers/README.md): `/dev` 노출과 device node

## 기억할 문장

mount namespace를 isolation 관점에서 읽을 때 핵심은 “container가 보는 path view와 실제 fd/resource 접근 가능성이 일치하는가?”다.
