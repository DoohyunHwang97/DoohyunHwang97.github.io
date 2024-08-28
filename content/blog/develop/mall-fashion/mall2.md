---
title: 효율적인 예외 처리 - RestControllerAdvice를 활용한 전역 예외 처리
date: "2024-08-27T22:12:03.284Z"
description: "개인 프로젝트 mall-fashion 개선기 #2"
version: v3
---

## 도입 배경

기존의 프로젝트에서는 각 도메인마다 개별적으로 컨트롤러 내에 `@ExceptionHandler`를 정의하여 예외를 처리했습니다. 이로 인해 같은 예외 클래스를 처리하는 코드의 중복이 발생했고, 예외 처리가 분산되어 관리가 어려워지는 문제를 겪었습니다. 이에 따라 모든 예외를 한 곳에서 일관되게 처리할 수 있는 방법을 찾게 되었고, 그 결과 `@RestControllerAdvice`를 활용하여 전역 예외 처리기를 구현하게 되었습니다.

## RestControllerAdvice와 ExceptionHandler의 역할

### 1. @RestControllerAdvice: 전역 예외 처리기

`@RestControllerAdvice`는 Spring4.3에서부터 제공하는 어노테이션으로, 여러 컨트롤러에 대해서 전역적으로 예외를 처리할 수 있도록 도와줍니다. 기존의 컨트롤러 내에 개별적으로 정의되어 있던 `@ExceptionHandler` 메서드를 하나의 클래스에 모아 전역적으로 관리할 수 있게 합니다. 이를 통해 코드의 중복을 줄이고, 예외 처리 로직을 중앙에서 일괄적으로 관리할 수 있습니다. 또한 이는 공통된 예외 처리 응답과 함께 사용하여 사용자에게 일관된 예외 응답을 제공할 수 있다는 장점을 갖습니다.

하지만 여러 `@ControllerAdvice` 를 사용하는 경우 일관된 순서를 보장하기 위해 `@Order` 어노테이션을 사용해야 하거나 `basePackages` 지정을 통해 제어를 할 필요가 있습니다.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    private final EnumSet<ErrorEnum> errorEnumSet = EnumSet.allOf(ErrorEnum.class);

    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ErrorResult> exceptionHandler(ApiException e) {
        ErrorEnum error = e.getError();

        if (errorEnumSet.contains(error)) {
            return toErrorResponseEntity(error);
        } else return toErrorResponseEntity(ErrorEnum.FAILED_INTERNAL_SYSTEM_PROCESSING);
    }

    // 기타 예외 핸들러 메서드들...

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResult> exceptionHandler(Exception e) {
        log.error("Internal Error = {}", e.getMessage());
        return toErrorResponseEntity(ErrorEnum.FAILED_INTERNAL_SYSTEM_PROCESSING);
    }

    private ResponseEntity<ErrorResult> toErrorResponseEntity(ErrorEnum error) {
        return ResponseEntity
                .status(error.getStatus())
                .body(new ErrorResult(error));
    }
}
```

- 전역 예외 처리: @RestControllerAdvice를 사용하면 프로젝트 내의 모든 컨트롤러에서 발생하는 예외를 하나의 클래스에서 처리할 수 있습니다.
- 유연한 예외 처리: 특정 예외에 대해 다른 응답을 반환할 수 있는 메서드를 자유롭게 정의할 수 있습니다.
- 주의해야 할 점: `@ExceptionHandler` 어노테이션 내 파라미터로 넘겨준 예외 클래스와 클래스 매개변수의 타입이 일치해야 합니다.

### 2. @ExceptionHandler: 특정 예외 클래스 처리기

`@ExceptionHandler`는 추상화된 예외처리 전략을 제공합니다. 사용자는 예외클래스와 핸들러에 따라 다른 예외처리 전략을 가져갈 수 있습니다. `@ExceptionHandler`는 컨트롤러 내에서 발생하는 특정 예외를 처리하는 메서드를 정의할 때 사용되는데 이 어노테이션을 사용하면 특정 예외 클래스와 매핑된 메서드를 호출하여 예외를 처리할 수 있습니다. 그리고 `@ExceptionHandler`는 예외의 유연한 처리라는 장점을 갖는데 이는 다양한 형식의 응답을 지원하며 사용자가 자유롭게 payload를 설정할 수 있다는 점에서 기인합니다. 이는 제한적인 정보(ex. 응답코드)만 설정가능한 `@ResponseStatus` 나 `ResponseStatusException` 에 비해 `@ExceptionHandler`가 갖는 특화된 장점이라고 할 수 있습니다.

`@ExceptionHandler` 어노테이션은 컨트롤러의 메소드 혹은 `@ControllerAdvice`이 붙은 클래스 내 메소드에 사용이 가능합니다.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResult> exceptionHandler(MethodArgumentNotValidException e) {
    return toErrorResponseEntity(ErrorEnum.INVALID_REQUEST);
}
```

## 전역 예외 처리의 구현

GlobalExceptionHandler 클래스를 통해 프로젝트 내에서 발생할 수 있는 다양한 예외 상황을 하나의 클래스에서 처리하도록 구성했습니다. 각 예외에 대해 적절한 ErrorEnum을 사용하여 일관된 응답을 반환합니다.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    private final EnumSet<ErrorEnum> errorEnumSet = EnumSet.allOf(ErrorEnum.class);

    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ErrorResult> exceptionHandler(ApiException e) {
        ErrorEnum error = e.getError();

        if (errorEnumSet.contains(error)) {
            return toErrorResponseEntity(error);
        } else return toErrorResponseEntity(ErrorEnum.FAILED_INTERNAL_SYSTEM_PROCESSING);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResult> exceptionHandler(MethodArgumentNotValidException e) {
        return toErrorResponseEntity(ErrorEnum.INVALID_REQUEST);
    }

    @ExceptionHandler(NoResourceFoundException.class)
    public ResponseEntity<ErrorResult> exceptionHandler(NoResourceFoundException e) {
        return toErrorResponseEntity(ErrorEnum.NOT_FOUND);
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResult> exceptionHandler(IllegalArgumentException e) {
        return toErrorResponseEntity(ErrorEnum.INVALID_REQUEST);
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ErrorResult> exceptionHandler(HttpMessageNotReadableException e) {
        return toErrorResponseEntity(ErrorEnum.INVALID_REQUEST);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResult> exceptionHandler(Exception e) {
        log.error("Internal Error = {}", e.getMessage());
        return toErrorResponseEntity(ErrorEnum.FAILED_INTERNAL_SYSTEM_PROCESSING);
    }

    private ResponseEntity<ErrorResult> toErrorResponseEntity(ErrorEnum error) {
        return ResponseEntity
                .status(error.getStatus())
                .body(new ErrorResult(error));
    }
}

```

- 중복 코드 제거: 도메인별로 분산되어 있던 예외 처리 로직을 하나의 모듈로 통합하여 코드의 중복을 줄였습니다.
- 중앙 집중식 관리: 예외 처리 로직을 중앙에서 관리함으로써 유지 보수가 용이해졌습니다.
- 로깅: 예상치 못한 예외 발생 시 로그를 남겨, 문제의 원인을 빠르게 파악할 수 있습니다.

## 결론

`@RestControllerAdvice`를 활용한 전역 예외 처리는 프로젝트의 예외 처리 로직을 일관성 있게 만들고, 코드 중복을 줄이며, 유지 보수성을 크게 향상시켰습니다. 이 방법을 통해 예외 처리가 더욱 간결하고 명확해졌으며, 향후 발생할 수 있는 다양한 예외 상황에 유연하게 대응할 수 있게 되었습니다.

이러한 방식으로 예외 처리를 통합함으로써, 개발자는 보다 중요한 비즈니스 로직에 집중할 수 있게 되며, 클라이언트는 일관된 에러 메시지와 상태 코드를 받을 수 있어 사용자 경험을 개선할 수 있습니다.

## 참고 자료

- [스프링의 다양한 예외 처리 방법](https://mangkyu.tistory.com/204)
- [@RestControllerAdvice를 이용한 Spring 예외 처리 방법](https://mangkyu.tistory.com/205)

## 프로젝트 링크

- [mall-fashion](https://github.com/f-lab-edu/shopping-mall-fashion)
