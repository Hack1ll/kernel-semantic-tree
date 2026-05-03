# 문서 제목

## 한 문장 정의

이 개념을 한 문장으로 정의한다.

## 왜 중요한가

이 개념이 붙어 있는 큰가지의 질문을 먼저 적는다.
그다음 이 개념이 없으면 커널이 어떤 상태를 안전하게 유지하지 못하는지 설명한다.

## 중심 구조

이 개념에서 반드시 구분해야 하는 객체, 상태, 경계, 실행 흐름을 적는다.

```text
user input or kernel event
    -> kernel object 선택 또는 생성
    -> permission / lifetime / concurrency 규칙 적용
    -> 상태 변경 또는 결과 반환
```

## 코드에서 확인할 것

- 어떤 entry point에서 도달하는가?
- 어떤 kernel object가 만들어지거나 선택되는가?
- owner와 lifetime은 어떻게 정해지는가?
- 어떤 lock, refcount, RCU 규칙이 적용되는가?
- 어떤 permission 또는 namespace 기준을 쓰는가?
- 실패 경로와 비동기 경로에서도 같은 규칙이 유지되는가?

## 보안 관점

이 개념에서 깨지기 쉬운 규칙과 그 결과를 적는다.

- lifetime이 깨지는 경우
- bounds check가 틀리는 경우
- permission check가 빠지는 경우
- concurrent path가 같은 상태를 다르게 보는 경우
- cleanup 또는 error path가 정상 path와 다른 경우

## 다른 문서와의 연결

- [관련 큰가지](../README.md): 어떤 방식으로 연결되는지 설명한다.

## 기억할 문장

이 문서를 읽고 반드시 남겨야 할 핵심 문장을 하나 적는다.
