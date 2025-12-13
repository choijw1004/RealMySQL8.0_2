<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(2)

--- 
</div>


# SELECT
일반적인 온라인 트랜잭션 처리 환경의 데이터베이스에서는 INSERT나 UPDATE 같은 작업은 레코드 단위로 발생하므로 성능상 문제가 되는 경우는 별로 없다. 하지만 SELECT는 여러 개의 테이블로부터 데이터를 조합해서 빠르게 가져와야 하기 떄문에 여러 개의 테이블을 어떻게 읽을 것인지 많은 주의를 기울여야한다. 

## SELECT 절의 처리 순서
먼저 SELECT절이라고 하면 SQL 전체를 의미한다. 

일반적으로 SELECT 절의 처리 순서는 다음과 같다. 

> FROM -> JOIN -> WHERE -> GROUP BY -> DISTINCT -> HAVING 조건 적용 -> SELECT -> ORDER BY -> LIMIT

다만, 실행 순서를 벗어나는 쿼리가 필요하다면 서브쿼리로 작성된 인라인 뷰를 사용하면 된다.

### 인라인 뷰(Inline View)란?
하나의 질의문 내에서만 생성되어 사용되어지고 질의문 수행 종료 후에는 사라지는 뷰를 뜻한다.  
일반적으로 FROM 절에서 서브쿼리를 하나의 테이블로 사용하는 형태를 말한다.

하지만 인라인 뷰가 사용되면 임시 테이블이 사용되기 때문에 주의해야 한다.

## WHERE 조건과 GROUP BY 절, ORDER BY 절의 인덱스 사용

### 인덱스를 사용하기 위한 기본 규칙

#### 1. 인덱스 컬럼 변형 없이 사용하기

WHERE 절이나 ORDER BY, GROUP BY가 인덱스를 사용하려면 값에 대한 변형 없이 인덱스 컬럼을 그대로 사용해야 한다.

```sql
SELECT * FROM salaries WHERE salary * 10 > 150000;
```

위와 같은 쿼리는 salary 인덱스가 있더라도 값을 변형해서 비교하므로 인덱스를 이용할 수 없다.

#### 2. 비교 대상의 데이터 타입 일치

WHERE 절에 사용되는 비교 조건에서 연산자 양쪽의 비교 대상 값은 데이터 타입이 일치해야 한다.

```sql
SELECT * FROM tb_test WHERE age = 2;
```

위 쿼리의 age 컬럼은 문자열 타입으로 선언되었지만 비교 연산에는 숫자 타입으로 비교된다.  
따라서 `ref`나 `range`가 아닌 `index`(인덱스 풀 스캔)이 사용된다.

이 경우 MySQL 옵티마이저가 내부적으로 문자열 타입을 숫자 타입으로 변환한 후 비교 작업을 처리하는데, 타입이 변환된 후 비교 작업을 처리해야 하므로 인덱스 레인지 스캔이 불가능한 것이다.


### WHERE 절의 인덱스 사용

WHERE 절의 인덱스 사용 방법은 작업 범위 결정 조건과 체크 조건 두 가지 방식으로 구분한다.

작업 범위 결정 조건은 동등 비교 조건이나 IN으로 구성된 조건에 사용된 컬럼들이 인덱스의 구성과 좌측부터 얼마나 일치하는가에 따라 달라진다.


> WHERE 조건절의 순서는 실제 인덱스의 사용 여부와 무관하다. 그 이유는 옵티마이저는 인덱스를 사용할 수 있는 조건을 뽑아서 최적화를 수행할 수 있기 때문이다.

WHERE 조건절에서 COL_1과 COL_2는 동등 비교 조건이고 COL_3은 범위 비교 조건이다.  
따라서 그 뒤 컬럼인 COL_4는 인덱스의 범위 결정 조건으로 사용되지 못하고 체크 조건으로 사용된다.

다중 컬럼 인덱스에서는 N-1번째 컬럼이 N번째 컬럼에 의존해 다시 정렬된다.  
즉, COL_3 컬럼까지는 비교 작업의 범위를 줄이는데(작업 범위 결정 조건) 도움을 주지만 COL_4 컬럼은 COL_3에 의존적이므로 범위를 좁히지 못하고 단순히 비교 용도(필터링 조건)로만 사용되어 체크 조건으로 사용되는 것이다.

> 위 예시들은 모두 AND 조건으로 연결되는 경우를 가정한 것이며, OR 조건으로 묶이면 더욱 복잡해진다. 또한 OR로 연결되면 읽어서 비교해야 할 레코드가 더 늘어나기 때문에 주의해야 한다.


### GROUP BY 절의 인덱스 사용

GROUP BY 절의 각 컬럼은 비교 연산자를 가지지 않으므로 작업 범위 결정 조건이나 체크 조건을 구분해서 생각할 필요 없이 GROUP BY 절에 명시된 컬럼의 순서가 인덱스 구성 컬럼 순서와 동일하면 인덱스를 이용할 수 있다.

#### GROUP BY 인덱스 사용 조건
1. GROUP BY 절에 명시된 컬럼이 인덱스 컬럼의 순서와 같아야 한다.
2. 순서상 인덱스의 앞쪽에 있는 컬럼이 GROUP BY 절에 명시되지 않으면 인덱스를 사용할 수 없다.
3. GROUP BY 절에 명시된 컬럼이 하나라도 인덱스에 없으면 인덱스를 사용하지 못한다.

#### WHERE 절과 GROUP BY 절의 혼용

WHERE 절과 GROUP BY 절이 혼용된 쿼리가 인덱스를 사용할 수 있는지를 판별하기 위해서는 WHERE 절에서 동등 비교 조건으로 사용된 컬럼을 GROUP BY 절로 옮겨보자.

```sql
# 원본 쿼리
... WHERE COL_1 = '상수' ... GROUP BY COL_2, COL_3

# WHERE 조건절의 COL_1 컬럼을 GROUP BY 절의 앞쪽으로 포함시켜 본 쿼리
... WHERE COL_1 = '상수' ... GROUP BY COL_1, COL_2, COL_3
```

위와 같이 변경해도 똑같은 결과가 조회된다면 인덱스를 사용할 수 있는 쿼리로 판단하면 된다.

### ORDER BY 절의 인덱스 사용

ORDER BY와 GROUP BY는 처리 방법이 거의 비슷하다. 따라서 인덱스 사용 조건도 거의 흡사하다.

한 가지 다른 점이 있다면, 정렬되는 컬럼의 오름차순 및 내림차순 옵션이 인덱스와 같거나 정반대인 경우에만 사용할 수 있다는 것이다.

```sql
# 인덱스: (COL_1 ASC, COL_2 ASC, COL_3 ASC)

# 인덱스 사용이 가능한 경우
ORDER BY COL_1, COL_2, COL_3

# 역순 가능
ORDER BY COL_1 DESC, COL_2 DESC, COL_3 DESC 

# 인덱스 사용이 불가능한 경우
ORDER BY COL_1 ASC, COL_2 DESC, COL_3 ASC   
```


### WHERE 조건과 ORDER BY(또는 GROUP BY) 절의 인덱스 사용

우리가 사용하는 쿼리는 일반적으로 WHERE 절과 GROUP BY 절, ORDER BY 절 등을 포함한 복잡한 형태의 쿼리로 구성된다.

하나의 쿼리에는 하나의 인덱스만 사용 가능하므로(index_merge 제외) 여러 개의 절이 같이 사용된 쿼리 문장은 다음 3가지 중 한 가지 방법으로만 인덱스를 이용한다.

| 방식 | 설명 | 특징 |
|------|------|------|
| WHERE + ORDER BY 동시 사용 | WHERE 절의 비교 조건과 ORDER BY 절의 정렬 대상이 모두 하나의 인덱스에 연속해서 포함 | 가장 빠른 성능. 이 방식으로 튜닝 권장 |
| WHERE 절만 인덱스 사용 | ORDER BY 절은 별도 정렬 처리 과정(Using Filesort)으로 처리 | WHERE 조건에 일치하는 레코드 건수가 많지 않을 때 효율적 |
| ORDER BY 절만 인덱스 사용 | ORDER BY 절의 순서대로 인덱스를 읽으며 WHERE 절의 조건에 일치하는지 비교 | 대량의 레코드를 조회해서 정렬해야 할 때 유용 |

이 방식도 마찬가지로 WHERE 절에 동등 비교로 비교된 컬럼과 ORDER BY 절에 명시된 컬럼이 순서대로 빠짐없이 인덱스에 왼쪽부터 일치해야 한다.

#### 범위 조건 비교 예제

```sql
# 인덱스 사용이 가능한 경우
SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_1, COL_2, COL_3;
```
COL_1에 만족하는 값은 여러 개일 수 있지만 ORDER BY 절에서 인덱스가 순서대로 모두 명시되었기 때문에 인덱스를 사용할 수 있다.

```sql
# 인덱스 사용이 불가능한 경우
SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_2, COL_3;
```
COL_1이 동등 비교 조건이었다면 인덱스를 사용할 수 있었겠지만, 범위 조건이므로 ORDER BY 절에서 COL_1이 명시되지 않은 이 쿼리는 인덱스를 이용하지 못한다.


### GROUP BY 절과 ORDER BY 절의 인덱스 사용

두 절이 모두 하나의 인덱스를 사용해서 처리되려면 GROUP BY 절과 ORDER BY 절에 명시된 컬럼의 순서와 내용이 모두 같아야 한다.

따라서 둘 중 하나라도 인덱스를 사용할 수 없는 경우 두 절 모두에서 인덱스를 사용할 수 없다.

```sql
# 인덱스 사용이 불가능한 경우
... GROUP BY col_1, col_2 ORDER BY col_2
... GROUP BY col_1, col_2 ORDER BY col_1, col_3
```

## WHERE 절의 비교 조건 사용 시 주의사항

### NULL 비교

MySQL에서는 NULL 값이 포함된 레코드도 인덱스로 관리한다.

```sql
SELECT * FROM titles WHERE to_date IS NULL;
```
컬럼의 값이 NULL인지 확인할 때는 ISNULL() 함수를 사용해도 되지만, 가급적 IS NULL 연산자를 사용하는 것을 권장한다.

```sql
# 인덱스 사용이 불가능한 경우
SELECT * FROM titles WHERE ISNULL(to_date) = 1;
SELECT * FROM titles WHERE ISNULL(to_date) = true;

# 인덱스 사용이 가능한 경우
SELECT * FROM titles WHERE to_date IS NULL;
```


### 문자열이나 숫자 비교

문자열 컬럼이나 숫자 컬럼을 비교할 때는 반드시 그 타입에 맞는 상수값을 사용해야 한다.  
컬럼의 타입에 맞게 상수 리터럴을 비교 조건에 사용하는 것은 인덱스 사용 여부에도 영향을 미치므로 아주 중요하다.


### 날짜 비교

MySQL에는 다양한 날짜/시간 타입이 있어 비교 조건이 상당히 헷갈릴 수 있다.

| 타입 | 저장 내용 |
|------|----------|
| DATE | 날짜만 |
| DATETIME | 날짜 + 시간 |
| TIMESTAMP | 날짜 + 시간 (타임존 적용) |
| TIME | 시간만 |

#### DATE 또는 DATETIME과 문자열 비교

DATE 또는 DATETIME 타입의 값과 문자열을 비교할 때는 문자열 값을 자동으로 DATETIME 타입의 값으로 변환해서 비교를 수행한다.

```sql
# 인덱스 사용이 불가능한 경우(컬럼 변형)
SELECT COUNT(*) FROM employees
WHERE DATE_FORMAT(hire_date, '%Y-%m-%d') > '2011-07-23';

# 인덱스 사용이 불가능한 경우(컬럼 변형)
SELECT COUNT(*) FROM employees
WHERE DATE_ADD(hire_date, INTERVAL 1 YEAR) > '2011-07-23';

# 인덱스 사용이 가능한 경우 (상수 변경)
SELECT COUNT(*) FROM employees
WHERE hire_date > DATE_SUB('2011-07-23', INTERVAL 1 YEAR);
```

#### DATE와 DATETIME의 비교

DATETIME 값을 DATE 타입으로 만들지 않고 비교하면 MySQL 서버가 DATE 타입의 값을 DATETIME으로 변환해서 비교를 수행한다.

즉, DATE 타입인 `"2022-11-19"` 값을 DATETIME 타입인 `"2022-11-19 00:00:00"`으로 변환해서 비교를 수행한다.

DATE와 DATETIME을 비교할 때는 성능보다는 쿼리 결과에 주의해서 사용해야 한다.

#### DATETIME과 TIMESTAMP의 비교

DATE 혹은 DATETIME 타입 값과 TIMESTAMP의 값을 별도 타입 변환 없이 비교하면 원하는 결괏값을 얻을 수 없다.

| 함수 | 변환 방향 |
|------|----------|
| `FROM_UNIXTIME()` | TIMESTAMP → DATETIME |
| `UNIX_TIMESTAMP()` | DATETIME → TIMESTAMP |


### Short-Circuit Evaluation

Short-circuit Evaluation이란 여러 개의 표현식이 AND 또는 OR 연산자로 연결된 경우 선행 표현식의 결과에 따라 뒤에 연산을 평가할지 말지 결정하는 최적화를 말한다.

```java
if (isMember() && hasName()) {
    // ...
}
```

- `isMember()`가 True를 반환하면 → `hasName()` 함수 호출
- `isMember()`가 False를 반환하면 → `hasName()` 검사 불필요 (결과는 False)


## DISTINCT

DISTINCT는 SELECT하는 레코드를 유니크하게 조회할 때 사용한다.

```sql
# 두 쿼리는 동일 (DISTINCT는 함수가 아님, 괄호는 무의미)
SELECT DISTINCT first_name, last_name FROM employees;
SELECT DISTINCT(first_name), last_name FROM employees;
```

> DISTINCT는 SELECT하는 레코드 전체를 유니크하게 가져온다. 특정 컬럼만 유니크하게 조회하는 것이 아니다.


## LIMIT n

LIMIT는 쿼리 결과에서 지정된 순서에 위치한 레코드만 가져오고자 할 때 사용한다.

일반적으로 MySQL 서버에서 LIMIT는 쿼리의 가장 마지막에 실행된다.  
따라서 GROUP BY 절이나 DISTINCT 등과 같이 사용됐을 때 어떻게 동작하는지 알아둘 필요가 있다.

### LIMIT의 동작 방식

```sql
# 풀 테이블 스캔하면서 10개 레코드를 읽어 들인 시점에 읽기 작업 멈춤
# LIMIT를 이용해 쿼리를 상당히 빨리 끝낼 수 있음
SELECT * FROM employees LIMIT 0, 10;
```

```sql
# GROUP BY 처리가 완료되고 나서 LIMIT 처리 수행
# LIMIT가 실질적 서버 작업을 크게 줄여주지 못함
SELECT * FROM employees GROUP BY first_name LIMIT 0, 10;
```

```sql
# 중복 제거 작업(DISTINCT)을 반복 처리하다 유니크한 레코드가 10건 채워지면  중지
# LIMIT 절을 활용해 작업량을 줄인 쿼리
SELECT DISTINCT first_name FROM employees LIMIT 0, 10;
```

### LIMIT 주의 사항

LIMIT의 인자로 표현식이나 별도의 서브쿼리를 사용할 수 없다.

```sql
SELECT * FROM employees LIMIT (100-10);
```

### 페이징 처리 시 주의사항

```sql
SELECT * FROM salaries ORDER BY salary LIMIT 0, 10;

SELECT * FROM salaries ORDER BY salary LIMIT 2000000, 10;
```

이 경우 10건의 레코드를 가져오는 결과는 같지만 MySQL 서버가 처리하는 작업 내용이 다르다.

아래 쿼리의 경우 2,000,010건의 레코드를 읽은 후 2,000,000건은 버리고 마지막 10건만 사용자에게 반환한다.

#### 효율적인 페이징 쿼리

```sql
SELECT * FROM salaries
WHERE salary >= 154888 AND NOT(salary = 154888 AND emp_no <= 109334)
ORDER BY salary LIMIT 0, 10;
```

`NOT(salary=154888 AND emp_no <= 109334)` 조건을 명시한 이유는 중복이나 누락을 방지하기 위함이다.

중복을 허용하는 인덱스를 사용하는 경우, 단순히 이전 페이지의 마지막 값보다 큰 값을 조회하거나 크거나 같은 경우를 조회하면 중복이나 누락이 발생할 수 있다.


## COUNT()

COUNT() 함수는 결과 레코드의 건수를 반환하는 함수다.

InnoDB 스토리지 엔진에서는 WHERE 조건이 없는 COUNT(*) 쿼리라고 해도 데이터나 인덱스를 읽어야 레코드 건수를 가져올 수 있다.  
따라서 큰 테이블에서 COUNT() 함수를 사용할 때는 주의해야한다.

| 주의 사항 | 설명 |
|----------|------|
| 불필요한 ORDER BY, LEFT JOIN | COUNT(*) 쿼리에 ORDER BY 절은 어떤 경우에도 필요하지 않다. LEFT JOIN도 레코드 건수의 변화가 없는 경우 제거하는 것이 성능상 좋다. |
| 인덱스 미사용 | 인덱스를 사용하지 못하는 COUNT()는 페이징 처리보다 몇 배~몇십 배 느리게 실행될 수 있다. |
| NULL 처리 | COUNT(column)은 NULL이 아닌 레코드 건수만 반환한다. NULL이 될 수 있는 컬럼은 의도대로 작동하지 않을 수 있다. |

```sql
# NULL 포함 전체 레코드 수
SELECT COUNT(*) FROM employees;

# to_date가 NULL이 아닌 레코드 수만
SELECT COUNT(to_date) FROM titles;
```

## JOIN

### JOIN의 순서와 인덱스

인덱스 레인지 스캔은 인덱스 탐색(Index Seek)과 인덱스 스캔(Index Scan) 과정으로 구분할 수 있다.  
일반적으로 인덱스를 이용한 쿼리는 요청 레코드 건수가 소량이기 때문에 인덱스 스캔보다 탐색 과정이 더 부하가 큰 편이다.

- 드라이빙 테이블: B-Tree의 루트부터 내려가는 과정이 한 번
- 드라이븐 테이블: 드라이빙 테이블에서 스캔되는 레코드 건수만큼 반복

> 드라이븐 테이블의 부하가 훨씬 크기 때문에 옵티마이저는 항상 드라이븐 테이블을 최적으로 읽을 수 있는 실행 계획을 수립한다.

### 인덱스 존재 유무에 따른 조인 동작

```sql
SELECT *
FROM employees e, dept_emp de
WHERE e.emp_no = de.emp_no;
```

| 인덱스 상태 | 동작 |
|------------|------|
| 두 컬럼 모두 인덱스 있음 | 옵티마이저가 통계 정보를 활용해 최적의 드라이빙 테이블 선택 |
| employees.emp_no에만 인덱스 | dept_emp 테이블을 드라이빙으로 선택 (employees 풀 스캔 방지) |
| dept_emp.emp_no에만 인덱스 | employees 테이블을 드라이빙으로 선택 |
| 두 컬럼 모두 인덱스 없음 | MySQL 8.0.18+에서 해시 조인 수행. 레코드 건수가 적은 테이블이 드라이빙 |


### JOIN 컬럼의 데이터 타입

조인 컬럼 간 비교에서 각 컬럼의 데이터 타입이 일치하지 않으면 인덱스를 효율적으로 사용할 수 없다.

```sql
CREATE TABLE tb_test1 (user_id INT, user_type INT, PRIMARY KEY (user_id));
CREATE TABLE tb_test2 (user_type CHAR(1), type_desc VARCHAR(10), PRIMARY KEY (user_type));

# 타입 불일치로 인덱스를 사용할 수 없는 경우
SELECT *
FROM tb_test1 tb1, tb_test2 tb2
WHERE tb1.user_type = tb2.user_type;
```

#### 인덱스를 사용하지 못하는 대표적 타입 불일치 케이스

| 케이스 | 예시 |
|--------|------|
| 데이터 타입 종류가 다른 경우 | CHAR vs INT |
| 같은 CHAR 타입이지만 문자 집합/콜레이션이 다른 경우 | utf8mb4_general_ci vs latin1_general_ci |
| 같은 INT 타입이지만 부호(Sign) 여부가 다른 경우 | INT vs INT UNSIGNED |

데이터베이스 모델에 대한 표준화 규칙을 수립하고 규칙을 기반으로 설계를 진행하는 것이 좋다.


### OUTER JOIN의 성능과 주의사항

#### 외부 조인(OUTER JOIN)의 문제점

```sql
SELECT *
FROM employees e
LEFT JOIN dept_emp de ON de.emp_no = e.emp_no
LEFT JOIN departments d ON d.dept_no = de.dept_no AND d.dept_name = 'Development';
```

MySQL 옵티마이저는 외부 조인 시 외부의 테이블들을 절대 드라이빙 테이블로 선택할 수 없다.  
따라서 풀 스캔이 필요한 employees 테이블을 드라이빙 테이블로 선택하게 되어 쿼리가 매우 비효율적이게 된다.

#### 내부 조인(INNER JOIN)으로 개선

```sql
# INNER JOIN: 옵티마이저가 조인 순서를 자유롭게 최적화 가능
SELECT *
FROM employees e
INNER JOIN dept_emp de ON de.emp_no = e.emp_no
INNER JOIN departments d ON d.dept_no = de.dept_no AND d.dept_name = 'Development';
```

내부 조인으로 변경하면 MySQL 옵티마이저가 조인 순서를 변경하는 최적화를 적용할 수 있어 쿼리 성능을 비약적으로 향상시킬 수 있다.

#### 외부 조인이 내부 조인으로 변환되는 경우

```sql
# WHERE 절에 OUTER 테이블 조건 → INNER JOIN으로 자동 변환
SELECT *
FROM employees e
LEFT JOIN dept_manager mgr ON mgr.emp_no = e.emp_no
WHERE mgr.dept_no = 'd001';  

# ON 절로 옮겨야 올바른 OUTER JOIN 동작
SELECT *
FROM employees e
LEFT JOIN dept_manager mgr ON mgr.emp_no = e.emp_no AND mgr.dept_no = 'd001';
```

#### 안티 조인(Anti-Join)

외부로 조인되는 테이블에 존재하지 않는 데이터만 추출하는 경우에는 WHERE 절에 OUTER 테이블 컬럼을 명시할 수 있다.

```sql
# 안티 조인: dept_manager에 존재하지 않는 employees 조회
SELECT *
FROM employees e
LEFT JOIN dept_manager dm ON dm.emp_no = e.emp_no
WHERE dm.emp_no IS NULL
LIMIT 10;
```

> 안티 조인만이 외부 테이블의 컬럼을 WHERE 절에 사용할 수 있는 유일한 경우이다.


### JOIN과 외래 키(FOREIGN KEY)

외래 키는 참조 무결성 보장을 위해 사용하는 것이고, 테이블 간의 조인을 위해 외래 키가 필요한 것은 아니다.

### 지연된 조인(Delayed Join)

조인을 사용한 쿼리에 GROUP BY 또는 ORDER BY 절이 사용될 때 인덱스를 사용하지 못하면, MySQL 서버는 우선 조인을 마친 후 GROUP BY, ORDER BY 절을 처리한다.

조인은 대체로 실행되면 될수록 결과 레코드 건수가 늘어나기 때문에 조인의 결과에 대해 GROUP BY, ORDER BY 절 처리를 수행하면 더 많은 레코드를 처리해야 한다.

지연된 조인(Delayed Join)은 조인 실행 전에 GROUP BY, ORDER BY 절을 처리하는 방식이다.

#### 일반 조인 (비효율적)

```sql
SELECT e.*
FROM salaries s, employees e
WHERE e.emp_no = s.emp_no
  AND s.emp_no BETWEEN 10001 AND 13000
GROUP BY s.emp_no
ORDER BY SUM(s.salary) DESC
LIMIT 10;
```

#### 지연된 조인으로 개선

```sql
# salaries만 먼저 처리하여 10건 추출 → employees와 조인
SELECT e.*
FROM (
    SELECT s.emp_no
    FROM salaries s
    WHERE s.emp_no BETWEEN 10001 AND 13000
    GROUP BY s.emp_no
    ORDER BY SUM(s.salary) DESC
    LIMIT 10
) x, employees e
WHERE e.emp_no = x.emp_no;
```

> 조인의 중간 결과는 메모리 또는 디스크에 저장된다. 저장되는 중간 결과의 레코드 건수를 줄이면 실행 효율도 그만큼 개선된다.

#### 지연된 조인 적용 조건

1. OUTER JOIN: 드라이빙 테이블과 드라이븐 테이블은 1:1 또는 N:1 관계여야 함 
2. INNER JOIN: 1:1 또는 N:1 관계 + 드라이빙 테이블에 있는 레코드와 대응되는 레코드가 드라이븐 테이블에 반드시 존재해야 함 


### 래터럴 조인(Lateral Join)

MySQL 8.0 버전부터는 래터럴 조인 기능을 통해 그룹별로 서브쿼리를 실행하여 그 결과와 조인하는 것이 가능해졌다.

```sql
# 각 직원별로 최근 급여 2건씩 조회
SELECT *
FROM employees e
LEFT JOIN LATERAL (
    SELECT *
    FROM salaries s
    WHERE s.emp_no = e.emp_no
    ORDER BY s.from_date DESC
    LIMIT 2
) s2 ON s2.emp_no = e.emp_no
WHERE e.first_name = 'Matt';
```

> LATERAL 키워드를 갖는 서브쿼리는 조인 순서상 후순위로 밀리고 외부 쿼리의 결과 레코드 단위로 임시 테이블이 생성되기 때문에 꼭 필요한 경우에만 사용해야 한다.


### 실행 계획으로 인한 정렬 흐트러짐

| 조인 방식 | 정렬 유지 여부 |
|----------|---------------|
| 네스티드-루프 조인 | 드라이빙 테이블에서 읽은 레코드의 순서가 그대로 유지 |
| 해시 조인 | 쿼리 결과의 레코드 정렬 순서가 흐트러짐 |

```sql
# 해시 조인 사용 시 emp_no 컬럼에 대해 오름차순으로 결과가 출력되지 않음
SELECT e.emp_no, e.first_name, e.last_name, de.from_date
FROM dept_emp de, employees e
WHERE de.from_date > '2001-10-01' AND e.emp_no < 10005;
```

> 특정 컬럼에 대해 정렬된 결과가 필요하다면 ORDER BY 절을 통해 명시적으로 결과를 정렬해야 한다.


## GROUP BY

### WITH ROLLUP

롤업(ROLLUP) 기능을 사용하여 그룹별로 소계를 집계할 수 있다.

```sql
SELECT dept_no, COUNT(*)
FROM dept_emp
GROUP BY dept_no WITH ROLLUP;
```
#### 다중 컬럼 롤업

```sql
SELECT first_name, last_name, COUNT(*)
FROM employees
GROUP BY first_name, last_name WITH ROLLUP;
```



#### GROUPING() 함수 (MySQL 8.0+)

MySQL 8.0 버전부터는 롤업 기능 사용 시 NULL의 대체 값을 사용자가 직접 정의할 수 있는 GROUPING() 함수를 지원한다.

```sql
SELECT
    IF(GROUPING(first_name), 'All first_name', first_name) AS first_name,
    IF(GROUPING(last_name), 'All last_name', last_name) AS last_name,
    COUNT(*)
FROM employees
GROUP BY first_name, last_name WITH ROLLUP;
```


### 레코드를 컬럼으로 변환 (Pivot)

하나의 레코드를 여러 개의 컬럼으로 나누는 Pivot 기능은 SUM(), COUNT() 같은 집계 함수와 CASE 구문을 사용해 구현할 수 있다.

#### 원래의 결과 (행 형태)

```sql
SELECT dept_no, COUNT(*) AS emp_count
FROM dept_emp
GROUP BY dept_no;
```

#### Pivoting 결과 (열 형태)

```sql
SELECT
    SUM(CASE WHEN dept_no='d001' THEN emp_count ELSE 0 END) AS count_d001,
    SUM(CASE WHEN dept_no='d002' THEN emp_count ELSE 0 END) AS count_d002,
    SUM(CASE WHEN dept_no='d003' THEN emp_count ELSE 0 END) AS count_d003,
    ...
    SUM(emp_count) AS count_total
FROM (
    SELECT dept_no, COUNT(*) AS emp_count FROM dept_emp GROUP BY dept_no
) tb_derived;
```


#### 하나의 컬럼을 여러 컬럼으로 분리

```sql
SELECT de.dept_no,
    SUM(CASE WHEN e.hire_date BETWEEN '1980-01-01' AND '1989-12-31' THEN 1 ELSE 0 END) AS cnt_1980,
    SUM(CASE WHEN e.hire_date BETWEEN '1990-01-01' AND '1999-12-31' THEN 1 ELSE 0 END) AS cnt_1990,
    SUM(CASE WHEN e.hire_date BETWEEN '2000-01-01' AND '2009-12-31' THEN 1 ELSE 0 END) AS cnt_2000,
    COUNT(*) AS cnt_total
FROM dept_emp de, employees e
WHERE e.emp_no = de.emp_no
GROUP BY de.dept_no;
```


## ORDER BY

### ORDER BY 절이 없는 SELECT 쿼리의 정렬

1. 인덱스 사용 시: 인덱싱된 컬럼 값 기준으로 정렬 
2. 풀 테이블 스캔 시: 물리적 저장 순서 (MyISAM: 저장 순서, InnoDB: 클러스터링 인덱스) 
3. 임시 테이블 사용 시: 순서를 예측하기 어려움

ORDER BY를 명시하지 않으면 정렬 순서를 보장하지 않는다.

### Using filesort

ORDER BY에서 인덱스를 사용하지 못할 때는 추가 정렬 작업이 수행되며, 쿼리 실행 계획의 Extra 컬럼에 "Using filesort" 문구가 표기된다.

정렬 대상이 많은 경우 여러 부분으로 나눠 처리되고 정렬된 결과가 임시적으로 디스크나 메모리에 저장된다.

#### 정렬 관련 상태 값

```sql
SHOW STATUS LIKE 'Sort_%';
```

1. Sort_merge_passes: 소트 버퍼와 디스크에 저장된 레코드를 병합한 횟수. 값이 증가하면 디스크를 이용했다는 의미 
2. Sort_range: 인덱스 레인지 스캔을 통해 읽은 레코드를 정렬한 횟수 
3. Sort_scan: 풀 테이블 스캔을 통해 읽은 레코드를 정렬한 횟수 
4. Sort_rows: 정렬을 수행했던 전체 레코드 건수의 누적 값 


### ORDER BY 사용법 및 주의사항

MySQL의 기본 모드에서 쌍따옴표로 감싼 문자열은 문자열 리터럴로 인식되기 때문에 다른 DBMS와 같이 `ORDER BY "first_name"`과 같은 표기를 사용하면 정렬을 수행할 수 없다.

```sql
# 문자열 리터럴로 인식되어 정렬되지 않음
SELECT * FROM employees ORDER BY "first_name";

# 올바르게 사용한 경우(백틱)
SELECT * FROM employees ORDER BY first_name;
SELECT * FROM employees ORDER BY `first_name`;  
```


### 여러 방향으로 동시 정렬 

MySQL 8.0 버전부터는 오름차순과 내림차순을 혼용해서 인덱스를 생성할 수 있다.

```sql
ALTER TABLE salaries ADD INDEX ix_salary_fromdate (salary DESC, from_date ASC);
```

#### 인덱스 역순 스캔의 한계

어떤 종류의 인덱스든 단일 컬럼의 정렬은 인덱스 역순 스캔(Index Backward Scan)을 활용해 수행할 수 있다.

하지만 인덱스 역순 스캔 시 Read-Ahead 최적화가 적용되지 않기 때문에 약간의 성능 차이가 발생할 수 있어 사용 목적에 맞는 인덱스를 생성하는 것이 중요하다.


### 함수나 표현식을 이용한 정렬 

MySQL 8.0 버전부터는 함수 기반의 인덱스를 지원하여 함수 값을 기준으로 정렬하는 작업이 인덱스를 사용하도록 튜닝할 수 있다.

```sql
# 함수 값 기반 정렬
SELECT * FROM salaries ORDER BY COS(salary);

# 함수 기반 인덱스 생성
CREATE INDEX ix_cos_salary ON salaries ((COS(salary)));
```