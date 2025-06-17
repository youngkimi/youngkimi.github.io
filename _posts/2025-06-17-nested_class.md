---
title: "중첩 클래스, 자바와 코틀린"
date: 2025-06-17 17:00:00 +/-TTTT
categories: [Kotlin]
tags: [java, kotlin]
math: true
toc: true
pin: true
image:
  path: ../assets/image/icons/kotlin.svg
  alt: kotlin
---

중첩 클래스(Nested Class)는 한 클래스 내부에서 정의된 클래스이다. 중첩 클래스를 정적으로 선언하냐에 따라 그 성질이 다르다. 

## 1. Java의 중첩 클래스

```java
public class OuterClass {
  int outerValue;

  class NestedClass {
    int nestedValue;

    public NestedClass() {
      outerValue = 1; // 정상
      nestedValue = 2; // 정상
    }
  }

  static class StaticNestedClass {
    int nestedValue;

    public StaticNestedClass() {
      outerValue = 3; // 에러
      nestedValue = 4; // 정상
    }
  }
}
```

Java에서 중첩 클래스를 비정적(Non-Static)으로 선언하는 경우, 내부 변수 뿐 아니라, 해당 중첩 클래스가 정의된 외부 클래스의 변수에도 접근할 수 있다. 자바에서 중첩 클래스 선언 시 기본적으로 비정적이다.

하지만 정적(Static)인 경우에는 정적 중첩 클래스가 선언된 외부 클래스의 변수에 접근할 수 없다. 해당 변수에 접근하고 싶다면, 외부 클래스를 직접 참조해야 가능하다. 중첩 클래스이지만, 별도의 클래스와 같이 기능한다. 

비정적 내부 클래스 사용 시에는 메모리 누수 문제가 발생할 수 있다. 비정적 내부 클래스는 외부 클래스에 의존성을 가지기 때문이다. 이러한 구조는 GC가 메모리를 회수를 방해하는 요인이 될 수 있다. 이러한 이유 때문에 Java에서는 정적 중첩 클래스를 사용하는 것을 더 권장한다.

## 2. Kotlin의 중첩 클래스

`Kotlin`에서는 정적 중첩 클래스가 기본이다. 오히려 비정적 중첩 클래스를 선언하기 위해서는 `inner`라는 키워드를 부여해야 한다. 

```kotlin
class OuterClass {
  val outerValue: Int = 0;

  class NestedClass {
    init {
      outerValue = 1 // 에러
      var nestedValue: Int = 2 // 정상 
    }
  }

  inner class InnerClass {
    init {
      outerValue = 3 // 정상 
      var nestedValue: Int = 4 // 정상 
    }
  }
}
```

> 위의 `init`은 `Kotlin`에서 생성자에서 구현하는 코드를 포함하는 블록이다. 아래와 같이 처리하면 객체 생성 시 출력문이 출력될 것이다. 

```kotlin
class Car(val passengers: Int = 4) {
  init {
    println("created car for $passengers passengers")
  }
}
```