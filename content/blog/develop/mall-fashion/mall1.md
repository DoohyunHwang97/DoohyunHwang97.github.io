---
title: 공통 응답 API 구현 - 프로젝트에서의 효율적이고 일관된 응답 관리
date: "2024-08-26T22:12:03.284Z"
description: "개인 프로젝트 mall-fashion 개선기 #1"
---

## 도입 배경

프로젝트가 커지면서 각 컨트롤러에서 개별적으로 `ResponseEntity`를 사용하여 응답을 반환하는 방식이 비효율적임을 느꼈습니다. 특히, 성공 응답과 예외 응답을 각각 다르게 처리하면서 코드가 중복되고 일관성이 떨어지기 시작했습니다. 이를 개선하기 위해 성공 응답과 예외 응답을 각각 `SuccessResult`와 `ErrorResult`라는 두 개의 클래스로 통일했습니다. 또한, 예외 응답의 경우 `ErrorEnum`이라는 열거형(enum)을 사용하여 에러 코드를 일관되게 관리할 수 있도록 재구성했습니다.

## 공통 응답 처리: SuccessResult와 ErrorResult

### 1. 성공 응답: SuccessResult 클래스

성공적인 응답은 프로젝트 전반에 걸쳐 공통된 형태로 반환될 수 있도록 `SuccessResult` 클래스로 캡슐화했습니다. 이 클래스는 제네릭 타입을 사용하여 다양한 데이터 타입을 응답으로 포함할 수 있습니다.

```java
@Getter
@NoArgsConstructor
public class SuccessResult<T> {
    private T response;

    @Builder
    public SuccessResult(T response) {
        this.response = response;
    }
}
```

- 제네릭 타입 T: 성공 응답이 포함할 수 있는 데이터 타입을 유연하게 설정할 수 있습니다.
- Builder 패턴: 객체를 생성할 때 보다 읽기 쉽게 코드를 작성할 수 있게 도와줍니다.

### 2. 예외 응답: ErrorResult 클래스

예외 응답을 일관되게 처리하기 위해 `ErrorResult` 클래스를 사용했습니다. 이 클래스는 에러 코드와 메시지를 포함하며, 프로젝트의 다양한 예외 상황에 대한 정보를 제공합니다.

```java
@Getter
@NoArgsConstructor
public class ErrorResult {
    private String code;
    private String message;

    @Builder
    public ErrorResult(ErrorEnum errorEnum) {
        this.code = errorEnum.getCode();
        this.message = errorEnum.getMessage();
    }
}
```

- 에러코드와 메시지: 각각의 예외에 대해 고유한 에러코드와 메시지를 포함합니다.
- Builder 패턴: ErrorEnum을 통해 에러 정보를 설정할 수 있습니다.

## 예외 관리: ErrorEnum을 이용한 재사용 가능한 에러 정의

프로젝트 내에서 발생할 수 있는 다양한 예외 상황을 관리하기 위해 `ErrorEnum`을 도입했습니다. 각 예외 상황에 대해 고유한 코드와 메시지를 정의하여, 예외 상황이 발생할 때 일관된 응답을 보장할 수 있습니다.

```java
@Getter
public enum ErrorEnum {
    // 시스템 예외
    INVALID_REQUEST(HttpStatus.BAD_REQUEST, "잘못된 요청입니다."),
    FORBIDDEN_REQUEST(HttpStatus.UNAUTHORIZED, "허용되지 않은 요청입니다."),
    FAILED_INTERNAL_SYSTEM_PROCESSING(HttpStatus.INTERNAL_SERVER_ERROR, "내부 시스템 처리 작업이 실패했습니다. 잠시 후 다시 시도해주세요."),
    NOT_FOUND(HttpStatus.NOT_FOUND, "존재하지 않는 정보 입니다."),

    // 사용자 예외
    INACTIVATED_USER(HttpStatus.UNAUTHORIZED, "비활성화된 계정입니다."),
    USERNAME_DUPLICATED(HttpStatus.CONFLICT, "이미 등록된 아이디입니다."),
    EMAIL_DUPLICATED(HttpStatus.CONFLICT, "이미 등록된 이메일입니다."),
    // ... 기타 예외들
    ;

    private final HttpStatus status;
    private final String code;
    private final String message;

    ErrorEnum(HttpStatus status, String message) {
        this.status = status;
        this.code = name();
        this.message = message;
    }
}
```

- `HttpStatus`와 함께 사용: 각 예외는 적절한 HTTP 상태 코드를 포함합니다.
- 코드와 메시지의 재사용: 여러 컨트롤러에서 일관되게 예외 처리를 할 수 있습니다.

## ResponseEntity의 활용

`ResponseEntity`는 스프링 프레임워크에서 응답을 정의할 때 자주 사용되는 클래스입니다. 이 클래스는 HTTP 응답 상태 코드, 헤더, 그리고 본문을 포함할 수 있어 매우 유용합니다.

프로젝트에서 `ResponseEntity`는 주로 예외 응답을 반환할 때 사용되었습니다. 예외 발생 시 `ErrorResult` 객체와 함께 적절한 HTTP 상태 코드를 포함하여 클라이언트에 응답을 반환할 수 있습니다.

```java
public ResponseEntity<ErrorResult> handleException(SomeException e) {
    ErrorResult errorResult = ErrorResult.builder()
                                         .errorEnum(ErrorEnum.INVALID_REQUEST)
                                         .build();
    return new ResponseEntity<>(errorResult, errorEnum.getStatus());
}
```

- HTTP 상태 코드 포함: 예외 상황에 따라 적절한 상태 코드가 자동으로 포함됩니다.
- 유연한 응답 구성: ResponseEntity를 통해 헤더, 본문, 상태 코드 등을 자유롭게 설정할 수 있습니다.

## 결론

공통 응답 API를 도입함으로써 프로젝트의 코드 일관성과 유지 보수성이 크게 향상되었습니다. 성공적인 응답과 예외적인 응답을 각각 SuccessResult와 ErrorResult 클래스로 통일함으로써, 보다 명확하고 직관적인 코드를 작성할 수 있게 되었습니다. 또한, ErrorEnum을 사용하여 예외를 관리함으로써 코드의 재사용성을 높이고, ResponseEntity를 통해 유연하게 응답을 제어할 수 있었습니다.

이러한 접근 방식을 통해 프로젝트의 응답 구조가 보다 체계적으로 정리되었으며, 이를 통해 클라이언트와의 통신이 더욱 명확해지고 오류 처리가 간소화되었습니다. 앞으로도 이 구조를 확장하여 다양한 상황에 대응할 수 있는 공통 응답 API를 만들어 나갈 수 있을 것입니다.

[mall-fashion 프로젝트 링크](https://github.com/f-lab-edu/shopping-mall-fashion)
