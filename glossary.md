# Glossary

이 용어집은 커널 문서를 읽을 때 반복해서 마주치는 단어를 짧고 쉬운 말로 정리한 문서다. 각 문서를 읽다가 단어 때문에 막히면 여기로 돌아오면 된다.

## kernel

하드웨어와 사용자 프로그램 사이에서 자원을 대신 관리하는 권한 있는 프로그램이다.

## user space

일반 프로그램이 실행되는 영역이다. 하드웨어와 커널 메모리에 직접 접근할 수 없다.

## kernel space

커널이 실행되는 영역이다. 하드웨어와 권한 있는 커널 객체에 접근할 수 있다.

## syscall

user program이 커널 기능을 요청하는 공식 입구다.

## kernel object

커널이 관리하는 상태 단위다. `struct file`, `task_struct`, `sk_buff` 같은 구조체가 대표적이다.

## fd

file descriptor의 약자다. user space가 커널 객체를 가리키기 위해 받는 작은 정수 handle이다.

## VFS

Virtual File System이다. 서로 다른 파일시스템과 장치를 파일처럼 다루게 하는 공통 계층이다.

## page

커널이 물리 메모리를 관리하는 기본 단위다.

## slab

작은 커널 객체를 빠르게 할당하고 재사용하기 위한 allocator 구조다.

## kmalloc

작은 kernel heap object를 할당하는 대표 API다.

## kfree

`kmalloc()` 계열로 받은 object를 allocator에 돌려주는 API다.

## kmem_cache

특정 타입이나 크기의 object를 빠르게 재사용하기 위한 slab cache다.

## VMA

`vm_area_struct`를 줄여 부르는 말이다. process 주소 공간의 연속된 영역과 권한을 표현한다.

## refcount

객체를 몇 곳에서 사용 중인지 세는 값이다.

## ownership

객체를 해제할 책임이 누구에게 있는지에 대한 규칙이다.

## RCU

reader가 객체를 보는 동안 writer가 해제를 늦추는 동시성/수명 관리 기법이다.

## spinlock

sleep하지 않고 짧은 임계구역을 보호하는 lock이다.

## mutex

sleep 가능한 context에서 공유 상태를 보호하는 lock이다.

## cred

task의 uid, gid, capability 같은 권한 상태를 담는 객체다.

## capability

root 권한을 여러 세부 권한으로 나눈 권한 비트다.

## namespace

같은 커널을 공유하면서도 서로 다른 view를 보이게 하는 격리 장치다.

## LSM

SELinux/AppArmor 같은 보안 정책을 커널 hook으로 적용하는 framework다.

## sk_buff

리눅스 네트워크 스택에서 packet 데이터와 메타데이터를 담는 객체다.

## netlink

user space와 커널 subsystem이 구조화된 message로 설정과 상태를 주고받는 socket 기반 인터페이스다.

## nftables

netfilter 위에서 table, chain, rule, expression, set으로 packet policy를 구성하는 subsystem이다.

## verifier

BPF program이 커널에서 실행되기 전에 pointer, range, control flow가 안전한지 검사하는 분석기다.

## BPF map

BPF program과 user space가 공유할 수 있는 key-value 저장소다.

## io_kiocb

io_uring에서 하나의 비동기 I/O 요청을 표현하는 커널 객체다.

## SQE

io_uring의 submission queue entry다. user space가 제출할 I/O 요청을 담는다.

## CQE

io_uring의 completion queue entry다. 커널이 완료 결과를 user space에 돌려줄 때 사용한다.

## 이 용어집을 읽는 방법

한 번에 전부 외울 필요는 없다. 커널 문서를 읽을 때 같은 단어가 반복해서 나오면 그때 다시 확인한다. 중요한 것은 단어 하나를 고립해서 외우는 것이 아니라, 그 단어가 어떤 큰가지에 붙는지 이해하는 것이다.
