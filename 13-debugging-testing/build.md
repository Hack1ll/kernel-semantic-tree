# build

## 한 문장 정의

`build`는 특정 kernel source, commit, config, compiler 조합으로 실험 가능한 kernel image와 symbol을 만드는 과정이다.

## 왜 중요한가

커널 분석에서 build는 준비 작업이 아니라 실험의 기준점이다. 같은 crash라도 commit, config, compiler, debug option이 다르면 재현 여부와 report 내용이 달라질 수 있다.

이 문서의 줄기는 이것이다.

```text
build는 내가 분석하는 커널이 실제로 어떤 코드와 설정으로 실행되는지 고정하는 단계다.
```

따라서 kernel image만 만들면 끝나는 것이 아니다. 나중에 crash log, stack trace, ftrace, syzkaller report를 해석할 수 있도록 config와 symbol까지 함께 관리해야 한다.

## build에서 고정해야 하는 것

실험 결과를 다시 믿으려면 다음 값이 명확해야 한다.

- kernel source tree
- git commit 또는 release version
- `.config`
- compiler 종류와 version
- architecture
- build output directory
- debug option
- sanitizer option
- module build 여부
- initramfs 또는 rootfs와의 호환성

이 중 하나가 달라지면 같은 reproducer도 다른 결과를 낼 수 있다.

## config의 역할

`.config`는 어떤 kernel 기능과 debug 기능이 들어갈지 정한다.

분석용 config에서 자주 확인하는 항목은 다음이다.

- target subsystem이 built-in인지 module인지
- target syscall, filesystem, network feature가 켜져 있는지
- `CONFIG_DEBUG_INFO` 계열 symbol 정보
- `CONFIG_KASAN`, `CONFIG_KCSAN`, `CONFIG_UBSAN`
- `CONFIG_LOCKDEP`, `CONFIG_PROVE_LOCKING`
- `CONFIG_FTRACE`, `CONFIG_FUNCTION_TRACER`
- `CONFIG_KCOV`
- panic, warning, slab debug 관련 option

config는 기능 목록이면서 관찰 가능성을 결정하는 입력이다.

## 산출물

build 후 중요한 산출물은 여러 개다.

- `vmlinux`: symbol이 포함된 ELF kernel image
- `bzImage` 또는 architecture별 boot image
- kernel modules
- `System.map`
- `.config`
- debug info
- generated headers

QEMU boot에는 boot image가 필요하고, crash 분석과 symbolization에는 `vmlinux`가 중요하다. `vmlinux` 없이 stack trace만 보면 함수 이름과 line number를 안정적으로 연결하기 어렵다.

## out-of-tree build

커널은 source tree와 build output을 분리할 수 있다.

```text
source tree: kernel source
output tree: .config, object file, vmlinux, image, modules
```

분리 build를 쓰면 여러 config를 같은 source에서 관리하기 쉽다. 예를 들어 KASAN용 build, lockdep용 build, syzkaller용 build를 따로 둘 수 있다.

확인할 점은 다음이다.

- 내가 실행한 image가 어느 output directory에서 나온 것인가?
- 현재 `.config`와 boot한 kernel image가 같은가?
- stale object나 이전 module이 섞이지 않았는가?

## compiler와 architecture

compiler 차이는 sanitizer report와 code generation에 영향을 줄 수 있다.

확인할 항목은 다음이다.

- GCC build인지 LLVM/Clang build인지
- target architecture가 무엇인지
- cross compile 환경인지
- debug info format이 분석 도구와 맞는지
- compiler optimization level이 debug 목적에 적절한지

architecture가 바뀌면 structure alignment, calling convention, atomic instruction, memory ordering 관찰 결과도 달라질 수 있다.

## build와 module

target code가 module이면 kernel image만 바꿔서는 부족하다. module도 같은 source와 config로 build되어야 한다.

검토할 질문은 다음이다.

- target code가 built-in인가, module인가?
- QEMU rootfs에 들어간 module이 새로 build한 module인가?
- module version mismatch가 없는가?
- module load 시 dmesg에 경고가 없는가?
- crash stack이 old module 주소를 가리키지 않는가?

module mismatch는 분석 시간을 크게 낭비하게 만든다.

## debug build와 성능 build

debug option을 많이 켠 kernel은 느려진다. 하지만 취약점 분석에서는 느린 kernel이 더 좋은 증거를 줄 수 있다.

구분할 목적은 다음이다.

- reproducer 탐색: sanitizer, KCOV, debug info를 켠 build
- root cause 분석: KASAN, lockdep, ftrace 등 필요한 option을 켠 build
- 성능 영향 확인: debug option을 줄인 build
- exploitability 판단: mitigations와 hardening config를 명확히 한 build

한 build로 모든 질문에 답하려고 하면 config 의미가 흐려진다.

## 코드에서 확인할 것

build 관련 문제를 점검할 때는 source code보다 artifact metadata를 먼저 본다.

확인할 항목은 다음이다.

- `.config`에 target feature가 켜져 있는가?
- `vmlinux`와 boot image가 같은 build에서 나왔는가?
- `System.map`이 현재 image와 일치하는가?
- module timestamp와 vermagic이 맞는가?
- sanitizer와 tracing option이 실제 config에 들어갔는가?
- QEMU command가 올바른 kernel image와 rootfs를 사용했는가?

## 보안 관점

build 관련 실수는 취약점 자체를 만들지는 않지만, 잘못된 결론을 만들 수 있다.

- 취약 commit이 아닌 kernel을 테스트
- sanitizer가 꺼진 build로 crash를 찾으려 함
- old module을 로드해 patch 검증을 잘못함
- debug symbol이 없어 stack trace를 잘못 해석
- config 차이 때문에 접근 가능한 syscall이나 subsystem이 달라짐
- mitigation 설정을 모른 채 exploitability를 판단

검토할 질문은 다음과 같다.

1. 이 kernel image가 내가 분석하는 commit에서 나온 것이 확실한가?
2. `.config`가 report와 함께 보관되어 있는가?
3. crash stack을 symbolization할 `vmlinux`가 같은 build 산출물인가?
4. target subsystem과 debug option이 실제로 켜져 있는가?
5. 재현 환경을 다시 만들 수 있을 만큼 build 입력이 남아 있는가?

## 다른 문서와의 연결

- [QEMU](qemu.md): build 산출물을 VM에서 부팅해 실험한다.
- [KASAN / KCSAN / UBSAN](kasan-kcsan-ubsan.md): sanitizer는 build config에서 켜야 한다.
- [lockdep](lockdep.md): lock 검증 기능도 config와 함께 결정된다.
- [syzkaller](syzkaller.md): fuzzing용 kernel은 KCOV와 debug option이 중요하다.

## 기억할 문장

커널 build는 binary 생성이 아니라, 이후 모든 crash와 trace를 해석할 기준이 되는 실험 조건 고정이다.
