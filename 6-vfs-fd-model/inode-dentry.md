# inode / dentry

## 한 문장 정의

inode는 filesystem object의 identity와 metadata를 표현하고, dentry는 path component 이름과 inode의 연결을 cache하는 VFS object다.

## 왜 중요한가

path는 문자열이고, inode는 filesystem object이며, dentry는 이름 lookup 결과를 cache한다. 이 셋은 같은 것이 아니다.

VFS 코드를 읽을 때 path string에서 바로 inode로 간다고 생각하면 rename, hard link, mount crossing, negative dentry 같은 지점에서 흐름이 깨진다.

```text
path string
    -> component lookup
    -> dentry
    -> inode
```

## inode

inode는 filesystem 안의 object identity를 표현한다.

대표적으로 다음 정보를 담는다.

- file type: regular file, directory, symlink, device node 등
- permission bits와 owner
- size, timestamps, link count
- filesystem-specific state
- inode operation table

inode는 이름이 아니다. 같은 inode가 여러 이름으로 도달될 수 있다.

## dentry

dentry는 directory entry cache다. path component 이름과 inode 연결을 저장해 path lookup을 빠르게 한다.

예를 들어 `/tmp/a`를 찾을 때 VFS는 `/`, `tmp`, `a` component를 따라 dentry를 찾는다.

dentry가 가리키는 것은 보통 다음 중 하나다.

- 유효한 inode
- 존재하지 않는 이름을 cache하는 negative dentry
- mount point를 통과하는 entry

dentry는 이름 cache이므로 rename, unlink, mount 변경과 밀접하다.

## hard link와 rename

하나의 inode는 여러 dentry 이름으로 접근될 수 있다. hard link가 대표적이다.

```text
name A -> dentry A -> inode X
name B -> dentry B -> inode X
```

rename은 dentry의 이름 연결을 바꾸며, inode 자체가 곧바로 다른 object가 되는 것은 아니다. path 기반 보안 검사는 이 차이를 고려해야 한다.

## negative dentry

존재하지 않는 이름도 dentry cache에 남을 수 있다. 이를 negative dentry라고 한다.

negative dentry는 repeated lookup 비용을 줄이지만, create, unlink, rename과 함께 상태가 바뀔 수 있다. “lookup 결과가 없었다”는 사실도 concurrency 규칙 아래에서만 의미가 있다.

## inode operation과 file operation

inode와 `struct file`은 서로 다른 operation table을 가진다.

- inode operation: create, lookup, link, unlink, mkdir 같은 filesystem namespace 조작
- file operation: read, write, ioctl, mmap 같은 open instance 조작

directory entry를 조작하는 코드인지, open file을 조작하는 코드인지 구분해야 한다.

## 코드에서 확인할 것

1. 지금 다루는 object는 path string, dentry, inode, `struct file` 중 무엇인가?
2. 같은 inode에 여러 dentry가 연결될 수 있는가?
3. rename이나 unlink와 동시에 실행될 수 있는가?
4. negative dentry를 유효한 inode처럼 취급하지 않는가?
5. inode permission과 open file permission을 혼동하지 않는가?
6. mount point 통과가 dentry/inode 해석에 영향을 주는가?

## 보안 관점

inode/dentry 실수는 path 기반 검사의 틈으로 이어질 수 있다.

- path string 검증 뒤 rename으로 대상이 바뀐다.
- dentry만 보고 inode identity를 확인하지 않는다.
- hard link 때문에 “이 이름만 안전하다”는 가정이 깨진다.
- negative dentry 상태 변화와 create path가 race를 만든다.
- inode operation과 file operation 권한 검사를 섞는다.

## 다른 문서와의 연결

- [path lookup](path-lookup.md): path component가 dentry와 inode로 해석되는 과정
- [struct file](struct-file.md): open 뒤 만들어지는 file instance
- [mount namespace](mount-namespace.md): mount tree가 dentry/inode 해석에 주는 영향
- [7. Permission Model](../7-permission-model/README.md): inode permission과 LSM hook
- [5. Concurrency](../5-concurrency/README.md): rename, unlink, dcache concurrency

## 기억할 문장

inode와 dentry를 읽을 때 핵심은 “이 이름이 가리키는 cache entry와 실제 filesystem object identity가 어떻게 연결되는가?”다.
