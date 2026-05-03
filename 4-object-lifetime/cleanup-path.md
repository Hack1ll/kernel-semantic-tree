# cleanup path

## 한 문장 정의

cleanup path는 object 생성, 등록, 요청 처리 중 실패하거나 취소될 때 이미 얻은 resource를 올바른 순서로 되돌리는 경로다.

## 왜 중요한가

커널 함수는 보통 여러 단계를 거쳐 object를 만든다. memory를 할당하고, reference를 얻고, lock을 잡고, list에 등록하고, callback을 예약한 뒤에야 성공할 수 있다.

중간 단계에서 실패하면 성공 path와 다른 정리 경로가 실행된다. 이 경로가 틀리면 leak, double free, UAF가 생긴다.

```text
allocate A
    -> get B
    -> register C
    -> start timer
    -> 실패
    -> timer 취소
    -> unregister C
    -> put B
    -> free A
```

## 역순 정리

cleanup의 기본은 얻은 순서의 반대로 내려놓는 것이다. 나중에 얻은 resource가 앞서 얻은 resource에 의존하는 경우가 많기 때문이다.

```text
acquire:
    A -> B -> C

cleanup:
    C -> B -> A
```

하지만 실제 커널 코드에서는 ownership transfer, partial initialization, registration 상태 때문에 단순 역순만으로 충분하지 않을 수 있다.

## goto label 구조

C 코드에서는 error path를 `goto` label로 정리하는 패턴이 흔하다.

```c
obj = kzalloc(...);
if (!obj)
    return -ENOMEM;

ret = init_a(obj);
if (ret)
    goto err_free_obj;

ret = register_b(obj);
if (ret)
    goto err_cleanup_a;

return 0;

err_cleanup_a:
cleanup_a(obj);
err_free_obj:
kfree(obj);
return ret;
```

label 이름은 “무엇을 정리하는가”와 맞아야 한다. label이 많아질수록 어떤 단계까지 성공했는지 추적해야 한다.

## 성공 경로와 실패 경로의 ownership 차이

성공하면 callee나 subsystem이 ownership을 가져가지만, 실패하면 caller가 정리해야 하는 함수가 많다.

확인할 패턴은 다음과 같다.

- 등록 성공 후에는 unregister가 필요하다.
- 등록 실패 후에는 등록 대상이 내부에서 정리되었는지 caller가 정리해야 하는지 봐야 한다.
- reference get 성공 후 다음 단계 실패 시 put이 필요하다.
- timer나 work를 예약했다면 실패 경로에서 취소 또는 drain이 필요하다.
- user-visible handle을 공개한 뒤 실패하면 외부 lookup을 먼저 차단해야 한다.

## cleanup과 idempotency

같은 cleanup 함수가 여러 경로에서 호출될 수 있다면, 두 번 호출되어도 안전한지 확인해야 한다.

예를 들어 다음 상태를 구분해야 한다.

- pointer가 NULL이면 free하지 않는가?
- list에 올라간 object인지 확인하고 제거하는가?
- timer가 실제로 pending인지 확인하는가?
- reference를 이미 put한 뒤 다시 put하지 않는가?
- callback이 실행 중이면 기다리는가?

idempotent cleanup을 의도하지 않았다면 호출 경로가 한 번만 실행되도록 상태를 엄격히 관리해야 한다.

## cancel path

cleanup path는 allocation failure에만 있는 것이 아니다. 비동기 요청 취소, device remove, process exit, namespace cleanup도 cleanup path다.

cancel path에서는 다음 문제가 자주 생긴다.

- completion path와 cancel path가 동시에 같은 request를 정리한다.
- callback이 이미 실행 중인데 object를 free한다.
- user-visible handle을 닫았지만 worker가 request를 계속 참조한다.
- error code를 반환한 뒤 내부 object가 실제로는 남아 있다.

## 코드에서 확인할 것

1. 각 단계에서 성공한 resource 목록이 무엇인가?
2. 실패 label이 성공한 단계의 정확한 역순으로 정리하는가?
3. ownership이 이동한 뒤 caller가 같은 object를 정리하지 않는가?
4. registration 이후 실패하면 lookup 가능성을 먼저 끊는가?
5. cancel path와 completion path가 같은 cleanup을 동시에 실행하지 않는가?
6. cleanup 함수가 한 번만 호출되는지, 여러 번 호출되어도 안전한지 명확한가?

## 보안 관점

cleanup path 버그는 정상 path보다 리뷰에서 놓치기 쉽다.

- error path leak: 실패 시 reference나 memory를 놓지 않는다.
- double free: 두 error label이 같은 object를 정리한다.
- UAF: unregister 전에 free해 lookup path가 죽은 object를 본다.
- callback after free: work, timer, RCU callback 취소 없이 memory를 반환한다.
- inconsistent state: 일부 table에는 남고 일부 state만 정리된다.
- wrong error return: 실패했지만 object가 공개된 상태로 남는다.

## 다른 문서와의 연결

- [ownership](ownership.md): 실패 시 책임이 caller에게 남는지 판단한다.
- [refcount](refcount.md): get 성공 뒤 실패할 때 put이 필요하다.
- [async callback lifetime](async-callback-lifetime.md): cancel, flush, drain 순서
- [12. io_uring](../12-io-uring/README.md): cancel과 completion이 만나는 request cleanup
- [10. Device Drivers](../10-device-drivers/README.md): probe 실패와 remove path

## 기억할 문장

cleanup path를 읽을 때 핵심은 “어디까지 성공했고, 그 성공을 어떤 역순으로 되돌리는가?”다.
