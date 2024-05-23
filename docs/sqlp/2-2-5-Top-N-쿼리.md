---
title: 2-2-5. Top N 쿼리
tags: 
---

# 과목2. SQL 기본과 활용
# 제2장 SQL 활용
# 제5절 Top N 쿼리

## 1. ROWNUM 슈도 칼럼
Oracle의 ROWNUM은 칼럼과 비슷한 성격의 Pseudo Column으로써 SQL 처리 결과 집합의 각 행에 대해 임시로 부여되는 일련번호이며, 테이블이나 집합에서 원하는 만큼의 행만 가져오고 싶을 때 WHERE 절에서 행의 개수를 제한하는 목적으로 사용한다.<br>

건의 행만 가져오고 싶을 때는 
- SELECT PLAYER_NAME FROM PLAYER WHERE ROWNUM = 1; 이나 
- SELECT PLAYER_NAME FROM PLAYER WHERE ROWNUM <= 1; 이나 
- SELECT PLAYER_NAME FROM PLAYER WHERE ROWNUM < 2; 처럼 사용할 수 있다.<br>

두 건 이상의 N 행을 가져오고 싶을 때는 ROWNUM = N; 처럼 사용할 수 없으며
- SELECT PLAYER_NAME FROM PLAYER WHERE ROWNUM <= N; 이나
- SELECT PLAYER_NAME FROM PLAYER WHERE ROWNUM < N+1; 처럼 출력되는 행의 한계를 지정할 수 있다.<br>

추가적인 ROWNUM의 용도로는 테이블 내의 고유한 키나 인덱스 값을 만들 수 있다. - UPDATE MY_TABLE SET COLUMN1 = ROWNUM;

## 2. TOP 절
SQL Server는 TOP 절을 사용하여 결과 집합으로 출력되는 행의 수를 제한할 수 있다. TOP 절의 표현식은 다음과 같다.

>TOP (Expression) [PERCENT] [WITH TIES]

- Expression : 반환할 행의 수를 지정하는 숫자이다. 
- PERCENT : 쿼리 결과 집합에서 처음 Expression%의 행만 반환됨을 나타낸다. 
- WITH TIES : ORDER BY 절이 지정된 경우에만 사용할 수 있으며, TOP N(PERCENT)의 마지막 행과 같은 값이 있는 경우 추가 행이 출력되도록 지정할 수 있다.<br>

한 건의 행만 가져오고 싶을 때는 
- SELECT TOP(1) PLAYER_NAME FROM PLAYER; 처럼 사용할 수 있다.<br>

두 건 이상의 N 행을 가져오고 싶을 때는 
- SELECT TOP(N) PLAYER_NAME FROM PLAYER; 처럼 출력되는 행의 개수를 지정할 수 있다.<br>

SQL 문장에서 ORDER BY 절이 사용되지 않으면 Oracle의 ROWNUM과 SQL Server의 TOP 절은 같은 기능을 하지만, ORDER BY 절이 같이 사용되면 기능의 차이가 발생한다. 이 부분은 1장 8절 ORDER BY 절에서 설명하도록 한다.

## 3. ROW LIMITING 절

Oracle은 12.1버전, SQL Server는 2012 버전부터 ROW LIMITING 절로 Top N 쿼리를 작성할 수 이?ㅆ다. ROW LIMITING 절은 ANSI 표준 SQL 문법이다.<br>
아래는 ROW LIMITING 절의 구문이다. ROW LIMITING 절은 ORDER BY 절 다음에 기술하며, ORDER BY 절과 함께 수행된다. ROW와 ROWS는 구분하지 않아도 된다.

>```sql
> [OFFSET offset {ROW | ROWS}]
> [FETCH {FIRST | NEXT} [{rowcount | percent PERCENT}] {ROW | ROWS} {ONLY | WITH TIES}]
>```

- OFFSET offset : 건너뛸 행의 개수를 지정한다.
- FETCH : 반환할 행의 개수나 백분율을 지정한다.
- ONLY : 지정된 행의 개수나 백분율만큼 행을 반환한다.
- WITH TIES : 마지막 행에 대한 동순위를 포함해서 반환한다.

[예제] 아래는 ROW LIMITING 절을 사용한 Top N 쿼리다.

###### [예제]

>```sql
>SELECT ENAME, SAL FROM EMP 
> ORDER BY SAL, EMPNO
> FETCH FIRST 5 ROWS ONLY;
>```

[예제] 아래와 같이 OFFSET만 기술하면 건너뛴 행 이후의 전체 행이 반환된다.

###### [예제]

>```sql
>SELECT ENAME, SAL FROM EMP 
> ORDER BY SAL, EMPNO
> OFFSET 5 ROWS
>```

<br><br><br>
> 출처 : 데이터온에어 – 한국데이터산업진흥원([https://dataonair.or.kr](https://dataonair.or.kr))