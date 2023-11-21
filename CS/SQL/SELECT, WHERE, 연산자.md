## 04장. SELECT문의 기본 형식

- SELECT
- FROM
- DISTINCT : 중복 제거
- ALL : 안 써도 디폴트임
- AS : select문에서 별칭 줄 때 사용.
- ORDER BY
    - ASC
    - DESC
    - 여러개도 가능

<br/>

## 05장. 더 정확하고 다양하게 결과를 출력하는 WHERE절과 연산자

- WHERE
- AND
- OR
- +, -, /, *
    - SELECT * FROM EMP WHERE SAL * 12 = 36000;
- \>, >=, <, <=
    - 문자열끼리도 대소비교 가능
- =
    - 같음
- !=, <>, ^=
    - 다름
    - 문자열 비교는 불가능함. 문자열은 NOT IN 사용해서 비교해라. 아니면 다음과 같이 사용하거나…`SELECT * FROM Customers WHERE NOT Country = "Germany";`
- NOT
    - 논리 부정 연산자
    - `SELECT * FROM EMP WHERE NOT SAL = 3000;` `SELECT * FROM EMP WHERE NOT SAL ≠ 3000;` 두 sql문은 같은 의미.
    - 보통 IN, BETWEEN, IS NULL과 함께 씀.
- IN
    - `SELECT .. FROM .. WHERE 칼럼_이름 IN (데이터1, 데이터2, …);`
    - 특정 열에 포함된 데이터를 여러 개 조회할 때 활용.
    - NOT IN으로 사용하면 열거한 값들 중 하나라도 해당되지 않을 경우를 원할 때 활용.
- BETWEEN A AND B
    - A는 최솟값, B는 최댓값으로 각 구간 포함.
    - `SELECT … FROM … WHERE SAL BETWEEN 300 AND 500;`
    - NOT을 사용하면 해당 범위가 아닌 것만 출력함. (위 예에서는 300 미만, 500 초과)
- LIKE
    - 일부 문자열이 포함된 데이터 조회 시 사용.
    - `SLECET … FROM … WHRER NAME LIKE ‘KIM%’`
    - 와일드 카드 문자: `_`(어떤 값이든 상관 없이 한 개의 문자 데이터를 의미), `%`(길이와 상관 없이 모든 문자 데이터를 의미)
    - 와일드 카드를 포함한 데이터 조회하고 싶은 경우 - e.g. A_A : `SELECT … FROM … WHRER NAME LIKE ‘A₩_A%’ ESCAPE ‘₩’;`  escape 절을 사용하면 됨.
- IS NULL
    - `SELECT … FROM … WHRER SAL IS NULL;`
    - `SELECT … FROM … WHRER SAL IS NOT NULL;`
- 집합 연산자
    - 집합 연산자로 두 개의 SELECT문의 결과값을 연결할 때 각 SELECT문이 출력하려는 열 개수와 각 열의 자료형이 순서별로 일치해야 함.
    - UNION : 연결된 SELECT문의 결과 값을 합집합으로 묶어줌. 당연히 중복 제거됨.
    - UNION ALL : 연결된 SELECT문의 결과 값을 합집합으로 묶어줌. 중복된 결과 값도 제거 없이 모두 출력됨.
    - MINUS : 먼저 작성한 SELECT문에서 이후 SELECT문의 결과 값을 차집합 처리함.
    - INTERSECT : 교집합.
    - `SELECT … FROM A WHRER … UNION SELECT … FROM B WHERE …`

<br/>

![image](https://github.com/eunjjungg/TIL/assets/100047095/0addb081-25b0-415a-b864-6f55289f2e59)

다음과 같이 연산자 우선순위가 존재하지만, 먼저 수행해야 하는 연산식을 ()로 묶어주면 기본 우선순위와 별개로 먼저 실행되므로 활용하면 됨.