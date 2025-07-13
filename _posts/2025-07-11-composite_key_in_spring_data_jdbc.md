---
title: "Spring Data JDBC에서 복합키(Composite Key) 사용하기"
date: 2025-07-11 17:00:00 +/-TTTT
categories: [Spring, Data, JDBC]
tags: [spring, spring-data, jdbc, composite-key]
math: true
toc: true
pin: true
comments: true
image:
  path: ../assets/image/icons/spring-data.webp
  alt: spring_data
---

# Spring Data JDBC에서 복합키(Composite Key) 사용하기 (1)

Spring Data JDBC는 JPA에 비해 단순한 구조와 직관적인 매핑 방식 덕분에 많은 개발자들이 선호하지만, **복합키(Composite Primary Key)**를 사용하려는 순간부터 생각보다 많은 제약을 만나게 된다. 이번 글에서는 Spring Data JDBC (3.5.3 기준)에서 복합키를 어떻게 사용할 수 있는지, 그리고 어떤 문제가 생기는지를 실제 코드와 함께 정리해 본다.

## 1. Spring Data JDBC에서 복합키 사용은 가능한가?

다음과 같은 형태로 복합키를 객체로 정의하고 `@Embedded @Id` 조합으로 사용할 수 있다.

```java
@Data
@Table("SAMPLE")
public class Sample {

    @Id
    @Embedded.Empty
    private CompositeKey compositeKey;

    private Long val1;
    private Long val2;

    @Data
    public static class CompositeKey {
        private Long id1;
        private Long id2;
    }
}
```

Spring Data JDBC 3.5.3에서도 이 구조는 컴파일 오류 없이 사용 가능하다. 하지만...

## 2. 문제

Spring Data JDBC는 복합키 객체를 인식하더라도, **기본 Repository의 일부 메서드를 사용할 수 없다.**이다. 대표적으로 다음과 같은 문제가 있다:

- `CrudRepository<Sample, CompositeKey>` 에서 `findById()` 호출 시:

  ```
  Unknown column 'SAMPLE.composite_key' in 'where clause'
  ```

  → 내부적으로 `WHERE composite_key = ?` 와 같은 쿼리를 생성해서 실패한다.

- `BeanPropertyRowMapper`는 `compositeKey.id1`, `compositeKey.id2` 같은 중첩 필드를 자동으로 매핑하지 못한다.

즉, **복합키 객체 구조는 허용되지만, 이를 자동으로 활용하는 기능은 사실상 사용할 수 없다.**

## 3. 해결 방법: 완전 수동 DAO 구성

Spring의 Repository 기능은 버리고, NamedParameterJdbcTemplate을 기반으로 한 수동 DAO를 작성해야 했다. 아래를 참고.

```java
// 인터페이스 외부로 노출
@Repository
public class SampleDao implements SampleJdbcRepository {

    // 기본 CrudRepository Interface. 해당 인터페이스가 제공하는 기능은 해당 인터페이스로 위임(delegate)한다.
    // 하지만 복합키를 사용하므로, findById와 같은 메서드는 복합키를 제대로 처리하지 못한다. 해당 메서드는 Dao 에서 직접 처리해주어야 한다.
    private final SampleJdbcRepository delegate;
    private final NamedParameterJdbcTemplate jdbcOperations;

    public SampleDao(SampleJdbcRepository delegate, NamedParameterJdbcTemplate jdbcOperations) {
      this.delegate = delegate;
      this.jdbcOperations = jdbcOperations;
	  }

	private static final String FINDBYID_QUERY = """
        SELECT * FROM SAMPLE
        WHERE id1 = :id1 AND  id2 = :id2
        """;

	@Override
	public Optional<Sample> findById(Sample.CompositeKey compositeKey) {
		Map<String, Object> params = new HashMap<>();
		params.put("id1", compositeKey.getId1());
		params.put("id2", compositeKey.getId2());

		try {
			Sample sample = jdbcOperations.queryForObject(
				FINDBYID_QUERY,
				params,
				(rs, rowNum) -> {
					Sample.CompositeKey key = new Sample.CompositeKey(
						rs.getLong("id1"),
						rs.getLong("id2")
					);
					return new Sample(
						key,
						rs.getLong("val1"),
						rs.getLong("val2")
					);
				}
			);
			return Optional.ofNullable(sample);
		} catch (EmptyResultDataAccessException e) {
			return Optional.empty();
		}
	}

  private static final String UPSERT_QUERY = """
        INSERT INTO SAMPLE (id1, id2, val1, val2)
        VALUES (:id1, :id2, :val1, :val2)
        ON DUPLICATE KEY UPDATE
            val1 = VALUES(val1),
            val2 = VALUES(val2)
        """;

	public <S extends Sample> int upsert(S entity) {
		return jdbcOperations.update(
			UPSERT_QUERY,
			new MapSqlParameterSource(entity.params())
		);
	}
}
```

## 4. 간단한 테스트

```java
@Test
void mergeValueWithJdbcCompositeKey() {
    Sample.CompositeKey key = new Sample.CompositeKey(1L, 1L);
    Sample sample = new Sample(key, 5L, 5L);

    sampleDao.upsert(sample);

    Sample merged = sampleDao.findById(key).orElseThrow();
    assertThat(merged.getVal1()).isEqualTo(5L);
    assertThat(merged.getVal2()).isEqualTo(5L);
}
```

---

> 💡 **정리**
>
> - 복합키를 쓰려면 아직까지는 **Spring Data JDBC의 "자동화"를 포기하고 수동 처리에 가까운 구조로 전환**해야 한다.
> - Id를 사용하지 않는 메서드들은 기본 CrudRepository의 기능을 활용할 수 있지만, `findById`, `deleteById` 따위는 직접 `Parameter`와 `ResultSet`을 매핑해줘야 한다.
> - Kotlin에서는 `by` 키워드를 통해 인터페이스 위임이 가능해 보일러플레이트 코드를 줄일 수 있다.

> 다음 글에서는 Spring Data JDBC 4.x에서 예정된 복합키 관련 변화를 다룰 예정이다.
