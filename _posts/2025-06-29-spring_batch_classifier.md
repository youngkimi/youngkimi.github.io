---
title: "Spring Batch Classifier"
date: 2025-06-28 17:00:00 +/-TTTT
categories: [Spring, Batch]
tags: [spring, batch]
math: true
toc: true
pin: true
image:
  path: ../assets/image/icons/spring-batch.webp
  alt: spring_batch
---

# Spring Batch - ClassifierCompositeItemWriter

배치 작업을 하다 보면, 하나의 데이터 소스에서 **특정 기준에 따라 다른 처리가 필요한 경우**가 있다. 예를 들어, 쿼리 결과를 분기해서 서로 다른 DB 작업이나 API 호출 등으로 나누어야 할 경우, `ClassifierCompositeItemWriter`를 사용하면 효율적으로 처리할 수 있다. 

---

## 기본 구조

Job 설정은 아래와 같다. 테스트를 위해 `chunkSize`는 작게 설정하였다.

```java
@Configuration
public class ClassifyJobConfig extends DefaultBatchConfiguration {
    private int chunkSize = 10;

    @Bean
    Job classifyJob(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        return new JobBuilder("classifyJob", jobRepository)
            .start(classifyStep(jobRepository, transactionManager))
            .build();
    }
}
```

---

## Step 에서 ClassifierCompositeItemWriter 사용

Step 설정에서는 기존의 `ItemWriter` 대신 `classifier()`를 사용한다.

```java
@Bean
Step classifyStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("classifyStep", jobRepository)
        .<BookDto, BookDto>chunk(chunkSize, transactionManager)
        .reader(bookReader())
        .writer(classifier())  // <- ClassifierCompositeItemWriter
        .listener(/* ChunkListener */)
        .build();
}
```

여기서 주의할 점은, **모든 item에 대해 반드시 ItemWriter가 할당되어야 한다**는 것이다. 그렇지 않으면 런타임에서 예외가 발생한다.
따라서 `write()`를 원하지 않는 item은 **Processor에서 미리 필터링**하는 방식이 안전하다.

---

## Classifier 구현

아래는 Flag 기준으로 분기하는 간단한 classifier 구현 예시다:

```java
@Bean
ClassifierCompositeItemWriter<BookDto> classifier() {
    return new ClassifierCompositeItemWriterBuilder<BookDto>()
        .classifier(bookDto -> {
            if (bookDto.flag() == Flag.INSERT) {
                return insertItemWriter();
            } else if (bookDto.flag() == Flag.UPDATE) {
                return updateItemWriter();
            } else {
                return deleteItemWriter();
            }
        })
        .build();
}
```

> 위 코드에서는 `BookDto`에 `Flag` enum이 존재한다고 가정하였지만, 실제 구현 시에는 각자의 로직에 맞게 판단 기준을 작성하면 된다.

---

## 개별 ItemWriter

각 처리 로직은 별도의 `ItemWriter`로 분리한다.

```java
@Bean
ItemWriter<BookDto> insertItemWriter() {
    return chunk -> {
        for (BookDto b : chunk) {
            log.info("INSERT: {}", b.toString());
        }
    };
}
```

이 외에도 `updateItemWriter()`, `deleteItemWriter()`를 유사하게 작성하면 된다.
실제 프로젝트에서는 DB 작업, 외부 API 호출, 파일 쓰기 등으로 확장될 수 있다.

---

## 테스트 데이터 준비

Flag가 순환되도록 50개의 데이터를 생성하였다.

```java
@Bean
ItemReader<BookDto> bookReader() {
    List<BookDto> books = new ArrayList<>();
    for (int i = 1; i <= 50; i++) {
        books.add(new BookDto(i, "BOOK " + i, "AUTHOR" + i, Flag.values()[i % 3]));
    }
    return new ListItemReader<>(books);
}
```

---

## 로그 확인 및 실행 흐름

실행 로그는 다음과 같다:

```
UPDATE: BookDto{id=1, name='BOOK 1', author='AUTHOR1', flag=UPDATE}
UPDATE: BookDto{id=4, name='BOOK 4', author='AUTHOR4', flag=UPDATE}
UPDATE: BookDto{id=7, name='BOOK 7', author='AUTHOR7', flag=UPDATE}
UPDATE: BookDto{id=10, name='BOOK 10', author='AUTHOR10', flag=UPDATE}
DELETE: BookDto{id=2, name='BOOK 2', author='AUTHOR2', flag=DELETE}
DELETE: BookDto{id=5, name='BOOK 5', author='AUTHOR5', flag=DELETE}
DELETE: BookDto{id=8, name='BOOK 8', author='AUTHOR8', flag=DELETE}
INSERT: BookDto{id=3, name='BOOK 3', author='AUTHOR3', flag=INSERT}
INSERT: BookDto{id=6, name='BOOK 6', author='AUTHOR6', flag=INSERT}
INSERT: BookDto{id=9, name='BOOK 9', author='AUTHOR9', flag=INSERT}
=========== Read chunk done. Count: classifyStep
DELETE: BookDto{id=11, name='BOOK 11', author='AUTHOR11', flag=DELETE}
DELETE: BookDto{id=14, name='BOOK 14', author='AUTHOR14', flag=DELETE}
DELETE: BookDto{id=17, name='BOOK 17', author='AUTHOR17', flag=DELETE}
DELETE: BookDto{id=20, name='BOOK 20', author='AUTHOR20', flag=DELETE}
INSERT: BookDto{id=12, name='BOOK 12', author='AUTHOR12', flag=INSERT}
INSERT: BookDto{id=15, name='BOOK 15', author='AUTHOR15', flag=INSERT}
INSERT: BookDto{id=18, name='BOOK 18', author='AUTHOR18', flag=INSERT}
UPDATE: BookDto{id=13, name='BOOK 13', author='AUTHOR13', flag=UPDATE}
UPDATE: BookDto{id=16, name='BOOK 16', author='AUTHOR16', flag=UPDATE}
UPDATE: BookDto{id=19, name='BOOK 19', author='AUTHOR19', flag=UPDATE}
=========== Read chunk done. Count: classifyStep
INSERT: BookDto{id=21, name='BOOK 21', author='AUTHOR21', flag=INSERT}
INSERT: BookDto{id=24, name='BOOK 24', author='AUTHOR24', flag=INSERT}
INSERT: BookDto{id=27, name='BOOK 27', author='AUTHOR27', flag=INSERT}
INSERT: BookDto{id=30, name='BOOK 30', author='AUTHOR30', flag=INSERT}
UPDATE: BookDto{id=22, name='BOOK 22', author='AUTHOR22', flag=UPDATE}
UPDATE: BookDto{id=25, name='BOOK 25', author='AUTHOR25', flag=UPDATE}
UPDATE: BookDto{id=28, name='BOOK 28', author='AUTHOR28', flag=UPDATE}
DELETE: BookDto{id=23, name='BOOK 23', author='AUTHOR23', flag=DELETE}
DELETE: BookDto{id=26, name='BOOK 26', author='AUTHOR26', flag=DELETE}
DELETE: BookDto{id=29, name='BOOK 29', author='AUTHOR29', flag=DELETE}
=========== Read chunk done. Count: classifyStep
UPDATE: BookDto{id=31, name='BOOK 31', author='AUTHOR31', flag=UPDATE}
UPDATE: BookDto{id=34, name='BOOK 34', author='AUTHOR34', flag=UPDATE}
...
```

* chunkSize가 10이므로 10개 단위로 로그가 출력된다.
* item들이 **flag 기준으로 분류되어 각 writer에 전달**되고, 해당 writer 내에서 순차적으로 처리된다.
* 각 writer의 실행 순서는 chunk 내부에서의 호출 순서로 추측된다. 매 chunk 마다 writer가 반환된 순서대로 해당 write 작업이 실행되었다. 

---

## ClassifierCompositeItemWriter를 사용하는 이유

* **Bulk 처리**를 효율적으로 하기 위해서는 유사한 작업끼리 묶이는 것이 중요하다.
* `ClassifierCompositeItemWriter`를 사용하면, `INSERT`, `UPDATE`, `DELETE` 등 **처리 유형별로 item을 분류**할 수 있다.
* 이를 통해 **유형별 batch 처리(DB bulk insert/delete 등)** 가 가능하다.
* 반면, `classifier` 없이 하나의 Writer에서 분기하려면 **복잡한 조건문**이나 **비대한 SQL**이 필요해진다.
* 실제 프로젝트에서는 writer 에서 chunk를 받아 `<foreach>` 구문을 통해 처리하였다. 
* 실제 프로젝트에서는 `chunkSize`가 너무 커서 Bulk 처리에 장애가 생길 수 있다. `guava` 의 **Lists.partition**을 활용해 적정한 사이즈로 나누어 Bulk 처리를 수행할 수 있다.
