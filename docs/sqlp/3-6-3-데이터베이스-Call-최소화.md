---
title: 3-6-3. 데이터베이스 Call 최소화
tags: 
---

# 과목3. SQL 고급 활용 및 튜닝
# 제6장 고급 SQL 튜닝
# 제3절 데이터베이스 Call 최소화

## 1. 데이터베이스 Call 종류

### 가. SQL 커서에 대한 작업 요청에 따른 구분

>```sql
>SELECT CUST_NM
>     , BIRTHDAY
>  FROM CUSTOMER
> WHERE CUST_ID = :cust_id 
>```
>
>| call  |count| cpu |elapsed|disk|query|current|rows |
>|-------|----:|----:|------:|---:|----:|------:|----:|
>|Parse  |1    |0.00 |0.00   |0   |0    |0      |0    |
>|Execute|5000 |0.18 |0.14   |0   |0    |0      |0    |
>|Fetch  |5000 |0.21 |0.25   |0   |20000|0      |50000|
>|-----  |-----|-----|------ |----|-----|------ |-----|
>|total  |10001|0.39 |0.40   |0   |20000|0      |50000|
>

#### 1) Parse Call

> SQL 파싱을 요청하는 Call

#### 2) Execute Call

> SQL 실행을 요청하는 Call

#### 3) Fetch Call

> SELECT문의 결과 데이터 전송을 요청하는 Call

### 나. Call 발생 위치에 따른 구분

#### 1) User Call

DBMS 외부로부터 요청되는 Call을 말한다. 동시 접속자 수가 많은 Peak 시간대에 시스템 확장성을 떨어뜨리는 가장 큰 요인 중 한 가지는 User Call이다. User Call이 많이 발생하도록 개발된 프로그램은 결코 성능이 좋을 수 없고, 이는 개발자의 기술력에 의해서도 좌우되지만 많은 경우 애플리케이션 설계와 프레임워크 기술구조에 기인한다. 이를테면, Array Processing을 제대로 지원하지 않는 프레임워크, 화면 페이지 처리에 대한 잘못 설계된 표준가이드, 사용자 정의 함수/프로시저에 대한 무조건적인 제약 등이 그것이다. 그리고 프로시저 단위 모듈을 지나치게 잘게 쪼개서 SQL을 건건이 호출하도록 설계하는 것도 대표적이다. DBMS 성능과 확장성(Scalability)을 높이려면 User Call을 최소화하려는 노력이 무엇보다 중요하며, 이를 위해 아래와 같은 기술요소를 적극적으로 활용해야만 한다.<br>

* Loop 쿼리를 해소하고 집합적 사고를 통해 One SQL로<br>
* Array Processing : Array 단위 Fetch, Bulk INSERT/UPDATE/DELETE<br>
* 부분범위처리 원리 활용<br>
* 효과적인 화면 페이지 처리<br>
* 사용자 정의 함수/프로시저/트리거의 적절한 활용

#### 2) Recursive Call

DBMS 내부에서 발생하는 Call을 말한다. SQL 파싱과 최적화 과정에서 발생하는 데이터 딕셔너리 조회, 사용자 정의 함수/프로시저 내에서의 SQL 수행이 여기에 해당한다. Recursive Call을 최소화하려면, 바인드 변수를 적극적으로 사용해 하드파싱 발생횟수를 줄여야 한다. 그리고 사용자 정의 함수와 프로시저가 어떤 특징을 가지며 내부적으로 어떻게 수행되는지를 잘 이해하고 시의 적절하게 사용해야만 한다. 무조건 사용하지 못하도록 제약하거나 무분별하게 사용하지 말아야 한다는 뜻이다.

## 2. 데이터베이스 Call과 성능

### 가. One SQL 구현의 중요성

루프를 돌면서 여러 작업을 반복 수행하는 프로그램을 One SQL로 구현했을 때 얼마나 놀라운 성능 개선 효과가 나타나는지 경험해 본 적이 있는가? 있다면 그 원리가 무엇이라고 생각하는가? 그것은 방금 설명한 데이터베이스 Call 횟수를 줄인 데에 있다. 1번과 10번, 10번과 100번의 차이는 크지 않지만 1번과 10만 번, 1번과 100만 번의 차이는 실로 엄청나다. 아래 JAVA 소스를 예제로 살펴보자.

>```java
>public class JavaLoopQuery {
>    public static void
>    INSERTData(
>        Connection con, String param1, String param2, String param3, long param4
>    ) throws Exception {
>        String SQLStmt = "INSERT INTO 납입방법별_월요금집계" + "(고객번호, 납입월, 납입방법코드, 납입금액) " + "VALUES(?, ?, ?, ?)";
>        PreparedStatement st = con.prepareStatement(SQLStmt);
>        st.setString(1, param1);
>        st.setString(2, param2);
>        st.setString(3, param3);
>        st.setLong(4, param4);
>        st.execute();
>        st.close();
>    }
>    public static void execute(Connection con, String input_month) throws Exception {
>        String SQLStmt = "SELECT 고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷 " + "
>        FROM 월요금납부실적 " + "
>        WHERE 납입월 = ? ";
>        PreparedStatement stmt = con.prepareStatement(SQLStmt);
>        stmt.setString(1, input_month);
>        ResultSet rs = stmt.executeQuery();
>        while (rs.next()) {
>            String 고객번호 = rs.getString(1);
>            String 납입월 = rs.getString(2);
>            long 지로 = rs.getLong(3);
>            long 자동이체 = rs.getLong(4);
>            long 신용카드 = rs.getLong(5);
>            long 핸드폰 = rs.getLong(6);
>            long 인터넷 = rs.getLong(7);
>            if (지로 > 0) INSERTData(con, 고객번호, 납입월, "A", 지로);
>            if (자동이체 > 0) INSERTData(con, 고객번호, 납입월, "B", 자동이체);
>            if (신용카드 > 0) INSERTData(con, 고객번호, 납입월, "C", 신용카드);
>            if (핸드폰 > 0) INSERTData(con, 고객번호, 납입월, "D", 핸드폰);
>            if (인터넷 > 0) INSERTData(con, 고객번호, 납입월, "E", 인터넷);
>        }
>        rs.close();
>        stmt.close();
>    }
>    static Connection getConnection() throws Exception {
>        ……}
>    static void releaseConnection(Connection con) throws Exception {
>        ……}
>    public static void main(String[] args) throws Exception {
>        Connection con = getConnection();
>        execute(con, "200903");
>        releaseConnection(con);
>    }
>}
>```

만약 처리해야 할 월요금납부실적이 10만 건이면 이 테이블에 대한 Fetch Call이 10만 번(뒤에서 설명할 Array 단위 Fetch 기능을 이용하지 않을 때), 납입방법별_월요금집계 테이블로의 INSERT를 위한 Parse Call과 Execute Call이 각각 최대 50만 번, 따라서 최대 110만 번의 데이터베이스 Call이 발생할 수 있다. 위 프로그램을 DBMS 내에서 수행되는 사용자 정의 프로시저로 개발하면 네트워크 트래픽 없는 Recursive Call만 발생하므로 제법 빠르게 수행될 것이다. 하지만 위와 같이 JAVA나 C, VB, Delphi 등으로 개발된 애플리케이션에선 수행 성능에 심각한 문제가 나타난다.<br>

실제 수. 자세히 분석해 보면 그 이유를 알 수 있는데, 대부분 시간을 네트워크 구간에서 소비(그 중 일부는 애플리케이션 단에서 소비한 시간일 것임)하거나 데이터베이스 Call이 발생할 때마다 OS로부터 CPU와 메모리 리소스를 할당받으려고 기다리면서 소비한다.<br>

위 프로그램을 아래와 같이 One SQL로 통합하면 1~2초 내에 수행되는 것을 확인할 수 있다. 원리는 최대 110만 번 발생할 수 있는 데이터베이스 Call을 단 2회(Parse Call 1회, Execute Call 1회)로 줄인 데에 있다.

>```java
>public class JavaOneSQL {
>    public static void execute(Connection con, String input_month) throws Exception {
>        String SQLStmt = "
>        INSERT INTO
>        납입방법별_ 월요금집계 " + "(납입월, 고객번호, 납입방법코드, 납입금액)
>        " + "
>        SELECT x.납입월, x.고객번호, CHR(64 + Y.NO) 납입방법코드 " + ", DECODE(Y.NO, 1, 지로, 2, 자동이체, 3, 신용카드, 4, 핸드폰, 5, 인터넷)
>        " + "
>        FROM 월요금납부실적x, (SELECT LEVEL NO FROM DUAL CONNECT BY LEVEL <= 5) y " + "
>        WHERE x.납입월 = ? " + "
>        AND y.NO IN(DECODE(지로, 0, NULL, 1), DECODE(자동이체, 0, NULL, 2)
>            " + ", DECODE(신용카드, 0, NULL, 3), DECODE(핸드폰, 0, NULL, 4)
>            " + ", DECODE(인터넷, 0, NULL, 5))
>        " ;
>        PreparedStatement stmt = con.prepareStatement(SQLStmt);
>        stmt.setString(1, input_month);
>        stmt.executeQuery();
>        stmt.close();
>    }
>    static Connection getConnection() throws Exception {
>        ……}
>    static void releaseConnection(Connection con) throws Exception {
>        ……}
>    public static void main(String[] args) throws Exception {
>        Connection con = getConnection();
>        execute(con, "200903");
>        releaseConnection(con);
>    }
>}
>```

### 나. 데이터베이스 Call과 시스템 확장성

데이터베이스 Call은 개별 프로그램의 수행 속도에 큰 영향을 미칠 뿐만 아니라 궁극적으로 시스템 전체의 확장성에 영향을 미친다. 인터넷 쇼핑몰에서 조회한 상품 중 일부를 선택한 후 위시리스트(WishList)에 등록하는 프로그램을 예로 들어 보자. '위시리스트' 버튼을 클릭할 때 수행되는 프로그램을 아래 처럼 구현했다면, 선택한 상품이 5개일 때 메소드(method)도 5번 호출해야 하기 때문에 Parse Call과 Execute Call이 각각 5번씩 발생한다.

>```java
>void
>INSERTWishList(String p_custid, String p_goods_no) {
>    SQLStmt = "INSERT INTO wishlist " 
>            + "SELECT custid, goods_no "
>            + "  FROM cart " 
>            + " WHERE custid = ? "
>            + " AND goods_no = ? ";
>    stmt = con.preparedStatement(SQLStmt);
>    stmt.setString(1, p_custid);
>    stmt.setString(2, p_goods_no);
>    stmt.execute();
>}
>```

반면, 아래와 같이 구현했다면 메소드를 1번만 호출하기 때문에 Parse Call과 Execute Call도 각각 한 번씩만 발생한다. 단적으로 말해, 24시간 내내 이 프로그램만 수행된다면 시스템이 5배의 확장성을 갖는 것이며, AP 설계가 DBMS 성능을 좌우하는 중요한 요인임을 보여주는 사례라고 하겠다.

>```java
>void
>INSERTWishList(String p_custid, String[] p_goods_no) {
>    SQLStmt = "INSERT INTO wishlist "
>            + "SELECT custid, goods_no "
>            + "  FROM cart "
>            + " WHERE custid = ? "
>            + "  AND goods_no in ( ? , ? , ? , ? , ? )";
>    stmt = con.preparedStatement(SQLStmt);
>    stmt.setString(1, p_custid);
>    for ( int i = 0; i < 5; i++ ) {
>        stmt.setString(i + 2, p_goods_no[i]);
>    }
>    stmt.execute();
>}
>```

## 3. Array Processing 활용

Array Processing 기능을 활용하면 한 번의 SQL(INSERT/UPDATE/DELETE) 수행으로 다량의 레코드를 동시에 처리할 수 있다. 이는 네트워크를 통한 데이터베이스 Call을 줄이고, 궁극적으로 SQL 수행시간과 CPU 사용량을 획기적으로 줄여준다. 앞서 보았던 '납입방법별_월요금집계' 테이블 가공 사례에 Array Processing 기법을 적용하면 다음과 같다.

>```java
>public class JavaArrayProcessing {
>    public static void INSERTData(Connection con, PreparedStatement st, String param1, String param2, String param3, long param4) throws Exception {
>        st.setString(1, param1);
>        st.setString(2, param2);
>        st.setString(3, param3);
>        st.setLong(4, param4);
>        st.addBatch();
>    }
>
>    public static void execute(Connection con, String input_month)
>    throws Exception {
>        long rows = 0;
>        String SQLStmt1 = "SELECT 고객번호, 납입월" +
>            ", 지로, 자동이체, 신용카드, 핸드폰, 인터넷" +
>            "  FROM 월요금납부실적" +
>            " WHERE 납입월= ? ";
>
>        String SQLStmt2 = "INSERT INTO 납입방법별_월요금집계 " +
>            "(고객번호, 납입월, 납입방법코드, 납입금액) " +
>            "VALUES(?, ?, ?, ?)";
>
>        con.setAutoCommit(false);
>
>        PreparedStatement stmt1 = con.prepareStatement(SQLStmt1);
>        PreparedStatement stmt2 = con.prepareStatement(SQLStmt2);
>        stmt1.setFetchSize(1000);
>        stmt1.setString(1, input_month);
>        ResultSet rs = stmt1.executeQuery();
>        while (rs.next()) {
>            35 String 고객번호 = rs.getString(1);
>            String 납입월 = rs.getString(2);
>            long 지로 = rs.getLong(3);
>            long 자동이체 = rs.getLong(4);
>            long 신용카드 = rs.getLong(5);
>            long 핸드폰 = rs.getLong(6);
>            long 인터넷 = rs.getLong(7);
>
>            if (지로    > 0) INSERTData(con, stmt2, 고객번호, 납입월, "A", 지로);
>            if (자동이체 > 0) INSERTData(con, stmt2, 고객번호, 납입월, "B", 자동이체);
>            if (신용카드 > 0) INSERTData(con, stmt2, 고객번호, 납입월, "C", 신용카드);
>            if (핸드폰  > 0) INSERTData(con, stmt2, 고객번호, 납입월, "D", 핸드폰);
>            if (인터넷  > 0) INSERTData(con, stmt2, 고객번호, 납입월, "E", 인터넷);
>            if (++rows % 1000 == 0) stmt2.executeBatch();
>        }
>        rs.close();
>        stmt1.close();
>
>        stmt2.executeBatch();
>        stmt2.close();
>
>        con.commit();
>        con.setAutoCommit(true);
>    }
>
>    static Connection getConnection() throws Exception {}
>    static void releaseConnection(Connection con) throws Exception {
>        ……}
>
>    public static void main(String[] args) throws Exception {
>        Connection con = getConnection();
>        execute(con, "200903");
>        releaseConnection(con);
>    }
>}
>```

INSERT할 데이터를 계속 Array에 담기만 하다가(12번 라인) 1,000건 쌓일 때마다 한 번씩 executeBatch를 수행하는 부분(58번 라인)을 주의 깊게 살펴보기 바란다. SELECT 결과집합을 Fetch할 때도 1,000개 단위로 Fetch하도록 조정(31번 라인)하였다. 위 프로그램을 수행해 보면 One SQL로 구현할 때와 거의 비슷한 속도를 보인다. One SQL로 통합했을 때 나타나는 극적인 성능개선 효과가 데이터베이스 Call 횟수를 줄이는 데 있음을 여기서도 알 수 있다.<br>

대용량 데이터를 처리하는 데 있어 Array Processing은 필수적인데, 그 효과를 극대화하려면 연속된 일련의 처리과정이 모두 Array 단위로 진행돼야 한다. 이를테면, Array 단위로 수천 건씩 아무리 빠르게 Fetch 하더라도 다음 단계에서 수행할 INSERT가 건건이 처리된다면 그 효과가 크게 반감되며, 반대의 경우도 마찬가지다. 이해를 돕기 위해 PL/SQL을 이용해 데이터를 Bulk로 1,000건씩 Fetch해서 Bulk로 INSERT하는 예제를 보이면 다음과 같다.

>```sql
>DECLARE l_fetch_size NUMBER DEFAULT 1000;
> -- 1,000건씩 Array 처리
> CURSOR c IS
> SELECT empno
>     , ename
>     , job
>     , sal
>     , deptno
>     , hiredate
>  FROM emp;
> … 
> BEGIN OPEN C;
> LOOP FETCH c BULK COLLECT INTO p_empno, p_ename, p_job, p_sal, p_deptno, p_hiredate LIMIT l_fetch_size;
> FORALL i IN p_empno.first..p_empno.last 
>INSERT INTO
>  emp2
>VALUES
>  (
>    p_empno (i)
>     , p_ename (i)
>     , p_job (i)
>     , p_sal (i)
>     , p_deptno (i)
>     , p_hiredate (i)
>  );
> EXIT WHEN c%NOTFOUND;
> END LOOP;
> CLOSE C;
>```

BULK COLLECT와 FORALL 구문에 대한 자세한 설명은 매뉴얼을 참조하기 바란다. 그리고 Array Processing 기법을 지원하는 인터페이스가 개발 언어마다 다르므로 API를 통해 반드시 확인하고 적극적으로 활용하기 바란다.

## 4. Fetch Call 최소화

### 가. 부분범위처리 원리

현재 자신이 사용하고 있는 시스템에서 가장 큰 테이블을 아무 조건절 없이 쿼리해 보자. 테이블에 데이터가 아무리 많아도 엔터를 누르자마자 결과가 출력되기 시작하는 것을 볼 수 있을 것이다. SQL Server를 사용 중이라면 쿼리 분석기(Query Analyzer)의 Text 모드에서 테스트하기 바란다. 데이터 양과 무관하게 이처럼 빠른 응답속도를 보일 수 있는 원리가 무엇일까?<br>

![](../images_files/SQL_269.jpg)

집을 짓는 공사장을 예로 들어 보자. [그림 Ⅲ-1-9]를 보면 미장공이 시멘트를 이용해 벽돌을 쌓는 동안 운반공은 수레를 이용해 벽돌을 일정량씩 나누어 운반하고 있다. 쌓여 있는 벽돌을 한 번에 실어 나를 수 없기 때문이다. 운반공은 미장공이 벽돌을 더 가져오라는 요청(→ Fetch Call)이 있을 때만 벽돌을 실어 나른다. 추가 요청이 없으면 운반작업은 거기서 멈춘다. DBMS도 이처럼 데이터를 클라이언트에게 전송할 때 일정량씩 나누어 전송한다. Oracle의 경우 ArraySize(또는 FetchSize) 설정을 통해 운반단위를 조절할 수 있다. 예를 들어, SQL\*PLUS에서 ArraySize를 변경하는 명령어는 다음과 같다.

>set arraysize 100<br>

그리고 아래는 ArraySize를 100으로 설정한 상태에서 SELECT 문장을 수행할 때의 SQL 트레이스 결과다.

>| call  |count| cpu |elapsed|disk |query|current| rows |
>|-------|----:|----:|------:|----:|----:|------:|-----:|
>|Parse  |1    |0.00 |0.00   |0    |0    |0      |0     |
>|Execute|1    |0.00 |0.02   |2    |2    |0      |0     |
>|Fetch  |301  |0.14 |0.18   |9    |315  |0      |30000 |
>|-----  |---- |-----|------ |-----|-----|-----  |------|
>|total  |303  |0.14 |0.20   |11   |317  |0 30000|      |
>

30,000개 로우를 읽기 위해 Fetch Call이 301번 발생한 것만 보고도 ArraySize가 100으로 설정된 상태에서 수행된 쿼리임을 짐작할 수 있다. SQL Server의 경우 네트워크 패키지 크기로 운반단위를 조절하는데, 쿼리 분석기(Query Analyzer) 옵션에서 '네트워크 패키지 크기' 항목을 보면 기본 값이 4,096 바이트로 설정된 것을 볼 수 있다. (참고로, Oracle도 Array 크기의 데이터를 내부적으로 다시 SDU(Session Data Unit, Session 레이어), TDU(Transport Data Unit, Transport 레이어) 단위로 나누어 전송한다. ArraySize를 작게 설정하면 하나의 네트워크 패킷에 담아 전송하겠지만, 크게 설정하면 여러 개 패킷으로 나누어 전송할 수 밖에 없음은 당연하다.) 전체 결과집합 중 아직 전송하지 않은 분량이 많이 남아있어도 클라이언트로부터 추가 Fetch Call을 받기 전까지 서버는 그대로 멈춰 서서 기다린다. 이처럼 쿼리 결과집합을 전송할 때, 전체 데이터를 연속적으로 전송하지 않고 사용자로부터 Fetch Call이 있을 때마다 일정량씩 나누어서 전송하는 것을 이른바 '부분범위처리'라고 한다. OLTP성 업무에서는 쿼리 결과집합이 아주 많아도 그 중 일부만 Fetch해서 보여주고 멈춰도 되는 업무가 많다. 화면상에서 수천 수만 건을 일일이 스크롤하며 데이터를 보는 사용자는 거의 없기 때문이다. 사용자가 '다음' 버튼을 클릭하거나 그리드 스크롤을 내릴 때만 추가적인 Fetch Call을 일으켜 필요한 만큼 더 가져오면 된다. 물론 커서를h Call이 아니라 별도의 쿼리 수행을 통해 나머지??른 개념으로 이해해야 한다.) 이런 화면 처리가 가능한 업무라면, 출력 대상 레코드가 많을수록 Array를 빨리 채울 수 있어 쿼리 응답 속도도 그만큼 빨라진다. 잘 설계된 인덱스와 부분범위처리 방식을 이용해 대용량 OLTP 환경에서 극적인 성능개선 효과를 얻을 수 있는 원리가 여기에 숨어있다. 참고로, 출력 대상 레코드가 많을수록 응답 속도가 빨라지는 것은 부분범위처리가 가능한 업무에만 해당한다. 결과집합 전체를 Fetch 하는 DW/OLAP성 업무나 서버 내에서 데이터를 가공하는 프로그램에선 결과집합이 많을수록 더 빨라지는 일은 있을 수 없다. DBMS 서버가 부분범위처리 방식으로 데이터를 전송하는데도 어떤 개발팀은 결과를 모두 Fetch 하고서야 출력을 시작하도록 애플리케이션을 개발한다. 또 어떤 개발팀은 첫 화면부터 빠르게 출력을 시작하도록 하지만 사용자의 명시적인 Fetch 요청이 없어도 백그라운드에서 계속 Fetch Call을 일으켜 클라이언트 캐시에 버퍼링하도록 개발하기도 한다. SQL Server 개발 환경에서 가장 많이 사용되는 쿼리 분석기의 Grid 모드가 전자에 해당하기 때문에 SQL Server 사용자들은 부분범위처리 원리를 설명해도 쉽게 이해하지 못하는 경향이 있다. Oracle을 위한 쿼리 툴 중에도 부분범위처리를 활용하지 않고 결과집합 전체를 모았다가 출력하는 툴이 있다. 이것은 클라이언트 쿼리 툴의 특성일 뿐이며 모든 DBMS는 데이터를 일정량씩 나누어 전송한다. 여기서 강조하고자 하는 바는, 불필요한 데이터베이스 Call과 네트워크 부하를 일으켜선 결코 고성능 데이터베이스 애플리케이션을 구축하기 힘들다는 사실이다. 팀 단위의 소규모 애플리케이션을 개발 중이라면 모르겠지만, 전사적 또는 전국 단위 서비스를 제공하는 애플리케이션을 개발 중이라면 본 장에서 설명하는 아키텍처 기반 튜닝 원리를 정확히 이해하고 적용하려고 노력해야 한다.

### 나. ArraySize 조정에 의한 Fetch Call 감소 및 블록 I/O 감소 효과

지금까지 설명한 부분범위처리 원리를 이해했다면, 네트워크를 통해 전송해야 할 데이터량에 따라 ArraySize를 조절할 필요가 있음을 직감했을 것이다. 예를 들어, 대량 데이터를 파일로 내려 받는다면 어차피 전체 데이터를 전송해야 하므로 가급적 값을 크게 설정해야 한다. ArraySize를 조정한다고 전송해야 할 총량이 줄지는 않지만, Fetch Call 횟수를 그만큼 줄일 수 있다. 반대로 앞쪽 일부 데이터만 Fetch하다가 멈추는 프로그램이라면 ArraySize를 작게 설정하는 것이 유리하다. 많은 데이터를 읽어 전송하고도 정작 사용되지 않는 비효율을 줄일 수 있기 때문이다. ArraySize를 증가시키면 네트워크 부하가 줄어들 뿐만 아니라 서버 프로세스가 읽어야 할 블록 개수까지 줄어드는 일거양득의 효과를 얻게 된다. ArraySize를 조정하는데 왜 블록 I/O가 줄어드는 것일까? [그림 Ⅲ-1-10]을 보면서 설명해 보자.<br>

![](../images_files/SQL_270.jpg)

[그림 Ⅲ-1-10]처럼 10개 행으로 구성된 3개의 블록이 있다고 하자. 총 30개 레코드이므로 ArraySize를 3으로 설정하면 Fetch 횟수는 10이고, 블록 I/O는 12번이나 발생하게 된다. 왜냐하면, 10개 레코드가 담긴 블록들을 각각 4번에 걸쳐 반복 액세스해야 하기 때문이다. 그림에서 보듯, 첫 번째 Fetch에서 읽은 1번 블록을 2~4번째 Fetch에서도 반복 액세스하게 된다. 2번 블록은 4~7번째 Fetch, 3번 블록은 7~10번 Fetch에 의해 반복적으로 읽힌다. 만약 ArraySize를 10으로 설정한다면 3번의 Fetch와 3번의 블록 I/O로 줄일 수 있다. 그리고 ArraySize를 30으로 설정하면 Fetch 횟수는 1로 줄어든다. ArraySize를 늘리면서 Fetch Count와 블록 I/O를 측정해 보면, 실제 [그림 Ⅲ-1-11]과 같은 그래프를 얻을 수 있다. 즉 ArraySize와 Fetch Count 및 블록 I/O는 반비례 관계를 보인다.<br>

![](../images_files/SQL_271.jpg)

[그림 Ⅲ-1-11]에서 눈에 띄는 것은, ArraySize를 키운다고 해서 Fetch Count와 블록 I/O가 같은 비율로 줄지 않는다는 점이다. 따라서 무작정 크게 설정한다고 좋은 것은 아니며 일정 크기 이상이면 오히려 리소스만 낭비하게 된다. 데이터 크기에 따라 다를 텐데, 위 데이터 상황에서는 100 정도로 설정하는 게 적당해 보인다.<br>

SQL\*PLUS 이외의 프로그램 언어에서 Array 단위 Fetch 기능을 활용하는 방법을 살펴보자. Oracle PL/SQL에서 커서를 열고 레코드를 Fetch 하면, (3항 Array Processing에서 보았던 Bulk Collect 구문을 사용하지 않는 한) 9i까지는 한 번에 한 로우씩만 처리(Single-Row Fetch)했었다. 10g부터는 자동으로 100개씩 Array Fetch가 일어나지만, 아래 처럼 커서의 Open, Fetch, Close가 내부적으로 이루어지는 Cursor FOR Loop 구문을 이용할 때만 작동한다는 사실을 기억하기 바란다.

>for item in cursor loop …… end loop;

JAVA에서는 어떻게 ArraySize를 조정하는지 살펴보자.

>```java
>String sql = "SELECT custid, name
>FROM customer ";
>PreparedStatement stmt = conn.prepareStatement(sql);
>stmt.setFetchSize(100);
>--Statement에서 조정 ResultSet rs = stmt.executeQuery();
>// rs.setFetchSize(100);
>--ResultSet에서 조정할 수도 있다.
>while(rs.next()) {
>    int empno = rs.getInt(1);
>    String ename = rs.getString(2);
>    System.out.println(empno + ":" + ename);
>}
>rs.close();
>stmt.close();
>```

setFetchSize 메소드를 이용해 FetchSize를 조정하는 예는 앞에서도 잠깐 본 적이 있다. JAVA에서 FetchSize 기본 값은 10이다. 대량 데이터를 Fetch할 때 이 값을 100~500 정도로 늘려 주면 기본 값을 사용할 때보다 데이터베이스 Call 부하를 1/10 ~ 1/50로 줄일 수 있다. 예를 들어, FetchSize를 100으로 설정했을 때 데이터를 Fetch 해 오는 메커니즘은 아래와 같다.<br>

* 최초 rs.next() 호출 시 한꺼번에 100건을 가져와서 클라이언트 Array 버퍼에 캐싱한다.<br>
* 이후 rs.next() 호출할 때는 데이터베이스 Call을 발생시키지 않고 Array 버퍼에서 읽는다.<br>
* 버퍼에 캐싱 돼 있던 데이터를 모두 소진한 후 101번째 rs.next() 호출 시 다시 100건을 가져온다.<br>
* 모든 결과집합을 다 읽을 때까지 2~3번 과정을 반복한다.

## 5. 페이지 처리 활용

부분범위처리 원리를 이용한 대용량 온라인 조회 성능 개선은 커서를 닫지 않은 상태에서 사용자가 명시적으로 요청(스크롤 바를 내리거나 '다음' 버튼을 클릭하는 등)할 때만 데이터를 Fetch 할 수 있는 개발환경에서나 가능하다. 데이터베이스와의 연??서를 계속 연 채로 결과집합을 핸들링할 수 없으므로 사용문을 수행하는 방식, 즉 페이지 처리 방식으로 구? 화면 페이지 처리를 아래와 같이 구현하기도 한다.

>```java
>void pagination(ResultSet rs, long pageNo, int pageSize) throws Exception {
>    int i = 0;
>    while (rs.next()) {
>        if (++i > (pageNo - 1) * pageSize) printRow(rs);
>        if (i == pageNo * pageSize) break;
>    }
>}
>```

우선, 사용자가 새로운 페이지 출력을 요청할 때마다 SQL을 수행한다. 매번 첫 레코드부터 읽기 시작해 현재 출력해야 할 페이지(pageNo)에 도달하면 printRow를 호출한다. printRow를 pageSize 개수만큼 호출하고 나서야 Fetch를 멈춘다. 뒤 페이지로 이동할수록 엄청나게 많은 Fetch Call을 유발하게 될 것이고, 전반적으로 이런 패턴으로 구현했다면 시스템에 얼마나 악영향을 끼칠지는 어렵지 않게 짐작할 수 있다. 그에 따른 성능 문제를 해결하려면, 페이지 처리를 서버 단에서 완료하고 최종적으로 출력할 레코드만 Fetch 하도록 프로그램을 고치는 수 밖에 없다. 위와 같이 비효율적인 방식으로 페이지 처리를 구현하는 경우가 있는가 하면 시스템 전체적으로 아예 페이지 처리 없이 개발하는 예도 종종 볼 수 있다. 업무 요건이 아예 그렇거나 짧은 개발 기간 때문이라고 하지만 대량의 결과 집합을 페이지 처리 없이 모두 출력하도록 구현했을 때 시스템 전반에 미치는 영향은 실로 엄청나다. 페이지 처리를 하지 않았을 때 발생하는 부하요인을 요약하면 아래와 같다.<br>

* 다량 발생하는 Fetch Call 부하<br>
* 대량의 결과 집합을 클라이언트로 전송하면서 발생하는 네트워크 부하<br>
* 대량의 데이터 블록을 읽으면서 발생하는 I/O 부하<br>
* AP 서버 및 웹 서버 리소스 사용량 증가<br>

이렇게 여러 가지 부하를 일으키지만 정작 사용자는 앞쪽 일부 데이터만 보고 업무처리를 완료하는 경우가 대부분이며, 쿼리 자체 성능도 문제지만 시스템 전반에 걸쳐 불필요한 리소스를 낭비하는 것이 더 큰 문제다. 이들 부하를 해소하는 열쇠는 페이지 처리에 있다.<br>

* 페이지 단위로, 화면에서 필요한 만큼만 Fetch Call<br>
* 페이지 단위로, 화면에서 필요한 만큼만 네트워크를 통해 결과 전송<br>
* 인덱스와 부분범위처리 원리를 이용해 각 페이지에 필요한 최소량만 I/O<br>
* 데이터를 소량씩 나누어 전송하므로 AP웹 서버 리소스 사용량 최소화<br>

결론적으로 말해, 조회할 데이터가 일정량 이상이고 수행빈도가 높다면 필수적으로 페이지 처리를 구현해야 한다. 효과적인 페이지 처리 구현 방안에 대해서는 5장 고급 SQL 튜닝에서 설명한다.

## 6. 분산 쿼리

부하 분산, 재해 복구, 보안 등 여러 가지 목적으로 분산 환경의 데이터베이스를 구축하게 되는데, 어디나 분산 쿼리 성능 때문에 골머리를 앓는다. 특히 원격 조인이 자주 문제시되는데, 분산 DB 간 테이블을 조인할 때 성능을 높일 방안은 무엇일까? 아래 예를 보자.

>```sql
>SELECT channel_id
>     , sum(quantity_sold) auantity_cold
>  FROM order a
>     , sales@lk_sales b
> WHERE a.order_date between :1 AND :2
>   AND b.order_no = a.order no
> GROUP BY channel_id 
>```
>
>| Rows |      Row Source Operation       |
>|-----:|---------------------------------|
>|     5|SORT GROUP BY                    |
>| 10981|NESTED LOOPS                     |
>|500000|REMOTE                           |
>| 10981|TABLE ACCESS BY INDEX ROWID ORDER|
>|500000|INDEX UNIQUE SCAN (ORDER_PK)     |
>

위 SQL과 Row Source Operation을 분석해 보면, 원격(Remote)에 있는 sales 테이블을 전송받아 order 테이블과 NL 방식으로 조인하고 있음을 알 수 있다. 50만 건이나 되는 sales 데이터를 네트워크를 통해 전송받으니 쿼리 성능이 나쁜 것은 당연하다. order 테이블도 작은 테이블은 아니지만 order_date 필터 조건이 있다. 이 조건에 해당하는 데이터만 원격으로 보내서 조인과 GROUP BY를 거친 결과집합을 전송받는다면 어떨까? 위 수행결과에서 알 수 있듯이 GROUP BY한 결과집합은 5건에 불과하므로 큰 성능 개선을 기대할 수 있다.<br>

아래는 원격 서버가 쿼리를 처리하도록 driving_site 힌트를 지정하고서 다시 수행한 결과이다.

>```sql
>SELECT /*+ driving_site(b) */
>       channel_id
>     , sum(quantity_sold) auantity_cold
>  FROM order a
>     , sales@lk_sales b
> WHERE a.order_date between :1 AND :2
>   AND b.order_no = a.order no
> GROUP BY channel_id 
>```
>
>|Rows |                            Row Source Operation                             |
>|----:|-----------------------------------------------------------------------------|
>|    5|SORT GROUP BY 10981 NESTED LOOPS                                             |
>|  939| TABLE ACCESS (BY INDEX ROWID) OF 'ORDER'                                    |
>|  939|INDEX (RANGE SCAN) OF 'ORDER_IDX2' (NON-UNIQUE)                              |
>|10981|REMOTE                                                                       |
>

인덱스를 이용해 939건의 order 데이터를 읽어 원격으로 보냈고, 거기서 처리가 완료된 5건만 전송받은 것을 확인할 수 있다. 분산 쿼리의 성능을 높이는 핵심 원리는, 네트워크를 통한 데이터 전송량을 줄이는 데에 있다.

## 7. 사용자 정의 함수/프로시저의 특징과 성능

일반 프로그래밍 언어에서는 반복적으로 사용되는 소스 코드를 가급적 함수로써 모듈화하는 것을 권장한다. 하지만 DBMS 내부에서 수행되는 사용자 정의 함수/프로시저(User Defined Function/Procedure)를 그런 용도로 사용한다면 성능 때문에 큰 낭패를 볼 수 있다. 이유를 잘 설명하진 못하더라도 경험 많은 개발자들은 이미 그런 사실을 잘 알고 있다. 아래에서 설명하는 사용자 정의 함수/프로시저의 특징을 잘 파악한다면 오히려 그것을 잘 활용해 성능을 높일 수 있는 방안이 무엇인지 스스로 터득할 수 있을 것이다.

### 가. 사용자 정의 함수/프로시저의 특징

사용자 정의 함수/프로시저는 내장함수처럼 Native 코드로 완전 컴파일된 형태가 아니어서 가상머신(Virtual Machine) 같은 별도의 실행엔진을 통해 실행된다. 실행될 때마다 컨텍스트 스위칭(Context Switching)이 일어나며, 이 때문에 내장함수(Built-In)를 호출할 때와 비교해 성능을 상당히 떨어뜨린다. 예를 들어, 문자 타입의 일자 데이터를 날짜 타입으로 변환해 주는 to_char 함수를 바로 호출할 때와 아래와 같은 사용자 정의 함수를 호출할 때를 비교하면, 보통 5~10배 가량 느려지는 것을 확인할 수 있다.

>```sql
>CREATE OR REPLACE FUNCTION DATE_TO_CHAR(P_DT DATE)
>RETURN VARCHAR2
>AS
>BEGIN
>    RETURN TO_CHAR(P_DT, 'yyyy/mm/dd hh24:mi:ss');
>END; 
>```

게다가, 메인 쿼리가 참조하는 사용자 정의 함수에 또 다른 쿼리문이 내장돼 있으면 수행 성능이 훨씬 나빠진다. 함수에 내장된 쿼리를 수행될 때마다 Execute Call, Fetch Call이 재귀적으로 일어나기 때문이다. 앞에서 잠시 언급한 Recursive Call이 반복적으로 일어나는 것이며, 다행히 Parse Call은 처음 수행할 때 한 번만 일어난다. 네트워크를 경유해 DBMS에 전달되는 User Call에 비해 Recursive Call의 성능 부하는 미미하다고 할 수 있지만, 가랑비에도 옷이 젖든 그 횟수가 무수히 반복되면 성능을 크게 떨어뜨릴 수 있다.

### 나. 사용자 정의 함수/프로시저에 의한 성능 저하 해소 방안

주문 테이블에서 주문일자가 잘못된 데이터를 찾아 정제하려고 아래와 같은 사용자 정의 함수를 정의했다고 가정하자. 주문을 받지 않는 휴무일에 입력된 데이터도 정제 대상이므로 해당 일자가 휴무일 테이블에서 찾아지는지도 검사하도록 구현하였다.

>```sql
>create or replace function 일자검사(p_date varchar2) return varchar2 as l_date varchar2(8);
> begin l_date := to_char(to_date(p_date, 'yyyymmdd'), 'yyyymmdd');
> -- 일자오류 시, Exception 발생 if l_date > to_char(trunc(sysdate), 'yyyymmdd') then return 'xxxxxxxx';
> -- 미래일자로 입력된 주문 데이터 end if;
> for i in (SELECT 휴무일자
>  FROM 휴무일
> WHERE 휴무일자= l_date
>) loop return 'xxxxxxxx';
> -- 휴무일에입력된 주문 데이터 end loop;
> return l_date;
> -- 정상적인주문 데이터 exception when others then return '00000000';
> -- 오류데이터 end;
>```

이 함수를 이용해 1,000만 개 주문 레코드를 아래와 같이 검사하면 1,000만 번의 컨텍스트 스위칭이 발생함은 물론 Execute Call과 Fetch Call이 각각 1,000만 번씩 발생한다. 이렇게 많은 일을 수행하도록 개발하고서 좋은 성능을 기대할 수 있겠는가.

>```sql
>SELECT *
>  FROM 주문
> WHERE 일자검사(주문일자) IN ( '00000000', 'xxxxxxxx' ); 
>```

요컨대, 대용량 조회 쿼리에서 함수를 남용하면 읽는 레코드 수만큼 함수 호출과 Recursive Call이 반복돼 성능이 극도로 나빠진다. 따라서 사용자 정의 함수는 소량의 데이터를 조회할 때, 또는 부분범위처리가 가능한 상황에서 제한적으로 사용해야 한다. 성능을 위해서라면 가급적 함수를 풀어 조인 또는 스칼라 서브쿼리 형태로 변환하려고 노력해야 한다.<br>

사용자 정의 함수를 사용하지 않고 위 프로그램을 One SQL로 구현하려면 어떻게 해야 할까? 이 회사가 창립 50주년을 맞는 회사라고 간주하고 아래와 같이 50년치 일자 테이블을 만들어 보자. 일자 테이블이 이미 만들어져 있다면 그것을 이용해도 된다.

>```sql
>CREATE TABLE 일자 as
>SELECT trunc(sysdate - rownum + 1) d_date
>     , to_char(trunc(sysdate - rownum + 1), 'yyyymmdd') c_date
>  FROM big_table
> WHERE rownum <= (trunc(sysdate)-trunc(add_months(sysdate, - (12*50)), 'yy')+1);
>
>CREATE unique index 일자검사_idx on 일자검사(c_date);
>```

그리고 아래와 같이 not EXISTS와 EXISTS 구문을 이용해 일자와 휴무일 테이블을 필터링하면 된다. 실제 테스트해 보면, 위에서 함수를 사용했을 때와는 비교할 수 없이 빠르게 수행될 것이다.

>```sql
>SELECT * FROM 주문 O
> WHERE NOT EXISTS (SELECT 'x' FROM 일자
>                    WHERE C_DATE = O.주문일자)
>        OR EXISTS (SELECT 'x' FROM 휴무일
>                    WHERE 휴무일자 = O.주문일자) 
>```

함수의 구현내용이 아주 복잡하면 One SQL로 풀어내는 것이 불가능할 수도 있다. 그럴 때는 함수 호출을 최소화하도록 튜닝해야 한다. 본 가이드에선 지면관계상 생략하지만, 별도의 튜닝 전문서적을 통해 반드시 학습하기 바란다. 지금까지의 설명을 사용자 정의 함수/프로시저를 절대 사용하지 말라는 의미로 오해하지 말기 바란다. 잘 활용하면 오히려 성능을 크게 향상시킬 수도 있는데, 소량 호출하고 내부에서 다량의 SQL을 수행하는 형태가 그렇다. 만약 같은 로직을 외부 프로그램 언어로 구현한다면 다량의 SQL을 User Call로써 수행해야 하기 때문에 훨씬 느려진다.

<br><br><br>
> 출처 : 데이터온에어 – 한국데이터산업진흥원([https://dataonair.or.kr](https://dataonair.or.kr))