---
layout: post
title: "jpa - persistence"
subtitle: "jpa - persistence"
category: jpa
---

## Persistence Context

```java
em.persist(member);
```

- 엔티티 매니저를 활용해 영속성 컨텍스트에 엔티티를 저장하는 과정이다.
- 엔티티 매니저 당 영속성 컨텍스트가 하나 있다. 엔티티 매니저를 통해 접근할 수 있고, 관리할 수 있다.
- 여러 엔티티 매니저가 하나의 영속성 컨텍스트에 접근할 수도 있다.

## Entity Lifecycle

1. 비영속 (transient) : 영속성 컨텍스트와 관계가 없는 상태
   ```java
   Member member = new Member();
   member.setId(1L);
   member.setName("youngkimi");
   ```
2. 영속 (managed) : 영속성 컨텍스트에 저장된 상태 (`em.persist()`)
   ```java
   // 엔티티 매니저에 의해 관리되는 상태 (영속화)
   em.persist(member);
   ```
3. 준영속 (detached) : 영속성 컨텍스트에 저장되었다가 분리된 상태
   ```java
   /*
      엔티티를 영속성 컨텍스트에서 분리.
      엔티티 매니저를 닫거나 (em.close()) 초기화해도 (em.clear())
      기존 엔티티 매니저가 관리하던 엔티티는 준영속 상태가 된다.
   */
   em.detach(member);
   em.close();
   em.clear();
   ```
4. 삭제 (removed) : 삭제된 상태. 엔티티를 영속성 컨텍스트와 데이터베이스에서 샂게한다.
   ```java
   em.remove(member);
   ```

## 영속성 컨텍스트의 특징

- 영속 상태는 식별자 값이 있어야 한다. 식별자 값으로 엔티티를 구분하기 때문이다.
- 영속성 컨텍스트가 데이터베이스에 반영되기 위해서는 `flush()` 과정이 필요하다.

#### 엔티티 조회

- 영속성 컨텍스트 내부에는 영속 상태의 엔티티를 저장하는 캐시를 가진다. (1차 캐시)
- Map이 하나 있고, Key는 @Id로 매핑한 엔티티의 식별자. 값은 엔티티 인스턴스.

```java
/*
   1차 캐시에서 우선 조회하고, 있으면 반환.
   없으면 데이터베이스를 조회하고 캐싱한 뒤에 반환한다.
*/
em.find(Member.class, "1L");

/*
   아래 두 인스턴스 비교 결과는 어떻게 될까?
*/
Member a = em.find(Member.class, "1L");
Member b = em.find(Member.class, "1L");

System.out.println(a == b); // true. 같은 인스턴스이므로. `동일성`을 보장한다.
// 동일성 : 인스턴스가 동일.
// 동등성 : 인스턴스는 달라도 값이 동일. (equals())

```

#### 엔티티 등록

- 엔티티 매니저는 데이터 변경시 트랜잭션을 시작해야 한다.
- 트랜잭션의 커밋 직전까지, 데이터베이스에 엔티티를 저장하지 않고 내부 쿼리 저장소에 SQL을 모아둔다.
- 이후 영속성 컨텍스트를 플러시(변경 내용을 동기화)하고, 트랜잭션을 커밋한다.
- 이를 `transactional write-behind` 이라 부른다.

```java
begin(); // 트랜잭션 시작

save(A);
save(B);
save(C);

commit(); // 트랜잭션 커밋
```

- 위 프로세스에서, `save()` 시 마다 데이터베이스로 등록 쿼리를 보내도, `commit()` 하지 않으면 의미가 없다.
- 결국은 `commit()` 직전에만 모든 SQL이 전달되기만 하면 된다.
- 이는 추후 성능 최적화와도 연관된다.

#### 엔티티 수정

- JPA로 엔티티를 수정하면, 엔티티를 조회해서 데이터만 변경하면 된다.
- 자동으로 변경사항을 데이터베이스에 반영해준다. 이를 `dirty checking` 이라고 한다.
- 영속화될 때, 최초 상태를 캐싱해둔다. (스냅샷)
- 이후 플러시 시점에 스냅샷과 엔티티의 값을 비교, 변경된 엔티티를 찾아 수정 쿼리를 작성한다.
- `dirty checking`은 영속 상태의 엔티티에만 적용된다.
- JPA의 업데이트 전략은 모든 컬럼을 업데이트 하는 것이다.
  - 데이터 사용량이 증가한다.
  - 수정 쿼리가 항상 동일하다. 수정 쿼리를 미리 생성하고 재사용할 수 있다.
  - 필드가 너무 많으면 동적으로 쿼리를 생성할 수 있다. `@DynamicUpdate`
  - 하지만 이정도로 필드가 많다면 애초에 설계에 오류가 있을 가능성이 높다.

#### 엔티티 삭제

- 삭제 쿼리는 쓰기 지연과 마찬가지로, SQL 저장소에 모여있다가 이후 커밋 시점에 `flush()`될 때 전달된다.
- 이후에는 삭제된 엔티티를 재사용하지 말고, GC에 위임하는 것이 좋다.

## 플러시 (flush())

- 플러시는 영속성 컨텍스트의 변경 내용을 데이터베이스에 `반영` 하는 것이다.
- 플러시는 `영속성 컨텍스트를 비우는 것이 아니다.` 데이터베이스와 영속성 컨텍스트의 동기화 과정이다.
- 실행시에는

  1. 변경 감지가 동작, 스냅샷과 엔티티를 비교해서 달라진 엔티티의 수정 쿼리를 저장.
  2. 저장된 SQL들을 데이터베이스로 전송.

- 영속성 컨텍스트를 플러시하는 방법은 세 가지 있다.
  1. `em.flush()` 직접 호출 : 테스트를 제외하고는 거의 사용하지 않는다.
  2. 트랜잭션 커밋시 자동 호출
  3. JPQL 쿼리 실행 시 자동 호출 : JPQL 쿼리 실행 시, 기존 JPA의 수정 사항이 데이터베이스에는 반영되어 있지 않을 수 있다. (영속성 컨텍스트 상에서만 존재)
- 기본적으로는 `FlushModeType.AUTO`을 그대로 사용.

## 준영속

- 영속 상태의 엔티티를 준영속으로 만드는 것도 세 가지 방법이 있다.
  1. 명시적 준영속 `em.detach(entity)`
  2. 영속성 컨텍스트 초기화 `em.clear()`
  3. 영속성 컨텍스트 종료 `em.close()`
- 준영속 상태에 돌입하면, 이후 트랜잭션 커밋 (`flush()`) 에도 데이터베이스와 동기화되지 않는다.
- 1차 캐시는 물론, 쓰기 지연 저장소에 존재하던 해당 엔티티의 관련 내용이 모두 제거된다.
- `지연 로딩`이 불가하다.

> `지연로딩` : 접근을 위한 프록시 객체를 로딩, 실제 사용할 때 영속성 컨텍스트로 데이터를 불러오는 것

## 테스트 코드

### 실패 코드

```java
// interface TeamService
@Transactional
public Team createTeam(Team team) {
    return teamRepository.save(team);
}

@Transactional
public void addTeamMember(Team team, Member member) {
    team.getMembers().add(member);
    member.setTeam(team);
}

// interface MemberService
@Transactional
public Member createMember(Member member) {
    memberRepository.save(member);
    return member;
}

// @SpringBootTest class MappingTest
@BeforeEach
public void setUp() {
    for (int i = 1; i <= 3; i++) {

        System.out.printf("%d 번째 팀 생성 시작\n", i);
        Team team = new Team(i, "team " + i, new ArrayList<>());

        for (int j = 1; j <= i; j++) {
            System.out.printf("- %d 번째 팀원 생성 시작\n", j);
            Member member = new Member(j, "Member " + (i*j), null);
            member = memberService.createMember(member);
            teamService.addTeamMember(team, member);
        }

        teams.add(teamService.createTeam(team));
    }
}

@DisplayName("테스트 팀과 멤버의 생성 확인")
@Test
public void setUpAndCleanUpTest() {
    assertEquals(3, teams.size(), "테스트 팀은 세 가지");

    for (int i = 1; i <= 3; i++) {
        Team team = teams.get(i-1);
        List<Member> members = team.getMembers();
        assertEquals(i, members.size(), i + "번째 테스트 팀의 멤버 수는 " + i + " 명이다.");
        for (Member member : members) {
            assertEquals(member.getTeam(), team, "멤버의 팀은 " + i + "번째가 맞다.");
        }
    }
}

```

```
> 실패 결과 :
>   필요:null
>   실제:Team(id=1, name=team 1, members=[Member(id=1, name=Member 1, team=null)])
```

<details>
<summary>실패 원인</summary>
멤버에 비영속 상태인 팀을 mapping 했다.
</details>

### 수정 코드

```java
@BeforeEach
public void setUp() {

    for (int i = 1; i <= 3; i++) {

        System.out.printf("%d 번째 팀 생성 시작\n", i);
        // 팀을 영속화(persist)한 후, 해당 트랜잭션이 종료되면서 준영속(detached) 상태가 된다.
        Team team = teamService.createTeam(new Team(i, "team " + i, new ArrayList<>()));

        for (int j = 1; j <= i; j++) {
            System.out.printf("- %d 번째 팀원 생성 시작\n", j);
            Member member = new Member(j, "Member " + (i*j), null);
            // 멤버를 영속화(persist)한다.
            member = memberService.createMember(member);
            // 준영속 상태인 팀과 멤버를 @Transactional이 적용 메서드 내에서 영속화(persist)한다.
            // 연관관계를 설정한 후, 트랜잭션 종료로 인해 다시 준영속(detached) 상태가 된다.
            teamService.addTeamMember(team, member);
        }

        teams.add(team);
    }
}
```
