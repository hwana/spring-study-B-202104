# 3.4 컨텍스트와 DI

## 3.4.1) JdbcContext 의 분리

- 전략 패턴의 구조 ⇒ UserDao의 메소드 =  클라이언트 
                              / 익명 내부 클래스로 만들어지는 것 = 개별적인 전략

                                 / jdbcContextWithStatementStrategy() = 컨텍스트

<클래스 분리>

```java
//JDBC 작업 흐름을 분리해서 만든 JdbcContext 클래스
package springbook.user.dao;

public class JdbcContext{
	private DataSource dataSource;
	public void setDataSource(DataSource dataSource){
	   this.dataSource = dataSource;
	}
	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException{
			Connection c = null;
	PreparedStatement ps = null;

try {
  c = dataSource.getConnection();
	ps = c.prepareStatement("delete from users");
	ps.executeUpdate();
	//여기서 예외가 발생하면 바로 메소드 실행이 중단된다
	} catch(SQLException e){
		throw e;
	} finally{
		if(ps !=null){
			try {ps.close();} catch(SQLException e) {}	
		}
			if(c !=null){
			try {c.close();} catch(SQLException e) {}	
		}
	}
	}
			
	}
}
```

```java
//Jdbccontext 를 DI 받아서 사용하도록 만든 UserDao

public class UserDao{
	private JdbcContext jdbccontext;
	public void setJdbccontext(JdbcContext jdbcContext){
	 this.jdbcContext = jdbcContext;
	}
	public void add(final User user) throws SQLException{
		this.jdbcContext.workWithStatementStrategy{ new StatementStrategy(){.....}};
		}
	public void deleteAll() throws SQLException{
		this.jdbcContext.workWithStatementStrategy{
			new StatementStrategy(){.....}	
	}		
}
}
```

<빈 의존 관계 변경>

- 새롭게 설정된 오브젝트 간의 의존관계를 스프링 설정에 적용
- xml 이야기라 생략하겠음

![3%204%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%A6%E1%86%A8%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%E1%84%8B%E1%85%AA%20DI%207ffdf7e09c77452c9deb8eaf0101fdcc/Untitled.png](3%204%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%A6%E1%86%A8%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%E1%84%8B%E1%85%AA%20DI%207ffdf7e09c77452c9deb8eaf0101fdcc/Untitled.png)

## 3.4.2) JdbcContext 의 특별한 DI

- UserDao 는 인터페이스를 거치지 않고 코드에서 바로 JdbcContext 클래스를 사용하고 잇음
- UserDao와 JdbcContext는 클래스 레벨에서 의존관계가 결정됨
- 비록 런타임시에 DI방식으로 외부에서 오브젝트를 주입해주는 방식을 사용하긴 했지만, 의존 오브젝트의 구현 클래스를 변경할 수 없다
- 

<스프링 빈으로 DI>

- 이렇게 인터페이스를 사용하지 않고 DI를 적용하는 것은 문제가 있지 않을까?
- 스프링DI 의 기본 의도에 맞게 JdbcContext 의 메소드를 인터페이스로 뽑아내어 정의해두고, 이를 UserDao 에서 사용하게 해야 하지 않을까?
- 물론 그렇게 해도 상관없음. 그런데 그럴 필요 없음
- 의존관계 주입이라는 개념을 보면,
인터페이스를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않게 하고, 
런타임시에 의존할 오브젝트와의 관계를 다이내믹하게 주입해주는 것이 맞음
따라서, 인터페이스를 사용하지 않았다면 엄밀히 말해서 온전한 DI 라고 볼 수 없음
그러나, 스프링의 DI는 넓게 보자면 객체의 생성과 관계설정에 대한 제어권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC 개념을 포괄
그런 의미에서 JdbcContext  를 스프링을 이용해 Userdao 객체에서 사용하게 주입했다는 건 DI를 따르고 있음

<JdbcContext를 UserDao 와 DI 구조로 만들어야 할 이유>

1) JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문에 Jdbccontext는 그 자체로 변경되는 상태정보를 갖지 않음(읽기전용)
  JdbcContext는 JDBC 컨텍스트 메소드를 제공해주는 일종의 서비스 오브젝트
  그래서 싱글톤으로 등록돼서 여러 오브젝트에서 공유해 사용되는 것이 이상적

2) JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문
    JdbcContext는 dataSource 프로퍼티를 통해 DataSource 오브젝트를 주입받도록 되어 있음
  DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록되어야 한다

  스프링이 생성하고 관리하는 IoC 대상이어야 DI에 참여할 수 있음

⇒ 여기서 중요한 건 인터페이스의 사용 여부 
⇒ 인터페이스를 사용하지 않았을까?

⇒ 인터페이스가 없다는 건 UserDao 가 JdbcContext가 강한 응집도

⇒ 이런 경우는 싱글톤으로 만드는 것과 JdbcContext 에 대한 DI 필요성을 위해 스프링의 빈으로 등록해서 UserDao 에 DI 되도록 만드는게 좋음

<코드를 이용하는 수동 DI>

- JdbcContext를 스프링의 빈으로 등록해서 UserDao 에  DI 하는 대신 사용할 수 잇는 방법

    = UserDao 내부에 직접 DI를 적용하는 방법⇒ 싱글톤으로 만들려는 것은 포기

- 조금만 타협해서 DAO 마다 하나의 JdbcContext 오브젝트를 갖고 잇게 하는 것
- JdbcContext에는 내부에 두는 상태 정보가 없기에 메모리에 부담 거의 없음
- GC에 대한 부담도 없음
- JdbcContext를 스프링빈으로 만들지 않았기 때문에 Jdbccontext 제어권은 Userdao가 갖는 것이 적당
- 남은 문제 = JdbcContext를 스프링 빈으로 등록해서 사용했던 두번째 이유
- UserDao 에서 Jdbccontext를 직접 생성해서 사용하는 경우에 여전히 Jdbccontext는 dataSource 타입 빈을 다이내믹하게 주입받아서 사용해야 한다. 그렇지 않으면 DataSource 구현 클래스를 자유롭게 바꿔가면서 적용할 수 없다(DI컨테이너를 통해 DI받을 수 없음)
- 이런 경우에 JdbcContext에 대한 제어권을 갖고 생성과 관리를 담당하는 UserDao 에게 DI까지 맡기는 것
- UserDao 가 임시로 DI 컨테이너처럼 동작하게 만듬

![3%204%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%A6%E1%86%A8%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%E1%84%8B%E1%85%AA%20DI%207ffdf7e09c77452c9deb8eaf0101fdcc/Untitled%201.png](3%204%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%A6%E1%86%A8%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%E1%84%8B%E1%85%AA%20DI%207ffdf7e09c77452c9deb8eaf0101fdcc/Untitled%201.png)

```java
//jdbcContext 생성과 DI 작업을 수행하는 setDataSource() 메소드
public class UserDao{
	private JdbcContext jdbcContext;
	public void setDataSource(DataSource datasource){// 수정자 메소드이면서 JdbcContext 에 대한 생성, DI작업을 동시에 수행
		this.jdbcContext = new JdbcContext(); // JdbcContext 생성 (IoC)
		this.jdbcContext.setDataSource(dataSource); // 의존 오브젝트 주입(DI)
		this.dataSource = dataSource; // 아직 JdbcContext를 적용하지 않은 메소드를 위해 저장
	}
	
}
```

- JdbcContext와 같이 인터페이스를 사용하지 않고 DAO 와 밀접한 관계를 갖는 클래스를 DI에 적용하는 2가지 방법을 알아봄
1. 인터페이스를 사용하지 않는 클래스와의 의존관계지만 스프링의 DI 를 이용하기 위해 빈으로 등록해서 사용하는 방법
- 장점 : 오브젝트 사이의 실존 의존 관계가 설정파일에 명확하게 드러난다
- 단점 : DI의 근본적인 원칙에 부합하지 않는 구체적인 클래스와의 관계가 설정에 직접 노출

2. DAO코드를 이요해 수동으로 DI하는 방법

- 장점 : JdbcContext와 UserDao 의 그 관계를 외부에는 드러내지 않음
- 단점 : 여러 오브젝트가 사용하더라도 싱글톤으로 만들 수 없고, DI 작업을 위한 부가적인 코드가 필요