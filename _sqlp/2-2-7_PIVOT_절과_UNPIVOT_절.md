---
title: 제7절 PIVOT 절과 UNPIVOT 절
nav_order: 7
parent: 제2장 SQL 활용
grand_parent: 과목2 SQL 기본과 활용
toc: true
toc_label: "Contents"
---

## 1. 개요

PIVOT은 회전시킨다는 의미를 갖고 있다. PIVOT 절은 행을 열로 회전시키고, UNPIVOT 절은 열을 행으로 회전시킨다.

## 2. PIVOT 절

PIVOT 절은 행을 열로 전환한다. PIVOT 절의 구문은 아래와 같다.

>```sql
>PIVOT [XML]
>     ( aggregate_function (expr) [[AS] alias]
>    [, aggregate_function (expr) [[AS] alias]] ...
>	  FOR {column | (column [, column]...)} 
>	  IN ({ { { expr | (expr [, expr]...) } [[AS] alias] }...
>	   | subquery
>	   | ANY [, ANY]...
>	   })
>   )
>```

- aggregate_function은 집계할 열을 지정한다.
- FOR 절은 PIVOT할 열을 지정한다.
- IN 절은 PIVOT할 열 값을 지정한다.

다음은 PIVOT 절을 사용한 쿼리다. PIVOT 절은 집계함수와 FOR 절에 지정되지 않은 열을 기준으로 집계되기 때문에 인라인 뷰를 통해 사용할 열을 지정해야 한다.

###### [예제]
>```sql
>SELECT *
>  FROM ( SELECT JOB, DEPTNO, SAL FROM EMP )
> PIVOT (SUM (SAL) FOR DEPTNO IN (10, 20, 30))
> ORDER BY 1;
>```

###### [실행 결과]

>|JOB|10|20|30|
>|---|---|---|---|
>|ANALYST||6000||
>|CLERK|1300|1900|950|
>|MANAGER|2450|2975|2850|
>|PRESIDENT|5000|||
>|SALESMAN|||5600|
>
>###### 5 개의 행이 선택되었습니다.

다음 쿼리는 인라인 뷰에 yyyy 표현식을 추가한 것이다. 행 그룹에 yyyy 표현식이 추가된 것을 확인할 수 있다.

###### [예제]

>```sql
>SELECT *
>  FROM (SELECT TO_CHAR (HIREDATE, 'YYYY') AS YYYY, JOB, DEPTNO, SAL FROM EMP)
> PIVOT (SUM(SAL) FOR DEPTNO IN (10, 20, 30))
> ORDER BY 1, 2;
>```

###### [실행 결과]

>|YYYY|JOB|10|20|30|
>|---|---|---|---|---|
>|1980|CLERK||800||
>|1981|ANALYST||3000||
>|1981|CLERK|||950|
>|1981|MANAGER|2450|2975|2850|
>
>###### 9 개의 행이 선택되었습니다.

다음 쿼리는 집계함수와 IN 절에 별칭을 지정했다. 별칭을 지정하면 결과 집합의 열 명이 변경된다.

###### [예제]

```sql
SELECT *
  FROM (SELECT JOB, DEPTNO, SAL FROM EMP)
 PIVOT (SUM (SAL) AS SAL FOR DEPTNO IN (10 AS D10, 20 AS D20, 30 AS D30))
 ORDER BY 1;
```

[SQL]
2 971 71015

###### [실행 결과]

```sql
|JOB|D10_SAL|D20_SAL|D30_SAL|
|---|---|---|---|
|ANALYST||6000||
|CLERK|1300|1900|950|
|MANAGER|2450|2975|2850|
|PRESIDENT|5000|||
|SALESMAN|||5600|
```

###### 5 개의 행이 선택되었습니다.

집계함수와 IN 절에 지정한 별칭에 따라 아래와 같은 규칙으로 열 명이 부여된다. 집계함수와 IN 절 모두 별칭을지정하는 편이 바람직하다.

10 10 AS d10 10 D10 SUM (sal) SUM (sal) AS sal 10_SAL D10_SAL

SELECT 절에 부여된 열 명을 지정하면 필요한 열만 조회할 수 있다.

###### [예제]

SELECT JOB, D20 SAL FROM (SELECT JOB, DEPTNO, SAL FROM EMP) PIVOT (SUM (SAL) AS SAL FOR DEPTNO IN (10 AS D10, 20 AS D20, 30 AS D30)) WHERE D20_SAL > 2500 ORDER BY 1;

###### [실행 결과]JOBD20_SAL6000ANALYSTMANAGER2975

###### 2 개의 행이 선택되었습니다.


PIVOT 절은 다수의 집계함수를 지원한다. 다음 쿼리는 SUM 함수와 COUNT 함수를 함께 사용했다.

###### [예제]

* SELECT FROM (SELECT JOB, DEPTNO, SAL FROM EMP) PIVOT (SUM (SAL) AS SAL, COUNT (*) AS CNT FOR DEPTNO IN (10 AS D10, 20 AS D20)) ORDER BY 1;

###### [실행 결과]
JOBD10_SALD10_CNTD20 SALD20_CNT060002119002ANALYSTCLERKMANAGERPRESIDENTSALESMAN 13002450129751500010005 개의 행이 선택되었습니다.

FOR 절에도 다수의 열을 기술할 수 있다. 다음과 같이 IN 절에 다중 열을 사용해야 한다.

###### [예제]

* SELECT FROM (SELECT TO_CHAR (HIREDATE, 'YYYY') AS YYYY, JOB, DEPTNO, SAL FROM EMP) PIVOT (SUM (SAL) AS SAL, COUNT (*) AS CNT FOR (DEPTNO, JOB) IN ((10, 'ANALYST) AS D10A, (10, 'CLERK') AS D100 (20, 'ANALYST') AS D20A, (20, 'CLERK') AS D20C))

ORDER BY 1;

###### [실행 결과]
D10A_SALD10A_CNTD10C_SALD10C_CNTYYYYD20A_SALD20A_CNTD20C_SALD20C_CNT00800103000101980198119821987oooo1300100030001110014 개의 행이 선택되었습니다.

PIVOT 절을 사용할 수 없는 경우 집계함수와 CASE 표현식으로 PIVOT을 수행할 수 있다.

###### [예제]

SELECT JOB

SUM (CASE DEPTNO WHEN 10 THEN SAL END) AS D10_SAL SUM (CASE DEPTNO WHEN 20 THEN SAL END) AS D20_SAL

SUM (CASE DEPTNO WHEN 30 THEN SAL END) AS D30_SAL FROM EMP GROUP BY JOB ORDER BY JOB;

###### [실행 결과] JOB D10_SAL D20_SAL D30_SAL 6000 ANALYST CLERK MANAGER PRESIDENT SALESMAN 1300 2450 5000 1900 2975 950 2850 5600

###### 5 개의 행이 선택되었습니다.

2. UNPIVOT 절

UNPIVOT 절은 PIVOT 절과 반대로 동작한다. 열이 행으로 전환된다. UNPIVOT 절의 구문은 아래와 같다.

UNPIVOT [{ INCLUDE { EXCLUDE} NULLS] ( {column | (column C, Col]…)} FOR {column | (column [, col]…)} IN ({column | (column C, col)...)} [AS (literal | (literal [, literal] ---)}] [, {column { (column C, col]…)} [AS {literal { (literal [, literal])}]] …

UNPIVOT column 절은 UNPIVOT된 값이 들어갈 열을 지정한다.FOR 절은 UNPIVOT된 값을 설명할 값이 들어갈 열을 지정한다.IN 절은 UNPIVOT할 열과 설명할 값의 리터럴 값을 지정한다.-

예제를 위해 다음과 같이 테이블을 생성하자.

DROP TABLE T1 PURGE; CREATE TABLE T1 AS SELECT JOB, D1O_SAL, D20_SAL, D10_CNT, D20_CNT FROM (SELECT JOB, DEPTNO, SAL FROM EMP WHERE JOB IN ('ANALYST', 'CLERK')) PIVOT (SUM (SAL) AS SAL, COUNT(*) AS CNT FOR DEPTNO IN (10 AS D10, 20 AS D20));

아래는 tl 테이블을 조회한 결과다.

###### [예제]

SELECT * FROM T1 ORDER BY JOB;

###### [실행 결과]JOBD10_SALD20_SALD10_CNTD20_CNTANALYST600002CLERK13001900122 개의 행이 선택되었습니다.

아래는 UNPIVOT 절을 사용한 쿼리다. d10_sal, d20_sal 열이 행으로 전환된다.

[###### [예제]

SELECT JOB, DEPTNO, SAL FROM T1 UNPIVOT (SAL FOR DEPTNO IN (D10_SAL, D2O_SAL)) ORDER BY 1, 2;

###### [실행 결과] JOB DEPTNO SAL ANALYST CLERK CLERK D20_SAL D10_SAL D2O_SAL 6000 1300 1900

###### 3 개의 행이 선택되었습니다.

IN 절에 별칭을 지정하면 FOR 절에 지정한 열의 값을 변경할 수 있다. 다음 쿼리는 10, 20으로 값을 변경했다.

###### [예제]

SELECT JOB, DEPTNO, SAL

FROM T1

UNPIVOT (SAL FOR DEPTNO IN (D10_SAL AS 10, D20_SAL AS 20)) ORDER BY 1, 2;

###### [실행 결과]JOBDEPTNOSAL20ANALYSTCLERKCLERK60001300102019003 개의 행이 선택되었습니다.

다음과 같이 INCLUDE NULLS 키워드를 기술하면 UNPIVOT된 열의 값이 널인 행도 결과에 포함된다.

###### [예제]

SELECT JOB, DEPTNO, SAL

FROM T1

UNPIVOT INCLUDE NULLS (SAL FOR DEPTNO IN (D10_SAL AS 10, D20_SAL AS 20)) ORDER BY 1, 2;

###### [실행 결과]JOBDEPTNOSAL10206000ANALYSTANALYSTCLERKCLERK1300102019004 개의 행이 선택되었습니다.

FOR 절에 다수의 열, IN 절에 다수의 별칭을 지정할 수도 있다.

###### [예제]

* SELECT

FROM T1 UNPIVOT SAL, CNT)

FOR DEPTNO IN ((D1O_SAL, D1O_CNT) AS 10, (D2O_SAL, D20_CNT) AS 20)) ORDER BY 1, 2;

###### [실행 결과] JOB DEPTNO SAL CNT 10 ㎞ | 0212 20 ANALYST ANALYST CLERK CLERK 600 10 1300 1900 20

###### 4 개의 행이 선택되었습니다.

FOR 절에 다수의 열, IN 절에 다수의 별칭을 지정할 수도 있다.

###### [예제] * SELECT FROM T1 UNPIVOT ((SAL, CNT)

FOR (DEPTNO, DNAME) IN ((D10_SAL, D10_CNT) AS (10, ACCOUNTING') (D20_SAL, D20_CNT) AS (20, 'RESEARCH' ))) ORDER BY 1, 2; ###### [실행 결과] JOB DEPTNO DNAME SAL CNT 10 0 20 ANALYST ANALYST CLERK CLERK 2 ACCOUNTING RESEARCH ACCOUNTING RESEARCH 10 6000 1300 1900 1 20 2 4 개의 행이 선택되었습니다.

UNPIVOT 절을 사용할 수 없는 경우 카티션 곱을 사용해 UNPIVOT을 수행할 수 있다. UNPIVOT할 열의 개수만큼 행을 복제하고, CASE 표현식으로 UNPIVOT할 열을 선택하는 방식이다.

[01|2||]

SELECT A. JOB CASE B.LV WHEN 1 THEN 10 WHEN 2 THEN 20 END AS DEPTNO CASE B.LV WHEN 1 THEN A.D10_SAL WHEN 2 THEN A.D20_SAL END AS SAL CASE B.LV WHEN 1 THEN A.D10_CNT WHEN 2 THEN A.D20_CNT END AS CNT FROM T1 A (SELECT LEVEL AS LV FROM DUAL CONNECT BY LEVEL (= 2) B ORDER BY 1, 2;

###### [실행 결과]

JOB

DEPTNO

SAL

ANALYST ANALYST CLERK CLERK

CNT

10

20

6000

1300

###### 4 개의 행이 선택되었습니다.

1900

2.

0

10

1

20

2



다음 실행 결과에서 강조한 부분이 CASE 표현식으로 선택한 값이다.

###### [예제]

SELECT A.JOB, B.LV, A.D10_SAL, A.D20_SAL, A.D10_CNT, A.D20_CNT

FROM T1 A

(SELECT LEVEL AS LV FROM DUAL CONNECT BY LEVEL = 2) B ORDER BY A. JOB, B.LV;

###### [실행 결과]JOBLVD10_SALD20_SALD10_CNTD20_CNT102.0 ANALYSTANALYSTCLERKCLERK 6000600019001900NNNN22221130013001 - 1 124 개의 행이 선택되었습니다.

