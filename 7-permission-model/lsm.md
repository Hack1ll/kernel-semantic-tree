# LSM

## 한 문장 정의

LSM은 커널의 주요 security decision point에 hook을 두고 SELinux, AppArmor, Smack, Landlock, BPF LSM 같은 보안 정책이 추가 판단을 하게 하는 framework다.

## 왜 중요한가

일반 Unix permission과 capability check가 허용해도, 시스템 보안 정책은 추가로 거부할 수 있다. LSM은 이 추가 정책이 커널 내부의 file, task, socket, inode, BPF, IPC 같은 object 접근에 개입하는 지점이다.

```text
kernel operation
    -> 기본 permission / capability check
    -> security_* LSM hook
    -> active LSM policy 판단
    -> allow 또는 deny
```

## hook의 의미

LSM hook은 상태 변경 직전 또는 object 생성/사용 지점에 배치된다.

대표 영역은 다음과 같다.

- file open, read, write, ioctl, mmap
- inode create, unlink, rename, permission
- task create, ptrace, signal
- socket create, bind, connect, sendmsg
- BPF program load, map create
- module load, kernel parameter, keyring

hook이 없으면 특정 보안 정책이 그 operation을 관찰하거나 거부할 기회를 잃는다.

## security blob과 label

LSM은 object에 security state를 붙일 수 있다. 이를 보통 security blob이나 label 관점으로 본다.

예를 들어 다음 object에 보안 정보가 붙을 수 있다.

- task credential
- inode
- file
- socket
- superblock
- IPC object

SELinux에서는 label과 policy rule이 핵심이고, AppArmor는 path/profile 중심 규칙을 쓴다. 구현은 다르지만 kernel hook 위치가 decision point라는 점은 같다.

## DAC, capability, LSM 관계

LSM은 기존 permission check를 대체하는 단일 체계가 아니다. 여러 계층이 함께 결론을 만든다.

```text
DAC permission
    -> capability exception
    -> LSM hook
    -> final decision
```

어떤 hook은 기본 check 전후에 배치될 수 있다. 중요한 것은 상태가 바뀌기 전에 모든 필요한 check가 끝나는가다.

## stacking

여러 LSM이 함께 활성화될 수 있다. 하나의 LSM이 허용해도 다른 LSM이 거부하면 operation은 실패할 수 있다.

정책을 분석할 때는 다음을 봐야 한다.

- 어떤 LSM들이 활성화되어 있는가?
- hook return value가 어떻게 결합되는가?
- audit/logging만 하는 LSM과 enforcement LSM이 구분되는가?
- Landlock처럼 unprivileged sandbox policy가 추가될 수 있는가?

## hook placement

LSM에서 가장 중요한 구현 질문은 hook 위치다.

잘못된 위치의 hook은 다음 문제를 만든다.

- 이미 상태가 바뀐 뒤 거부한다.
- 일부 syscall path에는 hook이 있는데 다른 equivalent path에는 없다.
- fast path가 hook을 우회한다.
- object 생성 path는 검사하지만 reconfiguration path는 검사하지 않는다.
- namespace cleanup이나 async path가 hook 없이 상태를 바꾼다.

## 코드에서 확인할 것

1. 이 operation에 대응하는 `security_*` hook이 있는가?
2. hook이 상태 변경 전에 호출되는가?
3. 같은 효과를 내는 다른 syscall path도 같은 hook을 거치는가?
4. hook에 전달되는 subject와 object가 정확한가?
5. object의 security blob이나 label lifetime이 보장되는가?
6. LSM return value를 무시하거나 덮어쓰지 않는가?

## 보안 관점

LSM 관련 버그는 policy bypass로 이어질 수 있다.

- hook 누락으로 특정 operation이 정책을 우회한다.
- hook 위치가 늦어 거부 전에 상태가 변경된다.
- equivalent path 중 하나만 hook을 호출한다.
- file label, inode label, task label을 혼동한다.
- security blob lifetime을 잘못 관리해 UAF가 생긴다.
- LSM deny return을 caller가 잘못 처리한다.

## 다른 문서와의 연결

- [cred](cred.md): task security label과 credential
- [capabilities](capabilities.md): capability check 이후 추가 정책
- [seccomp](seccomp.md): syscall entry 제한과 LSM object policy의 차이
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): file/inode operation hook
- [11. eBPF](../11-ebpf/README.md): BPF LSM과 BPF program load 정책

## 기억할 문장

LSM을 읽을 때 핵심은 “이 kernel operation의 보안 정책 hook이 정확한 subject, object, action 앞에 놓여 있는가?”다.
