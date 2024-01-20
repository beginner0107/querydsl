# Querydsl

<details>
<summary>초기설정 + Querydsl 세팅(boot3.x이상)</summary>
<div markdown="1">
  
```
// Query dsl 추가
implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
annotationProcessor "jakarta.annotation:jakarta.annotation-api"
annotationProcessor "jakarta.persistence:jakarta.persistence-api"

/**
 * QueryDSL Build Options
 */
def querydslDir = "src/main/generated"

sourceSets {
	main.java.srcDirs += [ querydslDir ]
}

tasks.withType(JavaCompile).configureEach {
	options.getGeneratedSourceOutputDirectory().set(file(querydslDir))
}

clean.doLast {
	file(querydslDir).deleteDir()
}
```
- gitignore 추가
```
/src/main/generated/
```

</div>
</details>
<details>
<summary>로깅 라이브러리 추가</summary>
<div markdown="2">

```
// P6Spy 의존성 추가
implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0'
```

</div>
</details>
<details>
<summary>JPQL, Querydsl 사용법 비교</summary>
<div markdown="3">

- JPQL
```java
Member findMember = em.createQuery("select m from Member m "
                                         + "where m.username = :username", Member.class)
    .setParameter("username", "member1")
    .getSingleResult();

assertThat(findMember.getUsername()).isEqualTo("member1");
```

- Querydsl
```java
@Autowired
EntityManager em;

JPAQueryFactory queryFactory;

@Test
public void startQuerydsl() {
  queryFactory = new JPAQueryFactory(em);
  QMember m = QMember.member;

  Member findMember = queryFactory
      .select(m)
      .from(m)
      .where(m.username.eq("member1"))
      .fetchOne();

  assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

</div>
</details>
<details>
<summary>기본 조인</summary>
<div markdown="4">

- 조인의 기본 문법은 첫 번째 파라미터에 조인 대상을 지정하고, 두 번째 파라미터에 별칭(alias)으로 사용할 Q타입을 지정하면 된다.
- join(조인 대상, 별칭으로 사용할 Q타입)

```java
@Test
public void join() throws Exception {
  QMember member = QMember.member;
  QTeam team = QTeam.team;

  List<Member> result = queryFactory
      .selectFrom(member)
      .join(member.team, team)
      .where(team.name.eq("teamA"))
      .fetch();

  assertThat(result)
      .extracting("username")
      .containsExactly("member1", "member2");
}
```

</div>
</details>
<details>
<summary>세타 조인</summary>
<div markdown="5">

- 연관관계가 없는 필드로 조인

```java
List<Member> result = queryFactory
      .select(member)
      .from(member)
      .join(team)
      .on(member.username.eq(team.name))
      .fetch();

  List<Member> result2 = queryFactory
      .select(member)
      .from(member, team)
      .where(member.username.eq(team.name))
      .fetch();
```

</div>
</details>
<details>
<summary>연관관계 없는 엔티티 외부 조인</summary>
<div markdown="6">

  
- 하이버네이트 5.1부터 on을 사용해서 서로 관계가 없는 필드로 외부 조인하는 기능이 추가되었음
   - 일반조인 : leftJoin(조인대상, 별칭으로 사용할 Q타입)
   - on조인 : from(조인대상1).leftJoin(조인대상2).on(조건)
 

```java
List<Tuple> result = queryFactory
    .select(member, team)
    .from(member)
    .leftJoin(team).on(member.username.eq(team.name))
    .fetch();
```

</div>
</details>
<details>
<summary>페치 조인 적용</summary>
<div markdown="7">

```java
Member findMember = queryFactory
      .selectFrom(member)
      .join(member.team, team).fetchJoin()
      .where(member.username.eq("member1"))
      .fetchOne();
```
- join(), leftJoin() 등 조인 기능 뒤에 fetchJoin()이라고 추가하면 된다.

</div>
</details>
<details>
<summary>서브쿼리</summary>
<div markdown="6">

- JPAExpressions

```java
import com.querydsl.jpa.JPAExpressions;

QMember memberSub = new QMember("memberSub");

List<Member> result = queryFactory
    .selectFrom(member)
    .where(member.age.eq(
        select(memberSub.age.max())
            .from(memberSub)
    ))
    .fetch();
```

</div>
</details>
<details>
<summary>CASE WHEN THEN</summary>
<div markdown="8">

```java
public void caseWhenThen() {
  List<String> result = queryFactory
      .select(member.age
          .when(10).then("열살")
          .when(20).then("스무살")
          .otherwise("기타")
      )
      .from(member)
      .fetch();

  List<String> result2 = queryFactory
      .select(new CaseBuilder()
          .when(member.age.between(0, 20)).then("0~20살")
          .when(member.age.between(21, 30)).then("21~30살")
          .otherwise("기타")
      )
      .from(member)
      .fetch();
}
```
```java
/**
 * 예를 들어 다음과 같은 임의의 순서로 회원을 출력하고 싶다면?
 * 1. 0~30살이 아닌 회원을 가장 먼저 출력
 * 2. 0~20살 회원 출력
 * 3. 21~30살 회원 출력
 */
public void orderByAndCase() {
  NumberExpression<Integer> rankPath = new CaseBuilder()
      .when(member.age.between(0, 20)).then(2)
      .when(member.age.between(21, 30)).then(1)
      .otherwise(3);

  List<Tuple> result = queryFactory
      .select(member.username, member.age, rankPath)
      .from(member)
      .orderBy(rankPath.desc())
      .fetch();

  for (Tuple tuple : result) {
    String username = tuple.get(member.username);
    Integer age = tuple.get(member.age);
    Integer rank = tuple.get(rankPath);
    System.out.println("username = " + username + " age = " + age + " rank = " + rank);
  }
}
```

</div>
</details>
<details>
<summary>상수, 문자 더하기(constant, concat)</summary>
<div markdown="9">

```java
/**
   * 참고: 아래와 같이 최적화가 가능하면 SQL에 constant 값을 넘기지 않는다.
   * 상수를 더하는 것처럼 최적화가 어려우면 SQL에 constant 값을 넘긴다.
   */
  @Test
  public void strPlus() {
    Tuple result = queryFactory
        .select(member.username, Expressions.constant("A"))
        .from(member)
        .fetchFirst();
  }

  /**
   * 문자 더하기 concat
   */
  @Test
  public void concat() {
    String result = queryFactory
        .select(member.username.concat("_").concat(member.age.stringValue()))
        .from(member)
        .where(member.username.eq("member1"))
        .fetchOne();
    // 참고: member.age.stringValue() 부분이 중요한데, 문자가 아닌 다른 타입들은 stringValue()로 문자로
    // 변환할 수 있다. 이 방법은 ENUM을 처리할 때도 자주 사용한다.
  }
```

</div>
</details>
<details>
<summary>프로젝션 결과 반환</summary>
<div markdown="10">

```java
@Test
public void simpleProjection() {
  List<String> result = queryFactory
      .select(member.username)
      .from(member)
      .fetch();
}

@Test
public void tupleProjection() {
  List<Tuple> result = queryFactory
      .select(member.username, member.age)
      .from(member)
      .fetch();
}
```

</div>
</details>
<details>
<summary>JPQL DTO 프로젝션</summary>
<div markdown="11">

```java
List<MemberDto> result = em.createQuery
        ("select new study.querydsl.dto.MemberDto(m.username, m.age) from Member m",
            MemberDto.class)
    .getResultList();
```

</div>
</details>
<details>
<summary>JPQL DTO 프로젝션</summary>
<div markdown="11">

```java
List<MemberDto> result = em.createQuery
        ("select new study.querydsl.dto.MemberDto(m.username, m.age) from Member m",
            MemberDto.class)
    .getResultList();
```

</div>
</details>
<details>
<summary>Querydsl setter 프로젝션</summary>
<div markdown="11">

```java
List<MemberDto> result = queryFactory
    .select(Projections.bean(MemberDto.class
        , member.username, member.age))
    .from(member)
    .fetch();
```

</div>
</details>
<details>
<summary>Querydsl 필드 프로젝션</summary>
<div markdown="12">

```java
@Test
public void findDtoByField() { // getter setter가 없어도 됨, 필드에다 값을 넣어줌
  List<MemberDto> result = queryFactory
      .select(Projections.fields(MemberDto.class
          , member.username, member.age))
      .from(member)
      .fetch();
}
```

</div>
</details>
<details>
<summary>Querydsl 생성자 프로젝션</summary>
<div markdown="13">

```java
@Test
public void findDtoByConstructor() {
  List<MemberDto> result = queryFactory
      .select(Projections.constructor(MemberDto.class
          , member.username, member.age)) // 타입 순서가 맞아야 함(생성자)
      .from(member)
      .fetch();
}
```

</div>
</details>
<details>
<summary>필드명이 다를 경우 as 사용</summary>
<div markdown="14">

```java
List<UserDto> result = queryFactory
    .select(Projections.constructor(UserDto.class
        , member.username.as("name"), member.age)) // 타입 순서가 맞아야 함(생성자)
    .from(member)
    .fetch()
```

</div>
</details>
<details>
<summary>ExpressionUtils를 활용한 서브쿼리 + 프로젝션</summary>
<div markdown="15">

```java
import com.querydsl.core.types.ExpressionUtils;

QMember memberSub = new QMember("memberSub");

List<UserDto> result = queryFactory
    .select(Projections.constructor(UserDto.class
        , member.username.as("name")
        , ExpressionUtils.as(JPAExpressions
            .select(memberSub.age.max())
            .from(memberSub), "age")
    ))
    .from(member)
    .fetch();
```

</div>
</details>
<details>
<summary>QueryProjection</summary>
<div markdown="16">

```java
@QueryProjection
public MemberDto(String username, int age) {
  this.username = username;
  this.age = age;
}
@Test
public void findDtoByQueryProjection() {
  List<MemberDto> result = queryFactory
      .select(new QMemberDto(member.username, member.age))
      .from(member)
      .fetch();
}
```

</div>
</details>
<details>
<summary>동적 쿼리 - BooleanBuilder</summary>
<div markdown="16">

```java
@Test
public void dynamicQuery_BooleanBuilder() {
  String usernameParam = "member1";
  Integer ageParam = null;

  List<Member> result = searchMember1(usernameParam, ageParam);
  assertThat(result).hasSize(1);
}

private List<Member> searchMember1(String usernameCond, Integer ageCond) {
  BooleanBuilder builder = new BooleanBuilder();

  if (usernameCond != null) {
    builder.and(member.username.eq(usernameCond));
  }

  if (ageCond != null) {
    builder.and(member.age.eq(ageCond));
  }

  return queryFactory
      .selectFrom(member)
      .where(builder)
      .fetch();
}
```

</div>
</details>
<details>
<summary>동적 쿼리 - Where 다중 파라미터 사용(1)</summary>
<div markdown="17">

```java
@Test
public void dynamicQuery_WhereParam() {
  String usernameParam = "member1";
  Integer ageParam = 10;

  List<Member> result = searchMember2(usernameParam, ageParam);
  assertThat(result).hasSize(1);
}

private List<Member> searchMember2(String usernameCond, Integer ageCond) {
  return queryFactory
      .selectFrom(member)
      .where(usernameEq(usernameCond), ageEq(ageCond))
      .fetch();
}

private Predicate usernameEq(String usernameCond) {
  if (usernameCond == null) {
    return null;
  }
  return member.username.eq(usernameCond);
}

private Predicate ageEq(Integer ageCond) {
  if (ageCond == null) {
    return null;
  }
  return member.age.eq(ageCond);
}
```

</div>
</details>
<details>
<summary>동적 쿼리 - Where 다중 파라미터 사용(2)</summary>
<div markdown="18">

```java
private Predicate usernameEq(String usernameCond) {
    return usernameCond != null ? member.username.eq(usernameCond) : null;
}
```

</div>
</details>
<details>
<summary>동적 쿼리 - Where 다중 파라미터 사용(3) BooleanExpression</summary>
<div markdown="19">

```java
return queryFactory
    .selectFrom(member)
    .where(allEq(usernameCond, ageCond))
    .fetch();
private BooleanExpression usernameEq(String usernameCond) {
  return usernameCond != null ? member.username.eq(usernameCond) : null;
}

private BooleanExpression ageEq(Integer ageCond) {
  return ageCond != null ? member.age.eq(ageCond) : null;
}

private BooleanExpression allEq(String usernameCond, Integer ageCond) {
  return usernameEq(usernameCond).and(ageEq(ageCond));
}
```

</div>
</details>
<details>
<summary>수정, 삭제 벌크 연산</summary>
<div markdown="20">

```java
@Test
@Commit
public void bulkUpdate() {
  // 영속성 컨텍스트 무시하고 쿼리가 나감
  // DB 와 영속성 컨텍스트 간의 상태가 불일치
  long count = queryFactory
      .update(member)
      .set(member.username, "비회원")
      .where(member.age.lt(28))
      .execute();

  em.flush(); // 데이터 보내고
  em.clear(); // 영속성 컨텍스트 비우기(초기화)

  List<Member> result = queryFactory
      .selectFrom(member)
      .fetch();

  for (Member member1 : result) {
    System.out.println("member1 = " + member1);
  }
}

@Test
public void bulkAdd() {
  long count = queryFactory
      .update(member)
      .set(member.age, member.age.add(1))
      .execute();
}

@Test
public void bulkMultiply() {
  long count = queryFactory
      .update(member)
      .set(member.age, member.age.multiply(2))
      .execute();
}

@Test
public void bulkDelete() {
  long count = queryFactory
      .delete(member)
      .where(member.age.gt(18))
      .execute();
}
```

</div>
</details>
<details>
<summary>SQL function 호출하기</summary>
<div markdown="21">

```java
@Test
public void sqlFunction() {
  List<String> result = queryFactory
      .select(
          Expressions.stringTemplate(
              "function('replace', {0}, {1}, {2})",
              member.username, "member", "M"))
      .from(member)
      .fetch();
}

@Test
public void sqlFunction2() {
  List<String> result = queryFactory
      .select(member.username)
      .from(member)
      .where(member.username.eq(member.username.lower()))
      .fetch();
}
```

</div>
</details>
<details>
<summary>사용자 정의 리포지토리</summary>
<div markdown="22">

- https://github.com/beginner0107/querydsl/commit/2c6d82f06600d1c2f11bd4c4d421552d2bb10c48

</div>
</details>
