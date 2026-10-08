# 1. JPA 소개

## 1.1 SQL을 직접 다룰 때 발생하는 문제점

### 1.1.1 반복적인 작업

자바와 관계형 데이터베이스를 사용하여 회원 관리 기능을 개발해보자.

`회원 객체`
```java
public class Member {
    private String memberId;
    private String name
}
```
`회원용 DAO`
```java
public class MemberDAO {
    public Member find(String memberId) {...}
}
```

<br>

#### 1. 회원 조회 기능 개발 
1. 회원 조회용 SQL을 작성한다.
```java
SELECT MEMBER ID, NAME FROM MEMBER M WHERE MEMBER ID =?
```
2. JDBC API을 사용해서 SQL을 실행한다
```java
ResultSet rs = stmt.executeQuery(sql);
```
3. 조회 결과를 Member 객체로 매핑한다.
```java
String memberId = rs.getString("MEMBER ID");
String name = rs.getString("NAME");
Member member = new Member();
member.setMemberId (memberId);
member.setName (name);
```

<br>

#### 2. 회원 등록 기능 추가
회원 조회 기능을 완성하고, 회원 등록 기능을 만들어보자
```java
public class MemberDAO {
    public Member find(String memberId) {...}
    public void save(Member member) (...) //추가
}
```
1. 회원 등록용 SQL을 작성한다
```java
String sql = "INSERT INTO MEMBER (MEMBER ID, NAME) VALUES(?,?)";
```
2. 회원 객체의 값을 꺼내서 등록한다.
```java
pstmt.setString(1, member.getMemberId());
pstmt.setString(2, member.getName ());
```
3. JDBC API를 사용해서 SQL을 실행한다.
```
pstmt.executeUpdate (sql);
```

<br>

회원 조회 기능과 등록 기능을 만들었고, 회원을 수정하고 삭제하는 기능도 추가해보자.
SQL을 작성하고 JDBC API를 사용하는 비슷한 일을 반복해야 할 것이다.

회원 객체를 데이터베이스가 아닌 자바 컬렉션에 보관한다면, 다음 한줄로 객체를 저장할 수 있다.
```java
list.add(member);
```
하지만 데이터베이스는 객체 구조와는 다른 데이터 중심의 구조를 가지므로, 개발자는
데이터베이스 중간에서 SQL과 JDBC API를 사용해서 변환 작업을 직접 해줘야 한다.

문제는 객체를 데이터베이스 CRUD하려면, 반복적으로 SQL과 JDBC API를 코드로 작성해야 한다는 점이다.

### 1.1.2 SQL에 의존적인 개발
####  등록 코드 변경
MemberDao 및 기능 개발 완료 후 회원의 연락처 저장 요구사항 추가되었다.

1. 필드 추가
```java
public class Member {
    private String memberId;
    private String name;
    private String tel; //추가
}
```
2. INSERT SQL 수정
```java
String sql = "INSERT INTO MEMBER (MEMBER ID, NAME, TEL) VALUES(?,?, ?)";
```
3. 회원 객체의 연락처 값을 꺼내 등록 SQL에 전달
```java
pstmt.setString (3, member.getTel());
```
<br>

#### 조회 코드 변경
다음으로 회원 조회 화면을 수정해서 연락처 필드가 출력되도록 하였다.
그런데 모든 연락처의 값이 null로출력된다.

1. 조회 SQL 수정
```java
SELECT MEMBER_ID, NAME, TEL FROM MEMBER WHERE MEMBER_ID = ?
```
2. 조회 결과를 추가 매핑
```java
String tel = rs.getString("TEL");
member.setTel (tel); //추가
```

<br>

#### 수정 코드 변경
연락처가 수정되지 않는 버그가 발견되었다. MeberDao를 열어서 UPDATE SQL을 확인하니, TEL 컬럼을 추가하지 않아서
발생하는 문제였다. MemberDao.update()의 일부 코드를 변경해서 연락처가 수정되도록 하였다.

<br>

#### 연관된 객체
회원은 어떤 한 팀에 필수로 소속되어야 한다는 요구사항이 추가되었다. 코드를 받아보니, team 필드가 추가되었다.
```java
class Member {
    private String memberId;
    private String name;
    private String tel;
    private Team team; //추가
    ...
}

//추가된 팀
class Team {
    ...
    private String teamName;
    ...
}
```

코드를 실행해보니 meber.getTeam()의 값이 항상 null이었다.
MemberDao 코드를 열어서 확인해보니 회원 출력에 사용하는 find() 메소드를 그대로 유지하였기 때문에 발생한 문제이다.

findWithTeam() 메소드를 추가하여 문제를 해결하였다.
``` SQL
SELECT M.MEMBER ID, M.NAME, M.TEL, T.TEAM ID, T.TEAM NAME
FROM MEMBERM
JOIN TEAM T
ON M.TEAM ID = T.TEAM ID
```

<br>

####  문제점
Meber나 Team과 같은 엔티티가 SQL에 모든 것에 의존하는 상황에서는, 엔티티를 신뢰하고 사용할 수 없다.
대신 DAO를 열어서 어떤 SQL이 실행되고 어떤 객체들이 함꼐 조회되는지 일일이 확인해야 한다.

SQL과 JDBC API를 데이터 접근 계층에 숨기는데 성공했을지는 몰라도, 회원을 조회하거나 필드를 하나 추가할 때도 
DAO의 CRUD와 SQL 대부분을 변경해야 하는 문제점이 발생한다. **(강한 의존 관계)**

1. 진정한 의미의 계층 분할이 어렵다
2. 엔티티를 신뢰할 수 없다
3. SQL에 의존적인 개발을 피하기 어렵다.

### 1.1.3 JPA와 문제 해결
JPA를 사용하면, 객체를 데이터베이스에 저장하고 관리할 떄, JPA가 제공하는 API를 사용하면 된다.
그러면 **JPA개 개발자 대신에 적절한 SQL을 생성해서 데이터베이스에 전달한다.**

#### 저장 기능
```java
jpa.persist (member); //저장
```

#### 조회 기능
```java
String memberId = "helloId";
Member member = jpa.find (Member.class, memberId);
```

#### 수정 기능
```java
Member member = jpa.find (Member.class, memberId);
member.setName("이름변경"); //수정
```

#### 연관된 객체 조회
```java
Member member = jpa.find(Member.class, memberId);
Team team = member.getTeam(); // 연관된 객체 조회
```

## 1.2 패러다임의 불일치

비즈니스 요구사항을 정의한 도메인 모델도 객체로 모델링하면, 객체지향 언어가 가진 장점들을 활용할 수 있다.
문제는 도메인 모델을 저장할 때 발생한다.

- 객체는 속성과 필드를 가진다. 객체의 기능은 클래스에 정의되어 있으므로, 인스턴스 상태인 속성만 저장했다가 필요할 때 불러서 복구하면 된다.
- 하지만 관계형 데이터베이스는 데이터 중심으로 구조화 되어있고, 집합적인 사고를 요구한다. 따라서 추상화, 상속, 다형성 같은 개념이 없다.

객체와 관계형 데이터베이스는 지향하는 목적이 서로 다르므로, 둘의 기능과 표현방법도 다르다. 
이것을 객체와 관계형 데이터베이스의 패러다임 불일치 문제라 한다.

### 1.2.1 상속

#### JPA와 상속
JPA는 상속과 관련된 패라다임의 불일치 문제를 개발자 대신 해결해준다. 마치 자바 컬렉션에 객체를 저장하듯이 JPA에게 객체를 저장하면 된다.

JPA를 사용해서 Item을 상속한 Album객체 저장해보자. persist()메소드를 사용해서 객체를 저장하면 된다.
```java
jpa.persist(album);
```
JPA는 다음 SQL을 실행해서 객체를 ITEM, ALBUM 두 테이블에 나누어 저장한다.
```sql
INSERT INTO ITEM ...
INSERT INTO ALBUM ...
```

다음으로 Alubum 객체를 조회해보자. find() 메소드를 사용해서 객체를 조회하면 된다.
```java
String albumId = "id100";
Album album = jpa.find (Album.class, albumId);
```
JPA는 ITEM과 ALBUM 두 테이블을 조인해서 필요한 데이터를 조회하고 그 결과를 반환한다.
```sql
SELECT I.*, A.*
FROM ITEM I
JOIN ALBUM A ON I.ITEM ID = A.ITEM ID
```

### 1.2.2 연관 관계

- 객체는 참조를 사용해서 다른 객체와 연관관계를 가지고 참조에 접근해서 연관된 객체를 조회
- 테이블은 외래키를 사용해서 다른 테이블과 연관관계를 가지고 조인을 사용해서 연관된 테이블을 조회

#### 객체를 테이블에 맞추어 모델링
```java
class Member {
    String id; //MEMBER_ID 컬럼 사용
    Long teamId; //TEAM ID FK 컬럼 사용
    String username; //USERNAME 컬럼 사용
}

class Team {
    Long id; //TEAM ID PK 사용
}
```

객체를 테이블에 저장하거나 조회할 떄는 편리하다. 하지만 객체는 연관된 객체의 참조를 보관해서 참조를 통해 연관된 객체를 찾을 수 있다.
`Member.teamId` 필드처럼 TEAM_ID 외래 키까지 관계형 데이터베이스가 사용하는 방식에 맞추면 Member 객체와 연관된 Team 객체를 참조를 통해서 찾을 수 없다.
```java
Member member = jpa.find (Member.class, memberId);
Team team = member.getTeam();
```

### 1.2.3 객체 그래프 탐색
객체에서 회원이 소속된 팀을 조회할 때는, 참조를 사용해서 연괸돤 팀을 찾으면 되는데, 이것을 `객체 그래프 탐색` 이라 한다.

SQL을 직접 다루면 처음 실행하는 SQL에 따라 객체 그래프를 어디까지 탐색할 수 있는지 정해진다. 이것은 객체지향 개발자에게는 너무 큰 제약이다.
왜냐하면 비즈니스 로직에 따라 사용하는 개체 그래프가 다른데, 언제 끊어질지 모를 객체 그래프를 함부로 탐색할 수는 없기 때문이다.
```java
class MemberService

public void process() {
    Member member = memberDAO.find(memberId);
    member.getTeam(); //member->team 객체 그래프 탐색이 가능한가?
    member.getOrder().getDelivery(); //???
}
```
MyService는 membrDao를 통해서 member 객체를 조회했지만, 이 객체와 연관된 Team, Order, Delivery 방향으로 객체 그래프를 탐색할 수 있을지 없을지는
이 코드만 보고는 예측할 수 없다. 결국 어디까지 그래프 탐색이 가능한지 알아보려면 데이터 접근 계층인 DAO를 열어서 SQL을 직접 확인해야 한다.

#### JPA와 객체 그래프 탐색
JPA를 사용하면 객체 그래프를 마음껏 탐색할 수 있다.
JPA는 연관된 객체를 사용하는 시점에 적절한 SELECT SQL을 실행한다. 따라서 JPA를 사용하면 연관된 객체를 신뢰하고 마음껏 조회할 수 있따.
이 기능은 실재 객체를 사용하는 시점까지 데이터베이스 조회를 미룬다고 해서 `지연 로딩`이라 한다.
```java
//처음 조회 시점에 SELECT MEMBER SQL
Member member = jpa.find (Member.class, memberId);

Order order = member.getOrder();
order.getOrderDate(); //Order를 사용하는 시점에 SELECT ORDER SQL
```
마지막 줄의 order.getOrderDate() 같이 실제 Order 객체를 사용하는 시점에 JPA는 데이터베이스에서 ORDER 테이블을 조회한다.

### 1.2.4 비교
- 데이터베이스 : 기본 키의 값으로 각 ROW를 구분한다.
- 객체는 동일성(identity) 비교, 동등성(equality) 비교라는 두가지 방법이 있다.

```java
// 조회환 회원 비교

String memberId = "100";
Member memberl = memberDAO.getMember (memberId);
Member member2 = memberDAO.getMember (memberId);
memberl == member2; // 다르다.
```
기본 값이 같은 회원 객체를 두번 조회했다. 그런데 동일성(==) 비교하면 false가 반환된다. 왜냐하면 member1과 member2는 같은 
데이터 베이스 로우에서 조회했지만, 객체 측면엣서 볼 때 둘은 다른 인스턴스이기 때문이다.
따라서 같은 로우를 조회했지만 객체의 동일성 비교에는 실패한다.

#### JPA와 비교
JPA는 같은 트랜잭션일 때 같은 객체가 조회되는 것을 보장한다. 그러므로 다음 코드에서 member1과 meber2는 동일성 비교에 성공한다.
```java
String memberId = "100";
Member memberl = jpa.find (Member.class, memberId);
Member member2 = jpa.find (Member.class, memberId);
memberl == member2; //같다.
```
## 1.3 JPA란 무엇인가?
JPA는 자바 진영의 ORM 기술 표준이다. 애플리케이션과 JDBC 사이에서 동작한다.

#### 1.3.2 왜 JPA를 사용해야 하는가?
1. 생산성: 자바 컬랙션에 객체를 저장하듯이 JPA에게 저장할 객체를 전달하면 된다. INSERT SQL을 작성하고 JDBC API를 사용하는 반복적인 일은 JPA가 대신 처리해준다.
```java
jpa.persist (member); //저장
Member member = jpa find (memberld); //조회
```
2. 유지보수: SQL을 직접 다루면 엔티티에 필드 하나만 추가해도 관련된 관련된 CRUD SQL 기능과 JDBC API 코드를 모두 변경해야 했다.
JPA를 사용하면 이런 과정을 JPA가 대신 처리해주므로 유지보수해야 하는 코드가 줄어든다.
3. 패러다임의 불일치 해결: JPA는 상속, 연관관계, 객체 그래프 탐색, 비교하기와 같은 패러다임의 불일치 문제를 해결해준다.
4. 성능: JPA는 애플리케이션과 데이터베이스 사이에서 동작한다. 이것은 최적화 관점에서도 시도할 수 있는 것들이 많다.
```java
String memberId = "helloId";
Member memberl = jpa.find (memberId);
Member member2 = jpa.find (memberId);
```
JDBC API를 사용해서 해당 코드를 직접 작성했다면, 데이터베이스와 두 번 통신했을 것이다. 하지만 JPA를 사용한다면 SELECT SQL을 한 번만 데이터베이스에 전달하고 두번째는 조회환 회원 객체를 재사용한다.
5. 데이터 접근 추상화와 벤더 독립성: 관계형 데이터베잇느느 같은 기능도 벤더마다 사용법이 다른 경우가 많다. 단적인 예로 페이징 처리는 데이터베이스므다 달라서 사용법을 각각 배워야한다. JPA는 애플리케이션과 데이터베이스 추상화된 데이터 접근 계층을 제공해서, 애플리케이션이 특정 데이터베으시 기술에 종속되지 않도록 한다.




