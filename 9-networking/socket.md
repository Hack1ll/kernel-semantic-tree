# socket

## 한 문장 정의

socket은 user space가 network protocol endpoint를 fd로 다루게 해주는 VFS object이며, 내부적으로 protocol-specific `struct sock` 상태와 연결된다.

## 왜 중요한가

네트워크 syscall은 대부분 fd를 통해 socket에 도달한다. `socket()`, `bind()`, `connect()`, `listen()`, `accept()`, `sendmsg()`, `recvmsg()`, `setsockopt()`는 모두 socket state를 만들거나 바꾼다.

```text
user fd
    -> fd table
    -> struct file
    -> struct socket
    -> struct sock
    -> protocol implementation
```

socket을 읽을 때는 VFS fd layer와 protocol state layer를 구분해야 한다.

## struct socket과 struct sock

네트워크 코드에는 이름이 비슷한 두 object가 자주 나온다.

- `struct socket`: VFS와 socket syscall layer에 가까운 object
- `struct sock`: TCP, UDP, raw socket 같은 protocol state를 담는 object

`struct file`은 socket fd를 가리키고, socket layer는 다시 `struct sock`으로 내려간다. 취약점 분석에서는 어떤 layer의 lock, refcount, state를 보는지 명확히 해야 한다.

## socket state

socket은 protocol마다 다른 state machine을 가진다.

TCP를 예로 들면 다음 전이가 중요하다.

```text
create
    -> bind
    -> listen 또는 connect
    -> established
    -> shutdown
    -> close
```

UDP는 connectionless 성격이 강하고, raw socket은 더 강한 권한이 필요할 수 있다. state 전이를 protocol 공통으로 단순화하면 안 된다.

## send와 receive

send path와 receive path는 서로 다른 방향으로 packet buffer를 움직인다.

```text
sendmsg()
    -> user buffer copy 또는 iov 처리
    -> skb 생성
    -> protocol output
    -> qdisc / net_device transmit

packet receive
    -> net_device
    -> skb
    -> protocol input
    -> socket receive queue
    -> recvmsg()
```

`sendmsg()`와 `recvmsg()`는 user pointer, length, socket state, queue state가 함께 만나는 entry point다.

## socket option

`setsockopt()`와 `getsockopt()`는 protocol-specific 설정을 바꾸는 control path다.

확인할 지점은 다음이다.

- option level과 option name이 정확히 검증되는가?
- user buffer length가 option 구조체 크기와 맞는가?
- option 변경이 socket state와 충돌하지 않는가?
- namespace와 capability check가 필요한 option인가?
- lock을 잡은 상태에서 user memory copy를 하지 않는가?

## namespace와 credential

socket은 network namespace와 credential에 연결된다. socket 생성 시점의 namespace, operation 시점의 credential, file credential이 서로 다를 수 있다.

특히 다음을 구분한다.

- socket이 속한 `struct net`
- raw socket이나 admin option에 필요한 capability
- user namespace 기준 `CAP_NET_ADMIN`
- socket fd가 다른 process나 namespace로 전달되는 경우

## 코드에서 확인할 것

1. 이 fd는 socket fd인가, regular file이나 device fd인가?
2. `struct socket`과 `struct sock` 중 어느 object를 다루는가?
3. socket state transition이 lock 아래에서 일관되게 처리되는가?
4. send/receive queue lifetime이 close와 race나지 않는가?
5. option length와 user buffer copy가 정확히 검증되는가?
6. socket의 network namespace와 capability 기준이 맞는가?

## 보안 관점

socket 버그는 state transition, queue lifetime, option parsing에서 자주 나온다.

- close와 send/recv가 동시에 실행되어 UAF가 생긴다.
- `setsockopt()` length 검증이 부족해 OOB read/write가 생긴다.
- raw socket 권한 check가 빠진다.
- socket이 속한 network namespace를 잘못 선택한다.
- accept queue, receive queue, backlog cleanup에서 reference가 어긋난다.

## 다른 문서와의 연결

- [sk_buff](sk-buff.md): socket send/receive path에서 이동하는 packet buffer
- [net_device](net-device.md): transmit와 receive가 도달하는 interface object
- [netlink](netlink.md): network state를 바꾸는 별도 socket family
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): socket fd와 `struct file`
- [8. Namespace / Isolation / net ns](../8-namespace-isolation/net-ns.md): socket의 network namespace

## 기억할 문장

socket을 읽을 때 핵심은 “이 fd가 어떤 protocol state로 이어지고, 그 state transition과 queue lifetime이 어떻게 보호되는가?”다.
