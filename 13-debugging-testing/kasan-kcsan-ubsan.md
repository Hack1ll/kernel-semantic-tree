# KASAN / KCSAN / UBSAN

## 한 문장 정의

`KASAN`, `KCSAN`, `UBSAN`은 커널 실행 중 memory bug, data race, undefined behavior를 탐지하는 runtime instrumentation이다.

## 왜 중요한가

커널 버그는 증상 위치와 원인이 멀리 떨어져 있을 수 있다. 일반 crash는 이미 상태가 망가진 뒤에 발생할 수 있다. sanitizer는 규칙 위반이 일어난 순간에 가까운 위치에서 report를 남긴다.

이 문서의 줄기는 이것이다.

```text
sanitizer는 kernel이 실행 중 자기 안전 규칙을 어긴 순간을 report로 바꾸는 도구다.
```

세 sanitizer는 같은 역할을 하지 않는다. 어떤 bug를 찾으려는지에 따라 켜야 할 도구가 다르다.

## KASAN

KASAN은 memory access 오류를 찾는다.

대표적으로 탐지하는 문제는 다음이다.

- slab out-of-bounds
- stack out-of-bounds
- global out-of-bounds
- use-after-free
- invalid free
- double free 일부 경로

KASAN report에서 중요한 정보는 다음이다.

- 잘못된 access 위치
- read/write 여부와 access size
- object allocation stack
- object free stack
- slab cache 이름
- fault address 주변 shadow memory 정보

UAF 분석에서는 allocation, free, use stack을 모두 연결해야 한다. access stack만 보고 root cause를 단정하면 안 된다.

## KCSAN

KCSAN은 data race 가능성을 찾는다.

KCSAN은 모든 접근을 항상 감시하는 방식이 아니라 sampling 기반으로 race를 찾는다. 따라서 report가 없다고 race가 없다는 뜻은 아니다.

KCSAN report에서 보는 항목은 다음이다.

- 경쟁한 두 memory access 위치
- read/write 조합
- access size
- task와 CPU 정보
- lock 없이 접근했는지 여부
- atomic 또는 data_race annotation 여부

KCSAN은 “동시에 접근했다”는 신호를 준다. 그 접근이 실제 bug인지, 의도된 lockless pattern인지, annotation이 필요한지는 코드를 읽어 판단해야 한다.

## UBSAN

UBSAN은 C 언어의 undefined behavior를 runtime에 보고한다.

대표적으로 다루는 문제는 다음이다.

- signed integer overflow
- shift out-of-bounds
- array index out-of-bounds
- invalid enum value
- misaligned access
- null pointer 관련 일부 동작

커널에서는 모든 UBSAN report가 곧바로 exploit 가능 bug라는 뜻은 아니다. 하지만 size 계산, offset 계산, bounds check 주변의 undefined behavior는 보안 영향으로 이어질 수 있다.

## config와 비용

sanitizer는 build config에서 켠다. 실행 비용과 memory 사용량이 늘어난다.

확인할 항목은 다음이다.

- target architecture에서 해당 sanitizer가 지원되는가?
- KASAN mode가 어떤 방식인가?
- KCSAN sampling 설정이 bug 재현에 적절한가?
- UBSAN report가 panic으로 이어지도록 할 것인가?
- debug info가 있어 stack을 source line으로 연결할 수 있는가?
- sanitizer overhead가 race timing을 바꾸지 않는가?

sanitizer build는 일반 kernel과 timing이 다를 수 있다. 특히 concurrency bug에서는 이 차이를 고려해야 한다.

## report 읽는 순서

sanitizer report를 읽을 때는 다음 순서가 좋다.

1. 어떤 sanitizer가 낸 report인지 확인한다.
2. access type, size, address를 확인한다.
3. 현재 stack이 어떤 semantic path인지 찾는다.
4. allocation/free stack 또는 competing access stack을 연결한다.
5. object type과 owner를 찾는다.
6. lock, reference, bounds check 중 어떤 규칙이 깨졌는지 판단한다.
7. 같은 reproducer로 반복되는지 확인한다.

report는 증거이고, root cause는 커널 객체 규칙에서 찾아야 한다.

## false positive와 limitation

sanitizer는 강력하지만 한계가 있다.

- KASAN은 모든 temporal bug를 완벽히 잡지 못한다.
- KCSAN은 sampling 기반이라 report가 비결정적일 수 있다.
- KCSAN report 중 일부는 의도된 lockless access일 수 있다.
- UBSAN report는 보안 영향이 작은 코드 패턴일 수도 있다.
- sanitizer overhead가 timing을 바꿀 수 있다.
- compiler와 config에 따라 report 형태가 달라질 수 있다.

따라서 sanitizer report는 시작점이지 결론이 아니다.

## 코드에서 확인할 것

sanitizer report를 코드와 연결할 때는 다음을 찾는다.

- report stack의 function과 line
- object allocation site
- object free 또는 release site
- access 전에 필요한 refcount, lock, bounds check
- error path에서 cleanup 순서
- concurrent path에서 같은 field를 만지는 위치
- user input으로 access size나 offset을 조작할 수 있는지

## 보안 관점

sanitizer가 잘 잡는 취약점 유형은 다음이다.

- UAF
- heap overflow
- stack overflow
- out-of-bounds read/write
- data race로 인한 state corruption
- integer overflow 후 작은 allocation
- shift나 bounds 계산 오류
- uninitialized 또는 stale state로 이어지는 undefined behavior

검토할 질문은 다음과 같다.

1. report가 보여주는 access는 어떤 kernel object를 대상으로 하는가?
2. object lifetime, bounds, lock 중 어떤 규칙이 깨졌는가?
3. user space에서 해당 path에 도달할 수 있는가?
4. sanitizer가 꺼진 일반 build에서도 같은 root cause가 남는가?
5. fix 후 같은 sanitizer config에서 report가 사라지는가?

## 다른 문서와의 연결

- [build](build.md): sanitizer는 config에 포함되어야 한다.
- [QEMU](qemu.md): report는 VM console과 dmesg로 수집한다.
- [lockdep](lockdep.md): data race와 lock order 문제를 함께 확인할 수 있다.
- [4. Object Lifetime](../4-object-lifetime/README.md): KASAN UAF report는 lifetime 분석으로 이어진다.

## 기억할 문장

sanitizer report는 crash의 이름표가 아니라, 깨진 memory, concurrency, undefined behavior 규칙을 찾아가기 위한 실행 증거다.
