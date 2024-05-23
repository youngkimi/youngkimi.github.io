---
layout: single
title: "스프링의 예외 처리"
---

## 스프링의 예외처리 (Exception)

### 예외 종류
모든 예외 클래스는 `Throwable` 클래스를 상속받는다.
- Checked Exception
	- Compile Time Exception. 
	- 개발자가 작성 단계(컴파일 단계)에서 처리하는 예외. 반드시 명시적으로 처리한다.
	- 예외 발생 시 롤백하지 않는다. 
	- `IOException`, `SQLException` ...
- Unchecked Exception
	- 명시적으로 처리하지 않는다. 
	- 실행 중 발생한다. 
	- 예외 발생 시 롤백한다. 
	- `NullPointException`, `Illegal ArgumentException`, `SystemException` ...

### 스프링 부트의 예외 처리 방식
1. `@ControllerAdvice`를 통한 모든 Controller에서 발생하는 예외 처리
2. `@ExceptionHandler`를 통한 특정 Controller의 예외 처리

`@ControllerAdvice`에서 모든 컨트롤러에서 발생할 예외를 정의하고, `@ExceptionHandler`를 통해 발생하는 예외마다 처리할 메서드를 정의한다. 

#### `@ControllerAdvice`, `@RestControllerAdvice`
- Spring 제공 어노테이션. 
- `@Controller`나 `@RestController`에서 발생하는 예외를 한 곳에서 관리, 처리할 수 있게 하는 어노테이션. 
- 설정을 통해 범위 지정이 가능. 기본적으로 모든 Controller의 예외 처리 관리. 
- 예외 발생 시 json 형태로 결과 반환을 원한다면 `@RestControllerAdvice` 사용하면 됨. 

#### `@ExceptionHandler`
- 예외 발생 시 해당 Handler로 처리하겠다고 명시하는 어노테이션
- 어떤 예외를 처리할 것인지 설정할 수 있음. `@ExceptionHandler(BlahException.class)`
- 하위 세부 예외 클래스를 처리하는 Handler가 우선순위가 높다. 
- 각 Controller에서 설정할 수도 있으나, `@ControllerAdvice` 설정 클래스에서 메서드로 정의도 가능. 
- 전역 설정보다는 지역 설정(해당 Controller 처리)가 우선 순위가 높다. 

### 