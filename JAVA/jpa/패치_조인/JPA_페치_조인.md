[yeon-dodo](https://yeon-dodo.tistory.com/10)

# 페치 조인(fetch join)
- SQL 조인 종류는 아니고, JPQL에서 성능 최적화를 위해 제공하는 기능.
- 연관된 엔티티나 컬렉션을 SQL 한번에 함꼐 조회하는 기능이.

(즉시로딩과 동일함. 그치만 명시적으로 동적인 타이밍에 정할 수 있다는 특징이 있음.)

## 엔티티 페치 조인
(예시 엔티티)
```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;
    private String username;
    private int age;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    public Member(String username, int age, Team team) {
        this.username = username;
        this.age = age;
        if (team != null) {
            changeTeam(team);
        }
    }

    public void changeTeam(Team team) {
        this.team = team;
        team.getMembers().add(this);
    }
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();

    public Team(String name) {
        this.name = name;
    }
}
```

페치 조인 문법 <br>

[JPQL]<br>
`select m from Member m join fetch m.team`

[SQL]<br>
`select M.*, T.* from member M inner join team T on m.team_id=t.id`


```java

String jpql = "select m from Member m join fetch m.team"; 
List<Member> members = em.createQuery(jpql, Member.class) .getResultList();

for (Member member : members) {
    //페치 조인으로 회원과 팀을 함께 조회해서 지연 로딩
    System.out.println(
            "username = " + member.getUsername() + ", " + 
            "teamName = " + member.getTeam().name()); }
}
```

결과<br> 
username = 회원1, teamname = 팀A <br>
username = 회원2, teamname = 팀A <br>
username = 회원3, teamname = 팀B <br>



## 컬렉션 페치 조인
- 일대다 관계, 컬렉션 페치 조인
- 데이터가 뻥튀기 되서 가져와진다.
  (member 데이터가 뻥튀기되서 가져와진다.)

[JPQL]<br>
`select t from Team t join fetch t.members where t.name = "teamA"`

[SQL]<br>
`select T.*, M.* from Team T inner join member M on T.id = M.team_id where t.name = "teamA"`


## 페치 조인과 distinct
- sql 문에서 distinct를 사용해봤자 데이터가 다르기 때문에 중복이 제거 되지 않는다.
- 그렇지만 jpql에 distinct를 사용하면 DISTINCT가 추가로 **애플리케이션에서 중복 제거**를 시도한다.
- 같은 식별자를 가진 Team 엔티티를 제거 한다.
  (member 데이터가 뻥튀기 되지 않는다.)


## 페치조인과 일반 조인의 차이
- 일반 조인 실행시 연관된 엔티티를 함께 조회하지 않는다.
- 페치조인을 사용할때만 연관된 엔티티도 함께 조회가 된다. (즉시 로딩 -> 엔티티에 lazy로딩을 걸어둬도 fetch join 사용시 더 우선시 된다.)
- 페치조인은 객체 그래프를 sql 한번에 조회하는 개념

회원을 조회하면서 연관된 팀을 함께 조회
- inner join 했을 경우
  `select member.* from member inner join team on team.id = member.id;`

- fetch join 했을 경우
  `select member.*, team.* from inner join team on team.id = member.id;`

## 페치 조인의 한계
- 페치 조인 대상에는 별칭을 줄 수 없다.
- 둘 이상의 컬렌션은 페치 조인 할 수 없다.
- 켈렉션을 페치 조인하면 페이징 API를 사용할 수 없다.
- 연관된 엔티티들을 sql 한번으로 조회가 가능함 - 성능 최적화

## 정리
- 객체 그래프를 유지할때 사용하면 효과적이다.
- 여러 테이블을 조인해서 엔티티가 가진 모양이 아닌 전혀 다른 결과는 일반 조인을 사용하고 필요한 
데이터들만 조회해서 DTO로 반환하는것이 효과적이다.
- (lazy 로딩으로 인한) N+1 문제 해결된다.
  - n+1 이란?
    - 조회 시 1개의 쿼리를 생각하고 설계를 했으나 나오지 않아도 되는 조회의 쿼리가 N개가 더 발생하는 문제. 
    - 연관 관계가 설정된 엔티티를 조회할 경우에 조회된 데이터 갯수(n) 만큼 연관관계의 조회 쿼리가 추가로 발생하여 데이터를 읽어오는 현상

