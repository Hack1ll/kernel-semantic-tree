# path lookup

## 한 문장 정의

path lookup은 user가 넘긴 문자열 경로를 호출자의 root, cwd, mount namespace 기준으로 dentry, inode, mount 조합으로 해석하는 과정이다.

## 왜 중요한가

path는 단순 문자열이 아니다. 같은 문자열도 process의 root, cwd, mount namespace, symlink 처리 규칙에 따라 다른 object를 가리킬 수 있다.

fd 기반 syscall은 이미 열린 object를 다루지만, path 기반 syscall은 먼저 문자열을 VFS object로 해석해야 한다.

```text
"/tmp/a"
    -> 시작점 결정
    -> component walk
    -> dentry cache 조회
    -> inode 확인
    -> mount / symlink / permission 처리
    -> final path 획득
```

## 시작점

path lookup은 어디서 시작하는지가 중요하다.

- absolute path: 호출자의 root에서 시작한다.
- relative path: 호출자의 current working directory에서 시작한다.
- `openat(dirfd, path, ...)`: `dirfd`가 relative path의 기준이 될 수 있다.
- process마다 root와 cwd가 다를 수 있다.
- mount namespace에 따라 같은 absolute path도 다른 mount tree를 볼 수 있다.

따라서 path string만 보고 대상 object를 확정하면 안 된다.

## component walk

VFS는 path를 component 단위로 해석한다.

```text
/a/b/c
    -> /
    -> a
    -> b
    -> c
```

각 component에서 dentry cache를 보고, 필요하면 filesystem lookup operation을 호출한다. directory execute/search permission, symlink, mount point, `.`과 `..` 처리도 이 과정에 포함된다.

## symlink와 mount crossing

symlink는 path lookup을 다른 경로로 이어지게 한다. mount point는 같은 dentry 이름 아래에서 다른 filesystem tree로 넘어가게 한다.

보안 코드를 읽을 때는 다음을 확인해야 한다.

- symlink follow를 허용하는가?
- 마지막 component만 symlink 금지인가, 중간 component도 제한하는가?
- mount point를 넘어가도 되는가?
- `..`가 제한된 root 밖으로 나가지 못하게 막는가?

path resolution flag가 있는 API에서는 flag 의미를 정확히 따라가야 한다.

## TOCTOU

path lookup은 race에 취약한 영역이다. path를 검사한 뒤 나중에 같은 문자열을 다시 열면 그 사이 rename, unlink, mount 변경으로 대상이 바뀔 수 있다.

위험한 흐름은 다음과 같다.

```text
check("/tmp/x")
    -> 안전하다고 판단
rename 또는 symlink 변경
open("/tmp/x")
    -> 다른 object 열림
```

가능하면 path string을 반복 검사하기보다, lookup 결과에 reference를 잡거나 fd 기반 API로 전환하는 설계를 봐야 한다.

## open과 path lookup의 차이

path lookup은 object를 찾는 과정이고, open은 그 object에 대한 `struct file`을 만드는 과정이다.

```text
path lookup
    -> dentry / inode / mount

open
    -> permission 확인
    -> struct file 생성
    -> fd table에 등록
```

open이 끝난 뒤에는 fd가 open file instance를 가리킨다. 이후 path 이름이 바뀌어도 이미 열린 `struct file`은 계속 같은 open object를 가리킨다.

## 코드에서 확인할 것

1. path는 absolute인가, relative인가, `dirfd` 기준인가?
2. 호출자의 root, cwd, mount namespace가 무엇인가?
3. symlink follow와 mount crossing이 허용되는가?
4. lookup 결과에 reference를 잡고 사용하는가?
5. 검사한 path와 실제 사용하는 path가 같은 lookup 결과인가?
6. rename, unlink, mount 변경과 동시에 실행될 수 있는가?

## 보안 관점

path lookup 취약점은 검사한 이름과 실제 object가 달라질 때 생긴다.

- path validation 이후 같은 문자열을 다시 lookup한다.
- symlink를 통한 우회를 고려하지 않는다.
- mount namespace 또는 bind mount로 다른 object를 보게 된다.
- `..` 처리로 제한 밖 path에 도달한다.
- final component와 parent directory permission을 혼동한다.
- fd 기반 안정성을 path 기반 검사로 대체한다.

## 다른 문서와의 연결

- [inode / dentry](inode-dentry.md): path component가 dentry와 inode로 해석되는 방식
- [mount namespace](mount-namespace.md): path lookup의 mount tree 기준
- [struct file](struct-file.md): path lookup 뒤 open으로 만들어지는 object
- [7. Permission Model](../7-permission-model/README.md): path permission과 LSM hook
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): namespace가 path view를 나누는 방식

## 기억할 문장

path lookup을 읽을 때 핵심은 “이 문자열이 어느 기준점과 어느 namespace에서 어떤 object로 해석되는가?”다.
