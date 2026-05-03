# QEMU

## 한 문장 정의

`QEMU`는 직접 빌드한 kernel을 격리된 가상 머신에서 부팅하고 crash를 반복 관찰하기 위한 실행 환경이다.

## 왜 중요한가

커널 실험은 host에서 바로 실행하기 위험하다. crash, panic, filesystem 손상, network 설정 변경이 실험 중 발생할 수 있다. QEMU를 쓰면 kernel image, rootfs, command line, CPU 수, memory 크기를 명확히 정한 VM에서 같은 실험을 반복할 수 있다.

이 문서의 줄기는 이것이다.

```text
QEMU는 분석 대상 kernel을 통제된 하드웨어 환경에서 실행하게 해주는 재현 장치다.
```

QEMU를 잘 설정하면 crash를 안전하게 수집하고, boot log와 console output을 안정적으로 저장할 수 있다.

## QEMU가 정하는 실행 조건

QEMU command는 단순 실행 명령이 아니라 실험 조건이다.

확인할 항목은 다음이다.

- boot할 kernel image
- initramfs 또는 disk image
- kernel command line
- CPU architecture와 machine type
- vCPU 수
- memory 크기
- console 출력 방식
- network device
- virtio device
- snapshot 사용 여부
- gdb stub 사용 여부

이 조건이 달라지면 race timing, device path, filesystem behavior가 바뀔 수 있다.

## boot 구성

QEMU로 커널을 부팅하려면 보통 다음 세 가지가 필요하다.

- kernel image
- rootfs 또는 initramfs
- kernel command line

command line에서 자주 중요한 값은 다음이다.

- `console=ttyS0`: serial console로 log 출력
- `panic=1`: panic 후 재부팅 또는 종료 정책
- `oops=panic`: oops를 panic으로 승격
- `nokaslr`: 주소 randomization을 꺼서 주소 해석을 쉽게 함
- `root=`: root filesystem 지정
- `init=`: 실행할 init process 지정

분석 목적에 따라 command line을 고정해야 report가 비교 가능해진다.

## console과 log 수집

커널 crash 분석에서 가장 중요한 출력은 dmesg와 console log다.

QEMU에서는 serial console을 통해 다음을 수집한다.

- boot log
- warning
- sanitizer report
- lockdep report
- panic stack
- reproducer output

log가 잘리지 않도록 console 설정과 terminal logging 방식을 명확히 해야 한다. crash 후 VM이 바로 사라지면 중요한 stack trace를 잃을 수 있다.

## rootfs와 initramfs

rootfs는 reproducer와 필요한 tool을 넣는 공간이다.

확인할 항목은 다음이다.

- reproducer binary가 들어 있는가?
- busybox나 shell이 필요한가?
- `/proc`, `/sys`, `/dev`, `debugfs`, `tracefs` mount가 필요한가?
- module load가 필요한가?
- network test에 필요한 tool이 들어 있는가?
- syzkaller executor가 실행 가능한가?

minimal initramfs는 단순 재현에 좋고, disk image는 여러 tool을 넣어 반복 실험하기 좋다.

## CPU와 memory 설정

vCPU 수와 memory 크기는 재현성에 영향을 준다.

특히 다음 bug는 CPU 수와 timing에 민감하다.

- race condition
- lockdep warning
- KCSAN report
- RCU stall
- workqueue ordering bug
- interrupt 또는 timer 관련 bug

단일 CPU에서만 돌리면 일부 concurrency bug가 사라질 수 있다. 반대로 CPU 수를 늘리면 sanitizer overhead와 timing이 바뀐다.

## device model

QEMU가 제공하는 device는 kernel path를 바꾼다.

예시는 다음과 같다.

- virtio-blk, virtio-scsi: block I/O 경로
- virtio-net, e1000: network driver와 stack 경로
- 9p, virtiofs: host와 guest 파일 공유
- serial device: console 출력
- rng device: random source

driver나 networking bug를 분석할 때는 어떤 virtual device를 쓰는지 반드시 기록해야 한다.

## snapshot과 격리

reproducer가 filesystem을 망가뜨리거나 VM 상태를 바꿀 수 있다. snapshot이나 disposable rootfs를 쓰면 매 실험을 같은 상태에서 시작할 수 있다.

확인할 질문은 다음이다.

- 실행 후 rootfs가 오염되어 다음 재현에 영향을 주는가?
- crash 후 disk image가 손상될 수 있는가?
- host directory를 공유할 때 guest가 host 파일을 바꿀 수 있는가?
- network가 host 또는 외부에 불필요하게 열려 있지 않은가?

취약점 실험에서는 격리 수준이 분석 정확성과 안전성 모두에 중요하다.

## gdb와 디버깅

QEMU는 gdb stub을 제공할 수 있다. 이를 통해 boot 중단, breakpoint, register 확인, memory 확인을 할 수 있다.

다만 sanitizer와 tracing으로 충분한 경우도 많다. gdb를 사용할 때는 다음을 맞춰야 한다.

- `vmlinux` symbol
- QEMU gdb stub port
- KASLR 설정
- compiler debug info
- 최적화로 인한 line mapping 차이

gdb는 특정 지점을 깊게 볼 때 유용하고, ftrace와 sanitizer는 실행 흐름과 report를 넓게 볼 때 유용하다.

## 코드에서 확인할 것

QEMU 환경을 점검할 때는 command line과 guest 내부 상태를 함께 본다.

확인할 항목은 다음이다.

- boot한 kernel version이 예상과 같은가?
- `/proc/config.gz` 또는 저장된 `.config`가 build config와 맞는가?
- `/proc/cmdline`이 원하는 option을 포함하는가?
- CPU 수와 memory가 의도한 값인가?
- 필요한 filesystem이 mount되어 있는가?
- reproducer가 같은 path와 권한으로 실행되는가?
- dmesg가 host로 저장되는가?

## 보안 관점

QEMU 설정 실수는 재현 실패나 잘못된 원인 분석으로 이어진다.

- 다른 kernel image를 부팅함
- old rootfs의 module을 로드함
- serial console 설정이 없어 crash log를 잃음
- CPU 수 차이로 race가 사라짐
- KASLR이 켜져 symbol 해석이 어려워짐
- host 공유 디렉터리를 통해 실험 파일이 오염됨
- network 격리가 부족해 불필요한 외부 영향이 생김

검토할 질문은 다음과 같다.

1. 이 VM은 내가 빌드한 kernel image를 부팅하고 있는가?
2. crash log를 host에서 안정적으로 저장하고 있는가?
3. rootfs가 매 실험마다 같은 상태로 시작되는가?
4. CPU, memory, device 설정이 report와 함께 기록되는가?
5. 분석 중인 subsystem을 실제로 지나가는 virtual device를 쓰고 있는가?

## 다른 문서와의 연결

- [build](build.md): QEMU는 build 산출물을 실행한다.
- [KASAN / KCSAN / UBSAN](kasan-kcsan-ubsan.md): sanitizer report는 QEMU console에서 수집한다.
- [ftrace](ftrace.md): tracefs를 mount해 VM 안에서 runtime trace를 수집한다.
- [syzkaller](syzkaller.md): syzkaller는 QEMU VM을 반복 생성해 fuzzing한다.

## 기억할 문장

QEMU는 kernel crash를 안전하게 반복하기 위한 환경이며, command line과 device 설정까지 포함해 하나의 실험 조건으로 기록해야 한다.
