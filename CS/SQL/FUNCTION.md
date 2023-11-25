## 06장. 데이터 처리와 가공을 위한 오라클 함수

```markdown
오라클 함수 
- 내장 함수
    - 단일행 함수: 데이터가 한 행씩 입력되고 입력된 한 행당 결과가 하나씩 나오는 함수
    - 다중행 함수: 여러 행이 입력되어 하나의 행으로 결과가 반환되는 함수
- 사용자 정의 함수
```

<br/>

### 문자

- 대소문자
    - UPPER(문자열) : 대문자로 변환
    - LOWER(문자열) : 소문자로 변환
    - INITCAP(문자열) : 첫 문자 대문자, 나머지는 소문자로 변환
    - `SELECT NAME, UPPER(NAME), INITCAP(NAME) FROM …;`
    - `SELECT * FROM EMP WHERE UPPER(NAME) LIKE UPPER(’%JUNG%’)` : 이름에 JUNG이 들어간 모든 행 출력 (대소문자 상관 ㄴ)
- 길이
    - LENGTH(문자열)
    - LENGTHB(문자열) : 바이트 단위 길이 출력
- SUBSTR
    - SUBSTR(칼럼명, 시작 위치, 추출 길이)
    - SUBSTR(칼럼명, 시작 위치) : 끝까지 출력
    - 시작 위치 값으로는 음수도 가능함. 음수로 하게 되면 문자열의 맨 뒤가 -1이 되고 문자열의 첫 번째가 -n이 됨.
    - 맨 첫 번째 글자의 인덱스는 1임.
- INSTR
    - INSTR(대상 문자열 데이터, 위치 찾으려는 부분 문자)
    - INSTR(대상 문자열 데이터, 위치 찾으려는 부분 문자, 위치 찾기 시작할 대상 문자열 데이터 위치)
    - INSTR(대상 문자열 데이터, 위치 찾으려는 부분 문자, 위치 찾기 시작할 대상 문자열 데이터 위치, 시작 위치에서 찾으려는 문자가 몇 번째인지 지정)
    - [ ]  칼럼 지정 안 됨?
- REPLACE
    - 특정 문자열 데이터에 포함된 문자를 다른 문자로 대체
    - REPLACE(문자열 데이터 or 칼럼 네임, 찾는 문자) : 찾는 문자가 모두 제거됨.
    - REPLACE(문자열 데이터 or 칼럼 네임, 찾는 문자, 대체할 문자)
- LPAD, RPAD
    - LPAD(문자열 데이터 or 칼럼 네임, 데이터의 자릿수) : 공백 문자로 왼쪽 채움
    - LPAD(문자열 데이터 or 칼럼 네임, 데이터의 자릿수, [채울 문자])
    - RPAD(문자열 데이터 or 칼럼 네임, 데이터의 자릿수) : 공백 문자로 오른쪽 채움
    - RPAD(문자열 데이터 or 칼럼 네임, 데이터의 자릿수, [채울 문자])
- CONCAT
    - 두 개의 문자열 합치기
    - `SELECT CONCAT(CONCAT(NAME, ‘:’), ‘AGE’) FROM ..;`
    - 이렇게 실행하면 NAME_값 : AGE 형태가 나오게 됨.
    - `SELECT NAME || ‘:’ || ‘AGE’`도 위의 예시와 같은 결과가 나오게 됨.
- TRIM
    - TRIM([삭제 옵션] [삭제 문자] FROM 원본 문자열 데이터) : 콤마 없이 열거함.
    - LTRIM, RTRIM
    - 걍 리플레이스 쓰자…

<br/>

### 숫자

- ROUND
    - 특정 위치에서 반올림.
    - `SELECT ROUND(칼럼명, 위치) FROM …;`
    - `SELECT ROUND(123.456, 2) FROM …;`의 결과는 123.46이 됨.
- TRUNC
    - 특정 위치에서 숫자를 버림 처리함.
    - `SELECT TRUNC(123.456, 2) FROM …;`의 결과는 123.45가 됨.
- CEIL, FLOOR
    - 반올림, 반내림한 정수 반환
    - `SELECT CEIL(123.456), FLOOR(123.456) FROM …;`의 결과는 124, 123이 됨.
- MOD
    - 나머지 값 구함.
    - `SELECT MOD(15, 6) FROM …;`의 결과는 3이 됨. 15 % 6임.

<br/>

### 날짜

> 날짜 - 날짜 : 일수 차이 계산
날짜 + 숫자 : 날짜 + 숫자 일수만큼의 이후 날짜 계산
날짜 - 숫자 : 날짜 - 숫자 일수만큼의 이전 날짜 계산
>
- SYSDATE
    - 오라클 데이터베이스가 놓인 OS의 현재 날짜와 시간을 보여줌.
    - `SELECT SYSDATE FROM …;`
- ADD_MONTHS
    - `SELECT ADD_MONTHS(SYSDATE, -1) FROM …;` 한 달 전 날짜와 시간 출력
- MONTHS_BETWEEN
    - `SELECT MONTHS_BETWEEN(SYSDATE, HIREDATE) FROM …;` 두 날짜간의 개월 수 출력
- NEXT_DAY
    - NEXT_DAY(날짜 데이터, 요일 문자)
    - 요일 문자 : Mon, Tue, Wed, Thu, Fri, Sat, Sun
- LAST_DAY
    - 날짜가 속한 달의 마지막 날짜 출력
    - `LAST_DAY(SYSDATE)` 요런 식으로 사용

<br/>

### Type Casting

- DATE 변환
    - `SELECT TO_CHAR(SYSDATE, ‘YY-MM/DD/HH24:MI^SS AM’) FROM …;`
    - CC(세기), YYYY, YY(연도 2글자), MM(월 숫자), MON(월 축약어 문자), MONTH(월 전체 문자), DD(일 숫자), DY(요일 약자), DAY(요일 문자 전체), W
    - HH24(시간을 24시간으로 표현), HH(12시간으로 시간 표현), MI, SS, AM, PM
- NUM → STRING
    - 9 : 숫자 하나 의미하며 빈자리 채우지 않음
    - 0 : 빈자리일 경우 0으로 채움을 의미
    - $ : 달러 붙임
    - L : 로컬 화폐 단위 붙임
    - . : 소숫점
    - , : 천 단위 구분 기호로 콤마 사용
    - `SELECT TO_CHAR(SAL, ‘$000,999,999.00) FROM …;`
- STRING → NUM
    - TO_NUMBER(문자열 _데이터, 인식될_숫자_형태)
    - `TO_NUMBER(‘1,300’, ‘999,999’)`
- STRING → DATE
    - TO_DATE(문자열_데이터, 인식될_날짜_형태)
    - `TO_DATE(‘201809-14’, ‘YYYYMM-DD’)`

<br/>

### NULL 처리 함수

- NVL
    - NVL(NULL 체크할 칼럼, NULL일 경우 반환할 데이터)
    - `SELECT EMPNO+NVL(SAL, 30) FROM …;`
    - NVL(NULL 체크 칼럼, NULL이 아닐 경우의 데이터나 계산식, NULL일 경우의 데이터나 계산식)

<br/>

### 상황에 따른 값

- CASE

```kotlin
SELECT ..., 
CASE 칼럼명 WHEN 값1 THEN 결과값1
					WHEN 값2 THEN 결과값2
					ELSE 디폴트 결과값
END AS 칼럼_ALIAS 
FROM ...;
```

```kotlin
SELECT ..., 
CASE WHEN 칼럼명 = 값1 THEN 결과값1
		 WHEN 칼럼명 = 값2 THEN 결과값2
		 ELSE 디폴트 결과값
END AS 칼럼_ALIAS 
FROM ...;
```